# Scheduled Task 核心测试用例

本目录是 Desktop Scheduled Task 的 reviewer 风险导航图，与
[决策目录](../../../../decisions/desktop/core/scheduled-task) 镜像。系统承诺来自
[定时任务执行归属根决策](../../../../decisions/desktop/core/scheduled-task/0-local-and-cloud-execution-ownership.md)。

## 阅读与维护规则

- 用例按调度身份、授权、执行隔离和历史事实组织，不按 crate 或页面组织。
- `Covered` 仅表示已有直接证据；定义层或 repository 层只覆盖部分义务时标为 `Partial`。
- Cloud 真实运行仍由后续 `decisions/cloud/` ADR 决定；当前只登记 Desktop 必须保持不可激活的边界。

| 文件 | 覆盖的风险面 |
| --- | --- |
| [schedules-and-occurrences.md](schedules-and-occurrences.md) | 受限周期、occurrence 幂等、迟到/离线 missed、manual 身份与时区边界 |
| [admission-and-approval.md](admission-and-approval.md) | Git 项目资格、Cloud 不可激活、无人值守同意、重授权和撤销 |
| [execution-isolation.md](execution-isolation.md) | revision 冻结、新会话、独立 worktree、并发、超时和无自动重试 |
| [history-notifications-and-residency.md](history-notifications-and-residency.md) | 状态事实、通知、软删除、100 条保留与本地数据驻留 |

