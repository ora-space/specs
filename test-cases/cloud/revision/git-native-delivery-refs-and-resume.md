# Revision 经 Git Server 交付与续作

依据已批准的 [Cloud Revision Git 交付决策](../../../decisions/cloud/revision/20261017-git-native-delivery-refs-and-resume.md)。下文的
D 编号与不变量编号都指这篇 ADR；“契约”指
[代码中心契约决策](../../../decisions/cloud/git-server/20261016-project-repository-as-code-hub-and-ref-contract.md)。

现有的 [Cloud 校验后登记的 bundle Revision](verified-bundle-revisions.md) 与
[同一 Issue 的运行续接最近一次 Revision](../issue-run/resume-from-latest-revision.md) 仍以已实现的根决策与续接决策为依据，实现合入
并把本 ADR 改为 `implemented` 时再按本文改写或改挂。内部线的实现尚未合入 cloud 的 main，下列义务都是 **Missing**。

## The Delivery Input Fixes The Repository, Delivery Ref And Workspace Ref

- 风险：交付输入由 Controller 或 Node 拼地址，重传时变化；交付 ref 不是本工作项自己的，两次尝试写同一个 ref；`workspaceRef` 与 clone 时冻结的分支不同。
- 前置状态：clone 成功、进入 `delivering` 的运行；配置齐全的 `git_server`。
- 触发：放出交付工作项；Controller 改写输入中的地址后 `RecordDispatch`；放出后修改 Git Server 地址并重试交付。
- 必须成立：输入含 `repository_base_url` 加 `/<tenantId>/<projectId>.git` 拼成的地址、`refs/ora/deliveries/<该工作项 ID>`、逐字取自 checkout clone 输入的 `workspaceRef`，`baseCommit` 等于 Workspace 基线；改写的输入被拒；重试的新工作项使用新地址与新的交付 ref。
- 禁止结果：输入含 `bundleKey` 或任何凭据；`revisionRef` 出现在 Git Server 上。
- 验证义务与证据：**Missing**。
- 决策依据：D1，不变量 1、9、11。

## Delivery Is Skipped Without Object Store Or Git Server Configuration

- 风险：缺少配置时仍放出交付工作项，或把未保存伪装成已保存。
- 前置状态：运行进入 `delivering`；分别去掉 `object_store`、`git_server`、两者都去掉。
- 触发：运行推进；`git_server` 地址或证书形状非法时启动 Cloud。
- 必须成立：不放出交付工作项，同一事务以 `skipped` 结算并释放，原因依次为 `object_store_unconfigured`、`git_server_unconfigured`，每个运行只记一次；形状非法时拒绝启动，启动时不访问网络。
- 禁止结果：登记 Revision；同一运行记录两次跳过。
- 验证义务与证据：**Missing**。
- 决策依据：D2，不变量 10。

## A Revision Is Registered Only After The Delivery Ref Is Verified

- 风险：Node 声明的最终提交未经核验就登记；Git Server 短暂不可用被判为核验失败，白白消耗重试；ref 不存在仍登记，下一次续作才失败。
- 前置状态：Node 报告 `delivered` 与 `unchanged`；Git Server 与对象存储可以注入故障。
- 触发：ref 查询返回一致的提交、`404 ref_not_found`、不同的提交、`5xx`、超时；JSONL 大小不符；结果带 bundle 或 `bundle_failed`；`unchanged` 的最终提交既不是基础提交也不是前序最终提交。
- 必须成立：比对在接管前完成且不访问外部；一致时在同一事务登记 `revision_verifications`、带 `delivery_ref` 的 `revisions` 与收据并调用 `deliverySettled`；不存在或不符时接管原事件、记 `verification_failed`、不登记并按放弃窗口重试；不可用时返回 `UNAVAILABLE`，不接管、不确认，重放同一事件后成功。
- 禁止结果：未核验就登记；`workspaceRef` 的快进结果参与核验；不可用被记为 `verification_failed`。
- 验证义务与证据：**Missing**。
- 决策依据：D3，不变量 2、3。

## Every Git Revision Has Its Own Delivery Ref And Old Rows Stay Untouched

- 风险：`unchanged` 的 Revision 没有自己的 ref，续作要一跳引用；两次尝试登记同一个 ref；迁移改写旧的 bundle 行。
- 前置状态：迁移前已有三种形状的 bundle Revision 行。
- 触发：执行迁移；登记一个 `delivered` 与一个 `unchanged`；试图插入 bundle 列非空的 Git 行或 `delivery_ref` 重复的行。
- 必须成立：旧行原样通过 `revisions_shape`；新行 `delivery_ref` 非空、形状合法、唯一，bundle 列全空，`bundle_revision_id` 不写入；`unchanged` 也有交付 ref。
- 禁止结果：迁移删除或改写旧行；违反形状的行被写入。
- 验证义务与证据：**Missing**。
- 决策依据：D4，不变量 4、5。

## Git Access Follows The IssueRun Phase

- 风险：交付执行在 `delivering` 之外仍能写交付 ref；会话在取消或 Thread 结束后仍能改变 Git Server；WorkflowRun 的会话套用 IssueRun 的条件获授权。
- 前置状态：IssueRun 的会话与交付执行；一个执行到 agent 节点的 WorkflowRun。
- 触发：会话在 `starting`、`running`、请求取消后、Thread `ending` 时访问；交付执行在 `delivering` 与之后访问；WorkflowRun 的会话访问。
- 必须成立：会话只在 `phase ∈ {starting, running}`、无取消请求、Thread 未在结束时获授权，不要求带 `priorRevision`；交付执行只在 `delivering` 时获授权；WorkflowRun 的会话一律 `403`。
- 禁止结果：已结束或已取消的会话推送成功；WorkflowRun 的会话读到任何 ref。
- 验证义务与证据：**Missing**。传输中途不复查授权是已知取舍，不作为义务。
- 决策依据：D5，不变量 7。

## Resume Selects Only Revisions With A Delivery Ref And New Commits

- 风险：续作选中只含快照提交的 Revision，把本地分支拨回更旧的提交；地址变化后仍续作，恢复必然失败；一次取消让 Revision 永久不可续作。
- 前置状态：同一 Issue 有旧 bundle 行、最终提交等于基础提交的 Git 行、有新提交的 Git 行。
- 触发：再次运行该 Issue；改变 Git Server 地址后运行；前序恢复以 `prior_revision_base_unavailable` 结束，分别在运行已请求取消与未取消时结算。
- 必须成立：候选只含有交付 ref 且有新提交的最新 Revision；地址不等于本次 clone 输入时记 `repository_changed` 并从头开始；会话前序带 `{revisionId, finalCommit, baseCommit, gitRepositoryUrl, deliveryRef}`，交付前序只带 `{revisionId, finalCommit}`；未取消时标记不可续接，已取消时不标记。
- 禁止结果：跳过后回退到更早的候选；旧 bundle 行成为候选；查 `bundle_revision_id`。
- 验证义务与证据：**Missing**。
- 决策依据：D7，不变量 6。

## Public Revision Reads Keep Their Shape

- 风险：Git Revision 的 `changed` 按 bundle 判断而恒为 `false`；仓库地址或交付 ref 进入公开 API。
- 前置状态：旧 bundle 行、Git `delivered` 行、Git `unchanged` 行。
- 触发：读取运行与 Issue 页的 Revision。
- 必须成立：投影形状不变；Git 行的 `changed` 取自核验结论，`bundleSize` 为 `null`；旧行照旧。
- 禁止结果：公开 API 出现仓库地址、交付 ref、`workspaceRef` 或对象键。
- 验证义务与证据：**Missing**。
- 决策依据：D8。
