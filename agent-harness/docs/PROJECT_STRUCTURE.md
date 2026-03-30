# 项目目录结构（复现用）

本文档列出复现 **an-agent** 所需的**源码与配置型文件**目录树。生成规则：

- **已排除**：`.git/`、`.venv/`、`__pycache__/`、`.pytest_cache/`、`.ruff_cache/`、`.mypy_cache/`、`.DS_Store`、`*.pyc`。
- **未列入树内（需本地自建）**：`.env.development` / `.env.staging` 等环境文件（含密钥，勿提交；复现时按 README 从示例模板复制填写）。
- **运行时产物（可选）**：`logs/*.jsonl`、仓库根目录 `*.log` 可随时删除，**不属于**复现所必需源文件。

与 [README.md](../README.md) 对照：文档中曾提及根目录 `prometheus/`、顶层 `skills/`，**当前仓库快照中不存在**；监控以 `grafana/dashboards/` 与代码内 Prometheus 挂载为准。

---

## 目录树

```
an-agent/
├── app/
│   ├── api/
│   │   ├── v1/
│   │   │   ├── api.py
│   │   │   ├── auth.py
│   │   │   └── chatbot.py
│   │   └── exceptions.py
│   ├── core/
│   │   ├── agents/
│   │   │   ├── main/
│   │   │   │   ├── __init__.py
│   │   │   │   └── agent.py
│   │   │   ├── prompts/
│   │   │   │   └── main.md
│   │   │   ├── __init__.py
│   │   │   ├── base.py
│   │   │   └── registry.py
│   │   ├── checkpoint/
│   │   │   ├── __init__.py
│   │   │   └── mysql_saver.py
│   │   ├── skills/
│   │   │   ├── __init__.py
│   │   │   ├── base.py
│   │   │   ├── intent_router.py
│   │   │   └── registry.py
│   │   ├── tools/
│   │   │   ├── mcp/
│   │   │   │   ├── __init__.py
│   │   │   │   └── client.py
│   │   │   ├── __init__.py
│   │   │   └── registry.py
│   │   ├── app_factory.py
│   │   ├── config.py
│   │   ├── limiter.py
│   │   ├── logging.py
│   │   ├── metrics.py
│   │   ├── middleware.py
│   │   └── startup.py
│   ├── data/
│   │   └── idiom.json
│   ├── models/
│   │   ├── base.py
│   │   ├── session.py
│   │   └── user.py
│   ├── plugins/
│   │   ├── agents/
│   │   │   ├── an_debug/
│   │   │   │   ├── __init__.py
│   │   │   │   └── agent.py
│   │   │   ├── prompts/
│   │   │   │   ├── an_debug.md
│   │   │   │   └── skill_runner.md
│   │   │   └── __init__.py
│   │   ├── skills/
│   │   │   ├── dag_tracer/
│   │   │   │   ├── references/
│   │   │   │   │   ├── sample_output_funnel.json
│   │   │   │   │   └── sample_output_items.json
│   │   │   │   ├── scripts/
│   │   │   │   │   └── diagnose_search_trace.py
│   │   │   │   ├── tools/
│   │   │   │   │   └── dag_trace_tool.py
│   │   │   │   └── SKILL.md
│   │   │   ├── item_analysis/
│   │   │   │   ├── references/
│   │   │   │   ├── scripts/
│   │   │   │   │   └── item_quality_analyzer.py
│   │   │   │   ├── tools/
│   │   │   │   │   └── item_quality_analyzer.py
│   │   │   │   └── SKILL.md
│   │   │   └── __init__.py
│   │   ├── tools/
│   │   │   ├── diagnostic_tools_portable/
│   │   │   │   ├── core/
│   │   │   │   │   ├── __init__.py
│   │   │   │   │   ├── constants.py
│   │   │   │   │   ├── debug_tools.py
│   │   │   │   │   ├── helpers.py
│   │   │   │   │   └── models.py
│   │   │   │   ├── diagnose_item_missing.py
│   │   │   │   ├── diagnose_ranking_effect.py
│   │   │   │   └── README.md
│   │   │   ├── itemInfo/
│   │   │   │   ├── __init__.py
│   │   │   │   ├── analysis.py
│   │   │   │   ├── CONFIG.md
│   │   │   │   ├── debug_api.py
│   │   │   │   ├── itemInfo.py
│   │   │   │   └── test_itemInfo.py
│   │   │   ├── __init__.py
│   │   │   ├── calculator.py
│   │   │   ├── diagnostic_tools.py
│   │   │   ├── duckduckgo_search.py
│   │   │   ├── http_request.py
│   │   │   ├── idiom_chain.py
│   │   │   ├── item_quality_analyzer.py
│   │   │   ├── kb_query.py
│   │   │   └── python_exec.py
│   │   └── __init__.py
│   ├── schemas/
│   │   ├── __init__.py
│   │   ├── auth.py
│   │   ├── chat.py
│   │   ├── graph.py
│   │   ├── interrupt.py
│   │   └── skill.py
│   ├── services/
│   │   ├── __init__.py
│   │   ├── checkpoint.py
│   │   ├── database.py
│   │   ├── llm.py
│   │   └── memory.py
│   ├── utils/
│   │   ├── __init__.py
│   │   ├── auth.py
│   │   ├── graph.py
│   │   └── sanitization.py
│   ├── __init__.py
│   └── main.py
├── config/
│   ├── base.yml
│   ├── development.yml
│   ├── production.yml
│   └── staging.yml
├── docs/
├── docs-restore/
│   ├── ARCHITECTURE.md
│   ├── module-01-config-and-settings.md
│   ├── module-02-startup-registries.md
│   ├── module-03-main-agent-and-graph.md
│   ├── module-04-base-agent-and-subagents.md
│   ├── module-05-tools-and-mcp.md
│   ├── module-06-skills.md
│   ├── module-07-api-chat-auth.md
│   ├── module-08-services-datastores.md
│   ├── module-09-cross-cutting.md
│   ├── PROJECT_STRUCTURE.md
│   ├── README.md
│   └── REPRODUCTION-CHECKLIST.md
├── evals/
│   ├── metrics/
│   │   ├── prompts/
│   │   │   ├── conciseness.md
│   │   │   ├── hallucination.md
│   │   │   ├── helpfulness.md
│   │   │   ├── relevancy.md
│   │   │   └── toxicity.md
│   │   └── __init__.py
│   ├── evaluator.py
│   ├── helpers.py
│   ├── main.py
│   └── schemas.py
├── front/
│   ├── index.html
│   └── login.html
├── grafana/
│   └── dashboards/
│       ├── json/
│       │   └── llm_latency.json
│       └── dashboards.yml
├── scripts/
│   ├── migrate_full_repository.sh
│   ├── migrate_srda_agent.sh
│   └── set_env.sh
├── tests/
│   ├── test_python_exec.py
│   ├── test_skill_registry.py
│   └── test_skill_routing.py
├── .gitignore
├── an-agent-update.sql
├── Makefile
├── pyproject.toml
├── README.md
└── uv.lock
```

---

## 按角色的路径速查

| 角色 | 路径 |
|------|------|
| 应用入口 | `app/main.py` |
| FastAPI 工厂与 lifespan | `app/core/app_factory.py` |
| 启动与注册表编排 | `app/core/startup.py` |
| 主 Agent 图 | `app/core/agents/main/agent.py` |
| Agent 基类 / 子 Agent | `app/core/agents/base.py`，`app/plugins/agents/*/` |
| 工具注册与 MCP | `app/core/tools/registry.py`，`app/core/tools/mcp/client.py` |
| Func 工具插件 | `app/plugins/tools/` |
| Skill 插件 | `app/plugins/skills/{name}/SKILL.md` 及同目录子文件夹 |
| HTTP API | `app/api/v1/*.py` |
| Pydantic 协议 | `app/schemas/*.py` |
| LLM / 记忆 / checkpoint / DB | `app/services/*.py` |
| 分层配置 | `config/base.yml` + `config/{environment}.yml` |
| 依赖锁定 | `pyproject.toml`，`uv.lock` |
| 复现文档 | `docs-restore/*` |

---

## 配置与代码可能不一致处（复现时注意）

`config/base.yml` 中存在 **`agents.skill_runner`** 段，且存在 `app/plugins/agents/prompts/skill_runner.md`。当前快照下 **`app/plugins/agents/skill_runner/` 包（含 `agent.py`）未出现在仓库中**。复现完整功能时需自行补全该子 Agent，或从配置中移除 `skill_runner` 相关条目以免启动期 `create` 失败。

---

## 更新本文件

在仓库根目录执行（与上文相同的排除规则）：

```bash
python3 << 'PY'
import os
ROOT = "."
SKIP_DIR = {".git", ".venv", "__pycache__", ".pytest_cache", ".ruff_cache", ".mypy_cache"}
SKIP_NAME = {".DS_Store"}

def should_skip_dir(name):
    return name in SKIP_DIR or name.endswith(".egg-info")

def tree(path, prefix=""):
    try:
        entries = sorted(os.listdir(path), key=lambda s: (not os.path.isdir(os.path.join(path, s)), s.lower()))
    except (PermissionError, FileNotFoundError):
        return
    dirs, files = [], []
    for name in entries:
        if name in SKIP_NAME:
            continue
        full = os.path.join(path, name)
        if os.path.isdir(full):
            if should_skip_dir(name):
                continue
            dirs.append(name)
        else:
            if name.endswith(".pyc"):
                continue
            files.append(name)
    items = [(d, True) for d in dirs] + [(f, False) for f in files]
    for i, (name, is_dir) in enumerate(items):
        is_last = i == len(items) - 1
        branch = "└── " if is_last else "├── "
        print(prefix + branch + name + ("/" if is_dir else ""))
        if is_dir:
            ext = "    " if is_last else "│   "
            tree(os.path.join(path, name), prefix + ext)

os.chdir(ROOT)
print("an-agent/")
tree(".", "")
PY
```

将输出粘贴回本文档的「目录树」代码块即可（并酌情修订「说明」小节）。
