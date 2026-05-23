# 告警输入怎么写

OpenSRE 能不能顺利使用 GitHub 和数据源集成，不只取决于集成本身，还取决于你的告警输入是否带了足够的定位信息。

## 1. 全局重要字段

为了让场景更容易跑通，告警里建议至少带上：

- `alert_source` — `elasticsearch` 或 `kafka`
- `alert_name`
- `pipeline_name`
- `severity`
- `message`
- `repo_url`
- `github_owner`
- `github_repo`
- `branch`
- `error_message`
- `commonAnnotations`：包含针对数据源的查询参数

## 2. 为什么这些字段重要

### `alert_source`

决定 `detect_sources()` 将告警归类到哪个数据源路径：

- `"alert_source": "elasticsearch"` → 映射到 `available_sources["opensearch"]`
- `"alert_source": "kafka"` → 映射到 `available_sources["kafka"]`

### `repo_url` / `github_owner` / `github_repo`

GitHub 工具需要知道要查哪个仓库。当前代码会优先尝试：

- `github_owner` + `github_repo`
- `repository` / `repo`
- `repo_url`

最稳妥的做法是三个都给。

### `elasticsearch_query`

这会进入 `available_sources["opensearch"]["default_query"]`，直接影响第一轮日志调查范围。

### `opensearch_index_pattern`

告诉日志工具优先查哪些 index。

### Kafka 相关字段

Kafka 场景的关键字段：

- `commonAnnotations.kafka_topic` — 要检查的 Kafka 主题
- `commonAnnotations.kafka_group_id` — 要检查的消费者组
- `error_message` — 用于上下文描述

## 3. Elasticsearch 场景的最小可用告警

```json
{
  "alert_name": "Elasticsearch error spike for target service",
  "pipeline_name": "ollama_03_02_claude_prompt02_template",
  "severity": "critical",
  "alert_source": "elasticsearch",
  "message": "Investigate recent application failures using Elasticsearch logs and correlate them with the target GitHub repository.",
  "service_name": "ollama-03-02-claude-prompt02-template",
  "repo_url": "https://github.com/woyao-cloud/ollama-03-02-claude-prompt02-template.git",
  "github_owner": "woyao-cloud",
  "github_repo": "ollama-03-02-claude-prompt02-template",
  "branch": "main",
  "error_message": "error OR exception OR failed",
  "commonAnnotations": {
    "summary": "Recent production logs show repeated application errors. Investigate logs first, then correlate with recent code changes.",
    "repo_url": "https://github.com/woyao-cloud/ollama-03-02-claude-prompt02-template.git",
    "elasticsearch_query": "error OR exception OR failed",
    "opensearch_index_pattern": "logs-*",
    "github_query": "error OR exception OR traceback OR failed",
    "branch": "main",
    "correlation_id": "replace-me"
  }
}
```

## 4. Kafka 场景的最小可用告警

```json
{
  "alert_name": "Kafka consumer lag spike for target service",
  "pipeline_name": "ollama_03_02_claude_prompt02_template",
  "severity": "warning",
  "alert_source": "kafka",
  "message": "Investigate recent Kafka consumer group lag or topic health issues and correlate with the target GitHub repository.",
  "service_name": "ollama-03-02-claude-prompt02-template",
  "repo_url": "https://github.com/woyao-cloud/ollama-03-02-claude-prompt02-template.git",
  "github_owner": "woyao-cloud",
  "github_repo": "ollama-03-02-claude-prompt02-template",
  "branch": "main",
  "error_message": "Consumer group lag detected on target topic.",
  "commonAnnotations": {
    "summary": "Kafka consumer group lag is increasing for the target service. Investigate topic health and correlate with recent code changes.",
    "repo_url": "https://github.com/woyao-cloud/ollama-03-02-claude-prompt02-template.git",
    "kafka_topic": "application-errors",
    "kafka_group_id": "error-processor-group",
    "github_query": "kafka OR consumer OR error-handler OR failed OR exception",
    "branch": "main",
    "correlation_id": "replace-me"
  }
}
```

## 5. 针对目标仓库的告警（模板回归版）

同时适用于 ES 和 Kafka 两个场景：

```json
{
  "alert_name": "Potential prompt or template regression in target repository",
  "pipeline_name": "ollama_03_02_claude_prompt02_template",
  "severity": "critical",
  "alert_source": "elasticsearch",
  "message": "Investigate whether a recent template, prompt, Modelfile, or parameter change correlates with recent production failures.",
  "service_name": "ollama-03-02-claude-prompt02-template",
  "repo_url": "https://github.com/woyao-cloud/ollama-03-02-claude-prompt02-template.git",
  "github_owner": "woyao-cloud",
  "github_repo": "ollama-03-02-claude-prompt02-template",
  "branch": "main",
  "error_message": "error OR exception OR failed",
  "commonAnnotations": {
    "summary": "Recent failures may correlate with a prompt, template, or Modelfile change in the target repository.",
    "repo_url": "https://github.com/woyao-cloud/ollama-03-02-claude-prompt02-template.git",
    "elasticsearch_query": "error OR exception OR failed",
    "opensearch_index_pattern": "logs-*",
    "github_query": "Modelfile OR prompt OR template OR system OR parameter OR claude OR ollama",
    "branch": "main",
    "correlation_id": "replace-me"
  }
}
```

## 6. 为什么示例里同时给了顶层字段和 `commonAnnotations`

因为当前代码的源检测会同时看：

- 标准 annotations
- 原始 payload 顶层字段

双写的目的不是美观，而是提高跑通概率。

## 7. 如何针对自己的系统替换字段

建议优先替换：

- `pipeline_name`
- `message`
- `error_message`
- `commonAnnotations.elasticsearch_query` / `kafka_topic`
- `commonAnnotations.opensearch_index_pattern`

**Elasticsearch 场景：** `elasticsearch_query` 必须能在你的日志里匹配到真实数据，`opensearch_index_pattern` 必须覆盖到你的真实日志索引。

**Kafka 场景：** `kafka_topic` 必须在你的 Kafka 集群上真实存在，`kafka_group_id` 必须对应一个活跃的消费者组。
