---
status: proposed
---

# Controller–Node Protocol：统一消息帧与可替换 Transport

Controller 与 Node 使用一套与部署位置无关的协议语义。协议定义统一消息帧、身份、执行、事件、
结果、确认和恢复；底层 Transport 可根据本地、SSH、集群或远端网络替换。Controller 主动管理
Node 的创建和停止；Node 启动后可以通过本机 IPC、SSH 通道或网络连接建立双向会话。对于普通
容器服务，当前推荐由 Controller 主动拨号 Node 的私有服务 endpoint。

在 Cloud 部署中，三者的职责关系固定为：**API Server 是权威状态源，Controller 是 reconcile
worker，Node 是执行环境。** Node 不成为业务状态源，也不直接与 API Server 交换业务数据；Controller
根据 API Server 提供的已授权状态驱动 Node，并将 Node 事实回写给 API Server。

本决策处于 `proposed` 状态，覆盖第一个 Worktree 闭环。它不决定具体编解码格式、集群编排器、
SSH 凭据管理或 Node 能力实现。

## 背景与约束

Desktop、SSH 远端主机和集群沙盒都需要执行同样的 Node 能力。若按传输分别设计协议，执行、事件、
重试和恢复语义会分裂；若把 Node 限制为标准流，则独立 Node 和集群 Node 无法自然接入。

第一版目标是让目标 Node 在已有 Main Workspace 上执行 task worktree 创建和删除，并验证跨进程、
断线、重传和结果未知的处理。

## 决策

### D1：协议层与 Transport 层分离

```text
Controller–Node Protocol
        │
        └── FramedConnection
                │
                ├── LocalIpcTransport
                ├── StdioTransport
                ├── SshTransport
                └── NetworkTransport
```

Protocol 只理解消息和会话，Transport 只负责可靠地发送和接收字节流。Unix Socket、Windows Named
Pipe、SSH stdin/stdout、SSH tunnel 和 TLS 网络连接都必须映射到相同的消息语义。

### D2：统一消息帧

所有字节流都使用明确 framing，例如长度前缀加版本、消息类型、请求／执行身份、事件序号和负载。
一次 `read` 不保证得到一条完整消息；超过最大帧大小、版本不兼容或无法解码时必须明确拒绝。

消息帧至少支持：

```text
protocol_version
message_type
request_id
operation_id
execution_id
sequence
payload
```

具体 JSON、MessagePack、Protobuf 或其他编码格式留待后续决策，不得改变上述身份和恢复语义。

这些字段按消息语义使用，不要求握手等无业务操作的消息携带执行身份。身份分工沿用
[Client–Controller Contract](../../desktop/protocol/0-client-controller-contract.md)：`request_id`
关联原 Client Command，`operation_id` 关联业务操作，`execution_id` 标识一次 Node 执行尝试。
`operation_id` 与 `execution_id` 不是别名；执行事件按 `execution_id` 与该执行内的单调序号识别。

首个 Worktree 闭环中，每个创建或删除操作只建立一次执行尝试。Controller 在派发前将操作、执行、
目标 Node 和输入的关联保存到权威存储：Desktop 使用本地持久化，Cloud 使用 API Server。
网络重传、重新领取协调工作及 Controller／Node 重启均保留原执行身份。结果未知时先对账，不能
新建操作或执行身份绕过去重；确定旧执行结果后是否允许新的业务重试，不由传输恢复自动决定。

### D3：Controller 主动拉起，Node 建立会话

Controller 通过部署适配器申请或启动 Node：本机可启动子进程，SSH 可远端执行命令，集群可调用
沙盒编排 API。启动参数包含 Node 服务 endpoint、一次性 bootstrap credential、Workspace 绑定
和资源限制。

Node 启动后在私有 endpoint 上监听并完成握手，Controller 通过 mTLS 主动建立双向会话。SSH
场景使用 SSH 会话或隧道承载协议；容器服务使用平台 Service、DNS 或 endpoint 发现。对于只能
出站访问或禁止 Controller 入站的网络，保留 Node 出站回连作为兼容连接器；生命周期操作的
发起方不决定协议角色。

### D4：身份、能力和会话恢复

每个会话绑定 `ControllerId`、持久 `NodeId`、本次运行的 `NodeIncarnationId`、协议版本和能力
集合。Node 重启必须生成新的运行实例标识；bootstrap credential 使用、过期或撤销后不可复用。

连接断开时 Controller 暂停向该 Node 派发新任务。重连后双方以 `ExecutionId` 和事件序号进行
恢复对账；Node 重启改变运行实例身份，但不改变已有执行身份或重放事件的身份。
Transport 断开不能直接把执行标记为失败。

Node 在启动外部操作前持久保存执行身份与输入，并保留未确认的事件和结果。Controller 只有在
权威存储持久接管事实、去重身份及可恢复的后续处理责任后，才能确认对应事件：

| 部署 | 允许向 Node 确认的时点 |
|---|---|
| Desktop | Controller 本地持久化成功后 |
| Cloud | API Server 持久化成功并经 Internal Control API 确认后 |

Controller 收到消息或在内存中完成去重都不满足确认条件。API Server 不可用或持久化结果未知时，
事件保持未确认；Node 收到确认后才可清理对应重放记录。清理事件不意味着删除执行去重记录，
后者仍须阻止协议允许的重传重新执行操作。确认丢失时，Node 重放同一事件，Controller 根据权威
存储中的记录重新确认。Cloud 侧的提交恢复遵循
[Internal Control Protocol](../../controller/protocol/0-internal-control-protocol.md)。

持久执行记录不等于外部副作用恰好一次。若 Node 在 Git 操作发生后、结果落盘前崩溃，必须结合
执行记录和实际资源对账；无法确定结果时保留结果未知状态，不盲目重跑。

#### 状态查询、事件重放与确认的职责

`GetExecutionStatus` / `ExecutionStatus` 用于执行对账，回答当前状态及已保留的终态结果。
`ExecutionStatus.node` 标识当前报告者及其运行实例；`Completed` 内层结果保留原始 Node 和运行
实例身份。内外持久 `NodeId` 必须一致，`NodeIncarnationId` 可以因重启不同，不能用当前实例覆盖
历史结果身份。协议收发均拒绝跨 Node 的结果归属冲突。会话／Controller 还须验证报告者匹配会话
绑定的 Node，且执行确实派发给该 Node；内部一致性校验不替代这些依赖会话和派发记录的检查。
状态回复不携带原事件的 `sequence`，不承担事件交付或确认职责。`Completed` 表示执行已完成，
不表示 Controller 已持久接管对应事件。`EventAck` 始终确认原事件的精确
`(execution_id, sequence)`。

Node 在会话恢复后主动驱动未确认事件的重放，保持原事件身份与结果；重放不依赖 Controller
先查询状态或从查询回复推导事件序号。状态查询不会确认事件、停止重放或触发未确认记录清理。
查询回复与重放事件可能以任意顺序到达；Controller 对同一执行结果的处理必须幂等，不能因为
两条路径都携带结果而重复触发业务后续处理。事件仍须满足上述持久接管条件后才能确认。

例如 Node 完成执行并持久保存终态事件后断线：重连查询可以返回 `Completed`，Node 仍重放
原事件；Controller 持久接管后按原事件身份确认。若确认丢失，Node 再次重放，Controller 根据
持久记录重新确认。已确认并清理的事件不因后续状态查询而重新进入重放；查询可返回执行记录
仍保留的结果，也不要求 Controller 为该查询回复再发送确认。

这个选择让执行对账与可靠事件交付各有单一职责。代价是恢复流程必须显式驱动重放，并处理查询
与事件乱序；给查询回复增加原事件身份可以让它也承担交付，但会引入另一条交付与确认路径，
首个闭环不采用该方案。

后续恢复实现需验证：结果发送时断线后仍会重放、查询不会停止未确认事件重放、查询回复与事件
任意先后到达均不重复触发业务处理、确认丢失后可重复确认，以及查询已确认的执行不会重新要求
事件交付。这些是后续 Node / Controller / session 实现的验证义务，协议消息往返测试不构成
恢复保证的证据。

### D5：第一版最小消息集合

Worktree 闭环至少需要：

```text
Controller → Node: EnsureWorktree
Node → Controller: WorktreeReady | WorktreeFailed

Controller → Node: RemoveWorktree
Node → Controller: WorktreeRemoved | WorktreeRemovalFailed
```

消息引用 `operation_id`、`execution_id`、`worktree_id`、`workspace_id`、目标 Node、RepositoryRef、Main Workspace
绑定、base ref、期望分支和路径策略。成功结果包含 Node 身份、运行实例、实际 Node-scoped path、
分支和 base commit。

同一执行身份和参数一致时，已完成的执行返回原结果，执行中的返回当前状态；参数不同则冲突。
首个闭环内同一操作不能通过更换执行身份再次启动。删除已不存在的工作树可以按
`AlreadyAbsent` 幂等成功；结果不明确时进入恢复状态，不能猜测成功或自动创建第二个工作树。

### D6：逻辑通道与流量控制

同一物理连接可以复用三个逻辑通道：

- `Control`：执行、取消、探测和配置；
- `Events`：进度、ACP 事件、状态和结果；
- `Bulk`：文件、日志和产物。

第一版可以仅使用消息类型和优先级复用一条连接。控制和心跳必须优先于大量日志；缓冲达到上限
时必须暂停可暂停生产者或明确失败，不能静默丢失终态。

## 不变量

1. 本地、SSH、集群和远端网络复用同一 Controller–Node Protocol。
2. 同一执行的重传复用原 `operation_id` 和 `execution_id`，不重复启动外部操作。
3. Node 身份由 `NodeId` 和 `NodeIncarnationId` 共同确认，不能使用进程 ID 或路径替代。
4. Node 断线、重启或响应丢失不会让 Controller 无证据地标记成功或再次启动执行。
5. Node 内部日志、进程句柄和绝对路径不成为协议的跨部署语义。
6. Node 只在自己的环境中解析和验证 Workspace、Repository 和路径边界。
7. Node 事件在权威存储持久接管后才能确认；Controller 内存去重不构成持久化确认。
8. 执行身份及输入先持久化再启动；运行实例变化和结果重放不改变已有执行与事件身份。
9. 状态查询与事件确认相互独立；未确认事件在会话恢复后重放，查询与重放乱序不重复触发业务处理。

## 为什么不是其他方案

| 方案 | 否决理由 |
|---|---|
| 本地、SSH、网络各自定义业务协议 | 重复执行、事件和恢复语义，无法保证跨部署一致 |
| 所有 Node 强制使用标准流 | 独立进程、集群和网络 Node 无法自然连接；生命周期与父进程耦合 |
| Controller 直接把本机路径传给 Node | 远端 Node 无法解释该路径，且越过 Node 的路径授权边界 |
| 只使用一次性请求响应 | 无法承载进度、心跳、确认、重放和取消协调 |

## 本决策未解决的问题

- Frame Codec 的具体编码和兼容版本策略；
- 本机 IPC endpoint、TLS、证书和 bootstrap credential 的管理；
- Node 能力声明和能力变更的版本化模型；
- 大文件、终端和产物的独立传输实现；
- Controller 直接拨号 Node 与 Node 回连的连接器选择。
