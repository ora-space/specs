# 更新前的清单日志核心用例

依据[更新清单携带最近 10 个版本的日志，用户在确认安装之前看到](../../../decisions/desktop/release-notes/20261124-manifest-notes-before-update.md)。
下文的 D 编号和不变量编号都指这篇 ADR，指根决策时另外写明。

ADR 为 `approved`，desktop 的实现尚未合并，下列义务目前都是 `Missing`。证据在实现合并后登记，路径相对 desktop 仓库根目录。

| 不变量 | 用例 | 状态 |
|---|---|---|
| 2、3、5 | [The Manifest Carries The Latest Ten Versions](#the-manifest-carries-the-latest-ten-versions) | Missing |
| 4、7 | [Only The Versions An Update Brings Are Shown](#only-the-versions-an-update-brings-are-shown) | Missing |
| 5、6 | [Unusable Manifest Notes Are Hidden As A Whole](#unusable-manifest-notes-are-hidden-as-a-whole) | Missing |
| 1 | [Manifest Notes Never Change An Update](#manifest-notes-never-change-an-update) | Missing |
| 8 | [Installing Needs One Explicit Confirmation](#installing-needs-one-explicit-confirmation) | Missing |

## The Manifest Carries The Latest Ten Versions

### 风险

清单漏掉目标版本，用户安装前看不到要装的版本改了什么；标准字段类型变化，所有客户端都检查不到更新；文件里以后加的字段泄漏到公开清单。

### 前置状态

一个带日志的 desktop 检出和已签名的安装包；日志多于 10 个版本，含高于目标版本的；分别制造目标版本没有日志、某个历史文件不合格。

### 触发

按正式版 tag 和预发布 tag 生成更新清单。

### 必须成立

- 带不高于目标正式版本的最近 10 个版本，必须包含目标版本；预发布 tag 按它的正式版本取。
- `ora_release_notes` 带 `schema` 和 `versions`，只含已知字段；标准 `notes` 是字符串。
- 目标版本没有日志或任一文件不合格时不写清单，`release` job 失败。
- 某个版本超过 50 条时打包检查失败。

### 禁止结果

- 静默产出没有日志或缺少目标版本的清单。
- 已有的清单被失败的生成覆盖。

### 验证义务与证据

| 验证义务 | 直接证据 | 状态 |
|---|---|---|
| 版本选择、字段投影和标准 `notes` | 尚无 | `Missing` |
| 目标版本没有日志或文件不合格时不写清单 | 尚无 | `Missing` |
| 超过 50 条时带 `--tag` 的打包失败 | 尚无 | `Missing` |
| 真实 `release` job 失败与成功 | 尚无 | `Missing` |

### 决策依据

ADR D1，不变量 2、3、5。

## Only The Versions An Update Brings Are Shown

### 风险

展示已经装了的版本，或展示与要安装的版本不符的日志，用户据此作出错误的决定。

### 前置状态

清单有多个版本；当前和目标分别是正式版、预发布、带 `+build`。另有一次成功的检查之后检查失败、保留旧的 `Ready`。

### 触发

检查更新，打开设置的更新日志页。

### 必须成立

- 展示 `(stable(当前), stable(目标)]`，按版本倒序，按界面语言。
- 日志来自产生当前 `Ready` / `ManualUpdate` 状态的那一次检查；保留旧 `Ready` 时日志仍是那个版本的。
- 日志只在“可更新”区块里，历史列表不变。
- 展示清单日志不写入“已提示过的最高版本”，更新后的卡片照弹。

### 禁止结果

- 清单日志进入历史列表，或被持久化。
- 前端拿到原始清单。

### 验证义务与证据

| 验证义务 | 直接证据 | 状态 |
|---|---|---|
| 区间与预发布、`+build` 的归一化 | 尚无 | `Missing` |
| 日志与更新状态绑定，保留 `Ready` 时不换 | 尚无 | `Missing` |
| 清单日志不进入历史列表、不影响已提示记录 | 尚无 | `Missing` |

### 决策依据

ADR D3、D5，不变量 4、7。

## Unusable Manifest Notes Are Hidden As A Whole

### 风险

清单没有签名。接受超限或不合规则的内容会冲击界面或误导用户；只丢掉一部分，用户会以为看到的是全部。

### 前置状态

清单的日志分别：超过 10 个版本、某版超过 50 条、文字超过 500 码点、分类未知、缺一种语言、含控制或双向控制字符、日期不存在、`schema` 未知；另有带未知字段、没有日志字段的清单。

### 触发

检查更新后读取日志。

### 必须成立

- 任一不合规则的情形都隐藏全部日志，并在 Desktop 日志里记录原因。
- 未知字段被忽略；没有日志字段不算异常。
- 打包检查与客户端对同一组样例判断一致。
- 日志按纯文本显示，不产生元素、样式或链接。

### 禁止结果

- 只展示剩下的合格部分，或截断文字。
- 前后两端对同一条内容判断不同。

### 验证义务与证据

| 验证义务 | 直接证据 | 状态 |
|---|---|---|
| 每种不合规则的情形都整组隐藏 | 尚无 | `Missing` |
| 未知字段忽略、未知 `schema` 隐藏 | 尚无 | `Missing` |
| 打包检查与客户端共用样例 | 尚无 | `Missing` |
| “可更新”区块只按纯文本 | 尚无 | `Missing` |

### 决策依据

ADR D2，不变量 5、6；根决策不变量 6、8。

## Manifest Notes Never Change An Update

### 风险

日志是附带内容。它缺失、损坏或超限时若改变更新结果，用户会因为一段说明文字装不上修复版本。

### 前置状态

更新可用；清单的日志分别为合格、缺失、损坏、超限。

### 触发

检查、下载、安装。

### 必须成立

- 更新器成功解析清单之后，检查、下载、安装的结果与没有日志时相同。
- 没有日志时“可更新”区块仍显示可更新的版本和安装操作。

### 禁止结果

- 日志问题变成更新错误，或使安装操作不可用。

### 验证义务与证据

| 验证义务 | 直接证据 | 状态 |
|---|---|---|
| 日志的各种情形都不改变更新状态 | 尚无 | `Missing` |
| 没有日志时仍能安装 | 尚无 | `Missing` |

### 决策依据

ADR D2、D3，不变量 1；根决策不变量 7。

## Installing Needs One Explicit Confirmation

### 风险

安装会重启应用，而更新模块不检查有没有正在运行的会话。误点一次就重启会丢掉用户正在做的事。

### 前置状态

有可安装的更新；另有只能手动更新的安装。

### 触发

点更新入口、点“立即更新”、在确认框里取消或确认。

### 必须成立

- 更新入口只切到更新日志页，不安装。
- 点“立即更新”弹出不带日志的确认框，写明会自动重启；确认后才安装，取消不安装。
- 手动更新的安装能看日志，有固定文案，没有安装按钮。

### 禁止结果

- 没有确认就安装。
- 手动更新的安装出现安装操作，或使用清单里的链接。

### 验证义务与证据

| 验证义务 | 直接证据 | 状态 |
|---|---|---|
| 确认前不安装，取消不安装 | 尚无 | `Missing` |
| 手动更新没有安装操作 | 尚无 | `Missing` |
| 真实应用里确认后安装并重启 | 尚无 | `Missing` |

### 决策依据

ADR D3、D4，不变量 8。
