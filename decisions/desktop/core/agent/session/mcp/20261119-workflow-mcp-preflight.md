---
status: implemented
---

# 工作流派发前拒绝缺失的已选 MCP

Desktop 在 `start`、`restart` 以及保留或切换快照的 `resume` 开始派发前，检查本次要执行的整张冻结图。任何节点启用的 MCP 无法在已安装插件目录中解析时，命令返回 `workflow_mcp_not_found`，运行保留原状态。切换快照的检查顺序见 D3。

本决策只约束 Desktop。Cloud 延用自己的[工作流发布与启动门禁](../../../../../cloud/workflow/0-cloud-side-scheduler-owns-workflow-execution.md)，两者都在运行期判断环境是否可用，用户可以先编辑草稿。

## 继承与修改

| 前序约定 | 处理 | 当前规则 |
| --- | --- | --- |
| [工作流显式选择](20260912-workflow-explicit-selection.md) D1 | 修改 | 冻结节点中显式启用的 ID 构成白名单；派发前发现已选 MCP 缺失时拒绝整个运行 |
| 显式选择 D2 | 继承 | Session 持久化自己的选择，恢复读取原选择，缺失时 fail closed |
| 显式选择 D3 | 继承 | ACP 发送边界继续记录不含配置实值的诊断身份 |
| 显式选择 D4 | 修改 | 是否有可用的已安装 MCP 在派发前判断；配置和 Agent 能力仍在 session setup 判断。草稿、发布和 Skill 物化规则沿用前序决策 |
| [MCP 健康探测](20260916-mcp-health-probing.md) | 继承 | 健康状态独立于选择资格；预检不读取探测结果 |

## 决策

### D1：派发前校验整张目标图

`startWorkflowRun`、`restartWorkflowRun` 和 `resumeWorkflowRunFromFailure` 在即将开始新一轮调度时，遍历目标快照中各节点的 `enabled: true` MCP 绑定，按完整插件 ID 查询 session setup 使用的已安装插件目录。未选、禁用及空集合沿用显式选择语义。保存草稿、发布或创建运行时不增加这项 MCP 校验。

`Pending` 的 start、允许重启的运行（含 `Succeeded`），以及可恢复的 `Failed` 或 `Cancelled` 运行进入这道门禁。已在运行时重复 start 仍返回当前运行；restart 一个 `Running` 运行仍返回 `workflow_run_not_restartable`。这两条路径上卸载插件也不会把原来的结果改为依赖错误。

### D2：把安装可用性与配置资格分开

缺少安装包、安装包无效、相同 ID 的插件没有提供 MCP 时，预检都拒绝派发。提示说明节点所需的 MCP 不可用，用户可以安装或恢复符合该 ID 的 MCP。已安装的有效 MCP 即使配置不完整、Agent 不支持该 transport 或健康探测失败，也通过这项安装检查。

运行已经派发后，若插件在后续节点开始前被移除，按前序 D1 在 session setup 以 `session_mcp_setup_failed` 报告节点失败。预检与第一次派发之间发生卸载也由这层检查兜底。配置不完整、能力不足同样在 session setup 失败；恢复既有 Session 遵从前序 D2，不能缩减已选集合继续执行。

### D3：切换快照的 resume 在回滚前校验目标图

保留快照时检查运行当前的冻结图。指定其他快照时，读取目标快照，对其中启用的 MCP 和声明的 Skill 完成依赖检查，再允许 checkpoint 恢复或其他 worktree 修改。拒绝时保留原运行状态、快照和 worktree。仅旧图缺失、目标图已经消除此绑定时，可以用目标快照继续。Skill 按其既有[来源与物化规则](../../../effect/skill/0-plugin-skill-source.md)解析。

### D4：错误足以定位节点和依赖

`workflow_mcp_not_found` 携带节点 id、节点标题和 MCP id。用户界面在标题缺省时显示节点 id。参数只暴露这些身份；command、arguments、env、URL、headers 和其他 Setting 实值不会进入此错误。

## 后果

冻结图是部署单元。同一次运行从任意合法入口开始时，得到相同的依赖判定。代价是未命中的条件分支、不会进入的循环体，甚至 resume 后已经成功且不会重跑的节点，其 MCP 缺失也会阻止调度。旧的 `Failed` 或 `Cancelled` 运行可能因此无法续跑；用户可以恢复所需插件，或改用已去掉该绑定的目标快照。

## 不变量

1. 预检时缺失的已选 MCP 使运行在首个节点派发前保持原状态。
2. 预检和 session setup 以同一插件目录和完整 ID 判断 MCP 的安装可用性。
3. 预检只读，不创建 Session、不发送 ACP、不改写 Session 选择或 worktree。
4. 切换快照时在 checkpoint 恢复及任何 worktree 改动前检查目标图。
5. 诊断只公开节点和 MCP 身份；配置与能力问题仍由节点 setup 报告。

## 为什么不是这些替代方案

| 方案 | 取舍 |
| --- | --- |
| 所有错误都等到节点 setup | 已知的安装缺失也要等其他节点可能产生副作用后才能发现 |
| 只检查还会执行的节点 | 能恢复更多旧运行，但需预判条件、循环与恢复路径；同一冻结图的依赖判断会随入口或中间结果变化 |
| 把配置、能力与健康加入预检 | 用户会在运行级收到需由特定节点的 Agent、设置或探测解释的错误 |
| 每次节点派发都重扫整图 | session setup 已按当时的显式选择解析该节点；全图重扫会让不相关节点的卸载中断进行中的运行，且仍消除不了检查后的竞态 |

## 核心测试依据

[派发前 MCP 预检](../../../../../../test-cases/desktop/core/agent/session/mcp/workflow-mcp-preflight.md)记录派发、错误处理、切换快照和 checkpoint 恢复的直接证据与缺口。
