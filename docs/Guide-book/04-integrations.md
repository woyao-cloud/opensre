# `integrations` 模块：如何接第三方系统

这一章专门回答：  
OpenSRE 是如何把第三方系统接进来的？

## 为什么需要 `integrations`

OpenSRE 面对的不是一个统一平台，而是一组异构外部系统：

- Grafana
- Datadog
- CloudWatch
- GitHub / GitLab / Bitbucket
- RabbitMQ / Kafka / PostgreSQL / MongoDB
- Vercel / ArgoCD / Airflow
- Slack / Discord / Telegram

这些系统的差异体现在：

- 配置字段不同
- 凭证形式不同
- API 风格不同
- 是否支持验证不同
- 是否允许多实例不同

如果直接让节点或工具自己处理这些差异，代码会迅速失控。  
所以 OpenSRE 把“第三方接入标准化”单独做成了一层。

## 模块构成

`app/integrations/` 主要由几部分组成：

- `config_models.py`：严格配置模型
- `registry.py`：集成元数据注册表
- `catalog.py` 与 `_catalog_impl.py`：解析和归一化入口
- `store.py`：本地集成存储
- `verify.py` 与 `_verification_adapters.py`：验证入口
- `selectors.py`：多实例选择
- 各个具体集成模块：如 `airflow.py`、`betterstack.py`、`mysql.py`

## 集成层的五个职责

### 1. 配置标准化

每个第三方系统都要先变成内部统一配置对象。

例如在 `config_models.py` 中定义：

- `GrafanaIntegrationConfig`
- `DatadogIntegrationConfig`
- `AWSIntegrationConfig`
- `BetterStackIntegrationConfig`
- `RabbitMQIntegrationConfig`

这些模型解决的问题：

- 给 env / store / remote 配置统一字段语义
- 在系统启动或调查时尽早暴露错误
- 让下游 service client 不必重复做字段清洗

### 2. 来源合并

OpenSRE 的集成配置可能来自：

- 远端组织配置
- 本地 `~/.config/opensre/integrations.json`
- 环境变量

`resolve_integrations` 节点最终会依赖：

- `catalog.py`
- `_catalog_impl.py`

把这些来源合并成统一的 `resolved_integrations`。

### 3. 运行时分类

`classify_integrations()` 的核心作用，是把“原始集成记录”转成“运行时扁平配置”。

它的输出特征是：

- `resolved[service]`：默认实例的扁平配置
- `resolved["_all_<service>_instances"]`：多实例列表
- `resolved["_all"]`：全部原始 active records

这是一种兼顾“新能力”和“向后兼容”的设计：

- 老代码仍然可以直接读取 `resolved["grafana"]`
- 新代码可以通过 `selectors.py` 读多实例

### 4. 可验证性

`verify.py` 暴露统一入口 `verify_integrations()`。

它会：

1. 解析 effective integrations
2. 根据 `registry.py` 里的 `VERIFIER_REGISTRY`
3. 调用每个服务对应的 verifier
4. 返回标准结果列表

验证结果会统一落到：

- `passed`
- `failed`
- `missing`

这套机制的价值在于：  
“能不能连接某个第三方系统”是一个平台级概念，不应该散落到每个命令里各写一份。

### 5. 多实例选择

`selectors.py` 解决的问题是：

- 一个服务可能有多个实例
- 工具或节点需要按名称或 tags 选择

提供的能力包括：

- `get_instances()`
- `get_default_instance()`
- `get_instance_by_name()`
- `get_instances_by_tag()`
- `select_instance()`

这使得 OpenSRE 可以从“单实例配置系统”自然演进到“多实例运行时”。

## 本地集成存储：`store.py`

这是集成层最像“基础设施代码”的部分之一。

### 它解决的问题

- 本地需要持久化集成配置
- 格式需要演进
- 多进程写入要安全
- 旧格式要自动迁移

### 工作机制

`store.py` 提供：

- `load_integrations()`
- `get_integration()`
- `upsert_integration()`
- `remove_integration()`
- `get_instances()`
- `upsert_instance()`

并通过以下手段保证稳定性：

- 版本化存储格式
- v1 -> v2 自动迁移
- `FileLock` 文件锁
- 临时文件 + `os.replace` 原子写入

这说明集成层不是“拿个 JSON 文件凑合”，而是认真当成配置系统设计的。

## 一个典型模式：配置型集成

以 `app/integrations/airflow.py` 为例。

### 它做什么

- 定义 `AirflowConfig`
- 处理 env 到 config 的转换
- 提供 lightweight validate
- 暴露几个直接查询函数，如 DAG runs、task instances

### 设计特点

- 配置和查询逻辑写在同一个小模块里
- 使用 `StrictConfigModel` 做严格归一化
- 对上层返回调查友好的 dict，而不是 SDK 原始结构

这类集成适合：

- API 相对简单
- 功能边界比较清楚
- 还没抽到独立 service 包也能保持可维护

## 另一个典型模式：查询型集成

以 `app/integrations/betterstack.py` 为例。

### 它做什么

- 构建 Better Stack SQL API 配置
- 定义可用性判断
- 抽取工具调用参数
- 提供验证逻辑
- 负责安全地构造 SQL 查询

### 设计特点

- integration 模块不仅存配置，还直接参与 tool 可用性判断
- 对 source 名称做正则白名单，避免 SQL 注入
- 工具通过 `extract_params()` 把 alert-derived source hint 映射成具体参数

这说明 integration 层并不只是“被动配置”，它也参与运行时能力裁剪。

## 集成层如何与调查主链协作

从执行链看，集成层的位置如下：

1. `resolve_integrations` 先把远端、本地、环境变量的配置收敛成 `resolved_integrations`
2. `detect_sources()` 再结合告警内容，从 `resolved_integrations` 推导出 `available_sources`
3. 工具层的 `is_available()` 和 `extract_params()` 决定某个工具是否能调用以及如何调用
4. service client 再拿到标准化配置真正访问第三方系统

因此可以把集成层理解成：

> 把“第三方世界的多样性”变成“OpenSRE 运行时可消费的统一配置和 source 视图”。

## 第三方集成在 OpenSRE 中的标准接入路径

一个新集成通常要经过这几个步骤：

1. 在 `config_models.py` 或具体 integration 模块中定义标准配置对象
2. 在 `registry.py` 中注册 service 元数据和 verifier
3. 在 `_catalog_impl.py` 中加入分类和 env 解析逻辑
4. 如需本地持久化，接入 `store.py`
5. 提供验证逻辑给 `verify.py`
6. 通过 `detect_sources()` 把它暴露给规划层
7. 通过 tool 和 service 层真正参与调查

这条路径也说明：  
集成层只是“接进来”，真正“用起来”还要靠后续的 tools 和 services。
