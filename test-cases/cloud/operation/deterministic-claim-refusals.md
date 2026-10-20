# 确定性领取拒绝与轮转

依据[已批准的 operation 后续决策](../../../decisions/cloud/operation/20261014-settle-deterministic-dispatch-refusals.md)。
本文件登记 Cloud 在同一个领取事务里结算确定性仓库凭据拒绝、按最近一次领取机会轮转候选，以及两者与
已登记执行、旧快照和 clone 预算的边界。

直接证据来自 cloud#74 的 `internal/core` 与 `integration` 包：全部用例在真实、隔离的 PostgreSQL schema 上经
`Store.Control` 的 `claim` 动作驱动，与 Controller 的生产路径一致，断言不直接改写操作状态。真实 Rust
Controller 的联合矩阵仍按 #108 记为未覆盖；Go 模拟 Controller 的成功不能替代它。

## Deterministic Repository Refusals Settle Inside The Claim Transaction

### 风险

领取之后才发生的确定性拒绝让操作永远停在 `running`，其他项目的正常操作也领不到；或者结算只提交一半，
操作已经 `failed` 而 Workspace 仍可准入、运行时维护没有归还；或者把未知外部执行当成业务拒绝。

### 前置状态

一个 `create_project` / `create_workspace` 操作停在 `clone` 步，没有未结算或成功的 clone，也没有未决的
外部 effect 或执行；其 Project 的仓库绑定了一个仓库凭据。

### 触发

Controller 调用 `claim`；候选的仓库凭据不存在、已冻结、与仓库范围不匹配，或该凭据类型的执行边界尚未提供。

### 必须成立

- 同一事务把操作置为 `failed`，写入对应的 `repository_credential_*` 错误码，`retry_at=NULL` 并递增 version。
- 主 Workspace 置为 `unavailable` 并关闭执行准入；运行 Workspace 在同一事务走 `runWorkspaceSettled(failed)`。
- 归还该操作的运行时维护控制，保留阶段、请求、外部 effect 与资源身份。
- 只有提交成功才返回结果；运行 Workspace 结算钩子失败时整笔领取回滚。

### 禁止结果

半结算（操作终态与 Workspace/维护控制分属不同结果）；把已登记执行或成功 clone 改写成拒绝；为凭据项目
改发匿名 clone 或删除历史凭据引用。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 四类确定性拒绝各自结算 | `TestClaimSettlesEachDeterministicRefusalCode`：冻结、软删、范围不匹配、执行边界关闭分别得到 `failed` + 对应错误码 + `retry_at=NULL` + version 2，Workspace `unavailable` 且准入关闭 | Covered |
| 运行 Workspace 同事务结算并归还维护 | `TestClaimRefusalSettlesRunWorkspaceInSameTransaction`：IssueRun 走到 `releasing/failed/workspace_unavailable`，运行时维护不再由该操作持有 | Covered |
| 钩子失败整笔回滚 | `TestClaimRefusalRollsBackWhenTheRunSettlementHookFails`：claim 返回错误，操作仍是 `queued`/v1、Workspace 仍 `provisioning`、维护仍由该操作持有 | Covered |
| 阶段、请求与已登记执行保留 | `TestClaimSettlesRetryWaitCandidateAfterSettledCloneFailure`：操作带着一条已结算的 `clone_failed` 执行行，领取后按新的确定性拒绝结算为 `failed`；测试只读 `state`/`error_code`/`retry_at`，不查询 `clone_executions`，也不逐字节断言 `step`/`request`/effect 列 | Partial（执行行是否保留、执行计数和阶段/请求列都没有直接断言） |

### 决策依据

D1、D2，不变量 1–3。

## A Refused Claim Frees The Queue For Other Operations

### 风险

一个必然被拒绝的旧候选占住队列，其他项目永久失去领取机会；或者在一个事务里无限遍历，把事务时间拖长。

### 前置状态

一个较早创建、凭据必然被拒绝的候选，以及一个较新创建、可以正常领取的候选。

### 触发

同一轮及后续多轮 `claim`；同一事务内连续命中 16 个拒绝候选。

### 必须成立

- 结算一个拒绝候选后在同一事务继续检查下一个候选，不把它原样交回。
- 同一轮领取返回健康候选；结算后的快照不再包含终态操作。
- 单事务最多检查 16 个候选；预算耗尽返回空，下一轮周期继续，不能推断队列永久为空。

### 禁止结果

把被拒绝或已终态的操作再次交给 Controller；用预算上限判断队列为空；一次领取跨越无限候选。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 拒绝候选让位给其他项目 | `TestClaimRefusalFreesQueueHeadForOtherProjects`：同一领取返回较新的健康操作，旧候选结算为 `failed`；再次领取仍返回健康候选 | Covered |
| 预算有界且可续 | `TestClaimRefusalBudgetReturnsEmptyAndNextPeriodContinues`：16 个拒绝候选耗尽预算时返回空，下一轮返回预算外的健康候选 | Covered |

### 决策依据

D2 的候选预算段、D3。

## Claim Rotation Remembers Consumed Opportunities

### 风险

只按创建时间排序时，一个长期 `running` 或反复失败的旧操作一直占住队头；或者把心跳、无关写入当成调度
机会，重启后丢失轮转位置。

### 前置状态

同一项目之外的两个可领取操作；历史数据库中的操作行 `last_claimed_at` 为 NULL。

### 触发

连续多轮 `claim`；从 0049 升级并重复执行迁移。

### 必须成立

- 只在真正返回操作时以数据库时间写 `last_claimed_at`。
- 候选按 `COALESCE(last_claimed_at, created_at), created_at, id` 升序选择，形成可预期的轮转。
- 历史行保持 NULL，按创建时间首次参与，不伪造过去的领取时间。
- 重复迁移不改写已记录的领取时间和历史列。
- 该字段是内部调度簿记，不进入公开操作和 gRPC 快照。

### 禁止结果

未返回操作却写入领取时间；用 `updated_at` 充当机会；迁移改写历史操作行；把该字段暴露到公开契约。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 轮转顺序 | `TestClaimRotatesByLastClaimedAt`：按 A/B/A/B 顺序返回，候选只在真正返回后改变下次顺序 | Covered |
| 历史行、升级与重复迁移 | `TestMigration0050ClaimRotationAppliesFreshAndUpgrades`：空库、0049 数据库升级、重复迁移后历史列不变、历史行 NULL、部分索引存在、已记录的领取时间不被重写 | Covered |
| 只在返回时写入且不暴露到公开契约 | 上述用例覆盖返回后的顺序；字段剥离在 `store.go` 的行映射处完成，尚无直接断言公开序列化不含该字段 | Partial |

### 决策依据

D3、不变量 4。

## A Claim Never Overrides Registered Or Unknown Work

### 风险

凭据后来冻结，改写已经成功的 clone 或结果未知的执行；把未知外部执行当成拒绝并终止，破坏安全停止契约。

### 前置状态

凭据已冻结的项目；候选分别已有成功 clone、未结算 clone、未决 effect/执行，或不在 clone 步。

### 触发

`claim` 遇到这些候选；未知 clone 结果被 defer 为 `blocked` 后再次领取。

### 必须成立

- 只有 clone 步且没有未结算/成功 clone、未决 effect 或执行的候选才进入确定性结算。
- 已登记执行、成功 clone 和非 clone 步骤按原路径交回，不因后来冻结的凭据改写。
- 未知结果保持 `blocked`，不重新派发、不自动重试；外部 effect 与执行身份保留。

### 禁止结果

改写成功 clone 的事实；把未知执行当确定性拒绝；删除历史凭据引用或改为匿名 Git。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 已登记 clone 与非 clone 步不被结算 | `TestClaimRefusalPreservesRegisteredAndNonCloneWork`：成功 clone、未结算 clone、sandbox 步三个候选都被原样交回为 `running` | Covered |
| 未知结果不再被领取 | `TestCloneFailureRetriesAsNewExecutionAndUnknownBlocks`：未知 clone 结果 defer 为 `blocked` 后 claim 返回空，准入不开放 | Covered |
| 未决 effect / Node 执行的保护 | 结算查询已排除计划中的 effect 与未结算的 Node 执行，但还没有在 claim 级用例里直接构造这两类身份 | Partial |

### 决策依据

D1、D2 第 1 步、D4。

## Stale Snapshots Cannot Advance A New Claim

### 风险

旧 Controller 或旧快照推进新一轮领取；领取之后凭据被冻结，Controller 仍按旧快照派发。

### 前置状态

一个已经返回给 Controller 的操作；另一个持有旧 epoch 或旧 version 的调用方；领取后、派发前凭据发生变化。

### 触发

使用过期 epoch 或过期 version 的控制写入；`claim` 返回之后绑定/冻结凭据；派发时的仓库访问再核验。

### 必须成立

- 领取写入当前 `controller_epoch` 并递增 version，旧 epoch 与旧 version 的写入被拒绝。
- 领取后授权变化时，下一轮权威 `claim` 把该候选结算为确定性拒绝，不登记新的 clone 执行。
- `RecordDispatch` 仍再次核验仓库访问，拒绝时保持原操作、不登记执行、不改写资源所有权。

### 禁止结果

旧快照推进新周期；用一个 RPC 错误的解析结果直接调用强停接口；派发时绕过凭据核验。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 结算递增版本 | `TestClaimSettlesEachDeterministicRefusalCode`：结算使 version 从 1 变为 2 | Covered |
| 旧 epoch / 旧 version 被拒 | `TestControlGRPCLeaseHolderAndFencing`（过期 epoch 的续租与释放被拒）；`TestGRPCControllerDrivesWorkspaceThroughNodeAndClone`（旧 epoch 上报 Node、旧 version 推进操作分别得到 `STALE_CONTROLLER` / `CONFLICT`） | Covered |
| 领取后授权变化 | `TestClaimSettlesCredentialFreezeBetweenClaimAndDispatch`：claim 返回后绑定并冻结凭据，下一轮把 `running` 候选结算为 `failed`，没有登记 clone 执行 | Covered |
| 派发时再核验 | `TestCredentialedCloneRefusesMissingExecutorAndAnonymousFallback`：已领取的凭据项目在派发时被拒绝，未登记执行、未改写所有权与凭据引用 | Covered |

### 决策依据

D1 最后一行、D2 末段、D3。

## Rotation And Refusal Keep Clone Budgets And Restarts

### 风险

轮转或领取事务的重放把 clone 次数清零；重启后重新 clone 或复活已经失败的操作。

### 前置状态

一次真实 clone 失败或结果未知的创建操作；已达到失败预算的操作。

### 触发

三次真实 clone 失败；Workspace 重启；领取轮转与确定性结算。

### 必须成立

- clone 次数按持久执行计数，达到上限后操作终态 `failed`，不因轮转或显式 retry 重置。
- 重启只产生一个持久意图，不重新 clone，也不重开终态。
- 轮转与结算只改调度簿记和结算状态，不删除已有 clone 执行或计数。

### 禁止结果

用轮转代替结算；重启后重新 clone；回滚程序版本后自动复活 `failed`。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 持久失败预算 | `TestCloneFailureExhaustsItsDurableBudget`：三次失败后操作 `failed`，次数来自持久执行而非内存状态 | Covered |
| 重试是新执行且保留历史 | `TestCloneFailureRetriesAsNewExecutionAndUnknownBlocks`：失败执行保留，显式 retry 成功为第二次执行 | Covered |
| 重启不再 clone | `TestRestartIsOneDurableIntentAndNeverClonesAgain`：重启保留 `baseCommitId`、只保留一条 clone 执行，重启操作成功结束 | Covered |
| 轮转/结算不重置计数 | `TestClaimSettlesRetryWaitCandidateAfterSettledCloneFailure`：操作带着一条已结算的 `clone_failed` 执行行，下一次领取按确定性拒绝结算；测试不查询 `clone_executions`，没有断言该执行行仍在或轮转前后的计数相等 | Partial（持久执行计数不因轮转或结算改变没有直接断言） |

### 决策依据

D2、D3、D4，不变量 4。
