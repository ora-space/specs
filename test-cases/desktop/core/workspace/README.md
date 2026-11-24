# Desktop Workspace 核心测试用例

本目录是 Desktop 本机目录项目与 Workspace Git 能力的 reviewer 风险导航图，与
[决策目录](../../../../decisions/desktop/core/workspace) 镜像。系统承诺来自
[本机目录项目根决策](../../../../decisions/desktop/core/workspace/0-local-directory-project-with-probed-git-capability.md)。

## 阅读与维护规则

- 用例按探测分类、操作守卫和入口降级组织，不按 crate 或页面组织。
- 证据钉在 desktop `c9a76541`（ora-space/desktop#812 与 ora-space/desktop#818 合并后）。`Covered` 只表示已有直接证据。
- “active local 定时任务必须以 Git 仓库为目标”的义务由
  [Scheduled Task 接纳与授权](../scheduled-task/admission-and-approval.md#local-activation-must-require-a-git-project-and-explicit-consent)
  承担。本目录只登记普通目录下的表单与后端表现，链接过去，不重复登记。

| 文件 | 覆盖的风险面 |
| --- | --- |
| [probe-classification.md](probe-classification.md) | 三态分类、损坏仓库不报普通目录、嵌套目录、重新探测、时限、清理租约与语言环境 |
| [git-entry-degradation.md](git-entry-degradation.md) | 添加普通目录项目不初始化 Git、Git 审阅操作守卫、`unknown` 保持原有行为、变更面板、worktree 入口、拒绝后重探、定时任务表单 |
