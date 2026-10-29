# 调度串行、节点提交与恢复

依据[工作流执行根决策](../../../decisions/cloud/workflow/0-cloud-side-scheduler-owns-workflow-execution.md)（D1 条件②、D4、D6）。
本用例立项时 Cloud 没有工作流调度器：生产里的运行停在 `pending`，唯一会写节点状态的是开发夹具
`MockWorkflowRunSimulator`（[`internal/collab/collab.go`](https://github.com/ora-space/cloud/blob/7704089d453f88d66e06d462aafe554e03a11855/internal/collab/collab.go)），
而根决策明确模拟 trace 不能作为验收证据。

Cloud 侧调度器已在 [cloud#67](https://github.com/ora-space/cloud/pull/67)（认领、从已提交状态重算就绪集合、节点结果比较交换）、
[cloud#72](https://github.com/ora-space/cloud/pull/72)（按会话派发与结算）、[cloud#91](https://github.com/ora-space/cloud/pull/91)
（已派发会话的 Node 不可达时结算失败）、[cloud#100](https://github.com/ora-space/cloud/pull/100)（agent 运行的取消写入者）、
[cloud#108](https://github.com/ora-space/cloud/pull/108)（已派发会话的回合窗口，merge `a4a13e8`）、
[cloud#109](https://github.com/ora-space/cloud/pull/109)（工作流的取消写入者与有界终态，merge `27e7f48`）、
[cloud#113](https://github.com/ora-space/cloud/pull/113)（调度器崩溃接手，merge `9814cacd`）、
[cloud#119](https://github.com/ora-space/cloud/pull/119)（WorkflowRun 的 Workspace 释放终态与有界放弃，merge `df9799c5`）、
[cloud#120](https://github.com/ora-space/cloud/pull/120)（工作流取消的真实 HTTP 边界与 G-026 并发顺序证据，merge `eb5ae5b`）、
[cloud#140](https://github.com/ora-space/cloud/pull/140)（每 run 的排他事务权、过期 Controller 代的拒绝、一条 run 的 Workspace
归属唯一，merge `f837c454`）的堆叠里落地。下表把控制面规则、真实执行侧路径与部署级竞争分别登记，
不以单一派发者或单一终态写入者代替两个副本同步竞争取锁的直接证据。

本次补充证据固定到 cloud `1dc2a46afe20adcc6428316dd9959435599cbfa9`、desktop
`d91344097ac9a2d0f079585cb1d8fbffa8dd78f7`、cluster `28bac762bd42753a82bb142039dfca976c71fe5b`。
Phase A 工具已随 [cluster#51](https://github.com/ora-space/cluster/pull/51) 合入 main
[`562351d`](https://github.com/ora-space/cluster/commit/562351d)；Phase B 工具见
[cluster#54](https://github.com/ora-space/cluster/pull/54)（head [`e4a0882`](https://github.com/ora-space/cluster/commit/e4a0882)）。下文按报告 basename 与 WorkflowRun ID
定位既有报告结论，不把报告读取时界当作数据库提交时钟。剩余部署级竞争与真实执行侧缺口继续由
[ora-space/circus#665](https://github.com/ora-space/circus/issues/665) 承接。

## At Most One Serial Executor Advances A Workflow Run

- 风险：多副本或多请求同时推进同一个运行，两个执行者各自派发同一批就绪节点，同一运行的 Workspace 与节点结果互相踩踏。
- 前置状态：运行已创建并冻结身份；部署有多个 Cloud 副本；运行有多个可派发的就绪节点。
- 触发：两个副本同时收到推进请求；一次推进的租约或排他权在事务中途失效；同一运行内多个 agent 节点。
- 必须成立：同一 WorkflowRun 在同一时刻至多有一个串行执行者，排他权是跨进程的（按 run 取排他事务权，不依赖任何进程内状态）；首期同一运行内的多个 agent 节点串行执行，共用该运行的 Workspace；排他权随事务结束释放，不靠心跳维持。
- 禁止结果：两个副本同时派发、用进程内锁表当跨进程锁、把"并行执行的多个 agent 节点"当作优化、锁未释放时就开始第二批派发。
- 验证义务与证据：

| 义务 | 所需直接证据 | cloud 侧状态 |
| --- | --- | --- |
| 多副本下同一运行只有一个串行执行者 | 多副本真实 PostgreSQL：同步竞争的推进请求，检查只有一个副本派发、另一个要么等待要么幂等退出 | Partial |
| 同一运行内的多个 agent 节点串行且共用同一 Workspace | 真实执行侧：一个运行内两个 agent 节点的派发与回调顺序，Workspace 归属唯一 | Covered |

- 决策依据：D1 条件②（无状态调度、单 run 串行、多副本与租约），D4（同 run 内 agent 节点首期串行），不变量 3。
- desktop 参考实现（`8909f07`）一侧：**Partial** — 同一 run 只有一个推进者有直接证据：per-run 排他锁是 backend composition 里的 `KeyedResourceLocks`（[`crates/backend/src/git_cleanup/keyed_locks.rs`](https://github.com/ora-space/desktop/blob/8909f07/crates/backend/src/git_cleanup/keyed_locks.rs) 的 `exclusive_waits_for_shared_holders`），[`crates/backend/src/workflow/run/operations/tests.rs`](https://github.com/ora-space/desktop/blob/8909f07/crates/backend/src/workflow/run/operations/tests.rs) 的 `concurrent_resume_from_failure_reruns_the_failed_node_once`（两线程各持排他权，结果恰为 (resumed, refused)=(1,1)）与 `concurrent_cancel_and_complete_report_the_committed_outcome`。**但"同一 run 内多个 agent 节点串行"与参考实现相反**：desktop 在同一次调度 pass 里并行派发 sibling agent，并允许它们同时 in-flight（[`crates/application/src/workflow_run/engine/aggregator_tests.rs`](https://github.com/ora-space/desktop/blob/8909f07/crates/application/src/workflow_run/engine/aggregator_tests.rs) 的 `parallel_fan_in_uses_the_declared_priority`、`d2_tests.rs` 的 `failed_run_binds_in_flight_sibling_and_does_not_dispatch_join`），其文档也写明并行 agent 共用 Workspace 只提供时间窗 diff、不提供改动归属。首期串行是本决策相对参考实现的**偏离**，没有 desktop 证据可依托，只能靠 Cloud 自己的测试证明。
- 本次直接证据：**第 1 行 Partial，第 2 行 Covered**。控制面代表性证据保留：
  [`TestWorkflowAdvanceSkipsARunHeldByAnotherReplica`](https://github.com/ora-space/cloud/blob/1dc2a46afe20adcc6428316dd9959435599cbfa9/internal/core/workflow_run_exclusivity_db_test.go#L215)
  用独立连接池持有同一 run 的排他权，推进放弃该 run、不写、不建 Workspace，释放后正常推进；
  [`TestWorkflowAdvanceAdvancesAnotherRunWhileOneRunIsHeld`](https://github.com/ora-space/cloud/blob/1dc2a46afe20adcc6428316dd9959435599cbfa9/internal/core/workflow_run_exclusivity_db_test.go#L255)
  证明另一个 run 仍可推进；
  [`TestWorkflowRunRightIsReleasedWhenTheTransactionAborts`](https://github.com/ora-space/cloud/blob/1dc2a46afe20adcc6428316dd9959435599cbfa9/internal/core/workflow_run_exclusivity_db_test.go#L329)
  证明事务回滚释放排他权。这里引用的是既有隔离 PostgreSQL 通过结果，本次没有重跑。
  部署场景 a（run `ccf3c0a2-3d5d-419a-bda1-738b9f1974a8`；`workflow-phase-a.json`、
  `cloud-race-postgres-proof-v2.json`、`statement-v2-analysis.json`）观察到恰好一次派发、单一终态写入者及两个副本的受保护读取，
  但未观察到两个副本同步争抢同一 run 时的 `pg_try_advisory_xact_lock = false`，故第 1 行仍为 `Partial`。
  第 2 行由 run `80280628-cb66-43f7-9223-7b4ddaf05a2d` 的 `workflow-phase-b.json` 与
  `phase-b-timeline-summary.json` 直接覆盖：Agent1 的结果提交后才派发 Agent2；两个 agent 各有不同的 execution、各为 attempt 1 且只派发一次，
  共用唯一 Workspace `900367a9-ab30-4e6a-92a1-07daa60e7223`、generation 1，运行最终 `succeeded` 且 `released`。
- 仍缺：部署级同步竞争的失败取锁分支；单一派发及受保护读取不能替代该分支的直接观察。

## A Node Result Commit Is A Compare And Swap And Late Callbacks Are No-ops

- 风险：重复或迟到的回调推进图，或者旧执行代的回调被当成当前结果，同一节点被结算两次、后续节点被派发两次。
- 前置状态：节点已派发并带有派发时的执行代；执行侧可能重发回调，或在代已过期后回调。
- 触发：同一结果重复提交；代过期（新的派发或控制代已经产生）后提交；两个回调几乎同时到达。
- 必须成立：节点结果提交带 from→to 的当前状态，是状态比较交换；重复或迟到的回调是空操作，不改变节点状态、不推进图；回调携带派发时的执行代，代过期的回调被拒绝；被拒绝的回调不产生任何副作用。
- 禁止结果：无条件覆写节点状态、把过期的回调当当前结果、重复回调触发第二次派发、拒绝回调时留下部分写入。
- 验证义务与证据：

| 义务 | 所需直接证据 | cloud 侧状态 |
| --- | --- | --- |
| 结果提交是比较交换，顺序重复提交为空操作 | 真实 PostgreSQL + 执行侧：先完成首次提交，再重放同一执行结果，检查状态只前进一次 | Covered |
| 真正并发提交同一节点结果只前进一次 | 真实 PostgreSQL：重叠提交的允许结果集合，检查只有一次状态迁移与后继派发 | Partial |
| 回调受派发代约束，过期代被拒绝且无副作用 | 真实执行侧：延迟到过期代之后的回调被拒，节点状态与派发记录都不变 | Partial |

- 决策依据：D1 条件②（节点结果提交是状态比较交换、回调带执行代），不变量 3。
- desktop 参考实现（`8909f07`）一侧：**Covered** — 重复完成是空操作（[`crates/backend/src/workflow/run/engine.rs`](https://github.com/ora-space/desktop/blob/8909f07/crates/backend/src/workflow/run/engine.rs) 的 `duplicate_completion_reports_are_idempotent_no_ops`）；被替换 attempt 的迟到回调与重复唤醒不改变状态、不派发（[`crates/backend/src/workflow/run/retry_tests.rs`](https://github.com/ora-space/desktop/blob/8909f07/crates/backend/src/workflow/run/retry_tests.rs) 的 `late_callbacks_and_repeated_wakes_are_no_ops_and_concurrent_waits_fire_independently`）；非 Running 节点完成返回 `NotRunning` 且不会派发 join（[`crates/application/src/workflow_run/engine/d2_tests.rs`](https://github.com/ora-space/desktop/blob/8909f07/crates/application/src/workflow_run/engine/d2_tests.rs) 的 `failed_run_binds_in_flight_sibling_and_does_not_dispatch_join`）。契约写在 [`crates/application/src/workflow_run/engine/ports.rs`](https://github.com/ora-space/desktop/blob/8909f07/crates/application/src/workflow_run/engine/ports.rs) 的 `transition_node_run_status` 注释里（仅当当前状态恰为 `from` 才转移）。仍缺：DB 层没有"from 不匹配则不改变"的直接测试，行为证据在引擎与集成层。
- 本次直接证据：**顺序重复 Covered，真正并发 Partial，真实执行侧过期代 Partial**。
  顺序重复由 run `0fb3b602-dbf9-4fe6-ac3f-4fdd86a4e548` 的 `workflow-phase-b.json` 与
  `duplicate-offline-verdict.json` 覆盖：重放同一 execution、sequence 5 与 submission 后，结果、receipts、sequence、
  `updated_at` 不变，仍只有一条工作项与一条 execution。首次上游成功在 05:16:04.629982，重试转发在
  05:16:14.875602，二者不重叠，因此不作为真正并发证据。原 CLI 因旧动作名误报 `inconclusive`，修正动作名后的
  离线判定为 `succeeded`，不改变这条顺序边界。
  [cloud#72](https://github.com/ora-space/cloud/pull/72) 的 `TestWorkflowSessionLateReceiptIsANoOp` 与
  `TestWorkflowSessionDispatchIsNotRepeatable` 保留为控制面重复回执及重复派发证据。
  [cloud#91](https://github.com/ora-space/cloud/pull/91) 的 `TestWorkflowSessionResultAfterTheNodeSettledIsKept`
  证明节点已按 Node 不可达结算后，晚到会话结果可以保留在 execution 的 `result`，同时不改写运行状态、错误与版本；
  这不是“所有行拒绝写入”，也不代表本次真实栈已经观察到晚到结果。
  过期代的控制面规则由 [cloud#140](https://github.com/ora-space/cloud/pull/140) 的
  `TestWorkflowSessionResultFromAnExpiredControllerEpochIsRejected` 直接证明：旧代请求以 409 `stale_controller`
  被拒且没有结果、回执或提交记录，运行不变；当前代仍可按相同 execution 身份提交并结算。它没有覆盖真实
  Controller→Node 往返中的旧代回调。
- 仍缺：真正并发提交同一节点结果的直接证据；真实执行侧旧代回调未触发，当前没有不修改产品或 SQL 的公开 trigger 入口。

## Recovery Recomputes The Ready Set From Committed State

- 风险：调度器把图状态放在内存里，副本崩溃或重启后进程内状态丢失，已完成的节点被重跑、就绪节点再也派发不出去，或运行停在中间状态无人推进。
- 前置状态：运行有已提交结果的节点、在途未提交结果的节点，以及尚未就绪的节点；调度副本随时可能崩溃。
- 触发：推进过程中副本崩溃；崩溃后另一个副本（或重启后的同一副本）接手；派发已发出但结果未知。
- 必须成立：每次推进都从 Cloud 已提交的状态重算已完成、在途与就绪集合，不缓存图状态；副本崩溃后不丢失进度，接手者从已提交状态继续；已经提交结果的节点不重复派发；**恢复接手不等于中断**——在途没有结果的节点继续持有，由这次执行自己的结果结算，Node 失联（`node_unavailable`）或超过回合窗口仍无结果（`session_turn_deadline_exceeded`）时才结算为失败，持有不会无限期；没有已提交结果的节点绝不记为成功；Cloud 重启不推进派发时的执行代，重启前派发的会话其结果在重启后仍按同一执行身份被接受。
- 禁止结果：把内存中的进度当成权威、崩溃后重跑已提交的节点、崩溃后运行永久卡在中间状态、把"派发过"当作"已完成"、把恢复接手当作中断而误杀仍在正常执行的会话、让在途节点无限期持有而没有失败判定、把重启前派发的会话结果当作过期而拒绝。
- 验证义务与证据：

| 义务 | 所需直接证据 | cloud 侧状态 |
| --- | --- | --- |
| 就绪集合只从已提交状态重算 | 真实 PostgreSQL：以任意中间状态作为起点（等价于崩溃后接手），重算结果与线性推进一致 | Covered |
| 崩溃接手不重跑已提交节点、不丢进度 | 真实执行侧：推进中途杀死调度副本后由另一副本接手，检查派发集合与最终结果 | Covered |

- 决策依据：D1 条件②（每次推进从已提交状态重算、恢复接手不等于中断），D4（崩溃恢复只依据已提交的节点结果），不变量 4。
- desktop 参考实现（`8909f07`）一侧：**Covered** — 分支激活与就绪集合只从持久化行推导（[`crates/application/src/workflow_run/engine/branch_projection.rs`](https://github.com/ora-space/desktop/blob/8909f07/crates/application/src/workflow_run/engine/branch_projection.rs) 的 `selected_branch_activates_only_the_matching_successor`、`fan_in_with_an_inactive_branch_still_becomes_ready`、`unresolved_condition_leaves_successors_unreached` 等），并且经编码 → 真实 SQLite 重载 → 再解析后执行仍成立（[`crates/backend/src/workflow/run/aggregator_persistence_tests.rs`](https://github.com/ora-space/desktop/blob/8909f07/crates/backend/src/workflow/run/aggregator_persistence_tests.rs) 的 `persisted_branch_outputs_reach_downstream`、`persisted_false_value_reaches_downstream`）；崩溃后只重做失败节点与其后继、已成功 sibling 保留（[`crates/backend/src/workflow/run/resume_tests.rs`](https://github.com/ora-space/desktop/blob/8909f07/crates/backend/src/workflow/run/resume_tests.rs) 的 `resume_from_failure_after_orphaned_crash_recovery`、`resume_from_failure_redoes_failed_join_and_finishes`）。说明：desktop 的串行执行者是进程内的，跨进程多副本不适用，这一条的多副本部分在参考实现里没有对应证据。
- 本次直接证据：**两行均 Covered**。就绪集合的既有代表性证据为
  [cloud#67](https://github.com/ora-space/cloud/pull/67) 的 `TestWorkflowProjectionLinearGraph`、
  `TestWorkflowAdvanceSucceedsLinearRun` 与 `TestWorkflowAdvanceIsIdempotent`；
  [cloud#113](https://github.com/ora-space/cloud/pull/113) 的 `TestWorkflowAdvanceResumesFromCommittedStateAfterRestart`
  从已提交中点以第二个 `Store` 推进，保留已提交分支输出、不重跑已提交节点并正常收尾。
  另一副本的真实接手由场景 c 覆盖（run `a44dd496-bd7f-4911-a0e1-7604a3673709`；`workflow-phase-a.json`、
  `c-analysis.json`、`dispatch-statement-supplement.json`）：推进副本被 SIGKILL 后，幸存副本的归因直接来自
  `source=postgres-commit`，保留同一 execution 身份，从已提交进度继续推进且不重派已提交节点，
  不依赖隔离窗口的空白推断谁接手。
  在途会话保留由 cloud#113 的 `TestWorkflowAdvanceResumesAnInFlightSessionFromItsOwnResult`、
  `TestWorkflowAdvanceHoldsAProductionSessionAcrossRestart` 与运行 #377 的 `restart-takeover` 覆盖：
  没有结果时保持运行、不重派，原执行自己的结果到达后才结算。
- 仍缺：两个副本同步竞争取锁的部署级证据另列于串行执行者义务，不降低已经有直接提交归因的接手义务。

## A Node With No Committed Result Is Never Recorded Successful

- 风险：未知结果被当作成功，运行的终态谎报了一次没有发生或没有留下结果的执行；下游节点基于不存在的输出继续跑。
- 前置状态：节点已派发，回调未到、结果丢弃，或进程在写入结果前消失。
- 触发：执行侧超时、结果响应丢失、运行被中断后恢复、回调从未到达。
- 必须成立：没有已提交结果的节点永不被记为成功；恢复接手不改变这一点——在途无结果的节点由这次执行自己的结果结算，Node 失联或超过回合窗口仍无结果时结算为失败，失败随运行终态对外可见；只有提交了结果的成功才允许推进下游节点。
- 禁止结果：把"已派发"或"未知"记成成功、用 HTTP 200 或夹具 trace 代替真实结果、让在途节点无限期持有或在 Node 已失联、超过回合窗口后仍不结算、把失败改写成成功以让运行通过。
- 验证义务与证据：

| 义务 | 所需直接证据 | cloud 侧状态 |
| --- | --- | --- |
| 无已提交结果的节点不进入成功状态，且不执行下游 Output | 真实 PostgreSQL + 执行侧：丢弃回调、超时与中断后的节点状态、运行终态与 Output 派发集合 | Covered |
| 运行终态不早于最后一个节点的结果提交 | 真实执行侧：运行进入终态的时点与节点结果提交顺序 | Covered |
| 无结果失败后真实迟到结果不改写节点与运行终态 | 真实执行侧：恢复回调传输后观察真实迟到结果及节点、运行与 execution 的变化 | Missing |

- 决策依据：D1 条件②（在途未提交结果的节点由这次执行自己的结果结算，Node 失联或超过回合窗口时结算为失败、绝不记为成功），D4（崩溃恢复），D7（首期节点产出落在节点状态里），不变量 5。
- desktop 参考实现（`8909f07`）一侧：**Covered** — 恢复把在途节点结算为 `interrupted_by_restart` 并让运行失败，已成功的节点保持成功（[`crates/backend/src/workflow/run/operations/tests.rs`](https://github.com/ora-space/desktop/blob/8909f07/crates/backend/src/workflow/run/operations/tests.rs) 的 `recovery_fails_interrupted_turns_and_resumes_stalled_runs`、[`crates/backend/src/workflow/run/recovery.rs`](https://github.com/ora-space/desktop/blob/8909f07/crates/backend/src/workflow/run/recovery.rs) 的 `an_interrupted_outer_row_still_fails_the_run` 与 `a_crashed_round_inside_a_fail_iteration_fails_the_node_and_the_run`、[`crates/backend/src/workflow/run/resume_gap_tests.rs`](https://github.com/ora-space/desktop/blob/8909f07/crates/backend/src/workflow/run/resume_gap_tests.rs) 的 `boot_sweep_interrupted_node_resumes_as_attempt_two`）。说明：desktop 的串行执行者与进程同生死，进程崩溃就是执行中断，所以参考实现把在途节点结算为中断失败；Cloud 是无状态调度器，重启不打断已派发的会话，在途节点按这次执行的 Node 失联（`node_unavailable`）或回合窗口（`session_turn_deadline_exceeded`）结算为失败，两者都不把未知记为成功。
- 本次直接证据：**无结果不成功且不执行 Output、终态时序均 Covered；真实迟到结果 Missing**。
  run `319004d6-d1b1-44e7-8366-900a7a25c1cc` 的 `workflow-phase-b.json` 与 `lost-offline-verdict.json`
  覆盖无结果失败路径：Controller 暂停时 `result` 为 NULL、只有 4 条 receipt，EndSession 排队；原有 300 秒
  end-confirm 窗口到期后节点与运行 `failed`，下游 Output 未执行。报告整体 `inconclusive` 仅因没有真实迟到结果，
  不抹去已直接观察到的非成功与 noOutput 结果。恢复后 181 秒内没有出现迟到终态 receipt，因此不能声称迟到结果
  已被拒绝，也不能声称没有任何副作用。
  终态顺序见 `phase-a-terminal-timelines.json` 与 `phase-b-timeline-summary.json`；这些报告的读取时界不是 COMMIT
  时钟，需结合固定 cloud 提交的
  [`internal/core/workflow_scheduler.go:679–682`](https://github.com/ora-space/cloud/blob/1dc2a46afe20adcc6428316dd9959435599cbfa9/internal/core/workflow_scheduler.go#L679-L682)：
  Output 与 run 终态在同一次 CAS 中原子写入。读取到最终节点结果及随后终态的真实轨迹与该提交路径共同支持第 2 行，
  不把轮询先后当作精确提交时间差。
  控制面代表性证据保留：`TestWorkflowSessionNodeGoneSettlesTheNodeFailed`、
  `TestWorkflowSessionTurnDeadlineFailsNode`、`TestWorkflowAdvanceHoldsAProductionSessionAcrossRestart` 分别覆盖
  Node 失联、回合超窗与恢复后无结果仍持有；`TestWorkflowSessionResultAfterTheNodeSettledIsKept` 与
  `TestWorkflowSessionTurnDeadlineResultAfterTheNodeSettledIsKept` 允许晚到结果保留在 execution，
  同时不改写节点与运行终态。它们是 PostgreSQL 接缝证据，不是本次真实栈的迟到结果观察。
- 仍缺：真实执行侧迟到结果的观察（本次 `not observed`）；该独立缺口不降低已经覆盖的无结果失败、noOutput 与终态时序义务。

## Cancellation Is A Recorded Request With A Bounded Terminal State

- 风险：取消请求被当成可立即生效的终态，正在收尾的资源与执行失去归属；或者根本没有取消入口，用户只能看着运行跑完。
- 前置状态：运行处于可执行状态并已派发节点；运行拥有 Workspace 等需要释放的资源。
- 触发：发起者请求取消；同一运行在取消请求后又收到节点回调；取消写入者与清理路径并发。
- 必须成立：取消先记录请求，运行进入有界的收尾路径，实际终态在清理完成后才写入；清理期间到达的回调不把运行拉回执行中；首期必须有工作流的取消写入者，agent 运行侧复用同一条收尾路径并一并补上；取消不泄漏运行的 Workspace。
- 禁止结果：收到请求就直接写终态、运行卡在收尾状态、清理后仍留资源、取消后回调把运行改回 `running`、"没有取消入口"被当成首期可接受状态。
- 验证义务与证据：

| 义务 | 所需直接证据 | cloud 侧状态 |
| --- | --- | --- |
| 取消是请求加有界终态，终态在清理后写入 | 真实 PostgreSQL + 执行侧：取消与清理、回调的并发顺序与最终行状态 | Covered |
| 工作流与 agent 两侧都有取消写入者 | 真实 HTTP 入口：两侧的取消请求都落到持久状态并走同一条收尾路径 | Covered |
| 两个副本同步竞争推进同一正在取消的运行 | 部署级真实 PostgreSQL：同窗推进及失败取锁分支，只有一个终态写入者且另一副本安全退出 | Partial |

- 决策依据：D4（取消是请求加有界终态），D6（取消写入者纳入首期，agent 侧一并补上），不变量 3；继承 [issue-run 根决策](../../../decisions/cloud/issue-run/0-agent-run-in-disposable-isolated-workspace.md) D6。
- desktop 参考实现（`8909f07`）一侧：**Covered** — 取消是有界终态而非即时终态：清理失败时仍提交 `Cancelled`（[`crates/backend/src/workflow/run/operations/tests.rs`](https://github.com/ora-space/desktop/blob/8909f07/crates/backend/src/workflow/run/operations/tests.rs) 的 `cancellation_commits_even_when_session_cleanup_fails`）、取消与完成并发时以已提交结果为准（同文件的 `concurrent_cancel_and_complete_report_the_committed_outcome`）、取消后迟到唤醒是空操作（[`crates/backend/src/workflow/run/retry_tests.rs`](https://github.com/ora-space/desktop/blob/8909f07/crates/backend/src/workflow/run/retry_tests.rs) 的 `cancel_during_the_wait_cancels_immediately_and_the_late_wake_is_a_no_op`）、已取消运行不能被清理复活（[`crates/backend/src/workflow/run/loop_exit_tests.rs`](https://github.com/ora-space/desktop/blob/8909f07/crates/backend/src/workflow/run/loop_exit_tests.rs) 的 `loop_exit_cleanup_cannot_revive_a_cancelled_run`）。desktop 侧缺的是入口本身：取消由本地界面/命令触发，Cloud 需要的 HTTP 写入者没有参考实现。
- 本次直接证据：**前两行 Covered，第 3 行 Partial** — 第 2 行的 agent 侧已就位：[cloud#100](https://github.com/ora-space/cloud/pull/100)（merge `f405823e`）落地
  `POST /tenants/:tid/issues/:iid/runs/:rid/cancel`。真实 HTTP + PostgreSQL 的 `TestCancelRunRecordsTheRequest` 断言请求只记录
  `issue_runs.cancel_requested_at`、返回 202 与记录时刻，运行的 phase、status、Thread 状态与 Workspace 都不动；
  `TestCancelRunIsIdempotentUnderTheSameKey` 与 `TestCancelRunKeepsTheFirstInstantAcrossFreshKeys` 断言重复请求返回同一时刻且不再写入；
  `TestCancelRunOnlyAcceptsTheAgentExecutionWindow`、`TestCancelRunIsNotFoundOutsideItsScope` 与 `TestCancelRunAuthorizesLikeEndingTheThread`
  断言越过 agent 执行窗口、越界与越权的调用者被拒；`TestCancelledRequestEndsTheDeclaredSession` 与
  `TestCancelledRequestReleasesTheUndeclaredSession` 断言端点写入的请求正是 `ReactToCancelledAgentRunsOnce` 消费的事实，
  分别落到 EndSession{cancelled} 与 `deliveryState=skipped` 的释放。工作流侧已由 [cloud#109](https://github.com/ora-space/cloud/pull/109)
  （merge `27e7f48`）落地 `POST /api/v1/tenants/:tid/workflows/:wfid/runs/:rid/cancel`；
  `TestWorkflowRunCancelRequestRecordsTheInstant` 断言请求只写 `workflow_runs.cancel_requested_at`、
  返回 202 与数据库记录的时刻，运行保持 `pending` 且 `started_at` 为空，换幂等键重复请求返回同一时刻；
  `TestWorkflowRunCancelRequiresTheInitiatorOrAnAdmin` 断言有冻结执行者绑定时非发起者的普通成员被拒
  （`workflow_run_cancel_forbidden`）且不落库，发起者可以取消；`TestWorkflowRunCancelAfterTheRunSettledIsRefused` 断言终态运行
  返回 `run_not_cancellable` 而不是被记录；`TestWorkflowRunCancelRacesToOneRequest` 断言两个并发请求只记录一个时刻、只发生一次
  请求引起的版本变化。`TestWorkflowRunCancelBeforeStartNeverStarts` 断言未开始的运行在推进时直接结算 `cancelled`：不认领、
  不写 `started_at`、不建 Workspace、不派生删除操作，请求在结算后保留且后续推进不再改动终态运行；
  `TestWorkflowRunCancelStopsTheRunningSession` 断言在途会话经既有 EndSession 中继停止（`end_session`、`reason=cancelled`，
  指向该运行的执行），重复推进是重放而不是第二条请求，会话终态后节点与运行结算 `cancelled` 并释放 Workspace；
  `TestWorkflowRunCancelEndWindowSettlesCancelled` 断言结束请求超过 `AgentEndConfirmTimeout` 仍未获确认时，节点与运行结算
  `cancelled`、Workspace 释放，之后到达的结果只保留在执行行、不改写终态；`TestWorkflowRunCancelKeepsACommittedSessionResult`
  与 `TestWorkflowRunCancelReusesTheExistingEndRequest` 断言已提交结果与会话自己上报的终态优先于取消，运行仍结算 `cancelled`
  并释放 Workspace；`TestWorkflowRunCancelWinsOverAnUnreachableNode` 断言 Node 不可达时由取消而不是 `node_unavailable` 决定
  终态，且不再向已消失的 Node 释放结束请求，Workspace 照常释放。
  [cloud#120](https://github.com/ora-space/cloud/pull/120)（merge `eb5ae5b`）在真实 HTTP + PostgreSQL 上补齐工作流侧的入口与顺序证据：`integration/workflow_run_cancel_test.go` 的 `TestWorkflowRunCancelRecordsOnlyTheRequest` 断言响应只有 `cancelRequestedAt` 且等于库中时刻、版本恰好 +1、其余行逐列不变、不产生 Workspace/执行工作项/Thread 命令/派生操作，同键与换键重放都返回首次时刻且不再写，未知字段 400；`TestWorkflowRunCancelSettlesThroughTheScheduler` 断言 HTTP 写入的请求被 `AdvanceWorkflowRunsOnce` 消费，pending 运行直接结算 `cancelled`（不写 `started_at`、不建 Workspace、不派生删除），S5 释放为 `not_required` 且没有释放时刻，结算后重放仍返回原时刻、后续推进不改版本；`TestWorkflowRunCancelAuthorizesTheInitiatorOrAnAdmin` 断言发起者/管理员 202、普通成员 403 `workflow_run_cancel_forbidden`、非成员 403 `membership_required`、无绑定时沿用成员规则；`TestWorkflowRunCancelIsNotFoundOutsideItsScope` 断言跨租户、跨工作流、未知 id、非 UUID 与软删除运行都返回 404 且不落库；`TestWorkflowRunCancelAfterSettlementIsRefused` 断言 succeeded/failed/cancelled/awaiting_input 都返回 409 `run_not_cancellable` 且不落库；`TestWorkflowRunCancelRacesToOneRequest` 断言两个并发请求只记录一个时刻、只有一次版本变化。同 PR 的 `internal/core/workflow_run_cancel_order_db_test.go` 在真实 PostgreSQL 上让两个写入者并发：`TestWorkflowRunCancelRacesTheSchedulerPass` 断言取消与调度推进竞争后只释放一条 `end_session{cancelled}`、运行保持 `running` 直到会话自己上报，最终 `cancelled`、节点 `cancelled`、删除声明恰好一条、释放为 `releasing`、请求保留；`TestWorkflowRunCancelAndTheSessionCallbackRace` 断言两种允许顺序——请求先提交则运行 `cancelled` 且节点保留实际运行的 `idle_timeout` 结果，回调先结算则请求 409 `run_not_cancellable` 且不落库、运行 `succeeded`——每例都保留已提交结果并声明释放；`TestWorkflowRunCancelWindowAndALateCallbackRace` 断言结束确认窗口到期与迟到回调竞争时运行必为 `cancelled`，节点要么保留自己的 `idle_timeout` 结果、要么记为 `cancelled`，迟到结果始终留在执行行，释放为 `releasing`，终态后推进不改版本。真实执行侧的取消时点由运行 #347 的隔离栈验收覆盖（cloud `27e7f48`、desktop `e19d752`）：run `4312baa9-6ca2-431f-879d-7e6a88adac9d` 在 `running` 时两次 `POST …/cancel` 都返回 202 且 `cancelRequestedAt` 同为 `13:44:50.207Z`，`end_session` 于 `13:44:53.811Z` 创建、`13:44:53.984Z` 送达，执行 `b4143a86-…` 于 `13:44:55.256Z` 以 `agent_session_ended / cancelled` 终止，运行于 `13:45:03.829Z` 以 `cancelled` 结束，Workspace 于 `13:45:05.084Z` 变为 `deleted/deleted`、容器消失；同结论在独立栈 `ora-wf347` 复跑。
  第 1 行余下的"真实执行侧并发顺序"由运行 #507 的 `cancel-race` 场景补上：真实栈上节点进入 `running` 后发
  `POST /api/v1/tenants/:tid/workflows/:wfid/runs/:rid/cancel`，run `4050b211-a1d4-4e07-b18e-6fbd97508be1` 在取消请求
  （`cancel_requested_at` 03:58:35.9）之后仍收到执行侧回调写进 `node_executions`（1 条，最后更新 03:58:46.9），agent
  节点保留 `cancelled` 结果、已提交节点的输出仍在，运行以 `cancelled` 结束，Workspace 的释放记为 `released` 且释放时刻
  等于 Workspace 自己的 `deleted_at`。这条证据采自 cluster PR #42 合并前（merge `d75b92e`）的工具版本，run id 为
  `4050b211`；合并后的工具版本复跑了同组其余场景，这一场没有再次取到相同结论（细节见运行 #507）。
- 部署竞争补充证据：场景 b（run `8c0982cc-001d-4fbb-b28f-6e7cff0db0f3`；`workflow-phase-a.json`、
  `b-analysis.json`、`dispatch-statement-supplement.json`）观察到恰好一次派发、单一终态写入者及两个副本的受保护读取，取消约 10 秒收敛，
  但没有观察到两个副本同步竞争同一 run 时 `pg_try_advisory_xact_lock = false`。因此新增的第 3 行为 `Partial`，
  不以单一终态写入者替代锁竞争证据，也不降低既有取消入口及取消/清理/回调顺序两项 `Covered`。
- 仍缺：第 3 行的部署级同步竞争失败取锁分支，由 [ora-space/circus#665](https://github.com/ora-space/circus/issues/665) 承接。

## The Workspace Release Is A Terminal Fact With A Bounded Give-Up

- 风险：运行进入终态后 Workspace 的归属无人记账——删除操作丢失或 Node 失联后无限期挂起，沙盒泄漏；或者反过来，Cloud 在没有确认删除的情况下报告 Workspace 已释放，事后无法判断哪些资源真的回收了。
- 前置状态：运行已进入终态（成功、失败或取消）且绑定了运行 Workspace；删除由 Controller 的 `delete_workspace` 工作项执行；Node 可能失联、被暂停，或已随沙箱终止。
- 触发：终态结算的同一事务登记释放意图并声明删除操作；删除完成回执结算操作；Node 在释放窗口内不可达；Workspace 被另一条路径先行删除（运维或数据库直接删除——公开的 `DELETE /projects/:id` 在运行持有 Workspace 期间会被拒，见下）。
- 必须成立：终态运行不能把 Workspace 的去向留成未知——结算事务里同时写释放意图与删除声明；只有删除完成的回执才把释放记为 `released` 并写入释放时刻，且与 Workspace 的 `deleted_at` 在同一事务提交；没有 Workspace 的运行记为 `not_required`，不得与 `released` 混为一谈；Node 在窗口内或有心跳时保持 `releasing`，部署未配置窗口时不放弃；超过有界窗口才放弃为 `abandoned` + `workspace_unavailable`，保留仍存活的 Workspace 行与绑定，未完成的删除操作失败为 `node_unavailable`；已终止的沙箱不因 Node 不可达而被放弃；Workspace 已被另一条路径删除时，释放以该 Workspace 自己的删除时刻补记为 `released`；释放是单向终态，重复推进不重复声明删除、不改变已写入的释放状态。
- 禁止结果：终态运行留下 `pending` 或空的释放状态、把"已声明删除"当作"已释放"、没有删除回执就写 `workspace_released_at`、把无 Workspace 的运行记成 `released`、Node 失联后无限期等待、放弃时顺手删掉仍存活的 Workspace 行或丢失绑定、重复推进重复声明删除或改写已放弃的释放状态。
- 验证义务与证据：

| 义务 | 所需直接证据 | cloud 侧状态 |
| --- | --- | --- |
| 终态结算同时登记释放意图与删除声明，重放不重复声明 | 真实 PostgreSQL：终态事务后的释放行与删除操作数 | Covered |
| 只有删除完成回执才写 `released` 与释放时刻，且与 Workspace 删除同事务 | 真实 PostgreSQL：删除操作结算事务里释放行与 Workspace 删除时刻一致 | Covered |
| Node 不可达的放弃有界；窗口内、未配置与已终止沙箱都不放弃，且保留仍存活 Workspace 的归属 | 真实 PostgreSQL：窗口内外、无窗口、终止沙箱与重复推进后的释放状态 | Covered |
| 真实栈上的端到端释放与放弃 | 真实执行侧：普通成功与取消后的释放、Node 失联后的 `abandoned` 与删除操作失败、另一条路径删除后的补记，以及多 Controller 下删除操作只有一个完成者 | Partial |
| Workspace 由另一条路径删除后的端到端补记 | 控制面补记分支已有直接测试；固定候选未找到无需产品改动或 SQL 编辑的真实外部删除触发器，端到端补记未观察到 | Partial |

- 决策依据：D7（首期保证资源释放，不能泄漏沙箱），D3（运行 Workspace 归属在创建时冻结），D4（取消进入有界收尾、实际终态在清理后写入），不变量 2；[issue-run 根决策](../../../decisions/cloud/issue-run/0-agent-run-in-disposable-isolated-workspace.md) D4 的交付后释放语义对工作流首期不适用，只保留资源释放。
- desktop 参考实现（`8909f07`）一侧：没有等价的持久释放事实。desktop 的 Workspace 是本地资源，运行终态的清理只覆盖会话与本地目录，相关失败语义见 [`crates/backend/src/workflow/run/operations/tests.rs`](https://github.com/ora-space/desktop/blob/8909f07/crates/backend/src/workflow/run/operations/tests.rs) 的 `cancellation_commits_even_when_session_cleanup_fails` 与 [`crates/backend/src/workflow/run/loop_exit_tests.rs`](https://github.com/ora-space/desktop/blob/8909f07/crates/backend/src/workflow/run/loop_exit_tests.rs) 的 `loop_exit_cleanup_failure_fails_closed`；它没有 Cloud 删除操作契约下的"释放终态 + 有界放弃"，不能为这张表提供证据。
- 本次直接证据：**Covered**（前 3 行）— [cloud#119](https://github.com/ora-space/cloud/pull/119)（merge `df9799c5`）的
  `internal/core/workflow_run_release_db_test.go` 在隔离 schema 的真实 PostgreSQL 上按生产路径驱动这些迁移：第 1 行由
  `TestWorkflowRunCleanupRecordsTheReleaseIntent` 覆盖——终态清理在结算事务里写 `releasing`、不写释放时刻、不删除 Workspace，
  只声明恰好一条删除操作；同一清理重放不改变释放行与版本，也不声明第二条操作。第 2 行由
  `TestWorkflowRunDeleteCompletionRecordsReleased` 覆盖——经控制面自己的 `advance` 走删除完成步骤，断言 `released` 与释放时刻
  和 Workspace 的 `observed_state='deleted'`、`deleted_at` 在同一事务提交，重放该钩子不再写，释放只带来一次版本变化；
  `TestWorkflowRunDeleteCompletionRefusesANonTerminalRun` 断言仍在执行的运行收到删除完成时不写任何释放状态；
  `TestWorkflowRunWithoutWorkspaceRecordsNotRequired` 断言从未建立 Workspace 的运行记 `not_required` 且没有释放时刻。第 3 行由
  `TestWorkflowRunGiveUpAbandonsAnUnreachableWorkspace` 覆盖——心跳超过 30 分钟窗口后，释放变为 `abandoned` +
  `workspace_unavailable`、没有释放时刻，未完成的删除操作失败为 `node_unavailable`，Workspace 行仍然存活且绑定不变，第二次推进
  不改写释放状态也不重复声明删除；`TestWorkflowRunReleaseHoldsWithinTheWindowAndWithoutOne` 断言窗口内的心跳与未配置窗口都保持
  `releasing` 且删除操作仍在等待；`TestWorkflowRunReleaseDoesNotAbandonATerminatedSandbox` 断言沙箱已终止时保持 `releasing`；
  `TestWorkflowRunReleaseHealsAWorkspaceDeletedByAnotherPath` 断言被其他路径删除的 Workspace 由下一轮推进补记为 `released`，
  释放时刻取 Workspace 自己的 `deleted_at`。迁移 `0055_workflow_run_workspace_release.sql` 的
  `TestMigration0055WorkflowRunWorkspaceReleaseAppliesFreshAndUpgrades` 覆盖全新安装与升级回填：非终态保持 `pending`、无 Workspace 记
  `not_required`、已软删除记 `released`、其余记 `releasing`，并由两条 CHECK 保证 `released` 必有释放时刻、`abandoned` 必有原因。
  第 4 行已部分就位：运行 #377 的 `release-normal` 在真实栈上让普通成功的运行到达 `released`、写入释放时刻，Workspace 的
  `deleted_at` 已置且沙箱容器消失；`release-giveup` 暂停仍在运行的沙箱、让 Node 心跳过期，运行以 `node_unavailable` 失败后
  放弃释放为 `abandoned` + `workspace_unavailable`、没有释放时刻、Workspace 行保留，删除操作失败为 `node_unavailable`。
  运行 #507 的调度场景补上第 4 行余下的三段：`cancel-receipt` 在 run `7297dd7f-a379-40ed-8ea7-9efb316ac218` 上从
  Workspace 行出现起取消，恰好一条 `delete_workspace` 操作 succeeded、没有存活的 Workspace 行，`workspace_released_at`
  与 Workspace 的 `deleted_at` 同为 04:00:03.070601；`controller-replica` 在 run
  `e0c9355a-0a0c-4340-8230-d08d4f564826` 上把第二个 Controller（`ora-cluster-acceptance-controller-b`）接到同一个 store，
  与 `ora-cluster-controller-1` 同时在线，取消窗口里只有一条删除操作 `3d57546a-1ef9-4e4d-a8e8-e5be4559e298`，其完成代
  （8）等于完成时生效的租约代（8）。`healed` 场景（run `1549e8ab-01a0-4662-91f0-37033daf4bc7`）证实补记分支在真实栈上
  没有公开入口：脚本从 Workspace 行出现起每 0.25 秒重试 `DELETE /projects/:id` 直到运行自己释放，每一次都被拒（345 次：
  `409 runtime_control_held` 169 次、`409 operation_in_progress` 176 次），运行随后按调度器自己的删除结算为 `released`，
  没有发生补记。`TestWorkflowRunReleaseHealsAWorkspaceDeletedByAnotherPath` 直接证明控制面补记，
  该真实栈场景仅观察到公开删除入口被拒。
  固定候选的后续设计分析见 `backfill-drill-report.md`（代码分析，非新部署证据）：
  [`workflow_run_release.go:277–285、308–315`](https://github.com/ora-space/cloud/blob/1dc2a46/internal/core/workflow_run_release.go#L277-L315)
  要求 Workspace 已有 `deleted_at`，run 已终态且 release 仍为 `pending/releasing`，才以 Workspace 删除时刻补写 `released`。
  [`node.go:101–130`](https://github.com/ora-space/cloud/blob/1dc2a46/internal/core/node.go#L101-L130)
  将物理 Node 断连对应的 Workspace 标为 `unavailable`，不会写删除事实；正常 sandbox terminate 也不等同于 cleanup。
  Workspace 的软删除由 [`control.go:515–530、598–600`](https://github.com/ora-space/cloud/blob/1dc2a46/internal/core/control.go#L515-L600)
  的操作 cleanup 写入。独立 Workspace 删除入口排除运行 Workspace（[`store.go:742–745`](https://github.com/ora-space/cloud/blob/1dc2a46/internal/core/store.go#L742-L745)），
  项目删除要求 runtime control 为 idle（[`public.go:215–219`](https://github.com/ora-space/cloud/blob/1dc2a46/internal/core/public.go#L215-L219)），
  而运行持有期间控制权为 maintenance（[`runtime_control.go:129–140`](https://github.com/ora-space/cloud/blob/1dc2a46/internal/core/runtime_control.go#L129-L140)）。
  因而未运行不能建立软删除前提的容器故障，不把它登记为端到端补记。
  [`TestWorkflowRunReleaseHealsAWorkspaceDeletedByAnotherPath:333–343`](https://github.com/ora-space/cloud/blob/1dc2a46/internal/core/workflow_run_release_db_test.go#L333-L343)
  用 SQL 构造该前提，仅作控制面分支证据，本次没有重跑。
- 仍缺：真实栈上其他路径先提交 Workspace 删除事实、随后由调度器补记且不再声明删除的直接观察。第 4 行保留原有汇总状态，
  端到端补记单列 `Partial`：控制面补记已证明，物理容器删除不等于 Workspace 软删除，真实端到端路径未触发。承接事项为
  [ora-space/circus#665](https://github.com/ora-space/circus/issues/665)；不改变前 3 项已覆盖的释放义务。
