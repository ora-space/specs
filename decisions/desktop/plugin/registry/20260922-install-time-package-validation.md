---
status: implemented
---

# 插件分发：安装期以包内清单校验并保证失败零残留

> 历史决策：D2 中由身份决定的 `effect_sources` 行已由[Skill 按存储路径以会话清单交给 Agent](../../core/agent/session/skill/0-skill-catalog-from-storage-paths.md)（`approved`）随 Effect 退役删除；安装目录、私有数据目录与 `skills` 行的身份一致性规则不变。

安装 marketplace 插件时，宿主不再用 registry 清单校验解包结果，而是解析并校验**包内** `orax.toml`
（installed 形态），用与 discovery 完全相同的校验路径判定这个包装上之后能否被发现。同时把下载
产物 `.orax` 明确定义为传输中间物：安装开始先清扫缓存中的历史残留，成功或失败都删除本次下载，
失败时再清掉空的 `<namespace>/<name>/` 父目录。结果是两个系统承诺：**装上即可被发现**，以及
**安装失败后磁盘零残留**。

本 ADR 为 `implemented`，代码已随同一变更落地。涉及 `ora-plugin-manager`（安装编排、缓存所有权、
包内清单解析）、`ora-contracts`（`plugin_package_invalid` 错误契约）、`crates/backend`（错误分类
与映射）以及前端错误文案；清单字段规则仍由
[清单根决策](../manifest/0-versioned-dual-form-manifest.md)拥有，本文不重复。

本决策不改变源检出、索引缓存格式、namespace 派生或 release 选择规则，也不引入安装期的包内自声明
`sha256` 校验。存量已落盘的坏包如何发现、展示和清理不在本决策范围：本文只保证新的坏包不再产生。

## 继承与修改

本决策直接继承本链前序
[本地 pack 与隐藏条目索引决策](20260915-local-pack-installation-and-hidden-listings.md)，在其继承的
[分发根决策](0-registry-distribution.md)、[Namespace 决策](20260831-source-derived-namespace.md) 与
[Source 级插件包获取决策](20260904-source-configured-artifact-retrieval.md) 上收紧安装期校验与
下载产物生命周期。

| 前序约定 | 本决策 |
| --- | --- |
| Git 同步提供元数据，源检出是事实来源 | 继承，不变；registry 清单仍决定下载地址与摘要 |
| namespace 绑定唯一 source，安装目录以 namespace 为首层 | 继承，不变；包内清单不参与 namespace 判定 |
| 安装 resolve 后按 Source 获取方式选择和下载制品 | 继承，不变；下载后新增包内清单校验步骤 |
| 任何下载先通过 SHA-256 才进入 `installed/` | 继承，不变；摘要校验仍在解包之前，且失败同样清理缓存 |
| target 形态比对包内清单 `[artifact].target` 与所选 release target | 继承，不变；改为复用已解析的包内清单，不再二次读取 |
| pack 从源检出本地创建安装态，普通 listing 走下载链路 | 继承，不变；pack 成员走同一条安装路径，自动获得同样的校验与清理 |
| 索引原子写入，打开市场只读缓存 | 继承，不变；缓存目录新增“只保留派生索引”的所有权约定 |
| 安装失败不产生 ownership 关系或半成品目录 | 修改：明确到缓存文件与空父目录级别，失败后磁盘零残留 |

## 决策

### D1：marketplace 安装必须解析并校验包内清单

安装解包到 staging 后，必须读取包内 `orax.toml` 并以 installed 形态解析，再以该清单执行与
discovery 相同的宿主侧校验。缺失清单、清单不是普通文件、TOML 语法错误、缺少必填字段、字段取值
不合法、kind 专属贡献（如 `assets/config.json` 的 MCP 形状）不合格，都在提交之前失败。

为什么用包内清单而不是 registry 清单：落盘之后，discovery 读的是包内清单，registry 清单不会被
复制进安装目录。用 registry 清单校验等于校验了一个不会落盘的文档，包内缺字段、语法错、kind 与内容
不符都能“安装成功”，随后在 discovery 阶段以内部错误暴露，并留下一个坏包目录。

### D2：身份一致性以 registry 清单为对照，展示字段以包内为准

包内清单的 `identifier`、`version`、`kind` 必须与 registry 清单一致，不一致即拒绝安装。`title`、
`description`、`homepage`、`license`、图标等展示字段允许不同，装后以包内清单为准。

身份决定安装目录、插件私有数据目录、`skills` 行与 `effect_sources` 行；两处不一致时无论采信哪一侧，
都会把一份身份的数据挂到另一份身份上。展示字段不承载身份，作者在发布包内更新文案不应因此被拒绝。

### D3：校验发生在 staging 阶段，提交是最后一步

校验全部通过后，才用一次 rename 把 staging 目录提交到
`installed/<namespace>/<name>/<version>/`。任何一步失败都不提交，staging 目录由临时目录生命周期
自动回收。

这保持了既有的原子提交边界：installed 树中不会出现“写了一半”的版本目录，也不会因为校验失败
而在安装目录留下部分文件。

### D4：下载产物是传输中间物，安装结束时缓存只剩派生索引

`plugins/cache/` 中唯一持久住户是 registry 同步产出的 `registry_index.json`；它是市场列表的唯一
数据源，任何清理都不得触碰。除此之外的 `*.orax` 都是传输中间物：

1. 每次安装开始先扫描缓存，删除所有 `*.orax`（包括旧扁平布局直接写在缓存根下的历史残留），
   兜住进程崩溃留下的窗口；
2. 成功与失败路径都删除本次下载的 `.orax`；
3. 失败时 best-effort 删除空的 `<namespace>/<name>/` 父目录；
4. 清理只精确匹配 `*.orax`，因此下载中的 `<destination>.tmp` 与 `registry_index.json` 都不在
   清理范围内。

缓存此前只增不减：每次安装写一个 `.orax`，没有任何读取者、复用、续传或回收机制，装一次留一个。
把清理放在安装生命周期内、而不是新增 GC 任务，是为了让“谁写谁清”与失败路径共享同一个边界。

### D5：缓存文件名以 namespace 为目录层，消除跨源同名碰撞

下载产物路径为 `<cache>/<namespace>/<name>-<version>.orax`。

namespace 与插件名都允许连字符，扁平拼接（如 `<namespace>-<name>-<version>.orax`）不是单射：
`foo` + `bar-baz` 与 `foo-bar` + `baz` 会映射到同一路径。两个源发布同名同版本插件时，并发安装可能
互相覆盖对方的字节，导致一个安装解出另一个源的包。把 namespace 作为目录层后路径由构造唯一，
同时保持缓存根下只有索引与 namespace 目录两种住户。

### D6：包内容失败以单一错误码携带字段与原因

新增公开错误 `plugin_package_invalid`，参数为 `{field, message}`，分类为 `Unprocessable`。缺失清单、
清单语法或字段错误、宿主侧校验失败、身份不一致、下载摘要不匹配都映射到该码，`field` 取稳定字段路径
（`description`、`webview.allowed_origins[0]`、`assets/config.json`、`orax.toml`、`sha256` 等）。
清单解析失败的字段归因由清单 schema 的所有者 `ora-plugin-manifest` 提供：缺少必填字段归因到该字段
的完整路径（嵌套时如 `webview.downloads.rules[0].page.path_prefix`），TOML 语法错误属于整份文档，
`field` 取 `orax.toml`。安装侧不自行解析反序列化报文，避免两处对同一 schema 各持一份归因规则。

传输失败、宿主 I/O 失败与解包失败不进入该码，仍为 `internal_error`：前者没有可操作的字段，后者
属于归档错误族而非清单字段错误。分类为 `Unprocessable` 是因为这类失败重试同样的包不会成功，
需要用户或作者修改内容。

### D7：已安装判定提前到下载之前

`AlreadyInstalled` 检查移到下载之前。已安装的版本不需要为了被拒绝而先下载整个包；`update` 路径
自身已先比较版本，提前检查不改变其语义。

### D8：范围边界

- 存量坏包（已落盘、discovery 只记 warning、UI 不可见）另开任务，本决策不提供发现或删除入口。
- 包内自声明 `sha256` 的语义不成立（它声称对包含自身的归档做摘要），本决策既不新增校验也不移除
  本地导入路径上既有的该项校验。
- pack 安装逻辑不新增分支：pack 成员复用同一条安装路径，自动获得本文的校验与清理。
- 桌面更新器缓存（`~/.ora/cache/update/`）属于另一所有者，不在本文范围。

## 为什么不是这些替代方案

| 替代方案 | 否决理由 |
| --- | --- |
| 继续用 registry 清单校验解包结果 | 校验的不是会落盘的内容，包内缺字段或语法错仍能安装成功，随后以内部错误在 discovery 阶段暴露 |
| 装完再校验、失败回滚 | 需要为“已提交但不可用”的中间态设计回滚与恢复；staging 阶段拒绝把失败挡在提交之前，成本更低 |
| 用包内清单作为唯一事实来源（含下载地址） | 下载地址与摘要是分发信息，由源检出拥有；包内清单不携带可信的获取配置 |
| 保留缓存供复用或续传 | 当前没有读取者，复用需要过期、失效与并发协议；先按“谁写谁清”收敛，未来若真需要复用再引入所有权 |
| 扁平缓存文件名 | 不是单射，跨源同名同版本会共享路径并可能互相覆盖字节（D5） |
| 安装开始时清空整个缓存目录 | `registry_index.json` 是市场列表唯一数据源，删除它会让 UI 失去缓存内容 |
| 校验包内自声明 `sha256` | 自指摘要无法自洽，真实打包工具也不写入该字段；registry 清单的摘要才是成立的强校验 |
| 把解包失败也归入 `plugin_package_invalid` | `field` 没有诚实的取值；归档错误已有独立错误族，强行合并会让契约字段失去含义 |

## 不变量

1. 进入 `installed/` 的每个包，都能通过 discovery 对同一目录的校验：装上即可被发现。
2. 安装失败后磁盘零残留：没有版本目录、没有空 `<namespace>/<name>/` 父目录、没有本次下载的 `.orax`。
3. `plugins/cache/` 在一次安装结束时只保留派生索引；清理只匹配 `*.orax`，不触碰索引与下载中的 `.tmp`。
4. 包内清单与 registry 清单的 `identifier`/`version`/`kind` 必须一致；不一致一律拒绝，不做仲裁。
5. 展示字段不参与一致性判定，装后以包内清单为准。
6. 包内容失败必须携带可操作的字段路径与原因，不得降级为无字段的内部错误。
7. 校验与清理在提交之前完成；installed 树中不存在未通过校验的包。
8. 同一插件名与版本在不同 namespace 下不共享缓存路径。

## 落地顺序

1. 安装编排：读取并解析包内清单、身份一致性检查、以包内清单执行宿主侧校验、复用已解析清单做
   target 比对（D1–D3）。完成条件：坏包在 staging 阶段被拒绝，合法包端到端安装成功。
2. 缓存所有权：namespace 目录层、安装开始清扫、成功/失败清理、空父目录清理（D4、D5）。完成条件：
   一次安装结束后缓存只剩派生索引，崩溃残留可在下一次安装被清扫。
3. 错误契约：`plugin_package_invalid` 契约、后端映射、前端中英文案（D6）。完成条件：缺字段与身份
   不一致在 UI 上显示字段级原因，而不是请求 ID。
4. 文档与证据：ADR、核心测试用例与 crate README 同步更新（本文与
   [install-time-package-validation.md](../../../../test-cases/desktop/plugin/registry/install-time-package-validation.md)）。

前两步改变安装期行为：坏包不再落盘，缓存不再增长。第三步改变用户可见错误。下载地址、摘要校验、
target 选择、namespace 派生、索引格式与 pack 编排保持不变。
