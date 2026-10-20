---
status: approved
---

# Node 只从 Git Server 快照 clone 并校验 `baseCommit`，checkout 不配置 origin；Agent 经沙盒内固定动词的 `ora-git` 请求 Node 代为 fetch 与推送 Workspace 分支

Node 的 clone 不再接受外部仓库地址，也不再使用部署凭据。clone 输入是
[Git Server 代码中心决策](../../cloud/git-server/20261016-project-repository-as-code-hub-and-ref-contract.md)（下称“契约 ADR”）D5 第
1 项的快照输入 `{repository_url, snapshot_ref, base_commit, workspace_ref}`。Node 沿用
[Node 交付推送与 Git 恢复决策](../revision/20261017-push-delivery-ref-and-restore-from-git-server.md)（下称“交付推送决策”）D1 的私有传输仓库：
以 Node 身份、带 git-access 证书把 `snapshot_ref` 拉进传输仓库，核对它指向 `base_commit`，再以工作负载身份从传输仓库把
对象取入新的 checkout，检出名为 Workspace 分支短名的本地分支。checkout 不配置任何 remote，也不留下任何与证书有关的配置；
成功前校验 HEAD 等于 `base_commit`。会话期间，Agent 用沙盒里的 `ora-git` 命令发起 fetch 与推送：命令只有固定的几个动词，
经本地 Unix socket 请求 Node 执行，Node 按 socket 所属的会话判定身份，用该会话执行的 ID 访问 Git Server，私钥不离开 Node。
推送把 checkout 的提交当数据搬进传输仓库，带声明的旧值推到 `refs/heads/workspaces/<workspaceId>/<branch>`；fetch 把 upstream
与其他 Workspace 的分支取到 checkout 的 `refs/remotes/ora/...`。

本 ADR 已经 owner 批准，处于 `approved`（核心测试用例见[内部线用例](../../../test-cases/node/repository/snapshot-clone-and-ora-git.md)；实现合入、端到端验收通过后改为 `implemented`），继承 [clone 根决策](0-clone-selected-repository-branch.md)、
[最小 clone 执行契约](20260918-minimal-clone-execution-contract.md)（下称“clone 契约”）、
[Git 副作用移出受理 worker](20260925-clone-git-effects-run-outside-the-admission-worker.md) 与
[中断失败判定](20260925-terminated-clone-attempt-fails-as-interrupted.md)，承担契约 ADR 中“Node 怎样 clone、fetch、推送 Workspace
分支”。Cloud 一侧的输入与授权见 [clone 步骤消费快照决策](../../cloud/operation/20261016-clone-step-consumes-binding-snapshot.md) 与
[快照 clone 输入与执行授权决策](../../cloud/controller-integration/20261016-snapshot-clone-input-and-execution-ref-grants.md)。涉及
desktop 的 Node 协议（`crates/node-protocol`）、Node 的 clone 执行与核实（`apps/ora-node/src/repository/`、
`crates/gitlancer/src/git/branch_clone.rs`）、git-access 配置（`apps/ora-node/src/git_access.rs`）、新的 `ora-git` 命令与 Node
的本地 socket 服务；cluster 的 Sandbox Server（git-access 身份签发）、Compose、Node 镜像、echo agent fixture 与验收脚本。内部线同时按
[交付推送决策](../revision/20261017-push-delivery-ref-and-restore-from-git-server.md) 实现交付推送（含 Workspace 分支的快进）与续作恢复，删除 bundle 的代码与协议字段；
这两部分的规则都由交付推送决策承担，本文只给出它们用到的 clone 记录与账本。

兼容性边界：**破坏性切换**（契约 ADR 范围边界）。Node 协议与 Cloud、Controller 同时发布，旧形状的命令被拒绝；不迁移 Node
数据库中的旧 clone 记录与已有的 checkout，开发环境的 Workspace 在切换后重建。

## 继承与修改

| 前序约定 | 本文处理 |
| --- | --- |
| 根 D1、不变量 1：仓库与分支是显式输入，不静默用另一个仓库或分支替代 | 继承；输入改为快照输入，Node 从不以别的 ref 或提交替代（D1、D3） |
| 根 D2：完成是业务事实；根 D3：Worktree 不是 Node 的能力 | 继承，不变 |
| clone 契约 D1：输入为 HTTPS 或 `ssh://` 地址与分支名 | 修改：只接受 Git Server 的 `https` 仓库地址与快照输入四个字段；不再接受外部地址与 SSH（D1） |
| clone 契约 D2：凭据由 Node 部署提供（HTTPS 凭据来源、SSH 身份与主机校验） | 修改：clone 不使用任何部署凭据；只用 git-access 证书，且只在 Node 身份的传输仓库中使用（D2） |
| clone 契约 D3：Node 在仓库根下分配并独占目标目录 | 继承，不变 |
| clone 契约 D4：`git clone --single-branch --branch <branch> --origin origin` 获取完整单分支历史；禁 hooks、不递归 submodule、抑制 LFS | 修改：改为传输仓库拉取快照、工作负载身份建立 checkout；不配置 remote；完整历史、禁 hooks、submodule 与 LFS 规则不变（D2） |
| clone 契约 D5：结果是实际获取的提交，不承诺接受时的远端快照；需要锁定提交时另行引入以提交为输入的能力 | 修改：这就是那项能力。输入锁定 `base_commit`，结果必须等于它，不等即失败（D3） |
| clone 契约 D6：先持久关联再派发；未知不自动重跑或删除 | 继承；传输仓库的只读拉取例外，见 D4 |
| clone 契约 D7：clone 是独立能力与消息 | 继承；消息形状见 D10 |
| clone 契约 D8：网络 Git 使用 `network_timeout_ms` | 继承，并用于传输仓库的拉取、`ora-git` 的 fetch 与推送 |
| Git 副作用移出 worker D1–D5；中断失败判定 D1–D4 | 继承，不变；它们同样适用于 clone 中工作负载身份的命令（D4） |
| 交付推送决策 D1：带证书的 Git 访问以 Node 身份在私有传输仓库中执行，不读取 checkout 的配置、钩子或 ref | 继承，并扩展到 clone、`ora-git` 的 fetch 与推送（D2、D6、D7） |
| 交付推送决策 D2、D3：交付把最终提交推到交付 ref，再带账本中的旧值快进 `workspace_ref`，冻结计划后重放 | 继承，由内部线实现；本文提供 `workspace_ref` 的来源（clone 记录）与共用的账本（D6、D8） |
| 交付推送决策 D5：恢复把 Workspace 本地分支移到前序最终提交，改写提示以本 checkout 的 `base_commit` 判定，不改 `refs/remotes/ora/**` 与配置 | 继承，由内部线实现；本文提供分支短名与 `base_commit`（D9） |
| 交付推送决策 D7：Cloud 模式必须配置 `git_access` | 继承；clone 也需要它，Node 只在配置了 `git_access` 时声明 clone 能力（D10） |
| [Git Server 根决策](../../cloud/git-server/0-project-code-hub-git-server-over-https.md) D5：沙盒里由 Node 进程执行 Git，证书与上下文头只在当次命令中传入，不写入 checkout 的 `.git/config`、全局配置、日志或 journal | 继承，不变；本文另规定 checkout 不配置任何 remote（D2） |
| Git Server 根决策 D5：Agent 在会话中自己执行 `git push` 没有凭据，会失败；平台管理的写入只经持证组件进入 Git Server | 继承：Agent 仍然没有凭据，它的 Workspace 分支经 `ora-git` 请求 Node 这一持证组件推送（D5、D6） |
| Git Server 根决策 D7：Workspace 的代码输入只来自 Git Server，Node 不为平台管理的代码输入访问外部仓库 | 继承；本文是它在 Node 一侧的落实：clone 只接受 Git Server 的快照输入（D1、D2） |
| [Node 协议决策](../protocol/20260928-streamed-thread-events-and-session-commands.md)：会话执行的输入与事件 | 继承；会话输入增加 Workspace 目录，其余 Workspace 信息从 checkout 的 clone 记录取得（D10） |
| Git Server 根决策 D2：沙盒持 Sandbox Server 按租户、Workspace、代次另签的 git-access 证书 | 继承，由内部线实现（落地顺序第 1 步）；未合并的已有实现只作参考 |

## 决策

### D1：clone 输入与校验

`CloneExecutionSpec` 为 `{node_id, repository_url, snapshot_ref, base_commit, workspace_ref}`，四个业务字段由 Cloud 固定、
Controller 原样转交。Node 受理时校验：

| 字段 | 规则 |
| --- | --- |
| `repository_url` | `https`；没有用户信息、查询与片段；沿用现有地址校验中拒绝空白、控制字符与反斜杠的规则。不接受 `ssh://` |
| `snapshot_ref` | `refs/ora/upstreams/<bindingId>/snapshots/<oid>`，`<bindingId>` 是规范 UUID，`<oid>` 等于 `base_commit` |
| `base_commit` | 完整提交 ID（40 或 64 位小写十六进制） |
| `workspace_ref` | `refs/heads/workspaces/<workspaceId>/<branch>`，`<workspaceId>` 是规范 UUID，`<branch>` 通过现有 `GitBranchName` 校验且不是 `HEAD` |

- 同一执行身份必须匹配原输入与目标 Node，同身份不同输入冲突（clone 契约 D1，不变）。
- `<branch>` 是 Workspace 分支短名，也是 checkout 中本地分支的名字；`<workspaceId>` 与 `<bindingId>` 记入 clone 记录，
  供 `ora-git` 使用（D5）。

为什么只接受 `https`、不接受 SSH：Git Server 只提供 smart HTTP（Git Server 根决策 D1）；输入里的地址只可能指向它。
保留 SSH 等于保留一条 Node 访问外部仓库的路径，违背契约 ADR 需求 3。

### D2：经私有传输仓库 clone，checkout 不配置 origin

一次 clone 分两段，身份不同：

1. **拉取快照（Node 身份）。** Node 按交付推送决策 D1 建立该执行私有的裸传输仓库
   （`<repository_root>/.ora-transport/clone-<sha256(execution)>`，`0700`，属主 Node），以同一组调用级配置运行
   `git fetch --no-tags <repository_url> <snapshot_ref>:refs/ora/input`：证书、私钥、CA 与 `Ora-Execution-Id: <clone 执行 ID>`
   只在命令行上传入，禁用 credential helper、重定向、系统与全局配置、终端提示，只允许 `https`。然后确认 `refs/ora/input`
   是提交且等于 `base_commit`。
2. **建立 checkout（工作负载身份）。** Node 把传输仓库交给工作负载用户（其中没有任何凭据），在已预留的目标目录中以
   工作负载身份、沿用 clone 契约的受管执行与加固配置（`core.hooksPath=/dev/null`、LFS 抑制、不递归 submodule、不读模板），
   依次执行：`git init --template=`；从传输仓库路径取入 `refs/ora/input`，建立本地分支 `refs/heads/<branch>` 指向
   `base_commit`；让 HEAD 指向这个分支并检出工作区。只有取入这一条命令放开 `file` 协议（与恢复时的取入相同）。

完成后（无论成败）删除传输仓库；Node 启动时按交付推送决策 D1 清理残留，只保留工作负载段尚未完成本地取入的 clone 传输仓库
（这一步之前不会再拉取快照，它的 Run 也可能仍在 guardian 下读取传输仓库）。

checkout 的结果：

- 没有任何 remote（没有 `origin`，也没有指向传输仓库的 remote），没有 `remote.*`、`http.*`、`credential.*` 配置；
- 有完整的、非浅的历史；没有 `objects/info/alternates`，对象全部在 checkout 自己的对象库里；
- 当前分支是 `<branch>`，指向 `base_commit`，工作区干净；
- 没有 `refs/remotes/**`。之后的 `refs/remotes/ora/**` 只由 `ora-git fetch` 与推送写入（D6、D7）。

为什么不让工作负载身份直接 clone：证书私钥只有 Node 身份可读（Git Server 根决策 D5），以工作负载身份访问 Git Server
就要把私钥交给工作负载。为什么不让 Node 身份直接 clone 出 checkout：checkout 属于工作负载，之后 Agent 会改写它的配置；
Node 身份的 Git 进程从一开始就不进入 checkout，规则最简单，也和交付、恢复一致。

为什么不配置 origin：checkout 里没有任何凭据，origin 指向 Git Server 也无法访问；指向外部仓库则会诱导 Agent 或工具绕过
平台直接访问外部仓库。没有 remote 时，所有远端交互只能经 `ora-git`，失败的原因也明确。

### D3：成功的核实与失败分类

成功前，在收尾确认之后（中断失败判定不变量 1），Node 以只读命令核实：

- 目录身份与预留一致，是非 bare、非浅的仓库，`objects/info/alternates` 与 `commondir` 不存在（现有规则）；
- `git remote` 输出为空；
- `refs/heads/<branch>` 存在，`HEAD` 是指向它的符号引用；
- `HEAD^{commit}` 等于 `base_commit`；工作区相对 HEAD 没有已跟踪文件的改动。

成功结果 `CloneReady` 的 `commit` 等于 `base_commit`。替换现有核实中对 `remote.origin.url` 与 `refs/remotes/origin/<branch>` 的检查：
它们在新的 checkout 中都不存在。

| 情况 | 失败码 |
| --- | --- |
| Git Server 返回 `401`、`403`（无权与不存在给出相同应答），或报告没有 `snapshot_ref` | `SnapshotUnavailable` |
| 拉取到的 `refs/ora/input` 不是提交，或不等于 `base_commit` | `BaseCommitMismatch` |
| `5xx`、`429`、连接、DNS、TLS、低速中止或时限到期，重试耗尽 | `SourceUnavailable` |
| 工作负载段的命令以非 0 退出 | `OperationFailed` |
| 任一受管命令被终止且收尾已确认 | `Interrupted` |
| 目录冲突 | `DestinationConflict` |

- `BranchNotFound` 不再产生：快照 ref 缺失归入 `SnapshotUnavailable`。
- 核实通过但 HEAD 不等于 `base_commit`（工作负载段被篡改或缺陷）：保留 Unknown 与目录，不报告成功。
- 为什么 `BaseCommitMismatch` 单独成码：它说明 Git Server 上的快照 ref 与 Cloud 登记的不一致，正常情况下不会发生
  （快照 ref 只能新建，契约 ADR D4）；与网络失败混在一起，会掩盖数据不一致。

### D4：持久阶段与恢复

clone 的持久阶段沿用 Git 副作用移出 worker 的步骤机：`Reserved → DirectoryCreated → Dispatched → 终态`，在
`DirectoryCreated` 与 `Dispatched` 之间增加 Node 身份的拉取：

| 阶段 | 内容 | 恢复 |
| --- | --- | --- |
| `DirectoryCreated` 之后、`Dispatched` 之前 | 拉取快照到传输仓库并核对（D2 第 1 段）；每次拉取尝试开始前先持久记录累计次数，同一段内的重试也计入 | 重启后丢弃传输仓库，重新拉取；累计次数到 3 次后以 `SourceUnavailable` 结束 |
| `Dispatched` | 工作负载段的固定命令序列（D2 第 2 段），每条命令先持久关联再派发 | 按中断失败判定：已退出、已终止、未核实分别处理；不在目标目录中重跑 |

工作负载段持久记录当前步骤：`init`、本地取入、检出各有一个独立 Run；前一步成功且清理完成后才能记录下一步。
同一步不能新建第二个 Run，重启按当前步骤的原 Run 核对，不在目标目录中重跑。

- 为什么拉取可以在重启后重做：它只写 Node 私有的传输仓库，不碰目标目录，也不改变 Git Server（只读）。clone 契约 D6
  禁止重跑，是为了不在已有残留中再次写入目标目录；这一段没有这样的残留。次数上限防止反复重启时无限拉取。
- 拉取对 `5xx` 与网络错误重试，间隔 5 秒、30 秒，与交付推送决策 D4 的分类相同；`401`、`403` 与 ref 不存在不重试。每次拉取
  受 `network_timeout_ms` 约束。
- **段内重试与重启后的重新拉取共用同一个累计次数，上限 3 次**：一次拉取无论是因为重试还是因为重启而发起，都先把累计次数
  加一并持久化，再开始拉取；到 3 次仍失败就以 `SourceUnavailable` 结束，最坏一共拉取 3 次，而不是每次重启各重试 3 次。
  重启后的下一次拉取不再等待剩余的退避间隔。为什么合并成一个计数：两个计数相乘会让最坏情况变成 9 次拉取，Node 反复重启
  时 clone 迟迟不失败；一个持久计数也让“已经试了几次”只有一个事实来源。
- 拉取失败时目标目录为空，残留按现有规则保留并报告。

### D5：`ora-git`：Agent 发起 Git 远端操作的唯一入口

Node 镜像提供 `ora-git` 命令。它不执行 Git 网络操作，只把请求经本地 Unix socket 交给 Node，由 Node 执行并返回结果。

**动词**（固定集合，不接受任何选项、refspec、URL 或透传给 Git 的参数）：

| 命令 | 访问 Git Server | 作用 |
| --- | --- | --- |
| `ora-git info` | 否 | 打印本 Workspace 的 `workspaceId`、分支短名、`base_commit` 与可用的远端跟踪 ref 前缀 |
| `ora-git list` | 读 | 列出 upstream 跟踪分支与各 Workspace 的分支（ref 名与提交，来自 Git Server 的 ref 广告），并按会话输入中的 Workspace 目录附上 Workspace 类型、Issue 标题、Issue 与 run 的 ID 和 run 状态（[快照 clone 输入与执行授权决策](../../cloud/controller-integration/20261016-snapshot-clone-input-and-execution-ref-grants.md) D7）；目录里没有的 Workspace 只显示 ID，并注明目录是会话开始时的快照 |
| `ora-git fetch upstream` | 读 | 取 `refs/ora/upstreams/<bindingId>/heads/*` 到 `refs/remotes/ora/upstream/*`（D7） |
| `ora-git fetch workspace <workspaceId> [<branch>]` | 读 | 取 `refs/heads/workspaces/<workspaceId>/<branch>`（省略时为该 Workspace 的全部分支）到 `refs/remotes/ora/workspaces/<workspaceId>/<branch>`（D7） |
| `ora-git push [<branch>]` | 写 | 把本地 `refs/heads/<branch>`（省略时为 Workspace 分支短名）推到 `refs/heads/workspaces/<自己的 workspaceId>/<branch>`（D6） |

**参数限制**：

- `<workspaceId>` 必须是规范的小写 UUID；可以是自己的 Workspace。
- `<branch>` 必须通过 `GitBranchName` 校验，不是 `HEAD`，不以 `-` 开头，不超过 200 字节；不接受通配符、`:`、`+`。
- 多余的参数、未知动词、未知选项一律是 `usage` 错误。`<bindingId>` 不由 Agent 给出：Node 取自本 checkout 的 clone 记录。
- 推送的目标永远在自己的 Workspace 前缀下：命令无法表达写别的 Workspace 的分支。

**socket 与身份判定**：

1. Agent 会话执行进入运行时，Node 为它创建一个监听 socket（`/run/ora-git/<随机名>.sock`，随机名含 128 位随机数），并把路径
   放进 Agent 插件进程的环境变量 `ORA_GIT_SOCKET`，Agent 启动的子进程随之继承。`ora-git` 只连接这个路径。
   - 目录 `/run/ora-git` 属主 Node、权限 `0711`：任何本地 UID 都能按完整路径进入，但不能列出目录，拿不到别的会话的 socket 名。
   - socket 文件属主 Node、权限 `0666`：文件系统层不挡连接，是否接受只由第 3 条的对端 UID 检查决定。
2. **会话由 socket 决定，不由请求声明。** 请求中没有执行 ID、Workspace ID 或仓库地址；Node 把从某个 socket 收到的请求
   归属于创建该 socket 的会话执行。
3. Node 读取连接对端的凭据（`SO_PEERCRED`），对端 UID 必须是工作负载 UID，否则以 `forbidden` 拒绝。
4. 每个请求在访问 Git Server 前，Node 确认该会话在 Node 账本中仍在运行（没有进入结束、没有被取消、交付尚未开始），
   否则 `no_session`。
5. 会话离开运行时（结束、取消、Node 停止）时，Node 关闭并删除 socket；Node 启动时清空 `/run/ora-git/`。
6. Node 访问 Git Server 时用该会话执行的 ID 作为 `Ora-Execution-Id`，仓库地址取自 clone 记录。最终授权仍由 Git Server
   按 Cloud 当前状态判定（快照 clone 输入与执行授权决策 D5）；Node 的检查只是让结束的会话更早、更明确地失败。
7. 每个会话同一时刻只处理一个请求，并发的请求立即得到 `busy`。请求是一行不超过 4 KiB 的 JSON，应答是若干行进度与一行结果。

**错误码**（`ora-git` 的退出码与 stderr 中的稳定代码 `ora-git: <code>: <提示>`）：

| 代码 | 退出码 | 含义 |
| --- | --- | --- |
| `usage` | 2 | 参数不合法 |
| `no_session` | 3 | 没有 `ORA_GIT_SOCKET`、socket 不存在，或会话不在运行 |
| `forbidden` | 4 | 对端不是工作负载用户 |
| `busy` | 5 | 本会话已有请求在进行 |
| `not_found` | 6 | 推送时本地分支不存在；fetch 时远端没有匹配的 ref |
| `conflict` | 7 | 推送时远端的当前值不等于声明的旧值（D6） |
| `not_fast_forward` | 8 | 推送的提交不是远端当前值的后代 |
| `denied` | 9 | Git Server 返回 `401`、`403`，或以其他原因拒绝 ref 更新 |
| `unavailable` | 10 | `5xx`、网络、TLS 或时限，重试耗尽 |
| `internal` | 11 | Node 本地失败 |

`conflict` 与 `not_fast_forward` 的提示写明下一步：先 `ora-git fetch workspace <自己的 workspaceId>`，在本地合并或变基，再推送。
提示只含 ref 名与提交 ID，不含 Git 的原始输出。

为什么用固定动词的命令加 socket，而不是给 Agent 一个能用的 remote：任何能让 `git push` 直接成功的方式，都要让工作负载拿到
证书私钥，或在沙盒里开一个不经 Node 的代理；前者违背 Git Server 根决策 D5，后者让 Node 无法按会话收口。固定动词让 Node 只
执行它能完整校验的操作，Agent 也不需要了解 ref namespace。

为什么身份绑在 socket 上：同一沙盒中工作负载的进程都以同一个 UID 运行，请求内容可以任意伪造；socket 由 Node 为某个会话
创建，连接从哪个 socket 进来是 Node 自己知道的事实。对端 UID 检查挡住 Node 身份以外、工作负载以外的进程（例如以其他 UID
运行的插件）。

为什么 socket 是 `0666`、只靠对端 UID 判定，而不是用 `0660` 加工作负载组在文件系统层挡住：目录 `0700` 时工作负载连不上；
用组控制则要在镜像里多维护一个组，并让拒绝分散在两层，非工作负载 UID 在文件系统层得到 `EACCES`，`forbidden` 永远不可达，
验收也就无法证明 Node 的检查生效。对端 UID 检查是唯一的判定点，`0711` 的目录与随机名只是让别的进程找不到 socket，不承担授权。

### D6：push-workspace：带声明旧值推送自己的 Workspace 分支

`ora-git push [<branch>]`：

1. 以工作负载身份、加固配置在 checkout 中读取 `refs/heads/<branch>^{commit}`，得到提交 `C`；没有该分支则 `not_found`。
2. 声明的旧值 `O` 取 Node 账本中本 checkout 对 `refs/heads/workspaces/<self>/<branch>` 的上次已知值；没有记录时为“不存在”。
   账本只在本 checkout 的推送成功和 `ora-git fetch workspace <self>` 之后更新。
3. Node 建立本请求的传输仓库，以 `objects/info/alternates` 借用 checkout 的对象库（借用前确认 checkout 没有自己的
   `objects/info/alternates`），不枚举 checkout 的 ref；以 Node 身份运行
   `git push --porcelain --force-with-lease=<dst>:<O> <repository_url> <C>:<dst>`，`<dst>` 是
   `refs/heads/workspaces/<self>/<branch>`。`--force-with-lease` 只用来声明旧值；能否非快进由 Git Server 决定，它总是拒绝非快进
   （契约 ADR D4）。
4. 结果：

| Git 的应答 | 处理 |
| --- | --- |
| 更新成功，或远端已是 `C` | 成功；更新账本，并以工作负载身份把 checkout 的 `refs/remotes/ora/workspaces/<self>/<branch>` 设为 `C` |
| 旧值不符（stale info） | 读取远端当前值 `R`：`R = C` 时按成功处理（上一次推送已写入，但账本没来得及更新）；否则 `conflict`，提示中给出 `R` |
| 推送前本地判定为非快进（见下） | `not_fast_forward`，不发起推送 |
| 其他 ref 更新被拒、`401`、`403` | `denied`，不重试 |
| `5xx`、网络、TLS、时限 | 重试，共 3 次，间隔 5 秒、30 秒；耗尽为 `unavailable` |

- 非快进在推送前由 Node 在传输仓库里判定：`O` 存在且 `git merge-base --is-ancestor <O> <C>` 不成立，就是 `not_fast_forward`。
  Git Server 的 `pre-receive` 只返回统一的拒绝信息（现有 hook 输出 `ora: push rejected`），Node 无法从应答区分非快进与其他拒绝；
  本地判定之后仍被拒的，一律按 `denied` 处理。
- 推送只写这一个 ref，不推 tag，不推其他分支。
- 推送不改变 checkout 的本地分支、HEAD、工作区与 index。
- 交付与此无关：交付仍把最终提交推到交付 ref（交付推送决策 D2），Workspace 分支是否推送过不影响 Revision。

为什么声明旧值，而不是只依赖服务端的快进规则：快进规则只保证不丢提交；声明旧值还保证“Agent 以为远端在哪”与远端一致，
例如同一 Workspace 的上一次会话推过新提交而本会话没有 fetch，推送会以 `conflict` 失败，提示 Agent 先看一眼再推，而不是
在不知情时把自己的提交叠上去。旧值取 Node 账本而不是 checkout 里的跟踪 ref，因为后者可以被 Agent 任意改写。

### D7：fetch：取入 `refs/remotes/ora/...`

`ora-git fetch upstream` 与 `ora-git fetch workspace <id> [<branch>]`：

1. Node 建立本请求的传输仓库，借用 checkout 的对象库；只写入 Node 自己的协商 ref，指向本 checkout 的 `base_commit` 与
   账本中的已知值，使 Git Server 只发送 checkout 尚未拥有的对象。
2. 以 Node 身份运行 `git fetch --no-tags <repository_url> <src>:refs/ora/fetched/<...>`：

| 动词 | `<src>` | checkout 中的目标 |
| --- | --- | --- |
| `fetch upstream` | `refs/ora/upstreams/<bindingId>/heads/*` | `refs/remotes/ora/upstream/*` |
| `fetch workspace <id>` | `refs/heads/workspaces/<id>/*` | `refs/remotes/ora/workspaces/<id>/*` |
| `fetch workspace <id> <branch>` | `refs/heads/workspaces/<id>/<branch>` | `refs/remotes/ora/workspaces/<id>/<branch>` |

   没有匹配的 ref 时 `not_found`。失败分类与重试同 D6。
3. 把传输仓库交给工作负载用户，以工作负载身份、加固配置在 checkout 中执行
   `git fetch --no-tags <传输仓库路径> refs/ora/fetched/<...>:refs/remotes/ora/<...>`（只有这条命令放开 `file` 协议）；取的是自己的
   分支时更新账本。删除传输仓库。

- fetch 只写 `refs/remotes/ora/**` 与对象，不改本地分支、HEAD、工作区、index 与仓库配置；合并由 Agent 自己做。
- 远端的 ref 不会被删除（契约 ADR D4），所以不需要 prune。
- `refs/remotes/ora/**` 是 Agent 可写的普通 ref，Node 从不信任它们的值：推送的旧值取账本，协商只用账本与 `base_commit`。

### D8：交付时把最终提交快进到 `workspaceRef`

交付输入增加 `workspace_ref`（快照 clone 输入与执行授权决策 D6），必须等于本 checkout clone 记录中的值。交付怎样在推送交付
ref 之后快进它、失败为什么不影响交付，由交付推送决策 D2 承担。本文只规定它与 `ora-git push` 共用 D6 的账本：快进带账本中的
已知值作为声明的旧值，成功后更新账本；与 `ora-git push` 不同，交付的快进不做本地祖先检查，非快进由 Git Server 拒绝。

### D9：续作恢复用到的 clone 记录

续作恢复由交付推送决策 D5 承担。它从本 checkout 的 clone 记录取得两项输入：协商起点与改写提示的判定基准 `base_commit`，以及
`checkout --force -B` 的本地分支名（`workspace_ref` 的分支短名，即 clone 时检出的分支）。恢复仍在 Agent 启动之前，所以恢复期间
`ora-git` 的 socket 还没有交给 Agent；恢复不改 `refs/remotes/ora/**` 与仓库配置。

### D10：Node 协议与持久记录的相应修改

按交付推送决策 D8 的写法列出：

- `CloneExecutionSpec`：删除 `repository: CloneRepositoryUrl` 与 `branch: BranchName`，增加 `repository_url`、`snapshot_ref`、
  `base_commit`、`workspace_ref`，校验见 D1。`CloneRepositoryUrl` 的 SSH 分支与外部地址规则从协议中删除。
- `CloneReady`：形状不变，`commit` 必须等于 `spec.base_commit`，`spec` 是新形状。
- `CloneFailureCode`：增加 `SnapshotUnavailable`、`BaseCommitMismatch`；`BranchNotFound` 只出现在历史记录中。
- `DeliverRevision`：按交付推送决策 D8 删除 bundle、增加仓库地址、交付 ref 与 `workspace_ref`（D8）。
- 会话的 `AgentSession`：按交付推送决策 D8 改 `PriorRevision`；增加 `workspace_directory`，Node 原样用于 `ora-git list`，不据此做
  任何判断。仓库地址、`workspaceId`、分支短名、`bindingId` 与 `base_commit` 从 `checkout_execution_id` 指向的 clone 记录取得。
- Controller–Node 协议不增加消息：`ora-git` 是工作负载与 Node 之间的本地接口；Workspace 分支的状态由 Cloud 需要时经 ref 查询
  读取（契约 ADR D6）。
- 能力：Node 只在同时配置了 `clone` 与 `git_access` 时声明 clone 能力；Cloud 模式缺少 `git_access` 时拒绝启动（交付推送决策
  D7，不变）。
- Node 数据库：clone 记录的输入改为新形状，schema 升一版，不迁移旧记录（破坏性切换，开发环境的 Node 数据随 Workspace 重建）。
  新增本地账本，记录每个 checkout 对自己 Workspace 分支的已知值（D6、D8）。
- Node 镜像包含 `ora-git`，置于工作负载的 `PATH`。

Node 协议的消息拒绝未知字段，新旧 Node 与 Controller 不能混用，与“同时发布”一致。

## 不变量

1. Node 的 clone 只访问输入中的 Git Server 仓库，只读取 `snapshot_ref`；不访问任何外部仓库，不使用任何部署凭据。
2. clone 成功时，checkout 的 HEAD 是本地分支 `<branch>`，指向 `base_commit`；结果的 `commit` 等于 `base_commit`。
3. checkout 中没有任何 remote，也没有证书、私钥路径、`Ora-Execution-Id` 或其他凭据相关的配置。
4. git-access 私钥只被 Node 身份的 Git 进程经命令行引用；工作负载身份的进程从不获得它，带证书的进程从不读取 checkout
   的配置、钩子或 ref。
5. `ora-git` 的请求只归属于创建该 socket 的会话执行，对端必须是工作负载 UID；会话不在运行时请求被拒绝。
6. `ora-git` 只能推送 `refs/heads/workspaces/<自己的 workspaceId>/` 下的分支，每次只推一个 ref，带声明的旧值。
7. `ora-git fetch` 只写 checkout 的 `refs/remotes/ora/**` 与对象。
8. 推送的旧值与 fetch 的协商起点只来自 Node 账本与 `base_commit`，不来自 checkout 中 Agent 可写的 ref。
9. 恢复与交付快进所用的分支短名、`base_commit` 与 `workspace_ref` 都来自本 checkout 的 clone 记录。
10. 旧形状的 clone 命令被拒绝。
11. 交付对 `workspace_ref` 的快进与 `ora-git push` 共用同一份账本。

## 为什么不是这些替代方案

| 方案 | 取舍 |
| --- | --- |
| 以工作负载身份直接 clone Git Server，私钥经继承的文件描述符传入 | 少一个传输仓库；但私钥进入工作负载进程，违背 Git Server 根决策 D5（交付推送决策已否决同一做法） |
| 以 Node 身份直接 clone 出 checkout，再把属主交给工作负载 | 一条 `git clone` 完成；但之后的 fetch 与推送仍要回到传输仓库模式，clone 成为唯一以 Node 身份写 checkout 的路径，还要额外删除 Node 写下的 remote 配置 |
| checkout 配置 `origin` 指向 Git Server | Agent 熟悉 `git fetch origin`；但没有凭据它永远失败，且诱导 Agent 改 remote 去访问外部（D2） |
| checkout 保留指向外部仓库的 `origin` | 与开发者习惯一致；但让 Agent 与工具绕过平台访问外部仓库，与契约 ADR 需求 4 的方向相反 |
| 沙盒内运行一个 Git HTTP 代理，由它附加证书，Agent 用普通 `git push` | Agent 不需要学新命令；但代理要解析 Git 协议才能按会话与 ref 收口，等于在沙盒里复制一份 Git Server 的授权，且代理本身需要私钥 |
| `ora-git` 透传任意 Git 参数 | 更灵活；但 Node 以 Node 身份运行 Git，任意参数可以改写目标、配置或读取文件 |
| 请求中声明会话执行 ID | socket 可以共用一个；但同 UID 的进程能任意伪造请求内容，身份失去意义（D5） |
| socket `0660` 加工作负载组，在文件系统层拒绝其他 UID | 多一层防护；但要多维护一个组，拒绝分散在两层，Node 的对端 UID 检查无法被观察和验收（D5） |
| clone 拉取的段内重试与重启后的重新拉取各自计数 | 单次重启内的重试不受之前重启影响；但最坏 9 次拉取，反复重启时 clone 迟迟不失败（D4） |
| 推送只依赖服务端的快进规则，不声明旧值 | 少一份账本；但 Agent 在不知道远端已前进时叠加提交，冲突发现得更晚（D6） |
| 旧值取 checkout 里的 `refs/remotes/ora/...` | 不需要账本；但 Agent 可以改写这些 ref，旧值失去意义（D6） |
| fetch 直接写本地分支或自动合并 | Agent 少一步；但会改写 Agent 的工作分支，合并冲突也要由 Node 决定怎样处理 |
| clone 失败时按外部 clone 的方式回退 | 快照缺失时仍能工作；但外部凭据会回到 Node，违背契约 ADR 需求 3 |

## 本决策未解决的问题


- **大仓库的首次 clone 时限**：与交付推送决策相同，本文沿用 `network_timeout_ms`。
- **Workspace 分支的非快进改写**：第一阶段只快进（契约 ADR 未解决项）；`ora-git` 不提供 force 推送。
- **非 Agent 的工作负载进程使用 `ora-git`**：目前只有 Agent 会话拥有 socket；插件进程等其他工作负载若需要，另行决定。
- **纵深安全加固**（契约 ADR 未解决项，第一阶段不处理）：
  - Node 以 Node 身份借用 checkout 的对象库（`objects/info/alternates`），读取工作负载可控的对象文件，带来的解析与资源消耗攻击面；
  - 本文依赖“工作负载与插件进程读不到 git-access 私钥”这个前提（Git Server 根决策 D5 的既有缺口）。第一阶段验收逐项检查它
    （验收 5g），但不在本文增加新的防护。

## 落地顺序

这些步骤都属于契约 ADR 落地顺序第 2 步中的内部线，在底座（第 1 步）之后进行，不依赖外部线。

1. **git-access 身份与 Compose**（cluster）：Sandbox Server 按 [Git Server 根决策](../../cloud/git-server/0-project-code-hub-git-server-over-https.md)
   D2 为每个沙盒签发 git-access 客户端身份，存入管理 tmpfs；Compose 接入 Git Server，仓库监听只在沙盒网络、ref 查询只在默认网络
   （根决策 D6）；两个 Node 配置模板启用 `git_access`（交付推送决策 D7）。未合并的已有实现只作参考，按本组 ADR
   重新实现。完成条件：Sandbox Server 测试证明证书 SAN、用途与 WSS、model-access 身份分开；真实栈中沙盒只能连到仓库监听。
2. **协议与校验**（desktop）：`node-protocol` 的新 `CloneExecutionSpec`、失败码、`DeliverRevision` 与会话输入的修改，以及交付推送
   决策 D8 的 bundle 删除（D1、D10）；锁定 Cloud 的新契约 commit。完成条件：协议测试覆盖每个字段的拒绝规则、旧形状被拒、
   `snapshot_ref` 末段与 `base_commit` 不等被拒。
3. **clone**：传输仓库拉取、工作负载段、核实与失败分类、持久阶段（D2–D4）。完成条件：以真实 Git 与 Git Server 程序测试：成功后
   没有 remote、HEAD 等于 `base_commit`；ref 缺失与 `403` 为 `SnapshotUnavailable`；ref 指向别的提交为 `BaseCommitMismatch`；
   `503` 重试后成功与耗尽；拉取阶段重启后重做且有上限；工作负载段被终止为 `Interrupted`；现有正常停止、命令超时、强杀恢复、
   guardian 丢失的测试原样通过。
4. **交付与恢复改走 Git**：按交付推送决策实现交付推送与 `workspaceRef` 快进、冻结计划、恢复与 `git_access` 必填，删除 bundle 的
   生成、上传与下载。完成条件：交付推送决策落地顺序第 3、4 步的完成条件。
5. **`ora-git` 与 socket 服务**：动词、参数校验、socket 身份、错误码、push 与 fetch、`list` 附带目录（D5–D7）。完成条件：测试覆盖
   不变量 4–8，包括 checkout 中恶意的 `credential.helper`、`http.<url>.sslVerify`、`url.<base>.insteadOf` 与
   `objects/info/alternates` 不生效；对端 UID 不符被拒；会话结束后 socket 消失；推送重放时 stale info 但远端等于 `C` 按成功处理。
6. **端到端验收与合并**：下一节，与 Cloud、Controller 的实现一起验收，然后三个仓库一起合并。从合并起，Workspace 只从快照
   clone；外部线合并之前，只有用预置工具准备了快照的 Project 能创建 Workspace（契约 ADR 落地顺序）。

保持不变：受理与 Git 执行的线程划分、中断判定、目标目录的独占与保留、会话 JSONL。
从第 6 步开始改变：Node 的代码来源只有 Git Server 快照；交付只经 Git，不再有 bundle；Agent 可以经 `ora-git` 与其他 Workspace 协作。

## 验收

内部线端到端验收，不依赖外部线；三篇内部线 ADR 共用这一个栈与脚本（cluster 的 `task workspace-git:fixture:up` 与
`task workspace-git:acceptance`，文档与现有验收文档放在一起）。

### 起栈

- cluster 现有 Compose，加落地顺序第 1 步接入的 Git Server（`git-server` 服务，仓库监听在沙盒网络、ref 查询在默认网络）与
  echo agent fixture（`compose.agent-fixture.yaml`）。Node 配置启用 `git_access`。
- 验收 overlay 把沙盒网络建成 `--internal`：fixture 脚本以 `docker network create --internal` 建一个专用网络，并以
  `ORA_SANDBOX_NETWORK` 指向它。沙盒仍能访问同网络上的 Git Server、插件源与 Controller，但没有通往外部的路由。脚本先在一个
  沙盒里证明这一点：解析或连接一个公网地址失败。
- 被验收的 Project 的外部仓库地址故意填一个外部地址；clone 成功本身就证明 Node 没有使用它。

### 替身

外部线不在栈中。先经公开 API 创建 Project：底座让建 Project 的事务同时建立 Binding 和一行 `pending` 的同步 Operation
（契约 ADR 落地顺序第 1 步），所以这时 Project 已经是真实流程里的形状。然后由底座的预置工具充当外部线：

1. 把一个本地生成的 fixture 仓库（含两次提交）的提交写成 `refs/ora/upstreams/<bindingId>/snapshots/<oid>` 与
   `refs/ora/upstreams/<bindingId>/heads/main`；
2. 写当前快照记录，并把同步 Operation 推进为 `succeeded`（需要失败场景时推进为 `failed`）；
3. 运行共享的契约断言，确认 Git Server 上的 ref 与 Cloud 的记录符合契约 ADR D5。

预置工具不写 Workspace 记录：Workspace 都经真实的 API 与 IssueRun 创建。

需要时再写第二份快照：一个是前一份的子提交（外部仓库前进），一个与前一份无关（外部仓库被 force-push）。

### echo 插件的扩展

echo agent 的标记按任务正文中的出现顺序执行；`[write]` 同样保持顺序，追加的 `\n\nContext\n` 上下文不解释标记。
Git 命令都在 checkout（会话工作目录）中运行，提交用固定的身份
`echo <echo@ora.invalid>`：

| 标记 | 执行 |
| --- | --- |
| `[commit]` | `git add -A`，`git commit -m "echo: <提问>"` |
| `[push]` | `ora-git push` |
| `[fetch <ws>]` | `ora-git fetch workspace <ws>`，然后 `git merge --no-edit refs/remotes/ora/workspaces/<ws>/main` |

每个标记的结果（`ok` 或 `ora-git` 的错误码）追加在回复里，例如 `echo: … [push: ok]`，脚本从 Thread 事件读取。desktop 的 Rust
echo fixture 同步增加相同标记。

### 场景

| # | 场景 | 断言 |
| --- | --- | --- |
| 1 | **从快照 clone 并校验**：经公开 API 创建 Project（Binding 与第一行同步随之建立），替身再写入快照 ref 与当前快照记录，然后对一个 Issue @Echo，提问 `[write]` | `psql`：clone 执行的 `result.commit` 等于快照提交，Workspace 的 `base_commit_id` 相同。会话中 `docker exec -u <工作负载 UID>` 进沙盒：`git -C <checkout> remote` 为空；`git config --list` 不含 `remote.`、`http.`、`credential.`；`git symbolic-ref HEAD` 为 `refs/heads/main`；`rev-parse HEAD` 等于快照提交；`objects/info/alternates` 不存在 |
| 2 | **commit 后 push-workspace**：Issue A，提问 `[write] [commit] [push]` | 回复含 `[push: ok]`。在 Git Server 容器里 `git for-each-ref refs/heads/workspaces/`：A 的 `workspaces/<A>/main` 指向新提交，其父提交是快照提交 |
| 3 | **第二个 Workspace fetch、merge 再推送**：A 完成后，Issue B @Echo，提问 `[fetch <A>] [write] [commit] [push]` | 回复含 `[push: ok]`；Git Server 上 `workspaces/<B>/main` 的祖先包含 A 的提交；B 的 checkout 中有 `refs/remotes/ora/workspaces/<A>/main`。`workspaces/<A>/main` 不变 |
| 4 | **交付、Revision 登记、续作恢复**：另一个 Issue C 提问 `[write] [commit]`，不推送，结束会话；然后 A 运行结束；同一 Issue 再 @Echo 一次（续作 A 的 Revision），提问 `[push]` | `psql`：A 的 `revisions.delivery_ref` 存在，Git Server 上该 ref 指向登记的最终提交；交付顺带把 `workspaces/<A>/main` 快进到 A 的最终提交。C 从未推送，交付后 `workspaces/<C>/main` 指向 C 的最终提交；C 的工作区在 `[commit]` 之后是干净的，所以最终提交就是 C 的 `[commit]` 提交。续作运行：Agent 启动前 checkout 的 `main` 指向 A 的最终提交；新 Workspace 的 `base_commit_id` 是快照提交；`[push: ok]` 后 `workspaces/<新 Workspace>/main` 指向 A 的最终提交。替身先写一份无关的快照再续作时，首条提问末尾出现历史改写的提示；写子提交快照时不出现 |
| 5a | **没有快照**：新 Project 不写快照 | 见 [clone 步骤消费快照决策](../../cloud/operation/20261016-clone-step-consumes-binding-snapshot.md) 的验收：同步未结束时停在 `snapshot_not_ready`，同步失败后以 `repository_sync_failed` 结束；两种情况都没有 clone 执行，Node 不访问外部地址 |
| 5b | **写别人的分支**：会话中以工作负载身份运行 `ora-git push main:refs/heads/workspaces/<A>/main` 与 `ora-git push --force` | 两者都是 `usage`（退出码 2）；再以 Node 身份直接推送到 `workspaces/<A>/main`（[快照 clone 输入与执行授权决策](../../cloud/controller-integration/20261016-snapshot-clone-input-and-execution-ref-grants.md) 的验收）被 Git Server 拒绝；Git Server 上该 ref 不变 |
| 5c | **非快进**：B 推送后，以工作负载身份在 B 的 checkout 中 `git reset --hard <快照提交>`、提交一个新提交，再 `ora-git push` | 退出码 8，`not_fast_forward`；Git Server 上 `workspaces/<B>/main` 不变 |
| 5d | **声明的旧值不符**：以 Node 身份直接把 `workspaces/<B>/main` 快进到一个新提交（模拟同一 Workspace 的另一次推送），再在 B 的会话中 `ora-git push` | 退出码 7，`conflict`，提示中给出远端当前值；`ora-git fetch workspace <B>` 并合并后再推送成功 |
| 5e | **旧代次**：脚本以 root 从 main Workspace 的沙盒复制 git-access 证书，然后以租户管理员身份经公开 API 停止并重新启动 main Workspace（新 generation；非管理员还需要运行时会话），用旧证书从沙盒网络上的辅助容器 `ls-remote` | 访问被拒绝（`403`）；新沙盒的证书 SAN 中 generation 已递增。run Workspace 不能经公开 API 重启，main Workspace 又没有进行中的会话执行，所以端到端无法构造“只有代次不符、其余条件都成立”的请求；这一条由 [快照 clone 输入与执行授权决策](../../cloud/controller-integration/20261016-snapshot-clone-input-and-execution-ref-grants.md) 落地顺序第 2 步的集成测试覆盖 |
| 5f | **执行结束后再推**：会话结束后，以工作负载身份运行 `ora-git push` | `no_session`（退出码 3），socket 文件已不存在；以 Node 身份用会话执行 ID 推送得到 `403` |
| 5g | **工作负载用户读不到证书**：会话中以工作负载 UID 读取管理 tmpfs 中的 git-access 私钥与证书；再以既不是工作负载也不是 Node 的另一个 UID（例如 `nobody`）对 `ORA_GIT_SOCKET` 运行 `ora-git info` | 读取失败（权限不足）；Agent 进程的环境中只有 `ORA_GIT_SOCKET`，没有证书路径；`ls /run/ora-git` 失败；另一个 UID 能连上 socket（文件系统层不拒绝），由 Node 的对端 UID 检查以 `forbidden`（退出码 4）拒绝。断言的是 Node 应用层的判定，不是文件系统权限 |

5b、5c、5d、5g 需要会话仍在运行，脚本在结束对应会话之前执行它们；5f 在会话结束之后执行。

### 断言的手段

沿用 cluster 现有验收的惯例：`psql` 查 Cloud 数据库；`docker exec` 进沙盒以指定 UID 运行 `git` 与 `ora-git`；`docker exec` 进
`git-server` 容器，对仓库卷中的 `/{tenantId}/{projectId}.git` 运行 `git for-each-ref` 查 ref。证据写入
`.acceptance/workspace-git.json`，只含资源 ID、ref 名、提交与错误码，不含证书内容或路径以外的材料。

## 核心测试用例

本 ADR 的核心测试用例见[内部线用例](../../../test-cases/node/repository/snapshot-clone-and-ora-git.md)。实现尚未合入 main，义务都如实标为 Missing；现有以已实现决策为依据的用例
在实现合入、本 ADR 改为 `implemented` 时再按本文改写或改挂，见该用例文件开头的说明。
