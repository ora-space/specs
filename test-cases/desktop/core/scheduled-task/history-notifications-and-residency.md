# Scheduled Task 历史、通知与数据驻留

## History Must Preserve Durable Outcomes Across Disable And Delete

- 风险：UI 从当前时间推断状态，或禁用/删除 definition 时级联删除审计事实。
- 前置状态：task 已产生 run、skip 与 missed 历史。
- 触发：禁用、软删除或重启 Desktop 后重新读取列表。
- 必须成立：运行状态、failure reason、missed reason、revision、occurrence 和 session/worktree 引用来自持久事实；定义删除保留墓碑与历史；
  每个 task 只保留最新 100 条记录并删除更旧记录。
- 禁止结果：硬删除历史、按“现在已过 nextRunAt”临时生成 missed，或清理最新 100 条内的记录。
- 决策依据：根决策 D7 与不变量 3、8。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| definition CRUD 和软删除从列表隐藏但保留数据库行语义 | `ora_db::repository::scheduled_task::tests::definitions_round_trip_update_and_soft_delete` | `Partial` |
| run/missed 事实在禁用、删除和重启后保持 | 尚未实现 | `Missing` |
| 第 101 条历史写入后只清理最旧记录，保留最新 100 条 | 尚未实现 | `Missing` |

## Notifications Must Reflect Persisted Run Facts

- 风险：通知先于事务提交导致点击后没有历史，或漏报 failure/timeout/missed。
- 前置状态：run 开始或终止，或 scheduler 持久化 missed。
- 触发：对应事实成功提交。
- 必须成立：run start、failed、timeout、missed 各发一条桌面通知；开始通知点击进入新会话，终态通知定位对应历史；通知失败不改写运行事实。
- 禁止结果：通知充当状态源、事务回滚后仍发送成功通知，或同一事实重复通知。
- 决策依据：根决策 D7。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| start 通知在 run/session 持久化后发送并可导航 | 尚未实现 | `Missing` |
| failed、timeout、missed 各自通知且绑定唯一事实 | 尚未实现 | `Missing` |
| 通知发送失败不回滚或伪造运行状态 | 尚未实现 | `Missing` |

## Local Data Must Not Require Login Or Leave The Device

- 风险：本机路径、Prompt、run 结果或凭据引用被意外加入 Cloud 同步，或未登录用户无法使用 local task。
- 前置状态：Desktop 未登录或 Cloud 不可用，用户创建并运行 local task。
- 触发：definition/run 持久化、列表读取、应用重启及网络不可用。
- 必须成立：所有 local Scheduled Task 数据写本地 SQLite/本地运行资产；CRUD 与执行不需要账号；没有上传、跨设备同步或 Cloud fallback。
- 禁止结果：把本地 task 放入账号数据 API、因未登录拒绝运行，或将 Prompt/路径写入 Cloud telemetry payload。
- 决策依据：根决策 D1、D5、D8 与不变量 11。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| definition 通过 local SQLite repository 往返 | `ora_db::repository::scheduled_task::tests::definitions_round_trip_update_and_soft_delete` | `Partial` |
| 未登录、离线环境可完成 local CRUD 和执行 | 尚未实现 | `Missing` |
| contracts/backend 不调用 Cloud 同步或上传接口 | 尚无跨边界直接证据 | `Missing` |

