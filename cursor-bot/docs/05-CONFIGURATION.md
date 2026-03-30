# 配置系统详解

## 文档概述

本文档详细介绍 Algo Bot 的配置系统 (`src/algo_bot/settings.py`)，包括配置文件格式、环境变量、配置加载流程和所有配置项说明。

**前置阅读**: [00-ARCHITECTURE.md](./00-ARCHITECTURE.md)

---

## 一、配置概览

### 1.1 配置来源

Algo Bot 支持多种配置来源，按优先级从高到低:

1. **环境变量** (最高优先级)
2. **配置文件** (`config/config.yml`)
3. **代码默认值** (最低优先级)

### 1.2 配置文件

**默认路径**: `config/config.yml`

**自定义路径**:
```bash
export CONFIG_FILE=/path/to/custom-config.yml
uv run algo-bot-agent
```

---

## 二、配置文件格式

### 2.1 完整配置示例

```yaml
# config/config.yml

# HTTP 服务配置
server:
  host: 0.0.0.0
  port: 8080

# Redis 配置（用于消息队列和分布式锁）
redis:
  host: localhost
  port: 6379
  password:  # 可选
  db: 0

# SeaTalk Bot API 配置
seatalk_bot_api:
  app_id: "your_app_id"
  app_secret: "your_app_secret"
  signing_secret: "your_signing_secret"
  download_dir: "./projects_src/downloads/seatalk"

# Cursor Agent 配置
cursor_agent:
  agent_command: "cursor-agent"
  agent_timeout: 120               # 秒
  agent_model: "composer-2"        # AI 模型
  agent_workdir: "./projects_src"  # 工作目录
  cursor_api_key: "cursor_api_key_***"
  session_store_type: "sqlite"
  session_store_path: "data/sessions.db"
  session_max_age_hours: 24        # 会话过期时间

# GitLab 权限验证配置（可选）
mcp:
  gitlab_url: "https://git.garena.com"
  gitlab_private_token: "glpat-***"
  gitlab_allowed_group_ids: [123, 456]  # 允许的 Group IDs

# Git 同步配置
git_sync:
  fetch_timeout: 600   # git fetch 超时（秒）
  pull_timeout: 600    # git pull 超时（秒）
  clone_timeout: 600   # git clone 超时（秒）

# 项目同步配置
project_sync:
  run_on_start: true             # 启动时立即同步
  hour: 9                        # 定时同步：每天 09:00
  minute: 0
  csv_file: "projects_src/project_info.csv"
  projects_dir: "./projects_src"
  enable_export: true            # 是否导出 MySQL 数据
  max_workers: 3                 # 并发数

# MySQL 配置
mysql:
  host: localhost
  port: 3306
  user: root
  password: root123
  database: testdb
  cursor_projects_table: "algo_bot_cursor_projects_tab"

# 日志配置
logger:
  level: INFO  # DEBUG, INFO, WARNING, ERROR
  file: "log/agent.log"
```

### 2.2 最小配置

```yaml
# 最小可运行配置

seatalk_bot_api:
  app_id: "your_app_id"
  app_secret: "your_app_secret"

cursor_agent:
  agent_workdir: "./projects_src"
  cursor_api_key: "cursor_api_key_***"

redis:
  host: localhost
  port: 6379

mysql:
  host: localhost
  database: testdb
  user: root
  password: root123
```

---

## 三、配置项详解

### 3.1 HTTP 服务配置

```yaml
server:
  host: 0.0.0.0      # 监听地址（0.0.0.0 监听所有网卡）
  port: 8080         # 监听端口
```

**说明**:
- 用于健康检查接口 `/health`
- 容器部署时设置为 `0.0.0.0`
- 本地开发可设置为 `127.0.0.1`

### 3.2 Redis 配置

```yaml
redis:
  host: localhost             # Redis 服务器地址
  port: 6379                  # Redis 端口
  password: ""                # Redis 密码（可选）
  db: 0                       # Redis 数据库编号
```

**用途**:
1. **消息队列**: `algo:bot:seatalk:list`
2. **分布式锁**: `algo:bot:sync:lock`

**生产环境建议**:
- 使用 Redis 哨兵或集群
- 启用持久化 (AOF + RDB)
- 设置密码

### 3.3 SeaTalk Bot API 配置

```yaml
seatalk_bot_api:
  app_id: "your_app_id"                  # SeaTalk 应用 ID
  app_secret: "your_app_secret"          # SeaTalk 应用密钥
  signing_secret: "your_signing_secret"  # Webhook 签名密钥
  download_dir: "./projects_src/downloads/seatalk"  # 文件下载目录
```

**获取方式**:
1. 登录 SeaTalk 开放平台
2. 创建或选择应用
3. 在「应用信息」页面获取 App ID 和 App Secret
4. 在「事件订阅」页面获取 Signing Secret

**安全性**:
- ⚠️ **不要提交到版本控制**
- ✅ 使用环境变量或配置文件（加密存储）

### 3.4 Cursor Agent 配置

```yaml
cursor_agent:
  agent_command: "cursor-agent"       # 命令路径
  agent_timeout: 120                  # 超时时间（秒）
  agent_model: "composer-2"           # AI 模型
  agent_workdir: "./projects_src"     # 工作目录（绝对路径）
  cursor_api_key: "cursor_api_key_***"  # Cursor API Key
  session_store_type: "sqlite"        # 会话存储类型
  session_store_path: "data/sessions.db"  # SQLite 数据库路径
  session_max_age_hours: 24           # 会话过期时间（小时）
```

**字段说明**:

| 字段 | 说明 | 默认值 |
|------|------|--------|
| `agent_command` | cursor-agent 命令路径 | `cursor-agent` |
| `agent_timeout` | AI 分析超时时间（秒） | `120` |
| `agent_model` | AI 模型名称 | `composer-2` |
| `agent_workdir` | 工作目录（所有仓库的父目录） | `./projects_src` |
| `cursor_api_key` | Cursor API Key | - |
| `session_max_age_hours` | 会话过期时间（小时） | `24` |

**工作目录说明**:
```
./projects_src/
├── .cursor/
│   ├── sessions/      # AI 会话缓存
│   └── skills/        # → 软链到 resources/skills/
├── farm-api/          # Git 仓库 1
├── admin-gateway/     # Git 仓库 2
└── project_info.csv   # 项目元信息
```

### 3.5 GitLab 权限验证配置

```yaml
mcp:
  gitlab_url: "https://git.garena.com"          # GitLab 服务器地址
  gitlab_private_token: "glpat-***"             # GitLab Private Token
  gitlab_allowed_group_ids: [123, 456, 789]     # 允许的 Group IDs
```

**获取 Private Token**:
1. 登录 GitLab
2. 进入 「用户设置」→「访问令牌」
3. 创建新令牌，勾选 `read_api` 权限

**获取 Group ID**:
1. 进入 GitLab Group 页面
2. URL 中的数字即为 Group ID
3. 或使用 API: `curl -H "PRIVATE-TOKEN: xxx" https://git.example.com/api/v4/groups`

**禁用权限检查**:
```yaml
# 方法1: 不配置 mcp 部分
# 方法2: group IDs 设为空
mcp:
  gitlab_allowed_group_ids: []
```

### 3.6 Git 同步配置

```yaml
git_sync:
  fetch_timeout: 600    # git fetch 超时（秒）
  pull_timeout: 600     # git pull 超时（秒）
  clone_timeout: 600    # git clone 超时（秒）
```

**超时设置建议**:

| 网络环境 | fetch | pull | clone |
|----------|-------|------|-------|
| **内网快速** | 60 | 60 | 120 |
| **默认** | 600 | 600 | 600 |
| **外网慢速** | 1200 | 1200 | 1800 |

### 3.7 项目同步配置

```yaml
project_sync:
  run_on_start: true                          # 启动时立即同步
  hour: 9                                     # 定时同步：每天 09:00
  minute: 0
  csv_file: "projects_src/project_info.csv"   # CSV 文件路径
  projects_dir: "./projects_src"              # Git 仓库目录
  enable_export: true                         # 是否导出 MySQL 数据
  max_workers: 3                              # 并发数
```

**字段说明**:

| 字段 | 说明 | 默认值 |
|------|------|--------|
| `run_on_start` | 启动时立即同步 | `true` |
| `hour` | 定时同步：小时（0-23） | `9` |
| `minute` | 定时同步：分钟（0-59） | `0` |
| `csv_file` | 项目信息 CSV 文件路径 | `projects_src/project_info.csv` |
| `projects_dir` | Git 仓库存储目录 | `./projects_src` |
| `enable_export` | 是否从 MySQL 导出 CSV | `true` |
| `max_workers` | 并发同步数 | `3` |

**禁用定时同步**:
```yaml
project_sync:
  run_on_start: false
  hour: -1  # 负数表示禁用
```

### 3.8 MySQL 配置

```yaml
mysql:
  host: localhost                                 # MySQL 服务器地址
  port: 3306                                      # MySQL 端口
  user: root                                      # MySQL 用户名
  password: root123                               # MySQL 密码
  database: testdb                                # 数据库名
  cursor_projects_table: "algo_bot_cursor_projects_tab"  # 项目信息表名
```

**表结构**:

请参考 `sql/create_project_info_table.sql`:

```sql
CREATE TABLE IF NOT EXISTS algo_bot_cursor_projects_tab (
    id INT AUTO_INCREMENT PRIMARY KEY,
    repo_name VARCHAR(255) NOT NULL,
    git_repo_link VARCHAR(512) NOT NULL,
    project_type VARCHAR(50),
    description TEXT,
    owner VARCHAR(255),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);
```

### 3.9 日志配置

```yaml
logger:
  level: INFO              # 日志级别: DEBUG, INFO, WARNING, ERROR
  file: "log/agent.log"    # 日志文件路径
```

**日志级别说明**:

| 级别 | 说明 | 适用场景 |
|------|------|----------|
| `DEBUG` | 详细调试信息 | 开发调试 |
| `INFO` | 一般信息 | 生产环境（默认） |
| `WARNING` | 警告信息 | 生产环境 |
| `ERROR` | 错误信息 | 生产环境 |

**日志文件位置**:
- 默认: `log/agent.log`
- 同步服务: `log/sync_projects.log`

---

## 四、环境变量

### 4.1 环境变量优先级

环境变量优先级 **高于** 配置文件。

**示例**:

```bash
# .env
CURSOR_API_KEY=cursor_api_key_from_env

# config.yml
cursor_agent:
  cursor_api_key: "cursor_api_key_from_config"

# 结果: 使用 cursor_api_key_from_env
```

### 4.2 环境变量命名规则

配置文件路径 → 环境变量名:

```
server.host → SERVER_HOST
redis.port → REDIS_PORT
cursor_agent.agent_timeout → AGENT_TIMEOUT
mcp.gitlab_url → GITLAB_URL
```

**规则**:
1. 全部大写
2. `.` 替换为 `_`
3. 嵌套路径展开

### 4.3 常用环境变量

```bash
# Cursor API
export CURSOR_API_KEY=cursor_api_key_***

# SeaTalk API
export SEATALK_APP_ID=your_app_id
export SEATALK_APP_SECRET=your_app_secret

# Redis
export REDIS_HOST=localhost
export REDIS_PORT=6379

# MySQL
export MYSQL_HOST=localhost
export MYSQL_PORT=3306
export MYSQL_USER=root
export MYSQL_PASSWORD=root123
export MYSQL_DATABASE=testdb

# GitLab
export GITLAB_URL=https://git.garena.com
export GITLAB_PRIVATE_TOKEN=glpat-***
export GITLAB_ALLOWED_GROUP_IDS=123,456

# 配置文件路径
export CONFIG_FILE=/path/to/config.yml
```

---

## 五、配置加载流程

### 5.1 加载顺序

```python
1. 加载 .env 文件（环境变量）
   ↓
2. 读取配置文件（config/config.yml）
   ↓
3. 合并配置（环境变量覆盖配置文件）
   ↓
4. 填充默认值（代码中定义的默认值）
   ↓
5. 验证配置（必填项检查）
   ↓
6. 应用配置（Config 类）
```

### 5.2 Config 类

```python
from algo_bot.settings import Config

# 加载配置
Config.load_config()

# 访问配置
print(Config.SERVER_HOST)
print(Config.CURSOR_API_KEY)
print(Config.GITLAB_ALLOWED_GROUP_IDS)

# 打印所有配置（隐藏敏感信息）
Config.print_config()
```

### 5.3 配置验证

**必填项**:
- `SEATALK_APP_ID`
- `SEATALK_APP_SECRET`
- `CURSOR_API_KEY`
- `REDIS_HOST`
- `MYSQL_HOST`

**验证失败**: 启动时报错并退出

---

## 六、配置最佳实践

### 6.1 生产环境

```yaml
# config/production.yml

server:
  host: 0.0.0.0
  port: 8080

redis:
  host: redis.production.example.com
  port: 6379
  password: "***"  # 强制设置密码
  db: 0

cursor_agent:
  agent_timeout: 120
  agent_model: "composer-2"
  agent_workdir: "/data/projects_src"  # 绝对路径

git_sync:
  fetch_timeout: 300
  pull_timeout: 300
  clone_timeout: 600

project_sync:
  run_on_start: true
  hour: 3  # 凌晨 3 点同步（避开高峰）
  minute: 0
  max_workers: 5  # 增加并发

logger:
  level: INFO
  file: "/var/log/algo-bot/agent.log"
```

### 6.2 开发环境

```yaml
# config/development.yml

server:
  host: 127.0.0.1
  port: 8080

redis:
  host: localhost
  port: 6379
  db: 1  # 使用独立的 db

cursor_agent:
  agent_timeout: 180  # 开发时延长超时
  agent_workdir: "./projects_src"

project_sync:
  run_on_start: false  # 手动触发
  hour: -1  # 禁用定时同步

logger:
  level: DEBUG  # 详细日志
```

### 6.3 容器部署

```bash
# Dockerfile
ENV CONFIG_FILE=/app/config/config.yml
ENV CURSOR_API_KEY=cursor_api_key_***
ENV REDIS_HOST=redis
ENV MYSQL_HOST=mysql

# docker-compose.yml
services:
  algo-bot:
    environment:
      - CONFIG_FILE=/app/config/config.yml
      - CURSOR_API_KEY=${CURSOR_API_KEY}
      - REDIS_HOST=redis
      - MYSQL_HOST=mysql
```

---

## 七、配置故障排查

### 7.1 配置未生效

**排查步骤**:
1. 检查配置文件路径: `echo $CONFIG_FILE`
2. 检查文件内容: `cat config/config.yml`
3. 检查环境变量: `printenv | grep CURSOR`
4. 启动时查看日志: `grep "已从.*加载配置" log/agent.log`

### 7.2 敏感信息泄露

**症状**: 日志中出现 API Key 或密码

**检查**:
```bash
# 搜索日志中的敏感信息
grep -i "api_key\|password\|secret" log/*.log
```

**修复**:
- 代码中使用 `_mask_value_for_log()` 函数
- 日志记录前过滤敏感字段

### 7.3 配置冲突

**症状**: 配置值与预期不符

**排查**:
```python
# 打印最终生效的配置
Config.load_config()
Config.print_config()
```

---

**下一步阅读**: [06-DEPLOYMENT.md](./06-DEPLOYMENT.md) - 部署和运维文档
