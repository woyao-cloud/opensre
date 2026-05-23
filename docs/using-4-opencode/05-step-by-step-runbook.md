# Step By Step 跑通手册

这一章是整本书的核心。照着做，目标是跑通一次完整的本地 investigation。

全书提供两套数据源路径，根据你的实际环境选择 **方案 A（ES）** 或 **方案 B（Kafka）**。

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

## Step 2. 安装并登录 OpenCode CLI

```bash
# macOS
brew install anomalyco/tap/opencode

# Windows
choco install opencode

# 验证安装
opencode --version

# 登录（二选一）
opencode auth login                        # 交互式
# 或 export ANTHROPIC_API_KEY=sk-...      # 环境变量

# 验证登录
opencode auth list
```

如果 `opencode` 不在 PATH，记住它的完整路径，稍后填到：

```bash
OPENCODE_BIN=/full/path/to/opencode
```

## Step 3. 让 OpenSRE 选择 `opencode`

```bash
uv run opensre onboard
```

在向导中：

1. 选择 **OpenCode CLI**（位于 "Local CLI providers" 分组）
2. 选择模型，如 `anthropic/claude-opus-4.7`，或保持 CLI 默认
3. 确认检测通过

## Step 4. 补全 `.env`

把对应的示例 `.env` 配置补进项目根目录 `.env`。

### 方案 A：Elasticsearch/OpenSearch

参考 [examples/env.opencode-github-opensearch.example](./examples/env.opencode-github-opensearch.example)

你至少需要填：

- `GITHUB_MCP_AUTH_TOKEN`
- `OPENSEARCH_URL`
- `OPENSEARCH_API_KEY`（如果你的集群需要）
- `OPENSEARCH_INDEX_PATTERN`

### 方案 B：Kafka

参考 [examples/env.opencode-github-kafka.example](./examples/env.opencode-github-kafka.example)

你至少需要填：

- `GITHUB_MCP_AUTH_TOKEN`
- `KAFKA_BOOTSTRAP_SERVERS`
- `KAFKA_SECURITY_PROTOCOL`（和 SASL 信息，如需）

## Step 5. 验证 provider、GitHub 和集成

```bash
uv run opensre doctor
uv run opensre integrations verify github
uv run opensre health
```

### 你应该怎么看结果

- `doctor` 重点看 `llm_provider`：应显示 `provider=opencode, CLI ready`
- `verify github` 重点看 MCP 是否通过
- `health` 重点看 GitHub 和数据源是否出现在结果里

注意：

- `opensearch` 的 verify / health 当前偏"配置存在性检查"
- Kafka 的验证也是类似的配置存在性检查
- 真正能不能查到数据，还是要靠后面的 `investigate`

## Step 6. 准备告警 JSON

### 方案 A：ES 场景

直接使用：

- [examples/elasticsearch-github-opencode-alert.json](./examples/elasticsearch-github-opencode-alert.json)
- [examples/ollama-template-repo-alert.json](./examples/ollama-template-repo-alert.json)

### 方案 B：Kafka 场景

直接使用：

- [examples/kafka-github-opencode-alert.json](./examples/kafka-github-opencode-alert.json)

或者在以上基础上替换真实查询参数。

## Step 7. 执行调查

```bash
uv run opensre investigate -i docs/using-4-opencode/examples/elasticsearch-github-opencode-alert.json
```

或者 Kafka 版本：

```bash
uv run opensre investigate -i docs/using-4-opencode/examples/kafka-github-opencode-alert.json
```

如果需要保存结果：

```bash
uv run opensre investigate -i docs/using-4-opencode/examples/elasticsearch-github-opencode-alert.json --output docs/using-4-opencode/examples/rca-result.json
```

## Step 8. 判断这次运行是否真的"跑通"

最少满足这几点：

1. 进程没有在 provider、auth、integration 阶段报错退出
2. 返回了最终 JSON
3. JSON 里至少有：`report`、`problem_md`、`root_cause`

如果你使用了 `--output`，可以打开结果文件确认。

## Step 9. 进阶验证：确认它真的拿到了代码和数据源上下文

从代码逻辑看，成功的调查应当具备这些条件：

- 告警被识别为 `elasticsearch` 或 `kafka` 来源
- `resolved_integrations` 中存在 `opensearch` 或 `kafka`
- `repo_url` 或 `github_owner/github_repo` 被识别

一旦这些条件成立，规划层就会把相应的调查能力暴露给 LLM：

**ES 场景可用工具：**

- `query_opensearch_analytics`
- `list_github_commits`
- `search_github_code`
- `get_github_file_contents`
- `get_git_deploy_timeline`

**Kafka 场景可用工具：**

- `get_kafka_topic_health`
- `get_kafka_consumer_group_lag`
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

接下来确认"GitHub 工具和日志工具是否真的都被调用了"，继续看：

- [09-verification-checklist.md](./09-verification-checklist.md)
