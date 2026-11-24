# 取消一轮与一轮结束后保留会话

依据[取消一轮决策](../../../decisions/node/protocol/20261116-cancel-turn-and-keep-session-after-turn-ends.md)（`implemented`）。
本文的 30 个核心用例与该决策"验证义务"一节的 30 条一一对应，用例标题前的编号就是验证义务的编号。各条义务不带括号的一行按
desktop main [`447a59de`](https://github.com/ora-space/desktop/commit/447a59de6722500cae0be011860e63e5127a3b23) 登记：3 条 `Covered`，25 条 `Partial`，2 条 `Missing`，缺口逐条写在行内。
其中 `turn_processes/` 与 `turn_fence_restart.rs` 下的测试由 `447a59de` 加入，`agent_session.rs` 的 `provider_failure_…`、
`missing_agent_credentials_…` 与 `admission/tests.rs` 的 `timeout_and_actor_commit_…` 早于本决策，其余由
[`2fc5e2ca`](https://github.com/ora-space/desktop/commit/2fc5e2cacdf3c47ca840785e141167ff947aa5aa) 加入。会话命令、事件序号与终态的既有规则继续受
[Agent 运行的协议消息用例](agent-run-messages.md)约束；结束会话时的停止责任与停止证据受
[会话 MCP 刷新与宿主健康用例](../agent-runtime/session-mcp-lifecycle.md)约束，本文不推导它们的证据。
[尝试环境标记决策](../../../decisions/node/agent-runtime/20261120-attempt-mark-in-session-stop-evidence.md)（`implemented`）修改了 D3 复述的停止证据：用例 11、12、13 各增加一行带“（尝试标记）”的
义务，不另占验证义务编号，一一对应关系不变。这三行按 [desktop#802](https://github.com/ora-space/desktop/pull/802) 的 head
`198cc415`（当时尚未合入 main）登记：11、12 为 `Partial`，13 为 `Missing`；13 的这一行后来被下文轮次进程栅栏决策的
“13（轮次进程栅栏）”一行取代，后者现为 `Partial`。

[轮次进程栅栏决策](../../../decisions/node/protocol/20261123-turn-process-fence-before-keeping-session.md)（`implemented`）
修改了验证义务 10、13，并给 D3 情况表新增 detail `turn_processes_unstopped`：用例 10、13、19 各增加一行
带“（轮次进程栅栏）”的义务，同样不另占编号；用例 13 中原有的“（尝试标记）”一行“保留会话的取消不杀任何进程”与新决策
相反，改写为栅栏下的一致性测试。这三行按 [desktop#807](https://github.com/ora-space/desktop/pull/807) 的合并提交
[`447a59de`](https://github.com/ora-space/desktop/commit/447a59de6722500cae0be011860e63e5127a3b23) 登记：10、19 为
`Covered`，13 为 `Partial`。取消决策自身的义务不在这次登记范围内。栅栏自身的义务见
[轮次进程栅栏用例](turn-process-fence.md)。

[一轮的失败与取消分开记录决策](../../../decisions/node/protocol/20261124-turn-failure-reason-distinct-from-cancel.md)（`approved`）
修改了验证义务 6、18、19、20、30：`TurnEnded` 带 `ended_by` 或 `failure`，选择了失败后保留的会话在
失败之后有 `TurnFailureSettled`，`keep_session_on_failure` 另要求协商 `turn_failure`，终态 detail 有三处变化（两处按 D3 表修正，
ACP 错误码 `-32000` 为 `agent_authentication_required`），对所有会话生效。用例 6、18、19、
20、30 各增加一行带“（失败原因）”的义务，不另占编号。新增的义务尚未实现，状态为 `Missing`。失败原因自身的义务见
[失败原因用例](turn-failure-reason.md)。

| 验证义务 | 核心用例 |
| --- | --- |
| 1 | [Cancelling a running turn keeps the session for the next turn](#cancelling-a-running-turn-keeps-the-session-for-the-next-turn) |
| 2 | [Cancelling a queued turn settles immediately regardless of the turn ahead](#cancelling-a-queued-turn-settles-immediately-regardless-of-the-turn-ahead) |
| 3 | [Cancelling an ended or unknown turn reports already ended or unknown](#cancelling-an-ended-or-unknown-turn-reports-already-ended-or-unknown) |
| 4 | [A cancel racing a natural turn end never cancels the next turn](#a-cancel-racing-a-natural-turn-end-never-cancels-the-next-turn) |
| 5 | [Duplicate cancels during a stop share one conclusion](#duplicate-cancels-during-a-stop-share-one-conclusion) |
| 6 | [A cancel during a failed turn's silent window follows the failure judgement](#a-cancel-during-a-failed-turns-silent-window-follows-the-failure-judgement) |
| 7 | [Turn traffic inside the silent window ends the session](#turn-traffic-inside-the-silent-window-ends-the-session) |
| 8 | [A late update with the previous turn's identity is detected as a leak](#a-late-update-with-the-previous-turns-identity-is-detected-as-a-leak) |
| 9 | [Unfinished tools or subagent projection prevent keeping the session](#unfinished-tools-or-subagent-projection-prevent-keeping-the-session) |
| 10 | [An unlisted agent version never keeps the session](#an-unlisted-agent-version-never-keeps-the-session) |
| 11 | [An unanswered cancel settles in time and ends only after stop evidence](#an-unanswered-cancel-settles-in-time-and-ends-only-after-stop-evidence) |
| 12 | [Missing stop evidence never produces an early terminal or delivery](#missing-stop-evidence-never-produces-an-early-terminal-or-delivery) |
| 13 | [Turn boundary conformance gates the allowlist](#turn-boundary-conformance-gates-the-allowlist) |
| 14 | [A full in-flight window delays delivery but not local settlement](#a-full-in-flight-window-delays-delivery-but-not-local-settlement) |
| 15 | [The Controller declares an unsettled cancel only after silence beyond T](#the-controller-declares-an-unsettled-cancel-only-after-silence-beyond-t) |
| 16 | [An old Controller cannot take over an opted-in session](#an-old-controller-cannot-take-over-an-opted-in-session) |
| 17 | [Old Controller handshakes still decode](#old-controller-handshakes-still-decode) |
| 18 | [A failed turn keeps the session when failure keeping is chosen](#a-failed-turn-keeps-the-session-when-failure-keeping-is-chosen) |
| 19 | [Every mandatory end still ends the session and settles pending cancels](#every-mandatory-end-still-ends-the-session-and-settles-pending-cancels) |
| 20 | [Sessions without either choice keep the previous behaviour](#sessions-without-either-choice-keep-the-previous-behaviour) |
| 21 | [EndSession takes precedence over a queued cancel](#endsession-takes-precedence-over-a-queued-cancel) |
| 22 | [Session-level notifications do not break quiescence](#session-level-notifications-do-not-break-quiescence) |
| 23 | [Cancel, admission and MCP wait timeout have one result](#cancel-admission-and-mcp-wait-timeout-have-one-result) |
| 24 | [A cancel after admission but before the prompt write keeps the session](#a-cancel-after-admission-but-before-the-prompt-write-keeps-the-session) |
| 25 | [Cancels merged into a watchdog stop share one deadline](#cancels-merged-into-a-watchdog-stop-share-one-deadline) |
| 26 | [Leak detection spans all later turns but not normally completed ones](#leak-detection-spans-all-later-turns-but-not-normally-completed-ones) |
| 27 | [A restart before the conclusion settles the cancel as session ending](#a-restart-before-the-conclusion-settles-the-cancel-as-session-ending) |
| 28 | [A restart after the conclusion keeps the original end decision](#a-restart-after-the-conclusion-keeps-the-original-end-decision) |
| 29 | [A persistently unwritable ledger never settles from memory](#a-persistently-unwritable-ledger-never-settles-from-memory) |
| 30 | [The cancel and failure choices act independently](#the-cancel-and-failure-choices-act-independently) |

## Cancelling a running turn keeps the session for the next turn

### 风险

取消一轮后会话被结束，用户只能新开对话；或者事件顺序错乱，Controller 先看到下一轮再看到取消结论。

### 前置状态与触发

会话选择了 `keep_session_on_cancel`，Agent 版本在轮次边界清单中，一轮正在执行；Controller 发 `CancelTurn`，Agent 在截止时刻前回应，静止条件成立。

### 必须成立

- 事件顺序为这一轮的 Thread 事件 → `TurnEnded{cancelled}` → `TurnCancelSettled{stopped}` → 下一轮的 Thread 事件。
- 同一会话接着受理 `SubmitUserTurn`，下一轮能看到之前的上下文。

### 禁止结果

会话因取消而结束；`TurnCancelSettled` 早于 `TurnEnded`；下一轮丢失上下文。

### 验证义务与证据

| 义务 | 直接证据与缺口 | 状态 |
|---|---|---|
| 1：取消运行中的一轮后会话保留、上下文保留、事件顺序正确 | [a_cancelled_running_turn_keeps_the_session_for_the_next_turn](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/apps/ora-node/tests/agent_session/cancel.rs#L77)：清单登记的版本上取消运行中的一轮，`TurnEnded` 早于 `TurnCancelSettled{stopped}`，同一会话随后执行下一轮、没有终态。缺口：没有断言下一轮能看到之前的上下文；provider 会话失效后仍只有一次 `session/new` 的断言在用例 18 的恢复测试中，不推导直接续用时的上下文。后续：[circus#1411](https://github.com/ora-space/circus/issues/1411) | Partial |

### 决策依据

D1、D2、D4（事件顺序），不变量 7。

## Cancelling a queued turn settles immediately regardless of the turn ahead

### 风险

取消排队轮次要等前一轮跑完才有结论，超过 Controller 的等待上限；或者被取消的轮次仍被发给 Agent。

### 前置状态与触发

前一轮正在执行且运行时间超过 T，目标轮次已受理、仍在排队；Controller 取消目标轮次。

### 必须成立

- 受理后立即写入 `TurnCancelSettled{not_started}`，目标命令结算为 Discarded。
- 目标轮次从未发给 Agent，没有它的用户消息和 `TurnEnded`；前一轮照常执行。

### 禁止结果

结论等前一轮结束才写入；目标轮次被发给 Agent；前一轮被取消或丢弃。

### 验证义务与证据

| 义务 | 直接证据与缺口 | 状态 |
|---|---|---|
| 2：排队轮次的取消立即结算，不受前一轮运行时间影响 | [a_queued_turn_is_dropped_at_once](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/apps/ora-node/tests/agent_session/cancel.rs#L123)：前一轮一直运行时取消排队轮次，立即得到 `not_started`，目标命令 Discarded、从未进入历史，前一轮不受影响；[a_dropped_turn_is_discarded_with_its_settlement](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/crates/node-db/src/tests/session/turn_cancel.rs#L114)：丢弃与结论同一事务、重复结算不产生第二条；[cancels_during_the_restore_settle_without_waiting_for_it](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/apps/ora-node/tests/agent_session/resume.rs#L241)：恢复 prior revision 期间取消排队轮次同样立即结算。前一轮在测试中没有上限，所以结论不取决于它运行多久，包括超过 T | Covered |

### 决策依据

D1（排队），不变量 1、5。

## Cancelling an ended or unknown turn reports already ended or unknown

### 风险

对已有结果或不存在的轮次，Controller 收不到结论，只能等到自己的超时。

### 前置状态与触发

分别取消：正常完成的轮次；以取消或失败结束且已判定保留会话的轮次；这个会话没有受理过的 `turn_id`。

### 必须成立

前两者立即得到 `already_ended`，最后一个立即得到 `unknown`；会话状态不变。

### 禁止结果

没有结论；改写已有结果；结束会话。

### 验证义务与证据

| 义务 | 直接证据与缺口 | 状态 |
|---|---|---|
| 3：已结束与不存在的轮次立即得到对应结论 | [cancels_of_ended_or_unknown_turns_do_nothing](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/apps/ora-node/tests/agent_session/cancel.rs#L162)：正常完成的一轮得到 `already_ended`、没有受理过的 `turn_id` 得到 `unknown`，会话不变。缺口：没有取消“以取消或失败结束、已判定保留”的一轮的用例。后续：[circus#1411](https://github.com/ora-space/circus/issues/1411) | Partial |

### 决策依据

D1（已有结果、没有这个 `turn_id`），不变量 2。

## A cancel racing a natural turn end never cancels the next turn

### 风险

用户点停止时这一轮恰好自然结束，取消落在用户随后提交的下一轮上。

### 前置状态与触发

一轮自然结束与 `CancelTurn` 的生效竞争，用户随后提交下一轮。

### 必须成立

`TurnEnded` 保留 Agent 自己的停止原因；结论为 `already_ended`；下一轮照常执行，不被取消。

### 禁止结果

下一轮被取消；`TurnEnded` 被改写为取消。

### 验证义务与证据

| 义务 | 直接证据与缺口 | 状态 |
|---|---|---|
| 4：取消与自然结束竞争时只得到 `already_ended`，不误伤下一轮 | [a_prompt_that_ended_on_its_own_reports_no_applied_cancel](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/crates/agent-runtime/src/turn_cancel_tests.rs#L157)：运行时中 prompt 先自己结束时，取消不写出 `session/cancel`、`cancel_applied` 为假；[only_an_already_queued_answer_is_taken](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/crates/agent-runtime/src/turn_cancel.rs#L400)：只有写出取消之前已排队的回应算 Agent 先到；[an_end_turn_answer_to_a_cancel_counts_as_cancelled](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/apps/ora-node/tests/agent_session/cancel.rs#L403) 固定相反的一侧（取消写出后的 `end_turn` 按取消处理）。缺口：没有 Node 层的竞争用例，`already_ended`、`TurnEnded` 保留停止原因与下一轮不受影响没有在同一条路径上断言；回应已排队而取消先被处理的时刻无法经完整 actor 确定性构造。后续：[circus#1411](https://github.com/ora-space/circus/issues/1411) | Partial |

### 决策依据

D1（为什么带 `turn_id`）、D4（先到者为准），不变量 5。

## Duplicate cancels during a stop share one conclusion

### 风险

停止中受理的第二个取消提前得到 `already_ended`，而这次停止最终以会话结束收场，两个结论互相矛盾。

### 前置状态与触发

同一轮先后受理两个 `CancelTurn`：第二个分别落在等待回应的 G 内、静默窗口 Q 内；另覆盖这次停止以兜底结束。

### 必须成立

两者得到相同结论；兜底时两者都得到 `session_ending`；各自的 `TurnCancelSettled` 按受理顺序紧接 `TurnEnded`。

### 禁止结果

任何一个得到 `already_ended`；两者结论不同；第二个重复发 `session/cancel`。

### 验证义务与证据

| 义务 | 直接证据与缺口 | 状态 |
|---|---|---|
| 5：停止中重复取消并入同一次停止 | [duplicate_cancels_share_one_conclusion](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/apps/ora-node/tests/agent_session/cancel.rs#L455)：第二个取消在同一次停止开始时、静默窗口内受理，以及未登记版本兜底，三种情形两者结论相同；[a_cancel_accepted_as_the_quiet_window_ends_shares_the_conclusion](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/apps/ora-node/tests/agent_session/cancel.rs#L722)：Q 内受理、唤醒丢失时仍并入；[a_cancel_accepted_while_the_stop_is_judged_settles_stopped](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/apps/ora-node/tests/agent_session/turn_processes/order.rs#L80)、[a_cancel_accepted_during_an_unclean_fence_shares_its_end](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/apps/ora-node/tests/agent_session/turn_processes/order.rs#L109)、[a_cancel_accepted_during_the_recheck_merges_into_the_stop](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/apps/ora-node/tests/agent_session/turn_processes/order.rs#L310)：判定期间受理的第二个取消按受理顺序与第一个同结论。缺口：第一个取消已写出、仍在等待回应时才受理第二个的时刻没有单独构造；没有断言不重复发 `session/cancel`，也没有断言两条结论紧接 `TurnEnded`。后续：[circus#1411](https://github.com/ora-space/circus/issues/1411) | Partial |

### 决策依据

D1（停止中），不变量 4。

## A cancel during a failed turn's silent window follows the failure judgement

### 风险

一轮失败后的静默窗口内受理的取消，结论与这一轮最终的判定不一致。

### 前置状态与触发

会话选择了 `keep_session_on_failure`，一轮失败，Agent 已回应，正处于静默窗口；此时受理 `CancelTurn`。

### 必须成立

判定会话保留时结论为 `already_ended`；判定走兜底时结论为 `session_ending`。（失败原因）这一轮的 `TurnEnded` 带
`failure`、不带 `ended_by`；`TurnFailureSettled` 与取消结论一致（`kept` 对应 `already_ended`，`session_ending` 对应
`session_ending`），写在取消结论之前；保留时两者在同一事务中写入。

### 禁止结果

在判定之前写结论；得到 `stopped`；（失败原因）因为并入了取消而把这一轮改记为 `caller_cancel`。

### 验证义务与证据

| 义务 | 直接证据与缺口 | 状态 |
|---|---|---|
| 6：失败后静默窗口内的取消随判定结算 | [a_cancel_during_a_failed_turns_quiet_window_is_already_ended](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/apps/ora-node/tests/agent_session/cancel.rs#L504)：失败的一轮在静默窗口内受理取消，判定保留时得到 `already_ended`，会话接着执行下一轮。缺口：判定走兜底时得到 `session_ending` 没有用例。后续：[circus#1411](https://github.com/ora-space/circus/issues/1411) | Partial |
| 6（失败原因）：并入取消的失败轮次保持 `failure`，`TurnFailureSettled` 与取消结论一致且在它之前 | 尚未实现，无直接证据 | Missing |

### 决策依据

D1（停止中）、D2，不变量 4；[失败原因决策](../../../decisions/node/protocol/20261124-turn-failure-reason-distinct-from-cancel.md) D1、D6。

## Turn traffic inside the silent window ends the session

### 风险

Agent 回应后仍在发上一轮的工具更新，下一轮仍被发出，两轮的工作混在一起。

### 前置状态与触发

取消后 Agent 回应，静默窗口 Q 内又收到上一轮的工具更新。

### 必须成立

这些更新不记入任何一轮；取消得到 `session_ending`；会话以 `turn_not_quiescent` 结束；下一轮从未发给 Agent。

### 禁止结果

更新记入上一轮或下一轮；会话保留；下一轮被发出。

### 验证义务与证据

| 义务 | 直接证据与缺口 | 状态 |
|---|---|---|
| 7：静默窗口内的轮次流量触发兜底 | [only_a_quiet_settled_turn_keeps_the_session](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/apps/ora-node/tests/agent_session/cancel.rs#L246) 的 `[hold-chatty]`：回应后的正文消息使取消得到 `session_ending`，会话以 `turn_not_quiescent` 结束；[turn_traffic_caused_by_the_fence_ends_the_session](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/apps/ora-node/tests/agent_session/turn_processes/order.rs#L58)：栅栏杀进程后 Agent 自发的轮次流量同样如此。缺口：触发的是正文而不是工具更新；没有断言这些流量不记入任何一轮，也没有提交下一轮并断言它从未发给 Agent。后续：[circus#1411](https://github.com/ora-space/circus/issues/1411) | Partial |

### 决策依据

D2（静止条件 3）、D3，不变量 8。

## A late update with the previous turn's identity is detected as a leak

### 风险

静默窗口之后才到达的上一轮更新混进下一轮，被当作下一轮的内容。

### 前置状态与触发

静默窗口已过，下一轮已开始，之后收到引用上一轮工具调用身份的更新。

### 必须成立

该更新不进入任何一轮的 Thread 事件与实时预览（会话 JSONL 例外，见决策的差异表）；会话以 `turn_leak_detected` 结束。

### 禁止结果

更新进入下一轮的 Thread 事件或实时预览；会话继续。

### 验证义务与证据

| 义务 | 直接证据与缺口 | 状态 |
|---|---|---|
| 8：跨进下一轮的有身份更新被检出 | [a_leaked_update_in_a_later_turn_ends_the_session](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/apps/ora-node/tests/agent_session/cancel.rs#L290)：引用上一轮工具身份的更新不进入 Thread，会话以 `turn_leak_detected` 结束；[a_pending_leaked_update_never_reaches_the_preview](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/apps/ora-node/tests/agent_session/cancel.rs#L548)：紧接的下一轮中仍挂起的同类更新不进入实时预览；[a_fenced_tool_update_in_a_later_turn_is_a_leak](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/apps/ora-node/src/session/fence/tests.rs#L56)：栅栏的判定规则。缺口：该更新已先写入会话 JSONL，“不记入下一轮”只对 Thread 与实时预览成立（见决策的差异表）；子代理来源的迟到更新没有用例。后续：JSONL 记录顺序 [circus#1414](https://github.com/ora-space/circus/issues/1414)、子代理来源的测试 [circus#1411](https://github.com/ora-space/circus/issues/1411) | Partial |

### 决策依据

D2（跨轮次检测）、D3，不变量 8。

## Unfinished tools or subagent projection prevent keeping the session

### 风险

回应时仍有工具在运行，或子代理在父轮次结束后继续运行，保留会话后它们继续修改 checkout。

### 前置状态与触发

取消或失败后 Agent 回应时，分别存在：未终态的工具调用；这一轮启动过子代理投影。

### 必须成立

两种情况下会话都以 `turn_not_quiescent` 结束。

### 禁止结果

会话保留；把子代理的终态从父 prompt 结束推断出来。

### 验证义务与证据

| 义务 | 直接证据与缺口 | 状态 |
|---|---|---|
| 9：未终态工具或子代理投影使静止条件不成立 | [only_a_quiet_settled_turn_keeps_the_session](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/apps/ora-node/tests/agent_session/cancel.rs#L246) 的 `[hold-running-tool]`：取消回应时有未终态工具，会话以 `turn_not_quiescent` 结束；[a_running_tool_leaves_the_turn_unsettled](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/apps/ora-node/src/session/fence/tests.rs#L19)、[a_child_projection_leaves_the_turn_unsettled](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/apps/ora-node/src/session/fence/tests.rs#L38)：两种情况都使这一轮未静止。缺口：子代理投影只有判定规则的单测，没有会话以 `turn_not_quiescent` 结束的用例；失败结束的一轮没有用例。后续：[circus#1411](https://github.com/ora-space/circus/issues/1411) | Partial |

### 决策依据

D2（静止条件 2），不变量 7。

## An unlisted agent version never keeps the session

### 风险

没有验证过轮次边界的 Agent 版本也保留会话，无身份的迟到更新无法排除。

### 前置状态与触发

Agent 插件版本不在轮次边界清单中，一轮取消或失败结束。另一种情形（轮次进程栅栏）：（插件，版本）只为其他平台登记，
例如在 Linux Node 上只有 `windows` 条目。ora-node 目前只有 Linux 版，所以没有 Windows Node 上的场景可测。

### 必须成立

会话以 `turn_boundary_unverified` 结束，即使会话选择了对应的保留字段。只为其他平台登记时同样如此，且不运行栅栏。
同一插件的其他版本不因某个版本已登记而被认可。

### 禁止结果

会话保留；按其他平台的条目保留会话；栅栏在未登记的组合上运行。

### 验证义务与证据

| 义务 | 直接证据与缺口 | 状态 |
|---|---|---|
| 10：不在清单中的版本走兜底 | [an_unverified_agent_ends_the_session_after_a_cancel](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/apps/ora-node/tests/agent_session/cancel.rs#L198)：未登记版本在取消后以 `turn_boundary_unverified` 结束、取消得到 `session_ending`；[another_version_of_a_listed_plugin_is_not_verified](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/apps/ora-node/tests/agent_session/cancel.rs#L677)：只登记了另一版本时同样。缺口：选择了 `keep_session_on_failure` 的未登记版本在一轮失败后的用例缺失。后续：[circus#1411](https://github.com/ora-space/circus/issues/1411) | Partial |
| 10（轮次进程栅栏）：清单按（插件，版本，平台）判定，只为其他平台登记的组合以 `turn_boundary_unverified` 结束 | [`a_boundary_is_verified_for_its_exact_version_and_platform`](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/apps/ora-node/src/session/turn_cancel.rs#L111)：只认精确的插件、版本与平台，默认清单认可 Linux 上的 OpenCode 0.6.11，不认可同插件的 0.6.12、Claude 0.4.5 与只为 Windows 登记的版本；[`a_version_verified_only_on_another_platform_is_unverified`](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/apps/ora-node/tests/agent_session/turn_processes.rs#L298)：只为 Windows 登记时，取消以 `turn_boundary_unverified` 结束会话；[`another_version_of_a_listed_plugin_is_not_verified`](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/apps/ora-node/tests/agent_session/cancel.rs#L677)：清单只有同插件的另一版本时同样结束；[`a_stop_that_cannot_keep_the_session_runs_no_fence`](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/apps/ora-node/tests/agent_session/turn_processes/limits.rs#L56)：未登记的版本被取消时，结束决定写入的那一刻本轮作业仍存活，栅栏没有运行 | Covered |

### 决策依据

D2（可依赖的轮次边界）、D3，不变量 7；[轮次进程栅栏决策](../../../decisions/node/protocol/20261123-turn-process-fence-before-keeping-session.md) D5，不变量 9。

## An unanswered cancel settles in time and ends only after stop evidence

### 风险

Agent 不回应取消时，Controller 等不到结论；或者 Node 在取得停止证据之前就写终态。

### 前置状态与触发

正在执行的一轮被取消，截止时刻前 Agent 没有回应。

### 必须成立

- 受理后 G + Q 内已写入 `TurnEnded{cancelled}` 与 `TurnCancelSettled{session_ending}`。
- 之后不再写出任何 `session/prompt`。
- 取得停止证据之后才写 `AgentSessionEnded{agent_failed}`，detail `turn_cancel_unanswered`。
- 这一轮工具起的、脱离进程组但保留环境的后台作业（`cmd &`、`setsid`）在终态之前与会话的其他带标记进程一起被杀；
  停止证明记 `lock_processes_and_marks`。

### 禁止结果

结论晚于 G + Q；结论之后仍有 `session/prompt`；终态早于停止证据；后台作业仍存活时写终态。

### 验证义务与证据

| 义务 | 直接证据与缺口 | 状态 |
|---|---|---|
| 11：没有回应时按期结算，终态等停止证据 | [missing_stop_proof_keeps_cancel_settled_without_a_terminal](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/apps/ora-node/tests/standalone/agent_sessions/turn_cancel/stop.rs#L6)：真实 Node 上 Agent 不回应，`session_ending` 在 17 秒（G + Q）内写入，取得原证据后才写 `turn_cancel_unanswered` 终态；[the_cancel_deadline_counts_from_acceptance](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/apps/ora-node/tests/agent_session/cancel.rs#L587)：期限从受理起算；[an_unanswered_cancel_ends_the_session](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/apps/ora-node/tests/agent_session/cancel.rs#L221)：结论与 detail。缺口：没有断言 `TurnEnded{cancelled}` 先于结论写入，没有断言结论之后不再写出 `session/prompt`。后续：[circus#1411](https://github.com/ora-space/circus/issues/1411) | Partial |
| 11（尝试标记）：以 `session_ending` 结束时本轮的脱组后台作业在终态前被杀 | [ending_a_running_turn_kills_its_detached_tool_job](https://github.com/ora-space/desktop/blob/198cc41517a837ec436a897655403adc790b8c0d/apps/ora-node/tests/standalone/agent_sessions/background_jobs.rs#L211)：轮次运行中以 EndSession 结束会话，本轮工具起的 `setsid` 作业在 `AgentSessionEnded` 之前退出、证明为 `lock_processes_and_marks`、之后不写入。缺口：仓库里没有经 `CancelTurn` 以 `session_ending` 结束会话的用例。评审在 `198cc415` 上核对取消以 `turn_boundary_unverified` 结束后进入同一停止确认，并用临时用例与真实插件复测确认作业先被杀，这些是运行记录，补进仓库后再登记 | Partial |

### 决策依据

D1（正在执行）、D3、D4（本地结算期限），不变量 2、9；[尝试环境标记决策](../../../decisions/node/agent-runtime/20261120-attempt-mark-in-session-stop-evidence.md) D4。

## Missing stop evidence never produces an early terminal or delivery

### 风险

停止证据始终取不到时，Node 提前写终态或允许交付、恢复，残留进程继续修改 checkout。

### 前置状态与触发

G 内没有回应；停止请求返回（分别覆盖成功与失败）之后，原尝试锁仍被持有、直接进程仍在，或仍有被尝试标记扫描
认领的存活进程（例如杀不掉的脱组后台作业）。

### 必须成立

- `TurnCancelSettled{session_ending}` 按期写入。
- 停止确认期限内和期限到达后都没有 `AgentSessionEnded`；期限到达后 `GetExecutionStatus` 返回 `StopUnconfirmed`。
- 这个执行自己的交付被拒绝（`session_not_settled`），带 `prior_revision` 的恢复被拒绝。
- 之后对账取得原证据，才补发终态，detail 仍是 `turn_cancel_unanswered`；终态之后没有事件。

### 禁止结果

把停止请求的返回当作停止证据；期限到达就写终态；交付或恢复被放行；补发的终态改了原因；锁与线索干净而带标记
进程仍存活时写终态。

### 验证义务与证据

| 义务 | 直接证据与缺口 | 状态 |
|---|---|---|
| 12：证据缺席时不提前发终态、不提前交付，迟到证据才补报 | [missing_stop_proof_keeps_cancel_settled_without_a_terminal](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/apps/ora-node/tests/standalone/agent_sessions/turn_cancel/stop.rs#L6)：原锁标记损坏使证据缺席时，结论按期写入、期限前后都没有终态，`GetExecutionStatus` 报 `stop_unconfirmed`，本执行交付以 `session_not_settled` 失败，带 `prior_revision` 的启动被拒绝（`checkout_busy`）；恢复原证据后补发 `turn_cancel_unanswered` 终态，结论不重复，之后没有事件。缺口：没有锁仍被持有、直接进程仍在的情形；停止请求返回失败的情形没有覆盖。后续：[circus#1411](https://github.com/ora-space/circus/issues/1411) | Partial |
| 12（尝试标记）：只剩带标记进程存活时同样缺证据 | [a_surviving_job_or_damaged_scope_is_never_proof](https://github.com/ora-space/desktop/blob/198cc41517a837ec436a897655403adc790b8c0d/apps/ora-node/src/session/stop/tests/marks.rs#L109)：插件进程已退出、锁与线索干净，只剩脱组作业时证据为 Running；[end_session_kills_a_detached_tool_job_and_spares_unmarked_processes](https://github.com/ora-space/desktop/blob/198cc41517a837ec436a897655403adc790b8c0d/apps/ora-node/tests/standalone/agent_sessions/background_jobs.rs#L150) 等真实 Node 用例断言作业存活期间账本从未 `confirmed`。缺口：没有作业杀不掉时终态延后、期限后 `StopUnconfirmed`、交付与恢复被拒绝的用例 | Partial |

### 决策依据

D3（沿用会话 MCP 刷新 D5）、D4（Controller 等待终态），不变量 9；[尝试环境标记决策](../../../decisions/node/agent-runtime/20261120-attempt-mark-in-session-stop-evidence.md) D3。

## Turn boundary conformance gates the allowlist

### 风险

插件版本未经测试就进入轮次边界清单，回应后仍有轮次流量的 Agent 被允许保留会话。

### 前置状态与触发

对清单按（插件，版本，平台）登记的每个组合，在该平台上以生产构建的 `ora-node`（轮次进程栅栏启用）、真实 Deno 与该版本
未修改的发布包运行，由回环模型驱动真实工具。每次运行：一个普通轮次 → 目标轮次（运行中取消；`error` 场景让它自己失败）
→ 一个普通轮次。场景为 `shell`、`subagent`、`background`、`detached`、`error`，每个场景重复运行。

### 必须成立

- 一次运行通过，当且仅当：停止以 `stopped` 结算、会话保留；结算时这一轮在 Agent 更早进程的会话之外新起的进程都已不在；
  结算之后作业的延迟写入没有出现；下一轮正常完成。`error` 场景没有取消结论，进程在下一轮的第一个事件到达时采样，此时
  Node 已判定完失败的一轮、运行过栅栏与复扫；检查的是会话没有结束、那一刻这一轮的进程都已不在、之后没有延迟写入、
  下一轮正常完成。全部场景的全部运行都通过的组合
  才进入清单，条目旁写明发布包摘要。
- 测试验证的是 Agent 回应与 Node 栅栏的组合：后台作业由栅栏杀掉，测试在栅栏之后观察进程与写入，不能假定 Agent 自己停下了
  它们，也不能把另一个平台的结果当作本平台的结论。
- Linux 上的 `official/ora-space.opencode 0.6.11` 通过后才登记；`official/ora-space.claude 0.4.5` 因后台任务被杀后自发起一轮
  （`background` 以 `turn_not_quiescent` 结束）而不登记。

### 禁止结果

有任一运行未通过的组合进入清单；不经真实 Node 栅栏的测试结果被用来登记；按一个平台的结果登记另一个平台。

### 验证义务与证据

| 义务 | 直接证据与缺口 | 状态 |
|---|---|---|
| 13：轮次边界一致性测试决定清单 | 没有 CI 中可重复的一致性证据。本条已由轮次进程栅栏决策 D5 改写，现行义务、仓库内按需运行的一致性测试与登记结果见下一行，由该决策的落地登记，本行不重复登记。后续：[circus#1408](https://github.com/ora-space/circus/issues/1408) | Missing |
| 13（轮次进程栅栏）：一致性测试按（插件，版本，平台）经真实 Node 的栅栏运行五个场景，全部通过才登记。原“（尝试标记）”一行“保留会话的取消不杀任何进程”已被该决策取代 | [`turn_boundary_conformance`](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/apps/ora-node/tests/standalone/agent_sessions/turn_boundary.rs#L286)：以独立进程运行的 Node 驱动安装好的发布包，执行 `shell`、`subagent`、`background`、`detached`、`error` 五个场景与前后的普通轮次，按决策 D5 的通过条件判定；它标为 `#[ignore]`，需要外部发布包与 Deno，只按需运行。[circus#1340](https://github.com/ora-space/circus/issues/1340) 在 x86_64 Linux 上用它测得 OpenCode 0.6.11 五个场景各 5/5、Claude 0.4.5 `background` 0/3，这是运行记录，不作为证据登记。清单内容由 [`a_boundary_is_verified_for_its_exact_version_and_platform`](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/apps/ora-node/src/session/turn_cancel.rs#L111) 断言：只登记 Linux 上的 OpenCode 0.6.11，Claude 0.4.5 不登记（由 [circus#1381](https://github.com/ora-space/circus/issues/1381) 承接）。缺口：一致性测试不在 CI 中运行，没有可重复的证据；aarch64 未测；场景不专门构造懒启动的常驻进程。后续：[circus#1408](https://github.com/ora-space/circus/issues/1408) | Partial |

### 决策依据

D2（可依赖的轮次边界），不变量 7；[轮次进程栅栏决策](../../../decisions/node/protocol/20261123-turn-process-fence-before-keeping-session.md) D5；[尝试环境标记决策](../../../decisions/node/agent-runtime/20261120-attempt-mark-in-session-stop-evidence.md) D4（经栅栏决策修改）。

## A full in-flight window delays delivery but not local settlement

### 风险

在途窗口满时，Node 已按时结算，Controller 却把投递延迟误判为停止失败。

### 前置状态与触发

执行已有 256 个已发送未确认的事件时受理 `CancelTurn`；Controller 之后释放确认。

### 必须成立

- 结论在本地期限内写入账本。
- Controller 释放确认后按序收到结论。
- Controller 持续收到并确认事件期间，不判为超时。

### 禁止结果

结论绕过窗口越序发送；Controller 在持续收到事件时判为超时。

### 验证义务与证据

| 义务 | 直接证据与缺口 | 状态 |
|---|---|---|
| 14：窗口背压只延迟投递 | [a_full_window_does_not_block_cancel_settlement](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/apps/ora-node/tests/standalone/agent_sessions/turn_cancel/window.rs#L6)：256 个未确认事件占满窗口时受理取消，结论在 17 秒内写入账本；窗口满期间只有命令受理和查询回复，释放确认后按序收到同一条结论。缺口：Controller 在持续收到并确认事件期间不判超时，属 Controller 一侧，desktop 中没有证据。后续：[circus#1413](https://github.com/ora-space/circus/issues/1413) | Partial |

### 决策依据

D4（本地结算期限、投递、Controller 等待取消结论），不变量 2、3。

## The Controller declares an unsettled cancel only after silence beyond T

### 风险

Node 确实没有结算时，Controller 一直等下去；或者按固定时长过早下结论。

### 前置状态与触发

连接仍在，Controller 已确认它收到的全部事件，自受理与最后一次确认中较晚的时刻起超过 T 没有新事件。

### 必须成立

Controller 判定 Node 未在期限内结算，把这次停止视为未确认，按自己的故障规则处理。

### 禁止结果

无限等待；把这次停止当作已停下。

### 验证义务与证据

| 义务 | 直接证据与缺口 | 状态 |
|---|---|---|
| 15：超过 T 的静默才判为未结算 | Controller 一侧的规则，`ora-node` 不实现，desktop 中没有证据。由接入 `CancelTurn` 的各 Controller 自己的用例承担。后续：[circus#1413](https://github.com/ora-space/circus/issues/1413) | Missing |

### 决策依据

D4（Controller 等待取消结论）。

## An old Controller cannot take over an opted-in session

### 风险

不理解新事件的旧 Controller 接管已选择新行为的会话，收到无法解码的事件，或为它发出永远等不到结果的命令。

### 前置状态与触发

新 Controller 启动选择了新行为的会话，在确认 `TurnCancelSettled` 前断开；旧 Controller 重连，之后具备能力的 Controller 连接。

### 必须成立

- 旧 Controller 在 `HelloAccepted` 之前被以 `4403` 拒绝，没有收到任何事件，命令不被受理。
- 具备能力的 Controller 从最小未确认序号起按序收到全部事件。

### 禁止结果

旧 Controller 收到或确认任何事件；事件丢失或越序。

### 验证义务与证据

| 义务 | 直接证据与缺口 | 状态 |
|---|---|---|
| 16：接管边界与降级重连 | [a_keep_session_is_taken_over_only_by_a_controller_that_negotiated_turn_cancel](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/apps/ora-node/tests/standalone/agent_sessions/turn_cancel.rs#L87)：没有协商的重连在 `HelloAccepted` 之前被拒绝、没有收到任何帧，具备能力的连接从最小未确认序号开始重放；[keep_sessions_hold_the_takeover_boundary_until_drained](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/crates/node-db/src/tests/session/turn_cancel.rs#L295)：终态且全部确认后才解除；[a_keep_session_cannot_be_started_without_turn_cancel](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/apps/ora-node/tests/standalone/agent_sessions/turn_cancel.rs#L176) 与 [turn_cancel_messages_and_both_start_entries_need_negotiation](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/apps/ora-node/src/service/tests.rs#L127)：未协商的连接不能启动保留会话。缺口：IPC 上以日志原因识别拒绝，没有断言 WebSocket 的 `4403`；断开前没有未确认的 `TurnCancelSettled`，只核对了第一条重放事件的序号。后续：[circus#1411](https://github.com/ora-space/circus/issues/1411) | Partial |

### 决策依据

D5（接管边界、拒绝的持续时间），不变量 10。

## Old Controller handshakes still decode

### 风险

Node 升级后，对能力使用封闭枚举的旧 Controller 握手失败，即使没有任何会话选择新行为。

### 前置状态与触发

用本决策之前版本的 desktop `ora-node-protocol`，以及 cloud v2-server 的握手解码，分别解码新 Node 回给旧 `Hello`（不含 `turn_cancel`）的 `HelloAccepted`；Node 上没有选择新行为的会话。

### 必须成立

- 两者都接受这次握手，旧 Controller 正常连接、收发命令与事件。
- 只有声明了 `turn_cancel` 的 `Hello` 才收到这项能力和 `turn_cancel: true`。

### 禁止结果

旧 Controller 因未知能力值握手失败；未声明的 Controller 收到 `turn_cancel`。

### 验证义务与证据

| 义务 | 直接证据与缺口 | 状态 |
|---|---|---|
| 17：旧 Controller 的实际握手解码 | [a_keep_session_is_taken_over_only_by_a_controller_that_negotiated_turn_cancel](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/apps/ora-node/tests/standalone/agent_sessions/turn_cancel.rs#L87) 的握手部分：旧 `Hello` 得到不含 `turn_cancel` 的 `HelloAccepted`，声明后才有能力与 `turn_cancel: true`；[turn_cancel_negotiation_requires_its_capability](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/crates/node-protocol/tests/protocol/turn_cancel.rs#L104)：确认字段与能力必须同时出现。缺口：没有用本决策之前的 `ora-node-protocol` 与 cloud v2-server 的解码器实际解码新 Node 的握手。后续：[circus#1412](https://github.com/ora-space/circus/issues/1412) | Partial |

### 决策依据

D5（能力协商），不变量 10。

## A failed turn keeps the session when failure keeping is chosen

### 风险

选择了失败后保留的会话仍因一轮失败而结束；或者 provider 会话已失效时下一轮发到失效的会话上。

### 前置状态与触发

会话选择了 `keep_session_on_failure`；一轮因 ACP prompt 报错或看门狗放弃而结束，满足 D2；另覆盖报错后 provider 会话已失效，以及 MCP 生命周期模式的会话。

### 必须成立

- 会话保留。
- provider 会话已失效时，下一轮先恢复再继续。
- MCP 生命周期模式的会话只 load 原 provider ID，失败时 blocked，不新建。
- （失败原因）会话协商了 `turn_cancel` 与 `turn_failure`；保留时调用方先收到带 `failure` 的 `TurnEnded`，再收到
  `TurnFailureSettled{kept}`，之后才是下一轮。

### 禁止结果

会话结束；下一轮发到已失效的 provider 会话；生命周期模式的会话新建 provider 会话；（失败原因）会话保留了却没有
`TurnFailureSettled{kept}`，或调用方看不到失败原因。

### 验证义务与证据

| 义务 | 直接证据与缺口 | 状态 |
|---|---|---|
| 18：失败后保留与 provider 会话恢复 | [a_failed_turn_keeps_only_a_session_that_chose_it](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/apps/ora-node/tests/agent_session/cancel.rs#L362)：ACP prompt 报错后选择了 `keep_session_on_failure` 的会话执行下一轮；[a_failure_the_session_survives_runs_the_fence](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/apps/ora-node/tests/agent_session/turn_processes/limits.rs#L101)：同一路径经过轮次进程栅栏；[an_invalidated_provider_session_is_loaded_before_the_next_prompt](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/apps/ora-node/tests/agent_session/cancel/recovery.rs#L17)：provider 会话失效后先 `session/load`（普通与生命周期模式各一）；[a_lifecycle_restore_failure_blocks_without_creating_a_provider](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/apps/ora-node/tests/agent_session/cancel/recovery.rs#L56)：生命周期模式 load 失败时 blocked、不新建；[failure_restores_cross_the_process_budget_without_ending_the_session](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/apps/ora-node/tests/agent_session/cancel/recovery.rs#L105)、[a_cancelled_turn_can_resume_across_agent_replacement](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/apps/ora-node/tests/agent_session/cancel/recovery.rs#L183)：达到 load 阈值替换 Agent 不结束会话。缺口：看门狗放弃后保留没有用例；非生命周期会话恢复与重建都失败时结束会话没有用例。后续：[circus#1411](https://github.com/ora-space/circus/issues/1411) | Partial |
| 18（失败原因）：失败后保留的会话收到失败原因与 `TurnFailureSettled{kept}`；保留要求协商 `turn_failure`；保留和不能保留时的义务见[失败原因用例](turn-failure-reason.md) 11、12、13 | 尚未实现，无直接证据 | Missing |

### 决策依据

D2（续用方式）、D5，不变量 7；[失败原因决策](../../../decisions/node/protocol/20261124-turn-failure-reason-distinct-from-cancel.md) D6。

## Every mandatory end still ends the session and settles pending cancels

### 风险

新行为让某些必须结束的情况被误当作可保留；或者结束时有取消没有结论。

### 前置状态与触发

逐一触发 D3 表中的每一种情况，其中一部分发生时有尚未结算的 `CancelTurn`。D3 表包括轮次进程栅栏决策新增的一行：
栅栏在 F 内不干净、这一轮没有开始时刻、范围记录不可用或尝试在这一轮里被替换。

### 必须成立

每种情况都结束会话，终态与 D3 表一致；结束决定作出时尚未结算的 `CancelTurn` 都得到 `session_ending`。栅栏不干净时
detail 为 `turn_processes_unstopped`。（失败原因）看门狗取消没有得到回应时 detail 为 `turn_cancel_unanswered`；一轮进行中
Agent 进程退出或 ACP 连接断开时为 `agent_unavailable`；选择了失败后保留的会话，失败的一轮在结束决定同一事务中得到
`TurnFailureSettled{session_ending}`。

### 禁止结果

任何一种情况保留会话；有已受理的 `CancelTurn` 没有结论。

### 验证义务与证据

| 义务 | 直接证据与缺口 | 状态 |
|---|---|---|
| 19：D3 的每种情况结束会话并结算取消 | 账本：[the_end_decision_settles_pending_cancels](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/crates/node-db/src/tests/session/turn_cancel.rs#L220)、[only_the_full_decision_settles_cancels](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/crates/node-db/src/tests/session/turn_cancel.rs#L527)、[a_terminal_settles_pending_cancels_before_it](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/crates/node-db/src/tests/session/turn_cancel.rs#L273)：结束决定同事务结算全部待结算取消。逐行：`turn_cancel_unanswered`、`turn_boundary_unverified`、`turn_not_quiescent`、`turn_leak_detected` 见用例 11、10、7/9、8；`ledger_unavailable` 见 [a_failed_end_decision_ends_with_ledger_unavailable](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/apps/ora-node/tests/agent_session/cancel.rs#L698) 与 [a_ledger_error_during_preparation_waits_for_the_step](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/apps/ora-node/tests/agent_session.rs#L1076)；`model_access_revoked` 见 [a_cancel_during_a_model_ending_settles_within_its_deadline](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/apps/ora-node/tests/agent_session.rs#L995)；`EndSession` 见 [an_end_during_the_restore_settles_cancels_before_the_restore_winds_down](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/apps/ora-node/tests/agent_session/resume.rs#L314)；Node 重启见用例 27。缺口：Agent 退出或连接断开、运行时不再受理、Thread 镜像不可用三行，没有选择了保留且有未结算取消的用例。后续：[circus#1411](https://github.com/ora-space/circus/issues/1411) | Partial |
| 19（轮次进程栅栏）：`turn_processes_unstopped` 一行同样结束会话并结算取消；各种注入见[栅栏不干净用例](turn-process-fence.md#an-unclean-fence-ends-the-session-as-turn-processes-unstopped) | [`jobs_that_keep_coming_back_end_the_session`](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/apps/ora-node/tests/agent_session/turn_processes.rs#L237)、[`an_unknown_fence_ends_the_session`](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/apps/ora-node/tests/agent_session/turn_processes.rs#L264) 与 [`a_recheck_past_its_budget_ends_the_session`](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/apps/ora-node/tests/agent_session/turn_processes/order.rs#L374)：栅栏或复扫不干净时会话以 `agent_failed`、detail `turn_processes_unstopped` 结束，尚未结算的取消得到 `session_ending`；[`a_cancel_accepted_during_an_unclean_fence_shares_its_end`](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/apps/ora-node/tests/agent_session/turn_processes/order.rs#L109)：两个并入的取消都得到 `session_ending`。只在栅栏层断言的注入及其缺口见栅栏不干净用例 | Covered |
| 19（失败原因）：看门狗取消无回应与 Agent 进程退出的 detail 与 D3 表一致；失败的一轮随结束决定得到 `TurnFailureSettled{session_ending}`；义务见[失败原因用例](turn-failure-reason.md) 6、7、12 | 尚未实现，无直接证据 | Missing |

### 决策依据

D3，不变量 2；[轮次进程栅栏决策](../../../decisions/node/protocol/20261123-turn-process-fence-before-keeping-session.md) D3；[失败原因决策](../../../decisions/node/protocol/20261124-turn-failure-reason-distinct-from-cancel.md) D6、D7。

## Sessions without either choice keep the previous behaviour

### 风险

没有选择新行为的调用方行为被改变，依赖"失败即终态"的 Controller 看到会话停在空闲。

### 前置状态与触发

两个字段都没有选择的会话；另覆盖旧 Node 与旧 Controller 的消息、握手与输入编解码。

### 必须成立

- `CancelTurn` 被拒绝（`turn_cancel_unsupported`）。
- 一轮失败仍以 `agent_failed` 和既有 detail 结束。
- 旧 Node 与旧 Controller 的消息、握手与输入编解码不变。
- （失败原因）没有 `TurnFailureSettled`；`TurnEnded` 带 `failure`；detail 只有失败原因决策 D7 的三处变化：看门狗取消无回应为
  `turn_cancel_unanswered`，取消没有写出时 Agent 进程退出为 `agent_unavailable`，ACP 错误码 `-32000` 为
  `agent_authentication_required`。

### 禁止结果

会话因失败而保留；产生 `TurnCancelSettled` 或新的 detail（失败原因决策 D7 的三处变化除外，其中 `turn_cancel_unanswered` 对
这些会话是新值）；旧形状消息无法解码；（失败原因）产生 `TurnFailureSettled`。

### 验证义务与证据

| 义务 | 直接证据与缺口 | 状态 |
|---|---|---|
| 20：未选择的会话与旧对端保持原行为 | [a_session_without_the_cancel_choice_refuses_cancel_turn](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/crates/node-db/src/tests/session/turn_cancel.rs#L89)：没有选择取消保留的会话拒绝 `CancelTurn`，不落账；[provider_failure_ends_the_session_with_a_safe_code_and_releases_resources](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/apps/ora-node/tests/agent_session.rs#L66)、[missing_agent_credentials_end_the_session_with_an_actionable_code](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/apps/ora-node/tests/agent_session.rs#L91)：未选择时一轮失败仍以 `agent_failed` 和既有 detail 结束（先于本决策的测试，保持通过）；[unchosen_keep_fields_are_omitted](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/crates/node-protocol/tests/protocol/turn_cancel.rs#L149)、[cancel_and_settlement_round_trip](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/crates/node-protocol/tests/protocol/turn_cancel.rs#L20)：未选择时输入编码不变，新消息往返。缺口：拒绝测试的会话选择了 `keep_session_on_failure`，两个字段都未选择的拒绝没有单独用例；没有旧 Node、旧 Controller 的实际解码。后续：[circus#1412](https://github.com/ora-space/circus/issues/1412) | Partial |
| 20（失败原因）：未选择的会话没有 `TurnFailureSettled`，`TurnEnded` 带 `failure`，detail 除 D7 的三处变化外不变；义务见[失败原因用例](turn-failure-reason.md) 14 | 尚未实现，无直接证据 | Missing |

### 决策依据

D5，不变量 11；[失败原因决策](../../../decisions/node/protocol/20261124-turn-failure-reason-distinct-from-cancel.md) D6、D7。

## EndSession takes precedence over a queued cancel

### 风险

`EndSession` 与 `CancelTurn` 同时排队时，会话被保留，或者取消没有结论。

### 前置状态与触发

`EndSession` 与 `CancelTurn` 同时在队列中。

### 必须成立

会话按 `EndSession` 结束；`CancelTurn` 得到 `session_ending`，早于终态。

### 禁止结果

会话保留；`CancelTurn` 没有结论或结论晚于终态。

### 验证义务与证据

| 义务 | 直接证据与缺口 | 状态 |
|---|---|---|
| 21：`EndSession` 优先 | [end_session_wins_over_a_queued_cancel](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/apps/ora-node/tests/agent_session/cancel.rs#L334)：两者同时在队列中时按 `EndSession` 结束，取消得到 `session_ending`；[the_end_decision_settles_pending_cancels](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/crates/node-db/src/tests/session/turn_cancel.rs#L220)：已受理的 `EndSession` 使其他结论被拒绝；[an_end_during_the_restore_settles_cancels_before_the_restore_winds_down](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/apps/ora-node/tests/agent_session/resume.rs#L314)：结论可见时终态尚未写入；[an_end_session_accepted_during_the_recheck_takes_precedence](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/apps/ora-node/tests/agent_session/turn_processes/order.rs#L344)：判定期间到达的 `EndSession` 同样优先 | Covered |

### 决策依据

D4（`EndSession` 优先），不变量 6。

## Session-level notifications do not break quiescence

### 风险

Agent 在两轮之间正常发送的标题、用量等通知被当作违约，本可续用的会话被结束。

### 前置状态与触发

取消后 Agent 回应，静默窗口内只收到会话级通知（标题、用量、可用命令）。

### 必须成立

会话保留；这些通知不记入任何一轮。

### 禁止结果

会话以 `turn_not_quiescent` 结束；通知记入某一轮。

### 验证义务与证据

| 义务 | 直接证据与缺口 | 状态 |
|---|---|---|
| 22：会话级通知不影响静止判定 | [only_a_quiet_settled_turn_keeps_the_session](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/apps/ora-node/tests/agent_session/cancel.rs#L246) 的 `[hold-titled]`：静默窗口内只收到标题更新时取消得到 `stopped`，会话保留。缺口：用量与可用命令通知没有用例；没有断言通知不记入任何一轮。后续：[circus#1411](https://github.com/ora-space/circus/issues/1411) | Partial |

### 决策依据

D2（轮次流量与会话级通知），不变量 13。

## Cancel, admission and MCP wait timeout have one result

### 风险

在 MCP 门禁中等待的轮次同时被取消、准入和等候超时，产生两个结果，或者取消之后仍被发给 Agent。

### 前置状态与触发

目标轮次已到队首、正在等 MCP 门禁或会话接入，分别让取消、等候超时、准入先提交；另对初始轮次（没有命令）覆盖三种顺序；
再让等候满 300 秒而超时任务还没有提交，目标分别在队首与仍排在队首之后。

### 必须成立

- 取消先提交：`TurnCancelSettled{not_started}`，目标命令 Discarded，之后到 300 秒也没有 `SessionTurnFailed`，没有 `TurnEnded`；MCP 收敛继续，下一轮在门禁放行后照常准入。
- 等候超时先提交：`SessionTurnFailed{mcp_refresh_wait_timeout}` 与 Failed 已落账，之后生效的 `CancelTurn` 得到 `already_ended`，不改写失败。
- 等候已满 300 秒、超时任务还没有提交：取消生效时先落账 `SessionTurnFailed{mcp_refresh_wait_timeout}`，再得到 `already_ended`，不是 `not_started`；准入也不能提交。队首与排在队首之后的轮次相同。
- 准入先提交：取消按"已准入"或"正在执行"处理。
- 初始轮次的三种结果互斥，各至多记录一次。

### 禁止结果

同一轮既有 `not_started` 又有 `SessionTurnFailed`；取消后这一轮被准入；取消中止了 MCP 收敛。

### 验证义务与证据

| 义务 | 直接证据与缺口 | 状态 |
|---|---|---|
| 23：准入前三方竞争只有一个结果 | 取消先提交：[a_lifecycle_restore_failure_blocks_without_creating_a_provider](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/apps/ora-node/tests/agent_session/cancel/recovery.rs#L56)（MCP 门禁 blocked 中的轮次）、[a_cancel_during_model_authorization_settles_before_the_grant](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/apps/ora-node/tests/agent_session.rs#L943) 与 [cancels_during_the_restore_settle_without_waiting_for_it](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/apps/ora-node/tests/agent_session/resume.rs#L241)（初始轮次在接入前）得到 `not_started`。超时先提交：[the_first_commit_decides_a_waiting_turn](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/crates/node-db/src/tests/session/turn_cancel.rs#L163)、[a_cancel_of_a_timed_out_queued_turn_is_already_ended](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/crates/node-db/src/tests/session/turn_cancel.rs#L553)、[an_expired_queued_wait_is_timed_out_before_a_cancel_decides_it](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/apps/ora-node/src/session/driver/conversation/admission/tests.rs#L82)。准入先提交：[the_first_commit_decides_a_waiting_turn](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/crates/node-db/src/tests/session/turn_cancel.rs#L163) 拒绝丢弃，[timeout_and_actor_commit_have_one_winner_at_the_deadline](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/apps/ora-node/src/session/driver/conversation/admission/tests.rs#L5) 固定准入与超时只有一个胜者。缺口：取消之后到 300 秒没有 `SessionTurnFailed`、MCP 收敛继续且下一轮在门禁放行后准入，没有用例；初始轮次只覆盖取消先提交；超时先提交与到期未提交只在账本与判定函数层，没有真实等候超时与取消的会话级用例，队首的到期分支没有单独测试；前序轮次仍在执行时取消排队轮次，业务运行时间会被算进等候而提前判超时（实现偏差，见决策的差异表），没有测试。后续：[circus#1411](https://github.com/ora-space/circus/issues/1411)，计时偏差 [circus#1417](https://github.com/ora-space/circus/issues/1417) | Partial |

### 决策依据

D1（取消、准入与 MCP 等候超时的先后），不变量 1。

## A cancel after admission but before the prompt write keeps the session

### 风险

准入已提交、prompt 还没写出时受理取消，Agent 仍收到这次 prompt，或者会话被无谓地结束。

### 前置状态与触发

Actor 已准入目标轮次，`session/prompt` 还没有写给 Agent 时受理 `CancelTurn`。

### 必须成立

Agent 从未收到这次 prompt；事件为 `TurnEnded{cancelled}` → `TurnCancelSettled{stopped}`；会话保留，与轮次边界清单无关。

### 禁止结果

prompt 被写出；会话以 `turn_boundary_unverified` 或其他兜底原因结束。

### 验证义务与证据

| 义务 | 直接证据与缺口 | 状态 |
|---|---|---|
| 24：准入后、写出前的取消直接保留会话 | [a_cancel_before_the_prompt_write_never_reaches_the_agent](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/crates/agent-runtime/src/turn_cancel_tests.rs#L12)：准入提交点上的取消使 prompt 从未写出、这一轮以 `cancelled` 结束、下一轮可用；[a_cancel_during_admission_keeps_the_session_with_the_empty_list](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/apps/ora-node/tests/agent_session/cancel.rs#L618)：清单为空的 Node 上 `TurnEnded` 早于 `TurnCancelSettled{stopped}`，会话接着执行下一轮 | Covered |

### 决策依据

D1（已准入，`session/prompt` 还没有写给 Agent），不变量 7、14。

## Cancels merged into a watchdog stop share one deadline

### 风险

后到的取消重新起算宽限期，把先到的取消拖过它自己的期限；或者看门狗在用户取消后仍重发这一轮。

### 前置状态与触发

看门狗取消期间受理第一个 `CancelTurn`（A），在 A 的 G 即将到期时受理第二个 `CancelTurn`（B）；Agent 分别恰在截止时刻之前、之后回应。

### 必须成立

- 截止时刻为 A 的受理时刻加 G，B 不延长它。
- 截止前回应时两者都得到 `stopped`；截止后回应时两者都得到 `session_ending`，且 A 的结论不晚于 A 受理后 G + Q。
- 这一轮不再重发。

### 禁止结果

B 延长截止时刻；A 的结论超过 G + Q；看门狗重发这一轮。

### 验证义务与证据

| 义务 | 直接证据与缺口 | 状态 |
|---|---|---|
| 25：看门狗并入后只有一个截止时刻 | [a_cancel_merged_into_a_watchdog_stop_owns_its_deadline](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/crates/agent-runtime/src/turn_cancel_tests.rs#L52)：看门狗停止期间并入的第一个取消把截止时刻改为自己的、第二个不延长，截止前回应按取消结束、截止后报 `turn_cancel_unanswered`，prompt 不重发；[the_first_deadline_is_kept](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/crates/agent-runtime/src/turn_cancel.rs#L386)：第一个截止时刻不变。缺口：在运行时层以截止时刻驱动，没有 Node 层两个 `CancelTurn` 得到相同 `TurnCancelSettled` 的用例，也没有断言 A 的结论不晚于 G + Q；第二个取消不是在 A 的 G 即将到期时到达。后续：[circus#1411](https://github.com/ora-space/circus/issues/1411) | Partial |

### 决策依据

D1（一次停止只有一个截止时刻）、D2（时间参数），不变量 4、14。

## Leak detection spans all later turns but not normally completed ones

### 风险

被取消的一轮隔了几轮才到达的迟到更新漏检；或者正常完成的一轮之后的空闲期更新误结束会话。

### 前置状态与触发

被取消的一轮之后隔了一轮以上，才到达引用它的工具调用身份的更新；另覆盖一轮正常完成后空闲期到达的更新。

### 必须成立

前者会话以 `turn_leak_detected` 结束；后者按运行时既有规则丢弃，不结束会话。

### 禁止结果

隔轮的泄漏漏检；正常完成后的空闲更新结束会话。

### 验证义务与证据

| 义务 | 直接证据与缺口 | 状态 |
|---|---|---|
| 26：跨轮次检测的范围 | [a_leaked_update_in_a_later_turn_ends_the_session](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/apps/ora-node/tests/agent_session/cancel.rs#L290)：隔了一轮之后到达的更新仍以 `turn_leak_detected` 结束会话。缺口：正常完成的一轮之后空闲期到达的更新不结束会话，没有用例。后续：[circus#1411](https://github.com/ora-space/circus/issues/1411) | Partial |

### 决策依据

D2（跨轮次检测），不变量 8。

## A restart before the conclusion settles the cancel as session ending

### 风险

Node 在取消结论写入前重启，已受理的取消永远没有结论。

### 前置状态与触发

`CancelTurn` 受理之后、结论写入之前 Node 重启；另覆盖有未确认停止责任的情况。

### 必须成立

重启收口中这个命令得到 `session_ending`，早于 `interrupted` 终态；有未确认停止责任时，终态在取得原证据后才写入。

### 禁止结果

取消没有结论；结论晚于终态；没有停止证据就写终态。

### 验证义务与证据

| 义务 | 直接证据与缺口 | 状态 |
|---|---|---|
| 27：结论写入前重启的收口 | [restart_settles_a_pending_cancel_while_stop_evidence_is_missing](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/apps/ora-node/src/service/agents/tests.rs#L739)：重启收口先写 `session_ending`，缺停止证据时没有终态，重复收口不增加结论；[a_node_killed_during_the_fence_ends_the_session_and_stops_the_attempt](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/apps/ora-node/tests/standalone/agent_sessions/turn_fence_restart.rs#L267)：真实 Node 在取消结论写入前被 SIGKILL，重启后得到一条 `session_ending` 与 `interrupted`；[a_terminal_settles_pending_cancels_before_it](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/crates/node-db/src/tests/session/turn_cancel.rs#L273)：结论早于终态。缺口：证据后来取得、终态随之补发这一段没有带未结算取消的用例。后续：[circus#1411](https://github.com/ora-space/circus/issues/1411) | Partial |

### 决策依据

D3（重启后沿用结束决定）、D4（本地结算期限），不变量 2。

## A restart after the conclusion keeps the original end decision

### 风险

`session_ending` 已写入、停止信号还没发出时 Node 重启，重启收口按 `interrupted` 重新结算，丢掉原因和停止时钟，或者重复产生结论。

### 前置状态与触发

G 内没有回应，`TurnCancelSettled{session_ending}` 已写入（分别覆盖已投递、未投递），停止信号还没有发出时 Node 重启。

### 必须成立

- 重启后不重复产生取消结论。
- 终态原因不是 `interrupted`，仍是 `turn_cancel_unanswered`。
- 停止时钟沿用原来的 `first_stop_at_ms` 与 `bound_at_ms`，不续期。
- 取得原停止证据之后才写终态。

### 禁止结果

重复的 `TurnCancelSettled`；终态改记为 `interrupted`；停止时钟被重新锚定；没有证据就写终态。

### 验证义务与证据

| 义务 | 直接证据与缺口 | 状态 |
|---|---|---|
| 28：结论可见后重启沿用原结束决定 | [a_restart_after_the_settlement_keeps_the_original_decision](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/crates/node-db/src/tests/session/turn_cancel.rs#L316)：重启后沿用 `turn_cancel_unanswered`、不重复写结论；[the_stop_clock_commits_with_the_decision_and_survives_restart](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/crates/node-db/src/tests/session/turn_cancel.rs#L470)：停止时钟与决定同事务落账，重启后不被后来的停止请求改写。缺口：只在账本层；没有真实 Node 在停止信号发出前重启的用例，没有分别覆盖结论已投递、未投递，取得证据后才写终态没有在这条路径上断言。后续：[circus#1411](https://github.com/ora-space/circus/issues/1411) | Partial |

### 决策依据

D3（结束决定与结论同一事务落账、重启后沿用结束决定），不变量 15。

## A persistently unwritable ledger never settles from memory

### 风险

账本写失败时，Node 把内存状态当作结论发出，或者继续派发下一轮；恢复后重复补写或漏写结论。

### 前置状态与触发

`CancelTurn` 成功受理后，注入账本持续写失败；之后让账本恢复可写。

### 必须成立

- Node 不发送任何没有落账的结论，不再写出 `session/prompt`。
- Controller 超过 T 后按未确认收尾，不把超时或内存状态当成已结算。
- 账本恢复后，在终态之前为这个命令补写恰好一条 `session_ending`，结束原因为 `ledger_unavailable`。
- 恢复前已有结论的命令不重复写。

### 禁止结果

发出未落账的结论；账本不可写期间仍写出 `session/prompt`；Controller 把这次停止当作已停；一个命令有两条结论。

### 验证义务与证据

| 义务 | 直接证据与缺口 | 状态 |
|---|---|---|
| 29：账本持续不可写时的结算与恢复 | [an_unwritable_ledger_settles_only_after_recovery](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/crates/node-db/src/tests/session/turn_cancel.rs#L358)：写失败期间没有落账的结论，恢复后每个未结算取消恰好补一条 `session_ending`，已有结论的不重写；[a_failed_acceptance_stores_and_settles_nothing](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/crates/node-db/src/tests/session/turn_cancel.rs#L595)：受理写失败不存也不结算；[a_failed_end_decision_ends_with_ledger_unavailable](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/apps/ora-node/tests/agent_session/cancel.rs#L698)：结束决定写失败后以 `ledger_unavailable` 结束。缺口：会话层注入的是单次失败而不是持续失败，没有断言期间不写出 `session/prompt`；Controller 超过 T 后按未确认收尾，属 Controller 一侧。后续：会话层 [circus#1411](https://github.com/ora-space/circus/issues/1411)、Controller 一侧 [circus#1413](https://github.com/ora-space/circus/issues/1413) | Partial |

### 决策依据

D4（账本持续不可写），不变量 2。

## The cancel and failure choices act independently

### 风险

只选择取消后保留的会话在失败时也被保留，失败对调用方不可见；或者两个字段互相影响。

### 前置状态与触发

分别构造：只选择 `keep_session_on_cancel`；只选择 `keep_session_on_failure`；两者都选择；以及看门狗取消期间并入了 `CancelTurn` 的一轮。

### 必须成立

- 只选择 `keep_session_on_cancel`：取消后满足 D2 时会话保留；一轮失败时以 `agent_failed` 和既有 detail 结束，并入的 `CancelTurn` 得到 `session_ending`。
- 只选择 `keep_session_on_failure`：`CancelTurn` 被拒绝（`turn_cancel_unsupported`），一轮失败后满足 D2 时会话保留。
- 两者都选择：两种结束分别按对应规则。
- 看门狗取消期间并入了 `CancelTurn` 的一轮，按取消处理。
- （失败原因）选择 `keep_session_on_failure` 要求当前连接协商了 `turn_cancel` 与 `turn_failure`；只选择取消的会话没有
  `TurnFailureSettled`；看门狗取消期间并入了 `CancelTurn` 的一轮记为 `ended_by: caller_cancel`，不写 `failure`。

### 禁止结果

只选择取消的会话因失败而保留；只选择失败的会话受理 `CancelTurn`；并入取消的看门狗轮次按失败处理；（失败原因）只协商了
`turn_cancel` 的连接能选择失败后保留。

### 验证义务与证据

| 义务 | 直接证据与缺口 | 状态 |
|---|---|---|
| 30：两个字段的组合 | 只选取消：[a_failed_turn_keeps_only_a_session_that_chose_it](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/apps/ora-node/tests/agent_session/cancel.rs#L362)：一轮失败以 `agent_turn_failed` 结束，取消后保留见用例 1。只选失败：同一测试保留会话，[a_session_without_the_cancel_choice_refuses_cancel_turn](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/crates/node-db/src/tests/session/turn_cancel.rs#L89) 拒绝 `CancelTurn`。两者都选：[a_cancel_during_a_failed_turns_quiet_window_is_already_ended](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/apps/ora-node/tests/agent_session/cancel.rs#L504) 的失败按失败规则。看门狗并入：[a_cancel_merged_into_a_watchdog_stop_owns_its_deadline](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/crates/agent-runtime/src/turn_cancel_tests.rs#L52) 报 `UserStopped`。缺口：只选取消的会话在一轮失败时并入的取消得到 `session_ending` 没有用例；两者都选时的取消、Node 层看门狗并入按取消处理没有用例。后续：[circus#1411](https://github.com/ora-space/circus/issues/1411) | Partial |
| 30（失败原因）：失败后保留要求协商 `turn_failure`；只选取消的会话没有 `TurnFailureSettled`；看门狗并入取消的一轮记为 `caller_cancel`；义务见[失败原因用例](turn-failure-reason.md) 8、13、14 | 尚未实现，无直接证据 | Missing |

### 决策依据

D2（结束方式表）、D5（两个字段与启用顺序），不变量 7、11；[失败原因决策](../../../decisions/node/protocol/20261124-turn-failure-reason-distinct-from-cancel.md) D1、D6。
