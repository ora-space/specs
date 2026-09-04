# 插件清单 schema 与校验

架构边界与不变量来自
[插件清单根决策](../../../../decisions/desktop/plugin/manifest/0-versioned-dual-form-manifest.md)。
namespace 归属与安装目录身份匹配的核心用例在
[registry 核心测试用例](../registry/identity-and-namespace.md)，本文不重复。

## One Manifest Text Must Yield a Fully Validated Domain Object or a Stable Field Path

### 风险

清单里的值最终会被拼进下载校验、导航判定、进程参数与文件系统路径。如果任何一个值以未校验的
字符串进入域模型，或错误只给出一句消息而不指向字段，作者可以用一份看似合法的清单让宿主接受
非 HTTPS 下载、带凭据的 URL、控制字符或超长文本，而失败又无法被自动定位。

### 前置状态

- 一份 release 或 installed 形态的 `orax.toml` 文本。
- 文本可能合法，也可能在任意单个字段上违反文本、URL、digest 或结构策略。

### 触发

`ora-plugin-manifest` 解析该文本，随后 `ora-plugin-manager` 在包发现时把失败映射为 discovery
issue。

### 必须成立

- 合法清单产生一个所有字段都已通过其类型策略的域对象，可选字段缺省时取确定的缺省语义。
- 文本字段拒绝空值、首尾空白与控制字符，并各自受字节上限约束；`license` 必须是 ASCII。
- 链接字段按字段区分 query 策略：release URL 允许 query，`homepage` 与 `head.repository` 拒绝；
  三者都要求 HTTPS、无 userinfo、无 fragment，并在校验前解包一次 Markdown 链接包装。
- 结构性失败携带出错值的点路径；语义失败携带渲染为同形状点路径的结构化字段。
- 同一份存在多处错误的清单在任何机器上都报同一个首个字段。

### 禁止结果

- 自动 trim、截断或替换非法文本，而不是拒绝。
- 接受 `http://`、带用户名密码或带 fragment 的清单 URL。
- 把 `%252F` 二次解码后用于 `path_prefix` 匹配。
- 只报告一句无字段归属的错误消息，或让首个错误随字段遍历顺序变化。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| 完整清单与仅必填清单都产生确定的域对象 | `ora_plugin_manifest::tests::parses_complete_manifest_into_full_domain_object`; `parses_manifest_without_optional_fields`; `normalizes_empty_dependencies_table`; `parses_only_the_ora_dependency`; `parses_full_semantic_version_syntax` | `Covered` |
| 文本策略（空、首尾空白、控制字符、字节上限、ASCII license）生效 | `ora_plugin_manifest::tests::rejects_empty_required_strings`; `rejects_description_outer_whitespace`; `accepts_unicode_description_with_internal_spaces`; `enforces_text_byte_limits`; `rejects_invalid_license_text` | `Covered` |
| URL 策略按字段区分 query，并统一拒绝非 HTTPS、凭据与 fragment | `ora_plugin_manifest::urls::tests::applies_field_specific_query_policies`; `rejects_common_url_policy_violations`; `unwraps_markdown_linked_urls`; `enforces_url_byte_limit`; `ora_plugin_manifest::tests::parses_full_new_marketplace_manifest_with_markdown_urls` | `Covered` |
| digest 与分支名等标量只能通过不变量构造进入域模型 | `ora_plugin_manifest::sha256::tests::canonicalizes_digest_display`; `rejects_invalid_digest_text`; `ora_plugin_manifest::tests::preserves_git_branch_error_in_head_field`; `rejects_incomplete_head_but_ignores_unknown_head_fields` | `Covered` |
| 结构性与语义失败都携带稳定 field path | `ora_plugin_manifest::tests::reports_structural_toml_errors_with_spans`; `reports_structural_webview_errors_with_paths`; `formats_structured_manifest_fields`; `rejects_invalid_webview_fields_with_index`; `reports_an_action_without_a_known_key_as_ambiguous` | `Covered` |
| 多处错误时首个错误确定 | `ora_plugin_manifest::tests::returns_first_root_field_error_deterministically` | `Covered` |
| `path_prefix` 与页面路径只解码一次 | `PathPrefix` 与匹配共用同一解码函数，且 webview 解析测试覆盖非法 percent 转义；尚无针对 `%252F` 不被二次解码的专属行为断言 | `Partial` |
| 清单层 field path 完整传递到 discovery issue | `ora_plugin_manager::kind_tests::rejects_webview_cross_value_violations_with_field_paths` 证明跨值路径；清单层字段路径的端到端传递尚无专属断言 | `Partial` |

### 决策依据

- 清单根决策 D6：会被当作能力使用的标量在解析期收窄为不变量类型。
- 清单根决策 D7：清单只做单值语义校验，错误必须指向稳定 field path 且顺序确定。
- 清单根决策不变量 1、8。

## Plugin Kind Must Pair Exactly With the Sections It Owns

### 风险

`kind` 决定运行时权限、生命周期与允许携带的 section。如果一个包能同时携带两种 kind 的 section、
webview 能缺失其信任声明，或未知 kind 被降级执行，宿主就会在一个未定义的权限模型下运行作者的
代码，或对一个没有导航信任集合的页面放行。

### 前置状态

- 清单声明六种 kind 之一，或一个未知拼写。
- 清单可能携带与该 kind 不匹配的 `[workbench]` 或 `[webview]` section，或对 webview 缺失
  `[webview]`。

### 触发

`ora-plugin-manifest` 在语义校验阶段把 `kind` 与已出现的 section 配对。

### 必须成立

- `[webview]` 对 `webview` kind 必需且独占，缺失即整份拒绝。
- `[workbench]` 对 `workbench` kind 独占但可选；空的 `methods` 数组非法。
- `skill`、`mcp`、`hook`、`agent` 携带 `[workbench]` 或 `[webview]` 时整份拒绝。
- 方法名逐项校验，拒绝重复项与保留的 `ora/` 前缀。
- 未知 `kind` 拼写不被接受。

### 禁止结果

- 忽略与 kind 不符的 section 后按剩余字段安装。
- 把缺失 `[webview]` 的 webview 包按某个默认导航策略放行。
- 用空 allowlist 表达“没有方法”，或让空数组与省略 section 语义等价。
- 让清单把 `ora/` 前缀的宿主协议方法暴露给插件页面。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| `[webview]` 与 webview kind 必需且独占地配对 | `ora_plugin_manifest::tests::pairs_webview_section_with_kind`; `parses_webview_section_into_download_policy`; `webview_downloads_default_to_reject` | `Covered` |
| `[workbench]` 独占但可选，空 `methods` 非法 | `ora_plugin_manifest::tests::workbench_section_is_optional_and_kind_exclusive`; `parses_workbench_section_into_method_list`; `rejects_invalid_and_duplicate_workbench_methods` | `Covered` |
| 无进程/页面的 kind 拒绝这两个 section | `ora_plugin_manifest::tests::skill_kind_rejects_workbench_and_webview_sections`; `mcp_kind_rejects_workbench_and_webview_sections`; `hook_kind_rejects_workbench_and_webview_sections` | `Covered` |
| 方法名拒绝保留前缀、非法字符与重复项 | `ora_plugin_manifest::tests::rejects_invalid_and_duplicate_workbench_methods` | `Covered` |
| 每种 kind 的合法清单都能解析 | `ora_plugin_manifest::tests::parses_agent_kind_manifest`; `parses_skill_kind_marketplace_manifest`; `parses_mcp_kind_marketplace_manifest`; `parses_targeted_hook_manifest` | `Covered` |
| 未知 `kind` 拼写被拒绝 | `ora_plugin_manifest::tests::returns_first_root_field_error_deterministically` 中以 `kind = "gadget"` 作为第二个错误出现；`rejects_empty_required_strings` 覆盖空值，尚无针对未知拼写归属 `kind` 字段的专属断言 | `Partial` |

### 决策依据

- 清单根决策 D4：`kind` 是封闭集合并与独占 section 严格配对。
- 清单根决策不变量 4。

## Release and Installed Forms Must Refuse Each Other's Release Declarations

### 风险

release 清单描述“可以下载什么”，installed 清单描述“已经落盘的是什么”。如果两种形态互相接受对方
的发布 section，包内清单就能夹带下载地址，或一个 marketplace 条目能自述它的物理 target；两者都
会让宿主在错误的一侧判定主机兼容，把打包错误变成运行时崩溃或错误资产的静默下载。

### 前置状态

- 一份 release 清单，可能声明 universal 资产、`[[targets]]`、两者、或都不声明。
- 一份 installed 清单，可能声明 `[artifact]`、`[[targets]]`，或都不声明。
- kind 可能属于、也可能不属于允许携带原生二进制的集合。

### 触发

marketplace 索引构建解析 release 清单；本地导入与包发现解析 installed 清单。

### 必须成立

- universal 资产与 `[[targets]]` 互斥；universal 形态的 `url` 与 `sha256` 必须同时出现。
- `[[targets]]` 中 `target` 不得重复，且每项必须同时含 `target`、`url`、`sha256`。
- `[[targets]]` 与 `[artifact]` 只允许 `hook` 与 `agent`。
- release 形态拒绝 `[artifact]`；installed 形态拒绝 `[[targets]]`。
- installed hook 必须声明 `[artifact].target`；installed agent 可以省略。
- 两种发布形态都缺省是合法的：条目仍可被索引，只是本机不可安装。
- target triple 必须精确匹配白名单，未知拼写不被接受。

### 禁止结果

- 同一份 release 清单同时携带两种发布来源并由某条优先级规则择一。
- 包内清单携带下载地址，或 release 清单自述物理 target。
- 让非原生二进制 kind 声明 target，从而在 UI 上产生一个必然失败的安装入口。
- 把未知或相近的 triple 当作“另一种可安装架构”。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| universal 与 targeted 互斥且各自完整 | `ora_plugin_manifest::tests::parses_universal_hook_manifest`; `parses_targeted_hook_manifest`; `rejects_both_universal_and_targeted_release_sources`; `rejects_duplicate_target_triples` | `Covered` |
| target 形态只属于可能携带原生二进制的 kind | `ora_plugin_manifest::tests::parses_targeted_release_for_the_agent_kind`; `rejects_targeted_release_for_kinds_without_native_binaries`; `rejects_artifact_section_for_kinds_without_native_binaries` | `Covered` |
| 两种形态互相拒绝对方的发布 section | `ora_plugin_manifest::tests::rejects_artifact_section_on_release_manifest`; `rejects_targets_section_on_installed_manifest` | `Covered` |
| installed hook 必须自述 target，agent 可省略 | `ora_plugin_manifest::tests::rejects_installed_hook_without_artifact`; `parses_installed_hook_manifest_with_artifact`; `parses_installed_agent_manifest_with_artifact`; `parses_installed_agent_manifest_without_artifact` | `Covered` |
| target triple 精确匹配白名单 | `ora_plugin_manifest::tests::rejects_unsupported_target_triples`; `ora_plugin_manifest::target::tests::rejects_invalid_target_triples`; `parses_the_windows_x86_64_msvc_triple` | `Covered` |
| 无发布来源的清单仍然合法 | `ora_plugin_manifest::tests::parses_new_marketplace_schema_without_download_fields`; `parses_installed_manifest_without_download_fields` | `Covered` |
| installed 清单的顶层 `url`/`sha256` 不构成可下载 release | 缺省路径已由 `parses_installed_manifest_without_download_fields` 覆盖；带 `url`/`sha256` 的 installed 清单发布来源为空尚无专属断言 | `Partial` |

### 决策依据

- 清单根决策 D5：发布来源是互斥枚举，installed 形态只自述 `[artifact].target`。
- 清单根决策不变量 6、7。

## Schema Evolution Must Stay Compatible Within One Resolver Version

### 风险

同一个 marketplace 源仓库同时被新旧宿主扫描。如果未知字段导致整条 listing 被丢弃，增加一个可选
展示字段就会让旧版本 Ora 的插件列表凭空少掉条目；反过来，如果放宽未知字段的同时也放过了缺失的
必填字段，一份残缺清单会被当作有效插件安装。

### 前置状态

- 清单可能携带当前 schema 未定义的字段，出现在顶层或嵌套 section。
- 清单可能缺失必填字段、字段类型不符，或声明一个不受支持的 `resolver`。
- installed 清单可能完全省略 `resolver`。

### 触发

索引构建解析 release 清单，包发现解析 installed 清单。

### 必须成立

- 未知字段在两种形态、任意层级都被忽略，不改变解析结果。
- 必填字段缺失或类型不符始终是硬错误，放宽未知字段不得连带放宽这一条。
- 不受支持的 `resolver` 在任何字段语义校验之前就整份拒绝，错误只指向 `resolver`。
- installed 形态省略 `resolver` 被解释为当前版本。
- 新旧 marketplace 写法（有无 `title`、有无下载字段）都能解析。

### 禁止结果

- 因一个新的可选字段而跳过整条 listing。
- 用缺省值补齐缺失的必填字段。
- 在报告 `resolver` 不受支持之前先报告一串字段错误。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| 未知字段在两种形态被忽略 | `ora_plugin_manifest::tests::ignores_unknown_manifest_fields_in_both_forms`; `rejects_incomplete_head_but_ignores_unknown_head_fields` | `Covered` |
| 放宽未知字段后仍拒绝缺失或错类型的必填字段 | `ora_plugin_manifest::tests::still_rejects_missing_required_fields_after_relaxing_unknown_ones`; `rejects_missing_and_mistyped_required_fields` | `Covered` |
| 不受支持的 `resolver` 先于字段判定 | `ora_plugin_manifest::tests::rejects_unsupported_resolver_before_fields` | `Covered` |
| installed 形态省略 `resolver` 取当前版本 | `ora_plugin_manifest::tests::parses_installed_skill_manifest_without_resolver_or_download_fields`; `parses_new_installed_schema_with_title` | `Covered` |
| 新旧 marketplace 写法共存 | `ora_plugin_manifest::tests::parses_new_marketplace_schema_without_download_fields`; `parses_full_new_marketplace_manifest_with_markdown_urls` | `Covered` |
| 清单文本本身的大小与编码边界 | `ora_plugin_manager::tests::rejects_non_utf8_manifest`; `rejects_oversized_manifest` | `Covered` |

### 决策依据

- 清单根决策 D1：`resolver` 承担断裂性变化，且先于字段判定。
- 清单根决策 D2：未知字段被忽略，缺失必填字段仍是硬错误。
- 清单根决策不变量 2、3。
