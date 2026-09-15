# 运行时形态与完成回报

三种执行形态共享同一个回报入口与同一把 per-run 串行门。本文件守住锁内安全、session 归属与
回报幂等。

## Swift Runtimes Perform No IO Under The Run Lock

### 风险

快执行器若在 per-run 排他锁内执行 IO 或无界等待（一次 30 秒的 HTTP 调用），该 run 的全部
调度被冻结，其他节点的完成回调排队阻塞。

### 前置状态

- 一个含多个并发在途节点的 run。
- 某快执行器节点（Start/Condition/Output/聚合）进入调度波。

### 触发

快执行器执行。

### 必须成立

- 执行在锁内同步完成，无任何网络、文件或进程 IO，工作量有界。
- 需要 IO 的节点形态必须以异步执行器实现（`dispatch` 立即返回）。

### 禁止结果

- 快执行器发起网络请求、文件读写或子进程调用。
- 异步执行器持有 per-run 排他锁跨越任何等待。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
|---|---|---|
| 快执行器实现无 IO 的静态或测试约束 | 两层约束：(1) `SwiftNodeRuntime` trait 设计约束——`complete_running` 只接收内存态 `SwiftCompletion`（node-run 行、变量池、run 输入），不向实现递入任何仓库句柄或异步能力，杜绝"顺手"IO；(2) 可执行源码约束 `ora_application::workflow_run::engine::tests::node_runtime_module_performs_no_io_or_waiting`——扫描 node_runtime 模块全部源文件（含未来新增的运行时文件），断言无 `std::fs`/`std::net`/`std::process`/`std::thread`/`tokio`/`block_on`/`sleep`/持久化句柄等 IO 与等待原语。签名不构成静态保证（Rust 无法阻止实现自行调用 std 的 IO），该保证由源码扫描测试承担；有界工作量仍属评审义务 | `Verified` |

### 决策依据

[节点运行时编排决策](../../../../decisions/desktop/core/workflow/0-node-runtime-orchestration.md)
不变量 3 与 D2 表格。

## Only The Agent Runtime Owns Sessions

### 风险

若非 Agent 运行时创建并绑定 session，Theater 的会话列表、`find_node_run_by_session_id` 寻址
与 HITL 路径会把不持有 ACP 会话的节点误当交互节点。

### 前置状态

任意节点形态的 node-run 行被查询。

### 触发

运行时 dispatch 与完成。

### 必须成立

- 只有 Agent 运行时创建 session 并经 `bind_node_run_session` 绑定。
- 其他运行时（聚合、Code、HTTP、远程、迭代）的 `workflow_node_runs.session_id` 保持 NULL。

### 禁止结果

- 非会话节点持有 session_id。
- 迭代运行时为驱动区域而创建会话（区域内的 Agent 轮次各自由 Agent 运行时创建）。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
|---|---|---|
| 非 Agent 运行时的 node-run 行 session_id 为 NULL | `ora_backend::workflow::run::engine::tests::control_node_runs_keep_their_session_id_null`（start→condition→output 全部快执行器完成的 run，所有 node-run 的 session_id 为 NULL） | `Verified` |
| 迭代节点行（含各轮区域行）session_id 为 NULL | 尚无直接证据 | `Missing` |

### 决策依据

同上决策不变量 8 与 D2 末段；迭代运行时继承该约束，驱动区域不经会话。

## Late And Duplicate Completion Reports Are Idempotent No Ops

### 风险

异步执行器完成与 run 取消、人工完成竞态时，晚到的回调若被接受会复活已终态的节点或重复写
入变量池。

### 前置状态

- 某 node-run 已因取消或重复回报进入终态。
- 同一 node-run 的第二个完成回报到达。

### 触发

sink 收到晚到/重复回报。

### 必须成立

- 仓库幂等拒绝：`NotRunning`/`NotFound` 结果成为 no-op，不产生任何状态或变量池写入。
- 该行为对全部运行时形态一致（含未来的异步执行器）。

### 禁止结果

- 晚到回报改写终态节点的 output 或变量池。
- 重复回报使 `current_nodes` 锚点二次变更。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
|---|---|---|
| 晚到/重复完成回报被幂等拒绝 | `ora_backend::workflow::run::engine::tests::duplicate_completion_reports_are_idempotent_no_ops`（第二次 complete_node 无状态变更、无失效事件；既有 operations 测试 `concurrent_completions_commit_once_and_leave_the_session_read_only` 亦覆盖并发完成仅提交一次） | `Verified` |

### 决策依据

同上决策 D2 与既有 `WorkflowRunCallback` 串行化契约。
