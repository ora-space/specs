# 持久 Thread

依据[已批准的 Thread 根决策](../../../decisions/cloud/thread/0-durable-agent-thread-with-user-turns.md)。
本文件登记 Thread 条目的顺序、用户轮次与结束条件。证据来自 `ora-space/cloud` 的 `integration` 包（真实 HTTP/gRPC 与
PostgreSQL）和 `internal/core` 包（白盒 PostgreSQL 测试，经生产钩子驱动接管与派发扫描）；Issue 页 Thread 面板另有前端组件测试
（`frontend/src/features/issues/thread/*.test.*`），但本文件的义务都是 Cloud 端承诺，未把它们登记为证据。

跨仓库真实链路证据来自 cluster 的 M2 验收（`task agent:session:acceptance` 与 `task agent:session:browser`，见
[Agent 会话验收](https://github.com/ora-space/cluster/blob/main/docs/testing/agent-session.md)）：真实
Gateway/Cloud/Controller/Node 与 Deno 运行的 echo agent 上，@agent 评论创建运行；Thread 依次出现 `meta`、echo 回复与
`turnEnded` 并进入 `idle`；订阅者收到 SSE `issue_run.thread_appended`；追加的用户轮次变为 `delivered` 并得到回复；结束会话后
`threadState` 为 `ended`，会话结果为 `agent_session_ended/user_ended`；未配置对象存储时交付记为 `skipped`，运行 Workspace
被删除，运行以 `phase = done` 完成。该验收发现并修复了（ora-space/cloud#46）Cloud 把同一轮里 agent 的记录当作回显丢弃的问题，
对应回归测试为 `TestThreadTakeoverKeepsTheTurnsOwnRecords`。验收失败只能说明端到端链路断开，不能单独定位到下列某条义务，
因此只作为补充引用，不据此标记 `Covered`。

## Thread Entries Are Gap Free And Taken Over Once

### 风险

Node 事件重放产生重复条目，或乱序接管把对话顺序排错；前端的分页游标出现空洞。

### 前置状态

运行处于 `running`，会话执行已登记。

### 触发

`TakeOverThreadEvents` 接管批次；同一批次重放；同序号不同内容；批次首序号出现缺口。

### 必须成立

- 条目 `seq` 在 Thread 内从 1 连续递增，来源包括 Node、用户与系统。
- 重放不产生新条目；同序号不同内容与缺口以 `CONFLICT` 拒绝，整批不写入。
- 条目内容与 Node 记录逐字节相同。

### 禁止结果

重复条目；`seq` 空洞或重排；部分写入的批次。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 连续与幂等 | `integration/agent_run_thread_events_test.go::TestThreadAppendedDuplicatesAndLossAreHarmless`：Node 批次写入 seq 2，原样重放不追加条目，随后用户轮次写入 seq 3，读回为 1..3；`internal/core/agent_run_thread_lifecycle_db_test.go::TestThreadTakeoverUserTurnLifecycleIsOneWay`：两个 Node 批次与一个用户轮次交错后绕过收据直接重放钩子，seq 仍为 1..4、条目逐字节不变；`internal/core/agent_run_thread_lifecycle_db_test.go::TestThreadTakeoverKeepsTheTurnsOwnRecords`：首轮 `meta`/回复/`turnEnded` 与系统首条提问构成连续 1..4，只跳过用户消息回显；`integration/agent_run_thread_takeover_test.go::TestAgentRunThreadTakeoverOverGRPC`：经真实 gRPC 重放同一批次，收据与条目数都不变 | Covered |
| 冲突整批拒绝 | `integration/agent_run_thread_takeover_test.go::TestAgentRunThreadTakeoverGRPCRejections`：首序号缺口与批内缺口都返回 `CONFLICT`，收据为 0、条目只剩 seq 1；`integration/agent_run_control_test.go::TestThreadEventsAreTakenOverInOrder`：含缺口的批次被拒绝且不写收据。缺口：没有测试覆盖 Thread 批次中“同序号不同内容”返回 `CONFLICT` 且整批不写入（`TestSessionEndedRefusalsLeaveNoTrace` 只覆盖会话终态事件的同序号不同字节，不是 Thread 批次） | Partial |

### 决策依据

Thread D1、D2，不变量 1、2、6。

## User Turns Are Persisted Before Delivery And Settle Exactly Once

### 风险

用户消息因 Controller 重启丢失、被执行两次，或会话结束后仍在新会话中执行。

### 前置状态

Thread 处于 `pending | active | idle`。

### 触发

带 `Idempotency-Key` 追加消息；同一键重试；Thread 进入 `ending` 后追加；会话结束时仍有排队轮次。

### 必须成立

- 条目与 `SubmitUserTurn` 命令在同一事务写入，`turn_id` 由 Cloud 生成。
- 同一 `Idempotency-Key` 返回同一条目；`ending | ended` 时返回 `409 thread_closed`。
- 接管到带该 `turn_id` 的用户消息记录时轮次变为 `delivered`；会话终态事务中仍 `queued` 的轮次变为 `discarded`。

### 禁止结果

命令先于条目存在；同一键产生两个轮次；`discarded` 的轮次之后被执行。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 同事务写入与幂等 | `integration/agent_run_thread_message_test.go::TestThreadMessagePostAcceptsPendingActiveAndIdle`：`pending/active/idle` 下 POST 后恰好一个 `queued` 条目与一个未送达的 `submit_user_turn` 命令，三处 `turn_id` 相同且由 Cloud 生成；`::TestThreadMessageWriteFailureRollsBackEverything`：命令写入失败时条目、命令、幂等记录与 Thread 状态全部回滚，重试后各一条；`::TestThreadMessagePostReplaysUnderTheSameKey`：同键重放返回原响应，条目与命令数不变；`::TestThreadMessageConcurrentSameKeyCreatesOneTurn`：同键并发只产生一个轮次 | Covered |
| 关闭后拒绝 | `integration/agent_run_thread_message_test.go::TestThreadMessagePostRejectsClosedAndUnknownRuns`：`ending` 与 `ended` 时 POST 返回 `409 thread_closed`，不写条目与命令；`integration/agent_run_session_end_test.go::TestSessionEndedEndsThreadDiscardsQueuedAndReleasesDelivery`：会话终态后再 POST 返回 `thread_closed` | Covered |
| 轮次结算 | `internal/core/agent_run_thread_lifecycle_db_test.go::TestThreadTakeoverUserTurnEchoMarksDelivered`：带 `turn_id` 的用户消息记录把轮次置为 `delivered`，不新增条目与命令；`::TestThreadTakeoverKeepsTheTurnsOwnRecords`：同一轮的回复与 `turnEnded` 带 `turn_id` 仍是普通条目，只有用户消息记录结算轮次；`integration/agent_run_session_end_test.go::TestSessionEndedEndsThreadDiscardsQueuedAndReleasesDelivery`：终态接管把仍 `queued` 的轮次置为 `discarded`，已 `delivered` 的不变；`::TestDiscardedAppearsOnlyOnTheSessionTerminalTakeover`：`ending` 时轮次仍为 `queued`，只有终态接管写 `discarded`。M2 真实链路验收中追加轮次变为 `delivered` 并得到 echo 回复 | Covered |

### 决策依据

Thread D3，不变量 3、4。

## Idle Sessions End On Database Time

### 风险

依赖 Controller 或 Node 时钟判断空闲，时钟偏差导致会话过早结束或永不结束。

### 前置状态

Thread 为 `idle`，`idle_since` 已记录。

### 触发

数据库时间越过 `idle_since + thread_idle_timeout`；在此之前用户追加消息。

### 必须成立

- 越过窗口后派发循环把 Thread 置为 `ending` 并放出 `EndSession{idle_timeout}`，只放出一次。
- 窗口内追加消息使 Thread 回到 `active`，不结束。
- 只有成功结束且无待执行消息的轮次进入 `idle`；取消的 `turnEnded` 不建立空闲窗口。
- Agent 请求失败或流意外关闭时，Node 以有限安全分类结束会话。Cloud 在终态接管事务中关闭 Thread，
  丢弃排队消息并追加一个 `source=system/kind=session_failed` 条目；重放不追加第二个条目。
  原始 Node 历史保留原样，失败条目不携带供应商诊断、认证头或错误链。

### 禁止结果

用应用进程时钟判断；重复放出结束命令。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 空闲期满结束 | `internal/core/agent_run_thread_end_db_test.go::TestIdleScanEndsAnExpiredThreadOnce`：以数据库 `now()` 回拨 `idle_since`，窗口内扫描不结束，越过窗口后 Thread 为 `ending`、恰好一个 `EndSession{idle_timeout}`，再扫描两次不新增；`::TestThreadEndingConcurrencyMatrix`（two idle ticks emit one request）：并发两次扫描只放出一个请求；`internal/core/agent_run_thread_lifecycle_db_test.go::TestThreadTakeoverIdleWhenTurnEndedAndNoQueuedTurn`：`idle_since` 取数据库时钟 | Covered |
| 追加消息重置 | `integration/agent_run_thread_message_test.go::TestThreadMessagePostAcceptsPendingActiveAndIdle`（idle 子用例）：POST 后 `thread_state = active`、`idle_since` 清空，唯一的命令是 `submit_user_turn`；`internal/core/agent_run_thread_end_db_test.go::TestIdleScanLeavesUnconfiguredAndUnfinishedWindowsAlone`：过期窗口内先写入用户轮次，之后的扫描不放出 `EndSession` | Covered |

### 决策依据

Thread D4，不变量 5。
