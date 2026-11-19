# Controller 发送前镜像与按能力派发 Session MCP 选择

依据[显式 Session MCP 决策](../../../decisions/node/agent-runtime/20261114-explicit-session-mcp-in-agent-session-spec.md)
D1 的 Controller 义务与 D4 的能力门禁。这些规则只能由发送 `start_agent_session` 的一方（Controller 侧，
在 Cloud v2 中就是 Cloud 进程）验证，Node 侧义务见[显式 Session MCP 用例](../../node/agent-runtime/session-mcp.md)。
正式使用方的接入尚未交付，下表证据一律为 `Missing`，随它的交付登记直接测试与固定提交。Cloud v2 的
`v2-server` 已按同一组规则实现选择校验（`v2-server/internal/execution/mcp.go` 的 `MCPSelection.Validate`）
与派发前能力门禁（`v2-server/internal/agentruntime/mcp.go` 的 `prepareMCP`），但 v2 `phase:1` 是临时实现，
按现行规定不登记为 specs 证据（v2 另要求 `session_mcp_lifecycle`，不使用本 ADR 的冻结 setup）。

## Controller Sends Only Selections The Node Would Accept

### 风险

Controller 用另一套规则构造选择：`settings` 被省略或多了未知字段、排序按本地化规则、版本范围被解析成
某个具体版本，Node 在解码阶段就以协议错误关闭整条控制连接，重连后重发又被关闭；超过 16 个 Server 或
64 KiB 的凭据驻留量进入控制帧；向未声明 `session_mcp` 的旧 Node 发送非空选择，让 Node 拒绝整帧；这些
判断发生在会话已经启动、额度已经用掉之后。

### 前置状态

可派发的 Node 分别声明与不声明 `session_mcp`；工作流显式绑定与租户 MCP 配置已持久化；Controller 在
发送 `start_agent_session` 前构造 `AgentSessionSpec.mcp`。

### 触发

用同一组边界样例构造选择：省略与空列表、恰好与超过 16 个 Server、键/标点/转义分别与合计恰好及超过
64 KiB、乱序、重复、非规范 ID、`v1.2.0` 与非完整 semver、缺失 `settings`、非标量 Setting 值；向只声明
`agent_session` 的 Node 派发非空选择。

### 必须成立

- 发送前按 D1 逐项镜像客户端规则：最多 **16** 个 Server；所有 `settings` 的紧凑 JSON UTF-8 长度之和
  最多 **64 KiB**（含键、标点与转义）；`plugin_id` 按原始 UTF-8 字节序严格递增、无重复且通过严格 ID
  解析；`plugin_version` 是精确 canonical 版本字符串；`settings` 必填，没有覆盖值时是空对象。
- 两项上限独立于控制帧上限；非法或超限选择是 spec 校验错误，不启动 Agent、不发送该 `start_agent_session`。
- 只向同时声明 `session_mcp` 与 `agent_session` 的 Node 发送非空选择；空选择省略协议字段，不要求该能力。
- 同一组边界样例在 Controller 与 Node 上得到相同的接受或拒绝结论。

### 禁止结果

- Controller 自行补排序、去重、规范化 ID 或补默认值后受理；超限选择被截断或分批发送。
- 非法选择在会话启动后才被拒绝，或向未声明能力的 Node 发送非空选择。
- 同一输入在 Controller 与 Node 上的判断不同。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 发送前镜像 D1 客户端规则 | Controller 校验测试遍历边界样例：16 项与 64 KiB 的闭区间、字节序与严格 ID、canonical 版本、`settings` 必填；非法或超限选择在发送前被拒绝，不启动 Agent、不发送该 `start_agent_session` | Missing |
| 按能力派发 | Controller 派发测试：不声明 `session_mcp` 的 Node 收不到非空选择（或以有名字的原因拒绝派发），空选择不要求该能力 | Missing |

### 决策依据

D1、D4 失败表第一行与能力门禁，不变量 1、2；Node 侧对应义务与证据见[显式 Session MCP 用例](../../node/agent-runtime/session-mcp.md)。
