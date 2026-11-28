---
status: approved
---

# 插件来源展示：后端下发 Plugin Origin 与查询时计算的市场标签

Desktop 在聊天框 `/` 菜单、设置 → 技能页和插件市场卡片上为每个 Skill / 插件显示来源标签，标签带 tooltip。
来源是插件级属性，分为四类：**用户**（本地 Skill）、**本地包**（本地导入的 `.orax`）、**官方**（默认
marketplace）和**第三方市场**。前三类的标签是固定文案；第三方市场的标签由后端在查询时从 canonical source
URL 计算：默认取最后一个路径段，与其他来源重名时补上从末尾往前第一个不同的段。后端把来源以
`PluginOrigin` 随 `Skill`、`InstalledPlugin`、`AvailablePlugin` 一起下发，前端只渲染，不解析 URL，也不
计算消歧。标签只用于展示，不是身份，新增同名来源后已有标签可能变化；身份始终是 namespace。

本决策为 `approved`：owner 于 2026-11-27 经两轮评审后批准，核心测试用例见
[插件来源展示用例](../../../../test-cases/desktop/plugin/registry/origin-presentation.md)。实现尚未合入
desktop 的 main，用例义务如实标为 `Missing`；实现合入并完成验证后，由落地运行按最终实现同步正文并改为
`implemented`。涉及 desktop 的 `crates/contracts`（`SkillSource`、`InstalledPlugin`、`AvailablePlugin`）、
`crates/domain`（`PluginOrigin` 与来源分类）、`crates/utils`（市场标签计算）、`crates/backend`（按
namespace 反查 canonical URL、三个查询的装配）、`crates/db`（只读 `plugin_source_namespace`，不改
schema）以及 `packages/app-shell`（共用来源标签组件、三处接入、插件详情的来源市场行、市场源变更后的查询
失效、中英文文案）。

- **不改变**：namespace 的派生、绑定与不可变规则，安装目录与 `skills` 表的身份匹配，索引缓存格式，
  `plugin_source_namespace` 的 schema 与只增不删的写入规则。
- **不承担的迁移**：标签不持久化，因此没有存量数据需要迁移；契约字段变化随 desktop 同一版本的前后端一起
  发布，不保留旧形状。
- **不涉及**：工作流编辑器中同名 Skill 的选择与绑定（由[工作流 Skill 绑定改存 Catalog ID](../category/workflow/20261126-workflow-skill-binding-by-catalog-id.md)
  负责），cloud v1 与 v2-server 的界面，以及用户自定义市场显示名（见“本决策未解决的问题”）。

## 继承与修改

本决策直接继承本链前序
[安装期包内清单校验决策](20260922-install-time-package-validation.md)，在其继承的
[分发根决策](0-registry-distribution.md)与[来源派生 Namespace 决策](20260831-source-derived-namespace.md)
上补齐展示层的来源呈现。Skill 来源的契约归属来自
[Skill 按存储路径以会话清单交给 Agent](../../core/agent/session/skill/0-skill-catalog-from-storage-paths.md)。

| 前序约定 | 本决策 |
| --- | --- |
| namespace 由 canonical source URL 派生，绑定持久化且不可变（Namespace D2、D3） | 继承，不变；标签从同一张绑定表反查 URL 后计算，不参与身份 |
| namespace 的准确语义是“首次引入该插件的来源”，必须写进用户可见文案（Namespace D3） | 继承，并落到第三方市场 tooltip 的文案原则（D4） |
| `official`、`local` 为保留 namespace（Namespace D4） | 继承；两者分别映射为“官方”和“本地包”两类来源（D1） |
| 不按 `owner/repo` 结构解析 URL，只取最后一段（Namespace D2） | 继承，并作为标签的默认取值；消歧时只按段比较，仍不解释段的含义（D3） |
| 列表必须显示条目来源，namespace 不直接展示；索引条目带归属 source 的 canonical URL（Namespace D6） | 继承；插件市场卡片按 D6 落实，本决策把同一套来源呈现扩展到 `/` 菜单与技能页，并新增 `origin` 字段（D2） |
| `SkillSource` 只区分 `Local` 与 `Plugin { plugin_id }`（Skill 存储路径决策） | **修改**：`Plugin` 变体增加 `origin`（D2） |
| `InstalledPlugin` 只带 `id`、`namespace`、`name` 等清单与安装态字段 | **修改**：增加 `origin`（D2） |
| `AvailablePlugin` 带 `namespace` 与 `source_url` | **修改**：增加 `origin`；`source_url` 保留（D2） |
| 安装期校验、发布资产获取、隐藏 listing 与零残留规则 | 继承，不变 |

## 背景

namespace 决策让不同来源的同名插件可以并存，同时把消歧责任交给了展示层，但界面没有跟上：

- 聊天框 `/` 菜单的 Skills 组只显示图标和名字。本地 Skill 与插件 Skill、或两个插件提供的同名 Skill 看起来完全一样，
  而 Skill 存储路径决策让它们都会同时出现在清单里。
- 技能页只给插件 Skill 一个“拼图 + plugin ID”按钮。plugin ID 含 `<slug>.<8 位摘要>`，正是 Namespace D6 说不应直接
  展示的形态；超过一定长度还会折叠成只剩图标。本地 Skill 没有任何来源标记。
- 插件市场卡片已经收到 `AvailablePlugin.source_url`，但只用于搜索，没有显示。

来源判定需要的数据都在后端：`official` 与 `local` 是保留值，其余 namespace 可以经只增不删的
`plugin_source_namespace(canonical_url ↔ namespace)` 表反查 canonical URL。但 `Skill` 与 `InstalledPlugin`
契约不带 URL，前端无从判断。

## 决策

### D1：来源分为四类，是插件级属性

| 来源 | 判定 | 标签 | 含义 |
| --- | --- | --- | --- |
| 用户 | `SkillSource::Local` | 用户 | 用户新建或导入（文件夹、`.zip`、`.skill`）的 Skill，可编辑和删除 |
| 本地包 | namespace = `local` | 本地包 | 本地导入的 `.orax` 插件包，只读，卸载插件后移除 |
| 官方 | namespace = `official` | 官方 | Ora 官方插件市场（默认 marketplace），只读，随插件更新 |
| 第三方市场 | 其余 namespace | 计算出的市场标签（D3） | 首次从该市场引入的插件，只读，随插件更新 |

规则：

1. 来源只属于插件。插件提供的 Skill 继承所属插件的来源，同一插件的所有 Skill 来源相同。“用户”不是
   `PluginOrigin` 的变体：本地 Skill 不属于任何插件，仍由 `SkillSource::Local` 表达。
2. 判定只看 namespace，不看当前的市场源配置。插件的来源是“首次引入它的来源”（Namespace D3）：市场源被删除、
   禁用，或插件在当前市场里已经找不到时，来源仍然不变。
3. 官方与第三方市场的区分沿用 Namespace D4 的保留值，不按 URL 判断。默认 marketplace 短路到 `official`，
   它的 URL 不在绑定表中，也不需要在；但它的 canonical URL 进入第三方标签的比较集合（D3），只参与消歧
   比较，自身的来源与标签仍是“官方”。

为什么来源是插件级而不是 Skill 级：Skill 没有独立于插件的来源，它是插件包的一部分，随插件安装、升级和卸载。
把来源挂在 Skill 上会让同一插件的两个 Skill 有机会显示不同来源，这种状态没有意义。

### D2：后端以 `PluginOrigin` 随三个契约下发来源

`PluginOrigin` 是带判别的枚举，语义示意如下（字段名与序列化以实现为准，不构成实现契约）：

```rust
enum PluginOrigin {
    LocalArchive,
    Official,
    Marketplace { source_url: Option<String>, label: String },
}

enum SkillSource {
    Local,
    Plugin { plugin_id: String, origin: PluginOrigin },
}
```

| 契约 | 变化 |
| --- | --- |
| `SkillSource::Plugin` | 增加 `origin`，`Skill` 与 `SkillDetails` 同时受影响 |
| `InstalledPlugin` | 增加 `origin` |
| `AvailablePlugin` | 增加 `origin`；`source_url` 保留，继续用于搜索并满足 Namespace D6 的归属下发要求 |

规则：

1. 所有返回 `Skill`、`SkillDetails`、`InstalledPlugin` 或 `AvailablePlugin` 的响应都由后端填好 `origin`，
   包括 Skill 列表与详情、已安装插件列表及扫描、激活和停止响应，以及市场列表和索引同步响应。各响应复用同一套
   来源判定与标签规则；前端不再从 `namespace` 或 `plugin_id` 推断来源，也不再显示原始 namespace。
2. `AvailablePlugin` 只会出现 `Official` 与 `Marketplace`：市场列表里的条目都来自某个市场源，`LocalArchive`
   不在其中。
3. `Marketplace.source_url` 是绑定表里的 canonical URL，用于插件详情信息表的“来源市场”一行（D4）。它只在
   绑定缺失这一损坏状态下为空（见规则 4），正常数据都有值。
4. 按 namespace 反查 URL：后端读取 `plugin_source_namespace` 的 namespace → canonical URL 映射（namespace 列唯一）。
   绑定表只增不删，且插件安装前绑定必定已经写入（Namespace D3），所以已安装插件的派生 namespace 都能查到 URL。
   查不到时属于损坏状态：后端记录告警，不让整个列表查询失败，该条目的来源降级为 `Marketplace`，`source_url`
   为空，标签取 namespace 的 slug 段（`.` 之前的部分）。读取绑定表本身失败时同样处理：记录告警，按空绑定
   集合计算（比较集合只剩默认 marketplace 的 canonical URL），所有派生 namespace 都按上述规则降级，响应
   不因此失败。

为什么由后端下发而不是让前端反查：绑定表在后端，标签计算需要完整的比较集合（D3）。让前端拿到绑定表再自己算，
等于把一张身份表和一套消歧规则都复制到前端，三个界面还要各自保证算出同一个结果。

为什么 `AvailablePlugin` 保留 `source_url`：市场搜索按 URL 匹配，Namespace D6 要求索引条目把归属 URL 随前端契约
下发，而官方条目的 `origin` 不带 URL。去掉顶层字段会让搜索改为匹配一个嵌套、且对官方条目不存在的字段，换来的
只是少一个重复字段。

### D3：第三方市场标签在查询时由后端计算

**分段**：把 canonical URL 去掉 `https://` 后按 `/` 切成段，第一段是 host（带非默认端口），其余是路径段。
canonical URL 已经去掉 userinfo、默认端口、尾部 `/` 与 `.git` 并转为小写（Namespace D2），所以分段结果是确定的。

**比较集合**：`plugin_source_namespace` 中的全部绑定（包括市场源已被删除、但绑定仍在的来源），加上默认
marketplace 的 canonical URL。默认 marketplace 不在绑定表中，它的 canonical URL 由后端在计算时补入，只参与
消歧比较：它自身的来源仍是 `official`，显示为“官方”，不产生第三方标签。

**计算规则**：

1. 默认标签是最后一段（仓库名）。比较集合里没有其他来源的最后一段与它相同时，标签就是这一段。
2. 有重名时，在最后一段相同的那组来源内，从倒数第二段往前逐段比较，找到第一个使本来源与组内其他所有来源都不同的段，
   标签写成“该段 + 仓库名”；两段不相邻时，中间用 `…` 表示省略。
3. 段数不同的 URL 按从末尾对齐的位置比较。其他来源在该位置没有段时，视为与本来源在该位置的段不同；本来源
   自己在该位置没有段时，不能用这个位置作标签，跳过并继续往前。
4. 不存在这样的单个段时，该来源的标签回落为去掉 `https://` 的完整 canonical URL。
5. 整组标签生成后，再按实际显示文本判重：规则 2 只保证所选段在同一位置上唯一，不同位置选中的同一个段会渲染成
   相同文本。仍有重名时，该组全部使用去掉 `https://` 的完整 canonical URL。

| 比较集合（canonical URL，省略 `https://`） | 标签 |
| --- | --- |
| `github.com/acme/awesome-plugins` | `awesome-plugins` |
| `github.com/acme/marketplace`、`github.com/obsismc/marketplace` | `acme/marketplace`、`obsismc/marketplace` |
| `github.com/ora-space/marketplace`（默认源）、`github.com/obsismc/marketplace` | `obsismc/marketplace`（默认源自身显示“官方”，不产生第三方标签） |
| `github.com/acme/tools`、`gitlab.com/acme/tools` | `github.com/…/tools`、`gitlab.com/…/tools` |
| `git.example.com:8443/scm/infra/registry`、`git.example.com/scm/infra/registry` | `git.example.com:8443/…/registry`、`git.example.com/…/registry` |
| `github.com/a/x/m`、`github.com/a/y/m`、`github.com/b/x/m` | `github.com/a/x/m`（回落）、`y/m`、`b/…/m` |
| `github.com/p/x/a/m`、`github.com/x/b/a/m`、`github.com/q/b/c/m` | 前两个按规则 2 都会显示 `x/…/m`，按规则 5 整组回落为完整 URL |
| `dev.azure.com/contoso/a/_git/registry`、`dev.azure.com/contoso/a/scm/registry` | `_git/registry`、`scm/registry`（平台字面量照样显示） |

默认源的 canonical URL 恒在比较集合中；除显式列出它的那一行外，表中各行只列绑定表中的绑定。默认源的最后
一段 `marketplace` 只与两个 `marketplace` 行重名，组内倒数第二段仍两两不同，各行标签不变；其余行的仓库名
与它不重名，结果也不受影响。

规则 2 只比较段的字面值，不判断哪一段是组织、项目或平台字面量。即使某一段是 Azure 的 `_git` 或 Bitbucket Server 的
`scm`，只要它是第一个不同的段，就照样显示。这延续 Namespace D2 的立场：不理解 URL 结构，只用跨平台都成立的
“段”。

**时机与身份边界**：

- 标签在每次查询时计算，不写入任何表或缓存。每个响应读取一次全部绑定，补入默认 marketplace 的 canonical URL，
  构造一份快照，响应内所有条目用这份快照计算；各响应用同一个函数，所以同一份比较集合下，`/` 菜单、技能页与
  市场卡片对同一来源看到的标签一致。
- 标签不是身份。新增一个同名来源后，已有来源的标签会从 `marketplace` 变成 `acme/marketplace`；删除市场源不会
  让标签变回去，因为绑定还在。任何匹配、存储和日志都继续用 namespace 或 Canonical Plugin ID，不用标签。

为什么比较集合是全部绑定而不是当前启用的市场源：已安装插件的来源可能已经不在市场源列表里，但它仍会出现在
`/` 菜单和技能页上。只和当前市场源比较，删除一个源就可能让两个已安装插件的标签变得相同。绑定表正好是“出现过的
所有来源”，且只增不删。

为什么还要补入默认 marketplace 的 canonical URL：本决策的典型场景正是官方市场的 fork——默认源
`github.com/ora-space/marketplace` 与 `github.com/obsismc/marketplace` 并存时，若默认源不参与比较，唯一的
fork 的标签就只是 `marketplace`，和“官方”放在一起看不出它是谁的 fork；参与比较后它显示为
`obsismc/marketplace`。默认源自身仍短路到 `official`、显示为“官方”，不产生第三方标签，所以补入它不会引入
与“官方”混淆的新标签。

为什么不持久化：标签依赖整个比较集合，任何一次新增绑定都可能改变已有来源的标签。持久化就要在每次新增绑定时
重算并改写其他行，而这只是为了缓存一个展示字段；绑定表通常只有个位数到几十行，查询时计算的代价可以忽略。

### D4：标签与 tooltip 的文案原则

1. 四类来源用同一个标签组件呈现：标签是 D1 表中的文字，tooltip 由“来源类型 + 一句说明”组成，说明对应 D1 表的
   “含义”列：是否可编辑、卸载或更新时会怎样。
2. 第三方市场的说明必须体现“首次从该市场引入”，不能写成“来自该市场”。这是 Namespace D3 对用户可见文案的要求：
   市场源变化后，插件的来源不跟着变。
3. 本地包的说明带上插件的显示名，让用户知道卸载哪个插件会移除这个 Skill。
4. tooltip 只显示来源类型和一句说明，不附加市场标签、URL 或域名。带协议的完整 canonical URL 只在插件详情信息表
   的“来源市场”一行显示，带复制按钮；`source_url` 为空时这一行不显示。官方插件的这一行显示“Ora 官方市场”。
5. tooltip 限宽并允许换行。标签在窄空间里可以截断，但用户应能在 tooltip 之外查看完整标签（例如悬停标签本身或在
   插件详情中）。
6. 文案提供中英文两套，标签和说明都不直接显示 namespace 或 Canonical Plugin ID。标签里为消歧带出的 host 等段是
   标签的一部分，不属于“在 tooltip 中放域名”。

为什么 tooltip 不放 URL：tooltip 在 `/` 菜单这类高频、狭窄的位置出现，URL 长度不可控，放进去会把说明挤掉；而且
标签已经由 URL 的段构成，域名在多数情况下（都在 github.com）并不提供更多区分信息。需要核对完整地址的用户，在插件
详情里能看到并复制。

### D5：三处界面的接入

1. **聊天框 `/` 菜单**：Skills 组的每一行在名称右侧显示来源标签，本地 Skill 显示“用户”。
2. **技能页**：每张 Skill 卡片显示来源标签，替换原来显示 plugin ID、会折叠成图标的按钮；插件 Skill 的标签仍可以
   点击跳转到所属插件。
3. **插件市场卡片**：按 Namespace D6 显示来源标签，与前两处使用同一组件和文案。
4. **查询失效**：添加、编辑或删除市场源的操作结束后，无论成功还是失败，前端都让 Skill 列表、已安装插件列表、市场
   列表和市场源列表的查询失效。新增绑定可能改变其他来源的标签，而多步操作（如先添加、再写入获取配置）在后一步
   失败并回滚配置时，前一步写入的绑定仍会保留。三处界面因此不需要重启就能看到消歧后的标签。插件安装和卸载沿用
   已有的失效规则。

## 为什么不是这些替代方案

| 方案 | 否决理由 |
| --- | --- |
| **按 `owner/repo` 解析 URL 作为标签** | GitHub 形态最可读，但 Namespace D2 已说明它不是通用结构：Azure 的 `_git`、GitLab 的子组、Bitbucket Server 的 `/scm/` 都会让“owner”落在错误的段上。按段比较对所有平台成立，在 GitHub 上的常见结果也正好是 `owner/repo` |
| **前端计算标签** | 前端要拿到完整绑定表并复制消歧规则，三个界面各自计算，容易在不同查询返回的不同集合上算出不同标签；`Skill` 与 `InstalledPlugin` 本来也没有 URL |
| **持久化标签** | 标签依赖整个比较集合，新增一个绑定就可能要改写其他行；持久化只缓存了一个展示字段，却引入了“存的标签和当前集合不一致”这种新的错误状态 |
| **只和当前启用的市场源比较** | 已安装插件的来源可能已被删除，比较集合变小后，两个已安装插件可能显示相同标签，正是要消除的问题 |
| **默认 marketplace 不进比较集合** | 官方市场的 fork 是唯一第三方来源时只显示 `marketplace`，和“官方”放在一起看不出它是谁的 fork；默认源参与比较后 fork 显示 `obsismc/marketplace`，而默认源自身仍显示“官方”，不产生第三方标签 |
| **tooltip 放完整 URL 或域名** | 位置狭窄、长度不可控，会挤掉说明；域名在多数情况下没有区分度。完整 URL 放在插件详情，需要时可以复制 |
| **直接展示 namespace 或其 slug** | Namespace D6 已明确 namespace 含摘要后缀，对用户没有可读性；slug 还会因截断和折叠而重名 |
| **用户自定义市场显示名作为本期方案** | 最可读，也不会随新增来源变化，但需要新增配置字段、编辑界面和重名处理，而且显示名只存在于本机；本期先用计算标签解决区分问题，自定义显示名留作后续增量（见下文） |

## 不变量

1. 来源只由插件的 namespace 决定：`local` 是本地包，`official` 是官方，其余是第三方市场；本地 Skill 是用户来源。
2. 插件提供的 Skill 与所属插件的来源相同。
3. 第三方市场标签只在后端计算，比较集合是 `plugin_source_namespace` 的全部绑定加默认 marketplace 的
   canonical URL；同一份比较集合下，所有带 `origin` 的响应对同一来源给出相同标签，同一组内的标签按显示文本
   互不相同。
4. 标签不持久化、不参与任何身份匹配；身份仍是 namespace 与 Canonical Plugin ID。
5. 标签计算不按平台解析 URL 结构，只比较 canonical URL 的段。
6. 用户可见的标签和 tooltip 不显示 namespace 或 Canonical Plugin ID；tooltip 不显示标签、URL 或域名。带协议的完整
   canonical URL 只出现在插件详情；标签里为消歧带出的段（含 host）或回落时的完整地址属于标签本身。
7. 第三方市场的 tooltip 体现“首次从该市场引入”。
8. 绑定缺失或绑定表读取失败时响应不失败，受影响的条目降级显示，并记录告警。

## 风险

- **标签会变**：用户新增一个同名市场后，已有来源的标签会变长。这是有意的：标签是展示，不是身份。用户可能一时认不出
  变化后的标签，tooltip 的类型说明和插件详情的完整 URL 用来兜底。
- **回落为完整 URL 时标签很长**：只在比较集合里出现规则 4、5 的交叉差异时发生，实际很少见；标签组件需要支持截断，
  并让用户在 tooltip 之外看到完整标签。

## 本决策未解决的问题

- **用户自定义市场显示名**：给市场源加一个仅用于展示的名称，覆盖计算标签。它不改变 namespace，但需要新增配置
  字段与界面，并决定重名时如何处理；本期不做，需要时另写增量 ADR。
- **工作流编辑器中的来源展示**：工作流 Skill 选择器是否复用同一来源标签，由工作流 Skill 绑定的决策链负责，不在本文。

## 落地顺序

1. **契约与后端**：增加 `PluginOrigin`，`SkillSource::Plugin`、`InstalledPlugin`、`AvailablePlugin` 带上 `origin`；
   实现 namespace 反查与标签计算，所有返回这些类型的响应装配 `origin`。完成条件：标签计算对 D3 表中的例子给出表中
   结果，各响应对同一来源返回相同标签，绑定缺失或读取失败时降级而不失败。
2. **前端**：共用来源标签组件接入 `/` 菜单、技能页和插件市场卡片，插件详情增加来源市场行，市场源变更结束后（含失败）失效相关查询，
   补中英文文案。完成条件：四类来源在三处都显示正确标签和 tooltip；新增同名市场源后不重启即可看到消歧标签。

两步在同一个 desktop 版本发布，前后端契约同时变化。从这个版本开始，三处界面显示来源标签，技能页不再显示 plugin ID；
插件身份、安装目录、Skill 匹配与市场源配置的行为不变。

## 核心测试用例

见[插件来源展示用例](../../../../test-cases/desktop/plugin/registry/origin-presentation.md)：来源按
namespace 分为四类且是插件级属性、所有带 `origin` 的响应装配同一套判定，D3 例表全部行（含默认源 fork 行与
规则 5 整组回落行）的标签计算，绑定缺失或绑定表读取失败时降级而不失败，同一份比较集合下各响应标签一致，
以及新增同名来源后标签变化而身份不变。实现尚未合入 desktop 的 main，义务如实标为 `Missing`；
[身份与 namespace 用例](../../../../test-cases/desktop/plugin/registry/identity-and-namespace.md)的证据保持
各自范围，不推导本文新义务。
