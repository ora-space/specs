---
status: proposed
---

# Cloud 定时任务首期通过无头 Prompt 单会话复用 IssueRun / Thread

Cloud Scheduled Task 的首期执行目标与 Desktop local 首期一致：只能执行随 task revision 冻结的 Prompt。到期后，
Cloud 不新造第二条 Agent runtime，也不调用 Server Workflow；而是通过一个受控的 application use case 复用已有
IssueRun / Thread、一次性 Workspace、结果记录和释放链路。

本文是 Desktop Scheduled Task 根决策中 Cloud 首期的后续 ADR，状态为 `proposed`。它只定义无头单会话的执行入口和
权限边界；`ScheduledTask`、`scheduledAt` 与 `(taskId + scheduledAt)` 的 occurrence 身份及幂等接纳语义仍以根决策为准。
已发布 Workflow 与 `executorSlot` 映射属于 #67 之后的第二阶段，不在本文范围内。

## 决策

### D1：到期 occurrence 只可请求一个无头 Prompt 会话

Cloud scheduler 在其持久化的 Scheduled Task occurrence 已按根决策接纳后，调用 Cloud application use case
`StartScheduledPromptSession`。该 use case 的概念输入至少包含：

- `taskId`、冻结的 `taskRevision` 与 `scheduledAt`；
- 已冻结的 Prompt；
- 目标 Project 引用；
- 本次 Prompt 运行需要的模型配置引用；
- 用于校验无人值守授权的任务授权引用。

它只创建一个新的、一次性的 Agent 会话，并返回可关联到 Scheduled Run 的 IssueRun 与 Thread 身份。相同
`taskId + scheduledAt` 的重放不得再创建第二个会话。该 use case 是 Cloud 内部 application boundary，不要求把
调度器暴露为通用的公开 Agent 启动 API。

### D2：执行只能复用既有 IssueRun / Thread 生命周期

通过准入后，use case 复用既有 IssueRun / Thread 执行链：在目标 Project 的一次性 isolated Workspace 中启动会话，
沿用既有 Thread 记录、会话结束、结果投影、Workspace 释放和失败收尾。调度器不得直接向 Node 发会话命令，
不得以浏览器计时器、mock runtime 或仅有成功状态的占位记录冒充运行成功。

本文不改变用户从 Issue 触发 Agent 的现有路径，也不改变其续接 Revision 的语义。定时任务的无头调用只允许新建单会话；
不会向已有 Thread 追加用户轮次，也不续接前一 Scheduled Run 的 Revision。

### D3：每次保存和触发都重新校验最小授权边界

保存 Cloud 任务和每次到期触发都必须检查：

1. 目标 Project 仍可用；
2. 任务引用的模型配置仍可用；
3. 创建者对这次无人值守执行的授权仍有效。

任一检查失败时，本次 occurrence 明确失败，不借用其他用户的模型配置，不回退到 local，也不排队等待人工补全。
失败原因须能映射到 Scheduled Run 历史，至少区分 Project、模型和无人值守授权不可用。

Cloud 的无人值守会话在沙盒内全自动运行，不访问用户本机凭据；它不继承 Desktop 的
`full_auto_in_worktree` 策略，也不以该策略作为 Cloud 准入凭据。Cloud 具体的 sandbox 能力、凭据来源与网络边界
必须由 Cloud 侧实现和授权契约单独验证。

### D4：首期明确排除 Workflow 和执行器槽位

首期不得要求或读取已发布 Workflow snapshot、Start schema、`executorSlot` 或个人执行器槽位映射；也不得以缺少这些
第二阶段资源为由把首期 Prompt 伪装为成功或改派给其他执行者。

未来支持“Cloud 定时触发已发布 Workflow”时，另立后续 ADR：等待 #67 的 Workflow server 执行链定型，并按任务所有者的
个人槽位映射解析执行器；所需槽位缺失时，该 occurrence 失败并通知所有者。

## 不变量

1. Cloud 首期每个已接纳 occurrence 最多创建一个新的无头 Prompt 会话。
2. 调度器只经 `StartScheduledPromptSession` 调用已有 IssueRun / Thread application use case，不直接驱动 Node。
3. Project、模型或无人值守授权在保存或触发时无效，均不得启动会话、不得回退 local、不得借用其他用户配置。
4. 首期不接入 Workflow 或 `executorSlot`；第二阶段能力不得反向改变首期 Prompt 的语义。
5. 无头 Cloud 会话运行在 Cloud sandbox 内，不访问用户本机凭据。

## 为什么不是这些替代方案

| 方案 | 取舍 |
| --- | --- |
| 调度器直接调用 Node 或模型 | 少一层调用，但绕开既有会话、结果与释放事实，无法可靠审计或收尾。 |
| 首期复用 Server Workflow | 可复用未来图执行，但把 Prompt 首期阻塞在 #67 和执行器槽位契约上。 |
| 复用已有 Thread 或前一 Revision | 看似连续，但会混淆独立 occurrence 的输入、取消和结果归属。 |
| 授权失效时回退 local 或借用他人模型 | 提高表面成功率，却改变用户选择、授权边界和费用归属。 |

## 本决策未解决的问题

- Scheduled Task 与 IssueRun 的持久化关联、是否需要用户可见的 Issue，以及公开读取 API 的具体形状。
- Cloud Scheduled Task 的创建者、费用与额度归属、配额、通知、取消和历史保留策略。
- Cloud sandbox 的具体能力、模型凭据来源、网络访问和无人值守授权的撤销/审计模型。
- scheduler 的存储、抢占、重试、分区与灾难恢复实现；它们不得改变根决策的 occurrence 幂等语义。

## 核心测试用例

在本文状态变为 `approved` 后，至少覆盖：同一 `taskId + scheduledAt` 重放只创建一个会话；有效 Project、模型与
无人值守授权时通过既有 IssueRun / Thread 链完成一次会话并收尾；三类授权校验分别失败时不启动会话且不回退 local；
调度路径不读取 Workflow 或 `executorSlot`；无头会话不追加到已有 Thread，也不续接前一 Scheduled Run 的 Revision；
重放和失败收尾不遗留运行 Workspace。

## 落地顺序

1. 在 Cloud 实现 `StartScheduledPromptSession` application use case 与 Scheduled Run/IssueRun/Thread 的可审计关联。
2. 在保存和触发两处落实 D3 的 Project、模型与无人值守授权复查；失败写入明确的运行事实。
3. 通过现有 IssueRun / Thread 集成测试验证真实一次性 Workspace、会话、结果记录与释放链；不以 mock runtime 代替。
4. Cloud Scheduled Task 的公开 API、UI 和通知仅在上述真实链路验证后开放。
