# Agent 运行的协议消息

依据[已批准的 Node 协议后续决策](../../../decisions/node/protocol/20260928-streamed-thread-events-and-session-commands.md)。
消息与校验已在 desktop `crates/node-protocol` 合并（desktop#629），证据来自
`crates/node-protocol/tests/protocol/{plugin,agent_session,revision}.rs`。协议往返测试只证明线上形状，
持久化、重放与恢复由 Node/Controller 测试承担，因此相应义务为 `Partial` 或 `Missing`。
2026-10-12 获批的 [node/revision Git 交付决策](../../../decisions/node/revision/20261012-push-delivery-ref-and-fetch-prior-revision.md) D8
修改了交付输入、会话前序、结果与授权消息，下文“Upload Grants Stay In Memory And Deliveries Stay In Their Namespace”
已按它改写；协议尚未修改，新增义务为 `Missing`。

## Thread Events Share One Gap Free Sequence With The Terminal Event

### 风险

Thread 事件与终态事件序号冲突或出现空洞，Controller 无法按序接管；Node 在持久化前发送事件，崩溃后丢段。

### 前置状态

会话执行正在运行。

### 触发

会话产生定型记录、超大记录、会话结束；Node 在事件确认前断线重连。

### 必须成立

- Thread 事件是带序号的普通事件，与终态事件共用执行内序号，终态之后没有事件。
- 记录是 JSON 对象，编码后不超过 256 KiB；超大记录以截断版本发送并标注 `truncated`。
- 事件在账本持久化后才发送，未确认前保留，重连后按原身份重放；同一执行在途不超过 256 个。

### 禁止结果

记录不是对象或超限仍被编码；终态后仍有事件；未持久化即发送。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 线上形状与记录上限 | `agent_session.rs` `thread_events_round_trip_and_bound_the_record`：非对象记录解码失败，超过 `MAX_THREAD_RECORD_BYTES` 被双向拒绝 | Partial |
| 终态在结果族中可区分 | `agent_session.rs` `session_end_round_trips_as_event_and_completed_status`：终态事件与 Completed 查询都解码为会话结果族，自由文本 `detail` 被拒绝 | Partial |
| 持久化、窗口与重放 | `exact_ack_and_sequence_survive_reopen`、`failed_event_rolls_back_all_session_effects` 证明账本事务；`window_bounds_delivery_without_blocking_queries_and_reconnect_replays_exactly` 通过真实 Node 验证 256 窗口、重复 ACK 不多释放位置、窗口满仍响应查询、重开按原内容重放到终态 | Covered |

### 决策依据

协议后续决策 D2、D3，不变量 2、3。

## Session Commands Execute At Most Once In Acceptance Order

### 风险

命令重复投递导致用户轮次执行两次；会话结束后的命令被执行。

### 前置状态

会话执行正在运行或已结束。

### 触发

`SubmitUserTurn`、`EndSession` 首次与重复发送；同一 `command_id` 不同内容；会话已终态时发送。

### 必须成立

- 命令持久化后回复受理；回复无序号、无需确认。
- 同一 `command_id` 重发只回复受理；不同内容为协议错误；已终态时回复 `session_ended` 拒绝。
- 用户轮次按受理顺序执行；`EndSession` 丢弃排队轮次。

### 禁止结果

同一命令执行两次；终态后执行命令。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 命令与回复形状 | `agent_session.rs` `session_commands_and_replies_round_trip`：四种消息往返，`command_id` 必填且非空 | Partial |
| 去重、顺序与终态拒绝 | `commands_are_ordered_deduplicated_and_discarded_at_end` 证明持久队列规则；`session_commands_use_the_production_ledger_and_installer_catalog` 通过真实 IPC 验证重复受理回复只执行一次、终态后回复拒绝及 JSONL 与 Thread 一致 | Covered |
| 运行许可与关闭 | `controlled_session_start_round_trips_and_binds_exact_identity` 验证许可身份；`controlled_start_and_command_closure_use_original_runtime_authority` 验证真实 Node 受控启动及关闭后拒绝命令；`session_command_authority_expires_and_closes_without_losing_existing_work` 验证账本过期、刷新与 incarnation 隔离 | Covered |

### 决策依据

协议后续决策 D4，不变量 4。

## Upload Grants Stay In Memory And Deliveries Stay In Their Namespace

### 风险

预签名 URL 进入账本或日志；交付写到分支或其他对象键。

### 前置状态

交付执行正在运行。

### 触发

收到 `UploadGrant`；授权过期或 Node 重启后需要上传；交付输入的 `revision_ref` 不在 `refs/ora/revisions/` 下、
`delivery_ref` 不在 `refs/ora/deliveries/` 下或 JSONL 键非规范；会话前序缺少仓库地址、交付 ref 或基础提交。

### 必须成立

- `UploadGrant`/`UploadGrantNeeded` 无序号、无确认，不写账本与日志；`Debug` 不输出 URL。
- 没有有效授权时发送 `UploadGrantNeeded` 并等待，执行保持 `Running`。
- `DeliverRevision` 的 `revision_ref` 必须在 `refs/ora/revisions/` 下，`delivery_ref` 必须在 `refs/ora/deliveries/` 下，
  同时带 `git_repository_url`，不带 `bundle_key`；JSONL 键规范。`RevisionDelivered` 不带 bundle；`unchanged` 与
  `delivered` 与 commit 是否变化一致；失败码有 `push_rejected`、`push_failed`，没有 `bundle_failed`。
- 会话的 `PriorRevision` 带 `git_repository_url`、`delivery_ref`、`base_commit`，不带 bundle；交付的
  `PriorRevisionCommit` 不变。
- `UploadGrantNeeded` 的 `checksums` 只含 JSONL 键；没有 `DownloadGrantNeeded` 与 `DownloadGrant`。

### 禁止结果

授权持久化或出现在日志；交付移动分支；交付推送到 `refs/ora/deliveries/` 之外的 ref。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 授权脱敏与形状 | `revision.rs` `upload_grants_round_trip_without_exposing_the_url`；`UploadGrantNeeded` 的 checksums 必须非空、键规范、摘要为小写规范形式（同文件 upload grant needed 用例），`ControlledDeliverRevision` 的许可必须精确对应执行、node operation 与 Node（`controlled_delivery_requires_a_matching_runtime_permit`）；desktop `apps/ora-controller/tests/workspaces/deliveries.rs::delivery_waits_for_its_permit_and_relays_checksum_bound_grants_unlogged` 以作用域内的记录型 subscriber 捕获 Controller 全部日志，断言其中没有存储主机与签名；Node 侧日志尚无同等断言 | Partial |
| ref 与对象键校验 | `revision.rs` `deliver_revision_rejects_refs_keys_and_commits_outside_the_contract`、`revision_results_round_trip_and_keep_commits_consistent`：`revision_ref` 越界、对象键非规范与 commit 不一致被拒绝。缺口：`delivery_ref`、`git_repository_url` 与去掉 bundle 后的形状尚无测试 | Partial |
| 会话前序与失败码的形状 | `revision_restore.rs` `prior_revision_round_trips_in_session_and_delivery_inputs` 证明的是带 bundle 的旧前序；新字段、`push_rejected`、`push_failed` 尚无测试 | Missing |
| 没有下载授权消息 | `revision_restore.rs` `download_grants_round_trip_without_exposing_the_url` 仍证明下载授权存在；删除后需由协议测试证明这些消息无法解码 | Missing |
| 授权不落盘、过期重请求 | desktop `apps/ora-node/tests/standalone/agent_sessions/delivery.rs::ended_session_is_delivered_with_requested_grants`：真实 Node 二进制完成交付后，账本文件字节中不含签名 URL；`apps/ora-node/src/revision/tests/upload.rs::grant_requests_carry_frozen_checksums_and_headers_travel_verbatim`：过期授权被丢弃并以冻结的 checksums 重新请求，`forbidden_grant_is_replaced`：403 后重新请求；Node 日志无授权尚未直接断言，Controller 重启后由 Node 在新连接上重发请求尚无真实重连测试 | Partial |

### 决策依据

协议后续决策 D1、D5，不变量 5、6；node/revision Git 交付决策 D8。
