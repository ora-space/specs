# 安装期包内清单校验与失败清理核心测试用例

本文件是 [安装期包内清单校验 ADR](../../../../decisions/desktop/plugin/registry/20260922-install-time-package-validation.md)
的风险导航图，覆盖 marketplace 安装如何判定一个包能否落盘，以及安装结束后磁盘与缓存的状态。
清单字段规则本身由
[清单配套决策](../../../../decisions/desktop/plugin/manifest/0-versioned-dual-form-manifest.md)
拥有，本文件只记录安装期承诺与直接证据。

## Marketplace Install Must Validate the Package's Own Manifest Before Committing

### 风险

如果安装用 registry 清单校验解包结果，而不是用包内 `orax.toml`，那么包内清单缺失、TOML 语法
错误、缺少 `description` 等必填字段、或 kind 专属贡献（MCP 的 `assets/config.json`）不合格时，
包依然会“安装成功”并落盘。直到 discovery 重新读取包内清单时才失败，用户看到的是内部错误和
一个留在磁盘上的坏包。反过来，如果安装期校验比 discovery 更严或走另一套规则，合法包会被拒绝，
或者安装期通过而 discovery 拒绝，同样破坏“装上即可被发现”。

### 前置状态

- 一个 marketplace listing 已解析出 release 与 SHA-256，下载产物已通过摘要校验。
- 归档已解包到 `installed/<namespace>/<name>/` 下的 staging 临时目录，尚未提交。
- 包内 `orax.toml` 可能缺失、不是普通文件、TOML 非法、缺少必填字段，或其 kind 贡献不合法。

### 触发

调用安装路径（`Installer::install` / `install_with_progress`，以及 pack 成员的同一路径）完成一次
marketplace 安装。

### 必须成立

- 包内 `orax.toml` 被以 installed 形态解析，解析与宿主侧校验走与 discovery 相同的代码路径。
- 缺失清单、清单不是普通文件、清单超出大小预算、非法 UTF-8、TOML 语法错误、缺少必填字段、
  宿主侧字段校验失败都在提交之前失败，并携带稳定字段路径。
- 合法包照常提交，其目录内容与 staging 内容一致。
- 校验失败不产生版本目录，也不产生空的 `<namespace>/<name>/` 父目录。

### 禁止结果

- 用 registry 清单替代包内清单做校验。
- 包内清单缺失或非法时仍提交安装目录。
- 安装期通过的包在 discovery 阶段被拒绝（校验规则分叉）。
- 校验失败留下 staging 目录、部分文件或版本目录。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
|---|---|---|
| 合法包端到端安装成功，目录内容完整 | `ora_plugin_manager::install::tests::installs_local_release_end_to_end` | `Covered` |
| 包内缺少 `orax.toml` 时拒绝安装，且零残留 | `ora_plugin_manager::install::tests::rejects_a_release_without_an_in_package_manifest` | `Covered` |
| `orax.toml` 不是普通文件时以该字段拒绝安装 | `ora_plugin_manager::install::tests::rejects_a_manifest_entry_that_is_not_a_regular_file` | `Covered` |
| 缺少必填字段时以结构性清单错误拒绝，并归因到该字段 | `ora_plugin_manager::install::tests::rejects_a_package_manifest_missing_a_required_field` | `Covered` |
| TOML 语法错误时拒绝安装，且错误不归因到任何单个字段 | `ora_plugin_manager::install::tests::rejects_a_package_manifest_that_is_not_valid_toml` | `Covered` |
| kind 专属贡献不合法（MCP 配置）时以 `assets/config.json` 拒绝 | `ora_plugin_manager::install::tests::rejects_a_package_whose_kind_contribution_is_malformed` | `Covered` |
| 校验失败不产生版本目录、空父目录与缓存残留 | `ora_plugin_manager::install::tests::assert_no_install_residue`（由上述用例调用） | `Covered` |
| 同一失败经生产入口映射为可操作的公开错误 | `ora_backend::plugin::operations::install_tests::incomplete_package_manifest_is_reported_as_a_package_error` | `Covered` |

### 决策依据

- 安装期包内清单校验决策 D1、D3 与不变量 1、2、7。

## Package Identity Must Match the Listing Before It Is Committed

### 风险

包内清单的 `identifier`、`version`、`kind` 与 registry 清单不一致时，安装目录、插件私有数据目录、
`skills` 行会按其中一个身份建立，而 discovery 之后按包内清单读取。无论采信
哪一侧，都会把一份身份的数据挂到另一份身份上，或让一次安装的产物永远无法被发现。反过来，如果
把展示字段也纳入一致性判定，作者在包内更新 `title`、`description` 或图标就会被拒绝安装。

### 前置状态

- registry 清单声明了 `identifier`、`version`、`kind` 与下载地址。
- 包内清单合法且可解析，但其中至少一个身份字段与 registry 清单不同。

### 触发

完成解包后的身份一致性检查。

### 必须成立

- `identifier`、`version`、`kind` 三项逐一比对，任一不一致即拒绝安装并指出该字段。
- 展示字段（`title`、`description`、`homepage`、`license`、图标）不参与判定；装后以包内清单为准。
- 不一致时不提交任何版本目录，也不修改已安装的其他版本。

### 禁止结果

- 按包内清单静默改写安装目录或按 registry 清单静默改写包内身份。
- 身份不一致的包被提交后再由 discovery 报错。
- 展示字段差异导致安装失败。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
|---|---|---|
| `identifier`、`version`、`kind` 各自不一致都被拒绝并指出字段 | `ora_plugin_manager::install::tests::rejects_packages_that_contradict_the_listing_identity` | `Covered` |
| 版本不一致经生产入口报告为 `version` 字段错误且不落盘 | `ora_backend::plugin::operations::install_tests::package_identity_mismatch_is_reported_as_a_package_error` | `Covered` |
| 展示字段差异不阻止安装，discovery 读回包内文案 | `ora_plugin_manager::install::tests::installs_a_package_whose_descriptive_metadata_differs_from_the_listing` | `Covered` |

### 决策依据

- 安装期包内清单校验决策 D2 与不变量 4、5。

## Failed Installs Must Leave No Package or Cache Residue

### 风险

安装失败若留下版本目录、空的 `<namespace>/<name>/` 目录或下载的 `.orax`，磁盘会随着每次失败
持续增长。残留的缓存文件没有任何读取者，用户无法在 UI 中看到或清理；残留的空目录会让后续扫描
出现噪声。若清理范围过宽，又可能误删派生索引或正在下载的临时文件。

### 前置状态

- `plugins/cache/` 中可能存在历史遗留的 `*.orax`（含旧扁平布局写在缓存根下的文件）、
  `registry_index.json`，以及一个仍在下载中的 `<destination>.tmp`。
- 一次安装即将开始，其下载或校验可能失败。

### 触发

执行一次安装，并在下载、解包、校验的任一阶段失败。

### 必须成立

- 安装开始先清扫缓存中的 `*.orax`，包括旧布局写在缓存根下的历史残留。
- 成功与失败都删除本次下载的 `.orax`。
- 失败时 best-effort 删除空的 `<namespace>/<name>/` 父目录；非空目录保持不动。
- 清理只匹配 `*.orax`：`registry_index.json` 与下载中的 `.tmp` 文件不受影响。
- 已安装的其他版本不受影响。

### 禁止结果

- 失败后残留本次下载的 `.orax`。
- 删除 `registry_index.json` 或正在写入的临时文件。
- 为清理而删除其他插件、其他版本或其他 workspace 的文件。
- 清理失败使一次已经成功的安装被报告为失败。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
|---|---|---|
| 清扫删除历史残留（含旧扁平布局）并移除空 namespace 目录 | `ora_plugin_manager::install::tests::sweeps_leftover_archives_without_touching_the_registry_index` | `Covered` |
| 清扫不触碰派生索引与下载中的临时文件 | `ora_plugin_manager::install::tests::sweeps_leftover_archives_without_touching_the_registry_index` | `Covered` |
| 失败路径删除缓存文件并移除空父目录 | `ora_plugin_manager::install::tests::{rejects_a_release_without_an_in_package_manifest,rejects_packages_that_contradict_the_listing_identity}` | `Covered` |
| 成功后缓存只剩派生索引 | `ora_plugin_manager::install::tests::installs_local_release_end_to_end` | `Covered` |
| 失败不影响已安装版本与其他成员 | `ora_backend::plugin::operations::install_tests::missing_artifact_aborts_install_without_creating_directory` | `Covered` |

### 决策依据

- 安装期包内清单校验决策 D4、D7 与不变量 2、3。

## Same Name and Version From Different Sources Must Not Share a Cache Path

### 风险

不同 marketplace source 可以发布同名同版本的插件。若缓存文件名由 `<name>-<version>` 扁平拼接，
namespace 与插件名又都允许连字符，两个来源可能落到同一路径：并发安装时，后写入的字节会覆盖前
一个安装即将解包的归档，使一个安装解出另一个源的包——这是静默的错误内容落盘，而不是可见失败。

### 前置状态

- 两个 source 各自发布 `identifier` 与 `version` 相同的插件。
- 两个安装可能并发进行，且都会写入 `plugins/cache/`。

### 触发

构造下载产物路径并执行安装。

### 必须成立

- 缓存路径为 `<cache>/<namespace>/<name>-<version>.orax`，namespace 是独立目录层。
- namespace 或插件名含连字符时，不同来源仍得到不同路径。
- 路径始终位于 `plugins/cache/` 之内（namespace 是已验证的单个路径段）。

### 禁止结果

- 两个来源映射到同一缓存路径。
- 缓存路径逃出缓存目录。
- 为区分来源而在文件名中混入未验证的原始字符串。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
|---|---|---|
| 连字符歧义下的两个来源得到不同缓存路径 | `ora_plugin_manager::install::tests::keeps_same_name_and_version_releases_from_different_namespaces_apart` | `Covered` |
| 并发安装互不覆盖字节 | 尚无真实并发安装用例（当前仅证明路径唯一）。每次安装开始的全量清扫可能删除并发安装的在途归档，真实并发用例的结果因此不确定，需先有可控的调度注入点 | `Partial` |

### 决策依据

- 安装期包内清单校验决策 D5 与不变量 8。

## Package Content Failures Must Surface as Actionable Field Errors

### 风险

包内容失败若统一映射为 `internal_error`，前端只能显示通用文案与 requestId。用户无法知道是缺字段、
TOML 错误、身份不一致还是摘要不匹配，也就无法修正包或向作者反馈。反过来，如果把传输失败、宿主
I/O 失败也塞进包内容错误，用户会被引导去修一个并不存在的包问题。

### 前置状态

- 安装或本地导入因包内容失败：缺失清单、清单语法或字段错误、宿主侧校验失败、身份不一致或摘要
  不匹配。
- 前端通过 `localizeContractError` 展示远端错误。

### 触发

后端把安装错误映射为公开契约错误，前端按错误码取文案并插值。

### 必须成立

- 包内容失败映射为 `plugin_package_invalid`，参数 `{field, message}`，分类 `Unprocessable`。
- `field` 取稳定字段路径：`description`、`assets/config.json`、`identifier`/`version`/`kind`、
  `orax.toml`、`sha256` 等。
- 中英文文案都存在并插入 `field` 与 `message`，未知错误码仍回退到通用文案。
- 传输失败、宿主 I/O 失败与解包失败保持 `internal_error`。

### 禁止结果

- 包内容失败只显示 requestId。
- 传输或宿主故障被报告为包内容错误。
- 契约新增错误码后前端缺少文案（覆盖率检查失败）。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
|---|---|---|
| 缺字段经生产入口报告为 `plugin_package_invalid` 且分类为 `Unprocessable` | `ora_backend::plugin::operations::install_tests::incomplete_package_manifest_is_reported_as_a_package_error` | `Covered` |
| 身份不一致经生产入口报告为对应字段错误 | `ora_backend::plugin::operations::install_tests::package_identity_mismatch_is_reported_as_a_package_error` | `Covered` |
| TOML 语法错误经生产入口报告为 `orax.toml` 字段错误 | `ora_backend::plugin::operations::install_tests::malformed_package_manifest_is_reported_against_the_manifest_file` | `Covered` |
| 清单解析失败归因到用户需修改的字段（根级与嵌套缺字段、语义错误），语法错误不归因 | `ora_plugin_manifest::tests::attributes_parse_failures_to_the_field_to_fix` | `Covered` |
| 本地导入与更新复用同一映射 | `ora_backend::plugin::operations::install_tests::{importing_an_invalid_package_is_reported_as_a_package_error,updating_to_an_invalid_package_keeps_the_installed_version}` | `Covered` |
| 摘要不匹配在 pack 结果中报告为该错误码 | `ora_backend::plugin::operations::install_tests::sha256_mismatch_aborts_install_without_phantom_ownership` | `Covered` |
| 契约码与 serde 标签、样本清单一致 | `ora_contracts::error::tests::public_error_codes_match_serde_tags_for_every_variant` | `Covered` |
| 每个公开错误码在两种语言都有文案且插值字段与参数一致 | `@ora/app-shell` `contract error translations > covers every generated public code in Chinese and English with valid interpolation` | `Covered` |
| 前端渲染字段与原因而非 requestId | `@ora/app-shell` `plugin package failures > renders the rejected field and reason instead of a bare request ID` | `Covered` |

### 决策依据

- 安装期包内清单校验决策 D6 与不变量 6。
