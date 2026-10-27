# Agent 模板与输出

依据[工作流根决策](../../../decisions/cloud/workflow/0-cloud-side-scheduler-owns-workflow-execution.md)。下列源码证据固定于 cloud 合并提交 `f433499be5af0afc59a0c4d8400c5090ffb8af7a`；真实模型栈证据的版本边界单独列出。

## Agent prompt and final output follow the committed scope

- 风险：派发原始模板、读取错误轮次或 attempt 的回答，或以会话结束原因冒充节点输出，导致下游与 Issue 收到错误结果。
- 前置状态：Start 提供输入，Agent 的 prompt 引用变量，Output 绑定 Agent 输出；运行可能处于 loop 或 retry。
- 触发：派发 Agent，会话结束，调度下游 Output。
- 必须成立：prompt 按所属作用域渲染；成功终态从当前 attempt 的完整初始 turn 历史提交最终回答；Output 按绑定取值，Issue 投影输出。失败、取消或截断不得发布部分答案；重放不得改写已提交结果。
- 禁止结果：未解析模板被派发、跨 attempt 串值、结束原因作为回答、截断历史静默发布或终态重放改写。
- 决策依据：工作流根决策 D1、D5 与 D6。

| 验证义务 | 直接证据与范围 | 状态 |
|---|---|---|
| 输入经模板、Agent 回答、Output 绑定和 Issue 投影 | [PostgreSQL 回归](https://github.com/ora-space/cloud/blob/f433499be5af0afc59a0c4d8400c5090ffb8af7a/internal/core/workflow_agent_output_db_test.go) 的 `TestWorkflowAgentOutputAndPrompt`；[#679 真实模型栈验收](https://github.com/ora-space/circus/issues/679)覆盖手动和 Issue 两条路径，但仅适用于旧候选 `b327040`。合并后主干真实栈复验由 [#682](https://github.com/ora-space/circus/issues/682) 承接 | Partial |
| attempt 隔离、无效模板拒绝、失败及截断不发布、取消保留已完成结果 | 同一文件的 `TestWorkflowAgentOutputRetryIsolation`、`TestWorkflowAgentPromptRejectsUnresolvedTemplate`、`TestWorkflowAgentOutputRequiresSuccessfulTurn`、`TestWorkflowAgentOutputUsesCompleteHistory` 和 `TestWorkflowAgentOutputSurvivesRunCancellation` | Covered |
| loop 作用域与 retry 派发 | [PostgreSQL 回归](https://github.com/ora-space/cloud/blob/f433499be5af0afc59a0c4d8400c5090ffb8af7a/internal/core/workflow_agent_scope_db_test.go) 的 `TestWorkflowAgentPromptUsesLoopScopeOnRetry`；iteration 真实模型栈仍缺直接证据 | Partial |
| 超时排空不发布伪输出 | [超时回归](https://github.com/ora-space/cloud/blob/f433499be5af0afc59a0c4d8400c5090ffb8af7a/internal/core/workflow_session_drain_db_test.go) 的 `TestWorkflowLoopTimeoutDrainsBeforeFailure` 与 `TestWorkflowIterationTimeoutDrainsBeforeNextRound` | Covered |
