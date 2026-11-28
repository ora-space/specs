# 会话选项的送达与失去 provider

决策依据：[会话选项以 session actor 为唯一来源](../../../../../decisions/desktop/core/agent/session/20261125-session-options-single-source.md)（`implemented`）。

义务后的标注：**复现**在修复前失败，**回归保护**在修复前已经成立，**新增**是新的行为。证据固定到 desktop `d35f4d1a`（会话选项三步落地
desktop#837/#853/#857、Q2 desktop#839 与收尾小 PR desktop#860 合并后的 main）。Desktop 只有一份会话副本，不做界面级的多视图
验证：多视图在 runtime 层用多条加载流、跟随流和应用事件订阅验证，前端用一份会话副本验证单个视图的写入规则。

## Every Session Stream Must Start With The Current Options

### 风险

流没有可靠的起点，视图在流开始时丢弃取回的结果，就可能丢掉唯一的新值。

### 前置状态

- 会话已附着，上次流结束之后当前选项变过。
- 另设当前选项为空集合、会话没有 provider 两种状态。

### 触发

分别打开 prompt 流、空闲时的加载流、回合中跟随的加载流。

### 必须成立

- 有 provider 时，每条流的开头都是 actor 的当前选项，空集合也带。
- 没有 provider 时流不带选项，流结束后视图取一次，得知「没有 provider」。

### 禁止结果

- 已附着会话的 prompt 流不带选项。
- 因选项为空而省略，视图保留旧选项。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| 已附着会话的 prompt 流以当前选项开头（复现） | [every_stream_starts_from_the_options_the_session_is_on](https://github.com/ora-space/desktop/blob/d35f4d1a636d1f64cdfaeaae2064d2b7c79f557f/crates/agent-runtime/src/restored_sessions/set_config_tests.rs#L1046) 已附着 prompt 流首帧是当前选项；[options_an_mcp_refresh_answers_with_are_taken](https://github.com/ora-space/desktop/blob/d35f4d1a636d1f64cdfaeaae2064d2b7c79f557f/crates/agent-runtime/src/restored_sessions/set_config_tests.rs#L457) MCP 刷新后首帧取刷新答复 | `Covered` |
| 空集合也带（复现） | [every_stream_starts_from_the_options_the_session_is_on](https://github.com/ora-space/desktop/blob/d35f4d1a636d1f64cdfaeaae2064d2b7c79f557f/crates/agent-runtime/src/restored_sessions/set_config_tests.rs#L1046) 空列表首帧 `NoOptions` | `Covered` |
| 加载流以非空的当前选项开头（回归保护） | [a_view_opened_after_a_change_is_told_about_it](https://github.com/ora-space/desktop/blob/d35f4d1a636d1f64cdfaeaae2064d2b7c79f557f/crates/agent-runtime/src/restored_sessions/set_config_tests.rs#L211) | `Covered` |
| 跟随流以非空的当前选项开头（回归保护） | [every_stream_starts_from_the_options_the_session_is_on](https://github.com/ora-space/desktop/blob/d35f4d1a636d1f64cdfaeaae2064d2b7c79f557f/crates/agent-runtime/src/restored_sessions/set_config_tests.rs#L1046) 跟随流首帧；[a_view_opened_after_a_change_is_told_about_it](https://github.com/ora-space/desktop/blob/d35f4d1a636d1f64cdfaeaae2064d2b7c79f557f/crates/agent-runtime/src/restored_sessions/set_config_tests.rs#L211) 回合中分支 | `Covered` |
| 没有 provider 的加载结束后视图得知「没有 provider」（新增） | [reads after a stream that brought no options, and not after one that brought an empty list](https://github.com/ora-space/desktop/blob/d35f4d1a636d1f64cdfaeaae2064d2b7c79f557f/packages/chat/tests/options-sync.test.ts#L190) 加载无选项→流结束后读→`providerLost` | `Covered` |

### 决策依据

- D3 第 2、5 点。
- 不变量 6。

## Every Option Change Must Be Announced Without Carrying Options

### 风险

只在空闲时通知会漏掉回合中的变化；通知带值又多一个来源。空闲时没有流，通知丢了视图会停在旧值。

### 前置状态

- 视图加载完成，没有流；会话分别空闲、有工作流等后端发起的回合在跑。
- 另有一条正在跟随该回合的流。

### 触发

Agent 推送选项；另在应用事件断开期间推送，再重新订阅。

### 必须成立

- 每采用一帧选项都随流送出并发出应用级广播，不比较值，广播只带 Ora 会话 id。
- 没有流的视图取一次，显示新值；有流的视图以流为准。
- 重新订阅后，对正在展示的会话重新取一次，得到新值。
- 广播按会话合并，突发不撑满每个客户端的事件队列。
- 广播和查询不访问 Agent，不改变会话状态，不写历史。

### 禁止结果

- 广播带选项。
- 因回合在跑而不广播。
- 用加载代替查询，清掉「已切换模型」标记。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| 空闲时 Agent 推送，单个视图经广播和查询显示新值（复现） | [every_options_frame_is_announced_and_can_be_read](https://github.com/ora-space/desktop/blob/d35f4d1a636d1f64cdfaeaae2064d2b7c79f557f/crates/agent-runtime/src/restored_sessions/set_config_tests.rs#L1080)；前端 [reads the options of a session without a stream when they are announced](https://github.com/ora-space/desktop/blob/d35f4d1a636d1f64cdfaeaae2064d2b7c79f557f/packages/chat/tests/options-sync.test.ts#L105) | `Covered` |
| 值没有变的帧也随流送出并广播（新增） | [every_options_frame_is_announced_and_can_be_read](https://github.com/ora-space/desktop/blob/d35f4d1a636d1f64cdfaeaae2064d2b7c79f557f/crates/agent-runtime/src/restored_sessions/set_config_tests.rs#L1080) 同值推送也 `announced+1`；前端 [a change during a stream ends on the options the stream brings, also unchanged ones](https://github.com/ora-space/desktop/blob/d35f4d1a636d1f64cdfaeaae2064d2b7c79f557f/packages/chat/tests/setting-options.test.ts#L144) | `Covered` |
| 回合中变化也广播，没有跟随的视图取到新值（新增） | [every_options_frame_is_announced_and_can_be_read](https://github.com/ora-space/desktop/blob/d35f4d1a636d1f64cdfaeaae2064d2b7c79f557f/crates/agent-runtime/src/restored_sessions/set_config_tests.rs#L1080) 回合中分支，查询读到新值 | `Covered` |
| 有流的视图以流为准（新增） | [drops a read that a stream started or brought options during](https://github.com/ora-space/desktop/blob/d35f4d1a636d1f64cdfaeaae2064d2b7c79f557f/packages/chat/tests/options-sync.test.ts#L149)；[keeps a model chosen while following a turn that completed/failed](https://github.com/ora-space/desktop/blob/d35f4d1a636d1f64cdfaeaae2064d2b7c79f557f/packages/chat/tests/store.test.ts#L347) | `Covered` |
| 重新订阅后取到新值（新增） | [tells the chat store which session's options changed, and has it read them again on each subscription](https://github.com/ora-space/desktop/blob/d35f4d1a636d1f64cdfaeaae2064d2b7c79f557f/packages/app-shell/src/state/hooks/use-app-events.test.ts#L121) 验证重新订阅触发 `resyncOptions`；[reads every conversation on screen again after announcements may have been lost](https://github.com/ora-space/desktop/blob/d35f4d1a636d1f64cdfaeaae2064d2b7c79f557f/packages/chat/tests/options-sync.test.ts#L212) 验证发起两次读取，未断言重读结果写入会话 | `Partial` |
| 广播按会话合并，突发不撑满队列（新增） | [merges_option_announcements_waiting_for_a_client](https://github.com/ora-space/desktop/blob/d35f4d1a636d1f64cdfaeaae2064d2b7c79f557f/crates/backend/src/app_event.rs#L371) 连发 100×2 条，每会话只到一条、流不断 | `Covered` |
| 广播和查询只读（新增） | [reading_the_options_changes_nothing](https://github.com/ora-space/desktop/blob/d35f4d1a636d1f64cdfaeaae2064d2b7c79f557f/crates/agent-runtime/src/restored_sessions/set_config_tests.rs#L1164) 不访问 Agent、不改会话、不写历史 | `Covered` |

### 决策依据

- D3 第 1、3、4、8、9 点。
- 不变量 5、8。

## Fetched Options Must Not Overwrite Newer Stream Options

### 风险

取回的结果和流是两条路，到达先后不定。取回的旧值盖掉流中的新值，或有流时忽略的广播恰是唯一的新值，界面都会停在旧值。

### 前置状态

- 视图正在取当前选项，由用例控制结果返回的时刻。

### 触发

取的期间：收到流中的选项、流开始、又收到广播；以及回合刚结束后选项变化，广播先于流的结束到达视图。

### 必须成立

- 期间从流收到选项或流开始，丢弃这次结果。
- 期间又收到广播，取完再取一次。
- 有流时收到的广播，流结束后补取；流没有送来选项就结束时也补取，空集合也算送来了选项。

### 禁止结果

- 为判断先后给选项加版本号。
- 有流时收到的广播被直接忽略。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| 晚到的取回结果不盖掉流中的选项（新增） | [drops a read that a stream started or brought options during](https://github.com/ora-space/desktop/blob/d35f4d1a636d1f64cdfaeaae2064d2b7c79f557f/packages/chat/tests/options-sync.test.ts#L149) | `Covered` |
| 取的期间又收到广播会再取（新增） | [reads once more when the options are announced during a read](https://github.com/ora-space/desktop/blob/d35f4d1a636d1f64cdfaeaae2064d2b7c79f557f/packages/chat/tests/options-sync.test.ts#L135) | `Covered` |
| 流结束之前收到的广播在流结束后补取（新增） | [reads once a stream ends, when the options were announced during it](https://github.com/ora-space/desktop/blob/d35f4d1a636d1f64cdfaeaae2064d2b7c79f557f/packages/chat/tests/options-sync.test.ts#L169)；[counts a prompt stream as open: an announcement during it is read once it ends](https://github.com/ora-space/desktop/blob/d35f4d1a636d1f64cdfaeaae2064d2b7c79f557f/packages/chat/tests/options-sync.test.ts#L221) | `Covered` |
| 流没有送来选项就结束时补取；只送来空集合且期间未收到广播时不补取（新增） | [reads after a stream that brought no options, and not after one that brought an empty list](https://github.com/ora-space/desktop/blob/d35f4d1a636d1f64cdfaeaae2064d2b7c79f557f/packages/chat/tests/options-sync.test.ts#L190) | `Covered` |

### 决策依据

- D3 第 5 点。
- 不变量 5。

## Losing The Provider Must Turn Selection Into An Intent

### 风险

视图不知道会话已失去 provider，选择器会继续向已停止的会话发设置请求；按选项是否为空判断，又会丢掉只读标签，或把回合中的会话误判为没有 provider。

### 前置状态

- 会话有选项，随后分别：停止、空闲时队列溢出、连接断开（包括 Agent 进程被替换）。插件仍可用。
- 另有一个工作流节点的只读选择器，以及一个回合进行中的会话。

### 触发

失去 provider；用户选模型；再发送一条 prompt 重新附着；另在没有 provider 时直接发设置请求；另在回合、MCP 刷新、标题轮询、附着进行中发查询。

### 必须成立

- 发出广播；查询按是否有通道回答，答「没有 provider」。
- 视图记下这一状态，不清空最后的选项；可设置的模型选择器只记下次附着的模型意图，不发设置请求，不报错；推理强度不记意图。
- 只读选择器仍显示最后的模型；没有 provider 时隐藏推理强度控件，只读节点仍显示。
- 重新附着后流开头带选项，视图解除这一状态，意图按附着的规则应用。
- 没有 provider 时到达的设置返回失败。
- 回合、MCP 刷新、标题轮询期间查询立即答当前选项；附着等不读命令的步骤完成后答复。

### 禁止结果

- 停止的会话里选模型报错。
- 被操作占用的通道被当成没有 provider。
- 只读选择器改为显示发现目录里的默认模型。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| 停止、空闲溢出后广播，查询答「没有 provider」（新增） | [a_session_without_a_provider_reads_as_none](https://github.com/ora-space/desktop/blob/d35f4d1a636d1f64cdfaeaae2064d2b7c79f557f/crates/agent-runtime/src/restored_sessions/set_config_tests.rs#L1124) 停止广播+读 `none`；[a_session_that_overflows_while_idle_reads_as_none](https://github.com/ora-space/desktop/blob/d35f4d1a636d1f64cdfaeaae2064d2b7c79f557f/crates/agent-runtime/src/restored_sessions/set_config_tests.rs#L1193) 空闲溢出广播+读 `none` | `Covered` |
| 连接断开（包括 Agent 进程被替换）后广播，查询答「没有 provider」（新增） | 尚无直接证据。断连收尾用例只验证在途设置返回失败 | `Missing` |
| 可设置的模型选择器只记意图、不发设置请求（复现） | [records a pick as an intent although the session kept its last options](https://github.com/ora-space/desktop/blob/d35f4d1a636d1f64cdfaeaae2064d2b7c79f557f/packages/app-shell/src/features/chat/model-selector.test.tsx#L345)、[records a pick as an intent instead of configuring an absent provider](https://github.com/ora-space/desktop/blob/d35f4d1a636d1f64cdfaeaae2064d2b7c79f557f/packages/app-shell/src/features/chat/model-selector.test.tsx#L248)、[records a pick that finds the provider gone as an intent](https://github.com/ora-space/desktop/blob/d35f4d1a636d1f64cdfaeaae2064d2b7c79f557f/packages/app-shell/src/features/chat/model-selector.test.tsx#L370) 竞态记意图、不报错 | `Covered` |
| 只读选择器保留最后的模型（回归保护） | [keeps naming the last model in a read-only picker](https://github.com/ora-space/desktop/blob/d35f4d1a636d1f64cdfaeaae2064d2b7c79f557f/packages/app-shell/src/features/chat/model-selector.test.tsx#L406) | `Covered` |
| 没有 provider 时隐藏推理强度控件，只读节点仍显示（新增） | [withdraws once the session is known to have no provider](https://github.com/ora-space/desktop/blob/d35f4d1a636d1f64cdfaeaae2064d2b7c79f557f/packages/app-shell/src/features/chat/thought-level-selector.test.tsx#L425)、[keeps showing a read-only level once the session is known to have no provider](https://github.com/ora-space/desktop/blob/d35f4d1a636d1f64cdfaeaae2064d2b7c79f557f/packages/app-shell/src/features/chat/thought-level-selector.test.tsx#L400) | `Covered` |
| 加载、预览、失败分支保留已知的「没有 provider」，流送来选项才解除（新增） | [a load that brings no options keeps a lost provider lost, whether it completes or fails](https://github.com/ora-space/desktop/blob/d35f4d1a636d1f64cdfaeaae2064d2b7c79f557f/packages/chat/tests/options-sync.test.ts#L270)、[replaying an unfinished turn keeps a lost provider lost](https://github.com/ora-space/desktop/blob/d35f4d1a636d1f64cdfaeaae2064d2b7c79f557f/packages/chat/tests/options-sync.test.ts#L292)、[options a stream brings say the session has a provider again, even if the load then fails](https://github.com/ora-space/desktop/blob/d35f4d1a636d1f64cdfaeaae2064d2b7c79f557f/packages/chat/tests/options-sync.test.ts#L250) | `Covered` |
| 重新附着后解除状态（新增） | [options a stream brings say the session has a provider again, even if the load then fails](https://github.com/ora-space/desktop/blob/d35f4d1a636d1f64cdfaeaae2064d2b7c79f557f/packages/chat/tests/options-sync.test.ts#L250)；[reads the options of a session without a stream when they are announced](https://github.com/ora-space/desktop/blob/d35f4d1a636d1f64cdfaeaae2064d2b7c79f557f/packages/chat/tests/options-sync.test.ts#L105) 读到选项解除 | `Covered` |
| 没有 provider 时的设置返回失败（新增） | [a_change_asked_without_a_provider_fails_and_asks_nothing](https://github.com/ora-space/desktop/blob/d35f4d1a636d1f64cdfaeaae2064d2b7c79f557f/crates/agent-runtime/src/restored_sessions/set_config_tests.rs#L572)；前端 [a change that finds the session without a provider is an intent, not an error](https://github.com/ora-space/desktop/blob/d35f4d1a636d1f64cdfaeaae2064d2b7c79f557f/packages/chat/tests/setting-options.test.ts#L216) | `Covered` |
| 回合、MCP 刷新、标题轮询期间立即答复，附着完成后答复（新增） | [the_options_are_read_at_once_during_a_title_poll_and_an_mcp_refresh](https://github.com/ora-space/desktop/blob/d35f4d1a636d1f64cdfaeaae2064d2b7c79f557f/crates/agent-runtime/src/restored_sessions/set_config_tests.rs#L1220)、[the_options_are_read_without_waiting_for_a_title_poll](https://github.com/ora-space/desktop/blob/d35f4d1a636d1f64cdfaeaae2064d2b7c79f557f/crates/agent-runtime/src/restored_sessions/set_config_tests.rs#L1291)、[a_session_without_a_provider_reads_as_none](https://github.com/ora-space/desktop/blob/d35f4d1a636d1f64cdfaeaae2064d2b7c79f557f/crates/agent-runtime/src/restored_sessions/set_config_tests.rs#L1124) 附着进行中读等待 | `Covered` |

### 决策依据

- D1 第 7 点，D3 第 3、4、6 点。

## The Initiating View Must Finish On The Fetched Options

### 风险

返回值不带选项后，选择器过早结束「进行中」会闪回旧值，等不到选项则一直转；标记的来源改变后，可能漏记或重复记。

### 前置状态

- 会话有 provider 且至少有一个回合；分别空闲、有流。
- Agent 分别答复新值、仍答复原值。

### 触发

用户切换模型，通知丢失或晚到；设置在途时停止会话；另在没有设置在途时由 Agent 自己改模型。

### 必须成立

- 空闲时成功后主动取一次，取到才结束「进行中」，不依赖通知；有流时，成功且请求发出后从流收到过选项就结束。
- 有流且 Agent 仍答复原值时，流照样送来选项，视图结束。
- `Stop` 提前结算为成功时，视图得知「没有 provider」就结束；成功先于流结束到达、广播又丢失时，流结束后取一次得知。
- 设置成功后的那次读失败也结束「进行中」，不重试。
- 结束时模型和发起前不同，记一次「已切换模型」标记；没有设置在途时，Agent 自己改的模型不记。
- 重叠设置共用这组首个设置发起前的模型作基准，标记按模型判断；替换和撤掉只动这组自己留下的标记，不影响之前已结束设置在同一位置的标记。
- 失败只结束该次设置，不改变基准和标记；组内所有设置都结束后，下次设置才重新取基准。

### 禁止结果

- 空闲时闪回旧值再显示新值。
- 设置已处理完仍一直「进行中」。
- 同一次切换记两次标记。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| 空闲时成功后主动取一次，通知丢失也不停在旧值（新增） | [a change on an idle session is in progress until a read after it answers](https://github.com/ora-space/desktop/blob/d35f4d1a636d1f64cdfaeaae2064d2b7c79f557f/packages/chat/tests/setting-options.test.ts#L122) | `Covered` |
| 得到选项后才结束「进行中」（回归保护） | [keeps the thumb on the chosen level while the provider answers](https://github.com/ora-space/desktop/blob/d35f4d1a636d1f64cdfaeaae2064d2b7c79f557f/packages/app-shell/src/features/chat/thought-level-selector.test.tsx#L561)；[a change during a stream ends on the options the stream brings, also unchanged ones](https://github.com/ora-space/desktop/blob/d35f4d1a636d1f64cdfaeaae2064d2b7c79f557f/packages/chat/tests/setting-options.test.ts#L144)、[a change on an idle session is in progress until a read after it answers](https://github.com/ora-space/desktop/blob/d35f4d1a636d1f64cdfaeaae2064d2b7c79f557f/packages/chat/tests/setting-options.test.ts#L122) | `Covered` |
| 有流且 Agent 答复仍是原值时结束（新增） | [a change during a stream ends on the options the stream brings, also unchanged ones](https://github.com/ora-space/desktop/blob/d35f4d1a636d1f64cdfaeaae2064d2b7c79f557f/packages/chat/tests/setting-options.test.ts#L144) 送来原值也结束 | `Covered` |
| `Stop` 提前结算为成功时，得知「没有 provider」就结束（新增） | [a change settled by a stop ends once the session is known to have no provider](https://github.com/ora-space/desktop/blob/d35f4d1a636d1f64cdfaeaae2064d2b7c79f557f/packages/chat/tests/setting-options.test.ts#L204) | `Covered` |
| 成功先于流结束到达、广播丢失时，流结束后补取一次并结束；已在成功之后取的不重复取（新增） | [a change made just before its stream ends is told by the read after the stream](https://github.com/ora-space/desktop/blob/d35f4d1a636d1f64cdfaeaae2064d2b7c79f557f/packages/chat/tests/setting-options.test.ts#L181)、[a read that started before the change was made does not end it](https://github.com/ora-space/desktop/blob/d35f4d1a636d1f64cdfaeaae2064d2b7c79f557f/packages/chat/tests/setting-options.test.ts#L315) | `Covered` |
| 设置成功后的读失败也结束「进行中」，不重试（新增） | [a change whose read after it is lost ends anyway, on the options the view had](https://github.com/ora-space/desktop/blob/d35f4d1a636d1f64cdfaeaae2064d2b7c79f557f/packages/chat/tests/overlapping-changes.test.ts#L122) | `Covered` |
| 结束时模型和发起前不同才记一次标记，没有设置在途时 Agent 自己改的模型不记（回归保护） | [a change on an idle session is in progress until a read after it answers](https://github.com/ora-space/desktop/blob/d35f4d1a636d1f64cdfaeaae2064d2b7c79f557f/packages/chat/tests/setting-options.test.ts#L122) 移动才记一次；[a model the agent changes on its own, with no change in flight, is not marked](https://github.com/ora-space/desktop/blob/d35f4d1a636d1f64cdfaeaae2064d2b7c79f557f/packages/chat/tests/setting-options.test.ts#L229)、[a model the agent changes mid-turn, with no change in flight, is not marked](https://github.com/ora-space/desktop/blob/d35f4d1a636d1f64cdfaeaae2064d2b7c79f557f/packages/chat/tests/overlapping-changes.test.ts#L110) | `Covered` |
| 重叠设置共用这组首个设置发起前的模型作基准，回到基准的模型撤掉这组自己的标记（新增） | [two quick changes that come back to the first model leave no stale marker](https://github.com/ora-space/desktop/blob/d35f4d1a636d1f64cdfaeaae2064d2b7c79f557f/packages/chat/tests/overlapping-changes.test.ts#L89)、[overlapping changes are marked against the options before the first of them](https://github.com/ora-space/desktop/blob/d35f4d1a636d1f64cdfaeaae2064d2b7c79f557f/packages/chat/tests/overlapping-changes.test.ts#L138)、[three changes in flight are marked once, against the options before the first](https://github.com/ora-space/desktop/blob/d35f4d1a636d1f64cdfaeaae2064d2b7c79f557f/packages/chat/tests/overlapping-changes.test.ts#L202) | `Covered` |
| 替换和撤掉只动这组自己的标记，不影响之前已结束设置同位置的标记；放回前按位置判断（新增） | [a group that comes back to its baseline leaves the mark of an earlier change in place](https://github.com/ora-space/desktop/blob/d35f4d1a636d1f64cdfaeaae2064d2b7c79f557f/packages/chat/tests/overlapping-changes.test.ts#L178)、[a mark an agent switch left at the same point is not covered by one the group put back](https://github.com/ora-space/desktop/blob/d35f4d1a636d1f64cdfaeaae2064d2b7c79f557f/packages/chat/tests/overlapping-changes.test.ts#L251) | `Covered` |
| 失败只结束该次设置，组内仍在途的设置沿用原基准；组内全部结束后，下次设置才重新取基准（新增） | [a change that fails still ends its group, and the next group takes its own baseline](https://github.com/ora-space/desktop/blob/d35f4d1a636d1f64cdfaeaae2064d2b7c79f557f/packages/chat/tests/overlapping-changes.test.ts#L220) 验证同组另一设置成功后记标记、整组结束后新设置重新取基准。尚未覆盖首个设置失败、另一个仍在途时发起第三个设置，且当时模型已偏离原基准的情形 | `Partial` |
| 切换 Agent 时在携带切换的消息之前记标记（回归保护） | [marks an agent move before the message that carried it](https://github.com/ora-space/desktop/blob/d35f4d1a636d1f64cdfaeaae2064d2b7c79f557f/packages/chat/tests/store.test.ts#L2580) | `Covered` |

### 决策依据

- D3 第 1、3、7 点。