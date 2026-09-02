---
status: implemented
---

# MCP 插件通过 Effect 自动物化到 Agent 项目配置

本决策继承 `0-effect-system-foundation.md` 的 Generic Target、完整 generation、Resource ownership、
journal 与 readiness 边界，并将 MCP 作为第二个内建 Effect kind 落地。它不建立 MCP 专用调度器，
也不让插件代码或 UI 成为密钥分发面。

## 决策

每个已安装且配置完整的 MCP 插件发布一个 immutable、secret-free `ora/mcp` Revision。Source revision
由插件版本与配置 revision 标识；Setting 实值只存在于 host-owned `store.json`。配置缺失、损坏、
插件卸载或 contribution 失效时，Source 退役，所有 Workspace 的完整 Desired generation 各推进一次。

OpenCode 和 Claude Agent 各声明两个 Resource，并以同一个 Target 协调：

| Agent | Skill Resource | MCP Resource | ownership sidecar |
|---|---|---|---|
| OpenCode | `.opencode/skills` | `.opencode/opencode.json` 的 `mcp` | `.opencode/.ora-mcp-managed.json` |
| Claude Code | `.claude/skills` | `.mcp.json` 的 `mcpServers` | `.claude/.ora-mcp-managed.json` |

MCP 文件是 shared JSON/JSONC Resource。Adapter 只增删 Ora sidecar 精确拥有的 server key，保留其他
顶层字段、注释和用户 server；OpenCode 的 `.opencode/opencode.jsonc` 具有更高优先级，存在同名 server
时整份 Resource 阻塞。sidecar 只保存稳定 identity、plugin id、server name、配置 revision 与语义
fingerprint，不保存 Setting 值。sidecar 缺失/不匹配、用户漂移、同名冲突或半完成写入都不得被接管；
不确定状态进入 blocking 或 RecoveryRequired。

Target projection 一次选择 Skill 与 MCP Desired，并按 binding 的 kind 精确路由到对应 Resource。
同一 generation 的所有安全 Resource mutation 完成后，Agent 只执行一次 reactivate；ready watermark
只有在完整 Target projection 被 Consumer 验证后推进。UI 以 `(Workspace, Agent plugin)` 查询该 Target，
发送门不得以“进程 running”代替 Effect readiness。

Secret 只在 Agent 子进程创建时由 host 注入。变量名使用确定性的 `ORA_MCP_*` 保留命名空间；插件
spawn 请求不得设置该前缀。Host 只有在 Agent identity、canonical Workspace、sidecar、项目配置
fingerprint、当前插件 contribution 与配置 revision 全部匹配时才解析 Setting 并注入。OpenCode 的
`{env:VAR}` 内容按 JSON string content 编码，Claude 的 `${VAR}` 使用 raw value。插件日志、Effect
Revision、Operation journal、sidecar、项目文件和 UI response DTO 均不得包含 Setting 实值。

## 兼容与升级

项目配置与 sidecar 的路径是硬兼容边界。首次升级不会接管既有同名 server；旧文件可以继续启动，
冲突只会阻塞相关 Target。配置文件写入采用原子替换，Operation 在副作用前持久化；配置与 sidecar
之间发生崩溃时依赖 journal 隔离恢复，而不是猜测 ownership。

OpenCode Agent 0.6.0 与 Claude Code Agent 0.2.0 是首个双 Resource Consumer。Claude 包以
`claude-agent-acp` 0.64.0 为验证基线但不设置运行时硬门槛，其 approval 行为按 adapter 的 Fixed
语义原样转发。

## 被拒绝的替代方案

- 不把 MCP 复制逻辑放入 Workspace 创建、Agent 启动或插件安装事件处理器；这些事件只唤醒
  level-triggered reconciliation。
- 不把 API key 写进 Agent 配置、sidecar、Revision 或插件进程环境。
- 不按 Skill/MCP 分别重启 Agent，也不允许不同 planner 对同一 Target 各自投影完整 Desired。
- 不重写整份 JSON/JSONC，不因 server 名称或内容相同自动取得 ownership。

## 核心测试依据

见 `specs/test-cases/desktop/core/effect/mcp-agent-materialization.md`。
