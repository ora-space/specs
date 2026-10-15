# Node 决策目录

本目录承载 Execution Plane / Node 的架构决策。子目录按 Node 能力或协议边界划分；每个叶子目录
的第一个 ADR 使用 `0-...md` 作为根决策，后续决策按日期追加。

| 目录 | 范围 |
|---|---|
| [repository](repository/0-clone-selected-repository-branch.md) | 当前最小闭环：clone 指定仓库的指定分支；范围根决策与[最小执行契约](repository/20260918-minimal-clone-execution-contract.md)均 approved。凭据来源由[凭据来源修订](repository/20261010-minimal-clone-execution-contract-credential-source-amendment.md)（proposed）扩展，执行侧隔离由[受控 Node 的仓库凭据执行使用独立 OS 身份](repository/20261010-controlled-node-git-authentication-isolation.md)（proposed）规定，其执行器归属由[凭据执行器归属当前正式产品](repository/20261012-credential-executor-belongs-to-the-current-product.md)（proposed）调整为当前正式产品的独立部署单元。执行器、凭据物化、部署固定的 helper 与 `broker-resolver` 客户端已实现，真实私有仓库 Clone 全链路有证据；[checkout 交付到 Workspace 的通道](repository/20261012-checkout-delivery-into-the-workspace.md)仍为 **proposed**，交付未实现，`repository_credential_executor_unavailable` 继续拒绝 |
| `worktree` | main worktree、task worktree 及 Git worktree 生命周期 |
| `agent-runtime` | Agent 进程、ACP 转发、会话连接和运行恢复 |
| `plugin-runtime` | Plugin 安装、启动、停止、运行和本地状态 |
| `effect` | ResourceAdapter、ConsumerAdapter、观测、mutation 和 receipt |
| `filesystem` | Node 文件系统访问、路径边界和 Workspace 文件操作 |
| [process](process/README.md) | 进程运行宿主、控制权、生命周期、纳管、I/O 与恢复；当前根决策不包含运行配额 |
| `protocol` | Node Protocol 消息、握手、事件、重放和 Transport 边界 |
| `revision` | 运行成果的内部快照、增量 bundle 与对象存储上传 |
| `lifecycle` | Node 启动、注册、Ready、Draining、停止和恢复 |

目录是职责导航，不预先决定 crate 名称或进程拆分。跨能力的决策放在最小共同边界上；只有确实
拥有 Node 特有不变量的内容才放入对应子目录。

## 已批准的跨仓库依赖

[运行时控制同步交付决策](../cloud/controller-integration/20260927-fenced-runtime-control-delivery.md)为 `implemented`，要求 Cloud 执行入口以持久控制绑定约束迟到指令、交互输入和恢复，并验证管理程序与工作负载的权限隔离。Node 单连接、心跳及宿主代次不是用户独占证明；本决策不自动恢复进程控制领域此前延期的完整信任体系。完整联合故障矩阵仍有缺口，直接证据见[跨进程核心用例](../../test-cases/cloud/controller-integration/fenced-runtime-control-delivery.md)。
