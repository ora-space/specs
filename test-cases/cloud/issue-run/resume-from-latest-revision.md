# 同一 Issue 的运行续接最近一次 Revision

依据[已实现的 IssueRun 续接决策](../../../decisions/cloud/issue-run/20261010-resume-run-from-latest-issue-revision.md)。
证据来自 `ora-space/cloud` 的 `integration/agent_run_resume_test.go`（真实 gRPC 与 PostgreSQL，以 simulator 扮演
Controller/Node）、迁移测试与前端测试。Node 的恢复步骤见
[Node 恢复前序 Revision](../../node/revision/prior-revision-restore.md)。

跨仓库真实链路证据来自 cluster 的 `task revision:resume:acceptance`（见
[Revision 交付与续接验收](https://github.com/ora-space/cluster/blob/main/docs/testing/revisions.md)）：同一 Issue
连续运行三次，第二次看到第一次的提交，第三次的 bundle 包含前两次的提交，空运行沿用前序 bundle。验收失败不能单独
定位到下列某条义务，因此只作为补充引用，不据此标记 `Covered`。

## New Runs Resume The Issue's Latest Bundled Revision Fixed At Session Start

### 风险

续接了错误的 Revision（其他 Issue、其他仓库、较旧的成果），或前序在会话启动后被改变，Node 恢复的不是 Cloud
登记的那一个。

### 前置状态

同一 Issue 上有已登记的 Revision，其中一些带 bundle 或沿用 bundle。

### 触发

新运行进入 `starting`，放出会话工作项。

### 必须成立

- 选取同一 Issue 其他运行中按登记时间最新、带 bundle 的 Revision，写入会话输入的 `priorRevision` 与
  `issue_runs.resume_revision_id`；首条提问的 Context 带 `resume`。
- 没有候选时会话输入不带 `priorRevision`，Context 不带 `resume`，不签发下载授权。
- 候选的 Project 或仓库地址与本次不同时从头开始，Context 记 `{skippedRevisionId, reason: project_changed}`。
- 交付输入的前序从会话输入复制，不带 bundle。

### 禁止结果

续接其他 Issue 或其他仓库的 Revision；会话启动后前序变化。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 续接最近一个并固定 | `TestFollowUpRunResumesTheIssueLatestRevision`；`TestResumedRunWithoutNewCommitsReusesThePriorBundle`：第三次运行续接第二次，经第一次的 bundle 一跳，交付输入的前序不带 bundle | Covered |
| 没有历史时不变 | `TestRunWithoutHistoryStartsFresh` | Covered |
| 换了项目时从头开始 | `TestRunOfAMovedIssueStartsFresh`。缺口：没有“同一项目只改了仓库地址”的单独用例 | Partial |

### 决策依据

续接决策 D1、D4，不变量 1、2、5。

## Only A Missing Base Refuses Further Resumes

### 风险

暂时的恢复失败永久阻断续接；或远端历史被改写后，每次运行都以同样原因失败。

### 前置状态

运行续接了前序 Revision，Node 恢复失败。

### 触发

会话以 `agent_failed` 与 `prior_revision_*` 的 `detail` 结束，下一次运行启动。

### 必须成立

- `prior_revision_base_unavailable` 在会话结束结算中把前序标为不可续接；下一次运行从头开始，Context 记
  `{skippedRevisionId, reason: resume_refused}`，不回退到更早的候选。
- `prior_revision_unavailable` 不做标记，下一次运行仍续接同一个 Revision。
- 两种情况运行都为 `failed`，照常交付并登记一个没有前序与 bundle 的 `unchanged` Revision。

### 禁止结果

`prior_revision_unavailable` 标记不可续接；被标记后回退到更早的 Revision。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 两种失败的标记与下一次选取 | `TestRestoreFailuresAndTheNextRunSelection` 的两个子测试 | Covered |
| 不回退到更早候选 | 现有用例中被标记的候选之前没有更早的候选 | Missing |

### 决策依据

续接决策 D3，不变量 3、4。

## Resumed Runs Without New Commits Reuse The Prior Bundle

### 风险

空运行重复存储 bundle；或沿用 bundle 的 Revision 找不到 bundle，之后无法续接。

### 前置状态

续接运行的交付输入带前序最终提交。

### 触发

交付结果为 `unchanged`（停在前序最终提交）或 `delivered`。

### 必须成立

- 停在前序最终提交：登记新行，没有自有 bundle，`prior_revision_id` 为前序，`bundle_revision_id` 指向实际持有 bundle
  的 Revision；`changed` 为 `false`，`deliveryState` 为 `unchanged`。
- 有新提交：自己持有 bundle，`prior_revision_id` 为前序。
- 数据库约束拒绝既有自有 bundle 又沿用 bundle 等非法形状；迁移在新库与升级路径上都成立。

### 禁止结果

沿用 bundle 的行需要多跳才能找到 bundle；非法形状被写入。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 沿用并一跳 | `TestResumedRunWithoutNewCommitsReusesThePriorBundle` | Covered |
| 新提交存自有 bundle | `TestResumedRunWithNewCommitsStoresItsOwnBundle` | Covered |
| 形状约束与迁移 | `TestRevisionResumeColumnsKeepTheBundleShape`；`integration/migration_upgrade_path_test.go::TestMigration0033RevisionResumeAppliesFreshAndUpgrades` | Covered |

### 决策依据

续接决策 D5，不变量 6。

## Resume Is Visible To The Agent And The Public API

### 风险

用户与 Agent 不知道本次运行接着哪一次成果工作，或公开 API 泄漏内部标记。

### 前置状态

运行续接了前序 Revision。

### 触发

读取运行；查看 Thread 首条记录。

### 必须成立

- 运行带 `resumeRevisionId`，`revision` 投影带 `priorRevisionId`；不可续接标记不公开。
- Issue 页显示续接来源，续接后没有新提交时显示沿用了续接的成果。

### 禁止结果

公开 `resume_refused_*`；续接运行看不出来源。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 公开投影 | `TestPublicRunProjectsResume` | Covered |
| 前端展示 | 前端 `run-resume.test.tsx`；`run-delivery.test.tsx` 中续接后没有新提交的用例 | Covered |

### 决策依据

续接决策 D4。
