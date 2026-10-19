# 发布与启动门禁

依据[工作流执行根决策](../../../decisions/cloud/workflow/0-cloud-side-scheduler-owns-workflow-execution.md)（D2、D3）
与它的[执行器槽位后续决策](../../../decisions/cloud/workflow/20261015-executor-slots-replace-agent-model-ids.md)。

首期门禁由 D1 落地顺序第 2 步（ora-space/circus#69，运行 [#132](https://github.com/ora-space/circus/issues/132)）
实现，已随 [cloud#65](https://github.com/ora-space/cloud/pull/65) 合并进 main（合并提交
[`df507f0`](https://github.com/ora-space/cloud/commit/df507f0)）。下面的状态是逐条核对这个提交的实现与测试后登记的：
有直接证据的记 `Covered`，只覆盖了义务一部分的记 `Partial` 并写清缺哪一部分，本文件已没有 `Missing` 行。本次更新把
§2 第二行与 §5 第二行的措辞按实现实际能提供证据的地方收紧，理由写在各行下面。

本次更新（2026-10-19）叠加 [cloud#67](https://github.com/ora-space/cloud/pull/67) / [cloud#68](https://github.com/ora-space/cloud/pull/68) /
[cloud#72](https://github.com/ora-space/cloud/pull/72) 的堆叠：§1 第一行多了一类拒绝（condition 配置，按 [circus#202](https://github.com/ora-space/circus/issues/202)
的 owner 结论）；§4 第一行升为 `Covered`（它的两半分别由 #65 的 HTTP 用例与 #72 的绑定回滚用例覆盖），
第二行维持 `Partial`（现有用例把同一份图同时写进活稿与快照，区分不出门禁读的是哪一份）。
关于拒绝码：启动门禁的**信封码**是 `409 workflow_executor_unconfigured`，每个缺口在 `params.slots[]` 里的 `reason`
才是 `slot_not_configured`。本文件提到 `slot_not_configured` 时指的都是后者；把信封码写成 `409 slot_not_configured` 是错的。

## Publishing Refuses An Unsupported Node Or Configuration And Names It

- 风险：发布把一份服务端执行不了的图冻结成"运行的权威文档"，运行在派发时才失败，作者看不到是哪个节点、哪个字段不受支持。
- 前置状态：活稿可编辑并可含任意节点类型；首期服务端只执行 `start` / `agent` / `condition` / `output`。
- 触发：发布含 `junction` / `human` / `subflow` / `loop` / `iteration` / `tool` / `prompt` / `aggregator` / `loopExit` 的图；发布引用未声明槽位的图；发布含 `skills` / `mcps` / `interactive` / `outputContract` / `retry` / `promptInactivity` 等首期不使用配置的 agent 节点。
- 必须成立：发布按服务端首期可执行子集拒绝，并给出可定位原因（节点 id 与类型、或节点与字段名）；规范 schema 认识双方的每一种节点，所以拒绝原因是"这个节点首期不执行"而不是"未知类型"；图的边与槽位引用合法；恰好一个 start 节点；一次发布最多产生一个快照版本。
- 禁止结果：静默丢弃不支持的节点或配置后发布、把未知类型当合法、只拒绝其中一部分节点而放过其它、并发发布产生两个同号版本。
- 验证义务与证据：

| 义务 | 所需直接证据 | cloud 侧状态 |
| --- | --- | --- |
| 发布按首期子集拒绝并点名节点 | 真实 HTTP + PostgreSQL：逐类不支持节点的发布拒绝与错误参数，且不产生快照 | Partial |
| 首期不使用的 agent 配置字段被明确拒绝并点名字段 | 真实 HTTP：含 `skills` / `mcps` / `retry` 等字段的发布被拒绝并指出字段 | Partial |
| 边、槽位与 start 节点数量的结构校验 | 真实 HTTP：悬空边、未声明槽位、零个或两个 start 节点的发布拒绝 | Partial |

- 决策依据：D2（节点逐个去向表、发布门禁与服务端可执行子集），不变量 8。
- desktop 参考实现（`8909f07`）一侧：**Partial** — 拒绝不可执行节点的谓词有直接单元测试（[`crates/application/src/workflow_run/engine/tests.rs`](https://github.com/ora-space/desktop/blob/8909f07/crates/application/src/workflow_run/engine/tests.rs) 的 `first_unsupported_node_reports_tool`、`first_unsupported_node_is_none_for_supported_graphs`，[`crates/application/src/workflow_run/engine/loop_graph/tests.rs`](https://github.com/ora-space/desktop/blob/8909f07/crates/application/src/workflow_run/engine/loop_graph/tests.rs) 的 `qualification_generator_obeys_execution_graph_contract` 对全量生成场景断言 `first_unsupported_node().is_none()`）。仍缺：`WorkflowEngine::start` 调用该谓词并返回 `WorkflowValidationError::UnsupportedNodeType` 的接线没有被测试覆盖（该错误值全仓只出现在 `engine.rs:119`）；而且 desktop 只在**启动**时拒绝，没有发布门禁——"在发布时冻结一份确定可执行的图"是 Cloud 新增的义务（D2）。
- 本次直接证据：**Partial** — cloud#65 的 [`internal/core/workflow_graph.go`](https://github.com/ora-space/cloud/blob/df507f0/internal/core/workflow_graph.go) 引入 `workflowExecutableNodeKinds`（只有 `start` / `agent` / `condition` / `output`）、`workflowUnsupportedAgentConfigFields` 与 `node_kind_unsupported` / `node_config_unsupported` / `start_node_count` / `node_id_duplicate` / `edge_endpoint_unknown` / `slot_not_declared` 等问题码，`requireValidWorkflowGraph` 在 `publishWorkflow` 冻结之前调用它，拒绝时一个问题码携带全部问题、一个快照也不写。真实 HTTP 的直接证据是 [`integration/workflow_admission_test.go`](https://github.com/ora-space/cloud/blob/df507f0/integration/workflow_admission_test.go) 的 `TestWorkflowPublishValidatesTheGraphAndSavingDoesNot`（含 `tool` 节点且无 start 的图发布被拒，问题里同时点名节点 id 与 kind，`workflow_snapshots` 计数为 0）与 `TestWorkflowPublishRefusesUnusedAgentConfiguration`（带 `retry` 与 `skills` 的图发布被拒，问题列表逐项比对字段名，改回首期配置后同一份图发布成功）；单元矩阵是 [`internal/core/workflow_graph_test.go`](https://github.com/ora-space/cloud/blob/df507f0/internal/core/workflow_graph_test.go) 的 `TestValidateWorkflowGraph`（33 个用例）与 `TestRequireValidWorkflowGraphIsOneStableFault`（线协议：一个稳定错误码 + 逐节点参数）。[cloud#67](https://github.com/ora-space/cloud/pull/67) 按 [circus#202](https://github.com/ora-space/circus/issues/202) 的 owner 结论补上 condition 配置门禁：`workflowUnsupportedConditionConfigs` 的 9 个码各带节点与字段，`TestWorkflowPublishRefusesUnsupportedConditionConfigs` 把用例与该清单**双向**核对（清单里声明的每个码都要有用例产出，用例产出的每个码也都必须在清单里），并断言编辑器会正常发布的几种图不被误拒（填好的条件节点、加了还没配置的条件节点）；清单会随 [circus#210](https://github.com/ora-space/circus/issues/210) 的语义对齐工作逐项缩短。
- 仍缺：**节点种类只逐类验了 `tool` 一种**（HTTP 与单元各一处，触发里列的另外 8 种没有各自的发布拒绝用例），把执行不了的种类加进 `workflowExecutableNodeKinds` 时没有测试会挂，第一条义务因此按 `Partial` 记。**配置字段只有 `skills` 与 `retry` 出现在真实 HTTP 证据里**，`mcps` / `interactive` / `outputContract` / `promptInactivity` 的单元用例由实现自己读的那份字段清单循环生成（`for _, field := range workflowUnsupportedAgentConfigFields`），字段从清单里掉出去时用例跟着一起消失，第二条义务因此按 `Partial` 记。单元里覆盖了悬空边（`edge_endpoint_unknown`）与两个 start 节点，二者都没有真实 HTTP 证据，第三条义务因此按 `Partial` 记。

## Saving An Incomplete Draft Stays Allowed And Installation Is Not A Publish Gate

- 风险：把"是否已安装""映射是否完整"前移成发布门禁，作者无法保存半成品编排；或反过来，草稿被当成可执行的冻结文档。
- 前置状态：活稿正在编辑；发布者对插件安装状态与个人槽位映射的可见性可能与运行者不同。
- 触发：保存缺节点、少边、含未声明槽位的草稿；发布一份引用了当前未安装插件的图；发布者本人没有配置对应映射。
- 必须成立：保存草稿始终允许未完成的编排；发布只校验图本身（结构、节点类型、节点配置、canonical `PluginId`），不校验安装状态与个人映射；是否可跑由启动门禁在运行时判断。
- 禁止结果：用安装状态或映射挡住保存或发布、把草稿当可执行文档放行、因为发布者没配置映射而拒绝发布。
- 验证义务与证据：

| 义务 | 所需直接证据 | cloud 侧状态 |
| --- | --- | --- |
| 不完整的草稿可以保存 | 真实 HTTP：含不支持节点与非法结构的文档保存成功，只发布失败 | Covered |
| 安装状态与个人映射不是发布门禁 | 真实 HTTP：引用任意 `agentCli`（hint 或旧 `executor`）的图、发布者没有映射的图都发布成功，前者在运行里按 `default` 解析、后者在启动门禁被拒 | Covered |

- 决策依据：D2（发布门禁与服务端可执行子集、未完成的草稿仍可保存、安装状态不是发布门禁），不变量 8；继承 [MCP 决策](../../../decisions/desktop/core/agent/session/mcp/20260912-workflow-explicit-selection.md) D4。
- desktop 参考实现（`8909f07`）一侧：**Partial** — "安装状态不是图校验门禁"有直接证据：[`crates/backend/src/workflow/run/executor.rs`](https://github.com/ora-space/desktop/blob/8909f07/crates/backend/src/workflow/run/executor.rs) 的 `resolve_agent_ref_accepts_any_named_agent` 断言任意具名 agent（包括本 build 从未听说过的）被原样接受、未安装只在运行时报告为不可用。仍缺：desktop 没有 Cloud 的"草稿/发布"分离，所以"不完整的草稿可以保存"在参考实现里没有对应义务与测试。
- 本次直接证据：**Covered** — 保存与发布的分离由 [`integration/workflow_admission_test.go`](https://github.com/ora-space/cloud/blob/df507f0/integration/workflow_admission_test.go) 的 `TestWorkflowPublishValidatesTheGraphAndSavingDoesNot` 直接对照：含 `tool` 节点且无 start 的图 `POST /workflows` 200 并原样存下、`PUT` 200，同一份文档发布 400；"安装状态与映射不是发布门禁"的两半也都在真实 HTTP 上：`TestWorkflowRunNamedSlotGate` 发布一份 hint 里写着 `official/ora-space.opencode` / `vendor/model-v1` 的图成功（`hint` 按槽位后续决策 D3 只是展示用，发布不校验任何安装集合），随后同一个图因为发起者没有该槽位的映射在启动门禁被拒（409 `workflow_executor_unconfigured`，该槽位在 `params.slots[]` 里的 `reason` 是 `slot_not_configured`）；`TestWorkflowRunDefaultSlotGate` 反复改个人模型连接的可用性，发布过的那份图始终能启动到门禁为止，被拒只发生在启动这一步。
- 说明（本行措辞收紧）：首期图只通过 `executorSlot` 引用执行器，旧图的 `agentConfig.executor` 被接受并丢弃（槽位后续决策 D5），所以 Cloud 的图里没有可以表达"某个插件已安装/未安装"的位置；这条义务在 Cloud 上的形态因此是"发布不看任何安装或绑定状态，可跑性留给启动门禁判断"，上面的两处 HTTP 证据正是这个形态。
- 仍缺：无。

## Every Entry To A Workflow Run Passes One Admission Gate

- 风险：三种入口各自校验，规则不一致——某一入口用任意 UUID 就能造出绕过存在性、租户、归档与发布状态的工作流运行，或者放行一份与提交值不符的图。
- 前置状态：直接运行、issue 里的 manual workflow IssueRun、`@` Confirm 三种入口都存在，且都会创建运行。
- 触发：manual 入口给一个任意 UUID 或别的 workflow 的 id；`@` Confirm 提交的表单值与所选版本或活稿不一致；同一 interaction 重复确认；对已归档的 workflow 启动运行。
- 必须成立：三种入口走同一套门禁（同一个启动函数），都按同一份冻结文档判断存在性、租户、归档状态与发布状态并校验输入；被拒的启动不留下运行行、不占用 interaction、不产生活动；重复确认幂等；manual 入口不能再用 UUID 形状绕过校验。
- 禁止结果：某入口跳过门禁、按活稿而不是所选版本校验、被拒后留下可派发的运行行、重复确认产生第二个运行。
- 验证义务与证据：

| 义务 | 所需直接证据 | cloud 侧状态 |
| --- | --- | --- |
| 三种入口共用同一个启动门禁 | 真实 HTTP + PostgreSQL：同一份非法图在三个入口同样被拒，拒绝后无运行行 | Partial |
| 输入按运行将执行的那一份文档校验 | 真实 HTTP：与所选版本不符的提交被拒且可定位到字段，改选版本后可继续 | Covered |
| 拒绝不留痕且重复确认幂等 | 真实 PostgreSQL：拒绝后 `issue_runs` 计数不变，同一 interaction 成功确认一次 | Covered |

- 决策依据：D3（三种启动方式表、三种入口走同一个启动函数），不变量 1、2。
- desktop 参考实现（`8909f07`）一侧：**Missing** — desktop 只有一个本地启动入口，没有"多个入口共用一个门禁"这件事，也没有"手动运行不能靠 UUID 形状绕过校验"的对应义务；这条完全靠 Cloud 自己的测试证明。
- 本次直接证据：**Partial** — [`internal/core/workflow_admission.go`](https://github.com/ora-space/cloud/blob/df507f0/internal/core/workflow_admission.go) 的 `admitWorkflowRun` 是唯一的启动门禁（解析要执行的快照、按它的图收集并解析槽位），三个调用点是 `workflow_run.go`、`issue_runs.go`、`issue_interactions.go`。真实 HTTP 的证据按入口分开：manual 入口 `TestManualWorkflowRunResolvesItsExecutor`（未知 id 404、没有已发布版本 400 `workflow_no_published_snapshot`、归档 404，且被拒的那次不留运行行）与 `TestManualWorkflowRunCannotNameAForeignWorkflow`（别的租户的 workflow 同样 404，`issue_runs` 计数为 0）；直接运行入口 `TestWorkflowRunSnapshotIsScopedToItsWorkflow`（别的 workflow 的 snapshot 与未知 id 都是 404，且不回退到最新版本）与 `TestWorkflowRunValidatesAgainstTheVersionItRuns`（旧版本只按自己的 schema 收，未指名版本时用最新版本，运行行记下执行的 `snapshot_id`）；`@` Confirm 入口 `TestWorkflowConfirmRefusesAVersionTheLiveFormHasLeft`（提交与所选版本不符时 409 并点名字段，不创建运行、不占用 interaction，改发布或改选版本后同一次提交通过）；幂等由 `TestWorkflowConfirmIdempotencyAndDoubleConfirm` 证明（同 key 重放返回同一个运行、同 key 不同体 409、已确认的 interaction 换 key 409，`issue_runs` 只有一个）。
- 仍缺：没有任何一个用例把**同一份**非法文档依次送进三个入口并断言同样的拒绝；"三种入口同一个门禁"今天由"一个 `admitWorkflowRun` + 三个调用点"与各入口各自的拒绝用例支撑，按 `Partial` 记。另外，门禁落地后仍需按 D3 在同一事务里冻结发起者与 Project（落地顺序第 4 步），本用例不把"门禁到位"等同于"身份已冻结"。

## A Slot The Initiator Cannot Supply Refuses The Run Instead Of Substituting

**本用例只维护工作流侧的门禁接线**：启动门禁在创建工作流运行的事务里，按本次要执行的那一份冻结文档收集槽位并
逐槽位解析，解析失败就拒绝整次启动。解析谁（发起者）的映射、一次列出全部缺口、绝不顶替、`default` 是隐式保留
槽位、四种入口各自的发起者身份、定时任务的通知与不重试，这些**语义与它们的证据统一由[个人槽位映射用例](../model-access/executor-slot-mappings.md#an-unresolvable-slot-is-reported-to-the-initiator-and-never-substituted)维护**，
本文件不重复登记。

- 风险：工作流侧绕开启动门禁创建运行——按活稿而不是被运行的文档收集槽位，或在解析失败后仍留下运行行与绑定行，
  于是"槽位不可解析就拒绝"在图之外被削弱。
- 前置状态：图已发布并引用 `default` 与命名槽位；三种入口都能创建运行；槽位解析与运行创建在同一个事务里。
- 触发：用一个无法解析的槽位启动；改活稿的槽位声明后启动已发布的快照；启动被拒后检查是否留下运行行。
- 必须成立：槽位从**本次运行将执行的那一份冻结文档**收集，与活稿无关；解析在创建事务里完成，任一项失败则整个
  启动被拒，不产生运行行与绑定行，也不留下可供派发的活动。
- 禁止结果：按活稿收集槽位、先创建运行再解析槽位、解析失败后留下运行行或绑定行。
- 验证义务与证据：

| 义务 | 所需直接证据 | cloud 侧状态 |
| --- | --- | --- |
| 槽位解析与运行创建在同一事务，拒绝不留运行行 | 真实 PostgreSQL：槽位不可解析的启动被拒且运行行、绑定行都不存在 | Covered |
| 槽位从被运行的冻结文档收集 | 真实 HTTP + PostgreSQL：改活稿的槽位声明不影响已发布快照的启动判定 | Partial |

- 决策依据：D2（首期只用 `executorSlot`）、D3（逐槽位模型绑定），不变量 7；[槽位后续决策](../../../decisions/cloud/workflow/20261015-executor-slots-replace-agent-model-ids.md)与[个人槽位映射后续决策](../../../decisions/cloud/model-access/20261015-executor-slot-mappings-and-per-slot-run-bindings.md)；义务本身记在[个人槽位映射用例](../model-access/executor-slot-mappings.md#an-unresolvable-slot-is-reported-to-the-initiator-and-never-substituted)。
- 本次直接证据：**第一行 Covered，第二行 Partial** — [`internal/core/workflow_admission.go`](https://github.com/ora-space/cloud/blob/df507f0/internal/core/workflow_admission.go) 的 `admitWorkflowRun` 先取要执行的快照行，再用 `requireWorkflowExecutorSlots(s.O("graph"), uid)` 按**快照里的图**收集并解析槽位，三个调用点都在创建运行的那个事务里。第一行的两半各有直接用例，归属分开：[cloud#65](https://github.com/ora-space/cloud/pull/65) 的 `TestWorkflowRunDefaultSlotGate` 与 `TestWorkflowRunNamedSlotGate` 在每次拒绝后都断言 `workflow_runs` 只存在被放行的那几次（前者 1 次、后者 0 次），这是**不留运行行**那一半；[cloud#72](https://github.com/ora-space/cloud/pull/72) 的 `TestWorkflowRunGateRefusesUnresolvedSlots` 断言同一个事务回滚后 `run_model_bindings` 里 0 条**绑定行**，这是另一半。该用例的 `seedSlotGateScene` 预先插入了 `workflow_runs` 行，它不断言运行行不存在，所以"不留运行行"不要归给 #72。`TestWorkflowRunGateFreezesBindings` 证明两个被引用的槽位各冻结一组绑定，但它的图是同一份 JSON 同时写进 `workflows.graph` 与 `workflow_snapshots.graph`，实现就算读活稿它也会通过，因此它不是第二行的直接证据。五个原因码与"解析发起者映射、一次列出全部缺口"的证据按上面那条链接记在个人槽位映射用例里，本文件不再重复。
- 仍缺：**"改活稿的 `executorSlots` 声明再启动一份已发布快照"的 HTTP 对拍**仍没有用例。第二行"所需直接证据"栏写的判据就是这一条，现有证据只是 DB 层"从快照行读 `graph`"，两者不等价，所以第二行记 `Partial`——判据是那一栏，不用"收集来源是快照行"重新解释。`TestWorkflowRunValidatesAgainstTheVersionItRuns` 对拍的是必填**变量**（活稿加了 `added-later` 后对旧版本提交照样通过），不是槽位声明，补不上这一条。要升 `Covered`，需先在 [cloud#72](https://github.com/ora-space/cloud/pull/72) 补一条"发布 v1 → 修改活稿的 `executorSlots` → 仍用 v1 启动"的用例。desktop 没有对应的启动门禁，参考实现侧在这一条上同样是 `Missing`（它不是"有门禁但接线不同"，而是根本没有这层接线，可比的只有它本地启动时对具名 executor 的解析，那个不涉及发起者个人配置）。

## The Start Form Names The Snapshot It Freezes

- 风险：表单字段名与实际含义不符，会被固化进表单描述符与已存 interaction 输入，越晚改代价越大。
- 前置状态：`@` Confirm 的启动表单里有 key 为 `snapshotId` 的字段，其值就是 snapshot id（改名前的 key 是 `version`）；已发布快照的图与历史 interaction 的 `input` 都是不可变数据。
- 触发：渲染启动表单；提交确认（包括改名前后端渲染、改名后提交的表单，提交里带旧 key）；在同一份表单上调用 AI Assist（确认之前，读的是同一份提交）；读取历史 interaction 的输入；把该字段与 `workflow_runs.version`（乐观并发计数器）混淆。
- 必须成立：写入一律用 `snapshotId`，与运行创建接口的请求体字段和 `workflow_runs.snapshot_id` 一致；读取时接受旧 key `version`——提交里带它就按它解析版本并折叠成 `snapshotId`，作者自己把 `version` 或 `snapshotId` 声明成变量时那个值仍是作者的；历史 interaction 不需要改写（`input` 只在确认时写一次，此后只读）；人类可读的版本号继续由标签（`v3 · 发布 3`）携带；`workflow_runs.version` 作为乐观并发计数器保持不变。
- 禁止结果：继续用 `version` 写入新数据、改写已存 interaction 的输入、把表单字段当成并发计数器、改名后不认旧 key 的提交（按旧 key 提交会被当成未知字段拒绝，或让用户选的版本被丢掉而悄悄跑最新版本）。
- 验证义务与证据：

| 义务 | 所需直接证据 | cloud 侧状态 |
| --- | --- | --- |
| 写入使用 `snapshotId`，描述符与运行创建接口命名一致 | 真实 HTTP：表单描述符与提交后的 `issue_interactions.input` 都用 `snapshotId` | Covered |
| 读取仍接受旧的 `version` key | 真实 HTTP + PostgreSQL：按旧 key 提交的确认被接受、用户选的版本被执行、interaction 行只存 `snapshotId`；同一份提交的 Assist 也被接受；改名前发布的快照里存的字段声明也按新名字读出 | Covered |

- 决策依据：D3（"`version` 字段的命名要修掉"段）。
- desktop 参考实现（`8909f07`）一侧：**Missing** — desktop 没有 `@` Confirm 这套启动表单与 interaction 输入，字段命名与旧 key 兼容都是 Cloud 自己的义务。
- 本次直接证据：**Covered（两行）** — cloud#65 把启动字段的 key 改成 `snapshotId`（[`internal/core/workflow_collaboration.go`](https://github.com/ora-space/cloud/blob/df507f0/internal/core/workflow_collaboration.go) 的 `fieldKeySnapshotID`，旧名由 `legacyFieldKeyVersion` 承载、`foldLegacyVersionAnswer` 折叠）。第一行：`integration/workflow_snapshot_form_test.go` 的 `TestWorkflowSnapshotFormProjectsTheChosenVersion` 与 `integration/workflow_interaction_test.go` 的 `TestAuthorDeclaredSnapshotIDIsNotTheVersionCursor` 断言表单描述符的版本选择器以 `snapshotId` 出现（作者自己声明了 `snapshotId` 时那个控件是作者的、平台不再插入选择器），`TestWorkflowConfirmCarriesPlatformFields` 与 `TestWorkflowConfirmReadsThePreRenameVersionKey` 断言提交后的 interaction 行按 `snapshotId` 存，运行创建接口的请求体字段同样是 `snapshotId`（`TestWorkflowRunSnapshotIsScopedToItsWorkflow`）。第二行：`TestWorkflowConfirmReadsThePreRenameVersionKey` 断言按旧 key 提交的确认被执行（跑的是用户选的那个版本）、interaction 行读回来只有 `snapshotId`，作者声明了 `snapshotId` 时作者的值不被覆盖；改名前的快照里存着的字段声明由 [`internal/core/workflow_collaboration.go`](https://github.com/ora-space/cloud/blob/df507f0/internal/core/workflow_collaboration.go) 按旧 key 读出并落到新名字（`TestLaunchFieldsDeclarationAcceptsThePreRenameKey` 断言撤回的字段没有复活、要求的必填性没有丢）；Assist 一侧由 `TestWorkflowAssistReadsThePreRenameVersionKey`（[ora-space/cloud#78](https://github.com/ora-space/cloud/pull/78)）断言同一份改名前的提交先走 Assist 再走 Confirm 都被接受、运行的 `workflowSnapshotId` 仍是用户选的那份，未声明的 key 仍被拒（证明前一条通过是"读了旧 key"而不是"不再校验 key 集合"），作者自己声明 `version` / `snapshotId` 的两种图也按同一规则处理；折叠规则本身（旧 key 被改写、已有新 key 时以新 key 为准、作者声明的旧 key 不动、没有旧 key 时不动）由 `internal/core/workflow_collaboration_test.go` 的 `TestFoldLegacyVersionAnswer` 逐例覆盖。
- 行文修正：第二行原写"用旧 key 存下的历史 interaction 仍能读取并确认"。这个形态在 Cloud 里不可达，也没有被断言的对象：`issue_interactions.input` 只在确认那一次写入（`UPDATE ... WHERE run_id IS NULL`），而折叠就发生在同一次确认里，所以带旧 key 的行是已经被确认、已经跑过的行——它不会被再确认，也不会被改写；改名前的构建写下的那种行同样只会被原样读出（读接口按存储原样返回 `input`，没有任何读取路径再去解释它的 `version` 键）。旧 key 真正还能出现的地方是**提交**（渲染早于改名的表单在改名后提交；Confirm 与 Assist 读的是同一份提交）与**改名前发布的快照里的字段声明**，行文按这两处收紧。
- 仍缺：无。assist 这条缺口由 [ora-space/cloud#78](https://github.com/ora-space/cloud/pull/78) 补上：`assistWorkflow` 在 `validateFormValues` 之前调用 Confirm 用的同一个 `foldLegacyVersionAnswer`，兼容规则只有一处定义（此前 assist 解析**活稿**的描述符后直接校验，同一个改名前的表单确认能过、点 AI Assist 被当成未知字段拒，400 `invalid_field_value`）。记一条**测试可观测性**的边界，它不是系统缺口：assist 无状态、只回建议，既不持久化折叠后的值也不回显收到的值，所以"两个 key 同时提交时哪个胜出"在 HTTP 层观察不到——这一半由 `TestFoldLegacyVersionAnswer` 的单元用例覆盖，而 assist 现在调的就是那个函数，规则不可能与 Confirm 不同。
- 说明：`workflow_runs.version` 是与 `workflows.version` 一致的乐观并发计数器，不在本用例内，保持不动。
