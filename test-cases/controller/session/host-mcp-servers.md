# Controller 交付宿主 MCP

依据[Controller 交付宿主 MCP 决策](../../../decisions/node/agent-runtime/20261118-host-mcp-servers-in-agent-session-spec.md)
D1 与 D6。这些义务只能由提供宿主 Server 的一方（Controller 侧，在 Cloud v2 中就是 Cloud 进程）验证，Node
侧义务见[Node 承载的宿主 MCP 用例](../../node/agent-runtime/host-mcp-servers.md)。第一个使用方是
[Cloud 经 Node 宿主 MCP 提供完成工具](../../../decisions/cloud/workflow/20261124-host-completion-tool-for-node-hosted-interactive-nodes.md)
（`approved`），工作流侧的义务在[完成工具用例](../../cloud/workflow/host-completion-tool.md)。下表按首批实现合入 cloud main 的提交
[`4ba9dc5a`](https://github.com/ora-space/cloud/commit/4ba9dc5a) 登记直接测试，缺口如实保留；停靠会话旧 execution 的结算
还没有实现。

## Controller Sends Only Valid Host Servers To Capable Nodes

### 风险

Controller 用 Go `net/url` 之类非 WHATWG 解析器的“解析再序列化相等”判断 URL，把 Node 判为非法的输入发出去，
Node 关闭整条控制连接，重连后重发又被关闭；向未声明能力的旧 Node 发送字段让它拒绝整帧；需要宿主能力的
节点被派到不具备能力的 Node 后静默降级；前面的节点已经跑完、用掉模型额度后才在派发时拒绝；给 Agent 无法
到达的网络环境配了 loopback 地址。

### 前置状态

可派发的 Node 分别声明与不声明 `session_host_mcp_servers`；单机部署，Agent 与 Controller 在同一网络
命名空间；Controller 使用 `node-protocol` 的共用接受与拒绝样例。

### 触发

用样例中的每一项构造宿主列表；启动冻结图里含 `agent` 完成策略交互节点的 WorkflowRun，而可用 Node 不具备
能力。

### 必须成立

- 发送前按 D1 文法逐字符校验，对共用样例逐项给出与标注相同的判断，非法输入不发送。
- 只向同时声明 `session_host_mcp_servers` 与 `agent_session` 的 Node 发送非空列表。
- 需要宿主能力而没有具备能力的 Node 时，在 WorkflowRun 启动准入时以有名字的原因拒绝（或另选 Node），
  不降级为不需要宿主能力的行为。
- 单机部署的端点只绑定 `127.0.0.1`，发给 Node 的 URL 是回环明文；loopback 明文只用于与 Agent 同网络命名空间的端点。
- 多机、容器或沙箱部署的 URL 怎样取自部署配置，由另立的决策规定后再登记。

### 禁止结果

- 任何样例在 Controller 上的判断与标注不同；旧 Node 收到 `host_mcp_servers`。
- 需要宿主能力的节点在不具备能力的 Node 上运行，或者在前序节点已经运行后才被拒绝。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 共用样例判断一致 | Controller 校验测试遍历共用样例文件，逐项断言接受或拒绝与标注相同。证据：[`TestHostMCPServersMatchNodesSharedSamples`](https://github.com/ora-space/cloud/blob/4ba9dc5a/v2-server/internal/execution/hostmcp_test.go) 遍历 `url_cases` 与 `list_cases`，逐项比较判断；[`TestAnInvalidHostListIsNotSent`](https://github.com/ora-space/cloud/blob/4ba9dc5a/v2-server/internal/agentruntime/host_tools_test.go)：非法列表不发送。缺口：`raw_cases` 没有断言（Controller 的类型无法构造）；样例是锁定 Node 提交上的副本，没有测试检查副本与来源一致 | Partial |
| 按能力派发与启动准入 | Controller 测试：不声明能力的 Node 收不到字段；需要宿主能力而无可用 Node 时，WorkflowRun 在启动准入时以有名字的原因被拒绝，没有任何节点开始运行。证据：[`TestHostToolsNeedANodeThatTakesThem`](https://github.com/ora-space/cloud/blob/4ba9dc5a/v2-server/internal/agentruntime/host_tools_test.go)、[`TestARunIsRefusedAtItsStartWithoutTheTool`](https://github.com/ora-space/cloud/blob/4ba9dc5a/v2-server/internal/workflowhost/agent_completion_test.go)。缺口：没有改变 `agent_session` 的声明 | Partial |
| 单机部署的端点只在本机回环 | Controller 测试：真实端点只绑定 `127.0.0.1`，发给 Node 的 URL 是回环明文。证据：[`TestTheEndpointServesTheOneTool`](https://github.com/ora-space/cloud/blob/4ba9dc5a/v2-server/internal/completiontool/server_test.go) 用真实端点签发的 URL 通过 D1 文法校验并可以调用，文法只允许字面回环地址用明文 http；[`TestHostToolsReachTheAgentAndNothingElse`](https://github.com/ora-space/cloud/blob/4ba9dc5a/v2-server/internal/agentruntime/host_tools_test.go)：给定的 URL 原样随 start 发给 Node。缺口：没有直接断言监听地址与 URL 的 scheme、主机。多机、容器或沙箱部署的部署配置另立决策后再登记 | Partial |

### 决策依据

D1、D6“按能力派发、启动前准入”，不变量 1、5；“可达性由部署配置决定”在单机部署上由使用方决策 D1 修改。

## Host Endpoints Stay Valid Or The Execution Ends Before Serving

### 风险

Controller 照 Desktop 的写法每个进程随机绑定端口、随机生成 token，Controller 重启后 Node 上仍在运行的会话
拿着失效的 URL，节点永远等不到完成信号；Controller 在“已发送、未持久化”时崩溃，重启后生成新凭据而 Node
保留第一次的值，每次调用都被拒绝；Controller 靠重发 start 轮换凭据，但 Node 忽略重发的值。

### 前置状态

一个 Running 的 execution 带宿主 Server。Controller 采用两种合规做法之一：凭据持久保存或可推导、跨重启
仍有效；或者重启时、宿主 Server 提供服务之前结束所有受影响的 execution（Cloud v2 重启即中断全部
execution）。

### 触发

重启 Controller 进程，之后 Agent 调用宿主工具；在发送 start 与持久化之间杀死 Controller 再重启；在采用第一种
做法时模拟凭据无法恢复（例如推导密钥丢失）；重发 start。

### 必须成立

二选一，且至少满足其一：

- 重启后同一 URL 与请求头仍然被宿主 Server 接受，调用在当前 attempt 内照常生效；
- 或者在宿主 Server 开始提供服务之前，Controller 经正常结束路径结束该 execution，并记录有名字的原因。

无论哪种做法：

- 端点和凭据（或推导它们的输入）在第一次发送 start 之前已经持久化；发送与持久化之间崩溃后，重启后的
  值与 Node 保留的值相同，或者该 execution 在提供服务前被结束。
- 凭据无法恢复时，启动检查在提供服务前结束该 execution。
- 每次重发 start 的宿主列表与第一次逐字节相同。

### 禁止结果

- 重启后在用 execution 的宿主调用被拒绝而 execution 继续运行，只能靠 `force` 结束节点。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 跨重启有效或在提供服务前结束 | Controller 重启测试：同一 execution 在重启后能成功调用宿主工具，或者在宿主 Server 接受第一个请求之前已经以有名字的原因结束。选择后者。证据：[`TestAReportSurvivesARestartButItsCredentialsDoNot`](https://github.com/ora-space/cloud/blob/4ba9dc5a/v2-server/internal/workflowhost/agent_completion_test.go) 断言重启前的 token 被拒绝；[`TestRecoveryKeepsTheSessionOfAParkedInteractiveNode`](https://github.com/ora-space/cloud/blob/4ba9dc5a/v2-server/internal/store/interactive_recovery_test.go) 断言没有停靠的会话在恢复时结束；[`TestTheCompletionToolIsServedAfterEarlierNodesStopped`](https://github.com/ora-space/cloud/blob/4ba9dc5a/v2-server/completion_tool_order_test.go) 只断言源码里的启动顺序。缺口：停靠会话的旧 execution 还没有在提供服务前结算并记录原因 | Partial |
| 先持久化再发送 | Controller 测试：在发送与持久化之间注入崩溃，重启后的值与第一次发送的逐字节相同，或该 execution 在提供服务前被结束。选择“在提供服务前结束”；没有注入崩溃的测试 | Missing |
| 无法恢复时启动检查结束 execution | Controller 测试：凭据不可恢复时，启动检查在提供服务前走正常结束路径并记录原因。证据：[`TestRecoveryKeepsTheSessionOfAParkedInteractiveNode`](https://github.com/ora-space/cloud/blob/4ba9dc5a/v2-server/internal/store/interactive_recovery_test.go)：没有停靠的会话在恢复时结束。缺口：没有断言原因，停靠会话的旧 execution 没有结算 | Partial |
| 重发相同字节 | Controller 测试：重发 start 时宿主列表与首次发送逐字节相同。没有重发 start 的测试 | Missing |

### 决策依据

D3“重发 start 时值被忽略”，D6“先持久化再发送，重发相同字节”与“启动时检查可恢复性”，不变量 3、9。

## Host Credentials Follow The Persisted Execution State And Bind Each Attempt

### 风险

吊销名单只在内存中，凭据又可以推导，Controller 重启后已结束 execution 的凭据又能用了；依赖推送事件吊销，
而 `stop_unconfirmed` 只能经 `GetExecutionStatus` 查到，可能始终没有终态；宿主 Server 把“请求到达了端点”当作
授权，接受另一个 execution 或早先 attempt 的 token；开启 `autoAcceptAgentCompletion` 的节点因跨 execution 或
跨 attempt 的调用直接完成。

### 前置状态

同一 Node 上两个 execution 各带自己的宿主凭据；其中一个节点开启 `autoAcceptAgentCompletion`。

本用例只证明跨 execution、跨 attempt 的重放被拒绝；同一系统用户下读取当前 prompt 的进程不在保护范围内
（决策 D5、D6），不登记为义务。

### 触发

用另一个 execution 的凭据或 token 调用；用早先 attempt 的 token 调用；在以下每种事件之后再调用：Controller
主动结束、发出 `EndSession`、收到 `AgentSessionEnded`、Controller 把 execution 的持久记录标为中断、经
`GetExecutionStatus` 观察到 `StopUnconfirmed`；execution 结束（吊销）后重启 Controller 再调用。

### 必须成立

- 凭据只对签发它的 execution 有效，且只在 Controller 持久的 execution 记录处于非终态时有效；以上每种事件
  最先发生时记录离开运行态，之后的调用被拒绝。
- 吊销后重启 Controller，已结束 execution 的凭据仍被拒绝。
- token 只在绑定的会话与 attempt 内短期有效；越权、过期或已失效的调用被拒绝且不改变任何状态，包括开启
  自动接受的节点。

### 禁止结果

- 跨 execution 或跨 attempt 的调用产生完成信号；记录已离开运行态后的调用被接受，包括重启之后。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 按 execution 隔离凭据 | 宿主 Server 测试：另一个 execution 的凭据被拒绝且状态不变。证据：[`TestAnotherNodesCredentialOrTokenIsRefused`](https://github.com/ora-space/cloud/blob/4ba9dc5a/v2-server/internal/completiontool/server_test.go)、[`TestCredentialsEndWithTheirAgent`](https://github.com/ora-space/cloud/blob/4ba9dc5a/v2-server/internal/completiontool/server_test.go)：另一个节点的凭据、换掉的 execution 的凭据与编造的凭据都被拒绝，没有信号 | Covered |
| attempt 绑定与有效期 | 宿主 Server 测试：早先 attempt 或过期的 token 被拒绝且状态不变。证据：[`TestACallIsAcceptedOnceAndOnlyForItsOwnTurn`](https://github.com/ora-space/cloud/blob/4ba9dc5a/v2-server/internal/completiontool/server_test.go)、[`TestResendingAPromptReplacesItsToken`](https://github.com/ora-space/cloud/blob/4ba9dc5a/v2-server/internal/completiontool/server_test.go)、[`TestAReportDoesNotOutliveItsTurn`](https://github.com/ora-space/cloud/blob/4ba9dc5a/v2-server/internal/workflowhost/agent_completion_test.go) | Covered |
| 以持久状态判定有效期 | Controller 测试：五种离开运行态的事件各自之后，同一凭据被拒绝。证据：[`TestWorkflowNodeTurnLive`](https://github.com/ora-space/cloud/blob/4ba9dc5a/v2-server/internal/store/workflow_completion_tool_test.go)：停靠、轮次结束、运行取消后不再有效；[`TestCancellingARunWithdrawsTheTool`](https://github.com/ora-space/cloud/blob/4ba9dc5a/v2-server/internal/workflowhost/agent_completion_test.go)、[`TestAnAgentWithoutHTTPMCPFailsTheNode`](https://github.com/ora-space/cloud/blob/4ba9dc5a/v2-server/internal/workflowhost/agent_completion_test.go)：取消与 Agent 丢失之后端点拒绝。缺口：五种事件没有逐一测试，没有 `EndSession` 与 `StopUnconfirmed` | Partial |
| 吊销后重启仍拒绝 | Controller 测试：execution 结束后重启 Controller，同一凭据仍被拒绝。证据：[`TestAReportSurvivesARestartButItsCredentialsDoNot`](https://github.com/ora-space/cloud/blob/4ba9dc5a/v2-server/internal/workflowhost/agent_completion_test.go)：重启后旧 token 被拒绝。缺口：没有用重启前的端点 URL 再调用，也没有先结束 execution 再重启 | Partial |
| 自动接受不放大跨 execution 或跨 attempt 的调用 | 宿主 Server 测试：开启自动接受的节点收到越权调用时不完成。证据：[`TestAReportComesOnlyFromTheNodesOwnLiveTurn`](https://github.com/ora-space/cloud/blob/4ba9dc5a/v2-server/internal/workflowhost/agent_completion_test.go)：开启自动接受的两个节点互相越权调用都不留信号。缺口：只在首轮，首轮本来不自动接受 | Partial |

### 决策依据

D5、D6“凭据只在持久 execution 记录处于非终态时有效”与“attempt 级授权”，不变量 9；完成信号 D1、D2。
