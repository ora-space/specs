# Node 从 Git Server 快照 clone 与 `ora-git`

依据已批准的 [Node 从快照 clone 决策](../../../decisions/node/repository/20261016-clone-from-git-server-snapshot-and-ora-git.md)。
下文的 D 编号与不变量编号都指这篇 ADR；“交付推送决策”指
[Node 交付推送与 Git 恢复决策](../../../decisions/node/revision/20261017-push-delivery-ref-and-restore-from-git-server.md)。

现有的 [仓库获取与结果边界](clone-and-result-boundaries.md)、[最小 clone 执行](minimal-clone-execution.md) 与
[准入与 Git 执行](admission-and-git-execution.md) 仍以已实现的外部 clone 决策为依据，实现合入并把本 ADR 改为 `implemented` 时再按本文
改写或改挂。内部线的实现尚未合入 desktop 的 main，下列义务都是 **Missing**。

## Clone Accepts Only A Git Server Snapshot Input

- 风险：旧形状或外部地址的 clone 命令仍被执行，Node 以部署凭据访问外部仓库；`snapshot_ref` 与 `base_commit` 不对应。
- 前置状态：新 Node 协议与执行器。
- 触发：带 `repository`/`branch` 的旧命令；`ssh://` 或带用户信息的地址；`snapshot_ref` 末段不等于 `base_commit`；`workspace_ref` 短名为 `HEAD`；同一执行身份换了输入。
- 必须成立：只接受 D1 形状的输入；同身份不同输入冲突；clone 只访问输入中的 Git Server 仓库并只读取 `snapshot_ref`。
- 禁止结果：旧形状被执行；任何外部仓库或部署凭据被使用。
- 验证义务与证据：**Missing**。
- 决策依据：D1、D10，不变量 1、10。

## A Snapshot Checkout Has No Remote And Points At The Base Commit

- 风险：checkout 留下 origin、凭据相关配置或 alternates，Agent 绕过平台访问外部，或删除传输仓库后对象缺失；成功时 HEAD 不是快照提交。
- 前置状态：Git Server 上有快照 ref 指向 `C`。
- 触发：clone 成功；核实时 HEAD 被篡改为另一个提交。
- 必须成立：HEAD 是指向 `refs/heads/<branch>` 的符号引用，指向 `C`，工作区干净；`git remote` 为空，没有 `remote.*`、`http.*`、`credential.*` 配置，没有 `objects/info/alternates`，历史完整非浅；`CloneReady.commit = C`；传输仓库已删除。
- 禁止结果：HEAD 不等于 `C` 时报告成功（必须保留 Unknown 与目录）；checkout 出现证书路径或 `Ora-Execution-Id`。
- 验证义务与证据：**Missing**。
- 决策依据：D2、D3，不变量 2、3。

## Clone Failures Are Classified By The Git Server Answer

- 风险：快照缺失被当作网络失败反复重试；快照 ref 与 Cloud 登记不一致被掩盖。
- 前置状态：可注入应答的 Git Server 替身。
- 触发：`401`、`403`、快照 ref 不存在；取到的提交不等于 `base_commit`；`5xx`、`429`、连接与 TLS 失败、低速中止。
- 必须成立：前三者为 `SnapshotUnavailable` 且不重试；不等为 `BaseCommitMismatch`；其余重试后为 `SourceUnavailable`；`BranchNotFound` 不再产生。
- 禁止结果：`401`/`403` 被重试；不等被报告为 `SourceUnavailable`。
- 验证义务与证据：**Missing**。
- 决策依据：D3。

## Snapshot Fetch Attempts Share One Persistent Counter Across Restarts

- 风险：段内重试与重启后的重新拉取各自计数，最坏 9 次拉取，反复重启时 clone 迟迟不失败；拉取段在重启后写入目标目录。
- 前置状态：Git Server 持续返回 `5xx`；可以在拉取中杀掉 Node。
- 触发：第一次拉取失败后杀掉 Node 并重启，直到失败；工作负载段的某一步中途重启。
- 必须成立：每次拉取开始前先持久化累计次数，段内重试与重启后的拉取共用这个计数，满 3 次以 `SourceUnavailable` 结束；重启后不等待剩余退避；工作负载段的 `init`、取入、检出各一个 Run，重启按原 Run 核对，不重跑。
- 禁止结果：总拉取次数超过 3；同一步出现第二个 Run；拉取段写入目标目录。
- 验证义务与证据：**Missing**。
- 决策依据：D4。

## Credentialed Git Never Reads The Checkout Configuration

- 风险：Node 身份的 Git 读取 Agent 写过的 `credential.helper`、`core.askPass`、`http.<url>.*` 或 `url.<base>.insteadOf`，以 Node 身份执行任意命令、关闭证书校验或改写目标；私钥进入工作负载进程。
- 前置状态：checkout 中写入上述恶意配置与钩子，以及一个 `objects/info/alternates`。
- 触发：clone 之后的 `ora-git fetch`、`ora-git push`；Node 借用对象库。
- 必须成立：恶意配置与钩子都不生效；checkout 自带 alternates 时拒绝借用；私钥只以命令行配置引用，不出现在任何仓库配置、日志或结果中。
- 禁止结果：工作负载身份的进程打开私钥；传输仓库保存任何凭据。
- 验证义务与证据：**Missing**。
- 决策依据：D2、D6、D7，不变量 4；交付推送决策 D1。

## ora-git Requests Belong To The Session That Owns The Socket

- 风险：任意进程伪造请求冒充别的会话；结束的会话仍能推送；非工作负载 UID 借 socket 访问 Git Server；文件系统权限挡住连接，让 Node 的检查无法验收。
- 前置状态：两个会话各有自己的 socket；`/run/ora-git` 为 `0711`，socket 为 `0666`。
- 触发：工作负载 UID 与另一个 UID（如 `nobody`）分别连接 socket；`ls /run/ora-git`；会话结束后连接；同一会话并发两个请求；请求中夹带执行 ID 或地址。
- 必须成立：请求只归属于创建该 socket 的会话；另一个 UID 能连上但被 `SO_PEERCRED` 检查以 `forbidden` 拒绝；目录不能列出；会话结束后 socket 已删除，请求为 `no_session`；并发请求为 `busy`；未知动词、选项与多余参数为 `usage`。
- 禁止结果：请求内容影响会话归属；Node 以外的检查成为唯一拒绝点。
- 验证义务与证据：**Missing**。
- 决策依据：D5，不变量 5。

## ora-git Push Writes Only The Own Workspace Branch With A Declared Old Value

- 风险：Agent 推到别的 Workspace 或改写历史；旧值取自 Agent 可写的 ref；重放时把已写入的推送报告为冲突。
- 前置状态：Workspace B 的会话；账本记录 `workspaces/<B>/main` 的已知值。
- 触发：推送新提交；推送非快进的提交；远端被另一次推送前进后推送；上一次推送写入但账本未更新时重放；尝试 `--force` 或 refspec。
- 必须成立：只推 `refs/heads/workspaces/<B>/<branch>` 一个 ref，带账本中的旧值；成功后更新账本与 `refs/remotes/ora/workspaces/<B>/<branch>`；非快进在本地判定为 `not_fast_forward`、不发起推送；旧值不符为 `conflict` 并给出远端值；远端已等于本次提交时按成功处理；`--force` 与 refspec 为 `usage`。
- 禁止结果：旧值取自 `refs/remotes/ora/**`；推送其他 ref 或 tag。
- 验证义务与证据：**Missing**。
- 决策依据：D6，不变量 6、8。

## ora-git Fetch Writes Only Remote Tracking Refs

- 风险：fetch 改写 Agent 的本地分支、HEAD 或工作区；协商使用 Agent 可写的 ref。
- 前置状态：Git Server 上有 upstream 跟踪分支与其他 Workspace 的分支；checkout 有未提交改动。
- 触发：`ora-git fetch upstream`、`fetch workspace <id>`、`fetch workspace <id> <branch>`、取不存在的分支；`ora-git list`。
- 必须成立：只写 `refs/remotes/ora/upstream/*` 与 `refs/remotes/ora/workspaces/<id>/*` 和对象；协商只用账本与 `base_commit`；取自己的分支时更新账本；不存在为 `not_found`；`list` 按会话输入的目录附上 Workspace 信息，不在目录中的只显示 ID。
- 禁止结果：本地分支、HEAD、工作区、index 或仓库配置改变；目录参与任何判断。
- 验证义务与证据：**Missing**。
- 决策依据：D5、D7，不变量 7、8。

## Delivery Fast Forward And ora-git Push Share One Ledger

- 风险：交付快进与 `ora-git push` 各自记旧值，交付后 Agent 下一次推送以过时的旧值失败；交付或恢复用的分支名与 `workspace_ref` 不来自 clone 记录。
- 前置状态：Agent 推送过 Workspace 分支后进入交付；续作的会话。
- 触发：交付快进 `workspace_ref`；交付输入的 `workspace_ref` 与 clone 记录不同；续作恢复。
- 必须成立：快进以账本中的值为旧值，成功后更新同一份账本；不同的 `workspace_ref` 被拒；恢复的分支短名与 `base_commit` 取自 clone 记录。
- 禁止结果：存在第二份记录 Workspace 分支已知值的账本。
- 验证义务与证据：**Missing**。
- 决策依据：D8、D9，不变量 9、11。
