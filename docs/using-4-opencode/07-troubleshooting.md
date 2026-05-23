# 故障排查与当前限制

## 1. `opensre doctor` 报 `provider=opencode, CLI not installed`

处理顺序：

1. 先执行 `opencode --version`
2. 如果命令不存在，重新安装 opencode：`brew install anomalyco/tap/opencode` 或 `choco install opencode`
3. 如果装了但不在 PATH，把完整路径写到 `OPENCODE_BIN`

## 2. `opensre doctor` 报 `CLI not authenticated`

先执行：

```bash
opencode auth list
```

确认输出中是否有 `N credentials` 或 `N environment variable(s)`。

如果都没有：

```bash
opencode auth login
```

如果你走 API key 路径，确认相关的环境变量已设置（如 `ANTHROPIC_API_KEY`）。

## 3. `opensre doctor` 报 `CLI auth status unclear`

OpenCode 的 `_parse_opencode_auth_list_output` 解析失败时返回 `logged_in=None`。

通常原因：

- `opencode auth list` 输出格式与预期不匹配
- 第一次运行超时（OpenCode 首次运行可能迁移本地数据）

处理方式：

- 重试一次
- 确认 `opencode auth list` 能正常输出
- 如果持续失败，可以在 GitHub 提 issue 附上 `opencode auth list` 的输出样例

## 4. `integrations verify github` 失败

优先检查：

- `GITHUB_MCP_AUTH_TOKEN` 是否有权限访问目标仓库
- `GITHUB_MCP_TOOLSETS` 是否包含 `repos`
- MCP endpoint 是否可访问

## 5. 调查运行了，但没有用到 GitHub 工具

最常见原因是告警里没把仓库定位信息写清楚。

至少确认存在以下之一：

- `repo_url`
- `github_owner + github_repo`
- `repository`

本书示例为了提高成功率，同时写了：

- `repo_url`
- `github_owner`
- `github_repo`

## 6. ES 场景：调查运行了，但没有拿到日志

优先检查：

- `OPENSEARCH_URL` 是否正确
- `OPENSEARCH_INDEX_PATTERN` 是否覆盖真实索引
- `elasticsearch_query` 是否真的能匹配到最近日志
- 日志时间窗口内是否确实有匹配文档

## 7. Kafka 场景：调查运行了，但没有用到 Kafka 工具

优先检查：

- `KAFKA_BOOTSTRAP_SERVERS` 是否能连通
- 告警中 `alert_source` 是否为 `kafka`
- 告警中是否提供了 `kafka_topic` 和 `kafka_group_id`
- `resolved_integrations` 阶段是否出现了 `kafka`

## 8. `health` 或 `verify` 通过，但调查仍失败

这是当前代码的一个现实限制。

基于代码扫描，`opensearch` 和 Kafka 的 verifier 当前更像：

- "配置存在性检查"

而不是：

- "真实执行日志搜索并确认返回结果"

所以判断数据源是否真正可用，应该以：

```bash
uv run opensre investigate -i <alert.json>
```

的真实执行结果为准。

## 9. 当前 Elasticsearch / OpenSearch 支持边界

### 当前稳妥支持

- 无认证 endpoint
- API key 认证 endpoint

### 当前不应默认假设可用

- Basic Auth 用户名 / 密码
- 复杂代理或自定义签名认证

原因是 `ElasticsearchClient` 当前只内建了 `ApiKey` 头逻辑。

## 10. 当前 Kafka 支持边界

### 当前稳妥支持

- PLAINTEXT 协议
- SASL_PLAINTEXT + SCRAM-SHA-256/512
- SASL_PLAINTEXT + PLAIN 机制
- 只读操作：list topics、consumer group lag、partition health

### 当前不应默认假设可用

- SSL 加密连接（未在默认配置中测试）
- 生产/消费消息（操作设计为只读）
- 复杂 ACL 环境

## 11. OpenCode 适配器的限制

来自 `app/integrations/llm_cli/opencode.py` 及 `runner.py`：

### 目前不是真流式

`CLIBackedLLMClient.invoke_stream()` 目前是：

- 等子进程跑完
- 把完整结果一次 yield 出去

也就是说 CLI provider 目前满足的是接口兼容，不是 token-by-token streaming。

### 超时控制

OpenCode 适配器的默认执行超时为 `120` 秒。如果调查耗时较长，复杂推理可能超时。

当前无法通过环境变量调整这个值（硬编码在 `OpenCodeAdapter.default_exec_timeout_sec = 120.0`）。

### 认证探测超时

`opencode auth list` 的超时为 `25` 秒。首次运行时 OpenCode 可能迁移本地数据，导致超时。如果遇到，重试即可。

## 12. 为什么 `integrations setup` 没有成为本书主路径

因为基于当前代码，最直接、最清晰、最可控的路径是环境变量：

- `OPENSEARCH_URL` / `OPENSEARCH_API_KEY` / `OPENSEARCH_INDEX_PATTERN`
- `KAFKA_BOOTSTRAP_SERVERS` / `KAFKA_SECURITY_PROTOCOL`

这些路径在 `_catalog_impl.py` 里是显式存在的。

## 13. 跑通后下一步建议

如果你已经跑通这本书的流程，下一步最值得做的是：

1. 把示例告警换成你自己的真实告警
2. 把查询参数收紧到真实服务和索引/主题
3. 如果仓库不止一个，给不同告警分别绑定不同 `repo_url`
4. 考虑将 OpenCode 切换到更适合的模型（如 `anthropic/claude-sonnet-4-6`）
