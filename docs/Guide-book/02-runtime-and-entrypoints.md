# 运行入口与执行外壳

这一章回答的问题是：  
OpenSRE 从哪里启动？不同运行面之间是什么关系？

## 入口全景

OpenSRE 不是只有一个 `main()`。它有多种入口，但大都复用同一套调查能力。

主要入口包括：

- `app.cli.__main__:main`
- `app.main.py`
- `app.entrypoints.sdk:run_investigation`
- `app.entrypoints.mcp:main`
- `app.webapp:app`
- `app.remote.server:app`

## 1. 顶层 CLI：`app/cli/__main__.py`

这是 `opensre` 命令的主入口。

它负责：

- 加载环境变量
- 初始化 Sentry
- 注册顶层命令
- 决定是否进入交互式 REPL
- 处理全局异常、analytics 和退出码

命令注册集中在 `app/cli/commands/__init__.py`，顶层命令包括：

- `investigate`
- `onboard`
- `deploy`
- `remote`
- `tests`
- `integrations`
- `guardrails`
- `health`
- `doctor`
- `update`
- `uninstall`
- `version`

这说明 CLI 在 OpenSRE 中不只是“一个启动脚本”，而是完整的运维和使用外壳。

## 2. 直接调查入口：`app/main.py`

`app/main.py` 是一个更窄的入口，主要服务于单次调查：

- 读取输入载荷
- 调用 `run_investigation_cli`
- 输出结果 JSON

它更像一个“直接执行调查任务”的轻量包装层。

## 3. SDK 入口：`app/entrypoints/sdk.py`

SDK 入口非常薄，只暴露：

- `run_investigation()`

它延迟导入 `app.pipeline.runners.run_investigation`，目的是：

- 减少导入时的依赖抖动
- 让外部程序可以把 OpenSRE 当成库来调用

## 4. MCP 入口：`app/entrypoints/mcp.py`

这里把调查能力封装成 MCP tool：

- MCP server 名称：`opensre`
- 暴露工具：`run_rca`

工作机制：

1. 接收 `alert_payload` 和可选覆盖字段。
2. 对 `commonLabels` 做最小化改写。
3. 调用现有 CLI 调查入口 `run_investigation_cli`。
4. 返回结构化的 `ok/result/error/suggestion`。

也就是说，MCP 不是另一套实现，而是把现有调查能力重新包装成标准工具协议。

## 5. 健康检查入口：`app/webapp.py`

`app/webapp.py` 暴露一个极简 `FastAPI` 应用，提供 `/health`。

它检查三件事：

- graph 是否加载
- LLM 配置是否有效
- 当前运行环境和版本

它的目标不是承载完整业务，而是作为部署场景中的 readiness / liveness 面。

## 6. 远程服务入口：`app/remote/server.py`

这是另一个重要的 `FastAPI` 服务面。

它负责：

- 远程接收调查请求
- 运行调查
- 产出并保存结果
- 处理流式返回
- 暴露 health、version 等接口
- 作为远程部署形态的一部分运行

它和 `webapp.py` 的区别是：

- `webapp.py` 是轻量健康接口
- `remote/server.py` 是完整远程调查服务

## 7. 交互式 Shell：`app/cli/interactive_shell/`

这是 OpenSRE 的另一个“外壳系统”，不是简单的 REPL。

核心机制：

- `loop.py`：异步循环与终端渲染
- `router.py`：将输入分类为 slash / CLI help / CLI agent / new alert / follow-up
- `commands.py` 与 `command_registry/`：slash 命令系统
- `cli_agent.py`：LangGraph 外的轻量终端助手

Shell 的一个关键设计是：

- 终端帮助和命令引导，不一定要走调查图
- 只有真正像事故调查输入的内容，才会进入调查 pipeline

这让 OpenSRE 的“命令助手”和“事故调查 agent”可以共存而不互相污染。

## 本地运行与远程运行的关系

OpenSRE 在本地和远程运行时，尽量复用同一套主链路。

### 本地

- 通过 `app.pipeline.runners.run_investigation`
- 直接构造初始状态并调用编译后的 LangGraph

### 远程

- 通过 `remote/server.py` 或 LangGraph 托管运行时接收请求
- 同样运行调查主链
- 通过 SSE 或封装接口把结果流回客户端

这使得“运行位置”成为部署差异，而不是业务差异。

## LangGraph 托管入口

`langgraph.json` 指定了托管运行时需要的三个关键入口：

- graph：`app.graph_pipeline:build_graph`
- auth：`app.auth.auth:auth`
- http：`app.webapp:app`

其中 `app/graph_pipeline.py` 只是一个兼容层，真正的图构建在 `app/pipeline/graph.py`。

## 运行外壳的设计含义

从架构角度看，这些入口层的价值在于：

- 把“如何运行系统”与“系统如何调查事故”分离
- 允许 CLI、MCP、远程服务和 LangGraph 托管共用一套内核
- 保留多个面向不同用户群的交互方式

因此，入口层不是业务核心，但它决定了 OpenSRE 能否成为一个真正的工程化平台，而不是单一脚本。
