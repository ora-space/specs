# Cloud 核心测试用例

本目录与 `specs/decisions/cloud/` 完全镜像。Cloud 各领域的 ADR 进入 `approved` 后，才在对应叶子
目录登记需要长期维护的租户与 owner 隔离、认证授权、聚合约束、幂等、并发、恢复、协议和升级用例。

占位目录只用于稳定领域路径，不表示已有决策或测试证据。核心用例必须按业务风险组织，并使用
`Covered`、`Partial` 或 `Missing` 如实标记直接证据。

## 不可用 Node 延后与 clone 分轮等待

对应[Controller 决策](../../decisions/controller/node-management/20261028-defer-unavailable-node-operations.md)为
`implemented`；实现基线为 Cloud #206 合入版本 `1cfaed3` 与 Controller #750 合入版本 [`df694c8`](https://github.com/ora-space/desktop/commit/df694c8836fc70b1c789cc7e8e523d4d5b1b6c66)（最终 PR head `65fc457`）。
跨层义务保留在同一份 Controller 核心用例，不复制 Cloud 用例；直接测试与部署实际覆盖按项标为
`Covered`、`Partial`、`Missing`，实现状态不消除缺口。受控联合测试和部署/真实 30 分钟等待分别说明，
第 17 项的多项排队义务要求不同 Project 各自的 Workspace Node 同时失联，旧项持续循环时健康 B/C 真实排队并推进；
blocked/node_unavailable 无界 retrySeconds 保留为独立后续问题。

当前对应的 18 项证据汇总为 7 Covered / 11 Partial / 0 Missing；多失联项已包含各 Project 独立 Node 的真实队列时间线。

| 领域 | 核心用例 | 重点风险 |
| --- | --- | --- |
| operation / controller-integration（跨层引用） | [Cloud 持久退避与有限原因](../controller/node-management/20261028-defer-unavailable-node-operations.md#cloud-persists-backoff-counts-and-bounded-reasons) | 计数或 retry_at 重启丢失、部分提交、重放多计、原因混淆、迁移改写旧身份与误耗三次预算 |
| operation / controller-integration（跨层引用） | [总窗口](../controller/node-management/20261028-defer-unavailable-node-operations.md#the-total-result-window-uses-durable-registration-wall-clock-time)、[晚到成功](../controller/node-management/20261028-defer-unavailable-node-operations.md#a-late-known-success-automatically-recovers-a-timed-out-operation)、[晚到失败](../controller/node-management/20261028-defer-unavailable-node-operations.md#a-late-known-failure-follows-the-existing-clone-failed-budget)、[Node Unknown](../controller/node-management/20261028-defer-unavailable-node-operations.md#a-node-reported-unknown-remains-blocked) | 截止续期、超时覆盖已知结果、blocked 无人接管、失败预算重置、Unknown 被自动解除 |

## 已批准的成员加入来源与链接预检

| 领域 | 核心用例 | 重点风险 |
| --- | --- | --- |
| tenancy | [成员加入来源与链接预检](tenancy/member-entry-provenance-and-link-preview.md) | 匿名越权与写入、失效原因泄露、空操作消费、来源非原子或伪造、历史猜测回填、登录恢复和原文泄露 |

对应 ADR 为 `approved`；新增义务均为 `Missing`，既有邀请测试不自动覆盖新契约。

## 已批准的小队与定向加入

小队编组及 tenancy 后续 ADR 均为 approved，尚待实现；新增验证义务均为 Missing。

| 领域 | 用例 | 主要风险 |
| --- | --- | --- |
| squad | [编组与生命周期](squad/membership-and-lifecycle.md) | 跨租户编组、权限扩大、leader 失效、撤权/归档竞争、mock 与隐式执行 |
| tenancy | [定向加入与撤权](tenancy/targeted-joining-and-revocation.md) | 部分加入、空操作消费、来源混写、匿名泄露、失权复活、升级旧写入者 |

## IssueRun 残留恢复

| 领域 | 核心用例 | 重点风险 |
| --- | --- | --- |
| issue-run | [无删除操作的 D8 显式接管](issue-run/d8-residual-without-delete-operation.md) | 默认扩权、错绑与重复责任、缺失来源被伪造、升级回放失真、终止后误删数据、原任务恢复丢失成果 |
| issue-run | [交付结算与执行未受理证明](issue-run/delivery-execution-settlement-and-non-acceptance.md) | 截止伪造终态、受理与围栏竞态、错绑原实例、接管前解除责任、迟到事实冲突、存量与混合版本丢失保护 |

## 已批准的多人协作运行时控制

使用权限、独占准入、停止/强停、控制交付及插件维护五份 ADR 的后端规则为 `implemented`；仓库凭据连续性也已 `implemented`。三处 UI 与完整联合故障矩阵由[后续验收 ADR](../../decisions/cloud/execution-admission/20261013-runtime-control-ui-and-fault-acceptance.md)承接，各用例继续按直接证据保留 Partial/Missing。Node 插件执行已经落地，但不能据数据库或模拟器宣布完整产品与故障验收完成。

| 领域 | 核心用例 | 重点风险 |
| --- | --- | --- |
| workspace | [使用权限与历史创建者](workspace/runtime-workspace-use-permissions.md) | 内容越权、归属误判、升级保真、独立运行时隔离 |
| execution-admission | [独占与安全交接](execution-admission/exclusive-runtime-control.md) | 并发获取、会话内冲突、到期、撤权与收尾 |
| operation | [普通停止与强停](operation/safe-stop-and-administrative-force-stop.md) | 活动保护、目标范围、迟到创建、终止及持久数据 |
| controller-integration | [跨进程执行约束](controller-integration/fenced-runtime-control-delivery.md) | 服务身份、旧指令、接受结果未知、重启和协议兼容 |
| plugin-marketplace | [管理员选择与维护等待](plugin-marketplace/admin-selection-and-idle-maintenance.md) | 管理员权限、维护独占、持久 pending、期望变更 |
| project | [凭据连续性](project/repository-credential-continuity.md) | 成员离开、受控团队绑定、静态回退、历史引用 |
| execution-admission | [UI 与联合故障验收](execution-admission/runtime-control-ui-and-fault-acceptance.md) | 三处 UI、十一项独立故障义务与未开放入口归属 |

真实端到端证据采用 cluster Compose 或等价真实部署链路，遵守 minicloud 与 Controller 过渡监听退役。核心用例使用英文二级标题作为稳定代码引用，实施时按义务逐项补充直接测试和对应仓库提交。

## 工作流执行决策

[工作流执行根决策](../../decisions/cloud/workflow/0-cloud-side-scheduler-owns-workflow-execution.md)与它的后续决策
[执行器槽位决策](../../decisions/cloud/workflow/20261015-executor-slots-replace-agent-model-ids.md)、
[个人槽位映射与按槽位绑定决策](../../decisions/cloud/model-access/20261015-executor-slot-mappings-and-per-slot-run-bindings.md)、
[工作流会话结束决策](../../decisions/cloud/workflow/20261019-workflow-session-end-after-initial-turn.md)与
[插件资产导入决策](../../decisions/cloud/workflow/20261016-asset-import-and-published-snapshots.md)
的核心用例同批建立；根决策、会话结束与插件资产导入已 `implemented`，执行器槽位与个人槽位映射仍为 `approved`。
[节点重试与恢复契约](../../decisions/cloud/workflow/20261021-node-retry-and-resume-contract.md)
于 2026-10-21 经 owner 批准，核心用例同批建立，义务全部为 `Missing`。[人工节点运行级输入契约](../../decisions/cloud/workflow/20261021-human-node-run-input.md)
于 2026-10-22 经 owner 批准，核心用例同批建立，义务全部为 `Missing`（产品实现尚未开始，发布门禁继续
拒绝 `human` 与 `interactive: true`）。[loop 与 loopExit 执行决策](../../decisions/cloud/workflow/20261025-loop-and-loop-exit-execution.md)
于 2026-10-26 经 owner 批准，核心用例同批建立，义务全部为 `Missing`。[变量聚合执行决策](../../decisions/cloud/workflow/20261025-variable-aggregator-execution.md)
与[iteration 执行契约](../../decisions/cloud/workflow/20261025-iteration-foreach-execution.md)同日经 owner 批准，核心用例同批建立，义务全部为 `Missing`。[工作流节点会话读取决策](../../decisions/cloud/workflow/20261026-workflow-node-session-read.md)
于 2026-10-26 经 owner 批准；实现已由运行
[ora-space/circus#720](https://github.com/ora-space/circus/issues/720) 经 [cloud#195](https://github.com/ora-space/cloud/pull/195)（merge
`63034cf`）合入 main，ADR 与核心用例同步为 `implemented`，cloud 侧义务按合并后的真实 PostgreSQL 用例、路由契约测试与前端渲染测试
逐条登记 `Covered` / `Partial`；2026-10-28 在部署 `63034cf` 上完成的真实栈验收（#720）把直接证明的按尝试读取、
重试与 `output` / `sessionEndReason` 分离登记为证据，其余缺口写在对应行内。Cloud 侧调度器已随根决策落地：运行从 `pending` 进入真实执行，节点状态由
调度器与真实会话写入，模拟 trace 按根决策排除在验收之外；执行类义务的直接证据登记在各用例里，按 `Covered` / `Partial` 如实
标注；调度用例已补充直接接手、同 Workspace 双 agent、顺序重复、无结果失败及终态时序证据，部署级同步竞争、真正并发的重复提交与真实执行侧旧代回调仍为 `Partial`、真实迟到结果为 `Missing`；端到端补记为 `Missing`（控制面分支测试仅作相邻证据，固定候选没有符合约束的真实外部删除触发器），由[承接事项 #665](https://github.com/ora-space/circus/issues/665) 继续跟踪。
门禁类义务（图校验、启动门禁、槽位解析）的实现在
[cloud#65](https://github.com/ora-space/cloud/pull/65) 上，已随合并提交
[`df507f0`](https://github.com/ora-space/cloud/commit/df507f0) 落到 main，按逐条核对该提交的实现与测试后得到的证据
状态登记（`Covered` / `Partial`），不再按"未合并"整体降级。

[经 Node 承载的交互节点默认 `end_turn`](../../decisions/cloud/workflow/20261119-end-turn-default-until-host-completion-tool.md)
于 2026-11-19 经 owner 批准，D2 由 [cloud#403](https://github.com/ora-space/cloud/pull/403)（merge `2b80c023`）实现、
[cloud#417](https://github.com/ora-space/cloud/pull/417)（merge `43ad6eb2`）补齐提示文案与运行测试后改为 `implemented`；
12 项义务为 4 Covered / 3 Partial / 5 Missing：编辑器默认值、`agent` 按 `end_turn` 执行、自动接受不生效与暂不生效标记有直接证据，
保留已有值与跨端导入只有部分证据，不依赖探测以及恢复版本之后的行为没有直接证据。

desktop 参考实现（`8909f07`）的直接证据在每个用例里单独标注，并如实记下与它不同的地方：同一 run 内多个 agent 节点首期串行（desktop 在同一次调度 pass 里并行派发 sibling agent，所以"至多一个串行执行者"这一条在它身上是 `Partial`）、图在整个运行生命周期内不可变（desktop 另有 `snapshot_switch` 可替换冻结图）、发布时就拒绝不可执行节点（desktop 只在启动时拒绝）、启动表单用 `snapshotId`（desktop 无此表单）。这些义务没有 desktop 证据可依托，只能由 Cloud 自己的测试证明。

| 领域 | 核心用例 | 重点风险 |
| --- | --- | --- |
| workflow | [执行身份与调用绑定](workflow/execution-identity-and-invocation.md) | 图随活稿漂移、身份不全、IssueRun 与 WorkflowRun 的半连接 |
| workflow | [调度串行与恢复](workflow/scheduler-serialism-and-recovery.md) | 多副本重复派发、迟到回调、崩溃丢进度、谎报成功、没有取消入口 |
| workflow | [发布与启动门禁](workflow/publish-and-start-admission.md) | 冻结执行不了的图、UUID 绕过校验、槽位被顶替、快照字段错名 |
| workflow | [执行器槽位](workflow/executor-slots.md) | 槽位名变体、`hint` 被当兜底或被自动写入、旧图拒收或保留执行器 |
| workflow | [工作流会话结束与有界等待](workflow/session-end-after-initial-turn.md) | 成功回合无人结束会话、重复或跨节点结束请求、确认窗口无界、把未知当成功 |
| workflow | [Start 输入 kickoff 取值](workflow/start-input-kickoff.md) | 运行读到编辑器默认提示词而非本次 kickoff、unset 与空串混淆、非字符串 `input` 被放行、历史行被误读 |
| workflow | [节点重试与恢复](workflow/retry-and-resume.md) | 可重试失败被直接失败或误判成功、等待依赖进程内定时器、跨尝试证据串用、取消被等待拖延、resume 改写状态 |
| workflow | [人工节点的运行级输入与有界等待](workflow/hitl-run-input.md) | 等待无界、输入绕过校验、重复提交、无绑定运行越权、409/410 顺序、取消竞争、超时误判、活动被当回合 |
| workflow | [loop 轮次执行、早退与容量边界](workflow/loop-and-loop-exit.md) | 越界读取其它容器或旧轮、feedback 半组赋值、耗尽策略漂移、早退伪造成功、恢复重放成员、体积超限被截断成成功 |
| workflow | [变量聚合的发布门禁与有序透传](workflow/variable-aggregator.md) | 首个已赋值与空值混淆、多值顺序漂移、no-match 空成功、非前驱放行、跨容器与跨轮串值、非法配置冻结 |
| workflow | [Iteration 的 foreach 执行与按轮账本](workflow/iteration-foreach-execution.md) | 迭代源非前驱、轮内变量泄漏、收集陈旧值、策略丢账本、恢复重跑、取消后开新轮、体积截断、最后一轮重复展示 |
| workflow | [工作流节点会话读取](workflow/node-session-read.md) | 跨尝试串会话、游标跳过记录、旧尝试冒领新输出、终态回执按形状误判、sessionId 直读越权、输出与结束原因混用、`result` 为空误判在途、单页即宣称完整 |
| workflow | [完成工具送达前的默认完成方式](workflow/end-turn-default-before-completion-tool.md) | 默认值与实际执行不一致、显式选择被覆盖、静默降级、跨端导入改写、恢复时批量改写或能力不足时降级 |
| model-access | [个人槽位映射与按槽位绑定](model-access/executor-slot-mappings.md) | 映射越权、槽位被顶替、在途运行被改映射、会话借用别的槽位授权 |

## 已批准的定时任务决策

[Cloud Scheduled Task 根决策](../../decisions/cloud/scheduled-task/0-cloud-owned-scheduled-prompt-runs.md)已按最终实现同步为 `implemented`；核心用例见 [admission-and-session-bounds](scheduled-task/admission-and-session-bounds.md)。[Cloud Scheduled Run verdict 与产物 ADR](../../decisions/cloud/scheduled-task/20261022-run-verdict-and-artifacts.md)经 owner 批准为 `approved`，核心用例同批建立（[verdict-and-artifacts](scheduled-task/verdict-and-artifacts.md)）。根决策的存储与调度 worker、公开 API 与 Web 管理面已随 [cloud#111](https://github.com/ora-space/cloud/pull/111) 合入 main，worker/API 领域测试随后续 PR 合入；admission-and-session-bounds 按 cloud main [`88c665a`](https://github.com/ora-space/cloud/commit/88c665a61ab169240f42328c3a4f74491ab673c0) 的直接测试与 2026-10-26 的 cluster Compose 真实栈验收（真实 Controller / Node / Sandbox、官方 Claude 插件与真实模型，任务不声明产物）登记，18 项义务全部 `Covered`。verdict-and-artifacts 的证据随该 ADR 的实现另行登记，状态以该用例为准。[任务对话准入 ADR](../../decisions/cloud/scheduled-task/20261027-task-conversation-admission.md)已随 [cloud#193](https://github.com/ora-space/cloud/pull/193) 实现并同步为 `implemented`；[task-conversation-admission](scheduled-task/task-conversation-admission.md) 按 cloud main [`3bbaf94`](https://github.com/ora-space/cloud/commit/3bbaf940ff00b23cd3629bbdf3fb821b1d9bb1fe) 的直接测试登记（拒绝后对话不变、对话恢复后的下一次接纳与两条入口在同一状态上的联合验证随 [cloud#199](https://github.com/ora-space/cloud/pull/199) 补齐），8 项义务全部 `Covered`。

| 领域 | 核心用例 | 重点风险 |
| --- | --- | --- |
| scheduled-task | [接纳、会话边界与历史保留](scheduled-task/admission-and-session-bounds.md) | 重复接纳、停机补记与超时误判、无人值守会话空等、资格失效仍接纳、历史无界增长 |
| scheduled-task | [verdict 与产物](scheduled-task/verdict-and-artifacts.md) | verdict 伪造、产物目录进快照绕过可见性、迟到回执绕过保留、产物排行与鉴权缺运行身份 |
| scheduled-task | [任务对话准入](scheduled-task/task-conversation-admission.md) | 对话暂停时接纳后永久排队、越过 waiting 消息执行、两条入口准入结论分叉、历史只显示原因码 |

## 已批准的会话分析与模型请求统计

会话读取 ADR 为 `approved`，HTTP 尝试统计 ADR 为 `implemented`；直接测试证据固定到 Cloud PR #69 与
#98 的提交，各义务独立标记 Covered/Partial/Missing，不据受控 HTTP 或组件测试宣布真实部署已验收。

| 领域 | 核心用例 | 重点风险 |
| --- | --- | --- |
| thread | [会话分析读取](thread/session-analysis-read-model.md) | 当前权限、读取后撤权、归档完整性、只读身份、来源与未知值 |
| model-access | [已登记 HTTP 尝试统计](model-access/recorded-http-attempt-statistics.md) | 发送前登记、幂等收尾、取消、未知用量、IssueRun/WorkflowRun 读取授权和无回填升级 |

## 已批准的 Git 原生代码链路契约

[Git Server 根决策](../../decisions/cloud/git-server/0-project-code-hub-git-server-over-https.md)与[代码中心契约决策](../../decisions/cloud/git-server/20261016-project-repository-as-code-hub-and-ref-contract.md)经 owner 批准为 `approved`，契约是内部线与外部线共同遵守的边界，两篇共用一组核心用例。底座（落地顺序第 1 步）的直接证据在 [cloud#75](https://github.com/ora-space/cloud/pull/75) 与 [cluster#26](https://github.com/ora-space/cluster/pull/26)；外部线的真实同步已随 cloud#76（`ceb39bc`）接入，真实 clone 由内部线接入，相应义务如实保留 Partial/Missing。

| 领域 | 核心用例 | 重点风险 |
| --- | --- | --- |
| git-server | [代码中心契约](git-server/project-repository-code-hub-contract.md) | 监听与上下文头错配、ref 规则绕过、已结束 Operation 仍可写、删除回滚丢同步、快照领先 Git Server、跨 Project 可达、执行器出网越过部署白名单 |

## 已实现的外部线

外部线的四篇决策（[Repo Executor 根决策](../../decisions/cloud/repo-executor/0-repo-executor-sole-external-git-boundary.md)、
[同步](../../decisions/cloud/project/20261016-repository-binding-and-snapshot-sync.md)、
[发布](../../decisions/cloud/project/20261017-explicit-fast-forward-publish.md)、
[Git 凭据存储](../../decisions/cloud/identity-access/20261016-git-credentials-in-independent-credential-store.md)）已随
cloud#76（`ceb39bc`）与 cluster#27（`033a4ea`）合入 main 并改为 `implemented`；它们在契约之上各自登记
执行器、同步、发布与凭据的义务。证据按 cloud 该提交的直接测试登记，没有直接测试的义务保持 Missing。

| 领域 | 核心用例 | 重点风险 |
| --- | --- | --- |
| repo-executor | [执行器边界](repo-executor/repo-executor-boundary.md) | 授权与任务不符、明文凭据泄露、出网越过共享策略或 DNS rebinding、lease 失效、`uncertain` 后重推、进程残留 |
| project | [仓库同步](project/repository-binding-and-snapshot-sync.md) | 同一 Operation 执行两次、expected old 过时、未核验推进快照、终态事务建两个后继、凭据来源重复、等待快照不结束 |
| project | [显式发布](project/explicit-fast-forward-publish.md) | 越权审批、发布源校验与契约不一致、失去租约后重推、`uncertain` 无上限、发布后快照不含发布提交 |
| identity-access | [Git 凭据存储](identity-access/git-credential-store.md) | token 经过 Cloud、授权重放或跨用途、未确认就 `stored`、能力跨版本延续、管理员使用他人个人凭据、撤销不彻底 |

内部线的三篇 Cloud 决策（[clone 步骤消费快照](../../decisions/cloud/operation/20261016-clone-step-consumes-binding-snapshot.md)、
[快照 clone 输入与执行授权](../../decisions/cloud/controller-integration/20261016-snapshot-clone-input-and-execution-ref-grants.md)、
[Revision Git 交付](../../decisions/cloud/revision/20261017-git-native-delivery-refs-and-resume.md)）为 `implemented`，在契约之上
登记 clone 消费快照、执行授权与 Git 交付的义务，按现有测试登记证据并保留缺口；Node 一侧见 [Node 核心测试用例](../node/README.md)。

| 领域 | 核心用例 | 重点风险 |
| --- | --- | --- |
| operation | [clone 步骤消费快照](operation/clone-step-consumes-binding-snapshot.md) | 重试换了快照、等待者不结束或挡住领取、分支不是同步的分支、隐式同步、删除与进行中的同步竞争 |
| controller-integration | [快照 clone 输入与执行授权](controller-integration/snapshot-clone-input-and-execution-grants.md) | Controller 拼输入、基线提交写错、残留外部 clone 入口、执行越权写 ref、WorkflowRun 会话误获授权、bundle 契约残留 |
| revision | [Git 交付与续作](revision/git-native-delivery-refs-and-resume.md) | 未核验就登记、短暂不可用被判失败、`unchanged` 没有自己的 ref、续作拨回旧提交、一次取消永久不可续 |

## 多 Agent 个人配置与静态声明分发

[Agent 配置声明](../../decisions/desktop/plugin/manifest/20261017-agent-configuration-declarations.md)、
[Cloud 静态元数据分发](../../decisions/cloud/plugin-marketplace/20261017-agent-configuration-metadata.md)、
[个人 Agent 配置](../../decisions/cloud/model-access/20261017-personal-agent-configurations.md)与
[插件模型 adapter](../../decisions/node/agent-runtime/20261017-plugin-owned-model-adapters.md)四份 ADR 于 2026-10-17 经 owner
批准，现均已实现，状态为 `implemented`。声明分发与个人配置义务的证据多为 store 层或单用户 HTTP 的 Partial，缺口逐项写在
用例里；声明分发中离线发布一致性仍是 Missing；OpenCode 与个人连接的既有 Covered/Partial 证据保持原范围。Node 一侧的 adapter 用例见 [Node 核心测试用例](../node/README.md)，Desktop 一侧的声明用例见其领域目录。

| 领域 | 核心用例 | 重点风险 |
| --- | --- | --- |
| plugin-marketplace | [配置元数据分发](plugin-marketplace/agent-configuration-metadata.md) | 配置读取触发执行/下载、catalog 更新改写已选声明、安装结果与计划脱钩、安装权越权读个人配置 |
| model-access | [个人 Agent 配置、冻结与授权](model-access/personal-agent-configurations.md) | Key 进普通配置、CAS 失效、默认值漂移、Thread 越权、冻结不全、撤销复活、修复覆盖并发、多空间失配、别名越权、真实验收缺口 |

## 已实现的 Issue 协作交互

[Issue 协作根决策](../../decisions/cloud/issue-collaboration/0-typed-targets-and-explicit-execution-intent.md)为 `implemented`，补登 Cloud 已上线的协作契约。

| 领域 | 核心用例 | 重点风险 |
| --- | --- | --- |
| issue-collaboration | [协作交互](issue-collaboration/collaboration-interactions.md) | 提及误触发执行、模式与作者伪造、Confirm 重复或绕过校验、发起表单字段覆盖作者声明、Timeline 顺序 |

## 既有实现证据基线

| 仓库 | 提交 | PR | 交付职责 |
| --- | --- | --- | --- |
| Cloud | fe33eb49633028f3fd466de966d3a5754d126557 | [#37](https://github.com/ora-space/cloud/pull/37) | PostgreSQL 权威、使用权限、控制、强停、pending 与凭据冻结 |
| desktop | 9b788efbaa388908e4068e6f92a3da48439d4ec0 | [#632](https://github.com/ora-space/desktop/pull/632) | 精确锁定 Cloud；Controller 派发、Rust Node 持久绑定及权限隔离 |
| cluster | 3ce153f322ca31dd402cb1704d5d01e3dcc86d7e | [#6](https://github.com/ora-space/cluster/pull/6) | 管理认证、迟到创建屏障、真实 Git/进程/存储验收 |

这些固定提交提供后端实现的历史验证基线。真实验收八组检查的定位信息和可重跑脚本见对应 cluster 版本的 docs/runtime-control.md；它们只证明所测路径，后续新增义务以各核心用例的直接证据为准。

## 最终门禁

- Cloud fe33eb4：本机 Linux 使用真实 PostgreSQL 完整 check/race/build、官方前端生成/check/check:diff 通过；[后端 CI](https://github.com/ora-space/cloud/actions/runs/36435316523/job/108971606116)、[前端 CI](https://github.com/ora-space/cloud/actions/runs/36435316560/job/108971606485) 通过。
- desktop 9b788efb：[完整 CI](https://github.com/ora-space/desktop/actions/runs/36435361097) 全部通过，包括 Linux/Windows/macOS test:crates、前端、test:tauri、test:e2e。已核对 Linux、Tauri、e2e 的具体测试步骤为 success，未跳过。
- 本机 WSL 访问 Windows 文件系统时曾有前端 worker 启动超时与 Rust 并发时限失败；全部前端迁至 Linux 原生源码目录通过。保留失败记录，未改断言、期限、线程数或 stderr 门禁；完整 Rust/桌面验证以对应提交的成功 CI 为直接证据，停止额外复制/重跑。
- cluster 3ce153f：完整 Go race、格式/vet、真实公共 TLS Git 夹具及八组真实 Compose 验收通过。cluster 没有对应的 CI check-run，不能把本机结果写成 CI 通过。

这些门禁不补齐受控凭据解析/重绑或完整 UI、故障/升级矩阵，也不能替代各后续 ADR 的独立验证；各用例证据继续按义务标记 Covered、Partial、Missing。

## Cloud 单进程根决策

[Cloud 单进程根决策](../../decisions/cloud/lifecycle/0-single-process-cloud-on-enterprise-intranet.md)为 `approved`。
证据按 cloud `80e9c9f1` 的 `v2-server/` 登记（24 小时一条按 `0ca0a9a0`，租户授权一条按 `b03ac2b4`，首次内网部署准备一条按 `85bc9a5f` / cloud#362），9 项汇总为 2 Covered / 5 Partial / 2 Missing。

| 领域 | 核心用例 | 重点风险 |
| --- | --- | --- |
| lifecycle | [Cloud 单进程](lifecycle/single-process-cloud.md) | 重启后执行挂在进行中、恢复时重放、Node 退出后执行不结束、Key 和令牌下发给 Node、租户路由漏判成员或撤权后后台照常执行、全局锁回归、迁移乱序、运行超过 24 小时后失效、离线部署清数据/带入演示数据、同源前端吞掉 API 或路径穿越、离线资源搬移与关闭官方市场失效 |

## 项目仓库根决策

[项目仓库根决策](../../decisions/cloud/project-git/0-external-repository-pull-and-encrypted-token.md)为 `implemented`：手动同步、
当前快照随同步推进和私有仓库令牌已由 I7（circus #1057）实现。证据按 cloud `affb2a88` 的 `v2-server/` 登记，4 项汇总为
1 Covered / 3 Partial / 0 Missing。

| 领域 | 核心用例 | 重点风险 |
| --- | --- | --- |
| project-git | [项目仓库](project-git/external-repository-pull.md) | 地址夹带凭据或非 https、当前快照指向不完整的提交、同步改动已有会话的基准、同一项目并行同步、令牌泄露或被挪用 |

[回合交付](../../decisions/cloud/project-git/20261112-single-machine-turn-delivery.md)为 `implemented`：由 circus #1108 实现（cloud PR #410）。证据按 cloud
`7fb7c972` 的 `v2-server/` 与 `frontend-v2/` 登记，7 项汇总为 7 Covered / 0 Partial / 0 Missing。

| 领域 | 核心用例 | 重点风险 |
| --- | --- | --- |
| project-git | [回合交付](project-git/turn-delivery.md) | 回合没有交付或重复交付、交付漏文件或带进物化技能、改动 Agent 的 Git 状态、有记录无 ref 或 ref 被改写、同步动了交付 ref、交付前就进待验收、清理后看不到改动、越权读对话交付、重启后重做或晚交付、经 Cloud 入口写 ref |

[删除对话时回收工作目录](../../decisions/cloud/project-git/20261119-reclaim-deleted-conversation-workspace.md)为 `approved`：由 circus #1196 的运行 #1319
（cloud PR #419）实现，尚未合并。8 项汇总为 0 Covered / 0 Partial / 8 Missing。

| 领域 | 核心用例 | 重点风险 |
| --- | --- | --- |
| project-git | [工作目录回收](project-git/workspace-reclaim.md) | 默认删除丢掉运维也恢复不了的改动、过期的丢弃删掉用户没看到的内容、先删记录再盲扫、删到别人或项目外的目录、回收阻塞整个项目、崩溃后残留或误判保留、回收改动交付、保留的目录没人看得到 |

## 模型接入根决策

[模型接入根决策](../../decisions/cloud/model-proxy/0-in-process-model-proxy-with-deployment-connections.md)为 `implemented`（最后两项证据由 cloud#377 补齐），
后续决策 [长请求的空闲超时与总上限](../../decisions/cloud/model-proxy/20261113-in-flight-model-calls-bounded-by-idle-and-total-limits.md)为 `implemented`（D1–D3 由 cloud#367、D4 由 cloud#369 实现）。
证据按 cloud `80e9c9f1` 的 `v2-server/` 登记（24 小时一条按 `0ca0a9a0`，Key 不下发 Node 与重启后令牌失效两条按 cloud#377 head `24d7466b`，后续决策 D1–D3 的三条按 `4183eb9b`，D4 一条按 cloud#369 head `00c2fa52`），9 项汇总为 9 Covered / 0 Partial / 0 Missing。

| 领域 | 核心用例 | 重点风险 |
| --- | --- | --- |
| model-proxy | [模型接入](model-proxy/in-process-model-proxy.md) | Key 或上游信息泄露、未登记或跨项目的执行拿到令牌、会话结束后请求继续、Agent 换模型或夹带认证头、运行超过 24 小时后模型访问失效、长请求被令牌到期或固定时限切断、被切断的流像正常结束一样收尾、被切断后 Agent 反复重发 |

## 登录与会话根决策

[登录与会话根决策](../../decisions/cloud/login/0-idaas-or-email-login-with-shared-web-and-desktop-sessions.md)为 `approved`，
IDaaS 登录、邀请链接、移出成员和 desktop 令牌尚待实现。证据按 cloud `0ca0a9a0` 的 `v2-server/` 登记（“Exactly One Login Method Is Open” 的 `V2_LOGIN` 必填部分按 `85bc9a5f` / cloud#362 补充），10 项汇总为
0 Covered / 3 Partial / 7 Missing。

| 领域 | 核心用例 | 重点风险 |
| --- | --- | --- |
| login | [登录与会话](login/login-and-sessions.md) | 部署版开着邮箱登录、IDaaS 凭据落库或进日志、同一个人两个账号、登录即入队、邀请链接被重复使用、管理员来源不可信、移出后仍能访问、会话令牌明文落库、desktop 令牌未经确认签发、停用后会话仍可用 |
