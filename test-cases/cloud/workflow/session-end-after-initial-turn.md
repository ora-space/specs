# 工作流会话结束与有界等待

依据[工作流会话结束后续决策](../../../decisions/cloud/workflow/20261019-workflow-session-end-after-initial-turn.md)
（D1、D3、D4、D5）。这些用例覆盖单回合工作流会话的结束请求、按节点隔离、有界等待与会话结果的证据地位；
调度器整体行为见[调度串行、节点提交与恢复](scheduler-serialism-and-recovery.md)。

证据状态按 ADR 生命周期在 `implemented` 阶段登记。实现 PR [cloud#96](https://github.com/ora-space/cloud/pull/96)
已带直接测试（`TestWorkflowSessionEndsAfterSuccessfulTurn`、`TestWorkflowSessionCancelledTurnDoesNotRequestEnd`、
`TestWorkflowSessionEndRequestsAreScopedPerNode`、`TestWorkflowSessionEndDeadlineFailsNode`），合并前下表按当前
主分支状态记为 `Missing`；落地运行时按合并后的测试重新登记。

## A Successful Turn Releases Exactly One Scoped Session End Request

- 风险：成功回合之后没有任何写入者请求结束会话，运行永久 `running`（本决策要修复的缺陷）；重复重算放出
  第二条命令，或把某个节点的结束请求投给它自己的会话之外；`cancelled` 的回合被当作完成，提前结束会话。
- 前置状态：运行已派发一个 agent 节点，`node_executions.result` 为空；会话回执里最近一条是 `turnEnded`。
  运行可以先后执行多个 agent 节点。
- 触发：调度器按 10 秒周期重算，同一节点被连续重算；同一运行的两个 agent 节点先后进入执行；回执的
  `stop_reason` 为 `end_turn`、`max_tokens`、`refusal` 或 `cancelled`。
- 必须成立：最近一条回执是成功 `turnEnded`（`stop_reason != cancelled`）后，Cloud 放出**恰好一条**
  `end_session{idle_timeout}`；命令归属该工作流运行，并限定在发起请求的图节点上；重复重算返回同一条命令，
  不新增、不重置计时；`cancelled` 的回执不放出命令。
- 禁止结果：成功回合后没有命令；同一节点出现第二条 `end_session`；命令被投递给同一运行的另一个节点的
  会话；`cancelled` 回合触发结束请求；重算改变命令的 `created_at` 或确认窗口。
- 验证义务与证据：

| 义务 | 所需直接证据 | cloud 侧状态 |
| --- | --- | --- |
| 成功回合后恰好一条 `end_session{idle_timeout}`，重复重算幂等 | 真实 PostgreSQL：驱动调度重算，检查命令数、kind/body 与重复 tick 后的行不变 | Missing |
| 命令按图节点限定，跨节点投递被拒绝 | 真实 PostgreSQL：两个 agent 节点的运行，检查命令的节点归属与跨节点投递的 `dispatch_conflict` | Missing |
| `cancelled` 的 `turnEnded` 不放出命令 | 真实 PostgreSQL：`stop_reason = cancelled` 的回执后检查命令为 0 条 | Missing |

- 决策依据：D1（触发判据与触发器位置）、D2（复用 `idle_timeout`）、D3（归属与节点限定、每节点唯一索引），
  不变量 1、2、6。

## An Unconfirmed End Request Fails The Node Within The Bounded Window

- 风险：结束请求已放出，但 Node 始终不领取命令或会话始终不上报终态，运行又一次永久 `running`；确认窗口
  从错误的起点或进程时钟计算，多副本与崩溃重试得到不同的失败时点；超时路径伪造成功或泄漏 Workspace。
- 前置状态：该节点的 `end_session` 命令已存在；`node_executions.result` 为空。
- 触发：重算发生在确认窗口内与窗口之后；同一个节点被重放请求；命令的 `created_at` 由数据库写入。
- 必须成立：窗口内节点与运行保持 `running`，不提前失败；超过窗口（按数据库时钟从命令 `created_at` 起算，
  复用 `AgentEndConfirmTimeout`）仍无会话终态时，节点以 `session_end_deadline_exceeded` 失败，运行进入
  失败终态，并按根决策 D7 声明删除运行 Workspace；重放不重置窗口。
- 禁止结果：窗口内判失败；窗口之后一直等待；把超时写成成功或伪造会话结果；运行进入终态但 Workspace
  继续存活。
- 验证义务与证据：

| 义务 | 所需直接证据 | cloud 侧状态 |
| --- | --- | --- |
| 窗口之后以命名原因失败并释放资源，窗口内不失败 | 真实 PostgreSQL：命令超过窗口后重算，检查节点错误、运行终态与 Workspace 删除声明 | Missing |
| 窗口从命令 `created_at` 起算，重放不重置 | 真实 PostgreSQL：重复重算与已有命令的 `created_at` 保持不变 | Missing |

- 决策依据：D4（有界等待、数据库时钟、复用配置）、D5（超时是唯一由 Cloud 直接判定的失败路径），
  不变量 4，根决策 D7 与不变量 5。

## The Session Result Remains The Only Terminal Evidence

- 风险：Cloud 从回执直接结算节点（被否定的方案 C），在会话仍存活时推进后继节点或删除 Workspace；迟到与
  重复的终态回执二次推进图。
- 前置状态：结束命令已投递，会话即将上报终态；可能先到达重复或迟到的回执。
- 触发：Node 上报会话终态；同一终态回执被重复交付；终态在结束命令之后才到达。
- 必须成立：节点只在 `node_executions.result` 提交后推进；成功回合的结束请求不改变节点状态；重复与迟到的
  终态回执是空操作；节点/运行的终态与 Workspace 释放顺序不因结束请求改变。
- 禁止结果：结束命令本身把节点记为成功；没有已提交结果的节点进入成功；终态回执被应用两次或把运行改回
  `running`。
- 验证义务与证据：

| 义务 | 所需直接证据 | cloud 侧状态 |
| --- | --- | --- |
| 节点只在会话结果提交后推进，重复/迟到回执是空操作 | 真实 PostgreSQL：重放同一终态回执，比较运行、节点与版本 | Partial |
| 结束请求之后的成功路径由会话结果结算节点与运行 | 真实 PostgreSQL：投递命令后上报终态，检查节点 succeeded 与会话原因映射 | Missing |

- 现有证据：Cloud 已合并的 `TestWorkflowSessionLateReceiptIsANoOp` 与
  `TestWorkflowSessionDispatchIsNotRepeatable` 覆盖重复回执不推进图（见
  [调度串行、节点提交与恢复](scheduler-serialism-and-recovery.md#a-node-result-commit-is-a-compare-and-swap-and-late-callbacks-are-no-ops)）；
  结束请求之后的结算路径由实现 PR cloud#96 的直接测试补足，落地阶段登记。
- 决策依据：D5（节点终态仍只来自会话结果）、根决策 D4 与不变量 5，不变量 3。
