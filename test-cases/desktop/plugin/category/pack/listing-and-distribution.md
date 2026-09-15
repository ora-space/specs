# 插件集合包的清单、展示与分发边界

决策依据为已批准的
[Plugin Pack 根决策](../../../../../decisions/desktop/plugin/category/pack/0-plugin-pack.md)。
本文记录跨领域能力；已批准的配套规则分别见
[清单核心用例](../../manifest/pack-manifest-and-visibility.md)与
[registry 核心用例](../../registry/local-packs-and-hidden-listings.md)。

## Pack Manifests Must Remain Non Runtime and Reject Marketplace Visibility

**风险**：pack 被当成普通可执行插件，或复制时删除非法字段让不合法源清单进入安装态。

**前置状态**：合法 pack release、含 `marketplace_visible`（含显式 `true`）/下载资产/运行时
section 的 pack、携带 pack 成员声明的普通插件，以及 pack 包含运行时文件的非法布局。

**触发**：从源解析、本地创建并发现 pack，随后进入插件生命周期。

**必须成立**：合法 pack 保留身份和有序成员声明，禁止字段直接拒绝；真实布局中的 `assets/`、
`main.js` 等运行时内容由包校验拒绝。合法 pack 成为可识别安装实体，但不启动 runtime、不进行
handshake、不注册 host methods。普通 release 的可见性缺省 `true`，显式值必须是布尔；所有
installed 清单禁止该字段。普通无资产 listing 仍不可安装，pack 无资产可本地创建。

**禁止结果**：下载 pack、剥离非法字段后接受、给 pack runtime 权限或放宽普通插件资产要求。

| 验证义务 | 直接证据与缺口 | 状态 |
| --- | --- | --- |
| pack 与普通插件的字段/kind/形态约束符合已批准范围 | 待真实清单入口验证合法与禁止组合，包含非空成员约束；字段级义务见清单核心用例 | `Missing` |
| 本地复制保留声明，非法运行时内容被拒绝，生命周期不启动 runtime | 待 manager 与宿主集成验证 | `Missing` |
| pack 无资产可创建，普通无资产仍不可安装 | 待可安装性契约及安装入口验证 | `Missing` |

**决策依据**：D1、D3；不变量 1、2、13。

## Hidden Members Must Remain Installable After Index Filtering

**风险**：隐藏成员从源解析中一起消失，或 pack 安装绕过来源、host 和包校验。

**前置状态**：同源包含可见 pack、隐藏普通成员和可见普通插件；另一源存在同名成员，可配置
不同 Source 获取方式。成员制品可有摘要错误或 target 自述不匹配。

**触发**：成功重建索引，再从正常 pack 安装入口安装成员，并查询已安装插件。

**必须成立**：新索引没有隐藏 listing，浏览/搜索不提供独立入口；源内按 ID 仍可解析并通过
pack 安装，安装后真实实体可见。成员使用 pack 所属源的获取方式与正常制品校验，错误进入
pack 失败与补偿流程，不从其他源兜底。

**禁止结果**：resolve 依赖展示索引、隐藏成员安装后仍不列出、跨源下载或以 pack 名义跳过
摘要/target 校验。

| 验证义务 | 直接证据与缺口 | 状态 |
| --- | --- | --- |
| 新索引无隐藏条目，隐藏成员仍可被 pack 安装并进入已安装列表 | 待真实源、索引文件、安装入口与列表的完整链路验证 | `Missing` |
| 隐藏成员保留 Source 认证、host 与包校验，不跨源兜底 | 待双源及错误制品驱动生产安装接口验证 | `Missing` |

**决策依据**：D2–D4；不变量 4、12、13。

## Readable Old Caches May Retain Hidden Listings Until Successful Refresh

**风险**：可见性升级造成不必要的空市场，或为保留旧条目绕过既有缓存版本与格式检查。

**前置状态**：仍可读取的旧缓存包含源内现已隐藏的成员；另有独立格式变化造成版本不匹配或
无法反序列化的缓存。可使刷新失败或成功，也可使用旧宿主清单规则解析新源。

**触发**：打开市场、尝试刷新，再次打开；另行验证旧宿主规则的兼容行为。

**必须成立**：可读旧缓存在刷新前可以显示隐藏条目，刷新失败保留，成功后原子替换为无隐藏
条目的新索引。打开市场不隐式联网；不可读缓存仍按图标 ADR 降级。旧宿主拒绝未知 pack kind，
忽略普通成员的未知可见性字段，不能把这两种行为解释为迁移数据的依据。

**禁止结果**：可见性变化本身强制作废可读缓存、失败刷新删除缓存、保留半写入索引或绕过
缓存格式校验。

| 验证义务 | 直接证据与缺口 | 状态 |
| --- | --- | --- |
| 可读旧缓存经历失败/成功刷新后分别保留/原子替换 | 待缓存 fixture 与生产同步/列表接口验证 | `Missing` |
| 可见性容忍与不可读缓存降级互不冲突，读取不触发网络 | 待版本及格式不匹配 fixture 的组合验证 | `Missing` |
| 旧宿主对 pack 与隐藏普通成员的行为符合既有解析规则 | 待冻结旧解析器或旧宿主兼容 fixture 的直接验证 | `Missing` |

**决策依据**：D3、D9；不变量 12；
[图标资产决策](../../../../../decisions/desktop/plugin/manifest/20260909-plugin-logo-as-themed-asset.md)
的索引版本与不可读缓存降级边界。
