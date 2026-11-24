# 工作目录回收：只回收能恢复或用户明确丢弃的已删对话目录、先登记后删除、精确归属、可重试、不动交付

依据 [删除对话时回收工作目录](../../../decisions/cloud/project-git/20261119-reclaim-deleted-conversation-workspace.md)（`implemented`）。
下文的 D 编号与不变量编号都指这篇决策；交付本身的承诺见 [回合交付](turn-delivery.md)。

证据读取的是 cloud `fe0b378b`（PR #435 的 merge，审过的 head `c50c648c`；工作目录回收本身的基线仍是 PR #419 的 `d4ec60d5`）的 `v2-server/`，路径相对该目录；前端证据以 `frontend-v2/` 开头。需要 PostgreSQL
的测试在设置 `TEST_DATABASE_URL` 时运行；基线 `d4ec60d5` 的相关包在 Git 2.34.1 与 2.53 上都通过（`fe0b378b` 新增用例在 Git 2.53 上通过）。29 项汇总为 12 Covered / 16 Partial / 1 Missing；Partial 与 Missing 的缺口写在各行里。

## Only A Recoverable Workspace Is Reclaimed By Default

- 风险：默认的删除把运维也恢复不了的改动一起删掉：最后一轮交付之后又写入的文件、stash、没检出分支上的提交、子模块里
  没提交的改动；或者把还在交付、旧会话的目录当成能恢复。
- 前置状态：同一项目里若干对话，分别处于：最后一轮 `delivered` 且之后没有变化；最后一轮 `empty`；最后一轮交付失败；
  最近一条记录落在更早的回合上；交付 ADR 落地前的旧会话；第一轮还在跑；最后一轮交付之后又写了文件、只改了被忽略文件、
  只改了物化技能；有 stash、有不在起点与 `HEAD` 祖先里的本地 ref、有指向非提交对象的 ref、有改动过的 gitlink、有已
  初始化且有未提交改动的子模块、有 gitfile 形式的嵌套 `.git`；顶层 `.git` 是解析不出来的 gitfile。
- 触发：用户以默认模式（`reclaim`，或不带参数）删除这些对话。
- 必须成立：只有五条（D1）全部成立的目录被回收；其余目录保留，回收记录以 `kept` 结束并写明 D1 的原因（`delivering`
  除外，它保持 `pending`）；判定顺序先 `delivering` 再 `legacy`；只改了被忽略文件或物化技能的不判 `changed_after_delivery`；
  `.git` 解析不出来判 `unverifiable`；判定时磁盘、镜像、Git 临时失败记 `check_failed` 保持 `pending`，不判 `kept`。
- 禁止结果：上面任何一种不能恢复的目录被删除；临时失败被记成终态。
- 验证义务与证据：

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 能恢复的两种（`delivered` 无变化、`empty`）被回收 | `internal/workspacereclaim/reclaim_test.go` `TestReclaimRemovesOnlyRecoverableCheckoutsUnlessDiscarded`：两轮对话（交付失败后再交付成功），回收后目录不存在，记录为 `reclaimed`，带交付 ID、交付提交、起点提交，释放量大于 0。`empty` 只有 store 层 `internal/store/workspace_reclaims_test.go` `TestChatCheckoutReadsTheSessionsDeliveries/empty_after_delivered` 证明读到的最后一轮状态是 `empty`；没有"最后一轮 `empty` 被回收"的端到端用例（夹具对真实仓库只做 delivered 交付）。 | Partial |
| 交付相关的不能恢复原因各一例都保留 | `TestReclaimWaitsForDeliveryWorkThenRemovesAtStartup`：最后一轮还在交付，结果为 `pending`/`delivering`，attempts 为 0，目录在，没有调用删除。`TestReclaimRemovesOnlyRecoverableCheckoutsUnlessDiscarded`：`legacy`、`last_turn_undelivered` 以 `kept` 结束，目录在。`TestReclaimKeepsCheckoutsWithWorkNoDeliveryHolds/written_after_the_delivery`：`changed_after_delivery` 以 `kept` 结束，目录在。缺口：①"最近一条记录落在更早回合上"只有 store 层 `TestChatCheckoutReadsTheSessionsDeliveries/last_turn_unenrolled` 断言最后一轮状态为空，没有回收端的用例；②"第一轮还在跑、尚无记录"时先判 `delivering` 再判 `legacy` 的顺序，没有直接用例（现有 delivering 用例前面已有一轮交付）。 | Partial |
| Git 状态的各种情况都保留 | 回收端，`kept`/`git_state`、目录在：`TestReclaimKeepsCheckoutsWithWorkNoDeliveryHolds` 的 `stash`、`branch_beyond_HEAD`、`nested_repository`（gitfile 形式的嵌套 `.git`）三个子用例；`TestReclaimKeepsSubmodulesAndUnverifiableCheckouts`：已初始化且有改动的子模块判 `git_state`，`.git` 是指向不存在目录的 gitfile 时判 `unverifiable`。检查层 `internal/workspacefs/reclaim_check_test.go` `TestInspectReclaimFindsWorkOnlyTheRepositoryHolds`：`two_stashes`、`branch_beyond_base_and_HEAD`（提交数 2）、`ref_to_a_blob`（`NotCommit`）、`gitlink`、`gitfile_in_an_ignored_directory`、`initialized_submodule_with_changes`，并验证提交在 HEAD 上、伪 ref 不算；`TestInspectReclaimCannotVerifyAnUnresolvableGit` 断言返回 `ErrUnverifiable`。HEAD 与 index：回收端 `TestReclaimKeepsWorkOnlyHEADOrTheIndexHolds` 的 `staged_only`（改动只在暂存区）和 `committed_only`（改动只在 HEAD）以 `kept`/`git_state` 结束，index 和那份 blob、提交都还在；`agent_commit_matching_the_delivery`（Agent 把交付内容原样提交）仍被回收。检查层 `TestInspectReclaimFindsWorkOnlyTheRepositoryHolds` 的 `staged_only`（`IndexDiffers`）、`committed_only`（`HeadDiffers`，`HeadCommits` 为 1）、`intent_to_add_and_unstaged_edits_are_fine`（不报）。（这一表现在先交付、再与交付比对，和回收时的做法一致。）HEAD 解析失败时 fail-closed：`TestReclaimFailsClosedOnAnUnreadableHEAD/detached_HEAD_with_its_commit_missing`（detached HEAD 的提交对象缺失，以 `kept`/`unverifiable` 结束，tree 和 blob 都在）、`/unborn_branch`（仍被回收）；检查层 `TestInspectReclaimFailsClosedOnAnUnreadableHEAD`（detached、已有分支缺对象判 `ErrUnverifiable`，unborn 不报错）。Git 读不出的 ref：`TestReclaimKeepsRefsGitCannotRead/truncated_branch_ref`（分支 ref 被截断，以 `kept`/`unverifiable` 结束，提交和 blob 仍可读）、`/HEAD_naming_a_dangling_alias`（`kept`/`unverifiable`）；检查层 `TestInspectReclaimDoesNotTrustRefsGitSkips`。HEAD 所在分支损坏：`TestReclaimKeepsAHEADWhoseBranchIsBroken` 的四个子用例（loose 或 packed-refs 里的 ref 被截短、或指向 blob），都以 `kept`/`unverifiable` 结束，提交和 blob 对象文件仍在。HEAD 提交对象损坏：`TestReclaimKeepsAHEADWhoseCommitIsCorrupt` 的 `cut_to_half`（截断一半）、`emptied`（零字节）都以 `kept`/`unverifiable` 结束。过期的 ref 锁不算 ref：`TestReclaimIgnoresStaleRefLocks`（空的和写着 OID 的 `refs/heads/main.lock` 各一例，干净对话仍被回收）；ref 名规则 `internal/workspacefs/reclaim_internal_test.go` `TestValidRefName`；`listed` 对照 `TestCheckRecordedRefsRequiresListedRefs`。`packed-refs` 结构损坏：`TestReclaimKeepsADamagedPackedRefs` 的两个子用例（最后一行没有换行、追加截断的 peeled 行）都以 `kept`/`unverifiable` 结束。嵌套 `.git` 内含指向共享仓库的符号链接：`TestDiscardIsRefusedWhenNestedGitUsesSymlinksOrAlternates/symlinked_shared_git_state` 的默认 `reclaim` 部分以 `kept`/`git_state` 结束，`wt` 与共享的 `store.git` 都还在。非提交 ref 和改动的 gitlink 在检查层有证据，判定层的映射（findings 非空即 `git_state`）由前面几例覆盖。 | Covered |
| 交付之后只改被忽略文件或物化技能不算变化 | 检查层 `TestInspectReclaimIgnoresIgnoredFilesAndMaterializedSkills`：交付后写入被忽略的 `later.log` 和物化技能，仍判 `Unchanged`，差异为 0。预览层 `TestPreviewTellsWhatDeletingDoes`：写入被忽略文件后仍判 `recoverable`，`Ignored` 为 true。缺口：没有"这两种情况下回收并删除"的回收端用例，物化技能也只在检查层出现。 | Partial |
| 临时失败可重试 | 分类单元测试 `internal/workspacefs/reclaim_internal_test.go` `TestCorruptClassifiesOnlyObjectDamage`（上下文结束、I/O 诊断在前的都可重试；只有对象与 packed-refs 损坏判不可验证）。`TestReclaimRetriesWhenTheCheckCannotRun`：镜像改名后回收得到 `pending`/`check_failed`，记录保持 `pending`，attempts 为 1，`last_error` 非空，目录在；镜像恢复后由新的 Reclaimer `Resume` 回收为 `reclaimed`。`TestReclaimRetriesAPassingHEADReadError`：用 `workspacefs.InjectGitFault` 只让 `ls-tree … HEAD` 返回一次 I/O 错误，结果 `pending`/`check_failed`、attempts 为 1，`Resume` 后回收；检查层 `TestInspectReclaimRetriesAPassingHEADReadError`（返回可重试错误，不是 `ErrUnverifiable`；故障去掉后检查成功）。`TestReclaimRetriesPassingGitFailures`：HEAD 解析时注入 I/O 错误（分支、detached 各一例），以及带双重诊断的权限错误（`for-each-ref`、`ls-tree HEAD`，权限错误后面跟着 Git 的通用对象错误），都得到 `pending`/`check_failed`，`Resume` 后回收。HEAD 解析超时：`TestReclaimLeavesAHEADThatTimesOutPending`（返回 deadline 错误，记录保持 `pending`，之后 `Resume` 回收）；检查层 `TestInspectReclaimDoesNotJudgeAHEADThatTimesOut`（不返回 `ErrUnverifiable`）。HEAD 对象所在目录暂时不可访问：`TestReclaimRetriesWhenHEADsObjectIsUnreadable`（`pending`/`check_failed`，恢复权限后 `Resume` 回收）。 | Covered |

- 决策依据：D1、D3，不变量 2、3。

## Deleting Tells The User What Happens And Discard Requires The Previewed State

- 风险：用户在确认框里看到的和实际删除的不一致：预览之后改动又变了，`discard` 却仍然执行；`git_state` 时显示“没有
  未交付的改动”，用户以为丢弃什么都不会丢；预览读不到时界面承诺保留，实际却回收了。
- 前置状态：能恢复、未交付（含 `git_state`）、没有工作目录、目录已不存在的对话各一个；Cloud 有或没有 Agent 运行时。
- 触发：读预览；用预览给出的 `revision` 或过期的 `revision` 以 `discard` 删除；预览之后把一个文件同路径、同增删行数地
  改写；预览请求失败后在界面上删除。
- 必须成立：预览给出 `none`（`no_workspace` 或 `workspace_missing`）、`recoverable`（有被忽略文件时标出）、`undelivered`
  及原因、未交付改动列表（只有内容检查成功时才有，无法检查时为空且不代表没有改动）、`git_state` 时的 stash 数量、ref 与提交数、嵌套仓库路径、最近一次 `delivered` 交付；
  `discard` 带回的 `revision` 与删除时（结束会话之前）重算的不一致、或会话有回合未结束时返回 `409 deletion_preview_stale`，
  对话、目录、回收记录都不动；`revision` 一致而判为 `undelivered` 且不可丢弃（`unverifiable`、摘要不完整）时返回
  `409 discard_unverifiable`，对话与目录都不动；内容改写也使 `revision` 改变；没有运行时时，有工作目录的对话的预览和 `discard` 在任何
  改动之前返回 `503 workspace_reclaim_unavailable`，默认删除照常并登记 `pending`/`reclaim_unavailable`；删除返回
  `reclaimed`（字节与条目数）、`kept`（原因）、`pending`（原因）或 `none`；界面在预览读不到时禁用删除、提供重试，
  能恢复时提供 patch 下载，`delivering` 时不提供丢弃。
- 禁止结果：过期的 `discard` 删掉了用户没看到的内容；没有运行时时登记了未经验证的 `discard`；确认框承诺了与结果不符的事。
- 验证义务与证据：

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 预览各种结果与字段 | `TestPreviewTellsWhatDeletingDoes`：`recoverable`（`LatestDelivery`、revision 非空），写入被忽略文件后 `Ignored` 为 true，交付后写入 `late.txt` 后判 `changed_after_delivery` 并列出该文件、Total 为 1、revision 改变，同一状态 revision 稳定，以及 `none`/`workspace_missing` 和 `none`/`no_workspace`。`TestGitStatePreviewNamesWhatTheRepositoryHolds`：`git_state` 预览的 stash 数为 1、嵌套仓库为 `data`，再加一条 stash 后数为 2 且 revision 改变。HTTP 层：`internal/server/chat_test.go` `TestChatConversationLifecycle` 按 `chat.openapi.json` 校验 `none`/`no_workspace`、未知对话 404。缺口：①预览里 ref 及其提交数只在检查层（见用例 1）验证，没有预览层用例；②`legacy`、`last_turn_undelivered` 的预览没有专门用例；③带 `git` 字段的预览没有经过 HTTP 契约校验。 | Partial |
| 过期或内容改写后的 `discard` 返回 409 且什么都不动 | 都经 `chat.Service`、用真实 Git 和 PostgreSQL：`TestDiscardOfRewrittenContentIsStale`（同路径、同增删行数改写后，旧 revision 返回 `ErrDeletionPreviewStale`，对话在、内容不变、没有回收记录；用新 revision 得到 `reclaimed`）；`TestDiscardIsStaleWhenTheStashIsReplaced`、`TestDiscardIsStaleWhenARefMovesWithTheSameCount`、`TestDiscardIsStaleWhenANestedFileIsRewrittenInPlace`、`TestDiscardIsStaleWhenARepositoryBehindAPointerChanges/{worktree_gitfile,link}`、`TestDiscardIsStaleWhenAStagedBlobIsReplaced/{nested_repository,checkout_repository}`、`TestDiscardIsStaleWhenASharedRepositoryNamedByCommondirChanges`（每例先断言展示内容前后相同，再断言返回 409、对话和 checkout 保留）；`TestDiscardIsRefusedWhenNestedGitUsesSymlinksOrAlternates/symlink_swapped_in_after_preview`（预览之后把嵌套 `.git/refs` 换成指向共享仓库的符号链接：旧 revision 返回 `ErrDeletionPreviewStale`，换新 revision 后判不可丢弃、返回 `ErrDiscardUnverifiable`，对话与目录都在）。`internal/chat/chat_test.go` `TestDeleteRecordsAndExecutesTheCheckoutReclaim`：过期 revision、缺 revision、还在交付时都被拒，对话在、没有执行回收。409 映射到 HTTP 的 `deletion_preview_stale` 只是一行 `errors.Is`，没有单独的 HTTP 用例（真实栈脚本覆盖了）。 | Covered |
| 不可丢弃时拒绝 `discard` | `internal/workspacereclaim/reclaim_test.go` 的 `TestDiscardIsRefusedWhenANestedRepositoryIsTooLarge`、`TestDiscardIsRefusedWhenNestedRepositoriesExhaustTheBudget`：当前 `revision` 有效、判为 `undelivered` 且 `discardable` 为假时，`chat.Service` 的删除返回 `ErrDiscardUnverifiable`，对话与目录都保留。嵌套 Git 目录内部的摘要边界：检查层 `internal/workspacefs/reclaim_check_test.go` `TestInspectReclaimMarksSymlinksAndAlternatesInNestedGitIncomplete` 的 8 个负面子用例（`symlinked_refs_and_objects_inside_nested_.git`、`symlinked_index_inside_nested_.git`、`symlink_inside_gitfile_target`、`symlink_inside_commondir_target`、`symlink_inside_.git/modules_entry`、`symlinked_.git/modules_entry`、`commondir_inside_.git/modules_entry`、`alternates_inside_nested_.git`）均判 `GitIdentityComplete` 为假，正向对照 `working_tree_symlink_stays_complete` 仍为真；回收端 `TestDiscardIsRefusedWhenNestedGitUsesSymlinksOrAlternates` 的 `symlinked_shared_git_state`、`symlinked_nested_index`、`nested_alternates` 均断言预览 `discardable` 为假、`discard` 返回 `ErrDiscardUnverifiable` 且对话与工作树保留，其中 `symlinked_shared_git_state` 还断言链接指向的共享 Git 状态仍可读（`git -C wt show only-here:new.txt`）；正向对照 `TestDiscardStillAllowsWorkingTreeSymlinksInNestedRepository` 断言嵌套仓库工作树里的符号链接不影响摘要完整，仍被回收。缺口：模块 `commondir` 只有检查层的摘要不完整断言，Git 目录解析到 checkout 根缺少直接用例，HTTP 层映射为 `409 discard_unverifiable` 没有直接断言。 | Partial |
| 没有运行时时不登记未经验证的丢弃 | `internal/chat/chat_test.go` `TestWithoutReclaimerAnExistingCheckoutIsNeverDiscarded`：预览和 discard 返回 `ErrReclaimUnavailable`，对话在、没有回收记录；默认删除得到 `pending`/`reclaim_unavailable` 并登记 `reclaim` 模式的 `pending`；没有 checkout 的对话预览为 `none`。`internal/server/chat_test.go` `TestChatDeletionWithoutReclaimerAnswersUnavailable`：HTTP 503 `workspace_reclaim_unavailable`，默认删除返回 200 `pending`/`reclaim_unavailable`。 | Covered |
| 界面不承诺与结果不符的事 | `frontend-v2/src/features/chat/page.test.tsx`：`does not let a deletion through while the preview cannot be read, and retries it`（删除按钮禁用，重试后可删）；`deletes a recoverable conversation with its working directory, offering its patch first`（patch 下载请求，被忽略文件提示，没有丢弃选项）；`does not offer discarding while the last turn still delivers`；`names what only the repository holds when it blocks reclaiming`；`does not offer discarding a nested repository too large to confirm`；`shows the new preview when the discard turned out stale`；`deletes without discarding when the server cannot judge the working directory`。 | Covered |

- 决策依据：D2，不变量 3。

## Every Reclaim Is Registered In The Deleting Transaction And Nothing Unregistered Is Removed

- 风险：先删对话记录、事后再扫目录找没有主人的；对话首轮失败的回滚被当成用户删除去回收；模式在登记后被改。
- 前置状态：一个有工作目录的对话；一个首轮被拒、正在回滚的新对话；一个没有工作目录的对话。
- 触发：用户删除第一个和第三个对话；第二个对话的创建失败回滚；登记后尝试修改回收的模式。
- 必须成立：删除对话记录的同一个事务里写入回收登记（工作目录身份、项目、会话、来源对话、路径、`clone_execution`、模式）；
  没有工作目录的不登记；回滚不登记；模式不可修改；Cloud 只删登记过的目录。
- 禁止结果：对话已删除却没有登记；没有登记的目录被删除；回滚产生登记。
- 验证义务与证据：

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 登记与删除对话在同一事务 | `internal/store/workspace_reclaims_test.go` `TestDeleteChatConversationRecordsReclaimInTheSameTransaction`：删除后对话不存在，登记的工作目录 ID、项目、会话、对话、路径、模式、状态都对，判定能只凭会话进行；非法模式时什么都不做。缺口：没有"事务中途失败时两者都不存在或都存在"的故障注入用例（原子性只在代码里用单个 `pgx.BeginFunc` 保证）。 | Partial |
| 回滚与无工作目录不登记 | `TestChatCheckoutWithoutCheckoutAndForOthers`：没有工作目录的对话经用户删除不登记；首轮被拒时 `Create` 走的回滚删除 `DeleteChatConversation` 不登记；他人删不了别人的对话，也不留登记。 | Covered |
| 模式不可变 | `TestReclaimModeIsImmutable`：经过各个状态迁移后模式不变；直接 UPDATE 改 mode 被数据库触发器拒绝。 | Covered |

- 决策依据：D3，不变量 4。

## Only The Exact Workspace Of The Deleted Conversation Is Removed

- 风险：删到活跃会话、别的用户、工作流运行、任务运行的目录；登记的路径或它的某一级父目录被换成链接，删到项目之外；
  登记与 `v2_workspaces` 不一致时仍按登记删。
- 前置状态：项目里有活跃会话的目录、别的用户对话的目录、工作流运行共享的目录、任务运行的目录；一个待回收的目录，其
  路径越界、本身或某一级父目录是符号链接，或在检查之后、打开之前被移走并换成指向原处的链接；目录里有指向项目之外的
  链接；登记的路径或 `clone_execution` 与 `v2_workspaces` 不一致，或有别的行用同一路径。
- 触发：执行这些回收。
- 必须成立：上述身份或路径检查不通过的都以 `kept` 结束并记原因，目录和项目外文件都不变；目录里指向外部的链接只删除
  链接本身；只删除登记的那个 checkout。
- 禁止结果：任何不属于登记对话的目录或项目之外的文件被删除。
- 验证义务与证据：

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 活跃、他人、共享、任务目录不删 | `TestReclaimKeepsSharedLiveUnsafeAndForeignCheckouts`：仍有活会话判 `kept`/`session_live`，别的会话用过判 `kept`/`shared`，目录都在。`TestReclaimRemovesOnlyRecoverableCheckoutsUnlessDiscarded`：没被删除的对话的目录保留。store 层他人删不了。`internal/agentruntime/remove_checkout_test.go` `TestRemoveCheckoutRefusesAdmittedSessionAndDropsCachedCheckout`：进程内有会话占用时拒绝。缺口：任务运行、工作流运行的目录没有专门用例（它们不经用户删除对话登记，是结构上保证的；共享目录的一例由 `shared` 覆盖）。 | Partial |
| 路径越界与符号链接不跟随 | 拒绝不安全路径：`internal/workspacefs/remove_test.go` 的 `TestRemoveCheckoutRefusesPathsOutsideTheRoot`、`TestRemoveCheckoutRefusesSymlinks`（checkout 本身、父目录、checkout 根）、`TestRemoveCheckoutRefusesADirectorySwappedAfterTheCheck`、`TestRemoveCheckoutRefusesAParentSwappedForALinkOutOfTheRoot`（父目录被移走并换成指向原处的链接，两个注入时机）都返回 `ErrUnsafeCheckout`，项目外文件不变；回收端 `TestReclaimKeepsSharedLiveUnsafeAndForeignCheckouts` 越界路径判 `kept`/`unsafe_path`；`internal/agentruntime/remove_checkout_test.go` 的 `TestRemoveCheckoutRefusesPathOfAnotherProject`。目录内外链只删链接：`TestRemoveCheckoutMeasuresAndRemovesOnlyTheCheckout` 断言删除成功，目录内指向项目外的链接被删，外部目标、兄弟目录都在。 | Covered |
| 登记与 `v2_workspaces` 核对 | `internal/store/workspace_reclaims_test.go` `TestCheckReclaimIdentitySeesLiveSharedAndChangedCheckouts`：同一路径另有行、`clone_execution` 不一致，都使 `Recorded` 为 false。回收端 `TestReclaimKeepsSharedLiveUnsafeAndForeignCheckouts`：`clone_execution` 被改后判 `kept`/`identity_mismatch`，目录在。 缺口：没有“只有登记路径不同、`WorkspaceID` 与 `clone_execution` 一致”时保留的独立断言。 | Partial |

- 决策依据：D4，不变量 1、5。

## Reclaim Does Not Block The Project And Runs Once Per Workspace

- 风险：删除大目录期间整个项目的会话都启动不了；同一个目录被两次回收同时删除；回收与交付、技能投影互相破坏。
- 前置状态：一个大目录待回收，删除被拖慢；同项目有新会话要启动、技能要投影；同一条回收被删除请求和启动同时触发。
- 触发：执行回收的同时启动同项目会话、投影技能、对同一条回收再触发一次。
- 必须成立：回收期间同项目的会话准入、`attachSkills`、`attachUserSkills`、`ApplySkill` 不被阻塞；投影结果不含被回收的
  checkout，也不因它消失而报错（`ENOENT` 跳过）；项目 runtime 不在时，回收持项目锁期间新 runtime 的创建等待；同一条回收
  只删除一次；回收与同目录的交付串行，交付记录不变。
- 禁止结果：会话准入在回收期间卡住；投影失败；同一目录被删除两次或统计重复。
- 验证义务与证据：

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 回收不阻塞同项目会话与投影 | `internal/agentruntime/remove_checkout_test.go` `TestRemoveCheckoutDoesNotHoldTheSkillLockWhileDeleting`：删除被阻塞期间同项目的 `attachSkills` 在 5 秒内完成。`TestProjectSkillsSkipsAVanishedCheckout`：已消失的 checkout 被跳过，其他 checkout 照常得到链接。`TestRemoveCheckoutWithoutRuntimeForgetsSkillIndex` 和 `TestRemoveCheckoutRefusesAdmittedSessionAndDropsCachedCheckout`：`skill-checkouts.json` 和 `p.checkouts` 里不再有被回收的 checkout。缺口：会话准入、`attachUserSkills`、`ApplySkill` 没有单独用例（它们拿的是同一把 `skillsMu`，`attachSkills` 一例证明锁在删除期间是空闲的）；"回收与同目录交付串行、交付记录不变"没有直接用例。 | Partial |
| 项目 runtime 不在时与创建互斥 | `TestProjectRuntimeCreationWaitsForTheSkillListRewrite`：持项目锁期间 `d.project()` 不插入新 runtime，释放后继续（用 50ms sleep 观察）。 | Covered |
| 同一条回收只执行一次 | `internal/workspacereclaim/reclaim_test.go` `TestConcurrentReclaimsRemoveOnce`：三路并发 `Reclaim`，删除只调用一次，三路结果都是 `reclaimed`。 缺口：没有核对并发返回的释放量与持久化记录一致。 | Partial |

- 决策依据：D4、D5，不变量 5、6。

## Interrupted Or Failed Reclaims Resume At Boot Without Re-judging

- 风险：删到一半崩溃后目录成了没有主人的残留；重启后重新判定，删了一半的目录因为“变了”落到 `kept`；删除请求超时后
  要等几周后的重启才继续；事务提交后的错误被报成“对话没删除”。
- 前置状态：登记后、删除前崩溃的回收；删到一半崩溃（`deleting`）的回收；IO 失败的回收；一个大到短时限内删不完的目录。
- 触发：重启 Cloud；删除请求在时限内删不完；事务提交后回收出错。
- 必须成立：启动补做发生在 `RecoverTurnDeliveries` 之后、恢复工作流和开始服务之前；`pending` 按 D1 判定，`deleting`
  只核对身份与路径就删完；IO 失败保持原状态、尝试次数加一、记最后一次错误；删除请求在时限内删不完返回
  `pending`/`in_progress` 并在后台删完，关机时留 `deleting`；事务提交后的错误返回 200 和回收状态，对话确已删除。
- 禁止结果：`deleting` 因为重新按 D1 判定而落到 `kept`（D4 的身份、共享、活会话、路径检查不通过时保留目录是允许的）；未完成的回收在启动时被遗漏；重试删到别的目录；对话已删除却返回失败。
- 验证义务与证据：

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 启动位置与补做 | `internal/boot/boot_test.go` `TestRecoverDeliversBeforeWorkflowResumeAndAdmission`：`workspace-reclaims` 排在 `deliveries` 之后、`workflow-resume` 之前，任何一步失败都会停下后续步骤。`TestReclaimRetriesFailedRemovalAtStartup`：登记后未执行（`pending`）和删除失败（`deleting`）的两条，都由 `Resume` 回收。`TestReclaimWaitsForDeliveryWorkThenRemovesAtStartup`：等交付的 `pending` 在 `Resume` 时回收。 | Covered |
| `deleting` 不重新判定 | `TestDeletingReclaimIsFinishedWithoutJudgingAgain`：标为 `deleting` 后删掉部分内容（目录已不再与交付一致），`Resume` 仍删完并记 `reclaimed`。 | Covered |
| 失败可重试、计数与日志 | `TestReclaimRetriesFailedRemovalAtStartup`：删除失败后结果为 `pending`/`removal_failed`，记录保持 `deleting`，attempts 为 1，`last_error` 非空；重启后回收成功。`internal/workspacefs/remove_test.go` `TestRemoveCheckoutFailsWhenAnEntryCannotBeMeasured`：统计失败返回带路径的错误，目录不动。日志内容不作断言。 缺口：没有对失败日志的断言。 | Partial |
| 删除请求的执行不随请求取消 | `internal/chat/chat_test.go` `TestDeleteReportsASlowOrFailingReclaimAsPending`：回收慢时返回 `pending`/`in_progress`，回收失败时返回 `pending`/`removal_failed`，两种情况对话都已删除、调用不报错（替身回收器）。`internal/workspacereclaim/reclaim_test.go` `TestBackgroundRemovalStoppedAtShutdownIsFinishedAtStart`：`Start` 启动的后台删除在 `Close` 时被取消，记录留 `deleting`，下次启动删完。`internal/workspacefs/remove_test.go` `TestRemoveCheckoutStopsWhenCancelled`。缺口：没有"真实大目录返回 `in_progress` 之后后台删完"的端到端用例；"提交后出错仍返回 200"只在服务层断言了调用不报错，HTTP 层没有专门用例。 | Partial |

- 决策依据：D3，不变量 6。

## Reclaim Leaves Deliveries And Their Retention Unchanged

- 风险：回收顺带删了交付记录、交付义务、交付 ref、快照 ref 或 `v2_workspaces`；回收之后运维无法恢复成果；任务运行和
  工作流运行的交付读取受影响。
- 前置状态：一个能恢复的对话，项目镜像里有它的交付 ref；同项目有任务运行和工作流运行的交付。
- 触发：回收这个对话的目录；运维按 D6 从镜像恢复。
- 必须成立：回收前后交付记录、交付义务、交付 ref、快照 ref、`v2_workspaces` 不变；回收记录里的交付提交与起点提交能在
  镜像上打出完整 patch，并能在起点上应用；任务运行与工作流运行的变更视图和 patch 读取不变。本用例与
  [回合交付](turn-delivery.md#the-change-view-and-patch-read-the-delivery-not-the-working-directory) 的读取承诺一起成立。
- 禁止结果：回收改动了任何交付数据；回收后成果在镜像里找不到。
- 验证义务与证据：

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 交付数据不变 | 只有 `TestDeleteChatConversationRecordsReclaimInTheSameTransaction` 断言回收结束后 `v2_workspaces` 的行还在。没有回收前后交付记录、交付义务、交付 ref、快照 ref 逐项一致的断言。 | Partial |
| 运维能从镜像恢复 | `TestReclaimRemovesOnlyRecoverableCheckoutsUnlessDiscarded`：在镜像上执行 `git diff --stat <记录的起点> <记录的交付提交>`，包含该对话的改动文件。缺口：没有断言"打出的 patch 能在起点上 `git apply`"。 | Partial |
| 其他来源的交付读取不受影响 | 没有"回收之后读任务运行、工作流运行的变更视图和 patch"的用例。现有 `internal/server/workspace_delivery_test.go` 照常通过，但它们不包含回收。 | Missing |

- 决策依据：D6，不变量 7；[回合交付](../../../decisions/cloud/project-git/20261112-single-machine-turn-delivery.md) D4、D6。

## Operators Can List Reclaims And Discard Kept Workspaces Offline

- 风险：保留下来的目录和反复失败的回收没人看得到；离线丢弃在服务运行时执行，与服务争用目录；丢弃绕过 D4 的检查。
- 前置状态：有 `kept`、`pending`、`deleting`、`reclaimed` 的回收记录；服务在运行或已停止。
- 触发：运维运行列出子命令；在服务运行时和停止后分别运行离线丢弃。
- 必须成立：服务运行时能列出全部回收记录与原因、尝试次数、最后一次错误、释放量和依据的交付；数据目录锁被占用时离线
  丢弃拒绝执行；停服后离线丢弃只作用于 `kept` 的记录，按 D4 检查后删除，记为 `reclaimed` 并写明是运维丢弃。
- 禁止结果：服务运行时离线丢弃执行了删除；丢弃作用于非 `kept` 的记录或跳过路径检查。
- 验证义务与证据：

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 运行时可列出 | `reclaims_command_test.go` `TestReclaimsListsWhileTheServerRuns`：数据目录锁被占用时 `reclaims kept` 成功并输出表头，未知状态被拒。`TestOperatorDiscardsAKeptCheckout` 用 store 的 `WorkspaceReclaims(kept)` 列出了那条 `kept` 记录和路径。缺口：子命令测试库里没有记录，没有断言输出各条记录及其字段（原因、尝试次数、最后错误、释放量、依据的交付）。 | Partial |
| 离线丢弃与服务互斥 | `TestDiscardKeptRefusesWhileTheServerRuns`：数据目录锁被占用时 `discard-kept` 返回"already in use"，此时还没连数据库。 | Covered |
| 离线丢弃按 D4 检查并留痕 | `internal/workspacereclaim/reclaim_test.go` `TestOperatorDiscardsAKeptCheckout`：只处理 `kept`；删除后记录为 `reclaimed`、原因 `operator_discard`；对已回收的记录再丢弃被拒。路径检查走同一个 `RemoveCheckout`，由用例 4 的测试覆盖；`DiscardKept` 在删除前还会核对身份（登记、共享、活会话）。子命令本身走的 `agentruntime.OfflineCheckouts` 没有端到端用例。 缺口：没有从 `DiscardKept` 入口验证 D4 检查不通过时拒绝删除的用例。 | Partial |

- 决策依据：D6，不变量 4、5、6。
