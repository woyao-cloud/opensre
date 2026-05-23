# 使用目标与前提

## 最终目标

跑通下面这条最小但完整的本地链路：

1. OpenSRE 从 **OpenCode CLI** 获取推理能力（通过 `opencode run` 非交互模式）。
2. OpenSRE 从 **Elasticsearch / OpenSearch** 兼容接口拉取日志，**或**从 **Kafka** 错误消息队列获取消费延迟和主题健康信息。
3. OpenSRE 从 **GitHub MCP** 访问指定代码仓库。
4. OpenSRE 读取一个包含数据源查询提示和仓库信息的告警 JSON。
5. OpenSRE 完成一次本地 investigation，并输出 RCA 结果。

## 本书假设你已经有

- 当前仓库源码（opensre）
- Python / `uv`
- 一个可用的 **OpenCode CLI**（已安装并登录）
- 一个可访问的 **GitHub token**
- 以下 **至少一项** 可用数据源：
  - Elasticsearch / OpenSearch 兼容 endpoint（无认证或 API Key 认证）
  - Kafka 集群（PLAINTEXT 或 SASL 认证）

## 本书不假设你已经有

- 远端 LangGraph 部署
- Slack、Grafana、Datadog 等其他集成
- 完整的生产环境告警平台

## 本书的关键判断

基于代码扫描，这次跑通流程最关键的几个事实是：

### 1. `opencode` 是正式支持的 CLI provider

在 `app/config.py`、`app/cli/wizard/config.py`、`app/integrations/llm_cli/registry.py` 里，`opencode` 都是正式 provider，可以通过 `opensre onboard` 向导直接配置。

### 2. OpenCode 是多 provider 网关

OpenCode 本身支持多种 LLM 供应商。模型名格式为 `provider/model`（如 `anthropic/claude-opus-4.7`）。认证方式可以是 `opencode auth login` 或通过 `ANTHROPIC_API_KEY` / `OPENAI_API_KEY` 等环境变量。

### 3. Elasticsearch 在调查链路里走的是 `opensearch` 入口

同 `using-book` 的一致发现：

- `detect_sources()` 会把 `alert_source=elasticsearch` 映射到 `available_sources["opensearch"]`
- 因此需要配置 `OPENSEARCH_URL` / `OPENSEARCH_API_KEY` / `OPENSEARCH_INDEX_PATTERN`
- 当前 `ElasticsearchClient` 只支持无认证和 `ApiKey` 两种模式

### 4. Kafka 是独立的一等集成

Kafka 有自己的工具集（`get_kafka_topic_health`、`get_kafka_consumer_group_lag`）和环境变量加载路径（`KAFKA_BOOTSTRAP_SERVERS` 等），通过 `alert_source=kafka` 触发。

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

1. `uv run opensre doctor` 报告 `provider=opencode, CLI ready`
2. `uv run opensre integrations verify github` 通过
3. `uv run opensre health` 至少能看到 GitHub 和数据源（OpenSearch / Kafka）配置已生效
4. `uv run opensre investigate -i <alert.json>` 能返回 RCA JSON 且至少包含 `report`、`problem_md`、`root_cause`
