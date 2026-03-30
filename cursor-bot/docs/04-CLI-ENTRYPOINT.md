# CLI入口和启动流程

## 文档概述

本文档详细介绍 Algo Bot 的 CLI 入口模块 (`src/algo_bot/cli/`)，包括主服务入口和同步工具的实现细节。

**前置阅读**: [00-ARCHITECTURE.md](./00-ARCHITECTURE.md)

---

## 一、模块概览

```
src/algo_bot/cli/
├── algo_bot.py    # 主服务入口: algo-bot-agent
└── sync_once.py   # 同步工具: algo-bot-sync
```

**命令行工具**:
- `algo-bot-agent`: 启动主服务（消息处理 + HTTP 健康检查 + 定时同步）
- `algo-bot-sync`: 手动触发项目同步

---

## 二、主服务入口 (algo_bot.py)

### 2.1 模块职责

Algo Bot 的主入口，负责启动所有服务组件。

**核心功能**:
- 加载配置和环境变量
- 启动 HTTP 健康检查服务（后台线程）
- 启动项目同步服务（后台线程）
- 启动消息处理主循环（主线程，阻塞）
- 提供管理命令（查看会话、清理会话）

### 2.2 启动流程

```
main()
  ↓
1. 加载环境变量 (.env)
  ↓
2. 加载配置 (config.yml)
  ↓
3. 解析命令行参数
  ↓
4. 执行管理命令（如果指定）
  ├─ --list-sessions: 列出所有会话
  ├─ --cleanup-sessions: 清理过期会话
  └─ --version: 显示版本信息
  ↓
5. 启动 HTTP 健康检查服务（后台线程）
  ↓
6. 启动项目同步服务（后台线程）
  ↓
7. 启动消息处理主循环（主线程，阻塞）
```

### 2.3 核心代码结构

```python
def main():
    """主入口函数"""
    
    # 1. 加载环境变量
    load_main_process_env()
    
    # 2. 加载配置
    Config.load_config()
    Config.print_config()
    
    # 3. 解析命令行参数
    args = parse_args()
    
    # 4. 执行管理命令
    if args.list_sessions:
        list_sessions()
        return
    
    if args.cleanup_sessions:
        cleanup_sessions()
        return
    
    # 5. 启动 HTTP 健康检查服务（后台线程）
    http_thread = threading.Thread(
        target=start_http_server,
        args=(Config.SERVER_HOST, Config.SERVER_PORT),
        daemon=True,
        name="http_server"
    )
    http_thread.start()
    logger.info("HTTP 健康检查服务已启动")
    
    # 6. 启动项目同步服务（后台线程）
    if Config.PROJECT_SYNC_RUN_ON_START or Config.PROJECT_SYNC_HOUR >= 0:
        sync_thread = threading.Thread(
            target=start_integrated_sync,
            daemon=True,
            name="project_sync"
        )
        sync_thread.start()
        logger.info("项目同步服务已启动")
    
    # 7. 启动消息处理主循环（主线程，阻塞）
    start_message_loop()

if __name__ == "__main__":
    main()
```

### 2.4 消息处理主循环

```python
def start_message_loop():
    """
    消息处理主循环
    
    从 Redis 队列消费 SeaTalk 消息，提交到线程池异步处理。
    """
    # 1. 创建 Redis 客户端
    redis_client = redis.Redis(
        host=Config.REDIS_HOST,
        port=Config.REDIS_PORT,
        password=Config.REDIS_PASSWORD,
        db=Config.REDIS_DB,
        decode_responses=True
    )
    
    # 2. 创建线程池（最多 5 个并发）
    executor = ThreadPoolExecutor(
        max_workers=5,
        thread_name_prefix="agent_worker"
    )
    
    # 3. 创建 Session 管理器
    session_manager = SessionManager(
        db_path=Config.SESSION_STORE_PATH or "data/sessions.db",
        max_age_hours=Config.SESSION_MAX_AGE_HOURS
    )
    
    # 4. 创建客户端
    seatalk_client = SeaTalkAPIClient(
        app_id=Config.SEATALK_APP_ID,
        app_secret=Config.SEATALK_APP_SECRET
    )
    
    cursor_client = CursorAgentClient(
        command=Config.AGENT_COMMAND,
        timeout=Config.AGENT_TIMEOUT,
        working_dir=Config.AGENT_WORKDIR or "./projects_src",
        api_key=Config.CURSOR_API_KEY,
        model=Config.AGENT_MODEL
    )
    
    gitlab_checker = None
    if Config.GITLAB_URL and Config.GITLAB_PRIVATE_TOKEN:
        gitlab_checker = GitLabAuthChecker(
            gitlab_url=Config.GITLAB_URL,
            private_token=Config.GITLAB_PRIVATE_TOKEN,
            allowed_group_ids=Config.GITLAB_ALLOWED_GROUP_IDS
        )
    
    # 5. 消息处理主循环
    logger.info("开始消费 Redis 队列...")
    
    while True:
        try:
            # BRPOP 阻塞式拉取消息（超时 5 秒）
            result = redis_client.brpop("algo:bot:seatalk:list", timeout=5)
            
            if result is None:
                continue
            
            _, message_json = result
            message = json.loads(message_json)
            
            # 提取消息信息
            event_data = json.loads(message.get("message", {}).get("body", "{}"))
            event_type = event_data.get("event_type")
            event = event_data.get("event", {})
            timestamp = event_data.get("timestamp", 0)
            
            # 超时检查（>5分钟拒绝）
            if time.time() - timestamp > 300:
                logger.warning(f"消息超时，跳过: {timestamp}")
                continue
            
            # 事件过滤
            if event_type not in SUPPORTED_EVENT_TYPES:
                logger.info(f"不支持的事件类型，跳过: {event_type}")
                continue
            
            # 提交到线程池异步处理
            future = executor.submit(
                _process_message,
                event_data,
                session_manager,
                cursor_client,
                seatalk_client,
                gitlab_checker
            )
            
            logger.info(f"消息已提交到线程池: {event_type}")
        
        except KeyboardInterrupt:
            logger.info("收到中断信号，关闭服务...")
            executor.shutdown(wait=True)
            break
        
        except Exception as e:
            logger.error(f"处理消息失败: {e}", exc_info=True)
            time.sleep(1)
```

### 2.5 消息处理函数

```python
def _process_message(
    event_data: Dict[str, Any],
    session_manager: SessionManager,
    cursor_client: CursorAgentClient,
    seatalk_client: SeaTalkAPIClient,
    gitlab_checker: Optional[GitLabAuthChecker]
) -> None:
    """
    处理单条消息（在线程池中异步执行）
    
    Args:
        event_data: SeaTalk 事件数据
        session_manager: Session 管理器
        cursor_client: Cursor Agent 客户端
        seatalk_client: SeaTalk API 客户端
        gitlab_checker: GitLab 权限检查器（可选）
    """
    try:
        event_type = event_data.get("event_type")
        event = event_data.get("event", {})
        
        # 1. 提取消息信息
        message_data = event.get("message", {})
        message_id = message_data.get("message_id")
        thread_id = message_data.get("thread_id")
        sender_email = event.get("email")
        
        # 2. 提取消息内容
        message_content = format_seatalk_message(event_data)
        
        # 3. 确定会话标识
        if event_type == MESSAGE_FROM_BOT_SUBSCRIBER:
            # 私聊：使用 dm_{user_id}
            user_id = event.get("seatalk_id")
            thread_id = f"dm_{user_id}"
        else:
            # 群聊：使用 thread_id
            thread_id = message_data.get("thread_id")
        
        # 4. Session 管理
        session = session_manager.get_or_create_session(thread_id)
        
        # 消息去重
        if session_manager.is_message_processed(thread_id, message_id):
            logger.info(f"消息已处理，跳过: {message_id}")
            return
        
        # 标记已处理
        session_manager.mark_message_processed(thread_id, message_id)
        
        # 5. 权限验证
        if gitlab_checker:
            if not gitlab_checker.check_user_permission(email=sender_email):
                logger.warning(f"用户 {sender_email} 无权限")
                seatalk_client.send_message(
                    thread_id=thread_id,
                    content="抱歉，您没有权限使用此服务。",
                    reply_to_message_id=message_id
                )
                return
        
        # 6. 检查会话是否过期
        cursor_session_id = session.session_id
        if cursor_session_id and session.is_expired(session_manager.max_age_seconds):
            logger.info(f"会话已过期，重置 session_id")
            cursor_session_id = None
        
        # 7. 构建提示消息
        prompt = build_prompt_message(message_content, cursor_client.get_working_dir())
        
        # 8. 调用 Cursor Agent
        logger.info(f"开始处理消息: thread_id={thread_id}, session_id={cursor_session_id}")
        
        response = cursor_client.send_message(
            message=prompt,
            session_id=cursor_session_id
        )
        
        # 9. 更新 Session
        if response.success and response.session_id:
            session_manager.update_session(
                thread_id=thread_id,
                session_id=response.session_id,
                last_message_id=message_id
            )
        
        # 10. 发送结果到 SeaTalk
        if response.success:
            result_content = response.result.get("content", "")
            seatalk_client.send_long_message(
                thread_id=thread_id,
                content=result_content,
                reply_to_message_id=message_id,
                is_markdown=True
            )
        else:
            seatalk_client.send_message(
                thread_id=thread_id,
                content=f"处理失败: {response.error}",
                reply_to_message_id=message_id
            )
    
    except Exception as e:
        logger.error(f"消息处理异常: {e}", exc_info=True)
```

### 2.6 管理命令

```python
def list_sessions():
    """列出所有会话"""
    session_manager = SessionManager(
        db_path=Config.SESSION_STORE_PATH or "data/sessions.db"
    )
    
    sessions = session_manager.list_sessions(limit=100)
    
    print(f"\n共 {len(sessions)} 个会话:\n")
    for session in sessions:
        age_hours = (time.time() - session.updated_at) / 3600
        print(f"Thread ID: {session.thread_id}")
        print(f"  Session ID: {session.session_id}")
        print(f"  创建时间: {datetime.fromtimestamp(session.created_at)}")
        print(f"  更新时间: {datetime.fromtimestamp(session.updated_at)} ({age_hours:.1f}h ago)")
        print(f"  已处理消息: {len(session.processed_message_ids)}")
        print()

def cleanup_sessions():
    """清理过期会话"""
    session_manager = SessionManager(
        db_path=Config.SESSION_STORE_PATH or "data/sessions.db",
        max_age_hours=Config.SESSION_MAX_AGE_HOURS
    )
    
    deleted = session_manager.cleanup_expired_sessions()
    print(f"已清理 {deleted} 个过期会话")
```

### 2.7 命令行参数

```python
def parse_args():
    """解析命令行参数"""
    import argparse
    
    parser = argparse.ArgumentParser(description="Algo Bot Agent")
    parser.add_argument("--list-sessions", action="store_true", help="列出所有会话")
    parser.add_argument("--cleanup-sessions", action="store_true", help="清理过期会话")
    parser.add_argument("--version", action="store_true", help="显示版本信息")
    
    return parser.parse_args()
```

### 2.8 使用示例

**启动主服务**:

```bash
# 使用默认配置
uv run algo-bot-agent

# 使用自定义配置
export CONFIG_FILE=/path/to/config.yml
uv run algo-bot-agent
```

**管理命令**:

```bash
# 列出所有会话
uv run algo-bot-agent --list-sessions

# 清理过期会话
uv run algo-bot-agent --cleanup-sessions

# 显示版本
uv run algo-bot-agent --version
```

---

## 三、同步工具 (sync_once.py)

### 3.1 模块职责

提供手动触发项目同步的命令行工具。

**核心功能**:
- 从 MySQL 导出项目信息到 CSV
- 批量同步 Git 仓库
- 发送同步结果通知

### 3.2 实现

```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
"""
手动触发项目同步工具

用法:
    uv run algo-bot-sync
"""

import sys
from algo_bot.env_loader import load_main_process_env
from algo_bot.services.project_sync_service import run_integrated_sync
from algo_bot.settings import Config
from algo_bot.logging import get_logger

logger = get_logger("sync_once")

def main():
    """主入口"""
    # 1. 加载环境变量
    load_main_process_env()
    
    # 2. 加载配置
    Config.load_config()
    
    # 3. 执行同步
    logger.info("开始手动同步...")
    
    stats = run_integrated_sync(sync_type="手动同步")
    
    # 4. 打印结果
    print("\n同步完成:")
    print(f"  总记录数: {stats['total']}")
    print(f"  成功: {stats['success']}")
    print(f"  失败: {stats['failed']}")
    print(f"  跳过: {stats['skipped']}")
    print(f"  重复: {stats.get('duplicates', 0)}")
    
    # 5. 退出码
    if stats['failed'] > 0:
        sys.exit(1)
    else:
        sys.exit(0)

if __name__ == "__main__":
    main()
```

### 3.3 使用示例

```bash
# 手动触发同步
uv run algo-bot-sync

# 查看退出码
echo $?  # 0: 全部成功, 1: 部分失败
```

---

## 四、环境变量加载 (env_loader.py)

### 4.1 模块职责

加载 `.env` 和 `.env.skills` 文件，设置环境变量供 Cursor Agent 使用。

### 4.2 实现

```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
"""
环境变量加载器

加载 .env 和 .env.skills 文件，供主进程和 Cursor Agent 子进程使用。
"""

import os
import sys
from typing import List

try:
    from dotenv import load_dotenv
except ImportError:
    print("python-dotenv 未安装，请先执行: python -m pip install -e .")
    sys.exit(1)

def find_dotenv_files(repo_root: str) -> List[str]:
    """
    查找所有需要加载的 .env 文件
    
    Args:
        repo_root: 仓库根目录
    
    Returns:
        List[str]: .env 文件路径列表
    """
    env_files = []
    
    # 主 .env 文件
    main_env = os.path.join(repo_root, ".env")
    if os.path.exists(main_env):
        env_files.append(main_env)
    
    # Skills .env 文件
    skills_env = os.path.join(repo_root, ".env.skills")
    if os.path.exists(skills_env):
        env_files.append(skills_env)
    
    return env_files

def load_main_process_env():
    """
    加载主进程环境变量
    
    加载顺序:
      1. .env (主配置)
      2. .env.skills (Skills 配置)
    """
    # 确定仓库根目录
    repo_root = os.path.dirname(os.path.dirname(os.path.dirname(os.path.abspath(__file__))))
    
    # 查找并加载所有 .env 文件
    env_files = find_dotenv_files(repo_root)
    
    for env_file in env_files:
        load_dotenv(env_file, override=False)
        print(f"已加载环境变量: {env_file}")
    
    if not env_files:
        print("警告: 未找到 .env 文件")
```

### 4.3 .env 文件示例

**.env**:
```bash
# Cursor API
CURSOR_API_KEY=cursor_api_key_***

# SeaTalk API
SEATALK_APP_ID=your_app_id
SEATALK_APP_SECRET=your_app_secret
SEATALK_SIGNING_SECRET=your_signing_secret

# Redis
REDIS_HOST=localhost
REDIS_PORT=6379
REDIS_PASSWORD=

# MySQL
MYSQL_HOST=localhost
MYSQL_PORT=3306
MYSQL_USER=root
MYSQL_PASSWORD=root123
MYSQL_DATABASE=testdb

# GitLab
GITLAB_URL=https://git.garena.com
GITLAB_PRIVATE_TOKEN=glpat-***
GITLAB_ALLOWED_GROUP_IDS=123,456
```

**.env.skills**:
```bash
# Trace 诊断工具 API
TRACE_API_URL=https://trace.example.com/api
TRACE_API_KEY=trace_api_key_***

# Confluence API
CONFLUENCE_URL=https://wiki.example.com
CONFLUENCE_TOKEN=confluence_token_***
```

---

## 五、启动检查清单

### 5.1 必需配置

**配置文件** (`config/config.yml`):
- ✅ `seatalk_bot_api.app_id`
- ✅ `seatalk_bot_api.app_secret`
- ✅ `cursor_agent.agent_workdir`
- ✅ `redis.host` 和 `redis.port`
- ✅ `mysql.host` 和 `mysql.database`

**环境变量** (`.env`):
- ✅ `CURSOR_API_KEY`
- ⚠️ `SEATALK_APP_SECRET` (可从配置文件读取)

### 5.2 可选配置

- `gitlab_url` 和 `gitlab_private_token` (权限验证)
- `project_sync.*` (自动同步)
- `.env.skills` (扩展工具)

### 5.3 依赖检查

```bash
# 1. 检查 cursor-agent 是否安装
which cursor-agent

# 2. 检查 Redis 连接
redis-cli -h localhost -p 6379 ping

# 3. 检查 MySQL 连接
mysql -h localhost -P 3306 -u root -p -e "SHOW DATABASES;"

# 4. 检查工作目录
ls -la ./projects_src/
```

---

## 六、故障排查

### 6.1 服务无法启动

**症状**: `algo-bot-agent` 启动失败

**排查步骤**:
1. 检查配置文件: `cat config/config.yml`
2. 检查环境变量: `printenv | grep CURSOR`
3. 检查日志: `tail -f log/agent.log`
4. 检查依赖: `cursor-agent --version`

### 6.2 消息处理失败

**症状**: 用户发消息，AI 无响应

**排查步骤**:
1. 检查 Redis 队列: `redis-cli llen algo:bot:seatalk:list`
2. 检查线程池: `ps -T -p <pid>`
3. 检查日志: `grep ERROR log/agent.log`
4. 检查 Cursor API: `echo $CURSOR_API_KEY`

### 6.3 会话丢失

**症状**: 多轮对话失败

**排查步骤**:
1. 检查 SQLite: `ls -l data/sessions.db`
2. 查询会话: `uv run algo-bot-agent --list-sessions`
3. 检查过期时间: `Config.SESSION_MAX_AGE_HOURS`

---

**下一步阅读**: [05-CONFIGURATION.md](./05-CONFIGURATION.md) - 配置系统详解
