---
status: superseded
---

# Effect 系统基础：以 Generic Target 组织声明式收敛

> 已被取代：本决策由 [Skill 按存储路径以会话清单交给 Agent，取代 Effect 物化](../agent/session/skill/0-skill-catalog-from-storage-paths.md) 取代，原因：Effect 只有 Skill 一个消费者，Skill 改为按存储路径交给 Agent 后，Ora 不再写 Workspace，没有需要收敛的外部状态，desktop 整体退役 Effect。当前系统承诺以取代者为准；取代者的实现合并之前，desktop 里仍在运行的 Effect 以本文为历史依据。Effect 的核心用例暂时保留，在取代者进入 `implemented` 时随 Effect 代码及其中的引用一并删除。

Ora 采用 Workspace-scoped、level-triggered 的 Effect 系统，把完整 Desired State 安全收敛到
外部环境，并以可持久化证据证明 Consumer 已经能够消费结果。系统以 Generic Target 作为调度
与 readiness 边界，以 Resource 作为观察、加锁、修改、ownership 和恢复边界；Consumer、
Target 与 Resource 始终是三个正交概念。

该决策已经由 `ora-effect`、SQLite Effect repository、桌面端 Effect worker、应用层契约和
插件 Agent adapter 落地。本 ADR 记录稳定的架构边界、理由和不变量；具体字段、序列化形态与
调用签名以实现为准，不在此复制代码结构。

## 背景

Skill 安装等功能并非一次性文件复制。Ora 需要在 Desired 变化、Consumer 能力变化、Workspace
新增、外部状态漂移、进程重启和部分失败后继续收敛，同时不能覆盖用户拥有的内容，也不能让
两个 Target 并发写入同一个共享位置。

如果按 Agent、远程环境或遥测出口分别建立状态机，会重复 generation、重试、恢复和可观测性
语义；如果只按 Target 建模，又无法安全表示多个 Target 共享的外部资源；如果只按 Resource
建模，则无法证明依赖多个 Resource 的 Consumer 已整体就绪。因此本系统保留 Target 与 Resource
两层状态，并通过 projection、binding 和 receipt 连接它们。

## 决策

### Scope、Consumer、Target 与 Resource

`EffectScope` 是 Desired State、Target、Resource 和 ownership 的隔离根。首个且当前唯一的
Scope 形态是 Workspace；强类型 `EffectScopeId::Workspace` 保留了未来扩展空间，但 Core 不接受
任意字符串 Scope。当前 filesystem/Agent 垂直实现只为具有本地文件系统位置的 Workspace 建立
Target。Scope 进入 `Retiring` 后不再接受新的 Desired 或 Target，只允许完成未终结 Operation、
清理 Managed State 和释放 Resource。

`Consumer` 是消费 Effect 结果、并能按契约参与协调和 readiness 验证的稳定外部运行时。
Agent 插件是当前实现的 Consumer，但 Agent 身份和 Session 状态不进入 Effect Core。只观察
Effect 事件、日志或指标的订阅者不是 Consumer，也不创建 Target。

`EffectTarget` 表示一个 Consumer 在一个 Scope 内的完整收敛实例，是 request、claim、projection
和 readiness 的边界。同一个 `(Scope, Consumer)` 最多存在一个 active Target。Consumer 的完整
声明发生变化时，repository 创建新的 immutable Consumer Revision，令旧 Target 进入 retiring，
并以新 identity 建立使用新 Revision 的 Target；未变化的重复声明是 no-op，不触碰状态或 request。
Target declaration 是绑定精确 Consumer Revision 的完整、可替换 snapshot；其中每个 binding 同时
记录 Resource、可接受的 capability requirement 和 mutation 前的 coordination requirement，完整
声明以 digest 判定是否变化。旧 Target 在 retiring 期间保留原 bindings，以空 Desired contribution
参与共享 Resource 合并和清理；只有其空 projection 成功收敛后才删除 Target。失去最后一个 binding
且已无 ManagedItem 的 Resource 才能随之删除。

`EffectResource` 表示可独立定位、观察、加锁、修改、拥有和恢复的外部资源。一个 Target 可以
绑定多个 Resource，多个 Target 也可以共享同一个 Resource。同一 Scope 内，相同 adapter-normalized
physical key 只对应一个 active Resource；不兼容的 descriptor 或 materialization format 是声明
错误，不能通过创建第二个 writer 规避冲突。当前 filesystem 实现用规范化 Workspace-relative
locator 形成 physical key；同一 Workspace、相同 locator、adapter 和 format 的声明共享 Resource，
但各自 Target identity 与 contribution 仍保持独立。

Target 可以没有适用的 Desired Effect，也可以没有 Resource binding。空 projection 仍可被验证为
Current；任何持久外部 mutation 则必须归属于 Resource。

### 稳定身份、Revision 与 generation

所有跨重启、跨声明和跨持久化边界的身份均使用独立强类型，不能以显示名称、进程 ID、连接 ID、
路径或内容摘要代替。

`EffectSource` 表示稳定来源，publication 指向当前可供新 Desired State 选择的 immutable
`EffectRevision`。当前 Skill catalog 集成会把首次发布的 Source 加入现有 Scope，在新 Scope 中
seed 所有 active/published Source，并在重新发布时把引用该 Source 的 Desired 更新到新 Revision；
这些变更仍然按 Scope 各推进一次 generation。Source retirement 会删除其当前 Desired intent，
并把已有 Revision 标记为 unavailable，但保留 immutable Revision 行，使 Managed State、projection
和未终结 Operation 的历史引用仍然有效。Revision 的 definition、revision key 和 digest 发布后
不变；availability 是独立的运行时事实，可以在不改写 Revision 内容身份的前提下变化。

Skill catalog/package adapter 在调用 repository 前计算并验证 manifest digest 与完整 package
fingerprint；repository 只原子持久化已经验证的 publication，不通过 Resource Adapter 读取 source
目录。这样 catalog I/O 与 SQLite transaction seam 不会反向依赖物化实现。

`DesiredEffectIdentity` 在参数或 Revision 更新时保持稳定。`ManagedIdentity` 是 Ora 对外部 item
的 durable ownership 身份，不随 Revision、native name、路径或 fingerprint 改变。`Digest`
用于证明不可变输入或规范化 projection 的内容身份，`Fingerprint` 用于证明观察到的外部状态；
二者都不授予 ownership。

`Generation` 是一个 Scope 的整份 Desired State 的单调版本，而不是某个 Effect kind、Target 或
Resource 的局部版本。完整替换使用 expected-generation compare-and-swap；规范化后的集合没有
语义变化时为 no-op。Desired 的增删或语义变化在一个事务中只推进一次 generation，并原子唤醒
所有 active Target。观察、重试、Condition 和运行状态变化不推进 generation。

### Effect kind、能力与声明

`EffectKind` 是稳定、带命名空间的种类身份。kind-specific definition 和 parameters 使用封闭、
验证过的强类型分支；Core 不依据任意 JSON 执行业务逻辑。当前实现的第一个 kind 是 Skill，
其 parameters 暂无额外输入。

Consumer Revision 是一个稳定 Consumer 的完整 immutable 能力与声明快照。能力集合分别表达
Effect protocol、materialization contract、coordination contract 和 readiness contract。
`TargetSelector` 通过所需 protocol/materialization capability、显式 inclusion 和 exclusion
决定 Desired Effect 投影到哪些 Consumer；被选中只表示适用，不表示已经接受、应用或 ready。

Consumer declaration 是 Scope-independent 的完整快照。当前插件契约通过 `effectResources`
声明 Workspace-relative filesystem Resource template、materialization format、capability requirement
以及 `uninterrupted` 或 `quiesce_before_mutation`。插件 ID 被 host 映射为稳定 Consumer identity；
插件不能选择数据库 Target、Resource 或 Scope identity，也不能提供绝对 Workspace 路径。

声明配对是双向、level-triggered 的：新 Consumer 注册时立即与已有本地 Workspace 配对；worker
每轮又把当前声明快照与后来创建的 Workspace 配对。这样 correctness 不依赖一次性的注册事件。

### Projection 与共享贡献

一个静态分派的 `EffectPlanner` 同时负责完整 Target projection 与 Resource merge planning，避免调用方
把一个 kind 的 Target planner 和另一个 materialization planner 错配。它根据完整 Desired State、精确
Consumer Revision、Target declaration、Resource 集合和 immutable Effect Revisions，生成确定性的
`TargetProjection`；再把绑定到同一 Resource 的所有 active 或 retiring Target contribution 合并成
唯一 `ResourceProjection`。Projection 记录 generation、Consumer Revision、适用 Desired Effects、
每个 Resource 的 requirement 和内容 digest；它不是 Operation plan，也不授权 mutation。

materialization contract 从 Consumer Resource template 原样进入 Target binding 和
`ResourceRequirement`。Resource merge 会检测 native identity、format、ownership 和 contributor
contract 冲突；不同 contract 不会被某个 planner 默认值覆盖，而会形成 blocking Condition。任何
Target 都不能把自己的 requirement 当成共享 Resource 的最终状态，也不能删除其他 Target 仍然需要的 item。

缺失 capability、unsupported effect、不可用 Revision、无效声明或共享贡献冲突不会被静默过滤。
Planner 返回结构化 blocking Condition，repository 在持有 Target claim 的事务中保存该事实。
Planner 保持确定性和无副作用；Consumer/Resource Adapter 承担外部协议，Reconciler 负责 durable
状态机、fencing、执行顺序和原子提交。

### Desired、Managed、Observed 与 Preserved State

系统严格区分四类状态：

- Desired State 是 Scope 当前完整、规范化的意图集合，不证明外部结果存在。
- Managed State 由 `ManagedItem` ledger 表示，是 Ora 修改或删除外部 item 的 durable authority。
- Observed State 是 Resource Adapter 对外部事实的一次完整快照，只参与规划，不授予 ownership。
- Preserved State 是观察到但没有与 ledger 和 adapter ownership evidence 精确匹配的外部 item；Ora
  不得修改或删除它。

即使名称、路径、Source、内容或 fingerprint 与 Desired 相同，Observed item 也不能被自动接管。
如果 Preserved item 与 projection 冲突且目标格式无法共存，reconcile 必须阻塞。只有精确
ManagedItem，或已持久化 Operation/Artifact 所携带的有限 authority，才能授权 mutation 或清理。

当前 filesystem adapter 将 Skill 物化为 Workspace 下的目录，并使用 `.ora-managed.json` 作为
adapter ownership evidence。数据库 ledger、marker identity、native identity 和上次应用的
fingerprint 必须精确对应；路径通过共享的 portable path 能力验证，Workspace root、逃逸路径和
不安全链接不会进入 mutation。缺失的 managed 目录可以重建，unowned、marker-mismatched 或
drifted 目录不会被静默覆盖或删除。

### 状态水位与 Condition

Target 保存 desired、observed、applied、ready 四个 generation 水位，并始终满足
`ready <= applied <= observed <= desired`。observed 表示 planner 已对完整 generation 和精确
Consumer Revision 完成评估；applied 表示 Target 依赖的全部 Resource projection 已验证；ready
只在 Consumer Adapter 返回与 Target、generation、Consumer Revision 和 projection digest 全部
匹配的 `ReadinessReceipt` 后推进。

Resource 独立保存 desired、observed、applied 水位，并满足 `applied <= observed <= desired`。
Resource 没有 ready，因为物化正确不能证明 Consumer 已能消费完整 Target projection。

Target phase 只表达通用收敛位置：Pending、Planning、Coordinating、Applying、Verifying、Activating、
Current、CurrentWithIssues、Retiring 或 RecoveryRequired。Resource phase 只表达 Pending、
Reconciling、Current、Retiring 或 RecoveryRequired。Agent idle、进程重启、插件断连等原因通过
Condition 或 adapter receipt 表达，不扩展 Core phase。

`EffectCondition` 是当前、结构化的控制事实。它具有 Target 或 Resource owner、强类型 subject、
稳定 code、blocking impact、retry classification、可选 generation 和安全详情。控制流依赖 code、
impact 与 retry，而不是解析 message。Blocking Condition 阻止相应水位；NonBlocking Condition
允许 Target 成为 CurrentWithIssues。安全详情不得包含 Secret、配置原文、敏感路径或任意 error chain。

### 调度、claim 与公平性

每个 Target 最多有一个 durable `ReconcileRequest`。连续 wakeup 合并最大的 requested generation；
wake reason 仅用于诊断，worker claim 后始终重新读取当前事实。Pending、到期 RetryScheduled 和
lease 已过期的 Claimed request 可再次调度；Blocked 只由匹配 change trigger 或显式用户动作恢复。
RecoveryRequired 使用 blocking/manual Condition，不参与 timer retry，也不能由普通 generation
wakeup 或 lease expiry 绕过。

桌面端 worker 在专用线程中运行，进程内 notify 只用于降低延迟；SQLite request 是 owed work 的
唯一事实来源，worker 还会周期扫描。每轮按 due time、request time 和 Target identity claim 有限
batch，因此通知丢失、进程重启或单个阻塞 Target 不会丢失或独占其他 obligation。

claim Target request 时，repository 递增保存在 Target 行上的 monotonic fence，并把 token、worker
和 lease 写入 request；删除或重建 request row 不会重置 fence。Target claim 只授权推进该 Target。
当前实现会按稳定 identity 顺序一次性取得 Target 所有 bound Resource 的 fenced claims；任一
Resource 不可用时不获得部分 authority。取得 claims 后必须再次加载完整 snapshot，并确认 binding
集合未变化，然后才允许 observation 支持 ResourceStatus 或 readiness。Resource 自身也保存单调
fence，释放 claim row 不会重置 fencing token。外部插件 IPC 或 filesystem 调用期间不持有数据库
事务。

### 协调、应用与 readiness

如果 Resource plan 不需要 mutation，reconciler 在精确 observation 后推进 Resource 和 Target
applied，调用 Consumer readiness 验证，并在一个事务中提交 projections、ledger、statuses、
Conditions、receipt 和 request；此路径不创建 Attempt 或 Operation journal。

如果需要 mutation，reconciler 合并所有受影响 binding 的 coordination requirement。标记为
`Uninterrupted` 的参与者仍保留在计划中但无需 IPC；要求 quiescence 的每个 Target 都必须返回与
Target、contract 和 `SafeToMutate` 状态精确匹配的 receipt。全部必要参与者完成协调后才能修改
Resource。验证完成后，相同参与者必须返回 `Reactivated` receipt。协调 receipt 只证明本次
mutation barrier，不自动推进任一 Target 的整体 readiness。

当前 Agent Consumer Adapter 通过 `effect/coordinate`、`effect/reactivate` 和
`effect/verifyReady` 实现上述契约。coordinate 与 reactivate 请求携带精确 Target identity 和本次
coordination plan 的完整 Resource identity 集合；verifyReady 请求携带 Target、generation、
Consumer Revision 和 projection digest。所有 payload 都是 Consumer Adapter 的版本化契约，不能
扩展 Effect Core phase。断连的 Consumer 已经不能消费旧状态，因此 adapter 可记录安全 receipt，
而不会为了协调主动启动插件；它在下次启动时读取已收敛 Resource。若 reactivation 替换 Agent
实例，adapter 会使现有 ACP Session 失效，由既有 session reload 路径在下一次使用前恢复。

### Journal、Artifact 与崩溃恢复

有 mutation 时，`ReconcileAttempt` 固化 Target、generation、Consumer Revision、Target/Resource
projection digests、完整 coordination plan 和有序 Operation ids。每个 `EffectOperation` 固化
Resource、sequence、mutation、exact previous/planned state 和版本化 adapter payload。Attempt、
projections、Prepared Operations 与 Artifacts 必须在任何外部副作用前持久化；更高 generation
只能保留为新 request，不能修改已 Prepared 的输入。

每次 coordinate、apply、verify 和 reactivate 外部调用之间都会持久化已经取得的 receipts、Operation
progress 与 Attempt phase。成功路径在单个事务中 finalize Operations 和 Attempt，更新完整 ownership
ledger、statuses、readiness、Conditions，并完成或保留 request。只有
`requested_generation <= ready_generation` 时 request 才会删除；期间到来的更高 generation 会回到
Pending。

`OperationArtifact` 是 Operation 为 staging、backup 或 compensation 持有的临时外部资源。它不是
ManagedItem，只有精确 Operation、versioned locator 与 expected fingerprint 同时匹配时才能使用或
清理。Operation Finalized 表示业务状态已提交，不表示 Artifact 已清理；清理失败会保留 durable
CleanupFailed 事实，不能被误报为已清理。

失败发生在 journal 创建前时，worker 将 request 放入带计数和 not-before 的 retry schedule；新的
Desired 或 declaration wakeup 会重置旧 retry sequence。Prepared 或 Applied journal 变得不明确时，
普通 reconcile 不根据路径、名称或相似内容猜测结果，也不会创建第二次 mutation。startup 和每次
普通 claim 前的 recovery scan 会把过期且未终结的 Operation、Attempt、TargetStatus 与
ResourceStatus 原子隔离为 RecoveryRequired，保留 Artifacts，写入 blocking/manual Conditions，
并令 claim 查询排除相关 Target。仍持有有效 worker lease 的 Operation 不会被误隔离；恢复只能通过
能够验证 exact adapter state 的显式受控命令完成：`recoverEffectTarget` 对每个被隔离的 Operation
用 Resource adapter 证明其精确状态——证明达到 exact planned 状态则终结 journal 并保留 ownership；
证明仍处 exact expected 基线则以 `FinalizedWithoutEffect` 关闭且不授予任何 ownership；两者都不
匹配则拒绝恢复并保留隔离。恢复不重放 mutation、不猜测、不伪造历史；关闭最后一个 journal 后，
request 回到调度，manual Conditions 清除，普通 reconcile 按当前 Desired State 重新收敛，
retiring Target 额外推进 Scope epoch 使空投影落到新的不可变 identity。

证明前必须无创建副作用地重新校验 journal 固化的 Workspace / Resource 路径边界，解析后的
Resource 根目录必须与 journal 一致。文件、链接、不可访问路径或重定向不能冒充 Missing。
关闭事务把各 Operation 的原检测时间与关闭结果写入恢复 audit，再清空当前阶段的 `detected_at`；
字段仅属于 RecoveryRequired 的时间规则见[行审计时间与业务时间](20260905-separate-audit-and-business-time.md)。

Planner 产生 blocking Conditions 时，repository 会在同一事务中保存 Target/Resource statuses 与
Conditions，把 request 转为 Blocked，并释放本轮 Resource claims。这个提交只保留未来必须重新
评估的 obligation，不允许阻塞 Target 长期占有共享 Resource authority。

### 持久化与应用边界

`EffectRepository` 是静态分派的深接口。SQLite migration v0006 持久化 Scopes、Sources/Revisions、
Desired State、Consumers/Revisions、Targets、Resources/bindings、digest-addressed projections、
Managed ledger、独立 statuses、Conditions、requests/claims、Attempts、Operations/Artifacts、
coordination/readiness receipts 和 append-only audit events。repository 负责事务边界和跨 Scope、
claim、projection、ledger 的一致性校验；调用方不能逐字段写入状态或 ownership。

应用层公开完整 Workspace Effect State 的读取和 expected-generation replacement、Target status 与
当前 Conditions 的事务一致读取，以及显式 Target retry。retry 只唤醒 reconcile，不改变 Desired。
传输 DTO 使用封闭的 Effect parameters 和稳定 Consumer identity，并在边界处转换为领域类型。

Audit event 是可裁剪的业务历史，不参与 ownership、调度、恢复或当前状态推导，也不是 event-sourcing
log。删除历史 Audit 不得改变系统行为。

## 扩展边界

时间语义由后续决策
[Effect 的行审计时间由写事务拥有，业务时间独立保存](20260905-separate-audit-and-business-time.md)
细化：行审计不再由源数据版本或 worker snapshot 决定，恢复检测时间独立持久化；原有 generation、
fencing、ownership 和事务原子性约定保持不变。

Effect Core 通过泛型和 trait 进行静态分派，同时拥有内建实现持久化所需的封闭、强类型 wire model。
新增内建 Effect kind 会扩展相应 definition/parameters/payload enum，并在集成 crate 提供新的 planner
逻辑；这项显式改动优于用任意 JSON 隐藏未知业务分支。它不应复制 Target、Status、Request、
Condition、Attempt 和 Operation 状态机。Consumer-specific 和 Resource-specific 数据只允许出现在
版本化 definition、descriptor、materialization input、operation payload、locator 或 adapter receipt 中。

当前第一条完整垂直实现是：本地 Workspace Scope、Skill Effect kind、Agent 插件 Consumer、
filesystem directory Resource、SQLite repository 和后台 worker。Skill/filesystem 名称会出现在封闭
wire payload 中，但不能进入通用 phase、claim、watermark、fencing 或 recovery 状态机；Agent、
Session 和插件 transport 仍只存在于 adapter 实现。

## 必须保持的不变量

1. Scope 的 Desired、Target、Resource、ownership、Operation 和状态不得越过隔离边界。
2. 一个 `(Scope, Consumer)` 最多有一个 active Target；Consumer、Target 与 Resource identity 不可互换。
3. Target 与 Resource 是多对多关系；Target readiness 覆盖完整 projection，Resource mutation 使用独立 authority。
4. ResourceProjection 合并全部 active 或 retiring contributor；一个 Target 不能删除其他贡献者仍需要的 item。
5. Desired、Managed、Observed 与 Preserved State 含义独立；observation、digest 或 fingerprint 不证明 ownership。
6. 只有精确 Managed ledger 或 durable Operation/Artifact authority 允许更新、删除或清理外部状态。
7. Target claim 不授权 Resource mutation；Resource claims 必须 fenced、稳定排序并以 all-or-none 方式取得。
8. Resource claim 后必须重新加载并重新规划；claim 前的 snapshot 或 wake payload 不能成为 mutation 依据。
9. 共享 Resource mutation 前必须满足所有受影响 binding 的协调要求，协调 receipt 不等同于 readiness。
10. Target 与 Resource 水位只由对应的完整 projection、observation、verification 和 receipt 证据推进。
11. Prepared Attempt、Operation 和 Artifact authority 不因新 generation 改写；新工作只通过 request 合并。
12. Consumer Revision 变化重新投影相关 Target，不能沿用旧 capability 或 projection 的 readiness。
13. Blocking Condition 阻止对应水位；NonBlocking Condition 不能被报告为无问题的 Current。
14. 外部 mutation 必须可精确验证、可安全重试或进入 RecoveryRequired，不能猜测完成状态。
15. 外部调用期间不长期持有数据库事务；事务用于领取 authority、持久化证据和原子提交领域转换。
16. Consumer-specific 与 Resource-specific 词汇不得进入通用 phase、claim、watermark 和 recovery 状态机。

## 被否决的边界

- 不以 Agent、容器或遥测类型分别建立 Effect Core；它们是 Consumer Adapter 的实现。
- 不以 filesystem “surface” 或目录作为调度根；它们是 Resource Adapter 的具体 descriptor。
- 不把 Target 与 Resource 合并；前者无法独自保证共享写入，后者无法独自证明 Consumer 整体 ready。
- 不使用 edge-triggered 事件 payload 作为 correctness 来源；durable request 只表示仍欠一次重新评估。
- 不从 Observed State、路径或内容相似性推断 ownership，也不在恢复时推断未完成副作用的结果。
- 不使用通用 JSON 扩展 Core 业务分支；未知版本化契约必须被拒绝，而不是猜测执行。

## 结果

该设计引入了 Target 与 Resource 两层状态、独立 fencing 和 journal-first mutation，模型与持久化
成本高于一次性安装流程。但它让共享 Resource、跨重启恢复、Consumer 协调、完整 readiness 和
用户内容保护拥有明确且可验证的证据链。

Skill 与 Agent 只是首个实现。后续 Consumer 和 Resource 可以复用同一套 generation、调度、
ownership、恢复与可观测性语义，而其协议细节仍被限制在强类型 planner 和 adapter 接缝内。
