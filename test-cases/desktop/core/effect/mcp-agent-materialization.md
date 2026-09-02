# MCP Agent 自动物化

## Secret Values Must Reach Only the Exact Agent Child Process

### 风险

MCP API key 若进入项目文件、sidecar、Revision、日志、UI DTO 或插件进程环境，会扩大泄露面；若 host
只按变量名注入，则错误 Agent 或错误 Workspace 也可能读取密钥。

### 前置状态

- MCP 配置完整并已发布 secret-free Revision。
- 一个 Agent Target 已将该 revision 物化并写入匹配 sidecar。

### 触发

Agent 插件请求 host 在 Workspace cwd 下启动其 CLI/adapter。

### 必须成立

- 插件请求中的 `ORA_MCP_*` 被拒绝。
- Host 同时校验 Agent identity、Workspace containment、sidecar、配置 fingerprint 与配置 revision。
- 只有子进程环境含有效 Setting 值；其他持久化与 DTO 只有引用或 redacted state。
- OpenCode JSON-string-content 与 Claude raw 编码分别正确。

### 禁止结果

- 密钥进入 Agent 插件进程或任何项目/Effect 持久化数据。
- sidecar 缺失、漂移或 revision 过期时仍注入。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
|---|---|---|
| 保留变量不能由插件提供，host provider 绑定 plugin/cwd | `ora_plugin_lifecycle::childprocess_tests::spawn_rejects_plugin_supplied_reserved_mcp_environment`、`spawn_injects_host_authorized_mcp_environment_after_binding_the_workspace` | `Covered` |
| Revision template 不含 secret，OpenCode/Claude 编码不同 | `ora_effect_mcp::template::tests::http_templates_are_secret_free_while_each_agent_receives_its_required_encoding` | `Covered` |
| sidecar、fingerprint、revision 全链路失配时不注入 | backend provider performs the complete fail-closed check; filesystem lifecycle and forged ownership are covered by `ora_effect_mcp::planner_tests` | `Covered` |
| MCP response DTO 不返回绑定值，未修改的 redacted 值可显式保留 | typed redaction + `preserveSettingIds` path, backend compilation and frontend settings tests | `Covered` |

### 决策依据

- ADR `20260901-mcp-agent-materialization.md` 的 Secret 注入边界。

## Shared MCP Files Must Preserve User State and Ownership

### 风险

项目 JSON/JSONC 同时由用户、Agent 与 Ora 使用。整文件重写、同名接管或错误删除会破坏项目配置。

### 前置状态

- 文件含注释、未知顶层字段或用户 server。
- Ora 可能拥有零个或多个 sidecar 记录。

### 触发

MCP Desired 新增、更新、卸载、漂移检测或进程恢复后重新 reconcile。

### 必须成立

- 只修改目标 object 中 sidecar 精确拥有的 key。
- 注释、未知字段与用户 key 保留；重复运行 byte-stable/no-op。
- JSONC 高优先级冲突、sidecar mismatch、drift 与同名 collision 阻塞整份 Resource。
- 卸载仅删除精确 owned key；最后一个 key 删除后仍保留合法用户文件。

### 禁止结果

- 依据名称、模板或 fingerprint 相似性接管用户 server。
- 在 ownership 不确定时准备 mutation 或删除 sidecar 证据。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
|---|---|---|
| JSONC merge 保留周围注释与字段 | `ora_utils::jsonc::tests::preserves_jsonc_around_nested_updates` | `Covered` |
| literal `.` 不会被 workspace context 替换 | `ora_effect_mcp::template::tests::workspace_context_does_not_replace_a_literal_dot_argument` | `Covered` |
| sidecar drift/collision/JSONC override 形成整文件 blocking | `name_and_ownership_collisions_block_the_whole_resource`、`managed_entry_drift_requires_manual_recovery`、`mismatched_sidecar_claim_does_not_grant_mutation_authority` | `Covered` |
| 安装、重复收敛、卸载删除与崩溃重放 | `shared_file_lifecycle_preserves_user_jsonc_and_replays_idempotently` | `Covered` |

### 决策依据

- 基础 ADR 的 Desired/Managed/Observed/Preserved 与 journal 不变量。
- MCP ADR 的 shared-file ownership 决策。

## One Target Projection Must Coordinate Skills and MCP Once

### 风险

若 planner 把全部 Desired 送到每个 Resource，或 Skill/MCP 分别协调，Agent 会看到部分 generation、
重复重启，ready watermark 也可能提前推进。

### 前置状态

- Agent Target 同时绑定 Skill 与 MCP Resource。
- 同一 Workspace generation 同时包含两种 Desired。

### 触发

worker 对 Target 执行完整 projection 与 reconcile。

### 必须成立

- 每个 binding 只接收其接受的 Effect kind。
- 两个 Resource 属于同一 coordination plan；mutation 后只 reactivate 一次。
- 任一 Resource blocking 时 Target 不 ready，聊天发送门显示不可用。

### 禁止结果

- Skill Desired 被送入 MCP planner，或 MCP Desired 被送入 Skill planner。
- 任一单独 Resource current 就推进 Target ready。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
|---|---|---|
| Agent 注册同时声明精确双 Resource | OpenCode/Claude `tests/host-simulator.ts` registration assertions | `Covered` |
| 双 Resource 共享一次 barrier/restart | OpenCode/Claude `tests/host-simulator.ts` spawn-count assertions | `Covered` |
| composite/kind planner 按 binding kind 精确路由 | `ora_effect_skill::tests::skill_planner_routes_only_skill_effects_to_skill_bindings` | `Covered` |
| UI 按 Workspace+Agent 完整 Target watermark 阻止发送 | `useAgentEffectReadiness` polls the tagged selector; Plugin Configuration separately exposes the four-state aggregate | `Covered` |

### 决策依据

- 基础 ADR 的 Generic Target 与 complete generation 决策。
- MCP ADR 的 shared composite reconciler 决策。
