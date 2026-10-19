# Agent 运行编排

依据 [IssueRun 根决策](../../../decisions/cloud/issue-run/0-agent-run-in-disposable-isolated-workspace.md)，
事务边界见 [controller-integration D6](../../../decisions/cloud/controller-integration/20260928-agent-run-executions-thread-and-upload-grants.md)。
本文件登记 Agent IssueRun 从评论触发到释放运行 Workspace 的阶段承诺。证据来自 `ora-space/cloud` 的 `integration` 包
（真实 HTTP/gRPC 与 PostgreSQL，部分以 simulator 扮演 Controller/Node）和 `internal/core` 包（白盒 PostgreSQL 测试，经生产
钩子驱动结算、派发与释放）。

跨仓库真实链路证据来自 cluster 的 M2 验收（`task agent:session:acceptance` 与 `task agent:session:browser`，见
[Agent 会话验收](https://github.com/ora-space/cluster/blob/main/docs/testing/agent-session.md)）：真实
Gateway/Cloud/Controller/Node 与 Deno 运行的 echo agent 上，@agent 评论创建运行；Thread 依次出现 `meta`、echo 回复与
`turnEnded` 并进入 `idle`；订阅者收到 SSE `issue_run.thread_appended`；追加的用户轮次变为 `delivered` 并得到回复；结束会话后
`threadState` 为 `ended`，会话结果为 `agent_session_ended/user_ended`；未配置对象存储时交付记为 `skipped`，运行 Workspace
被删除，运行以 `phase = done` 完成。该验收发现并修复了（ora-space/cloud#46）Cloud 把同一轮里 agent 的记录当作回显丢弃的问题。
验收失败只能说明端到端链路断开，不能单独定位到下列某条义务，因此只作为补充引用，不据此标记 `Covered`。

## Agent Target Requires A Project

### 风险

没有 Project 的 Issue 创建出永远无法开始的 `queued` 运行，用户误以为 Agent 在排队。

### 前置状态

Space 有 `active` 的 Space Agent；Issue 未设置 `project_ref`。

### 触发

Task Mode 评论的 `targets` 含该 Agent。

### 必须成立

- 请求以 `409 issue_project_required` 拒绝，评论、IssueRun、Timeline 活动都不写入。
- 设置 `project_ref` 后同样的评论创建 `queued` 运行，运行输入固定了 Agent 的插件 ID 与版本。

### 禁止结果

写入评论但不创建运行；创建运行后再失败。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 无 Project 时整条评论拒绝 | `integration/agent_target_project_gate_test.go::TestAgentTargetMissingProjectAtomicRollback`：无 `project_ref` 时返回 `409 issue_project_required`，评论、运行、交互与 Timeline 活动都为 0；`::TestAgentTargetNonexistentProjectRejected`：不存在的 Project 同样整条拒绝；`::TestAgentTargetResolvesAndSnapshots`：有 Project 时创建运行并固定插件 ID 与版本 | Covered |
| 运行输入固定插件版本 | `integration/agent_target_project_gate_test.go::TestAgentRunSnapshotPinsPluginVersion`：升级 Space 插件到 2.0.0 后，已有运行输入仍为 1.0.0，新运行为 2.0.0；`integration/agent_run_control_test.go::TestSessionWorkIsRegisteredOnTheCloudChosenNode`：登记会话执行时改写 `agentPluginVersion` 的输入以 `CONFLICT` 拒绝。缺口：没有测试在升级后经会话启动扫描生成会话工作项，断言其中的版本仍取运行输入（后者的输入是手写的）。M2 真实链路验收也只断言运行输入固定插件 ID 与版本 | Partial |

### 决策依据

IssueRun D1、D2，不变量 6、7。

## Run Phases Advance Only In Evidence Transactions

### 风险

阶段在证据落库前推进或倒退，导致会话在 Workspace 就绪前下发，或同一阶段的工作被放出两次。

### 前置状态

`queued` 的 Agent 运行，Controller 持有有效租约。

### 触发

依次接管：运行 Workspace `create_workspace` 成功、首条 Thread 事件、会话终态、交付终态、`delete_workspace` 成功。

### 必须成立

- 每次转换与对应证据的接管在同一事务中提交：`provisioning → starting → running → delivering → releasing → done`。
- `starting` 放出恰好一个会话工作项；`delivering` 放出恰好一个未登记的交付工作项。
- Project 忙时运行保持 `queued`/`provisioning`，派发循环稍后重试，不报错。
- `create_workspace` 失败时进入 `releasing`，`status = failed`，`failure_reason = workspace_unavailable`。

### 禁止结果

阶段倒退；同一运行同时存在两个未登记的工作项；会话在 Workspace 就绪前放出；Agent 会话占用 operation。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 完整阶段链 | `integration/agent_run_delivery_test.go::TestFullLifecycleSessionEndToDoneSkipsNoPhase`：从 `starting` 依次经过 `running → delivering → releasing → done`，终态为 `done/completed/ended`、Revision 已登记、恰好一个成功的 delete；`internal/core/agent_run_settle_db_test.go::TestPhase2ASettleReady`：`create_workspace` 成功把 `provisioning` 推进到 `starting`；`integration/agent_run_thread_events_test.go::TestOnlyTheFirstRecordAdvancesRunning`：声明会话、POST、命令登记都不推进，只有首条 Node 记录进入 `running`；`integration/agent_run_session_end_test.go::TestSessionEndedEndsThreadDiscardsQueuedAndReleasesDelivery`：终态接管恰好放出一个未登记的交付工作项。缺口：没有一个测试从 `queued` 经 `provisioning` 连续驱动到 `done`；“`starting` 放出恰好一个会话工作项”只在测试辅助函数里以“存在工作项”检查，没有计数断言；Agent 会话不占用 operation 没有直接断言。M2 真实链路验收以真实 Controller/Node 把一次运行从评论驱动到 `done`（交付 `skipped`） | Partial |
| Project 忙时排队 | 缺口：派发器在 `createRunWorkspace` 返回 `busy` 时保持 `queued`（`internal/core/agent_run_dispatcher.go`），但没有测试以忙 Project 调用派发并断言运行保持 `queued`、operation 结束后再创建运行 Workspace；`seedDispatchScene` 支持 busy 参数但没有测试使用。`TestRunWorkspaceAndPluginStep` 的 busy 只针对删除，不是本义务 | Missing |
| 建 Workspace 失败 | `internal/core/agent_run_settle_db_test.go::TestPhase2ASettleWorkspaceFailure`：`failed` 结果使运行进入 `releasing`、`status = failed`、`failure_reason = workspace_unavailable`，为该运行 Workspace 恰好声明一个 delete，并写一条 `run.failed`；`integration/agent_workspace_failure_hook_test.go::TestWorkspaceFailedHookAndForceEvidenceCommitTogether`：失败钩子与控制面证据同事务提交 | Covered |
| 钩子失败回滚 | `integration/agent_hook_atomicity_test.go::TestSessionEndedHookRollsBackAndReplaysOnce`：注入会话结束钩子错误，收据、执行结果与钩子写入全部回滚，重放后成功且钩子只执行一次；`::TestWorkspaceReadyAndDeletedHooksShareTheOperationTransaction`：就绪与删除钩子失败时准入与删除都不提交，重放后成功一次；`integration/agent_run_boundary_test.go::TestThreadBusinessHookSharesTheTakeoverTransaction`：Thread 接管钩子失败时收据与序号都不写入，重放后成功一次；`integration/agent_run_session_end_test.go::TestSessionEndedHookFailureRollsBackTheWholeTakeover`：生产钩子失败时收据、序号、Thread、轮次状态、交付工作与阶段都不变 | Covered |

### 决策依据

IssueRun D3、D7，不变量 2、5；controller-integration D6。

## Run Workspace Is Released Only After Delivery Settles

### 风险

Revision 未登记就删除运行 Workspace，Agent 的修改丢失；或交付永远失败导致沙盒无限期占用。

### 前置状态

运行处于 `delivering`。

### 触发

交付执行成功、失败、连续失败超过 2 小时、运行 Workspace 的 Node 未知超过 30 分钟；或运行在 `provisioning`/`starting` 被取消。

### 必须成立

- 只有 Revision 已登记、交付按上限放弃、或无会话取消时才创建 `delete_workspace`。
- 交付失败以 30 秒起、10 分钟封顶的退避放出新的交付工作项；放弃时 `deliveryState = failed`。
- `status` 与 `deliveryState` 分别记录：`failed`/`cancelled` 的运行也可以有 Revision。

### 禁止结果

交付未结算就释放；Node 报告“已上传”即视为交付完成；取消立即删除正在运行的会话的 Workspace。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 未结算不释放 | `integration/agent_run_delivery_test.go::TestFailedDeliveryKeepsDeliveringAndReleasesOneBackoffRetry`：交付失败后运行保持 `delivering/running`、没有 delete，恰好放出一个 30 秒退避、尚不可领取的新交付工作项；`internal/core/agent_run_release_db_test.go::TestDeliveryRetryBackoffFollowsD5sNumbers`：退避 30 秒起、翻倍、10 分钟封顶；`integration/agent_run_delivery_test.go::TestRevisionVerificationFailureIsCloudsOwnVerdict`：对象校验不符时由 Cloud 判定失败，不登记 Revision | Covered |
| 放弃上限 | `integration/agent_run_delivery_test.go::TestDeliveryGiveUpReleasesTheRunWithD5StateAndD4Status`：连续失败窗口与 Node 不可达窗口到期都使运行进入 `releasing`、`deliveryState = failed`、`revisionId` 为 null，`status` 取自会话结束原因，恰好一个 delete；`::TestGiveUpStaleDeliveriesPassReleasesOnItsOwnClock`：无 Node 上报时放弃扫描按 Cloud 时钟释放，重复扫描不再写入。缺口：测试把窗口配置为 1ns 或回拨心跳，默认值 2 小时/30 分钟（`internal/config/config.go`）没有测试断言 | Partial |
| 取消是请求 | `internal/core/agent_run_thread_end_db_test.go::TestCancelWithASessionEndsTheThread`：已声明会话（`starting` 的 `pending`、`active`、`idle`）时取消只把 Thread 置为 `ending` 并放出一个 `EndSession{cancelled}`，阶段不变；`::TestCancelWithoutASessionReleasesTheRun`：尚无会话时取消直接进入 `releasing/cancelled`、`deliveryState = skipped` 并声明一个 delete；`::TestCancelDuringProvisioningIsLeftToSettlement`：`provisioning` 的取消留给建 Workspace 结算，不声明 delete；`integration/agent_run_release_giveup_test.go::TestG032_8_BusinessStatusMatrixSurvivesTheGiveUp`（cancelled 子用例）：以 `cancelled` 结束的会话经交付后 `status = cancelled` | Covered |

### 决策依据

IssueRun D4、D5、D6，不变量 3、4。

## Run Workspace Release Gives Up Only When Unreachable

### 风险

运行 Workspace 的删除持续失败时运行永远停在 `releasing`；或者反过来，删除仍在推进时被判为放弃，运行被谎报为已释放。

### 前置状态

运行处于 `releasing`，运行 Workspace 有一个未完成的 `delete_workspace`。

### 触发

删除成功；Node 明确拒绝 quiesce；运行 Workspace 的 Node 状态未知时间未超过或超过不可达窗口；删除成功与放弃扫描竞态。

### 必须成立

- 只有运行 Workspace 仍有存活沙盒、且 Node 状态未知超过 D5 的不可达窗口时才放弃；拒绝 quiesce、失败次数都不触发放弃。
- 放弃时 `delete_workspace` 以 `node_unavailable` 终结，运行进入 `done`，`status` 保持不变，`failure_reason = workspace_unavailable`。
- 运行 Workspace 行保留且仍关联该运行，不写成功形态的删除结果。

### 禁止结果

把放弃写成删除成功；`done` 之后再次声明删除或改写运行；放弃后阶段倒退。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 只以不可达为条件 | `integration/agent_run_release_giveup_test.go::TestG032_2_UnreachableBelowTheWindowStaysReleasing`：窗口内保持 `releasing`；`::TestG032_4_RefusedQuiesceNeverTriggersGiveUpEvenPastTheWindow`：拒绝 quiesce 越过窗口也不放弃 | Covered |
| 放弃终态与残留 | `::TestG032_3_UnreachablePastTheWindowSettlesDoneWithoutClaimingDeletion`：越过窗口进入 `done`、`failure_reason = workspace_unavailable`，Workspace 未软删除，删除以失败码终结；`::TestG032_8_BusinessStatusMatrixSurvivesTheGiveUp`：`status` 保持交付时写入的值 | Covered |
| 竞态与终态稳定 | `::TestG032_5_SuccessfulDeleteWinsTheRaceAndThePassIsANoOp`、`::TestG032_6_GiveUpWinsTheRaceAndAStaleDeleteSettlementIsRefused`、`::TestG032_7_ADoneRunIsNeverRedeclaredOrRewritten`、`::TestG032_9_NoBackwardLifecycleAfterGiveUp`；`::TestG032_1_SuccessfulDeleteIsUnchangedByTheGiveUpPass`：正常删除不受扫描影响 | Covered |

### 决策依据

IssueRun D8，不变量 8。
