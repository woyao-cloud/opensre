# 这套配置为什么能跑通

这一章不是讲全部架构，只讲这次使用场景需要理解的执行链。

## 1. 调查入口

你执行的是：

```bash
uv run opensre investigate -i <alert.json>
```

CLI 会：

1. 读取 JSON
2. 解析 alert payload
3. 调用 investigation runner

## 2. LLM provider 选择

`app/services/llm_client.py` 会读取 `LLM_PROVIDER`。

当它发现：

```bash
LLM_PROVIDER=kimi
```

就会：

1. 去 `CLI_PROVIDER_REGISTRY` 找到 `KimiAdapter`
2. 构造 `CLIBackedLLMClient`
3. 通过子进程调用 `kimi`

## 3. GitHub 来源是怎么发现的

`detect_sources()` 会从告警里提取：

- `repo_url`
- `github_owner`
- `github_repo`
- `branch`
- `sha`
- `github_query`

如果能确定 owner/repo，就会把 `available_sources["github"]` 标出来。

## 4. Elasticsearch 来源是怎么发现的

这是这次场景的关键实现细节。

当前代码在 `detect_sources()` 中的逻辑是：

- 如果 `alert_source in ("opensearch", "elasticsearch", "")`
- 并且 `resolved_integrations["opensearch"]` 存在
- 那么生成 `available_sources["opensearch"]`

所以：

- 告警来源叫 `elasticsearch`
- 运行时能力来源却叫 `opensearch`

这就是为什么说明书要求你配置 `OPENSEARCH_*`。

## 5. planner 会得到什么提示

`build_prompt.py` 会把已发现的数据源提示给 planner。

如果 GitHub 可用，它会告诉模型：

- 仓库名
- commit/ref
- code query
- 优先使用提交、代码搜索、文件内容工具

如果 OpenSearch 可用，它会告诉模型：

- index pattern
- default query
- 可以使用 `query_opensearch_analytics`

## 6. 这次场景下最可能出现的工具

并不是每次都完全相同，但按当前代码，最相关的是：

- `query_opensearch_analytics`
- `list_github_commits`
- `search_github_code`
- `get_github_file_contents`
- `get_git_deploy_timeline`

## 7. 一条简化执行链

```text
alert json
  -> extract_alert
  -> resolve_integrations
  -> detect_sources
  -> plan_actions
  -> run log/code tools
  -> merge evidence
  -> diagnose
  -> publish final report
```

## 8. 为什么这次书里不用“只靠自然语言输入”

因为这次目标不是“能聊起来”，而是“让完整 pipeline 稳定识别出日志与代码来源”。

自然语言输入虽然也能触发调查，但：

- repo 定位不稳定
- query 提示不稳定
- index pattern 不稳定

而 JSON 告警可以把这些关键约束直接送进系统。
