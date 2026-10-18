---
status: proposed
---

# 确定性不可恢复的凭据 Clone 拒绝结算为 Operation 终态：新增 `failed` 转移，fail-closed 不变

当 `requireRepositoryAccess` 以一条**确定性**理由拒绝一次 clone 派发时，这个拒绝就是被拒 Operation 的
最终事实，而不是一次等待重试的冲突。今天有三条这样的理由，它们的共同点是「**关于持久状态，而不是关于
这一次尝试**」：

| 拒绝码 | 为什么不可能改变 |
|---|---|
| `repository_credential_executor_unavailable` | 执行器边界在启动时一次性安装；本部署没有安装，就没有任何重试、任何后续尝试、任何其他 Controller 能改变这个答案 |
| `repository_credential_capability_unsupported` | 凭据的 `capabilities` 在凭据创建时写入且没有更新路径；引用不允许这次 clone，就不会因为再派发一次而允许 |
| `repository_credential_revoked` | 所有者的撤销把 `availability` 写成 `invalid`（`unavailable_reason='revoked_by_owner'`，`revoked_at` 非空），并在凭据服务里销毁了 Token；产品没有解冻、没有恢复、没有写回 `available` 的路径，因此再派发一次只会得到同一个答案 |

第三条与本文的其余部分有一处结构差别，必须显式写出：它是一种**状态被分开报告**——此前
`availability` 只要不是 `available` 就统一报 `repository_credential_frozen`，本文把它拆成"冻结"与
"撤销"两个码（见 D1 与"继承与修改"），因为只有后者是终态。**冻结仍然不结算**，本文不改变它的重试语义。

本文新增一条受控的终态转移，把这类拒绝结算为被拒 Operation 的最终事实：

- 该 Operation 进入 `state = 'failed'`，`errorCode` = **该次拒绝自己的码**（逐字相同），`retry_at` 清空；
- 除这一条转移及其适用范围外，**其余一切不变**：守卫继续拒绝每一次带凭据的 clone，fail-closed 不解除，
  其他错误码的重试行为不变，公开 API 形状不变，匿名 clone 链路不变。

适用面由一张**具名清单**（`terminalCredentialRefusals`）穷举，而不是由"像不像永久失败"的推断决定；
清单里只有上表三码，新增成员必须同时满足"关于持久状态"与"守卫的答案不随重试改变"两条。

**本文的 `status` 是 `proposed`，不是 `approved`。** 它修改了
[运行时 Workspace 生命周期根决策](0-workspace-runtime-follows-desktop-node.md) 已批准的步骤与状态机
（新增一条终态转移），属于"涉及已批准决策冲突、要求审批"的 ADR，因此按规范先提交待批准状态，**不自行
标记 `approved`，也不在本文内维护核心测试用例**——核心测试用例在评审通过后才依据最终同意的方案补充
（见"落地顺序"）。

涉及模块：Cloud `internal/core` 的凭据准入（`repository_credentials.go`）、控制动作入口
（`control.go`）与 clone 派发（`workspace_clone.go`、`control.go:claim`）。兼容性边界：`operations`
的 `state`／`error_code`／`retry_at`／`version` 列语义、`one_project_operation` 部分唯一索引的判定、
公开 `Operation` 契约、匿名 clone 链路，都按硬边界处理。本文**不承担**的迁移义务：不新增 migration，
不改写任何已应用迁移；不改变 `credential_refs` 的冻结与归属语义（撤销的分报不新增或改写任何列，也不改变
谁写这些列）；不实现 Node 认证通道；不决定凭据
能力的**词汇表**（哪些能力存在、是否新增 Push 或候选验证能力，见
[仓库凭据能力词汇表](../identity-access/20261012-repository-credential-capability-vocabulary.md)）；
不解除 fail-closed（解禁条件见 [用户自助 HTTPS Git 凭据](../identity-access/20261010-user-managed-git-credentials.md)
D11-B）。

## 继承与修改

| 前序决策 | 处理 |
|---|---|
| [运行时 Workspace 生命周期根决策](0-workspace-runtime-follows-desktop-node.md)（implemented）D1：`create_project` 为 sandbox → node → clone → done，clone 成功后开放准入并进入 `ready` | **修改**：新增一条终态转移——clone 步骤被**确定性**拒绝时 Operation 进入 `failed`，而不是留在 `running` 等待下一次派发。步骤链本身不变 |
| 同上：clone 失败进入 `retry_wait`/`blocked`，未知保持 `blocked` | 继承，不变；本文只覆盖"守卫拒绝"这一种情况，Node 上报的 `clone_failed`、`clone_result_unknown` 与 defer 码的重试行为逐条不变 |
| [具体默认分支决策](20260926-require-concrete-default-branch.md)（approved）：isolated Workspace 的 `baseRef: HEAD` 指 Project 默认分支 | 继承，不变 |
| [plugin 步骤与运行 Workspace 决策](20260928-plugin-step-and-run-workspace-release.md)（approved，本叶子形式前序）：步骤表与 defer 规则 | 继承，不变；本文不改变任何步骤的进入、推进或 defer 条件 |
| [仓库凭据归属连续性补录](../identity-access/20261010-repository-credential-ownership-continuity.md)（proposed）D7、不变量：`repository_credential_executor_unavailable` 继续有效 | 继承，不变；本文**加强**它的可结算性，不改变该码的判定条件与顺序 |
| [仓库凭据连续性](../project/20260927-repository-credential-continuity.md)（approved）D4：执行端真正按引用选择，执行计划固定引用身份、版本、仓库与所需能力，**能力不支持时明确拒绝** | **修改（实现侧已先行）**：守卫新增一条能力条款，以新码 `repository_credential_capability_unsupported` 拒绝"引用存在、范围匹配、但不允许本次操作"的情况。这是守卫**多拒绝一种输入**，不是多放行一种输入；新增该条款的理由、被拒绝的取值与它对既有取值的影响见 D5 |
| [用户自助 HTTPS Git 凭据](../identity-access/20261010-user-managed-git-credentials.md)（proposed）D2：`capabilities` 第一版只有 `["clone"]`，不蕴含 fetch／pull／push | 继承，不变；本文不扩能力词汇表，只把"引用未声明 `clone`"这一既有事实变成可结算的拒绝。词汇表是否扩展由[仓库凭据能力词汇表](../identity-access/20261012-repository-credential-capability-vocabulary.md)裁决 |
| [用户自助 HTTPS Git 凭据](../identity-access/20261010-user-managed-git-credentials.md)（proposed）D5：撤销沿用 `0023` 语义、映射到 `availability` 的 `frozen` 或 `invalid`，"撤销只关闭新授权"；其"失败与恢复语义"表把"Clone dispatch 时凭据已冻结／已撤销"一律记为 `409 repository_credential_frozen` | **修改（实现侧已先行）**：撤销不再借用冻结的码，而以新码 `repository_credential_revoked` 拒绝并**结算**。D5 已把两种状态分开持久化（`availability` 的 `frozen` 与 `invalid` 是两个取值，`unavailable_reason` 记录原因），而只有 `invalid` 没有回写路径；报成"冻结"既让用户等待一个不会发生的解冻，又使这条拒绝不可能成为终态（冻结必须继续重试，见 D5）。D5 的其余结论逐条不变：撤销仍然只关闭**新**授权，Ora 的撤销不等于吊销上游 PAT，已完成的执行、结果结算、restart 与 cleanup 不受影响 |
| [用户自助 HTTPS Git 凭据](../identity-access/20261010-user-managed-git-credentials.md)（proposed）D6／D11-B：Node 认证通道落地前保持 fail-closed，解禁需要 B 组全部证据 | 继承，不变；本文不构成任何解禁依据，也不改变 B 组义务 |
| [凭据执行器归属决策](../../node/repository/20261012-credential-executor-belongs-to-the-current-product.md)（proposed）：执行能力归属当前正式产品的独立组件，`SetCredentialedCloner` 是部署装配点 | 继承，不变；本文只处理"该装配点未被装配"时的 Operation 归宿 |

本叶子此前的最后一篇决策是 [plugin 步骤与运行 Workspace 决策](20260928-plugin-step-and-run-workspace-release.md)，
因此它是本文的形式前序；本文只新增一条转移，不修改该决策的任何结论。

## 背景与现状

### 缺陷：被拒的派发不是 Operation 的终点

`clone_dispatch` 的 Clone 准入只有一处，在 `workspaceCloneDispatch`（`internal/core/workspace_clone.go`）：
它先校验 `step='clone'`、`state='running'` 与 `controller_epoch`，再调用 `requireRepositoryAccess`。守卫按
固定顺序判定，其中给出**确定性**拒绝的有三处。

前缀校验里的一处，与被拒的绑定状态有关：

- 引用的 `availability` 不是 `available` 时按其真实取值分报：`frozen` 报
  `reject(409, "repository_credential_frozen")`，`invalid`（所有者撤销，
  `unavailable_reason='revoked_by_owner'`）报 `reject(409, "repository_credential_revoked")`。两者都先于
  成员、scope 与部署级的判定：引用自身的状态是"这次 clone 还能不能被授权"的前提，而部署级的答案对一个
  带楔子的 Project 不携带信息——一个没有执行器的部署对**任何**引用都拒绝，用它回答一个已被撤销的绑定，
  等于让用户去修一个不是原因的原因。

尾部（权限校验全部通过之后）的两处，按顺序：

- 部署未安装执行器时 `reject(409, "repository_credential_executor_unavailable")`；
- 引用没有声明本次操作所需的 `clone` 能力时
  `reject(409, "repository_credential_capability_unsupported")`。后者位于前者**之后**：一个没有执行器的
  部署对任何引用都拒绝，报告"引用的能力不支持"会把部署说成引用的问题，而那个部署级的拒绝正是
  fail-closed 决策固定并结算的那一个。

三者都是关于持久状态的判断（执行器边界在启动时安装一次；`capabilities` 在凭据创建时写入且没有更新
路径；撤销没有回写路径），因此都落在本文的适用面内，且都沿同一条路径产生同一个缺陷：

`reject` 会 panic，`transact` 捕获后 `defer tx.Rollback()`，因此这次派发**什么也没写**：Operation 仍停在
`state='running'`、`step='clone'`。Controller 把这个 409 读作可重试的冲突，下一轮再派发、再被拒，
**永远如此**。这不是无害的空转：

- `one_project_operation` 是一条 `WHERE state IN ('queued','running','retry_wait','blocked')` 的部分唯一
  索引，`running` 在其中。因此该 Project 的**唯一** Operation 槽位被永久占用，Project 停在
  `provisioning`，用户在界面上永远看到"创建中"。
- claim 的候选集包含 `running`（这是崩溃恢复所必需，不能去掉）。上一轮加入的排序
  `ORDER BY (state='running'), created_at, id` 让等待中的工作**第一次**被优先领取；但一个真实的
  provisioning Operation 的步骤链是 sandbox → node → clone → cleanup → done，**每一步都要一次 claim**，
  第一次 claim 之后它一直是 `running`。于是从第二轮起，带楔子的栈上"同为 running 时按 `created_at`
  由旧到新"会让这个永久失败的 Operation **每一轮都赢**，其后所有 Project 的 Operation 只能被领取
  一次就再也推进不了。排序修复把症状从"一次都不被领取"变成"被领取一次"，业务影响仍在：**项目创建
  依旧无法完成**。

### 为什么必须由 Cloud 的 Operation 承载这个终态

`failed` 是 Cloud 关于"这项工作结果如何"的持久事实，而这次拒绝正是这项工作的结果。把拒绝记录在
Operation 上同时解决三件事：Controller 不再被交回这项工作；`one_project_operation` 槽位释放；公开
`Operation` 契约**已经有** `errorCode`（`internal/contract/openapi.go` 的 `Operation` 资源含
`errorCode`，类型为可空字符串），因此原因码无需新增字段即可到达前端。

### 与既有"取消"机制的关系

Cloud 今天已经有一条受支持、受权限控制、可审计的 Operation 终态入口：`force_confirm`
（`internal/core/runtime_force_stop.go`）把该 Workspace 的 `queued`／`running`／`retry_wait`／`blocked`
Operation 置为 `state='failed', error_code='administrative_force_stop'`，保留原 input、actor 与执行身份，
并把 `provisioning` 的 Project 置回 `active`。本文**不**替代它：强停是管理员对**运行时**发起的取消，
需要 `impactConfirmed`、会停 Workspace、会关闭准入；本文记录的是 Cloud 自己已经做出的**能力判定**，
不需要人工介入，也不触碰 Workspace 或运行时控制。两者都是"记录失败"，因此必须共用同一套所有权与版本
约束，不得互相覆盖。

## 决策

### D1：确定性拒绝结算为 `failed` + 该拒绝自己的 `errorCode`

当 Cloud 以 `terminalCredentialRefusals` 中的任一码拒绝一次 `clone_dispatch` 时，被拒的 Operation 进入
终态：

| 列 | 值 |
|---|---|
| `state` | `'failed'` |
| `error_code` | 该次拒绝的错误码，逐字相同 |
| `retry_at` | `NULL` |
| `version` | `+1`（与既有 `failed` 写入者一致，保持乐观版本单调） |
| `updated_at` | `now()` |

| 拒绝原因 | 结算写入的 `error_code` |
|---|---|
| 本部署没有安装凭据执行器 | `repository_credential_executor_unavailable` |
| 引用不允许本次操作所需的能力 | `repository_credential_capability_unsupported` |
| 引用已被所有者撤销（删除凭据） | `repository_credential_revoked` |

`errorCode` 就是守卫返回的那个码，而不是一个新码，也不是一个合并后的类别码。理由：前端与运维要回答的
问题是"为什么这个 Project 一直创建不出来"，答案必须与 Cloud 实际给出的拒绝**逐字对应**，否则同一个
原因会有两个名字，而日志、公开响应与 Operation 三处的对照关系断裂；反过来，把几种原因压成一个
"凭据不可用"的类别码，会让"装执行器就好""换一个声明了 `clone` 能力的引用才好""新建项目并绑定替代凭据
才好"这三条**不同的恢复路径**在界面上变成同一条，用户只能靠猜。清空 `retry_at` 是必要的：claim 会把到
期的 `retry_wait` 移回 `running` 而**不**清除它触发时用的截止时间，因此终态若保留一个 `retry_at`，它会被
读回成"仍在等待重试"。

**为什么撤销必须有自己的码，而不是复用冻结的码。** 两者在界面上都表现为"这个凭据不能用"，但恢复动作
相反：冻结的结束方式是等待（成员回到租户、成员资格被恢复），撤销的结束方式**不存在**——没有解冻、没有
恢复、上游 Token 已销毁，用户唯一的出路是换一个绑定。既然 D5 已经把两者分开持久化（`availability` 的
`frozen` 与 `invalid`），在准入处把它们合并成一个码就等于把已持久化的区别在这里丢掉，并且**在结构上**
使撤销不可能被结算：结算按码判定，而复用冻结的码会让撤销继承冻结的"必须继续重试"，正是本文要修掉的
无限 `retry_wait`。反向的错误同样不允许：冻结**不得**被当作永久失效——把冻结结算掉会把一个会结束的
状态写成对 Project 的永久判决。

**清单被两个通道共享。** `terminalCredentialRefusals` 同时是 mirror 准备路径的分类表
（`mirrorPreparationFailureCode`），因此同一条凭据拒绝在"Cloud 直接执行凭据 clone"与"内部 Git Server
镜像通道"两条通道上落成同一个 Operation `error_code`。镜像通道自身的结算语义（何时结算、栅栏是什么）不在
本文范围，本文只固定它对这些码的分类必须与准入处逐字一致。

**为什么适用面是一张具名清单而不是一个函数。** 判断"这条拒绝是不是永久的"不能靠读错误码的名字或
匹配子串——`repository_credential_executor_unavailable` 与执行器自己的
`credential_executor_unavailable` 只差一个词而语义相反（见 D4）。清单是唯一可评审的形态：一条拒绝要
进入清单，必须同时满足"它是关于持久状态的判断"与"守卫的答案不随重试改变"。今天只有上面三码满足。

### D2：结算是一次独立事务，入口在 `Store.Control`

结算不能写在守卫里：`reject` 会 panic，`transact` 会回滚，**走向拒绝途中写下的任何内容都不会留下**。
因此它必须是拒绝之后、由拒绝的**调用者**发起的第二次事务。

入口选在 `Store.Control` 的尾部——所有控制动作（gRPC 与开发期 simulator HTTP 客户端）的唯一咽喉：

```
if s.credentialedCloner != nil && r.Action == "clone_dispatch" {
    admitted, err := s.admitCredentialedClone(ctx, r)
    if err != nil {
        return nil, s.refusalOutcome(ctx, r, err)
    }
    if admitted != nil {
        return s.registerCredentialedClone(ctx, r, admitted)
    }
}
out, err := s.control(ctx, r)
if err == nil { return out, nil }
return nil, s.refusalOutcome(ctx, r, err)
```

拒绝有**两个**入口，因为带凭据的 clone 在 `Store.Control` 里有**两条**出口：派发路径自己做的准入
（`admitCredentialedClone`）与随后的通用 `s.control`。两条都必须结算——只接住后者会让"部署安装了执行器
但引用不支持 `clone`"这一种拒绝**永不结算**，而那正是本文要消除的楔子，只是换了一个原因。因此两条都
经过同一个 `refusalOutcome`。`admitted != nil` 表示准入成立，此时 `registerCredentialedClone` 自己的
失败按既有路径重试，**不**结算。

- 调用者拿到的错误**永远是拒绝本身**，逐字不变；结算只可能用**它自己的**错误替换返回，且仅当拒绝属于
  `terminalCredentialRefusals`、结算又写不进去时。此时调用者重试，拒绝会被重复给出、结算会被再次尝试，
  因此瞬时失败自愈。
- 对每一个其他错误，`refusalOutcome` 都是空操作。

### D3：`WHERE` 子句就是安全论证，每个条件都承重

```sql
UPDATE operations SET state='failed',error_code=$2,retry_at=NULL,version=version+1,updated_at=now()
 WHERE id=$1 AND workspace_id IS NOT NULL AND state='running' AND step='clone' AND controller_epoch=$3
   AND EXISTS (SELECT 1 FROM projects p WHERE p.id=operations.project_id AND p.credential_ref_id IS NOT NULL)
```

- `state='running' AND step='clone'` —— 被拒的那次派发**仍停在原处**。终态 Operation 不再匹配，因此
  重复结算是空操作而**不是**第二个、互相冲突的事实；这正是幂等性的来源。
- `controller_epoch=$3` —— 观察到此拒绝的那个 Controller 仍然拥有该 Operation。
  `workspaceCloneDispatch` 在到达守卫**之前**就校验了 epoch，所以 epoch 已经移动意味着更新的 Controller
  正在驱动它：那个 Controller 会被同样拒绝、并在它自己的回合结算，这个陈旧的结算**不得**覆盖新状态。
- `credential_ref_id IS NOT NULL` —— 拒绝是关于**这个 Project 的凭据绑定**的（D7 的权威列）。没有这一条，
  一个不命名任何凭据的 Project 也可能被一个对它不可能成立的码结算。
- `workspace_id IS NOT NULL` —— 只有 Workspace Operation 的 clone 步骤会走到这个守卫。

#### 为什么栅在 `controller_epoch` 而不是 `version`

epoch 只在 Controller lease 换手时前进（`lease` 的 `epoch=epoch+1`）；`lease_renew` 不改变它。因此
同一个 Controller 的任期内，`claim` 会反复领取同一个 Operation——**包括**一个停在 clone 步骤的
Operation——每次都推进 `version`，而 epoch 不变。于是：

- 栅在 `version` 上，会让结算在"拒绝之后又被领取一次"时**静默匹配不到**，Operation 继续占着 Project
  唯一的槽位：本文要消除的缺陷被一个顺序测试看不见的竞态复原。Liveness 要求栅在 epoch 上。
- 栅在 `controller_epoch` 上则允许"拒绝所属的那个 epoch 仍然拥有该 Operation"时结算落地，而这**仍然
  安全**，因为这次拒绝与具体尝试无关：它是 `t.store.credentialedCloner == nil` 这一**部署属性**
  （启动时一次性安装），守卫又是全量的，所以被领取的**下一次**尝试会被逐字相同的码拒绝。结算落在
  那一次上，记录的是同一件本来就已成立的事实。

代价要写明：epoch 无法区分同一 Operation 的第 N 次与第 N+k 次尝试——这两者由 `step` 与绑定列区分。

#### 同一个 epoch 内，什么阻止"误终结"

| 条件 | 排除的情形 | 为什么它足够 |
|---|---|---|
| `step='clone'` | Operation 已被**重新推进**过 clone 准入 | `advance` 的步骤链严格向前（`sandbox → node → clone → …`），没有任何转移回到 `clone`；越过 clone 的 Operation 不可能再次停在那里，而这次拒绝命名的正是 clone 步骤 |
| `credential_ref_id IS NOT NULL` | Project 在拒绝之后**不再命名凭据** | 解绑会让 Project 回到匿名 clone——一条仍被支持、会真实派发并运行的路径。把一个已与凭据无关的 Operation 结算成"你的凭据没有执行器"，是终结了一件不再依赖凭据的工作 |

而"结算了另一个 Operation"在结构上不可表达：被结算的行就是守卫评估过的行——`cloneDispatch` 与
`settleRefusedCredentialedClone` 读的是**同一个** `r.Body.S("operationId")`（`clone.go` 的
`cloneDispatch`），守卫的 Project 也来自这一行。

### D4：范围恰好等于拒绝的范围

结算与拒绝一样窄，且**重复声明**这些约束，因为后续对派发路径的改动不得静默放宽它：

- 只对 `r.Action == "clone_dispatch"` 生效。**同样的错误码可以从运行时控制投递路径出现，那不是这次
  Operation 的 clone 准入**，不得结算。
- 只对 `terminalCredentialRefusals` 中**逐字列出的**码生效，今天即
  `repository_credential_executor_unavailable`、`repository_credential_capability_unsupported` 与
  `repository_credential_revoked`。清单**只增不改**：往清单里加一个码要有 D1 的两条满足条件，从清单里
  移除任何码则等于把一条已结算的拒绝改回无限重试，属于对本文终态的撤销，需要单独的决策。
- `operationId` 必须是合法 ID。

特别地，执行器自己的 `credential_executor_unavailable`（`internal/cloneexec` 的
`ReasonCodeUnavailable`）与本文执行器级那个码**只差一个词**，但语义相反：它是"已安装边界的部署发生传输
故障"，是可重试的暂时失败，`registerCredentialedClone` 把它映射为 `&Fault{409, ...}` 返回。以子串匹配
错误码，或把两个码合并，都会把暂时故障误判为永久失败。两者必须保持逐字不同。

### D5：不伪造成功，不绕过检查，不改其他错误码

- **不产生伪造的 Clone 执行。** 结算只写 `operations` 一行，不写 `clone_executions`。一个被拒绝的 clone
  不是"跑过的 clone"；写一行会让恢复逻辑把这笔执行派给一个**完全没有凭据的 Node**。
- **不绕过 scope／tenant／成员／冻结／撤销。** 结算发生在守卫**已经**跑完全部前置校验并拒绝之后；它只记录
  结论，不改变 `credential_refs`、`projects.credential_ref_id`、`projects.lifecycle` 或
  `workspaces.admission_open`。能力条款位于部署级拒绝之后、且先于任何授权产生，因此它也不可能让一次
  clone 越过任何检查。
- **不改变其他错误码的重试行为。** `repository_credential_frozen`、`repository_credential_scope_mismatch`、
  `repository_credential_binding_conflict`、`repository_credential_unavailable`、`dispatch_conflict`、
  `stale_operation` 以及 Node 上报的 `clone_failed`／`clone_result_unknown` 与全部 defer 码，进入
  `retry_wait`／`blocked` 的既有路径逐条不变。这些是**可能改变**的条件（凭据可被解冻、重新绑定或换成
  另一个引用），因此必须继续重试。`repository_credential_frozen` 的语义**只被收窄、不被削弱**：它继续
  覆盖冻结（含个人归属的成员离开、成员资格不再有效）以及 scope 不匹配，只有"引用已被撤销"这一种输入
  改由 `repository_credential_revoked` 报告，而那种输入本来就是一条拒绝。
- **撤销的拒绝码被拆分，拒绝集合未变。** 同步的实现把 `availability != 'available'` 的拒绝按引用的真实
  取值分报，而不是新增一条准入条款：此前会被拒的输入今天**一律仍被拒**，守卫的**放行**条件没有变化，
  fail-closed 未解除。改变的只有"这次拒绝叫什么"，以及由此它能否被结算。与之相对，能力条款是真正新增
  的拒绝，见下一条。
- **守卫只多了拒绝，没有多放行。** 本文同步的实现里，`requireRepositoryAccess` 新增了一条能力条款
  （引用必须声明本次操作所需的 `clone` 能力）。这条偏离必须写明：它**收紧**守卫而不是放宽——守卫对带
  凭据的 Project 仍然是全量拒绝，新增条款只是让一种此前会走到"稍后失败"的输入在准入处就被明确拒绝，
  依据是[仓库凭据连续性](../project/20260927-repository-credential-continuity.md) D4 的"能力不支持时
  明确拒绝"。守卫的**放行**条件一个字节未改，fail-closed 未解除，匿名 clone 路径未触碰。这一条款改变
  了守卫的判定集合，因此不变量 1 按新的表述重新声明。
- **能力条款对部署托管引用的后果必须显式处理。** `ConfigureCredential` 写下的引用不带 `capabilities`
  （列默认 `'[]'`），因此它会被这一条款拒绝。今天不可观测——执行器未安装，部署级拒绝先拦住，集成套件
  里的那些引用仍看到旧的码；但它会在**解禁那次变更**中暴露。这不是本文的裁决范围，已登记在
  [仓库凭据能力词汇表](../identity-access/20261012-repository-credential-capability-vocabulary.md)，
  并要求解禁变更把它作为一项显式的迁移义务处理，而不是顺带放过。

### D6：终态的三个对外后果

- **不占队列。** `one_project_operation` 的谓词不含 `failed`，因此槽位释放，用户可以对该 Project 发起
  新的操作。
- **永久失败不可由用户重试。** 公开的重试入口只接受 `blocked`／`retry_wait`，其他状态返回
  `409 operation_not_retryable`。这对本终态是正确的：重试一个**能力不存在**或**凭据已被撤销**的操作，
  只会再次被拒。用户恢复的路径是显式的（部署侧安装执行器、绑定一个具备能力的引用、或对 Project 发起新的
  操作），不是重试。
- **原因经公开契约到达前端。** `Operation.errorCode` 是既有字段，无需 OpenAPI 变更、无需重新生成前端
  客户端；前端只需在既有的 Project／Workspace 操作界面上展示失败状态与原因码。本轮实现选定并落地为：
  经既有的 Workspace 列表取到 `creatorOperationId`，再读该 Operation 的 `errorCode` 并在项目详情页展示
  ——公开契约没有"某项目的 Operation 列表"路由，因此没有新增路由。文案覆盖清单里的三个码与
  `administrative_force_stop`，未知码回退为原因码本身而不是一句泛化的安慰话；不提供重试按钮（公开重试
  入口本来就拒绝 `failed`）。位置与文案的调整不需要改动本文的任何决策。

## 不变量

1. `repository_credential_executor_unavailable` 的判定条件与顺序不变，继续无条件拒绝，fail-closed
   不因本文解除；`requireRepositoryAccess` 对带凭据 Project 的**放行**条件一个字节未改。
2. 守卫相对本文起草时多的**唯一**判定是能力条款：引用必须声明本次操作所需的 `clone` 能力，否则以
   `repository_credential_capability_unsupported` 拒绝。它是**新增一种拒绝**，不是新增一种放行，且位于
   部署级拒绝之后（见 D5）。撤销的分报**不是**新增判定：它把此前已被拒绝（并报为冻结）的输入按真实原因
   报告，拒绝集合与放行集合都不变。
3. 只有 `terminalCredentialRefusals` 中逐字列出的错误码会结算 Operation，今天即
   `repository_credential_executor_unavailable`、`repository_credential_capability_unsupported` 与
   `repository_credential_revoked`。
4. 冻结**永不**结算。`availability='frozen'`（含个人归属的成员离开或成员资格失效）与 scope 不匹配继续
   报 `repository_credential_frozen`／`repository_credential_scope_mismatch` 并照旧进入重试；把其中任何
   一种结算掉，等于把一个可能结束的状态写成对 Project 的永久判决。
5. 结算只在 `state='running'`、`step='clone'`、`controller_epoch` 匹配、且 Project 的
   `credential_ref_id` 非空时写入；每次结算至多写一行、至多 `version+1` 一次。
6. 任何一道栅栏不成立的结算都是**静默空操作**：Operation 已 `failed`、epoch 不再是当前拥有者、步骤已
   越过 `clone`、或 Project 已不命名凭据时，`state`／`error_code`／`version`／`updated_at` 全部不变，
   且不向调用者返回错误。
7. 陈旧 epoch 的结算不写入；当前 Controller 的结算必定落地。
8. 结算**不写** `clone_executions`，也不改 `credential_refs`、`projects.credential_ref_id`、
   `projects.lifecycle`、`workspaces.admission_open`。尤其地，结算**不写** `credential_refs`：撤销是
   所有者动作的既成事实，Operation 只是记录它的后果。
9. `failed` 的 Operation 不再被 claim 返回，也不再占用 Project 的 `one_project_operation` 槽位。
10. 除本文这一条转移、不变量 2 的条款与撤销的分报外，任何 Operation 的状态、步骤、defer 与重试语义
    不变；匿名 clone 链路不变。
11. 公开 `Operation` 契约不新增字段；`errorCode` 继续是可空字符串。

## 安全条件的核实（评审材料）

评审要求先回答："独立事务结算的并发栅栏，是否足以阻止迟到的拒绝误终结同一 epoch 下已被新领取或重新
推进的执行。"核实方式是读实现路径，并在隔离 PostgreSQL 上对每道栅栏做**变异检验**——移除该条件，
对应测试必须失败；无法失败的条件不算证据。

| 安全条件 | 实现依据 | 证据（移除条件即失败） |
|---|---|---|
| 被结算的行就是守卫评估过的行 | `cloneDispatch` 与结算读同一个 `r.Body.S("operationId")`，守卫的 Project 取自这一行 | 结构上不可表达跨 Operation 污染 |
| 只有当前拥有者能写入 | 派发在到达守卫**之前**已校验 epoch；`claim` 写入的 epoch 来自 `leaseValid`，而 lease 只在换手时前进 | 陈旧 epoch 的结算不写入，后继者自己的结算必定落地 |
| 同一 epoch 内的重新领取不会让结算静默失效 | 栅在 `controller_epoch` 而非 `version`（见 D3） | 领取推进 `version` 后，结算仍然落地 |
| 已被重新推进过 `clone` 的 Operation 不被终结 | `step='clone'`，且 `advance` 严格向前 | 移除该条件即失败 |
| 拒绝之后解绑凭据的 Project 不被终结 | `credential_ref_id IS NOT NULL` | 移除该条件即失败 |
| 两条拒绝路径都被接住 | 准入与 `s.control` 的失败都经同一个 `refusalOutcome`（见 D2） | 只接住一条时，"已安装执行器但引用不支持能力"的拒绝不结算，对应测试失败 |
| 部署级拒绝不被能力条款抢答 | 能力条款位于 `credentialedCloner == nil` 之后 | 两者同时不成立时，结算写入的是部署级的码而非能力码 |
| 撤销被终结，冻结不被终结 | 结算按码判定，`repository_credential_frozen` 刻意不在清单内；分报由引用的 `availability` 取值决定 | `TestARevokedReferenceSettlesTheOperationToo` 断言撤销写入 `failed` + 该码且 `retry_at` 为空；`TestSettlementIgnoresEveryRefusalButThePermanentOnes` 逐条断言冻结与 scope 不匹配不写 `failed`。把 `repository_credential_frozen` 加进清单即失败——这正是"暂时冻结被误当永久失效"的变异 |
| 撤销先于部署级判定被报告 | `availability` 的判定位于守卫前缀，在成员、scope 与部署级条款之前 | `TestARevokedReferenceIsRefusedAsRevokedBeforeTheDeploymentIsConsulted`；把该判定移到部署条款之后即失败 |
| 并发领取与结算重叠不产生第三个状态 | 两次写入落在同一行，靠 `-race` 与完整允许集断言 | 并发领取 + 结算下终局唯一 |
| 终态后不再被领取，Project 槽位释放 | `failed` 不在 `one_project_operation` 谓词内，也不在 claim 候选内 | 结算后同一 Project 可再发起 Operation，且受害者 Project 的完整步骤链每步都被领取 |

**结论：栅栏充分，没有发现需要修改实现的缺陷。** 本轮只补了两道此前从未被当作栅栏检验过的条件
（`step`、`credential_ref_id`）的测试，实现未改。上面"证据"一列的具名测试属于实现侧证据，正式登记
为 `specs/test-cases` 的核心用例仍按规范留到本文变为 `approved` 之后（见"落地顺序"第 2 步）。

## 为什么不是这些替代方案

| 方案 | 优势 | 否决理由 |
|---|---|---|
| 保留上一轮的 claim 排序修复，不加终态转移 | 零状态机改动，改动面最小 | 排序只在"等待中的工作"与"运行中的工作"之间取舍，而真实 Operation 只在第一次 claim 前是 `queued`。从第二轮起永久失败的 `running` 楔子按 `created_at` 每轮都赢，业务影响（项目创建无法完成）仍在 |
| 让 claim 按"最久未被触碰"（`updated_at`）排序 | 不碰生命周期 | 楔子每轮被刷新、`updated_at` 最新，会被排到最后而受害者得以推进——但这是**启发式**而不是不变式：任何一次对楔子的触碰都会翻转结论，且它让调度器的排序承担了"区分永久失败与临时失败"的职责。可作为上一轮排序修复的补充，不能替代终态 |
| 在 `claim` 里检测并终结 | 单一入口 | 把生命周期决策塞进调度器：claim 必须为此读取 Project 凭据、复制守卫的判定，等于在第二个地方重新实现准入。守卫是唯一的准入点，这一条必须保持 |
| 由 Controller 处理 409 并自行 defer 成永久失败 | 不改 Cloud 状态机 | Controller 无法区分"部署缺能力"与"执行器暂时不可达"，两个码只差一个词；且它必须自己推断 Cloud 的能力，把 Cloud 的部署事实复制到另一个进程。`desktop/` 亦为只读 |
| 新增 `terminally_refused` 之类的 Operation 状态 | 语义更精确 | 会改变公开契约的状态枚举（兼容性硬边界），并让每一个既有的 `state` 分支（claim 候选、重试入口、前端映射、`one_project_operation`）都要重新判定。`failed` + 精确 `errorCode` 已经完整表达了"这项工作失败、原因是能力缺失" |
| 把几类确定性拒绝合并成一个类别码（如 `repository_credential_unusable`） | 前端只需认识一个码 | 几条原因对应**不同的恢复动作**（装执行器 / 换一个支持 `clone` 的引用 / 给项目绑一个替代凭据），合并后界面无法区分，用户只能靠猜；且它与 Cloud 实际返回给调用者的 409 码不再逐字对应，日志、公开响应与 Operation 三处对照断裂 |
| 把撤销与冻结合并成"凭据不可用"一类 | 减少一个码 | 两者的**结束方式**相反：冻结会结束（成员回租户即可解冻），撤销不会。合并后要么把冻结也结算（把临时状态写成永久判决），要么让撤销继续无限重试（本文要修的楔子），二者必居其一 |
| 用"错误码不在已知可重试集合内"来推断是否结算 | 新增码无需改结算逻辑 | 把"永久"变成了对**未知**码的默认判定：将来任何新的守卫拒绝码都会自动终结 Operation，包括那些本该重试的。适用面必须是具名清单（D1） |
| 撤销复用 `repository_credential_frozen`，由结算事务再读一次 `credential_refs.availability` 决定是否终结 | 公开契约不新增拒绝码，调用者在准入处看到的行为与今天逐字相同 | 让公开响应说错原因：用户删掉的凭据会被回答"已冻结"，而冻结是一条**会结束**的状态，用户会去等一个不会发生的解冻，且无法在准入处知道该换绑定。结算的判定也不再是"守卫返回了什么"，而是它在另一个事务里第二次查到的状态，把一条准入事实变成两次读之间可漂移的推断。合并正是 D1 已经否定的"把恢复路径不同的原因压成一个码" |
| 在守卫内部写终态 | 代码集中在一处 | `reject` panic 且 `transact` 回滚，走向拒绝途中写下的内容不会留下。写在守卫里等于写在会被丢弃的事务里 |
| 用强停（`administrative_force_stop`）替代 | 复用既有机制 | 强停是管理员对**运行时**发起的取消：需要 `impactConfirmed`、会停 Workspace、会关闭准入，而且需要人工介入才能解决一个 Cloud 自己已经知道答案的问题。两者可以并存，但不能互相替代 |
| 直接删除或重置该 Operation | 立刻解除阻塞 | 破坏历史执行记录与审计；Cloud 的持久化边界要求失败事实保留 |
| 把 Operation 置为 `succeeded` 以放行 | 让链路立刻走通 | 伪造成功，并会让后续步骤在没有代码的 Workspace 上推进 |

## 失败与恢复语义

| 情形 | 对外结果 | 持久事实与恢复 |
|---|---|---|
| 部署未安装执行器，Project 绑定了凭据 | 该次 clone 被拒 `409 repository_credential_executor_unavailable`；Operation 结算为 `failed` + 该码 | `operations` 一行被写入；不写 `clone_executions`；Project 的槽位释放 |
| 执行器已安装，但引用未声明 `clone` 能力 | 该次 clone 被拒 `409 repository_credential_capability_unsupported`；Operation 结算为 `failed` + 该码 | 同上；结算与拒绝不触及 `credential_refs` 与 Project 的绑定列 |
| 引用未声明 `clone` 能力且部署也没有执行器 | 只报部署级的码 | 结算写部署级的码：部署级拒绝先判定，且它才是两者中用户无法通过换凭据绕开的那个 |
| 引用已被所有者撤销（删除凭据） | 该次 clone 被拒 `409 repository_credential_revoked`；Operation 结算为 `failed` + 该码 | 同上；撤销本身由所有者的 DELETE 在**更早**的事务里写成（`availability='invalid'`、`revoked_at`、Token 已销毁），结算只记录它的后果，不写 `credential_refs`，也不改 Project 的绑定列 |
| 引用已被撤销，**且**部署没有执行器 | 报 `repository_credential_revoked`，不报部署级的码 | 撤销是绑定自身的持久状态，判定在守卫前缀；部署级拒绝对一个已撤销的绑定不携带信息。两者都是终态，结算都会落地，差别只在 Operation 记录的原因码 |
| 同一拒绝被重复观察到 | 调用者每次仍收到同一个 409 | 结算是空操作，`state`／`error_code`／`version`／`updated_at` 全部不变 |
| 拒绝发生后 Operation 已被新 epoch 的 Controller 接管 | 新 Controller 收到它自己的 409 | 陈旧结算不写入；新 Controller 在自己的回合结算，终态必定落地 |
| 结算事务本身失败（数据库瞬时错误） | 该次调用返回结算的错误而非拒绝 | 调用者重试；拒绝被重复给出，结算再次尝试，因此自愈 |
| 执行器已安装但传输不可达 | `409 credential_executor_unavailable`（执行器自己的码） | **不**结算；Operation 保持 `running`，按既有路径重试 |
| 凭据被冻结或 scope 不匹配 | 既有的 `repository_credential_frozen`／`repository_credential_scope_mismatch` | **不**结算；这些是可能改变的条件，重试行为逐条不变。冻结覆盖"引用被显式冻结"与"个人归属的所有者已离开或成员资格失效"，两者都会在成员恢复后自然结束 |
| 匿名 Project（未绑定凭据） | clone 正常派发 | 守卫提前返回，结算永不适用 |
| Node 上报 `clone_failed` | 既有 `retry_wait`／`clone_failed` | **不**结算；Node 侧失败的重试边界由既有 defer 规则承担 |
| 管理员对该 Workspace 发起强停 | 既有 `administrative_force_stop` | 两者都写 `failed`，共用同一套所有权与版本约束；强停额外停 Workspace 并把 `provisioning` 的 Project 置回 `active` |

## 本决策未解决的问题

- **创建项目时的绑定拒绝码未随之拆开**：`CreateProject` 的绑定校验对 `availability != 'available'` 仍一律
  返回 `409 repository_credential_frozen`（公开契约行为不变，`.acceptance/git-credential.json` 记录的
  即此行为）。该路径是同步的，调用者当场就能改选另一个凭据，因此不存在"被拒的派发不是终点"这个问题；
  但同一条已撤销的引用在准入处叫 `repository_credential_revoked`、在绑定处叫 `repository_credential_frozen`。
  是否统一为同一个码（以及随之要重跑的凭据验收证据）留待评审裁决，本文不单方面改变已记录的公开契约行为。
- **`clone_failed` 的错误分类与重试边界**：本文只处理"守卫拒绝"这一种确定性失败。Node 上报的
  `clone_failed` 是否需要在某处区分为"临时"与"确定性"，以及是否需要上限或退避策略，是一个尚未裁决的
  独立问题——**不得**把所有 clone 失败统一改成永久失败，也不得靠无限增大重试间隔掩盖它。
- **长期重试的可见性**：`retry_wait` + 10 秒节奏在没有终态的情况下会无限增长执行记录。本文不改变任何
  既有的重试节奏，该类副作用的治理属于上述独立问题。
- **解禁后的语义**：部署安装了执行器之后，`repository_credential_executor_unavailable` 不再出现，本转移
  自然不再触发；这是期望行为，无需回滚路径。但既有 `failed` 的 Project 需要用户显式发起新操作才能恢复，
  是否需要一条"凭据执行器就绪后重新驱动"的路径，属于后续裁决。
- **能力条款带来的恢复路径**：被 `repository_credential_capability_unsupported` 终结的 Project，恢复方式
  只能是换一个**声明了 `clone` 能力**的引用（或让现有引用具备该能力）。今天 `credential_refs.capabilities`
  与 `projects.credential_ref_id` 都没有更新路径，因此实际上只能新建凭据引用并重新绑定 Project。是否
  需要一条"管理员验证后重绑"的受控入口，属于
  [仓库凭据能力词汇表](../identity-access/20261012-repository-credential-capability-vocabulary.md)的裁决范围，
  本文不预判。

## 落地顺序

本文的状态是 `proposed`。按规范，`proposed` 阶段**不编写或维护核心测试用例**；实现已经先行落地，是因为
项目负责人已就"新增受控终态转移"给出明确批准（约束为：不得超过既有 Operation 所有权、Controller
epoch、version、lease、事务与状态机约束；不解除 fail-closed）。**评审通过前，本文不得标记为
`approved`，也不得据此补充核心测试用例。**

1. **评审本文并裁决状态（材料齐备，待裁决）**：确认 D1–D6、不变量 1–11 与最终同意的方案一致；确认"新增
   一条终态转移"这一修改被接受为对根决策 D1 的**前向修改**，而不是对既有语义的重新解释；确认不变量 2
   记录的能力条款被接受为守卫的**收紧**（依据连续性补录 D4"能力不支持时明确拒绝"）；并确认撤销的**分报**
   被接受为对
   [用户自助 HTTPS Git 凭据](../identity-access/20261010-user-managed-git-credentials.md) D5 与其"失败与
   恢复语义"表的修改（拒绝集合不变，只有拒绝码与能否结算改变）。评审前要求核实的并发安全条件已完成并
   记入"安全条件的核实（评审材料）"：栅栏充分，未发现需要修改实现的缺陷。
   完成条件：本文状态变更为 `approved`，且正文已按评审反馈纠正。
2. **同步核心测试用例（随第 1 步完成）**：在
   `specs/test-cases/cloud/operation/workspace-runtime-lifecycle.md` 增加一个英文标题的稳定引用用例，
   登记"确定性拒绝结算为终态"的验证义务（结算、幂等、陈旧 epoch、非本码不结算、匿名不回归、多步推进、
   各条拒绝路径都被接住、部署级拒绝不被能力条款抢答、冻结不被结算而撤销被结算），证据指向
   `cloud/internal/core/repository_credential_settlement_test.go`。完成条件：该用例的每项义务都有
   指名的直接证据，或如实标为 `Partial`／`Missing`。
3. **前端呈现（已完成）**：在既有 Project／Workspace 操作界面上展示 Operation 失败状态与
   `errorCode`，使"凭据执行器不可用""凭据不支持所需能力""凭据已被删除"可被用户看到，而不是永远停留在
   "创建中"。实现位于 `frontend/src/features/projects/provisioning.ts`（经 Workspace 的
   `creatorOperationId` 取 Operation 的 `errorCode`）与其详情页呈现，文案与测试同轮更新，两个模块
   README 同步。完成条件已满足：前端门（`npm run check` 与生成客户端漂移检查）通过。
4. **验收栈证据（部分完成）**：在独立验收栈上要求五项证据，逐项状态如下。
   - 一个带凭据的 Clone 因**确定性拒绝**被拒并终结：**已取得**——隔离栈 `ora-credential-acceptance` 上
     `repository_credential_executor_unavailable` 的结算、0 条 `clone_executions`、
     `.acceptance/git-credential.json`。`repository_credential_revoked` 与
     `repository_credential_capability_unsupported` 在**真实栈**上尚无证据（只有真实 PostgreSQL 支持的
     core／DB 测试），如实记为缺口。
   - 另一个正常 Project 能**完整推进后续步骤**：**已取得**（同一份证据的"受害者多步推进"检查；本轮
     镜像通道的私有 Clone 成功路径另见 `internal-git-server-mirror-acceptance.md`）。
   - 队列不被永久拒绝的 Operation 卡住：**已取得**（同一份证据的静置检查）。
   - 原因能经 API 到达前端：**部分**——接口与前端实现齐备，端到端的人工确认属于手动验收流程。
   - 匿名 Clone 不回归：**已取得**（本轮镜像通道 E2E 中 Node 侧的匿名 HTTPS clone 即该路径）。
   完成条件：上表中仍标为缺口的项各有可指名的证据，或由评审明确接受该缺口。

保持不变的行为：`requireRepositoryAccess` 对带凭据 Project 的**放行**条件、fail-closed、所有其他错误码的
重试行为、Operation 既有的步骤表与 defer 规则、匿名 clone 链路、公开 API 形状。从本文开始改变的行为：
**三处**——(1) 因部署缺少凭据执行器而被确定性拒绝的 clone 步骤，其 Operation 由"停留在 `running` 等待
重试"改为"结算为 `failed` 并携带该拒绝的 `errorCode`"；(2) 守卫新增一条能力条款，把"引用存在、范围匹配、
但不允许本次操作"从"稍后失败"提前为准入处的明确拒绝；(3) 被所有者撤销的引用不再报"冻结"，而以
`repository_credential_revoked` 拒绝并同样结算——这是拒绝码的分报，不是判定集合或放行集合的改变。
