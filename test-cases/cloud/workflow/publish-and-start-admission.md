# 发布与启动门禁

依据[工作流执行根决策](../../../decisions/cloud/workflow/0-cloud-side-scheduler-owns-workflow-execution.md)（D2、D3）
与它的[执行器槽位后续决策](../../../decisions/cloud/workflow/20261015-executor-slots-replace-agent-model-ids.md)。
main 上的发布只冻结活稿、不校验图，三种入口也还没有共用的启动门禁；首期门禁由 D1 落地顺序第 2 步（#69，运行
[#132](https://github.com/ora-space/circus/issues/132)）实现，当前是未合并的 `preflight`
[cloud#65](https://github.com/ora-space/cloud/pull/65)（head `0c064a2d`）。所以本文件的证据按"实现已在评审、尚未落到 main"
如实标 `Partial`，并把其中还没实现的部分标 `Missing`。

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
| 首期不使用的 agent 配置字段被明确拒绝并点名字段 | 真实 HTTP：含 `skills` / `mcps` / `retry` 等字段的发布被拒绝并指出字段 | Missing |
| 边、槽位与 start 节点数量的结构校验 | 真实 HTTP：悬空边、未声明槽位、零个或两个 start 节点的发布拒绝 | Partial |

- 决策依据：D2（节点逐个去向表、发布门禁与服务端可执行子集），不变量 8。
- desktop 参考实现（`8909f07`）一侧：**Partial** — 拒绝不可执行节点的谓词有直接单元测试（[`crates/application/src/workflow_run/engine/tests.rs`](https://github.com/ora-space/desktop/blob/8909f07/crates/application/src/workflow_run/engine/tests.rs) 的 `first_unsupported_node_reports_tool`、`first_unsupported_node_is_none_for_supported_graphs`，[`crates/application/src/workflow_run/engine/loop_graph/tests.rs`](https://github.com/ora-space/desktop/blob/8909f07/crates/application/src/workflow_run/engine/loop_graph/tests.rs) 的 `qualification_generator_obeys_execution_graph_contract` 对全量生成场景断言 `first_unsupported_node().is_none()`）。仍缺：`WorkflowEngine::start` 调用该谓词并返回 `WorkflowValidationError::UnsupportedNodeType` 的接线没有被测试覆盖（该错误值全仓只出现在 `engine.rs:119`）；而且 desktop 只在**启动**时拒绝，没有发布门禁——"在发布时冻结一份确定可执行的图"是 Cloud 新增的义务（D2）。
- 本次直接证据：**Partial** — [cloud#65](https://github.com/ora-space/cloud/pull/65) 的 `internal/core/workflow_graph.go` 引入 `workflowExecutableNodeKinds`（只有 `start` / `agent` / `condition` / `output`）、`node_kind_unsupported` / `start_node_count` / `node_id_duplicate` 等问题码与槽位校验，`publishWorkflow` 在冻结前调用它；`internal/core/workflow_graph_test.go` 的 `TestValidateWorkflowGraph` 与 `TestRequireValidWorkflowGraphIsOneStableFault` 是直接证据。main 的 [`internal/core/workflow.go`](https://github.com/ora-space/cloud/blob/7704089d453f88d66e06d462aafe554e03a11855/internal/core/workflow.go) 今天只把活稿写进 `workflow_snapshots`，没有任何校验。
- 仍缺：**agent 配置字段的拒绝没有实现**（该 PR 没有触及 `skills` / `mcps` / `interactive` / `outputContract` / `retry` / `promptInactivity`），而 D2 要求"静默忽略会让作者以为配置生效"，这一项按 `Missing` 记；上述证据都在未合并的 `preflight` 分支上，合入 main 前不算已落地。

## Saving An Incomplete Draft Stays Allowed And Installation Is Not A Publish Gate

- 风险：把"是否已安装""映射是否完整"前移成发布门禁，作者无法保存半成品编排；或反过来，草稿被当成可执行的冻结文档。
- 前置状态：活稿正在编辑；发布者对插件安装状态与个人槽位映射的可见性可能与运行者不同。
- 触发：保存缺节点、少边、含未声明槽位的草稿；发布一份引用了当前未安装插件的图；发布者本人没有配置对应映射。
- 必须成立：保存草稿始终允许未完成的编排；发布只校验图本身（结构、节点类型、节点配置、canonical `PluginId`），不校验安装状态与个人映射；是否可跑由启动门禁在运行时判断。
- 禁止结果：用安装状态或映射挡住保存或发布、把草稿当可执行文档放行、因为发布者没配置映射而拒绝发布。
- 验证义务与证据：

| 义务 | 所需直接证据 | cloud 侧状态 |
| --- | --- | --- |
| 不完整的草稿可以保存 | 真实 HTTP：含不支持节点与非法结构的文档保存成功，只发布失败 | Partial |
| 安装状态与个人映射不是发布门禁 | 真实 HTTP：未安装插件、发布者无映射的图发布成功，随后在启动门禁被拒 | Missing |

- 决策依据：D2（发布门禁与服务端可执行子集、未完成的草稿仍可保存、安装状态不是发布门禁），不变量 8；继承 [MCP 决策](../../../decisions/desktop/core/agent/session/mcp/20260912-workflow-explicit-selection.md) D4。
- desktop 参考实现（`8909f07`）一侧：**Partial** — "安装状态不是图校验门禁"有直接证据：[`crates/backend/src/workflow/run/executor.rs`](https://github.com/ora-space/desktop/blob/8909f07/crates/backend/src/workflow/run/executor.rs) 的 `resolve_agent_ref_accepts_any_named_agent` 断言任意具名 agent（包括本 build 从未听说过的）被原样接受、未安装只在运行时报告为不可用。仍缺：desktop 没有 Cloud 的"草稿/发布"分离，所以"不完整的草稿可以保存"在参考实现里没有对应义务与测试。
- 本次直接证据：**Partial** — [cloud#65](https://github.com/ora-space/cloud/pull/65) 的 `integration/workflow_admission_test.go` 里 `TestWorkflowPublishValidatesTheGraphAndSavingDoesNot` 直接对照了"保存放行、发布拒绝"；main 的保存路径本来就不校验图。
- 仍缺：安装状态与个人映射不挡发布这一半没有任何直接证据（该 PR 没有触及安装状态）；证据同样停留在未合并的 `preflight` 分支。

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
| 输入按运行将执行的那一份文档校验 | 真实 HTTP：与所选版本不符的提交被拒且可定位到字段，改选版本后可继续 | Partial |
| 拒绝不留痕且重复确认幂等 | 真实 PostgreSQL：拒绝后 `issue_runs` 计数不变，同一 interaction 成功确认一次 | Partial |

- 决策依据：D3（三种启动方式表、三种入口走同一个启动函数），不变量 1、2。
- desktop 参考实现（`8909f07`）一侧：**Missing** — desktop 只有一个本地启动入口，没有"多个入口共用一个门禁"这件事，也没有"手动运行不能靠 UUID 形状绕过校验"的对应义务；这条完全靠 Cloud 自己的测试证明。
- 本次直接证据：**Partial** — [cloud#65](https://github.com/ora-space/cloud/pull/65) 的 `internal/core/workflow_admission.go` 提供共用的 `admitWorkflowRun`，三个调用点是 `workflow_run.go`、`issue_runs.go`、`issue_interactions.go`；`integration/workflow_admission_test.go` 的 `TestManualWorkflowRunResolvesItsExecutor` / `TestManualWorkflowRunCannotNameAForeignWorkflow` / `TestWorkflowRunValidatesAgainstTheVersionItRuns` 与 `integration/workflow_interaction_test.go` 的 `TestWorkflowConfirmRefusesAVersionTheLiveFormHasLeft` 是直接证据。main 的 [`internal/core/issue_runs.go`](https://github.com/ora-space/cloud/blob/7704089d453f88d66e06d462aafe554e03a11855/internal/core/issue_runs.go) 今天只校验 UUID 形状（注释原文 "Team/workflow keep the opaque UUID shape, resolved by ports later"）。
- 仍缺：门禁落地后仍需按 D3 在同一事务里冻结发起者与 Project（落地顺序第 4 步），本用例不把"门禁到位"等同于"身份已冻结"；证据在未合并的 `preflight` 分支上。

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
| 槽位解析与运行创建在同一事务，拒绝不留运行行 | 真实 PostgreSQL：槽位不可解析的启动被拒且运行行、绑定行都不存在 | Partial |
| 槽位从被运行的冻结文档收集 | 真实 HTTP + PostgreSQL：改活稿的槽位声明不影响已发布快照的启动判定 | Missing |

- 决策依据：D2（首期只用 `executorSlot`）、D3（逐槽位模型绑定），不变量 7；[槽位后续决策](../../../decisions/cloud/workflow/20261015-executor-slots-replace-agent-model-ids.md)与[个人槽位映射后续决策](../../../decisions/cloud/model-access/20261015-executor-slot-mappings-and-per-slot-run-bindings.md)；义务本身记在[个人槽位映射用例](../model-access/executor-slot-mappings.md#an-unresolvable-slot-is-reported-to-the-initiator-and-never-substituted)。
- 本次直接证据：**Partial** — [cloud#65](https://github.com/ora-space/cloud/pull/65) 把槽位解析放在共用的 `admitWorkflowRun` 里（三个调用点是 `internal/core/workflow_run.go`、`issue_runs.go`、`issue_interactions.go`），`integration/workflow_admission_test.go` 的 `TestWorkflowRunDefaultSlotGate` 在每次拒绝后断言只存在那一次被放行的运行，即拒绝不留下行；五个原因码与"解析发起者映射、一次列出全部缺口"的证据按上面那条链接记在个人槽位映射用例里，本文件不再重复。
- 仍缺：事务边界只有"拒绝后没有运行行"这一层间接证明，没有直接断言解析与创建同事务；"按冻结文档而不是活稿收集槽位"没有直接证据；desktop 没有对应的启动门禁，参考实现侧在这一条上同样是 `Missing`（它不是"有门禁但接线不同"，而是根本没有这层接线，可比的只有它本地启动时对具名 executor 的解析，那个不涉及发起者个人配置）；证据在未合并的 `preflight` 分支上。

## The Start Form Names The Snapshot It Freezes

- 风险：表单字段名与实际含义不符，会被固化进表单描述符与已存 interaction 输入，越晚改代价越大。
- 前置状态：`@` Confirm 的启动表单里有 key 为 `version` 的字段，其值实际是 snapshot id；历史 interaction 的 `input` 是不可变数据。
- 触发：渲染启动表单；提交确认；读取历史 interaction 的输入；把该字段与 `workflow_runs.version`（乐观并发计数器）混淆。
- 必须成立：写入一律用 `snapshotId`，与运行创建接口的请求体字段和 `workflow_runs.snapshot_id` 一致；读取时接受已存的旧 key `version`，历史 interaction 不需要改写；人类可读的版本号继续由标签（`v3 · 发布 3`）携带；`workflow_runs.version` 作为乐观并发计数器保持不变。
- 禁止结果：继续用 `version` 写入新数据、改写已存 interaction 的输入、把表单字段当成并发计数器、改名后无法读取历史输入。
- 验证义务与证据：

| 义务 | 所需直接证据 | cloud 侧状态 |
| --- | --- | --- |
| 写入使用 `snapshotId`，描述符与运行创建接口命名一致 | 真实 HTTP：表单描述符与提交后的 `issue_interactions.input` 都用 `snapshotId` | Missing |
| 读取仍接受旧的 `version` key | 真实 PostgreSQL + HTTP：用旧 key 存下的历史 interaction 仍能读取并确认 | Missing |

- 决策依据：D3（"`version` 字段的命名要修掉"段）。
- desktop 参考实现（`8909f07`）一侧：**Missing** — desktop 没有 `@` Confirm 这套启动表单与 interaction 输入，字段命名与旧 key 兼容都是 Cloud 自己的义务。
- 本次直接证据：**Missing** — main 与 [cloud#65](https://github.com/ora-space/cloud/pull/65) 都用 `fieldKeyVersion = "version"`（`internal/core/workflow_collaboration.go`），改名与兼容读取都还没有实现；`integration/workflow_interaction_test.go` 的 `TestAuthorDeclaredVersionIsNotTheVersionCursor` 只固定"作者声明的 `version` 不是版本游标"，不覆盖改名。
- 仍缺：改名、描述符与已存输入的兼容读取，属 D1 落地顺序第 2 步的完成条件。
