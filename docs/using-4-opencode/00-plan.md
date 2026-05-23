# 写作计划

## 目标

这本 Using Book 基于 OpenSRE 的现有代码与使用手册，面向 **OpenCode CLI** 作为 LLM 推理后端的场景，重新整理一套完整的使用说明书。

它要解决的问题是：

- 如何在当前仓库中安装并启动 OpenSRE。
- 如何把 **OpenCode CLI** 配成可用的 LLM provider。
- 如何接入 **Elasticsearch / OpenSearch** 日志或 **Kafka** 错误消息队列。
- 如何接入 **GitHub MCP** 访问指定代码仓库。
- 如何准备输入告警，让 OpenSRE 走完整条 investigation pipeline。
- 如何验证这次运行确实用到了日志 / Kafka 和代码仓库上下文。

## 与 `docs/using-book/` 的差异

| 维度 | using-book (Kimi) | using-4-opencode |
|------|------------------|------------------|
| LLM 后端 | `Kimi Code CLI` | `OpenCode CLI` |
| LLM 模型格式 | `kimi-k2.5` | `anthropic/claude-opus-4.7` 等 |
| 日志数据源 | Elasticsearch / OpenSearch | Elasticsearch / OpenSearch **+ Kafka** |
| 认证探测 | `kimi login status` + `KIMI_API_KEY` | `opencode auth list` + 环境变量 |
| 安装方式 | `uv tool install kimi-cli` | `brew / choco install opencode` |

## 场景约束

本书固定采用以下场景：

- 本地从当前仓库源码运行，不依赖远端部署。
- LLM provider 使用 `opencode`。
- 日志来自 Elasticsearch / OpenSearch 兼容接口 **或** Kafka 错误消息队列。
- 代码仓库固定为 `woyao-cloud/ollama-03-02-claude-prompt02-template`。

## Key 发现

### 1. `opencode` 是正式支持的 CLI provider

在 `app/config.py`、`app/cli/wizard/config.py`、`app/integrations/llm_cli/registry.py` 里，`opencode` 都被注册为正式 provider，支持 `opensre onboard` 向导。

### 2. OpenCode 是多 provider 网关

OpenCode 本身支持多个 LLM 供应商（Anthropic、OpenAI、OpenRouter 等）。模型名格式为 `provider/model`，例如 `anthropic/claude-opus-4.7`。认证机制覆盖 `auth.json` 和 `ANTHROPIC_API_KEY` / `OPENAI_API_KEY` 等环境变量。

### 3. Elasticsearch 入口实际是 `opensearch`

同 using-book，调查链路的 `detect_sources()` 会把 `alert_source=elasticsearch` 映射到 `available_sources["opensearch"]`，需要配置 `OPENSEARCH_*` 环境变量。

### 4. Kafka 是独立集成

Kafka 有自己的 `alert_source=kafka` 检测路径，环境变量为 `KAFKA_BOOTSTRAP_SERVERS` 等。Kafka 工具独立注册：`get_kafka_topic_health` 和 `get_kafka_consumer_group_lag`。

## 章节设计

### `01-overview.md`

说明最终要跑通什么、需要哪些前置条件、哪些不是本书范围。

### `02-install-and-opencode.md`

说明：
- 如何安装项目依赖
- 如何安装 OpenCode CLI
- 如何登录 opencode
- 如何通过 `opensre onboard` 选择 `opencode`

### `03-github-and-datasources.md`

说明：
- GitHub MCP 怎么配置
- Elasticsearch / OpenSearch 怎么配置
- Kafka 怎么配置
- 当前代码对各数据源的真实支持边界

### `04-alert-payload-and-examples.md`

说明：
- 一个"对 ES 场景足够友好"的告警 JSON 该怎么写
- 一个"对 Kafka 场景足够友好"的告警 JSON 该怎么写
- 哪些字段会帮助 OpenSRE 发现数据源和 GitHub 来源

### `05-step-by-step-runbook.md`

给出完全可照着执行的操作步骤。

### `06-pipeline-explained.md`

解释这一套配置为什么能跑通，以及运行时大概会走哪些工具。

### `07-troubleshooting.md`

列出最常见失败点和当前代码的几个重要限制。

### `08-target-repo-playbook.md`

专门补一个面向目标仓库的定制章节。

### `09-verification-checklist.md`

补一章手工验证清单。

## 输出形式

输出保存到 `docs/using-4-opencode/` 目录下，采用多文件 Markdown 结构，并附：

- `.env` 示例（ES + GitHub 版、Kafka + GitHub 版）
- 告警 JSON 示例（ES 版、Kafka 版）
