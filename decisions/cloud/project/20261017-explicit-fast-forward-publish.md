---
status: implemented
---

# 外部写入只经显式 Publish：经批准的 PublishOperation 以 expected old 快进推送，结果不明时只读核验、不重推

Workspace 的推送只到达 Git Server，永远不会触达外部仓库。要把代码写回外部仓库，用户必须显式创建一个 PublishOperation，写明
源 ref、源提交、目标分支和期望的外部旧值 `expectedOldOid`，并由凭据决定的审批人批准：个人凭据只有所有者本人能批准，团队凭据
由拥有 Project 发布权限的人批准。批准后 Cloud 调用 [Repo Executor](../repo-executor/0-repo-executor-sole-external-git-boundary.md)
从 Git Server 读取源提交，确认是快进，以 `ls-remote` 比对旧值，再以 `--force-with-lease` 推送并再次核验。结果不明时进入
`uncertain`，由 reconciler 以只读的 `ls-remote` 判定，从不重推。发布成功后自动同步一次，让当前快照反映外部仓库。

本 ADR 处于 `implemented`：实现已由 cloud#76（`ceb39bc`）与 cluster#27（`033a4ea`）合入 main；核心测试用例见[显式发布用例](../../../test-cases/cloud/project/explicit-fast-forward-publish.md)，是 `cloud/project` 领域在[同步 ADR](20261016-repository-binding-and-snapshot-sync.md)之后的后续决策。
它按评审中推荐的方案确定只快进发布和审批人，遵守 [Git Server 代码中心契约](../git-server/20261016-project-repository-as-code-hub-and-ref-contract.md)
（下称“契约”）D5 第 3 项：发布 Operation 在 Git Server 上没有写权限，执行器只取 Operation 记录的源 ref 与源提交。涉及 Cloud（API、审批、状态机、reconciler、
审计、前端）与 Repo Executor。凭据能力由 [identity-access 后续决策](../identity-access/20261016-git-credentials-in-independent-credential-store.md)
规定。

兼容性边界：

- 新增 API 与状态；不改变 Workspace 推送、Revision 交付和同步的任何行为。
- 破坏性切换（契约范围边界）：不设过渡开关，不做数据迁移。
- 未合并的旧实现中的 PushRequest（`/workspaces/{wid}/pushes`）不沿用。
- 不支持非快进发布、删除外部 ref、发布 tag（见“本决策未解决的问题”）。

## 继承与修改

| 前序约定 | 本文处理 |
| --- | --- |
| 同步 ADR D1：每个 Project 一个 Binding，带外部仓库与分支 | 继承，不变。发布的外部仓库就是 Binding 的仓库 |
| 同步 ADR D2：`repository_operations` 表、租约与代次、一次授权只在进入 `running` 时签发、`running` 只发生一次 | 继承。发布 Operation 是同表中 `type = repository_publish` 的行；`running` 中租约过期的处理不同：进入 `uncertain`（D3） |
| 同步 ADR D3：`after_publish` 触发 | 继承，由本文 D6 规定触发条件 |
| 同步 ADR D6：外部 force-push 后旧快照保留 | 继承。基于旧快照的发布以 `conflict` 结束（D4） |
| [凭据连续性](20260927-repository-credential-continuity.md) D2：凭据不可用时拒绝新的远程操作；已开始的任务可以收尾 | 继承。批准与派发前各检查一次；`running` 中的发布可以收尾（D2、D3） |
| [运行时使用权限](../workspace/20260927-runtime-workspace-use-permissions.md)：运行时内容只供创建者与租户管理员使用 | 契约已在 Git 层面修改：同一 Project 内的 Workspace 分支与交付 ref 可以作为发布源。本文据此不按 Workspace 的创建者或管理员校验发布源，只校验发起人与审批人（D1、D2） |
| identity-access 后续决策 D4、D5：`publish` 能力必须验证；个人凭据只有所有者能批准 | 继承，不变（D2） |
| 契约 D4：Workspace 分支只能快进；交付 ref 不可变 | 继承。因此源提交总能从源 ref 取到（D1） |

## 决策

### D1：API 与字段

```text
POST /projects/{projectId}/repository/publishes        Idempotency-Key 必填
{
  "sourceRef":      "refs/heads/workspaces/<workspaceId>/<branch>",
  "sourceCommit":   "<40 或 64 位小写 OID>",
  // 或者只给 "revisionId": "<id>"，直接发布一次 run 的 Revision（最终实现不支持，见下表）
  "targetBranch":   "main",
  "expectedOldOid": "<40 或 64 位小写 OID，或全零表示新建分支>",
  "credentialRefId": "<可选，缺省用 Binding 的凭据>"
}
```

| 字段 | 校验 |
| --- | --- |
| `sourceRef` | 本 Project 仓库中任意 Workspace 的分支 `refs/heads/workspaces/<workspaceId>/<branch>`，或任意交付 ref `refs/ora/deliveries/<deliveryWorkId>`；不按 Workspace 的创建者校验（契约 D3 的修改）。Cloud 经 ref 查询确认它存在 |
| `revisionId` | 与 `sourceRef`、`sourceCommit` 二选一。最终实现中 Revision 记录保存的是逻辑 ref `refs/ora/revisions/<runId>`，不是交付 ref；发布只接受 `refs/ora/deliveries/<deliveryWorkId>`，所以带 `revisionId` 的发布总是以 `409 revision_not_publishable` 结束。客户端必须直接给出 `sourceRef` 与 `sourceCommit`。这是实现缺陷，见“与批准版本的偏离” |
| `sourceCommit` | 完整 OID。Cloud 只能查到 ref 的当前值，无法判定祖先关系，祖先关系由执行器判定（执行器根决策 D6 第 2 步） |
| `targetBranch` | 具体分支短名，组成 `refs/heads/<targetBranch>`；不能是 `HEAD` |
| `expectedOldOid` | 完整 OID 或全零；等于 `sourceCommit` 时以 `400 nothing_to_publish` 拒绝 |
| `credentialRefId` | 必须作用于 Binding 的仓库，`secretState = stored`，`available`，能力集合包含 `publish`；否则 `409 credential_publish_unverified` 等明确错误 |

- **发起人**：拥有 Project 发布权限的人，或所用个人凭据的所有者。其他成员不能发起。
- 界面在目标分支等于 Binding 分支时，用 Binding 的当前快照预填 `expectedOldOid`：Workspace 就是从它开始的，这正是用户期望覆盖的
  外部状态。目标是新分支时预填全零。
- 字段在创建后不可变。批准针对的是这一组字段，任何改动都要新建 Operation。
- 同一 `Idempotency-Key` 与相同请求体返回同一个 Operation；请求体不同返回 `409 idempotency_conflict`。

### D2：审批人

| 凭据归属 | 谁能批准 |
| --- | --- |
| 个人 | 只有凭据的所有者本人，且其仍是租户的有效成员 |
| 团队 | 拥有 Project 发布权限的人；第一阶段即租户 `admin`（租户成员角色只有 `admin` 和 `member`） |
| 未知 | 不能用于发布：未知归属无法确定审批人 |

- 发起人本身就是审批人时，创建即批准，直接进入 `pending`；否则进入 `awaiting_approval`，等待审批人批准或拒绝。
- 租户管理员**不能**批准使用他人的个人凭据（identity-access 后续决策 D5）。所有者离开后，个人凭据冻结，等待中的发布在派发前
  检查失败；要继续发布，管理员改绑团队凭据后重新发起。
- 批准记录审批人、时间，以及批准时的字段与凭据版本。`approval_expires_at` 决定批准的有效期：待批准请求在创建时写入创建后
  24 小时；人工批准时重设为批准后 24 小时；自批准（创建即批准）从创建时刻起计时 24 小时（与批准版本的偏离：人工批准会
  重置有效期，见文末）。过期的 `awaiting_approval` 和 `pending` 都以 `cancelled(approval_expired)` 结束。
  凭据在批准后提交了新版本，等待中的发布以 `cancelled(credential_changed)` 结束：审批人批准的是用旧 token 写入。
- 发起人可以在 `awaiting_approval` 和 `pending` 时取消；审批人可以拒绝，以 `cancelled(rejected_by_approver)` 结束。

为什么不让管理员兜底批准：个人凭据代表所有者在外部平台的身份。管理员批准等于以别人的身份写入外部仓库，外部平台的审计会把
这次写入记在所有者名下。把团队凭据作为唯一的兜底，外部平台上的身份与 Ora 中的批准人就总是一致的。

### D3：状态机

```text
awaiting_approval ──approve──▶ pending ──lease──▶ leased ──issue authorization──▶ running
        │                        │                  │                               │
        └──reject/cancel/expire──┴──cancel──────────┼──▶ cancelled                   ├─▶ succeeded
                                                    └─lease expired─▶ pending         ├─▶ conflict
                                                                                     ├─▶ rejected
                                                                                     ├─▶ failed
                                                                                     └─▶ uncertain ──reconcile──▶ succeeded | conflict | failed
```

| 状态 | 含义 | 外部仓库可能已被改变 |
| --- | --- | --- |
| `awaiting_approval` | 等待审批人 | 否 |
| `pending` | 已批准，等待 worker 领取 | 否 |
| `leased` | worker 持有租约，尚未签发授权 | 否 |
| `running` | 已签发 `publish` 授权，执行器在运行；Git Server 只在此状态下接受它读取 Project 仓库 | 可能 |
| `succeeded` | 外部的目标分支已经是 `sourceCommit` | 是 |
| `conflict` | 外部的目标分支不等于 `expectedOldOid`，没有写入 | 否 |
| `rejected` | 外部平台明确拒绝了写入（保护分支、hook、权限），或本地判定不是快进 | 否 |
| `failed` | 推送开始前的确定失败（源不可用、凭据不可用、认证失败、超时、执行器容量已满）；或核验确认推送没有生效 | 否 |
| `cancelled` | 在 `running` 之前结束 | 否 |
| `uncertain` | 推送可能已经发出，结果不明 | 可能 |

执行器分类到状态的映射：

| 执行器分类（执行器根决策 D8） | 状态 |
| --- | --- |
| `applied` | `succeeded` |
| `conflict` | `conflict` |
| `rejected`、`not_fast_forward` | `rejected` |
| `source_unavailable`、`failed` | `failed` |
| `uncertain` | `uncertain` |

- 每个 `(bindingId, targetBranch)` 同时至多一个处于 `leased` 或 `running` 的发布。`awaiting_approval`、`pending` 和 `uncertain`
  不占用：前两者还没有触达外部，`uncertain` 可能长期无法判定，让它阻塞后续发布会让一次网络故障永久卡住一个分支。并发的安全性
  由 `expectedOldOid` 的比较并交换保证，不依赖互斥。
- `running` 中租约过期（worker 失联）或 Cloud 与执行器的连接在任务进行中断开，一律进入 `uncertain`，不回到 `pending`：授权已经
  签发，执行器可能已经推送。
- 只有 `running` 之前可以取消。

### D4：只做快进

- `expectedOldOid` 非全零时，它必须是 `sourceCommit` 的祖先，否则以 `rejected(not_fast_forward)` 结束，不访问外部仓库的写入接口。
- `expectedOldOid` 全零表示新建分支：外部的目标分支必须不存在。
- 外部仓库在用户发起之后前进了（别人推送、外部 force-push、上一次发布已生效），`expectedOldOid` 不再等于外部的当前值，
  以 `conflict` 结束。界面提示：先同步，在 Workspace 中合并或变基到新的快照，再重新发布。
- 不支持删除外部 ref、非快进发布和发布 tag。

为什么快进可以只在本地判定：快进意味着 `expectedOldOid` 在源提交的历史中，而源提交的历史已经从 Git Server 取回；如果
`expectedOldOid` 不在其中，无论外部仓库里有什么，这次写入都不是快进。所以执行器不需要从外部下载任何对象。

### D5：`uncertain` 的核验

Cloud 的 reconciler 处理 `uncertain` 的发布：

1. 检查凭据仍可用，签发 `probe` 授权（`service = upload-pack`），调用执行器读取外部的目标分支。
2. 按观察值判定：

   | 外部目标分支 | 结论 |
   | --- | --- |
   | 等于 `sourceCommit` | `succeeded`，标注“由核验确认”；外部可能是别人推送了同一提交，但用户要的状态已经达成 |
   | 等于 `expectedOldOid` | `failed(not_applied)`：推送没有生效。不自动重推，用户可以重新发起 |
   | 其他值 | `conflict(moved_after_uncertain)`：外部已被改动，本次推送是否曾经生效无法判定；审计记录观察值 |

3. 读取失败（网络、平台故障）或凭据不可用时保持 `uncertain`，按指数退避重试，最长间隔 1 小时；界面显示状态“正在核验”和
   “立即核验”按钮，但不显示最近一次失败的原因（与批准版本的偏离，见“与批准版本的偏离”）。
4. **核验次数有上限**（契约落地顺序第 1 步）：自动核验最多 20 次（按上述退避约跨 16 小时），用尽后以 `failed(outcome_unknown)` 终态
   结束，审计记录每次核验的失败原因，界面提示“无法确认外部仓库是否已更新，请同步后查看”。上限保证 `uncertain` 不会永远是
   未结束的 Operation，删除 Project 等待它们时总会结束（同步 ADR D2）。状态枚举由底座定全，所以用 `failed` 加错误码表达，
   不新增状态。
5. 第一次核验不早于执行器任务的截止时间（Cloud 在任务请求中给出，执行器根决策 D4）。这样核验时执行器已经不可能再发起推送。

等于 `expectedOldOid` 时为什么可以判定未生效：核验发生在执行器任务的截止时间之后（第 5 条），执行器已经不可能再发起推送；
外部平台对单个 ref 的更新是原子的，推送若已生效，目标分支不会停留在旧值上，除非之后又被别人改回。后一种情况的结果对用户而言
同样是“外部没有我的提交”，判为未生效、让用户重新发起是安全的。

为什么从不重推：重推的时刻可能远晚于批准，外部状态和批准时不同；核验只读，永远不会产生用户没有预期的外部写入。
未合并的旧实现在租约过期后重新派发同一推送，本文不沿用（执行器根决策 D8）。

### D6：发布后自动同步

发布以 `succeeded` 结束（包括核验确认的成功），并且 `targetBranch` 等于 Binding 的分支时，Cloud 在同一事务内为该 Binding 触发
一次 `after_publish` 同步（同步 ADR D3）：

- 该 Binding 没有未结束的同步：新建一个。
- 已有同步处于 `pending` 或 `leased`：复用它。它还没有 fetch 外部仓库，fetch 时发布已经生效。
- 已有同步处于 `running`：它可能在发布生效之前就 fetch 完了，不能复用。发布在 Binding 上置 `resync_requested = true`；那个同步
  登记终态的同一事务里检查并清除该标记，按同步 ADR D3 的“后继同步”规则至多新建一个后继：同时需要自动重放时只建重放，它在发布
  生效之后才 fetch，同样包含发布的提交。这样“发布成功”之后，当前快照最终会包含发布的提交（除非外部仓库随后又被改写，或后继
  同步失败）。
- 新建同步的条件不满足时（凭据已冻结或不可用，Project 正在删除），不新建，把原因记入本发布 Operation 的
  `result.resyncSkipped`：凭据不可用时是凭据检查的错误码原文（如 `repository_credential_frozen`）；Project 不是 `active` 时不
  记录。发布照常以 `succeeded` 落库，不回滚（同步 ADR D3；与批准版本的偏离，见文末）。

- 同步结果通常是 `sourceCommit`，但外部仓库可能在发布后又前进了，当前快照总是以同步得到的为准（契约 D5 第 2 项：外部内容只经
  同步进入当前快照，发布不直接改写它）。
- 自动同步失败不影响发布的结果。
- 目标是其他分支时不同步：第一阶段只同步 Binding 的分支。

### D7：审计

每个发布 Operation 及其事件追加记录，不修改历史：

- 发起人、审批人、批准与拒绝的时间、取消原因；
- 凭据引用、`storeVersion` 和审批人类型（`approverKind`）；归属本身不记录（与批准版本的偏离，见文末）；
- `sourceRef`、`sourceCommit`、`targetBranch`、`expectedOldOid`；
- 前置与后置 `ls-remote` 观察到的值、执行器分类、最终状态与错误码；
- 每次核验的时间与观察值；
- 外部平台的拒绝原因：先脱敏（去掉凭据与 URL 中的用户信息），再截断到 2 KiB。

审计不包含 token、平台的完整原始输出或 Workspace 的文件内容。

### D8：界面

- **发起**：Project 详情页的“代码来源”区块有“发布到外部仓库”按钮，没有按分支的入口。对话框列出 Workspace 的分支（经 ref 查询按
  前缀 `refs/heads/workspaces/<workspaceId>/` 取得），可以选择源分支、填写 Revision ID（最终实现会拒绝，D1），并显示源提交、
  目标分支（默认 Binding 的分支，可改为新分支名）、可编辑的完整 `expectedOldOid`（标签“基于外部提交（全零表示新分支）”）、只列出
  已验证 `publish` 的凭据，以及根据凭据算出的审批提示。发起人就是审批人时按钮写作“发布”，否则写作“请求发布”。
- **审批**：Project 详情页的“发布”列表显示每个 Operation；没有单独的审批人待办提示。审批人看到源 ref、完整 `expectedOldOid`、
  凭据版本、发起人和截断到 12 位的源提交；不显示凭据名称与归属，也不显示源提交与 `expectedOldOid` 之间的提交数
  （与批准版本的偏离，见文末）。
- **状态与结果**：发布列表显示状态、目标分支、截断的源提交和错误码；`succeeded` 时注明“结果由核验确认”。错误码按固定的对照表
  映射，`external_rejected`、`cancelled_by_requester`、`rejected_by_approver` 和未列出的错误码原样显示；没有为每个状态单独写
  结果说明（与批准版本的偏离，见文末）。

## 不变量

1. 外部仓库只经 `running` 状态的发布 Operation 被写入；Workspace 的推送不会触达外部仓库。
2. 每个发布在进入 `pending` 之前都由凭据决定的审批人批准；租户管理员不能批准使用他人的个人凭据。
3. 批准绑定 Operation 的全部字段和凭据版本；任何变化都需要新的 Operation。
4. 发布只做快进，以 `expectedOldOid` 作为外部 ref 的比较值。
5. 发布 Operation 在 Git Server 上没有写权限。
6. 一个发布 Operation 至多进入一次 `running`；`running` 中失去租约或连接进入 `uncertain`。
7. `uncertain` 的核验只读；系统从不自动重推。
8. 同一 `(bindingId, targetBranch)` 同时至多一个 `leased` 或 `running` 的发布。
9. 发布不直接改写当前快照；成功发布到 Binding 分支后触发一次同步。
10. 审计中没有 token 和平台的原始输出。

## 为什么不是这些替代方案

| 方案 | 否决理由 |
| --- | --- |
| Workspace 推送时自动同步到外部 | 违反契约需求 4；Agent 的每次推送都会成为外部写入，无法审批 |
| 租户管理员也能批准使用个人凭据（Q7.2 备选） | 所有者离开后仍能发布，减少改绑的麻烦；但外部平台上的写入者与 Ora 的批准人不一致（D2） |
| 发起即发布，不设审批 | 对“发起人就是审批人”的情况等价，本文已经这样处理；其余情况会让任何成员用别人的凭据写外部仓库 |
| 允许非快进发布（Q5 备选） | 需要另行设计改写的审批与告警；第一阶段的 Workspace 分支也只能快进 |
| `uncertain` 时重推（旧实现） | 可能在用户不知情时再次发布（D5） |
| 用 `uncertain` 互斥同一目标分支 | 一次网络故障就能永久阻塞一个分支；比较并交换已经保证并发安全 |
| 复用旧实现的 PushRequest | 只能移动到外部已有的提交，无法发布 Workspace 的内容（执行器根决策 D6） |
| Cloud 在创建时判定快进 | Cloud 只能查询 ref 的值，不读取对象；判定放在已经取回对象的执行器里，不需要给 Cloud 读对象的能力 |

## 与批准版本的偏离

对照 cloud `ceb39bc` 与 cluster `033a4ea`，最终实现与本文批准时相比有以下偏离，正文已按最终实现表述：

| 批准时的决策 | 最终实现 | 原因 |
| --- | --- | --- |
| `revisionId` 从 Revision 的登记记录取出交付 ref 和最终提交 | Revision 记录的 `revision_ref` 是 `refs/ora/revisions/<runId>`；发布要求 `GitDelivery`，所以所有带 `revisionId` 的发布以 `409 revision_not_publishable` 结束 | 实现缺陷：Revision 记录没有保存交付 ref，发布侧又只接受交付 ref；需要让 Revision 暴露实际交付 ref 才能恢复这个入口。界面和公开契约仍然列出 `revisionId`，且没有测试覆盖 |
| 批准在 24 小时内有效，计时从批准时刻开始 | 待批准请求在创建时写为创建后 24 小时；人工批准时重设为批准后 24 小时；自批准从创建（即批准）时刻计时。过期的 `awaiting_approval` 和 `pending` 都以 `cancelled(approval_expired)` 结束 | 实现给待批准请求一个创建后 24 小时的窗口，并在人工批准时重新计时；自批准没有单独的批准动作，从创建时刻计时 |
| OID 为 40 位小写十六进制 | 接受 40 或 64 位小写十六进制 | 支持 SHA-256 仓库；校验沿用 `GitObjectID` |
| 发布触发的后继同步跳过原因写入 `result.successorSkipped`，取值 `credential_frozen`、`credential_unavailable`、`project_deleting` | 写入 `result.resyncSkipped`，值为凭据检查的错误码原文；Project 不是 `active` 时不记录 | 与同步 ADR D3 同一条偏离：两条触发路径未统一键名和错误码 |
| `uncertain` 时界面显示最近一次失败的原因 | 只显示状态和“立即核验”按钮，不显示 `lastCheckFailure` | 界面未实现该项，没有记录原因 |
| 审计记录凭据归属 | 记录凭据引用、`storeVersion` 和审批人类型（`approverKind`），不记录归属 | 归属可由凭据引用查询；审计写入时没有复制 |
| 发起、审批和结果界面按 D8 的完整描述实现 | 没有按分支的发布入口、短 OID、提交数、审批人待办和逐状态结果说明；凭据只显示版本 | 本阶段只实现了发布表单和操作按钮；没有记录原因 |

## 本决策未解决的问题

- **非快进发布与受控改写**：与 Workspace 分支的非快进改写一起设计（契约未决项）。
- **按 Project 细分的发布权限**：第一阶段 Project 发布权限即租户 `admin`；更细的角色需要成员授权表，留给后续决策。
- **发布到 Binding 之外的仓库**：第一阶段只能发布到 Binding 的仓库。
- **创建外部 PR**：需要平台 API，不是 Git 协议的一部分。
- **纵深安全加固**（契约“本决策未解决的问题”，第一阶段以跑通与正确为目标）：
  - 审批界面不展示 diff，审批人只看到字段、凭据和提交数；
  - 不限制目标分支：可以发布到外部仓库的任意分支，包括会触发外部 CI 的分支；
  - 因此发布的内容可能修改外部仓库的 workflow，借外部 CI 读取该仓库的 secrets。展示 diff、限制目标分支、对 workflow 文件的改动
    单独提示，留给后续决策。
