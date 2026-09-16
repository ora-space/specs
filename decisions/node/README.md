# Node 决策目录

本目录承载 Execution Plane / Node 的架构决策。子目录按 Node 能力或协议边界划分；每个叶子目录
的第一个 ADR 使用 `0-...md` 作为根决策，后续决策按日期追加。

| 目录 | 范围 |
|---|---|
| `worktree` | main worktree、task worktree 及 Git worktree 生命周期 |
| `agent-runtime` | Agent 进程、ACP 转发、会话连接和运行恢复 |
| `plugin-runtime` | Plugin 安装、启动、停止、运行和本地状态 |
| `effect` | ResourceAdapter、ConsumerAdapter、观测、mutation 和 receipt |
| `filesystem` | Node 文件系统访问、路径边界和 Workspace 文件操作 |
| [process](process/README.md) | 进程运行宿主、控制权、生命周期、纳管、I/O 与恢复；当前根决策不包含运行配额 |
| `protocol` | Node Protocol 消息、握手、事件、重放和 Transport 边界 |
| `lifecycle` | Node 启动、注册、Ready、Draining、停止和恢复 |

目录是职责导航，不预先决定 crate 名称或进程拆分。跨能力的决策放在最小共同边界上；只有确实
拥有 Node 特有不变量的内容才放入对应子目录。
