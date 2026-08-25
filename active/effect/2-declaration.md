# Effect 声明

## 定位

Effect 使用声明式模型：Workspace 声明期望的 Skill 和 MCP 集合，Ora 针对每个 Agent target 持续比较 Desired State、Managed State 与 Observed State，并将目标环境收敛到当前 generation。

声明和收敛均以 Workspace 为隔离边界。不同 Workspace 即使选择相同 Skill、MCP 或 Agent 插件，也分别维护 generation、ownership 和运行状态，不能互相更新或清理资源。

## Workspace 声明

```text
WorkspaceEffect {
  workspace_id,
  generation,
  spec: WorkspaceEffectSpec,
  status: WorkspaceEffectStatus
}

WorkspaceEffectSpec {
  skills: Map<SkillName, SkillDefinition>,
  mcps: Map<McpId, McpDefinition>
}
```

`generation` 是当前 Desired State 的单调递增 revision。Skill 或 MCP 的增加、删除或定义变化都会产生新的 generation；运行状态和 reconcile 结果变化不会修改 generation。

`WorkspaceEffectSpec` 只声明期望状态，不表示任何 Agent 已经应用这些配置。

## SkillDefinition

Workspace 中的 Skill 使用稳定名称和 `SKILL.md` 内容摘要建模：

```text
SkillDefinition {
  name: SkillName,
  skill_md_digest: Digest,
  source: InstalledSkillRef
}
```

`name` 是 Skill 的逻辑身份，`skill_md_digest` 是期望内容版本。更新 `SKILL.md` 只改变摘要，不改变 Skill 身份，不能被解释成删除旧 Skill 后创建一个无关的新 Skill。

`source` 将定义绑定到 Ora 已安装并校验的精确 Skill 制品。摘要用于比较和校验，不能代替可信的制品定位信息，也不能要求 Agent adapter 根据摘要反查文件。

同一个 Workspace 的 Desired State 中不能出现两个同名 Skill。目标环境中已存在的同名 Skill 如果没有 Ora ownership record，则属于 Preserved State；Ora 不能仅凭名称或摘要接管它。

## McpDefinition

MCP 在进入 Effect Desired State 前必须转换成与目标 Agent 无关的 Ora definition：

```text
McpDefinition {
  id: McpId,
  exact_version,
  transport: StdioDefinition | HttpDefinition,
  settings,
  secret_refs,
  definition_digest: Digest
}
```

`McpDefinition` 是规范化声明，不包含 Claude Code、Codex、OpenCode 等目标 Agent 的配置格式。Agent adapter 负责将它安全物化为目标格式。

Secret 必须保持为 `SecretRef`，不能进入 definition digest 的明文输入，也不能作为明文写入 Workspace 或目标 Agent 配置。目标 Agent 无法安全表达 SecretRef 时，本次 reconcile 失败。

`definition_digest` 标识会影响目标物化结果的规范化内容。MCP Setting、transport、精确版本或安全引用变化时摘要必须变化。

## AgentTarget

Effect 的独立收敛目标是一个 Workspace 与一个 Agent 插件的组合：

```text
AgentTargetId {
  workspace_id,
  agent_plugin_id
}

AgentPlugin
└── AgentTarget(workspace_id)
    ├── AgentInstance A
    │   ├── Session 1
    │   └── Session 2
    └── AgentInstance B
        └── Session 3
```

同一 Workspace 的不同 AgentTarget 独立 reconcile。一个 Agent 插件因 Session 正在工作而等待时，不能阻止其他 Agent 插件应用同一 Workspace generation。

一个 AgentTarget 下的全部 AgentInstance 都由对应 Agent 插件管理。只要目标 Agent 的 Skill 或 MCP 配置以 Workspace 为作用域，该 target 下所有实例都必须停止使用旧配置后才能物化新 generation。

## Session 活动状态

Ora 必须识别每个 AgentInstance 下是否有 Session 正在工作，不能只根据当前是否收到流式文本判断：

```text
SessionActivity
├── Idle
├── Working {
│     turn_id
│   }
└── StoppingTurn {
      turn_id
    }
```

等待模型、执行工具、调用 MCP、等待权限交互和流式输出都属于 `Working`。用户请求停止输出后进入 `StoppingTurn`；只有 Agent adapter 确认当前 turn 已经终止，Session 才进入 `Idle`。

用户停止当前输出与 Effect 重配使用的 Session 生命周期操作是两个不同动作：

- 用户停止当前输出是 `turn/cancel`；
- 空闲 Session 为重配而停止是 `session/close`；
- 配置应用完成后恢复 Session 是 `session/resume`。

`session/close` 不能被用来隐式取消正在工作的 turn。

## AgentTargetStatus

```text
AgentTargetStatus {
  desired_generation,
  observed_generation,
  applied_generation,
  ready_generation,
  phase: AgentTargetPhase,
  instances: Map<AgentInstanceId, AgentInstanceStatus>
}

AgentTargetPhase
├── Current
├── WaitingForIdle { busy_sessions }
├── Quiescing
├── StoppingSessions
├── Applying
├── ResumingSessions
└── Degraded { error, failed_generation }
```

三个 generation 字段表达不同事实：

- `observed_generation`：reconciler 已经观察并处理到的最新 Desired State；
- `applied_generation`：对应 Skill/MCP 配置已经成功物化到目标环境；
- `ready_generation`：该 generation 的配置已应用，原有 Session 已成功恢复，并且 target 可以接受新的 turn。

配置已经写入但 `session/resume` 失败时，`applied_generation` 可以前进，但 `ready_generation` 不能前进，phase 必须为 `Degraded`。不得使用单一 revision 掩盖这种部分完成状态。

## Reconcile 时序

Skill 或 MCP 变化后，AgentTarget 按以下时序收敛：

```text
Workspace generation changes
        ↓
inspect all AgentInstance and Session activity
        ↓
any Session is Working or StoppingTurn?
        ├── yes → WaitingForIdle; do not mutate target configuration
        └── no
              ↓
          enter Quiescing and stop admitting new turns
              ↓
          session/close for all affected idle Sessions
              ↓
          reconcile Skill and MCP definitions atomically
              ↓
          session/resume
              ↓
          allow new turns and advance ready_generation
```

进入 `Quiescing` 是本次 reconcile 的线性化点。检查到全部 Session 空闲后，必须先阻止新的 turn 进入，再调用 `session/close`；否则新的工作可能在空闲检查与停止之间启动。

Effect 不主动打断正在工作的 Session。配置变化只能等待工作自然结束，或等待用户取消且 Agent adapter 确认 turn 已终止。

## Managed State

每个 AgentTarget 分别维护 Ora ownership ledger：

```text
ManagedAgentState {
  target_id: AgentTargetId,
  applied_generation,
  skills: Map<ManagedIdentity, ManagedSkill>,
  mcps: Map<ManagedIdentity, ManagedMcp>
}

ManagedSkill {
  name,
  skill_md_digest,
  target_locator,
  applied_fingerprint
}

ManagedMcp {
  id,
  definition_digest,
  target_locator,
  applied_fingerprint
}
```

名称、摘要、路径和配置内容都不能单独证明 ownership。只有存在于当前 Workspace、当前 AgentTarget 的 Managed State 中，并通过 `managed_identity` 匹配的资源，Ora 才能更新或删除。

reconcile 必须读取目标环境的 Observed State，保留所有没有匹配 ownership record 的 Skill、MCP 和其他配置。目标格式无法同时表达 Ora-managed 资源与同名 Preserved State 时，reconcile 必须报告冲突，不能覆盖既有配置。

## 不变量

1. Skill 的名称是逻辑身份，`SKILL.md` 摘要是内容版本；摘要变化不能改变 ownership 身份。
2. MCP 必须先转换为 Ora 的 `McpDefinition`，Agent-specific 格式只能出现在 adapter 的物化边界之后。
3. Secret 在声明和持久化状态中始终保持为 `SecretRef`。
4. AgentTarget 以 `Workspace × AgentPlugin` 为边界独立 reconcile。
5. AgentTarget 下任一 Session 不为 `Idle` 时，不得开始 Skill/MCP 配置物化。
6. 进入 `Quiescing` 后必须阻止新 turn，直到 reconcile 成功恢复或明确失败。
7. `session/close` 只作用于空闲 Session，不得隐式取消用户正在进行的工作。
8. 只有成功恢复并重新接受 turn 后，`ready_generation` 才能推进。
9. Ora 只能更新或删除当前 AgentTarget 的 ownership ledger 中明确记录的资源。
