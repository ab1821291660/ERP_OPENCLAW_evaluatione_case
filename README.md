# Agent Harness Platform

面向 AI Agent 的工程化平台——把 Agent 从"能跑"推进到**可观测、可评测、可回归、可上线**。

---

## 目录

- [10 秒快速开始](#10-秒快速开始)
- [这个项目是什么](#这个项目是什么)
- [项目结构](#项目结构)
- [启动服务](#启动服务)
- [配置 LLM](#配置-llm)
- [API 接口手册](#api-接口手册)
- [三个内置工具](#三个内置工具)
- [自动评测系统](#自动评测系统)
- [全链路追踪](#全链路追踪)
- [沙箱安全执行](#沙箱安全执行)
- [CI 回归门禁](#ci-回归门禁)
- [Docker 部署](#docker-部署)

---

## 10 秒快速开始

```bash
# 1. 安装依赖
pip install -r requirements.txt

# 2. 启动服务（无需任何外部服务，SQLite 本地存储）
uvicorn app.main:app --host 0.0.0.0 --port 8000

# 3. 打开浏览器 http://localhost:8000
#    点击 [API 文档] → 找到 POST /api/v1/runs → Try it out → 执行
```

> 默认使用内置 Mock LLM，不需要 API key 即可体验。接入真实 LLM 见 [配置 LLM](#配置-llm)。

---

## 这个项目是什么

**解决的问题**：做了个 AI Agent，本地能跑，但缺观测、缺评测、缺安全边界、不敢上生产。

**平台能力**：

```
你问一个问题
  ↓
POST /api/v1/runs  ─── 创建任务
  ↓
LangGraph 工作流 ───── 7 个节点串联执行
  ├─ plan           LLM 分析问题
  ├─ select_tool    判断是否需要工具
  ├─ invoke_tool    通过 Tool Gateway 调用工具
  ├─ synthesize     综合生成回答
  └─ validate       校验输出
  ↓
GET /api/v1/runs/{id}  ─── 获取结果
  ↓
GET /api/v1/runs/{id}/trace/spans  ─── 查看调用链耗时
```

**技术栈**：Python 3.11+ / FastAPI / LangGraph / SQLAlchemy / OpenTelemetry / Docker

---

## 项目结构

```
ai-agent-runtime/
├── app/
│   ├── main.py                    ← 入口，首页 HTML
│   ├── api/routers/
│   │   ├── runs.py                ← 创建/查询 Agent 任务
│   │   ├── tools.py               ← 工具注册/调用
│   │   ├── evals.py               ← 评测任务
│   │   ├── reports.py             ← 报告查询
│   │   └── config.py              ← 运行时配置（在线设 API key）
│   ├── runtime/                   ← LangGraph 工作流
│   │   ├── state.py               ←   状态定义
│   │   ├── nodes.py               ←   7 个节点实现
│   │   ├── graph.py               ←   图编排
│   │   └── runner.py              ←   执行入口
│   ├── tool_gateway/              ← 工具网关（核心模块）
│   │   ├── registry.py            ←   工具注册中心
│   │   ├── client.py              ←   统一调用接口
│   │   ├── policy.py              ←   安全策略
│   │   ├── executor.py            ←   分发执行
│   │   └── audit.py               ←   审计日志
│   ├── sandbox/                   ← 沙箱安全执行
│   ├── observability/             ← OpenTelemetry 追踪
│   ├── evals/                     ← 评测引擎
│   │   └── evaluators/            ← exact_match / rubric_judge / tool_success
│   ├── reports/                   ← 报告生成
│   ├── db/                        ← 数据层
│   └── schemas/                   ← Pydantic 模型
├── mcp_servers/                   ← 工具实现
│   ├── filesystem_server.py       ←   文件系统工具
│   └── sandbox_server.py          ←   沙箱命令工具
├── datasets/                      ← 评测数据集
├── scripts/ci-eval.py             ← CI 回归检测
├── docker-compose.yml             ← 一键部署
└── requirements.txt
```

---

## 启动服务

### 方式 1：直接运行（推荐开发用）

```bash
pip install -r requirements.txt
uvicorn app.main:app --host 0.0.0.0 --port 8000
```

打开 `http://localhost:8000`，首页有中文功能卡片，点 `[API 文档]` 进入 Swagger 页面。

### 方式 2：Docker Compose（含 Postgres + Redis + Jaeger）

```bash
docker compose up -d
```

服务：
| 服务 | 端口 | 用途 |
|------|------|------|
| api | 8000 | 主服务 |
| postgres | 5432 | 持久化存储 |
| redis | 6379 | 任务队列 |
| jaeger | 16686 | Trace 可视化 |

---

## 配置 LLM

### 方式 1：API 在线配置（推荐）

服务启动后，无需重启即可设置：

```bash
curl -X POST http://localhost:8000/api/v1/config/llm \
  -H "Content-Type: application/json" \
  -d '{
    "api_key": "sk-your-key",
    "base_url": "https://api.deepseek.com",
    "model": "deepseek-chat"
  }'
```

查看当前配置（key 已脱敏）：

```bash
curl http://localhost:8000/api/v1/config/llm
# → {"api_key": "sk-y****key", "configured": true}
```

### 方式 2：环境变量

```bash
export LLM_API_KEY="sk-your-key"
export LLM_BASE_URL="https://api.deepseek.com"
export LLM_MODEL="deepseek-chat"
uvicorn app.main:app --host 0.0.0.0 --port 8000
```

### 支持的 LLM 提供商

| 提供商 | base_url | model 示例 |
|--------|----------|-----------|
| DeepSeek | `https://api.deepseek.com` | `deepseek-chat` |
| OpenAI | 不设（默认） | `gpt-4.1` |
| 其他兼容 | 你的代理地址 | 对应模型名 |

> 任何兼容 OpenAI `/v1/chat/completions` 格式的服务都可以接入。

---

## API 接口手册

### 核心流程（3 步上手）

```bash
# 第1步：提交任务
curl -X POST http://localhost:8000/api/v1/runs \
  -H "Content-Type: application/json" \
  -d '{"agent_name":"demo","task_type":"general","input":{"question":"帮我看看当前目录有什么文件"}}'
# → {"run_id": "run_xxx", "status": "running"}

# 第2步：查看结果（稍等 3-5 秒让 Agent 执行）
curl http://localhost:8000/api/v1/runs/run_xxx
# → {"status": "completed", "output": {"answer": "当前目录包含..."}}

# 第3步：查看调用链
curl http://localhost:8000/api/v1/runs/run_xxx/trace/spans
# → 12 个 span，每个节点的耗时
```

### 全部接口一览

| 方法 | 路径 | 说明 |
|------|------|------|
| `POST` | `/api/v1/runs` | 创建 Agent 任务 |
| `GET` | `/api/v1/runs/{id}` | 查询任务结果 |
| `GET` | `/api/v1/runs/{id}/trace` | Trace 摘要 |
| `GET` | `/api/v1/runs/{id}/trace/spans` | 完整调用链 |
| `GET` | `/api/v1/tools` | 列出已注册工具 |
| `POST` | `/api/v1/tools/register` | 注册新工具 |
| `POST` | `/api/v1/tools/{name}/invoke` | 手动测试工具 |
| `POST` | `/api/v1/evals` | 提交评测任务 |
| `GET` | `/api/v1/evals/{id}` | 评测摘要 |
| `GET` | `/api/v1/evals/{id}/report?format=md` | 评测报告 |
| `GET` | `/api/v1/evals/datasets` | 可用数据集 |
| `GET` | `/api/v1/metrics` | 平台指标 |
| `GET` | `/api/v1/config/llm` | 查看 LLM 配置 |
| `POST` | `/api/v1/config/llm` | 设置 LLM 配置 |
| `GET` | `/health` | 健康检查 |

### task_type 说明

| task_type | 适用场景 | 可用工具 |
|-----------|---------|---------|
| `general` | 文件操作、命令执行（默认） | fs_list_dir, fs_read_file, exec_command |
| `customer_support` | 客服对话 | 仅对话，无工具 |

---

## 三个内置工具

服务启动时自动注册，可通过 `GET /api/v1/tools` 查看。

### fs_list_dir — 列出目录

```bash
curl -X POST http://localhost:8000/api/v1/tools/fs_list_dir/invoke \
  -H "Content-Type: application/json" \
  -d '{"payload": {"path": "."}}'
```

### fs_read_file — 读取文件

```bash
curl -X POST http://localhost:8000/api/v1/tools/fs_read_file/invoke \
  -H "Content-Type: application/json" \
  -d '{"payload": {"path": "README.md"}}'
```

### exec_command — 沙箱执行命令

```bash
# 允许的命令
curl -X POST http://localhost:8000/api/v1/tools/exec_command/invoke \
  -H "Content-Type: application/json" \
  -d '{"payload": {"command": ["ls", "-la"], "cwd": "."}}'

# 被禁止的命令
curl -X POST http://localhost:8000/api/v1/tools/exec_command/invoke \
  -H "Content-Type: application/json" \
  -d '{"payload": {"command": ["rm", "-rf", "/"]}}'
# → {"status": "failed", "error": "Sandbox policy violation: 'rm' not in allowlist"}
```

**白名单命令**：`ls`, `cat`, `head`, `tail`, `wc`, `grep`, `find`, `echo`, `date`, `whoami`, `pwd`, `env`, `git`, `python3`, `python`

---

## 自动评测系统

### 数据集格式

放在 `datasets/` 目录，JSON 格式：

```json
[
  {
    "case_id": "support-001",
    "task_type": "customer_support",
    "input": {"question": "订单什么时候发货？"},
    "expected": {"must_include": ["订单", "物流", "发货"], "format": "json"},
    "metadata": {"difficulty": "easy"}
  }
]
```

### 提交评测

```bash
curl -X POST http://localhost:8000/api/v1/evals \
  -H "Content-Type: application/json" \
  -d '{
    "agent_name": "support-agent",
    "dataset_name": "customer_support_v1",
    "evaluators": ["exact_match", "rubric_judge"]
  }'
```

### 查看报告

```bash
# JSON 格式
curl http://localhost:8000/api/v1/evals/{eval_id}/report

# Markdown 格式
curl http://localhost:8000/api/v1/evals/{eval_id}/report?format=md
```

### 三种 Evaluator

| Evaluator | 评分方式 |
|-----------|---------|
| `exact_match` | 检查回答是否包含 expected 中的 `must_include` 关键词 |
| `tool_success` | 检查 Agent 调用的工具是否全部成功 |
| `rubric_judge` | LLM 作为裁判，从相关性、完整性、准确性三维评分 |

---

## 全链路追踪

每次 Agent 请求自动采集 12 个 OpenTelemetry Span：

```
agent_run (总耗时)
 ├─ prepare_context
 ├─ plan
 │  └─ llm_call          ← LLM 调用耗时
 ├─ select_tool
 ├─ invoke_tool
 │  ├─ tool_gateway.invoke
 │  └─ mcp_call          ← 工具执行耗时
 ├─ synthesize_answer
 │  └─ llm_call
 ├─ validate_output
 └─ finish
```

查看 Trace：

```bash
curl http://localhost:8000/api/v1/runs/{run_id}/trace/spans
```

启动 Jaeger 后可在 `http://localhost:16686` 查看可视化瀑布图。

---

## 沙箱安全执行

所有命令执行经过三层防护：

1. **命令白名单** — 只允许 `ls`, `cat`, `grep`, `git` 等安全命令
2. **超时控制** — 默认 15 秒超时
3. **输出限制** — 最大 1MB

尝试执行 `rm`, `sudo`, `chmod` 等危险命令会被直接拒绝并记录审计日志。

---

## CI 回归门禁

在 GitHub Actions 中自动运行评测，阻止质量回退：

```yaml
# .github/workflows/eval-regression.yml
# PR 提交时自动触发
python scripts/ci-eval.py --dataset customer_support_v1
# 不通过阈值 → exit 1 → PR 被阻止合并
```

本地运行：

```bash
python scripts/ci-eval.py --dataset customer_support_v1
```

可配阈值（环境变量）：

```bash
export CI_PASS_RATE_MIN=0.6      # 通过率不低于 60%
export CI_EXACT_MATCH_MIN=0.6    # 关键词匹配不低于 0.6
export CI_RUBRIC_JUDGE_MIN=0.6   # LLM 评分不低于 0.6
```

---

## Docker 部署

```bash
docker compose up -d
```

启动后：
- API: `http://localhost:8000`
- Jaeger UI: `http://localhost:16686`

---

## License

MIT
