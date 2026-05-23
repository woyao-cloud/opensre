# 告警输入怎么写

OpenSRE 能不能顺利使用 GitHub 和日志集成，不只取决于集成本身，还取决于你的告警输入是否带了足够的定位信息。

## 1. 这次场景最重要的字段

为了让这次“GitHub + Elasticsearch”场景更容易跑通，建议告警里至少带上：

- `alert_source`
- `alert_name`
- `pipeline_name`
- `severity`
- `message`
- `repo_url`
- `github_owner`
- `github_repo`
- `branch`
- `error_message`
- `commonAnnotations.elasticsearch_query`
- `commonAnnotations.opensearch_index_pattern`
- `commonAnnotations.github_query`

## 2. 为什么这些字段重要

### `alert_source`

设为：

```json
"alert_source": "elasticsearch"
```

这样 `detect_sources()` 才会把它归到 Elasticsearch / OpenSearch 日志调查路径。

### `repo_url` / `github_owner` / `github_repo`

GitHub 工具需要知道要查哪个仓库。  
当前代码会优先尝试：

- `github_owner` + `github_repo`
- `repository` / `repo`
- `repo_url`

因此，最稳妥的做法是三个都给。

### `elasticsearch_query`

这会进入 `available_sources["opensearch"]["default_query"]`，直接影响第一轮日志调查范围。

### `opensearch_index_pattern`

这告诉日志工具优先查哪些 index。

## 3. 推荐的最小可用告警

完整示例见：

- [examples/elasticsearch-github-kimi-alert.json](./examples/elasticsearch-github-kimi-alert.json)

它绑定的代码仓库是：

```text
https://github.com/woyao-cloud/ollama-03-02-claude-prompt02-template.git
```

## 4. 一个最小结构长什么样

```json
{
  "alert_name": "Elasticsearch error spike in target service",
  "pipeline_name": "ollama_03_02_claude_prompt02_template",
  "severity": "critical",
  "alert_source": "elasticsearch",
  "message": "Investigate recent application errors using Elasticsearch logs and correlate with the target GitHub repository.",
  "repo_url": "https://github.com/woyao-cloud/ollama-03-02-claude-prompt02-template.git",
  "github_owner": "woyao-cloud",
  "github_repo": "ollama-03-02-claude-prompt02-template",
  "branch": "main",
  "error_message": "error OR exception OR failed",
  "commonAnnotations": {
    "summary": "Application errors are increasing in production.",
    "repo_url": "https://github.com/woyao-cloud/ollama-03-02-claude-prompt02-template.git",
    "elasticsearch_query": "error OR exception OR failed",
    "opensearch_index_pattern": "logs-*",
    "github_query": "error OR exception OR traceback OR failed"
  }
}
```

## 5. 如何针对自己的系统替换字段

建议你优先替换这几项：

- `pipeline_name`
- `message`
- `error_message`
- `commonAnnotations.elasticsearch_query`
- `commonAnnotations.opensearch_index_pattern`

其中最关键的是：

- `elasticsearch_query` 必须能在你的日志里匹配到真实数据
- `opensearch_index_pattern` 必须覆盖到你的真实日志索引

## 6. 为什么示例里同时给了顶层字段和 `commonAnnotations`

因为当前代码的源检测会同时看：

- 标准 annotations
- 原始 payload 顶层字段

双写的目的不是美观，而是提高跑通概率。
