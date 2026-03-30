# Algo Bot 完整文档索引

## 文档概述

本目录包含 Algo Bot 项目的完整技术文档,旨在帮助其他 AI 或开发者完全理解和复现这个项目。

## 文档结构

### 核心文档（按阅读顺序）

1. **[00-ARCHITECTURE.md](./00-ARCHITECTURE.md)** - 系统架构文档
   - 项目定位和核心特性
   - 整体架构和设计理念
   - 技术栈和目录结构
   - 核心流程和数据流
   - 关键技术决策

2. **[01-CORE-SERVICES.md](./01-CORE-SERVICES.md)** - 核心服务模块详解
   - SeaTalk 回调工具函数
   - Git 仓库同步服务
   - 项目同步调度服务
   - HTTP 健康检查服务

3. **[02-INTEGRATIONS.md](./02-INTEGRATIONS.md)** - 外部集成模块详解
   - Cursor Agent CLI 客户端
   - SeaTalk OpenAPI 客户端
   - GitLab 权限验证
   - Redis 客户端封装

4. **[03-DATA-LAYER.md](./03-DATA-LAYER.md)** - 数据层和仓库模块详解
   - Session 会话管理（SQLite）
   - 项目信息访问（MySQL）
   - 数据流和并发控制

5. **[04-CLI-ENTRYPOINT.md](./04-CLI-ENTRYPOINT.md)** - CLI入口和启动流程
   - 主服务入口（algo-bot-agent）
   - 消息处理主循环
   - 同步工具（algo-bot-sync）
   - 环境变量加载

6. **[05-CONFIGURATION.md](./05-CONFIGURATION.md)** - 配置系统详解
   - 配置文件格式（config.yml）
   - 所有配置项详细说明
   - 环境变量规则
   - 配置最佳实践

7. **[06-DEPLOYMENT.md](./06-DEPLOYMENT.md)** - 部署和运维文档
   - 本地开发环境搭建
   - 生产环境部署（Systemd/Docker）
   - 监控和日志
   - 备份和恢复
   - 故障排查

---

## 快速导航

### 按角色阅读

**AI/复现者（完整理解）**:
1. 00-ARCHITECTURE.md（理解系统架构）
2. 01-CORE-SERVICES.md（理解核心服务）
3. 02-INTEGRATIONS.md（理解外部集成）
4. 03-DATA-LAYER.md（理解数据层）
5. 04-CLI-ENTRYPOINT.md（理解启动流程）
6. 05-CONFIGURATION.md（理解配置系统）
7. 06-DEPLOYMENT.md（理解部署流程）

**开发者（快速上手）**:
1. 00-ARCHITECTURE.md（了解架构）
2. 06-DEPLOYMENT.md（本地环境搭建）
3. 05-CONFIGURATION.md（配置说明）
4. 04-CLI-ENTRYPOINT.md（启动和调试）

**运维人员（部署维护）**:
1. 00-ARCHITECTURE.md（了解系统）
2. 05-CONFIGURATION.md（配置管理）
3. 06-DEPLOYMENT.md（部署和监控）

**架构师（系统设计）**:
1. 00-ARCHITECTURE.md（整体架构）
2. 01-CORE-SERVICES.md（服务设计）
3. 02-INTEGRATIONS.md（集成设计）
4. 03-DATA-LAYER.md（数据设计）

### 按主题查找

**架构设计**:
- 00-ARCHITECTURE.md - 整体架构
- 00-ARCHITECTURE.md - 核心设计理念
- 00-ARCHITECTURE.md - 技术选型

**服务实现**:
- 01-CORE-SERVICES.md - 消息处理
- 01-CORE-SERVICES.md - Git 同步
- 02-INTEGRATIONS.md - Cursor Agent 调用
- 02-INTEGRATIONS.md - SeaTalk API

**数据管理**:
- 03-DATA-LAYER.md - Session 管理
- 03-DATA-LAYER.md - 项目信息访问
- 03-DATA-LAYER.md - 并发控制

**配置和部署**:
- 05-CONFIGURATION.md - 配置文件
- 05-CONFIGURATION.md - 环境变量
- 06-DEPLOYMENT.md - 本地开发
- 06-DEPLOYMENT.md - 生产部署

**运维和故障排查**:
- 06-DEPLOYMENT.md - 监控和日志
- 06-DEPLOYMENT.md - 备份和恢复
- 06-DEPLOYMENT.md - 故障排查

---

## 文档特点

### 1. 完整性

每个文档都包含:
- ✅ 模块职责说明
- ✅ 核心代码示例
- ✅ 数据结构定义
- ✅ 使用示例
- ✅ 常见问题和排查

### 2. 可复现性

文档提供:
- ✅ 完整的技术栈说明
- ✅ 详细的配置步骤
- ✅ 可运行的代码示例
- ✅ 清晰的数据流图

### 3. 实用性

文档包含:
- ✅ 最佳实践建议
- ✅ 故障排查指南
- ✅ 性能优化建议
- ✅ 安全加固措施

---

## 技术栈总览

### 核心技术

| 技术 | 版本 | 用途 |
|------|------|------|
| Python | ≥3.11 | 主要开发语言 |
| FastAPI | ≥0.115.0 | Web框架（可选Webhook适配器） |
| Redis | ≥5.0.0 | 消息队列、分布式锁 |
| SQLite | - | Session会话管理 |
| MySQL | ≥5.7 | 项目信息存储 |
| cursor-agent | latest | AI代码分析引擎 |
| SeaTalk | - | 即时通讯平台 |

### Python 依赖

```toml
[project]
dependencies = [
    "redis>=5.0.0",
    "pyyaml>=6.0",
    "requests>=2.31.0",
    "python-dotenv>=1.0.0",
    "fastapi>=0.115.0",
    "uvicorn>=0.30.0",
    "pymysql>=1.1.0",
    "cryptography>=42.0.0",
    "schedule>=1.2.0",
]
```

---

## 关键概念

### 1. 双层 Session 映射

```
业务层: thread_id (SeaTalk线程)
  ↓ SQLite持久化映射
AI层: cursor_session_id (AI会话)
```

### 2. 共享工作目录 + 多层隔离

```
./projects_src/  # 所有进程共享
├── .cursor/sessions/  # Session文件隔离
├── farm-api/          # 子目录隔离
└── admin-gateway/     # 子目录隔离
```

### 3. 异步非阻塞架构

```
Webhook → [验证 <100ms] → 响应 200 ✅
   ↓
Redis 队列
   ↓
异步处理 [30-120s]
```

### 4. 进程隔离

```
每次 AI 分析 = 独立 subprocess
  ↓
子进程崩溃 ≠ 主服务崩溃
```

---

## 核心数据流

### 消息处理流程

```
SeaTalk 用户消息
  ↓ Webhook
Webhook 适配器（签名验证）
  ↓ LPUSH
Redis 队列
  ↓ BRPOP
消息处理服务
  ↓ 权限验证
GitLab Auth
  ↓ Session查询
SQLite
  ↓ 调用AI
cursor-agent 子进程
  ↓ 结果发送
SeaTalk API
```

### Git 同步流程

```
MySQL 项目信息
  ↓ 定时导出
CSV 文件
  ↓ 批量同步
Git clone/pull
  ↓ 持久化
./projects_src/<repos>/
  ↓ AI访问
cursor-agent 代码分析
```

---

## 项目规模

- **代码量**: ~5000 行 Python
- **服务数**: 2 个（Webhook适配器 + 业务服务）
- **数据库**: SQLite + MySQL
- **支持项目**: 50+ 个 Git 仓库
- **并发能力**: 5 个并发任务
- **服务可用性**: 99.5%+

---

## 性能指标

| 指标 | 当前值 |
|------|--------|
| Webhook 响应时间 | <100ms |
| AI 分析响应时间（P50） | ~40s |
| AI 分析响应时间（P95） | ~80s |
| 并发处理能力 | 5个任务 |
| 峰值吞吐 | 10 req/s |
| Git 仓库复用节省 | 29% |

---

## 安全特性

- ✅ SHA256 HMAC 签名验证
- ✅ GitLab Group 权限控制
- ✅ 消息去重（message_id）
- ✅ 超时消息过滤（>5分钟）
- ✅ Prompt工程约束（只读分析）
- ✅ 敏感信息保护（日志脱敏）

---

## 扩展性

### 水平扩展

```
3台机器 → 支持 100x 流量
（当前 ~5 req/h → 500 req/h）
```

### Skills 扩展

```bash
# 添加新 Skill（无需修改代码）
mkdir -p resources/skills/new-tool/
cat > resources/skills/new-tool/SKILL.md <<EOF
# New Tool Skill
...
EOF
# cursor-agent 自动发现
```

---

## 未来优化方向

### 性能优化
- 📝 增加 max_workers（10个并发）
- 📝 Redis 队列优先级
- 📝 Cursor Agent 结果缓存

### 功能扩展
- 📝 支持更多事件类型
- 📝 支持其他消息队列（Kafka）
- 📝 支持消息预处理

### 运维优化
- 📝 Prometheus 监控集成
- 📝 分布式链路追踪
- 📝 自动化部署脚本

---

## 相关资源

### 仓库链接
- **项目仓库**: 链接待补充
- **Issues**: 链接待补充

### 外部文档
- **Cursor Agent 文档**: https://docs.cursor.com/
- **SeaTalk API 文档**: (企业内部)
- **GitLab API 文档**: https://docs.gitlab.com/ee/api/
- **Redis 文档**: https://redis.io/docs/

### 社区支持
- **讨论组**: 链接待补充
- **Wiki**: 链接待补充

---

## 文档维护

### 版本历史

| 版本 | 日期 | 说明 |
|------|------|------|
| v1.0 | 2026-03-30 | 初始版本，完整文档体系 |

### 维护者

- Algo Bot 团队

### 贡献指南

如需更新文档，请:
1. Fork 项目
2. 创建文档分支
3. 提交 Pull Request
4. 等待 Review

---

## 最后

希望这套文档能够帮助你完全理解和复现 Algo Bot 项目。

如有任何问题，欢迎:
- 查看其他文档
- 提交 Issue
- 联系维护者

**祝你学习愉快！** 🎉
