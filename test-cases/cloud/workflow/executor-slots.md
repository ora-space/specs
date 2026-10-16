# 执行器槽位：声明、`hint` 与旧图兼容

依据[执行器槽位决策](../../../decisions/cloud/workflow/20261015-executor-slots-replace-agent-model-ids.md)（2026-10-16 经 owner 批准）。
槽位的运行时解析、启动门禁和按槽位锁定的绑定由
[个人槽位映射与按槽位绑定用例](../model-access/executor-slot-mappings.md)维护；本文件只管图本身：槽位怎样声明、`hint`
能做什么、旧图怎样进入 cloud。引用未声明槽位的发布拒绝属于工作流根决策的发布门禁，由该领域的发布门禁用例统一维护。

证据基线：cloud#65 已随合并提交
[`df507f0`](https://github.com/ora-space/cloud/commit/df507f0) 落到 Cloud main，槽位校验与 `hint` 的形状校验在
[`internal/core/workflow_graph.go`](https://github.com/ora-space/cloud/blob/df507f0/internal/core/workflow_graph.go) 里；编辑器与导入路径还没有跟上，
仍靠演示目录 `DEMO_AGENT_MODELS` 往 `agentConfig.executor` 写 agent CLI 与模型 ID，见下面各条。desktop 不受本决策约束
（ora-space/circus#124），没有可依托的参考实现。

## Executor Slot Declarations Are Validated Before A Graph Is Published

- 风险：槽位名有大小写、空白或 Unicode 变体，运行者的映射"看起来配了却查不到"；缺说明的槽位让运行者无从判断该配什么模型；作者声明 `default` 后与隐式保留槽位冲突。
- 前置状态：图顶层有 `executorSlots`，agent 节点用 `agentConfig.executorSlot` 引用或省略槽位。
- 触发：发布含非法槽位名、重复槽位名、名为 `default` 的槽位、缺 `description` 或 `description` 超长的图；发布声明了但没有节点引用的槽位；发布显式引用或省略 `default` 的节点。
- 必须成立：槽位名满足 `^[a-z][a-z0-9_-]{0,63}$` 且图内唯一；`description` 去掉首尾空白后为 1–500 个 Unicode 字符；`default` 不能被声明，但总能被引用，省略 `executorSlot` 等于引用 `default`；声明了但未被引用的槽位不拒绝发布；每个问题都指出槽位名（以及引用它的节点）。
- 禁止结果：放行非法或重复的名字、允许声明 `default`、把省略槽位当成错误、因为未被引用的槽位拒绝发布、按字节而不是按字符限制 `description`。
- 验证义务与证据：

| 义务 | 所需直接证据 | cloud 侧状态 |
| --- | --- | --- |
| 槽位名语法、唯一性与 `default` 保留 | 单元测试 + 真实 HTTP：逐条违规的发布拒绝与问题码，且不产生快照 | Partial |
| `description` 必填，长度按 Unicode 字符计 | 单元测试：空白、恰好 500 个中文字符、501 个字符三种边界 | Covered |
| 省略或显式引用 `default`、声明但未引用的槽位都不拒绝 | 单元测试：三种合法文档通过校验 | Covered |

- 决策依据：D1、D2，不变量 2、3。
- 本次直接证据：**Covered（第二、三行）、Partial（第一行）** — [`internal/core/workflow_graph.go`](https://github.com/ora-space/cloud/blob/df507f0/internal/core/workflow_graph.go) 的槽位校验产生 `slot_name_missing` / `slot_name_invalid` / `slot_name_reserved` / `slot_name_duplicate` / `slot_description_invalid` / `slot_hint_invalid`；`internal/core/workflow_graph_test.go` 的 `TestValidateWorkflowGraph` 逐条覆盖这些违规，并覆盖"an explicit default reference"与"a slot declared for later"两个必须放行的文档。字符计数由 `utf8.RuneCountInString` 完成，边界用例是写死的字面量而不是从 `maxSlotDescription` 派生：`strings.Repeat("深", 500)` 必须通过、`strings.Repeat("深", 501)` 与 501 个 ASCII 字符必须被拒（500 个中文字符是 1500 字节，按字节计的实现从第 167 个字符起就会拒，这正是这条用例固定的回归）。
- 仍缺：**槽位名违规只有单元证据**，没有一条真实 HTTP 的发布请求带非法槽位名并断言问题码与"不产生快照"，第一行因此按 `Partial` 记（真实 HTTP 只覆盖了 `slot_not_declared`，记在[发布与启动门禁用例](../workflow/publish-and-start-admission.md#publishing-refuses-an-unsupported-node-or-configuration-and-names-it)）。**"声明了但没有节点引用的槽位只给警告"（D2）还没有实现**：校验放行了这种文档（单元用例"a declared but unreferenced slot"），但没有任何警告输出，这条义务今天也没有单独的登记行——留在这里作为缺口，等编辑器侧决定警告怎么给时再拆行。

## A Hint Never Resolves An Executor And Is Written Only By The Author

- 风险：作者在图里留下的 `hint` 被当成兜底，运行者没配置的槽位悄悄跑在作者的模型上；或导入、保存、发布自动把执行器写进 `hint`，把作者没打算公开的模型名随图共享出去。
- 前置状态：图里的槽位带或不带 `hint { agentCli, modelId }`；运行者的个人映射里没有该槽位。
- 触发：运行者启动引用该槽位的运行；作者保存、发布图；导入带旧 `executor` 字段的图；作者在编辑器里显式"用我的映射填入"或清空 `hint`。
- 必须成立：`hint` 永不参与解析，运行者没配置的槽位照样被门禁拒绝，`hint` 只随拒绝信息展示给运行者；只有作者在编辑器里的显式操作会写入或清空 `hint`；导入、保存、发布都不写入 `hint`；`hint` 不含连接；写入前提示模型名会随图共享。
- 禁止结果：用 `hint` 解析执行器或作兜底、导入时把旧执行器写进 `hint`、保存或发布时自动填充 `hint`、`hint` 里出现连接或凭据。
- 验证义务与证据：

| 义务 | 所需直接证据 | cloud 侧状态 |
| --- | --- | --- |
| `hint` 不解析执行器，运行者未配置的槽位仍被拒绝 | 真实 HTTP + PostgreSQL：带 `hint` 的槽位在运行者无映射时被拒，拒绝信息带 `hint`，且不产生运行 | Partial |
| 导入、保存、发布不写入 `hint` | 真实 HTTP + 前端：三条路径前后的图文档中 `hint` 不变 | Missing |
| 只有作者的显式操作写入或清空 `hint`，写入前有共享提示 | 前端测试：填入、清空与提示 | Missing |

- 决策依据：D3、D5，不变量 4、5；运行时"不顶替"的完整义务见[个人槽位映射用例](../model-access/executor-slot-mappings.md#an-unresolvable-slot-is-reported-to-the-initiator-and-never-substituted)。
- 本次直接证据：**Partial** — [cloud#65](https://github.com/ora-space/cloud/pull/65) 的 `integration/workflow_admission_test.go` 中 `TestWorkflowRunNamedSlotGate` 给槽位带上 `hint`，断言运行者无映射时以 `slot_not_configured` 拒绝，`hint` 只作为拒绝参数返回；`validSlotHint` 只校验形状。
- 仍缺：cloud 编辑器还没有槽位和 `hint` 的界面，导入路径仍保留 `executor`（见下一条），"不自动写入"和"作者显式写入"都没有直接证据。

## Importing A Pre-slot Graph Drops Its Executor And Runs On Default

- 风险：desktop 导出的旧图带 `agentConfig.executor`，cloud 拒收导致工作流带不过来；或者照单全收，图里继续保存作者的 agent CLI 和模型 ID，运行时又被当成执行器使用。
- 前置状态：旧图的 agent 节点带 `agentConfig.executor { agentCli, modelId }`，可能同时带 `executorSlot`；cloud 已存的图里也可能有旧字段。
- 触发：导入旧图；读取 cloud 已存的旧图并再次保存；运行由旧图发布的快照。
- 必须成立：导入接受旧图，丢掉 `agentConfig.executor`，这些节点不写 `executorSlot` 而改用 `default`；导入结果列出受影响的节点并说明改用了 `default`，旧的 agent CLI 和模型 ID 只出现在这份摘要里；同时带 `executor` 和 `executorSlot` 时以 `executorSlot` 为准；已存旧图读取时按同一规则处理，下次保存以新 schema 落盘；cloud 新保存的图里 agent 节点不含 agent CLI 和模型 ID。
- 禁止结果：因 `executor` 字段拒收、把旧执行器写进 `hint` 或自动生成槽位、运行时按旧 `executor` 选执行器、新保存的图仍带 `executor`。
- 验证义务与证据：

| 义务 | 所需直接证据 | cloud 侧状态 |
| --- | --- | --- |
| 旧图被接受，节点按 `default` 校验和解析 | 单元测试 + 真实 HTTP：带 `executor` 的图通过发布校验，启动门禁只检查 `default` | Partial |
| 导入丢掉 `executor`、列出受影响节点，`executorSlot` 优先 | 前端或真实 HTTP：导入前后的图文档与导入摘要 | Missing |
| 新保存的图不含 agent CLI 和模型 ID，已存旧图下次保存时改为新 schema | 真实 HTTP + PostgreSQL：保存后的文档不含 `executor` | Missing |

- 决策依据：D2、D5，不变量 1、5。
- 本次直接证据：**Partial** — [cloud#65](https://github.com/ora-space/cloud/pull/65) 的 `workflowReferencedSlots` 把没有 `executorSlot` 的节点（含仍带 `executor` 的旧节点）读作 `default`；`internal/core/workflow_graph_test.go` 的 `TestValidateWorkflowGraph` 用例"the pre-slot executor field is accepted"是直接证据。
- 仍缺：Cloud main 的编辑器（`frontend/src/features/workflows/editor/workflow-inspector-fields.tsx`）仍把选中的 agent CLI 与模型 ID 写成 `agentConfig.executor`，导入（`frontend/src/features/workflows/runtime/graph-import.ts`）仍按 `executor` 校验并保留它；丢弃字段、导入摘要、`executorSlot` 优先和新 schema 落盘都没有实现，属本决策落地顺序第 2、3 步。
