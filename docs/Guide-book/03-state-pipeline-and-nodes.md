# 状态模型、执行图与节点编排

这一章是全书核心。  
如果只想理解“一次 OpenSRE 调查到底怎么跑起来”，这章最重要。

## 统一状态模型：`app/state/`

OpenSRE 以 `AgentState` 作为整个图执行过程的共享上下文。

主要定义位于：

- `app/state/agent_state.py`
- `app/state/factory.py`
- `app/state/types.py`

这里有两个同步表示：

- `AgentState`：`TypedDict`，给 LangGraph 用
- `AgentStateModel`：Pydantic 模型，给校验和构造器用

### 为什么必须统一状态

调查流程不是单步函数调用，而是一个多阶段、可循环、可并发的执行图。  
如果没有统一状态，节点之间就要靠大量零散参数传递，最终会让：

- 节点接口不稳定
- 回环很难做
- 远程运行和本地运行难以统一

### 状态里主要放什么

`AgentState` 里大致包含以下几类信息：

- 模式与路由：`mode`、`route`
- 认证上下文：`org_id`、`user_id`、`thread_id`、`run_id`
- 输入告警：`raw_alert`、`alert_name`、`severity`
- 规划信息：`planned_actions`、`available_sources`、`tool_budget`
- 证据信息：`context`、`evidence`、`executed_hypotheses`
- 调查结果：`root_cause`、`validated_claims`、`investigation_recommendations`
- 输出：`slack_message`、`report`
- 保护性上下文：`incident_window`、`masking_map`

## 初始状态如何构造

`app/state/factory.py` 负责状态工厂。

### `make_initial_state()`

用于 investigation 模式，做几件事：

1. 处理 OpenRCA 评估 rubric
2. 规范化原始告警载荷
3. 注入基础默认字段
4. 生成严格校验后的初始状态

### `make_chat_state()`

用于 chat 模式，初始化消息和用户上下文。

这说明 OpenSRE 从一开始就把“聊天”和“调查”放在同一个状态体系里，只是模式不同。

## 执行图：`app/pipeline/graph.py`

`build_graph()` 构建一个 `StateGraph(AgentState)`。

图的入口是：

- `inject_auth`

然后根据 `route_by_mode()` 分叉成：

- `chat`
- `investigation`

这是一种很重要的设计：  
不是两个完全独立系统，而是一个统一图中有两条主路径。

## 调查主链路

调查分支的主干是：

```text
inject_auth
  -> extract_alert
  -> resolve_integrations
  -> plan_actions
  -> investigate_hypothesis (fan-out)
  -> merge_hypothesis_results
  -> diagnose
      -> adapt_window -> plan_actions
      -> opensre_eval -> publish
      -> publish
```

下面按顺序解释。

## 节点 1：`extract_alert`

实现位于：

- `app/nodes/extract_alert/extract.py`
- `app/nodes/extract_alert/extract_node.py`

### 解决的问题

输入告警格式不统一，可能来自：

- Slack 文字
- Grafana / Alertmanager webhook
- Datadog 风格告警
- OpenRCA 数据集
- 其他 JSON 结构

这个节点负责把“原始输入”转成“调查可用结构”。

### 工作机制

1. 根据输入内容决定是否需要把整个 JSON 暴露给 LLM。
2. 通过一次 LLM 结构化抽取，得到：
   - `alert_name`
   - `pipeline_name`
   - `severity`
   - `alert_source`
   - `kube_namespace`
   - `error_message`
   - 其他调查路由字段
3. 若 LLM 失败，则走 fallback 逻辑。
4. 对原始告警做 enrichment，把抽取出的字段补回原始字典。
5. 解析 `incident_window`，为后续时序工具提供统一时间范围。

这个节点的本质是“输入标准化器 + 初始调查种子”。

## 节点 2：`resolve_integrations`

实现位于：

- `app/nodes/resolve_integrations/node.py`

### 解决的问题

后续很多工具都依赖第三方集成，但这些集成的来源不固定：

- 可能来自远端组织配置
- 可能来自本地 store
- 可能来自环境变量

调查图不应该在每个节点都重复处理这些逻辑。

### 工作机制

它采用优先级回退策略：

1. 如果状态里有 `_auth_token`，优先按组织从远端拉取 integrations
2. 如果有 `JWT_TOKEN`，尝试远端拉取再回填本地缺口
3. 否则退回本地 store + env

最终产物是一个标准化的 `resolved_integrations` 字典，供后续节点和工具使用。

## 节点 3：`plan_actions`

实现位于：

- `app/nodes/plan_actions/node.py`
- `app/nodes/plan_actions/plan_actions.py`
- `app/nodes/plan_actions/detect_sources.py`
- `app/nodes/plan_actions/build_prompt.py`

### 解决的问题

调查系统不能“盲查一切”。它需要知道：

- 这条告警可能涉及哪些系统
- 当前有哪些工具可用
- 预算允许执行多少动作
- 哪些动作已经执行过，不该重复

### 工作机制

#### 第一步：检测数据源

`detect_sources()` 从以下信息中识别 source：

- 告警 annotations / labels
- enriched top-level fields
- `resolved_integrations`
- 部分上下文信息

输出是 `available_sources`，例如：

- `grafana`
- `datadog`
- `cloudwatch`
- `github`
- `s3`
- `betterstack`
- `openclaw`
- `eks`

#### 第二步：候选动作筛选

候选动作来自 `app/tools/investigation_registry/`。

系统会按：

- source 相关性
- 关键词匹配
- 已执行动作屏蔽
- tool budget

来选出当前轮可执行动作。

#### 第三步：交给工具规划 LLM

`build_prompt.py` 负责把：

- 问题描述
- 可用动作
- 可用数据源提示
- 已执行历史

拼成提示词，并要求 LLM 返回结构化计划。

#### 第四步：控制器兜底

如果 LLM 返回空计划，控制器会强制挑一个保底动作，避免图陷入空转。

因此，这个节点是“LLM 规划 + 代码约束”的结合体，而不是完全信任模型。

## 节点 4：`investigate_hypothesis`

实现位于：

- `app/nodes/investigate/parallel.py`

### 解决的问题

一轮调查通常不是一个动作，而是一批动作。  
例如同时查日志、查指标、查部署时间线。

### 工作机制

1. 从 `action_to_run` 读取一个具体动作名。
2. 从 investigation registry 中找到对应工具。
3. 调用 `execute_actions()` 执行。
4. 把结果序列化为 `hypothesis_results`。

图通过 `distribute_hypotheses()` 做 `Send(...)` 扇出，因此多个动作是并行子图。

## 节点 5：`merge_hypothesis_results`

实现位于：

- `app/nodes/investigate/merge.py`
- `app/nodes/investigate/processing/post_process.py`

### 解决的问题

并行执行之后，需要把多个动作结果：

- 合并回统一证据集
- 形成执行历史
- 更新下一轮规划所需上下文

### 工作机制

1. 把 `hypothesis_results` 反序列化回 `ActionExecutionResult`
2. 汇总 `evidence` 和 `executed_hypotheses`
3. 必要时把 OpenSRE benchmark telemetry 注入证据
4. 更新 `available_sources`
5. 对证据做 masking
6. 清空已消费的 `hypothesis_results`

这个节点是“并行结果归并器”。

## 节点 6：`diagnose`

实现位于：

- `app/nodes/root_cause_diagnosis/node.py`
- `prompt_builder.py`
- `claim_validator.py`
- `evidence_checker.py`

### 解决的问题

收集到证据后，系统需要：

- 给出根因结论
- 判断证据是否支撑这些结论
- 决定是继续调查还是结束

### 工作机制

1. 先做证据充足性检查。
2. 对明显健康场景走 deterministic short-circuit，不调用 LLM。
3. 对证据不足场景给出“需要继续查”的 recommendation。
4. 正常场景下构建 prompt，调用 reasoning LLM。
5. 对 LLM 输出做 `parse_root_cause()`。
6. 对 claims 做证据校验并计算 `validity_score`。
7. 如有必要，设置 `investigation_recommendations`，驱动下一轮循环。

这说明根因诊断不是“模型说了算”，而是：

- 规则前置
- LLM 推理
- 证据回验

三步组合。

## 节点 7：`adapt_window`

实现位于：

- `app/nodes/adapt_window/node.py`
- `app/nodes/adapt_window/rules.py`

### 解决的问题

当第一轮时间窗口内没有查到足够证据时，系统不应该原地重复同一查询。

### 工作机制

- 根据规则判断是否要扩展事故时间窗
- 把旧窗口写入 `incident_window_history`
- 将新窗口写回状态
- 再回到 `plan_actions`

因此，回环不是盲重试，而是带状态调整的重规划。

## 节点 8：`opensre_eval`

实现位于：

- `app/nodes/evaluate_opensre/node.py`

### 作用

这是一个可选节点，用于把最终调查结果拿去和 OpenRCA rubric 比较评分。  
它更偏评估和 benchmark，而不是正常事故处理主路径。

## 节点 9：`publish`

实现位于：

- `app/nodes/publish_findings/node.py`
- `formatters/`
- `renderers/`
- `report_context.py`

### 解决的问题

调查最终要变成可读、可投递、可归档的结果，而不是只留在状态里。

### 工作机制

1. 从状态构建报告上下文
2. 格式化 Slack message / report
3. 反脱敏恢复用户可读标识符
4. 写入 investigation URL
5. 发送到 Slack / Discord / Telegram
6. 触发 GitLab MR writeback 等附加动作

这个节点把“内部状态”转换成“外部可消费结果”。

## 聊天分支

聊天分支实现位于 `app/nodes/chat.py`。

主要节点：

- `router_node`
- `chat_agent_node`
- `general_node`
- `tool_executor_node`

### 工作机制

- `router_node` 判断用户输入是普通问答还是数据型问题
- `chat_agent_node` 给 chat LLM 绑定 chat surface 工具
- `tool_executor_node` 执行 AI 发起的 tool calls
- `general_node` 在无需工具时直接回答

这条路径和调查分支共享图运行时，但服务于完全不同的交互方式。

## 本章小结

OpenSRE 的执行主线可以概括为：

1. 标准化输入
2. 解析可用能力
3. 规划动作
4. 并发取证
5. 汇总证据
6. 诊断根因
7. 必要时扩大窗口并重新规划
8. 发布结果

如果只从代码架构上理解 OpenSRE，这条主链就是它的“业务骨架”。
