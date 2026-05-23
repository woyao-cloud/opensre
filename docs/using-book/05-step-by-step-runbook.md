# Step By Step 跑通手册

这一章是整本书的核心。  
照着做，目标是跑通一次完整的本地 investigation。

## Step 1. 安装依赖

在仓库根目录执行：

```bash
make install
```

或：

```bash
uv sync --frozen --extra dev
uv run python -m app.analytics.install
```

## Step 2. 安装并登录 Kimi CLI

```bash
uv tool install --python 3.13 kimi-cli
kimi login
```

如果 `kimi` 不在 PATH，上一步完成后记住它的完整路径，稍后填到：

```bash
KIMI_BIN=/full/path/to/kimi
```

## Step 3. 让 OpenSRE 选择 `kimi`

```bash
uv run opensre onboard
```

在向导中：

1. 选择 `Kimi Code CLI`
2. 选择模型，或者保持 CLI 默认
3. 确认检测通过

## Step 4. 补全 `.env`

把本书的示例 `.env` 里的 GitHub 和 OpenSearch 配置补进项目根目录 `.env`。

参考文件：

- [examples/env.kimi-github-opensearch.example](./examples/env.kimi-github-opensearch.example)

你至少需要填：

- `GITHUB_MCP_AUTH_TOKEN`
- `OPENSEARCH_URL`
- `OPENSEARCH_API_KEY`（如果你的集群需要）
- `OPENSEARCH_INDEX_PATTERN`

## Step 5. 验证 provider 和 GitHub

```bash
uv run opensre doctor
uv run opensre integrations verify github
uv run opensre health
```

### 你应该怎么看结果

- `doctor` 重点看 `llm_provider`
- `verify github` 重点看 MCP 是否通过
- `health` 重点看 OpenSearch/GitHub 是否出现在结果里

注意：

- `opensearch` 的 verify / health 当前偏“配置存在性检查”
- 真正能不能查到日志，还是要靠后面的 `investigate`

## Step 6. 准备告警 JSON

直接使用：

- [examples/elasticsearch-github-kimi-alert.json](./examples/elasticsearch-github-kimi-alert.json)
- [examples/ollama-template-repo-alert.json](./examples/ollama-template-repo-alert.json)

或者在此基础上替换：

- `elasticsearch_query`
- `opensearch_index_pattern`
- `pipeline_name`
- `message`

如果你希望 GitHub 侧更偏向“模板 / 提示词 / Modelfile”调查，优先使用第二个示例。

## Step 7. 执行调查

```bash
uv run opensre investigate -i docs/using-book/examples/elasticsearch-github-kimi-alert.json --output docs/using-book/examples/rca-result.json
```

如果你在交互终端里直接看流式过程，也可以不加 `--output`：

```bash
uv run opensre investigate -i docs/using-book/examples/elasticsearch-github-kimi-alert.json
```

## Step 8. 判断这次运行是否真的“跑通”

最少满足这几点：

1. 进程没有在 provider、auth、integration 阶段报错退出
2. 返回了最终 JSON
3. JSON 里至少有：
   - `report`
   - `problem_md`
   - `root_cause`

如果你使用了 `--output`，可以打开结果文件确认。

## Step 9. 进阶验证：确认它真的拿到了代码和日志上下文

从代码逻辑看，成功的调查应当具备这些条件：

- 告警被识别为 `elasticsearch` 来源
- `resolved_integrations` 中存在 `opensearch`
- `repo_url` 或 `github_owner/github_repo` 被识别

一旦这些条件成立，规划层就会把下列能力暴露给 LLM：

- `query_opensearch_analytics`
- `list_github_commits`
- `search_github_code`
- `get_github_file_contents`
- `get_git_deploy_timeline`

注意：

- 具体工具顺序是 planner 决定的，不保证每次完全相同
- 但如果源检测成功，这些就是理论上的可用调查动作

## Step 10. 可选：用交互式 shell 做后续追问

完成首次 file-based 调查后，你还可以进入 REPL：

```bash
uv run opensre
```

然后继续问：

```text
为什么它认为最近提交和异常日志有关？
```

这一步不是首次跑通所必需，但适合后续追问分析。

如果你接下来要确认“GitHub 工具和日志工具是否真的都被调用了”，继续看：

- [09-verification-checklist.md](./09-verification-checklist.md)
