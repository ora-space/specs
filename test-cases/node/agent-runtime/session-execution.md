# Node Agent 会话执行

依据[已实现的 agent-runtime 根决策](../../../decisions/node/agent-runtime/0-shared-agent-runtime-crate-hosted-by-node.md)。
证据来自 `ora-agent-runtime` 单元测试与 `apps/ora-node/tests/agent_session.rs`：后者通过 `SessionHost` 驱动真实的
echo agent 插件进程，以内存 `SessionLedger`、固定 checkout 与计数租约的插件目录代替账本、clone 账本与插件安装。
账本持久化另由 `crates/node-db/src/tests/session.rs`、`session/recovery.rs` 与
`apps/ora-node/tests/session_ledger.rs` 的真实 SQLite 测试证明；生产服务接线与启动恢复由 `apps/ora-node/tests/standalone/agent_sessions.rs` 及子模块的真实 Node 进程测试验证。
强杀后任意后代进程的 scope 清理仍受 ADR D2 偏离的限制。

## Session Records Reach JSONL Before The Thread

### 风险

Thread 中出现 JSONL 里没有的记录，或崩溃后两边不一致且无法解释。

### 前置状态

Node 运行 echo agent 的会话执行。

### 触发

会话产生多条记录；在追加 JSONL 之后、`append_thread_event` 之前注入崩溃；Node 重启。

### 必须成立

- 每条 Node 会话记录先追加 JSONL，再经 `append_thread_event` 持久化，之后才发送。Cloud 终态事务生成的 `source=system/session_failed` 条目是独立平台事实，不伪造成 Node 原始记录。
- 崩溃后 Thread 至多缺最后一条，JSONL 完整；重启时会话以 `interrupted` 结束，记录已封口。
- 用户消息记录携带 `turn_id`。

### 禁止结果

Thread 的 Node 来源记录在 JSONL 中不存在；平台失败条目混入 Node JSONL；重启后恢复出运行中的会话。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 记录顺序 | `records_reach_the_thread_as_history_lines_attributed_to_their_turns`：Thread 记录逐条等于 JSONL 行、顺序一致，用户消息的 `messageId` 与事件 `turn_id` 等于轮次 ID；`ora-agent-runtime` 的 `settled_lines_reach_the_host_exactly_as_the_file_holds_them` 证明回调行与文件一致 | Covered |
| 崩溃窗口 | `a_crash_before_the_thread_append_leaves_the_history_one_line_ahead`：在 Thread 追加处阻塞模拟崩溃点，JSONL 恰比 Thread 多一行；新的 `AgentSessions`（模拟重启）`recover_interrupted` 返回 `interrupted`，`sealed_history` 返回该 JSONL | Partial：`killed_session_replays_and_ends_before_accepting_commands` 已覆盖真实 Node 强杀、原始事件重放和新命令前中断结算；任意后代的 scope 清理仍未接入 |

### 决策依据

agent-runtime D3、D6、D7，不变量 4、7；协议后续决策 D6。

## Sessions Start Only Their Plugin With Identity Confined To Its Process Tree

### 风险

Node 启动了未请求的插件、错误版本的插件，或把 git 身份写进仓库配置影响后续交付。

### 前置状态

Workspace 插件根装有 agent 插件与其他插件。

### 触发

会话以正确版本、错误版本、未安装的插件启动；Agent 进程执行 `git commit`。

### 必须成立

- 只启动指定版本的 agent 插件，会话结束时插件进程组整树终止；不符或未安装时以 `agent_failed{agent_plugin_unavailable}` 结束。
  脱离进程组的后代由尝试环境标记覆盖，其义务与证据见[会话 MCP 刷新与宿主健康用例](session-mcp-lifecycle.md#a-session-stop-kills-every-attempt-marked-process-and-finished-turns-kill-none)，本用例的证据不推导它。
- Agent 进程树的提交使用 `GIT_AUTHOR_*`/`GIT_COMMITTER_*`，checkout 的 `.git/config` 与全局配置不变。
- 会话期间持有插件租约，结束后释放。
- 配置了工作负载用户时，插件进程与宿主为它启动的全部子进程以工作负载 UID/GID 运行（无附加组、`no_new_privs`、
  无能力），从会话目录中的只读包视图启动、HOME 在会话目录内；会话结束后会话目录被删除。

### 禁止结果

启动其他插件；修改 git 配置；会话结束后租约仍被持有；配置了工作负载用户时任何 Agent 进程以 root 运行、能读到
Node home 或管理密钥，或身份切换失败后仍以原身份执行。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 插件选择 | `a_plugin_of_another_version_ends_the_session_without_starting_it`：目录无该版本、或插件根中的包不在该版本目录时，以 `agent_plugin_unavailable` 结束，没有插件进程启动，租约已释放 | Covered |
| git 身份隔离 | `agent_commits_carry_the_session_identity_without_touching_git_config`：经宿主 `ora/childprocess/spawn` 与插件直接派生的两次提交，author/committer 都是输入身份，`.git/config` 逐字节不变 | Covered |
| 租约 | `ending_cancels_the_running_turn_and_discards_queued_turns` 验证结束后租约归零；`session_commands_use_the_production_ledger_and_installer_catalog` 验证生产安装器与会话共享租约，活跃插件移除返回 `plugin_in_use` | Covered |
| 执行进程 scope | 插件进程与 Agent CLI 纳入该执行的 host/guardian 进程 scope | Missing：插件由 `ora-process` 直接启动，崩溃后的后代回收是 ADR D2 记录的偏离，留在“本决策未解决的问题” |
| 工作负载身份 | `ora-process` 的 `an_identity_the_parent_cannot_take_fails_the_spawn`（无法切换身份时启动失败、不留子进程）与 `a_root_parent_drops_the_whole_tree_to_the_identity`（root 父进程下整树为目标 UID/GID、无附加组、`NoNewPrivs`、`CapEff` 为 0、umask `0077`，整树可终止）；`apps/ora-node` 的 `a_separate_workload_runs_the_agent_from_its_session_directory`（插件 cwd 为包视图、HOME 为会话 home，结束后目录删除）、`the_view_launcher_moves_the_launch_into_the_package_view`、`the_session_environment_reaches_the_plugin_and_its_host_spawned_processes`、`purging_removes_every_leftover_without_following_links` | Partial：需要 root 的测试标记为 `#[ignore]`，CI 不运行（2026-10-09 在 root 容器中手工运行通过）；ora-node 测试以当前用户身份运行，未经真实降权；cluster `task real-agent:acceptance` 在真实栈上核对 opencode 进程树为 UID 1000，只作补充证据 |

### 决策依据

agent-runtime D2、D4、D7，不变量 3、5、8。

## Session Commands Run Once In Acceptance Order And An End Discards The Queue

### 风险

用户轮次乱序、重复执行或被遗漏；`EndSession` 等到排队轮次跑完才生效，或结束后仍有轮次运行、命令未结算。

### 前置状态

Node 运行 echo agent 的会话执行，首个轮次尚未结束或正在运行。

### 触发

在首个轮次运行期间受理多个 `SubmitUserTurn`，并多次唤醒会话；或在轮次运行期间先后受理 `SubmitUserTurn` 与
`EndSession`。

### 必须成立

- 用户轮次在当前轮次结束后按受理顺序执行，每个命令恰好结算一次为 `executed`，重复唤醒不重复执行。
- `EndSession` 立即取消进行中的轮次（记录 `TurnEnded{cancelled}`），排在它之前的用户轮次结算为 `discarded` 且不运行，
  自身结算为 `executed`；终态为请求的原因，插件进程退出、租约释放后才写入终态。

### 禁止结果

结束后仍有轮次运行；同一命令被结算两次；轮次顺序与受理顺序不同。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 顺序与至多一次 | `queued_turns_run_once_each_in_acceptance_order` 验证轮次顺序；`session_commands_use_the_production_ledger_and_installer_catalog` 验证生产 IPC 重发同一命令只运行一次；`failed_command_settlement_never_runs_the_queued_turn` 验证 Executed 写入失败时不提交 prompt | Covered |
| 结束 | `ending_cancels_the_running_turn_and_discards_queued_turns` 验证取消、丢弃与租约释放；`service_shutdown_stops_the_active_agent_and_persists_its_end` 验证真实 Node 正常停止清理活跃 echo 插件后留下终态 | Covered |
| 规划规则 | `apps/ora-node` 的 `an_end_anywhere_in_the_queue_takes_precedence`：队列中任意位置的 `EndSession` 优先并只丢弃其前的轮次 | Covered |

### 决策依据

agent-runtime D6、D7；协议后续决策 D4，不变量 4。

## Provider Failure Ends The Session Without Publishing Its Diagnostic

### 风险

ACP 错误或提前结束被当作成功，Thread 继续显示空闲；直接传播上游错误又可能泄露请求或凭据。

### 触发与必须成立

实际插件返回 prompt 错误，或事件流在 Completed 前结束。Node 以有限类别 `agent_turn_failed` / `agent_stream_closed`
结束会话，不发布原始错误；停止插件并释放租约、模型授权。已持久受理的结束命令优先，保留取消原因。
Cloud 关闭 Thread、丢弃排队消息，并在终态事务内追加一次安全 System 失败条目；重放不追加第二次。
原始 Node JSONL 不改变，已有仓库成果仍走 Revision 交付和 Workspace 释放。

### 验证义务与证据

| 义务 | 直接证据 | 状态 |
|---|---|---|
| prompt 失败不再当作成功 | `provider_failure_ends_the_session_with_a_safe_code_and_releases_resources`：真实 echo 插件的 ACP 错误导致 AgentFailed、安全 detail、进程退出和租约归零 | Covered |
| 取消优先 | `durable_cancel_wins_a_racing_provider_failure`：阻塞记录追加，先持久受理取消但不唤醒，再放行 ACP 错误；最终 Cancelled | Covered |
| 持久失败与重放 | Cloud `TestSessionFailureProjectsOnlySafeCodes`：真实 PostgreSQL + gRPC，未知恶意 detail 收为 `agent_failed`、单一 System 条目、不可追加且重放不重复 | Covered |
| 实际 CLI 和浏览器失败链路 | 2026-10-11 两协议受控上游 HTTP 401 → Gateway 安全 502 → OpenCode ACP 错误；页面显示失败并禁止追加，同 Run 完成交付、撤销及容器/卷释放 | Covered：仅受控供应商故障；真实供应商和全栈崩溃演练仍未完成 |

## Durable Session Ledger Preserves Events Commands And Terminal State

### 风险

ACK 后复用事件序号，重启丢失命令，或终态只提交了一部分，导致继续执行已经结束的会话。

### 前置状态

Node 使用真实 SQLite，会话已受理或正在运行，存在待确认事件和排队命令。

### 触发

追加事件、乱序或重复 ACK、重复命令、结束会话、注入 outbox 写入失败，以及关闭数据库后重新打开。

### 必须成立

- 序号由事务分配并在提交后返回；ACK 只删除对应事件，保留序号和终态事实。
- 命令按受理顺序完整返回，同 ID 不同输入被拒绝；结算不可改变，重发不重新入队。
- 终态、最后一个事件和排队命令的丢弃原子提交；与结束并发的受理只能被丢弃或被拒绝。
- 句柄共享 Node 租约，多个 actor 连接不分配重复序号；v7 升级保留历史事实。
- checkout 只来自成功完成的 clone 账本，受控会话启动须重新检查 runtime 权限。

### 禁止结果

事件删除导致序号回退；结束后仍追加事件或留下 queued 命令；新句柄成为第二个 Node 权威；恢复时自动重启 Agent。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 持久序号与精确 ACK | `exact_ack_and_sequence_survive_reopen`：乱序 ACK 不删除其他事件，全部 ACK 后重开仍递增，终态保留 | Covered |
| 命令队列与终态 | `commands_are_ordered_deduplicated_and_discarded_at_end`、`terminal_write_serializes_with_command_admission`：持久顺序、单向结算、终态拒绝与受理竞态 | Covered |
| 事务回滚 | `failed_event_rolls_back_all_session_effects`：outbox 失败时状态、队列、序号均不改变 | Covered |
| 并发与租约 | `concurrent_journals_share_sequences_and_owner_lease`：独立连接的序号连续无重复，最后句柄释放前不能另开 Node | Covered |
| v7 兼容 | `migrates_v7_without_losing_existing_executions`：保留 plugin 终态、事件、Controller 归属并可新增会话；已有旧版迁移测试继续保留 | Covered |
| runtime 隔离 | `controlled_sessions_respect_closure_and_incarnation`：拒绝无 permit 入口、未完成责任冲突，关闭或换 incarnation 后不能启动 | Covered |
| checkout 来源 | `checkout_requires_a_successful_clone`、`checkout_rejects_unknown_and_failed_clone`：不存在、未完成、失败及非 clone 不提供路径，成功 clone 的原始目标在重开后保留 | Covered |
| 运行时接口 | `runtime_ports_use_durable_ledger`：通过生产 `SessionLedger`/`CheckoutResolver` trait 调用 SQLite，实现排队、结算、追加与终态 | Covered |
| 生产启动恢复 | `killed_session_replays_and_ends_before_accepting_commands`：真实 Node 强杀后，原始事件重放且只补一个 interrupted 终态，新命令被拒绝，echo 插件不重启 | Partial：echo 插件退出已证明，忽略 stdio 关闭的后代回收仍缺失 |

### 决策依据

agent-runtime D7；协议后续决策 D2、D3、D4、D6。
