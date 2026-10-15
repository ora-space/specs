# Scheduled Task 接纳与授权

## Local Activation Must Require A Git Project And Explicit Consent

- 风险：无人值守 Agent 在用户当前目录直接运行，或在未告知权限、凭据和额度风险时自动执行工具。
- 前置状态：用户保存 enabled local task，目标可能不存在、不是 Git 项目，或没有
  `full_auto_in_worktree` 同意。
- 触发：创建、编辑后启用，或 scheduler 每次接纳。
- 必须成立：active local task 的 Project 必须存在且为 Git；有效同意随任务持久化；缺失任一资格即拒绝保存或明确失败，
  不回退原目录。
- 禁止结果：默认勾选同意、只在 UI 校验，或项目无 worktree 能力时直接在原目录执行。
- 决策依据：根决策 D4、D8 与不变量 9、10、12。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| enabled local task 没有显式 policy 时拒绝创建 | `ora_application::scheduled_task::tests::enabled_local_task_requires_explicit_unattended_approval` | `Covered` |
| 合法 Project 和 consent 创建 revision 1 的 active local task | `ora_application::scheduled_task::tests::approved_local_task_is_created_at_revision_one` | `Partial` |
| 数据库约束使 policy 与 consented-at 同时存在或同时为空 | `ora_db::migration::schema::schema_v0015::tests::creates_constrained_scheduled_task_storage_and_rolls_back` | `Covered` |
| 非 Git、缺失目录和 worktree 创建失败均不能降级原目录 | 尚无直接证据 | `Missing` |
| UI 在同意前展示 worktree 非沙箱、命令/凭据及额度风险 | 尚无直接证据 | `Missing` |

## Reconsent Must Follow Project Or Policy Changes Only

- 风险：任意编辑都反复索取授权，或真正改变执行边界时沿用旧同意。
- 前置状态：任务已有有效 `full_auto_in_worktree` 同意。
- 触发：分别修改 Prompt/schedule、agent/model、Project 或 approval policy。
- 必须成立：Prompt 和 schedule 修改增加 revision 但保留同意；Project 或 policy 变化清除旧同意，保持 enabled 时必须重新同意。
- 禁止结果：以 revision 变化本身强制重授权，或换 Project 后继续使用旧同意。
- 决策依据：根决策 D2、D8 与不变量 6、9。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| Prompt/schedule 修改 revision+1 且 consent 时间不变 | `ora_application::scheduled_task::tests::prompt_and_schedule_changes_increment_revision_without_reconsent` | `Covered` |
| Project 变化且没有 fresh consent 时不能保持 enabled | `ora_application::scheduled_task::tests::changing_project_requires_fresh_consent_before_remaining_enabled` | `Covered` |
| policy 变化必须重新展示风险并写入新 consent | 尚无直接证据 | `Missing` |

## Revoking Consent Must Disable Future Scheduling

- 风险：用户撤销无人值守权限后，任务仍在后台接纳新运行，或通过删除历史掩盖旧事实。
- 前置状态：任务已启用并有既有运行或历史。
- 触发：用户撤销 consent。
- 必须成立：任务原子清除 approval 并停用；未来 occurrence 不再接纳；既有 revision、run 和 missed 历史不改写。
- 禁止结果：只清除 UI 开关、取消并删除已完成历史，或允许 scheduler 使用缓存授权继续运行。
- 决策依据：根决策 D8 与不变量 8、9。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| revoke 清除 approval、停用任务且不改变 definition revision | `ora_application::scheduled_task::tests::revoking_approval_disables_the_task_without_changing_revision` | `Covered` |
| scheduler 观察撤销后取消未来 job 且不改写历史 | 尚未实现 | `Missing` |

## Cloud Must Stay Visible But Inactive Until Its Real Adapter Exists

- 风险：Desktop 用本地 timer、mock success 或本机凭据冒充 Cloud 执行。
- 前置状态：Cloud 独立 ADR、持久调度、IssueRun/Thread adapter 或取消写入者尚未全部就绪。
- 触发：用户查看 Cloud 说明或尝试保存 Cloud task。
- 必须成立：UI 可解释 Cloud，但不能激活或保存为 execution owner；backend 同样拒绝请求；不会产生模拟 run。
- 禁止结果：仅禁用按钮却允许直接 IPC 保存，或失败后回退 local。
- 决策依据：根决策 D3、D5 与不变量 1、5、7。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| application 拒绝 Cloud definition | `ora_application::scheduled_task::tests::cloud_definitions_are_rejected_while_cloud_is_unavailable` | `Covered` |
| Desktop 显示 Cloud 说明但所有保存/激活入口不可用 | 尚无直接证据 | `Missing` |
| Cloud 启用前具备真实 issue-bound IssueRun/Thread、资格复查和 60 分钟取消写入者 | 后续 Cloud ADR 与实现 | `Missing` |

