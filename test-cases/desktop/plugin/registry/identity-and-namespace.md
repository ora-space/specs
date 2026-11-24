# Namespace 派生与身份边界

## Namespace Derivation From a Canonical Source URL Is a Frozen Compatibility Contract

### 风险

`derive_from_canonical_url` 的结果一旦写入某个已安装插件的目录名，就会被冻结进
`installed/<namespace>/<identifier>/<version>/`、插件私有数据目录与 `skills` 表。如果规范化规则、slug 折叠规则或摘要长度在未来实现中漂移，同一 source URL
会在不同版本或不同平台下派生出不同 namespace，导致同一份已安装数据在重启或升级后找不到自己的
身份。规范化把大小写、默认端口、尾部 `/`、尾部 `.git` 全部折叠掉，任何一步遗漏都会让两个本应
等价的写法产生两个身份；反之，折叠力度过大也可能让本应不同的仓库共享一个 namespace。

### 前置状态

- 一个 marketplace source 的 URL 以任意大小写、端口、尾部斜杠或 `.git` 后缀书写。
- 派生函数尚未对该 URL 求值。

### 触发

对该 URL 的 canonical 形式调用 `PluginNamespace::derive_from_canonical_url`。

### 必须成立

- 同一 canonical URL 在任意调用中派生出完全相同的 namespace 字符串。
- 等价拼写（大小写、显式默认端口、尾部斜杠、尾部 `.git`、userinfo）先归一为同一 canonical URL，
  再派生同一 namespace。
- 仅在被 slug 丢弃的部分（host、端口、组织、子组层级）不同的两个仓库，必须派生出不同 namespace。
- 派生结果始终是合法、有界的单一路径段，且长度不超过 `MAX_PLUGIN_NAMESPACE_BYTES`（33 字节）。
- 派生结果永远不等于保留字 `official` 或 `local`，即使仓库名字面上就是这两个词。
- 仓库名不含任何 `[a-z0-9-]` 字符时，slug 回退为固定占位值而不是空字符串。

### 禁止结果

- 同一 canonical URL 在不同调用、不同平台之间派生出不同 namespace。
- 因两个仓库的组织、host 或子组路径不同，但仓库名相同，而错误地共享一个 namespace。
- 派生结果超出长度上界，或包含 `.` `..` 之外的非法字符。
- 派生结果与 `official` / `local` 相撞。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
|---|---|---|
| ADR 文档示例摘要值是被固定的字面量，不随实现漂移 | `ora_domain::plugin_namespace::tests::derives_the_documented_namespace` | `Covered` |
| 等价 URL 拼写派生同一 namespace；仅在 slug 丢弃部分不同的仓库派生不同 namespace | `ora_domain::plugin_namespace::tests::derives_per_canonical_url_not_per_spelling` | `Covered` |
| 任意仓库名（含深层子组、平台字面量中缀、超长名、非 ASCII）派生结果都是合法且有界的单一路径段 | `ora_domain::plugin_namespace::tests::always_derives_a_legal_bounded_segment` | `Covered` |
| slug 取自仓库名最后一段并按规则折叠、截断，无可读片段时回退占位值 | `ora_domain::plugin_namespace::tests::derives_the_readable_slug_from_the_repository_segment` | `Covered` |
| 派生结果永远不等于保留字，即使仓库名字面上就是 `official`/`local` | `ora_domain::plugin_namespace::tests::never_derives_a_reserved_namespace` | `Covered` |
| 非法拼写（空、大写、`/`、`.`/`..`、超长）在解析时被拒绝并携带确定的错误变体 | `ora_domain::plugin_namespace::tests::rejects_namespaces_that_are_not_legal_path_segments` | `Covered` |

### 决策依据

- 来源派生 Namespace 决策 D2：规范化规则与摘要长度是兼容性契约，发布后不可修改。
- 来源派生 Namespace 决策 D4：`official`、`local` 是保留字，派生函数必须显式拒绝产出这两个值。
- 来源派生 Namespace 决策不变量 2、4、5。

## A Source's Namespace Binding Survives Deletion and Re-Adding Under an Equivalent URL

### 风险

派生只在 source 首次绑定时发生一次，此后必须只读。如果重新配置或删除后重新添加同一仓库时重新
现算 namespace 而不是复用历史绑定，已安装插件会与新计算出的 namespace 失配：安装目录、私有数据
目录与 `skills` 行全部按旧 namespace 索引，新 namespace 找不到任何一行，
表现为数据凭空消失或残留孤儿行永远清不掉。

### 前置状态

- 一个 marketplace source 已绑定 namespace（首次添加时派生或短路到 `official`）。
- 该 source 被删除，随后以同一仓库的等价 URL 拼写重新添加；或该仓库以不同拼写被第二次配置。

### 触发

`MarketplaceSourceStore` 对 canonical URL 执行绑定查找与写入。

### 必须成立

- 默认 marketplace 短路绑定到 `official`，不经过派生函数。
- 同一仓库的两种等价拼写只产生一次绑定：第二次添加复用已有 namespace，而不是派生新值或允许配置
  出第二个指向同一 canonical URL 的 source。
- 删除 source 后以等价 URL 重新添加，复用历史绑定的 namespace。

### 禁止结果

- 同一 canonical URL 在不同绑定动作之间产生不同 namespace。
- 同一仓库的两种拼写被配置成两条独立的 source 记录。
- 已安装插件在其 source 被删除又以等价 URL 重新添加后，因 namespace 改变而与自己的安装目录、
  `skills` 行失配。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
|---|---|---|
| 默认 marketplace 短路绑定到 `official`，不派生 | `ora_backend::marketplace_sources::tests::binds_the_reserved_namespace_only_to_the_default_source` | `Covered` |
| 删除 source 后以等价 URL 重新添加复用历史绑定的 namespace | `ora_backend::marketplace_sources::tests::reuses_one_binding_across_equivalent_urls_and_re_adds` | `Covered` |
| 同一仓库的第二种拼写不能被配置成第二条 source 记录 | `ora_backend::marketplace_sources::tests::rejects_a_second_spelling_of_a_configured_source` | `Covered` |
| 端到端场景：真实已安装插件（携带 `skills` 行）在其 source 删除并以等价 URL 重新添加、宿主重启后，namespace 与全部持久化行保持一致 | 尚无跨进程集成证据；上述测试只覆盖绑定表本身，未覆盖已安装插件的下游持久化行 | `Missing` |

### 决策依据

- 来源派生 Namespace 决策 D3：source 与 namespace 的绑定持久化且不可变。
- 来源派生 Namespace 决策「namespace 一旦落盘就不可改变」一节：source 配置变化下已安装插件最坏
  只能与来源脱钩，不能损坏持久化状态。
- 来源派生 Namespace 决策不变量 3、7。

## Installed Package Namespace Comes From the Install Directory, Never From the Manifest

### 风险

根决策曾规定「目录名不是身份，清单才是身份」；namespace 移出清单后这条不再成立，因为包内清单由
插件作者书写，作者既不知道也不应该知道用户从哪个源安装。如果 discovery 转而信任清单中可能残留
的 `namespace` 字段，或对宿主自己没有写过的目录结构做出猜测，插件就可能被错误地归属到作者声明
的任意 namespace，绕开「namespace 由宿主生成」这条不变量。

### 前置状态

- `installed/` 下存在一个由宿主写入的 `<namespace>/<identifier>/<version>/` 目录。
- 目录内清单的 `identifier`、`version` 可能与目录名一致，也可能因损坏而不一致；namespace 段本身
  只应由目录层级决定。

### 触发

`PluginManager::discover` 递归扫描 `installed/` 并逐个校验目录与清单。

### 必须成立

- 每个包的 namespace 完全取自其所在的 `<namespace>` 目录层级，按 `PluginId` 语法校验。
- 清单中的 `identifier`、`version` 必须与其所在目录名一致；不一致时报告 mismatch 并跳过该包，
  不猜测应归属哪一侧。
- 宿主没有写过的目录名（非法字符、超长、`.`/`..`）被报告为 `invalid_install_path` 并跳过，不被
  当作某种特殊 namespace 处理。

### 禁止结果

- 从清单读取 namespace 并以此决定包身份。
- identifier/version 与目录不一致时把该包归属到清单一侧的拼写。
- 把非法目录名解析成尽力而为的 namespace，而不是报告并跳过。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
|---|---|---|
| namespace 完全取自安装目录层级，不读取清单 | `ora_plugin_manager::tests::takes_namespace_from_the_install_directory_not_the_manifest` | `Covered` |
| 清单 identifier 与所在目录不一致时报告 mismatch 并跳过 | `ora_plugin_manager::tests::reports_identifier_that_differs_from_directory` | `Covered` |
| 不符合 `PluginId` 语法的目录名被拒绝为 `invalid_install_path`，不产生猜测出的 namespace | `ora_plugin_manager::tests::rejects_namespace_directories_outside_the_id_grammar` | `Covered` |

### 决策依据

- 来源派生 Namespace 决策 D5：identifier、version、kind 由包内清单决定；namespace 由安装目录
  决定。
- 来源派生 Namespace 决策不变量 1、6。

## Same-Identifier Entries From Different Sources Must Coexist Without Shadowing

### 风险

在本决策之前，跨源同 `identifier` 按源顺序去重，先出现的源静默遮蔽其余条目，用户在市场里看不出
条目来自哪个仓库，安装与更新都会跟随遮蔽者。namespace 与来源绑定后，两条同 `identifier` 的条目
天然是不同 id；如果索引构建、去重或安装解析的实现仍按旧的「先到先得」语义合并或遮蔽，冲突消失
的承诺就没有兑现，第三方源依旧可以静默覆盖官方条目。

### 前置状态

- 两个不同的 marketplace source 各自的 `registry/` 下都发布了同一 `identifier` 的条目。
- 两个 source 都已同步。

### 触发

`RegistryIndex::build_all` 合并全部 source 的扫描结果；随后对其中一个 id 调用
`resolve_manifest_all`。

### 必须成立

- 索引条目按 `namespace/identifier` 排序，去重只发生在同一个 source 内部；两个 source 各自的
  条目都保留在合并结果中，互不遮蔽。
- 索引条目携带其归属 source 的 canonical URL，供展示层消歧。
- 解析某个 id 的完整清单时，只回到拥有该 id namespace 的那个 source 的检出，与 source 在配置列表
  中的顺序无关。

### 禁止结果

- 两个 source 发布同一 `identifier` 时，索引中只剩一条。
- 安装或更新解析到的 source 随 source 列表顺序变化而变化。
- 索引条目丢失来源信息，导致 UI 无法区分两张几乎相同的卡片。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
|---|---|---|
| 索引条目的 `namespace`/`id` 取自其归属 source，而非清单自述 | `ora_plugin_registry::index::tests::takes_identity_from_the_source_not_the_manifest` | `Covered` |
| 两个 source 发布同一 `identifier` 时，两条条目在合并索引中并存，互不遮蔽 | `ora_plugin_registry::index::tests::lists_same_identifier_from_two_sources_without_shadowing` | `Covered` |
| 解析 id 的清单只回到拥有该 id namespace 的 source，与 source 顺序无关 | `ora_plugin_registry::index::tests::resolves_manifest_from_the_source_that_owns_the_id` | `Covered` |
| 前端市场页对同 `identifier` 的两条条目分别渲染 `sourceUrl` 且用户可据此区分安装哪一个 | 尚无直接证据；已知契约字段存在，但前端渲染路径未被测试覆盖 | `Missing` |

### 决策依据

- 来源派生 Namespace 决策 D6：索引合并语义变化，消歧责任交给展示层。
- 来源派生 Namespace 决策不变量 9。
- 来源派生 Namespace 决策「为什么不是这些替代方案」：`index.toml` 自述方案会让第三方源冒充
  `official` 遮蔽官方条目，正是本用例要防止的回归。

## Manifest Schema Must Ignore Unknown Fields While Still Rejecting Missing Required Ones

### 风险

清单校验从 `deny_unknown_fields` 改为忽略未知字段，目的是让新版本 Ora 增加的可选字段不会使旧
版本因为一个字段而拒绝整条 listing。这个改动天然有一个必须守住的反面：放宽未知字段不能连带放宽
必填字段缺失的检测，否则结构性错误会从「立刻报错」退化为「静默接受一个不完整的清单」。

### 前置状态

- 一份 `orax.toml`（release 形态或 installed 形态）包含若干当前 schema 不认识的字段，散布在根级
  与 `[head]`、`[webview]` 等嵌套段中。
- 同一份或另一份清单缺少某个必填字段。

### 触发

`ora-plugin-manifest` 解析该清单。

### 必须成立

- 含未知字段（根级或嵌套段内）的清单被正常解析，未知字段被忽略，不产生错误或告警。
- 必填字段缺失时解析仍然失败，并携带确定的 TOML 路径。

### 禁止结果

- 未知字段导致整条清单被拒绝。
- 放宽未知字段后，必填字段缺失也被静默接受。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
|---|---|---|
| release 与 installed 两种清单形态中，根级与嵌套段的未知字段均被忽略而不报错 | `ora_plugin_manifest::tests::ignores_unknown_manifest_fields_in_both_forms` | `Covered` |
| 未知字段被放宽之后，必填字段缺失仍然是硬错误 | `ora_plugin_manifest::tests::still_rejects_missing_required_fields_after_relaxing_unknown_ones` | `Covered` |

### 决策依据

- 来源派生 Namespace 决策 D1：未知字段被忽略而不是被拒绝；断裂性变化仍由 `resolver` 版本承担。
- 来源派生 Namespace 决策不变量 8。
- 本用例只守住未知字段与必填字段这一对边界。`resolver` 判定顺序、两种清单形态与完整 schema
  演进义务见
  [Schema Evolution Must Stay Compatible Within One Resolver Version](../manifest/schema-and-validation.md#schema-evolution-must-stay-compatible-within-one-resolver-version)。
