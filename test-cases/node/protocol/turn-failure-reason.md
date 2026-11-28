# 一轮失败的原因与失败后的会话结论

依据[一轮的失败与取消分开记录决策](../../../decisions/node/protocol/20261124-turn-failure-reason-distinct-from-cancel.md)
（`approved`）。本文的 15 个核心用例与该决策“验证义务”一节的 15 条一一对应，用例标题前的编号
就是验证义务的编号。决策尚未实现，所有义务都是 `Missing`。该决策对取消决策验证义务 6、18、19、20、30 的修改登记在
[取消一轮用例](cancel-turn-and-keep-session.md)的对应用例中；取消、静止条件与栅栏的既有义务见取消一轮用例与
[轮次进程栅栏用例](turn-process-fence.md)，本文不推导它们的证据。

| 验证义务 | 核心用例 |
| --- | --- |
| 1 | [An ACP error ends the turn as failed with its code and the Agent's text](#an-acp-error-ends-the-turn-as-failed-with-its-code-and-the-agents-text) |
| 2 | [Authentication required is recognised by ACP code or the known texts](#authentication-required-is-recognised-by-acp-code-or-the-known-texts) |
| 3 | [The Agent's text is sanitized and bounded and stays out of codes and logs](#the-agents-text-is-sanitized-and-bounded-and-stays-out-of-codes-and-logs) |
| 4 | [A watchdog stop answered other than cancelled fails as no progress without a resend](#a-watchdog-stop-answered-other-than-cancelled-fails-as-no-progress-without-a-resend) |
| 5 | [A watchdog that runs out of windows fails as no progress after the last resend](#a-watchdog-that-runs-out-of-windows-fails-as-no-progress-after-the-last-resend) |
| 6 | [An unanswered watchdog cancel fails as no progress and ends as turn cancel unanswered](#an-unanswered-watchdog-cancel-fails-as-no-progress-and-ends-as-turn-cancel-unanswered) |
| 7 | [A lost Agent or an Ora-side error mid-turn fails as agent unavailable or internal](#a-lost-agent-or-an-ora-side-error-mid-turn-fails-as-agent-unavailable-or-internal) |
| 8 | [A caller cancel is recorded as caller cancel whatever the Agent answers](#a-caller-cancel-is-recorded-as-caller-cancel-whatever-the-agent-answers) |
| 9 | [A session end during a turn is recorded as session end and never rewrites a failure](#a-session-end-during-a-turn-is-recorded-as-session-end-and-never-rewrites-a-failure) |
| 10 | [Turn end records stay compatible in both directions](#turn-end-records-stay-compatible-in-both-directions) |
| 11 | [A kept failure is settled once as kept before the next turn](#a-kept-failure-is-settled-once-as-kept-before-the-next-turn) |
| 12 | [A failure that cannot keep is settled as session ending with the end decision](#a-failure-that-cannot-keep-is-settled-as-session-ending-with-the-end-decision) |
| 13 | [Failure settlement is negotiated and guards takeover](#failure-settlement-is-negotiated-and-guards-takeover) |
| 14 | [Sessions without failure keeping get the reason but no settlement](#sessions-without-failure-keeping-get-the-reason-but-no-settlement) |
| 15 | [A restart or unwritable ledger before the settlement backfills session ending](#a-restart-or-unwritable-ledger-before-the-settlement-backfills-session-ending) |

## An ACP error ends the turn as failed with its code and the Agent's text

### 风险

Agent 以 ACP 错误回应 `session/prompt`，这一轮仍只记 `TurnEnded{cancelled}`，调用方看不出失败，也看不到 Agent 给出的原因；
具体原因只在 `data` 中而被丢掉；或者错误文字进入了只允许码的字段。

### 前置状态与触发

一轮正在执行，没有取消作用到它；Agent 以带错误码与文字的 ACP 错误回应这次 prompt。分别构造 `data` 缺省、`data` 为字符串、
`data` 为带字符串 `message` 的对象、`data` 为其他形状（数字、数组、没有 `message` 的对象），以及 `data` 中的字符串与 `message`
相同。

### 必须成立

- `TurnEnded` 为 `stop_reason: cancelled`，`failure` 为 `{code: agent_error, acp_code, message}`，`acp_code` 等于 Agent 回应的
  错误码。
- `message` 为清洗后的主文字；`data` 为字符串或 `data.message` 为字符串、且与主文字不同时，以 `: ` 接上这段补充文字；
  其他形状的 `data` 不取；补充文字与主文字相同时不重复。
- Node 把这条记录原样作为 Thread 事件转发，会话 JSONL 中是同一条记录。
- 终态 `detail`（没有选择失败后保留时为 `agent_turn_failed`）与 `TurnFailureSettled` 都不含文字。

### 禁止结果

`TurnEnded` 没有 `failure`；`stop_reason` 不是 `cancelled`；字符串形式的 `data` 被丢掉，或结构化的 `data` 被序列化进文字；
文字出现在 `detail` 或 `TurnFailureSettled` 中；错误码丢失。

### 验证义务与证据

| 义务 | 直接证据与缺口 | 状态 |
|---|---|---|
| 1：ACP 错误回应带码与文字（含 `data` 的补充文字）记为失败 | 尚未实现，无直接证据 | Missing |

### 决策依据

D1、D2、D3、D4，不变量 1、4、5。

## Authentication required is recognised by ACP code or the known texts

### 风险

需要重新认证的失败被记成一般错误，用户不知道要去认证；文字碰巧相似的普通错误被记成需要认证；或者 `failure.code` 与
终态 detail 对同一轮各说一套。

### 前置状态与触发

分别以错误码 `-32000`（文字不是两句已知文字）、以文字 `Authentication required` 和 `Not logged in`（带首尾空白）、以其他
错误码加其他文字回应 prompt；会话没有选择失败后保留。

### 必须成立

- 前三种为 `agent_authentication_required`，带 `acp_code` 与 `message`，会话以 detail `agent_authentication_required` 结束。
- 其他错误码与文字为 `agent_error`，会话以 detail `agent_turn_failed` 结束。

### 禁止结果

`-32000` 被记为 `agent_error`，或 `failure.code` 为 `agent_authentication_required` 而 detail 为 `agent_turn_failed`；只是包含
已知文字、或文字不完全一致的错误被记为需要认证。

### 验证义务与证据

| 义务 | 直接证据与缺口 | 状态 |
|---|---|---|
| 2：认证按错误码或已知文字识别，detail 跟随 | 尚未实现，无直接证据 | Missing |

### 决策依据

D2、D7，不变量 4、11。

## The Agent's text is sanitized and bounded and stays out of codes and logs

### 风险

Agent 的文字带控制字符、双向文字控制符、零宽字符、标签字符或行分隔符，展示时冒充 Ora 的说明或藏入看不见的文字；过长的
文字挤占 Thread 条目；截断产生非法 UTF-8，使读者把整行当作损坏；文字经日志或 Agent 切换的交接扩散到 owner 接受的范围之外。

### 前置状态与触发

Agent 的错误文字分别含：换行、回车与制表符；U+2028、U+2029；其他控制字符；双向文字控制符、零宽字符（U+200B–U+200D）、
U+FEFF 与标签字符（U+E0000–U+E007F）；首尾的非 ASCII 空白（例如 U+3000、U+00A0）；超过 1024 字节、截断点落在多字节字符
中间的内容；清洗后只剩空白的内容。随后发生一次 Agent 切换。

### 必须成立

- 换行、回车、制表符、`Zl`、`Zp` 变为空格；其余 `Cc` 与全部 `Cf` 被删除；按 Unicode `White_Space` 去掉首尾空白。
- 清洗后超过 1024 字节时在码位边界截断，带 `message_truncated: true`；结果是合法 UTF-8。
- 清洗后为空时没有 `message` 字段。
- Node 与运行时的日志只有 `code` 与 `acp_code`；交给下一个 Agent 的记录中没有这段文字。

### 禁止结果

`message` 中留有 `Cc`、`Cf`、`Zl`、`Zp` 字符或首尾空白；超过 1024 字节；截断出非法 UTF-8；空的 `message`；日志或交接中出现
文字。

### 验证义务与证据

| 义务 | 直接证据与缺口 | 状态 |
|---|---|---|
| 3：文字清洗、限长、合法 UTF-8，不进日志与交接 | 尚未实现，无直接证据 | Missing |

### 决策依据

D3，不变量 5。

## A watchdog stop answered other than cancelled fails as no progress without a resend

### 风险

看门狗取消后 Agent 回 `end_turn`，Node 既不重发也不说明原因；或者 Node 改为重发，把一整轮连同工具副作用再执行一遍；或者
`attempts` 把不属于看门狗的发送算进去。

### 前置状态与触发

一轮在第一个窗口内没有有效更新，看门狗发 `session/cancel`；Agent 分别回 `end_turn`、其他非 `cancelled` 的 stop reason
（`max_tokens`、`refusal`）、带错误码与文字的 ACP 错误、错误码 `-32800` 的 ACP 错误。另覆盖第一次重发之后才这样回应的
情况，以及 load 之后第一次发送失败、重建 provider 会话重发之后再被看门狗取消的情况。

### 必须成立

- 不再写出 `session/prompt`。
- `failure` 为 `no_progress`；`attempts` 等于这一轮写出的首次发送与看门狗重发之和（第一个窗口为 1，第一次重发之后为 2），
  load 之后的重建发送不计入；`answer` 为 Agent 回应的 stop reason，ACP 错误（包括 `-32800`）时为 `error` 并带 `acp_code`
  与 `message`。
- Agent 回应了这次 prompt，这一轮按取消决策 D2 归为失败：没有选择失败后保留时以 `agent_turn_failed` 结束会话。

### 禁止结果

重发（包括收到 `-32800` 时）；`TurnEnded` 没有 `failure`；`attempts` 或 `answer` 与实际不符；这一轮被记为成功完成。

### 验证义务与证据

| 义务 | 直接证据与缺口 | 状态 |
|---|---|---|
| 4：看门狗被回非 `cancelled` 时以 `no_progress` 失败、不重发 | 尚未实现，无直接证据 | Missing |

### 决策依据

D5、D2，不变量 6。

## A watchdog that runs out of windows fails as no progress after the last resend

### 风险

重发用尽后的失败与 Agent 提前结束无法区分，排查方向错误；或者重发次数与窗口表不符。

### 前置状态与触发

Agent 对每一次看门狗取消都回 `cancelled`，并且每个窗口内都没有有效更新，直到窗口表用完。

### 必须成立

- 共写出 4 次 `session/prompt`，记录中有 3 条 `PromptRetried`。
- 第 4 个窗口之后不再重发，`failure` 为 `{code: no_progress, attempts: 4, answer: cancelled}`。

### 禁止结果

第 5 次写出；`attempts` 与写出次数不符；`answer` 不是 `cancelled`。

### 验证义务与证据

| 义务 | 直接证据与缺口 | 状态 |
|---|---|---|
| 5：重发用尽后以 `no_progress` 失败 | 尚未实现，无直接证据 | Missing |

### 决策依据

D5，不变量 6。

## An unanswered watchdog cancel fails as no progress and ends as turn cancel unanswered

### 风险

看门狗取消没有得到回应时，会话以 `agent_turn_failed` 结束，Controller 把它读作模型请求失败；或者在 Agent 可能仍在运行时
保留会话。

### 前置状态与触发

看门狗发 `session/cancel` 后，Agent 在宽限期内没有回应，期间没有并入 `CancelTurn`；分别覆盖会话选择与没有选择
`keep_session_on_failure`，以及没有协商 `turn_cancel` 的连接上启动的会话。

### 必须成立

- `failure` 为 `{code: no_progress, answer: none}`。
- 会话以 `agent_failed`、detail `turn_cancel_unanswered` 结束，没有协商 `turn_cancel` 的会话也是如此；终态在停止证据之后写入。
- 选择了失败后保留时，先有 `TurnFailureSettled{session_ending}`，与结束决定同一事务。

### 禁止结果

detail 为 `agent_turn_failed`；会话保留；`answer` 不是 `none`；选择了失败后保留时没有 `TurnFailureSettled` 就写终态。

### 验证义务与证据

| 义务 | 直接证据与缺口 | 状态 |
|---|---|---|
| 6：看门狗取消无回应时的原因与 detail | 尚未实现，无直接证据 | Missing |

### 决策依据

D5、D7、D6，不变量 7、11。

## A lost Agent or an Ora-side error mid-turn fails as agent unavailable or internal

### 风险

一轮进行中 Agent 进程退出，会话以 `agent_turn_failed` 结束，Controller 把它读作模型请求失败；Node 读取的运行时事件流关闭
被误记为 Agent 进程退出；Ora 自己的错误被记成 Agent 的错误；或者选择了失败后保留的会话没有结论。

### 前置状态与触发

一轮正在执行、没有取消写出时：杀掉 Agent 进程；断开 ACP 连接；让这次 prompt 的 ACP 传输出错；看门狗发出取消后、宽限期内
断开 ACP 连接；让 Agent 发出不合法的 ACP 帧或回应；让事件队列溢出。另构造运行时的事件流在终点之前关闭。分别覆盖会话选择
与没有选择 `keep_session_on_failure`。

### 必须成立

- 前四种：`failure` 为 `agent_unavailable`，会话以 `agent_failed`、detail `agent_unavailable` 结束，不论是否选择了失败后保留。
- 不合法的 ACP 帧或回应、事件队列溢出：`failure` 为 `internal`，会话结束。
- 事件流在终点之前关闭：会话以 detail `agent_stream_closed` 结束，没有 `TurnEnded`，也没有 `TurnFailureSettled`。
- 选择了失败后保留时，带 `failure` 的各种情况都先有 `TurnFailureSettled{session_ending}`，与结束决定同一事务。

### 禁止结果

detail 为 `agent_turn_failed`；会话保留；`agent_unavailable` 或 `internal` 带 `message`；看门狗等待期间断开时记为
`no_progress`。

### 验证义务与证据

| 义务 | 直接证据与缺口 | 状态 |
|---|---|---|
| 7：Agent 不在与 Ora 内部错误的原因、detail 与结论 | 尚未实现，无直接证据 | Missing |

### 决策依据

D2、D5、D6、D7，不变量 7、11。

## A caller cancel is recorded as caller cancel whatever the Agent answers

### 风险

Controller 自己的取消被记成失败，页面显示“这一轮失败了”；取消写出之后 Agent 回的错误或 Agent 的消失被当作这一轮的失败
原因；或者已经排在队列里的失败被取消掩盖。

### 前置状态与触发

会话选择了 `keep_session_on_cancel`；分别构造：准入之后、prompt 写出之前受理 `CancelTurn`；取消写出之后 Agent 回
`cancelled`、`end_turn`、ACP 错误；看门狗停止期间并入 `CancelTurn`；截止时刻前没有回应；取消写出之后、Agent 回应之前杀掉
Agent 进程或断开 ACP 连接；取消即将写出时，Agent 的 ACP 错误回应已在队列中。

### 必须成立

- 前六种的 `TurnEnded` 都是 `stop_reason: cancelled`、`ended_by: caller_cancel`，没有 `failure`；取消结论与会话的结局按取消
  决策不变；截止时刻前没有回应、以及取消写出之后 Agent 消失，会话都以 detail `turn_cancel_unanswered` 结束。
- 最后一种记为失败：`failure` 为 `agent_error`，取消得到 `already_ended`（会话保留时）或 `session_ending`。

### 禁止结果

前六种带 `failure`、没有 `ended_by`、或 `ended_by` 为 `session_end`；Agent 消失时 detail 为 `agent_unavailable`；最后一种记为
`caller_cancel`。

### 验证义务与证据

| 义务 | 直接证据与缺口 | 状态 |
|---|---|---|
| 8：调用方取消的归属与先到者为准 | 尚未实现，无直接证据 | Missing |

### 决策依据

D1、D7，不变量 2、11。

## A session end during a turn is recorded as session end and never rewrites a failure

### 风险

会话结束时被取消的一轮被记成失败；失败之后到达的 `EndSession` 抹掉了失败原因；或者 `EndSession` 恰在 D2 判定的末尾到达，
Controller 先看到“会话保留”，再收到 `session_ending` 与终态。

### 前置状态与触发

分别构造：一轮正在执行时受理 `EndSession`；一轮正在执行时 Node 按取消决策 D3 决定结束会话（例如 Thread 镜像不可用）；
会话已开始结束、Agent 随后回 ACP 错误；运行时已观察到 ACP 错误之后、D2 判定期间受理 `EndSession`（会话选择了失败后
保留）；会话选择了失败后保留、这一轮并入了 `CancelTurn`，在 D2 判定最后一次读队列之后、`kept` 写入之前受理 `EndSession`。

### 必须成立

- 前三种：`TurnEnded` 带 `ended_by: session_end`，没有 `failure`。
- 第四种：`TurnEnded.failure` 不变，会话以 `EndSession` 的原因结束，`TurnFailureSettled` 为 `session_ending`，早于终态。
- 第五种：账本拒绝 `kept`，同一事务中的取消结论也不写入；这一轮的 `TurnFailureSettled` 与并入的 `TurnCancelSettled` 都在
  `EndSession` 的结束决定事务中得到 `session_ending`；没有任何一条 `kept` 或 `already_ended`。

### 禁止结果

前三种带 `failure`；第四种的 `failure` 被改写或去掉；第五种出现 `kept`，或 `kept` 与 `session_ending` 同时出现在同一轮上。

### 验证义务与证据

| 义务 | 直接证据与缺口 | 状态 |
|---|---|---|
| 9：会话结束的归属、先到者为准与 `EndSession` 对 `kept` 的竞争 | 尚未实现，无直接证据 | Missing |

### 决策依据

D1、D6，不变量 3、8；[栅栏决策](../../../decisions/node/protocol/20261123-turn-process-fence-before-keeping-session.md) D4（第 7 步读取之后到达的 `EndSession`）。

## Turn end records stay compatible in both directions

### 风险

新字段让旧读者拒绝历史；或者 Agent 自己的 `cancelled`、旧记录被推断成取消或失败；或者没有归属的记录与之前的字节不同。

### 前置状态与触发

Agent 自己以 `cancelled` 结束一轮；读取本决策之前写下的 JSONL；以本决策之前版本的 `ora-history` 与 cloud v2-server 的历史
读取解析带新字段的 JSONL。

### 必须成立

- Agent 自己的 `cancelled` 与旧记录都没有 `ended_by` 与 `failure`。
- 两个字段都省略的新行与本决策之前写出的行逐字节相同；`meta` 行的 `schemaVersion` 不变。
- 旧行能被新读者解码；新行能被旧版 `ora-history` 与 cloud v2-server 的历史读取解码，不被当作损坏。
- 读者把不认识的 `failure.code` 当作 `internal`，忽略不认识的 `failure` 字段。

### 禁止结果

新行被旧读者拒绝；`schemaVersion` 改变；没有归属的记录被标为取消或失败。

### 验证义务与证据

| 义务 | 直接证据与缺口 | 状态 |
|---|---|---|
| 10：记录在新旧读者之间双向兼容 | 尚未实现，无直接证据 | Missing |

### 决策依据

D1、D4，不变量 10。

## A kept failure is settled once as kept before the next turn

### 风险

失败后会话保留了，Controller 却不知道，页面停在“已结束”或一直等待；结论在下一轮之后才到，顺序错乱；或者 `kept` 与并入
取消的结论分两次写入，中间被其他结论插入。

### 前置状态与触发

会话协商了 `turn_cancel` 与 `turn_failure`，选择了 `keep_session_on_failure`，插件版本为本平台登记；一轮以 ACP 错误结束，
另一轮以得到回应的看门狗放弃结束，都满足保留条件；其中一轮在静默窗口内并入了两个 `CancelTurn`。

### 必须成立

- 每一轮恰好一条 `TurnFailureSettled{kept}`。
- `kept` 与并入的各个 `TurnCancelSettled{already_ended}` 在同一个账本事务中写入。
- 执行内的顺序为：这一轮的 Thread 事件 → `TurnEnded{failure}` → `TurnFailureSettled{kept}` → 并入的
  `TurnCancelSettled{already_ended}`（按受理顺序）→ 下一轮的 Thread 事件。
- 自 Agent 回应起 F + Q + R 内落账；同一会话接着执行下一轮。

### 禁止结果

没有结论或有两条；结论晚于下一轮的事件或晚于并入的取消结论；`kept` 与并入的取消分属不同事务；结论不带 `turn_id` 或指向
别的轮次。

### 验证义务与证据

| 义务 | 直接证据与缺口 | 状态 |
|---|---|---|
| 11：保留时的结论、同一事务、顺序与期限 | 尚未实现，无直接证据 | Missing |

### 决策依据

D6，不变量 7、8；[栅栏决策](../../../decisions/node/protocol/20261123-turn-process-fence-before-keeping-session.md) D4。

## A failure that cannot keep is settled as session ending with the end decision

### 风险

失败后会话不能保留，Controller 却没有结论，或结论晚于终态；D2 的 detail 覆盖之后，原失败原因消失；或者 Controller 为一轮
看不到的失败等待结论。

### 前置状态与触发

会话选择了 `keep_session_on_failure`；一轮以 ACP 错误结束后，分别构造：插件版本没有为本平台登记、回应时有未终态的工具调用、
栅栏不干净、静默窗口内有轮次流量、复扫不干净；以及 D3 的情况（账本写入失败、模型授权被撤销）与已受理的 `EndSession`。
另构造两种没有 `TurnEnded` 的失败：重发前写记录失败使会话记录降级、运行时事件流在终点之前关闭。

### 必须成立

- 带 `failure` 的每一种都恰好一条 `TurnFailureSettled{session_ending}`，与结束决定、尚未结算的
  `TurnCancelSettled{session_ending}` 同一事务，早于终态。
- 终态 detail 是 D2 或 D3 的 detail（`turn_boundary_unverified`、`turn_not_quiescent`、`turn_processes_unstopped` 等），
  `EndSession` 时是它的原因；`TurnEnded.failure` 仍是 `agent_error` 与原来的文字。
- `session_ending` 之后不再写出 `session/prompt`。
- 没有 `TurnEnded` 的两种失败没有 `TurnFailureSettled`，会话由终态收尾。

### 禁止结果

带 `failure` 时没有结论；结论晚于终态；结论为 `kept`；`TurnEnded.failure` 被改写；`session_ending` 之后仍有 `session/prompt`；
没有 `TurnEnded` 时出现 `TurnFailureSettled`。

### 验证义务与证据

| 义务 | 直接证据与缺口 | 状态 |
|---|---|---|
| 12：不能保留时的结论与原因保留 | 尚未实现，无直接证据 | Missing |

### 决策依据

D2、D6、D7，不变量 7、8。

## Failure settlement is negotiated and guards takeover

### 风险

不认识 `TurnFailureSettled` 的 Controller 收到它，或接管了选择了失败后保留的会话；旧 Controller 因为新能力握手失败；或者
不对称的确认让两端对协商结果的理解不同。

### 前置状态与触发

分别以只声明 `turn_cancel`、同时声明 `turn_cancel` 与 `turn_failure`、只声明 `turn_failure`、两者都不声明的 `Hello` 连接；
解码只有 `turn_failure: true` 而没有能力 `turn_failure`、或没有 `turn_cancel` 确认的 `HelloAccepted`；在只协商了 `turn_cancel`
的连接上启动 `keep_session_on_failure: true` 的会话；Node 上有选择了失败后保留、尚未终态的会话时，以没有协商
`turn_failure` 的连接接入。

### 必须成立

- 只有同时声明两项时，`HelloAccepted` 才含能力 `turn_failure` 与 `turn_failure: true`；其他情况下握手与本决策之前相同，
  旧 Controller 的解码器接受它。
- 不对称的 `HelloAccepted` 被校验拒绝。
- 未协商 `turn_failure` 的连接上选择失败后保留，连接按协议违规关闭，不进入持久受理。
- 有这类会话时，未协商 `turn_failure` 的连接以 `4403` 被拒绝，不收到任何事件；具备能力的 Controller 随后连接，从最小未确认
  序号起按序收到全部事件。

### 禁止结果

向未同时声明两项的 Controller 返回 `turn_failure`；不对称的确认被接受；未协商的连接能选择失败后保留，或收到
`TurnFailureSettled`；旧 Controller 握手失败。

### 验证义务与证据

| 义务 | 直接证据与缺口 | 状态 |
|---|---|---|
| 13：协商、对称校验、选择前提与接管边界 | 尚未实现，无直接证据 | Missing |

### 决策依据

D6，不变量 9；[取消决策](../../../decisions/node/protocol/20261116-cancel-turn-and-keep-session-after-turn-ends.md) D5。

## Sessions without failure keeping get the reason but no settlement

### 风险

没有选择失败后保留的会话行为被改变，依赖“失败即终态”的 Controller 收到不认识的事件；这些会话拿不到失败原因；或者 detail
的变化超出 D7 的三处。

### 前置状态与触发

会话只选择了 `keep_session_on_cancel`，或两个字段都没有选择（包括没有协商 `turn_cancel` 的连接上启动的会话）；一轮分别以
ACP 错误、错误码 `-32000` 的 ACP 错误、得到回应的看门狗放弃、看门狗取消无回应、Agent 进程退出结束。

### 必须成立

- 没有 `TurnFailureSettled`。
- 会话以 `agent_failed` 结束：ACP 错误与得到回应的看门狗放弃为 `agent_turn_failed`；`-32000` 为
  `agent_authentication_required`；看门狗取消无回应为 `turn_cancel_unanswered`；Agent 进程退出为 `agent_unavailable`。
- `TurnEnded` 带 `failure`，与选择了失败后保留时相同。

### 禁止结果

产生 `TurnFailureSettled`；会话保留；detail 出现 D7 三处以外的变化；`TurnEnded` 没有 `failure`。

### 验证义务与证据

| 义务 | 直接证据与缺口 | 状态 |
|---|---|---|
| 14：未选择失败后保留的会话 | 尚未实现，无直接证据 | Missing |

### 决策依据

D6、D7，不变量 11；[取消决策](../../../decisions/node/protocol/20261116-cancel-turn-and-keep-session-after-turn-ends.md)不变量 11。

## A restart or unwritable ledger before the settlement backfills session ending

### 风险

Node 在失败的 `TurnEnded` 之后、结论之前重启，Controller 一直等不到结论，或重启后补出与事实不符的结论；账本不可写时把内存
状态当作结论发送；补写重复。

### 前置状态与触发

会话选择了失败后保留；一轮以 ACP 错误结束、`TurnEnded` 已作为 Thread 事件持久化、D2 判定还没有结束时：杀掉并重启 Node；
另一次注入账本持续写失败，之后恢复；再一次在会话 JSONL 已写、Thread 事件还没有持久化时杀掉 Node。

### 必须成立

- 重启：重启收口读这一轮的待结算标记，在 `interrupted` 结束决定的同一事务中补写恰好一条
  `TurnFailureSettled{session_ending}`，早于终态；终态在取得停止证据之后写入。
- Controller 已经 ACK 了失败的 `TurnEnded` 之后再重启或结束，仍然补写这一条结论。
- 账本不可写：不发送未落账的 `TurnFailureSettled`，不再写出 `session/prompt`；恢复后，在写结束决定的同一事务中补写
  `TurnFailureSettled{session_ending}`，早于终态。
- 已经有结论的轮次不再补写。
- Thread 事件没有持久化时，Controller 没有看到这一轮的 `TurnEnded`，也没有 `TurnFailureSettled`。

### 禁止结果

重启后出现 `TurnFailureSettled{kept}`，或没有任何结论就写终态；没有落账的结论被发送；同一轮有两条结论；ACK 之后的结束或重启不补写。

### 验证义务与证据

| 义务 | 直接证据与缺口 | 状态 |
|---|---|---|
| 15：重启与账本恢复后的补写 | 尚未实现，无直接证据 | Missing |

### 决策依据

D6，不变量 7；[取消决策](../../../decisions/node/protocol/20261116-cancel-turn-and-keep-session-after-turn-ends.md) D3（重启沿用结束决定）、D4（账本持续不可写）。
