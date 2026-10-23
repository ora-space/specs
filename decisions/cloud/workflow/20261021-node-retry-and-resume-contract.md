---
status: approved
---

# 工作流节点重试与恢复契约：继承 desktop 的重试策略，恢复首期在接口上明确不可用

本文是[工作流执行根决策](0-cloud-side-scheduler-owns-workflow-execution.md)（`implemented`）的后续决策，承接
[ora-space/circus#448](https://github.com/ora-space/circus/issues/448) 的 owner 结论（2026-10-21 选 A：继续在
Cloud Go 调度器里移植，首期只实现 retry；resume 只定契约，实现另开决策），把根决策 D1 条件①的硬触发落到
retry 与 resume 两项上。结论：

- **retry**：agent 节点的 `agentConfig.retry` 沿用 desktop 的形状、默认值与上下限；发布门禁从「拒绝任意
  非 null 值」改为「校验对象、字段与范围」。一次可重试的尝试失败后，节点提交 `retrying` 状态并记录
  `attempt` 与 `retryAt`（数据库时钟）；调度器在后续 pass 里到期才启动下一次尝试。等待期间运行保持
  `running`，Workspace 不释放；取消优先于等待；只有不可重试或预算耗尽才按既有路径失败。
- **resume**：语义与接口按 desktop 参考实现写进契约（同一个 WorkflowRun、清除失败节点及其后代、从存活
  状态重排、三种回滚模式），但首期不可用：接口已注册并返回命名错误 `workflow_run_resume_unavailable`
  （501），不改变任何状态，页面不提供入口；实现前另开决策定失败运行的 Workspace 保留窗口与检查点方案。

**状态与影响面**：本文为 `approved`，2026-10-21 经 owner 评审批准（历史快照按默认策略重试、可重试集合只含
`agent_failed` 均经 owner 确认），核心用例见[节点重试与恢复](../../../test-cases/cloud/workflow/retry-and-resume.md)；实现由
[ora-space/circus#460](https://github.com/ora-space/circus/issues/460) 承接。涉及 cloud 的调度器、发布门禁、
`workflow_runs.node_states` 投影、`execution_work` / `node_executions` / `thread_commands` 的持久化形状与
按尝试唯一索引，以及 OpenAPI 的 `nodeStates` 枚举与 resume 接口。**兼容边界**：新增列一律带默认值 `1`，存量
行不回填；`nodeStates` 新增 `retrying` 状态与 `attempt` / `retryAt` 字段，旧读取方忽略未知字段即可；
Controller 线上契约不变（attempt 是 Cloud 内部路由字段，如 `workflow_node_id`）。**行为变化**：没有声明
`retry` 的历史快照在实现上线后按默认策略重试（D1、D8）——这是与 desktop 对齐的有意结果，不是回归。
**非目标**：不引入 loop、iteration、snapshot switch、subflow；不建检查点；不新增运行状态；不改变失败、取消
与资源释放的既有结算（除本文新增的重试分支与取消对等待中重试的处理外）。

## 继承与修改

| 前序约定 | 本文处理 |
|---|---|
| 根决策 D1 条件①：服务端引入 loop / iteration / retry / resume 前先决策「继续 Go 移植」还是「抽取 Rust 核心」 | **落实**：owner 选「继续 Go 移植」。本文只覆盖 retry 与 resume；loop、iteration、snapshot switch、subflow 仍按条件①单独触发 |
| 根决策 D2：`retry` 是发布门禁拒绝的 agent 配置字段 | **修改**：从「拒绝任意非 null 值」改为「校验对象、字段与范围」，错误码按节点与字段定位 |
| 根决策 D4：首期节点失败即运行失败 | **修改**：可重试的失败在预算内先进入 `retrying`；只有不可重试或预算耗尽才失败运行。节点与运行的其他生命周期规则不变 |
| 根决策 D4：同一 run 的 agent 节点首期串行 | **继承并扩展**：等待中的重试同样占用这个串行位，期间不派发同一 run 的任何其他节点 |
| 根决策 D4 / 不变量 5：节点的终态只来自会话执行结果 | **继承**：每次尝试的结算仍只读该尝试的 `node_executions.result`；等待与派发都不推进节点，重试不伪造成功 |
| 根决策不变量 3：节点结果提交是带状态的比较交换，迟到或重复回执是空操作 | **扩展**：证据按尝试隔离，只有当前尝试的执行能结算节点；被取代尝试的迟到结果不改变投影 |
| 根决策 D6 / D7：取消是请求，终态才释放 Workspace | **继承并补充**：取消优先于等待中的重试；等待不是终态，不释放 Workspace |
| [会话结束后续决策](20261019-workflow-session-end-after-initial-turn.md) D1–D5：成功回合后 Cloud 放出按节点唯一的 `end_session`，确认窗口有界 | **修改**：`end_session` 与唯一约束按尝试限定；单个尝试内的结束路径不变。会话自行以失败结束的尝试不会产生结束命令 |
| 会话结束后续决策不变量 2：每个（运行，节点）至多一条 `end_session` | **修改**：每个（运行，节点，尝试）至多一条 |
| desktop `engine/retry.rs`：策略形状、默认值、上下限、退避公式、可自动重试的失败集合 | **继承**：形状与数值逐项一致；失败集合按 Cloud 的会话原因语义折叠（D2） |
| desktop `resume_from_failure`：同一运行内清除失败节点及后代、从存活状态重排；`keep` / `node_files` / `checkpoint` 三种回滚 | **继承语义，首期不实现**（D7）；不继承「在更新的已发布快照上恢复」——Cloud 的运行身份在创建时冻结（根决策不变量 2） |
| [IssueRun 根决策 D4](../issue-run/0-agent-run-in-disposable-isolated-workspace.md)的会话原因映射：`agent_failed` / `interrupted` 为失败，`cancelled` 为取消，自然结束为成功 | **继承**：映射不变；重试只依据原因的闭集判断，不改变映射 |

## 决策

### D1：配置沿用 desktop 的 `agentConfig.retry`，发布门禁改为校验

`agentConfig.retry` 的形状与 desktop `parse_retry_policy` 一致：

```json
{"enabled": true, "maxRetries": 2, "initialDelaySeconds": 10}
```

- **缺省（字段缺失或 `null`）等于默认策略**：`enabled=true`、`maxRetries=2`、`initialDelaySeconds=10`，
  即每个 agent 节点默认最多重试两次。这与 desktop `AgentRetryPolicy::default()` 完全相同。
- **出现时必须是对象，且三个字段全部必填**：`enabled` 是布尔；`maxRetries` 是 `0..=5` 的整数；
  `initialDelaySeconds` 是 `0..=300` 的整数。缺字段或类型/范围不符都拒绝，不做「缺哪个用默认」的部分
  回退——编辑器的一个 bug 不该让作者执行一份他没看见过的策略。未知键与图文档的其他位置一样忽略。
- **等待时长**：第 `n` 次重试（`n` 从 1 起）等待 `initialDelaySeconds × 2^(n-1)`，单次上限 600 秒；与
  desktop 的 `wait_seconds` 一致。
- **预算**：每个节点每次运行一个预算。首次尝试的序号是 1，`maxRetries` 是额外尝试数，总尝试数至多
  `maxRetries + 1`。`maxRetries=0` 或 `enabled=false` 都表示不重试。预算随节点状态提交，调度器重启、
  换副本、重放都不重置。
- **发布门禁**：把 `retry` 从 `workflowUnsupportedAgentConfigFields` 移除，改为逐项校验，错误码
  `agent_retry_not_an_object`、`agent_retry_enabled_invalid`、`agent_retry_max_retries_invalid`、
  `agent_retry_initial_delay_invalid`，每个都带 `nodeId` 与 `field`（`agentConfig.retry...`），
  沿用既有门禁的三要素风格。保存草稿仍然宽松（继承 MCP 决策 D4 / 根决策 D2）。
- **历史快照**：已发布的冻结快照不重新过门禁。没有 `retry` 的历史快照在实现上线后按缺省策略执行，
  见 D8 的行为变化说明。

### D2：可重试集合只含会话自身报告的 `agent_failed`

Cloud 的会话结果只有五个结束原因：`agent_failed`、`interrupted`、`cancelled`、`user_ended`、
`idle_timeout`。本文的可重试集合是**闭集，只含 `agent_failed`**：

| 原因 | 是否重试 | 理由 |
|---|---|---|
| `agent_failed` | 是 | desktop 自动重试的六类失败（Session、SessionEndedWithoutStopReason、SessionBindingRejected、StructuredOutput、AgentRefusal、UnknownStopReason）在 Cloud 的会话结果里没有细分，都落在 `agent_failed`；它就是「Agent 这一轮没给出可用结果」的 Cloud 语义 |
| `interrupted` | 否 | 对应 desktop 明确不自动重试的 `InterruptedByRestart`：会话被宿主/Node 中断，不是 Agent 的回答；自动重试会掩盖崩溃循环 |
| `cancelled` | 否 | 取消优先（D5），且取消是用户决定 |
| `user_ended`、`idle_timeout` | 否（本来就是成功） | 映射为成功，不进入失败路径 |
| Cloud 自己判定的失败：`session_start_deadline_exceeded`、`session_turn_deadline_exceeded`、`session_end_deadline_exceeded`、`node_unavailable`、`workspace_unavailable` | 否 | 每个都已经是 Cloud 的有界等待或宿主/资源判定，再乘上重试次数只是延长资源占用；desktop 的同类基础设施失败（Repository、BaselinePersist 等）同样不自动重试 |
| 配置与永久原因：`workflow_run_project_required`、`workflow_executor_unconfigured`、`agent_plugin_version_unfrozen` | 否 | 重试不会改变配置或冻结身份，只会把失败推迟 |

**判定读的是原因字符串，不是映射后的状态**：`applyWorkflowAgentTerminal` 把未知原因默认映射为成功；重试
决策必须在映射之前按 `result.reason` 的闭集判断，不能写成「映射结果是 failed 就重试」。将来新增原因时，
本表随之更新，这是 ADR 变更，不是实现细节。

### D3：等待中的尝试是已提交的 `retrying` 节点状态，由 pass 到期唤醒

一次 `agent_failed` 的尝试结束后，如果策略启用、`attempt <= maxRetries` 且没有取消请求，节点不提交
`failed`，而是提交 `retrying`：

```json
{"status": "retrying", "attempt": 1, "retryAt": "2026-10-21T12:00:10Z",
 "error": "agent_failed", "output": {"reason": "agent_failed"}}
```

- `attempt` 是刚失败的那次尝试的序号（1 起）；`retryAt` 是下一次尝试的最早开始时刻，用数据库时钟
  `clock_timestamp() + make_interval(secs => wait)` 计算；`error` / `output` 保留这次失败的原因；
  `startedAt` 保留，`finishedAt` 清空。字段随 `workflow_runs.node_states` 的既有版本 CAS 一起提交，
  因此跨重启、跨副本、重放读到的是同一份状态和同一个到期时刻——这是根决策条件②「无状态调度」的直接
  要求，也与会话结束决策用命令 `created_at` 计窗口是同一个模式。
- **没有进程内定时器，也不依赖租约**：调度器每轮 pass 扫描 `running` 的运行；`retryAt <=
  clock_timestamp()` 时启动下一次尝试。这与 `operations.retry_at` 的到期重试同构。到期后启动尝试、
  提交 `running`、放出执行工作项，再返回——和其它 agent 节点的启动路径一致。
- **启动下一次尝试**：复用 `startWorkflowAgentSession` 的既有路径（同一个运行 Workspace、同一份冻结
  快照与槽位绑定、同一个 Workspace target）。节点状态从 `retrying` 回到 `running`，`attempt` 加一，
  `retryAt` / `error` / `finishedAt` 清空，`startedAt` 记为这次尝试的开始。
- **等待期间运行保持 `running`**：运行不进入终态，Workspace 不释放（根决策 D7 的释放只在终态），不产生
  新的执行工作项。运行视图从 `nodeStates` 读到 `retrying`、`attempt` 与 `retryAt`。
- **串行位**：等待中的节点仍占用「同一 run 一个 agent 节点」的串行位（根决策 D4）。存在 `retrying`
  节点时，本轮 pass 不派发同一 run 的任何其他节点。
- **必须显式挡住终态结算**：现有 pass 的结尾是「没有失败节点就成功结算」。`retrying` 既不是 `failed`
  也不是 `running`，如果不处理，运行会被误结算为 `succeeded`（根决策不变量 5 的反面）。因此 pass 在
  取消分支之后、会话结算之前先检查 `retrying`：未到期就原样返回，到期就启动下一次尝试并返回，绝不落到
  结尾的终态结算。
- **下游等待**：投影把未知状态当 `pending`（既有 `edgeState` 的 default 分支），`retrying` 节点的下游
  因此保持未就绪，不会提前执行。这一点是实现必须保持的行为，而不是碰巧成立。
- **`retryAt` 的写入时机**：在观察到失败结果并决定重试的同一事务里写。若进程在「会话结果已提交、重试
  状态未提交」之间崩溃，下一次 pass 会重新判定并重新计算 `retryAt`——等待从下一次观察开始，只会变长，
  不会把节点永久卡住，也不会把失败误记为成功。并发 pass 由运行级排他事务与版本 CAS 串行化，输的一方
  什么也不写。

### D4：每次尝试有独立的会话证据，按节点唯一的约束改为按尝试唯一

重试意味着同一个（运行，节点）在整个运行生命周期里有多条 `agent_session` 执行行。既有的按节点唯一
约束必须改为按尝试唯一，否则第二次尝试无法登记：

- **新增 `workflow_node_attempt integer NOT NULL DEFAULT 1`**：`execution_work`（图节点工作项）、
  `node_executions`（执行证据）、`thread_commands`（结束命令）各一列。存量行默认 `1`，不回填；IssueRun
  的行保持默认 `1`，语义不变。
- **`one_session_per_workflow_node`** 从 `(operation_id, workflow_node_id)` 改为
  `(operation_id, workflow_node_id, workflow_node_attempt)`。
- **`one_workflow_session_end`** 从 `(workflow_run_id, workflow_node_id)` 改为
  `(workflow_run_id, workflow_node_id, workflow_node_attempt)`。第 1 次尝试的结束命令不能被投递给第 2 次
  尝试的会话；`claimThreadCommands` 的 join 与 `recordThreadCommandDelivered` 的归属校验都要带上
  attempt。
- **所有按节点读取证据的查询都必须带当前尝试**：会话终态证据、`end_session` 的查找与放出、取消路径的
  证据读取。只按 `(operation_id, workflow_node_id)` 读取会在第 2 次尝试在途时读到第 1 次的失败结果。
- **只有当前尝试的执行能结算节点**：被取代尝试的迟到或重复结果只写在自己的执行行上，对投影是空操作
  （根决策不变量 3 的扩展）。尝试是串行的，第 1 次的结果就是触发第 2 次尝试的那份证据，不存在两个
  尝试同时在途。
- **不新增跨仓库字段**：attempt 与 `workflow_node_id` 一样是 Cloud 内部路由，claim 仍然只把运行 id
  投影给 Controller；执行工作项、claim、回执与代校验的线上形状不变（根决策 D8）。Controller 登记执行时，
  Cloud 从被登记的执行工作项复制 `workflow_node_attempt` 到 `node_executions`，attempt 不来自线上请求。
- **既有约束不需要改**：`one_pending_run_execution`（每个运行至多一条在途 agent_session）仍然成立，
  因为尝试串行；`one_unregistered_workflow_execution_work`（每个运行至多一条未登记工作项）也仍然成立，
  等待期间没有工作项，到期才放出一条。

### D5：取消优先于等待中的重试

- 取消请求一旦记录，下一轮 pass 的取消分支先于重试分支执行：存在 `retrying` 节点时，把该节点提交为
  `cancelled`（原因 `cancelled`，清空 `retryAt`，写 `finishedAt`），运行结算 `cancelled` 并释放
  Workspace。**不启动新尝试，即使 `retryAt` 已经到期**。
- 取消请求发生在节点 `running` 期间时，走既有的结束会话 / 确认窗口路径，不变。
- 取消不等待重试到期，也不重置或缩短等待；等待中的重试不会让取消多等一个退避周期。
- 并发由运行级排他事务与版本 CAS 串行化：先提交的一方决定——若下一次尝试先提交，节点是 `running`，
  取消走既有路径；若取消先提交，新尝试不会启动。

### D6：预算耗尽或不可重试时按最后一次失败的证据结算

策略未启用、原因不在闭集、或 `attempt > maxRetries` 时，节点以最后一次尝试的原因提交 `failed`，运行以同一
原因失败，Workspace 按既有路径释放。不新增失败原因或运行状态；节点状态保留 `attempt` 作为历史，清空
`retryAt`。结算后不会再有新的尝试。

### D7：resume 的语义与接口，首期明确不可用

**语义（契约，首期不实现）**：resume 是对一个终态 `failed` 运行的用户发起恢复，继承 desktop
`resume_from_failure` 的语义：

- 同一个 WorkflowRun，同一份冻结快照、输入与逐槽位绑定（根决策不变量 2 不允许换运行身份）；不创建新运行，
  「重新跑一遍」仍是既有的一次新运行。
- 清除失败节点的已提交状态及其全部后代状态，保留成功的兄弟分支；运行回到 `running`，清空 `error` 与
  `finishedAt`；下一轮 pass 从存活状态重新投影并派发。
- 三种回滚模式，请求体 `{"mode": "keep" | "node_files" | "checkpoint"}`，缺省 `keep`：`keep` 不改
  Workspace；`node_files` 只还原失败节点改过的路径（需要 `file_changes`）；`checkpoint` 把整个
  Workspace 还原到恢复单元的检查点（需要每节点 Git checkpoint）。后两种在 Cloud 首期都不可用——Git
  Server 不接受 `refs/ora/checkpoints/...`，Cloud 也没有每节点检查点与 `file_changes`。
- 恢复与取消一样是**请求 + 调度器 pass 反应**，接口本身不直接改节点状态：这样节点状态的写入者仍只有
  调度器一个，请求不会与 pass 竞争同一份状态。恢复后节点从 attempt 1 重新计数（与 desktop「自上次
  resume 起」一致）。
- 没有失败节点可清的运行（例如图校验失败、一个节点都没跑）不可恢复。

**首期不可用**：接口已注册并进入 OpenAPI，但实现只做三件事：校验请求、按取消的同一规则做授权、返回
命名错误。

- `POST /api/v1/tenants/:tid/workflows/:wfid/runs/:rid/resume`。
- 授权与取消相同（`mayControlWorkflowRun`）：带冻结执行器绑定的运行只有发起者与租户管理员可以请求；
  其他成员 403 `workflow_run_resume_forbidden`。找不到或不属于该租户 / 工作流的运行 404 `not_found`。
- 形式合法的请求一律 501 `workflow_run_resume_unavailable`，不写任何状态：不清节点、不改运行状态、
  不碰 Workspace、不写幂等记录。请求体形状先按最终契约校验（未知字段 400 `unknown_field`、非法 `mode`
  400 `invalid_field_value`），因此客户端现在就能按最终形状接入；`mode` 不影响不可用的结论。
- **为什么是显式错误而不是不注册接口**：owner 的决定是「在接口上明确拒绝并标注首期不可用（与
  `awaiting_input` 相同）」。没有接口时，客户端无法区分「服务端没有这个能力」和「路径写错」；注册并
  返回稳定错误码，契约与页面都能诚实地表达「首期不可用」。
- **页面不提供入口**：运行页不渲染恢复按钮，前端不调用该接口；恢复入口随实现一起上线。
- **为什么 `keep` 也不能先实现**：失败运行进入终态时 Workspace 已被声明删除（根决策 D7）。恢复时没有
  Workspace 可用，必须先改「终态即释放」的生命周期并定义保留窗口与放弃路径——这正是 owner 把 resume
  实现留给后续决策的原因。

### D8：可观察性与兼容性

- **节点状态**：`nodeStates` 新增 `retrying`；`retrying` 与终态节点可带 `attempt`，`retrying` 还带
  `retryAt`。OpenAPI 的节点状态枚举与描述同步更新；运行视图按 `retrying` 渲染「等待第 n 次重试」与
  到期时刻。前端改动属于实现运行（#460），不属于本文。
- **运行状态**：不新增。等待期间运行仍是 `running`，没有新的运行级字段；运行的 `error` 在等待期间为空，
  只在最终失败时写最后一次尝试的原因。
- **历史快照的行为变化**：实现上线后，没有 `retry` 的历史快照按缺省策略（启用、2 次、10 秒起）执行，
  即同一个冻结图在失败时会重试而不再立即失败。这与 desktop 的参考语义一致，也与条件求值收紧的先例
  一致（历史快照不重新过门禁，升级后按新语义执行）。如果 owner 要求历史快照保持不重试，需要按快照
  发布时间解释默认值——那会让行为依赖隐藏日期并与 desktop 分叉，本文不采用。
- **不做的事**：不新增事件流、不新增运行状态、不改变 Controller 线上契约、不改变取消与释放的既有语义。

## 不变量

1. 一次尝试的终态只来自该尝试的 `node_executions.result`；等待、派发与重试都不推进节点，重试不伪造成功。
2. 一个节点任意时刻至多一个在途尝试；重试只在上一尝试的会话终态提交之后发生。
3. 等待中的重试不是终态：运行保持 `running`，Workspace 不释放；运行只在终态释放。
4. 只有原因在闭集内、预算未耗尽且没有取消请求时，节点才进入 `retrying`；否则按既有失败结算。
5. `retrying` 的到期唤醒只读已提交的 `retryAt`（数据库时钟），重启、多副本与重放读到同一时刻。
6. 取消优先：记录了取消请求的运行不再启动任何新尝试，等待中的节点直接结算 `cancelled`。
7. 只有当前尝试的执行能结算节点；被取代尝试的迟到结果不改变投影。
8. resume 首期不改变任何状态，只返回命名错误；恢复的节点状态写入者仍只有调度器。
9. Controller 线上契约不变：attempt 是 Cloud 内部路由字段。

## 为什么不是这些替代方案

| 方案 | 否决理由 |
|---|---|
| 等待中的尝试保持 `running`（desktop 用 `Running` 且无 `started_at` 的行表示等待） | Cloud 的 `running` 会进入会话证据路径：`workflowRunningAgentNode` 会去找执行行，找不到就按启动超时把节点判失败。要区分「等待」与「在途」，得在 `running` 上再挂一个内部标记，等于把状态藏进字段；显式的 `retrying` 让取消、结算、投影三条路径都必须正面处理它 |
| 进程内定时器或延迟队列（desktop 的 `retry_scheduler`） | Cloud 是无状态调度 + 多副本（根决策条件②）；进程内定时器在崩溃或换副本后丢失。写库的 `retryAt` 由任意 pass 重算，与 `operations.retry_at` 同构 |
| 把 `interrupted`、Cloud 的 deadline、`node_unavailable` 也纳入重试 | desktop 明确不对 `InterruptedByRestart` 自动重试；Cloud 的 deadline 是已经发生过的有界等待，再重试等于把上界乘上重试次数；`node_unavailable` 是执行宿主消失，不是 Agent 的回答。首期只复刻 desktop 真正自动重试的那一类 |
| 每次重试创建新的 WorkflowRun | 会让一次用户运行的节点历史、输入与 Workspace 归属分裂成多个运行实体；根决策 D3 已把 WorkflowRun 定为执行聚合 |
| 用 `workflow_runs.error` 或新列记录重试计数，不改节点状态 | 调度器只从 `node_states` 重算就绪与结算；重试计数放在别处会让「节点是否还会再跑」不再由投影回答，恢复与多副本读取也多一处状态源 |
| 首期直接实现 resume 的 `keep` 模式 | 失败运行进入终态时 Workspace 已被声明删除，恢复时无 Workspace 可用；必须先改 Workspace 生命周期，这正是 owner 把 resume 实现留给后续决策的原因 |
| resume 不注册接口，只在前端隐藏入口 | 客户端无法区分「没有这个能力」与「路径写错」，契约也无法登记稳定错误码；owner 要求在接口上明确拒绝（与 `awaiting_input` 相同） |
| 让历史快照保持「不重试」 | 需要按快照发布时间解释默认值，行为依赖隐藏日期，且与 desktop 的参考语义分叉；条件求值收紧已有先例，历史快照不重新过门禁 |

## 本决策未解决的问题

- **resume 的 Workspace 保留窗口与放弃路径**：终态即释放（根决策 D7）与可恢复互斥。保留多久、窗口内
  资源成本、窗口结束仍无恢复时如何释放，都要在实现 resume 前另开决策。
- **检查点与回滚基础设施**：`refs/ora/checkpoints/...` 的 Git Server 白名单、Node 侧 checkpoint 写入、
  `file_changes` 捕获、回滚前 Workspace 快照，以及非快进回滚的正确性，跨 cloud / desktop / node 三个仓库。
- **resume 是否允许在更新的已发布快照上继续**：Cloud 根决策不变量 2 冻结运行身份，本文不继承 desktop 的
  换快照能力；将来若要支持，先改不变量 2。
- **重试与运行级超时的交互**：等待期间运行保持 `running`，若将来给运行加总超时，重试等待是否计入、上限
  如何计算，由那份决策定义；本文不预设。

## 落地顺序

- **阶段 1（ADR 评审）**：本 ADR 以 `proposed` 提交 specs PR，只含正文，不碰
  `test-cases/`（`proposed` 阶段不编写核心测试用例）。完成条件是 PR 已开、owner 开始评审，且改动文件
  只有这一份 ADR。
- **阶段 2（批准后补用例并合并）**：owner 批准后，在同一 PR 补
  `test-cases/cloud/workflow/retry-and-resume.md`，把状态改为 `approved` 再合并。核心用例至少覆盖：
  「可重试的失败被有界等待取代」「等待中的重试按已提交的到期时刻唤醒」「只有当前尝试能结算节点」
  「预算耗尽使运行失败」「取消优先于等待」「等待期间运行保留 Workspace」「resume 首期明确不可用」。
  完成条件是每条不变量都有对应的用例或明确的 `Missing`，`decisions` 与 `test-cases` 的目录镜像一致，
  PR 由 owner 合并。
- **阶段 3（Cloud 实现，运行 ora-space/circus#460）**：新增 attempt 列与按尝试唯一索引的迁移；发布门禁
  校验；`retrying` 的提交、唤醒、串行与终态结算；取消对等待中重试的结算；会话证据与结束命令的尝试
  作用域；resume 的 501 接口与 OpenAPI。完成条件是真实 PostgreSQL 的调度器测试覆盖上述路径、OpenAPI
  与前端生成物已同步、`task check` 全绿。
- **阶段 4（状态同步）**：实现合并后由落地运行把本文改为 `implemented`，核对偏离、精简正文，并按最终
  实现登记核心用例的证据状态。完成条件是正文与实现一致、证据表如实反映 `Covered` / `Partial` /
  `Missing`。
