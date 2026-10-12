# 仓库凭据归属连续性

依据[仓库凭据归属连续性](../../../decisions/cloud/identity-access/20261010-repository-credential-ownership-continuity.md)
（`approved`，`0023` 的历史契约补录）。本文件登记 `0023` 已经交付、并且今后不得被反向改写的持久语义：
凭据的归属列、离开与停用时的冻结、以及"补录不改写迁移"。这些语义今天由已应用的迁移与触发器承担，
后续凭据决策（Secret Store、Broker 授权、Node 隔离）只能在其之上叠加，不得改写。

核心用例面向**已应用的持久契约**，因此证据形态以"数据库约束与触发器行为"和"迁移不可改写"为主，
而不是端到端业务流。凭据元数据与授权语义另见
[凭据元数据](repository-credential-metadata.md)，秘密承载另见
[凭据 Secret Store](repository-credential-secret-store.md)。

**当前实现状态（2026-10-10 本轮实现后）**：本文件登记的 `0023` 语义未变——`0001` 与 `0023` 逐字未动，
本轮的用户托管凭据只以前向迁移 `0033_user_git_credentials.sql` 追加了两个可加列。有一处**读取行为**
按凭据 ADR D7／D10 收紧（属于元数据决策，不属于本文件）：`projects.credential_ref_id` 成为唯一权威，
`repository_credential_ref_id` 不再被读取以选择凭据，两列不一致时以
`repository_credential_binding_conflict` 拒绝。其证据见
[凭据元数据](repository-credential-metadata.md) 的第一个用例。

## Departed Member Freezes Personal Credentials Without Rewriting Ownership

### 风险

成员离开或账号停用后凭据仍可用，或冻结被实现为删除／改写归属，使历史 Project 与 `clone_executions`
失去可解释的引用对象。

### 前置状态

`credential_refs` 存在同一 owner 的 `personal`、`unknown` 与 `team` 三类记录；`projects.credential_ref_id`
与 `projects.repository_credential_ref_id` 已指向其中一条；存在已完成或进行中的 `clone_executions`
绑定同一条凭据。

### 触发

把该成员的 `tenant_memberships.status` 置为非 `active`；随后把 `users.status` 置为非 `active` 或写
`users.deleted_at`。

### 必须成立

- `personal` 与 `unknown` 记录被置为 `availability='frozen'`，并写入可解释的 `unavailable_reason`
  （成员离开与用户停用分别对应 `associated_member_departed`、`associated_user_disabled`）。
- `team` 记录**不**因个人离开而冻结；团队凭据的归属规则尚未定义，不在这里顺带改变。
- `owner_user_id`、`tenant_id`、`secret_ref` 都**不被改写**：冻结是状态变化，不是归属变化或删除。
- `version` 递增，使冻结成为可 fencing 的状态迁移，而不是静默更新。
- 升级时已经处于非活跃状态的成员同样被冻结（迁移内的回填语句），不因"变更发生在升级前"而漏掉。

### 禁止结果

删除 `credential_refs` 行、清空 `secret_ref`、把归属改到他人或团队、把冻结实现为软删除、
只冻结 `personal` 而漏掉 `unknown`。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 成员离开冻结 `personal`／`unknown`，且不改写归属与 `secret_ref` | `integration.repository_credentials_test.go` 的 `TestCredentialDepartureFreezesPersonalAndUnknownButRetainsTeamHistory` 用真实 PostgreSQL 断言冻结状态、`unavailable_reason`、owner 与 `secret_ref` 不变 | Covered |
| 账号停用（`users.status`／`users.deleted_at`）走同一冻结语义 | 触发器路径仍缺，但**恢复扫描一侧本轮有了直接证据**：`core.TestFreezeSweepFreezesExactlyTheRowsWhoseOwnerIsGoneAndIsIdempotent` 为 `users.status='disabled'` 与 `users.deleted_at IS NOT NULL` 各配一行可用的 `personal` 凭据，且行写在账号状态改变**之后**（`users` 触发器看不到的那一批），断言 `freezeInactiveRepositoryCredentials`（`internal/core/repository_credentials.go:229`）把两行都冻结为 `frozen/associated_member_departed/2`。扫描对停用账号写入的是成员离开的 reason，测试把这个事实本身钉住，没有假装它区分两种原因。**仍缺**：`users` 触发器自身的路径测试（即不经扫描的那条路）| Partial |
| `team` 记录不因个人离开而冻结 | 同上测试的 `team` 分支断言 `availability='available'`；`internal/core` 的 `TestATeamReferenceSurvivesItsOwnersDepartureAtAdmission` 把它推进到**准入**：同样的记录、同样的离开，归属改成 `team` 后得到的第一个拒绝是部署级 `repository_credential_executor_unavailable`（而不是 `repository_credential_frozen`），且该行仍为 `available`——若个人规则被套用到 `team` 上，这条测试会在 owner 换角色时立刻失败；`TestTheRecoveryScanFreezesWhatTheTriggersCouldNotSeeAndLeavesTeamAlone` 另外断言恢复扫描不会把 `team` 行冻结 | Covered |
| 冻结经由触发器，而不是可选的应用层路径 | `internal/core` 的 `TestADepartedOwnersReferenceIsFrozenAtAdmission` 用**纯 SQL** 把成员置为 `disabled`、整个过程没有任何产品代码参与，随后断言该行变成 `frozen` 且 `unavailable_reason='associated_member_departed'`，因此冻结来自数据库触发器而不是某条可选的应用路径；`TestTheRecoveryScanFreezesWhatTheTriggersCouldNotSeeAndLeavesTeamAlone` 覆盖触发器看不到的补漏路径（离开之后才写入的行）。**不能绕过这一半仍无证据，而且不可能有**：直接改 `credential_refs` 的 SQL 本来就能绕过任何触发器——这正是准入守卫独立复核同一条件的原因，该复核由上面那条测试的第二段断言（把行改回 `available` 后仍以 `repository_credential_frozen` 被拒）。恢复扫描本轮的精度与幂等也有直接证据：`TestFreezeSweepFreezesExactlyTheRowsWhoseOwnerIsGoneAndIsIdempotent` 用 11 个行见证断言"恰好这些行"——`personal`／`unknown`／第三租户／跨租户双成员四种离开形态冻结，而 `team`、已撤销行（保住 `invalid/revoked_by_owner/3`）、已冻结行（保住 `version=5`）、两个活跃 owner 的行逐字节不变，活跃 Project 的绑定列不被改写；紧接着断言第二次扫描**一个字节都不改**（version 即写见证）。`TestASweepFrozenReferenceIsRefusedAtAdmission` 把被扫描冻结的行推进到准入：`repository_credential_frozen`、执行器零调用、`clone_executions` 零行，因此"扫描冻结了但没有任何路径看它"这一形态被排除 | Partial |
| 升级回填覆盖已非活跃成员 | 尚缺针对 `0023` 回填语句的迁移级测试 | Missing |

### 决策依据

归属连续性 ADR D1–D6，不变量 1、2、5。

## Rejoining Never Unfreezes A Frozen Credential

### 风险

"重新加入即恢复"被当作便利实现，使已经判定为不可用的凭据复活，并让"冻结"失去终态含义。

### 前置状态

某 owner 的 `personal`／`unknown` 凭据已因离开被冻结；该成员随后重新以 `active` 状态加入同一 tenant。
凭据记录本身未被触碰。

### 触发

重新激活 `tenant_memberships` 行；随后查询或尝试使用原凭据。

### 必须成立

- 重新加入**不**把 `availability` 改回 `available`；冻结只能由显式决策解除，不能由成员状态变化解除。
- 冻结与撤销的权威固定在 Cloud 元数据上：`secret_ref` 不变，判定读的是 `availability` 而不是
  Store 侧是否存在秘密。
- 冻结只影响**新的**远程 Git 工作；已经落地的历史记录（`clone_executions` 的绑定与版本）不被改写。

### 禁止结果

重新加入时批量解冻、按"成员已活跃"推断凭据可用、为了使用户"开箱即用"而重建一条新凭据行、
把冻结行删除再让用户重新提交。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 重新加入不解冻 | `integration.repository_credentials_test.go` 的 `TestCredentialDepartureFreezesPersonalAndUnknownButRetainsTeamHistory` 在改回 `active` 后断言仍有两条 `frozen` 记录 | Covered |
| 冻结不影响历史绑定与 `secret_ref` | 同上测试断言冻结后 `secret_ref` 不变；`integration.runtime_upgrade_test.go` 的 `TestRuntimeUpgradeOnlyBackfillsUniqueCreationEvidenceAndKeepsUnknownReferences` 断言升级不猜测 `attribution`、不改写 `credential_ref_id`／`repository_credential_ref_id` 绑定 | Partial |
| 冻结是终态而非可逆缓存 | 尚缺"重新加入后再次尝试凭据 clone 仍被拒绝"的直接测试 | Missing |

### 决策依据

归属连续性 ADR D3、D4，不变量 3、4、6。

## Applied Migration Semantics Are Records, Not Editable Drafts

### 风险

后续决策为了让自己的模型更整齐而"顺手"修改 `0023`，或新增第二份权威归属来源，使已部署实例与仓库
文本分叉、校验和不一致，且历史数据失去唯一解释。

### 前置状态

`0023_repository_credential_continuity.sql` 已在所有环境应用；存在解释其语义的补录 ADR
（本文依据的那份）与指向它的凭据决策。

### 触发

引入需要改变归属、可用性、`secret_ref` 解读或冻结规则的前向需求；或有人提议直接编辑该迁移文件。

### 必须成立

- 已应用的迁移文件**逐字不改**；任何语义变化都以前向迁移或前向决策承载，并说明对既有行的处理。
- 补录 ADR 只做记录与确认：不为既有语义新增写入者或读路径，不解除任何 fail-closed。
- 迁移校验和与 `specs/` 中的描述保持一致；`specs` 侧不得成为第二份可执行权威。
- 现有触发器是语义的实现手段，不是可以被应用层绕过的可选路径；若要改变触发行为，需要前向迁移
  加前向决策，而不是修改 `0023`。

### 禁止结果

编辑 `0023` 的行、在补录 ADR 中夹带实现变更、新增第二张"权威归属表"、用应用层逻辑覆盖触发器结论。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 迁移文件与校验和未被改动 | 尚缺显式的迁移校验和回归测试（当前依赖代码评审与 `internal/core/migrations` 的既有约定） | Missing |
| 新增迁移不反向改写既有语义 | `integration.runtime_upgrade_test.go` 的 `TestRuntimeUpgradeOnlyBackfillsUniqueCreationEvidenceAndKeepsUnknownReferences` 覆盖升级不猜测 `attribution`、不重写凭据绑定；`integration/migration_upgrade_path_test.go` 的 `TestMigrationFullSequenceFreshDB` 覆盖全序列可应用（含本轮追加的 `0033_user_git_credentials.sql`），但两处都不专门断言归属列与冻结规则在升级后逐项不变 | Partial |
| 补录 ADR 不引入行为变化 | 该补录本身不产生可执行证据；判定方式是核对 `specs` 与迁移的差异为零。**这属于评审证据，不是测试证据** | Missing |

### 决策依据

归属连续性 ADR D1、D7 与"为什么不能直接改写"；`cloud/AGENTS.md` 关于 migration 与兼容性的规则。
