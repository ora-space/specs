# Effect 持久化

## 定位

Workspace Desired State 必须持久化到 Ora 数据库。数据库保存当前声明、每个 AgentTarget 的收敛进度和 Ora ownership ledger，使应用重启后仍能继续 reconcile，并能准确区分 Ora-managed 与 Preserved State。

持久化兼容由数据库 migration 和 repository module 负责。领域模型、reconciler 和 Agent adapter 只处理当前版本的强类型对象，不能感知旧表结构或长期维护多套历史 definition。

第一版保证以下升级方向：

```text
旧数据库 + 新 Ora
        ↓ transaction migrations
当前数据库 + 当前领域模型
```

不保证新数据库可以由旧 Ora 继续读写。旧 Ora 遇到高于自身支持版本的数据库时必须返回 `DatabaseVersionTooNew`，不能忽略未知表或字段后继续运行。开发和测试可以提供受控 down migration，但产品不承诺降级不会丢失新版本状态。

## 持久化范围

数据库分别保存：

1. Workspace 当前 Desired State 及 generation；
2. 每个 AgentTarget 的 observed、applied 和 ready generation；
3. Ora 已物化资源的 ownership record；
4. 最近一次 reconcile 的持久状态和诊断错误。

Session 的实时 `Idle`、`Working` 和 `StoppingTurn` 不是持久化事实源。它们属于运行时 Observed State；应用重启后必须向 Agent adapter 重新观察，无法确认时按非空闲处理。数据库可以保存诊断快照，但旧快照不能作为允许配置变更或资源清理的证据。

数据库保存当前 Desired State 快照，不要求以事件日志作为事实源。若以后需要审计或回滚，应增加独立历史记录，不能让 reconciler 通过重放 UI 操作恢复当前声明。

## Desired State 表

稳定身份和并发控制字段使用关系表表达：

```sql
CREATE TABLE workspace_effect_specs (
    workspace_id TEXT PRIMARY KEY,
    generation   INTEGER NOT NULL,
    created_at   INTEGER NOT NULL,
    updated_at   INTEGER NOT NULL
);

CREATE TABLE workspace_effect_skills (
    workspace_id       TEXT NOT NULL,
    name               TEXT NOT NULL,
    installed_skill_id TEXT NOT NULL,
    skill_md_digest    TEXT NOT NULL,
    digest_algorithm   TEXT NOT NULL,
    PRIMARY KEY (workspace_id, name)
);

CREATE TABLE workspace_effect_mcps (
    workspace_id              TEXT NOT NULL,
    mcp_id                    TEXT NOT NULL,
    exact_version             TEXT NOT NULL,
    definition_schema_version INTEGER NOT NULL,
    definition_json           TEXT NOT NULL,
    definition_digest         TEXT NOT NULL,
    digest_algorithm          TEXT NOT NULL,
    PRIMARY KEY (workspace_id, mcp_id)
);
```

表结构展示领域约束，不代替具体 migration 中的外键、索引和 SQLite table rebuild 细节。

Skill definition 结构简单且身份稳定，使用独立列保存。MCP transport、Setting 和 SecretRef 是可演进的 tagged union，允许将其规范化内容保存为版本化 JSON。`definition_json` 只能由 repository module 解析和生成，不能越过持久化 seam 暴露给领域或 Agent adapter。

Secret 只能以 `SecretRef` 出现在持久化 definition 中。任何 Desired State、ownership record、错误详情或摘要输入都不得包含 Secret 明文。

## Status 与 ownership

Desired State、AgentTarget status 和 Managed State 的更新频率与事实来源不同，必须分别持久化：

```sql
CREATE TABLE workspace_agent_effect_status (
    workspace_id        TEXT NOT NULL,
    agent_plugin_id     TEXT NOT NULL,
    observed_generation INTEGER,
    applied_generation  INTEGER,
    ready_generation    INTEGER,
    phase               TEXT NOT NULL,
    last_error_code     TEXT,
    updated_at          INTEGER NOT NULL,
    PRIMARY KEY (workspace_id, agent_plugin_id)
);

CREATE TABLE managed_effect_resources (
    workspace_id        TEXT NOT NULL,
    agent_plugin_id     TEXT NOT NULL,
    managed_identity    TEXT NOT NULL,
    resource_kind       TEXT NOT NULL,
    logical_name        TEXT NOT NULL,
    target_locator      TEXT NOT NULL,
    applied_fingerprint TEXT NOT NULL,
    applied_generation  INTEGER NOT NULL,
    PRIMARY KEY (
        workspace_id,
        agent_plugin_id,
        managed_identity
    )
);
```

更新 AgentTarget status 不能推进 Workspace generation。ownership record 必须和目标配置物化处于同一可恢复操作中：配置未成功写入时不能提前声明 ownership；删除未完成时也不能提前移除 ownership record。

持久化的错误只保存稳定错误码和安全诊断信息，不保存 Secret、完整环境变量、目标配置全文或不受限的底层 stderr。

## 跨存储恢复

SQLite transaction 不能与 Agent 配置文件、sandbox 或 Agent runtime 原子提交。Effect 必须使用 durable operation record 连接数据库状态与外部副作用：

```text
EffectOperation {
  operation_id,
  target_id,
  desired_generation,
  previous_fingerprint,
  planned_fingerprint,
  planned_ownership,
  phase: Prepared | Applied | Finalized | RecoveryRequired
}
```

reconcile 在修改外部资源前先持久化 `Prepared` operation；外部原子替换成功后记录 `Applied`；ownership、AgentTarget status 和 operation 的 `Finalized` 必须在同一个数据库 transaction 中提交。

应用崩溃后，reconciler 根据 operation 和目标环境重新观察：

- observed fingerprint 等于 `planned_fingerprint`：外部写入已经成功，可以完成 ownership 和 status 提交；
- observed fingerprint 等于 `previous_fingerprint`：外部写入没有生效，可以安全重试或取消 operation；
- observed fingerprint 两者都不匹配：进入 `RecoveryRequired` 并报告冲突，不能覆盖或删除无法证明归属的状态。

`planned_ownership` 是待提交意图，不是有效 Managed State。只有确认外部目标与计划结果一致后，才能将其提升为正式 ownership record。已终结 operation 可以按保留策略清理，但未终结 operation 必须跨应用重启保留。

## Repository interface

持久化 seam 使用小型强类型 interface 隐藏 SQL、JSON schema、migration 历史和并发控制实现：

```text
WorkspaceEffectRepository
├── load(workspace_id) -> WorkspaceEffectSnapshot
└── replace(workspace_id, expected_generation, desired)
      -> ReplaceOutcome
```

`replace` 必须在单个数据库 transaction 中：

1. 读取并校验当前 generation 等于 `expected_generation`；
2. 规范化并比较新旧 Desired State；
3. 内容没有实质变化时返回 no-op，不推进 generation；
4. 原子替换 Skill 和 MCP 子表；
5. 将 generation 单调递增一次；
6. 提交完整变更。

generation 不匹配时返回 `GenerationConflict`，调用方必须重新读取当前声明后重新计算意图，不能覆盖并发更新。

AgentTarget status 和 ownership 通过同一持久化 module 的窄 interface 更新，但不能要求调用方直接操作表行。Agent adapter 无权读取数据库；它只接收 reconciler 提供的当前强类型 definition 和 ownership context。

## 数据库 schema 演进

Ora 使用线性、事务化 migration 将旧数据库升级到当前 schema。每个 migration 必须完成旧数据的确定性转换；迁移成功后，正常运行路径中只能存在当前表结构。

新增必填字段时采用可验证的阶段式 migration：

```text
add nullable or defaulted storage
        ↓
backfill and validate every existing row
        ↓
rebuild table with final constraints when required
```

不能通过在领域对象中长期保留含糊的 `Option` 字段兼容旧数据。无法可靠推导的新值必须转换为明确状态，例如 `NeedsConfiguration`，不能猜测用户意图。

数据库 migration 负责存储兼容，不等于保留旧业务行为。新版本完成迁移后，不再双写旧表，也不保留“以防万一”的旧 repository 路径。

## Definition schema 演进

数据库 schema version 与 `McpDefinition` schema version 是两个不同概念：

- 数据库 schema version 描述表结构；
- `definition_schema_version` 描述 `definition_json` 的领域编码格式。

MCP definition 从旧版本升级时，migration 必须执行显式转换：

```text
parse McpDefinitionV1
        ↓
validate and convert
        ↓
encode current McpDefinition
        ↓
recompute canonical digest
```

正常 repository 读取路径不能长期按版本分派。旧 definition decoder 只属于对应 migration 的实现，升级完成后领域和 reconciler 只看当前 `McpDefinition`。

如果新 definition 语义需要用户选择，migration 只能保留可证明的数据并将资源标记为 `NeedsConfiguration`。它不能选择任意默认值后宣称 Desired State 已经收敛。

仅改变编码、摘要算法或 canonicalization 而不改变 Desired State 语义时，migration 可以更新持久化摘要而不推进 Workspace generation。转换改变实际目标配置语义时，必须在同一 transaction 中推进 generation，使所有 AgentTarget 重新 reconcile。

## 摘要稳定性

摘要记录必须包含算法标识。Skill 摘要针对 `SKILL.md` 的精确字节计算；MCP 摘要针对当前 schema version 的规范化 definition 计算。

MCP canonicalization 必须固定以下规则：

- 字段顺序；
- 缺省值是否显式编码；
- map key 排序；
- number、boolean 和 string 的规范表示；
- `SecretRef` 的非明文身份表示。

普通 JSON 字段重排或缺省值的等价写法不能产生新的 definition digest。canonicalization 规则发生变化时必须通过版本化 migration 统一重算，不能在读取时悄悄改变摘要。

## 首次引入 Effect State

没有 ownership ledger 的现有 Workspace 必须保守初始化：

```text
Desired State = empty
Managed State = empty
existing Agent configuration = Preserved State
```

migration 不能扫描已有 Agent 配置并根据名称、路径、摘要或内容相似性推断 Ora ownership。即使某个条目可能由旧版 Ora 创建，只要没有可信 ownership record，也必须作为 Preserved State 处理。

后续如需接管既有配置，必须设计显式 adoption 流程；首次 migration、普通 reconcile 和插件卸载都不能隐式接管。

## 不变量

1. Workspace Desired State 的事实源是 Ora 数据库，不是 Agent 配置文件。
2. 旧数据库只通过 transaction migration 升级，领域模型不承担历史 schema 兼容。
3. 高于当前程序支持版本的数据库必须拒绝打开。
4. Desired State 的完整替换和 generation 推进必须在同一 transaction 中完成。
5. Status 更新不能修改 Workspace generation。
6. Session 活动状态必须重新观察，不能从数据库旧快照推断安全空闲。
7. Agent adapter 不能读取持久化 JSON、数据库表或 migration 版本。
8. Secret 明文不得进入任何 Effect 持久化数据。
9. 没有可信 ownership record 的既有配置必须初始化为 Preserved State。
10. migration 无法确定性构造的新语义必须进入显式待配置状态，不能使用猜测值。
11. 数据库与外部配置之间的写入必须由 durable operation record 支持崩溃恢复。
