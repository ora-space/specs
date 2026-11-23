---
status: implemented
---

# Iteration 在 Cloud 执行：区域成员、foreach 轮次与按轮账本

本文是[工作流执行根决策](0-cloud-side-scheduler-owns-workflow-execution.md)（`implemented`）的后续决策：把 desktop
可执行的 `iteration`（foreach）节点迁到 Cloud，将根决策 D4 延后的该 kind 纳入可发布、可执行的集合。

结论：

- **成员模型统一**：decode 时取 `effectiveContainer(node) = data.containerId ?? parentId`。iteration
  的存量草稿用 `parentId`、loop 用 `containerId`；归一后门禁与调度器只认一个键，存量草稿继续可读。
- **wire** 沿用 Cloud 编辑器已有的 `data.iterationConfig`：`iteratorSelector`、`collectSelector`、
  `errorStrategy`（`fail` 默认 / `continue`）、`maxIterations`（默认 50，`1..100`）。
- **轮次**：数组长度即轮数，第 r 轮把 `items[r]` 绑定到 `{iterationId}.item`、r 绑定到
  `{iterationId}.index`，然后驱动区域内节点一轮；全部轮结算后节点完成。
- **暴露变量**：`output`、`entries`、`failed_count`，三者类型固定，不随 `errorStrategy` 改变。
- **上限**：`N > maxIterations` 在**启动边界失败**，一轮都不执行，绝不静默截断；空数组立即成功。
- **失败**：区域内失败可被 `continue` 吸收进账本；iteration 自身的失败（迭代源非数组、超限、暴露写入
  失败）总是传播为 run 失败。

**状态与影响面**：`implemented`。串行 foreach、区域门禁、账本、容量保护和逐轮视图已在 Cloud 落地，
真实栈已验证快节点、Agent 产出收集、取消及手动 Run / Issue @ Confirm 两个入口。
`implemented` 不表示全部边界证据为 `Covered`；完整证据矩阵及明确承接的缺口见核心用例。
核心验证义务见[迭代执行用例](../../../test-cases/cloud/workflow/iteration-foreach-execution.md)。涉及 `internal/core` 的门禁、调度器、运行状态与
前端检查器；**不改数据库 schema、不改 OpenAPI**（沿用 `workflow_runs.node_states` 单行 CAS）。
**非目标**：并行迭代、`maxIterations` 的界面软警告、子流程画布、变量选择器化。

## 继承与修改

| 前序约定 | 本文处理 |
|---|---|
| 根决策 D2：发布门禁拒绝首期子集之外的 kind | **修改**：`iteration` 进入可执行集合，门禁改为按配置与区域校验 |
| 根决策 D4：iteration、loop 等复合运行时延后 | **落实**：iteration 的首期可执行契约；loop 由后测循环决策独立定义 |
| Cloud 根决策 D1 / D4、不变量 3：调度器只从已提交状态重算，节点结果随版本 CAS 提交 | **继承**：每轮 body 节点状态、轮终态与账本在同一次提交里写入 |
| [desktop 根决策](../../desktop/core/workflow/0-node-runtime-orchestration.md) D3：六条区域边界校验 | **继承边界、收紧预算**：Cloud 复用 loop 的有效容器校验；`maxIterations` 必须在 `1..100` 内（D2、D6） |
| desktop `iteration.rs` 与 [iteration 复合运行时决策](../../desktop/core/workflow/20260915-iteration-composite-runtime.md)：数组长度即轮数、item/index 绑定、三暴露变量、fail/continue、超限启动失败、空数组成功 | **继承语义，修改容量边界**：Cloud 增加 100 项与字节硬上限，以约束单行 CAS 的写入体积（D5、D6） |
| desktop 根决策 D6：变量根声明与下游可见性 | **继承**：iteration 声明三个暴露变量；`item` / `index` 只对区域内可见 |
| [loop 与 loopExit 后续决策](20261025-loop-and-loop-exit-execution.md)：有效容器统一解析、区域前驱只在本 body 计算、`inactive` 复用既有状态、8 MiB 总预算 | **继承**：成员归一、区域前驱、未选中成员状态与字节预算采用同一规则（D1、D3、D5） |
| [变量聚合后续决策](20261025-variable-aggregator-execution.md)：候选必须是严格传递前驱，容器边界在发布时校验 | **继承**：迭代源与 body 内 aggregator 采用同一前驱与容器边界（D2、D3） |

## 决策

### D1：成员归属的归一

decode 工作流图时，把每个节点的有效容器定为：

```
effectiveContainer(node) = data.containerId  (存在且非空)
                          ?? parentId         (React Flow 的父节点)
```

- **为什么**：Cloud 前端两条路径已经固定——iteration 成员持久化 `parentId`（`use-workflow-draft.ts`
  序列化保留），loop 成员用 `data.containerId`（调色板工厂创建）。Go 侧若各写一套成员集合，门禁、
  调度器和变量可见性会出现三份不一致的判断。
- **兼容边界**：只读归一，不回写草稿、不改 wire；`parentId` 仍是 iteration 的持久化形式。
- **优先级**：非空 `containerId` 优先；缺失或空值时才用 `parentId`。两者冲突时只使用显式
  `containerId`，门禁、调度器与变量可见性复用同一解析入口；[loop ADR](20261025-loop-and-loop-exit-execution.md) 采用相同规则。
- **区域规则**（沿用 loop 的校验器）：成员不能是 `loop` / `iteration` / `subflow`；`output` 不能出现在
  区域内；成员不能跨区域；区域必须非空。
- **编辑器入口边**：允许 iteration 本身以 `sourceHandle: iteration-entry` 指向自身有效成员；该边可选，
  原样保留在草稿与发布快照中。它表达容器入口，不进入区域调度依赖或成员的传递前驱，成员不会等待容器
  先完成。外部节点、其他容器和区域成员不能借此 handle 绕过跨区域限制。

### D2：配置与门禁

`iterationConfig` 是必填对象，逐项校验（`code` / `nodeId` / `field`）：

| 拒绝原因 | 条件 |
|---|---|
| `iteration_config_missing` | 没有配置或不是对象 |
| `iteration_iterator_unsupported` | `iteratorSelector` 不是「节点 + 根」两段，或带嵌套路径 |
| `iteration_iterator_undeclared` | 迭代源根未声明，或其节点不是 iteration 在外层图上的传递前驱；已声明的 globals / `sys.*` 豁免前驱检查 |
| `iteration_collect_unsupported` | `collectSelector` 不是两段根选择器 |
| `iteration_collect_outside_region` | 收集目标不是区域成员声明的根 |
| `iteration_error_strategy_unknown` | `errorStrategy` 不是 `fail` / `continue` |
| `iteration_max_iterations_invalid` | `maxIterations` 不是 `1..100` 的整数 |
| `iteration_body_empty` | 区域没有成员 |
| `iteration_body_unreachable` | 区域内没有入口成员（区域内无前驱的成员），或存在从入口不可达的成员 |

`maxIterations` 缺省 50，硬上限 100。desktop 只要求 ≥ 1；Cloud 为控制运行行聚合的体积增加该约束，
导入超过上限的图时保留可编辑草稿，但发布明确拒绝，不静默改成 100。
100 项与 Loop 的轮数边界保持同一数量级，并保留默认 50 项的余量；单项和总字节预算共同约束大项和
重复归档造成的整行写入放大。这些是 Cloud 的容量承诺，不继承 desktop 的无上限配置。
保存草稿不校验；发布是冻结执行文档的时刻（根决策 D2）。

迭代源的节点前驱在 iteration 所属的**外层图**内沿入边传递计算，不能指向自身、区域成员、其他容器的
成员或外层非前驱节点；`sys.*` 与 globals 只豁免前驱检查，仍须声明合法的根。即使非前驱节点偶然先提交了
值也不能成为合法来源，否则同一张图的迭代数组会随调度顺序改变。发布和运行使用同一校验规则。

迭代源来自 Start JSON 控件时，共享运行准入按选定发布快照将声明为 `array`、typed array 或 `object`
的 JSON 文本解析一次并验证结构；原生 API 结构值同样接受。非法 JSON / 类型不符在创建运行前返回
`invalid_field_value` 与字段定位。普通文本、`any` 和保留的 kickoff `input` 不转换；该准入不替代
iteration 启动时对其他来源执行的数组与项数检查。

### D3：轮次与作用域

- 迭代源解析为数组后得到 `N`；`N = 0` 是合法输入，节点立即成功，`output`/`entries` 为空数组、
  `failed_count = 0`。
- 第 r 轮（`0 ≤ r < N`）把 `{iter}.item = items[r]`、`{iter}.index = r` 注入**本轮**作用域，然后驱动
  区域 body 从入口成员开始的一轮执行；一轮内 body 的节点状态按既有 `node_states` 提交。
- 轮次串行：第 r 轮全部结算（成功或失败）后才开始第 r+1 轮。
- `item` / `index` 只对区域内可见；区域外不可见（轮内私有）。区域外可见三个暴露变量。
- body 入口成员：区域内没有来自其他成员入边的成员。没有入口或存在入口不可达成员时发布被拒（D2）。
- **区域前驱**：body 内所有节点（包括 aggregator）的传递前驱只在该 iteration 的区域图内计算。
  区域外的节点值只能从 iteration 的可见外层祖先导入，另加已声明的 globals / `sys.*` 与本轮
  `{iter}.item` / `{iter}.index`；它们是显式作用域来源，不伪造成 body 内的前驱。区域外非祖先、其他
  容器的成员与其他轮次均不可直接读取。body 内 aggregator 对本区域候选执行同一前驱检查，对上述显式
  导入的作用域来源按可见性校验；与 aggregator 的传递前驱规则及 loop
  的区域前驱规则一致。当前容器的三个最终暴露变量不在本轮变量池内。
- **未选中成员**：condition 未选中分支上的成员在本轮结算前持久化为 `status: "inactive"`，随后随本轮
  归档；不写成功结果，不产生输出或执行会话。`inactive` 是既有节点状态，运行详情显示“未到达”，
  不是新增的 WorkflowRun 状态，也不需要 API 枚举或旧前端迁移。已开始的成员不能被改写成 `inactive`。

### D4：暴露变量与账本

| 变量 | 类型 | 含义 |
|---|---|---|
| `{iter}.output` | `array` | 只含**成功轮**的收集值，保持输入顺序 |
| `{iter}.entries` | `array[object]` | 已结算轮的完整账本：每轮 `{item, status, output, error}`，与输入数组的已处理前缀对齐 |
| `{iter}.failed_count` | `number` | 失败轮数 |

- `output` / `entries` / `failed_count` 的类型不随 `errorStrategy` 改变：策略只决定「失败轮之后是否继续」，
  不改变下游变量的类型。
- 每轮的收集值来自 `collectSelector` 指向的根，且目标必须有本轮 `succeeded` 的成员状态；该根本轮未运行时，该轮记为失败，错误为
  `collect target did not run this round`。
- 账本以零起始轮号为键、连续且不可变更；`entries` 按轮次投影，`output` 过滤失败轮。全部 N 轮结束时
  `entries` 长度为 N；`fail` 提前结束时只记到失败轮，后续未执行项不伪造条目。
- 成功条目是 `{item, status:"succeeded", output:收集值, error:null}`；失败条目是
  `{item, status:"failed", output:null, error:失败说明}`。成功收集到的 `null` 与失败依靠 `status` 区分。
  两种策略都先把失败轮写入账本并归档，再决定失败还是推进下一轮；容器自身失败不伪装成 body 失败条目。

### D5：持久化与取消

复用 `workflow_runs.node_states` 单行 CAS：活动轮成员以各自节点 id 留在扁平表，iteration 节点的
`iteration` 字段保存可恢复的轮次文档。下面展示第 0 轮已经结算、第 1 轮正在执行的持久化 JSON：

```json
{
  "iter-1": {
    "status": "running",
    "displayName": "逐项处理",
    "startedAt": "2026-10-25T02:00:00Z",
    "iteration": {
      "round": 1,
      "items": ["甲", "乙"],
      "strategy": "continue",
      "ledger": [
        {"item": "甲", "status": "succeeded", "output": "结果甲", "error": null}
      ],
      "output": ["结果甲"],
      "failedCount": 0,
      "completed": [
        {
          "index": 0,
          "states": {
            "choose": {"status": "succeeded", "output": {"selectedBranch": "yes"}},
            "writer": {"status": "succeeded", "output": {"output": "结果甲"}},
            "other": {"status": "inactive"}
          }
        }
      ]
    }
  },
  "choose": {"status": "succeeded", "output": {"selectedBranch": "yes"}},
  "writer": {"status": "running", "startedAt": "2026-10-25T02:01:00Z"}
}
```

| 字段 | 持久化含义与约束 |
|---|---|
| `iteration.round` | 零起始的当前轮号；终态保留最后启动的轮号，不递增为 N。空数组使用 0，但没有活动轮或归档 |
| `iteration.items` | 进入时一次解析并冻结的输入数组；总项数是其长度，恢复不重新读取来源 |
| `iteration.strategy` | 冻结的 `fail` / `continue`，缺省值在进入时写成 `fail` |
| `iteration.ledger` | D4 信封数组，第 r 项就是第 r 轮；仅追加已结算轮，绝不记录尚未执行的后续项 |
| `iteration.output` / `failedCount` | 账本中成功值的有序过滤 / 失败条目数；与账本在同一次 CAS 中提交 |
| `iteration.completed[]` | 每轮一个 `{index, states}`，`index` 零起始且唯一；`states` 是该轮成员完整状态快照，包含 condition 选择、会话 / 尝试信息及 `inactive`，沿用既有节点状态形状 |
| 当前 `item` / `index` | 不另存第二份值；由 `items[round]` 与 `round` 推导，只注入当前区域变量池，不写成 iteration 节点的最终 `output` |
| 顶层成员状态 | 只属于 `iteration.round`；开始下一轮时旧成员从扁平表清空，旧结果只能从归档读取 |

- 一轮结算时，账本条目、成功值 / 失败计数、`completed` 归档与轮次推进在同一次版本 CAS 中提交。
  有下一轮且策略允许继续时，清空顶层成员并将 `round` 加一；成功结束或 `fail` 停止时，保留最后一轮
  成员及其轮号，同时保留归档。空数组的 `items` / `ledger` / `output` / `completed` 都是 `[]`，
  `failedCount` 为 0，不创建成员状态。
- **最后一轮去重**：运行详情以 `(iteration 节点 id, 轮号)` 标识一轮。存在 `completed[index = round]` 时
  只展示该归档一次，顶层成员是同一轮的兼容投影；不存在该归档时，顶层成员就是未结算的活动轮。
  取消或体积失败发生在轮中时保留活动轮，不凭空生成归档或账本；两个来源不能被计作两轮。
- 成功终态的节点 `output` 为
  `{"output": iteration.output, "entries": iteration.ledger, "failed_count": iteration.failedCount}`；
  失败时沿既有 `error` / `output.reason` 写入有界原因，历史账本仍从 `iteration` 读取。
- **不新增表**、不改 OpenAPI；`nodeStates` 仍是自由对象，新增的 `iteration` 字段由详情读取方解析，旧前端
  继续读取既有节点字段并忽略未知字段。节点 `status` 沿用既有集合（包括成员的 `inactive` / `retrying`），
  不新增 WorkflowRun 状态。
- 取消：run 取消时停止当前轮、把 iteration 标记为 cancelled，不再启动后续轮；已提交的轮保持原样。
- 恢复：调度器只依据已提交状态重算，崩溃后从未结算的轮继续；已结算的轮不重跑。活动轮内已提交结果不重派，
  在途 agent 按根决策持有到自身结果，不能因换副本另建会话。恢复校验轮号、账本、归档与派生值一致性；
  不合法的轮次文档稳定失败为 `iteration_invalid_round`，不能把它当作“未开始”清空后从第 0 轮重跑。
- **体积预算**：MiB 为 `1024 * 1024` 字节，按实际写入的紧凑 UTF-8 JSON 编码计算。
  每项输入、每项 collect 值、每轮归档（该轮所有成员状态合计）分别不得超过 1 MiB；
  完整 `node_states` 不得超过 8 MiB，其中预留 64 KiB 给失败记录；普通提交最多使用 `8 MiB - 64 KiB`。
  完整体积包括迭代源、
  `iteration.completed`、当前成员以及 `output` / `entries` 中的所有重复值，不用逻辑去重后的大小代替。
- 检查覆盖启动边界的数组/项、每次成员提交、归档和最终暴露变量提交。超限保留上一次已提交的状态与账本，
  不追加超大数据，利用预留空间记录容器、轮次、字节数、上限与稳定失败原因，沿既有路径有界收尾；
  不截断值、不删掉既有失败轮后宣告成功。该预算与 loop 相同，只约束新引入的复合运行，历史运行不回写。
  8 MiB 初值约束模型 X 每次整行 CAS 的重写与 WAL 放大；以后放宽预算兼容，收紧却可能让已发布图失败，
  因此先采用保守上限，后续调整以接近上限的测量为依据。
- **运行视图**：iteration 节点显示当前轮 / 总项数、成功和失败项数及终态原因；展开后先按 `entries`
  展示每项的 item、status、output、error，再从 `iteration.completed` 按轮展示成员名称、状态、输出、
  错误。普通节点会话读取已交付，所选历史轮与会话尝试的绑定仍由节点执行证据读取能力承接；
  不能用当前尝试的会话或输出摘要代替所选轮的完整会话。
  未结算的活动轮读顶层成员状态，已归档的最后一轮按上述轮号去重；取消、`continue` 吸收的失败和
  体积超限应能定位到项和成员，容量失败的完整视图矩阵仍缺直接证据。轮次详情在运行详情中展开；
  列表只取摘要的目标尚未落实到传输层，当前运行历史接口仍携带完整 `nodeStates`，不能宣称归档已从列表响应移除。

### 实现与 approved 方案的差异及边界

- 入口边是 D1 区域边界的明确例外：只表示容器到自身成员的入口，不改变区域前驱与执行顺序；保留
  desktop / 编辑器 wire，避免通过删除边来掩盖发布拒绝。
- 64 KiB 预留按**整次状态转移**开放：新失败 / 取消或新失败账本可以使用预留，已有历史失败不会放宽
  后续普通增长；相同已提交状态的重读允许。最后一项 `continue` 失败时，失败账本与最终暴露变量复制
  在同一次 CAS 中，因此这次复制也可能使用预留，仍受 8 MiB 硬上限约束。这比“仅诊断字节使用预留”的
  解释更宽；最后轮复制与下一轮普通增长已由 [cloud#198](https://github.com/ora-space/cloud/pull/198)
  的 PostgreSQL / 生产 CAS 边界测试直接验证，固定证据见核心用例。其他容量检查点、恢复与读取矩阵
  仍由 #743 承接，不把这两个边界扩大解释为完整容量保证已验证。
- 按轮会话入口由 [#619](https://github.com/ora-space/circus/issues/619) 及其读取实现承接；当前视图只保证
  已有轮次、成员状态、输出和错误的呈现。列表载荷摘要化及剩余边界 / 恢复 / 读取直接证据由
  [#743](https://github.com/ora-space/circus/issues/743) 承接，调整公开响应须遵守既有兼容性流程。
  这些缺口不因 ADR 标为 `implemented` 而消失，核心用例保留 `Partial` / `Missing`。
- 超项数运行错误已包含实际项数和配置上限，但没有专门的调整指引；D6 的这一项用户提示仍由 #743
  的上限验收承接，不把稳定失败码等同于完整提示已经交付。

### D6：失败与上限

- `N > maxIterations`：节点在启动边界失败，不执行任何轮，错误信息包含数组长度、上限值与调整指引。
  稳定原因是 `iteration_over_limit`。数组项数还受 100 的硬上限约束；在启动任何一轮前完成检查。
  **绝不截断后成功**：那会让作者以为 N 项都处理了。
- 每项输入、collect 值或归档超过 D5 单项预算：`iteration_state_size_exceeded`；整体 `node_states`
  超过可用字节预算：`workflow_state_size_exceeded`。两者都是容器自身失败，`continue` 不吸收，
  错误只带有界诊断，不复制导致超限的值。
- `errorStrategy = fail`（默认）：首个失败轮使 iteration 失败，run 失败，后续轮不执行。
- `errorStrategy = continue`：失败轮记入账本并继续；iteration 正常完成，失败留在 `entries` 与
  `failed_count`。
- iteration **自身**的失败（迭代源不是数组、暴露变量写入失败、超限）总是传播为 run 失败，不被自身策略吸收。
- 恢复发现轮号、账本、归档或派生值不一致：`iteration_invalid_round`，不重置轮次或重新执行已结算项。
- 无自动重试：失败轮不重跑。

## 不变量

1. 成员只由 `effectiveContainer` 决定，两个持久化键不会产生两套成员集合。
2. 计划轮数等于数组长度；空数组立即成功且账本为空。
3. 第 r 轮的 `item` / `index` 只在该轮区域内可见。
4. `output` 只含成功轮，`entries` 与已结算的输入前缀位置对齐，`failed_count` 等于失败轮数；三者类型固定。
5. `N > maxIterations` 时零轮执行并失败，绝不截断。
6. 区域内失败按策略处理；iteration 自身失败总是 run 失败。
7. 已结算的轮不可变更、不重跑；取消后不再启动新轮。
8. 不新增表、不改 OpenAPI、不改已发布图的 wire 形状。
9. 轮数与序列化字节数同时有界；超限零截断、失败原因稳定，既有账本仍可在运行详情读取。
10. 迭代源来自 iteration 节点在外层图上的传递前驱（globals / `sys.*` 豁免）；body 节点的前驱只在本区域计算，外层只导入可见祖先。
11. 同一轮只展示一次；condition 未选中成员使用既有 `inactive` 状态，归档与最终顶层投影不伪造执行。

## 为什么不是这些替代方案

| 方案 | 取舍 |
|---|---|
| 只认 `parentId`（desktop 的做法） | Cloud loop 的成员已经是 `containerId`，只认 `parentId` 会让 loop 区域在 Go 侧变空；统一归一代价更小 |
| 只认 `containerId` 并迁移 iteration 草稿 | 迁移存量草稿是兼容边界变更，收益只是少一个 fallback；归一读法同时兼容两者 |
| 超限时截断到上限并成功 | 正确性灾难：作者以为 500 项全处理、实际只跑 50 项；与仓库 fail-loud 哲学冲突 |
| 三模式下拉（Dify） | 会让 `output` 的类型随配置变化，下游引用被迫重校验；desktop 已用「策略 × 读哪个变量」解耦 |
| 把失败轮从 `entries` 中省略 | 丢失失败归因；账本必须与输入位置对齐 |
| 新增 scope / node-run 表（模型 Y） | 独立查询与恢复能力更强；本次按模型 X 沿用单行 CAS，并以轮数及字节预算约束代价。以后需要 Y 的能力时另开决策 |

## 本决策未解决的问题

- **并行迭代**：轮次串行；并行完成序与轮次序的编码留给后续决策（账本以轮次为键已为之留路）。
- **启动边界软警告**：迭代源来自 Start 输入时，部署界面可提前提示超限；权威检查仍在节点启动边界。
- **类型系统**：`collectSelector` 的声明类型相等校验依赖变量类型系统，当前不引入。
- **检查器体验**：selector 仍是文本输入；选择器化不改变契约。

## 落地与验证边界

核心实现见 [cloud#179](https://github.com/ora-space/cloud/pull/179)，入口边保真见
[cloud#189](https://github.com/ora-space/cloud/pull/189)，结构化 Start 输入准入见
[cloud#191](https://github.com/ora-space/cloud/pull/191)，逐轮视图见
[cloud#184](https://github.com/ora-space/cloud/pull/184) 与 [cloud#187](https://github.com/ora-space/cloud/pull/187)。
它们已经合并；固定版本的直接测试、真实栈证据及 CAS 耗时 / WAL 测量登记在
[核心用例](../../../test-cases/cloud/workflow/iteration-foreach-execution.md)。近 8 MiB 测量是整行提交基线，
不代表完整 100 轮运行成本或生产延迟承诺。未覆盖的边界按上节分工继续验证，不用共用调度器测试替代
iteration 专属恢复、会话与取消竞争的直接证据。
