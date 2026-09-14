---
status: proposed
---

# 插件集合包：Plugin Pack 是非运行时插件，可整体安装与卸载

Ora 的 marketplace 新增一种条目形态：Plugin Pack（插件集合包）。pack 用一份 `kind = "pack"` 的
清单声明一组成员插件（`[[pack.members]]`），自身不携带任何运行时贡献物与 `assets/` 目录。用户
安装一个 pack 时，宿主在 preflight 通过后按声明顺序逐个安装成员、复用既有单插件安装链路，并把
每个新安装的成员登记进 pack 的数据目录；pack 自身作为无进程的非运行时插件随之落盘，进入已安装
模型——前端可以像对待其他插件一样，对 pack 独立地整体安装与整体卸载，而卸载 pack 只卸载登记
在案的成员。同时新增 release 形态可选字段 `marketplace_visible`（缺省 `true`）：声明为 `false`
的条目不进入派生索引、不在 marketplace 前端独立展示，但仍保留在源仓库中、可按 identifier 解析、
可被 pack 安装，安装后是真实 Installed Plugin。

本 ADR 处于 `proposed`：决策与需求尚未确认，正文保留示例与推测以帮助评审纠偏。文中的 TOML 与
契约形状只用于固定语义，不构成实现契约；按 ADR 生命周期，核心测试用例在本决策 `approved` 后
补充。涉及模块为 `ora-plugin-manifest`（清单 schema）、`ora-plugin-registry`（索引构建）、
`crates/backend`（安装与卸载编排）、`ora-plugin-manager`（成员安装复用）、`ora-plugin-lifecycle`
（per-plugin 数据目录）、`ora-contracts` 与 `packages/app-shell`（聚合结果与前端呈现）。

兼容性边界：pack 条目在旧宿主上整条消失（未知 kind 硬拒绝的既定行为），隐藏成员在旧宿主上
仍然可见（未知字段忽略的既定保护），两者的不对称是接受的代价而非需要迁移的状态（见 D9）。
本决策不承担插件依赖求解与安装前成员预览——各自见"本决策未解决的问题"；成员的 `agents`
适配范围声明被评审确认为必要能力，但同样留待后续追加 ADR，不在本决策展开。

## 术语与边界

- **Listing（marketplace 条目）**是 marketplace 源仓库 `registry/**/orax.toml` 中的一份
  release 清单；**派生索引（Registry Index）**是宿主扫描全部源后写出的展示缓存
  `registry_index.json`；**Installed Plugin**是 `installed/<namespace>/<identifier>/<version>/`
  下真实落盘的包。三者是不同层，本文显式区分。
- **Plugin Pack**是 `kind = "pack"` 的插件；**成员**是它的 `[[pack.members]]` 声明的插件
  identifier；**隐藏条目**是声明 `marketplace_visible = false` 的 listing。
- **成员登记（membership record）**是 pack 数据目录中记录"由该 pack 新安装的成员"的登记，
  是 pack 卸载时确定卸载集合的唯一事实来源。
- **Agent Plugin**是 `kind = "agent"` 的已安装插件；其运行期身份是完整 canonical Plugin ID
  `<namespace>/<identifier>`（来源派生 Namespace 决策 D8），会话把它持久化在 `agent_cli`。

一个 pack 条目与其隐藏成员的示例：

```toml
# registry/o/ora-space.python-extension-pack/orax.toml
resolver = 1
identifier = "ora-space.python-extension-pack"
title = "Python Extension Pack"
kind = "pack"
version = "0.1.0"
description = "Python 开发所需的技能与 MCP 集合"
url = "…"        # pack 包只含 orax.toml，体积极小且全平台一致
sha256 = "…"

[[pack.members]]
identifier = "ora-space.python-core"

[[pack.members]]
identifier = "ora-space.python-mcp"

# registry/o/ora-space.python-core/orax.toml（隐藏成员，节选）
# resolver = 1 / identifier = "ora-space.python-core" / kind = "skill"
marketplace_visible = false
url = "…"
sha256 = "…"
```

## 与相邻决策的边界

本文固定 Plugin Pack 能力本身的系统承诺，沿用既有分工而不复制权威规则：

- `orax.toml` 字段级 schema（`pack` 进入 kind 封闭集合、`[[pack.members]]` 的类型策略与双形态
  约束、`marketplace_visible` 的类型策略）的权威定义属于
  [插件清单根决策](../../manifest/0-versioned-dual-form-manifest.md)，需以该链的日期决策登记；
  本文 D1、D3 只固定能力视角的规则。
- 索引构建与缓存机制的权威定义属于
  [插件分发根决策](../../registry/0-registry-distribution.md)，可见性过滤需以该链的日期决策
  登记；本文 D3 只固定过滤语义与"resolve 不受影响"的承诺。
- [来源派生 Namespace 决策](../../registry/20260831-source-derived-namespace.md)不变：namespace
  由宿主派生并持久化、agent 身份是 canonical Plugin ID（该决策 D8）；本文在其之上引用。
- per-plugin 数据目录（`<data-dir>/plugins/data/<namespace>/<name>/`，按插件身份键控、随卸载
  删除）是 `ora-plugin-lifecycle` 的既有约定；本文 D6 只固定 pack 对它的复用方式，不改变其
  所有权与生命周期规则。

## 决策

### D1：pack 是 `kind = "pack"` 的非运行时插件，包内不携带 `assets/`

`kind = "pack"` 进入清单 kind 封闭集合，与其他 kind 一样拥有完整的插件身份：完整 canonical
Plugin ID、SemVer 版本、已安装目录与已安装快照条目。pack 不承载任何运行时贡献物：禁止
`[workbench]`/`[webview]`，不携带 `main.js`，不创建 Deno runtime、不参与 plugin handshake、
不能注册 host methods，也不获得任何插件权限；`[[targets]]` 与 `[artifact]` 对 pack 禁止（与
"只有可能内含原生可执行文件的 kind 允许 target 形态"的既有规则一致），release 形态使用顶层
`url`/`sha256` 通用资产——pack 包内只有一份 `orax.toml`，天然全平台一致。`[[pack.members]]`
对 pack 必需且独占，对其余 kind 禁止，且在两种清单形态下都必须存在：pack 的自述身份就是成员
集合。宿主包校验对 pack 拒绝 `assets/` 目录与 `main.js`："这个插件没有任何运行时内容"必须能
从包内容一眼判定，而不是靠"恰好没写"的组合推断。

pack 之所以是"真插件"而不是编排指令，是因为前端需要能对它独立地整体安装与整体卸载：只有进入
已安装模型，marketplace 的已安装列表、卸载入口与卸载结果才有统一的承载。它的"非运行时"与
Skill、MCP 插件同类——无进程不是无生命周期；差别只在 Skill/MCP 的贡献物在包内，pack 的
"贡献物"是它的成员登记。

让非法组合在解析期失败（pack 带被禁止的 section、非 pack 带 `[[pack.members]]`），保持了
清单根决策 D4"清单不能是半个两种 kind"的承诺。

### D2：成员用裸 `identifier` 声明，并解析在 pack 所属的同一个 Marketplace Source 内

`[[pack.members]]` 的 `identifier` 是 `PluginName` 形态的裸标识符；成员的完整 Plugin ID 由宿主
组装为 `<pack 的 namespace>/<member identifier>`，成员解析永远发生在 pack 自己的源检出内。
V1 不支持跨源成员。

namespace 由宿主从 source canonical URL 派生并在首次绑定时持久化，作者无法预知第三方源派生出
的 namespace（来源派生 Namespace 决策 D2/D3 的同一逻辑）：让 pack 作者书写完整 canonical ID，
等于让不可验证的自述参与引用。裸 identifier 还使 pack 在官方源与 fork 源之间原样可用——这正是
"fork marketplace 仓库后作为自己的源使用"的工作流所要求的。跨源引用需要成员级依赖求解与多源
顺序语义，超出 V1 边界。

### D3：`marketplace_visible = false` 的条目不进入派生索引，但保持可解析、可安装

`marketplace_visible` 是 release 形态的展示意图声明：`false` 的唯一语义是"该条目不作为独立
listing 进入 marketplace 前端的浏览、搜索与详情入口"。它不表示 disabled、不可安装或不可解析——
条目仍在源仓库 registry 中，仍可按 identifier 解析，仍可被 pack 安装，安装后是真实 Installed
Plugin。缺省 `true`；installed 形态出现该字段直接报错（与 `[[targets]]` 不允许出现在 installed
形态同一先例）：可见性是 marketplace listing 的属性，已落盘的包没有展示面。

过滤发生在索引构建期、生成索引条目之前，而不是投影或前端：隐藏条目不可能通过任何派生路径泄漏
给前端，每一层消费者都不需要理解和转发一个可见性标志。索引 schema 不因此新增字段，缓存版本
不变；旧宿主（或升级前）写出的缓存可能短暂包含后来变为隐藏的条目，到下一次同步自然清除——为
一个无害的临时可见性去作废整份缓存、让市场在用户手动刷新前显示为空，不值得。

字段放在清单而不是 registry 侧的旁路元数据里，是因为索引构建与安装 resolve 解析的是同一份
`orax.toml`；旁路元数据会让"列表里看不到"与"安装时读到的声明"成为两个可漂移的事实。命名上
否决 `listed`（会被误读为"是否存在于 registry"——隐藏条目恰恰存在于 registry）与
`hidden = true`（否定式默认要求读者先计算"未写即可见"）。

### D4：安装一个 pack = preflight + 顺序安装成员 + 安装 pack 自身

安装入口复用既有 `installPlugin` 契约（`pluginId` = pack 的 canonical ID）。宿主 resolve 出
清单后发现 `kind = "pack"`，转入编排而不是直接落盘：

1. **preflight**（下载任何字节之前）：成员 `identifier` 去重校验、自引用校验（成员 identifier
   等于 pack identifier）、逐成员在 pack 所属源内解析清单（成员不存在即失败）、成员清单
   `kind = "pack"` 即失败（V1 禁止嵌套）、对每个成员执行纯函数的 release 选择（当前 host triple
   不兼容即失败）。
2. **顺序安装成员**：按声明顺序对每个成员执行既有单插件安装（下载 → SHA-256 → staging → 包
   校验 → 原子提交，全部既有校验不变）。成员已安装（任意版本）则记为 `skipped` 并继续，不报
   错，既有版本不被改动或降级；每个新安装的成员在提交后立即登记（见 D6）。
3. **安装 pack 自身**：全部成员处理完毕后，pack 按既有单插件安装链路落盘到
   `installed/<namespace>/<identifier>/<version>/`，成为真实的 Installed Plugin。
4. **结果**：返回聚合 outcome（见 D5）；进度事件沿用现有 `plugin-install-progress`，以 pack 的
   pluginId 发出，进度按"已完成成员数 + 当前成员内部进度，除以成员总数"聚合。

pack 自身最后安装：pack 的安装完成即代表全部成员就绪，失败的安装不会留下"已安装但成员残缺"的
pack；已安装成员按 D5 保留，重新安装即可补齐（见 D8）。

顺序而非并行安装：复用单插件安装的全部原子性与错误定位，进度与失败一一对应；V1 成员数量小，
并行的收益不值得引入并发编排。

### D5：preflight 失败零下载；成员失败停止且不回滚，结果逐成员可辨

preflight 任一失败即整包失败，返回 typed 错误且不产生任何下载与落盘。成员重复、自引用、成员
不存在、嵌套 pack、成员与当前 host 不兼容，复用或新增封闭的 public error 变体；host 兼容性
判定发生在 preflight 而非首个成员安装中途。

preflight 通过后的成员安装中，任一成员失败：已安装的成员保留（每个成员的安装本身已是原子
事务），剩余成员不再尝试，不自动回滚，pack 自身不落盘。契约 `InstallOutcome` 新增
`PackInstalled { members, skipped, failed }`：`members` 按声明顺序携带已安装成员的
`{ plugin_id, outcome }`（复用既有单插件 outcome 枚举，含命令别名冲突报告），`skipped` 列出
已安装被跳过的成员，`failed` 携带首个失败成员与其错误码。用 `Ok` outcome 如实上报部分成功，
而不是把部分成功伪装成整包失败——调用方需要知道哪些成员已装好，才能决定重试或放弃。

### D6：新安装的成员登记进 pack 的数据目录 `data/<ns>/<identifier>/`

pack 复用既有的 per-plugin 数据目录约定：成员登记写入
`<data-dir>/plugins/data/<pack namespace>/<pack identifier>/`（即插件数据目录
`data/<ns>/<identifier>/`）。登记按插件 ID 去重，每项携带成员的完整 Plugin ID 与安装时的
版本；proposed 阶段以 `members.json` 固定语义，文件名与字段形状是实现细节，不构成契约。
每个由 pack 实际新安装的成员在安装提交后立即登记；因已安装而跳过的成员不新增登记——它们不是
pack 安装的，卸载 pack 时不属于 pack。

登记放在数据目录而不是 pack 安装包目录内：包目录按版本键控、随更新被整体替换，登记必须跨
pack 版本存活；已安装树也不是插件的可写位置，per-plugin 数据目录是唯一按插件身份键控的宿主
管理区域。登记随安装逐步写入，而不是全部完成后一次性写：安装中途失败时，已完成成员的归属
事实不会丢失。

### D7：卸载 pack 只卸载登记的成员，随后移除 pack 自身

卸载入口复用既有插件卸载。卸载 pack 的顺序固定为：读取成员登记 → 卸载 pack 自身（失败即
中止，不动任何成员）→ 逐个卸载登记中仍存在的成员（沿用既有单插件卸载；已被用户单独卸载的
成员直接跳过；单个成员卸载失败不阻塞其余成员，失败成员保留为普通已安装插件并在结果中报告）
→ 删除 pack 数据目录，登记随之消失。

只卸载登记的成员是所有权语义：登记是"这个成员由该 pack 安装"的持久事实。用户在安装 pack
之前就已独立安装的成员（安装时被跳过、未登记）不随 pack 卸载；用户在安装 pack 之后单独卸载
过的成员自然不在登记中残留。登记不随后续 pack 版本的清单变化收缩：成员在某次 pack 更新中
离开 `[[pack.members]]` 不等于所有权转移，它仍会随 pack 卸载移除。

### D8：重新安装 pack 等价于补齐缺失成员

pack 进入已安装模型后，marketplace 已安装列表呈现 pack，卸载入口即 D7 的整体卸载。pack 条目
的 `version` 是成员集合的内容版本，纯展示；重新安装 pack（包括同版本重复执行）合法且等价于
补齐：缺失成员按 D4 安装并登记，已安装成员继续跳过。pack 的版本更新沿用既有更新语义（更高
版本替换包目录、退役旧版本目录），新版本新增的成员在更新编排中补装并登记。成员被单独卸载后
再次安装 pack 会重装该成员——这是期望行为：pack 表达的是"这套集合应该存在"，不是"曾经装过"。

同版本重复执行是对既有安装入口"拒绝无操作"语义的一个显式豁免，仅对 `kind = "pack"` 成立：
pack 的安装动作是成员编排而不仅是制品替换，"已安装"不构成"无事可做"。

### D9：旧宿主对新条目的行为是确定的

- **pack 条目在旧宿主上整条消失**：未知 kind 是清单根决策 D4 的显式硬错误，索引构建按既有
  规则跳过并记 warning。这是封闭 kind 设计的既定代价；pack 消费者需要新版宿主。
- **隐藏成员在旧宿主上仍然可见**：`marketplace_visible` 是未知字段，被旧宿主忽略，条目照常
  进入索引。这是清单根决策 D2"未知字段忽略"对增量 schema 的既定保护。

两者都是既有决策在新条目上的自然投影，本决策不引入版本探测、不承担对旧宿主的迁移义务。

## 为什么不是这些替代方案

| 方案 | 否决理由 |
| --- | --- |
| pack 作为纯编排条目，不落盘、不进入已安装模型（本 ADR 初版方案） | 前端无法对 pack 独立整体卸载；成员归属无处登记，"卸载 pack"无从定义；已安装列表与 marketplace 状态不一致。它省去包格式的优势，不能抵消生命周期缺口 |
| 成员登记写进 pack 安装包目录 | 包目录按版本键控且随更新被替换，登记必须跨版本存活；已安装树也不是插件的可写位置 |
| 卸载 pack 时按当前清单的 `[[pack.members]]` 卸载 | 用户独立安装的成员会被误卸；清单是作者意图、会漂移，登记才是安装时刻的归属事实 |
| 独立 `pack.toml` 与独立 listing 流水线 | 索引扫描、README/logo 约定、resolve 路径、前端 listing 模型都要第二套；两个条目模型漂移后"能上架"与"能安装"再次分叉 |
| 成员引用使用完整 canonical ID | namespace 由宿主从 source URL 派生，作者不可预知；fork 源部署下所有引用都要改写，与清单不声明 namespace 的既定身份边界冲突 |
| `marketplace_visible` 放进 registry 侧旁路元数据，由投影或前端过滤 | 索引与 resolve 读取两个事实来源，可见性声明与安装声明可漂移；每层消费者都要记得尊重标志，遗漏即泄漏 |
| 成员并行安装 | 单插件安装的原子性与错误定位被并发编排稀释；V1 成员量小，收益不成比例 |
| 成员失败自动回滚已装成员 | 单插件安装是独立原子事务；回滚需要新的多包事务模型，且"卸掉刚装好的可用插件"对用户是破坏性行为 |
| 通用 plugin-to-plugin 依赖求解器 | 版本区间、拓扑排序、冲突求解是依赖系统问题；当前需求只是"一次装多个 + 整体卸载"，用编排与登记即可满足，求解器是不受支持的复杂度 |

## 不变量

1. pack 安装后是真实的 Installed Plugin：进入 installed 树与已安装快照，可被列出，可被整体
   卸载。
2. pack 包不携带 `assets/`、`main.js` 或任何运行时贡献物；不创建 runtime、不注册 host
   method、不获得插件进程权限。
3. preflight 未通过时，不发生任何下载，`installed/` 与 pack 数据目录都不发生变化。
4. 成员只在 pack 所属的 Marketplace Source 内解析；V1 不存在跨源成员与嵌套 pack。
5. 已安装的成员在 pack 安装中被跳过且不算失败；其既有版本不被改动或降级，也不新增登记。
6. 任一成员安装失败后，不再尝试后续成员；已安装成员保留，pack 自身不落盘，不回滚。
7. pack 安装的结果必须逐成员可辨（已装/跳过/失败），不存在整体布尔成功。
8. 每个由 pack 新安装的成员在安装提交后登记进 pack 数据目录；成员登记是 pack 卸载集合的唯一
   事实来源。
9. 卸载 pack 恰好卸载登记成员，然后移除 pack 自身与其数据目录；登记之外的已安装插件不受
   影响。
10. `marketplace_visible = false` 的条目不出现在任何派生索引文件中，但始终可按 identifier 在
    其所属源内解析与被 pack 安装。
11. 成员安装复用单插件安装的全部校验（SHA-256、staging、包校验、原子提交），不引入新的落盘
    路径。

## 本决策未解决的问题

- 成员的 `agents` 字段（可选）：评审确认为必要能力——把成员标记为 agent 专属、按已安装的
  agent 插件过滤安装集合。它涉及匹配语义、清单字段策略与"Ora 无全局当前 agent"的落地方式，
  本决策不纳入，留待下一个追加 ADR。
- pack 详情页与"安装前成员预览"：需要把成员的 listing 元数据经新契约暴露给前端，或为隐藏
  条目增加另一条读取路径；留待 V1 之后的 ADR。
- 已安装 pack 的成员分组展示（"随 pack 安装"标记）：需要登记内容经契约进入前端模型，超出
  V1。
- 嵌套 pack、跨源成员、成员版本区间约束：需要依赖求解语义，明确不在 V1。

## 落地顺序

1. **清单与索引层**：`ora-plugin-manifest` 登记 `kind = "pack"`、`[[pack.members]]`、
   `marketplace_visible`（清单链日期决策）；`ora-plugin-registry` 索引构建过滤隐藏条目
   （registry 链日期决策）。完成条件：pack/隐藏条目的解析、索引、resolve 行为有单元测试。
2. **编排与契约层**：`crates/backend` 安装与卸载编排 + preflight + 成员登记 + `PackInstalled`
   outcome；`ora-plugin-lifecycle` 数据目录复用；`ora-contracts` 与 TS 类型再生成。完成条件：
   preflight 失败零下载、成员失败停止不回滚、登记与整体卸载的集成测试。
3. **前端层**：marketplace 卡片对 pack 的呈现、聚合进度、已安装列表中的 pack 与整体卸载入口。
   完成条件：安装与卸载 pack 的端到端用户路径可用。
4. **内容层**：marketplace 仓库提交 `python-extension-pack` 及其成员条目（含
   `marketplace_visible = false` 的隐藏成员）与 README。完成条件：真实源下完成一次完整安装与
   整体卸载。
