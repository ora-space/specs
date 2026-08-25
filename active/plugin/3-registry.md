# 插件分发

插件列表从中心仓拉取更新到 `~/.ora/plugins/sources/github.com/ora-space/marketplace`，类似 homebrew。插件信息放在 `registry/` 下面。

为了避免单个文件夹内文件过多，仓库内按首字母进行二级子目录划分（如 registry/a/, registry/b/）。

插件示例（`registry/o/ora-space.weather/orax.toml`）：

```toml
resolver = 1
# 1. 基础元数据 (Metadata)
name = "user.ora-weather"
namespace = "official"
kind = "workbench"
version = "1.2.0"
description = "获取实时天气信息的 Ora 插件"
homepage = "https://github.com/user/ora-weather"
license = "MIT"

# 2. 核心发布包配置（配置 url 表示该 .orax 全平台通用）
url = "https://github.com/user/ora-weather/releases/download/v1.2.0/ora-weather-v1.2.0.orax"
sha256 = "feab001d7e9ff4ce66011ebd70791de93eb1554d34d3ea44c33d102a25c1be0a"

# 3. 开发版源码地址（可选，支持 --head 安装）
[head]
repository = "https://github.com/user/ora-weather.git"
branch = "main"

# 4. 依赖控制 (Environment & API Dependencies)
[dependencies]
ora = ">= 0.8.0"          # 限定最小支持的 Ora 版本
```

## 发布资产与 Target

一个插件版本必须选择以下两种发布方式之一。

### 全平台通用资产

如果 `.orax` 在所有受支持平台上内容相同，直接在顶层配置 `url` 和 `sha256`：

```toml
url = "https://github.com/user/plugin/releases/download/v1.2.0/plugin-v1.2.0.orax"
sha256 = "feab001d7e9ff4ce66011ebd70791de93eb1554d34d3ea44c33d102a25c1be0a"
```

顶层存在 `url` 即表示该资产是全平台通用资产。Ora 在任何平台安装这个版本时都下载同一个 URL，并使用与其配套的 `sha256` 校验内容。

这种方式适用于只包含 JavaScript、配置或其他平台无关资源的插件。`url` 与 `sha256` 必须同时出现。

### Target 专用资产

如果插件包含平台相关的二进制或原生资源，不配置顶层 `url` 和 `sha256`，改用一个或多个 `[[targets]]` 声明每个编译目标的发布资产：

```toml
[[targets]]
target = "aarch64-apple-darwin"
url = "https://github.com/user/plugin/releases/download/v1.2.0/plugin-v1.2.0-aarch64-apple-darwin.orax"
sha256 = "1111111111111111111111111111111111111111111111111111111111111111"

[[targets]]
target = "x86_64-apple-darwin"
url = "https://github.com/user/plugin/releases/download/v1.2.0/plugin-v1.2.0-x86_64-apple-darwin.orax"
sha256 = "2222222222222222222222222222222222222222222222222222222222222222"

[[targets]]
target = "x86_64-pc-windows-msvc"
url = "https://github.com/user/plugin/releases/download/v1.2.0/plugin-v1.2.0-x86_64-pc-windows-msvc.orax"
sha256 = "3333333333333333333333333333333333333333333333333333333333333333"

[[targets]]
target = "x86_64-unknown-linux-gnu"
url = "https://github.com/user/plugin/releases/download/v1.2.0/plugin-v1.2.0-x86_64-unknown-linux-gnu.orax"
sha256 = "4444444444444444444444444444444444444444444444444444444444444444"
```

`target` 使用规范化的 Rust target triple，同时表达 CPU 架构、操作系统和 ABI。Ora 根据当前运行环境计算自己的精确 target，并只选择 `target` 完全相等的资产。

### 选择与校验规则

1. 顶层 `url`/`sha256` 与 `[[targets]]` 互斥，不能同时配置。这样不会产生“通用资产还是 target 资产优先”的歧义。
2. 配置顶层 `url` 时，`url` 与 `sha256` 必须同时存在，且该资产被视为全平台通用。
3. 不配置顶层 `url` 时，必须至少声明一个 `[[targets]]`，每项都必须同时包含 `target`、`url` 和 `sha256`。
4. 同一插件版本中 `target` 不得重复。URL 可以相同，但每个 target 仍需显式声明，且摘要必须与实际资产一致。
5. 使用 target 发布时必须精确匹配，不做相近架构、相近 ABI 或其他 target 的隐式回退。例如 `x86_64-unknown-linux-gnu` 不能替代 `x86_64-unknown-linux-musl`。
6. 当前平台没有匹配 target 时，该版本在本机显示为“不支持当前平台”，Ora 不应下载其他平台资产后尝试运行。
7. 市场同步阶段验证字段形状和 target 唯一性；安装阶段只下载选中的资产，并再次验证其 SHA-256 和包内 manifest。
8. 插件的 canonical ID、kind 和 version 在各 target 包中必须一致。target 只决定发布资产，不能形成不同的插件身份或版本语义。

## orax cli

每个插件目录下有 orax.toml, logo.svg, README.md。提交 pr 到 ora-space/marketplace 跑 action，使用 orax cli 进行校验 toml 配置。

插件安装解压过程中：
- 用 Rust 的官方 `semver` crate 来解析和校验依赖版本
- README 校验：禁止嵌入未过滤的恶意 HTML 标签。
- SVG 校验：解析 logo.svg XML 树，禁用 `<script>` 标签、`<foreignObject>`、外部网络资源请求（如 `<image href="http://...">`），且文件限制在 50KB 以内。
- 路径安全：确保 Zip 压缩包解压路径无 ../ 路径穿越漏洞（Zip Slip 漏洞）。

- **签名与验签 (Verification & Integrity)：** 校验 `.orax` 文件的哈希（SHA-256）及数字签名（GPG / Ed25519），防止篡改或中间人攻击。

依赖与兼容性 (Dependency & Compatibility)：

- **版本兼容校验 (Version Compatibility)：** 校验宿主（Ora 版本）与插件要求的 API 版本边界（如 `semver` 匹配）。
- **依赖树拓扑求解 (Dependency Resolution)：** 插件依赖其他插件时的自动依赖下载、冲突检测（Conflict Resolution）。
- **平台适配 (Platform Matrix)：** 针对 OS（Windows / macOS / Linux）与 CPU 架构（x86_64 / aarch64）进行二进制或原生依赖的打标与拦截。
- **发布资产选择：**顶层 `url` 表示全平台通用；否则按当前 Rust target triple 从 `[[targets]]` 中精确选择。两种形式不能混用。

## ora-space/marketplace PR 自动化分发流水线

```
# .github/workflows/validate-plugin.yml 示例逻辑
name: Validate and Pack Plugin
on: [pull_request]

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
      - name: Install orax cli
        run: npm install -g @ora-space/orax
      - name: Validate TOML & Assets
        run: orax validate --path registry/
      - name: Test Download & Check SHA256
        run: orax verify-release --manifest registry/o/ora-space.weather/orax.toml
```

## 页面展示

在 `~/.ora/plugins/cache/` 目录下生成 `registry_index.json`。

索引产物需兼顾**轻量**与**UI 展现所需的基础元数据**，避免 UI 再次逐个读取 `orax.toml`。

```json
{
  "updated_at": 1776244428,
  "version": "1.0",
  "plugins": [
    {
      "id": "official/user.ora-weather",
      "name": "user.ora-weather",
      "namespace": "official",
      "version": "1.2.0",
      "description": "获取实时天气信息的 Ora 插件",
      "release": { "kind": "universal" }
    }
  ]
}

```

* **`id`**：由 `namespace` + `name` 生成的唯一标识。
* **`release`**：通用资产写 `{ "kind": "universal" }`；target 发布写 `{ "kind": "targets", "targets": ["aarch64-apple-darwin", ...] }`。索引不需要包含下载 URL 和摘要，安装时读取对应的 registry manifest。

构建触发时机放在 Sync 成功之后：

```
[git pull / sync] ──> [扫描 & 解析 registry/ 目录] ──> [生成 index.json.tmp] ──> [原子覆盖 index.json]
```

1. **并发并行解析 (Parallel Parsing)**：
* 在 Rust 中通过 `ignore::WalkBuilder` 或 `walkdir` 配合 `rayon` 多线程并发解析所有的 `orax.toml`。全量扫描数万个小 TOML 文件通常在 **50ms 以内** 完成。

2. **原子覆盖 (Atomic Swap)**：
* 构建时先写入临时文件 `registry_index.json.tmp`，解析校验全部完成后再 `std::fs::rename` 覆盖目标文件，确保 UI 在任何时刻读取到的索引均完整无损。

3. **容错机制 (Error Handling)**：
* 遇到格式错误的 `orax.toml` 时记录 warning 日志并跳过该项，避免单个坏文件阻断整个索引构建。

后续支持：黑名单与撤销 (Revocation / Advisory)： 当某个版本的插件被发现重大安全漏洞时，中央仓推送到客户端强制禁用/下架（Vulnerability Advisory）。
