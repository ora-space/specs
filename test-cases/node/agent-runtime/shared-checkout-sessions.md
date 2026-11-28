# Node 同一 checkout 的多个 Agent 会话

依据[共享 checkout 会话决策](../../../decisions/node/agent-runtime/20261115-shared-checkout-agent-sessions.md)（Node 侧 `implemented`；v2 按能力启用同 run 多会话由 v2 事项承接）。
Node 实现依据固定两个合并提交：desktop#774 [`c5265072`](https://github.com/ora-space/desktop/commit/c5265072ace476b6cdd88fc69fb58ffafec20066)（[desktop#774](https://github.com/ora-space/desktop/pull/774)）交付原测试，desktop#864 [`89b7ce37`](https://github.com/ora-space/desktop/commit/89b7ce375579d286f70b9bff17cf5475d9d64b22)（[desktop#864](https://github.com/ora-space/desktop/pull/864)）只补测试，补齐并发受理、未清理恢复尝试围栏、已结束恢复阶段与 `RLIMIT_NOFILE` 推导 4 个缺口。
“所需直接证据”保留原验证目标（路径相对 desktop 仓库根目录），每个用例末尾列出固定源码的代表性直接证据和不能由它推导的边界。
37 项验证义务全部 `Covered`，无 `Partial`、`Missing`。其中两项不是测试义务：`RLIMIT_NOFILE` 最低要求是文档义务（测试只固定文档里的最低值始终由遍历上限推导，不证明部署提供了该限制），混合版本边界是部署边界（不登记测试）。
[会话执行用例](session-execution.md)、[显式 Session MCP 用例](session-mcp.md)与
[恢复截止与停止确认用例](../revision/restore-deadline-and-stop-confirmation.md)的证据保持各自范围，不推导
本文新义务。

## Sessions On One Checkout Run And End Independently

### 风险

同一 checkout 上的第二个普通会话被拒绝或排队；结束一个会话时停止兄弟会话的插件、清理共享 checkout、
撤销兄弟的插件租约或改动其账本；两个会话共用执行身份、事件序号、会话目录或 MCP setup。

### 前置状态

一个 clone 执行已创建 checkout；Node 为非受控模式；Controller 在同一控制连接上发出多个不带
`prior_revision`、指向该 checkout 的 `start_agent_session`。

### 触发

在同一 checkout 上启动两个会话并分别运行回合；只对其中一个发送 `end_session`；一个会话选择 MCP、
另一个不选。

### 必须成立

- 恢复围栏允许时，新会话进入已有其他未结束会话的 checkout；每个会话使用自己的 execution id、插件进程、
  会话目录、事件序号、MCP setup 和终态。
- 结束会话 A 只停止 A 的插件、只释放 A 的租约；B 继续运行并接受回合，checkout 的 HEAD 与文件不因 A
  结束而被清理或改动。
- 交付与定时产物快照整个 checkout，可能包含存活兄弟的写入；这是明确接受的共享 Workspace 语义。

### 禁止结果

- 普通会话因兄弟存活被拒绝或排队；A 的终态、插件退出或会话目录清理改变 B 的进程、账本或 checkout
  归属；无 MCP 的兄弟收到另一会话的 MCP Server。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 两个存活会话独立运行与结束 | `apps/ora-node/tests/standalone/agent_sessions/admission.rs` 的 `two_live_sessions_run_turns_and_end_independently`：真实 Node 进程上两个会话各自应答回合，结束 A 只停止 A 的插件，B 仍运行，checkout HEAD 不变 | Covered |
| 各自的 MCP setup 与租约 | `apps/ora-node/tests/agent_session/mcp.rs` 的 `an_mcp_session_shares_a_checkout_with_a_plain_one`：无 MCP 的兄弟 `session/new` 不列 Server，结束 MCP 会话释放其包租约而兄弟保持 `Running` | Covered |
| 账本层共享准入 | `crates/node-db/src/tests/session/admission.rs` 的 `plain_sessions_share_a_checkout_until_the_named_limit`：同一 checkout 的普通会话在上限内均被受理 | Covered |

### 代表性证据

- [two_live_sessions_run_turns_and_end_independently](https://github.com/ora-space/desktop/blob/c5265072ace476b6cdd88fc69fb58ffafec20066/apps/ora-node/tests/standalone/agent_sessions/admission.rs#L205)：真实 Node 进程上同一 checkout 启动 A、B 两个会话并各自应答回合；结束 A 后恰好一个插件进程退出（B 的仍在运行），checkout HEAD 不变；B 继续应答后以 `UserEnded` 结束。
- [an_mcp_session_shares_a_checkout_with_a_plain_one](https://github.com/ora-space/desktop/blob/c5265072ace476b6cdd88fc69fb58ffafec20066/apps/ora-node/tests/agent_session/mcp.rs#L338)：MCP 会话与无 MCP 的兄弟共享 checkout，各自报告自己的 `session/new`（"MCP frame verified" 对 "MCP frame invalid"，兄弟不列 Server）；租约计数 4（两个 MCP 包加两个 Agent 包），结束 MCP 会话后降为 1、兄弟仍 `Running`；结束兄弟后降为 0。
- [plain_sessions_share_a_checkout_until_the_named_limit](https://github.com/ora-space/desktop/blob/c5265072ace476b6cdd88fc69fb58ffafec20066/crates/node-db/src/tests/session/admission.rs#L126)：账本层普通会话在上限内均被受理。
- 边界：交付与定时产物快照整个 checkout 是 D5 接受的共享 Workspace 语义，本文不另登记测试。

### 决策依据

D1，不变量 3；D5 的共享 Workspace 交付语义；继承根决策的每执行宿主隔离。

## Each Checkout Admits At Most Eight Unfinished Sessions

### 风险

共享目录无界累积插件进程；已结束会话仍占名额或终态不释放名额；上限按 Node 而不是按 checkout 计数，
使一个满的 checkout 拒绝其他 clone 的会话；并发受理依据过期计数越过上限；重发重复计数。

### 前置状态

某 checkout 已有若干未结束会话；Node 定义 `MAX_SESSIONS_PER_CHECKOUT = 8`；另有属于其他 clone 的
checkout。

### 触发

在同一 checkout 上依次受理第 1 至第 9 个普通会话；结束其中一个后再受理；在另一 checkout 上受理会话；
重发已受理的启动；并发受理多个启动。

### 必须成立

- 已有 8 个未结束会话时，第 9 个新会话以 `checkout_session_limit` 拒绝；一个会话写入终态后释放恰好
  一个名额；不同 checkout 分别计数。
- 围栏与数量在持久化受理的同一 `IMMEDIATE` 事务内判定并写入；并发受理路径不能依据已被另一方改变的
  计数准入。
- 持久投影相同的输入重发只返回原持久状态，不重复计数或启动。
- 上限是 Node 本地命名常量并写入文档，不扩展 `Accepted` 字段，不构成全 Node 会话上限、模型并发额度
  或调度公平性承诺。

### 禁止结果

- 第 9 个会话被受理；已结束会话占名额；满的 checkout 拒绝其他 checkout 的会话；并发受理使未结束会话
  超过 8 个；重发被计为新会话。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 8 个上限与终态释放名额 | `crates/node-db/src/tests/session/admission.rs` 的 `plain_sessions_share_a_checkout_until_the_named_limit`：第 9 个以 `checkout_session_limit` 拒绝，一个终态后恰好再受理一个 | Covered |
| 按 checkout 分别计数 | 同文件 `limit_counts_each_checkout_separately`：满的 checkout 不拒绝其他 clone 的会话 | Covered |
| 并发受理不越过上限 | `crates/node-db/src/tests/session/admission.rs` 的 `concurrent_admissions_cannot_both_take_the_last_slot`：两个准入连接在 `Accept` 写边界会合、同时争夺最后一个名额，恰好一方被受理，另一方得到 `checkout_session_limit` | Covered |

### 代表性证据

- [plain_sessions_share_a_checkout_until_the_named_limit](https://github.com/ora-space/desktop/blob/c5265072ace476b6cdd88fc69fb58ffafec20066/crates/node-db/src/tests/session/admission.rs#L126)：第 9 个以 `checkout_session_limit` 拒绝，拒绝记录与序号 1 的终态只写一次，重发返回同一记录；结束一个后恰好再受理一个。上限由 [`MAX_SESSIONS_PER_CHECKOUT = 8`](https://github.com/ora-space/desktop/blob/c5265072ace476b6cdd88fc69fb58ffafec20066/crates/node-db/src/session/admission.rs#L5) 定义。
- [limit_counts_each_checkout_separately](https://github.com/ora-space/desktop/blob/c5265072ace476b6cdd88fc69fb58ffafec20066/crates/node-db/src/tests/session/admission.rs#L171)：满的 checkout 不拒绝其他 clone 的会话。
- [concurrent_admissions_cannot_both_take_the_last_slot](https://github.com/ora-space/desktop/blob/89b7ce375579d286f70b9bff17cf5475d9d64b22/crates/node-db/src/tests/session/admission.rs#L436)：会合式 `WriteGuard` 让两个准入连接在 `Accept` 写边界同时到达 `BEGIN IMMEDIATE`，3 个 checkout 各跑一次，结果恒为一方 `Accepted`、一方 `checkout_session_limit`。把 `checkout_refusal` 挪到事务之前时两条都被受理、事务改成 `Deferred` 时输家得到 `database is locked`，两种变异都被杀死（owner 在 `62a8dc43` 上复核）。

### 决策依据

D1 的数量上限，不变量 1、2、5。

## Restore Keeps The Checkout Exclusive And Stop Evidence Keeps Fencing It

### 风险

恢复在其他 Agent 仍写树时强制移动分支与工作区；普通会话在兄弟恢复尚未获准启动 Agent 时进入；
`legacy_unknown`、未清理恢复尝试或未确认停止被普通共享绕过，或因原执行已有业务终态、停止等待上限
到期而释放；`UnknownIdentity` 被当成停止确认；判定顺序使恢复冲突被报告为数量上限。

### 前置状态

checkout C 上存在下列之一：带 `prior_revision` 且处于 `pending`、`git_admitted`、`restored` 或
`agent_admitted` 阶段的未结束会话；`legacy_unknown` 恢复；未清理的 `restore_process_attempts`；
未确认停止的恢复；或已结束历史中的恢复阶段。

### 触发

在 C 上启动带 `prior_revision` 的会话、普通会话；在 C 已满 8 个会话时启动带 `prior_revision` 的会话；
升级后在仍有 in-flight legacy 恢复的 C 上启动会话。

### 必须成立

- C 上任何执行有 `legacy_unknown` 恢复、未清理恢复尝试或未确认恢复停止时，以 `checkout_busy` 拒绝新会话，
  无论这些执行是否已有业务终态；停止确认沿用 20261021 的 `cleaned` 证据定义，`UnknownIdentity` 不释放围栏。
- 新会话带 `prior_revision` 且 C 上有任何未结束会话时以 `checkout_busy` 拒绝。
- C 上未结束的恢复会话尚未到达 `agent_admitted`（`pending`、`git_admitted`、`restored`）时，普通会话以
  `checkout_busy` 拒绝；到达 `agent_admitted` 后普通会话可以共享。
- 已结束历史中的任何恢复阶段（含 `pending`）不构成围栏；`checkout_busy` 的判定先于数量上限。
- 拒绝新会话的立即终态不解除原恢复的 D6 未知责任，也不改变原恢复的终态规则。

### 禁止结果

- 停止证据未知时新会话被受理或 checkout 被复用；恢复会话在兄弟存活时被受理；普通会话在恢复的
  `restored` 阶段被受理；恢复冲突被报告为 `checkout_session_limit`；拒绝新会话改变原恢复的终态或交付规则。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 恢复不在存活兄弟下进行 | `crates/node-db/src/tests/session/admission.rs` 的 `prior_revision_is_refused_while_a_sibling_is_unfinished` | Covered |
| 普通会话等待兄弟恢复的 `agent_admitted` | 同文件 `plain_session_is_refused_while_a_sibling_restore_is_pending`、`plain_session_waits_for_the_sibling_restores_agent_admission`：`pending` 至 `restored` 拒绝，`agent_admitted` 后受理 | Covered |
| `checkout_busy` 先于上限 | 同文件 `a_prior_revision_start_at_the_limit_reports_busy` | Covered |
| 未知停止证据继续围住 checkout | 同文件 `legacy_unknown_fences_a_plain_session`；`crates/node-db/src/tests/session/restore.rs` 的 `unknown_stop_retains_clock_terminal_and_checkout_responsibility`、`upgrade_keeps_inflight_legacy_restore_fenced`：替代会话改以 `checkout_busy` 拒绝 | Covered |
| 未清理 `restore_process_attempts` 独立围栏 | 同文件 `an_uncleaned_restore_attempt_keeps_fencing_after_its_execution_ended`：原执行已结束、停止上限到期标成 `stop_unconfirmed`、只剩未清理恢复尝试时普通会话仍以 `checkout_busy` 拒绝，`confirm_restore_stop` 之后才受理 | Covered |
| 已结束历史不构成围栏 | 同文件 `finished_restores_fence_nothing_whatever_phase_they_reached`：`pending`、`git_admitted`、`restored`、`agent_admitted` 各阶段结束前都拒绝普通会话，结束后以全新身份受理的普通会话均被受理 | Covered |
| 受控模式的单例先于 checkout 分类 | 同文件 `controlled_start_keeps_the_runtime_singleton_conflict`：受控启动仍以 `ResourceConflict` 拒绝 | Covered |

### 代表性证据

- [prior_revision_is_refused_while_a_sibling_is_unfinished](https://github.com/ora-space/desktop/blob/c5265072ace476b6cdd88fc69fb58ffafec20066/crates/node-db/src/tests/session/admission.rs#L214)：存活普通兄弟时带 `prior_revision` 的启动以 `checkout_busy` 拒绝并记终态，恢复阶段停在 `pending`。
- [plain_session_is_refused_while_a_sibling_restore_is_pending](https://github.com/ora-space/desktop/blob/c5265072ace476b6cdd88fc69fb58ffafec20066/crates/node-db/src/tests/session/admission.rs#L243)、[plain_session_waits_for_the_sibling_restores_agent_admission](https://github.com/ora-space/desktop/blob/c5265072ace476b6cdd88fc69fb58ffafec20066/crates/node-db/src/tests/session/admission.rs#L261)：兄弟恢复停在 `pending` 或已 `restored` 时普通会话被拒，`agent_admitted` 后受理；恢复未结束时另一个带 `prior_revision` 的会话仍被拒。
- [a_prior_revision_start_at_the_limit_reports_busy](https://github.com/ora-space/desktop/blob/c5265072ace476b6cdd88fc69fb58ffafec20066/crates/node-db/src/tests/session/admission.rs#L196)：满 8 个时带 `prior_revision` 的启动报 `checkout_busy` 而不是上限。
- [legacy_unknown_fences_a_plain_session](https://github.com/ora-space/desktop/blob/c5265072ace476b6cdd88fc69fb58ffafec20066/crates/node-db/src/tests/session/admission.rs#L302)：升级后 in-flight 恢复成为 `legacy_unknown`，不带 `prior_revision` 的普通会话仍以 `checkout_busy` 拒绝；[unknown_stop_retains_clock_terminal_and_checkout_responsibility](https://github.com/ora-space/desktop/blob/c5265072ace476b6cdd88fc69fb58ffafec20066/crates/node-db/src/tests/session/restore.rs#L69)（停止等待上限到期、重启后替代会话仍被拒，L105）、[upgrade_keeps_inflight_legacy_restore_fenced](https://github.com/ora-space/desktop/blob/c5265072ace476b6cdd88fc69fb58ffafec20066/crates/node-db/src/tests/session/restore.rs#L262)（升级后两道围栏都在，L283）。
- [an_uncleaned_restore_attempt_keeps_fencing_after_its_execution_ended](https://github.com/ora-space/desktop/blob/89b7ce375579d286f70b9bff17cf5475d9d64b22/crates/node-db/src/tests/session/admission.rs#L485)：原执行已结束、停止上限到期标成 `stop_unconfirmed`、只剩未清理的 `restore_process_attempts` 时普通会话仍以 `checkout_busy` 拒绝，`confirm_restore_stop` 后才受理。删掉 `OR EXISTS(restore_process_attempts … cleaned=0)` 的变异只被它杀死（原有用例里原执行仍 `running`，被第三个条件挡住）。
- [finished_restores_fence_nothing_whatever_phase_they_reached](https://github.com/ora-space/desktop/blob/89b7ce375579d286f70b9bff17cf5475d9d64b22/crates/node-db/src/tests/session/admission.rs#L578)：`pending`、`git_admitted`、`restored`、`agent_admitted` 四个阶段结束前都拒绝普通会话，各自结束后以全新身份受理的普通会话均被受理。删掉 `state<>'completed' AND` 的变异被杀死。
- [controlled_start_keeps_the_runtime_singleton_conflict](https://github.com/ora-space/desktop/blob/c5265072ace476b6cdd88fc69fb58ffafec20066/crates/node-db/src/tests/session/admission.rs#L328)：受控启动以 `ResourceConflict` 拒绝，不进入 checkout 分类。

### 决策依据

D2、D5；继承[恢复截止与停止确认](../../../decisions/node/revision/20261018-complete-restore-deadline-and-cancel-confirmation.md)
D6 与[恢复停止确认接受 `NotStarted`](../../../decisions/node/revision/20261021-stop-confirmation-accepts-not-started.md)。

## A Checkout Refusal Ends Only The New Execution

### 风险

checkout 拒绝复用 `ResourceConflict` 并关闭控制连接，使所有兄弟会话传输失败；拒绝终态与事件非原子提交，
或在插件、恢复或 checkout 写入之后才登记；重发被当作身份冲突而关闭连接或再次产生事件；被拒绝的执行
可交付，把存活兄弟的写入快照成它的 Revision；拒绝路径把 MCP Setting 值写进 ledger。

### 前置状态

Node 为非受控模式；某 checkout 因 D2 恢复／独占条件或 8 个会话上限而必须拒绝新启动；同一连接上已有
存活兄弟会话；被拒绝的启动可以携带带 Setting 值的 MCP 选择。

### 触发

发送会被以 `checkout_busy` 或 `checkout_session_limit` 拒绝的启动；以不同 Setting 值重发；令终态的
outbox 写入失败；对被拒绝的执行发送 `DeliverRevision`；之后继续向兄弟发送 `submit_user_turn` 与
`end_session`。

### 必须成立

- Node DB 以专门的存储错误类型区分两种拒绝，不复用 `ResourceConflict`；服务层只捕获这两种预期拒绝。
- 在任何插件、恢复或 checkout 写入之前，执行被登记为 Completed，并在同一事务中写入序号 1 的
  `AgentSessionEnded { reason: agent_failed, detail }`；返回现有 `ExecutionStatus`，不启动 actor，不新增
  消息；终态省略 `plugin_id`、`setting_id` 与 `transport`。
- 持久化的是与受理路径相同的、去除 MCP Setting 值而保留身份与版本的投影；原始控制帧和 Setting 值不进入
  ledger（含 WAL）、日志或会话历史。
- 重发持久投影相同的输入（Setting 值不参与比较）返回原持久终态，不再产生事件、启动 Agent 或关闭连接。
- 控制连接保持打开，兄弟会话继续接受命令；持久化终态本身失败时仍是 Node 内部错误，不伪称拒绝已记录。
- 被拒绝的执行没有会话历史，`DeliverRevision` 以 `session_not_settled` 拒绝，且在读取、推送或上传共享
  checkout 之前拒绝。

### 禁止结果

- 拒绝关闭控制连接或影响兄弟会话；终态与事件只提交一半；拒绝前出现插件进程、恢复尝试或 checkout 修改；
  重发以身份冲突关闭连接或产生第二个事件；Setting 值出现在任何持久文件；被拒绝的执行交付出 Revision。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 连接与兄弟保持可用、重发返回原终态 | `apps/ora-node/tests/standalone/agent_sessions/admission.rs` 的 `checkout_refusals_keep_the_connection_and_sibling_sessions_alive`：两种代码各有立即状态与序号 1 的终态，树未改动，以不同 Setting 值重发返回同一终态，兄弟继续应答，Node 写下的全部文件无 Setting 值 | Covered |
| 终态与事件原子提交 | `crates/node-db/src/tests/session/admission.rs` 的 `refusal_terminal_and_outbox_are_atomic`：outbox 写入失败时身份、终态与事件一并回滚 | Covered |
| 拒绝路径只持久化无值投影 | `apps/ora-node/tests/session_ledger.rs` 的 `session_mcp_settings_never_enter_the_durable_ledger`：拒绝记录等于清空 settings 的投影，不同值重发返回同一记录，数据库目录字节扫描无值，重开后可查到原终态 | Covered |
| 被拒绝的执行不可交付 | `apps/ora-node/tests/standalone/agent_sessions/delivery.rs` 的 `delivery_for_a_checkout_refused_session_is_refused`：两种拒绝类别均以 `session_not_settled` 失败，共享 checkout 未被读取、推送或上传 | Covered |
| 受控启动不进入拒绝路径 | `crates/node-db/src/tests/session/admission.rs` 的 `controlled_start_keeps_the_runtime_singleton_conflict` | Covered |

### 代表性证据

- [checkout_refusals_keep_the_connection_and_sibling_sessions_alive](https://github.com/ora-space/desktop/blob/c5265072ace476b6cdd88fc69fb58ffafec20066/apps/ora-node/tests/standalone/agent_sessions/admission.rs#L249)：两种代码各有立即状态与序号 1 的终态（`assert_refusal`，L129），树未改动；第 9 个启动带 Setting 值被拒后换值重发返回同一终态而不是身份冲突，兄弟继续应答、8 个插件进程都在；结束后递归扫描 Node 写下的全部文件无 Setting 值（`assert_no_secret`）。
- [refusal_terminal_and_outbox_are_atomic](https://github.com/ora-space/desktop/blob/c5265072ace476b6cdd88fc69fb58ffafec20066/crates/node-db/src/tests/session/admission.rs#L86)：outbox 写入失败时身份、终态与事件一并回滚（`find_session` 为空、待发事件为空），故障清除后重写得到序号 1。
- [session_mcp_settings_never_enter_the_durable_ledger](https://github.com/ora-space/desktop/blob/c5265072ace476b6cdd88fc69fb58ffafec20066/apps/ora-node/tests/session_ledger.rs#L125)：L154 起把第二个 checkout 填满后拒绝一个带 Setting 值的 MCP 启动，拒绝记录等于清空 settings 的投影（L184），换值重发返回同一记录，数据库目录字节扫描无值。
- [delivery_for_a_checkout_refused_session_is_refused](https://github.com/ora-space/desktop/blob/c5265072ace476b6cdd88fc69fb58ffafec20066/apps/ora-node/tests/standalone/agent_sessions/delivery.rs#L323)：两种拒绝类别的 `DeliverRevision` 均以 `session_not_settled` 失败，Git Server 的请求数不变（未读取、未推送），`revision-deliveries` 目录为空（未上传、未冻结）。
- [controlled_start_keeps_the_runtime_singleton_conflict](https://github.com/ora-space/desktop/blob/c5265072ace476b6cdd88fc69fb58ffafec20066/crates/node-db/src/tests/session/admission.rs#L328)：受控启动以 `ResourceConflict` 拒绝。

### 决策依据

D3，不变量 1–5；D2 的受控模式单例；继承[显式 Session MCP](../../../decisions/node/agent-runtime/20261114-explicit-session-mcp-in-agent-session-spec.md)
的无值持久投影与[流式 Thread 事件与会话命令](../../../decisions/node/protocol/20260928-streamed-thread-events-and-session-commands.md)
的既有终态与重放。

## Every Session Hands Over Its Own Checkout Through A Pinned No-Follow Walk

### 风险

后启动的会话依赖更早兄弟的交接（兄弟可能从未遍历就结束），Agent 以未交接的树启动；特权遍历重新解析
checkout 内的路径，存活兄弟把名称换成符号链接或另一个目录，使重属主越出 checkout；留在原处的硬链接使
外部 inode 被重属主；兄弟并发删除条目使交接失败；遍历持有的描述符泄漏给并发派生的进程。

### 前置状态

部署设置了 `process.workload_uid`（分离工作负载用户）；同一 checkout 上已有以工作负载用户身份运行、
持续改动树的兄弟 Agent；checkout 内可能含符号链接、硬链接与被并发删除的条目。

### 触发

启动新会话；在遍历期间把目录换成指向外部的符号链接、删除或重建文件；预先或在钉住前后植入指向外部文件
的硬链接；交付准备执行重属主步骤。

### 必须成立

- 每个会话在自己的 Agent 启动之前亲自完成交接遍历，不依赖兄弟；checkout 打不开时本次启动失败。
- 根目录以 `O_PATH | O_NOFOLLOW | O_DIRECTORY` 钉住，其下每个条目相对父目录描述符以
  `openat(O_PATH | O_NOFOLLOW)` 钉住；检查、重属主与列举只作用于钉住的 inode；符号链接按链接本身重属主，
  从不跟随。交付准备使用同一遍历。
- 钉住之前消失的条目（`ENOENT`）跳过；checkout 根必须作为目录存在；无权打开、列举或重属主的条目使启动
  失败（`agent_failed{agent_start_failed}`）或交付失败（`snapshot_failed`）。
- 链接数大于一、尚不属于工作负载用户的非目录条目保持原属主；链接数为一的 inode 照常重属主。这只挡住
  留在原处的链接，“钉住后删名”的竞争只由 `fs.protected_hardlinks = 1` 阻止。
- 遍历持有的每个描述符都是 close-on-exec。

### 禁止结果

- Agent 在本会话交接完成前启动；任何 checkout 之外的 inode 经符号链接或名称替换被重属主；留在原处的
  外部硬链接被重属主；兄弟的并发删除使交接失败；缺失的根被当作空树。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 每个会话亲自遍历 | `apps/ora-node/src/session/workload/tests.rs` 的 `every_separate_start_walks_its_checkout`：遍历打不开的 checkout 使本次启动失败，而不是信任更早会话的交接 | Covered |
| 钉住、不跟随、不越界 | `crates/utils/src/fs/owned_tree/tests.rs` 的 `walks_the_tree_without_following_links`、`reowns_every_entry_and_links_as_themselves`、`a_name_swapped_after_pinning_changes_nothing_outside`、`a_concurrent_symlink_swap_never_reowns_outside_the_root` | Covered |
| 并发删除不失败、根必须存在 | 同文件 `an_entry_removed_before_it_is_pinned_is_skipped`、`entries_removed_concurrently_do_not_fail_the_walk`、`the_root_must_exist_as_a_directory` | Covered |
| 无权条目报告失败 | `crates/utils/src/fs/owned_tree/tests/errors.rs` 的 `reports_entries_it_cannot_reown`、`reports_a_directory_it_cannot_list`、`reports_an_entry_it_cannot_pin`；启动侧 `apps/ora-node/tests/agent_session/workload.rs` 的 `a_checkout_the_walk_cannot_list_ends_the_session_as_a_start_failure`，交付侧 `apps/ora-node/src/revision/tests/snapshot.rs` 的 `a_failed_hand_over_fails_snapshot_failed` | Covered |
| 硬链接按链接数跳过及其边界 | `crates/utils/src/fs/owned_tree/tests/links.rs` 的 `a_hard_link_that_stays_in_place_is_not_reowned`、`a_hard_linked_file_keeps_its_owner`；`a_hard_link_removed_after_pinning_is_reowned` 固定该跳过不构成保证的边界 | Covered |
| 描述符不泄漏给子进程 | `crates/utils/src/fs/owned_tree/tests.rs` 的 `every_descriptor_the_walk_holds_is_close_on_exec` | Covered |

### 代表性证据

- [every_separate_start_walks_its_checkout](https://github.com/ora-space/desktop/blob/c5265072ace476b6cdd88fc69fb58ffafec20066/apps/ora-node/src/session/workload/tests.rs#L169)：遍历打不开的 checkout 使本次启动失败，而不是信任更早会话的交接。
- [walks_the_tree_without_following_links](https://github.com/ora-space/desktop/blob/c5265072ace476b6cdd88fc69fb58ffafec20066/crates/utils/src/fs/owned_tree/tests.rs#L79)、[reowns_every_entry_and_links_as_themselves](https://github.com/ora-space/desktop/blob/c5265072ace476b6cdd88fc69fb58ffafec20066/crates/utils/src/fs/owned_tree/tests.rs#L129)、[a_name_swapped_after_pinning_changes_nothing_outside](https://github.com/ora-space/desktop/blob/c5265072ace476b6cdd88fc69fb58ffafec20066/crates/utils/src/fs/owned_tree/tests.rs#L186)、[a_concurrent_symlink_swap_never_reowns_outside_the_root](https://github.com/ora-space/desktop/blob/c5265072ace476b6cdd88fc69fb58ffafec20066/crates/utils/src/fs/owned_tree/tests.rs#L232)：链接按本身处理、不跟随；真实改属主到达根、嵌套目录、文件与链接而不触及链接目标；钉住后名称被换成文件、删除或换成指向外部目录的链接时只作用于钉住的 inode；并发写方持续把目录换成外部链接也不会重属主 checkout 之外的任何东西。
- [an_entry_removed_before_it_is_pinned_is_skipped](https://github.com/ora-space/desktop/blob/c5265072ace476b6cdd88fc69fb58ffafec20066/crates/utils/src/fs/owned_tree/tests.rs#L157)、[entries_removed_concurrently_do_not_fail_the_walk](https://github.com/ora-space/desktop/blob/c5265072ace476b6cdd88fc69fb58ffafec20066/crates/utils/src/fs/owned_tree/tests.rs#L298)、[the_root_must_exist_as_a_directory](https://github.com/ora-space/desktop/blob/c5265072ace476b6cdd88fc69fb58ffafec20066/crates/utils/src/fs/owned_tree/tests.rs#L101)：钉住前消失的条目跳过、并发删除不使遍历失败；缺失的根、是链接的根、是文件的根都报错而不当作空树。
- [reports_entries_it_cannot_reown](https://github.com/ora-space/desktop/blob/c5265072ace476b6cdd88fc69fb58ffafec20066/crates/utils/src/fs/owned_tree/tests/errors.rs#L17)、[reports_a_directory_it_cannot_list](https://github.com/ora-space/desktop/blob/c5265072ace476b6cdd88fc69fb58ffafec20066/crates/utils/src/fs/owned_tree/tests/errors.rs#L34)、[reports_an_entry_it_cannot_pin](https://github.com/ora-space/desktop/blob/c5265072ace476b6cdd88fc69fb58ffafec20066/crates/utils/src/fs/owned_tree/tests/errors.rs#L55)；启动侧 [a_checkout_the_walk_cannot_list_ends_the_session_as_a_start_failure](https://github.com/ora-space/desktop/blob/c5265072ace476b6cdd88fc69fb58ffafec20066/apps/ora-node/tests/agent_session/workload.rs#L72) 以 `agent_failed{agent_start_failed}` 结束且不误报 `checkout_too_deep`，交付侧 [a_failed_hand_over_fails_snapshot_failed](https://github.com/ora-space/desktop/blob/c5265072ace476b6cdd88fc69fb58ffafec20066/apps/ora-node/src/revision/tests/snapshot.rs#L533) 以 `snapshot_failed` 失败且不留冻结对象。
- [a_hard_link_that_stays_in_place_is_not_reowned](https://github.com/ora-space/desktop/blob/c5265072ace476b6cdd88fc69fb58ffafec20066/crates/utils/src/fs/owned_tree/tests/links.rs#L17)、[a_hard_linked_file_keeps_its_owner](https://github.com/ora-space/desktop/blob/c5265072ace476b6cdd88fc69fb58ffafec20066/crates/utils/src/fs/owned_tree/tests/links.rs#L70)：留在原处的硬链接（遍历前植入、或钉住前后换入）不重属主其外部 inode，链接数为一的照常重属主；[a_hard_link_removed_after_pinning_is_reowned](https://github.com/ora-space/desktop/blob/c5265072ace476b6cdd88fc69fb58ffafec20066/crates/utils/src/fs/owned_tree/tests/links.rs#L99) 固定该跳过不构成保证：写方在钉住后删掉自己的名称，遍历读到链接数为一就会重属主外部 inode。
- [every_descriptor_the_walk_holds_is_close_on_exec](https://github.com/ora-space/desktop/blob/c5265072ace476b6cdd88fc69fb58ffafec20066/crates/utils/src/fs/owned_tree/tests.rs#L341)：从 hook 处检查钉住的条目与各级列举目录的描述符都是 close-on-exec。
- 边界：真实改属主依赖运行用户有可用的附属组（root 可用任意组）；没有时测试失败而不是静默跳过，除非设 `ORA_TEST_ALLOW_NO_FOREIGN_GROUP=1`。

### 决策依据

D1 的交接规则与留在原处的硬链接条款，不变量 4。

## The Hand-Over Walk Refuses Trees Deeper Than 4096 As Checkout Too Deep

### 风险

能在 checkout 里建目录的 Agent 决定特权进程持有多少描述符，使后续启动、交付及 Node 其他工作以 `EMFILE`
失败；过深的树被部分交接或部分快照；深度失败被混进 `agent_start_failed` 或 `snapshot_failed`，或其他交接
失败被误报为 `checkout_too_deep`；Controller 无法把新失败码映射到固定的 Cloud 契约。

### 前置状态

分离工作负载部署；checkout 内存在恰好 4096 层或更深的目录链。

### 触发

在过深的 checkout 上启动会话；对过深的 checkout 准备交付；Controller 处理 `checkout_too_deep` 交付结果。

### 必须成立

- 遍历在打开第 4097 层目录之前以独立错误拒绝，恰好 4096 层的树完整交接；单次遍历至多持有 4099 个描述符。
- 启动以 `agent_failed{checkout_too_deep}` 结束，且在任何插件进程或会话目录出现之前；Node 继续服务。
- 交付以 `failed{checkout_too_deep}` 失败，且不留下冻结对象；其他交接失败不使用该代码。
- `checkout_too_deep` 是 Node 协议的封闭交付失败码；Controller 把它映射为 `REVISION_FAILURE_REASON_SNAPSHOT_FAILED`。

### 禁止结果

- 遍历持有的描述符随树深无界增长；过深树被部分重属主或部分快照；深度失败与其他失败共用代码；
  Controller 遇到该代码时解码失败或映射为其他原因。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 深度上限与描述符上界 | `crates/utils/src/fs/owned_tree/tests/depth.rs` 的 `the_walk_refuses_trees_deeper_than_the_bound`：4096 层完整交接，多一层以 `TooDeep` 拒绝且不重属主，更深的链不使描述符超过上界 | Covered |
| 启动侧终态 | `apps/ora-node/tests/agent_session/workload.rs` 的 `a_checkout_deeper_than_the_walk_bound_ends_the_session` | Covered |
| 交付侧失败码 | `apps/ora-node/src/revision/tests/snapshot.rs` 的 `a_checkout_deeper_than_the_walk_bound_fails_checkout_too_deep`；同文件 `a_failed_hand_over_fails_snapshot_failed` 固定其他失败不用该代码 | Covered |
| wire 拼写与 Controller 映射 | `crates/node-protocol/tests/protocol/revision.rs` 的 `revision_failures_name_push_outcomes` 固定 `checkout_too_deep` 拼写；`apps/ora-controller/src/cloud/deliveries/mapping.rs` 的 `results_project_onto_the_contract` 映射为 `SnapshotFailed` | Covered |

### 代表性证据

- [the_walk_refuses_trees_deeper_than_the_bound](https://github.com/ora-space/desktop/blob/c5265072ace476b6cdd88fc69fb58ffafec20066/crates/utils/src/fs/owned_tree/tests/depth.rs#L107)：恰好 [`MAX_OWN_TREE_DEPTH`](https://github.com/ora-space/desktop/blob/c5265072ace476b6cdd88fc69fb58ffafec20066/crates/utils/src/fs/owned_tree.rs#L29)（4096）层完整交接，多一层以 `TooDeep` 拒绝且不重属主，更深的链不使描述符超过上界。
- [a_checkout_deeper_than_the_walk_bound_ends_the_session](https://github.com/ora-space/desktop/blob/c5265072ace476b6cdd88fc69fb58ffafec20066/apps/ora-node/tests/agent_session/workload.rs#L36)：启动以 `agent_failed{checkout_too_deep}` 结束，之前没有插件进程或会话目录，Node 继续服务。
- [a_checkout_deeper_than_the_walk_bound_fails_checkout_too_deep](https://github.com/ora-space/desktop/blob/c5265072ace476b6cdd88fc69fb58ffafec20066/apps/ora-node/src/revision/tests/snapshot.rs#L504)：交付以 `checkout_too_deep` 失败且不留冻结对象；[a_failed_hand_over_fails_snapshot_failed](https://github.com/ora-space/desktop/blob/c5265072ace476b6cdd88fc69fb58ffafec20066/apps/ora-node/src/revision/tests/snapshot.rs#L533)：无法列举的目录与失去硬链接保护的宿主都以 `snapshot_failed` 失败，不用该代码。
- [revision_failures_name_push_outcomes](https://github.com/ora-space/desktop/blob/c5265072ace476b6cdd88fc69fb58ffafec20066/crates/node-protocol/tests/protocol/revision.rs#L255)：L261 固定 `checkout_too_deep` 的 wire 拼写；[results_project_onto_the_contract](https://github.com/ora-space/desktop/blob/c5265072ace476b6cdd88fc69fb58ffafec20066/apps/ora-controller/src/cloud/deliveries/mapping.rs#L516)：L563 起把 `CheckoutTooDeep` 映射为 `SnapshotFailed`（生产映射在 L154）。

### 决策依据

D1 的深度上限，D3 末段的交付失败码与 Cloud 映射，不变量 4。

## A Separate Workload Requires Protected Hardlinks At Startup And Before Every Walk

### 风险

宿主未开启 `fs.protected_hardlinks`，工作负载用户把 root 拥有的 Agent 包或 Node 状态硬链接进 checkout，并以
“钉住后删名”的竞争让特权遍历重属主它；该 sysctl 不随容器命名空间隔离、启动后被改回 0 而 Node 继续遍历；
共享工作负载部署被不必要地要求该 sysctl。

### 前置状态

设置了 `process.workload_uid` 的 Node；`/proc/sys/fs/protected_hardlinks` 读出 `0`、`1` 或在运行期间被改动；
另有未设置 `workload_uid` 的部署。

### 触发

启动 Node；Node 运行期间把该 sysctl 改为 `0` 后启动会话或准备交付；再改回 `1` 后重试。

### 必须成立

- 设置了 `process.workload_uid` 时，Node 启动前要求该 sysctl 读出 `1`，否则在打开任何资源前以点名该
  sysctl 的错误拒绝启动；未设置时不读取。
- 每次启动遍历与交付遍历之前重新读取；读到 `1` 以外的值时在触碰 checkout 前拒绝该次交接：启动以
  `agent_failed{agent_start_failed}` 结束且无插件进程，交付以 `failed{snapshot_failed}` 失败且无冻结对象，
  日志点名该 sysctl；保护恢复后下一次遍历照常进行。

### 禁止结果

- 分离工作负载的 Node 在保护关闭时启动或继续遍历；只在启动时检查；拒绝使用新代码或误报为
  `checkout_too_deep`；共享工作负载部署因该 sysctl 拒绝启动。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 启动时检查 | `apps/ora-node/src/service/tests.rs` 的 `a_separate_workload_requires_protected_hardlinks`、`protected_hardlinks_is_required_only_for_a_separate_workload` | Covered |
| 每次遍历前重读 | `apps/ora-node/src/hand_over/tests.rs` 的 `every_walk_rereads_protected_hardlinks`：改为非 `1` 后下一次遍历在触碰 checkout 前被拒，恢复后再次运行 | Covered |
| 启动与交付的拒绝代码 | `apps/ora-node/tests/agent_session/workload.rs` 的 `a_host_without_protected_hardlinks_refuses_the_start`；`apps/ora-node/src/revision/tests/snapshot.rs` 的 `a_failed_hand_over_fails_snapshot_failed` | Covered |

### 代表性证据

- [a_separate_workload_requires_protected_hardlinks](https://github.com/ora-space/desktop/blob/c5265072ace476b6cdd88fc69fb58ffafec20066/apps/ora-node/src/service/tests.rs#L76)、[protected_hardlinks_is_required_only_for_a_separate_workload](https://github.com/ora-space/desktop/blob/c5265072ace476b6cdd88fc69fb58ffafec20066/apps/ora-node/src/service/tests.rs#L107)：分离工作负载且 sysctl 不为 `1` 时在打开任何资源前以点名该 sysctl 的错误拒绝启动；共享工作负载不读取。
- [every_walk_rereads_protected_hardlinks](https://github.com/ora-space/desktop/blob/c5265072ace476b6cdd88fc69fb58ffafec20066/apps/ora-node/src/hand_over/tests.rs#L66)：Node 启动后把设置改为非 `1`，下一次遍历在触碰 checkout 前被拒，恢复后再次运行。
- [a_host_without_protected_hardlinks_refuses_the_start](https://github.com/ora-space/desktop/blob/c5265072ace476b6cdd88fc69fb58ffafec20066/apps/ora-node/tests/agent_session/workload.rs#L97)：启动以 `agent_failed{agent_start_failed}` 结束且无插件进程；交付侧见 [a_failed_hand_over_fails_snapshot_failed](https://github.com/ora-space/desktop/blob/c5265072ace476b6cdd88fc69fb58ffafec20066/apps/ora-node/src/revision/tests/snapshot.rs#L533)。

### 决策依据

D1 的 `fs.protected_hardlinks` 条款，不变量 4；“风险、兼容性与落地顺序”中两次读取之间的窗口不由 Node 承诺。

## At Most Nine Hand-Over Walks Run At Once Across The Node

### 风险

工作负载用户把任意多个 checkout 弄深，按 checkout 限制的并发遍历使描述符总量随 checkout 数增长，文档中的
`RLIMIT_NOFILE` 不再是上限；等待许可的遍历被丢弃而不是最终运行；部署不知道需要多少描述符。

### 前置状态

分离工作负载部署；多个 checkout 上同时有启动遍历与交付遍历请求。

### 触发

同时发起多于 9 次的启动与交付遍历。

### 必须成立

- 启动遍历与交付遍历共用一个进程级信号量，共 9 个许可；任意时刻至多 9 次遍历重叠，等待的遍历在许可
  释放后全部运行。
- Node 文档写明分离 UID 部署需要的软 `RLIMIT_NOFILE`：至少 38,000（推荐 65,536），依据 Node 自身约 1,024
  个描述符加 9 × 4,099；Node 不自行调整该限制。

### 禁止结果

- 超过 9 次遍历同时运行；限制按 checkout 计算；等待的遍历永不运行或被静默跳过；文档缺失或给出低于
  推算值的最低要求。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| Node 级 9 次并发上限 | `apps/ora-node/src/hand_over/tests.rs` 的 `at_most_nine_walks_run_at_once`：任意多的启动与交付请求至多 9 次重叠且全部最终运行 | Covered |
| `RLIMIT_NOFILE` 最低要求 | 文档义务：`docs/node/agent-sessions.md` 与 `docs/node/agent-sessions.zh.md` 写明 38,000 最低值、65,536 推荐值与 9 × 4,099 + 1,024 的推算；`apps/ora-node/src/hand_over/tests.rs` 的 `the_documented_descriptor_minimum_follows_the_walk_bounds` 解析两份文档，把遍历数与每次描述符数钉到 `MAX_CONCURRENT_WALKS` 与 `MAX_OWN_TREE_DEPTH + 3`，要求每个数字覆盖前一个。测试证明的是文档里的最低值始终由遍历上限推导，不是部署真的提供了该限制——后者没有测试能证明，Node 也不读取或调整该限制 | Covered |

### 代表性证据

- [at_most_nine_walks_run_at_once](https://github.com/ora-space/desktop/blob/c5265072ace476b6cdd88fc69fb58ffafec20066/apps/ora-node/src/hand_over/tests.rs#L20)：任意多的启动与交付遍历请求至多 9 次重叠，许可释放后全部最终运行；上限由 [`MAX_CONCURRENT_WALKS = 9`](https://github.com/ora-space/desktop/blob/c5265072ace476b6cdd88fc69fb58ffafec20066/apps/ora-node/src/hand_over.rs#L25) 定义。
- `RLIMIT_NOFILE` 是文档义务：[`docs/node/agent-sessions.md` L98–99](https://github.com/ora-space/desktop/blob/c5265072ace476b6cdd88fc69fb58ffafec20066/docs/node/agent-sessions.md#L98) 与 [`docs/node/agent-sessions.zh.md` L71–72](https://github.com/ora-space/desktop/blob/c5265072ace476b6cdd88fc69fb58ffafec20066/docs/node/agent-sessions.zh.md#L71) 写明 `9 × 4,099 + 1,024 ≈ 38,000`、至少 38,000、推荐 65,536。[the_documented_descriptor_minimum_follows_the_walk_bounds](https://github.com/ora-space/desktop/blob/89b7ce375579d286f70b9bff17cf5475d9d64b22/apps/ora-node/src/hand_over/tests.rs#L197) 解析两份文档，把遍历数和每次描述符数钉到 `MAX_CONCURRENT_WALKS` 与 `MAX_OWN_TREE_DEPTH + 3`（4099），并要求每个数字覆盖前一个：常量改成 10 / 5000 而不改文档、文档最低值或 `≈` 总数改成 30,000 三种变异都被杀死。它证明的是文档里的最低值始终由遍历上限推导出来，不是部署真的提供了该限制——后者没有测试能证明，ADR 也如此声明；Node 不读取也不调整该限制（`apps/ora-node` 与 `ora-utils` 的生产代码没有 `setrlimit`）。

### 决策依据

D1 的 Node 级遍历上限与 `RLIMIT_NOFILE` 推算，不变量 4；“风险、兼容性与落地顺序”的描述符剩余边界。

## Shared Checkout Sessions Is Advertised And Bound To Same-Release Controllers

### 风险

Node 未声明能力或拼写漂移，Controller 无法按能力启用同 run 多会话；能力被解读为取消受控单例或替代
`revision_restore`；同版本 Controller 因新值或新失败码解码失败；混合版本 Rust Controller 握手的不兼容被
隐藏而不是作为部署边界。

### 前置状态

Agent 功能启用的 Node 回复 `HelloAccepted`；同版本 `ora-controller` 以 `contains` 查询能力；v2 Go 客户端把
能力解析为字符串列表。

### 触发

真实 Node 进程握手；同版本 Controller 解析握手和含 `checkout_too_deep` 的交付结果。

### 必须成立

- Agent 功能启用时，`HelloAccepted.capabilities` 同时含 `agent_session` 与 `shared_checkout_sessions`，线上值
  为 `shared_checkout_sessions`；能力只表示同一 checkout 可共享及 D2/D3 语义，不替代 `revision_restore`，
  不表示受控单例已取消。
- 同版本 Controller 无需新增业务分支即可容忍该能力与 `checkout_too_deep`。
- 旧版 Rust Controller 不能与新 Node 握手，这是既有“Controller 与 Node 必须同版本”的部署边界；本决策不
  承诺未知能力或未知失败码的混合版本解析。

### 禁止结果

- 能力缺失或拼写不同；能力在 Agent 功能关闭时声明；同版本 Controller 解码失败；文档或协议暗示支持混合
  版本 Rust Controller。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| wire 拼写 | `crates/node-protocol/tests/protocol/session.rs` 的 `accepts_independent_and_combined_execution_capabilities` 固定 `shared_checkout_sessions` | Covered |
| 生产握手声明 | `apps/ora-node/tests/standalone/agent_sessions/support.rs` 的 `connect`：真实 Node 进程声明 `SharedCheckoutSessions` | Covered |
| 同版本 Controller 解析交付失败码 | `apps/ora-controller/src/cloud/deliveries/mapping.rs` 的 `results_project_onto_the_contract` | Covered |
| 混合版本边界 | 部署边界，不登记测试：`NodeCapability` 是封闭的 Serde 枚举（`crates/node-protocol/src/message/session.rs`），没有未知能力的回退分支，旧版 Rust Controller 对 `shared_checkout_sessions` 解码失败即“Controller 与 Node 必须同版本”的既有边界；Controller 在握手中按能力启用同 run 多会话由 v2 事项另行交付并登记，不计入本文 | Covered |

### 代表性证据

- [accepts_independent_and_combined_execution_capabilities](https://github.com/ora-space/desktop/blob/c5265072ace476b6cdd88fc69fb58ffafec20066/crates/node-protocol/tests/protocol/session.rs#L10)：固定 `shared_checkout_sessions` 与 `agent_session` 可独立、可同时声明。
- [connect](https://github.com/ora-space/desktop/blob/c5265072ace476b6cdd88fc69fb58ffafec20066/apps/ora-node/tests/standalone/agent_sessions/support.rs#L155)：所有 standalone 用例共用的生产握手（`connect`，L136）断言真实 Node 进程声明 `SharedCheckoutSessions`（与 `AgentSession`、`RevisionRestore` 等同批）。
- [results_project_onto_the_contract](https://github.com/ora-space/desktop/blob/c5265072ace476b6cdd88fc69fb58ffafec20066/apps/ora-controller/src/cloud/deliveries/mapping.rs#L516)：同版本 Controller 把 `checkout_too_deep` 映射为 `SnapshotFailed`，不解码失败。
- 混合版本边界不登记测试：[`NodeCapability`](https://github.com/ora-space/desktop/blob/c5265072ace476b6cdd88fc69fb58ffafec20066/crates/node-protocol/src/message/session.rs#L11) 是封闭的 Serde 枚举，没有未知能力的回退分支，旧版 Rust Controller 解码 `shared_checkout_sessions` 失败即“Controller 与 Node 必须同版本”的既有部署边界（D4）；Controller 在握手中按能力启用同 run 多会话由 v2 事项另行交付并登记。

### 决策依据

D4；继承[流式 Thread 事件与会话命令](../../../decisions/node/protocol/20260928-streamed-thread-events-and-session-commands.md)
D1 的能力协商。
