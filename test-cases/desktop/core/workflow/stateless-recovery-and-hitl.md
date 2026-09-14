# 无状态恢复与循环内人工介入

反应式调度承诺"任意时刻崩溃可恢复"与"HITL 停靠数天"。循环引入后这两条承诺必须在复合
节点上继续成立。

## Loop State Is Recovered From Persistence After A Crash

### 风险

Loop 运行时若在崩溃后无法从持久化行重建"当前第几轮、区域进行到哪"，循环要么从头重跑
（浪费已完成的轮次），要么卡死在 Running。

### 前置状态

- 一个 Running 的 run，其 Loop 节点行 Running，第 k 轮区域行部分完成。
- 进程在任意时刻被杀（含区域节点 dispatch 后、回调前）。

### 触发

进程重启，boot sweep 扫描到该 run。

### 必须成立

- boot sweep 把 Running 的 Loop 行交还其运行时续算。
- 运行时从区域行的 `max(iteration)` 推导当前轮，继续第 k 轮的调度（补齐未完成节点），不
  重跑已 Succeeded 的轮内节点。
- 区域内的非交互 `Pending` 行按既有 fail-closed 规则处理，不误判为可恢复。

### 禁止结果

- 循环从第 1 轮重跑。
- run 永久滞留 Running 无 in-flight 节点。
- 恢复路径依赖崩溃前的内存状态或日志文件。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
|---|---|---|
| 第 k 轮中途崩溃后恢复到第 k 轮续算，已完成轮内节点不重跑 | 尚无直接证据 | `Missing` |
| 既有"Running 无在途节点则 resume"语义在含 Loop 的 run 上成立 | 尚无直接证据（既有 `reconcile_running_workflow_runs` 测试不含 Loop） | `Missing` |

### 决策依据

[节点运行时编排决策](../../../../decisions/desktop/core/workflow/0-node-runtime-orchestration.md)
不变量 2 与 D4 末段。

## Interactive Nodes Inside A Region Park And Resume Through Existing Paths

### 风险

循环体内的交互节点若不能停靠 `Pending`，或人工完成后循环无法推进，HITL 与循环的组合就不可
用——而"reviewer 打回"恰恰是最核心的循环场景。

### 前置状态

- 循环体内的 Agent 节点声明 `interactive: true`。
- 该节点第一轮以 EndTurn 结束。

### 触发

第一轮结束；随后人工追问若干轮；最终人工完成该节点。

### 必须成立

- 节点停靠 `Pending`（AwaitingInput），迭代投影 `has_in_flight` 计为在途，外层不误判 run
  结束。
- 人工追问经既有 `begin_human_turn`/`end_human_turn` 路径，节点在 Running/Pending 间翻转。
- 人工完成的回报经统一 sink 进入引擎，触发调度波；调度波把 Running 的 Loop 交还运行时，
  循环自动推进决策或下一轮。
- 终态节点的会话只读，拒绝新 prompt。

### 禁止结果

- 循环体内交互节点被当作普通节点自动重试或跳过。
- 人工完成后需要专门的通知通道才能推进循环（决策明确禁止）。
- 终态节点的 session 仍可接收 prompt。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
|---|---|---|
| 循环体内交互节点停靠后人工完成，循环经调度波自动推进 | 尚无直接证据 | `Missing` |
| 终态节点会话拒绝新 prompt（既有行为在区域内保持） | 既有 `interactive` 路径测试不覆盖区域内场景 | `Missing` |

### 决策依据

同上决策 D5。
