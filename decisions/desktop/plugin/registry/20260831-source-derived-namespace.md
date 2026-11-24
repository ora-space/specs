---
status: implemented
---

# 来源派生 Namespace：用 Marketplace Source 身份替代清单自述的 namespace

> 历史决策：“namespace 一旦落盘就不可改变”一节中 `effect_sources`、`effect_consumers` 两行，以及 native identity 冲突和“先卸载让 Effect 收敛”的迁移说明，已由[Skill 按存储路径以会话清单交给 Agent](../../core/agent/session/skill/0-skill-catalog-from-storage-paths.md)（`approved`）随 Effect 退役而失去对象：Effect 表由前向迁移删除，同名 Skill 按存储路径区分、不再冲突。`skills` 表与插件目录按完整 Plugin ID 匹配的规则不变。

插件的 `namespace` 不再由条目清单声明，而由发布该条目的 Marketplace Source 决定：字段从清单
schema 中移除，`<namespace>/<identifier>` 中的 namespace 段由宿主从**规范化后的 source URL** 派生为一个
定长、文件系统安全的标识，并在 source 首次绑定时持久化。同时清单解析从「拒绝未知字段」改为
「忽略未知字段」，使清单 schema 具备向前兼容能力。

本决策处于 `implemented`：下文描述的是最终系统承诺，不是实现过程。承担该承诺的模块为
`ora-domain`（namespace 类型、长度与字母表边界、派生函数）、`ora-utils`（与领域无关的仓库 URL
规范化）、`ora-plugin-manifest`（schema 策略与字段移除）、`ora-plugin-registry`（索引按源归属与
来源标注）、`ora-plugin-manager`（安装目录写入与发现回读）、`crates/backend`（绑定持久化与 agent
标识）。实现与 approved 版本的偏离集中记录在文末「与 approved 决策的偏离」。

**本决策不承担向后兼容义务。** 已发布的清单、已安装的包与既有本地布局不在考虑范围内；下文的
兼容性讨论只针对本决策生效**之后**的身份稳定性。

## 继承与修改

本决策继承 [0-registry-distribution](0-registry-distribution.md) 的分发拓扑，修改与身份归属和
schema 策略相关的部分：

| 前一决策的约定 | 本决策 |
|---|---|
| 多源、同步方式、安装目录形状 `installed/<namespace>/<identifier>/<version>/` | 继承，不变 |
| 清单使用 `deny_unknown_fields`，未知字段报错并给出 TOML 路径 | **修改**：未知字段被忽略 |
| 清单声明 `namespace`，resolver 1 下封闭为 `official` | **修改**：字段移除，改由 source 派生 |
| 条目唯一标识为 `<namespace>/<identifier>` | 继承标识形状，**修改**其中 namespace 的来源 |
| 跨源同 id 去重，先出现的源胜出 | **修改**：不同源的同一插件天然是不同 id，不再互相遮蔽 |
| 源顺序决定重复 id 解析到哪个源 | **修改**：id 自带 namespace，安装与更新只由拥有该 namespace 的源解析；源顺序只影响列表与扫描次序 |
| 检出目录按 URL 原文逐段派生 | **修改**：改按 canonical URL 派生，等价写法共用一份检出 |
| 源配置以 URL 原文判重 | **修改**：按 canonical URL 判重，同一仓库的两种写法不能被配置成两个源 |
| 安装身份完全由包内清单决定 | **修改**：identifier/version/kind 归清单，namespace 归安装目录 |
| 发布资产、target 选择、索引缓存、README/Logo | 继承，不变 |

## 背景

在本决策之前，`PluginNamespace` 是封闭枚举，resolver 1 下只接受 `official`，所有条目、所有源、
所有安装产物的 namespace 都是同一个常量。namespace 恒为常量，等于身份空间里没有「来源」这一维：

- **枚举无法容纳用户自建源。** 新增一个 namespace 取值意味着改 Rust 封闭枚举并发版，而允许用户
  自行添加 marketplace source，恰恰要求身份空间在运行期可扩展。
- **任何源都可以发布任何 id。** 第三方源只要写下与官方相同的 `identifier`，就产生完全相同的
  `official/<identifier>`。索引合并按源顺序先到先得去重，排在前面的源可以静默遮蔽官方条目，用户在
  市场里看不出条目来自哪个仓库；安装与后续更新都会跟随遮蔽者。
- **命名冲突没有仲裁者。** 两个互不相识的第三方源使用同一 `identifier` 时，谁先配置谁获胜，
  结果随源顺序变化而变化，不是稳定身份。
- **字段本身没有信息量。** 它是所有条目都必须原样抄写的常量，既不提供隔离也不携带来源。

来源是唯一天然可信、且宿主可独立验证的隔离维度：条目所在的 Git 仓库是用户显式配置并信任的
对象。把 namespace 绑定到来源，冲突就退化为「不同来源的同名插件」，可以并存、可以分别安装、
可以如实呈现，而不是互相覆盖。

## 决策

### D1 清单 schema 改为忽略未知字段，并移除 `namespace`

两处修改，一次做完：

**未知字段被忽略而不是被拒绝。** 根决策选择 `deny_unknown_fields`，让打错的字段名立刻暴露。
但清单由第三方作者书写、由不同版本的 Ora 读取，严格拒绝意味着**任何一次 schema 增量都会让旧版本
Ora 拒绝新清单** —— 一个纯增量的可选字段足以让整个条目从旧客户端的市场里消失。schema 演进的
断裂性变化由 `resolver` 版本承担，增量变化应当由未知字段容忍承担，两者分工明确。

代价必须说清楚：**可选字段的拼写错误从报错退化为静默忽略**。`sha25` 写错一个字符不再报错，而是
让该条目表现为「没有声明摘要」。缓解来自两侧 —— 必填字段缺失仍然是硬错误，而可选字段的错字
表现为功能缺失（条目不可安装、图标不显示），在发布流程中可见。判断是：让旧客户端因一个新字段
而丢失整个条目，比让作者的错字晚一步暴露更糟。

**`namespace` 字段被移除。** 不再是 schema 的一部分，也不再有对应的字段路径与错误变体 ——
namespace 不从清单解析，「清单里 namespace 非法」这一错误类别永远不会发生。清单里残留该字段时，
按上一条被忽略，不影响身份，也不产生告警。

### D2 namespace 由规范化 source URL 派生

namespace 不是 URL 原文，而是从 canonical URL 派生的定长标识：

```
<repo-slug 截断至 24 字节>.<canonical URL 的 sha256 前 8 位十六进制>

https://GitHub.com/Acme/Awesome-Plugins.git/
  ⇒ canonical: https://github.com/acme/awesome-plugins
  ⇒ namespace: awesome-plugins.2aa64f48
```

上例中的摘要是真实值，不是示意值：任何版本的 Ora 对该 canonical URL 都必须得出
`awesome-plugins.2aa64f48`。

**规范化**（决定摘要输入，因此是兼容性契约的一部分）：去 userinfo、去默认端口、去尾部 `/`、
去尾部 `.git`，整串转小写。转小写对 host 是规范要求，对路径是一处**有意为之的有损选择**：
主流 Git 托管对 owner/repo 大小写不敏感，让同一仓库的两种拼写产生两个身份，比让两个仅大小写
不同的仓库合并成一个更糟。

**slug** 取 canonical URL 的最后一个路径段，仅保留 `[a-z0-9-]`，其余字符折叠为 `-`，合并连续
`-`，去首尾 `-`，截断至 24 字节；为空时使用固定占位值。

**摘要**取 canonical URL 的 sha256 前 4 字节，渲染为 8 位小写十六进制。

**为什么只取最后一段，其余信息全部交给摘要。** `https://github.com/ora-space/desktop` 这种
`host/owner/repo` 三段式只是 GitHub 的形态，不是 Git 托管的通用结构：路径深度因平台而异，中间还
可能夹着平台固定字面量。

| 来源 URL | 最后一段（slug） | 若按「owner/repo」结构解析 |
|---|---|---|
| `https://github.com/ora-space/desktop` | `desktop` | `ora-space` / `desktop`，符合预期 |
| `https://dev.azure.com/contoso/analytics/_git/data-pipeline` | `data-pipeline` | owner 落在 `_git` 上，是平台字面量而非组织名 |
| `https://gitlab.com/acme/platform/tools/orax-registry` | `orax-registry` | 子组嵌套深度不定，「owner」是哪一段没有答案 |
| `https://git.example.com:8443/scm/infra/very-long-repository-name-for-tooling` | `very-long-repository-nam`（截断） | `/scm/` 是 Bitbucket Server 的固定中缀，同样不是组织名 |

任何试图**理解** URL 结构的派生都会走向两条死路之一：为每个平台特化解析规则，或者把整条路径折叠
进标识、让长度随嵌套深度失控。而最后一段在上述全部平台上都恰好是仓库名 —— 它是唯一跨平台稳定
可取的人读片段。摘要则相反，它不理解结构也不需要理解：组织名、项目名、子组层级、host、端口全部
参与哈希，因此两个仅在被丢弃部分上不同的 URL（同名仓库分属不同组织）仍然得到不同 namespace。

设计取舍：

- **slug 负责可读，摘要负责唯一。** slug 会因折叠与截断而碰撞（子组路径、非 ASCII 仓库名、
  超长名），把唯一性交给它就等于把身份正确性押在折叠规则是否单射上。摘要把这个不可控问题
  从身份保证里拿掉，slug 退化为纯粹的人读前缀。
- **定长上界 33 字节**（24 + `.` + 8）。namespace 是三条文件系统路径的一级目录名，长度必须有
  硬上界，不能随 URL 形态（自建 host、深层子组路径）增长。
- `.` 是 `PluginId` 段的合法字符，因此 `slug.digest` 无需引入新的字母表。
- 派生是纯函数，且被固定为**兼容性契约**：同一 canonical URL 在任何版本、任何平台必须得到同一
  namespace。规范化规则与摘要长度一经发布即不可修改。

### D3 source 与 namespace 的绑定持久化且不可变

派生只在一个 source **首次绑定时**发生一次，结果随 source 配置持久化，之后只读：

  - 规范化规则若在将来需要扩展（新的等价 URL 形态），已绑定的身份不跟着变。派生函数是契约，
    持久化是这条契约在本机的兑现记录。
    例如，若未来把某种新的 URL 写法识别为与现有 source 等价，重新计算时也不能因此改写已有插件的
    namespace、安装目录或关联数据；新的写法只能解析到原有绑定。
  - 删除某个 source 后再以同一 canonical URL 添加回来，命中历史绑定复用原 namespace，而不是生成
    第二个身份。
  - 因此 namespace 的准确语义是「**首次引入该插件的来源**」，不是「该插件当前所在的 URL」：它表示
    插件首次由哪个 Marketplace Source 引入并完成绑定，而不是每次根据当前 URL 重新现算的结果。即使
    source 被删除后以等价 URL 重新添加，仍应复用原绑定；URL 的变化最多使插件暂时与市场条目脱钩，
    不能改变已安装插件的身份。这一点必须写进用户可见文案。

### D4 保留 namespace

| namespace | 绑定对象 | 理由 |
|---|---|---|
| `official` | 默认 marketplace，短路返回，不走派生 | 第一方来源的可读标识；绝大多数插件出自这里，`official/<identifier>` 比 `marketplace.a1b2c3d4/<identifier>` 在目录、日志与错误信息中都可读得多 |
| `local` | 本地 `.orax` 导入 | 本地包没有 source URL，无从派生，必须有确定归属 |

派生函数必须显式拒绝产出这两个值。

`official` 是可读性取舍而非必需：让默认源和其他源一样走派生也是自洽的，代价是第一方插件的
身份带上一段无意义的摘要。`local` 则是结构性必需 —— 没有 URL 就没有派生输入。

### D5 安装目录是 namespace 的权威记录

前一决策规定「目录名不是身份，清单才是身份」。namespace 移出清单后这条不再成立：包内清单由
插件作者书写，作者无法也不应知道用户从哪个源安装。

新规则：

> **identifier、version、kind 由包内清单决定；namespace 由安装目录决定。**

安装路径 `installed/<namespace>/<identifier>/<version>/` 本来就是宿主写的，一级目录名即是权威记录，
不需要额外的来源 sidecar。这条同时保证身份一旦落盘就不再由任何可变输入决定 —— 见下文。

### D6 索引合并语义变化

不同源发布同一 `identifier` 时不再合并成一条，而是两条不同 id 的条目同时出现。这是本决策要的
结果（来源可见、可分别安装、互不遮蔽），但它把消歧责任交给了展示层：

- 列表必须显示条目来源，否则用户会看到两张几乎一样的卡片：卡片上能看到的 title、描述与图标
  全部来自条目清单，而清单可以被照抄。
- 索引条目记录归属 source 的 canonical URL，并随前端契约一起下发；合并后不得丢弃归属。
- namespace 自身含摘要后缀，对用户没有可读性，只用于身份与搜索，不应直接展示。

### D7 namespace 的类型归属

`PluginNamespace` 与 `PluginId` 同住于 `ora-domain`，不再属于 `ora-plugin-manifest`：它不是清单
概念，而是宿主生成的身份片段。类型化同时闭合了一处缺口 —— namespace 段此前只校验字符集而没有
长度上限，却是文件系统路径的一级目录名。长度与字母表约束因此有唯一归属，D2 的 33 字节上界由
类型本身保证，`.` 与 `..` 也在同一处被拒绝。

### D8 agent 标识使用 canonical Plugin ID

`agent` kind 的运行期标识以及 `sessions.agent_cli` 字段使用完整的 canonical Plugin ID，而不是只使用
其中的 `identifier`（历史上的 name 段）。因此，两个 source 发布相同 `identifier` 的 agent 时，仍然
拥有不同的 agent 标识，可以同时安装、监督和绑定会话；不再因为 name 段相同而在安装期拒绝。

这项变更不承担向后兼容义务：既有只保存 name 段的 `agent_cli` 数据不在本决策范围内，读写一律按
canonical Plugin ID 进行，匹配不到的旧值表现为该 agent 未安装。相同 canonical Plugin ID 表示同一个
agent，按正常的更新语义处理；只有 canonical Plugin ID 不同的 agent 才是不同实例。

这样，运行期监督表、agent 选择器和会话绑定都以完整插件身份为键，namespace 不会在身份匹配时被
丢弃。D6 关于不同 source 的同名条目可以并存的规则也因此对 `agent` kind 成立，不需要额外的安装期
冲突仲裁或依赖遍历顺序决定胜负。

## 为什么不是这些替代方案

| 方案 | 否决理由 |
|---|---|
| **URL 原文作为 namespace** | `PluginId` 段只接受 `[a-z0-9-.]`，URL 含 `:`、`/`、大写；`/` 尤其致命，它是 canonical id 的分隔符，会让 `parse` 与 `canonical` 不再互逆。URL 上限 2048 字节，而 namespace 进入每条安装路径 |
| **反向 DNS 折叠（`com.github.acme.awesome`），不带摘要** | 可读性最好，但唯一性押在「折叠是否单射」上：含点的路径段、深层子组、非 ASCII 仓库名都会破坏单射，且长度随路径深度增长，没有上界 |
| **用户添加 source 时自行命名** | 天然不漂移且最可读，但 id 退化为**本机别名**：同一个源在不同机器上叫不同名字，日志、错误信息、文档与未来任何跨机器场景都需要一层翻译。同时给用户增加一个必填字段和一次重名处理 |
| **由源仓库根部的 `index.toml` 声明本源全部插件的 namespace** | 可读性最好，且**唯一独有的优势是源可迁移**——换域名、换托管平台、组织改名都不改变身份。但它给不了唯一性：两个互不相识的源都可以声明 `acme`，第三方源更可以声明 `official` 直接冒充第一方并遮蔽官方条目，正是本决策要消灭的问题原样回来。宿主只能在添加源时检测冲突并拒绝，那退化为**本机唯一**，且冲突时用户无从解决（他改不了别人仓库里的文件）。更根本的是信任模型倒置：`index.toml` 是仓库内容，与插件清单同属第三方可写数据，违反不变量 1；它还使 namespace 可变（源维护者随时能改，触发「namespace 一旦落盘就不可改变」一节列出的全部后果），并让添加源从纯本地操作变成必须先联网同步才能确定身份。该方案在源级元数据上的价值与 namespace 归属正交，不构成用它决定身份的理由 |
| **保留 `deny_unknown_fields`，只把 `namespace` 声明为忽略字段** | 保住了错字检测，但把 schema 演进锁死在「旧客户端拒绝新清单」上；每加一个可选字段都要等全部客户端升级 |

## namespace 一旦落盘就不可改变

D3 的不可变绑定与 D5 的目录权威是一组约束，共同保证**已安装插件的 namespace 不会因任何可变输入
而改变**。这不是向后兼容义务，而是本决策生效后必须长期成立的正确性前提 —— namespace 是完整
plugin id 的一半，而 plugin id 是五处持久化状态的匹配键：

| 位置 | plugin id 在其中的角色 |
|---|---|
| `installed/<ns>/<identifier>/<version>/` | 安装目录 |
| `plugins/data/<ns>/<identifier>/` | 插件私有数据目录，绑定给 `ora/storage/*`；也是 MCP 与全部插件配置 `store.json` 的所在 |
| `skills` 表 | `namespace` 列存**完整 canonical Plugin ID**，行 id 为 `plugin:<plugin_id>:<skill-name>` |
| `effect_sources` 表 | `UNIQUE(effect_kind, source_kind, namespace, identifier)`，namespace 同为完整 plugin id |
| `effect_consumers` 表 | identity_key 为完整 plugin id，storage key 为其摘要 |

若某个已安装插件的 namespace 事后被改写，后果不是「多出一条无害的旧数据」：

- 插件 Skill 的写入与清理**完全按 skills 表的 `namespace = <canonical Plugin ID>` 匹配**。namespace 一变，
  新 id 的写入和卸载都碰不到旧行，旧 skills 行与 `effect_sources` 永远保持 active，且卸载插件也清不掉。
- skills 的活跃唯一索引是 `(namespace, name)`，两行 namespace 不同即合法共存，不会报错，只会
  静默出现两个同名技能。
- Skill 的 native identity 是**技能名本身，不含 namespace**。两个 Desired Effect 落到同一
  Workspace 的同一 Resource 会触发 native identity 冲突，该 Workspace 的 Skill 收敛拿不到干净的
  Current。
- 插件私有数据目录随之改变，等于让该插件看到一个空 storage：Agent 插件持有的会话映射与授权态
  丢失，恢复既有会话时按持久化的 agent session id 加载会失败；MCP 与插件配置表现为用户填写的
  内容凭空消失。

`sessions`、`workflows`、`agents`、会话历史文件**不在这个清单里**，但理由各不相同。`workflows`、
`agents` 与历史文件确实不受影响：前两张表的 namespace 恒为 `local` 且插件不向它们贡献任何行，历史
路径只由 Ora 自己的 session UUID 分片。`sessions.agent_cli` 在 D8 之后同样存完整 canonical Plugin
ID，因此 namespace 一变它也会失配 —— 但它的后果只是该会话报告 agent 未安装，这是一个既有的、可
恢复的运行期状态：前端应提示切换 agent，用户选择可用 agent 后按普通会话切换流程继续对话，不以重新
安装原插件为前置条件。会话历史由 Ora 持有，加载时必须能在原 agent 不可用的情况下独立回放，不能在读取
历史前因恢复原 provider 失败而终止。它不是上表那种无人认领的 ownership 残留；区别在于能否被清理，不在于是否失配。

在 D3 + D5 之下，即便 source 配置发生变化，已安装插件的最坏后果也只是**与来源脱钩**（市场显示
未安装、无法更新、可被重复安装一份），不会损坏任何持久化状态。

若将来确有必要迁移某个插件的 namespace，正确做法**不是就地 UPDATE**：必须先以旧 id 走完整卸载
（让 Effect 以空 projection 收敛掉 Workspace 中的旧物化产物），再以新 id 走一次安装。直接改写
持久化行会绕过 Effect 的 ownership 账本，留下无主的 Managed Item；且已发布的 Effect revision 是
不可变的，其中冻结的安装路径会与改写后的 source 行互相矛盾。

## 不变量

1. namespace 由宿主生成，永远不来自任何第三方可编辑的清单内容。
2. 同一 canonical source URL 在任何版本、任何平台派生同一 namespace；规范化规则与摘要长度是
   兼容性契约，发布后不可修改。
3. source 与 namespace 的绑定一旦写入即不可变。
4. namespace 是单个合法路径段：`[a-z0-9-.]`、非空、不超过 33 字节、不是 `.` 或 `..`。
5. `official` 与 `local` 是保留字，派生结果永远不得等于它们。
6. 安装包的 identifier/version/kind 来自包内清单，namespace 来自安装目录；两者不一致时判为损坏并上报，
   不猜测。
7. 已安装插件的 namespace 不因任何配置变化而改变。
8. 清单中的未知字段被忽略；必填字段缺失仍然是硬错误，断裂性 schema 变化仍由 `resolver` 版本承担。
9. 跨源同名条目并存而不互相遮蔽；消歧发生在展示层，不发生在索引合并层。
10. `agent` 的运行期标识与 `sessions.agent_cli` 使用完整 canonical Plugin ID；不同 source 的同名
    agent 可以并存，只有相同 canonical Plugin ID 才表示同一个 agent。

## 与 approved 决策的偏离

实现落地时相对 approved 版本有以下偏离，均为已采用的最终决策：

**派生函数归 `ora-domain`，而非 approved 版指定的 `crates/backend`。** 保留字判定与 namespace 的
长度、字母表边界属于身份类型本身，把纯函数放在类型旁边，兼容性契约才能脱离后端装配被单独判定。
`crates/backend` 仍然独占绑定与持久化，两者的职责边界没有变化。URL 规范化不含任何 Ora 领域词汇，
按仓库约定落在 `ora-utils`。

**D2 的示例摘要值更正。** approved 版把示例写作 `awesome-plugins.3f9a1c2d`，该值不是对应 canonical
URL 的真实 sha256 前缀。真实值为 `awesome-plugins.2aa64f48`，D2 已更正。派生规则本身未变，只是示例
从示意值改为契约值。

**检出目录与源判重同时改用 canonical URL。** approved 版只规定 canonical URL 决定摘要输入，未涉及
这两处。若不一并改：同一仓库的两种写法会克隆两份检出，并被配置成两条源记录，而这两条记录会绑定
到同一个 namespace —— 一个源被列成两个，却共享身份与检出。这是不变量 2 的直接后果，不是独立选择。

**两处身份仲裁随规则消失而删除。** 二者都是 D5 与 D8 的推论，但会改变宿主对外可观察的行为，故在此
记录：

- discovery 不再报告 `duplicate_plugin_id`。id 的两段都成为目录层级后，遍历对每个
  `<namespace>/<identifier>` 只访问一次，两个包无法占用同一个 id。散落在别处目录的副本改由「清单
  identifier/version 与所在目录不一致」报告。
- agent 监督不再做身份仲裁。标识改为完整 canonical Plugin ID 后，已安装集合内不可能重名，监督
  集合恒等于已安装的 agent 集合，不存在「谁先被监督谁获胜」。

## 本决策未解决的问题

- **namespace 迁移没有工具。**「namespace 一旦落盘就不可改变」给出的安全路径是旧 id 完整卸载 + 新
  id 重新安装，需要用户手动完成，配置也随卸载一并丢失。是否提供一条保留配置的迁移路径，留给后续
  决策；本决策只保证不发生需要迁移的漂移。
- **source 迁移（换域名、换托管平台、组织改名）不被支持。** 这是「为什么不是这些替代方案」中
  `index.toml` 方案唯一独有的优势，本决策明确用它换取唯一性与信任模型。等价 URL 归一只覆盖同一
  仓库地址的不同写法，不覆盖仓库真正搬家。
