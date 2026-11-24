---
status: implemented
---

# MCP 通过 ACP Session 注入并令运行中会话收敛到最新配置

> 历史决策：[Skill 按存储路径以会话清单交给 Agent](../skill/0-skill-catalog-from-storage-paths.md)（`approved`）修改本决策中与 Effect 共享 barrier 的部分：开头“Skill 仍按 Effect 决策物化”、术语“Agent Session Barrier”、D4、不变量 6 与验收场景中的 Skill Effect 并发一条不再适用。Skill 改为按存储路径以会话清单交给 Agent，不占用 barrier、不重启 Agent、不 reload 会话；barrier 只剩 MCP refresh 一个原因。D1–D3、D5–D9 与其余不变量不变。

Ora 在 ACP `session/new` 与 `session/load` 前，从当前已安装且配置完整的 MCP 插件生成一次性的
`mcpServers` 内存快照，并随请求交给 Agent。插件安装、更新、卸载或 Settings revision 变化后，Host
还会令持有 provider channel 的运行中会话通过标准 `session/load` 收敛到最新快照：当前 prompt 不被
中断，但任何后续 prompt 都不能继续使用已过期的 MCP 集合。

MCP 不发布 Effect Source、不进入 Workspace Desired State，也不创建、合并或删除 Agent 原生项目配置
文件。Session Runtime 自己拥有 MCP 会话收敛，并与 Effect Agent adapter 共享同一个 Agent session
协调 seam；这样 Skill Resource mutation 与 MCP 变化并发时可以安全串行和合并恢复，而不会把瞬时会话
输入伪装成需要 filesystem ownership、journal 和 recovery 的 Effect Resource。

该决策已经由 `ora-plugin-config`、`ora-plugin-manager`、`crates/backend` 的 Session Setup 与 Agent
Runtime、Effect Agent adapter，以及 Agent 插件现有的透明 ACP relay 落地。本文固定 MCP 作为 Session
Runtime Input 的系统承诺。MCP 静态包和 Settings 规则继续由
`desktop/plugin/category/mcp/0-static-mcp-configuration.md` 定义；Skill 仍按 Effect 决策物化。

## 术语与边界

- **Effective MCP Set** 是一次 session setup 或 refresh 必须完整选择的 MCP 集合，来源是同一 installed
  snapshot 中静态有效、配置状态为 `Complete` 的全部 MCP 插件。
- **Desired Session MCP Revision** 是从 Effective MCP Set 的 canonical Plugin ID、精确 package version
  与 Configuration revision 得出的 secret-free 内容身份。它用于合并唤醒和判断运行中会话是否过期，
  不是持久化 ownership 或 Server 健康证据。
- **Session MCP Snapshot** 是 Host 针对一个 Agent、一个 Workspace `cwd` 和一次 ACP `session/new` 或
  `session/load` 解析出的有序 `mcpServers`。它包含 Setting 实值，但只在内存与该次 ACP 帧中存在。
- **Active Session MCP Revision** 是 Live Session 最近一次成功 setup 接受的 Desired Session MCP
  Revision。停止的 Ora Session 不持有 provider channel，因而没有 Active Session MCP Revision。
- **Live Session MCP Convergence** 是把 Live Session 从旧 Active revision 切换到最新 Desired revision
  的过程。它通过标准 `session/load` 完成，不表示 ACP 支持会话内增量修改 MCP。
- **Agent Session Barrier** 是 Effect Resource mutation、Agent replacement 和 MCP refresh 共用的协调
  seam。它阻止新 prompt 穿过变更窗口，并等待已经开始的 prompt 到达安全点。
- Ora 负责选择、解析、传递和刷新 MCP 配置；Agent 负责按 ACP 连接并运行 Server。Ora 不代理 MCP
  流量，也不把 Server 健康状态解释为插件安装、配置或会话收敛状态。

## 决策

### D1：Effective MCP Set 由 installed snapshot 与配置完整性共同决定

Host 为每次 session setup 或 refresh 枚举当前 installed snapshot。只有同时满足以下条件的条目进入
Effective MCP Set：

1. installed package 的 canonical Plugin ID、version、kind 和 MCP Configuration 已通过静态验证；
2. 插件仍是当前 selected installed version；
3. 无 Settings，或当前 Configuration revision 的状态为 `Complete`。

集合自动适用于全部本地 Workspace，以及交互式聊天和 Workflow 会话；一期不增加 Workspace 或
Session 级启用、排除和选择状态。配置不完整的 MCP 不进入集合，这不是 setup 或 refresh 失败；静态包
无效的条目本就不属于 installed snapshot。

Host 不为该集合创建 MCP Effect Source、Revision、Desired Effect、Target、Resource、Managed ledger
或 readiness watermark。installed、configuration complete、Session 已接受 Snapshot 与 Server 健康
仍是不同事实，任何一个都不能被统称为 `MCP Ready`。

### D2：所有 new/load 使用同一个精确且无副作用的 Snapshot resolver

Host 在发送 ACP 请求前解析一次 Effective MCP Set，并把解析结果作为该请求的完整 `mcpServers`。
`session/new`、显式恢复和运行中 refresh 的 `session/load` 使用同一个 resolver、transport 映射和失败
语义；Workflow 与交互式聊天不能维护另一套 MCP 组装逻辑。

Snapshot 的一致性单位是单个 ACP setup 请求：每个成员绑定精确 installed package version 与
Configuration revision。若枚举后任一成员的 package 或 revision 已变化，Host 必须在发送前放弃并重新
生成完整 Snapshot，或使本次 setup/refresh 失败，不能发送跨 revision 拼接或缺少成员的列表。

Resolver 自身无副作用。它返回有序 ACP payload 与对应 Desired Session MCP Revision；调用方不能把
Setting 实值放进 wakeup、session actor command、digest 或持久化状态。

### D3：配置变化令 Live Session 在安全点热更新

插件安装、更新、卸载、Settings 保存、清除或修复损坏配置在提交 durable configuration/catalog 事实
后，只发布“Desired MCP 可能变化”的 level-triggered wakeup。通知只降低延迟，不携带 Snapshot，也不是
correctness 的唯一来源；Session Runtime 在 refresh 和每次 prompt admission 前重新读取当前 Desired
revision。

Live Session 收到 wakeup 后遵守以下规则：

1. 空闲会话立即尝试 refresh；正在执行 prompt 的会话允许当前 turn 使用开始时的 Active revision 完成，
   并把 refresh 记为 owed work。
2. refresh 期间禁止新的 prompt 进入 Agent。Session Runtime 先解析并验证完整 Snapshot，再释放旧
   provider channel，并以相同 provider session identity 调用 `session/load(mcpServers)`。
3. 只有成功响应才能把 Active revision 推进到该请求携带的 Desired revision。请求完成时若 Desired
   revision 已再次变化，Session 仍保持 refresh pending，并继续收敛到最新值。
4. refresh 失败后 Session 进入带稳定、脱敏错误码的 blocked 状态；后续 prompt 必须先重试收敛，不能
   静默使用旧集合。其他 Session 可以独立继续收敛。
5. 已停止或尚未显式 load 的 Ora Session 没有 Live MCP 状态，不做后台 refresh；下一次 `session/load`
   直接使用当时最新 Snapshot。

这构成 Ora 的 MCP 热更新承诺：不打断进行中的 turn，但最新配置提交之后开始的任何 turn，要么使用最新
Desired revision，要么在进入 Agent 前明确失败。配置保存成功只表示新的 durable 输入已经提交并唤醒
收敛，不等待全部 Live Session 完成 refresh。

ACP `session/load` capability 是 Ora-managed MCP 的必要 Agent 能力。若 Effective MCP Set 非空而 Agent
未声明 load support，则新的 setup 和已有 Session refresh 都以稳定 capability 错误失败；Host 不通过
`session/new` 偷换 provider identity，也不依靠重放 UI 历史伪造原 provider 上下文。

### D4：Session Runtime 与 Effect 共享 Agent Session Barrier

MCP refresh 不创建 Effect generation 或虚拟内存 Resource。Session Runtime 提供统一的 Agent Session
Barrier，接收至少两类强类型原因：MCP Desired revision 变化，以及 Effect reactivation 后 Agent 实例被
替换。Effect Core 仍只依赖 Consumer Adapter；具体的 Session 枚举、prompt admission、ACP channel 与
MCP resolver 留在 Agent Session 模块内。

当 Skill Effect 和 MCP 变化并发时，协调必须满足：

- Effect 的 `coordinate` 先阻止会消费待变 Resource 的新工作，并等待进行中的 turn 结束；Resource
  mutation 和验证完成后才 `reactivate`。
- 若 reactivation 替换 Agent 实例，所有由该 Agent 承载的 Live Session 都失去旧 provider channel，并
  通过相同 Session convergence 路径恢复；恢复请求现场解析最新 MCP Snapshot。
- MCP wakeup 在已有 barrier 内只提高待收敛的 Desired revision，不启动第二条绕过 Effect fencing 的
  Agent restart 或 Session reload 路径。
- 若 Agent 没有运行，Effect 可以继续记录 disconnected receipt；停止的 Session 会在未来显式 load 时
  取得最新 MCP，无需为了 MCP 主动启动插件。

Effect Target readiness 只证明其 Resource projection 和 Consumer receipt，不证明任何 Session 已加载
MCP；Session MCP convergence 也不推进 Effect watermark。共享的是协调 seam，不是状态或 readiness。

### D5：只使用 ACP 1.6.0 的原生 `mcpServers` wire contract

Host 使用 ACP `NewSessionRequest.mcpServers` 与 `LoadSessionRequest.mcpServers`，不新增 Host/Agent 私有
setup 或 update 方法。每个 MCP Configuration 按 transport 转换为 ACP 的封闭类型：

| MCP Configuration | ACP payload | 额外约束 |
| --- | --- | --- |
| `stdio` | `McpServer::Stdio` | `command` 是当前 installed package 内经重新验证的绝对普通文件路径 |
| Streamable HTTP | `McpServer::Http` | Agent 初始化必须声明 `mcp_capabilities.http = true` |

stdio 是 ACP Agent 必须支持的 transport，不额外建立可选 capability。`{ "context": "workspace" }`
在 Snapshot 解析时替换为本次 Session 的绝对 `cwd`；普通 `.` 参数保持 literal。stdio env 和 HTTP
headers 都使用 ACP 的 name/value 列表，不转换为某个 Agent 的项目文件 schema。

`session/load` 同时承担显式恢复与 Live Session refresh。本文不引入 `session/resume` 扩展；未来 ACP
若提供标准的会话内 MCP update，Session convergence 可替换 transport adapter，而不改变 Desired/Active
语义和 prompt admission 不变量。

### D6：canonical Plugin ID 同时提供 Server 身份与确定性顺序

每个 `mcpServers[].name` 使用完整 canonical Plugin ID `<namespace>/<identifier>`，列表也按该值升序
排列。canonical Plugin ID 在 installed snapshot 内唯一且不含 version，因此不同 source 的同名插件可
并存，同一插件升级不会改变 Server name；代价是 Agent 展示的名称比短 identifier 更长。

Host 不使用展示名称、identifier 尾段、安装目录名或内容 hash 作为 Server name，也不为冲突追加临时
后缀。相同输入必须产生字段顺序和成员顺序稳定的 Snapshot。Desired/Active revision digest 只用于内存
收敛和防止过期 prompt，不是持久化身份、ownership 或 Server 健康证据。

### D7：入选成员按单个 Session setup/refresh 全量成败

一旦 MCP 进入 Effective MCP Set，下列任一情况都使整个 `session/new` 或 `session/load` 失败，并确保
包含部分列表的 ACP 请求尚未发送：

- package version、Configuration revision 或 Setting binding 无法精确重读；
- required Setting 在解析时缺失、类型错误或出现非法运行时文本；
- stdio command 不再是当前 package 内的普通文件，或 workspace context 无法安全解析；
- Agent 缺少 `session/load` 或所需 HTTP capability；
- canonical name、环境变量名或 header 结构无法转换为合法 ACP payload。

失败必须返回稳定、可诊断且脱敏的 setup/refresh 错误。Host 不静默省略失败成员，也不把完整集合降级
为空集合。all-or-nothing 的单位是单个 Session：多个 Live Session 可以分别成功、重试或阻塞，系统不
宣称跨 Session、跨 Agent 的全局原子切换。

配置不完整的插件按 D1 不属于集合，因此不会让其他配置完整 MCP 无法启动。若一个已 Active 的 MCP
随后因 Settings 清除而失去资格，最新完整 Snapshot 会移除它；这仍是一次全量 refresh，不是失败成员
被静默跳过。

### D8：Setting 实值只进入内存 Snapshot 与受信 Agent relay

Host 在 setup/refresh 时把 Setting binding 解析为 stdio env 或 HTTP header 的实际字符串。实值可以
经过 Agent 插件现有的 `agent/acp` relay，因为 Agent 插件已经拥有启动 Agent CLI 所需的近似 Host
权限，属于本文明确接受的可信执行边界；未来若要求插件不可见秘密，必须另行设计 Host 到 Agent 子进程
的直连或秘密通道，不能用非标准占位符伪装成 ACP 支持。

Setting 实值不得写入 Effect、Ora Session、SQLite、Workspace、临时物化文件、sidecar、wakeup、digest、
日志、错误文本或 UI DTO。Host 不生成 `ORA_MCP_*` 占位符，也不通过 Agent 进程环境建立第二条 Setting
注入链路。Snapshot 在请求完成或失败后释放；错误只允许包含 Plugin ID、Setting ID、transport 与稳定
错误码，不能包含原值、带 prefix/suffix 的结果或可逆转义形式。

### D9：只有 Session convergence 状态，没有 materialization 生命周期

Settings 继续只展示未配置完整、配置完整或配置损坏。Live Session 可在内存中区分 Active、Refresh
Pending、Refreshing 与 Blocked，以执行 prompt admission 和重试；这些状态不写入 Ora Session 或
SQLite，也不命名为 Effect `Projected`、`Applied`、`Ready` 或 Server `Healthy`。进程重启后没有可信的
Active revision，显式 load 必须重新建立事实。

无论安装、配置保存、setup、refresh、失败还是插件卸载，Ora 都不得为 MCP 创建、修改或删除
`.mcp.json`、OpenCode JSON/JSONC、ownership sidecar、Git exclude 或任何其他 Workspace 文件。用户和
其他工具已有的原生 MCP 配置属于 Preserved State；Ora 不读取它来授予权限，也不尝试合并、接管、去重
或清理。

当前文件物化实现没有进入正式版、预览版或需要支持升级的内部构建，因此删除该开发分支方案时不执行
产品级迁移。开发测试产生的文件由开发环境自行清理，不能把无发行历史变成运行时扫描所有 Workspace
的理由。

## 为什么不是这些替代方案

| 方案 | 优势 | 否决理由 |
| --- | --- | --- |
| 将 MCP 写入 Agent 原生项目配置 | Agent 脱离 Ora 也能读取，配置变化可在文件中观察 | 引入 shared-file ownership、JSONC 保真、journal、恢复、restart 与文件兼容边界；ACP 已原生承载同一输入 |
| 继续发布 MCP Effect Source，但 Resource 只在内存中应用 | 可复用 generation 与 readiness 状态 | MCP 是 per-Session runtime input，不是 Workspace 外部 Resource；Workspace Target 还会对共享 Agent 产生重复 fan-out 和伪 Ready |
| 为 MCP 建立独立的 Agent restart 流程 | 可以强制 Agent 丢弃旧 Server | MCP-only 变化只需 Session load；独立 restart 会中断更多状态，并与 Effect coordination 竞争同一个 Agent |
| 只在下一次用户打开会话时懒刷新 | 实现简单，不需要后台枚举 Live Session | 空闲但已打开的会话长期保留旧集合，而且新 prompt 容易在刷新前穿过；不满足运行中热更新承诺 |
| 不支持 load 时用 `session/new` 和 UI 历史重放替代 | 可以继续创建一个可 prompt 的会话 | 改变 provider identity，且 UI 历史不等于 Agent 内部上下文；会制造无证据的“同一会话已恢复” |
| 某个 MCP 失败时跳过并继续 | 会话可用率更高 | 用户无法从成功 setup 判断完整集合是否存在，需要额外的部分生效协议 |
| 使用短 identifier 作为 Server name | Agent UI 更简洁 | 不同 namespace 的同名插件会碰撞，需要失败或不稳定后缀，破坏 canonical identity |
| 用 `ORA_MCP_*` 引用代替 ACP 实值 | Agent plugin relay 看不到明文帧 | ACP 不定义该替换语义，HTTP headers 尤其无法跨 Agent 一致解释，并保留第二条环境注入链路 |

## 不变量

1. MCP 只通过 ACP `session/new` 和 `session/load` 进入 Agent，不通过 Workspace 文件或 Effect Resource。
2. Effective MCP Set 只包含当前 installed snapshot 中静态有效且配置完整的 MCP 插件，并全局适用于本地
   Workspace 的交互式与 Workflow 会话。
3. 所有 setup/refresh 使用同一 resolver 生成完整、有序、绑定精确 package/configuration revision 的
   Snapshot。
4. 配置变化不打断已经开始的 prompt；变化提交后开始的 prompt 必须使用最新 Desired revision，或在
   进入 Agent 前明确失败。
5. Live Session 只能在匹配 Snapshot 的 `session/load` 成功后推进 Active revision；过期成功响应不能
   清除更新 revision 的 owed work。
6. Effect 与 MCP 共享 Agent Session Barrier，但各自保持独立状态和 readiness；MCP wakeup 不得绕过
   Effect fencing 启动第二条 Agent lifecycle 路径。
7. `mcpServers[].name` 必须是 canonical Plugin ID，成员顺序必须按该 ID 确定。
8. 任一入选成员无法安全解析或 Agent 缺少必要 capability 时，单个 Session 的完整 setup/refresh 必须
   在发送部分 ACP payload 前失败。
9. Setting 实值只可存在于配置存储、内存 Snapshot 和受信 Agent ACP relay，不得进入其他持久化、通知、
   digest、日志、错误或 UI 数据。
10. MCP 生命周期不得创建、修改或删除任何 Workspace 文件；用户已有 Agent 原生配置始终保持不变。
11. 配置完整、Session 已接受 Snapshot 与 Server 健康必须保持为不同事实，系统不得暴露无对应证据的
   `MCP Ready`。

## 验收场景

- 同一 installed/configuration snapshot 对 `session/new`、显式恢复和运行中 refresh 产生完全相同、按
  canonical Plugin ID 排序的 `mcpServers`；workspace context 分别使用各 Session 的 `cwd`。
- Settings 修改时，空闲 Live Session 自动执行 `session/load`；成功后下一次 prompt 使用新值，旧 Setting
  不出现在持久化、通知、日志或错误中。
- Settings 在 prompt 期间修改时，当前 turn 正常结束；新 prompt 被挡在 admission 前，直到 refresh
  成功或返回明确的脱敏错误。
- 连续发生安装、保存和卸载时，Session 可以完成过期的 in-flight load，但不能把它当作 Current；最终
  收敛到最新 Desired revision，且不会在两次 refresh 之间接受旧 revision 的 prompt。
- 一个 Session refresh 失败不会回滚已提交配置，也不会阻止其他 Session 成功；失败 Session 不能继续
  使用旧集合发送 prompt。
- Skill Effect mutation 与 MCP 更新并发时，Agent Session Barrier 串行化 lifecycle；Agent replacement
  后恢复的 Session 现场解析最新 MCP，不先恢复旧集合再遗漏 owed update。
- Agent 缺少 `session/load` 或 HTTP MCP capability 时，包含对应 MCP 的 setup/refresh 在发送 ACP 请求前
  以稳定 capability 错误失败。
- required Settings 从完整变为不完整后，该 MCP 从最新完整 Snapshot 中移除；其他配置完整 MCP 仍可
  共同完成 refresh。
- setup、refresh、失败、重试、配置变更和卸载都不会改变 Workspace 中任何 JSON、JSONC、sidecar 或 Git
  exclude 文件。

## 本决策未解决的问题

- MCP Server 的连接健康、工具发现、调用失败和重连属于 Agent/MCP 运行期，不在 setup/refresh 成功
  语义内。
- Workspace 或 Session 级 MCP 选择、排除与权限提示需要新的用户意图和持久化模型，不在全局一期范围。
- 专用 Secret/Credential Store 及其授权、轮换和 redaction 规则不由本文定义；当前继续使用插件
  Configuration Store。
- ACP 后续若提供标准的会话内 MCP update，是否采用及如何协商 capability 留给后续决策；本文的
  Desired/Active 收敛与 prompt admission 语义保持不变。

## 核心测试依据

见 `specs/test-cases/desktop/core/agent/session/mcp/acp-session-setup-delivery.md`。
