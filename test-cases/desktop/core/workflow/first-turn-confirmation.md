# 首轮确认与异常停靠

本文件是[首轮确认与异常停靠决策](../../../../decisions/desktop/core/workflow/20261102-first-turn-confirmation.md)的核心用例。实现随 desktop#751 合入，park 级集成证据由 desktop#755 补；`Sessions::prompt → WorkflowSessionTurns::prompt` 的正常 host 接线已有 e2e；[desktop#763](https://github.com/ora-space/desktop/pull/763) 补充了三类异常的[人类回合跨 crate e2e](https://github.com/ora-space/desktop/blob/8307f484bacb4dd727d0d1a2ce44f75e23e2bbe5/e2e/tests/desktop/core/session/tests/abnormal_human_turn.rs) 与[自动交互式首轮失败 e2e](https://github.com/ora-space/desktop/blob/8307f484bacb4dd727d0d1a2ce44f75e23e2bbe5/e2e/tests/desktop/core/session/tests/abnormal_first_turn.rs)。owner stream drop 与 provider 崩溃的区别及测试控制边界见异常分类义务。

## First Automatic Turn Awaits Explicit Confirmation

### 风险

workflow 自动发起的首轮以 `EndTurn` 正常结束，但 agent 实际上在向用户提问；界面把它当成 ready，用户一点就跳过回答。

### 前置状态

- 交互式 Agent 节点由 workflow 自动发起首轮；
- 本轮 `EndTurn`、历史完整、输出契约通过、没有其他 blocker；
- 节点策略为 `completionSignal: end_turn` 或升级前 legacy；`agent` 策略的首轮组合见 [agent completion signal 用例](agent-completion-signal.md)。

### 触发

首轮 park；用户点击完成。

### 必须成立

- 快照写入 `turnOrigin = "workflow"` 与 `turn_unconfirmed`，`ready = false`；
- 在 `end_turn`/legacy 下，步骤卡片显示「确认并完成」+「继续当前步骤」，右下角按钮禁用；`agent` 策略的卡片分支见新用例；
- 不带 `confirmInitialTurn` 的完成返回 `WorkflowNodeIncomplete`，列出 `turn_unconfirmed`；
- 在 `end_turn`/legacy 下，带 `confirmInitialTurn` 的完成成功，且不写 `completion_forced`；
- 在 `end_turn`/legacy 下，只带 `force` 被拒绝。

### 禁止结果

- 首轮自动回合在未确认时直接推进；或 `force` 把 `turn_unconfirmed` 当成可覆盖的异常 blocker。

### 验证义务与证据

| 验证义务 | 代表性证据 | 状态 |
| --- | --- | --- |
| 首轮自动 park 写 `turn_unconfirmed` 且不 ready | `workflow_initial_turn_is_not_confirmed_complete`、`the_opening_turn_parks_as_awaiting_user_confirmation`；e2e `the_opening_turn_completes_only_by_explicit_confirmation` | `Covered` |
| 未确认完成被拒、确认完成成功且无 force 审计 | `confirming_the_opening_turn_completes_without_a_forced_audit`、`force_cannot_stand_in_for_confirmation`、`confirmation_is_a_noop_on_a_ready_snapshot` | `Covered` |

### 决策依据

[首轮确认与异常停靠决策](../../../../decisions/desktop/core/workflow/20261102-first-turn-confirmation.md) D1、D2。

## Abnormal Parking Is Distinct From Awaiting User

有限 `wait` 到期时，自动首轮直接失败，人类回合在原会话以 `prompt_timed_out` 异常停靠。两条生产入口及会话视图的直接证据见[有限等待到期用例](prompt-inactivity.md#a-bounded-wait-settles-once-before-node-failure-handling)，该义务整体为 `Partial`（自动首轮释放执行会话尚无独立断言）。

### 风险

超时、截断、拒答、未知、会话崩溃等异常与「等待参与」共用同一状态，用户可以在没有确认的情况下推进。

### 前置状态

- 交互式节点停靠，`stopCause` 为 `prompt_timed_out` / `stream_dropped` / `session_crashed` / `unknown`，或 stop reason 为截断/拒答/未知；
- 另一节点由用户主动停止（`user_stopped`）。

### 触发

park 与读取投影。

### 必须成立

- 快照写入 `parkKind = "abnormal"` 与 `stopCause`；用户主动停止写 `awaiting_user` + `user_stopped`；
- 异常停靠显示异常原因，默认不可推进，只能继续或 force（二次确认）；
- 用户主动停止显示「等待参与」，继续或 force 均可；
- 右下角按钮只在 ready 时启用。

### 禁止结果

- 异常停靠显示为「等待参与」；或没有 force 就能推进。

### 验证义务与证据

| 验证义务 | 代表性证据 | 状态 |
| --- | --- | --- |
| 异常与用户停止的 `parkKind` / `stopCause` 分类（拿到 `stopCause` 后） | `a_user_stopped_turn_parks_as_awaiting_user`、`a_user_stop_parks_as_awaiting_participation`、`a_timed_out_turn_parks_as_abnormal`、`an_unattributed_stop_parks_as_abnormal`、`other_blocked_turns_park_as_abnormal`；`silent_timeout_stream_drop_and_session_crash_park_as_abnormal_with_their_cause`（park 级集成：真实 history + SQLite + `end_human_turn`）；e2e `a_stopped_opening_turn_waits_for_the_user`；三类异常的真实串联见下方跨 crate e2e 义务 | `Covered` |
| 归因（`PromptEndReport` → `StopCause` → park 快照） | `a_workflow_turn_records_the_reported_end_cause_in_its_parked_snapshot`、`a_provider_crash_reports_session_crashed_rather_than_a_dropped_stream` | `Covered` |
| `Sessions::prompt → WorkflowSessionTurns::prompt` 真实 host 接线 | e2e `a_stopped_step_blocks_completion_and_continues_after_a_restart`、`a_plain_conversation_saying_continue_does_not_advance_a_node` 通过 `backend.sessions().prompt(...)` 走真实 host，覆盖 workflow 人类回合的 admission、步骤边界与停靠 | `Covered` |
| 异常 cause（静默超时、provider 崩溃、owner stream drop）的跨 crate e2e | `a_timed_out_human_turn_parks_as_abnormal_with_prompt_timed_out`、`a_crashed_human_turn_parks_as_abnormal_with_session_crashed`、`a_dropped_human_prompt_stream_parks_as_abnormal_with_stream_dropped`：经真实 `Sessions::prompt`、runtime、history 与 SQLite，断言 `abnormal`、准确 `stopCause` 及默认完成被拒 | `Covered` |
| 自动交互式首轮静默超时耗尽、会话崩溃或 owning stream 提前结束时不停靠，节点失败 | `a_silent_workflow_first_turn_exhausts_inactivity_and_fails_the_node`、`a_crashed_workflow_first_turn_fails_the_node_without_parking`、`a_workflow_first_turn_without_a_terminal_event_fails_the_node`：发布并启动冻结 `interactive: true` 的真实 workflow，经 `drive_agent_node` 断言节点失败、无 completion 停靠快照且后继未执行 | `Covered` |
| 异常默认不可推进、只能 force | `blocked_turn_returns_every_blocker_with_its_detail`；前端 `requires confirmation before force-completing an abnormal stop` | `Covered` |

证据边界：静默用例按 Backend 实例缩短四个 inactivity 窗口，仍断言初次发送及全部三次重发，生产窗口与策略保持原值。fake provider 收到含本次唯一 marker 的 prompt 后写 journal，再静默或退出；owner drop 在 journal 握手后取消真实 prompt 流。自动首轮由仅 `test-support` 的 executor 控制对它持有的流调用 `cancel_and_wait()`，随后走无 `Completed` EOF 的既有失败分支；该证据证明 owning consumer 放弃流，不把 provider transport 断开复用为 `stream_dropped`。

### 决策依据

[首轮确认与异常停靠决策](../../../../decisions/desktop/core/workflow/20261102-first-turn-confirmation.md) D4、D5。

## Explicit Confirmation Is A Normal Completion

### 风险

把首轮确认当成强制完成，写入 `completion_forced`，污染审计语义。

### 前置状态

- 快照只有 `turn_unconfirmed`，没有其他 blocker。

### 触发

带 `confirmInitialTurn: true` 的完成请求。

### 必须成立

- 走普通完成路径，不写 `completion_forced`；
- 完成后的 payload 保留 `turn_unconfirmed`、turn key 与 `outputDigest`；
- `confirmInitialTurn` 不越过状态/claim、turn key、digest、输出契约或文件 blockers。

### 禁止结果

- 确认请求写入强制审计；或越过其他 blocker。

### 验证义务与证据

| 验证义务 | 代表性证据 | 状态 |
| --- | --- | --- |
| 确认完成不写 `completion_forced` | `confirming_the_opening_turn_completes_without_a_forced_audit` | `Covered` |
| 确认不越过其他 blocker | `confirmation_cannot_lift_other_blockers`、`force_cannot_stand_in_for_confirmation` | `Covered` |

### 决策依据

[首轮确认与异常停靠决策](../../../../decisions/desktop/core/workflow/20261102-first-turn-confirmation.md) D2、D3。

## Forced Completion Audits Only Covered Blockers

### 风险

`force` 对没有 blocker 的快照也写审计，或审计不记录 `stopCause` 与实际覆盖的 blocker。

### 前置状态

- 快照含非 `turn_unconfirmed` 的 blocker；或快照已 ready。

### 触发

带 `force: true` 的完成请求。

### 必须成立

- 实际覆盖至少一个 blocker 时才写 `completion_forced`；
- 审计包含 `blockers`、`stop_cause`、`without_turn`、`output_source`；
- 快照已 ready 时按普通完成处理，不写空审计；
- `force && confirmInitialTurn` 在例外之外返回 `invalid_request`；首轮正常结束且无信号的组合请求例外见 [agent completion signal 用例](agent-completion-signal.md)。

### 禁止结果

- 空审计；或 `force` 覆盖 `turn_unconfirmed`。

### 验证义务与证据

| 验证义务 | 代表性证据 | 状态 |
| --- | --- | --- |
| force 审计形状与覆盖范围（`blockers` / `stop_cause` / `without_turn` / `output_source`，与完成同一 payload） | `forcing_a_stopped_turn_records_the_turn_window`、`forcing_an_unsettled_turn_records_the_last_assistant_message`、`a_forced_completion_records_its_audit_in_the_completed_payload` | `Covered` |
| 快照已 ready 时 `force` 按普通完成处理、不写空审计 | `ready_force_does_not_write_an_empty_audit`（公开 `complete_node(force: true)`，断言无 `completion_forced` 且快照不变） | `Covered` |
| 互斥请求在例外之外返回 `invalid_request`（首轮无信号的组合请求例外见 [agent completion signal 用例](agent-completion-signal.md)） | `conflicting_completion_flags_are_rejected_before_claiming` | `Covered` |

### 决策依据

[首轮确认与异常停靠决策](../../../../decisions/desktop/core/workflow/20261102-first-turn-confirmation.md) D2、D3。

## Legacy Snapshots Keep Their Previous Semantics

### 风险

升级时给旧快照追溯加 `turn_unconfirmed`，把可能由人类发起的停靠误判成首轮。

### 前置状态

- 升级前已停靠、没有 `turnOrigin` 的 `Pending` 节点。

### 触发

升级后读取/完成该节点。

### 必须成立

- 旧快照不追溯加 `turn_unconfirmed`；
- legacy 物化默认 `turnOrigin = "human"`；
- 旧客户端不带 `confirmInitialTurn` 时不能完成首轮停靠，必须升级；
- 徽标沿用 legacy，不追溯改成异常。

### 禁止结果

- 批量回填或改写旧 payload；或把旧快照当首轮自动 gate。

### 验证义务与证据

| 验证义务 | 代表性证据 | 状态 |
| --- | --- | --- |
| 旧快照不追溯、物化默认 human | `a_legacy_node_is_materialized_and_blocked_by_an_unsettled_turn`、`a_legacy_node_with_a_settled_turn_completes_on_its_turn_window`、`legacy_snapshot_without_turn_origin_defaults_to_human`（缺 `turnOrigin` 默认 Human，且不加 `turn_unconfirmed`）；前端 `keeps old snapshots and unknown enum values on the legacy projection`、`keeps the ordinary completion entry for a node parked without a snapshot` | `Covered` |
| 旧客户端请求被正确拒绝 | e2e `the_opening_turn_completes_only_by_explicit_confirmation`（同一用例中不带 `confirmInitialTurn` 的普通完成与 force 都被拒） | `Covered` |

### 决策依据

[首轮确认与异常停靠决策](../../../../decisions/desktop/core/workflow/20261102-first-turn-confirmation.md) D7。

## UI Entries Share The Park Classification

### 风险

步骤卡片、右下角按钮与 Theater act card 各自推导暂停状态：首轮未确认被当成可普通推进，用户主动停止被显示成异常，或旧快照被追溯标成异常停靠。

### 前置状态

- 读取投影给出 `needsConfirmation` / `parkKind` / `stopCause`，或快照是 provisional / legacy（缺字段或无快照）。

### 触发

读取运行投影；渲染步骤卡片、右下角按钮与 Theater act card。

### 必须成立

- 投影透出 `needsConfirmation` / `parkKind` / `stopCause`；前端读取投影值，不按 blockers 重新推导。
- `end_turn`/legacy 首轮卡片给「确认并完成」+「继续当前步骤」，确认请求带 `confirmInitialTurn` 且不带 `force`；`agent` 策略首轮由 [agent completion signal 用例](agent-completion-signal.md)覆盖（有信号时 `confirmInitialTurn` + `confirmAgentCompletion`；无信号时主操作继续、次操作 `confirmInitialTurn` + `force`），对应 UI 义务为 `Missing`；右下角按钮禁用。
- 用户主动停止显示「等待参与」，异常停靠显示「异常停靠」并只给继续或 force（二次确认）；act card 读同一个 `parkKind`。
- 右下角完成按钮只对 `ready` 启用；确认与 force 只从步骤卡片发起。
- legacy 快照与无快照节点保留普通完成入口和 legacy 徽标，不追溯改成异常。

### 禁止结果

- 组件各自推导暂停状态；首轮未确认可以普通推进；用户停止被显示成异常；legacy 快照被追溯标成异常。

### 验证义务与证据

| 验证义务 | 代表性证据 | 状态 |
| --- | --- | --- |
| 投影透出确认与停靠分类，前端读取投影值 | `workflow-runs.test.ts` 的 `keeps the unconfirmed opening turn awaiting confirmation on both sides`、`does not ask for confirmation on a provisional or multi-blocker snapshot`；`run-node-completion.test.ts` 的 `reads the projected needsConfirmation flag instead of re-deriving blockers` | `Covered` |
| `end_turn`/legacy 步骤卡片按分类给出主/次操作，首轮确认不写 force | `run-node-session-chat.test.tsx` 的 `shows a ready step card that completes without force and names the next step`、`keeps the opening step awaiting confirmation instead of offering to advance`、`confirms the opening step with confirmInitialTurn and without force`、`sends the unconfirmed scoped turn when continuing the opening step`、`keeps a user-stopped turn awaiting input instead of reading as an abnormal stop`、`sends a scoped turn that does not frame a user stop as a system failure`、`requires confirmation before force-completing an abnormal stop` | `Covered` |
| `agent` 策略首轮卡片：有信号时双确认、无信号时 confirm + force | 见 [agent completion signal 用例](agent-completion-signal.md) 的 `Missing` 义务 | `Missing` |
| 右下角按钮只对 `ready` 启用 | `run-node-session-chat.test.tsx` 的 `disables the bottom-right completion action while blockers remain`、`disables the complete button while the agent is responding`、`completes the node when the button is clicked while awaiting input` | `Covered` |
| act card 读 `parkKind` 而不是 paused 启发式 | `run-theater-act-card.test.tsx` 的 `shows the backend's abnormal park classification instead of the paused heuristic`、`shows a user stop as awaiting input, not as an abnormal stop`、`keeps an unconfirmed opening turn out of the paused badge`、`never labels the unconfirmed opening turn as an abnormal stop` | `Covered` |
| legacy 投影与无快照节点保留旧入口 | `workflow-runs.test.ts` 的 `keeps old snapshots and unknown enum values on the legacy projection`、`leaves completion absent on older node runs without a snapshot`；`run-node-session-chat.test.tsx` 的 `keeps the ordinary completion entry for a node parked without a snapshot`；`run-theater-act-card.test.tsx` 的 `labels a parked interactive turn as paused without printing the raw stop reason` | `Covered` |

### 决策依据

[首轮确认与异常停靠决策](../../../../decisions/desktop/core/workflow/20261102-first-turn-confirmation.md) D5、不变量 11、12。
