# Pack 清单与普通插件可见性校验

决策依据为已批准的
[pack 清单与可见性字段 ADR](../../../../decisions/desktop/plugin/manifest/20260915-pack-manifest-and-marketplace-visibility.md)。
本文聚焦清单所有者的验证边界；整体安装与恢复见
[pack 核心用例](../category/pack/README.md)。新行为尚无直接可执行证据，均标为 `Missing`。

## Pack Members Must Form a Nonempty Ordered Set of Bare Identifiers

**风险**：空集合或混合 kind 清单进入域模型，完整 ID 被误当成裸 identifier，或解析时发生源查询。

**前置状态**：release 与 installed pack 清单，分别包含合法有序成员、缺失/空 members、缺失
identifier、错误类型或完整 `namespace/identifier`；普通 kind 可携带 `[pack]`。

**触发**：通过清单所有者的两种生产解析入口读取文本。

**必须成立**：合法 pack 产生非空有序成员集合，identifier 复用 PluginName 规则且原值保留；
缺失、空集合、类型错误、非法 identifier 与普通 kind 的 pack section 均被拒绝，错误归属明确。
未知字段仍按既有规则忽略；纯解析不补 namespace、不读源或安装状态。重复、自引用与嵌套等
跨值或跨清单约束仍由 pack preflight 负责，不在本用例中把字段解析误当作完整计划验证。

**禁止结果**：把空集合当有效 pack、静默规范化 identifier、解析时查安装状态或丢失成员顺序。

| 验证义务 | 直接证据与缺口 | 状态 |
| --- | --- | --- |
| 双形态均拒绝缺失/空成员集合，合法对象保留完整有序声明 | 待两种解析入口的完整对象与字段错误断言 | `Missing` |
| identifier 类型与 kind 独占约束正确，未知字段策略不变 | 待覆盖裸 ID、完整 ID、错误类型和普通 kind 携带 pack 的输入 | `Missing` |
| 清单解析保持纯函数，源/安装状态判定留在 preflight | 待解析与 preflight 分别验证职责边界 | `Missing` |

**决策依据**：清单配套 ADR D1；不变量 1、2、6。

## Marketplace Visibility Must Be Validated by Kind Form and Presence

**风险**：默认值抹掉字段存在性，导致 pack 或 installed 显式写 `true` 被错误接受；错误类型被
当作布尔值转换。

**前置状态**：普通 release、pack release、普通 installed 与 pack installed 清单，字段分别
缺省、为 `true`、为 `false` 或为字符串/数字等非法类型。

**触发**：通过对应清单解析入口校验并读取域模型。

**必须成立**：普通 release 缺省等价于 `true`，合法 `false` 保留展示意图；其他形态/kind 组合
只要出现字段就报错，显式 `true` 也拒绝。非法类型归属于 `marketplace_visible`；隐藏状态不
删除有效 release 来源，不把普通插件解析成无资产或禁用状态。

**禁止结果**：先填默认值再无法区分缺省与显式声明、字符串真值转换、pack 隐藏开关或
installed 清单保留可见性字段。

| 验证义务 | 直接证据与缺口 | 状态 |
| --- | --- | --- |
| kind × 形态 × 字段存在性矩阵全部遵守约束 | 待生产解析入口的表驱动字段与完整对象断言 | `Missing` |
| 普通隐藏 release 仍保留完整来源与插件语义 | 待对完整域对象进行对比，验证仅展示意图变化 | `Missing` |

**决策依据**：清单配套 ADR D2；不变量 3、4。

## Local Pack Copy Must Preserve Valid Declarations Without Repairing Invalid Ones

**风险**：宿主复制时修剪非法字段，掩盖源错误；两种形态分离后成员或元数据漂移。

**前置状态**：合法 pack release；带资产、运行时 section 或可见性字段的非法 pack；包含
普通插件既有资产形态的对照清单。

**触发**：源清单解析后执行本地复制与 installed 校验；另行验证包目录身份和运行时文件。

**必须成立**：合法复制保留成员原值、顺序、身份及元数据，installed 入口仍接受；非法 release
在复制前被拒绝。namespace 不写入清单，目录身份由包校验确认；运行时文件的存在性校验留在
manager。普通 release locator、普通 installed 的既有资产容忍和图标目录约定保持不变。

**禁止结果**：删掉禁止字段后接受、丢失成员声明、把宿主状态写入清单或在纯解析中检查文件。

| 验证义务 | 直接证据与缺口 | 状态 |
| --- | --- | --- |
| 合法复制双形态一致，非法 pack 资产/运行时 section/可见性声明被拒绝 | 待生产复制与解析入口的组合验证 | `Missing` |
| 目录身份和运行时文件由包校验拒绝，清单不承载宿主状态 | 待实际包布局与 manager 校验验证 | `Missing` |
| pack 例外不改变普通插件资产或图标规则 | 待新解析模型下的普通插件与图标回归验证 | `Missing` |

**决策依据**：清单配套 ADR D1、D3；不变量 3、5、6。
