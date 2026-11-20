# Node 显式 Session MCP

依据[显式 Session MCP 决策](../../../decisions/node/agent-runtime/20261114-explicit-session-mcp-in-agent-session-spec.md)（Node 侧 `implemented`，Controller 义务随使用方接入）。
Node 实现依据固定合并提交 [`98c0414f`](https://github.com/ora-space/desktop/commit/98c0414fdaf998ae7ede817f743e3431d94c81d7)（[desktop#773](https://github.com/ora-space/desktop/pull/773)）。
“所需直接证据”保留原验证目标（路径相对 desktop 仓库根目录），每个用例末尾列出固定源码的代表性直接证据和不能由它推导的缺口。
20 项验证义务中：13 项 `Covered`、7 项 `Partial`，无 `Missing`。只能由 Controller 验证的发送前镜像与按能力派发在 [Controller 侧用例](../../controller/session/session-mcp-selection.md)维护，本文不登记。
本文件只核对 `desktop#773`（`98c0414f`）交付的显式选择、Node 解析、能力门禁、Setting 值隔离、包租约与 Desktop 共享解析器不变；被后续[会话 MCP 刷新与宿主健康](../../../decisions/node/agent-runtime/20261116-session-mcp-refresh-and-host-health.md)修改的生命周期条款在[会话 MCP 刷新与宿主健康用例](session-mcp-lifecycle.md)维护。
[会话执行用例](session-execution.md)与[子代理投影用例](child-projection-and-diagnostics.md)的证据保持各自范围，不推导本文新义务。

## The Spec Selection Must Be Explicit Complete And Bounded

### 风险

省略与空选择被区别对待，Node 把缺失选择解释为自动发现或 `Automatic`；乱序、重复或非规范的 ID 与
版本被受理；超过上限的进程扇出或凭据驻留量被接受；非法选择被折叠成会话终态而不是帧校验失败。

### 前置状态

Controller 构造 `start_agent_session` 的 `AgentSessionSpec.mcp`；Node 以 `deny_unknown_fields` 解码
spec，旧 Node 未声明 `session_mcp` 能力。

### 触发

发送省略 `mcp`、空列表、乱序或重复的 `plugin_id`、非严格解析的 ID、`v1.2.0` 或非完整 semver、缺失
`settings`、17 个 Server、settings 紧凑 JSON 超过 64 KiB（键、标点与转义都计入）的选择；重发旧格式
spec。

### 必须成立

- 省略与显式空列表都表示空插件 MCP 集合，Node 从不自动发现、不退回 `Automatic`；空选择不读取无关
  插件。旧格式 spec 继续解析为空选择，空选择不被序列化，旧 `deny_unknown_fields` Node 仍接受。
- `plugin_id` 通过严格 `PluginId` 解析并按原始 ID 的 UTF-8 字节序严格递增、无重复；
  `plugin_version` 满足 canonical semver 往返；`settings` 必填，没有覆盖值时是空对象。
- 一次会话最多 16 个 Server；所有 `settings` map 的紧凑 JSON UTF-8 编码长度之和最多 64 KiB；两项
  上限独立于控制帧上限，超限是 spec 校验错误。
- 非法或超限选择是帧校验失败：Node 以协议错误关闭控制连接（继承 close-code ADR），不受理该执行、
  不启动 Agent。

### 禁止结果

- Node 端补排序、去重或改写非规范 ID；超限选择被受理；缺失 `settings` 被默认成空对象；空选择被
  当成错误或全部插件；省略被当成非空。
- 帧校验失败被折叠为 `agent_failed` 会话终态而不是关闭控制连接。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 空与省略等价、旧 spec 兼容 | `crates/node-protocol/tests/protocol/session_mcp.rs` 的 `session_mcp_round_trip_and_old_spec`：旧输入等于空选择、空选择不序列化、分帧往返保留显式 settings | Covered |
| 有序唯一规范、版本 canonical、上限 | 同文件 `session_mcp_rejects_ambiguous_or_unbounded_selection`：重复、乱序、非严格 ID、`v1.2.0`、17 个 Server、值/键/转义分别越过 64 KiB 的选择在编码与接收边界都被拒绝；恰好 16 个 Server 与恰好 64 KiB 被接受 | Partial |

### 代表性证据与剩余缺口

- [session_mcp_round_trip_and_old_spec](https://github.com/ora-space/desktop/blob/98c0414fdaf998ae7ede817f743e3431d94c81d7/crates/node-protocol/tests/protocol/session_mcp.rs#L23)：分帧往返保留 string/number/boolean 标量 `settings`，省略 `mcp` 与空列表 `[]` 相等，且空选择不序列化。另在 [resolves_exact_packages_and_secret_free_revision](https://github.com/ora-space/desktop/blob/98c0414fdaf998ae7ede817f743e3431d94c81d7/apps/ora-node/src/session/mcp/tests.rs#L87)（L141–161）验证空选择在无 `load`/`http` 能力时返回空集合；同一测试的 `with_selection(SessionMcpSelection::Automatic)` 不改变已解析的选择，但它断言的对象是非空 `setup`（L156–161），空选择不被放宽没有单独断言。
- [session_mcp_rejects_ambiguous_or_unbounded_selection](https://github.com/ora-space/desktop/blob/98c0414fdaf998ae7ede817f743e3431d94c81d7/crates/node-protocol/tests/protocol/session_mcp.rs#L49)：拒绝重复 ID、乱序 ID、非规范 ID（`Official/acme.mcp`）、`v1.2.0`、17 个 Server、单值/长键/转义超 64 KiB 与双项合计超 64 KiB，并接受恰好 16 个 Server 与恰好 64 KiB。
- 缺口（`Partial` — 有序唯一规范、版本 canonical、上限）：
  1. `session_mcp_rejects_ambiguous_or_unbounded_selection` 只调用 `encode_controller_frame`，未在同一测试直接调用 `decode_controller_frame` / `read_controller_message` 验证接收边界（生产解码在 [decode_payload](https://github.com/ora-space/desktop/blob/98c0414fdaf998ae7ede817f743e3431d94c81d7/crates/node-protocol/src/frame.rs#L140) 的 `L149` 共用 `message.validate()?`）。
  2. 缺少 `settings` 字段、非标量值或未知字段由 `serde` 类型定义与 `deny_unknown_fields` 约束，未在该测试单独构造对应负例断言。
  3. 没有向真实 Node 进程发送非法 `spec.mcp` 帧并断言控制连接以协议错误关闭、不受理且不启动 Agent 的独立集成测试（通用协议错误关连由 [node_closes_sessions_with_the_reason_code](https://github.com/ora-space/desktop/blob/98c0414fdaf998ae7ede817f743e3431d94c81d7/apps/ora-node/tests/standalone/repository_websocket.rs#L568) 覆盖，未针对 `spec.mcp` 重测）。

### 决策依据

D1、D4 失败表第一行，不变量 1、2；close-code ADR 的协议错误关闭。

## The Whole Snapshot Resolves From The Installed Package Or The Setup Fails

### 风险

Node 按非 spec 版本解析、跳过失败成员交付部分快照、把声明默认值或覆盖校验放宽；Controller 提供的
主机命令或 Node 本地路径进入 ACP setup；revision 混入 Setting 值或其摘要。

### 前置状态

Node plugin catalog 已按 `InstallPlugins` 精确版本安装所选 MCP 包，包内含 `assets/config.json`；
spec 携带完整冻结选择。

### 触发

启动带完整选择的会话；其中一项未安装精确版本、包无效、不是 MCP、stdio 命令越界或丢失；一项必填或
transport 引用的值缺失、覆盖未声明或类型错误、对无声明 settings 的包发送覆盖。

### 必须成立

- Node 只从自身 plugin catalog 取得每项的精确版本并持有使用租约；编译包内 `assets/config.json`，
  由共享纯解析器映射到 ACP；Node 的配置来源为 spec，没有 Desktop 的配置 store。
- 任何一项成员失败都拒绝整个快照，绝不跳过插件、绝不交付部分 MCP 集合；包、Setting 与 spec 错误
  在 Agent 启动前检查。
- Settings 使用声明默认值；显式覆盖必须已声明且类型正确；被 transport 引用而不存在的值（含可选但
  无默认且被省略）失败；不猜测值。
- 协议与 ACP setup 不携带 Node 本地路径或 Controller 提供的主机命令；stdio 命令必须是该版本包
  `assets/` 内的普通文件，发送前重新验证 containment。
- Server 名称为 canonical Plugin ID，输出按该 ID 排序；revision 只含身份、版本、配置 revision（冻结
  会话输入为 `0`）与 transport，不散列 Setting 值，也不用于不同会话之间的值相等判断。

### 禁止结果

- 部分快照被交付或失败成员被静默跳过；声明默认值未应用；未声明的覆盖被接受或被静默丢弃；
  revision 含值或值摘要；Controller 主机命令或 Node home 路径进入 ACP setup。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 全量解析或整体拒绝 | `apps/ora-node/src/session/mcp/tests.rs` 的 `rejects_missing_capabilities_and_missing_or_invalid_packages_and_settings`、`maps_optional_referenced_setting_missing_to_full_failure`（可选声明被 transport 引用而未提供时按 `mcp_setting_missing` 整体拒绝，失败携带 `setting_id` 与 transport）、`rejects_an_override_for_a_settings_less_package` | Partial |
| 精确版本与无值 revision | 同文件 `resolves_exact_packages_and_secret_free_revision`：按精确安装版本解析、租约计数、revision 不含值 | Covered |
| 真实 `session/new` 交付 | `apps/ora-node/tests/agent_session/mcp.rs` 的 `mcp_reaches_real_acp_session_new_without_persisting_or_logging_settings`：fixture 校验完整 `session/new` 帧的成员、顺序与默认值应用 | Partial |
| MCP 包经既有安装路径 | `apps/ora-node/tests/plugins/mcp.rs` 的 `installs_mcp_packages_through_the_existing_plan_path`：`kind = "mcp"` 走既有 plan 安装入口与 MCP 包验证 | Covered |

### 代表性证据与剩余缺口

- 全量解析或整体拒绝：
  - [rejects_missing_capabilities_and_missing_or_invalid_packages_and_settings](https://github.com/ora-space/desktop/blob/98c0414fdaf998ae7ede817f743e3431d94c81d7/apps/ora-node/src/session/mcp/tests.rs#L167)：双成员集合中任意成员缺失 `load`/`http` 能力或 `assets/server` 被删除即整体失败，同时覆盖缺必填 `token`、非法控制字符 `\n`、错误版本 `2.0.0` 与未安装包。
  - [maps_optional_referenced_setting_missing_to_full_failure](https://github.com/ora-space/desktop/blob/98c0414fdaf998ae7ede817f743e3431d94c81d7/apps/ora-node/src/session/mcp/tests.rs#L274) 与 [rejects_an_override_for_a_settings_less_package](https://github.com/ora-space/desktop/blob/98c0414fdaf998ae7ede817f743e3431d94c81d7/apps/ora-node/src/session/mcp/tests.rs#L312) 覆盖可选引用缺失与无声明 settings 的多余覆盖。
  - [mcp_preflight_failures_start_no_agent](https://github.com/ora-space/desktop/blob/98c0414fdaf998ae7ede817f743e3431d94c81d7/apps/ora-node/tests/agent_session/mcp.rs#L273) 在双绑定输入（一个失败、一个有效）下证明不启动 Agent、不发送任何 ACP 帧。
  - [explicit_mcp_settings_validate_types_required_values_and_defaults](https://github.com/ora-space/desktop/blob/98c0414fdaf998ae7ede817f743e3431d94c81d7/crates/plugin-config/src/mcp/resolve.rs#L255) 覆盖声明默认值、空白必填字符串、类型错误与未声明覆盖。
  - 缺口（`Partial` — 全量解析或整体拒绝）：在 `NodeSessionMcp::prepare` 入口没有专门构造非 MCP `kind` 包、损坏 `config.json`、stdio `command` 越出 `assets/` 或类型错误/未声明覆盖的直接负例；这些规则分别在底层 crate 测试中验证（[explicit_mcp_settings_validate_types_required_values_and_defaults](https://github.com/ora-space/desktop/blob/98c0414fdaf998ae7ede817f743e3431d94c81d7/crates/plugin-config/src/mcp/resolve.rs#L255)、[rejects_commands_outside_the_package_assets_directory](https://github.com/ora-space/desktop/blob/98c0414fdaf998ae7ede817f743e3431d94c81d7/crates/plugin-config/src/mcp/tests.rs#L435)、[rejects_mcp_package_entrypoint_and_config_shape_violations](https://github.com/ora-space/desktop/blob/98c0414fdaf998ae7ede817f743e3431d94c81d7/crates/plugin-manager/src/kind_tests.rs#L269)）。
- 精确版本与无值 revision：[resolves_exact_packages_and_secret_free_revision](https://github.com/ora-space/desktop/blob/98c0414fdaf998ae7ede817f743e3431d94c81d7/apps/ora-node/src/session/mcp/tests.rs#L87) 验证按精确 `1.0.0` 安装路径解析、应用声明默认值（`argument` → `default-arg`）、替换 `cwd`、按 canonical Plugin ID 排序、替换 Setting 值后 revision 仍相等（配置 revision 为 `0`）且 `Debug` 不含值；该单元测试的 `Catalog::lease` 是空实现，租约获取与释放由第五个用例的 fixture 假体 [`CountedLease`](https://github.com/ora-space/desktop/blob/98c0414fdaf998ae7ede817f743e3431d94c81d7/apps/ora-node/tests/agent_session/support.rs#L246) 计数（[mcp_reaches_real_acp_session_new_without_persisting_or_logging_settings](https://github.com/ora-space/desktop/blob/98c0414fdaf998ae7ede817f743e3431d94c81d7/apps/ora-node/tests/agent_session/mcp.rs#L119) L168、L184），不经过生产 catalog 的租约保护。
- 真实 `session/new` 交付：[mcp_reaches_real_acp_session_new_without_persisting_or_logging_settings](https://github.com/ora-space/desktop/blob/98c0414fdaf998ae7ede817f743e3431d94c81d7/apps/ora-node/tests/agent_session/mcp.rs#L119) 配合 echo-agent 的 [verify_mcp_frame](https://github.com/ora-space/desktop/blob/98c0414fdaf998ae7ede817f743e3431d94c81d7/apps/ora-node/fixtures/echo-agent/agent.rs#L509) 在 Shared 与 Separate 两种工作负载下在内存中核对完整 `session/new` `mcpServers`（`official/acme.stdio` 与 `official/acme.web` 顺序、替换后的 `cwd`、env、prefix、headers 与 view 内命令）。
  - 缺口（`Partial` — 真实 `session/new` 交付）：该集成测试在 `configure`（L58–61）显式覆盖了 `argument`（未省略以触发 `"default": "unused"`），因此声明默认值应用在单元测试 `resolves_exact_packages_and_secret_free_revision`（L98–120）与 `explicit_mcp_settings_validate_types_required_values_and_defaults` 证明，未在真实 `session/new` fixture 中省略字段重测。
- MCP 包经既有安装路径：[installs_mcp_packages_through_the_existing_plan_path](https://github.com/ora-space/desktop/blob/98c0414fdaf998ae7ede817f743e3431d94c81d7/apps/ora-node/tests/plugins/mcp.rs#L7) 验证无 `main.js` 的 `kind = "mcp"` `.orax` 包经 `PluginInstaller::execute` 安装并落盘 `assets/server`。

### 决策依据

D2、D5，不变量 2、4；继承 Desktop 的 ACP Session setup 交付与工作流显式选择。

## Setting Values Must Never Persist Or Reach Any Diagnostic

### 风险

Setting 值进入 ledger（含 SQLite WAL、空闲页与备份）、会话 JSONL、Thread 事件、日志、错误链或输入
对象的 `Debug`；重发改写正在使用的内存值；崩溃重启后 Agent 带旧值复活或要求 Controller 重放凭据。

### 前置状态

spec 的 `settings` 含敏感值；Node 使用真实 SQLite ledger；会话运行并产生记录与日志。

### 触发

受理并启动带值会话；重发同一 execution 携带不同值；受理后强杀 Node 再重开数据库；以 TRACE 级日志
运行；制造包、Setting 与能力失败。

### 必须成立

- Setting 值只通过 `start_agent_session` 控制帧进入 Node 内存，成功解析后进入 ACP `session/new`；
  不得进入 ledger（包括 SQLite WAL）、会话 JSONL、Thread、日志、错误链或 `Debug`；ACP 帧诊断对
  `session/new`/`session/load` 的 `mcpServers` 脱敏，不把 ACP 请求体写日志。
- 运行时值与持久输入在首次数据库写入前分开；持久投影只保留 MCP 身份、版本与空 Settings；身份、
  版本及其他原字段参与重复执行比较，Setting 值不参与；重发相同 execution（含不同值）返回既有记录，
  不替换正在使用的内存值、不重启；需要改变值必须创建新 execution。
- 受理后进程崩溃按原规则结算 `interrupted`，不从持久投影启动 Agent，不要求 Controller 重放凭据来
  恢复；历史空选择无需数据迁移。

### 禁止结果

- 任何值、值摘要或可逆编码出现在持久或诊断数据；重发改变运行中会话的值；重启后 Agent 复活或凭据
  被重放；先写入再删除式的“擦除”路径。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 账本无值 | `apps/ora-node/tests/session_ledger.rs` 的 `session_mcp_settings_never_enter_the_durable_ledger`：持久记录等于清空 settings 的投影；重发不同值返回同一记录，身份或版本冲突仍被拒绝；数据库目录全部文件字节扫描无值；重开后恢复投影无值 | Covered |
| 运行时无泄漏 | `apps/ora-node/tests/agent_session/mcp.rs` 的 `mcp_reaches_real_acp_session_new_without_persisting_or_logging_settings`：Shared 与 Separate 两种工作负载下，Node home 全文件递归字节扫描、待发 Thread 事件与记录的 TRACE 日志均无值 | Covered |
| 失败路径无泄漏 | 同文件 `mcp_capability_failures_stop_before_session_new`、`mcp_preflight_failures_start_no_agent`：终态 JSON、日志与文件扫描无值 | Covered |
| `Debug` 与帧诊断脱敏 | `crates/node-protocol/tests/protocol/session_mcp.rs` 的 `session_mcp_debug_redacts_settings`；`crates/acp/src/frame.rs` 的 `mcp_setup_trace_redacts_servers_without_mutating_the_frame`（`session/new` 与 `session/load` 都脱敏且不改原帧） | Covered |
| 崩溃不恢复 Agent | 带非空 MCP 输入的会话强杀后以 `interrupted` 收口、不重启 Agent 的直接断言；无 MCP 输入时的既有义务见[会话执行用例](session-execution.md#durable-session-ledger-preserves-events-commands-and-terminal-state)，不推导本文新义务 | Partial |

### 代表性证据与剩余缺口

- 账本无值：[session_mcp_settings_never_enter_the_durable_ledger](https://github.com/ora-space/desktop/blob/98c0414fdaf998ae7ede817f743e3431d94c81d7/apps/ora-node/tests/session_ledger.rs#L125) 通过真实 `NodeDatabase` 证明 `accept_session` 仅持久化清空 `settings` 的投影、以不同值重发同一 execution 返回原记录、修改 `plugin_version` 触发冲突拒绝、扫描数据库目录下全部文件不含新旧秘密、重开 `NodeDatabase` 后 `recoverable_sessions()` 读出空 `settings` 投影。（说明：运行中会话重发不改写内存值在 `98c0414f` 由 `NodeDatabase::accept_session` 去重边界证明，未另加对运行中进程重发同一 start 帧的独立 socket 测试。）
- 运行时无泄漏：[mcp_reaches_real_acp_session_new_without_persisting_or_logging_settings](https://github.com/ora-space/desktop/blob/98c0414fdaf998ae7ede817f743e3431d94c81d7/apps/ora-node/tests/agent_session/mcp.rs#L119) 在 Shared 与 Separate 两种工作负载下递归扫描 `fixture.home()` 全部文件（ledger、会话 JSONL 等）、待发 Thread 事件 JSON 及录制的 TRACE 日志（并确认 TRACE 日志包含 `"session/new"`），均不含 `SECRET`。
  - 范围：上述扫描证明 Node 自身写入的持久与诊断表面；Agent 自己写到 stderr、由 Node 作为插件日志持久化到 `plugin.log` 的内容不在 D3 范围内，按已知运行时值的尽力脱敏由 [circus#1294](https://github.com/ora-space/circus/issues/1294) 承接。
- 失败路径无泄漏：[mcp_capability_failures_stop_before_session_new](https://github.com/ora-space/desktop/blob/98c0414fdaf998ae7ede817f743e3431d94c81d7/apps/ora-node/tests/agent_session/mcp.rs#L206) 与 [mcp_preflight_failures_start_no_agent](https://github.com/ora-space/desktop/blob/98c0414fdaf998ae7ede817f743e3431d94c81d7/apps/ora-node/tests/agent_session/mcp.rs#L273) 证明终态 JSON、TRACE 日志与 `fixture.home()` 全文件扫描不含 `SECRET`；解析错误 `Debug` 脱敏另见 [rejects_missing_capabilities_and_missing_or_invalid_packages_and_settings](https://github.com/ora-space/desktop/blob/98c0414fdaf998ae7ede817f743e3431d94c81d7/apps/ora-node/src/session/mcp/tests.rs#L167)（L213）、[rejects_an_override_for_a_settings_less_package](https://github.com/ora-space/desktop/blob/98c0414fdaf998ae7ede817f743e3431d94c81d7/apps/ora-node/src/session/mcp/tests.rs#L312)（L344）及 [explicit_mcp_settings_validate_types_required_values_and_defaults](https://github.com/ora-space/desktop/blob/98c0414fdaf998ae7ede817f743e3431d94c81d7/crates/plugin-config/src/mcp/resolve.rs#L255)（L284）。
- `Debug` 与帧诊断脱敏：[session_mcp_debug_redacts_settings](https://github.com/ora-space/desktop/blob/98c0414fdaf998ae7ede817f743e3431d94c81d7/crates/node-protocol/tests/protocol/session_mcp.rs#L38) 证明整条 `StartAgentSession` 命令与 `SessionMcpBinding` 的 `Debug` 均脱敏；[mcp_setup_trace_redacts_servers_without_mutating_the_frame](https://github.com/ora-space/desktop/blob/98c0414fdaf998ae7ede817f743e3431d94c81d7/crates/acp/src/frame.rs#L169) 证明 `session/new` 与 `session/load` 的 `/params/mcpServers` 被脱敏为 `"[REDACTED]"` 且原帧不变。
- 缺口（`Partial` — 崩溃不恢复 Agent）：`session_mcp_settings_never_enter_the_durable_ledger`（L165–169）仅在数据库重开层证明带非空 MCP 的持久行可被 `recoverable_sessions()` 以空 `settings` 读出；真实进程 SIGKILL 后重启结算 `interrupted` 且不重启 Agent 的 standalone 测试（[killed_session_replays_and_ends_before_accepting_commands](https://github.com/ora-space/desktop/blob/98c0414fdaf998ae7ede817f743e3431d94c81d7/apps/ora-node/tests/standalone/agent_sessions/recovery.rs#L7)）走空 `mcp` 路径，未针对非空 `spec.mcp` 重测。（后续 `20261116` D5 已为所有会话重启结算增加原停止证据前提。）

### 决策依据

D3，不变量 3、7；协议后续决策“上传授权是唯一不持久化输入”的 MCP 例外。

## Capability Failures Stop Before Session New With Secret-Free Identifiers

### 风险

`session_mcp` 能力未声明仍收到非空选择；Agent 能力（`session/load`、HTTP MCP）在任何 ACP 帧之前被
断言，或缺失时仍发送 `session/new` 使值到达 Agent；失败终态或日志携带 Setting 值或原始错误文本。

### 前置状态

Node 在 `HelloAccepted.capabilities` 广告 `session_mcp`；echo agent 的 ACP `initialize` 响应可控地
声明或省略 `session/load` 与 HTTP MCP 能力。

### 触发

向未声明 `session_mcp` 的 Node 发送非空列表；非空选择且 Agent 未声明 `session/load`；HTTP 插件遇到
未声明 HTTP MCP 能力的 Agent；包、Setting 或 spec 校验失败。

### 必须成立

- `session_mcp` 只与 `AgentSession` 一起广告；它表示 Node 能解析，不保证所选 Agent 支持 MCP；
  Controller 只向声明该能力的 Node 发送非空列表。
- 包、Setting 与 spec 错误在 Agent 启动前检查（无 ACP `initialize`、无插件进程）；能力错误在
  `initialize` 之后、`session/new` 之前检查；任何解析失败都不发送 `session/new`，失败路径没有 MCP
  值到达 Agent；不以健康探测代替 Agent 能力协商。
- 四个失败 code 及语义：未安装精确版本、包无效、不是 MCP、stdio 命令越界或丢失为
  `mcp_plugin_missing`；必填或 transport 引用值缺失、覆盖不合法、值含非法运行时控制字符为
  `mcp_setting_missing`；Agent 未声明 HTTP MCP 为 `mcp_http_capability_missing`；非空选择且 Agent
  未声明 `session/load` 为 `mcp_load_session_missing`。运行时失败以 `agent_failed` 加稳定 detail 结束
  会话，遵循原进程清理和终态顺序。
- MCP setup 失败时 `AgentSessionEnded` 另携带可选的无值 `plugin_id`、`setting_id` 与 `transport`
  （`stdio` 或 `http`），无法确定的字段与其他终态一律省略；`detail` 继续只含稳定 code；包编译或
  I/O 错误不透传原始文本；preflight 与能力失败的日志同样只记录 code 与这些标识。

### 禁止结果

- 能力失败路径出现 `session/new` 或任何值到达 Agent；能力在任何 ACP 帧之前断言；终态、日志或错误
  链含值；非 MCP 终态携带 `plugin_id`、`setting_id` 或 `transport`；以健康探测替代能力协商。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 能力广告与 wire 拼写 | `crates/node-protocol/tests/protocol/session.rs` 的 `accepts_independent_and_combined_execution_capabilities` 固定 `session_mcp` 拼写；`apps/ora-node/tests/standalone/agent_sessions/support.rs` 的生产握手断言真实 Node 进程广告 `SessionMcp`（与 `AgentSession` 同批） | Partial |
| 能力失败停在 `session/new` 前 | `apps/ora-node/tests/agent_session/mcp.rs` 的 `mcp_capability_failures_stop_before_session_new`：ACP 方法列表只有 `initialize`，终态为 `agent_failed` + `mcp_load_session_missing` / `mcp_http_capability_missing`，进程清理与租约归零 | Partial |
| preflight 失败不启动 Agent | 同文件 `mcp_preflight_failures_start_no_agent`：`mcp_plugin_missing` / `mcp_setting_missing` 无 ACP 帧、无插件进程 | Covered |
| 失败标识无值 | 上述两测试断言终态携带 `plugin_id` / `setting_id` / `transport` 且其 JSON、日志与文件扫描无值；spec 级失败按帧校验拒绝（`session_mcp_rejects_ambiguous_or_unbounded_selection`） | Covered |

### 代表性证据与剩余缺口

- 能力广告与 wire 拼写：[accepts_independent_and_combined_execution_capabilities](https://github.com/ora-space/desktop/blob/98c0414fdaf998ae7ede817f743e3431d94c81d7/crates/node-protocol/tests/protocol/session.rs#L10)（L28、L36）固定 `NodeCapability::SessionMcp` 的 wire 拼写 `"session_mcp"`；standalone 握手辅助 [connect](https://github.com/ora-space/desktop/blob/98c0414fdaf998ae7ede817f743e3431d94c81d7/apps/ora-node/tests/standalone/agent_sessions/support.rs#L136)（L148–159，由 [session_commands_use_the_production_ledger_and_installer_catalog](https://github.com/ora-space/desktop/blob/98c0414fdaf998ae7ede817f743e3431d94c81d7/apps/ora-node/tests/standalone/agent_sessions.rs#L25) 等调用）断言真实 `ora-node` 进程广告 `AgentSession` 与 `SessionMcp`（同批分支见 [worker.rs#L137](https://github.com/ora-space/desktop/blob/98c0414fdaf998ae7ede817f743e3431d94c81d7/apps/ora-node/src/service/worker.rs#L137) 的 `if agents.is_some()`）。
  - 缺口（`Partial` — 能力广告与 wire 拼写）：无 Agent 配置的 Node 省略 `SessionMcp` 只由该分支读出（[worker.rs#L139](https://github.com/ora-space/desktop/blob/98c0414fdaf998ae7ede817f743e3431d94c81d7/apps/ora-node/src/service/worker.rs#L139)），没有「不配置 Agent 的 Node 不广告 `SessionMcp`」的直接测试；按能力派发属于 Controller 侧义务，在 [Controller 用例](../../controller/session/session-mcp-selection.md)登记。
- 能力失败停在 `session/new` 前：[mcp_capability_failures_stop_before_session_new](https://github.com/ora-space/desktop/blob/98c0414fdaf998ae7ede817f743e3431d94c81d7/apps/ora-node/tests/agent_session/mcp.rs#L206) 证明 `echo-acp-methods` 只有 `"initialize\n"`（没有 `session/new`）、终态为 `agent_failed` 与 `mcp_load_session_missing` / `mcp_http_capability_missing`、终态 JSON/日志/文件无秘密。
  - 缺口（`Partial` — 能力失败停在 `session/new` 前）：该测试先 `sessions.shutdown()`（[L234](https://github.com/ora-space/desktop/blob/98c0414fdaf998ae7ede817f743e3431d94c81d7/apps/ora-node/tests/agent_session/mcp.rs#L234)）再断言租约归零（[L257](https://github.com/ora-space/desktop/blob/98c0414fdaf998ae7ede817f743e3431d94c81d7/apps/ora-node/tests/agent_session/mcp.rs#L257)），分不清释放来自失败路径清理还是 shutdown；也没有断言插件进程在失败时已退出，`shutdown` 之前的进程与租约释放没有直接证据。
- preflight 失败不启动 Agent：[mcp_preflight_failures_start_no_agent](https://github.com/ora-space/desktop/blob/98c0414fdaf998ae7ede817f743e3431d94c81d7/apps/ora-node/tests/agent_session/mcp.rs#L273) 证明 `mcp_plugin_missing` 与 `mcp_setting_missing` 下 `fixture.plugin_pids()` 为空（无插件进程、无 ACP 帧）且日志/终态只记 code 与标识。
- 失败标识无值：[session_mcp_failure_identifiers_round_trip_without_values](https://github.com/ora-space/desktop/blob/98c0414fdaf998ae7ede817f743e3431d94c81d7/crates/node-protocol/tests/protocol/agent_session.rs#L511) 证明 `AgentSessionEnded` 往返可选 `plugin_id`/`setting_id`/`transport`、旧帧省略兼容、非法 `plugin_id` 或含控制字符 `setting_id` 被拒绝；spec 级帧校验实机连接关闭的缺口交叉见第一个用例的 `Partial` 说明。

### 决策依据

D4，不变量 5；协议根决策的能力协商与 close-code ADR。

## The Package Lease And The Read-Only View Live Until The Agent Stops

### 风险

会话运行期间 MCP 包被移除，Agent 之后从已删除路径重新 spawn server；Separate 工作负载收到 Node
home 内的命令无法执行；Setting 被写入 view；Agent 停止后租约或 view 残留，或清理越过本次会话。

### 前置状态

选择含 stdio MCP 包；Separate 工作负载身份不能读取 Node 私有 home；会话正在运行。

### 触发

会话运行期间移除插件；会话正常结束或 Node 停止；以 Separate 工作负载启动带 stdio 插件的会话。

### 必须成立

- Node 对每个所选 MCP 包持有使用租约直到 Agent 停止；会话期间活跃插件移除被租约拒绝。
- stdio command 属于被选版本包 `assets/` 内的普通文件；不可写的 MCP 包文件以 hard-link 进入本次
  会话拥有的 package view，交付的是 view 内的 command；view 保留同一版本字节与 assets containment。
- 不修改 Node home 权限、不把 Setting 写入 view；Agent 停止后释放 view 与包租约；会话子目录使用
  独立名称，复用既有会话目录清理，不碰其他会话、工作区或用户文件。

### 禁止结果

- 会话期间包被移除或命令路径失效；Agent 收到 Node home 路径或 Controller 提供的命令；view 可写或
  含 Setting 值；结束后租约或 view 残留；清理越界到其他会话或全局目录。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 租约活到 Agent 停止 | `apps/ora-node/tests/agent_session/mcp.rs` 的 `mcp_reaches_real_acp_session_new_without_persisting_or_logging_settings`：会话期间租约计数包含所选 MCP 包，结束并关闭会话后归零；运行期间 `RemovePlugins` 对被选 MCP 包返回 `plugin_in_use` 的直接断言尚缺 | Partial |
| view 交付与清理 | 同测试 Separate 分支：交付的 command 位于 workload 目录内而不在 Node home 下，会话结束后 workload 目录清空 | Covered |

### 代表性证据与剩余缺口

- 租约活到 Agent 停止：[mcp_reaches_real_acp_session_new_without_persisting_or_logging_settings](https://github.com/ora-space/desktop/blob/98c0414fdaf998ae7ede817f743e3431d94c81d7/apps/ora-node/tests/agent_session/mcp.rs#L119)（L168、L184）证明会话运行期间租约计数为 3（1 个 Agent + 2 个 MCP 包），`UserEnded` 结束并 `sessions.shutdown()` 后租约归零。
  - 缺口（`Partial` — 租约活到 Agent 停止）：`98c0414f` 基线上没有会话运行期间向被选 MCP 包发 `RemovePlugins` 并断言返回 `plugin_in_use` 的直接测试（[session_commands_use_the_production_ledger_and_installer_catalog](https://github.com/ora-space/desktop/blob/98c0414fdaf998ae7ede817f743e3431d94c81d7/apps/ora-node/tests/standalone/agent_sessions.rs#L25) L63–94 仅对 Agent 包测过 `RemovePlugins` → `plugin_in_use`）。后续 `20261116` D6 已将 MCP 租约细化为精确版本租约并在[会话 MCP 刷新与宿主健康用例](session-mcp-lifecycle.md)维护其移除验证，不回溯消除本基线缺口。
- view 交付与清理：[mcp_reaches_real_acp_session_new_without_persisting_or_logging_settings](https://github.com/ora-space/desktop/blob/98c0414fdaf998ae7ede817f743e3431d94c81d7/apps/ora-node/tests/agent_session/mcp.rs#L119) Separate 分支（L137–151、L186–193）配合 echo-agent 的 [verify_mcp_frame](https://github.com/ora-space/desktop/blob/98c0414fdaf998ae7ede817f743e3431d94c81d7/apps/ora-node/fixtures/echo-agent/agent.rs#L509) 与 [command_is_in_required_view](https://github.com/ora-space/desktop/blob/98c0414fdaf998ae7ede817f743e3431d94c81d7/apps/ora-node/fixtures/echo-agent/agent.rs#L529) 证明交付的 stdio 命令位于 workload 目录内且不在 Node home 下，会话结束后 workload 目录清空；`link_tree_no_follow` 的硬链接同 inode、`0o755` 目录权限、拒绝符号链接/特殊文件/已有目标由 [links_files_into_traversable_directories](https://github.com/ora-space/desktop/blob/98c0414fdaf998ae7ede817f743e3431d94c81d7/crates/utils/src/fs/linked_tree.rs#L148)、[rejects_symbolic_links](https://github.com/ora-space/desktop/blob/98c0414fdaf998ae7ede817f743e3431d94c81d7/crates/utils/src/fs/linked_tree.rs#L195)、[rejects_special_files](https://github.com/ora-space/desktop/blob/98c0414fdaf998ae7ede817f743e3431d94c81d7/crates/utils/src/fs/linked_tree.rs#L211) 与 [refuses_an_existing_destination](https://github.com/ora-space/desktop/blob/98c0414fdaf998ae7ede817f743e3431d94c81d7/crates/utils/src/fs/linked_tree.rs#L228) 直接证明。（注：`98c0414f` 上 `link_tree_no_follow` 含 `EXDEV` 跨文件系统复制回退；`20261116` 的 D5 要求同文件系统并拒绝 `EXDEV`，desktop `83b450b9` 起 MCP view 与 session 的 package view 改用只做硬链接的 `hard_link_tree_no_follow`，通用 `link_tree_no_follow` 的回退不变。）
  - 范围说明（`Covered` 不覆盖的一项禁止结果）：`assert_no_secret_files` 只扫 `fixture.home()`；Separate 的 view 在 `fixture.workload_directory()`（`<root>/agent`）下，不在扫描范围内。「不把 Setting 写入 view」目前靠结构与帧断言支撑（view 只硬链接安装包文件，Setting 值只进内存与 `session/new` 帧），没有对 view 目录的字节扫描。

### 决策依据

D5，不变量 6；`InstallPlugins` 的精确版本安装与包验证。

## Desktop Resolution Behaviour Stays Unchanged Behind The Shared Resolver

### 风险

解析器从 Backend 移到共享 crate 时改变 Desktop 的选择、映射、顺序或错误分类；Desktop 引入 Node 的
spec 来源，或 Node 引入 Desktop 的配置 store、自动选择、刷新或健康探测职责。

### 前置状态

Desktop 原有 MCP 选择、映射与错误分类测试在移动前的基线通过；纯解析器已移至共享 crate，由 Desktop
与 Node 共用。

### 触发

在移动后的代码上运行 Desktop 既有 `mcp` 测试套件与新增用例；执行 stdio/HTTP 映射、能力缺失与
revision 生成。

### 必须成立

- 共享解析器保持 Desktop 原行为：canonical Plugin ID 排序、精确 revision、stdio 与 HTTP 映射、错误
  分类不变；Desktop 既有 `mcp` 测试在移动后原样通过，只允许新增用例。
- Desktop 的自动选择、配置 store、刷新与 Host 健康探测规则及归属不变；Node 侧按 spec 解析，不使用
  Desktop 的配置来源。

### 禁止结果

- 移动引入选择、映射、顺序或错误分类漂移；Desktop 测试被改写以适配新行为；健康探测变成 Node 职责
  或 Node 出现 Desktop 的 store。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 映射与顺序不漂移 | `crates/backend/src/session_setup/mcp/tests.rs` 的 `maps_stdio_and_http_in_canonical_plugin_id_order_with_exact_revisions`、`regenerates_when_package_version_changes_during_resolve`、`resolves_the_plugin_behind_the_agent_identity_the_runtime_supervises`、`resolves_nothing_for_an_agent_no_package_supplies`；Desktop 既有 `mcp` 测试套件在移动后原样通过（基线用例不变，仅新增） | Covered |
| 错误分类不变 | 同文件 `fails_closed_when_http_capability_is_missing`、`fails_before_send_when_load_capability_is_missing_for_a_non_empty_set`、`empty_set_does_not_require_load_capability`、`omits_incomplete_mcp_without_failing_complete_peers` | Covered |
| 值与文件边界 | 同文件 `desired_revision_excludes_setting_values`、`resolver_does_not_create_workspace_files` | Covered |

### 代表性证据与剩余缺口

`desktop#773`（`98c0414f`）将纯解析器移入 `crates/agent-runtime/src/mcp_resolver/` 时未改动 `crates/backend/src/session_setup/mcp/tests.rs`，下列 10 项既有 Desktop 基线测试原样通过：

- 映射与顺序不漂移：[maps_stdio_and_http_in_canonical_plugin_id_order_with_exact_revisions](https://github.com/ora-space/desktop/blob/98c0414fdaf998ae7ede817f743e3431d94c81d7/crates/backend/src/session_setup/mcp/tests.rs#L171)、[regenerates_when_package_version_changes_during_resolve](https://github.com/ora-space/desktop/blob/98c0414fdaf998ae7ede817f743e3431d94c81d7/crates/backend/src/session_setup/mcp/tests.rs#L378)、[resolves_the_plugin_behind_the_agent_identity_the_runtime_supervises](https://github.com/ora-space/desktop/blob/98c0414fdaf998ae7ede817f743e3431d94c81d7/crates/backend/src/session_setup/mcp/tests.rs#L499)、[resolves_nothing_for_an_agent_no_package_supplies](https://github.com/ora-space/desktop/blob/98c0414fdaf998ae7ede817f743e3431d94c81d7/crates/backend/src/session_setup/mcp/tests.rs#L547)。
- 错误分类不变：[fails_closed_when_http_capability_is_missing](https://github.com/ora-space/desktop/blob/98c0414fdaf998ae7ede817f743e3431d94c81d7/crates/backend/src/session_setup/mcp/tests.rs#L296)、[fails_before_send_when_load_capability_is_missing_for_a_non_empty_set](https://github.com/ora-space/desktop/blob/98c0414fdaf998ae7ede817f743e3431d94c81d7/crates/backend/src/session_setup/mcp/tests.rs#L333)、[empty_set_does_not_require_load_capability](https://github.com/ora-space/desktop/blob/98c0414fdaf998ae7ede817f743e3431d94c81d7/crates/backend/src/session_setup/mcp/tests.rs#L360)、[omits_incomplete_mcp_without_failing_complete_peers](https://github.com/ora-space/desktop/blob/98c0414fdaf998ae7ede817f743e3431d94c81d7/crates/backend/src/session_setup/mcp/tests.rs#L254)。
- 值与文件边界：[desired_revision_excludes_setting_values](https://github.com/ora-space/desktop/blob/98c0414fdaf998ae7ede817f743e3431d94c81d7/crates/backend/src/session_setup/mcp/tests.rs#L416)、[resolver_does_not_create_workspace_files](https://github.com/ora-space/desktop/blob/98c0414fdaf998ae7ede817f743e3431d94c81d7/crates/backend/src/session_setup/mcp/tests.rs#L449)。

### 决策依据

D2；继承 Desktop 的 ACP Session setup 交付、工作流显式选择与 Host 健康探测归属。
