# 轮次进程栅栏

依据[保留会话前先停掉这一轮新起的进程决策](../../../decisions/node/protocol/20261123-turn-process-fence-before-keeping-session.md)
（`implemented`）。本文的 8 个核心用例与该决策“验证义务”一节的第 1–8 条一一对应，用例标题前的编号
就是验证义务的编号。决策由 [desktop#807](https://github.com/ora-space/desktop/pull/807) 实现，证据按合并提交
[`447a59de`](https://github.com/ora-space/desktop/commit/447a59de6722500cae0be011860e63e5127a3b23) 登记：21 项义务中
11 项 `Covered`、9 项 `Partial`、1 项 `Missing`，缺口逐项写在表中。Node 级测试用 echo-agent fixture 模拟部署插件的工具
shell（`setsid` 后放到后台的作业、Agent 会话中的 helper），栅栏层测试直接调用栅栏与复扫并注入时钟或扫描失败。
真实插件经 Node 的一致性运行是登记清单的运行记录，不作为本文证据。该决策对取消决策验证义务 10、13 的修改登记在
[取消一轮用例](cancel-turn-and-keep-session.md)的对应用例中；结束会话时停止整个尝试的义务继续由
[会话 MCP 刷新与宿主健康用例](../agent-runtime/session-mcp-lifecycle.md)约束，本文不推导它们的证据。

| 验证义务 | 核心用例 |
| --- | --- |
| 1 | [A kept cancel kills the turn's detached jobs before settling stopped](#a-kept-cancel-kills-the-turns-detached-jobs-before-settling-stopped) |
| 2 | [The fence spares lazily started processes in the Agent's own session and everything older than the turn](#the-fence-spares-lazily-started-processes-in-the-agents-own-session-and-everything-older-than-the-turn) |
| 3 | [The turn scope follows descendants that clear their environment](#the-turn-scope-follows-descendants-that-clear-their-environment) |
| 4 | [An unclean fence ends the session as turn processes unstopped](#an-unclean-fence-ends-the-session-as-turn-processes-unstopped) |
| 5 | [The fence runs between the static checks and the quiet window within its own bound](#the-fence-runs-between-the-static-checks-and-the-quiet-window-within-its-own-bound) |
| 6 | [Normal completion and sessions that cannot keep never run the fence](#normal-completion-and-sessions-that-cannot-keep-never-run-the-fence) |
| 7 | [A restart around the fence ends the session and stops the whole attempt](#a-restart-around-the-fence-ends-the-session-and-stops-the-whole-attempt) |
| 8 | [Forged marks only kill the forger's own turn processes or end the session](#forged-marks-only-kill-the-forgers-own-turn-processes-or-end-the-session) |

## A kept cancel kills the turn's detached jobs before settling stopped

### 风险

会话在取消后被保留，而这一轮工具起的后台作业仍在运行，与下一轮并发修改 checkout（circus#1339 的反例）；或者下一轮
在栅栏完成之前就被派发。

### 前置状态

Linux；会话选择了 `keep_session_on_cancel`；Agent 的（插件，版本）为 Linux 登记；一轮正在执行，它的工具在新会话中
起了后台作业（工具命令以 `setsid` 启动，作业放到后台），作业会在 Agent 回应之后继续写 checkout；Controller 已提交下一轮。

### 触发

Controller 发 `CancelTurn`，Agent 在截止时刻前回应，工具调用都已终态，没有子代理投影。

### 必须成立

- `TurnCancelSettled{stopped}` 写入时，这些作业都已不在；之后 checkout 不再被它们写入。
- 事件顺序为 `TurnEnded{cancelled}` → `TurnCancelSettled{stopped}` → 下一轮的 Thread 事件；下一轮的 `session/prompt` 在
  `stopped` 之后才写出。
- 作业在回应与被杀之间写下的内容留在 checkout 中。
- 栅栏干净之后，Agent 这类豁免进程不发轮次流量地新起一个 `setsid` 进程，它在结算前的复扫时仍存活：会话以
  `turn_processes_unstopped` 结束，不写 `stopped`；复扫不向它发信号，它由结束会话的停止杀掉。

### 禁止结果

`stopped` 写入时作业或栅栏之后新起的范围内进程仍存活；`stopped` 之后出现作业的写入；下一轮在栅栏与复扫完成之前写出
prompt；栅栏向 Agent 进程或其进程组发信号；复扫发出任何信号。

### 验证义务与证据

| 义务 | 直接证据与缺口 | 状态 |
|---|---|---|
| 1：保留会话的取消在 `stopped` 之前杀掉这一轮新会话中的后台作业，下一轮在其后派发 | [`a_kept_cancel_kills_the_turns_setsid_job_before_it_settles`](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/apps/ora-node/tests/agent_session/turn_processes.rs#L125)：echo-agent 的工具在新会话中起一个 3 秒后写 checkout 的作业，取消以 `stopped` 落账的那一刻作业已退出，之后没有写入，会话接着完成下一轮。缺口：下一轮是测试看到 `stopped` 之后才提交的，没有“下一轮在取消之前已排队、它的 `session/prompt` 在 `stopped` 之后才写出”的断言；作业在回应与被杀之间的写入留在 checkout 中也没有断言。后续：[circus#1407](https://github.com/ora-space/circus/issues/1407) | Partial |
| 1（复扫）：栅栏之后豁免进程静默新起的 `setsid` 进程让会话以 `turn_processes_unstopped` 结束 | [`a_job_started_after_a_clean_fence_ends_the_session`](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/apps/ora-node/tests/agent_session/turn_processes/order.rs#L30)：Agent 会话中的 helper 在栅栏杀掉它的哨兵作业 1 秒后、Q（4 秒）之内，不发轮次流量地起一个 `setsid` 作业；结果没有 `stopped`，取消为 `session_ending`，detail 为 `turn_processes_unstopped`，终态时作业已被结束会话的停止杀掉。[`the_recheck_finds_a_job_started_after_a_clean_fence_and_kills_nothing`](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/apps/ora-node/src/session/stop/tests/turn.rs#L111) 直接调用复扫，断言它对这样的作业报告 Running，作业仍存活，即复扫不发信号 | Covered |

### 决策依据

D1、D2（结算前的复扫）、D4，不变量 3、5。

## The fence spares lazily started processes in the Agent's own session and everything older than the turn

### 风险

栅栏杀掉 Agent 懒启动、要跨轮次使用的 LSP 或 MCP，保留下来的会话功能退化（实测“本轮新起一律杀”会杀掉
`deno lsp`）；或者栅栏碰到轮次开始前就存在的进程。

### 前置状态

同上一条；此外，之前正常完成的一轮工具在自己的新会话里起了一个作业或开发服务器，仍在运行；被取消的这一轮里，
Agent 在自己的会话中懒启动了一个常驻进程（例如 LSP 或 MCP），它又起了子进程。

### 触发

取消这一轮，Agent 回应，栅栏运行。

### 必须成立

- 懒启动的常驻进程及其同一会话中的子进程在栅栏之后存活，下一轮找到的是同一个进程。
- 之前一轮留下的作业、插件运行时、Agent CLI 以及轮次开始前的任何进程，都没有收到冻结或杀信号；之前一轮留下的作业
  只在会话结束时被杀。

### 禁止结果

豁免会话中的进程被冻结或杀掉；轮次开始前就存在的进程收到信号；Agent 的进程组被终止。

### 验证义务与证据

| 义务 | 直接证据与缺口 | 状态 |
|---|---|---|
| 2：豁免会话中的懒启动进程与轮次开始前的进程都不收到信号 | [`the_agents_own_helper_survives_a_kept_cancel`](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/apps/ora-node/tests/agent_session/turn_processes.rs#L157)：Agent 在自己会话中起的 helper 在保留会话的取消后存活，下一轮报告的是同一 PID，旁边的本轮 `setsid` 作业被杀；[`jobs_of_a_finished_turn_survive_the_next_turns_cancel`](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/apps/ora-node/tests/agent_session/turn_processes.rs#L195)：正常完成的一轮留下的 `setsid` 作业与 helper 在下一轮被取消、会话保留后仍存活，作业在会话结束时才被杀。两例中 Agent 进程都继续服务后面的轮次 | Covered |

### 决策依据

D1（轮次范围、为什么用 sid）、D7，不变量 3。

## The turn scope follows descendants that clear their environment

### 风险

轮次范围内的进程 fork 出清空环境的子进程，栅栏先杀父进程，子进程成为孤儿后再也认领不到，在会话保留后继续写入；
反过来，豁免会话中常驻进程的子进程被当作这一轮的进程杀掉；或者一个已经没有更早被认领成员的会话、一个已经消失的
会话的编号，让这一轮新起的进程被误豁免。

### 前置状态

这一轮的后台作业（新会话中）fork 出用 `env -i` 清空环境的子进程；更早启动、在另一个会话中的 helper 在这一轮里起了
一个子进程；某个更早的会话里只有不带标记的进程，这一轮有带标记的进程留在其中。

### 触发

取消这一轮，Agent 回应，栅栏运行。

### 必须成立

- 清空环境的子进程经被冻结的父进程认领，与父进程一起被杀；栅栏结论只在它们都不在之后才是干净。
- helper 的子进程留在 helper 的会话中，不被杀。
- 只含不带标记的更早进程的会话不豁免任何进程，留在其中的这一轮进程被杀；豁免会话只取自同一遍扫描中存活的更早被认领
  进程。
- 待定进程只在范围内计数：更早启动、读不到环境的进程不使栅栏为 Unknown；这一轮新起、不在豁免会话中、读不到环境的进程
  使栅栏为 Unknown。

### 禁止结果

清空环境的子进程在 `stopped` 之后存活；豁免会话中常驻进程的子进程被杀；不被认领的更早进程或已消失的会话让这一轮的
进程被豁免。

### 验证义务与证据

| 义务 | 直接证据与缺口 | 状态 |
|---|---|---|
| 3：轮次范围对后代封闭，豁免会话的后代不在范围内，豁免只来自同一遍扫描中存活的更早被认领进程，待定进程只在范围内计数 | [`a_new_session_reach_spares_the_sessions_of_earlier_claimed_processes`](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/crates/utils/src/process/environment_mark/scan.rs#L332)：收窄规则的表驱动单测。经父进程认领、清空环境并自建会话的较晚进程，以及留在“只含不带标记的更早进程”的会话中的较晚进程被报告；更早被认领进程会话中的较晚进程不被报告；较早的不可读进程不计入待定，较晚且不在豁免会话中的不可读进程计入（去掉启动时刻过滤时失败）。[`a_new_session_scope_kills_later_jobs_and_spares_earlier_sessions`](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/crates/utils/tests/environment_mark.rs#L167)：真实进程下只有较晚的 `setsid` 作业被报告并被杀，更早的进程、自建会话的 helper 及它在自己会话中的较晚子进程、留在更早进程会话中的较晚进程都存活。[`children_that_cleared_their_environment_die_with_the_turn`](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/apps/ora-node/tests/agent_session/turn_processes/scope.rs#L26)：本轮 `setsid` shell 持续 fork 清空环境的子进程，取消以 `stopped` 结算时一个都不剩（把冻结改成直接杀时失败） | Covered |

### 决策依据

D1（范围对后代封闭）、D2，不变量 3、4。

## An unclean fence ends the session as turn processes unstopped

### 风险

栅栏判断不了这一轮的进程是否停下，或杀不掉它们，会话仍被保留；或者会话结束时没有可解释的 detail、并入的取消没有结论。

### 前置状态

会话选择了保留，版本已登记，一轮正在执行。

### 触发

分别注入：Agent 回应后不断起新的 `setsid` 作业，直到 F 用完；尝试的范围记录被删除；栅栏的扫描失败；范围内有待定进程
（Node 与 workload 不同用户、没有 `CAP_SYS_PTRACE`）；观察排在其他会话的停止观察之后超出预算；这一轮没有开始时刻（时钟
读取失败）；接收这一轮 prompt 的尝试在这一轮里被 MCP 生命周期替换；结算前的复扫超出 R 或扫描失败。之后取消这一轮或让它
失败，Agent 回应。

### 必须成立

- 会话以 `agent_failed`、detail `turn_processes_unstopped` 结束；结束决定与各个 `TurnCancelSettled{session_ending}` 同一事务
  落账，之后不再写出 `session/prompt`。
- 提前返回的不干净观察只在 F 剩余时重来，最后一次观察不干净才结束；没有开始时刻、范围记录不可用或尝试已被替换时
  不发信号，直接以同一 detail 结束。
- 结束后停止整个尝试（替换时覆盖两个尝试），栅栏没杀掉的进程在终态之前被杀；终态在取得停止证据之后才写入。

### 禁止结果

任何一种情况下会话被保留；以其他 detail 结束；并入的取消没有结论或得到 `stopped`；终态早于停止证据。

### 验证义务与证据

| 义务 | 直接证据与缺口 | 状态 |
|---|---|---|
| 4：F 内不干净（持续出现新的范围内进程、范围记录缺失、观察超出预算）时以 `turn_processes_unstopped` 结束，并入的取消得到 `session_ending`；提前返回的不干净观察在 F 剩余时退避重来 | [`jobs_that_keep_coming_back_end_the_session`](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/apps/ora-node/tests/agent_session/turn_processes.rs#L237)（Agent 回应后不断重起 `setsid` 作业，F 为 1 秒）、[`an_unknown_fence_ends_the_session`](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/apps/ora-node/tests/agent_session/turn_processes.rs#L264)（范围记录被删除）、[`a_fence_past_its_budget_ends_the_session`](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/apps/ora-node/tests/agent_session/turn_processes/limits.rs#L160)（F 为 0，第一次观察来不及开始）都断言 detail 与 `session_ending`；[`a_cancel_accepted_during_an_unclean_fence_shares_its_end`](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/apps/ora-node/tests/agent_session/turn_processes/order.rs#L109) 断言栅栏期间受理的第二个取消同为 `session_ending`；[`observations_back_off_until_one_is_clean_or_the_bound_passes`](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/apps/ora-node/src/session/stop/tests/turn.rs#L149) 在暂停的时钟下断言不干净的观察按 25 ms 起倍增的退避重来，干净即停，剩余时间不够一次退避时不再观察，不可重试的观察立即为 Unknown | Covered |
| 4：扫描失败、没有开始时刻、尝试在这一轮里被替换 | [`a_failing_scan_is_never_stopped`](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/apps/ora-node/src/session/stop/tests/turn.rs#L189)（注入失败的扫描：栅栏与复扫都为 Unknown，栅栏仍杀掉自己扫到的作业）、[`a_turn_without_a_recorded_start_is_never_stopped`](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/apps/ora-node/src/session/stop/tests/turn.rs#L59)（注入失败的时钟：记为没有开始时刻，栅栏与复扫为 Unknown，作业没有收到信号）、[`an_attempt_replaced_during_the_turn_is_never_stopped`](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/apps/ora-node/src/session/stop/tests/turn.rs#L82)（尝试被替换：栅栏为 Unknown，作业没有收到信号）。判定把任何不干净的结论都映射为同一 detail，这一点由上一行的会话级测试证明。缺口：这三种注入只在栅栏层断言 Unknown，没有经会话判定断言 detail、`session_ending` 的测试；替换时结束会话的停止覆盖新旧两个尝试也没有测试。后续：[circus#1407](https://github.com/ora-space/circus/issues/1407) | Partial |
| 4：范围内有待定进程（Node 与 workload 不同用户、没有 `CAP_SYS_PTRACE`） | [`a_new_session_reach_spares_the_sessions_of_earlier_claimed_processes`](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/crates/utils/src/process/environment_mark/scan.rs#L332) 断言范围内、不在豁免会话中的不可读进程计为待定；待定进程使观察为 Unknown 是标记决策既有的证据规则。缺口：没有 workload 以另一用户运行、范围内留下不可读进程时栅栏为 Unknown 并以 `turn_processes_unstopped` 结束的会话级测试。后续：[circus#1407](https://github.com/ora-space/circus/issues/1407) | Partial |
| 4（复扫）：复扫为 Running、Unknown 或超出 R 时以 `turn_processes_unstopped` 结束 | [`a_recheck_past_its_budget_ends_the_session`](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/apps/ora-node/tests/agent_session/turn_processes/order.rs#L374)（R 为 0：取消为 `session_ending`，detail 为 `turn_processes_unstopped`）；[`a_job_started_after_a_clean_fence_ends_the_session`](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/apps/ora-node/tests/agent_session/turn_processes/order.rs#L30)（复扫为 Running）；[`a_failing_scan_is_never_stopped`](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/apps/ora-node/src/session/stop/tests/turn.rs#L189) 与 [`a_spent_budget_is_never_stopped`](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/apps/ora-node/src/session/stop/tests/turn.rs#L215)（扫描失败或预算用完时复扫为 Unknown，干净的范围在有预算时为干净）。缺口：与上面“扫描失败”一行同一口径，扫描失败造成的复扫 Unknown 只在栅栏层断言，经会话判定的复扫 Unknown 只有 R 为 0 这一种。后续：[circus#1407](https://github.com/ora-space/circus/issues/1407) | Partial |
| 4：结束之后不再写出 `session/prompt`，停止整个尝试、取得停止证据之后才写终态 | [`a_job_started_after_a_clean_fence_ends_the_session`](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/apps/ora-node/tests/agent_session/turn_processes/order.rs#L30) 断言终态写入时栅栏之后起的作业已被结束会话的停止杀掉。缺口：没有断言结束之后不再写出 `session/prompt` 的栅栏用例。结束会话的停止证据属于[会话 MCP 刷新与宿主健康用例](../agent-runtime/session-mcp-lifecycle.md#a-session-stop-kills-every-attempt-marked-process-and-finished-turns-kill-none)的义务，本文不推导。后续：[circus#1407](https://github.com/ora-space/circus/issues/1407) | Partial |

### 决策依据

D1（轮次开始）、D2、D3，不变量 4；取消决策 D3。

## The fence runs between the static checks and the quiet window within its own bound

### 风险

栅栏放错位置：在已决定结束会话时仍运行，推迟结论；在静默窗口之后运行，杀进程引起的 Agent 反应混进下一轮；栅栏
期间到达的轮次流量被忽略；或者栅栏的耗时让结论超出期限、并入的取消与 `EndSession` 得到不一致的结果。

### 前置状态

会话选择了保留，版本已登记。

### 触发

分别构造：回应时仍有未终态的工具调用；版本未为本平台登记；栅栏杀掉作业后 Agent 发来一条轮次流量（例如自发起一轮）；
栅栏期间受理第二个 `CancelTurn`；栅栏期间受理 `EndSession`（栅栏分别干净与不干净）；复扫期间分别受理第二个
`CancelTurn` 与 `EndSession`；一轮因 ACP 错误失败（选择了 `keep_session_on_failure`），栅栏运行到接近 F。

### 必须成立

- 前两种情况不运行栅栏，分别以 `turn_not_quiescent`、`turn_boundary_unverified` 结束，结论期限与取消决策相同。
- 栅栏期间的轮次流量使会话以 `turn_not_quiescent` 结束；静默窗口 Q 从栅栏干净时起算。
- 栅栏不因命令中断；第二个 `CancelTurn` 在栅栏返回后并入，与这次停止得到同一结论；`EndSession` 在栅栏返回后优先，
  无论栅栏是否干净，会话都以它的原因结束，未结算的取消得到 `session_ending`。
- 结算前的复扫在 Q 与空闲期流量检查之后、`stopped` 之前运行，只扫描不杀，不因命令中断。复扫期间受理的 `CancelTurn`
  在它返回后并入这次停止，与它同为 `stopped`（复扫不干净时同为 `session_ending`）；复扫期间受理的 `EndSession` 优先，
  不写 `stopped`，会话以它的原因结束，未结算的取消得到 `session_ending`。
- 结论在发起的 `CancelTurn` 受理后 G + F + Q + R（23 秒）内落账；失败发起时在回应后 F + Q + R 内落账。

### 禁止结果

已决定结束会话后仍运行栅栏；Q 在栅栏之前或与之并行结束；栅栏期间的轮次流量不被计入；复扫早于 Q 或晚于 `stopped`；
两个取消得到不同结论；复扫期间受理的 `CancelTurn` 得到 `already_ended`；复扫期间受理 `EndSession` 后仍写出 `stopped`；
`EndSession` 之后会话仍被保留；结论超出期限。

### 验证义务与证据

| 义务 | 直接证据与缺口 | 状态 |
|---|---|---|
| 5：选择、登记或静止条件 2 不满足时不运行栅栏 | [`a_stop_that_cannot_keep_the_session_runs_no_fence`](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/apps/ora-node/tests/agent_session/turn_processes/limits.rs#L56)：没有选择失败保留的失败、未登记的版本、回应时仍有运行中的工具三种情况，在结束决定写入的那一刻本轮 `setsid` 作业仍存活（栅栏会在决定之前杀掉它），detail 分别为 `agent_turn_failed`、`turn_boundary_unverified`、`turn_not_quiescent` | Covered |
| 5：栅栏在 Q 之前，杀进程引起的轮次流量使会话以 `turn_not_quiescent` 结束 | [`turn_traffic_caused_by_the_fence_ends_the_session`](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/apps/ora-node/tests/agent_session/turn_processes/order.rs#L58)：Agent 监视的作业被栅栏杀掉后，Agent 发出一条消息，会话以 `turn_not_quiescent` 结束（把栅栏挪到 Q 之后时失败） | Covered |
| 5：复扫在 Q 与空闲期流量检查之后、`stopped` 之前 | [`a_job_started_after_a_clean_fence_ends_the_session`](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/apps/ora-node/tests/agent_session/turn_processes/order.rs#L30)：作业在栅栏之后 1 秒才出现、Q 为 4 秒，复扫若在 Q 之前运行就看不到它；会话没有写 `stopped`（去掉复扫时失败） | Covered |
| 5：栅栏期间受理的 `CancelTurn` 与 `EndSession` 在栅栏返回后按取消决策处理 | [`a_cancel_accepted_while_the_stop_is_judged_settles_stopped`](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/apps/ora-node/tests/agent_session/turn_processes/order.rs#L80)（栅栏最终干净：第二个取消在判定期间受理，两者都为 `stopped`，会话保留）；[`a_cancel_accepted_during_an_unclean_fence_shares_its_end`](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/apps/ora-node/tests/agent_session/turn_processes/order.rs#L109)（栅栏不干净：两者都为 `session_ending`）；[`an_end_session_accepted_during_the_fence_takes_precedence`](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/apps/ora-node/tests/agent_session/turn_processes/order.rs#L136)（栅栏干净与不干净两种：`EndSession` 都优先，会话以它的原因结束，取消为 `session_ending`）。缺口：栅栏干净的两例只在第一个取消之后固定等 300 ms 再受理命令，没有断言命令落在栅栏期间；命令落在 Q 内时，复扫之后最后一次处理队列也会得到相同结论，所以栅栏吞掉命令的唤醒、或收到命令就提前按干净返回的实现同样能通过。不干净一侧命令确定落在栅栏期间，但两侧都没有断言“栅栏不因命令中断”。后续：[circus#1407](https://github.com/ora-space/circus/issues/1407) | Partial |
| 5：复扫期间受理的 `CancelTurn` 与 `EndSession` | [`a_cancel_accepted_during_the_recheck_merges_into_the_stop`](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/apps/ora-node/tests/agent_session/turn_processes/order.rs#L310) 与 [`an_end_session_accepted_during_the_recheck_takes_precedence`](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/apps/ora-node/tests/agent_session/turn_processes/order.rs#L344) 把范围记录换成 FIFO，在复扫读它时挂住并受理命令：复扫干净时第二个取消并入，两者都为 `stopped`；`EndSession` 优先，取消为 `session_ending`，不写 `stopped`。缺口：复扫不干净时并入的取消同为 `session_ending` 没有测试；决策 D4 记录的已知窗口（第 8 步逐条写入，`EndSession` 落在两条之间时并入的取消结论不同）尚未修复。后续：测试由 [circus#1407](https://github.com/ora-space/circus/issues/1407) 补，窗口由 [circus#1403](https://github.com/ora-space/circus/issues/1403) 修复 | Partial |
| 5：失败发起的栅栏与结论期限 | [`a_failure_the_session_survives_runs_the_fence`](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/apps/ora-node/tests/agent_session/turn_processes/limits.rs#L101)：选择了失败保留的一轮因 ACP 错误失败，本轮 `setsid` 作业在下一轮之前被杀、没有写出，会话完成下一轮；[`a_failure_fence_ends_within_its_bound`](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/apps/ora-node/tests/agent_session/turn_processes/limits.rs#L131)：失败发起、栅栏一直不干净时，结束决定在回应后 F（1 秒）加余量之内写入，不等 Q 与 R。缺口：没有断言取消发起时结论在 G + F + Q + R 内落账、或保留会话的失败在回应后 F + Q + R 内完成判定的测试。后续：[circus#1407](https://github.com/ora-space/circus/issues/1407) | Partial |

### 决策依据

D4，不变量 2、6；取消决策 D1、D4。

## Normal completion and sessions that cannot keep never run the fence

### 风险

栅栏扩大到它不该作用的路径：一轮正常完成后杀掉这一轮起的开发服务器；没有选择保留或未登记的会话也运行栅栏、
扫描 procfs；或者记下轮次开始给每一轮增加扫描。

### 前置状态

三个会话的某一轮都起了后台作业：第一个选择了保留、版本已为本平台登记；第二个没有选择失败后保留
（`keep_session_on_failure` 为假）；第三个选择了保留，但版本没有为本平台登记。

### 触发

第一个会话的这一轮正常完成，随后提交下一轮；第二个会话的这一轮因 ACP 错误失败；第三个会话的这一轮被取消，Agent 回应。
没有选择取消后保留（`keep_session_on_cancel` 为假）的会话，Node 在受理时就拒绝 `CancelTurn`，所以“没有选择保留”只能用
失败来构造。

### 必须成立

- 第一个会话不运行栅栏，不发任何冻结或杀信号；后台作业进入下一轮。
- 后两个会话不运行栅栏，按取消决策 D3 直接结束会话（分别为 `agent_turn_failed` 与 `turn_boundary_unverified`）；结束决定
  写入时本轮的后台作业仍存活；它之后如何停止由会话 MCP 生命周期用例约束。
- 记下轮次开始只是一次时钟读取，任何会话都不因轮次开始扫描 procfs。

### 禁止结果

正常完成的一轮之后有进程被冻结或杀掉；不能保留会话的停止运行栅栏；轮次开始时扫描 procfs。

### 验证义务与证据

| 义务 | 直接证据与缺口 | 状态 |
|---|---|---|
| 6：一轮正常完成后不发信号，作业进入下一轮 | [`jobs_of_a_finished_turn_survive_the_next_turns_cancel`](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/apps/ora-node/tests/agent_session/turn_processes.rs#L195)：正常完成一轮之后，作业与 helper 都存活，并活过下一轮保留会话的取消 | Covered |
| 6：没有选择保留、或版本未为本平台登记的会话不运行栅栏 | [`a_stop_that_cannot_keep_the_session_runs_no_fence`](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/apps/ora-node/tests/agent_session/turn_processes/limits.rs#L56)：没有选择失败后保留的会话一轮失败、版本未为本平台登记的会话一轮被取消时，结束决定写入的那一刻本轮作业仍存活 | Covered |
| 6：记下轮次开始不扫描 procfs | [`recording_a_turn_start_reads_only_the_clock`](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/apps/ora-node/src/session/stop/tests/turn.rs#L240)：删掉尝试的停止文件目录、换上必然失败的扫描之后，记下轮次开始仍成功，结果只是时钟读数与当前尝试。缺口：这只证明记下轮次开始不依赖扫描与范围记录；它若调用扫描却忽略结果，测试照样通过，没有直接断言记下轮次开始不扫描 procfs。后续：[circus#1407](https://github.com/ora-space/circus/issues/1407) | Partial |

### 决策依据

D1（轮次开始）、D4，不变量 1、7；标记决策 D4。

## A restart around the fence ends the session and stops the whole attempt

### 风险

轮次范围只在内存中，Node 崩溃后没人杀这一轮的进程；被栅栏冻结的进程永远停在 T 状态；或者为此改了账本 schema。

### 前置状态

会话选择了保留，版本已登记，这一轮起了后台作业。

### 触发

分别在以下时刻 SIGKILL Node 并重启：一轮开始之后、取消之前（这一轮正在执行，作业在运行）；栅栏冻结了作业、还没杀掉时；
`TurnCancelSettled{stopped}` 写入之后、下一轮执行中。轮次开始记下与 prompt 写出之间只有一次提交后的瞬间，无法单独
构造；崩溃时轮次开始只在内存中，两者的结果相同。

### 必须成立

- 会话以 `interrupted` 结束；尚未结算的 `CancelTurn` 在重启收口中得到 `session_ending`，早于终态。
- 重启收口按尝试的范围记录冻结并杀掉原尝试的全部被认领进程，包括被冻结的作业；取得停止证据之后才写终态。
- 账本 schema 版本与本决策之前相同。

### 禁止结果

会话在重启后被保留或恢复；本轮作业在终态之后存活；被冻结的进程留在 T 状态；为轮次开始新增账本表或列。

### 验证义务与证据

| 义务 | 直接证据与缺口 | 状态 |
|---|---|---|
| 7：Node 在一轮开始之后、栅栏之中、保留会话之后被杀并重启，会话以 `interrupted` 结束，未结算的取消得到 `session_ending`，本轮进程在重启收口中被杀 | [`a_node_killed_after_the_turn_started_ends_the_session_and_stops_the_attempt`](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/apps/ora-node/tests/standalone/agent_sessions/turn_fence_restart.rs#L294)（一轮正在执行、作业在运行时 SIGKILL：作业活过崩溃，重启后会话以 `interrupted` 结束，本尝试不剩带标记的进程，作业没有写出）；[`a_node_killed_during_the_fence_ends_the_session_and_stops_the_attempt`](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/apps/ora-node/tests/standalone/agent_sessions/turn_fence_restart.rs#L267)（栅栏冻结作业时 SIGKILL：断言崩溃时确有停在 T 状态的作业，重启后取消为 `session_ending`、会话以 `interrupted` 结束，被冻结的作业与本尝试的其他进程都不剩）；[`a_node_killed_after_a_kept_cancel_ends_the_session_and_stops_the_attempt`](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/apps/ora-node/tests/standalone/agent_sessions/turn_fence_restart.rs#L322)（取消以 `stopped` 保留会话、下一轮起了作业之后 SIGKILL：重启只重放 `stopped`，会话以 `interrupted` 结束，作业被杀）。三例都以独立进程运行 Node，会话使用 Node 编译进的清单所登记的身份 | Covered |
| 7：账本 schema 版本不变 | 没有测试断言。desktop#807 没有修改 `crates/node-db`，这是结构性事实，不作为证据。后续：[circus#1407](https://github.com/ora-space/circus/issues/1407) | Missing |

### 决策依据

D6，不变量 8；协议 D6；取消决策 D3、D4；标记决策 D2、D3。

## Forged marks only kill the forger's own turn processes or end the session

### 风险

伪造本尝试标记的进程借栅栏杀掉不归这一轮的进程，或让会话在仍有本轮进程存活时被保留。

### 前置状态

同 uid 的进程读到了标记；它在这一轮开始之前启动了一个带伪造标记的进程，又在这一轮开始之后、在新会话中启动了另一个；
另一种情形下，它在这一轮里持续 fork 带标记的进程。

### 触发

取消这一轮，Agent 回应，栅栏运行。

### 必须成立

- 轮次开始前启动的伪造进程不收到任何信号：它让自己所在的会话豁免。
- 轮次开始后、新会话中的伪造进程被杀。
- 持续 fork 时栅栏在 F 内不干净，会话以 `turn_processes_unstopped` 结束。

### 禁止结果

轮次开始前的进程收到信号；持续 fork 期间会话被保留。

### 验证义务与证据

| 义务 | 直接证据与缺口 | 状态 |
|---|---|---|
| 8：伪造只影响伪造者自己在这一轮新会话中的进程，或让会话结束 | [`a_forged_mark_only_kills_the_forgers_own_turn_processes`](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/apps/ora-node/tests/agent_session/turn_processes/scope.rs#L134)：轮次开始前带伪造标记的两个进程（自建会话的伪造者与留在测试会话中的进程）都存活，且这两个都没有停在 T 状态；伪造者在这一轮里于自己会话中起的子进程存活；伪造者在这一轮里于新会话中起的子进程、轮次开始后在新会话中启动的伪造进程被杀；不带标记的进程不受影响；取消以 `stopped` 结算。伪造者的父进程是测试进程，不被认领，所以这证明伪造者只凭标记就被认领，并让自己的会话豁免。持续 fork 一项由这个测试与 [`jobs_that_keep_coming_back_end_the_session`](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/apps/ora-node/tests/agent_session/turn_processes.rs#L237) 组合证明：后者中一个更早启动、被认领、会话被豁免的进程（Agent）在回应后不断起 `setsid` 子进程，会话以 `turn_processes_unstopped` 结束；被认领之后的伪造者在范围判定中处于同样的位置（更早启动、被认领、会话被豁免），两者的拓扑相同，只是认领路径不同（伪造者只凭标记被认领，Agent 及其重起的作业另经被认领的父进程认领），而范围判定不看认领路径 | Covered |

### 决策依据

D8，不变量 3、4；标记决策 D8。
