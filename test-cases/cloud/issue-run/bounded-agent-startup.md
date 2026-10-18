# Agent 启动与创建未知结果的有界收尾

依据[有界启动 ADR](../../../decisions/cloud/issue-run/20261012-bounded-agent-startup-and-undispatched-withdrawal.md)。
第一期 D1–D4 与第二期 D5 分别验收；下列代码路径相对 cloud 仓库。局部回归不替代真实 Controller/Node 故障演练。

代表性[候选部署证据](https://github.com/ora-space/circus/issues/36#issuecomment-6162711653)固定 Cloud `116e932`、Controller/Node `cf166325`、cluster `aef10f3`，脚本快照归档在同一运行 issue。部署使用真实个人模型、普通/恢复 2m/6m 与确认 30s 配置，暂停已登记 Node 并重启 Cloud；默认确认 5m 由配置回归覆盖。正常多轮、Revision 保存/恢复、正常资源释放也在该候选上验证。证据限单机指定场景，不代表领取延后、登记响应丢失或完整故障矩阵。

## Startup Deadlines Must Remain Fixed Across Retries And Restarts

- 风险：声明、输入或配置变更延长已接受 Run 的等待。
- 前置：普通或恢复 Run 已 ready，工作项可能延后；另一副本使用不同配置。
- 触发：重复扫描、输入、延后领取及服务重启。
- 必须成立：普通/恢复/结束确认默认 2/6/5 分钟，数据库绝对截止持久冻结，非法显式配置拒绝；恢复不叠加普通窗口。
- 禁止结果：重放续期、从可变 updated_at 推断新起点或默默接受零/负时长。
- 证据：`internal/core/agent_run_startup_budget_db_test.go::TestStartupDeadlineIsFrozenAcrossReplicasAndRetries`；`internal/config/agent_startup_test.go`；`integration/agent_run_resume_test.go::TestFollowUpRunResumesTheIssueLatestRevision`；候选部署在确认窗口中重启 Cloud，前后绝对截止不变。持久窗口与配置义务 Covered；真实调度暂时失败/延后联合矩阵 Partial。
- 决策依据：D1、D2。

## Withdrawal And Registration Must Have One Transactional Owner

- 风险：已撤回的旧 claim 重新登记，或注册结果未知却清理 Workspace。
- 前置：未登记启动、用户结束/取消、明确缺能力或截止与派发竞争。
- 触发：撤回扫描与 RecordDispatch 并发，旧身份重放。
- 必须成立：同事务判断登记；撤回、终态、queued 输入与 owned delete 原子提交；明确拒绝和暂时不可达分别处理。
- 禁止结果：撤回后新登记、清理已登记未知执行、能力过期被当确定性拒绝。
- 证据：`internal/core/agent_run_startup_settlement_db_test.go` 的 `TestStartupEndingAndDispatchRaceKeepsOneOwner`、`TestUndispatchedStartupWithdrawalRejectsLateDispatchAndRepeats`、`TestStaleNodeCannotWithdrawDeclaredStartup`。数据库/控制入口义务 Covered；真实断连及登记响应丢失的 Controller 整链 Partial。
- 决策依据：D1、D3。

## Confirmation Expiry Must Preserve Unknown Execution Responsibility

- 风险：永久 ending，或业务关闭时丢弃残留资源、伪造物理终止。
- 前置：已登记而首次启动期间进入 ending，确认截止到达仍无真实终态。
- 触发：截止扫描、重复处理或残留账本写入失败。
- 必须成立：failed/done/ended、失败交付与独立账本同事务提交；资源和原执行保留，新输入/派发/模型授权关闭；普通删除/接管拒绝。Node 重连可恢复 ready 事实，但未清除残留继续关闭 admission。
- 禁止结果：假 Node 结果/收据/Revision，账本失败但 Run 已 done，因 done 解除保护。
- 证据：`TestUnknownStartupEndsWithResidualWithoutFabricatingNodeResult`、`TestStartupResidualFailureRollsBackBusinessTerminalAndProtection`、`TestNodeReconnectPreservesStartupResidualAdmissionFence`（`internal/core/agent_run_startup_budget_db_test.go`）。数据库、普通清理及重连受理保护 Covered。候选部署直接验证业务终态时无 Node 结果/Revision、保留容器/卷、拒绝输入/Project 删除，以及真实 model-gateway 创建/续期/旧 bearer 请求均拒绝；单机配置窗口场景 Covered，完整部署故障组合仍 Partial。
- 决策依据：D3、D4；D8 旧删除残留状态保持。

## Late Facts Must Be Receipted Without Resurrecting An Abandoned Run

- 风险：晚到首记录/终态被丢弃，或再次进入 running/delivering。
- 前置：确认超时已业务 done，原 Node 执行与序号仍存在。
- 触发：HTTP/gRPC 真实记录、终态与重复序号接管。
- 必须成立：按原身份保存事实并 ACK；业务终态不变，不新建会话、交付或普通清理。
- 禁止结果：修改失败结果、恢复输入、把迟到终态当成资源已删除。
- 证据：`integration/agent_run_startup_settlement_test.go::TestStartupAbandonmentReceiptsLateFactsWithoutResurrectingRun`。HTTP/gRPC/PG 接管义务 Covered。候选部署验证恢复 Node 后接管真实 terminal、Run 版本/终态不变、admission 保持关闭，Node completed 且持久 outbox 排空（Controller ACK）；该终态路径 Covered。部署中的迟到首记录、多种历史记录及 Controller 跨重启组合仍 Partial。
- 决策依据：D4。

## Startup Migration Must Preserve Existing Outcomes And Freeze Legacy Windows Once

- 风险：升级改写历史完成结果，或重复迁移/观察不断刷新截止。
- 前置：0034 前已有运行，包含未知执行与旧 D8 残留。
- 触发：新库、升级、重复迁移及第一次策略扫描。
- 必须成立：旧结果不改；可靠成功创建证据锚定 ready，无证据则明确首次观察，不猜测精确历史时间；后续不续期。
- 禁止结果：改旧 SQL、历史 completed/cancelled 改 failed、每次部署重新计时。
- 证据：`integration/agent_startup_migration_test.go::TestMigration0034StartupWindowsPreserveExistingRunOnUpgrade` 与固定窗口回归。新库/升级/重复迁移 Covered；完整存量与旧 D8 组合矩阵 Partial。
- 决策依据：D2、D4 兼容边界。

## Blocked Creation Abandonment Must Release The Project Slot Without Losing Residual Protection

- 风险：未知创建无限占槽，或放弃后迟到 ensure 复活可写实例。
- 前置：运行 Workspace 创建在 sandbox/clone/plugin 阶段首次 blocked，可能尚无 Node；另有同 Project 的独立 Workspace。
- 触发：固定创建截止、取消/成功竞争、旧许可、迟到事实和升级回填。
- 必须成立：默认可配置 30m 不续期；原 operation 失败、Run failed/done、槽释放与账本原子；残留资源受保护，迟到证据只补原身份。
- 禁止结果：以无 Agent 登记推断没有创建副作用、重新领取业务终态 operation、残留目标复用/普通删除或伪造物理完成。
- 证据：
  - Cloud：`internal/core/agent_run_workspace_creation_db_test.go` 的 `TestBlockedCreationFreezesDeadlineInDeferTransaction`、`TestBlockedCreationDeadlineIsNotRenewedByRetryRestartOrConfiguration`、`TestLegacyBlockedCreationAnchorsOnceAsFirstObserved`、`TestBlockedCreationExpirySettlesReleasesSlotAndProtectsResidual`、`TestBlockedCreationLedgerFailureRollsBackTerminalAndSlotRelease`、`TestBlockedCreationRealSettlementAndCancelDoNotResurrectOrReorder`、`TestBlockedCreationAbandonmentIsIsolatedPerTenant`；迟到事实与新派发拒绝见 `internal/core/agent_run_provisioning_fact_db_test.go` 与 `integration/agent_run_provisioning_fact_test.go::TestRecordProvisioningFactReceiptsLateFactsOverGRPCWithoutResurrectingRun`；升级保留见 `integration/agent_startup_migration_test.go::TestMigration0038WorkspaceBlockedDeadlineAppliesFreshAndUpgrades`。数据库事务、截止冻结与不续期、存量首次观测、槽释放与账本原子性、租户隔离、Cloud 侧 gRPC 迟到事实接管及新库/升级/重复迁移 Covered。
  - Controller 适配器接入 Covered：[desktop `apps/ora-controller/tests/workspaces.rs`](https://github.com/ora-space/desktop/blob/6ca9610f23e065152b1048c171780dc3fb8adb81/apps/ora-controller/tests/workspaces.rs) 的 `a_creation_abandoned_mid_flight_takes_the_late_sandbox_fact_over` 验证 effect 获许可后创建被放弃，Controller 将 `stale_operation` 对应的同一观测改经 `RecordProvisioningFact` 补入原残留账本，operation 保持 failed、无 advance 且 admission 不重开；`a_creation_closed_for_another_reason_refuses_the_late_fact` 验证非放弃关闭拒绝补记、不伪造残留事实。两项测试运行生产 Controller 协调路径，Cloud、Substrate 和 Node 使用测试替身，证明接入义务，不证明真实部署故障链路。
  - 已执行的真实部署义务 Covered：[固定候选证据](https://github.com/ora-space/circus/issues/115#issuecomment-6253280597)与 [cluster 演练入口](https://github.com/ora-space/cluster/blob/df6a2f124adc4fcf9ccbcadece436ba596c3334f/scripts/provisioning_drill.py) 覆盖 sandbox 拒绝、clone/plugin 命令丢包后真实 Node 报告未知、两个不同配置 Cloud 副本及重启不续期、到期一次结算和独立残留保护、同 Project 独立 Workspace 的真实推进；`late-ensure` 通过真实 Controller 接管跨截止的 Substrate 成功，部署上的同/新 submission 重放均 ACK 且事实数和业务版本不变，已关闭创建的新 effect 许可与规划被拒绝。
  - 成功先到与截止先到的代表性顺序 Covered；取消意图由 PostgreSQL 布置，只证明原意图与截止不被改写，不证明公开取消 API。单机候选未穷尽并发交错、跨部署及完整升级组合，这些矩阵保持 Partial；物理终止、自动回收与 late ensure fencing 由后续回收决策单独验收。
- 决策依据：D5。
