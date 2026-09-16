---
status: proposed
---

# Node 管理已有主工作树并创建任务工作树

Node 负责其所在执行环境中的 Git worktree 操作。第一版要求目标环境已经存在一个可用的
`main worktree`；Controller 只持有任务和工作树意图、选择 Node、发起操作并持久化结果，不在
Controller 所在机器上直接执行 Git。该边界同时适用于本地 Node、SSH 远端 Node 和集群沙盒 Node。

本决策处于 `proposed` 状态，记录第一条 Node 垂直闭环的范围。它不决定 Node 的启动编排、
Controller–Node 消息编码、远程仓库 clone／fetch、跨 Node 迁移或 Workspace 文件同步。

在 Server 场景中，Worktree 状态遵循三层职责：API Server 是权威状态源，Controller 是 reconcile
worker，Node 是执行环境。API Server 持久化 Worktree 意图和状态，Controller 发起协调，Node 在
目标环境执行 Git 操作并上报事实。

## 背景与约束

当前单体 Backend 在任务创建、数据库记录和 Git worktree 操作之间直接调用。拆分后，任务可以
在远端 Node 执行；Controller 不能把自己的绝对路径、Git 环境或本地 checkout 假定为执行环境。

第一版需要一个足够小、可验证的闭环：Controller 能将创建或删除任务工作树的指令交给 Node，
Node 在自己的环境中执行并返回结果，Controller 再让 Client 查询到一致状态。目标环境已经准备好
主工作树，可以降低首个闭环对 clone、凭据和远程仓库同步的依赖。

## 决策

### D1：Node 是 Git worktree 的实际执行者

Controller 为一个已有 `main worktree` 的 Workspace 选择 Node，并发送创建或删除任务工作树的
指令。Node 在该 Workspace 绑定的仓库中执行 Git 操作，返回实际分支、路径、基准提交和状态。

Controller 不直接调用 `git worktree add`、`git worktree remove`，也不把 Controller 本地路径传给
Node。Node 必须在执行端解析并验证仓库和工作树路径。

### D2：第一版只支持已有 main worktree

一个可执行 Workspace 在目标 Node 上必须已经绑定一个可验证的 `main worktree`。第一版不负责：

- clone 或初始化仓库；
- 从远程仓库拉取或推送；
- 在 Node 之间迁移 Workspace；
- 把 main worktree 自动复制到另一台机器；
- 将 Controller 所在机器的 checkout 暂时当作远端 Node 的仓库。

如果 main worktree 不存在、不是有效 Git worktree、路径不在 Node 授权根目录内，Node 必须在产生
任务工作树副作用前拒绝操作，并返回结构化错误。

### D3：Controller 持有意图和状态，Node 持有事实和执行记录

Controller 持久化 `Workspace`、`Task`、`Worktree` 的业务身份、目标 `NodeId`、期望分支和生命周期
状态。Node 持久化本地执行记录和 worktree 事实，以便在响应丢失、连接重建或进程重启后按操作身份
对账。

Worktree 的实际路径是 Node-scoped 数据。Controller 可以保存它用于查询和后续请求，但 Client
不能把该路径当作本机路径；所有文件、Git 和 Agent 操作必须继续携带 Node 与 Workspace 上下文。

### D4：创建与删除必须幂等

创建和删除请求都携带稳定的 `operation_id` 与 `worktree_id`。Node 对重复请求执行以下规则：

| 情况 | 行为 |
|---|---|
| 已完成且参数一致 | 返回原结果，不重复执行 Git 操作 |
| 正在执行 | 返回当前状态或等待同一操作完成 |
| 相同身份但参数不同 | 拒绝并报告冲突 |
| 删除目标已不存在 | 返回 `AlreadyAbsent`，由 Controller 按幂等成功处理 |
| 执行结果不确定 | 保留恢复状态，不猜测成功，也不自动创建第二个工作树 |

Controller 在超时后使用原 `operation_id` 查询或重传，不能生成新的操作身份来绕过不确定状态。

### D5：最小协议闭环

第一版只需要覆盖以下消息语义，具体编码和 Transport 由 Controller–Node Protocol 决定：

```text
Controller → Node: EnsureWorktree
Node → Controller: WorktreeReady | WorktreeFailed

Controller → Node: RemoveWorktree
Node → Controller: WorktreeRemoved | WorktreeRemovalFailed
```

`EnsureWorktree` 至少引用 `operation_id`、`worktree_id`、`workspace_id`、目标 Node、仓库引用、
main worktree 绑定、base ref、期望分支和路径策略。成功结果至少包含 `node_id`、运行实例、实际
工作树路径、分支和 base commit。

### D6：两个 Workspace 及其 worktree 的边界

`main worktree` 是 Project 的 Main Workspace 在 Node 上已经存在的主 checkout。Main Workspace
本身是一个独立的 Workspace，具有自己的 `WorkspaceId` 和 `WorkspaceLocation`；它是任务工作树
创建的执行前置条件，不是第一版由 Ora 创建和回收的对象。当前数据模型只在 `workspaces` 与
`workspace_locations` 中表示 Main Workspace，不为它建立 `worktrees` 表记录。

`task worktree` 是 Task 所属的 Isolated Workspace 对应的、由 Ora 为该 Task 创建并拥有生命周
期记录的隔离 checkout。Main Workspace 与 Task Workspace 是两个独立的 Workspace；一个 Task
最多绑定一个 active task worktree。删除 Task 的清理流程必须通过 Node 删除 Task Workspace 对应
的 worktree 和由 Ora 创建的任务分支。main worktree 不属于 task worktree 的删除范围，即使任务
清理失败也不得删除或重置它。

## 不变量

1. Git worktree 的创建、删除、分支操作和仓库事实检查只在目标 Node 执行。
2. 目标 Node 在副作用前验证 main worktree、仓库归属、授权根目录和分支冲突。
3. Controller 不把本地绝对路径解释为远端 Node 的路径。
4. 一个 Task 最多有一个 active task worktree；main worktree 不被任务清理删除。
5. 相同 `operation_id` 和参数的重试不会产生第二个 worktree 或第二次删除副作用。
6. Node 断线、重启或响应丢失不会让 Controller 无证据地把 worktree 标记为已创建或已删除。
7. Node 上报的路径和 Git 状态必须带 `NodeId` 与 `NodeIncarnationId`，Controller 不能跨 Node 复用事实。
8. 本决策不要求 Node 能访问 Controller 的文件系统，也不要求 Controller 能直接访问 Node 的路径。

## 为什么不是其他方案

| 方案 | 否决理由 |
|---|---|
| Controller 继续在本地执行 Git，再把路径交给 Node | 远端 Node 无法使用该路径；执行事实和资源所有权仍停留在错误的机器上 |
| Controller 和 Node 各自维护一套 worktree 状态并互相覆盖 | 会产生两个写入者，断线和恢复时无法判断谁是权威 |
| 第一版同时实现 clone、同步和 worktree 创建 | 把仓库传输、凭据和大文件同步问题混入首个协议闭环，难以单独验证 Node 边界 |
| 每次重试都创建新的 worktree | 响应丢失时会产生重复 checkout、分支冲突和无法回收的外部副作用 |

## 落地顺序

1. 为 Node 定义 main worktree 绑定和仓库引用，完成执行端路径及 Git 事实验证。
2. 实现 `EnsureWorktree`，先支持本机 Node；用统一协议在 SSH Node 上复用同一语义。
3. 持久化 Worktree 结果、操作身份和失败状态，验证超时、重传、Node 重启和结果未知。
4. 实现 `RemoveWorktree` 和任务删除清理，验证 main worktree 始终不受影响。
5. 将现有 Backend 的本地 `GitTaskWorktreeProvisioner` 替换为 Controller–Node 适配器，删除重复的
   Controller 侧 Git 执行路径。

## 本决策未解决的问题

- main worktree 如何在 Node 注册时发现、绑定和更新；
- 一个 Node 上多个 Workspace 的仓库锁和并发策略；
- 分支命名、路径策略和已有同名分支的迁移规则；
- Worktree 事实、Controller 状态和 API Server 持久化之间的具体字段及事务协议；
- clone、fetch、push、跨 Node 迁移和文件同步。
