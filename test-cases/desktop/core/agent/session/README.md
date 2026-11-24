# Session 核心测试用例

本目录镜像 session 决策领域；会话根决策的验证义务见
[会话身份、绑定历史与统一诊断](identity-bindings-and-diagnostics.md)。

| 用例文件 | 风险面 |
| --- | --- |
| [identity-bindings-and-diagnostics.md](identity-bindings-and-diagnostics.md) | 切换与重复绑定、并发读取、历史降级、未创建状态、重试与循环归属、批量隔离、统一入口及暴露边界 |

子领域 MCP 的验证义务保留在 [mcp](mcp/) 下。

子领域 skill 镜像 [Skill 会话清单决策](../../../../../decisions/desktop/core/agent/session/skill/0-skill-catalog-from-storage-paths.md)
（`approved`，尚无实现证据，义务全部为 `Missing`）：

| 用例文件 | 风险面 |
| --- | --- |
| [skill/catalog-delivery.md](skill/catalog-delivery.md) | 首条全量、路径增量、记录丢失后重发、清单块位置、Workspace 不写入与发送不等待 |
| [skill/session-setup-and-hosts.md](skill/session-setup-and-hosts.md) | 可读根的能力协商、Node host 为空、旧 `effectResources` 兼容 |
| [skill/workflow-skill-resolution.md](skill/workflow-skill-resolution.md) | 工作流只冻结 skill ID 与调用名、派发时解析、缺失时的 Skill 依赖错误 |
| [skill/legacy-effect-cleanup.md](skill/legacy-effect-cleanup.md) | 升级时只删除与账本完全匹配的旧受管目录、幂等重做 |
