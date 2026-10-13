# 从交付 ref 续作前序 Revision

依据[已批准的 Cloud Revision 后续决策](../../../decisions/cloud/revision/20261012-git-server-delivery-and-verified-delivery-refs.md)
（下称“主 ADR”）D7，它修改了 [IssueRun 续接决策](../../../decisions/cloud/issue-run/20261010-resume-run-from-latest-issue-revision.md)
的候选规则与不可续接标记。主 ADR 尚未实现：`ora-space/cloud` 仍按 bundle 选取前序并经 `bundle_revision_id` 一跳，
现有 `integration/agent_run_resume_test.go` 只能证明继承不变的部分。cluster 的 `revision:resume:acceptance` 改走 Git
后作为端到端补充证据，不据此标记 `Covered`。

## Resume Selects Only A Revision With Its Own Delivery Ref And New Commits

### 风险

续作选中停在基础提交的 Revision，把新 checkout 的默认分支拨回比本次 clone 更旧的提交；选中升级前只有 bundle 的
Revision，Node 无从拉取；续接后没有新提交的 Revision 仍要经一跳找到别的 Revision。

### 前置状态

同一 Issue 有多个已登记的 Revision：Git `delivered`；停在基础提交的 Git `unchanged`；续接后没有新提交、最终提交等于
前序最终提交的 Git `unchanged`；升级前的 bundle 行；被标为不可续接的行；Issue 移动前其他 Project 的行。

### 触发

Issue 上的新运行进入 `starting`，Cloud 生成会话输入与首条提问。

### 必须成立

- 候选只含有 `delivery_ref` 且最终提交不等于基础提交的 Revision，按登记时间取最新一个；续接后没有新提交的 Revision
  用它自己的交付 ref，不经 `bundle_revision_id`。
- 会话输入的 `priorRevision` 为 `{revisionId, finalCommit, baseCommit, gitRepositoryUrl, deliveryRef}`：地址按当前
  `git_server.repository_base_url` 与候选的 tenant、Project 拼成，`baseCommit` 是候选的基础提交；交付输入的
  `priorRevision` 只有 `{revisionId, finalCommit}`。
- 没有 `delivery_ref` 的旧行不是候选，首条提问的 `resume` 也不为它记录跳过；没有候选时从头开始。
- Project 或仓库地址不同、或候选被标为不可续接时从头开始，并照旧记录原因（续接决策 D1 第 2、3 条，不变）。

### 禁止结果

选中最终提交等于基础提交的 Revision；选中只有 bundle 的旧行；会话输入缺少 `baseCommit` 或交付 ref；为旧行记录
`skippedRevisionId`。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 候选规则 | 尚无实现。`integration/agent_run_resume_test.go::TestFollowUpRunResumesTheIssueLatestRevision`、`::TestResumedRunWithoutNewCommitsReusesThePriorBundle` 证明的是按 bundle 选取与一跳，需改写为按交付 ref 选取、筛掉停在基础提交的 Revision | Missing |
| 会话与交付输入的形状 | `internal/controlgrpc/session_models_restore_test.go::TestSessionWirePreservesModelBindingAndPriorRevision` 证明 `PriorRevision` 经编解码逐字保留，但字段仍是 `bundle`；带 `baseCommit`、`gitRepositoryUrl`、`deliveryRef` 的会话输入尚无测试 | Missing |
| 旧行不是候选且不记录跳过 | 升级前 bundle 行与 Git 行并存时的选取尚无测试 | Missing |
| 换 Project 或不可续接时从头开始 | `integration/agent_run_resume_test.go::TestRunOfAMovedIssueStartsFresh`：Issue 移到另一个 Project（仓库地址也不同）后从头开始并记 `project_changed`；`::TestRestoreFailuresAndTheNextRunSelection`：被标为不可续接后下一运行从头开始并记 `resume_refused`。缺口：同一 Project 只改仓库地址没有单独断言 | Partial |
| 没有历史时从头开始 | `integration/agent_run_resume_test.go::TestRunWithoutHistoryStartsFresh`：没有 Revision 的 Issue 不带 `priorRevision`，首条提问不变 | Covered |

### 决策依据

主 ADR D7，不变量 6；IssueRun 续接决策 D1。

## A Refused Restore Marks The Prior Revision Unless The Run Was Cancelled

### 风险

用户在恢复的几秒内取消运行，Git Server 对已取消运行的读请求返回 `403`，Node 报告
`prior_revision_base_unavailable`，前序 Revision 被永久拒绝续作；或暂时失败也被永久标记。

### 前置状态

续作运行的会话执行正在恢复前序 Revision。

### 触发

会话以 `agent_failed` 加 `prior_revision_base_unavailable` 或 `prior_revision_unavailable` 结束；结算时运行有或没有取消
请求。

### 必须成立

- `prior_revision_base_unavailable` 且运行没有取消请求：结算事务把前序 Revision 标为不可续接，下一运行从头开始并
  说明原因。
- 运行在结算时已有取消请求：不标记，前序 Revision 仍可被之后的运行选中。
- `prior_revision_unavailable`：不标记。

### 禁止结果

一次取消让前序 Revision 永久不可续作；暂时失败被标为不可续接。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 永久失败才标记 | `integration/agent_run_resume_test.go::TestRestoreFailuresAndTheNextRunSelection`：`prior_revision_base_unavailable` 标记不可续接、下一运行从头开始；`prior_revision_unavailable` 不标记、下一运行仍续作同一 Revision | Covered |
| 取消时不标记 | 结算时检查取消请求尚无实现与测试 | Missing |

### 决策依据

主 ADR D7；IssueRun 续接决策 D3。
