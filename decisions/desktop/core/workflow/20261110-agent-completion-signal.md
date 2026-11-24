---
status: implemented
---

# 交互式节点：agent 显式完成信号驱动可完成，人工确认后推进

交互式 Agent 节点目前只要本轮以 `EndTurn` 结束且 blockers 为空就视为 ready；但 agent 可能用 `EndTurn` 结束一个「向用户提问」的回合，界面于是把提问当成完成。本决策给交互式 agent 节点增加一个由 host 提供的显式完成信号：workflow 节点会话额外获得一个只在该会话可用的完成工具，agent 认为本步骤完成时调用它；host 把「当前 turn 的完成请求」绑定到发起它的 attempt，只有该 attempt 结算出的当前 turn 有被接受的请求时节点才可完成，人仍在步骤卡片上确认（或按节点配置自动接受）。没有信号时 `EndTurn` 不再让节点 ready；首轮 gate、异常停靠与 `force` 的既有语义继续适用。

本决策处于 `implemented` 状态，由 [ora-space/desktop#767](https://github.com/ora-space/desktop/pull/767) 实现；验证证据登记在[agent completion signal 用例](../../../../test-cases/desktop/core/workflow/agent-completion-signal.md)。它只定义 host 侧契约与行为，不定义 agent 插件如何实现工具调用；`Loop`/`Iteration` 区域、非交互节点、Cloud 与既有 workflow 包格式不在范围内。

## 继承与修改

继承[交互式节点完成契约](20261021-interactive-node-completion.md)（下称 #385 ADR）、[首轮确认与异常停靠](20261102-first-turn-confirmation.md)、[交互式节点产出物声明](20261023-interactive-deliverable-contract.md)与[工作流显式选择 MCP](../agent/session/mcp/20260912-workflow-explicit-selection.md)（下称 MCP ADR）。

| 前序约定 | 处理 | 当前规则 |
| --- | --- | --- |
| MCP ADR D1/D4：冻结的 `Explicit` 集合是插件 MCP 的完整白名单，空集合不注入任何插件 MCP | 修改边界，不修改白名单 | `Explicit` 仍只描述插件 MCP；host 完成工具是独立的宿主能力，不进入该集合，也不改变空集合的含义 |
| #385 D1：只有本轮 `EndTurn` 且无 blocker 才能完成 | 修改 | 对 `completionSignal: agent` 的节点，还要求当前 turn 有被接受的 agent 请求并完成人工确认 |
| #385 D5：完成校验按状态 → turn key → digest → blockers | 扩展 | 增加「当前 turn 的 agent 请求 / 确认」校验与 `confirmAgentCompletion`；`force` 规则按 D3 |
| 首轮 ADR D1：自动首轮写 `turn_unconfirmed` | 修改 | 首轮 gate 仍只在没有更精确 stop blocker 时加 `turn_unconfirmed`；之后再叠加信号 blocker，正常等待信号/确认时 `parkKind = awaiting_user`；有信号的首轮要同时确认两个 gate |
| 首轮 ADR D2：`force` 与 `confirmInitialTurn` 不能同时为 true | 修改 | 首轮正常结束且无信号时允许两者同时出现：`confirmInitialTurn` 解除 `turn_unconfirmed`，`force` 覆盖信号 blocker；没有可覆盖项时仍返回 `invalid_request` |
| 首轮 ADR D6：`#427` 定义 agent 请求的 requester、权限、幂等与审计 | 落实 | 本决策定义宿主能力、attempt 绑定、确认与审计；不复用 `confirmInitialTurn` 冒充 agent 请求 |
| #728 方向 2：只有 agent 显式完成信号能让回合变成「可完成」 | 采纳 | 信号缺失时节点保持「等待参与」；`force` 是显式人工兜底，见 D3 |

## 决策

### D1：完成工具是独立的宿主能力，不修改插件 MCP 白名单

workflow 执行器为每个 interactive agent 节点会话额外注入一个 host 提供的 MCP server（本地 loopback HTTP，带单 attempt 的 attempt token），暴露：

```text
ora_complete_workflow_step({ "attemptToken": "..." })
  -> { "accepted": true, "signalId": "..." }
```

- 每个 prompt attempt 在发送 prompt 前由 host 生成仅在本次 attempt 有效的 `completionToken`，随步骤边界注入 prompt（不写入转写），并要求 agent 调用时原样传入；token 绑定 `(session, attemptId)`，是发起 attempt 的可信身份，接收端不靠查询「当前活动 attempt」补认。

- 它**不是插件 MCP**：不进入冻结图的 `mcps`、不进入 `SessionMcpSelection::Explicit`、不出现在插件目录，也不改变「空 `Explicit` 表示不注入任何插件 MCP」的语义。MCP ADR 的白名单继续只约束插件 MCP；本决策在 ACP 发送边界追加一个 host-owned 能力。
- **启用条件**：节点是 interactive agent、冻结图的 `completionSignal: agent`、且节点处于 `Pending` 或 `Running`。`completionSignal: end_turn` 的遗留节点、非交互节点和普通对话不注入。
- **授权来源**：节点冻结图的完成策略 + host 签发的单 attempt attempt token；用户 MCP 配置不能添加或移除它，插件也不能声明它。
- **创建/恢复/刷新**：`session/new` 与 `session/load` 时按「节点绑定 + 冻结图策略」重新推导并注入；绑定缺失或损坏时不注入（fail closed），不扩大权限。Agent 替换或 provider 重建时签发新 token、旧 token 失效。
  > 历史决策：经 Node 承载的 Cloud v2 例外：Node 在同一 execution 内重启 Agent（插件刷新）时，已排队的一轮沿用为它签发的 token，不重签，见[Cloud 经 Node 宿主 MCP 提供完成工具](../../../cloud/workflow/20261124-host-completion-tool-for-node-hosted-interactive-nodes.md)（`approved`）D3。换到新的 execution 时，以及 desktop 直接承载的会话，仍按本条签发新 token。
- **撤销**：节点离开 `Pending`/`Running`（完成、失败、取消）、运行结束或 Session 停止时作废 token；之后到达的调用返回 `not_awaiting`。OpenCode 的进程级 MCP 注册表可能仍让工具在进程内可见，这是 provider 一致性缺口；host server 只认 token 与当前绑定，不假设可见性等于授权。
- 工具不读写用户文件、不执行命令，只记录完成意图；是否推进仍由完成门禁决定。

### D2：请求绑定到发起它的 attempt，park 只接受同一 attempt

- host 为每个 prompt attempt 在发送 prompt 前分配 `attemptId` 与 `completionToken`；token 只在这次 attempt 有效，并随步骤边界交给 agent。工具调用必须携带 `attemptToken`。
- MCP server 按 token 解析出 `(sessionId, attemptId)`，再校验 session 绑定与节点 `Running` 状态；不接受无 token、旧 token 或与绑定不一致的调用。发起身份来自 token，接收时不再查询「当前活动 attempt」。
- 接受的请求持久化为 `(nodeRunId, attemptId, sessionId, requestedAt, signalId)`；token 在 attempt 结束（结算、失败、取消、超时）时失效，之后到达的调用返回 `stale_attempt`。
- park 时用刚结算的 attempt 绑定快照的 `turnKey`；只有 `request.attemptId` 与该 attempt 一致才算当前 turn 的有效信号，其它请求丢弃。下一轮或用户继续对话产生新 attempt 和新 token，旧请求不能复用。
- 同一 attempt 的重复调用幂等，返回同一个 `signalId`；attempt 失败、崩溃、超时或取消时丢弃未绑定的请求，不写入完成快照。

### D3：缺信号与待确认是两个状态，确认绑定到用户所见的 turn

快照增加：

```json
"agentSignal": { "signalId": "...", "turnKey": { "userSeq": 3, "endedSeq": 4 }, "requestedAt": "..." }
```

- 快照同时记录 `completionPolicy: "agent" | "end_turn"`，由节点冻结图在 park 时写入；投影按这个字段判断新语义，不能靠 `agentSignal` 是否存在判断 legacy。
- 首轮组合：先按首轮 ADR 的规则判定精确 stop blocker。只有本轮正常结束（没有 `turn_stopped`、`turn_truncated`、`turn_refused`、`turn_unknown`、`structured_invalid`、`output_missing` 等精确 blocker）时，才加 `turn_unconfirmed`，然后再按策略加 `agent_signal_pending`（无信号）或 `agent_confirmation_pending`（有信号未确认）；正常等待信号/确认时 `parkKind = awaiting_user`。有精确 blocker 的停靠（包括首轮用户主动停止的 `turn_stopped` + `user_stopped`）优先按原有规则处理，不叠加信号 blocker，卡片仍可继续或 force。

门禁按两个独立 blocker 区分：

| 状态 | blocker | 普通完成 | `confirmAgentCompletion` |
| --- | --- | --- | --- |
| 当前 turn 没有有效信号 | `agent_signal_pending` | 拒绝 | 拒绝（没有可确认的信号） |
| 有有效信号、尚未确认 | `agent_confirmation_pending` | 拒绝 | 接受（同时解除该 blocker） |
| 有有效信号且已确认 | 无 | 正常完成 | 幂等无操作 |

- `confirmAgentCompletion` 是对象 `{ "signalId": "..." }`，不是 bool；后端校验 `signalId` 与快照一致、`turnKey` 与用户所见一致。不匹配返回 `WorkflowNodeSignalChanged`（前端刷新运行后重试），旧确认不能落到下一轮信号上。
- 首轮同时有 `turn_unconfirmed` 与 agent 信号时，确认请求必须同时带 `confirmInitialTurn: true` 和 `confirmAgentCompletion`；只带一个会被另一个 blocker 拒绝。
- `force` 可以覆盖 `agent_signal_pending` / `agent_confirmation_pending` 并写强制审计，但单独使用时不能覆盖 `turn_unconfirmed`：首轮正常结束且无信号时只带 `force` 仍返回 `WorkflowNodeIncomplete`，列出 `turn_unconfirmed`，不推进、不写审计。
- 例外：首轮正常结束且无信号时，允许同一请求同时带 `confirmInitialTurn: true` 和 `force: true`——confirm 解除 `turn_unconfirmed`，force 覆盖 `agent_signal_pending` 及 `prepare` 复算出的 blockers，按 D6 写 `requester: "human"` 与完整审计；由 confirm 解除的 `turn_unconfirmed` 不进入 forced blockers。该组合只在快照确有 `turn_unconfirmed` 且至少有一个非 `turn_unconfirmed` 的 blocker 可被 force 覆盖时成立，否则返回 `invalid_request`。`confirmInitialTurn` 必须由人显式发送，首轮必须人工确认这一原则不变。
- `ready` 仍由「blockers 为空且非 provisional」推导；信号与确认通过 blocker 表达，不另造 ready 旁路。

### D4：自动接受在同一轮结算后走同一完成门禁

- 工具在 `Running` 期间只记录意图；自动接受**不在 turn 进行中完成节点**，而是等本轮 park 后由 host 触发一次内部完成请求（携带 `confirmAgentCompletion`），走与人工确认完全相同的门禁。
- 自动接受只在节点配置 `autoAcceptAgentCompletion: true` 且本轮已正常结算、无 `turn_unconfirmed`、无异常 blocker 时生效；`confirmation.by = "auto"`。
- 首轮仍保留人工 `confirmInitialTurn`：首轮不自动接受；人工确认时一次完成。
- `parkKind = abnormal` 时自动接受不执行；用户仍需继续或 force。
- 自动接受失败（例如门禁拒绝）只记录日志并保持 `Pending`，不重试，也不写 `completion_forced`。

### D5：卡片动作按「异常 → 首轮 → 信号」的顺序判定

| 状态 | 徽标 | 主操作 | 次操作 |
| --- | --- | --- | --- |
| `parkKind = abnormal` | 异常停靠 | 继续当前步骤 | 仍然完成（force，二次确认） |
| provisional | 等待参与 | 继续当前步骤 | 无（等修复） |
| 首轮正常结束且无信号（`turn_unconfirmed` + `agent_signal_pending`） | 等待参与 | 继续当前步骤 | 仍然完成（force，二次确认，同时带 `confirmInitialTurn`） |
| 首轮用户主动停止（`turn_stopped` + `user_stopped`，无 `turn_unconfirmed`） | 等待参与 | 继续当前步骤 | 仍然完成（force，二次确认） |
| 首轮有信号 | 等待确认 | 确认并完成（同时带两个确认字段） | 继续当前步骤 |
| 非首轮无信号 | 等待参与 | 继续当前步骤 | 仍然完成（force，二次确认） |
| 非首轮有信号、待确认 | 等待确认 | 确认并完成 | 继续当前步骤 |
| `completionSignal: end_turn` legacy | 按 #385/首轮 ADR | 原行为 | 原行为 |

- 展示动作与后端可执行条件一致：首轮正常结束且无信号时，force 与 `confirmInitialTurn` 一起发送（`turn_unconfirmed` 仍必须人工确认）；首轮用户主动停止按既有规则可 force；有信号时主操作是确认，不是普通完成。

### D6：requester 由服务端按可信绑定生成，请求不携带

- `CompleteWorkflowNodeRequest.requester` 不再作为客户端输入：客户端不能声称 `Agent`；实现时移除该字段，或只接受 `Human` 并忽略。
- 服务端按路径写 payload：
  - agent 信号 + 人工确认：`requester: "agent"`，`confirmation: { "by": "human", "at": ... }`；
  - agent 信号 + 自动接受：`requester: "agent"`，`confirmation: { "by": "auto", "at": ... }`；
  - force 且无信号：`requester: "human"`，`completion_forced: { "by": "human", ... }`；
  - 首轮正常结束且无信号、`confirmInitialTurn` + `force` 同时出现：`requester: "human"`，`confirmation: null`；`turn_unconfirmed` 由 confirm 解除，不写进 forced blockers；`completion_forced` 记录实际覆盖的 `agent_signal_pending` 与 `prepare` 复算的 blockers、`stop_cause`；
  - force 且已有信号、用户选择强制推进：`requester: "agent"`，保留 `agentSignal`，`confirmation: null`；`completion_forced` 记录实际覆盖的完整有效 blocker 集合（快照 blockers + `prepare` 复算的输出契约/文件 blockers）与已有 `stop_cause`，不能只写 `agent_confirmation_pending`；`confirmation: null` 明确表示用户没有点确认，而是强制推进；
  - legacy/普通完成：`requester: "human"`。
- `agentSignal` 保存信号身份（`signalId`、`turnKey`、`requestedAt`）；`completion_forced` 沿用首轮 ADR D3 的形状并增加 `by`。
- attempt token 绑定发起时的 session 与 attempt；`requester: "agent"` 只由 token 对应的已接受 attempt 推导，客户端不可构造。

### D7：兼容与迁移

- 升级前冻结的图没有 `completionSignal` 字段：按 `end_turn` 遗留语义执行，不注入完成工具，不追溯改变已停靠节点。
- 升级后新建或重新保存的交互节点默认 `agent`；编辑器明示，允许显式改回 `end_turn`。
  > 历史决策：经 Node 承载的 Controller（如 Cloud v2）例外：在它的产品版本接入完成工具之前，编辑器给新交互节点默认写 `end_turn`，显式 `agent` 按 `end_turn` 执行并提示暂不生效，恢复条件见[经 Node 承载的交互节点默认 `end_turn`](../../../cloud/workflow/20261119-end-turn-default-until-host-completion-tool.md)。desktop 直接承载的会话仍按本条默认 `agent`。
- 旧快照没有 `completionPolicy`：按 legacy 读取；`agent` 策略的新快照即使缺 `agentSignal` 也按 `agent_signal_pending` 处理，不能回退成 legacy。`prepare` 按节点冻结策略重新物化并写入 `completionPolicy`；升级前已停靠节点在下次 park/prepare 前保持 legacy。
- host 完成工具不影响插件 MCP 白名单；`Explicit` 为空、插件缺失、刷新失败等 MCP 错误仍按 MCP ADR 失败，不会因为 host 工具而放宽。
- 不支持工具调用的 agent：节点不会 ready；用户可以继续对话，首轮正常结束且无信号时可用 `confirmInitialTurn` + `force` 二次确认推进，非首轮可用 force 兜底；不在本决策内做能力探测或自动降级。

## 不变量

1. 快照 `completionPolicy: agent` 时，当前 turn 没有与结算 attempt 一致的有效信号则 `ready = false`，普通完成返回 `agent_signal_pending`；缺 `completionPolicy` 的旧快照按 legacy。
2. 有有效信号但未确认时，普通完成返回 `agent_confirmation_pending`；`confirmAgentCompletion` 必须携带匹配的 `signalId` 并绑定用户所见的 turn。
3. 信号只对 owning session、token 绑定的 attempt 和该 attempt 结算出的当前 turn 生效；旧 attempt、其他 session、非 interactive 节点一律拒绝，不改变状态。
4. 信号与确认都不绕过状态/claim、turn key、`outputDigest`、输出契约或文件 blockers。
5. 人工确认是普通完成，不写 `completion_forced`；force 覆盖 blocker 时写审计，记录实际覆盖的完整有效 blocker 集合与 `stop_cause`。force 不能单独覆盖 `turn_unconfirmed`；首轮正常结束且无信号时允许 `confirmInitialTurn` + `force` 同请求，confirm 解除 `turn_unconfirmed`，force 覆盖信号 blocker。
6. 首轮 gate 与信号只在正常结束时叠加：仍有 `turn_unconfirmed` 的首轮无信号主操作为继续，次操作允许 `confirmInitialTurn` + `force`；首轮用户主动停止等有精确 blocker 的情况按原有规则（继续或 force），不叠加信号 blocker。
7. 自动接受只在轮次正常结算后走同一门禁，首轮和异常停靠不自动接受，且不写 `completion_forced`。
8. host 完成工具是独立宿主能力，不进入 `SessionMcpSelection`，不改变插件 MCP 白名单；`completionSignal: end_turn` 和升级前冻结图不注入。
9. `requester` 由服务端根据信号/确认/force 路径生成，客户端不能自称 `Agent`。
10. 新语义以快照 `completionPolicy` 判定；`end_turn` 节点与缺该字段的旧快照保持 legacy。

## 为什么不是这些替代方案

| 方案 | 取舍 |
| --- | --- |
| structured output 里的完成字段 | 不覆盖 text 契约节点；用户 schema 可能 `additionalProperties: false`，加保留字段会破坏校验；无法在 turn 中途表达 |
| assistant 文本中的完成标记 | 需要解析文本并剥离标记，模型可能漏写或误写；与「不解析文本判断意图」的既有边界冲突 |
| 自定义 ACP session update | 需要每个 agent 插件实现新的协议消息，存量 agent 不支持；无法跨 agent |
| CLI 命令 | agent 需要拿到 run/node 上下文、凭据和可执行路径，权限面最大；沙盒与出网边界复杂 |
| 继续只靠 `EndTurn`，用文案提示 | 不解决「提问回合被当成完成」的误判，只是把问题留给用户点击 |
| 把完成工具并入节点 `mcps` 白名单 | 会让用户以为它是可配置插件 MCP，破坏「空集合不注入任何插件 MCP」和「安装不等于授权」；它应由节点完成策略授权 |

## 落地结果

1. **契约**：`NodeCompletionRequester::Agent`、`confirmAgentCompletion { signalId }`、`agent_signal_pending` / `agent_confirmation_pending` 与 `WorkflowNodeSignalChanged` 已落地；客户端不再提供 `requester`，服务端按信号与 force 路径派生。
2. **host 能力与门禁**：完成 MCP server、per-attempt token、park 信号绑定、双 blocker 门禁、确认校验与自动接受已接线。`agent_signal_pending` / `agent_confirmation_pending` 在 park 时物化进快照 blockers，卡片、投影与完成门禁读同一事实；跨回合的旧 `agentSignal` / `confirmation` 会被清除。
3. **前端**：新建或重新保存的交互节点写入 `completionSignal: "agent"` 与 `schemaVersion: 4`，旧冻结图仍按 `end_turn` 读取；`completion.files`、`completionSignal` 与 `autoAcceptAgentCompletion` 共享 version-4 边界，删空文件不会把仍带策略的节点降回 3，关闭交互模式会清理这两个策略字段；步骤卡片按 D5 矩阵展示，首轮有信号时主操作同时发送两个确认字段，首轮无信号时次操作发送 `confirmInitialTurn` + `force`；自动接受只在已结算的人类回合触发。
   > 历史决策：本条的默认值不适用于经 Node 承载的 Controller（如 Cloud v2）：在它接入完成工具之前，新交互节点写 `completionSignal: "end_turn"`，见[经 Node 承载的交互节点默认 `end_turn`](../../../cloud/workflow/20261119-end-turn-default-until-host-completion-tool.md)。desktop 的前端仍按本条写 `agent`。
4. **恢复与运行边界**：Session 持久化 `host_capabilities`，重启后按冻结节点策略恢复完成工具；只有交互式 agent-policy 节点签发 token 并注入完成指令，非交互节点即使保留策略字段也不会拿到 token 或工具；attempt 结束、节点完成与运行取消都会撤销 token；完成 server 限制 64 KiB 请求体与 10 秒连接总时限。
5. **验证**：核心用例除保留的一处 `Partial` 外均为 `Covered`，直接测试见[agent completion signal 用例](../../../../test-cases/desktop/core/workflow/agent-completion-signal.md)。

## 风险、未决问题与验收

- **宿主能力可见性**：OpenCode 共享进程可能让完成工具在撤销后仍可见；host server 只认 token 与绑定，越权调用必须被拒绝。
- **agent 不调用工具**：没有信号时节点不可普通完成；非首轮可用 force 兜底；首轮正常结束且无信号时可用 `confirmInitialTurn` + `force`（人工确认后强制推进），用户主动停止等其它首轮情况按既有规则继续或 force。是否按 agent 能力自动降级留给后续决策。
- **attempt 时序**：工具调用可能早于/晚于 turn 结束；park 必须绑定结算 attempt，晚到请求丢弃。
- **确认绑定**：只用 bool 无法防止旧确认落到新信号上，所以用 `signalId` + `turnKey` 双重校验。
- **自动接受**：只在轮次结算后触发，首轮/异常不自动，避免绕过人工确认或异常门禁。

验收：

- 工具调用绑定发起 attempt；同一 turn 的 park 快照带 `agentSignal`，卡片显示「确认并完成」；确认后完成，payload 为 `requester: agent` + `confirmation.by: human`，无 `completion_forced`。
- 无信号普通完成返回 `agent_signal_pending`；有信号未确认返回 `agent_confirmation_pending`；`confirmInitialTurn` 不能替代 agent 确认。
- 首轮有信号时一次确认同时带 `confirmInitialTurn` 与 `confirmAgentCompletion`；首轮正常结束且无信号时主操作继续，次操作 `confirmInitialTurn` + `force` 可完成（`requester: human`，`completion_forced` 记录 `agent_signal_pending` 与 `stop_cause`），单独 force 仍被拒。
- 首轮用户主动停止（`turn_stopped` + `user_stopped`）：卡片提供继续或 force，不叠加信号 blocker，也不要求 `turn_unconfirmed`。
- 重复调用幂等；旧 attempt、其他 session、非 interactive 节点、park 后调用都被拒绝且不改变状态。
- 自动接受只在正常结算且非首轮时生效，走同一门禁，记录 `confirmation.by: auto`。
- force 且已有信号时，payload 为 `requester: agent`、保留 `agentSignal`、`confirmation: null`；`completion_forced` 记录实际覆盖的完整 blocker 集合与 `stop_cause`。
- 新策略缺信号的快照仍投影为 `agent_signal_pending`；缺 `completionPolicy` 的旧快照走 legacy，不互相回退。
- `completionSignal: end_turn` 的节点、升级前冻结图与旧快照行为不变；空 `Explicit` 仍不注入任何插件 MCP，host 完成工具不改变该语义。
