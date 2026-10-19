---
status: proposed
---

# Cloud 首期定时任务由服务端持久调度，并以任务所有者的无头 Agent IssueRun 执行

Cloud 首期 Scheduled Task 是绑定到一个 Issue 的持久定义：Cloud 以所选时区计算未来到期点，按
`(taskId, scheduledAt)` 幂等接纳，并在每个到期点以**任务所有者**的个人模型配置创建一个无头、单会话的
Agent IssueRun。运行仍使用既有 IssueRun、Thread、isolated Workspace、Revision 与残留回收链路；调度器不
直接调用模型、不另建 Agent 执行入口，也不触发 Workflow 或 `executorSlot`。本 ADR 处于 `proposed`：它只
冻结首期 Cloud 的行为与准入边界，不宣称调度器、API 或 Cloud UI 已经实现。

Cloud 的任务、运行与通知是 Cloud 持久事实，Desktop 关闭不影响已启用任务；本地任务和本地历史不上传、不
同步。Cloud Web 是首期唯一管理入口，Desktop 在对应 Cloud API、真实执行链和端到端验证全部就绪前继续将
`cloud` 显示为不可用，不能保存、代跑或模拟成功。

## 继承与修改

| 前序约定 | 本文处理 |
| --- | --- |
| [定时任务执行归属](../../../desktop/core/scheduled-task/0-local-and-cloud-execution-ownership.md) D1–D3：任务定义、冻结 revision、`ScheduledOccurrence{taskId, scheduledAt}` 与绑定 Issue 的 Agent IssueRun | 继承；本文补 Cloud 存储、准入和 API 边界 |
| 同 ADR D5：Cloud 只由服务端产生 future occurrence、首期只跑无头 Prompt 单会话、60 分钟超时需要 Cloud 取消写入者 | 继承并具体化；Workflow / `executorSlot` 保持非目标 |
| [IssueRun 根决策](../issue-run/0-agent-run-in-disposable-isolated-workspace.md) D2–D8：一个运行一个 Workspace、持久阶段、Thread/Revision/释放和残留事实 | 继承；定时任务只能经该链创建 Agent IssueRun，不能伪造终态 |
| [个人模型连接](../model-access/0-personal-model-connections-and-run-scoped-proxy.md) D1–D4：个人连接、运行绑定、撤销与短期模型授权 | 继承；调度服务身份不能取得或承担用户模型凭据 |
| [个人 Agent 配置](../model-access/20261017-personal-agent-configurations.md) D1–D4：运行冻结声明、配置与单模型绑定 | 继承；每个 occurrence 在接纳时重新核验后再冻结 |
| [残留资源回收](../issue-run/20261018-residual-agent-run-reclamation.md) | 继承；超时或失败不把未知资源写成已释放，残留仍按其授权和证据边界收尾 |

## D1：Cloud 任务归属于创建者、绑定一个 Space 内的 Issue，并分离定义可见性与运行可见性

`CloudScheduledTask` 至少持久化 Cloud 分配的稳定 ID、`tenantId`、`spaceId`、`issueId`、`projectId`、
`ownerUserId`、执行归属 `cloud`、启用状态、受限周期与 IANA 时区、下一次触发投影、定义 revision、冻结的
Prompt 目标、无人值守同意引用及创建/更新时间。创建请求的已认证用户就是不可变的 `ownerUserId`；请求体
不能代填他人。任务所有者承担每次 Agent Run 的模型使用和费用归属，也是该运行的 `Run Initiator`。

任务、Issue、Project 与所有者的当前 Membership 必须属于同一 Tenant/Space。一个 Cloud 任务只能绑定一个
Issue，首次创建后不得切换 owner、Issue、Project 或执行归属；需要改变这些边界时复制为新任务并停用旧任务。
Prompt、schedule、时区与启用状态可以编辑；影响执行输入的编辑提升 definition revision，已经接纳的运行
保留原 revision 和输入，不随之后编辑漂移。

任务定义（尤其 Prompt、模型配置显示和无人值守同意）仅任务所有者与当前 Space 管理员可读取、编辑或删除；
普通能读取绑定 Issue 的成员只可读取该 Issue 下的运行摘要、Thread 与已持久的结果，不能据此读取任务所有者的
模型连接、密钥、同意正文或编辑任务。owner 离开 Space、账号失效或任务被删除时不改写既有运行和审计事实，
只阻止后续接纳；重新加入或重新配置不会复活已撤销的旧运行绑定。

这一区分避免“能看 Issue”被误解释为能花费他人个人模型额度，也让审计仍能解释为何某次运行发生。

## D2：公开管理 API 只创建定义；调度器经内部用例创建无头 Agent IssueRun

首期公开 API 是 Cloud Web 调用的任务定义与历史接口：创建、读取、编辑、启停、软删除，以及按绑定 Issue
读取任务和 occurrence / 运行摘要。所有写请求带 `Idempotency-Key`，按所有者和请求语义比较重放；API 不接受
“scheduler actor”、任意 `ownerUserId`、模型凭据或任意 IssueRun 输入。无权限、跨 Space/Project/Issue 绑定、
无效 IANA 时区、非受限周期、无效同意或模型资格都在写入前明确拒绝。

到期接纳只调用 Cloud 内部的 `CreateScheduledAgentIssueRun(task, occurrence)` application use case。它在同一
业务事务中锁定 occurrence、读取任务 revision、快照当时的绑定 Issue 上下文和冻结 Prompt，并创建
`executor_type = agent` 的 IssueRun；新行继续使用既有 `issue_runs.issue_id`、Thread 路由、Workspace 和
Issue Timeline 契约。`IssueRun.input` 标记来源为 `scheduled_task`、引用 task / occurrence / revision，并保存
该次 Prompt 与 Issue 输入快照。公开 `POST .../runs`、评论 @Agent、Workflow 表单和 `ExecutionDispatcher`
不成为定时任务的旁路入口。

每个 occurrence 最多创建一个 Agent IssueRun。创建运行、冻结模型绑定和记录接纳结果必须原子化：重试、多个
scheduler 副本、事务重放或时钟回拨都读取同一 `(taskId, scheduledAt)` 记录；已创建运行则返回原引用，绝不
再次扣用模型或创建第二个 Thread。任务删除/停用与到期接纳竞争时，以同一事务首先锁定到的持久状态为准：已接纳的
运行按冻结输入继续收尾，未接纳者不再创建。

## D3：保存与每次接纳均复核任务所有者的授权；调度服务不替用户选择或支付模型

创建或启用任务时，Cloud 必须验证：owner 是有效成员、绑定 Issue/Project 仍在同一 Space、目标 Agent 与个人
配置可用、owner 已对 `full_auto_in_worktree` 作出明确同意、以及所选 Prompt 单会话目标不含 Workflow/
`executorSlot`。同意必须说明：Agent 会在 Cloud 运行 Workspace 中以所有者的模型额度执行任意工具调用，可能
读写该 Workspace、使用允许的仓库凭据并产生模型费用；同意时间和配置/目标 revision 与任务一起保存。

每次 occurrence 接纳时重复核验上述当前资格，再创建该次 IssueRun 的不可变模型绑定、Agent 配置快照、Prompt
输入和 Run Initiator。普通编辑只影响未来运行；连接 Key 清除、连接或绑定撤销、成员/账号失效、Project 失效、
模型不再可用或同意不再有效时，不能以重新启用任务、换 Key、服务身份或其他成员的默认配置复活。该 occurrence
以可解释的失败事实结束，并通知 owner；不得回退 local、换用其他成员模型、换成平台模型或默默排队到资格恢复。

运行期间的模型授权仍由既有 run-scoped grant 决定：Node 只取得属于该 IssueRun 的短期授权，Cloud scheduler、
Controller、Thread、日志与 Revision 永远不持有原始 Key。管理员可停用任务或结束运行，但不因管理权限取得
owner 的密钥、个人配置读取权或模型费用归属。

## D4：服务端按 IANA 时区持久求值，到期点有界接纳且不补跑无限积压

Cloud schedule 只支持每 N 小时、每天某时刻、每周若干天某时刻；每 N 小时最短 1 小时，不存储或解释 cron。
每天/每周按任务持久的 IANA 时区求值，每 N 小时以创建时写入的 UTC anchor 求值；最终 occurrence 一律持久为
UTC `scheduledAt`。夏令时跳过的当地时间不补跑，回拨的重复当地时间只接纳第一个 UTC 点；以后修改时区只影响
尚未生成的 future occurrence。

Cloud scheduler 是唯一产生和扫描 future occurrence 的写入者，周期与 Web 进程、浏览器或 Desktop 生命周期无关。
它以数据库时间判断到期，并在延迟不超过 10 分钟时接纳原 `scheduledAt`；超过 10 分钟的点记录一次 `missed`
事实且不执行、不集中补跑。这样服务重启或短暂延迟不会制造重复运行，长期停机也不会把日报等时效工作变成无界队列。
`nextRunAt` 只是可重算投影，不是接纳事实。

同一任务使用 `skip_if_running`：前一 Scheduled IssueRun 未终止时，新到期点记录 `skipped_busy`，不排队；项目
级运行准入或配额关闭时记录 `skipped_capacity`，同样不隐式重试。它们都保留 occurrence 身份和可读原因，不能
伪装成成功或由 local 承接。

## D5：60 分钟超时与通知通过既有 IssueRun 收尾；资源未知时交给残留回收而非伪造完成

从 Agent IssueRun 被接纳起，Cloud 以数据库时间计算固定 60 分钟 deadline。到期时由 Cloud 取消写入者向同一
IssueRun 写入取消请求并停止后续模型授权；它不直接删除 Workspace、Thread 或 Revision，也不把仍在途的会话写成
`cancelled`/`failed`。真实会话终态、交付和 Workspace 释放仍由 IssueRun 生命周期接管。取消写入者、scheduler、
模型授权和无头 Agent 执行链任一项尚未真正可用时，Cloud 任务不可启用。

每个持久事实在提交后生成 owner 范围的 Cloud 通知：开始、资格失败、失败、超时、`missed`、`skipped_busy` 与
`skipped_capacity`。通知只指向任务历史或相应 IssueRun / Thread；它不是运行状态源，也不含密钥、完整 Prompt
或未授权的配置内容。首期只承诺 Cloud Web 内的持久未读通知和历史入口，不承诺邮件、移动推送或 Desktop 原生通知。

若 IssueRun 的 Workspace 删除无法以真实证据完成，运行和关联资源继续由残留回收 ADR 的
`reconciling` / `reclaiming` / `cleared` 语义管理。定时任务不能因为 60 分钟 deadline、请求超时、无通知接收者
或 effect 查询 `404` 而把残留标为已清除、再次创建相同 occurrence，或绕开系统清理授权。

## D6：Cloud Web 首期管理 Cloud 任务；Desktop 只有在真实 Cloud 能力完成后才解除禁用

Cloud Web 首期负责 Cloud 任务定义、启停、历史、通知和跳转至绑定 IssueRun / Thread。它只展示本 ADR D1 定义的
可见范围，并把 API 返回的持久状态作为事实；没有可用的 scheduler、取消写入者或无头执行链时，Web 不能显示“已启用”
或接受 Cloud 任务。

Desktop 的 local 定义和历史仍只存本机，不列入 Cloud API，也不与 Cloud 任务合并。首期 Desktop 保持现有 Cloud
入口不可用并说明由 Cloud Web 管理；它不以本地 timer 代跑，不上传 local Prompt/历史，也不读取或伪造 Cloud
运行结果。后续 Desktop integration 只有在 Cloud API、持久 scheduler、幂等接纳、真实无头 IssueRun、取消写入者、
通知和本 ADR 的端到端验证均已合并且可操作时，才能另立实现切片解除禁用并选择是否展示远端列表。

## 不变量

1. 每个 Cloud Scheduled Task 的执行所有权、owner、Issue 和 Project 不可迁移；更换边界只能复制新任务。
2. 同一 `(taskId, scheduledAt)` 最多产生一个 Agent IssueRun，任何重放不产生第二次模型使用。
3. 每次运行的 Prompt、Issue 输入、Agent 配置、模型绑定和 Run Initiator 都在接纳时冻结。
4. scheduler 服务身份不能选择、读取或支付其他用户的个人模型；撤销的绑定不能复活。
5. Cloud 不以浏览器、Desktop timer、Workflow 或模拟终态代替持久调度和真实 IssueRun 生命周期。
6. 超时、通知投递失败或资源不可达不构成会话完成或资源已释放的证据。
7. local 任务和历史不上传 Cloud；Cloud 任务不回退到 local。

## 为什么不是这些替代方案

| 方案 | 取舍 |
| --- | --- |
| scheduler 以系统账号或项目 owner 的模型运行 | 实现看似集中，却会让没有明确同意的人承担费用并绕过个人连接归属 |
| 每次到期读取任务所有者当前默认模型 | 配置调整立即生效，但无法重放或解释某次运行实际使用的模型，也会让编辑影响在途运行 |
| Cloud 不可用时让 Desktop/local 自动接管 | 表面可用性更高，但会改变执行归属、凭据和费用，还可能与 Cloud occurrence 重复执行 |
| 重启后补跑全部历史到期点 | 不丢表面触发，但长期停机形成无界工作和过期副作用；首期选择明确 `missed` |
| 调度器直接调用模型或写运行成功 | 少一层调用，但绕开 Workspace、Thread、Revision、撤销和残留回收的现有事实边界 |
| 首期同时支持 Workflow 与 `executorSlot` | 覆盖更多目标，但 WorkflowRun 的冻结、槽位解析和执行链是独立决策，混入会让无头 Prompt 契约不可评审 |

## 本决策未解决的问题

- Cloud 任务的配额数值、跨任务全局并发上限、邮件/移动推送和通知保留期限；首期只固定可解释的
  `skipped_capacity` 和 Web 通知事实。
- Workflow 目标、`executorSlot`、已发布 Workflow snapshot 与个人槽位解析；这些需要单独 ADR 和实现切片。
- Desktop 对 Cloud 任务的远端列表、创建与管理体验；在 Cloud 首期真实链路完成后单独决定。
- Cloud 任务定义、运行摘要和通知的长期保留/归档政策。

## 落地顺序

1. **Cloud 领域与 API**：持久任务、occurrence、历史/通知模型和定义 API；完成条件是 Tenant/Space/Issue/Project/owner
   边界、IANA 时区、revision 与 `(taskId, scheduledAt)` 唯一约束均有 PostgreSQL 直接验证。
2. **准入与无头 IssueRun**：实现每次接纳的资格复核、模型绑定和内部 IssueRun 创建；完成条件是无有效同意、配置、
   Membership 或 Project 时明确失败，重放不创建第二次运行，且个人费用归属保持 owner。
3. **调度、超时与收尾**：服务端产生 occurrence、处理 `missed`/busy/capacity、写取消请求和通知；完成条件是多副本重放、
   重启、迟到、超时和通知失败都不伪造成功或释放。
4. **真实联调**：以真实 Agent IssueRun、Thread、Workspace、Revision 和残留回收进行端到端验证；所有环节完成前 Cloud
   选项继续禁用。
5. **Desktop integration（后续）**：另立任务消费已验证的 Cloud API；不得把远端展示或开关当作第 1–4 步完成的替代证据。

保持不变：local-only 数据边界、IssueRun/Thread/Revision 生命周期、个人模型密钥边界和残留资源回收。
从第 1–4 步完成后改变：Cloud Web 能创建并可靠运行绑定 Issue 的无头 Prompt 定时任务；从第 5 步完成后，Desktop
才可能解除 Cloud 入口禁用。
