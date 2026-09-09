# 插件图标资产

架构边界与不变量来自
[插件图标决策](../../../../decisions/desktop/plugin/manifest/20260909-plugin-logo-as-themed-asset.md)。
清单字段本身的解析与校验用例在[插件清单 schema 与校验](schema-and-validation.md)，本文不重复：
该决策不新增任何清单字段，图标信息完全不经过清单。

该决策处于 `approved` 但尚未实现，因此下列证据状态以 `Missing` 为主。标记为 `Partial` 的行只出现
在会被直接复用的既有下层策略上（`ora_utils::svg` 的安全校验与字节上限），它们证明的是被复用的
那一层，不证明本决策新增的语义。

## Logo Resolution Must Be Total and Yield the Same Variant in Market and After Install

### 风险

判定表若不是全函数，某些磁盘组合会落到未定义分支，作者少放一个文件就可能收到一条无从修复的
discovery issue。而候选名表与判定表若在 registry 索引构建与包发现各有一份实现，同一个插件会在
市场里和安装后显示不同的图标——一种用户能看见、但没人会主动去测的差异。

### 前置状态

- 包根或 registry 条目目录下存在十五个候选文件名的任意子集。
- 同一主题角色下可能同时存在多个扩展名。
- 主题对的两半可能是不同扩展名。

### 触发

registry 索引构建扫描条目目录；包发现扫描已安装包根。

### 必须成立

- 三个角色的八种存在组合各自产生确定结果，没有任何一种是错误或未定义。
- 同一角色下多个扩展名共存时按 `.svg` > `.png` > `.webp` > `.jpg` > `.jpeg` 取一个，结果与目录
  枚举顺序无关。
- 只有 `light` 或只有 `dark`、且 `universal` 存在时，用 `universal` 补齐缺失的另一半。
- 三个角色都存在时主题对胜出，`universal` 被忽略。
- 主题对的两半允许使用不同扩展名。
- 解析结果是角色到**扩展名**的映射，能区分 `logo.jpg` 与 `logo.jpeg`。
- 同一份磁盘状态经索引构建与包发现解析出完全相同的映射。

### 禁止结果

- 任何一种存在组合产生 discovery issue，或使该条目被跳过、被移出列表。
- 解析结果依赖目录枚举顺序。
- 强制主题对两半使用同一种格式。
- 解析结果只记录格式而丢失 `.jpg` / `.jpeg` 的区别。
- 解析过程向包目录或 registry 条目目录写入任何文件。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| 八种角色存在组合各自产生确定变体 | 尚未实现 | `Missing` |
| 扩展名优先级确定且与枚举顺序无关 | 尚未实现 | `Missing` |
| 单侧主题角色由 `universal` 补齐，双侧缺失时退回 `Universal` | 尚未实现 | `Missing` |
| 主题对两半允许不同扩展名 | 尚未实现 | `Missing` |
| 解析结果记录扩展名而非格式 | 尚未实现 | `Missing` |
| 索引构建与包发现对同一磁盘状态给出相同映射 | 尚未实现 | `Missing` |
| 图标解析路径不产生任何写入 | 尚未实现 | `Missing` |

### 决策依据

- 图标决策 D2：主题变体与格式由文件名约定确定，判定表覆盖全部磁盘状态。
- 图标决策不变量 3、4、11。

## Any Logo Failure Must Be Indistinguishable From a Missing Candidate File

### 风险

图标是展示层的东西，但它的读取发生在索引构建与包发现这两条决定"插件是否可见、是否可安装"的
路径上。任何一个图标失败若向上冒泡，一张损坏的图片就能让插件从市场消失或无法安装。若失败被
建模为区别于"不存在"的第三种状态，判定表还会从三个布尔维度膨胀成三个三值维度。

### 前置状态

某个候选文件处于下列任一状态：不可读、格式不在白名单、magic bytes 与扩展名不符、超出字节或
像素上限、SVG 未通过安全校验、WebP 带 ANIM 标志。同一角色下可能还存在其他合法候选。

### 触发

registry 索引构建与包发现读取候选文件。

### 必须成立

- 失败的候选被当作不存在，解析沿扩展名优先级与判定表继续。
- 损坏的 `logo.dark.webp` 与合法的 `logo.dark.png` 共存时，深色角色解析为 PNG。
- 损坏的 `logo.light.svg` 与合法的 `logo.svg` 共存时，结果是 `Universal(logo.svg)` 而非 `None`。
- 每次失败记 warning。
- 插件仍然被发现、被索引、可安装、可加载、可运行。

### 禁止结果

- 图标失败使条目从 `listAvailablePlugins` 的结果中消失。
- 图标失败使索引构建或包发现返回错误。
- 失败被建模为进入判定表的第三种状态。
- 失败被静默吞掉而不记 warning。
- 一个角色的失败连带使其他角色或整个条目退回无图标。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| 六类失败各自等同于该候选不存在 | 尚未实现 | `Missing` |
| 失败候选被跳过后同角色的次优扩展名胜出 | 尚未实现 | `Missing` |
| 单个角色的失败不影响其他角色的解析结果 | 尚未实现 | `Missing` |
| 图标失败不改变插件的可见性、可安装性与可运行性 | 尚未实现 | `Missing` |
| 每次失败产生一条可观测的 warning | 尚未实现 | `Missing` |

### 决策依据

- 图标决策 D5：任何图标失败都等同于该候选文件不存在。
- 图标决策不变量 5。

## Image Format Must Be Decided by Magic Bytes and Bounded Without Decoding Pixels

### 风险

格式判定若信任扩展名，一个内容是 SVG 的 `logo.png` 就绕过全部 SVG 安全校验。尺寸若不在解码之前
判定，一张高压缩比的巨幅位图会在 webview 的解码器里展开成解压炸弹，而 marketplace 列表会同时
渲染几十个图标。WebP 是容器而非单一格式，只解析其中一种子格式会把最常见的合法图标误判为损坏。

### 前置状态

- 候选文件的字节可能是 SVG、PNG、WebP、JPEG、GIF 或白名单之外的任意格式。
- WebP 可能是 `VP8 `、`VP8L` 或 `VP8X`；`VP8X` 可能带 ANIM 标志。
- 字节数与像素数可能各自超出或未超出上限。
- 扩展名可能与真实字节相符或不符。

### 触发

registry 索引构建与包发现读取候选文件并判定其格式与尺寸。

### 必须成立

- 格式一律由 magic bytes 判定；扩展名不参与判定。
- 判定出的格式必须与扩展名映射出的格式相等，否则该候选按不存在处理。
- 白名单封闭为 SVG、PNG、WebP、JPEG；GIF 与其余格式一律拒绝。
- WebP 的 `VP8 `、`VP8L`、`VP8X` 三种子格式都能读出画布尺寸；`VP8X` 的 ANIM 位使该候选被拒绝。
- JPEG 通过逐段跳转到 SOF 标记读出宽高。
- 50 KiB 字节上限对五个扩展名统一生效，且在格式判定**之前**生效。
- 1024 × 1024 像素上限对三种位图独立于字节上限生效。
- 尺寸解析不解码任何像素，不引入图像解码依赖。

### 禁止结果

- 按扩展名或响应头决定走哪套校验。
- 只解析 `VP8X` 而把 `VP8 ` / `VP8L` 判为损坏。
- 接受动画 WebP 或任何 GIF。
- 只设字节上限，使一张 50 KiB 的 8000 × 8000 PNG 通过。
- 为读出尺寸而引入图像解码依赖。
- 扩展名与真实字节不符的文件仍被服务，使 content-type 与字节不符。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| 格式由 magic bytes 判定，扩展名不参与 | 尚未实现 | `Missing` |
| 扩展名与判定格式不符的候选按不存在处理 | 尚未实现 | `Missing` |
| 白名单封闭，GIF 与其余格式被拒绝 | 尚未实现 | `Missing` |
| WebP 三种子格式都能读出尺寸，ANIM 被拒绝 | 尚未实现 | `Missing` |
| JPEG 由 SOF 段读出尺寸 | 尚未实现 | `Missing` |
| 字节上限统一为 50 KiB 且先于格式判定 | 尚未实现 | `Missing` |
| 像素上限独立于字节上限生效 | 尚未实现 | `Missing` |
| SVG 分支复用既有安全策略（脚本、外部引用、畸形 XML、字节上限） | `ora_utils::svg::tests::rejects_script_element`; `rejects_foreign_object`; `rejects_external_reference`; `rejects_event_handler`; `rejects_malformed_xml`; `rejects_oversized_svg` 覆盖被复用的策略本身，但不覆盖它作为五个候选格式之一时的失败语义 | `Partial` |

### 决策依据

- 图标决策 D3：格式由 magic bytes 判定，白名单封闭且必须与扩展名一致。
- 图标决策 D4：单一 50 KiB 字节上限，位图另加像素上限。
- 图标决策不变量 6、7。

## The Logo Asset Branch Must Refuse Everything Outside the Main Window, Two Roots, and the Closed Candidate Set

### 风险

图标请求没有 surface instance 可绑——它由主窗口为任意插件发起，未安装的 marketplace 条目更是
连 instance 都不存在——因此必须自成一条授权分支，不能复用现有的按 instance 授权链。这条分支若
接受调用方提供的路径、不限制调用窗口、或不把两个根收敛，它就是一条任意文件读取通道，而它服务
的正是已安装插件目录与一份不受信任的 git 检出。

### 前置状态

- 一个 `ora-plugin` 协议请求，调用方可能是主窗口，也可能是某个插件的 workbench 或 webview。
- URL 可能携带越界的主题角色、越界的扩展名、路径分隔符，或 percent 编码的穿越序列。
- 请求的插件 id 可能已安装、只存在于某个源检出、或完全不存在。

### 触发

`ora-plugin` 协议 handler 处理图标分支的请求。

### 必须成立

- 只有主窗口 label 的调用方被接受。
- URL 只由插件 id、主题角色与扩展名三段构成。
- 主题角色取自 `universal` / `light` / `dark`，扩展名取自五个允许值，两者都先经封闭集合校验。
- id 解析到两个根之一：已安装包根，或该 id 所属源检出下的条目目录。
- 两个根都经 `CanonicalPathRoot` 收敛，且 percent 解码只发生一次。
- 文件名由角色与扩展名拼出，从不取自 URL 中的任意字符串。
- content-type 由扩展名查表得出；应答既不嗅探字节，也不读取索引。

### 禁止结果

- 插件的 workbench 或 webview 能取到图标分支的应答。
- URL 中出现路径片段并被用于定位文件。
- percent 编码的穿越序列在收敛之前被二次解码。
- 一个插件的 id 解析到另一个插件的目录，或解析到两个根之外。
- 应答时回头查索引以确定文件名或 content-type。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| 非主窗口调用方被拒绝 | 尚未实现 | `Missing` |
| 越界的主题角色与扩展名被拒绝 | 尚未实现 | `Missing` |
| 携带路径片段或穿越序列的 URL 被拒绝，且只解码一次 | 尚未实现 | `Missing` |
| id 只解析到两个根之一，且经 `CanonicalPathRoot` 收敛 | 尚未实现 | `Missing` |
| content-type 由扩展名得出，应答不读索引、不嗅探字节 | 尚未实现 | `Missing` |

### 决策依据

- 图标决策 D1：图标分支自成一路，URL 只承载 id、角色与扩展名。
- 图标决策不变量 1、13。

## A Logo-Shaped Index Cache Change Must Degrade to Not-Yet-Synced Instead of Failing the Marketplace

### 风险

`registry_index.json` 是持久化的派生缓存，形状与写它的代码绑定。`logo` 从 SVG 源文本变为对象，
对升级前写下的缓存而言是**类型不符而非字段缺失**，根决策的 serde 默认值机制不适用。而现状下
`RegistryIndex::load` 从不比对已有的 `INDEX_VERSION` 字段，`list_available_plugins` 又只把
`Io(NotFound)` 降级为空列表、其余错误一律转成内部错误——因此一次形状变更不会退化成"没有图标"，
而会让整个市场端点报错。

### 前置状态

- 磁盘上存在一份由旧版本写入的 `registry_index.json`：`logo` 是 SVG 源文本，`version` 是旧值。
- 宿主已升级到写入新形状的版本。
- 源检出存在且内容有效。

### 触发

用户打开插件市场（`listAvailablePlugins`），随后手动同步一次（`syncAvailablePlugins`）。

### 必须成立

- `load` 比对文件中的 `version` 与当前 `INDEX_VERSION`，不等即视为缓存不可用。
- 缓存不可用（版本不符或反序列化失败）与文件不存在归入同一条降级路径：返回空列表与
  `updated_at = 0`。
- 同步一次后索引以新形状与新版本号原子重写，此后每次读取正常。
- 该窗口每次索引 schema 变更出现一次，不随每次启动重现。

### 禁止结果

- 版本不符或反序列化失败被转成 `BackendError::internal`。
- 按新 schema 解析旧形状缓存并接受其中一部分字段。
- 为修复缓存而在打开市场时触发网络同步。
- 缓存不可用与"源里确实没有插件"在响应中无法区分。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| `load` 比对 `INDEX_VERSION`，不等即视为缓存不可用 | 尚未实现；该字段今天只被写入，`version()` 的唯一调用方是 `ora_plugin_registry::index::tests` 中的构建断言 | `Missing` |
| 缓存不可用与文件不存在走同一条降级路径 | 尚未实现 | `Missing` |
| 降级响应携带 `updated_at = 0`，与"已同步但为空"可区分 | 尚未实现；前端已有 `updatedAt === 0n` 的独立分支，但无断言证明缓存不可用时会到达该分支 | `Missing` |
| 同步一次后索引以新形状重写且后续读取正常 | 尚未实现 | `Missing` |
| 打开市场的读取路径不触发网络 | 尚未实现 | `Missing` |

### 决策依据

- 图标决策"兼容性边界"中索引缓存一节：提升 `INDEX_VERSION`、补上版本校验、并把缓存不可用并入
  既有的空列表降级路径。
- 图标决策不变量 12。
- 分发根决策：打开插件市场只读缓存索引，不触发网络。
