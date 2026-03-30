# 部署和运维文档

## 文档概述

本文档详细介绍 Algo Bot 的部署流程、运维指南和常见问题解决方案。

**前置阅读**: [00-ARCHITECTURE.md](./00-ARCHITECTURE.md), [05-CONFIGURATION.md](./05-CONFIGURATION.md)

---

## 一、环境准备

### 1.1 系统要求

| 组件 | 要求 |
|------|------|
| **操作系统** | Linux (推荐 Ubuntu 20.04+) 或 macOS |
| **Python** | ≥ 3.11 |
| **Redis** | ≥ 5.0.0 |
| **MySQL** | ≥ 5.7 或 8.0 |
| **cursor-agent** | latest |
| **磁盘空间** | ≥ 50GB（用于 Git 仓库） |
| **内存** | ≥ 4GB |

### 1.2 依赖服务

**必需服务**:
- Redis（消息队列）
- MySQL（项目信息存储）
- GitLab（可选，用于权限验证）
- SeaTalk（IM 平台）

**网络要求**:
- 能够访问 SeaTalk OpenAPI
- 能够访问 GitLab API
- 能够 clone Git 仓库（SSH 或 HTTPS）

---

## 二、本地开发环境搭建

### 2.1 克隆项目

```bash
git clone https://github.com/your-org/algo-bot.git
cd algo-bot
```

### 2.2 安装依赖

```bash
# 1. 安装 Python 3.11+
python3 --version  # 确认版本 >= 3.11

# 2. 安装 uv（推荐）
curl -LsSf https://astral.sh/uv/install.sh | sh

# 3. 安装项目依赖
uv sync

# 4. 安装 cursor-agent
# 请参考 Cursor 官方文档
```

### 2.3 配置服务

**Redis**:
```bash
# macOS
brew install redis
brew services start redis

# Ubuntu
sudo apt install redis-server
sudo systemctl start redis
```

**MySQL**:
```bash
# macOS
brew install mysql
brew services start mysql

# Ubuntu
sudo apt install mysql-server
sudo systemctl start mysql

# 创建数据库
mysql -u root -p
CREATE DATABASE testdb CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

### 2.4 配置文件

**复制示例配置**:
```bash
cp .env.example .env
cp config/config.example.yml config/config.yml
```

**编辑 .env**:
```bash
vim .env
```

```bash
# Cursor API
CURSOR_API_KEY=cursor_api_key_***

# SeaTalk API
SEATALK_APP_ID=your_app_id
SEATALK_APP_SECRET=your_app_secret

# Redis
REDIS_HOST=localhost
REDIS_PORT=6379

# MySQL
MYSQL_HOST=localhost
MYSQL_PORT=3306
MYSQL_USER=root
MYSQL_PASSWORD=your_password
MYSQL_DATABASE=testdb
```

**编辑 config/config.yml**:
```bash
vim config/config.yml
```

```yaml
server:
  host: 127.0.0.1
  port: 8080

cursor_agent:
  agent_workdir: "./projects_src"
  agent_timeout: 180

# ... 其他配置
```

### 2.5 初始化数据库

```bash
# 创建项目信息表
uv run python -m algo_bot.repositories.project_info_repository create_table
```

### 2.6 创建工作目录

```bash
# 创建必要的目录
mkdir -p projects_src
mkdir -p projects_src/.cursor
mkdir -p data
mkdir -p log
mkdir -p pids

# 创建 Skills 软链
ln -sfn ../../resources/skills projects_src/.cursor/skills
```

### 2.7 启动服务

```bash
# 启动主服务
uv run algo-bot-agent

# 查看日志
tail -f log/agent.log
```

---

## 三、生产环境部署

### 3.1 部署架构

```
┌─────────────────────────────────────────────────┐
│              Nginx (负载均衡)                     │
└─────────────────────┬───────────────────────────┘
                      │
        ┌─────────────┼─────────────┐
        │             │             │
        ↓             ↓             ↓
┌───────────┐ ┌───────────┐ ┌───────────┐
│ Algo Bot  │ │ Algo Bot  │ │ Algo Bot  │
│ Instance1 │ │ Instance2 │ │ Instance3 │
└─────┬─────┘ └─────┬─────┘ └─────┬─────┘
      │             │             │
      └─────────────┼─────────────┘
                    │
        ┌───────────┼───────────┐
        │           │           │
        ↓           ↓           ↓
┌───────────┐ ┌──────────┐ ┌──────────┐
│  Redis    │ │  MySQL   │ │ GitLab   │
└───────────┘ └──────────┘ └──────────┘
```

### 3.2 使用 Systemd

**创建服务文件**:

```bash
sudo vim /etc/systemd/system/algo-bot.service
```

```ini
[Unit]
Description=Algo Bot Service
After=network.target redis.service mysql.service

[Service]
Type=simple
User=algo-bot
Group=algo-bot
WorkingDirectory=/opt/algo-bot
Environment="PATH=/opt/algo-bot/.venv/bin:/usr/local/bin:/usr/bin:/bin"
Environment="CONFIG_FILE=/opt/algo-bot/config/production.yml"
ExecStart=/opt/algo-bot/.venv/bin/algo-bot-agent
Restart=always
RestartSec=10
StandardOutput=append:/var/log/algo-bot/stdout.log
StandardError=append:/var/log/algo-bot/stderr.log

[Install]
WantedBy=multi-user.target
```

**启动服务**:

```bash
# 重载 systemd 配置
sudo systemctl daemon-reload

# 启动服务
sudo systemctl start algo-bot

# 查看状态
sudo systemctl status algo-bot

# 查看日志
sudo journalctl -u algo-bot -f

# 开机自启
sudo systemctl enable algo-bot
```

### 3.3 使用 Docker

**Dockerfile**:

```dockerfile
FROM python:3.11-slim

# 安装系统依赖
RUN apt-get update && apt-get install -y \
    git \
    curl \
    && rm -rf /var/lib/apt/lists/*

# 安装 uv
RUN curl -LsSf https://astral.sh/uv/install.sh | sh
ENV PATH="/root/.cargo/bin:$PATH"

# 安装 cursor-agent
RUN curl -fsSL https://install.cursor.com | sh

# 创建工作目录
WORKDIR /app

# 复制项目文件
COPY . /app

# 安装 Python 依赖
RUN uv sync --frozen

# 创建必要的目录
RUN mkdir -p projects_src data log pids

# 创建 Skills 软链
RUN ln -sfn ../../resources/skills projects_src/.cursor/skills

# 暴露端口
EXPOSE 8080

# 启动服务
CMD ["uv", "run", "algo-bot-agent"]
```

**docker-compose.yml**:

```yaml
version: '3.8'

services:
  algo-bot:
    build: .
    container_name: algo-bot
    restart: unless-stopped
    ports:
      - "8080:8080"
    environment:
      - CONFIG_FILE=/app/config/production.yml
      - CURSOR_API_KEY=${CURSOR_API_KEY}
      - REDIS_HOST=redis
      - MYSQL_HOST=mysql
    volumes:
      - ./config:/app/config
      - ./projects_src:/app/projects_src
      - ./data:/app/data
      - ./log:/app/log
    depends_on:
      - redis
      - mysql
    networks:
      - algo-bot-network

  redis:
    image: redis:7-alpine
    container_name: algo-bot-redis
    restart: unless-stopped
    ports:
      - "6379:6379"
    volumes:
      - redis-data:/data
    networks:
      - algo-bot-network

  mysql:
    image: mysql:8.0
    container_name: algo-bot-mysql
    restart: unless-stopped
    ports:
      - "3306:3306"
    environment:
      - MYSQL_ROOT_PASSWORD=${MYSQL_ROOT_PASSWORD}
      - MYSQL_DATABASE=testdb
    volumes:
      - mysql-data:/var/lib/mysql
    networks:
      - algo-bot-network

volumes:
  redis-data:
  mysql-data:

networks:
  algo-bot-network:
```

**启动容器**:

```bash
# 启动所有服务
docker-compose up -d

# 查看日志
docker-compose logs -f algo-bot

# 停止服务
docker-compose down
```

---

## 四、监控和日志

### 4.1 日志文件

```bash
log/
├── agent.log              # 主服务日志
└── sync_projects.log      # Git 同步日志
```

**查看日志**:

```bash
# 实时查看
tail -f log/agent.log

# 查看最近 100 行
tail -n 100 log/agent.log

# 搜索错误
grep ERROR log/agent.log

# 按时间范围查看
awk '/2024-03-30 10:00/,/2024-03-30 11:00/' log/agent.log
```

### 4.2 监控指标

**关键指标**:

| 指标 | 说明 | 告警阈值 |
|------|------|----------|
| **Redis 队列长度** | `LLEN algo:bot:seatalk:list` | > 100 |
| **消息处理成功率** | 成功 / 总数 | < 95% |
| **AI 分析超时率** | 超时 / 总数 | > 10% |
| **磁盘使用率** | projects_src 目录大小 | > 80% |
| **内存使用率** | 进程内存占用 | > 90% |
| **CPU 使用率** | 进程 CPU 占用 | > 80% |

**监控脚本示例**:

```bash
#!/bin/bash
# monitor.sh

# Redis 队列长度
QUEUE_LEN=$(redis-cli LLEN algo:bot:seatalk:list)
if [ $QUEUE_LEN -gt 100 ]; then
    echo "告警: Redis 队列积压 $QUEUE_LEN 条消息"
fi

# 进程存活检查
if ! pgrep -f algo-bot-agent > /dev/null; then
    echo "告警: Algo Bot 进程未运行"
fi

# 磁盘空间检查
DISK_USAGE=$(df -h projects_src | tail -1 | awk '{print $5}' | sed 's/%//')
if [ $DISK_USAGE -gt 80 ]; then
    echo "告警: 磁盘使用率 $DISK_USAGE%"
fi
```

**定时执行**:

```bash
# 每 5 分钟检查一次
*/5 * * * * /path/to/monitor.sh >> /var/log/algo-bot/monitor.log 2>&1
```

### 4.3 性能分析

**查看线程数**:

```bash
# 查看进程 PID
ps aux | grep algo-bot-agent

# 查看线程
ps -T -p <PID>

# 查看线程数
ps -T -p <PID> | wc -l
```

**查看资源占用**:

```bash
# 实时监控
top -p <PID>

# 详细信息
htop -p <PID>
```

---

## 五、备份和恢复

### 5.1 SQLite 备份

```bash
# 备份 Session 数据库
cp data/sessions.db data/sessions.db.backup.$(date +%Y%m%d)

# 定期备份（每天凌晨 2 点）
0 2 * * * cp data/sessions.db data/sessions.db.backup.$(date +\%Y\%m\%d)

# 清理旧备份（保留 30 天）
find data/ -name "sessions.db.backup.*" -mtime +30 -delete
```

### 5.2 MySQL 备份

```bash
# 备份项目信息表
mysqldump -u root -p testdb algo_bot_cursor_projects_tab > backup/projects_$(date +%Y%m%d).sql

# 定期备份（每天凌晨 3 点）
0 3 * * * mysqldump -u root -p testdb algo_bot_cursor_projects_tab > /backup/projects_$(date +\%Y\%m\%d).sql

# 清理旧备份
find /backup/ -name "projects_*.sql" -mtime +30 -delete
```

### 5.3 Git 仓库备份

```bash
# 方法1: 打包整个工作目录
tar -czf projects_src_backup_$(date +%Y%m%d).tar.gz projects_src/

# 方法2: 只备份 project_info.csv
cp projects_src/project_info.csv backup/project_info_$(date +%Y%m%d).csv
```

---

## 六、故障排查

### 6.1 服务无法启动

**症状**: `systemctl start algo-bot` 失败

**排查步骤**:

1. 查看 systemd 日志:
```bash
sudo journalctl -u algo-bot -n 50
```

2. 检查配置文件:
```bash
uv run python -c "from algo_bot.settings import Config; Config.load_config(); Config.print_config()"
```

3. 检查依赖服务:
```bash
# Redis
redis-cli ping

# MySQL
mysql -u root -p -e "SHOW DATABASES;"

# cursor-agent
cursor-agent --version
```

4. 检查权限:
```bash
ls -la /opt/algo-bot
ls -la /var/log/algo-bot
```

### 6.2 消息处理失败

**症状**: 用户发消息，AI 无响应

**排查步骤**:

1. 检查 Redis 队列:
```bash
redis-cli LLEN algo:bot:seatalk:list
redis-cli LINDEX algo:bot:seatalk:list 0
```

2. 检查日志:
```bash
grep ERROR log/agent.log | tail -20
```

3. 检查线程池:
```bash
ps -T -p $(pgrep -f algo-bot-agent)
```

4. 手动测试 Cursor Agent:
```bash
cd projects_src
cursor-agent -p "测试消息"
```

### 6.3 Git 同步失败

**症状**: 项目代码未更新

**排查步骤**:

1. 检查同步日志:
```bash
tail -f log/sync_projects.log
```

2. 手动测试 Git 操作:
```bash
cd projects_src/farm-api
git fetch origin
git pull origin release
```

3. 检查 Git 凭据:
```bash
# SSH
ssh -T git@git.example.com

# HTTPS (需要 Git token)
git config --global credential.helper store
```

4. 手动触发同步:
```bash
uv run algo-bot-sync
```

### 6.4 内存泄漏

**症状**: 进程内存持续增长

**排查步骤**:

1. 监控内存:
```bash
watch -n 5 'ps -p <PID> -o pid,vsz,rss,cmd'
```

2. 检查会话数量:
```bash
uv run algo-bot-agent --list-sessions
```

3. 清理过期会话:
```bash
uv run algo-bot-agent --cleanup-sessions
```

4. 重启服务:
```bash
sudo systemctl restart algo-bot
```

---

## 七、升级和迁移

### 7.1 升级流程

```bash
# 1. 备份数据
cp data/sessions.db data/sessions.db.backup
mysqldump -u root -p testdb algo_bot_cursor_projects_tab > backup/projects_backup.sql

# 2. 拉取最新代码
git pull origin main

# 3. 更新依赖
uv sync --frozen

# 4. 检查配置变更
git diff HEAD~1 config/config.example.yml

# 5. 重启服务
sudo systemctl restart algo-bot

# 6. 验证服务
curl http://localhost:8080/health
```

### 7.2 数据迁移

**从旧版本迁移 Session 数据**:

```python
#!/usr/bin/env python3
# migrate_sessions.py

import sqlite3
import json

# 连接旧数据库
old_db = sqlite3.connect("old_sessions.db")
old_cursor = old_db.cursor()

# 连接新数据库
new_db = sqlite3.connect("data/sessions.db")
new_cursor = new_db.cursor()

# 迁移数据
old_cursor.execute("SELECT * FROM sessions")
for row in old_cursor.fetchall():
    thread_id, session_id, created_at, updated_at = row
    new_cursor.execute(
        """INSERT INTO sessions 
        (thread_id, session_id, created_at, updated_at, 
         last_message_id, processed_message_ids, extra_data)
        VALUES (?, ?, ?, ?, ?, ?, ?)""",
        (thread_id, session_id, created_at, updated_at, 
         None, "[]", "{}")
    )

new_db.commit()
old_db.close()
new_db.close()

print("迁移完成")
```

---

## 八、安全加固

### 8.1 敏感信息保护

1. **不要提交敏感信息到版本控制**:
```bash
# .gitignore
.env
.env.*
config/production.yml
*.key
*.pem
```

2. **使用环境变量**:
```bash
# 不要硬编码在代码中
export CURSOR_API_KEY=***
export MYSQL_PASSWORD=***
```

3. **文件权限**:
```bash
chmod 600 .env
chmod 600 config/production.yml
```

### 8.2 网络安全

1. **Redis 密码保护**:
```bash
# redis.conf
requirepass your_redis_password
```

2. **MySQL 密码强度**:
```sql
ALTER USER 'root'@'localhost' IDENTIFIED BY 'Strong_P@ssw0rd_123';
```

3. **防火墙规则**:
```bash
# 只允许本地访问 Redis
ufw allow from 127.0.0.1 to any port 6379

# 只允许特定 IP 访问 MySQL
ufw allow from 10.0.0.0/8 to any port 3306
```

---

## 九、性能优化

### 9.1 增加并发

```yaml
# config.yml
project_sync:
  max_workers: 5  # 增加到 5 个并发
```

### 9.2 优化 Git 同步

```bash
# 使用 Git token（HTTPS）
git config --global credential.helper store
echo "https://username:token@git.example.com" > ~/.git-credentials

# 使用 SSH（更快）
ssh-keygen -t ed25519 -C "algo-bot@example.com"
# 添加公钥到 GitLab
```

### 9.3 Redis 优化

```bash
# redis.conf
maxmemory 2gb
maxmemory-policy allkeys-lru
save 900 1
save 300 10
save 60 10000
```

---

## 十、总结

### 10.1 部署检查清单

- [ ] Python 3.11+ 已安装
- [ ] Redis 已安装并运行
- [ ] MySQL 已安装并运行
- [ ] cursor-agent 已安装
- [ ] 配置文件已创建（config.yml）
- [ ] 环境变量已设置（.env）
- [ ] 数据库表已创建
- [ ] 工作目录已创建（projects_src）
- [ ] Skills 软链已创建
- [ ] 服务已启动
- [ ] 健康检查通过（/health）
- [ ] 日志记录正常
- [ ] 监控已配置
- [ ] 备份已配置

### 10.2 日常运维任务

- [ ] 每天检查日志
- [ ] 每周清理过期会话
- [ ] 每月检查磁盘空间
- [ ] 每月更新依赖
- [ ] 每季度备份数据

---

**恭喜！您已完成 Algo Bot 的部署和配置。**

如有问题，请查看:
- [00-ARCHITECTURE.md](./00-ARCHITECTURE.md) - 系统架构
- [05-CONFIGURATION.md](./05-CONFIGURATION.md) - 配置详解
- GitHub Issues: https://github.com/your-org/algo-bot/issues
