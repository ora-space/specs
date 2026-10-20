---
status: implemented
---

# 插件安装与移除由 Node 执行，新建 Workspace 自动补装 Space 插件

> 后续决策：[Agent 配置元数据](20261017-agent-configuration-metadata.md) 已批准扩展个人模型
> 路线的声明关联与安装结果校验，尚待实现；维护互斥和安装租约不变。

插件的执行面从“`plugin_ensure`/`plugin_delete` Substrate effect”改为“Controller 派发给 Workspace Node 的
执行”：载荷不变（仍由 Cloud 从目录快照自包含拼装、SHA-256 必选），但证据来自 Node 执行结果，经
`ExecutionService` 的派发前登记与原子接管进入 Cloud。新建或启动的 Workspace 在 `plugin` 步骤安装 Space
期望的全部插件，补上根决策中留给后续 phase 的“新建 workspace 自动补装”。agent 类插件收敛为 `installed`
时同时维护 Space 的 Agent 行。

本 ADR 处于 `implemented`，继承 [插件市场根决策](0-plugin-marketplace.md)。步骤表见
[operation 后续决策](../operation/20260928-plugin-step-and-run-workspace-release.md)；Node 上的安装语义见
[Node plugin-runtime 根决策](../../node/plugin-runtime/0-node-installs-plugins-into-workspace-home.md)；
Agent 行见 [IssueRun 根决策](../issue-run/0-agent-run-in-disposable-isolated-workspace.md) D1。

## 继承与修改

| 前序约定 | 本文处理 |
|---|---|
| D1：`space_plugins` 选择状态与 `workspace_plugin_instances` 执行实例两级分离，聚合规则 | 继承，不变 |
| D2：fan-out 到 live workspace，受 `one_project_operation` 串行 | 继承，不变 |
| D3：目录同步语义 | 继承，不变 |
| D4：`plugin_ensure` effect 载荷与 `installed=true` 证据 | 修改：载荷成为 Node 执行输入的元素，证据为 Node 执行的逐插件结果（D1） |
| v1 边界：新建 workspace 的自动补装为后续 phase | 修改：由 `plugin` 步骤补装（D2） |
| [管理员维护决策](20260927-admin-selection-and-idle-maintenance.md) D4：未接线执行器/新实例不补装 | 修改：执行结果由 Node 提供（D1），create/start 的 plugin 步骤补装当前期望（D2）；管理员门控、持久等待与维护互斥继承不变 |
| v1 边界：真实 Node 执行器接线为后续工作，simulator 扮演 Node | 修改：由 desktop Node 执行；simulator 保留用于 Cloud 测试 |
| 不变量 4：sha256 校验必选 | 继承，由 Node 在安装前校验 |
| desktop 本期零改动 | 修改：desktop Node 增加安装能力 |

## 决策

### D1：执行输入与结果

- 输入 `InstallPlugins{plugins: [{pluginId, version, universal{url, sha256} | targets[{target, url, sha256}]}]}`
  与 `RemovePlugins{plugins: [{pluginId, version}]}`，每项字段与原 effect 载荷逐一对应。
- 结果逐项：`installed{version}` 必须等于计划版本，否则 Cloud 以 `invalid_plugin_evidence` 拒绝整个结果；
  `removed`；`failed{reason}`，`reason` 是有界错误码（`download_failed`、`checksum_mismatch`、
  `no_matching_target`、`invalid_package`、`install_failed`、`plugin_in_use`），写入实例的 `install_error`。
  `plugin_in_use` 表示有运行中的 Agent 会话正在使用该插件，Node 既不替换也不移除它。
- 结果必须与计划逐项对应：项数相同、插件不重复、`removed` 只出现在移除执行中、失败码在白名单内；任何一项
  不符都以 `invalid_plugin_evidence` 拒绝整个结果。
- 同一 operation 的插件执行在失败后重试时是新执行，旧执行的结果保留。

### D2：新建与启动的 Workspace 补装期望集合

`plugin` 步骤的期望集合是进入步骤时 Space 中 `desired_state = installed` 的插件。为每个插件 upsert
该 Workspace 的实例行为 `installing` 后执行；结果写回后按根决策 D1 重算 Space 聚合。执行期间用户新装的
插件不在本次集合中，由它自己的 install_plugin fan-out 覆盖。

### D3：agent 类插件维护 Space Agent 行

聚合结果使一个 `kind = agent` 的插件变为 `installed` 时，同一事务创建或恢复其 `space_agents` 行；
`desired_state` 变为 `removed` 时把该行置为 `retired`。Agent 行的存在只依赖 Space 级选择与聚合，不依赖
某个 Workspace 的实例，因此运行 Workspace 上个别插件安装失败不会让 Agent 从 @ 列表消失——该运行自己
以 `agent_plugin_unavailable` 失败。

## 不变量

1. 插件执行的载荷由 Cloud 从目录快照拼装，Node 不自行解析市场。
2. SHA-256 校验先于任何安装落盘。
3. 插件实例的状态只由 Node 执行结果的接管写入。
4. Space Agent 行与 agent 类插件的 Space 级选择一致。

## 为什么不是这些替代方案

| 方案 | 取舍 |
|---|---|
| 保留 Substrate effect 并在 Sandbox Server 实现 | 契约不变；但沙盒外的服务要写 Node home 并理解插件包格式（见 operation 后续决策） |
| Node 按 Space 选择自行同步 | 少一次派发；与根决策“Node 不自己决定装什么”冲突 |

## 本决策未解决的问题

- pack 展开安装；插件配置与密钥下发；运行中插件升级。

## 落地

Cloud 的接管写回与 Agent 行维护以 PostgreSQL 集成测试覆盖；desktop Node 执行安装与移除；simulator 改为模拟
Node 执行结果，保留用于 Cloud 测试。
