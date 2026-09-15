---
status: approved
---

# 迭代节点：以 foreach 复合运行时遍历数组，用按轮账本解耦错误策略

迭代节点（foreach 语义）成为[节点运行时编排决策](0-node-runtime-orchestration.md)规划的第一个复合运行时：它以一个数组变量为迭代源，区域内的节点图对每个元素执行一轮，每轮绑定 `{iter}.item` / `{iter}.index`，完成时产出类型固定的暴露变量 `{iter}.output`、`{iter}.entries` 与 `{iter}.failed_count`。错误容忍不以 Dify 的三模式下拉建模，而解耦为二值控制流开关（`fail` / `continue`）加按轮账本：账本记录每轮的 `{item, status, output, error}`，`entries` 是它的完整投影，`output` 是只含成功项的过滤投影——Dify 三模式成为"策略 × 读哪个变量"的组合，而不是改变输出类型的配置项。失败传播获得作用域化的吸收能力：区域内失败可被 `continue` 策略吸收为账本条目，迭代节点自身的失败仍然终止 run。

本决策已完成评审并标记为 `approved`：依据已 `approved` 的 0 号决策的区域机制与复合运行时契约细化，评审期间的修正（D5 账本编码由序列改为按轮索引、D2 把 `max(iteration)` 推导收窄为 v1 串行实现机制并补充 advance 协调语义）均已并入正文；核心测试用例已依据最终同意的不变量补充于 `test-cases/desktop/core/workflow/`（新增 `iteration-node.md`，并同步更新 `region-and-loop.md`、`stateless-recovery-and-hitl.md`、`variable-pool-boundary.md`、`runtime-forms.md`），证据缺口如实标记，待各落地阶段代码实现后回填。涉及模块为 `crates/application/src/workflow_run/engine`（`node_type.rs`、`node_runtime.rs`、`branch_projection.rs`、`variable_pool.rs`、`ports.rs`、`skill_delivery.rs`）、`crates/backend/src/workflow/run`（`executor.rs`、`recovery.rs`）、`crates/db/src/repository/workflow_run_engine.rs`、`ora-contracts`（`WorkflowNodeRun`）与前端 workflow-editor / workflow-run。兼容性边界：只含既有四种节点类型的冻结图执行行为不变（0 号决策不变量 9 继承）；旧版本 Ora 打开含迭代节点的图由 `UnknownNodeType` 解析拒绝，不产生半解析状态。本决策不承担 while 语义 Loop、并行迭代、嵌套复合、失败轮重试与区域内 HITL——各自留在后续决策，见"本决策未解决的问题"。

## 继承与修改

| 0 号决策约定 | 处理 | 当前规则 |
| --- | --- | --- |
| D1：引擎核心只调度，节点策略外置注册表 | 继承，不变 | 迭代节点 = 一个复合运行时 + 注册表一行；`run_schedule` 不出现节点类型字面量 |
| D2：运行时按 swift / async / 复合三形态分类，统一 sink 回报 | 继承，落地 | 复合形态与 advance 续算协调（0 号决策 D5 预留的唯一新增协调面）随本决策进入实现 |
| D3：区域包含，六条边界校验 | 继承，微调 | 六条规则不变；第六条读作"迭代节点必须声明 `1 ≤ maxIterations`" |
| D4：Loop（while 语义）是第一个复合运行时 | 修改 | 第一个复合运行时改为迭代节点（foreach 语义）；数组长度即轮数，无需 `exit_when`；`max_iterations` 演化为节点级安全上限并在启动边界检查；`iteration` 列、每轮变量注入、区域输出覆盖写、boot sweep 交还 Running 复合行等承诺全部继承。while-Loop 移至后续 ADR |
| D5：区域内 HITL 沿现有交互路径 | 修改 | v1 结构校验禁止区域内 interactive 节点；机制与协调义务保留，二期放开 |
| D6：后续节点以运行时实例扩展 | 继承 | 迭代节点是第一个实例，验证"新节点 = 运行时 + 注册一行"的扩展契约 |
| D7：失效事件广播 | 继承，不变 | 轮次进度经持久化行重查获得 |
| 未解决问题：区域内 Condition 决策按轮持久化 | 本决策固定 | 按轮记录、迭代投影只消费当前轮（D5） |
| 未解决问题：节点级错误策略 | 落地最小切片 | `FailurePropagation` 限定在复合区域作用域内，不泛化到任意节点（D6） |
| 未解决问题：并行迭代 | 仍留后续 | v1 串行；账本按轮索引、无需为并行迁移编码，隔离语义见"未解决" |

## 决策

### D1：迭代节点是第一个复合运行时，foreach 语义

wire 节点类型为 `iteration`（`NodeType::Iteration`；前端现有原型 `loop` kind 保留给未来 while-Loop，两者不混用）。节点配置携带四个字段：

```json
{
  "iteratorSelector": ["start", "prs"],
  "collectSelector": ["review", "structured_output"],
  "errorStrategy": "continue",
  "maxIterations": 50
}
```

- `iteratorSelector`：迭代源变量选择器，解析结果必须是 JSON 数组；数组长度即轮数。
- `collectSelector`：输出收集目标，v1 必须是根变量（无嵌套路径），且指向区域内某节点声明并拥有的变量；其声明类型 T 决定 `{iter}.output` 的元素类型。
- `errorStrategy`：`fail`（默认）或 `continue`，只控制"某轮失败后是否继续"（D4）。
- `maxIterations`：安全上限，`≥ 1`，默认 50，节点级可配置（D2）。

执行语义：节点启动时解析迭代源得到 N 个元素；第 r 轮（`0 ≤ r < N`）把 `items[r]` 绑定到 `{iter}.item`、r 写入 `{iter}.index`，然后用迭代投影（scope = 区域，iteration = r）驱动区域内节点；每轮结算后进入下一轮；全部轮结算后节点完成。空数组是合法输入：节点立即成功，`output` / `entries` 为空数组、`failed_count` 为 0。

图解析校验：`iteratorSelector` 指向的变量声明类型必须是数组类型（`array` / `array[any]` / `array[T]`）；`collectSelector` 必须落在区域内且指向已声明变量。`{iter}.item` 的声明类型由迭代源的声明数组类型派生（`array[string]` → `string`；无类型数组 → `any`），保证图解析时静态可判。

为什么第一个复合运行时是 foreach 而不是 0 号决策原定的 while-Loop：批量处理（对一组 PR 逐个审查、对一组文档逐个翻译、对一批失败测试逐项修复）是编排场景的高频需求；更关键的是 foreach 的轮数在启动时就是已知事实，使安全上限可以在启动边界零成本检查（D2），而 while 必须在运行中判定终止。两者共享全部区域基础设施，顺序调换没有架构代价；while-Loop 的 `exit_when` 求值编排随其自身 ADR 后置。

### D2：轮次是持久化事实；上限在启动边界检查

- `workflow_node_runs` 增加 `iteration` 列（整数；外层行为 NULL，区域行为轮次，与 `{iter}.index` 同值、0 起）。同一节点多轮执行 = 多行，表结构已容纳（0 号决策 D4）。
- `{iter}.item` / `{iter}.index` 的 writer 均为迭代节点。每轮开始时，`item` / `index` 写入与该轮首批节点行的启动在同一个 SQLite 事务提交——崩溃后要么整轮未开始，要么轮变量与轮内行一致。
- v1 串行执行下，当前轮从区域行 `max(iteration)` 推导，引擎不为迭代持有任何内存态。该推导是串行实现机制而非语义承诺——不变的承诺是"轮次进度恒可从持久化行与账本重算"，并行迭代（见"未解决"）下同一承诺以在途轮集合的形式成立，无需改写本不变量。
- 崩溃后 boot sweep 把 Running 的复合行交还运行时续算（0 号决策 D4 承诺；这修改了现有 sweep 对非终态行一律标记 `interrupted_by_restart` 失败的行为，但仅对复合行生效）：被打断的当轮行已由 sweep 标记失败，续算时按"该轮失败"结算（D4）。取消沿现有 run 级取消路径，晚到的轮内回调由既有幂等拒绝兜底。
- advance 协调面：每次调度波把 Running 的迭代节点行交还其运行时续算（0 号决策 D5 预留的唯一新增协调义务，随本决策进入实现）。advance 是纯重算——从持久化行与账本推导当前轮与轮内缺口，不携带任何内存态；对无缺口的调用是 no-op。区域内完成回报（二期含人工完成）触发的调度波由此自动推进轮次，不需要新通知通道。
- 区域内节点输出按 selector 覆盖写（继承 `pool.set` 语义）：第 r 轮完成后 `{{#fix.output#}}` 解析为第 r 轮的值。
- 安全上限：`N > maxIterations` 时节点在启动边界失败——不执行任何轮，错误信息包含数组长度、上限值与调整指引。绝不静默截断：用户以为 500 项全部处理、实际只有 50 项而运行显示成功，这是正确性灾难，违反本仓库 fail-loud 的基本哲学（模板引用一个未赋值变量都是硬错误）。也不跑到上限才失败（0 号决策给 while 设计的行为）：那已消耗 N 个 Agent session 才发现问题，而 foreach 的 N 先知让先检查零浪费。迭代源来自 Start 输入时，部署界面可提前给出软警告，但权威检查在节点启动边界。

### D3：显式输出收集与类型稳定的暴露变量

完成时迭代节点暴露三个变量，声明在图解析时的变量目录中，**类型不随 `errorStrategy` 改变**：

| selector | 类型 | 语义 |
| --- | --- | --- |
| `{iter}.output` | `array[T]` | 仅成功轮的收集值，按输入顺序 |
| `{iter}.entries` | `array[object]` | 完整账本：每轮一个 `{item, status, output, error}` 信封，与输入数组位置对齐 |
| `{iter}.failed_count` | `number` | 失败轮数 |

这修改了"所有产数据节点声明单数 `.output` 且类型为 string"的现有规则（`WorkflowVariablePool::from_graph` 对非 Start/Condition 节点一律声明 `.output: string`）：迭代节点按自身配置声明 `.output: array[T]` 及其余暴露变量。`.output` 这个名字保留——与本仓库"每个产数据节点暴露 `{node}.output`"的惯例和 Dify 迭代节点的输出命名同时对齐，Dify 迁移用户无需学新名字。

暴露变量在完成时**从账本派生写入**（数组与计数），不经由文本 `output` 的池写入路径——后者写的是字符串，无法通过 `array[T]` 的类型检查。迭代节点自身 node-run 行的 `output` 列保存 `output` 数组的 JSON 序列化，仅用于展示与审计；完整对话历史属于各轮 Agent 的 Ora session（继承 0 号决策不变量 8：session 归属 Agent 运行时）。

轮结算做**结构性检查**：收集目标在本轮存在 `Succeeded` 的节点行，才收集其池值。反例：body 内有 Condition，第 r 轮恰好走了绕开收集目标的分支——此时池里残留的是第 r-1 轮的值，`pool.resolve(collect)` 不会报错而是返回陈旧值；结构性检查以"本轮行"为准，陈旧值永不冒充本轮产出。收集目标本轮未运行，v1 记为该轮失败（"collect target did not run this round"）；`skipped` 作为第三种 status 会把信封判别器从二值变三值并牵动下游 Condition 写法，留给后续。

### D4：错误策略解耦——二值控制流开关 + 按轮账本

`errorStrategy` 只回答"某轮失败后，迭代节点停不停"：

- `fail`（默认）：第一个失败轮 → 迭代节点失败 → run 失败。与现有引擎行为一致。
- `continue`：失败轮记入账本，继续执行剩余轮；节点最终照常完成，失败信息保留在 `entries` 与 `failed_count` 中。

Dify 的三模式成为"策略 × 读哪个变量"的组合，而不是改变输出形状的配置项：

| Dify 错误响应方法 | Ora 组合 |
| --- | --- |
| 出错时终止 | `fail` |
| 忽略错误并继续（失败槽位为 null） | `continue` + 读 `entries` |
| 移除错误输出 | `continue` + 读 `output` |

为什么解耦：捆绑式下拉会随配置改变 `{iter}.output` 的类型（信封数组 vs 值数组），工作流是要发布、复用、给别人跑的——一个藏在节点里的开关能远程改变下游变量的类型，切换一次就要重校验全部下游引用。解耦后三个 selector 类型永远固定，切换策略对下游零影响。Dify 的 null 槽位在 Ora 的类型化数组检查下本身不成立：`array[object]` 要求每个元素是 object，`pool.set` 逐元素校验会拒绝含 null 的数组；而且 null 丢失了失败归因——`entries` 的 `error` 字段来自失败轮 node-run 行的 `error` 列，保留"为什么失败"。

账本在 Rust 侧用真枚举建模，只有暴露到变量池时展平为 JSON 信封：

```rust
enum RoundOutcome {
    Succeeded { item: Value, output: Value },
    Failed { item: Value, error: String },
}
```

`Succeeded` 不可能带 `error`、`Failed` 不可能带 `output`——非法状态不可表示的约束在引擎内部完整保持，退化为信封只发生在变量池的 JSON 边界。`entries[i].output.approved` 这类嵌套路径可像 `structured_output` 的属性一样在编辑器变量目录中展开，现有机制复用。

### D5：按轮决策与账本的持久化编码（serde 向后兼容）

run payload 增加两个内部结构（与 `condition_decisions` 同级的私有状态，不进入用户可选变量目录）：

- **迭代账本** `iteration_ledger`：按迭代节点 id 分组、**以轮次为键**的 `RoundOutcome` 映射。为什么按轮索引而不是序列：串行下轮完成序即轮次序，序列编码本也成立，但位置对齐将依赖 append 顺序的巧合；以轮次为键使 `entries` 与输入数组的对齐成为编码的结构性质，后续并行迭代（完成序 ≠ 轮次序）不需要再迁移编码。轮结算时该轮条目的写入与该轮终态在同一事务提交；已记录的轮结局不可变更，只新增条目；`restart` 重置运行时随之清空。
- **按轮分支决策**：body 内 Condition 的选择按轮记录。现有 `condition_decisions: BTreeMap<String, String>` 以 node_id 为键，单值字典装不下"同一节点每轮一个答案"——第 r+1 轮的 `insert` 会物理抹掉第 r 轮的记录，崩溃恢复时迭代投影会拿后一轮的答案回答前一轮的问题，与 node-run 行的事实自相矛盾。按轮键（迭代节点 + 轮次 + Condition 节点）修复；迭代投影只消费当前轮的决策。

serde 向后兼容：两个新字段 `#[serde(default)]`，旧 payload（含 HITL 停靠数天、随时恢复的历史运行）解析不失败；旧编码的单轮决策保持可读——`resolved_condition_decisions()` 同时读两种编码的先例（`variable_pool.values` 的 `.selected_branch_id` 后缀 → 独立字段迁移）在本仓库已经存在，模式照抄。具体 serde 形态实现期固定，语义已定死：每轮的分支决策与轮结局是持久化事实，重启后精确重放。

### D6：失败传播的作用域吸收

失败路径的仓库端口从"行失败必然 run 失败"演化为携带传播策略：

```rust
enum FailurePropagation {
    Run,       // 现行为：行 Failed + run Failed 同一事务
    Composite, // 行 Failed，run 保持，复合运行时记账并推进
}
```

策略由引擎在 fail 路径上做**图结构判断**解析：失败的 node-run 是否位于某个声明了 `continue` 策略的复合节点的区域内。这是结构判断而非 `run_schedule` 内的节点类型分支——0 号决策 D1 的源码测试（`run_schedule_contains_no_node_type_literals`）继续成立。

迭代节点**自身**的失败永远 `Run` 传播：迭代源非数组、暴露变量写入失败、上限超限。只有**区域内**的失败（body 节点失败、收集目标本轮未运行）可被 `continue` 吸收。这是 0 号决策"节点级错误策略"预留的最小切片：先在复合区域作用域内做，不泛化到任意节点，也不引入失败边拓扑——Dify 并行分支"全部节点标记相同错误、兄弟节点连带失败归因缺失"的已知缺陷提示，失败归因需要显式边界，区域就是本决策的边界。

### D7：编辑器与运行视图的呈现承诺

- **内嵌区域呈现**（React Flow `parentId` + `extent`）：迭代节点渲染为同一画布上的大容器框，body 节点摆在框内，外层连线接到容器边框；配折叠态（折叠成小框 + 摘要）缓解大 body 的画布空间。不采用子流程画布：body 是同一冻结图的结构区域而非独立工作流（同一变量池、同一 worktree、同一快照），子流程画布在视觉上宣称了不存在的东西；该选择不泄漏进 wire format 与引擎，纯前端表现。
- 节点目录放出 `iteration`。Inspector 暴露迭代源选择器（限数组类型变量）、收集目标选择器（限区域内已声明的根变量）、错误策略开关、上限输入。
- 变量目录区域规则：区域内节点可见全局变量、迭代节点上游可见变量、本迭代的 `item` / `index`，以及区域内的上游产物；不可见本迭代的 `output` / `entries` / `failed_count`（尚未产出）。区域外消费者按既有祖先闭包规则可见三个暴露变量，但不可见 `item` / `index`（轮内私有）。
- 编辑器结构校验是提示性的；权威校验在 Rust（发布期 + run 启动期），遵守单一事实源原则。
- 运行视图投影从"一节点一态"（`nodeStates: Record<nodeId, state>`）演化为按 `(node_id, iteration)` 分组；剧场把各轮归到迭代 act 下分组展示；每轮 Agent 会话按节点 + 轮次入口查看。

## 为什么不是这些替代方案

| 方案 | 否决理由 |
| --- | --- |
| Dify 式三模式下拉捆绑 | 输出类型随配置漂移，切换迫使下游全部引用重校验；null 槽位过不了 Ora 的 `array[T]` 逐元素类型检查且丢失失败归因。承认其优势：Dify 用户零学习成本、变量目录少一项——代价是长期类型稳定性 |
| 首个复合运行时做 while-Loop（0 号决策原顺序） | foreach 是更高频需求；轮数先知使上限检查前移到启动边界零浪费；while 的 `exit_when` 求值编排可后置；区域设施完全共享，调换无架构代价 |
| 隐式收集"最后一个节点输出"（Dify 规则） | Ora 区域是任意 DAG——可分支、可多出口，"最后节点"无定义；显式 `collectSelector` 把输出类型变成图解析时静态可判，符合非法状态不可表示 |
| 只靠 node-run 行重建账本 | 结构化输出只存在于变量池且逐轮覆盖写（`complete_node` 写行时只存文本 `output` 与 stop_reason / file_changes），行上没有每轮收集值；账本必须独立持久化 |
| 并行迭代 v1 | 串行覆盖写语义（`{{#fix.output#}}` 解析最新轮）在并发下是写竞态；共享 worktree 的并行轮次文件互踩属执行环境议题。需要迭代级变量隔离（独立副本 + 聚合）与"反馈式 vs 批处理"的静态区分，留给后续 ADR。承认 Dify 并行模式（并发上限 10）的效率优势 |
| 到达上限才失败（0 号决策给 while 设计的行为） | foreach 的 N 在启动时已知，先检查零浪费；跑到上限再失败已消耗 N 个 Agent session |
| 子流程画布呈现 | body 是同一冻结图的结构区域，不是独立工作流；子流程画布实现成本低但隐喻错误，还会让运行视图被迫两级钻入。承认其大 body 画布空间优势——用内嵌 + 折叠态弥补 |

## 本决策未解决的问题

- **while-Loop（`exit_when` 语义）**：区域机制共享，驱动源与终止判定另立 ADR。`specs/test-cases/desktop/core/workflow/region-and-loop.md` 中 Loop 专属用例（`exit_when` 上限失败）随其迁移；区域包含与轮次持久化用例保持共享。
- **并行迭代**：迭代级变量隔离、在途轮的并发放行与乱序归集（账本已按轮索引，编码侧无需迁移；不变量 2 的重算承诺已与串行实现解耦）、共享 worktree 的文件边界。worktree 边界是并行 ADR 的首要立项决策——它决定并行迭代是只读批处理场景的效率优化，还是需要轮级文件隔离的执行环境工程，两者规模相差一个数量级。反馈式循环（引用上一轮产出）只能串行、批处理循环（处理独立列表项）可并行，该区分可在冻结图上静态判定，编辑器可标注。
- **嵌套复合区域**：`iteration` 为整数列，嵌套需升级为路径形式并迁移（0 号决策已推迟，继承）。
- **区域内 interactive 节点**：机制免费（0 号决策 D5 的停靠 + 自动续算），但"数组多长就打断人多少次"需要配套 UX（批量审核队列），且 advance × HITL × 崩溃恢复 × 取消的组合测试面在首版翻倍。v1 结构禁止，二期放开。Dify 同样不支持容器内 Human Input，这是 Ora 的差异化机会而非追赶项。
- **失败轮重试**：v1 失败轮只记账不重跑；`retry` 与 `{iter}.last_error` 属后续。
- **`skipped` 第三状态**：分支绕过收集目标时 v1 记为轮失败；三值判别器牵动下游 Condition 写法，留给后续。
- **收集目标的嵌套路径**：v1 `collectSelector` 限根变量以保证 T 静态可知；嵌套路径收集需要目录跟踪对象字段的声明类型，留给后续。
- **错误策略向非复合节点泛化**（失败边 / 默认值替代 / 首个完成取消其余的竞速）：0 号决策已推迟；`FailurePropagation` 是作用域受限的最小切片。

## 不变量

1. 外层冻结图仍是 DAG；0 号决策 D3 六条区域边界校验全部适用（第六条读作迭代节点必须声明 `1 ≤ maxIterations`）。
2. 轮次是持久化事实：区域内节点每轮执行各占一行并携带轮次；轮次进度恒可从持久化行与账本重算，引擎不为迭代持有任何内存态（v1 串行下当前轮从区域行 `max(iteration)` 推导）。
3. 每轮开始时 `item` / `index` 写入与该轮首批节点行启动同一事务；轮结算时账本条目写入与该轮终态同一事务。
4. `{iter}.output` 元素类型恒等于 `collectSelector` 指向变量的声明类型；`entries` 恒为 `array[object]`；`failed_count` 恒为 `number`；三者不随 `errorStrategy` 改变。
5. `entries` 与输入数组位置对齐；账本以轮次为键，已记录的轮结局在 run 期间不可变更，只新增条目。
6. 轮结算以"本轮节点行"为准做结构性检查；上一轮遗留的池值永不冒充本轮产出。
7. 数组长度超过 `maxIterations` 时节点在启动边界失败，不执行任何轮，绝不静默截断。
8. 迭代节点自身失败（迭代源非数组、暴露写入失败、超限）总是传播为 run 失败；只有区域内失败可被 `continue` 吸收。
9. v1 区域内不含 interactive 节点与嵌套复合节点；轮次串行执行。
10. 不含迭代节点的冻结图执行行为与现状一致；旧版本打开含 `iteration` 的图由 `UnknownNodeType` 解析拒绝。

## 落地顺序

本决策的 Stage B / C 取代 0 号决策落地顺序中的 Stage 3（Loop 复合运行时）；while-Loop 另立 ADR 后按相同模式接入。

- **Stage A（0 号决策 Stage 2，继承）**：区域机制——`BranchProjection` 增加 scope / iteration 参数与六条结构校验。完成条件：现有图行为不变；区域校验独立单测。
- **Stage B：持久化基座**。`iteration` 列 + 索引迁移；run payload 增加账本与按轮决策（`#[serde(default)]`）；boot sweep 对 Running 复合行交还运行时而非一律标失败；`current_nodes` 排除区域行；`compute_run_output` 的 Agent 回退限定 `iteration IS NULL` 的外层行。完成条件：迁移可从旧库升级；无复合节点类型时行为零变化。
- **Stage C：迭代复合运行时**。`NodeType::Iteration` 与 wire 解析；变量声明（`item` / `index` / `output` / `entries` / `failed_count`）；advance 状态机与轮结算结构性检查；`FailurePropagation` 端口演化；注册表一行接入。完成条件：串行 foreach、空数组、fail / continue 两策略、超限启动失败、轮内崩溃恢复各有端到端验证；`run_schedule` 无节点类型字面量（既有源码测试钉死）。
- **Stage D：契约与前端**。`WorkflowNodeRun.iteration` 字段；目录放出节点；内嵌区域编辑与折叠态；inspector；变量目录区域规则；运行视图按轮投影与剧场分组；双语文档。

Stage A / B 不改变任何现有行为；从 Stage C 起含迭代节点的图成为可执行对象。
