---
status: implemented
---

# Project 经 Repository Binding 同步外部仓库：Repo Executor 写快照，Cloud 核验后推进当前快照

每个 Project 有一个 Repository Binding，记录外部仓库地址、要同步的分支和凭据引用。外部仓库的内容只经同步进入 Ora：
Cloud 创建一个持有租约的同步 Operation，调用 [Repo Executor](../repo-executor/0-repo-executor-sole-external-git-boundary.md)；
执行器在临时裸仓库里 fetch 外部分支，再经 Git Server 的服务仓库监听，一次原子推送写入以提交 OID 命名的快照 ref 和跟踪 ref。
Cloud 经 ref 查询核验两个 ref 都指向报告的提交之后，才在事务内推进 Binding 的当前快照。Workspace 只从当前快照取得代码输入，
没有当前快照时，Workspace 的 clone 步骤只能等待，不回退为访问外部仓库，公开仓库也要先同步一次。本文保证首次同步不会悄无声息地
停下：要么核验成功、写入当前快照，要么以 `failed` 终态落库，内部线据此结束等待（D9）。

本 ADR 处于 `implemented`：实现已由 cloud#76（`ceb39bc`）与 cluster#27（`033a4ea`）合入 main；核心测试用例见[仓库同步用例](../../../test-cases/cloud/project/repository-binding-and-snapshot-sync.md)，是 `cloud/project` 领域在[仓库凭据连续性](20260927-repository-credential-continuity.md)之后的后续决策，
遵守 [Git Server 代码中心契约](../git-server/20261016-project-repository-as-code-hub-and-ref-contract.md)（下称“契约”）的 D4、D5、D6。
涉及 Cloud（Binding、同步 Operation、ref 查询的调用方、公开 API 与前端）、Repo Executor 和 Credential Store。
发布（Publish）由[后续 ADR](20261017-explicit-fast-forward-publish.md) 承担；凭据的保管与能力验证由
[identity-access 后续决策](../identity-access/20261016-git-credentials-in-independent-credential-store.md) 承担。

兼容性边界：

- **破坏性切换**（契约范围边界）：不设过渡开关，不做数据迁移。Binding 直接替换 `projects` 的仓库字段：底座已经删除
  `projects` 的 `repository_url`、`default_branch`、`credential_ref_id`、`repository_credential_ref_id` 四列；Binding 上重复的
  凭据列由本文删除（D1）。已有的开发环境数据在切换后重建。
- Workspace 怎样消费当前快照、Node 怎样 clone，由内部线的 `cloud/operation`、`cloud/controller-integration`、`node/repository`
  后续 ADR 规定。本文只保证当前快照按契约 D5 第 2 项存在并被正确推进。
- 不改变 Revision 交付（内部线实现）。

## 继承与修改

| 前序约定 | 本文处理 |
| --- | --- |
| [Project 根决策](0-project-space-scoping.md)：Project 按租户成员可见和授权，`owner_user_id` 只作创建者审计 | 继承，不变。同步的触发与查看按同一成员规则（D3、D8） |
| [仓库凭据连续性](20260927-repository-credential-continuity.md) D1、D2：引用分为团队、个人、未知；关联成员离开后个人和未知引用冻结，拒绝新的远程操作；已开始的任务可以收尾 | 继承，不变。同步是“新的远程操作”，创建 Operation 和签发授权前都检查引用可用（D7） |
| 凭据连续性 D3：Controller/Node 在受控目标下验证候选引用，“只验证读取不能被记录为已经拥有推送权限” | 修改：验证的执行者改为 Repo Executor；读取能力由第一次成功同步证明（D5），推送能力的验证见 identity-access 后续决策 |
| 凭据连续性 D3：项目绑定保留原引用，候选验证成功后原子切换“有效引用”（实现为 `credential_ref_id` 与 `repository_credential_ref_id` 两列） | 修改：Binding 只保留 `credential_ref_id` 一列，候选验证成功后直接改写它（D1、D5）；切换的原子性、版本检查和失败不覆盖原引用不变。历史执行与引用的关联由每个同步 Operation 记录的 `credentialRefId`、`credentialVersion` 保留（D2） |
| 凭据连续性 D4：执行端按引用选择，不回退全局静态凭据；凭据临时提供、结束后回收 | 继承，不变。执行端从 Controller/Node 改为 Repo Executor |
| [默认分支决策](../operation/20260926-require-concrete-default-branch.md)：Project 必须给出具体默认分支；Cloud 不访问远端仓库 | 继承规则。修改存放位置：默认分支就是 `Binding.branch`，`projects.default_branch` 删除，公开 API 的 `defaultBranch` 从 Binding 读出（D1）；Cloud 仍不访问外部仓库，只经执行器访问 |
| 契约 D4：快照 ref create-only、名称等于目标；跟踪 ref tracking-cas，允许非快进 | 继承，不变。执行器的一次推送同时写两者（D2） |
| 契约 D5 第 2 项：每个 Binding 一个当前快照，只在核验后写，总是最新一次核验通过的同步结果 | 继承，不变。本文规定由哪些同步、在什么时机推进它（D3、D5） |
| 契约 D5 第 3 项：同步 Operation 的上下文（类型、Project、Binding、租约、可读可写） | 继承。本文规定 Operation 的状态、租约和 expected old 的取得方式（D2） |
| 契约落地顺序第 1 步：底座建立 Binding、当前快照和 Operation 的最小存储（状态枚举已定全，Binding 带补同步标记 `resync_requested`）；建 Project 的事务同时建立 Binding 和一行 `pending` 的同步 Operation | 继承。本文在 Operation 上增加租约、结果等字段，在 Binding 上增加 `version`，删除重复的凭据列，不增删状态（D1、D2）；首次同步由底座建立的那一行承担（D3） |
| 契约 D5 第 3 项：登记同步终态的事务至多新建一个后继同步；系统触发的同步建不了时只记录原因，不回滚所在事务 | 继承。本文规定两种后继同时需要时用哪个 `trigger`，以及记录在哪里（D3） |
| 契约 D5 第 3 项：进入 `deleting` 的事务取消未 `running` 的 Operation 并清除补同步标记；删除回滚时补回 `deletion_aborted` 的同步或重新置上补同步标记（底座已实现） | 继承，不变。本文把 `deletion_aborted` 列入 `trigger`，并说明它与本文 worker 的关系（D2、D3） |

## 决策

### D1：Repository Binding

Binding 是 Project 与一个外部仓库分支的绑定：

| 字段 | 含义 |
| --- | --- |
| `bindingId` | 稳定 ID，出现在 Git Server 的 ref 名 `refs/ora/upstreams/<bindingId>/...` 中，创建后不变 |
| `projectId`、`tenantId` | 所属 Project，复合外键保证同租户 |
| `repositoryUrl` | 外部仓库地址，第一阶段只接受 `https://`，不能带用户信息（`user:pass@`）、查询串和片段 |
| `branch` | 要同步的分支短名，必须是具体分支，不能是 `HEAD` |
| `credentialRefId` | 凭据引用，列名 `credential_ref_id`；为空表示公开仓库（D4） |
| `resyncRequested` | 补同步标记，列名 `resync_requested`：发布成功时已有同步在 `running`，需要在它结束后再同步一次（D3、发布 ADR D6）。底座已经建立 |
| `version` | 乐观版本，候选凭据切换时用于检查（D5）。底座没有，由本文增加 |

- 第一阶段每个 Project 恰好一个 Binding，数据库以 `UNIQUE(project_id)` 保证；多 Binding 留给后续决策。
- `bindingId` 不复用 `projectId`：ref 名中的 ID 要能在多 Binding 时保持不变，第一个 Binding 也用独立 ID。
- 修改 `repositoryUrl` 或 `branch` 等于换了上游，第一阶段不允许修改，只能新建 Project。修改 `credentialRefId` 按凭据连续性 D3
  的候选验证切换：候选引用用一次同步证明读取能力，成功后才原子切换（D5）。
- **Binding 的分支就是 Project 的默认分支**，只有这一处来源。创建 Project 的公开 API 字段不变（`repositoryUrl`、`defaultBranch`、
  凭据引用），但它们写入 Binding；读取 Project 时 `defaultBranch` 从 `Binding.branch` 读出。内部线的 Workspace 分支短名与要同步
  的分支都读 `Binding.branch`。
- 创建 Project 时按上表校验，不合法返回 `400`，不写任何记录。
- **凭据引用只有 `credential_ref_id` 一列**：底座的 Binding 表从 `projects` 原样搬来了 `credential_ref_id` 和
  `repository_credential_ref_id` 两列（后者是凭据连续性 D3 的“有效引用”，读取时优先于前者），公开 API 也有
  `credentialRefId` 和 `repositoryCredentialRefId` 两个字段。本文以 `credential_ref_id` 为准：它是创建 Project 时写入的列，
  也对应公开 API 必填的 `credentialRefId`；候选凭据切换（D5）直接改写它。`repository_credential_ref_id` 列和
  `repositoryCredentialRefId` 字段删除，原来优先读它的仓库访问检查改读 `credential_ref_id`。两列并存时，候选切换要同时维护
  “原引用”和“有效引用”，每个读取方都要按优先级合并；Binding 只有一个当前引用，切换是原子的，不需要第二列。

**为什么直接替换而不迁移**：Cloud 目前没有用户（契约范围边界）。保留旧列会出现两处“默认分支”和两处“凭据引用”，每个读取方都要
回答以哪一处为准；直接删除，Binding 就是唯一来源。

### D2：同步 Operation

同步记录为 `repository_operations` 中 `type = repository_sync` 的一行。这张表的 ID、类型、Binding、状态和状态枚举由底座建立
（契约落地顺序第 1 步），本文增加下表其余字段，不增删状态。发布 Operation 共用这张表（见发布 ADR），因为 Git Server 按
`Ora-Git-Operation-Id` 解析上下文时只需查一张表。

| 字段 | 含义 |
| --- | --- |
| `id` | Operation ID，也是执行器访问 Git Server 时的 `Ora-Git-Operation-Id` |
| `bindingId`、`projectId`、`tenantId` | 范围 |
| `trigger` | `project_created`、`manual`、`after_publish`、`credential_candidate`、`deletion_aborted`（底座在删除回滚时写入，见下文） |
| `requestedBy` | 触发的用户；系统触发时为空并记录来源 Operation |
| `credentialRefId`、`credentialVersion` | 创建时固定的凭据引用与版本；公开仓库为空 |
| `expectedTrackingOid` | `leased → running` 之前经 ref 查询读到的跟踪 ref 值，不存在时为全零 |
| `state` | 见下表 |
| `leaseOwner`、`leaseGeneration`、`leaseExpiresAt` | 租约 |
| `result` | 成功时的 `commitOid`、`forced`（是否非快进）；失败原因在独立的 `error_code` 列（公开为 `errorCode`），不在 `result` 中 |

状态：

```text
pending ──lease──▶ leased ──issue authorization──▶ running ──▶ succeeded
   │                 │                                │──────▶ superseded
   │                 └─lease expired─▶ pending        │──────▶ failed
   └──────────────▶ cancelled                         └─lease expired─▶ failed(lease_lost)
```

- **每个 Binding 同时至多一个未结束的同步**（`pending`、`leased`、`running`），以部分唯一索引保证。已有未结束的同步时，
  新的触发返回已有的 Operation，不另建。
- **租约**：Cloud 的同步 worker 以 `leaseGeneration + 1` 领取 `pending` 行，进入 `leased`；之后每次状态变化都以
  `(id, leaseGeneration)` 做条件更新，代次不符的 worker 写不进任何结果。worker 在调用执行器期间续约。
- **授权只在进入 `running` 时签发**，与状态变化在同一事务内。`leased` 状态下租约过期，回到 `pending` 是安全的：
  此时还没有一次性授权，旧 worker 不能调用执行器。
- **`running` 只发生一次**。`running` 中租约过期，Operation 以 `failed(lease_lost)` 结束，不回到 `pending`。
  契约 D5 第 3 项规定“Operation 结束或租约过期，同一个 ID 立即失效”；如果同一个 ID 可以被再次领取，失联的旧执行器就能借新租约
  写入，Git Server 无从区分。一次只运行一次，`Ora-Git-Operation-Id` 才能等同于一次执行。重试由新的 Operation 承担（D3）。
- **`expectedTrackingOid` 在进入 `running` 之前取得**：持有租约的 worker 在事务外经 ref 查询读取
  `refs/ora/upstreams/<bindingId>/heads/<branch>`，再在 `leased → running`、签发执行授权的同一事务里写入。不在创建时取：
  底座在建 Project 时就建立了同步 Operation，它可能排队很久，期间的值会过时；离执行越近，被旧执行器的迟到写入打乱的窗口越小。
  也不从当前快照记录取值：契约 D5 第 2 项允许记录落后于跟踪 ref，用记录作 expected old 会让之后的每一次同步都 CAS 失败。
- **`superseded`**：执行器推送时以 `stale info` 被拒，跟踪 ref 不等于 `expectedTrackingOid`、也不等于本次 fetch 得到的提交，
  说明有别的写入者在读取之后推进了它（实际只可能是失联的旧 Operation 在租约过期前的写入）。Operation 以 `superseded` 结束，
  不登记；Cloud 在同一事务里自动重放一次（D3）：新 Operation 会重新读取跟踪 ref。跟踪 ref 已经等于本次的提交时按成功处理，
  见执行器根决策的重放规则。Git Server 的 `pre-receive` 拒绝（`hook declined`）不是 `superseded`，而是 `failed(git_server_rejected)`。
- 取消只能发生在 `running` 之前。`running` 中的同步不能取消：推送一旦开始就由 Git Server 的原子性和 CAS 决定结果，取消只会
  让 Cloud 失去结果。
- **Project 的生命周期**：
  - `provisioning` 的 Project 只有两种同步：建 Project 时由底座建立的那一行，以及它按 D3 的自动重放。手动同步、候选凭据同步和
    发布只限 `active` 的 Project。
  - 首次同步以不再重放的终态失败时，main Workspace 的 clone 步骤以 `repository_sync_failed` 结束，内部线在同一事务里把 Project
    置为 `active`（clone 步骤消费快照决策）。之后用户可以提交候选凭据（D5）、手动同步，或者删除 Project；仓库地址或分支填错只能
    删除后重建（D1）。
- **Project 删除**（契约 D5 第 3 项，取消与恢复的逻辑都在底座）：
  - Project 进入删除流程（`deleting`）的事务里，该 Project 尚未进入 `running` 的同步与发布 Operation 一律以
    `cancelled(project_deleting)` 结束，**同时清除 Binding 的补同步标记**：删除中的 Project 不再新建任何同步。已经 `running`
    的照常收尾，它们写入的 ref 随 Project 的仓库一起由删除流程处理；收尾时即使符合 D3 的后继条件也不新建同步（见 D3）。
  - delete_project 等这些 Operation 结束后才开始（内部线在领取 delete_project 时检查）；发布的 `uncertain` 核验有次数上限
    （发布 ADR D5），所以等待总会结束。
  - **删除被回滚**（Node 拒绝 quiesce，Project 从 `deleting` 回到 `active`）：进入 `deleting` 时取消过 `pending` 或 `leased` 的
    同步、或者清除过补同步标记的，恢复准入的同一事务把欠下的同步补回来：该 Binding 没有未结束的同步时，新建一行 `pending`、
    `trigger = deletion_aborted` 的同步；仍有未结束的同步（进入 `deleting` 时已在 `running`、还没收尾）时，改为重新置上补同步
    标记，由它收尾时按 D3 新建后继。被取消的发布不恢复，由用户重新发起。`deletion_aborted` 的同步与其他同步一样由本文的 worker
    领取，签发授权前检查凭据（D7）。

### D3：触发时机

| 触发 | 时机 | 说明 |
| --- | --- | --- |
| `project_created` | 底座在创建 Project 的事务内同时建立 Binding 和一行 `pending` 的同步 Operation（契约落地顺序第 1 步） | 本文的 worker 领取并执行这一行。Project 的创建本身不等待同步完成；main Workspace 的 clone 步骤等待这次同步（D9） |
| `manual` | 有 Project 修改权限的成员在界面或 API 触发 | 带 `Idempotency-Key`；已有未结束的同步时返回它 |
| `after_publish` | 发布 Operation 成功（包括 `uncertain` 核验为成功）且目标分支等于 Binding 的分支 | 由发布 ADR 规定。已有同步处于 `running` 时不能复用它（它可能在发布生效前就 fetch 完了）：发布在 Binding 上记下 `resync_requested`，那个同步登记终态的同一事务里检查并清除它，再新建后继同步（见下文“后继同步”） |
| `credential_candidate` | 管理员为 Binding 提交候选凭据引用 | 用候选引用同步一次，证明读取能力；成功后原子切换（D5） |
| `deletion_aborted` | 删除被回滚、欠下一次同步时，由底座在恢复准入的事务里建立（D2） | 本文的 worker 照常领取执行 |

- **建 run Workspace 时不隐式同步。** Workspace 的代码输入总是当前快照（契约 D5 第 1 项）。隐式同步会让 Workspace 的创建
  依赖外部仓库的可用性和凭据状态，并让同一 Project 的两个 Workspace 因创建时间不同而得到不同的输入，这正是契约要消除的不确定性。
  需要外部的新提交时，用户先手动同步。
- 第一阶段没有定时同步和 webhook 同步，留给后续决策（见“本决策未解决的问题”）。
- 失败的同步不自动重试，以下例外由 Cloud 在登记终态的**同一事务**里为同一 Binding 创建一个 `trigger` 相同的新 Operation：
  - `failed(lease_lost)`、`failed(executor_unreachable)`（Cloud 与执行器的连接在任务中断开）：最多连续三次，避免进程重启或网络
    抖动让同步无声地停下；
  - `superseded`：重放一次。

  次数用完后等用户手动触发。重放是安全的：执行器可能已经写入的快照 ref 和跟踪 ref，会被下一次同步当作空操作或新的 expected old。
  终态与重放在同一事务里，所以任何时刻该 Binding“最近的同步”要么还未结束，要么是不会再重放的终态（D9）。

**后继同步：一个终态事务至多新建一个**（契约 D5 第 3 项）。登记同步终态的事务可能同时需要自动重放和补同步
（`resync_requested`）。部分唯一索引只允许一个未结束的同步，两个都建会让整个终态事务失败，连原 Operation 的终态也落不了库。
所以规则是：

| 重放 | 补同步标记 | 新建 | `trigger` | 标记 |
| --- | --- | --- | --- | --- |
| 需要 | 已置 | 一个 | 沿用被重放 Operation 的 `trigger`（重放优先） | 清除 |
| 需要 | 未置 | 一个 | 沿用被重放 Operation 的 `trigger` | — |
| 不需要 | 已置 | 一个 | `after_publish` | 清除 |
| 不需要 | 未置 | 不建 | — | — |

- 重放优先、同时清除标记是安全的：后继同步在本事务提交之后才 fetch 外部仓库，而置上标记的发布在此之前已经生效，所以它一定能
  看到发布的提交，补同步的目的同样达到。沿用原 `trigger` 让重放保持原来的语义：被重放的是 `credential_candidate` 时，后继仍用
  候选引用、成功后仍切换（D5）。代价是候选引用的重放若最终失败，发布后的补同步不再单独进行；界面显示最近的同步失败，用户手动
  同步即可。
- **系统触发的后继建不了时只记录原因，不回滚**。系统触发指自动重放和补同步，以及发布成功后新建的 `after_publish`（发布 ADR D6）。
  新建前按 D7 检查凭据引用，并检查 Project 不在 `deleting`。条件不满足时不新建、照常清除标记，并把原因记入触发它的那个
  Operation：由同步触发时写入 `result.successorSkipped`，值为 `credential_frozen`、`credential_unavailable` 或
  `project_deleting`；由发布触发时写入 `result.resyncSkipped`，值为凭据检查返回的错误码原文（如
  `repository_credential_frozen`），Project 不是 `active` 时不记录（与批准版本的偏离，见“与批准版本的偏离”）。所在事务照常提交。
  D7 的 `409` 只用于用户经 API 发起的同步：用户可以修正后重试，而系统触发发生在另一个 Operation 的终态事务里，回滚会让那个
  Operation 停在未结束的状态。凭据恢复后由用户手动同步；界面只显示 Binding 的 `resyncRequested`，不显示跳过的原因（D8）。

### D4：公开仓库

`credentialRefId` 为空的 Binding 不兑换任何凭据，执行器匿名 fetch。其余步骤与私有仓库完全相同：同样经一次性授权调用执行器、
同样写快照和跟踪 ref、同样由 Cloud 核验。

- 匿名 fetch 时外部仓库要求认证（HTTP 401，或平台对私有仓库返回的 404），同步以 `failed(repository_requires_credential)` 结束，
  界面提示绑定凭据。不尝试任何部署级的默认凭据（凭据连续性 D4）。
- 公开仓库也必须先同步一次，Workspace 的 clone 步骤才能继续。Node 不再直接访问外部仓库，这一点对公开仓库同样成立（契约落地顺序第 2 步）。

### D5：推进当前快照

执行器报告 `{commitOid, snapshotRef, trackingRef, forced}` 后，Cloud 按契约 D6 的顺序处理：

1. 事务外经 ref 查询读取 `snapshotRef` 和 `trackingRef`。
2. 两者都指向 `commitOid`：在一个事务内以 `(id, leaseGeneration)` 条件把 Operation 置为 `succeeded`，把 Binding 的当前快照
   更新为 `(commitOid, snapshotRef, operationId, now)`；Operation 带凭据时，同一事务记录该引用版本的读取能力已验证（identity-access
   后续决策 D4）。
3. 不一致：Operation 以 `failed(verification_mismatch)` 结束，不更新当前快照。这表示执行器的报告与 Git Server 不符，记录为需要
   排查的错误。
4. ref 查询网络失败：在租约内重试；租约过期则 `failed(lease_lost)`，按 D3 由新 Operation 重放。快照 ref 已经存在，重放的 fetch
   和推送都是空操作，只需再核验一次。

当前快照总是更新为最新一次核验通过的结果，不比较新旧提交的先后（契约 D5 第 2 项）。同一 Binding 同时只有一个同步在运行（D2），
所以“最新一次”没有歧义。

**候选凭据**：`credential_candidate` 同步成功时，同一事务把 Binding 的 `credentialRefId` 切换为候选引用，前提是 Binding 的
`version` 自候选提交以来没有变化（凭据连续性 D3 的版本检查）。失败时不切换，原引用若已冻结则继续冻结。

### D6：同步的内容边界

| 内容 | 第一阶段的处理 | 理由 |
| --- | --- | --- |
| 分支 | 只同步 Binding 的一个分支 | 契约 D5 第 1 项只需要一个快照；多分支留在 namespace 中（`heads/<branch>`） |
| Tag | 不同步（`--no-tags`） | Workspace 的输入不依赖 tag；tag 可被外部改写，引入另一套不可变性问题 |
| 历史 | 完整历史，不做 shallow | shallow 快照会让 Workspace 的 `git log`、`blame`、merge-base 失效，发布时也无法在本地判定快进（发布 ADR D4） |
| LFS | 不下载 LFS 对象，仓库中的指针文件按普通文件同步 | 执行器的 fetch 禁用 LFS 过滤器；LFS 对象需要另一套存储与配额。同步成功，界面不区分指针文件（与批准版本的偏离，见“与批准版本的偏离”） |
| submodule | 不递归；gitlink 按提交指针保留 | 子模块是另一个外部仓库，需要自己的 Binding 和凭据；递归 fetch 会让一个 Binding 的凭据访问别的仓库 |
| 大小 | 执行器对任务目录设字节上限（`REPO_EXECUTOR_MAX_TASK_BYTES`，默认 2 GiB），并有任务超时；没有对象数上限，也不读取 Git Server 的 `receive.maxInputSize` 或仓库配额 | 超限以 `failed(repository_too_large)` 或 `failed(timeout)` 结束，不留半个快照：推送是原子的 |

**外部 force-push**：外部分支被改写时，下一次同步照常写入新的快照 ref；跟踪 ref 按 tracking-cas 允许非快进；当前快照更新为
新提交。执行器在推送前判定新提交是否是 `expectedTrackingOid` 的后代，结果写入 `forced`，界面据此提示“外部分支被改写”。

- 旧快照 ref 不删除（契约不变量 5），已经从旧快照开始的 Workspace 不受影响，它们的 Revision 和交付 ref 也不受影响。
- 基于旧快照发起的发布，`expectedOldOid` 与外部的当前值不一致，以 `conflict` 结束（发布 ADR）。
- **外部分支被删除**：同步以 `failed(upstream_branch_missing)` 结束，当前快照和跟踪 ref 都不变。Ora 不跟随删除：契约禁止删除
  任何 ref，而且已有快照是 Workspace 的输入。

### D7：凭据使用

- 创建同步 Operation 前、签发一次性授权前，各检查一次引用可用：未冻结、未撤销、范围与 `repositoryUrl` 匹配、关联成员仍有效
  （凭据连续性 D2）。用户经 API 发起时不可用返回 `409 repository_credential_unavailable` 或 `409 repository_credential_frozen`；
  系统触发的后继不可用时不新建、只记录原因，不返回错误也不回滚（D3）。已创建的 Operation 在签发前发现不可用，以
  `failed(credential_unavailable)` 结束。
- **删除回滚补回的同步**（D2 的 `deletion_aborted`）在创建时不检查引用：底座在恢复准入的事务里直接建立 `pending` 行，worker 在
  签发授权前发现不可用时以 `failed(credential_unavailable)` 结束（与批准版本的偏离，见“与批准版本的偏离”）。
- 引用被撤销后，新的同步一律拒绝。已经进入 `running` 的同步已兑换过凭据，按凭据连续性 D2 允许收尾。
- 同步只使用 Binding 当前的引用。个人引用被所有者绑定到 Binding，等于所有者同意该 Project 用它读取；读取不需要逐次批准，
  发布需要（发布 ADR D2）。
- 外部仓库以认证失败拒绝时（HTTP 401/403），同步以 `failed(credential_rejected)` 结束，并把该引用版本标为 `invalid`
  （凭据连续性 D3 表的最后一行），等用户更新凭据。

### D8：公开 API 与界面

API（均在 Project 作用域下，按 Project 根决策校验成员身份）：

| 路由 | 语义 |
| --- | --- |
| `GET /projects/{projectId}/repository` | Binding、当前快照、最近一次同步 Operation 的状态与错误码 |
| `POST /projects/{projectId}/repository/syncs` | 手动同步；带 `Idempotency-Key`；已有未结束的同步时返回 `200` 和它 |
| `GET /projects/{projectId}/repository/operations` | 同步和发布 Operation 的列表，按时间倒序分页 |
| `GET /projects/{projectId}/repository/operations/{operationId}` | 单个 Operation 的详情 |

Operation 状态变化后，沿用 [Space 事件流](../api-boundary/0-space-api-and-event-stream.md)只发失效通知，客户端重新读取 REST。

界面（Project 详情页的“代码来源”区块）：

- 显示外部仓库地址、分支和凭据名称（匿名仓库显示“匿名读取”）；不显示凭据归属和读取、发布能力的验证状态（与批准版本的偏离，
  见“与批准版本的偏离”）。
- 显示当前快照的完整 `commitOid`；`forced` 时提示外部历史已改写、旧快照仍保留。不显示快照的来源同步完成时间。
- 显示最近一次同步的状态和错误码，按下表映射；未列出的错误码原样显示。系统触发的后继同步被跳过时（D3），界面只显示 Binding 的
  `resyncRequested`（“发布后的补同步已安排”），不显示 `result.successorSkipped` 的原因（与批准版本的偏离，见文末）。

  | 错误码 | 界面说明 |
  | --- | --- |
  | `repository_requires_credential` | 私有仓库需要凭据 |
  | `credential_rejected` | 凭据被外部平台拒绝，请轮换 |
  | `credential_unavailable` | 凭据不可用，请检查或更换 |
  | `upstream_branch_missing` | 外部分支不存在 |
  | `repository_too_large` | 仓库超过大小限制 |

- “立即同步”按钮对能查看代码来源的成员可见，只在请求进行中禁用；不检查 Project 修改权限、未结束的同步或凭据状态
  （与批准版本的偏离，见文末）。
- 没有当前快照时，Workspace 的创建入口和提示由内部线的消费方负责（见兼容性边界）；本文不实现这部分界面。

### D9：外部线对等待快照的 operation 的保证

等待由内部线负责（契约落地顺序第 1 步）：Workspace 的 clone 步骤在领取时复核，有当前快照就继续；没有快照、而该 Binding 最近的
同步已经结束且没有成功，就以失败结束；否则以 `snapshot_not_ready` 继续等待。底座不提供额外的“结束等待者”函数。

这个规则能结束等待，前提是外部线给出以下保证，本文承担它们：

1. **每个 Binding 从创建起就有同步**：底座在建 Project 的事务里建立一行 `pending` 的同步 Operation，本文的 worker 一定会领取它
   （D3）。所以“没有快照、也从没同步过”的 Binding 不存在。
2. **同步一定落到终态**：`running` 只发生一次，租约过期也以 `failed(lease_lost)` 结束（D2），不会停在中间状态。唯一停不下来的
   情形是同步 worker 整体不可用，Operation 停在 `pending` 或 `leased`；这是基础设施故障，与其他 `retry_wait` 的 operation 一样
   等待恢复，不另设超时。
3. **会重放的失败不会被误读为最终失败**：自动重放（D3）与原 Operation 的终态在同一事务里创建，所以内部线看到的“最近的同步”
   要么还未结束，要么是不会再重放的终态。
4. **最终失败带着原因落库**：首次同步永久失败时，Operation 以 `failed` 终态落库，`error_code` 列中有错误码（公开为 `errorCode`，例如 `credential_rejected`、
   `upstream_branch_missing`、`repository_requires_credential`），内部线和界面据此说明原因。`superseded` 在用完一次重放后同样
   是不会产生快照的终态。

为什么用同步的终态而不是固定的等待时长判断“快照会不会出现”：首次同步的耗时取决于仓库大小，从几秒到几十分钟（执行器根决策 D4）。
固定时长要么让大仓库的首次创建失败，要么让凭据错误这种立刻就能确定的失败也要等满。

结束后怎样重新创建 Workspace，沿用内部线对失败的 create_project / create_workspace 的处理，本文不改变。

## 不变量

1. 每个 Project 第一阶段恰好一个 Binding；`bindingId` 创建后不变。
2. 每个 Binding 同时至多一个未结束的同步 Operation。
3. 一个同步 Operation 至多进入一次 `running`；租约过期后它的 ID 永久失效。
4. 一次性授权只在进入 `running` 的事务内签发。
5. 当前快照只在 Cloud 经 ref 查询确认快照 ref 和跟踪 ref 都指向报告的提交后更新。
6. 同步失败、取消、`superseded` 都不改变当前快照。
7. 外部 force-push 和分支删除不会删除或改写任何已有快照 ref。
8. 不可用的凭据引用不会被用于新的同步；公开 Binding 不使用任何凭据。
9. 创建 run Workspace 不触发同步。
10. Project 的默认分支只存在 `Binding.branch` 一处。
11. 每个 Binding 从创建起就有同步 Operation；每个同步 Operation 都会落到终态。
12. 自动重放与原 Operation 的终态在同一事务中创建。
13. 登记终态的事务至多新建一个后继同步；系统触发的后继建不了时，所在事务照常提交。
14. Binding 只有 `credential_ref_id` 一个凭据引用列。

## 为什么不是这些替代方案

| 方案 | 否决理由 |
| --- | --- |
| 建 run Workspace 时隐式同步 | Workspace 的创建会依赖外部仓库的可用性，同一 Project 的两个 Workspace 输入不确定（D3）；同步失败要回滚 Workspace，牵连内部线 |
| 同步 Operation 复用现有 `operations` 表 | 现有表的 `kind`、`step` 是 Workspace 生命周期的步骤（`storage`、`worktree`、`sandbox`……），由 Controller 推进；同步与发布由 Cloud 自己的 worker 加租约推进，状态集合和终态都不同。混在一起会让 Git Server 的上下文解析与 Workspace 生命周期耦合 |
| 租约过期后重新领取同一个 Operation | 实现更简单，也少一行记录；但同一个 `Ora-Git-Operation-Id` 会先后属于两次执行，失联的旧执行器能借新租约写入，违反契约 D5 第 3 项“租约过期，同一个 ID 立即失效” |
| 以当前快照记录作为 expected old | 记录可能落后于跟踪 ref（契约 D5 第 2 项），会让后续同步永久 CAS 失败；ref 查询取得的值才是 Git Server 的事实 |
| shallow 同步 | 节省传输，但 Workspace 的历史不完整，发布时无法在本地判定快进 |
| 同步 tag 和所有分支 | 契约第一阶段只消费一个快照；全量同步放大数据量和凭据的使用面，namespace 已留出位置 |
| 同步时递归 submodule、下载 LFS | 一个 Binding 的凭据会访问别的仓库或别的存储；需要独立的 Binding 模型和配额 |
| 保留 `projects` 的仓库列，与 Binding 并存 | 不需要改动现有读取方；但同一事实有两处来源，读取方都要决定以哪处为准（D1）。Cloud 没有用户，不需要兼容 |
| 底座提供“结束等待者”的函数，同步收尾时调用 | 等待者结束得更早；但两条线要调用同一个函数，耦合更紧。内部线在领取时复核同步的终态已经足够（D9） |
| `snapshot_not_ready` 设固定的等待上限 | 实现简单，不依赖同步的状态；但上限无法同时适应大仓库和立即可知的失败（D9） |
| 同步失败后让等待者继续等，直到用户手动同步成功 | 修好凭据后不用重新创建 Workspace；但等待期间 Project 的操作槽一直被占，其他操作都返回 busy，而用户可能根本不知道要去修 |

## 与批准版本的偏离

对照 cloud `ceb39bc` 与 cluster `033a4ea`，最终实现与本文批准时相比有以下偏离，正文已按最终实现表述：

| 批准时的决策 | 最终实现 | 原因 |
| --- | --- | --- |
| `result` 同时承载成功字段、失败错误码和核验时间 | 成功字段在 `result`；错误码在 `error_code`（公开为 `errorCode`）；核验时间不在 `result`，由 Operation 的 `finishedAt` 和快照的 `recorded_at` 表示 | 错误码作为独立列，便于查询与状态的 `CHECK` 约束；核验时间由记录的时间戳表示，不重复存储 |
| 发布触发的后继同步跳过原因写入 `result.successorSkipped`，取值 `credential_frozen`、`credential_unavailable`、`project_deleting` | 发布触发时写入 `result.resyncSkipped`，值为凭据检查的错误码原文；Project 不是 `active` 时不记录 | 同步与发布两条触发路径分别实现，未统一键名和错误码；没有测试覆盖这条路径 |
| 单次 fetch 的数据量、对象数和耗时上限与 Git Server 的 `receive.maxInputSize` 和仓库配额一致 | 只有任务目录字节上限（默认 2 GiB）和任务超时；没有对象数上限，也不读取 Git Server 配置 | 代码注释把 Git Server 的上限列为第二道检查，但没有实现联动 |
| 界面在检测到 `filter=lfs` 时提示“LFS 文件只有指针” | 不提示，指针文件与普通文件没有区别 | 界面未实现该项，没有记录原因 |
| `deletion_aborted` 同步在创建前检查凭据 | 创建时不检查，签发授权前检查，失败以 `failed(credential_unavailable)` 结束 | 补回同步的底座路径直接插入 `pending` 行，可用性检查放在 worker 启动时 |
| 界面显示凭据类型与能力、短 OID 与同步完成时间、后继跳过原因、`repository_credential_frozen` 的说明，并按权限、未结束同步和凭据状态控制“立即同步” | 界面只显示凭据名称、完整 OID、状态和映射后的错误码；“立即同步”只在请求进行中禁用；`repository_credential_frozen` 原样显示 | 本阶段只实现了读取与操作入口，未实现这些展示和禁用规则；没有记录原因 |
| 等待中的 Workspace 显示“等待首次同步”，或禁用创建入口并说明“需要先完成一次同步” | cloud 与 cluster 都没有这些提示和禁用逻辑 | 等待由内部线的消费方负责，内部线尚未实现（见兼容性边界） |

## 本决策未解决的问题

- **多 Binding、多分支、tag**：namespace 已经允许，模型与界面留给后续决策。
- **定时同步与 webhook**：需要外部平台的回调入口和去重，第一阶段只有手动和事件触发。
- **SSH 地址和 GitHub App 身份**：第一阶段只有 `https://` 加 token。凭据类型的扩展见 identity-access 后续决策。
- **LFS 与 submodule 的真正支持**：需要对象存储配额与子仓库的 Binding。
- **快照回收**：与契约一致，第一阶段全部保留。
