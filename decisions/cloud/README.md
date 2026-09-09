# Cloud 决策目录

本目录承载 Ora Cloud / API Server 的架构决策。Cloud 是多租户业务状态的权威入口，负责身份与
成员授权、Project 和 Workspace 聚合、持久操作、执行准入以及对 Controller 的受限内部接口；
Controller 和 Node 不得绕过 Cloud 建立另一份业务权威状态。

每个叶子目录的第一个 ADR 使用 `0-...md` 作为根决策，后续决策按 `YYYYMMDD-...md` 追加。目录按
Cloud 的领域与协议边界划分，不预先决定 Go package、数据库表或部署进程的拆分方式。

| 目录 | 范围 |
|---|---|
| `api-boundary` | 公开与内部 HTTP API、OpenAPI、分页、错误契约和输入边界 |
| `identity-access` | 外部身份映射、服务与用户凭据、认证上下文和访问判定 |
| `tenancy` | Tenant、Membership、角色、管理员保护和 owner 隔离 |
| `project` | Project 聚合、Repository 意图、凭据引用和 Project 级资源状态 |
| `workspace` | Workspace、Task、Worktree 业务身份、归属和生命周期意图 |
| `operation` | 持久操作、幂等、版本、重试、状态转换和恢复证据 |
| `execution-admission` | Access、Execution Ticket、准入关闭、idle 证据和活动保护 |
| `controller-integration` | Controller lease、内部快照、effect plan、epoch fencing 和接管边界 |
| `persistence` | PostgreSQL schema、migration、事务顺序、约束和兼容性 |
| `lifecycle` | Cloud 启动、健康检查、优雅停止、配置、升级和部署边界 |

Controller 的调度与恢复策略放在 `specs/decisions/controller/`，Node 的 Git、文件、进程、Agent、
Plugin 和本地执行职责放在 `specs/decisions/node/`。跨层决策放在拥有业务不变量的一侧，并从其他
层引用，不复制成多份权威规则。
