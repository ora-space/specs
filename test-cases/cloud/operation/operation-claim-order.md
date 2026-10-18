# Operation 领取顺序

`claim` 是 Cloud 决定 Controller 下一个处理哪个 Operation 的唯一位置，因此它内部的排序就是 Cloud 的
调度策略。本文件登记这一策略及其风险。依据[已批准的运行时 Workspace 生命周期决策](../../../decisions/cloud/operation/0-workspace-runtime-follows-desktop-node.md)
D2：clone 失败有界重试的前提是队列不被单一 Operation 占满。证据来自 `ora-space/cloud` 的 `internal/core`
（真实 PostgreSQL schema，直接驱动 `claim` 控制动作）。排序本身没有单独的 ADR 文本，它是 Cloud 自身的
调度契约，本节把它固定下来。

## The Claim Prefers Waiting Work And Serves The Work That Has Waited Longest

### 风险

排序把“最旧”等同于“`created_at` 最小”时，一个不断失败的 Operation 同时永远是最旧的：它的 clone 失败把
自己送回 `retry_wait` 并刷新到期时刻，下一次领取又发现它到期，于是同一个 Operation 被反复交回，永远
如此。排在它后面的等待工作始终可领取却从不被领取，Project 停在 `provisioning`，而队列看起来是空的。
另一侧的对称风险是：为了让等待工作优先而把 `running` 完全排除，会让一个 Controller 中途死掉的
Operation 再也无法被继任者接手。

### 前置状态

同一租户下多个 Project 各自有一个活 Operation（部分唯一索引 `one_project_operation` 允许一个 Project
一个活动 Operation），Controller 持有有效租约与 epoch。

### 触发

Controller 调用 `claim`，候选中有 `queued`、已到期的 `retry_wait` 与 `running` 的混合。

### 必须成立

- 排序的主键是“是否为 `running`”：任何等待中的工作都先于已在运行的 Operation，`running` 仍然留在候选
  集合里（没有别的等待工作时它照常被领取，这就是恢复）。
- 次键是每个候选**变为可运行**的时刻，而不是创建时刻：`queued` 自创建起可运行，`retry_wait` 自其
  `retry_at` 起可运行。因此一个刚到期、创建很早的重试是**较年轻**的等待工作，让位于一直在排队的
  `queued`。
- `retry_wait` 在其时刻之前不是候选。
- 未被选中的 Operation 不被写入：状态与版本都不变，它只是排在后面。
- 陈旧 epoch 的 claim 整体被拒（`stale_controller`），排序根本不可达，也不写任何行。

### 禁止结果

把 `created_at` 当作唯一的“最旧”；让 `running` 从候选中消失；领取一个尚未到期的 `retry_wait`；为了排序
而结算、改写或删除任何 Operation。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 等待中的工作先于 `running`，且被越过的 Operation 不被写入 | `internal/core/operation_claim_db_test.go` `TestClaimPrefersWorkThatIsNotAlreadyRunning`、`TestClaimDoesNotLetOneRunningOperationHoldEveryProjectBack`：`queued` 与 `running` 并存时前者被领取、后者的版本不变；多个 `queued` 依次按年龄被领取，卡住的 `running` 仍是候选 | Covered |
| `running` 仍是候选，恢复就是普通领取 | 同一文件 `TestClaimStillRecoversARunningOperationWhenNothingElseWaits`：唯一候选是 `running` 时被领取，状态、epoch、版本按既有 UPDATE 前进，且没有自行为它写 `error_code` | Covered |
| 次键是变为可运行的时刻而非创建时刻 | `TestClaimServesTheOldestWaitingWorkNotTheOldestOperation`（6 小时前创建、1 秒前到期的重试让位于排队 2 小时的 `queued`，且状态与版本不变，紧接着的下一次领取就轮到它）、`TestClaimDoesNotLetARetryingOperationHoldEveryProjectBack`（排队的两个 Project 依次被领取后才是重试） | Covered |
| 到期前不领取、到期后领取 | `TestClaimHonoursTheRetryDelay`：未到期时领取无结果且行未变；到期后被领取回 `running` | Covered |
| 陈旧 epoch 的 claim 被拒且不写入 | `TestClaimRejectsAStaleControllerEpoch`：`stale_controller`，候选 Operation 版本不变 | Covered |
| 持续到达下 `running` 的恢复会被推迟 | `TestClaimDefersRunningRecoveryUnderContinuousArrivals` 只钉住这一限制：到达停止后恢复立即发生，但在持续到达率下这一延迟无界。这是本排序未解决的调度风险，不是它建立的保证 | Partial |

### 决策依据

运行时 Workspace 生命周期 D2（clone 失败的三次预算使队列不再被单一 Operation 永久占满），不变量 5；
本排序是 Cloud 调度边界的最小改动，不改变 Controller 协议。
