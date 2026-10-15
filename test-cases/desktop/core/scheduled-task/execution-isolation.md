# Scheduled Task 执行隔离

## Accepted Runs Must Freeze A Revision And Start A New Session

- 风险：编辑任务改变在途运行，或复用交互式会话导致上下文、权限和结果归属串线。
- 前置状态：任务 revision R 被接纳，随后任务可编辑为 R+1，且可能已有其他会话。
- 触发：Scheduled Run 启动目标 Prompt。
- 必须成立：run 持久冻结 R 的 Prompt、Project、agent/model 和 approval policy；每次 run 新建 session 并记录其 ID；后续编辑
  只影响未来 occurrence。
- 禁止结果：执行时重新读取最新 Prompt、复用现有 session，或用 task 当前 revision 重写历史。
- 决策依据：根决策 D2、D3 与不变量 6、7。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| 定义内容变化增加 revision，纯名称变化不增加 revision | Prompt/schedule 已由 `prompt_and_schedule_changes_increment_revision_without_reconsent` 部分覆盖；名称路径尚无证据 | `Partial` |
| run 持久冻结定义快照并调用真实 application use case | 尚未实现 | `Missing` |
| 每个 Scheduled Run 创建不同 session ID 且不复用交互式会话 | 尚未实现 | `Missing` |

## Every Local Run Must Use An Independent Worktree

- 风险：无人值守运行覆盖用户工作区，多个 run 互相污染，或清理逻辑删除仍有结果的 worktree。
- 前置状态：Git Project 可创建 worktree；run 最终可能无改动或有改动。
- 触发：local run 准备、执行并终止。
- 必须成立：每个 run 使用独立 worktree；无改动时删除；有改动时保留路径并提供打开/丢弃；运行历史持有恢复入口；保留数量有界。
- 禁止结果：直接使用 main workspace、不同 run 共用 worktree，或无条件删除有改动结果。
- 决策依据：根决策 D4、D8 与不变量 10、12。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| run 创建独立 worktree 且 Session cwd 指向它 | 尚未实现 | `Missing` |
| 无改动自动删除，有改动保留并可打开/丢弃 | 尚未实现 | `Missing` |
| 有界保留清理只选择最旧且未使用的保留项 | 尚未实现，具体上限待实现确定 | `Missing` |

## Busy And Capacity Limits Must Skip Without Queuing

- 风险：高频任务无限排队或并发耗尽额度；同一任务重叠修改同一 Project。
- 前置状态：同一 task 已有 active run，或全局已有 4 个 local Scheduled Runs。
- 触发：新的到期 occurrence 被评估。
- 必须成立：同 task 记录 `skipped_busy`；全局第 5 项记录 `skipped_capacity`；均不创建 session、不排队、不自动重试。
- 禁止结果：静默丢弃、等待槽位后补跑、并发上限可由用户调高，或把两种原因合并为 failed。
- 决策依据：根决策 D6、D7。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| 同任务 active run 阻止第二次启动并持久 `skipped_busy` | 尚未实现 | `Missing` |
| 全局 4 个 active run 时第 5 项持久 `skipped_capacity` | 尚未实现 | `Missing` |
| skipped occurrence 不进入 retry/queue，也不创建 session | 尚未实现 | `Missing` |

## Timeout And Failure Must Be Terminal Without Automatic Retry

- 风险：无人值守 run 无界占用资源，或失败后自动重试重复产生副作用和费用。
- 前置状态：run 已运行，目标可能失败、无响应或超过 60 分钟。
- 触发：target 返回错误或固定 timeout 到期。
- 必须成立：60 分钟时取消目标并记录 `timeout`；其他错误映射明确 failure reason；同 occurrence 不自动重试。
- 禁止结果：可配置更长 timeout、后台自动重启 session，或把取消失败记作成功。
- 决策依据：根决策 D7、D8。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| 固定 60 分钟触发真实取消写入并终止 run | 尚未实现 | `Missing` |
| failure/timeout 不注册自动 retry 且同 occurrence 不新建 run | 尚未实现 | `Missing` |

