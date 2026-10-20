# Repository Binding 与快照同步

依据 [同步 ADR](../../../decisions/cloud/project/20261016-repository-binding-and-snapshot-sync.md)（`implemented`）。下文的 D 编号与
不变量编号都指这篇 ADR；“契约”指
[代码中心契约决策](../../../decisions/cloud/git-server/20261016-project-repository-as-code-hub-and-ref-contract.md)。

Binding、当前快照与 Operation 的最小存储，以及删除时的取消与回滚恢复，由底座实现并登记在
[代码中心契约用例](../git-server/project-repository-code-hub-contract.md)（`Repository Operation Storage Fixes States And Allows One Unfinished Sync`、
`Project Deletion Cancels Queued Repository Operations And An Aborted Deletion Restores The Owed Sync`）。本文登记外部线在其上增加的义务。
同步 worker、租约与执行器接入已随 cloud#76（`ceb39bc`）合入 cloud 的 main，下列义务按该提交的直接测试登记；
没有直接测试的义务保持 **Missing**。

## A Sync Operation Runs At Most Once And Settles Under Its Lease Generation

- 风险：同一个 Operation ID 先后属于两次执行，失联的旧执行器借新租约写入；代次过期的 worker 写入结果；同步停在中间状态。
- 前置状态：`pending` 的同步；可以杀掉的 worker。
- 触发：`leased` 中租约过期；`running` 中杀掉 worker；旧 worker 在新代次后写结果。
- 必须成立：状态变化都以 `(id, leaseGeneration)` 条件更新；授权只在 `leased → running` 的事务内签发；`leased` 过期回到 `pending`，`running` 过期以 `failed(lease_lost)` 结束；之后用旧 ID 访问服务仓库监听被拒绝。
- 禁止结果：同一 Operation 第二次进入 `running`；代次不符的写入生效；`running` 中的同步被取消。
- 验证义务与证据：**Covered** — `integration/external_git_test.go` 的 `TestRepositorySyncLeaseFencingReplayAndResyncAreAtomic`、
  `TestRepositorySyncConcurrentLeasesAndIdempotentTrigger`；`integration/git_operation_access_test.go` 的
  `TestGitOperationAccessFollowsOperationAndProjectLifecycle`。
- 决策依据：D2，不变量 3、4、11。

## Expected Tracking Oid Is Read Just Before Running

- 风险：expected old 取自创建时或当前快照记录，排队期间过时，之后每次同步都 CAS 失败；跟踪 ref 被旧执行器推进后无法恢复。
- 前置状态：建 Project 时由底座建立的同步排队等待；预置工具改写跟踪 ref。
- 触发：worker 领取并进入 `running`；跟踪 ref 在读取之后被推进。
- 必须成立：`expectedTrackingOid` 在 `leased → running`、签发授权的同一事务里写入，值来自事务外的 ref 查询；`stale info` 时以 `superseded` 结束并在同一事务里自动重放一次，重放重新读取后成功。
- 禁止结果：在创建时或从当前快照记录取值；`superseded` 改变当前快照；`hook declined` 被判为 `superseded`。
- 验证义务与证据：**Partial** — `internal/repoworker/operation_test.go` 的
  `TestSyncReadsTheTrackingRefBeforeItStartsTheOperation`、`TestSyncReturnsTheLeaseWhenTheTrackingRefIsUnreadable`；
  `integration/external_git_test.go` 的 `TestRepositorySyncLeaseFencingReplayAndResyncAreAtomic`。已覆盖：ref 在事务外读取后才开始
  任务，代次过期或重复开始被拒。仍缺：未断言值在 `leased → running` 的同一事务落库；没有端到端用例证明 `superseded` 后重放会
  重新读取跟踪 ref 并成功。
- 决策依据：D2，不变量 6、12。

## The Current Snapshot Advances Only After Cloud Verifies The Reported Refs

- 风险：执行器的报告未经核验就推进当前快照；核验网络失败被当作不一致。
- 前置状态：执行器报告 `{commitOid, snapshotRef, trackingRef, forced}`。
- 触发：两个 ref 都指向报告的提交；任一不一致；ref 查询网络失败；外部 force-push 后同步；外部分支被删除。
- 必须成立：两个 ref 都一致才在同一事务里置 `succeeded`、推进当前快照并记录 `read` 能力；不一致以 `failed(verification_mismatch)` 结束；网络失败在租约内重试；force-push 后 `forced = true`、旧快照 ref 保留；分支删除以 `failed(upstream_branch_missing)` 结束。
- 禁止结果：未核验就推进；失败、取消、`superseded` 改变当前快照；任何已有快照 ref 被删除或改写。
- 验证义务与证据：**Partial** — `internal/repoworker/operation_test.go` 的
  `TestSyncSucceedsOnlyAfterBothRefsNameTheReportedCommit`、`TestSyncVerificationMismatches`、
  `TestSyncVerificationIsRetriedWhileTheLeaseHolds`、`TestSyncVerificationStopsWithoutSettlingWhenTheLeaseEnds`；
  `internal/repoexec/sync_test.go` 的 `TestSyncFollowsExternalForcePush`、`TestSyncExternalFailures`。已覆盖：两个 ref 都指向报告
  提交才落终态；不一致为 `failed(verification_mismatch)`；租约内重试；force-push 后 `forced = true` 且旧快照 ref 保留；分支删除为
  `failed(upstream_branch_missing)`。仍缺：未断言当前快照行与 `read` 能力在同一终态事务内写入。
- 决策依据：D5、D6，不变量 5、6、7。

## A Terminal Sync Transaction Creates At Most One Successor

- 风险：自动重放与补同步同时需要时建两个，撞上“一个未结束的同步”的部分唯一索引，整个终态事务失败，原 Operation 停在未结束状态；系统触发的同步因凭据冻结而回滚所在事务。
- 前置状态：`running` 的同步；发布成功后 Binding 带补同步标记；凭据可以被冻结。
- 触发：带标记时同步以 `failed(lease_lost)` 结束；不带标记时以可重放的失败结束；带标记时同步成功；冻结凭据后让带标记的同步结束；Project 进入 `deleting` 后已在 `running` 的同步收尾。
- 必须成立：按 D3 的四种组合至多新建一个后继，重放优先并沿用原 `trigger`、清除标记；只有标记时新建 `after_publish`；后继建不了时不新建、清除标记、把原因记入 `result.successorSkipped`，原 Operation 照常落到终态。
- 禁止结果：同一事务新建两个同步；标记残留到下一次无关的终态；系统触发的条件不满足让终态事务回滚或返回 `409`。
- 验证义务与证据：**Partial** — `integration/external_git_test.go` 的
  `TestRepositorySyncLeaseFencingReplayAndResyncAreAtomic`、`TestRepositorySyncSystemSuccessorRecordsSkipReason`；
  `integration/repository_publish_external_test.go` 的 `TestRepositoryPublishSuccessSchedulesExactlyOneSyncAfterRunningSync`。
  已覆盖：终态事务由重放吸收补同步标记、重放有上限、后继建不了时只记 `result.successorSkipped`、多次补同步合并为一个
  `after_publish`。仍缺：未断言重放沿用原 `trigger`。
- 决策依据：D3、D7，不变量 12、13；契约 D5 第 3 项。

## Deletion Aborted Syncs Are Executed Like Any Other Sync

- 风险：底座在删除回滚时补回的 `deletion_aborted` 同步不被 worker 领取，或被当作非法 `trigger`。
- 前置状态：删除被回滚，底座建立了一行 `pending`、`deletion_aborted` 的同步。
- 触发：worker 领取；凭据在回滚期间被冻结。
- 必须成立：worker 照常领取执行并推进当前快照；凭据不可用时在签发授权前以 `failed(credential_unavailable)` 结束。
- 禁止结果：`deletion_aborted` 的同步一直停在 `pending`；未检查凭据就签发授权。
- 验证义务与证据：**Missing** — `integration/repository_binding_test.go` 的
  `TestAbortedProjectDeletionRestoresTheOwedSync` 只断言底座建立了这一行；回滚补回同步本身由代码中心契约用例登记（**Covered**）。
  没有测试让 worker 领取并执行 `deletion_aborted` 的同步，也没有它签发授权前的 `failed(credential_unavailable)` 路径。
- 决策依据：D2、D3、D7。

## The Binding Is The Only Source Of Repository, Branch And Credential

- 风险：`projects` 与 Binding、或 Binding 上的两列凭据并存，读取方各自选择来源；默认分支出现两处。
- 前置状态：经公开 API 建立的 Project。
- 触发：读取 Project；候选凭据同步成功后切换；用户经 API 修改仓库地址或分支。
- 必须成立：`defaultBranch` 从 `Binding.branch` 读出；Binding 只有 `credential_ref_id` 一列凭据，`repositoryCredentialRefId` 不再出现在 API；候选切换在 `version` 未变时原子改写 `credential_ref_id`；仓库地址与分支不可修改。
- 禁止结果：存在 `repository_credential_ref_id` 列或按优先级合并两列；`version` 已变仍切换；修改分支成功。
- 验证义务与证据：**Partial** — `integration/repository_binding_test.go` 的 `TestCreateProjectRecordsBindingAndInitialSync`；
  `integration/migration_upgrade_path_test.go` 的 `TestMigration0043ExternalGitLineAppliesFreshAndUpgrades`。已覆盖：Project 的
  `defaultBranch` 从 Binding 读出；迁移后 Binding 只剩一个凭据列且有 `version`。仍缺：没有经 API 拒绝修改仓库地址或分支的测试，
  也没有 `version` 未变时候选凭据原子切换的测试。
- 决策依据：D1、D5，不变量 1、10、14。

## Unavailable Credentials Never Start A Sync

- 风险：冻结、撤销或范围不符的凭据被用于新的同步；公开仓库回退到部署级默认凭据。
- 前置状态：团队、个人、未知三类引用；成员离开；公开 Binding。
- 触发：手动同步；签发授权前凭据被撤销；匿名同步要求认证的仓库；外部以认证失败拒绝。
- 必须成立：用户发起时返回 `409 repository_credential_unavailable` 或 `409 repository_credential_frozen`；签发前发现不可用以 `failed(credential_unavailable)` 结束；匿名要求认证时 `failed(repository_requires_credential)`；认证失败把引用版本标为 `invalid`；已 `running` 的同步可以收尾。
- 禁止结果：不可用的引用被兑换；公开 Binding 使用任何凭据。
- 验证义务与证据：**Partial** — `internal/repoworker/operation_test.go` 的 `TestRefusedStartNeverReachesTheExecutor`；
  `internal/repoexec/sync_test.go` 的 `TestSyncExternalFailures`；`integration/external_git_test.go` 的
  `TestRepositorySyncSystemSuccessorRecordsSkipReason`。已覆盖：拒绝的开始不会到达执行器；匿名访问私有仓库为
  `repository_requires_credential`；认证失败为 `credential_rejected`；冻结凭据阻止后继。仍缺：没有用户侧
  `409 repository_credential_unavailable` / `repository_credential_frozen` 的测试，也没有签发前撤销、公开 Binding 不使用凭据、
  认证失败把版本标为 `invalid` 的测试。
- 决策依据：D4、D7，不变量 8。

## Snapshot Waiters Always End Once The Owed Sync Settles

- 风险：内部线的 clone 步骤在没有快照时无限等待；会重放的失败被误读为最终失败。
- 前置状态：新建的 Project，main Workspace 的 clone 步骤以 `snapshot_not_ready` 等待。
- 触发：首次同步成功；以永久失败结束；以可重放的失败结束。
- 必须成立：每个 Binding 从创建起就有同步；同步一定落到终态；重放与终态同一事务，所以“最近的同步”要么未结束、要么不会再重放；永久失败以 `failed` 和错误码落库。
- 禁止结果：没有快照也没有未结束同步的 Binding；终态与重放之间存在可见的空窗。
- 验证义务与证据：**Missing** — 没有测试驱动快照等待者或 `snapshot_not_ready` 路径；只有
  `TestCreateProjectRecordsBindingAndInitialSync` 断言 Binding 从创建起就有同步。等待者的结束依赖内部线的 clone 步骤，
  本线无法单独证明。
- 决策依据：D9，不变量 11、12。
