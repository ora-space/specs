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
归属唯一，merge `f837c454`）的堆叠里落地，下表因此不再全是 `Missing`。**多副本**现有控制面证据：cloud#140 用独立连接池的
第二个 `Store` 在同一个数据库上持有同一个 run 的排他权，锁被持有时推进放弃该 run、不写、不建 Workspace；部署级的一半由
运行 #507 补上：两个 Controller 接到同一个 store、在它们的窗口里取消运行，删除操作恰好一条、完成代等于完成时生效的租约代。
两个副本同时推进**同一个 run** 的竞争仍只在控制面层面观察过。**崩溃接手**中重启后的同一副本接手已由 cloud#113 的 DB 用例
与运行 #377 的真实重启验收覆盖，由另一副本接手仍缺部署级证据；其余要求"真实执行侧"而尚未覆盖的路径继续记 `Partial`，
整体属 D1 落地顺序第 3 步（ora-space/circus#70）。

## At Most One Serial Executor Advances A Workflow Run

- 风险：多副本或多请求同时推进同一个运行，两个执行者各自派发同一批就绪节点，同一运行的 Workspace 与节点结果互相踩踏。
- 前置状态：运行已创建并冻结身份；部署有多个 Cloud 副本；运行有多个可派发的就绪节点。
- 触发：两个副本同时收到推进请求；一次推进的租约或排他权在事务中途失效；同一运行内多个 agent 节点。
- 必须成立：同一 WorkflowRun 在同一时刻至多有一个串行执行者，排他权是跨进程的（按 run 取排他事务权，不依赖任何进程内状态）；首期同一运行内的多个 agent 节点串行执行，共用该运行的 Workspace；排他权随事务结束释放，不靠心跳维持。
- 禁止结果：两个副本同时派发、用进程内锁表当跨进程锁、把"并行执行的多个 agent 节点"当作优化、锁未释放时就开始第二批派发。
- 验证义务与证据：

| 义务 | 所需直接证据 | cloud 侧状态 |
| --- | --- | --- |
| 多副本下同一运行只有一个串行执行者 | 多副本真实 PostgreSQL：同步竞争的推进请求，检查只有一个副本派发、另一个要么等待要么幂等退出 | Covered |
| 同一运行内的多个 agent 节点串行且共用同一 Workspace | 真实执行侧：一个运行内两个 agent 节点的派发与回调顺序，Workspace 归属唯一 | Partial |

- 决策依据：D1 条件②（无状态调度、单 run 串行、多副本与租约），D4（同 run 内 agent 节点首期串行），不变量 3。
- desktop 参考实现（`8909f07`）一侧：**Partial** — 同一 run 只有一个推进者有直接证据：per-run 排他锁是 backend composition 里的 `KeyedResourceLocks`（[`crates/backend/src/git_cleanup/keyed_locks.rs`](https://github.com/ora-space/desktop/blob/8909f07/crates/backend/src/git_cleanup/keyed_locks.rs) 的 `exclusive_waits_for_shared_holders`），[`crates/backend/src/workflow/run/operations/tests.rs`](https://github.com/ora-space/desktop/blob/8909f07/crates/backend/src/workflow/run/operations/tests.rs) 的 `concurrent_resume_from_failure_reruns_the_failed_node_once`（两线程各持排他权，结果恰为 (resumed, refused)=(1,1)）与 `concurrent_cancel_and_complete_report_the_committed_outcome`。**但"同一 run 内多个 agent 节点串行"与参考实现相反**：desktop 在同一次调度 pass 里并行派发 sibling agent，并允许它们同时 in-flight（[`crates/application/src/workflow_run/engine/aggregator_tests.rs`](https://github.com/ora-space/desktop/blob/8909f07/crates/application/src/workflow_run/engine/aggregator_tests.rs) 的 `parallel_fan_in_uses_the_declared_priority`、`d2_tests.rs` 的 `failed_run_binds_in_flight_sibling_and_does_not_dispatch_join`），其文档也写明并行 agent 共用 Workspace 只提供时间窗 diff、不提供改动归属。首期串行是本决策相对参考实现的**偏离**，没有 desktop 证据可依托，只能靠 Cloud 自己的测试证明。
- 本次直接证据：**第 1 行 Covered，第 2 行 Partial** — 第 1 行由 [cloud#140](https://github.com/ora-space/cloud/pull/140)
  （merge `f837c454`）落地：WorkflowRun 事务先以共享模式取全局事务锁，再对 run 取 `pg_try_advisory_xact_lock`，
  取不到就退出本次 tick、不形成决策、不写、不发提示。`TestWorkflowAdvanceSkipsARunHeldByAnotherReplica` 让第二个
  `Store`（独立连接池、同一数据库）持有 run 排他权，断言推进放弃该 run、运行行与版本不变、不建 Workspace，锁释放后
  下一 tick 正常认领并建出恰好一个 Workspace；`TestWorkflowAdvanceAdvancesAnotherRunWhileOneRunIsHeld` 断言锁按 run
  而不是全局划分（另一个 run 在同一次推进里照常结算）；`TestWorkflowRunRightIsReleasedWhenTheTransactionAborts`
  断言事务回滚就释放排他权，不靠心跳。运行 #507 的 `controller-replica` 场景在部署层面补上另一半：两个 Controller
  同时接到同一个 store，取消窗口里的删除操作只有一条、完成代等于完成时生效的租约代。第 2 行仍为 `Partial`：
  [cloud#72](https://github.com/ora-space/cloud/pull/72) 的 `TestWorkflowSessionRunsTwoAgentNodesInOrder` 在真实
  PostgreSQL schema 上跑完同一运行内的两个 agent 节点，断言 a1 结算后 a2 才派发、a2 在途时运行保持 `running` 且下游
  `output` 不跑、两个节点各拿自己的执行与自己的结果（a2 的 `reason` 不是 a1 的）、每个 agent 节点恰好一条注册会话；
  cloud#140 在同一次运行里补上"两个节点落在同一个 Workspace"的断言（两次执行的 `workspace_id` 相等，`workspaces`
  恰好一行）。两者都在控制面接缝上，真实栈还没有一条让同一运行的两个 agent 节点真实执行的验收。
- 仍缺：同一运行两个 agent 节点的真实执行侧证据（控制面已断言两节点共用同一个 Workspace 行，运行 #507 的真实栈只跑过
  单 agent 节点）；排他权没有续租与失效窗口（锁就是事务本身），具体参数由实现给出；两个副本同时推进**同一个 run** 的
  竞争仍未在部署层面观察，属 D1 落地顺序第 3 步（ora-space/circus#70）。

## A Node Result Commit Is A Compare And Swap And Late Callbacks Are No-ops

- 风险：重复或迟到的回调推进图，或者旧执行代的回调被当成当前结果，同一节点被结算两次、后续节点被派发两次。
- 前置状态：节点已派发并带有派发时的执行代；执行侧可能重发回调，或在代已过期后回调。
- 触发：同一结果重复提交；代过期（新的派发或控制代已经产生）后提交；两个回调几乎同时到达。
- 必须成立：节点结果提交带 from→to 的当前状态，是状态比较交换；重复或迟到的回调是空操作，不改变节点状态、不推进图；回调携带派发时的执行代，代过期的回调被拒绝；被拒绝的回调不产生任何副作用。
- 禁止结果：无条件覆写节点状态、把过期的回调当当前结果、重复回调触发第二次派发、拒绝回调时留下部分写入。
- 验证义务与证据：

| 义务 | 所需直接证据 | cloud 侧状态 |
| --- | --- | --- |
| 结果提交是比较交换，重复与迟到回调为空操作 | 真实 PostgreSQL：并发与重复提交的允许结果集合，检查状态只前进一次 | Partial |
| 回调受派发代约束，过期代被拒绝且无副作用 | 真实执行侧：延迟到过期代之后的回调被拒，节点状态与派发记录都不变 | Partial |

- 决策依据：D1 条件②（节点结果提交是状态比较交换、回调带执行代），不变量 3。
- desktop 参考实现（`8909f07`）一侧：**Covered** — 重复完成是空操作（[`crates/backend/src/workflow/run/engine.rs`](https://github.com/ora-space/desktop/blob/8909f07/crates/backend/src/workflow/run/engine.rs) 的 `duplicate_completion_reports_are_idempotent_no_ops`）；被替换 attempt 的迟到回调与重复唤醒不改变状态、不派发（[`crates/backend/src/workflow/run/retry_tests.rs`](https://github.com/ora-space/desktop/blob/8909f07/crates/backend/src/workflow/run/retry_tests.rs) 的 `late_callbacks_and_repeated_wakes_are_no_ops_and_concurrent_waits_fire_independently`）；非 Running 节点完成返回 `NotRunning` 且不会派发 join（[`crates/application/src/workflow_run/engine/d2_tests.rs`](https://github.com/ora-space/desktop/blob/8909f07/crates/application/src/workflow_run/engine/d2_tests.rs) 的 `failed_run_binds_in_flight_sibling_and_does_not_dispatch_join`）。契约写在 [`crates/application/src/workflow_run/engine/ports.rs`](https://github.com/ora-space/desktop/blob/8909f07/crates/application/src/workflow_run/engine/ports.rs) 的 `transition_node_run_status` 注释里（仅当当前状态恰为 `from` 才转移）。仍缺：DB 层没有"from 不匹配则不改变"的直接测试，行为证据在引擎与集成层。
- 本次直接证据：**Partial** — 第 1 行已部分就位：[cloud#67](https://github.com/ora-space/cloud/pull/67) 的
  `TestWorkflowAdvanceIsIdempotent` 连跑三次推进，断言已结算的运行不被二次认领、版本不漂移；[cloud#72](https://github.com/ora-space/cloud/pull/72)
  的 `TestWorkflowSessionLateReceiptIsANoOp` 重放同一个终态回执，断言运行状态、节点状态与版本逐字节不变，`TestWorkflowSessionDispatchIsNotRepeatable`
  断言同一节点的工作项被派发消耗掉后重放派发是同一个事实而不是第二次会话；[cloud#91](https://github.com/ora-space/cloud/pull/91)（merge
  `068d6718`）的 `TestWorkflowSessionResultAfterTheNodeSettledIsKept` 覆盖另一种迟到回调：节点已被 Cloud 按 Node 不可达结算、
  运行已终态后，真实会话结果到达仍不改写运行状态、错误与版本，只保留在执行行的 `result` 上。第 2 行现为 **Partial**：
  [cloud#140](https://github.com/ora-space/cloud/pull/140)（merge `f837c454`）的
  `TestWorkflowSessionResultFromAnExpiredControllerEpochIsRejected` 在真实 PostgreSQL 上把执行派发在 epoch 1，让租约
  过期并由替换 Controller 取得 epoch 2，再用旧代提交同一执行的结果：请求以 409 `stale_controller` 被拒，执行没有写入
  结果、事件回执与提交记录都是 0 条、`last_event_sequence` 仍为 0，运行行逐字段不变；随后用当前代提交同一个执行身份
  被接受并正常结算，因为结算按执行身份（`operation_id` + `execution_id`）而不是按派发时的代。这条证据走控制面的 lease
  与 result 入口，不是真实 Controller→Node 的往返，因此记 `Partial`。
- 仍缺：两个回调**并发**提交同一节点结果的允许结果集合；过期代回调在真实执行侧（真实 Node 经 Controller 上报）被拒的用例。

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
| 崩溃接手不重跑已提交节点、不丢进度 | 真实执行侧：推进中途杀死调度副本后由另一副本接手，检查派发集合与最终结果 | Partial |

- 决策依据：D1 条件②（每次推进从已提交状态重算、恢复接手不等于中断），D4（崩溃恢复只依据已提交的节点结果），不变量 4。
- desktop 参考实现（`8909f07`）一侧：**Covered** — 分支激活与就绪集合只从持久化行推导（[`crates/application/src/workflow_run/engine/branch_projection.rs`](https://github.com/ora-space/desktop/blob/8909f07/crates/application/src/workflow_run/engine/branch_projection.rs) 的 `selected_branch_activates_only_the_matching_successor`、`fan_in_with_an_inactive_branch_still_becomes_ready`、`unresolved_condition_leaves_successors_unreached` 等），并且经编码 → 真实 SQLite 重载 → 再解析后执行仍成立（[`crates/backend/src/workflow/run/aggregator_persistence_tests.rs`](https://github.com/ora-space/desktop/blob/8909f07/crates/backend/src/workflow/run/aggregator_persistence_tests.rs) 的 `persisted_branch_outputs_reach_downstream`、`persisted_false_value_reaches_downstream`）；崩溃后只重做失败节点与其后继、已成功 sibling 保留（[`crates/backend/src/workflow/run/resume_tests.rs`](https://github.com/ora-space/desktop/blob/8909f07/crates/backend/src/workflow/run/resume_tests.rs) 的 `resume_from_failure_after_orphaned_crash_recovery`、`resume_from_failure_redoes_failed_join_and_finishes`）。说明：desktop 的串行执行者是进程内的，跨进程多副本不适用，这一条的多副本部分在参考实现里没有对应证据。
- 本次直接证据：**Partial** — 第 1 行 `Covered`，由两部分构成：就绪集合是纯函数，[cloud#67](https://github.com/ora-space/cloud/pull/67) 的
  `TestWorkflowProjectionLinearGraph`、`TestWorkflowProjectionPendingUpstreamWaits`、`TestWorkflowProjectionRootlessNodeIsInactive`、
  `TestWorkflowProjectionUnlabeledEdgeIsElse` 覆盖它的四类结果，`TestWorkflowAdvanceSucceedsLinearRun` 与
  `TestWorkflowAdvanceIsIdempotent` 证明推进每次都从行上的已提交状态重算、不缓存图状态（参考语义为 desktop 的
  `resume`，`engine.rs:328-330`："Recomputes the ready set from persisted state"）；[cloud#113](https://github.com/ora-space/cloud/pull/113)
  （merge `9814cacd`）的 `TestWorkflowAdvanceResumesFromCommittedStateAfterRestart` 再以崩溃留下的已提交中点为起点，
  用第二个 `Store`（同一配置、同一数据库，等价于重启后的新进程）推进：分支条件读到的是已提交的启动输出而不是重跑会
  产生的值，已提交节点没有被重跑，运行正常收尾，重复推进不再改动终态行。第 2 行 `Partial`：重启后的同一副本接手由
  cloud#113 的 `TestWorkflowAdvanceResumesAnInFlightSessionFromItsOwnResult`、`TestWorkflowAdvanceHoldsAProductionSessionAcrossRestart`
  与运行 #377 的 `restart-takeover` 场景共同覆盖，义务要求的"由另一副本接手"没有直接证据：崩溃留下已注册但无结果的执行与未登记工作项，接手者不发明结果、
  不重派、运行与节点保持 `running`、下游不提交，未登记工作项仍可领取，会话自己的结果到达后由下一轮结算并推进；
  生产形态的用例保留真实 Workspace 绑定与心跳中的 Node。#377 在真实栈上让真实 bash 工具阻塞回合后重启 Cloud 容器，
  15 秒后仍只有一条执行、`executionId` 与 `dispatchedEpoch` 与重启前相同、执行无结果、运行保持 `running`，
  阻塞解除后由原执行的结果结算为 `succeeded` 并按 S5 释放 Workspace。
- 仍缺：多副本同时推进与**另一个副本接管**仍没有部署级证据，属多副本验收；运行 #377 覆盖的是重启后的同一副本
  接手，与上一条共同构成 D1 落地顺序第 3 步的验收面（ora-space/circus#70："真实跑通 Start → Agent → Output，fixtures=false"）。

## A Node With No Committed Result Is Never Recorded Successful

- 风险：未知结果被当作成功，运行的终态谎报了一次没有发生或没有留下结果的执行；下游节点基于不存在的输出继续跑。
- 前置状态：节点已派发，回调未到、结果丢弃，或进程在写入结果前消失。
- 触发：执行侧超时、结果响应丢失、运行被中断后恢复、回调从未到达。
- 必须成立：没有已提交结果的节点永不被记为成功；恢复接手不改变这一点——在途无结果的节点由这次执行自己的结果结算，Node 失联或超过回合窗口仍无结果时结算为失败，失败随运行终态对外可见；只有提交了结果的成功才允许推进下游节点。
- 禁止结果：把"已派发"或"未知"记成成功、用 HTTP 200 或夹具 trace 代替真实结果、让在途节点无限期持有或在 Node 已失联、超过回合窗口后仍不结算、把失败改写成成功以让运行通过。
- 验证义务与证据：

| 义务 | 所需直接证据 | cloud 侧状态 |
| --- | --- | --- |
| 无已提交结果的节点不进入成功状态 | 真实 PostgreSQL + 执行侧：丢弃回调、超时与中断后的节点状态与运行终态 | Partial |
| 运行终态不早于最后一个节点的结果提交 | 真实执行侧：运行进入终态的时点与节点结果提交顺序 | Partial |

- 决策依据：D1 条件②（在途未提交结果的节点由这次执行自己的结果结算，Node 失联或超过回合窗口时结算为失败、绝不记为成功），D4（崩溃恢复），D7（首期节点产出落在节点状态里），不变量 5。
- desktop 参考实现（`8909f07`）一侧：**Covered** — 恢复把在途节点结算为 `interrupted_by_restart` 并让运行失败，已成功的节点保持成功（[`crates/backend/src/workflow/run/operations/tests.rs`](https://github.com/ora-space/desktop/blob/8909f07/crates/backend/src/workflow/run/operations/tests.rs) 的 `recovery_fails_interrupted_turns_and_resumes_stalled_runs`、[`crates/backend/src/workflow/run/recovery.rs`](https://github.com/ora-space/desktop/blob/8909f07/crates/backend/src/workflow/run/recovery.rs) 的 `an_interrupted_outer_row_still_fails_the_run` 与 `a_crashed_round_inside_a_fail_iteration_fails_the_node_and_the_run`、[`crates/backend/src/workflow/run/resume_gap_tests.rs`](https://github.com/ora-space/desktop/blob/8909f07/crates/backend/src/workflow/run/resume_gap_tests.rs) 的 `boot_sweep_interrupted_node_resumes_as_attempt_two`）。说明：desktop 的串行执行者与进程同生死，进程崩溃就是执行中断，所以参考实现把在途节点结算为中断失败；Cloud 是无状态调度器，重启不打断已派发的会话，在途节点按这次执行的 Node 失联（`node_unavailable`）或回合窗口（`session_turn_deadline_exceeded`）结算为失败，两者都不把未知记为成功。
- 本次直接证据：**Partial** — 第 1 行已部分就位：[cloud#72](https://github.com/ora-space/cloud/pull/72) 的
  `TestWorkflowAdvanceFailsRunOnProjectlessAgentNode`（#67 时代的 `TestWorkflowAdvanceFailsRunOnUnwiredAgentNode` 随真实会话接线改名）
  断言缺 Project 的 agent 节点以 `workflow_run_project_required` 结算为失败、既不进成功状态也不让下游提交状态；
  同 PR 的 `TestWorkflowSessionRunsTwoAgentNodesInOrder` 断言在途无结果的
  节点保持 `running` 而不被记成功、`TestWorkflowSessionStartDeadlineFailsNode` 与 `TestWorkflowSessionFailureFailsRun`
  断言超时与失败的节点连同运行一起进入失败终态。[cloud#91](https://github.com/ora-space/cloud/pull/91)（merge `068d6718`）的
  `TestWorkflowSessionNodeGoneSettlesTheNodeFailed` 用两种 Node 消失形态（心跳超过 `delivery_unreachable_after`、强制停止把沙箱
  与其中未完成的执行一起终止）断言已派发而无结果的节点以 `node_unavailable` 失败、运行随之进入失败终态并声明 Workspace 删除；
  `TestWorkflowSessionNodeSilenceWithinWindowHolds` 与 `TestWorkflowSessionUnconfiguredWindowHoldsTheNode` 断言窗口内或窗口
  未配置时会话保持 `running`，不把沉默记成成功；`TestWorkflowSessionResultAfterTheNodeSettledIsKept` 断言结算后到达的真实结果
  留在执行行，不改写节点与运行的终态。[cloud#108](https://github.com/ora-space/cloud/pull/108)（merge `a4a13e8`，ora-space/circus#299
  选项 A）补上已派发而既无结果也无成功 `turnEnded` 的会话：`TestWorkflowSessionTurnDeadlineFailsNode` 断言执行行注册时刻超过
  `workflow.session_turn_timeout`（默认 2 小时）时，节点与运行以 Cloud 自己的 `session_turn_deadline_exceeded` 进入失败终态并
  声明 Workspace 删除，重复推进不再扫描终态运行；`TestWorkflowSessionTurnDeadlineHoldsWithinWindow` 与
  `TestWorkflowSessionTurnDeadlineUsesTheConfiguredWindow` 断言窗口内、以及部署配置的窗口值下会话保持 `running`，不把沉默记成
  成功；`TestWorkflowSessionTurnDeadlineDoesNotOverrideSuccessfulTurn` 与 `TestWorkflowSessionTurnDeadlineDoesNotOverrideAResult`
  断言成功 `turnEnded` 与已提交结果分别优先于超窗结算（判定顺序为结果 → Node 不可达 → 回合窗口）；
  `TestWorkflowSessionTurnDeadlineResultAfterTheNodeSettledIsKept` 断言超窗结算后到达的真实结果留在执行行、不改写节点与运行的
  终态；`TestWorkflowSessionTurnDeadlineIsPerExecution` 断言窗口按执行行计时，重试的新执行行重新计时。
  重启接手同样不改变这一点：[cloud#113](https://github.com/ora-space/cloud/pull/113)（merge `9814cacd`）的
  `TestWorkflowAdvanceHoldsAProductionSessionAcrossRestart` 保留真实 Workspace 绑定与心跳中的 Node，用第二个 `Store`
  接手后断言在途节点仍为 `running`、运行不进入终态、下游不提交、原执行没有被重派；会话自己的结果到达后才由下一轮
  结算为 `succeeded` 并推进下游。运行 #377 的 `restart-takeover` 在真实栈上于回合阻塞期间重启 Cloud 容器，15 秒后
  仍只有一条无结果的执行、运行保持 `running`，回合解除后由同一执行的身份结算为 `succeeded`。
  第 2 行已部分就位：同 PR 的 `TestWorkflowRunningCarriesNoFinishedAt`
  断言运行仍在执行时不带 `finishedAt`。唯一会写 `node_states` 成功的 `MockWorkflowRunSimulator` 在没有真实结果的
  情况下直接产出成功状态，根决策已把这类模拟 trace 排除在验收之外。
- 仍缺：显式断言"终态写入发生在最后一个节点的结果提交之后"（现有用例只覆盖运行中不带 `finishedAt`）；回调丢失或
  中断后重试仍不成功的用例；Node 不可达与回合窗口仍只在控制面接缝上验证，真实执行侧的失败结算与终态时点验证仍属 D1 落地顺序第 3 步。

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

- 决策依据：D4（取消是请求加有界终态），D6（取消写入者纳入首期，agent 侧一并补上），不变量 3；继承 [issue-run 根决策](../../../decisions/cloud/issue-run/0-agent-run-in-disposable-isolated-workspace.md) D6。
- desktop 参考实现（`8909f07`）一侧：**Covered** — 取消是有界终态而非即时终态：清理失败时仍提交 `Cancelled`（[`crates/backend/src/workflow/run/operations/tests.rs`](https://github.com/ora-space/desktop/blob/8909f07/crates/backend/src/workflow/run/operations/tests.rs) 的 `cancellation_commits_even_when_session_cleanup_fails`）、取消与完成并发时以已提交结果为准（同文件的 `concurrent_cancel_and_complete_report_the_committed_outcome`）、取消后迟到唤醒是空操作（[`crates/backend/src/workflow/run/retry_tests.rs`](https://github.com/ora-space/desktop/blob/8909f07/crates/backend/src/workflow/run/retry_tests.rs) 的 `cancel_during_the_wait_cancels_immediately_and_the_late_wake_is_a_no_op`）、已取消运行不能被清理复活（[`crates/backend/src/workflow/run/loop_exit_tests.rs`](https://github.com/ora-space/desktop/blob/8909f07/crates/backend/src/workflow/run/loop_exit_tests.rs) 的 `loop_exit_cleanup_cannot_revive_a_cancelled_run`）。desktop 侧缺的是入口本身：取消由本地界面/命令触发，Cloud 需要的 HTTP 写入者没有参考实现。
- 本次直接证据：**两行均 Covered** — 第 2 行的 agent 侧已就位：[cloud#100](https://github.com/ora-space/cloud/pull/100)（merge `f405823e`）落地
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
- 仍缺：取消、清理与真实执行侧回调的同窗竞争已有直接证据（第 1 行据此记 `Covered`），但两个副本同时推进**同一个**正在
  取消的 run 仍没有直接证据：运行 #507 的 `controller-replica` 场景只证明两个 Controller 的窗口里删除操作只有一个完成者，
  没有让两个副本竞争推进同一个 run。属 D1 落地顺序第 3 步（ora-space/circus#70）。

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
  没有发生补记。因此补记按「控制面单测 + 真实栈证明无公开入口、项目删除在运行持有期间被拒并记拒绝码」的口径登记：补记本身
  由 `TestWorkflowRunReleaseHealsAWorkspaceDeletedByAnotherPath` 在控制面层面直接证明，端到端补记在真实栈上没有入口。
- 仍缺："另一条路径已删除 Workspace"的端到端补记没有公开入口（项目删除在运行持有期间被拒并记下 `409 runtime_control_held` /
  `409 operation_in_progress`），补记只由控制面单测直接证明，第 4 行因此仍按 `Partial` 记；两个副本同时推进同一个 run 的
  竞争仍缺部署级证据，属 D1 落地顺序第 3 步（ora-space/circus#70）。
