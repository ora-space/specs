---
status: implemented
---

# ACP Agent 运行时抽成共享 crate `ora-agent-runtime`，由 Desktop Backend 与 Node 各自提供适配器；Node 以会话执行运行 Agent 插件

Desktop Backend 中的 ACP Agent 运行时（`crates/backend/src/agent_runtime`，约一万行：连接监管、会话 actor、
路由与背压、prompt 无活动重试、会话记录、切换与 handoff）抽成独立 crate `ora-agent-runtime`，Backend 专属
的依赖（SQLite 仓储、应用事件、插件 API、Session MCP 解析、Task/Workspace 目录解析、设置）改为该 crate
定义的接口，由 Backend 实现，Desktop 行为不变。Node 以同一 crate 运行 Agent 会话执行：只启动本次会话需要
的 agent 插件，以 checkout 为 `cwd`，以环境变量注入 git 身份，把每条定型记录同时写入会话 JSONL 与 Thread
事件。

> 历史决策：D5 中“模型凭据由沙盒部署以环境变量提供”已被[运行范围的模型代理](20261010-run-scoped-model-proxy.md)
> 修改：模型绑定的会话经 Node 的模型代理访问平台代理，真实 Key 不进入沙盒。其余规则不变。

> 历史决策：D6“Agent 轮次失败、超时、连接丢失”一行的“会话保持”已被
> [取消一轮并在轮次结束后保留会话](../protocol/20261116-cancel-turn-and-keep-session-after-turn-ends.md)（`implemented`）
> 修改：连接丢失结束会话；失败与超时只在会话选择了 `keep_session_on_failure` 且满足该决策 D2 时保留。其余规则不变。

> 历史决策：D6 中“按运行时既有规则记录 `TurnEnded`”与兼容性边界中“会话 JSONL 格式（`ora-history` 所有）不变”，由
> [一轮的失败与取消分开记录](../protocol/20261124-turn-failure-reason-distinct-from-cancel.md)（`approved`）修改：`TurnEnded`
> 增加可选字段 `ended_by` 与 `failure`（失败原因码、ACP 错误码与清洗后至多 1 KiB 的 Agent 文字），`stop_reason`、
> `SCHEMA_VERSION` 与文件布局不变，旧记录照常读取。这些部分以该决策为准。
> 其余规则不变。

本 ADR 处于 `implemented`：共享运行时在 desktop 的 `crates/agent-runtime`，Node 侧会话执行在 `apps/ora-node/src/session/`
（D7 的接口定义在 `session/ports.rs`），验证见 `test-cases/node/agent-runtime/` 下的用例。它是 `node/agent-runtime` 的根决策（本目录范围：“Agent 进程、ACP 转发、会话连接和
运行恢复”）。协议消息见 [Node 协议后续决策](../protocol/20260928-streamed-thread-events-and-session-commands.md)；
插件安装位置见 [plugin-runtime 根决策](../plugin-runtime/0-node-installs-plugins-into-workspace-home.md)；
git 身份来源见 [Cloud git 身份决策](../../cloud/identity-access/20260928-user-git-commit-identity.md)。
Desktop 侧已有的 Session MCP 决策（`desktop/core/agent/session/mcp/`）继续约束 Backend 适配器，本文不改变。
兼容性边界：Desktop 的 SQLite 数据、会话 JSONL 格式（`ora-history` 所有）与前端契约不变；本地文件布局不变。

## 背景

- Agent 运行时的全部行为（见 desktop `docs/agent-runtime.md`）已经在 Desktop 中验证，Node 需要的是同一套
  ACP 会话语义：一个 Agent 插件进程、会话 actor、prompt 重试、记录与封口。
- 运行时对 Backend 的耦合集中在少数边界：`ora_db::SqliteSessionRepository`、`AppEventPublisher`、
  `PluginApi::attach_agent`、`session_setup`（MCP Snapshot）、`task::resolve_workspace_cwd`、设置读取。
- Node 目前只运行 Git，没有插件生命周期与 Deno 运行时；Node 镜像也没有 Deno。

## 决策

### D1：`ora-agent-runtime` 拥有会话语义，宿主提供五个适配接口

| 接口 | 职责 | Desktop 实现 | Node 实现 |
|---|---|---|---|
| `SessionStore` | Ora Session 行的读写（状态、binding、标题） | SQLite 仓储 | Node 账本中的会话表 |
| `AgentAttach` | 按 canonical 插件 ID 附着 agent 插件进程并取得 ACP 帧通道 | `PluginApi` | Node 的插件生命周期（D2） |
| `SessionSetup` | 为 `session/new`/`session/load` 解析 `mcpServers` | Session MCP Snapshot | 第一版返回空集合（D5） |
| `RuntimeEvents` | 发布会话生命周期与标题等应用事件 | `AppEventHub` | 丢弃（Node 以 Thread 事件对外） |
| `WorkspaceDirectory` | 把会话的工作区解析为 `cwd` | Task → Workspace → worktree | checkout 执行 → clone 路径 |

- 接口在 crate 内定义、以泛型参数注入（优先静态分发）；crate 不依赖 `ora-backend`、`ora-db` 或任何
  Tauri 代码。会话记录仍由 `ora-history` 所有，crate 增加一个“记录已定型”回调点，供 Node 生成 Thread 事件。
- 抽取是**行为不变的重构**：先移动代码与测试，再替换 Backend 中的具体依赖；Backend 现有测试在抽取前后都必须
  通过，`docs/agent-runtime*.md` 相应改为描述 crate 与适配器。

为什么抽 crate 而不是在 Node 里另写一个最小 ACP 客户端：prompt 无活动重试、取消收敛、背压、封口与 handoff
都是踩过坑后的规则，最小客户端会重新遇到同样的问题，并让两处 Agent 行为分叉。代价是一次大的重构，需要在
Desktop 中先完成并验证。

### D2：Node 只运行本次会话需要的 agent 插件

- Node 引入插件生命周期（`ora-plugin-lifecycle`、`ora-plugin-runtime`）与 Deno 运行时；Node 镜像增加固定
  版本的 Deno。插件包来自 [plugin-runtime 根决策](../plugin-runtime/0-node-installs-plugins-into-workspace-home.md)
  的安装目录。
- 会话执行开始时，Node 按 `agent_plugin_id` 与版本找到已安装的插件，只启动这一个插件进程；未安装或版本
  不符时以 `AgentSessionEnded{agent_failed, detail: agent_plugin_unavailable}` 结束，不启动 Agent。
- 每个会话执行组合自己的插件生命周期，只含其指定插件，并以该会话的 Git 身份启动。插件进程用 `ora-process`
  直接启动（与 Desktop 相同），会话结束与 Node 正常停止时按进程组整树终止并等待退出。
- 部署配置了工作负载用户（`process.workload_uid`）时，Node 必须同时配置 `agent.workload_directory`（缺一启动
  失败）。插件进程及宿主经 `ora/childprocess/spawn` 为它启动的全部子进程都以工作负载 UID/GID 运行：在 fork 与
  exec 之间放弃附加组、设置 `no_new_privs`、清空能力并把 umask 设为 `077`，任一步失败则启动失败，不会以 root
  执行。
- 每个会话执行在 `workload_directory/<sha256(执行 ID)>/` 下得到只读的插件包硬链接视图（安装目录本身保持 Node
  私有）与属于工作负载用户的私有 HOME（`XDG_*`、`DENO_DIR` 都在其中），插件停止后删除，Node 启动时清空整个
  目录；启动前把 checkout 属主交还工作负载用户。Agent 的 Git 因此与 clone、交付 Git 同属一个用户，不被 Git 以
  `dubious ownership` 拒绝，也读不到 Node home、管理密钥与账本。未配置工作负载用户的 Node 与 Desktop 行为不变。

实际决策与偏离：原决策要求插件进程与 Agent CLI 都在执行的进程 scope 内、受 host/guardian 纳管。guardian 不提供
stdin 与插件协议流（[process/io 根决策](../process/io/0-bounded-output-and-fenced-input.md)），而 Deno 插件依赖 stdio
上的 JSON-RPC，所以插件改为由 `ora-process` 直接启动。代价是 Node 崩溃时插件因 stdio 关闭而退出，但忽略这一点
的后代进程不被回收；这些后代以工作负载用户运行，不持有 root 权限。纳入执行的进程 scope 留在“本决策未解决的
问题”。

### D3：会话执行的记录同时进入 JSONL 与 Thread 事件

- Ora Session ID 由会话执行 ID 确定（同一 UUID），会话 JSONL 位于 Node home 下 `ora-history` 规定的路径，
  随 Workspace 数据卷存在，交付时上传。
- 每条定型记录先追加到 JSONL，再以 Thread 事件写入 Node 账本并发送。运行时逐行追加 JSONL，每行写入后、
  下一行写入前同步回调宿主（`RuntimeEvents::record_settled`），因此“至多缺最后一条”按行成立，而不是按一次
  flush 的整批记录。二者之间崩溃时，最后一条记录可能只在
  JSONL 中；此时会话按协议 D6 以 `interrupted` 结束，Thread 缺少的最后一条仍随 Revision 的 JSONL 保存。
  选择这个顺序是为了让 JSONL 作为完整归档，Thread 最多缺尾部而不会有 JSONL 中不存在的条目。
- 首条 prompt 来自执行输入的 `initial_turn`，后续轮次来自 `SubmitUserTurn`，写入 JSONL 的用户消息记录都带
  `turn_id`。

### D4：git 身份只注入 Agent 进程树的环境

Node 通过插件生命周期的子进程环境接口（`ChildProcessEnvironmentProvider`）为该会话的插件进程及其派生的
全部子进程设置 `GIT_AUTHOR_NAME`、`GIT_AUTHOR_EMAIL`、`GIT_COMMITTER_NAME`、`GIT_COMMITTER_EMAIL`；
不修改 checkout 的 `.git/config`、不写全局配置。Node 自身执行的 Git（clone、Revision 快照）不使用这组变量，
它们有各自决策规定的身份。

### D5：第一版的运行时输入与权限

- 权限请求沿用 Desktop 的自动批准（运行时没有人工审批路径）；Thread 中可见工具调用结果。
- 不注入 MCP 服务（`SessionSetup` 返回空集合）：MCP 插件的配置与密钥下发尚无决策。
- 模型选择使用 Agent 默认值。
- Agent 调用模型所需的凭据由沙盒部署以环境变量提供（本地由 Sandbox Server 的 Node 配置注入），不经过 Cloud、
  Controller 或协议消息。这是第一版的部署约定，不是凭据模型的最终决策。

### D6：结束与失败

| 情况 | Node 行为 | 终态 |
|---|---|---|
| `EndSession` | 取消当前轮次，丢弃排队轮次，`session/close`（支持时），停止插件 scope，封口记录 | `AgentSessionEnded{reason}` |
| Agent 轮次失败、超时、连接丢失 | 按运行时既有规则记录 `TurnEnded`；会话保持，等待用户下一轮或结束命令 | 无 |
| agent 插件不可用或熔断打开 | 记录说明后结束 | `agent_failed` |
| Node 重启 | 见协议 D6 | `interrupted` |

### D7：Node 内会话执行与账本、插件目录、clone 账本之间的边界

会话执行（本文）与协议执行、插件安装、交付（[协议后续决策](../protocol/20260928-streamed-thread-events-and-session-commands.md)、
[plugin-runtime](../plugin-runtime/0-node-installs-plugins-into-workspace-home.md)、[revision](../revision/0-internal-snapshot-and-incremental-bundle-upload.md)）
在 Node 进程内只经下列接口交互。接口以 trait 定义在 `apps/ora-node/src/session/ports.rs`。

**Node 账本（`ora-node-db`）新增的持久结构**

| 表 | 内容 |
|---|---|
| `node_executions` | 四个新种类的执行：`execution`、`operation`、`kind`、`input`、`state`（`accepted \| running \| completed`）、`result`、`last_sequence`；`execution_identities.kind` 增加对应取值 |
| `execution_events` | `(execution, sequence)` 主键与事件 JSON；多事件 outbox，确认后删除。`last_sequence` 保证删除后序号不回退 |
| `session_commands` | `(execution, command_id)` 主键、命令 JSON、受理顺序、`state`（`queued \| executed \| discarded`） |

operation 的唯一事实仍在 `execution_identities`，不在会话表重复存储。`session_commands` 用受理顺序作物理主键，
并以 `UNIQUE(execution, command_id)` 保证命令身份语义。`ora-node-db::SessionJournal` 共享 Node 的独占租约；应用层
为它实现下面的 trait，避免数据库反向依赖应用。

**账本一侧提供给会话执行的接口**（`SessionLedger`）

| 方法 | 承诺 |
|---|---|
| `append_thread_event(execution, turn_id?, record, truncated) -> Sequence` | 分配下一个序号并持久化后返回；发送、在途窗口与重放由协议侧负责 |
| `end_session(execution, AgentSessionEnded) -> Sequence` | 写入终态结果与终态事件；之后同一执行的任何追加都失败；此时仍排队的命令一并结算为 `discarded` |
| `queued_commands(execution) -> Vec<(CommandId, SessionCommand)>` | 按受理顺序返回全部 `queued` 命令（落地时由 `next_command` 改来：轮次进行中必须看见排在用户轮次之后的 `EndSession`） |
| `settle_command(execution, command_id, executed \| discarded)` | 命令至多结算一次 |

**会话执行一侧提供给协议处理的接口**（`SessionHost`）

| 方法 | 调用时机 | 承诺 |
|---|---|---|
| `start(execution, AgentSessionSpec)` | 执行输入已持久化之后 | 按 D2 启动插件并送出首个轮次；立即返回，不等待会话 |
| `command_arrived(execution)` | 命令已持久化并回复受理之后 | 唤醒该会话读取 `queued_commands`；丢失唤醒不丢命令 |
| `recover_interrupted(execution) -> AgentSessionEnded` | Node 启动时，对没有终态的会话执行 | 关闭进程 scope、封口会话记录，返回 `interrupted` 结果由账本写入 |
| `sealed_history(execution) -> path` | 交付执行开始时 | 仅当会话已结束且记录已封口时返回 JSONL 路径，否则 `history_unavailable` |

**其他两个边界**

- `CheckoutResolver::checkout(clone_execution) -> path | checkout_unavailable`：由 clone 账本提供，会话与交付共用；
  调用方不自行拼接路径。
- `PluginCatalog::installed(plugin_id, version) -> 安装目录 | 无` 与 `PluginCatalog::lease(plugin_id) -> 使用租约`：
  由插件安装一侧提供。会话在启动插件前取得租约、结束后释放；安装与移除在租约存在时返回 `plugin_in_use`。

顺序与崩溃：会话执行先追加 JSONL，再调用 `append_thread_event`（D3）；协议侧在 `append_thread_event` 返回之后才发送。
Node 启动时先由协议侧恢复账本，再对未终态的会话调用 `recover_interrupted`，最后才接受新的会话命令。

为什么在这里固定：会话执行与账本、插件目录由不同的人同时实现。先固定接口，会话一侧可以用内存实现独立测试，
账本一侧可以用假 `SessionHost` 测试恢复与重放，而不必等待对方完成。

## 不变量

1. Desktop 在抽取前后的 Agent 行为与会话 JSONL 格式不变。
2. `ora-agent-runtime` 不依赖 Backend、SQLite 实现或 Tauri。
3. Node 的一个会话执行只启动其指定版本的 agent 插件；会话结束或 Node 正常停止时插件进程组整树终止。
4. Thread 事件中的记录都存在于同一会话的 JSONL 中。
5. git 身份只经环境变量进入 Agent 进程树，Node 不修改 git 配置。
6. 模型凭据不经过 Cloud、Controller 或 Controller–Node 协议。
7. 会话执行只经 D7 的接口读写 Node 账本、插件目录与 clone 账本；Thread 事件在账本持久化之后才会发送。
8. 插件在被会话使用期间不会被安装或移除替换。

## 为什么不是这些替代方案

| 方案 | 取舍 |
|---|---|
| Node 内写最小 ACP 客户端 | 快；重走 Desktop 已解决的重试、取消、背压问题，行为分叉（D1） |
| Node 直接依赖 `ora-backend` | 无需抽取；把 SQLite、Tauri 相关依赖和整个 Desktop 应用层拖进沙盒镜像 |
| Node 启动全部已安装插件 | 与 Desktop 一致；运行 Workspace 只服务一个 Agent，其余插件进程是浪费且扩大攻击面 |
| git 身份写入仓库配置 | 见 Cloud git 身份决策 |
| 模型凭据经 Cloud 下发 | 可按用户或 Space 计费与隔离；需要凭据模型与加密传递，超出本轮范围 |

## 本决策未解决的问题

- 模型凭据的最终来源（按用户、按 Space、平台托管）。
- MCP 与 Skill 插件在 Node 会话中的配置与注入。
- 权限请求的人工审批。
- Node 重启后恢复会话。
- 插件进程纳入执行的进程 scope，使 Node 崩溃后忽略 stdio 关闭的后代也能被回收（见 D2 的偏离）。

## 落地

Desktop 抽取 `ora-agent-runtime` 并由 Backend 提供五个适配器；Node 接入插件生命周期与 Deno，以 echo agent 插件
测试记录顺序、命令排队、结束与中断；Node 镜像加入 Deno，cluster 以真实 agent 插件验收。保持不变：Desktop 的 Agent
行为、数据与前端。开始改变：Node 能运行 Agent 会话。
