# Hook 插件静态包与命令改写协议

## Invalid Hook Package Must Fail Closed Without Starting A Plugin Process

### 风险

如果 Hook shape 可以由其他 kind 携带、非法配置被部分接受、或宿主需要启动插件进程才能获得
descriptor，同一个 Plugin ID/version 的贡献与权限边界将不再确定；携带原生可执行文件的恶意包还
可能借 Hook 名义获得代码执行机会，或在错误的架构上落地一个无法运行的二进制。

### 前置状态

- installed package 的 `orax.toml` 声明 `kind = "hook"`，或其他 kind 错误携带 Hook shape。
- 包可能缺少 `assets/config.json`、携带 `main.js`、声明错误 shape，或 `executable` 逃逸 package。

### 触发

`PluginManager::discover` 选择最高 installed version 并验证 contribution，随后 lifecycle 构造该插件的
运行时状态。

### 必须成立

- release 与 installed manifest 接受 Hook kind，且拒绝进程/页面专属 section。
- Hook kind 必须无 `main.js`，并携带唯一可编译的 Hook Configuration。
- Hook-bearing configuration 只能属于 Hook kind；其他 kind 声明 `hook` 时整包失败。
- `hook.executable` 必须 canonicalize 为当前 package `assets/` 内的普通非链接文件；Windows 上必须
  以 `.exe` 结尾。
- 任一条件失败时，整包不进入 installed snapshot，并返回稳定 field path；有效 Hook 也不创建 runtime、
  不注册方法、不获得 Deno 权限。

### 禁止结果

- 忽略非法 Hook shape 后把插件当作普通 Settings-only 插件安装。
- 因最高版本损坏而静默回退到较低版本。
- 通过 symlink、reparse point、PATH lookup 或父目录穿越执行包外 executable。
- 为 Hook contribution 启动 `main.js` 或允许 runtime handshake。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| manifest 接受 Hook kind 并拒绝 workbench/webview section | `ora_plugin_manifest::tests::parses_targeted_hook_manifest`; `ora_plugin_manifest::tests::hook_kind_rejects_workbench_and_webview_sections` | `Covered` |
| Hook package 必须无 entrypoint 且配置必须是 Hook shape | `ora_plugin_manager::kind_tests::rejects_hook_package_entrypoint_and_config_shape_violations` | `Covered` |
| 非 Hook kind 不得携带 Hook shape | `ora_plugin_config::hook::tests` 与 `ora_plugin_manager::validation` 互斥校验；非 Hook kind 携带 `hook` 由 manifest 校验拒绝，尚无脱离 discovery 的专属 fixture | `Partial` |
| executable 必须存在、是普通文件且留在 package assets 内 | `ora_plugin_manager::kind_tests::rejects_hook_executable_containment_and_extension_violations`; `rejects_hook_executable_that_is_a_directory`; `rejects_hook_executable_symlink_escape` | `Covered` |
| Hook 不注册、不启动且权限集为空 | `ora_plugin_lifecycle::registration::tests::hook_plugins_cannot_register`; `ora_plugin_lifecycle::permissions` 对 Hook 返回空集 | `Covered` |
| 最高版本损坏时不回退 | 通用插件发现测试覆盖该规则，尚无 Hook 专属双版本 fixture | `Partial` |

### 决策依据

- Hook 插件根决策 D1、D2、D4：Hook 是无进程、却携带原生可执行文件的单一静态贡献，所有安装期
  验证 fail closed。
- Hook 插件根决策不变量 1、2、4、9：包 shape、executable containment 与权限边界必须稳定。

## Hook Descriptor Must Remain Unambiguous Before Runtime Resolution

### 风险

如果 descriptor 字段能携带未知值、command alias 能伪装成路径、或 tool version 不独立于插件版本，
session setup 可能在读取配置后才发现声明不可解释，并把协议兼容性误判为插件版本兼容。

### 前置状态

- `assets/config.json` 声明 schema version 1。
- configuration 包含唯一 Hook Protocol descriptor，并可能声明 Settings。

### 触发

配置编译器将 JSON 编译为 `CompiledHookConfiguration`，供后续运行边界绑定精确 Configuration revision。

### 必须成立

- `hook.protocol` 必须解析为已知枚举变体；当前仅 `rtk-rewrite-v1`。
- `hook.executable` 必须是 package `assets/` 内的安全相对路径。
- `hook.command` 必须是 bare ASCII 名，禁止空、路径分隔符、控制字符、空白与非 ASCII。
- `hook.toolVersion` 必须是合法 SemVer，且独立于 Hook Plugin 版本表达协议兼容性。
- 顶层、Settings 与 descriptor 均拒绝未知字段、未知 schema version 与未知 protocol。
- 编译结果只保存 Setting identity，不读取或保存用户 Setting 实值。

### 禁止结果

- 同一文件同时声明 `transport` 与 `hook`，或 descriptor 携带未知字段。
- 让 command alias 携带路径分隔符，使 Hook 伪装成任意文件系统路径。
- 把插件 SemVer 当成协议兼容性的唯一输入。
- 在静态验证期间解析、复制或记录 Setting 实值。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| 合法 descriptor 编译为强类型 descriptor | `ora_plugin_config::hook::tests::compiles_the_rtk_hook_configuration` | `Covered` |
| 未知 schema version、未知 protocol 与缺失/未知字段失败 | `rejects_unsupported_schema_version`; `rejects_unknown_protocol`; `rejects_missing_descriptor_field`; `rejects_unknown_descriptor_field` | `Covered` |
| command alias 拒绝路径分隔符、空白与控制字符 | `rejects_command_with_path_separator`; `hook_command_rejects_whitespace_and_separators` | `Covered` |
| toolVersion 必须是合法 SemVer | `rejects_non_semver_tool_version` | `Covered` |
| 可选 Settings subset 与保留类型策略生效 | `compiles_hook_configuration_with_settings_subset`; `rejects_reserved_setting_type` | `Covered` |
| 编译结果保持 Setting 引用且不读取用户配置值 | compiler 输入只包含插件包声明，尚无直接断言序列化结果不含配置值的专属测试 | `Partial` |

### 决策依据

- Hook 插件根决策 D2、D3：配置使用封闭 schema、四个字段各自封闭校验，tool version 独立于插件
  版本。
- Hook 插件根决策不变量 2、3、5、6：shape、protocol 枚举、command 裸名、tool version 与 executable
  containment 不得由运行时猜测。

## Target Selection Must Prove Host Compatibility Without Architecture Fallback

### 风险

如果 target 选择能跨架构回退、targeted 包不必自述 `[artifact]`、或本地导入路径不校验 host，
一个错误架构的原生二进制会作为有效包落地，把打包错误推迟成运行时崩溃或无法启动的 Hook。

### 前置状态

- release manifest 用 `[[targets]]` 声明精确 target 资产，或 installed manifest 用 `[artifact]` 自述
  target。
- host 可能不在 `CANONICAL_RUST_TARGET_TRIPLES` 内，或 host 三元组无匹配资产。

### 触发

marketplace install 经 `select_release` 选择资产，installer 解压后校验包内 `[artifact]` 与所选
release target 一致；本地 `.orax` import 按 `HostTarget` 校验自述 target。

### 必须成立

- `HookTarget` 解析器只接受 `CANONICAL_RUST_TARGET_TRIPLES` 内的已知三元组，未知字符串不解析。
- host 不支持时 marketplace 选择返回 `UnsupportedHost`，host 三元组无匹配资产时返回
  `NoArtifactForTarget`。
- targeted 包解压后必须携带 `[artifact]`，且其 target 与所选 release target 逐字节相等。
- installed Hook 缺 `[artifact]` 直接失败；本地导入路径对自述 target 的包按当前 host 校验。
- target 匹配精确，不跨架构、操作系统、libc 或 ABI 回退。

### 禁止结果

- 把 `x86_64-unknown-linux-gnu` 当成 `x86_64-unknown-linux-musl` 的可替代 target。
- targeted 包缺失 `[artifact]` 仍安装为有效。
- 本地导入 Hook 时跳过 host 校验或允许 `Unsupported` host 放行。
- 接受未知三元组如 `not-a-real-triple` 作为可安装架构。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| `HookTarget` 接受已知三元组并拒绝未知值 | `ora_plugin_manifest::target::tests::parses_the_windows_x86_64_msvc_triple`; `rejects_invalid_target_triples` | `Covered` |
| targeted Hook release 安装并校验 `[artifact]` 一致 | `ora_plugin_manager::install::tests::installs_targeted_hook_release_with_executable`; `rejects_targeted_hook_release_with_mismatched_artifact_target` | `Covered` |
| targeted 包缺失 `[artifact]` 失败 | `rejects_targeted_hook_release_without_artifact_section` | `Covered` |
| host 不支持或无匹配资产时选择失败 | `select_release_rejects_unsupported_host_target`; `select_release_rejects_a_targeted_release_without_a_host_target` | `Covered` |
| 本地导入 Hook 按 host 校验自述 target | `rejects_local_hook_import_with_mismatched_host_target`; `rejects_local_hook_import_without_a_host_target` | `Covered` |
| universal Hook release 可在非 target host 安装 | `select_release_accepts_a_universal_release_without_a_host_target` | `Covered` |

### 决策依据

- Hook 插件根决策 D5：target 选择精确匹配，installed 包必须自述 `[artifact]`，不跨架构回退。
- Hook 插件根决策不变量 7：target 匹配必须精确，且独立于 marketplace 元数据证明主机兼容。

## Command Alias Conflict Must Be Observed Without Forcing Uniqueness

### 风险

如果同名 command alias 的多个 Hook 被静默共存而无观测，PATH 解析会在未来 consumer 调用时歧义选择
错误的 Hook；如果冲突时禁用或拒绝安装任一方，会丢失用户意图，并伪装成“无 enablement 状态即可
禁用”。

### 前置状态

- 一个已安装的 Hook 贡献了 bare command alias（如 `rtk`）。
- 另一个 Hook 以相同 command alias 安装，可能来自不同 namespace 或 Plugin ID。

### 触发

`finalize_new_install` 在安装成功并刷新 installed snapshot 后调用
`detect_hook_command_conflict`，比较新 Hook 与其他已安装 Hook 的 command alias。

### 必须成立

- 安装总会保留包：Ora 没有 enablement 状态，冲突不会禁用或卸载任一方。
- 同名 command alias 冲突时安装结果为 `InstalledWithCommandConflict`，携带冲突方 canonical
  Plugin ID。
- 重新安装同一插件不与自身冲突：检测显式排除新安装的 Plugin ID。
- 非 Hook kind 不参与 command alias 冲突检测。
- frontend 接收扁平的 `InstalledPluginContribution::Hook`，只含 protocol、command、target 与
  toolVersion，不学习 executable 路径。

### 禁止结果

- 冲突时禁用、卸载或拒绝安装任一 Hook 包。
- 把冲突静默成普通 `Installed` 而不报告冲突身份。
- 让新安装的 Hook 与自身冲突。
- 把 executable 路径、origin allow list 或下载规则暴露给 frontend。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| 第二个同名 command alias 的 Hook 安装报告冲突且不禁用 | `ora_backend::plugin::operations::install_tests::importing_a_second_hook_with_the_same_command_reports_a_conflict_without_disabling` | `Covered` |
| Hook 运行时恒为 Stopped 且全局可用 | `ora_plugin_lifecycle::tests::manages_static_hook_plugin_without_a_runtime` | `Covered` |
| marketplace 安装、卸载与发现复用同一 contract | `ora_backend::plugin::tests::rtk_hook_plugin_clean_data_directory_e2e`（需 `RTK_RELEASE_ORAX` 环境变量，CI 默认跳过） | `Partial` |

### 决策依据

- Hook 插件根决策 D7、D8：command alias 冲突以观测而非强制唯一性处理；Hook 是全局可用、运行时恒
  为 stopped 的 processless 插件。
- Hook 插件根决策不变量 9、10：不创建 runtime，冲突以 `InstalledWithCommandConflict` 观测。
