# Algo Bot 系统架构文档

## 文档概述

本文档是 Algo Bot 项目的顶层架构设计文档，描述系统的整体架构、技术选型、核心设计理念和各模块关系。

**目标读者**: 需要完整理解或复现该项目的开发者/AI

**相关文档**:
- `01-CORE-SERVICES.md`: 核心服务模块详解
- `02-INTEGRATIONS.md`: 外部集成模块详解
- `03-DATA-LAYER.md`: 数据层和仓库模块详解
- `04-CLI-ENTRYPOINT.md`: CLI入口和启动流程
- `05-CONFIGURATION.md`: 配置系统详解
- `06-DEPLOYMENT.md`: 部署和运维文档

---

## 一、项目定位

### 1.1 项目概述

**Algo Bot** 是一个基于 **Cursor Agent**、**SeaTalk** 和 **Redis** 构建的企业级智能代码分析平台，为团队提供 7×24 小时的 AI 代码咨询服务。

**核心价值**:
- 将先进的 AI 代码分析能力集成到企业内部工作流
- 通过即时通讯界面（SeaTalk）与 AI 交互
- 支持跨仓库代码分析、多轮对话、问题诊断等场景
- 提升研发效率和代码质量

### 1.2 核心特性

| 特性 | 说明 |
|------|------|
| **多轮对话** | 支持上下文连续对话，AI 记得之前的内容 |
| **跨仓库分析** | 可同时分析多个 Git 仓库的代码 |
| **权限管控** | 基于 GitLab Group 的访问权限验证 |
| **异步处理** | 消息队列解耦，支持高并发 |
| **进程隔离** | 每次 AI 分析独立子进程，保证稳定性 |
| **智能同步** | 定时预同步 Git 仓库，加速响应 |
| **可扩展性** | 通过 Skills 机制扩展 AI 能力 |

---

## 二、整体架构

### 2.1 系统架构图

```
┌──────────────────────────────────────────────────────────────────┐
│                        SeaTalk 平台                               │
│  (即时通讯，用户发送消息，接收 AI 回复)                           │
└────────────────────────┬─────────────────────────────────────────┘
                         │ Webhook (HTTP POST)
                         ↓
┌──────────────────────────────────────────────────────────────────┐
│                   Webhook 适配器 (可选独立部署)                    │
│  • 快速签名验证 (<100ms)                                          │
│  • 事件过滤和协议转换                                              │
│  • 技术栈: FastAPI + Uvicorn                                     │
└────────────────────────┬─────────────────────────────────────────┘
                         │ LPUSH
                         ↓
┌──────────────────────────────────────────────────────────────────┐
│                   Redis List 消息队列                             │
│  • Key: algo:bot:seatalk:list                                    │
│  • 作用: 异步解耦、削峰填谷                                        │
└────────────────────────┬─────────────────────────────────────────┘
                         │ BRPOP (阻塞拉取)
                         ↓
┌──────────────────────────────────────────────────────────────────┐
│                  Algo Bot 业务处理服务 (本项目)                    │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │  消息处理主循环 (services/seatalk_callback.py)             │  │
│  │  • Redis 消费者                                            │  │
│  │  • 消息去重                                                │  │
│  │  • 权限验证                                                │  │
│  └──────────────────────┬─────────────────────────────────────┘  │
│                         │                                         │
│  ┌──────────────────────▼─────────────────────────────────────┐  │
│  │  异步任务处理 (ThreadPoolExecutor, max_workers=5)          │  │
│  │  • 并发处理多个用户请求                                    │  │
│  │  • 进程隔离，崩溃不影响主服务                               │  │
│  └──────────────────────┬─────────────────────────────────────┘  │
│                         │                                         │
│  ┌──────────────────────▼─────────────────────────────────────┐  │
│  │  Session 管理 (repositories/session_repository.py)         │  │
│  │  • 双层映射: thread_id → cursor_session_id                 │  │
│  │  • SQLite 持久化                                           │  │
│  │  • 24小时自动过期                                          │  │
│  └──────────────────────┬─────────────────────────────────────┘  │
│                         │                                         │
│  ┌──────────────────────▼─────────────────────────────────────┐  │
│  │  Cursor Agent 调用 (integrations/cursor_agent.py)          │  │
│  │  • subprocess.Popen 启动子进程                             │  │
│  │  • 流式 JSON 输出解析                                      │  │
│  │  • 超时控制和进程管理                                       │  │
│  └──────────────────────┬─────────────────────────────────────┘  │
│                         │                                         │
└─────────────────────────┼─────────────────────────────────────────┘
                          │
                          ↓
┌──────────────────────────────────────────────────────────────────┐
│                   cursor-agent (子进程)                           │
│  • AI 引擎 (Claude Sonnet 4.5)                                   │
│  • 代码分析、Git 操作、文件读取                                   │
│  • 共享工作目录: ./projects_src/                                  │
│  • Session 恢复支持多轮对话                                       │
└──────────────────────────────────────────────────────────────────┘
```

### 2.2 核心设计理念

#### 1. 关注点分离

```
┌─────────────────┐       ┌─────────────────┐
│ Webhook 适配器   │       │  业务处理服务    │
├─────────────────┤       ├─────────────────┤
│ • 签名验证       │       │ • AI 代码分析    │
│ • 事件过滤       │       │ • 权限管理       │
│ • 快速响应       │       │ • 会话管理       │
│ • 轻量级         │       │ • Git 同步       │
└─────────────────┘       └─────────────────┘
      轻量适配层                重量业务层
```

**优势**:
- 职责清晰，易于维护
- 故障隔离（一个服务挂了不影响另一个）
- 独立扩展（可根据负载独立扩展各个服务）

#### 2. 异步解耦

```
同步阻塞方案（不可行）:
  Webhook → [处理 30-120秒] → 响应 ❌ 超时

异步解耦方案（本项目）:
  Webhook → [验证 <100ms] → 响应 200 ✅
     ↓
  Redis 队列
     ↓
  异步处理 [30-120秒]
```

**优势**:
- Webhook 0% 超时
- 支持高并发（峰值 10 req/s）
- 削峰填谷，保护下游服务

#### 3. 进程隔离

```
每次 AI 分析 = 独立 subprocess
  ↓
子进程崩溃 ≠ 主服务崩溃
  ↓
自动资源回收 + 超时强制终止
```

**优势**:
- 稳定性高（崩溃不影响主服务）
- 资源隔离（内存、CPU 完全独立）
- 易于超时控制（kill 子进程即可）

#### 4. 共享工作目录 + 多层隔离

```
./projects_src/              # 所有进程共享（持久化）
├── .cursor/sessions/
│   ├── session_001.json     # 进程1的会话
│   ├── session_002.json     # 进程2的会话
├── farm-api/                # 仓库A（子目录隔离）
├── admin-gateway/           # 仓库B（子目录隔离）
└── project_info.csv         # 项目元信息（共享读取）
```

**多层隔离机制**:

| 隔离层 | 机制 | 效果 |
|--------|------|------|
| **进程隔离** | 操作系统 | 内存、CPU 完全独立 |
| **Session 隔离** | 不同 session_id | AI 上下文独立 |
| **子目录隔离** | 不同仓库在不同子目录 | farm-api/ vs admin-gateway/ |
| **Git 锁保护** | .git/index.lock | 防止并发 git 操作破坏 |
| **文件名隔离** | 按 session_id 命名 | session_001.json vs session_002.json |

**优势**:
- 支持跨仓库分析（工作目录共享）
- 并发安全（多层隔离机制）
- 资源复用（Git 仓库持久化，跨重启复用）

---

## 三、技术栈

### 3.1 核心技术

| 类别 | 技术 | 版本要求 | 用途 |
|------|------|----------|------|
| **语言** | Python | ≥3.11 | 主要开发语言 |
| **Web框架** | FastAPI | ≥0.115.0 | HTTP 服务（可选的 Webhook 适配器） |
| **ASGI服务器** | Uvicorn | ≥0.30.0 | FastAPI 生产部署 |
| **消息队列** | Redis | ≥5.0.0 | 异步解耦、分布式锁 |
| **数据库** | SQLite | - | Session 会话管理 |
| **数据库** | MySQL | ≥5.7 | 项目元信息存储 |
| **AI引擎** | cursor-agent | latest | 代码分析、AI对话 |
| **IM平台** | SeaTalk | - | 用户交互界面 |
| **版本控制** | Git | ≥2.0 | 代码仓库管理 |

### 3.2 Python 依赖

**核心依赖** (pyproject.toml):
```toml
[project]
dependencies = [
    "redis>=5.0.0",           # Redis 客户端
    "pyyaml>=6.0",            # 配置文件解析
    "requests>=2.31.0",       # HTTP 请求（SeaTalk API）
    "python-dotenv>=1.0.0",   # 环境变量加载
    "fastapi>=0.115.0",       # Web 框架（可选）
    "uvicorn>=0.30.0",        # ASGI 服务器（可选）
    "pymysql>=1.1.0",         # MySQL 客户端
    "cryptography>=42.0.0",   # MySQL SSL 连接
    "schedule>=1.2.0",        # 定时任务
]
```

### 3.3 外部服务依赖

| 服务 | 用途 | 是否必需 |
|------|------|----------|
| **Redis** | 消息队列、分布式锁 | ✅ 必需 |
| **MySQL** | 项目元信息存储 | ✅ 必需 |
| **SeaTalk** | 用户交互界面 | ✅ 必需 |
| **GitLab** | 权限验证、代码托管 | ⚠️ 可选（可禁用权限检查） |
| **cursor-agent** | AI 代码分析引擎 | ✅ 必需 |

---

## 四、目录结构

```
algo-bot/
├── src/algo_bot/              # 主包源码
│   ├── __init__.py
│   ├── cli/                   # 命令行入口
│   │   ├── algo_bot.py        # 主入口: algo-bot-agent
│   │   └── sync_once.py       # 同步工具: algo-bot-sync
│   ├── services/              # 业务服务层
│   │   ├── seatalk_callback.py    # SeaTalk 回调处理
│   │   ├── projects_git_sync.py   # Git 同步服务
│   │   ├── project_sync_service.py # 项目同步调度
│   │   ├── http_server.py         # HTTP 健康检查服务
│   │   └── health.py              # 健康检查实现
│   ├── integrations/          # 外部集成层
│   │   ├── cursor_agent.py        # Cursor Agent 客户端
│   │   ├── seatalk_api.py         # SeaTalk API 客户端
│   │   ├── gitlab_auth.py         # GitLab 权限验证
│   │   └── redis_client.py        # Redis 客户端封装
│   ├── repositories/          # 数据访问层
│   │   ├── session_repository.py      # Session 管理（SQLite）
│   │   └── project_info_repository.py # 项目信息访问（MySQL）
│   ├── settings.py            # 配置管理
│   ├── logging.py             # 日志系统
│   └── env_loader.py          # 环境变量加载
├── config/
│   └── config.yml             # 主配置文件
├── resources/
│   ├── prompt.txt             # AI 提示词模板
│   └── skills/                # 可扩展的 Skills
│       ├── dag_tracer/        # Trace 诊断工具
│       └── confluence-tools/  # Confluence 查询工具
├── sql/
│   └── create_project_info_table.sql  # MySQL 表结构
├── data/                      # 运行时数据目录
│   └── sessions.db            # SQLite 数据库文件
├── log/                       # 日志文件目录
├── projects_src/              # Git 仓库工作目录
│   ├── .cursor/
│   │   ├── sessions/          # AI 会话缓存
│   │   └── skills/            # → 软链到 resources/skills/
│   ├── project_info.csv       # 项目元信息（从 MySQL 导出）
│   └── <各个Git仓库>/
├── pyproject.toml             # 项目配置和依赖
├── .env                       # 环境变量（需手动创建）
└── docs/                      # 项目文档
    ├── 00-ARCHITECTURE.md     # 本文档
    ├── 01-CORE-SERVICES.md
    ├── 02-INTEGRATIONS.md
    ├── 03-DATA-LAYER.md
    ├── 04-CLI-ENTRYPOINT.md
    ├── 05-CONFIGURATION.md
    └── 06-DEPLOYMENT.md
```

---

## 五、核心流程

### 5.1 消息处理完整流程

```
┌─────────────────────────────────────────────────────────────────┐
│ 1. 用户在 SeaTalk 发送消息                                       │
│    "@AlgoBot 分析 farm-api 的配置加载逻辑"                       │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ↓
┌─────────────────────────────────────────────────────────────────┐
│ 2. SeaTalk 平台发送 Webhook 到适配器                             │
│    POST /algobot/api/callback                                   │
│    Header: signature                                            │
│    Body: {event_type, event, timestamp...}                      │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ↓
┌─────────────────────────────────────────────────────────────────┐
│ 3. Webhook 适配器处理（可选独立部署）                             │
│    • 签名验证（SHA256）                                          │
│    • 事件过滤（只转发 message_from_bot_subscriber 等）            │
│    • LPUSH 到 Redis: algo:bot:seatalk:list                      │
│    • 快速响应 200 OK (<100ms)                                   │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ↓
┌─────────────────────────────────────────────────────────────────┐
│ 4. Algo Bot 从 Redis 消费消息                                    │
│    BRPOP algo:bot:seatalk:list 0                                │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ↓
┌─────────────────────────────────────────────────────────────────┐
│ 5. 消息去重和超时检查                                             │
│    • 检查 message_id 是否已处理                                  │
│    • 检查消息时间戳（>5分钟拒绝）                                 │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ↓
┌─────────────────────────────────────────────────────────────────┐
│ 6. 权限验证（如果配置了 GitLab）                                  │
│    • 获取用户 email                                              │
│    • 调用 GitLab API 查询用户是否在允许的 Group                  │
│    • 拒绝或允许继续                                              │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ↓
┌─────────────────────────────────────────────────────────────────┐
│ 7. 提交到线程池异步处理                                           │
│    ThreadPoolExecutor.submit(_process_message, ...)             │
│    （主循环继续消费下一条消息）                                    │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ↓
┌─────────────────────────────────────────────────────────────────┐
│ 8. Session 管理                                                 │
│    • 获取 thread_id（群聊）或 dm_{user_id}（私聊）               │
│    • 查询 SQLite: thread_id → cursor_session_id                 │
│    • 检查会话是否过期（24小时）                                   │
│    • 决定是恢复会话还是创建新会话                                 │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ↓
┌─────────────────────────────────────────────────────────────────┐
│ 9. 调用 Cursor Agent                                            │
│    subprocess.Popen([                                           │
│      'cursor-agent',                                            │
│      '--workspace=./projects_src',                              │
│      '--resume=session_xxx',  # 如果是多轮对话                   │
│      prompt                                                     │
│    ])                                                           │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ↓
┌─────────────────────────────────────────────────────────────────┐
│ 10. 流式读取 JSON 输出                                           │
│     for line in process.stdout:                                 │
│       event = json.loads(line)                                  │
│       if event['type'] == 'session_id':                         │
│         session_id = event['value']  # 保存到 SQLite             │
│       elif event['type'] == 'result':                           │
│         final_result = event['content']                         │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ↓
┌─────────────────────────────────────────────────────────────────┐
│ 11. 发送结果到 SeaTalk                                           │
│     SeaTalkAPIClient.send_message(                              │
│       thread_id=thread_id,                                      │
│       content=final_result                                      │
│     )                                                           │
└─────────────────────────────────────────────────────────────────┘
```

### 5.2 多轮对话流程

```
首次对话:
  用户: "分析 farm-api"
    ↓
  SQLite: thread_id → NULL（无 session_id）
    ↓
  cursor-agent 启动（无 --resume）
    ↓
  cursor-agent 输出: {"type": "session_id", "value": "session_abc123"}
    ↓
  SQLite: UPDATE thread_id → "session_abc123"
    ↓
  AI 回复: [分析报告]

第二次对话（同一线程）:
  用户: "它的配置文件在哪？"
    ↓
  SQLite: thread_id → "session_abc123"（已存在）
    ↓
  cursor-agent --resume=session_abc123 启动
    ↓
  AI 加载上下文: farm-api 相关的历史对话
    ↓
  AI 理解"它"指的是 farm-api
    ↓
  AI 回复: [配置文件位置]
```

---

## 六、数据流

### 6.1 消息数据流

```
SeaTalk 原始消息
  ↓ JSON
Redis 队列（完整保留）
  ↓ 解析
thread_id, user_email, message_content
  ↓ 权限验证
允许/拒绝
  ↓ Session 查询
cursor_session_id
  ↓ Cursor Agent
AI 分析结果
  ↓ 格式化
Markdown 文本
  ↓ SeaTalk API
用户收到回复
```

### 6.2 Session 数据流

```
SQLite (data/sessions.db)
  table: sessions
    columns: thread_id, session_id, created_at, updated_at, ...

业务层查询:
  thread_id → session_id

Cursor Agent 持久化:
  ./projects_src/.cursor/sessions/session_abc123.json
    包含: 历史对话、已访问文件、AI推理状态
```

### 6.3 Git 仓库数据流

```
MySQL (项目元信息)
  ↓ 定时导出（每天 09:00）
CSV (project_info.csv)
  ↓ Git 同步服务读取
批量 git clone/pull
  ↓ 持久化
./projects_src/<repo_name>/
  ↓ Cursor Agent 访问
代码分析
```

---

## 七、关键技术决策

### 7.1 为什么选择 Redis List 而不是 Kafka？

**Redis List 的优势（当前场景）**:
- ✅ 轻量级，运维简单
- ✅ 低延迟（<1ms）
- ✅ 支持阻塞拉取（BLPOP/BRPOP）
- ✅ 已有基础设施（分布式锁、缓存）

**Kafka 的优势（大规模场景）**:
- ✅ 更高吞吐（百万级/秒）
- ✅ 持久化更可靠
- ✅ 分区并行消费

**当前消息量**（~5 req/h）: Redis 完全够用，Kafka 是 overkill

**如果未来扩展到 1000 req/h**: 可以无缝切换到 Kafka（代码已预留接口）

### 7.2 为什么用 SQLite 存储 Session？

**SQLite 的优势**:
- ✅ 零配置，单文件数据库
- ✅ 支持并发读
- ✅ 足够的性能（<1ms 查询）
- ✅ 数据持久化（跨重启）

**Redis 的劣势（对于 Session）**:
- ❌ 需要额外配置持久化（RDB/AOF）
- ❌ 占用宝贵的 Redis 内存
- ❌ 过期策略复杂（需要定期清理）

**当前 Session 量**（<1000条）: SQLite 完全够用

### 7.3 为什么每次 AI 分析启动新进程？

**进程隔离的优势**:
- ✅ 稳定性高（崩溃不影响主服务）
- ✅ 资源隔离（内存、CPU 完全独立）
- ✅ 超时控制简单（kill 子进程即可）
- ✅ 自动资源回收（进程退出自动清理）

**常驻进程的劣势**:
- ❌ 内存泄漏风险（长时间运行）
- ❌ 异常传播（一个任务崩溃影响其他）
- ❌ 资源竞争（需要复杂的池管理）

### 7.4 为什么共享工作目录？

**共享的优势**:
- ✅ 支持跨仓库分析（"对比 farm-api 和 admin-gateway"）
- ✅ Git 仓库复用（节省 29% 时间和带宽）
- ✅ Project 元信息共享（project_info.csv）

**隔离机制保证安全**:
- ✅ 进程级隔离（操作系统保证）
- ✅ Session 文件隔离（session_id 命名）
- ✅ 子目录隔离（不同仓库）
- ✅ Git 锁机制（.git/index.lock）

---

## 八、扩展性设计

### 8.1 水平扩展方案

**当前架构支持**:
```
Webhook 适配器（无状态）
  ↓ Nginx 负载均衡
多实例部署（3台机器）
  ↓
Redis（单实例或 Cluster）
  ↓
Algo Bot 消费者（无状态）
  ↓ 共享 Redis 队列
多实例部署（3台机器）
```

**预估能力**:
- 3台机器可支持 100x 流量（当前 ~5 req/h → 500 req/h）

### 8.2 Skills 扩展机制

**添加新 Skill**:
```bash
1. 创建 Skill 目录:
   mkdir -p resources/skills/new-tool/

2. 编写 SKILL.md:
   cat > resources/skills/new-tool/SKILL.md <<EOF
   # New Tool Skill
   
   When user asks about X, you can use this tool...
   EOF

3. cursor-agent 自动发现:
   无需修改代码，AI 自动加载新 Skill
```

**实际案例**:
- `dag_tracer`: 诊断搜索 trace，调用内部诊断脚本
- `confluence-tools`: 查询企业 Wiki 文档

---

## 九、性能指标

### 9.1 关键性能指标

| 指标 | 当前值 | 目标值 |
|------|--------|--------|
| **Webhook 响应时间** | <100ms | <200ms |
| **AI 分析响应时间（P50）** | ~40s | <60s |
| **AI 分析响应时间（P95）** | ~80s | <120s |
| **并发处理能力** | 5个任务 | 5-10个任务 |
| **峰值吞吐** | 10 req/s | 20 req/s |
| **服务可用性** | 99.5%+ | 99.9%+ |

### 9.2 性能优化

**已实现**:
- ✅ Git 仓库复用（节省 29% 时间）
- ✅ 定时预同步（95% 的请求无需 clone）
- ✅ 消息去重（减少 15% 无效处理）
- ✅ ThreadPoolExecutor 并发（5x 吞吐量）

**可优化**:
- 📝 增加 max_workers（10个并发）
- 📝 Redis 队列优先级（重要消息优先）
- 📝 Cursor Agent 结果缓存（相同问题直接返回）

---

## 十、安全设计

### 10.1 多层安全机制

| 安全层 | 实现 | 说明 |
|--------|------|------|
| **签名验证** | SHA256 HMAC | 防止伪造请求 |
| **权限验证** | GitLab Group 成员 | 只有授权用户可访问 |
| **Prompt 约束** | 只读分析 | 禁止修改代码 |
| **消息去重** | message_id | 防止重复处理 |
| **超时过滤** | >5分钟拒绝 | 防止旧消息攻击 |

### 10.2 敏感信息保护

**不记录日志**:
- ❌ `CURSOR_API_KEY`
- ❌ `SEATALK_APP_SECRET`
- ❌ `GITLAB_PRIVATE_TOKEN`
- ❌ Redis/MySQL 密码

**不暴露错误**:
- ✅ 所有 HTTP 异常的 `detail=""` 
- ✅ 详细错误只记录在服务端日志

---

## 十一、故障恢复

### 11.1 容错机制

| 故障场景 | 容错措施 | 恢复时间 |
|----------|----------|----------|
| **Redis 故障** | 持久化（AOF+RDB） | <1分钟 |
| **MySQL 故障** | 已导出的 CSV 可继续使用 | 0秒（降级） |
| **Cursor Agent 崩溃** | 子进程隔离，不影响主服务 | 0秒 |
| **SeaTalk 超时** | 异步处理，不阻塞 | 0秒 |
| **GitLab 故障** | 可配置跳过权限检查 | 0秒（降级） |

### 11.2 监控告警

**关键指标**:
- Redis 队列长度（>100 告警）
- 消息处理失败率（>5% 告警）
- AI 分析超时率（>10% 告警）
- 服务进程存活（down 告警）

---

## 十二、下一步阅读

**建议阅读顺序**:

1. **01-CORE-SERVICES.md**: 核心服务模块详解
   - 消息处理主循环
   - Git 同步服务
   - HTTP 健康检查

2. **02-INTEGRATIONS.md**: 外部集成模块详解
   - Cursor Agent 客户端
   - SeaTalk API 客户端
   - GitLab 权限验证
   - Redis 客户端封装

3. **03-DATA-LAYER.md**: 数据层和仓库模块详解
   - Session 管理（SQLite）
   - 项目信息访问（MySQL）

4. **04-CLI-ENTRYPOINT.md**: CLI入口和启动流程
   - 主入口：algo-bot-agent
   - 同步工具：algo-bot-sync

5. **05-CONFIGURATION.md**: 配置系统详解
   - config.yml 配置文件
   - 环境变量
   - 配置加载流程

6. **06-DEPLOYMENT.md**: 部署和运维文档
   - 本地开发环境搭建
   - 生产环境部署
   - 监控和日志

---

## 附录

### A. 术语表

| 术语 | 说明 |
|------|------|
| **Cursor Agent** | Cursor 官方提供的 CLI 工具，用于在命令行调用 AI 分析代码 |
| **SeaTalk** | 企业即时通讯平台（类似 Slack） |
| **thread_id** | SeaTalk 消息线程 ID（群聊或私聊） |
| **cursor_session_id** | Cursor Agent 会话 ID（AI 上下文标识） |
| **BRPOP** | Redis 命令，阻塞式右侧弹出（FIFO 队列消费） |
| **LPUSH** | Redis 命令，左侧插入（FIFO 队列生产） |
| **Skill** | Cursor Agent 的扩展机制，通过 SKILL.md 文件定义 AI 能力 |

### B. 相关链接

- **Cursor Agent 文档**: https://docs.cursor.com/
- **SeaTalk API 文档**: (企业内部)
- **GitLab API 文档**: https://docs.gitlab.com/ee/api/
- **Redis 文档**: https://redis.io/docs/

### C. 仓库目录与文件索引

本节列出与交付、运行、文档相关的**目录与文件**完整清单，便于导航与审计。

**不纳入本索引的内容**（按约定省略，不逐一展开）：

- `tests/` 及其中全部测试文件
- `.gitignore`、`.git/` 版本库目录
- Python 缓存与虚拟环境：`__pycache__/`、`*.pyc`、`.venv/`、`.pytest_cache/` 等
- 编辑器与本地工具产生的目录（若存在）

#### C.1 目录树（仅目录）

```
algo-bot/
├── config/
├── deploy/
├── docs/
├── examples/
├── resources/
│   └── skills/
│       ├── abtest-assistant/
│       │   ├── references/
│       │   └── scripts/
│       ├── confluence-tools/
│       │   └── scripts/
│       └── dag_tracer/
│           ├── references/
│           └── scripts/
├── scripts/
├── sql/
└── src/
    └── algo_bot/
        ├── cli/
        ├── integrations/
        ├── repositories/
        └── services/
```

#### C.2 文件清单（按路径）

**仓库根目录**

| 文件 | 说明 |
|------|------|
| `README.md` | 项目说明与快速开始 |
| `pyproject.toml` | 包元数据与依赖声明 |
| `uv.lock` | uv 依赖锁定文件 |
| `.env.example` | 环境变量模板 |

**`config/`**

| 文件 | 说明 |
|------|------|
| `config.yml` | 应用主配置（服务、Redis、GitLab 等） |

**`deploy/`**

| 文件 | 说明 |
|------|------|
| `deploy.json` | 部署相关元数据/配置 |

**`docs/`**

| 文件 | 说明 |
|------|------|
| `README.md` | 文档索引说明 |
| `00-ARCHITECTURE.md` | 系统架构（本文档） |
| `01-CORE-SERVICES.md` | 核心服务详解 |
| `02-INTEGRATIONS.md` | 外部集成详解 |
| `03-DATA-LAYER.md` | 数据层与仓库 |
| `04-CLI-ENTRYPOINT.md` | CLI 与启动流程 |
| `05-CONFIGURATION.md` | 配置系统 |
| `06-DEPLOYMENT.md` | 部署与运维 |

**`examples/`**

| 文件 | 说明 |
|------|------|
| `project_info.csv` | 项目信息示例数据 |

**`resources/`**

| 文件 | 说明 |
|------|------|
| `prompt.txt` | 提示词/系统提示等资源文本 |

**`resources/skills/abtest-assistant/`**

| 文件 | 说明 |
|------|------|
| `SKILL.md` | Skill 定义 |
| `references/abtest_demo_outputs.md` | 参考文档 |
| `references/abtest_query_outputs.md` | 参考文档 |
| `references/change_patterns.md` | 参考文档 |
| `references/workflow.md` | 参考文档 |
| `scripts/abtest_dag_validate.py` | 脚本 |
| `scripts/abtest_query.py` | 脚本 |

**`resources/skills/confluence-tools/`**

| 文件 | 说明 |
|------|------|
| `SKILL.md` | Skill 定义 |
| `scripts/confluence_tool.py` | Confluence 工具脚本 |

**`resources/skills/dag_tracer/`**

| 文件 | 说明 |
|------|------|
| `SKILL.md` | Skill 定义 |
| `references/request_retry_example.json` | 示例数据 |
| `references/sample_diagnostic_reports.md` | 参考文档 |
| `scripts/diagnose_search_trace.py` | 诊断脚本 |

**`scripts/`**

| 文件 | 说明 |
|------|------|
| `start.sh` | 启动脚本 |
| `status.sh` | 状态检查脚本 |
| `sync_once.sh` | 单次同步脚本 |

**`sql/`**

| 文件 | 说明 |
|------|------|
| `create_project_info_table.sql` | 项目信息表 DDL |

**`src/algo_bot/`**（Python 包根）

| 文件 | 说明 |
|------|------|
| `__init__.py` | 包初始化 |
| `env_loader.py` | 环境变量加载 |
| `logging.py` | 日志配置 |
| `settings.py` | 设置与配置对象 |

**`src/algo_bot/cli/`**

| 文件 | 说明 |
|------|------|
| `__init__.py` | 子包初始化 |
| `algo_bot.py` | CLI 主入口与命令实现 |
| `sync_once.py` | 同步 CLI 入口 |

**`src/algo_bot/integrations/`**

| 文件 | 说明 |
|------|------|
| `__init__.py` | 子包初始化 |
| `cursor_agent.py` | Cursor Agent 集成 |
| `gitlab_auth.py` | GitLab 鉴权 |
| `redis_client.py` | Redis 客户端 |
| `seatalk_api.py` | SeaTalk API 客户端 |

**`src/algo_bot/repositories/`**

| 文件 | 说明 |
|------|------|
| `__init__.py` | 子包初始化 |
| `project_info_repository.py` | 项目信息持久化 |
| `session_repository.py` | 会话持久化 |

**`src/algo_bot/services/`**

| 文件 | 说明 |
|------|------|
| `__init__.py` | 子包初始化 |
| `health.py` | 健康检查 |
| `http_server.py` | HTTP 服务（如 Webhook 适配） |
| `project_sync_service.py` | 项目同步服务 |
| `projects_git_sync.py` | 多仓库 Git 同步 |
| `seatalk_callback.py` | SeaTalk 回调与消息消费主循环 |

> **维护提示**：新增或移动目录/文件后，请同步更新本节，以保持与仓库一致。

---

**文档版本**: v1.0  
**最后更新**: 2026-03-30  
**维护者**: Algo Bot 团队
