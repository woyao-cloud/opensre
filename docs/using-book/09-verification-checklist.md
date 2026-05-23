# 手工验证清单

这一章只回答一个问题：

> 跑完以后，怎么判断这次 investigation 真的调用了 GitHub 工具和日志工具，而不是只做了泛化推理？

这件事在当前代码里很重要，因为：

- `--output` 返回的最终 JSON 很精简
- 它不会直接列出“调用了哪些工具”
- 真正的工具调用线索主要出现在流式终端输出里

## 先记住一个前提

如果你要验证工具是否真的被调用，请优先这样跑：

```bash
uv run opensre investigate -i docs/using-book/examples/ollama-template-repo-alert.json
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

如果不满足，就会走非流式路径。  
非流式路径仍然会给你最终结果，但你看不到细粒度的工具调用过程。

## A. 运行前检查

在真正调查前，先确认这三个条件：

### 1. `doctor` 通过 Kimi provider

```bash
uv run opensre doctor
```

你想看到的是：

- `provider=kimi`
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
- `opensearch` 在结果中可见

注意：

- 这里只能证明“配置已被加载”
- 还不能证明“日志查询已经成功”

## B. 运行时检查

这一段最关键。  
你需要直接看终端。

## 1. 看 `resolve_integrations`

流式输出中，`resolve_integrations` 节点完成时，当前 renderer 会把已解析的集成名称打出来。

你重点想看到：

- `github`
- `opensearch`

如果这里看不到其中一个，后面的工具几乎不可能正常触发。

## 2. 看 `plan_actions`

`plan_actions` 节点完成时，renderer 会打印：

```text
Planned actions: [...]
```

这一步是最直接的第一层验证。

### 你希望在 planned actions 里看到的典型动作

GitHub 侧常见：

- `search_github_code`
- `list_github_commits`
- `get_github_file_contents`
- `get_git_deploy_timeline`

日志侧常见：

- `query_opensearch_analytics`

如果你传的是更通用的示例，也可能看到：

- GitHub 工具较多
- OpenSearch 工具只有一个主查询动作

这都正常。

## 3. 看 `investigate` 阶段的工具调用提示

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

实际显示名是否完全一致，取决于工具注册时的 display name。  
但含义应该是明确的：

- 在调用 GitHub 工具
- 在调用日志工具

## 4. 特别关注“至少一个 GitHub + 一个日志动作”

对你这次场景，最实用的验证标准不是“所有相关工具都被调了”，而是：

### 最低通过标准

满足以下两条即可判断“这次确实用了两类外部证据”：

1. 至少看到一个 GitHub 相关动作
2. 至少看到一个 OpenSearch / 日志相关动作

## C. 跑完后检查

即使你没有把流式过程录下来，跑完以后也还能做第二层验证。

## 1. 看最终报告内容

调查结束后，重点看：

- `report`
- `problem_md`
- `root_cause`

### GitHub 侧的典型证据痕迹

如果 GitHub 证据真的参与了诊断，报告中更可能出现：

- 仓库名
- 分支名
- commit / 提交时间
- 文件路径
- 模板、提示词、配置文件名
- “recent change” 之类的因果描述

### 日志侧的典型证据痕迹

如果日志证据真的参与了诊断，报告中更可能出现：

- 错误关键词
- 异常片段
- 日志现象描述
- “from logs” / “error logs show ...” 这类因果描述

## 2. 不要误判“只要有结论就等于用到了工具”

这很重要。

当前 OpenSRE 即使在证据不强时，也可能给出一个推理结果。  
所以判断标准不应是：

- “它给了 root cause”

而应是：

- `resolve_integrations` 看到了 `github/opensearch`
- `plan_actions` 里出现了对应动作
- 运行时真的有对应工具调用
- 报告里能看到对应证据痕迹

## 3. 如果你还要保存最终 JSON

建议做两次运行：

### 第一次：只做验证

```bash
uv run opensre investigate -i docs/using-book/examples/ollama-template-repo-alert.json
```

目的：

- 看流式工具调用
- 手工确认 GitHub / OpenSearch 都被实际使用

### 第二次：只做结果归档

```bash
uv run opensre investigate -i docs/using-book/examples/ollama-template-repo-alert.json --output docs/using-book/examples/rca-result.json
```

目的：

- 保存最终 RCA JSON

## D. 对照实验

如果你想更强地证明“输入字段真的影响了工具选择”，做两个很便宜的对照实验就够了。

## 实验 1：去掉 GitHub 定位字段

把下面字段去掉或清空：

- `repo_url`
- `github_owner`
- `github_repo`

然后重跑。

### 预期变化

- `planned_actions` 里的 GitHub 工具会明显减少，甚至消失
- 终端里的 GitHub `calling ...` 提示减少或消失
- 最终报告更难引用提交、文件或代码变化

如果出现这个变化，就说明 GitHub 工具确实是由这些字段驱动的。

## 实验 2：把日志查询改成无意义查询

把：

```json
"elasticsearch_query": "error OR exception OR failed"
```

临时改成一个大概率匹配不到的值，例如：

```json
"elasticsearch_query": "zzzz_nonexistent_marker_12345"
```

然后重跑。

### 预期变化

- 仍可能看到 `query_opensearch_analytics`
- 但日志证据会明显变弱
- 最终报告里更难引用具体日志现象

这说明系统不是“假装用了日志”，而是真的受日志查询结果影响。

## E. 最短验证路径

如果你只想用 30 秒判断这次是否真正调用了 GitHub 和日志工具，就按这个最短清单走：

1. `uv run opensre doctor`
2. `uv run opensre integrations verify github`
3. 直接交互运行 `uv run opensre investigate -i docs/using-book/examples/ollama-template-repo-alert.json`
4. 在终端里确认：
   - `Resolved: [...]` 里有 `github` 和 `opensearch`
   - `Planned actions: [...]` 里有 GitHub 和日志动作
   - 调查阶段至少出现一个 GitHub `calling ...`
   - 调查阶段至少出现一个日志 `calling ...`

满足这四点，就可以认为：

> 这次 investigation 不是只做了空泛总结，而是确实把 GitHub 和日志两类外部证据接进了 pipeline。
