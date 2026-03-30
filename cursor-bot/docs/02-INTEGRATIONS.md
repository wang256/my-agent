# 外部集成模块详解

## 文档概述

本文档详细介绍 Algo Bot 的外部集成层 (`src/algo_bot/integrations/`)，包括 Cursor Agent、FeiShu API、GitLab 权限验证和 Redis 客户端的实现细节。

**前置阅读**: [00-ARCHITECTURE.md](./00-ARCHITECTURE.md)

---

## 一、模块概览

```
src/algo_bot/integrations/
├── cursor_agent.py       # Cursor Agent CLI 客户端
├── FeiShu_api.py        # FeiShu OpenAPI 客户端
├── gitlab_auth.py        # GitLab 权限验证
└── redis_client.py       # Redis 客户端封装
```

---

## 二、Cursor Agent 客户端 (cursor_agent.py)

### 2.1 模块职责

通过 `subprocess` 调用 `cursor-agent` CLI 工具，实现 AI 代码分析功能。

**核心功能**:
- 启动 cursor-agent 子进程
- 流式读取 JSON 输出
- Session 会话管理（多轮对话）
- 超时控制和进程管理

### 2.2 数据结构

```python
@dataclass
class CursorAgentResponse:
    """Cursor Agent 响应数据结构"""
    success: bool                              # 是否成功
    session_id: Optional[str]                  # 会话 ID
    result: Optional[Dict[str, Any]]           # 最终结果
    error: Optional[str] = None                # 错误信息
    raw_output: Optional[str] = None           # 原始输出
    stream_events: List[Dict[str, Any]] = []   # 所有流式事件
```

### 2.3 核心类

```python
class CursorAgentClient:
    def __init__(
        self,
        command: str = "cursor-agent",
        timeout: int = 120,
        working_dir: Optional[str] = None,
        api_key: Optional[str] = None,
        model: str = "composer-2"
    ):
        """
        初始化 Cursor Agent 客户端
        
        Args:
            command: cursor-agent 命令路径
            timeout: 命令执行超时时间（秒）
            working_dir: 工作目录（限制只能访问该目录）
            api_key: Cursor API Key
            model: AI 模型名称
        """
```

### 2.4 核心方法

#### 2.4.1 发送消息

```python
def send_message(
    self,
    message: str,
    session_id: Optional[str] = None,
    context: Optional[Dict[str, Any]] = None,
    on_stream_event: Optional[Callable[[Dict[str, Any]], None]] = None
) -> CursorAgentResponse:
    """
    发送消息给 Cursor Agent（流式处理）
    
    Args:
        message: 要发送的消息内容
        session_id: 会话 ID（用于多轮对话）
        context: 额外的上下文信息
        on_stream_event: 流式事件回调函数
    
    Returns:
        CursorAgentResponse: 响应结果
    """
```

**命令构建**:

```bash
# 首次对话
cursor-agent \
  -p \
  -f \
  --output-format stream-json \
  --workspace ./projects_src \
  --model composer-2 \
  --approve-mcps \
  --api-key *** \
  "分析 farm-api"

# 多轮对话（恢复 session）
cursor-agent \
  --resume=session_abc123 \
  -p \
  -f \
  --output-format stream-json \
  --workspace ./projects_src \
  --model composer-2 \
  --approve-mcps \
  --api-key *** \
  "它的配置文件在哪？"
```

**参数说明**:
- `-p`: Plan 模式（自主规划任务）
- `-f`: Force（跳过确认）
- `--output-format stream-json`: 流式 JSON 输出
- `--workspace`: 工作目录
- `--model`: AI 模型
- `--approve-mcps`: 自动批准 MCP 工具调用
- `--resume`: 恢复会话（多轮对话）

#### 2.4.2 流式输出处理

```python
# 启动子进程
process = subprocess.Popen(
    args=cmd_args,
    stdout=subprocess.PIPE,
    stderr=subprocess.PIPE,
    text=True,
    cwd=workspace_path
)

# 流式读取 stdout，每行一个 JSON
for line in process.stdout:
    line = line.rstrip('\n\r')
    
    try:
        event = json.loads(line)
        stream_events.append(event)
        
        # 提取 session_id
        if event.get('type') == 'session_id':
            new_session_id = event.get('value')
            logger.info(f"新 Session ID: {new_session_id}")
        
        # 提取最终结果
        elif event.get('type') == 'result':
            final_result = event
        
        # 回调处理
        if on_stream_event:
            on_stream_event(event)
    
    except json.JSONDecodeError:
        # 非 JSON 行，跳过
        continue

# 等待进程退出
return_code = process.wait(timeout=self.timeout)
```

**流式事件类型**:

| type | 说明 | 示例 |
|------|------|------|
| `session_id` | 会话 ID | `{"type": "session_id", "value": "abc123"}` |
| `thinking` | AI 思考过程 | `{"type": "thinking", "content": "..."}` |
| `tool_call` | AI 调用工具 | `{"type": "tool_call", "tool": "read_file", ...}` |
| `result` | 最终结果 | `{"type": "result", "content": "..."}` |

#### 2.4.3 超时和错误处理

```python
try:
    return_code = process.wait(timeout=self.timeout)
    
    if return_code != 0:
        # 进程退出码非 0
        error_msg = ''.join(stderr_output)
        return CursorAgentResponse(
            success=False,
            session_id=new_session_id,
            result=None,
            error=f"进程退出码: {return_code}, 错误: {error_msg}"
        )

except subprocess.TimeoutExpired:
    # 超时，强制终止进程
    logger.error(f"命令执行超时（>{self.timeout}s），终止进程")
    process.kill()
    process.wait()
    
    return CursorAgentResponse(
        success=False,
        session_id=new_session_id,
        result=None,
        error=f"执行超时（>{self.timeout}s）"
    )
```

### 2.5 使用示例

**首次对话**:

```python
from algo_bot.integrations.cursor_agent import CursorAgentClient

client = CursorAgentClient(
    command="cursor-agent",
    timeout=120,
    working_dir="./projects_src",
    api_key="cursor_api_key_***",
    model="composer-2"
)

response = client.send_message("分析 farm-api 的配置加载逻辑")

if response.success:
    print(f"Session ID: {response.session_id}")
    print(f"结果: {response.result['content']}")
else:
    print(f"错误: {response.error}")
```

**多轮对话**:

```python
# 第二次对话，使用之前的 session_id
session_id = response.session_id

response2 = client.send_message(
    message="它的配置文件在哪？",
    session_id=session_id  # 恢复会话
)

if response2.success:
    print(f"结果: {response2.result['content']}")
```

**流式回调**:

```python
def on_event(event: Dict[str, Any]):
    if event['type'] == 'thinking':
        print(f"AI 思考: {event['content']}")
    elif event['type'] == 'tool_call':
        print(f"AI 调用工具: {event['tool']}")

response = client.send_message(
    message="分析代码",
    on_stream_event=on_event
)
```

---

## 三、FeiShu API 客户端 (FeiShu_api.py)

### 3.1 模块职责

封装 FeiShu OpenAPI，提供消息查询、发送、文件下载等功能。

**核心功能**:
- Access Token 自动管理（获取、刷新）
- 消息查询（根据 message_id）
- 消息发送（私聊、群聊、回复、Markdown）
- 文件下载
- 群组管理

### 3.2 核心类

```python
class FeiShuAPIClient:
    """FeiShu OpenAPI 客户端"""
    
    BASE_URL = "https://openapi.FeiShu.io"
    MAX_TEXT_MESSAGE_LENGTH = 4096
    TEXT_CHUNK_SAFE_LIMIT = 3800
    TOKEN_REFRESH_BUFFER = 300  # 提前 5 分钟刷新
    
    def __init__(
        self,
        app_id: str,
        app_secret: str,
        base_url: str = None,
        timeout: int = 30
    ):
        """初始化 FeiShu API 客户端"""
```

### 3.3 Token 管理

```python
def _get_access_token(self) -> str:
    """
    获取有效的 access token（自动刷新）
    
    线程安全，支持并发调用
    """
    with self._token_lock:
        # 检查 token 是否需要刷新
        if (self._access_token and 
            time.time() < self._token_expire_at - self.TOKEN_REFRESH_BUFFER):
            return self._access_token
        
        # 刷新 token
        url = f"{self.base_url}/auth/app_access_token"
        payload = {
            "app_id": self.app_id,
            "app_secret": self.app_secret
        }
        
        response = requests.post(url, json=payload, timeout=self.timeout)
        data = response.json()
        
        if data.get("code") != 0:
            raise FeiShuAPIError(
                code=data.get("code", -1),
                message=data.get("msg", "获取 token 失败")
            )
        
        self._access_token = data["app_access_token"]
        self._token_expire_at = data["expire"]
        
        return self._access_token
```

**设计要点**:
- **线程安全**: 使用 `threading.Lock` 保护 token 访问
- **自动刷新**: 在过期前 5 分钟自动刷新
- **缓存**: 避免频繁请求 token

### 3.4 核心方法

#### 3.4.1 消息查询

```python
def get_message_by_id(self, message_id: str) -> Dict[str, Any]:
    """
    根据 message_id 获取消息内容
    
    Args:
        message_id: 消息 ID
    
    Returns:
        Dict: 消息内容 {
            "message_id": "...",
            "text": {"content": "..."},
            "tag": "text",
            ...
        }
    """
    endpoint = "/messaging/v2/get_message_by_message_id"
    params = {"message_id": message_id}
    
    data = self._make_request("GET", endpoint, params=params)
    return data.get("message", {})
```

#### 3.4.2 消息发送

```python
def send_message(
    self,
    thread_id: str,
    content: str,
    reply_to_message_id: Optional[str] = None,
    is_markdown: bool = False
) -> Dict[str, Any]:
    """
    发送消息到指定线程（私聊或群聊）
    
    Args:
        thread_id: 线程 ID（群聊或私聊）
        content: 消息内容
        reply_to_message_id: 回复的消息 ID（可选）
        is_markdown: 是否为 Markdown 格式
    
    Returns:
        Dict: 发送结果 {"message_id": "..."}
    """
    endpoint = "/messaging/v2/send_message"
    
    # 构建消息体
    message = {
        "tag": "text",
        "text": {"content": content}
    }
    
    payload = {
        "thread_id": thread_id,
        "message": message
    }
    
    # 回复消息
    if reply_to_message_id:
        payload["reply_to_message_id"] = reply_to_message_id
    
    # Markdown 格式
    if is_markdown:
        payload["message"]["text"]["text_type"] = "text/markdown"
    
    data = self._make_request("POST", endpoint, json_data=payload)
    return data.get("message", {})
```

#### 3.4.3 长消息分片

```python
def send_long_message(
    self,
    thread_id: str,
    content: str,
    reply_to_message_id: Optional[str] = None,
    is_markdown: bool = False
) -> List[Dict[str, Any]]:
    """
    发送长消息（自动分片）
    
    FeiShu 限制单条消息最大 4096 字符，
    此方法会自动分片发送。
    
    Returns:
        List[Dict]: 所有分片的发送结果
    """
    chunks = self._split_text_by_code_blocks(
        content,
        max_length=self.TEXT_CHUNK_SAFE_LIMIT
    )
    
    results = []
    for i, chunk in enumerate(chunks):
        prefix = f"[{i+1}/{len(chunks)}]\n\n" if len(chunks) > 1 else ""
        
        result = self.send_message(
            thread_id=thread_id,
            content=prefix + chunk,
            reply_to_message_id=reply_to_message_id,
            is_markdown=is_markdown
        )
        results.append(result)
        
        # 避免频繁请求
        if i < len(chunks) - 1:
            time.sleep(0.5)
    
    return results
```

**分片策略**:
- 按代码块边界分割（保持代码完整性）
- 每片 <3800 字符（安全裕量）
- 添加分片标识 `[1/3]`

#### 3.4.4 文件下载

```python
def download_file(
    self,
    file_key: str,
    save_path: str
) -> str:
    """
    下载 FeiShu 文件
    
    Args:
        file_key: 文件唯一标识
        save_path: 保存路径
    
    Returns:
        str: 保存的文件路径
    """
    # 1. 获取下载 URL
    endpoint = "/messaging/v2/download"
    params = {"file_key": file_key}
    
    data = self._make_request("GET", endpoint, params=params)
    download_url = data.get("url")
    
    # 2. 下载文件
    response = requests.get(download_url, timeout=self.timeout)
    response.raise_for_status()
    
    # 3. 保存文件
    os.makedirs(os.path.dirname(save_path), exist_ok=True)
    with open(save_path, "wb") as f:
        f.write(response.content)
    
    return save_path
```

### 3.5 使用示例

```python
from algo_bot.integrations.FeiShu_api import FeiShuAPIClient

# 创建客户端
client = FeiShuAPIClient(
    app_id="your_app_id",
    app_secret="your_app_secret"
)

# 发送消息
result = client.send_message(
    thread_id="NzU5ODYzODAyMzc5",
    content="Hello from Algo Bot!",
    is_markdown=True
)

# 发送长消息（自动分片）
client.send_long_message(
    thread_id="NzU5ODYzODAyMzc5",
    content="很长的消息内容..." * 1000,
    is_markdown=True
)

# 下载文件
client.download_file(
    file_key="file_abc123",
    save_path="./downloads/file.txt"
)
```

---

## 四、GitLab 权限验证 (gitlab_auth.py)

### 4.1 模块职责

通过 GitLab API 检查用户是否在指定的 Group 中，实现权限控制。

### 4.2 核心类

```python
class GitLabAuthChecker:
    def __init__(
        self,
        gitlab_url: str,
        private_token: str,
        allowed_group_ids: List[int],
        timeout: int = 10
    ):
        """
        初始化 GitLab 权限检查器
        
        Args:
            gitlab_url: GitLab 服务器地址
            private_token: GitLab Private Token
            allowed_group_ids: 允许的 group ID 列表
            timeout: 请求超时时间（秒）
        """
```

### 4.3 核心方法

```python
def check_user_permission(
    self,
    username: Optional[str] = None,
    email: Optional[str] = None
) -> bool:
    """
    检查用户是否有权限（是否在任何一个允许的 group 中）
    
    Args:
        username: GitLab 用户名（优先）
        email: 邮箱地址（从此提取用户名）
    
    Returns:
        bool: 有权限返回 True，否则返回 False
    """
    # 1. 未配置 group，默认允许所有用户
    if not self.allowed_group_ids:
        return True
    
    # 2. 确定用户名
    if not username and email:
        username = email.split('@')[0]
    
    if not username:
        return False
    
    # 3. 检查用户是否在任何一个允许的 group 中
    for group_id in self.allowed_group_ids:
        try:
            if self.check_user_in_group(username, group_id):
                return True
        except GitLabAuthError:
            continue
    
    return False

def check_user_in_group(self, username: str, group_id: int) -> bool:
    """
    检查用户是否在指定的 group 中
    
    Returns:
        bool: 在 group 中返回 True，否则返回 False
    """
    url = f"{self.gitlab_url}/api/v4/groups/{group_id}/members"
    headers = {"PRIVATE-TOKEN": self.private_token}
    params = {"query": username}
    
    response = requests.get(url, headers=headers, params=params, timeout=self.timeout)
    
    if response.status_code != 200:
        return False
    
    members = response.json()
    
    # 检查成员列表中是否有匹配的用户
    for member in members:
        if (member.get('username') == username and 
            member.get('state') == 'active'):
            return True
    
    return False
```

### 4.4 使用示例

```python
from algo_bot.integrations.gitlab_auth import GitLabAuthChecker

# 创建检查器
checker = GitLabAuthChecker(
    gitlab_url="https://git.garena.com",
    private_token="glpat-***",
    allowed_group_ids=[123, 456]
)

# 检查权限
if checker.check_user_permission(email="user@example.com"):
    print("用户有权限")
else:
    print("用户无权限")
```

---

## 五、Redis 客户端 (redis_client.py)

### 5.1 模块职责

封装 Redis 客户端，提供消息队列和分布式锁功能。

### 5.2 核心功能

```python
import redis

# 创建连接
client = redis.Redis(
    host="localhost",
    port=6379,
    password=None,
    db=0,
    decode_responses=True
)

# 消息队列：生产者
client.lpush("algo:bot:FeiShu:list", json.dumps(message))

# 消息队列：消费者（阻塞）
result = client.brpop("algo:bot:FeiShu:list", timeout=0)
if result:
    _, message_json = result
    message = json.loads(message_json)

# 分布式锁
lock_acquired = client.set(
    "sync_lock",
    "1",
    ex=600,  # 10 分钟过期
    nx=True  # 不存在时才设置
)

if lock_acquired:
    try:
        # 执行同步任务
        pass
    finally:
        client.delete("sync_lock")
```

---

## 六、集成示例

### 6.1 完整的消息处理流程

```python
from algo_bot.integrations.cursor_agent import CursorAgentClient
from algo_bot.integrations.FeiShu_api import FeiShuAPIClient
from algo_bot.integrations.gitlab_auth import GitLabAuthChecker

# 1. 创建客户端
cursor_client = CursorAgentClient(
    working_dir="./projects_src",
    timeout=120
)

FeiShu_client = FeiShuAPIClient(
    app_id="your_app_id",
    app_secret="your_app_secret"
)

gitlab_checker = GitLabAuthChecker(
    gitlab_url="https://git.example.com",
    private_token="token",
    allowed_group_ids=[123]
)

# 2. 权限检查
user_email = "user@example.com"
if not gitlab_checker.check_user_permission(email=user_email):
    FeiShu_client.send_message(
        thread_id="...",
        content="抱歉，您没有权限使用此服务"
    )
    return

# 3. 调用 Cursor Agent
response = cursor_client.send_message(
    message="分析 farm-api",
    session_id=session_id  # 从数据库恢复
)

# 4. 发送结果到 FeiShu
if response.success:
    FeiShu_client.send_long_message(
        thread_id="...",
        content=response.result['content'],
        is_markdown=True
    )
else:
    FeiShu_client.send_message(
        thread_id="...",
        content=f"处理失败: {response.error}"
    )
```

---

**下一步阅读**: [03-DATA-LAYER.md](./03-DATA-LAYER.md) - 数据层和仓库模块详解
