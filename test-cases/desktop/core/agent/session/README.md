# Session 核心测试用例

本目录镜像 session 决策领域；会话根决策的验证义务见
[会话身份、绑定历史与统一诊断](identity-bindings-and-diagnostics.md)。

| 用例文件 | 风险面 |
| --- | --- |
| [identity-bindings-and-diagnostics.md](identity-bindings-and-diagnostics.md) | 切换与重复绑定、并发读取、历史降级、未创建状态、重试与循环归属、批量隔离、统一入口及暴露边界 |

[会话选项决策](../../../../../decisions/desktop/core/agent/session/20261125-session-options-single-source.md)（`implemented`，代表性证据固定于 desktop `d35f4d1a`）的验证义务：

| 用例文件 | 风险面 |
| --- | --- |
| [options-source-and-order.md](options-source-and-order.md) | 答复与推送的先后、建立和恢复时的旧推送、重建后的旧绑定、设置的返回、在途设置的收尾 |
| [options-delivery.md](options-delivery.md) | 流开头的选项、广播与查询、取回与流的先后、失去 provider、发起设置的视图 |

子领域 MCP 的验证义务保留在 [mcp](mcp/) 下。

子领域 skill 镜像 [Skill 会话清单决策](../../../../../decisions/desktop/core/agent/session/skill/0-skill-catalog-from-storage-paths.md)
（`implemented`，代表性证据固定于 `desktop#827`）：

| 用例文件 | 风险面 |
| --- | --- |
| [skill/catalog-delivery.md](skill/catalog-delivery.md) | 首条全量、读取失败保护、路径增量、记录丢失后重发、清单块位置、Workspace 不写入与发送不等待 |
| [skill/session-setup-and-hosts.md](skill/session-setup-and-hosts.md) | 可读根的能力协商、Node host 为空、旧 `effectResources` 兼容 |
| [skill/workflow-skill-resolution.md](skill/workflow-skill-resolution.md) | 工作流只冻结 skill ID 与调用名、同名裸名 `workflow_skill_ambiguous` 拒绝、派发时精确 ID 解析、缺失时的 Skill 依赖错误 |
| [skill/legacy-effect-cleanup.md](skill/legacy-effect-cleanup.md) | 升级时只删除与账本完全匹配的旧受管目录、pre-0025 原子备份与不可逆降级拒绝 |
