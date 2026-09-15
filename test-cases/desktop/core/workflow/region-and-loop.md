# 区域包含与循环节点

循环是节点的结构语义而非图上的回边：复合节点拥有一个区域，在外层投影中是黑盒。本文件守住
DAG 不变量、区域边界与轮次持久化；首个复合运行时为迭代节点（foreach 语义），while-Loop 的
`exit_when` 终止用例随其 ADR 落地后迁移。

## Region Interior Nodes Never Enter The Outer Ready Set

### 风险

区域内部节点泄漏进外层 ready 集合会让外层调度直接 dispatch 区域节点，绕过复合运行时的
迭代编号与变量作用域，产生无归属的 node-run 行与错误的 `current_nodes` 锚点。

### 前置状态

- 冻结图含一个复合节点（首个实例为迭代节点），其区域由结构边可达性推导。
- 复合节点行处于 Running，某轮迭代正在进行。

### 触发

外层投影构造并计算 ready 集合。

### 必须成立

- 外层投影的 ready 集合不含任何区域内部节点。
- 外层只看到 Loop 节点自身作为一个在途节点。
- 区域内部节点的调度只经迭代投影（scope + iteration 参数）发生。

### 禁止结果

- 外层直接 dispatch 区域内部节点。
- 区域内部行出现在外层 `current_nodes` 锚点中。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
|---|---|---|
| Loop Running 期间外层 ready 集合排除全部区域内部节点 | `ora_application::workflow_run::engine::branch_projection::tests::outer_projection_treats_regions_as_black_boxes`（外层投影排除成员与区域行）与 `ora_backend::workflow::run::iteration_tests::region_rows_never_enter_the_outer_current_nodes_anchor`（端到端：区域行在途时 `current_nodes` 仅含迭代节点） | `Verified` |

### 决策依据

[节点运行时编排决策](../../../../decisions/desktop/core/workflow/0-node-runtime-orchestration.md)
不变量 5 与 D3；迭代节点的落地形态见
[迭代节点决策](../../../../decisions/desktop/core/workflow/20260915-iteration-composite-runtime.md)
D1。

## Region Structural Validation Rejects Invalid Enclosures

### 风险

区域边界不封闭（出边流出区域、退出边指回区域、区域内嵌 Output 或另一个复合节点）会分别造成
数据半路流失、真实环、run 终点语义被破坏与嵌套语义未定义。

### 前置状态

一组违反 D3 六条校验规则的候选冻结图。

### 触发

图结构校验（发布期或 run 启动期）执行。

### 必须成立

- 六条规则各自拒绝对应违规：区域非空、无 Output、无嵌套复合节点、出边不出区域、退出边目标
  不在区域内、`1 ≤ maxIterations`（迭代节点语义；0 号决策的 `max_iterations` 承诺由迭代决策
  D3 微调读法）。
- 拒绝发生在执行前，错误信息能定位到违规节点。

### 禁止结果

- 任一违规图被接受并开始执行。
- 校验放宽为警告。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
|---|---|---|
| 六条区域校验规则各自拒绝违规图 | `ora_application::workflow_run::engine::iteration::tests`（`derives_a_valid_region_from_parent_id_containment`、`rejects_a_region_without_entry_edges`、`rejects_a_member_unreachable_from_the_entry_edge`、`rejects_output_nested_composite_and_escaping_edges`、`rejects_outer_edges_targeting_members`、`rejects_parent_id_referencing_a_non_iteration_node`；第六条 maxIterations 由 `parses_iteration_config_with_defaults_and_validation` 覆盖） | `Verified` |

### 决策依据

同上决策 D3 的六条区域边界校验与不变量 7；第六条的迭代节点读法见
[迭代节点决策](../../../../decisions/desktop/core/workflow/20260915-iteration-composite-runtime.md)
D3。

## Outer Graph Remains A DAG

### 风险

循环若被实现为图上的回边，`toposort` 环校验、可达性与闭包查询的既有语义全部失效，prompt
全景排序与输入血缘推导失去依据。

### 前置状态

含环的候选冻结图，以及依赖拓扑序的既有调用方（prompt 全景、闭包查询）。

### 触发

`WorkflowGraph::parse` 执行环校验；拓扑相关查询被调用。

### 必须成立

- 环图仍被 `CycleDetected` 拒绝，含自环。
- 拓扑序、可达性、闭包查询语义与重构前一致。
- 循环表达为复合节点的结构语义，不引入任何回边。

### 禁止结果

- 为支持循环而放宽环校验。
- 拓扑查询在含复合节点的图上返回区域内部与外层混合的顺序。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
|---|---|---|
| 环与自环仍被解析拒绝 | `ora_application::workflow_run::engine::tests::rejects_cycle`、`ora_application::workflow_run::engine::tests::rejects_self_loop_as_cycle` | `Partial`（证明现状语义；含 Loop 图上的不变待 Stage 2 验证） |
| 不可达节点报告语义不变 | `ora_application::workflow_run::engine::tests::unreachable_from_start_reports_isolated_nodes` | `Partial`（同上） |

### 决策依据

同上决策不变量 1 与 D3 第二段。

## Loop Iterations Are Persisted Facts

### 风险

迭代轮次若只存在于运行时内存，进程崩溃后循环丢失进度；审计也无法回放"同一节点第几轮的
执行"。

### 前置状态

- 冻结图含迭代节点，`workflow_node_runs` 已有 `iteration` 列。
- 区域节点在第 k 轮执行。

### 触发

查询该 run 的 node-run 行。

### 必须成立

- 同一节点多轮执行对应多行，各带其 `iteration` 值。
- `{iter}.item` 与 `{iter}.index` 由迭代节点在每轮开始写入变量池，区域模板可解析。
- 区域内节点输出按 selector 覆盖写，`{{#fix.output#}}` 解析为最新完成轮的值。
- 轮次进度可从持久化行与账本重算，无需任何内存状态（v1 串行下当前轮从区域行
  `max(iteration)` 推导）。

### 禁止结果

- 轮次只存在于内存或单一复合节点行的临时字段中。
- 覆盖写语义被并发或恢复路径破坏。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
|---|---|---|
| 多轮执行产生带正确 iteration 的多行，且变量池解析为最新轮 | `ora_backend::workflow::run::iteration_tests::serial_foreach_executes_every_round_and_exposes_the_ledger`（三轮三行各带 iteration；`fix.output`、`iter.item`、`iter.index` 均为最新轮值） | `Verified` |

### 决策依据

同上决策 D4 与[迭代节点决策](../../../../decisions/desktop/core/workflow/20260915-iteration-composite-runtime.md)
D2。

## Loop Fails At max_iterations Without Satisfying Exit Conditions

### 风险

无硬上限或上限不生效的循环会无限消耗 Agent session 与 token，桌面场景下无法安全中断。

### 前置状态

- Loop 声明 `max_iterations = N`。
- 前 N 轮结束后 `exit_when` 求值始终为假。

### 触发

第 N 轮区域耗尽后的 Loop 决策。

### 必须成立

- Loop 节点进入失败终态，run 随之终态。
- 失败信息可区分"到达迭代上限"与其他失败原因。

### 禁止结果

- 以任何形式发起第 N+1 轮。
- 上限校验被 `exit_when` 的求值错误绕过（求值失败同样失败节点）。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
|---|---|---|
| 到达上限且退出条件不满足时进入失败终态，不发起下一轮 | 尚无直接证据 | `Missing` |

### 决策依据

同上决策不变量 6 与 D4 终止条件。while-Loop（`exit_when`）已由
[迭代节点决策](../../../../decisions/desktop/core/workflow/20260915-iteration-composite-runtime.md)
D4 移至后续 while-Loop ADR（首个复合运行时改为 foreach 迭代，数组长度即轮数）；本用例随
该 ADR 迁移，迁移前保留以记录 while 语义承诺。
