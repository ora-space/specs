---
status: implemented
---

# 同一 checkout 允许多个存活 Agent 会话，恢复继续独占；拒绝只终结当前会话

> 历史决策：D2 的停止证据围栏已由[会话 MCP 刷新与宿主健康](./20261116-session-mcp-refresh-and-host-health.md) D5 扩展：Agent/探测的
> stop_unconfirmed 与未排空升级的 legacy_unknown 只挡 prior_revision 恢复和该 execution 自己的交付，
> 普通新会话仍按本决策准入；既有恢复围栏不变。

一个 clone 执行创建的 checkout 可以同时承载多个互相独立的 Agent 会话，行为对齐 Desktop 在同一运行 Workspace 中按节点各建会话的方式。准入只在前序 Revision 恢复、停止确认未知等需要独占 checkout 的情况下拒绝，并以每 checkout 8 个未结束会话为上限。拒绝只把新会话登记为 `agent_failed`，不关闭 Controller 控制连接，也不影响兄弟会话；被拒绝的执行从未运行，也不可交付。Node 以 `shared_checkout_sessions` 能力声明这项行为。

本决策为 `implemented`：owner 于 2026-11-16 批准 D1–D5，Node 侧实现经 desktop#774（合并提交 `c5265072`）合入，审查留下的 4 个测试缺口（并发受理、未清理恢复尝试围栏、已结束恢复阶段、`RLIMIT_NOFILE` 推导）由 desktop#864（合并提交 `89b7ce37`）补齐，最终实现没有偏离 approved 决策；核心测试用例见文末“核心测试用例”。本决策覆盖 desktop 的 `ora-node-db`、`ora-node`、`ora-node-protocol` 及消费能力的 Controller。它不改变消息版本、执行身份、事件序号、JSONL、插件进程隔离或数据库布局，不需要数据库迁移，也不改 Cloud／v2 代码。受控模式的运行时单例不在本次范围内。`checkout_busy`、`checkout_session_limit` 与 `checkout_too_deep` 是稳定的终态 `detail` 代码，不是新的协议消息；交付失败码新增 Node 侧的 `checkout_too_deep`。分离工作负载 UID 的部署新增两项宿主要求：`fs.protected_hardlinks = 1`（Node 启动时检查，并在每次交接遍历前重新读取）和足够的 `RLIMIT_NOFILE`（文档要求；Node 在所有 checkout 上同时至多运行 9 次交接遍历）。

## 继承与修改

历史上没有任何现行 ADR 写下“一个 checkout 只允许一个 Agent 会话”：准入检查来自恢复围栏的代码实现（desktop `934a2cd9`），不是某份 ADR 的决策。本文因此不修改任何 ADR 的条款，只继承现行承诺并新增共享语义；已被取代的[共享进程入口与资源交接](../process/integration/0-shared-process-entry-and-resource-handoff.md) D4 记录了“资源冲突以 clone 目录为单位、交付在会话结束后开始”的历史资源语义，仅作背景引用——该 ADR 整体已被[Node 直接执行自己的 Git](../git-execution/0-node-runs-git-directly-with-attempt-locks.md) 取代，其文本不再随实现同步，本文不恢复其地位。

本决策承接本目录最新的[显式 Session MCP](20261114-explicit-session-mcp-in-agent-session-spec.md)（Node 侧 `implemented`，经 ora-space/specs#289 批准），并经它沿用[子代理投影、历史与诊断](20261020-child-projection-history-and-diagnostics.md)，不改变其 MCP 选择、子代理来源、历史或诊断规则；同时补充[共享运行时根决策](0-shared-agent-runtime-crate-hosted-by-node.md)的每执行宿主隔离。

| 前序约定 | 处理 |
| --- | --- |
| [恢复截止与停止确认](../revision/20261018-complete-restore-deadline-and-cancel-confirmation.md) D6：停止证据未知时不发原会话终态、不交付、不复用 checkout | **继承，不变**：`legacy_unknown`、未清理恢复尝试及未确认停止继续阻断该 checkout 的新会话；本文的立即终态只属于尚未开始、没有恢复副作用的新执行，不能用于解除原恢复的未知责任 |
| [恢复停止确认接受 host 权威的 NotStarted](../revision/20261021-stop-confirmation-accepts-not-started.md)：D6 的停止确认证据扩展为「原 Scope 已 `Closed` 且（该 Run 的直接进程已退出或 host 权威报告 `NotStarted`）」 | **继承，不变**：本决策的围栏（a）沿用这一定义下的 `cleaned` 证据判定，`UnknownIdentity` 仍不构成停止确认，不释放围栏 |
| [流式 Thread 事件与会话命令](../protocol/20260928-streamed-thread-events-and-session-commands.md) D1：握手以能力声明新增行为；执行沿用既有终态与重放 | **继承并新增能力**：增加 `shared_checkout_sessions`；复用 `AgentSessionEnded` 与既有执行账本，不新增消息类型 |
| [显式 Session MCP](20261114-explicit-session-mcp-in-agent-session-spec.md)：MCP Setting 值不得进入 Node ledger（含 SQLite WAL）；持久输入只保留 MCP 身份与版本，重复执行比较不含 Setting 值；`plugin_id`、`setting_id`、`transport` 只出现在 MCP setup 失败终态 | **继承，不变**：被 checkout 拒绝的执行同样只持久化这份去除 Setting 值的投影，投影由受理路径唯一产生并随拒绝交给登记；重发被拒绝的启动按同一投影比较，Setting 值不同也返回原持久终态，不构成身份冲突；拒绝终态省略这三个字段 |
| 每会话执行拥有独立插件生命周期、历史、ora-git socket 与进程树 | **继承，不变**：共享的是 checkout，不是会话身份、进程或历史 |

## D1：同一 checkout 可容纳多个存活会话，但数量有界

不带 `prior_revision` 的新会话在恢复围栏允许时可以进入已有其他未结束会话的 checkout。每个会话仍使用自己的 execution id、插件进程、会话目录、事件序号和终态；结束其中一个不停止兄弟会话，不清理共享 checkout，也不撤销其他会话的插件租约。

分离工作负载用户的部署下，会话启动要把 checkout 交给工作负载用户。**每个会话都在自己的 Agent 启动之前亲自完成这次交接**，不依赖兄弟会话：准入顺序既不说明更早会话的遍历是否已经完成，也不说明它是否执行过（它可能在遍历前就以 `agent_plugin_unavailable` 等原因结束）。遍历以特权身份运行，而存活兄弟 Agent 同时以工作负载用户身份改动这棵树，所以遍历对每个名称只解析一次：根目录以 `O_PATH | O_NOFOLLOW | O_DIRECTORY` 钉住，其下每个条目都相对列出它的目录描述符以 `openat(O_PATH | O_NOFOLLOW)` 钉住；此后检查（`fstat`）、重属主（`fchownat(AT_EMPTY_PATH)`）和目录列举（相对钉住的描述符打开 `.`）都只作用于钉住的 inode。符号链接按链接本身重属主，从不跟随；条目钉住之后，兄弟再把名称换成符号链接或另一个目录，都不改变遍历作用的对象，因此不能借此把重属主引到 checkout 之外；硬链接是钉住解决不了的情形，见下文。钉住之前就消失的条目（`ENOENT`）直接跳过；checkout 根本身必须存在，无权打开、列举或重属主的条目仍使启动失败。交付准备中的重属主步骤使用同一遍历。已经正确的树只花一次遍历。

遍历还有四条边界，它们是交接的一部分，不是实现细节：

- **深度上限 4096。** 遍历在根到当前条目之间每层目录保持一个描述符，而描述符是整个 Node 进程共享的资源；若深度不设限，能在 checkout 里建目录的 Agent 就能决定特权进程持有多少描述符，使同一 checkout 的后续启动和交付以 `EMFILE` 失败，并在每次遍历期间让 Node 的其他工作（控制连接、插件管道、`ora-git` socket、SQLite）一同失败。遍历在打开第 4097 层目录之前以独立错误拒绝，单次遍历因此至多持有 4099 个描述符。Node 把它映射为稳定代码 `checkout_too_deep`：启动以 `agent_failed{checkout_too_deep}` 结束，交付以 `failed{checkout_too_deep}` 失败。4096 远超真实源码树的深度，又让描述符预算可以事先计算。
- **`fs.protected_hardlinks = 1` 是硬链接的保证。** 遍历看到的是 inode，看不到它在别处的名称，所以任何用户态复查都挡不住硬链接进来的外部文件：写入方把外部文件链接进 checkout，在遍历钉住它之后立刻删掉这个名称，`fstat` 读到的链接数就是一，外部 inode 随即被重属主；任何 `fstat`/`statat` 顺序都可以被链接与删除穿插，而 inode 一旦被重属主，写入方又可以再链接它。能阻止这件事的只有宿主拒绝创建这个链接：在这项保护下，工作负载用户只能链接自己拥有、或本来就能读写的普通非 setuid 文件，已安装的 Agent 包与 Node 状态等 root 文件进不了 checkout。设置了 `process.workload_uid` 时，Node 启动前要求 `/proc/sys/fs/protected_hardlinks` 读出 `1`，否则以点名该 sysctl 的错误拒绝启动；该 sysctl 是全局宿主状态、不随容器命名空间隔离，内核默认值为 0，启动后仍可能被改动，所以每次启动遍历和交付遍历之前都重新读取，读到 `1` 以外的值时拒绝该次交接：启动以 `agent_failed{agent_start_failed}` 结束，交付以 `failed{snapshot_failed}` 失败，日志点名该 sysctl。复用这两个既有代码，因为它们本来就表示“部署侧原因使启动／快照无法进行”，新增代码不会给调用方带来不同的处理。
- **留在原处的硬链接不重属主（纵深防御）。** 链接数大于一、且尚不属于工作负载用户的非目录条目保持原属主：它的另一个名称可能在 checkout 之外。这只挡住留在原处的链接，不是保证（见上一条）。只被 checkout 命名（链接数为一）的 inode 照常重属主；checkout 内正当互为硬链接的文件因此保留旧属主及权限位允许的访问，直到只剩一个名称。Git 不在工作区创建硬链接，本地克隆硬链接的对象本来就属于执行克隆的工作负载用户。
- **Node 级遍历上限 9。** 工作负载用户能写的每个 checkout 都可能被它弄深，而 Node 不限制跨 checkout 的会话数，所以并发遍历必须按整个 Node 限制，而不是按 checkout。启动遍历与交付遍历共用一个进程级信号量，共 9 个许可（一个 checkout 可有的 8 次启动遍历，加上 Node 同时只准备的一个交付）；更多遍历等待许可，代价只是深树上的延迟，已经正确的树遍历很快。

分离 UID 的 Node 需要的软 `RLIMIT_NOFILE` 据此推算：Node 自身约需 1,024 个描述符，9 次并发遍历至多 9 × 4,099，合计 ≈ 38,000，无论有多少 checkout 很深；部署至少给 38,000（推荐 65,536）。Node 不自行调整这一限制，由部署提供并写入 Node 文档。

Node 定义命名常量 `MAX_SESSIONS_PER_CHECKOUT = 8`。某 checkout 已有 8 个未结束会话时，第 9 个新会话被拒绝；一个会话写入终态后释放一个名额，不同 checkout 分别计数。8 是防止单个共享目录无界累积插件进程的保守资源上限，同时覆盖交互节点等待时并行分支继续运行的常见形状；它不是模型并发额度、全 Node 进程上限或调度公平性承诺。上限保持 Node 本地常量并写入文档，不扩展 `Accepted` 字段。

不变量：

1. 围栏与数量在持久化受理的同一 `IMMEDIATE` 事务内判定并写入，并发受理路径也不能依据已被另一方改变的计数准入。
2. 已结束会话不占名额；持久投影相同的输入重发只返回原持久状态，不重复计数或启动。
3. 会话 A 的终态、插件退出和会话目录清理不改变会话 B 的进程、账本或 checkout 归属。
4. 每个会话的 Agent 启动前，该会话自己的交接遍历已经完成；任何以特权运行的重属主遍历都只作用于它钉住的 inode，不重新解析 checkout 内的路径，所以兄弟并发删除、替换或换成符号链接、另一个目录的条目不能让交接越出 checkout。外部文件经硬链接进入 checkout 只由宿主的 `fs.protected_hardlinks = 1` 阻止，每次遍历之前都确认它仍然生效；遍历按链接数跳过只挡住留在原处的链接。兄弟的并发改动不能让交接失败，唯一例外是把目录嵌套到超过 4096 层：此时交接以 `checkout_too_deep` 失败，而不是让 Node 进程耗尽描述符；一次遍历至多持有 4099 个描述符，整个 Node 至多 9 次遍历同时进行。
5. 本决策不建立全 Node 会话上限；既有的其他资源围栏继续有效。

## D2：恢复需要独占，普通共享不得绕过停止证据

对新会话 S 和目标 checkout C，按以下顺序判定：

| 条件 | 结果 |
| --- | --- |
| C 上任何执行有 `legacy_unknown` 恢复、未清理恢复尝试或未确认恢复停止 | 以 `checkout_busy` 拒绝 S，无论这些执行是否已有业务终态 |
| S 带 `prior_revision`，且 C 上有任何未结束会话 | 以 `checkout_busy` 拒绝 S；恢复会在其他 Agent 仍写树时强制移动分支与工作区 |
| C 上未结束会话带 `prior_revision`，其恢复阶段尚未到达 `agent_admitted` | 以 `checkout_busy` 拒绝 S；恢复会话在 Agent 获准启动之前仍独自准备 checkout |
| C 上已有 8 个未结束会话 | 以 `checkout_session_limit` 拒绝 S |
| 其他情况 | 受理 S |

`pending`、`git_admitted` 与 `restored` 都是 Agent 尚未获准启动的恢复阶段：`restored` 只表示恢复 Git 已完成并确认停止，恢复会话此后仍要准备会话目录与模型访问，之后才进入 `agent_admitted`。普通会话要等到 `agent_admitted` 才能共享；`agent_admitted` 之后只要原恢复会话未结束，另一个带 `prior_revision` 的会话仍被拒绝。已结束历史中的任何恢复阶段（含 `pending`）不构成围栏。未清理 `restore_process_attempts` 和 `legacy_unknown` 独立检查，不因原执行状态或停止等待上限到期而释放。`checkout_busy` 的判定先于数量上限。

受控模式的运行时许可检查先于 checkout 分类：受控启动保持“Node 上存在任何未结束执行即以 `ResourceConflict` 拒绝并按既有路径关闭连接”的单例契约，checkout 拒绝只对裸启动生效。

## D3：checkout 拒绝是当前执行的持久终态，不是连接错误

命中 D2 前三类恢复／独占条件时使用稳定代码 `checkout_busy`，命中数量上限时使用 `checkout_session_limit`。Node 必须：

1. 保留这次启动的完整执行身份与输入的持久投影：与受理路径相同、去除 MCP Setting 值而保留 MCP 身份与版本的那一份，原始控制帧不进入 ledger；
2. 在任何插件、恢复或 checkout 写入之前，把该执行直接登记为 Completed；
3. 在同一事务中写入序号 1 的 `AgentSessionEnded { reason: agent_failed, detail }`，使其可查询、重放和确认；该终态省略只属于 MCP setup 失败的 `plugin_id`、`setting_id` 与 `transport`；
4. 返回现有 `ExecutionStatus`，不启动 actor，不新增拒绝消息。

Node DB 以专门的存储错误类型区分 `checkout_busy` 与 `checkout_session_limit`，不得复用 `ResourceConflict`。服务层只捕获这两种预期拒绝并形成上述终态；真正的身份、协议、运行时许可和其他资源冲突仍按原路径处理。预期拒绝不会到达 `close_reason`，因此控制连接保持打开，所有兄弟会话继续接受 `submit_user_turn` 与 `end_session`。若持久化终态本身失败，则仍是 Node 内部错误，不能伪称拒绝已可靠记录。

拒绝只发生在裸启动路径：受控启动永远先被运行时单例以 `ResourceConflict` 拒绝，其“关闭连接”的既有契约不变。被拒绝的执行没有 Agent 轮次、恢复副作用或会话历史；其 `agent_failed` 不表示原兄弟会话失败，也不改变原恢复的 D6 终态规则。

被拒绝的执行不可交付：`DeliverRevision` 命名它时以 `session_not_settled` 失败。复用该代码而不是新增失败码，语义上同为“该会话从未进入可快照的交付状态”——被拒绝的执行没有任何可交付成果，共享 checkout 上的存活兄弟写入绝不能被快照成它的 Revision。

D1 的 `checkout_too_deep` 则是新的 Node 交付失败码：它说明的是 checkout 本身无法交接，与会话状态无关，混进 `snapshot_failed` 会让 Node 账本和日志无法区分这种由 Agent 造成的拒绝与 Git 快照故障。固定的 Cloud 契约（`RevisionFailureReason`）没有对应原因，所以 Controller 在 Cloud 契约补充之前把它映射为 `REVISION_FAILURE_REASON_SNAPSHOT_FAILED`——对 Cloud 而言，checkout 确实没有被快照。

不变量：

1. 一次 checkout 拒绝只终结新 execution，连接与其他 execution 保持可用。
2. 拒绝终态和事件原子提交；重发持久投影相同的输入（Setting 值不参与比较）只返回原终态，不会再次产生事件、启动 Agent 或关闭连接。
3. `ResourceConflict` 不承载这两种预期 checkout 拒绝；受控启动不进入 checkout 拒绝路径。
4. 没有插件进程、恢复尝试或 checkout 修改可以先于拒绝终态发生。
5. 被拒绝的执行没有会话历史文件，其交付以 `session_not_settled` 拒绝。

## D4：能力声明语义，不承诺 Rust Controller 混合版本解析未知枚举

Agent 功能启用时，Node 在 `HelloAccepted.capabilities` 中同时声明 `agent_session` 与 `shared_checkout_sessions`，线上值为 `shared_checkout_sessions`。能力只表示“同一 checkout 可共享及 D2/D3 语义”，不替代 `revision_restore`，也不表示受控模式单例已取消。

当前 `ora-controller` 对能力使用 `contains` 查询并保留完整列表，没有穷举 match，因此同版本 Controller 无需新增业务分支即可容忍此值。旧版 Rust Controller 使用同一封闭 Serde 枚举解析 `HelloAccepted`，遇到未知 `shared_checkout_sessions` 会解码失败，**不能**与新 Node 混合版本握手；交付结果中的新失败码 `checkout_too_deep` 同理。这符合现有“Controller 与 Node 必须同版本”的部署边界。Cloud v2 的 Go 客户端（ora-space/cloud `v2-server/internal/node/client.go`）把能力解析为字符串列表，接受未知值，因此 v2 升级锁定的 Node 不受此边界影响。若未来要支持 Rust Controller 混合版本，需要另立未知能力解析决策，不能在本文暗中改变协议。

## D5：受控单例、交付与定时产物的边界

受控模式继续执行“一个 Node 同时只有一个未结束执行”的运行时许可规则；本决策只解除同一 checkout 的额外单会话限制，不把 Cloud 受控运行改为多会话。该单例及其 runtime permit 不在本次落地范围，也不需要 schema 变化。新规则只查询已有 `node_executions.input`、`session_restores` 与 `restore_process_attempts`。

Revision 交付仍要求所属会话已经结束，但它快照的是整个 checkout；兄弟会话若仍存活，其未提交写入和 HEAD 变化可能进入该交付。定时产物同样在所属会话结束时从 checkout 读取，因此可能包含兄弟写入。本决策明确接受这种共享 Workspace 语义，不把快照或产物伪装成单会话隔离成果；调用方需要隔离成果时必须选择独立 checkout。当前 v2 不使用 Node Revision delivery，本次也不改 v2。

## 为什么不是这些替代方案

| 方案 | 未采用原因 |
| --- | --- |
| 每个节点一个 checkout，或为每个节点创建 worktree | 可提供文件隔离，但违背共享运行 Workspace 规则，偏离 Desktop 的节点会话模型，也增加 clone／worktree 生命周期 |
| 交互节点等待时把其他 Agent 串行排队 | 避免并发写入，却与 Desktop 同 Workspace 并行执行不一致，交互等待会无界阻塞兄弟分支 |
| 新增 typed rejection 消息 | 拒绝分类更显式，但扩大 Controller／Cloud 协议与重试处理；现有执行终态、查询和重放已经能承载稳定代码 |
| 继续以 `ResourceConflict` 关闭控制连接 | 实现最小，但把单个新会话的准入失败扩散为所有兄弟会话传输失败，违反故障隔离目标 |
| 不设置每 checkout 上限 | 对齐 Desktop 的无上限形状，但 Node 每会话有独立插件进程，共享目录可成为无界资源放大点 |
| 遍历不设深度上限，下降时关闭父目录描述符、回溯时经 `..` 重开并校验 `st_dev`/`st_ino` | 描述符占用恒定，但回溯要重新解析 `..`，而兄弟可以同时把目录移到别处，校验失败后还需重新定位与重试，正好回到 D1 要消除的路径重解析；深度上限更简单，且让 `RLIMIT_NOFILE` 可以事先推算 |
| 只在启动时检查 `fs.protected_hardlinks` | 少一次 `/proc` 读取，但该 sysctl 是全局宿主状态，启动后可能被改回 0；那时链接数跳过可被“钉住后删名”的竞争击败（实测约五次遍历赢一次），所以每次遍历前都重读，把运行时改动变成被拒绝的启动／交付 |
| 不按链接数跳过，只依赖 `fs.protected_hardlinks` | 规则更少，但在保护生效时，工作负载用户仍可链接自己能读写的外部文件并让其留在原处，遍历会把这类文件的属主（及组）交给工作负载用户；跳过对留在原处的链接成本为零 |
| 按 checkout 而不是按 Node 限制并发遍历 | 与每 checkout 8 个会话的上限对齐，但工作负载用户能把任意多个 checkout 弄深，描述符总量就随 checkout 数增长，文档中的 `RLIMIT_NOFILE` 不再是上限 |

## 风险、兼容性与落地顺序

共享 checkout 允许 Agent 写入互相可见，文件级冲突、同一分支推送竞争和交付混入兄弟写入都是显式代价；现有 Git 非快进冲突与会话独立进程仍提供各自边界。本决策不承诺按节点归属文件变更，不新增回滚、锁或合并。

重属主遍历的剩余边界：描述符是进程级资源，遍历把单次占用限制在 4099 个、整个 Node 并发遍历限制在 9 次，总量仍要由部署按 D1 的 `RLIMIT_NOFILE` 推算提供；低于该值时，深 checkout 上的遍历仍可能让 Node 其他工作暂时遇到 `EMFILE`。硬链接的保证只是宿主的 `fs.protected_hardlinks = 1`，Node 在启动时和每次遍历前确认它；在两次读取之间被改为 0 的宿主仍留下一个窗口，这超出 Node 能控制的范围。在保护生效时，工作负载用户仍可让遍历重属主它本来就能读写的外部文件（用“钉住后删名”的竞争，或在其可写目录中改名移入），于是获得这些文件的属主身份，原属组失去组访问；它够不着 root 拥有的包与状态文件。遍历的总耗时仍不设上限：持续向尚未列举的目录添加条目的写入方可以拖长遍历，这是遍历本身的既有性质，不在本次范围。

落地：

1. 显式 Session MCP（desktop#773，`98c0414f`）先于本决策独立合入，两者的准入与能力变化分开评审。
2. Node 侧实现经 desktop#774（合并提交 `c5265072`）合入：Node DB 准入与原子拒绝终态（只持久化 MCP 无值投影）、`ora-utils` 中钉住条目、有深度上限的不跟随重属主遍历、Node 级遍历信号量及宿主 sysctl 检查（启动时与每次遍历前）、服务能力、协议声明、双语 Node 文档和直接行为测试，不附带 Cloud 改动；审查留下的 4 个测试缺口由 desktop#864（合并提交 `89b7ce37`）只补测试补齐。核心用例按这两个提交回填证据，无 `Missing`。
3. v2 升级锁定的 Node 并按能力启用同 run 多会话，以及 v2 的 predecessor 结束逻辑、Node pin 和交互工作流验收，由 v2 事项承接，不计入本决策的实现状态。

保持不变：执行身份、事件序号、命令顺序、插件与历史隔离、恢复停止证据、受控运行时单例、数据库布局。开始改变：非受控 Node 可在同一 checkout 受理最多 8 个普通存活会话；checkout 拒绝不再关闭控制连接，且被拒绝的执行以 `session_not_settled` 不可交付；每个会话在自己的 Agent 启动前亲自把 checkout 交给工作负载用户，特权遍历钉住每个条目、不重新解析路径、不重属主留在原处的硬链接文件，目录深于 4096 层时以 `checkout_too_deep` 拒绝，整个 Node 同时至多 9 次遍历；分离 UID 部署要求 `fs.protected_hardlinks = 1`（启动时与每次遍历前检查）并按文档提供 `RLIMIT_NOFILE`；被拒绝的执行同样不持久化 MCP Setting 值。

## 核心测试用例

见[共享 checkout 会话用例](../../../test-cases/node/agent-runtime/shared-checkout-sessions.md)：同一 checkout
的多个存活会话与独立结束、每 checkout 8 个会话上限、恢复独占与停止证据围栏、拒绝只终结新执行、每个会话
亲自完成的钉住式不跟随交接、4096 层深度上限、`fs.protected_hardlinks` 检查、Node 级 9 次并发遍历与
`RLIMIT_NOFILE` 要求，以及 `shared_checkout_sessions` 能力与同版本 Controller 边界。37 项验证义务按 desktop#774
（`c5265072`）与 desktop#864（`89b7ce37`）回填证据，全部 `Covered`；其中 `RLIMIT_NOFILE` 是文档义务，测试只固定
文档最低值由遍历上限推导，不证明部署提供了该限制。[会话执行用例](../../../test-cases/node/agent-runtime/session-execution.md)、
[显式 Session MCP 用例](../../../test-cases/node/agent-runtime/session-mcp.md)与
[恢复截止与停止确认用例](../../../test-cases/node/revision/restore-deadline-and-stop-confirmation.md)的证据
保持各自范围，不推导本文新义务。
