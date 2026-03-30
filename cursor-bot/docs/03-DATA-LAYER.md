# 数据层和仓库模块详解

## 文档概述

本文档详细介绍 Algo Bot 的数据访问层 (`src/algo_bot/repositories/`)，包括 Session 会话管理（SQLite）和项目信息访问（MySQL）的实现细节。

**前置阅读**: [00-ARCHITECTURE.md](./00-ARCHITECTURE.md)

---

## 一、模块概览

```
src/algo_bot/repositories/
├── session_repository.py         # Session 会话管理（SQLite）
└── project_info_repository.py    # 项目信息访问（MySQL）
```

---

## 二、Session 会话管理 (session_repository.py)

### 2.1 模块职责

使用 SQLite 管理 AI 多轮对话的会话映射关系，支持 24小时自动过期和定期清理。

**核心功能**:
- thread_id → cursor_session_id 映射
- 会话创建和更新
- 会话过期检查
- 消息去重（message_id 记录）
- 定期清理过期会话

### 2.2 数据库 Schema

```sql
CREATE TABLE IF NOT EXISTS sessions (
    thread_id TEXT PRIMARY KEY,           -- 业务线程 ID（群聊或 dm_{user_id}）
    session_id TEXT,                      -- Cursor Agent 会话 ID
    created_at REAL NOT NULL,             -- 创建时间（Unix timestamp）
    updated_at REAL NOT NULL,             -- 最后更新时间
    last_message_id TEXT,                 -- 最后一条消息 ID
    processed_message_ids TEXT,           -- 已处理的消息 ID 列表（JSON 数组）
    extra_data TEXT                       -- 额外数据（JSON 对象）
);

CREATE INDEX IF NOT EXISTS idx_updated_at ON sessions(updated_at);
```

**字段说明**:

| 字段 | 类型 | 说明 |
|------|------|------|
| `thread_id` | TEXT | 主键，业务线程 ID（群聊用 thread_id，私聊用 `dm_{user_id}`） |
| `session_id` | TEXT | Cursor Agent 会话 ID（用于多轮对话） |
| `created_at` | REAL | 创建时间（Unix timestamp 浮点数） |
| `updated_at` | REAL | 最后更新时间（每次交互更新） |
| `last_message_id` | TEXT | 最后一条消息 ID |
| `processed_message_ids` | TEXT | 已处理的消息 ID 列表（JSON 数组字符串） |
| `extra_data` | TEXT | 额外数据（JSON 对象字符串） |

### 2.3 数据类

```python
@dataclass
class SessionMapping:
    """会话映射数据类"""
    thread_id: str
    session_id: Optional[str] = None
    created_at: float = field(default_factory=time.time)
    updated_at: float = field(default_factory=time.time)
    last_message_id: Optional[str] = None
    processed_message_ids: List[str] = field(default_factory=list)
    extra_data: Dict[str, Any] = field(default_factory=dict)
    
    def is_expired(self, max_age_seconds: int) -> bool:
        """检查会话是否过期"""
        return time.time() - self.updated_at > max_age_seconds
```

### 2.4 核心类

```python
class SessionManager:
    """
    会话管理器
    
    使用 SQLite 存储会话映射关系，支持多线程访问。
    """
    
    def __init__(
        self,
        db_path: str = "data/sessions.db",
        max_age_hours: int = 24
    ):
        """
        初始化会话管理器
        
        Args:
            db_path: SQLite 数据库文件路径
            max_age_hours: 会话最大存活时间（小时）
        """
        self.db_path = db_path
        self.max_age_seconds = max_age_hours * 3600
        self._lock = threading.Lock()
        
        # 确保目录存在
        os.makedirs(os.path.dirname(db_path), exist_ok=True)
        
        # 初始化数据库
        self._init_database()
```

### 2.5 核心方法

#### 2.5.1 获取或创建会话

```python
def get_or_create_session(self, thread_id: str) -> SessionMapping:
    """
    获取或创建会话映射
    
    Args:
        thread_id: 线程 ID
    
    Returns:
        SessionMapping: 会话映射对象
    """
    with self._lock:
        conn = sqlite3.connect(self.db_path)
        cursor = conn.cursor()
        
        try:
            # 查询现有会话
            cursor.execute(
                "SELECT * FROM sessions WHERE thread_id = ?",
                (thread_id,)
            )
            row = cursor.fetchone()
            
            if row:
                # 解析现有会话
                session = SessionMapping(
                    thread_id=row[0],
                    session_id=row[1],
                    created_at=row[2],
                    updated_at=row[3],
                    last_message_id=row[4],
                    processed_message_ids=json.loads(row[5] or "[]"),
                    extra_data=json.loads(row[6] or "{}")
                )
                
                # 检查是否过期
                if session.is_expired(self.max_age_seconds):
                    logger.info(f"会话 {thread_id} 已过期，重置 session_id")
                    session.session_id = None
                    session.updated_at = time.time()
                    self._update_session(cursor, session)
                
                conn.commit()
                return session
            
            else:
                # 创建新会话
                session = SessionMapping(thread_id=thread_id)
                cursor.execute(
                    """
                    INSERT INTO sessions 
                    (thread_id, session_id, created_at, updated_at, 
                     last_message_id, processed_message_ids, extra_data)
                    VALUES (?, ?, ?, ?, ?, ?, ?)
                    """,
                    (
                        session.thread_id,
                        session.session_id,
                        session.created_at,
                        session.updated_at,
                        session.last_message_id,
                        json.dumps(session.processed_message_ids),
                        json.dumps(session.extra_data)
                    )
                )
                conn.commit()
                logger.info(f"创建新会话: {thread_id}")
                return session
        
        finally:
            conn.close()
```

#### 2.5.2 更新会话

```python
def update_session(
    self,
    thread_id: str,
    session_id: Optional[str] = None,
    last_message_id: Optional[str] = None,
    extra_data: Optional[Dict[str, Any]] = None
) -> None:
    """
    更新会话信息
    
    Args:
        thread_id: 线程 ID
        session_id: Cursor Agent 会话 ID（可选）
        last_message_id: 最后一条消息 ID（可选）
        extra_data: 额外数据（可选）
    """
    with self._lock:
        conn = sqlite3.connect(self.db_path)
        cursor = conn.cursor()
        
        try:
            # 获取现有会话
            session = self._get_session(cursor, thread_id)
            if not session:
                logger.warning(f"会话 {thread_id} 不存在，无法更新")
                return
            
            # 更新字段
            if session_id is not None:
                session.session_id = session_id
            if last_message_id is not None:
                session.last_message_id = last_message_id
            if extra_data is not None:
                session.extra_data.update(extra_data)
            
            # 更新时间戳
            session.updated_at = time.time()
            
            # 保存到数据库
            self._update_session(cursor, session)
            conn.commit()
        
        finally:
            conn.close()
```

#### 2.5.3 消息去重

```python
def is_message_processed(self, thread_id: str, message_id: str) -> bool:
    """
    检查消息是否已处理（去重）
    
    Args:
        thread_id: 线程 ID
        message_id: 消息 ID
    
    Returns:
        bool: 已处理返回 True，否则返回 False
    """
    with self._lock:
        conn = sqlite3.connect(self.db_path)
        cursor = conn.cursor()
        
        try:
            session = self._get_session(cursor, thread_id)
            if not session:
                return False
            
            return message_id in session.processed_message_ids
        
        finally:
            conn.close()

def mark_message_processed(self, thread_id: str, message_id: str) -> None:
    """
    标记消息已处理
    
    Args:
        thread_id: 线程 ID
        message_id: 消息 ID
    """
    with self._lock:
        conn = sqlite3.connect(self.db_path)
        cursor = conn.cursor()
        
        try:
            session = self._get_session(cursor, thread_id)
            if not session:
                logger.warning(f"会话 {thread_id} 不存在，无法标记消息")
                return
            
            # 添加到已处理列表
            if message_id not in session.processed_message_ids:
                session.processed_message_ids.append(message_id)
                
                # 限制列表长度（只保留最近 100 条）
                if len(session.processed_message_ids) > 100:
                    session.processed_message_ids = session.processed_message_ids[-100:]
            
            session.updated_at = time.time()
            self._update_session(cursor, session)
            conn.commit()
        
        finally:
            conn.close()
```

#### 2.5.4 清理过期会话

```python
def cleanup_expired_sessions(self) -> int:
    """
    清理过期的会话
    
    Returns:
        int: 清理的会话数量
    """
    with self._lock:
        conn = sqlite3.connect(self.db_path)
        cursor = conn.cursor()
        
        try:
            expired_threshold = time.time() - self.max_age_seconds
            
            cursor.execute(
                "DELETE FROM sessions WHERE updated_at < ?",
                (expired_threshold,)
            )
            
            deleted_count = cursor.rowcount
            conn.commit()
            
            if deleted_count > 0:
                logger.info(f"清理了 {deleted_count} 个过期会话")
            
            return deleted_count
        
        finally:
            conn.close()
```

### 2.6 使用示例

```python
from algo_bot.repositories.session_repository import SessionManager

# 创建管理器
manager = SessionManager(
    db_path="data/sessions.db",
    max_age_hours=24
)

# 获取或创建会话
session = manager.get_or_create_session(thread_id="NzU5ODYzODAyMzc5")

# 首次对话：session_id 为 None
if not session.session_id:
    print("首次对话，创建新 AI 会话")
else:
    print(f"恢复会话: {session.session_id}")

# 消息去重
message_id = "msg_abc123"
if manager.is_message_processed(session.thread_id, message_id):
    print("消息已处理，跳过")
else:
    # 处理消息
    print("处理消息...")
    
    # 标记已处理
    manager.mark_message_processed(session.thread_id, message_id)

# 更新会话（保存 AI 返回的 session_id）
manager.update_session(
    thread_id=session.thread_id,
    session_id="session_abc123",
    last_message_id=message_id
)

# 定期清理过期会话
deleted = manager.cleanup_expired_sessions()
print(f"清理了 {deleted} 个过期会话")
```

---

## 三、项目信息访问 (project_info_repository.py)

### 3.1 模块职责

从 MySQL 数据库读取项目元信息，支持导出到 CSV 供 Cursor Agent 使用。

**核心功能**:
- 连接 MySQL 数据库
- 查询项目信息
- 导出到 CSV
- 连接池管理

### 3.2 数据库 Schema

```sql
CREATE TABLE IF NOT EXISTS algo_bot_cursor_projects_tab (
    id INT AUTO_INCREMENT PRIMARY KEY,
    repo_name VARCHAR(255) NOT NULL,          -- 仓库名称
    git_repo_link VARCHAR(512) NOT NULL,      -- Git 仓库地址
    project_type VARCHAR(50),                 -- 项目类型
    description TEXT,                         -- 项目描述
    owner VARCHAR(255),                       -- 项目负责人
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);
```

### 3.3 核心类

```python
class ProjectInfoRepository:
    """
    项目信息仓库（MySQL）
    
    从 MySQL 读取项目元信息，支持导出到 CSV。
    """
    
    def __init__(
        self,
        host: str = None,
        port: int = None,
        user: str = None,
        password: str = None,
        database: str = None,
        table_name: str = None
    ):
        """
        初始化项目信息仓库
        
        Args:
            host: MySQL 主机地址（默认从配置读取）
            port: MySQL 端口（默认从配置读取）
            user: MySQL 用户名（默认从配置读取）
            password: MySQL 密码（默认从配置读取）
            database: MySQL 数据库名（默认从配置读取）
            table_name: 项目信息表名（默认从配置读取）
        """
        # 从配置加载（如果未提供参数）
        self.host = host or Config.MYSQL_HOST
        self.port = port or Config.MYSQL_PORT
        self.user = user or Config.MYSQL_USER
        self.password = password or Config.MYSQL_PASSWORD
        self.database = database or Config.MYSQL_DATABASE
        self.table_name = table_name or Config.MYSQL_CURSOR_PROJECTS_TABLE
        
        self.logger = get_logger("project_info_repo")
```

### 3.4 核心方法

#### 3.4.1 查询所有项目

```python
def list_all_projects(self) -> List[Dict[str, Any]]:
    """
    查询所有项目信息
    
    Returns:
        List[Dict]: 项目列表，每个项目是一个字典
    """
    conn = None
    try:
        conn = pymysql.connect(
            host=self.host,
            port=int(self.port),
            user=self.user,
            password=self.password,
            database=self.database,
            charset='utf8mb4',
            cursorclass=pymysql.cursors.DictCursor
        )
        
        with conn.cursor() as cursor:
            sql = f"SELECT * FROM {self.table_name}"
            cursor.execute(sql)
            results = cursor.fetchall()
        
        self.logger.info(f"查询到 {len(results)} 个项目")
        return results
    
    except pymysql.Error as e:
        self.logger.error(f"查询项目失败: {e}")
        return []
    
    finally:
        if conn:
            conn.close()
```

#### 3.4.2 导出到 CSV

```python
def export_to_csv(self, csv_path: str) -> bool:
    """
    导出项目信息到 CSV
    
    Args:
        csv_path: CSV 文件路径
    
    Returns:
        bool: 导出成功返回 True，否则返回 False
    """
    projects = self.list_all_projects()
    
    if not projects:
        self.logger.warning("没有项目数据可导出")
        return False
    
    try:
        # 确保目录存在
        os.makedirs(os.path.dirname(csv_path), exist_ok=True)
        
        # 写入 CSV
        with open(csv_path, "w", encoding="utf-8", newline="") as f:
            if not projects:
                self.logger.warning("没有数据可导出")
                return False
            
            fieldnames = projects[0].keys()
            writer = csv.DictWriter(f, fieldnames=fieldnames)
            
            writer.writeheader()
            writer.writerows(projects)
        
        self.logger.info(f"已导出 {len(projects)} 个项目到 {csv_path}")
        return True
    
    except Exception as e:
        self.logger.error(f"导出 CSV 失败: {e}")
        return False
```

### 3.5 使用示例

```python
from algo_bot.repositories.project_info_repository import ProjectInfoRepository

# 创建仓库
repo = ProjectInfoRepository()

# 查询所有项目
projects = repo.list_all_projects()
for project in projects:
    print(f"{project['repo_name']}: {project['git_repo_link']}")

# 导出到 CSV
success = repo.export_to_csv("projects_src/project_info.csv")
if success:
    print("导出成功")
```

---

## 四、数据流

### 4.1 Session 数据流

```
用户发送消息（FeiShu）
  ↓
消息消费者读取 Redis 队列
  ↓
SessionManager.get_or_create_session(thread_id)
  ↓ SQLite 查询
SessionMapping {
  thread_id: "NzU5ODYzODAyMzc5",
  session_id: "session_abc123",  # 如果存在
  updated_at: 1709280000.0
}
  ↓ 检查过期
如果过期 → session_id = None（创建新会话）
  ↓
调用 Cursor Agent（--resume=session_id）
  ↓ 返回新 session_id
SessionManager.update_session(thread_id, new_session_id)
  ↓ SQLite 更新
保存到数据库，updated_at 更新
```

### 4.2 Project 数据流

```
MySQL (项目元信息中心)
  ↓ ProjectInfoRepository.list_all_projects()
List[Dict] (内存)
  ↓ ProjectInfoRepository.export_to_csv()
CSV 文件 (projects_src/project_info.csv)
  ↓ projects_git_sync.py 读取
Git 同步（clone/pull）
  ↓ Cursor Agent 访问
代码分析
```

---

## 五、性能和并发

### 5.1 SQLite 并发

**问题**: SQLite 不支持高并发写入

**解决方案**:
- 使用 `threading.Lock` 保护所有数据库访问
- 单线程写入，多线程读取（SQLite 支持）
- 连接池（每个操作独立连接）

**当前场景**: 写入频率低（~5 req/h），完全够用

### 5.2 MySQL 连接池

**问题**: 频繁创建 MySQL 连接开销大

**当前方案**: 每次查询创建新连接（简单）

**优化方案**（未来）:
```python
from pymysql.pooling import ConnectionPool

pool = ConnectionPool(
    host=Config.MYSQL_HOST,
    port=Config.MYSQL_PORT,
    user=Config.MYSQL_USER,
    password=Config.MYSQL_PASSWORD,
    database=Config.MYSQL_DATABASE,
    maxconnections=10,
    blocking=True
)

conn = pool.connection()
```

---

## 六、数据备份和恢复

### 6.1 SQLite 备份

```bash
# 备份 SQLite 数据库
cp data/sessions.db data/sessions.db.backup

# 恢复
cp data/sessions.db.backup data/sessions.db
```

### 6.2 MySQL 备份

```bash
# 导出项目信息表
mysqldump -u root -p testdb algo_bot_cursor_projects_tab > projects.sql

# 导入
mysql -u root -p testdb < projects.sql
```

---

## 七、故障排查

### 7.1 Session 丢失

**症状**: 多轮对话失败，AI 不记得之前的内容

**排查步骤**:
1. 检查 SQLite 数据库是否存在: `ls -l data/sessions.db`
2. 查询会话: `sqlite3 data/sessions.db "SELECT * FROM sessions WHERE thread_id='...';"`
3. 检查 `updated_at` 是否超过 24 小时（过期）
4. 检查日志: `log/agent.log`

### 7.2 MySQL 连接失败

**症状**: 项目同步失败，无法导出 CSV

**排查步骤**:
1. 检查配置: `Config.MYSQL_HOST`, `Config.MYSQL_PORT`
2. 测试连接: `mysql -h <host> -P <port> -u <user> -p`
3. 检查防火墙规则
4. 检查 MySQL 用户权限

---

**下一步阅读**: [04-CLI-ENTRYPOINT.md](./04-CLI-ENTRYPOINT.md) - CLI入口和启动流程
