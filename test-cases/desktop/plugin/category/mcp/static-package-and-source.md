# MCP 插件静态包与 Effect Source

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

## Transport Bindings Must Remain Unambiguous And Secret Free

### 风险

如果 transport shape 能混合、Setting 引用悬空、HTTP literal 能携带凭据，或绑定文本允许控制字符，
发布后的模板可能产生注入、秘密泄漏或不同消费端解释不一致。

### 前置状态

- `assets/config.json` 声明 schema version 1。
- configuration 包含 `stdio` 或 HTTP transport，并可能声明 Settings 与绑定表达式。

### 触发

配置编译器将 JSON 编译为 `CompiledMcpConfiguration`，后续模板解析器生成 secret-free Agent-specific
materialization input。

### 必须成立

- `stdio` 与 HTTP 是互斥枚举，跨 transport 字段、未知字段、未知版本和未知 transport 均失败。
- 所有 Setting 引用都指向本文件声明的 Setting，绑定文本拒绝控制字符。
- HTTP 只接受无 userinfo、query、fragment 的 HTTPS URL，header 值只接受 Setting 引用。
- stdio 环境变量名遵守 portable grammar，workspace context 与普通 literal 保持不同语义。
- 编译及模板结果只保存 Setting identity/binding，不保存用户值；不同 Agent 所需编码在消费端生成。

### 禁止结果

- 同一文件同时声明 `transport` 与 `hook`，或同时承载 stdio/HTTP 字段。
- 将未声明 Setting、literal API key 或 CR/LF 注入 transport。
- 把 `.` literal 参数自动解释为 Workspace context。
- 将 Setting 实值写入 immutable MCP template。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| HTTP 与 stdio 合法 shape 编译为互斥 transport | `ora_plugin_config::mcp::tests::compiles_http_configuration_with_header_setting_reference`; `ora_plugin_config::mcp::tests::compiles_stdio_configuration_with_arguments_and_environment` | `Covered` |
| 未知结构、跨 transport 字段及 MCP/Hook 混合失败 | `ora_plugin_config::mcp::tests::rejects_unknown_fields_versions_and_transport_types`; `rejects_cross_transport_field_combinations`; `rejects_a_mixed_transport_and_hook_declaration` | `Covered` |
| HTTP URL、header、Setting 引用和控制字符策略生效 | `rejects_http_url_policy_violations`; `rejects_invalid_header_names_and_header_literals`; `rejects_references_to_undeclared_settings`; `rejects_control_characters_in_stdio_literals` | `Covered` |
| command path 与环境变量 grammar 可移植 | `rejects_commands_outside_the_package_assets_directory`; `rejects_invalid_environment_variable_names` | `Covered` |
| 模板不含秘密并为 Agent 产生正确编码 | `ora_effect_mcp::template::tests::http_templates_are_secret_free_while_each_agent_receives_its_required_encoding` | `Covered` |
| workspace context 不与普通 literal 混淆 | `ora_effect_mcp::template::tests::workspace_context_does_not_replace_a_literal_dot_argument` | `Covered` |

### 决策依据

- MCP 插件根决策 D2–D4：配置使用封闭 schema、同源 Setting binding 与互斥 transport。
- MCP 插件根决策不变量 2、3、5：shape、秘密边界和 HTTP 安全策略不得由消费端猜测。

## Only Complete Configuration May Publish The Current MCP Source

### 风险

如果未配置的 MCP 也进入 Desired State，Agent 项目会出现不可启动的半成品 Server；如果插件版本或配置
变化原地改写 revision，已有 Workspace 无法可靠判断是否需要收敛或退役旧配置。

### 前置状态

- 一个静态有效的 MCP 插件已安装。
- 该插件可能不需要 Settings，或声明了未填写、完整、随后修改/清除的 Settings。
- SQLite 中可能存在同一 Plugin ID 的旧 MCP Source revision。

### 触发

后端启动、安装/更新、本地 import、配置保存/清除或卸载触发 MCP Source 同步。

### 必须成立

- 无 Settings 的 MCP 以配置 revision `0` 发布；有 Settings 的 MCP 仅在配置 `Complete` 时发布。
- revision key 同时包含插件版本与配置 revision，且相同 key 的 definition 不可变。
- 配置保存产生新 revision 并推进受影响 Workspace；配置清除、不完整、插件缺失或卸载退役当前 Source。
- Effect Definition 只包含 transport 模板和 Setting binding，不包含用户 Setting 实值。
- 启动同步以完整 installed MCP snapshot 清理不再存在的 active MCP Source。

### 禁止结果

- 用默认空字符串填补 required Setting 并发布 Source。
- 配置值变化后继续把旧 revision 报告为 current。
- 卸载插件后保留 active MCP Source，或删除其他 Plugin ID 的 Source。
- 因项目物化失败而改写 installed package 的静态有效性。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| 本地 import 后未配置 MCP 为 incomplete，保存配置后变 complete | `ora_backend::bootstrap::tests::tavily_mcp_local_import_and_configuration` | `Covered` |
| marketplace install 与配置复用同一 contract | `ora_backend::bootstrap::tests::tavily_mcp_marketplace_install_and_configuration` | `Covered` |
| 配置保存使用 desktop plugin home 并推进 revision | `ora_backend::bootstrap::tests::tavily_mcp_save_configuration_in_desktop_plugin_home` | `Covered` |
| Source revision key 绑定插件版本与配置 revision，配置不完整时退役 | 当前由 `persist_discovered_plugin_mcp` 实现并被后端流程间接覆盖，缺少 repository 级直接断言 | `Partial` |
| 启动 snapshot 退役已不存在插件的 Source | 当前有实现路径，尚无删除 installed package 后重启的直接集成测试 | `Missing` |
| 配置清除、插件更新与显式卸载分别推进所有受影响 Workspace | Effect repository 与 API 有分层覆盖，尚无一组 MCP 专属端到端生命周期测试 | `Partial` |

### 决策依据

- MCP 插件根决策 D3、D6：配置完整性控制 Source publication，版本与配置 revision 共同定义不可变身份。
- MCP 插件根决策不变量 6、7：静态有效性与运行可用性分离，Source 生命周期必须可收敛。
- Agent 项目文件 ownership、原子 mutation、单次重启和 readiness 证据由 MCP Agent Effect 决策及其核心
  用例继续验证。
