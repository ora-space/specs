---
status: implemented
---

# MCP Effect：将配置完整的 Plugin MCP 发布为不可变 Source 并安全收敛

Ora 把每个配置完整的 MCP 插件发布为 immutable、secret-free Effect Source revision，再由通用
Effect Target/Resource 协调、shared-file ownership adapter 和 Agent lifecycle 将 MCP Server 模板
收敛到 Consumer 声明的项目配置 Resource。发布只改变声明式 Desired State，不直接写入 Workspace；
配置失效或插件卸载时退休 Source，并通过后续 generation 清理由 Ora 管理的结果。

该决策已经由 SQLite Effect repository、`crates/backend`、`ora-effect-mcp` 与 Agent Consumer 落地。
它承接 `desktop/plugin/category/mcp/0-static-mcp-configuration.md` 对 MCP package、配置完整性与 Source
发布条件的定义，并具体化 `desktop/core/effect/0-effect-system-foundation.md` 的通用 Source、Revision、
generation、Resource、coordination 与 ownership 规则。插件如何分发、安装、验证 transport 和保存
Settings 不属于本决策。

## 术语与边界

- **Plugin MCP Source** 是由一个配置完整的 active MCP 插件派生的稳定 Effect 来源。
- **Plugin MCP revision** 是绑定插件版本、配置 revision 和 secret-free Server 模板的 immutable
  Effect Revision。
- **MCP Project Resource** 是 Agent Consumer 声明的 shared JSON/JSONC 配置文件，以及证明 Ora
  精确 ownership 的 sidecar。
- **MCP Ready** 表示完整 Target projection 已应用、Consumer 已重新激活且 ready watermark 已推进；
  它不等同于插件已安装、配置完整、项目文件已写入或 Agent 进程正在运行。
- **Runtime secret injection** 是 host 在启动精确 Agent 子进程时解析 Setting binding 的动作；它不属于
  Source publication 或项目文件 materialization。

## 决策

### D1：插件配置与 Effect publication 是相邻但独立的领域转换

MCP 插件只有在静态 contribution 有效且 Settings 完整时才发布 Source；配置保存、清除、插件更新、
卸载与启动 snapshot 同步只改变 Source publication 和 Scope Desired，不直接修改 Workspace 文件或
Agent 进程。

Installed MCP Descriptor、配置完整性、Effect publication、Resource materialization 与 Agent readiness
是不同事实。其中任一项成立都不能替代下一项的持久化证据，也不能单独授权 filesystem mutation 或
secret injection。安装、配置和 Workspace 创建等事件只唤醒 level-triggered reconciliation，不执行
另一套 MCP 专用复制流程。

### D2：Source 身份与 revision 绑定插件版本和配置 revision

每个 Plugin MCP publication 使用以下稳定映射：

| Effect 字段 | Plugin MCP 来源 |
| --- | --- |
| effect kind | `ora/mcp` |
| source kind | `plugin` |
| source namespace / identifier | canonical Plugin ID 的 namespace / identifier |
| revision key | `<plugin-version>:<configuration-revision>` |
| definition | secret-free Server template 与 Setting binding metadata |

无 Settings 的 MCP 使用配置 revision `0`；声明 Settings 的 MCP 只有在配置完整时才产生 revision。
同一 revision key 的 definition 必须保持 immutable，内容变化但 key 不变视为持久化损坏。Setting 实值
只存在于 host-owned `store.json`，不能进入 Revision、Desired、Operation journal、ownership sidecar、
项目配置、日志或 UI response DTO。

同一插件的新版本或新配置 revision 沿用稳定 Source identity，并发布新的 immutable revision。配置
缺失、损坏、不完整、contribution 失效或插件卸载会退休 active Source；旧 revision 行保持不可变，供
已有 projection、Managed State 与恢复链路继续引用。

### D3：Source lifecycle 通过完整 Scope generation 表达

新 Source 发布时，Effect repository 为现有 Workspace Scope 引入该 Source 的 Desired Effect，并各自
只推进一次 generation；新 Workspace 创建时 seed 当前 active Plugin MCP Sources。重复发布相同 Source
revision 是 no-op，不产生无意义 generation。

Source retirement 删除其当前 Desired intent，并令引用它的 Scope 进入下一 generation。retirement
不直接删除项目文件、sidecar 或历史 revision；清理由正常 reconcile 根据 Managed ledger 完成。启动
同步以完整 installed MCP snapshot 退休已不存在的 Plugin MCP Source，但 filesystem 外观本身仍不能
授予或撤销 Resource ownership。

### D4：Agent Consumer 以固定 Resource contract 消费 MCP

首批内建 Agent Consumer 各自把 Skill 与 MCP Resource 绑定到同一个 Target：

| Agent | Skill Resource | MCP Resource | ownership sidecar |
| --- | --- | --- | --- |
| OpenCode | `.opencode/skills` | `.opencode/opencode.json` 的 `mcp` | `.opencode/.ora-mcp-managed.json` |
| Claude Code | `.claude/skills` | `.mcp.json` 的 `mcpServers` | `.claude/.ora-mcp-managed.json` |

以上项目配置与 sidecar 路径是 filesystem compatibility boundary。升级不能改写或猜测既有同名 Server
的 ownership；旧文件继续可用，冲突只阻塞相关 Resource/Target。第三方 Agent 必须通过新的 Consumer
contract 显式声明 materialization format、项目路径与 lifecycle 能力，不能仅凭 Agent 名称复用内建
路径约定。

### D5：shared JSON/JSONC 只能通过精确 ownership 证据修改

MCP adapter 只增删 Ora sidecar 与数据库 Managed ledger 共同证明拥有的 Server key，保留其他顶层
字段、JSONC 注释和用户 Server。OpenCode 的 `.opencode/opencode.jsonc` 具有更高优先级；其中存在同名
Server 时会阻塞对应 Resource，而不是由 Ora 重写另一份配置规避。

sidecar 只保存稳定 identity、Plugin ID、Server name、配置 revision 与语义 fingerprint，不保存
Setting 实值。sidecar 缺失、不匹配、用户漂移、同名冲突或半完成写入都不能被自动接管；不确定状态
进入 blocking Condition 或 `RecoveryRequired`。配置文件写入采用原子替换，Operation 在副作用前
持久化；配置与 sidecar 之间发生崩溃时依赖 journal 隔离恢复，而不是根据现存内容猜测 ownership。

已收敛的相同 generation 重放必须 byte-stable/no-op。Source retirement 只能删除精确 owned key；
最后一个 Ora key 被删除后，仍必须保留合法的用户文件及其未拥有内容。

### D6：一个 Target projection 统一协调 Skill 与 MCP

Target projection 一次选择当前完整 Desired generation，并按 Resource binding 接受的 Effect kind 精确
路由；Skill Desired 不能进入 MCP planner，MCP Desired 也不能进入 Skill planner。同一 generation 的
所有安全 Resource mutation 完成后，Agent 只执行一次 reactivate，避免部分 generation 和重复重启。

任一 Resource blocking、Operation 未终结或 Consumer 验证失败时，Target 都不能推进 ready watermark。
UI 以 `(Workspace, Agent plugin)` 查询完整 Target readiness；聊天发送门不得以单个 Resource current
或“进程 running”替代 Effect readiness。

### D7：Secret 只在精确 Agent 子进程创建时注入

Plugin MCP revision 只携带 Setting binding。Host 仅在 Agent identity、canonical Workspace、sidecar、
项目配置 fingerprint、当前插件 contribution 与配置 revision 全部匹配时解析 Setting，并把结果注入
该 Agent 子进程。任何匹配失败都 fail closed，不退化为按变量名或 Server name 注入。

环境变量使用确定性的 `ORA_MCP_*` 保留命名空间，Agent 插件的 spawn 请求不得自行设置该前缀。
OpenCode 的 `{env:VAR}` 内容按 JSON string content 编码，Claude 的 `${VAR}` 使用 raw value；两种编码
都只在各自 Consumer contract 中解释，不改变 Source definition 的 secret-free 性质。

## 为什么不是这些替代方案

| 方案 | 否决理由 |
| --- | --- |
| 在 Workspace 创建、Agent 启动或插件安装处理器中直接复制 MCP | 绕过 generation、Target/Resource 协调、journal 与 ownership，部分失败后没有统一恢复证据 |
| 把 API key 写入项目配置、sidecar 或 Revision | 扩大持久化泄露面，无法按 Agent/Workspace 精确授权，也破坏 immutable revision |
| Skill 与 MCP 使用独立 Target 或分别重启 Agent | Agent 可能观察到部分 generation，ready watermark 会提前推进，并产生重复 lifecycle 副作用 |
| 每次重写整份 JSON/JSONC | 会删除用户字段与注释，且无法证明哪些 Server key 由 Ora 拥有 |
| 根据 Server name、模板或 fingerprint 相似性自动接管 | Observed State 不能证明 ownership，会把用户或其他工具的内容纳入 Ora 删除权限 |
| 把 secret 注入 Agent 插件进程后再转交 | 插件进程获得不必要的秘密访问面，且无法证明值只到达精确 Workspace 的 Agent child |

## 不变量

1. Installed MCP、配置完整、Effect publication、Resource materialization 与 MCP Ready 必须保持为可分别
   观察的事实，任一阶段都不能替代下一阶段的证据。
2. Plugin MCP revision 必须绑定 canonical Plugin ID、插件版本、配置 revision 和 secret-free Server
   template，发布后不可改写。
3. Setting 实值不得进入 Effect/Workspace 持久化、日志、插件进程环境或 UI DTO；只可注入通过完整
   identity 与 ownership 校验的 Agent 子进程。
4. Source publication 与 retirement 只能通过 Desired State 和 Scope generation 驱动收敛，不能直接
   修改 Workspace filesystem 或 Agent lifecycle。
5. shared MCP Resource mutation 必须同时满足 fenced Resource authority、durable journal、Managed
   ledger 与匹配 sidecar；名称或内容相似不能授权接管。
6. 同一 Target generation 的 Skill 与 MCP Resource 必须作为完整 projection 协调，所有 mutation 后
   最多 reactivate 一次，全部验证后才推进 ready watermark。
7. Source retirement 只能清理 Ora 精确拥有的 Server key，并保留用户文件、注释、未知字段与未拥有
   Server。
8. 项目配置与 sidecar 路径是 filesystem compatibility boundary；升级必须安全处理既有文件和冲突。

## 当前未解决的问题

- 第三方 Agent 的 MCP Consumer declaration、格式协商与热重载能力尚未定义；当前仅内建 OpenCode 和
  Claude Code contract。
- OpenCode 高优先级 JSONC 文件存在时的自动迁移或合并策略尚未定义；当前保持 blocking Condition。
- sidecar 与项目配置跨文件原子性仍依赖 durable journal 和恢复隔离；不引入多文件事务猜测机制。
- MCP Server 运行后的协议健康状态、工具发现与会话级故障不属于 Effect readiness；后续若纳入，必须
  单独定义观测来源与降级语义。

## 核心测试依据

见 `specs/test-cases/desktop/core/effect/mcp/plugin-mcp-materialization.md`。
