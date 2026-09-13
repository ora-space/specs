# Workflow 核心测试用例

本目录是工作流编排领域的 reviewer 风险导航图，与
[决策目录](../../../../decisions/desktop/core/workflow) 镜像。

架构边界和不变量来自
[节点运行时编排决策](../../../../decisions/desktop/core/workflow/0-node-runtime-orchestration.md)。

## 当前状态

根决策尚处于 `proposed`。按 ADR 生命周期，核心测试用例在该决策评审通过并标记为 `approved`
后，依据最终同意的不变量补充；在此之前本目录不登记任何用例。

预计首批核心用例将覆盖的风险面（以决策评审结论为准）：

- 快执行器在 per-run 排他锁内不执行 IO 或无界等待。
- 复合节点的区域内部节点不进入外层投影的 ready 集合。
- Loop 到达 `max_iterations` 上限时进入失败终态，不无限运行。
- 不含新节点类型的冻结图行为与重构前一致。
- 失效事件不携带工作流状态；事件丢失或乱序只导致过期视图，由重查收敛。
