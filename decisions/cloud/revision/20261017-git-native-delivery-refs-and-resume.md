---
status: approved
---

# Revision 的代码改动只经 Git Server 交付：交付输入固定仓库地址、交付 ref 与 Workspace 分支，Cloud 经 ref 查询核验后登记交付 ref；续作只从前序自己的交付 ref 恢复；删除 bundle 与下载授权

从本决策落地起，Agent 运行的交付只有一种形态：Node 把最终提交推送到 Project 仓库中本次交付尝试专属、只能新建的
`refs/ora/deliveries/<deliveryWorkId>`，顺带把它快进到本 Workspace 的分支 `workspaceRef`，会话 JSONL 照旧上传对象存储。
Cloud 在放出交付工作项时把仓库地址、交付 ref 与 `workspaceRef` 写进固定输入，Controller 原样转交。Node 报告结果后，Cloud
先比对输入，再在事务外经 Git Server 的 ref 查询确认交付 ref 指向声明的最终提交、经对象存储 `HEAD` 确认 JSONL，最后在
事务内登记 Revision 并记下交付 ref。无改动的运行同样推送并登记自己的交付 ref。续作时，前序 Revision 以仓库地址、交付
ref 与基础提交交给 Node 拉取。Revision 只记录交付 ref 与最终提交：Git Server 上不存在 `refs/ora/revisions/*`，
`revisionRef`（`refs/ora/revisions/<runId>`）只是 Node 在 checkout 里写的本地 ref。代码 bundle 的对象键、bundle 的上传
授权与下载授权全部删除。

本 ADR 已经 owner 批准，处于 `approved`（核心测试用例见[内部线用例](../../../test-cases/cloud/revision/git-native-delivery-refs-and-resume.md)；实现合入、端到端验收通过后改为 `implemented`），继承 [Cloud Revision 根决策](0-cloud-owned-object-store-and-verified-revisions.md)，同时修改
[IssueRun 续接决策](../issue-run/20261010-resume-run-from-latest-issue-revision.md) 的选取与沿用规则。它取代
[Git Server 交付与已核验交付 ref 决策](20261012-git-server-delivery-and-verified-delivery-refs.md)：那份决策写在 Git Server
只存交付 ref 的设计上，随 Git Server 成为 Project 代码中心（[Git Server 根决策](../git-server/0-project-code-hub-git-server-over-https.md)）整体作废，本文按新设计重新写出当前
决策，不需要再读旧文。Git Server 一侧的 ref namespace、规则集授权与 ref 查询见
[Git Server 代码中心决策](../git-server/20261016-project-repository-as-code-hub-and-ref-contract.md)（下称“契约 ADR”）；
Node 的推送与恢复见 [Node 交付推送与 Git 恢复决策](../../node/revision/20261017-push-delivery-ref-and-restore-from-git-server.md)
（下称“Node 交付决策”）；Internal Control 的消息形状与 Git Server 的写入规则集见
[快照 clone 输入与执行授权决策](../controller-integration/20261016-snapshot-clone-input-and-execution-ref-grants.md)（下称
“执行授权决策”）。涉及 Cloud 的交付工作项、核验登记、续作选取、Git 授权的运行阶段条件、迁移、配置与公开投影；cluster
的 Compose 配置与端到端验收。

兼容性边界：**破坏性切换**（契约 ADR 范围边界）。Cloud 一律派 Git 交付，不设部署开关，不做 Node 能力协商，不迁移数据；
cloud、desktop、cluster 同时发布。升级前只有 bundle 的 Revision 行保留不改，但不再参与续作；开发环境的数据在切换后
重建，不承诺可用。本文不决定对象存储是否下线（会话 JSONL 仍在其中），不处理 Git Server 中 ref 的回收（契约 ADR 未解决项）。

## 继承与修改

| 前序约定 | 本文处理 |
| --- | --- |
| 根 D1：对象存储是 Cloud 的启动配置，凭据只在 Cloud | 继承；对象存储只存会话 JSONL。新增 `git_server` 配置段（D2） |
| 根 D1：缺少 `object_store` 时交付以 `skipped` 直接释放 | 修改：交付需要对象存储与 Git Server 两项配置，缺任一项都 `skipped`，先查对象存储，原因分别为 `object_store_unconfigured`、`git_server_unconfigured`（D2） |
| 根 D2：每个交付工作项固定 `.../revision.bundle` 与 `.../session.jsonl` 两个对象键与 `refs/ora/revisions/<runId>` | 修改：只保留 `session.jsonl`；代码改动的位置由仓库地址、交付 ref 与 `workspaceRef` 表达，同样在放出工作项时固定；`revisionRef` 不变，仍是 Node 本地 ref（D1） |
| 根 D2：PUT 只能首次创建（签入 `If-None-Match: *`），412 后仍交 Cloud 校验 | 继承，只作用于 JSONL |
| 根 D3：上传授权不属于执行输入，`GrantRevisionUpload` 随时重新申请 | 修改：只签发 JSONL 的授权；推送与拉取没有授权材料，由 Git Server 按 Cloud 当前状态逐请求判定（D5） |
| 根 D3：续接运行的只读下载授权由 `GrantRevisionDownload` 承担 | 删除：续作从 Git Server 拉取（D7） |
| 根 D4：先比对输入，事务外核验对象，事务内登记；矛盾声明在接管前拒绝 | 继承结构；事务外核验改为 ref 查询加 JSONL 的 `HEAD`（D3） |
| 根 D4 的“实际决策与偏离”：错误码按声明字段归类 | 继承：`history` 的键与形状为 `INVALID_INPUT`，其余矛盾为 `CONFLICT`；bundle 一侧的归类随 bundle 删除（D3） |
| 根 D4：`revisions` 表字段；`unchanged` 没有 bundle | 修改：增加 `delivery_ref`，形状约束改为互斥的两种形态；新行一律不带 bundle，`unchanged` 也有交付 ref（D4） |
| 根 D5：公开读取只暴露元数据 | 继承；`changed` 改由 Cloud 核验结论得出，Git Revision 的 `bundleSize` 为 `null`（D8） |
| 根不变量 2：Node 只能写入 Cloud 指定的对象键 | 继承并扩展：在 Git Server 上只能新建 Cloud 指定的那一个交付 ref，并快进本 Workspace 的分支（契约 ADR D4、执行授权决策 D5） |
| 根不变量 3：Revision 只在 Cloud 亲自校验对象后登记 | 修改：亲自校验交付 ref 与 JSONL（不变量 2） |
| 根不变量 5：无改动的运行有 Revision 且没有 bundle | 修改：有 Revision、有自己的交付 ref、没有 bundle（不变量 4） |
| 根不变量 6：未配置对象存储时交付记为 `skipped` | 修改：未配置对象存储或 Git Server 时都记为 `skipped`（不变量 10） |
| 契约 ADR D3：每个沙盒请求按证书目标、声明的执行与 Cloud 当前状态解析为规则集；执行类型的映射由 controller-integration 后续 ADR 给出 | 继承；本文给出交付与会话执行的运行阶段条件（D5），规则集的写入映射在执行授权决策 D5 |
| 契约 ADR D4：交付 ref 只能新建；不新增 `refs/ora/revisions/*`，Revision 记录交付 ref 与提交 | 继承（D1、D4） |
| 契约 ADR D5 第 1 项：`workspaceRef` 是 Workspace 自己提交的去处 | 继承；交付输入携带它，Node 在交付时快进它（D1） |
| 契约 ADR D6：写入者写 ref，Cloud 事务外查询、事务内登记；Cloud 不写 Git Server | 继承（D3） |
| [IssueRun 续接决策](../issue-run/20261010-resume-run-from-latest-issue-revision.md) D1：候选是带 bundle（自有或沿用）的最近 Revision；跳过原因为 `project_changed`、`resume_refused` | 修改：候选是有交付 ref、且最终提交不等于基础提交的最近 Revision；新增跳过原因 `repository_changed`（D7） |
| IssueRun 续接决策 D2：恢复后本地默认分支指向前序最终提交，`origin/<默认分支>` 不动 | 修改：checkout 没有 origin；本地分支是 Workspace 分支短名（Node 交付决策 D5） |
| IssueRun 续接决策 D3：`prior_revision_base_unavailable` 表示基础提交从远端消失，标记不可续接 | 修改：改为前序交付 ref 在 Git Server 上取不到；仍标记不可续接，但运行已请求取消时不标记（D7） |
| IssueRun 续接决策 D5：续接后没有新提交时沿用前序 bundle，`bundle_revision_id` 一跳 | 修改：删除沿用与一跳；每个 Git Revision 都有自己的交付 ref（D7） |
| IssueRun 续接决策不变量 6：可续接的 Revision 一跳找到 bundle | 修改：可续接的 Revision 有自己的交付 ref，指向它的最终提交（不变量 6） |
| [IssueRun 根决策](../issue-run/0-agent-run-in-disposable-isolated-workspace.md) D5：交付失败重试新工作项，按放弃窗口收敛 | 继承，不变；Node 的推送时限按放弃窗口取值（Node 交付决策 D4） |
| [Agent 运行执行决策](../controller-integration/20260928-agent-run-executions-thread-and-upload-grants.md) 不变量 1：`v1` 已发布的消息与方法不变 | 修改：随本组破坏性切换删除 bundle 字段与下载授权，形状见执行授权决策 D9 |

## 决策

### D1：交付工作项的固定输入写入仓库地址、交付 ref 与 `workspaceRef`，不再有 bundle 对象键

Cloud 在放出交付工作项（`deliveryWorkInput`）时固定以下服务端位置：

```json
{
  "kind": "deliver_revision",
  "sessionExecutionId": "…",
  "checkoutExecutionId": "…",
  "baseCommit": "<40 或 64 位十六进制>",
  "revisionRef": "refs/ora/revisions/<runId>",
  "historyKey": "revisions/<tenantId>/<runId>/<deliveryWorkId>/session.jsonl",
  "gitRepositoryUrl": "https://<git-server>/<tenantId>/<projectId>.git",
  "deliveryRef": "refs/ora/deliveries/<deliveryWorkId>",
  "workspaceRef": "refs/heads/workspaces/<workspaceId>/<branch>",
  "priorRevision": { "revisionId": "…", "finalCommit": "…" }
}
```

- `gitRepositoryUrl` 由 `git_server.repository_base_url`（D2）加运行 Workspace 的 `/<tenantId>/<projectId>.git` 拼成；
  `deliveryRef` 是 `refs/ora/deliveries/` 加交付工作项 ID；`workspaceRef` 逐字取自该运行 checkout 的 clone 执行输入
  （clone 已成功、输入中有 `workspaceRef`）。`baseCommit` 必须等于运行 Workspace 的基线提交。这些值与 `historyKey` 一样在
  插入工作项时写入，`RecordDispatch` 校验 Controller 转交的输入与之逐字相同。`bundleKey` 不再生成。
- `revisionRef` 不变，仍是 `refs/ora/revisions/<runId>`：它是 Node 在 checkout 里用 `update-ref` 写的本地 ref，只存在于
  checkout，从不推送。Git Server 上没有 `refs/ora/revisions/*`（契约 ADR D4），Revision 的远端事实只有交付 ref 与提交。
- Git Server 授权写入时，允许新建的交付 ref 只取自这份固定输入，并且必须等于 `refs/ora/deliveries/<该工作项 ID>`；
  允许快进的 `workspaceRef` 也取自这份输入，且必须在证书所属 Workspace 的分支前缀下（D5）。
- 交付工作项输入只含地址与 ref，不含任何凭据；推送凭据是 Node 的 git-access 证书，授权在 Git Server。

为什么由 Cloud 写完整地址，而不是 Controller 或 Node 配置 Git Server 地址：沙盒可达的 Git Server 地址是部署事实，与对象
存储签发 URL 所用的 `public_endpoint` 同一模式；仓库路径要用到 Workspace 所属的 Project，这是 Cloud 的业务事实。写进固定
输入后，地址与 ref 受 `RecordDispatch` 的逐字校验保护，重传同一执行时输入不变，Controller 不需要新配置。若在工作项等待
期间改了 Git Server 地址，该次尝试以旧地址推送失败，按 IssueRun 根决策 D5 的重试放出新工作项，新工作项使用新地址。

为什么 `workspaceRef` 从 clone 输入取而不重新拼：它在 clone 步骤冻结时就已确定（[clone 步骤消费快照决策](../operation/20261016-clone-step-consumes-binding-snapshot.md) D1、D3），Node
也会核对交付输入中的值等于自己 clone 记录中的值（Node 交付决策 D2）；从同一来源复制，两边不会因拼法不同而分叉。

### D2：Cloud 配置 `git_server` 段；交付需要对象存储与 Git Server 两项配置

`git_server` 段的每个字段都可由 `CLOUD_GIT_SERVER_<KEY>` 环境变量覆盖：

| 字段 | 含义 |
| --- | --- |
| `repository_base_url` | 沙盒可达的仓库监听基地址，只允许 `https`，不带用户信息、查询、片段与结尾斜杠；写进 clone、会话与交付输入。为空表示没有 Git Server |
| `ref_query_url` | 只对 Cloud 开放的 ref 查询监听（契约 ADR D6），`https`，经默认网络访问 |
| `ca_file` | 校验 Git Server 服务端证书的 CA |
| `client_cert_file`、`client_key_file` | Cloud 查询 ref 的固定客户端身份，与 Controller 访问 Cloud 的身份分开；私钥只以文件引用出现，不进入配置值、日志或数据库 |
| `ref_query_timeout` | 单次 ref 查询的时限，缺省 10 秒 |

- `repository_base_url` 非空时，其余四个文件与地址都必填；地址形状不合法、证书或 CA 不能加载时拒绝启动。启动时不访问
  网络：Git Server 不可达、证书不匹配只让核验返回不可用（D3），或让推送失败并按 IssueRun 根决策 D5 重试。ref 查询的客户端
  只用 TLS 1.3、不跟随重定向。
- 交付需要 `object_store`（JSONL）与 `git_server`（代码改动）两项配置。运行进入 `delivering` 时任一缺失，Cloud 不放出交付
  工作项，在同一事务中以 `skipped` 结算并进入释放，原因按先对象存储、后 Git Server 的顺序记为
  `object_store_unconfigured` 或 `git_server_unconfigured`，每个运行只记录一次（`revision_delivery_skips`）。这延续根决策
  D1“不把未保存伪装成已保存”的规则。没有 Git Server 时 clone 步骤同样不能进行（clone 步骤消费快照决策 D1），所以这条
  `skipped` 只在 Workspace 创建之后才取消 Git Server 配置时出现。
- 这不是“bundle 或 Git”的开关：配置齐全时只有 Git 交付一种形态，没有可选的回退路径。

### D3：核验与登记：比对输入 → 事务外 ref 查询与 JSONL `HEAD` → 事务内登记

Node 报告 `RevisionDelivered{final_commit, base_commit, revision_ref, history}` 或
`RevisionUnchanged{final_commit, base_commit, revision_ref, history}` 后，Cloud 按以下顺序处理：

1. **比对**（`validateDeliveryDeclaration`），在接管之前、不访问 Git Server 与对象存储、不写任何记录：
   - Node 运行身份、`baseCommit`、`revisionRef` 与固定输入不符，`finalCommit` 格式非法，或结果仍带 bundle：
     `ABORTED`/`CONFLICT`（`result_conflict`）；
   - `history` 的键与输入不符或形状非法：`INVALID_ARGUMENT`/`INVALID_INPUT`（`invalid_result`）；
   - `unchanged` 要求最终提交等于基础提交或等于输入中前序的最终提交，`delivered` 要求二者都不等，否则 `CONFLICT`；
   - 失败结果只接受 `session_not_settled`、`checkout_unavailable`、`snapshot_failed`、`history_unavailable`、`upload_failed`、
     `push_rejected`、`push_failed`；`bundle_failed` 与 Node 自报的 `verification_failed` 被拒绝。
2. **事务外核验**（`prepareRevision`）：
   - 调用 ref 查询 `GET /internal/v1/refs?tenantId=…&projectId=…&ref=<deliveryRef>`：`200 {commit}` 且 `commit` 等于声明的
     最终提交才算一致；`404 ref_not_found` 或提交不同是否定结论。
   - 对 JSONL 照旧 `HEAD` 核对大小与 SHA-256。
   - Git Server 在运行中被取消配置、ref 查询的其他应答、网络失败或超时，以及对象存储不可用：不接管、不确认，返回
     `UNAVAILABLE`（`git_server_unavailable` / `object_store_unavailable`），Controller 稍后重放同一事件。
3. **事务内登记**：再次检查租约、执行身份与输入、运行与当前 Node，写入 `revision_verifications`、`revisions`（含
   `delivery_ref`）与收据，同事务调用 `deliverySettled`。
4. ref 不存在、指向其他提交或 JSONL 不一致：接管原 Node 事件，Cloud 验证结论记 `failed{verification_failed}`，不登记
   Revision，按 IssueRun 根决策 D5 重试新交付；原 Node 声明保留在 `node_executions.result`。

`workspaceRef` 的快进结果不参与核验：交付的成败只看交付 ref 与 JSONL，Workspace 分支是否前进由 Cloud 需要时按需查询
（契约 ADR D6）。

为什么核验结论在登记前后都不会变：交付 ref 只能新建、不能更新或删除（契约 ADR D4），查询到的提交在事务提交时与之后
都不变。为什么 `unchanged` 也要核验 ref：Revision 能否续作只取决于它自己的交付 ref（D7），登记一个从未被确认存在的 ref
会把失败推迟到下一次运行。为什么 Git Server 不可用不判核验失败：与对象存储网络失败同一规则，短暂故障不应把一次已经
成功的推送判为失败，白白消耗一次重试与一段放弃窗口。

为什么继续按字段而不是按性质归类错误码：根决策 D4 记录的实现已按字段归类，Controller 对 `CONFLICT` 与 `INVALID_INPUT`
的处理相同，两者都在接管前拒绝、都不留记录；bundle 删除后只剩 `history` 一类走 `INVALID_INPUT`，不值得为此改变已实现的
错误映射。

### D4：`revisions` 表增加交付 ref，形状约束改为互斥的两种形态；旧行保留不改

- 增加 `delivery_ref text`：新行必填，形如 `refs/ora/deliveries/<规范 UUID>`；旧行为 `NULL`。对非空值建部分唯一索引，作为
  “一个交付尝试一个 ref”的数据库层保护。
- 原有的 `revisions_bundle_shape` 约束换成 `revisions_shape`，两种互斥形态：
  - **bundle 形态**：`delivery_ref IS NULL`，且满足原有三种形状之一（既有行原样通过）；
  - **Git 形态**：`delivery_ref IS NOT NULL` 且形状合法，`bundle_key`、`bundle_size`、`bundle_sha256`、`bundle_revision_id`
    全为空；`final_commit` 可以等于 `base_commit`。
- `revision_ref`（Node 本地 ref）、`history_*`、`repository_url`（登记时 Binding 的外部仓库地址）与 `prior_revision_id`
  继续写入；`prior_revision_id` 只在续作且最终提交不等于基础提交时写入；`bundle_revision_id` 不再写入。bundle 列不删除。
- 旧行不迁移、不改写、不删除，它们引用的 bundle 对象也留在对象存储；清理仍是根决策的未解决项。

为什么不能把旧行改写成 Git 形态：`revisions` 行记录的是 Cloud 当时亲自核验过的事实。旧行核验的是 bundle 对象；改写成
指向某个 ref，等于声称一个从未核验过的 ref 存在。破坏性切换下也没有必要删除：保留旧行让公开读取中历史运行的 Revision
不消失，代价只是约束多一个分支。

### D5：Git Server 授权的运行阶段条件

契约 ADR D3 规定每个沙盒请求都要按 Cloud 当前状态判定；执行授权决策 D5 给出各执行类型能写哪些 ref。本节给出交付执行与
Agent 会话执行在 IssueRun 业务上的有效性条件。共同条件：证书目标是该 Workspace 当前存活的代次，执行属于当前 Node 与
沙盒，执行已登记、尚无结果、未被强停，运行时控制处于绑定本运行且已确认的 maintenance，触发者仍能使用该运行时，Project
有效。

| 执行 | 运行与执行还须满足 | 获授权后 |
| --- | --- | --- |
| 交付执行 | 运行 `phase = delivering`；输入的 `deliveryRef` 等于 `refs/ora/deliveries/<该工作项 ID>` | 读整个 Project 仓库；新建 `deliveryRef`；快进输入中属于本 Workspace 的 `workspaceRef` |
| Agent 会话执行 | 运行 `phase ∈ {starting, running}`，没有取消请求，`thread_state` 不是 `ending` 或 `ended` | 读整个 Project 仓库；写本 Workspace 分支前缀（执行授权决策 D5） |

- 所有 Agent 会话都能访问，不再要求输入带 `priorRevision`：会话要随时经 `ora-git` fetch upstream 与其他 Workspace 的分支
  （[Node 从快照 clone 决策](../../node/repository/20261016-clone-from-git-server-snapshot-and-ora-git.md) D5–D7），恢复前序只是其中一次读取。
- **WorkflowRun 的 Agent 会话第一阶段不授权。** 本节条件只按 IssueRun 的 `phase` 与 `thread_state` 写；WorkflowRun 没有 Thread、
  没有 `phase`，也不交付 Revision，它的会话执行在 Git Server 上一律 `403`，`ora-git` 访问 Git Server 的动词对它返回 `denied`。WorkflowRun 的 Agent
  在 Git 上能写什么、何时有效，属于 workflow 模块的决策，留给后续 ADR（执行授权决策“本决策未解决的问题”）。
- 会话的访问在运行请求取消或 Thread 进入结束时立即失效：此时用户要求停止，或交付即将开始，会话不应再改变 Git Server。
- **长 fetch 不在传输中途复查**，记为已知取舍：已通过判定的 `git-upload-pack` 请求在运行被取消后仍可能传完。它只读本
  Project 的仓库，且是该会话在请求开始时有权读的内容；恢复整体受 Node 的时限约束（Node 交付决策 D5）。写入一侧有
  `pre-receive` 的二次判定，不受此影响。

### D6：只保留 Git 交付，不设开关、不做能力协商

Cloud 一律派 Git 交付，不设部署开关，不新增“Git 交付”能力，Controller 不按能力挑选交付形态。Internal Control 契约
随之删除 `DeliverRevisionSpec.bundle_key`、`PriorRevision.bundle`、`RevisionDelivered.bundle`、`DownloadGrant` 与
`GrantRevisionDownload`，增加仓库地址、交付 ref、`workspaceRef`、前序基础提交与 `PUSH_REJECTED`、`PUSH_FAILED`；
`GrantRevisionUpload` 只签发 `history_key` 的授权。字段号、保留号与 `buf breaking` 的一次性豁免见执行授权决策 D9。

为什么不保留 bundle 路径或按能力回退：两套交付、核验与续作逻辑长期并存，每处都要回答“这是哪种 Revision”；能力不足的
工作项还会在 `ClaimWork` 中排队阻塞后面的工作。Cloud 目前没有用户，desktop 的 `ora-controller` 是唯一使用方，与 Cloud
同时发布（契约 ADR 范围边界）。

### D7：续作只选有交付 ref 且有新提交的 Revision

续作选取（`selectPriorRevision`，IssueRun 续接决策 D1）改为：

1. 候选：同一租户、同一 Issue 的其他运行登记的 Revision 中，有 `delivery_ref` 且最终提交不等于基础提交的那些，按登记时间
   （相同时按 id）取最新一个。没有候选时从头开始，首条提问不带 `resume`。
2. 依次判断是否跳过，第一条成立的原因记入首条提问的 `resume{skippedRevisionId, reason}`：
   - `project_changed`：运行 Workspace 不存在，或候选登记时的 `project_id`、`repository_url` 与本次运行的 Project 及其 Binding
     当前外部仓库地址不同（续接决策 D1 第 2 条，不变）；
   - `resume_refused`：候选被标为不可续接（续接决策 D1 第 3 条，不变）；
   - `repository_changed`：本次 checkout 没有基线提交对应的已成功 clone，没有配置 Git Server，或按当前配置拼出的仓库地址
     不等于本次 checkout 的 clone 输入 `repositoryUrl`。前序只能从本 checkout clone 时的那个 Git Server 仓库拉取；地址变了，
     恢复必然失败，不如从头开始并说明原因。
   跳过时不回退到更早的候选（续接决策 D1 第 3 条的理由不变）。
3. 否则会话输入的 `priorRevision` 为 `{revisionId, finalCommit, baseCommit, gitRepositoryUrl, deliveryRef}`：`gitRepositoryUrl`
   与本次 clone 输入的 `repositoryUrl` 逐字相同，`deliveryRef` 是候选自己的交付 ref，`baseCommit` 是候选自己的基础提交，
   供 Node 判断外部历史是否被改写（Node 交付决策 D5）。交付输入中的 `priorRevision` 仍只有 `{revisionId, finalCommit}`。
   首条提问的 `resume{revisionId, runId, finalCommit, baseCommit}` 不变。

- 沿用与一跳删除：每个 Git Revision（包括续接后没有新提交的）都推送并登记了自己的交付 ref，指向自己的最终提交；选取与
  恢复都不再查 `bundle_revision_id`。
- 为什么筛掉最终提交等于基础提交的 Revision，而不是字面上“有 ref 就选”：这样的 Revision 只含它当时的快照提交，续作它会把
  新 checkout 的本地分支拨回比本次快照更旧的提交。旧规则因为它们没有 bundle 而自然排除，新规则保持同一候选集合。
- 没有交付 ref 的行（升级前只有 bundle 的 Revision）不在候选集合中，不设专门分支，首条提问也不记录跳过：破坏性切换，
  升级前的成果不续作。
- 不可续接标记（续接决策 D3）：会话以 `agent_failed` 加 `prior_revision_base_unavailable` 结束时，结算事务照旧把前序 Revision
  标为不可续接；运行在结算时已有取消请求时不标记。Git Server 对已取消运行的读请求同样拒绝（D5），Node 无法把它与“ref
  不存在”区分（契约 ADR D3 有意对无权与不存在返回相同响应）；不加这一条，一次取消就会让该 Revision 永久不可续作。
- 前序的交付 ref 不会被续作改写：新运行的成果写入它自己的交付 ref 与它自己 Workspace 的分支。

### D8：公开读取保持形状

运行的 `revision` 投影仍为 `{id, baseCommit, finalCommit, changed, bundleSize, historySize, priorRevisionId, createdAt}`：

- `changed` 取自 Cloud 核验结论：`revision_verifications.outcome = delivered` 为 `true`，`unchanged` 为 `false`；没有核验结论的
  旧行按是否有自有 bundle 判断。这与旧定义“本次存了自己的 bundle”在旧行上一致，在 Git 行上表示本次运行产生了新提交。
- Git Revision 的 `bundleSize` 为 `null`；旧行照旧返回大小。
- Issue 页的交付展示不再显示 bundle 大小：有改动时显示已保存的 Revision 与最终提交，并注明含改动。
- 仓库地址、交付 ref、`workspaceRef` 与对象键都不进入公开 API；下载与 diff 预览仍未提供。

### D9：切换前提

- cloud、desktop（Controller 与 Node）、cluster 同时发布：cluster 同一次部署签发 git-access 身份、启用 Node 配置模板中的
  `git_access`、为 Cloud 配置 `git_server` 并接入 Git Server（Node 从快照 clone 决策落地顺序第 1 步）。
- 不迁移进行中的工作项：升级时处于 `delivering` 的运行，其交付输入只有 `bundleKey`，新的编解码与 Node 都无法执行，按
  IssueRun 根决策 D5 的放弃窗口以 `deliveryState = failed` 释放；带旧式 `priorRevision.bundle` 的会话同理。系统不会因此进入
  不一致状态。按契约 ADR 范围边界的破坏性切换，Cloud 目前没有用户，开发环境数据在切换后重建，不为这些运行设迁移路径。

为什么不迁移进行中的工作项：要么同时保留两套编解码与 Node 路径，要么在迁移中替换已经固定的执行输入；前者与破坏性切换
矛盾，后者破坏“输入在放出工作项时固定”的幂等规则。

## 不变量

1. 每个新交付工作项的固定输入恰好含一个 `gitRepositoryUrl`、一个 `deliveryRef = refs/ora/deliveries/<该工作项 ID>` 与取自
   checkout clone 输入的 `workspaceRef`，不含代码 bundle 的对象键；Controller 转交的输入与之逐字相同。
2. Cloud 只在 ref 查询确认交付 ref 指向声明的最终提交、且 JSONL 经 `HEAD` 核对一致之后登记 Revision。
3. ref 不存在或指向其他提交时不登记 Revision，记 `verification_failed`；Git Server 不可用时不接管、不确认，可重放。
4. 每个新登记的 Revision（包括 `unchanged`）都有唯一的 `delivery_ref`，且 bundle 列全为空。
5. 升级前的 Revision 行不删除、不改写。
6. 被选为前序的 Revision 有自己的交付 ref，且最终提交不等于其基础提交；会话输入中前序的仓库地址等于本次 clone 输入的
   仓库地址。
7. Git Server 只在运行处于 `delivering` 时让交付执行新建它的交付 ref；只在运行处于 `starting`/`running`、没有取消请求且
   Thread 未在结束时让会话执行访问仓库。
8. 上传授权只针对会话 JSONL 的键；不存在任何下载授权。
9. 执行输入、数据库与日志中没有 Git Server 访问材料或对象存储凭据，只有地址、ref 与对象键。
10. 缺少对象存储或 Git Server 配置时交付记为 `skipped`，不登记 Revision。
11. Git Server 上不存在 `refs/ora/revisions/*`；`revisionRef` 只在 Node 的 checkout 中。

## 为什么不是这些替代方案

| 方案 | 取舍 |
| --- | --- |
| 保留 bundle 路径，以部署开关或回退兼容旧部署 | 升级平滑；但两套交付、核验、续作逻辑长期并存，`unchanged` 与续作语义分叉（一跳 vs 自有 ref），而唯一使用方本来就同步发布（D6） |
| Node 上报 Git 交付能力，Cloud 按能力选择形态 | 允许新旧 Node 混部；但除了双路径成本，能力不足的工作项会在 `ClaimWork` 中排队阻塞后面的工作 |
| Git Server 上另建 `refs/ora/revisions/*`，由 Cloud 经内部写接口创建 | 每个运行一个稳定名字；但交付 ref 已经不可变，再加一层只是重复，Cloud 写接口违反契约 ADR D6，还要 outbox 与对账（契约 ADR 否决的同一方案） |
| 交付 ref 与 `workspaceRef` 一次原子推送，或把 `workspaceRef` 计入核验 | 两者总是一致；但 Agent 改写本地历史时 Workspace 分支快进会被拒，原子推送会拖垮整个交付（D3、执行授权决策 D6） |
| Controller 或 Node 配置 Git Server 地址 | Cloud 少一项配置；但地址不再受固定输入保护，Controller 还需知道 Project 才能拼路径 |
| 只在 `delivered` 时推送，`unchanged` 不推 | 少一次推送；但续接后没有新提交的 Revision 又要间接引用前序的 ref，`unchanged` 也没有可核验的 Git 事实 |
| 把旧 bundle 导入 Git Server，使旧 Revision 可续作 | 旧成果不丢续作能力；但 bundle 不含基础提交，导入要先从外部仓库取得基础提交，又回到本组决策要去掉的依赖 |
| 删除旧 `revisions` 行 | 表只剩一种形态；但丢掉已核验的历史，公开 API 中这些运行的 Revision 也随之消失（D4） |
| 续作候选按字面“有交付 ref” | 规则更短；会选中停在快照提交的 Revision，把本地分支拨回旧提交（D7） |
| 前序的仓库地址与本次 clone 不同时仍交给 Node | 不需要 `repository_changed`；但 Node 只能访问本 checkout 所属的仓库，恢复必然失败，还会被当作恢复失败让运行失败（D7） |
| ref 查询失败按核验失败处理 | 交付更快收敛；Git Server 短暂不可用就会把成功的推送判失败（D3） |
| Git Server 推送成功后写收据，Cloud 不再查询 | 少一次调用；ref 与收据写入不是原子操作，登记方也不再亲自核验（契约 ADR D6） |

## 本决策未解决的问题

- Git Server 中失败交付尝试留下的 ref 与对象、仓库配额与 GC、删除 Project 时的清理（契约 ADR 未解决项）。
- 旧 bundle 对象与会话 JSONL 的过期清理（根决策未解决项）；对象存储是否在 JSONL 另有去处后下线。
- 同一 Issue 并发运行的协调（续接决策未解决项）。
- Git Server 多副本时 ref 查询与推送之间的一致性（契约 ADR 未解决项）。

## 落地顺序

1. **specs**：本文与 Node 交付决策、执行授权决策、Node 从快照 clone 决策、clone 步骤消费快照决策一起获批。
2. **cloud**（契约 ADR 落地顺序第 2 步中的内部线）：`git_server` 配置与 ref 查询客户端、`deliveryWorkInput`、核验登记、迁移、
   续作选取与 `repository_changed`、`GrantRevisionUpload` 只签 JSONL、Git 授权的运行阶段条件、公开投影与前端展示；契约形状
   随执行授权决策落地。完成条件：以真实 PostgreSQL、真实 Git Server 与 S3 兼容服务的集成测试证明不变量 1–8、10、11，包括：
   `delivered` 与 `unchanged` 都登记交付 ref；ref 缺失、指向别的提交或 JSONL 不符记 `verification_failed`；Git Server 不可达
   不接管、可重放；未配置 Git Server 时只跳过一次；续作候选排除停在基础提交与没有交付 ref 的行；取消的运行不标记不可续接；
   clone 自另一个仓库地址的运行以 `repository_changed` 从头开始；数据库拒绝带 bundle 的 Git 行与重复的交付 ref。
3. **desktop**：锁定新契约 commit；Controller 与 Node 按执行授权决策与 Node 交付决策修改。
4. **cluster 与端到端验收**：下一节。
5. **cloud**：下一个修改 `proto/buf.yaml` 的 PR 移除豁免（执行授权决策 D9）。

第 2–4 步分别评审，但必须同时发布（D9）。保持不变：会话 JSONL 的存放与校验、IssueRun 的阶段与重试、旧 Revision 的公开
读取。从同时发布起改变：新交付的代码改动只在 Git Server，续作只从 Git Server 拉取，旧 Revision 不再被续作。

## 验收

端到端验收与内部线的 clone、`ora-git` 共用一个栈与脚本，见
[Node 从快照 clone 决策](../../node/repository/20261016-clone-from-git-server-snapshot-and-ora-git.md) 的“验收”一节。本文承担的
断言在其中的场景 4（交付、Revision 登记、续作恢复）与执行授权决策验收中的“交付更新 Workspace 分支”“续作的组合”：

- `psql`：每个运行的 `revisions.delivery_ref` 存在、唯一，bundle 列为空；`unchanged` 的运行也有交付 ref；
- Git Server 容器中 `git for-each-ref`：交付 ref 指向登记的最终提交；没有 `refs/ora/revisions/*`；
- 续作运行的会话输入 `priorRevision` 带前序自己的 `deliveryRef` 与 `baseCommit`，`gitRepositoryUrl` 等于 `request.clone.repositoryUrl`；
- S3 兼容服务中只有 `session.jsonl`，没有 `revision.bundle`。

## 核心测试用例

本 ADR 的核心测试用例见[内部线用例](../../../test-cases/cloud/revision/git-native-delivery-refs-and-resume.md)。实现尚未合入 main，义务都如实标为 Missing；现有以已实现决策为依据的用例
在实现合入、本 ADR 改为 `implemented` 时再按本文改写或改挂，见该用例文件开头的说明。
