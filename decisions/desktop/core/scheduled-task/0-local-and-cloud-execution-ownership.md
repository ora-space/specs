---
status: proposed
---

# 定时任务以持久定义产生幂等触发，执行归属在创建时确定且不支持切换

用户可以在 Desktop 左侧进入“定时任务”，创建周期性的代码质量检查、工作汇报、待办整理等任务，并为每个任务选择
`local` 或 `cloud` 执行模式。执行归属在创建时确定，之后不支持在两端间切换；需要换执行位置时，复制成新任务并停用
旧任务。`local` 只在 Desktop 进程运行时触发；`cloud` 由服务端拥有未来触发，因此 Desktop 关闭后仍可运行。两种模式
共享任务定义、occurrence 和运行状态的语义，各自持久化，不把浏览器计时器、内存状态或示例模板当作执行事实。

本 ADR 处于 `proposed`。它定义 Scheduled Task 新领域的执行归属、持久状态与分阶段交付边界，只冻结共用语义与本地
模式，不宣称 Cloud 调度、远程凭据、计费或任意 Workflow 已经交付。Cloud 首期的无头单会话 API 与权限校验作为
`decisions/cloud/` 下的独立 ADR 推进，不依赖 #60/#67 的 Server Workflow 执行链；只有该 Cloud 契约和真实执行适配器
就绪后，`cloud` 才能成为可选值。Cloud 触发已发布 Workflow 则是依赖 #67 的后续阶段。

> 本版依据 issue #84 的成员评审意见修订：删除执行归属切换握手，幂等键改为 `taskId + scheduledAt`，补触发迟到与
> 睡眠唤醒的 missed 规则，Cloud 部分收缩为另行决策，并按补充评审冻结本地数据驻留默认（不上传、不要求登录）。
> 最终以 owner 拍板结论为准。

## 决策

### D1：Scheduled Task 是持久定义，模板只负责预填

`ScheduledTask` 至少持久化：稳定 ID、名称、启用状态、schedule、执行目标引用及其冻结版本、执行归属、定义 revision、
创建/更新时间和下一次触发投影。执行归属在创建时写入，此后不可变更。代码质量检查、每日工作汇报和待办整理是可编辑的
模板；保存后都是普通 `ScheduledTask`，不会在运行时分叉出三套特殊逻辑。

本地任务及其运行历史只保存在这台设备上：不上传 Cloud、不跨设备同步，也不要求登录即可创建和运行；本地路径与提示词
等内容不离开本机。Cloud 任务及其运行历史的存储位置、可见范围与账号归属随 Cloud 决策定义，本地实现不得预埋上传、
同步或登录依赖。

schedule 首版只承诺可由产品表单完整表达的周期规则；若暴露 cron，存储与校验必须使用同一语法，不能把未经校验的
字符串直接交给执行端解释。local 任务不提供时区选择，schedule 按系统当前时区求值；时区作为持久化字段留给 Cloud
任务随 Cloud 决策定义。任务列表中的 `nextRunAt` 是可重算投影，不是触发事实。

### D2：每个到期点产生稳定的 occurrence，接纳运行是幂等操作

一次计划触发由 `ScheduledOccurrence{taskId, scheduledAt}` 唯一标识，其中 `scheduledAt` 是 UTC 时间点。同一个
occurrence 最多接纳一个 Scheduled Run；进程重启、事件重复投递、系统时间回拨或网络重试必须复用该身份，不能另建
一次运行。执行归属在创建时即固定，不参与身份计算；local 与 cloud 各自在自己的持久库中保证不重复，幂等是各端内部
规则而不是跨端协议。运行的实际副作用仍遵循目标执行器自己的重试与幂等约束；本决策只保证调度层不会把同一到期点重复
接纳为多个运行。

修改 schedule 或执行目标会增加 task revision。名称等纯展示修改不改变 revision。已经接纳的运行冻结原 revision 和
执行目标，不随任务后续编辑漂移；运行记录中的 `revision` 只标识这次运行使用的定义版本，不参与幂等。

### D3：执行目标引用既有真实用例，不由调度器解释 Prompt 或 Workflow

Scheduled Task 保存带版本的 `ExecutionTargetRef`，到期后把 occurrence 交给相应的 application use case。目标可以
在后续决策中扩展为 Agent Task、冻结 Workflow revision 等，但每种目标必须先定义：输入如何冻结、由谁取得凭据、运行
与 session 如何关联、取消及结果如何投影。调度器不得直接调用模型、拼装未版本化 Prompt，或把内存 Workflow runtime
的模拟成功写成真实运行。本地首版的执行目标是随 task revision 冻结的提示词，由新增的 application use case 承载。

Cloud 首期的执行目标同样只接受随 task revision 冻结的提示词，不绑定 Workflow snapshot。到期时 Cloud 调度器复用既有
Agent `IssueRun` / `Thread` application use case，在目标 Project / Workspace 下拉起一次会话，并沿用既有结果记录与收尾。
首期不涉及 `executorSlot`；无头单会话的 API 和权限校验在独立 Cloud ADR 定义，调度器不另建 Agent 运行入口。Cloud 触发
已发布 Workflow 是后续扩展，届时再定义目标校验与凭据边界。

### D4：local 由 Desktop 进程内 scheduler 拥有，错过与迟到的到期点一律记 missed

`local` 任务由 Desktop 后端的持久 repository 与原生 scheduler 组合驱动。WebView 只读取、编辑和订阅状态，不运行
计时器作为事实源。首版不注册操作系统服务、不设置开机自启，也不在 Desktop 关闭时唤醒机器。

实际触发时间晚于 `scheduledAt` 超过 10 分钟（固定阈值，不提供配置）的到期点一律记为 `missed`，不执行也不补跑。
这同时覆盖 Desktop 未运行、笔记本合盖睡眠后唤醒补触发和系统繁忙导致的迟到；10 分钟内的迟到触发照常执行。
`missed` 是任务级观察，在历史中可见并写明原因；不为离线区间逐条制造运行。用户可显式“立即运行”，它产生独立的
manual occurrence，不冒充原计划时间。未来若增加 catch-up，必须另行决定上限、合并、过期和并发策略。

每次本地定时运行在独立 worktree 中执行目标项目，不直接改动用户正在编辑的目录；worktree 创建复用 Desktop 已有的
初始化端口。运行结束后，无改动的 worktree 直接删除；有改动的 worktree 保留，运行历史显示其路径并提供“打开”和“丢弃”。
保留数量设上限，超过后删除最旧项，具体上限在实现阶段确定。保存为 active 的 local 任务必须通过目标目录是 Git 仓库的
校验；不满足时不得降级为直接在原目录执行。

### D5：cloud 交由服务端拥有的语义冻结，细节另行决策

`cloud` 任务的未来 occurrence 只由 Cloud 持久调度器产生，Desktop 关闭不影响它；这是需要提前冻结的共用语义。
Cloud 首期的未来 occurrence 由 Cloud 持久调度器产生，并以本 ADR 的 `(taskId + scheduledAt)` 规则在 Cloud 侧幂等
接纳。首期只调用 D3 所述的无头 Prompt 单会话链：保存及每次触发均校验目标 Project、模型配置和无人值守授权；不接入
#60/#67 的 Server Workflow 执行链，也不解析 `executorSlot`。Cloud 的任务登记、存储、取消、状态读取、授权、通知、发起者
归属与费用，以及无头会话 API 细节由独立 `decisions/cloud/` ADR 定义，本决策不预置协议。

Cloud 触发已发布 Workflow 是第二阶段能力，等待 #67 的结论及 Workflow server 执行链落地后再提出。该阶段每次触发按
任务所有者的个人执行器槽位映射解析执行器；不得复用其他用户的映射。所需槽位缺失时，该次触发明确失败并通知任务所有者。

在 Cloud 首期 ADR 获批且真实无头单会话链路就绪之前，Desktop 可以展示 `cloud` 模式及其说明，但必须标为不可用，不能
保存为执行归属、不能用本地 timer 代跑，也不能展示模拟成功。

两端只对齐展示用的字段名与状态名，不共享 ID、存储或同步协议；Desktop 任务列表是否合并展示 Cloud 任务、运行历史
能否跨端可见、Cloud 任务的账号与归属，均随 Cloud 决策一并定案。

### D6：时区求值、夏令时和并发规则

schedule 按系统当前时区求值，occurrence 落盘后只以 UTC `scheduledAt` 标识。夏令时跳过的本地时间不补跑；夏令时
回拨造成同一当地时间重复时，只接纳第一个对应的 UTC 时间点，第二个不产生 occurrence。系统时区变化只影响尚未求值的
未来触发，不改写已落盘的 occurrence；local 任务不接受用户单独指定时区。

同一任务的默认并发策略为 `skip_if_running`：到期时若前一 Scheduled Run 仍未终止，记录 `skipped_busy`，不排队
补跑。所有本地定时任务共享一个全局并发上限（首版固定 16）；达到上限时该到期点同样跳过并记录可区分的原因，不排队。
后续若提供 queue 或 parallel，必须显式进入任务定义并由目标执行器声明支持，不能只在 UI 增加选择项。

### D7：运行记录统一表达计划结果，删除任务不删除审计事实

Scheduled Run 至少关联 task、revision、occurrence、目标运行/session ID，并使用
`accepted / running / succeeded / failed / cancelled / skipped_busy / skipped_capacity` 状态。`missed` 是任务级观察，
不是伪造的运行，其原因只能是 `app_not_running`（Desktop 未运行期间到期）或 `late_trigger`（进程内触发晚于 10 分钟）。
失败至少区分 `target_unavailable`、`model_unavailable`、`worktree_failed`、`approval_policy_invalid`、`timeout` 与
`execution_error`。UI 的 last run、失败提示和历史列表来自这些持久事实，不从当前时间反推。

每次定时运行开始时发出桌面通知，点击进入对应会话；通知是交互辅助，不是运行事实源。

禁用任务只阻止未来接纳，不取消已经运行的工作。删除任务采用软删除或等价墓碑，保留既有运行关联；数据保留期由对应
产品决策定义。

### D8：local 与 Cloud 分别在保存和每次运行时检查资格

local 不记录账号身份或 Cloud 授权引用。保存 local 任务时检查目标项目目录存在、是 Git 仓库且模型配置可用；每次接纳
时再次检查，失败则该 occurrence 以明确失败结束，不换用其他模型。Cloud 首期保存及每次接纳均检查该 Prompt 运行需要的
目标 Project、模型配置与无人值守授权；失败时明确结束，不回退到 local，也不涉及 Workflow 的执行器槽位。Cloud 的创建者、
Workspace、模型凭据、费用与无头会话权限细节由独立 Cloud ADR 定义；第二阶段 Workflow 执行再继承 #67 的最终契约。

首版唯一的无人值守工具批准策略是 `full_auto_in_worktree`：只在独立 worktree 中运行，运行时工具调用全自动批准。
创建任务时必须以清晰风险说明取得用户显式同意，并将策略名与同意时间随任务持久化；说明至少覆盖 worktree 不是沙箱、
agent 仍可执行任意命令并读写 worktree 外文件、可使用本机凭据（例如 `git push`），且会消耗模型额度。仅当目标项目或
策略变更时才必须重新同意；只改提示词或计划时间不要求重新同意。每次接纳均校验该策略仍受 runtime 支持且 worktree 可
创建；不满足则以 `approval_policy_invalid` 或 `worktree_failed` 失败。每次运行固定 60 分钟，不可配置；超时后取消并记录
`timeout`。用户撤销同意后，任务自动停用，已有运行事实不改写。以后新增细粒度策略时，既有 `full_auto_in_worktree` 任务
不自动扩大或缩小权限。

## 不变量

1. 任务的执行归属在创建时确定，之后不可切换；每个任务的未来触发始终只由一个执行端产生。
2. 同一 `taskId + scheduledAt` 最多接纳一个 Scheduled Run。
3. WebView timer、模板和 `nextRunAt` 投影都不是触发事实源。
4. local 不承诺 Desktop 关闭或睡眠期间运行；实际触发晚于 `scheduledAt` 超过 10 分钟的到期点记 `missed`，不执行
   也不补跑。
5. cloud 首期只能通过真实调度和无头 Prompt 单会话链成为可选模式；不得以本地 timer 或 Workflow 模拟成功代替。
6. 已接纳运行冻结原 task revision 与执行目标，不因后续编辑漂移。
7. 调度器只调用已定义的 application use case，不自行解释或模拟执行目标。
8. 任务禁用或删除不改写既有运行事实。
9. local 真实执行只能使用已显式同意、仍有效的 `full_auto_in_worktree` 策略；目标项目或策略变更必须重新同意，撤销同意
   后任务自动停用；没有有效策略时不接纳运行。
10. 本地定时运行不直接改动用户正在编辑的目录，默认在独立 worktree 中执行。
11. 本地任务及其运行历史只保存在本机：不上传 Cloud、不跨设备同步，也不要求登录。
12. 保存为 active 的 local 任务必须以 Git 仓库为目标；不允许回退为直接修改原目录。

## 为什么不是这些替代方案

| 方案 | 取舍 |
| --- | --- |
| React / WebView `setInterval` 直接触发 | UI 开着时实现快，但刷新、休眠和多窗口会漏触发或重复触发，也没有持久幂等事实 |
| local 与 cloud 同时监听，谁先抢到谁执行 | 能自动容灾，但需要跨端强一致租约；离线 local 无法证明 Cloud 未执行，容易重复产生有费用的 Agent 运行 |
| 允许任务在 local 与 cloud 间迁移切换 | 看似灵活，但两端执行环境、凭据与执行内容完全不同，切换实质是重新绑定执行目标，并引入双 owner 与幂等键漂移风险；复制成新任务、停用旧任务更简单可预期 |
| 为迟到触发提供可配置的补跑窗口 | 表面体贴，但阈值、合并和并发策略都会成为新的决策面；首版选择“10 分钟内照跑、之外记 missed”的最简单可预期行为 |
| Desktop 重启后补跑全部离线 occurrence | 看似完整，但长时间离线会形成无界队列，日报等时效任务也会产生失真结果 |
| Cloud 不可用时自动回退 local | 提高表面成功率，却改变用户选择、授权边界与费用归属，并可能在 Desktop 重开后重复执行 |
| 为三个首发模板分别实现 scheduler | 初期代码直观，但模板行为会分叉，用户自定义任务也无法复用同一运行与审计模型 |

## 本决策未解决的问题

- 首版表单采用受限周期枚举还是公开 cron，以及精确的输入 schema。
- 本地提示词运行的 application use case 如何与 Agent Session 关联（新建会话还是复用会话）；本地 Workflow revision
  目标属于后续扩展。
- 更细的无人值守工具批准策略取值、worktree 的数量上限，以及与现有 Session approval 行为的迁移关系。
- Cloud 首期无头单会话的公开 API、存储、调度、发起者与费用归属、Project/模型/无人值守授权复查、通知与配额，作为
  `decisions/cloud/` 下的独立 ADR 提出，不以 #67 为前置条件。
- Cloud 触发已发布 Workflow 的目标冻结、Start 参数校验及执行器槽位映射，作为第二阶段能力，等待 #67 及 Workflow
  server 执行链落地后另立 ADR。
- 本地与 Cloud 任务数据的互通边界：Desktop 任务列表是否合并展示 Cloud 任务、Cloud 网页端对两类任务与运行历史的
  可见与管理范围、运行历史的跨端展示位置、Cloud 任务的账号与归属（创建者个人还是 Space），以及跨执行位置复制任务
  携带的数据范围。本地侧默认（只存本机、不上传、不要求登录）已冻结为决策，其余随 Cloud 决策一并定案，避免本地数据
  模型做出相反假设。
- 运行失败后的自动重试策略、失败通知与历史保留期。
- 操作系统后台服务、开机自启及显式 catch-up。

## 核心测试用例

按仓库规则，本 ADR 状态变为 `approved` 后在 `test-cases/desktop/core/scheduled-task/` 补充核心用例。至少覆盖：
同一 occurrence 重放只接纳一次；Desktop 关闭期间 local 不执行且重启记录 missed；实际触发晚于 `scheduledAt`
超过 10 分钟记 `missed` 且不执行、10 分钟内迟到照常执行；manual run 不冒充计划触发；运行中到期产生
`skipped_busy`；达到全局并发上限时产生 `skipped_capacity`；本地运行在独立 worktree 中进行且不改动用户当前工作区，
非 Git 目标不能保存为 active；任务编辑冻结旧运行；Cloud 能力不足时不能激活；授权失效明确失败且不自动回退；禁用/删除保留历史；本地任务及其运行
历史不上传、不跨设备同步且不要求登录。
还需覆盖：没有 `full_auto_in_worktree` 显式同意时不接纳运行；仅改提示词或计划时间不要求重新同意、目标项目或策略变更
要求重新同意；策略撤销后任务自动停用且既有运行事实不被改写；固定 60 分钟超时取消运行并记录 `timeout`；无改动 worktree
在运行后删除、有改动 worktree 保留且可从历史打开或丢弃；Cloud 首期 Prompt 复用 IssueRun / Thread 并校验 Project、模型和
无人值守授权，不解析 Workflow 或执行器槽位。第二阶段 Workflow 的槽位缺失时，该次运行失败并通知任务所有者。

## 落地顺序

1. **specs**：本根决策获批，并据批准内容补核心测试用例。
2. **desktop / local definition**：新增 domain、repository 与应用用例，契约生成后接入左侧入口、列表和编辑器；
   Cloud 选项保持可见但不可激活。
3. **desktop / local scheduler**：用后端 scheduler 重建 active local jobs，落实 occurrence 幂等、迟到/离线
   missed、独立 worktree 执行、全局并发上限、`skip_if_running` 和真实 ExecutionTarget adapter；没有真实
   adapter 时不得把模拟运行作为 MVP 完成。
4. **cloud / first phase**：以 `decisions/cloud/` 下的独立 ADR 定义无头 Prompt 单会话的 API、权限与持久化；复用
   IssueRun / Thread 真实执行链并验证后，开放 `cloud` 选择，不以 #60/#67 为前置条件。
5. **cloud / second phase**：#67 及 Workflow server 执行链落地后，另立 ADR 支持 Cloud 触发已发布 Workflow，并按
   任务所有者的个人执行器槽位映射解析执行器。
