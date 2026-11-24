---
status: implemented
---

# Node 协议增加"取消一轮"命令；一轮以取消或失败结束、且 Agent 已静止时，会话保留

> 历史决策：D3 中复述的停止证据（“原尝试锁与全部直接进程线索，进程组 BestEffort”及“本决策不加强也不削弱它”）
> 与风险表中对应的一句，已由[会话尝试的进程带环境标记](../agent-runtime/20261120-attempt-mark-in-session-stop-evidence.md)
> （`implemented`）修改：停止证据加入尝试环境标记扫描，脱组但保留环境的后台作业在终态之前被杀，仍为 BestEffort。
> 这些部分以该决策为准。

> 历史决策：[保留会话前先停掉这一轮新起的进程](20261123-turn-process-fence-before-keeping-session.md)（`implemented`）
> 修改本决策的以下部分：D2 的“可依赖的轮次边界”改为 Agent 回应加 Node 轮次进程栅栏的组合，清单按（插件，版本，平台）
> 登记，保留会话另要求栅栏与结算前的复扫都干净，静默窗口改为从栅栏干净时起算；D3 情况表新增 detail `turn_processes_unstopped`；
> D1、D4 与不变量 2 的结算期限在栅栏运行时为 G + F + Q + R（F 为 5 秒、R 为 1 秒，共 23 秒），T 须大于它加传输余量；不变量 7、风险表中关于后台任务的一行、验证义务
> 10 与 13 相应修改。这些部分以该决策为准。

> 历史决策：[一轮的失败与取消分开记录](20261124-turn-failure-reason-distinct-from-cancel.md)（`approved`）修改本决策的
> 以下部分：D2 结束方式表的“记录”一列与末段“本决策不区分两者”，改为 `TurnEnded` 带 `ended_by` 或 `failure`，`stop_reason`
> 仍为 `cancelled`；D3 结束决定的事务另写失败一轮的 `TurnFailureSettled{session_ending}`；D4 的事件顺序在失败的一轮之后
> 增加 `TurnFailureSettled`；D5 新增能力 `turn_failure`，`keep_session_on_failure` 另要求协商它，`Hello.turn_cancel` 的含义与
> “不会产生新的 detail”一条随 detail 的变化修改，启用顺序的前置条件随之具体化；D6 由该决策承担；不变量 11、风险表最后
> 一行、验证义务 6、18、19、20、30 与“落地顺序”第 4 步相应修改。D3 情况表不变；终态 detail 有三处变化，对所有会话生效：
> 实现中与情况表不一致的两处改为一致，ACP 错误码 `-32000` 结束的会话为 `agent_authentication_required`。这些部分以该决策为准。

Controller 新增会话命令 `CancelTurn{command_id, turn_id}`。Node 持久受理后，在有界时间内在本地确定这次取消的
结论，每个被受理的取消恰好得到一条新的非终态事件 `TurnCancelSettled`：目标轮次正在执行时，向 Agent 发 ACP
`session/cancel`，这一轮以 `TurnEnded{cancelled}` 结束；目标轮次已被准入、但 prompt 还没有写给 Agent 时不再写出；
目标轮次还没有被准入（在排队，或在等会话接入、MCP 门禁）时直接丢弃；目标轮次已经在停止中时，并入那一次停止的
结论；目标轮次已经有结果或不存在时不做任何事。会话保留分两项逐会话选择：`keep_session_on_cancel`（一轮被
`CancelTurn` 取消后保留）与 `keep_session_on_failure`（一轮失败后保留，失败指 ACP prompt 报错、无进展看门狗放弃）。
选择了对应一项的会话，在一轮以该方式结束后，只有 Agent 具备可依赖的轮次边界、且满足可验证的静止条件（D2）时才
保留会话、接着执行下一轮；
否则，或属于 D3 清单中的情况时，Node 不再向 Agent 派发任何一轮并请求停止 Agent，这次取消的结论是
`session_ending`，会话终态按[会话 MCP 刷新与宿主健康](../agent-runtime/20261116-session-mcp-refresh-and-host-health.md)
D5 在取得停止证据之后才发出。新行为由 Node 与 Controller 在握手中共同协商 `turn_cancel`、Controller 在
`StartAgentSession` 中逐会话选择；第一版只启用取消后保留，失败后保留等失败原因上报落地后再启用（D5）。没有选择的
会话和旧对端保持原行为，没有声明 `turn_cancel` 的 Controller 看到的
握手内容不变，不具备该能力的 Controller 不能接管已选择新行为的会话。

本 ADR 状态为 `implemented`：Node 一侧由 desktop main 的提交 `2fc5e2cacdf3c47ca840785e141167ff947aa5aa` 实现（协议、账本、
共享运行时、会话执行、握手与接管），当时轮次边界清单为空；提交 `447a59de6722500cae0be011860e63e5127a3b23` 加入
[轮次进程栅栏](20261123-turn-process-fence-before-keeping-session.md)并登记 Linux 上的 `official/ora-space.opencode 0.6.11`，
运行中被取消的一轮自此可以保留会话。该状态表示 Node 侧的机制已经落地，不表示 Controller 已接入、失败后保留已启用
（“落地顺序”第 3、4 步），也不表示每条验证义务都有完整的直接证据：
[核心用例](../../../test-cases/node/protocol/cancel-turn-and-keep-session.md)与“验证义务”编号一一对应，逐条记
`Covered`、`Partial`、`Missing`。实现与批准文本不同或更具体的地方汇总在“实现与批准文本的差异”。影响 desktop 的 `crates/node-protocol`（消息、事件、握手字段、能力、
会话输入字段）、`crates/node-db`（受理新命令、持久化新事件）与 `apps/ora-node/src/session/`（会话执行）；共享
`crates/agent-runtime` 已有取消当前一轮、取消宽限期、丢弃空闲期流量与续用 provider 会话的能力；本决策要它
区分用户取消与看门狗取消的宽限期、在准入提交处和看门狗重发前认取消、识别已失效的 provider 会话，并向
Node 报告 D2 判定所需的事实。结束会话沿用已经落地的停止责任与停止证据（会话 MCP 刷新 D5），本决策不修改它。
Controller 侧（停止按钮改用新命令、展示一轮的结束）由各 Controller 自己接入，不在本 ADR 内。兼容性边界：
已有消息、已有会话输入和未选择新行为的会话完全不变；不恢复 Node 重启前的会话（协议 D6 不变）；失败原因的
上报不在本 ADR 内（见 D6）。

## 继承与修改

继续[实时预览](20261018-live-thread-previews.md)，会话命令、事件与终态的规则来自
[会话执行与命令](20260928-streamed-thread-events-and-session-commands.md)，连接关闭的分类来自
[Controller 心跳与 close code](20260925-controller-heartbeat-and-websocket-close-codes.md)，结束规则来自
[agent-runtime 根决策](../agent-runtime/0-shared-agent-runtime-crate-hosted-by-node.md) D6，子代理投影的
终态规则来自[子代理投影、历史与诊断](../agent-runtime/20261020-child-projection-history-and-diagnostics.md)，轮次准入、
MCP 门禁和结束会话时的停止责任来自[会话 MCP 刷新与宿主健康](../agent-runtime/20261116-session-mcp-refresh-and-host-health.md)。

| 前序约定 | 本文处理 |
| --- | --- |
| 协议根：能力在握手中协商，执行输入先持久化再启动，同输入重发幂等 | 继承；新增能力 `turn_cancel`，新字段属于执行输入（D5） |
| 实时预览：Controller 在 `Hello` 中声明可选能力，双方都支持时 `HelloAccepted` 才确认 | 继承同一形式；新增 `Hello.turn_cancel` 与 `HelloAccepted.turn_cancel`（D5） |
| close code：身份或能力不符时以 `4403` 结束会话，归为 `Mismatch` | 继承；Node 拒绝不具备 `turn_cancel` 的 Controller 接管已选择新行为的会话时使用它（D5） |
| 会话命令 D4：命令先持久受理再回复 `SessionCommandAccepted`；同一 `command_id` 至多执行一次；用户轮次按受理顺序执行 | 继承；`CancelTurn` 沿用同一受理、幂等与回复规则（D1） |
| 会话命令 D4：只有 `EndSession` 丢弃排队轮次 | 修改：`CancelTurn` 也可以丢弃它指定的那一个排队轮次，并以 `TurnCancelSettled{not_started}` 告知（D1） |
| 会话命令 D4：`EndSession` 取消当前轮次、丢弃排队轮次并结束会话 | 继承，不变；`EndSession` 优先于 `CancelTurn`，尚未结算的 `CancelTurn` 得到 `session_ending`（D4） |
| 会话命令 D4：会话已终态时命令回复 `SessionCommandRejected{session_ended}` | 继承；`CancelTurn` 同样适用。新增拒绝原因 `turn_cancel_unsupported`（D1） |
| 会话执行 D2 与会话 MCP 刷新 D4：非终态事件是 Thread 事件与 `SessionTurnFailed`，与终态共享序号空间，持久化后才发送，确认前保留，重连后重放 | 修改：新增非终态事件 `TurnCancelSettled`，沿用同一序号、持久化、确认与重放规则；它不是 Thread 记录，不写入会话 JSONL（D1） |
| 会话 MCP 刷新 D4：轮次在 Actor 最终准入时才结算 Executed；准入前可能等会话接入、MCP 门禁；从首次受阻起 300 秒未准入，以 `SessionTurnFailed{mcp_refresh_wait_timeout}` 与命令 Failed 同事务结算 | 继承；准入前的轮次可被 `CancelTurn` 丢弃，取消、准入与等候超时在同一边界上以先提交者为准（D1） |
| 会话 MCP 刷新 D4：生命周期模式的会话只 `session/load` 原 provider ID，失败时 blocked，不新建 provider 会话 | 继承；D2 的续用方式对这些会话同样只 load（D2） |
| 会话 MCP 刷新 D5：所有 Node 会话结束时交接停止责任，全部责任取得停止证据后才发终态；超期登记 `stop_unconfirmed`，终态可能始终缺席 | 继承，不变；D3 的结束会话都走这条路径，取消的结论不等终态（D1、D3） |
| 会话 MCP 刷新 D3：新消息要求 Controller 与 Node 同版本发布，不对旧 Rust 枚举解码器透明 | 继承；另外只向声明了 `turn_cancel` 的 Controller 返回新能力，不选择新行为的旧 Controller 握手不变（D5） |
| 会话执行 D3：每个执行最多 256 个已发送未确认的事件，满时暂停发送、不暂停记录 | 继承，不变；结论同样受窗口约束，因此本文区分本地结算期限与投递，Controller 按投递进展等待（D4） |
| 协议 D6：Node 重启后会话以 `interrupted` 结束，不恢复 Agent | 继承，不变 |
| 实时预览：旁路不改变权威状态，查看与断开不触发取消 | 继承，不变 |
| 子代理投影：不从静默、父工具完成或父 prompt 结束推断子代理终态 | 继承；启动过子代理投影的轮次不满足 D2 的静止条件 |
| agent-runtime D6："轮次失败、超时、连接丢失：记录 `TurnEnded`，会话保持" | 修改。当前实现为"只有成功的 ACP 完成才接受下一轮，失败以 `agent_failed` 结束会话"，与 D6 原文不符。改为：连接丢失结束会话；失败与超时只在选择了 `keep_session_on_failure`、且满足 D2 时保留会话（D2、D3）；MCP 门禁中的轮次失败按会话 MCP 刷新 D4，本来就不结束会话；未选择的会话保持当前实现的行为 |

## 问题与约束

Node 协议的会话命令只有 `SubmitUserTurn` 与 `EndSession`，没有"只取消当前一轮"。Controller 想停下一轮，只能
`EndSession`，会话随之结束，用户的对话只能新开、之前的上下文全部丢失。

一轮失败同样结束会话：模型服务报错或限流、key 失效、要求重新认证、模型长时间没有输出被看门狗放弃，这些情况下
Agent 进程通常还活着，只是这一轮没能完成，但 Node 以 `agent_failed` 结束整个会话。这样做的原因是：此前失败的
一轮被当作正常完成，调用方看不出这一轮失败，会话停在空闲直到空闲超时，同时一直占着插件租约和模型授权。所以
任何放宽都必须满足：

- 依赖"失败即终态"的调用方（例如以会话终态结算一次运行的 Controller）行为不变；
- 每一次取消都在有界时间内在 Node 本地得到确定结论，并按序送达 Controller，不能让 Controller 在某个分支上
  无限等待，也不能让投递背压被误判为 Node 未能停止；
- 会话结束要等停止证据：Node 只有取得本执行全部 Agent 责任的停止证据才写终态，证据可能迟到，也可能始终缺席，
  所以取消的结论不能依赖终态；
- 一轮在被 Actor 准入之前可能等待会话接入或 MCP 门禁，期间可能因等候超时失败，取消要与准入、等候超时有确定的
  先后；
- 同一会话的下一轮开始前，上一轮的工作必须可验证地静止：ACP 的更新不带请求身份。上一轮回应之后、下一轮
  发出之前到达的更新，运行时已经丢弃；下一轮的 prompt 发出之后才到达的上一轮更新，Node 无法把它和下一轮
  的更新分开，上一轮仍在运行的工具或子代理也可能继续修改 checkout；
- 用户的停止与 Agent 自己结束这一轮存在竞争，取消不能误伤用户刚提交的下一轮，重复的停止请求也不能得到与
  实际不符的结论；
- 会话跨连接存在，任何时刻接管它的 Controller 都必须理解它会收到的事件。

这里的"并发修改"只指同一会话内的两轮。同一 checkout 可能同时承载多个会话，不同会话之间的并发修改由
checkout 共享的规则约束，不属于本决策。

共享运行时已经具备所需的底层能力：取消当前一轮时先以 `cancelled` 应答挂起的权限请求，再发 `session/cancel`，
在宽限期内等待 Agent 对这次 prompt 的回应；回应到达时 provider 会话可以续用，回应没有到达时运行时隔离该
provider 会话，下一轮通过 `session/load`（Agent 支持时）或带 Ora 记录交接的新 provider 会话继续。运行时在
本机会话上已经这样取消一轮并续用会话：它以回应作为唯一的栅栏，在两次 prompt 之间丢弃空闲期到达的更新、
以 `cancelled` 应答空闲期的权限请求，并保留标题更新。本决策规定 Node 协议与会话执行如何使用这些能力，
以及何时不能使用。

## 决策

### D1：`CancelTurn` 命令及其结论

`CancelTurn{node_id, command_id, turn_id}`，Controller → Node，与 `SubmitUserTurn`、`EndSession` 同属一个会话执行。
持久化后回复 `SessionCommandAccepted{command_id}`；同一 `command_id` 重复发送只回复受理，内容不同以协议错误
拒绝，与其他会话命令相同。目标可以是这个会话任何一轮的 `turn_id`，包括随 `StartAgentSession` 提交的初始轮次。

`SessionCommandAccepted` 只表示命令已持久受理。每个被受理的 `CancelTurn` 恰好得到一条非终态事件
`TurnCancelSettled{command_id, turn_id, outcome}`，它是这次取消的结论，与这个命令自己的结算在同一事务中写入：

| `outcome` | 含义 |
| --- | --- |
| `stopped` | 目标轮次因取消而停，会话保留，可以接着执行下一轮 |
| `not_started` | 目标轮次在准入之前被本命令丢弃，Agent 从未收到它 |
| `already_ended` | 本命令生效之前，目标轮次已经有了自己的结果（`TurnEnded`、`SessionTurnFailed`，或已被更早的取消丢弃）；本命令什么也没做 |
| `unknown` | 这个会话没有受理过这个 `turn_id` |
| `session_ending` | 会话不会保留：Node 不再向 Agent 派发任何一轮，已经请求或即将请求停止 Agent。它不表示 Agent 已经停止，也不是终态 |

结论不由会话终态表示。会话是否已经结束、Agent 是否已经停止，只以终态 `AgentSessionEnded` 和
`GetExecutionStatus` 为准（D3）。

**轮次以 Actor 准入为界分成"排队"和"已开始"。** 一轮被受理后先排队。Node 的对话循环让它出队后，要等会话接入
完成、MCP 门禁放行，Actor 才准入它，并在同一个提交点把它的 `SubmitUserTurn` 结算为 Executed（会话 MCP 刷新 D4；
初始轮次没有命令，同样经过这次准入）。准入之前这一轮没有任何记录，Agent 也没有收到它；准入之后才记录用户消息、
写出 `session/prompt`。所以排在其他轮次后面的轮次、已到队首但还在等会话接入或 MCP 门禁的轮次，都是"排队"。

一轮从被取消或失败开始、到 D2 判定会话保留或结束为止，处于"停止中"。按本命令生效时目标轮次 `turn_id` 的状态
（"生效"指对话循环在串行结算边界上处理到它，受理后只隔账本写入）：

| 状态 | Node 行为 | 结论 | 本地结算期限（自受理起） |
| --- | --- | --- | --- |
| 排队：还没有被准入 | 从队列中丢弃，永不发给 Agent，不写它的用户消息和 `TurnEnded`；正在进行的会话接入和 MCP 收敛不因此取消 | 目标命令结算为 Discarded，同一事务写 `TurnCancelSettled{not_started}` | 立即（只受账本写入约束） |
| 已准入，`session/prompt` 还没有写给 Agent | 不写出这次 prompt | 先写 `TurnEnded{cancelled}`，再写 `TurnCancelSettled{stopped}`，会话保留；Agent 没有收到这一轮，不需要 D2 判定 | 立即 |
| 正在执行（prompt 已写给 Agent） | 以 `cancelled` 应答挂起的权限请求，向 Agent 发 ACP `session/cancel`，在截止时刻前等待回应，再按 D2 判定 | 会话保留：先写 `TurnEnded{cancelled}`，再写 `TurnCancelSettled{stopped}`。会话不能保留：先写 `TurnEnded{cancelled}`，再写 `TurnCancelSettled{session_ending}`，会话按 D3 结束 | G + Q |
| 停止中（已有一个 `CancelTurn`，或看门狗正在取消这次 prompt，或这一轮已失败，正在等待回应或静默窗口） | 不再发 `session/cancel`，并入正在进行的那一次停止，不改变它的截止时刻（唯一的例外见下）；看门狗正在取消时，这一轮不再重发 | 与那一次停止同时结算：会话保留时结果为 `stopped`（这一轮因取消而停，包括看门狗取消期间受理的 `CancelTurn`）或 `already_ended`（这一轮因失败而停）；会话不能保留时为 `session_ending` | 不晚于那一次停止的截止时刻加 Q，因此也不超过自本命令受理起的 G + Q |
| 已有结果：正常完成；以取消或失败结束且 D2 已判定会话保留；以 `SessionTurnFailed` 失败；已被更早的 `CancelTurn` 丢弃 | 不做任何事 | `TurnCancelSettled{already_ended}` | 立即 |
| 没有这个 `turn_id` | 不做任何事 | `TurnCancelSettled{unknown}` | 立即 |
| 会话已经决定结束、终态还没有写入（D3 的任何情况，包括等待停止证据期间） | 不做任何事，不再区分目标轮次的状态 | `TurnCancelSettled{session_ending}`，在受理的同一事务中写入 | 立即 |

- **取消、准入与 MCP 等候超时的先后**：在一个还没有被准入的轮次上，这三件事在同一个串行结算边界上竞争，以账本中
  先提交的为准，与命令受理的先后无关，这一轮只有一个结果：
  - 取消先提交：目标命令结算为 Discarded，同一事务写 `TurnCancelSettled{not_started}`。这一轮的 MCP 等候计时
    随之作废，之后不为它写 `SessionTurnFailed`，也没有 `TurnEnded`。MCP 收敛照常继续，供后面的轮次使用。
  - MCP 等候超时先提交（会话 MCP 刷新 D4）：`SessionTurnFailed{mcp_refresh_wait_timeout}` 与目标命令的 Failed 已在
    同一事务落账。之后生效的 `CancelTurn` 得到 `already_ended`，不改写这次失败，不补 `TurnEnded`；超时之前已受理、
    之后才生效的 `CancelTurn` 也是如此。
  - 准入先提交：这一轮已经开始，取消按"已准入"或"正在执行"处理，MCP 等候超时不再适用于它。

  初始轮次没有命令可结算，这三种结果对它同样互斥，各至多记录一次。等候已满 300 秒而超时任务还没有提交的轮次，见差异表。
- **一次停止只有一个截止时刻**：等待 Agent 回应的截止时刻在一次停止开始时确定，之后不延长。
  - `CancelTurn` 发起的停止：截止时刻是这个命令的受理时刻加 G。
  - 一轮失败发起的停止：Agent 已经回应，没有回应截止时刻，只有静默窗口。
  - 看门狗发起的停止：先按运行时既有的看门狗宽限期。第一个并入的 `CancelTurn` 把截止时刻改为它的受理时刻加 G，
    只改这一次；看门狗已经发出的 `session/cancel` 就是这次停止的取消，看门狗在两次发送之间时不再发出下一次。
  - 之后并入的每个 `CancelTurn` 共用已经确定的截止时刻，不重新起算。第二个命令在第一个命令的 G 即将到期时到达，
    两者仍按第一个命令的截止时刻一起结算，没有命令越过自己的期限。
- 同一次停止并入的多个 `CancelTurn`，各自的 `TurnCancelSettled` 按受理顺序紧接在这一轮的 `TurnEnded` 之后写入。
- `TurnEnded` 在 Agent 回应这次 prompt、或截止时刻到期时写入，早于 D2 的判定。所以"停止中"包含 `TurnEnded`
  已经写入之后的静默窗口，Controller 可能先收到 `TurnEnded`，之后才收到 `TurnCancelSettled`。
- `CancelTurn` 只作用于 `turn_id` 那一轮：不丢弃其他排队轮次，不结束会话；被取消的一轮结束后按受理顺序
  执行下一轮。排在目标轮次前面的轮次照常执行，取消排队轮次的结论不等它们结束，所以前一轮运行多久都不影响
  这次取消的本地结算时间。
- `TurnCancelSettled` 与 Thread 事件、`SessionTurnFailed` 共享执行内的序号空间：持久化后才发送，确认前保留，
  重连后按原序号重放，`EventAck` 确认精确的 `(execution_id, sequence)`，同样受在途窗口约束（D4）。它不是 Thread
  记录，不写入会话 JSONL，也不进入实时预览。
- Controller 只在目标 `SubmitUserTurn` 已被受理之后发送 `CancelTurn`；目标尚未受理（包括被
  `mcp_update_required` 拒绝、还没有重试成功）时结论是 `unknown`，Node 不为它保留取消。
- 会话已终态时回复 `SessionCommandRejected{session_ended}`。会话没有选择 `keep_session_on_cancel`（D5）时
  回复 `SessionCommandRejected{turn_cancel_unsupported}`，不执行，也不改变会话。

为什么以准入为界：准入是运行时已有的提交点，准入与 MCP 等候超时在这里已经互斥，取消加入同一个边界。准入之前这一轮
没有任何记录，丢弃它不需要 Agent 配合，结论也不取决于会话接入或 MCP 收敛要多久；准入之后这一轮已有记录，必须有
`TurnEnded`。为什么结论不用终态表示：终态要等停止证据（D3），正常时在数秒内，证据缺席时可能始终没有；把结论绑在
终态上，取消就可能没有结论。`session_ending` 在 Node 作出决定时写入，Controller 据此结束对这次取消的等待，再按
既有规则等终态。为什么带 `turn_id`：用户点停止时，这一轮可能恰好自然结束、下一轮随即开始。"取消当前轮"会落在
用户刚提交的下一轮上；按 `turn_id` 定位，这种竞争只会得到 `already_ended`。为什么每个分支都有显式结论：Controller
不能从"没有收到 `TurnEnded`"推断目标是在排队、已结束还是不存在；没有显式结论，这些分支都会让 Controller 一直等到
自己的超时。为什么停止中的重复取消要并入：此时这一轮还没有确定结果，回 `already_ended` 会先于第一个取消宣称
"已结束"，而两者最终可能一起得到 `session_ending`。为什么并入不重新起算：重新起算会让先到的命令被后到的命令拖过
自己的期限。为什么排队轮次直接丢弃：轮到时再记 `TurnEnded` 会让结论时间取决于前一轮跑多久；丢弃与 `EndSession`
丢弃排队轮次的语义一致，Controller 以 `not_started` 把这一轮标为丢弃。

### D2：一轮以取消或失败结束后会话保留的条件

一轮以下列方式结束时，会话先要选择了对应的那一项，再同时满足"可依赖的轮次边界"和三条静止条件，才保留，
回到"等待下一轮或结束命令"。没有选择对应一项时，会话按本决策之前的规则结束：取消没有对应的旧规则，因为没有选择
`keep_session_on_cancel` 的会话不受理 `CancelTurn`；失败以 `agent_failed` 和既有 detail 结束，并入这次停止的
`CancelTurn` 得到 `session_ending`。prompt 从未发给 Agent 的轮次不在此列，按 D1 直接保留：

| 这一轮怎么结束 | 归类 | 要求的选择 | 记录 |
| --- | --- | --- | --- |
| `CancelTurn`，Agent 在截止时刻前回应了这次 prompt | 取消 | `keep_session_on_cancel` | `TurnEnded{cancelled}` |
| 看门狗取消期间并入了 `CancelTurn`，Agent 在截止时刻前回应 | 取消 | `keep_session_on_cancel` | `TurnEnded{cancelled}` |
| ACP `session/prompt` 返回错误（模型请求失败、限流、key 失效、要求重新认证、Agent 内部错误） | 失败 | `keep_session_on_failure` | `TurnEnded{cancelled}` |
| 无进展看门狗取消后 Agent 回应了这次 prompt，但不再重发（回应不是 `cancelled`，或重发次数用尽），期间没有并入 `CancelTurn` | 失败 | `keep_session_on_failure` | `TurnEnded{cancelled}` |

**轮次流量与会话级通知**：本节的条件只针对轮次流量，即会写入一轮内容的消息：正文、思考、用户消息片段、
工具调用及其更新、计划、权限请求，以及带子代理来源的更新；Node 不认识的更新类型按轮次流量处理。会话级通知
（`session_info_update`、`available_commands_update`、`current_mode_update`、`config_option_update`、
`usage_update`）和 Agent 对 Node 自己发出的请求的回应不属于任何一轮：Agent 会在两轮之间正常发送它们（例如
标题和用量），运行时按既有规则处理，它们不影响轮次边界、静止条件和跨轮次检测。

**可依赖的轮次边界**：静止条件只能观察 Agent 发来的流量，挡不住静默窗口之后才到达、又没有身份的更新（例如
正文片段）。所以 Node 只对已验证具备轮次边界的 Agent 保留会话：Agent 回应一次 prompt 之后，不再为这一轮发出
任何轮次流量，也没有在回应后继续运行、却不报告的工作。这是上游 Agent 实现的属性，清单按能唯一确定它的版本
登记（插件锁定上游版本时，即插件的精确版本），由该版本的一致性测试证明（"验证义务"第 13 条），Node 只对
清单中的版本启用会话保留。不在清单中的 Agent，这一轮的取消或失败一律走兜底分支（D3，detail
`turn_boundary_unverified`），即使会话选择了新行为。

按 OpenCode（`@opencode/cli` 2.0.32）与 Claude Code（`@agentclientprotocol/claude-agent-acp` 0.107.0）的源码，
两者的主轮次都具备这一边界：OpenCode 每一轮只转发自己投递之后的事件，Claude Code 丢弃被中断工作的后续输出。
两者都不保证子代理和后台任务的边界：OpenCode 的子代理在取消后继续运行，Claude Code 的后台子代理在取消后
继续运行并可能发出权限请求。所以静止条件 2 排除启动过子代理的轮次。两者能否进入清单，以一致性测试为准；
清单的现行内容与登记规则见轮次进程栅栏决策 D5。

**静止条件**，Node 在发出下一轮之前逐条检查，任何一条不满足就按 D3 结束会话（detail `turn_not_quiescent`）：

1. **回应栅栏**：Agent 回应了这次 prompt（结果或错误）。ACP 要求 Agent 在中止进行中的操作、发完待发的更新
   之后才回应取消，回应是这一轮主流程结束的栅栏。
2. **没有未终态或无法收尾的工作**：回应时，这一轮的每个工具调用都已有终态（`completed` 或 `failed`），且这一轮
   没有启动过子代理投影。子代理可能在父轮次结束后继续运行，其终态不能由父 prompt 结束推断，首期子代理投影也
   没有子代理终态的来源。这是第一版的已知限制：用过子代理的轮次（长任务中常见）取消或失败后，会话一律结束。
3. **静默窗口**：回应之后的 Q 时间内，这个会话没有收到任何轮次流量。会话级通知不计入。

**跨轮次检测**：一轮以取消或失败结束并保留会话后，在这个会话其后的所有轮次中，若收到引用这一轮工具调用
身份或子代理来源的更新，Node 认定这一轮的流量跨进了后面的轮次：该更新不进入任何一轮的 Thread 事件与实时预览（会话 JSONL 例外，见差异表），按 D3 结束会话
（detail `turn_leak_detected`）。静止条件 2 已经保证这些工具调用在回应时都有终态，所以之后任何引用它们的更新
都是违约。这是对具备身份的更新的兜底检测；没有身份的更新只能依靠轮次边界清单排除。一轮正常完成之后的迟到
更新不在本决策内，沿用运行时既有规则：空闲期到达的更新被丢弃，不结束会话。

为什么这样分层：ACP 的更新不带请求身份。上一轮 A 回应后、下一轮 B 发出前到达的更新，运行时会丢弃；B 的
prompt 发出之后才到达的 A 的无身份更新，与 B 的更新在协议上无法区分，任何静默窗口都只能降低概率，不能兑现
隔离。可依赖的轮次边界把"回应之后不再有 A 的流量"
变成对插件版本的可测试承诺；静止条件在每一次结束时检查 Agent 是否真的遵守了它；跨轮次检测拦截有身份的违约。
三者任何一个不成立，Node 都不再向这个 Agent 派发下一轮，按 D3 结束会话，由停止证据而不是 Agent 的承诺来约束
上一轮仍可能在运行的工作。

**时间参数**：

- **G（取消宽限期）**：`CancelTurn` 等待 Agent 回应的上限，自发起这次停止的 `CancelTurn` 受理时起算，并入的命令
  不重新起算（D1）。G 覆盖所支持 Agent 正常取消的
  最坏耗时，而不是 Agent 卡死时的兜底：

  | Agent | 正常取消的最坏耗时 | 卡死时的兜底 |
  | --- | --- | --- |
  | OpenCode | 中断后自己最多等 5 秒，shell 工具再过 3 秒强制终止，约 8 秒 | 同左 |
  | Claude Code | 中断通常很快 | SDK 无响应时 30 秒后强制回 `cancelled`，此时 provider 会话状态本身可疑 |

  G 暂定 15 秒，没有取消耗时的分布实测；共享运行时原有的 5 秒短于 OpenCode 的正常取消耗时，
  不能沿用。G 只是上限，Agent 回应后立即结束等待。看门狗取消的宽限期不随之加长，仍按运行时既有规则，以免
  模型无输出时的重发被拖慢；看门狗取消没有得到回应时同样按 D3 结束会话。看门狗取消期间并入的第一个 `CancelTurn` 把截止时刻改为它的受理
  时刻加 G，之后不再改变（D1）。
- **Q（静默窗口）**：Q 暂定 2 秒，没有残余流量的分布实测。它只用于发现违约，不是隔离的依据。只在一轮
  以取消或失败结束后检查，一轮正常完成后不等待，不影响正常对话的延迟。取消后立刻提交下一轮是最常见的用法，
  Q 直接加在这次等待上，所以应取能发现残余轮次流量的最小值；以后的实测没有残余轮次流量时可以取 0。
- 取得停止证据的等待不属于取消的期限：它从 Node 决定结束会话时开始，上限是会话 MCP 刷新 D5 的停止确认期限，
  只影响终态，不影响 `TurnCancelSettled`（D3、D4）。

**续用方式**：满足上述条件后，下一轮在同一个会话执行、同一个 Ora 会话记录上进行。provider 会话仍可用时直接
续用；运行时已隔离它时，按运行时既有规则 `session/load` 恢复，或开新的 provider 会话并交接 Ora 的记录。
选择了 MCP 生命周期模式的会话只 `session/load` 原 provider ID，失败时 blocked，不新建（会话 MCP 刷新 D4），这次
load 计入每个 Agent 进程的运行时 load 次数（会话 MCP 刷新 D6）。ACP 错误
回应不等于 provider 会话仍然可用：Agent 可能在报错的同时关闭了这个 provider 会话（之后的 prompt 回"会话已结束"
或"会话不存在"），Node 把这类回应视为 provider 会话已失效，下一轮先恢复（允许时重建），未选择 MCP 生命周期模式的会话恢复和重建
都失败时按 D3 结束会话；生命周期模式的会话按会话 MCP 刷新 D4 进入 blocked。会话的模型授权、插件租约和工作区在会话结束前一直保留，与一轮成功完成后相同。

失败的一轮与取消的一轮在 Thread 里都记为 `TurnEnded{cancelled}`，本决策不区分两者（见 D6）。

### D3：必须结束会话的情况

**Node 结束会话**的路径沿用[会话 MCP 刷新与宿主健康](../agent-runtime/20261116-session-mcp-refresh-and-host-health.md)
D5，本决策不修改它，选择了新行为的会话同样适用：

1. Node 决定结束会话时，先停止准入，不再向这个 Agent 派发任何一轮，再请求停止这个执行的 Agent 与探测。
   停止请求的返回只用于诊断，不是停止证据。
2. 只有本执行全部 Agent 与探测责任都取得停止证据（原尝试锁与全部直接进程线索，进程组 BestEffort），Node 才
   写唯一的 `AgentSessionEnded`。
3. 停止确认期限（默认 10 分钟）内取不到证据时，Node 登记 `stop_unconfirmed`，`GetExecutionStatus` 返回
   `StopUnconfirmed{scope_id, run_id, first_stop_at_ms, bound_at_ms, guarantee}`，转交残留对账。之后取得原证据才补发
   终态，原因与 detail 是最初决定结束时的那一个；终态也可能始终缺席。

D3 的情况与本决策的取消结论之间的关系：

- **结束决定与结论同一事务落账**：Node 作出结束决定时，在一个事务中写入这份结束决定，再在事务提交之后发停止
  信号。事务的内容是：
  - 首次终态原因与 detail；
  - 停止时钟 `first_stop_at_ms` 与 `bound_at_ms`（会话 MCP 刷新 D5 的锚定，跨重启不续期）；
  - 所有尚未结算的 `CancelTurn` 的 `TurnCancelSettled{session_ending}`。
  所以任何一条 `session_ending` 可见时，结束决定都已经能从账本唯一恢复。决定之后、终态之前受理的 `CancelTurn`，
  在受理的同一事务中得到 `session_ending`（D1）。
- **重启后沿用结束决定**：Node 在结束决定落账之后重启，包括结论已写入或已投递、停止信号还没发出：
  - 重启收口读到这份决定就按它继续，不再写取消结论，每个 `command_id` 仍只有原来那一条；
  - 不把会话改记为 `interrupted`，终态的原因、detail 和停止时钟都用原来的；
  - 仍要取得原停止证据才写终态。
  只有重启时还没有结束决定的会话，才按协议 D6 以 `interrupted` 结束。
- 取消的结论在决定结束时就已写入，不等停止证据；终态什么时候到、到不到，只影响 Controller 对会话的结算，不影响
  这次取消。

| 情况 | 终态 |
| --- | --- |
| Agent 进程退出、崩溃，或 Node 与 Agent 之间的 ACP 连接断开 | `agent_failed`，detail 为既有的 `agent_unavailable` 或 `agent_stream_closed` |
| 取消或看门狗取消后，截止时刻前 Agent 没有回应这次 prompt | `agent_failed`，detail `turn_cancel_unanswered` |
| 一轮以取消或失败结束，而 Agent 插件版本不在轮次边界清单中 | `agent_failed`，detail `turn_boundary_unverified` |
| 一轮以取消或失败结束后，D2 的静止条件不成立 | `agent_failed`，detail `turn_not_quiescent` |
| 其后的轮次中收到可归属某个被取消或失败轮次的更新 | `agent_failed`，detail `turn_leak_detected` |
| 运行时不再受理新的一轮（会话记录已降级、轮次未被受理、provider 会话恢复失败） | `agent_failed`，detail 为既有代码 |
| Thread 镜像不可用 | `agent_failed`，detail `thread_unavailable` |
| Node 账本读写失败，包括命令结算失败 | `agent_failed`，detail `ledger_unavailable` |
| 模型授权被撤销 | `agent_failed`，detail `model_access_revoked` |
| `EndSession` | 命令给出的原因 |
| Node 重启 | 重启前没有结束决定时为 `interrupted`（协议 D6）；已有结束决定时沿用它。有未确认的停止责任时，取得原停止证据后才写终态（会话 MCP 刷新 D5） |

MCP 门禁中的情况（等候超时、刷新失败、blocked）只让对应轮次失败或等待，不结束会话（会话 MCP 刷新 D4），不在
此表中。

**Controller 结束会话**：Controller 一侧的故障由 Controller 判断，需要时发 `EndSession`，或按 Controller 自己的
决策把会话当作结束，本决策不改变这些规则。包括：与 Node 的连接断开且不再恢复、事件在 Controller 一侧持久化
失败、Node 回复 `SessionCommandRejected`、Node 回复的身份与会话对不上、以及 D4 中 Node 未在期限内结算。

没有选择 `keep_session_on_failure` 的会话，D2 中的失败仍以 `agent_failed` 结束会话，与本决策之前一致。

**为什么兜底是安全的**：Node 决定结束会话后不再向 Agent 派发任何一轮，下一轮不可能与上一轮混在同一个 provider
会话里。上一轮仍在运行的工具对 checkout 的影响由停止责任约束：终态只在取得停止证据后写入；取不到证据时没有终态，
残留门禁挡住这个执行自己的交付和带 `prior_revision` 的恢复（会话 MCP 刷新 D5）。停下之前 Agent 已经做出的修改留在
checkout 中，取得终态后由 Revision 交付照常保存。停止证据是进程组 BestEffort，不是整棵树静止的证明，这一边界与
会话 MCP 刷新 D5、D8 相同，本决策不加强也不削弱它。不确定的只是这一轮里没有收到终态的工具调用：它们在记录中
保持 `pending` 或 `in_progress`，由 `TurnEnded{cancelled}` 说明这一轮被打断，不被记成成功。代价是用户的对话结束，
而不是工作区与记录不一致。

### D4：先后关系、本地结算与投递

- **`EndSession` 优先**：队列中任意位置的 `EndSession` 仍优先于其前面的命令。`EndSession` 生效时，尚未结算的
  `CancelTurn` 在同一事务中得到 `session_ending`；被丢弃的排队轮次照常结算为 Discarded，不再单独得到 `not_started`。
  终态之后到达的 `CancelTurn` 回复 `session_ended`。已持久受理的 `EndSession` 与一轮失败同时发生时，会话以
  `EndSession` 的原因结束（沿用当前实现）。
- **先到者为准**：`CancelTurn` 生效前这一轮已经有了自己的结果（D1"已有结果"），结论为 `already_ended`，不改写那个
  结果。生效时这一轮仍在停止中，按 D1 并入那一次停止。准入之前的竞争见 D1。
- **工具调用与权限请求**：取消前先以 `cancelled` 应答这一轮挂起的权限请求。Agent 回应之前发出的更新都记入这
  一轮；回应之后到达的轮次流量不记入任何一轮，并按 D2 让会话走兜底分支，会话级通知照常处理；这一轮结束时
  仍未终态的工具调用保持原状态，并使静止条件不成立。
- **事件顺序**：正在执行的一轮被取消时，执行内的事件顺序为：这一轮的 Thread 事件 → `TurnEnded` → 并入这次停止
  的各个 `TurnCancelSettled` → 会话保留时是下一轮的 Thread 事件，会话结束时是取得停止证据后的终态。终态之后没有
  事件。
- **本地结算期限**：Node 持续运行时，自受理 `CancelTurn` 起，在 G + Q 内把它的 `TurnCancelSettled` 持久化到账本。
  这一期限只取决于 Node 与 Agent，与连接、在途窗口和停止证据都无关。这一期限以账本可写为前提。Node 重启时，
  已有结束决定的会话按 D3 沿用它；没有结束决定的会话，在重启收口中写入以 `interrupted` 为原因的结束决定，同一事务
  为未结算的 `CancelTurn` 写 `session_ending`。Node 不运行的时间不计入期限。
- **账本持续不可写**：`CancelTurn` 已受理、之后的账本写入失败时：
  - **Node**：不把内存状态当作结算。写不进账本的 `TurnCancelSettled`、命令结算和结束决定，既不发送，也不当作已经
    发生。Node 立即停止准入，不再写出任何 `session/prompt`，按 D3 的 `ledger_unavailable` 结束会话。停止信号照常
    发出，因为停止总是安全的；但没有落账的停止不产生终态，也不解除停止责任。账本不可写期间，新命令无法持久受理，
    不回复 `SessionCommandAccepted`。
  - **Controller**：收不到结论，按下一条的等待规则超过 T 后，判定 Node 未在期限内结算。它把这次停止当作未确认：
    不当作已停，不交付，不恢复，不再向这个会话提交轮次；可以查询 `GetExecutionStatus`，按自己的故障规则收尾。
  - **恢复**：账本恢复可写时（或之后 Node 重启时），Node 在写任何终态之前先补齐。先写结束决定：已有落账的决定就
    沿用；没有就以 `ledger_unavailable` 为原因，Node 重启后则是 `interrupted`。同一事务为每个已受理、还没有
    `TurnCancelSettled` 的 `CancelTurn` 写一条 `session_ending`，每个 `command_id` 至多一条，已有结论的不再写。之后
    终态仍要等停止证据。账本一直不可写时，结论和终态都不会出现，由 Controller 按上面的规则收尾。
- **投递**：结论与其他事件一样按序号投递，受在途窗口约束：前序事件未被确认、窗口已满时，结论在账本中等待，
  确认释放窗口后才发送；断线时在重连后按序重放。所以 Controller 收到结论的时间等于本地结算时间加上它确认
  前序事件所用的时间，后者由 Controller 自己的确认进度决定。
- **Controller 等待取消结论**：Controller 按投递进展而不是固定时长等待结论。等待期间，只要连接仍在、并且持续收到
  这个执行的事件，就继续确认并等待，不判为超时。只有连接仍在、Controller 已确认它收到的全部事件、且自受理与
  最后一次确认两者中较晚的时刻起超过 T 仍没有收到任何新事件时，才判定 Node 未在期限内结算，按 Controller 自己的
  故障规则处理（例如发 `EndSession`，并把这次停止视为未确认）。T 必须大于 G + Q 加上传输余量；T 更短时，Controller
  会在 Node 仍在正常等待 Agent 回应时下结论。连接断开时按 Controller 的连接断开规则处理，重连后从重放的事件重新
  开始计算。Controller 可以在收到结论之前提交下一轮，下一轮按受理顺序排在被取消的一轮之后执行。
- **Controller 等待终态**：收到 `session_ending` 后，Controller 不再等待这次取消，改为按会话 MCP 刷新 D5 等待终态：
  终态到达前会话仍未结束，停止可能还在进行；可以用 `GetExecutionStatus` 查询，看到 `StopUnconfirmed` 时按残留责任
  处理，按自己的业务窗口收尾。`session_ending` 和等待到期都不是停止证明，Controller 不能据此把 checkout 当作已经
  静止，也不能据此交付或恢复。

### D5：能力协商、逐会话选择与接管

- Controller 在 `Hello` 中以 `turn_cancel: true` 声明它理解 `CancelTurn`、`TurnCancelSettled`、新的拒绝原因与终态
  detail；旧 Controller 省略该字段。Node 支持、且这次 `Hello` 声明了 `turn_cancel` 时，Node 才在
  `HelloAccepted.capabilities` 中声明 `turn_cancel`，并在 `HelloAccepted` 中回 `turn_cancel: true`，这次连接才算
  协商成功，与实时预览的协商形式相同。否则 `HelloAccepted` 不含这项能力和确认字段，与本决策之前的内容相同。
  原因是 Controller 的解码器不一定容忍未知能力：desktop `crates/node-protocol` 的 `NodeCapability` 是封闭枚举，
  使用它的旧 Controller 遇到未知能力值会握手失败。
- `AgentSessionSpec` 增加两个可选字段，省略即 `false`：`keep_session_on_cancel: bool` 与
  `keep_session_on_failure: bool`。Controller 只在当前连接协商成功时设置为 `true`；它们和其他输入一样先持久化，同一
  执行重发时参与输入比较。下文"选择了新行为的会话"指两个字段中至少一个为 `true` 的会话。
- `keep_session_on_cancel` 为 `true`：接受 `CancelTurn`，按 D1–D4 处理，被取消的一轮满足 D2 时会话保留。为 `false`：
  拒绝 `CancelTurn`（`turn_cancel_unsupported`）。
- `keep_session_on_failure` 为 `true`：一轮失败（D2 的归类）且满足 D2 时会话保留。为 `false`：一轮失败仍以
  `agent_failed` 和既有 detail 结束会话。
- 两个字段都为 `false` 的会话，不会产生 `TurnCancelSettled` 或新的 detail，行为与本决策之前相同。
- **启用顺序**：第一版的 Controller 只设置 `keep_session_on_cancel`，不设置 `keep_session_on_failure`。失败后保留
  要等失败原因上报的后续决策落地（D6），调用方能从协议看到失败和原因（包括需要重新认证）之后，才启用。Node 第一版
  就支持两个字段，启用顺序由 Controller 遵守。
- **接管边界**：只要 Node 上还有选择了新行为、且尚未终态或仍有未确认事件的会话执行，Node 就拒绝没有协商
  `turn_cancel` 的连接：在 `HelloAccepted` 之前以 `4403`（能力不符）结束这次连接，不投递、不重放任何事件，也不
  受理任何命令。这些执行在 Node 上照常进行并持久化事件，等到具备能力的 Controller 连接后，从最小的未确认序号
  起按序重放。所以已选择新行为的会话不会被不理解其事件的 Controller 接管。
- **拒绝的持续时间**：close code 决策把 `4403` 归为重连不会自行恢复的配置错误，旧 Controller 仍按 `reconnect_ms`
  重连，在拒绝解除前每次都被拒。拒绝在这些会话终态、且事件全部被确认后解除。事件只能由具备能力的 Controller
  确认；终态还可能因停止证据缺席而始终不到（D3）。所以只剩旧 Controller 时，拒绝不会自行解除，出路只有前向恢复：
  让具备能力的 Controller 连接这个 Node，确认事件、按 D3 处理残留责任。
- **部署前提**：可能连接这个 Node 的全部 Controller 实例都具备 `turn_cancel` 之后，Controller 才为新会话选择新行为。
  滚动升级期间新旧实例并存时不选择，否则旧实例重连就会被拒绝。
- **回退条件**：把可能连接这个 Node 的 Controller 回退到不具备 `turn_cancel` 的版本之前，这个 Node 上每个选择了新
  行为的会话执行都必须已终态、停止责任已解除（没有 `stop_unconfirmed`）、事件全部被确认。有执行排不空时不回退，
  只做前向恢复：由具备能力的 Controller 接管，确认事件、处理残留责任。
- 旧 Node 不声明 `turn_cancel`，且 `StartAgentSession` 使用 `deny_unknown_fields`：Controller 不能向它发送新字段
  或 `CancelTurn`，停止仍用 `EndSession`。旧 Controller 不设置新字段：Node 对它启动的会话保持原行为。
- 新命令与新事件写入 Node 账本；需要时账本 schema 前向迁移，旧程序拒绝打开新库（沿用 Node 升级规则）。

为什么按会话选择，而不是只在握手里打开：会话比连接活得久，重连可能连到另一个 Controller 版本。选择写进执行输入
后，同一会话从开始到结束只有一套规则；握手协商只决定当前连接能否发出选择、能否接管已选择的会话。为什么只向声明了
`turn_cancel` 的 Controller 返回这项能力：会话 MCP 刷新 D3 要求新消息随 Controller 与 Node 同版本发布，但不选择新
行为的旧 Controller 不应因为 Node 升级就连不上；只有握手内容不变，"不选择就保持原行为"才对握手本身也成立。为什么在
连接层拒绝而不是只扣住这些执行的事件：扣住事件时，旧 Controller 仍会为这些会话发命令，却永远等不到结果，也无法区分
"Node 扣住了"与"Node 出故障了"；以 `4403` 拒绝连接，归类明确，且不改变任何执行的状态。

### D6：与失败原因上报的边界

本决策只规定一轮以取消或失败结束时会话是否保留，以及调用方如何观察这一轮和这次取消的结束。它不区分"取消"
和"失败"，也不携带失败原因：两者在 Thread 中都是 `TurnEnded{cancelled}`。Controller 可以用"这一轮是否收到了
`TurnCancelSettled{stopped}`"判断这是不是自己请求的取消，但不能据此认定其余的结束都是失败，也拿不到原因。
区分取消与失败、在协议和历史记录中携带失败原因、看门狗取消后 Agent 回 `end_turn` 时是重发还是以"无进展"
失败，都由后续决策（[circus#1138](https://github.com/ora-space/circus/issues/1138)）承担；它们建立在本决策的会话保留之上。

失败后保留有一个缺口，所以它的启用要等后续决策（D5 启用顺序）。没有选择 `keep_session_on_failure` 的会话，一轮
失败时以 `agent_failed` 和它的 detail（例如 `agent_authentication_required`、`agent_turn_failed`）结束，调用方能
看到这次失败。选择了它并保留会话后，这一轮只有 `TurnEnded{cancelled}`：调用方看不出这是失败，也拿不到"需要重新
认证"。取消后保留没有这个缺口：Controller 以 `TurnCancelSettled` 知道这是自己请求的取消，而一轮失败时，只选择了
`keep_session_on_cancel` 的会话仍按原规则结束并带 detail。

## 实现与批准文本的差异

| 批准文本 | 实际采用 | 原因 |
| --- | --- | --- |
| D2 跨轮次检测、不变量 8：可归属上一轮的迟到更新“不记入任何一轮” | 该更新不进入 Thread 事件和实时预览，会话以 `turn_leak_detected` 结束，但它已经写入会话 JSONL | 运行时先写会话记录，Node 才能拦截；要在写入之前拦截须改变共享运行时的记录顺序，由 [circus#1414](https://github.com/ora-space/circus/issues/1414) 决定 |
| D1、D4“先到者为准” | 写出 `session/cancel` 之前先取已经排队的回应：成功回应保留原停止原因，取消得到 `already_ended`；ACP 请求错误按失败走 D2，保留会话时返回 `already_ended`，否则返回 `session_ending`。写出取消后，在截止时刻前收到的成功回应（含 `end_turn`）或 ACP 请求错误（`RequestFailed`）都按 `cancelled` 走 D2 | 取消写出之后，Agent 的回应不能证明这一轮已经静止，仍须通过 D2 的保留判定 |
| D1“排队：还没有被准入” | 包含准入之前的整个准备阶段：恢复 prior revision、布置工作负载、模型授权期间，取消立即结算，准备照常继续 | 这些步骤耗时没有上限，等它们结束会超过 G + Q |
| D1 取消与 MCP 等候超时“以先提交的为准” | 等候已满 300 秒、超时任务还没有提交时，取消先按等候超时结算，准入也不能提交，无论这一轮在队首还是排在后面。排队轮次的取消另有一处计时偏差：目标已有 MCP 等候记录、前序轮次仍在执行时，取消检查会把前序轮次的业务运行时间算进去，可能提前写入 `mcp_refresh_wait_timeout` 并返回 `already_ended`，按会话 MCP 刷新 D4 应为 `not_started`；由 [circus#1417](https://github.com/ora-space/circus/issues/1417) 修复 | 到期检查防止取消或准入绕过会话 MCP 刷新 D4 的上限。前序轮次的业务时间在它退出时才从预算中扣除，所以它运行期间的取消检查可能提前判定到期 |
| D2 续用方式、D3 第一行“Agent 进程退出” | MCP 生命周期模式下达到 load 次数阈值而替换 Agent 进程，以及替换失败，都按会话 MCP 刷新 D6 处理，不结束会话 | 替换由 Node 主动发起，保留同一执行与原 provider ID，不是 Agent 退出 |
| D2“把这类回应视为 provider 会话已失效” | 按一组固定的错误文字精确匹配（不区分大小写）；带会话 ID 的文字匹配不上 | ACP 没有为此规定错误码；匹配不上时下一轮的 prompt 会再次失败 |
| D3 模型授权被撤销 | 授权进入 Ending 时，并行请求停止 Agent 并等待 `EndSession` 提供结束原因。等待上限为 30 秒，有未结算的取消时也不得超过最早一条的受理时刻加 G；到期仍无 `EndSession` 时以 `model_access_revoked` 落账 | 结束决定落账后不能改写（D3），取消必须按期结算，停机清理不阻塞落账 |
| D3 重启收口 | 重启时先写完整的结束决定与取消结论，只有终态等停止证据 | 结论不等停止证据（D4） |
| D5 未协商的连接 | 未协商 `turn_cancel` 的连接发 `CancelTurn`，或启动任一保留字段为 `true` 的会话，按协议违规关闭连接，不进入持久受理 | 否则它之后会收到不理解的事件，违反不变量 10 |

## 不变量

1. 一个被受理的 `SubmitUserTurn` 恰好以下列之一结束：带其 `turn_id` 的一个 `TurnEnded`；被 `CancelTurn` 丢弃，命令
   结算为 Discarded 并有 `TurnCancelSettled{not_started}`；MCP 等候超时，以 `SessionTurnFailed` 与命令 Failed 结算；
   会话先结束时按 `EndSession` 的既有规则被丢弃。准入之前，取消、准入与等候超时三者至多一个生效。
2. 每个被受理的 `CancelTurn` 恰好得到一条 `TurnCancelSettled`。Node 持续运行且账本可写时，它在受理后 G + Q 内
   持久化，不等停止证据，也不等终态；它在任何终态之前写入。账本不可写时，没有落账的状态不被当作结论，恢复后在
   终态之前补齐，每个命令至多一条。
3. 结论按执行内的序号投递，在前序事件被确认、窗口释放后送达；连接仍在时，只要 Controller 持续确认，结论一定到达。
4. 并入同一次停止的多个 `CancelTurn` 得到与这次停止一致的结论，共用一个截止时刻；在这一轮停止中时，没有
   `CancelTurn` 得到 `already_ended`，除非这一轮因失败而停。
5. `CancelTurn` 只影响 `turn_id` 那一轮，不丢弃其他轮次；它导致会话结束只经由 D3 的兜底分支。
6. `EndSession` 优先于 `CancelTurn`；会话已终态后命令不复活会话。
7. 会话在一轮取消（或失败）后保留，当且仅当会话选择了 `keep_session_on_cancel`（或 `keep_session_on_failure`），并且：这一轮的 prompt 从未写给 Agent；或者 Agent
   版本在轮次边界清单中、Agent 连接仍在、D2 的三条静止条件都成立。
8. 同一会话内，上一轮满足 D2 之前，Node 不向 Agent 发送下一轮；回应之后到达的上一轮轮次流量不进入任何一轮的 Thread 事件与实时预览（会话 JSONL 例外，见差异表），
   其后的轮次中收到可归属被取消或失败轮次的更新时会话结束。
9. `TurnCancelSettled{session_ending}` 写入之后，Node 不再向这个会话的 Agent 写出任何 `session/prompt`。
   `AgentSessionEnded` 只在本执行全部 Agent 与探测责任取得停止证据之后写入（会话 MCP 刷新 D5）；`session_ending`
   不是停止证据。
10. 已选择新行为、尚未终态或仍有未确认事件的会话，不会把事件投递给没有协商 `turn_cancel` 的 Controller；没有声明
    `turn_cancel` 的 Controller 收到的 `HelloAccepted` 不含这项能力。
11. 两个字段都没有选择的会话，行为与本决策之前完全一致；只选择了 `keep_session_on_cancel` 的会话，一轮失败时的行为与本决策之前一致。
12. `SessionCommandAccepted` 不代表取消已有结论；结论只以 `TurnCancelSettled` 为准，会话是否结束只以终态为准。
13. 会话级通知不改变任何一轮的结论，也不导致会话结束。
14. 被取消的一轮，在 `CancelTurn` 生效之后不再有新的 `session/prompt` 发给 Agent：prompt 尚未写出时不写出，
    看门狗不再重发。
15. 任何 `TurnCancelSettled{session_ending}` 可见时，对应的结束决定（首次终态原因、detail 与停止时钟）已经落账；
    重启不改变它，也不重复产生取消结论。

## 为什么不是这些替代方案

| 方案 | 取舍 |
| --- | --- |
| `EndSession` 增加"只取消一轮"的原因 | 不新增消息；但 `EndSession` 的规则是丢弃排队轮次并产生终态，一个命令两种语义会让"终态之后没有事件"和命令幂等的判断都依赖原因字段 |
| `CancelTurn` 不带 `turn_id`，取消"当前"一轮 | Controller 更简单；但停止与这一轮自然结束、下一轮开始存在竞争，会取消用户刚提交的下一轮（D1） |
| 会话结束时以终态作为取消的结论 | 少一种结果；但终态要等停止证据，证据缺席时终态可能始终不到，已受理的取消就没有结论，Controller 也无法区分"还在停"和"Node 出故障"（D1、D3） |
| 取消的期限包含停止 Agent 的耗时 K | 结论就是"已经停下"；但停止证据的等待上限是分钟级，且可能取不到证据，取消的期限因此失去意义（D4） |
| 并入的取消各自从受理时起算 G | 每个命令看起来都有完整的宽限期；但先到的命令会被后到的命令拖过自己的期限（D1） |
| 等 MCP 门禁放行、prompt 准入之后再处理取消 | 只有一种"已开始"状态；但结论时间取决于会话接入与 MCP 收敛（最长 300 秒），等候超时还可能先让这一轮失败（D1） |
| 向所有 Controller 声明 `turn_cancel` 能力 | 与其他能力的声明方式一致；但使用封闭枚举的旧 Controller 会握手失败，不选择新行为也连不上（D5） |
| 不新增结论事件，只用 `TurnEnded` 表示取消完成 | 少一种事件；但排队、已结束和未知三个分支没有 `TurnEnded` 或要等前一轮结束才有，Controller 只能等到超时（D1） |
| 排队轮次轮到时再记 `TurnEnded{cancelled}` | Thread 保留这条用户消息；但结论时间取决于前一轮跑多久，可能超过 Controller 的等待上限 |
| 停止中的重复取消立即回 `already_ended` | 实现简单；但这一轮还没有确定结果，两次请求最终可能一起以会话终态收场，第二次的结论与事实不符（D1） |
| 结论绕过在途窗口优先发送 | Controller 更早知道结论；但破坏执行内序号连续、结论必须在 `TurnEnded` 之后的顺序，重放时还要处理越序事件 |
| Controller 按固定时长 T 等待 | 规则简单；但窗口背压下 Node 已按时结算，Controller 仍会等满 T，把投递延迟误判为停止失败（D4） |
| 只以 Agent 的回应为准，不设清单、静默窗口与跨轮次检测（共享运行时在本机会话上的现有做法） | 规则最简单，且已在使用；但它发现不了回应之后才到达的轮次流量。本决策在回应之上加了三层检查，代价是规则更复杂、更多会话在取消后被结束。轮次边界清单是启用条件，实测不能省去它，因为没有身份的更新只能靠它排除；静默窗口 Q 按实测定值，可以为 0 |
| 静默窗口内收到任何消息都算违约 | 不需要区分消息类型；但 Agent 在两轮之间正常发送标题、用量等会话级通知，本可续用的会话会被结束（D2） |
| 只靠静默窗口隔离迟到更新，加长 Q | 不依赖 Agent 的承诺；但静默窗口之后到达的无身份更新仍无法区分，Q 只能降低概率，并拖慢每一次取消后的下一轮 |
| 每次取消或失败后都重建 provider 会话，靠会话 ID 区分迟到更新 | 迟到更新可以按 provider 会话 ID 丢弃；但每次都失去原 provider 上下文，且无法阻止上一轮的工具继续修改 checkout |
| 失败后一律保留会话，不协商 | 不改输入；但依赖"失败即终态"的调用方会看到会话停在空闲直到超时，并一直占用租约和模型授权，这正是当前实现改为失败即结束的原因 |
| 只在握手中打开新行为，不写入会话输入 | 与实时预览一致；但会话跨连接存在，重连到不同版本的 Controller 会让同一会话前后规则不同（D5） |
| 旧 Controller 接管时，只扣住已选择新行为的执行的事件 | 不影响同一 Node 上的其他执行；但旧 Controller 仍会为这些会话发命令、永远等不到结果，也无法与 Node 故障区分（D5） |
| 取消没有得到 Agent 回应、或静止条件不成立时也保留会话 | 用户少遇到一次对话结束；但上一轮可能仍在执行，下一轮与它并发修改同一 checkout，迟到的更新也会混进下一轮 |
| 兜底时只重启 Agent 进程、保留会话：取得停止证据后，在同一执行内启动新的 Agent 进程并 `session/load` 原 provider ID | 卡死时对话也能保住；会话 MCP 刷新 D6 已经为包升级定义了同一执行内重启自己的 Agent、load 原 provider ID、跨重启保留模型授权与 ora-git 的路径，可以复用。但与"Agent 进程退出即结束会话"的规则相反，停止证据缺席时下一轮要一直等，Agent 不支持跨进程 load 时会话 blocked，内存中的上下文也会丢失。第一版不采用，可作为后续决策 |
| 宽限期取 30 秒，覆盖 Claude Code 卡死时的强制回应 | 卡死的 Claude Code 也能保留会话；但为一个 provider 会话本身已可疑的情况让用户多等，并迫使每个 Controller 把等待上限放宽到 30 秒以上，卡死时用户要等更久才知道对话已结束 |
| 失败后结束会话，由 Controller 新开会话并重放历史 | Node 不变；但同一 Agent 进程仍可用时要重建上下文，成本高，并把会话连续性的责任推给每个 Controller |

## 已知风险与待实测

按 OpenCode（`@opencode/cli` 2.0.32）与 Claude Code（`@agentclientprotocol/claude-agent-acp` 0.107.0）的 ACP
实现核对：取消、模型请求错误、要求重新认证、看门狗取消后回 `end_turn`，以及取消时有进行中的工具调用或挂起的
权限请求，这些情况之后两者都能在同一个 ACP 会话上继续，上下文保留。

实现阶段的真实 Agent 实测用的是部署的发布包 OpenCode 0.6.11（CLI 1.19.3）和 Claude 0.4.5（ACP 0.99.1），不是上述两个
源码核对版本，结论只对实测的版本成立：

- 运行中的普通 shell、前台子代理、模型错误三个场景，没有观察到回应之后的轮次流量或写入，两者都能接下一轮；
- 后台 shell 在回应之后仍会写工作目录、且不再有任何轮次流量，两个包都不能只凭本决策的三层检查登记。轮次进程栅栏
  决策据此改写轮次边界，并登记 Linux 上的 OpenCode 0.6.11；
- 取消得到回应的耗时为数十到数百毫秒，只是少量观察，没有时间分布，所以 G 与 Q 仍是暂定值；
- 没有实测：OpenCode 内部异常后 server 端是否仍在执行、Claude Code 内部或传输错误后 provider 会话被关闭、被取消
  轮次中用过子代理的比例。OpenCode 与子代理比例由 [circus#1415](https://github.com/ora-space/circus/issues/1415) 承接，Claude 由 [circus#1381](https://github.com/ora-space/circus/issues/1381) 承接。

下表是批准时列出的风险与现行处理：

| 风险 | 影响 | 处理 |
| --- | --- | --- |
| 主轮次回应之后仍有无身份的更新到达 | 静默窗口之后到达时会混进下一轮 | 由轮次边界一致性测试判定，现行登记规则见轮次进程栅栏决策 D5；未登记的版本取消与失败一律走兜底（D2） |
| OpenCode 的子代理、Claude Code 的后台任务可能在回应之后继续运行，并发出轮次之外的更新和权限请求 | 启动过子代理的轮次由静止条件 2 排除；不报告的后台任务 Node 无法观察 | 已实测成立。栅栏范围内的后台进程由 Node 在保留会话前停掉，范围之外的仍由一致性测试排除（轮次进程栅栏决策） |
| OpenCode 内部异常时，ACP 层返回错误，但 server 端的执行可能仍在进行 | 下一轮混入上一轮未停的工作 | 尚未实测，由 [circus#1415](https://github.com/ora-space/circus/issues/1415) 承接；成立时把这类错误归入兜底分支。取消写出后到达的同类错误按取消判定，所以只选择了 `keep_session_on_cancel` 的会话也受影响；启用失败后保留之前必须有结论（“落地顺序”第 4 步）；取消后保留由 owner 接受这一风险先行启用（“落地顺序”第 3 步） |
| Claude Code 内部或传输错误后关闭 provider 会话，Agent 进程仍在 | 下一轮发到已失效的会话 | 按 D2 识别后 `session/load` 恢复；真实 Agent 上的实测由 [circus#1381](https://github.com/ora-space/circus/issues/1381) 承接 |
| G 短于 Agent 正常取消的耗时，或 Q 短于 Agent 回应后残余流量的间隔 | 本可续用的会话被结束，或违约未被静默窗口发现 | G 15 秒、Q 2 秒是暂定值，没有分布实测；OpenCode 的耗时与残余流量分布由 [circus#1415](https://github.com/ora-space/circus/issues/1415) 承接，Claude 的由 [circus#1381](https://github.com/ora-space/circus/issues/1381) 承接 |
| 一轮失败后保留会话，会话存在的时间变长 | 同一 checkout 承载多个会话时，会更久地占用 checkout 的会话名额 | 由 Controller 的空闲超时与 `EndSession` 释放；checkout 名额的规则不在本决策内 |
| 降级 Controller 时仍有已选择新行为的会话 | 该 Node 拒绝旧 Controller 的所有连接，同一 Node 上的其他执行也无人接管；这些会话终态后仍有未确认事件、或终态因停止证据缺席而始终不到时，拒绝不会自行解除 | 全部 Controller 实例具备能力后才打开选择（D5）；`4403` 让拒绝原因可诊断；只能前向恢复；回退前按 D5 的回退条件排空 |
| 取消后会话结束、停止证据缺席 | 取消已有 `session_ending`，但会话可能始终没有终态，这个执行不能交付，checkout 不能带 `prior_revision` 恢复 | 沿用会话 MCP 刷新 D5 的残留对账与 R9；本决策不加强 BestEffort 停止保证 |
| 用过子代理的轮次取消或失败后会话一律结束 | 长任务常用子代理，这些轮次得不到会话保留，功能的实际覆盖面可能明显小于预期 | 比例的统计由 [circus#1415](https://github.com/ora-space/circus/issues/1415) 承接；放宽需要子代理终态的来源，由后续决策承担 |
| 一轮失败后保留会话，调用方没有失败信号 | 选择了 `keep_session_on_failure` 的会话失败时，用户看不到失败，也看不到需要重新认证（D6） | 第一版不启用 `keep_session_on_failure`，等失败原因上报（[circus#1138](https://github.com/ora-space/circus/issues/1138)）落地后再启用（D5） |

## 验证义务

已逐条转为[核心用例](../../../test-cases/node/protocol/cancel-turn-and-keep-session.md)，证据状态以用例为准。10、13 的现行文字以
轮次进程栅栏决策为准：

1. 取消正在执行的一轮后，同一会话可以接着 `SubmitUserTurn`，上下文保留；事件顺序为 `TurnEnded` →
   `TurnCancelSettled{stopped}` → 下一轮。
2. 取消排队中的一轮时，前一轮运行超过 Controller 的等待上限 T，`TurnCancelSettled{not_started}` 仍在受理后立即
   写入，目标命令结算为 Discarded，被取消的轮次从未发给 Agent，前一轮不受影响。
3. 取消正常完成的轮次、取消已结束且已判定保留会话的轮次、取消不存在的轮次，分别立即得到 `already_ended`、
   `already_ended` 与 `unknown`。
4. 取消与这一轮自然结束竞争时，`TurnEnded` 保留 Agent 自己的停止原因，结论为 `already_ended`，用户随后提交的下一轮
   不被取消。
5. 同一轮的两个 `CancelTurn`：第二个分别在 G 内（仍在等待回应）和 Q 内（仍在静默窗口）受理时，两者得到相同的结论；
   这次停止以兜底结束时，两者都得到 `session_ending`；没有一个得到 `already_ended`。
6. 一轮失败后、静默窗口内受理的 `CancelTurn`，在判定后得到 `already_ended`，或随兜底得到 `session_ending`。
7. Agent 回应后在静默窗口内仍发来上一轮的工具更新：这些更新不记入任何一轮，取消得到 `session_ending`，会话以
   `turn_not_quiescent` 结束，下一轮从未发给 Agent。
8. 静默窗口已过、下一轮已开始后，才到达引用上一轮工具调用身份的更新：它不进入任何一轮的 Thread 事件与实时预览（会话 JSONL 例外，见差异表），会话以 `turn_leak_detected`
   结束。
9. 回应时仍有未终态的工具调用，或这一轮启动过子代理投影：会话以 `turn_not_quiescent` 结束。
10. 插件版本不在轮次边界清单中时，一轮取消或失败后会话以 `turn_boundary_unverified` 结束。
11. G 内没有回应：受理后 G + Q 内已写入 `TurnEnded{cancelled}` 与 `TurnCancelSettled{session_ending}`，之后不再写出任何
    `session/prompt`；Agent 取得停止证据之后才写入 `AgentSessionEnded{turn_cancel_unanswered}`。
12. **停止证据始终缺席**：G 内没有回应，停止请求返回（成功或失败）之后原尝试锁仍被持有或直接进程仍在。验证：
    `TurnCancelSettled{session_ending}` 按期写入；停止确认期限内和期限到达后都没有 `AgentSessionEnded`；期限到达后
    `GetExecutionStatus` 返回 `StopUnconfirmed`；这个执行自己的交付被拒绝（`session_not_settled`），带 `prior_revision`
    的恢复被拒绝；之后对账取得原证据，才补发终态，detail 仍是 `turn_cancel_unanswered`，之后没有事件。
13. 轮次边界一致性测试（按清单登记的版本）：对真实 Agent 构造取消运行中的 shell 工具、取消带子代理和后台任务的
    轮次、模型请求错误，记录 prompt 回应之后到达的全部流量；没有任何轮次流量才进入清单。
14. 在途窗口已满（256 个未确认事件）时受理 `CancelTurn`：结论在本地期限内写入账本；Controller 释放确认后按序收到
    结论；Controller 持续收到并确认事件期间不判为超时。
15. Controller 已确认全部事件、超过 T 仍无新事件时，判定 Node 未在期限内结算。
16. 新 Controller 启动选择新行为的会话，在确认 `TurnCancelSettled` 前断开；旧 Controller 重连时被以 `4403` 拒绝，
    没有收到任何事件；具备能力的 Controller 随后连接，从最小未确认序号起按序收到全部事件。
17. **旧 Controller 的实际握手**：用本决策之前版本的 desktop `ora-node-protocol` 解码新 Node 回给旧 `Hello`（不含
    `turn_cancel`）的 `HelloAccepted`，以及 cloud v2-server 的握手解码，两者都接受；Node 上没有选择新行为的会话时，
    旧 Controller 正常连接、收发命令与事件。声明了 `turn_cancel` 的 `Hello` 才收到这项能力和 `turn_cancel: true`。
18. 选择了 `keep_session_on_failure` 的会话，ACP prompt 报错、看门狗放弃后满足 D2 时，会话保留；报错后 provider 会话已失效时，下一轮经恢复继续；MCP 生命周期
    模式的会话只 load 原 provider ID，失败时 blocked。
19. D3 中每一种情况仍结束会话，Node 作出结束决定时尚未结算的 `CancelTurn` 都得到 `session_ending`。
20. 两个字段都没有选择的会话：`CancelTurn` 被拒绝，一轮失败仍以 `agent_failed` 结束，行为与之前一致；旧 Node 与旧
    Controller 的消息、握手与输入编解码不变。
21. `EndSession` 与 `CancelTurn` 同时排队时按 `EndSession` 结束，`CancelTurn` 得到 `session_ending`，早于终态。
22. 静默窗口内只收到会话级通知（标题、用量、可用命令）时会话保留，这些通知不记入任何一轮。
23. **MCP 门禁中的轮次**：目标轮次已到队首、正在等 MCP 门禁或会话接入。
    - 取消先提交：`TurnCancelSettled{not_started}`，目标命令 Discarded，之后到 300 秒也没有 `SessionTurnFailed`，没有
      `TurnEnded`；MCP 收敛继续，下一轮在门禁放行后照常准入。
    - 等候超时先提交：`SessionTurnFailed{mcp_refresh_wait_timeout}` 与 Failed 已落账，之后生效的 `CancelTurn` 得到
      `already_ended`，不改写失败。
    - 等候已满 300 秒、超时任务还没有提交时生效的取消（目标在队首，或排在队首之后）：先按等候超时落账，取消得到
      `already_ended`；这时准入也不能提交。
    - 准入先提交：取消按"已准入"或"正在执行"处理。
    - 初始轮次（没有命令）同样覆盖三种顺序，各至多一个结果。
24. 准入已提交、`session/prompt` 还没有写出时受理 `CancelTurn`：Agent 从未收到这次 prompt，事件为
    `TurnEnded{cancelled}` → `TurnCancelSettled{stopped}`，会话保留，与轮次边界清单无关。
25. **看门狗并入与 G 的临界点**：看门狗取消期间受理第一个 `CancelTurn`（A），截止时刻为 A 的受理时刻加 G；A 的 G
    即将到期时受理第二个 `CancelTurn`（B），B 不延长截止时刻。Agent 恰在截止时刻之前回应时两者都得到 `stopped`；
    恰在之后回应时两者都得到 `session_ending`，且 A 的结论不晚于 A 受理后 G + Q。这一轮都不再重发。
26. 被取消的一轮之后隔了一轮以上，才到达引用它的工具调用身份的更新：会话仍以 `turn_leak_detected` 结束。一轮
    正常完成后，空闲期到达的更新不结束会话。
27. Node 在 `CancelTurn` 受理之后、结论写入之前重启：重启收口中这个命令得到 `session_ending`，早于 `interrupted`
    终态；有未确认停止责任时，终态在取得原证据后才写入。
28. **结论可见之后重启**：G 内没有回应，`TurnCancelSettled{session_ending}` 已写入（分别覆盖已投递、未投递），停止
    信号还没有发出时 Node 重启。验证：
    - 重启后不重复产生取消结论；
    - 终态原因不是 `interrupted`，仍是 `turn_cancel_unanswered`；
    - 停止时钟沿用原来的 `first_stop_at_ms` 与 `bound_at_ms`，不续期；
    - 取得原停止证据之后才写终态。
29. **受理之后账本持续不可写**：`CancelTurn` 成功受理后，注入账本持续写失败。验证：
    - Node 不发送任何没有落账的结论，不再写出 `session/prompt`；
    - Controller 超过 T 后按未确认收尾，不把超时或内存状态当成已结算；
    - 账本恢复后，在终态之前为这个命令补写恰好一条 `session_ending`，结束原因为 `ledger_unavailable`；
    - 恢复前已有结论的命令不重复写。
30. **两个字段的组合**：
    - 只选择 `keep_session_on_cancel`：取消后满足 D2 时会话保留；一轮失败时以 `agent_failed` 和既有 detail 结束，
      并入的 `CancelTurn` 得到 `session_ending`；
    - 只选择 `keep_session_on_failure`：`CancelTurn` 被拒绝（`turn_cancel_unsupported`），一轮失败后满足 D2 时会话保留；
    - 两者都选择：两种结束分别按对应规则；
    - 看门狗取消期间并入了 `CancelTurn` 的一轮，按取消处理。

## 本决策未解决的问题

- 区分取消与失败、上报失败原因，以及看门狗取消后 Agent 回 `end_turn` 的处理（D6），由 [circus#1138](https://github.com/ora-space/circus/issues/1138) 承接。
- 在写入会话 JSONL 之前拦截跨轮次泄漏的更新，需要改变共享运行时的记录顺序，由 [circus#1414](https://github.com/ora-space/circus/issues/1414) 承接。
- Agent 进程退出、兜底结束会话之后的会话恢复，以及 Node 重启后的会话恢复（协议 D6）。
- 轮次边界清单由插件自己声明（例如写入插件的 Agent 声明）还是由 Node 维护；第一版由 Node 维护，随一致性测试更新。
- 用过子代理的轮次取消或失败后保留会话，需要子代理终态的来源。
- 各 Controller 如何展示被取消或失败的一轮、何时自动发 `EndSession`。

## 落地顺序

1. **协议与账本**：`crates/node-protocol` 增加 `CancelTurn` 消息、`TurnCancelSettled` 事件、`Hello`／`HelloAccepted`
   的 `turn_cancel`、能力 `turn_cancel`、`keep_session_on_cancel` 与 `keep_session_on_failure` 字段、拒绝原因 `turn_cancel_unsupported` 与新的
   终态 detail，补校验与往返测试；`crates/node-db` 受理并持久化新命令与新事件。完成条件：旧形状的消息、握手与
   输入编解码不变，新字段省略时输入比较与之前相同。
2. **共享运行时、Node 会话执行与接管**：`crates/agent-runtime` 区分用户取消与看门狗取消的宽限期，在准入提交处
   和看门狗重发前认取消，识别已失效的 provider 会话；`apps/ora-node/src/session/` 按 D1–D4 执行 `CancelTurn`，
   取消与准入、MCP 等候超时共用准入提交的锁；按 D2 检查轮次边界清单、静止条件与跨轮次泄漏，按 D3 区分一轮失败与
   必须结束会话的情况，结束会话沿用已有的停止责任路径；连接层按 D5 只向声明了能力的 Controller 返回能力，并拒绝
   不具备能力的 Controller 接管。
   第 1、2 步已落地。批准时的完成条件是“验证义务”每一项都有直接测试、“已知风险与待实测”每一项都有真实 Agent 的
   实测结论；两者没有全部满足，缺口逐条记在核心用例和“已知风险与待实测”中。
3. **Controller 接入**（不属于 Node 侧实现，由各 Controller 落地）：各 Controller 在握手中声明 `turn_cancel`；可能连接 Node 的全部 Controller 实例都具备能力
   后，才为新会话选择 `keep_session_on_cancel`，停止改用 `CancelTurn`，按 D4 的进展规则等待结论，收到 `session_ending` 后按停止责任
   等待终态。
   上文 OpenCode 内部异常一行的风险不作为本步的前置条件：owner 接受这一风险，先行启用取消后保留。依据是 v2 在真实 Node 上的
   实测（[circus#1404](https://github.com/ora-space/circus/issues/1404)）：OpenCode 0.6.11 在取消开始后 12–24 ms 内中止这一轮的模型请求，
   晚于取消送达的文本与工具调用都不会进入被保留的会话或下一轮。内部异常这条更窄的路径仍由
   [circus#1415](https://github.com/ora-space/circus/issues/1415) 实测，成立时归入兜底分支。

4. **启用失败后保留**（尚未启用）：前置条件是失败原因上报的后续决策（[circus#1138](https://github.com/ora-space/circus/issues/1138)）落地、Controller 能展示失败和原因，
   并且 OpenCode 内部异常后的执行状态已实测（[circus#1415](https://github.com/ora-space/circus/issues/1415)），必要时归入兜底。都满足之后，才为新会话选择 `keep_session_on_failure`。

保持不变：已有消息、已有会话、两个字段都没有选择的会话，以及没有结束决定时 Node 重启后的 `interrupted`。从第 3 步
开始改变：选择了 `keep_session_on_cancel` 的会话在一轮被取消、且满足 D2 后保留；第 4 步之后，一轮失败也可以保留。
