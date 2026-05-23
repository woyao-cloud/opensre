# 安装 OpenSRE 与配置 Kimi

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

## 2. 安装 Kimi Code CLI

当前 `KimiAdapter` 的安装提示来自代码：

```bash
uv tool install --python 3.13 kimi-cli
```

如果你已经装过，也可以直接检查：

```bash
kimi --version
```

如果 `kimi` 不在 `PATH` 上，后面可以在 `.env` 里补：

```bash
KIMI_BIN=/full/path/to/kimi
```

## 3. 登录 Kimi

```bash
kimi login
```

当前代码会用两种方式判断认证状态：

- `kimi login status`
- `KIMI_API_KEY` 或 `~/.kimi/config.toml`

所以这两条路径都可以：

1. 直接 `kimi login`
2. 或者通过 `KIMI_API_KEY` / `KIMI_SHARE_DIR` 提供认证材料

但最简单的路径仍然是：

```bash
kimi login
```

## 4. 通过 onboarding 选择 `kimi`

运行：

```bash
uv run opensre onboard
```

在向导中：

1. 选择 `Kimi Code CLI`
2. 选择你想使用的模型，或者保持 CLI 默认模型
3. 让向导完成 CLI 探测

向导成功后，会把至少两个键写到项目根目录 `.env`：

```bash
LLM_PROVIDER=kimi
KIMI_MODEL=...
```

如果你故意留空模型，则 `KIMI_MODEL=` 为空，OpenSRE 会让 Kimi CLI 使用它自己的默认模型。

## 5. 用 `doctor` 验证 Kimi 路径

```bash
uv run opensre doctor
```

成功时你应该看到类似结果：

```text
llm_provider  provider=kimi, CLI ready (...)
```

如果失败，优先检查：

- `kimi` 是否真的可执行
- 是否已经 `kimi login`
- 如果二进制不在 PATH，是否设置了 `KIMI_BIN`

## 6. 为什么本书让 Kimi 走 onboarding

因为当前代码里，`opensre onboard` 对 CLI provider 的支持是完整的：

- 能探测安装状态
- 能探测登录状态
- 能把 `LLM_PROVIDER` 和模型写到 `.env`

这比手写 `.env` 更稳，也更符合当前项目的正式使用路径。
