---
status: superseded
---

# Target 生命周期转换必须推进 Scope 收敛纪元

> 已被取代：本决策由 [Skill 按存储路径以会话清单交给 Agent，取代 Effect 物化](../agent/session/skill/0-skill-catalog-from-storage-paths.md) 取代，原因：Effect 退役后没有 Target 与纪元，发送门禁与 `obligation` 一并删除，发送不再等待 Skill。当前系统承诺以取代者为准；取代者的实现合并之前，desktop 里仍在运行的 Effect 以本文为历史依据。Effect 的核心用例暂时保留，在取代者进入 `implemented` 时随 Effect 代码及其中的引用一并删除。

Scope generation 不仅是 Desired State 的版本，而是该 Scope 内所有不可变 projection 的收敛
纪元。任何改变“某 Target 在给定 generation 下会投影出什么内容”的事务都必须推进纪元：Desired
替换如此，Target 拓扑变化（Consumer 退役、声明替换）同样如此。否则退役 Target 的空贡献会在
同一不可变身份 `(target_id, generation, consumer_revision_id)` 上与它 active 期间持久化的
非空 projection 冲突，退役与共享同一 Resource 的接替 Consumer 都永远无法收敛
（desktop issue #6）。缺失的 Workspace 根目录是独立观察事实：没有存活贡献者仍 desire 内容时
退役完成；仍有贡献者时相关 Target 停在 manual Condition `workspace_root_missing`，由显式 retry
在目录恢复后重新收敛。没有 declared Consumer 的 Agent 不是“正在同步”。本决策已实现并通过回归
验证；Desired 替换、审计时间与 RecoveryRequired 语义不变。

## 继承与修改

继承 [Effect 系统基础](0-effect-system-foundation.md)。

| 约定 | 本决策 |
| --- | --- |
| generation 是 Scope 的单调收敛版本，CAS 替换推进一次 | 继承；语义从“Desired State 版本”扩展为“projection 输入纪元” |
| 观察、重试、Condition、运行状态不推进 generation | 继承，不变 |
| 退役 Target 以空贡献参与共享 Resource 合并，收敛后才删除 | 继承，不变 |
| Blocked 只由匹配的 change trigger 或显式用户动作恢复；retry 不修改 Desired | 继承，不变 |
| Consumer 声明变化替换 Target 时不推进 generation | 修改：同一事务推进受影响 Scope 的纪元 |
| retire_consumer 用当前 generation 唤醒退役 Target | 修改：退役事务推进纪元并以新 generation 唤醒全部 Target |
| Source retirement 删除 Desired intent | 继承，不变；republish 恢复语义见 D3 |
| Workspace+Agent 的 status 只有“有 Target”或“空” | 修改：同时返回 obligation，见 D4 |
| 缺失 Workspace 根目录与缺失 Resource 目录都是空观察 | 修改：根目录缺失是独立事实，见不变量 5 |

## D1：拓扑变化与 Desired 替换同构地推进纪元

`Generation` 的不变量是：同一 `(target_id, generation, consumer_revision_id)` 永远对应同一份
projection 内容。Target 的 lifecycle 从 active 变为 retiring 会把该 Target 的 Desired 选择清空，
这是与 Desired 替换同级别的 projection 输入变化，因此必须在同一事务内推进受影响 Scope 的
generation，并原子唤醒 Scope 内全部 active 与 retiring Target。推进记录一条
`topology_changed` 审计事件，唤醒原因为 `topology_changed`；两者都只作诊断。

不推进纪元的失败模式正是 issue #6 的插件来源切换：退役 Target 在旧 generation 上的空 projection
与已持久化的非空 projection 同身份不同内容，持久化以“不可变 projection digest 冲突”失败；接替
Consumer 的 reconcile 会把退役 Target 作为共享 Resource 的 contributor 一并投影，同样冲突，于是
新旧 Target 同时进入无限重试，需要 Effect 的 Agent 其 readiness gate 无法放行。

## D2：两个退役入口都在同一事务内推进纪元

- `retire_consumer`（插件卸载）：标记 Consumer 与全部 Target retiring 后，对每个受影响 Scope
  推进一次纪元。Skill catalog 是否同时变化不影响该义务——纯 Consumer 型插件卸载时 Desired 不变，
  纪元仍必须推进。
- 声明替换（插件更新）：`declare_consumer` 检测到 Consumer Revision 变化而替换旧 Target 时，
  对该 Scope 推进纪元；新 Target 直接在新纪元建立，旧 Target 在新纪元完成退役。

两个入口与 Desired 种子（seed_scope_sources）互不合并：全新 Scope 没有可退役的 Target，
既有 Scope 的 Desired 集合完整，两个推进集合在实践中不相交；即使相交，多推进一次纪元只是
多一个空转 epoch，不破坏任何不变量。

## D3：退役 Source 重新发布必须恢复 Desired intent

Source retirement 删除该 Source 在所有 Scope 的当前 Desired intent。若同一 Source（含同一
revision）再次发布——例如同一插件卸载后重装——repository 必须像新 Source 一样把 Desired intent
重新装入全部 active Scope 并推进各自 generation，而不是仅恢复 revision availability。否则重装
后 Desired 永远缺席，接替 Target 无法收敛。active Source 重复发布同一 revision 仍是 no-op。

## D4：发送门禁用 obligation 区分“不需要 Effect”和“Target 尚未就绪”

Workspace+Agent 的 status 读取同时返回 `obligation`：

- `required`：该 Agent identity 存在 lifecycle 为 `declared` 的 Consumer。这个 Workspace 还没有
  active Target 时义务仍然成立，发送门禁继续等待配对和 ready。新建 Workspace 因此不会在 worker
  配对之前放行。
- `not_required`：不存在 declared Consumer。从未声明 Effect、卸载完成后的旧插件 ID，以及升级后
  所选身份与已注册 Consumer 不一致，都是这条路径。发送门禁不等待一个不会出现的 Target。

`required` 且 Target 带有 blocking 且 retry 为 `manual` 的 Condition，或 phase 为
`recovery_required` 时，门禁是失败态并展示原因。显式 retry 只唤醒该 Target，不修改 Desired；
桌面端在 wakeup 落库后通知 worker。Target status 包含该 Target 自己的 Condition，以及它每个
绑定 Resource 的 Condition，所以 Resource 上的 `workspace_root_missing` 能进入门禁。观察、
retry 和 Condition 仍不推进 generation。

## 为什么不是这些替代方案

| 方案 | 否决理由 |
| --- | --- |
| 把 lifecycle 纳入 projection 身份（如新增列参与 UNIQUE） | 需要迁移不可变 projection 表；Resource contributor 边仍引用 requirement digest，改动面大且不解决“同身份不同内容”的根因 |
| 退役 Target 跳过投影持久化 | Resource projection 的 contributor 边外键引用 Target projection digest，空贡献也必须可持久化 |
| 失败时删除旧 projection 行 | 破坏不可变历史：readiness receipt、attempt 与审计链都引用历史 digest |
| 仅修 retire_consumer，不修声明替换入口 | 插件更新（同一 Consumer、声明变化）在旧 generation 上收敛过的 Target 会命中同一冲突 |
| 前端对长期 pending 超时放行 | 掩盖收敛失败，违背 Effect 必需 Agent 的 readiness 证据义务 |
| 把缺失 Workspace 根目录观察成空 Resource 目录 | 无法区分“目录尚未创建、应当 Create”和“根目录已被用户删除、不得重建”；接替者仍 desire 内容时会规划 Create，并在 prepare 上无限失败 |
| 已安装 Agent 在 status 为空时一律视为仍在同步 | 没有 declared Consumer 时 Target 不会出现，升级或插件 ID 不一致会永久挡住发送 |

## 不变量

1. 同一 `(target_id, generation, consumer_revision_id)` 永远对应同一份 projection 内容。
2. Target lifecycle 进入 retiring 的事务必须推进该 Target 所在每个 Scope 的 generation。
3. 拓扑纪元推进原子唤醒 Scope 内全部 active 与 retiring Target，并各记录一条 `topology_changed`
   审计事件。
4. 退役 Source 重新发布后，其 Desired intent 在全部 active Scope 恢复且只推进一次 generation。
5. 缺失的 Workspace 根目录不同于“Resource 目录尚不存在”。没有存活贡献者仍 desire 内容时，退役
   完成并清理其 ledger。仍有存活贡献者 desire 内容时，相关 Target 以 code `workspace_root_missing`、
   blocking、retry `manual` 的 Condition 进入 Blocked，不自动重试，也不重建该根目录。目录恢复后，
   显式 retry 触发的下一次 reconcile 重新收敛；空 projection 成功后删除退役 Target。
6. 没有 declared Consumer 的 Agent，其 Workspace Effect 义务是 `not_required`，发送门禁不把它当成
   同步中。已声明 Consumer 的义务是 `required`，即使该 Workspace 尚无 active Target。

## 落地与验收

`retire_consumer`、`declare_consumer`（声明替换）、`publish_skill_revision`（退役 Source 重发）、
缺失根目录的观察与 `workspace_root_missing` Condition、以及 Workspace+Agent 的 `obligation`
已按本决策实现。回归证据见
[Effect 收敛核心用例](../../../../test-cases/desktop/core/effect/convergence.md) 与
[Plugin Skill 发布用例](../../../../test-cases/desktop/core/effect/skill/plugin-skill-publication.md)。
