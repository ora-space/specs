# Cloud 核心测试用例

本目录与 `specs/decisions/cloud/` 完全镜像。Cloud 各领域的 ADR 进入 `approved` 后，才在对应叶子
目录登记需要长期维护的租户与 owner 隔离、认证授权、聚合约束、幂等、并发、恢复、协议和升级用例。

占位目录只用于稳定领域路径，不表示已有决策或测试证据。核心用例必须按业务风险组织，并使用
`Covered`、`Partial` 或 `Missing` 如实标记直接证据。

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
拒绝 `human` 与 `interactive: true`）。Cloud 侧调度器已随根决策落地：运行从 `pending` 进入真实执行，节点状态由
调度器与真实会话写入，模拟 trace 按根决策排除在验收之外；执行类义务的直接证据登记在各用例里，两个副本同时推进
同一个 run 的竞争与部分真实执行侧路径仍为 `Partial`。门禁类义务（图校验、启动门禁、槽位解析）的实现在
[cloud#65](https://github.com/ora-space/cloud/pull/65) 上，已随合并提交
[`df507f0`](https://github.com/ora-space/cloud/commit/df507f0) 落到 main，按逐条核对该提交的实现与测试后得到的证据
状态登记（`Covered` / `Partial`），不再按"未合并"整体降级。

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
| model-access | [个人槽位映射与按槽位绑定](model-access/executor-slot-mappings.md) | 映射越权、槽位被顶替、在途运行被改映射、会话借用别的槽位授权 |

## 已批准的定时任务决策

[Cloud Scheduled Task 根决策](../../decisions/cloud/scheduled-task/0-cloud-owned-scheduled-prompt-runs.md)经 owner 评审批准，状态为 `approved`；核心用例随合并同批建立（[admission-and-session-bounds](scheduled-task/admission-and-session-bounds.md)）。[Cloud Scheduled Run verdict 与产物 ADR](../../decisions/cloud/scheduled-task/20261022-run-verdict-and-artifacts.md)经 owner 批准为 `approved`，核心用例同批建立（[verdict-and-artifacts](scheduled-task/verdict-and-artifacts.md)）。实现尚未合入 main，义务全部为 `Missing`，随实现落地逐项更新；完成后 ADR 转 `implemented`。

| 领域 | 核心用例 | 重点风险 |
| --- | --- | --- |
| scheduled-task | [接纳、会话边界与历史保留](scheduled-task/admission-and-session-bounds.md) | 重复接纳、停机补记与超时误判、无人值守会话空等、资格失效仍接纳、历史无界增长 |
| scheduled-task | [verdict 与产物](scheduled-task/verdict-and-artifacts.md) | verdict 伪造、产物目录进快照绕过可见性、迟到回执绕过保留、产物排行与鉴权缺运行身份 |

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
