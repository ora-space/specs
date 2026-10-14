# Git 交付的输入、授权、核验与登记

依据[已批准的 Cloud Revision 后续决策](../../../decisions/cloud/revision/20261012-git-server-delivery-and-verified-delivery-refs.md)
（下称“主 ADR”），并承担 [Git Server 根决策](../../../decisions/cloud/git-server/0-sandbox-scoped-git-server-over-https.md)
不变量 12。主 ADR 尚未实现：`ora-space/cloud` 仍以对象存储中的 bundle 交付，所以新增义务大多为 `Missing`。现有证据
分三类：

- 继承不变的规则（JSONL 的 `HEAD` 核对、比对先于外部访问、外部 I/O 后重新围栏）沿用 `integration` 包中经真实
  PostgreSQL 与 RustFS 的测试。这些测试同时断言了将被删除的 bundle 部分，第 4 步改写时须保留 JSONL 与顺序的断言。
- Git Server 一侧的授权与 ref 查询来自 `internal/core/git_access_db_test.go`、`internal/gitserver/service_test.go` 与
  `integration/git_server_test.go`。它们只证明 Git Server 的判定与应答，不证明 Cloud 的核验与登记顺序。
- cluster 的 `revision:acceptance` 与 `revision:resume:acceptance` 改走 Git 后作为端到端补充证据。验收失败只能说明
  链路断开，不能单独定位到下列某条义务，因此只作补充引用，不据此标记 `Covered`。

## Delivery Work Fixes Its Repository URL And Delivery Ref At Release

### 风险

交付输入在派发时漂移或仍带 bundle 对象键，Node 推送到错误的仓库或 ref；Controller 改写地址后 Cloud 无从发现；
Git 访问材料或对象存储凭据进入输入、数据库或日志。

### 前置状态

运行进入 `delivering`，`object_store` 与 `git_server` 都已配置；运行 Workspace 属于某个 Project。

### 触发

Cloud 放出交付工作项，包括失败后重试放出的新工作项；Controller 以相同或改写过的输入调用 `RecordDispatch`；
工作项等待期间修改 `git_server.repository_base_url`。

### 必须成立

- 输入恰好含 `gitRepositoryUrl = <repository_base_url>/<tenantId>/<projectId>.git` 与
  `deliveryRef = refs/ora/deliveries/<该工作项 ID>`，`historyKey` 照旧，不含 `bundleKey`；`revisionRef` 仍是 Node 本地的
  `refs/ora/revisions/<runId>`。
- 地址与 ref 在插入工作项时写入；重试的新工作项有新的 `deliveryRef`；等待期间修改配置只影响之后放出的工作项。
- `RecordDispatch` 要求 Controller 转交的输入与固定输入逐字相同，改写地址或 ref 以冲突拒绝。
- 输入、数据库与日志中只有地址、ref 与对象键，没有 git-access 材料、Cloud 的 ref 查询身份或对象存储凭据。

### 禁止结果

输入带 `bundleKey`；两个工作项共用一个交付 ref；登记被改写过的地址；任何凭据出现在输入、表或日志中。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 新工作项输入的形状 | 尚无实现。`integration/agent_run_delivery_test.go::TestDeliveryExecutionClaimCarriesTheFixedSpecAndMovesNothing` 现在断言的是逐次尝试的 `bundleKey`/`historyKey`，需改为断言 `gitRepositoryUrl`、`deliveryRef` 且没有 `bundleKey`；`::TestFailedDeliveryKeepsDeliveringAndReleasesOneBackoffRetry` 需同时断言重试换用新的 `deliveryRef` | Missing |
| 逐字转交 | `integration/agent_run_control_test.go::TestSessionWorkIsRegisteredOnTheCloudChosenNode`：改写过的输入以 `CONFLICT` 拒绝，规则不依赖字段。缺口：带 `gitRepositoryUrl`、`deliveryRef` 的交付输入被改写时尚无测试 | Partial |
| 不含凭据 | `integration/agent_run_control_test.go::TestThreadCommandsAndUploadGrantsAreNotLostOrStored` 扫描数据库，确认上传授权不落库。缺口：git-access 材料与 ref 查询身份不出现在输入、表与日志中尚无断言 | Partial |

### 决策依据

主 ADR D1，不变量 1、9；Git Server 根决策 D4、D5。

## Delivery Needs Both The Object Store And The Git Server

### 风险

只配置了对象存储的部署照常放出交付，推送注定失败，沙盒被占用到放弃窗口；或缺少配置时把未保存的成果伪装成已保存。

### 前置状态

会话结束，运行进入 `delivering`。

### 触发

缺少 `object_store`；配置了对象存储但缺少 `git_server`；两者都缺；放出交付的事务回滚后重放。

### 必须成立

- 任一配置缺失时不放出交付工作项，交付以 `skipped` 结算，运行直接进入释放；原因先看对象存储、后看 Git Server，
  分别为 `object_store_unconfigured`、`git_server_unconfigured`。
- `skipped` 与业务钩子同事务提交，重复放出只结算一次，不登记 Revision。
- 健康检查报告缺失的依赖；两项配置齐全时只有 Git 交付一种形态。

### 禁止结果

缺少 Git Server 时仍放出交付；登记没有代码改动位置的 Revision；同一运行结算两次 `skipped`。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 缺对象存储 | `integration/agent_hook_atomicity_test.go::TestUnconfiguredRevisionStorageSkipsOnceAndRollsBackWithBusiness`：不放出工作项，`skipped` 与业务钩子同事务回滚，重复放出只结算一次，不登记 Revision | Covered |
| 缺 Git Server | 尚无 `git_server` 配置；缺 Git Server 时以 `git_server_unconfigured` 结算、两者都缺时记对象存储原因，均无测试 | Missing |
| 健康检查 | `git_server` 的健康报告尚无实现与测试 | Missing |

### 决策依据

主 ADR D2，不变量 10。

## Git Server Grants Write Only To A Delivering Delivery And Read Only To A Restoring Session

### 风险

运行离开对应阶段后沙盒仍能写入或读取 Project 仓库；会话执行获得写权限；用户取消后恢复仍能继续读取。

### 前置状态

Git Server 根决策 D3 的共同条件成立：证书目标是当前存活代次，执行属于当前 Node/沙盒，maintenance 绑定已确认，
触发者是有效租户成员且账号有效，Project 有效。

### 触发

交付执行在 `delivering` 与离开 `delivering` 后请求；带或不带前序的会话执行在 `starting`、`running` 请求；运行请求
取消、Thread 进入 `ending`/`ended`、执行已有结果或被强停；clone 或插件执行请求。

### 必须成立

- 交付执行：尚无结果、未被强停且运行 `phase = delivering` 时可读写，只能新建固定输入中的 `deliveryRef`。
- 会话执行：输入带 `priorRevision`、尚无结果、未被强停、运行 `phase ∈ {starting, running}`、没有取消请求且 Thread
  不在 `ending`/`ended` 时只读。
- 其他执行无权；任何拒绝都是同一个不含细节的 `403`。

### 禁止结果

会话执行写入；不带前序的会话读取；取消或 Thread 结束后的读取被放行；交付执行新建固定输入之外的 ref。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 交付写权限的阶段条件 | `internal/core/git_access_db_test.go::TestGitAccessGrantsRestoreReadAndDeliveryWriteFromCurrentState`：`delivering` 中的交付执行获得写权限；`::TestGitAccessDeniesWhenAnyCurrentFactIsWithdrawn`：执行已有结果、运行离开 `delivering`、运行删除，以及代次、Project、触发者、Node、沙盒、绑定任一条件撤回后，都以相同 `403` 拒绝。缺口：被强停终止的执行没有单独断言 | Partial |
| 可写 ref 取自固定输入 | `internal/core/git_access.go` 现在按 `refs/ora/deliveries/<workId>` 推导，与主 ADR 写入的 `deliveryRef` 恒等，但不读取输入；`internal/gitserver/service_test.go::TestPushRulesRejectEveryOtherUpdate` 证明只能新建授权给出的那一个 ref。读取固定输入要在第 4 步实现并测试 | Partial |
| 会话读权限的阶段条件 | `TestGitAccessGrantsRestoreReadAndDeliveryWriteFromCurrentState`：`starting` 中不带前序拒绝、带前序只读、会话有结果后拒绝；`::TestGitAccessRestoreReadEndsWithTheSession`：取消请求与 Thread `ending` 立即拒绝；`internal/gitserver/service_test.go::TestReadOnlyExecutionFetchesButCannotPush`：只读执行能拉取、不能推送。缺口：`running` 阶段与 Thread `ended` 没有单独断言 | Partial |
| 其他执行无权 | `internal/core/git_access.go` 只接受 `agent_session` 与 `deliver_revision` 两种执行；clone 与插件执行被拒绝尚无测试 | Missing |

### 决策依据

主 ADR D5，不变量 7；Git Server 根决策 D3、D4。

## Revision Is Registered Only After The Ref Query Confirms The Delivery Ref

### 风险

Cloud 登记一个从未确认存在、或指向其他提交的交付 ref，续作要到下一次运行才失败；Git Server 短暂不可用把一次
已经成功的推送判为失败；外部核验之后运行或 Node 已经变化仍然登记。

### 前置状态

交付执行已登记；Node 报告 `RevisionDelivered` 或 `RevisionUnchanged`。

### 触发

结果与输入矛盾；ref 查询返回 `200` 且提交相同、`200` 但提交不同、`404 ref_not_found`、`503`、超时或网络失败；
JSONL 的 `HEAD` 一致或不一致；外部核验之后租约、运行或当前 Node 变化。

### 必须成立

- 先比对：`baseCommit`、`revisionRef`、`history.key` 等于固定输入，`unchanged` 的最终提交等于基础提交或输入中前序的
  最终提交，`delivered` 二者都不等；不一致在接管前拒绝，不访问 Git Server 与对象存储，不写任何记录。
- 拒绝按性质分类：与固定输入矛盾（Node、`baseCommit`、`revisionRef`、`history.key`、`unchanged`/`delivered` 与最终
  提交的关系）返回 `ABORTED`/`CONFLICT`（`result_conflict`）；形状不合法（`finalCommit` 格式、`history` 的大小与
  SHA-256 形状）返回 `INVALID_ARGUMENT`/`INVALID_INPUT`（`invalid_result`）。
- 事务外：ref 查询确认交付 ref 指向声明的最终提交，JSONL 经 `HEAD` 核对大小与 SHA-256；两者都一致才进入登记。
- 事务内：重新检查执行身份、运行与当前 Node，写入核验结论、Revision（含 `delivery_ref`）与收据，并在同一事务结算
  交付。`unchanged` 同样经 ref 查询核验并登记交付 ref。
- ref 不存在、指向其他提交或 JSONL 不一致：接管事件，Cloud 记 `failed{verification_failed}`，不登记 Revision，按
  IssueRun 根决策 D5 重试新交付；原 Node 声明保留在 `node_executions.result`。
- ref 查询或对象存储不可用：不接管、不确认，返回不可用，Controller 稍后重放同一事件。

### 禁止结果

未经 ref 查询即登记 Revision；登记的 Revision 指向未经确认的提交；Git Server 不可用被判为核验失败；矛盾结果触发
外部访问；与输入矛盾的结果以 `invalid_result` 拒绝，或形状不合法的结果以 `result_conflict` 拒绝。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 比对先于外部访问 | `integration/agent_run_delivery_test.go::TestRevisionResultContradictingItsInputNeverReachesTheObjectStore`：矛盾结果在接管前拒绝，且不访问对象存储。缺口：不访问 Git Server 尚无断言 | Partial |
| 拒绝按性质分类 | `internal/core/revision.go::validateDeliveryDeclaration` 现在按字段分类：`history.key` 与输入不一致返回 `invalid_result`，与本规则不符。`history.key` 不一致返回 `result_conflict`、`finalCommit` 与 `history` 形状不合法返回 `invalid_result` 的直接测试尚无 | Missing |
| ref 查询确认后才登记（Git Server 不变量 12） | Cloud 尚无 ref 查询客户端，登记前没有 ref 核验。`internal/gitserver/service_test.go::TestDeliveryPushCreatesOnlyItsRefAndReplayIsUpToDate` 与 `integration/git_server_test.go::TestGitServerAuthorizesDeliveryPushFromCloudState` 只证明 ref 查询在推送后返回该提交、未知 ref 返回 `404`，不证明 Cloud 的登记顺序 | Missing |
| JSONL 核对 | `integration/revision_test.go::TestRevisionS3VerificationAndAtomicTakeover`：真实 RustFS 上对象缺失、大小或摘要不符都不登记，钩子与提交一起回滚，原始证据保留，重放幂等 | Covered |
| ref 不存在或不符记核验失败 | `integration/agent_run_delivery_test.go::TestRevisionVerificationFailureIsCloudsOwnVerdict`、`::TestNodeMayNotReportCloudsOwnVerificationVerdict`：对象不符时由 Cloud 判定 `verification_failed`、不登记，Node 不能自报该结论。缺口：ref 返回 `404` 与指向其他提交尚无测试 | Partial |
| 不可用时可重放 | `integration/revision_test.go::TestRevisionNetworkFailureRemainsReplayable`、`integration/agent_run_delivery_test.go::TestDeliveryWithAnUnreachableObjectStoreIsRefusedTransiently`：对象存储不可用时不接管、不确认，可重放。缺口：ref 查询的 `503`、超时与网络失败尚无测试 | Partial |
| 外部核验后重新围栏 | `integration/revision_test.go::TestRevisionLeaseAndStateAreRecheckedAfterExternalVerification`：外部 I/O 之后重新检查租约与运行状态。缺口：外部 I/O 包含 ref 查询时尚无测试 | Partial |
| `unchanged` 也核验并登记交付 ref | 现在的 `unchanged` 只核对 JSONL；经 ref 查询核验并登记 `delivery_ref` 尚无实现与测试 | Missing |

### 决策依据

主 ADR D3，不变量 2、3；Git Server 根决策 D6，不变量 12。

## Every New Revision Has A Delivery Ref And No Bundle

### 风险

同一行同时出现 bundle 与交付 ref 两种形态；一个交付 ref 被登记两次；迁移改写或删除已经核验过的旧 Revision。

### 前置状态

数据库中有迁移 0033 形态的旧 Revision：自有 bundle、沿用前序 bundle、停在基础提交三种。

### 触发

应用新迁移；登记 Git 的 `delivered`、`unchanged` 以及续接后没有新提交的 Revision；写入混合形态或重复的
`delivery_ref`。

### 必须成立

- 新迁移之后旧行原样满足约束；新行的 `delivery_ref` 形如 `refs/ora/deliveries/<uuid>`，`bundle_key`、`bundle_size`、
  `bundle_sha256`、`bundle_revision_id` 全为空，`final_commit` 可以等于 `base_commit`。
- 非空的 `delivery_ref` 在表内唯一。
- 不再写入 `bundle_revision_id`；`prior_revision_id` 的写入规则不变。

### 禁止结果

同时有 `delivery_ref` 与任一 bundle 列的行；旧行被改写、删除或补上 `delivery_ref`。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 迁移与形状约束 | 新迁移尚未编写。`integration/migration_upgrade_path_test.go::TestMigration0033RevisionResumeAppliesFreshAndUpgrades` 与 `integration/agent_run_resume_test.go::TestRevisionResumeColumnsKeepTheBundleShape` 只约束 0033 的三种 bundle 形态 | Missing |
| 交付 ref 唯一 | 部分唯一索引尚无实现与测试 | Missing |
| 旧行保留 | 升级前后旧行逐列不变尚无测试 | Missing |

### 决策依据

主 ADR D4，不变量 4、5。

## Public Revision Projection Keeps Its Shape

### 风险

Git Revision 改变公开 API 的形状或泄露仓库地址与交付 ref；前端把 `bundleSize: null` 显示成 `bundle 0 B`。

### 前置状态

运行登记了 Git Revision 或升级前的 bundle Revision。

### 触发

读取运行详情；Issue 页展示运行的交付。

### 必须成立

- `revision` 投影仍为 `{id, baseCommit, finalCommit, changed, bundleSize, historySize, priorRevisionId, createdAt}`；
  `changed` 取自 Cloud 核验结论（`delivered` 为 `true`，`unchanged` 为 `false`）。
- Git Revision 的 `bundleSize` 为 `null`，旧行照旧返回大小。
- 仓库地址、交付 ref 与对象键不进入公开 API。
- 前端展示有改动的 Git Revision 时不出现 bundle 大小。

### 禁止结果

投影字段增删或改名；公开响应中出现 `refs/ora/deliveries/` 或 Git Server 地址；显示 `bundle 0 B`。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 旧行投影 | `integration/agent_run_resume_test.go::TestPublicRunProjectsResume`：续接后没有新提交的旧 Revision 投影为 `changed = false` 并给出 `priorRevisionId`。缺口：没有断言完整字段集合与 `bundleSize` | Partial |
| Git Revision 投影 | `changed` 取自核验结论、`bundleSize` 为 `null`、地址与 ref 不出现，尚无实现与测试 | Missing |
| 前端展示 | cloud `frontend/src/features/issues/thread/run-delivery.tsx` 与 `run-delivery.test.tsx` 尚未去掉 bundle 大小 | Missing |

### 决策依据

主 ADR D8。
