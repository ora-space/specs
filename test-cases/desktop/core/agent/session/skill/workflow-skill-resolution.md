# 工作流按 skill ID 解析 Skill 存储路径

决策依据：[Skill 按存储路径以会话清单交给 Agent](../../../../../../decisions/desktop/core/agent/session/skill/0-skill-catalog-from-storage-paths.md)
（`approved`）D8。实现尚未开始，本文件全部义务的证据状态为 `Missing`。

## Workflow Nodes Must Resolve Skills By ID At Dispatch

### 风险

工作流不再冻结 Skill 内容或路径。如果节点仍沿用创建运行时的路径，或升级前冻结的工作树相对 `packagePaths`，它会读到
已清理的副本或已删除的旧版本目录。如果 Skill 在运行中被卸载，节点退回到同名的其他 Skill，就会静默执行错误的说明。
启动检查如果仍要求 Agent 声明 Skill 交付能力，所有 Agent 都会被误拒。

### 前置状态

- 一个工作流有两个顺序执行的 Agent 节点，各启用一个 Skill：节点 1 用本地 Skill，节点 2 用插件 Skill。
- 另有一个与插件 Skill 同名的本地 Skill。
- 另有一个升级前创建、冻结记录中带工作树相对 `packagePaths` 的运行。
- 执行节点的 Agent 从未声明 Skill 交付能力。

### 触发

1. `start` 运行；节点 1 完成后，把插件升级到新版本，再派发节点 2。
2. 再次 `start`；节点 1 完成后卸载节点 2 用的插件，再派发节点 2。
3. 在缺少某个已启用 Skill 时，对运行执行 `start`、`restart` 和切换快照的 `resume`。
4. 恢复升级前创建的运行，派发它的下一个节点。

### 必须成立

- 运行冻结的只有节点 ID、skill ID 和调用名。
- 启动、重启与恢复前的检查是：skill ID 能在 Catalog 中解析，且其存储路径下的 `SKILL.md` 存在并可解析；不满足时
  返回现有的 Skill 依赖错误，带节点标题，运行保持原状态。
- 每个节点派发时按 skill ID 重新解析当前存储路径；必需 Skill 块呈现调用名和 `SKILL.md` 绝对路径，说明要求读取并
  遵循它，前导 `/调用名` 保留。
- 插件升级后，节点 2 读到新版本目录下的 `SKILL.md`。
- 插件被卸载后，节点 2 以同一 Skill 依赖错误失败。
- 升级前的运行照常读取冻结记录，忽略其中的工作树相对 `packagePaths`，按 skill ID 解析。
- 未声明 Skill 交付能力的 Agent 照常执行带 Skill 的节点。
- 同一运行中两个不同 Skill 解析出相同调用名时，启动被拒绝。

### 禁止结果

- 冻结 Skill 内容、版本或路径，或按 revision 保留存储目录。
- 回退到同名的其他 Skill，或使用旧路径、物化副本路径。
- 因 Agent 没有声明 Skill 交付能力而拒绝启动。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| 冻结记录只含节点 ID、skill ID 与调用名 | 尚无实现 | `Missing` |
| 启动、重启、切换快照恢复按“Catalog 可解析且 `SKILL.md` 存在”检查，失败时用现有 Skill 依赖错误 | 尚无实现 | `Missing` |
| 派发时按 skill ID 解析，运行中升级后后续节点读到新路径 | 尚无实现 | `Missing` |
| 运行中卸载后节点以 Skill 依赖错误失败，不回退到同名 Skill | 尚无实现 | `Missing` |
| 升级前运行的工作树相对 `packagePaths` 被忽略 | 尚无实现 | `Missing` |
| 不再要求 Agent 声明 Skill 交付能力；调用名冲突仍被拒绝 | 尚无实现 | `Missing` |

### 决策依据

- Skill 会话清单决策 D8；派发预检中的 Skill 依赖检查由此修改。
- 不变量 10。
