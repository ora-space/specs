# Workspace Git 能力探测

## A Plain Folder Is NotRepository Only On Git's Explicit Answer With No Git Entry

- 风险：把普通目录报成仓库（Git 入口照常出现，用了就失败），或者把仓库报成普通目录（可用的功能被藏起来）。
- 前置状态：项目目录存在，且不在任何仓库内。
- 触发：`listWorkspaces`。
- 必须成立：Git 明确报出 `not a git repository (or any`（含 "parent directories" 和 "mount point" 两种措辞），且规范化目录及其所有祖先都没有 `.git` 条目时，返回 `present` + `notRepository`；输出 `true` 时返回 `repository`。
- 禁止结果：只凭退出码判定；只看消息、不检查 `.git`。
- 决策依据：根决策 D2、D3，不变量 3。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| 只有 Git 的明确回答才映射为探测结果，两种消息措辞都能识别 | `gitlancer::git::work_tree_probe::tests::classifies_explicit_answers` | `Covered` |
| 后端列表把普通目录报为 `present` + `notRepository` | `ora_backend::workspace::git_capability_tests::plain_folder_reports_not_git_and_refuses_git_operations_until_initialized`、`ora_backend::project::tests::upgrade_lets_the_folder_decide_a_project_created_before_its_folder` | `Covered` |

## A Damaged Unreadable Or Untrusted Repository Must Never Be Reported As A Plain Folder

- 风险：把损坏的仓库说成“不是 Git 仓库”，诱导用户对有历史的仓库重新 `git init`。
- 前置状态：分别构造断链的 `.git` 文件、损坏的 `HEAD`、空的 `.git`、不可读的 `.git`、嵌套目录上方有损坏的 `.git`、经符号链接添加且目标上方有损坏的 `.git`、所有权不受信任、配置损坏。
- 触发：`listWorkspaces`，以及 Git 审阅操作。
- 必须成立：以上情况都报 `unknown`，审阅操作不被当作普通目录拒绝。
- 禁止结果：任何一种情况报成 `notRepository`。
- 决策依据：根决策 D3，不变量 4。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| 所有权不受信任、配置损坏、断链 `.git`、空输出都保持为错误 | `gitlancer::git::work_tree_probe::tests::keeps_other_failures_as_errors` | `Covered` |
| 断链的 `.git` 报 `unknown`，审阅操作不按普通目录拒绝 | `ora_backend::workspace::git_capability_tests::damaged_repository_reports_unknown_rather_than_plain` | `Covered` |
| Git 跳过的无效 `.git`（损坏 `HEAD`、空目录、嵌套目录上方损坏）报 `unknown` | `ora_backend::workspace::git_capability_tests::damaged_git_directories_report_unknown_rather_than_plain` | `Covered` |
| 不可读的 `.git` 报 `unknown` | `ora_backend::workspace::git_capability_tests::unreadable_git_directory_reports_unknown`（以忽略文件权限的身份运行时跳过） | `Covered` |
| 沿规范化路径检查祖先 | `ora_backend::workspace::git_capability_tests::symlinked_folder_under_a_damaged_repository_reports_unknown` | `Covered` |

## A Folder Nested In A Checkout Reports Repository Like The Operations Behave

- 风险：能力报告与操作的实际行为不一致：隐藏了确实会改动外层仓库的操作。
- 前置状态：普通子目录位于外层 checkout 内；另有普通 checkout、以 linked worktree 为根的项目、隔离 task worktree。
- 触发：`listWorkspaces` 与审阅操作。
- 必须成立：以上都报 `repository`；嵌套目录上的审阅操作作用于外层 checkout，不被拒绝。
- 禁止结果：嵌套目录报 `notRepository`；隔离 Workspace 根据存储数据跳过探测。
- 决策依据：根决策 D2、D3，不变量 2、5。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| 嵌套目录报 `repository`，审阅操作作用于外层 checkout | `ora_backend::workspace::git_capability_tests::folder_nested_in_a_checkout_reports_git_like_the_operations_behave` | `Covered` |
| 普通 checkout、linked worktree 根和隔离 task worktree 都报 `repository` | `ora_backend::workspace::git_capability_tests::checkouts_and_linked_worktrees_report_git` | `Covered` |

## Capability Is Re-Probed On Every List And Flips After Git Init

- 风险：能力被缓存或从存储推断，用户在 Ora 外 `git init` 之后入口仍处于降级状态。
- 前置状态：普通目录项目，列表报 `notRepository`。
- 触发：在目录里 `git init`，然后再次列出。
- 必须成立：下一次列出报 `repository`，status 可用；目录缺失时报 `missing`，不带 Git 能力。
- 禁止结果：重启后才生效；目录缺失时报 `notRepository`。
- 决策依据：根决策 D2、D7，不变量 2。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| `git init` 后下一次列出翻转为 `repository`，status 可用 | `ora_backend::workspace::git_capability_tests::plain_folder_reports_not_git_and_refuses_git_operations_until_initialized`（后半段） | `Covered` |
| 缺失的目录报 `missing`，不带 Git 能力 | `ora_backend::workspace::git_capability_tests::missing_folder_reports_missing_without_git_capability`、`ora_backend::workspace::tests::reports_a_deleted_project_folder_as_missing_until_it_returns` | `Covered` |

## Listing Never Waits On Cleanup Or A Stalled Git

- 风险：一次 worktree 删除或一个卡住的 Git 进程拖住整个 Workspace 列表；或者清理中的 checkout 被误报。
- 前置状态：某 Workspace 的清理租约被占用；或 Git 超过 5 秒没有结束。
- 触发：`listWorkspaces`。
- 必须成立：清理期间列表不等待，该 Workspace 报 `unknown`，租约释放后恢复真实结果；Git 超时后被 kill，探测记为 `unknown`。
- 禁止结果：列表阻塞到清理结束；超时被报为 `notRepository`。
- 决策依据：根决策 D2、D3，不变量 4、8。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| 清理期间列表不等待、报 `unknown`，释放后报 `repository` | `ora_backend::workspace::git_capability_tests::listing_skips_the_probe_of_a_checkout_being_cleaned_up`、`ora_backend::git_cleanup::keyed_locks::tests::try_shared_yields_to_writers_without_waiting` | `Covered` |
| 超过时限的 Git 被 kill 并按超时报告，子进程占着管道时也一样（全平台运行） | `gitlancer::exec::runner::tests::kills_a_command_that_outlives_its_deadline`、`gitlancer::exec::runner::tests::deadline_covers_output_held_open_by_a_child_process` | `Covered` |
| 探测超时最终报为 `unknown` | 缺口：没有经过 `probe_workspace_git` 或列表的端到端测试，只有 runner 层的超时测试；超时映射为 `unknown` 目前只能从“任何错误都是 `unknown`”的分支推出 | `Partial` |

## Probe Classification Is Independent Of The User's Locale

- 风险：用户的非英文 locale 把 Git 消息翻译掉，普通目录被误报为 `unknown`；或者全局改 locale 影响了用户钩子。
- 前置状态：调用方环境设置了非英文的 `LC_ALL`。
- 触发：探测命令与普通 Git 命令各运行一次。
- 必须成立：探测命令收到 `LC_ALL=C`；普通命令不额外设置 `LC_ALL`。
- 禁止结果：所有 Git 命令都被强制设置 `LC_ALL=C`。
- 决策依据：根决策 D3，不变量 11。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| 只有显式要求时才设置 `LC_ALL` | `gitlancer::exec::runner::tests::pins_the_message_locale_only_when_asked`（仅 Unix） | `Partial` |
| 非英文 locale 下普通目录仍报 `notRepository` | 缺口：没有在本地化 Git 环境下跑探测的测试；Windows 也没有覆盖环境传递 | `Partial` |
