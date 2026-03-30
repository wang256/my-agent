# 核心服务模块详解

## 文档概述

本文档详细介绍 Algo Bot 的核心服务层 (`src/algo_bot/services/`)，包括消息处理、Git 同步、HTTP 服务等关键模块的实现细节。

**前置阅读**: [00-ARCHITECTURE.md](./00-ARCHITECTURE.md)

---

## 一、模块概览

```
src/algo_bot/services/
├── seatalk_callback.py      # SeaTalk 回调工具函数
├── projects_git_sync.py     # Git 仓库同步服务
├── project_sync_service.py  # 项目同步调度服务
├── http_server.py           # HTTP 健康检查服务
└── health.py                # 健康检查实现
```

---

## 二、SeaTalk 回调工具 (seatalk_callback.py)

### 2.1 模块职责

提供 SeaTalk webhook 回调的**共享工具函数**，用于签名验证、消息解析、事件过滤等。

**注意**: 这不是完整的回调处理服务，而是工具函数集合。

### 2.2 核心常量

```python
# 事件类型常量
EVENT_VERIFICATION = "event_verification"
MESSAGE_FROM_BOT_SUBSCRIBER = "message_from_bot_subscriber"
NEW_MENTIONED_MESSAGE_RECEIVED_FROM_GROUP_CHAT = "new_mentioned_message_received_from_group_chat"

# 支持的事件类型（会被转发处理）
SUPPORTED_EVENT_TYPES = {
    MESSAGE_FROM_BOT_SUBSCRIBER,                        # 用户私聊机器人
    NEW_MENTIONED_MESSAGE_RECEIVED_FROM_GROUP_CHAT,     # 群聊 @ 机器人
}
```

**设计说明**:
- `EVENT_VERIFICATION`: SeaTalk 验证 webhook URL 可用性时发送，需特殊处理
- `SUPPORTED_EVENT_TYPES`: 定义哪些事件会被业务处理，其他事件静默忽略

### 2.3 签名验证函数

```python
def is_valid_signature(signing_secret: str, body: bytes, signature: str) -> bool:
    """
    验证 SeaTalk 回调签名
    
    Args:
        signing_secret: 签名密钥（来自配置）
        body: 原始 HTTP body（必须是 bytes，不能先解析再序列化）
        signature: HTTP header 中的签名
    
    Returns:
        bool: 签名是否有效
    """
    digest = hashlib.sha256(body + signing_secret.encode("utf-8")).hexdigest()
    return digest == signature
```

**关键点**:

1. **签名算法**: `sha256(body + signing_secret)`
   - ⚠️ **不是标准 HMAC**，而是简单的字符串拼接哈希
   - 这是 SeaTalk 平台定义的协议，必须遵循

2. **body 必须是原始字节**:
   ```python
   # ✅ 正确
   body = await request.body()  # 原始 bytes
   is_valid = is_valid_signature(secret, body, signature)
   
   # ❌ 错误
   data = await request.json()
   body = json.dumps(data).encode()  # 序列化后的 body 可能不同
   is_valid = is_valid_signature(secret, body, signature)
   ```

3. **安全性**:
   - 所有回调请求必须验证签名
   - 签名错误应拒绝请求（返回 401）
   - 防止伪造请求和重放攻击

### 2.4 消息解析函数

```python
def parse_callback_event(body: bytes) -> Dict[str, Any]:
    """
    解析并验证 SeaTalk 回调消息
    
    Args:
        body: 原始 HTTP body（bytes）
    
    Returns:
        Dict: 解析后的 JSON 对象
    
    Raises:
        ValueError: 如果 JSON 无效、格式错误、缺少必需字段
    """
    try:
        payload = json.loads(body.decode("utf-8"))
    except (UnicodeDecodeError, json.JSONDecodeError) as exc:
        raise ValueError("invalid_json") from exc
    
    # 验证是否为字典
    if not isinstance(payload, dict):
        raise ValueError("invalid_payload")
    
    # 验证必需字段
    event_type = payload.get("event_type")
    event = payload.get("event")
    if not isinstance(event_type, str) or not event_type:
        raise ValueError("missing_event_type")
    if not isinstance(event, dict):
        raise ValueError("missing_event")
    
    return payload
```

**错误处理**:
- `invalid_json`: JSON 解析失败
- `invalid_payload`: payload 不是字典
- `missing_event_type`: 缺少 event_type 字段
- `missing_event`: 缺少 event 字段

### 2.5 使用示例

**在 Webhook 适配器中使用**:

```python
from algo_bot.services.seatalk_callback import (
    is_valid_signature,
    parse_callback_event,
    SUPPORTED_EVENT_TYPES,
    EVENT_VERIFICATION
)

@app.post("/algobot/api/callback")
async def callback(request: Request):
    # 1. 获取签名和 body
    signature = request.headers.get("signature")
    if not signature:
        raise HTTPException(status_code=400, detail="Missing signature")
    
    body = await request.body()
    
    # 2. 验证签名
    if not is_valid_signature(Config.SEATALK_SIGNING_SECRET, body, signature):
        raise HTTPException(status_code=401, detail="Invalid signature")
    
    # 3. 解析消息
    try:
        payload = parse_callback_event(body)
    except ValueError as e:
        raise HTTPException(status_code=400, detail=str(e))
    
    # 4. 处理事件
    event_type = payload["event_type"]
    
    # 特殊事件：验证 URL
    if event_type == EVENT_VERIFICATION:
        challenge = payload["event"].get("seatalk_challenge")
        return {"seatalk_challenge": challenge}
    
    # 过滤事件
    if event_type not in SUPPORTED_EVENT_TYPES:
        return {"status": "ignored"}
    
    # 转发到 Redis 队列
    redis_client.lpush("algo:bot:seatalk:list", body.decode("utf-8"))
    return {"status": "accepted"}
```

---

## 三、Git 仓库同步服务 (projects_git_sync.py)

### 3.1 模块职责

负责从 MySQL 导出的 CSV 文件读取项目信息，批量同步 Git 仓库到本地工作目录。

**核心功能**:
- 从 `project_info.csv` 读取项目列表
- 执行 `git clone`（首次）或 `git pull`（更新）
- 智能分支选择（release → master → main）
- 并发同步（ThreadPoolExecutor）
- 发送同步结果通知到 SeaTalk

### 3.2 核心配置

```python
# 默认配置（可通过 Config 类覆盖）
DEFAULT_CSV_FILE = "projects_src/project_info.csv"
DEFAULT_PROJECTS_DIR = "./projects_src"
DEFAULT_SYNC_NOTIFY_GROUP_ID = "Nzg0NjMzMDc3NzQ1"

# 分支优先级
BRANCH_PRIORITY = ("release", "master", "main")

# 超时配置（从 Config 读取）
GIT_SYNC_FETCH_TIMEOUT = 600  # 秒
GIT_SYNC_PULL_TIMEOUT = 600   # 秒
GIT_SYNC_CLONE_TIMEOUT = 600  # 秒
```

### 3.3 核心函数

#### 3.3.1 URL 规范化

```python
def normalize_git_url(git_url: str) -> Optional[str]:
    """
    规范化 Git URL，保留原始协议（HTTPS 或 SSH）
    
    Args:
        git_url: 原始 Git URL
    
    Returns:
        规范化后的 URL，可直接用于 git clone/pull
        如果无效则返回 None
    
    Examples:
        "https://git.example.com/group/repo.git" → 保留 HTTPS
        "git@git.example.com:group/repo.git" → 保留 SSH
    """
    if not git_url:
        return None
    
    # 移除首尾空白
    git_url = git_url.strip()
    
    # SSH URL 格式检查
    if git_url.startswith("git@"):
        if ".git" in git_url or ":" in git_url:
            return git_url
        return None
    
    # HTTPS URL 格式检查
    if git_url.startswith(("http://", "https://")):
        try:
            parsed = urlparse(git_url)
            if parsed.netloc and parsed.path:
                return git_url
        except Exception:
            return None
    
    return None
```

**设计要点**:
- **保留原始协议**: HTTPS 地址可配合 Git token，SSH 地址继续按 SSH 方式使用
- **格式验证**: 基本的 URL 格式检查
- **容错**: 无效 URL 返回 None，不抛异常

#### 3.3.2 分支选择

```python
def detect_best_branch(repo_dir: str, sync_logger=logger) -> Optional[str]:
    """
    检测本地仓库的最佳分支（按优先级：release → master → main）
    
    Args:
        repo_dir: 本地仓库目录
        sync_logger: 日志记录器
    
    Returns:
        分支名称，如果都不存在则返回 None
    """
    for branch_name in BRANCH_PRIORITY:
        result = subprocess.run(
            ["git", "rev-parse", "--verify", f"refs/remotes/origin/{branch_name}"],
            cwd=repo_dir,
            stdout=subprocess.PIPE,
            stderr=subprocess.PIPE,
            text=True,
            timeout=5
        )
        if result.returncode == 0:
            sync_logger.debug(f"选择分支: {branch_name}")
            return branch_name
    
    sync_logger.warning("未找到 release/master/main 分支")
    return None
```

**分支优先级说明**:
1. **release**: 生产环境分支（优先）
2. **master**: 主分支
3. **main**: 新仓库的默认分支名

#### 3.3.3 Git 操作

**克隆新仓库**:

```python
def clone_repository(
    git_url: str,
    repo_name: str,
    projects_dir: str,
    sync_logger=logger
) -> Dict[str, any]:
    """
    克隆新仓库
    
    Returns:
        {"success": bool, "message": str}
    """
    repo_path = os.path.join(projects_dir, repo_name)
    
    # 确保父目录存在
    os.makedirs(projects_dir, exist_ok=True)
    
    # 执行 git clone
    cmd = ["git", "clone", git_url, repo_path]
    timeout = get_git_sync_timeouts()["clone"]
    
    try:
        result = subprocess.run(
            cmd,
            stdout=subprocess.PIPE,
            stderr=subprocess.PIPE,
            text=True,
            timeout=timeout
        )
        
        if result.returncode != 0:
            error_msg = result.stderr.strip() or result.stdout.strip()
            return {"success": False, "message": error_msg}
        
        # 切换到最佳分支
        best_branch = detect_best_branch(repo_path, sync_logger)
        if best_branch:
            subprocess.run(
                ["git", "checkout", best_branch],
                cwd=repo_path,
                timeout=30
            )
        
        return {"success": True, "message": "克隆成功"}
    
    except subprocess.TimeoutExpired:
        return {"success": False, "message": f"超时（>{timeout}s）"}
    except Exception as e:
        return {"success": False, "message": str(e)}
```

**更新已有仓库**:

```python
def pull_repository(
    repo_name: str,
    projects_dir: str,
    sync_logger=logger
) -> Dict[str, any]:
    """
    更新已有仓库（git fetch + git pull）
    
    Returns:
        {"success": bool, "message": str}
    """
    repo_path = os.path.join(projects_dir, repo_name)
    timeouts = get_git_sync_timeouts()
    
    try:
        # 1. git fetch
        result = subprocess.run(
            ["git", "fetch", "origin"],
            cwd=repo_path,
            stdout=subprocess.PIPE,
            stderr=subprocess.PIPE,
            text=True,
            timeout=timeouts["fetch"]
        )
        
        if result.returncode != 0:
            return {"success": False, "message": "fetch 失败"}
        
        # 2. 检测最佳分支
        best_branch = detect_best_branch(repo_path, sync_logger)
        if not best_branch:
            return {"success": False, "message": "未找到可用分支"}
        
        # 3. git checkout
        subprocess.run(
            ["git", "checkout", best_branch],
            cwd=repo_path,
            timeout=30
        )
        
        # 4. git pull
        result = subprocess.run(
            ["git", "pull", "origin", best_branch],
            cwd=repo_path,
            stdout=subprocess.PIPE,
            stderr=subprocess.PIPE,
            text=True,
            timeout=timeouts["pull"]
        )
        
        if result.returncode != 0:
            return {"success": False, "message": "pull 失败"}
        
        return {"success": True, "message": "更新成功"}
    
    except subprocess.TimeoutExpired:
        return {"success": False, "message": "超时"}
    except Exception as e:
        return {"success": False, "message": str(e)}
```

#### 3.3.4 并发同步

```python
def sync_projects_from_csv(
    csv_file: str = DEFAULT_CSV_FILE,
    projects_dir: str = DEFAULT_PROJECTS_DIR,
    max_workers: int = 3,
    seatalk_client: Optional[SeaTalkAPIClient] = None,
    sync_logger=logger
) -> Dict[str, int]:
    """
    从 CSV 文件批量同步项目
    
    Args:
        csv_file: CSV 文件路径
        projects_dir: 目标目录
        max_workers: 并发数
        seatalk_client: SeaTalk 客户端（用于发送通知）
        sync_logger: 日志记录器
    
    Returns:
        统计信息字典 {
            "total": 总记录数,
            "success": 成功数,
            "failed": 失败数,
            "skipped": 跳过数,
            "duplicates": 重复数
        }
    """
    stats = {
        "total": 0,
        "success": 0,
        "failed": 0,
        "skipped": 0,
        "duplicates": 0
    }
    
    # 1. 读取 CSV
    projects = []
    seen_repos = set()
    
    with open(csv_file, "r", encoding="utf-8") as f:
        reader = csv.DictReader(f)
        for row in reader:
            stats["total"] += 1
            
            git_url = row.get("git_repo_link", "").strip()
            repo_name = row.get("repo_name", "").strip()
            
            # 跳过无效记录
            if not git_url or not repo_name:
                stats["skipped"] += 1
                continue
            
            # 去重
            if repo_name in seen_repos:
                stats["duplicates"] += 1
                continue
            
            seen_repos.add(repo_name)
            projects.append({"git_url": git_url, "repo_name": repo_name})
    
    sync_logger.info(f"从 CSV 读取 {stats['total']} 条记录，去重后 {len(projects)} 个项目")
    
    # 2. 并发同步
    with ThreadPoolExecutor(max_workers=max_workers) as executor:
        futures = {}
        
        for project in projects:
            future = executor.submit(
                _sync_one_project,
                project["git_url"],
                project["repo_name"],
                projects_dir,
                sync_logger
            )
            futures[future] = project
        
        # 3. 收集结果
        for future in as_completed(futures):
            project = futures[future]
            try:
                result = future.result()
                if result["success"]:
                    stats["success"] += 1
                else:
                    stats["failed"] += 1
                    sync_logger.error(
                        f"同步失败: {project['repo_name']} - {result['message']}"
                    )
            except Exception as e:
                stats["failed"] += 1
                sync_logger.error(f"同步异常: {project['repo_name']} - {e}")
    
    # 4. 发送通知
    if seatalk_client:
        send_sync_result_notification(
            seatalk_client,
            DEFAULT_SYNC_NOTIFY_GROUP_ID,
            stats,
            sync_logger
        )
    
    sync_logger.info(f"同步完成: {stats}")
    return stats
```

**并发策略**:
- 使用 `ThreadPoolExecutor` 并发执行 Git 操作
- `max_workers=3`: 默认 3 个并发（可配置）
- 避免过多并发导致 Git 服务器压力过大

### 3.4 SeaTalk 通知

```python
def build_sync_result_message(stats: Dict[str, int], sync_type: str) -> str:
    """
    构建同步结果通知消息（Markdown 格式）
    
    Returns:
        Markdown 格式的通知文本
    """
    timestamp = datetime.now().strftime('%Y-%m-%d %H:%M:%S')
    total = stats.get('total', 0)
    success = stats.get('success', 0)
    failed = stats.get('failed', 0)
    skipped = stats.get('skipped', 0)
    duplicates = stats.get('duplicates', 0)
    unique_projects = success + failed
    
    # 根据失败数选择 emoji
    if failed == 0:
        status_emoji = "✅"
    elif failed < success:
        status_emoji = "⚠️"
    else:
        status_emoji = "❌"
    
    return f"""{status_emoji} **项目代码{sync_type}完成**

**时间**: {timestamp}

**统计信息**:
- CSV总记录数: {total}
- 去重后项目数: {unique_projects}
- 重复跳过: {duplicates}
- ✅ 成功: {success}
- ❌ 失败: {failed}
- ⏭️ 跳过: {skipped}

**状态**: {'全部成功' if failed == 0 else f'{failed} 个项目同步失败'}"""

def send_sync_result_notification(
    seatalk_client: Optional[SeaTalkAPIClient],
    group_id: str,
    stats: Dict[str, int],
    sync_logger=logger,
    sync_type: str = "定时同步"
) -> bool:
    """
    发送同步结果通知到 SeaTalk 群组
    
    Returns:
        bool: 是否发送成功
    """
    if not seatalk_client or not group_id:
        return False
    
    try:
        message = build_sync_result_message(stats, sync_type)
        seatalk_client.send_message_to_group(group_id=group_id, content=message)
        sync_logger.info(f"同步结果通知已发送到群组: {group_id}")
        return True
    except Exception as e:
        sync_logger.error(f"发送同步结果通知失败: {e}", exc_info=True)
        return False
```

### 3.5 使用示例

**命令行直接执行**:

```bash
# 使用默认配置
uv run python -m algo_bot.services.projects_git_sync

# 指定 CSV 文件和目标目录
uv run python -m algo_bot.services.projects_git_sync \
  --csv-file /path/to/projects.csv \
  --projects-dir /path/to/workdir
```

**在代码中调用**:

```python
from algo_bot.services.projects_git_sync import sync_projects_from_csv
from algo_bot.integrations.seatalk_api import get_seatalk_client

# 创建 SeaTalk 客户端（可选）
seatalk_client = get_seatalk_client(app_id, app_secret)

# 执行同步
stats = sync_projects_from_csv(
    csv_file="projects_src/project_info.csv",
    projects_dir="./projects_src",
    max_workers=3,
    seatalk_client=seatalk_client
)

print(f"同步完成: 成功 {stats['success']}, 失败 {stats['failed']}")
```

---

## 四、项目同步调度服务 (project_sync_service.py)

### 4.1 模块职责

集成项目信息导出（MySQL → CSV）和 Git 仓库同步，提供定时调度功能。

**核心功能**:
- MySQL 项目信息导出到 CSV
- 调用 Git 同步服务
- 支持启动时立即同步
- 支持定时同步（每天固定时间）

### 4.2 核心函数

#### 4.2.1 导出项目信息

```python
def export_projects_to_csv() -> bool:
    """
    从 MySQL 导出项目信息到 CSV
    
    Returns:
        bool: 是否导出成功
    """
    from algo_bot.repositories.project_info_repository import ProjectInfoRepository
    
    try:
        repo = ProjectInfoRepository()
        projects = repo.list_all_projects()
        
        csv_file = Config.PROJECT_SYNC_CSV_FILE
        os.makedirs(os.path.dirname(csv_file), exist_ok=True)
        
        with open(csv_file, "w", encoding="utf-8", newline="") as f:
            if not projects:
                logger.warning("MySQL 中无项目数据")
                return False
            
            fieldnames = projects[0].keys()
            writer = csv.DictWriter(f, fieldnames=fieldnames)
            writer.writeheader()
            writer.writerows(projects)
        
        logger.info(f"已导出 {len(projects)} 个项目到 {csv_file}")
        return True
    
    except Exception as e:
        logger.error(f"导出项目信息失败: {e}", exc_info=True)
        return False
```

#### 4.2.2 集成同步

```python
def run_integrated_sync(sync_type: str = "手动同步") -> Dict[str, int]:
    """
    执行完整的同步流程：MySQL 导出 → Git 同步
    
    Args:
        sync_type: 同步类型标识（用于通知消息）
    
    Returns:
        Git 同步的统计信息
    """
    logger.info(f"开始 {sync_type}...")
    
    # 1. 导出项目信息（如果启用）
    if Config.PROJECT_SYNC_ENABLE_EXPORT:
        logger.info("步骤 1/2: 导出项目信息...")
        if not export_projects_to_csv():
            logger.error("导出失败，同步终止")
            return {"total": 0, "success": 0, "failed": 0}
    else:
        logger.info("跳过导出（PROJECT_SYNC_ENABLE_EXPORT=False）")
    
    # 2. Git 同步
    logger.info("步骤 2/2: 同步 Git 仓库...")
    seatalk_client = create_sync_notification_client()
    
    stats = sync_projects_from_csv(
        csv_file=Config.PROJECT_SYNC_CSV_FILE,
        projects_dir=Config.PROJECT_SYNC_DIR,
        max_workers=Config.PROJECT_SYNC_MAX_WORKERS,
        seatalk_client=seatalk_client,
        sync_type=sync_type
    )
    
    logger.info(f"{sync_type}完成: {stats}")
    return stats
```

#### 4.2.3 定时调度

```python
import schedule
import time

def start_integrated_sync():
    """
    启动集成同步服务（支持定时调度）
    """
    # 1. 启动时立即同步（如果配置启用）
    if Config.PROJECT_SYNC_RUN_ON_START:
        logger.info("启动时立即同步...")
        run_integrated_sync(sync_type="启动同步")
    
    # 2. 设置定时任务
    schedule_time = f"{Config.PROJECT_SYNC_HOUR:02d}:{Config.PROJECT_SYNC_MINUTE:02d}"
    schedule.every().day.at(schedule_time).do(
        run_integrated_sync,
        sync_type="定时同步"
    )
    
    logger.info(f"定时同步已设置: 每天 {schedule_time}")
    
    # 3. 调度循环
    while True:
        schedule.run_pending()
        time.sleep(60)  # 每分钟检查一次
```

### 4.3 配置说明

```yaml
# config/config.yml
project_sync:
  run_on_start: true          # 启动时立即同步
  hour: 9                     # 定时同步：每天 09:00
  minute: 0
  csv_file: "projects_src/project_info.csv"
  projects_dir: "./projects_src"
  enable_export: true         # 是否导出 MySQL 数据
  max_workers: 3              # 并发数
```

### 4.4 使用示例

**作为后台服务运行**:

```python
from algo_bot.services.project_sync_service import start_integrated_sync
import threading

# 在单独线程中运行（不阻塞主线程）
sync_thread = threading.Thread(target=start_integrated_sync, daemon=True)
sync_thread.start()
```

**手动触发同步**:

```python
from algo_bot.services.project_sync_service import run_integrated_sync

stats = run_integrated_sync(sync_type="手动同步")
print(f"同步完成: {stats}")
```

---

## 五、HTTP 健康检查服务 (http_server.py + health.py)

### 5.1 模块职责

提供简单的 HTTP 健康检查接口，用于容器编排（K8s/Docker）的健康探测。

**核心功能**:
- GET /health: 返回 200 OK
- 轻量级，无鉴权
- 不检测外部依赖（Redis/MySQL）

### 5.2 实现 (health.py)

```python
from http.server import BaseHTTPRequestHandler, HTTPServer
import json

class HealthCheckHandler(BaseHTTPRequestHandler):
    """健康检查 HTTP 处理器"""
    
    def do_GET(self):
        """处理 GET 请求"""
        if self.path == "/health":
            # 返回 200 OK
            self.send_response(200)
            self.send_header("Content-Type", "application/json")
            self.end_headers()
            
            response = {"status": "ok"}
            self.wfile.write(json.dumps(response).encode("utf-8"))
        else:
            # 其他路径返回 404
            self.send_response(404)
            self.end_headers()
    
    def log_message(self, format, *args):
        """禁用日志输出（避免过多日志）"""
        pass
```

### 5.3 HTTP 服务器 (http_server.py)

```python
def start_http_server(host: str = "0.0.0.0", port: int = 8080):
    """
    启动 HTTP 健康检查服务器
    
    Args:
        host: 监听地址
        port: 监听端口
    """
    from algo_bot.services.health import HealthCheckHandler
    
    server = HTTPServer((host, port), HealthCheckHandler)
    logger.info(f"HTTP 健康检查服务启动: http://{host}:{port}/health")
    
    try:
        server.serve_forever()
    except KeyboardInterrupt:
        logger.info("HTTP 服务器关闭")
        server.shutdown()
```

### 5.4 使用示例

**在主服务中启动**:

```python
from algo_bot.services.http_server import start_http_server
import threading

# 在单独线程中运行（不阻塞主线程）
http_thread = threading.Thread(
    target=start_http_server,
    args=("0.0.0.0", 8080),
    daemon=True
)
http_thread.start()
```

**健康检查**:

```bash
curl http://localhost:8080/health
# 输出: {"status": "ok"}
```

**K8s 配置**:

```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 30

readinessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 10
```

---

## 六、服务集成示例

### 6.1 在主服务中集成所有服务

```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
"""Algo Bot 主服务"""

import threading
from algo_bot.services.http_server import start_http_server
from algo_bot.services.project_sync_service import start_integrated_sync
from algo_bot.settings import Config

def main():
    # 1. 加载配置
    Config.load_config()
    Config.print_config()
    
    # 2. 启动 HTTP 健康检查服务（后台线程）
    http_thread = threading.Thread(
        target=start_http_server,
        args=(Config.SERVER_HOST, Config.SERVER_PORT),
        daemon=True
    )
    http_thread.start()
    
    # 3. 启动项目同步服务（后台线程）
    sync_thread = threading.Thread(
        target=start_integrated_sync,
        daemon=True
    )
    sync_thread.start()
    
    # 4. 启动消息处理主循环（阻塞）
    from algo_bot.cli.algo_bot import start_message_loop
    start_message_loop()

if __name__ == "__main__":
    main()
```

---

## 七、测试建议

### 7.1 签名验证测试

```python
import hashlib

def test_signature_validation():
    secret = "test_secret"
    body = b'{"event_type": "test"}'
    
    # 生成正确签名
    correct_sig = hashlib.sha256(body + secret.encode()).hexdigest()
    
    # 测试正确签名
    assert is_valid_signature(secret, body, correct_sig) == True
    
    # 测试错误签名
    assert is_valid_signature(secret, body, "wrong_sig") == False
```

### 7.2 Git 同步测试

```bash
# 准备测试 CSV
cat > test_projects.csv <<EOF
repo_name,git_repo_link
test-repo,https://github.com/user/test-repo.git
EOF

# 执行同步
uv run python -m algo_bot.services.projects_git_sync \
  --csv-file test_projects.csv \
  --projects-dir ./test_workdir

# 验证结果
ls test_workdir/test-repo/
```

---

## 八、故障排查

### 8.1 Git 同步失败

**症状**: 同步统计显示 `failed > 0`

**排查步骤**:
1. 检查日志: `log/sync_projects.log`
2. 检查 Git URL 格式是否正确
3. 检查网络连通性: `git clone <url>`
4. 检查 Git 凭据配置（SSH key 或 HTTPS token）
5. 检查超时配置是否足够

### 8.2 健康检查不通过

**症状**: `curl http://localhost:8080/health` 失败

**排查步骤**:
1. 检查进程是否启动: `ps aux | grep algo-bot`
2. 检查端口是否监听: `netstat -an | grep 8080`
3. 检查防火墙规则
4. 检查配置: `Config.SERVER_PORT`

---

**下一步阅读**: [02-INTEGRATIONS.md](./02-INTEGRATIONS.md) - 外部集成模块详解
