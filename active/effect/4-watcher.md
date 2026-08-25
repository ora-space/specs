# Effect Watcher

## 定位

Effect Watcher 负责在 Workspace Desired State 变化后调度 AgentTarget reconcile。它不保存 Desired State，不携带配置 payload，也不直接修改 Agent 配置；它只保证每个受影响的 AgentTarget 最终被唤醒并处理数据库中的最新 generation。

Watcher 使用 level-triggered 语义：是否需要工作由持久状态之间的差异决定，而不是由某个事件是否被成功消费决定。

```text
desired_generation > ready_generation
or unfinished EffectOperation exists
        ↓
AgentTarget requires reconcile
```

进程内通知只用于降低延迟。通知丢失、Watcher 重启或应用崩溃都不能使已提交的 Desired State 永久失去 reconcile 机会。

## 调度粒度

Watcher 的最小调度目标是 `AgentTargetId`：

```text
AgentTargetId {
  workspace_id,
  agent_plugin_id
}
```

同一 AgentTarget 在任一时刻最多运行一个 reconcile。不同 Workspace 或同一 Workspace 下的不同 Agent 插件可以受并发上限约束地独立 reconcile，不能因为一个 target 等待 Session 空闲而阻塞其他 target。

一个 AgentTarget 下的所有 AgentInstance 和 Session 仍由同一次 reconcile 统一观察和 quiesce，不能为同一份 Workspace-scoped Agent 配置并发执行实例级 reconcile。

## 触发来源

以下变化必须使相关 AgentTarget 获得一次 durable wakeup：

1. Workspace Skill 或 MCP Desired State 的 generation 前进；
2. 新 AgentTarget 或 AgentInstance 注册，需要应用当前 generation；
3. `WaitingForIdle` target 的 Session activity 发生变化；
4. Agent 插件从不可用恢复为可用；
5. transient reconcile failure 到达重试时间；
6. 用户显式请求重试；
7. 应用启动时发现 generation 尚未收敛或存在未终结 `EffectOperation`；
8. 目标配置发生外部变化，需要重新观察 ownership 或 conflict。

触发原因只用于诊断和调度策略。reconciler 不能根据事件中的旧 payload 执行，而必须重新读取数据库中的当前 Desired State、Managed State 和持久化 operation。

## Durable reconcile request

每个 AgentTarget 最多保留一个可合并的 durable request：

```text
EffectReconcileRequest {
  target_id: AgentTargetId,
  requested_generation,
  state: Pending | Claimed | Blocked | RetryScheduled,
  wake_reason,
  attempt_count,
  not_before,
  lease_owner,
  lease_expires_at,
  updated_at
}
```

概念表结构如下：

```sql
CREATE TABLE effect_reconcile_requests (
    workspace_id        TEXT NOT NULL,
    agent_plugin_id     TEXT NOT NULL,
    requested_generation INTEGER NOT NULL,
    state               TEXT NOT NULL,
    wake_reason         TEXT NOT NULL,
    attempt_count       INTEGER NOT NULL,
    not_before          INTEGER,
    lease_owner         TEXT,
    lease_expires_at    INTEGER,
    updated_at          INTEGER NOT NULL,
    PRIMARY KEY (workspace_id, agent_plugin_id)
);
```

同一 target 的多次 wakeup 使用 upsert 合并，`requested_generation` 只能保留较大值，不能为每次编辑追加无界队列。Watcher 处理的是“至少收敛到这个 generation”，不是按顺序重放每次用户操作。

`wake_reason` 不参与正确性判断。后续 wakeup 可以覆盖或合并诊断原因，但不能降低 requested generation。

## Desired State 提交

`WorkspaceEffectRepository::replace` 必须在推进 generation 的同一个数据库 transaction 中，为所有已注册 AgentTarget upsert reconcile request：

```text
begin transaction
  verify expected_generation
  replace Desired State
  generation += 1
  upsert requests for registered AgentTargets
commit
        ↓
best-effort in-process notify
```

数据库提交和 durable request 之间不能存在调用方负责衔接的缝隙。否则调用方可能在 generation 已提交后、调用 scheduler 前崩溃，留下永远没有事件的 Desired State。

没有 AgentTarget 的 Workspace 只保存 Desired State。以后创建 AgentTarget 时，注册操作必须在同一 transaction 中创建 status 和指向当前 generation 的 reconcile request。

Desired State 规范化后没有实质变化时不推进 generation，也不创建新的 request。Skill/MCP 删除与增加使用相同时序，不能走独立的命令式清理通知。

## Watcher module

Watcher module 对外保持小型 interface：

```text
EffectWatcher
├── start()
├── notify()
└── shutdown()
```

`notify()` 不接受 Workspace、generation 或 definition payload。它只唤醒 worker 重新 claim 数据库中已到期的 durable request，因而允许多个通知安全合并。

Desired State repository、Agent runtime 事件和重试 timer 在完成各自持久化更新后调用 `notify()`。调用失败或进程在调用前退出不会破坏正确性，因为 request 已经持久化。

Watcher implementation 内部依赖强类型 request store 和 reconciler interface：

```text
ReconcileRequestStore
├── claim_ready(worker_id, now, lease_until) -> [ClaimedRequest]
├── renew(claim, lease_until)
├── complete(claim, completed_generation)
├── block(claim, reason)
└── retry_at(claim, attempt_count, not_before, error_code)

AgentTargetReconciler
└── reconcile(target_id) -> ReconcileOutcome
```

调用方不能分别更新 request、status 和 lease 字段。状态转换、claim ownership 和条件删除都封装在 request store 中。

## 启动与安全扫描

Watcher 启动时必须先执行恢复扫描，再开始消费普通 wakeup：

1. 找出未终结 `EffectOperation` 并安排对应 target 恢复；
2. 找出 `desired_generation > ready_generation` 的 AgentTarget；
3. 找出 lease 已过期的 `Claimed` request；
4. 找出已到 `not_before` 的 transient retry；
5. 为缺失 request 的上述 target 补建 durable request；
6. 开始正常 claim 循环。

运行期间还必须执行低频 safety scan。它不是主要调度路径，只修复丢失的进程内 Session 事件、意外缺失 request 或状态漂移。正确性不能依赖高频轮询，也不能因为使用通知而取消安全扫描。

## Claim 与并发

claim 必须通过数据库 transaction 将 request 从可运行状态原子转换为 `Claimed`，同时写入 `lease_owner` 和 `lease_expires_at`。同一个 request 不能被两个 worker 同时成功 claim。

执行时间可能超过初始 lease 时，worker 必须续租。应用崩溃或 worker 放弃后不再续租，过期 request 可由后续 worker 重新 claim。外部副作用是否已经发生由 durable `EffectOperation` 恢复协议判断，不能仅凭 request lease 过期而重复写入或删除资源。

第一版即使只有一个 Ora 进程，也保留持久化 claim 语义，以覆盖 worker panic、应用崩溃和启动恢复。进程内 keyed lock 可以减少重复工作，但不能代替数据库 claim。

Watcher 必须设置全局并发上限。并发按 AgentTarget 隔离，不能持有数据库 transaction、全局 mutex 或 Workspace 锁跨越 Agent adapter 调用、Session 等待或文件写入。

## Generation 合并

worker claim request 后必须读取 Workspace 当前 generation，而不是假设 `requested_generation` 就是最新值：

```text
claim requested_generation N
        ↓
load current Desired State generation M, where M >= N
        ↓
reconcile generation M
```

在进入 `Quiescing` 前可以短暂 debounce 并合并连续编辑，但 debounce 只是延迟优化，不能作为可靠性机制。

进入 `Quiescing` 并阻止新 turn 后，reconciler 必须再次读取当前 generation，再为该不可变快照创建 `Prepared` operation。一旦 `Prepared` 已持久化，本次 operation 的 generation 和计划不能被后续编辑原地改变；它必须完成、恢复或进入 `RecoveryRequired`。

如果 reconcile 期间 Desired State 又产生了更高 generation，新的 upsert 会保留在同一 request 中。本轮完成时只能条件性清除不高于 `completed_generation` 的请求：

```text
requested_generation <= completed_generation
        → request may complete

requested_generation > completed_generation
        → keep request pending
```

第一版完成当前 operation 后恢复 Session，再处理后续 generation，避免持续编辑使 AgentTarget 无限停留在 `Quiescing`。后续可以优化为在有界时间内保持 quiesced 并连续应用，但不能牺牲可用性或改变 operation 的不可变性。

## Session 空闲唤醒

发现任一相关 Session 为 `Working` 或 `StoppingTurn` 时，reconciler 不进入 `Quiescing`，而是：

1. 将 AgentTarget phase 更新为 `WaitingForIdle`；
2. 将 request 标记为 `Blocked(ActiveSessions)`；
3. 释放 claim，不进行定时热重试；
4. 保留 requested generation。

Agent runtime 在 Session activity 变为 `Idle` 时检查所属 AgentTarget：如果该 target 的所有 Session 都已空闲且 Desired State 尚未收敛，则把 request 重新置为 `Pending` 并调用 `notify()`。

从“最后一个 Session 变为空闲”到“进入 Quiescing”之间仍可能有新 turn 到达，因此 Session 事件不是安全证明。reconciler 必须在同一个 AgentTarget admission gate 下重新检查所有 Session，并原子地从 accepting turns 转换到 `Quiescing`。

用户点击停止输出只触发 `turn/cancel`。Watcher 必须等待 Agent adapter 确认 turn 终止并观察到 `Idle`，不能把取消请求已发送当成空闲事件。

## Reconcile outcome

```text
ReconcileOutcome
├── Current { ready_generation }
├── WaitingForIdle { busy_sessions }
├── Blocked { error_code }
├── RetryAfter { error_code, delay }
└── RecoveryRequired { error_code }
```

Watcher 根据结构化 outcome 调度，不解析错误字符串：

- `Current`：条件性完成 request；若已有更高 generation，立即重新变为 Pending；
- `WaitingForIdle`：等待 Session activity 事件；
- `Blocked`：等待 Desired State、插件能力、用户配置或外部目标变化，不自动热重试；
- `RetryAfter`：持久化指数 backoff、jitter 和 `not_before`；
- `RecoveryRequired`：停止自动写入并等待明确修复或用户决策。

新的 generation 到达时必须清除旧 generation 的 transient backoff 并立即调度，因为新声明可能已经消除失败原因。对于 `RecoveryRequired`，新 generation 不能自动授权覆盖无法证明归属的外部状态；必须先执行恢复检查。

错误详情遵守 Effect 持久化的安全约束，只保存稳定错误码和受限诊断信息。

## 关闭

应用关闭时，Watcher 必须停止 claim 新 request，并等待或取消尚未开始外部副作用的任务。已经持久化 `Prepared` operation 的 reconcile 不能通过清除记录伪装成完成；它可以在安全点结束，并由下次启动恢复。

关闭不需要清空 durable request。未完成 request、未到期 retry、blocked target 和 operation 都必须保留，供下次启动扫描继续处理。

## 测试要求

实现至少覆盖：

1. Desired State 与 request 在同一 transaction 中提交，任一步失败都不留下半更新；
2. 连续多个 generation 合并为一个 target request，并始终处理最新 Desired State；
3. no-op replace 不推进 generation，也不唤醒 target；
4. 两个 worker 不能同时 claim 同一 AgentTarget；
5. claim lease 过期后可以恢复，但不会绕过 `EffectOperation` 重复副作用；
6. 进程内通知丢失后，启动扫描或 safety scan 仍能补建并执行 request；
7. 一个 target 等待 Session 空闲时，其他 target 可以继续 reconcile；
8. 最后一个 Session 变为空闲会重新调度，新的 turn 不能穿过 admission gate；
9. reconcile 期间 generation 再次变化时，旧完成结果不能删除新 request；
10. transient failure 遵守持久化 backoff，Blocked 和 RecoveryRequired 不会热循环；
11. 应用在 `Prepared`、外部写入后和 ownership 提交前分别崩溃时，都能按 fingerprint 恢复；
12. 应用关闭和重启不会丢失未完成 request。

## 不变量

1. Desired State 和 durable request 必须在同一数据库 transaction 中提交。
2. 通知只负责唤醒，不能作为 reconcile 的事实源。
3. Watcher 每次执行都重新读取当前 Desired State，不能消费事件携带的配置快照。
4. 同一 AgentTarget 同一时刻最多有一个有效 claim 和一个 reconcile。
5. 多次 wakeup 必须合并，requested generation 不能倒退。
6. Session 未全部空闲时不得进入 `Quiescing` 或修改目标配置。
7. 只有 admission gate 能证明从空闲检查到停止接受新 turn 之间没有竞态。
8. request lease 只证明 worker ownership，不能证明外部副作用是否完成。
9. 旧 generation 的完成不能清除更高 generation 的 request。
10. 应用重启必须恢复未终结 operation、过期 claim 和未收敛 generation。
