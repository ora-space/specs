# Node 核心测试用例

本目录与 `specs/decisions/node/` 镜像。Node 各能力的根决策进入 `approved` 后，再在对应子目录
登记需要长期维护的跨进程、外部副作用、ownership、重试和恢复用例。

## Node 直接执行 Git

[Node 直接执行 Git 决策](../../decisions/node/git-execution/0-node-runs-git-directly-with-attempt-locks.md)为
`implemented`（Linux）；核心用例的证据按最终实现核对，缺口逐项写在用例里。它取代了 host／guardian 体系，
[process](process/README.md) 下只保留 containment 的用例，它们原有的证据随被取代的代码一起删除。

| 领域 | 核心用例 | 重点风险 |
| --- | --- | --- |
| git-execution | [尝试锁、停止证据与重启收口](git-execution/attempt-locks-and-stop-evidence.md) | 用超时或"查不到"推断已停止、锁泄漏给无关进程、后代仍在写入时上报结果、按过期线索误杀、Node 死后 Git 无人终止、重启后未收口就恢复、升级后旧尝试卡住、clone 留下常驻进程 |

## Git 原生代码链路（内部线）

[Node 从快照 clone 决策](../../decisions/node/repository/20261016-clone-from-git-server-snapshot-and-ora-git.md)为
`implemented`，repository 一行的用例按现有测试登记证据并保留缺口。
[Node 交付推送与 Git 恢复决策](../../decisions/node/revision/20261017-push-delivery-ref-and-restore-from-git-server.md)为
`implemented`，revision 一行的用例按现有测试登记证据并保留缺口。

| 领域 | 核心用例 | 重点风险 |
| --- | --- | --- |
| repository | [快照 clone 与 `ora-git`](repository/snapshot-clone-and-ora-git.md) | 访问外部仓库、checkout 留下 remote、带证书的 Git 读到 checkout 配置、拉取次数无界、冒充会话、推到别人的分支 |
| revision | [Git 交付与恢复](revision/git-delivery-push-and-restore.md) | 交付 ref 未写入就成功、`403` 被重试、快进失败拖垮交付、恢复访问外部、`401` 让前序永久不可续、缺证书仍启动 |

## 已批准的恢复截止、取消与停止确认

[Node 恢复截止与停止确认决策](../../decisions/node/revision/20261018-complete-restore-deadline-and-cancel-confirmation.md)经 owner 批准为
`approved`。恢复已改为从 Git Server 拉取前序交付 ref，bundle 恢复不再存在；完整截止、取消与停止确认的实现尚未合入 desktop 的 main，
这些义务目前都是 Missing。内部线恢复的证据在[Git 交付与恢复](revision/git-delivery-push-and-restore.md)，不推导本文的新义务。

| 领域 | 核心用例 | 重点风险 |
| --- | --- | --- |
| revision | [恢复截止、取消与停止确认](revision/restore-deadline-and-stop-confirmation.md) | 截止被重置或命令越过截止、执行预算/停止请求/停止确认混同、未知停止证据下交付或复用、上限被重启重置、取消中途打断 Git、启动清理越过原执行交接、BestEffort 当作 Strong |

## 插件模型 adapter

[插件模型 adapter 决策](../../decisions/node/agent-runtime/20261017-plugin-owned-model-adapters.md)于 2026-10-17 经 owner
批准，已实现，状态为 `implemented`。义务多为 Partial：Node 侧测试用回显 fixture 代替真实 CLI，缺口逐项写在
用例里；[既有模型代理用例](agent-runtime/model-proxy.md)的证据保持原范围。

| 领域 | 核心用例 | 重点风险 |
| --- | --- | --- |
| agent-runtime | [插件模型 adapter](agent-runtime/plugin-model-adapters.md) | 首轮前准备失败残留授权、上下文泄入账本/日志、CLI 高优先级配置改路由、共享 provider 状态、别名与发现越权、收尾与撤销不收敛 |

## 子代理投影与运行诊断

[子代理投影来源决策](../../decisions/node/agent-runtime/20261020-child-projection-history-and-diagnostics.md)为
`implemented`，核心用例按现有测试登记证据并保留缺口；
[CodeAgent _subagent/update 子代理进度通道决策](../../decisions/node/agent-runtime/20261118-codeagent-subagent-update-channel.md)为
`implemented`，其新增的 11 项义务按 [desktop#783](https://github.com/ora-space/desktop/pull/783)（`1211708d`）登记，
并补入 [desktop#800](https://github.com/ora-space/desktop/pull/800)（合并提交 `217942ec`）的 `session/load` 回放隔离证据。
现有 4 项 `Covered`、7 项 `Partial`，无 `Missing`。剩余缺口逐项保留在用例里。
[会话执行用例](agent-runtime/session-execution.md)与[静默策略用例](../desktop/core/workflow/prompt-inactivity.md)的
证据保持原范围，不推导来源保真或终态缺口已实现。

| 领域 | 核心用例 | 重点风险 |
| --- | --- | --- |
| agent-runtime | [子代理投影、终态缺口与运行诊断](agent-runtime/child-projection-and-diagnostics.md) | 父子内容互串或按 depth 猜归属、同 messageId 跨来源拼接、父回合结束把未知结局伪装成终态、忽略已返回的委托或把委托返回当子组终态、子活动不当续期或暂停父提示、把重放时间当实时活动、只读详情崩溃或长出控制入口、`_subagent/update` 乱序/内层工具 ID 碰撞/越权改写父会话状态或污染工作流输出 |

## 显式 Session MCP

[显式 Session MCP 决策](../../decisions/node/agent-runtime/20261114-explicit-session-mcp-in-agent-session-spec.md)的
Node 侧为 `implemented`，证据按 desktop 固定合并提交 [`98c0414f`](https://github.com/ora-space/desktop/commit/98c0414fdaf998ae7ede817f743e3431d94c81d7)（desktop#773）
登记，共 20 项义务（13 项 `Covered`、7 项 `Partial`，无 `Missing`）。7 项 `Partial` 是：选择规则在接收边界与
真实控制连接关闭上的缺口（`session_mcp_rejects_ambiguous_or_unbounded_selection`）、`NodeSessionMcp::prepare`
入口的负例路径、真实 `session/new` 未省略字段重测声明默认值、非空 MCP 崩溃重启结算 `interrupted`、
无 Agent 配置时不广告 `SessionMcp` 的反向用例、能力失败路径自身的进程清理与租约释放、运行期间
`RemovePlugins` 对 MCP 包返回 `plugin_in_use`。
Controller 侧的发送前镜像校验与按能力派发在 [Controller 用例](../controller/session/session-mcp-selection.md)登记，本文不登记。
[会话执行用例](agent-runtime/session-execution.md)的证据保持原范围，不推导本文的新义务。

| 领域 | 核心用例 | 重点风险 |
| --- | --- | --- |
| agent-runtime | [显式 Session MCP](agent-runtime/session-mcp.md) | 空选择被当成全量或自动发现、乱序重复非规范选择被受理、部分快照照常交付、Node 本地路径或 Controller 命令进入 setup、Setting 值进入账本/JSONL/Thread/日志/错误/Debug、重发改写运行中的值、能力缺失仍发 session/new、失败终态或日志携带值、租约提前释放、Separate 工作负载拿到不可执行命令、共享解析器使 Desktop 行为漂移 |

## 同一 checkout 的多个 Agent 会话

[共享 checkout 会话决策](../../decisions/node/agent-runtime/20261115-shared-checkout-agent-sessions.md)于
Node 侧已实现为 `implemented`：原测试随 desktop#774（`c5265072`）合入，4 个测试缺口由 desktop#864（`89b7ce37`）只补测试补齐。
37 项验证义务全部 `Covered`，证据链接固定到这两个合并提交；`RLIMIT_NOFILE` 是文档义务（测试只固定文档最低值由遍历上限推导，
不证明部署提供了该限制），混合版本边界是部署边界、不登记测试；不把 Node 合入外推为 v2 按能力启用同 run 多会话已完成。
[会话执行用例](agent-runtime/session-execution.md)、[显式 Session MCP 用例](agent-runtime/session-mcp.md)与
[恢复截止与停止确认用例](revision/restore-deadline-and-stop-confirmation.md)的证据保持原范围，不推导本文的新义务。

| 领域 | 核心用例 | 重点风险 |
| --- | --- | --- |
| agent-runtime | [同一 checkout 的多个 Agent 会话](agent-runtime/shared-checkout-sessions.md) | 结束一个会话波及兄弟、共享目录无界累积会话、恢复在存活兄弟下移动工作区或未知停止证据被绕过、拒绝关闭控制连接或可交付、重发产生第二个终态、拒绝路径持久化 Setting 值、依赖兄弟的交接或特权遍历越出 checkout、深树耗尽描述符、硬链接保护关闭仍遍历、跨 checkout 并发遍历无界、能力缺失或混合版本边界被隐藏 |

## 会话 MCP 刷新与宿主健康

[会话 MCP 刷新与宿主健康决策](../../decisions/node/agent-runtime/20261116-session-mcp-refresh-and-host-health.md)于
Node 侧已实现为 `implemented`。18 条核心用例分别维护直接证据与缺口，Node 侧验收 4、5、6、8、12 为 Passed；
不把 Node 合入或验收通过外推为 Controller/v2 全链完成，也不把默认 ignored 的 root 用例算作 CI 通过。
[显式 Session MCP 用例](agent-runtime/session-mcp.md)、[共享 checkout 会话用例](agent-runtime/shared-checkout-sessions.md)与
[尝试锁用例](git-execution/attempt-locks-and-stop-evidence.md)的证据保持原范围，不推导本文的新义务。
[会话尝试环境标记决策](../../decisions/node/agent-runtime/20261120-attempt-mark-in-session-stop-evidence.md)为
`implemented`，在同一文件新增“会话停止杀掉全部带标记进程”用例，并在另外六个用例中补充义务；19 项义务按
[desktop#802](https://github.com/ora-space/desktop/pull/802) 的 head `198cc415` 登记直接证据（7 项 `Covered`、10 项 `Partial`、
2 项 `Missing`），缺口逐项保留在用例里。其中“轮次之间与保留会话不杀常驻进程”一项随下文轮次进程栅栏决策的实现，按
desktop#807 `447a59de` 改登记为 `Covered`，19 项现为 8 项 `Covered`、10 项 `Partial`、1 项 `Missing`。

| 领域 | 核心用例 | 重点风险 |
| --- | --- | --- |
| agent-runtime | [会话 MCP 刷新与宿主健康](agent-runtime/session-mcp-lifecycle.md) | 刷新扩大白名单或发送部分列表、旧 generation 回滚目标、Setting 值进入账本/日志/停止记录、准入前结算 Executed 或 MCP 门禁结束 execution、受阻轮次无界等待、刷新失败后以 session/new 重建、同一进程 load 无界累积或升级不重启、在用包版本被删、宿主子进程不持锁被判已停止、脱组后台作业活过会话结束或被判已停止、轮次之间杀掉常驻进程、跨主机凭 boot_id 确认停止、残留挡住普通后继或放行恢复、未排空升级起不来或旧会话被直接结算、无证据的运维释放、排队探测被判超时或清理被遗弃、HTTP 探测随跳转或代理外发秘密、三层状态互相冒充 |

## Node 承载的宿主 MCP

[Controller 交付宿主 MCP 决策](../../decisions/node/agent-runtime/20261118-host-mcp-servers-in-agent-session-spec.md)的
Node 侧为 `implemented`，证据按 desktop 固定合并提交登记，受理后崩溃重启与 `mcp_lifecycle` 的 load 要求两项为
Partial。只能由 Controller 验证的义务在
[Controller 侧用例](../controller/session/host-mcp-servers.md)维护；[显式 Session MCP 用例](agent-runtime/session-mcp.md)与
[会话 MCP 刷新与宿主健康用例](agent-runtime/session-mcp-lifecycle.md)的证据保持原范围。

| 领域 | 核心用例 | 重点风险 |
| --- | --- | --- |
| agent-runtime | [Node 承载的宿主 MCP](agent-runtime/host-mcp-servers.md) | 两端 URL 判断不一致关闭控制连接、非字面 loopback 上的明文 http、上限口径不清、某次 setup 漏带或改写宿主 Server、宿主 Server 混入插件状态或 setup 窗口、重发改写在用的值、URL/请求头进入账本/日志/解码错误、持久行解码失败、能力缺失仍发 session/new、宿主门禁改写插件失败、Node 探测宿主 URL |

## 取消一轮与一轮结束后保留会话

[取消一轮决策](../../decisions/node/protocol/20261116-cancel-turn-and-keep-session-after-turn-ends.md)为 `implemented`，
由 desktop `2fc5e2ca` 与 `447a59de` 实现 Node 一侧。30 条核心用例与决策的 30 条验证义务一一对应，按 desktop main `447a59de`
登记为 3 项 `Covered`、25 项 `Partial`、2 项 `Missing`。会话命令与事件的既有规则见
[Agent 运行的协议消息用例](protocol/agent-run-messages.md)，结束会话时的停止证据见
[会话 MCP 刷新与宿主健康用例](agent-runtime/session-mcp-lifecycle.md)，它们的证据保持原范围。用例 11、12、13
另含[会话尝试环境标记决策](../../decisions/node/agent-runtime/20261120-attempt-mark-in-session-stop-evidence.md)
带来的义务，按 desktop#802 `198cc415` 登记时为 2 项 `Partial`、1 项 `Missing`；其中用例 13 的那一项（保留会话的取消不杀
任何进程，`Missing`）与下文的轮次进程栅栏决策相反，已被“13（轮次进程栅栏）”一行取代，后者现为 `Partial`。用例 10、13、19 另含轮次进程栅栏决策带来的义务，
按 desktop#807 `447a59de` 登记为 10、19 `Covered`，13 `Partial`。用例 6、18、19、20、30 另含下文失败原因决策带来的义务，
均为 `Missing`。

| 领域 | 核心用例 | 重点风险 |
| --- | --- | --- |
| protocol | [取消一轮与一轮结束后保留会话](protocol/cancel-turn-and-keep-session.md) | 取消没有结论或结论晚于期限、取消误伤下一轮、上一轮的迟到更新混进下一轮、没有停止证据就写终态或放行交付、MCP 门禁中取消与超时各记一个结果、重复取消拖过期限、重启丢失原结束决定、账本不可写时把内存状态当作结论、旧 Controller 握手失败或接管新会话、只选取消的会话在失败后被保留 |

## 轮次进程栅栏

[保留会话前先停掉这一轮新起的进程决策](../../decisions/node/protocol/20261123-turn-process-fence-before-keeping-session.md)为
`implemented`，由 [desktop#807](https://github.com/ora-space/desktop/pull/807) 实现。8 个核心用例的 21 项义务按合并提交 `447a59de`
登记为 11 项 `Covered`、9 项 `Partial`、1 项 `Missing`。取消一轮用例与
[会话 MCP 刷新与宿主健康用例](agent-runtime/session-mcp-lifecycle.md)的证据保持原范围，不推导本文的新义务。

| 领域 | 核心用例 | 重点风险 |
| --- | --- | --- |
| protocol | [轮次进程栅栏](protocol/turn-process-fence.md) | 保留会话时本轮后台作业仍在写 checkout、下一轮早于栅栏派发、杀掉懒启动的 LSP/MCP 或轮次开始前的进程、清空环境的子进程逃出栅栏、判断不清仍保留会话、栅栏位置错误让杀进程后的反应混进下一轮、栅栏之后豁免进程静默新起的进程活过 `stopped`、结论超出 G + F + Q + R、正常完成也杀进程、重启后本轮进程无人清理、伪造标记借栅栏杀别人的进程 |

## 一轮失败的原因

[一轮的失败与取消分开记录决策](../../decisions/node/protocol/20261124-turn-failure-reason-distinct-from-cancel.md)为
`approved`。决策尚未实现，15 条义务均为 `Missing`。取消一轮用例与轮次进程栅栏用例的证据保持原范围，
不推导本文的新义务。

| 领域 | 核心用例 | 重点风险 |
| --- | --- | --- |
| protocol | [一轮失败的原因与失败后的会话结论](protocol/turn-failure-reason.md) | 失败被记成取消或取消被记成失败、ACP 错误码与 Agent 文字丢失、文字未清洗或超长或进入码与日志、看门狗被回 `end_turn` 时重发或不说明原因、看门狗无回应与 Agent 进程退出的 detail 与 D3 表不符、失败后会话保留却没有结论或结论顺序错乱、不能保留时原失败原因被覆盖、未协商的 Controller 收到新事件或选择失败后保留、新字段让旧读者拒绝历史、重启后补出与事实不符的结论 |
