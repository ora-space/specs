# 工作流派发前 MCP 预检

## Workflow Mcp Preflight Must Reject Missing Selections Before Dispatch

### 风险

若在已选 MCP 缺失时开始调度，前面的节点可能已经产生副作用。若把配置或能力错误也提前到运行门禁，用户会收到错误的恢复提示。

### 前置状态与触发

冻结图包含启用的 MCP 绑定。分别对 `Pending` 运行调用 start，对可重启的运行（含 `Succeeded`）调用 restart，对可恢复的 `Failed` 或 `Cancelled` 运行保留快照调用 resume。另以 `Running` 运行验证重复命令。

### 必须成立

- 已启用绑定对应的插件缺失、包无效或未提供 MCP 时，调度命令返回 `workflow_mcp_not_found`，可定位节点与 MCP，状态不变。只在未执行的条件分支、循环体或已经成功的节点中出现的已选 MCP 也受检查。
- 未选择或禁用的绑定允许其插件缺失。运行期间卸载依赖后，重复 start 仍返回当前运行，restart 仍返回 `workflow_run_not_restartable`。
- 配置不完整或 Agent 能力不足要在节点 session setup 报错；运行开始后卸载、或在预检和派发之间卸载，由后续 session setup 拒绝已选插件缺失。已有 Session 的恢复保持原选择并 fail closed。
- 错误只含节点 id、节点标题和 MCP id；预检只读。

### 禁止结果

预检拒绝后不得派发节点、创建 Session、发送 ACP、改写 worktree 或改变运行状态；不得暴露配置实值，亦不得以部分 MCP 集合继续。

### 验证义务与证据

| 决策与验证义务 | 受测提交中的直接证据或缺口 | 状态 |
| --- | --- | --- |
| D1：卸载已选 MCP 后 start 被拒，状态仍为 `Pending` | [移除后启动](https://github.com/ora-space/desktop/blob/3cc1f0e7e89d432a537c0fc2e351a94fb8bf13a1/crates/backend/src/workflow/run/operations/precheck_tests.rs#L394)比较完整错误并重新读取 `Pending`；重装后同一运行可以启动 | `Covered` |
| D1：同一 MCP 缺失阻止 restart 和保留快照 resume，原状态不变 | [取消后移除 MCP](https://github.com/ora-space/desktop/blob/3cc1f0e7e89d432a537c0fc2e351a94fb8bf13a1/crates/backend/src/workflow/run/operations/precheck_tests.rs#L425)分别调用 restart 和保留快照的 resume，两次拒绝后持久运行完全相同；`Succeeded` 的 restart 和 `Failed` 的 resume 尚缺同样的对照 | `Partial` |
| D1：卸载后重复 start 和对 Running 的 restart 仍给出原结果 | [移除后重复命令](https://github.com/ora-space/desktop/blob/3cc1f0e7e89d432a537c0fc2e351a94fb8bf13a1/crates/backend/src/workflow/run/operations/precheck_tests.rs#L620)断言再次 start 返回 `Running`，对该运行 restart 仍返回 `workflow_run_not_restartable` | `Covered` |
| D1：检查整张图，忽略禁用绑定 | [禁用绑定时启动](https://github.com/ora-space/desktop/blob/3cc1f0e7e89d432a537c0fc2e351a94fb8bf13a1/crates/backend/src/workflow/run/operations/precheck_tests.rs#L455)验证禁用绑定被跳过；尚缺对条件分支、循环体和已成功节点的直接证明 | `Partial` |
| D2：包无效及同 ID 非 MCP 均按缺失处理 | [损坏 MCP 包](https://github.com/ora-space/desktop/blob/3cc1f0e7e89d432a537c0fc2e351a94fb8bf13a1/crates/backend/src/workflow/run/operations/precheck_tests.rs#L655)和[同 ID 的 Skill 包](https://github.com/ora-space/desktop/blob/3cc1f0e7e89d432a537c0fc2e351a94fb8bf13a1/crates/backend/src/workflow/run/operations/precheck_tests.rs#L467)分别断言 start 被拒且运行保持 `Pending` | `Covered` |
| D2：已安装但配置不完整的 MCP 通过预检，随后在 session setup 失败 | [缺少必填设置的运行获准启动](https://github.com/ora-space/desktop/blob/3cc1f0e7e89d432a537c0fc2e351a94fb8bf13a1/crates/backend/src/workflow/run/operations/precheck_tests.rs#L620)；[节点解析对配置不完整的显式选择返回错误](https://github.com/ora-space/desktop/blob/adaa6672d10f60be2edd10ef61a61102b04a9d89/crates/backend/src/session_setup/mcp/selection_tests.rs#L85)。两个独立用例覆盖各自的判断边界，未在同一次运行观测节点失败 | `Partial` |
| D2：预检之后发生卸载，节点按 setup 错误失败，恢复不会放宽选择 | 缺少贯穿调度与节点 setup 的竞态用例 | `Missing` |
| D4：错误只含节点与 MCP 身份，标题缺省时仍可定位 | [比较完整后端错误](https://github.com/ora-space/desktop/blob/3cc1f0e7e89d432a537c0fc2e351a94fb8bf13a1/crates/backend/src/workflow/run/operations/precheck_tests.rs#L394)与[前端按标题和 MCP id 显示](https://github.com/ora-space/desktop/blob/3cc1f0e7e89d432a537c0fc2e351a94fb8bf13a1/packages/app-shell/src/i18n/contract-error.test.ts#L110)。缺少无标题时显示回退及设置实值不外露的直接断言 | `Partial` |
| D1/D4：拒绝不造成 Session、ACP 或 worktree 副作用 | [拒绝目标依赖时保留未提交文件](https://github.com/ora-space/desktop/blob/3cc1f0e7e89d432a537c0fc2e351a94fb8bf13a1/crates/backend/src/workflow/run/operations/precheck_tests.rs#L483)直接观测 worktree，尚未观测 Session 创建或 ACP 流量 | `Partial` |

### 决策依据

[工作流派发前拒绝缺失的已选 MCP](../../../../../../decisions/desktop/core/agent/session/mcp/20261119-workflow-mcp-preflight.md)，D1、D2、D4 及不变量 1–3、5。

## Snapshot Switch Resume Must Validate The Target Graph Before Rollback

### 风险

校验旧图会漏掉目标快照新增的依赖，也会错误阻止用户切到已经修复的快照。先恢复 checkpoint 再报错，会让被拒绝的命令改写工作目录。

### 前置状态与触发

运行处于可 resume 状态；保留可识别的 worktree 内容。用 `snapshot_id` 指定与当前冻结图有不同 MCP 或 Skill 绑定的目标快照并调用 resume。

### 必须成立

- 在回滚（checkpoint 恢复）或任何 worktree 改动之前，从目标图解析所需 Skill，检查启用的 MCP。
- 目标图缺少 MCP 时返回 `workflow_mcp_not_found`，缺少 Skill 时返回对应的 Skill 错误。运行状态、已选快照及 worktree 内容与命令前一致。
- 仅旧快照含有缺失绑定时允许切换到满足依赖的目标快照。

### 禁止结果

只查旧图、忽略新图中的启用绑定、先执行回滚再报依赖错误，或让被拒绝的运行进入 `Running`。

### 验证义务与证据

| 决策与验证义务 | 受测提交中的直接证据或缺口 | 状态 |
| --- | --- | --- |
| D3：目标图缺 MCP 或 Skill 时在回滚前拒绝，保留调用前运行和 worktree | [含真实 checkpoint 的两种目标图](https://github.com/ora-space/desktop/blob/3cc1f0e7e89d432a537c0fc2e351a94fb8bf13a1/crates/backend/src/workflow/run/operations/precheck_tests.rs#L483)分别拒绝缺失 MCP、Skill，核对完整持久运行与未提交文件。恢复依赖后同一请求切换到目标快照并写回 checkpoint 内容 | `Covered` |
| D3：旧图缺失、目标图已修复时可以切换 | 缺少只改变目标绑定的直接主线证据 | `Missing` |

### 决策依据

[派发预检 D3](../../../../../../decisions/desktop/core/agent/session/mcp/20261119-workflow-mcp-preflight.md)及[Plugin Skill 来源和物化规则](../../../../../../decisions/desktop/core/effect/skill/0-plugin-skill-source.md)。
