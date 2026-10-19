# Agent 运行执行、Thread 与上传授权的 gRPC 契约

依据[已批准的 controller-integration 后续决策](../../../decisions/cloud/controller-integration/20260928-agent-run-executions-thread-and-upload-grants.md)。
proto 已在 `ora-space/cloud` 合并（cloud#35）；服务端证据来自 [cloud#41](https://github.com/ora-space/cloud/pull/41)
原分支后续提交中经真实 gRPC、PostgreSQL、RustFS 的集成测试。以下覆盖控制面；生产 Rust Node/Controller 与 B 业务投影仍需各自验收。Revision 草案状态保留，实际实现差异见
[历史实现与验收记录](https://github.com/ora-space/specs/blob/41f9dd1910feb071e055685a1172a4dd488cc5ec/implementation/agent-control-plane-a.md)。
该固定快照仅保存原证据，不替代下述验证义务与覆盖状态。

## Session And Delivery Work Is Registered On Its Cloud-Chosen Node

### 风险

Controller 把会话或交付派发到非运行 Workspace 当前 generation 的 Node，或以不同输入重复登记。

### 前置状态

运行放出了会话或交付工作项。

### 触发

`ClaimWork` 后以正确/错误的 `node_id`、相同/不同的输入调用 `RecordDispatch`；重复调用。

### 必须成立

- `WorkItem.target` 给出运行 Workspace 当前 generation 的 Node；`RecordDispatch` 的 `node_id` 与输入必须与工作项一致。
- 同一执行同输入重复登记是幂等的；不同输入 `CONFLICT`。
- 登记后该工作项不再由 `ClaimWork` 返回；`GetDispatch`、`ListPendingDispatches` 覆盖新种类。

### 禁止结果

登记到其他 Node；一个工作项对应两个执行。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 目标与输入校验 | `integration/agent_run_control_test.go` 的 `TestSessionWorkIsRegisteredOnTheCloudChosenNode`；`agent_run_boundary_test.go` 的 `TestRunTargetIncludesTheCurrentSandbox`、`TestExecutionIdentityCannotCrossRegistries`：错误 Node、sandbox、输入和跨表身份冲突被拒绝 | Covered |
| 恢复读取 | `TestSessionWorkIsRegisteredOnTheCloudChosenNode` 覆盖会话 pending；Revision 测试公共 `registeredDelivery` 对 pending 与 `GetDispatch` 返回的完整输入作 protobuf 等值比较 | Covered |
| 运行独占许可 | `TestRunExecutionPermitRetainsScopeAndChecksCurrentPermission`：初始化后保留运行 maintenance epoch，不占 Project operation；旧 epoch、权限撤销拒绝 | Covered |
| 触发者身份 | `TestRunWorkspaceUsesTheTriggerActorAndRequiresEvidence`：Issue 作者与触发者不同，Workspace creator/operation actor 来自 run.enqueued；缺失证据拒绝，Project durable owner 保留 | Covered |

### 决策依据

controller-integration D1、D6，不变量 2。

## Thread Events Are Acknowledged Only After Ordered Takeover

### 风险

Controller 在 Cloud 提交前确认事件，Node 删除后 Cloud 回滚，Thread 永久缺段；或终态先于最后的 Thread 事件落库。

### 前置状态

会话执行已登记。

### 触发

按序、重放、有缺口的 `TakeOverThreadEvents`；在前序 Thread 事件未全部接管时接管会话终态。

### 必须成立

- 批次原子：全部写入或全部不写；返回已接管的最大序号。
- 终态事件只在其前面全部序号已接管后才接受，否则 `CONFLICT`。
- 业务钩子只对首次接管的事件调用一次。

### 禁止结果

部分批次；终态早于 Thread 事件；重放触发第二次钩子。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 批次原子与顺序 | `TestThreadEventsAreTakenOverInOrder`：含合法前缀及后续缺口的批次全部回滚，补齐后成功 | Covered |
| 终态顺序 | `TestThreadEventsAreTakenOverInOrder`：缺少前序记录时终态拒绝；`TestThreadBusinessHookSharesTheTakeoverTransaction`：终态后重放旧记录幂等，第二个新终态拒绝 | Covered |
| 业务与收据同事务 | `TestThreadBusinessHookSharesTheTakeoverTransaction`：钩子经调用方 sql.Tx 写业务证据，失败后业务行、收据和水位一起回滚；重放不再次调用 | Covered |

### 决策依据

controller-integration D2、D6，不变量 3、4。

## Thread Commands Survive Controller Restarts And Upload Grants Are Never Stored

### 风险

用户消息在 Controller 重启时丢失或重复执行；预签名 URL 落入数据库或日志。

### 前置状态

Cloud 有未投递的 Thread 命令；运行处于 `delivering`，交付执行已登记。

### 触发

`ClaimThreadCommands` 后不登记投递即重启、再次领取；`RecordThreadCommandDelivered` 重复提交；`GrantRevisionUpload` 对进行中与已结束的交付。

### 必须成立

- 未登记投递的命令在重新领取时再次返回；登记后不再返回；重复登记幂等。
- 会话执行尚未登记时，命令留在 Cloud、不返回。
- 授权只对尚无结果的交付签发，对象键等于交付输入中的键；Cloud 不留下任何授权记录。

### 禁止结果

命令在登记前消失；授权出现在任何表或日志中；对已结束交付签发授权。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 命令至少投递一次 | `TestThreadCommandsAndUploadGrantsAreNotLostOrStored`：未登记会话时保留；领取后重领、重复登记和投递后为空 | Covered |
| 命令与模拟器恢复 | `TestThreadCommandsPreserveOrderWithinOneBusinessTransaction` 保持同事务顺序；`TestSimulatorRelaysEchoTurnsAcrossControllerRestart` 重开磁盘日志后追加并结束同一会话；`TestSimulatorServicesEndCommandsWhileRunDeletionIsQueued` 防止 quiesce 饿死命令。尚无真实 Controller/Node 恢复验收 | Partial |
| 授权不持久化 | `TestThreadCommandsAndUploadGrantsAreNotLostOrStored` 扫描数据库，终态拒绝授权；`TestSimulatorDeliversRevisionToS3AcrossRestart` 真实上传/刷新后扫描替身磁盘无签名/凭据；sandbox PUT 不挂载 root 凭据。生产 Rust Controller/Node 日志仍未验收 | Partial |
| 前版升级 | `integration/agent_control_upgrade_test.go` 的 `TestAgentControlUpgradePreservesPendingCommandOrder`：从 0024 升级并重复迁移，既有次序保留，新序号连续 | Covered |
| 成功交付等待 Cloud 验证 | `TestRevisionS3VerificationAndAtomicTakeover` 真实 RustFS changed/unchanged、缺失/大小/摘要不符、hook/提交回滚、原始证据保留及重放；`TestRevisionNetworkFailureRemainsReplayable` 网络故障无 ACK；`TestRevisionLeaseAndStateAreRecheckedAfterExternalVerification` 外部 I/O 后重新围栏 | Covered |

### 决策依据

controller-integration D3、D4，不变量 5、6。
