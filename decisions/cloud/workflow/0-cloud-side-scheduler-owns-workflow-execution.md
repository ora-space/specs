---
status: proposed
---

# 工作流由 Cloud 侧调度器执行：WorkflowRun 是执行聚合，服务端首期只做 start / agent / condition / output

Cloud 侧新增一个工作流调度器，复用既有控制面（`execution_work`、claim、`node_executions`、hooks）和 Node 侧
`ora-agent-runtime`，把 `workflow_runs` 从"写完 `pending` 就没有执行者"变成真实的执行聚合；运行的身份
（快照、输入、插件版本、逐槽位模型绑定、Project、Workspace）在创建运行的同一事务里冻结；**specs 是语义的
唯一来源，desktop 是参考实现**；服务端首期只执行 `start` / `agent` / `condition` / `output` 四个节点，
Controller 契约不变，不建 `controller/workflow`。

本决策状态为 `proposed`。涉及 cloud 的规范图 schema、运行身份模型、调度与执行契约、发布与启动门禁；涉及
specs 的规范图 schema 与首期核心用例（见「落地顺序」）；涉及 desktop 只作为参考实现，**本决策不修改任何
desktop 决策**。参考实现的证据基线：desktop `8909f07`（引擎模块非测试代码约 9.3 千行、测试约 8.2 千行，
`WorkflowRunEngineRepository` 31 个方法，覆盖 loop / iteration / region / retry / aggregator / resume /
snapshot switch），cloud 现状 `a307027`（生产里 `workflow_runs` 只能是 `pending`，
`executor_type='workflow'` 的 IssueRun 永远停在 `queued`）。

**非目标**：iteration、loop、HITL、retry、resume、snapshot switch 首期一律不做（理由与去向见 D1 与 D4）；
私有仓库的凭据与克隆（D5）；成果交付（D7）；按工作流覆盖个人槽位映射（属 #118 的后续扩展）。
**兼容边界**：desktop 的图行为不变（见 ora-space/circus#124）；cloud 已存的图不做数据迁移；
已永久 `queued` 的 `executor_type='workflow'` IssueRun 按 D3 一次性结算。

## 继承与修改

| 前序约定 | 本文处理 |
|---|---|
| [cloud/issue-run 根决策](../issue-run/0-agent-run-in-disposable-isolated-workspace.md) D2：每个 Agent IssueRun 独占一个运行 Workspace，`issue_runs.workspace_id` 与 `workspaces.issue_run_id` 互相唯一引用 | 修改：工作流的运行 Workspace 改绑 WorkflowRun；**agent 类 IssueRun 保持不变** |
| issue-run 根决策 D3：`issue_runs.phase` 只对 `executor_type = agent` 使用，`status` 保持原义 | 继承，不变：工作流运行用自己的状态机（`workflow_runs.status`），不扩 `phase`，也不改 `status` 的既有含义 |
| issue-run 根决策 D4 / D5：执行结果与成果保存分开记，交付完成后才释放 | 修改，仅限工作流：首期只保证资源释放，成果交付明确延后（D7），理由与去向写在 D7 |
| issue-run 根决策 D6：取消是请求加有界终态，实际终态在清理后写入 | 继承，不变；本文把它扩展到工作流运行（D4、D6） |
| [cloud/model-access 根决策](../model-access/0-personal-model-connections-and-run-scoped-proxy.md) D3：创建运行的同一事务里冻结连接版本、密钥引用、模型和发起者 Git 身份，任一项不可用则回滚 | 修改：冻结对象从"一次运行一组"改为"一次运行、每个用到的槽位一组"，且绑定挂在 WorkflowRun 上（D3）。不变量（冻结后不受配置变更影响）继承，不变 |
| model-access 根决策 D5：模型 ID 必须与运行冻结值相同 | 继承，不变，扩展到按槽位取冻结值 |
| [执行器槽位后续决策](20261015-executor-slots-replace-agent-model-ids.md)：图顶层 `executorSlots`，agent 节点用 `executorSlot` 引用，`default` 隐式保留，`hint` 只展示 | 继承，不变；它是本文 D2 规范图 schema 的一部分。本文补上它未定的运行实体（D3） |
| [个人槽位映射与按槽位锁定绑定](../model-access/20261015-executor-slot-mappings-and-per-slot-run-bindings.md) D3："（运行，槽位）唯一"，运行实体待定 | 继承，并落定：运行实体是 WorkflowRun（工作流）或 IssueRun（agent），见 D3 |
| [cloud/workspace 根决策](../workspace/0-collab-space-boundary.md) D1：`collab_workspaces` 与运行时 `workspaces` 是两套互不共享状态的聚合 | 继承，不变 |
| [desktop/plugin/registry 决策](../../desktop/plugin/registry/20260831-source-derived-namespace.md)：`workflows` 表的 namespace 恒为 `local`，插件不向它贡献任何行 | 继承，不变；工作流的归属与命名空间不受插件影响 |
| [MCP 后续决策](../../desktop/core/agent/session/mcp/20260912-workflow-explicit-selection.md) D4：发布只接受 canonical `PluginId` 并校验图结构，是否已安装属于运行时检查，不能阻止保存草稿 | 继承，不变；它是本文发布门禁的上界，D2 的拒绝只针对图结构与节点配置，不针对安装状态 |
| [desktop/core/workflow 根决策](../../desktop/core/workflow/0-node-runtime-orchestration.md)：desktop 引擎的范围，明确把"远程执行"和"跨机器 run ownership"留给后续决策（`:14`、`:113`） | 继承，不变：本文接住它排除的跨机器 run ownership，不修改 desktop 的任何决策 |
| [desktop 迭代决策](../../desktop/core/workflow/20260915-iteration-composite-runtime.md) 已是 `implemented` | 继承**语义**，不继承首期范围：首期不实现 iteration，理由与去向见 D4；启动 D1 的硬触发条件 |
| [decisions/controller/](../../controller/README.md) 已有的 Controller 职责与协议 | 继承，不变：**本决策不新增 Controller 契约**，不建 `controller/workflow`（D8） |

## 决策

### D1：执行归属归 Cloud 侧调度器，并锁死首期范围

Cloud 是工作流执行的权威执行侧。Cloud 侧新增一个**工作流调度器**，复用既有的控制面能力（`execution_work`
工作项、claim、`node_executions` 与事件回执、hooks）和 Node 侧 `ora-agent-runtime`；agent 节点仍走既有的
AgentSession 执行路径。**specs 是语义的唯一来源，desktop 是参考实现**：两边行为不一致时改实现，不改语义。

选 B（Cloud 侧调度器）而不是把 desktop 引擎整体搬到服务端，是因为首期只需要四个节点，而 desktop 的
`WorkflowRunEngineRepository` 有 31 个方法、约 9.3 千行非测试代码，覆盖 loop / iteration / region / retry /
aggregator / resume / snapshot switch，全部搬到 Cloud 权威存储上成本高。但 B 只在首期成立，因此附四个条件。

#### 条件①（硬触发）：首期范围锁死

**服务端引入 loop、iteration、retry、resume 中任意一项之前，必须先重新决策"继续在 Go 里移植"，还是"抽取
desktop 的 Rust 引擎核心为共享组件"。** 这是硬触发条件，不是"届时评估"：触发时先冻结该项的新功能开发，
做出决策后再继续。理由是 desktop 的 loop / iteration 把持久化形状写进了接口
（`start_loop_round`、`settle_iteration_round`、`complete_iteration_node`、`list_node_runs_in_scope`，以及
带 `WorkflowScopeId` 的 scope 级推进），逐个在 Cloud 权威存储上复现这些原子复合事务，复现的是引擎的一大块，
而不只是几个纯函数。snapshot switch 依赖冻结图替换，与 loop / iteration 同批决策。

#### 条件②：沿用"无状态调度 + 单 run 串行"，用 Cloud 的 claim/lease 实现，并正面解决多副本

参考实现的语义（desktop `workflow_run/engine/engine.rs:43-49`）：调度器同步且无状态，**每条命令都从持久化
状态重新计算已完成、在途和就绪集合**；**同一个 run 的所有命令与回调必须经过同一个串行执行者**。desktop 用
进程内的锁表实现这个串行执行者（`KeyedResourceLocks`），cloud 必须换成跨进程的等价物，且**不能依赖任何
进程内状态**。具体要求：

- **每次推进都从 Cloud 已提交状态重算就绪集合**：调度器不缓存图状态，崩溃后不需要恢复内部状态，只从
  `workflow_runs` / 节点结果重算。这对应 desktop 的 `resume`（`engine.rs:328-330`："Recomputes the ready set
  from persisted state"）。
- **单 run 串行用排他的数据库事务权实现**：延续 Cloud 既有的做法（`store.go:328` 在每次控制事务开头取事务级
  咨询锁），把作用域从全局收窄到单个 run（按 run id 取排他咨询锁）。任何副本拿到该 run 的排他权才能推进它。
- **节点结果提交是状态比较交换**：提交必须带 from→to 的当前状态，重复或迟到的回调是幂等的空操作（对应
  desktop 的 `transition_node_run_status` 与 `AdvanceWorkflowRunResult::NotRunning`，`ports.rs:193-196`）。
- **回调带派发时的执行代（epoch）**：复用控制面既有的代机制（`node_executions.dispatched_epoch`、
  `controller_leases.epoch`、`operations.controller_epoch`）；代过期的回调被拒绝，不得推进图。
- **多副本与租约**：多副本下同一个 run 只能有一个执行者，靠上面的排他事务权 + 代校验共同保证；副本崩溃时
  锁随事务结束释放，下一个副本从已提交状态重算。**在途未提交结果的节点在恢复时按中断结算为失败，绝不记为
  成功**（与条件④、D4 的崩溃恢复要求一致）。锁的键、租约时长、续租与失效窗口的具体数值由实现给出并在运行
  issue 记录；本文只固定上面这些不变量。

#### 条件③：对拍用例与批准同时就位

本 ADR 处于 `proposed` 期间不写核心用例（[specs/AGENTS.md](../../../AGENTS.md) 的 ADR 生命周期规定）。
**ADR 批准为 `approved` 时，首期子集的核心用例必须同时就位，并要求 desktop 与 cloud 两侧都通过**，之后 #70
才开工。这是"语义单一来源"唯一能落地的约束。

#### 条件④：调度器边界按 desktop 端口的形状定义

服务端调度器的接口形状对齐 desktop 的端口，将来抽取 Rust 核心时只替换这一层，其余不动。至少要对应：

| desktop 端口 | 形状（参考实现） | 服务端对应物 |
|---|---|---|
| 执行上下文 | `ExecutionContext { run, root_scope_id, workspace, graph_json }`（`ports.rs:81-87`） | 调度器派发节点时交给执行侧的一份只读上下文（冻结图、运行输入、运行 Workspace、变量池） |
| 节点结果提交 | `complete_node(...)` / `fail_node(..., FailurePropagation)`（`ports.rs:442-465`） | 同一语义的提交入口：成功带输出，失败带失败种类；首期只支持"节点失败即运行失败"，不实现传播策略（见 D4） |
| 就绪集合推进 | `BranchProjection::ready_nodes()`（`branch_projection.rs:141`）从冻结图 + 已提交节点结果 + 条件决策推导 | 从 `workflow_runs.status`/节点结果 + 冻结图重算"下一批就绪节点"，分支激活是推导出来的，不持久化 |

`NodeExecutor` / `WorkflowRunCallback` 这对分发-回调接口（`node_executor.rs:30-128`）是 desktop 与执行侧之间
唯一的集成缝，服务端也要保持"只有一个集成缝"。

### D2：规范图 schema 由 specs 拥有，节点逐个写明去向

**specs 拥有规范的图 schema**，cloud 前端的 `WORKFLOW_NODE_KINDS`（`frontend/src/features/workflows/runtime/types.ts:7-21`）
与 desktop 的 `NodeType`（`workflow_run/engine/node_type.rs:12`）都向它对齐。规范 schema 包含
[执行器槽位后续决策](20261015-executor-slots-replace-agent-model-ids.md) 定义的 `executorSlots`。

两侧不一致的节点**逐个写明去向**，并且规范 schema 要**认识**双方的每一种节点——包括 cloud 侧不能创作的
desktop 节点——这样门禁才能给出确切原因，而不是"未知类型"：

| 节点（wire 字符串） | desktop 参考实现 | cloud 编辑器现状 | 首期处置 |
|---|---|---|---|
| `start` | `Start`，可执行 | 有 | **首期可执行** |
| `agent` | `Agent`，可执行 | 有 | **首期可执行**；`agentConfig.executor { agentCli, modelId }` 改为 `executorSlot` 引用（槽位决策 D2） |
| `condition` | `Condition`，可执行 | 有 | **首期可执行** |
| `output` | `Output`，可执行 | 有 | **首期可执行** |
| `junction` | 无此类型 | 有 | 首期不执行，**发布门禁拒绝**；归类见「本决策未解决的问题」 |
| `human` | 无此类型 | 有 | 延后（HITL，需要 `awaiting_input` 与 Thread 交互），**发布门禁拒绝** |
| `subflow` | 无此类型 | 有 | 延后（嵌套运行，与条件①的硬触发条件合并），**发布门禁拒绝** |
| `loop` | `Loop`，可执行 | 有 | 延后（D4），**发布门禁拒绝** |
| `iteration` | `Iteration`，可执行 | 有 | 延后（D4），**发布门禁拒绝** |
| `tool` | `Tool`，已知但**不可执行**（引擎在启动时拒绝，`node_type.rs:56-68`） | 有 | 首期不可执行，**发布门禁拒绝** |
| `prompt` | `Prompt`，已知但**不可执行** | 无 | 首期不可执行，**发布门禁拒绝** |
| `aggregator` | `Aggregator`，可执行 | 无 | 延后（D4），**发布门禁拒绝** |
| `loopExit` | `LoopExit`，可执行 | 无 | 延后（D4），**发布门禁拒绝** |

拼写采用两侧的现有拼写，不改名。`loopExit` 是唯一的驼峰拼写，规范 schema 记为保留：desktop 已经把它持久化
在 `workflow_node_runs.node_type` 里，改名是破坏性变更且没有收益；将来若要统一拼写，需要单独决策。

**agent 节点配置字段的去向同样要写明**，避免"配置被静默忽略"：首期只使用 `executorSlot`、`roleId`、`prompt`；
`skills`、`mcps`、`interactive`、`outputContract`、`retry`、`promptInactivity` 延后，**发布门禁遇到它们要明确
拒绝并指出节点与字段**。理由：这些字段会改变执行行为，静默忽略会让作者以为配置生效。

**发布门禁与服务端可执行子集**：发布时校验图结构、节点类型、节点配置 schema 和 canonical `PluginId`，
**按服务端首期可执行子集拒绝，并给出原因**（先例：desktop 的 `first_unsupported_node()`，
`loop_graph.rs:51-59`），不静默忽略。**未完成的草稿仍然可以保存**：保存保持宽松，发布才是"这份图将成为运行
执行的冻结文档"的时刻（继承 MCP 决策 D4）。**安装状态不是发布门禁**：是否已安装、配置是否完整属于运行时
检查（MCP D4），由启动门禁处理（#69）。发布也不检查映射（槽位决策：发布者不一定是运行者）。

### D3：WorkflowRun 是执行聚合，IssueRun 通过 `workflow_invocation_ref` 引用它

**`workflow_runs` 是执行的聚合**，IssueRun 只是"某次调用"的记录。三种启动方式：

| 启动方式 | 产生的运行实体 | 幂等键 |
|---|---|---|
| 直接运行（`POST …/workflows/:wfid/runs`） | 只有 WorkflowRun | HTTP `Idempotency-Key`（沿用现有机制） |
| issue 里的 manual workflow IssueRun | IssueRun（`executor_type='workflow'`）+ WorkflowRun | 同上 |
| `@` Confirm | IssueRun（`executor_type='workflow'`）+ WorkflowRun | **interaction id**（沿用 `issue_interactions.run_id IS NULL → NOT NULL` 的比较交换） |

- `issue_runs.workflow_invocation_ref`（该列已存在于 `0010_issue_collaboration.sql:56`，至今没有任何写入者）
  成为到 `workflow_runs(id)` 的实引用，租户一致。**一个 workflow IssueRun 恰好引用一个 WorkflowRun，
  一个 WorkflowRun 至多被一个 workflow IssueRun 引用**；重跑创建新的 WorkflowRun（`issue_runs` 已有
  `rerun_of_run_id` / `retry_of_run_id` 的既有形状）。
- `issue_interactions.run_id` 的语义**不变**，仍指向那次调用的 IssueRun；WorkflowRun 经由
  `workflow_invocation_ref` 到达。这样重复 Confirm 的幂等与既有时间线行为（`executor_type='workflow'` 的
  IssueActivity）都不需要改动。
- **三种入口走同一个启动函数**（#69）：同一个函数负责门禁校验、冻结身份、创建运行实体。manual workflow
  IssueRun 不能再像今天那样只校验 UUID 形状（`issue_runs.go:81-104`，注释原文"Team/workflow keep the opaque
  UUID shape, resolved by ports later"）就落库。

**创建运行的同一事务里冻结以下内容，之后只读**：

| 冻结项 | 落地位置 |
|---|---|
| 快照 | `workflow_runs.snapshot_id`（已有；draft 一律拒绝，沿用今天的 `workflow_no_published_snapshot`） |
| 输入 | `workflow_runs.input`（已有） |
| 发起者 | **新增** `workflow_runs.created_by` —— 模型调用的承担者，也是 Thread 语义里"只有发起者可追加"的那个人 |
| Project | **新增** `workflow_runs.project_id` —— 运行 Workspace 的归属；直接运行必须在请求体给出并通过调用者权限校验 |
| 运行 Workspace 归属 | **新增** `workspaces.workflow_run_id`，与既有 `workspaces.issue_run_id` 互斥（恰好一个非空）；行本身仍按控制面既有的 `create_workspace` 路径建立，`kind='isolated'`，与 agent run 一致 |
| 逐槽位模型绑定 | `run_model_bindings` 增至（运行，槽位）唯一：agent IssueRun 的绑定仍以 `issue_runs.id` 为运行实体（槽位记 `default`，现有行为不变），工作流运行的绑定以 `workflow_runs.id` 为运行实体。运行实体判别列 + 两个唯一约束（运行 + 槽位）。槽位语义见 [model-access 后续决策](../model-access/20261015-executor-slot-mappings-and-per-slot-run-bindings.md) |
| 插件版本 | 运行解析到的插件版本（首期即 agent CLI 插件的版本，`space_plugins` 的当时值）；在途的插件升级不改变执行内容 |

**`version` 字段的命名要修掉。** `@` Confirm 的启动表单里那个 key 为 `version` 的字段，**值实际是 snapshot id**
（`workflow_collaboration.go:353-367`，注释原文"The value is the snapshot id rather than the version number"；
版本号只出现在标签 `v3 · 发布 3` 里）。这是个错误命名：它会以 `version` 的名字进入表单描述符和
`issue_interactions.input`。处置：**改名为 `snapshotId`**，与运行创建接口的请求体字段、`workflow_runs.snapshot_id`
一致；标签继续携带人类可读的版本号。兼容方式：**读取**时接受已存的旧 key `version`（历史 interaction 的
`input` 是不可变数据），写入一律用 `snapshotId`。注意这不是
`workflow_runs.version`（那是与 `workflows.version` 一致的乐观并发计数器，命名没有问题，保持不动）。

**存量数据**：已经永久 `queued`（或 `dispatched`）的 `executor_type='workflow'` IssueRun **统一标记失败**，
不做迁移。理由：这些行没有任何冻结的执行身份（无快照绑定、无输入、无插件版本、无逐槽位绑定、无 Project /
Workspace 归属），在新契约下无法执行，也无法诚实补出来；标记失败是状态更正，不是数据丢失——它们从未执行
过。实现时用一次追加迁移写入终态与 `failure_reason`（取值标识"运行时尚无工作流执行能力"），此后启动门禁
保证不再产生这种行。

### D4：首期节点与运行生命周期

**首期可执行节点**：`start`、`agent`、`condition`、`output`。其余节点一律在发布门禁被拒绝（D2）。

**延后的项、理由与去向**：

| 延后项 | 理由 | 去向 |
|---|---|---|
| iteration、loop | 它们把持久化形状写进了 desktop 的接口（`start_loop_round` / `settle_iteration_round` / `list_node_runs_in_scope` / scope 级推进），在 Cloud 权威存储上复现是一大块引擎；desktop 的迭代决策虽已 `implemented`，首期只继承其**语义** | 与条件①的硬触发条件合并：先决策"继续 Go 移植"还是"抽取 Rust 核心" |
| snapshot switch | 依赖冻结图替换与在途节点兼容性规划 | 同上 |
| retry | desktop 的重试是引擎内决策加进程内定时器；Cloud 首期没有等价物 | 后续决策；首期节点失败即运行失败 |
| HITL（`human`、`awaiting_input`） | 需要 Thread 的对话式交互与有界的等待窗口 | 后续的交互式运行控制决策 |
| subflow | 嵌套运行 | 与条件①合并 |
| `aggregator`、`loopExit`、`tool`、`prompt` | 要么属于 loop/iteration 生态，要么本来就不是可执行节点 | 随所属特性一起；`tool` / `prompt` 在 desktop 侧也只在启动时被拒绝 |

**同一 run 内多个 agent 节点首期串行。** 理由：它们共用该运行的 Workspace，并行会让文件改动互相踩踏，也会
让成果归属无法界定。并行留作后续扩展。

**取消是请求加有界终态**（继承 issue-run D6）：取消先记录请求（`cancel_requested_at` 的等价列），运行进入
有界的收尾路径，**实际的终态在清理完成后才写入**；不因为收到请求就直接写终态。**首期必须有取消的写入者**（D6）。

**崩溃恢复只依据已提交的节点结果**：恢复时从 `workflow_runs` 与已提交的节点结果重算就绪集合；已经提交结果的
节点不重复派发；在途没有结果的节点按中断结算为失败。**绝不把未知当作成功。**

### D5：仓库凭据首期只支持匿名公开 Git

首期执行只支持从匿名可访问的公开 Git 仓库取得代码，私有仓库的凭据链路（ora-space/circus#16）**不作为前置**。
理由是 #16 体量大，而当前真实可用的路径就是公开 Git。私有仓库与凭据属于后续决策，本决策不预设它的形状。

### D6：取消写入者纳入首期

**取消的写入者纳入首期**：没有写入者就无法取消，工作流的取消（以及 #75 依赖它的能力）都落不了地。
agent 运行侧同样缺取消的写入者（G-026），**一并补上**，因为它与工作流运行共用同一条收尾路径。

### D7：首期只保证资源释放，成果交付延后

首期保证运行的资源收尾：运行 Workspace 等资源按既有契约释放，不能因为工作流运行而泄漏沙盒。
**成果交付明确延后**：首期不把节点产出登记为 Revision、不做交付后的保存与重试（issue-run D4 / D5 的
交付语义不适用于工作流运行）。去向：由后续决策给出工作流的成果契约。节点产出在首期落在运行的节点状态里，
供运行详情展示。

### D8：根决策落地，且不新增 Controller 契约

`decisions/cloud/workflow/0-*.md`（本文）是这个领域的根决策。**不建 `controller/workflow`**：本文选 Cloud
侧调度器，agent 节点仍走既有的 AgentSession 执行路径，Controller 的职责与协议不变，**没有新的 Controller
契约**。[执行器槽位后续决策](20261015-executor-slots-replace-agent-model-ids.md) 与
[个人槽位映射决策](../model-access/20261015-executor-slot-mappings-and-per-slot-run-bindings.md)
（specs PR #98）是本文的后续决策，一起评审。

复用控制面时有一处必须说清的边界：`execution_work.run_id` 与 `node_executions` 现在都指向 `issue_runs`
（`0024_agent_run_control_plane.sql:33-63`），而工作流运行没有 IssueRun。处置：**由 Cloud 自己的表承载这个
差异**——给 `execution_work` / `node_executions` 增加可选的 WorkflowRun 归属（判别列 + "恰好一个归属"约束，
并把 `one_unregistered_execution_work`、`one_session_per_run` 一类唯一约束改成按归属成立）。**Controller 的
线上契约不变**（工作项种类、claim、回执、代校验都不改）；如果某一步会让 Controller 必须理解工作流，那就不是
本文允许的做法，需要按 D8 重新决策。

## 不变量

1. 一个 WorkflowRun 执行的图在创建后不可变；编辑、发布或插件升级都不改变在途运行执行的内容。
2. 执行身份（快照、输入、发起者、Project、运行 Workspace 归属、逐槽位模型绑定、插件版本）在创建事务里全部
   落库，之后只读。
3. 同一个 WorkflowRun 在同一时刻至多有一个串行执行者；节点结果提交是带当前状态的比较交换，迟到或重复的
   回调是空操作。
4. 就绪集合只从 Cloud 已提交的状态重算；调度器不持有跨事务的必需内部状态，副本崩溃后不丢失进度。
5. 没有已提交结果的节点永不被记为成功。
6. 一个 workflow IssueRun 恰好引用一个 WorkflowRun；一个 WorkflowRun 至多被一个 workflow IssueRun 引用。
7. 未配置或不可用的槽位不会被 `default`、`hint` 或作者的选择顶替（继承槽位决策与 model-access 后续决策）。
8. 发布只校验图本身，不校验安装状态与个人映射；保存草稿始终允许未完成的编排。

## 为什么不是这些替代方案

| 方案 | 取舍 |
|---|---|
| **A：把 desktop 引擎整体搬到服务端**（用 Cloud 权威存储实现那 31 个方法） | 语义天然一致，且不必重复实现；但首期就要复现约 9.3 千行非测试代码和 31 个方法的原子复合事务，而首期只需要四个节点。A 没有被否决，它是条件①触发时的候选之一 |
| 在 Cloud 用 Go 实现 desktop 的全部特性（含 loop/iteration）作为首期 | 用户一次就能用到完整能力；但会在 1.35 万行量级的重复实现上做语义对齐，且违背条件①"先决策再动手"的要求 |
| 让 desktop 承担服务端执行 | 复用成熟引擎；但 desktop 的根决策明确把远程执行与跨机器 run ownership 排除在自己的范围外（`0-node-runtime-orchestration.md:14`、`:113`），这是它的边界而不是待补的缺口 |
| 继续用 dev 装配的 `MockWorkflowRunSimulator` 交付 | 改动最小；但它是模拟 trace，不是真实 Node 执行，直接违反验收要求（不得以 HTTP 200、手改数据库或 fixture 成功验收） |
| 工作流运行继续留在 `executor_type='workflow'` 的 queued IssueRun 上，不建 WorkflowRun | 不新增运行实体；但无法冻结执行身份、无法按（运行，槽位）绑定模型、无法用 interaction id 幂等，也解释不了"同一个 IssueRun 只有一个 executor_id" |
| 为每个 agent 节点创建一个 IssueRun 来复用控制面 | 不用改控制面的表；但会把"调用记录"和"执行单元"混为一谈，还会污染 issue 时间线 |
| 把"是否已安装"前移成发布门禁 | 让不合法的图更早失败；但违反 MCP 决策 D4：安装状态是运行时检查，不能阻止用户保存可编辑草稿 |
| 首期就并行执行同一 run 内的多个 agent 节点 | 更快；但共用 Workspace 时文件改动会互相踩踏，成果归属无法界定 |
| 存量永久 queued 的 IssueRun 按新契约迁移重跑 | 看起来"不丢任务"；但这些行没有任何冻结的执行身份，迁移只能凭空补造，等于伪造执行证据 |
| 让 `version` 字段继续以 `version` 之名携带 snapshot id | 不改契约；但错误命名会固化进表单描述符与已存 interaction 输入，越晚改代价越大 |

## 本决策未解决的问题

- **`junction` 的归类**：它是纯结构性的合流节点，没有执行语义；首期 `condition` 的多分支激活是否需要一个显式
  合流节点，取决于分支激活方案（对应 desktop 的 `branch_projection`）。本文按 owner 的结论把首期节点限定为
  四个，因此把 `junction` 列为延后；若要把它并入首期，需要一次单独决策。
- **多副本下单 run 排他权的参数**：锁的键、租约时长、续租、失效窗口的具体数值，由实现按 Cloud 既有的
  claim/lease 给出并在运行 issue 记录。本文只固定 D1 条件②的不变量。
- **成果交付的具体契约**（D7 延后）：由后续决策给出。
- **首期核心用例的具体条目**：在本文变为 `approved` 时随它落入 `test-cases/cloud/workflow/`，
  并要求 desktop 与 cloud 两侧通过（条件③）。
- **存量 queued 结算的执行时点**：随实现 PR 还是独立迁移，由实现方定。
- **`awaiting_input` 的处置**：`workflow_runs.status` 的 CHECK 里已有该取值；首期不产生它，是否保留由实现定。
- **按工作流覆盖槽位映射**：属 #118 之后的扩展，不在本文范围。

## 落地顺序

1. **本 ADR 评审**（与 specs PR #98 的槽位后续决策一起）→ owner 置 `approved`；**同一时刻**首期子集的核心
   用例落入 `test-cases/cloud/workflow/`，desktop 与 cloud 两侧通过。这是条件③，也是下一步开工的前提。
2. **统一启动与发布门禁**（#69）：三种入口共用一个启动函数；发布门禁按首期可执行子集明确拒绝并给出原因；
   `version` → `snapshotId` 改名。完成条件：引用未声明槽位、含首期不支持节点或配置的图被拒绝且原因可定位；
   合法草稿仍可保存；manual workflow IssueRun 不能再用任意 UUID 绕过校验。
3. **Cloud 侧调度器**（#70）：按 D1 条件②、④ 实现无状态调度、单 run 串行、节点结果提交与就绪集合推进，
   真实跑通 Start → Agent → Output，fixtures=false 下直接运行从 `pending` 进入真实运行与成功/失败结算。
4. **运行身份与绑定**（#70 或紧随其后）：`workflow_runs` 增加 `created_by` / `project_id`；
   `workspaces.workflow_run_id`；`run_model_bindings` 改成按（运行，槽位）唯一；`workflow_invocation_ref`
   成为实引用。
5. **存量结算与取消写入者**（D3、D6）：一次性结算永久 queued 的 `executor_type='workflow'` IssueRun；
   补上工作流与 agent 两侧的取消写入者（G-026）。
6. **触发条件**：服务端引入 loop / iteration / retry / resume / snapshot switch 中任意一项之前，先执行
   D1 条件①的重新决策。
