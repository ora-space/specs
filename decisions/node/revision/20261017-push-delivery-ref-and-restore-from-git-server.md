---
status: approved
---

# Node 交付把最终提交推送到交付 ref（无改动也推）并顺带快进 Workspace 分支，续作从 Git Server 拉取前序交付 ref 并把 Workspace 本地分支移到前序成果；带证书的 Git 访问只在 Node 私有的传输仓库中进行，Cloud 模式必须配置 `git_access`

Node 交付时仍按根决策生成内部快照与本地 `refs/ora/revisions/<runId>`，但不再生成 bundle：无论有没有改动，都把最终提交
推送到交付输入给出的 Git Server 仓库与 `refs/ora/deliveries/<deliveryWorkId>`，再尝试把同一提交快进到本 Workspace 的分支
`workspaceRef`，最后照旧上传会话 JSONL。交付 ref 被拒不重试，以 `push_rejected` 失败；Git Server 不可用有限次重试，耗尽以
`push_failed` 失败；每次推送尝试有 30 分钟时限与低速中止。Workspace 分支的快进只是附带效果，任何失败都只记日志，不影响
交付结果。续作恢复前序 Revision 时，Node 从 Git Server 拉取会话输入给出的前序交付 ref，把 Workspace 本地分支移到前序最终
提交；不再申请下载授权、下载 bundle 或访问任何外部仓库。所有带 git-access 证书的 Git 命令都以 Node 身份在 Node 私有的
传输仓库中执行，只把 checkout 的对象当数据借用，从不读取 checkout 的仓库配置。Node 在 Cloud 模式下必须配置 `git_access`。

本 ADR 已经 owner 批准，处于 `approved`（核心测试用例见[内部线用例](../../../test-cases/node/revision/git-delivery-push-and-restore.md)；实现合入、端到端验收通过后改为 `implemented`），继承 [Node 交付 Revision 根决策](0-internal-snapshot-and-incremental-bundle-upload.md) 与
[恢复前序 Revision 后续决策](20261010-restore-prior-revision-before-session.md)。它取代
[交付 ref 推送与前序拉取决策](20261012-push-delivery-ref-and-fetch-prior-revision.md)：那份决策写在 Node 仍从外部仓库
clone、Git Server 只存交付 ref 的设计上，随 Git Server 成为 Project 代码中心（[Git Server 根决策](../../cloud/git-server/0-project-code-hub-git-server-over-https.md)）整体作废，本文按新设计
重新写出当前决策。交付契约、核验登记与续作选取由
[Cloud Revision Git 交付决策](../../cloud/revision/20261017-git-native-delivery-refs-and-resume.md)（下称“Cloud 交付决策”）
承担；Workspace 分支的 clone、本地账本与 `ora-git` 由
[Node 从快照 clone 决策](../repository/20261016-clone-from-git-server-snapshot-and-ora-git.md)（下称“clone 决策”）承担。涉及
desktop 的 Node（交付、恢复、传输仓库、配置校验、能力声明）、Node 协议与 Controller 的交付映射。

兼容性边界：**破坏性切换**（契约 ADR 范围边界）。Node 不再理解带 bundle 的输入，新旧 Node 与 Controller 不能混用；不迁移 Node
数据库中的旧交付记录。Node 本地目录布局改为把传输仓库放在 clone 根目录下的 `.ora-transport/`；旧 Node 留下的
`revision-restores/` 下载目录不再使用，也不清理，开发环境的 Node 数据随 Workspace 重建。

## 继承与修改

| 前序约定 | 本文处理 |
| --- | --- |
| 根 D1：交付只在会话结束且插件进程树已停止后开始 | 继承，不变 |
| 根 D2：内部快照不经过提交钩子、不改动工作区与分支；`update-ref` 本地 `revision_ref` | 继承，不变；`revision_ref` 只在 checkout 中，从不推送（D2） |
| 根 D3：相对基础提交生成并校验累计增量 bundle；无改动不生成 bundle | 修改：不生成 bundle；`delivered`/`unchanged` 只按最终提交判定，两者都推送到交付 ref（D2） |
| 根 D4：上传 bundle 与 JSONL；首个 PUT 前冻结对象字节与声明；授权只在内存 | 修改：只上传 JSONL；首次推送之前冻结最终提交、声明与 JSONL（D3）；授权规则继承 |
| 根 D5：失败码与“冻结后不重新快照” | 修改：删除 `bundle_failed`，`upload_failed` 只指 JSONL，新增 `push_rejected`、`push_failed`（D4）；重放规则继承并扩展到推送（D3） |
| 根不变量 1：快照只写 `refs/ora/revisions/` 下的 ref | 修改：checkout 中仍只写这个 ref；Git Server 上写交付 ref 与本 Workspace 的分支（不变量 5） |
| 根不变量 4（bundle 上传前通过 `verify`）、6（无改动不生成 bundle） | 删除：已没有 bundle；由本文不变量 3、7 取代 |
| 恢复 D1：恢复在 checkout 定位后、Agent 启动前；失败以 `agent_failed` 结束；停机时不重做 | 继承；会话在恢复期间结束或被取消时，恢复在下一条 Git 命令前停止（D5） |
| 恢复 D2：经 Controller 申请下载授权，下载并校验 bundle；等待与下载合计 5 分钟 | 修改：删除下载；改为凭 git-access 证书从 Git Server 拉取，拉取、重试与取入共用 5 分钟时限（D5） |
| 恢复 D3：读 bundle 头、向 `origin` 补取前置提交、`bundle verify`、取入、`checkout -B <默认分支>` | 修改：拉取交付 ref 到传输仓库、核对最终提交、以工作负载身份取入、`checkout -B <Workspace 分支短名>`；不访问任何外部仓库（D5） |
| 恢复 D3：`prior_revision_unavailable` / `prior_revision_base_unavailable` 的判定 | 修改：按 Git Server 的应答重新划分（D6） |
| 恢复 D4：前置提交不是 `origin/<默认分支>` 的祖先时在首条提问末尾提示 | 修改：前序的基础提交不是本 checkout 快照提交的祖先时提示（D5） |
| 恢复 D5：最终提交等于基础提交或前序最终提交时 `unchanged` | 继承判定；`unchanged` 也推送（D2） |
| 恢复 D6：会话与交付都可用时声明 `revision_restore` | 继承；交付可用改为要求 `git_access`（D7） |
| 恢复不变量 4：恢复不改 `origin` 的远端跟踪分支与仓库配置 | 修改：checkout 没有 origin；恢复不改 `refs/remotes/ora/**` 与仓库配置（不变量 9） |
| [clone 最小执行契约](../repository/20260918-minimal-clone-execution-contract.md)：checkout 中的 Git 以受管执行与加固配置、以工作负载身份运行 | 继承；唯一例外是带 git-access 证书的推送与拉取，以 Node 身份在传输仓库中运行（D1） |
| clone 决策 D6：Node 账本记录本 checkout 对自己 Workspace 分支的已知值 | 继承；交付的快进与 `ora-git push` 共用这份账本（D2） |
| [Git Server 代码中心决策](../../cloud/git-server/20261016-project-repository-as-code-hub-and-ref-contract.md)（下称“契约 ADR”）D4：交付 ref 只能新建，Workspace 分支只能新建或快进；读侧不隐藏 ref | 继承（D2、D5） |
| [Node 协议](../protocol/20260928-streamed-thread-events-and-session-commands.md) D5：`UploadGrant`/`UploadGrantNeeded`；恢复决策增加的下载授权消息 | 修改：上传授权只涉及 JSONL；删除下载授权消息（D8） |

## 决策

### D1：带证书的 Git 访问以 Node 身份在私有传输仓库中执行

git-access 私钥在管理 tmpfs（`/run/ora`）中只有 Node 身份可读，所以推送与拉取必须以 Node 身份运行。但不能以 Node 身份直接
在 checkout 中运行 Git：checkout 的 `.git/config` 由 Agent 写过，Git 会读取其中的 `credential.helper`、`core.askPass`（以 Node
身份执行任意命令），以及按 URL 匹配的 `http.<url>.*` 与 `url.<base>.insteadOf`（它们优先于命令行上的通用 `http.*`，能关闭证书
校验或改写目标）。因此：

- 每次交付、恢复、clone 与 `ora-git` 请求各有一个私有的裸**传输仓库**：
  `<clone.repository_root>/.ora-transport/<kind>-<sha256(execution)>`，`kind` 为 `delivery`、`restore`、`clone` 或 `ora-git`。
  `.ora-transport/` 目录权限 `0711`，传输仓库本身 `0700`、属主 Node，以空模板初始化，配置只由 Node 写入，从不保存凭据。
- 传输仓库通过 `objects/info/alternates` 把 checkout 的对象库当作只读数据借用；checkout 有 `commondir`、自己的 alternates
  或对象目录不是普通目录时拒绝借用。`core.alternateRefsCommand=true` 让 Git 不枚举 checkout 的 ref，所以 Node 身份的 Git
  进程从不读取 checkout 的配置、钩子或 ref。
- 证书、私钥与 CA 只以 `-c http.sslCert=… -c http.sslKey=… -c http.sslCAInfo=… -c http.sslVerify=true` 传入，执行 ID 以
  `-c http.extraHeader=Ora-Execution-Id: <id>` 传入；另以 `credential.helper=`、`core.askPass=`、`http.followRedirects=false`、
  `http.lowSpeedLimit=1024`、`http.lowSpeedTime=60`、只允许 `https` 的协议策略、`core.hooksPath=/dev/null`、关闭自动 gc、
  maintenance 与 submodule 递归运行，环境为 `GIT_CONFIG_NOSYSTEM=1`、`GIT_CONFIG_GLOBAL=/dev/null`、`GIT_TERMINAL_PROMPT=0`、
  `HOME` 指向传输仓库。这些值不写入任何仓库配置，也不进入日志、journal 或结果；错误信息去掉命令参数，Node 只记录有限的
  分类码。
- 仓库地址与 ref 只取自执行输入：地址必须是 `https`、不带用户信息、查询与片段；按 URL 访问，不新增 remote。交付使用交付
  执行的 ID，恢复使用会话执行的 ID。
- 需要工作负载身份从传输仓库取入对象时，Node 先把传输仓库交给工作负载用户（其中没有任何凭据）。传输仓库用完即删；Node
  启动时清理 `.ora-transport/` 中的残留，只保留工作负载段尚未完成本地取入的 clone 传输仓库（clone 决策 D2、D4）。

为什么放在 clone 根目录下而不是 Node home：工作负载用户要按路径从传输仓库取入对象，而 Node home 对工作负载关闭；clone
根目录本来就是工作负载可达、由 Node 管理的位置。`0711` 的父目录让工作负载只能进入已知名字的子目录，不能列出其他执行的
传输仓库。为什么不以工作负载身份运行并把私钥经继承的文件描述符交给 Git：那样私钥会进入工作负载身份的进程，违背
“git-access 私钥不进入工作负载”的前提。

### D2：交付推送最终提交（无改动也推），再快进 Workspace 分支

快照（根 D2）完成并冻结计划（D3）后，无论结果是什么，Node 都在交付的传输仓库中依次执行：

1. `git push --porcelain <gitRepositoryUrl> <最终提交>:<deliveryRef>`，不带 lease：交付 ref 只能新建，由 Git Server 保证。
2. 交付 ref 推送成功后，`git push --porcelain --force-with-lease=<workspaceRef>:<O> <gitRepositoryUrl> <最终提交>:<workspaceRef>`，
   `O` 取 clone 决策 D6 的账本中本 checkout 对 `workspaceRef` 的已知值，没有记录时声明“不存在”。
3. 上传 JSONL（根 D4 的规则）。

| 最终提交 | 结果 | 推送 |
| --- | --- | --- |
| 等于基础提交 | `unchanged` | 交付 ref 指向基础提交；尝试快进 Workspace 分支 |
| 等于输入中前序的最终提交 | `unchanged` | 交付 ref 指向前序最终提交；尝试快进 Workspace 分支 |
| 其他 | `delivered` | 推送交付 ref；尝试快进 Workspace 分支 |

- 成功结果为 `RevisionDelivered{node, final_commit, base_commit, revision_ref, history}` 或
  `RevisionUnchanged{node, final_commit, base_commit, revision_ref, history}`，不带 bundle；交付 ref 与 `workspaceRef` 是固定
  输入的一部分，结果不重复声明。
- `revision_ref`（`refs/ora/revisions/<runId>`）只写在 checkout 中，从不推送；Git Server 上没有 `refs/ora/revisions/*`。
- `workspaceRef` 必须在 `refs/heads/workspaces/` 下，且等于本 checkout clone 记录中的值，否则 Node 拒绝该交付输入。
- **Workspace 分支的快进不做本地祖先检查**：与 `ora-git push`（clone 决策 D6）不同，Node 不先判断最终提交是不是已知值的
  后代，直接带声明旧值推送；非快进由 Git Server 拒绝（`pre-receive` 与 `receive.denyNonFastForwards`，契约 ADR D4）。
  旧值不符时 Node 用 `ls-remote` 读取远端当前值，等于最终提交时按成功处理（上一次推送已写入，但账本没来得及更新）。
  成功后在完成交付之前更新账本。
- 快进被拒、旧值不符、`403`、重试耗尽都只记一条有限的分类日志，不改变交付结果，也不阻止 JSONL 上传。交付结果不增加字段。
- 工作区干净时最终提交就是 HEAD；有未提交改动时，最终提交是 HEAD 之上的内部快照提交。Agent 推送过的分支只要仍是 HEAD 的
  祖先，快进就成立；Agent 改写了已推送的历史时快进被拒，成果仍在交付 ref 上。
- Node 的 checkout 不是浅克隆：第一次向某个 Project 推送时，Git Server 已持有快照及其历史（clone 决策 D2），推送只传输快照
  之后的对象；指向已有提交的 `unchanged` 推送不传输对象。

为什么无改动也推送：Revision 能否续作只看它自己的交付 ref（Cloud 交付决策 D7），`unchanged` 也推送就不再需要“沿用前序
bundle”的一跳引用，Cloud 对两种结果做同一种核验。代价是多一次几乎不传数据的推送，以及 Git Server 不可用时 `unchanged`
也会失败。

为什么快进放在交付 ref 之后、失败不影响交付：交付 ref 是 Revision 的核验对象，必须先成功；Workspace 分支只是让 run 的成果
出现在协作的地方（执行授权决策 D6），它被拒通常是因为 Agent 自己改写了历史，不应拖垮交付。为什么这里不做本地祖先检查：
`ora-git push` 的本地检查是为了给 Agent 一个可操作的 `not_fast_forward` 提示；交付时没有 Agent 读这个结果，服务端的快进规则
已经保证不丢提交，结果又只记日志，本地检查只多一条命令。

### D3：冻结计划与重放

- 首次推送之前，Node 把最终提交、结果声明与 JSONL 的字节、大小、SHA-256 作为冻结计划写入账本；JSONL 复制在
  `<Node home>/revision-deliveries/<sha256(execution)>/history.jsonl`（`0600`）。冻结之前的重启重新准备，之后的重启沿用同一
  计划，不重新快照。最终提交的对象已在 checkout 中，本地 `revision_ref` 也指向它。
- 顺序为推送交付 ref、快进 Workspace 分支、上传 JSONL；交付 ref 与 JSONL 都成功才发送成功结果。推送是更大、更可能失败的
  一步，先做可以在它确定失败时不再等待 JSONL 的上传授权。
- 重放推送同一提交：远端交付 ref 已指向该提交，`git push` 报告 up to date，按成功处理；Workspace 分支的重放见 D2。远端交付
  ref 已存在且指向其他提交时推送被拒，按 `push_rejected` 处理；正常情况下这不会发生，因为只有本执行能新建该 ref。
- Controller 断开、Node 关闭与生命周期取消时终止进行中的 Git 进程，不写失败；重启后继续同一执行（根 D5）。冻结文件在终态
  持久化后删除，Node 启动时清理残留。

### D4：失败分类、重试与时限

| 情况 | 识别 | 处理 |
| --- | --- | --- |
| HTTP `401`、`403` 或 `429` 以外的其他 `4xx` | Git 报告服务端返回的状态码 | 不重试，`failed{push_rejected}` |
| ref 更新被拒（`pre-receive` 拒绝、ref 已存在、远端没有该 ref） | `--porcelain` 输出中该 ref 以 `!` 标记，或 Git 报告找不到远端 ref | 不重试，`failed{push_rejected}` |
| HTTP `5xx`、`429`、连接失败、DNS、TLS 握手失败、低速中止、单次时限到期、无法识别的错误 | 其余情况 | 重试；共 3 次尝试，间隔 5 秒、30 秒；耗尽后 `failed{push_failed}` |
| 本地准备失败（传输仓库无法建立、最终提交不在 checkout 中） | Node 自身 | `failed{snapshot_failed}` 或 `failed{checkout_unavailable}` |

- 交付的每次推送尝试时限 30 分钟，与 clone 配置的 `network_timeout_ms` 无关；另以 `http.lowSpeedLimit=1024`、
  `http.lowSpeedTime=60` 在一分钟低于 1 KiB/s 时中止。最坏情况 3 次尝试加退避约 91 分钟，小于 IssueRun 根决策 D5 的交付
  放弃窗口（默认 2 小时，从第一个交付工作项创建时计），所以推送不会在 Cloud 已放弃之后仍长时间占用沙盒。
- Workspace 分支的快进沿用同一分类、重试与时限，但所有失败都只记日志（D2）。
- JSONL 上传规则不变（根 D4）：授权缺失时 `UploadGrantNeeded` 等待，单对象 3 次失败为 `failed{upload_failed}`。

为什么 `403` 不重试：Git Server 只在 Cloud 当前状态不允许时返回 `403`（运行不在 `delivering`、执行已有结果、沙盒换代等），
同一尝试内重试不会改变状态；是否新建尝试由 Cloud 决定。为什么无法识别的错误归为可重试：误判可重试最多多花两次尝试；
误判永久会把一次可能成功的交付判失败，要等 Cloud 退避后才重来。分类依赖 Git 的输出措辞，措辞变化时退化为可重试。
为什么单次时限不用 `network_timeout_ms`：后者为 clone 的单次网络命令设计（默认 10 分钟），首次交付可能要推送大量对象；
30 分钟加三次尝试仍落在放弃窗口之内。

### D5：恢复：从 Git Server 拉取前序交付 ref，Workspace 本地分支指向前序成果

恢复的位置不变（恢复 D1：定位 checkout、确认插件之后，交还工作负载用户与启动插件之前）。会话输入的 `priorRevision` 为
`{revisionId, finalCommit, baseCommit, gitRepositoryUrl, deliveryRef}`。步骤：

1. 校验输入：`finalCommit`、`baseCommit` 是完整提交 ID，地址符合 D1 的规则且等于本 checkout clone 记录中的仓库地址，ref 在
   `refs/ora/deliveries/` 下。
2. 以 Node 身份建立 `restore` 传输仓库，借用 checkout 的对象；拉取协商只以本 checkout clone 记录中的 `base_commit`（快照提交）
   为起点，使 Git Server 只发送 checkout 尚未拥有的对象。协商不使用 checkout 中 Agent 可写的任何 ref。
3. 以会话执行 ID `git fetch --no-tags <gitRepositoryUrl> <deliveryRef>:refs/ora/prior`；确认取到的是提交且等于 `finalCommit`。
4. 把传输仓库交给工作负载用户，以工作负载身份、沿用交付的受管执行与加固配置执行
   `git fetch --no-tags <传输仓库路径> refs/ora/prior`，只有这一条命令放开 `file` 协议；再确认 `finalCommit` 可解析为提交。
5. `git checkout --force -B <Workspace 分支短名> <finalCommit>`：本地分支就是 clone 时检出的那个分支（clone 决策 D2），
   checkout 没有 origin。之后工作区与 index 等于前序最终提交；clone 的基线提交不变。
6. 若前序的 `baseCommit` 不是本 checkout `base_commit` 的祖先（或本地没有它），在首条提问末尾追加固定格式的英文提示：本次
   运行续接了前序最终提交；它依赖的基础提交不在本 Workspace 快照的历史中（外部仓库的历史被改写）；本地分支指向续接的成果；
   可以用 `ora-git fetch upstream` 查看外部仓库的最新内容并调和。`baseCommit` 是快照提交的祖先（外部仓库只是前进了）时
   不提示。
7. 删除传输仓库。

- 拉取、重试与取入共用 5 分钟时限；Git Server 不可用时在时限内最多尝试 3 次，间隔 5 秒、30 秒，下一次间隔会越过时限时
  不再等待。到期按 `prior_revision_unavailable` 失败。
- Node 停止，或会话在恢复期间收到 `EndSession`、被取消时，恢复在下一条 Git 命令开始前停止，并等正在运行的那条结束；会话
  按相应原因结束，恢复不再写 checkout。
- 恢复不改 `refs/remotes/ora/**` 与仓库配置；不访问任何外部仓库：前序最终提交的完整历史都在 Git Server 上（快照与前序的交付
  ref 都在同一个 Project 仓库）。外部历史被改写不导致恢复失败，第 6 步的提示是 Agent 与用户唯一的信号。
- 长时间的拉取不在中途复查授权（Cloud 交付决策 D5 的已知取舍），由本节的时限约束。

### D6：恢复失败的分类

| 情况 | `detail` |
| --- | --- |
| Git Server 上没有该交付 ref（`couldn't find remote ref`） | `prior_revision_base_unavailable` |
| HTTP `403` 或 `401`、`429` 以外的其他 `4xx`（仓库不存在与无权访问返回相同响应） | `prior_revision_base_unavailable` |
| HTTP `401`（Node 自己的 git-access 身份没有通过认证） | `prior_revision_unavailable` |
| 取到的提交不等于 `finalCommit`，或不是提交 | `prior_revision_unavailable` |
| `5xx`、`429`、网络、TLS、重试耗尽或时限到期 | `prior_revision_unavailable` |
| 输入校验、传输仓库、取入或检出在本地失败 | `prior_revision_unavailable` |

- 两种失败都以 `AGENT_FAILED` 加 `detail` 结束会话，插件从未启动（恢复 D1、D3，不变）。
- 名称保留以免改动契约与 Cloud 的结算规则，含义变为“前序 Revision 的 Git 内容在 Git Server 上永久取不到”：原含义“基础
  提交已从用户远端消失”在 Git Server 持有完整历史后不再出现。
- `401` 只说明 Node 自己的 git-access 材料配错或过期，与前序 Revision 的 Git 内容是否存在无关；归入
  `base_unavailable` 会让一次部署配置错误把前序永久标为不可续接，所以归入可再续的 `unavailable`，在时限内也不重试（同一
  材料重试不会通过）。推送一侧的 `401` 仍按 D4 不重试、以 `push_rejected` 失败：交付失败只影响本次尝试，不改变前序是否可续。
- `403` 也可能来自运行在恢复期间被取消；Cloud 在结算时对已请求取消的运行不标记不可续接（Cloud 交付决策 D7），所以 Node
  不需要区分。
- 不等于 `finalCommit` 只可能是缺陷（交付 ref 只能新建），按可重试的 `unavailable` 处理，不因一次缺陷永久放弃该 Revision。

### D7：Cloud 模式必须配置 `git_access`；能力声明

- `ServiceConfig` 的顶层 `git_access`（`ca_cert`、`client_cert`、`client_key`）在配置了 `control.target` 时必填；缺失时 Node
  拒绝启动，错误只给有限代码 `git_access_required`。文件校验沿用现有规则（绝对路径、可读、CA 不含私钥块、证书与私钥匹配）。
- 能力：同时配置 `clone` 与 `git_access` 时声明 clone；交付策略与 Git Server 访问都可用时声明 `revision_delivery`；会话与交付
  都可用时声明 `revision_restore`（恢复 D6，规则不变，因此也隐含 `git_access`）。
- cluster 的 Node 配置模板与本 ADR 的实现在同一次发布中启用 `git_access`；git-access 身份的签发见 clone 决策落地顺序第 1 步。

为什么启动时拒绝而不是声明能力：Cloud 不做能力协商（Cloud 交付决策 D6），Cloud 模式的 Node 缺少证书就永远无法 clone 与
交付；在启动时失败能立即暴露配置错误，而不是在一整次 Agent 会话之后才以交付失败出现。

### D8：Node 协议与 Controller 的相应修改

- `DeliverRevision`：删除 `bundle_key`，增加 `git_repository_url`（`https`，不带用户信息、查询与片段）、`delivery_ref`（在
  `refs/ora/deliveries/` 下）与 `workspace_ref`；`revision_ref` 仍须在 `refs/ora/revisions/` 下。交付的 `PriorRevisionCommit`
  不变，只有 `{revision_id, final_commit}`。
- 会话的 `PriorRevision`：删除 `bundle`，增加 `git_repository_url`、`delivery_ref`、`base_commit`。
- `RevisionDelivered` 删除 `bundle`；`RevisionFailureCode` 增加 `push_rejected`、`push_failed`，`bundle_failed` 不再出现在新结果中。
- `UploadGrantNeeded` 的 `checksums` 只含 JSONL 的键；删除 `DownloadGrantNeeded` 与 `DownloadGrant`。
- Controller 删除下载授权中继；原样转交交付输入中的仓库地址、交付 ref 与 `workspace_ref`，交付输入中的前序若带拉取位置或
  基础提交则拒绝；交付终态核对 `base_commit`、`revision_ref` 与 `history.key` 等于派发输入，`unchanged` 的最终提交等于基础
  提交或前序最终提交。交付 ref 与最终提交的对应关系只由 Cloud 经 ref 查询核验，Controller 不访问 Git Server（执行授权决策 D9）。

Node 协议的消息拒绝未知字段，新旧 Node 与 Controller 不能混用，这与破坏性切换一致。

## 不变量

1. git-access 私钥只被 Node 身份的 Git 进程经命令行配置引用；不写入任何仓库配置、日志、journal 或结果，工作负载身份的进程
   从不获得它。
2. 带 git-access 证书的 Git 进程不读取 checkout 的仓库配置、钩子或 ref；传输仓库中没有凭据。
3. 每次交付（包括 `unchanged`）在发送成功结果之前，交付 ref 已在 Git Server 上指向冻结的最终提交。
4. 同一交付执行在重启后推送相同的最终提交，重放推送不失败。
5. 交付只推送输入给出的交付 ref 与 `workspaceRef`，不推送其他 ref 或 tag，从不推送 `refs/ora/revisions/*`。
6. `workspaceRef` 的快进结果不改变交付结果。
7. 交付不生成 bundle，上传的只有会话 JSONL；`403` 与 ref 更新被拒不重试，可重试的失败有尝试上限，每次推送尝试有时限。
8. 恢复不访问任何外部仓库；恢复出的前序最终提交来自 Git Server 的交付 ref，且等于会话输入中的最终提交。
9. 恢复成功后 HEAD 是 Workspace 本地分支，指向前序最终提交，工作区干净；`refs/remotes/ora/**`、仓库配置与 clone 基线不变。
10. 恢复失败时 Agent 插件从未启动。
11. 配置了 `control.target` 而缺少 `git_access` 的 Node 不会启动。

## 为什么不是这些替代方案

| 方案 | 取舍 |
| --- | --- |
| 以 Node 身份直接在 checkout 中推送与拉取 | 最简单；但会读取 Agent 写过的仓库配置，以 Node 身份执行任意命令或改写目标（不变量 2） |
| 以工作负载身份运行，私钥经继承的文件描述符传入 | 不需要传输仓库；但私钥进入工作负载身份的进程（不变量 1） |
| 传输仓库放在 Node home | 与其他 Node 私有状态放在一起；但工作负载用户无法按路径从中取入对象，Node home 也不能为此对工作负载开放（D1） |
| 恢复时以 Node 身份直接拉取进 checkout | 少一步取入；同第一条，且会以 Node 身份写入工作负载拥有的对象库 |
| `unchanged` 不推送 | 少一次推送；需要恢复一跳引用，Cloud 也无从核验（D2） |
| 交付 ref 与 `workspaceRef` 一次原子推送 | 两者总是一致；但 Agent 改写历史时快进被拒会拖垮整个交付（D2） |
| 交付快进前也做本地祖先检查 | 与 `ora-git push` 一致，非快进时少一次往返；但结果只进日志，服务端已拒绝非快进，多一条命令没有可观察的收益（D2） |
| `403` 也重试 | 能吸收偶发拒绝；但 `403` 反映 Cloud 当前状态，重试只延长沙盒占用（D4） |
| 只靠低速中止、不设单次时限，或沿用 `network_timeout_ms` | 大仓库不会被误杀，或少一个配置；但前者可能越过 Cloud 的放弃窗口，后者对首次大推送太短（D4） |
| 浅推送以缩短首推 | 首推更快；但 Git Server 已持有快照历史，浅推送没有收益，还要处理浅仓库 |
| 恢复失败时按 `403` 与 ref 不存在细分原因 | 更精确；Git Server 有意对无权与不存在返回相同响应，Node 拿不到这个区分（D6） |
| 启动时清理旧 Node 的 `revision-restores/` | 不留残余目录；但破坏性切换下 Node 数据随 Workspace 重建，为一个不再使用的目录写一次性清理没有收益 |

## 本决策未解决的问题

- 大仓库首次交付的进度反馈与时限调优。
- 失败分类依赖 Git 输出措辞（与原 `base_unavailable` 的已知限制相同）。
- Agent 其他分支、子模块内容与 LFS 实体的保存（根决策未解决项）；Agent 可经 `ora-git push` 自行推送其他分支。
- 纵深安全加固（契约 ADR 未解决项）：Node 以 Node 身份借用 checkout 对象库时，工作负载可控的对象文件带来的解析与资源消耗
  攻击面；“工作负载与插件进程读不到 git-access 私钥”这个前提。
- 交付前提中插件后代进程可能仍在写 checkout 的缺口（根决策未解决项，不变）。

## 落地顺序

这些步骤属于契约 ADR 落地顺序第 2 步中的内部线，与 clone 决策落地顺序第 2、4 步同时进行。

1. **协议**（desktop `crates/node-protocol`）：D8 的字段增删与校验；锁定 Cloud 的新契约 commit。完成条件：协议测试覆盖交付
   输入中地址、ref 与提交的拒绝规则，失败码含 `push_rejected`、`push_failed`，下载授权消息不再属于协议。
2. **传输仓库与带证书的 Git**（D1）：完成条件：以真实 Git 与带客户端证书的本地 Git Server 替身测试，checkout 中恶意的
   `credential.helper`、`http.<url>.sslVerify`、`url.<base>.insteadOf` 对 Node 身份的 Git 不生效；推送与拉取的每种应答被正确分类。
3. **交付**（D2–D4）：完成条件：`delivered` 与两种 `unchanged` 都推送交付 ref，重放成功；被拒为 `push_rejected`、不可达为
   `push_failed`；Workspace 分支快进被拒时交付仍成功；冻结后重启复用同一计划；授权不落盘。
4. **恢复**（D5、D6）：完成条件：恢复后本地分支名正确、HEAD 等于前序最终提交、`refs/remotes/ora/**` 与配置不变；改写提示只在
   前序基础提交不是快照提交的祖先时出现；交付 ref 缺失、`403` 与 `401` 以外的其他 `4xx` 为 `prior_revision_base_unavailable`，`401` 与其余
   为 `prior_revision_unavailable`；会话结束、取消或时限到期时恢复不再写 checkout，插件未启动。
5. **配置与 Controller**（D7、D8）：Cloud 模式缺 `git_access` 拒绝启动；Controller 删除下载中继、上传授权只请求 JSONL 键、
   交付终态按 D8 核对。
6. **端到端验收与合并**：见下一节，与 Cloud、Controller 的实现一起验收，三个仓库一起合并。

保持不变：快照语义、会话 JSONL 的冻结与上传、交付前提、恢复在 Agent 启动前进行。从第 6 步开始改变：交付只经 Git，不再有
bundle；续作只从 Git Server 拉取；run 的成果同时出现在 Workspace 分支上。

## 验收

端到端验收与 clone、`ora-git` 共用一个栈与脚本，见 clone 决策的“验收”一节。本文承担其中场景 4（交付、Revision 登记、续作
恢复）的 Node 一侧断言：交付后 Git Server 上交付 ref 指向登记的最终提交，`workspaces/<Workspace>/main` 被快进到最终提交（Agent
未推送时新建）；续作运行在 Agent 启动前 checkout 的本地分支指向前序最终提交；替身写入无关快照后续作，首条提问末尾出现改写
提示，写入子提交快照时不出现；S3 兼容服务中没有 bundle 对象。

## 核心测试用例

本 ADR 的核心测试用例见[内部线用例](../../../test-cases/node/revision/git-delivery-push-and-restore.md)。实现尚未合入 main，义务都如实标为 Missing；现有以已实现决策为依据的用例
在实现合入、本 ADR 改为 `implemented` 时再按本文改写或改挂，见该用例文件开头的说明。
