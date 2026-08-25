# Workbench 插件 Webview

## 定位

Workbench 插件可以向 Ora 提供一块由 HTML、CSS 和 JavaScript 实现的交互界面。页面运行在独立的 Tauri Webview 中，静态资源来自插件的不可变安装目录，业务请求通过一个受限桥接命令转发给同一插件当前运行的 `main.js`。

插件 Webview 是不可信 UI，不是 Ora 主界面的延伸。它不能直接获得主 Webview 的 Tauri command、窗口、对话框或其他原生权限。页面是否显式依赖 `@tauri-apps/api` 不构成安全边界：只要 Tauri IPC 被注入，页面就可能自行构造 invoke 请求，因此所有隔离必须由 Tauri ACL 和 Ora 宿主共同强制。

第一版不支持 iframe 嵌入主界面、页面直连网络、插件主动向页面推送事件或页面直接调用任意 Ora command。

```text
Plugin Webview (untrusted HTML/JS)
        │
        │ window.ora.invoke(method, params)
        ▼
Tauri ACL: plugin_webview_invoke only
        │
        ▼
WorkbenchSurface module
  identify caller Webview
  resolve SurfaceEntry::Ready
  validate lifecycle, generation and method
        │
        ▼
PluginGenerationLease.invoke(method, host-owned envelope)
        │
        ▼
Plugin main.js (Deno, zero permissions)
        │
        │ Ora SDK when a host capability is required
        ▼
Ora host capability authorization
```

这一调用链不会绕过 [0-overview.md](./0-overview.md) 和 [1-capability.md](./1-capability.md) 定义的插件权限模型。Webview 只能调用自己的 `main.js`；`main.js` 若要访问文件、网络、进程或其他宿主资源，仍必须通过 Ora SDK，并接受原有的声明、授权与资源归属校验。

## 包结构与声明

`kind = "workbench"` 的第一版包结构为：

```text
plugin.orax
├── orax.toml
├── README.md
├── logo.svg
├── main.js
└── assets/
    ├── index.html
    ├── app.js
    ├── app.css
    └── ...
```

一个 Workbench 插件贡献一个逻辑 Surface，可以同时创建多个 Surface 实例。入口固定为 `assets/index.html`，避免第一版引入可配置入口和多 Surface 路由。插件可在 `orax.toml` 中声明页面允许调用的 `main.js` 方法：

```toml
resolver = 1
name = "user.ora-weather"
namespace = "official"
kind = "workbench"
version = "1.2.0"
description = "获取实时天气信息的 Ora 插件"

[workbench]
methods = ["weather/get_current", "weather/search_city"]
```

安装时必须严格校验：

1. `assets/index.html` 存在，是当前精确安装版本内的普通文件；
2. `assets/` 下的全部可访问资源均不能通过绝对路径、父目录、符号链接、junction 或 reparse point 逃逸安装目录；
3. `methods` 不得重复或为空，必须符合插件协议的方法名语法；
4. Workbench 插件必须包含 `main.js`，并按普通 Deno 插件的零权限模型启动；
5. 非 Workbench kind 不能声明 `[workbench]`；
6. 未知字段和未来 schema 版本一律拒绝，不能静默忽略。

`methods` 是页面可见接口，不是宿主 Capability。它只能收窄页面对本插件 `main.js` 的调用面，不能赋予 `main.js` 新的系统能力。每个方法还必须出现在该进程 generation 的 `ora/register.params.methods` 中；有效方法集合为：

```text
orax.toml 中的 workbench.methods
∩ 当前 main.js generation 注册的 methods
```

任一侧没有声明的方法都不可调用。这样静态安装契约与当前运行实现必须同时同意，页面既不能触达插件内部的其他方法，也不能调用旧版本曾经存在但当前 generation 未注册的方法。

## 领域模型

```text
InstalledWorkbenchDescriptor
├── plugin_id
├── exact_version
├── entrypoint
└── declared_methods

SurfaceInstance
├── SurfaceIdentity
└── SurfaceEntry
    ├── Creating { identity }
    ├── Ready { identity, target, effective_methods }
    │                    │
    │                    └── PluginGenerationLease
    │                                └── PluginProcessGeneration
    └── Closing { identity }
```

必须满足以下不变量：

1. `surface_instance_id` 由 Ora 分配，在当前应用进程内唯一；页面和插件都不能选择该 ID。
2. `SurfaceIdentity` 创建后不可变，并通过一个 `InstalledWorkbenchDescriptor` 精确绑定插件 ID、安装版本和资源入口，通过 `PluginGenerationKey` 绑定创建它的 process generation。
3. 只有 `Ready` entry 持有 `PluginGenerationLease`；该 lease 的 key 必须等于 `SurfaceIdentity.generation_key`，并且不能被重定向到后续 generation。
4. Surface 实例不能在插件升级或进程重启后悄悄迁移到新 generation；旧实例必须关闭，由宿主显式创建新实例。
5. 一个 Webview label 在任一时刻最多对应一个 Surface 实例，绑定建立后不可修改。
6. 生命周期变化通过原子替换 `SurfaceEntry` variant 表达，不能用一个可变 enum 字段配合若干在部分状态下无效的字段表达。
7. 关闭开始后不再接受新调用；Surface 进入 `Closing` 必须早于 Webview 销毁。
8. 插件停用、卸载、版本替换或进程 generation 结束时，宿主必须关闭属于它的全部 Surface 实例。

## WorkbenchSurface 模块

WorkbenchSurface 是主界面、Tauri Webview 和插件运行时之间的深模块。外部 interface 保持为三项操作：

```text
open(plugin_id) -> surface_instance_id
invoke(caller_webview, method, params) -> JSON value
close(surface_instance_id) -> ()
```

资源定位、Webview 创建、ACL 标签约定、调用方识别、方法白名单、generation 校验、超时、错误映射和清理都隐藏在该 interface 后。调用者不能分别操作 `SurfaceRegistry` 与 Webview，否则二者可能产生不一致状态。

模块内部维护带辅助索引的 `SurfaceRegistry`：

```text
SurfaceRegistry {
  by_webview: Map<WebviewLabel, SurfaceEntry>,
  by_instance: Map<SurfaceInstanceId, WebviewLabel>,
  by_generation: Map<PluginGenerationKey, Set<WebviewLabel>>
}

SurfaceEntry
├── Creating {
│     identity: SurfaceIdentity
│   }
├── Ready {
│     identity: SurfaceIdentity,
│     target: PluginGenerationLease,
│     effective_methods: MethodSet
│   }
└── Closing {
      identity: SurfaceIdentity
    }

SurfaceIdentity {
  surface_instance_id,
  descriptor: InstalledWorkbenchDescriptor,
  generation_key: PluginGenerationKey
}

InstalledWorkbenchDescriptor {
  plugin_id,
  exact_version,
  entrypoint,
  declared_methods
}
```

`by_webview` 是 IPC 鉴权和资源请求的主索引；`by_instance` 支持 `close(surface_instance_id)`；`by_generation` 支持插件进程结束时只清理属于该 generation 的 Surface。三个索引必须在同一临界区内更新，不能出现只更新其中一部分的可观察状态。

`PluginGenerationLease` 是插件生命周期模块提供给 WorkbenchSurface 的窄 interface。它封装 generation identity、当前 registration 和 `invoke`，但不允许调用者取得或替换裸 `PluginRuntime`。lease 不拥有进程的重启策略，也不能让已经结束的 generation 复活；generation 结束后，任何尚未开始的调用都返回 `PluginUnavailable`。`PluginGenerationKey` 只是不可调用的稳定身份，可安全保留在所有状态中用于索引和清理。

进入 `Ready` 时一次性计算不可变的 `effective_methods`：

```text
descriptor.declared_methods
∩ generation_lease.registered_methods
```

状态迁移通过替换整个 `SurfaceEntry` variant 完成。`Ready` 中不存在缺少 target 或方法集合的状态，`Creating` 和 `Closing` 中也不存在可被误用的调用 target。

`webview_label` 使用宿主生成的不透明值，例如：

```text
plugin-webview:<surface_instance_id>
```

label 只用于 Tauri ACL 匹配和注册表查找。宿主不得从 label 文本解析并信任插件 ID、版本或 generation；这些身份只能来自 Registry 中的 `SurfaceIdentity` 和 `PluginGenerationLease`。

## Tauri ACL

Ora 必须把可信主 Webview 与插件 Webview 分配到互不重叠的 capability。

主界面 capability 必须用 Webview label 精确匹配：

```json
{
  "identifier": "default",
  "webviews": ["main"],
  "permissions": [
    "allow-main-commands",
    "core:default",
    "core:window:allow-minimize",
    "core:window:allow-maximize",
    "core:window:allow-unmaximize",
    "core:window:allow-toggle-maximize",
    "core:window:allow-is-maximized",
    "core:window:allow-start-dragging",
    "core:window:allow-close",
    "dialog:allow-open",
    "dialog:allow-save"
  ]
}
```

不能改回 `"windows": ["main"]`。一个 Tauri window 可以容纳多个 Webview，按 window 授权会使其中的插件子 Webview 一并匹配主界面 capability。

插件 capability 只允许桥接命令：

```json
{
  "identifier": "plugin-webviews",
  "webviews": ["plugin-webview:*"],
  "permissions": ["allow-plugin-webview-invoke"]
}
```

`allow-plugin-webview-invoke` 的 command allowlist 只能包含：

```text
plugin_webview_invoke
```

它不能包含 `allow-main-commands`、`core:default`、dialog、window、shell、filesystem、HTTP 或其他 Tauri/plugin 权限。以后新增插件页面能力时，也应优先通过 Ora 的强类型桥接扩展，而不是把通用 Tauri 权限直接授予插件 Webview。

所有 Ora 自定义 Tauri command 必须继续通过同一命令注册表同时进入：

```text
tauri::generate_handler![...]
tauri_build::AppManifest::commands(...)
```

只加入 `invoke_handler`、但没有加入 `AppManifest::commands` 的自定义命令不受应用 ACL 正确保护，禁止使用这种注册方式。构建或测试应验证 handler registry、AppManifest registry 和 permission allowlist 没有漂移。

## 页面 SDK

页面只使用 Ora 注入的最小接口：

```ts
type OraWebview = {
  invoke<T = unknown>(method: string, params?: unknown): Promise<T>;
};

declare global {
  interface Window {
    ora: OraWebview;
  }
}
```

示例：

```js
const weather = await window.ora.invoke("weather/get_current", {
  city: "Shanghai",
});
```

SDK 负责稳定的调用形状、类型声明和错误规范化，但不负责鉴权。即使页面绕过 `window.ora`，直接访问 Tauri 注入对象或自行实现 invoke，仍只能通过 Tauri ACL 调用 `plugin_webview_invoke`。

第一版请求字段只包含：

```json
{
  "method": "weather/get_current",
  "params": { "city": "Shanghai" }
}
```

请求不得接受 `plugin_id`、`exact_version`、`surface_instance_id`、`webview_label` 或 `plugin_generation`。这些字段全部由宿主根据命令的调用方 Webview 查找；允许页面提交这些身份会形成 confused deputy 风险。

## 调用处理

`plugin_webview_invoke` 必须把 Tauri 注入的调用方 `Webview` 作为身份来源，并按以下顺序处理：

1. 读取调用方 Webview label，通过 `by_webview` 查找 entry；
2. 只有 `SurfaceEntry::Ready` 可以继续，并在同一 Registry 临界区内取得其不可变调用快照；`Creating`、`Closing` 或不存在均返回 `SurfaceUnavailable`；
3. 校验 `method` 和 `params` 的形状与大小；
4. 确认 `method` 位于调用快照的 `effective_methods`；
5. 由宿主构造调用上下文，通过快照中的 `PluginGenerationLease` 调用精确 generation；
6. 对结果执行大小限制与稳定错误映射，再返回页面。

取得 `Ready` 调用快照是 invoke 的线性化点。已经越过该点的调用可以完成；随后发生的 close 不需要取消它。close 进入 `Closing` 后到达的新调用必须失败，不能取得旧 lease，也不能被路由到同插件的新 generation。

宿主发给 `main.js` 的 params 使用宿主拥有的 envelope：

```json
{
  "surface": {
    "instance_id": "<host allocated>",
    "generation": 7
  },
  "input": {
    "city": "Shanghai"
  }
}
```

`plugin_id` 和精确版本已经由 descriptor 与 generation lease 确定，不需要作为可路由字段再次交给插件。`surface.instance_id` 只帮助同一 `main.js` 区分自己的多个页面实例，不能用于访问其他插件或扩大权限。

桥接调用沿用插件运行时的 30 秒 call timeout。请求和响应必须各自设置低于插件协议 16 MiB frame 上限的独立 Webview 限额；第一版统一限制为 1 MiB 的 UTF-8 JSON。超限请求在进入插件运行时前拒绝，超限响应不返回页面。

同一 Surface 可以并发调用；请求关联由 Tauri invoke 和 lease 背后的插件运行时分别管理。页面销毁只取消页面一侧的等待，不应把迟到但协议合法的插件响应误判为 process generation 故障。

## 生命周期

创建顺序：

```text
trusted main Webview requests open(plugin_id)
        ↓
resolve enabled InstalledWorkbenchDescriptor
        ↓
ensure main.js generation is Ready and acquire PluginGenerationLease
        ↓
allocate surface_instance_id and Webview label
        ↓
atomically insert Creating entry and auxiliary indexes
        ↓
create child Webview with plugin resource URL
        ↓
compute effective_methods from the acquired lease
        ↓
atomically replace Creating with Ready
```

如果 generation lease 获取、Webview 创建或首个页面加载失败，宿主必须销毁可能已创建的 Webview，并在同一临界区删除 `Creating` entry 及其全部辅助索引，不能留下半注册实例。页面在 entry 进入 `Ready` 前发起的调用返回 `SurfaceUnavailable`。

关闭顺序：

```text
atomically replace Creating or Ready with Closing
        ↓
reject new invokes
        ↓
destroy Webview
        ↓
atomically remove entry and all auxiliary indexes
```

Webview 自行关闭、主界面关闭 Surface、插件停用、插件崩溃和应用退出都必须汇聚到同一幂等关闭路径。重复 close 不影响其他 Surface。插件 generation 结束时，宿主通过 `by_generation` 找到相关实例并逐一进入同一关闭路径，不能扫描并重新解释 Webview label。

插件进程重启时不复用旧 Surface，因为其页面状态和旧 generation 的内部状态可能已经分叉。宿主关闭旧实例后可向主界面报告插件已重启，由主界面决定是否重新打开新实例。

## 资源加载与浏览器约束

插件页面只能从当前 `SurfaceIdentity` 的 descriptor 绑定的精确安装版本加载资源。资源请求仅在 entry 为 `Creating` 或 `Ready` 时有效；`Closing` 和不存在的 entry 必须拒绝。宿主资源 adapter 必须：

1. 使用 Ora 自有的本地 scheme 为每个 Surface 提供资源，不暴露任意 `file://` 路径；
2. 对每次资源请求从 `SurfaceIdentity.descriptor` 解析包根目录，不信任 URL 中的插件 ID 或磁盘路径；
3. 对 URL 解码后再执行规范化与目录 containment 校验，拒绝双重编码、父目录、绝对路径和符号链接逃逸；
4. 只返回普通文件，并根据受控扩展名表设置 MIME type；未知类型使用 `application/octet-stream`；
5. 对 HTML 使用 `no-store`，避免插件升级或 Surface 重建后复用旧入口；带内容摘要的静态资源可以使用 immutable cache；
6. 插件停用、卸载或 Surface 关闭后立即停止为该绑定提供资源。

宿主为插件页面强制生成 Content Security Policy，插件不能放宽它。基线要求是：

```text
default-src 'none'
script-src 'self'
style-src 'self'
img-src 'self' data:
font-src 'self'
connect-src <only Tauri internal IPC endpoints required by this build>
object-src 'none'
base-uri 'none'
form-action 'none'
frame-src 'none'
frame-ancestors 'none'
```

不得启用远程脚本、远程样式、远程图片、任意 `connect-src`、`unsafe-eval` 或插件自定义 CSP。内联脚本默认禁止；如果构建工具确实需要内联 bootstrap，必须由安装阶段计算 hash 并由宿主加入精确 hash，不能使用 `unsafe-inline`。

此外，宿主必须在 Webview 层执行：

- 拒绝主 frame 导航到插件资源 origin 之外；
- 拒绝新窗口、popup 和未经 Ora SDK 处理的外部 URL；
- 拒绝下载；
- 禁止摄像头、麦克风、定位、通知、剪贴板等浏览器权限请求；
- 不把 Ora 主界面的 cookie、localStorage 或 sessionStorage origin 与插件共用；
- 不启用开发者工具，除非 Ora 明确处于开发者模式，并向用户标明当前页面不可信。

CSP、导航限制和浏览器权限控制是纵深防御，不能代替 Tauri ACL。即使其中一层配置错误，插件页面仍然不应获得 `plugin_webview_invoke` 以外的原生命令。

## 错误语义

页面 SDK 至少规范化以下错误码：

| 错误码 | 含义 |
| --- | --- |
| `InvalidRequest` | method、params 或 JSON 大小不合法 |
| `SurfaceUnavailable` | 调用方没有绑定、尚未 Ready 或正在关闭 |
| `StaleSurface` | Surface 绑定的插件版本或 generation 已失效 |
| `MethodNotAllowed` | 方法未在 Workbench 静态 allowlist 中 |
| `MethodUnavailable` | 方法未由当前 `main.js` generation 注册 |
| `PluginUnavailable` | 插件进程不存在、已退出或不能接收调用 |
| `PluginCallTimedOut` | 插件方法超过运行时调用期限 |
| `PayloadTooLarge` | 请求或响应超过 Webview 桥接限制 |
| `PluginMethodFailed` | `main.js` 返回了可恢复的远端方法错误 |

错误可以包含供开发者诊断的非敏感 message，但不能把宿主绝对路径、其他插件身份、原始 ACL 配置、内部堆栈或 Secret 返回页面。Tauri 在 ACL 层拒绝的非桥接 command 不保证使用上述错误形状，因为请求尚未进入 Ora 桥接模块。

## 明确拒绝的调用

以下页面代码都必须失败：

```js
// 主界面 command：由 Tauri ACL 拒绝。
invoke("create_project", { /* ... */ });

// Tauri plugin command：页面没有 dialog capability。
invoke("plugin:dialog|open", { /* ... */ });

// 伪造其他插件身份：桥接请求不接受 pluginId。
invoke("plugin_webview_invoke", {
  pluginId: "official/another-plugin",
  method: "weather/get_current",
  params: {},
});

// 调用本插件未对页面公开的方法：由 Surface allowlist 拒绝。
window.ora.invoke("agent/start", {});
```

## 验证要求

实现至少覆盖以下测试：

1. 主 Webview 可以调用 `allow-main-commands`，插件 Webview 不能；
2. 插件 Webview 只能调用 `plugin_webview_invoke`，不能调用 core、window、dialog 或其他 plugin command；
3. 命令参数无法伪造插件 ID、Surface ID、版本或 generation；
4. 两个插件和同一插件的两个 Surface 并发存在时，请求只到达各自绑定的 generation lease；
5. 静态 allowlist 或 runtime registration 任一缺失都会拒绝方法；
6. 旧 generation 的 Surface 在插件重启后拒绝调用且被关闭；
7. close 与 invoke 并发时遵守调用快照的线性化语义：已接受调用可以完成，进入 `Closing` 后的新调用被拒绝；
8. 路径穿越、编码穿越、符号链接逃逸、远程资源、导航、popup、下载和浏览器权限请求均被拒绝；
9. 请求超限、响应超限、timeout、插件退出和远端方法错误映射为稳定错误；
10. 自定义 command 注册表与 Tauri AppManifest、main permission、plugin permission 保持一致，新增 command 不会因遗漏 ACL 注册而默认暴露。

安全验收不能只通过 SDK 单元测试完成，必须至少包含一组真实 Tauri runtime 集成测试或打包应用 smoke test。测试页面应绕过 `window.ora` 直接构造 invoke，以证明安全性来自 ACL 和宿主身份绑定，而不是客户端封装。

## 第一版不支持

- 页面直接使用 `fetch`、WebSocket 或其他网络连接；
- 页面直接读取 Workspace、插件 data 目录或任意本地文件；
- 页面直接调用主界面 command 或通用 Tauri plugin；
- `main.js` 主动向页面发送事件或双向流；
- 一个插件声明多个逻辑 Surface；
- Surface 在插件进程重启后保活或自动迁移；
- 插件自定义 CSP、浏览器权限或 Tauri capability。

这些能力如需增加，必须扩展 WorkbenchSurface 的小 interface 和强类型协议，并继续以调用方 Webview、精确插件版本、process generation 与资源归属为鉴权依据，不能通过扩大插件 Webview 的通用 Tauri 权限实现。
