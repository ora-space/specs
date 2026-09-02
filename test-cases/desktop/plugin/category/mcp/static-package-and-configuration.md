# MCP 插件静态包与配置资格

## Invalid MCP Package Must Fail Closed Without Starting A Plugin Process

### 风险

如果 MCP shape 可以由其他 kind 携带、非法配置被部分接受，或宿主需要启动插件进程才能获得 transport，
同一个 Plugin ID/version 的贡献与权限边界将不再确定；恶意包还可能借 MCP 名义获得代码执行机会。

### 前置状态

- installed package 的 `orax.toml` 声明 `kind = "mcp"`，或其他 kind 错误携带 MCP transport。
- 包可能缺少 `assets/config.json`、携带 `main.js`、声明错误 shape，或具有逃逸 package 的 stdio
  command。

### 触发

`PluginManager::discover` 选择最高 installed version 并验证 contribution，随后 lifecycle 构造该插件的
运行时状态。

### 必须成立

- release 与 installed manifest 接受 MCP kind，且拒绝进程/页面专属 section。
- MCP kind 必须无 `main.js`，并携带唯一可编译的 MCP Configuration。
- transport-bearing configuration 只能属于 MCP kind。
- `stdio.command` 必须 canonicalize 为当前 package `assets/` 内的普通文件。
- 任一条件失败时，整包不进入 installed snapshot，并返回稳定 field path；有效 MCP 也不创建 runtime、
  不注册方法、不获得 Deno 权限。

### 禁止结果

- 忽略非法 transport 后把插件当作普通 Settings-only 插件安装。
- 因最高版本损坏而静默回退到较低版本。
- 通过 symlink、reparse point、PATH lookup 或父目录穿越执行包外 command。
- 为 MCP contribution 启动 `main.js` 或允许 runtime handshake。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| manifest 接受 MCP kind 并拒绝 workbench/webview section | `ora_plugin_manifest::tests::parses_mcp_kind_marketplace_manifest`; `ora_plugin_manifest::tests::mcp_kind_rejects_workbench_and_webview_sections` | `Covered` |
| MCP package 必须无 entrypoint 且配置必须是 MCP shape | `ora_plugin_manager::kind_tests::rejects_mcp_package_entrypoint_and_config_shape_violations` | `Covered` |
| 非 MCP kind 不得携带 transport | `ora_plugin_manager::kind_tests::rejects_transport_bearing_configuration_on_other_kinds` | `Covered` |
| stdio command 必须存在并留在 package 内 | `ora_plugin_manager::kind_tests::validates_stdio_command_containment_on_disk`; parser 层另覆盖绝对路径、穿越与 PATH lookup | `Partial` |
| MCP 不注册、不启动且权限集为空 | `ora_plugin_lifecycle::registration::tests::mcp_plugins_cannot_register`; `ora_plugin_lifecycle::permissions::tests::mcp_plugins_get_no_permissions` | `Covered` |
| 最高版本损坏时不回退 | 通用插件发现测试覆盖该规则，尚无 MCP 专属双版本 fixture | `Partial` |

### 决策依据

- MCP 插件根决策 D1、D2、D5：MCP 是无进程单一静态贡献，所有安装期验证 fail closed。
- MCP 插件根决策不变量 1、2、4、8：包 shape、command containment 与权限边界必须稳定。

## Transport Bindings Must Remain Unambiguous Before Runtime Resolution

### 风险

如果 transport shape 能混合、Setting 引用悬空、HTTP literal 能携带凭据，或绑定文本允许控制字符，
session setup 可能在读取用户值后才发现声明不可解释，并产生注入或秘密泄漏风险。

### 前置状态

- `assets/config.json` 声明 schema version 1。
- configuration 包含 `stdio` 或 HTTP transport，并可能声明 Settings 与绑定表达式。

### 触发

配置编译器将 JSON 编译为 `CompiledMcpConfiguration`，供后续运行边界绑定精确 Configuration revision。

### 必须成立

- `stdio` 与 HTTP 是互斥枚举，跨 transport 字段、未知字段、未知版本和未知 transport 均失败。
- 所有 Setting 引用都指向本文件声明的 Setting，绑定文本拒绝控制字符。
- HTTP 只接受无 userinfo、query、fragment 的 HTTPS URL，header 值只接受 Setting 引用。
- stdio 环境变量名遵守 portable grammar，workspace context 与普通 literal 保持不同语义。
- 编译结果只保存 Setting identity/binding，不读取或保存用户 Setting 实值。

### 禁止结果

- 同一文件同时声明 `transport` 与 `hook`，或同时承载 stdio/HTTP 字段。
- 将未声明 Setting、literal API key 或 CR/LF 注入 transport。
- 把 `.` literal 参数自动解释为 Workspace context。
- 在静态验证或资格判断期间解析、复制或记录 Setting 实值。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| HTTP 与 stdio 合法 shape 编译为互斥 transport | `ora_plugin_config::mcp::tests::compiles_http_configuration_with_header_setting_reference`; `ora_plugin_config::mcp::tests::compiles_stdio_configuration_with_arguments_and_environment` | `Covered` |
| 未知结构、跨 transport 字段及 MCP/Hook 混合失败 | `ora_plugin_config::mcp::tests::rejects_unknown_fields_versions_and_transport_types`; `rejects_cross_transport_field_combinations`; `rejects_a_mixed_transport_and_hook_declaration` | `Covered` |
| HTTP URL、header、Setting 引用和控制字符策略生效 | `rejects_http_url_policy_violations`; `rejects_invalid_header_names_and_header_literals`; `rejects_references_to_undeclared_settings`; `rejects_control_characters_in_stdio_literals` | `Covered` |
| command path 与环境变量 grammar 可移植 | `rejects_commands_outside_the_package_assets_directory`; `rejects_invalid_environment_variable_names` | `Covered` |
| 编译结果保持 Setting 引用且不读取用户配置值 | compiler 输入只包含插件包声明，尚无直接断言序列化结果不含配置值的专属测试 | `Partial` |
| workspace context 不与普通 literal 混淆 | parser 分别产生 `WorkspaceContext` 与 `Value`，尚无脱离运行时 renderer 的完整行为测试 | `Partial` |

### 决策依据

- MCP 插件根决策 D2–D4：配置使用封闭 schema、同源 Setting binding 与互斥 transport。
- MCP 插件根决策不变量 2、3、5：shape、秘密边界和 HTTP 安全策略不得由运行时猜测。

## Configuration Completeness Must Not Claim Agent Availability

### 风险

如果配置完整性被解释为 Agent 已接受 MCP 或 Server 已在线，设置界面会在真正的 session setup 之前给出
无法证明的可用性承诺；如果 required Setting 缺失仍获得资格，运行边界只能制造半配置 Server。

### 前置状态

- 一个静态有效的 MCP 插件已安装。
- 插件可能不需要 Settings，或声明了缺失、完整、随后修改或清除的 Settings。
- Configuration Store 可能损坏或包含与当前声明不兼容的旧值。

### 触发

宿主读取插件配置详情，或安装、保存、清除配置后重新计算 Configuration summary。

### 必须成立

- 无 Settings 的 MCP 使用配置 revision `0`，配置状态为 `Complete`。
- 有 Settings 的 MCP 只有所有 required 值存在且类型正确时才为 `Complete`。
- 保存、清除和恢复损坏配置保持 revision 单调，并重新计算完整性。
- 配置 DTO 只报告未配置完整、配置完整或配置损坏，不从该状态推断 Agent capability、setup 成功或
  Server 健康。
- Setting 引用值在配置详情 DTO 中按既有 redaction 规则处理，不因 MCP 资格判断泄漏。

### 禁止结果

- 用默认空字符串填补 required Setting 并报告 `Complete`。
- 把 configuration complete 命名为 `MCP Ready`、`Applied` 或其他运行时状态。
- 配置损坏时退化为 revision `0` 或无配置插件。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| required Setting 缺失与完整值产生不同 summary | `ora_plugin_config::service::tests` 的 configuration completeness 覆盖；backend Tavily 配置流程提供高层证据 | `Covered` |
| 保存、清除与损坏恢复保持 revision 语义 | `ora_plugin_config::service::tests` 的 save/reset/recovery 用例 | `Covered` |
| 本地 import 和 marketplace install 复用同一配置 contract | `ora_backend::bootstrap::tests::tavily_mcp_local_import_and_configuration`; `tavily_mcp_marketplace_install_and_configuration` | `Covered` |
| MCP 绑定 Setting 在配置 DTO 中脱敏并可显式保留 | typed redaction + `preserveSettingIds` backend/frontend tests | `Covered` |

### 决策依据

- MCP 插件根决策 D3、D6：Configuration revision 决定配置完整性与后续解析资格，不构成运行时状态。
- MCP 插件根决策不变量 3、6、7：静态有效、配置完整与 Agent 消费必须保持为不同事实。
