# Node 插件安装

依据[已实现的 Node plugin-runtime 根决策](../../../decisions/node/plugin-runtime/0-node-installs-plugins-into-workspace-home.md)。
证据来自 desktop 的实际包、本地 HTTP、Node 进程与 SQLite 测试。状态按直接覆盖范围维护；跨仓库 M1 联调不由这些测试替代，由 [cluster `task plugin:acceptance`](https://github.com/ora-space/cluster/blob/main/docs/testing/plugin-step.md) 提供：
真实 Node 从 GitHub Releases 下载通用发布包、校验后发布到 Workspace 数据卷的
`/var/lib/ora/node/plugins/installed/<ns>/<id>/<version>/`。它不覆盖原生 target 发布包与失败路径，下表状态不变。

## Plugins Appear Only After Digest And Package Verification

### 风险

下载被篡改或包不合法的插件被放进插件根，随后被 Agent 会话启动。

### 前置状态

Workspace 插件根可写；本地 HTTP 服务提供 release。

### 触发

安装：同版本已装、未装、摘要不符、包校验失败、无匹配 target、下载失败三次、插件正被会话使用。

### 必须成立

- 只有 SHA-256 与载荷一致且通过安装期校验后才原子出现在插件根；D2 表中每种情况得到对应的逐项结果。
- 同版本已装时不下载；安装执行不启动插件进程。
- 正被会话租用的插件返回 `plugin_in_use`，不替换、不移除。

### 禁止结果

校验失败的包出现在插件根；安装过程中插件根出现半成品；替换使用中的插件。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| D2 表逐项 | `apps/ora-node/tests/plugins.rs`：`install_retry_upgrade_and_remove_respect_use_leases`、`invalid_packages_leave_no_discoverable_installation`、`unmatched_target_never_downloads`、`http_installation_and_three_failed_attempts_have_bounded_results`；覆盖有效包、摘要、版本身份、无 target、三次 HTTP 失败；原生 target 成功与原子发布边界故障注入仍缺少直接证据 | Partial |
| 使用中保护 | `apps/ora-node/tests/plugins.rs::install_retry_upgrade_and_remove_respect_use_leases`：持有实际 catalog 租约拒绝升级／移除，释放后升级和移除成功并保留 data | Covered |
| 根目录隔离 | `apps/ora-node/tests/plugins.rs::linked_plugin_root_fails_the_execution_without_following_it`：符号链接根整体失败，外部目录无写入 | Covered |

### 决策依据

plugin-runtime 根决策 D2、D4，不变量 1、2、4；agent-runtime D7。

## Install Executions Rerun Safely After Restart

### 风险

Node 在安装中途重启，留下临时文件或永远没有终态。

### 前置状态

安装执行已受理、尚无终态。

### 触发

Node 重启。

### 必须成立

- 本地未围栏执行恢复时清理临时下载目录，按原输入重跑，最终只发出一个终态事件。
- 云端旧 incarnation 不复用旧许可；未完成执行以 `interrupted` 结算，新的尝试必须取得新许可。
- 已完成事件保持原执行身份，重启不改变结果。

### 禁止结果

两个终态事件；残留临时目录；云端旧 incarnation 绕过许可自动重跑。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 本地重跑与临时清理 | `apps/ora-node/tests/standalone/repository_plugins.rs::killed_download_recovers_original_input_and_cleans_temporary_package`：阻塞下载时查询 Running，杀死真实进程，重启不重发命令即可完成原执行并清理临时目录；下载完成后、发布前的精确故障点仍缺证据 | Partial |
| 云端旧许可隔离 | `crates/node-db/src/tests/plugin.rs::controlled_plugin_start_requires_current_open_incarnation`：真实 SQLite 重开、关闭及换 incarnation 后拒绝开始；完整云端进程恢复结果还需联调 | Partial |
| 终态与回传原子性 | `crates/node-db/src/tests/plugin.rs::terminal_fact_cannot_commit_without_its_event`：outbox 写失败回滚终态 | Covered |
| 原事件重放、准确 ACK 与查询 | `apps/ora-node/tests/standalone/repository_plugins.rs::plugin_result_replays_after_restart_until_exact_ack_and_remains_queryable`；`crates/node-db/src/tests/plugin.rs::input_result_and_exact_event_survive_restart_and_acknowledgement` | Covered |

### 决策依据

plugin-runtime 根决策 D3，不变量 3。
