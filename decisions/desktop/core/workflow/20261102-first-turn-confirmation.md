---
status: implemented
---

# 交互式节点首轮停靠：首次自动回合需显式确认，异常停靠与等待参与分开显示

> 历史决策：有限 `wait` 到期时，交互节点的自动首轮失败，人类回合按 `prompt_timed_out` 异常停靠。见[已实施的工作流等待策略](20261123-workflow-agent-wait-limit.md)。

交互式 Agent 节点目前只要本轮以 `EndTurn` 结束且 blockers 为空就视为 ready；但 agent 也可能用 `EndTurn` 结束一个「向用户提问」的回合，界面于是同时显示「等待参与」和「完成并进入下一步」，点击后跳过用户回答。本决策给 workflow 自动发起的首轮 park 增加 `turn_unconfirmed` blocker：首轮正常结束也不再自动 ready，用户通过新增的 `confirmInitialTurn` 请求字段完成一次普通推进，不记 `completion_forced`。`force` 与 `completion_forced` 只用于覆盖非确认类 blocker；同时把停靠路径上的超时、截断、拒答、未知、会话崩溃等异常停靠与用户主动停止、agent 等待确认分开显示，异常停靠默认不可无确认推进，只能继续或显式强制完成。

本决策为 `implemented`。实现随 [desktop#751](https://github.com/ora-space/desktop/pull/751) 合并（merge `12c29e51`），按批准方案落地；D1、D4 已按最终实现更正，偏离见 D4 末段「与批准版本的偏离」。人类回合的静默超时/断流/崩溃已有 park 级集成测试（真实 history + SQLite + `end_human_turn`），归因链路已由 `a_workflow_turn_records_the_reported_end_cause_in_its_parked_snapshot` 与 `a_provider_crash_reports_session_crashed_rather_than_a_dropped_stream` 覆盖：runtime 的 `PromptEndReport` 经 `StopCause` 写入 park 快照。人类回合三类异常 cause 与自动交互式首轮失败已由 [desktop#763](https://github.com/ora-space/desktop/pull/763) 的跨 crate e2e 覆盖：人类回合经 `Sessions::prompt → WorkflowSessionTurns::prompt` 写入异常快照，自动首轮经 `drive_agent_node` 失败且不产生停靠快照；`stream_dropped` 指 owning consumer 放弃流，与 provider 连接丢失的 `session_crashed` 分开。具体测试边界见核心用例；旧快照兼容与 ready 时 force 保持已有直接回归证据。它修改[交互式节点完成契约](20261021-interactive-node-completion.md)（下称 #385 ADR）的 D1、不变量 1 与 D7，并扩展 D5 的请求字段/错误优先级以及不变量 9、12 的审计表述；其余决策与不变量继承。影响面包括 `crates/contracts` 的完成请求与 blocker kind、backend 的 park/完成路径、`packages/workflow-runtime` 的完成投影，以及 app-shell 的步骤卡片、右下角按钮和 Theater act card 徽标；数据库只读写既有 `payload` 列，不改 schema、冻结图字段或本地文件布局。正常人类回合的 `EndTurn` 完成行为、非交互节点、Cloud 与 `Loop`/`Iteration` 区域内的 interactive 行为保持不变。升级前已停靠且缺少新字段的快照不追溯加 `turn_unconfirmed`；旧客户端不带 `confirmInitialTurn` 时无法完成首轮停靠，必须升级。核心测试用例见[首轮确认与异常停靠用例](../../../../test-cases/desktop/core/workflow/first-turn-confirmation.md)。

## 继承与修改

继承[节点运行时编排决策](0-node-runtime-orchestration.md)、[迭代节点](20260915-iteration-composite-runtime.md)、[prompt 静默策略](20261008-prompt-inactivity-policy.md)、[while-Loop](20261013-while-loop-with-durable-round-scopes.md)与[交互式节点产出物声明](20261023-interactive-deliverable-contract.md)。

| 前序约定 | 处理 | 当前规则 |
| --- | --- | --- |
| #385 D1：只有本轮 `EndTurn` 且无 blocker 才能完成；交互式节点对所有 terminal stop reason 停靠 | 修改 | 增加「workflow 自动发起首轮」维度：首轮 `EndTurn` 且没有更精确 blocker 时写 `turn_unconfirmed`，不自动 ready；人类回合仍是 `EndTurn` + 无 blocker 即 ready（D1） |
| #385 D2：用 `{ userSeq, endedSeq }` 唯一标识本轮，最终文本取 `(attemptStart, endedSeq]` | 继承，不变 | 普通确认与 force 仍先比对 turn key 与 `outputDigest`；不新增回合身份或文本来源 |
| #385 D3：首轮与人类 cleanup 共用同一个 park 函数，快照与 `Running → Pending` 同事务提交 | 继承，不变 | 仍是同一个函数、同一个 `Immediate` guarded UPDATE；快照增加 `turnOrigin` / `stopCause` / `parkKind` 三个字段，origin 由调用点传入（D1、D4、D7） |
| #385 D4：人类 cleanup 先等 actor 收敛，超时写 provisional，`prepare` 修复 | 继承，不变 | provisional 只由人类回合产生；修复保留原 `turnOrigin` / `stopCause`，按修复后的 blockers 重算 `parkKind`，不补 `turn_unconfirmed` |
| #385 D5：`completeWorkflowNode` 按状态 → turn key → digest → blockers 的优先级校验，`force` 只覆盖 blockers | 继承，并扩展 | 原有 1–4 顺序不变；增加请求形状检查与确认分支；`force` 不再覆盖 `turn_unconfirmed`，它由 `confirmInitialTurn` 处理（D2、D3） |
| #385 D6：会话消息不被拦截，每个 workflow 人类回合注入步骤边界 | 继承，不变 | 确认和强制都是界面动作，不新增或改写任何用户消息；首轮自动回合仍沿用既有 handoff |
| #385 D7：当前步骤卡片与右下角按钮两个显式入口推进，徽标区分「等待参与」与「已暂停」 | 修改 | 拆成 `awaiting_user` 与 `abnormal` 两态；首轮确认走卡片，右下角按钮只对 `ready` 启用；异常停靠显示原因且只能继续或 force（D4、D5） |
| #385 D8：`Pending` 跨重启保留，provider 恢复由 runtime 处理 | 继承，不变 | 确认状态随 `payload.completion` 保留；不新增恢复路径、自动续算或重跑本步骤 |
| #385 不变量 1：正常完成只认本轮 `EndTurn` | 修改 | 人类回合仍是 `EndTurn` + 无 blocker；自动首轮还要求 `confirmInitialTurn` 才能完成，`force` 不能覆盖 `turn_unconfirmed`（D1、D2） |
| #385 不变量 9、12：错误优先级固定；`force` 必须二次确认并留痕 | 继承，并扩展 | 增加请求形状检查；`force` 的覆盖范围限定为非确认类 blocker；审计增加 `stop_cause`，区分异常来源（D2、D3） |
| #385 其余不变量 | 继承，不变 | 单事务 park、投影不读历史、`completeWorkflowNode` 唯一提交权威、区域边界等均不变 |
| 产出物声明 D2–D4：文件 blockers 在 `prepare` 按现场 workspace 判定，沿用 `WorkflowNodeIncomplete` 与 `force` | 继承，并澄清 | `confirmInitialTurn` 不覆盖文件 blockers；补齐文件后重试确认即可；强制完成仍走 #426 的 force 语义，审计的 `blockers` 记录实际覆盖的文件 blocker |
| 静默策略 D1–D4：`timeout` / `wait` 冻结在节点配置，`timeout` 才可能取消 prompt | 继承，不变 | 本决策只记录「这次停靠是否由 prompt 静默超时造成」，不改变策略、窗口、重试或 `wait` 语义 |
| 区域与 Cloud 边界 | 继承，不变 | `Loop`/`Iteration` 区域内的 interactive 不在本决策范围；Cloud 不执行 interactive Agent，无对应发布门禁变更 |

## 决策

### D1：workflow 自动发起的首轮以 `turn_unconfirmed` 停靠，人类回合维持原规则

park 函数新增一个必填的 `turnOrigin` 输入（`stopCause` 按 D4 同时传入），由调用点提供，而不是从历史推断：

- executor 的自动首轮停靠传 `workflow`；
- 人类回合 cleanup 传 `human`。

快照把 `turnOrigin` 写入 `payload.completion`。自动首轮只有在同时满足下列条件时，才追加 `turn_unconfirmed`：

1. `turnOrigin` 是 `workflow`；
2. 本轮以 `EndTurn` 结束；
3. `HistoryIntegrity` 为 `Complete`；
4. 输出契约（`None` / `Text` / `Structured`）没有产生 blocker；
5. 当前 blockers 为空。

只要已经存在更精确的 blocker，就只保留精确的那个，**不叠加** `turn_unconfirmed`。精确 blocker 指 `turn_stopped`、`turn_truncated`、`turn_refused`、`turn_unknown`、`structured_invalid`、`output_missing`。

首轮自动回合运行期间节点是 `Running`，人类 prompt 不会被 admission 接受；一个 park 的 origin 由 owning prompt 决定，不存在首轮与人类回合交错的情况。

| 自动首轮结果 | blockers | `parkKind` | 完成入口 |
| --- | --- | --- | --- |
| `EndTurn`，历史完整，输出契约通过 | `[turn_unconfirmed]` | `awaiting_user` | 仅普通确认 |
| `EndTurn`，`Text` 空输出 | `[output_missing]` | `abnormal` | 继续或 force |
| `EndTurn`，`Structured` 校验失败 | `[structured_invalid]` | `abnormal` | 继续或 force |
| 用户主动停止（`Cancelled`） | `[turn_stopped]` | `awaiting_user` | 继续或 force |
| prompt 静默超时耗尽、会话崩溃（prompt 流返回错误）、流结束但没有 stop reason | 不停靠 | — | 节点失败，走既有失败/重试语义（见 D4） |
| `MaxTokens` / `MaxTurnRequests` | `[turn_truncated]` | `abnormal` | 继续或 force |
| `Refusal` | `[turn_refused]` | `abnormal` | 继续或 force |
| 未知 stop reason、无 `TurnEnded`、历史不可读、历史完整性缺口（均非 provisional） | `[turn_unknown]` | `abnormal` | 继续或 force |

人类回合的用户主动停止与 prompt 静默超时的区分依赖 `stopCause`。新快照必须记录这个原因；无法判定时按 D4 保守归入 `abnormal`，不能默认成用户主动停止。自动首轮的静默超时不进入停靠，也不产生快照。

`turn_unconfirmed` 是 blocker，`ready` 仍按「blockers 为空且非 provisional」推导，因此首轮正常结束也是 ready = false。人类回复之后的回合仍按 #385 的规则：`EndTurn` + 无 blocker 即 ready，正常显示「完成并进入下一步」。首轮 gate 是**每个 node run 的一次性 gate**：用户回复后，新的快照 `turnOrigin` 为 `human`，旧的首轮快照被替换，gate 不再适用；用户也可以从卡片继续当前步骤，同样会开启人类回合。

provisional 快照只由人类回合的 cleanup 产生，自动首轮不写 provisional：首轮要么在收到 `Completed` 后以非 provisional 快照停靠，要么直接让节点失败。provisional 修复保留快照的 `turnOrigin`（`human`），不补 `turn_unconfirmed`；修复出超时、截断等只保留对应 blocker。

选择只拦首轮，是因为首轮是唯一一个用户没有参与发起的回合：workflow 自己发出 prompt，agent 的「回答」与「反问」在没有显式完成信号时不可区分。把每次正常完成都变成确认会显著增加摩擦，也会和 #427 的 agent 自报完成重复。首轮确认的代价只是一次显式点击，却能把原来的一次点击跳过变成一个有明确语义的决定。

本决策不解析文本、不判断 agent 是否在提问，也不调用模型判定。确认只是把「是否带着当前产出推进」的决定交给用户；用户仍可以在确认后跳过 agent 的提问，这是显式选择，不是自动误判。

### D2：新增 `confirmInitialTurn` 请求字段，只解除 `turn_unconfirmed`

`CompleteWorkflowNodeRequest` 增加可选布尔字段：

```json
{
  "runId": "run-1",
  "nodeId": "agent",
  "confirmInitialTurn": true
}
```

- 缺省与 `false` 表示未确认；只有显式 `true` 才解除 `turn_unconfirmed`。
- `force` 与 `confirmInitialTurn` 不能同时为 `true`：这是自相矛盾的请求，返回现有 `invalid_request`，不读也不改任何状态。
- `confirmInitialTurn` 只解除 `turn_unconfirmed`。状态/claim、turn key、`outputDigest`、输出契约、#426 的文件 blockers 以及其他任何 blocker 都不被它越过。
- 快照只有 `turn_unconfirmed` 且没有 `confirmInitialTurn` 时，返回 `WorkflowNodeIncomplete { blockers: [turn_unconfirmed] }`。这是默认 gate，老客户端不带字段时也会得到这个结果，而不是静默推进。
- 快照只有 `turn_unconfirmed` 却发送 `force: true` 时，仍返回 `WorkflowNodeIncomplete` 并列出 `turn_unconfirmed`；`force` 不是确认，不能把首轮正常结束记成强制完成。
- 快照没有 blockers 时，`confirmInitialTurn` 是幂等的无操作，按普通完成处理；不写 `completion_forced`。
- 快照含非 `turn_unconfirmed` 的 blocker 时，`confirmInitialTurn` 不能完成，返回 `WorkflowNodeIncomplete`；错误只列出请求尚未满足的 blocker，已经被确认解除的 `turn_unconfirmed` 不再出现在列表里。
- 如果修复后的快照变成超时、截断等异常 blocker，确认请求得到同一个 `WorkflowNodeIncomplete`，用户仍需继续或 force。

provisional 快照只属于人类回合，先按 #385 D4 的既有路径修复，再按上表判定；修复后是普通的人类回合，`confirmInitialTurn` 不能跳过修复，也不会把一个尚未收敛的回合直接当成已确认。

校验顺序固定为：

判定使用的**有效 blocker 集合** = 快照 blockers + `prepare` 复算出的输出契约/文件 blockers；下表第 5–10 行都基于这个集合，`confirmInitialTurn` 不能越过集合里的任何非 `turn_unconfirmed` 项。

| 顺序 | 检查 | 结果 |
| --- | --- | --- |
| 1 | `force && confirmInitialTurn` | `invalid_request`，不写任何状态 |
| 2 | run 不是 `Running`、节点不是 `Pending`、completion claim 不持有 | `workflow_node_not_awaiting_input`；两个字段都不能越过 |
| 3 | turn key 与快照不一致 | `WorkflowNodeTurnChanged`；前端刷新运行后重试 |
| 4 | `outputDigest` 与快照不一致 | `WorkflowNodeOutputChanged`；前端刷新运行后重试 |
| 5 | 有效 blocker 集合为空 | 普通完成，不写 `completion_forced` |
| 6 | 仅有 `turn_unconfirmed` 且 `confirmInitialTurn` | 普通完成，不写 `completion_forced` |
| 7 | 仅有 `turn_unconfirmed` 且 `force` 或没有确认 | `WorkflowNodeIncomplete`，列出 `turn_unconfirmed` |
| 8 | blockers 只含非 `turn_unconfirmed` 的 blocker 且 `force` | 强制完成，写 `completion_forced` |
| 9 | blockers 含 `turn_unconfirmed` 与任何其他 blocker，或含非 `turn_unconfirmed` 的 blocker 但没有 `force` | `WorkflowNodeIncomplete`，列出请求未满足的 blocker |
| 10 | 输出契约或文件 blockers 在 `prepare` 门外仍失败 | 有 `force` 则强制完成并审计；否则 `WorkflowNodeIncomplete` |

确认与 force 都不改变输出来源：有 turn key 时仍取本轮 `(userSeq, endedSeq]` 窗口的最终 assistant 文本；没有 turn key 的 `turn_unconfirmed` 不会出现（首轮正常结束必有 turn key），因此确认不涉及 `without_turn` 的强制路径。`confirmInitialTurn` 不写入快照，快照仍保留 `turn_unconfirmed`；完成后 `turn_unconfirmed` 只在 payload 中作为「用户看到并确认过的停靠状态」保留，投影在节点离开 `Pending` 后不再使用它。

该字段只用于人类确认 workflow 自己发起的首轮。`NodeCompletionRequester::Agent` 与 agent 自报完成属于 #427，不在本决策范围；后续如果 agent 复用同一命令，必须定义它不能使用 `confirmInitialTurn`，或定义独立语义。

### D3：普通确认不写 `completion_forced`，force 审计记录阻断类型与停靠原因

本决策把除 `turn_unconfirmed` 外的 blocker 统称为**非确认类 blocker**；`force` 只用于覆盖它们。首轮正常结束后的确认是一次**普通完成**，不写 `completion_forced`，也不新增 `completion_confirmed` 之类的第二个审计键。判断方式保持简单：

- 完成后的 payload 仍保留 `completion.turnOrigin = "workflow"`、`completion.blockers = [turn_unconfirmed]`、`turnKey` 与 `outputDigest`；
- 没有 `completion_forced`；
- 因此「已完成 + 快照含 `turn_unconfirmed` + 没有强制审计」就是用户使用显式确认完成的记录。

不写 `completion_confirmed`，是因为普通完成本身已经是持久记录；再加一个确认审计会和 force 的「覆盖 blocker」语义混在一起，也会诱导把用户自由文本写进 payload。若以后需要单独追踪确认动作，应另开决策，而不是在本决策里引入第二种审计。

`force` 只在**实际覆盖至少一个 blocker** 时写审计；`force` 遇到已经 ready 的快照按普通完成处理，不写空审计。审计形状在 #385 的基础上增加 `stop_cause`：

```json
"completion_forced": {
  "blockers": [
    { "kind": "turn_stopped", "params": {} }
  ],
  "stop_cause": "prompt_timed_out",
  "without_turn": false,
  "output_source": "turn_window"
}
```

- `blockers` 是本次完成实际覆盖的有效 blocker 列表：已存快照 blockers，加上 `prepare` 复算出的输出契约/文件 blockers（如有）。它只含结构化字段，不含用户文本。
- `stop_cause` 是停靠原因，取值 `user_stopped`、`prompt_timed_out`、`stream_dropped`、`session_crashed`、`unknown`，不适用时为 `null`。它区分 blocker kind 无法单独表达的异常来源：`turn_stopped` 覆盖用户主动停止、prompt 静默超时，以及已以 `Cancelled` 收敛的会话崩溃/流断开；`turn_unknown` 只用于没有可用 `TurnEnded`、历史不可读或 stop reason 未知的回合。
- `without_turn` / `output_source` 沿用 #385；结构化失败时 force 仍不写 `structured_output`。
- `completion_forced` 与完成仍在同一个事务提交；审计不能后写，也不能只写审计不完成。
- `force` 覆盖用户主动停止的 blocker 时，审计写 `stop_cause: "user_stopped"`。这不是把用户停止归为异常，而是记录「用户选择带着一个未正常结束的回合推进」；界面仍按 `awaiting_user` 显示。

`force` 不能覆盖 `turn_unconfirmed`：如果快照只有它，请求返回 `WorkflowNodeIncomplete`，不产生强制审计。这样 `completion_forced` 只出现在「带着一个被覆盖的阻断原因推进」的场景，首轮正常结束的确认不会污染它。

### D4：停靠投影分 `awaiting_user` 与 `abnormal` 两类，异常必须带原因

快照增加三个字段：

- `turnOrigin`：`"workflow"` 或 `"human"`，由 park 调用点传入；
- `stopCause`：`"user_stopped"`、`"prompt_timed_out"`、`"stream_dropped"`、`"session_crashed"`、`"unknown"` 或 `null`，由 owning prompt 的结束原因传入；
- `parkKind`：`"awaiting_user"` 或 `"abnormal"`，在 park 与 `prepare` 的修复/物化写回时按下面的表计算并写入快照；ready 的快照不写 `parkKind`。

| 停靠情形 | blocker | `stopCause` | `parkKind` |
| --- | --- | --- | --- |
| 人类回合 `EndTurn` 且无 blocker | 无 | `null` | 不写（ready） |
| 自动首轮 `EndTurn` 且无其他 blocker | `turn_unconfirmed` | `null` | `awaiting_user` |
| 用户主动停止 | `turn_stopped` | `user_stopped` | `awaiting_user` |
| prompt 静默超时（回合已以 `Cancelled` 收敛） | `turn_stopped` | `prompt_timed_out` | `abnormal` |
| 输出截断 | `turn_truncated` | `null` | `abnormal` |
| 拒答 | `turn_refused` | `null` | `abnormal` |
| 会话崩溃 / 流断开（回合已以 `Cancelled` 收敛） | `turn_stopped` | `session_crashed` / `stream_dropped` | `abnormal` |
| 未知 stop reason / 无 `TurnEnded` / 历史不可读 / 历史完整性缺口（非 provisional） | `turn_unknown` | `unknown` / 记录到的实际原因 | `abnormal` |
| 结构化输出不符合 | `structured_invalid` | `null` | `abnormal` |
| `Text` 输出缺失 | `output_missing` | `null` | `abnormal` |
| provisional（本轮仍在收尾） | `turn_unknown` | 记录实际原因 | `awaiting_user`（暂时） |

本表只覆盖**停靠路径**。超时、会话崩溃等如果直接让节点失败而不是停靠，走既有节点失败/重试语义，不进入本决策的两态。

`awaiting_user` 覆盖首轮等待确认、用户主动停止与 provisional；正常 ready 的人类回合不写 `parkKind`，界面同样显示「等待参与」。它们的共同点是「需要用户回答、确认或选择」，而不是系统以异常方式结束了本轮。停靠时，`abnormal` 覆盖超时、截断、拒答、未知、会话崩溃、结构化失败和输出缺失；它们共同点是「本轮没有产生一个可以直接推进的正常结果」，界面必须说明原因，默认只能继续或 force。

用户主动停止与 prompt 静默超时必须分开，因为两者都可能以 `Cancelled` 结束、都写 `turn_stopped`。本决策不新增 `turn_timed_out` blocker，也不改变 stop reason 的映射；超时是「停靠原因」，不是 ACP stop reason，由 `stopCause` 表达。这样 blocker 的语义保持稳定，审计还能区分「用户自己停的」与「系统等超时了」。静默策略本身不变：`wait` 不会产生超时，只有节点配置为 `timeout` 且窗口到期时才可能记录 `prompt_timed_out`。

上表的 blocker 按停靠路径有两种来源。

人类回合由 cleanup（`end_human_turn`）停靠。活跃回合被系统结束时，runtime 会先把本回合写成 `TurnEnded { stop_reason: Cancelled }`；只要这份 boundary 在 cleanup 的等待窗口内收敛，静默超时、流断开和会话崩溃的 park blocker 就是 `turn_stopped`，再由 `stopCause` 区分具体来源。等待窗口内拿不到 `TurnEnded` 时写 `provisional: true` 且带 `turn_not_settled` 的快照，历史读不出来时写带 `history_unavailable` 的 provisional 快照，两者的 `turn_unknown` 都因 provisional 暂归 `awaiting_user`；`settle_snapshot` 在新推导的 `turnKey` 为空时保留原快照，不把这次未收敛当成新的异常 blocker。

`stream_dropped` 表示 owning consumer 放弃 prompt 事件流；provider 连接丢失归为 `session_crashed`。人类回合的两条路径都由 runtime 结束归因写入快照；自动首轮的 owning stream 提前结束且没有 `Completed` 时按节点失败处理。代表性跨 crate 证据与 owner EOF 故障注入边界见[异常分类核心用例](../../../../test-cases/desktop/core/workflow/first-turn-confirmation.md#abnormal-parking-is-distinct-from-awaiting-user)。

自动首轮（`park_first_interactive_turn`）只在收到 prompt 的 `Completed` 后停靠。静默超时耗尽和会话崩溃让 prompt 流返回错误，流结束却没有 stop reason 时报 `SessionEndedWithoutStopReason`，这些情况都让节点失败，不进入上表；流断开同样不会走到首轮停靠。收到 `Completed` 后只读一次历史、不等待，也不写 provisional，所以无 `TurnEnded` 或历史不可读直接得到非 provisional 的 `turn_unknown`，归 `abnormal`。

**与批准版本的偏离**：以下三处按最终实现更正，实现未改。

1. 批准版把会话崩溃、流断开写成 `turn_unknown`，静默超时写成「`turn_stopped` 或 `turn_unknown`」。实际在人类回合里，这些结束先以 `Cancelled` 收敛，blocker 是 `turn_stopped`。两者同归 `abnormal`，差别只在 force 审计记录的 blocker kind，来源仍由 `stopCause` 区分。
2. 批准版 D1 把自动首轮的静默超时、会话崩溃写成 `abnormal` 停靠。实际这两种情况让节点失败，不停靠，用户也就没有「继续或 force」的入口。
3. 批准版把无 `TurnEnded`、历史不可读一律归 `abnormal`。实际只有自动首轮如此；人类回合写 provisional，暂归 `awaiting_user`，等修复后再判定。

如果新快照的 `turn_stopped` 没有可用的 `stopCause`，按 `abnormal` 处理，不能假设是用户主动停止；`turn_unknown` 同理。旧快照没有 `parkKind` / `stopCause` 时走 D7 的 legacy 展示，不强行归类。

`parkKind` 是快照字段，不是节点状态：节点仍是 `Pending`，run 仍是 `Running`，「异常停靠」不是节点失败。用户仍可继续当前步骤、force 推进或关闭运行；本决策不新增失败状态、自动重试或自动恢复。

### D5：读取投影、步骤卡片与右下角按钮共用同一分类

读取投影继续只解析 `payload.completion`，不读会话历史、不跑 schema、不做修复。三类异常 cause 从人类 prompt 写入该快照的跨 crate 证据见[异常分类核心用例](../../../../test-cases/desktop/core/workflow/first-turn-confirmation.md#abnormal-parking-is-distinct-from-awaiting-user)。新增/透出：

- `turnOrigin`、`stopCause`、`parkKind`：有则透出，旧快照缺省；
- `needsConfirmation`：当且仅当快照非 provisional、blockers 恰好是 `[turn_unconfirmed]`；
- `ready`：仍由「blockers 为空且非 provisional」推导，不包含 `parkKind`；
- 未知 blocker kind 仍按阻塞处理，绝不能因为不认识而读成 ready。

步骤卡片、右下角按钮与 Theater act card 都消费同一组 `ready` / `needsConfirmation` / `parkKind`，不在各自组件里重新推导「是否暂停」。

| 状态 | 徽标 | 卡片主操作 | 卡片次操作 | 右下角按钮 |
| --- | --- | --- | --- | --- |
| ready（人类回合正常结束） | 等待参与 | 完成并进入下一步：`<next>`（不带字段） | 继续对话 | 启用，发送普通完成 |
| `needsConfirmation`（首轮正常结束） | 等待参与 | 确认并完成：`<next>`（`confirmInitialTurn: true`） | 继续当前步骤 | 禁用 |
| `awaiting_user`（用户主动停止） | 等待参与 | 继续当前步骤 | 仍然完成（force，二次确认） | 禁用 |
| `abnormal`（停靠时的超时/截断/拒答/未知/崩溃/输出不符合） | 异常停靠 | 继续当前步骤 | 仍然完成（force，二次确认） | 禁用 |
| provisional | 等待参与 | 完成（provisional 只属于人类回合） | 继续当前步骤 / 仍然完成（force） | 禁用 |
| 旧快照缺少 `parkKind` / 无快照 | 已暂停或状态未知（legacy） | 沿用现有 legacy 完成入口 | 继续当前步骤 / force | 沿用现有行为 |

- 首轮确认不使用通用的「完成当前节点」入口，也不使用右下角按钮：右下角按钮只对 `ready` 启用。原因是它没有「确认首轮」的上下文，重复使用会把新 gate 又变回一次无差别点击；步骤卡片把「确认并完成」与「继续当前步骤」并列，用户能看清自己是在推进还是在补充。
- `needsConfirmation` 的主操作是一次**普通完成**，不是 force；按钮文案不用「仍然完成」，也不需要二次确认对话框。它仍然会经过 turn key、digest、输出契约和文件 blockers 的全部校验。
- `abnormal` 与用户主动停止都不提供普通完成按钮；主操作是继续，次操作才是 force。force 仍走现有二次确认对话框，审计写 D3 的形状。
- 用户主动停止虽然只能 force 推进，但徽标仍是「等待参与」，原因是用户自己停了本轮，界面应该邀请他继续或选择推进，而不是把用户动作显示成系统异常。
- 「继续当前步骤」的提示文案按 `parkKind` 与 blocker 生成：`turn_unconfirmed` 说「首轮结果尚未确认」，用户主动停止说「你已停止本轮」，异常停靠说「上一轮异常结束：<原因>」；不能把首轮未确认写成异常，也不能把用户停止写成系统失败。
- provisional 只出现在人类回合，卡片主操作是普通「完成并进入下一步」；后端会先修复快照，修复出异常 blocker 时返回 `WorkflowNodeIncomplete`，卡片随后切换为 `abnormal`。同时保留「仍然完成」作为修复失败时的兜底。provisional 始终显示「等待参与」和「本轮仍在收尾」，避免在尚未收敛时把可能正常的回合标成异常。
- 旧快照缺少新字段时保留 legacy 路径：无快照的 `Pending` 节点仍显示「状态未知」，完成后由 `prepare` 物化；已有快照但缺 `parkKind` 的节点沿用「已暂停」而不是新「异常停靠」，直到下一次 park 或 `prepare` 修复写入新字段。legacy 路径不追溯加 `turn_unconfirmed`。
- Theater act card 优先读 `parkKind`：有它时不再使用「`awaiting_input` 且 `ready !== true` 且非 provisional 就是 paused」的启发式；`awaiting_user` → 「等待参与」，`abnormal` → 「异常停靠」；缺省时沿用 legacy 启发式。act card 只显示徽标，具体原因留在步骤卡片，避免两处文案分叉。
- 卡片仍只保留当前步骤一张，不按轮次累加；不新增第二个主操作。`WorkflowNodeTurnChanged` / `WorkflowNodeOutputChanged` 仍让前端刷新运行后重试；`invalid_request` 视为客户端请求错误，不改变卡片状态。

### D6：与方向 2（#427）的边界

本决策不是 agent 显式完成信号。它只在 workflow 自动发起的首轮加一个**人的确认**，不改变 `ready` 对 agent 意图的判断：

- `confirmInitialTurn` 只由人类界面发送，不进入 `NodeCompletionRequester::Agent` 的语义；#427 定义 agent 请求时必须单独说明 requester、权限、幂等与审计，不能复用本字段来冒充用户确认。
- 用户回复之后的回合仍按 #385：`EndTurn` + 无 blocker 即 ready。如果 agent 在用户回复后再次提问，快照仍是 ready，界面仍可能同时显示「等待参与」和「完成并进入下一步」；这个盲点由 [#427](https://github.com/ora-space/circus/issues/427) 的显式完成信号解决，不在本 ADR 内假装已修复。
- 如果 #427 先落地，它可能取代首轮 gate 或引入新的 blocker；届时需要重新评审 `turn_unconfirmed` 与 `confirmInitialTurn` 是否仍有必要，本决策不承诺它们在 #427 之后保持原样，只承诺复用同一条完成请求路径时不绕过状态、turn key、digest 与输出契约。
- 本决策不引入 LLM judge、不解析「继续/下一步」等词表、不根据 agent 文本猜测是否在提问；这些要么成本与误判不可接受，要么属于 #427 的信号设计。

### D7：兼容与升级边界：新快照生效，旧快照不追溯

- 新字段只写入 `payload.completion`，数据库不改 schema、不迁移既有 run、不改冻结图字段或本地文件布局。
- 本决策上线后新产生的 park 快照总是带 `turnOrigin` 与 `parkKind`；无法判定停靠原因时 `stopCause` 为 `unknown`，`parkKind` 保守取 `abnormal`。
- 升级前已停靠的快照没有 `turnOrigin` / `parkKind` / `stopCause`：保持 legacy 行为，不追溯加 `turn_unconfirmed`，不追溯改徽标；用户仍可按 #385 的规则完成或 force。`prepare` 只在修复 provisional 或物化无快照节点时按新形状重写；物化时没有 origin 的证据，按 `human` 处理，避免把历史停靠误判成首轮自动回合。
- 同一 `userSeq` 的 provisional 修复保留原 `turnOrigin` / `stopCause`，只重算本轮事实与 blockers；provisional 只属于人类回合，修复出 `EndTurn` 不补 `turn_unconfirmed`，修复出异常时只保留异常 blocker。修复不改变已经离开 `Pending` 的节点。
- 新 blocker kind `turn_unconfirmed` 是 wire 契约的增量。旧客户端不认识它时必须按阻塞处理；投影若采用未知 kind 归一策略，应归一到 `turn_unknown`，不能因为不认识而读成 ready。wire 枚举的穷举匹配必须在编译期或契约测试中更新，不能靠默认分支漏掉。
- 旧客户端不带 `confirmInitialTurn`：首轮停靠会得到 `WorkflowNodeIncomplete`，无法完成。desktop 的界面与 backend 同版本发布，不存在长期混用；Cloud 不执行 interactive Agent；未来 CLI 或其他客户端若要确认首轮，必须补发该字段。这是有意的兼容性代价，不是迁移遗漏。
- 不做 `Pending` 节点的批量回填，也不在读取投影里为了补字段去读历史；升级前停靠的节点可能仍保留原来的误判窗口，直到下一次 park 或 `prepare` 物化。这与 #385「投影不读历史、修复只在 prepare」的边界一致。

## 不变量

1. workflow 自动发起的首轮以 `EndTurn` 正常结束、历史完整、输出契约通过且没有其他 blocker 时，快照恰好写一个 `turn_unconfirmed`，不 ready。
2. 人类回合以 `EndTurn` 结束且没有其他 blocker 时仍是 ready；`turn_unconfirmed` 永远不写给人类回合。
3. 新快照中 `turn_unconfirmed` 不与更精确的 blocker 共存；若旧或异常快照同时含两者，`confirmInitialTurn` 只解除 `turn_unconfirmed`，其余 blocker 仍阻断。
4. `confirmInitialTurn` 只解除 `turn_unconfirmed`；状态/claim、turn key、`outputDigest`、输出契约与文件 blockers 都不能被它越过。
5. `force` 只在至少覆盖一个 blocker 时写 `completion_forced`；`force` 遇到已经 ready 的快照按普通完成处理。
6. `force` 不能覆盖 `turn_unconfirmed`；首轮正常结束的完成必须走 `confirmInitialTurn`，且不写 `completion_forced`。
7. `completion_forced` 记录实际覆盖的 blockers 与 `stop_cause`；`stop_cause` 区分用户停止、prompt 超时、流断开、会话崩溃与未知，不含用户文本。
8. `parkKind = awaiting_user` 只用于 `turn_unconfirmed`、用户主动停止与 provisional；正常 ready 的人类回合不写 `parkKind`，界面同样显示「等待参与」，其余停靠一律 `abnormal`。
9. 停靠时，`parkKind = abnormal` 覆盖超时、截断、拒答、未知、会话崩溃、结构化失败与输出缺失；此时节点仍是 `Pending`，run 仍是 `Running`，不是节点失败。自动首轮的静默超时、会话崩溃/流断开在停靠前就按节点失败处理，不进入本不变量的停靠分类。
10. `parkKind` 与 `stopCause` 在 park 或 `prepare` 修复时写入快照；读取投影只读快照，不读历史、不跑 schema、不按时间推断。
11. 右下角推进按钮只对 `ready` 启用（legacy 无快照路径除外）；首轮确认与 force 只从步骤卡片发起。
12. 步骤卡片与 act card 使用同一组 `ready` / `needsConfirmation` / `parkKind`，不在组件内重新推导暂停状态。
13. 首轮 gate 是每个 node run 的一次性 gate；用户回复后新的快照是 `human`，gate 不再适用。
14. 旧快照与旧客户端保持 legacy 行为，不追溯加 `turn_unconfirmed`，不做批量回填。
15. 非交互节点、正常人类 `EndTurn`、Cloud、`Loop`/`Iteration` 区域内的 interactive 行为不变。
16. #427 继续拥有「用户回复后 agent 再次提问」与 agent 自报完成的语义；本决策不改变 `ready` 对 agent 意图的判断。

## 为什么不是这些替代方案

| 方案 | 取舍 |
| --- | --- |
| 只改文案或加二次确认，不改完成契约 | 仍然是 `EndTurn` + 无 blocker 即 ready，卡片与按钮仍可一步推进；二次确认只是把误判推迟到用户点击之后 |
| 所有停靠回合都默认不 ready，每次完成都要确认 | 正常人类回合也被降级为每次强制确认，摩擦大，且与 #427 的 agent 意图信号重复；本决策只拦一次性的自动首轮 |
| 首轮确认沿用 `force` | 会把一次正常的首轮完成记成 `completion_forced`，把「用户确认了正常结果」和「用户带着异常 blocker 冒险推进」混为一种审计 |
| 新增 `turn_unconfirmed` 但仍用 `force` 完成 | 同上的问题；而且 `force` 可以越过任意 blocker，无法表达「只解除首轮未确认、其余异常仍然阻断」 |
| 完全依赖 #427 的 agent 显式完成信号 | 方向 2 需要新的 agent→host 信号、权限与人工确认契约，当前不可用；在它落地前，首轮提问回合仍会误判 |
| 用 LLM judge 判断 agent 是否在提问 | 自由文本没有可判定的目标，成本与误判都高，且需要模型凭据与降级策略；不应作为完成门禁 |
| 把用户主动停止也归为异常 | 用户自己停了本轮，界面应邀请继续或选择推进，而不是显示成系统异常；这会误导用户 |
| 把超时和用户主动停止都显示为「等待参与」 | 超时不是用户意图，且无法从 `Cancelled` 区分；必须用 `stopCause` 分开，否则异常会继续和参与混在一起 |
| 为超时新增 `turn_timed_out` blocker | 超时不是 ACP stop reason，新增 blocker 会把「停靠原因」塞进 stop reason 映射，扩大 wire 枚举；用 `stopCause` 更小且能同时表达流断开、会话崩溃 |
| 在读取投影里按时间/静默推断超时 | 投影只读 payload，没有可信的时间与 prompt 上下文；推断会把正常长回合误标成异常，且轮询成本高 |
| 从历史推断「这是不是首轮」 | 自动 prompt 本身也会写成用户消息，历史无法可靠区分自动回合与人类回合；必须由调用点显式传入 `turnOrigin` |
| 每次 park 都重新读历史判断 origin | park 已经读历史，但 origin 是调用上下文而非历史事实；再读一遍既不能证明，也会让 cleanup 路径与 executor 路径分叉 |
| 让右下角按钮也发送 `confirmInitialTurn` | 按钮没有「首轮」上下文，用户容易把它当成普通推进；保留卡片作为确认入口，按钮只服务已经 ready 的回合 |
| 普通确认也写 `completion_confirmed` 审计 | 正常完成本身已是持久记录，再加一个确认键会与 force 审计混淆；本决策用「无 `completion_forced`」作为普通确认的判据 |
| `parkKind` 只在 UI 派生，不写快照 | 后端在 park 时才知道 origin 与 `stopCause`，投影不应重新推断；写入快照能让两个界面入口读到同一分类 |
| 旧快照也追溯加 `turn_unconfirmed` | 没有 origin 证据，追溯会把可能由人类发起的停靠误判成首轮；还会破坏已停靠节点的既有操作预期 |

## 落地顺序

1. **决策评审**：核心用例见 `test-cases/desktop/core/workflow/first-turn-confirmation.md`；本 ADR 已标记为 `implemented`，证据状态按最终实现标注。
2. **契约与后端**：`crates/contracts` 增加 `turn_unconfirmed` blocker kind 与 `confirmInitialTurn` 请求字段；backend 的 park 函数接收 `turnOrigin` 与 `stopCause`，快照写入三个新字段并计算 `parkKind`；完成路径按 D2 的优先级实现确认分支，按 D3 写 `completion_forced.stop_cause`。不改数据库 schema，不迁移既有 run。
3. **投影与界面**：`packages/workflow-runtime` 与 app-shell 解析新字段并透出 `needsConfirmation` / `parkKind` / `stopCause`；步骤卡片按 D5 增加确认、异常原因与 force 入口；右下角按钮只对 `ready` 启用；act card 改用 `parkKind`；中英文案同步；`docs/workflows/workflow-runs.md` 与 `.zh.md` 的 interactive nodes / recovery 说明已同步。
4. **产品 PR**：`ora-space/desktop#751` 从 `cab59a3c` 改造并合并（merge `12c29e51`），把原来的「`turn_unconfirmed` + force」改成「显式确认 + 普通完成」，并加入异常停靠的徽标、原因与 force 审计。
5. **状态同步**：本 ADR 已改为 `implemented`，正文按最终实现同步并删除已失效的推测，核心用例证据按实现标注。

## 风险、未解决问题与验收

- **最大的已知限制在 #427，不在本决策**：用户回复后 agent 再次提问，快照仍是 `EndTurn` + 无 blocker，界面仍可能同时显示「等待参与」和「完成并进入下一步」。本决策只把首轮的这个盲点关掉；后续回合必须由 #427 的显式完成信号解决，不能用本 ADR 的确认字段冒充。
- **首轮 origin 依赖调用点**：任何新的 park 入口如果忘记传 `turnOrigin`，首轮 gate 就会丢失。实现中 `turnOrigin` 是 park 函数的必填参数（不设默认值），executor 与 cleanup 两个入口都显式传入；新增 park 入口时同样不能省略。
- **超时原因依赖 prompt 结束路径**：如果超时路径没有把 `prompt_timed_out` 传到 park，新快照会按 `turn_stopped` + 未知 cause 保守判成 `abnormal`。这比误显示成「等待参与」安全，但会把用户主动停止误标成异常；实现把 cancel 来源（用户停止 / 静默超时）随 owning prompt 的结束原因传入 park，而不是在 UI 或投影里猜。
- **旧快照与旧客户端**：升级前停靠的节点可能保留原来的误判窗口；旧客户端不带 `confirmInitialTurn` 无法完成首轮停靠。desktop 同版本发布可接受，Cloud 不执行 interactive Agent；其他客户端需要补字段。旧快照的徽标沿用 legacy，不追溯改成「异常停靠」。
- **`parkKind` 可能和 blockers 漂移**：如果某条路径只写 blockers 不写 `parkKind`，或手工改 payload，投影会显示缺失分类。实现把分类计算收敛到唯一的快照函数，所有写入路径共用；读取端对缺失字段走 legacy，不猜。
- **结构化/输出失败归入「异常停靠」**：它们不是 stop reason 异常，但也不是「等待参与」；徽标统一为「异常停靠」，原因文案分别说明「结构化输出不符合」/「没有输出」。如果产品希望单独做「输出不符合」第三态，应另开决策，不在本 ADR 临时加第三类。
- **用户主动停止仍可 force**：这符合 #728 的矩阵（用户停止：等待参与，否，需继续或 force），但审计会记录 `stop_cause: user_stopped`。需要避免把「force 只给异常」误读成「用户停止不能 force」；本决策的准确表述是「force 只给非确认类 blocker」。
- **首轮确认不是回答**：用户仍可在 agent 提问后点「确认并完成」跳过回答，这是显式选择，不是自动误判。卡片文案必须把「确认」和「继续当前步骤」的差别说清楚，否则只是把误点从底部按钮移到卡片。
- **provisional 的短暂分类**：provisional 期间显示「等待参与」，修复后可能变成 `abnormal`。这是 #385 已有的 provisional 语义，本决策不改变；如果用户在这段时间点完成，后端会先修复再判定，失败时返回 blockers。
- **区域与 Cloud 不在范围**：`Loop`/`Iteration` 区域内的 interactive 仍按既有边界处理；Cloud 不执行 interactive Agent，不新增发布门禁。
- **核心测试用例**：首轮确认、异常分类与归因、审计形状、旧快照兼容与 UI 入口已登记在 `test-cases/desktop/core/workflow/first-turn-confirmation.md`；证据状态按实际证据标注：归因链路、真实 host 接线、人类三类异常 cause 的跨 crate e2e 与自动交互式首轮异常失败均为 `Covered`，故障注入边界见该核心用例。

验收：

- workflow 自动首轮以 `EndTurn` 正常结束 → 快照为 `turn_unconfirmed`，`ready = false`；不带 `confirmInitialTurn` 的完成返回 `WorkflowNodeIncomplete`；带 `confirmInitialTurn` 的完成成功且 payload 没有 `completion_forced`；只带 `force` 被拒绝。
- 人类回合以 `EndTurn` 正常结束 → 仍是 ready，普通完成行为不变；用户回复后的回合不会被加上 `turn_unconfirmed`。
- 首轮/人类回合的用户主动停止 → `turn_stopped` + `stop_cause = user_stopped`，`parkKind = awaiting_user`，只能继续或 force；force 审计记录 `user_stopped`。
- 人类回合的 prompt 静默超时 → `stop_cause = prompt_timed_out`，`parkKind = abnormal`，普通完成与 `confirmInitialTurn` 都不能推进，只能继续或 force；force 审计记录 `prompt_timed_out`。自动首轮的同名结束是节点失败，不产生 `stopCause` / `parkKind`。
- 截断、拒答、未知、结构化失败、输出缺失 → `parkKind = abnormal`，界面显示对应原因，默认只能继续或 force；force 审计的 blockers 与 `stop_cause` 能区分类型。人类回合的会话崩溃/流断开同样停靠为 `abnormal`；自动首轮的同名情况是节点失败，不进入停靠。
- `confirmInitialTurn` 不能越过状态/claim、turn key、digest、输出契约或文件 blockers；`force` 与 `confirmInitialTurn` 同时为 true 返回 `invalid_request`；`WorkflowNodeIncomplete` 只列出请求尚未满足的 blocker。
- 步骤卡片：首轮确认只给「确认并完成」+「继续当前步骤」；异常停靠只给「继续当前步骤」+「仍然完成」；用户停止保持「等待参与」；右下角按钮只对 ready 启用。
- 投影与 act card 使用同一 `parkKind`；旧快照缺字段时走 legacy，不追溯加 `turn_unconfirmed`；不读历史、不跑 schema、不做批量回填。
- 非交互节点、正常人类完成、Cloud 与区域 interactive 行为不变；数据库无 schema 变更，冻结图不变。
- 核心测试用例已按最终实现补充证据；归因链路、真实 host 接线、人类三类异常 cause 的跨 crate e2e 与自动交互式首轮异常失败均为 `Covered`，故障注入边界见该核心用例。
