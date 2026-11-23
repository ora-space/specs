# Iteration 的 foreach 执行、作用域与按轮账本

依据[Cloud iteration 执行契约](../../../decisions/cloud/workflow/20261025-iteration-foreach-execution.md)
（`implemented`，D1–D6）。按业务风险登记发布门禁、轮次执行、持久化、取消与容量边界。
源码证据核对基线为 cloud `09982ab350886b028cea8e0ed74a9a0abed7a43b`，来自已合并的
cloud#179、#182、#184、#187、#189、#191；源码链接固定到该提交。失败预留补充证据固定到已合并
cloud#198 的 `e328be2e6d05aa712ff24602e04367ded0f28ff3`，其[六项 CI][reserve-ci]通过，
PostgreSQL / 聚焦 race 验证记录见[补证运行][reserve-run]。本次源码核对未重跑测试；现场证据各自保留部署版本，
不把较早版本的观察当作最新 main 重跑，也不以测试名称、代码实现或共用测试代替完整义务的直接断言。
调度器共同义务见[调度串行与恢复](scheduler-serialism-and-recovery.md)。

本文的 `Partial` 表示已有直接证据只覆盖义务的一部分；每行写明尚缺范围。剩余承接：

- [#743][gaps]，@ElephantGit：G1 门禁/作用域，G2 数据/轮次矩阵，G3 恢复/取消竞争，G4 容量检查点/读取与视图矩阵。
- [#702][reserve-gap] 的最后失败轮复制、下一轮普通增长边界及双语说明已由 cloud#198 交付；对应直接义务见容量表，不再列为待补范围。
- [#619][sessions] / #691：完整节点会话读取及按轮入口；[共同恢复缺口 #665][recovery-gap] 保持原承接，不在此重复登记为已覆盖。

真实栈的快节点 foreach/fail/continue/超项数/空数组、IA1/IC1、I8、ID1 和两入口验收分别见 [快节点][live-fast]、[真实 Agent 与取消][live-agent]、
[main 入口边][live-entry]、[Agent 回答收集][live-output]、[两入口及拒绝副作用][live-input]。
I6 的超大请求在 HTTP 层先返回 413，不能证明 iteration 内部 1 MiB 检查。
现场版本边界：快节点与 IA1/IC1 使用 cloud `d27158a`，I8/ID1 使用已合并 main `0b25990`；
两入口首次复验使用 main `5a0a504`，完整拒绝矩阵使用 `f4bd031`（该 main 加独立的前端撤销修复），
desktop 均为 `d913440`。两入口补充报告记录 `deployment-ready`、`dirty:false`，手动 WorkflowRun
`645ffcc8`、IssueRun `9ab621fd` 关联 WorkflowRun `8c32ce3c`；后者的非法输入无运行/活动/claim 副作用。

## The Publish Gate Uses One Membership And A Deterministic Iterator Source

- 风险：两种容器键产生不同成员集合；并行兄弟节点偶然先提交值，被用作不确定的迭代源。
- 前置状态：图分别使用 `parentId`、非空 `data.containerId`、二者冲突或空 `containerId`；迭代源分别
  指向传递前驱、外层非前驱、自身、body 成员、其他容器成员及已声明 globals / `sys.*`。
- 触发：保存草稿、发布图，并在不同调度顺序下启动合法图。
- 必须成立：非空 `containerId` 优先，其余回落 `parentId`，所有读取共用同一成员集合且不回写草稿；
  来源根必须已声明，其节点必须是 iteration 在外层图上的传递前驱，globals / `sys.*` 仅豁免前驱检查。
  非法来源返回 `iteration_iterator_undeclared` 并定位节点和字段；D2 其他配置与区域错误也明确拒绝。
- 禁止结果：外层非前驱因为恰好有状态就被放行；成员或自身提供迭代数组；发布自动截断配置或改写草稿。
- 验证义务与证据：

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 成员解析在缺失、空值与冲突时一致 | [成员解析][membership] 的 `TestWorkflowContainerIDReadsBothMembershipKeys` 覆盖缺失与冲突；[入口边测试][entry] 覆盖显式键优先和 parentId 图执行。尚缺空字符串及冲突同图贯穿调度的完整矩阵（G1） | Partial |
| 传递前驱与已声明环境根合法，其他来源按命名错误拒绝 | [作用域门禁][scope] 的 `TestWorkflowIterationProductionGate`、`TestWorkflowAmbientSelectorsNeedDeclarationsButNotEdges` 拒绝自身/非前驱与未声明环境根。尚缺非前驱已有值的运行反例及全部来源矩阵（G1） | Partial |
| D2 九个拒绝码定位节点与字段，合法草稿仍可保存 | [配置矩阵][config] 检查九码及适用字段，[生产门禁][scope] 验证接线；[现场导入发布][live-import] 记录拒绝且未生成版本。尚缺九码完整 HTTP/PG 定位与草稿/快照副作用矩阵（G1） | Partial |
| 可选入口边保真且不成为区域依赖，其他跨区域边仍拒绝 | [入口边测试][entry] 的 Publication、DoesNotBecomeADependency、RunsWithAndWithoutEdge 检查完整问题对象、ready 成员、门禁/运行前驱及两轮相同结果；[HTTP 草稿发布][entry-http] 保留 nodes/edges；[I8][live-entry] 真实栈发布并执行成功 | Covered |
| 两种表单入口将声明的结构化 Start 输入送入运行，非法输入先拒绝 | [三入口 HTTP/PG 回归][input-http] 读回原生数组/对象并检查拒绝后的运行、活动、claim；[准入矩阵][input-unit] 覆盖原生输入、typed array、编码错误和普通字符串不转换；[两入口现场][live-input] 成功执行及拒绝无副作用 | Covered |

- 决策依据：D1、D2，不变量 1、10。

## An Empty Iterator Succeeds Without Starting Any Member

- 风险：空数组误报无入口失败，或创建一次空轮 / Agent 会话。
- 前置状态：合法 iteration 区域，迭代源为 `[]`。
- 触发：执行至该 iteration。
- 必须成立：立即成功，`output=[]`、`entries=[]`、`failed_count=0`；持久化的 `items`、`ledger`、
  `output`、`completed` 均为空数组，`round=0`；不创建任何成员执行或归档。
- 禁止结果：启动成员、生成第 0 轮账本条目、把数组编码为 null 或以缺收集值为由失败。
- 验证义务与证据：

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 空数组输出与持久化形状正确，零成员执行 | [调度器][scheduler] 的 `TestWorkflowAdvanceIterationEmptyArraySucceeds` 断言 succeeded 与空输出/账本、零失败；[现场空数组][live-fast] 通过。测试用长度判断，不能证明全部字段严格为 [] 而非 null，也未计数工作项/会话（G2） | Partial |

- 决策依据：D3–D5，不变量 2。

## An Oversized Iterator Fails Before The First Round

- 风险：先执行到上限才失败，或截断输入后以成功结束。
- 前置状态：合法 `maxIterations=M`，来源分别为长度 M、M+1 的数组；配置缺省、100、101 或非法类型。
- 触发：发布配置并启动已发布的合法图。
- 必须成立：缺省 50，配置只能为 `1..100` 整数；配置非法以 `iteration_max_iterations_invalid` 拒绝。
  N>M 在启动边界以 `iteration_over_limit` 失败，错误含 N、M 和调整指引，任何成员均未启动。
- 禁止结果：处理前 M 项后成功或失败；`continue` 吸收项数超限；自动把 101 改成 100。
- 验证义务与证据：

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 缺省、硬上限与非法配置在发布时区分 | [配置矩阵][config] 拒绝 0/小数；[生产门禁][scope] 接受 100、拒绝 101；[现场][live-import] 超限草稿保留且发布拒绝。尚缺缺省 50、全部非法类型及默认策略直接断言（G2） | Partial |
| N=M 正常运行，N=M+1 在两种策略下都零轮失败 | [调度器][scheduler] 的 RefusesAnOverLimitArray 覆盖 fail；[启动拒绝][invalid-source] 的 ContinueRejectsInvalidSourceBeforeFirstRound 覆盖 continue 且无成员状态；[现场超项数][live-fast] 失败。尚缺 N=M 精确边界及两策略的账本/归档/工作项完整计数（G2） | Partial |
| 超项数诊断包含实际 N、上限 M 与调整指引 | [启动实现][iteration-run-current] 的 iteration_over_limit 仅提供 N 与 M，尚无调整指引；现有失败码断言不能证明完整提示，仍需实现和直接验证（G2） | Partial |

- 决策依据：D2、D6，不变量 5。

## Every Round Sees Only Its Own Item Index And Regional Predecessors

- 风险：本轮读到上一轮的变量、其他容器成员或非祖先产物；未选中的分支被当成执行成功。
- 前置状态：至少两项的迭代源，body 含 condition、两条分支和 aggregator；外层含可见祖先及并行非祖先。
- 触发：两轮选择不同分支；区域内外分别引用 item/index、成员根、外层来源及容器最终暴露根。
- 必须成立：第 r 轮只有 `items[r]` / r；body 节点的传递前驱只在本区域计算，aggregator 对区域内候选
  检查传递前驱，外部只允许 iteration 可见祖先、globals / `sys.*` 及本轮 item/index。item/index 不向
  外层暴露，最终三个暴露根不注入 body；未选中成员写为既有 `inactive`，不输出值，不创建会话。
- 禁止结果：外层非祖先、其他容器或上一轮状态成为本轮前驱 / 值来源；已运行成员被改为 inactive；
  外层直接引用 item/index 或 body 根。
- 验证义务与证据：

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 区域前驱与显式导入边界在门禁和取值时一致 | [作用域门禁][scope] 的 BodyScopeIsolation 覆盖 body aggregator、祖先/item 与外层兄弟/其他容器拒绝；[入口依赖][entry] 对齐两套前驱。尚缺同矩阵的实际变量池读取（G1） | Partial |
| 两轮绑定独立，condition 选择与 inactive 按轮归档 | [入口执行][entry] 检查两轮成员分别输出自己的 item；[inactive collect][body-failure] 检查单轮 inactive 及归档。尚缺两轮相反分支、index 与会话计数联合断言（G1/G2） | Partial |
| item/index 轮内私有、外层只读最终暴露根 | [入口执行][entry] 直接核对 item 透传和最终三个根；[ID1][live-output] 读取真实 Agent 本项回答。未证明 index 私有与外层引用拒绝的完整矩阵（G1） | Partial |

- 决策依据：D2、D3、D5，不变量 3、10、11。

## A Collect Target Must Have Succeeded In The Current Round

- 风险：收集目标本轮没有运行，却收集到上一轮残留值或把缺值当成成功的 null。
- 前置状态：第 0 轮成功运行收集目标，第 1 轮 condition 绕开该目标；两种 errorStrategy 分别运行。
- 触发：第 1 轮 body 结算并收集输出。
- 必须成立：只收集本轮 `succeeded` 目标的根；未运行目标使本轮失败，说明为
  `collect target did not run this round`，账本失败条目 `output=null`，不进入成功 output 数组。
  fail 停止，continue 记录失败后推进下一轮。
- 禁止结果：读取第 0 轮的值、为 inactive 目标合成输出、无错误地跳过该输入项。
- 验证义务与证据：

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 相反分支选择时收集缺失稳定记失败，策略边界正确 | [TestWorkflowIterationArchivesInactiveCollectTarget][body-failure] 证明 continue 下未执行目标不被收集且 inactive 入档；仅一轮，缺先成功后绕开的两/三轮场景、精确错误信封和两策略检查（G2） | Partial |

- 决策依据：D4、D6，不变量 4、6。

## Fail Stops At The First Failed Round While Continue Keeps Its Ledger

- 风险：continue 掩盖容器自身失败，或 fail 丢失失败轮证据 / 继续启动后续项。
- 前置状态：三项数组，第 1 轮成员失败；分别采用缺省、显式 fail、continue。
- 触发：执行失败轮；另以非数组来源、暴露写入失败和字节超限触发容器自身失败。
- 必须成立：fail（含缺省）先提交失败轮账本和归档，再使容器与 run 失败，后续项不执行；continue 记录
  失败轮后执行后续项，最终成功并保留错误。容器自身失败在两种策略下都终止 run，不伪装成可吸收的轮失败。
- 禁止结果：自动重跑失败轮；遗漏失败账本；以 continue 吸收非数组、暴露写入或预算失败。
- 验证义务与证据：

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| fail 停止、continue 完成，失败轮两者均留档且无自动重跑 | [FailureLedgerAndFinalProjection][guards] 精确检查两策略终态、轮号、账本/归档/失败数与最后成员投影；[现场 fail/continue][live-fast] 通过。尚缺中间轮失败、默认策略及逐轮派发次数完整断言（G2） | Partial |
| 容器自身失败不被策略吸收 | [ContinueRejectsInvalidSourceBeforeFirstRound][invalid-source] 覆盖非数组/超项数；[OversizedInputNeverStartsRound、BudgetCASPreservesCommittedState][guards] 覆盖输入和成员体积失败。尚缺最终暴露提交失败及两策略/各检查点组合（G4） | Partial |
| body Agent 超时在结束确认前不得推进下一轮，确认耗尽有界失败 | [TestWorkflowIterationTimeoutDrainsBeforeNextRound][drain] 检查重复 pass 只有一个结束命令、确认后才推进并归档原因，确认超时无后续 attempt 且 Workspace 释放 | Covered |

- 决策依据：D4、D6，不变量 6、7。

## Exposed Variables Keep Their Types And Input Order

- 风险：切换策略改变下游变量类型，或 output / entries 因完成顺序、假值而错位。
- 前置状态：成功、失败混合输入，成功值包含 null、false、空字符串及对象；两种策略有合法完成的运行。
- 触发：读取已完成 iteration 的三个根与中途失败运行的内部账本。
- 必须成立：成功终态 output 为仅成功项的有序数组，entries 为按输入位置排列的信封数组，failed_count
  为失败条目数；信封四个键齐全，成功 error=null，失败 output=null，以 status 区分成功 null 与失败。
  fail 提前结束的内部账本只含已结算前缀，不伪造后续条目。
- 禁止结果：把成功 null / false / 空串过滤掉；把失败项混进 output；按策略切换 array 与其他类型。
- 验证义务与证据：

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 类型固定、假值保真、过滤与完整账本的顺序一致 | [入口执行][entry] DeepEqual 完整两项成功输出和信封；[调度器][scheduler] 检查 continue 失败数及空成功输出；[ID1][live-output] 两项 Agent 回答有序收集。尚缺 null/false/空串/对象与混合成功失败的完整矩阵（G2） | Partial |
| fail 终态账本为连续已结算前缀 | [FailureLedgerAndFinalProjection][guards] 首轮失败时仅一条账本/归档且 round=0；尚缺中间轮失败及信封的完整前缀比较（G2） | Partial |

- 决策依据：D4、D5，不变量 4。

## Recovery Uses One Committed Round Document Without Replaying Settled Work

- 风险：恢复重新读取迭代源或清空账本，从第 0 轮重跑；半次 CAS 形成轮号与归档不一致。
- 前置状态：第 0 轮已归档，第 1 轮分别处于未派发、已有结果、在途会话；输入来源之后发生变化。
- 触发：换调度器实例接手，重复 / 并发 pass，轮结算前后模拟 CAS 失效；另加载不一致的轮次文档。
- 必须成立：只读已冻结 items 与 D5 wire 字段；账本、派生值、归档、清空顶层及轮号推进原子提交。
  已结算轮和活动轮已提交结果不重派，在途会话持有到自身结果；无下一轮时保留最后启动的 round，
  空数组 round=0。非法轮次文档以 `iteration_invalid_round` 失败，不能按“未开始”重新执行。
- 禁止结果：改变 items、重复 append 同轮账本、归档缺项、CAS 失败仍推进、换副本重建在途会话。
- 验证义务与证据：

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| wire 编解码与 round / ledger / completed 的关系符合 D5 | [ProgressSurvivesStateJSON][codec] 检查运行中 round-trip 及取消不改账本；[终态投影][guards] 检查两策略轮号。尚缺全部终态和空数组逐字段 round-trip（G3） | Partial |
| 接手不重跑已结算轮 / 已提交成员或在途会话 | [入口执行][entry] 证明终态重复推进的版本/状态不变；[结束屏障][drain] 证明在途确认期间重复 pass 不派发新 attempt。尚缺 iteration 专属换 Store、并发接手、冻结来源变化和活动轮已提交成员计数（G3），不能用普通 DAG 的恢复证据补齐 | Partial |
| CAS 原子性与非法轮次稳定失败 | [BudgetCASPreservesCommittedState][guards] 直接拒绝旧版本 CAS；[RejectsCorruptRecovery][budget] 检查缺字段/非法轮号的稳定原因及矛盾账本校验。尚缺轮结算 CAS 故障和损坏文档经生产 DB 入口的组合（G3） | Partial |

- 决策依据：D5，不变量 7、11，根决策 D1 / D4。

## Cancellation Preserves History And Never Starts Another Round

- 风险：取消只改容器状态但下一轮仍被派发；归档被清空，取消不能定位活动项。
- 前置状态：至少两轮，取消分别发生在活动轮有在途会话及两轮交界处。
- 触发：请求取消并重复推进调度器，等待既有有界收尾。
- 必须成立：当前轮按既有路径停止，会话和 Workspace 按运行取消路径收尾；容器最终为 cancelled，
  已提交 completed / ledger 不变，活动轮状态保留可读；请求被提交后不启动新轮。
- 禁止结果：新建后续轮会话；伪造未完成轮的成功账本；取消后丢掉已结算记录或无界等待。
- 验证义务与证据：

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 轮中与轮边界取消均不启动新轮，历史可读且有界收尾 | [CancellationDrainsInflightAgent][body-failure] 检查等待确认、当前轮取消、Workspace 释放；[状态 round-trip][codec] 保留已有账本；[IC1][live-agent] 第 2 项取消且第 3 项未开始。尚缺轮边界竞争和同一用例的全部历史/资源联合断言（G3） | Partial |

- 决策依据：D5，不变量 7。

## Byte Limits Fail Stably Without Truncating Values Or Losing History

- 风险：仅限制轮数仍写爆运行行；错误说明复制超大值挤掉失败预留；continue 把容量失败当成成功。
- 前置状态：紧凑 UTF-8 JSON 编码的输入项、collect 值、单轮归档分别在 1 MiB 边界内外；完整
  node_states 接近 `8 MiB - 64 KiB`，包括输入、当前成员、归档、账本与输出的重复值。
- 触发：启动检查、成员提交、轮结算和最终暴露提交；两种策略分别触发超限。
- 必须成立：每项输入 / collect / 轮归档各不超过 1 MiB，普通提交不使用 64 KiB 失败预留，总量不超过
  8 MiB。单项超限以 `iteration_state_size_exceeded`、整体超限以 `workflow_state_size_exceeded` 终止，
  保留上次已提交状态，错误只含有界的容器 / 轮次 / 字节数 / 上限，既有账本与取消 / 失败记录不丢失。
- 禁止结果：逻辑去重后计费、漏算 UTF-8 或重复值、截断值 / 删除历史后记成功、continue 吸收预算失败。
- 验证义务与证据：

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 每个检查点与两类字节边界按实际编码生效 | [预算测试][budget] ExactJSONBytes 精确断言 UTF-8/转义输入 1 MiB 与 +1，LocalObligations 检查 collect/归档超限，TotalBudgetCountsRepeatedValues 检查重复值；[DB][guards] 覆盖多字节输入和成员提交。尚缺所有检查点的等于/超出边界与两策略 DB 矩阵（G4） | Partial |
| 超限保留既有提交，并在失败预留内稳定结算 | [BudgetCASPreservesCommittedState][guards] 检查旧成员/来源未变、候选未泄漏、总量不超硬限；[预留测试][reserve] 检查历史失败不能放宽新增长、相同状态可重读及新失败结算。下两行补齐最后轮复制与下一轮增长；其他检查点的完整诊断/账本/后续派发矩阵仍由 G4 承接 | Partial |
| 最后一轮 continue 失败的账本与最终输出复制在同次 CAS 共用预留，仍受 8 MiB 硬限约束 | [TestWorkflowLastFailedIterationOutputCopyUsesBoundedReserve][reserve-boundary] 通过真实 PostgreSQL、生产 round engine 与 CAS，精确检查普通预算、普通预算+1、8 MiB−1/恰好/+1。去掉最终输出复制后候选仍在普通预算内；允许范围内完整持久化，超硬限时拒绝整个候选并保留旧账本和成员 | Covered |
| 已提交失败轮超过普通预算后，下一轮普通增长拒绝且终态重放幂等 | [TestWorkflowContinueAfterReservedFailureRejectsOrdinaryGrowth][reserve-boundary] 使用两个 item，先提交失败轮再由调度器从 PostgreSQL 读取；断言下一轮以 workflow_state_size_exceeded 失败、账本/归档保留、总量不越 8 MiB，两次终态调度保持完整状态和版本 | Covered |
| 接近总预算时整行 CAS 的耗时与 WAL 成本可复核 | [可复现测试][measurement] 使用生产 settleWorkflowRunStates 与真实事务；[原始测量][measurement-report] 记录负载、版本、逐次耗时和 WAL LSN，范围见下文 | Covered |

测量边界：WSL/Docker、Go 1.27.1、PostgreSQL 17.11 x86_64，`wal_compression=off`，独占测试数据库；
`MEASURE_WORKFLOW_CAS=1 REQUIRE_POSTGRES=1 go test ./internal/core -run TestWorkflowNearLimitCASMeasurement -count=1 -v`。
24 项、12 个已结算轮，固定种子 base64 数据及 2,111,786 字节 padding，包含 source/items/ledger/output/archive
的物理重复，紧凑 JSON 8,322,048 字节，距普通上限 1,024 字节。10 次 CAS 合计 1.661229 s、平均 166.123 ms；
WAL insert LSN 从 `0/791ABA58` 到 `0/7E8A38B0`，增量 91,192,920 字节。结果来自 #660 的记录，
本次未重新测量；只证明该负载的整行 CAS 基线，不证明 100 轮完整成本或生产性能。

预留边界测试从受控的合法失败成员检查点开始，直接证明生产 CAS 与后续调度的行为；不等同于真实 Agent
从发布到完成的端到端执行，也不覆盖所有局部 1 MiB 检查点、两策略组合、活动轮 UI 归因或并发恢复。
正式说明见[中文容量契约][reserve-doc-zh]与[英文容量契约][reserve-doc-en]。

- 决策依据：D5、D6，不变量 9、落地与验证边界。

## RunDetails Show Each Round Once And Preserve Member Attribution

- 风险：最后一轮同时来自顶层和归档而显示两次；continue 失败、inactive 或取消活动轮失去归因。
- 前置状态：成功、fail、continue、取消与预算失败运行，含已归档轮和未结算活动轮。
- 触发：读取运行详情和运行列表，展开 iteration 的逐项账本及逐轮成员。
- 必须成立：按 `(iteration id, index)` 唯一展示轮次；`completed[index=round]` 存在时只展示归档，
  否则顶层成员显示为未结算活动轮。能定位 item / index、成员输出、错误、会话和 inactive“未到达”；
  列表只给摘要，旧读取方仍能读取既有状态字段并忽略新增 iteration 文档。
- 禁止结果：显示第 N+1 轮、把最后一轮算两遍、把未选中成员显示为成功、丢掉取消或容量失败的活动轮。
- 验证义务与证据：

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 活动 / 最终轮去重，成员和错误归因完整 | [轮次投影][rounds] 检查三项含 continue 失败、fail 后无后续轮、空源及活动轮；[详情组件][round-views] 检查 item/collect/成员与失败账本，现场 [I8][live-entry]、[IC1][live-agent]、[ID1][live-output] 补运行视图。普通节点会话区已交付，但[详情接线][detail-current]未向会话区传递所选历史轮的 attempt，按轮会话绑定仍缺（#619）；iteration 容量失败完整呈现矩阵尚缺（G4） | Partial |
| API 兼容且列表不展开归档 | [历史 API][run-api] 的 workflowRunSummary 仍返回完整 nodeStates；现有轮次解析测试不能证明旧读取方兼容或列表载荷隔离。该完整义务尚无直接证据，且摘要载荷目标未实现，由 G4 核对兼容方案及直接测试 | Missing |

- 决策依据：D3、D5，不变量 11。

<!-- 固定源码证据；现场报告保留各自部署版本。 -->
[membership]: https://github.com/ora-space/cloud/blob/09982ab350886b028cea8e0ed74a9a0abed7a43b/internal/core/workflow_loop_graph_test.go
[entry]: https://github.com/ora-space/cloud/blob/09982ab350886b028cea8e0ed74a9a0abed7a43b/internal/core/workflow_iteration_entry_test.go
[entry-http]: https://github.com/ora-space/cloud/blob/09982ab350886b028cea8e0ed74a9a0abed7a43b/integration/workflow_iteration_entry_test.go
[scope]: https://github.com/ora-space/cloud/blob/09982ab350886b028cea8e0ed74a9a0abed7a43b/internal/core/workflow_scope_test.go
[config]: https://github.com/ora-space/cloud/blob/09982ab350886b028cea8e0ed74a9a0abed7a43b/internal/core/workflow_iteration_graph_test.go
[input-http]: https://github.com/ora-space/cloud/blob/09982ab350886b028cea8e0ed74a9a0abed7a43b/integration/workflow_structured_input_test.go
[input-unit]: https://github.com/ora-space/cloud/blob/09982ab350886b028cea8e0ed74a9a0abed7a43b/internal/core/workflow_admission_test.go
[scheduler]: https://github.com/ora-space/cloud/blob/09982ab350886b028cea8e0ed74a9a0abed7a43b/internal/core/workflow_iteration_scheduler_db_test.go
[guards]: https://github.com/ora-space/cloud/blob/09982ab350886b028cea8e0ed74a9a0abed7a43b/internal/core/workflow_contract_guards_db_test.go
[body-failure]: https://github.com/ora-space/cloud/blob/09982ab350886b028cea8e0ed74a9a0abed7a43b/internal/core/workflow_body_failure_db_test.go
[invalid-source]: https://github.com/ora-space/cloud/blob/09982ab350886b028cea8e0ed74a9a0abed7a43b/internal/core/workflow_review_obligations_test.go
[drain]: https://github.com/ora-space/cloud/blob/09982ab350886b028cea8e0ed74a9a0abed7a43b/internal/core/workflow_session_drain_db_test.go
[codec]: https://github.com/ora-space/cloud/blob/09982ab350886b028cea8e0ed74a9a0abed7a43b/internal/core/workflow_iteration_state_test.go
[budget]: https://github.com/ora-space/cloud/blob/09982ab350886b028cea8e0ed74a9a0abed7a43b/internal/core/workflow_state_budget_test.go
[reserve]: https://github.com/ora-space/cloud/blob/09982ab350886b028cea8e0ed74a9a0abed7a43b/internal/core/workflow_failure_reserve_db_test.go
[reserve-boundary]: https://github.com/ora-space/cloud/blob/e328be2e6d05aa712ff24602e04367ded0f28ff3/internal/core/workflow_failure_reserve_boundary_db_test.go
[reserve-ci]: https://github.com/ora-space/cloud/actions/runs/40618547983
[reserve-run]: https://github.com/ora-space/circus/issues/745
[reserve-doc-zh]: https://github.com/ora-space/cloud/blob/e328be2e6d05aa712ff24602e04367ded0f28ff3/docs/workflows/workflows.md
[reserve-doc-en]: https://github.com/ora-space/cloud/blob/e328be2e6d05aa712ff24602e04367ded0f28ff3/docs/workflows/workflows.en.md
[iteration-run-current]: https://github.com/ora-space/cloud/blob/e328be2e6d05aa712ff24602e04367ded0f28ff3/internal/core/workflow_iteration_run.go
[detail-current]: https://github.com/ora-space/cloud/blob/e328be2e6d05aa712ff24602e04367ded0f28ff3/frontend/src/features/workflows/run/run-detail-page.tsx
[measurement]: https://github.com/ora-space/cloud/blob/09982ab350886b028cea8e0ed74a9a0abed7a43b/internal/core/workflow_cas_measurement_test.go
[rounds]: https://github.com/ora-space/cloud/blob/09982ab350886b028cea8e0ed74a9a0abed7a43b/frontend/src/features/workflows/run/run-rounds.test.ts
[round-views]: https://github.com/ora-space/cloud/blob/09982ab350886b028cea8e0ed74a9a0abed7a43b/frontend/src/features/workflows/run/run-round-views.test.tsx
[run-api]: https://github.com/ora-space/cloud/blob/09982ab350886b028cea8e0ed74a9a0abed7a43b/internal/core/workflow_run.go
[live-fast]: https://github.com/ora-space/circus/issues/688#issuecomment-6416677057
[live-agent]: https://github.com/ora-space/circus/issues/688#issuecomment-6432713447
[live-entry]: https://github.com/ora-space/circus/issues/688#issuecomment-6433659478
[live-output]: https://github.com/ora-space/circus/issues/688#issuecomment-6435562917
[live-import]: https://github.com/ora-space/circus/issues/688#issuecomment-6438442434
[live-input]: https://github.com/ora-space/circus/issues/688#issuecomment-6438477946
[measurement-report]: https://github.com/ora-space/circus/issues/660#issuecomment-6407674257
[gaps]: https://github.com/ora-space/circus/issues/743
[reserve-gap]: https://github.com/ora-space/circus/issues/702
[sessions]: https://github.com/ora-space/circus/issues/619
[recovery-gap]: https://github.com/ora-space/circus/issues/665
