---
status: approved
---

# 残留 Agent Run 资源以真实证据分层回收：证据齐沿 owned delete，未确认执行经部署运维受控回收，账本按目标闭环清除

D4/D5 业务收尾留下的残留账本驱动持续对账。自动对账层周期收集真实证据——迟到的执行终态与交付结果、执行已终结、沙盒已物理终止——证据齐的条目取得本文新增的系统清理授权后，复用 `delete_workspace` owned delete 的执行步骤清理，成果能救则先救；清理完成前账本保护不解除。证据收不齐的条目只能由部署运维按租户与资源范围发起受控回收：显式确认放弃未交付成果，复用管理强停的终止与围栏语义（终止与数据删除分离、逐目标确认 `terminated` 与 `lateEnsureFenced`、不因超时判成功），经当前 Controller lease 与既有 Substrate effects 执行，不新增物理路径、不伪造 `impactConfirmed`、不由业务 `done` 隐式强停。账本条目在全部目标资源取得闭环证据或显式处置记录后清除；Substrate 永久不可用时保留未完成意图，由运维外部确认兜底。存量 D8 删除残留按原身份接入同一路径，不改写历史。

本文为 `approved`：决策方向已经 owner 批准，核心测试用例见[残留资源回收用例](../../../test-cases/cloud/issue-run/residual-resource-reclamation.md)。实现尚未合入 cloud、desktop、cluster 的 main，用例义务如实标为 `Missing`；实现合入并完成联合验证后，再由落地运行按最终实现同步正文并改为 `implemented`。

本文承接 [有界启动与未派发撤回决策](20261012-bounded-agent-startup-and-undispatched-withdrawal.md)（下称“启动决策”）D4 预留的“Agent Run 残留资源对账与回收”ADR。联合职责为 Cloud 控制面（账本、对账编排、受控回收意图）、Controller/Fleet（许可、对账与派发证据）与 Substrate（终止与围栏执行）。涉及 cloud 的 `internal/core`（残留账本、operation、受控回收意图）与部署管理端点；desktop Controller 与 cluster Sandbox Server 不新增机制，只按既有决策提供证据与执行。兼容性边界：不改启动决策 D4/D5 的结算语义、不改根 D8 残留的状态与记录、不开放隐藏运行 Workspace 的公开操作入口；`agent_run_residuals` 表只增列。本文不承担对象存储中 Revision 的过期清理、告警与通知机制、以及生产 Substrate 适配器的具体实现。

## 继承与修改

| 前序约定 | 本文处理 |
|---|---|
| 启动决策 D4：残留责任表指定“后续回收 ADR”负责清理授权、任务持久化与重试、真实终止、late ensure fencing、成果救援与删除顺序、运维入口与失败升级 | 本文承担全部指定项；责任表中 Cloud/Controller/Node 的既有职责不变 |
| 启动决策 D4/D5：业务 `done/failed` 保留残留账本与资源保护，真实终态及交付/放弃证据齐全时可沿已有 owned delete 清理 | 继承执行步骤；新增按原资源身份和完整证据授权的系统入口，清理完成前仍保留账本保护，普通删除入口不获得旁路（D2） |
| 启动决策 D5：事实接管入口按原 effect/执行身份幂等补证据，不重开 admission、不再结算 | 继承并扩展接管时段至清理中和 `cleared` 后；不得仅因账本已清除而拒收同身份真实事实，不改 Run 终态（D2） |
| 根 D8：删除侧仅在存活沙盒的 Node 未知超过不可达窗口时放弃，残留如实保留，运维处置，无后台重试 | 继承条件、状态与不自动重试的边界；D6 仅新增按原身份发起受控回收的入口，不改写原结算来源 |
| 根不变量 8：`done` 不代表运行 Workspace 已删除 | 继承并加强：账本清除同样需要资源处置证据，业务终态不是清除条件（不变量 2） |
| [管理强停](../operation/20260927-safe-stop-and-administrative-force-stop.md)：强停是独立授权意图，`impactConfirmed=true`，终止与数据删除分离，逐目标确认 | 继承语义并扩展授权边界：隐藏运行 Workspace 对公开 API 为 404，租户管理员无入口；受控回收由部署运维发起，确认语义同强停（D3） |
| [运行时控制围栏](../controller-integration/20260927-fenced-runtime-control-delivery.md) D4：终止确认包括 `terminated` 与 `lateEnsureFenced`，紧急终止用独立短期 permit | 继承；受控回收的完成条件沿用双确认，permit 规则不变 |
| [Docker 沙盒生命周期](../../cluster/sandbox-server/20260926-docker-sandbox-lifecycle-via-substrate-effects.md) D6：`sandbox_terminate` 以容器不存在为证据、先写墓碑，目标从未 ensure 过也确认终止与围栏 | 继承为回收完成证据；对生产 Substrate 适配器提出同等语义要求（D4） |
| 启动决策 D4“需要强停时遵循强停与围栏原则，但对隐藏运行 Workspace 的授权扩展由回收 ADR 决定，不能伪造 `impactConfirmed`” | 本文即该扩展：授权归部署运维的受控回收命令，自动层与业务终态都无此授权 |

## 决策

### D1：残留账本增加清理状态机，清除需要资源处置证据

`agent_run_residuals` 每行增加清理状态 `reclaim_state`，取值 `reconciling → reclaiming → cleared`：

| 状态 | 含义 | 进入条件 |
|---|---|---|
| `reconciling` | 自动收集证据，尚未取得清理推进责任；业务结算写入账本时的初始状态 | D4/D5 登记账本（现状不变） |
| `reclaiming` | 一个持久清理责任在推进；记录自动 owned delete 或运维受控回收模式 | D2 的系统授权或 D3 的显式命令在同一竞争边界内取得责任 |
| `cleared` | 残留责任闭环 | 全部目标取得闭环证据或显式处置记录（D4/D5），写入 `cleared_at` 与处置方式 |

- `cleared` 的处置方式 `disposition` 取值：`released`（沙盒终止且数据已按 owned delete 或受控回收删除）、`terminated_data_retained`（终止与围栏已确认，数据按运维决定保留）、`externally_confirmed`（Substrate 之外的外部确认，D5）。
- 对账证据逐条记录在账本行上（证据类型、来源、时间），不改写 `reason` 与既有 `responsibility`；重复对账只追加证据，不重建 Run、Workspace、ensure 或执行身份。
- 存量行（D4/D5 已登记的）迁移时统一置 `reconciling`，不回溯判定证据。
- 清理责任固定原租户、Project、Run、Workspace、sandbox/generation 与计划/执行身份，并关联唯一推进意图或 operation、模式及授权证据。推进者重启或 Controller lease 更替只能接管同一责任，不能取得第二份删除或终止权。

为什么给账本加状态而不是新建清理任务表：账本行已经是残留责任的唯一权威（启动决策 D4），清理是这份责任的收尾，分表会让“谁还欠什么”需要两表对齐；`cleared_at` 列在 D4/D5 落地时已预留。

### D2：自动对账层只收集真实证据并沿 owned delete 清理，永不强停

Cloud 以可配置周期（默认 1 分钟）扫描 `reconciling` 条目，按下表收集证据。表中每行只是证据的一部分；必须覆盖原责任集合、已封闭新派发，且所有执行已终结或其全部可写实例已取得终止与围栏双确认，交付已验证完成或已有明确放弃依据，才有资格申请系统清理授权。单个终态、单个沙盒确认或“没有结果记录”都不足以代替完整证明。

| 证据 | 来源 | 允许的清理动作 |
|---|---|---|
| 迟到真实终态与交付结果 | 迟到事实按原执行身份接管（既有路径） | 补登记交付（成果救援），随后沿 owned delete；交付结果未确认时按启动决策保持 `deliveryState` 已结算值 |
| 执行已终结、无成果记录 | Cloud 重放查询原执行身份；Controller 按[围栏交付](../controller-integration/20260927-fenced-runtime-control-delivery.md) D3 回报封闭状态与未完成 execution IDs | 仍须核对交付完成或明确放弃依据；完整证据齐后经下述新增授权，以原资源身份声明 owned delete，quiesce 通过后释放 |
| 沙盒已物理终止与围栏 | 经 Controller 查询 effect 状态：`sandbox_terminate` `succeeded`，或 ensure 从未执行且 terminate 已确认 | 沙盒风险已消除；同一 `delete_workspace` 继续推进数据删除步骤 |
| 以上都取不到 | — | 保持 `reconciling`，等待下一周期或运维受控回收 |

**新增授权与保护转换**：当前普通 `deleteRunWorkspace` 会拒绝 `cleared_at IS NULL` 的残留，创建侧事实接管也只接受未清除账本；因此不能直接把现有入口称为可用，更不能先填 `cleared_at` 再调用它。本文要求新增 Cloud 系统清理入口，执行步骤复用 owned delete，授权按以下规则改变：

1. 在与原资源计划、派发登记、账本事实写入和 D3 受控回收互斥的事务边界，重读原资源关联及完整证据，固定全部待处置目标，关闭后续新派发资格。只针对这次残留和原资源授予清理权，不授予新会话、ensure 或换代权；Project/Workspace 的既有冲突及范围保护仍须核验。
2. 取得单一持久责任并置 `reclaiming`，同时声明或关联唯一 owned delete operation；事务失败全部回滚。自动层与受控命令竞争时只有一方取得推进权，同一输入重放返回原责任，另一模式冲突不能并行计划副作用。自动层不能自行把模式升级为受控回收；需要强停时仍须 D3 的人工确认，已发动作先按原责任对账，不用覆盖意图制造第二个执行者。
3. `reclaiming` 期间 `cleared_at` 保持空值。只有绑定该责任的系统动作可越过针对本条目的残留阻挡；公开 API、普通删除及涉及残留的 Project stop/delete 继续受保护。执行仍依次核验 quiesce、逐目标终止与围栏、数据删除证据，不能因已有清理授权跳过任何物理证据。
4. 只有全部目标闭环后，才在核对最新责任及处置证据的事务中写 `cleared_at`、处置方式和完成关联；不能先清账以腾出删除入口。`terminated_data_retained` 只证明终止责任已收口，保留的数据仍归原 Workspace，不隐含后续删卷授权。

**事实接管贯穿清理前后**：创建侧事实入口与原执行接管必须扩展到 `reconciling`、`reclaiming` 和 `cleared`，仍验证原租户、资源、effect/execution 身份及当前服务资格；不能用 Run `done` 或账本已清除拒收。先持久接管再 ACK：相同事实幂等重放，补充事实只追加证据，不重开 admission、不再次结算、不重建已删除资源。事实与清理授权/完成竞争时，后提交者须看到先提交者的证据与责任；新发现的未闭环目标阻止清除。清除后若事实与处置结论冲突，保留原结论及冲突审计，阻断受影响资源的后续删除/复用并交运维核查，不静默丢弃事实或伪造新的完成证明。

- 自动层的边界：只做证据收集、成果救援与上述系统授权的 owned delete；它**不发起任何强停、不排针对未知执行的数据删除、不把证据不足解释为已放弃**。quiesce 拒绝说明当前不能安全清理，保留同一持久责任和保护，重新取证后按 owned delete 规则重试，不因 Run 已 `done` 丢失任务，也不自动升级。
- 成果救援只经既有链路：重放与查询用原执行身份，不复制 Agent 会话，不因救援重开输入或续期任何窗口（继承启动决策 D1/D3）。
- 对账查询本身失败（Controller 不可达、租约失效）按暂时失败处理，下个周期重试；不把查询失败记为任何证据。

为什么证据收集放自动层而强停不放：证据只缩短未知（把“不知道”变成“知道已终结”），不产生新的不可逆动作；强停会放弃未确认成果并终止可能仍在执行的进程，其代价必须由人显式选择（D3）。

### D3：受控回收是部署运维的显式命令，确认语义同强停，终止与数据删除分离

- 发起者：部署运维，经 Cloud 的部署管理端点（不暴露给租户用户；租户管理员对隐藏运行 Workspace 也没有入口——公开 API 恒 404）。按租户与资源范围查询账本是同一端点的只读能力。
- 命令输入：目标（账本条目 / Run / Workspace 范围）、非空 `reason`、显式放弃确认（语义同强停的 `impactConfirmed=true`：确认终止未知执行并放弃未交付成果）。幂等键、同键冲突规则沿用强停意图的模式。
- 意图持久化：独立的残留回收意图（`registered → terminating → succeeded` 三态，响应丢失或不可达时保持未完成并继续对账，不因一过期限就成功——继承强停 D4 的偏离说明）；意图引用账本条目与目标资源身份，发起者、时间、reason 留审计。
- 两阶段分离：**终止**（默认，对每个目标沙盒取得 `terminated` 与 `lateEnsureFenced`）与**数据删除**（单独的显式命令，才排 `workspace_data_delete`）。只终止不删数据是合法终态：账本以 `terminated_data_retained` 闭环，Workspace 行与卷按运维决定保留；删除数据后以 `released` 闭环。
- 回收意图在 D2 的同一竞争边界取得单一责任后，账本为 `reclaiming`；已有自动清理责任时不并行取得终止/删除权。意图失败或不可达不回退状态，继续对账直到闭环。
- 受控回收不改变 Run 终态、不重开 admission、不重写 `deliveryState`；它只终止资源与闭环账本。目标涉及其他运行、主 Workspace 或历史 Revision 时按既有 Project stop/delete 的保护拒绝（继承启动决策 D5“涉及残留的 Project delete/stop 仍拒绝无证据操作”）。

为什么新意图而不复用租户强停表：发起者与授权边界不同（部署运维 vs 租户管理员），合并会让租户侧出现可枚举隐藏 Workspace 的旁路；而确认、幂等、逐目标确认与三态语义完全复用强停已评审的规则，不发明新状态机。

### D4：物理终止经当前 Controller lease 与既有 effects，完成条件是逐目标双确认

- Cloud 为回收意图计划**既有**的 `sandbox_terminate` / `workspace_data_delete` effect；Controller 以当前 lease epoch 与许可执行，Cloud 不可用或许可失效时不发起新动作（继承围栏交付 D3）。不新增 Cloud 直连 Substrate 的路径。
- 终止前 Controller 先向可达 Node 投递既有结束请求与控制关闭（尽力而为，不把其确认作为终止前置）；Node 收到则按会话收尾规则结束，收不到则容器级停止。
- 目标集合 = 账本 `responsibility` 中记录的全部沙盒计划 ID 与预分配身份，包括**从未执行过的 ensure 计划**：对每个 ID 逐一 `sandbox_terminate`。[Sandbox Server](../../cluster/sandbox-server/20260926-docker-sandbox-lifecycle-via-substrate-effects.md) D6 已保证目标从未 ensure 也写墓碑并报告 `{terminated: true, lateEnsureFenced: true}`；生产 Substrate 适配器必须保持同等语义（先墓碑后破坏、幂等、对不存在的目标也确认），这是本 ADR 对 Substrate 的接口要求。
- 迟到许可与在途 ensure 的 fencing 完全依赖既有机制：墓碑之后同 ID 的 ensure 以 `sandbox_terminated` 拒绝；回收意图不删除原 effect 行、不假报 sandbox 缺失、不创建新 generation 挂载同一可写卷（继承启动决策 D5）。
- **完成条件**：意图下每个已计划的稳定 sandbox ID 都实际取得 `terminated` 与 `lateEnsureFenced` 双确认，数据删除阶段另需每个卷的删除确认。全部确认后意图 `succeeded`、账本按处置方式 `cleared`；任何目标未确认则保持 `terminating` 并继续对账，或由运维显式进入 D5 的外部处置分支。
- **未计划与未到达分开判断**：只有 Cloud 在与创建计划、派发登记互斥的事务内，证明目标从未产生 sandbox/ensure 计划、许可或在途责任，并持久封闭继续产生这些责任的入口，才可记录“无目标待终止”。已有计划不能因尚未受理或请求可能丢失而移出目标集合。Substrate 的 effect 查询 `404` 只说明查询时本地 journal 没有该 intent，既不写墓碑，也不排除持有效许可的 ensure 随后到达；它不是上述 Cloud 证明，更不是终止双确认的替代品。

反例：Cloud 已持久计划 ensure A，Controller 的请求仍在网络中；此时查询 A 得到 `404`。回收仍须按 A 的稳定 sandbox ID 执行 terminate，取得双确认；不能先清账或删卷再等待 A。墓碑持久化后 A 即使迟到，也必须被拒绝。

- 回收完成与迟到事实并存时：迟到真实终止/清理证据缩小残留范围，迟到终态与交付按原身份接管，但 Run 终态与业务结算不再变化（继承启动决策 D4）。

### D5：重试、运维升级与 Substrate 永久不可用的兜底

- 自动层重试：对账查询按周期重试；已授权的 owned delete 由 D2 新增的持久清理责任驱动，复用既有 quiesce/删除步骤和幂等规则，不依赖只扫描 `releasing` 的业务重试循环。失败保留责任、证据和保护，无新增超时成功规则。
- 受控层重试：意图保持未完成即持续对账（`terminating` 含结果未知），Controller/Substrate 恢复后继续；不以固定超时把未确认判为成功。
- 升级路径固定为三级，逐级显式：

| 层级 | 动作 | 触发 |
|---|---|---|
| 自动对账 | 证据收集、成果救援、owned delete | 账本登记后持续 |
| 受控回收 | 运维显式确认后终止（+可选数据删除） | 证据长期收不齐或运维判断需要 |
| 外部确认 | 运维凭 Substrate 之外的事实关闭条目 | Substrate 永久不可用（宿主退役、平台故障不可恢复） |

- 外部确认兜底：运维对无法经 Substrate 取证的目标提供外部事实（如宿主已销毁的处置记录），非空确认说明随意图留存；账本以 `externally_confirmed` 闭环。这是运维责任的显式转移与审计记录，不是系统证据——确认文本不写成 `terminated`，效果上不伪造围栏。外部确认要求先存在未完成的回收意图，不允许无意图直接销账。
- 升级不自动发生：自动层不发起受控回收、不申请外部确认；“证据长期收不齐”的可见信号是账本查询结果本身（`reconciling` 且无新证据），告警与通知机制不在本文范围。

### D6：存量 D8 删除残留按原身份接入，不回填不改写

- 存量特征（根 D8）：Run 已会话结算、`delete_workspace` 以 `node_unavailable` 终结失败、Workspace 行与运行绑定保留、无 `agent_run_residuals` 行。
- 接入规则：受控回收与外部确认以**资源身份**（Workspace、失败的 delete operation、沙盒计划）为目标，同样适用于存量 D8 残留，不要求先回填账本表——`reason` 的 CHECK 约束（`agent_start_failed` / `workspace_unavailable`）也不允许无据改写来源。回收意图单独记录“D8 删除残留”来源。
- 自动层不扫描或重启这些存量删除：根 D8 放弃后 Run 已为 `done`，既有重新声明循环仅扫描 `releasing`，因此 Node 恢复可达也不会触发既有重试。本 ADR 不新增存量 D8 的自动恢复扫描；存量只由 D3 新增的受控回收入口接管，证据仍未知时保留原资源，不把恢复连接等同于清理授权。
- 接管与处置：受控命令核验原租户、Run/Workspace、失败 operation 与全部计划目标，在 D2 同等的互斥边界取得单一持久回收责任；无账本行时由该意图承载保护与证据，不伪造 D4/D5 账本。终止仍逐目标双确认，数据删除仍需单独显式命令，不能将历史 `done` 改回 `releasing` 来借用普通扫描。
- 处置完成后的记录：由新的回收意图及其处置记录引用原失败 delete operation，保存真实新证据；不把原 operation 的 `failed/node_unavailable` 改为成功，不改 Run 的原 `status`、`done` 或业务结算来源。外部确认同样先有意图并保留 D5 的审计边界。

## 不变量

1. 业务终态（`done`/`failed`）与时间流逝都不触发物理终止；终止只来自证据齐的 owned delete 或显式确认的受控回收。
2. 账本 `cleared` 需要每个目标的闭环证据或显式处置记录；`cleared` 不是业务终态的重复，`reconciling` 不是可丢弃状态。
3. 受控回收的确认语义是运维显式放弃未确认成果；系统任何一侧都不伪造 `impactConfirmed`、`terminated` 或 `lateEnsureFenced`。
4. 回收不删除原 effect 行、不重开 admission、不改 Run 终态与 `deliveryState`；迟到事实只能补证据。
5. 终止与数据删除分离；自动清理须有证据齐全后的系统 owned delete 意图，受控终止后的删卷须有独立显式命令，终止成功本身不授予删卷权。
6. 未完成的回收意图不因超时转为成功；外部确认必须先有意图且留审计。
7. 存量 D8 残留按原稳定身份接入，不改写历史结算与状态。
8. 已计划的 sandbox 逐目标取得终止与围栏双确认；Substrate 查询 `404`、Node 不可达和许可过期都不能代替该证据。
9. 同一残留只有一个持久清理责任；完成前不提前清账，清理授权不会解除其他调用方的残留保护。
10. `cleared` 后同身份真实事实仍先持久接管再 ACK；重放不复活资源或业务，冲突不得静默丢弃。

## 为什么不是这些替代方案

| 方案 | 取舍 |
|---|---|
| 业务 `done` 后自动强停残留沙盒 | 收尾即时；但放弃未确认成果与终止未知执行是人的决定，Substrate 不可用时仍会永久悬挂（启动决策已否决该方向） |
| 残留登记后自动排普通 delete | 走既有路径看似省事；quiesce 对未知执行必然失败，只是把创建阻塞换成删除阻塞（启动决策 D5 已否决同构方案） |
| 先写 `cleared_at` 再复用普通删除入口 | 可绕过当前阻挡；但物理资源未清理就解除保护，并切断创建侧迟到事实接管，须改为限定原责任的系统授权（D2） |
| 复用租户管理员强停 API 清理残留 | 少一套意图；但授权边界错位——隐藏 Workspace 对租户侧 404，开放枚举与操作即制造旁路，且租户管理员不应为部署级故障买单（D3） |
| Cloud 直连 Substrate 执行终止 | 少一跳；绕过 Controller lease 与既有 effect 的幂等、围栏与许可，生产 Substrate 适配器要另开一条路径（D4） |
| 对账层拿不到证据超时后自动升级强停 | 无需人工；把“放弃成果”变成默认行为，违背显式确认（不变量 1、3） |
| 删除失败即自动外部确认销账 | 清账快；把未知写成已处置，伪造证据（不变量 3） |
| 用 Substrate 查询 `404` 排除已计划的 ensure | 少一次 terminate；但查询不写墓碑，尚未到达的 ensure 仍可创建实例，不能解除删卷保护（D4） |
| 存量 D8 残留回填进账本表统一状态机 | 单表查询方便；`reason` 约束与来源语义不允许，强行改写历史结算违背 D8 继承（D6） |

## 本决策未解决的问题

- 部署管理端点的 API 形状、认证与部署运维的角色模型：实现承接 issue 决定，本文只固定授权边界与确认语义。
- 告警与通知（何时提醒运维存在长期 `reconciling` 条目）：不在本文；可见信号是账本查询。
- 自动对账周期、外部确认的最小证据要求等运维细则的默认值调优。
- 生产 Substrate 适配器对“terminate 从未 ensure 的目标也双确认”的验收方式：随实现承接 issue 的联合故障演练。
- 对象存储侧：迟到交付补登记后的 Revision 对象生命周期（含过期清理）沿用 Revision 既有规则，不由回收路径处理。

## 落地顺序

1. **账本状态机与自动对账层**（cloud）：`reclaim_state`/证据与清理责任迁移、周期扫描、新增系统清理授权、清理完成前后的事实接管与 owned delete 持久推进。完成条件：完整证据齐才授权；未清账即可执行获授权的删除且普通入口仍拒绝；授权/事实/完成竞争、清理重启、清除后重复事实和冲突事实均有直接验证，证据不足不排强停或数据删除。D2 的系统授权与清理前后事实接管是本文实现复杂度最高的部分，**竞争场景（授权与事实、自动与受控、完成与迟到事实）是这一阶段的验证重点**，不能只用顺序路径的用例代替。
2. **受控回收意图与管理端点**（cloud）：意图三态、两阶段命令、审计与按范围查询。完成条件：幂等/冲突、确认缺失拒绝、自动清理与运维命令竞争时只有单一责任、只终止不删数据的合法终态均有测试。
3. **Controller/Substrate 语义验证**（desktop、cluster）：既有 terminate 对从未 ensure 的计划 ID 双确认、迟到 ensure 被墓碑拒绝，作为回收完成条件成立。完成条件包括：已计划且持有效许可的 ensure 在途时，查询 `404` 不能清账或删卷；terminate 双确认后再送达该 ensure 必须被拒绝。Cloud 的“从未计划”证明与并发计划/派发只能有一方取得责任。
4. **联合故障演练**：Controller 失联恢复后意图继续、Substrate 不可达保持 `terminating`、外部确认留审计。存量 D8 需直接验证 Node 恢复不自动重新声明删除；受控入口按原身份取得责任后才能回收，无账本行也有保护；重复命令不产生第二份责任，原 failed operation 与 Run 终态始终不变。

保持不变：D4/D5 结算语义、根 D8 残留状态、公开 API 对运行 Workspace 的 404、`agent_run_residuals` 既有列义。从第 1 步起改变：账本条目有了清理状态、单一推进责任、系统授权和贯穿清理前后的事实接管；从第 2 步起改变：残留有了受控回收的显式入口。

## 核心测试用例

本 ADR 的核心测试用例见[残留资源回收用例](../../../test-cases/cloud/issue-run/residual-resource-reclamation.md)。实现尚未合入 cloud、desktop、cluster 的 main，义务都如实标为 `Missing`；每项落地步骤完成时按直接证据更新对应义务的证据状态。
