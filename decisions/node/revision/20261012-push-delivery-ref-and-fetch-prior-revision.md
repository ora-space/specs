---
status: superseded
---

# Node 交付把最终提交推送到 Git Server 的交付 ref（无改动也推），恢复从 Git Server 拉取前序交付 ref；Git 访问在 Node 私有传输仓库中进行，Cloud 模式必须配置 `git_access`

> 已被取代：本决策由 [Node 交付推送交付 ref 并快进 Workspace 分支、续作从 Git Server 拉取（20261017）](20261017-push-delivery-ref-and-restore-from-git-server.md) 取代。原因：本决策写在 Git Server 只存交付 ref、Node 仍从外部仓库 clone 的设计上；Git Server 成为 Project 代码中心（[Git Server 根决策](../../cloud/git-server/0-project-code-hub-git-server-over-https.md)）后，owner 决定把本决策整体改为 `superseded`，按新设计重新写取代者，不在原文上续改。当前系统承诺以取代者为准。

Node 交付时仍按根决策生成内部快照与本地 `refs/ora/revisions/<runId>`，但不再生成 bundle：无论有没有改动，都把
最终提交推送到交付输入给出的 Git Server 仓库与 `refs/ora/deliveries/<deliveryWorkId>`，再照旧上传会话 JSONL。
`403` 与 ref 更新被拒不重试，直接以 `push_rejected` 失败；`503` 与网络错误有限次重试，耗尽以 `push_failed` 失败；
单次推送有总时限与低速中止。恢复前序 Revision 时，Node 从 Git Server 拉取会话输入给出的交付 ref，不再申请下载
授权、下载 bundle 或向 `origin` 补取基础提交。带 git-access 证书的 Git 命令以 Node 身份在 Node 私有的传输仓库中
执行，只把 checkout 的对象当数据借用，从不读取 checkout 的仓库配置。Node 在 Cloud 模式（配置了 `control.target`）
下必须配置 `git_access`，缺失时拒绝启动。

本 ADR 处于 `proposed`，继承 [Node 交付 Revision 根决策](0-internal-snapshot-and-incremental-bundle-upload.md) 与
[恢复前序 Revision 后续决策](20261010-restore-prior-revision-before-session.md)，承担
[Git Server 根决策](../../cloud/git-server/0-sandbox-scoped-git-server-over-https.md) D5 在 Node 一侧的落地。交付契约、
核验登记与不兼容规则由 [Cloud Revision 后续决策](../../cloud/revision/20261012-git-server-delivery-and-verified-delivery-refs.md)
承担，本文不重复。涉及 desktop 的 Node（交付、恢复、配置校验、能力声明）与 Node 协议。兼容性边界：与 Cloud
后续决策相同，不兼容，Node 不再理解带 bundle 的输入；Node 本地目录布局用每个执行的传输仓库目录替换恢复的下载
目录，启动时清空。

## 继承与修改

| 前序约定 | 本文处理 |
| --- | --- |
| 根 D1：交付只在会话结束且进程已清理后开始 | 继承，不变 |
| 根 D2：内部快照不经过提交钩子、不改动工作区与分支；`update-ref` 本地 `revision_ref` | 继承，不变 |
| 根 D3：相对基础提交生成并校验累计增量 bundle；无改动不生成 bundle | 修改：不生成 bundle；`delivered`/`unchanged` 只按最终提交判定，两者都推送到交付 ref（D2） |
| 根 D4：上传 bundle 与 JSONL；首个 PUT 前冻结对象字节与声明；授权只在内存 | 修改：只上传 JSONL；首次推送或 PUT 之前冻结最终提交、声明与 JSONL（D3）；授权规则继承 |
| 根 D5：失败码与“首次上传后不重新快照” | 修改：删除 `bundle_failed`，`upload_failed` 只指 JSONL，新增 `push_rejected`、`push_failed`（D4）；重放规则继承并扩展到推送（D3） |
| 根不变量 4（bundle 上传前通过 `verify`）、6（无改动不生成 bundle） | 删除：已没有 bundle；由本文不变量 3、7 取代 |
| 恢复 D1：恢复在 checkout 定位后、Agent 启动前；失败以 `agent_failed` 结束 | 继承，不变 |
| 恢复 D2：经 Controller 申请下载授权，下载并校验 bundle | 修改：删除；改为凭 git-access 证书从 Git Server 拉取（D5） |
| 恢复 D3：读 bundle 头、向 `origin` 补取前置提交、`bundle verify`、取入 bundle、`checkout -B` | 修改：拉取交付 ref 到传输仓库、核对最终提交、以工作负载身份取入 checkout、`checkout -B`；不再访问 `origin`（D5） |
| 恢复 D3：`prior_revision_unavailable` / `prior_revision_base_unavailable` 的判定 | 修改：按 Git Server 的应答重新划分（D6） |
| 恢复 D4：远端历史被改写时在首条提问末尾提示 | 继承；判定改为会话输入中前序的 `base_commit` 是否为 `origin/<默认分支>` 的祖先（D5） |
| 恢复 D5：最终提交等于基础提交或前序最终提交时 `unchanged` | 继承判定；`unchanged` 也推送（D2） |
| 恢复 D6：会话与交付都可用时声明 `revision_restore` | 继承；交付可用改为要求 `git_access`（D7） |
| [clone 最小执行契约](../repository/20260918-minimal-clone-execution-contract.md)：Git 以受管执行与加固配置、以工作负载身份运行 | 继承；唯一例外是带 git-access 证书的推送与拉取，以 Node 身份在私有传输仓库中运行（D1） |
| Git Server 根 D5：证书与执行 ID 只在当次命令中传入，不写配置、日志或 journal，不新增 remote | 继承（D1） |
| Git Server 根 D2：Node 配置启用 Git Server 时才签发 git-access 身份 | 修改（Node 一侧）：Cloud 模式必须启用（D7） |
| [Node 协议](../protocol/20260928-streamed-thread-events-and-session-commands.md) D5：`UploadGrant`/`UploadGrantNeeded`；恢复决策增加的下载授权消息 | 修改：上传授权只涉及 JSONL；删除下载授权消息（D8） |

## 决策

### D1：带证书的 Git 访问以 Node 身份在私有传输仓库中执行

git-access 私钥在管理 tmpfs 中只有 Node 身份可读，所以推送与拉取必须以 Node 身份运行。但不能以 Node 身份直接
在 checkout 中运行 Git：checkout 的 `.git/config` 由 Agent 写过，Git 会读取其中的 `credential.helper`、`core.askPass`
（以 Node 身份执行任意命令），以及按 URL 匹配的 `http.<url>.*` 与 `url.<base>.insteadOf`（它们优先于命令行上的
通用 `http.*`，能关闭证书校验或改写目标）。因此：

- 每个交付或恢复执行在 Node home 下有一个私有的裸**传输仓库**（`revision-transport/<sha256(execution)>/`，目录
  `0700`，属主 Node）。它的配置只由 Node 写入；它通过 `objects/info/alternates` 把 checkout 的对象库当作只读数据
  借用，不枚举 checkout 的 ref（例如把 `core.alternateRefsCommand` 设为无输出的命令），因此 Node 身份的 Git 进程
  从不读取 checkout 的配置、钩子或 ref。
- 证书、私钥与 CA 只以 `-c http.sslCert=… -c http.sslKey=… -c http.sslCAInfo=…` 传入，执行 ID 以
  `-c http.extraHeader=Ora-Execution-Id: <id>` 传入；另以 `-c credential.helper=`、`-c http.followRedirects=false`、
  `GIT_CONFIG_NOSYSTEM=1`、`GIT_CONFIG_GLOBAL=/dev/null`、`GIT_TERMINAL_PROMPT=0` 与只允许 `https` 的协议策略运行。
  这些值不写入传输仓库或 checkout 的配置，也不进入日志、journal 或结果；Node 只记录有限的分类码。
- 仓库地址与 ref 只取自执行输入：地址必须是 `https`、不带用户信息、查询与片段，ref 必须在
  `refs/ora/deliveries/` 下；按 URL 访问，不新增 remote。交付使用交付执行的 ID，恢复使用会话执行的 ID。
- 传输仓库在执行的终态证据持久化（交付）或恢复结束（无论成败）后删除；Node 启动时清空整个目录。

为什么不以工作负载身份运行并把私钥经继承的文件描述符交给 Git：那样私钥会进入工作负载身份的进程，违背 Git Server
根决策 D5“证书私钥不进入工作负载”。传输仓库让“凭据只在 Node 身份”与“不信任 checkout 的配置”同时成立，代价是
每次执行多建一个小目录。

### D2：交付推送最终提交；无改动也推送

快照（根 D2）完成后，无论结果是什么，Node 都在传输仓库中执行
`git push --porcelain <gitRepositoryUrl> <最终提交>:<deliveryRef>`，然后上传 JSONL：

| 最终提交 | 结果 | 推送 |
| --- | --- | --- |
| 等于基础提交 | `unchanged` | 推送（新 ref 指向基础提交） |
| 等于输入中前序的最终提交 | `unchanged` | 推送（新 ref 指向与前序相同的提交） |
| 其他 | `delivered` | 推送 |

- 成功结果为 `RevisionDelivered{node, final_commit, base_commit, revision_ref, history}` 或
  `RevisionUnchanged{node, final_commit, base_commit, revision_ref, history}`，不带 bundle；交付 ref 是固定输入的一部分，
  结果不重复声明。
- Node 的 checkout 不是浅克隆，第一次向某个 Project 推送会传输基础提交及其全部历史；之后同一 Project 只传输 Git
  Server 尚未拥有的对象，指向已有提交的 `unchanged` 推送不传输对象。

为什么无改动也推送：Revision 能否续作只看它自己的交付 ref（Cloud 后续决策 D7），`unchanged` 也推送就不再需要
“沿用前序 bundle”的一跳引用，Cloud 对两种结果都能做同一种核验。代价是多一次几乎不传数据的推送，以及
Git Server 不可用时 `unchanged` 也会失败。

### D3：冻结计划与重放

- 首次推送或首个 PUT 之前，Node 把最终提交、结果声明与 JSONL 的字节、大小、SHA-256 作为冻结计划写入账本；
  之前的重启重新准备，之后的重启沿用同一计划，不重新快照。最终提交的对象已在 checkout 中，本地
  `revision_ref` 也指向它。
- 顺序为先推送、再上传 JSONL，两者都成功才发送成功结果。推送是更大、更可能失败的一步，先做可以在它确定失败时
  不再等待 JSONL 的上传授权。
- 重放推送同一提交：远端 ref 已指向该提交，`git push` 报告 up to date，按成功处理（Git Server 根决策不变量 11）。
  远端 ref 已存在且指向其他提交时推送被拒，按 `push_rejected` 处理；正常情况下这不会发生，因为只有本执行能新建
  该 ref。
- Controller 断开、Node 关闭与生命周期取消时终止进行中的 Git 进程，不写失败；重启后继续同一执行（根 D5）。

### D4：失败分类、重试与时限

| 情况 | 识别 | 处理 |
| --- | --- | --- |
| HTTP `401`、`403` 或其他 `4xx` | Git 报告服务端返回的状态码 | 不重试，`failed{push_rejected}` |
| ref 更新被拒（`pre-receive` 拒绝、ref 已存在） | `--porcelain` 输出中该 ref 以 `!` 标记 | 不重试，`failed{push_rejected}` |
| HTTP `5xx`（含 `503`）、`429`、连接失败、DNS、TLS 握手失败、低速中止、单次时限到期、无法识别的错误 | 其余情况 | 重试；共 3 次尝试，间隔 5 秒、30 秒；耗尽后 `failed{push_failed}` |
| 本地准备失败（传输仓库无法建立、最终提交不在 checkout 中） | Node 自身 | `failed{snapshot_failed}` 或 `failed{checkout_unavailable}` |

- 单次推送的时限 30 分钟；另以 `http.lowSpeedLimit=1024`、`http.lowSpeedTime=60` 在一分钟低于 1 KiB/s 时中止。
  最坏情况 3 次尝试加退避约 91 分钟，小于 IssueRun 根决策 D5 的交付放弃窗口（默认 2 小时，从第一个交付工作项创建
  时计），所以推送不会在 Cloud 已放弃之后仍长时间占用沙盒。这些值是 proposed 默认值。
- JSONL 上传规则不变（根 D4）：授权缺失时 `UploadGrantNeeded` 等待，单对象 3 次失败为 `failed{upload_failed}`。

为什么 `403` 不重试：Git Server 只在 Cloud 当前状态不允许时返回 `403`（运行不在 `delivering`、执行已有结果、沙盒
换代等），同一尝试内重试不会改变状态；是否新建尝试由 Cloud 决定。为什么无法识别的错误归为可重试：误判可重试
最多多花两次尝试；误判永久会把一次可能成功的交付判失败，要等 Cloud 退避后才重来。分类依赖 Git 的输出措辞，
措辞变化时退化为可重试，不会误判为永久失败。

### D5：恢复：从 Git Server 拉取前序交付 ref

恢复的位置不变（恢复 D1：定位 checkout、确认插件之后，交还工作负载用户与启动插件之前）。会话输入的
`priorRevision` 为 `{revisionId, finalCommit, baseCommit, gitRepositoryUrl, deliveryRef}`。步骤：

1. 校验输入：`finalCommit`、`baseCommit` 是完整提交 ID，地址与 ref 符合 D1 的规则。
2. 以 Node 身份建立传输仓库，借用 checkout 的对象；只写入一个 Node 自己的 ref 指向本次 clone 的基础提交，作为
   拉取协商的起点，使 Git Server 只发送 checkout 尚未拥有的对象。
3. 以会话执行 ID `git fetch --no-tags <gitRepositoryUrl> <deliveryRef>:refs/ora/prior`；确认取到的是提交且等于
   `finalCommit`。
4. 把传输仓库交给工作负载用户（其中没有任何凭据），以工作负载身份、沿用交付的受管执行与加固配置执行
   `git fetch --no-tags <传输仓库路径> refs/ora/prior`，只有这一条命令放开 `file` 协议（与原先取入 bundle 相同）；
   再确认 `finalCommit` 可解析为提交。
5. `git checkout --force -B <默认分支> <finalCommit>`；`origin/<默认分支>` 不动（恢复 D3 第 5 步，不变）。
6. 若 `baseCommit` 不是 `origin/<默认分支>` 的祖先，或本地没有它，按恢复 D4 在首条提问末尾追加提示。
7. 删除传输仓库。

- 拉取与取入共用 5 分钟时限（沿用恢复 D2 的时限）；`5xx`、网络与 TLS 错误在时限内最多重试 3 次。
- 不再访问 `origin`：前序最终提交的完整历史都在 Git Server（Git Server 根决策 D7：首次推送携带完整历史）。
  因此远端历史被改写不再导致恢复失败；第 6 步的提示成为 Agent 与用户唯一的信号，需要 `baseCommit` 才能判断。
- 长时间的拉取不在中途复查授权（Cloud 后续决策 D5 的已知取舍），由本节的时限约束。

### D6：恢复失败的分类

| 情况 | `detail` |
| --- | --- |
| Git Server 上没有该交付 ref（`couldn't find remote ref`） | `prior_revision_base_unavailable` |
| HTTP `401`、`403`（仓库不存在与无权访问返回相同响应） | `prior_revision_base_unavailable` |
| 取到的提交不等于 `finalCommit`，或不是提交 | `prior_revision_unavailable` |
| `5xx`、网络、TLS、重试耗尽或时限到期 | `prior_revision_unavailable` |
| 传输仓库、取入或检出在本地失败 | `prior_revision_unavailable` |

- 名称保留以免改动契约与 Cloud 的结算规则，含义变为“前序 Revision 的 Git 内容在 Git Server 上永久取不到”：原含义
  “基础提交已从用户远端消失”在 Git Server 持有完整历史后不再出现。
- `403` 也可能来自运行在恢复期间被取消；Cloud 在结算时对已请求取消的运行不标记不可续接（Cloud 后续决策 D7），
  所以 Node 不需要区分。
- 不等于 `finalCommit` 只可能是缺陷（交付 ref 只能新建），按可重试的 `unavailable` 处理，不因一次缺陷永久放弃该
  Revision。

### D7：Cloud 模式必须配置 `git_access`

- `ServiceConfig` 的顶层 `git_access`（`ca_cert`、`client_cert`、`client_key`）在配置了 `control.target` 时必填；缺失时
  Node 拒绝启动，错误只给有限代码 `git_access_required`。文件校验沿用现有规则（绝对路径、可读、CA 不含私钥块、
  证书与私钥匹配）。
- 没有 `control.target` 的本地部署与测试中 `git_access` 可选；Node 只在同时配置了 `clone` 与 `git_access` 时声明
  `revision_delivery`，`revision_restore` 的规则（会话与交付都可用）不变，因此也隐含 `git_access`。
- cluster 的两个 Node 配置模板（`fixtures/opencode/node-config.json`、`sandbox-server/node-config.example.json`）在同一次
  发布中启用 `git_access`。这一要求与本 ADR 的实现一起落地，不提前加入只接收证书的 Node 改动，否则在模板启用之前
  合并会让现有栈的 Node 无法启动。

为什么启动时拒绝而不是声明能力：Cloud 不做能力协商（Cloud 后续决策 D6），Cloud 模式的 Node 缺少证书就永远无法
交付；在启动时失败能立即暴露配置错误，而不是在一整次 Agent 会话之后才以交付失败出现。

### D8：Node 协议的相应修改

- `DeliverRevision`：删除 `bundle_key`，增加 `git_repository_url`、`delivery_ref`；`revision_ref` 仍须在
  `refs/ora/revisions/` 下，`delivery_ref` 须在 `refs/ora/deliveries/` 下。
- 会话的 `PriorRevision`：删除 `bundle`，增加 `git_repository_url`、`delivery_ref`、`base_commit`；交付的
  `PriorRevisionCommit` 不变。
- `RevisionDelivered` 删除 `bundle`；`RevisionFailureCode` 删除 `bundle_failed`，增加 `push_rejected`、`push_failed`。
- `UploadGrantNeeded` 的 `checksums` 只含 JSONL 的键；删除 `DownloadGrantNeeded` 与 `DownloadGrant`。

Node 协议的消息拒绝未知字段，新旧 Node 与 Controller 不能混用，这与“同时发布”的前提一致。

## 不变量

1. git-access 私钥只被 Node 身份的 Git 进程经命令行配置引用；不写入任何仓库配置、日志、journal 或结果，工作负载
   身份的进程从不获得它。
2. 带 git-access 证书的 Git 进程不读取 checkout 的仓库配置、钩子或 ref。
3. 每次交付（包括 `unchanged`）在发送成功结果之前，交付 ref 已在 Git Server 上指向冻结的最终提交。
4. 同一交付执行在重启后推送相同的最终提交，重放推送不失败。
5. 交付只推送输入给出的那一个 ref，不推送其他 ref 或 tag。
6. `403` 与 ref 更新被拒不重试；可重试的失败有尝试上限，单次推送有总时限。
7. 交付不生成 bundle，上传的只有会话 JSONL。
8. 恢复不访问 `origin`；恢复出的前序最终提交来自 Git Server 的交付 ref，且等于会话输入中的最终提交。
9. 恢复失败时 Agent 插件从未启动。
10. 配置了 `control.target` 而缺少 `git_access` 的 Node 不会启动。

## 为什么不是这些替代方案

| 方案 | 取舍 |
| --- | --- |
| 以 Node 身份直接在 checkout 中推送与拉取 | 最简单；但会读取 Agent 写过的仓库配置，以 Node 身份执行任意命令或改写目标（不变量 2） |
| 以工作负载身份运行，私钥经继承的文件描述符传入 | 不需要传输仓库；但私钥进入工作负载身份的进程，违背 Git Server 根决策 D5 |
| 恢复时以 Node 身份直接拉取进 checkout | 少一步取入；同第一条，且会以 Node 身份写入工作负载拥有的对象库 |
| `unchanged` 不推送 | 少一次推送；需要恢复一跳引用，Cloud 也无从核验（D2） |
| `403` 也重试 | 能吸收偶发拒绝；但 `403` 反映 Cloud 当前状态，重试只延长沙盒占用（D4） |
| 只靠低速中止、不设总时限 | 大仓库不会被误杀；但推送可能越过 Cloud 的放弃窗口继续占用沙盒（D4） |
| 浅推送以缩短首推 | 首推更快；但之后的拉取与恢复需要连通的历史，Git Server 也要处理浅仓库 |
| 恢复失败时按 `403` 与 ref 不存在细分原因 | 更精确；Git Server 有意对无权与不存在返回相同响应，Node 拿不到这个区分（D6） |

## 本决策未解决的问题

- 大仓库首次推送的进度反馈与时限调优（本文给出的是 proposed 默认值）。
- 失败分类依赖 Git 输出措辞（与原 `base_unavailable` 的已知限制相同）。
- Agent 其他分支、子模块内容与 LFS 实体的保存（根决策未解决项）。
- 工作负载能读到 git-access 私钥的既有缺口（Git Server 根决策 D5），由工作负载降权解决。

## 核心测试用例

按仓库规则，核心测试用例在本 ADR 状态变为 `approved` 后补充到 `test-cases/node/revision/`：推送 `delivered` 与两种
`unchanged`；重启后重放推送不失败；`403` 与 ref 被拒不重试、`503` 重试后成功与耗尽；单次时限与低速中止；checkout
中恶意的 `credential.helper`、`http.<url>.sslVerify`、`url.<base>.insteadOf` 不生效；恢复成功、交付 ref 不存在与 `403`
为 `base_unavailable`、提交不符与网络失败为 `unavailable`、远端改写提示；Cloud 模式缺 `git_access` 拒绝启动。

## 落地

desktop：Node 协议与 Controller 的契约锁定（见 controller-integration 后续决策）、交付推送与冻结计划、恢复拉取、
`git_access` 必填与能力声明；以真实 Git 与本地 HTTPS Git 服务（或 Git Server 程序）测试上节各项。cluster：两个 Node
配置模板启用 `git_access`，`revision:acceptance`、`revision:resume:acceptance` 走 Git。与 Cloud 后续决策同时发布。
