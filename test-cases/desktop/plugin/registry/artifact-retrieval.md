# 插件包获取核心测试用例

## Existing Sources Migrate to Direct HTTPS Without Changing Their Identity

### 风险

新增 S3 能力若要求旧行补齐一组配置，或在迁移时改写 source URL、namespace 和本地路径，升级后的
Ora 会无法启动、无法识别既有安装，或把公开市场误判成私有对象存储。

### 前置状态

- SQLite 中存在不含 `artifact_retrieval` 的 Marketplace Source 行。
- 该 source 已拥有稳定 URL、顺序和 namespace 绑定。

### 触发

应用数据库迁移并重新读取 Marketplace Source。

### 必须成立

- 旧行得到完整的 `{"type":"direct_https"}` 默认配置。
- JSON 必须是对象且存在 discriminator；数据库允许未来新增形态，当前 repository 仍须拒绝无法解码
  的形态且不得静默回退。
- 迁移不修改 source URL、namespace 绑定、检出目录或安装目录。

### 禁止结果

- 旧行需要伪造 S3 字段才能启动。
- 非法或当前版本未知的 JSON 被当成 Direct HTTPS。
- 获取方式参与 namespace 或文件系统路径派生。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
|---|---|---|
| 旧行取得 Direct HTTPS 默认值；数据库拒绝缺少 discriminator 的对象但允许未来 tagged variant | `ora_db::tests::marketplace_artifact_retrieval_migration_has_a_safe_default` | `Covered` |
| 迁移前后真实 source 绑定、检出和已安装插件均保持可用 | 尚无跨版本文件数据库与插件目录的端到端升级测试 | `Missing` |

### 决策依据

- Source 级插件包获取决策 D1、D2 与不变量 1、2。

## Retrieval Mode and Release Locator Must Form One Valid Download Source

### 风险

如果清单的对象 key 在没有 S3 配置时仍进入网络层，或 S3 模式把任意 HTTPS URL 当作普通下载，
配置的 endpoint/bucket 就不再是可信边界。反过来，如果 locator 解析器读取全局配置，清单域模型会
依赖 I/O，并可能在不同调用点产生不同解释。

### 前置状态

- release `url` 是绝对 HTTPS URL 或无 scheme 的对象 key。
- 所属 source 配置为 Direct HTTPS 或 S3 SigV4。

### 触发

解析 release 清单、选择当前主机的 release，并把 locator 转为下载请求。

### 必须成立

- 解析器纯粹区分 `Https` 与 `ObjectKey`，并拒绝非 HTTPS scheme、绝对路径、控制字符和 `..` 段。
- HTTPS locator 转为普通 URL 下载源；对象 key 转为 S3 下载源。
- Direct HTTPS 遇到对象 key 时在请求前失败。
- S3 SigV4 只接受对象 key，或精确匹配自身 endpoint/bucket 的 path-style HTTPS URL。

### 禁止结果

- Direct HTTPS 猜测 endpoint 或 bucket。
- S3 SigV4 对其它 host、端口或 bucket 的 URL 无签名直连。
- release locator 参与插件身份或 target 选择。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
|---|---|---|
| HTTPS 与对象 key 被解析为互斥 locator，非法 scheme/key 被拒绝 | `ora_plugin_manifest::locator::tests::{parses_https_urls_as_https_locators,parses_object_keys,rejects_non_https_schemes,rejects_invalid_object_keys}` | `Covered` |
| release 选择把两种 locator 映射到相应下载源 | `ora_plugin_manager::install::tests::{select_release_maps_an_object_key_to_s3,select_release_maps_https_to_a_url}` | `Covered` |
| 无 S3 配置的对象 key 与 S3 模式下的外部 HTTPS URL 均在下载前失败 | `ora_utils::http::tests::reqwest_integration::{rejects_s3_object_key_without_config,rejects_foreign_https_url_in_s3_mode}` | `Covered` |

### 决策依据

- Source 级插件包获取决策 D4 与不变量 3、5、6、7。

## S3 SigV4 Requests Stay Source-Scoped and Keep Integrity Verification

### 风险

全局 S3 配置会让一个私有 source 的凭据或 bucket 被用于另一个 source。签名 URL 与实际请求 URL
不一致会导致认证失败；把 SigV4 误当成包完整性证明，则可能让未通过清单摘要的内容进入安装目录。

### 前置状态

- 一个 source 拥有完整、已验证的 endpoint、bucket、region 与静态凭据。
- release locator 指向该 bucket 内的对象，并声明 SHA-256。

### 触发

从该 source 安装或更新插件。

### 必须成立

- endpoint、bucket 与对象 key 组合成唯一的 path-style HTTPS URL，签名和发送使用同一 URL。
- 请求携带 SigV4 所需 header，仍沿用该 source 的代理配置和统一下载限制。
- 下载内容必须通过清单 SHA-256 后才可原子进入安装目录；失败不破坏已有安装。
- 安装/更新只使用解析出该插件 namespace 的 source 配置。

### 禁止结果

- 凭据跨 source 复用或由进程级硬编码默认值提供。
- 签名成功后跳过 SHA-256。
- 下载或校验失败覆盖现有产物。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
|---|---|---|
| 对象 key 与匹配的完整 URL 均发送带签名 header 的 HTTPS 请求 | `ora_utils::http::tests::reqwest_integration::{downloads_s3_object_key_with_signed_headers,downloads_path_style_bucket_https_url_with_signed_headers}` | `Covered` |
| SigV4 canonical request、credential scope 与密钥相关签名稳定 | `ora_utils::http::s3::sign::tests::{builds_the_canonical_get_request,signs_a_path_style_get,signature_depends_on_the_secret_key}` | `Covered` |
| 统一下载器在摘要失败时不安装产物 | `ora_plugin_manager::install::tests::rejects_checksum_mismatch_and_installs_nothing` | `Covered` |
| 后端从 namespace 定位 source 到真实 S3 请求、摘要校验与安装提交的跨 crate 流程 | 当前证据分别覆盖 source 配置、release 映射、签名下载和安装校验，尚无一条贯穿全链路的测试 | `Partial` |

### 决策依据

- Source 级插件包获取决策 D1、D5 与不变量 5、7、8。

## Source Editing Must Treat the S3 Credential Pair as Write-Only State

### 风险

若 source 查询回显 Secret Access Key，或用 nullable 字段同时表达保留、清空和替换，前端很容易在
普通编辑时清空凭据，也会让凭据扩散到状态缓存、日志、Debug 输出和截图。

### 前置状态

- 用户在“管理市场 → 编辑插件源”中选择 S3 签名获取。
- source 可能首次启用 S3，也可能已有一对持久化凭据。

### 触发

保存获取方式、编辑其它 source 字段，或读取 source 列表。

### 必须成立

- 选择 S3 后只展示 endpoint、bucket、region、Access Key ID 与 Secret Access Key，且首次启用时全部
  必填。
- 已配置 S3 source 的查询结果只返回非敏感配置；留空凭据输入明确提交 `preserve`。
- 任意一项新凭据出现时必须同时提交完整 pair，并以 `replace` 原子替换。
- 切回 Direct HTTPS 后持久化状态不再保留任何 S3 字段。
- 契约、持久化类型和下载配置的 Debug 输出均不渲染凭据。

### 禁止结果

- source 列表响应返回 AK/SK 或签名 header。
- 使用 `null` 或空字符串猜测用户是想保留还是清空。
- 编辑 branch、代理或 enabled 状态时意外替换、清空凭据。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
|---|---|---|
| 后端查询脱敏，保留动作不改写持久化凭据 pair | `ora_backend::marketplace_sources::tests::updates_and_preserves_s3_retrieval_credentials` | `Covered` |
| 首次启用必须提供完整凭据，持久化配置 Debug 脱敏 | `ora_backend::marketplace_sources::artifact_retrieval::tests::{enabling_s3_requires_complete_credentials,debug_redacts_credentials}` | `Covered` |
| 更新请求的序列化形态明确区分 preserve/replace，Debug 不包含凭据 | `ora_contracts::plugin::tests::serializes_marketplace_source_contracts` | `Covered` |
| 编辑界面按模式展示字段、提交完整配置且响应不回显凭据 | `packages/app-shell/src/features/settings/plugin-sources-manager.test.tsx` 中 `configures S3 SigV4 retrieval without returning credentials` | `Covered` |
| 从 S3 切回 Direct HTTPS 后数据库不保留旧配置或凭据 | `ora_backend::marketplace_sources::tests::updates_and_preserves_s3_retrieval_credentials` | `Covered` |

### 决策依据

- Source 级插件包获取决策 D3、D6、D7 与不变量 4、9、10。
