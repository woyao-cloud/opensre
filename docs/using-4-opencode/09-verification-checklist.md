# 手工验证清单

这一章只回答一个问题：

> 跑完以后，怎么判断这次 investigation 真的调用了 GitHub 工具和数据源工具，而不是只做了泛化推理？

这件事在当前代码里很重要，因为：

- `--output` 返回的最终 JSON 很精简
- 它不会直接列出"调用了哪些工具"
- 真正的工具调用线索主要出现在流式终端输出里

## 先记住一个前提

如果你要验证工具是否真的被调用，请优先这样跑：

```bash
uv run opensre investigate -i docs/using-4-opencode/examples/elasticsearch-github-opencode-alert.json
```

不要一上来就：

```bash
uv run opensre investigate ... --output result.json
```

也不要一上来就把 stdout 重定向到文件。

### 原因

当前 CLI 逻辑里，只有在下面条件同时满足时才会走流式本地 UI：

- stdout 是 TTY
- 没开 `--json`
- 没指定 `--output`

如果不满足，就会走非流式路径。非流式路径仍然会给你最终结果，但你看不到细粒度的工具调用过程。

## A. 运行前检查

在真正调查前，先确认这三个条件：

### 1. `doctor` 通过 OpenCode provider

```bash
uv run opensre doctor
```

你想看到的是：

- `provider=opencode`
- `CLI ready`

### 2. GitHub MCP 验证通过

```bash
uv run opensre integrations verify github
```

你想看到的是：

- GitHub MCP 已通过
- 至少发现了一组仓库调查工具

### 3. `health` 能看到集成

```bash
uv run opensre health
```

你想确认的是：

- `github` 在 resolved / verified 结果中可见
- `opensearch` 或 `kafka` 在结果中可见

注意：

- 这里只能证明"配置已被加载"
- 还不能证明"数据查询已经成功"

## B. 运行时检查

这一段最关键。你需要直接看终端。

### 1. 看 `resolve_integrations`

流式输出中，`resolve_integrations` 节点完成时，当前 renderer 会把已解析的集成名称打出来。

你重点想看到：

- `github`
- `opensearch`（ES 场景）或 `kafka`（Kafka 场景）

如果这里看不到其中一个，后面的工具几乎不可能正常触发。

### 2. 看 `plan_actions`

`plan_actions` 节点完成时，renderer 会打印：

```text
Planned actions: [...]
```

这一步是最直接的第一层验证。

**ES 场景你希望在 planned actions 里看到的典型动作：**

- `query_opensearch_analytics`
- `search_github_code`
- `list_github_commits`
- `get_github_file_contents`
- `get_git_deploy_timeline`

**Kafka 场景你希望在 planned actions 里看到的典型动作：**

- `get_kafka_topic_health`
- `get_kafka_consumer_group_lag`
- `search_github_code`
- `list_github_commits`

### 3. 看 `investigate` 阶段的工具调用提示

当前流式 renderer 会把工具调用翻译成类似下面的短语：

- `calling <tool>`
- `<tool> returned`
- `<tool> done`

也就是说，运行过程中你应当在终端里看到类似：

```text
calling Search GitHub Code
calling List GitHub Commits
calling Query OpenSearch Analytics
```

或：

```text
calling Get Kafka Topic Health
calling Get Kafka Consumer Group Lag
```

### 4. 特别关注"至少一个 GitHub + 一个数据源动作"

对你这次场景，最实用的验证标准不是"所有相关工具都被调了"，而是：

#### 最低通过标准

满足以下两条即可判断"这次确实用了两类外部证据"：

1. 至少看到一个 GitHub 相关动作
2. 至少看到一个数据源相关动作（`opensearch` 或 `kafka`）

## C. 跑完后检查

即使你没有把流式过程录下来，跑完以后也还能做第二层验证。

### 1. 看最终报告内容

调查结束后，重点看：

- `report`
- `problem_md`
- `root_cause`

**GitHub 侧的典型证据痕迹：**

如果 GitHub 证据真的参与了诊断，报告中更可能出现：

- 仓库名
- 分支名
- commit / 提交时间
- 文件路径
- 模板、提示词、配置文件名
- "recent change" 之类的因果描述

**数据源侧的典型证据痕迹（ES）：**

- 错误关键词
- 异常片段
- 日志现象描述
- "from logs" / "error logs show ..." 这类因果描述

**数据源侧的典型证据痕迹（Kafka）：**

- 消费者组延迟
- 分区状态
- 主题健康信息
- 与代码变更的时间关联分析

### 2. 不要误判"只要有结论就等于用到了工具"

这很重要。

当前 OpenSRE 即使在证据不强时，也可能给出一个推理结果。所以判断标准不应是：

- "它给了 root cause"

而应是：

- `resolve_integrations` 看到了数据源和 GitHub
- `plan_actions` 里出现了对应动作
- 运行时真的有对应工具调用
- 报告里能看到对应证据痕迹

### 3. 如果你还要保存最终 JSON

建议做两次运行：

**第一次：只做验证**

```bash
uv run opensre investigate -i docs/using-4-opencode/examples/elasticsearch-github-opencode-alert.json
```

目的：看流式工具调用，手工确认 GitHub + 数据源都被实际使用。

**第二次：只做结果归档**

```bash
uv run opensre investigate -i docs/using-4-opencode/examples/elasticsearch-github-opencode-alert.json --output docs/using-4-opencode/examples/rca-result.json
```

目的：保存最终 RCA JSON。

## D. 对照实验

如果你想更强地证明"输入字段真的影响了工具选择"，做两个很便宜的对照实验就够了。

### 实验 1：去掉 GitHub 定位字段

把下面字段去掉或清空：

- `repo_url`
- `github_owner`
- `github_repo`

然后重跑。

**预期变化：**

- `planned_actions` 里的 GitHub 工具会明显减少，甚至消失
- 终端里的 GitHub `calling ...` 提示减少或消失
- 最终报告更难引用提交、文件或代码变化

### 实验 2：把数据源查询改成无意义值

**ES 场景：** 把 `elasticsearch_query` 改成大概率匹配不到的值：

```json
"elasticsearch_query": "zzzz_nonexistent_marker_12345"
```

**Kafka 场景：** 把 `kafka_topic` 改成不存在的主题：

```json
"kafka_topic": "nonexistent_topic_12345"
```

然后重跑。

**预期变化：**

- 仍可能看到数据源工具被调用
- 但数据源证据会明显变弱
- 最终报告里更难引用具体数据源现象

这说明系统不是"假装用了数据源"，而是真的受查询结果影响。

## E. 最短验证路径

如果你只想用 30 秒判断这次是否真正调用了 GitHub 和数据源工具，就按这个最短清单走：

1. `uv run opensre doctor`
2. `uv run opensre integrations verify github`
3. 直接交互运行 `uv run opensre investigate -i <alert.json>`
4. 在终端里确认：
   - `Resolved: [...]` 里有 `github` 和数据源
   - `Planned actions: [...]` 里有 GitHub 和数据源动作
   - 调查阶段至少出现一个 GitHub `calling ...`
   - 调查阶段至少出现一个数据源 `calling ...`

满足这四点，就可以认为：

> 这次 investigation 不是只做了空泛总结，而是确实把 GitHub 和数据源两类外部证据接进了 pipeline。
