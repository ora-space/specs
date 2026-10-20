---
status: implemented
---

# 工作流会话在初始回合结束后由 Cloud 调度器结束：复用 EndSession 中继，有界等待后以命名原因失败

本文是[工作流执行根决策](0-cloud-side-scheduler-owns-workflow-execution.md)（`approved`）的后续决策，只固定
「单回合工作流会话何时结束、由谁结束」这一接缝。结论：WorkflowRun 的 agent 会话结束由 Cloud 调度器决定——
调度器在重算时读取该会话执行最近一条事件回执，成功的 `turnEnded`（记录字段 `stop_reason` 不是 `cancelled`）即说明初始
回合已结束；调度器按既有控制面的「判别列 + 恰好一个归属」模式把 `thread_commands` 扩展到 WorkflowRun，
放出既有 Controller `EndSession` 中继（`reason = idle_timeout`），Node 结束会话并上报终态，节点仍只按
`node_executions.result` 推进。结束命令放出后超过确认窗口仍无会话终态时，节点以
`session_end_deadline_exceeded` 失败，运行进入终态并释放 Workspace：已请求结束的会话不再可能让运行永久
`running`（从未走到结束请求的路径见「本决策未解决的问题」）。

**状态与影响面**：本文经 owner 评审批准（2026-10-20），已由 cloud#96 实现并合并（merge commit
`644a65e2`），并在真实栈上验收通过（运行 ora-space/circus#293，成功运行 `847bca1a`），状态为
`implemented`；核心用例见
[工作流会话结束核心用例](../../../test-cases/cloud/workflow/session-end-after-initial-turn.md)。涉及 cloud 的
调度器与 `thread_commands` 持久化形状，以及 specs 的本文。**兼容边界**：`thread_commands` 从只接受
IssueRun 扩展为两个归属列恰有其一；存量行全部带 `run_id`，不回填，IssueRun 的 Thread 路径不变。Controller
的线上契约（工作项种类、claim、回执、代校验）不变（根决策 D8）。**非目标**：不新增跨仓库协议字段、不修改
图 schema 与发布门禁、不改变运行身份的冻结项、不重新定义失败与取消的既有结算。本决策不承担数据迁移之外
的迁移义务，也没有需要重建的历史状态。

## 继承与修改

| 前序约定 | 本文处理 |
|---|---|
| 根决策 D1：执行归属归 Cloud 侧调度器，agent 节点沿用既有 AgentSession 执行路径 | 继承，不变：结束命令仍由 Cloud 决定，Node 只执行命令 |
| 根决策 D2/D4：首期只执行 `start` / `agent` / `condition` / `output`，发布门禁拒绝 `interactive: true`；同一 run 的 agent 节点串行 | 继承，并成为前提：工作流里的 agent 会话是**单回合**的，运行不会再产生用户轮次 |
| 根决策 D4：节点终态以会话执行结果为证据；崩溃恢复只按已提交结果重算 | 继承，不变：本文只补上「请求结束」的写入者，节点终态的证据来源不变 |
| 根决策 D7：首期只保证资源释放，成果交付延后 | 继承，并扩展：结束请求本身也有界等待，超时按命名原因失败，仍然走到资源释放 |
| 根决策 D8：不新增 Controller 契约 | 继承，不变：复用既有 `EndSession` 命令中继，claim 与回执字段不变 |
| [Thread 根决策 D4](../thread/0-durable-agent-thread-with-user-turns.md)：Cloud 按数据库时间判断空闲；成功 `TurnEnded`（非 `cancelled`）且没有排队轮次是空闲证据；`EndSession{user_ended \| idle_timeout \| cancelled}` | 继承判据，不继承机制：WorkflowRun 没有 Thread、`thread_state` 与 `idle_since`，本文从同一份回执得出「初始回合已结束」，跳过为多轮对话设计的空闲窗口，直接请求结束 |
| [IssueRun 根决策 D6](../issue-run/0-agent-run-in-disposable-isolated-workspace.md)：取消是请求加有界终态 | 继承语义，不认领实现：Cloud 目前没有 WorkflowRun 的取消写入者，取消由根决策 D6 的后续实现承接；D5 说明每节点唯一的 `end_session` 约束对它的影响 |
| [控制面决策](../controller-integration/20260928-agent-run-executions-thread-and-upload-grants.md)：claim、回执与代校验由 Cloud 表承载归属 | 继承，不变：新增的归属只存在于 Cloud 自己的表，Controller 看到的仍是 run id 与 execution id |

## 决策

### D1：触发是调度器重算读取成功 `turnEnded` 回执

工作流 agent 节点是单回合的：发布门禁拒绝 `interactive: true`（`interactive: false` 已放行），Node 完成初始回合后会等待下一条命令，而运行
不可能再产生用户轮次。因此初始回合的 `turnEnded` 回执就是「会话可以结束了」的证据，不需要再等一个
空闲窗口。

判据是**该会话执行在 `node_event_receipts` 中最近一条回执**：记录类型为 `turnEnded`，且 `stop_reason`
不是 `cancelled`。只读最近一条而不是扫描任意历史回合——回合之后又出现其他记录时，对话并没有停在回合
边界；`cancelled` 的回合也不构成空闲证据，取消由会话终态结算。这与 Thread D4 的空闲判据同源，Cloud
仍然不解释记录的业务字段，只使用接管校验已经依赖的类型判别与 `stop_reason`。字段名按 `ora-history`
实际序列化的形状书写：serde 的容器级 `rename_all = "camelCase"` 只改枚举变体名（`turnEnded`），
变体字段保持 snake_case（`stop_reason`）。

**非 `cancelled` 的其他停止原因也按回合结束处理，这是有意的**：`max_tokens`、`refusal` 等同样意味着
本轮不会再产出更多内容，单回合会话不可能再有下一轮。调度器在这里只判断「本轮是否已经结束、还会不会
有用户轮次」，不判断输出是否充分；节点是否成功仍由会话终态按既有映射决定。若将来引入节点输出契约
（例如 `outputContract`）需要把截断或拒绝判为节点失败，那是节点输出语义的另一份决策，不改变本文的
触发判据。

触发位置是**调度器重算**（`AdvanceWorkflowRunsOnce` 的每轮 pass），不是接管回执的 hook。调度器本来
就只从已提交状态重算；进程在放出命令前后的任何时刻崩溃，下一轮重算都会重新读到同一份回执并重放同一
请求（D3 的唯一性使它成为幂等重放，而不是第二条命令）。把触发放进接管路径并不会省掉这一步：崩溃窗口
内的回执最终仍要由重算补发，而两条写入路径还会竞争同一节点唯一的结束命令。

### D2：结束原因复用 `idle_timeout`

`EndSession` 的 body 沿用既有的 `{"reason": ...}`，取 `idle_timeout`；不新增 `turn_completed` 或别的
枚举。Node 对既有三个原因的处理完全相同——停止 Agent、完成会话记录、以终态结束会话——差异只影响
IssueRun 的 `status` 映射；工作流节点的结算只看会话终态，不看原因。新增枚举会让 cloud proto、desktop
controller-proto、`node-protocol` 与 Node 驱动按序发布，而它在这里没有带来任何可观察的行为差异。

### D3：命令归属采用判别列 + 恰好一个归属，并按节点限定

`thread_commands` 原本是 `run_id NOT NULL REFERENCES issue_runs`，因为只有 IssueRun 有 Thread、也只有
Thread 有会话结束的写入者。本文按 `execution_work`（`0039`）、`node_executions`（`0041`）、
`model_request_attempts`（`0044`）已有的模式扩展它：

- `run_id` 改为可空，新增 `workflow_run_id uuid REFERENCES workflow_runs(id)` 与 `workflow_node_id text`；
- `thread_command_owner_exclusive`：`run_id` 与 `workflow_run_id` 恰有其一；
- `thread_command_workflow_scope`：`workflow_run_id` 是否存在与 `workflow_node_id` 是否存在一致；
- `one_workflow_session_end` 部分唯一索引：每个（工作流运行，节点）至多一条 `end_session` 命令。

**为什么必须按节点限定**：一个 WorkflowRun 一生会依次执行多个 agent 节点，每个节点有自己的会话执行
（根决策 D4 串行）。命令若不限定节点，claim 可能把某个节点的结束请求投给另一个节点的会话。唯一索引
同时让调度器的重复重算成为重放：同一节点第二次重算读到的还是同一条命令，确认窗口也不会被重置。

**为什么沿用判别列而不是为 WorkflowRun 建一条 IssueRun**：虚假的 IssueRun 会污染 Issue 时间线、Thread
语义与统计读取；根决策 D3 已经明确 WorkflowRun 才是执行聚合，IssueRun 只是调用记录。

对外契约不变：claim 把两个归属列**投影成同一个 `run_id`** 返回（Controller 按它匹配自己的 control
binding），投递回执按实际归属校验执行所属，并在工作流命令上校验节点范围。Controller 的字段与语义与
IssueRun 命令完全一致。

### D4：结束请求有界等待，超时按命名原因失败并释放资源

结束命令放出后，以命令的 `created_at`（数据库时钟）为起点计算确认窗口。窗口复用 Cloud 既有的
`AgentEndConfirmTimeout`（默认 5 分钟）：IssueRun 的结束确认把最早一条 `end_session` 的 `created_at`
加同一配置冻结成运行上的截止时间且从不续期（`freezeAgentEnding`），工作流路径在每轮重算时用同一条
命令的 `created_at` 比较同一窗口，两者对同一条命令给出相同的边界。超过窗口仍没有会话终态时：

- 该节点以 `session_end_deadline_exceeded` 失败，节点输出记录同名原因；
- 运行以 `session end deadline exceeded` 进入终态；
- 运行 Workspace 按根决策 D7 的既有收尾声明删除。

**为什么必须有界**：没有它，Node 始终不领取命令、或会话始终不上报终态时，运行会永久 `running`——正是
本文要消除的缺陷。**为什么从命令创建时间起算**：调度器每 10 秒重算一次，多副本与崩溃重试必须读到
同一个窗口；命令 `created_at` 由数据库写入，重放同一条命令不会重置计时。窗口内不重复放出命令，也不
提前判失败；超时判断同样在数据库时钟上完成，进程时钟偏差不能缩短或延长窗口。

超时失败发生在「请求结束且确认窗口已过」之后，此时会话终态仍然未知。它不把未知当作成功（根决策
不变量 5），而是显式失败并释放资源；会话执行行与其回执保留，作为事后诊断证据。

### D5：节点终态仍只来自会话结果，三条路径的结算不变

本文只增加「请求结束」的写入者，不改变各路径的结算：

| 路径 | 结算 |
|---|---|
| 成功 | Node 结束会话并上报正常结果；节点 succeeded，运行继续调度后继节点 |
| 失败 | 会话以失败结束（Agent 或模型错误）；Node 自行以 `agent_failed` 结束会话，节点 failed，运行失败，Workspace 释放 |
| 取消 | 本文不新增取消路径：Cloud 目前没有 WorkflowRun 的取消写入者，取消由根决策 D6 的后续实现承接；`cancelled` 的 `turnEnded` 不触发本文的结束请求，会话自行上报的取消仍按既有会话终态结算 |
| 超时 | D4 的命名失败是唯一由 Cloud 直接判定的路径；它不写入成功，也不伪造会话结果 |

资源释放顺序不变：节点终态先提交，运行进入终态后才声明删除 Workspace；结束命令、确认窗口与会话终态
都发生在同一个运行的短事务之外，不会把外部等待带进事务。

**每节点唯一的 `end_session` 约束会影响将来的取消设计**：`one_workflow_session_end` 限定每个
（工作流运行，节点）至多一条 `end_session`，后续取消不能直接再放出第二条 `end_session{cancelled}`。
D6 的后续实现有两条不改变本文归属与触发判据的路径：会话尚未结束时复用已有命令并把结束原因升级为
`cancelled`，或者放宽/替换该唯一索引（例如把 `reason` 纳入唯一键）。本文不预先决定取其哪一条。

## 不变量

1. 成功回合之后的会话结束只来自 Cloud 放出的 `end_session`；Cloud 不放命令，Node 不因成功回合自行
   结束。Node 在回合失败时仍按既有路径自行以 `agent_failed` 结束会话。
2. 每个（工作流运行，节点）至多一条 `end_session` 命令；重复重算只重放，不新增，也不重置确认窗口。
3. 会话终态（`node_executions.result`）仍是节点终态的唯一正面证据；结束请求本身不推进节点。
4. `end_session` 放出之后，确认窗口内仍没有会话终态时，节点以命名原因失败并释放资源；已请求结束的
   会话不会让运行永久 `running`。
5. Controller 线上契约不变：claim、回执与代校验字段和语义与 IssueRun 命令一致。
6. `cancelled` 的 `turnEnded` 不触发结束请求；取消由根决策 D6 的后续实现承接，本文不定义取消写入者。
7. 存量 `thread_commands` 行全部带 `run_id`，不需要回填；IssueRun 的 Thread 行为不变。

## 为什么不是这些替代方案

| 方案 | 否决理由 |
| --- | --- |
| Cloud 在会话配置里声明「单回合」，Node 在初始回合后主动上报结束 | 需要同时修改 cloud proto、desktop controller-proto、`node-protocol` 与 Node 驱动，跨仓按序发布；结束判断落到 Node，与 Thread 决策「Cloud 按数据库时间判断空闲」的分工相反；会话配置需要新增字段并可靠区分工作流与 IssueRun 会话 |
| Cloud 直接按 `turnEnded` 结算节点并释放资源，不等会话终态 | 绕过 `node_executions.result`，违反根决策 D4 与不变量 5；会在会话仍存活时删除 Workspace，执行工作项与会话结果的收尾关系需要重新定义 |
| 在接管回执的 hook 里放出 `EndSession` | 崩溃窗口内的回执仍要由重算补发，两条写入路径还会竞争同一节点唯一的结束命令；重算天然幂等，一个写入者足够 |
| 为工作流会话复用 Thread 的 `thread_state` / `idle_since` | WorkflowRun 没有 Thread，也没有用户轮次；引入空闲窗口会让单回合会话白等一个为多轮对话设计的时间，且需要为 WorkflowRun 伪造 Thread 行 |
| 新增结束原因 `turn_completed` | 需要跨仓库枚举变更；Node 对结束原因的处理完全相同，工作流节点的结算不看原因，只有 IssueRun 的 `status` 映射使用它 |

## 本决策未解决的问题

- 结束确认窗口是否应与 `AgentEndConfirmTimeout` 分开配置：首期复用同一配置；若工作流会话的确认需求
  与 IssueRun 不同，由后续 ADR 拆分，本文不预设新的配置项。
- 工作流运行取消的写入者、产品入口与端到端验收由根决策 D6 的后续实现承接；Cloud 目前没有 WorkflowRun
  的取消写入者。`one_workflow_session_end` 对将来取消的限制与两条可选路径见 D5。
- **已派发但始终没有成功 `turnEnded` 的会话没有有界等待**：例如 Agent 卡住、回执丢失或 Node 通道中断时，
  会话会停在无结果状态，运行仍可能永久 `running`。本文的确认窗口只从放出 `end_session` 之后起算，覆盖
  不到这条路径；这是既有行为，不在本缺陷（ora-space/circus#291）的范围内，由
  [ora-space/circus#299](https://github.com/ora-space/circus/issues/299) 单独跟踪。

## 落地顺序

- 已实现（Cloud，运行 ora-space/circus#293）：迁移 `0049_thread_commands_workflow_run.sql`、调度器触发与
  有界等待；成功回合、`cancelled` 回合、按节点隔离、超时失败四组数据库测试与既有 workflow/thread 测试
  全绿。cloud#96 已合并（`644a65e2`），并在真实栈通过成功场景验收（运行 #293，成功运行 `847bca1a`）。
- 状态同步：本文已由落地运行 ora-space/circus#324 标记为 `implemented`，并按最终实现同步正文；
  [核心用例](../../../test-cases/cloud/workflow/session-end-after-initial-turn.md) 已登记直接测试与
  真实栈验收证据。
- 保持不变：图 schema、发布门禁、运行身份的冻结项、节点终态的证据来源、失败与取消的既有结算，以及
  Controller 的线上契约。
