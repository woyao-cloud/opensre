# 写作计划

## 目标

这本 Using Book 不讲 OpenSRE 的内部架构细节，而是讲“怎么把它用起来”。

它要解决的问题是：

- 如何在当前仓库中安装并启动 OpenSRE。
- 如何把 `Kimi Code CLI` 配成可用的 LLM provider。
- 如何接入 GitHub 代码库与 Elasticsearch 日志。
- 如何准备一个输入告警，让 OpenSRE 能走完整条调查 pipeline。
- 如何验证这次运行确实用到了日志和代码仓库上下文。

## 场景约束

本书固定采用以下场景：

- 本地从当前仓库源码运行，不依赖远端部署。
- LLM provider 使用 `kimi`。
- 日志来自 Elasticsearch / OpenSearch 兼容接口。
- 代码仓库固定为 `woyao-cloud/ollama-03-02-claude-prompt02-template`。

## 为什么不完全按 docs 里的“通用向导”写

扫描代码后，当前最稳妥的使用路径不是“所有东西都交给交互式向导”，而是：

- `Kimi` 交给 `opensre onboard`
- GitHub MCP 交给 `.env`
- Elasticsearch/OpenSearch 交给 `.env`

原因有三个：

1. `Kimi` onboarding 在代码里是完整的一等路径，能自动探测 CLI 安装与登录状态。
2. GitHub MCP 的环境变量加载路径在 `app/integrations/_catalog_impl.py` 中明确可用。
3. Elasticsearch 日志的实际接入点是 `opensearch` 集成，运行时读 `OPENSEARCH_*`；这条路径比“交互式 setup -> store -> 规范化”更直接、更可控。

## 章节设计

### `01-overview.md`

说明最终要跑通什么、需要哪些前置条件、哪些不是本书范围。

### `02-install-and-kimi.md`

说明：

- 如何安装项目依赖
- 如何安装 `Kimi Code CLI`
- 如何登录 `kimi`
- 如何通过 `opensre onboard` 选择 `kimi`

### `03-github-and-elasticsearch.md`

说明：

- GitHub MCP 怎么配置
- Elasticsearch/OpenSearch 怎么配置
- 当前代码对 Elasticsearch 的真实支持边界是什么

### `04-alert-payload-and-examples.md`

说明：

- 一个“对这次场景足够友好”的告警 JSON 该怎么写
- 哪些字段会帮助 OpenSRE 发现 GitHub 和日志来源

### `05-step-by-step-runbook.md`

给出完全可照着执行的操作步骤。

### `06-pipeline-explained.md`

解释这一套配置为什么能跑通，以及运行时大概会走哪些工具。

### `07-troubleshooting.md`

列出最常见失败点和当前代码的几个重要限制。

### `08-target-repo-playbook.md`

专门补一个面向目标仓库的定制章节：

- 哪些仓库信息是已知事实
- 哪些是基于仓库名的推断
- `github_query` 应该怎么针对这类仓库改写
- 什么时候应该显式提供 `file_path`
- 针对该仓库的第二版示例告警

### `09-verification-checklist.md`

补一章手工验证清单：

- 运行前应该看哪些检查项
- 运行时应该看哪些流式终端信号
- 如何从 `Resolved:` / `Planned actions:` 判断外部证据是否真的接入
- 跑完后如何从报告内容做第二层验证
- 如何做最小对照实验

## 输出形式

输出保存到 `docs/using-book/` 目录下，采用多文件 Markdown 结构，并附：

- `.env` 示例
- 告警 JSON 示例
- 目标仓库定制示例
