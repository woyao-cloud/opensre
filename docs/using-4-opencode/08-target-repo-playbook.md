# 针对目标仓库的定制实战

这一章只服务于这一个目标仓库：

```text
https://github.com/woyao-cloud/ollama-03-02-claude-prompt02-template.git
```

## 先说明边界

当前环境下，我没有拿到这个仓库的公开 README / 文件树内容，所以这一章不假装"已经读过目标仓库源码"。

这章里的定制分成两类：

### 已确认的事实

这些来自你给出的 URL，本书可以确定：

- owner: `woyao-cloud`
- repo: `ollama-03-02-claude-prompt02-template`
- 它是 GitHub 仓库，可以被 GitHub MCP 工具按 `owner/repo` 方式定位

### 明确标注为推断的内容

这些来自仓库名本身，而不是仓库源码实读：

- 这个仓库大概率与 `ollama` 有关
- 大概率是某种 prompt / template / model configuration 仓库
- 排查时很可能应优先关注：
  - prompt 模板
  - system prompt
  - model 参数
  - `Modelfile`
  - README / 配置文件

如果这些推断不符合真实仓库结构，请直接按真实目录和关键字替换本章建议。

## 1. 为什么这个仓库需要更"定向"的 GitHub 查询

对这类仓库，通用查询：

```text
error OR exception OR failed
```

往往不够好。

原因是如果仓库本身主要存放的是：

- prompt 模板
- model 配置
- instruction 文本

那么代码搜索更应该先找：

- 模板定义
- 系统提示
- 参数配置
- 输出格式约束

而不是只找运行时异常关键词。

## 2. 推荐的 `github_query` 策略

### 方案 A：保守通用版

适合你还不知道仓库里关键文件名时：

```text
error OR exception OR failed OR prompt
```

### 方案 B：模板/提示词仓库版

如果这个仓库确实是 Ollama prompt/template 仓库，优先用：

```text
Modelfile OR prompt OR template OR system OR parameter OR claude OR ollama
```

### 方案 C：回归排查版

如果你怀疑是某次模板变更导致输出劣化、行为异常或提示词回归，优先用：

```text
prompt OR template OR system OR instruction OR output format OR parameter
```

## 3. 为什么 `github_query` 很关键

当前代码里：

- `detect_sources()` 会把 `github_query` 放进 `available_sources["github"]["query"]`
- `search_github_code` 会自动把这个查询补成：`你的查询 + repo:woyao-cloud/ollama-03-02-claude-prompt02-template`

所以这不是一个"只是给模型看的提示"，而是会直接影响 GitHub MCP 的代码搜索范围。

## 4. 如果你知道具体文件路径，要把它显式写进告警

这是一个很实用但容易忽略的点。

当前代码里，`get_github_file_contents` 想自动可用，需要 `github.path` 存在。而这个路径来自告警里的：

- `file_path`
- 或 `commonAnnotations.file_path`
- 或 `github_path`

所以如果你已经知道问题大概率在某个文件，可以直接写进去，例如：

```json
"file_path": "Modelfile"
```

或者：

```json
"file_path": "README.md"
```

或者你自己的真实路径，例如：

```json
"file_path": "prompts/claude-prompt02.txt"
```

### 这会带来的效果

如果路径有效，planner 除了能做：

- `search_github_code`
- `list_github_commits`

还更容易直接走：

- `get_github_file_contents`

这样第一轮证据会更聚焦。

## 5. `branch` 不要想当然

当前示例文件里用了：

```json
"branch": "main"
```

这是默认假设，不是已验证事实。如果目标仓库实际默认分支不是 `main`，请改成真实分支名。

## 6. 推荐的三种告警写法

### 写法 1：最通用的仓库关联调查

适合你只想让 OpenSRE 同时看数据源和仓库：

- `github_query = error OR exception OR failed OR prompt`
- 不指定 `file_path`
- 可用于 ES 或 Kafka 场景

### 写法 2：模板回归调查

适合你怀疑某次 prompt/template 修改造成行为变化：

- `github_query = prompt OR template OR system OR instruction OR output format`
- 如果知道路径，再补 `file_path`

### 写法 3：配置文件调查

适合你怀疑是 Ollama model config、Modelfile、参数设置变更引起：

- `github_query = Modelfile OR parameter OR ollama OR template`
- 如果知道路径，优先写 `file_path=Modelfile`

## 7. 针对这个仓库的推荐告警

```json
{
  "alert_name": "Potential prompt or template regression in target repository",
  "pipeline_name": "ollama_03_02_claude_prompt02_template",
  "severity": "critical",
  "alert_source": "elasticsearch",
  "message": "Investigate whether a recent template, prompt, Modelfile, or parameter change in the target repository correlates with recent production failures.",
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

## 8. 这个仓库场景下的推荐执行顺序

建议你按下面顺序尝试：

1. 先用通用 ES 示例跑一次，确认 GitHub + OpenSearch 整条链能工作
2. 再换成模板回归版告警，看 planner 是否更偏向 GitHub 模板/配置调查
3. 如果你知道真实关键文件，再把 `file_path` 写进去做第三轮
4. 如果实际使用的是 Kafka，先用 Kafka 示例确保 Kafka 链路通，再切回模板回归查询

## 9. 最值得你手工替换的字段

- `branch`
- `github_query`
- `file_path`
- `elasticsearch_query`（ES 场景）或 `kafka_topic` / `kafka_group_id`（Kafka 场景）
- `opensearch_index_pattern`（ES 场景）

其中最值钱的是：

- `github_query`
- `file_path`

因为它们最直接决定 GitHub 工具第一轮能不能快速命中真正相关内容。

## 10. 一页结论

对 `woyao-cloud/ollama-03-02-claude-prompt02-template` 这种目标仓库，OpenSRE 最应该避免的是"只用通用错误词做代码搜索"。

更好的做法是：

- 保留数据源查询来证明"确实出问题了"
- 用更定向的 `github_query` 去找模板、提示词、参数和配置文件
- 如果知道具体路径，就把 `file_path` 直接放进告警

这样 OpenSRE 才更容易把这次调查从"泛泛地看最近提交"推进到"直接审读真正可能出问题的模板文件"。
