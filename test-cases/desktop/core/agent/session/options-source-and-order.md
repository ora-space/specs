# 会话选项的来源、顺序与在途设置

决策依据：[会话选项以 session actor 为唯一来源](../../../../../decisions/desktop/core/agent/session/20261125-session-options-single-source.md)（`implemented`）。

义务后的标注：**复现**在修复前失败，**回归保护**在修复前已经成立，**新增**是新的行为。证据固定到 desktop `d35f4d1a`（会话选项三步落地
desktop#837/#853/#857、Q2 desktop#839 与收尾小 PR desktop#860 合并后的 main）。宿主的顺序义务在 runtime 层由用例控制 ACP 帧的先后验证。

## The Last Option Frame Read From The Agent Must Win

### 风险

设置的答复和 Agent 的推送若走不同的路，晚到的旧值会盖掉新值，界面显示的模型和 Agent 实际使用的不一致。

### 前置状态

- 会话有 provider，当前选项是 A。
- 分别处于：回合中、空闲、标题轮询进行中、MCP 刷新进行中。

### 触发

设置选项，由用例控制 Agent 先答复 B 再推送 C，或先推送 C 再答复 B；前端再交换答复和推送到达的先后。

### 必须成立

- 当前选项是宿主最后读到的那一帧的值，各处处理队列的阶段都一样。
- 正在跟随的加载流、prompt 流和之后的加载显示同一份选项。
- 标题轮询期间收到的推送，不在下一回合开头重放；下一回合开始后当前选项仍是之后的设置答复。

### 禁止结果

- 按到达前端的先后，或按命令先于事件的处理次序决定最终值。
- 旧推送经命令队列或延后重放，排到更晚的答复之后。
- 视图按用户点击的值或设置的返回值写入会话副本。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| 同一回合内两种次序，runtime 的流和之后的加载是最后的值（复现） | [the_last_frame_a_running_turn_reads_decides_the_options](https://github.com/ora-space/desktop/blob/d35f4d1a636d1f64cdfaeaae2064d2b7c79f557f/crates/agent-runtime/src/restored_sessions/set_config_tests.rs#L398) 两种次序、之后的加载是最后的值；[an_answer_its_caller_awaits_reaches_the_views_of_a_turn](https://github.com/ora-space/desktop/blob/d35f4d1a636d1f64cdfaeaae2064d2b7c79f557f/crates/agent-runtime/src/restored_sessions/set_config_tests.rs#L1314) 回合中 prompt 流与跟随流按采用先后收到 | `Covered` |
| 前端：答复和推送到达的先后交换，单个视图的结果不变（复现） | [an option change the agent reports while following a turn wins over an earlier choice](https://github.com/ora-space/desktop/blob/d35f4d1a636d1f64cdfaeaae2064d2b7c79f557f/packages/chat/tests/store.test.ts#L420)、[keeps a model chosen while following a turn that completed/failed](https://github.com/ora-space/desktop/blob/d35f4d1a636d1f64cdfaeaae2064d2b7c79f557f/packages/chat/tests/store.test.ts#L347) 返回值不再带选项后，视图只按流内先后写入，推送或答复经流到达都按流序生效 | `Covered` |
| 空闲时旧推送先到、答复后到，两帧在 actor 下一次取输入之前都已就绪（复现） | [the_last_frame_an_idle_session_reads_decides_the_options](https://github.com/ora-space/desktop/blob/d35f4d1a636d1f64cdfaeaae2064d2b7c79f557f/crates/agent-runtime/src/restored_sessions/set_config_tests.rs#L403) | `Covered` |
| 标题轮询进行中推送先到、答复后到，两帧在 actor 下一次取输入之前都已就绪（复现） | [the_last_frame_a_title_poll_reads_decides_the_options](https://github.com/ora-space/desktop/blob/d35f4d1a636d1f64cdfaeaae2064d2b7c79f557f/crates/agent-runtime/src/restored_sessions/set_config_tests.rs#L408) | `Covered` |
| 标题轮询期间的推送不在下一回合重放（复现） | [options_pushed_during_a_title_poll_do_not_return_with_the_next_turn](https://github.com/ora-space/desktop/blob/d35f4d1a636d1f64cdfaeaae2064d2b7c79f557f/crates/agent-runtime/src/restored_sessions/set_config_tests.rs#L419) | `Covered` |
| MCP 刷新进行中推送与答复相继到达（新增） | [the_last_frame_an_mcp_refresh_reads_decides_the_options](https://github.com/ora-space/desktop/blob/d35f4d1a636d1f64cdfaeaae2064d2b7c79f557f/crates/agent-runtime/src/restored_sessions/set_config_tests.rs#L413) | `Covered` |
| 答复在标题轮询或 MCP 刷新期间到达也被采用（回归保护） | [a_change_answered_during_a_title_poll_is_kept](https://github.com/ora-space/desktop/blob/d35f4d1a636d1f64cdfaeaae2064d2b7c79f557f/crates/agent-runtime/src/restored_sessions/set_config_tests.rs#L239)、[a_change_answered_during_an_mcp_refresh_is_kept](https://github.com/ora-space/desktop/blob/d35f4d1a636d1f64cdfaeaae2064d2b7c79f557f/crates/agent-runtime/src/restored_sessions/set_config_tests.rs#L263) | `Covered` |

### 决策依据

- D1 第 2、8 点，D2。
- 不变量 1、2、5。

## Pushes Older Than A Model Intent Answer Must Not Undo It

### 风险

建立或恢复会话时，意图答复之前的旧推送若在答复之后处理，用户首轮选的模型会在界面上被撤销，之后的加载也得到旧值。

### 前置状态

- 调用方带模型意图，Agent 提供该模型。
- 分别由 `session/new` 建立会话、由 `session/load` 恢复会话、切换 Agent。
- 旧推送 A 分别落在：短收集窗口内；收集窗口之后、意图答复之前；恢复答复之后、意图答复之前。
- 另设没有模型意图、选项推送早于 `session/new` 答复的情形，答复分别带选项、带显式空列表、不带选项。

### 触发

Agent 推送 A，再答复意图为 B，随后发送第一条 prompt 并加载会话。

### 必须成立

- 带模型意图时，当前选项是意图答复的 B，之后的加载得到 B，旧推送 A 不随第一条 prompt 送给视图。
- 不带模型意图时，Agent 先推送 A，再答复 `session/new`。若答复带选项（包括显式空列表），第一条 prompt 和之后的加载都以答复中的选项为准。若答复省略选项，两者都得到 A。
- 早于 `session/new` 答复的推送只在答复带选项时被替代，显式空列表也算带选项；不带选项的答复不替代推送。

### 禁止结果

- 只丢弃暂存的更新，而仍留在事件队列里的旧推送照样被采用。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| 收集窗口之后的旧推送不被采用，之后的加载得到意图答复的值（复现） | [a_snapshot_older_than_the_applied_model_does_not_undo_it](https://github.com/ora-space/desktop/blob/d35f4d1a636d1f64cdfaeaae2064d2b7c79f557f/crates/agent-runtime/src/restored_sessions/setup_options_tests.rs#L240) 的 `AppliedAfterPush` 分支 | `Covered` |
| 早于 `session/new` 答复的选项推送只在答复带选项时被替代，显式空列表也算带选项（新增） | [a_snapshot_pushed_before_setup_was_answered_does_not_undo_the_answer](https://github.com/ora-space/desktop/blob/d35f4d1a636d1f64cdfaeaae2064d2b7c79f557f/crates/agent-runtime/src/restored_sessions/setup_options_tests.rs#L267)、[an_empty_answer_replaces_a_snapshot_pushed_before_it](https://github.com/ora-space/desktop/blob/d35f4d1a636d1f64cdfaeaae2064d2b7c79f557f/crates/agent-runtime/src/restored_sessions/setup_options_tests.rs#L283)、[a_snapshot_pushed_before_an_answer_without_options_is_kept](https://github.com/ora-space/desktop/blob/d35f4d1a636d1f64cdfaeaae2064d2b7c79f557f/crates/agent-runtime/src/restored_sessions/setup_options_tests.rs#L299) 答复省略选项时保留推送 | `Covered` |
| 收集窗口之后的旧推送不随第一条 prompt 送给视图（回归保护） | [a_snapshot_older_than_the_applied_model_does_not_undo_it](https://github.com/ora-space/desktop/blob/d35f4d1a636d1f64cdfaeaae2064d2b7c79f557f/crates/agent-runtime/src/restored_sessions/setup_options_tests.rs#L240) 第一条 prompt 只带 mode，不带旧选项 | `Covered` |
| 恢复答复之后、意图答复之前的旧推送不被采用（复现） | [a_restored_session_is_on_the_model_its_agent_accepted](https://github.com/ora-space/desktop/blob/d35f4d1a636d1f64cdfaeaae2064d2b7c79f557f/crates/agent-runtime/src/restored_sessions/setup_options_tests.rs#L319) 恢复分支按读帧先后，意图答复生效 | `Covered` |
| 收集窗口内的旧推送不撤销意图答复，不随第一条 prompt 送出（回归保护） | [a_snapshot_older_than_the_applied_model_does_not_undo_it](https://github.com/ora-space/desktop/blob/d35f4d1a636d1f64cdfaeaae2064d2b7c79f557f/crates/agent-runtime/src/restored_sessions/setup_options_tests.rs#L240) | `Covered` |

### 决策依据

- D1 第 2 点，D2 第 2、3 点。

## Frames From A Replaced Binding Must Not Describe Its Replacement

### 风险

会话被重建后采用旧绑定的答复，界面会显示替代会话并没有使用的模型。

### 前置状态

- 恢复的会话首轮失败、即将重建，期间有一个设置在途。
- 替代会话报告 B，用户设置的是 A。

### 触发

旧绑定的答复 A 分别在替代会话的回合中和回合后到达。

### 必须成立

- 当前选项、所有流和之后的加载都是 B。
- 设置返回成功，视图得到 B；A 不在替代会话上重发，也不报错。

### 禁止结果

- 旧绑定晚到的答复或推送改变当前选项。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| 旧绑定晚到的答复不改变当前选项，之后的加载是替代会话的选项（回归保护） | [a_late_answer_does_not_describe_the_session_that_replaced_its_own](https://github.com/ora-space/desktop/blob/d35f4d1a636d1f64cdfaeaae2064d2b7c79f557f/crates/agent-runtime/src/restored_sessions/set_config_tests.rs#L287) | `Covered` |
| 重建时设置返回成功，视图得到替代会话的选项（回归保护） | [a_late_answer_does_not_describe_the_session_that_replaced_its_own](https://github.com/ora-space/desktop/blob/d35f4d1a636d1f64cdfaeaae2064d2b7c79f557f/crates/agent-runtime/src/restored_sessions/set_config_tests.rs#L287) 不等答复即以替代会话的选项收尾；[a_change_in_flight_is_settled_when_its_session_goes](https://github.com/ora-space/desktop/blob/d35f4d1a636d1f64cdfaeaae2064d2b7c79f557f/crates/agent-runtime/src/restored_sessions/set_config_tests.rs#L835) 的 Switched 分支 | `Covered` |
| 旧绑定晚到的推送不改变当前选项（回归保护） | [a_follower_sees_the_state_of_the_session_that_replaced_a_stale_one](https://github.com/ora-space/desktop/blob/d35f4d1a636d1f64cdfaeaae2064d2b7c79f557f/crates/agent-runtime/src/restored_sessions/follower_setup_tests.rs#L218) 替代会话的选项盖过旧绑定的报告；旧绑定晚到推送的单独用例尚无，与答复共用 D1.3 的绑定校验 | `Partial` |

### 决策依据

- D1 第 3 点与在途设置的收尾，D2 第 4 点。
- 不变量 3。

## Configuration Calls Must Report Only Success Or Failure

### 风险

返回值若仍被当作当前选项，就又多一个写入者；失败被当成成功，用户会以为模型已经生效。

### 前置状态

- 会话有 provider。
- Agent 分别：照所选的值生效；答复别的值；返回 ACP 错误；超时之后才答复；两个设置同时在途。

### 触发

设置选项，成功后立即打开一个新视图。

### 必须成立

- 契约的返回值不带选项，视图不按所选的值写入会话副本，选择器可以在「进行中」显示所选的值。
- 成功返回时 actor 已处理完答复，之后打开的视图看到这次改动。
- Agent 没有照所选的值生效时返回成功，视图显示 Agent 报告的值；返回 ACP 错误时失败，当前选项不变。
- 超时返回失败；绑定不变时，之后到达的答复仍被采用并送达视图。
- 在途的多个设置不串行，最终值由 Agent 答复的先后决定。

### 禁止结果

- 前一个设置没有答复时挡住后一个设置。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| 成功返回之后打开的视图看到改动（回归保护） | [a_view_opened_after_a_change_is_told_about_it](https://github.com/ora-space/desktop/blob/d35f4d1a636d1f64cdfaeaae2064d2b7c79f557f/crates/agent-runtime/src/restored_sessions/set_config_tests.rs#L211) | `Covered` |
| ACP 错误返回失败且当前选项不变（回归保护） | [a_refused_change_leaves_the_options_alone](https://github.com/ora-space/desktop/blob/d35f4d1a636d1f64cdfaeaae2064d2b7c79f557f/crates/agent-runtime/src/restored_sessions/set_config_tests.rs#L480)；前端 [reports an unreachable model selection instead of silently keeping the old value](https://github.com/ora-space/desktop/blob/d35f4d1a636d1f64cdfaeaae2064d2b7c79f557f/packages/chat/tests/store.test.ts#L2511)、[returns the thumb to the reported level when the request fails](https://github.com/ora-space/desktop/blob/d35f4d1a636d1f64cdfaeaae2064d2b7c79f557f/packages/app-shell/src/features/chat/thought-level-selector.test.tsx#L633) | `Covered` |
| 未照所选值生效时返回成功并显示 Agent 的值（回归保护） | [changes_in_flight_are_answered_in_any_order](https://github.com/ora-space/desktop/blob/d35f4d1a636d1f64cdfaeaae2064d2b7c79f557f/crates/agent-runtime/src/restored_sessions/set_config_tests.rs#L654) 答复与所选不同仍成功；前端 [adopts the agent's answer to a model selection over the requested value](https://github.com/ora-space/desktop/blob/d35f4d1a636d1f64cdfaeaae2064d2b7c79f557f/packages/chat/tests/store.test.ts#L2474) | `Covered` |
| 超时后的答复被采用并送达视图（新增） | [an_answer_after_the_caller_timed_out_is_taken](https://github.com/ora-space/desktop/blob/d35f4d1a636d1f64cdfaeaae2064d2b7c79f557f/crates/agent-runtime/src/restored_sessions/set_config_tests.rs#L676)、[an_answer_after_the_caller_timed_out_reaches_the_views_of_a_turn](https://github.com/ora-space/desktop/blob/d35f4d1a636d1f64cdfaeaae2064d2b7c79f557f/crates/agent-runtime/src/restored_sessions/set_config_tests.rs#L499) | `Covered` |
| 多个在途设置不串行（回归保护） | [changes_in_flight_are_answered_in_any_order](https://github.com/ora-space/desktop/blob/d35f4d1a636d1f64cdfaeaae2064d2b7c79f557f/crates/agent-runtime/src/restored_sessions/set_config_tests.rs#L654)；前端 [lets a newer pick overtake one still in flight without snapping back](https://github.com/ora-space/desktop/blob/d35f4d1a636d1f64cdfaeaae2064d2b7c79f557f/packages/app-shell/src/features/chat/thought-level-selector.test.tsx#L601) | `Covered` |
| 返回值不带选项，前端不用它写入（新增） | [adopts the agent's answer to a model selection over the requested value](https://github.com/ora-space/desktop/blob/d35f4d1a636d1f64cdfaeaae2064d2b7c79f557f/packages/chat/tests/store.test.ts#L2474) `setConfig` 返回空、选项由读取得到；[a change on an idle session is in progress until a read after it answers](https://github.com/ora-space/desktop/blob/d35f4d1a636d1f64cdfaeaae2064d2b7c79f557f/packages/chat/tests/setting-options.test.ts#L122) 答复不带选项，读才给出选项 | `Covered` |

### 决策依据

- D1 第 4–6、8 点。
- 不变量 4。

## In Flight Changes Must Settle When The Binding Is Released

### 风险

绑定释放后答复找不到路由。没有人收尾的设置会一直等到超时；读队列的某个循环不认识设置的答复，当前选项还会少一次更新。

### 前置状态

- 会话有 provider，一个设置在途，Agent 还没有答复。

### 触发

分别：切换 Agent（代表 `Stop` 命令）、prompt 流的接收方放弃、会话被重建、事件队列溢出、连接断开（包括 Agent 进程被替换）；另在每一处读会话事件队列的循环中让答复到达。

### 必须成立

- `Stop` 命令、接收方放弃、会话被重建返回成功。
- Agent 失联或失控（溢出、断连、进程被替换、不活跃超时、取消无应答、刷新 MCP 失败）返回失败；同时成立时按先检测到的原因。
- 释放之后没有仍在等待的设置，不等到超时。
- prompt、取消时的排空、回合取消的预读、空闲排空、标题轮询、MCP 刷新、恢复会话、建会话的收集窗口中，凡可能遇到在途设置的，都能识别答复并完成设置。

### 禁止结果

- 绑定释放后设置仍等待答复。
- 溢出后返回成功，而当前选项没有更新。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| 切换 Agent 时返回成功（新增） | [a_change_in_flight_is_settled_when_its_session_goes](https://github.com/ora-space/desktop/blob/d35f4d1a636d1f64cdfaeaae2064d2b7c79f557f/crates/agent-runtime/src/restored_sessions/set_config_tests.rs#L835) 的 Switched 分支 | `Covered` |
| 用户停止会话时返回成功（回归保护） | [a_change_in_flight_is_settled_when_its_session_goes](https://github.com/ora-space/desktop/blob/d35f4d1a636d1f64cdfaeaae2064d2b7c79f557f/crates/agent-runtime/src/restored_sessions/set_config_tests.rs#L835) 的 Stopped/StoppedMidTurn 分支，不等答复即收尾 | `Covered` |
| 接收方放弃时返回成功（新增） | [a_change_in_flight_is_settled_when_its_session_goes](https://github.com/ora-space/desktop/blob/d35f4d1a636d1f64cdfaeaae2064d2b7c79f557f/crates/agent-runtime/src/restored_sessions/set_config_tests.rs#L835) 的 Unread/EndUnread 分支；[a_change_in_flight_is_made_when_a_refused_turn_cannot_be_told](https://github.com/ora-space/desktop/blob/d35f4d1a636d1f64cdfaeaae2064d2b7c79f557f/crates/agent-runtime/src/restored_sessions/set_config_tests.rs#L1256)、[a_change_in_flight_is_made_when_a_cancelled_turn_has_nobody_to_tell](https://github.com/ora-space/desktop/blob/d35f4d1a636d1f64cdfaeaae2064d2b7c79f557f/crates/agent-runtime/src/turn_cancel_tests.rs#L513)、[a_change_in_flight_is_made_when_a_stopped_turn_has_nobody_to_tell](https://github.com/ora-space/desktop/blob/d35f4d1a636d1f64cdfaeaae2064d2b7c79f557f/crates/agent-runtime/src/turn_cancel_tests.rs#L542)、[a_change_in_flight_is_made_when_a_cancel_finds_a_refusal_nobody_reads](https://github.com/ora-space/desktop/blob/d35f4d1a636d1f64cdfaeaae2064d2b7c79f557f/crates/agent-runtime/src/turn_cancel_tests.rs#L567) | `Covered` |
| 会话被重建时不等答复返回成功（新增） | [a_late_answer_does_not_describe_the_session_that_replaced_its_own](https://github.com/ora-space/desktop/blob/d35f4d1a636d1f64cdfaeaae2064d2b7c79f557f/crates/agent-runtime/src/restored_sessions/set_config_tests.rs#L287)、[a_change_in_flight_is_settled_when_its_session_goes](https://github.com/ora-space/desktop/blob/d35f4d1a636d1f64cdfaeaae2064d2b7c79f557f/crates/agent-runtime/src/restored_sessions/set_config_tests.rs#L835) 的 Switched 分支 | `Covered` |
| 队列溢出时返回失败（新增） | [a_change_in_flight_is_settled_when_its_session_goes](https://github.com/ora-space/desktop/blob/d35f4d1a636d1f64cdfaeaae2064d2b7c79f557f/crates/agent-runtime/src/restored_sessions/set_config_tests.rs#L835) 的 Overflowed 分支 | `Covered` |
| 连接断开（包括 Agent 进程被替换）时返回失败（新增） | [a_change_in_flight_is_settled_when_its_session_goes](https://github.com/ora-space/desktop/blob/d35f4d1a636d1f64cdfaeaae2064d2b7c79f557f/crates/agent-runtime/src/restored_sessions/set_config_tests.rs#L835) 的 Disconnected/DisconnectedMidTurn 分支；进程被替换并入断连，无单独用例 | `Covered` |
| 各读队列的循环识别设置答复（新增） | [a_change_answered_while_a_turn_is_cancelled_is_kept](https://github.com/ora-space/desktop/blob/d35f4d1a636d1f64cdfaeaae2064d2b7c79f557f/crates/agent-runtime/src/restored_sessions/set_config_tests.rs#L697)、[a_change_answered_just_before_a_cancelled_turn_is_kept](https://github.com/ora-space/desktop/blob/d35f4d1a636d1f64cdfaeaae2064d2b7c79f557f/crates/agent-runtime/src/restored_sessions/set_config_tests.rs#L725)、[a_change_answered_before_the_cancel_is_written_is_taken](https://github.com/ora-space/desktop/blob/d35f4d1a636d1f64cdfaeaae2064d2b7c79f557f/crates/agent-runtime/src/turn_cancel_tests.rs#L464)、[a_change_answered_before_the_queue_is_drained_is_taken](https://github.com/ora-space/desktop/blob/d35f4d1a636d1f64cdfaeaae2064d2b7c79f557f/crates/agent-runtime/src/turn_cancel_tests.rs#L486)、[a_change_written_late_is_answered](https://github.com/ora-space/desktop/blob/d35f4d1a636d1f64cdfaeaae2064d2b7c79f557f/crates/agent-runtime/src/restored_sessions/set_config_tests.rs#L785)；标题轮询、MCP 刷新见「来源与顺序」首条用例 | `Covered` |

setup 重放、看门狗合并取消两个收尾点的确定性用例，待 `end_on_channel` 改为带原因的枚举时一起补（owner 结论，circus#1565），由 [circus#1647](https://github.com/ora-space/circus/issues/1647) 承接；共用收尾函数的变异已有用例抓到。

### 决策依据

- D1 在途设置的收尾。
- 不变量 7。

## Session Setup Answers Must Enter The Same Source

### 风险

刷新 MCP 的答复被忽略，或启动、切换的返回值晚于流中的更新写入，都会留下旧值。

### 前置状态

- 会话有 provider；另有一个刚启动的会话和一个刚切换 Agent 的会话。

### 触发

刷新 MCP 的 `session/load` 答复带选项 B；启动或切换的返回值在流送来更新的选项之后才到达前端。

### 必须成立

- 刷新的答复被采用，当前选项是 B；不带选项的答复不改变当前选项。
- 启动和切换的返回值按一次取回的结果处理，不盖掉流中更新的选项。

### 禁止结果

- 返回值盖掉更新的选项。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| 刷新 MCP 的答复带选项时被采用（新增） | [options_an_mcp_refresh_answers_with_are_taken](https://github.com/ora-space/desktop/blob/d35f4d1a636d1f64cdfaeaae2064d2b7c79f557f/crates/agent-runtime/src/restored_sessions/set_config_tests.rs#L457) | `Covered` |
| 刷新 MCP 的答复不带选项时不改变当前选项（回归保护） | [a_change_answered_during_an_mcp_refresh_is_kept](https://github.com/ora-space/desktop/blob/d35f4d1a636d1f64cdfaeaae2064d2b7c79f557f/crates/agent-runtime/src/restored_sessions/set_config_tests.rs#L263) | `Covered` |
| 启动、切换的返回值不盖掉流中的更新（新增） | [options a start or a switch answered with give way to newer ones a stream brought](https://github.com/ora-space/desktop/blob/d35f4d1a636d1f64cdfaeaae2064d2b7c79f557f/packages/chat/tests/setting-options.test.ts#L242) 守卫只跳过选项与其标记，用量重置和诊断清理照常 | `Covered` |

### 决策依据

- D1 第 2、8 点。