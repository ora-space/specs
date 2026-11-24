# 更新日志的打包与展示核心用例

依据[Desktop 更新日志随 main 维护，打包时检查并内置到安装包](../../../decisions/desktop/release-notes/0-release-notes-in-main-bundled-at-packaging.md)。
下文的 D 编号和不变量编号都指这篇 ADR。

ADR 为 `approved`，desktop 尚未实现，下列义务目前都是 `Missing`。证据在实现合并后登记，路径相对 desktop 仓库根目录。

| 不变量 | 用例 | 状态 |
|---|---|---|
| 1、2、3 | [Tagged Packaging Refuses Missing Or Invalid Notes](#tagged-packaging-refuses-missing-or-invalid-notes) | Missing |
| 9 | [Notes Cover The Packaged Commit Range](#notes-cover-the-packaged-commit-range) | Missing（人工核对） |
| 4、5 | [Bundled Notes Need No Network](#bundled-notes-need-no-network) | Missing |
| — | [The Update Card Lists Each Unseen Version Once](#the-update-card-lists-each-unseen-version-once) | Missing |
| — | [Fresh Installs And Upgrades Without A Record Are Told Apart](#fresh-installs-and-upgrades-without-a-record-are-told-apart) | Missing |
| 6 | [Notes Are Rendered As Plain Text](#notes-are-rendered-as-plain-text) | Missing |
| 7 | [Broken Notes Never Block The App Or Updates](#broken-notes-never-block-the-app-or-updates) | Missing |

## Tagged Packaging Refuses Missing Or Invalid Notes

### 风险

带版本号的安装包没有日志、日志损坏，或内置了没有提交的内容，用户看到的就和仓库里审核过的不一致；检查误伤不带版本号的
测试包，则会挡住日常开发。

### 前置状态

一个 desktop 检出，`release-notes/` 里有本版本和历史版本的日志。可以分别制造：缺少本版本文件、某个历史文件格式错误、文件名与
`version` 不符、版本重复、条目超长或含禁止的字符、某个版本超过 50 条、目录里有修改、删除、未跟踪或被忽略的文件、目录不在 git 检出里。

### 触发

用 `package-desktop.ts` 带 `--tag` 或不带 `--tag` 打包；`--tag` 分别取正式版、预发布后缀和 `+build` 后缀。

### 必须成立

- 上述每种问题单独出现时，带 `--tag` 的打包都失败，不产出安装包。
- `v0.7.0`、`v0.7.0-rc.1`、`0.7.0+yellow` 都要求 `release-notes/0.7.0.json`。
- `release-notes/` 之外的本地改动不影响日志检查。
- 不带 `--tag` 时，以上问题都不让打包失败。
- 同一份文件校验也在现有测试里运行；只改 `release-notes/` 的 PR 会触发它。

### 禁止结果

- CI 与本地走不同的检查，或某个带 `--tag` 的入口不经过检查。
- 只检查本版本文件，内置了不合格的历史日志。
- 检查访问网络，或要求 tag 已经存在。

### 验证义务与证据

| 验证义务 | 直接证据 | 状态 |
|---|---|---|
| 带 `--tag` 时，缺文件、任一内置文件不合格都使打包失败 | 尚无 | `Missing` |
| 带 `--tag` 时，`release-notes/` 与检出的提交不一致（含未跟踪和被忽略的文件）或不在 git 检出里都使打包失败 | 尚无 | `Missing` |
| 预发布和 `+build` 后缀归到正式版本的文件 | 尚无 | `Missing` |
| 不带 `--tag` 时不因日志失败 | 尚无 | `Missing` |
| 只改日志的 PR 在现有测试里被校验 | 尚无 | `Missing` |

### 决策依据

ADR D1、D3，不变量 1–4。

## Notes Cover The Packaged Commit Range

### 风险

日志 PR 合并之后又有改动合并，打 tag 时没有核对，随本版交付的改动在本版日志里看不到，或者下个版本把它写成自己的变化。

### 前置状态

本版本的日志 PR 已合并，之后 `main` 又合并了一项用户能感知的改动。

### 触发

准备发版的人核对日志 PR 之后的提交，补写日志，打 tag 并打包。

### 必须成立

- 日志描述打包所用的提交相对上一个正式版 tag 的全部改动，没有用户影响的不单列。
- tag 打在核对过的提交上；公开包和黄区包从同一个提交打。
- 核对之后合并的改动写进下个版本。

### 禁止结果

- 把随本版交付的改动写进下个版本的日志。
- 把这一条登记为已有自动检查。

### 验证义务与证据

| 验证义务 | 直接证据 | 状态 |
|---|---|---|
| 日志覆盖打包范围 | 本文不提供自动检查（见决策的"本决策未解决的问题"）；每次发版由准备发版的人和审 PR 的人核对，核对记录不算代码证据 | `Missing` |
| 公开包与黄区包来自同一个提交 | 没有机制保证，由准备包的人核对 | `Missing` |

### 决策依据

ADR D2、D4，不变量 9。

## Bundled Notes Need No Network

### 风险

展示依赖网络或运行时文件，离线或访问受限的环境里就看不到日志，公开包和黄区包的内容也会不同。

### 前置状态

从同一个提交打的安装包，内置多个版本的日志；机器离线。另有一个没有任何日志的构建。

### 触发

更新后首次启动；打开设置页历史；切换界面语言。

### 必须成立

- 卡片和历史页在联网与离线时内容相同，运行时不读文件、不联网。
- 历史按版本倒序，按界面语言取文字，只含不高于当前正式版本的日志。
- 没有日志时历史页为空，不弹提示。

### 禁止结果

- 显示高于当前正式版本的日志。
- 只内置了本版本的日志，跨版本更新的用户看不到中间版本。

### 验证义务与证据

| 验证义务 | 直接证据 | 状态 |
|---|---|---|
| 构建注入检出里的全部日志，运行时不读文件、不联网 | 尚无 | `Missing` |
| 历史的排序、语言和版本上限 | 尚无 | `Missing` |
| 没有日志时历史为空、不提示 | 尚无 | `Missing` |

### 决策依据

ADR D4、D5，不变量 4、5。

## The Update Card Lists Each Unseen Version Once

### 风险

重复提示打扰用户；漏掉中间版本或误判降级，用户看不到该看的变化。

### 前置状态

本机记录了已提示过的最高版本，安装包内置多个版本的日志。

### 触发

分别以更高、相同、更低的版本启动；以预发布构建启动后再升级到同一正式版；以没有当前版本日志文件的构建启动。

### 必须成立

- 升级后卡片列出（已提示过的最高版本, 当前正式版本] 里的全部日志，展示后记录提高到当前正式版本，再次启动不再提示。
- 降级不提示，记录不降低。
- 预发布构建按它的正式版本显示和记录，之后升级到该正式版不再提示。
- 没有当前正式版本日志文件的构建不提示，也不改记录；版本号与真实发布过的版本相同也一样。

### 禁止结果

- 同一版本提示两次。
- 不带 `--tag` 的构建改写记录，影响之后正式版本的提示。

### 验证义务与证据

| 验证义务 | 直接证据 | 状态 |
|---|---|---|
| 区间内的版本都列出，且只提示一次 | 尚无 | `Missing` |
| 降级不提示、不降低记录 | 尚无 | `Missing` |
| 预发布构建之后的正式版不再提示 | 尚无 | `Missing` |
| 没有当前版本日志文件的构建不提示、不改记录 | 尚无 | `Missing` |

### 决策依据

ADR D5。

## Fresh Installs And Upgrades Without A Record Are Told Apart

### 风险

从不写记录的版本升级上来和全新安装一样没有记录。分不出来，就会漏掉首批升级用户，或者打扰全新安装的用户。

### 前置状态

三种数据目录，都没有提示记录：空目录；v0.6.0 留下的数据库；跳过首个支持版本、由更早版本留下的数据库。

### 触发

以带日志的版本首次启动。

### 必须成立

- 空目录：不提示，记录设为当前正式版本。
- 已有迁移记录的数据库：卡片列出全部不高于当前正式版本的日志。
- 判断依据是打开数据库时已应用的迁移是否为空，并且在本次启动建库之前取得。

### 禁止结果

- 本次启动建好的数据库被当成已有数据。
- 全新安装弹出提示，或升级用户因为没有记录而被跳过。

### 验证义务与证据

| 验证义务 | 直接证据 | 状态 |
|---|---|---|
| 新建数据库判为全新安装，不提示 | 尚无 | `Missing` |
| 已有迁移记录而没有提示记录时判为升级，列出全部适用的日志 | 尚无 | `Missing` |
| 从 v0.6.0 真实升级能看到提示 | 尚无；信号只读代码核实过，没有实测 | `Missing` |

### 决策依据

ADR D5、D7。

## Notes Are Rendered As Plain Text

### 风险

应用没有 CSP，更新清单又不在安装包签名的范围内。日志被当成标记解析时，篡改清单就能在应用里执行内容或伪造链接。

### 前置状态

日志条目含 HTML、Markdown、链接、控制字符或双向覆盖字符，来自内置文件。

### 触发

在卡片和历史页里展示；清单日志在“可更新”区块里的展示见
[更新前的清单日志](manifest-notes-before-update.md)。

### 必须成立

- 所有位置都把条目原样当文本显示，不产生元素、样式或链接。
- 来自更新清单的日志超过长度或条数限制时整组隐藏，由[更新前的清单日志](manifest-notes-before-update.md#unusable-manifest-notes-are-hidden-as-a-whole)验证。

### 禁止结果

- 条目里的标记被解析，或生成可点的链接。

### 验证义务与证据

| 验证义务 | 直接证据 | 状态 |
|---|---|---|
| 卡片与历史页只按纯文本 | 尚无 | `Missing` |

### 决策依据

ADR D1、D5，不变量 6。不变量 8 的义务由[更新前的清单日志](manifest-notes-before-update.md)承担。

## Broken Notes Never Block The App Or Updates

### 风险

日志是附带内容。它缺失或损坏时若影响启动或更新，用户会因为一段说明文字装不上修复版本。

### 前置状态

内置日志注入失败或无法解析。

### 触发

启动应用、打开历史页。

### 必须成立

- 只有日志被隐藏，应用照常启动，历史页不报错。
- 清单日志不影响更新，由[更新前的清单日志](manifest-notes-before-update.md#manifest-notes-never-change-an-update)验证。

### 禁止结果

- 因为日志使应用无法启动，或使历史页报错。

### 验证义务与证据

| 验证义务 | 直接证据 | 状态 |
|---|---|---|
| 内置日志损坏只隐藏日志 | 尚无 | `Missing` |

### 决策依据

ADR D4、D6，不变量 7。
