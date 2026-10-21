# Scheduled Run 结果注解与产物持久化

本用例服务后续 ADR
[20261021-run-result-annotation-and-artifacts.md](../../../../decisions/desktop/core/scheduled-task/20261021-run-result-annotation-and-artifacts.md)
（决策单 ora-space/circus#441，覆盖事项 #439 / #440），覆盖运行结果注解与产物持久化的验证义务。

## Business Verdict Must Be Independent And Machine Readable

- 风险：业务结论复用执行层状态，破坏 `status/failure_reason` 语义、故障映射与通知，或无法表达「执行成功但业务失败」。
- 前置状态：运行已 `succeeded`；Agent 最终响应含或不含哨兵。
- 触发：runtime 在 `EndTurn` 收尾解析最终 Assistant 轮并写入 verdict；或旧任务运行无 verdict 被读取。
- 必须成立：`verdict` 独立、可空、取值 `pass/warn/fail/blocked/idle/range_unresolved` 且带受限 detail 与独立 `verdict_at`；
  `status=succeeded` 与 `verdict=fail` 合法并存；`failed/cancelled/timeout/skipped_*` 与 `missed` 均无 verdict；`accepted/running`
  无 verdict；verdict 在 `RunFinished` 发布前单次写入（前端刷新即见）；哨兵无匹配或格式非法、任务未采用哨兵约定时为
  「未报告」；runtime 只解析、不向 prompt 注入内容；旧运行无 verdict 显示「未报告」且行为不变。
- 禁止结果：verdict 触发失败通知或进入故障统计；复用 `failure_reason`；`verdict_at` 改写 `updated_at`；缺 verdict 时行为回退；
  解析失败改写运行事实；runtime 向任务 prompt 拼装哨兵说明。
- 决策依据：后续 ADR D1、D2、根决策 D3 与不变量 1–3。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| `succeeded + verdict=fail` 在历史中独立展示，不触发失败语义 | `ora_db::repository::scheduled_run::tests::annotate_verdict_persists_and_leaves_execution_status_untouched`；`scheduled-task-history-dialog.test.tsx::shows a business fail verdict without turning the run into a failure` | `Covered` |
| 哨兵解析：LF 归一、大小写不敏感、行尾空白忽略、不跨行、取首个匹配（含 fence 内）、多余标记忽略 | `ora_domain::scheduled_run::tests::parses_the_case_insensitive_sentinel_and_following_detail`、`takes_the_first_marker_and_rejects_an_illegal_value`、`ignores_non_verdict_marker_prefixes_without_a_valid_token` | `Covered` |
| `QA_DETAIL` 只认紧随标记的下一行，上限 2000 字符；无匹配或格式非法为「未报告」且不报错 | `ora_domain::scheduled_run::tests::only_reads_detail_from_the_immediately_following_line`、`rejects_an_over_long_detail`、`takes_the_first_marker_and_rejects_an_illegal_value` | `Covered` |
| 每个 `succeeded` 运行只解析、写入一次；未采用哨兵约定的任务恒「未报告」，runtime 不注入 prompt | 尚无直接证据：`finish_succeeded` 只在收尾调用一次 `annotate_verdict`，且 runtime 不拼装 prompt；缺运行级测试 | `Partial` |
| 非法枚举 / detail 超长 / 运行不存在 / 运行不是 `succeeded` 时写入被拒，且不改写运行事实 | `ora_db::repository::scheduled_run::tests::annotate_verdict_rejects_a_run_that_has_not_succeeded`（running + missing）；domain 解析测试覆盖非法枚举与超长 detail | `Covered` |
| `failed/cancelled/timeout/skipped_*` 与 `missed` 均无 verdict | SQL 仅允许 `succeeded` 写入；`annotate_verdict_rejects_a_run_that_has_not_succeeded` 覆盖 running，其余终态未逐一落测 | `Partial` |
| verdict 先于 `RunFinished` 落库，前端刷新即见最终 verdict | 尚无直接证据：`finish_succeeded` 顺序为 transition → `annotate_verdict` → `publish_finished`，缺事件顺序测试 | `Partial` |
| `verdict_at` 独立，不推进 `updated_at`（不改变保留淘汰排序） | `ora_db::repository::scheduled_run::tests::annotate_verdict_persists_and_leaves_execution_status_untouched` | `Covered` |
| 旧任务无 verdict 显示「未报告」，历史裁剪/通知/保留不回退 | `ora_db::migration::schema::schema_v0020::tests::adds_nullable_verdict_columns_without_rewriting_history`；`scheduled-task-history-dialog.test.tsx::shows no verdict when the run never reported one` | `Covered` |

## Artifacts Must Outlive The Provisioning Worktree

- 风险：产物停留在隔离 worktree，回收后丢失；或未受声明/大小/数量约束而膨胀或越权收集。
- 前置状态：Agent 在固定目录 `.ora-scheduled-artifacts/` 产出白名单类型文件；运行收尾。
- 触发：`succeeded` 收尾时 runtime 登记并 copy 产物；worktree 随后被回收；用户从运行历史打开产物。
- 必须成立：只在 `succeeded` 收尾采集，且采集早于 janitor 丢弃 worktree（`failed/cancelled/timeout` 与中断运行不采集）；
  固定目录 + 类型白名单之外的文件不登记；copy 后 worktree 保留判定不变（无改动仍可回收）；按 `run_id` 建立清单
  （已登记：名称/类型/大小/登记时间）与被拒清单（名称 + 原因），可列出/打开/下载；单产物 ≤10 MiB、单运行 ≤64 个、总 ≤64 MiB、
  每任务保留 ≤10 个运行的产物（独立于 worktree 全局 10、按最旧运行淘汰）；超限或非法路径进入被拒清单且可见；落盘顺序
  copy→临时文件→校验大小→原子 rename→写清单，崩溃对账幂等（孤文件清理、缺文件删索引、半写不残留）；产物随 100 条历史裁剪
  清理，保留 worktree 链接运行例外保留；软删除任务的产物由启动 sweep 清理，禁用任务的产物保留；产物存储只含从固定目录
  copy 的文件与清单元数据，Ora 不往其中写入凭据。
- 禁止结果：产物随 worktree 一起被回收；无索引或被拒不可见；目录外文件被登记；路径穿越/符号链接越界；清单指向半写/截断文件；
  只写了产物的 worktree 被判有改动而保留；禁用任务的产物被 sweep 删除；Ora 把模型 Key、token 或环境变量写进产物存储。
- 决策依据：后续 ADR D3、D4、根决策 D4 / D8 与不变量 4、5。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| 采集早于 janitor 丢弃：只写了产物的运行判定无改动仍回收，但产物先落库；采集失败进被拒清单、不改写运行事实 | `ora_backend::scheduled_task::worktree::tests::untracked_artifacts_do_not_mark_a_worktree_modified`、`committed_artifacts_do_not_mark_a_worktree_modified`；采集顺序与失败拒绝尚无运行级测试 | `Partial` |
| `failed/cancelled/timeout` 与启动恢复结算为 `failed` 的中断运行不采集（取舍） | 尚无直接证据：仅 `finish_succeeded` 调用采集；缺运行级测试 | `Partial` |
| 固定目录 + 白名单闭集（`markdown/json/txt/log/xml/html`）之外的文件不登记 | `ora_application::scheduled_task::artifact_store::tests::reports_unsupported_types_without_copying_them`、`collects_whitelisted_files_recursively_and_copies_them_out_of_the_worktree` | `Covered` |
| 产物目录下未跟踪、已暂存、已修改或已提交的路径都不计入改动判定；目录外的改动仍使 worktree 保留 | `ora_backend::scheduled_task::worktree::tests::untracked_artifacts_do_not_mark_a_worktree_modified`、`committed_artifacts_do_not_mark_a_worktree_modified`、`a_change_outside_the_artifact_directory_still_marks_the_worktree_modified` | `Covered` |
| 登记后 worktree 被回收，产物仍可从运行历史访问；copy 不改变保留判定 | `ora_application::scheduled_task::artifact_handlers::tests::copy_exports_an_artifact_after_the_worktree_was_reclaimed`；worktree 判定由上面的 artifact-only 测试覆盖 | `Covered` |
| 超限 / 路径穿越 / 符号链接越界 / Windows ADS、大小写、NFC、硬链接越界记入被拒清单且可见 | `artifact_store::tests::rejects_files_over_the_single_artifact_bound`、`reports_windows_alternate_data_stream_names_as_unsafe`、`rejects_symlinks_and_case_only_duplicates`、`rejects_a_file_that_shares_its_inode_with_an_outside_name`、`artifact_store::collection::tests::accepts_the_sixty_fourth_artifact_and_rejects_the_sixty_fifth`、`accepts_the_exact_total_bound_and_rejects_one_byte_more` | `Covered` |
| 落盘顺序（临时文件 + 大小校验 + 原子 rename + 清单后置）崩溃对账幂等（含 rename 后写清单前崩溃的取舍） | `artifact_store::tests::reconcile_removes_unregistered_files_and_reports_missing_or_resized_rows`、`reconcile_removes_a_file_with_no_registered_row`、`artifact_handlers::tests::sweep_deletes_rows_whose_file_vanished_and_files_without_rows` | `Covered` |
| 每任务 ≤10 个运行的产物按最旧运行淘汰，独立于 worktree 全局 10；链接例外优先于 10 上限 | `ora_application::scheduled_task::artifact_handlers::tests::sweep_keeps_the_newest_ten_unlinked_runs_and_exempts_linked_runs` | `Covered` |
| 产物随历史裁剪清理（先删索引行、事务提交后删文件，失败由启动 sweep 兜底） | `artifact_handlers::tests::sweep_removes_orphan_rows_and_soft_deleted_task_storage`、`ora_db::repository::scheduled_artifact::tests::pruned_runs_surface_as_missing_so_the_sweep_can_remove_their_artifacts`；100 条裁剪与 sweep 的联动尚无集成测试 | `Partial` |
| 软删除或已不存在的任务的产物由启动 sweep 清理；禁用任务的产物保留、不被 sweep 清理 | `artifact_handlers::tests::sweep_removes_orphan_rows_and_soft_deleted_task_storage` 覆盖软删除；禁用任务保留尚无直接测试 | `Partial` |
| 旧运行无产物显示「无产物」；访问跟随运行可见性；清单只存元数据 | `scheduled-task-artifacts-dialog.test.tsx::shows the empty state for a run that produced no artifacts`；`artifact_handlers::tests::list_and_read_reject_pruned_or_soft_deleted_runs` | `Covered` |
| 产物存储只含固定目录 copy 的文件与清单元数据，Ora 不写入模型 Key、token 或环境变量 | 尚无直接测试：实现只写入从固定目录复制的文件与清单行；待真实 local 集成验收确认 | `Partial` |

## Local Artifacts And Verdicts Must Stay On The Device

- 风险：本地产物/verdict 被意外加入 Cloud 同步或上传。
- 前置状态：local 定时运行产出 verdict 与产物。
- 触发：登记、读取、重启。
- 必须成立：产物与 verdict 只落本机、不自动上传、不跨设备同步；重启后历史与产物仍可读。
- 禁止结果：把本地产物写入账号数据 API 或 Cloud 上传路径。
- 决策依据：后续 ADR D5 与不变量 6、7。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| 产物/verdict 存储不调用 Cloud 上传/同步；重启后历史与产物可读 | `artifact_handlers::tests::copy_exports_an_artifact_after_the_worktree_was_reclaimed` 覆盖源目录移除后仍可读；Cloud 边界与重启尚无直接测试 | `Partial` |
| 本地产物随运行历史驻留本机，与 D1 数据驻留一致 | 尚无直接证据 | `Missing` |
