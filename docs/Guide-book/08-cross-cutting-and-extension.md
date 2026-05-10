# 认证、护栏、脱敏、Sandbox 与扩展方式

最后一章关注那些不直接属于主业务链，但又决定系统是否可靠、可扩展的能力。

## `auth`：认证与多租户

相关代码：

- `app/auth/auth.py`
- `app/auth/jwt_auth.py`

### 解决的问题

当 OpenSRE 作为 LangGraph 托管运行时存在时，它不是一个单用户玩具，而是多租户系统。

因此需要：

- 验证 JWT
- 抽取组织和用户信息
- 让 threads / assistants / crons 带上 `org_id`

### 工作机制

`auth.py` 使用 LangGraph SDK 的 `Auth()`：

- `@auth.authenticate`
- `@auth.on.threads.*`
- `@auth.on.assistants.*`
- `@auth.on.crons.*`

它的价值在于：

- 安全边界不写在业务节点里
- 组织过滤成为平台层能力

## `guardrails`：提示词与内容护栏

相关代码：

- `app/guardrails/rules.py`
- `app/guardrails/engine.py`
- `app/guardrails/audit.py`

### 解决的问题

调查场景里经常会碰到：

- 密钥
- token
- 敏感配置
- 内部标识符

这些内容不应该被无控制地送进模型或下游输出。

### 工作机制

1. 从 YAML 读取规则。
2. 规则可以是：
   - `redact`
   - `block`
   - `audit`
3. 扫描文本时，匹配 regex 和关键词。
4. 对重叠 span 做合并 redaction。
5. 在需要时直接阻断调用。

Guardrails 会被应用在：

- LLM client
- chat 消息

因此它属于平台级安全机制。

## `masking`：可逆脱敏

相关代码：

- `app/masking/policy.py`
- `app/masking/detectors.py`
- `app/masking/context.py`

### Guardrails 与 masking 的区别

- Guardrails 偏“安全策略”
- Masking 偏“推理期间的可逆占位替换”

### 工作机制

`MaskingContext` 会：

1. 检测待脱敏标识符
2. 生成稳定 placeholder
3. 记录 placeholder -> original 的映射
4. 在推理阶段替换
5. 在报告输出阶段恢复

这很适合事故调查场景，因为：

- 模型不一定需要真实基础设施名字
- 但用户最终看到的结果必须恢复真实标识

## `sandbox`：受限代码执行

相关代码：

- `app/sandbox/runner.py`

### 解决的问题

有些场景需要运行诊断 Python 代码，但不能允许这段代码：

- 访问网络
- 随意写文件
- 任意起 subprocess

### 工作机制

sandbox runner 通过：

- 注入 preamble
- 替换 `socket`
- 限制 `open()` 的写路径
- 阻断 `subprocess`
- 设置超时

把用户代码放进受控子进程里执行。

这不是容器级沙箱，但对很多轻量诊断场景已经足够。

## `analytics` 与 `utils`：平台支撑层

### `analytics`

主要记录：

- CLI 调用
- 评估过程
- 首次运行行为

它帮助项目理解运行方式和使用模式。

### `utils`

这里放的是跨模块共用但不适合挂在核心抽象上的能力，例如：

- Slack / Discord / Telegram 投递
- 配置辅助
- Sentry 初始化
- 截断和序列化工具

## 如何扩展 OpenSRE

这一部分是给贡献者看的最实用内容。

### 增加一个 integration

典型步骤：

1. 定义标准配置对象
2. 加入 registry 元数据
3. 在 catalog 中接入归一化逻辑
4. 如需本地保存，接入 store
5. 实现 verify
6. 在 `detect_sources()` 中让规划层能发现它
7. 再通过 tools + services 真正使用它

### 增加一个 tool

典型步骤：

1. 在 `app/tools/` 下新增工具
2. 通过 `BaseTool` 或 `@tool` 暴露元数据
3. 定义 `is_available()` 和 `extract_params()`
4. 复用现有 service 或新增 service client
5. 加测试

### 增加一个 node

典型步骤：

1. 在 `app/nodes/` 下实现节点
2. 在 `app/pipeline/graph.py` 注册
3. 如需控制流变化，更新 `routing.py`
4. 如需新状态字段，更新 `AgentState` 和 `AgentStateModel`

### 增加一个 service

典型步骤：

1. 先明确这个 service 解决的是“对外访问复用”问题
2. 让输入依赖标准化 config
3. 公开稳定的方法而不是暴露原始底层细节
4. 让多个工具可复用

## 建议的源码阅读路径

如果你要继续深入源码，推荐按下面顺序读：

1. `app/cli/__main__.py`
2. `app/cli/investigation/investigate.py`
3. `app/state/agent_state.py`
4. `app/pipeline/graph.py`
5. `app/nodes/extract_alert/*`
6. `app/nodes/resolve_integrations/node.py`
7. `app/nodes/plan_actions/*`
8. `app/nodes/investigate/*`
9. `app/nodes/root_cause_diagnosis/*`
10. `app/nodes/publish_findings/*`
11. `app/integrations/*`
12. `app/tools/*`
13. `app/services/*`

## 最后总结

OpenSRE 的关键不在某一个目录，而在这些目录之间的分工边界：

- `pipeline/nodes` 决定流程
- `state/types` 决定上下文
- `integrations` 决定如何接外部世界
- `tools` 决定系统能做什么动作
- `services` 决定这些动作如何真正访问外部能力
- `auth/guardrails/masking/sandbox` 决定系统是否可控

理解了这些边界，就基本理解了 OpenSRE 的设计思想。
