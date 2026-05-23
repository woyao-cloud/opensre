# OpenSRE Using Book

这是一套面向“实际使用”的 OpenSRE 说明书，目标不是解释架构，而是回答：

1. 如何在当前仓库里把 OpenSRE 配起来。
2. 如何使用 `Kimi Code CLI` 作为 LLM 后端。
3. 如何把“已知 Elasticsearch 日志 + 已知 GitHub 代码库”接进调查流程。
4. 如何一步一步跑通一次完整的 investigation pipeline。

本书基于对以下内容的联合扫描整理：

- `README.md`
- `docs/`
- `app/cli/`
- `app/integrations/`
- `app/services/`
- `app/tools/`
- `app/nodes/`

## 阅读顺序

1. [00-plan.md](./00-plan.md)
2. [01-overview.md](./01-overview.md)
3. [02-install-and-kimi.md](./02-install-and-kimi.md)
4. [03-github-and-elasticsearch.md](./03-github-and-elasticsearch.md)
5. [04-alert-payload-and-examples.md](./04-alert-payload-and-examples.md)
6. [05-step-by-step-runbook.md](./05-step-by-step-runbook.md)
7. [06-pipeline-explained.md](./06-pipeline-explained.md)
8. [07-troubleshooting.md](./07-troubleshooting.md)
9. [08-target-repo-playbook.md](./08-target-repo-playbook.md)
10. [09-verification-checklist.md](./09-verification-checklist.md)

## 本书采用的跑通场景

本书围绕下面这个具体场景展开：

- LLM: `Kimi Code CLI`
- 日志后端: Elasticsearch / OpenSearch 兼容接口
- 代码仓库: `https://github.com/woyao-cloud/ollama-03-02-claude-prompt02-template.git`
- 运行方式: 仓库根目录下使用 `uv run opensre ...`

## 一页结论

基于当前代码，最稳妥的跑通路径是：

1. 用 `opensre onboard` 只完成 `Kimi` provider 选择。
2. 用项目根目录 `.env` 明确写入 GitHub MCP 和 `OPENSEARCH_*` 变量。
3. 用一个显式带有 `repo_url`、`github_owner`、`github_repo`、`elasticsearch_query` 的告警 JSON 启动调查。
4. 用 `uv run opensre investigate -i <alert.json>` 做真正的端到端验证。

之所以这样选，是因为当前代码里：

- `Kimi` 已被正式注册为 CLI provider。
- GitHub MCP 的环境变量路径完整可用。
- Elasticsearch 日志在规划层实际通过 `opensearch` 集成入口接入，环境变量是 `OPENSEARCH_*`。
- `investigations verify opensearch` 目前只验证“是否配置了 URL”，真正能不能取到日志，要以实际调查运行为准。

## 示例文件

可直接参考本目录下的示例：

- [examples/env.kimi-github-opensearch.example](./examples/env.kimi-github-opensearch.example)
- [examples/elasticsearch-github-kimi-alert.json](./examples/elasticsearch-github-kimi-alert.json)
- [examples/ollama-template-repo-alert.json](./examples/ollama-template-repo-alert.json)
