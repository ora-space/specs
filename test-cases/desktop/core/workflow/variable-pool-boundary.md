# 变量池写入边界

变量池是节点间唯一的值传递通道，其 fail-fast 校验是全系统的数据完整性底线。

## Every Node Form Writes Through Declared Owner And Type Validation

### 风险

新增节点形态（Code、远程、聚合、循环体写入）若绕过 `pool.set` 的声明、owner、类型与长度
校验直接改写池，未声明变量、类型漂移与超长注入会进入下游 prompt。

### 前置状态

- 变量池按冻结图声明 catalog（含 `{loop_id}.index`）。
- 各节点形态产生输出并写入。

### 触发

节点完成时的变量池提交。

### 必须成立

- 写入必须经声明的 owner；未声明 selector、类型不符、超过声明长度均使节点失败，run 终态。
- Code 节点输出、远程子工作流返回、聚合节点选择结果全部通过同一校验路径。
- 迭代副本与聚合（预留能力）同样不得绕过。

### 禁止结果

- 任何节点形态以捷径写入未声明或类型不符的值。
- 校验失败被降级为默认值或警告。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
|---|---|---|
| 既有 `pool.set` 校验语义（未声明/错 owner/类型不符/超长即拒）在重构后不变 | 既有 `variable_pool.rs` 测试族持续通过 | `Partial`（证明现状语义；新节点形态的写入路径待各自实现后补证据） |
| Code/远程/聚合节点输出经同一校验 | 尚无直接证据 | `Missing` |

### 决策依据

[节点运行时编排决策](../../../../decisions/desktop/core/workflow/0-node-runtime-orchestration.md)
不变量 4 与 D6。
