# Cloud 核心测试用例

本目录与 `specs/decisions/cloud/` 完全镜像。Cloud 各领域的 ADR 进入 `approved` 后，才在对应叶子
目录登记需要长期维护的租户与 owner 隔离、认证授权、聚合约束、幂等、并发、恢复、协议和升级用例。

占位目录只用于稳定领域路径，不表示已有决策或测试证据。核心用例必须按业务风险组织，并使用
`Covered`、`Partial` 或 `Missing` 如实标记直接证据。

## 已批准的多人协作运行时控制

使用权限、独占准入、停止/强停、控制交付及插件维护五份 ADR 的后端规则为 `implemented`；仓库凭据连续性仍为 `approved`。三处 UI 与完整联合故障矩阵由[后续验收 ADR](../../decisions/cloud/execution-admission/20261013-runtime-control-ui-and-fault-acceptance.md)承接，各用例继续按直接证据保留 Partial/Missing。Node 插件执行已经落地，但不能据数据库或模拟器宣布完整产品与故障验收完成。

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

## 已批准的工作流执行决策

[工作流执行根决策](../../decisions/cloud/workflow/0-cloud-side-scheduler-owns-workflow-execution.md)与它的两份后续决策
[执行器槽位决策](../../decisions/cloud/workflow/20261015-executor-slots-replace-agent-model-ids.md)、
[个人槽位映射与按槽位绑定决策](../../decisions/cloud/model-access/20261015-executor-slot-mappings-and-per-slot-run-bindings.md)
于 2026-10-16 经 owner 批准，为 `approved`，本领域首期核心用例同批建立。Cloud 至今没有工作流执行者：生产里的
运行停在 `pending`，唯一写节点状态的是开发夹具，而根决策已把模拟 trace 排除在验收之外，所以执行类义务在
cloud 侧全部为 `Missing`；main 也还没有槽位与个人槽位映射。门禁类义务（图校验、启动门禁、槽位解析）的实现在
评审中的未合并 `preflight` [cloud#65](https://github.com/ora-space/cloud/pull/65)（head `0c064a2d`）上，按
`Partial` 登记，不当作已经落地。

desktop 参考实现（`8909f07`）的直接证据在每个用例里单独标注，并如实记下与它不同的地方：同一 run 内多个 agent 节点首期串行（desktop 在同一次调度 pass 里并行派发 sibling agent，所以"至多一个串行执行者"这一条在它身上是 `Partial`）、图在整个运行生命周期内不可变（desktop 另有 `snapshot_switch` 可替换冻结图）、发布时就拒绝不可执行节点（desktop 只在启动时拒绝）、启动表单用 `snapshotId`（desktop 无此表单）。这些义务没有 desktop 证据可依托，只能由 Cloud 自己的测试证明。

| 领域 | 核心用例 | 重点风险 |
| --- | --- | --- |
| workflow | [执行身份与调用绑定](workflow/execution-identity-and-invocation.md) | 图随活稿漂移、身份不全、IssueRun 与 WorkflowRun 的半连接 |
| workflow | [调度串行与恢复](workflow/scheduler-serialism-and-recovery.md) | 多副本重复派发、迟到回调、崩溃丢进度、谎报成功、没有取消入口 |
| workflow | [发布与启动门禁](workflow/publish-and-start-admission.md) | 冻结执行不了的图、UUID 绕过校验、槽位被顶替、快照字段错名 |
| workflow | [执行器槽位](workflow/executor-slots.md) | 槽位名变体、`hint` 被当兜底或被自动写入、旧图拒收或保留执行器 |
| model-access | [个人槽位映射与按槽位绑定](model-access/executor-slot-mappings.md) | 映射越权、槽位被顶替、在途运行被改映射、会话借用别的槽位授权 |

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
