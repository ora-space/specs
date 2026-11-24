# Node 会话 MCP 刷新与宿主健康

依据[已批准的会话 MCP 刷新与宿主健康决策](../../../decisions/node/agent-runtime/20261116-session-mcp-refresh-and-host-health.md)。
Node 实现依据固定合并提交 [`28da0a4e`](https://github.com/ora-space/desktop/commit/28da0a4e77ec2257468b15bffd99e166a186b31c)。
Node 侧验收结论按第三轮复审：第 4、5、6、8、12 条为 `Passed`，其余为 `Partial`。
`Passed` 是该轮 Node 侧验收结论，不表示跨仓库义务全部完成，也不替代表内逐项的
`Covered / Partial / Missing`。表内保留原验证目标，下面列固定源码的代表性直接证据及不能由它推导的缺口。
测试运行与变异结果保存在 [circus#1218](https://github.com/ora-space/circus/issues/1218)，
本次证据核对与 Low 处置保存在 [circus#1231](https://github.com/ora-space/circus/issues/1231)。
Controller/v2 的协议接线仍由 [circus#1171](https://github.com/ora-space/circus/issues/1171) 承接；旧门禁测试不算接线通过。
[显式 Session MCP 用例](session-mcp.md)、[共享 checkout 会话用例](shared-checkout-sessions.md)、
[会话执行用例](session-execution.md)与[尝试锁用例](../git-execution/attempt-locks-and-stop-evidence.md)的
证据保持各自范围，不推导本文新义务。

[会话尝试环境标记决策](../../../decisions/node/agent-runtime/20261120-attempt-mark-in-session-stop-evidence.md)（`implemented`）修改了本文的停止证据：在尝试锁、重启收口、未排空升级、运维对账、
版本重启与探测清理六个用例中补充义务，并新增[会话停止杀掉全部带标记进程](#a-session-stop-kills-every-attempt-marked-process-and-finished-turns-kill-none)。
这 19 项义务按 [desktop#802](https://github.com/ora-space/desktop/pull/802) 的 head
[`198cc415`](https://github.com/ora-space/desktop/commit/198cc41517a837ec436a897655403adc790b8c0d)（尚未合入 main）登记直接证据：
7 项 `Covered`、10 项 `Partial`、2 项 `Missing`。既有义务的证据按原固定提交 `28da0a4e` 保留，不因新决策改变状态。
其中“轮次之间与保留会话不杀常驻进程”一项随[轮次进程栅栏决策](../../../decisions/node/protocol/20261123-turn-process-fence-before-keeping-session.md)
的实现，按 [desktop#807](https://github.com/ora-space/desktop/pull/807) 的合并提交
[`447a59de`](https://github.com/ora-space/desktop/commit/447a59de6722500cae0be011860e63e5127a3b23) 改登记为 `Covered`，
这 19 项现为 8 项 `Covered`、10 项 `Partial`、1 项 `Missing`。
评审方在 `198cc415` 上用发布包 OpenCode 0.6.11 与 Claude 0.4.5 复测 EndSession 和导致会话结束的 CancelTurn、OpenCode 的
SIGKILL 后重启（Claude 的重启复测在中间版本 `07c495e0`），作业都在报告停止前被杀；这是运行记录，不是可重复测试，不登记为证据。

## Lifecycle Mode Is Opt-In Per Session And Never Widens The Selection

### 风险

旧会话被隐式升级为可刷新；Controller 向缺少新能力的 Node 发送生命周期输入或降级为冻结 setup 后照常
交付非空 MCP；刷新、挂接、provider 恢复或元数据丢失扩大显式白名单 S；更新省略成员后发送部分列表；
运行中丢失 S 时用空 update 或 Node 观察重建授权。

### 前置状态

Node 声明或不声明 `session_mcp_lifecycle`；业务 Session 持久化了显式选择 S（可为空）；部分会话以
`mcp_lifecycle: true` 启动，部分省略。

### 触发

v2 向只有 `session_mcp` 的 Node 派发带非空 MCP 的工作流会话；生命周期会话收到含全部 S 成员的
configured / unavailable 更新、缺成员或多成员的更新；Controller 丢失运行中 execution 的 S；重发 start。

### 必须成立

- 只有同时支持 `agent_session`、`session_mcp` 与本文全部行为的 Node 才声明 `session_mcp_lifecycle`；
  `mcp_lifecycle` 无值、持久并参与重复 start 比较，省略或 false 保持一次冻结 setup。
- Cloud v2 对非空选择要求新能力，缺少时拒绝启动带 MCP 的工作流会话；空选择仍按原能力启动。
- S 在 execution 全程不变；更新恰好为 S 的每个成员提供一项；unavailable、缺包或解析失败使完整 desired
  blocked，不删除成员、不发送部分列表。
- 运行中丢失 S 时立即停止派发并持久投递 EndSession。

### 禁止结果

- 省略模式的会话接受更新；缺能力的 Node 收到新字段或非空 MCP 被降级交付；任何路径使 Agent 看到 S 之外的
  Server 或少于 S 的部分列表；用 `GetSessionMcpState` 或空 update 重建、改写授权。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 能力声明与逐会话选择 | 真实 Node 握手声明 `session_mcp_lifecycle`，省略 `mcp_lifecycle` 的会话以 `not_enabled` 拒绝更新且保持冻结 setup 的协议级测试 | Partial |
| v2 能力门禁 | v2 面对只有 `session_mcp` 的 Node 时拒绝非空选择、放行空选择的 Controller 测试 | Missing |
| 白名单与完整列表 | 更新缺成员或多成员以 `invalid_selection` 拒绝、unavailable 成员使会话 blocked 且 ACP 帧从不出现部分列表的测试 | Partial |
| 运行中丢失 S | Controller 丢失 S 后不再派发轮次并投递 EndSession、不发空 update 的测试 | Missing |

### 代表性证据与剩余缺口

Node 侧验收：`Partial`。

- [lifecycle_agent_failure_is_terminal_and_unknown_execution_keeps_connection_open](https://github.com/ora-space/desktop/blob/28da0a4e77ec2257468b15bffd99e166a186b31c/apps/ora-node/tests/standalone/agent_sessions/mcp_lifecycle.rs#L11)。
- [stale_load_never_admits_and_unavailable_selection_is_accepted_but_blocked](https://github.com/ora-space/desktop/blob/28da0a4e77ec2257468b15bffd99e166a186b31c/apps/ora-node/src/session/mcp/control/tests.rs#L131)。

Node 已具备 opt-in 与完整选择校验，但还缺省略模式、所有白名单失败组合的独立端到端矩阵。v2 的运行中丢失 S 与新生命周期派发由消费端承接；现有拒绝门禁不能算正向接入通过。

### 决策依据

D1、D5 首段，不变量 1。

## Session MCP Updates Are Ordered By Generation Before Prompt Admission

### 风险

旧 generation 回滚目标；同一 generation 不同输入覆盖原目标或泄露差异；同一配置身份换值却复用 revision；
受理回复早于 admission 目标替换，使随后的 prompt 用旧配置；过时的异步解析或 ACP 成功解除新目标的阻塞；
待答请求无界积压。

### 前置状态

生命周期会话已受理最高 generation G；Controller 串行分配 generation，可能重发、乱序或丢失回复。

### 触发

发送 g > G、g = G 同输入、g = G 不同输入、g < G、同一成员配置身份但不同 Settings 的更新；向未知或已结束
执行发送更新与探测；单连接同时超过 64 个待答请求。

### 必须成立

- g > G 先原子替换 desired 与 prompt admission 目标再回复 accepted；包或设置不可用也推进并 blocked。
- g = G 同输入为 duplicate，不重做 ACP 或探测；不同输入为 conflict，不改目标；g < G 为 stale。
- 同一仍持有的配置身份给出不同 Settings 时以 `configuration_revision_conflict` 拒绝，不推进 generation。
- generation 纳入无值 revision，同内容的新 generation 也要完成新收敛；异步结果写回前复查身份。
- 未知执行返回 not_found 且不泄露其他执行；满 64 个待答请求时回复 `capacity_exceeded`，无副作用。

### 禁止结果

- 旧 generation 改变目标；conflict 输出值差异或摘要；受理回复后仍有 prompt 以旧目标准入；旧 ACP 成功
  被标为新目标完成；超额请求排队或产生更新、探测。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| generation 比较表 | 逐行覆盖 g>G / g=G 同与不同 / g<G / 配置身份换值的 Node 单元测试，断言目标与回复 | Covered |
| 受理先于回复替换目标 | Update 与 prompt 并发、回复 accepted 后准入的 prompt 必然看到新目标的测试 | Partial |
| 过时结果不解除阻塞 | 旧 generation 的解析或 ACP 成功晚于新 Update 到达时会话仍欠账的测试 | Covered |
| not_found 与容量拒绝 | 未知执行 not_found、第 65 个待答请求 `capacity_exceeded` 且无副作用的协议测试 | Partial |

### 代表性证据与剩余缺口

Node 侧验收：`Partial`。

- [generation_comparison_precedes_resolution_and_admission_tracks_latest_target](https://github.com/ora-space/desktop/blob/28da0a4e77ec2257468b15bffd99e166a186b31c/apps/ora-node/src/session/mcp/control/tests.rs#L81)。
- [desired_active_and_sending_each_reject_conflicting_held_values](https://github.com/ora-space/desktop/blob/28da0a4e77ec2257468b15bffd99e166a186b31c/apps/ora-node/src/session/mcp/control/tests.rs#L4)。
- [old_failure_and_late_success_cannot_release_a_new_target](https://github.com/ora-space/desktop/blob/28da0a4e77ec2257468b15bffd99e166a186b31c/crates/agent-runtime/src/session_setup/generation_tests.rs#L97)。
- [ordered_updates_floor_rejection_and_process_budget_use_the_real_service](https://github.com/ora-space/desktop/blob/28da0a4e77ec2257468b15bffd99e166a186b31c/apps/ora-node/tests/standalone/agent_sessions/mcp_lifecycle.rs#L348)。

比较表和仍持有身份冲突有直接断言；缺第 65 个待答请求无副作用的真实连接测试与全部并发线性化路径。not_found 连接存活由生命周期服务测试覆盖。

### 决策依据

D2，不变量 2。

## Setting Values Stay In Memory Across Updates Probes And Stop Records

### 风险

Settings 或其摘要进入命令队列、WAL、Thread、事件、日志、错误、停止责任记录、审计、临时文件或页面；
内存中保留无界数量的有值副本。

### 前置状态

生命周期会话持有 active、正在发送与最新 desired 输入；有探测 flight 和 cleanup_pending；停止责任与
审计表已启用。

### 触发

以带特征值的 Settings 连续更新、触发失败刷新与探测失败、停止 Agent 产生停止记录、运维对账。

### 必须成立

- 每个会话至多保留 active、正在发送和最新 desired 三份有值输入，另加至多 8 个 Node 探测 flight
  （每执行 2 个）的成员绑定；未发送的中间值被丢弃。
- 特征值只出现在允许的内存与 ACP / 探测外发中；错误只含稳定码及 plugin / setting / transport 标识；
  停止记录与审计不含 argv、env、URL、headers、Settings 或其摘要。

### 禁止结果

- 特征值或其摘要出现在 ledger / WAL、JSONL、Thread、事件、日志、错误、停止责任、审计或页面中；排队
  意图或通用后台队列复制有值输入。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 持久与诊断表面无值 | 带特征 Settings 跑完更新、失败、探测、停止与对账后扫描 SQLite（含 WAL）、JSONL、日志与对外消息的端到端测试 | Partial |
| 有值副本上界 | 连续多次更新后会话只保留三份有值输入、探测绑定受 8/2 约束的测试 | Partial |

### 代表性证据与剩余缺口

Node 侧验收：`Partial`。

- [crates/node-protocol/tests/protocol/session_mcp_lifecycle.rs](https://github.com/ora-space/desktop/blob/28da0a4e77ec2257468b15bffd99e166a186b31c/crates/node-protocol/tests/protocol/session_mcp_lifecycle.rs#L1)。
- [mcp_reaches_real_acp_session_new_without_persisting_or_logging_settings](https://github.com/ora-space/desktop/blob/28da0a4e77ec2257468b15bffd99e166a186b31c/apps/ora-node/tests/agent_session/mcp.rs#L120)。
- [desired_active_and_sending_each_reject_conflicting_held_values](https://github.com/ora-space/desktop/blob/28da0a4e77ec2257468b15bffd99e166a186b31c/apps/ora-node/src/session/mcp/control/tests.rs#L4)。
- [global_eight_session_two_and_round_robin_survive_slow_cleanup](https://github.com/ora-space/desktop/blob/28da0a4e77ec2257468b15bffd99e166a186b31c/apps/ora-node/src/session/mcp/probes/tests.rs#L73)。

DTO、初始 setup 扫描与容量测试只覆盖部分表面；缺连续更新、失败、停止、审计及页面全链特征值扫描，不能由类型结构或初始 setup 测试推导全部无泄露。

### 决策依据

D2、D5“停止证据与记录”，不变量 3；继承显式 Session MCP D3。

## A Turn Is Settled Executed Only After Actor Admission

### 风险

轮次在 MCP 准入前被结算 Executed；刷新进行中或 blocked 时 prompt 被拒并使整个 execution 以
`agent_unavailable` 结束；配置保存后受理的轮次在 Node 尚未收到更新时用旧配置；`mcp_update_required`
被登记为已送达；重复命令改变下限。

### 前置状态

生命周期会话处于 current、pending、refreshing 或 blocked；Controller 把每轮的 `mcp_generation` 下限
与派发记录一同持久化。

### 触发

下限高于 Node desired 的新轮次；空闲刷新进行中出队的已受理轮次；blocked 会话的新轮次；同一 command
以原下限或不同下限重发；Update 与 Actor 最终准入竞争。

### 必须成立

- 新轮次下限高于 desired 时以 `mcp_update_required` 拒绝，不写队列；Controller 不登记已送达，先补 Update
  再重试原 command；已受理的重复命令按原内容返回 accepted。
- `mcp_generation` 是命令内容的一部分；生命周期模式必填，旧模式缺省。
- 对话循环在 `settle(Executed)` 前等待 pending / refreshing / blocked 收敛（初始轮次也一样）；Actor 原子
  检查最新目标后才占有本轮，受理成功后才结算 Executed。
- MCP 原因的待刷新、busy 或刷新失败以非终结 `McpAdmissionPending` / `McpAdmissionBlocked` 返回，与
  agent unavailable 分开；Update 先于准入则所有未准入轮次都等最新目标。

### 禁止结果

- 未准入轮次被标 Executed；`session_busy` 或 MCP 门禁使 execution 结束；拒绝被登记为已送达；同一
  command 改下限被受理；Update 先于准入后仍有轮次以旧目标进入 Agent。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 下限拒绝与重试 | 下限领先时 `mcp_update_required` 不写队列、补 Update 后同 command 被受理的 Node + Controller 测试 | Partial |
| 准入后才结算 | 空闲刷新进行中出队的轮次保持未结算、刷新成功后才准入并结算的测试 | Covered |
| 非终结门禁 | blocked 或 busy 时 execution 保持运行、错误类型不是 agent unavailable 的测试 | Covered |
| 准入与 Update 的原子顺序 | 并发注入 Update 与最终准入，断言每轮所用目标与顺序一致的测试 | Covered |

### 代表性证据与剩余缺口

Node 侧验收：`Passed`。

- [ordered_updates_floor_rejection_and_process_budget_use_the_real_service](https://github.com/ora-space/desktop/blob/28da0a4e77ec2257468b15bffd99e166a186b31c/apps/ora-node/tests/standalone/agent_sessions/mcp_lifecycle.rs#L348)。
- [turn_arriving_during_idle_load_waits_for_that_load_without_destroying_sending](https://github.com/ora-space/desktop/blob/28da0a4e77ec2257468b15bffd99e166a186b31c/apps/ora-node/tests/standalone/agent_sessions/mcp_lifecycle/races.rs#L59)。
- [lifecycle_agent_failure_is_terminal_and_unknown_execution_keeps_connection_open](https://github.com/ora-space/desktop/blob/28da0a4e77ec2257468b15bffd99e166a186b31c/apps/ora-node/tests/standalone/agent_sessions/mcp_lifecycle.rs#L11)。
- [timeout_and_actor_commit_have_one_winner_at_the_deadline](https://github.com/ora-space/desktop/blob/28da0a4e77ec2257468b15bffd99e166a186b31c/apps/ora-node/src/session/driver/conversation/admission/tests.rs#L5)。

Node 侧准入、下限和结算验收通过；第一项包含 Controller 处理拒绝并重发原 command，这部分尚待消费端接线，保持 Partial。

### 决策依据

D4（准入表与宿主策略第 4 条），不变量 2、4。

## A Turn Blocked On MCP Fails After 300 Seconds With A Reliable Event

### 风险

受阻轮次无限等待；计时因 generation 更新、重连、重试或队首变化被重置；失败只在内存或被伪造成 Agent
Thread 记录、Executed 或普通 Discarded；终态后仍发轮次失败；blocked 但无轮次的会话被 Node 自行结束。

### 前置状态

已受理轮次因 MCP 门禁受阻（等待 barrier、刷新、Agent 重启或 blocked）；可能同时有 EndSession。

### 触发

受阻持续超过 300 秒；期间多次 Update、重连与重试；EndSession 与到期在同一结算边界竞争；Node 重启；
blocked 会话长时间没有新轮次。

### 必须成立

- 每个受阻轮次从首次因 MCP 门禁受阻起计时 300 秒，包括 barrier 等待、停止、spawn、initialize 和 load；
  不因任何更新、通知、重连或重试重置。owner 已接受慢重启可使排队轮次（含初始轮次）到期失败。
- 到期只该轮以 `mcp_refresh_wait_timeout` 失败：可靠的 `SessionTurnFailed { turn_id, command_id?, code }`
  与命令 `Failed` 结算同事务提交，共享执行序号与 Ack / 重放；execution 继续可用。
- EndSession 先生效则按原规则丢弃；终态后不再发轮次失败；Node 重启时未结算轮次按 interrupted 路径
  收尾，不重启新的 300 秒等待。
- ACP 刷新沿用 `session_setup_window`（有 MCP 为 120 秒无活动窗口，收到 session update 重置）；该窗口不延长
  轮次截止。blocked 而无排队轮次时不增加 Node 自有的结束时钟。

### 禁止结果

- 受阻轮次超过 300 秒仍未结算；失败事件缺失、不可重放或与结算分属不同事务；失败标为 Executed 或
  Discarded；终态后出现事件；Node 因 blocked 自行结束会话。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 300 秒截止不重置 | 受控时钟下多次 Update / 重连后第 300 秒恰好失败的测试 | Covered |
| 可靠失败事件 | `SessionTurnFailed` 与 `Failed` 同事务、断线重放后 Controller 区分已送达与轮次失败的测试 | Partial |
| 与 EndSession 竞争 | 同一结算边界只产生丢弃或失败之一、终态后无事件的测试 | Covered |
| 120 秒窗口与无 Node 结束时钟 | load 持续收到 update 时窗口重置但轮次仍按期失败、blocked 无轮次的会话不被 Node 结束的测试 | Partial |

### 代表性证据与剩余缺口

Node 侧验收：`Passed`。

- [blocked_initial_and_queued_turns_fail_once_without_ending_the_session](https://github.com/ora-space/desktop/blob/28da0a4e77ec2257468b15bffd99e166a186b31c/apps/ora-node/tests/standalone/agent_sessions/mcp_lifecycle.rs#L131)。
- [queued_turn_survives_business_time_longer_than_the_admission_budget](https://github.com/ora-space/desktop/blob/28da0a4e77ec2257468b15bffd99e166a186b31c/apps/ora-node/tests/standalone/agent_sessions/mcp_lifecycle/races.rs#L229)。
- [turn_failure_requires_running_and_end_session_wins_before_failure_write](https://github.com/ora-space/desktop/blob/28da0a4e77ec2257468b15bffd99e166a186b31c/crates/node-db/src/tests/session/mcp.rs#L6)。
- [timeout_and_actor_commit_have_one_winner_at_the_deadline](https://github.com/ora-space/desktop/blob/28da0a4e77ec2257468b15bffd99e166a186b31c/apps/ora-node/src/session/driver/conversation/admission/tests.rs#L5)。

两个真实 300 秒场景已进入默认测试集并经复审运行通过；Node 侧 Passed。Controller 区分送达/失败与完整断线重放消费仍待接入；120 秒 load 窗口与 300 秒的完整组合证据不足。

### 决策依据

D4（300 秒截止、ACP 窗口、无 Node 结束时钟），不变量 4。

## A Failed Refresh Keeps The Provider Identity And Retries Only Session Load

### 风险

刷新失败或进程替换后共享运行时把会话清成 Inactive，下一次 attach 失败时以 `session/new` 重建并迁移
transcript；resolver 失败停留在 RefreshPending；自动重试形成忙循环或重试风暴；Desktop 默认策略被改变。

### 前置状态

生命周期会话持有原 provider ID；刷新可能在解析、能力检查、ACP 请求、超时或进程替换处失败。

### 触发

resolver / 能力失败、ACP 拒绝或超时、load 期间 `AgentProcessReplaced`、`ConnectionLost` 后重连；之后
到来的若干轮次与重复 Update。

### 必须成立

- 失败与替换后保留 Blocked、最新 desired、上次 active 身份和原 provider ID，`channel = None`；不调用会
  清空 live MCP 的通用 `mark_stopped`。
- attach 与 reconnect 已有 provider ID 时只 `session/load`；不支持 load、原 ID 不存在或 load 失败均 blocked。
- 下一轮对当前 blocked generation 触发至多一次自动收敛，合并同一 in-flight 尝试；没有新轮次或新目标时
  不周期重试；重复命令和重复 Update 不触发重试。
- Desktop 默认 attach / rebuild 策略不变。

### 禁止结果

- 任何刷新、重连或替换路径调用 `create_provider_session` 或迁移 transcript；provider ID 被清除；同一轮
  触发多次重试；Desktop 行为改变。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 保留 Blocked 与 provider ID | 对 resolver 失败、ACP 失败、load 中替换分别断言状态与 provider ID 的共享运行时测试（Node 宿主策略） | Covered |
| 只 load、绝不 new | 原 ID load 失败后下一轮仍只发 `session/load`、ACP fixture 从未收到 `session/new` 的测试 | Covered |
| 单次重试 | 多轮与重复 Update 下每轮至多一次收敛尝试的测试 | Partial |
| Desktop 不变 | Desktop 默认策略下 attach 失败仍按原规则重建的回归测试 | Covered |

### 代表性证据与剩余缺口

Node 侧验收：`Passed`。

- [failed_replacement_resumes_on_new_target_without_new_provider_session](https://github.com/ora-space/desktop/blob/28da0a4e77ec2257468b15bffd99e166a186b31c/apps/ora-node/tests/standalone/agent_sessions/mcp_lifecycle/races.rs#L162)。
- [replacement_initialize_failure_is_blocked_and_cannot_auto_restart](https://github.com/ora-space/desktop/blob/28da0a4e77ec2257468b15bffd99e166a186b31c/apps/ora-node/tests/standalone/agent_sessions/mcp_lifecycle/races.rs#L352)。
- [one_start_or_refresh_attempt_replaces_at_most_once](https://github.com/ora-space/desktop/blob/28da0a4e77ec2257468b15bffd99e166a186b31c/crates/agent-runtime/src/plugin_replacement_tests.rs#L462)。
- [desktop_default_policy_refreshes_without_replacing_provider](https://github.com/ora-space/desktop/blob/28da0a4e77ec2257468b15bffd99e166a186b31c/crates/agent-runtime/src/plugin_replacement_tests.rs#L535)。

Node 保留 provider、只 load 与默认 Desktop 策略验收通过；单次替换有直接计数断言，但新增运行时重试仍有四个非等价变异存活，不能称完整重试故障矩阵 Covered。

### 决策依据

D4 宿主策略第 1–3 条，不变量 5；继承 Desktop ACP setup/refresh 的完整快照与 barrier。

## A Version Change Or The Ninth Runtime Load Restarts Only The Agent Of This Execution

### 风险

包版本变化继续运行中 load 而旧版本 MCP 进程仍在；同一进程上的 load 无界累积 provider 注册；重启以
`session/new` 替代原身份；一次重试循环拉起多个 Agent；旧 Agent 停止未确认就启动替代者或释放旧版本；
重启过程中晚到的目标被部分切换；模型授权或 ora-git socket 随重启关闭；重启波及兄弟会话。

### 前置状态

生命周期会话的 Agent 进程已写出若干运行时 load；目标包版本变化，或仅配置变化；同一 checkout 上有兄弟会话。

### 触发

版本更新；同一进程上的配置刷新、`ConnectionLost` 后重连、O1 重试与首次 load 失败后的重试累计到第 9 次；
新 Agent spawn 失败、initialize / ready 超时、load 前崩溃、load 失败；旧 Agent 停止中到达 V3；V2 已派发后
到达 V3。

### 必须成立

- 版本变化必须重启本 execution 独占的 Agent，并以原 provider ID load；仅配置变化继续运行中 load。
- 每个 Agent 进程写出的第一个 new / load 不计；此后写给同一进程的每个 `session/load`（配置刷新、重连
  重发、O1 重试、首次 load 失败后的重试；失败响应也计）加一，计满 8 次后下一次必须先走重启路径。
  新进程从 0 开始；普通 prompt 不改变计数。
- 重启先取得新集合租约并准备 view，失败则 blocked 且保留旧进程；等安全点后只停自己的 Agent；旧 Agent
  取得停止确认后才释放其独占旧版本、才启动新 Agent。
- 每次独立重试至多一次 spawn；spawn 失败或 load 前崩溃时先结算这次尝试，下一轮再 spawn + initialize +
  load；新进程可用而 load 失败时下一轮复用该进程 load。
- 目标合并边界是新 Agent spawn 前最后一次读取 desired：未派发的 V2 被丢弃并释放租约 / view，已派发的
  V2 先按 D5 停止确认再按最新目标准备。
- 模型授权与 ora-git socket 跨重启保留，spawn 前复核授权；兄弟会话不受影响。
- 替换结束旧尝试：旧尝试被标记扫描认领的全部进程（含后台作业与守护进程）先被冻结并杀掉，旧尝试取得
  `lock_processes_and_marks` 后才启动后继。

### 禁止结果

- 版本变化以运行中 load 完成；同一进程出现第 9 次运行时 load；任何重启路径发出 `session/new`；一次重试
  spawn 多个 Agent 或叠加未确认停止的旧 Agent；部分切换；重启关闭 ora-git / 模型访问；兄弟会话被停止。
- 旧尝试带标记的后台作业或守护进程活过替换，后继却已启动。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 版本变化走重启 | 版本更新后旧 Agent 进程退出、新进程以原 provider ID load 的 Node 集成测试 | Covered |
| 每进程运行时 load 计数 | 混合配置刷新、重连、O1 重试与失败 load，第 9 次改走重启、新进程从 0 计数的测试 | Partial |
| 重启失败矩阵 | 按 D6 表逐行注入 spawn 失败、ready 超时、load 前崩溃、load 失败，断言每次至多一次 spawn 与 blocked 状态的测试 | Partial |
| V2 / V3 合并 | 未派发 V2 被丢弃并释放租约、已派发 V2 先确认停止的测试 | Missing |
| execution 资源跨重启 | 重启前后 ora-git socket 地址与模型授权不变、撤销时按原策略结束的测试 | Partial |
| 替换先杀旧尝试的带标记进程 | 旧 Agent 的脱组后台作业在替换中被杀、旧尝试记 `lock_processes_and_marks` 后才启动后继的测试 | Partial |
| 真实 Agent 跨进程 load 前提 | 固定 OpenCode 1.19.3 与 claude-agent-acp 0.99.1 在保留 HOME/cwd 时新进程 load 原 ID 并保留历史，记录见 [circus#1204](https://github.com/ora-space/circus/issues/1204)；空 MCP、未经 Node 重启接缝，不能替代上面的 Node 集成测试 | Partial |

### 代表性证据与剩余缺口

Node 侧验收：`Partial`。

- [version_update_replaces_agent_and_host_probes_reap_real_descendants](https://github.com/ora-space/desktop/blob/28da0a4e77ec2257468b15bffd99e166a186b31c/apps/ora-node/tests/standalone/agent_sessions/mcp_lifecycle.rs#L220)。
- [ordered_updates_floor_rejection_and_process_budget_use_the_real_service](https://github.com/ora-space/desktop/blob/28da0a4e77ec2257468b15bffd99e166a186b31c/apps/ora-node/tests/standalone/agent_sessions/mcp_lifecycle.rs#L348)。
- [replacement_initialize_failure_is_blocked_and_cannot_auto_restart](https://github.com/ora-space/desktop/blob/28da0a4e77ec2257468b15bffd99e166a186b31c/apps/ora-node/tests/standalone/agent_sessions/mcp_lifecycle/races.rs#L352)。
- [one_start_or_refresh_attempt_replaces_at_most_once](https://github.com/ora-space/desktop/blob/28da0a4e77ec2257468b15bffd99e166a186b31c/crates/agent-runtime/src/plugin_replacement_tests.rs#L462)。

九次正常配置 load 与版本替换有真实 Node 证据；混合失败/重连计数、V2/V3 合并、所有 spawn/ready 失败、授权/socket 不变仍不完整。固定真实 Agent 的跨进程 load 前提实验仅见原记录，不等于真实 Agent 经 Node 替换的验证。旧 supervisor 等待无独立超时，Actor 调用路径也缺专门故障测试。

尝试环境标记（desktop#802 `198cc415`）：

- [replacement_waits_for_retirement_before_registering_the_successor](https://github.com/ora-space/desktop/blob/198cc41517a837ec436a897655403adc790b8c0d/apps/ora-node/src/session/stop/tests/replacement.rs#L19)：替换中旧尝试先记 `lock_processes_and_marks`，后继此时才登记为 `prepared`。

缺口：替换用例里旧 Agent 没有脱组后台作业，“替换中杀掉旧尝试的带标记作业”只由会话结束路径的同一终止逻辑间接支撑；真实插件的版本替换没有复测。

### 决策依据

D6（计数、重启表、目标合并、execution 资源），不变量 5；R4、R7、R11；[尝试环境标记决策](../../../decisions/node/agent-runtime/20261120-attempt-mark-in-session-stop-evidence.md) D4。

## Exact Version Leases Protect MCP Bytes In Use

### 风险

安装新版本删除仍被 Agent、重启中的集合或探测使用的旧版本；同 ID 换 kind 绕过 Agent 的按 ID 保护；
整 ID 移除或同版本修复改写在用字节；Separate view 仅按 ID 命名而被新版本覆盖；跨设备 view 悄悄退化为
完整复制。

### 前置状态

某 MCP 的版本 V1 被会话或探测租用；Separate 或 Shared placement；workload 目录、Node home 与 installed
包目录可能不在同一文件系统。

### 触发

安装 V2；对在用版本做同版本修复；整 ID `RemovePlugins`；以同 ID 发布非 MCP 包；跨设备部署启动或创建 view。

### 必须成立

- MCP 使用 canonical ID + 精确版本租约；新版本发布到独立目录，在用版本不得改写或删除；新旧均为 MCP
  才可并存，其他类别仍按 ID 保护。
- 整 ID 移除或在用同版本修复返回 `plugin_in_use`。
- Separate view 按 `(canonical ID, exact version)` 区分并由 execution 私有目录拥有；Shared 使用精确 installed
  目录。删除逐使用者结算。
- 启动检查设备与硬链接能力，每次创建 view 拒绝 `EXDEV`，跨设备部署报配置错误，不复制、不删除旧目录。

### 禁止结果

- 在用版本字节被删除或改写；kind 变化绕过保护；不同版本的 view 互相覆盖；跨设备时出现完整复制的 view。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 版本租约与并存 | V1 在用时安装 V2 成功且 V1 字节保留、V1 释放后可回收的 catalog 测试 | Covered |
| kind 与整 ID 保护 | 同 ID 改 kind 被拒、整 ID 移除与在用同版本修复返回 `plugin_in_use` 的测试 | Covered |
| view 命名与逐使用者删除 | 同一 execution 同时持有两个版本 view、Agent 停止不删除探测仍用的 view 的测试 | Partial |
| 同文件系统门禁 | 跨设备布局启动报配置错误、view 创建遇 `EXDEV` 拒绝的测试 | Partial |

### 代表性证据与剩余缺口

Node 侧验收：`Passed`。

- [exact_mcp_leases_allow_only_mcp_coexistence](https://github.com/ora-space/desktop/blob/28da0a4e77ec2257468b15bffd99e166a186b31c/apps/ora-node/tests/plugins.rs#L204)。
- [install_retry_upgrade_and_remove_respect_use_leases](https://github.com/ora-space/desktop/blob/28da0a4e77ec2257468b15bffd99e166a186b31c/apps/ora-node/tests/plugins.rs#L310)。
- [a_separate_placement_builds_a_package_view_and_a_private_home](https://github.com/ora-space/desktop/blob/28da0a4e77ec2257468b15bffd99e166a186b31c/apps/ora-node/src/session/workload/tests.rs#L96)。
- [automatic_confirmation_after_d5_releases_the_probe_lease](https://github.com/ora-space/desktop/blob/28da0a4e77ec2257468b15bffd99e166a186b31c/apps/ora-node/tests/standalone/agent_sessions/mcp_lifecycle/cleanup.rs#L11)。

精确版本租约与保留在用字节的 Node 验收通过。Separate view 与晚到探测租约释放各有证据，但双版本 view 的全部组合及真实跨设备 EXDEV 集成仍缺独立直接证据，保留 Partial。

### 决策依据

D6（租约与 view），D5“回收、部署与兼容”，R2；修改插件安装根决策 D2 / 不变量 4。

## The Lock Of An Agent Attempt Covers The Plugin Runtime And Every Host-Spawned Child

### 风险

锁只交给插件运行时，宿主随后 spawn 的 Agent CLI（另一进程组）不持锁；Node 崩溃且插件运行时退出后锁
可取得，CLI 与其 MCP 仍在写 checkout 却被判 Stopped；线索只指向插件运行时的进程组；停止请求结果替代原锁与线索；Node 关闭副本后仍有新宿主 spawn；登记后、spawn 前崩溃被当成未知或已运行。

### 前置状态

Node home 下有私有 `session-attempts` 目录；Agent 插件经 `ora/childprocess/*` 请宿主 spawn CLI；会话
spawner 为插件运行时与全部宿主子进程共用。

### 触发

插件运行时启动后请求宿主 spawn Agent CLI 与其他子进程；Node 崩溃；插件运行时先退出而 CLI 仍存活；停止
时某个子进程 kill 失败；停止请求后插件再请求 spawn；登记与 spawn 之间崩溃。

### 必须成立

- 锁文件与目录先 fsync，再持久登记尝试，再加独占 flock 并派发；锁 OFD 在 Node 内 close-on-exec，只交给
  本尝试指定的子进程。
- 一次 Agent 尝试 = 插件运行时 + 这一代插件请宿主 spawn 的全部子进程，每个都继承同一锁 OFD；Node 保留
  副本直到请求停止，先拒绝该代插件之后的宿主 spawn，再关闭副本，然后才探查锁。
- 每个进程组在 spawn 时各记一条 pid / start_ticks / boot_id / 主机标识线索；终止与直接进程检查覆盖全部
  线索；`kill_all` / `stop_plugin` 的返回只作诊断，成功不提前确认，错误也不跳过有界证据等待。
- 记录后、spawn 前崩溃留下已刷盘且空闲的锁，证明未启动；证据按 `StopContainment::BestEffort` 报告
  （进程组 + 继承锁 + 尝试环境标记），并假定 workload 配合（持 fd 的进程可主动解锁，清空环境后成为孤儿的后代
  不在证据内）。stdio 探测只有一次直接 spawn、一条线索。
- Linux 上 spawner 给尝试派发的每个进程最后写入 `ORA_SESSION_ATTEMPT=<attempt>`，调用方的环境不能去掉它；
  范围记录 `<attempt>.mark`（`since_ticks`、`uid`）与锁标记一起在登记前刷盘，账本登记时带范围记录标志。
- 标记扫描只认领尝试开始后启动、带标记或父进程已被认领的存活进程；停止时先经 pidfd 冻结（至多 32 轮）再全部
  杀掉，然后才终止线索进程组。
- 带标志的尝试另需连续两次扫描既无被认领进程也无待定进程，才确认停止并记 `lock_processes_and_marks`。
- 尝试用户的孤儿进程在 Node 同用户下读不到环境时不认领，也不使证据为 Unknown；Node 与 workload 不同用户且没有
  `CAP_SYS_PTRACE` 时，该用户读不到的进程为待定，证据为 Unknown。Node 启动时检查这一部署前提，不满足时告警、
  照常启动。

### 禁止结果

- 宿主子进程不持锁；任一持锁进程或任一线索的直接进程存活时判 Stopped；无锁/线索证据却仅凭停止调用结果显示已停止；
  停止请求后仍有宿主 spawn 成功；锁泄漏给其他线程并发 spawn 的进程；把 BestEffort 报成 Strong。
- 仍有被标记扫描认领的存活进程时判 Stopped；单次扫描干净即确认；先组杀后扫描，使清空环境的子进程成为孤儿；
  向尝试开始前启动的进程或 PID 复用的进程发信号；同用户读不到环境的无关进程让停止变为 Unknown；部署前提不满足
  时拒绝启动。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 宿主子进程继承同一锁 | 真实 Node 上插件运行时退出而宿主 spawn 的 CLI（独立进程组）存活时锁仍被占用、不判 Stopped 的测试 | Partial |
| 逐组线索与终止 | 两个进程组各有线索、重启后两组都被终止或保留责任的测试 | Partial |
| 停止调用结果不替代证据 | 注入 kill/stop 请求失败后仍等待原锁与线索；未确认不释放、取得真实证据后可确认的测试 | Partial |
| 停止后拒绝 spawn | 停止请求后插件的宿主 spawn 请求被拒绝的测试 | Covered |
| 登记与 spawn 的崩溃窗口 | 登记后、spawn 前崩溃重启得到空闲锁并确认未启动的测试 | Covered |
| 锁不泄漏 | 并发 spawn 的无关进程不继承锁 fd 的测试 | Partial |
| 每个进程带标记 | 宿主派发的每个进程（含调用方清空环境的）都带标记、范围记录与标志在登记前已持久化的测试 | Partial |
| 脱组作业由标记覆盖 | `setsid` 后以 `&` 起、关闭锁 fd 的作业存活时判 Running，停止后被杀，确认记 `lock_processes_and_marks` 的测试 | Covered |
| 冻结先于杀与组终止 | 冻结期间 fork 并清空环境的子进程经父进程被认领并杀掉的测试 | Partial |
| 窗口与 PID 复用 | 尝试开始前启动的带标记进程、PID 复用的进程不被发信号的测试 | Partial |
| 连续两次扫描 | 第一次扫描干净而第二次见到被认领进程时不确认的测试 | Covered |
| 读不到的进程与部署前提 | 同用户不可读孤儿不认领也不导致 Unknown、外用户无 `CAP_SYS_PTRACE` 下不可读进程为 Unknown、不满足前提时启动告警且照常服务的测试 | Partial |

### 代表性证据与剩余缺口

Node 侧验收：`Partial`。

- [plugin_and_each_host_child_share_one_attempt_while_siblings_stay_running](https://github.com/ora-space/desktop/blob/28da0a4e77ec2257468b15bffd99e166a186b31c/apps/ora-node/src/session/stop/tests.rs#L107)。
- [live_direct_child_that_closes_its_descriptor_is_not_stop_proof](https://github.com/ora-space/desktop/blob/28da0a4e77ec2257468b15bffd99e166a186b31c/apps/ora-node/src/session/stop/tests.rs#L150)。
- [a_durable_unstarted_lock_is_proof_but_missing_corrupt_or_foreign_boot_evidence_is_not](https://github.com/ora-space/desktop/blob/28da0a4e77ec2257468b15bffd99e166a186b31c/apps/ora-node/src/session/stop/tests.rs#L189)。
- [stop_wait_keeps_original_bound_and_does_not_treat_request_failure_as_proof](https://github.com/ora-space/desktop/blob/28da0a4e77ec2257468b15bffd99e166a186b31c/apps/ora-node/src/session/stop/tests.rs#L13)。

锁/线索和停止 helper 有直接测试；缺经生产 driver 注入 stop_plugin 错误仍等待原期限的路径（N-1），以及真实插件、Node 崩溃、各子组与并发无关 spawn 的完整组合。kill_all / stop_plugin 返回只作诊断，成功和失败都不能替代停止证据。

尝试环境标记（desktop#802 `198cc415`）：

- [a_marked_attempt_records_lock_processes_and_marks](https://github.com/ora-space/desktop/blob/198cc41517a837ec436a897655403adc790b8c0d/apps/ora-node/src/session/stop/tests/marks.rs#L195)：经生产 `prepare` 登记的尝试带范围记录标志，脱组作业被杀后确认为 `lock_processes_and_marks`。
- [a_detached_job_blocks_proof_until_terminate_kills_it](https://github.com/ora-space/desktop/blob/198cc41517a837ec436a897655403adc790b8c0d/apps/ora-node/src/session/stop/tests/marks.rs#L75)：`setsid` 后关闭全部继承 fd 的作业存活时判 Running，终止后才得到 `lock_processes_and_marks`。
- [stop_confirmation_terminates_plugin_and_cli_while_lifecycle_waits](https://github.com/ora-space/desktop/blob/198cc41517a837ec436a897655403adc790b8c0d/apps/ora-node/src/session/stop/tests.rs#L144)。
- [plugin_and_each_host_child_share_one_attempt_while_siblings_stay_running](https://github.com/ora-space/desktop/blob/198cc41517a837ec436a897655403adc790b8c0d/apps/ora-node/src/session/stop/tests.rs#L227)：插件与宿主子进程的停止证明改为 `lock_processes_and_marks`。
- [a_forking_parent_is_frozen_before_it_can_orphan_cleared_children](https://github.com/ora-space/desktop/blob/198cc41517a837ec436a897655403adc790b8c0d/crates/utils/tests/environment_mark_limits.rs#L250)：持续 fork 清空环境子进程的父进程被冻结，子进程一个不漏。
- [kill_reaches_orphans_and_env_cleared_children_but_not_strangers](https://github.com/ora-space/desktop/blob/198cc41517a837ec436a897655403adc790b8c0d/crates/utils/tests/environment_mark.rs#L114)。
- [an_unreadable_child_of_a_claimed_parent_is_killed_with_it](https://github.com/ora-space/desktop/blob/198cc41517a837ec436a897655403adc790b8c0d/crates/utils/tests/environment_mark.rs#L148)。
- [frozen_processes_die_at_the_bound_while_a_scan_hangs](https://github.com/ora-space/desktop/blob/198cc41517a837ec436a897655403adc790b8c0d/crates/utils/src/process/environment_mark/freeze.rs#L206)：扫描挂住时冻结的进程在上限内被杀。
- [frozen_processes_die_when_the_passes_finish](https://github.com/ora-space/desktop/blob/198cc41517a837ec436a897655403adc790b8c0d/crates/utils/src/process/environment_mark/freeze.rs#L240)。
- [a_marked_process_whose_leader_exited_is_claimed_and_killed](https://github.com/ora-space/desktop/blob/198cc41517a837ec436a897655403adc790b8c0d/crates/utils/tests/environment_mark_limits.rs#L181)：主线程已退出的线程组被认领并整体杀掉。
- [a_window_larger_than_the_descriptor_limit_is_still_scanned_and_killed](https://github.com/ora-space/desktop/blob/198cc41517a837ec436a897655403adc790b8c0d/crates/utils/tests/environment_mark_limits.rs#L72)：进程数超过 fd 软上限仍能扫描并杀掉，fd 耗尽时扫描失败而不是报干净。
- [scan_claims_only_marked_processes_started_within_the_scope](https://github.com/ora-space/desktop/blob/198cc41517a837ec436a897655403adc790b8c0d/crates/utils/tests/environment_mark.rs#L81)：尝试开始前启动的带标记进程与相近 token 的进程不被认领。
- [a_reused_pid_is_neither_pinned_nor_frozen](https://github.com/ora-space/desktop/blob/198cc41517a837ec436a897655403adc790b8c0d/crates/utils/src/process/environment_mark/freeze.rs#L181)：启动 tick 不符的 PID 不被 pin 也不被冻结。
- [a_parent_replaced_during_the_pass_lends_no_children](https://github.com/ora-space/desktop/blob/198cc41517a837ec436a897655403adc790b8c0d/crates/utils/src/process/environment_mark/scan.rs#L279)。
- [evidence_needs_two_clean_scans_without_undecided_processes](https://github.com/ora-space/desktop/blob/198cc41517a837ec436a897655403adc790b8c0d/apps/ora-node/src/session/stop/tests/marks.rs#L261)：先干净后认领为 Running，任一次待定或扫描失败为 Unknown。
- [classification_follows_parent_links_and_counts_foreign_unreadable_processes](https://github.com/ora-space/desktop/blob/198cc41517a837ec436a897655403adc790b8c0d/crates/utils/src/process/environment_mark/scan.rs#L241)：同用户扫描者的不可读孤儿不计为待定，root 扫描者的同类进程计为待定。
- [the_mark_scan_needs_a_shared_user_or_ptrace](https://github.com/ora-space/desktop/blob/198cc41517a837ec436a897655403adc790b8c0d/apps/ora-node/src/session/stop/marks.rs#L149)。
- [agent_startup_warns_when_the_mark_scan_cannot_read_the_workload](https://github.com/ora-space/desktop/blob/198cc41517a837ec436a897655403adc790b8c0d/apps/ora-node/src/service/agents/tests/startup.rs#L40)：其他 workload 用户且无 `CAP_SYS_PTRACE` 时告警一次，配置照常通过。

缺口：
- 每个进程带标记：没有直接检查子进程环境的测试，也没有覆盖 `ProcessEnvironment::Empty` 与 `remove_env_prefix` 的 spawn 仍带标记；范围记录先于登记只由 `prepare` 的调用顺序保证，没有在两者之间注入崩溃的测试。
- 冻结先于组终止：冻结与杀的顺序有直接测试，“标记杀先于线索进程组终止”只在 `files.rs` 的终止顺序中保证，没有测试；冻结上限之后改为直接杀的分支只由看门狗测试间接覆盖。pidfd 打不开时下一轮重试、收尾时重新确认失败的冻结进程留到下一次观察，都没有测试。
- 窗口与 PID 复用：尝试开始前的带标记进程只断言不被扫描认领，没有在 `kill` 之后断言它仍存活；同一 tick 内 PID 回绕是理论缺口，无法测试。
- 读不到的进程：分类规则用合成候选测试，没有在 Node 与 workload 分用户的真实部署（需要 root）下扫描真实进程；启动检查检测不到 Yama `ptrace_scope` 与 LSM 拒绝，这是决策接受的范围。

### 决策依据

D5“停止证据与记录”，不变量 7；扩展 Node 直接执行 Git 的尝试锁机制；[尝试环境标记决策](../../../decisions/node/agent-runtime/20261120-attempt-mark-in-session-stop-evidence.md) D1–D3、D6，不变量 1–5、11。

## A Session Stop Kills Every Attempt-Marked Process And Finished Turns Kill None

### 风险

Agent 工具起的后台作业（`cmd &`、`setsid`）在会话结束后仍写工作区，而停止被记为 `confirmed`（circus#1350）；
反过来，轮次之间或保留会话的取消杀掉 LSP、MCP 等常驻进程（保留会话的取消只杀这一轮的栅栏范围，见下文）；嵌套 Node 覆盖外层标记，外层停止漏掉内层的进程；
伪造的标记让 Node 杀到尝试开始前就存在的进程，或产生假的停止证据。

### 前置状态

Linux；部署的 OpenCode 与 Claude Agent 插件；工具 shell 起了后台作业，会话中还有开发服务器、语言服务与共享
守护进程；可能有在会话里运行的嵌套 Node；同 uid 的其他进程可以读到标记。

### 触发

EndSession、Agent 失败、导致会话结束的取消、Node 优雅关闭、Node 被 SIGKILL 后的重启收口；一轮正常完成；会话被
保留的取消或失败；外层尝试停止；伪造者持续启动带标记的进程。

### 必须成立

- 每种会话结束方式下，两个部署插件的后台作业（`cmd &` 与 `setsid`）都在 Node 报告停止之前被杀，之后不再写入；
  作业存活时不记 `confirmed`。
- 被杀的包括开发服务器、文件监视进程、MCP 与语言服务，以及会话中启动的共享守护进程，即使其他会话仍在使用。
- 一轮正常完成后，Node 不发任何冻结或杀信号，带标记的常驻进程跨轮次存活。会话被保留的取消或失败之后，按
  [轮次进程栅栏决策](../../../decisions/node/protocol/20261123-turn-process-fence-before-keeping-session.md)（`implemented`）只杀
  这一轮新起、且不在“这一轮开始前已启动、此刻仍存活的本尝试进程”所在会话中的进程，常驻进程同样存活；栅栏的义务见
  [轮次进程栅栏用例](../protocol/turn-process-fence.md)，本用例不重复登记。
- 嵌套 Node 给子进程的值为 `外层:attempt`；外层尝试停止时内层 Node 及其全部尝试的进程被杀；内层 Node 只认领
  列出自己 token 的进程。
- 伪造标记的进程只有在尝试开始之后启动的才会被认领；持续伪造只把确认拖到 `stop_unconfirmed`，不产生停止证据。

### 禁止结果

- 会话报告停止后后台作业仍写入；作业存活时停止被确认；轮次之间或保留会话的取消杀掉常驻进程；内层 Node 覆盖
  外层 token，或认领外层尝试的进程；伪造使 Node 向尝试开始前的进程发信号，或使确认提前。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 每种结束方式都杀后台作业 | 两个部署插件分别在 EndSession、导致会话结束的取消、Node 优雅关闭、SIGKILL 后重启下，`cmd &` 与 `setsid` 作业在停止报告前退出、之后不再写入的真实 Node 测试 | Partial |
| 守护进程随会话结束 | 会话中启动、其他会话仍在使用的守护进程随该会话结束被杀的测试 | Partial |
| 轮次之间与保留会话不杀常驻进程 | 一轮正常完成后带标记的进程仍存活、会话被保留的取消之后轮次开始前的常驻进程仍存活的测试；保留会话的取消杀掉本轮进程由[轮次进程栅栏用例](../protocol/turn-process-fence.md)登记。证据（desktop#807 `447a59de`）：[`jobs_of_a_finished_turn_survive_the_next_turns_cancel`](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/apps/ora-node/tests/agent_session/turn_processes.rs#L195) 与 [`the_agents_own_helper_survives_a_kept_cancel`](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/apps/ora-node/tests/agent_session/turn_processes.rs#L157) | Covered |
| 嵌套 Node 标记链 | 外层停止杀掉内层 Node 的全部进程、内层停止不触及外层其他进程的测试 | Partial |
| 伪造标记的边界 | 尝试开始前启动的带标记进程不被发信号、持续伪造时确认不提前而转为 `stop_unconfirmed` 的测试 | Partial |

### 代表性证据与剩余缺口

desktop#802 `198cc415` 的真实 Node 测试用 echo-agent fixture 模拟部署插件的工具 shell：`setsid` 下的 `cmd &`，关闭全部
继承 fd，被 init 收养。每个用例都由一个线程同时采样账本与作业，作业存活时一旦看到 `confirmed` 即失败，并在作业的写入时刻之后
确认它没有写出文件：

- [end_session_kills_a_detached_tool_job_and_spares_unmarked_processes](https://github.com/ora-space/desktop/blob/198cc41517a837ec436a897655403adc790b8c0d/apps/ora-node/tests/standalone/agent_sessions/background_jobs.rs#L150)：一轮结束后 EndSession；不带标记、带其他尝试或相近 token 的进程存活，尝试开始后伪造本尝试标记的进程被杀。
- [ending_a_running_turn_kills_its_detached_tool_job](https://github.com/ora-space/desktop/blob/198cc41517a837ec436a897655403adc790b8c0d/apps/ora-node/tests/standalone/agent_sessions/background_jobs.rs#L211)：轮次仍在运行时 EndSession。
- [restart_recovery_kills_the_job_a_killed_node_left_behind](https://github.com/ora-space/desktop/blob/198cc41517a837ec436a897655403adc790b8c0d/apps/ora-node/tests/standalone/agent_sessions/background_jobs.rs#L249)：Node 被 SIGKILL 后作业仍存活，重启收口先杀作业再记 interrupted。
- [graceful_shutdown_kills_detached_tool_jobs_before_the_node_exits](https://github.com/ora-space/desktop/blob/198cc41517a837ec436a897655403adc790b8c0d/apps/ora-node/tests/standalone/agent_sessions/background_jobs.rs#L301)：SIGTERM 优雅关闭，Node 退出时作业已死。
- [an_unreadable_job_under_a_waiting_tool_shell_is_killed_with_it](https://github.com/ora-space/desktop/blob/198cc41517a837ec436a897655403adc790b8c0d/apps/ora-node/tests/standalone/agent_sessions/background_jobs.rs#L350)：不可 dump 的作业经仍在等待它的工具 shell 被认领。
- [marks_match_whole_tokens_under_the_exact_name](https://github.com/ora-space/desktop/blob/198cc41517a837ec436a897655403adc790b8c0d/crates/utils/src/process/environment_mark.rs#L190)：嵌套值 `outer:attempt` 的生成与整段 token 匹配。
- [scan_claims_only_marked_processes_started_within_the_scope](https://github.com/ora-space/desktop/blob/198cc41517a837ec436a897655403adc790b8c0d/crates/utils/tests/environment_mark.rs#L81)：`outer:token` 被认领，`token-other` 与尝试开始前的带标记进程不被认领。

缺口：
- 结束方式：仓库里没有经 `CancelTurn` 以 `session_ending` 结束会话的用例，“轮次仍在运行时结束”走的是 EndSession；评审的临时
  用例与真实插件复测（运行记录）表明取消路径同样先杀作业，补进仓库后再登记。部署插件的优雅关闭没有真实复测。
- 守护进程：fixture 作业与被收养的守护进程在扫描面前相同，但没有“另一会话仍在使用”的用例。
- 轮次之间与保留会话：已由 desktop#807 `447a59de` 补齐。[`jobs_of_a_finished_turn_survive_the_next_turns_cancel`](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/apps/ora-node/tests/agent_session/turn_processes.rs#L195) 断言正常完成的一轮
  留下的带标记 `setsid` 作业与 Agent 会话中的 helper 跨轮次存活，并活过下一轮保留会话的取消，会话结束时作业才被杀；
  [`the_agents_own_helper_survives_a_kept_cancel`](https://github.com/ora-space/desktop/blob/447a59de6722500cae0be011860e63e5127a3b23/apps/ora-node/tests/agent_session/turn_processes.rs#L157) 断言 Agent 在被取消的一轮里于自己会话中起的 helper 活过保留会话的取消，
  下一轮找到同一进程。
- 嵌套 Node：只有标记值与匹配的单测，没有真实的嵌套 Node 用例。
- 伪造：持续伪造把确认拖到 `stop_unconfirmed` 没有测试。

macOS 与 Windows 不在本用例范围内。

### 决策依据

[尝试环境标记决策](../../../decisions/node/agent-runtime/20261120-attempt-mark-in-session-stop-evidence.md) D4、D5、D8，不变量 7、8；本文决策 D5“终态、转交与门禁”；[轮次进程栅栏决策](../../../decisions/node/protocol/20261123-turn-process-fence-before-keeping-session.md) D2、D7（修改标记决策 D4 中保留会话的一条）。

## Restart Recovery Confirms A Stop Only On The Same Kernel Or The Same Host

### 风险

Node 重启后把“锁可取得”在另一台主机上当作停止证据；boot_id 不同就认定原进程不存在，而 Workspace 卷
其实是从仍在运行的主机迁来；容器镜像的 machine-id 被当作主机标识；锁缺失被当成未启动；重启刷新
停止时钟；容器或 Node 进程重启时正常会话迟迟拿不到 interrupted。

### 前置状态

停止责任记录含原 Node 实例、boot_id、主机标识与线索；部署可能声明 Workspace 卷单点挂载且重新挂载前
已隔离原主机。

### 触发

同一内核上的 Node 或容器重启；同一主机重启；卷迁到另一主机；主机标识缺失或来自容器镜像；锁文件缺失或
损坏。

### 必须成立

- boot_id 相同：锁可取得且全部线索的直接进程都已不在时按 BestEffort 立即确认并结算 interrupted；锁被占用
  或任一直接进程存活时按已验证线索终止后再查；锁缺失、损坏或不可读为 Unknown。
- 同一 boot 上，带范围记录标志的尝试另需连续两次标记扫描干净才确认并记 `lock_processes_and_marks`；仍有被认领的
  进程时先冻结并杀掉、再终止线索进程组，然后再查；不带标志的旧尝试不扫描，记 `lock_and_processes`；带标志而
  范围记录缺失或损坏、procfs 不可读、扫描有待定进程时为 Unknown。
- boot_id 不同且记录的主机标识与当前相同：同机重启，可记录宿主重启证据确认。
- boot_id 不同且主机标识不同、缺失或不可信：Unknown，交给 `recheck` / 对账，除非部署声明单点挂载与重新
  挂载前隔离。容器镜像自带、各实例相同或为空的 machine-id 不算主机标识。
- 停止时钟首次锚定后跨重试与重启不续期；Node 先恢复围栏和 catalog 版本保护，再允许安装、回收和准入。

### 禁止结果

- 跨主机或缺主机标识时自动确认停止；PID 复用或 boot 不符时按线索发信号；锁缺失被当成未启动；重启重置
  停止时钟或返还探测额度。
- 残留的带标记作业未被杀就确认；带标志的尝试在范围记录缺失时降级为 `lock_and_processes`；不带标志的旧尝试被记成
  `lock_processes_and_marks`。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 同内核快速收口 | 容器 / Node 进程重启后空闲锁使会话立即 interrupted、不等满停止期限的测试 | Partial |
| 同机重启 | 主机标识相同、boot_id 不同的记录被确认的测试 | Covered |
| 跨主机不确认 | 主机标识不同或缺失时保持 Unknown、声明单点挂载后才确认的测试 | Partial |
| 锁缺失为 Unknown | 删除或损坏锁文件后保持 Unknown 并转交对账的测试 | Covered |
| 时钟与恢复顺序 | 多次重启后 first_stop_at_ms 不变、围栏先于准入恢复的测试 | Partial |
| 重启收口杀残留作业 | Node 被 SIGKILL 后留下的脱组作业在重启收口中被杀、之后才确认的测试 | Covered |
| 范围记录与标志 | 带标志而 `.mark` 缺失或损坏为 Unknown、procfs 不可读为 Unknown、不带标志的旧尝试记 `lock_and_processes` 的测试 | Covered |

### 代表性证据与剩余缺口

Node 侧验收：`Partial`。

- [foreign_or_missing_host_identity_cannot_turn_a_boot_change_into_stop_proof](https://github.com/ora-space/desktop/blob/28da0a4e77ec2257468b15bffd99e166a186b31c/apps/ora-node/src/session/stop/tests.rs#L66)。
- [a_durable_unstarted_lock_is_proof_but_missing_corrupt_or_foreign_boot_evidence_is_not](https://github.com/ora-space/desktop/blob/28da0a4e77ec2257468b15bffd99e166a186b31c/apps/ora-node/src/session/stop/tests.rs#L189)。
- [restart_preserves_clock_identity_terminal_intent_and_resource_fence](https://github.com/ora-space/desktop/blob/28da0a4e77ec2257468b15bffd99e166a186b31c/crates/node-db/src/tests/session/stop.rs#L94)。

原时钟和身份持久性、缺锁、跨主机拒绝有证据；真实卷迁移、单点挂载声明与重启恢复/准入排序的全路径仍缺。不能把直接函数或数据库重开当作完整宿主重启实验。

尝试环境标记（desktop#802 `198cc415`）：

- [restart_recovery_kills_the_job_a_killed_node_left_behind](https://github.com/ora-space/desktop/blob/198cc41517a837ec436a897655403adc790b8c0d/apps/ora-node/tests/standalone/agent_sessions/background_jobs.rs#L249)：真实 Node 被 SIGKILL 后重启，残留作业先被杀，账本在作业存活时从未 `confirmed`，最终记 `lock_processes_and_marks`。
- [a_scoped_attempt_whose_scope_is_lost_is_unknown](https://github.com/ora-space/desktop/blob/198cc41517a837ec436a897655403adc790b8c0d/apps/ora-node/src/session/stop/tests/marks.rs#L135)：带标志而 `.mark` 被删除：Unknown，到期记 `stop_unconfirmed`。
- [a_surviving_job_or_damaged_scope_is_never_proof](https://github.com/ora-space/desktop/blob/198cc41517a837ec436a897655403adc790b8c0d/apps/ora-node/src/session/stop/tests/marks.rs#L109)：`.mark` 损坏为 Unknown。
- [an_unscoped_attempt_keeps_lock_and_processes](https://github.com/ora-space/desktop/blob/198cc41517a837ec436a897655403adc790b8c0d/apps/ora-node/src/session/stop/tests/marks.rs#L229)：不带标志、没有 `.mark` 的旧尝试记 `lock_and_processes`。
- [evidence_needs_two_clean_scans_without_undecided_processes](https://github.com/ora-space/desktop/blob/198cc41517a837ec436a897655403adc790b8c0d/apps/ora-node/src/session/stop/tests/marks.rs#L261)：扫描失败为 Unknown。
- [a_window_larger_than_the_descriptor_limit_is_still_scanned_and_killed](https://github.com/ora-space/desktop/blob/198cc41517a837ec436a897655403adc790b8c0d/crates/utils/tests/environment_mark_limits.rs#L72)：fd 耗尽时扫描报错而不是报干净。

### 决策依据

D5 重启证据表与主机标识段，不变量 6、7；[尝试环境标记决策](../../../decisions/node/agent-runtime/20261120-attempt-mark-in-session-stop-evidence.md) D1、D3，不变量 2、3。

## Terminal Waits For Stop Evidence And Expiry Hands Over As Stop Unconfirmed

### 风险

终态先于停止证据发出，残留进程仍写 checkout；`stop_plugin` 失败后只记 warning 并释放租约、删除 view；
等待无界挂住 driver；超时被当作停止证据；`Completed` 同时携带 `stop_unconfirmed`；EndSession 因清理慢
阻塞控制连接或兄弟会话。

### 前置状态

execution 有 Agent 尝试与探测责任；任一可能无法在默认 10 分钟内取得停止确认。

### 触发

EndSession；`stop_plugin` 失败；停止等待超过上限；之后迟到的停止证据；Node 重启时缺证据。

### 必须成立

- EndSession 先停止准入、取消探测、请求停止 Agent 并正常回复受理，不阻塞心跳或兄弟会话。
- 只有本 execution 全部 Agent / 探测责任取得停止确认后才发唯一 `AgentSessionEnded`；对应使用者的租约 /
  view 在其确认后释放。
- 首次停止请求后有界等待（默认 10 分钟，可配置的正有限值）；超过上限或重启收口缺证据时登记
  `stop_unconfirmed` 并转交残留对账；`GetExecutionStatus` 返回原 `StopUnconfirmed` 身份；迟到证据才补终态，
  首次原因与序号不变。可能始终没有终态（R9）。
- `stop_plugin` 失败时生命周期 owner 接管租约、placement / view，不 drop。

### 禁止结果

- 终态早于停止确认；`Completed` 带 `stop_unconfirmed`；超时、断线或 future drop 解除责任；停止未知时释放
  租约或删除 view；清理阻塞其他会话的命令。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 终态在证据之后 | 注入 Agent 延迟退出，终态只在确认后发出的测试 | Covered |
| 有界等待与转交 | 受控时钟下第 10 分钟登记 `stop_unconfirmed`、结束等待任务、无终态的测试 | Partial |
| 迟到证据补终态 | 转交后取得原证据补发终态且原因与序号不变的测试 | Covered |
| `stop_plugin` 失败不释放资源 | 注入 stop 失败后租约与 view 保留的 driver 测试 | Partial |

### 代表性证据与剩余缺口

Node 侧验收：`Partial`。

- [normal_end_does_not_publish_terminal_until_original_stop_evidence_returns](https://github.com/ora-space/desktop/blob/28da0a4e77ec2257468b15bffd99e166a186b31c/apps/ora-node/tests/standalone/agent_sessions/mcp_lifecycle/races.rs#L20)。
- [probe_budget_reports_timeout_once_while_original_stop_bound_retains_capacity](https://github.com/ora-space/desktop/blob/28da0a4e77ec2257468b15bffd99e166a186b31c/apps/ora-node/tests/standalone/agent_sessions/mcp_lifecycle/cleanup.rs#L6)。
- [automatic_confirmation_after_d5_releases_the_probe_lease](https://github.com/ora-space/desktop/blob/28da0a4e77ec2257468b15bffd99e166a186b31c/apps/ora-node/tests/standalone/agent_sessions/mcp_lifecycle/cleanup.rs#L11)。
- [stop_wait_keeps_original_bound_and_does_not_treat_request_failure_as_proof](https://github.com/ora-space/desktop/blob/28da0a4e77ec2257468b15bffd99e166a186b31c/apps/ora-node/src/session/stop/tests.rs#L13)。

正常 End、探测预算和自动晚到确认有直接证据；N-1 的生产 driver 调用仍没有会杀死回退变异的测试，N-3 的 EndSession/探测 bound_at_ms 竞争修复没有确定性测试。

### 决策依据

D5“终态、转交与门禁”第 1–4 条，不变量 6、10；继承停止确认决策的有限等待与迟到确认。

## Stop Unconfirmed Fences Restores And Its Own Delivery But Not Sibling Sessions

### 风险

停止未知的残留进程仍写 checkout 时，带 `prior_revision` 的恢复移动分支与工作区，或该 execution 被交付；
反过来，正常停止中的前序会话或残留责任让普通后继会话以持久的 `checkout_busy` 失败；兄弟会话已结算的
交付被残留拒绝；新规则放宽既有恢复围栏。

### 前置状态

checkout C 上有会话 A 处于正常运行、期限内正常停止、`stop_unconfirmed` 或升级遗留 `legacy_unknown`，或
已确认停止；另有兄弟会话。

### 触发

在 C 上启动普通新会话与带 `prior_revision` 的会话；交付 A；交付已结算的兄弟会话；A 的责任被证据解除。

### 必须成立

- A 运行或期限内正常停止：不增加 `checkout_busy`；A 仍占 8 个未结束名额；恢复仍受原“有未结束会话不可
  恢复”规则。
- A 为 `stop_unconfirmed` 或 `legacy_unknown`：普通新会话不增加拒绝、仍按共享规则与数量准入；带
  `prior_revision` 的恢复以 `checkout_busy` 拒绝；A 自己的交付以 `session_not_settled` 拒绝，直到原责任被
  证据解除。
- 兄弟会话已结算后的交付不因 A 的残留被拒绝。
- 判定与现有准入在同一事务内；停止责任日志不可读时不作出“无残留”的肯定判断；既有 `session_restores`
  legacy_unknown 与 `restore_process_attempts` 围栏不变。

### 禁止结果

- 正常停止的 A 使普通后继 B 被拒；残留下恢复被受理或 A 被交付；兄弟的交付被残留拒绝；日志不可读时
  放行恢复；既有恢复围栏被放宽。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 正常停止不挡后继 | A 停止请求后、确认前启动普通会话 B 被受理的准入测试 | Partial |
| 残留挡恢复与自身交付 | A 为 `stop_unconfirmed` / `legacy_unknown` 时 `prior_revision` 恢复得 `checkout_busy`、A 的交付得 `session_not_settled` 的测试 | Covered |
| 不挡兄弟 | 同时普通新会话被受理、已结算兄弟的交付成功的测试 | Partial |
| 证据解除后放行 | 对账确认后恢复被受理的测试 | Covered |
| 日志不可读与既有围栏 | 停止责任不可读时拒绝恢复、既有恢复围栏行为不变的测试 | Partial |

### 代表性证据与剩余缺口

Node 侧验收：`Passed`。

- [prior_revision_is_fenced_by_residual_stop_even_with_a_legacy_terminal_row](https://github.com/ora-space/desktop/blob/28da0a4e77ec2257468b15bffd99e166a186b31c/crates/node-db/src/tests/session/stop.rs#L40)。
- [restart_preserves_clock_identity_terminal_intent_and_resource_fence](https://github.com/ora-space/desktop/blob/28da0a4e77ec2257468b15bffd99e166a186b31c/crates/node-db/src/tests/session/stop.rs#L94)。
- [an_mcp_session_shares_a_checkout_with_a_plain_one](https://github.com/ora-space/desktop/blob/28da0a4e77ec2257468b15bffd99e166a186b31c/apps/ora-node/tests/agent_session/mcp.rs#L339)。

残留围栏独立于 execution 表面终态、确认后解除已直接证明，N4 变异被杀；Node 验收 Passed。正常停止与兄弟交付全部组合、不可读日志错误注入尚未逐项完整覆盖，不把共享会话的较低层回归扩张成全矩阵。

### 决策依据

D5 门禁表与交付说明，不变量 10；扩展共享 checkout 决策 D2，与其 D5 的交付语义一致。

## An Undrained Upgrade Marks Legacy Sessions Legacy Unknown Instead Of Refusing To Start

### 风险

升级时未排空的旧会话让新二进制在迁移后拒绝启动，而旧二进制已无法打开新 schema，Workspace 永久不可用；
或者把没有尝试锁的旧会话直接结算为 interrupted，残留旧 Agent 继续写 checkout；accepted 的旧执行被无谓
围住；迁移后仍允许回滚到旧 schema 或旧备份。

### 前置状态

旧 schema 中有 running、accepted 和已完成的 Agent 会话执行；旧会话没有尝试锁；部署可能提供主机标识或
单点挂载保证。

### 触发

崩溃或 OOM 后直接以新镜像启动；之后同机重启宿主、对账 `recheck` / `after-host-reboot`、换主机。

### 必须成立

- 排空检查在任何迁移步骤之前执行，只读取旧 schema；未排空时不拒绝启动，照常迁移。
- 同一迁移事务为每个仍运行的旧 Agent 会话写入 `legacy_unknown` 停止责任（`session_stop_attempts` 中的
  独立状态，不是 `session_restores.phase`），并记下当时的主机标识与 boot_id 作为参照；accepted 的旧执行
  不标；已完成的旧执行只保留历史终态。
- `legacy_unknown` 不重启原 Agent，挡 `prior_revision` 恢复和该执行自己的交付，不挡普通新会话，继续占
  未结束名额；只有启动收口、`recheck` 或 `after-host-reboot` 核实同机重启（参照主机标识相同、boot_id 已变），
  或部署的单点挂载保证，才结算为 interrupted；不能单凭“现在找不到进程”解除；缺证据时可能始终没有终态。
- 不支持原地回滚旧二进制 / schema 或把账本恢复到旧备份。
- 同一 boot 的 `lock_and_processes` 与 `lock_processes_and_marks` 都不能解除 `legacy_unknown`。
- 停止责任表的迁移在一个事务中重建表：放宽证明 CHECK 接受 `lock_processes_and_marks`，新增范围记录标志列，全部行
  保留且不带标志；版本排在 desktop#793 所用 schema 版本之后；旧 Node 拒绝打开新库。未排空升级时早于标记的存活
  尝试保持 `lock_and_processes`，不新增规则。

### 禁止结果

- 未排空时拒绝启动；迁移后才做排空检查；旧会话被直接结算 interrupted；accepted 旧执行被标
  `legacy_unknown`；`recheck` 仅因找不到进程就解除；回滚路径被接受。
- 以 `lock_processes_and_marks` 解除 `legacy_unknown`；迁移丢行、改写既有证明或给旧行补上标志。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 迁移前排空检查且不拒绝启动 | 以含 running 旧会话的 v13 数据库启动新二进制，迁移成功且 Node 正常服务的迁移测试 | Partial |
| legacy_unknown 登记范围 | 迁移后 running 旧会话有 `legacy_unknown` 及参照主机 / boot，accepted 与已完成的没有的测试 | Covered |
| 围栏与出口 | legacy_unknown 挡恢复与自身交付、不挡普通会话，同机重启或单点挂载声明后才 interrupted 的测试 | Partial |
| 不可回滚 | 旧二进制拒绝新 schema、备份回滚流程被拒的测试 | Missing |
| 新证明不解除 legacy_unknown | 账本拒绝以 `lock_processes_and_marks` 确认 `legacy_unknown` 的测试 | Covered |
| 标记迁移保留记录 | 上一版数据库升级后全部停止行原样保留且不带标志、旧表拒绝新证明而新表接受的迁移测试 | Covered |

### 代表性证据与剩余缺口

Node 侧验收：`Partial`。

- [undrained_v13_upgrade_marks_only_running_agents_legacy_unknown](https://github.com/ora-space/desktop/blob/28da0a4e77ec2257468b15bffd99e166a186b31c/crates/node-db/src/tests/session/stop.rs#L232)。
- [restart_preserves_clock_identity_terminal_intent_and_resource_fence](https://github.com/ora-space/desktop/blob/28da0a4e77ec2257468b15bffd99e166a186b31c/crates/node-db/src/tests/session/stop.rs#L94)。

真实 v13 数据库迁移、登记范围和幂等已验证；完整新二进制服务、legacy_unknown 运维出口及旧二进制/旧备份回滚拒绝的端到端证据仍缺。

尝试环境标记（desktop#802 `198cc415`，schema 16）：

- [undrained_v13_upgrade_marks_only_running_agents_legacy_unknown](https://github.com/ora-space/desktop/blob/198cc41517a837ec436a897655403adc790b8c0d/crates/node-db/src/tests/session/stop.rs#L244)：`lock_and_processes` 与 `lock_processes_and_marks` 都不能确认 `legacy_unknown`，`host_reboot` 可以。
- [stop_marks_upgrade_keeps_rows_unflagged_and_admits_the_mark_scan_proof](https://github.com/ora-space/desktop/blob/198cc41517a837ec436a897655403adc790b8c0d/crates/node-db/src/tests/session/stop.rs#L414)：v15 表拒绝新证明；升级到 16 后已确认行除标志为“不带”外原样保留，新证明与标志可写可读。
- [v14_and_v15_files_upgrade_in_one_step_with_attempts_unflagged](https://github.com/ora-space/desktop/blob/198cc41517a837ec436a897655403adc790b8c0d/crates/node-db/src/tests/session/stop.rs#L516)：v14 与 desktop#793 的 v15 库都一次升到 16，待确认尝试不带标志并以 `lock_and_processes` 确认，v15 的取消结算保留。

旧 Node 拒绝打开 schema 16 属于上表“不可回滚”一行，仍为 `Missing`。

### 决策依据

D5“回收、部署与兼容”（升级段），沿用恢复迁移的 `legacy_unknown` 先例；R5、R9；[尝试环境标记决策](../../../decisions/node/agent-runtime/20261120-attempt-mark-in-session-stop-evidence.md) D3、D7，不变量 6、10。

## Operator Reconciliation Is Root-Only Audited And Never Forces A Release

### 风险

非 root 进程或自报 UID 的调用方解除围栏或释放探测名额；运维入口另开数据库与运行中的 Node 竞争；没有
证据的“强制清标记”；审计缺失、可含秘密或自由文本；重复请求重复执行；确认记录回收删除未确认责任或
裁掉账本事件；记录无界增长。

### 前置状态

Node 运行中，提供私有目录下的本地 Unix socket；存在 `stop_unconfirmed`、`legacy_unknown` 或耗尽探测名额
的 cleanup_pending 责任；已有确认满 7 天的记录。

### 触发

root 与非 root 连接 socket；以同一 request_id 重复调用、改输入重发；`recheck` 遇到仍运行 / Unknown /
Stopped；`after-host-reboot` 遇到同机重启、换主机、缺主机标识；审计写入失败；执行中崩溃；每小时回收。

### 必须成立

- 入口是运行中 Node 提供的 root-only Unix socket，以 `SO_PEERCRED` 取得对端 UID 并只接受 UID 0；CLI 只是
  客户端，不另开数据库；每次明确 execution、原 attempt、幂等 request_id 与 reason_code，不按模糊 PID 或
  目录批量释放。
- `recheck` 至多 10 秒核对原锁、全部线索与尝试标记扫描，必要时冻结并杀掉被认领的进程，得到 Stopped 才确认，
  否则不刷新首次停止时间、不释放许可；对 `legacy_unknown` 只重新核对同机重启条件。
- `after-host-reboot` 只在自动收口未确认时使用，按重启证据表同一规则核对；主机不同、缺主机标识或同 boot
  缺锁都拒绝；`lock_and_processes` 与 `lock_processes_and_marks` 都是同一 boot 的证明，不被它接受。
- 先写审计请求再执行，完成后写结果（操作者 UID、request_id、原身份、原因码、证据类型、前后状态、释放的
  资源）；审计写入失败则不提交解除；重复 request_id 返回同一结果，输入不同为冲突；崩溃后重新查证。
- 已确认且无引用的停止明细满 7 天后每小时分批（每批最多 1,000 条）回收；未确认记录永不按年龄删除；
  审计至少保留 90 天；回收不裁账本事件或 Ack 历史。

### 禁止结果

- 非 root 或自报 UID 成功；第二进程打开数据库；无证据解除围栏或返还名额；审计含秘密、命令行或自由文本；
  先删证据后提交确认；未确认记录被回收；`recheck` 在仍有被认领进程时确认；`after-host-reboot` 以同一 boot 的证明解除。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| root-only 与 `SO_PEERCRED` | 非 root 连接被拒、审计 UID 来自对端凭据的测试 | Partial |
| `recheck` 语义 | Running / Unknown / Stopped 三种结果与 legacy_unknown 只核对同机重启的测试 | Partial |
| `after-host-reboot` 语义 | 同机重启确认、换主机与缺主机标识拒绝的测试 | Partial |
| 审计与幂等 | 审计写入失败不解除、重复 request_id 同结果、崩溃后重新查证的测试 | Partial |
| 记录回收 | 7 天后回收已确认记录、保留未确认与被引用记录、审计 90 天的测试 | Covered |
| `recheck` 扫描标记 | `recheck` 冻结并杀掉被认领进程、扫描不干净时不确认的测试 | Partial |
| `after-host-reboot` 拒绝同 boot 证明 | `after-host-reboot` 拒绝 `lock_and_processes` 与 `lock_processes_and_marks` 的直接测试 | Covered |

### 代表性证据与剩余缺口

Node 侧验收：`Partial`。

- [root_reconciliation_wakes_probe_owner_and_lists_named_residues](https://github.com/ora-space/desktop/blob/28da0a4e77ec2257468b15bffd99e166a186b31c/apps/ora-node/tests/standalone/agent_sessions/mcp_lifecycle/cleanup.rs#L142)。
- [audit_is_root_only_idempotent_and_commits_proof_before_release](https://github.com/ora-space/desktop/blob/28da0a4e77ec2257468b15bffd99e166a186b31c/crates/node-db/src/tests/session/stop.rs#L181)。
- [gc_requires_age_release_ack_and_completed_audit_and_retains_a_summary](https://github.com/ora-space/desktop/blob/28da0a4e77ec2257468b15bffd99e166a186b31c/crates/node-db/src/tests/session/stop.rs#L306)。

root socket 用例默认 #[ignore]，只在显式 root 环境运行；数据库审计不是 SO_PEERCRED 证明。root 对账唤醒、中断审计恢复唤醒及残留过滤没有默认 CI 的独立回归（N-2）；真实非 root 对端和全部审计失败/重启矩阵仍缺。

尝试环境标记（desktop#802 `198cc415`）：

- [after_host_reboot_refuses_both_same_boot_proofs](https://github.com/ora-space/desktop/blob/198cc41517a837ec436a897655403adc790b8c0d/apps/ora-node/src/session/stop/admin.rs#L329)：`after-host-reboot` 只接受 `host_reboot` 与 `fenced_volume`，`recheck` 接受全部四种。
- [a_deadline_narrows_the_budget_and_a_spent_one_starts_nothing](https://github.com/ora-space/desktop/blob/198cc41517a837ec436a897655403adc790b8c0d/apps/ora-node/src/session/stop/tests.rs#L410)：`recheck` 的每一步按 10 秒截止时间收紧预算，用完后不再派发。
- [a_hung_observation_costs_its_budget_and_holds_the_worker](https://github.com/ora-space/desktop/blob/198cc41517a837ec436a897655403adc790b8c0d/apps/ora-node/src/session/stop/tests.rs#L389)。

缺口：`recheck` 复用会话结束路径的观察与终止（扫描、冻结、杀由上面尝试锁与本文其他用例的证据覆盖），但没有经 `recheck` 本身杀掉被认领作业、或在扫描不干净时拒绝确认的用例；root socket 用例仍默认 ignored。

### 决策依据

D5“留审计记录的运维对账”与“回收、部署与兼容”首段；R5、R10；[尝试环境标记决策](../../../decisions/node/agent-runtime/20261120-attempt-mark-in-session-stop-evidence.md) D3，不变量 6。

## Probes Start Their Budget At The Permit And Never Abandon Cleanup

### 风险

排队中的探测被报为 `mcp_probe_timeout`；探测以 Node 管理身份或任意 cwd 运行；同身份重复外连；手动重测
在 Controller 丢计数后静默失效；旧 generation 的晚到结果覆盖当前成员；清理未确认就释放名额 / 租约或重启
后返还额度；探测杀到 Agent 或兄弟会话。

### 前置状态

Node 全局 8 个、每 execution 2 个探测许可；当前 desired 中有最多 16 个 configured 成员；workload UID 与
checkout 绝对 cwd 已知。

### 触发

16 个成员同时补探测；手动 `probe_attempt` 相同、更小、更大；Controller 重启后重测；探测超时且进程组无法
在 2 秒内确认退出；Node 重启；8 个 cleanup_pending 占满名额。

### 必须成立

- 排队期间保持 Unknown / queued，不启动 10 秒预算，不判 Unhealthy；取得许可后 10 秒覆盖 spawn、握手与
  正常清理（至少最后 2 秒用于关闭 stdin、终止进程组并 wait）。
- stdio 探测以对应 Agent 的 workload 身份、受控基础环境与独立进程组运行，current_dir 为 Node 从 checkout
  解析的绝对 cwd；缺上下文时 Unknown(context_missing)。
- 同身份 single-flight；相同 attempt 为 duplicate、更小为 stale、更大加入现有 flight 或新建意图；快照暴露
  最高 attempt；自动触发用 0。新 generation 不复用旧健康，晚到结果不写回当前成员。
- 预算内停止未确认时给出 `mcp_probe_timeout` 与 cleanup_pending，保留名额、租约、view 并登记停止责任；
  恢复出的未确认探测计入额度；名额全被占用时显示“探测容量不可用”。
- stdio 探测尝试的进程同样带自己的尝试标记；清理时先冻结并杀掉被认领的进程，再终止进程组，确认按尝试环境标记决策 D3 记
  `lock_processes_and_marks`。

### 禁止结果

- 排队成员被判超时；探测继承 Node 管理密钥或全量环境；同身份并发两次外连；旧结果覆盖当前身份；未确认
  清理释放名额或重启返还；探测进程加入 Agent 组或杀死兄弟；探测清理认领 Agent 尝试的带标记进程。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 预算从许可开始 | 16 成员、2 个执行许可下末尾成员不因排队超时的测试 | Partial |
| 探测身份与 cwd | 探测进程 UID / GID、环境与 current_dir 符合规定、缺上下文不 spawn 的测试 | Partial |
| attempt 与 single-flight | 相同 / 更小 / 更大 attempt 与 Controller 丢计数后先查询再增号的测试 | Partial |
| 清理责任与容量 | 不可回收的探测保持 cleanup_pending 占名额、重启后仍计入、8 个占满时诊断可见的测试 | Partial |
| 直接 child 与同组后代清理 | 超时后直接子进程与同组后代都已退出的测试，不只断言返回 timeout | Covered |
| 脱组后代清理 | stdio 探测脱离进程组的带标记后代在清理中被杀、Agent 尝试的进程不受影响的测试 | Missing |

### 代表性证据与剩余缺口

Node 侧验收：`Partial`。

- [global_eight_session_two_and_round_robin_survive_slow_cleanup](https://github.com/ora-space/desktop/blob/28da0a4e77ec2257468b15bffd99e166a186b31c/apps/ora-node/src/session/mcp/probes/tests.rs#L73)。
- [one_execution_cannot_take_a_third_slot_even_when_node_has_six_free](https://github.com/ora-space/desktop/blob/28da0a4e77ec2257468b15bffd99e166a186b31c/apps/ora-node/src/session/mcp/probes/tests.rs#L6)。
- [recovered_cleanup_consumes_capacity_before_any_new_values_are_captured](https://github.com/ora-space/desktop/blob/28da0a4e77ec2257468b15bffd99e166a186b31c/apps/ora-node/src/session/mcp/probes/tests.rs#L119)。
- [larger_manual_attempt_joins_active_probe_without_scheduling_another](https://github.com/ora-space/desktop/blob/28da0a4e77ec2257468b15bffd99e166a186b31c/apps/ora-node/src/session/mcp/control/tests.rs#L26)。
- [automatic_confirmation_after_d5_releases_the_probe_lease](https://github.com/ora-space/desktop/blob/28da0a4e77ec2257468b15bffd99e166a186b31c/apps/ora-node/tests/standalone/agent_sessions/mcp_lifecycle/cleanup.rs#L11)。
- [version_update_replaces_agent_and_host_probes_reap_real_descendants](https://github.com/ora-space/desktop/blob/28da0a4e77ec2257468b15bffd99e166a186b31c/apps/ora-node/tests/standalone/agent_sessions/mcp_lifecycle.rs#L220)。

8/2、PATH、同 flight 合并、真实后代清理和自动确认唤醒已有证据；真实 workload UID/GID、16 成员全场景、Controller 丢计数恢复及 N-2/N-3 的竞争路径仍缺。panic 导致整个 Node fail closed，重启恢复持久清理责任；没有把该故障当单 execution 隔离。

尝试环境标记：desktop#802 `198cc415` 的探测清理改为从账本读范围记录标志、在阻塞 worker 上终止并判定，但没有 stdio 探测起脱组后代的测试，也没有断言探测清理不触及 Agent 尝试的进程；该义务仍为 `Missing`。

### 决策依据

D7（身份、attempt、结果模型）、D8，不变量 8；继承 Desktop Host 健康探测的完整握手；[尝试环境标记决策](../../../decisions/node/agent-runtime/20261120-attempt-mark-in-session-stop-evidence.md) D1、D2、D4。

## HTTP Probes Follow No Redirect Use No Proxy And Read At Most One MiB

### 风险

探测随 3xx 把自定义秘密头发往 Location；经 Node 环境中的代理外发秘密；无界读取响应体或 SSE；把第三方
响应文本或 Location 带进结果；HTTP 探测成功被外推为 Agent 可达；共享客户端变化未在 Desktop 回归。

### 前置状态

共享 `ora-utils` HTTP 探测客户端；成员配置带 `X-API-Key` 等自定义头；Node 环境可能设置
`HTTP_PROXY` / `HTTPS_PROXY` / `ALL_PROXY`（含小写）。

### 触发

端点返回同源 / 跨源 307、DELETE 时返回 3xx；响应超过 1 MiB、SSE 不结束；401 / 5xx；`tools/list` 失败；
挂死。

### 必须成立

- 关闭全部跳转（同源、跨源、会话 DELETE），3xx 按所处握手阶段失败，绝不向 Location 发出后续请求。
- 禁用系统代理自动发现，忽略上述代理变量；在 Node 进程的网络身份中执行，不继承管理认证或 cookie jar。
- 响应流在读取中累计限 1 MiB，超限立即终止；stdio 单帧与一次探测累计读取同样限 1 MiB；超限 / 3xx 使用
  既有封闭失败码，不回传 body 或 Location。
- 握手只做 `initialize`、`notifications/initialized`、`tools/list`，不调用工具。
- Desktop 的 HTTP 探测同受影响，有独立回归；Desktop 的选择、刷新与三事实语义不变。

### 禁止结果

- 出现第二次请求或秘密头到达跳转目标；请求经代理发出；读取超过 1 MiB；结果含第三方文本；探测发出
  `tools/call`。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 不跳转 | 本地 fixture 对同源 / 跨源 / DELETE 返回 307 + `X-API-Key`，断言 fixture 只收到一次请求的 `ora-utils` 测试 | Covered |
| 不走代理 | 设置大小写代理变量时请求仍直连 fixture 的测试 | Covered |
| 1 MiB 上限 | 超长 JSON、无结束 SSE 与 stdio 超长帧在读取中被截断并给出封闭失败码的测试 | Covered |
| Desktop 回归 | Desktop 探测在新客户端下的成功与失败矩阵回归 | Partial |

### 代表性证据与剩余缺口

Node 侧验收：`Partial`。

- [crates/utils/src/mcp/tests.rs](https://github.com/ora-space/desktop/blob/28da0a4e77ec2257468b15bffd99e166a186b31c/crates/utils/src/mcp/tests.rs#L1)。
- [crates/utils/src/mcp/tests/bounds.rs](https://github.com/ora-space/desktop/blob/28da0a4e77ec2257468b15bffd99e166a186b31c/crates/utils/src/mcp/tests/bounds.rs#L1)。

共享客户端有本地 HTTP/子进程 fixture 的跳转、代理与流式上限测试；它们证明客户端边界，不替代 Desktop 产品入口的完整成功/失败矩阵，故本用例整体保持 Partial。

### 决策依据

D7 HTTP 段，R8；继承 Desktop Host 健康探测 D2。

## Configured Delivered And Healthy Stay Separate Value-Free Facts

### 风险

已配置、已送达 ACP 与宿主健康被互相推导或合并为“MCP Ready”；pending / blocked 被显示为当前已送达；
健康或 Active 被持久化后跨重启复活；查询进入 Thread、事件序号或 Ack；容量不足被显示为某个 Server 的
Unhealthy；会话快照泄露其他执行。

### 前置状态

生命周期会话处于各 convergence 状态；有探测结果、cleanup_pending 与 Node 级容量汇总；Controller 可能断线
或重启。

### 触发

`GetSessionMcpState`；断线与 Node 实例变化；页面重载；分别注入“仅配置成功”“控制帧写出”“ACP 拒绝”
“ACP 成功但探测失败”。

### 必须成立

- 快照含 `NodeRuntimeIdentity`、execution、单调 `state_version`、desired / active 无值身份、convergence
  （`pending | refreshing | current | blocked | stopping | ended`）、稳定错误和逐成员健康；按 Node 实例 +
  execution + state_version 合并；断线或旧实例时显示未知并重新查询。
- 查询不进入 Thread、execution sequence 或 EventAck；结束的执行只返回 ended。
- 只有 active 匹配当前 desired 且 current 才显示“当前配置已送达”；健康为 Healthy / Unhealthy{code} /
  Unknown{not_probed | context_missing}，探测失败不改 desired / active、不阻塞 prompt。
- 容量不可用与服务失败分开显示；会话快照不暴露其他执行身份；展示只用成员 ID、setting ID 和稳定码。

### 禁止结果

- 任一事实由另一事实推导或合称 Ready；从帧写出或旧缓存恢复 current / Healthy；查询改变事件序号；健康
  失败阻塞 prompt；快照或页面含值、第三方文本或其他执行信息。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 三事实分别注入 | 四种故障注入下快照与页面三层状态互不替代的测试 | Partial |
| 查询不进事件窗口 | 查询前后 execution sequence 与 Ack 窗口不变的测试 | Partial |
| 断线与实例合并 | 旧实例或低 state_version 回复被丢弃、断线后显示未知的 Controller 测试 | Missing |
| 隔离与无值展示 | 会话快照只含本执行与容量汇总、展示无值的测试 | Partial |

### 代表性证据与剩余缺口

Node 侧验收：`Partial`。

- [stale_load_never_admits_and_unavailable_selection_is_accepted_but_blocked](https://github.com/ora-space/desktop/blob/28da0a4e77ec2257468b15bffd99e166a186b31c/apps/ora-node/src/session/mcp/control/tests.rs#L131)。
- [probe_intents_coalesce_and_old_generations_cannot_overlap_or_publish_health](https://github.com/ora-space/desktop/blob/28da0a4e77ec2257468b15bffd99e166a186b31c/apps/ora-node/src/session/mcp/control/tests.rs#L152)。
- [ordered_updates_floor_rejection_and_process_budget_use_the_real_service](https://github.com/ora-space/desktop/blob/28da0a4e77ec2257468b15bffd99e166a186b31c/apps/ora-node/tests/standalone/agent_sessions/mcp_lifecycle.rs#L348)。
- [crates/node-protocol/tests/protocol/session_mcp_lifecycle.rs](https://github.com/ora-space/desktop/blob/28da0a4e77ec2257468b15bffd99e166a186b31c/crates/node-protocol/tests/protocol/session_mcp_lifecycle.rs#L1)。

Node 无值快照和旧代隔离有直接断言；三事实页面、旧 Node 实例/低 state_version 丢弃与断线 Unknown 属 Controller/v2，尚待接入；查询前后完整事件/Ack 窗口不变的独立端到端断言仍缺。

### 决策依据

D3、D9，不变量 9；继承 Desktop ACP setup/refresh 与 Host 健康探测的事实分离。
