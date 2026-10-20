# Agent Run 残留资源的对账、受控回收与账本闭环

依据[已批准的残留资源回收决策](../../../decisions/cloud/issue-run/20261018-residual-agent-run-reclamation.md)。下文的 D 编号与不变量编号都指这篇 ADR；“启动决策”指[有界启动与未派发撤回决策](../../../decisions/cloud/issue-run/20261012-bounded-agent-startup-and-undispatched-withdrawal.md)，它的核心用例见[有界启动用例](bounded-agent-startup.md)。

本文覆盖 D4/D5 业务收尾之后残留责任的收尾：账本清理状态机、自动对账层的证据与系统清理授权、部署运维的受控回收意图、逐目标终止与 late ensure fencing 的完成条件、三级显式升级，以及存量 D8 删除残留的按身份接管。业务结算、启动与创建截止、迟到事实接管本身的义务仍由[有界启动用例](bounded-agent-startup.md)拥有，本文不重复登记。

回收实现尚未合入 cloud、desktop、cluster 的 main，下列义务都是 **Missing**。每条给出实现落地时必须产生的直接证据类型；落地运行按最终实现补上具体测试名与 commit，再按证据完整度改为 `Covered` 或 `Partial`。跨进程与跨仓库的义务（Controller 许可、Substrate 墓碑、联合故障演练）不能由 Cloud 侧单仓测试单独记为 `Covered`。

## Residual Ledger Reclaim State Advances Only With Per-Target Disposal Evidence

- 风险：账本被当作“业务终态之后的表单”提前清除，残留资源失去保护；或对账证据不落账，重启后无从判断谁还欠什么。
- 前置状态：`agent_run_residuals` 中有 D4 启动放弃与 D5 创建放弃两种来源的行，`cleared_at` 为空；另有一条只有部分沙盒取得终止证据的行。
- 触发：对账周期写入部分证据；全部目标闭环后写 `cleared_at`；推进者重启后再对同一行对账。
- 必须成立：每行有 `reclaim_state`（`reconciling → reclaiming → cleared`）与处置方式 `disposition`（`released` / `terminated_data_retained` / `externally_confirmed`）；证据逐条追加（类型、来源、时间），`reason` 与 `responsibility` 不被改写；`cleared` 只在全部目标取得闭环证据或显式处置记录时写入；重启后接管同一责任，不产生第二份推进责任。
- 禁止结果：证据不全即写 `cleared_at`；用业务 `done`/`failed` 或时间流逝代替处置证据；重复对账重建 Run、Workspace、ensure 或执行身份；改写 `reason` 与既有 `responsibility`。
- 验证义务与证据：**Missing** — 需直接数据库测试：部分证据后 `reclaim_state` 仍非 `cleared`；全部目标闭环后才写 `cleared_at` 与 `disposition`；重启接管后仍只有一份责任；证据追加不改 `reason`/`responsibility`。落地顺序第 1 步（cloud）。
- 决策依据：D1，不变量 2。

## Reclaim State Migration Marks Existing Rows Reconciling Without Backfilling Evidence

- 风险：升级时按历史推断出“已清理”，把仍有物理资源的条目销账；或迁移失败让既有保护消失。
- 前置状态：迁移前已有的 D4/D5 账本行，`cleared_at` 均为空，无任何证据列。
- 触发：新库迁移、升级迁移、重复执行迁移。
- 必须成立：存量行统一置 `reconciling`，不回溯判定任何证据；`cleared_at`、`reason`、`responsibility` 保持原值；`agent_run_residuals` 只增列，既有查询（`cleared_at IS NULL` 的保护条件）行为不变；重复迁移不重置状态。
- 禁止结果：迁移推断出处置证据或写 `cleared_at`；改 `reason` CHECK 之外的来源；升级后残留保护失效。
- 验证义务与证据：**Missing** — 需迁移测试：新库、升级、重复迁移三种路径下存量行状态与既有列不变；升级后普通删除入口与创建侧接管仍按残留拒绝。落地顺序第 1 步（cloud）。
- 决策依据：D1（存量行迁移），兼容性边界。

## Automatic Reconciliation Collects Evidence Only And Never Terminates

- 风险：自动层把“证据不足”解释成“已放弃”，对仍在执行的未知目标发起终止或数据删除；或把查询失败记成证据。
- 前置状态：`reconciling` 行，其中一条执行状态未知、Controller 暂时不可达，另一条已有迟到终态但交付未确认，第三条全部证据齐全。
- 触发：对账周期扫描；Controller 不可达；查询返回空结果；交付验证未完成。
- 必须成立：自动层只收集证据、做成果救援（按原执行身份重放）与已授权范围内的 owned delete；证据齐才申请系统清理授权；Controller 不可达或租约失效按暂时失败处理，下周期重试，不记为任何证据；“没有结果记录”不代替完整证明。
- 禁止结果：自动层排 `sandbox_terminate` 或针对未知执行的数据删除；把 `done`、超时或查询失败当作放弃证据；因救援重开会话、输入或续期窗口。
- 验证义务与证据：**Missing** — 需数据库与服务测试：证据不足的行不产生任何终止/删除副作用；不可达时无证据写入且下周期重试；证据齐的行才进入授权流程。落地顺序第 1 步（cloud）。
- 决策依据：D2，不变量 1、3。

## System Cleanup Authorization Keeps Residual Protection Until Every Target Closes

- 风险：为执行 owned delete 而提前清账，解除保护并拒收创建侧迟到事实；或自动层与运维命令同时取得删除/终止权。
- 前置状态：证据齐全的 `reconciling` 行，其 Workspace 的普通 `deleteRunWorkspace` 入口当前以 `resource_in_use` 拒绝；另有并发到达的受控回收命令与创建侧迟到事实。
- 触发：自动层申请系统清理授权；运维受控回收命令同时到达；清理进行中普通入口、Project stop/delete 与创建侧事实入口再次被调用；清理事务失败。
- 必须成立：新增的 Cloud 系统清理入口只在同一互斥事务内重读原资源关联与完整证据、固定全部待处置目标并封闭后续新派发后授予清理权；授权只针对这次残留与原资源，不授予新会话、ensure 或换代；取得单一持久责任后置 `reclaiming`，`cleared_at` 保持空值；`reclaiming` 期间只有绑定该责任的系统动作越过残留阻挡，公开 API、普通删除及涉及残留的 Project stop/delete 继续受保护；事务失败全部回滚，不留半份责任；执行仍逐步核验 quiesce、终止与围栏、数据删除证据。
- 禁止结果：先写 `cleared_at` 再调用普通删除入口；清理未完成时普通入口或 Project 操作取得旁路；自动层与受控命令并行计划副作用，或自动层自行升级为强停；授权授予新会话、ensure 或换代权。
- 验证义务与证据：**Missing** — 需直接事务与并发测试：未清账即可执行获授权删除且普通入口仍拒绝；自动与运维命令竞争只有一方取得责任、另一模式不能并行；清理事务失败后状态与证据完全回滚。落地顺序第 1 步（cloud）。
- 决策依据：D2（新增授权与保护转换），不变量 5、9。

## Late Facts Are Taken Over Across Reclaiming And Cleared Without Resurrecting Resources

- 风险：账本清除后拒收同身份真实事实，或迟到事实被用来复活已回收的资源与业务。
- 前置状态：一条已进入 `reclaiming` 的残留，另一条已 `cleared`；原 effect/execution 身份与租户、资源仍可核验；迟到事实与清理完成事务并发到达。
- 触发：Controller 以原身份提交迟到终态、交付结果或 effect 事实；同一事实重复提交；清理完成先提交、迟到事实后提交（以及相反顺序）。
- 必须成立：创建侧事实入口与原执行接管覆盖 `reconciling`、`reclaiming` 与 `cleared`；仍校验原租户、资源、effect/execution 身份及当前服务资格；先持久接管再 ACK，相同事实幂等重放只追加证据；不重开 admission、不再次结算、不重建已删除资源；后提交者看到先提交者的证据与责任，新发现的未闭环目标阻止清除。
- 禁止结果：因账本已清除或 Run `done` 拒收事实；迟到事实把 Run 改回运行态或复活 Workspace/ensure；事实与处置结论冲突时静默丢弃事实或伪造新的完成证明。
- 验证义务与证据：**Missing** — 需 gRPC 与数据库测试：清理中与清除后两条路径都能接管迟到事实且 Run 终态/版本不变；重复事实幂等；冲突事实保留原结论与审计并阻断后续处置。落地顺序第 1 步（cloud）。
- 决策依据：D2（事实接管贯穿清理前后），不变量 10。

## Controlled Reclamation Is An Explicit Deployment Intent With Confirmation And Audit

- 风险：受控回收变成租户侧可枚举隐藏 Workspace 的旁路，或终止成功即被当作删卷授权。
- 前置状态：证据长期收不齐的残留（含无账本行的存量 D8 残留）；隐藏运行 Workspace 对公开 API 恒 404；同租户另有运行中的 Workspace。
- 触发：部署运维经管理端点发起回收（带/不带显式放弃确认、重复幂等键、同键不同输入）；只终止不删数据；随后单独发起数据删除；租户管理员经公开 API 尝试同样操作。
- 必须成立：受控回收只经部署管理端点发起，输入含目标范围、非空 `reason` 与语义同强停 `impactConfirmed=true` 的显式放弃确认；意图独立持久化为 `registered → terminating → succeeded` 并留发起者、时间、reason 审计；确认语义同强停，终止与数据删除分两个命令；只终止不删数据是合法终态（`terminated_data_retained`），删卷必须另有显式命令（`released`）；同幂等键重放返回原意图，同键不同输入拒绝；租户侧与公开 API 仍 404，不获得入口。
- 禁止结果：租户管理员或普通成员发起受控回收；缺确认即终止；终止成功隐式排 `workspace_data_delete`；重复命令产生第二份责任；改写 Run 终态、重开 admission 或重写 `deliveryState`。
- 验证义务与证据：**Missing** — 需 HTTP/管理端点与数据库测试：确认缺失拒绝、幂等与同键冲突、两阶段分离、只终止的合法终态、租户侧无入口。落地顺序第 2 步（cloud）。
- 决策依据：D3，不变量 3、5。

## Termination Completes Only With Per-Target Terminated And Late Ensure Fenced Evidence

- 风险：用 Substrate 查询 `404` 或 Node 不可达代替终止证据，让仍在网络中的 ensure 之后创建出可写实例。
- 前置状态：账本 `responsibility` 记录多个沙盒计划 ID，其中 A 已 ensure 成功、B 已持久计划但请求仍在网络中（查询得到 `404`）、C 从未 ensure；B 的 ensure 持有效许可。
- 触发：回收意图经当前 Controller lease 计划 `sandbox_terminate`；对 A/B/C 逐一终止；terminate 双确认后再送达 B 的迟到 ensure；Controller 许可失效或 Substrate 不可达。
- 必须成立：Cloud 只为既有 `sandbox_terminate` / `workspace_data_delete` 计划 effect，不新增直连 Substrate 路径；Controller 以当前 lease epoch 与许可执行，许可失效时不发起新动作；目标集合包含从未执行过的 ensure 计划，逐 ID 取得 `terminated` 与 `lateEnsureFenced` 双确认；生产 Substrate 适配器保持 Sandbox Server 已有语义（先墓碑后破坏、幂等、对不存在的目标也确认）；只有 Cloud 在与创建计划/派发互斥的事务中证明目标从未形成计划、许可或在途责任并封闭后续入口，才可记录“无目标待终止”；`404` 既不写墓碑也不排除在途 ensure。
- 禁止结果：以 effect 查询 `404`、Node 不可达或许可过期作为终止或围栏证据；跳过未到达的 ensure 计划 ID；删卷先于全部终止确认；删除原 effect 行、假报 sandbox 缺失或新建 generation 挂载同一可写卷；墓碑后迟到 ensure 仍创建出实例。
- 验证义务与证据：**Missing** — 需三仓联合验证：Sandbox Server 侧已有 `sandbox-server/internal/lifecycle/lifecycle_test.go` 的 `TestTerminateOfUnknownSandboxSucceedsAndTombstones`、`TestTerminateRemovesContainerAndBlocksLateEnsure` 作为既有语义基础，但本 ADR 要求的“已计划持有效许可的 ensure 在途时查询 `404` 不能清账或删卷”“terminate 双确认后送达该 ensure 被拒绝”“Cloud 从未计划证明与并发计划/派发只有一方取得责任”都要新的直接证据（cloud 计划/派发事务测试 + desktop Controller 许可与查询测试 + cluster 墓碑与迟到 ensure 集成测试）。落地顺序第 3 步（desktop、cluster）。
- 决策依据：D4，不变量 8。

## Reclamation Intents Retry And Escalate Only On Explicit Operator Decision

- 风险：未完成的意图因超时被判成功，或把未知写成已处置，账本带着虚假证据闭环。
- 前置状态：`terminating` 状态的意图，Controller 失联、Substrate 长期不可达；另一条账本长期停在 `reconciling` 且无新证据。
- 触发：Controller 恢复；Substrate 持续不可达；运维发起外部确认（有意图与无意图两种）。
- 必须成立：自动层按周期重试，已授权的 owned delete 由持久清理责任驱动，复用既有 quiesce/删除步骤与幂等规则，不依赖只扫描 `releasing` 的业务重试循环；受控意图保持未完成即持续对账，恢复后继续；任何超时都不把未确认判为成功；升级只能是三级显式路径（自动对账 → 受控回收 → 外部确认），自动层不发起受控回收也不申请外部确认；外部确认必须先有未完成的回收意图，非空确认说明留存审计，账本以 `externally_confirmed` 闭环。
- 禁止结果：超时自动成功或自动升级；无意图直接销账；外部确认文本写成 `terminated` 或伪造 `lateEnsureFenced`；告警缺失被当作已完成。
- 验证义务与证据：**Missing** — 需故障与数据库测试：Controller 失联恢复后意图继续、Substrate 不可达时保持 `terminating` 且无成功写入、无意图的外部确认被拒绝、有意图的外部确认留审计。落地顺序第 1、2、4 步（cloud 为主，联合演练在 desktop、cluster 环境）。
- 决策依据：D5，不变量 6。

## Legacy D8 Residuals Are Taken Over By Identity Without Rewriting History

- 风险：Node 恢复后既有重试自动重新声明删除，或为接管而回填账本、改写历史失败结果。
- 前置状态：存量 D8 残留：Run 已 `done`、`delete_workspace` 以 `node_unavailable` 终结失败、Workspace 行与运行绑定保留、无 `agent_run_residuals` 行；对应 Node 随后恢复可达。
- 触发：Node 恢复；释放扫描周期运行；部署运维按原资源身份发起受控回收；回收完成或外部确认。
- 必须成立：Node 恢复不会让 `done` Run 重新进入只扫描 `releasing` 的既有循环，本 ADR 不新增存量自动扫描；接管以资源身份（Workspace、失败的 delete operation、沙盒计划）为目标，不要求也不伪造账本行；无账本行时由回收意图承载保护与证据；终止仍逐目标双确认，数据删除仍需单独显式命令；完成后由新意图及处置记录引用原失败 operation，原 operation 的 `failed/node_unavailable` 与 Run 的 `status`/`done`/结算来源都不变。
- 禁止结果：回填账本行或改写 `reason` 来源；把历史 `done` Run 改回 `releasing` 借用普通扫描；把原 operation 改成成功；Node 恢复连接被当作清理授权。
- 验证义务与证据：**Missing** — 需直接验证：Node 恢复后不自动重新声明删除（可基于既有 `integration/agent_run_release_giveup_test.go` 的 `TestG032_7_ADoneRunIsNeverRedeclaredOrRewritten` 扩展）；受控入口按原身份取得责任后才回收、无账本行也有保护、重复命令不产生第二份责任、原 failed operation 与 Run 终态不变。落地顺序第 2、4 步（cloud）。
- 决策依据：D6，不变量 7。

## 跨仓库验证边界

- 第 1、2 步的账本、授权、意图与事实接管义务在 cloud 内可直接验证；它们的 `Covered` 判定不依赖 desktop/cluster。
- 第 3 步的终止完成条件跨 cloud、desktop、cluster：Cloud 的计划与“从未计划”证明、Controller 的许可与查询、Sandbox Server 的墓碑与迟到 ensure 拒绝必须同时取证，任何一侧缺失即 `Partial`。
- 第 4 步的联合故障演练（Controller 失联恢复、Substrate 不可达、外部确认审计、存量 D8 接管）只能在真实部署环境取证，单机指定场景也不等于完整故障矩阵，如实保留缺口。
