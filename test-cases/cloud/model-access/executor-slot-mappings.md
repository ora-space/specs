# 个人槽位映射、启动门禁与按槽位锁定的绑定

依据[个人槽位映射与按槽位绑定决策](../../../decisions/cloud/model-access/20261015-executor-slot-mappings-and-per-slot-run-bindings.md)（2026-10-16 经 owner 批准），
它修改了[个人连接与运行代理根决策](../../../decisions/cloud/model-access/0-personal-model-connections-and-run-scoped-proxy.md)的 D3、D5。
单组绑定的冻结、撤销和授权语义仍由[个人模型访问用例](personal-model-access.md)维护；本文件只登记"一次运行多组绑定"带来的新义务。
图里槽位怎样声明见[执行器槽位用例](../workflow/executor-slots.md)。

证据基线：Cloud main 没有个人槽位映射，`run_model_bindings` 是
`run_id uuid NOT NULL UNIQUE REFERENCES issue_runs(id)`（`0033_personal_model_connections.sql`），即一次 IssueRun 一组绑定。
工作流启动门禁已随 [cloud#65](https://github.com/ora-space/cloud/pull/65) 合并进 main（合并提交
[`df507f0`](https://github.com/ora-space/cloud/commit/df507f0)），下面按逐条核对的证据登记。

## Personal Slot Mappings Stay Owner Scoped And Default Is The Default Model

- 风险：映射可被他人或管理员读写，运行者的槽位被指向别人的连接；`default` 另存一份后与个人默认模型不一致；连接删除时映射被级联删除，用户只看到"未配置"而不知道原因。
- 前置状态：两个有效用户，各有个人连接、默认模型和若干槽位映射；其中一个连接随后被停用、删除或下架模型。
- 触发：读写本人和他人的映射；写入名为 `default` 的映射；写入指向他人连接或连接里不存在的模型的映射；设置个人默认模型后解析 `default`；删除或停用映射引用的连接。
- 必须成立：归属来自已验证身份，只有本人能读写，管理员不因角色获得读取权；写入沿用资源版本和幂等键；写入时校验连接属于本人、模型在该连接的模型列表里；`default` 就是个人默认模型，映射里不能出现 `default` 条目；连接删除、停用或模型下架时映射保留，门禁按未配置拒绝并给出原因。
- 禁止结果：请求体选定映射归属、管理员读到他人映射、`default` 存在两份可分别修改的配置、映射指向他人连接、连接删除后映射消失。
- 验证义务与证据：

| 义务 | 所需直接证据 | cloud 侧状态 |
| --- | --- | --- |
| 映射只有本人可读写，写入受版本和幂等约束 | 真实 PostgreSQL + HTTP：跨用户、管理员读写拒绝，并发写入与重复写入 | Missing |
| 写入校验连接归属和模型存在 | 真实 HTTP：他人连接、不存在的模型被拒并可定位 | Missing |
| `default` 就是个人默认模型 | 真实 PostgreSQL：设置默认模型后 `default` 槽位按它解析；写 `default` 映射被拒 | Partial |
| 连接失效时映射保留并说明原因 | 真实 PostgreSQL + HTTP：删除、停用连接或下架模型后映射仍在，门禁原因可区分 | Missing |

- 决策依据：D1，不变量 5；继承根决策 D1。
- 本次直接证据：**Partial** — 个人默认模型本身的归属和版本由 main 的 `internal/core/model_connections_db_test.go`（`TestPersonalModelOwnershipDefaultsAndVersions`）覆盖；[cloud#65](https://github.com/ora-space/cloud/pull/65) 的 `requireWorkflowExecutorSlots` 让 `default` 槽位直接复用 OpenCode 运行绑定的 `defaultExecutorGap`，`integration/workflow_admission_test.go` 的 `TestWorkflowRunDefaultSlotGate` 断言设置默认模型后 `default` 通过门禁，停用连接、下架模型时原因分别是 `connection_unavailable` / `model_unavailable`。
- 仍缺：个人槽位映射的存储、接口和设置页都不存在（cloud#65 把所有命名槽位报为 `slot_not_configured`）；写 `default` 映射被拒、映射级联保留没有直接证据。

## An Unresolvable Slot Is Reported To The Initiator And Never Substituted

- 风险：解析用了错的人的映射（作者、发布者或别的成员）；某个槽位未配置或不可用时被 `default`、`hint` 或作者的选择悄悄顶替；定时任务无人在场时一直重试或静默跳过。
- 前置状态：已发布的图引用 `default` 和若干命名槽位；发起者的映射里有的槽位缺失、有的指向不可用的连接；图的作者、发布者和发起者是不同的人。
- 触发：直接运行、manual workflow IssueRun、`@` Confirm 和定时任务四种入口分别创建运行。
- 必须成立：解析的是本次运行发起者当时的映射（直接运行是点"运行"的人，manual IssueRun 是创建它的人，`@` 是点 Confirm 的人，定时任务是任务所有者）；从被运行的快照收集所有被引用的槽位（含隐式 `default`），任一槽位未配置或不可用，整次创建回滚，不产生运行、不消费额度；错误一次列出全部缺口，每个缺口带槽位名、`description`、引用它的节点和原因；定时任务的那次触发失败并通知任务所有者缺哪些槽位，不重试到配好为止；发布不检查任何人的映射。
- 禁止结果：用作者或发布者的映射解析、用 `default`、`hint` 或作者的选择顶替、只报第一个缺口、被拒后留下运行行、定时任务反复重试或无通知地跳过。
- 验证义务与证据：

| 义务 | 所需直接证据 | cloud 侧状态 |
| --- | --- | --- |
| 解析发起者的映射，作者与发布者的配置不起作用 | 真实 HTTP + PostgreSQL：作者已配置、发起者未配置时被拒；反之通过 | Partial |
| 任一缺口整次回滚，不发生顶替 | 真实 PostgreSQL：缺口下运行行与绑定行都不存在；其他槽位或 `default` 已配置时仍被拒 | Partial |
| 一次列出全部缺口，带说明、节点和原因 | 真实 HTTP：多个槽位同时缺失时的拒绝参数 | Partial |
| 定时任务触发失败、通知所有者、不重试 | 真实调度器 + 通知：缺槽位的触发结果与通知内容 | Missing |

- 决策依据：D2，不变量 3；继承根决策 D3 的回滚语义。
- 本次直接证据：**Partial** — [cloud#65](https://github.com/ora-space/cloud/pull/65) 的 `admitWorkflowRun` 以调用者身份解析，`requireWorkflowExecutorSlots` 先收集全部缺口再以 `workflow_executor_unconfigured` 拒绝，每个缺口带 `slot`、`nodeIds`、`reason`、`description` 和 `hint`；`integration/workflow_admission_test.go` 的 `TestWorkflowRunDefaultSlotGate` 断言每次拒绝后只存在那一次被放行的运行，`TestWorkflowRunNamedSlotGate` 断言默认模型已配置时命名槽位仍以 `slot_not_configured` 拒绝。
- 仍缺：没有个人槽位映射，"作者已配置、发起者未配置"无法构造；多缺口同时出现的断言、四种入口逐一的发起者身份没有直接证据；定时任务的工作流触发属[定时任务执行归属](../../../decisions/desktop/core/scheduled-task/0-local-and-cloud-execution-ownership.md)的第二阶段，尚无实现。

## Each Referenced Slot Freezes One Binding That Retries And Recovery Reuse

- 风险：一次运行仍只有一组绑定，槽位失去意义；或者节点启动时才解析映射，运行者改一次设置就改变在途运行；撤销后的绑定被别的槽位或新映射顶上。
- 前置状态：图引用 `default`、`strong`、`fast` 三个槽位；发起者三者都已配置；另有一个不带工作流的 OpenCode IssueRun。
- 触发：创建运行；运行期间修改映射、默认模型或作者的 `hint`；节点重试、运行恢复；清除密钥或停用某个槽位所用的连接；查看运行详情。
- 必须成立：创建事务里为每个用到的槽位锁定恰好一组绑定，记录槽位名，按运行实体（WorkflowRun 或 IssueRun）的（运行，槽位）唯一；节点重试和运行恢复都用这份记录，不重新解析映射；修改映射只影响之后新建的运行；撤销语义不变，被撤销的槽位后续节点申请授权失败、按节点失败处理，重新启用连接不复活；不带工作流的 OpenCode IssueRun 锁定一组槽位名为 `default` 的绑定，行为不变；运行详情展示每个槽位的连接名和模型，不展示密钥、密钥引用或服务地址。
- 禁止结果：同一（运行，槽位）出现两组绑定、未被引用的槽位产生绑定、重试或恢复时重新解析、撤销后换用别的绑定、现有 OpenCode IssueRun 行为改变、运行详情泄露密钥或服务地址。
- 验证义务与证据：

| 义务 | 所需直接证据 | cloud 侧状态 |
| --- | --- | --- |
| 每个用到的槽位恰好一组绑定，（运行，槽位）唯一 | 真实 PostgreSQL：多槽位运行的绑定行与唯一约束，两种运行实体各自成立 | Missing |
| 运行期间改映射不影响在途运行，重试和恢复不重新解析 | 真实 PostgreSQL + 执行侧：改映射后重试与恢复仍用原绑定 | Missing |
| 撤销某槽位只让该槽位的节点失败且不可复活 | 真实 PostgreSQL：撤销一个槽位的连接后，其他槽位照常授权，该槽位授权失败，重新启用仍失败 | Partial |
| 不带工作流的 OpenCode IssueRun 等价于单个 `default` 槽位 | 真实 PostgreSQL：迁移后的绑定行槽位名为 `default`，既有冻结与撤销测试不变 | Partial |
| 运行详情只展示连接名和模型 | 真实 HTTP：响应结构不含密钥、密钥引用和服务地址 | Missing |

- 决策依据：D3，不变量 1、2、6；修改根决策 D3。
- 本次直接证据：**Partial** — 单组绑定的冻结、撤销不复活由 main 的 `internal/core/model_connections_db_test.go`（`TestOpenCodeModelRunAtomicSnapshotAndThreadPermissions`、`TestModelGrantsScopeRenewalFreezeAndRevocation`）直接覆盖，它们是"每个槽位一组"在单槽位下的特例，也是 OpenCode IssueRun 行为不变的回归基线。
- 仍缺：`run_model_bindings` 还没有槽位名、运行实体判别列和（运行，槽位）唯一约束，WorkflowRun 也没有绑定；多槽位、重试与恢复不重新解析、运行详情都没有直接证据，随[工作流执行根决策](../../../decisions/cloud/workflow/)落地顺序第 4 步交付。

## A Node Session Is Granted Only Its Own Slot's Binding

- 风险：一个节点会话拿另一个槽位、另一次运行的绑定申请授权，用上了不属于它的模型和额度；代理仍按"运行冻结值"校验，在多组绑定下含义不明。
- 前置状态：一次运行有 `strong`、`fast` 两组绑定；引用 `strong` 的节点会话已派发，Cloud 已登记该会话执行与绑定的对应关系。
- 触发：会话用自己槽位的绑定申请、续期授权；会话改用同一运行另一个槽位的绑定，或另一次运行的绑定；用已取得的授权请求另一个模型 ID。
- 必须成立：会话规格带它所属槽位的 `model_binding_id`，Node 协议不变；授权除根决策 D4 的检查外，还要求绑定属于该会话执行登记的运行（agent IssueRun 或 WorkflowRun），并对应该节点引用的槽位；对应关系由 Cloud 在派发时登记，不取信于请求体；代理要求请求的模型 ID 等于授权所指绑定的冻结值。
- 禁止结果：会话用别的槽位或别的运行的绑定取得授权、按请求体声明的槽位授权、代理放行另一组绑定的模型、为此修改 Node 或 Controller 协议。
- 验证义务与证据：

| 义务 | 所需直接证据 | cloud 侧状态 |
| --- | --- | --- |
| 授权只对会话所属槽位的绑定成立 | 真实 PostgreSQL：同一运行另一槽位、另一运行的绑定申请授权都被拒 | Missing |
| 代理按授权所指绑定校验模型 ID | 真实 HTTP 代理：多槽位运行中用 `fast` 的授权请求 `strong` 的模型被拒 | Partial |

- 决策依据：D4，不变量 4；修改根决策 D5，继承根决策 D4。
- 本次直接证据：**Partial** — 单组绑定下"授权跟随确认的运行范围、模型必须等于冻结值"由 main 的 `TestModelGrantsScopeRenewalFreezeAndRevocation` 与 `internal/modelgateway/service_test.go` 覆盖；它们证明了按绑定校验模型的代理行为，但没有"同一运行多组绑定"的场景。
- 仍缺：会话与槽位绑定的登记、跨槽位授权拒绝都没有实现；`execution_work` / `node_executions` 的 WorkflowRun 归属也还不存在（工作流执行根决策 D8）。
