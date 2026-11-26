---
status: implemented
---

# Skill 按存储路径以会话清单交给 Agent，取代 Effect 物化

Desktop 退役 Effect 机制，Skill 不再物化到 Workspace。Skill 目录就是它的存储位置：本地导入的 Skill 在应用数据
目录（数据根）的 `atoms/skills/<name>/`，插件提供的 Skill 在插件安装目录内。新会话的第一条用户 prompt 前附一份
Skill 清单，每项只有名称、描述和 `SKILL.md` 的绝对路径，Agent 需要时自己读取 `SKILL.md`。会话中途安装或卸载
Skill 时，下一条用户 prompt 前只附增量；压缩或冷启动 `session/load` 后重发全量。注入由 `ora-agent-runtime` 通过 host 扩展点完成：
desktop host 返回已安装 Skill 和存储根，ora-node host 返回空。`session/new` 与 `session/load` 把存储根作为额外可读
目录交给 Agent 插件，插件可以选择把它们接到 Agent 的原生 Skill 机制，提示词清单是所有 Agent 的统一基线。

行为变化：

- Workspace 里不再出现 Ora 写入的 Skill 目录、`.ora-managed.json` 或任何 Skill 相关文件。
- 发送 prompt 不再等待 Skill 同步，也不再有同步失败、冲突和恢复横幅。
- 同名 Skill 可以同时出现在清单里，按路径区分，不再因物化目录冲突而阻塞。
- 运行中的工作流只冻结 skill ID 和调用名，运行中升级 Skill 后，后续节点读到新内容。

本决策为 `implemented`。Desktop 实现、退役与一次性清理已随 [desktop#827](https://github.com/ora-space/desktop/pull/827)（合并提交 [`9f3ed63f`](https://github.com/ora-space/desktop/commit/9f3ed63fa3f36701d4d2f8ca25414ea018d779cf)）落地（对应的决策记录是 ora-space/circus#1458，落地任务是 ora-space/circus#1460）；`packages/plugin-sdk` 已同步去掉 `effectResources`、`effect/*` 与 `SKILL_DIRECTORY_V1`（保留 `CLAUDE_MCP_CONFIG_V1` 与 `OPENCODE_MCP_CONFIG_V1`），版本升至 `0.12.0`。移除 `effectResources` 的新版 Agent 插件不得早于包含本决策实现的 Desktop 正式版发布；旧 Desktop 配新插件会丢失 Skill（见“风险与迁移”）。核心用例与代表性证据见 [`test-cases/desktop/core/agent/session/skill/`](../../../../../../test-cases/desktop/core/agent/session/skill/)。

- **涉及**：desktop 的 `crates/agent-runtime`（host 扩展点、清单前缀与增量、ACP setup 字段与压缩重置）、`crates/backend`
  （desktop host、工作流 Skill 解析、升级前清理与备份接入）、`crates/db`（`skills.package_root` 与删除 Effect 表的前向迁移 `0025`、迁移前备份与清理）、`apps/desktop`（删除
  Effect 命令、发送门禁与冲突横幅）、`packages/plugin-sdk` 与 Agent 插件（去掉 `effectResources` 和 `effect/*`
  处理）；`apps/ora-node` 只做编译适配。
- **整体退役**：`crates/effect`、`crates/effect-skill`、`crates/db` 的 `repository/effect/*`、backend 的
  `effect_worker`、`effect_registration` 与 `effects.rs`、Tauri 的 `getEffectTargetStatus`、`retryEffectTarget`、
  `recoverEffectTarget`，以及插件协议的 `effectResources` 与 `effect/coordinate`、`effect/reactivate`、
  `effect/verify_ready`。
- **取代**：[Effect 系统基础](../../../effect/0-effect-system-foundation.md)、
  [Effect 的行审计时间与业务时间](../../../effect/20260905-separate-audit-and-business-time.md)、
  [Target 生命周期转换推进收敛纪元](../../../effect/20261010-retirement-epoch.md)和
  [Plugin Skill Source](../../../effect/skill/0-plugin-skill-source.md)四份决策整份被本文取代。
- **不涉及**：Skill Catalog 的创建、导入、编辑与删除，Skill 插件的包格式与发现规则，Skill Settings 的启动环境
  注入，MCP 的会话交付，cloud v1 与 v2-server。v2-server 是否统一到同一方案另行决定。
- **不承担的迁移**：不保留 Effect 与本方案并存的阶段；旧 Workspace 中已物化的目录只按 D9 做一次性清理。

## 背景

Effect 为了把 Skill 复制到每个 Workspace 中 Agent 插件声明的发现目录（如 `.claude/skills`），建立了 Desired、
Target、Resource 三层收敛，配套 ownership 账本、journal、恢复隔离和发送门禁。Effect 的唯一 kind 是 Skill
（`skill_directory.v1`），这套机制只有这一个消费者。

物化本身在 desktop 的主要平台上代价很高。desktop 只支持 Windows 和 Linux
（[支持平台决策](../../../../platform/0-windows-and-linux-only.md)），主要用户在 Windows：符号链接需要管理员权限或
开发者模式；junction 有删除穿透问题，与同步盘、杀毒软件也有兼容问题；复制目录又要处理文件句柄占用
（ora-space/circus#689）。Workspace 里的副本还会进入用户的 Git 工作区，插件更新、卸载时都要收敛和清理。

Agent 读取 Skill 并不需要 Skill 位于 Workspace 内，只需要知道 `SKILL.md` 在哪里，并且有权读取。存储位置本身就满足
这个条件，所以 Ora 不必再维护副本。

## 术语

- **Skill 存储位置**：Skill package 的权威目录。本地导入的 Skill 在 `<应用数据目录>/atoms/skills/<name>/`，插件
  提供的 Skill 在 `<插件安装目录>/assets/<skill>/`，后者由[静态 Skill 包决策](../../../../plugin/category/skill/0-static-skill-package.md)
  D2 定义。
- **会话 Skill 清单**：某一时刻所有 Available Skill 的条目集合，每项包含名称、描述和 `SKILL.md` 的绝对路径。
  条目以 `SKILL.md` 绝对路径为身份。
- **已告知清单**：runtime 在内存中为一个 Live Session 记录的、已经告知该 Agent 的条目集合。
- **清单增量**：当前清单与已告知清单按路径比较得出的新增条目和移除条目。
- **Skill 可读根**：desktop host 提供的稳定存储根目录集合，即本地 Skill 根 `<应用数据目录>/atoms/skills/` 和插件安装根
  `<Ora 主目录>/plugins/installed/`。两者是不同的根目录。
- **旧受管 Skill 目录**：Effect 在 Workspace 中物化、带 `.ora-managed.json` 标记并登记在 Effect 账本中的目录。

## 与前序决策的关系

| 前序决策 | 处理 | 当前规则 |
| --- | --- | --- |
| [Effect 系统基础](../../../effect/0-effect-system-foundation.md) | 整份取代 | Desktop 不再有 Effect Scope、Target、Resource、账本或 worker（D1） |
| [Effect 审计时间与业务时间](../../../effect/20260905-separate-audit-and-business-time.md) | 整份取代 | Effect 表由前向迁移删除，审计时间规则失去对象（D1） |
| [Target 生命周期推进纪元](../../../effect/20261010-retirement-epoch.md) | 整份取代 | 发送门禁与 `obligation` 一并删除，发送不等待 Skill（D1） |
| [Plugin Skill Source](../../../effect/skill/0-plugin-skill-source.md) | 整份取代 | Plugin Skill 不再发布为 Effect Source，Catalog 中的 Available Skill 直接按存储位置进入清单（D2） |
| [静态 Skill 包](../../../../plugin/category/skill/0-static-skill-package.md) D1–D7 | 继承，不变 | 包布局、Catalog 身份、只读与 availability 规则不变；package root 就是交付位置 |
| 静态 Skill 包：“Catalog 之后如何物化为 Effect 属于 Effect 领域” | 修改 | Catalog 之后的交付由本文定义，不经过 Effect |
| [Skill Settings 与启动环境](../../../../plugin/category/skill/20261016-secret-settings-and-startup-environment.md) | 继承；措辞修改 | 配置完整性不影响 Skill 是否进入清单；原文“Skill 仍物化”改读为“Skill 仍进入清单” |
| [Session MCP 根决策](../mcp/0-acp-session-setup-delivery.md) D1–D3、D5–D9 | 继承，不变 | MCP 仍只经 ACP `mcpServers` 交付 |
| Session MCP 根决策 D4 与不变量 6 | 修改 | Agent Session Barrier 只剩 MCP refresh 一个原因；Skill 变化不占用 barrier（D7） |
| [工作流显式选择](../mcp/20260912-workflow-explicit-selection.md) D4 | 修改 | 已安装 Skill 进入每个会话的清单；节点启用表示必须使用该 Skill，仍不是安全白名单（D8） |
| [工作流 MCP 派发预检](../mcp/20261119-workflow-mcp-preflight.md) D3 | 修改 | Skill 依赖检查改为“Catalog 能解析且存储路径存在”（D8） |
| [来源派生 Namespace](../../../../plugin/registry/20260831-source-derived-namespace.md)、[安装期包校验](../../../../plugin/registry/20260922-install-time-package-validation.md) | 继承；对象减少 | `skills` 表仍按完整 Plugin ID 匹配；`effect_sources`、`effect_consumers` 表随 Effect 删除 |
| [Node 共享 Agent runtime](../../../../../node/agent-runtime/0-shared-agent-runtime-crate-hosted-by-node.md) | 继承，不变 | Node host 的扩展点返回空，Node 会话行为不变，Node 不再依赖 `ora-effect`（D5） |

## 决策

### D1：Desktop 整体退役 Effect

上文“整体退役”所列模块全部删除；新增前向迁移 `0025` 删除全部 Effect 表，不改写旧迁移文件。删除后：

- 发送 prompt 不读取任何 Skill 就绪状态，前端没有 Skill 同步门禁、冲突横幅和恢复入口。
- Host 不调用 `effect/*` 方法。旧版 Agent 插件在注册时仍可能携带 `effectResources`，host 忽略该字段，不使连接失败；
  这样新 desktop 可以继续使用尚未升级的 Agent 插件，代价只是该插件原生发现目录里不再出现 Skill。
- `packages/plugin-sdk` 去掉 `effectResources`、`effect/*` 处理接口与 `SKILL_DIRECTORY_V1`（保留 `CLAUDE_MCP_CONFIG_V1` 与 `OPENCODE_MCP_CONFIG_V1`），版本升至 `0.12.0`；Agent 插件去掉 `effectResources`、`effect/*` 处理与协调器，且移除 `effectResources` 的新版 Agent 插件不得早于包含本决策实现的 Desktop 正式版发布。
- `AppEvent::AgentModelsInvalidated` 接口保留，当前没有后端发出者，留待 `ora-space/circus#1474` 的 Skill 失效能力使用。

这样选的原因：Effect 的复杂度来自“安全地改写 Workspace 内的共享目录”。D2 之后 Ora 不再写 Workspace，没有需要收敛的
外部状态，保留通用 Effect Core 只会留下没有消费者的状态机、表和恢复入口。将来若出现新的 Workspace 副作用需求，
应基于它自己的约束另行决策，而不是为假想的消费者保留这套机制。

### D2：Skill 存储位置就是交付位置

清单条目指向 Skill 存储位置下的 `SKILL.md`，Ora 不复制、不链接、不在 Workspace 中写任何 Skill 相关文件，也不需要
Git exclude。条目只来自当前 Available 的 Skill：本地 Skill 的正式目录存在且 `SKILL.md` 可解析，插件 Skill 的
package root 按静态 Skill 包 D6 重新校验通过。插件 Skill 的存储路径由 `skills.package_root` 列持久化（schema `0025` 从 `effect_revisions` 回填）。Unavailable 的 Catalog 行不进入清单。

| 来源 | 存储位置 | 路径何时变化 |
| --- | --- | --- |
| 本地导入 | `<应用数据目录>/atoms/skills/<name>/` | 改名时目录随之改名，路径变化；只改内容时不变 |
| 插件提供 | `<Ora 主目录>/plugins/installed/<namespace>/<identifier>/<version>/assets/<skill>/` | 插件升级到新版本时路径变化 |

规则：

1. 路径使用当前平台的原生绝对路径形式，原样呈现，不做转义或缩写，并统一剥离 Windows `\\?\` verbatim 前缀（`strip_verbatim_prefix`）。两个根目录由用户机器决定，可能含空格或非 ASCII 字符。
2. `SKILL.md` 中的相对路径按 `SKILL.md` 所在目录解析。清单说明这一点，因为 Agent 的 `cwd` 是 Workspace，而不是
   Skill 目录。
3. 同名 Skill（本地与插件、或两个插件）各自成为一个条目，按路径区分，不合并、不改名、不阻塞。原先物化时同名 Skill
   落到同一 native 目录会冲突，存储位置天然互不重叠，这类冲突随之消失。
4. 插件 Skill 目录由插件拥有，按静态 Skill 包不变量 4 保持只读；本地 Skill 目录仍只由 Skill Catalog 的事务修改。
   本文不向 Agent 授予写这些目录的理由。

为什么以 `SKILL.md` 路径作为条目身份，而不是 Skill ID：Agent 消费的是路径，路径变化就意味着 Agent 手里的旧路径失效，
必须告知；路径不变而内容更新时，Agent 下次读取自然读到新内容，无需告知。Skill ID 在插件升级时保持不变，却对应
新版本目录，用 ID 做身份会漏掉这次路径变化。

### D3：新会话的第一条用户 prompt 携带全量清单

Live Session 发送第一条用户 prompt 时，runtime 在发送时刻读取当前清单；清单非空时，在该 prompt 中加入一个
`<available_skills>` host 上下文块，内容为：

- 一句说明：以下是可用的 Skill，任务与某个 Skill 的描述相符时，先读取它的 `SKILL.md` 并遵循其中说明；
- 每个条目的名称、描述和 `SKILL.md` 绝对路径，按名称的 ASCII 小写、再按路径排序；
- 相对路径按 `SKILL.md` 所在目录解析的规则。

```text
<available_skills>
Available skills are listed below. When a task matches a skill's description, read its SKILL.md first and follow its instructions. Relative paths inside SKILL.md resolve against its directory.
- review: Review a change against the team checklist.
  <应用数据目录>\atoms\skills\review\SKILL.md
- review: Security-focused review.
  <Ora 主目录>\plugins\installed\acme.3f9a1c2d\sec-pack\1.2.0\assets\review\SKILL.md
</available_skills>
```

规则：

1. 清单块位于用户内容之前。例外是首个文本块以斜杠命令开头（用户输入的 `/命令`，或接入原生 Skill 时工作流节点前导的 Skill 调用）：
   该块保持在最前，清单块紧随其后，避免破坏 Agent CLI 对前导斜杠命令的解析。
2. 清单块是 host 上下文，不是用户输入；界面不把它显示成用户消息的一部分。
3. 首轮清单为空时不加块；prompt 发送成功后，已告知清单初始化为空集合（后续首次安装 Skill 时发出 `<available_skills_update>`）。
4. 交互式聊天和工作流节点会话都使用同一套前缀逻辑；工作流节点在此之外仍有自己的必需 Skill 块（D8）。
5. 本文不截断清单。描述本身受 4096 bytes 上限约束；清单的总体预算留作未决项。

只列名称、描述和路径属于渐进加载：Agent 只在需要时读取正文，不把全部 Skill 正文塞进上下文。

### D4：会话中途的变化以增量告知，读取失败保护与重置边界

Runtime 在内存中为每个 Live Session 保存已告知清单。之后每条用户 prompt 发送前，runtime 读取当前清单，按
`SKILL.md` 路径与已告知清单比较：

| 比较结果 | prompt 中加入的内容 |
| --- | --- |
| 路径集合相同（包括名称、描述或内容在原路径上变化） | 不加任何块 |
| 有新增路径 | `<available_skills_update>` 列出新增条目的名称、描述和路径 |
| 有移除路径 | `<available_skills_update>` 列出移除条目的名称和路径，说明它们已不可用 |
| 两者都有（如插件升级、本地 Skill 改名） | 同一个 `<available_skills_update>` 块同时列出新增与移除 |

增量块的位置规则与 D3 相同。prompt 请求被 ACP 线缆接收后，runtime 才把已告知清单更新为本次比较所用的当前清单；
prompt 在进入 Agent 前失败（例如 MCP refresh 失败被挡在 admission 前）时不更新，下一次发送重新比较。

**清单读取失败语义**（区分整份清单失败与单条目失败）：

- **整份清单读取失败**（如 SQLite 查询错误）：本轮跳过清单注入，且不更新或初始化已告知清单（既不发出误移除增量，也不固化空基线）；在 `session/new` / `session/load` setup 阶段记录告警并省略可选的 `_meta["ora/skillDirectories"]`。
- **单个条目 `SKILL.md` 读取失败**（I/O 或权限错误）：将对应 `SKILL.md` 路径记入 `unreadable_paths` 标为“未知”——已告知的路径不宣告移除并在后续轮次保留记录，未告知的路径不新增，其余可读条目照常注入与更新。（已知边界：包目录本身不可读与 `SKILL.md` 不可读的区分由 `ora-space/circus#1533` 跟进。）

已告知清单这份记录只存在于内存，不写入 Ora Session 或 SQLite；清单块本身只是随 prompt 发给 Agent 的内容。以下
情况清空已告知清单，下一条用户 prompt 携带全量 `<available_skills>`：

- Live Session 的 runtime actor 新建，包括应用重启后的冷启动 `session/load`、停止的 Ora Session 再次打开（`mark_stopped` / `unload`），以及 stale load 等引发的 provider session rebuild（`restore_provider_session` / `rebuild_provider_session`）；
- OpenCode 父会话在 `SessionInfoUpdate` 中通过 `_meta["opencode/compaction"].status == "completed"` 报告上下文压缩完成（`started`、`failed` 与子会话的压缩事件不重置）。

同一 Live Session 内为了 MCP refresh 发起的 `session/load` 走 replay-only 通知观察路径，不清空已告知清单，也不触发压缩重置：provider 会话身份不变，已告知内容仍在 Agent 的会话上下文里。Claude Code 尚无压缩通知信号，压缩后可能失去清单直至下次冷启动，列为已知限制。

为什么不每条都重发全量：清单放在每条 prompt 前会改变 prompt 前缀，使 Agent 侧的 prompt cache 失效，并为每轮付出
整份清单的 token。为什么不在变化时 `session/load` 重建会话：它要求 Agent 支持 load，会在空闲点打断会话，而 Skill
变化并不需要 Agent 丢弃任何运行时状态。记录放在内存而不持久化，是因为持久化的记录无法证明重启后的 Agent 上下文
仍包含它，重发全量才是安全的重置点。

### D5：注入放在 `ora-agent-runtime`，由 host 扩展点提供内容

Runtime 负责首条前缀、增量计算、已告知清单和 ACP setup 字段；内容由 host 通过 `AgentRuntimeHost::Skills`（`SkillCatalogSource` trait）提供：

| 扩展点方法 | desktop host (`DesktopSkillCatalog`) | ora-node host (`NoSkillCatalog`) |
| --- | --- | --- |
| `skill_catalog(workspace)` | `Result<SkillCatalogSnapshot, RuntimeError>`（含当前 Available `entries` 与 `unreadable_paths`） | 空快照 |
| `extra_read_roots()` | Skill 可读根 | 空列表 |

`workspace` 参数为将来按 Workspace 选择 Skill 预留；本文中 desktop 对所有 Workspace 返回同一份清单。

Node host 返回空，所以 Node 会话没有前缀、没有增量、没有额外可读目录，ACP 请求与 prompt 和本决策前一致。`ora-node` 已去掉对 `ora-effect` 的依赖和 `replace_agent_effect_declaration` 空实现。

放在 runtime 而不是 desktop backend，是因为交互式聊天与工作流节点会话都经过 runtime 的 prompt admission 和 Live
Session actor；已告知清单必须与 actor 的生命周期一致，放在 backend 需要复制一份会话生命周期判断。

### D6：`session/new` 与 `session/load` 把存储根交给 Agent 插件

1. **额外可读目录**：Agent 在初始化时声明 ACP `sessionCapabilities.additionalDirectories` 时，runtime 在每次
   `session/new` 与 `session/load` 的 `additionalDirectories` 中发送 Skill 可读根。ACP 规定 load 请求中的列表是完整
   结果，所以 load 每次都发送完整列表。Agent 未声明该能力时不发送，setup 不因此失败；Agent 读取存储位置时可能走它
   自己的权限确认。这与 MCP 不同：缺少可读目录只影响读取体验，不会让会话的输入不完整。
2. **可读根取稳定的存储根**，而不是每个 Skill 的目录：会话中途安装的插件 Skill 落在已授权的插件安装根之下，不需要
   重建会话。插件安装根下还有其他插件的包文件，它们只读且不含 Setting 实值（Setting 存于配置存储）；本文接受这个
   读取范围。
3. **可选的原生 Skill 接入**：同一 setup 请求以 `_meta["ora/skillDirectories"]` 携带当前清单中各 Skill 的包目录；插件可以把它们接到 Agent 的原生 Skill 机制（例如
   Claude Code 的原生 Skill 工具和 `/skill名` 入口），也可以忽略。插件不得写入这些目录。清单为空或读取失败时不带该字段。
4. 原生接入只反映 setup 时刻的清单；会话中途的变化只通过 D4 的提示词增量告知。原生机制与提示词清单指向同一批路径，
   不会产生两份内容不同的 Skill。当前尚无 Agent 插件消费 `_meta["ora/skillDirectories"]`；Agent 插件（如 `claude-code-agent`）可将 `additionalDirectories` 与 `_meta["ora/skillDirectories"]` 透传给 ACP adapter，原生 Skill 接入与声明机制留待 `ora-space/circus#1511`。

为什么提示词是基线、原生接入可选：各 Agent 的原生 Skill 机制差异很大，有的没有；提示词对所有能读文件的 Agent 都
成立。只用提示词会让 Claude Code 失去原生 Skill 工具和斜杠入口，可靠性略低，因此保留可选接入，代价是协议多一个
可选字段。

### D7：Skill 变化不再经过 Agent Session Barrier

Skill 安装、更新、卸载、改名不占用 Agent Session Barrier，不重启 Agent，不 reload 任何会话，也不打断进行中的 turn。
Barrier 的原因只剩 MCP refresh：Effect mutation 和 Effect reactivation 引起的 Agent replacement 随 Effect 一起删除。
MCP 的 Desired/Active 收敛、prompt admission 和全量成败规则不变。

### D8：工作流只冻结 skill ID 和调用名

1. 运行创建与切换快照时冻结每个节点启用的 Skill：`{ node_id, skill_id, invocation_name }`（由 `catalog_skill_id` 字段精确记录冻结的 Catalog ID）。不冻结内容、版本或路径，也不按 revision 保留存储目录。
2. 当工作流节点引用的裸 Skill 名匹配到多个活跃 Catalog ID（保守计入不可用候选）时，创建运行、启动和切换快照均以公共错误 `workflow_skill_ambiguous { nodeId, nodeTitle, skillRef, candidates }` 拒绝，提示用户指定具体 Skill；显式 catalog ID 照常解析。该歧义只出现在保存裸名称的图记录上；编辑器改存 catalog ID 留待后续决定（`ora-space/circus#1527` / `ora-space/circus#1563`）。
3. `start`、`restart` 与 `resume`（保留或切换快照）派发前的 Skill 依赖检查为：已冻结的精确 `catalog_skill_id`（以及历史无该字段的裸引用）能在 Catalog 中解析为 Available Skill，且其存储路径下的 `SKILL.md` 存在并可解析。已冻结的精确 ID（尤其是带 `plugin:` 前缀或对应 Skill 已被删除/卸载时）绝不回退到同名其他 Skill；损坏的冻结 payload 在 `start`、`resume` 与 `snapshot_switch` 时显式失败，不静默回退到未冻结图重解析。不满足依赖时返回现有的 `workflow_skill_not_found` 错误，带节点标题。
4. 每个节点派发时，按冻结记录用同一解析器重新解析当前存储路径，节点 prompt 的 `<required_skills>` 块呈现调用名和剥离 Windows `\\?\` 前缀后的原生 `SKILL.md` 绝对路径（`<workspace_root>` 同步采用原生路径分隔符）；原工作树相对 `packagePaths` 不再写入也不再持久化。运行中途 Skill 被卸载时，该节点以 `MissingRequiredSkill` 节点失败结束（持久化为兼容值 `missing_skill_materialization`，不可续跑），不回退到其他同名 Skill 或旧路径，与预检命令返回的 `workflow_skill_not_found` 区分开。运行中途升级时，后续节点读到新内容。
5. `<required_skills>` 块的说明要求 Agent 在分析任务前读取并遵循每个必需 `SKILL.md`；**仅当 Agent 插件声明已接入原生 Skill 时才在块前附加前导 `/调用名`**（声明机制由 `ora-space/circus#1511` 定义）。由于当前没有插件消费 `_meta["ora/skillDirectories"]`，实现不附加前导 `/调用名`。
6. Agent 不再声明 Skill 交付能力；任何 Agent 都可以按路径读取 Skill，原先“Agent 不支持工作流 Skill”的启动错误删除。
7. 同一运行中两个不同 Skill 解析出相同归一化调用名（例如 `qa_review` 与 `qa-review`）时，仍在启动时拒绝，因为前导斜杠调用无法区分它们；同一 Skill 在多处复用允许通过。
8. 升级前创建的运行，其冻结记录里不含 `catalog_skill_id` 且带有工作树相对 `packagePaths` 时，照常读取并忽略 `packagePaths`，按既有 skill ID 解析。

**与 `approved` 决策的偏离及原因**：`approved` 版 D8.4 原先要求固定保留前导 `/调用名`。实现验收时确认当前没有任何 Agent 插件消费 `_meta["ora/skillDirectories"]`，如果无条件前置 `/调用名`，Claude Code 等 CLI 会把不认识的 `/x` 当成未知命令处理（见 `ora-space/circus#1476` owner 结论），因此收紧为“仅在插件声明已接入原生 Skill 时才前置 `/调用名`”，待 `ora-space/circus#1511` 接入原生 Skill 后再按插件能力开启。

为什么不冻结内容：冻结内容要求存储按 revision 分目录保留，等于把物化以另一种形式带回来。工作流冻结的是“这个节点
必须用哪个 Skill”，Skill 的内容随用户安装的版本演进，与普通会话一致。

### D9：升级时预备份、清理与账本完全匹配的旧受管目录并不可逆退役 Effect 表

首次启动删除 Effect 的版本时，桌面端 `Backend::open` 与开发期 `cargo xtask reconcile-migrations` 在执行数据库迁移前按固定顺序完成**备份、清理与迁移**（先生成自包含快照再执行任何清理或不可逆迁移，确保清理/迁移异常或用户回退时仍有完整的升级前状态）：

1. **升级前数据库自动备份**：只要数据库已初始化（存在 `migrations` 表）且仍存在 `effect_managed_items` 与 `effect_resources` 表（不依赖 `migrations` 中是否已有 `version='0025'` 行），先以 SQLite `VACUUM INTO` 写入临时文件并用 `rename_with_retry` 原子替换 `<data_dir>/ora.sqlite3.pre-0025.bak`。每次处于 pre-0025 状态的启动都会刷新该备份，不信任已有旧备份或 0 字节文件；若清理残留 `.tmp`、`VACUUM INTO` 或原子替换失败，保留既有备份文件（如存在），并使 `Backend::open` 直接返回 `PreMigrationBackupFailed` 错误、不执行迁移。全新安装与已完成 0025 的库跳过备份。
2. **一次性清理旧受管 Skill 目录**（仅在 Effect 表仍存在时读取账本）：
   - Workspace 根目录不存在、目标目录或任一祖先目录为符号链接/junction、目录不存在时跳过。
   - 目录内 `.ora-managed.json` 的标记 `schema_version`、`resource` 身份、`managed_identity`，以及目录当前 fingerprint 与账本记录完全匹配时，删除该目录。
   - 其他情况一律保留：没有标记、标记损坏或不匹配、内容被用户改过（fingerprint 漂移）、不在账本中的目录，以及未终结 Operation 的 staging 或 backup Artifact。父目录（如 `.claude/skills`）即使变空也保留。
   - 单个目录校验或删除失败时记录 Workspace 相对路径并继续，不阻止启动也不阻塞后续有效目录的清理。
   - 清理在迁移前中断时，下次启动重做；已删除的目录跳过，重复执行结果相同。
3. **前向不可逆迁移 `0025`**：从 `effect_revisions` 回填 `skills.package_root` 后，删除全部 `effect_*` 表、触发器与索引。迁移完成后账本不复存在，清理与备份不再运行。`0025` 的持久化降级语句通过事务内 `RAISE(ABORT, ...)` 明确拒绝降级并回滚，保留 `0025` 迁移状态，确保旧版尝试打开失败后新版仍可正常启动。

只删除完全匹配的目录，延续了 Effect 的 ownership 规则：只有账本与标记共同证明是 Ora 写入且未被修改的内容，Ora
才有权删除。清理必须先于 `0025` 迁移，因为迁移会删除 Effect 表，账本是判断受管目录归属的唯一依据；单个目录删除失败不重试，因为重试需要保留账本（即保留 Effect 表），残留的旧副本只是过期内容，用户可手动删除。每次 pre-0025 启动刷新备份是因为“回退旧版 → 在旧版写入数据 → 再次升级”时，沿用旧备份会导致回退时丢失旧版期间的数据。

## 不变量

1. Ora 不为 Skill 在任何 Workspace 内创建、复制、链接或修改文件；唯一例外是 D9 的一次性删除。
2. 清单条目只来自当前 Available 的 Skill，路径是其存储位置下 `SKILL.md` 的绝对路径（已剥离 Windows `\\?\` 前缀）。
3. 一个 Live Session 中，初始全量与之后各次增量累积得到的路径集合，等于最近一次发送 prompt 时的当前清单路径集合；整份清单读取失败时不更新该集合，单条目读取失败时不误发移除或新增。
4. 路径集合不变时不产生任何清单块；路径变化时下一条用户 prompt 必须携带对应增量。
5. 已告知清单只在内存中；actor 新建、provider session rebuild 或 OpenCode 父会话报告压缩完成后，下一条用户 prompt 携带全量清单；MCP refresh 的 `session/load` 回放不清空它。
6. 清单块不放在前导斜杠命令块之前，也不显示为用户输入。
7. 发送 prompt 不等待任何 Skill 状态；Skill 变化不占用 Agent Session Barrier，不重启 Agent，不 reload 会话。
8. Node host 的清单和可读根为空，Node 会话的 ACP 请求与 prompt 与本决策前一致。
9. 额外可读目录只在 Agent 声明能力时发送，缺少能力不使 setup 失败；清单为空或读取失败时不带 `_meta["ora/skillDirectories"]`。
10. 工作流运行只冻结 `{ node_id, skill_id, invocation_name }`；同名裸 Skill 名匹配多条 Catalog 记录时以 `workflow_skill_ambiguous` 拒绝；节点派发时按精确 `catalog_skill_id` 解析当前存储路径，绝不回退同名 Skill，损坏的冻结 payload 显式失败。
11. 升级清理只删除标记、`resource`、`managed_identity` 与 fingerprint 都与账本一致且目标及祖先目录不含符号链接/junction 的目录，其余内容不变；pre-0025 迁移前先以 `effect_*` 表存在为条件完成原子备份与清理，备份失败则不执行迁移，且 `0025` 拒绝降级。
12. Host 不调用 `effect/*`；插件注册中的 `effectResources` 被忽略，不导致连接失败。

## 为什么不是这些替代方案

| 方案 | 优势 | 否决理由 |
| --- | --- | --- |
| 保留 Effect，把复制改为符号链接或 junction | 不占磁盘，更新即时可见 | Windows 符号链接需要权限，junction 有删除穿透和同步盘、杀软兼容问题；收敛、账本和恢复的复杂度都还在 |
| 保留复制物化，去掉账本和收敛 | 实现更简单，Agent 原生发现不受影响 | 仍写 Workspace，仍要处理句柄占用和副本过期；没有账本就无法安全清理，违反不覆盖用户内容的约束 |
| 只用提示词，不把存储根交给插件 | 协议不变 | Claude Code 失去原生 Skill 工具和斜杠入口；Agent 读取 Workspace 外的路径还可能被权限确认或沙箱拦截 |
| 每条 prompt 都重发全量 | 不需要已告知清单，压缩后也不会丢 | 改变 prompt 前缀使 prompt cache 失效，每轮付出整份清单的 token |
| Skill 变化时 `session/load` 重建会话 | 可以随 setup 一并刷新原生接入 | 需要 Agent 支持 load，会在空闲点打断会话；Skill 变化不需要 Agent 丢弃状态 |
| 可读根按每个 Skill 目录传 | 读取范围最小 | 会话中途安装的 Skill 不在已授权范围内，要么重建会话，要么让用户逐次确认 |
| 工作流冻结 Skill 内容 | 运行可复现 | 存储要按 revision 分目录保留，等于半个物化；与普通会话读取最新内容的语义不一致 |
| 升级时不清理旧目录 | 零风险，不删除任何东西 | Agent 仍会从旧目录发现过期副本，与清单重复或冲突 |
| 升级时删除所有已知发现目录（如 `.claude/skills`） | 清理彻底 | 会删除用户和其他工具放入的内容，违反只删除可证明归属内容的规则 |
| 注入放在 desktop backend | Node 完全不受影响 | 聊天与工作流会话都在 runtime 的 actor 中，已告知清单必须跟随 actor 生命周期；放在 backend 要复制这套判断 |
| 保留已有 `pre-0025` 备份不刷新 | 升级启动略快 | 回退旧版写入新数据后再升级再回退会丢失旧版期间数据；必须每次 pre-0025 启动原子刷新 |

## 风险与迁移

- **旧 Agent 插件与新版插件发版约束**：仍声明 `effectResources` 的旧插件继续可用（D1），其原生发现目录不再出现 Skill，靠提示词清单交付。反之，移除 `effectResources` 的新版 Agent 插件**不得早于包含本决策实现的 Desktop 正式版发布**：旧 Desktop 只靠 Effect 物化，配新版插件时 `verify_agent_contract` 仍会通过但 Skill 整体消失。按 `ora-space/circus#1582` 的结论，插件不设最低宿主版本机制：manifest 不加版本字段，marketplace 不做版本门控，只靠这条发版顺序约束和插件的发版说明。发版说明写明需要的 Desktop 不低于包含 desktop#827 的版本，旧 Desktop 上没有 Skill，要先升级 Desktop。代价是发版顺序靠人守住，不升级 Desktop 的用户装上新版插件仍会静默失去 Skill。
- **旧 Workspace 副本**：只按 D9 清理。不能改为按目录名或内容相似度删除，因为 observation 不证明 ownership。
- **旧工作流运行**：冻结记录照常读取（D8 第 8 条），已物化路径被忽略，不会指向已清理的目录。
- **Effect 表与 `0025` 不可逆降级**：升级前在数据目录自动备份 `ora.sqlite3.pre-0025.bak`，`0025` 不可逆，旧版 Desktop 无法直接打开升级后的库；该备份只包含升级前的数据库状态，不包含升级后的写入，具体回退恢复步骤由发版说明给出。
- **预备份失败挡住启动**：`pre-0025` 备份失败会中止启动，且当前启动窗口没有 UI 提示，见 `ora-space/circus#1521`。

## 本决策未解决的问题

- **跨 Agent 压缩信号**：OpenCode 父会话 `opencode/compaction` 已落地，但 Claude Code 尚无压缩通知信号，压缩后可能失去清单直至下次冷启动。
- **Claude Code 原生 Skill 接入**：当前 Agent 插件仅转发 `additionalDirectories` 与 `_meta["ora/skillDirectories"]`，尚未接入 Claude Code 原生 Skill 工具与 `/skill名` 入口，原生 Skill 接入与声明机制由 `ora-space/circus#1511` 跟进。
- **工作流编辑器存 catalog ID**：当前编辑器仍按裸名称引用，遇同名多候选以 `workflow_skill_ambiguous` 拒绝；编辑器改存 catalog ID 及同名歧义处理由 `ora-space/circus#1527` / `ora-space/circus#1563` 决策跟进。
- **包目录不可读的细粒度区分**：包目录本身不可读时目前仍被 `Path::is_file()` 当作不存在（见 `ora-space/circus#1533`）。
- **清单预算**：Skill 很多时清单可能很长。截断、分页或按相关性选择会改变“Agent 知道全部已安装 Skill”的承诺，需要单独决策。
- **按 Workspace 或会话选择 Skill**：本文中已安装 Skill 对所有会话可见；选择与排除需要新的用户意图和持久化模型。
- **Skill 隔离**：清单与可读根不是安全边界，Agent 能读取所有已安装 Skill。严格隔离需要新的授权与文件可见性模型。
- **cloud v1 与 v2-server**：不在本文范围，是否统一另议。

## 落地顺序

1. **ADR 评审**：标记 `approved`，在 [`test-cases/desktop/core/agent/session/skill/`](../../../../../../test-cases/desktop/core/agent/session/skill/) 建立核心用例。
2. **desktop 交付与退役同一版本发布**：runtime 扩展点、清单前缀与增量、ACP setup 字段、工作流按存储路径解析、pre-0025 自动备份与受管目录清理、schema `0025` 删除 Effect 表、删除 Effect 模块与前端门禁、`@ora-space/plugin-sdk` 去掉 `effectResources`（升至 `0.12.0`）。交付与退役必须在同一版本，该步已随 [desktop#827](https://github.com/ora-space/desktop/pull/827) 合并。
3. **Agent 插件同步**：移除 `effectResources`、`effect/*` handler 与协调器；由于旧 Desktop 只靠 Effect 物化，**移除 `effectResources` 的新版 Agent 插件不得早于包含第 2 步实现的 Desktop 正式版发布**。
4. **specs 落地与删除 Effect 用例**：将本文标为 `implemented` 并按最终实现同步正文，回填核心用例证据，并一并删除原 `test-cases/desktop/core/effect/` 及其引用。

保持不变的行为：Skill Catalog 的增删改与导入、Skill 插件的包格式与发现、Skill Settings 的启动环境、MCP 的会话交付、Node 会话。已改变的行为：Workspace 不再接收 Skill 目录，发送不再等待 Skill，会话通过清单得知 Skill，工作流节点读取 Skill 的当前内容。
