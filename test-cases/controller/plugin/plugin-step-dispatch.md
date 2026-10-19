# plugin 步骤的派发

依据[已实现的 Controller plugin 根决策](../../../decisions/controller/plugin/0-plugin-installs-as-node-executions.md)。
证据来自 desktop `apps/ora-controller/tests/workspaces/plugins.rs`，使用假 Cloud gRPC（`test-server`）与假 Node WebSocket；不替代真实 Cloud 投影及跨仓库联调。跨仓库联调由 [cluster `task plugin:acceptance`](https://github.com/ora-space/cluster/blob/main/docs/testing/plugin-step.md) 提供：生产 Controller 对真实 Cloud 与 Node
派发创建路径与维护路径的插件执行，并在接管结果后推进。

## Plugin Step Dispatches Cloud's Input To A Capable Node

### 风险

Controller 自行决定安装内容，或把插件执行发给不支持的旧 Node，得到无法解析的拒绝。

### 前置状态

Cloud 的 operation 快照处于 `plugin` 步骤，带 `plugin_input`。

### 触发

快照中无执行、有未结果执行、有已接管结果、执行整体失败、结果未知；目标 Node 未声明 `plugin_install`；快照无 `plugin_input`。

### 必须成立

- 登记的执行输入与 `plugin_input` 逐字段相同，目标为当前 generation 的 Node。
- 结果已接管后 `AdvanceOperation`，单项失败不影响推进。
- 执行整体失败 `DeferOperation(retry_wait, PLUGIN_EXECUTION_FAILED)`；未知 `DeferOperation(blocked, PLUGIN_RESULT_UNKNOWN)`。
- Node 未声明能力时 `blocked`/`EXTERNAL_FAILURE`，不派发；无 `plugin_input` 时直接推进。
- 不再计划或执行插件 Substrate effect。

### 禁止结果

修改或拼装安装载荷；向无能力的 Node 派发；调用 Substrate 插件 effect。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 快照派发与结果推进 | `plugin_step_preserves_input_and_advances_only_after_terminal_items`：原样登记输入，成功／逐项失败均在结果接管后推进并开放 admission，无 Substrate 插件写入；`interrupted_plugin_execution_defers_the_operation`：整次中断进入 retry_wait；[cluster `task plugin:acceptance`](https://github.com/ora-space/cluster/blob/main/docs/testing/plugin-step.md) 以真实 Cloud 与 Node 走通创建与维护两条路径的派发、接管与推进；无输入、复用未完成快照、已接管快照和持续 Unknown 的直接场景仍缺少 | Partial |
| 能力门槛 | `missing_capability_blocks_without_opening_admission`：握手不含 `plugin_install`，无插件派发、operation blocked、admission 关闭 | Covered |

### 决策依据

Controller plugin 根决策 D1、D2，不变量 1–3。
