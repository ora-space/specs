# 运行时 Workspace 生命周期

依据[已批准的运行时 Workspace 生命周期决策](../../../decisions/cloud/operation/0-workspace-runtime-follows-desktop-node.md)
与[具体默认分支决策](../../../decisions/cloud/operation/20260926-require-concrete-default-branch.md)。
本文件登记 Cloud 运行时 Workspace（表 `workspaces`）按 sandbox → node → clone → plugin 推进、丢弃 storage/worktree
步骤、由 Controller 代报 Node 的承诺；gRPC 形状见
[Workspace 操作与 Node 上报用例](../controller-integration/workspace-operations-and-node-reports.md)。
证据来自 `ora-space/cloud` 的 `integration` 包（PostgreSQL 集成测试）。

## Workspace Opens Admission Only After Node And Clone Succeed

### 风险

Node 已连接但仓库尚未 clone 时开放准入，任务在没有代码的沙盒里执行；或 clone 失败、未知时仍把
Workspace 标为 ready。

### 前置状态

新建 Project 或 isolated Workspace，Controller 持有有效租约。

### 触发

`create_project`/`create_workspace` 依次推进 sandbox、node、clone；clone 分别成功、失败（含 `Interrupted`）、
结果未知。

### 必须成立

- sandbox 阶段只计划 `sandbox_ensure`（请求不含仓库字段），成功证据 `sandboxInstanceId` 等于 effect ID。
- node 阶段要求当前 generation 的 Node 已由 Controller 登记、initialized、30 秒内心跳。
- clone 成功后写入基线提交并进入 plugin；固定插件集合为空或全部逐项结果已接管后才开放准入并 ready。
  plugin 的直接证据见 [插件步骤用例](plugin-step-and-run-workspace.md)。已确认 clone 失败的前两次进入 `retry_wait`，
  第三次失败关闭受理并结束准备，全部执行记录保留；未知保持 `blocked`，不自动重试。
- 一个 operation 的 clone 阶段至多一个进行中的执行；重试是新执行。
- `start` 走 sandbox → node → plugin → done，不再 clone。

### 禁止结果

clone 未成功时开放准入；计划 `storage_ensure`、`worktree_ensure`、`worktree_delete`、`storage_delete`；
未知 clone 结果被自动重试。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 新步骤表与只计划保留的 effect | `integration/workspace_lifecycle_test.go` `TestWorkspaceStepsPlanOnlyRetainedEffects`：create_project、create_workspace、stop、start 各自计划的 effect 种类与 clone 执行数；计划退役 kind 返回 `invalid_step`，数据库拒绝退役行 | Covered |
| clone 成功才开放准入 | `integration/workspace_operations_grpc_test.go` `TestGRPCControllerDrivesWorkspaceThroughNodeAndClone`：空插件集合时 clone_ready 后写入基线并开放准入；非空集合由 `TestRunWorkspaceAndPluginStep` 证明 plugin 完成前仍关闭 | Covered |
| clone 失败与未知的处理 | `integration/workspace_lifecycle_test.go` `TestCloneFailureRetriesAsNewExecutionAndUnknownBlocks`：失败进入 `retry_wait`/`clone_failed` 并保留执行记录、准入保持关闭、重试为第二个执行；未知结果阻止登记第二个执行、不能推进，`blocked` 后不再被领取 | Covered |

### 决策依据

运行时 Workspace 生命周期 D1、D2，不变量 1、2、5。

## A Clone Step Is Attempted At Most Three Times Before Its Operation Is Failed

### 风险

clone 步骤的失败被无界重试：一个仓库在这套部署里不可达时，Operation 在 `retry_wait` 与 `running` 之间
永久循环，Workspace 永不 ready、Project 永远停在 `provisioning`；并且由于它始终是最旧的候选，它后面
的 Operation 全部得不到调度。

### 前置状态

一个已领取的 `create_project` Operation，停在 `clone` 步骤，其 Node 持续上报克隆失败（例如
`CLONE_FAILURE_REASON_INTERRUPTED`）。

### 触发

Controller 对同一步骤重复上报失败，第三次失败到达 Cloud。

### 必须成立

- 预算由 Cloud 持有并施加。前两次失败进入 `retry_wait`/`clone_failed`，退避依次为 5 秒与 10 秒；Controller
  在 defer 中提出的延迟不决定这两次的时间（它只是一个 Controller 常量）。
- 第三次已知失败把 Operation 结算为 `failed`/`clone_failed`，并清空 `retry_at`。
- 计数来自持久执行记录（每次尝试一条 `clone_executions`），因此每条记录都保留；人工 retry、重复提交与
  Controller 重启都不重置预算。
- 结算后的 Operation 不再被领取，也不能经 `POST /operations/{oid}/retry` 重试
  （`409 operation_not_retryable`）。
- 两条路径上准入都保持关闭：只有 clone 成功才开放准入。

### 禁止结果

出现第四次克隆尝试；把预算记在 Controller 或进程内存里；结算时删除或改写执行记录；结算为 `failed`
时留下非空 `retry_at`（`claim` 会把它读回成一次待重试）。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 前两次失败按 5／10 秒退避且各留一条执行记录 | `integration/workspace_lifecycle_test.go` `TestCloneStepStopsAfterItsThirdFailure`：逐次断言 `retry_wait`/`clone_failed`、`retry_at - now` 落在 5±1 秒与 10±1 秒、执行记录数等于本次尝试序号 | Covered |
| 第三次失败结算为终态 | 同一测试：第三次失败后 `state='failed'`、`error_code='clone_failed'`、`retry_at IS NULL`、准入未开放、下一次 claim 无工作、`POST /operations/{oid}/retry` 返回 `409 operation_not_retryable` | Covered |
| 预算不因人工 retry 重置 | 同一测试：每次 park 后用公开 retry 入口使其立即到期以驱动三次尝试，退避仍按执行记录递增，结算时三条记录都在 | Covered |

### 决策依据

运行时 Workspace 生命周期 D2（2026-10-11 检视修订），不变量 2、5。

## Workspace Data Is Deleted Only After Every Sandbox Is Terminated

### 风险

Workspace 或 Project 删除时仍有沙盒在写数据卷，清理与写入竞争。

### 前置状态

Workspace 有未终止或已终止的 sandbox。

### 触发

`delete_workspace`/`delete_project` 推进到 cleanup。

### 必须成立

- cleanup 为每个 Workspace 计划 `workspace_data_delete`，计划前要求该 Workspace 没有未终止 sandbox。
- `delete_project` 在全部 Workspace 的数据删除成功后才推进到 done，不再有 `storage_delete`。

### 禁止结果

存在未终止 sandbox 时计划 `workspace_data_delete`；Project 删除跳过任何 Workspace。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 终止后才计划删除 | `integration/workspace_lifecycle_test.go` `TestWorkspaceDataDeleteWaitsForTermination`：sandbox 未终止时计划 `workspace_data_delete` 返回 `termination_unconfirmed` 且未登记 effect | Covered |
| Project 删除覆盖全部 Workspace | `integration/endpoints_test.go` `TestProjectDeleteClosesEveryWorkspaceAndWaitsForCleanup`：两个 Workspace 的 Project 删除，数据删除失败时 Project 与 Workspace 均保留；重试后两个 `workspace_data_delete` 成功、数据目录消失后 Project 才删除 | Covered |

### 决策依据

运行时 Workspace 生命周期 D3，不变量 6。

## Retired Storage And Worktree Steps Leave History Intact

### 风险

迁移删除或改写已有 storage/worktree 记录，或在途旧步骤 operation 永远卡住。

### 前置状态

升级前数据库中存在 `project_storage`、`workspace_worktrees` 行，以及停在 `storage`/`worktree` 步骤的 operation。

### 触发

应用新迁移。

### 必须成立

- 旧行保持原值；在途旧步骤 operation 以 `failed`、`lifecycle_flow_retired` 结束；已计划的旧 effect 记录保留。
- 停在 `storage_delete` 的在途 `delete_project` 退回 `cleanup`，按新规则删除每个 Workspace 的数据后完成，不以
  `failed` 结束（它已越过 sandbox 终止，失败会留下已关闭却未删除的 Project）。
- 迁移可重试；新 Project/Workspace 不再写入旧表，数据库拒绝新的退役步骤、effect 与旧表行。

### 禁止结果

删除或改写旧行；在途旧步骤 operation 继续推进或保持非终态。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 从上一版 schema 升级 | `integration/migration_upgrade_path_test.go` `TestMigration0016RetiresStorageAndWorktreeStepsKeepingHistory`：在 0015 schema 构造 storage/worktree 行、旧 effect 与在途 operation 后升级，断言旧行与 effect 不变、`storage`/`worktree` 在途 operation 以 `failed`/`lifecycle_flow_retired` 结束 | Covered |
| 迁移可重试 | 同一测试连续执行两次 `Migrate` 后再断言上述结果 | Covered |

### 决策依据

运行时 Workspace 生命周期 D6，不变量 7。

## A Workspace Never Stores HEAD As Its Requested Ref

### 风险

Project 以 `HEAD` 为默认分支被创建，Workspace 的 `requestedRef` 为 `HEAD`，desktop Node 拒绝 clone，operation
永久阻塞在 clone 步骤而 Project 已经存在。

### 前置状态

租户成员可以创建 Project；已有一个默认分支为 `main` 的 Project，以及一个本规则之前以 `HEAD` 创建的 Project。

### 触发

创建 Project 时 `defaultBranch` 缺省、空白或为 `HEAD`；在两个 Project 下以 `baseRef: HEAD` 创建 isolated Workspace。

### 必须成立

- 缺省、空白或 `HEAD` 的 `defaultBranch` 返回 `400 default_branch_required`，不写入 Project。
- `baseRef: HEAD` 存为 Project 的默认分支（`main`）。
- 默认分支仍为 `HEAD` 的旧 Project 拒绝 `baseRef: HEAD`（`400 default_branch_required`）。

### 禁止结果

新建的 Project 或 Workspace 存储 `HEAD`；被拒绝的请求留下记录；Cloud 为解析分支访问远端。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 创建时拒绝 | `integration/default_branch_test.go` `TestProjectsNameAConcreteDefaultBranch`：三种输入均为 `default_branch_required` 且 `projects` 为空 | Covered |
| `baseRef: HEAD` 的解释 | 同一测试：isolated Workspace 的 `requestedRef` 为 `main`；旧 `HEAD` Project 被拒绝 | Covered |
| 不访问远端 | 代码中没有远端读取路径，只能由评审保证 | Missing |

### 决策依据

具体默认分支 D1、D2，不变量 1–3。

