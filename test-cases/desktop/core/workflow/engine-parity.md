# 引擎重构行为等价

重构的第一承诺是"不改行为"：引入 `NodeRuntime` 注册表拆解引擎核心时，既有四种节点类型的
冻结图必须逐字节等价地执行。本文件守住建构在旧行为之上的全部后续承诺的地基。

## Graphs Without New Node Types Execute Identically After The Refactor

### 风险

Stage 1 引入注册表时，任何一个控制节点的同步完成逻辑、Agent 异步分发路径或状态转移顺序发生
细微偏移（例如 Condition 求值时机、Output 汇聚规则、`current_nodes` 锚点更新顺序），都会在
既有用户的工作流上产生不可见的执行差异。这类回归最危险之处在于它不报错——run 照常完成，但
结果不同。

### 前置状态

- 一批覆盖既有四种节点类型（start/agent/condition/output）的冻结图，含分支、扇入与交互节点。
- 重构前的引擎在这些图上的执行结果（节点状态序列、输出、变量池终态）已由既有测试固定。

### 触发

Stage 1 重构合入后，同一批既有引擎测试不修改断言地重新执行。

### 必须成立

- 既有引擎测试全部原样通过，包括分支投影、扇入汇合、HITL 停靠与恢复路径。
- Start/Output/Condition 经快执行器完成后的持久化状态与重构前一致（status、output、
  payload、finished_at 语义不变）。
- Agent 节点仍由每 node-run 一个专属 session 驱动，绑定发布时机不变。

### 禁止结果

- 为通过测试而修改既有断言。
- 任何"顺带改进"造成的行为偏移（例如调度波内节点启动顺序变化影响了可观察输出）。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
|---|---|---|
| 扇入节点在全部前驱完成前不就绪，任一 active 即就绪的行为不变 | `ora_application::workflow_run::engine::tests::fan_in_is_not_ready_until_every_predecessor_completes`、`ora_application::workflow_run::engine::tests::ready_set_handles_fan_in` | `Partial`（证明重构前基线；等价性待 Stage 1 后同一测试原样通过） |
| 其余既有引擎测试（分支、HITL、恢复）在 Stage 1 后原样通过 | 既有 `engine/tests.rs` 全量 | `Missing`（待重构合入后执行） |

### 决策依据

[节点运行时编排决策](../../../../decisions/desktop/core/workflow/0-node-runtime-orchestration.md)
不变量 9 与 Stage 1 完成条件。

## The Scheduling Layer Contains No Node Type Literals

### 风险

若 `run_schedule` 及其同层函数残留节点类型字面量匹配，注册表就是装饰性的——下一个节点类型
仍会绕过它写进引擎核心，D1 的扩展承诺失效。

### 前置状态

Stage 1 重构合入，Start/Condition/Output 迁移为快执行器，`WorkflowRunNodeExecutor` 包装为
Agent 运行时。

### 触发

对 `crates/application/src/workflow_run/engine/` 的引擎核心文件做静态检查。

### 必须成立

- 引擎核心的调度路径（`run_schedule` 及其同层函数）不出现 `NodeType` 字面量匹配。
- 节点类型到运行时的映射只存在于注册表装配处。

### 禁止结果

- 以 `if node_type == ...` 形式出现的任何调度分支（注册表装配与测试除外）。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
|---|---|---|
| 引擎核心无节点类型字面量的静态约束 | 尚无直接证据；候选形式为专项测试或 lint 检查 | `Missing` |

### 决策依据

同上决策 D1 的验收判据。
