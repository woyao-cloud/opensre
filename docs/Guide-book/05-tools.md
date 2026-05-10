# `tools` 模块：如何把调查动作变成统一能力

这一章回答：  
OpenSRE 的工具系统是怎么设计的？工具如何注册、筛选、执行？

## 为什么需要 `tools`

调查系统最终落地时，一定要变成一组可执行动作。  
例如：

- 查 Grafana Loki 日志
- 查 Datadog monitor
- 查 GitHub commits
- 查 S3 审计对象
- 查 RabbitMQ backlog

如果不把这些动作抽象成统一工具，系统就会出现两个问题：

- 规划层不知道“有哪些动作可选”
- 执行层无法统一做 availability、参数抽取、并发和重试

因此 OpenSRE 把调查动作抽成单独的工具层。

## `tools/` 的结构

`app/tools/` 是代码量最大的目录之一。

其中主要有四类文件：

1. 工具元数据与注册基础设施  
   - `base.py`
   - `registered_tool.py`
   - `tool_decorator.py`
   - `registry.py`

2. 调查动作注册表  
   - `investigation_registry/`

3. 具体工具实现  
   - 各种 `...Tool/__init__.py`

4. 工具辅助函数  
   - `tools/utils/`

## 工具元数据契约：`BaseTool`

`app/tools/base.py` 定义了 `BaseTool` 和 `ToolMetadata`。

一个工具最少要声明：

- `name`
- `description`
- `input_schema`
- `source`

可选地还可以声明：

- `display_name`
- `use_cases`
- `requires`
- `outputs`
- `retrieval_controls`

这意味着 OpenSRE 的工具不是普通 Python 函数，而是“自描述能力单元”。

## 统一运行时表示：`RegisteredTool`

`app/tools/registered_tool.py` 把工具统一成一个运行时对象。

它额外补充了几个重要概念：

- `surfaces`：工具出现在哪些界面，通常是 `investigation` 或 `chat`
- `is_available()`：在当前 source 条件下是否允许执行
- `extract_params()`：从 `available_sources` 中提取运行参数
- `cost_tier`、`tags`：供未来调度、优化和解释使用

这是工具系统最关键的抽象之一：  
规划层和执行层不需要知道工具是 class 还是 function，只看 `RegisteredTool`。

## 函数式注册：`@tool`

`tool_decorator.py` 允许以函数方式定义轻量工具。

它的作用是：

- 给普通函数附加注册元数据
- 保持与 `BaseTool` 兼容
- 降低新增小工具的成本

因此，OpenSRE 的工具系统支持两种形态：

- class-based tool
- function-based tool

## 自动发现：`registry.py`

`registry.py` 会扫描 `app/tools/` 下的模块，并自动导入候选工具。

它跳过：

- `base`
- `registry`
- `registered_tool`
- `tool_decorator`
- `utils`
- `investigation_registry`

自动发现机制的价值在于：

- 新增工具不需要手工维护一个大列表
- chat 和 investigation 可共用一个统一注册表

## Surface 机制

工具并不是在所有场景都可见。

OpenSRE 区分了两个主要 surface：

- `investigation`
- `chat`

例如：

- 一些工具只适合自动调查图
- 一些工具可以暴露给交互式 chat

`get_registered_tools(surface)` 可以按 surface 过滤，这使系统能复用同一工具层，但在不同运行面上呈现不同能力集。

## 调查动作注册表：`investigation_registry`

`app/tools/investigation_registry/` 不是重复注册工具，而是给调查分支增加一层“动作视图”。

核心能力有：

- `get_available_actions()`
- `get_prioritized_actions()`
- `get_prioritized_actions_with_reasons()`

### 为什么还要这一层

因为调查流程里的问题不是“有哪些工具”，而是：

- 这一轮应该优先执行哪些动作
- 为什么是这些动作
- 如果没有强匹配，保底动作是什么

这里把 source、关键词和 fallback 行为组织成了独立逻辑，而不是写死在节点里。

## 工具如何参与规划

在 `plan_actions()` 中，系统会：

1. 从 registry 取出所有 investigation tools
2. 根据 `detect_sources()` 识别的 source 做优先级排序
3. 根据关键词进一步筛选
4. 屏蔽已经执行过的动作
5. 按 tool budget 截断
6. 把剩余动作喂给 LLM 做本轮选择

因此，工具层不是只在执行时发挥作用，它在规划阶段就已经是主角。

## 工具如何参与执行

真正执行发生在：

- `app/nodes/investigate/execution/execute_actions.py`

### 执行机制

1. 校验动作是否存在
2. 校验 `is_available(available_sources)`
3. 通过 `extract_params(available_sources)` 组装参数
4. 调用 `run(**kwargs)`
5. 对异常做错误包装
6. 并发执行多个动作
7. 对瞬时错误做指数退避重试

### 为什么这样设计

这样执行层无需理解任何具体第三方系统，只需要依赖工具合同。

## 一个具体例子：`query_grafana_logs`

`app/tools/GrafanaLogsTool/__init__.py` 是一个很典型的工具。

它声明了：

- `name="query_grafana_logs"`
- `source="grafana"`
- `requires=["service_name"]`
- `is_available`
- `extract_params`

### 工作机制

#### 可用性判断

`_query_grafana_logs_available()` 会检查：

- 是否已经验证过连接
- 或是否已经注入了测试 backend

#### 参数抽取

`_query_grafana_logs_extract_params()` 从 `available_sources["grafana"]` 中抽取：

- `service_name`
- `pipeline_name`
- `execution_run_id`
- `time_range_minutes`
- backend 或 credentials

#### 真正执行

工具本身不负责编排，只负责：

- 如果注入了 backend，用 backend 查询
- 否则创建 Grafana client
- 查询 Loki
- 对日志做压缩和错误分类
- 返回标准化结果 dict

这是 OpenSRE 工具设计的标准模式。

## 工具返回值为什么大多是 dict

因为后续的 merge、diagnose、publish 需要的是：

- 易序列化
- 易合并
- 易追加字段
- 对远程运行也友好

dict 不是最强类型安全形式，但在“多工具、多供应商、长链路状态流”场景中是现实而有效的中间表示。

## 工具层的设计价值

可以把 `tools` 理解成 OpenSRE 的“动作总线”：

- 对规划层，它是可选择能力集
- 对执行层，它是统一调用合同
- 对 service 层，它是上层封装出口
- 对报告层，它是证据来源生产者

工具层把“第三方 API 调用”提升成了“调查动作”，这正是 OpenSRE 与普通 SDK 集合的区别。
