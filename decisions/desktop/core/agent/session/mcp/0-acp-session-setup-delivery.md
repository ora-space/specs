---
status: proposed
---

# MCP 通过 ACP Session Setup 注入而不物化到 Workspace

Ora 在每次 ACP `session/new` 与 `session/load` 前，从当前已安装且配置完整的 MCP 插件生成一次性的
`mcpServers` 内存快照，并随该请求交给 Agent。MCP 不再发布 Effect Source，不进入 Workspace Desired
State，也不创建、合并或删除 Agent 原生项目配置文件。这样，MCP 的配置生效边界与 ACP 原生的会话边界
一致，避免为瞬时会话输入建设文件 ownership、journal、reconcile 和 Agent restart 协议。

本文是尚未实现的提案，涉及 `ora-plugin-manager`、`ora-plugin-config`、`crates/backend` 的 Agent
Runtime、warm session pool，以及 Agent 插件现有的透明 ACP relay。MCP 静态包和 Settings 的规则继续由
`desktop/plugin/category/mcp/0-static-mcp-configuration.md` 定义；Skill 仍按 Effect 决策物化，不受本文
影响。当前 MCP 文件物化只存在于未发布的开发分支，因此本文不承担已发行版本的旧文件迁移义务。

## 术语与边界

- **Effective MCP Set** 是一次 session setup 可选择的完整 MCP 集合，来源是同一 installed snapshot 中
  静态有效、配置状态为 `Complete` 的全部 MCP 插件。
- **Session MCP Snapshot** 是 Host 针对一个 Agent、一个 `cwd` 和一次 setup 解析出的有序
  `mcpServers` 列表。它包含运行所需的 Setting 实值，但只在内存和 ACP setup 帧中存在。
- **MCP Session Setup** 是 Host 发送 ACP `session/new` 或 `session/load` 的动作。本文所说的 resume
  指现有 ACP `session/load`，不引入 `session/resume` 扩展。
- **Active Session MCP Set** 是 Agent 已从成功 setup 请求接受的集合。它不会因插件或 Settings 在会话
  运行期间变化而自动改变。
- Ora 负责选择、解析和传递 MCP 配置；Agent 负责按 ACP 连接并运行 Server。Ora 不代理 MCP 流量，也不
  把 Server 健康状态解释为插件安装或配置状态。

## 决策

### D1：Effective MCP Set 由 installed snapshot 与配置完整性共同决定

Host 为每次 session setup 枚举当前 installed snapshot。只有同时满足以下条件的条目进入 Effective MCP
Set：

1. installed package 的 canonical Plugin ID、version、kind 和 MCP Configuration 已通过静态验证；
2. 插件仍是当前 selected installed version；
3. 无 Settings，或当前 Configuration revision 的状态为 `Complete`。

集合自动适用于全部本地 Workspace，以及交互式聊天和 Workflow 创建的会话；一期不增加 Workspace 或
Session 级启用、排除和选择状态。配置不完整的 MCP 不进入集合，这不是 setup 失败；静态包无效的条目
本就不属于 installed snapshot。

Host 不为该集合创建 MCP Effect Source、Revision、Desired Effect、Target、Resource、Managed ledger
或 readiness watermark。installed、configuration complete 与 Agent 已接受某次 Session MCP Snapshot
仍是不同事实，任何一个都不能被命名为 `MCP Ready`。

### D2：每次 new/load 使用当时最新且自洽的 Session MCP Snapshot

Host 在发送 ACP 请求前解析一次 Effective MCP Set，并把解析结果作为该次请求的完整 `mcpServers`。
`session/new` 与 `session/load` 使用同一 resolver 和失败语义；Workflow 通过与交互式聊天相同的 warm
session 路径获得相同行为，不能维护另一套 MCP 组装逻辑。

Snapshot 的一致性单位是单个 setup 请求：每个成员绑定精确 installed package version 与
Configuration revision。若枚举后任一成员的 package 或 revision 已变化，Host 必须放弃本次 Snapshot
并使 setup 失败或在发送前重新生成完整 Snapshot，不能发送跨 revision 拼接或缺少成员的列表。

插件安装、更新、卸载或 Settings 保存只影响后续 setup：

- 已成功 setup 且正在运行的会话继续使用原 Active Session MCP Set，不 restart、不热更新；
- `session/load` 恢复会话时使用恢复当时最新的 Snapshot，不复用首次 `session/new` 的集合；
- 已调用 `session/new` 但尚未 attach 的 warm session 若其 Snapshot 已过期，必须释放旧 provider session
  并重新执行 `session/new`；
- Session MCP Snapshot、成员 revision 和 digest 都不写入 Ora Session 或 SQLite。

### D3：只使用 ACP 1.6.0 的原生 `mcpServers` wire contract

Host 使用 ACP `NewSessionRequest.mcpServers` 与 `LoadSessionRequest.mcpServers`，不新增 Host/Agent 私有
setup 方法。每个 MCP Configuration 按 transport 转换为 ACP 的封闭类型：

| MCP Configuration | ACP payload | 额外约束 |
| --- | --- | --- |
| `stdio` | `McpServer::Stdio` | `command` 是当前 installed package 内经重新验证的绝对普通文件路径 |
| Streamable HTTP | `McpServer::Http` | Agent 初始化必须声明 `mcp_capabilities.http = true` |

stdio 是 ACP Agent 必须支持的 transport，不额外建立可选 capability。`{ "context": "workspace" }`
在 Snapshot 解析时替换为本次 session 的绝对 `cwd`；普通 `.` 参数保持 literal。stdio env 和 HTTP
headers 都使用 ACP 的 name/value 列表，不转换为某个 Agent 的项目文件 schema。

`session/load` 是本文唯一的会话恢复协议。图片或外部材料中出现的 `session/resume` 只表达产品动作，
不构成 Ora 的 wire method。

### D4：canonical Plugin ID 同时提供 Server 身份与确定性顺序

每个 `mcpServers[].name` 使用完整 canonical Plugin ID `<namespace>/<identifier>`，列表也按该值升序
排列。canonical Plugin ID 在 installed snapshot 内唯一且不含 version，因此不同 source 的同名插件可
并存，同一插件升级不会改变 Server name；代价是 Agent 展示的名称比短 identifier 更长。

Host 不使用展示名称、identifier 尾段、安装目录名或内容 hash 作为 Server name，也不为冲突追加临时
后缀。相同输入必须产生字段顺序和成员顺序稳定的 Snapshot；实现可计算仅用于 warm-session 失效判断的
内存 digest，但该 digest 不是持久化身份或 ownership 证据。

### D5：入选成员采用 all-or-nothing setup

一旦 MCP 进入 Effective MCP Set，下列任一情况都使整个 `session/new` 或 `session/load` 失败，并确保
包含部分列表的 ACP 请求尚未发送：

- package version、Configuration revision 或 Setting binding 无法精确重读；
- required Setting 在解析时缺失、类型错误或出现非法运行时文本；
- stdio command 不再是当前 package 内的普通文件，或 workspace context 无法安全解析；
- Agent 未声明 HTTP capability，但集合包含 HTTP MCP；
- canonical name、环境变量名或 header 结构无法转换为合法 ACP payload。

失败必须返回稳定、可诊断且脱敏的 setup 错误。Host 不静默省略失败成员，也不把完整集合降级为空集合；
否则会话虽然可发送 prompt，用户却无法判断声明的工具为何缺失。未配置完整的插件按 D1 不属于集合，
因此不会让其他已配置 MCP 无法启动。

### D6：Setting 实值只进入内存 Snapshot 与受信 Agent relay

Host 在 setup 时把 Setting binding 解析为 stdio env 或 HTTP header 的实际字符串。实值可以经过 Agent
插件现有的 `agent/acp` relay，因为 Agent 插件已经拥有启动 Agent CLI 所需的近似 Host 权限，属于本文
明确接受的可信执行边界；未来若要求插件不可见秘密，必须另行设计 Host 到 Agent 子进程的直连或秘密
通道，不能用非标准占位符伪装成 ACP 支持。

Setting 实值不得写入 Effect、Ora Session、SQLite、Workspace、临时物化文件、sidecar、日志、错误文本
或 UI DTO。Host 不再生成 `ORA_MCP_*` 占位符，也不通过 Agent 进程环境建立第二条 Setting 注入链路。
Snapshot 在请求完成或失败后释放；错误报告只允许包含 Plugin ID、Setting ID、transport 与稳定错误码，
不能包含原值、带 prefix/suffix 的结果或可逆转义形式。

### D7：MCP 没有后台 materialization 状态或文件生命周期

Settings 只展示现有配置状态：未配置完整、配置完整或配置损坏。系统删除 MCP 专属的 `Projected`、
`Applied`、`Current`、`Blocked`、`Ready` 等状态，不通过轮询、Effect Target 或 Agent 运行状态提前声称
下一次 setup 一定成功。Agent capability、package/revision 竞争及转换错误都在实际 setup 边界报告。

无论安装、配置保存、setup 成功、setup 失败还是插件卸载，Ora 都不得为 MCP 创建、修改或删除
`.mcp.json`、OpenCode JSON/JSONC、ownership sidecar、Git exclude 或任何其他 Workspace 文件。用户和
其他工具已有的原生 MCP 配置属于 Preserved State，Ora 不读取它来授予权限，也不尝试合并、接管、去重
或清理。

当前文件物化实现没有进入正式版、预览版或需要支持升级的内部构建，因此删除该开发分支方案时不执行
产品级迁移。开发测试产生的文件由开发环境自行清理，不能把无发行历史变成运行时扫描所有 Workspace
的理由。

## 为什么不是这些替代方案

| 方案 | 优势 | 否决理由 |
| --- | --- | --- |
| 将 MCP 写入 Agent 原生项目配置 | Agent 脱离 Ora 也能读取，配置变化可在文件中观察 | 引入 shared-file ownership、JSONC 保真、journal、恢复、restart 与文件兼容边界；ACP 已原生承载同一输入 |
| 继续发布 MCP Effect Source，但 Resource 只在内存中应用 | 可复用 generation 与 readiness 状态 | 没有持久外部副作用需要 reconcile；Effect 状态会重复 Session 的真实 setup 边界并产生伪 Ready |
| 只在 `session/new` 传 MCP，load 复用首次集合 | 恢复行为完全固定 | 必须为每个 Session 持久化含秘密或 binding 的快照，并使配置修复、卸载无法影响恢复 |
| 配置变化时 restart 所有活跃 Agent/Session | 已打开会话能尽快看到新工具 | ACP 没有会话 MCP 热更新协议，restart 会中断 prompt，并重新引入跨会话协调状态机 |
| 某个 MCP 失败时跳过并继续 | 会话可用率更高 | 用户无法从成功的 setup 判断完整集合是否存在，需要额外的部分生效状态和告警协议 |
| 使用短 identifier 作为 Server name | Agent UI 更简洁 | 不同 namespace 的同名插件会碰撞，需要失败或不稳定后缀，破坏 canonical identity |
| 用 `ORA_MCP_*` 引用代替 ACP 实值 | Agent plugin relay 看不到明文帧 | ACP 不定义该替换语义，HTTP headers 尤其无法跨 Agent 一致解释，并保留第二条环境注入链路 |

## 不变量

1. MCP 只通过 ACP `session/new` 和 `session/load` 进入 Agent，不通过 Workspace 文件或 Effect Resource。
2. Effective MCP Set 只包含当前 installed snapshot 中静态有效且配置完整的 MCP 插件，并全局适用于本地
   Workspace 的交互式与 Workflow 会话。
3. 每次 setup 使用同一 resolver 生成完整、有序、绑定精确 package/configuration revision 的 Snapshot。
4. `mcpServers[].name` 必须是 canonical Plugin ID，成员顺序必须按该 ID 确定。
5. 任一入选成员无法安全解析或不受 Agent 支持时，完整 setup 必须在发送 ACP 请求前失败。
6. 正在运行的会话不因 MCP 配置变化而 restart 或热更新；load 与失效 warm session 使用当时最新集合。
7. Setting 实值只可存在于配置存储、内存 Snapshot 和受信 Agent ACP relay，不得进入其他持久化、日志、
   错误或 UI 数据。
8. MCP 生命周期不得创建、修改或删除任何 Workspace 文件；用户已有 Agent 原生配置始终保持不变。
9. MCP 配置完整不等于 Agent 已接受或 Server 健康，系统不得暴露无真实 setup 证据的 `MCP Ready`。

## 验收场景

- 同一 installed/configuration snapshot 对 `session/new` 与 `session/load` 产生完全相同、按 canonical
  Plugin ID 排序的 `mcpServers`；stdio workspace context 分别使用各自 `cwd`。
- Settings 修改后，已运行会话不受影响，下一次 load 使用新值，未 attach 的旧 warm session 被释放并
  用新 Snapshot 重建。
- Workflow node 与交互式会话通过同一 setup seam 获得 MCP，不存在 Workflow 专用组装或绕过路径。
- 集合包含 Agent 不支持的 HTTP MCP、过期 revision 或失效 command 时，setup 返回脱敏错误且 Agent
  没有收到部分 `mcpServers`。
- required Settings 不完整的 MCP 不进入集合，其他配置完整的 MCP 仍可完成 setup。
- 包含引号、反斜杠或合法特殊字符的 Setting 能以 ACP value round-trip；控制字符或非法 header/env
  在进入 ACP 前失败，错误中不出现原值。
- setup 的成功、失败、重试、配置变更和卸载都不会改变 Workspace 中任何 JSON、JSONC、sidecar 或 Git
  exclude 文件。

## 本决策未解决的问题

- MCP Server 的连接健康、工具发现、调用失败和重连属于 Agent/MCP 运行期，不在 session setup 成功
  语义内。
- Workspace 或 Session 级 MCP 选择、排除与权限提示需要新的用户意图和持久化模型，不在全局一期范围。
- 专用 Secret/Credential Store 及其授权、轮换和 redaction 规则不由本文定义；当前继续使用插件
  Configuration Store。
- ACP 后续若提供会话内 MCP 更新能力，需要新决策说明与运行中 prompt、warm pool 和错误状态的关系。

## 落地顺序

1. 建立纯 Session MCP resolver 与 ACP 映射，以 exact package/configuration revision、capability、排序和
   全量失败测试证明 Snapshot 边界；此阶段不接入真实 session。
2. 将 resolver 同时接入 warm `session/new`、恢复 `session/load` 和 Workflow 共用路径，并加入 warm
   Snapshot 失效；完成后新会话开始通过 ACP 获得 MCP。
3. 删除 MCP Effect publication、Resource/adapter、文件 ownership、环境占位符和 materialization UI，
   保留 Skill Effect；完成后任何 MCP 事件都不再产生 Workspace 文件副作用。
4. 删除无发行历史的旧物化代码与文档，完成秘密扫描、文件不变性和端到端 ACP setup 验证后，将本文
   推进到 `approved`/`implemented` 并补充对应核心测试用例。
