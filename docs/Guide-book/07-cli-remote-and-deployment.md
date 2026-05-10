# CLI、交互式 Shell、远程运行与部署

这一章关注的是 OpenSRE 的“使用外壳”和“运行外壳”。

## CLI 命令系统

顶层 CLI 通过 `click` 实现，命令注册集中在：

- `app/cli/__main__.py`
- `app/cli/commands/__init__.py`

它有两个明显特点：

1. 顶层命令很多，说明项目定位是平台，不只是调查命令。
2. CLI 与调查核心解耦，更多是调度和外壳层。

## `investigate` 命令

`investigate` 最终会调用：

- `app.cli.investigation.run_investigation_cli`

这个函数会：

1. 校验 LLM settings
2. 解析 alert_name / pipeline_name / severity
3. 调用 pipeline runner
4. 返回 CLI 面向用户的结果结构

这说明 CLI 输出格式与内部状态结构不是完全一致的，中间有一层“外部返回值适配”。

## onboarding 向导

`app/cli/wizard/flow.py` 是本地 quickstart / onboarding 的核心。

### 它解决的问题

OpenSRE 的运行依赖很多配置：

- LLM provider
- API key
- 各类 integrations

如果全部靠用户手写 env 或 JSON，会很容易出错。

### 工作机制

向导层负责：

- 选择 provider
- 探测本地或远程目标
- 保存配置
- 调用 integration health 校验
- 写入本地 store

因此，wizard 本身就是“配置层的交互外壳”。

## 交互式 Shell

交互式 Shell 的核心位于：

- `interactive_shell/loop.py`
- `interactive_shell/router.py`
- `interactive_shell/command_registry/`
- `interactive_shell/cli_agent.py`

### 输入分类机制

`router.py` 会把输入分成：

- `slash`
- `cli_help`
- `cli_agent`
- `new_alert`
- `follow_up`

这套分类非常重要，因为它避免了“所有输入都送去 LangGraph 调查”的浪费。

例如：

- `/status` 应该走 slash command
- “how do I use opensre deploy” 应该走 CLI helper
- 一段事故描述才应该走 investigation pipeline

### Shell 的价值

它让 OpenSRE 不只是一个命令，而是一个“面向事故调查的终端环境”。

## MCP 作为外部协议面

MCP 入口位于：

- `app/entrypoints/mcp.py`

它做的事很克制：

- 接收 MCP tool 请求
- 复用现有 investigation CLI 能力
- 标准化返回值

这说明 OpenSRE 对外扩展协议时，优先做“薄包装”，而不是复制一套业务逻辑。

## 远程运行时：`app/remote/`

`remote/` 既有服务端，也有客户端。

### 服务端：`remote/server.py`

它是一个完整 `FastAPI` 服务，负责：

- 接收调查请求
- 运行调查
- 提供健康检查
- 持久化结果
- 处理流式返回

### 客户端：`remote/client.py`

它负责：

- 预检查远端服务状态
- 调用远端接口
- 解析 SSE 事件流
- 把事件转成统一 `StreamEvent`

### 渲染器：`remote/renderer.py`

它把远端流式事件渲染成和本地几乎一致的终端体验。

这套设计的关键价值是：

- 本地与远程不是两套用户体验
- 调查核心可以迁移到远程，而终端展示逻辑保持一致

## 部署模块：`app/deployment/`

部署逻辑集中在：

- `methods/langsmith.py`
- `methods/railway.py`
- `operations/health.py`

### Railway

`railway.py` 负责：

- 检查 CLI 是否安装
- 检查认证状态
- 设置环境变量
- 触发部署
- 轮询健康状态

### LangSmith / LangGraph

`langsmith.py` 负责：

- 检查 `langgraph` CLI
- 管理 LangSmith API key
- 运行 `langgraph build` 或 `langgraph deploy`

### 为什么部署逻辑单独分层

因为部署属于“如何运行 OpenSRE”，不是“OpenSRE 如何调查事故”。  
把它抽出可以避免调查主链混入平台运营逻辑。

## 健康检查与远程可观测性

健康相关的外壳包括：

- `webapp.py` 的 `/health`
- `remote/server.py` 的 `/ok` 和 deeper checks
- `remote/client.py` 的 preflight

它们共同支撑：

- 部署后自检
- CLI 健康命令
- 远程调用前探测

## 这一层的总体作用

如果说 `pipeline + nodes` 是 OpenSRE 的“业务内核”，  
那么 CLI、Shell、MCP、remote 和 deployment 就是它的“系统外壳”。

这些模块决定了 OpenSRE 能否：

- 作为本地开发工具使用
- 作为交互式调查终端使用
- 作为远程服务部署
- 作为外部协议工具接入

这也是 OpenSRE 之所以像“平台”而不是“单脚本 agent”的原因。
