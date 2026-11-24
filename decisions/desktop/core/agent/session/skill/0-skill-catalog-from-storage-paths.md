---
status: approved
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

本文为 `approved`：评审已接受下文决策，desktop 实现尚未开始，由 ora-space/circus#1460 承接；对应的决策记录是
ora-space/circus#1458。核心用例见
[`test-cases/desktop/core/agent/session/skill/`](../../../../../../test-cases/desktop/core/agent/session/skill/)，
尚无实现证据，全部义务标为 `Missing`。

- **涉及**：desktop 的 `crates/agent-runtime`（host 扩展点、清单前缀与增量、ACP setup 字段）、`crates/backend`
  （desktop host、工作流 Skill 解析、升级清理）、`crates/db`（删除 Effect 表的前向迁移）、`apps/desktop`（删除
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

上文“整体退役”所列模块全部删除；新增一个前向迁移删除全部 Effect 表，不改写旧迁移文件。迁移编号按 desktop main
的顺序顺延。删除后：

- 发送 prompt 不读取任何 Skill 就绪状态，前端没有 Skill 同步门禁、冲突横幅和恢复入口。
- Host 不调用 `effect/*` 方法。旧版 Agent 插件在注册时仍可能携带 `effectResources`，host 忽略该字段，不使连接失败；
  这样新 desktop 可以继续使用尚未升级的 Agent 插件，代价只是该插件原生发现目录里不再出现 Skill。
- `packages/plugin-sdk` 去掉 `effectResources` 与 `effect/*` 处理接口，Agent 插件删除对应 handler。

这样选的原因：Effect 的复杂度来自“安全地改写 Workspace 内的共享目录”。D2 之后 Ora 不再写 Workspace，没有需要收敛的
外部状态，保留通用 Effect Core 只会留下没有消费者的状态机、表和恢复入口。将来若出现新的 Workspace 副作用需求，
应基于它自己的约束另行决策，而不是为假想的消费者保留这套机制。

### D2：Skill 存储位置就是交付位置

清单条目指向 Skill 存储位置下的 `SKILL.md`，Ora 不复制、不链接、不在 Workspace 中写任何 Skill 相关文件，也不需要
Git exclude。条目只来自当前 Available 的 Skill：本地 Skill 的正式目录存在且 `SKILL.md` 可解析，插件 Skill 的
package root 按静态 Skill 包 D6 重新校验通过。Unavailable 的 Catalog 行不进入清单。

| 来源 | 存储位置 | 路径何时变化 |
| --- | --- | --- |
| 本地导入 | `<应用数据目录>/atoms/skills/<name>/` | 改名时目录随之改名，路径变化；只改内容时不变 |
| 插件提供 | `<Ora 主目录>/plugins/installed/<namespace>/<identifier>/<version>/assets/<skill>/` | 插件升级到新版本时路径变化 |

规则：

1. 路径使用当前平台的原生绝对路径形式，原样呈现，不做转义或缩写。两个根目录由用户机器决定，可能含空格或非 ASCII
   字符。
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
host 上下文块，内容为：

- 一句说明：以下是可用的 Skill，任务与某个 Skill 的描述相符时，先读取它的 `SKILL.md` 并遵循其中说明；
- 每个条目的名称、描述和 `SKILL.md` 绝对路径，按名称的 ASCII 小写、再按路径排序；
- 相对路径按 `SKILL.md` 所在目录解析的规则。

示例（仅说明语义，不是格式契约；实际路径是展开后的绝对路径）：

```text
<available_skills>
Read a skill's SKILL.md before using it. Relative paths inside SKILL.md resolve against its directory.
- review: Review a change against the team checklist.
  <应用数据目录>\atoms\skills\review\SKILL.md
- review: Security-focused review.
  <Ora 主目录>\plugins\installed\acme.3f9a1c2d\sec-pack\1.2.0\assets\review\SKILL.md
</available_skills>
```

规则：

1. 清单块位于用户内容之前。例外是首个文本块以斜杠命令开头（用户输入的 `/命令`，或工作流节点前导的 Skill 调用）：
   该块保持在最前，清单块紧随其后，避免破坏 Agent CLI 对前导斜杠命令的解析。
2. 清单块是 host 上下文，不是用户输入；界面不把它显示成用户消息的一部分。
3. 清单为空时不加块，也不记录任何内容；已告知清单视为空集合。
4. 交互式聊天和工作流节点会话都使用同一套前缀逻辑；工作流节点在此之外仍有自己的必需 Skill 块（D8）。
5. 本文不截断清单。描述本身受 4096 bytes 上限约束；清单的总体预算留作未决项。

只列名称、描述和路径属于渐进加载：Agent 只在需要时读取正文，不把全部 Skill 正文塞进上下文。

### D4：会话中途的变化以增量告知，记录丢失后重发全量

Runtime 在内存中为每个 Live Session 保存已告知清单。之后每条用户 prompt 发送前，runtime 读取当前清单，按
`SKILL.md` 路径与已告知清单比较：

| 比较结果 | prompt 中加入的内容 |
| --- | --- |
| 路径集合相同（包括名称、描述或内容在原路径上变化） | 不加任何块 |
| 有新增路径 | 增量块列出新增条目的名称、描述和路径 |
| 有移除路径 | 增量块列出移除条目的名称和路径，说明它们已不可用 |
| 两者都有（如插件升级、本地 Skill 改名） | 同一个增量块同时列出新增与移除 |

增量块的位置规则与 D3 相同。prompt 请求已发给 Agent 后，runtime 才把已告知清单更新为本次比较所用的当前清单；
prompt 在进入 Agent 前失败（例如 MCP refresh 失败被挡在 admission 前）时不更新，下一次发送重新比较。

已告知清单这份记录只存在于内存，不写入 Ora Session 或 SQLite；清单块本身只是随 prompt 发给 Agent 的内容。以下
情况把记录视为不存在，下一条用户 prompt 携带全量清单：

- Live Session 的 runtime actor 新建，包括应用重启后的冷启动 `session/load`，以及停止的 Ora Session 再次打开；
- Agent 插件报告该会话的上下文已压缩。

同一 Live Session 内为了 MCP refresh 发起的 `session/load` 不清空记录：provider 会话身份不变，已告知内容仍在 Agent
的会话上下文里。

为什么不每条都重发全量：清单放在每条 prompt 前会改变 prompt 前缀，使 Agent 侧的 prompt cache 失效，并为每轮付出
整份清单的 token。为什么不在变化时 `session/load` 重建会话：它要求 Agent 支持 load，会在空闲点打断会话，而 Skill
变化并不需要 Agent 丢弃任何运行时状态。记录放在内存而不持久化，是因为持久化的记录无法证明重启后的 Agent 上下文
仍包含它，重发全量才是安全的重置点。

压缩信号来自 Agent 插件。Ora 现在没有任何压缩信号的通道；插件以可选通知报告压缩，wire 形状随插件协议改动确定。
不报告压缩的 Agent，在压缩后可能失去清单，直到下一次冷启动；这是已知限制，见“本决策未解决的问题”。

### D5：注入放在 `ora-agent-runtime`，由 host 扩展点提供内容

Runtime 负责首条前缀、增量计算、已告知清单和 ACP setup 字段；内容由 host 通过扩展点提供，例如：

| 扩展点 | desktop host | ora-node host |
| --- | --- | --- |
| `skill_catalog(workspace)` | 当前全部 Available Skill 的条目 | 空 |
| `extra_read_roots()` | Skill 可读根 | 空 |

扩展点名称是示意，签名以实现为准。`workspace` 参数为将来按 Workspace 选择 Skill 预留；本文中 desktop 对所有
Workspace 返回同一份清单。

Node host 返回空，所以 Node 会话没有前缀、没有增量、没有额外可读目录，ACP 请求与 prompt 和本决策前一致。ora-node
只去掉对 `ora-effect` 的依赖和 `replace_agent_effect_declaration` 空实现。

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
3. **可选的原生 Skill 接入**：同一 setup 请求以 Ora 命名空间的 `_meta` 字段携带当前清单中各 Skill 的包目录，例如
   `_meta["ora/skillDirectories"]`（键名随插件协议改动确定）。插件可以把它们接到 Agent 的原生 Skill 机制（例如
   Claude Code 的原生 Skill 工具和 `/skill名` 入口），也可以忽略。插件不得写入这些目录。清单为空时不带该字段。
4. 原生接入只反映 setup 时刻的清单；会话中途的变化只通过 D4 的提示词增量告知。原生机制与提示词清单指向同一批路径，
   不会产生两份内容不同的 Skill。

为什么提示词是基线、原生接入可选：各 Agent 的原生 Skill 机制差异很大，有的没有；提示词对所有能读文件的 Agent 都
成立。只用提示词会让 Claude Code 失去原生 Skill 工具和斜杠入口，可靠性略低，因此保留可选接入，代价是协议多一个
可选字段。

### D7：Skill 变化不再经过 Agent Session Barrier

Skill 安装、更新、卸载、改名不占用 Agent Session Barrier，不重启 Agent，不 reload 任何会话，也不打断进行中的 turn。
Barrier 的原因只剩 MCP refresh：Effect mutation 和 Effect reactivation 引起的 Agent replacement 随 Effect 一起删除。
MCP 的 Desired/Active 收敛、prompt admission 和全量成败规则不变。

### D8：工作流只冻结 skill ID 和调用名

1. 运行创建时冻结每个节点启用的 Skill：节点 ID、skill ID 和调用名。不冻结内容、版本或路径，也不按 revision 保留
   存储目录。
2. `start`、`restart` 与 `resume`（包括切换快照）派发前的 Skill 依赖检查为：skill ID 能在 Catalog 中解析为一个
   Skill，且其存储路径下的 `SKILL.md` 存在并可解析。不满足时沿用现有的 Skill 依赖错误，带节点标题。
3. 每个节点派发时，按 skill ID 用同一解析器重新解析当前存储路径，节点 prompt 的必需 Skill 块呈现调用名和
   `SKILL.md` 绝对路径；原 `packagePaths` 改为派发时计算的存储位置绝对路径，不再持久化。运行中途 Skill 被卸载时，
   该节点以同一 Skill 依赖错误失败，不回退到其他同名 Skill 或旧路径。运行中途升级时，后续节点读到新内容。
4. 必需 Skill 块的说明从“使用本次运行中的物化副本”改为“读取并遵循上述 `SKILL.md`”。前导的 `/调用名` 保留，供接入了
   原生 Skill 的 Agent 直接调用。
5. Agent 不再声明 Skill 交付能力；任何 Agent 都可以按路径读取 Skill，原先“Agent 不支持工作流 Skill”的启动错误删除。
6. 同一运行中两个不同 Skill 解析出相同调用名时，仍在启动时拒绝，因为前导斜杠调用无法区分它们。
7. 升级前创建的运行，其冻结记录里的工作树相对 `packagePaths` 照常可读但被忽略，节点派发按 skill ID 解析。

为什么不冻结内容：冻结内容要求存储按 revision 分目录保留，等于把物化以另一种形式带回来。工作流冻结的是“这个节点
必须用哪个 Skill”，Skill 的内容随用户安装的版本演进，与普通会话一致。

### D9：升级时只删除与账本完全匹配的旧受管目录

首次启动删除 Effect 的版本时，在删除 Effect 表的迁移执行之前，host 对 Effect 账本中的每个受管条目做一次清理：

1. Workspace 根目录不存在、路径校验失败（含符号链接或 junction）、目录不存在时跳过。
2. 目录内 `.ora-managed.json` 的标记身份、native identity，以及目录当前 fingerprint 与账本记录完全匹配时，删除该
   目录。
3. 其他情况一律保留：没有标记、标记不匹配、内容被用户改过（fingerprint 漂移）、不在账本中的目录，以及未终结
   Operation 的 staging 或 backup Artifact。父目录（如 `.claude/skills`）即使变空也保留。
4. 单个目录删除失败（例如 Windows 句柄占用）时记录 Workspace 相对路径并继续，不阻止启动。
5. 清理完成后执行删除 Effect 表的迁移。清理在迁移前中断时，下次启动重做；已删除的目录按第 1 条跳过，所以重复执行
   结果相同。迁移完成后账本不复存在，清理不再进行。

只删除完全匹配的目录，延续了 Effect 的 ownership 规则：只有账本与标记共同证明是 Ora 写入且未被修改的内容，Ora
才有权删除。不清理的代价是 Agent 仍会从旧目录发现过期副本，与清单中的路径重复甚至冲突。删除失败不重试，是因为
重试需要保留账本，即保留 Effect 表；残留的副本只是过期内容，用户可以手动删除。

## 不变量

1. Ora 不为 Skill 在任何 Workspace 内创建、复制、链接或修改文件；唯一例外是 D9 的一次性删除。
2. 清单条目只来自当前 Available 的 Skill，路径是其存储位置下 `SKILL.md` 的绝对路径。
3. 一个 Live Session 中，初始全量与之后各次增量累积得到的路径集合，等于最近一次发送 prompt 时的当前清单路径集合。
4. 路径集合不变时不产生任何清单块；路径变化时下一条用户 prompt 必须携带对应增量。
5. 已告知清单只在内存中；actor 新建或 Agent 报告压缩后，下一条用户 prompt 携带全量清单。
6. 清单块不放在前导斜杠命令块之前，也不显示为用户输入。
7. 发送 prompt 不等待任何 Skill 状态；Skill 变化不占用 Agent Session Barrier，不重启 Agent，不 reload 会话。
8. Node host 的清单和可读根为空，Node 会话的 ACP 请求与 prompt 与本决策前一致。
9. 额外可读目录只在 Agent 声明能力时发送，缺少能力不使 setup 失败。
10. 工作流运行只冻结 skill ID 与调用名；节点派发时按 skill ID 解析当前存储路径，解析不到即以 Skill 依赖错误失败。
11. 升级清理只删除标记、native identity 与 fingerprint 都与账本一致的目录，其余内容不变，重复执行结果相同。
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

## 风险与迁移

- **旧 Agent 插件**：仍声明 `effectResources` 的插件继续可用（D1），其原生发现目录不再出现 Skill；提示词清单让它
  照常使用 Skill。插件升级后再按 D6 选择是否接原生机制。
- **旧 Workspace 副本**：只按 D9 清理。不能改为按目录名或内容相似度删除，因为 observation 不证明 ownership。
- **旧工作流运行**：冻结记录照常读取（D8 第 7 条），已物化路径被忽略，不会指向已清理的目录。
- **Effect 表**：只能由前向迁移删除；清理必须先于迁移，否则失去判断归属的唯一依据。

## 本决策未解决的问题

- **压缩信号**：插件报告上下文压缩的 wire 形状随插件协议改动确定；不报告压缩的 Agent 在压缩后可能失去清单，直到下次
  冷启动。是否需要 Ora 主动探测压缩，取决于各 Agent 是否暴露该事实，超出本文范围。
- **清单预算**：Skill 很多时清单可能很长。截断、分页或按相关性选择会改变“Agent 知道全部已安装 Skill”的承诺，需要
  单独决策。
- **按 Workspace 或会话选择 Skill**：本文中已安装 Skill 对所有会话可见；选择与排除需要新的用户意图和持久化模型。
- **Skill 隔离**：清单与可读根不是安全边界，Agent 能读取所有已安装 Skill。严格隔离需要新的授权与文件可见性模型。
- **cloud v1 与 v2-server**：不在本文范围，是否统一另议。

## 落地顺序

1. **本 ADR 评审**（已完成）：标记 `approved`，在
   [`test-cases/desktop/core/agent/session/skill/`](../../../../../../test-cases/desktop/core/agent/session/skill/)
   补充核心用例，覆盖首条全量、增量与重置、斜杠命令位置、可读目录能力协商、Node 不变、工作流解析、升级清理与旧
   `effectResources` 兼容。
2. **desktop 交付与退役同一版本发布**：runtime 扩展点、清单前缀与增量、ACP setup 字段、工作流按存储路径解析、升级
   清理与删除 Effect 表的迁移、删除 Effect 模块与前端门禁。交付与退役必须在同一版本：只有交付会让 Agent 同时看到
   副本和清单，只有退役会让 Skill 消失。完成条件是新会话的 Workspace 中没有任何 Ora 写入的 Skill 文件，Agent 能按
   清单读取 Skill，旧受管目录按 D9 清理。
3. **插件 SDK 与 Agent 插件**：去掉 `effectResources` 与 `effect/*` handler，按需接原生 Skill 与可读目录。由于 host
   忽略旧字段，这一步可以晚于第 2 步独立发布。
4. **实现后**：按最终实现同步本文并标记 `implemented`，同时更新本文核心用例的证据。`test-cases/desktop/core/effect/`
   下的 Effect 核心用例在这一步删除：desktop 中引用它们的测试注释与文档随 Effect 代码一并删除，两边同一轮完成，
   避免规格先删、代码仍引用。在此之前这些用例保留，作为现有 Effect 实现的证据。

保持不变的行为：Skill Catalog 的增删改与导入、Skill 插件的包格式与发现、Skill Settings 的启动环境、MCP 的会话交付、
Node 会话。从第 2 步开始改变的行为：Workspace 不再接收 Skill 目录，发送不再等待 Skill，会话通过清单得知 Skill，
工作流节点读取 Skill 的当前内容。
