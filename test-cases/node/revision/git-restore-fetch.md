# 恢复从 Git Server 拉取前序交付 ref

依据[已批准的 node/revision 后续决策](../../../decisions/node/revision/20261012-push-delivery-ref-and-fetch-prior-revision.md)
（下称“本 ADR”）D5、D6，它修改了[恢复前序 Revision 决策](../../../decisions/node/revision/20261010-restore-prior-revision-before-session.md)
的取得方式与失败分类。本 ADR 尚未实现：desktop 的 Node 仍申请下载授权、下载 bundle 并向 `origin` 补取基础提交，所以
拉取相关义务为 `Missing`。恢复在会话中的位置与失败时不启动插件不随取得方式变化，`apps/ora-node/tests/agent_session/resume.rs`
以替身恢复器直接证明这部分。cluster 的 `revision:resume:acceptance` 改走 Git 后只作补充证据，不据此标记 `Covered`。

## Restore Fetches The Prior Delivery Ref Without Origin

### 风险

恢复仍依赖用户远端，远端历史改写后恢复失败；取到的提交与会话输入不符却被检出；凭据进入工作负载身份的取入命令；
远端历史被改写时 Agent 没有得到提示。

### 前置状态

会话输入带 `priorRevision{revisionId, finalCommit, baseCommit, gitRepositoryUrl, deliveryRef}`；checkout 已定位、插件已确认，
尚未交还工作负载用户、尚未启动插件。

### 触发

恢复前序 Revision：Git Server 上有该交付 ref；本次 clone 的基础提交与前序的基础提交相同或不同；前序的基础提交
不再是 `origin/<默认分支>` 的祖先，或本地没有它。

### 必须成立

- 校验输入：`finalCommit`、`baseCommit` 是完整提交 ID，地址与 ref 符合凭据隔离用例的规则。
- 以 Node 身份在传输仓库中、以会话执行 ID 拉取 `<deliveryRef>`，协商起点是本次 clone 的基础提交；取到的必须是
  提交且等于 `finalCommit`。
- 再以工作负载身份、沿用交付的受管执行与加固配置，从传输仓库取入 checkout；只有这一条命令放开 `file` 协议。
- `git checkout --force -B <默认分支> <finalCommit>`，`origin/<默认分支>` 不动；全程不访问 `origin`。
- `baseCommit` 不是 `origin/<默认分支>` 的祖先或本地没有它时，在首条提问末尾追加提示。
- 拉取与取入共用 5 分钟时限；结束后删除传输仓库。

### 禁止结果

恢复访问 `origin`；检出的提交不等于 `finalCommit`；工作负载身份的命令带上证书；远端历史被改写时没有提示。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 从交付 ref 拉取并核对提交 | 尚无实现；cloud `internal/gitserver/service_test.go::TestReadOnlyExecutionFetchesButCannotPush` 只证明 Git Server 允许只读执行拉取交付 ref | Missing |
| 以工作负载身份取入且不访问 `origin` | 尚无实现 | Missing |
| 检出到默认分支、`origin` 不动 | desktop `apps/ora-node/src/revision/restore/tests/git.rs::restores_the_prior_final_commit_onto_the_clone_branch`：默认分支停在前序最终提交、工作区干净、`origin/main` 停在本次 clone 的基础提交。该测试经 bundle 取入，需改为经传输仓库取入 | Partial |
| 改写提示 | `apps/ora-node/tests/agent_session/resume.rs::a_rewritten_remote_history_is_noted_in_the_first_turn`：恢复器报告历史被改写时，首条提问末尾恰好追加一条提示。缺口：按会话输入的 `baseCommit` 判定是否改写尚无实现 | Partial |
| 时限与清理 | 尚无实现 | Missing |

### 决策依据

本 ADR D5，不变量 8；恢复前序 Revision 决策 D1、D3 第 5 步、D4。

## Restore Failures Are Classified And Never Start The Agent

### 风险

暂时失败被报告为 `prior_revision_base_unavailable`，Cloud 把前序 Revision 永久标为不可续接；Node 自身的身份配置
错误（`401`）也让 Revision 永久不可续作；恢复失败后 Agent 仍在不完整的 checkout 上启动。

### 前置状态

续作会话正在恢复前序 Revision。

### 触发

Git Server 上没有该交付 ref；返回 `403`；返回 `401`；取到的提交不等于 `finalCommit` 或不是提交；`5xx`、网络、TLS
错误、重试耗尽或时限到期；传输仓库、取入或检出在本地失败。

### 必须成立

- 没有该交付 ref 与 `403`：`agent_failed` 加 `prior_revision_base_unavailable`。
- `401`、提交不符、`5xx`、网络、TLS、重试耗尽、时限到期与本地失败：`agent_failed` 加 `prior_revision_unavailable`。
- `5xx`、网络与 TLS 错误在时限内最多重试 3 次。
- 任何恢复失败都在插件进程、租约与 Thread 记录出现之前结束会话。

### 禁止结果

`401` 或暂时失败报告为 `base_unavailable`；恢复失败后插件启动。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 失败分类 | 尚无实现。现有 `apps/ora-node/src/revision/restore/tests/git.rs::an_unfetchable_base_is_base_unavailable_and_leaves_the_checkout`、`::an_unreachable_origin_is_unavailable_not_base_unavailable` 与 `restore/tests/restorer.rs::refusal_mismatch_and_deadline_are_unavailable` 证明的是 bundle 与 `origin` 的旧分类 | Missing |
| 重试上限 | 尚无实现；`restore/tests/restorer.rs::retries_server_errors_and_renews_a_forbidden_grant_once` 针对下载授权 | Missing |
| 失败时插件从未启动 | desktop `apps/ora-node/tests/agent_session/resume.rs::a_failed_restore_ends_the_session_before_any_plugin_starts`：两种失败码都以 `agent_failed` 加对应 `detail` 结束会话，没有插件进程、租约或 Thread 记录。该测试用替身恢复器，与取得方式无关；输入夹具仍带 `bundle`，需随 Node 协议修改 | Covered |

### 决策依据

本 ADR D6，不变量 9；恢复前序 Revision 决策 D1。
