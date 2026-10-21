---
status: proposed
---

# 工作流节点自动重试与运行恢复契约：首期只实现 agent 节点的自动重试，resume 只定契约并明确不可用

本文是[工作流执行根决策](0-cloud-side-scheduler-owns-workflow-execution.md)（`approved`）的后续决策，回应其
D1 条件①与 D4 的两个延后项。结论：Cloud 首期在 Go 调度器内实现 **agent 节点的自动重试**，策略形状、默认值
和可重试失败类别与 desktop 对齐；重试期间运行保持 `running`，节点状态进入 `retrying`，调度器按数据库时钟
到期唤醒，每次尝试是独立的执行身份。**运行恢复（resume）本期只定语义、接口与权限，不实现**：运行进入终态
时 Workspace 仍按 D7 声明删除，Cloud 也没有每节点 checkpoint；对外接口对任何 resume 请求返回
`409 resume_unavailable`，页面不提供恢复入口。loop、iteration、snapshot switch、subflow 不在本文范围内，
仍各自触发 D1 条件①。

**状态与影响面**：本文为 `proposed`，供 owner 评审；批准前不实现。按 specs 的生命周期，核心用例在本文转为
`approved` 时同批登记（根决策条件③）。涉及 cloud 的发布门禁、调度器、`workflow_runs.node_states`，以及
`node_executions` / `thread_commands` 的尝试维度，和 specs 的本文。**兼容边界**：`node_states` 新增
`retrying` 取值与重试事实；`node_executions` 的 agent_session 唯一性从「每 run 每节点一条」收窄为「每 run
每节点每 attempt 一条」，`thread_commands` 的 end_session 唯一性同步收窄到 attempt；存量行按 attempt 1
解释，不回填内容。运行状态枚举不变，不新增运行状态。**非目标**：不实现 resume，不改变终态 Workspace 释放
策略，不新增检查点或回滚子系统，不改变发布门禁对 `human`、`interactive: true` 的拒绝。本决策不承担数据
迁移之外的迁移义务。

## 继承与修改

| 前序约定 | 本文处理 |
|---|---|
| 根决策 D1 条件①：服务端引入 retry、resume 前必须先重新决策「继续 Go 移植」还是「抽取 Rust 核心」 | **已回答**：owner 在 ora-space/circus#448 选 A（继续 Go；首期只实现 retry，resume 只定契约）。本文是该结论的 specs 落地 |
| 根决策 D1 条件②：无状态调度、单 run 串行、结果提交是比较交换、回调带执行代 | 继承，不变：重试决策与唤醒都由调度器从已提交状态重算，不使用进程内定时器 |
| 根决策 D1 条件④：调度器边界按 desktop 端口形状定义 | 继承，并扩展「节点结果提交」：提交必须带 attempt 身份，旧 attempt 的结果不再是当前节点的结果 |
| 根决策 D2：`agentConfig.retry` 延后，发布门禁拒绝任何非 null 值 | **修改**：合法的 retry 策略发布时放行；非法形状按节点、字段、原因拒绝；其他延后字段的处置不变 |
| 根决策 D4：首期节点失败即运行失败 | **修改**：agent 节点在可重试失败且预算未用尽时进入等待重试，运行保持 `running`；预算用尽或失败不可重试时仍按原规则失败 |
| 根决策 D4：同一 run 内 agent 节点串行，共用 Workspace | 继承，不变：同一节点同一时刻只有一个在途 attempt；自动重试不重置 Workspace |
| 根决策 D4：取消是请求加有界终态 | 继承，并明确：取消优先于重试，等待中的重试不派发新 attempt |
| 根决策 D7：运行终态声明删除 Workspace | 继承，不变：这是 resume 首期不可用的原因之一 |
| [会话在初始回合后结束](20261019-workflow-session-end-after-initial-turn.md) D1/D3：单回合会话、每节点唯一的 end_session 命令 | **修改**：每个 attempt 都是单回合；end_session 命令按 attempt 唯一，而不是按节点在整次运行内唯一 |
| desktop 参考实现 `retry.rs` / `retry_scheduler.rs`（`e73f0af`） | 继承语义：策略形状、默认值、退避上限、可重试的失败类别、等待中的 attempt 仍属在途；**偏离**：等待落成 `retrying` 状态，而不是一条 `Running` 的行；唤醒由调度器重算，而不是进程内定时器 |

## 决策

### D1：策略形状、默认值与发布门禁

`agentConfig.retry` 是 agent 节点唯一的重试配置；其他节点不读取它。

- **缺省即默认策略**：字段不存在或为 `null` 时，按
  `{enabled: true, maxRetries: 2, initialDelaySeconds: 10}` 执行，与 desktop 一致。
- **出现时三个字段全部必填**：`enabled` 是布尔；`maxRetries` 是 0–5 的整数；`initialDelaySeconds` 是
  0–300 的整数。未知键忽略。缺字段、类型不对或越界都在发布门禁按节点、字段、原因拒绝，避免编辑器缺陷
  静默落回作者没看过的策略。
- **退避**：第 n 次重试（从 1 起）等待 `min(initialDelaySeconds × 2^(n-1), 600)` 秒。
- `enabled: false` 或 `maxRetries: 0` 表示不重试；`initialDelaySeconds: 0` 合法，表示下一个调度 tick
  立即重试。
- **发布门禁**接受合法策略，草稿保存保持宽松；非法策略的拒绝原因必须同时给出节点、字段和原因，与 condition
  配置门禁的三要素一致。
- **这是相对现状的行为变化**：现有已发布快照没有 `retry` 字段，将按默认策略重试。选择对齐 desktop 而不是
  Cloud 自行默认关闭，理由是根决策把 desktop 定为参考实现，且策略有界、只覆盖可重试的瞬时失败；实现必须
  在运行视图显示重试次数与下一次尝试时刻，并让编辑器能查看和修改该策略，避免「配置看不见却生效」。
- **interactive Agent 不自动重试**：首期发布门禁拒绝 `interactive: true`；将来 HITL 落地后，等待人类输入
  不是可重试的失败，交互式节点的失败语义由 HITL 决策另定。

### D2：可重试的失败类别

重试只覆盖「会话断了或 agent 的本次产出不可用」这一类瞬时失败，与 desktop
`NodeFailureKind::auto_retry` 的取舍同源。Cloud 的失败码是封闭集合：

| 失败码 | 是否重试 | 理由 |
|---|---|---|
| `agent_failed` | 是 | Node 报告 agent 失败，换一次尝试可能改变结果 |
| `interrupted` | 是 | Node 报告会话被中断或结果未知。desktop 的 `interrupted_by_restart` 不重试，因为它属于启动清扫；Cloud 没有这个清扫，调度器重启也不会中断在途会话，所以 Node 报告的 `interrupted` 是会话失败 |
| `session_turn_deadline_exceeded` | 是 | 回合窗口内既无结果也无成功的 `turnEnded`，属于会话/产出失败（对应 desktop 的 `SessionEndedWithoutStopReason`） |
| `session_end_deadline_exceeded` | 是 | 结束命令在确认窗口内没有被确认，属于会话/传输失败 |
| `node_unavailable` | 否 | Workspace 所在的 Node 已经不可达；在同一个 Workspace 里重试无法把 Node 找回来，只会把一次诚实的失败推迟一个退避窗口 |
| `session_start_deadline_exceeded` | 否 | 会话从未派发成功，属于环境/准入失败 |
| `workspace_unavailable`、`workflow_executor_unconfigured`、`workflow_run_project_required`、`agent_plugin_version_unfrozen` | 否 | 定义、绑定或环境缺口，原样重跑不会改变结果 |
| condition、output、start 的内联失败，图结构错误，取消 | 否 | 不是 agent 会话失败，或已经由取消路径结算 |

不在表内的失败码一律立即失败。实现新增失败码时必须同时决定它的重试归属，不能默认落入「可重试」或
「不可重试」。

### D3：等待中的节点状态与唤醒

重试等待期间运行保持 `running`，节点的 `nodeStates` 条目进入 `retrying`：

```json
{
  "status": "retrying",
  "displayName": "Agent",
  "startedAt": "2026-10-21T12:00:00Z",
  "retry": {
    "attempt": 2,
    "maxAttempts": 3,
    "previousError": "agent_failed",
    "previousFinishedAt": "2026-10-21T12:00:30Z",
    "dueAt": "2026-10-21T12:00:40Z"
  }
}
```

- `attempt` 是下一次派发将运行的尝试编号（首次尝试为 1，第一次重试为 2）；`maxAttempts` 等于
  `1 + maxRetries`，在等待开始时从冻结的策略算出并写入状态，后续 pass 不再重新解析策略。
- `dueAt` 是数据库时钟下的最早开始时刻。`startedAt` 保留刚刚失败的那次尝试的开始时刻；
  `previousFinishedAt` 是它结束的时刻。下一次尝试真正开始时，状态回到 `running`，`startedAt` 改为新尝试的
  开始时刻，`retry` 只保留 `attempt`、`maxAttempts`、`previousError`，`dueAt` 被移除。按尝试重置
  `startedAt` 是必要的：会话启动截止时间与回合窗口读的就是这个时刻，沿用上一次尝试会提前判超时。
- `retrying` 节点视为**在途**：投影不派发它的下游，运行不能结算；它不是失败，也不是空闲。
- **唤醒由调度器重算**：调度器每轮扫描 `running` 运行，读到 `retrying` 节点时比较数据库时钟与 `dueAt`；
  未到期直接返回，到期则按正常会话路径派发下一次尝试。没有进程内定时器，没有需要登记或取消的定时器。
  等待因此是「至少一个退避窗口」：调度 tick 的延迟只会让尝试开始得更晚，不会更早。
- **唤醒是幂等的**：两个副本同时读到到期，只有拿到该运行排他事务权的一方派发；另一方重新从已提交状态
  重算，看到节点已经是 `running`，空操作。
- **等待没有独立截止时间**：到期后会一直保持到期，直到被派发或运行进入终态。上界由策略的单次等待上限
  （600 秒）和重试预算共同给出，不需要再加一层超时。

### D4：attempt 身份、持久化与迟到结果

每次尝试是一个独立的 agent_session 执行，而不是同一个执行被复用。

- `node_executions` 为 workflow 的 agent_session 增加尝试编号，agent_session 的唯一性从「每 run 每节点
  一条」变为「每 run 每节点每 attempt 一条」。存量行全部是第一次尝试，新增列取默认值 1 即可，不需要回填内容。
- `thread_commands` 的 end_session 唯一性同步收窄到 attempt：一个 attempt 的结束命令永远不能被当作另一个
  attempt 的命令复用。现有命令都属于 attempt 1。
- 节点状态记录当前 attempt 编号。调度器读取「这个节点的结果」时按当前 attempt 过滤：旧 attempt 的结果
  保留在执行行上，但**永远不能结算节点**，迟到与重复回调都是无副作用的空操作（继承条件②）。
- **预算持久化**：`attempt` / `maxAttempts` 写进节点状态；调度器重启、换副本、重复 pass 都不重置预算。
  只有后续的 resume 决策可以定义预算重置（desktop 在人工恢复时重置）。
- **自动重试不回滚 Workspace**：第 n+1 次尝试看到第 n 次尝试留下的文件状态，包括它已经做出的改动。这是
  desktop 的行为；恢复文件是 resume 的职责，不是 retry 的职责。
- 每次尝试的会话 spec 都从冻结的图、运行输入和模型绑定重新构建，prompt 不变；尝试编号属于执行身份，不注入
  prompt。`node_executions` 的追加行就是尝试历史，不另建 attempt 表。

**为什么不能直接复用执行行**：执行身份是回调归属与结果结算的依据。复用同一行会让上一次尝试的迟到结果
看起来像当前尝试的结果，直接违反「没有已提交结果的节点绝不记为成功」。**安全迁移路径**：一个 forward
migration 增加尝试列、重建两个部分唯一索引；存量行按 attempt 1 解释；命令与执行行按同一 attempt 对应。

### D5：取消、失败结算与边界

- **取消优先于重试**：一轮 pass 只要读到 `cancelRequestedAt`，就按取消路径结算，不派发任何新 attempt。
  等待中的 `retrying` 节点结算为 `cancelled`，运行 `cancelled`。
- **取消与失败结果竞争**：取消请求已记录时，即使会话结果是可重试失败，也不调度重试；运行已经不在
  `running` 的结算路径不产生新等待。
- **预算用尽**：节点结算为 `failed`，错误码取最后一次尝试的失败码；运行以同一码失败。节点状态保留
  `attempt` / `maxAttempts`，运行视图可以说清「3 次尝试均失败」。
- **不可重试的失败**或策略关闭时立即失败，与今天的路径一致。
- 重试等待不改变运行的 `error`：只有终态才写运行错误，等待期间运行错误保持空。

### D6：可观察性与公开契约

- `WorkflowRun.status` 不新增取值；等待重试的运行仍然是 `running`。
- `nodeStates[node].status` 新增 `retrying`；公开文档与前端状态联合类型同步增加。运行视图显示尝试次数和
  下一次尝试时刻。
- 节点终态失败时保留 `retry` 摘要。单次尝试的失败原因留在执行行上，公开的运行错误仍是最后的失败码。
- 每次自动重试不产生新的 Issue 活动：运行状态没有变化，重试在运行视图里可见，避免 Issue 线程被重复尝试
  刷屏。

### D7：resume 的语义、接口与首期不可用

**契约语义**（供后续实现，本期不实现）：

- resume 是 run 级的用户发起操作，只适用于 `failed` 的运行。它重新执行失败的节点，以及下游尚未提交成功
  结果的节点；已经成功的节点保留结果，不重跑。
- 被恢复的尝试从哪个 Workspace 状态开始，由回滚模式决定：desktop 的 `keep` / `node_files` / `checkpoint`
  三种模式及其语义是参考；Cloud 的保留窗口、检查点和回滚实现由后续决策确定。本文只固定「恢复后的执行必须
  与所选模式声明的状态一致」。
- **权限与取消一致**：运行带冻结模型绑定时，只有发起人和租户管理员可以发起；其他成员 403，非成员维持
  404。没有绑定的运行沿用成员规则。
- **幂等**：同一次失败至多产生一个恢复请求；重复请求是空操作，返回同一个结果，绝不派发第二次恢复。
- 对非 `failed` 运行的恢复请求返回 `409 resume_not_allowed`。
- resume 不改变冻结图、快照、输入或模型绑定；每次恢复开启新的重试预算（继承 desktop），恢复次数上限由
  后续决策确定。

**首期接口与不可用行为**：

- `POST /api/v1/tenants/{tid}/workflows/{wfid}/runs/{rid}/resume`，请求体是空 JSON 对象。
- 授权先于能力判断：按上面的权限规则判定，未通过时仍然返回 403 / 404，不泄露运行是否存在。
- 首期对所有通过的请求返回 `409 resume_unavailable`：不写任何状态、不记录幂等键、不创建 Workspace、不派发
  会话。拒绝是明确的、可测试的，而不是 404 或静默成功。
- 运行页不提供恢复入口；文档写明首期不可用及其原因。
- **为什么保留一个只拒绝的路由**：没有它，客户端只能得到 404，无法区分「能力尚未实现」与「路径写错或已被
  移除」。owner 的结论要求明确拒绝；路由形状现在就固定，后续实现只把拒绝换成真实行为，不再改客户端。

**未实现的前置条件**：失败运行的 Workspace 保留窗口、每节点检查点与回滚（三种模式）、恢复次数上限、
运行页入口。它们需要一个后续决策；在它生效前，本文的拒绝行为就是完整契约。

## 不变量

1. 重试期间运行保持 `running`；重试不改变运行状态。
2. `retrying` 节点视为在途：阻止下游派发，运行不能结算。
3. 同一节点同一时刻至多一个在途 attempt；同一 run 的 agent 节点仍然串行。
4. 每次 attempt 是独立执行身份；只有当前 attempt 的结果能结算节点，旧 attempt 的迟到或重复结果是无副作用的空操作。
5. 重试预算持久化在节点状态；调度器重启、换副本、重复 pass 都不重置预算。
6. 自动重试不回滚 Workspace；下一次 attempt 看到上一次 attempt 留下的状态。
7. 取消优先于重试；收到取消请求后不再派发任何 attempt。
8. 只有可重试类别且预算未用尽才重试；其他失败按原规则立即结算。
9. 没有已提交结果的节点绝不记为成功。
10. resume 首期不可用：任何 resume 请求都不改变状态、不创建 Workspace；页面没有恢复入口。

## 为什么不是这些替代方案

| 方案 | 取舍 |
|---|---|
| 重试等待用进程内定时器（desktop 的 `WorkflowRetryTimer`） | 最贴近参考实现，但违反根决策条件②：Cloud 不得依赖进程内状态，副本崩溃或切换会丢定时器。调度器重算把等待变成从 `dueAt` 推导，代价是最多一个 tick 的延迟 |
| 等待中的节点保持 `running`，把等待信息塞进 payload（desktop 的行形状） | 复用「running 即在途」的现有规则；但 Cloud 的 `node_states` 是面向运行视图的投影，不是 attempt 行。`running` 会让视图把等待画成「正在工作」，也无法表达下一次尝试的时刻。新增 `retrying` 的代价是前端一个状态，换来诚实的可观察性 |
| 新增 `workflow_node_attempts` 表 | 与 desktop 的 node-run 行最接近；但 Cloud 已有 `node_executions` 作为每次会话执行的持久记录，再加一张表会造出第二个 attempt 身份来源。选择给 `node_executions` 增加 attempt 维度 |
| 缺省不重试，只有显式 `enabled: true` 才重试 | 更保守，不改变存量快照的行为；但与 desktop 的默认策略分叉，导入的 desktop 图在 Cloud 上行为不同。选择对齐 desktop，并要求编辑器与运行视图把策略和重试显示出来 |
| resume 首期直接实现 | 需要先改终态 Workspace 释放策略并补检查点与回滚，跨 cloud/desktop/specs；owner 已在 ora-space/circus#448 否决 |
| resume 不提供路由，等实现时再加 | 客户端只能得到 404，无法区分「尚未支持」与「路径写错」，与 owner 要求的明确不可用错误不符 |

## 本决策未解决的问题

- resume 的 Workspace 保留窗口、检查点与 `keep` / `node_files` / `checkpoint` 三种回滚模式在 Cloud 的实现
  形状，以及恢复次数上限：需要后续决策；本文只固定用户可见语义与接口。
- 跨节点的重试并发：首期 agent 节点串行，重试不改变这一点；将来开放并行时，重试等待与并行的交互需要重新
  检查。
- interactive Agent 节点的失败与重试语义：首期发布门禁拒绝，等 HITL 决策与 specs#177 收敛后再定。
- 默认开启重试带来的模型调用成本：实现应在文档中说明默认值与上限；是否需要按租户配置预算，属于后续的运行
  策略。

## 落地顺序

1. **发布门禁与图 schema**：接受合法 retry 策略，按节点、字段、原因拒绝非法值；编辑器可以查看和编辑。
   完成条件：发布、运行准入与 @ 投影读同一份校验；测试覆盖缺省、边界与非对象。行为变化：显式 retry 从
   「拒绝」变为「接受或按字段拒绝」。
2. **调度器重试**：`retrying` 状态、attempt 身份、到期唤醒、预算持久化、取消优先、迟到结果无副作用。
   完成条件：真实 PostgreSQL 与真实调度器路径覆盖等待、唤醒、预算耗尽、取消、重启重算。行为变化：
   agent 节点在可重试失败后不再立即把运行判失败。
3. **运行视图与文档**：`retrying` 状态、重试摘要、中英文文档。完成条件：查看器对等待与耗尽两种终态都能
   说清尝试次数和原因。
4. **resume 首期不可用**：路由返回 `409 resume_unavailable`，页面无入口，文档说明。完成条件：授权先于
   拒绝、无状态写入、无 Workspace 创建；契约测试覆盖。行为变化：客户端从「没有这个接口」变为「得到明确的
   不可用错误」。

第 4 阶段不依赖前三个阶段；第 1、2 阶段可以同批落地。上述阶段全部完成前，不声称 Cloud 支持运行恢复。
