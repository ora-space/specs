# clone 步骤消费 Binding 当前快照

依据已批准的 [clone 步骤消费快照决策](../../../decisions/cloud/operation/20261016-clone-step-consumes-binding-snapshot.md)。下文的
D 编号与不变量编号都指这篇 ADR；“契约”指
[代码中心契约决策](../../../decisions/cloud/git-server/20261016-project-repository-as-code-hub-and-ref-contract.md)。

Binding、当前快照与同步 Operation 的存储，以及 create_project 失败时把 Project 置为 `active`，由底座实现并登记在
[代码中心契约用例](../git-server/project-repository-code-hub-contract.md)。本文登记 clone 步骤在其上增加的义务。内部线的实现
尚未合入 cloud 的 main，下列义务都是 **Missing**。

## The Clone Input Is Frozen Once From The Current Snapshot

- 风险：clone 输入每次重试从当前快照重新读取，同一 operation 的几次执行 clone 了不同代码；输入里仍出现外部仓库地址或凭据引用。
- 前置状态：Binding 有当前快照 `S1`；create_workspace 进入 clone 步骤。
- 触发：clone 第一次失败后，预置工具把当前快照改为 `S2`，operation 重试；Controller 转交的输入被改动一个字段。
- 必须成立：`request.clone` 在进入 clone 步骤或领取的事务里冻结为 `{kind: "clone", repositoryUrl, snapshotRef, baseCommit, workspaceRef}`，四个字段都指向 Git Server 与 `S1`；重试的执行输入与它逐字相同；`RecordDispatch` 拒绝不同的输入；成功后 `workspaces.base_commit_id` 等于 `baseCommit`。
- 禁止结果：重试使用 `S2`；输入含外部仓库地址、`credential_ref` 或 Project 的 `default_branch`；同一 operation 冻结两次。
- 验证义务与证据：**Missing**。
- 决策依据：D1、D3，不变量 1、2、5。

## A Clone Step Without A Snapshot Waits Or Fails By The Latest Sync

- 风险：没有快照时 clone 回退外部仓库；等待者永不结束；同步失败后 operation 仍在等待；等待消耗三次尝试次数。
- 前置状态：经公开 API 新建的 Project，Binding 只有建 Project 时的那一行同步，没有当前快照。
- 触发：同步保持 `pending`、`leased`、`running`；同步以 `failed`、`superseded`、`cancelled` 结束且没有后继；同步失败但同一事务建了重放。
- 必须成立：最近的同步未结束时 operation 为 `retry_wait`、`snapshot_not_ready`，`retry_at` 约 10 秒后，不登记 clone 执行；最近的同步以非成功终态结束时，下一次领取把 operation 以 `failed`、`repository_sync_failed` 结束并记录该同步 ID，运行的 `preparation.syncErrorCode` 取该同步的 `error_code`；有重放时继续等待；快照登记后的第一次执行仍是第 1 次尝试。
- 禁止结果：访问外部仓库；等待期间出现 `clone_executions` 行或 Node 工作；等待计入根 D2 的三次尝试；`preparation` 暴露 Binding 或快照身份。
- 验证义务与证据：**Missing**。
- 决策依据：D2，不变量 3；契约 D5 第 2、3 项。

## Claiming Skips Waiting Operations Instead Of Returning Empty

- 风险：领取只取最早的一个 operation，排在最前的等待者每 10 秒让其他 operation 空等一轮。
- 前置状态：一个到期的等待快照的 create_project，排在后面有一个可执行的 operation；另有一个等待同步结束的 delete_project。
- 触发：Controller 调用 `ClaimOperation`。
- 必须成立：领取事务把等待者推后，在同一事务里继续选下一个可领取的 operation 并交出；全部都在等待时才返回空。
- 禁止结果：领取返回空而还有可领取的 operation；等待者被交给 Controller。
- 验证义务与证据：**Missing**。
- 决策依据：D2“领取不因等待者落空”、D6，不变量 4。

## The Workspace Branch Comes From The Binding Branch

- 风险：分支短名从 `projects.default_branch` 或请求读出，与快照同步的分支不一致；isolated Workspace 以未同步的分支为基线。
- 前置状态：Binding 的 `branch` 为 `main`。
- 触发：创建 main、run 与 isolated Workspace；isolated 的 `baseRef` 分别为空、`HEAD`、`main`、`feature`。
- 必须成立：`requested_ref` 等于 `Binding.branch`，`workspaceRef` 等于 `refs/heads/workspaces/<workspaceId>/<requested_ref>`；`baseRef` 为空、`HEAD` 或等于 `Binding.branch` 时接受。
- 禁止结果：`baseRef: feature` 建出 Workspace 或 operation（必须 `400 base_ref_not_synced`，不留行）。
- 验证义务与证据：**Missing**。
- 决策依据：D3，不变量 6。

## Creating A Run Workspace Never Triggers A Sync

- 风险：每次 IssueRun 都隐式同步外部仓库，运行的代码随外部变化而漂移，也让 run 依赖外部仓库可达。
- 前置状态：Binding 有当前快照。
- 触发：触发 IssueRun。
- 必须成立：run Workspace 的 `request.clone` 等于触发时的当前快照；当前快照记录与同步 Operation 不变。
- 禁止结果：新建同步 Operation；`request.clone` 等待新的同步。
- 验证义务与证据：**Missing**。
- 决策依据：D4，不变量 7。

## Tenant Level Clone Requests No Longer Exist

- 风险：租户级 clone 的公开入口、表或领取分支残留，形成绕过快照、直接 clone 外部仓库的路径。
- 前置状态：迁移后的数据库与公开 API。
- 触发：请求原租户级 clone 的路由；Controller `ClaimWork`。
- 必须成立：`POST`/`GET /api/v1/tenants/{tid}/clones` 不在路由与 OpenAPI 中；`clone_requests` 表不存在；`ClaimWork` 不返回 clone 工作；`clone_executions` 中只有属于 Workspace operation 的执行。
- 禁止结果：为旧接口保留兼容响应；生成的前端客户端或 clones 页仍然存在。
- 验证义务与证据：**Missing**。
- 决策依据：D5，不变量 8。

## Project Deletion Waits For Repository Operations Before Quiesce

- 风险：同步或发布仍在进行时 Node 已经 quiesce、仓库被删除，进行中的 Operation 写入已删除的仓库；或者等待被当作 Node 不空闲，让 delete_project 失败并恢复准入。
- 前置状态：Project 有一个 `running` 的同步；另一个 Project 有 `running` 的发布。
- 触发：delete_project 进入 quiesce 步骤并被领取；Operation 随后结束。
- 必须成立：仍有未结束的同步或发布时，领取把 delete_project 置为 `retry_wait`、`repository_operation_running`，不交给 Controller；Operation 结束后的下一次领取正常交出；等待不计入任何尝试次数。
- 禁止结果：Controller 在 Operation 结束前开始 quiesce；delete_project 因等待而失败或恢复准入。
- 验证义务与证据：**Missing**。
- 决策依据：D6，不变量 9；契约 D5 第 3 项。
