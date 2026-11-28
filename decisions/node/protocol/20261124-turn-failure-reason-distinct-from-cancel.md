---
status: approved
---

# 一轮的失败与取消分开记录：`TurnEnded` 带归属与失败原因，失败后会话是否保留由 `TurnFailureSettled` 告知；看门狗被回 `end_turn` 时以无进展失败

共享运行时在每一条 `TurnEnded` 里写明这一轮为什么没有成功：调用方取消（`ended_by: caller_cancel`）、会话结束时被
取消（`ended_by: session_end`），或者失败（`failure`）。失败带有界的原因码；原因来自 Agent 的 ACP 错误回应时，还带
ACP 错误码与 Agent 报告的文字（JSON-RPC 的 `message`，以及字符串形式的 `data` 补充文字，清洗后合计至多 1 KiB）。
`stop_reason` 仍是 `cancelled`，依赖它的读者不变。选择了 `keep_session_on_failure` 的会话，每个以失败结束的轮次在终态
之前恰好得到一条新的非终态事件 `TurnFailureSettled{turn_id, outcome}`，`outcome` 为 `kept`（会话保留，可以接着执行
下一轮）或 `session_ending`（会话不会保留）。这条事件经新能力 `turn_failure` 协商，只有协商了它的连接才能为会话选择
`keep_session_on_failure`。无进展看门狗发出 `session/cancel` 后，Agent 的回应不是 `cancelled` 时（例如 `end_turn`）不重发，
这一轮以 `no_progress` 失败，记录发送次数与 Agent 的回应。终态 detail 有三处变化，对所有会话生效：看门狗取消没有得到
回应时为 `turn_cancel_unanswered`，一轮进行中 Agent 进程退出或 ACP 连接断开时为 `agent_unavailable`（这两处使实现与
取消决策 D3 一致），Agent 以 ACP 错误码 `-32000` 回应时为 `agent_authentication_required`（detail 跟随 `failure.code`）。

本 ADR 状态为 `approved`：2026-11-24 经 owner 评审批准（[specs#335](https://github.com/ora-space/specs/pull/335)），依据 [circus#1138](https://github.com/ora-space/circus/issues/1138) 的 owner 结论（2026-11-24）：
Agent 的错误文字以有界文字单独携带，owner 接受它随会话 JSONL 进入 Revision 归档；新增经协商的 `TurnFailureSettled`；
看门狗被回 `end_turn` 时以 `no_progress` 失败、不重发；两处 detail 按取消决策 D3 改代码；`ended_by` 本版只有两个值。
评审中补充的规则（取 `data` 中的补充文字、清洗范围、`-32000` 的 detail、取消写出后 Agent 消失的归属、`kept` 与并入取消
同一事务、重启后补写结论、`turn_cancel_unanswered` 对所有会话生效）由 owner 在评审中确认。
它是[保留会话前先停掉这一轮新起的进程](20261123-turn-process-fence-before-keeping-session.md)（下称“栅栏决策”，
`implemented`）的后续，修改
[取消一轮并在轮次结束后保留会话](20261116-cancel-turn-and-keep-session-after-turn-ends.md)（下称“取消决策”，`implemented`）
D2 的记录与末段、D3 结束决定事务的内容、D4 的事件顺序、D5 的协商、`Hello.turn_cancel` 的含义、“不会产生新的 detail”
一条与启用顺序、D6，以及不变量 11、风险表最后一行、验证义务 6、18、19、20、30 与“落地顺序”第 4 步；修改栅栏决策 D4 的
第 1、8 步与不变量 6；修改[agent-runtime 根决策](../agent-runtime/0-shared-agent-runtime-crate-hosted-by-node.md)
（`implemented`）D6 中“按运行时既有规则记录 `TurnEnded`”与兼容性边界中“会话 JSONL 格式不变”两处。

涉及 desktop 的 `crates/acp`（ACP 错误回应保留错误码与 `data`）、`crates/history`（`TurnEnded` 的新字段）、
`crates/agent-runtime`（一轮结束的归属、失败原因与文字清洗）、`crates/node-protocol`（能力、握手字段、事件与校验）、
`crates/node-db`（持久化新事件）与 `apps/ora-node/src/session/`（失败的结算、重启补写与 detail 变化）。兼容性边界：会话
JSONL 只在既有的 `turnEnded` 行上增加可选字段，`SCHEMA_VERSION` 与本地文件布局不变，旧记录照常读取；没有协商
`turn_failure` 的连接收到的握手与事件不变；两个保留字段都没有选择、或只选择了 `keep_session_on_cancel` 的会话，除 D7 的
三处 detail 变化外行为不变。D7 的变化也到达不声明 `turn_cancel` 的 Controller，其中 `turn_cancel_unanswered` 是它们以前收不到
的值，展示上的代价与补救列在 D7 与“落地顺序”中。本决策不规定 Controller 如何展示失败、如何与 Controller 自己知道的原因
合并（cloud v2-server 的规则待补入模型代理决策），不改变 Agent 切换时的记录交接，不改变 MCP 门禁中的
`SessionTurnFailed`，也不启用失败后保留：启用仍是取消决策“落地顺序”第 4 步，前置条件见本决策 D8。

放在 `protocol` 叶子目录，因为它承接取消决策 D6 明确留下的边界，新增的是 Node 协议的能力与事件；`TurnEnded` 的形状属于
`ora-history`，Node 协议把它作为不透明的 Thread 记录转发，所以记录形状在本决策中规定，并在 agent-runtime 根决策下补
`> 历史决策：` 指针，不另立 agent-runtime 决策。

## 继承与修改

承接本目录前序的栅栏决策，并经它承接取消决策。三份被修改的前序 ADR 标题下补 `> 历史决策：` 指针，其余正文不改。
会话执行、命令与事件序号的规则经取消决策继承自
[会话执行与命令](20260928-streamed-thread-events-and-session-commands.md)，结束会话的停止责任来自
[会话 MCP 刷新与宿主健康](../agent-runtime/20261116-session-mcp-refresh-and-host-health.md) D5，均不变。

| 前序约定 | 处理 |
| --- | --- |
| 取消决策 D1：`CancelTurn`、`TurnCancelSettled` 的五种结论、并入与截止时刻；“停止中”包括“这一轮已失败”；失败而停时并入的取消为 `already_ended` | 继承，不变 |
| 取消决策 D2 结束方式表：四种结束的“归类”与“要求的选择” | 继承，不变；“记录”一列由一律 `TurnEnded{cancelled}` 改为：取消两行带 `ended_by: caller_cancel`，失败两行带 `failure`（D1、D4） |
| 取消决策 D2 末段：“失败的一轮与取消的一轮在 Thread 里都记为 `TurnEnded{cancelled}`，本决策不区分两者” | 修改：`stop_reason` 仍为 `cancelled`，以 `ended_by` 与 `failure` 区分（D1） |
| 取消决策 D2 静止条件、轮次边界、跨轮次检测、续用方式；栅栏决策 D1–D3、D5–D8 | 继承，不变 |
| 取消决策 D3 情况表、结束会话的路径、重启沿用结束决定 | 继承；实现中两处 detail 与表不一致，随本决策改为与表一致（D7） |
| 取消决策 D3 结束决定事务的内容：首次终态原因与 detail、停止时钟、未结算取消的 `session_ending` | 修改：选择了失败后保留、失败的一轮尚未结算时，同一事务另写 `TurnFailureSettled{session_ending}`；重启收口写 `interrupted` 结束决定时同样补写（D6） |
| 取消决策 D4 事件顺序：Thread 事件 → `TurnEnded` → 并入的 `TurnCancelSettled` → 下一轮或终态 | 修改：选择了失败后保留的会话，失败的一轮在 `TurnEnded` 之后、并入的 `TurnCancelSettled` 之前有一条 `TurnFailureSettled`（D6） |
| 取消决策 D4：`EndSession` 优先、先到者为准、本地结算、账本不可写、投递与 Controller 的等待 | 继承；`TurnFailureSettled` 沿用同一规则，账本在已受理 `EndSession` 后同样拒绝 `kept`（D6） |
| 取消决策 D5：能力 `turn_cancel`、`Hello`／`HelloAccepted` 的同名字段、两个逐会话选择、接管边界、部署前提与回退条件 | 修改：新增能力 `turn_failure` 与同名字段；`keep_session_on_failure: true` 另要求当前连接协商了 `turn_failure`；选择了它的会话，接管边界同样要求 `turn_failure`（D6） |
| 取消决策 D5：`Hello.turn_cancel` 声明 Controller 理解“新的拒绝原因与终态 detail”；“两个字段都为 `false` 的会话，不会产生 `TurnCancelSettled` 或新的 detail” | 修改：`turn_cancel_unanswered` 不再只出现在协商了 `turn_cancel` 的连接上，看门狗取消没有得到回应时对所有会话都是这个 detail；`-32000` 的 `agent_authentication_required` 同样对所有会话生效。两个字段都为 `false` 的会话仍不产生 `TurnCancelSettled`（D7） |
| 取消决策 D5 启用顺序：失败后保留等失败原因上报的后续决策落地 | 修改：前置条件具体为本决策落地、Controller 协商 `turn_failure` 并能展示失败与原因，以及取消决策风险表中 OpenCode 内部异常一行有结论（D8） |
| 取消决策 D6：与失败原因上报的边界 | 由本决策承担 |
| 取消决策不变量 11：两个字段都没有选择的会话行为完全不变；只选择 `keep_session_on_cancel` 的会话，一轮失败时的行为不变 | 修改：除 D7 的三处 detail 变化外不变；`TurnEnded` 另带 `failure`，它是记录字段，不是新事件 |
| 取消决策风险表“一轮失败后保留会话，调用方没有失败信号” | 由本决策处理（D1–D3、D6） |
| 取消决策验证义务 6、18、19、20、30 | 修改：各增加本决策的义务，在取消一轮用例中以“（失败原因）”行登记，不另占编号（“验证义务”） |
| 栅栏决策 D4 第 1 步：写 `TurnEnded{cancelled}` | 修改：`TurnEnded` 带 D1 的归属或失败 |
| 栅栏决策 D4 第 8 步：按受理顺序逐条写并入的 `TurnCancelSettled`，会话保留 | 修改：选择了失败后保留、因失败而停的一轮，`TurnFailureSettled{kept}` 与并入的各个 `TurnCancelSettled{already_ended}` 在同一事务中写入，失败路径上不再有逐条写入的窗口（D6） |
| 栅栏决策不变量 6：结论落账期限 | 修改：同一期限也约束 `TurnFailureSettled`（D6） |
| agent-runtime 根决策 D6：“Agent 轮次失败、超时、连接丢失：按运行时既有规则记录 `TurnEnded`” | 修改：`TurnEnded` 带 D1 的归属或失败与原因（D1–D4） |
| agent-runtime 根决策兼容性边界：“会话 JSONL 格式（`ora-history` 所有）不变” | 修改：`turnEnded` 行增加可选字段，`SCHEMA_VERSION` 与文件布局不变，旧行可读，新行对忽略未知字段的旧读者透明（D4） |
| agent-runtime 运行时的无进展看门狗：Agent 确认 `cancelled` 才重发，窗口 45、60、90、120 秒；取消决策 D2 表第 4 行 | 继承，不变；本决策规定放弃时记录的原因（D5） |
| [Thread 根决策](../../cloud/thread/0-durable-agent-thread-with-user-turns.md) D2：Cloud 不解析记录的业务字段，只看 `TurnEnded.stopReason`，`kind` 在已知集合内 | 继承，不变；新字段对 Cloud 透明，不新增记录类型。Thread 根决策原文写作 `stopReason`，线上记录的字段名一直是 `stop_reason`（D4），本决策不改变它 |
| 会话 MCP 刷新 D4：MCP 门禁中的轮次以 `SessionTurnFailed{mcp_refresh_wait_timeout}` 失败，没有 `TurnEnded` | 继承，不变；这类失败不带 `failure`，也没有 `TurnFailureSettled` |
| 会话 MCP 刷新 D3：新消息要求 Controller 与 Node 同版本发布，不对旧 Rust 枚举解码器透明 | 继承；只向声明了 `turn_failure` 的 Controller 返回这项能力（D6） |

## 问题与约束

按 desktop main [`7a68e2db`](https://github.com/ora-space/desktop/commit/7a68e2dbfec32a99fa70063d7f51b522fef452be) 核对，
一轮没有成功结束时：

- **记录分不出取消与失败**：`ora-history` 的 `HistoryRecord::TurnEnded` 只有 `stop_reason`
  （`crates/history/src/record.rs`）。用户取消、`EndSession`、ACP 报错、看门狗放弃、Agent 进程退出，运行时都写
  `TurnEnded{stop_reason: cancelled}`（`crates/agent-runtime/src/actor_prompt.rs`、`turn_cancel.rs`、`prompt_retry.rs`）。
- **原因在第一层就丢了**：`ora-acp` 把 ACP 错误回应变成 `AcpError::RequestFailed(String)`，只留 `message`，丢掉 `code` 与
  `data`（`crates/acp/src/client.rs`）。运行时只把与 `Authentication required`、`Not logged in` 完全一致的文字归为需要认证，
  其余错误一律成为内部错误，文字只留在错误的 source 链里（`crates/agent-runtime/src/support.rs` 的 `map_prompt_error`）。
- **会话终态的 detail 分不出原因**：ACP 报错与看门狗放弃都以 `agent_turn_failed` 结束会话（`apps/ora-node/src/session/driver/conversation.rs`）。
  另外两处与取消决策 D3 不一致：看门狗取消没有得到回应时也是 `agent_turn_failed`（只有并入了 `CancelTurn` 时才是
  `turn_cancel_unanswered`）；一轮进行中 Agent 进程退出、ACP 连接断开时，运行时报告的是“运行时不可用”，同样落到
  `agent_turn_failed`，而不是 D3 表中的 `agent_unavailable`。
- **保留会话的失败没有任何信号**：选择了 `keep_session_on_failure` 的会话，失败后 D2 判定保留时 Node 不发任何事件；
  判定不保留时，原失败码还会被 D2 的 detail（`turn_boundary_unverified`、`turn_not_quiescent`、`turn_processes_unstopped`）取代。
- **看门狗被回 `end_turn` 时不说明原因**：看门狗取消后只有 Agent 回 `cancelled` 才重发；回 `end_turn` 或错误时以
  “agent prompt made no progress” 失败（`crates/agent-runtime/src/prompt_retry.rs`），这些事实只进日志。

所以取消决策 D6 指出的缺口成立：选择了失败后保留的会话，调用方看不出这一轮失败，也拿不到“需要重新认证”。约束：

- **`stop_reason` 的含义不能变**：Cloud 的 Thread 根决策 D2 只以 `stopReason`（线上字段 `stop_reason`）判断空闲证据，
  [Workflow 会话结束决策](../../cloud/workflow/20261019-workflow-session-end-after-initial-turn.md)以成功的 `turnEnded` 为结束信号，
  [交互节点完成决策](../../desktop/core/workflow/20261021-interactive-node-completion.md)以 `TurnEnded` 划分本轮；这些读者都依赖
  “失败与取消都是 `cancelled`、都不是成功”。它也是 ACP 的封闭枚举，Ora 不能往里加值。
- **不能新增记录类型**：Cloud 只接受已知 `kind`；cloud v2-server 读 Node 本地 JSONL 时对记录类型有白名单，未知类型按
  损坏处理，但会忽略已知类型上多出来的字段。
- **Agent 的文字不可信**：它由第三方 Agent 或上游模型服务写出，可能很长、含控制字符、格式字符或双向文字控制符，也可能
  带上内部地址或凭据片段。终态 `detail` 与 Controller 的失败码只能是有界码（`crates/node-protocol` 的 `AgentSessionEnded`）。
- **Controller 需要确定的结论**：会话保留与否由 Node 在 D2 判定之后才知道，最长要等栅栏决策的 F + Q + R；没有结论事件，
  Controller 只能猜。
- **旧解码器是封闭的**：`NodeCapability`、`SessionTurnFailureCode` 是封闭枚举；cloud v2-server 对 `session_turn_failed`
  只接受 `mcp_refresh_wait_timeout`，其他码按协议违规结束会话。
- **本地文件布局是硬兼容边界**：JSONL 的路径与布局不变，旧文件不迁移。Node 账本因本决策升到 schema v17，新增一张表，
  记录选择了失败后保留的会话中每个失败轮的待结算标记与结论（见 D6）；打开过一次的账本即为 v17，只认 v16 的 Node 会拒绝它，
  这是不可回退的边界。

## 决策

### D1：一轮没有成功结束时的归属

运行时为每个被准入、没有成功结束的轮次，在 `TurnEnded` 中记录它的归属，三者互斥：

| 归属 | 什么时候 | 记录 |
| --- | --- | --- |
| 调用方取消 | `CancelTurn` 作用到这次 prompt：prompt 写出之前被拦下；`session/cancel` 已写出（之后的任何回应，包括 `end_turn` 与 ACP 错误，都算这次取消的回应；之后 Agent 进程退出或 ACP 连接断开也仍是这次取消）；看门狗停止期间并入；截止时刻前没有回应。也包括不经 Node 的本机停止 | `ended_by: caller_cancel` |
| 会话结束 | 这一轮正在执行时会话开始结束：`EndSession`，或 Node 按取消决策 D3 决定结束会话而停止 Agent；宿主停止读取这一轮的事件（只在宿主结束会话或自身退出时发生） | `ended_by: session_end` |
| 失败 | 运行时先观察到失败：ACP 错误回应、看门狗放弃、Agent 进程退出或 ACP 连接断开、Ora 自己的内部错误 | `failure`（D2–D4） |

`stop_reason` 在三种情况下都是 `cancelled`。Agent 自己以 `cancelled` 或其他 stop reason 结束的一轮（取消没有作用到它）
保留 Agent 的 stop reason，不带 `ended_by` 和 `failure`；本决策之前写下的记录同样没有这两个字段。两者在读者看来都是
“没有归属”，不得被推断为取消或失败中的任何一种。

**先观察到的事实为准**：归属由运行时在写 `TurnEnded` 时根据它先观察到的事实决定，之后不改写：

- 失败的回应已在队列中、取消还没有写出：失败（取消决策差异表中的“先到者为准”，不变）；
- 取消已写出之后 Agent 回 ACP 错误：调用方取消；
- 取消已写出之后、Agent 回应之前，Agent 进程退出或 ACP 连接断开：调用方取消，不写 `failure`；这次取消没有得到回应，
  会话以 `turn_cancel_unanswered` 结束（与现在的实现相同，D7）；
- 失败已经观察到之后才受理的 `CancelTurn`、`EndSession`：仍是失败。`CancelTurn` 得到 `already_ended` 或 `session_ending`
  （取消决策 D1），会话以 `EndSession` 的原因结束（取消决策 D4），`TurnEnded.failure` 不变；
- 会话已开始结束之后 Agent 才回错误：会话结束。

为什么由运行时判定：判定所需的事实（取消是否已写出、回应是否已在队列中、看门狗是否在停止中）只有运行时同时看到，
它也是写 `TurnEnded` 的一方。让 Node 事后从错误推断，就会重演今天“两条路径各判一次”的不一致。为什么 `ended_by` 只有
两个值（owner 裁定）：Controller 需要的只是“是不是自己的取消”，会话为什么结束已经由终态的原因与 detail 说明；更细的
来源（停止按钮还是 `CancelTurn`，`EndSession` 的哪个原因）以后有消费者时再加。为什么取消写出之后 Agent 消失仍记为调用方
取消：取消一旦写出，这一轮怎样结束都由这次取消决定，与“之后的任何回应都算取消的回应”是同一条规则；`ended_by` 回答的
是“是不是自己的取消”，Agent 是否还在由终态 detail 说明。detail 取 `turn_cancel_unanswered` 而不是 `agent_unavailable`：
运行时等待取消回应时，连接关闭与宽限期到期是同一个结果（没有回应，`crates/agent-runtime/src/turn_cancel.rs` 的
`settle_prompt_cancel`），保持现有行为就不需要在这条路径上区分两者。为什么“没有归属”不推断：旧记录里的
`cancelled` 既可能是取消也可能是失败，任何推断都会把一部分记错。

### D2：失败原因码

`failure.code` 取下表之一。它是有界的 snake_case 码，属于这一轮，不同于会话终态的 detail（D7）：

| `code` | 什么时候 | 附带字段 |
| --- | --- | --- |
| `agent_authentication_required` | Agent 以 ACP 错误回应 `session/prompt`，错误码为 `-32000`（ACP 的 `AuthRequired`），或文字去掉首尾空白后恰为 `Authentication required` 或 `Not logged in` | `acp_code`、`message` |
| `agent_error` | Agent 以其他 ACP 错误回应 `session/prompt`：模型请求失败、限流、key 失效或额度用完、Agent 内部错误 | `acp_code`、`message` |
| `no_progress` | 无进展看门狗放弃这一轮（D5） | `attempts`、`answer`；`answer` 为 `error` 时另有 `acp_code`、`message` |
| `agent_unavailable` | 一轮进行中（取消没有写出时）Agent 进程退出或崩溃、ACP 连接断开，或这次 prompt 的 ACP 传输出错 | 无 |
| `internal` | Ora 一侧的错误：Agent 违反 ACP 帧或回应格式、事件队列溢出、权限请求无法应答 | 无 |

- **ACP 错误码**：`ora-acp` 保留 ACP 错误回应的 `code` 与 `data`，运行时把 `code` 原样写入 `acp_code`（JSON-RPC 的整数错误码）。
- **认证判定比今天多认一种**：今天只认两句文字；本决策另认错误码 `-32000`，文字匹配保留给不使用该错误码的 Agent。
  终态 detail 跟随这个判定（D7）。
- **是否可以保留会话**不由码决定，仍按取消决策 D2：Agent 回应了这次 prompt（ACP 错误回应，或看门狗取消得到回应）的
  失败才可能保留；`agent_unavailable`、`internal`，以及 `answer` 为 `none` 的 `no_progress`，按取消决策 D3 结束会话。
- **没有失败码的失败**：运行时没能写出 `TurnEnded` 时（会话记录已降级，包括重发前写记录失败；或运行时的事件流在终点之前
  关闭），这一轮没有 `failure`，也没有 `TurnFailureSettled`（D6），会话按取消决策 D3 结束，原因只在终态 detail 中。
- **MCP 门禁中的失败**不属于本表：那一轮没有被准入，也没有 `TurnEnded`，仍是 `SessionTurnFailed`（会话 MCP 刷新 D4）。

为什么码集合是封闭的：Controller 要为每个码写展示，开放的码等于让它展示未知内容。新增码要随新的后续决策，旧
Controller 把未知码当作 `internal` 展示。为什么认证优先看错误码：ACP 为需要认证规定了错误码，按文字匹配只认得两句
英文，也无法区分“文字碰巧相同”。为什么 `agent_unavailable` 与终态 detail 同名：它们说的是同一个事实（Agent 不在了），
同名让读者不必维护两张对照表。

### D3：Agent 报告的文字

`failure.message` 是 Agent 在 ACP 错误回应中写的文字，只在原因来自 ACP 错误回应时出现。它由 Agent 报告，不是 Ora 的
文字：

1. **来源**：主文字是 JSON-RPC 的 `message`。`data` 是字符串、或 `data` 是对象且其 `message` 是字符串时，这段字符串是
   补充文字；补充文字不为空且与主文字不同时，接在主文字之后，以 `: ` 分隔。`data` 的其他形状（数字、数组、其他对象字段）
   不取。
2. **清洗**：换行、回车、制表符以及行分隔符与段分隔符（`Zl`、`Zp`）换成空格；删除其余 Unicode 控制字符（`Cc`）与全部格式
   字符（`Cf`，包括双向文字控制符、零宽字符、U+FEFF 与标签字符）；按 Unicode `White_Space` 属性去掉首尾空白。清洗后为空
   则省略这个字段。
3. **限长**：清洗后按 UTF-8 编码至多 1024 字节，超出时在码位边界截断，并写 `message_truncated: true`。补充文字在后，所以
   先被截掉。结果总是合法的 UTF-8。
4. **只在这个字段里**：终态 `detail`、`TurnFailureSettled` 与 Controller 的失败码都不含它；Node 与运行时的日志只记 `code`
   与 `acp_code`，不记文字；Agent 切换时的记录交接不把它交给下一个 Agent。
5. **读者的义务**：它只能作为“Agent 报告”原样展示，不能当作 Ora 自己的说明，不能翻译成固定文案，不能从中解析出码或
   据此改变行为；展示时要与 Ora 自己的失败说明分开。

文字随 `TurnEnded` 写进会话 JSONL，所以出现在 Thread 条目与 Revision 归档中，owner 已接受这一点。

为什么携带文字（owner 裁定）：只有 Agent 知道具体原因（额度用完、模型不存在、上游拒绝），分类码说不出这些。为什么取
`data` 中的字符串：ACP 要求 `message` 是简短的一句话，细节放在可选的 `data` 中；常见的辅助函数会把 `message` 固定为
“Internal error” 这类通用文字、把具体原因放进 `data`，只取 `message` 就会让携带文字的目的落空。只取字符串，是因为
结构化的 `data` 没有约定的形状，拼进文字既不可读也无从清洗。为什么 1 KiB：常见的模型错误一两行，1 KiB 足够；Thread
条目上限为 256 KiB，文字不会挤掉记录。为什么删除全部格式字符、把 `Zl`、`Zp` 当换行：双向文字控制符能让展示的顺序与
实际内容不同，零宽字符与标签字符能藏入看不见的文字，行分隔符在不少渲染器中会换行；它们都能让 Agent 的文字冒充 Ora 的
说明。代价是删除零宽连接符会拆开组合表情，Agent 的错误文字里很少出现。为什么不进日志与交接：日志的保存与传播范围不在
owner 接受的范围内；把上一个 Agent 写的文字交给下一个 Agent 作为上下文，等于让第三方文字进入提示词。

### D4：`TurnEnded` 的记录形状

`ora-history` 的 `TurnEnded` 增加两个可选字段，与既有的 `stop_reason` 同为 snake_case：

```json
{"at":"…","seq":42,"type":"turnEnded","stop_reason":"cancelled","ended_by":"caller_cancel"}
{"at":"…","seq":42,"type":"turnEnded","stop_reason":"cancelled",
 "failure":{"code":"agent_error","acp_code":-32603,"message":"Internal error: Rate limit reached for requests"}}
{"at":"…","seq":42,"type":"turnEnded","stop_reason":"cancelled",
 "failure":{"code":"no_progress","attempts":1,"answer":"end_turn"}}
```

示例只用于固定语义。字段规则：

- `ended_by`：`caller_cancel` 或 `session_end`；与 `failure` 互斥；只与 `stop_reason: cancelled` 一起出现。
- `failure`：`code` 必有；`acp_code`、`message`、`message_truncated`、`attempts`、`answer` 按 D2、D3、D5 出现；只与
  `stop_reason: cancelled` 一起出现。
- 两个字段都省略时，记录与本决策之前逐字节相同。`SCHEMA_VERSION` 不变，文件布局不变，旧文件不迁移。
- 读者必须忽略 `failure` 中不认识的字段，把不认识的 `code` 当作 `internal`。

为什么用 snake_case：同一行的 `stop_reason` 已经是 snake_case，消费者按这个形状读它；同一行混用两种写法只会多出拼错的
机会。为什么不升 `SCHEMA_VERSION`：只是在已知行上加可选字段，desktop 与 Node 的旧读者都忽略未知字段，cloud v2-server
也一样；反过来，cloud v2-server 只接受 `schemaVersion` 为 1 或 2 的文件，升版本会让它拒绝整个新文件。

### D5：无进展看门狗放弃时的原因

看门狗的计时、窗口与重发规则不变：一个窗口内没有有效更新时发 `session/cancel`，只有 Agent 回 `cancelled` 且还有下一个
窗口时，才在同一个 provider 会话上重发。看门狗放弃这一轮时，`failure` 为 `no_progress`：

- `attempts`：这一轮写给 Agent 的 `session/prompt` 次数，即首次发送加看门狗重发；load 之后为恢复失效会话而重建的那一次
  发送不计入。
- `answer`：Agent 对最后一次看门狗取消的回应：ACP stop reason（`cancelled` 表示重发次数已用尽，`end_turn`、`max_tokens`、
  `max_turn_requests`、`refusal`），`error`（ACP 错误回应，另带 `acp_code` 与 `message`），或 `none`（看门狗宽限期内没有
  回应）。等待期间 ACP 连接关闭时不是 `no_progress`，而是 `agent_unavailable`（D2）。

Agent 回应的不是 `cancelled` 时**不重发**（owner 裁定）。以 ACP 错误回应看门狗的取消，包括错误码 `-32800`（ACP 的
“请求已取消”），同样记为 `answer: error`、不重发，与现在的实现相同。`answer` 不是 `none` 时这一轮按取消决策 D2 表第 4 行
归为失败，选择了失败后保留的会话按 D2 判定；`answer` 为 `none` 时按 D7 结束会话。看门狗停止期间并入了 `CancelTurn` 的
一轮仍按调用方取消处理（取消决策 D1），不写 `failure`。

为什么不重发：ACP 要求被取消的 prompt 回 `cancelled`；回 `end_turn` 说明 Agent 认为这一轮已在边界上结束，重发可能把
一整轮连同工具的副作用重复一遍。这种回应常见于 Agent 自己正在退避重发模型请求时，Node 再重发等于叠加两层重发，与
模型代理决策 D4 限制重发的方向相反。为什么 `-32800` 也不重发：它在语义上是“已取消”，可以当作确认；但 ACP 只为
`session/prompt` 的取消规定了 `cancelled` 这个 stop reason，以错误回应时 Agent 是否已收尾、provider 会话能否接着收 prompt
没有约定，把它当作确认要先在部署的 Agent 上实测（“已知风险与待实测”）。为什么记 `attempts` 与 `answer`：它们区分
“窗口用尽”与“Agent 提前结束”，前者指向模型长时间没有输出，后者指向 Agent 的行为，展示与排查的方向不同。

### D6：`TurnFailureSettled` 与能力 `turn_failure`

**协商**：Controller 在 `Hello` 中以 `turn_failure: true` 声明它理解 `TurnFailureSettled`；Node 支持、且这次 `Hello` 同时
声明了 `turn_cancel` 与 `turn_failure` 时，才在 `HelloAccepted.capabilities` 中声明 `turn_failure`，并回
`HelloAccepted.turn_failure: true`，与 `turn_cancel` 的协商形式相同。否则握手内容与本决策之前相同。`HelloAccepted` 的校验
对称：`turn_failure: true` 必须与能力 `turn_failure` 同时出现，且同时有 `turn_cancel` 的确认与能力，否则是协议错误，与
现有的 `turn_cancel` 协商校验（`crates/node-protocol/src/message/session.rs`）对应。

**选择**：`keep_session_on_failure: true` 只能在协商了 `turn_cancel` 与 `turn_failure` 的连接上设置；未协商时 Node 按协议
违规关闭连接，不进入持久受理，与取消决策差异表中“D5 未协商的连接”一行的处理相同。接管边界随之扩展：Node 上还有选择了
`keep_session_on_failure`、尚未终态或仍有未确认事件的会话执行时，没有协商 `turn_failure` 的连接以 `4403` 被拒绝。部署
前提与回退条件沿用取消决策 D5，能力换成 `turn_cancel` 与 `turn_failure` 两项。

**事件**：选择了 `keep_session_on_failure` 的会话，每条带 `failure`、已作为 Thread 事件持久化的 `TurnEnded`，在终态之前恰好
得到一条非终态事件 `TurnFailureSettled{turn_id, outcome}`：

| `outcome` | 什么时候写入 |
| --- | --- |
| `kept` | D2 判定会话保留：栅栏决策 D4 第 8 步，与并入这次停止的各个 `TurnCancelSettled{already_ended}` 在同一事务中写入，先于下一轮的准入 |
| `session_ending` | 会话不会保留：D2 判定不通过、属于取消决策 D3 的情况、或已受理的 `EndSession` 优先。与结束决定、尚未结算的 `TurnCancelSettled{session_ending}` 同一事务写入，早于终态 |

- **`EndSession` 与 `kept` 竞争**：账本在已持久受理 `EndSession` 之后拒绝 `kept`，与栅栏决策 D4 中“账本拒绝 `session_ending`
  以外的取消结论”是同一条规则。被拒时同一事务中的取消结论也不写入；这一轮在 `EndSession` 的结束决定事务中得到
  `session_ending`，并入的取消同样得到 `session_ending`。所以 `kept` 写入之后不会出现同一次停止的 `session_ending`。
- **重启与账本恢复后补写**：Node 写任何结束决定时，包括重启收口写 `interrupted` 结束决定、账本恢复可写后补写的结束决定，
  都为这个执行每条仍待结算的失败轮在同一事务中写 `TurnFailureSettled{session_ending}`，并排在尚未结算的
  `TurnCancelSettled{session_ending}` 之前。待结算标记在持久化带 `failure` 的 `turnEnded` Thread 事件的同一事务中写入，只限选择了
  `keep_session_on_failure` 的会话；它不属于 outbox，Controller 的 ACK 不会删除它；`kept` 与每个结束决定都消费它，所以每轮恰好
  一条结论。只读已持久化的 Thread 事件做不到这一点：outbox 在 ACK 后删除事件，Controller 已经看到失败的 `TurnEnded` 之后再
  重启或结束，就找不到这一轮。这与重启收口为未结算的 `CancelTurn` 写 `session_ending`（取消决策 D3、D4）对称。账本持续不可写时，
  按取消决策 D4 不发送未落账的事件。`kept` 只在这一轮有待结算标记时写入。
- **没有 Thread 事件的失败没有结论**：运行时没能写出 `TurnEnded`（D2），或会话 JSONL 已写而 Thread 事件还没有持久化时
  Node 崩溃，Controller 没有看到这一轮的失败，也就不等它的结论；会话由终态收尾。
- 它与 Thread 事件、`TurnCancelSettled`、`SessionTurnFailed` 共享执行内的序号空间：持久化后才发送，确认前保留，重连后
  按原序号重放，受在途窗口约束。它不是 Thread 记录，不写入会话 JSONL，也不进入实时预览。
- 它不带原因：原因属于这一轮，只在 `TurnEnded.failure` 中；这条事件只说 Node 对会话的判定。
- 没有选择 `keep_session_on_failure` 的会话不产生这条事件；失败照旧以 `agent_failed` 与 detail 结束会话。
- 执行内的顺序：这一轮的 Thread 事件 → `TurnEnded{failure}` → `TurnFailureSettled` → 并入的 `TurnCancelSettled` → 下一轮的
  Thread 事件或终态。
- **期限**：栅栏运行时自 Agent 回应起 F + Q + R 内落账（栅栏决策 D4）；D2 的静态检查已决定结束会话、或属于 D3 的情况
  时，在结束决定时立即落账。Controller 看到带 `failure` 的 `TurnEnded` 之后，按取消决策 D4 的投递进展等待它，等待上限大于
  F + Q + R 加上传输余量。Controller 可以在收到它之前提交下一轮，下一轮排在后面；结论为 `session_ending` 时它按
  `EndSession` 的既有规则被丢弃。

为什么要一条结论事件（owner 裁定）：会话保留与否取决于 D2 判定，最长要等 F + Q + R；没有事件时，Controller 要么先当作
保留、之后又被终态推翻，期间页面显示“可以继续”、用户提交的一轮被丢弃，要么一直等到空闲超时。为什么不带原因：
原因已在 `TurnEnded` 中，两处各写一份就会出现不一致，也违背“原因属于这一轮、判定属于 Node”的分工。为什么另立能力
而不复用 `turn_cancel`：已经声明 `turn_cancel` 的 Controller（cloud v2-server）不认识这条事件；另立能力后，它在升级之前
仍能连接，只是不能选择失败后保留。为什么 `kept` 与并入的取消同一事务：栅栏决策 D4 记下了并入取消逐条写入、
`EndSession` 落在两条之间时结论不同的窗口；失败路径多了一条结论，若仍逐条写，就可能先写 `kept`、再被 `EndSession`
推翻。为什么重启后补写：Controller 只需要一条规则“看到失败就等结论”；补写读失败轮的待结算标记，它与 Thread 事件同一事务写入，不随 ACK 删除（账本 v17）。

收紧选择条件的兼容依据：已实现的取消决策只要求协商 `turn_cancel`，本决策另要求 `turn_failure`，但当前没有 Controller
设置这个字段：cloud v2-server 的 `chooseKeep` 只设置 `keep_session_on_cancel`，注释写明在失败原因上报落地之前不启用
（cloud `v2-server/internal/agentruntime/session.go`）；desktop 的 `apps/ora-controller` 两个字段都写 `false`
（`apps/ora-controller/src/cloud/agents/mapping.rs`），握手也不声明 `turn_cancel`（`apps/ora-controller/src/session.rs`）。

### D7：终态 detail 的三处变化

取消决策 D3 的情况表不变。下表三处 detail 随本决策改变，对所有会话生效，不论会话是否选择了新行为、连接是否协商了
`turn_cancel`：

| 情况 | 现在的 detail | 改为 |
| --- | --- | --- |
| 看门狗取消后，宽限期内 Agent 没有回应这次 prompt（没有并入 `CancelTurn`） | `agent_turn_failed` | `turn_cancel_unanswered`（与 D3 表一致） |
| 一轮进行中（取消没有写出时）Agent 进程退出或崩溃、ACP 连接断开、这次 prompt 的 ACP 传输出错 | `agent_turn_failed` | `agent_unavailable`（与 D3 表一致） |
| Agent 以 ACP 错误码 `-32000` 回应 prompt，文字不是两句已知文字之一 | `agent_turn_failed` | `agent_authentication_required`（跟随 `failure.code`） |

不变的：取消写出之后 Agent 没有回应或消失，仍为 `turn_cancel_unanswered`（D1）；Node 读取运行时的事件流在这一轮的终点
之前关闭，仍为 `agent_stream_closed`；其他 ACP 错误回应仍为 `agent_turn_failed`，看门狗得到回应后放弃也仍为
`agent_turn_failed`。因 ACP 错误回应结束会话时，detail 由 `failure.code` 决定：`agent_authentication_required` 对应同名
detail，`agent_error` 对应 `agent_turn_failed`。选择了失败后保留、D2 判定不通过的会话，detail 仍是 D2 的 detail；这一轮原来
的原因在 `TurnEnded.failure` 中，不再丢失。

**对旧 Controller 的影响**：desktop 的 `apps/ora-controller` 不声明 `turn_cancel`，它背后的 cloud 旧链路会收到这三个值。
cloud 把 Node 的 detail 映射为公开失败码时只接受一组已知值（cloud `internal/core/agent_run_failure.go` 的
`publicSessionFailureCode`），前端按码显示文案（cloud `frontend/src/features/issues/thread/thread-failure.ts`）：

- `agent_unavailable` 与 `agent_authentication_required` 已在其中，分别显示“Agent 当前不可用”与“Agent 未登录或缺少凭据”，
  比原来的“模型请求失败”更贴近事实；
- `turn_cancel_unanswered` 不在其中，会退回 `agent_failed`，显示“Agent 会话失败”，比原来的“模型请求失败”更笼统。在 cloud
  为它补上公开码与文案之前，这是已知代价（“落地顺序”第 2 步）。

为什么改代码而不改 ADR（owner 裁定）：D3 的表述是 Controller 已经依赖的承诺，`agent_turn_failed` 被读作“模型请求失败”，
用它描述 Agent 进程退出是误导。为什么对所有会话生效：detail 描述的是会话为什么结束，同一事实对不同 Controller 给出不同
的码，只会让同一个 Node 上的会话无法比较；只对选择了新行为的会话修正，还要在 Node 里为旧会话保留一条错误的映射。为什么
`-32000` 的 detail 跟随 `failure.code`：Controller 需要的正是“需要重新认证”，若 detail 仍为 `agent_turn_failed`，同一轮的
记录与终态就各说一套。为什么 Agent 一侧的消失都记为 `agent_unavailable`、`agent_stream_closed` 只留给 Node 读取的事件流
提前关闭：运行时分不清进程退出与 ACP 连接断开，两者都表现为同一个“连接丢失”；而事件流提前关闭是 Node 与运行时之间的
事，与 Agent 是否还在无关。代价是 cloud 旧前端把 `agent_stream_closed` 显示为“Agent 连接中断”，这个文案对 ACP 连接断开
更贴切，对 Node 内部的事件流关闭反而不准确；两种情况都结束会话，展示上的偏差留给 Controller 的文案处理。

### D8：启用失败后保留

取消决策“落地顺序”第 4 步的前置条件具体为：

1. 本决策落地：Node 写出 `failure` 与 `TurnFailureSettled`，D7 的 detail 已改变；
2. Controller 协商 `turn_cancel` 与 `turn_failure`，能按 `TurnEnded.failure` 展示失败与原因（包括需要重新认证），按
   `TurnFailureSettled` 决定会话是否可以继续；
3. 取消决策风险表中“OpenCode 内部异常后 server 端的执行可能仍在进行”一行有了实测结论，成立时那类错误归入兜底分支。

可能连接这个 Node 的全部 Controller 实例都满足 2 之后，Controller 才为新会话选择 `keep_session_on_failure`。Controller 如何
把 `failure` 与自己知道的原因合并由 Controller 的决策规定；cloud v2-server 的规则待补入
[模型代理决策](../../cloud/model-proxy/20261113-in-flight-model-calls-bounded-by-idle-and-total-limits.md)，本决策不复述。

## 不变量

1. 每条带 `ended_by` 或 `failure` 的 `TurnEnded`，`stop_reason` 都是 `cancelled`；两个字段至多出现一个。
2. `CancelTurn` 作用到的一轮记为 `caller_cancel`，不写 `failure`，即使 Agent 对取消回了 `end_turn` 或 ACP 错误，或在回应
   之前消失。
3. 运行时已经观察到失败之后受理的命令，不改写这一轮的 `failure`。
4. `failure.code` 属于 D2 的封闭集合；`acp_code` 与 `message` 只在原因来自 ACP 错误回应时出现。
5. `failure.message` 不含 `Cc`、`Cf`、`Zl`、`Zp` 字符，没有首尾 `White_Space`，UTF-8 编码后不超过 1024 字节；它不出现在
   终态 `detail`、`TurnFailureSettled`、Node 与运行时的日志或 Agent 切换的交接中。
6. 看门狗只在 Agent 回 `cancelled` 且还有下一个窗口时重发；其他回应以 `no_progress` 失败，`attempts` 等于写给 Agent 的
   `session/prompt` 次数（不含 load 之后的重建发送）。
7. 选择了 `keep_session_on_failure` 的会话，每条带 `failure`、已作为 Thread 事件持久化的 `TurnEnded`，在终态之前恰好得到一条
   `TurnFailureSettled`，位于这条 `TurnEnded` 之后、并入的 `TurnCancelSettled` 之前，早于下一轮的任何 Thread 事件；Node
   重启或账本恢复后也是如此。
8. `TurnFailureSettled{kept}` 写入时，取消决策 D2 与栅栏决策的保留条件都已成立，并入的取消在同一事务中得到
   `already_ended`，且没有已受理的 `EndSession`；`session_ending` 写入之后，Node 不再向这个会话的 Agent 写出任何
   `session/prompt`。
9. 没有协商 `turn_failure` 的连接收到的握手不含这项能力，也收不到 `TurnFailureSettled`；只有协商了 `turn_cancel` 与
   `turn_failure` 的连接能选择 `keep_session_on_failure`。
10. 两个字段都省略的 `TurnEnded` 与本决策之前逐字节相同；旧行在新读者中、新行在忽略未知字段的旧读者中都能解码。
11. 对所有会话：看门狗取消没有得到回应、或取消写出之后 Agent 没有回应，detail 为 `turn_cancel_unanswered`；取消没有写出
    时 Agent 进程退出或 ACP 连接断开，detail 为 `agent_unavailable`；因 ACP 错误回应结束会话时，`failure.code` 为
    `agent_authentication_required` 则 detail 同名，否则为 `agent_turn_failed`。

## 为什么不是这些替代方案

| 方案 | 优势与取舍 |
| --- | --- |
| 新增记录类型 `turnFailed` | 读者一眼可见；但 Cloud 只接受已知 `kind`，cloud v2-server 读 JSONL 时把未知类型当作损坏，旧 Controller 会因此拒绝整段历史（D4） |
| `stop_reason` 改为新值 `failed` | 最直接；但 `stop_reason` 是 ACP 的封闭枚举，Thread 根决策、Workflow 会话结束与交互节点完成都依赖“失败也是 `cancelled`”（问题与约束） |
| 扩展 `SessionTurnFailed`，为准入后的失败加新码与文字 | 复用已有的事件；但它的码是封闭枚举，cloud v2-server 对其他码按协议违规结束会话；它的含义是“没有 `TurnEnded` 的一轮”，用在有 `TurnEnded` 的轮次上会让一轮有两个结果（D2） |
| 原因只放进 `TurnFailureSettled`，不写历史 | 原因只经协商的连接传递；但没有选择失败后保留的会话拿不到原因，JSONL 与 Revision 归档里也没有，回放与排查看不到；同一事实分在两处（D6） |
| 原因只放进终态 `detail` | 不改记录；但会话保留时没有终态，原因无处可放；`detail` 只能是码，放不下 Agent 的文字（D2、D3） |
| 只带码，不带 Agent 的文字 | 没有第三方文字进入归档；但具体原因（额度用完、模型不存在）只有 Agent 知道，用户只能看到分类。owner 选择带有界文字（D3） |
| 只取 JSON-RPC 的 `message`，不取 `data` | 来源单一；但 ACP 要求 `message` 简短，细节常在 `data` 中，只取 `message` 时用户常常只看到“Internal error”（D3） |
| 把结构化的 `data` 整体序列化进文字 | 不丢信息；但它没有约定的形状，可能很大、含内部字段，序列化后既不可读也难以清洗（D3） |
| 文字不清洗、不限长，或只删控制字符与双向文字控制符 | 保留 Agent 的原貌；但零宽字符、标签字符与行分隔符同样能伪造展示，过长的文字挤占 Thread 条目（D3） |
| 看门狗被回 `end_turn` 时，若这一次尝试没有产生输出就当作确认取消并重发 | 网络抖动时能自愈；但要按尝试追踪输出，而“没有输出”不等于“没有运行工具”，重发可能重复工具的副作用，并在 Agent 自己的重发之上再叠一层（D5） |
| 看门狗被回 `end_turn` 时当作正常完成 | 不打断对话；但这一轮实际上没有进展，失败被藏起来，调用方会把半截回答当作结果（D5） |
| 看门狗收到 `-32800` 时当作确认取消并重发 | 语义上就是“已取消”；但 ACP 没有约定以错误回应取消时 provider 会话的状态，先实测再定（D5） |
| 不加结论事件，Controller 先当作保留、终态到了再改 | 协议不变；但判定期间（最长 F + Q + R）页面显示“可以继续”，之后又结束，期间提交的一轮被丢弃（D6） |
| `TurnFailureSettled` 对所有会话发送，不看选择 | 规则统一；但没有选择失败后保留的会话本来就以终态结束，多一条事件只增加噪声，并改变取消决策不变量 11 承诺的“行为不变”（D6） |
| 复用 `turn_cancel` 能力，不另立 `turn_failure` | 少一项协商；但已经声明 `turn_cancel` 的 Controller 不认识新事件，Node 升级后会把它投递给不理解它的 Controller（D6） |
| `kept` 与并入的取消逐条写入 | 与栅栏决策第 8 步的现有写法一致；但 `EndSession` 落在两条之间时，Controller 会先看到“会话保留”再收到 `session_ending` 与终态（D6） |
| 重启后不补写，Controller 先收到终态就按 `session_ending` 处理 | Node 少一段重启逻辑；但 Controller 要多维护一条分支，“恰好一条”只在 Node 不重启时成立；补写读与 Thread 事件同一事务写入的待结算标记（D6） |
| `ended_by` 区分停止按钮、`CancelTurn` 与 `EndSession` 的各个原因 | 来源更细；但现有消费者只需要“是不是自己的取消”，会话结束的原因已在终态中。owner 选择本版只有两个值（D1） |
| 取消写出之后 Agent 消失记为 `failure: agent_unavailable` | 直接说出 Agent 不在了；但与“取消写出后的结局都归这次取消”矛盾，Controller 无法据 `ended_by` 判断这是不是自己的取消（D1） |
| 修改取消决策 D3，照实写为 `agent_turn_failed` | 不改代码；但 Controller 把 `agent_turn_failed` 读作模型请求失败，用它描述 Agent 进程退出是误导。owner 选择改代码（D7） |
| 只对选择了新行为的会话修正 detail | 不声明 `turn_cancel` 的 Controller 看不到 `turn_cancel_unanswered`；但同一事实在同一个 Node 上有两个码，Node 还要保留一条错误的映射（D7） |
| `-32000` 时 detail 仍为 `agent_turn_failed` | 不增加 detail 的变化；但同一轮的 `failure.code` 与终态 detail 各说一套，旧 Controller 也看不到“需要重新认证”（D7） |

## 已知风险与待实测

| 风险 | 影响 | 处理 |
| --- | --- | --- |
| 某个 Agent 把 `-32000` 用作一般错误 | 普通错误被记为 `agent_authentication_required`，用户被引导去重新认证 | 实现时在部署的 OpenCode 0.6.11 与 Claude 0.4.5 上实测各类错误的错误码；成立时只对该 Agent 退回文字匹配，并记入本决策 |
| 具体原因在 `data` 中，且不是字符串或 `data.message` 字符串 | 用户只看到“Internal error”这类通用文字 | 实现时在部署的 OpenCode 0.6.11 与 Claude 0.4.5 上实测模型失败、限流、额度用完、key 失效时错误对象的完整形状（`code`、`message`、`data`），结论记入本决策；需要其他形状时另行决定 |
| Agent 以 `-32800` 回应看门狗的取消 | 本可重发的一轮以 `no_progress` 失败 | 同一次实测记录两个 Agent 回应取消的方式；需要按确认处理时另行决定（D5） |
| Agent 的文字含内部地址、凭据片段或用户数据 | 文字进入 Thread 条目与 Revision 归档 | owner 已接受；清洗与限长不能去掉敏感内容，只防止伪造展示与超长（D3） |
| Agent 不给有用的文字 | 用户只看到分类与一句无意义的文字 | 分类码仍可展示；文字缺失时 Controller 只展示码对应的说明 |
| 删除格式字符拆开组合表情 | 文字中的表情显示为几个单独的字符 | 接受；Agent 的错误文字里很少出现（D3） |
| 模型请求失败后 Agent 先自己重发，看门狗先放弃 | 原因是 `no_progress`，而真实原因是上游模型失败 | 记 `attempts` 与 `answer`；Controller 按自己的决策与它知道的模型原因合并（D8） |
| cloud 旧链路没有 `turn_cancel_unanswered` 的公开码与文案 | 看门狗取消无回应的会话显示为“Agent 会话失败”，比原来更笼统 | cloud 补上公开码与文案（“落地顺序”第 2 步）；在此之前接受（D7） |
| OpenCode 内部异常后 server 端的执行可能仍在进行 | 失败后保留的会话里，下一轮混入上一轮未停的工作 | 沿用取消决策风险表，作为启用失败后保留的前置条件（D8） |
| 旧记录没有归属 | 回放旧会话时，失败与取消都只显示为没有归属的 `cancelled` | 不回填、不推断（D1） |
| Desktop 本机会话也写入新字段 | 本机 JSONL 中出现 Agent 的文字 | 与 Node 同一份规则；Desktop 是否展示由 Desktop 的决策决定 |

## 验证义务

以下各条转为[失败原因核心用例](../../../test-cases/node/protocol/turn-failure-reason.md)，编号一一对应。取消决策验证义务
6、18、19、20、30 的修改，在[取消一轮用例](../../../test-cases/node/protocol/cancel-turn-and-keep-session.md)中以“（失败原因）”行
登记，不另占编号。

1. ACP 错误回应：`TurnEnded{stop_reason: cancelled, failure: {code: agent_error, acp_code, message}}`；`message` 为主文字，`data`
   为字符串或 `data.message` 为字符串时接上补充文字，其他形状的 `data` 不取；Thread 事件原样转发这条记录；终态 detail 与
   `TurnFailureSettled` 不含文字。
2. 认证：错误码为 `-32000`，或文字恰为两句已知文字之一时为 `agent_authentication_required`；其他错误码与文字为
   `agent_error`。没有选择失败后保留时，`-32000` 的终态 detail 为 `agent_authentication_required`。
3. 文字清洗与限长：`Cc`、`Cf`（含双向文字控制符、零宽字符、U+FEFF、标签字符）被删除，换行、制表符、`Zl`、`Zp` 变为空格，
   首尾 `White_Space` 被去掉；超过 1024 字节时在码位边界截断，带 `message_truncated`；清洗后为空时省略；结果是合法
   UTF-8；日志与交接中没有这段文字。
4. 看门狗被回 `end_turn`、其他非 `cancelled` 的 stop reason 或 ACP 错误（包括 `-32800`）：不重发，`session/prompt` 的写出
   次数不再增加，`failure` 为 `no_progress`，`attempts` 与 `answer` 正确，load 之后的重建发送不计入 `attempts`；`answer` 为
   `error` 时带错误码与文字。
5. 看门狗重发用尽：每次都回 `cancelled`，第 4 个窗口之后放弃，`attempts` 为 4、`answer` 为 `cancelled`，记录中有 3 条
   `PromptRetried`。
6. 看门狗宽限期内没有回应：`failure` 为 `no_progress`、`answer` 为 `none`，会话以 `turn_cancel_unanswered` 结束；会话选择了
   失败后保留时，先有 `TurnFailureSettled{session_ending}`。
7. Agent 不在与 Ora 内部错误：取消没有写出时，Agent 进程退出、ACP 连接断开、看门狗等待期间连接关闭，`failure` 为
   `agent_unavailable`，会话以 `agent_unavailable` 结束；事件队列溢出、Agent 违反 ACP 帧或回应格式，`failure` 为 `internal`；
   Node 读取的运行时事件流在终点之前关闭时为 `agent_stream_closed`。会话选择了失败后保留时，带 `failure` 的情况先有
   `TurnFailureSettled{session_ending}`。
8. 调用方取消：prompt 写出之前被拦下、取消写出之后 Agent 回 `cancelled`、`end_turn` 或 ACP 错误、看门狗停止期间并入、
   截止时刻前没有回应、取消写出之后 Agent 进程退出或 ACP 连接断开，`TurnEnded` 都带 `ended_by: caller_cancel`，没有
   `failure`；后两种会话以 `turn_cancel_unanswered` 结束。失败的回应在取消写出之前已在队列中时，记为失败。
9. 会话结束：`EndSession` 或 Node 的结束决定停止正在执行的一轮时为 `session_end`；会话已开始结束之后 Agent 才回错误，
   仍为 `session_end`；运行时已观察到失败之后才受理的 `EndSession`，`failure` 不变，会话以 `EndSession` 的原因结束，选择了
   失败后保留时 `TurnFailureSettled` 为 `session_ending`。D2 判定的最后一次读队列之后、`kept` 写入之前受理 `EndSession`：
   不写 `kept`，这一轮与并入的取消都在结束决定中得到 `session_ending`。
10. 记录兼容：Agent 自己以 `cancelled` 结束的一轮与旧记录都没有两个新字段；两个字段都省略的新行与旧行逐字节相同；旧行
    在新读者中、新行在旧读者与 cloud v2-server 的历史读取中都能解码。
11. 失败后保留：选择了 `keep_session_on_failure` 的会话，ACP 错误与得到回应的看门狗放弃满足保留条件时，恰好一条
    `TurnFailureSettled{kept}`，与并入的 `TurnCancelSettled{already_ended}` 同一事务、位于 `TurnEnded` 之后与它们之前、
    下一轮之前，自 Agent 回应起 F + Q + R 内落账。
12. 失败后不保留：D2 的每种不通过、D3 的各种情况与已受理的 `EndSession`，都得到恰好一条
    `TurnFailureSettled{session_ending}`，与结束决定同一事务、早于终态；终态 detail 是 D2 或 D3 的 detail，原失败原因保留在
    `TurnEnded.failure` 中。没有写出 `TurnEnded` 的失败（会话记录已降级、运行时事件流提前关闭）没有 `TurnFailureSettled`。
13. 协商：`HelloAccepted` 只在 `Hello` 声明了 `turn_cancel` 与 `turn_failure` 时含 `turn_failure`，校验拒绝不对称的确认；旧
    Controller 的握手不变；未协商 `turn_failure` 却选择 `keep_session_on_failure` 时连接按协议违规关闭；有这类会话时，未协商
    `turn_failure` 的连接以 `4403` 被拒绝，不收到任何事件。
14. 没有选择失败后保留的会话：没有 `TurnFailureSettled`；一轮失败仍以 `agent_failed` 与 detail 结束会话，detail 只有 D7 的
    三处变化；`TurnEnded` 仍带 `failure`。
15. 重启与账本恢复：Node 在失败的 `TurnEnded` 持久化之后、`TurnFailureSettled` 之前重启，重启收口在 `interrupted` 结束决定的
    同一事务中补写恰好一条 `TurnFailureSettled{session_ending}`，早于终态；账本持续不可写时不发送未落账的事件，恢复后与
    结束决定同一事务补写；已有结论的轮次不重复写。

## 本决策未解决的问题

- **展示**：各 Controller 如何展示 `failure` 与 `message`、如何与 Controller 自己知道的原因合并，由 Controller 的决策规定；
  cloud v2-server 的规则待补入模型代理决策。
- **更细的归属**：区分停止按钮、`CancelTurn` 与 `EndSession` 的各个原因，等有消费者时再加。
- **provider 会话已失效的判定**：仍按固定文字匹配（取消决策差异表）；有了 `acp_code` 之后能否改按错误码判定，要先实测
  各 Agent 的错误码。
- **`-32800` 是否算确认取消**：等实测（D5）。
- **重启后的会话恢复**：沿用取消决策的未解决问题。

## 落地顺序

1. **评审本 ADR**：按评审意见纠正后由 owner 批准为 `approved`，核心用例随它一同评审（沿用栅栏决策的先例）。取消决策、
   栅栏决策与 agent-runtime 根决策标题下的三条 `> 历史决策：` 指针随本 ADR 评审：本 ADR 的修改范围在评审中改变时，同步改写
   这三条指针；本 ADR 被放弃时撤回它们。
2. **实现**（desktop 为主）：`crates/acp` 保留 ACP 错误码与 `data`；`crates/history` 增加 `TurnEnded` 的两个字段；
   `crates/agent-runtime` 按 D1 判定归属，按 D2、D3、D5 写失败原因与清洗后的文字；`crates/node-protocol` 增加能力、握手字段、
   事件与对称校验，`crates/node-db` 持久化它；`apps/ora-node/src/session/` 按 D6 结算（含 `kept` 与并入取消同一事务、账本拒绝
   `kept`、结束决定时补写），并按 D7 改变三处 detail。cloud 的旧链路为 `turn_cancel_unanswered` 补上公开失败码与文案（cloud
   `internal/core/agent_run_failure.go`、`frontend/src/features/issues/thread/thread-failure.ts`），与 Node 的改变同一版本部署，
   否则接受 D7 所述的展示代价。完成条件：“验证义务”各条有直接测试；在部署的 OpenCode 0.6.11 与 Claude 0.4.5 上实测错误
   对象的形状、`-32000` 的用法与回应取消的方式，结论写入“已知风险与待实测”。这一步之后，所有会话的 `TurnEnded` 开始带
   归属与原因，三处 detail 开始改变；没有 Controller 选择失败后保留，所以还没有 `TurnFailureSettled`。
3. **Controller 接入**（不属于 Node 侧，由各 Controller 落地）：协商 `turn_failure`，读取 `TurnEnded.failure` 并展示，按
   `TurnFailureSettled` 决定会话是否可以继续；满足 D8 后为新会话选择 `keep_session_on_failure`，即取消决策“落地顺序”第 4 步。
4. **落地**：本 ADR 改为 `implemented`，按实现提交同步正文与核心用例证据。

保持不变：`stop_reason`、旧记录与旧文件、没有协商 `turn_failure` 的连接、两个保留字段都没有选择的会话（除 D7 的三处
detail）。从第 2 步开始改变：`TurnEnded` 带归属与失败原因，三处 detail 按 D7；从第 3 步开始改变：选择了失败后保留的会话，
一轮失败后可以保留，并有明确的结论。
