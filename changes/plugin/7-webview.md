# Webview 插件与共享下载落盘实施指导

本文件指导 `specs/plugin/7-webview.md` 的实现，并记录从现有 Skill Marketplace 下载
代码和 `ora-utils::http` 中抽取共享安全落盘能力的工作。稳定的 manifest、领域语义、
安全边界和错误语义以主规范为准；本文只描述代码落点、迁移顺序和验收标准。

## 目标

实现 `kind = "webview"` 的纯配置插件，并以现有 SkillHub、Huawei Agent Center 原生
WebView 为第一批调用方。实施后必须满足：

- 外部站点运行在无 Ora/Tauri command 权限的独立 WebView；
- manifest 决定入口、导航 origin 和基于当前页面 URL 的下载 disposition；
- 浏览器继续拥有 cookie、SSO、Service Worker、POST 和 `blob:` 下载传输；
- Ora 使用宿主分配的 `download_id` 接管安全落盘和处理生命周期；
- HTTP 与 WebView 下载共享文件 reservation、净化、完成校验和清理不变量；
- Skill Marketplace 不再维护 provider-specific 下载存储实现或向前端暴露绝对路径。

## 非目标

本次不实现插件自定义下载代码、Deno `main.js`、通用浏览器自动化、OAuth popup 特例、
插件安装 action、断点续传或重启后恢复浏览器字节流。也不把浏览器 cookie、Header 或
profile 导出给 `ReqwestDownloader`。

## 模块划分

### `ora-utils::download_target`

在 `crates/utils/src/download_target/` 新建 domain-free 的目录模块，并同时添加英文
`README.md`。它只负责宿主拥有的文件目标，不携带 plugin、skill、WebView、Tauri、URL
规则或用户弹窗词汇。

建议 interface：

```text
DownloadTarget::new(limits)
reserve(TargetRequest) -> DownloadReservation
complete(DownloadReservation) -> DownloadedArtifact
abort(DownloadReservation) -> AbortedDownload

TargetRequest
├── Exact { destination, replacement: ReplacementPolicy }
└── Unique { authoritative_root, suggested_name }
```

`Exact` 保留 `HttpDownload` 的调用语义：调用方已经决定最终 destination，并通过强类型
`ReplacementPolicy` 明确是否允许替换。`Unique` 服务浏览器下载：模块在权威 root 下
根据不可信建议名分配不会覆盖现有文件的最终路径。不能用布尔参数区分这两种语义。

具体 Rust 类型可以根据实现调整，但必须保持以下不变量：

1. reservation 拥有唯一、不透明 ID、同文件系统 part path 和 final path；
2. adapter 可以取得 part path 交给可信传输实现，但该路径不进入前端 contract；
3. `complete` 和 `abort` 消费 reservation，类型层面避免同一值被完成两次；跨回调保存
   reservation 时由拥有它的状态 enum 保证唯一性；
4. part 文件使用不可预测或全局不冲突的名称，不能沿用固定
   `<destination>.tmp` 导致同目标并发写入；
5. `Unique` 对磁盘既有 final/part 文件及当前活动 reservation 同时检查冲突；
6. 文件名净化、Windows 设备名、大小写冲突、UTF-8 字节预算和缺省名规则集中实现；
7. 每次完成重新检查 root containment、普通文件类型和链接逃逸，再计算字节数及
   SHA-256；
8. 所有检查成功后执行同文件系统原子提升；失败或 abort 只删除该 reservation 的 part
   文件；
9. `DownloadedArtifact` 返回受控路径、净化文件名、字节数和 SHA-256，但不承担 archive
   或 Ora 领域格式校验；
10. 启动清理只识别模块自己的 part 命名和元数据，不能按宽泛 glob 删除调用方目录。

`ora-utils::atomic` 继续负责小型完整内容的原子写入；不要为了复用名称把下载流全部
缓存在内存中。`download_target` 负责已经由 adapter 流式写入的 part 文件及其最终提升。

### `ora-utils::http`

`HttpDownload` 继续是 Ora 主动获取 URL 或本地文件的传输 interface。保留：

- proxy、retry/backoff、timeout、progress 和 cancellation；
- 传输期间的增量字节限制和 SHA-256；
- 可选预期 checksum 校验；
- `DownloadRequest` 与 `DownloadOutcome` 的既有调用语义。

把 `crates/utils/src/http/target.rs` 中通用的目标创建、清理和原子提升逻辑改为调用
`download_target`。HTTP adapter 仍必须边下载边执行限额和摘要，完成时可以把已计算
结果交给目标模块或与最终检查交叉验证。不能因为共享模块在完成后会检查，就移除流式
限额或在磁盘上多复制一遍完整文件。

迁移必须保持现有测试证明的“失败不破坏既有 destination”语义。对同一 exact
destination 的并发下载要么由 reservation 明确拒绝冲突，要么使用互不冲突的 part
文件并定义最终线性化语义；不能让两个 writer 共享同一个 `.tmp`。

### Desktop `ExternalWebview`

在 `apps/desktop/src-tauri/src/` 下建立独立目录模块，并按仓库规则添加英文 README。
它拥有：

- `InstalledWebviewDescriptor` 到原生窗口配置的转换；
- 每插件 ID 单实例窗口与 profile 生命周期；
- navigation、new-window、browser-permission 和 Tauri ACL 策略；
- 下载事件到 `DownloadIntent`、manifest disposition 和 `DownloadReservation` 的编排；
- active download registry、可信前端事件、用户选择及 action 调用；
- 插件停用、升级、卸载、窗口关闭与应用退出清理。

外部 interface 按主规范收敛为 `open`、`resolve_download` 和 `close`。Tauri callback 是
模块内部 adapter，不把 `DownloadEvent`、WebView label 或裸路径泄露给其他模块。

registry 至少按以下身份索引：

```text
ExternalWebviewRegistry {
  by_plugin: Map<PluginId, WebviewInstanceId>,
  by_webview: Map<WebviewLabel, WebviewInstanceEntry>,
  downloads: Map<DownloadId, ManagedDownload>
}
```

所有索引和状态 variant 必须在同一临界区原子更新。下载事件按宿主 `download_id`
关联；若底层 Tauri `Finished` 事件只提供 URL 或 destination，平台 adapter 负责把它映射
回 reservation，不能让 URL 成为领域身份。需要明确处理同一 WebView、同一 URL 的并发
下载测试。

### 下载动作

第一版实现两个宿主动作：

- `ImportSkill`：把 `DownloadedArtifact` 适配为现有 `SkillImportSource::Archive`，继续使用
  prepare、review、commit/cancel 的两阶段流程；
- `SaveAs`：由可信主界面选择目标，使用安全复制或移动，并避免静默覆盖。

动作实现接收 `DownloadedArtifact`，不重新接收来自前端的源路径。处理模块返回强类型
outcome，明确 artifact 在成功、可重试失败、不可恢复失败和用户取消后的保留或清理
责任。前端只提交 `download_id` 和主规范允许的 action enum。

## Manifest 与安装 descriptor

扩展 `ora-plugin-manifest` 的 resolver 1 闭集，加入 `PluginKind::Webview` 及严格的
`[webview]`、downloads、rules、matcher 和 action 类型。不要把 kind-specific 字段继续
堆进只包含公共 metadata 的单一 struct；使用排他 contribution 模型，使 Agent、
Workbench、MCP 和 Webview 的非法字段组合不可表达。

安装 `.orax` 时编译并持久化不可变 `InstalledWebviewDescriptor`。至少完成：

1. 公共 manifest、registry identity、kind 和精确版本校验；
2. `start_url` 与 exact `allowed_origins` 规范化；
3. rule 顺序、matcher、fallback 和 action 闭集校验；
4. 重复及完全遮蔽规则诊断；
5. 确认包不要求 `main.js`，也不声明 Webview kind 不允许的 Capability；
6. 生成运行时不再需要重新解释 TOML 的强类型 descriptor。

主规范示例使用 `[webview.downloads]` 与 `[[webview.downloads.rules]]`；实现必须为该形状
增加解析 round-trip 和未知字段测试，不能引入另一套 JSON 配置。

## Tauri ACL 与运行时验证

为外部插件 WebView 使用与 `main`、`plugin-webview:*` 均不重叠的宿主 label 前缀。不要
为该前缀声明任何 Ora 或 Tauri command capability；下载和导航只能由构建 WebView 时
注册的 Rust callback 处理。

更新 command registry/ACL 一致性测试，证明：

- `main` 仍精确获得主界面 command；
- Workbench Webview 仍只获得 `plugin_webview_invoke`；
- External Webview 对两组 command 均无权限；
- 直接从远程页面构造 invoke 也会在 ACL 层失败。

真实 runtime smoke test 要使用本地测试 HTTPS 站点或可控打包 fixture，覆盖导航、下载
和 direct invoke。仅 mock URL matcher 或 JavaScript SDK 不能作为安全验收。

## Skill Marketplace 迁移

现有实现位于：

- `apps/desktop/src-tauri/src/skill_marketplace.rs`；
- `apps/desktop/src-tauri/src/skill_marketplace/downloads.rs`；
- `apps/desktop/web/tauri-platform-adapter.ts` 中的 provider-specific capability；
- App Shell 中消费 `skill-marketplace://status` 的导入流程。

迁移采用替换而不是兼容层：

1. 为 SkillHub 和 Huawei Agent Center 提供 `kind = "webview"` descriptor 或正式插件包，
   显式列出经过验证的 exact origin；
2. 用 `ExternalWebview.open(plugin_id)` 替换 `SkillMarketplaceProvider::definition()` 和
   `open_skill_marketplace(provider)`；
3. 把 `SkillDownloadCoordinator` 中通用的净化、唯一命名、reservation、`.part`、完成和
   清理测试迁移到 `ora-utils::download_target`；
4. 把 Skill archive 的后续处理迁移到 `ImportSkill` action；
5. 用通用 typed download events 替换 `skill-marketplace://status`，事件携带
   `download_id` 和安全展示元数据，不携带 `archive_path`；
6. 删除 provider enum、硬编码窗口定义、`skill-downloads` 特化目录和旧前端 capability；
7. 更新 Desktop 与相关目录 README，使 Webview ownership 和非职责与新模块一致。

Huawei 当前使用整个 `*.huawei.com` suffix 作为临时导航策略。新规范不允许 wildcard，
迁移前必须在真实 SSO 流程中记录并确认精确 origin 清单；清单未知时不能通过保留旧的
宽泛 suffix 绕过 manifest 规则。

## 实施顺序

### 阶段一：共享安全落盘

1. 新增 `ora-utils::download_target` 及 README；
2. 将 `SkillDownloadCoordinator` 的 domain-free 行为用共享模块测试重新表达；
3. 迁移 `LocalFileDownloader` 和 `ReqwestDownloader` 的目标处理，保持现有 interface；
4. 运行 `ora-utils` 最小 feature、`http` 和 `http-reqwest` 各 feature 组合测试；
5. 在调用方迁移完成后删除重复 helper，不能长期保留两套净化和原子完成实现。

### 阶段二：Manifest 与 descriptor

1. 增加 `PluginKind::Webview` 和排他 contribution；
2. 实现 URL、origin、matcher、disposition 和 action 强类型；
3. 编译及持久化 `InstalledWebviewDescriptor`；
4. 更新 registry/index 页面所需 kind 展示与兼容性测试。

### 阶段三：External Webview runtime

1. 实现单实例窗口、独立 profile 和生命周期 registry；
2. 接入严格导航、popup、权限与 ACL 策略；
3. 接入浏览器下载事件、URL 快照、共享 reservation 和状态机；
4. 实现通用状态事件、`resolve_download` 和两个宿主 action；
5. 增加真实 Tauri runtime 安全测试。

### 阶段四：替换 Marketplace 特化代码

1. 迁移两个现有 provider；
2. 删除旧 Rust、TypeScript contract、事件和前端 capability；
3. 更新模块 README 与用户文档；
4. 运行完整 `task test`，确认 stderr gate、Rust lint 和打包配置均通过。

## 验收标准

- `kind = "webview"` 可以从 registry 安装、启用、打开、升级、停用和卸载；
- 包不运行 `main.js`，远程页面无法调用任何 Ora/Tauri command；
- 浏览器 profile 按插件 ID 隔离并跨精确版本升级保留；
- exact origin 导航、popup 和浏览器权限规则通过单元及真实 runtime 测试；
- 下载使用触发时主页面 URL 快照，first-match 与默认拒绝语义稳定；
- HTTP GET、登录态下载、POST 触发和 `blob:` fixture 均走浏览器原始请求上下文，不由
  `ReqwestDownloader` 重放；
- 相同 URL 并发下载不会共享身份、part path 或最终文件；
- HTTP 与 WebView adapter 共享文件名、reservation、完成校验、原子提升和清理测试；
- HTTP 保留流式字节限制、checksum、retry、timeout、progress 和 cancellation；
- WebView 超限、失败和取消不会留下可被误认为完整 artifact 的文件；
- Prompt 只在完成后由可信主界面展示，非法或重复 action 选择稳定失败；
- `ImportSkill` 不绕过 archive 校验和两阶段 review，`SaveAs` 不静默覆盖目标；
- 前端 contract、事件和日志均不暴露绝对下载路径、profile 路径、cookie、认证信息或带
  敏感 query 的完整 URL；
- 旧 `SkillDownloadCoordinator`、provider-specific 下载事件和重复 target helper 已删除；
- `task format`、最小相关测试、`task lint` 和最终 `task test` 全部通过。
