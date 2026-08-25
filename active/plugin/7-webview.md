# Webview 插件

## 定位

`kind = "webview"` 的插件向 Ora 声明一个由外部 HTTPS 站点提供的浏览界面。Ora 在
独立的原生 WebView 中打开该站点，维护浏览器登录状态，限制页面可以导航到的
origin，并接管页面触发的下载。

Webview 插件是纯配置插件。第一版不包含 `main.js`，不启动 Deno 进程，也不调用 Ora
SDK。远程页面是完全不可信的第三方内容，不是 Ora 主界面的延伸，也不是
`kind = "workbench"` 的远程变体：

| kind | 页面来源 | 网络 | 与 Ora 的交互 |
| --- | --- | --- | --- |
| `workbench` | 插件不可变安装目录中的 HTML、CSS 和 JavaScript | 默认禁止 | 通过受限桥接调用同一插件的 `main.js` |
| `webview` | manifest 声明的外部 HTTPS 站点 | 页面按普通浏览器语义联网 | 页面不能调用 Ora；宿主只观察导航和下载生命周期 |

Webview 插件的主要能力是把浏览器下载安全落地，然后根据触发下载时的主页面 URL
选择一个 Ora 内置处理动作，或由可信主界面提示用户从声明的动作集合中选择。URL
规则只负责路由，不能证明下载内容可信，也不能绕过处理动作自身的格式校验、权限
审查或用户确认。

```text
External page (untrusted, browser-managed network and login state)
        │
        │ browser navigation / download event
        ▼
ExternalWebview module
  identify caller WebView
  enforce navigation policy
  snapshot initiating main-frame URL
  select declared download disposition
        │
        ▼
DownloadTarget module
  reserve host-owned staging path
  validate and atomically complete the file
        │
        ├── Auto   ────► host-owned download action
        ├── Prompt ────► trusted main Webview ────► selected action
        └── Reject ────► browser download denied
```

## 包结构与声明

第一版包结构为：

```text
plugin.orax
├── orax.toml
├── README.md
└── logo.svg
```

Webview 包不能包含可运行的 `main.js`，也不能声明供 Deno 插件进程使用的
Capability。浏览器 profile 和下载文件属于 Ora 管理的运行数据，不属于不可变安装
制品。

完整示例：

```toml
resolver = 1
name = "skillhub"
namespace = "official"
kind = "webview"
version = "1.0.0"
description = "Browse and import skills from SkillHub"
homepage = "https://www.skillhub.cn"
license = "MIT"

[webview]
start_url = "https://www.skillhub.cn"
allowed_origins = [
  "https://www.skillhub.cn",
  "https://skillhub.cn",
]

[webview.downloads]
fallback = { prompt = ["save_as"] }

[[webview.downloads.rules]]
page = { origin = "https://www.skillhub.cn", path_prefix = "/skills/" }
action = { auto = "import_skill" }

[[webview.downloads.rules]]
page = { origin = "https://www.skillhub.cn", path_prefix = "/downloads/" }
action = { prompt = ["import_skill", "save_as"] }
```

安装时必须严格校验：

1. `kind = "webview"` 必须且只能携带 `[webview]` 声明；
2. `start_url` 必须是无用户名、密码和 fragment 的绝对 HTTPS URL；
3. `allowed_origins` 必须非空，每项都是只包含 scheme、host 和可选显式端口的 HTTPS
   origin，且必须包含 `start_url` 的 origin；
4. origin 按 URL 标准规范化后不得重复，不支持 wildcard、registrable-domain suffix 或
   任意正则；需要多个 SSO origin 时逐项显式列出；
5. `rules` 保持声明顺序，每条规则必须包含一个合法 page matcher 和一个合法 action；
6. `path_prefix` 必须以 `/` 开始，不得包含 query、fragment、控制字符或无效百分号编码；
7. `auto` 必须引用一个动作，`prompt` 必须引用非空且无重复的动作集合；两者不能同时
   出现；
8. `fallback` 缺省为 `reject`，不能通过省略字段获得更宽松行为；
9. 未知字段、未知动作、非法 URL 和未来 resolver 语义一律拒绝，不能静默忽略。

## 导航与浏览器身份

Ora 为每个插件 ID 创建稳定、独立的浏览器 profile。精确版本升级继续使用同一插件
ID 的 profile，使用户登录状态不会仅因升级丢失；不同插件不能共享 cookie、缓存、
localStorage 或其他浏览器状态。插件停用不删除 profile，卸载时是否删除它必须与删除
插件数据使用同一套显式用户语义。

第一版同一插件最多存在一个 Webview 实例。再次打开已经存在的实例时，Ora 显示、
取消最小化并聚焦原窗口，而不是创建另一个共享 profile 的并发实例。WebView label 和
profile 的磁盘路径由宿主生成，manifest 和远程页面均不能选择或读取。

主 frame 导航只允许进入 `allowed_origins`。URL 必须继续满足 HTTPS、无用户名、无密码
的约束；origin 允许不代表页面路径或下载内容可信。第一版拒绝所有 popup 和新窗口，
也不把被拒绝的 URL 自动交给系统浏览器。以后支持 OAuth popup 或外部浏览器 handoff
时，必须作为独立的用户可见能力设计，不能把任意 `window.open` 直接放行。

## Tauri 隔离

远程 WebView 不能匹配可信主 Webview 或 Workbench 插件 Webview 的 Tauri capability。
宿主不得向其暴露 Ora command、`plugin_webview_invoke`、dialog、window、shell、文件
系统、HTTP 或其他 Tauri/plugin command。页面不能通过自行构造 invoke、猜测 WebView
label 或导航到另一个 origin 获得这些能力。

下载接管发生在 Rust 宿主注册的 WebView 生命周期回调中，不依赖页面 SDK，也不向页面
注入下载路径、`download_id` 或处理结果。远程站点只观察普通浏览器下载是否被接受；
Ora 的后续处理不会扩大页面权限。

摄像头、麦克风、定位、通知、剪贴板和其他浏览器权限请求默认拒绝。开发者工具只在
Ora 明确处于开发者模式时启用，并应向用户标明当前内容来自不可信远程站点。

## 下载规则

### URL 快照

浏览器报告下载请求时，宿主必须从发出事件的受管 WebView 读取当前主 frame URL，并在
同一请求处理中创建不可变的 `DownloadIntent`：

```text
DownloadIntent {
  download_id,
  webview_instance_id,
  plugin_id,
  exact_version,
  initiating_page_url,
  download_url,
  suggested_file_name,
  disposition
}
```

`download_id` 由 Ora 分配且在当前应用进程内唯一。不能使用下载 URL 作为身份，因为
同一 URL 可以并发下载，`blob:` URL 也不具备稳定的跨事件业务身份。

`initiating_page_url` 是下载触发时的主页面 URL 快照，也是规则匹配的唯一页面上下文。
下载结束时不得重新读取当前页面 URL：页面可能已在传输期间跳转。`download_url`、响应
MIME 和建议文件名是不可信元数据，只用于浏览器关联、诊断和文件名候选，不能选择更
高权限的动作。

### Page matcher

第一版 matcher 为：

```text
PageMatcher {
  origin,
  path_prefix
}
```

- `origin` 必须是 `allowed_origins` 中的精确规范化 origin；
- `path_prefix` 匹配 URL 解析器规范化并进行一次合法百分号解码后的 path；
- query 和 fragment 不参与匹配，不能把 token 或其他 Secret 写入规则；
- 规则按 manifest 数组顺序执行，第一个匹配项决定 disposition；
- 安装器应报告完全重复或永远被前序规则遮蔽的规则，避免插件作者误以为后序规则会
  生效。

若没有规则匹配，则使用 `fallback`。`fallback` 省略时拒绝下载。

### Disposition 与动作

第一版领域模型为排他结构：

```text
DownloadDisposition
├── Auto { action: DownloadAction }
├── Prompt { actions: NonEmptySet<DownloadAction> }
└── Reject

DownloadAction
├── ImportSkill
└── SaveAs
```

resolver 1 的 manifest 拼写分别为 `import_skill` 和 `save_as`。动作集合是 Ora 宿主能力
的闭集，不是由插件注册的任意字符串方法。新增动作必须扩展强类型领域模型、安装审查
和宿主实现；不能把下载文件路径传给插件代码作为通用扩展机制。

`Auto` 只跳过“选择哪个动作”的弹窗，不能跳过被选动作自身的安全流程：

- `ImportSkill` 把完成的 artifact 交给现有两阶段 Skill Import 流程。压缩包格式、展开
  预算、路径穿越、冲突和最终提交继续由该流程决定；
- `SaveAs` 只能由可信主界面打开宿主文件选择器。目标存在时不得静默覆盖，复制或移动
  失败不得删除唯一的已完成暂存文件。

`Prompt` 在下载完整且安全落地后才向可信主界面发送选择请求。弹窗显示来源插件、触发
页面 origin、净化后的文件名、文件大小和可选动作，不显示包含凭据的完整 URL。用户
取消等价于丢弃该 artifact。远程页面不能绘制、确认或伪造这个弹窗。

## 下载与处理生命周期

下载请求回调不能等待异步用户输入。登录态下载、POST、Service Worker 和 `blob:` URL
也通常不能由 Ora 取消后通过 HTTP 客户端可靠重放。因此，Ora 必须先完成浏览器传输，
再执行需要用户选择的处理：

```text
browser DownloadEvent::Requested
        ↓
snapshot page URL and select disposition
        ├── Reject: deny immediately
        └── Auto / Prompt
                ↓
reserve host-owned .part destination
                ↓
browser writes using its own cookies and request context
                ↓
DownloadEvent::Finished
        ├── failed: abort and clean reservation
        └── succeeded: validate and atomically complete
                ├── Auto: run declared action
                └── Prompt: notify trusted main Webview
```

状态必须由排他 enum 表达，而不是布尔字段与可选路径组合：

```text
ManagedDownload
├── Staging { intent, reservation }
├── AwaitingChoice { intent, artifact, allowed_actions }
├── Processing { intent, artifact, action }
├── Completed { intent, outcome }
├── Failed { intent, error }
└── Cancelled { intent }
```

只有 `Staging` 持有可写 reservation，只有 `AwaitingChoice` 和 `Processing` 持有已经完成
的 artifact。取得 `Processing` 是用户选择或自动动作的线性化点；同一 `download_id`
不得并发执行两个动作。重复提交用户选择返回稳定冲突，不能再次消费文件。

关闭 WebView 时，仍在浏览器传输中的下载必须取消并清理。已经完成并进入
`AwaitingChoice` 或 `Processing` 的 artifact 由下载模块继续拥有，不因窗口关闭而静默
丢失。应用异常退出后，启动清理只能删除超过宿主保留期限且没有权威处理记录的暂存
文件；不能把孤立 `.part` 当成完整下载。

## 安全落盘

浏览器 profile 和下载工作区位于插件运行数据中，例如：

```text
~/.ora/plugins/data/<namespace>/<plugin_name>/webview/
├── profile/
└── downloads/
```

该路径只是宿主布局，不进入 manifest、远程页面或前端事件。下载 adapter 与动作之间
只通过宿主拥有的 `download_id` 和 `DownloadedArtifact` 传递文件，不能把绝对路径作为
公共身份。

下载文件必须通过共享的安全落盘模块完成。该模块对 HTTP adapter 与 WebView adapter
提供同一组不变量：

1. 只在宿主提供的权威根目录下创建目标和同文件系统暂存文件；
2. 把建议文件名视为不可信输入，移除路径、控制字符、平台非法字符和保留设备名，并
   限制 UTF-8 字节长度；
3. 为并发下载分配不冲突的 reservation，不覆盖磁盘已有文件，也不依赖 URL 唯一；
4. 拒绝符号链接、junction、reparse point 和其他根目录逃逸；
5. 失败或取消只清理当前 reservation，不能删除其他下载或既有目标；
6. 完成时计算实际字节数和 SHA-256，并执行宿主强制的大小上限；
7. 所有检查成功后才在同一文件系统原子提升 `.part`，失败时不产生看似完整的文件；
8. 文件扩展名、响应 MIME、来源 origin 和 SHA-256 结果都不代替动作特定的内容校验。

WebView 引擎控制实际字节流，第一版只能在完成后可靠执行最终大小和摘要检查；超过
限制的文件必须立即删除并返回 `ArtifactTooLarge`。HTTP adapter 仍应在流式传输期间
提前执行相同限制，不能因为共享最终检查而退化为事后检查。

## ExternalWebview 模块

`ExternalWebview` 是可信主界面、Tauri WebView、manifest descriptor、共享安全落盘
模块和下载动作之间的深模块。外部 interface 保持为：

```text
open(plugin_id) -> webview_instance_id
resolve_download(download_id, selected_action) -> DownloadActionOutcome
close(webview_instance_id) -> ()
```

WebView 创建、profile 定位、导航策略、下载事件、URL 快照、规则选择、reservation、
状态机、可信 UI 通知和清理都隐藏在该 interface 后。`resolve_download` 只接受宿主曾为
该 `download_id` 发布的动作之一；请求不能携带插件 ID、文件路径、精确版本或新的动作
集合。

内部 descriptor 至少包含：

```text
InstalledWebviewDescriptor {
  plugin_id,
  exact_version,
  start_url,
  allowed_origins,
  download_policy
}
```

打开实例时取得 descriptor 的不可变快照。插件升级不会改变已打开 WebView 的导航或
下载规则；旧实例必须关闭后才能按新版本重新创建。插件停用、卸载或精确版本失效时，
Ora 必须阻止新下载并关闭对应实例。

## 错误语义

至少提供以下稳定错误类别：

| 错误 | 含义 |
| --- | --- |
| `WebviewPluginNotFound` | 插件不存在、未安装或不是 `webview` kind |
| `WebviewUnavailable` | 实例尚未 Ready、正在关闭或原生 WebView 已失效 |
| `NavigationDenied` | 页面试图离开声明的 HTTPS origin 集合 |
| `DownloadRejected` | 规则或 fallback 明确拒绝下载 |
| `DownloadReservationFailed` | 无法创建唯一、安全的宿主暂存目标 |
| `DownloadFailed` | 浏览器报告传输失败或取消 |
| `ArtifactTooLarge` | 完成文件超过宿主大小上限并已删除 |
| `ArtifactValidationFailed` | 暂存文件缺失、类型异常、逃逸或完成校验失败 |
| `DownloadChoiceRequired` | 该下载必须由可信主界面选择动作 |
| `DownloadActionNotAllowed` | 选择不在该下载冻结的动作集合内 |
| `DownloadAlreadyResolved` | 下载已经进入处理或终态，不能重复消费 |
| `DownloadActionFailed` | 被选中的宿主动作失败，artifact 按动作恢复语义保留或清理 |

错误和前端事件不能包含 cookie、认证 Header、Secret、带敏感 query 的完整 URL、浏览器
profile 路径或下载绝对路径。日志可以记录经过脱敏的 origin、插件身份、精确版本、
`download_id`、阶段、字节数和摘要。

## 验证要求

实现至少覆盖以下测试：

1. `webview` manifest 严格解析，非法 kind-specific 字段、HTTP URL、带凭据 URL、重复或
   wildcard origin、未知动作和非法 matcher 均被拒绝；
2. 远程 WebView 不能调用主界面 command、Workbench bridge、dialog、window、shell、
   filesystem 或其他 Tauri/plugin command；
3. 只允许声明 origin 内的主 frame 导航，popup、新窗口和浏览器权限请求均被拒绝；
4. 规则按顺序 first-match，fallback 缺省拒绝，query 和 fragment 不影响规则选择；
5. 下载期间发生页面跳转时，处理仍使用请求时冻结的 `initiating_page_url` 与
   disposition；
6. 两个相同下载 URL 可以拥有不同 `download_id` 和独立 reservation，并发完成不会
   互相覆盖或清理；
7. 建议文件名的路径穿越、绝对路径、控制字符、平台保留名、超长名称和大小写冲突均
   被安全处理；
8. 失败、取消、超限和完成校验失败只清理本次 `.part`，不会留下完整扩展名的部分文件；
9. `Prompt` 只在文件完成后通知可信主界面，用户只能选择冻结动作集合中的一项，重复
   选择不会重复消费文件；
10. `ImportSkill` 继续经过现有两阶段导入及 archive 安全校验，`SaveAs` 不静默覆盖目标；
11. WebView 关闭取消活动传输但不丢弃已经等待选择或正在处理的 artifact；插件停用、
    卸载、升级和应用重启均遵守声明的清理语义；
12. HTTP 与 WebView adapter 共享落盘不变量测试，同时保留各自独立的传输语义测试。

安全验收必须包含真实 Tauri runtime 集成测试或打包应用 smoke test。单元测试不能证明
远程页面实际上没有匹配 Tauri capability，也不能证明原生下载回调和浏览器 profile
符合预期。

## 第一版不支持

- Webview 插件携带或调用 `main.js`；
- 插件自定义下载处理代码或任意字符串 action；
- `install_plugin`、执行下载文件或自动打开下载文件；
- HTTP、任意 host wildcard、正则 URL matcher、基于 query 或 fragment 的规则；
- 插件自定义 User-Agent、代理、TLS、cookie、Header 或浏览器权限；
- 多个并发 WebView 实例、popup、新窗口或 OAuth popup 特例；
- 页面获知宿主下载路径、处理进度、处理结果或调用 Ora；
- 取消浏览器下载后用 `ora-utils::http` 重放请求；
- 断点续传或应用重启后继续未完成的浏览器传输。

未来扩展必须继续保持远程页面零 Ora 权限、宿主拥有下载身份和路径、动作强类型化以及
URL 路由不等于内容信任这些不变量。
