# 配置 GitHub 与数据源

这一章解决三件事：

1. 让 OpenSRE 能访问目标 GitHub 仓库
2. 让 OpenSRE 能访问 Elasticsearch / OpenSearch 日志
3. 让 OpenSRE 能访问 Kafka 集群（作为可选的替代数据源）

## 1. GitHub 的推荐配置方式

当前代码里，GitHub 的最稳定路径是 GitHub MCP 环境变量。

在项目根目录 `.env` 中加入：

```bash
GITHUB_MCP_URL=https://api.githubcopilot.com/mcp/
GITHUB_MCP_MODE=streamable-http
GITHUB_MCP_AUTH_TOKEN=ghp_your_token
GITHUB_MCP_TOOLSETS=repos,issues,pull_requests,actions,search
```

### 为什么这条路径可靠

因为 `app/integrations/_catalog_impl.py` 明确会从环境变量加载：

- `GITHUB_MCP_URL`
- `GITHUB_MCP_MODE`
- `GITHUB_MCP_COMMAND`
- `GITHUB_MCP_ARGS`
- `GITHUB_MCP_AUTH_TOKEN`
- `GITHUB_MCP_TOOLSETS`

也就是说，这不是文档假设，而是代码里确实存在的加载路径。

## 2. GitHub 验证命令

配置完后执行：

```bash
uv run opensre integrations verify github
```

如果通过，说明：

- token 可用
- MCP endpoint 可连
- OpenSRE 至少能发现一组 GitHub 工具

## 3. Elasticsearch 的推荐配置方式

### 代码里的真实入口是 `opensearch`

虽然仓库里有 `ElasticsearchLogsTool` 和 `ElasticsearchClient`，但调查主链路里的源检测是：

- `alert_source=elasticsearch`
- 映射到 `resolved_integrations["opensearch"]`
- 再暴露成 `available_sources["opensearch"]`

所以要让完整 pipeline 最稳妥地跑通，本书采用以下环境变量：

```bash
OPENSEARCH_URL=http://your-es-or-os-endpoint:9200
OPENSEARCH_API_KEY=your_api_key_if_needed
OPENSEARCH_INDEX_PATTERN=logs-*
OPENSEARCH_MAX_RESULTS=100
```

### 当前代码对 Elasticsearch 认证的实际边界

`app/services/elasticsearch/client.py` 当前只支持两种情况：

1. 无认证
2. API key 认证（请求头 `Authorization: ApiKey <token>`）

**不在当前"稳妥跑通路径"里的情况：**

- Basic Auth 用户名 / 密码
- 其他自定义认证头
- 复杂代理链路

## 4. Kafka 的推荐配置方式

Kafka 是独立的一等集成，有专用的工具和环境变量加载路径。

### 环境变量配置

```bash
KAFKA_BOOTSTRAP_SERVERS=localhost:9092
KAFKA_SECURITY_PROTOCOL=PLAINTEXT
KAFKA_SASL_MECHANISM=
KAFKA_SASL_USERNAME=
KAFKA_SASL_PASSWORD=
```

如果集群需要 SASL 认证，示例（SCRAM-SHA-512）：

```bash
KAFKA_BOOTSTRAP_SERVERS=kafka-cluster:9092
KAFKA_SECURITY_PROTOCOL=SASL_PLAINTEXT
KAFKA_SASL_MECHANISM=SCRAM-SHA-512
KAFKA_SASL_USERNAME=your_user
KAFKA_SASL_PASSWORD=your_password
```

### Kafka 集成验证

Kafka 的验证通过 `uv run opensre integrations verify <name>` 机制，但当前 `verify.py` 中对 Kafka 的支持需要确认。

最可靠的验证方式仍然是执行调查：

```bash
uv run opensre investigate -i docs/using-4-opencode/examples/kafka-github-opencode-alert.json
```

如果 Kafka 连接成功并在 `resolve_integrations` 阶段看到 `kafka`，工具 `get_kafka_topic_health` 和 `get_kafka_consumer_group_lag` 将可供调查使用。

### 当前代码对 Kafka 的支持边界

基于 `app/integrations/kafka.py` 的代码：

- 支持 `PLAINTEXT` 和 `SASL_PLAINTEXT` 协议
- 支持 `SCRAM-SHA-256`、`SCRAM-SHA-512`、`PLAIN` 等 SASL 机制
- 操作只读：list topics、consumer group lag、partition health
- 依赖 `confluent_kafka` 库（在 `pyproject.toml` 中声明）

## 5. 两套数据源怎么选

| 维度 | Elasticsearch / OpenSearch | Kafka |
|------|---------------------------|-------|
| 适用场景 | 应用日志检索、错误查询 | 消息队列消费延迟、主题健康 |
| alert_source | `elasticsearch` | `kafka` |
| 环境变量前缀 | `OPENSEARCH_*` | `KAFKA_*` |
| 主要工具 | `query_opensearch_analytics` | `get_kafka_topic_health`, `get_kafka_consumer_group_lag` |
| 认证方式 | 无认证 / ApiKey | PLAINTEXT / SASL |

你可以根据实际环境选择其中一种，或者在同一个告警中同时使用（但本书建议先只攻一个）。

## 6. `.env` 推荐最小集

### ES + GitHub 版

```bash
LLM_PROVIDER=opencode
OPENCODE_MODEL=anthropic/claude-opus-4.7

GITHUB_MCP_URL=https://api.githubcopilot.com/mcp/
GITHUB_MCP_MODE=streamable-http
GITHUB_MCP_AUTH_TOKEN=ghp_your_token
GITHUB_MCP_TOOLSETS=repos,issues,pull_requests,actions,search

OPENSEARCH_URL=http://your-es-or-os-endpoint:9200
OPENSEARCH_API_KEY=
OPENSEARCH_INDEX_PATTERN=logs-*
OPENSEARCH_MAX_RESULTS=100
```

### Kafka + GitHub 版

```bash
LLM_PROVIDER=opencode
OPENCODE_MODEL=anthropic/claude-opus-4.7

GITHUB_MCP_URL=https://api.githubcopilot.com/mcp/
GITHUB_MCP_MODE=streamable-http
GITHUB_MCP_AUTH_TOKEN=ghp_your_token
GITHUB_MCP_TOOLSETS=repos,issues,pull_requests,actions,search

KAFKA_BOOTSTRAP_SERVERS=localhost:9092
KAFKA_SECURITY_PROTOCOL=PLAINTEXT
```

完整示例见：

- [examples/env.opencode-github-opensearch.example](./examples/env.opencode-github-opensearch.example)
- [examples/env.opencode-github-kafka.example](./examples/env.opencode-github-kafka.example)
