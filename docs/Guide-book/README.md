# OpenSRE Code Guide Book

这是一套基于 `app/` 目录源码扫描整理出的 OpenSRE 代码导读书，目标是回答三类问题：

1. OpenSRE 解决什么问题。
2. OpenSRE 的实现原理是什么。
3. OpenSRE 由哪些模块组成，这些模块如何协作。

本书不是产品使用手册，而是面向开发者的代码理解指南。重点放在：

- 主执行链路
- 核心模块职责
- 关键抽象
- 与第三方系统的集成方式
- 工具调用和 service 封装的实现机制

## 阅读顺序

1. [00-plan.md](./00-plan.md)
2. [01-overview.md](./01-overview.md)
3. [02-runtime-and-entrypoints.md](./02-runtime-and-entrypoints.md)
4. [03-state-pipeline-and-nodes.md](./03-state-pipeline-and-nodes.md)
5. [04-integrations.md](./04-integrations.md)
6. [05-tools.md](./05-tools.md)
7. [06-services.md](./06-services.md)
8. [07-cli-remote-and-deployment.md](./07-cli-remote-and-deployment.md)
9. [08-cross-cutting-and-extension.md](./08-cross-cutting-and-extension.md)

## 一页结论

OpenSRE 是一个面向生产事故调查与根因分析的 AI SRE Agent 框架。它要解决的问题不是“如何调用某一个 LLM”，而是“当故障线索分散在日志、指标、追踪、CI/CD、工单、云资源和聊天上下文里时，如何把调查过程变成一条可编排、可扩展、可验证的程序链路”。

它的实现核心是：

- 用 `LangGraph` 建立统一执行图
- 用 `AgentState` 承载调查上下文
- 用 `integrations` 把第三方能力标准化
- 用 `tools` 把可执行调查动作注册成统一能力面
- 用 `services` 封装供应商 API、LLM 和远端系统访问
- 用 `nodes` 把“抽取告警 -> 识别来源 -> 规划动作 -> 并发取证 -> 诊断 -> 发布”串成闭环

## `app/` 模块版图

以下统计来自本次对 `app/` 目录的扫描：

| 模块目录 | Python 文件数 | 作用摘要 |
| --- | ---: | --- |
| `tools/` | 151 | 可执行调查动作，按 source 和 surface 注册 |
| `cli/` | 95 | 命令行入口、交互式 shell、向导、辅助命令 |
| `integrations/` | 62 | 集成配置建模、归一化、验证、选择、存储 |
| `services/` | 54 | 供应商 API 客户端、LLM 抽象、Grafana/Tracer 等服务封装 |
| `nodes/` | 49 | 图执行节点，负责调查编排 |
| `utils/` | 12 | 投递、配置、截断、Sentry 等通用工具 |
| `remote/` | 11 | 远程运行时、流式渲染、远程客户端/服务端 |
| `deployment/` | 8 | Railway/LangSmith 部署与健康操作 |
| `analytics/` | 5 | CLI 分析与埋点 |
| `guardrails/` | 5 | 敏感信息规则、扫描、阻断和审计 |
| `types/` | 5 | 跨模块共享类型合同 |
| `constants/` | 4 | 常量和提示词 |
| `masking/` | 4 | 可逆脱敏 |
| `pipeline/` | 4 | 图构建、路由和 runner |
| `state/` | 4 | 统一状态模型和工厂 |
| `entrypoints/` | 3 | SDK/MCP 等程序入口 |
| `alerts/` | 2 | 告警规范化 |
| `auth/` | 2 | JWT 和 LangGraph Auth |
| `sandbox/` | 2 | 受限 Python 执行 |

可以看出，OpenSRE 不是“一个大模型脚本”，而是一个明显分层的 Agent 平台。

## 推荐读法

- 第一次接触项目：先读 `01`、`02`、`03`
- 想看三方集成怎么接：读 `04`
- 想看工具规划和执行：读 `05`
- 想看供应商 API 与 LLM 是怎么包的：读 `06`
- 想理解 CLI、MCP、远端部署入口：读 `07`
- 想二次开发：读 `08`
