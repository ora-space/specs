# 经 Node 宿主 MCP 的完成工具

依据[Cloud 经 Node 宿主 MCP 提供完成工具](../../../decisions/cloud/workflow/20261124-host-completion-tool-for-node-hosted-interactive-nodes.md)
（D1–D8）。这些用例覆盖 Cloud v2 把完成工具交给哪些会话、怎样授权一次上报、能力不足时怎样拒绝、停靠节点怎样跨重启，
以及凭据与 token 的留存边界。提供宿主 Server 一方的通用义务（文法、按能力派发、凭据有效期）在
[Controller 交付宿主 MCP](../../controller/session/host-mcp-servers.md)；默认值恢复为 `agent` 见
[From The Restore Version Explicit Agent Policies Take Effect Without Rewriting Saved Values](end-turn-default-before-completion-tool.md#from-the-restore-version-explicit-agent-policies-take-effect-without-rewriting-saved-values)；
desktop 上完成信号的门禁见[Agent 完成信号用例](../../desktop/core/workflow/agent-completion-signal.md)。

ADR 为 `approved`，首批实现已合入 cloud main。证据基线是合并提交
[`4ba9dc5a`](https://github.com/ora-space/cloud/commit/4ba9dc5a)：v2-server 的单元测试与真实 PostgreSQL 运行测试、前端测试。
除非另外写明，运行测试用假的 Agent 与 Node。D2 的旧 execution 结算与 D8 的提示还没有实现，对应义务为 `Missing`。

## Only Root Agent Policy Nodes Get The Completion Tool

- 风险：完成工具进入 `end_turn`、非交互节点、容器成员、普通对话或任务运行的会话，Agent 能完成不归它的节点；`agent`
  节点没有拿到工具，永远停在等待上报；工具混入插件 MCP 的选择；端点在本机回环之外可达。
- 前置状态：单机部署；图里有根作用域的 `agent` 与 `end_turn` 交互节点、非交互节点、Loop 或 Iteration 的成员。
- 触发：启动各节点的会话并发送后续轮次；启动普通对话和任务运行；向端点发送各种请求。
- 必须成立：只有根作用域、`interactive`、`completionSignal: "agent"` 的节点的会话带 `ora-workflow-completion`；列表只随
  start 发送一次，后续轮次不带；端点在 `127.0.0.1`，只有 `ora_complete_workflow_step` 一个工具，只回普通 JSON，限制请求
  大小；插件 MCP 的选择不变。
- 禁止结果：其他会话带有宿主列表或 token；后续轮次再次发送宿主列表；端点列出其他工具或提供事件流。
- 验证义务与证据：

| 义务 | 所需直接证据 | 状态 |
| --- | --- | --- |
| 只有根作用域的 `agent` 交互节点带完成工具 | [`TestOnlyAnAgentPolicySessionGetsTheCompletionTool`](https://github.com/ora-space/cloud/blob/4ba9dc5a/v2-server/internal/workflowhost/agent_completion_test.go)：`end_turn` 节点的首轮与后续轮次、非交互节点都不带，`agent` 节点带；[`TestHostToolsNeedANodeThatTakesThem`](https://github.com/ora-space/cloud/blob/4ba9dc5a/v2-server/internal/agentruntime/host_tools_test.go) 断言没有宿主工具的会话不带字段。缺口：没有 Loop 与 Iteration 成员、普通对话与任务运行的会话测试 | `Partial` |
| 列表只随 start 发送，token 按轮附在 prompt 之后 | [`TestHostToolsReachTheAgentAndNothingElse`](https://github.com/ora-space/cloud/blob/4ba9dc5a/v2-server/internal/agentruntime/host_tools_test.go)：start 带一项 Server，后续轮次不带，每轮末尾是各自的 token 块 | `Covered` |
| 端点只有一个工具、只回普通 JSON、限制大小 | [`TestTheEndpointServesTheOneTool`](https://github.com/ora-space/cloud/blob/4ba9dc5a/v2-server/internal/completiontool/server_test.go)：`tools/list` 只有一项，其他工具与方法报错，`GET` 为 405，超长请求为 413。缺口：没有测试请求时长上限，也没有直接断言只绑定 `127.0.0.1` | `Partial` |
| 不进入插件 MCP 的选择 | 会话启动测试：同时有插件 MCP 选择与宿主列表时，插件选择与宿主列表互不影响 | `Missing` |

- 决策依据：D1，不变量 1；完成信号 ADR D1。

## A Report Counts Only For Its Own Live Turn

- 风险：别的节点的凭据或 token 完成本节点；上一轮的 token 落到下一轮；持久状态已经不在执行，内存里的记录仍接受调用；
  被停止、被截断或被拒绝的一轮留下有效信号；插件刷新重启 Agent 之后，已排队的一轮的 token 失效，Agent 无法上报。
- 前置状态：两个 `agent` 节点各在执行一轮；另有停靠、已完成、已取消的节点。
- 触发：用自己的、别的节点的、上一轮的、编造的 token 调用；重复调用；停止、截断、拒绝、丢失 Agent 后再停靠；持久状态读
  不到时调用；Node 在同一 execution 内重启 Agent，下一轮的 prompt 已经排队。
- 必须成立：只有凭据、token 与持久状态三者都成立才接受；重复调用返回同一个 `signalId`；拒绝给出稳定原因，
  `temporarily_unavailable` 不作废 token；只有正常结算的一轮保留上报；同一 execution 内重启 Agent 后，端点凭据与已排队那一轮
  的 token 继续有效，上一轮的 token 已经失效。
- 禁止结果：被拒绝的调用留下信号；上一轮的上报算进下一轮；不是正常结束的一轮带着信号停靠。
- 验证义务与证据：

| 义务 | 所需直接证据 | 状态 |
| --- | --- | --- |
| 跨节点的凭据或 token 被拒绝 | [`TestAnotherNodesCredentialOrTokenIsRefused`](https://github.com/ora-space/cloud/blob/4ba9dc5a/v2-server/internal/completiontool/server_test.go)、[`TestAReportComesOnlyFromTheNodesOwnLiveTurn`](https://github.com/ora-space/cloud/blob/4ba9dc5a/v2-server/internal/workflowhost/agent_completion_test.go)：两个方向都拒绝，停靠后没有信号 | `Covered` |
| 一轮一个 token，重复调用幂等，停住后失效 | [`TestACallIsAcceptedOnceAndOnlyForItsOwnTurn`](https://github.com/ora-space/cloud/blob/4ba9dc5a/v2-server/internal/completiontool/server_test.go)、[`TestAReportDoesNotOutliveItsTurn`](https://github.com/ora-space/cloud/blob/4ba9dc5a/v2-server/internal/workflowhost/agent_completion_test.go)：重复调用返回同一信号，上一轮的 token 为 `stale_attempt`，旧信号的确认被拒绝 | `Covered` |
| 以持久状态判定一轮是否在执行 | [`TestWorkflowNodeTurnLive`](https://github.com/ora-space/cloud/blob/4ba9dc5a/v2-server/internal/store/workflow_completion_tool_test.go)：未绑定、停靠、轮次已结束、运行已取消各自拒绝；[`TestStoredStateDecidesACall`](https://github.com/ora-space/cloud/blob/4ba9dc5a/v2-server/internal/completiontool/server_test.go)：读不到时 `temporarily_unavailable` 且 token 不作废；[`TestTheToolFollowsStoredExecutionState`](https://github.com/ora-space/cloud/blob/4ba9dc5a/v2-server/internal/workflowhost/agent_completion_execution_test.go)：真实执行服务与 PostgreSQL 上，执行中接受、停靠后不再 live。已知缺口：首轮在会话绑定前立即上报会被拒，没有测试覆盖这个窗口 | `Partial` |
| 不是正常结束的一轮丢弃上报 | [`TestAReportOfATurnThatDidNotEndNormallyIsDropped`](https://github.com/ora-space/cloud/blob/4ba9dc5a/v2-server/internal/workflowhost/agent_completion_test.go)：停止、截断、拒绝、未知停止与 Agent 丢失都不留信号，强制完成记 `requester: human`；[`TestAReportDoesNotCoverATurnThatDidNotEndNormally`](https://github.com/ora-space/cloud/blob/4ba9dc5a/v2-server/internal/workflowhost/agent_completion_test.go)：拒绝的一轮不被自动接受 | `Covered` |
| 同一 execution 内重启 Agent 后沿用凭据与该轮 token | Controller 与 Node 测试：插件刷新重启 Agent 后，已排队的一轮用原 URL 与原 token 上报成功，上一轮的 token 被拒绝 | `Missing` |

- 决策依据：D2（凭据按 execution）、D3、D5，不变量 2、3；完成信号 ADR D1（本决策修改的一处）、D2、D3。

## Missing Host Capability Refuses Start And Resume Without Downgrade

- 风险：前面的节点用掉模型额度之后才发现没有完成工具；能力不足时悄悄按 `end_turn` 执行；Node 的临时故障被报成缺少完成工具；
  “从失败处继续”绕过启动时的检查。
- 前置状态：冻结图含根作用域的 `agent` 节点；Node 分别声明与不声明 `session_host_mcp_servers`，Agent 分别支持与不支持
  HTTP MCP；另有一个失败的运行可以从失败处继续。
- 触发：启动运行（Node 已运行或没有运行）；从失败处继续；准入之后换成不具备能力的 Node；启动不声明 HTTP MCP 的 Agent。
- 必须成立：Node 没有运行时先启动它再判断；启动与“从失败处继续”缺能力时以 `workflow_completion_tool_unavailable` 拒绝，没有节点开始运行，运行状态不变；
  Node 的临时故障按执行服务不可用报告；派发时缺能力以 `host_mcp_node_capability_missing` 失败，Agent 不声明 HTTP MCP 以
  `host_mcp_http_capability_missing` 失败；不含 `agent` 节点的图不受影响。
- 禁止结果：`agent` 节点按 `end_turn` 执行；被拒绝的启动或继续留下运行中的节点。
- 验证义务与证据：

| 义务 | 所需直接证据 | 状态 |
| --- | --- | --- |
| 准入只看根作用域、已接入的 `agent` 交互节点 | [`TestUsesAgentCompletionCountsOnlyParkedNodes`](https://github.com/ora-space/cloud/blob/4ba9dc5a/v2-server/internal/workflow/completion_internal_test.go)：非交互、未接入与 Loop 成员不触发准入检查。缺口：没有 Iteration 成员 | `Partial` |
| Node 没有运行时先启动它再判断 | Controller 测试：Node 没有运行时启动含 `agent` 节点的运行，断言先启动 Node 再判断能力，分别覆盖具备与不具备能力 | `Missing` |
| 启动准入拒绝，临时故障不算缺少能力 | [`TestARunIsRefusedAtItsStartWithoutTheTool`](https://github.com/ora-space/cloud/blob/4ba9dc5a/v2-server/internal/workflowhost/agent_completion_test.go)：缺能力时运行仍为 pending、没有会话启动；Node 不可用与请求取消返回原错误；`end_turn` 图照常启动 | `Covered` |
| “从失败处继续”同样准入 | Controller 测试：缺能力时继续被同一原因拒绝，运行与节点状态不变；具备能力后可以继续 | `Missing` |
| 派发时再查，Agent 不支持 HTTP MCP 时明确失败 | [`TestHostToolsNeedANodeThatTakesThem`](https://github.com/ora-space/cloud/blob/4ba9dc5a/v2-server/internal/agentruntime/host_tools_test.go)：Node 不声明能力时不发送宿主列表，以 `host_mcp_node_capability_missing` 结束；[`TestAnAgentWithoutHTTPMCPKeepsNodesDetail`](https://github.com/ora-space/cloud/blob/4ba9dc5a/v2-server/internal/agentruntime/host_tools_test.go) 与 [`TestAnAgentWithoutHTTPMCPFailsTheNode`](https://github.com/ora-space/cloud/blob/4ba9dc5a/v2-server/internal/workflowhost/agent_completion_test.go)：Node 给出的原因原样到达节点，节点与运行失败 | `Covered` |

- 决策依据：D4，不变量 4；宿主 MCP ADR D4、D6。

## Agent Completion Waits For Confirmation Or A Starter Turn

- 风险：Agent 的一次上报不经确认就推进后继；自动接受在首轮或异常停靠时完成节点；自动接受绕过交付物检查；确认了过期的
  信号；`force` 越过过期检查；审计把人的决定记成 Agent 的。
- 前置状态：`agent` 节点，部分开启 `autoAcceptAgentCompletion`，部分声明交付文件。
- 触发：首轮与后续轮次分别带或不带上报；普通完成、确认首轮、确认信号、强制完成以及它们的组合。
- 必须成立：首轮需要确认首轮；有上报时还要确认该信号；过期信号返回 `workflow_node_signal_changed`，`force` 不能越过；没有
  上报时按完成信号 ADR D3 强制完成；自动接受只作用于人发起、正常结束、只剩 `agent_confirmation_pending` 的一轮，走同一
  门禁；`requester` 与 `confirmation` 由服务端推导；节点提交完成结果之后后继才启动。
- 禁止结果：被拒绝的完成推进了运行；首轮被自动接受；自动接受越过缺失的交付物。
- 验证义务与证据：

| 义务 | 所需直接证据 | 状态 |
| --- | --- | --- |
| 首轮的两个确认缺一不可，后继在完成之后启动 | [`TestAReportedFirstTurnNeedsBothConfirmations`](https://github.com/ora-space/cloud/blob/4ba9dc5a/v2-server/internal/workflowhost/agent_completion_test.go)：五种不完整的请求各自被拒绝且不推进运行，完成后记 `requester: agent`、`confirmation.by: human` | `Covered` |
| 没有上报时由人确认并强制完成 | [`TestAnUnreportedFirstTurnTakesConfirmationAndForceTogether`](https://github.com/ora-space/cloud/blob/4ba9dc5a/v2-server/internal/workflowhost/agent_completion_test.go)：只确认或只强制都被拒绝，合起来才完成，记 `requester: human` 与被越过的 blocker | `Covered` |
| 过期信号被拒绝，`force` 不越过 | [`TestAReportDoesNotOutliveItsTurn`](https://github.com/ora-space/cloud/blob/4ba9dc5a/v2-server/internal/workflowhost/agent_completion_test.go)：带或不带 `force` 都返回信号已变；[`TestACompletionOfAnotherReportIsStale`](https://github.com/ora-space/cloud/blob/4ba9dc5a/v2-server/internal/workflowhost/agent_completion_test.go)：五种存储与现场的组合逐一判断 | `Covered` |
| 强制越过未确认的信号有审计 | [`TestForcingPastAReportRecordsThatNobodyConfirmedIt`](https://github.com/ora-space/cloud/blob/4ba9dc5a/v2-server/internal/workflowhost/agent_completion_test.go)：保留信号，没有 `confirmation`，记下被越过的 blocker | `Covered` |
| 自动接受只在允许时生效，走同一门禁 | [`TestAutoAcceptTakesAReportedHumanTurn`](https://github.com/ora-space/cloud/blob/4ba9dc5a/v2-server/internal/workflowhost/agent_completion_test.go)：首轮与没有上报的一轮不接受，人发起且上报的一轮完成，记 `confirmation.by` 为自动；[`TestAutoAcceptRefusedByTheGateLeavesTheNodeToTheStarter`](https://github.com/ora-space/cloud/blob/4ba9dc5a/v2-server/internal/workflowhost/agent_completion_test.go)：交付物缺失时留给发起人 | `Covered` |

- 决策依据：D5，不变量 6；完成信号 ADR D3–D6。

## A Parked Node Survives A Restart While Its Old Execution Is Settled

- 风险：重启后旧凭据仍被接受；停靠会话的旧 execution 一直没有结论，也不知道为什么结束；宿主 Server 在旧 execution 结算之前
  就开始服务；为了结算把停靠会话一起结束；重建的轮次与信号对不上却仍然算数。
- 前置状态：`agent` 节点停靠等待，有或没有有效信号；另有没有停靠、正在执行的会话。
- 触发：停止 Cloud 进程（正常退出或被杀）后启动；用旧凭据与旧 token 调用；在停靠节点继续对话并完成。
- 必须成立：旧凭据与 token 全部失效；停靠节点的会话、完成策略与同一轮的信号仍在，可以继续对话并完成；在宿主 Server 开始
  服务之前，停靠会话的旧 execution 经正常结束路径结算，并记录有名字的重启原因；下一轮是新的 execution，拿到新的凭据与
  token；重建后与 `turnKey` 不一致的信号丢弃；升级前按 `end_turn` 停靠的节点不附加完成工具。
- 禁止结果：重启前的凭据被接受；旧 execution 没有结算就开始服务；停靠会话因重启结束。
- 验证义务与证据：

| 义务 | 所需直接证据 | 状态 |
| --- | --- | --- |
| 停靠会话、策略与信号保留，凭据与 token 换新 | [`TestAReportSurvivesARestartButItsCredentialsDoNot`](https://github.com/ora-space/cloud/blob/4ba9dc5a/v2-server/internal/workflowhost/agent_completion_test.go)：同一信号与 blocker 仍在，旧 token 被拒绝，新的 token 可以上报；[`TestRecoveryKeepsTheSessionOfAParkedInteractiveNode`](https://github.com/ora-space/cloud/blob/4ba9dc5a/v2-server/internal/store/interactive_recovery_test.go)：停靠会话仍打开，正在执行的会话结束。缺口：没有用重启前的端点 URL 再调用 | `Partial` |
| 停靠会话的旧 execution 在服务前结算，并有有名字的原因 | 重启测试（真实 PostgreSQL）：宿主 Server 接受第一个请求之前，旧 execution 已结算，记录里有有名字的重启原因，会话仍然打开 | `Missing` |
| 宿主 Server 在旧 Node 停止与恢复之后才监听 | [`TestTheCompletionToolIsServedAfterEarlierNodesStopped`](https://github.com/ora-space/cloud/blob/4ba9dc5a/v2-server/completion_tool_order_test.go)：只断言 `main.go` 源码里的顺序，没有运行重启 | `Partial` |
| 对不上的信号丢弃 | [`TestARestartDropsAReportOfAnotherTurn`](https://github.com/ora-space/cloud/blob/4ba9dc5a/v2-server/internal/workflowhost/agent_completion_test.go)：多出一轮时信号不再算数 | `Covered` |
| 升级前的停靠节点保持 `end_turn` | [`TestANodeParkedBeforeTheToolStaysEndTurn`](https://github.com/ora-space/cloud/blob/4ba9dc5a/v2-server/internal/workflowhost/agent_completion_test.go)：摘要记录为 `end_turn` 的节点在重启后不带工具，由发起人普通完成 | `Covered` |

- 决策依据：D2、D6，不变量 5、8；宿主 MCP ADR D6“启动时检查可恢复性”。

## An Unconnected Completion Tool Is Shown Without Failing The Node

- 风险：Agent 丢掉连不上的 Server 照常启动，发起人分不清是没有上报还是没有工具；识别之后把节点判为失败，丢掉已完成的产出。
- 前置状态：`agent` 节点的 execution 首次停住之前，端点分别收到与没有收到 `initialize` / `tools/list`。
- 触发：首轮停住；发起人查看节点、继续对话、强制完成。
- 必须成立：没有收到时节点上提示完成工具没有连上；节点与运行的状态、完成门禁不变，可以继续对话或“仍然完成”；收到过的
  节点没有提示；提示不含凭据或 token。
- 禁止结果：节点因此失败；提示改变 blocker；`end_turn` 节点出现提示。
- 验证义务与证据：

| 义务 | 所需直接证据 | 状态 |
| --- | --- | --- |
| 按凭据记录，首次停住前没有收到时提示 | Controller 运行测试：收到与没有收到两种 execution，比较节点上的提示 | `Missing` |
| 提示不改变状态与门禁 | Controller 运行测试：带提示的节点仍停在 `agent_signal_pending`，继续对话与强制完成照常可用 | `Missing` |
| 发起人看得到提示 | 前端测试：节点页面显示提示的中英文 | `Missing` |
| 提示的存储与显示没有凭据或 token | Controller 与前端测试：检索提示的存储、接口回答与页面文字 | `Missing` |

- 决策依据：D8，不变量 9；宿主 MCP ADR D2。

## Completion Credentials And Tokens Stay Out Of Cloud Surfaces

- 风险：端点凭据或 token 进入 Cloud 的日志、事件、数据库、会话历史、最终回答或 Node 事件诊断文件；重放的旧事件没有脱敏；
  Node 自己的持久输入或诊断留下端点凭据。
- 前置状态：`agent` 节点的 Agent 在回答与工具调用里回显 token 与端点 URL；Node 重放本进程不认识的 execution 的事件。
- 触发：正常结束一轮；Node 重建后重放事件；格式化宿主列表；检索各处存储。
- 必须成立：Cloud 的各个持久面都没有凭据或 token，token 块从发布的历史里去掉；未知 execution 的 Agent 会话消息按结构脱敏；
  Node 自身的持久输入、日志与诊断没有端点凭据。允许留存的只有：token 在 Node 的会话文件与 ledger 以及 Agent 自己的存储里；
  端点凭据在 Agent 自己的存储与宿主插件日志里 Agent 自写的 stderr 里。
- 禁止结果：上述允许之外的任何位置出现凭据或 token。
- 验证义务与证据：

| 义务 | 所需直接证据 | 状态 |
| --- | --- | --- |
| 事件、回答与诊断文件没有凭据或 token | [`TestHostToolsReachTheAgentAndNothingElse`](https://github.com/ora-space/cloud/blob/4ba9dc5a/v2-server/internal/agentruntime/host_tools_test.go)：回答与诊断文件里换成占位符，发布的事件里没有 token 块；[`TestTheJournalHasNoCredentialOfAnUnknownExecution`](https://github.com/ora-space/cloud/blob/4ba9dc5a/v2-server/internal/agentruntime/host_tools_test.go)：未知 execution 的四类会话消息在诊断文件里都没有 token 与凭据 | `Covered` |
| 节点、轮次与 prompt 的存储没有 token | [`TestTheToolFollowsStoredExecutionState`](https://github.com/ora-space/cloud/blob/4ba9dc5a/v2-server/internal/workflowhost/agent_completion_execution_test.go)：真实 PostgreSQL 上节点与轮次（含 prompt）都不含 token。缺口：没有断言端点凭据，没有检索整个数据库与进程日志 | `Partial` |
| 宿主列表不会被格式化出来 | [`TestAHostMCPServerNeverFormatsItsCredentials`](https://github.com/ora-space/cloud/blob/4ba9dc5a/v2-server/internal/execution/hostmcp_test.go) | `Covered` |
| 真实 Agent 的留存在允许范围内 | 真实 Node 与 OpenCode、Claude Code 各运行一次，检索 Node、Agent 与 Cloud 的存储、日志和进程参数。OpenCode 在 Node `447a59de` 上人工检索过一次，没有命中端点凭据，没有自动测试固定；Claude Code 没有验证 | `Missing` |

- 决策依据：D7，不变量 7；宿主 MCP ADR D3、D6。
