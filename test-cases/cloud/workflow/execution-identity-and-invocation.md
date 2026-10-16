# 工作流运行的身份冻结与调用绑定

依据[工作流执行根决策](../../../decisions/cloud/workflow/0-cloud-side-scheduler-owns-workflow-execution.md)（D3，2026-10-16 经 owner 批准）
与它的[个人槽位映射后续决策](../../../decisions/cloud/model-access/20261015-executor-slot-mappings-and-per-slot-run-bindings.md)。
Cloud 今天只把快照与输入落进 `workflow_runs`，运行没有发起者、Project、运行 Workspace 归属与逐槽位模型绑定，
创建后也没有执行者（`0029_workflow_runs.sql` 明确"Cloud has no workflow engine"，新建运行停在 `pending`）。
因此本文件多数义务只到 `Partial` / `Missing`，随 D3 落地顺序第 4 步补齐。

## A Workflow Run Executes One Frozen Document For Its Whole Life

- 风险：运行创建后作者继续编辑并再次发布，或插件升级，"这次运行跑了什么"随之改变，重跑、对账和排障都失去基准。
- 前置状态：workflow 已发布过一个快照，运行由它创建；作者同时在编辑活稿。
- 触发：运行创建后保存草稿、再发布新版本、还原历史快照、升级插件，然后读取该运行的图与节点状态。
- 必须成立：运行始终渲染并执行创建时冻结的那一份快照图；活稿的编辑与新发布不改变已存在运行的图、节点状态的含义与终态依据；插件升级不改变在途运行使用的插件版本。
- 禁止结果：运行跟随活稿漂移、把新版本的节点或连线补进已存在的运行、用"最新版本"回答一个已创建运行的执行内容、因发布新版本而重写历史运行的终态。
- 验证义务与证据：

| 义务 | 所需直接证据 | cloud 侧状态 |
| --- | --- | --- |
| 运行的图来自它自己的冻结快照，而不是活稿 | 真实 HTTP + PostgreSQL：创建运行后编辑并重新发布活稿，运行详情与节点渲染仍是旧图 | Partial |
| 执行内容与插件版本在整个运行生命周期内不变 | 真实执行侧：在途运行期间发布新版本并升级插件，运行按冻结内容完成，结果与两者无关 | Missing |

- 决策依据：D3（冻结表），不变量 1、2。
- desktop 参考实现（`8909f07`）一侧：**Missing** — 没有找到"运行期间执行内容不被后续编辑或发布改变"的直接测试。desktop 另有 [`crates/application/src/workflow_run/engine/snapshot_switch.rs`](https://github.com/ora-space/desktop/blob/8909f07/crates/application/src/workflow_run/engine/snapshot_switch.rs)：它支持在运行中替换冻结图，本决策按 D4 首期不继承，所以"图在整个生命周期内不可变"是 Cloud 首期的承诺，不是参考实现已有的承诺。
- 本次直接证据：**Partial** — [Cloud `0029_workflow_runs.sql`](https://github.com/ora-space/cloud/blob/7704089d453f88d66e06d462aafe554e03a11855/internal/core/migrations/0029_workflow_runs.sql) 把图经 `snapshot_id` 从不可变的 `workflow_snapshots.graph` 读入而不复制进运行行；[integration/workflow_run_test.go](https://github.com/ora-space/cloud/blob/7704089d453f88d66e06d462aafe554e03a11855/integration/workflow_run_test.go) 的 `TestWorkflowRunLifecycle` 断言运行钉住被选快照（`snapshotId`）并携带该快照的图（`definitionSnapshot`），`TestWorkflowRunTenantIsolation` 覆盖跨租户不可见。
- 仍缺：Cloud 还没有执行者，"运行期间"这段窗口不存在，也没有"创建后编辑并重新发布"的断言；插件版本尚未成为运行身份的一部分（D3 冻结表最后一行）。

## The Execution Identity Is Frozen In The Transaction That Creates The Run

- 风险：运行只冻结了图，发起者、Project、运行 Workspace 与逐槽位模型绑定在派发时才解析——那时代理配置、成员身份或映射可能已经变了，执行的用户与模型与用户批准的那次不是同一个。
- 前置状态：三种入口都可创建运行；发起者已配置好所需的个人连接与槽位映射。
- 触发：创建运行时其中一项不可用（连接被停用、凭据缺失、模型下架、槽位未映射）；或创建后发起者修改个人配置。
- 必须成立：快照、输入、发起者、Project、运行 Workspace 归属、逐槽位模型绑定与插件版本在创建运行的同一条事务里落库，任一项不可用则整个创建回滚；运行的 Workspace 属于 WorkflowRun（`workspaces.workflow_run_id`，与 `issue_runs` 的归属互斥且恰好一个非空）；绑定按（运行，槽位）唯一。
- 禁止结果：部分身份落库的运行、可被派发的半成品运行、创建后再解析绑定、agent IssueRun 的既有归属被顺带改变。
- 验证义务与证据：

| 义务 | 所需直接证据 | cloud 侧状态 |
| --- | --- | --- |
| 快照与输入在创建事务内冻结 | 真实 PostgreSQL：创建失败不留任何运行行 | Partial |
| 发起者、Project、Workspace 归属、逐槽位绑定与插件版本一并冻结，缺一即回滚 | 真实 PostgreSQL：逐项不可用时创建回滚，且运行行、Workspace 行、绑定行都不存在 | Missing |

- 决策依据：D3（冻结表），不变量 2；继承 [model-access 根决策](../../../decisions/cloud/model-access/0-personal-model-connections-and-run-scoped-proxy.md) D3 的冻结语义。
- desktop 参考实现（`8909f07`）一侧：**Missing** — 本轮没有取得"执行身份在创建事务里落库、缺一即回滚"的 desktop 证据；发起者的模型冻结语义在 Cloud 侧由 [个人模型访问用例](../model-access/personal-model-access.md#model-bindings-must-freeze-inputs-and-never-revive-after-revocation) 单独维护，本用例只负责工作流运行自己的身份列与 Workspace 归属。
- 本次直接证据：**Partial** — [Cloud `internal/core/workflow_run.go`](https://github.com/ora-space/cloud/blob/7704089d453f88d66e06d462aafe554e03a11855/internal/core/workflow_run.go) 在创建事务里解析快照并写入 `snapshot_id` / `input`，没有已发布快照时以 `workflow_no_published_snapshot` 拒绝，所以不会留下半成品运行；[integration/workflow_run_test.go](https://github.com/ora-space/cloud/blob/7704089d453f88d66e06d462aafe554e03a11855/integration/workflow_run_test.go) 覆盖该拒绝路径。
- 仍缺：`workflow_runs` 至今没有 `created_by` / `project_id`（[`0029_workflow_runs.sql`](https://github.com/ora-space/cloud/blob/7704089d453f88d66e06d462aafe554e03a11855/internal/core/migrations/0029_workflow_runs.sql)），也没有 `workspaces.workflow_run_id`；`run_model_bindings` 现在是 `run_id uuid NOT NULL UNIQUE REFERENCES issue_runs(id)`（[`0033_personal_model_connections.sql`](https://github.com/ora-space/cloud/blob/7704089d453f88d66e06d462aafe554e03a11855/internal/core/migrations/0033_personal_model_connections.sql)），即"一次运行一组"，判别列与（运行，槽位）唯一都还不存在。逐项不可用即回滚、绑定按槽位冻结都没有直接证据，属 D3 落地顺序第 4 步。

## One Workflow Issue Run References Exactly One Workflow Run

- 风险：两种运行实体（IssueRun 与 WorkflowRun）之间的引用可空、可多义或可被重跑复用，时间线展示的执行与真实执行对不上。
- 前置状态：`issue_runs.workflow_invocation_ref` 列已存在；manual 与 `@` Confirm 两种入口都会产生 workflow IssueRun。
- 触发：同一 issue 里重跑或重试；重复确认同一个 interaction；删除或归档 workflow；并发创建两个引用同一 WorkflowRun 的 IssueRun。
- 必须成立：一个 workflow IssueRun 恰好引用一个 WorkflowRun，一个 WorkflowRun 至多被一个 workflow IssueRun 引用；重跑创建新的 WorkflowRun 并沿用 `rerun_of_run_id` / `retry_of_run_id` 的既有形状；`issue_interactions.run_id` 仍指向那次调用的 IssueRun，其语义不变。
- 禁止结果：`workflow_invocation_ref` 保持不写入、一个 WorkflowRun 被两个 IssueRun 引用、用 `run_id` 反向改写 interaction 的既有含义、跨租户引用。
- 验证义务与证据：

| 义务 | 所需直接证据 | cloud 侧状态 |
| --- | --- | --- |
| 一个 workflow IssueRun 恰好引用一个 WorkflowRun（租户内） | 真实 PostgreSQL：重跑、重复确认与并发创建下的唯一性与租户隔离 | Missing |
| 无引用的 WorkflowRun 与无 WorkflowRun 的 workflow IssueRun 都不会被当成正常状态 | 真实 PostgreSQL：约束或门禁拒绝两种半连接状态，并给出可定位原因 | Missing |

- 决策依据：D3（三种启动方式表、`workflow_invocation_ref` 段），不变量 6；继承 [issue-run 根决策](../../../decisions/cloud/issue-run/0-agent-run-in-disposable-isolated-workspace.md) 的取消与终态语义。
- desktop 参考实现（`8909f07`）一侧：**Missing** — desktop 没有这条义务的对应实体：它的运行是本地聚合，不经 IssueRun 引用，"运行实体之间的引用"只存在于 Cloud。
- 本次直接证据：**Missing** — `workflow_invocation_ref` 只在 [`0010_issue_collaboration.sql`](https://github.com/ora-space/cloud/blob/7704089d453f88d66e06d462aafe554e03a11855/internal/core/migrations/0010_issue_collaboration.sql) 里作为可空列存在，全仓库没有写入者或读取者，也没有任何约束把它与 `workflow_runs` 关联。
- 仍缺：真实引用、两个方向的唯一约束、重跑与重复确认的行为，全部属 D3 落地顺序第 4 步；本用例在引用落地前不能标记为已有证据。
