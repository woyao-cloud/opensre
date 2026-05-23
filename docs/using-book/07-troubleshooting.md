# 故障排查与当前限制

## 1. `opensre doctor` 报 `provider=kimi, CLI not installed`

处理顺序：

1. 先执行 `kimi --version`
2. 如果命令不存在，重新安装 `kimi-cli`
3. 如果装了但不在 PATH，把完整路径写到 `KIMI_BIN`

## 2. `opensre doctor` 报 `CLI not authenticated`

先执行：

```bash
kimi login
kimi login status
```

如果你走 API key 路径，确认：

- `KIMI_API_KEY` 已设置
- 或 `~/.kimi/config.toml` 已存在有效 key

## 3. `integrations verify github` 失败

优先检查：

- `GITHUB_MCP_AUTH_TOKEN` 是否有权限访问目标仓库
- `GITHUB_MCP_TOOLSETS` 是否包含 `repos`
- MCP endpoint 是否可访问

## 4. 调查运行了，但没有用到 GitHub 工具

最常见原因是告警里没把仓库定位信息写清楚。

至少确认存在以下之一：

- `repo_url`
- `github_owner + github_repo`
- `repository`

本书示例为了提高成功率，同时写了：

- `repo_url`
- `github_owner`
- `github_repo`

## 5. 调查运行了，但没有拿到日志

优先检查：

- `OPENSEARCH_URL` 是否正确
- `OPENSEARCH_INDEX_PATTERN` 是否覆盖真实索引
- `elasticsearch_query` 是否真的能匹配到最近日志
- 日志时间窗口内是否确实有匹配文档

## 6. `health` 或 `verify opensearch` 看起来通过，但调查仍失败

这是当前代码的一个现实限制。

基于代码扫描，`opensearch` 的 verifier 当前更像：

- “配置存在性检查”

而不是：

- “真实执行日志搜索并确认返回结果”

所以判断 OpenSearch 是否真正可用，应该以：

```bash
uv run opensre investigate -i <alert.json>
```

的真实执行结果为准。

## 7. 当前 Elasticsearch / OpenSearch 支持边界

本书建议你明确接受以下现实：

### 当前稳妥支持

- 无认证 endpoint
- API key 认证 endpoint

### 当前不应默认假设可用

- Basic Auth 用户名 / 密码
- 复杂代理或自定义签名认证

原因是 `ElasticsearchClient` 当前只内建了 `ApiKey` 头逻辑。

## 8. 为什么 `integrations setup opensearch` 没有成为本书主路径

因为基于当前代码，最直接、最清晰、最可控的路径是环境变量：

- `OPENSEARCH_URL`
- `OPENSEARCH_API_KEY`
- `OPENSEARCH_INDEX_PATTERN`

这条路径在 `_catalog_impl.py` 里是显式存在的。

## 9. 如果你的 Elasticsearch 只能用户名/密码登录

那你需要把它当作“当前代码之外的扩展需求”，而不是这本书的标准跑通路径。

换句话说：

- 这本书能保证的是“按当前代码可复现的使用路径”
- 不是“所有 Elasticsearch 认证模型都已经被完整支持”

## 10. 跑通后下一步建议

如果你已经跑通这本书的流程，下一步最值得做的是：

1. 把示例告警换成你自己的真实告警
2. 把 `elasticsearch_query` 收紧到真实服务
3. 把 `opensearch_index_pattern` 收紧到真实索引
4. 如果仓库不止一个，给不同告警分别绑定不同 `repo_url`
