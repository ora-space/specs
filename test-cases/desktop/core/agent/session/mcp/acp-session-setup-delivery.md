# Session MCP 通过 ACP 注入与热更新

## Session Setup Snapshots Must Be Complete Ordered And Secret-Free

### 风险

如果不同 setup 路径各自组装 `mcpServers`，或 Snapshot 跨 revision 拼接、静默省略失败成员、把 Setting
实值编进 revision digest，运行中会话会拿到不完整或可泄漏的 MCP 集合，且无法从成功响应判断当前到底
加载了什么。

### 前置状态

- installed snapshot 中存在静态有效的 stdio 与 HTTP MCP 插件，canonical Plugin ID 不同。
- 部分插件 Settings 完整，部分 required Setting 缺失。
- Agent 声明 `session/load` 与 HTTP MCP capability。

### 触发

Host 为 `session/new`、显式 `session/load` 或 Live Session refresh 解析 Effective MCP Set。

### 必须成立

- 同一 installed/configuration snapshot 对所有 setup/refresh 产生完全相同、按 canonical Plugin ID
  排序的 `mcpServers`；workspace context 分别使用各 Session 的绝对 `cwd`。
- 配置不完整的 MCP 被省略，不使其他完整成员失败。
- Snapshot 成员绑定精确 package version 与 Configuration revision；解析期间 version 变化则放弃并
  重新生成完整列表。
- Desired Session MCP Revision 不含 Setting 实值。

### 禁止结果

- 发送跨 revision 拼接或缺少入选成员的部分 `mcpServers`。
- 使用展示名称、identifier 尾段或临时后缀作为 Server name。
- 把 Setting 实值写入 digest、wakeup、日志或错误。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| stdio 与 HTTP 按 canonical Plugin ID 排序，并绑定精确 package/configuration revision | `ora_backend::session_setup::mcp::tests::maps_stdio_and_http_in_canonical_plugin_id_order_with_exact_revisions` | `Covered` |
| 不完整 MCP 被省略且不拖垮完整成员 | `ora_backend::session_setup::mcp::tests::omits_incomplete_mcp_without_failing_complete_peers` | `Covered` |
| 解析期间 package version 变化则重新生成完整 Snapshot | `ora_backend::session_setup::mcp::tests::regenerates_when_package_version_changes_during_resolve` | `Covered` |
| Desired revision 不含 Setting 实值 | `ora_backend::session_setup::mcp::tests::desired_revision_excludes_setting_values` | `Covered` |
| `session/new`、显式恢复与 refresh 共用同一 resolver | 解析函数被 Session Setup 单入口调用，尚无 actor 级对比三类路径产出的 payload | `Partial` |

### 决策依据

- Session MCP 根决策 D1、D2、D6、D8：Effective MCP Set、无副作用 resolver、canonical 顺序与秘密边界。
- 不变量 2、3、7、9。

## Live Sessions Must Converge Without Interrupting In-Flight Prompts

### 风险

如果空闲会话继续使用旧集合，或 busy 会话把过期的成功 `session/load` 当成 Current，新 prompt 会在
配置已提交后仍带着旧 MCP 进入 Agent；如果 refresh 失败仍允许 prompt，用户无法区分阻塞与静默降级。

### 前置状态

- 一个 Live Session 已成功 setup，持有 Active Session MCP Revision。
- Durable configuration/catalog 随后发生变化，Host 发布 level-triggered wakeup。

### 触发

空闲或 busy Live Session 收到 Desired revision 变化；随后发生 prompt admission、refresh 成功、refresh
失败或更新的 Desired 在 in-flight load 期间再次变化。

### 必须成立

- 空闲会话立即欠一次 refresh；busy 会话允许当前 turn 完成，并把 refresh 记为 owed work。
- 只有匹配当时 Desired 的成功 load 才能推进 Active revision；更新的 Desired 使过期成功响应不能清
  除 owed work。
- refresh 失败后该 Session 进入 blocked，后续 prompt 必须先重试收敛。
- 已停止 Session 忽略 Desired 变化，不做后台 refresh。

### 禁止结果

- 打断已经开始的 prompt。
- 在 Active 与 Desired 不一致时放行新 prompt。
- 一个 Session 失败回滚已提交配置，或阻止其他 Session 独立收敛。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| 空闲 Live Session 在 Desired 变化后欠 refresh | `ora_backend::session_setup::mcp::tests::live_idle_session_owes_refresh_when_desired_changes` | `Covered` |
| busy 成功响应不能清除更新的 pending revision | `ora_backend::session_setup::mcp::tests::live_busy_success_cannot_clear_a_newer_pending_revision` | `Covered` |
| refresh 失败会阻塞 prompt，直到重试 | `ora_backend::session_setup::mcp::tests::live_refresh_failure_blocks_prompts_until_retry` | `Covered` |
| 停止的 Session 忽略 Desired 变化 | `ora_backend::session_setup::mcp::tests::stopped_sessions_ignore_desired_changes` | `Covered` |
| 真实 actor 在 Settings 保存后对空闲会话发出 `session/load` | 当前证据停在 `LiveMcpState`，尚未覆盖 ACP 帧发出 | `Missing` |

### 决策依据

- Session MCP 根决策 D3：配置变化令 Live Session 在安全点热更新。
- 不变量 4、5。

## Effect And MCP Must Share One Agent Session Barrier

### 风险

如果 Skill mutation 与 MCP refresh 各自重启 Agent 或 reload Session，会绕过 Effect fencing、丢失
owed MCP update，或让新 prompt 穿过变更窗口。

### 前置状态

- 同一 Agent 同时声明 Skill Effect Resource，并承载至少一个 Live Session。
- Skill Resource mutation 与 MCP Desired revision 变化可能并发。

### 触发

Effect `coordinate`/`reactivate` 与 MCP refresh 争用同一个 Agent Session Barrier。

### 必须成立

- 同一 Agent 的 MCP refresh 与 Effect mutation 互斥；已有 barrier 内只能提高待收敛 Desired revision。
- 不同 Agent 不共享 barrier。
- Agent 被替换后，恢复路径现场解析最新 MCP Snapshot。

### 禁止结果

- MCP wakeup 启动第二条绕过 Effect fencing 的 Agent restart 或 Session reload。
- 先恢复旧 MCP 集合再遗漏 owed update。
- 把 Session MCP 收敛解释为 Effect Target ready。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| 同一 Agent 上 MCP 与 Effect 原因互斥占用 barrier | `ora_backend::session_setup::mcp::tests::mcp_and_effect_share_one_agent_session_barrier` | `Covered` |
| 无关 Agent 不共享 barrier | `ora_backend::session_setup::mcp::tests::unrelated_agents_do_not_share_a_barrier` | `Covered` |
| Agent replacement 后恢复的 Session 现场解析最新 MCP | 现有 replacement 测试未断言恢复请求携带最新 `mcpServers` | `Missing` |

### 决策依据

- Session MCP 根决策 D4：Session Runtime 与 Effect 共享 Agent Session Barrier，但不共享 readiness。
- 不变量 6、11。

## MCP Lifecycle Must Not Touch Workspace Files Or Claim Ready

### 风险

如果 setup/refresh/卸载仍写入 Agent 原生项目文件，或把旧 MCP Effect format 继续当作 Consumer
Resource，会重新引入 shared-file ownership，并把配置完整误称为 `MCP Ready`。

### 前置状态

- Workspace 可能已有用户或其他工具写入的原生 MCP 配置。
- Agent 插件可能仍声明未发布的 `ora/opencode-mcp-config.v1` 或 `ora/claude-mcp-config.v1` Resource。

### 触发

解析 Snapshot、安装/保存/卸载 MCP、或把 Agent 注册为 Effect Consumer。

### 必须成立

- resolver 不创建、修改或删除 Workspace 文件。
- 未发布的 MCP 文件物化 format 被跳过；只声明这些 format 的插件不是 Effect Consumer。
- Skill Resource 仍可正常注册。

### 禁止结果

- 为 MCP 创建 `.mcp.json`、OpenCode JSON/JSONC、ownership sidecar 或 Git exclude。
- 读取用户已有原生 MCP 配置来授予权限或去重。
- 暴露无对应证据的 `MCP Ready`。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| resolver 不改变 Workspace 文件集合 | `ora_backend::session_setup::mcp::tests::resolver_does_not_create_workspace_files` | `Covered` |
| 未发布 MCP format 不进入 Effect Consumer | `ora_backend::agent_runtime::plugin_agent::tests::skips_unpublished_mcp_file_materialization_resources` | `Covered` |
| 仅声明 MCP format 的插件不是 Consumer | `ora_backend::agent_runtime::plugin_agent::tests::treats_mcp_only_materialization_as_no_consumer` | `Covered` |
| setup、refresh、失败与卸载的端到端文件不变性 | 尚无 Desktop e2e 覆盖安装到 refresh 全路径的 Workspace 快照 | `Missing` |

### 决策依据

- Session MCP 根决策 D9：只有 Session convergence 状态，没有 materialization 生命周期。
- 不变量 1、10、11。

## Missing Capabilities Or Member Failures Must Fail The Whole Session Before Send

### 风险

如果缺少 `session/load` 或 HTTP capability 时仍发送部分 ACP 帧，或某个入选成员解析失败后跳过该成员，
成功 setup 将无法代表完整 Effective MCP Set。

### 前置状态

- Effective MCP Set 可能为空、只含 stdio，或包含 HTTP MCP。
- Agent 可能未声明 `session/load` 或 `mcp_capabilities.http`。

### 触发

Host 在发送 `session/new` 或 `session/load` 前校验 capability 并解析每个入选成员。

### 必须成立

- 非空集合且缺少 `session/load` 时，在发送前以稳定 capability 错误失败。
- HTTP MCP 存在但 Agent 未声明 HTTP capability 时同样 fail closed。
- 空集合不要求 `session/load`。
- 失败错误脱敏，只含 Plugin ID、Setting ID、transport 与稳定错误码。

### 禁止结果

- 用 `session/new` 或 UI 历史重放代替缺少 load support 的恢复。
- 静默省略失败成员或把完整集合降级为空集合。
- 错误文本包含 Setting 原值或可逆转义形式。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| 缺少 HTTP capability 时 fail closed | `ora_backend::session_setup::mcp::tests::fails_closed_when_http_capability_is_missing` | `Covered` |
| 非空集合缺少 load capability 时在发送前失败 | `ora_backend::session_setup::mcp::tests::fails_before_send_when_load_capability_is_missing_for_a_non_empty_set` | `Covered` |
| 空集合不要求 load capability | `ora_backend::session_setup::mcp::tests::empty_set_does_not_require_load_capability` | `Covered` |
| PublicError 使用稳定 `session_mcp_setup_failed` 且参数不含 Setting 实值 | 契约枚举与 resolver 错误映射存在，尚无断言错误链不含原值的专属测试 | `Partial` |

### 决策依据

- Session MCP 根决策 D5、D7：只用 ACP 1.6.0 `mcpServers`，入选成员全量成败。
- 不变量 8、9。
