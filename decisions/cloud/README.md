# Cloud 决策目录

本目录承载 Ora Cloud / API Server 的架构决策。Cloud 是多租户业务状态的权威入口，负责身份与
成员授权、Project 和 Workspace 聚合、持久操作、执行准入以及对 Controller 的受限内部接口；
Controller 和 Node 不得绕过 Cloud 建立另一份业务权威状态。

每个叶子目录的第一个 ADR 使用 `0-...md` 作为根决策，后续决策按 `YYYYMMDD-...md` 追加。目录按
Cloud 的领域与协议边界划分，不预先决定 Go package、数据库表或部署进程的拆分方式。

| 目录 | 范围 |
|---|---|
| `api-boundary` | 公开与内部 HTTP API、OpenAPI、分页、错误契约和输入边界 |
| `identity-access` | 外部身份映射、服务与用户凭据、认证上下文和访问判定 |
| `tenancy` | Tenant、Membership、角色、管理员保护和 owner 隔离 |
| `squad` | [人机编组根决策](squad/0-workspace-scoped-human-agent-squads.md) approved，尚待实现；加入事务归 tenancy，执行另行决策 |
| `project` | Project 聚合、Repository 意图、凭据引用和 Project 级资源状态 |
| `workspace` | Workspace 业务身份、归属和生命周期意图：协作 Space（`collab_workspaces`）与运行时工作区（`workspaces`）两类聚合 |
| `workspace-view` | 工作区查看：只读查看对话、工作流运行、任务运行的 Agent 工作目录（目录、文件、变更、diff）；[根决策](workspace-view/0-read-only-session-working-directory-view.md) proposed，是 #1029 正式契约的一部分 |
| `operation` | 持久操作、幂等、版本、重试、状态转换和恢复证据 |
| `execution-admission` | Access、Execution Ticket、准入关闭、idle 证据和活动保护 |
| `controller-integration` | Controller lease、内部快照、effect plan、epoch fencing 和接管边界 |
| `project-git` | 项目仓库：外部仓库的绑定、Cloud 进程内拉取与手动同步、当前快照、私有仓库令牌；[根决策](project-git/0-external-repository-pull-and-encrypted-token.md) approved（同步和令牌尚待实现） |
| `model-proxy` | 模型接入：Cloud 进程内的模型代理、部署级模型连接、按 Agent 会话签发的临时令牌；[根决策](model-proxy/0-in-process-model-proxy-with-deployment-connections.md) approved |
| `persistence` | PostgreSQL schema、migration、事务顺序、约束和兼容性 |
| `lifecycle` | Cloud 启动、健康检查、优雅停止、配置、升级和部署边界；[单进程、第一版单机、崩溃即中断的根决策](lifecycle/0-single-process-cloud-on-enterprise-intranet.md) approved，是 phase3 基座的根决策 |
| `plugin-marketplace` | 插件市场目录快照同步、工作区插件选择状态、fan-out 与自闭合 effect 载荷 |
| `issue-collaboration` | Issue 上的协作交互：`@` 目标与模式、评论/交互/运行的关系、Timeline、Workflow 发起表单与 Confirm 边界 |
| `issue-run` | Issue 上 @Agent 的运行编排：运行 Workspace、阶段状态机、取消、交付与释放 |
| `thread` | Agent 运行的对话记录：Thread 条目、用户追加消息、结束条件与实时刷新 |
| `revision` | 对象存储配置、上传授权、Revision 校验与登记 |
| `git-server` | 沙盒访问的 Git Server：协议、git-access 身份、按 Cloud 当前状态的逐请求授权与 ref 规则 |
| `repo-executor` | 唯一访问外部 Git 仓库的执行器：服务身份、一次性授权与凭据兑换、同步与发布的执行步骤、结果分类和部署 |
| `workflow` | 工作流图 schema、执行器槽位、发布与运行的门禁和执行契约 |
| `scheduled-task` | 定时任务：持久定义、服务端调度、幂等触发、无人值守授权与任务历史 |

Controller 的调度与恢复策略放在 `specs/decisions/controller/`，Node 的 Git、文件、进程、Agent、
Plugin 和本地执行职责放在 `specs/decisions/node/`。跨层决策放在拥有业务不变量的一侧，并从其他
层引用，不复制成多份权威规则。

[不可用 Node 延后与 clone 分轮等待](../controller/node-management/20261028-defer-unavailable-node-operations.md)
为 `implemented`；实现基线为 Cloud #206 合入版本 `1cfaed3` 与 Controller #750 合入版本 [`df694c8`](https://github.com/ora-space/desktop/commit/df694c8836fc70b1c789cc7e8e523d4d5b1b6c66)（最终 PR head `65fc457`）。
Cloud 持有不可用退避次数及 pending/timeout/Unknown 有限原因，并自动接管 timeout 后
原执行晚到的已知结果；这些跨层义务统一登记在[Controller 核心用例](../../test-cases/controller/node-management/20261028-defer-unavailable-node-operations.md)，
按直接测试和部署实际覆盖分别标记 `Covered`、`Partial`、`Missing`，实现状态不消除缺口；
多项排队验证要求不同 Project 各自的 Workspace Node 同时失联，旧项持续循环时健康 B/C 真实排队并推进；
blocked/node_unavailable 无界 retrySeconds 保留为独立后续问题。
本处只导航，不另建重复的 Cloud 决策或用例。

[交付执行结算与未受理证明](issue-run/20261026-delivery-execution-settlement-and-non-acceptance.md)
为 `approved`：确定 Cloud 业务结算与 Node 原身份输入围栏的分工；不表示已经实现或部署。

## 成员加入来源与链接预检

[成员加入来源与私有链接预检](tenancy/20261027-member-entry-provenance-and-link-preview.md)为 `approved`：
确立受限只读匿名预检、空操作不消费邀请、真实资格变更时同事务记录来源；不代表已实现。
验证义务见[tenancy 核心用例](../../test-cases/cloud/tenancy/member-entry-provenance-and-link-preview.md)。

## 小队与定向加入

[小队根决策](squad/0-workspace-scoped-human-agent-squads.md)与
[tenancy 定向加入后续决策](tenancy/20261027-squad-targeted-joining-and-inviter-revocation.md)
均为 approved，尚待产品实现及直接验证。规则覆盖独立编组、原子加入、空操作不消费、匿名预检分级、
来源区分及撤权；不开放生产 Team 执行。核心用例见[小队](../../test-cases/cloud/squad/README.md)与
[定向加入](../../test-cases/cloud/tenancy/targeted-joining-and-revocation.md)，新增义务均为 Missing。

## 多人协作运行时控制

使用权限、独占准入、停止/强停、控制交付和插件维护五份 ADR 的后端规则为 `implemented`。
仓库凭据连续性也已 `implemented`（验证与执行改走 Repo Executor 与 Credential Store），三处前端 UI 与完整联合故障矩阵由独立 `approved` ADR 承接。
状态描述各自的交付边界；局部证据不等于真实跨进程故障矩阵全部通过。

Controller 继续遵守[只拨出与过渡监听退役](../controller/api-boundary/20260928-retire-transitional-json-surface-and-listener.md)，
不恢复已[退役的 minicloud](../minicloud/runtime/20260928-retire-minicloud.md)。真实联调由 cluster Compose 或等价部署承担。

| 阅读顺序 | ADR | 状态与边界 |
| --- | --- | --- |
| 1 | [运行时使用权限](workspace/20260927-runtime-workspace-use-permissions.md) | implemented：创建者/管理员、概况/内容与历史归属 |
| 2 | [独占操作与执行准入](execution-admission/20260927-exclusive-runtime-control.md) | implemented：会话、写活动、续期、撤回和安全交接 |
| 3 | [普通停止与管理员强停](operation/20260927-safe-stop-and-administrative-force-stop.md) | implemented：活动保护、真实终止证据与数据保留 |
| 4 | [跨进程协议与同步交付](controller-integration/20260927-fenced-runtime-control-delivery.md) | implemented：身份、旧资格隔离与持久责任 |
| 5 | [空间插件维护](plugin-marketplace/20260927-admin-selection-and-idle-maintenance.md) | implemented：管理员选择、维护互斥与持久等待 |
| 6 | [仓库凭据连续性](project/20260927-repository-credential-continuity.md) | implemented：团队/个人/未知引用、成员离开冻结与候选同步重绑 |
| 7 | [前端交互与联合故障验收](execution-admission/20261013-runtime-control-ui-and-fault-acceptance.md) | approved：三处 UI、十一项矩阵与未开放入口归属 |

### 实施边界与默认值

- 普通成员使用自己的运行时、当前管理员可使用同租户运行时；操作仍需独占。不建立成员共享授权或实时共同编辑。
- 后端实现 60 秒独占有效期、20 秒续期契约，以及管理通道双向 TLS 和工作负载隔离；完整故障及跨版本组合按后续 ADR 验收。
- [Node 插件执行与新 Workspace 补装](plugin-marketplace/20260928-node-executes-plugin-installs.md)已 implemented。按项目凭据解析和连续性仍单独交付，不能因插件执行落地而认为私仓能力完成。
- 未开放文件/PTY/完整 Agent 产品入口，须在各自能力决策批准并接入相同准入边界后开放；其验证责任见后续验收 ADR 的未解决问题。
- 直接证据见[核心用例索引](../../test-cases/cloud/README.md)，逐项维护 Covered、Partial、Missing。旧根决策状态不因后续 ADR implemented 而自动变化，原成员或 clone 测试不能替代新增安全边界的完整证据。
