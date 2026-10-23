---
status: approved
---

# 人工节点以运行级结构化输入继续：有界等待、追加式输入行，与 agent 会话解耦

本文是[工作流执行根决策](0-cloud-side-scheduler-owns-workflow-execution.md)（`implemented`）的后续决策，
承接其 D4 留下的「后续的交互式运行控制决策」。本文只固定首期 HITL 中 **`human` 节点**的契约：
运行进入 `awaiting_input` 后，由发起人或租户管理员通过运行级输入接口提交一份结构化输入；输入先追加
落库，调度器在下一次重算中消费它、把该节点结算为 `succeeded`，运行回到 `running` 继续推进。
等待必须有界（新配置 `workflow.hitl_wait_timeout`，默认 24 小时），取消或超时都进入终态，不允许运行
永久停在 `awaiting_input`。`human` 节点不创建 agent 会话，也不经过 Controller，因此单回合会话不变量
不变；首期 `human` 节点也不重试，同一（运行，节点）至多执行一次。interactive Agent 不在本期范围，
等 desktop 的[交互式节点完成契约](../../desktop/core/workflow/20261021-interactive-node-completion.md)
与 #351 收敛后另定。

**状态与影响面**：本文为 `approved`，2026-10-22 经 owner 评审批准（九项默认值按建议、等待中控制统一为
发起人或当前租户管理员均经 owner 确认），核心用例见
[人工节点的运行级输入与有界等待](../../../test-cases/cloud/workflow/hitl-run-input.md)。owner 已在
[ora-space/circus#449](https://github.com/ora-space/circus/issues/449) 拍板选 B：先定契约 ADR，暂不实现。
**本期产品行为不变**：发布门禁继续拒绝 `human` 节点和
`interactive: true`，`workflow_runs.status` 的 CHECK 里保留 `awaiting_input` 作为没有写入者的保留值；
公开 OpenAPI 与前端按 #449 的结论移除该状态（运行 ora-space/circus#450）；**#450 合并后**，再由实现
本文的同一变更把它加回公开枚举。**兼容边界**：新增输入接口、输入行、运行上的等待列与运行详情的两个
只读字段（`awaitingInput`、`inputs`）；不改图 schema 的既有节点、不改 Controller 契约、不改变取消接口
对既有状态的响应与错误码（等待中取消的谓词变化只作用于实现后才首次出现的 `awaiting_input`）。
**非目标**：不实现 HITL、不放开发布门禁、不改单回合 agent 会话、不定义指定处理人/通知渠道、不定义
interactive Agent 的后续回合。

## 继承与修改

| 前序约定 | 本文处理 |
|---|---|
| 根决策 D1：执行归属归 Cloud 侧调度器，节点终态只来自已提交状态 | 继承，不变：等待进入与退出都由调度器在重算里提交，输入请求只追加一条输入行 |
| 根决策 D2：规范图 schema 由 specs 拥有；`human` 首期被发布门禁拒绝 | 继承门禁结论，扩展 schema：`human` 节点声明 `humanConfig.inputSchema`，并声明 `output` 变量根；实现本文时发布门禁只放行 `human`，仍拒绝 `interactive: true` |
| 根决策 D4：HITL（`human`、`awaiting_input`）延后，去向「后续的交互式运行控制决策」；当时把 HITL 描述为需要 Thread 的对话式交互 | **承接并修改**：本文就是该后续决策，但 `human` 节点不再走 Thread 对话，改为运行级结构化输入；需要后续人类回合的 interactive Agent 仍延后 |
| 根决策 D4/D6：取消是请求加有界终态，取消写入者纳入首期 | 继承并扩展：实现本文后 `awaiting_input` 也接受取消；已记录的取消请求优先于未消费的输入与等待超时 |
| [个人模型发起者决策](../thread/20261010-personal-model-initiator-controls-input.md) 继承表：带模型绑定的运行只有发起者与当前租户管理员可结束，无模型绑定运行保持成员规则 | **限定修改**：`human` 等待中的控制（输入与取消）一律只有发起人或当前租户管理员可以执行，与是否冻结绑定无关；`pending` / `running` 的既有取消规则不变（D4） |
| 根决策 D7：首期只保证资源释放 | 继承：等待不是终态，不释放运行 Workspace；取消或超时进入终态后走既有释放路径，等待上限同时是 Workspace 保留上限 |
| 根决策 D8：不新增 Controller 契约 | 继承，不变：人工输入不经过 Node/Controller，不新增工作项、claim 或回执字段 |
| 根决策 D3：WorkflowRun 是执行聚合，IssueRun 只是调用记录 | 继承：输入行归属 WorkflowRun；IssueRun 只获得一条 Issue 活动，不是线程回合 |
| [会话结束后续决策](20261019-workflow-session-end-after-initial-turn.md) D1/D5：工作流 agent 会话是单回合的，节点终态仍只来自会话结果 | 继承，不变：`human` 节点不创建 agent 会话，因此不需要后续回合；interactive Agent 才会改变会话生命周期，不在本文 |
| [重试与恢复后续决策](20261021-node-retry-and-resume-contract.md) D4：同一（运行，节点）可因重试有多条执行证据，尝试由 `workflow_node_attempt` 区分 | **继承并限定**：首期 `human` 节点不重试，同一（运行，节点）至多执行一次，D6 的输入唯一键仍是 `(workflow_run_id, node_id)`；将来支持 human 重试或恢复时，按该决策 D4 引入 attempt 键，输入行再增加尝试代次 |
| 重试与恢复后续决策 D3/D8：等待中的 agent 重试提交节点 `retrying`，运行保持 `running`，不新增运行状态 | **修改**：HITL 等待是运行级事实，启用根决策已保留的 `awaiting_input`（不是新增状态）；等待中的人工节点在 `node_states` 里仍为 `running`，因此 pass 必须先处理运行级等待再进入会话证据路径（D2） |
| 重试与恢复后续决策「为什么不是这些替代方案」：拒绝用 `running` 表示等待 | **有条件沿用**：该否决针对 agent 节点——`running` 会进入会话证据读取。`human` 节点没有会话与执行证据，D2 用运行状态加等待列承载等待，并要求所有会话证据读取只作用于 `agent` 节点；不满足该前提就不能复用 `running` |
| 重试与恢复后续决策 D5：取消优先于等待中的重试 | 继承：已记录的取消请求优先于未消费的输入与等待超时 |
| 重试与恢复后续决策 D8：历史快照不重新过门禁，实现上线后按新语义执行 | **修改**：`human` 节点的 schema 是输入校验的必要前提，缺失时没有安全缺省；历史快照在运行准入处以命名原因失败，需重新发布后才能运行（D5） |
| 取消证据（运行 #347、cloud#120）与 S5 释放状态机（cloud#119） | 继承：取消路径继续按 `cancelled` 结算节点与运行，释放事实按既有 `workspace_release_*` 列登记 |

## 决策

### D1：交互通道是运行级输入接口加运行页

**规则**：输入走 `POST /api/v1/tenants/:tid/workflows/:wfid/runs/:rid/inputs`，请求体是
`{"nodeId": "<图节点 id>", "input": { ... }}`，`nodeId` 与 `input` 都必填；请求必须带平台既有的
`Idempotency-Key` 头（1–200 字符）。接口只追加一条输入行并返回 202，响应体是
`{"resource": {"id", "runId", "nodeId", "actorId", "input", "recordedAt"}}`。运行页用同一个接口提交，
不另开写入路径。

**接受条件**：授权检查最先执行（D4，403）；通过后在写入输入行的同一事务里用数据库时钟按固定顺序
判定，任一步不满足都不写输入行：

1. 运行 `status = 'awaiting_input'`，否则 `409 run_not_awaiting_input`；
2. 请求 `nodeId` 等于 `awaiting_input_node_id`，否则 `409 run_not_awaiting_input`；
3. `cancel_requested_at IS NULL`，否则 `409 run_not_awaiting_input`；
4. 该等待还没有输入行（同一个 `Idempotency-Key` 加同一请求由平台幂等记录先返回原响应），否则
   `409 workflow_input_already_recorded`；
5. 数据库时钟早于 `awaiting_input_at + workflow.hitl_wait_timeout`，否则
   `410 workflow_run_input_window_closed`。

**顺序决定可观察结果**：状态、节点、取消与「已有输入行」都先于窗口判定，因此 `410` 只在运行仍处于
`awaiting_input` 时可达。运行已被超时结算成 `failed` 或取消结算成 `cancelled` 后，晚到的输入无论窗口
是否已过都先命中状态检查，得到 `409 run_not_awaiting_input`；取消请求已记录但尚未结算、窗口也已过去
时，同样先命中取消检查得到 `409`。只有运行仍在等待、没有取消请求、也没有已接受的输入行，只是窗口
已过，才得到 `410`——此时输入不能再被消费，必须如实拒绝，而不是返回一个永远不会生效的 202。
D3 的「窗口后输入得到 410」与 D4 的「取消后输入得到 409」都以此顺序为准。

**读取形状（公开契约）**：运行详情（`GET …/workflows/:wfid/runs/:rid`）在现有 `WorkflowRun` 投影上新增
两个只读字段，与 `definitionSnapshot` / `modelBindings` 一样只在详情返回，列表行的投影不包含它们：

- `awaitingInput`：运行不在等待时为 `null`；等待时为 `{"nodeId", "startedAt", "deadline"}`。`startedAt`
  就是 `awaiting_input_at`；`deadline` 按 `awaiting_input_at + workflow.hitl_wait_timeout` 用同一部署
  配置与数据库时钟算法计算，不另存第三列——接受判定、调度器超时与这里始终使用同一算法，部署调整配置
  会让未结算等待的截止时间一起移动。
- `inputs`：该运行的输入行，按 `recordedAt` 升序、同一时刻按 `id` 升序，每项
  `{"id", "nodeId", "actorId", "input", "recordedAt"}`；详情读取始终返回数组，没有输入行时为空数组。

运行页因此不需要新的 GET 接口：等待节点、开始时刻、截止时间与输入历史都从运行详情读取；直接运行没有
Issue 活动，也只能从这里拿截止时间。列表只通过 `status = "awaiting_input"` 表达等待。输入行的 `input`
就是通过校验的节点输出对象，节点最终输出仍只从 `nodeStates[nodeId].output` 读取（D6）。

**202 的含义是「输入已在窗口内记录」**：消费它、结算节点和推进运行仍由调度器完成（D2）；若之后取消
先被结算，输入行保留在历史里但不再产生节点成功（D4）。

**为什么是运行级接口而不是 Issue 线程回复**：Issue 线程回复只覆盖 IssueRun，直接运行的 WorkflowRun
没有 Issue 可回；它还会把运行输入混进讨论时间线，让「讨论」和「运行控制」共用一种写入。运行级接口
同时服务直接运行和 IssueRun，权限与状态都以运行为准。**为什么不是专用对话会话**：专用会话需要引入
Thread、轮次与跨回合的会话生命周期，正是 interactive Agent 才需要的东西；`human` 节点只需要一次结构化
输入，用会话承载会把首期范围重新放大。

**为什么返回 202 而不是直接返回节点终态**：请求只完成「记录输入」；消费输入、结算节点与推进运行由
调度器在下一轮重算完成（D2）。这与取消接口「记录请求、调度器反应」的分工一致，也保证请求事务不携带
调度推进。

### D2：等待写入 `awaiting_input`，输入被调度器消费后回到 `running`

**规则**：运行上的等待事实由两个新列共同承载：`awaiting_input_at timestamptz` 与
`awaiting_input_node_id text`，两者同时写入、同时清空，CHECK 保证同为空或同非空。`human` 节点成为
就绪节点时，调度器在同一个事务里：

1. 把该节点在 `node_states` 里置为 `running`（节点状态词表不新增值，等待是运行级状态）；
2. 写入 `awaiting_input_at = clock_timestamp()`（数据库时钟）与 `awaiting_input_node_id`；
3. 把运行 `status` 置为 `awaiting_input`，版本 +1，并为关联的 IssueRun 记一条 Issue 活动（D8）。

输入行提交后，下一次调度重算在同一事务里消费它：把该节点置为 `succeeded`、`output` 为通过校验的输入
对象、写 `finished_at`，清空两个等待列，把运行放回 `running` 并继续按图推进；如果这是最后一个节点，
同一次推进直接进入 `succeeded`。**HTTP 请求不写运行状态、不写 `node_states`、不推进图**；它只追加
D6 的输入行。调度器崩溃后重启，下一轮重算读到的仍是同一条已提交输入，消费一次且只消费一次。
两个等待列在离开等待的每条路径上都同时清空——消费、取消、超时以及任何其它失败结算——因此「两列非空」
始终等价于「运行正在等待」，不会在终态留下半截等待事实。

**调度器如何看到等待**：现有 `scanAdvancableWorkflowRuns` 只扫 `pending` / `running`，且注释明确
「不重扫等待与终态」，pass 的第一条节点路径又是 `workflowRunningAgentNode`（只按
`state.Status == "running"` 取节点，不看 kind）。只按本文的等待写入，输入、取消与超时都不会被消费，
等待中的人工节点还会被当成「已派发但未回报的 agent 会话」，按启动超时判失败。因此实现本文时必须同时
落实三条契约：

1. **扫描覆盖等待**：`awaiting_input` 的运行由独立的有界扫描处理，不并入现有 `pending` / `running`
   批次。长期等待的运行按 `created_at` 排在前面，并入后会在等待数超过批次上限时挤占在途运行；独立
   扫描同样有界，超出的等待留到下一轮，保证在途运行与等待运行都前进。
2. **等待分支先于节点证据**：每个等待运行的 pass 在读取任何节点执行证据之前，按固定顺序处理：
   ① 已记录取消请求 → 按 D4 结算 `cancelled`；② 存在已接受的输入行 → 消费并结算节点（本 D2 上文）；
   ③ 数据库时钟已过截止时间 → 按 D3 结算 `failed`；④ 否则原样返回。任何一步都不落到
   `workflowRunningAgentNode` 的会话证据路径。
3. **会话证据读取只作用于 `agent` 节点**：`workflowRunningAgentNode`、`workflowAgentSessionOutcome`
   及其调用方（包括取消路径的同名判断）都必须结合图节点 kind，只有 `agent` 节点能进入会话证据读取。
   `human` 节点没有 `node_executions` 证据，不能被解释为「已派发但未回报」。

等待分支与 kind 限定是复用 `running` 表示等待的前提，不是性能优化；缺少任一条，D2 的状态选择就无法
只靠本文实现。

**状态转移**（实现本文后）：

| 起始 | 事件 | 结果 |
|---|---|---|
| `running` | `human` 节点就绪 | `awaiting_input`，等待列成对写入 |
| `awaiting_input` | 输入行被消费 | `running`（或图结束时的 `succeeded`） |
| `awaiting_input` | 取消请求 | `cancelled`（D4） |
| `awaiting_input` | 没有输入且超过等待上限 | `failed`（D3） |
| `awaiting_input` | 已记录取消请求后的输入请求 | 状态不变；`409 run_not_awaiting_input`，不写输入行 |
| `awaiting_input` | 无取消、无已接受输入，但窗口已过 | 状态不变；`410 workflow_run_input_window_closed`，不写输入行 |

**禁止**：在输入请求事务里直接把节点记为成功；在 `awaiting_input` 下派发其他节点；同一运行同时有两个
等待列指向不同节点；等待期间运行被投影成终态；等待分支被节点会话证据路径抢先；会话证据读取把一个
`human` 节点当成 agent 节点。

### D3：等待有界，默认 24 小时，到期按失败结算

**规则**：新增配置 `workflow.hitl_wait_timeout`，默认 **24 小时**，按部署配置，本期不做逐运行覆盖。
窗口从 `awaiting_input_at` 起算，用数据库时钟比较，与调度周期、进程时钟和副本数量无关。窗口到期且
该等待仍没有被消费的输入行时：

- 该 `human` 节点以命名原因 `hitl_wait_timeout` 失败，节点 `error` 记录同名原因；
- 运行进入 `failed`，`error` 记录 `human input wait timed out`；
- 运行 Workspace 按根决策 D7 的既有终态收尾声明删除，释放路径与失败运行相同。

**已接受的输入优先于超时**：「已接受」按 D1 的接受条件判定——在截止时间之前、没有取消请求时提交并
落库的输入行。只要存在这样的输入行，即使调度器在窗口之后才重算，也消费输入而不是判超时；运行仍在
`awaiting_input` 时，窗口之后的提交按 D1 的顺序得到 `410 workflow_run_input_window_closed`，不产生输入
行；若该等待已被本次超时结算成 `failed`，晚到的提交先命中状态检查，得到 `409 run_not_awaiting_input`。
两种回答都不产生输入行，因此不会出现「202 已接受但永远不会被消费」。超时只在该等待没有任何已接受
输入时成立。**取消优先于超时**：已记录的取消请求按 D4 结算 `cancelled`，不再写超时失败。

**为什么必须有界**：等待不是终态，运行 Workspace（以及运行本身）会被保留；没有上限就会重演「永久
`running`」的另一条形态。24 小时的依据：覆盖跨时区的一次人工审核（一个工作日加缓冲），同时把
Workspace 保留限制在一天量级；更长或更短的需求由部署配置调整，不把具体数字写进实现。**为什么按失败
而不是跳过**：跳过会把「没有人处理」伪装成「已批准」，节点后续分支可能因此执行错误路径；超时按失败
结算，让图和运行都停在可解释的终态。

### D4：输入与等待中的取消：发起人加租户管理员

**规则**：输入请求只允许该运行的发起人（`created_by`）与租户的当前管理员提交，其他活跃成员只读并得到
`403 workflow_run_input_forbidden`。谓词是「`uid = created_by` 或当前租户中 `status='active'` 且
`role='admin'` 的成员」，**与是否冻结模型绑定无关**：`workflowReferencedSlots` 只收集 `agent` 节点，
`start → human → output` 这类 human-only 图不会冻结任何 `run_model_bindings` 行；若沿用「无绑定运行
回退到成员规则」，输入就会对任意活跃成员开放。权限从当前 PostgreSQL 状态读取，不信任请求里的 actor
字段。授权检查先于状态、节点匹配与输入校验，未授权的调用者不能从 409/410/400 的差异推断运行是否在
等待。

**与既有取消谓词的关系**：运行在 `pending` / `running` 时的取消仍按既有规则——带冻结绑定时只有发起人
与当前租户管理员，无绑定时沿用 Issue 线程的成员规则（[个人模型发起者决策](../thread/20261010-personal-model-initiator-controls-input.md)
的「无模型绑定运行保持原规则」）。本文不改这条既有路径；只把**等待中的控制**（输入，以及
`awaiting_input` 状态下的取消）统一为发起人或当前租户管理员。这样 human-only 运行不会在等待期间成为
任意活跃成员可完成或可终止的对象，也避免「能提交输入的人不能取消、能取消的人不能提交」的不对称。
这是一条明确的修改，列入「继承与修改」。

**取消请求也接受 `awaiting_input`**：实现本文前，取消接口把 `awaiting_input` 当作不可取消
（`409 run_not_cancellable`）；实现本文后，它进入可取消状态集合，且等待中的取消使用本节的输入谓词。
取消请求记录后，调度器在下一次重算把等待中的 `human` 节点结算为 `cancelled`（没有会话可停止），运行
结算 `cancelled`，未消费的输入行保留为历史，但**不能把运行从终态复活**。取消请求与输入请求的竞争以
数据库提交顺序为准：取消先提交则之后的输入先命中 D1 的取消检查，得到 `409 run_not_awaiting_input`
（即使等待窗口也已过去）；输入先提交但尚未消费时取消到达，取消仍决定运行终态，该输入不产生节点成功。

**为什么等待中的取消也收窄**：输入与取消都是对等待中运行的控制写入。输入决定节点输出，取消决定运行
是否就此结束；如果输入只允许发起人/管理员而取消允许任意成员，权限模型就出现「不能完成、但能作废」的
旁路。指定处理人留到后续决策，不影响本谓词；非等待状态的取消保持既有规则，不在本文范围。

### D5：节点声明 JSON Schema，发布期只接受受支持的子集

**规则**：`human` 节点的配置为 `humanConfig.inputSchema`，是一个 JSON Schema 对象；`data.instruction`
仍是展示给处理人的提示文本。输入在写入前按 schema 校验；发布门禁在实现本文后同时校验 schema 本身。

**支持的子集**（闭合集合，不静默忽略未知关键字）：

- 顶层与每个嵌套对象都必须是 `"type": "object"`，且必须显式写 `"additionalProperties": false`；
- 对象关键字：`type`、`properties`、`required`、`additionalProperties`（只允许 `false`）；
- 数组关键字：`type: "array"`、`items`（一个受支持的子 schema）；
- 标量：`type` 为 `string` / `number` / `integer` / `boolean`，可用 `enum` 限定取值；
- `required` 必须是 `properties` 中已声明属性名的数组。

出现其他关键字（例如 `oneOf`、`$ref`、`patternProperties`、`format`、`minLength`）时，发布门禁以
`human_input_schema_unsupported` 拒绝并指出节点与字段；schema 本身不是对象、顶层不是 object、缺少
`additionalProperties: false`、`required` 指向未声明属性时，以 `human_input_schema_invalid` 拒绝。
这样门禁不会接受一份「看起来有约束、实际被实现忽略」的 schema。

**`humanConfig.inputSchema` 必填**：发布 `human` 节点时缺失（字段缺失、`null` 或空对象）以
`human_input_schema_missing` 拒绝。门禁不提供「缺失即接受任意 JSON」的缺省——那等于没有契约，也让
输入期校验无从执行。保存草稿仍宽松，与既有 draft / published 分工一致。

**历史快照**：已发布的冻结快照不重新过门禁，但运行准入读取同一份可执行子集。实现本文后，准入对快照
中的每个 `human` 节点要求存在有效的 `humanConfig.inputSchema`；缺失或非法时以命名原因拒绝创建运行
（`human_input_schema_missing` / `human_input_schema_invalid`），不回填、不猜测 schema。这样门禁之前
冻结或由其他客户端导入的 `human` 快照不会带着「没有校验」进入等待；需要运行它们的作者必须重新发布。
若某条执行路径绕过准入（防御分支），该节点首次进入等待时以同一原因失败整个运行，而不是接受无校验
输入。

**输入期校验使用的 schema 来自运行创建时冻结的图快照**（根决策不变量 1：运行只执行它冻结的文档），
不是工作流当前文档；创建运行之后编辑工作流或重新发布，不改变该运行的校验规则。

**输入期校验**：请求体中的 `input` 必须是通过 schema 的 JSON 对象；未知字段、缺失必填、类型不符、
`enum` 不匹配都拒绝，返回 `400 workflow_input_invalid`。错误形状沿用平台统一的
`{"code","params","requestId"}`，problems 放在 **`params.problems`**，不引入新的顶层形状：

```json
{"code": "workflow_input_invalid",
 "params": {"problems": [{"pointer": "<相对 input 的路径>", "reason": "<required|type|unknown_field|enum>"}]},
 "requestId": "..."}
```

problems 按 pointer 升序排列。**请求体与大小上限**：该路由单独设置 HTTP body 上限 **256 KiB**（与
Thread 消息路由相同，为语义上限留出 JSON 外壳余量）；超过上限由路由层在解码前截断并返回
`400 invalid_json`。JSON 解码成功后，`input` 以紧凑 JSON（键排序）序列化的 UTF-8 字节数上限为
**64 KiB**；超过时由输入接口在进入 schema 求值之前返回 `400 content_too_large`。空白不计入 64 KiB
但计入路由上限，因此一个合法的 64 KiB 输入不会先被默认的 64 KiB 路由上限截成 `invalid_json`，
`content_too_large` 对合法 JSON 可达。

**为什么用 JSON Schema 而不是自由文本**：自由文本无法在写入时判断「输入是否满足节点承诺」，只能把
校验推迟到下游节点运行时，错误会出现在离输入很远的地方。受支持子集覆盖首期需要的必填、类型、未知
字段与大小限制，剩余关键字在发布期明确拒绝，不制造「配置了但没生效」的假象。

### D6：输入是追加式行，`node_states` 只存最终结果

**规则**：新增追加式表 `workflow_run_inputs`，每行至少包含：`id`、`tenant_id`、`workflow_id`、
`workflow_run_id`、`node_id`、`actor_id`、`idempotency_key`、`input jsonb`、`recorded_at timestamptz`。
表只插入、不更新、不删除；一个（运行，节点）的等待至多接受一行输入（首期 `human` 节点至多执行一次、
不重试），唯一约束 `(workflow_run_id, node_id)` 固定这一点。这与[重试决策](20261021-node-retry-and-resume-contract.md)
D4 不冲突：该决策让 agent 节点按 `workflow_node_attempt` 区分多次尝试，而首期 `human` 节点不进入重试；
将来支持 human 重试或恢复时，输入行按 attempt 增加尝试代次键。每次等待最多只有一行，后续不同幂等键的
提交返回 `409 workflow_input_already_recorded`；同一个 `Idempotency-Key` 加同一请求由平台既有幂等记录
返回原响应，不产生第二行。运行页与详情读取的是输入行与 `node_states` 的组合：历史与 actor 来自输入行
（运行详情按 D1 的 `inputs` 公开），节点的最终输出只来自 `node_states[nodeId].output`。

**为什么追加而不是覆盖**：输入是「谁在什么时候给出了什么」的审计事实，覆盖会丢失 actor 与时刻，也让
幂等重放无法判定。**为什么 `node_states` 不存历史**：`node_states` 是运行推进的状态投影，节点终态必须
只有一个来源；把输入历史塞进去会让同一事实出现两份，并在恢复重算时产生分叉。

### D7：首期只做 `human` 节点，人工节点不创建 agent 会话

**规则**：本契约只覆盖 `human` 节点。 `human` 节点不派发 Node、不创建 AgentSession、不占用执行器槽位
或模型绑定、不写 `thread_commands`；它只是调度器图推进中的一个等待点，输入就是它的输出。因此本文不需要
改变单回合会话不变量，也不需要 Controller 契约。实现本文时，发布门禁只放行 `human`，`interactive: true`
的 agent 节点继续拒绝。

**interactive Agent 不在本期**：它需要同一会话的后续人类回合、desktop 的
[交互式节点完成契约](../../desktop/core/workflow/20261021-interactive-node-completion.md)与
#351/#426/#427 的收敛；这些依赖未定时实现会返工。interactive Agent 将来若需要改变「单回合、Cloud
主动结束」的生命周期，必须另开决策，本文不预设其形状。

### D8：等待时发 Issue 活动，不是线程回合

**规则**：WorkflowRun 关联了 workflow IssueRun 时，进入等待的同一次事务里追加一条 Issue 系统活动，
动作名 `run.awaiting_input`，正文包含节点 id、节点显示名与等待截止时间（`awaiting_input_at` 加配置
窗口）。Issue 活动里的截止时间是**进入等待时按当时配置算出的快照**，写入后不再随配置变化；部署调整
`workflow.hitl_wait_timeout` 后，以运行详情的 `awaitingInput.deadline` 为准（D1 按当前配置实时计算），
Issue 活动只作为当时的通知记录。输入被消费、运行回到 `running` 时，不新开线程回合、不写
`thread_commands`、不改变 IssueRun 的 `status`；IssueRun 继续按既有投影保持 `running`。直接运行（没有 IssueRun）只在运行页展示等待与截止时间
（D1 的 `awaitingInput.startedAt` / `awaitingInput.deadline`）。

**为什么发活动**：Issue 是参与者实际查看的地方；等待输入时没有活动，参与者只能靠轮询运行页才知道有人
在等。**为什么不算线程回合**：线程回合属于 agent 会话，`human` 节点没有会话；把输入记成回合会伪造一条
不存在的人类对话，破坏 Thread 的语义与统计。

### D9：公开契约只在存在写入者时声明 `awaiting_input`

**规则**：公开契约是否声明 `awaiting_input` 只取决于是否存在写入者，按三个时间点维护：

- **#450 合并后到实现本文前**：公开 OpenAPI、前端状态类型与文案都不含 `awaiting_input`；
  `workflow_runs.status` 的 CHECK 保留该值，取消接口把它当作不可取消（`409 run_not_cancellable`），
  输入接口不存在。若开始实现本文时 #450 尚未合并，实现变更不得在公开契约仍声明该状态时声称它
  「等待写入者上线前不可用」；应先合入 #450，或在同一组公开契约改动里完成「先移除、写入者上线时
  再加回」的时序。
- **实现本文时**：引入第一个写入者的同一次变更完成公开契约变更：把 `awaiting_input` 加回状态枚举、
  前端状态类型与运行页文案；按本 ADR D1/D2 发布输入接口；运行详情新增 `awaitingInput` 与 `inputs`
  两个只读字段及其 OpenAPI 子 schema（D1 的读取形状）；取消接口接受该状态，并对等待中的运行使用
  D4 的谓词。存储 CHECK 不删除，避免实现前后需要第二次迁移。
- **公开契约之外**：存储 CHECK 中的保留值不等于公开支持；只有在上述实现变更合并后，`awaiting_input`
  才是可观察的运行状态。

**为什么先移除**：没有写入者的公开枚举会被使用者读成「已支持」；契约应只声明实现能兑现的状态。
**为什么实现时再加回**：状态枚举与写入者必须在同一次变更里出现，否则又回到「枚举存在但没人写」的
误导状态。

## 不变量

1. 输入接受按 D1 的固定顺序判定：授权（403）→ 状态、节点、取消（409）→ 该等待尚无输入行（409）→
   数据库时钟早于 `awaiting_input_at + workflow.hitl_wait_timeout`（410）。判定与输入行写入在同一
   事务内完成；一个等待至多接受一行输入。
2. 等待不能永久存在：每个等待要么被输入消费、要么被取消请求、要么被有界超时终结。
3. 每个等待中的运行都被调度 pass 的独立有界扫描覆盖；等待分支先于任何节点执行证据路径执行，且会话
   证据读取只作用于 `agent` 节点。
4. 等待列成对非空当且仅当运行处于 `awaiting_input`；离开等待的每条路径都同时清空两列。
5. 输入请求只追加输入行，不写运行状态、不写 `node_states`、不推进图；节点结算只由调度器提交。
6. 已提交的输入不会被超时覆盖；已记录的取消请求不会被未消费的输入或节点结果复活。
7. 输入行不可变；`node_states` 只存节点最终结果与终态时间，不存输入历史。
8. `human` 节点不创建 AgentSession、不写 `thread_commands`、不占用执行器槽位与模型绑定，单回合会话
   不变量不变；首期 `human` 节点不重试。
9. 输入与等待中的取消只有运行发起人或当前租户管理员可以执行，与是否冻结模型绑定无关；权限从当前
   数据库状态判断，未授权调用者不能从错误差异推断运行状态。
10. 公开契约只有在存在写入者时才声明 `awaiting_input`；存储保留值不算公开支持。

## 为什么不是这些替代方案

| 方案 | 否决理由 |
| --- | --- |
| Issue 线程回复作为输入通道 | 直接运行的 WorkflowRun 没有 Issue；IssueRun 的讨论与运行控制混在同一时间线；权限仍需按运行重算，等于多一层映射 |
| 专用对话会话承载 `human` 节点输入 | 需要引入 Thread、轮次与会话生命周期，正是本期明确延后的 interactive Agent 能力；单次结构化输入不需要对话承载 |
| 不写 `awaiting_input`，从节点行推导等待状态 | Cloud 的列表、筛选与配额读取以运行状态列为准；推导会让每个读取点重复实现同一判断，也无法在运行行上表达等待截止时间 |
| 无上限等待，只能人工取消 | 运行与 Workspace 会被无限期保留；没有人处理时系统没有可解释的终态，重演「永久 `running`」的缺陷 |
| 超时后跳过该节点继续执行 | 把「没有人处理」伪装成「已批准」，下游分支可能基于不存在的输入执行；失败是可解释的终态，跳过不是 |
| 直接覆盖 `node_states` 保存最新输入 | 丢失 actor、时刻与幂等证据，无法审计也无法判断重复提交；恢复重算时同一事实有两个来源 |
| 只收自由文本，不声明 schema | 无法在写入时校验节点承诺，错误推迟到下游；调用方也无法从契约知道需要什么字段 |
| 首期同时实现 `human` 与 interactive Agent | interactive Agent 依赖 desktop 的[交互式节点完成契约](../../desktop/core/workflow/20261021-interactive-node-completion.md)与 #351、#426、#427；在完成契约与缺陷未收敛前实现，必然返工，且会改动单回合会话不变量 |
| 让 `human` 节点创建一条 agent 会话等待输入 | `human` 节点没有 agent 任务、模型调用与 Workspace 产出，创建会话只会引入无意义的执行与收尾路径，并把 HITL 与 agent 会话生命周期绑死 |
| 等待中的运行保持 `running`，只用节点状态表达等待（沿用重试决策的 `retrying` 模式） | 运行级等待没有可扫描的入口：输入、取消、超时都要先找到运行；只用节点状态会让列表、筛选与运行页继续显示「执行中」，也无法在运行行上表达等待截止时间。`awaiting_input` 是根决策已保留的运行状态，启用它比新增节点状态更小的公开面 |
| 为等待新增 `waiting` 节点状态 | 节点状态是公开契约（OpenAPI `nodeStates`），新增值要同步前端生成物与所有读取方；等待的权威事实在运行上，节点保持 `running` 已足以表示「已开始、未结束」。前提是 D2 的等待分支与 kind 限定成立 |
| 接口不做窗口与取消检查，交给下一轮 pass 判定 | 调用方会拿到 202 却永远没有节点成功；D3/D4 的可观察行为要求接口在记录前用数据库时钟拒绝取消后或窗口外的输入 |
| 无冻结绑定的运行沿用成员规则，输入对任意活跃成员开放 | human-only 图不冻结任何绑定，输入会落到任意活跃成员，与 owner 已拍板的「发起人加租户管理员」冲突；等待中的取消也会变成「不能完成、但能作废」的旁路。等待中的控制必须在与绑定无关的谓词上判定 |
| `human` 节点缺失 schema 时接受任意 JSON 对象 | 等于没有契约，无法在写入时校验节点承诺；历史快照应重新发布，而不是永久以无校验语义运行 |

## 本决策未解决的问题

- **运行页如何渲染 schema**：本期只定“提交与 API 相同形状的 JSON 对象”；把受支持子集渲染成表单控件
  （字段顺序、必填提示、数组编辑）属于产品实现，由实现运行在 UI 侧决定，不改变本契约。
- **指定处理人与通知渠道**：本期只支持发起人与租户管理员；指定处理人、邮件/IM 通知留给后续决策。
- **无冻结绑定运行的运行期取消谓词**：本文只把等待中的控制收窄为发起人或当前租户管理员；`pending` /
  `running` 的取消仍按[个人模型发起者决策](../thread/20261010-personal-model-initiator-controls-input.md)
  的成员回退。是否把所有无绑定工作流运行的控制统一到发起人或管理员，留待单独决策。
- **interactive Agent 的后续回合与完成契约**：依赖 desktop 的
  [交互式节点完成契约](../../desktop/core/workflow/20261021-interactive-node-completion.md)与
  #351、#426、#427，另开决策；本文不定义同一会话多回合的传输与结束语义。
- **逐运行覆盖等待上限**：本期只有部署级配置；如果将来需要按工作流或节点覆盖，另开决策，不在本文预设
  字段。
- **等待期间的 Workspace 复用策略**：本期保持运行 Workspace 存活到终态（D3 的上限约束保留时长）；
  是否在等待时挂起或重建 Workspace 属于资源优化，另行评估。
- **重试/循环导致同一节点再次等待**：首期 `human` 节点至多执行一次，`(workflow_run_id, node_id)` 唯一。
  [重试决策](20261021-node-retry-and-resume-contract.md) D4 已为 agent 节点引入
  `workflow_node_attempt`，但 `human` 首期不重试；将来支持 human 重试/恢复或循环时，输入行按该决策
  增加尝试键（或等待代次），届时另开决策。

## 落地顺序

- **阶段 0（本文）**：本 ADR 以 `approved` 合并，同批核心用例见
  [人工节点的运行级输入与有界等待](../../../test-cases/cloud/workflow/hitl-run-input.md)，覆盖等待进入、输入校验、
  幂等、无绑定运行的权限、409/410 顺序、取消竞争、超时、运行详情读取、Issue 活动与恢复重算；义务当前
  全部为 `Missing`。完成条件：ADR 与核心用例合并；产品行为不变。
- **阶段 1（实现，后续运行）**：新增迁移（`awaiting_input_at`、`awaiting_input_node_id`、
  `workflow_run_inputs`）、输入接口（含路由 body 上限、窗口与取消的接受判定）、调度器的独立等待扫描
  与等待分支（先于会话证据，会话读取限定 `agent` 节点）、消费/取消/超时的结算、取消接口对
  `awaiting_input` 的扩展与等待中取消的谓词、运行详情的 `awaitingInput` / `inputs` 字段、Issue 活动、
  运行页表单与文案、发布门禁放行带合法 `humanConfig.inputSchema` 的 `human`（仍拒绝
  `interactive: true`）、运行准入对历史 `human` 快照的 schema 校验。完成条件：核心用例在 cloud 侧有
  直接测试证据；ADR 改为 `implemented` 并同步实际实现的偏离。
- **阶段 2（interactive Agent，后续决策）**：在 desktop 的
  [交互式节点完成契约](../../desktop/core/workflow/20261021-interactive-node-completion.md)与 #351 收敛后，
  定义同一会话的后续人类回合、人工完成与完成声明；届时才需要修改单回合会话不变量。
- **保持不变**：首期节点白名单（除新增 `human`）、agent 会话的单回合生命周期、Controller 线上契约、
  取消与释放的既有结算、IssueRun 与 WorkflowRun 的投影关系。
