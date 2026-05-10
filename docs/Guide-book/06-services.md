# `services` 模块：供应商 API、LLM 与统一访问层

这一章回答：  
为什么 OpenSRE 还需要 `services`？它和 `integrations`、`tools` 的关系是什么？

## `services` 的定位

`services` 处在一个非常关键的位置：

- `integrations` 负责把配置标准化
- `tools` 负责把动作暴露成统一能力
- `services` 负责真正访问外部系统

如果把 `integrations` 比作“接线规范”，把 `tools` 比作“操作面板”，那么 `services` 就是“具体执行器”。

## 为什么不让工具直接写 HTTP 请求

因为那会造成：

- 同一供应商访问逻辑在多个工具里重复
- 认证、重试、URL 构造散落 everywhere
- 验证逻辑和业务查询逻辑难复用
- 工具实现变重，难维护

OpenSRE 的策略是：

- 工具负责“要查什么”
- service 负责“怎么查”

## `services/` 的主要类型

### 1. 供应商 API client

例如：

- `services/datadog/client.py`
- `services/grafana/`
- `services/jira/client.py`
- `services/splunk/client.py`
- `services/vercel/client.py`

### 2. 平台内部 client

例如：

- `services/tracer_client/`
- `services/aws_sdk_client.py`
- `services/cloudwatch_client.py`

### 3. LLM client

最核心的是：

- `services/llm_client.py`

### 4. service-specific helper packages

例如：

- `services/grafana/`
- `services/eks/`
- `services/google_docs/`

## 典型模式一：直接 HTTP client

以 `DatadogClient` 为例。

它的特点是：

- 直接使用 `httpx`
- 以标准化 config 作为输入
- 暴露清晰的方法，如 `search_logs()`、`list_monitors()`、`get_events()`
- 返回统一 dict，而非 Datadog SDK 原始对象

### 好处

- 不引入额外 SDK 依赖
- 响应结构由项目自己掌控
- probe 和业务查询可以复用同一个 client

这类 client 适合 HTTP API 比较直接、项目想完全掌控序列化形态的场景。

## 典型模式二：组合式 client

以 `GrafanaClient` 为例。

文件结构是：

- `grafana/base.py`
- `grafana/loki.py`
- `grafana/mimir.py`
- `grafana/tempo.py`
- `grafana/client.py`

最终：

- `GrafanaClient = LokiMixin + TempoMixin + MimirMixin + GrafanaClientBase`

### 这解决了什么问题

Grafana 不是单一 API，而是多个能力面：

- Loki
- Tempo
- Mimir
- alert rules

如果写成一个巨大类，会非常难维护。  
OpenSRE 用 mixin 拆能力，再组装成统一 client。

### 设计收益

- 每个后端能力单独演进
- 共享认证和基础请求逻辑
- 工具层只需要拿到一个统一 `GrafanaClient`

## 典型模式三：统一内部平台 client

以 `TracerClient` 为例。

它同样是 mixin 组合：

- `TracerPipelinesMixin`
- `TracerToolsMixin`
- `AWSBatchJobsMixin`
- `TracerLogsMixin`
- `TracerIntegrationsMixin`

共同依赖：

- `TracerClientBase`

这说明 OpenSRE 对“内部平台 API”也采用与外部平台相同的工程化方式，而不是特判。

## LLM 层：`services/llm_client.py`

这是整个 service 层里最特殊的一块。

### 它解决的问题

OpenSRE 既要支持：

- Anthropic / OpenAI / Gemini / Bedrock 等 API 型 provider
- Codex / Cursor / Claude Code 等 CLI 型 provider

又要在上层提供统一能力：

- `invoke()`
- `invoke_stream()`
- `with_structured_output()`
- `bind_tools()`

### 它的抽象方式

运行时对上暴露两类主要 client：

- `get_llm_for_reasoning()`
- `get_llm_for_tools()`

这样做的原因是：

- 诊断推理与工具规划对模型能力要求不同
- 成本与速度要求也不同

### API 型 provider

主要封装：

- Anthropic
- OpenAI-compatible providers
- Bedrock

这些 provider 共享：

- 认证处理
- 重试与超时
- guardrail 应用
- 结构化输出包装

### CLI 型 provider

通过：

- `integrations/llm_cli/runner.py`

实现 `CLIBackedLLMClient`。

它的工作机制是：

1. 通过 adapter 先做二进制和登录态探测
2. 将消息 flatten 成 prompt
3. 构造非交互 subprocess 调用
4. 捕获 stdout/stderr
5. 统一包装成 `LLMResponse`

这让 OpenSRE 可以把本来只适合人类终端使用的 CLI 模型，也纳入同一 Agent 运行时。

## `services` 如何与 `integrations` 和 `tools` 协作

三者的关系可以概括成：

### `integrations`

负责给出“正确配置”。

### `services`

负责用配置访问外部系统。

### `tools`

负责把 service 能力包成调查动作，并接进规划/执行链。

一个典型调用链是：

1. `resolve_integrations` 得到标准化 Grafana 配置
2. `detect_sources()` 识别出 `available_sources["grafana"]`
3. `query_grafana_logs` 工具从 source 提取参数
4. 工具调用 `get_grafana_client_from_credentials()`
5. `GrafanaClient` 发请求并返回标准化日志结果
6. merge / diagnose 节点消费这份结果

## service 设计上的共同特征

从现有代码看，好的 service 模块通常有这些共性：

- 输入是标准化 config，而不是散乱 env
- 公开方法语义清楚，围绕业务能力命名
- 返回值对上层友好，尽量不暴露供应商原生复杂结构
- 有 `probe` 或 validate 能力时，会共用 client
- 复用基础请求、认证和 URL 逻辑

## 为什么说 `services` 是 OpenSRE 的“适配层”

因为它既不做业务编排，也不直接面向终端用户。  
它做的是把外部世界翻译成 OpenSRE 可以稳定消费的调用接口。

这一层写得好，工具层和节点层就能保持轻。  
这一层写得差，整个项目最终会退化成“每个工具都手搓半个供应商 SDK”。
