---
status: implemented
---

# 插件分发：Git Marketplace Source 与派生 Registry 索引

Ora 的插件目录（marketplace）由一个或多个 **Git 仓库源**分发。后端把每个源检出到本地，扫描其
`registry/` 目录下的插件清单，合并成一份轻量的 **派生索引** 供 UI 渲染；安装时再回到源检出中
重新读取该条目的完整清单，取得下载地址与摘要。索引是缓存产物，源检出是唯一事实来源。

该决策已由 `ora-plugin-registry`（源同步、索引构建、条目模型、README/Logo 读取）、
`ora-plugin-manifest`（`orax.toml` 校验）、`ora-plugin-manager`（release 选择与安装）与
`crates/backend`（源配置、契约端点）落地。本文记录稳定的边界与不变量；字段细节以实现为准。

**namespace 的来源与派生规则已由
[来源派生 Namespace 决策](20260831-source-derived-namespace.md) 修改**：清单不再声明
`namespace`，条目唯一标识、跨源去重与安装目录归属均以该决策为准。本文下述涉及
`namespace` 的段落保留是为了说明分发拓扑本身未变的部分（多源、检出、同步、发布资产、
索引缓存），身份归属细节请以后者为准，不在本文重复。

**发布资产定位与下载认证已由
[Source 级插件包获取决策](20260904-source-configured-artifact-retrieval.md) 修改**：Marketplace Source
新增互斥的 Direct HTTPS / S3 SigV4 获取配置，release `url` 可表达 HTTPS locator 或对象 key。
本文下述把 release 地址统称为 URL、只按代理策略下载的段落，以后者为准。

## 分发拓扑

### Marketplace Source

源是可配置、可多个的，持久化在 SQLite 中，对前端暴露为
`MarketplaceSource { url, branch, useProxy }`（[plugin.rs](crates/contracts/src/plugin.rs)）。

- 首次打开后端且配置为空时，种子写入默认源 `https://github.com/ora-space/marketplace` 的 `main`
  分支（[marketplace_sources.rs](crates/backend/src/marketplace_sources.rs)）。
- 源按 `position` 形成**优先级顺序**（source precedence），决定列表与扫描的遍历次序。它不再是
  重复 id 的仲裁规则：`namespace` 由发布该条目的源派生，同一 `identifier` 在不同源下天然是不同
  id，安装与更新解析的是 id 自带的那个源，与列表顺序无关（详见来源派生 Namespace 决策 D6）。
- URL 必须是 HTTPS（`RepositoryUrl`），分支必须是合法的短分支名（`GitBranchName`）；两者在写入
  配置前校验，而不是等到 Git 命令失败。
- 每个源独立携带 `use_proxy`。开启时，该源的 Git 网络操作使用代理环境变量，其条目的 `.orax`
  下载也走同一代理；未配置代理却开启该开关时报 `invalid proxy settings`，不做静默降级。

### 本地检出布局

检出目录由**规范化后的 URL**（大小写、默认端口、尾部斜杠、`.git` 后缀均已归一，规则见来源派生
Namespace 决策 D2）派生：去掉 scheme 后按路径段逐级拼接在 `~/.ora/plugins/sources/` 下，因此同一
仓库的不同写法共享一份检出。

```
~/.ora/plugins/sources/github.com/ora-space/marketplace/            # 源检出
~/.ora/plugins/sources/github.com/ora-space/marketplace/registry/   # 被扫描的目录
~/.ora/plugins/cache/registry_index.json                            # 派生索引缓存
~/.ora/plugins/installed/<namespace>/<identifier>/<version>/              # 安装产物
```

派生规则保证不同源不会共享目录，同时与只有单一默认源时期的路径保持一致。

### 同步

`RegistrySync::sync`（[source.rs](crates/plugin-registry/src/source.rs)）通过注入的
`gitlancer::Git` 运行时执行，本身不直接调用 git 进程，便于测试：

- 检出不存在：`git clone --branch <branch> <url> <dir>`（在父目录中执行，父目录不存在则创建）。
- 检出已存在：`git fetch origin` → `git checkout <branch>` → `git pull --ff-only origin <branch>`。

只用 fast-forward 拉取：源检出是只读镜像，出现分叉时宁可失败也不合并。

## 仓库布局与条目

插件信息放在源仓库的 `registry/` 下。为避免单个目录内文件过多，仓库内按首字母做二级子目录
划分（如 `registry/a/`、`registry/o/`）。这是仓库侧的组织约定：**索引构建器递归扫描 `registry/`
下任意层级的 `orax.toml`**，不依赖固定深度，因此仓库可以调整分桶策略而不需要改动客户端。

一个条目目录包含：

| 文件 | 必需 | 作用 |
| --- | --- | --- |
| `orax.toml` | 是 | 条目清单（release 形态），决定身份、版本与发布资产 |
| `logo.svg` | 否 | 条目图标，构建索引时内联进索引 |
| `README.md` | 否 | 详情页文档，按需读取，不进索引 |

`logo.svg` 与 `README.md` 是**目录约定而非清单字段**：条目可以增删图标或文档而不改变 schema，
索引构建也永远不解析作者提供的路径。

### 条目清单示例

`registry/o/ora-space.weather/orax.toml`：

```toml
resolver = 1

# 1. 身份与元数据（namespace 不是清单字段，见下）
identifier = "ora-space.weather"
title = "Weather"
kind = "workbench"
version = "1.2.0"
description = "获取实时天气信息的 Ora 插件"
homepage = "https://github.com/ora-space/weather"
license = "MIT"

# 2. 发布资产（全平台通用形态）
url = "https://github.com/ora-space/weather/releases/download/v1.2.0/weather-v1.2.0.orax"
sha256 = "feab001d7e9ff4ce66011ebd70791de93eb1554d34d3ea44c33d102a25c1be0a"

# 3. 可选的源码与宿主依赖信息
[head]
repository = "https://github.com/ora-space/weather"
branch = "main"

[dependencies]
ora = ">=0.3"
```

字段规则（[manifest.rs](crates/plugin-manifest/src/manifest.rs)）。**清单 schema 本身的权威定义
已移交 [插件清单根决策](../manifest/0-versioned-dual-form-manifest.md)**：下面的列表是分发视角的
摘要，字段语义、两种清单形态的边界与校验顺序以该决策为准，本文只保留与索引构建、发布资产选择
直接相关的部分。

- `resolver` 必须为 `1`；未知字段被忽略而不报错，使增量式 schema 变化不会让旧版本 Ora 因一个新
  可选字段丢弃整条 listing（断裂性变化仍由 `resolver` 版本号承担）。必填字段缺失依旧是硬错误。
- 名字段拼写为 **`identifier`**（不是 `name`），与安装包内的清单保持一致：1–2 个 slug 段、
  以 `.` 分隔、总长 ≤128 字节，如 `weather`、`ora-space.weather`。
- `namespace` 不是清单字段：条目唯一标识为 `<namespace>/<identifier>`，但 namespace 段由发布该
  条目的 marketplace source 派生，不由作者声明（见来源派生 Namespace 决策）。
- `kind` 是封闭集合：`workbench`、`agent`、`webview`、`skill`、`mcp`、`hook`。
  `[webview]` 是 `webview` 的必需且专属段；`[workbench]` 是 `workbench` 的专属可选段
  （纯静态页面可以不声明方法）。
- `title` 可选，≤128 字节，缺省时回退为 `identifier`；`description` ≤1000 字节；
  `license` ≤256 字节且必须是 ASCII。三者都不允许首尾空白与控制字符。
- `homepage` 必须是 HTTPS 且不带 query；`url` 必须是 HTTPS，允许 query（签名下载）。
- `[artifact]` 只属于安装包内的清单（installed 形态），出现在 registry 清单里直接报错；
  反之 `[[targets]]` 不允许出现在安装包内清单中。

## 发布资产与 Target

一个插件版本要么不发布可下载资产，要么在下面两种发布方式中选择一种。

### 全平台通用资产

`.orax` 在所有受支持平台内容相同时，直接在顶层配置 `url` 与 `sha256`：

```toml
url = "https://github.com/ora-space/weather/releases/download/v1.2.0/weather-v1.2.0.orax"
sha256 = "feab001d7e9ff4ce66011ebd70791de93eb1554d34d3ea44c33d102a25c1be0a"
```

顶层存在 `url` 即表示通用资产，任何平台安装该版本都下载同一个 URL，并用配套 `sha256` 校验。
适用于只包含 JavaScript、配置或其他平台无关资源的插件。`url` 与 `sha256` 必须同时出现。

### Target 专用资产

插件包含平台相关的二进制或原生资源时，改用一个或多个 `[[targets]]` 声明每个编译目标的资产：

```toml
[[targets]]
target = "x86_64-pc-windows-msvc"
url = "https://github.com/ora-space/rtk/releases/download/v0.1.0/rtk-v0.1.0-x86_64-pc-windows-msvc.orax"
sha256 = "1111111111111111111111111111111111111111111111111111111111111111"

[[targets]]
target = "aarch64-apple-darwin"
url = "https://github.com/ora-space/rtk/releases/download/v0.1.0/rtk-v0.1.0-aarch64-apple-darwin.orax"
sha256 = "2222222222222222222222222222222222222222222222222222222222222222"
```

`target` 是规范化的 Rust target triple，同时表达 CPU 架构、操作系统与 ABI，并且只接受
[target.rs](crates/plugin-manifest/src/target.rs) 中的白名单三元组——未知拼写不会被当成“另一种
可安装架构”，而是打包错误。

**只有可能内含原生可执行文件的 kind 允许使用 target 形态**：`hook` 与 `agent`
（`PluginKind::may_ship_targeted_artifact`）。`workbench`、`webview`、`skill`、`mcp`
声明 `[[targets]]` 会被拒绝。

### 无可下载资产的条目

`url`/`sha256` 与 `[[targets]]` 都缺省是合法的：该条目仍然会被索引和展示，但在本机标记为
不可安装（理由为 “this listing has no downloadable release”）。这让 marketplace 可以先上架
条目、后补发布资产，而不需要伪造一个下载地址。

### 选择与校验规则

1. 顶层 `url`/`sha256` 与 `[[targets]]` 互斥，不能同时配置，避免“通用资产还是 target 资产优先”
   的歧义。
2. 配置顶层 `url` 时，`url` 与 `sha256` 必须同时存在，且该资产被视为全平台通用。
3. 使用 `[[targets]]` 时，每项都必须同时包含 `target`、`url` 和 `sha256`。
4. 同一插件版本中 `target` 不得重复。URL 可以相同，但每个 target 仍需显式声明，且摘要必须与
   实际资产一致。
5. target 发布必须**精确匹配**：不做相近架构、相近 ABI 或其他 target 的隐式回退。例如
   `x86_64-unknown-linux-gnu` 不能替代 `x86_64-unknown-linux-musl`。
6. 宿主自身的 triple 是编译期常量（[host.rs](crates/plugin-registry/src/host.rs)）。当前宿主不在
   支持集合内时，target 形态一律判为不兼容，而不是编造一个 triple。
7. 当前平台没有匹配 target 时，该版本在本机显示为“不支持当前平台”，Ora 不会下载其他平台资产
   后尝试运行；`select_release` 在下载之前就返回 `NoArtifactForTarget` / `UnsupportedHost`，
   后端把这类错误映射为 `PluginHostIncompatible` 契约错误。
8. 同步阶段只验证字段形状、kind 约束与 target 唯一性；安装阶段下载选中的资产，再次校验其
   SHA-256，并对 target 形态比对包内清单 `[artifact].target` 与所选 release target
   （`TargetMismatch` / `MissingArtifactTarget` 均 fail closed）。
9. 插件的 canonical ID、kind 和 version 在各 target 包中必须一致。target 只决定发布资产，
   不构成不同的插件身份或版本语义。

## 索引构建与缓存

同步成功后立刻重建索引，并原子覆盖 `~/.ora/plugins/cache/registry_index.json`：

```
[逐个源 clone / fetch+pull] ──> [递归扫描各源 registry/ 目录] ──> [解析 orax.toml + 读取 logo.svg]
    ──> [按 id 排序、跨源去重] ──> [写入同目录临时文件] ──> [原子重命名覆盖 registry_index.json]
```

构建规则（[index.rs](crates/plugin-registry/src/index.rs)）：

- 扫描不跟随符号链接，缺失目录不报错；路径先排序，保证结果与文件系统枚举顺序无关。
- 无法读取或解析的 `orax.toml` 被跳过并记 warning，同时通过 `RegistryBuild::skipped()` 报告，
  单个坏文件永远不会让整次构建失败。
- 条目按 `namespace/identifier` 排序，**去重只发生在同一个源内部**（同源同 id 保留路径序更靠前
  的一个）：不同源发布同一 `identifier` 时 namespace 不同，天然是不同 id，两条都保留在列表中，
  互不遮蔽（见来源派生 Namespace 决策 D6）。
- `logo.svg` 经安全校验后内联进索引：≤50 KiB、良构 XML、禁止 `<script>` / `<foreignObject>`、
  事件处理属性以及指向外部网络资源的 `href`/`xlink:href`（`ora_utils::svg`）。图标缺失是常态且
  静默；图标不安全或不可读只记 warning 并按“无图标”索引，绝不因此把插件从列表中移除。
- `updated_at` 取本地时钟的 Unix 秒（`ora_logging::clock::now_local`）。

### 索引结构

索引兼顾**轻量**与**UI 展现所需的基础元数据**，避免 UI 再逐个读取 `orax.toml`：

```json
{
  "updated_at": 1776244428,
  "version": "1.0",
  "plugins": [
    {
      "id": "official/ora-space.weather",
      "identifier": "ora-space.weather",
      "title": "Weather",
      "kind": "workbench",
      "namespace": "official",
      "version": "1.2.0",
      "description": "获取实时天气信息的 Ora 插件",
      "logo": "<svg xmlns=\"http://www.w3.org/2000/svg\">…</svg>",
      "release_targets": []
    }
  ]
}
```

- **`id`**：由 `namespace` + `identifier` 生成的唯一标识，序列化为 `namespace/identifier` 字符串。
- **`logo`**：已通过安全校验的 SVG 源文本；条目未提供时为 `null`。
- **`release_targets`**：缓存的发布资产目标集合——`null` 表示没有可下载 release，`[]` 表示通用
  资产，非空数组表示精确 target 列表。UI 据此在下载任何资产之前就能禁用不兼容条目的安装。
- 索引**不包含**下载 URL 与摘要：安装时回到源检出读取对应的 registry 清单。
- 新增的展示字段（`title`、`kind`、`logo`、`release_targets`）在反序列化时都有默认值，旧缓存
  可以直接读取，并在下一次同步后自然补齐。

## 读取路径

| 场景 | 契约端点 | 数据来源 |
| --- | --- | --- |
| 打开插件市场 | `listAvailablePlugins` | 只读缓存索引；文件不存在时返回空目录与 `updatedAt = 0`，不触发网络 |
| 手动刷新 | `syncAvailablePlugins` | 同步全部源 → 重建索引 → 原子覆盖 → 直接返回新结果 |
| 详情页文档 | `readPluginReadme` | 读取 id 所属源（namespace 段决定，见来源派生 Namespace 决策 D6）条目目录下的 `README.md`（≤256 KiB、必须是 UTF-8）；条目存在但无文档返回 `null`，id 不存在返回 `PluginNotFound` |
| 安装 / 更新 | `installPlugin` / `updatePlugin` | 重新解析 id 所属源的 `orax.toml` → 选择 release → 按该源的代理策略下载校验并解包到 `installed/` |

主机兼容性不写入缓存文件，而是在每次把条目投影成 `AvailablePlugin` 时由 `release_targets` 与
当前宿主 triple 计算，序列化为封闭枚举 `{ compatibility: "compatible" }` /
`{ compatibility: "incompatible", reason }`，使得一个条目不可能既“兼容”又携带理由，或“不兼容”
却没有理由。

## 不变量

1. 源检出是事实来源，索引是可随时重建的派生缓存；删除索引只损失一次刷新，不损失任何信息。
2. 索引写入必须原子：并发读者要么看到旧索引，要么看到完整的新索引，不存在半个文件。
3. 单个条目的任何问题（清单非法、图标不安全、README 过大）都不能影响其他条目的可见性。
4. 身份只由 `namespace` + `identifier` 决定；target 只选择物理资产，不参与身份或版本语义。
5. 任何下载都必须先通过 SHA-256 校验才进入 `installed/`，target 形态还需与包内自述 target 一致。
6. 宿主兼容性必须在下载之前判定，UI 不能提供一个必然失败的安装入口。
