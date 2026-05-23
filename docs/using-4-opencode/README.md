# OpenSRE Using Book — OpenCode 版

这是一套面向"使用 OpenCode CLI 作为 LLM 推理后端"的 OpenSRE 说明书，目标不是解释架构，而是回答：

1. 如何在当前仓库里把 OpenSRE 配起来。
2. 如何使用 **OpenCode CLI** 作为 LLM 后端。
3. 如何把 **Elasticsearch 日志** 或 **Kafka 错误消息队列** 接进调查流程。
4. 如何把 **GitHub 代码库** 接进调查流程。
5. 如何一步一步跑通一次完整的 investigation pipeline。

本书基于对以下内容的联合扫描整理：

- `README.md`
- `docs/`
- `app/cli/`
- `app/integrations/`
- `app/services/`
- `app/tools/`
- `app/nodes/`

以及 `docs/using-book/`（Kimi 版说明书）的章节结构。

## 与 `docs/using-book/` 的差异

| 维度 | using-book (Kimi) | using-4-opencode (本书) |
|------|------------------|----------------------|
| LLM 后端 | `Kimi Code CLI` | `OpenCode CLI` |
| 数据源 | ES / OpenSearch | ES / OpenSearch **+ Kafka** |
| 认证探测 | `kimi login status` | `opencode auth list` |

## 阅读顺序

1. [00-plan.md](./00-plan.md)
2. [01-overview.md](./01-overview.md)
3. [02-install-and-opencode.md](./02-install-and-opencode.md)
4. [03-github-and-datasources.md](./03-github-and-datasources.md)
5. [04-alert-payload-and-examples.md](./04-alert-payload-and-examples.md)
6. [05-step-by-step-runbook.md](./05-step-by-step-runbook.md)
7. [06-pipeline-explained.md](./06-pipeline-explained.md)
8. [07-troubleshooting.md](./07-troubleshooting.md)
9. [08-target-repo-playbook.md](./08-target-repo-playbook.md)
10. [09-verification-checklist.md](./09-verification-checklist.md)

## 本书采用的跑通场景

本书围绕下面这个具体场景展开：

- **LLM**: OpenCode CLI（模型如 `anthropic/claude-opus-4.7`）
- **日志后端**: Elasticsearch / OpenSearch 兼容接口（方案 A）或 Kafka 集群（方案 B）
- **代码仓库**: `https://github.com/woyao-cloud/ollama-03-02-claude-prompt02-template.git`
- **运行方式**: 仓库根目录下使用 `uv run opensre ...`

## 一页结论

基于当前代码，最稳妥的跑通路径是：

1. 用 `opensre onboard` 只完成 OpenCode provider 选择。
2. 用项目根目录 `.env` 明确写入 GitHub MCP 和数据源变量。
3. 用一个显式带有 `repo_url`、`github_owner`、`github_repo`、数据源查询参数的告警 JSON 启动调查。
4. 用 `uv run opensre investigate -i <alert.json>` 做真正的端到端验证。

## 示例文件

可直接参考本目录下的示例：

- [examples/env.opencode-github-opensearch.example](./examples/env.opencode-github-opensearch.example)
- [examples/env.opencode-github-kafka.example](./examples/env.opencode-github-kafka.example)
- [examples/elasticsearch-github-opencode-alert.json](./examples/elasticsearch-github-opencode-alert.json)
- [examples/kafka-github-opencode-alert.json](./examples/kafka-github-opencode-alert.json)
- [examples/ollama-template-repo-alert.json](./examples/ollama-template-repo-alert.json)
