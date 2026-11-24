# Controller 核心测试用例

本目录与 `specs/decisions/controller/` 镜像。Controller 决策进入 `approved` 后，在对应子目录
登记需要长期维护的租户边界、持久化、队列、并发、恢复、协议和跨进程用例。

| 领域 | 核心用例 | 验证义务与证据状态 |
| --- | --- | --- |
| node-management | [不可用 Node 延后与 clone 分轮等待](node-management/20261028-defer-unavailable-node-operations.md) | ADR 为 implemented；实现基线 Cloud 1cfaed3、Desktop #750 合入版本 [df694c8](https://github.com/ora-space/desktop/commit/df694c8836fc70b1c789cc7e8e523d4d5b1b6c66)（最终 PR head 65fc457）。18 项按直接测试与部署实际覆盖分别标为 Covered/Partial/Missing；第 17 项要求不同 Project 各自的 Workspace Node 同时失联、旧项持续循环时健康 B/C 真实排队并推进。实现状态不消除缺口，调老时间的联合测试不替代真实 30 分钟等待，保留 blocked/node_unavailable 无界 retrySeconds 后续问题；当前 18 项为 7 Covered / 11 Partial / 0 Missing |
| session | [Controller 交付宿主 MCP](session/host-mcp-servers.md) | 依据 Node 目录的 [Controller 交付宿主 MCP 决策](../../decisions/node/agent-runtime/20261118-host-mcp-servers-in-agent-session-spec.md)（Node 侧 implemented）D1、D6：D1 文法与共用样例校验、按能力派发与启动准入、端点与凭据跨 Controller 重启有效或在提供服务前结束、先持久化再发送、凭据以持久状态判定有效期、attempt 级授权；第一个使用方是[完成工具](../cloud/workflow/host-completion-tool.md)，按 cloud `4ba9dc5a` 登记：12 项为 2 Covered / 8 Partial / 2 Missing |
| session | [Controller 发送前镜像与按能力派发 Session MCP 选择](session/session-mcp-selection.md) | 依据 Node 目录的 [显式 Session MCP 决策](../../decisions/node/agent-runtime/20261114-explicit-session-mcp-in-agent-session-spec.md)（Node 侧 implemented）D1、D4：发送前镜像有序规范身份、精确版本、16 项与 64 KiB 上限、`settings` 必填，以及只向声明 `session_mcp` 的 Node 派发非空选择；使用方尚未接入，两项为 Missing，Node 侧义务在 [node 目录](../node/agent-runtime/session-mcp.md)维护 |

