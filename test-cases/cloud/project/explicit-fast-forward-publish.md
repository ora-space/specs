# 显式快进发布

依据已批准的 [发布 ADR](../../../decisions/cloud/project/20261017-explicit-fast-forward-publish.md)。下文的 D 编号与不变量
编号都指这篇 ADR。执行器一侧的推送与分类见 [执行器边界用例](../repo-executor/repo-executor-boundary.md)。发布的 API、
审批、状态机与 reconciler 尚未合入 cloud 的 main，下列义务都是 **Missing**。

## Only Approved Publish Operations Write The External Repository

- 风险：Workspace 推送触达外部仓库；未经批准或审批人不对的发布被派发；管理员以他人的个人凭据写入外部仓库。
- 前置状态：团队、个人、未知三类凭据；发起人是否有 Project 发布权限、是否为个人凭据的所有者。
- 触发：创建发布；管理员批准使用他人的个人凭据；批准后凭据轮换；批准过期；没有发布权限也不是凭据所有者的成员发起。
- 必须成立：发起人是有 Project 发布权限的人或所用个人凭据的所有者；发起人即审批人时创建即批准，否则 `awaiting_approval`；批准绑定全部字段与凭据版本；凭据换版本以 `cancelled(credential_changed)`、超过 24 小时以 `cancelled(approval_expired)` 结束；未知归属的凭据不能发布。
- 禁止结果：管理员批准使用他人的个人凭据；批准后改动字段仍派发；无权成员发起成功。
- 验证义务与证据：**Missing**。
- 决策依据：D1、D2，不变量 1、2、3。

## Publish Sources Are Any Workspace Branch Or Delivery Ref Of The Project

- 风险：发布源按 Workspace 创建者校验，与契约放宽后的共享读取不一致；`revisionId` 发布用了未核验的提交。
- 前置状态：同一 Project 内多个 Workspace 的分支和交付 ref；已登记的 Revision。
- 触发：以他人 Workspace 的分支作为源；以交付 ref 作为源；只给 `revisionId`；源 ref 不存在；`expectedOldOid` 等于 `sourceCommit`。
- 必须成立：本 Project 任意 Workspace 分支或交付 ref 都可作源，Cloud 经 ref 查询确认存在；`revisionId` 取 Revision 登记的交付 ref 与最终提交；`expectedOldOid` 等于源提交时 `400 nothing_to_publish`。
- 禁止结果：按创建者拒绝合法的源；跨 Project 的 ref 被接受；`revisionId` 与 `sourceRef` 同时给出时被接受。
- 验证义务与证据：**Missing**。
- 决策依据：D1，不变量 5。

## Publish States Follow The Executor Classification And Running Happens Once

- 风险：`running` 中失去租约或连接的发布回到 `pending` 后重推；同一目标分支并发写入。
- 前置状态：执行器替身返回各种分类；可以杀掉的 worker。
- 触发：执行器返回 `applied`、`conflict`、`rejected`、`not_fast_forward`、`source_unavailable`、`failed`、`uncertain`；`running` 中杀掉 worker；同一 `(bindingId, targetBranch)` 两个发布同时被领取。
- 必须成立：分类按 D3 的表映射到状态；`running` 中失去租约或连接进入 `uncertain`；同一 `(bindingId, targetBranch)` 至多一个 `leased` 或 `running`；只有 `running` 之前可以取消。
- 禁止结果：同一发布第二次进入 `running`；`uncertain` 占用目标分支；`running` 中被取消。
- 验证义务与证据：**Missing**。
- 决策依据：D3、D4，不变量 4、6、8。

## Uncertain Publishes Are Reconciled Read-Only Within A Bounded Number Of Checks

- 风险：结果不明时重推，在用户不知情时再次写入；外部长期不可达让 `uncertain` 永不结束，删除 Project 无限等待。
- 前置状态：`uncertain` 的发布；fixture 可以暂停、恢复或一直不可达。
- 触发：外部目标分支等于 `sourceCommit`、等于 `expectedOldOid`、等于其他值；读取持续失败。
- 必须成立：第一次核验不早于执行器任务的截止时间；核验只用 `probe` 授权读取；三种观察值分别得到 `succeeded`、`failed(not_applied)`、`conflict(moved_after_uncertain)`；读取失败按指数退避（最长间隔 1 小时）重试，最多 20 次（约 16 小时），用尽后 `failed(outcome_unknown)`。
- 禁止结果：任何核验路径发出更新命令；`uncertain` 没有上限；核验早于执行器的截止时间。
- 验证义务与证据：**Missing**。
- 决策依据：D5，不变量 7。

## A Successful Publish To The Binding Branch Leads To A Sync That Includes It

- 风险：发布成功后当前快照永远不包含发布的提交；复用了在发布生效前就 fetch 完的同步；建不了同步时让发布的终态事务回滚。
- 前置状态：发布的目标分支等于 Binding 的分支；Binding 没有同步、有 `pending`/`leased` 的同步、有 `running` 的同步；凭据可以被冻结。
- 触发：发布以 `succeeded` 结束（包括核验确认）。
- 必须成立：没有未结束的同步时新建 `after_publish`；`pending`/`leased` 时复用；`running` 时置补同步标记，由它的终态事务按同步 ADR D3 至多新建一个后继；建不了时把原因记入发布的 `result.successorSkipped`，发布照常 `succeeded`。
- 禁止结果：复用 `running` 的同步；发布直接改写当前快照；自动同步失败改变发布的结果。
- 验证义务与证据：**Missing**。
- 决策依据：D6，不变量 9；同步 ADR D3。

## Publish Audit Keeps Facts Without Secrets

- 风险：审计记录 token、平台的原始输出或文件内容；审计被修改。
- 前置状态：带 canary token 的凭据；平台返回包含凭据或 URL 用户信息的拒绝原因。
- 触发：发起、批准、拒绝、取消、推送、核验。
- 必须成立：每个事件追加记录，包括字段、凭据版本、前后 `ls-remote` 观察值、分类与错误码、每次核验；平台原因先脱敏再截断到 2 KiB。
- 禁止结果：审计中出现 canary token、完整原始输出或 Workspace 文件内容；历史记录被改写。
- 验证义务与证据：**Missing**。
- 决策依据：D7，不变量 10。
