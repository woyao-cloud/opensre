# 写作计划与章节规划

## 写作目标

这本 Guide Book 的目标不是复制 README，而是把 OpenSRE 的代码结构讲清楚，帮助读者回答：

- OpenSRE 解决什么问题
- 为什么这个项目需要 `pipeline + nodes + tools + integrations + services` 这么多层
- 一次调查是如何从输入告警推进到最终 RCA 报告的
- 每个模块解决什么问题，工作机制是什么
- 如果要新增一个 integration、tool 或 service，该改哪里

## 分析范围

本次分析范围限定在 `app/` 目录源码，不覆盖：

- `tests/` 中的详细断言逻辑
- `docs/` 中的产品说明
- GitHub Actions、CI、发布流水线

之所以这样切，是因为用户要求的是“关于此项目 OpenSRE 代码的书”，而 `app/` 是运行时代码主干。

## 写作方法

写作采用“先全局、再主链路、再能力层、最后横切能力与扩展”的顺序：

1. 先交代 OpenSRE 要解决的问题和整体设计。
2. 再梳理运行入口与统一执行图。
3. 之后深入状态、节点和调查闭环。
4. 然后分别展开 `integrations`、`tools`、`services` 三层。
5. 最后补 CLI、远程运行、认证、护栏、脱敏、sandbox 和扩展路径。

## 章节设计

### `01-overview.md`

回答三个最基础的问题：

- OpenSRE 解决什么问题
- 核心实现原理是什么
- 模块是如何分层的

### `02-runtime-and-entrypoints.md`

讲运行入口和系统是如何启动的：

- `opensre` CLI
- direct investigation
- interactive shell
- SDK
- MCP
- FastAPI health
- remote server

### `03-state-pipeline-and-nodes.md`

讲主干编排：

- `AgentState`
- `StateGraph`
- 调查分支
- 聊天分支
- 规划、执行、归并、诊断、循环和发布

### `04-integrations.md`

讲“OpenSRE 如何接第三方”：

- 配置模型
- registry
- catalog
- local store
- verify
- selectors
- env/store/remote 三种来源如何合并

### `05-tools.md`

讲“OpenSRE 如何把调查动作变成统一能力”：

- `BaseTool`
- `RegisteredTool`
- `@tool`
- registry auto-discovery
- surface 区分
- action prioritization
- 并发执行和重试

### `06-services.md`

讲“service 层为什么存在以及怎么设计”：

- 供应商 API client
- mixin 组合式 client
- LLM client 抽象
- CLI 型 LLM provider 适配

### `07-cli-remote-and-deployment.md`

讲运行面和外壳：

- CLI 命令注册
- onboarding/wizard
- interactive shell 路由
- remote FastAPI server
- remote client + SSE
- Railway / LangSmith 部署

### `08-cross-cutting-and-extension.md`

讲横切能力和二次开发：

- auth
- guardrails
- masking
- sandbox
- analytics / tracing
- 新增 integration / tool / node / service 的路径

### `09-cli-llm-adapters.md`

补一章专门讲“OpenSRE 如何调用外部 CLI LLM”：

- `llm_cli` 抽象合同
- provider registry
- `CLIBackedLLMClient` 的执行流程
- stdin / stdout 交互机制
- `kimi` 适配器的实现方式
- 结构化输出与限制

## 输出形式

输出保存到 `docs/Guide-book/` 目录下，使用 Markdown 多文件结构，方便：

- 按章节阅读
- 后续继续增量维护
- 在 docs 目录中与现有文档并存

## 本书的读者假设

默认读者是以下几类人：

- 第一次接手 OpenSRE 代码库的开发者
- 想为 OpenSRE 增加第三方集成的贡献者
- 想理解 LangGraph 在一个真实 Agent 项目中如何落地的工程师
- 想复用这一套“状态 + 图 + 工具 + 集成 + service”设计思路的人
