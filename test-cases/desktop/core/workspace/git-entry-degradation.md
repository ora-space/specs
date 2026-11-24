# 依赖 Git 的入口降级

## Adding A Plain Folder Project Must Not Initialize Git

- 风险：添加项目时要求目录是 Git 仓库，或替用户执行 `git init`，在用户目录留下未要求的 `.git`。
- 前置状态：一个存在的普通目录，不在任何仓库内。
- 触发：以该目录创建项目，然后列出 Workspace。
- 必须成立：项目创建成功；列出时主 Workspace 报 `present` + `notRepository`，即目录里没有被创建 `.git`。
- 禁止结果：创建被拒；创建后目录变成仓库。
- 决策依据：根决策 D1，不变量 1。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| 普通目录可以添加为项目，列出为 `notRepository` | `ora_backend::workspace::git_capability_tests::plain_folder_reports_not_git_and_refuses_git_operations_until_initialized`、`ora_backend::project::tests::upgrade_lets_the_folder_decide_a_project_created_before_its_folder` | `Covered` |
| 添加后目录里没有 `.git` | 缺口：没有直接断言 `.git` 不存在，由列出结果为 `NotRepository` 推出 | `Partial` |

## Git Review Operations Refuse A Plain Folder Before The Baseline Check

- 风险：普通目录上的 diff、提交、推送在 Git 深处失败，客户端只能看到内部错误；或者先报出与根因无关的基准缺失。
- 前置状态：普通目录项目的主 Workspace。
- 触发：`getWorkspaceDiff`（`Unstaged` 和 `Branch`）、`getWorkspaceStatus`、`stageWorkspaceChanges`、`unstageWorkspaceChanges`、`commitWorkspaceChanges`、`pushWorkspaceBranch`。
- 必须成立：全部返回 `workspace_not_git_repository`，不运行操作自身的 Git 命令；`Branch` 范围不先报 `workspace_diff_baseline_unavailable`。
- 禁止结果：返回 `internal_error`；在 `unknown` 时也拒绝。
- 决策依据：根决策 D5，不变量 7。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| 六个操作（共 7 次调用）都返回同一个带类型的拒绝，`Branch` 先于基准检查 | `ora_backend::workspace::git_capability_tests::plain_folder_reports_not_git_and_refuses_git_operations_until_initialized` | `Covered` |
| `unknown` 时不按普通目录拒绝 | `ora_backend::workspace::git_capability_tests::damaged_repository_reports_unknown_rather_than_plain` | `Covered` |
| 错误码与 serde 标签一致 | `ora_contracts::error::tests::public_error_codes_match_serde_tags_for_every_variant` | `Covered` |

## Unknown And Unanswered Capability Keep Previous Client Behavior

- 风险：结论不确定时把可用的 Git 功能藏起来，用户无从自救。
- 前置状态：列表报 `unknown`、`missing`、`remote`，目标不在列表中，或列表请求失败。
- 触发：打开变更面板、新建菜单、输入框的分支菜单。
- 必须成立：变更面板、新建 worktree 入口与分支菜单照常可用，Git 读取照常发出；列表加载中只推迟 Git 读取，新建 worktree 入口保持可见。
- 禁止结果：任何非 `notRepository` 的状态触发降级。
- 决策依据：根决策 D4，不变量 6。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| 列表失败、目标不在列表中时，变更面板照常加载 diff | `packages/app-shell/src/features/diff/task-diff-view.not-git.test.tsx` 的 `loads the diff as before when the workspace list fails`、`loads the diff as before when the workspace is absent from the list` | `Covered` |
| 探测报 `repository` 或 `unknown` 时，变更面板照常加载 diff 与 status | `packages/app-shell/src/features/diff/task-diff-view.not-git.test.tsx` 的 `it.each(["repository", "unknown"])` `loads the diff as before when the probe reports %s` | `Covered` |
| Git 项目上新建 worktree 入口保留 | `packages/app-shell/src/features/workspace/sidebar-create-menu.test.tsx` 的 `keeps the worktree task for a Git project`、`packages/app-shell/src/features/chat/composer-context-bar.test.tsx` 的 `keeps creating a worktree for a Git project` | `Partial` |
| `unknown`、`missing`、`remote` 时其他入口行为不变，列表加载中新建 worktree 入口保持可见 | 缺口：`missing`、`remote` 没有前端用例；侧栏、输入框、worktree 对话框与定时任务表单在 `unknown` 下没有前端用例；侧栏与输入框在列表加载中保持入口可见没有直接断言 | `Partial` |

## The Changes Panel Explains A Plain Folder And Issues No Git Reads

- 风险：普通目录上的变更面板显示通用的加载错误，或者仍在发出必然失败的 diff 和 status 查询；会话里的 diff 链接停在空 patch 上。
- 前置状态：Workspace 报 `notRepository`；或列表仍在加载。
- 触发：打开变更面板，点击会话中指向 diff 的链接。
- 必须成立：显示说明和“重新检查”，保留工具栏，不发 diff 和 status 查询；diff 链接改开文件面板；列表还在加载时，链接留在变更面板等待结果。
- 禁止结果：发出 Git 读取；加载中就提前跳到文件面板。
- 决策依据：根决策 D4、D6，不变量 9。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| 显示说明、保留工具栏，不发查询 | `packages/app-shell/src/features/diff/task-diff-view.not-git.test.tsx` 的 `explains the folder instead of loading a diff, and keeps the panel toolbar` | `Covered` |
| “重新检查”会重新探测 | 同文件 `re-probes the folder from the explanation` | `Covered` |
| diff 链接改开文件面板；加载中等待 | 同文件 `hands a chat jump to the Files panel because there is no diff to show`、`keeps a chat jump in Changes while the folder is still being checked` | `Covered` |

## A Not-Git Refusal Re-Probes And Switches To The Explanation

- 风险：面板打开期间 `.git` 被删，界面一直提供必然失败的“重试”。
- 前置状态：缓存的列表仍报可用，后端已拒为 `workspace_not_git_repository`。
- 触发：任一 diff、status 或写操作被拒。
- 必须成立：立即重新获取列表，面板切到说明页。
- 禁止结果：只显示通用错误和重试。
- 决策依据：根决策 D7。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| 被拒后重新探测并切到说明页 | `packages/app-shell/src/features/diff/task-diff-view.not-git.test.tsx` 的 `re-probes the folder when Git refuses it, then explains instead of a retry` | `Covered` |

## Worktree Task Entries Are Not Offered For A Plain Folder

- 风险：普通目录上仍出现新建 worktree 的入口，用了才失败；对话框对普通目录发 Git 分支查询。
- 前置状态：项目主 Workspace 报 `notRepository`。
- 触发：打开侧栏新建菜单、输入框分支菜单，以及被旧状态打开的 worktree 对话框；后端创建 worktree。
- 必须成立：两个入口隐藏；对话框只显示说明，不能提交，也不查询分支；后端用既有的 `worktree_requires_git_repository` 拒绝。
- 禁止结果：入口可见；对话框列出分支。
- 决策依据：根决策 D6，不变量 9。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| 侧栏和输入框的入口隐藏 | `packages/app-shell/src/features/workspace/sidebar-create-menu.test.tsx` 的 `leaves out the worktree task for a project folder that is not Git`、`packages/app-shell/src/features/chat/composer-context-bar.test.tsx` 的 `leaves out creating a worktree once the project folder is not Git` | `Covered` |
| 对话框只给说明，不能提交，不查分支 | `packages/app-shell/src/features/workspace/workspace-dialogs.test.tsx` 的 `explains the folder is not Git and offers no submit or branch lookup` | `Covered` |
| 后端拒绝映射为 `worktree_requires_git_repository`，前端显示原因 | `ora_application::error::tests::worktree_provisioner_errors_keep_their_dedicated_application_variants`、`ora_backend::error::tests::maps_semantics_without_inspecting_the_source_chain`、`workspace-dialogs.test.tsx` 的 `explains that worktree mode requires a Git repository` | `Covered` |

## The Scheduled-Task Form Blocks Only Saves The Backend Must Refuse On A Plain Folder

- 风险：表单放行必然失败的保存，用户只看到通用错误；表单拦下后端本可接受的保存（例如已冻结分支的停用任务改名）；
  或者普通目录上已启用的任务没有任何停止路径。
- 前置状态：项目主 Workspace 报 `notRepository`；任务分别处于新建、换项目、未冻结基准分支、已冻结且停用、
  保存为启用状态几种情形。
- 触发：表单保存、列表启用开关、撤销同意。
- 必须成立：后端以 `invalid_request` 拒绝新建、换到普通目录项目、未冻结任务的任何保存、保存为启用状态（含已启用
  任务保持启用的编辑）；已冻结、项目不变且保持停用的任务可以保存；表单只在 `notRepository` 时、只拦截这几类保存；
  列表启用开关改为打开表单；撤销同意总能停用任务。普通目录上从列表开关停用已启用、未冻结的任务目前会被拒，
  停止路径是撤销同意或删除（后续见 ora-space/circus#1472）。
- 禁止结果：后端报 `internal_error`；表单在 `unknown` 时拦截；已冻结的停用任务被拦下。
- 决策依据：根决策 D6 与不变量 10；active 资格本身由
  [Local Activation Must Require A Git Project And Explicit Consent](../scheduled-task/admission-and-approval.md#local-activation-must-require-a-git-project-and-explicit-consent)
  承担，本用例不重复登记。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| 后端以 `invalid_request` 拒绝普通目录项目上的新建（停用草稿也拒），不报内部错误 | `ora_backend::scheduled_task::tests::rejects_a_disabled_draft_for_a_plain_folder_project_as_invalid`、`ora_application::scheduled_task::target_validator::tests::reports_a_plain_folder_default_branch_as_not_a_repository` | `Covered` |
| 已冻结的停用任务可以保存，保存为启用状态被拒 | `ora_backend::scheduled_task::tests::edits_but_refuses_enabling_a_task_whose_project_folder_is_no_longer_git` | `Covered` |
| 未冻结任务的保存被拒，停用状态下也一样 | `ora_application::scheduled_task::tests::saving_a_disabled_task_without_a_frozen_branch_requires_a_git_project` | `Covered` |
| 表单拦截新建、换项目、未冻结、启用，放行已冻结的停用编辑 | `packages/app-shell/src/features/scheduled-task/scheduled-task-form.test.tsx` 的 `blocks creating a task for a project folder that is not Git`、`blocks moving a task to a project folder that is not Git`、`blocks saving an older task with no frozen base branch on a plain folder`、`lets a disabled task on a plain folder be edited, but not enabled`、`keeps creating and enabling available for a Git project` | `Covered` |
| 列表启用开关对普通目录项目改为打开表单 | `packages/app-shell/src/features/scheduled-task/scheduled-task-manager.test.tsx` 的 `the card enable toggle opens the form instead of enabling a task whose folder is not Git` | `Covered` |
| 撤销同意停用任务、清除同意，且不改定义 revision（撤销不读取项目目录） | `ora_application::scheduled_task::tests::revoking_approval_disables_the_task_without_changing_revision` | `Covered` |
| 普通目录上从列表开关停用已启用、未冻结的任务会被拒 | 尚无直接断言；后续见 ora-space/circus#1472 | `Missing` |
