---
status: implemented
---

# 插件清单：resolver 版本化的双形态 `orax.toml` 与单一验证域模型

**release `url` 的类型已由
[Source 级插件包获取决策](../registry/20260904-source-configured-artifact-retrieval.md) 修改**：字段名保持不变，
但其值现在是 HTTPS URL 或对象 key 的 `ReleaseLocator`；具体相容性由所属 Marketplace Source 的
Direct HTTPS / S3 SigV4 获取方式决定。本文下述仅允许 HTTPS 的表述以后者为准。

每个 Ora 插件用一份 `orax.toml` 自述身份、`kind`、版本与 kind 专属声明。该清单存在两种形态：
marketplace 源仓库中的 **release 形态**（可携带可下载资产的地址与摘要），以及 `.orax` 包内随包
分发的 **installed 形态**（描述已落盘的这一份包）。两种形态共用同一套字段、同一套语义校验和
同一个验证后的域模型，差别只在于各自允许与禁止的发布相关 section。schema 的断裂性变化由单调
版本号 `resolver` 承担，未知字段一律忽略。清单解析是纯函数：不做 I/O，也不做需要包路径或安装
上下文才能判定的跨值策略。

该决策已由 `ora-plugin-manifest` 落地，并被 `ora-plugin-manager`（发现、本地导入、安装）与
`ora-plugin-registry`（索引构建）消费。本文固定清单 schema 本身的系统承诺。

本文是 `orax.toml` 字段规则的权威定义。[插件分发根决策](../registry/0-registry-distribution.md)
中关于清单字段的段落是分发视角的摘要，字段语义以本文为准；源拓扑、检出布局、索引构建与安装
路径仍以该决策为准。本决策不承担：

- namespace 归属与完整 Plugin ID 的构成，见
  [来源派生 Namespace 决策](../registry/20260831-source-derived-namespace.md)。
- `assets/config.json` 的 Settings、MCP、Hook 配置编译，见
  [MCP 插件根决策](../category/mcp/0-static-mcp-configuration.md) 及同级 category 决策。
- `.orax` 归档布局、安装事务，以及需要文件系统才能判定的跨值策略（见 D7）。

## D1：一份 schema、两种形态，`resolver` 承担断裂性变化

release 形态必须显式声明 `resolver`；installed 形态可以省略，省略被解释为当前版本。当前只支持
`resolver = 1`，其它值在任何字段语义校验之前就整份拒绝，错误只指向 `resolver`。两种形态由不同
入口解析，但汇聚到同一个验证后的域模型。

两种形态的差异只有发布相关 section（见 D5）。如果把它们建模成两个域模型，文本策略、URL 策略与
kind 配对就要各实现一遍，一次漂移就让“能上架”与“能安装”的边界不再一致——而这两条路径判定的是
同一个作者写下的同一份声明。

resolver 先于字段判定，使旧宿主面对未来 schema 时给出一句“版本不支持”，而不是一串它无法解释的
字段错误；后者会把“你的 Ora 太旧”误报成“你的清单写错了”。

installed 形态允许省略 `resolver` 的代价是：包内清单无法表达“我是更旧的 schema”。这是可以接受
的，因为包内清单只在打包它的那一次与读取它的这台宿主之间使用，其跨版本兼容由 `.orax` 归档整体
承担；而 release 清单要被任意版本的宿主扫描，必须自述版本。

## D2：未知字段被忽略，缺失必填字段仍是硬错误

顶层与所有嵌套 section 都接受并忽略未知字段。必填字段缺失或类型不符始终是硬错误，并作为结构性
失败报告。

源仓库是跨版本共享的：同一个 `registry/` 目录同时被新旧宿主扫描。若未知字段导致整条 listing 被
丢弃，marketplace 只要增加一个可选展示字段，旧版本 Ora 的插件列表就会凭空少掉这些条目，而作者
与用户都无从察觉。断裂性变化交给 `resolver` 表达，正是为了让增量式 schema 变化不必付出这个
代价。

代价是拼错的可选字段（例如 `liscense`）不会报错，而是静默按缺省处理。因此清单校验不能替代打包
期的作者侧检查；宿主侧的沉默是对旧宿主的保护，不是对作者的反馈。

## D3：`identifier` 是清单能声明的全部身份，`title` 只是展示

- 名字段拼写为 `identifier`（不是 `name`），两种形态一致。
- `identifier` 是 1–2 个以 `.` 分隔的 slug 段，总长 ≤128 字节，例如 `weather`、
  `ora-space.weather`。解析**不做任何规范化**：拼写即身份。
- 清单没有 `namespace` 字段。完整 Plugin ID 由宿主决定；installed 形态的 `identifier` 与
  `version` 必须与宿主写下的安装目录一致，否则整包判为损坏，而不是在两个身份中择一采信。
- `title` 可选、≤128 字节，缺省时回退为 `identifier`，因此任何插件都有可展示的名字。

作者无法知道用户通过哪个 marketplace source 安装了这个包，让清单声明 namespace 就是让不可验证
的自述参与身份归属。不规范化拼写是因为 id 会进入文件系统路径、配置行与 Skill 归属行：一次有损
规范化（折叠大小写、互换 `-` 与 `_`）会让两个不同作者的包在大小写不敏感的文件系统上碰撞到同一
个安装目录与同一份配置。

`title` 与身份分离，使作者可以在不改变任何归属的前提下改名；反过来，没写 `title` 的插件在 UI 中
显示 `identifier`，而不是空字符串或占位符。

## D4：`kind` 是封闭集合，并与它独占的 section 严格配对

| `kind` | `[workbench]` | `[webview]` | 可用 target 形态 |
| --- | --- | --- | --- |
| `workbench` | 可选，独占 | 禁止 | 否 |
| `webview` | 禁止 | 必需，独占 | 否 |
| `agent` | 禁止 | 禁止 | 是；installed `[artifact]` 可选 |
| `hook` | 禁止 | 禁止 | 是；installed `[artifact]` 必需 |
| `skill` | 禁止 | 禁止 | 否 |
| `mcp` | 禁止 | 禁止 | 否 |

未知 `kind` 拼写不被接受。这是 D2 的显式例外：未知字段可以忽略，未知 kind 不能。`kind` 决定
运行时权限、生命周期与允许携带的 section；把未知 kind 降级成某种默认插件运行，等于在一个未定义
的权限模型下执行作者的代码。

`[workbench]` 对 workbench 可选而非必需：纯静态页面没有可被页面调用的方法。相应地，空的
`methods` 数组是非法的——“没有方法”由省略 section 表达，不由一个空 allowlist 表达，否则宿主要
额外规定“作者声明了空集合”与“作者什么都没声明”这两种同义状态是否等价。

`[webview]` 对 webview 必需：webview 的全部导航与下载信任边界都在这个 section 内。缺失它的
webview 包没有可判定的信任集合，只能整包拒绝，而不是回退到某个默认策略。

`workbench.methods` 中的方法名拒绝 `ora/` 前缀。该 namespace 属于宿主协议方法，清单不得把
`ora/storage/read` 这类宿主能力暴露给插件页面；`methods` 只能收窄页面可达的范围，永远不会为
`main.js` 增加任何宿主能力。

## D5：发布来源是互斥枚举，installed 形态只自述 `[artifact].target`

release 形态：

- 顶层 `url` + `sha256` 表示 **universal 资产**，两者必须同时出现。
- `[[targets]]` 表示 **精确 target 资产**，每项必须同时含 `target`、`url`、`sha256`，且
  `target` 在同一版本内不得重复。
- 两种形态互斥；两者都缺省是合法的——条目仍被索引与展示，只是本机不可安装。
- `[[targets]]` 只允许 `hook` 与 `agent` 使用。
- release 形态出现 `[artifact]` 直接报错。

installed 形态：

- `[[targets]]` 直接报错：包内清单不承载任何下载地址。
- `[artifact].target` 是这一份物理包对自己编译目标的自述，同样只允许 `hook` 与 `agent`。
  installed hook 缺少它是错误；installed agent 缺少它是合法的。
- installed 清单仍可携带顶层 `url`/`sha256`（打包工具会把来源写回包内），但它们不构成可下载
  release：解析结果的发布来源恒为空。

把“universal 还是 targeted”建模成一个枚举而不是若干 optional 字段，让“两者都在时听谁的”这个
问题在类型层面消失；一条这样的优先级规则只要在任何一个 consumer 上实现遗漏，就会静默下载错误
的资产。

hook 必须自述 target 而 agent 不必，是因为二者与原生二进制的关系不同：hook **就是**那个二进制，
缺少自述就无法在本地 `.orax` 导入路径上独立证明主机兼容（该路径没有 marketplace 元数据可
依赖）；agent 可以完全不打包二进制、由 PATH 解析它驱动的 CLI，这样的通用包没有 target 可述，
强制它声明只会逼作者填一个假值。

target 匹配是精确的：不跨架构、操作系统、libc 或 ABI 回退。`x86_64-unknown-linux-gnu` 不能替代
`x86_64-unknown-linux-musl`，因为两者的二进制不可互换；一次“相近即可”的回退会把打包错误推迟成
用户机器上的运行时崩溃。

## D6：会被当作能力使用的标量在解析期收窄为不变量类型

| 清单值 | 域类型 | 关键策略 |
| --- | --- | --- |
| `identifier` | `PluginName` | ≤128 字节、≤2 段、每段是 slug；不规范化 |
| `version` | `semver::Version` | 完整 semver 语法 |
| `dependencies.ora` | `semver::VersionReq` | 空的 `[dependencies]` 归一为“未声明” |
| `sha256`、`targets[].sha256` | `Sha256Digest` | 恰好 64 个十六进制字符，解码为 32 字节；渲染统一为小写 |
| `url`、`targets[].url` | `ReleaseUrl` | HTTPS、无 userinfo、无 fragment；**允许 query**（签名下载） |
| `homepage`、`head.repository` | `HomepageUrl`、`RepositoryUrl` | HTTPS、无 userinfo、无 fragment；**拒绝 query** |
| `head.branch` | `GitBranchName` | 合法短分支名 |
| `targets[].target`、`artifact.target` | `HookTarget` | 白名单内的规范 rustc triple，精确匹配 |
| `workbench.methods[]` | `MethodName` | `[a-z0-9_]` 段以 `/` 分隔；拒绝 `ora/` 前缀与重复项 |
| `webview.start_url` | `StartUrl` | HTTPS、无 userinfo、无 fragment |
| `webview.allowed_origins[]` | `Origin` | 归一化 origin；无路径、query、fragment，也无通配 |
| `webview.downloads.rules[].page.path_prefix` | `PathPrefix` | 以 `/` 开头，恰好一次 percent 解码后存储 |
| `webview.downloads` 的处置 | `DownloadDisposition` | 封闭的 `auto` / `prompt` / `reject` |

共享文本策略：`title`、`description`、`license` 必须非空、无首尾空白、无控制字符，字节上限分别
为 128、1000、256，且 `license` 必须是 ASCII。所有 URL ≤2048 字节。

两个容易误解的边界：

- 链接类字段接受 Markdown `[label](target)` 包装，并在校验前解包一次，因为 marketplace 作者
  习惯这样书写。解包后的 target 仍需通过该字段完整的 HTTPS 策略；解包不是放宽策略，只是去掉
  一层书写习惯。
- `path_prefix` 与被匹配的页面路径都只解码一次，并且走同一个解码函数。解码两次会让 `%252F`
  变成 `/`，使一条规则匹配到作者从未授权的路径。

这些值最终会被拼进进程参数、导航判定、下载校验与文件系统路径。若它们以 `String` 进入域模型，
每个 consumer 都要重复一遍策略，遗漏一次就是一个由清单作者触发的越权。收窄的代价是新增一种
合法写法必须改动类型而不是改动某个调用点——这正是我们希望的：策略只有一处。

## D7：清单只做单值语义校验，不做 I/O；错误必须指向一个稳定的 field path

清单解析不读文件系统、不发网络请求，也不检查被引用的文件是否存在。以下判定都不属于清单层，
而由持有包路径与安装上下文的 `ora-plugin-manager` 承担：

| 判定 | 归属 |
| --- | --- |
| `identifier`、`version` 是否与安装目录一致 | 包发现 |
| `allowed_origins` 是否重复、是否覆盖 `start_url`、规则是否互相遮蔽 | webview 包校验 |
| 入口文件、页面、可执行文件是否存在且落在包内 | kind 专属包校验 |
| 清单文件本身的大小、编码与文件类型（≤1 MiB、UTF-8、普通文件） | 包发现 |

错误报告有两层，但对 consumer 呈现同一种可用性：结构性失败（TOML 语法、类型不符）携带出错值的
点路径，例如 `webview.downloads.rules[0].page`；语义失败携带结构化字段枚举，渲染为同样形状的
稳定点路径，例如 `webview.allowed_origins[1]`、`artifact.target`。consumer 不需要解析错误消息
就能定位字段，包发现直接把它转成 discovery issue 的字段路径。

校验按字段声明顺序执行，因此第一个错误是确定的：同一份坏清单在任何机器上都报同一个字段。作者
面对的问题始终是“我的包为什么装不上”，稳定且确定的 field path 是这个问题唯一可自动化的答案；
不确定的错误顺序会让打包期检查与宿主报错互相矛盾。把 I/O 留在清单之外的附带收益是清单校验成为
纯函数，其测试不需要临时目录。

## 为什么不是这些替代方案

| 方案 | 否决理由 |
| --- | --- |
| release 与 installed 各自一套域模型 | 文本、URL、kind 配对策略要实现两遍；一次漂移就让“能上架”与“能安装”在判定同一份声明时得出不同结论 |
| 用 `deny_unknown_fields` 严格拒绝未知字段 | marketplace 新增一个可选展示字段就会让旧宿主整条丢弃 listing，作者与用户都察觉不到；断裂性变化本就该由 `resolver` 表达 |
| 接受未知 `kind` 并降级为默认插件 | `kind` 决定权限、生命周期与允许的 section；未定义的 kind 只能在未定义的权限模型下执行作者代码 |
| 清单声明 `namespace` | 作者不知道用户从哪个 source 安装；自述 namespace 让不可验证的输入参与身份与目录归属 |
| 规范化 `identifier` 拼写（折叠大小写或互换 `-`/`_`） | id 进入文件系统路径与归属行；有损规范化会让两个不同作者的包在大小写不敏感的文件系统上碰撞 |
| 允许同时声明 `url`/`sha256` 与 `[[targets]]` 并定义优先级 | 需要一条“哪个赢”的规则，而该规则在任何一次实现遗漏时都会静默下载错误资产；它相对枚举的唯一优势只是让作者少删几行 |
| target 允许相近架构或 ABI 回退 | `-gnu` 与 `-musl`、`-msvc` 与 `-gnu` 的二进制不可互换，回退把打包错误推迟成用户机器上的崩溃 |
| 用空的 `workbench.methods` 表示“没有方法” | 产生两种同义状态，宿主必须额外规定它们是否等价；省略 section 已经表达了同一件事 |
| 在清单层做跨值策略与文件存在性检查 | 清单不持有包路径与安装上下文；这些检查同时需要 I/O 和只在宿主侧成立的假设，会让清单校验既不可纯测试，也不可复用于打包期 |
| 把 URL、digest、方法名、origin 保留为 `String` | 每个 consumer 重复一遍策略，遗漏一次就是清单作者可触发的越权 |

## 不变量

1. 一份清单文本要么产生一个所有字段都已通过其类型策略的域对象，要么产生一个带稳定 field path
   的错误；不存在部分校验的清单。
2. 只有 `resolver = 1` 被接受，且该判定先于任何字段语义校验。
3. 未知字段在任何层级都不改变解析结果；必填字段缺失或类型不符始终是硬错误。
4. `kind` 是封闭集合，未知拼写不被接受；每个 kind 只能携带它独占的 section，且 `[webview]` 对
   webview kind 必需。
5. 清单不声明 namespace；installed 形态的 `identifier` 与 `version` 必须与宿主写下的安装目录
   一致，否则整包无效。
6. 一份 release 清单最多声明一种发布来源；installed 清单不声明任何可下载 release。
7. `[[targets]]` 与 `[artifact]` 只属于可能内含原生可执行文件的 kind；target 匹配精确，不跨
   架构、操作系统、libc 或 ABI 回退。
8. 清单解析不产生任何 I/O 或外部副作用；同一份文本在任何宿主上产生同一个域对象，或同一个首个
   错误。

## 本决策未解决的问题

- `resolver = 2` 的引入路径尚未定义：哪些变化必须提升版本、旧宿主如何在 UI 中呈现“版本不支持”
  的条目。当前只保证旧宿主整份拒绝，而不是部分解释。
- 清单不校验 `license` 是否为有效 SPDX 表达式，只保证它是有界的 ASCII 文本；引入 SPDX 校验需要
  同时决定未知许可证标识的处置。
- 清单解析 `dependencies.ora`，但宿主如何执行该约束（拒绝安装、警告，还是仅展示）不在本决策
  范围内。
- `HookTarget` 白名单随宿主支持的平台增长；新增 triple 需要同时更新白名单与打包流程，该列表的
  演进策略尚未由 ADR 固定。
- 打包期的作者侧校验（拼错的可选字段、与 `assets/` 实际内容不一致的声明）不在本决策范围内；
  D2 的静默容错使这类检查必须由打包工具补上。

## 核心测试依据

见 `specs/test-cases/desktop/plugin/manifest/schema-and-validation.md`。
