# 使用目标与前提

## 最终目标

跑通下面这条最小但完整的本地链路：

1. OpenSRE 从 `Kimi Code CLI` 获取推理能力。
2. OpenSRE 从 Elasticsearch / OpenSearch 兼容接口拉取日志。
3. OpenSRE 从 GitHub MCP 访问指定代码仓库。
4. OpenSRE 读取一个包含日志查询提示和仓库信息的告警 JSON。
5. OpenSRE 完成一次本地 investigation，并输出 RCA 结果。

## 本书假设你已经有

- 当前仓库源码
- Python / `uv`
- 一个可用的 `Kimi Code CLI` 账号或 API key
- 一个可访问的 GitHub token
- 一个可访问的 Elasticsearch / OpenSearch 兼容 endpoint

## 本书不假设你已经有

- 远端 LangGraph 部署
- Slack、Grafana、Datadog 等其他集成
- 完整的生产环境告警平台

## 本书的关键判断

基于代码扫描，这次跑通流程最关键的三个事实是：

### 1. `kimi` 是正式支持的 LLM provider

在 `app/config.py`、`app/cli/wizard/config.py`、`app/integrations/llm_cli/registry.py` 里，`kimi` 都是正式 provider，而不是临时 hack。

### 2. GitHub 需要 MCP 集成 + 仓库定位信息

GitHub 工具并不会凭空知道你想查哪个仓库。  
需要同时满足：

- GitHub MCP 已配置可用
- 告警中提供 `repo_url` 或 `github_owner` / `github_repo`

### 3. Elasticsearch 在调查链路里走的是 `opensearch` 入口

这是本书最重要的实现细节之一。

当前代码里：

- 调查工具层既有 `ElasticsearchLogsTool`
- 也有 `OpenSearchAnalyticsTool`

但规划阶段的 `detect_sources()` 会把：

- `alert_source=elasticsearch`

映射到：

- `available_sources["opensearch"]`

因此本书的“可复现跑通”路径采用：

- `OPENSEARCH_URL`
- `OPENSEARCH_API_KEY`
- `OPENSEARCH_INDEX_PATTERN`

而不是 `ELASTICSEARCH_*`。

## 运行方式选择

本书统一采用：

```bash
uv run opensre ...
```

原因很简单：

- 它能确保使用当前仓库的代码
- 不依赖你系统上是否还有别的 `opensre`
- 与仓库里的开发说明一致

## 成功标准

当这本书的流程跑通时，你应该能看到：

1. `uv run opensre doctor` 报告 `provider=kimi, CLI ready`
2. `uv run opensre integrations verify github` 通过
3. `uv run opensre health` 至少能看到 GitHub 和 OpenSearch 配置已生效
4. `uv run opensre investigate -i docs/using-book/examples/elasticsearch-github-kimi-alert.json` 能返回 RCA JSON
