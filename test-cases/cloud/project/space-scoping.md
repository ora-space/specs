# Project Space 作用域与角色门控

决策依据：[Project 归属 Space](../../../decisions/cloud/project/0-project-space-scoping.md)（历史归属与外键）及[一租户一空间](../../../decisions/cloud/tenancy/20260923-one-space-per-tenant.md)（当前成员授权）。
以下用例通过真实 PostgreSQL 集成测试（`integration/project_space_test.go`、
`integration/cloud_test.go`）与前端 Vitest 验证。

运行时内容与操作的目标规则已由[创建者/管理员权限](../workspace/runtime-workspace-use-permissions.md)收紧；仓库凭据见[连续性核心用例](repository-credential-continuity.md)。下列原证据保留共享项目、owner 外键与生命周期义务，不证明新增创建者记录、独占或凭据治理。

## Members create and read projects inside their space

- 风险：成员无法在 Space 内创建或读取 Project，协作边界形同虚设；或响应缺失归属信息。
- 前置状态：bob 为该 Space 所属租户的 member。
- 触发：bob `POST /spaces/:sid/projects`；随后列出 Space 项目。
- 必须成立：创建返回 202（既有异步语义）；列表恰含该项目且响应携带 `spaceId`。
- 禁止结果：member 创建被拒、列表缺失、响应不含 spaceId。
- 验证义务与证据：**Covered** — `integration TestProjectSpaceScopingAndRoleGates` 断言
  member 创建 202、列表 1 项且 spaceId 与 Space 一致。

## Members create and list independent runtimes in a shared project

- 风险：将操作发起者误当成持久资源所有者导致外键失败；运行时列表仍依赖已停用的工作树表而漏项。
- 前置状态：项目由 alice 创建，bob 是同租户普通成员，主运行时已就绪。
- 触发：bob 创建独立运行时，Controller 完成 Node 克隆，随后 bob 查询项目运行时列表。
- 必须成立：持久归属沿用项目所有者，操作发起者为 bob；双方可见安全概况，受限操作内容按运行时创建者/管理员校验；列表包含主运行时与独立运行时，均就绪且提交标识正确。
- 禁止结果：拒绝合法成员创建、资源归属被改写、列表为空或重新插入旧存储/工作树记录。
- 验证义务与证据：**Partial** — `integration.TestDifferentMemberCanCreateRuntimeInSharedProject` 覆盖跨成员创建、owner/actor、克隆及列表的原义务；`TestConcurrentMembersCannotStartTwoProjectOperations` 直接证明项目生命周期竞争。新增内容投影与运行时创建者使用权限为 `Missing`，见上方核心用例。
- 决策依据：一租户一空间 D1，以及上游每个运行时独立克隆的生命周期约束。

## Non-members cannot read a project or its runtime workspace

- 风险：仅凭 Project UUID 绕过 Space 边界读取内容或运行时状态。
- 前置状态：carol 未加入该 Space 对应租户。
- 触发：carol `GET /projects/:pid` 与 `GET /workspaces/:wid`。
- 必须成立：租户入口返回 403，不返回任何资源内容。
- 禁止结果：200、或返回 403 以外码泄露存在性。
- 验证义务与证据：**Covered** — `integration TestProjectSpaceScopingAndRoleGates`（carol
  读取 403）与 `TestIdentityConcurrencyMembershipAndIsolation`（成员加入后 200，形成对照）。

## Members update business content and stale versions conflict

- 风险：业务写被错误门控，或多人编辑丢失覆盖；乐观锁语义在 Space 化后失效。
- 前置状态：bob 为 member，Project version=v。
- 触发：bob 以 version=v 修改 name；随后以同一 version 再次修改。
- 必须成立：第一次 200 且 name 生效；第二次 `409 version_conflict`。
- 禁止结果：member 修改被拒、陈旧版本覆盖成功。
- 验证义务与证据：**Covered** — `integration TestProjectSpaceScopingAndRoleGates` 断言
  member PATCH 200 与陈旧 PATCH 409。

## A different member creates a runtime under the project's durable owner

- 风险：共享项目允许其他成员操作，但运行时行若误用操作人的 `owner_user_id`，会违反项目复合外键；若操作记录只允许项目创建人读取，发起人又无法跟踪结果。
- 前置状态：alice 创建并完成 Project，bob 为同租户有效 member。
- 触发：bob 在该项目创建 isolated Workspace，alice 和 bob 读取该 operation。
- 必须成立：创建返回 202；运行时持久 owner 与 Project 相同，operation actor 为 bob；两人可读取安全概况，受限内容需运行时使用权限。
- 禁止结果：外键错误变成 500、把运行时错误归属于 bob、成员无法查看共享操作。
- 验证义务与证据：**Partial** — `integration TestDifferentMemberCanCreateRuntimeInSharedProject` 证明原 owner/actor 及可读链路；新增创建者记录与内容过滤为 `Missing`，见运行时使用权限核心用例。

## Concurrent members cannot start two lifecycle operations on one project

- 风险：两名成员同时在同一项目下创建运行时，两个尚未完成的副作用交错并破坏项目工作树或归属。
- 前置状态：alice 创建的 Project 已就绪，bob 是同租户有效成员。
- 触发：两人同时提交各自的 isolated Workspace 创建请求，使用不同幂等键。
- 必须成立：恰有一个请求进入异步操作（202），另一个收到 `409 operation_in_progress`；只新增一个 isolated Workspace。
- 禁止结果：同一项目存在两个并行未完成的生命周期操作、部分资源重复创建。
- 验证义务与证据：**Covered** — `integration TestConcurrentMembersCannotStartTwoProjectOperations`。

## Members cannot delete projects; admins delete through the lifecycle state machine

- 风险：删除权限过宽；或 Space 化后删除绕过既有 lifecycle 编排变成直接物理删除。
- 前置状态：Project lifecycle=active；bob 为 member。
- 触发：bob `DELETE /projects/:pid`；随后 admin 删除并推进编排。
- 必须成立：bob 返回 `403 space_role_required`；admin 返回 202 且 resource lifecycle 进入
  deleting；编排完成后 `deleted_at` 置值。
- 禁止结果：member 删除成功、admin 删除不走状态机、或删除被静默降级为软删。
- 验证义务与证据：**Covered** — `integration TestProjectSpaceScopingAndRoleGates` 断言 403、
  deleting 状态、drain 后 deleted_at 置值。本用例的前置 `lifecycle=active` 是场景而非不变量：
  admin 可删除的 Project 状态由下一条用例固定。

## Deleting a project supersedes what is still running on it

- 风险：删除被要求依赖普通用户无法制造的中间状态，导致失败或永远停在创建中的 Project 永久不可删；
  或反过来，为了删掉它而把仍可被领取的 Operation 留在队列里，使已被删除的 Project 在删除之后继续被
  provisioning；或为了让删除通过而改写、丢弃既有执行记录与失败事实。
- 前置状态：admin 拥有的 Project，其 provisioning 已经失败（`lifecycle` 仍是 `provisioning`，其
  唯一 Operation 为 `failed`，Workspace 的 runtime control 仍由该已结束的 Operation 持有的
  `maintenance` 占据）；另一种前置是同一 Project 的 Operation 仍为 `queued`，尚未被任何 Controller
  领取。
- 触发：admin `DELETE /projects/:pid`，随后 Controller 推进该删除 Operation。
- 必须成立：两种前置都返回 202 并走既有 quiesce→terminate→cleanup 编排；删除请求把该 Project 上
  仍然存活的 Operation 结算为 `failed` + 该动作自己的 `error_code`、清空 `retry_at`，因此删除之后
  没有任何 Operation 会被领取或派发；已结束 Operation 遗留的 `maintenance` 占用（其 Operation 已
  不存活）可被删除接管，而已不存在的沙箱不构成拒绝理由；编排完成后 Project `lifecycle='deleted'`、
  其 Workspace 与沙箱按既有机制清理；既有 clone 执行记录、Operation 的 request 与 actor 原样保留，
  且该 Operation 不会因被删除而变成 `succeeded`。
- 禁止结果：以 `resource_unavailable` 或 `runtime_control_held` 笼统拒绝上述任一前置；把未完成的
  Operation 留在队列中导致 Project 删除后仍被 provisioning；丢弃或改写执行记录与失败原因；把删除
  降级为直接物理删除（绕过 quiesce 与数据清理）。
- 验证义务与证据：**Covered** — `integration
  TestProjectDeleteAcceptsAProjectWhoseProvisioningFailed`（clone 预算耗尽 → Operation `failed`、
  Project 仍为 `provisioning`、runtime control 仍被遗留占用 → 删除 202、重复删除
  `operation_in_progress`、drain 后 `lifecycle='deleted'`、无存活 Workspace／沙箱、clone 执行记录与
  `failed` 原因保留）与 `integration TestProjectDeleteSupersedesWaitingWorkAndDeletesTheProject`
  （`queued` Operation 被结算为 `failed` + `superseded_by_project_deletion`、`retry_at` 为空、未被
  执行、Project 被删除）。正常 `active` Project 的删除路径不回归由既有
  `TestProjectDeleteClosesEveryWorkspaceAndWaitsForCleanup` 继续证明。
- 决策依据：[Project 归属 Space](../../../decisions/cloud/project/0-project-space-scoping.md) D1
  （删除仍走既有 lifecycle 状态机且要求 admin/owner）与
  [凭据化 Clone 拒绝即终态](../../../decisions/cloud/operation/20261012-credentialed-clone-refusal-is-terminal.md)
  （拒绝被结算为 `failed` 后 Project 仍停在 `provisioning`）。本用例不改变该决策未决的"失败 Project
  如何重新驱动"问题：删除是用户的退出路径，不是对失败 Project 的重新驱动。

## Project lists include only projects from spaces the caller joined

- 风险：列表泄漏未加入 Space 的 Project；或继续按 owner 过滤导致共享项目不可见。
- 前置状态：alice 在租户唯一 Space 创建 Project；new-user 加入 tenant。
- 触发：new-user `GET /projects`（tenant 级）。
- 必须成立：列表恰含该共享 Project；加入前不可见。
- 禁止结果：按 owner 过滤导致列表为空、或未加入时可见。
- 验证义务与证据：**Covered** — `integration TestIdentityConcurrencyMembershipAndIsolation`
  断言加入后列表恰 1 项。

## Cross-tenant project-space bindings are rejected at SQL level

- 风险：复合外键缺失或可被绕过，出现 Project 与 Space 跨 tenant 归属。
- 前置状态：两个 tenant 各有唯一 Space。
- 触发：直接 SQL 插入 `space_id` 来自其他 tenant 的 Project。
- 必须成立：外键拒绝，插入失败。
- 禁止结果：跨 tenant 行入库。
- 验证义务与证据：**Partial** — 复合外键 `(space_id, tenant_id)` 由 migration 0007 建立且
  0007 的回填必须先于 NOT NULL 收紧（迁移成功即证明无 orphan）；尚无直接注入跨 tenant 行的
  违反性测试。

## The UI hides role-gated actions from members

- 风险：前端角色判断与后端矩阵不一致，member 看到并尝试被拒的操作。
- 前置状态：member 登录的浏览器会话。
- 触发：渲染 Project 详情、成员页与设置页。
- 必须成立：member 看不到删除入口与成员管理控件；admin 可见改名与管理控件。
- 禁止结果：member 视图出现删除/成员管理入口。
- 验证义务与证据：**Covered** — 前端 `project-detail-page.test.tsx`
  （hides the delete action from members）、`members-page.test.tsx`
  （hides management controls from members）、`general-settings-page.test.tsx`
  （prevents non-admins from renaming the tenant）。
