







##===================================##===================================
##===================================##===================================
##===================================##===================================
##===================================##===================================
# 金融服务领域的 LLM 与 AI Agent 部署门禁

这是一个面向金融服务领域 AI Agent 和 LLM 模型的**部署门禁**：让系统在金融黄金数据集上运行，使用领域专用评估器为每个输出打分，并以部署迭代的速度生成**有可复核证据支撑的 PASS/FAIL 结论**。本项目基于 [Langfuse](https://langfuse.com) 实验和开源金融评测数据集构建，并使用 [ClickHouse](https://clickhouse.com) 作为分析后端。

📖 **配套博客：** [Building Deployment Gates for LLMs and AI Agents in Financial Services](https://langfuse.com/blog/2026-07-15-llm-certification-financial-services)——介绍本仓库背后的故事与设计思路。

🤖 **编码 Agent：** 请从 [`AGENTS.md`](AGENTS.md) 开始——其中集中给出了黄金路径、凭据检查点、前置条件和成功标准（`make quickstart`），还介绍了如何使用 [Langfuse CLI](https://langfuse.com/docs/api-and-data-platform/features/cli) 和 [Agent Skill](https://langfuse.com/docs/api-and-data-platform/features/agent-skill) 操作 Langfuse 部署。

> **关于“certification（认证）”一词的说明。** 这是我们在构建过程中使用的内部名称，并保留在仓库名、数据集名、脚本名以及下方截图中。本项目不会颁发任何证书。该流水线是一个生成可复核证据的部署门禁；监管批准、独立验证、法律审查以及由责任人完成的签字确认，仍然必须由人来承担。

项目开箱即用地提供两种门禁模式：

| 模式 | 入口 | 通过条件 | 内置能力 |
|---|---|---|---|
| **模型门禁** | `run_certification.py` | 每个数据项调用一次 LLM；主准确率分数 ≥ 阈值 | 任意 OpenAI-compatible 端点或 Claude 模型 |
| **[AI Agent 门禁](#gating-the-whole-agent)** | `run_usecase_certification.py` | 每个数据项运行一个多步骤 **Agent**；**每个**门禁维度都必须通过（准确率、事实忠实度、合规性、工具使用等） | 三个已注册 Agent：**`10k-analyst`**（带计算器工具的 SEC 文件问答）、**`sentiment-triage`**（带人工分流的分类）、**`advisory-draft`**（受合规门禁约束的客户文案） |

这些组件端到端组成了 Langfuse 的 [AI 工程闭环](docs/ai-engineering-loop.md)（Trace → Monitor → Build Datasets → Experiment → Evaluate）：生产失败样本被回灌到黄金数据集；当 Prompt 在 Langfuse 中被提升时，CI 会重新运行受影响的门禁（一次性配置请参阅[闭环激活检查清单](docs/loop-activation-checklist.md)）。详见[持续更新证据](#keeping-the-evidence-current)。

## 项目背景

模型风险管理团队批准的不是裸模型，而是构建在模型之上的 **AI Agent 和应用**：例如 10-K 分析助手、舆情分流流水线、客户投顾文案工具。困难不在于只运行一次测试，而在于 LLM 系统的变化速度远高于底层模型：Prompt 会被提升，模型会被替换，工具和检索逻辑也会持续演进。**等人工审批完成时，被测试的系统往往已经发生了变化。**

漏检的代价是不对称的。假设一次 FinanceBench 运行在 85% 的准确率门槛下只得到 81.3%：如果门禁在测试阶段发现这个错误，代价几乎为零——仪表盘上一个数字变红，发布前有人介入排查；如果部署后才发现，同一个错误就意味着客户在 10-K 摘要中收到了错误数字，而在金融服务领域，这可能成为需要报告的事件。

因此，我们没有单纯扩大人工评审规模，而是**把证据生产脚本化**。流水线让一个 LLM 系统——可以是一次模型调用，也可以是完整的多步骤 Agent——在金融黄金数据集上运行，使用领域评估器为每个输出打分，并生成带完整、可复现审计轨迹的 PASS/FAIL 结论。原本需要两周的人工测试可以变成一天内的一条命令；接入 CI 后，每当系统变化，这项检查都会自动重跑。

## 架构

流水线将 Langfuse 的 AI 工程闭环——**Trace → Monitor → Build Datasets → Experiment → Evaluate**——实现为一个循环，并以 Langfuse 作为数据集、Trace、分数、Prompt 和评审队列的记录系统：

```text
 构建数据集             实验                          评估                             结果
+------------------+    +------------------------+    +------------------------+    +--------------+
| 黄金数据集        |    | 实验 Runner             |    | 评估器                  |    | 逐项分数      |
| - FinanceBench   |--->| run_certification.py   |--->| 确定性评估：             |--->| PASS / FAIL  |
| - Financial PB   |    |   模型门禁（任意        |    |   准确率、情绪、合规、   |    | 审计轨迹      |
| - advisory-      |    |   OpenAI-compatible    |    |   完整度、工具轨迹       |    | Portal (SPA) |
|   adversarial    |    |   模型）                 |    | LLM-as-a-Judge：        |    | 导出 MD/     |
| - 回灌的生产      |    | run_usecase_certifi-   |    |   groundedness         |    |   JSON/CSV   |
|   Trace          |    |   cation.py（3 个       |    | 门禁：单阈值 + 多维度   |    +--------------+
+------------------+    |   Claude Agent，嵌套    |    +------------------------+
     ^                  |   Trace）               |
     |                  +------------------------+
     |                    ^
     |                    | Edge B——发布 → 重新门禁：把 Prompt 提升到
     |                    | `production` 后触发 repository_dispatch →
     |                    | prompt-recert.yml → scripts/recert_for_prompt.py --ci
     |                  +------------------------+
     |                  | Langfuse Prompt 管理    | 托管模板：financial-qa、
     |                  |（编辑 / 版本化 /        | financial-sentiment、usecase-*
     |                  |  提升）                 |（Runner 在运行时获取）
     |                  +------------------------+
     |
     | Edge A——观测 → 开发：promote_trace_to_dataset.py --from-queue
     |（被标记 Trace → 数据项 `prod-<traceId>`；由人提供正确答案）
     |
+------------------+    +------------------------+    +------------------------+
| “Certification   |<---| monitor_production.py  |<---| 生产 Trace              |
|  Review”队列     |    | 对在线 Trace 做合规与   |    |（已接入 Langfuse 的      |
|（人工评审）       |    | 完整度检查；            |    |  已部署应用）             |
+------------------+    | --queue-violations     |    +------------------------+
 监控                   +------------------------+      Trace
```

上半部分是部署门禁路径，下半部分是生产可观测路径。两条反馈边闭合了循环：**Edge A** 将经过评审的生产失败转成黄金数据项，供下一次运行做回归测试；**Edge B** 在托管 Prompt 被提升时自动重新运行门禁。

## 黄金数据集

门禁的可信度取决于它所使用的真值。黄金数据集是每次运行都会评分的可复现测试集——本项目包含两个开放金融基准，以及一个用于合规门禁的小型对抗集。

### `setup_datasets.py`——数据集加载器

从 HuggingFace 或内置样本文件将金融黄金数据集加载到 Langfuse。

```text
选项：
  --dataset {financebench,fpb,advisory-adversarial,all}
                                     要加载的数据集（默认：all——
                                     包含 financebench + fpb；
                                     advisory-adversarial 需要显式选择）
  --sample                           使用内置样本数据（离线模式）
  --prefix PREFIX                    数据集名前缀（默认：certification）
  --dry-run                          仅预览，不创建
```

**支持的数据集：**

| 数据集 | 数据量 | 来源 | 关注点 |
|---|---:|---|---|
| `financebench` | 10（样本）/ 150（全量） | [PatronusAI/financebench](https://huggingface.co/datasets/PatronusAI/financebench) | 基于 SEC 文件的金融问答 |
| `fpb` | 10（样本）/ 约 4,850（全量） | [ChanceFocus/en-fpb](https://huggingface.co/datasets/ChanceFocus/en-fpb) | 金融情绪分类 |
| `advisory-adversarial` | 10（内置） | `sample_data/advisory_adversarial.json` | 客户更新简报：3 个合规控制样本 + 7 个分别诱导不同非合规投顾表达的样本（保证、买卖建议、无风险安抚、确定性表述、非公开信息、集中持仓建议） |

`advisory-adversarial` 会加载为 `certification/advisory-adversarial`（没有 `-sample`/`-v1` 后缀），并被刻意排除在 `--dataset all` 之外。它用于演示 `advisory-draft` Agent 的合规硬门禁：设置 `ADVISORY_TEMPT_NONCOMPLIANT=1` 运行后，Agent 会被诱导生成禁用表达，因此即使其他分数全部完美，合规维度仍会让整个运行失败。

> **稳定数据项 ID。** `setup_datasets.py` 为每个黄金数据项分配确定性 ID（`financebench-sample-<financebench_id>`、`fpb-sample-<sha256(text)[:12]>`、`advisory-adversarial-<nnn>`），所以重复运行会原地 upsert，而不会制造重复项。每次修改数据项都会在 Langfuse 中生成新的**数据集版本**；门禁运行会记录实际使用的版本（实验元数据中的 `dataset_version`），证据包也会将其打印出来。在稳定 ID 引入之前播种的数据集使用随机 ID，对它们重跑会多出一份副本，因此 `make seed` 仍会跳过已经包含数据项的数据集。

## 运行实验

每次运行都是一个 Langfuse 实验：待测系统会对黄金数据集中的每个数据项执行，所有输出都被评分，整个过程也会被追踪，以便后续复现和复核。

### 前置条件

- Python 3.11+
- [uv](https://docs.astral.sh/uv/)（推荐）或 pip
- Node.js 20+ / npm 10+（用于构建 Portal 前端）
- 一个 Langfuse 实例——见下方[选择 Langfuse 部署方式](#choose-your-langfuse-deployment)
- 一个 LLM API Key（OpenAI、Anthropic 或任意 OpenAI-compatible 端点）
- Docker 24+ 与 Compose v2——仅自托管 Langfuse 时需要

<a id="choose-your-langfuse-deployment"></a>

### 选择 Langfuse 部署方式

流水线只通过 `LANGFUSE_PUBLIC_KEY`、`LANGFUSE_SECRET_KEY` 和 `LANGFUSE_BASE_URL` 与 Langfuse 通信，因此同一套代码可用于两种部署方式。运行快速开始之前，请先选择一种。

| | Langfuse Cloud（推荐） | 自托管（OSS） |
|---|---|---|
| 最适合 | 最快开始评测，无需管理基础设施 | 隔离网络、严格数据驻留要求 |
| 设置时间 | 约 2 分钟（注册） | 约 10 分钟（Docker Compose） |
| 成本 | 免费套餐，超出后有付费方案（见[价格](https://langfuse.com/pricing)） | 仅基础设施成本——Langfuse OSS 使用 Apache 2.0 |
| 数据驻留 | 由 Langfuse 管理（欧盟或美国区域） | 由你自己的基础设施管理 |
| 维护 | 无 | 需要运维 Postgres、ClickHouse、Redis、MinIO |
| 本仓库前置条件 | Cloud 账号 | Docker 24+ 与 Compose v2 |

#### 方案 A——Langfuse Cloud

1. 在 <https://cloud.langfuse.com/auth/sign-up>（欧盟）或 <https://us.cloud.langfuse.com/auth/sign-up>（美国）注册。
2. 创建 **Organization → Project**。
3. 打开 **Settings → API Keys → Create new API keys**，复制公钥和密钥。
4. 将它们粘贴到 `.env`（该文件在[快速开始第 1 步](#1-setup)创建）。`LANGFUSE_BASE_URL` 默认已设为 `https://cloud.langfuse.com`；如果注册的是美国区域，请在 `.env.example` 中启用美国区域地址。

#### 方案 B——自托管 Langfuse

仓库附带一套经过整理的 Docker Compose 栈，Langfuse 固定在 v4（ClickHouse ≥ 25.12；本项目使用的 Python SDK v5 要求 Langfuse ≥ 4.59.0）。在仓库根目录执行：

```bash
docker compose -f selfhost/docker-compose.yml up -d
# 打开 http://localhost:3000 → 注册 → 创建 Organization + Project →
# Settings → API Keys → Create new API keys
```

然后在[快速开始第 1 步](#1-setup)中启用 `.env.example` 的自托管地址，并注释 Cloud 默认地址，使 `.env` 包含：

```bash
LANGFUSE_BASE_URL=http://localhost:3000
```

停止/重置命令、网络暴露建议、凭据轮换和通过 Kubernetes 进行生产自托管的说明，请参阅 [`selfhost/README.md`](selfhost/README.md)。如果计划在本地栈上运行完整的 150 条 FinanceBench 数据，还应阅读[故障排查：长任务卡住](#troubleshooting-hangs-on-long-runs)——包含较长思维链的长任务可能耗尽本地 Langfuse 的 OTel 队列。

> **建议：** 先使用 Cloud 免费套餐跑通端到端流水线；如果合规或数据驻留提出要求，再迁移到自托管。两种部署方式使用完全相同的脚本、评估器、数据加载器、Portal 和 CI 工作流。

#### 同时保留本地与 Cloud 配置（可选）

你不必只做一次选择：可以为每种部署保留一个凭据 profile，并随时切换。所有脚本（包括 Portal）都用 `override=True` 加载 `.env`，因此 `.env` **文件**总是优先于 Shell 导出的环境变量；真正的切换点是 `.env` 当前指向哪个 profile，`scripts/use_env.sh` 通过符号链接管理它：

```bash
mv .env .env.local                      # 一次性：将当前 .env 保存为 local profile
cp .env.local .env.cloud                # 然后编辑 .env.cloud，填入 Cloud Key + 基础 URL

bash scripts/use_env.sh status          # 查看当前启用的 profile
bash scripts/use_env.sh cloud           # 让 .env 指向 .env.cloud
bash scripts/use_env.sh local           # 让 .env 指向 .env.local

# 对两个部署依次执行同一条命令，完成后恢复原 profile：
bash scripts/use_env.sh both -- uv run python run_certification.py \
  --dataset certification/financebench-sample --model claude-sonnet-4-6
```

`.gitignore` 已覆盖两个 profile（`.env.*`）。系统没有双写模式——一次门禁运行只连接一个 Langfuse 实例——所以“同时发送到两个实例”的实际含义是对每个 profile 各运行一次同一命令，这正是 `both --` 自动完成的工作（两个实例各自获得 Trace、分数和运行）。Portal 只在启动时读取一次 `.env`；切换后需要重启 Portal。

<a id="quick-start"></a>

### 快速开始

<a id="1-setup"></a>

#### 1. 初始化

```bash
git clone https://github.com/doneyli/langfuse-llm-certification-finance.git
cd langfuse-llm-certification-finance
cp .env.example .env    # 编辑并填入 Langfuse + LLM API 凭据
```

> `.env.example` 中列出了三个 `LANGFUSE_BASE_URL` 选项（Cloud 欧盟、Cloud 美国、自托管）。运行任何命令之前，只启用其中一个并注释另外两个。参阅[选择 Langfuse 部署方式](#choose-your-langfuse-deployment)。

**安装依赖**（二选一）：

```bash
# 推荐：使用 uv（https://docs.astral.sh/uv/）
uv sync

# 备选：使用 pip
pip install -r requirements.txt
```

> 下方示例使用 `uv run`，它会自动创建虚拟环境并安装依赖。如果使用 pip 安装，请去掉 `uv run` 前缀，直接使用 `python`。

#### 2. 加载样本数据集（离线，不需要 HuggingFace）

```bash
uv run python setup_datasets.py --dataset financebench --sample
```

#### 3. 初始化 Langfuse 配置

```bash
uv run python setup_score_configs.py        # 在 Langfuse 中注册分数类型（通过/失败分数为 BOOLEAN）
uv run python setup_annotation_queues.py    # 创建人工评审队列
uv run python setup_prompts.py              # 注册 Prompt 模板
```

#### 4. 运行模型门禁

```bash
uv run python run_certification.py --dataset certification/financebench-sample \
  --model claude-sonnet-4-6 --queue-failures
```

#### 5. 对 AI Agent 执行门禁

对完整 Agent 而不仅是模型执行门禁。10-K Filing Analyst 会制定计划、检索证据、调用计算器工具并生成有事实依据的回答；只有准确率、groundedness、合规性和工具使用**全部**超过各自门槛时才会通过（见[对完整 Agent 执行门禁](#gating-the-whole-agent)）：

```bash
uv run python run_usecase_certification.py --list    # 10k-analyst、sentiment-triage、advisory-draft
uv run python run_usecase_certification.py --use-case 10k-analyst \
  --dataset certification/financebench-sample --queue-failures
```

#### 6. 构建 Portal 前端

```bash
cd portal/frontend && npm install && npm run build && cd ../..
```

#### 7. 启动 Portal

```bash
uv run python -m portal.app    # 打开 http://localhost:8050
```

#### 8. 查看结果

打开 `http://localhost:8050` 查看 Certification Dashboard。模型和 Agent 会显示为不同的行（`claude-sonnet-4-6` 与 `usecase:10k-analyst`）。也可以打开 Langfuse UI > **Experiments**（Runner 会打印实验 URL）。

#### 9. 评审失败项

打开 Langfuse UI > **Annotation Queues** > `Certification Review`，评审自动评估失败的数据项。

#### 10. 导出证据

```bash
uv run python export_results.py --dataset certification/financebench-sample
uv run python export_results.py --dataset certification/financebench-sample --format json --output report.json
```

### 全量数据集模式

要从 HuggingFace 加载完整的 150 条 FinanceBench 数据（需要联网）：

```bash
uv run python setup_datasets.py --dataset financebench        # 从 HuggingFace 下载
uv run python run_certification.py --dataset certification/financebench-v1 --model gpt-4o
```

### `run_certification.py`——实验 Runner

让一个模型运行 Langfuse 数据集、评估输出并报告通过/失败。

```text
选项：
  --dataset DATASET             Langfuse 数据集名称（必填）
  --model MODEL                 要执行门禁的模型（默认：claude-sonnet-4-6）
  --endpoint URL                LLM API 基础 URL（用于自定义网关）
  --threshold FLOAT             通过阈值（默认：0.85）
  --max-concurrency N           并发 API 调用数（默认：5）
  --evaluators {all,...}        要运行的评估器
  --run-name NAME               自定义实验运行名（默认：自动生成）
  --queue-failures              将失败项路由到人工评审队列
  --ci                          除非门禁通过，否则退出码为 1（用于 CI）
  --dry-run                     仅预览数据项
  --system-prompt-file PATH     原样用作 LLM system message 的 Markdown 文件
                                （例如 prompts/finance_expert.md——见下方
                                “领域适配变体”）
  --label NAME                  追加到元数据模型名和运行名的变体标识
                                （例如 finance-expert）。每个 label 都会成为
                                Portal 仪表盘中的独立行。
```

<a id="troubleshooting-hangs-on-long-runs"></a>

#### 故障排查：长任务卡住

如果一次运行在接近尾部时静默冻结（例如执行到 150 条中的约第 140 条，随后 10 分钟以上没有日志，Python 进程仍存在但 CPU 为 0%），原因通常是较慢的本地 Langfuse 导致 OTel `BatchSpanProcessor` 队列饱和：较大的 Span（长证据 + 长思维链输出）堆积速度超过本地实例的摄取速度，导出队列填满，流水线因此死锁。

`run_certification.py` 现在会在启动时设置更安全的 OTel 默认值（队列 20k、batch 64、每 2 秒 flush、导出超时 120 秒）。如需覆盖，请在运行前设置：

```bash
OTEL_BSP_MAX_QUEUE_SIZE=20000      # 默认队列为 2048——对 150+ 数据项过小
OTEL_BSP_MAX_EXPORT_BATCH_SIZE=64  # 更小的 batch 导出更快，队列压力更低
OTEL_BSP_SCHEDULE_DELAY=2000       # 每 2 秒而不是每 5 秒 flush
OTEL_BSP_EXPORT_TIMEOUT=120000     # 给较慢的本地 Langfuse 2 分钟导出时间
LANGFUSE_FLUSH_AT=64
LANGFUSE_FLUSH_INTERVAL=2
```

在 Langfuse Cloud 上未观察到该问题（其摄取更快）。目前只在本地自托管 Langfuse 上、使用完整 150 条 FinanceBench 数据和重思维链 system Prompt 时出现。

## 评估器

评估器承载领域知识。流水线同时使用**确定性评估器**和 **LLM-as-a-Judge 评估器**：确定性检查处理客观、可验证的事实（数字匹配、禁用短语），LLM Judge 评估主观质量维度（groundedness、对源文档的忠实度）。

`evaluators.py` 是一个可导入的评估函数模块，所有函数都遵循 Langfuse SDK 签名。

**确定性评估器**（快速、低成本、可复现）：

| 评估器 | 类型 | 检查内容 |
|---|---|---|
| `numerical_accuracy_evaluator` | 数据项级 | 提取数字，并以 5% 容差比较 |
| `exact_match_evaluator` | 数据项级 | 严格字符串包含 |
| `sentiment_evaluator` | 数据项级 | 情绪分类准确率 |
| `regulatory_compliance_evaluator` | 数据项级 | 扫描被禁止的金融短语 |
| `response_completeness_evaluator` | 数据项级 | 回答长度与结构（输出 `completeness` 分数） |
| `tool_use_correctness_evaluator` | 数据项级 | 轨迹检查：Agent 是否针对题型调用了预期工具（`calculate`、`route`、`compliance-self-check`）？对于非 Agent 的纯字符串输出不返回分数 |

**LLM-as-a-Judge 评估器**（更细致，可发现定性问题）：

| 评估器 | 类型 | 检查内容 |
|---|---|---|
| `groundedness_evaluator` | 数据项级 | 相对于源财报证据的忠实度 + 完整度 |

groundedness 评估器将模型输出、源证据和问题发送给 Judge 模型（默认 `claude-sonnet-4-6`，可通过 `JUDGE_MODEL` 环境变量配置），并使用金融审计员评分标准。它评估**忠实度**（回答中的主张是否有文档支持）和**完整度**（回答是否覆盖相关信息），最终按 70% 忠实度、30% 完整度加权。它只运行在包含源证据的数据项上（例如 FinanceBench）。

**运行级评估器**（跨所有数据项聚合）：

| 评估器 | 类型 | 检查内容 |
|---|---|---|
| `average_score_evaluator(name)` | 运行级 | 对所有数据项的指定分数取平均（输出 `avg_<name>`） |
| `certification_gate(name, threshold)` | 运行级 | 基于单一分数阈值给出 PASS/FAIL（模型门禁） |
| `usecase_certification_gate(thresholds)` | 运行级 | 多维 PASS/FAIL——字典中的每个维度都必须超过阈值；没有分数的维度平均值为 0.0 并失败（Agent 门禁） |

### 为什么需要两类评估器

它们覆盖不同的失败模式。例如，在较早的一次 Haiku 模型门禁运行中：

- **数值准确率**（确定性）：60%——Haiku 经常算错数字；
- **Groundedness**（LLM Judge）：97%——但只要使用了证据，它通常能够忠实地使用证据。

（当前 Haiku 在 10 条样本上的数值准确率为 90%～100%；上述例子用于说明问题，不代表今天的分数。）

如果没有 LLM Judge，你只会看到“60%，FAILED”，并认为模型整体不可靠。有了 Judge，就能看出失败集中在数值推理，而不是对源材料的忠实度。这一区分对模型风险评估很重要。

## 门禁与证据导出

门禁把一次运行的分数转换成评审人真正需要的结论：PASS/FAIL，以及支撑这个结论的证据。模型门禁检查主准确率指标是否达到配置阈值（默认 85%），运行结果只能是 PASSED 或 FAILED。Agent 门禁的原理相同，但使用**多维门禁**——该 Agent 注册的每个维度（准确率、groundedness、合规性、工具使用正确性）都必须同时达到各自阈值（见[对完整 Agent 执行门禁](#gating-the-whole-agent)）。

### 修改通过阈值

```bash
uv run python run_certification.py --dataset my-dataset --threshold 0.90
```

或者修改 `evaluators.py` 中的 `DEFAULT_THRESHOLD`。

### 结果在 Langfuse 中的位置

Langfuse 是证据的事实来源：

- **每次门禁运行都是一个 Experiment**（Langfuse UI → **Experiments**，或 Runner 打印的 URL）。可以在那里将运行与基线并排比较。
- **数据项级分数**（numerical_accuracy、groundedness 等）挂在每个实验数据项上，可在实验视图中逐项查看。
- **运行级分数**（`certification_result`、`avg_numerical_accuracy`、`avg_groundedness` 等）由 Langfuse SDK 的 `run_evaluators` **直接保存到实验本身**，无需额外写入。可通过 `GET /api/public/experiments?fields=scores` 读取（Portal 和 `export_results.py` 采用此方式），也可使用 `GET /api/public/v3/scores?experimentId=<experiment id>`：SDK 会把 `datasetRunId` 设为实验 ID，而这个过滤器正是按该字段匹配。若再次用 `traceId` 重新发布分数（本仓库旧的 `persist_run_evaluations` helper 曾这样做，即使在 SDK 3.14 中也已是重复写入），第二份分数会被归档为某条 Trace 的数据项分数，因而无法被该过滤器看到。详见 [`cicd/README.md`](cicd/README.md#the-verdict-is-a-boolean-score)。
- **通过/失败分数使用 BOOLEAN。** 门禁结论 `certification_result` 和二元数据项检查（`regulatory_compliance`、`tool_use_correctness`、`numerical_accuracy`、`exact_match`、`sentiment_accuracy`）都会写为 Langfuse BOOLEAN 分数（`true`/`false`）；`groundedness`、`completeness` 和所有 `avg_*` 仍为 NUMERIC。这使布尔仪表盘组件、按通过率告警（例如 `regulatory_compliance` 低于 100% 时通知）以及与人工标签的 Score Analytics 混淆矩阵成为可能。若项目以前把这些配置注册成 NUMERIC，可运行 `setup_score_configs.py --migrate-configs` 迁移（先运行 `--dry-run`；脚本会归档旧配置并创建 BOOLEAN 配置）。
- **Agent Trace 会渲染为 Agent Graph。** 每个数据项都在一个 `usecase:<name>` Agent observation 中执行，其下包含 `retriever`、`tool`、`guardrail` 步骤；由托管 Prompt 编译的 Generation 会关联到对应 Prompt 版本（Prompts → Metrics 可查看每个版本的分数、延迟和成本）。
- **运行接受判断时使用的门槛**会记录在实验元数据中（Agent 运行使用 `gate_thresholds`，模型运行使用 `threshold` + `gate`）。Agent 的建议门槛放在独立的 `advisory_thresholds` 中，因此任何重放结论的逻辑都不会把建议项误当成硬门槛。SDK v5 升级前记录的运行在 v4 API 中没有元数据，所以 Portal 只显示结论，不显示门槛。

### `export_results.py`——证据导出器

将实验分数导出为可复核证据包，用于合规/AMRM 报告生成。

证据包的 **Gate** 行会从运行自身元数据中读取实际判断门槛：模型运行显示 `numerical_accuracy ≥ 85%`，Agent 运行显示每个维度（`all must clear: …`），没有记录门槛的运行显示 `—`。建议门槛拥有单独的 **Advisory** 行，绝不会出现在 Gate 行；Scores Summary 的 **Bar** 列则将每个评估器标记为：相对于必选门槛的 `PASS`/`FAIL`、相对于建议门槛的 `MET`/`MISS (advisory)`，或 `—`。导出器与 Portal 共享同一套规则（`cert_common.recorded_gate`、`recorded_advisory`、`bar_verdict`），因此两者不会对同一次运行报告出不同门槛。

```text
选项：
  --dataset DATASET             Langfuse 数据集名称（必填）
  --run-name NAME               指定运行（默认：最新）
  --format {markdown,json,csv}  输出格式（默认：markdown）
  --output FILE                 输出文件（默认：stdout）
```

<a id="gating-the-whole-agent"></a>

## 对完整 Agent 执行门禁

除了对**模型**执行门禁，流水线还可以对 **AI Agent** 执行门禁——即针对特定业务目的部署的多步骤系统。只有当**整个系统**同时通过每一项生产就绪门槛时，Agent 才能通过。这是一个每个必选维度都必须通过的**多维门禁**，而不仅是单一准确率分数。

| | 模型门禁 | AI Agent 门禁 |
|---|---|---|
| 入口 | `run_certification.py` | `run_usecase_certification.py` |
| `task` | 一次 LLM 调用 | 多 Span **Agent** |
| Trace | 扁平 | 嵌套（每个 Agent 步骤一个 Span） |
| 门禁 | 一个分数 ≥ 阈值 | **多维**，全部必须通过 |
| 仪表盘行 | 模型名 | `usecase:<name>` |
| 模型 | 任意 OpenAI-compatible 端点或 Claude | 仅 Claude（Agent 调用原生 Anthropic SDK） |

**三个已注册 Agent**分别定义自己的 Trace 形态和门禁。Agent 需要检查哪些维度是 Agent 本身的属性；具体阈值放在 [`cicd/thresholds.json`](cicd/thresholds.json) 中，而不是 CLI 参数里：

| Agent | 步骤（Span） | 门禁：每个必选门槛都必须通过（建议门槛只报告，不阻断） |
|---|---|---|
| `10k-analyst`——带计算器的 SEC 文件事实型问答 | plan → retrieve-evidence → calculate（工具）→ compose-answer | numerical_accuracy ≥ 0.85、groundedness ≥ 0.80、regulatory_compliance = 1.00、tool_use_correctness ≥ 0.90 |
| `sentiment-triage`——情绪分类，将低置信度项转人工 | classify → rationale → route（工具） | sentiment_accuracy ≥ 0.85、regulatory_compliance = 1.00、tool_use_correctness = 1.00 |
| `advisory-draft`——带合规硬门禁的事实型客户摘要 | analyze → draft → compliance-self-check（工具） | groundedness ≥ 0.80、regulatory_compliance = 1.00、tool_use_correctness = 1.00；*建议项：* completeness ≥ 0.70 |

请注意，每个 Agent 的门禁不同：`sentiment-triage` 没有 groundedness 维度（没有需要忠实引用的源文档），`advisory-draft` 甚至没有准确率维度——它对有事实依据且**合规**的文案进行门禁。`regulatory_compliance = 1.00` 在所有场景中都是硬门槛：一个禁用短语就会让整个运行失败。

门槛也可以是**建议项**：在 `cicd/thresholds.json` 中配置 `{"threshold": 0.70, "required": false}`。建议项会被测量和报告（`avg_<dim>` 分数、`certification_result` 注释中的 `MET`/`MISS`、Runner 摘要、实验元数据中的 `advisory_thresholds`），但绝不会让运行或 `--ci` 退出码失败。`advisory-draft` 的 completeness 是建议项，因为它只是长度和结构启发式，并非正确性或安全属性。Agent 至少要保留一个必选门槛，否则在导入时抛错。详见[必选门槛与建议门槛](cicd/README.md#required-bars-and-advisory-bars)。

```bash
uv run python run_usecase_certification.py --list           # 列出已注册 Agent
uv run python run_usecase_certification.py --use-case 10k-analyst \
    --dataset certification/financebench-sample --model claude-sonnet-4-6
bash scripts/demo_usecase.sh                                 # 完整生命周期演示
```

```text
选项：
  --list                        列出已注册 Agent 并退出
  --use-case {10k-analyst,sentiment-triage,advisory-draft}
                                要执行门禁的 Agent（除非使用 --list，否则必填）
  --dataset DATASET             Langfuse 数据集名称（除非使用 --list，否则必填）
  --model MODEL                 Agent 使用的基础 Claude 模型（默认：claude-sonnet-4-6）
  --max-concurrency N           并发数据项数（默认：5）
  --run-name NAME               自定义实验运行名（默认：自动生成）
  --queue-failures              将失败项路由到标注队列
  --ci                          除非每个必选维度通过，否则退出码为 1（用于 CI）
  --dry-run                     仅预览数据项
```

`run_usecase_certification.py` 让一个已注册的多步骤 **Agent**（嵌套 Trace，每个步骤一个 Span）在 Langfuse 数据集上运行，并应用该 Agent 的**多维门禁**——每个必选维度都必须通过。

共享基础设施（多维门禁、轨迹评估器、Agent 注册表、追踪 helper）和三个 Agent 都已位于 `main`。参阅：

- [`docs/ai-engineering-loop.md`](docs/ai-engineering-loop.md)——**目标**：本项目如何端到端组成 Langfuse 的 [AI Engineering Loop](https://langfuse.com/academy/ai-engineering-loop)（Trace → Monitor → Build Datasets → Experiment → Evaluate），以及 Prompt 的 CI/CD 方案
- [`docs/loop-activation-checklist.md`](docs/loop-activation-checklist.md)——启用反馈边所需的一次性控制台/配置操作（Langfuse Automation、GitHub Secrets/PAT）
- [`docs/usecase-certification.md`](docs/usecase-certification.md)——实现规格（三个 Agent）
- [`docs/usecase-architecture.md`](docs/usecase-architecture.md)——架构 + 评估生命周期组件图
- [`docs/usecase-runbook.md`](docs/usecase-runbook.md)——运行手册 + 演示解说
- [`docs/hosting-demo.md`](docs/hosting-demo.md)——托管公开只读演示的计划（Vercel + Langfuse Cloud；已设计，尚未执行）
- [`cicd/README.md`](cicd/README.md)——**质量门槛即代码**：所有门禁阈值集中在一个可评审文件中，并解释硬/松门槛划分、尚未覆盖的维度，以及本仓库刻意**不在** CI 中运行的内容

## 面向签字负责人的 Portal

只有当对发布负责的人能够读懂证据时，门禁才有价值。**Certification Portal** 是一个 Web 仪表盘，业务和合规相关人员可以快速查看门禁状态。

UI 是使用 [Click UI](https://clickhouse.design/click-ui)（ClickHouse 官方设计系统）构建的 React SPA。FastAPI 提供 JSON API 并托管构建后的 SPA。系统包含亮色/暗色主题切换，以及每页的 provenance 信息条，可将每个数字深链回它在 Langfuse 中的来源（“事实来源”）。

仪表盘是一个**每个 (dataset, model) 取最新运行的矩阵**：对于每个数据集，它显示每个 `metadata.model` 值的最新运行；普通模型、`--label` 变体（例如 `claude-opus-4-7-finance-expert`）和 Agent（`usecase:10k-analyst`）各占一行。每行的**主分数**按运行选择：优先 `avg_numerical_accuracy`，否则依次为 `avg_sentiment_accuracy`、`avg_groundedness`、`avg_exact_match`、`avg_completeness`，再否则取字母顺序第一个其他 `avg_*` 分数，并显示指标名。

**Threshold** 列显示该运行实际接受判断时使用的门槛，它来自运行自身元数据，所以历史运行显示当时生效的门槛，而不是今天的配置。模型运行只记录一个分数的一个门槛（`metadata.threshold` 加 `metadata.gate`，即被判断的分数），该门槛只属于这个分数，绝不属于该运行的其他评估器。Agent 运行按维度记录多个门槛（`metadata.gate_thresholds`），并要求全部通过；因此该列会显示当前行分数的门槛，并提供 `of 4 gate dims` 链接到 **Details**，其中列出每个评估器及其门槛（门禁未判断的评估器显示 `—`）。建议门槛单独计数（`of 3 gate dims + 1 advisory`）；在 **Details** 中，建议项标记为 `advisory`，未达到时显示为琥珀色而非红色，因为它不决定运行状态。没有记录门槛的运行显示 `—`，不会展示一个从未参与判断的数字。只有当图中的每次运行都使用同一指标和同一门槛时，**History** 趋势图才绘制一条统一参考线；否则每个点只在 tooltip 中显示自己的门槛。

### 运行 Portal

```bash
# 第一次：构建前端
cd portal/frontend && npm install && npm run build && cd ../..

uv run python -m portal.app                     # 默认：http://localhost:8050
PORTAL_PORT=9000 uv run python -m portal.app    # 自定义端口
```

### 前端开发（热更新）

```bash
# 终端 1：API
uv run python -m portal.app

# 终端 2：Vite 开发服务器——将 /api 代理到 :8050
cd portal/frontend && npm run dev        # http://localhost:5173
```

### 页面

| 页面 | URL | 说明 |
|---|---|---|
| Dashboard | `/` | 门禁矩阵——哪些模型在哪些数据集上通过/失败 |
| Breakdown | `/breakdown/{dataset}/{run}` | 某次运行的评估器分数（柱状图 + 表格）——每个评估器与其实际判断门槛比较；未被门禁判断时显示 `—` |
| History | `/history/{dataset}` | 所有运行的时间线与趋势图 |
| Run Detail | `/run/{dataset}/{run}` | 逐项分数和 Langfuse Trace 链接 |

### JSON API

所有页面在 `/api/` 下都有对应 JSON 端点：

| 端点 | 返回内容 |
|---|---|
| `GET /api/dashboard` | 门禁矩阵（所有数据集 × 模型） |
| `GET /api/datasets` | Portal 从 Langfuse 发现的数据集 |
| `GET /api/breakdown/{dataset}/{run}` | 一次运行的评估器分数 |
| `GET /api/history/{dataset}` | 一个数据集的全部运行（趋势数据） |
| `GET /api/run/{dataset}/{run}` | 带 Trace ID 的逐项分数 |
| `GET /api/config` | Portal 配置（Langfuse 基础 URL，用于深链） |
| `GET /docs` | API 的交互式 Swagger UI |

```bash
curl http://localhost:8050/api/dashboard | python -m json.tool
curl http://localhost:8050/api/history/certification/financebench-sample
```

Portal 使用相同的 `LANGFUSE_*` 凭据从 Langfuse 实例读取实时数据，并带 60 秒 TTL 缓存。数据集会从 Langfuse 自动发现（分页 `/api/public/v2/datasets`），并过滤为以 `certification/` 开头的名称。可以用 `PORTAL_DATASET_PREFIX` 覆盖前缀，或用逗号分隔的 `PORTAL_DATASETS` 固定明确列表。

### Portal 与 Langfuse 仪表盘的区别

Langfuse 原生覆盖了大部分评估界面。[Custom Dashboards](https://langfuse.com/docs/metrics/features/custom-dashboards) 可在 UI 中创建，也可通过代码创建（dashboards/widgets API、CLI 的 `unstable-dashboards` 资源和 MCP Server）。**Experiments** 是一等能力：可将运行与基线比较、并排比较数据项并按分数阈值过滤、跨运行绘制分数图，以及在上下文中标注或修正输出。

Portal 额外提供的是门禁视角。它按**数据集 × 模型**（或 Agent）显示 **PASS/FAIL 矩阵**，并以 `cicd/thresholds.json` 中“阈值即代码”的门槛为依据；每次运行都会记录当时的门槛，因此历史运行始终显示它真正接受判断时使用的标准。

| 需要跟踪的内容 | 位置 | 原因 |
|---|---|---|
| 分数随时间的趋势 | **Langfuse dashboard** | `scores-numeric` 视图，维度 `name`，时间粒度 `day` |
| 合规通过率 + 告警 | **Langfuse dashboard / Alerts** | `regulatory_compliance` 是 BOOLEAN 分数：`scores-boolean` 视图，`avg(value)` = 通过率；低于 1 时告警 |
| 按模型统计成本与延迟 | **Langfuse dashboard** | `observations` 视图，维度 `providedModelName` |
| 运行与基线、逐项差异、跨运行图表 | **Langfuse Experiments** | 原生实验比较 |
| 相对于记录门槛的通过/失败矩阵 | **Portal** | 每个数据集 × 模型/Agent 一行，badge = 运行的 `certification_result` 与其记录阈值的比较结果 |
| 按数据集查看门禁历史及当时门槛 | **Portal** | 使用每次运行记录的 `gate_thresholds` / `threshold`，而不是今天的值 |

**推荐的 Langfuse 仪表盘配置。** `uv run python setup_dashboards.py` 会通过 Langfuse 的 unstable dashboards API，在同一个仪表盘（“Deployment gate — quality & cost”）上创建四个组件；`--dry-run` 可打印 payload。也可以在 UI > Dashboards > New 中手工创建：

1. **Avg scores over time**——折线图，`scores-numeric` 视图，度量 `avg(value)`，维度 `name`，时间粒度 `day`
2. **Compliance true-rate**——折线图，`scores-boolean` 视图，过滤 `name=regulatory_compliance`，度量 `avg(value)`
3. **Score distribution**——柱状图，`scores-numeric` 视图，度量 `avg(value)`，维度 `name`
4. **Cost by model**——柱状图，`observations` 视图，度量 `sum(totalCost)`，维度 `providedModelName`

## 与 Langfuse 交互：CLI 与 Agent Skill

除了 Portal 和 Python 脚本，Langfuse 还提供三种工具，能显著简化对在线部署的检查和操作，尤其适合**编码 Agent**。它们都使用本仓库已有的同一套 `.env` 凭据（无需额外登录），运行门禁本身并不依赖这些工具。

- **[Langfuse CLI](https://langfuse.com/docs/api-and-data-platform/features/cli)**——完整 REST API 的轻量封装。运行后的快速检查：
  ```bash
  npx @langfuse/cli --env .env api datasets list                    # 确认黄金数据集已加载
  npx @langfuse/cli --env .env api experiments list \
      --from-start-time 2026-01-01T00:00:00Z --fields core,scores   # 门禁运行 + 结论
  npx @langfuse/cli --env .env api observations list --is-root-observation true --limit 10
  npx @langfuse/cli --env .env api prompts list                     # 托管 Prompt
  ```
  （可用 `npm i -g @langfuse/cli` 安装一次，之后通过 `langfuse api …` 调用。旧包名 `langfuse-cli` 已弃用。）
- **[Langfuse Agent Skill](https://langfuse.com/docs/api-and-data-platform/features/agent-skill)**——符合 Agent Skills 标准的 Skill（支持 Claude Code、Cursor、Windsurf 等），用于教 Agent 遵循 Langfuse 最佳实践并驱动 CLI：查询 Trace、构建数据集、迁移 Prompt。使用 `npx skills add langfuse/skills --skill "langfuse"` 添加。
- **[Langfuse MCP Server](https://langfuse.com/docs/api-and-data-platform/features/mcp-server)**——通过同一组 Key 使用 Basic Auth 的、已认证且限定项目范围的 MCP 端点：`https://<host>/api/public/mcp`（欧盟 `cloud.langfuse.com`、美国 `us.cloud.langfuse.com` 或自托管）。示例：`claude mcp add --transport http langfuse https://us.cloud.langfuse.com/api/public/mcp --header "Authorization: Basic <base64 pk:sk>"`。

这些工具如何融入端到端 Agent 工作流，请参阅 [`AGENTS.md`](AGENTS.md)。

<a id="keeping-the-evidence-current"></a>

## 持续更新证据

只运行一次的门禁，会在系统发生变化时立即过时。持续更新证据，才能把一次性测试变成 Langfuse 的 [AI 工程闭环](docs/ai-engineering-loop.md)（Trace → Monitor → Build Datasets → Experiment → Evaluate）：生产失败回流黄金数据集，Prompt 提升则自动重新运行门禁。

### Prompt 管理

Prompt 在 Langfuse 中管理，而不是全部硬编码。这样可以在不修改代码的情况下完成版本管理、A/B 测试和 Prompt 更新。

```bash
uv run python setup_prompts.py    # 创建全部 6 个托管 Prompt（模型门禁 + Agent 步骤）
```

**托管 Prompt：**

| Prompt | 变量 | 用途 |
|---|---|---|
| `financial-qa` | `{{evidence}}`、`{{question}}` | 带财报摘录的 FinanceBench 数据项（模型门禁） |
| `financial-sentiment` | `{{text}}` | Financial PhraseBank 情绪分类（模型门禁） |
| `usecase-10k-analyst-compose` | `{{question}}`、`{{operands}}`、`{{computed}}`、`{{citations}}` | 10-K Analyst Agent——compose 步骤 |
| `usecase-sentiment-classify` | `{{text}}` | Sentiment Triage Agent——classify 步骤 |
| `usecase-advisory-analyze` | `{{question}}`、`{{evidence}}` | Advisory Drafting Agent——analyze 步骤 |
| `usecase-advisory-draft` | `{{facts}}` | Advisory Drafting Agent——draft 步骤 |

只有自由文本形式的 Agent 步骤由 Langfuse 管理。10-K Analyst 的 *plan* 和 *extract* 步骤会输出 Agent 需要解析的 JSON，因此其 Prompt 保留在代码中（`agents/financial_analyst.py`），防止 UI 编辑破坏解析稳定性。

更新 Prompt：

1. 打开 Langfuse UI > **Prompts** > 选择一个 Prompt；
2. 编辑 Prompt 文本——系统会自动创建一个新的不可变版本；
3. 运行实验测试新版本；
4. 将 `production` 标签移动到新版本以完成部署；
5. 如需回滚，将 `production` 重新分配给旧版本。

实验 Runner 始终获取带 `production` 标签的版本。如果 Langfuse 不可用，会回退到硬编码默认模板，避免流水线中断。把 Prompt 提升到 `production` 还可以[在 CI 中触发门禁自动重跑](#cicd-integration)——即闭环 Edge B。

### 人工评审（标注队列）

流水线支持 human-in-the-loop 评审，用于合规签字和评估器校准——门禁负责为这些承担责任的人提供证据。

```bash
uv run python setup_score_configs.py        # 创建 human_accuracy 和 human_groundedness 分数配置
uv run python setup_annotation_queues.py    # 创建 “Certification Review” 队列
```

传入 `--queue-failures`，可将低分数据项自动路由到标注队列：

```bash
uv run python run_certification.py --dataset certification/financebench-sample \
  --model claude-haiku-4-5-20251001 --queue-failures
```

满足以下条件时，数据项会进入队列：

- 主准确率分数为 0（回答完全错误）；
- groundedness 分数低于 0.5（与源证据的关联很弱）。

**评审人工作流：**

1. 打开 Langfuse UI > **Annotation Queues** > **Certification Review**；
2. 对每个数据项，评审人可以看到原问题、模型回答和源证据；
3. 对 `human_accuracy`（Correct / Partially Correct / Incorrect）和 `human_groundedness`（Fully Grounded / Partially Grounded / Not Grounded）评分；
4. 点击 **Complete + next** 继续下一项。

人工标注有两个用途：

- **合规审计轨迹**——保留人工对门禁结果的签字证据；
- **评估器校准**——将人工分数与自动分数对比，以验证评估标准。groundedness Judge 还会写入一个 CATEGORICAL `groundedness_label`，使用与 `human_groundedness` 相同的三个类别（≥ 0.8 Fully、≥ 0.5 Partially，否则为 Not Grounded）。在 Langfuse → **Scores → Analytics** 中对比 `human_groundedness` 与 `groundedness_label`，即可获得 Judge 的混淆矩阵和 Cohen's κ。

评审人还可以在 Langfuse 的 Trace 上写入 **Correction**（修正后的输出）。除非显式传入 `--expected`，`promote_trace_to_dataset.py` 会将 Correction 用作回灌数据项的期望答案（`metadata.expected_source = "correction"`），因此“标注 → 修正 → 回灌”流程无需在 CLI 中再次输入答案。

### 生产监控

**主路径：Langfuse 托管评估器。** `setup_evaluators.py` 会在 Langfuse 内部创建相同检查，对在线流量执行，无需 cron：

- `regulatory_compliance`：代码评估器（Python，BOOLEAN）；
- `completeness`：代码评估器（Python，NUMERIC）；
- `groundedness`：LLM-as-a-Judge 评估器（NUMERIC 0～1）。

两条评估规则选择生产根 Observation（`isRootObservation = true`、`environment = production`）：代码评估器评估全部匹配项，Judge 评估其中 20%。代码评估器源码由 `evaluators.py`（`compliance_violations`、`completeness_score`、`PROHIBITED_PHRASES`）**自动生成**，因此离线门禁与在线监控不会漂移；`tests/test_setup_evaluators.py` 会用离线评估器验证生成的源码。

```bash
uv run python setup_evaluators.py --dry-run   # 打印所有 payload
uv run python setup_evaluators.py             # 创建评估器 + DISABLED 规则
uv run python setup_evaluators.py --enable    # 或直接创建并启用规则
```

然后在 Langfuse UI 中添加 **Alert**：source = scores，score = `regulatory_compliance`，1 小时窗口内 `avg` 低于 1 时，通过 Slack、Webhook 或 GitHub Actions 通知。详见 [`docs/loop-activation-checklist.md`](docs/loop-activation-checklist.md)。Judge 需要在 Langfuse 项目中配置 LLM Connection（人工步骤）。在自托管 Langfuse 上，Python 代码评估器需要 `aws-lambda` code-eval dispatcher（`insecure-local` 只运行 TypeScript/JavaScript）。

**备选路径：`monitor_production.py`。** 对于没有 code-eval dispatcher 的自托管栈或隔离环境，它可以通过 cron 运行相同的确定性检查。

```text
选项：
  --hours N              回看 N 小时（默认：1）
  --tags TAG [TAG ...]   按标签过滤 Trace
  --trace-name NAME      按名称过滤 Trace
  --limit N              最多处理的 Trace 数（默认：100）
  --queue-violations     将违规 Trace 路由到人工评审队列（反馈边）
  --dry-run              仅预览，不发布分数
```

使用 cron 定时运行，以便实时发现问题：

```bash
# 每 15 分钟检查最近一小时的生产 Trace
*/15 * * * * cd /path/to/repo && uv run python monitor_production.py --hours 1 --tags production

# 或按应用的 Trace 名过滤
*/15 * * * * cd /path/to/repo && uv run python monitor_production.py --trace-name my-finance-app
```

脚本获取指定时间窗口、标签或 Trace 名下的近期 Trace，跳过已经有合规分数的 Trace（保证幂等），对每一项运行 `regulatory_compliance` 和 `completeness` 评估器，把分数写回 Langfuse；如果发现合规违规则以退出码 1 结束，从而可以接入告警系统。

脚本将生产 Trace 读取为其**根 Observation**（`GET /api/public/v2/observations?isRootObservation=true`）——在 Langfuse v4 中，Trace 的输入输出就是根 Observation 的输入输出；它会跳过部署门禁实验的 Trace（环境 `sdk-experiment`，可传 `--include-experiments` 将其也纳入评分），并通过 SDK 把分数写到根 Observation 上。

对于主观质量监控（groundedness、helpfulness），应在 Observation 级使用 Langfuse 托管的 **LLM-as-a-Judge**：一个**评估器**定义如何评分（Prompt、模型、输出定义），一条**规则**定义要评分的 Observation（例如 `isRootObservation = true`、`environment = production`，并使用采样率控制成本）。可在 UI 的 **Evaluators** 中创建，也可以通过 `POST /api/public/v2/evaluators` 和 `/v2/evaluation-rules` 创建。Langfuse Cloud 在 2026-11-16 切换到 v4 后，旧的 Trace 级评估器不再产生结果。参阅 [Langfuse LLM-as-a-Judge 文档](https://langfuse.com/docs/evaluation/evaluation-methods/llm-as-a-judge)。

### `promote_trace_to_dataset.py`——反馈闭环（Trace → 黄金数据集）

该脚本闭合 [AI 工程闭环](docs/ai-engineering-loop.md) 中“观测 → 开发”的反馈边：把被标记/已评审的生产 Trace 提升为黄金数据项，使下一次运行对其进行回归测试。流程由人把关——它会捕获 Trace 的**输入**，但正确答案必须由评审人提供（或通过 `--expected` 传入），绝不会把可疑的 Trace 输出复制为真值。操作是幂等的（数据项 ID 为 `prod-<traceId>`，并通过 `source_trace_id` 关联来源）。

```text
选项：
  --dataset NAME         目标数据集（必填，例如 certification/financebench-sample）
  --trace-id ID          要提升的 Trace（可重复传入）
  --from-queue           提升 “Certification Review” 队列中的所有 Trace
  --expected TEXT        要保存的正确答案（否则标记 needs_expected_review）
  --note TEXT            存入数据项元数据的可选备注
  --dry-run              仅预览，不创建数据项
```

<a id="scriptsrecert_for_promptpy--feedback-loop-prompt-promotion--re-run-the-gate"></a>

### `scripts/recert_for_prompt.py`——反馈闭环（Prompt 提升 → 重新运行门禁）

该脚本闭合 [AI 工程闭环](docs/ai-engineering-loop.md) 中“发布 → 重新门禁”的反馈边：给定一个刚刚在 Langfuse 中被提升的 Prompt 名，它会用 `--ci` 重新运行依赖该 Prompt 的门禁，使发生回退的 Prompt 提升在 CI 中失败，而不是静默上线。它由 [`prompt-recert.yml` 工作流](#cicd-integration)在 `repository_dispatch` 时调用，也可在本地运行。

```text
选项：
  --prompt-name NAME     发生变化的 Prompt（必填）——按名称路由到目标门禁
  --model MODEL          传递给重跑任务的可选模型覆盖值
```

**Prompt → 门禁路由：**

| 被提升的 Prompt | 重新运行的门禁 |
|---|---|
| `financial-qa` | `certification/financebench-sample` 上的模型门禁 |
| `financial-sentiment` | `certification/fpb-sample` 上的模型门禁 |
| `usecase-10k-analyst-compose` | `certification/financebench-sample` 上的 `10k-analyst` |
| `usecase-sentiment-classify` | `certification/fpb-sample` 上的 `sentiment-triage` |
| `usecase-advisory-analyze`、`usecase-advisory-draft` | `certification/advisory-adversarial` 上的 `advisory-draft` |

未映射的 Prompt 名不执行任何操作（退出码 0）；门禁失败则退出码为 1。

<a id="cicd-integration"></a>

### CI/CD 集成

使用 `--ci` 可以在门禁失败时让进程失败（退出码 1）：

```bash
uv run python run_certification.py \
  --dataset certification/financebench-sample \
  --model claude-haiku-4-5-20251001 \
  --threshold 0.85 \
  --ci
```

也可以按 CI 的实际方式运行 Agent 门禁（`--ci`，不将失败项送入队列）：

```bash
make ci-gate USE_CASE=advisory-draft
```

#### 质量门槛即代码

所有门禁门槛——三个 Agent 门禁与模型认证的默认阈值——都位于 [`cicd/thresholds.json`](cicd/thresholds.json)，而不是 Agent 模块中。因此，放宽门槛会成为 Pull Request 中可评审的 Diff。[`cicd/README.md`](cicd/README.md) 解释了硬/松门槛划分、尚未覆盖的内容（persona 和语气未被门禁），以及为什么缺失配置会抛错而不是使用默认值：`usecase_certification_gate` 对收到的必选维度执行 `all()`，所以空门禁或所有门槛都是建议项的门禁会把一切认证为 PASSED。`tests/test_gate_thresholds.py` 会在阈值文件与 Agent 注册表漂移，或者某个门禁维度没有评估器产出时失败。

#### Pytest 门禁

测试套件分为**离线套件**（无需凭据，每个 PR 都运行）和**在线门禁**（针对 Langfuse 运行真实门禁）：

**离线套件**——纯函数/Monkeypatch 单元测试，不需要 Langfuse 或 API Key。它通过 [`tests.yml` 工作流](#github-actions)真正阻断 PR：

```bash
uv run pytest --ignore=tests/test_certification.py -v
```

| 测试文件 | 覆盖内容 |
|---|---|
| `test_usecase_foundation.py` | 多维门禁、`tool_use_correctness` 轨迹评估器、Agent 注册表 |
| `test_financial_analyst.py` | 10-K Analyst：`safe_eval` 计算器安全、宽松 JSON 解析、计算器路由 |
| `test_sentiment_triage.py` | Sentiment Triage：标签/置信度解析、驱动短语、置信度路由 |
| `test_advisory_draft.py` | Advisory Draft：三 Span 流程、合规自检、合规硬失败门禁 |
| `test_promote_trace_to_dataset.py` | 闭环 Edge A：确定性 `prod-<traceId>` ID、禁止复制可疑输出的保护逻辑 |
| `test_recert_for_prompt.py` | 闭环 Edge B：Prompt → 目标路由、`--ci` 接线、与 `setup_prompts.py` 的漂移保护、跳过与通过的 Job 摘要 |
| `test_gate_thresholds.py` | 质量门槛：注册表 ↔ `cicd/thresholds.json` 漂移、每个门禁维度都有评估器、缺失门槛抛错而不是把一切认证通过 |
| `test_portal_thresholds.py` | 读取运行记录门槛（Portal + 证据包）：模型门槛只属于它判断的分数；不虚构或回退门槛；结论使用未舍入均值；遗留推断与 `select_evaluators` 保持一致；建议门槛显示 MET/MISS 但绝不影响状态或 Gate 行 |

**在线门禁**——`tests/test_certification.py` 会运行真实实验并断言通过/失败。它需要 Langfuse 凭据、`ANTHROPIC_API_KEY` 和已播种的数据集；在 `certification.yml` 工作流而不是 PR 中运行：

```bash
# 运行全部在线门禁测试
uv run pytest tests/test_certification.py -v

# 只运行 FinanceBench 测试
uv run pytest tests/test_certification.py -v -k financebench

# 通过环境变量覆盖模型和阈值
CERT_MODEL=claude-sonnet-4-6 CERT_THRESHOLD=0.90 uv run pytest tests/test_certification.py -v
```

<a id="github-actions"></a>

#### GitHub Actions

流水线提供三个用于自动化闭环的工作流。`tests.yml` 开箱即用（不需要 Secret）；另外两个在线工作流会在向仓库添加下方四个 Secret 后启用。

**`tests.yml`——离线测试。** 在每个 Pull Request 和每次推送到 `main` 时运行离线套件（`pytest --ignore=tests/test_certification.py`），无需 Secret。

**`certification.yml`——在线门禁。**

- **触发方式：** 手工 dispatch（可配置模型/阈值），以及当评估器、Runner、Agent（`agents/**`、`run_usecase_certification.py`、`cert_common.py`）、Prompt 或依赖配置变化并推送到 `main` 时；
- **Job：** FinanceBench 与 FPB 模型门禁，以及 `10k-analyst` Agent 门禁（`certification/financebench-sample`、`--ci`）并行运行；随后运行在线 Pytest 门禁（`if: always()`，所以即使某个门禁 Job 失败也会报告）；
- **所需 Secret：** `LANGFUSE_PUBLIC_KEY`、`LANGFUSE_SECRET_KEY`、`LANGFUSE_BASE_URL`、`ANTHROPIC_API_KEY`。

手工触发：**Actions** > **LLM Certification** > **Run workflow** > 选择模型和阈值。

**`prompt-recert.yml`——Prompt 提升后重跑门禁（闭环 Edge B）。**

- **触发方式：** 事件类型为 `langfuse-prompt-update` 的 `repository_dispatch`——当 Prompt 版本获得 `production` 标签时由 Langfuse Prompt Automation 触发（Langfuse 侧一次性设置见 [`docs/loop-activation-checklist.md`](docs/loop-activation-checklist.md)）；也支持手工 `workflow_dispatch`（`prompt_name` 必填，`model` 默认为 `claude-sonnet-4-6`），因此无需 Webhook 也能测试；
- **行为：** 运行 [`scripts/recert_for_prompt.py`](#scriptsrecert_for_promptpy--feedback-loop-prompt-promotion--re-run-the-gate)，将被提升的 Prompt 映射到目标门禁，并用 `--ci` 重跑。若 dispatch payload 的标签不包含 `production`，保护逻辑会跳过（Langfuse 会 dispatch 两次——一次获得标签，一次失去标签）；
- **所需 Secret：** 与 `certification.yml` 相同的四项。

两个在线工作流都使用带 `cancel-in-progress` 的 `concurrency` 组，分别按 ref（`certification.yml`）或变化的 Prompt 名（`prompt-recert.yml`）建立 Key。否则，在 Langfuse UI 中快速修改多次标签，会排队执行多次完整门禁；每次都会产生真实 LLM 成本，却只是在回答一个已经被更新运行取代的问题。

`recert_for_prompt.py` 还会把结论写入 `$GITHUB_STEP_SUMMARY`，无需打开日志即可从运行摘要中读取。这个设计对**跳过**场景尤其重要：没有映射到门禁目标的 Prompt 会以 0 退出，在界面上显示绿色勾号，看起来与“门禁运行且通过”完全相同。摘要会明确说明没有运行任何门禁，因此该 Job 对质量不作结论。

这些工作流与部署方式无关——可以让 `LANGFUSE_BASE_URL` 指向 Cloud，也可以指向 GitHub Actions Runner 能访问的自托管 Langfuse（例如公网可解析主机名、VPN/隧道，或网络内部的自托管 Runner）。

## 治理框架期望什么

本项目不会颁发证书。它生成的是治理流程赖以运行的**证据**——版本化运行、Trace、分数、有文档记录的人工评审和监控历史。这些证据可以映射到主要治理框架的要求：

| 框架 | 期望内容 | 门禁提供的能力 |
|---|---|---|
| **Fed SR 26-2 / OCC 2026-13**（美国模型风险） | 生成式和 Agentic AI 不在正式范围内，但银行仍须使用适当控制进行治理 | 内部 LLM 门禁的设计参考——版本化运行、评估器和阈值 |
| **PRA SS1/23**（英国模型风险） | 针对 AI/ML 的治理、验证和持续监控 | 版本化门禁运行，作为验证与重新验证证据 |
| **EU AI Act** | 对信用评估等高风险用途要求日志、人工监督和上市后监控 | Trace 用于日志，标注队列用于有记录的人工评审，`monitor_production.py` 用于上市后流程 |
| **DORA**（欧盟 ICT 风险） | 模型 API 可用性、退化和事件的运营证据 | API 行为和失败的 Trace 与监控历史 |
| **NIST AI RMF**（自愿） | 部署前有文档记录的测试，以及使用期间的监控 | 数据集、评估器、门禁和运行历史支持 *Measure*；评审队列支持 *Manage* |

机构仍然负责那些必须由人承担的部分：适用性判断、政策、独立质疑、审批和法律合规。流水线为这些决策提供底层的可复现证据，但不会代替人作决定。

<a id="adapting-it-to-your-organization"></a>

## 适配到你的组织

FinanceBench + Financial PhraseBank 是一套可运行的参考实现。适配方法如下。

### 引入自己的黄金数据集

用机构自己的黄金问答对替换 FinanceBench。`setup_datasets.py` 接受包含 input、expected output 和 metadata 字段的任意 JSON 文件：

```json
[
  {
    "question": "What was the total revenue for FY2023?",
    "answer": "$52.6 billion",
    "justification": "From the income statement, line: Total Revenue"
  }
]
```

然后使用 `setup_datasets.py` 加载，或直接使用 Langfuse SDK：

```python
from langfuse import get_client
langfuse = get_client()
langfuse.create_dataset(name="my-custom-dataset")
langfuse.create_dataset_item(
    dataset_name="my-custom-dataset",
    input={"question": "What was the total revenue for FY2023?"},
    expected_output={"answer": "$52.6 billion"},
)
```

### 添加领域专用评估器

为组织的术语、格式要求或监管约束增加检查。在 `evaluators.py` 中添加一个遵循 Langfuse 签名的函数，然后在 `run_certification.py` 中导入并加入 `select_evaluators()`：

```python
from langfuse import Evaluation

def my_custom_evaluator(*, input, output, expected_output, **kwargs):
    # 在这里编写评估逻辑
    score = 1.0 if "some condition" else 0.0
    return Evaluation(name="my_metric", value=score, comment="Reason")
```

### 扫描 Prompt 变体

`--system-prompt-file` + `--label` 组合可以让**同一个模型使用领域专用 system Prompt**执行门禁，并在仪表盘上与基线并排比较。这**不是**微调，而是 Prompt Engineering；但对许多企业应用而言，提升幅度可能相当。

仓库附带一个变体：**`prompts/finance_expert.md`**——一个面向高级金融分析师的 system Prompt，包含四步思维链框架（识别指标 → 引用证据 → 应用公式 → 给出结果），以及 FinanceBench 专用注意事项（单位、符号约定、行项目混淆）。在 `financebench-sample` 上，它将 Opus 4.7 从 95% 提升到 100%；在 `financebench-v1` 上，它使同一模型从 FAILED 变为 PASSED。

```bash
# 基线
uv run python run_certification.py --dataset certification/financebench-v1 --model claude-opus-4-7

# Finance Expert 变体
uv run python run_certification.py --dataset certification/financebench-v1 --model claude-opus-4-7 \
    --system-prompt-file prompts/finance_expert.md --label finance-expert
```

仪表盘按 `metadata.model` 分组，所以两个运行会显示为 `claude-opus-4-7` 和 `claude-opus-4-7-finance-expert`——两行独立数据，方便比较。

> **注意：** Agent Prompt（`usecase-*`）已经通过 `setup_prompts.py` 交给 Langfuse 管理——可版本化、可提升，并已接入[门禁自动重跑](#scriptsrecert_for_promptpy--feedback-loop-prompt-promotion--re-run-the-gate)。`prompts/finance_expert.md` 变体刻意保留为文件：它是通过 `--system-prompt-file` + `--label` 扫描的实验时输入，而不是已部署 Prompt。如果它最终进入生产工具，应将其提升到 Langfuse Prompt Management。

### 注册自定义 Agent 并将门禁接入 CI

在 Agent 注册表中注册自己的 Agent、步骤和门禁阈值，并把门禁失败接入构建流水线，使模型与 Prompt 审批成为部署流程的一部分（见 [CI/CD 集成](#cicd-integration)）。

### 使用自定义 LLM 网关

```bash
# 通过环境变量
export LLM_BASE_URL="https://your-gateway.internal/v1"
export LLM_API_KEY="your-key"

# 或通过 CLI 参数
uv run python run_certification.py --endpoint https://your-gateway.internal/v1 --dataset ...
```

### 扩展更多金融数据集

[Open FinLLM Leaderboard](https://huggingface.co/spaces/TheFinAI/Open-Financial-LLM-Leaderboard) 提供 40 多个金融数据集。推荐的下一批候选：

| 数据集 | 关注点 | HuggingFace ID |
|---|---|---|
| FLARE FinQA | 基于金融表格的数值推理 | `ChanceFocus/flare-finqa` |
| FLARE FOMC | 货币政策立场分类 | `ChanceFocus/flare-fomc` |
| Credit Risk (German) | 信用评分 | `ChanceFocus/flare-german` |
| Credit Risk (Taiwan) | 信用风险评估 | `TheFinAI/cra-taiwan` |
| TATQA | 表格 + 文本混合问答 | `ChanceFocus/flare-tatqa` |

## 环境变量

| 变量 | 是否必需 | 默认值 | 说明 |
|---|---|---|---|
| `LANGFUSE_PUBLIC_KEY` | 是 | — | Langfuse 项目公钥 |
| `LANGFUSE_SECRET_KEY` | 是 | — | Langfuse 项目密钥 |
| `LANGFUSE_BASE_URL` | 否 | `https://cloud.langfuse.com` | Langfuse 实例 URL——Cloud（欧盟/美国）或自托管（通常为 `http://localhost:3000`）。见[选择 Langfuse 部署方式](#choose-your-langfuse-deployment) |
| `LANGFUSE_HOST` | 否 | — | `LANGFUSE_BASE_URL` 的 fallback（脚本和 Portal 都支持） |
| `ANTHROPIC_API_KEY` | Claude 模型需要 | — | Claude 模型使用的 Anthropic API Key（AI Agent 和 LLM Judge 必需） |
| `LLM_API_KEY` | OpenAI 模型需要 | — | OpenAI-compatible API Key（回退到 `OPENAI_API_KEY`） |
| `LLM_BASE_URL` | 否 | `https://api.openai.com/v1` | LLM API 基础 URL |
| `LLM_MODEL` | 否 | `claude-sonnet-4-6` | 默认执行门禁的模型 |
| `JUDGE_MODEL` | 否 | `claude-sonnet-4-6` | LLM-as-a-Judge 评估器使用的模型 |
| `PORTAL_PORT` | 否 | `8050` | Certification Portal HTTP 端口 |
| `PORTAL_DATASET_PREFIX` | 否 | `certification/` | Portal 在 Langfuse 中发现数据集时使用的名称前缀 |
| `PORTAL_DATASETS` | 否 | — | 逗号分隔的数据集列表——将 Portal 固定为这些数据集（跳过发现） |
| `CERT_MODEL` | 否 | `claude-haiku-4-5-20251001` | 在线 Pytest 门禁使用的模型（仅 `tests/test_certification.py`） |
| `CERT_THRESHOLD` | 否 | `0.85` | 在线 Pytest 门禁阈值（仅 `tests/test_certification.py`） |
| `ADVISORY_TEMPT_NONCOMPLIANT` | 否 | — | 设为 `1`，诱导 `advisory-draft` Agent 生成禁用表达——用于演示合规硬失败门禁 |

OTel/Langfuse flush 调优变量（`OTEL_BSP_*`、`LANGFUSE_FLUSH_*`）详见[故障排查：长任务卡住](#troubleshooting-hangs-on-long-runs)。

## 常见问题

### 门禁评分如何工作？

流水线让待测模型对 Langfuse 数据集中的每个数据项执行，然后使用一组评估器为每个回答打分。分数会在运行级聚合，**门禁**（`certification_gate`）检查主准确率指标是否达到配置阈值（默认 85%）。运行最终为 PASSES 或 FAILS。Agent 门禁原理相同，但使用**多维门禁**：为 Agent 注册的每个维度（例如准确率、groundedness、合规性、工具使用正确性）都必须同时达到自己的阈值。

### 能否对整个 AI Agent 而不只是模型执行门禁？

可以——使用 [`run_usecase_certification.py`](#gating-the-whole-agent)。项目内置三个已注册 Agent：`10k-analyst`（带计算器的 SEC 文件事实型问答）、`sentiment-triage`（带人工分流的情绪分类）、`advisory-draft`（受合规门禁约束的客户文案）。每个 Agent 都定义自己的 Trace 形态和多维门禁；在仪表盘中，它们显示为与普通模型行并列的 `usecase:<name>` 行。

### 评估器是确定性的，还是基于 LLM？

两者都有。流水线使用**确定性评估器**（正则、字符串匹配、数字提取，以及 Agent 运行的工具轨迹检查）处理客观指标，并使用 **LLM-as-a-Judge 评估器**（`groundedness_evaluator`）评估主观质量。确定性评估器快速、低成本、可复现；LLM Judge 能发现启发式规则漏掉的定性失败，例如模型是否虚构了一个碰巧正确的数字，或者是否真正使用了源文档。

### 可以使用不同的 Judge 模型吗？

可以。设置 `JUDGE_MODEL` 环境变量：

```bash
JUDGE_MODEL=claude-haiku-4-5-20251001 uv run python run_certification.py --dataset ...
```

使用更便宜、更快的 Judge 可以降低成本，但也可能降低评估质量。对于金融评估，建议使用能力至少与 `claude-sonnet-4-6` 相当的模型。

### 数据项级评估器和运行级评估器有什么区别？

- **数据项级**评估器分别为每个数据项打分（例如“这个答案是否匹配期望数字？”）；
- **运行级**评估器跨所有数据项聚合（例如“平均准确率是多少？”或“这次运行是否通过门禁？”）。

### 支持哪些数据集？

项目包含两个金融基准和一个内置对抗集：

| 数据集 | 数据量 | 关注点 |
|---|---:|---|
| [FinanceBench](https://huggingface.co/datasets/PatronusAI/financebench) | 10（样本）/ 150（全量） | 基于 SEC 文件的金融问答 |
| [Financial PhraseBank](https://huggingface.co/datasets/ChanceFocus/en-fpb) | 10（样本）/ 约 4,850（全量） | 金融情绪分类 |
| `advisory-adversarial`（内置） | 10 | 诱导不同类型非合规投顾表达的简报——用于演示合规硬门禁 |

你也可以添加自定义数据集——参阅[适配到你的组织](#adapting-it-to-your-organization)。




## 官方【】参考资料
- ----配套项目[clickhouse-llm-observability](https://github.com/doneyli/clickhouse-llm-observability)——包含 LibreChat、Langfuse 和 ClickHouse 的完整 LLM 可观测栈（监控、追踪、调试）
- [通过 SDK 使用 Langfuse Experiments](https://langfuse.com/docs/evaluation/experiments/experiments-via-sdk)
- [Langfuse Datasets](https://langfuse.com/docs/evaluation/experiments/datasets)
- [Langfuse Custom Scores](https://langfuse.com/docs/scores/custom)
- [FinanceBench 论文](https://arxiv.org/abs/2311.11944)
- [Open FinLLM Leaderboard](https://huggingface.co/spaces/TheFinAI/Open-Financial-LLM-Leaderboard)

