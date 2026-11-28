# Node 决策目录

本目录承载 Execution Plane / Node 的架构决策。子目录按 Node 能力或协议边界划分；每个叶子目录
的第一个 ADR 使用 `0-...md` 作为根决策，后续决策按日期追加。

| 目录 | 范围 |
|---|---|
| [repository](repository/0-clone-selected-repository-branch.md) | 当前最小闭环：clone 指定仓库的指定分支；范围根决策与[最小执行契约](repository/20260918-minimal-clone-execution-contract.md)均 approved，尚未实现 |
| `worktree` | main worktree、task worktree 及 Git worktree 生命周期 |
| `agent-runtime` | Agent 进程、ACP 转发、会话连接和运行恢复；[显式 Session MCP](agent-runtime/20261114-explicit-session-mcp-in-agent-session-spec.md)（Node 侧 implemented，Controller 义务随使用方接入）；[同一 checkout 的多个会话](agent-runtime/20261115-shared-checkout-agent-sessions.md)（implemented，desktop#774 与 #864；v2 按能力启用同 run 多会话由 v2 事项承接）；[会话 MCP 刷新与宿主健康](agent-runtime/20261116-session-mcp-refresh-and-host-health.md)（Node 侧 implemented，消费端与证据边界见正文）；[Controller 交付宿主 MCP](agent-runtime/20261118-host-mcp-servers-in-agent-session-spec.md)（Node 侧 implemented，Controller 义务随使用方接入）；[CodeAgent _subagent/update 子代理进度通道](agent-runtime/20261118-codeagent-subagent-update-channel.md)（implemented）；[会话尝试的进程带环境标记](agent-runtime/20261120-attempt-mark-in-session-stop-evidence.md)（implemented，desktop#802；修改会话 MCP 刷新 D5/D6/D8 的停止证据与取消决策 D3） |
| `plugin-runtime` | Plugin 安装、启动、停止、运行和本地状态 |
| `effect` | ResourceAdapter、ConsumerAdapter、观测、mutation 和 receipt |
| `filesystem` | Node 文件系统访问、路径边界和 Workspace 文件操作 |
| [git-execution](git-execution/0-node-runs-git-directly-with-attempt-locks.md) | Node 直接执行自己的 Git：进程组、尝试锁给出的停止证据、重启收口；取代 process 下的 host／guardian 决策；implemented（Linux） |
| [process](process/README.md) | host／guardian 进程运行宿主的决策已被 git-execution 取代，作为历史记录保留；containment 继续定义保证等级和强纳管方向 |
| `protocol` | Node Protocol 消息、握手、事件、重放和 Transport 边界 |
| `revision` | 运行成果的内部快照、增量 bundle 与对象存储上传 |
| `lifecycle` | Node 启动、注册、Ready、Draining、停止和恢复 |

目录是职责导航，不预先决定 crate 名称或进程拆分。跨能力的决策放在最小共同边界上；只有确实
拥有 Node 特有不变量的内容才放入对应子目录。

## 已批准的跨仓库依赖

[运行时控制同步交付决策](../cloud/controller-integration/20260927-fenced-runtime-control-delivery.md)为 `implemented`，要求 Cloud 执行入口以持久控制绑定约束迟到指令、交互输入和恢复，并验证管理程序与工作负载的权限隔离。Node 单连接、心跳及宿主代次不是用户独占证明；本决策不自动恢复进程控制领域此前延期的完整信任体系。完整联合故障矩阵仍有缺口，直接证据见[跨进程核心用例](../../test-cases/cloud/controller-integration/fenced-runtime-control-delivery.md)。

[交付执行结算与未受理证明](../cloud/issue-run/20261026-delivery-execution-settlement-and-non-acceptance.md)
为 `approved`：它扩充 Node 原 execution 的持久受理 ledger、输入围栏和证明能力，以及 schema
升级与能力协商边界；[恢复停止确认的 `NotStarted` 证据](revision/20261021-stop-confirmation-accepts-not-started.md)
只针对特定 ProcessRun，不能替代原 execution 的未受理证明。
