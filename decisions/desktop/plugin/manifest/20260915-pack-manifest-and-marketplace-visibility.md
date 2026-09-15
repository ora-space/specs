---
status: approved
---

# 插件清单：pack 声明成员而不声明制品，可见性字段仅属于普通插件发布清单

在双形态 `orax.toml` 中增加 `kind = "pack"` 及其独占的成员声明，pack 不声明下载资产或运行时
贡献物。同时将 `marketplace_visible` 定义为普通插件（`kind != "pack"`）release 形态的可选
布尔字段，缺省 `true`；pack 的两种形态和所有 installed 清单都禁止该字段。

本 ADR 为 `approved`，细化已批准的
[Plugin Pack 根决策](../category/pack/0-plugin-pack.md) D1–D3 的清单边界，已完成评审，尚未实现。
影响 `ora-plugin-manifest` 的解析与域模型，以及 registry 和 manager 对该模型的消费。本文不
定义安装状态文件格式、重试与回滚协议，也不改变普通插件的发布资产获取方式。

清单术语见 [Glossary](../../../../domain/glossary.md#plugin)。`resolver` 保持为 `1`，未知 kind
仍被旧宿主拒绝；新增可选字段在旧宿主被忽略。本文不迁移已有安装目录，不将旧清单批量重写为
新形态；pack 的新目录与状态恢复由根决策 D6、D9 负责。

## 继承与修改

本决策直接继承本链前序
[插件图标资产决策](20260909-plugin-logo-as-themed-asset.md)，并在其保留的
[清单根决策](0-versioned-dual-form-manifest.md)上扩展 kind 与字段规则；release locator 继续
遵守 [Source 级插件包获取决策](../registry/20260904-source-configured-artifact-retrieval.md)。

| 前序约定 | 本决策 |
| --- | --- |
| 图标由本地候选文件名决定，不进入清单 | 继承，不变；pack 不新增图标字段，图标协议也不参与清单复制 |
| 双形态共用 schema 与验证域模型，解析无 I/O | 继承，不变；新字段也由同一所有者校验 |
| kind 是封闭集合，各 kind 与专属 section 配对 | 修改：加入 `pack`，`[pack]` 与其成员数组对 pack 必需、对其他 kind 禁止 |
| 无下载资产的 release 清单合法，由分发层判定不可安装 | 修改：pack 必须无下载资产，分发层将其识别为本地可创建；普通插件无资产语义不变 |
| installed 可携带顶层 `url`/`sha256`，但不产生 release 来源 | 修改：该历史容忍仅保留给普通插件；pack 在两种形态均禁止资产声明 |
| 未知字段忽略，已知必填字段缺失或类型错误拒绝 | 继承，不变；`pack` 与 `marketplace_visible` 成为已知字段后须执行所属 kind/形态约束 |
| release locator 支持 HTTPS 或 object key | 继承，不变；pack 不拥有 locator，普通成员继续使用既有类型 |

## 决策

### D1：成员声明由 pack 独占，非法字段组合在解析期失败

pack 成员保持声明顺序，每项只有当前支持的成员引用能力：必填的裸 `identifier`，复用已有
`PluginName` 类型与校验规则。缺失成员声明或成员 identifier、错误类型和非法 identifier 均
产生指向相应字段的错误。第一版要求成员集合非空，避免把没有集合内容的条目当作有效 pack。

以下矩阵是两种清单形态的能力边界；普通插件指既有的非 pack kind，其原有专属规则继续生效。

| 字段或 section | pack release | pack installed | 普通 release | 普通 installed |
| --- | --- | --- | --- | --- |
| `[pack]` / `[[pack.members]]` | 必需，成员非空 | 必需，成员非空 | 禁止 | 禁止 |
| 成员 `identifier` | 裸 PluginName，必填 | 裸 PluginName，必填 | 不适用 | 不适用 |
| `marketplace_visible` | 禁止 | 禁止 | 可选布尔，缺省 `true` | 禁止 |
| 顶层 `url` / `sha256` | 禁止 | 禁止 | 沿用既有资产配对规则 | 沿用既有容忍，不产生 release 来源 |
| `[[targets]]` / `[artifact]` | 禁止 | 禁止 | 沿用 kind 与形态规则 | 沿用 kind 与形态规则 |
| `[workbench]` / `[webview]` | 禁止 | 禁止 | 沿用 kind 配对规则 | 沿用 kind 配对规则 |

pack 不是运行时 kind 缺省字段的组合，验证后的域模型必须明确表达其成员集合；普通插件不能
携带不可达的 pack 数据。新字段的解析仍遵守既有未知字段容忍策略，不因此全局启用严格未知字段
拒绝。后续成员版本约束、`agents` 或跨源引用不在本决策中增加。

重复成员、自引用、成员不存在、嵌套 pack、已安装跳过与 host 兼容性由 pack preflight 统一
判定。清单层只验证字段与形态，不读 registry、不补 namespace、不查询安装状态；完整 canonical
ID 由宿主使用 pack 的 namespace 组装，不能把 `namespace/identifier` 当作合法成员 identifier。

### D2：可见性是普通 release 的展示意图，禁止字段按存在性判定

普通 release 不写 `marketplace_visible` 等价于 `true`，`false` 表示不作为独立 listing 进入
新构建的展示索引。字符串 `"false"`、数字或其他错误类型不能按真值转换；错误归属于该字段。

pack 或任何 installed 清单出现该字段即报错，显式 `true` 也不例外。解析必须保留“字段是否
出现”的信息直到形态校验完成，不能先填默认值再判断，否则会混淆“未声明”与“非法声明”。

字段的过滤效果由 [registry 配套决策](../registry/20260915-local-pack-installation-and-hidden-listings.md)
负责。清单解析不把隐藏条目当作无效插件，不根据可见性改变下载来源或成员内容。

### D3：本地 pack 清单复制复用两种形态校验，不修复非法 release

宿主从已解析的 registry 条目复制 pack 的 `orax.toml`，以 installed 入口重新校验，并由包校验
确认 identifier、版本与安装目录一致。成员声明与普通元数据保持原值和顺序，namespace 仍来自
宿主目录，不写入清单。

合法 pack release 本就不含 `marketplace_visible` 与资产声明，无需复制时剥离这些字段；存在
禁止字段的源清单必须被拒绝，不能先删字段再宣称它合法。清单中的作者声明不承载安装进度、
操作 ID 或归属记录，后者由 pack 的持久状态机独立拥有。

`main.js`、`assets/` 和其他运行时内容的实际存在性由安装/包校验层负责。解析器保持纯函数，
不能通过读取文件系统来证明一个 pack 没有运行时贡献物。

## 为什么不是这些替代方案

| 方案 | 否决理由 |
| --- | --- |
| pack 继续声明一个只含清单的下载制品 | 产生第二份清单事实和独立网络失败点，违背已批准的本地创建行为 |
| 所有 kind 都允许 `marketplace_visible` | 允许隐藏 pack，超出已批准的普通插件字段边界 |
| 禁止字段为 `true` 时视为未声明 | 字段所属形态依赖取值，解析器无法可靠拒绝非法声明 |
| 普通插件也可以携带未使用的 `[pack]` | 形成混合 kind 的半有效清单，消费者需要重新解释其含义 |
| backend 自行复制并修剪字段 | 两套字段规则会漂移，且修剪会把应拒绝的输入转成有效安装 |
| 为 pack 新增 resolver 版本或另一种清单文件 | 当前封闭 kind 已让旧宿主明确拒绝 pack，新文件或版本扩大不必要的迁移面 |

## 不变量

1. pack 的两种形态均有非空的有序成员集合，普通插件不拥有 pack section。
2. 成员 identifier 使用既有 PluginName 规则，不承载 namespace 或版本策略。
3. pack 两种形态均无发布资产、运行时 section 或 `marketplace_visible`。
4. `marketplace_visible` 仅在普通 release 合法，缺省 `true`，类型必须为布尔值。
5. 本地复制保留合法 pack 声明，非法声明不会通过删字段被修复。
6. 新规则不改变清单解析的纯函数边界、普通插件资产类型与图标目录约定。

## 风险、兼容与未决项

- 旧宿主将未知 pack kind 整条拒绝，并忽略普通插件的可见性字段；发布方需要与支持 pack 的
  宿主版本配套。这是能力兼容差异，不通过改写已有用户清单掩盖。
- 新宿主会拒绝曾被当作未知字段忽略的非法 `pack` 或 `marketplace_visible` 用法；作者须修正
  源内容，宿主不自动删除字段。本决策不将未知目录或旧试验状态解释成已验证的 pack。
- 非空集合将“必需成员声明”收窄为至少一个成员，已随本决策批准；缺失或空成员集合均拒绝，
  对应验证义务已登记在本领域核心用例中。
- 所有插件更新策略与成员 `agents` 声明留在后续 ADR，本决策不提前增加可选字段。

## 落地顺序

1. **验证基线**：依据已批准矩阵、非空约束和错误归属建立失败与成功输入；完成条件是本领域
   核心用例的每项义务都有待实现的明确验证边界，不把记录用例等同于实现完成。
2. **清单所有者**：实现类型和双形态校验，验证非法组合、布尔存在性、成员顺序和普通清单回归。
   完成时清单可解析，但不单凭解析成功开放 pack 安装入口。
3. **消费者接入**：与 registry 决策和 pack 编排一起验证本地复制、包校验与无制品分支，完成
   生产接口验证后再开放能力。普通插件的下载、图标与 namespace 行为保持不变。

本领域核心用例见
[pack 清单与可见性校验](../../../../test-cases/desktop/plugin/manifest/pack-manifest-and-visibility.md)，
新增行为的直接证据当前为 `Missing`。跨领域能力验收见
[pack 核心用例](../../../../test-cases/desktop/plugin/category/pack/README.md)。
