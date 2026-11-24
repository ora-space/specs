# Workflow 核心测试用例

本目录是工作流编排领域的 reviewer 风险导航图，与
[决策目录](../../../../decisions/desktop/core/workflow) 镜像。

架构边界和不变量来自
[节点运行时编排决策](../../../../decisions/desktop/core/workflow/0-node-runtime-orchestration.md)；首个
复合运行时（迭代节点）的落地承诺见
[迭代节点决策](../../../../decisions/desktop/core/workflow/20260915-iteration-composite-runtime.md)。
while 的独立轮池、上限两策略和早退见[while-Loop 决策](../../../../decisions/desktop/core/workflow/20261013-while-loop-with-durable-round-scopes.md)。
Agent 长任务的冻结配置及会话隔离见[静默策略决策](../../../../decisions/desktop/core/workflow/20261008-prompt-inactivity-policy.md)，当前计时、续期和到期规则见
[工作流有限与无限等待决策](../../../../decisions/desktop/core/workflow/20261123-workflow-agent-wait-limit.md)。
交互式节点的完成门禁、park 快照与 force 见
[交互式节点完成契约](../../../../decisions/desktop/core/workflow/20261021-interactive-node-completion.md)；
节点产出物声明（`completion.files`）见
[交互式节点产出物声明](../../../../decisions/desktop/core/workflow/20261023-interactive-deliverable-contract.md)。

自动重试后的交付窗口与后继调度见[最终尝试产出决策](../../../../decisions/desktop/core/workflow/20261118-interactive-retry-output.md)。

## 阅读与维护规则

- 核心用例按业务风险组织，不按单元测试、集成测试或 crate 组织。
- 一个核心用例可以由纯状态测试、repository 测试和跨 adapter 测试共同证明。
- 只有在测试失败会直接说明某项验证义务不成立时，才把该测试列为代表性证据。
- 文件路径与用例标题共同构成稳定引用；代表性测试可以在 doc comment 中引用它们。

## 当前状态

节点运行时重构、区域机制与第一个 foreach 迭代复合运行时均已实现并验证：引擎核心经运行时注册表
分发，轮次与账本由持久化事实重算，编辑器通过显式复合区域入口维护成员图，运行视图按轮投影。
while-Loop 已实现独立轮池、上限两策略与早退清理；普通执行轮在重启时失败，显式 resume 重开第一轮。
证据状态按各文件中的验证义务持续维护。

| 文件 | 覆盖的风险面 |
| --- | --- |
| [engine-parity.md](engine-parity.md) | 平图兼容；注册表分发及核心扫描范围的静态约束 |
| [region-and-loop.md](region-and-loop.md) | 区域黑盒与结构校验；DAG 不变量；轮次持久化；while 默认上限失败 |
| [while-loop.md](while-loop.md) | 独立轮池、同时 feedback、上限成功/失败、早退清理、恢复和显式 resume |
| [iteration-node.md](iteration-node.md) | 迭代节点启动边界、轮次事务性、错误策略与恢复；显式区域编辑、容器几何、级联删除与 Desktop 真实执行 |
| [stateless-recovery-and-hitl.md](stateless-recovery-and-hitl.md) | foreach 启动恢复；区域 HITL 的拒绝边界与证据缺口 |
| [runtime-forms.md](runtime-forms.md) | 快执行器锁内无 IO；session 归属；回报幂等 |
| [variable-pool-boundary.md](variable-pool-boundary.md) | 一切节点形态经声明 owner 与类型校验写入 |
| [variable-aggregator.md](variable-aggregator.md) | 候选声明门禁、按声明顺序透传首个已赋值值、空值与无匹配失败、分支合流 |
| [invalidation-events.md](invalidation-events.md) | 失效事件不携带工作流状态，重查收敛 |
| [prompt-inactivity.md](prompt-inactivity.md) | 显式无限等待、冻结策略与展示、可归属进展续期、到期收敛与节点结果 |
| [interactive-node-completion.md](interactive-node-completion.md) | 交互式节点完成门禁、park 快照与 turn key、force 审计、cleanup 修复与区域边界 |
| [interactive-retry-output.md](interactive-retry-output.md) | 自动重试的最终输出窗口、重复完成与自动接受门禁、原会话恢复及 force 审计 |
| [interactive-deliverable-contract.md](interactive-deliverable-contract.md) | 产出物声明的文件判定、路径与 glob 边界、基线缺失降级、不支持字段与版本拒绝 |
| [first-turn-confirmation.md](first-turn-confirmation.md) | 首轮自动回合显式确认、异常停靠与等待参与分离、普通确认与 force 审计、旧快照兼容 |

根决策“后果与后续边界”中的预留项（并行迭代、嵌套循环、Code 沙箱、远程子工作流细节）
不登记核心用例：它们尚未构成系统承诺，待各自后续 ADR 批准后随其义务登记。节点级错误策略的
复合区域作用域最小切片（`FailurePropagation`）已随迭代节点决策落地，其用例在
`iteration-node.md`；向任意节点的泛化仍属预留。
