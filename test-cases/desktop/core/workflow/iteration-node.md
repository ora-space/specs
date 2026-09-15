# 迭代节点复合运行时

迭代节点（foreach 语义）以数组为迭代源、区域内的冻结子图对每个元素执行一轮，错误策略解耦为
二值控制流开关加按轮账本。本文件守住启动边界、轮次事务性、错误策略的作用域与账本投影。

## Empty Iterator Array Completes With Empty Outputs

### 风险

空数组若被当作错误或让节点滞留 Running，"对零个元素做批量处理"这一合法场景（如过滤后结果为空）
就无法表达；用户会被迫在迭代节点外再包一层 Condition 判空。

### 前置状态

- 冻结图含一个迭代节点，`iteratorSelector` 指向已声明为数组类型的变量。
- 运行时该变量解析为空数组 `[]`，`errorStrategy` 为任一取值。

### 触发

迭代节点启动，解析迭代源。

### 必须成立

- 节点立即成功，不执行任何轮，不产生任何区域行。
- `{iter}.output` 与 `{iter}.entries` 为空数组，`{iter}.failed_count` 为 0。

### 禁止结果

- 空数组使节点失败或 run 失败。
- 产生任何带 `iteration` 值的 node-run 行。
- 下游引用三个暴露变量时报未声明。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
|---|---|---|
| 空数组输入下节点立即成功且三个暴露变量为空值 | `ora_backend::workflow::run::iteration_tests::an_empty_iterator_source_completes_with_empty_outputs`（真实 SQLite 端到端：无任何区域行、run 成功、三个暴露变量为空值） | `Verified` |

### 决策依据

[迭代节点决策](../../../../decisions/desktop/core/workflow/20260915-iteration-composite-runtime.md)
D1 空数组语义。

## Iteration Fails At The Startup Boundary When Input Exceeds maxIterations

### 风险

超限若被静默截断（只跑前 `maxIterations` 项），用户以为 500 项全部处理、实际只有 50 项而运行
显示成功——正确性灾难；若跑到上限才失败，已消耗 N 个 Agent session 才发现问题。

### 前置状态

- 迭代节点声明 `maxIterations = M`。
- 运行时迭代源解析为长度 N 的数组，`N > M`。

### 触发

节点启动边界检查。

### 必须成立

- 节点在启动边界失败，不执行任何轮。
- 失败传播为 run 失败（迭代节点自身失败，不受 `continue` 策略保护）。
- 错误信息包含数组长度 N、上限值 M 与调整指引。

### 禁止结果

- 静默截断为前 M 项而运行显示成功。
- 执行若干轮后才失败。
- `continue` 策略下超限被吸收为账本条目。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
|---|---|---|
| `N > maxIterations` 时启动边界失败、零轮执行、错误信息含 N 与 M | `ora_backend::workflow::run::iteration_tests::a_source_exceeding_the_ceiling_fails_at_the_startup_boundary`（`continue` 策略下自身失败仍传播为 run 失败，同时覆盖下方 continue 吸收边界义务的第二行） | `Verified` |

### 决策依据

同上决策 D2 安全上限与不变量 7、8。

## Round Start And Settlement Commit Atomically With Round Rows

### 风险

`item` / `index` 写入与轮内节点行启动若不在同一事务，崩溃后会出现"轮变量已写、轮内行未启动"或
反之的中间态——恢复时要么重跑已开始轮次、要么拿错轮次的变量值驱动节点。账本条目与轮终态分离
提交同理：账本缺失的失败轮会在恢复后被当作未发生。

### 前置状态

- 迭代节点 Running，准备进入第 r 轮 / 结算第 r 轮。
- 进程在事务提交前后任意时刻被杀。

### 触发

轮开始与轮结算的事务提交；随后 boot sweep 与 advance 续算。

### 必须成立

- 每轮开始时 `{iter}.item` / `{iter}.index` 写入与该轮首批节点行的启动在同一 SQLite 事务提交：
  崩溃后要么整轮未开始，要么轮变量与轮内行一致。
- 轮结算时账本条目写入与该轮终态在同一事务提交。
- 崩溃重放后不存在"轮变量存在但轮内行缺失"或"轮已终态但账本缺条目"的状态。

### 禁止结果

- 轮变量与轮内行分属不同事务导致崩溃后可观测的中间态。
- 账本条目滞后于轮终态单独提交。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
|---|---|---|
| 轮开始与轮结算的原子性（含崩溃注入后重放验证无中间态） | `ora_backend::workflow::run::recovery::tests::a_crashed_round_inside_a_running_iteration_is_absorbed_not_failed`（注入进程崩溃后走真实 boot sweep + reconcile 重放：被打断轮以失败结算、已完成轮不重跑、账本与轮内行一致；轮开始/结算的单事务边界由 `workflow_run_engine/iteration.rs` 的原子操作保证） | `Verified` |

### 决策依据

同上决策 D2 与不变量 3。

## Round Settlement Requires The Collect Target To Have Succeeded This Round

### 风险

区域内 Condition 分支可能使某轮绕开收集目标。此时变量池里残留的是第 r-1 轮的值，`pool.resolve`
不会报错而是返回陈旧值——如果不做结构性检查，上一轮的产出会冒充本轮产出进入账本。

### 前置状态

- 迭代节点 Running，第 r 轮区域内包含 Condition，其活跃分支绕开 `collectSelector` 指向的节点。
- 第 r-1 轮收集目标成功且池中残留其值。

### 触发

第 r 轮区域耗尽，轮结算执行结构性检查。

### 必须成立

- 结算以"本轮节点行"为准：收集目标在本轮不存在 `Succeeded` 行时，该轮记为失败。
- 失败原因可区分（"collect target did not run this round"）。
- `fail` 策略下该轮失败传播为 run 失败；`continue` 策略下记入账本。

### 禁止结果

- 第 r-1 轮的池值冒充第 r 轮产出进入账本或 `{iter}.output`。
- 收集目标未运行被静默跳过而运行显示成功。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
|---|---|---|
| 分支绕开收集目标时该轮按失败结算，陈旧池值不进入账本 | `ora_backend::workflow::run::iteration_tests::per_round_condition_decisions_settle_bypassed_rounds_as_failed`（端到端：绕开轮按 `collect target did not run this round` 失败）与 `ora_application::workflow_run::engine::node_runtime::iteration::tests::settlement_requires_the_collect_target_to_have_run_this_round`（纯函数：池中残留上一轮值也不冒充） | `Verified` |

### 决策依据

同上决策 D3 结构性检查与不变量 6。

## fail Strategy Propagates The First Failed Round To Run Failure

### 风险

`fail` 是默认策略，也是与既有引擎行为一致的路径。若区域内失败被静默吸收或延迟传播，默认配置
下的迭代节点就改变了"节点失败即 run 终态"的既有语义。

### 前置状态

- 迭代节点 `errorStrategy = fail`，迭代源含 N ≥ 2 个元素。
- 第 k 轮（`k < N`）区域内某节点失败。

### 触发

第 k 轮失败行的 fail 路径执行。

### 必须成立

- 失败以 `FailurePropagation::Composite` 语义解析到迭代节点后，因策略为 `fail`，迭代节点进入
  失败终态，run 随之失败。
- 后续轮（k+1 … N-1）不执行。
- 已完成轮的 node-run 行与账本条目保持已提交状态（审计事实不回滚）。

### 禁止结果

- 第 k 轮之后的任何轮开始执行。
- 失败被吸收后节点显示成功。
- 回滚已完成轮的持久化行。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
|---|---|---|
| `fail` 策略下首个失败轮终止节点与 run，后续轮不执行 | `ora_backend::workflow::run::iteration_tests::fail_strategy_stops_at_the_first_failed_round`（轮 1 经回调失败：迭代行与 run 失败、轮 2 零行、轮 0 提交历史保留） | `Verified` |

### 决策依据

同上决策 D4、D6 与不变量 8。

## continue Strategy Absorbs Failed Rounds Into The Ledger

### 风险

`continue` 的吸收边界若划错——吸收了迭代节点自身的失败，或账本投影错位——用户会得到"显示成功
但静默丢数据"或"失败归因错轮"的运行结果。

### 前置状态

- 迭代节点 `errorStrategy = continue`，迭代源含 N 个元素。
- 第 i、j 两轮区域内失败（`i < j`），其余轮成功。

### 触发

各轮结算与节点完成。

### 必须成立

- 失败轮记入账本 `{item, status, output, error}` 信封，`error` 来自该轮失败 node-run 行的
  `error` 列；剩余轮继续执行；节点最终完成，run 不因区域失败进入终态失败。
- `entries` 与输入数组位置对齐（N 个条目，位置即输入位置）；`failed_count` = 2；
  `output` 仅含成功轮的收集值，按输入顺序。
- 吸收只作用于区域内失败：迭代节点自身失败（迭代源非数组、暴露写入失败、超限）在 `continue`
  策略下仍传播为 run 失败。

### 禁止结果

- 迭代节点自身失败被 `continue` 吸收。
- `entries` 条目与输入位置错位（例如按完成序而非轮次排列）。
- 失败轮的 `error` 字段丢失或串轮。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
|---|---|---|
| `continue` 策略下失败轮入账本、节点完成、三个暴露变量投影正确 | `ora_backend::workflow::run::iteration_tests::continue_strategy_absorbs_failed_rounds_into_the_ledger`（失败轮 error 归因入账、entries 与输入对齐、output 仅含成功轮、failed_count=1、run 成功） | `Verified` |
| 迭代节点自身失败在 `continue` 下仍传播为 run 失败 | `ora_backend::workflow::run::iteration_tests::a_source_exceeding_the_ceiling_fails_at_the_startup_boundary`（`continue` 配置 + 超限 → run Failed） | `Verified` |

### 决策依据

同上决策 D4、D6 与不变量 5、8。

## Exposed Iteration Variables Keep Fixed Types Across Error Strategies

### 风险

暴露变量类型若随 `errorStrategy` 漂移（Dify 三模式的缺陷），切换一次策略就要重校验全部下游
引用；已发布快照上的远程切换会直接破坏下游 Condition 与模板。

### 前置状态

- 同一冻结图的两次运行：一次 `errorStrategy = fail`，一次 `continue`，均含失败轮（fail 运行在
  失败前已有成功轮）。
- 图解析期的变量目录与完成时的变量池。

### 触发

图解析声明变量目录；运行完成后读取三个暴露变量。

### 必须成立

- 图解析时迭代节点按自身配置声明 `.output: array[T]`（T 为收集目标声明类型）、
  `entries: array[object]`、`failed_count: number`，而非沿用"产数据节点一律 `.output: string`"
  的默认规则。
- 两种策略下三个 selector 的声明类型与实际值类型完全一致。
- 下游引用（含 `entries[i].output.x` 嵌套路径）在两种策略下都通过既有类型检查。

### 禁止结果

- 切换 `errorStrategy` 使任一暴露变量的声明类型或值类型改变。
- 暴露变量经由文本 `output` 的池写入路径写入（该路径只写 string）。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
|---|---|---|
| 变量目录声明 `array[T]` / `array[object]` / `number` 且跨策略不变 | `ora_backend::workflow::run::iteration_tests::exposed_variable_types_stay_fixed_across_error_strategies`（两策略下五个声明逐一断言）与前端 `packages/workflow-mock/src/variable-catalog.test.ts`（编辑器目录同一规则） | `Verified` |
| 账本派生写入通过 `pool.set` 的逐元素类型检查 | `ora_backend::workflow::run::iteration_tests::serial_foreach_executes_every_round_and_exposes_the_ledger`（端到端断言三个暴露变量经类型化池写入后的值，含 `array[string]` 逐元素检查） | `Verified` |

### 决策依据

同上决策 D3 与不变量 4；写入边界另见
[变量池写入边界](variable-pool-boundary.md)。

## Per-Round Condition Decisions Are Scoped To Their Round

### 风险

既有 `condition_decisions` 以 node_id 为键的单值字典装不下"同一 Condition 节点每轮一个答案"：
第 r+1 轮的写入会物理抹掉第 r 轮的记录，崩溃恢复时迭代投影会拿后一轮的答案回答前一轮的问题，
与 node-run 行的事实自相矛盾。

### 前置状态

- 迭代节点 Running，区域内含 Condition 节点。
- 第 r 轮该 Condition 选择了分支 A，第 r+1 轮选择了分支 B。

### 触发

两轮的 Condition 求值与按轮决策持久化；随后任意一轮的投影重算（含崩溃后恢复）。

### 必须成立

- 分支决策按（迭代节点 + 轮次 + Condition 节点）记录，第 r+1 轮的写入不覆盖第 r 轮的记录。
- 迭代投影（scope = 区域，iteration = r）只消费第 r 轮的决策。
- 崩溃重放后各轮的分支激活与该轮 node-run 行事实一致。
- 旧 payload（无按轮编码）解析不失败，单轮旧编码保持可读。

### 禁止结果

- 同一 Condition 节点跨轮共享单一决策值。
- 后一轮决策在恢复时回答前一轮的分支问题。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
|---|---|---|
| 跨轮不同分支选择的图上，两轮投影各自正确且互不污染 | `ora_backend::workflow::run::iteration_tests::per_round_condition_decisions_settle_bypassed_rounds_as_failed`（同一 Condition 三轮不同分支，按轮决策持久化、各自结算）与 `ora_application::workflow_run::engine::node_runtime::iteration::tests::the_region_round_projection_is_isolated_per_round`（轮 1 行不泄漏进轮 0 投影） | `Verified` |
| 旧 payload（无按轮字段）解析不失败 | `ora_backend::workflow::run::iteration_tests::legacy_payloads_without_iteration_fields_parse`（旧编码单轮决策保持可读） | `Verified` |

### 决策依据

同上决策 D5。

## An Interrupted Round Settles As Failed On Recovery

### 风险

崩溃打断在途轮后，若恢复路径把被打断轮当作"未发生"重跑，区域内已产生副作用的 Agent 轮次会
二次执行；若滞留 Running 无 in-flight 行，run 永久卡死。

### 前置状态

- 迭代节点 Running，第 r 轮区域行部分完成，账本无第 r 轮条目。
- 进程被杀；boot sweep 已把非复合的 Running 行标记 `interrupted_by_restart` 失败。

### 触发

boot sweep 把 Running 的迭代节点行交还运行时，advance 续算。

### 必须成立

- 被打断的第 r 轮按"该轮失败"结算：账本新增第 r 轮的 `Failed` 条目，不重跑该轮任何节点。
- `fail` 策略下节点失败、run 失败；`continue` 策略下继续第 r+1 轮。
- 已 Succeeded 的历史轮（0 … r-1）不重跑，其行与账本条目原样保留。
- v1 串行下当前轮从区域行 `max(iteration)` 推导，不依赖崩溃前内存态。

### 禁止结果

- 被打断轮的已完成节点重跑。
- run 滞留 Running 而无 in-flight 节点。
- 恢复依赖崩溃前的内存状态或日志文件。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
|---|---|---|
| 轮中途崩溃后按失败轮结算并续算，历史轮不重跑 | `ora_backend::workflow::run::recovery::tests::a_crashed_round_inside_a_running_iteration_is_absorbed_not_failed`（真实 boot sweep + reconcile：已完成轮 0 仅一行不重跑，被打断轮 1 以 `interrupted_by_restart` 结算入账本，节点完成） | `Verified` |

### 决策依据

同上决策 D2 与不变量 2；通用复合恢复承诺见
[无状态恢复与循环内人工介入](stateless-recovery-and-hitl.md)。

## v1 Iteration Regions Exclude Interactive Nodes

### 风险

区域内 interactive 节点会使"数组多长就打断人多少次"，且 advance × HITL × 崩溃恢复 × 取消的
组合测试面在首版翻倍；若无结构校验拦截，这类图会在运行期以更昂贵的失败方式暴露问题。

### 前置状态

- 候选冻结图：迭代节点区域内含声明 `interactive: true` 的 Agent 节点。

### 触发

图结构校验（发布期与 run 启动期）执行。

### 必须成立

- 结构校验拒绝该图，错误信息定位到违规节点。
- 拒绝发生在执行前；编辑器侧的同类校验是提示，Rust 侧是权威。

### 禁止结果

- 含区域内 interactive 节点的图通过校验并开始执行。
- 校验放宽为警告。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
|---|---|---|
| 区域内 interactive 节点被结构校验拒绝 | `ora_application::workflow_run::engine::iteration::tests::rejects_interactive_nodes_inside_the_region`（图解析期 `InvalidRegion` 拒绝并定位到违规节点） | `Verified` |

### 决策依据

同上决策 D5 修改（v1 结构校验禁止区域内 interactive 节点）与不变量 9；二期放开后的停靠与
续算承诺见[无状态恢复与循环内人工介入](stateless-recovery-and-hitl.md)。

## Older Versions Reject Graphs Containing Iteration Nodes

### 风险

旧版本 Ora 若以半解析状态打开含 `iteration` 的图，会在后续编辑或运行时产生更深的损坏；兼容性
边界要求拒绝发生在图解析期。

### 前置状态

- 含 `iteration` 节点的冻结图 JSON。

### 触发

旧版本（不识别 `iteration` kind）执行 `WorkflowGraph::parse`。

### 必须成立

- 解析以 `UnknownNodeType` 拒绝，错误定位到节点 id 与实际 kind 值。
- 不产生半解析状态（无部分构建的节点集合或区域）。

### 禁止结果

- 未知 kind 被跳过或降级为其他节点类型。
- 解析失败后残留可变的全局图状态。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
|---|---|---|
| 未知节点 kind 由解析拒绝且错误可定位 | `ora_application::workflow_run::engine::tests::rejects_unknown_node_type`（既有路径在 `bogus` kind 上拒绝；`iteration` kind 注册后的同类拒绝待 Stage C 补证据） | `Partial` |

### 决策依据

同上决策兼容性边界与不变量 10；既有图行为不变的通用承诺见
[引擎重构行为等价](engine-parity.md)。
