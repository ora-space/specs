# UI 插件与 WebView Surface

## 定位

UI 插件向 Ora 贡献 **Surface**：一块由 Ora 以原生 WebView 承载并显示在桌面端中的界面。Surface 的内容来自外部站点（`remote_site`）或插件安装包内的页面（`panel`）。Ora 负责创建、放置、隔离、关闭 WebView 并转发其中发生的事件；插件负责决定这些事件的业务含义，并在需要时与自己的页面交换数据。

UI 插件的 `main.js` 与其他插件类型一样运行在 Deno 中，只承担协议适配与流程编排。它不创建窗口，不读写文件系统，不发起网络请求；这些动作全部由 Ora 宿主在 SDK 请求下代为完成。WebView 本身属于宿主资源，插件通过稳定的 Surface 身份间接引用它。

## Surface 定义与实例

关系模型如下：

```text
UiPlugin
└── SurfaceDefinition (surface_id, source, placement policy)
    ├── SurfaceInstance (surface_instance_id, webview)
    └── SurfaceInstance …
```

- **SurfaceDefinition** 由安装包清单声明，随版本发布且不可变。
- **SurfaceInstance** 是一次打开得到的运行期对象，由 Ora 分配 `surface_instance_id`。同一定义可以先后或同时产生多个实例，数量受定义中的实例策略约束（默认单例）。
- `surface_instance_id` 是 Ora 分配的稳定运行期身份，不由插件生成，也不应替换为 WebView 标签、窗口句柄或页面内的任何自述身份。插件可以在内部维护实例与自身状态的映射。

必须满足以下不变量：

1. 插件只能通过 `surface_instance_id` 引用 Surface，不会获得 WebView 句柄、窗口句柄或进程内对象。
2. 一个实例在其生命周期内属于且仅属于一个插件版本的一个定义；插件升级后新打开的实例使用新版本定义，已打开的实例随旧进程关闭。
3. Ora 在把任何页面事件转发给插件之前，先把发生事件的 WebView 解析到实例记录；页面自述的身份一律不作为依据。
4. 插件进程失败或停止不会使 Surface 失败；远程站点可以在没有插件进程的情况下继续显示，只有依赖插件的功能（下载投递、Panel 请求）降级。

## 清单声明

UI 插件在 `orax.toml` 中声明 `kind = "ui"`，并在 `[[ui.surfaces]]` 中列出 Surface 定义：

```toml
resolver = 1
name = "ora-space.skillhub"
namespace = "official"
kind = "ui"
version = "0.3.0"
description = "SkillHub marketplace"
homepage = "https://github.com/ora-space/ui-plugins"
license = "Apache-2.0"

[[ui.surfaces]]
id = "market"
title = "SkillHub"
instances = "singleton"        # "singleton" | "multiple"

[ui.surfaces.source]
kind = "remote_site"
entry = "https://skillhub.example/"
hosts = ["skillhub.example"]
host_suffixes = ["cdn.skillhub.example"]

[ui.surfaces.web_data]
mode = "persistent"            # "persistent" | "ephemeral"
```

Panel 形式的定义：

```toml
[[ui.surfaces]]
id = "counter"
title = "Counter"

[ui.surfaces.source]
kind = "panel"
root = "assets/panel"          # 安装目录内的子目录
entry = "index.html"           # root 下存在的 .html 文件
```

### 字段

#### `ui.surfaces[].id` / `title`

`id` 是包内唯一、跨版本尽量稳定的 Surface 标识，由小写字母、数字和连字符组成。`title` 是用户可见名称。

#### `ui.surfaces[].instances`

实例策略。`singleton` 表示同一时刻最多一个实例，再次打开返回已有实例并将其置前；`multiple` 表示每次打开都创建新实例。缺省为 `singleton`。

#### `ui.surfaces[].source`

内容来源，两种形态互斥：

| `kind` | 必填字段 | 含义 |
| --- | --- | --- |
| `remote_site` | `entry`, `hosts` 与/或 `host_suffixes` | 外部站点。`entry` 必须是 `https` URL，不含凭据和显式端口；`hosts` 列出精确主机名，`host_suffixes` 列出允许其全部子域的主机名；两者至少一项非空 |
| `panel` | `root`, `entry` | 安装包内页面。`root` 必须是安装目录内的子目录，`entry` 必须是 `root` 下存在的 `.html` 文件 |

#### `ui.surfaces[].web_data`

WebView 的站点数据（Cookie、LocalStorage、缓存）保留方式。`persistent` 为每个 `(插件, Surface)` 维护独立的持久站点数据目录；`ephemeral` 每次打开使用一次性数据。Panel 始终使用按 Surface 隔离的持久站点数据。缺省为 `persistent`。

### 解析规则

1. 未知字段、未知枚举值、非法 URL、空允许列表和越出安装目录的 `root`/`entry` 均使插件安装失败，不能静默忽略。
2. 同一插件内 `id` 不得重复。
3. `source.kind` 决定哪些字段必填；不属于该形态的字段出现即为错误。
4. 清单中的 URL 与主机名在安装时规范化（小写主机、去除默认端口）；运行时的导航判断以规范化结果为准。

## 导航边界

Ora 对每个 Surface WebView 强制执行由清单推导出的导航边界：

- `remote_site` 只能导航到 `https` 且主机命中 `hosts` 或是 `host_suffixes` 中某项的子域的 URL；带凭据、带显式端口、非 `https` 的 URL 一律拒绝。后缀匹配按域名标签分界，`example.com.evil.test` 不匹配 `example.com`。
- `panel` 只能导航到其自身资产基址之下的 URL。
- 页面发起的 `window.open` 不会得到 Ora 托管的 WebView：命中允许列表的 URL 交由系统浏览器打开，其余拒绝。Panel 不打开任何弹窗。
- 被拒绝的导航与弹窗由 Ora 记录日志，不通知页面，也不通知插件。

导航边界是清单的函数，插件不能在运行期修改它。

## 存储布局

UI 插件遵循通用的代码与数据分离布局：

```text
~/.ora/plugins/
├── installed/<namespace>/<plugin_name>/<version>/
│   ├── main.js
│   └── assets/panel/…            # panel 资产，只读
└── data/<namespace>/<plugin_name>/
    ├── downloads/                # Ora 写入的 Surface 下载结果
    └── web-profile/<surface_id>/ # 持久站点数据（Windows / Linux）
```

- Panel 资产由 Ora 从不可变安装目录读取并提供给 WebView；插件不能在运行期改写或替换它们。
- `downloads/` 与 `web-profile/` 由 Ora 管理；插件通过 Storage SDK 以逻辑路径读取自己的 `downloads/`，不接触 `web-profile/`。
- 插件数据目录的位置由 Ora 根据插件身份解析，不通过环境变量或启动参数告知插件。

## Panel 资产服务

Panel 页面通过 Ora 注册的自定义协议加载：

```text
ora-plugin://localhost/<plugin_id>/<surface_id>/<path>
```

Ora 处理每个请求时：

1. 把发起请求的 WebView 解析到实例记录，确认它是该插件该 Surface 的活跃 Panel 实例；
2. 要求 URL 中的插件与 Surface 段与实例记录一致；
3. 把 `<path>` 解析到清单 `root` 的规范化目录内，拒绝任何越界、非普通文件或不在内容类型白名单内的路径；
4. 为 HTML 文档附加严格的内容安全策略：禁止内联脚本与样式、禁止嵌套框架、只允许连接回 Ora 宿主。

任何拒绝都返回 404，原因记入日志。HTML 文档不缓存，其他资产每次加载重新验证，保证插件升级后重新打开的 Panel 加载新版本资产。

## 宿主调用插件的方法

UI 插件按需实现以下方法。Ora 在握手时读取插件注册的方法集合；声明了 `remote_site` Surface 的插件必须实现 `ora/ui/download_completed`，声明了 `panel` Surface 的插件必须实现 `ora/ui/request`。

### `ora/ui/surface_opened`

通知：一个实例已进入打开状态。

```text
surface_id
surface_instance_id
plugin_generation
```

- `plugin_generation` 是当前插件进程的代际编号，由 Ora 在每次启动插件进程时递增分配；
- 插件进程启动时，Ora 为该插件每个已打开的实例各发送一次本通知，使插件总能重建"当前有哪些实例"的视图；
- 同一 `(plugin_generation, surface_instance_id)` 只通知一次，插件不需要自行去重。

### `ora/ui/surface_closed`

通知：一个实例已关闭。参数与 `surface_opened` 相同。实例关闭后，该 `surface_instance_id` 不再出现在任何后续通知或请求中。

### `ora/ui/download_completed`

通知：某个 `remote_site` 实例中的下载已完整写入插件数据目录。

```text
surface_id
surface_instance_id
plugin_generation
download:
  id
  page_url
  source_url
  file_name
  path            # 相对插件数据目录的逻辑路径，如 downloads/<file_name>
  size_bytes
  completed_at    # 本地时间 RFC 3339
```

- 下载目的地由 Ora 根据实例归属决定，页面不能指定；
- Ora 先写入临时文件、完成后原子提升为最终文件名，重名自动去重，不覆盖已有文件；
- 通知失败不重试，文件保留在磁盘上；插件可在下次启动时通过 Storage SDK 扫描 `downloads/` 补处理；
- 下载的后续语义（导入、解压、索引）完全由插件决定，Ora 不再内置任何自动导入行为。

### `ora/ui/request`

请求：Panel 页面向插件发出的一次业务调用。

```text
surface_id
surface_instance_id
plugin_generation
payload         # 页面提供的 JSON 值
```

返回 `{ payload }`，其中 `payload` 原样交回页面。请求与响应各有大小上限（1 MiB）。页面在请求时不声明自己的身份；`surface_id` 与 `surface_instance_id` 由 Ora 根据发起请求的 WebView 填入。插件返回的错误会被 Ora 归类为插件错误转交页面，宿主侧失败（进程不可用、超时、负载过大）由 Ora 直接回复页面。

## 插件调用宿主的方法

### `ora/ui/push`

通知：插件向某个 Panel 实例推送一条消息。

```text
surface_id
surface_instance_id
plugin_generation
payload
```

Ora 按 `(surface_id, surface_instance_id)` 定位实例，确认它属于调用插件、是 Panel 且 `plugin_generation` 与当前进程一致后，将 `payload` 投递到页面。不满足任一条件的推送被丢弃并记录日志。Ora 为每个实例维护递增的投递序号，页面可据此检测乱序。

### Storage SDK

插件通过通用 Storage SDK 读取自己的 `downloads/` 与其他私有数据，接口使用相对插件数据目录的逻辑路径。UI 插件没有专属的文件访问方法。

## 页面侧接口

Ora 向每个 Panel 页面注入 `acquireOraSurfaceApi()`，返回：

```text
request(payload) -> Promise<payload>     # 经 Ora 转交 ora/ui/request
onPush(listener) -> unsubscribe          # 接收 ora/ui/push 投递
```

页面不需要知道 Tauri、插件进程或实例 id 的存在；`remote_site` 页面不注入任何接口。

## 生命周期

```text
Opening ──成功──▶ Open ──▶ Closing ──▶ Closed
   │                │
   └──失败──▶ Failed ──重试──▶ Opening
                Open ◀──迁移──▶ Migrating   (embedded ⇄ windowed)
```

- 打开由用户在 Ora 界面触发；插件不能主动打开或关闭 Surface。
- 插件进程按需启动：首个实例打开或首个 Panel 请求到达时由 Ora 拉起；最后一个实例关闭后经固定宽限期停止。
- 停止、禁用或卸载插件时，Ora 先关闭该插件的全部实例，再停止进程。
- 主窗口销毁时关闭全部实例。
- 失败的实例保留 `surface_instance_id`，重试时在同一 id 上重建 WebView。

## 放置方式

每个实例以两种方式之一放置：

| 方式 | 含义 |
| --- | --- |
| `embedded` | 作为主窗口的子 WebView，覆盖在 Ora 界面预留的面板区域上 |
| `windowed` | 独立的原生窗口 |

是否支持 `embedded` 由 Ora 的构建与运行环境决定，前端在打开前查询；不支持时自动降级为 `windowed`。实例可以在两种方式之间迁移。放置方式对插件透明，`surface_opened` 等通知不携带它。

## 实现边界

Ora 负责：

- 解析并校验清单中的 Surface 定义；
- 分配 `surface_instance_id` 与 `plugin_generation`；
- 创建、放置、迁移、隐藏与关闭 WebView；
- 强制执行导航边界与站点数据隔离；
- 提供 Panel 资产并附加内容安全策略；
- 把页面事件解析到实例记录后转交插件；
- 写入下载文件并通知插件；
- 转接 Panel 请求与推送；
- 按需启动与空闲停止插件进程，并在停止前关闭其实例。

UI 插件负责：

- 在清单中声明 Surface 定义；
- 实现 `ora/ui/*` 方法，维护实例与自身状态的映射；
- 决定下载结果的业务处理；
- 实现 Panel 页面与 `ora/ui/request` / `ora/ui/push` 之间的应用协议；
- 通过 Storage SDK 管理自己的数据目录内容。
