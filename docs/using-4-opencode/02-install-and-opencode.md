# 安装 OpenSRE 与配置 OpenCode

## 1. 安装项目依赖

在仓库根目录执行：

```bash
make install
```

如果你没有 `make`，使用等价命令：

```bash
uv sync --frozen --extra dev
uv run python -m app.analytics.install
```

之后统一从仓库根目录运行：

```bash
uv run opensre version
```

如果能看到版本输出，说明本地可执行入口已经可用。

## 2. 安装 OpenCode CLI

OpenCode 的安装方式取决于你的操作系统。

**macOS：**

```bash
brew install anomalyco/tap/opencode
```

**Windows：**

```bash
choco install opencode
```

**Linux：**

参考 [OpenCode 官方安装文档](https://github.com/anomalyco/opencode)。

安装后验证：

```bash
opencode --version
```

如果 `opencode` 不在 `PATH` 上，后面可以在 `.env` 里补：

```bash
OPENCODE_BIN=/full/path/to/opencode
```

### OpenCode 适配器的安装提示

当前代码来自 `app/integrations/llm_cli/opencode.py`：

- macOS/Linux: `brew install anomalyco/tap/opencode`
- Windows: `choco install opencode`
- 或者设置 `OPENCODE_BIN` 环境变量指向完整二进制路径

## 3. 登录 OpenCode

OpenCode 支持多种认证方式：

### 方式 A：交互式登录（推荐）

```bash
opencode auth login
```

这会生成 `auth.json`，包含你的认证凭证。

### 方式 B：环境变量 API Key

OpenCode 会自动检测以下环境变量：

- `ANTHROPIC_API_KEY`
- `OPENAI_API_KEY`
- `OPENROUTER_API_KEY`
- `GEMINI_API_KEY`
- 等等（见 `app/integrations/llm_cli/env_overrides.py` 的 `HTTP_LLM_PROVIDER_ENV_KEYS`）

### 验证登录状态

```bash
opencode auth list
```

你会看到类似输出：

```
Credentials:
  1 credential group(s) in auth store
  Environment variables: 2 provider key(s)
```

当前 OpenSRE 的 `_parse_opencode_auth_list_output` 会解析这个输出来判断 `logged_in` 状态：只要 credentials ≥ 1 或 environment variables ≥ 1，就认为已登录。

## 4. 通过 onboarding 选择 `opencode`

运行：

```bash
uv run opensre onboard
```

在向导中：

1. 选择 **OpenCode CLI**（位于 "Local CLI providers" 分组下）
2. 选择模型，例如 `anthropic/claude-opus-4.7`，或保持 CLI 默认
3. 让向导完成 CLI 探测（安装检测 + 登录检测）

向导成功后，会把至少两个键写到项目根目录 `.env`：

```bash
LLM_PROVIDER=opencode
OPENCODE_MODEL=anthropic/claude-opus-4.7
```

如果你故意留空模型，则 `OPENCODE_MODEL=` 为空，OpenSRE 会让 OpenCode CLI 使用它自己的默认模型。

## 5. 用 `doctor` 验证 OpenCode 路径

```bash
uv run opensre doctor
```

成功时你应该看到类似结果：

```text
llm_provider  provider=opencode, CLI ready (...)
```

失败时的典型输出：

```text
llm_provider  provider=opencode, CLI not found (...)
```

如果失败，优先检查：

- `opencode` 是否真的可执行
- 是否已经 `opencode auth login` 或设置了供应商 API Key
- 如果二进制不在 PATH，是否设置了 `OPENCODE_BIN`

## 6. 为什么本书让 OpenCode 走 onboarding

因为当前代码里，`opensre onboard` 对 CLI provider 的支持是完整的：

- 能探测安装状态（`opencode --version`）
- 能探测登录状态（`opencode auth list`）
- 能把 `LLM_PROVIDER` 和模型写到 `.env`

这比手写 `.env` 更稳，也更符合当前项目的正式使用路径。
