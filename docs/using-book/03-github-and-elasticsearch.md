# 配置 GitHub 与 Elasticsearch

这一章只解决两件事：

1. 让 OpenSRE 能访问目标 GitHub 仓库
2. 让 OpenSRE 能访问 Elasticsearch / OpenSearch 日志

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

这里要特别注意。

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

## 4. 为什么不用 `ELASTICSEARCH_*`

因为当前代码里环境变量加载器明确读取的是：

- `OPENSEARCH_URL`
- `OPENSEARCH_API_KEY`
- `OPENSEARCH_INDEX_PATTERN`
- `OPENSEARCH_MAX_RESULTS`

并没有对应的 `ELASTICSEARCH_URL` 环境加载路径。

## 5. 当前代码对 Elasticsearch 认证的实际边界

这是本书最需要明确说明的限制。

`app/services/elasticsearch/client.py` 当前只支持两种情况：

1. 无认证
2. API key 认证

也就是请求头形如：

```text
Authorization: ApiKey <token>
```

### 不在当前“稳妥跑通路径”里的情况

下面这些不应视为当前书中的主路径：

- Basic Auth 用户名 / 密码
- 其他自定义认证头
- 复杂代理链路

原因是：

- `ElasticsearchClient` 只内建了 API key 头
- `verify opensearch` 当前只检查 URL 是否存在，不会实际执行日志搜索

## 6. 所以本书采用的可靠前提

本书默认你的日志后端满足以下任一条件：

1. 集群允许无认证访问
2. 集群支持 `ApiKey` 方式访问

如果你的 Elasticsearch 只能用用户名/密码，而不能用 API key，那么按当前代码，完整 pipeline 不应被视为已具备稳定支持。

## 7. `.env` 推荐最小集

这一节与 Kimi 合并后，最小配置大致如下：

```bash
LLM_PROVIDER=kimi
KIMI_MODEL=kimi-k2.5

GITHUB_MCP_URL=https://api.githubcopilot.com/mcp/
GITHUB_MCP_MODE=streamable-http
GITHUB_MCP_AUTH_TOKEN=ghp_your_token
GITHUB_MCP_TOOLSETS=repos,issues,pull_requests,actions,search

OPENSEARCH_URL=http://your-es-or-os-endpoint:9200
OPENSEARCH_API_KEY=
OPENSEARCH_INDEX_PATTERN=logs-*
OPENSEARCH_MAX_RESULTS=100
```

完整示例见：

- [examples/env.kimi-github-opensearch.example](./examples/env.kimi-github-opensearch.example)
