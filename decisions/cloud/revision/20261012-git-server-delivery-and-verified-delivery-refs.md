---
status: approved
---

# Revision 的代码改动只经 Git Server 交付：Cloud 写入仓库地址与交付 ref，经 ref 查询核验后登记；删除 bundle 与下载授权，proto `v1` 破例删字段

从本决策落地起，Agent 运行的交付只有一种形态：Node 把最终提交推送到 Git Server 上本次交付尝试专属的
`refs/ora/deliveries/<deliveryWorkId>`，会话 JSONL 照旧上传对象存储。Cloud 在放出交付工作项时，把完整仓库地址
（`https://<git-server>/<tenantId>/<projectId>.git`）与交付 ref 写进固定输入，Controller 原样转交。Node 报告结果后，
Cloud 先比对输入，再在事务外经 Git Server 的 ref 查询确认交付 ref 指向声明的最终提交、经对象存储 `HEAD` 确认
JSONL，最后在事务内登记 Revision 并记下交付 ref。无改动的运行同样推送并登记交付 ref。续作时，前序 Revision 以
仓库地址与交付 ref 交给 Node 拉取。代码 bundle 的对象键、bundle 的上传授权与下载授权全部删除；不向后兼容：升级前
只有 bundle 的 Revision 不迁移，行保留，但不在续作候选中。

本 ADR 于 2026-10-12 经 owner 审核通过，状态为 `approved`：规则是 Git Server 根决策落地顺序第 4 步的开发依据，
cloud、desktop、cluster 的实现与验收尚未完成；核心用例见文末“核心测试用例”一节，直接证据逐项登记为 `Covered`、
`Partial` 或 `Missing`。本文是 [Cloud Revision 根决策](0-cloud-owned-object-store-and-verified-revisions.md) 的第一个
后续决策，承担 [Git Server 根决策](../git-server/0-sandbox-scoped-git-server-over-https.md)“对 Revision 链路的影响”中 Cloud
一侧的修改，是这组修改的主 ADR。Node 的推送与恢复见
[node/revision 后续决策](../../node/revision/20261012-push-delivery-ref-and-fetch-prior-revision.md)，契约形状的修改说明见
[controller-integration 后续决策](../controller-integration/20261012-git-delivery-inputs-without-bundle-grants.md)。涉及 Cloud
的 proto、`internal/controlgrpc` 编解码、交付工作项、核验登记、续作选取、Git 授权条件、迁移与配置；desktop 的
Controller 与 Node；cluster 的 Node 配置模板、Compose 中的 Cloud 配置与 Revision 验收。

兼容性边界：**不兼容**。Cloud 一律派 Git 交付，不设部署开关，也不做 Node 能力协商；Internal Control proto `v1`
一次性破例删除字段与方法（D6），cloud、desktop、cluster 必须同时发布（D9）。本文不迁移已有的 bundle Revision，
不删除它们的数据库行与对象；不决定对象存储是否下线（会话 JSONL 仍在其中）；不处理 Git Server 中仓库与 ref 的清理。

## 继承与修改

| 前序约定 | 本文处理 |
| --- | --- |
| 根 D1：对象存储是 Cloud 的启动配置，凭据只在 Cloud | 继承；对象存储只存会话 JSONL。新增 `git_server` 配置段（D2） |
| 根 D1：缺少 `object_store` 时交付以 `skipped` 直接释放 | 修改：交付需要对象存储与 Git Server 两项配置，缺任一项都 `skipped`，原因分别为 `object_store_unconfigured`、`git_server_unconfigured`（D2） |
| 根 D2：每个交付工作项固定两个对象键 `.../revision.bundle` 与 `.../session.jsonl` | 修改：只保留 `session.jsonl`；代码改动的位置由仓库地址与交付 ref 表达，同样在放出工作项时固定（D1） |
| 根 D2：PUT 只能首次创建（签入 `If-None-Match: *`），412 后仍交 Cloud 校验 | 继承，只作用于 JSONL |
| 根 D3：上传授权不属于执行输入，`GrantRevisionUpload` 随时重新申请 | 修改：只签发 JSONL 的授权；推送没有授权，由 Git Server 按执行状态逐请求判定（D5） |
| 根 D4：先比对输入，事务外核验对象，事务内登记；矛盾声明在接管前拒绝 | 继承结构；事务外核验改为 ref 查询加 JSONL 的 `HEAD`（D3） |
| 根 D4：`revisions` 表字段；`unchanged` 没有 bundle | 修改：增加 `delivery_ref`，形状约束改写；`unchanged` 也有交付 ref（D4） |
| 根 D5：公开读取只暴露元数据 | 继承；`changed` 改由 Cloud 核验结论得出，Git Revision 的 `bundleSize` 为 `null`（D8） |
| 根不变量 2：Node 只能写入 Cloud 指定的对象键 | 继承并扩展：也只能新建 Cloud 指定的那一个 ref（由 Git Server 根决策 D4 保证） |
| 根不变量 3：Revision 只在 Cloud 亲自校验对象后登记 | 修改：亲自校验交付 ref 与 JSONL（不变量 2） |
| 根不变量 5：无改动的运行有 Revision 且没有 bundle | 修改：有 Revision、有交付 ref、没有 bundle（不变量 4） |
| Git Server 根 D3 第 5 条：运行阶段条件由 Revision 后续 ADR 列出 | 本文给出（D5） |
| Git Server 根 D4：交付 ref 由 Cloud 生成并写入交付执行的固定输入 | 继承；字段见 D1 |
| Git Server 根 D5：Git Server 地址随执行输入给出 | 继承并明确：仓库地址由 Cloud 按部署配置写入固定输入，Controller 原样转交（D1） |
| Git Server 根 D6：比对 → 事务外 ref 查询 → 事务内登记 | 继承（D3） |
| Git Server 根不变量 12：只在 ref 查询确认后登记 | 本文承担（D3、不变量 2） |
| [IssueRun 续接决策](../issue-run/20261010-resume-run-from-latest-issue-revision.md) D1：候选是带 bundle（自有或沿用）的最近 Revision | 修改：候选是有交付 ref、且最终提交不等于基础提交的最近 Revision，用它自己的交付 ref 续作（D7） |
| IssueRun 续接决策 D3：`prior_revision_base_unavailable` 表示基础提交从远端消失，标记不可续接 | 修改：改为前序交付 ref 在 Git Server 上取不到；仍标记不可续接，但运行已请求取消时不标记（D7） |
| IssueRun 续接决策 D5：续接后没有新提交时沿用前序 bundle，`bundle_revision_id` 一跳 | 修改：删除一跳；每个 Git Revision 都有自己的交付 ref（D7） |
| IssueRun 续接决策不变量 6：可续接的 Revision 一跳找到 bundle | 修改：可续接的 Revision 有自己的交付 ref，指向它的最终提交（不变量 6） |
| [IssueRun 根决策](../issue-run/0-agent-run-in-disposable-isolated-workspace.md) D5：交付失败重试新工作项，按放弃窗口收敛 | 继承，不变；推送时限按放弃窗口取值（node/revision 后续决策 D4） |
| [controller-integration Agent 运行执行决策](../controller-integration/20260928-agent-run-executions-thread-and-upload-grants.md) 不变量 1：`v1` 已发布的消息与方法不变 | 修改：本次一次性破例（D6），此后恢复只增不改 |

## 决策

### D1：交付工作项的固定输入写入仓库地址与交付 ref，不再有 bundle 对象键

Cloud 在放出交付工作项（`deliveryWorkInput`）时固定以下服务端位置：

```json
{
  "kind": "deliver_revision",
  "sessionExecutionId": "…",
  "checkoutExecutionId": "…",
  "baseCommit": "<40 或 64 位十六进制>",
  "revisionRef": "refs/ora/revisions/<runId>",
  "historyKey": "revisions/<tenantId>/<runId>/<deliveryWorkId>/session.jsonl",
  "gitRepositoryUrl": "https://git-server:8445/<tenantId>/<projectId>.git",
  "deliveryRef": "refs/ora/deliveries/<deliveryWorkId>",
  "priorRevision": { "revisionId": "…", "finalCommit": "…" }
}
```

- `gitRepositoryUrl` 由 `git_server.repository_base_url`（D2）加上运行 Workspace 的 `tenantId` 与 `projectId` 拼成；
  `deliveryRef` 是 `refs/ora/deliveries/` 加交付工作项 ID。两者与 `historyKey` 一样在插入工作项时写入，
  `RecordDispatch` 校验 Controller 转交的输入与之逐字相同。`bundleKey` 不再生成。
- `revisionRef` 不变，仍是 Node 本地的 `refs/ora/revisions/<runId>`；它只存在于 checkout，不是远端 ref。
- Git Server 授权写入时，允许新建的 ref 取自这份固定输入（Git Server 根决策 D4 的“只来自该执行的固定输入”）；
  它与 `refs/ora/deliveries/<workId>` 恒等，因为 Cloud 是唯一的生成方。
- 交付工作项输入只含地址与 ref，不含任何凭据；推送凭据是 Node 的 git-access 证书，授权在 Git Server（D5）。

为什么由 Cloud 写完整地址，而不是 Controller 或 Node 配置 Git Server 地址：Git Server 面向沙盒的地址是部署事实，
与对象存储签发 URL 所用的 `public_endpoint` 同一模式，由 Cloud 配置；仓库路径要用到 Workspace 所属的 Project，
这是 Cloud 的业务事实。写进固定输入后，地址与 ref 受 `RecordDispatch` 的逐字校验保护，重传同一执行时输入不变；
Controller 不需要新配置，也不因此获得任何权限。若在工作项等待期间改了 Git Server 地址，该次尝试以旧地址推送
失败，按 IssueRun 根决策 D5 的重试放出新工作项，新工作项使用新地址；这比让地址在派发时漂移更容易推理。

### D2：Cloud 配置 `git_server` 段；交付需要对象存储与 Git Server 两项配置

`git_server` 段的字段均可由 `CLOUD_GIT_SERVER_*` 环境变量覆盖：

| 字段 | 含义 |
| --- | --- |
| `repository_base_url` | 沙盒可达的仓库监听基地址，只允许 `https`，不带用户信息、查询与片段；写进交付与会话输入（D1、D7） |
| `ref_query_url` | 只对 Cloud 开放的 ref 查询监听（Git Server 根决策 D6），经默认网络访问 |
| `ca_file` | 校验 Git Server 服务端证书的 CA |
| `client_cert_file`、`client_key_file` | Cloud 查询 ref 的固定客户端身份，与 Controller 访问 Cloud 的身份分开；私钥不进入配置值、日志或数据库 |
| `ref_query_timeout` | 单次 ref 查询的时限，默认 10 秒 |

- 交付需要 `object_store`（JSONL）与 `git_server`（代码改动）两项配置。放出交付工作项时任一缺失，交付以
  `skipped` 结算，原因按先对象存储、后 Git Server 的顺序记为 `object_store_unconfigured` 或 `git_server_unconfigured`，
  运行直接进入释放，健康检查报告缺失的依赖。这延续根决策 D1“不把未保存伪装成已保存”的规则，让不需要 Revision
  的开发环境不被阻塞。
- 这不是“bundle 或 Git”的开关：配置齐全时只有 Git 交付一种形态，没有可选的回退路径。
- 配置错误（地址不可达、证书不匹配）不阻止启动，只让核验因 Git Server 不可用而不接管（D3），或让推送失败并按
  IssueRun 根决策 D5 重试。

### D3：核验与登记：比对输入 → 事务外 ref 查询与 JSONL `HEAD` → 事务内登记

Node 报告 `RevisionDelivered{final_commit, base_commit, revision_ref, history}` 或
`RevisionUnchanged{final_commit, base_commit, revision_ref, history}` 后，Cloud 按以下顺序处理：

1. **比对**（`validateDeliveryDeclaration`）：`baseCommit`、`revisionRef` 与 `history.key` 必须等于固定输入，
   `finalCommit` 格式合法；`unchanged` 要求最终提交等于基础提交或等于输入中前序的最终提交，`delivered` 要求二者
   都不等。结果中不再有 bundle，带 bundle 的结果无法解码（D6）。不一致在接管前拒绝，不访问 Git Server 与对象
   存储，不写任何记录。拒绝按问题的性质分类，不按字段分类：
   - **与固定输入矛盾**：报告的 Node 不是执行登记的 Node，`baseCommit`、`revisionRef` 或 `history.key` 不等于输入，
     `unchanged` / `delivered` 与最终提交的关系不成立：`ABORTED` / `CONFLICT`（`result_conflict`）。
   - **形状不合法**：`finalCommit` 不是合法提交 ID，`history` 的大小或 SHA-256 形状不合法：`INVALID_ARGUMENT` /
     `INVALID_INPUT`（`invalid_result`）。

   结果不携带交付 ref（Node 推送到输入给出的 `deliveryRef`），所以交付 ref 与输入的一致性不在这一步比对，而由
   第 2 步的 ref 查询确认；查询的否定结论按第 4 步记核验失败。bundle 路线现在按字段分类（`history` 的键与形状
   返回 `invalid_result`，bundle 的键与形状返回 `result_conflict`），本 ADR 落地时统一为上述规则。
2. **事务外核验**（`prepareRevision`）：
   - 调用 ref 查询 `GET /internal/v1/refs?tenantId=…&projectId=…&ref=<deliveryRef>`：`200 {commit}` 且 `commit`
     等于声明的最终提交才算一致；`404 ref_not_found` 或提交不同是否定结论。
   - 对 JSONL 照旧 `HEAD` 核对大小与 SHA-256。
   - ref 查询或对象存储的网络失败、`503`、超时：不接管、不确认，返回不可用（`git_server_unavailable` /
     `object_store_unavailable`），Controller 稍后重放同一事件。
3. **事务内登记**（`settleVerifiedRevision`）：再次检查执行身份、运行与当前 Node（`liveDelivery`），写入
   `revision_verifications`、`revisions`（含 `delivery_ref`）与收据，同事务调用 `deliverySettled`。
4. ref 不存在、指向其他提交或 JSONL 不一致：接管原 Node 事件，Cloud 验证结论记 `failed{verification_failed}`，
   不登记 Revision，按 IssueRun 根决策 D5 重试新交付；原 Node 声明保留在 `node_executions.result`。

为什么核验结论在登记前后都不会变：交付 ref 只能新建、不能更新或删除（Git Server 根决策 D4），查询到的提交在
事务提交时与之后都不变。为什么 `unchanged` 也要核验 ref：Revision 能否续作只取决于它自己的交付 ref（D7），
登记一个从未被确认存在的 ref 会把续作失败推迟到下一次运行。为什么 Git Server 不可用不判核验失败：与对象存储
网络失败同一规则，短暂故障不应把一次已经成功的推送判为失败、白白消耗一次重试与一个放弃窗口。

### D4：`revisions` 表增加交付 ref，形状约束改写；旧行保留不删

- 增加 `delivery_ref text`：新行必填，形如 `refs/ora/deliveries/<uuid>`；旧行为 `NULL`。对非空值加部分唯一索引，
  作为“一个交付尝试一个 ref”的数据库层保护。
- `bundle_key`、`bundle_size`、`bundle_sha256` 本来就没有 `NOT NULL`，现有的形状由迁移 0033 的
  `revisions_bundle_shape` 约束决定：`base_commit = final_commit` 时 bundle 列全空，否则必须有 bundle 或沿用前序。
  新迁移把它换成两种互斥形态：
  - **旧形态**：`delivery_ref IS NULL`，且满足 0033 的三种形状之一（既有行原样通过）；
  - **Git 形态**：`delivery_ref IS NOT NULL`，`bundle_key`、`bundle_size`、`bundle_sha256`、`bundle_revision_id` 全为空；
    `final_commit` 可以等于 `base_commit`。
- `revision_ref`（Node 本地 ref）、`history_*`、`prior_revision_id`（续作且最终提交不等于基础提交时写入，规则不变）
  继续写入；`bundle_revision_id` 不再写入。
- 旧行不迁移、不改写、不删除，它们引用的 bundle 对象也留在对象存储；清理仍是根决策的未解决项。

为什么不能把旧行改写成 Git 形态：`revisions` 行记录的是 Cloud 当时亲自核验过的事实。旧行核验的是 bundle 对象；
把它改写成指向某个 ref，等于声称一个从未核验过的 ref 存在。安全的路径是保留旧行，只让它们不参与续作（D7）。

### D5：Git Server 授权的运行阶段条件

填补 Git Server 根决策 D3 第 5 条。在根决策 D3 的共同条件（证书目标是当前存活代次、执行属于当前 Node/沙盒、
运行 Workspace 的 maintenance 绑定已确认、触发者仍是有效租户成员且账号有效、Project 有效）之上：

| 执行 | 权限 | 运行与执行须满足 |
| --- | --- | --- |
| 交付执行 | 读写；只能新建固定输入中的 `deliveryRef` | 已登记、尚无结果、未被强停；运行 `phase = delivering` |
| 会话执行 | 只读 | 输入带 `priorRevision`；已登记、尚无结果、未被强停；运行 `phase ∈ {starting, running}`，没有取消请求，`thread_state` 不是 `ending` 或 `ended` |
| 其他执行（不带前序的会话、clone、插件） | 无 | — |

- 只有带前序的会话需要读：其余会话从用户上游 clone，没有理由访问 Git Server。
- 会话的读权限在运行请求取消或 Thread 进入结束时立即失效：恢复只在 Agent 启动前进行，此时用户要求停止，就没有
  继续恢复的意义。
- **长 fetch 不在传输中途复查**，记为已知取舍：已通过判定的 `git-upload-pack` 请求在运行被取消后仍可能传完。
  它只读本 Project 的仓库，且是该会话在请求开始时有权读的内容；恢复整体受 Node 的时限约束。写入一侧有
  `pre-receive` 的二次判定，不受此影响。

### D6：只保留 Git 交付；proto `v1` 破例删除字段与方法，不开 `v2`

Cloud 一律派 Git 交付，不设部署开关，不新增“Git 交付”能力，Controller 不按能力挑选交付形态。

`ora/cloud/internal/v1` 的修改：

| 位置 | 修改 |
| --- | --- |
| `agent_executions.proto` `DeliverRevisionSpec` | 删除 `bundle_key`（`reserved 5; reserved "bundle_key";`）；增加 `git_repository_url = 8`、`delivery_ref = 9` |
| `agent_executions.proto` `PriorRevision` | 删除 `bundle`（`reserved 3; reserved "bundle";`）；增加 `git_repository_url = 4`、`delivery_ref = 5`、`base_commit = 6`（三者只在会话输入中设置，见 D7） |
| `agent_executions.proto` `RevisionDelivered` | 删除 `bundle`（`reserved 4; reserved "bundle";`） |
| `agent_executions.proto` `RevisionFailureReason` | 增加 `PUSH_REJECTED = 8`、`PUSH_FAILED = 9`；`BUNDLE_FAILED` 保留枚举值，但新结果中 Cloud 不再接受 |
| `agent_executions.proto` `DownloadGrant` | 删除消息 |
| `agent_runs.proto` `AgentRunService` | 删除 `GrantRevisionDownload` 及 `GrantRevisionDownloadRequest`、`GrantRevisionDownloadResponse`；`GrantRevisionUpload` 形状不变，只签发 `history_key` 的授权，`checksums` 只能含该键 |

`buf breaking` 使用 `FILE` 规则：即使写了 `reserved`，`FIELD_NO_DELETE` 仍会拒绝删字段；删除消息与方法另会触发
`MESSAGE_NO_DELETE` 与 `RPC_NO_DELETE`。本次变更的 PR 在 `proto/buf.yaml` 中只针对这两个文件、只对这三条规则豁免，
下一个 PR 移除豁免（此后比较基准已包含本次删除，检查自然通过）：

```yaml
breaking:
  use:
    - FILE
  ignore_only:
    FIELD_NO_DELETE:
      - ora/cloud/internal/v1/agent_executions.proto
    MESSAGE_NO_DELETE:
      - ora/cloud/internal/v1/agent_executions.proto
      - ora/cloud/internal/v1/agent_runs.proto
    RPC_NO_DELETE:
      - ora/cloud/internal/v1/agent_runs.proto
```

为什么不开 `v2`：`buf.yaml` 约定破坏性修改以并列的 `v2` 表达，是为了让旧使用方继续工作。这里的前提不成立：
desktop 的 `ora-controller` 是唯一使用方，按 commit 锁定契约、与 Cloud 同时发布；而且本次修改本身就不兼容，旧
Controller 拿不到仓库地址与交付 ref，无论 `v1` 是否保留都无法驱动 Git 交付。开 `v2` 意味着 Cloud 同时注册两套
服务与编解码，却没有任何使用方会再调用 `v1`。`reserved` 仍然写上，防止字段号与名字以后被赋予别的含义；本次
破例只在本 ADR 记录，此后 `v1` 恢复只增不改。

为什么也删 `RevisionDelivered.bundle` 与 `DownloadGrant`：只保留 Git 交付后它们没有任何合法取值；留着一个“必须
为空”的字段或一个无人引用的消息，会让读契约的人以为还存在 bundle 路径。

### D7：续作只选有交付 ref 且有新提交的 Revision

续作选取（`selectPriorRevision`，IssueRun 续接决策 D1）改为：

1. 候选：同一 Issue 的其他运行登记的 Revision 中，有 `delivery_ref` 且最终提交不等于基础提交的那些，按登记时间
   取最新一个。没有候选时从头开始。
2. Project 或仓库地址不同、或被标为不可续接：从头开始（续接决策 D1 第 2、3 条，不变）。
3. 否则会话输入的 `priorRevision` 为
   `{revisionId, finalCommit, baseCommit, gitRepositoryUrl, deliveryRef}`：地址按当前 `git_server.repository_base_url`
   与候选的 tenant、Project 拼成，`deliveryRef` 是候选自己的交付 ref，`baseCommit` 供 Node 判断远端历史是否被改写
   （node/revision 后续决策 D5）。交付输入中的 `priorRevision` 仍只有 `{revisionId, finalCommit}`。

- 一跳逻辑删除：每个 Git Revision（包括续接后没有新提交的）都推送并登记了自己的交付 ref，指向自己的最终提交；
  选取与恢复都不再查 `bundle_revision_id`。
- 为什么筛掉最终提交等于基础提交的 Revision，而不是字面上“有 ref 就选”：这样的 Revision 只含它当时的基础提交，
  续作它会把新 checkout 的默认分支拨回比本次 clone 更旧的提交，丢掉远端的进展。旧规则因为它们没有 bundle 而
  自然排除，新规则保持同一候选集合：凡最终提交不等于基础提交的 Revision，旧规则下都有自有或沿用的 bundle。
- 没有交付 ref 的行（升级前只有 bundle 的 Revision）不在候选集合中，不设专门分支，首条提问的 `resume` 也不记录
  跳过：不向后兼容，升级前的成果不续作，续作只从升级后登记的 Revision 开始。
- 不可续接标记（续接决策 D3）：会话以 `agent_failed` 加 `prior_revision_base_unavailable` 结束时，结算事务照旧标记
  前序 Revision 不可续接；但运行在结算时已有取消请求时不标记。Git Server 对已取消运行的读请求同样返回 `403`
  （D5），Node 无法把它与“仓库或 ref 不存在”区分（Git Server 根决策 D3 有意返回相同响应），不加这一条，一次取消
  就会让该 Revision 永久不可续作。

### D8：公开读取保持形状

运行的 `revision` 投影仍为 `{id, baseCommit, finalCommit, changed, bundleSize, historySize, priorRevisionId, createdAt}`：

- `changed` 取自 Cloud 核验结论：`revision_verifications.outcome = delivered` 为 `true`，`unchanged` 为 `false`。
  这与旧定义“本次存了自己的 bundle”在旧行上一致，在 Git 行上表示本次运行产生了新提交。
- Git Revision 的 `bundleSize` 为 `null`；旧行照旧返回大小。
- 前端随之修改：cloud 中 Issue 页的交付展示（`frontend/src/features/issues/thread/run-delivery.tsx`）现在把有改动的交付显示为
  “含改动，bundle N B”，`bundleSize` 为 `null` 时会显示成 `bundle 0 B`；第 4 步实现时去掉 bundle 大小这部分，并同步
  `run-delivery.test.tsx`。
- 仓库地址、交付 ref 与对象键都不进入公开 API；下载与 diff 预览仍未提供（Git Server 根决策未解决项）。

### D9：升级前提

- cloud、desktop（Controller 与 Node）、cluster 同时发布：cluster 同一次部署启用 Node 配置模板中的 `git_access`、
  Compose 中 Cloud 的 `git_server` 配置与 Git Server 服务。
- 升级时没有处于 `delivering` 的运行，也没有处于 `starting`、会话工作项尚未登记派发且输入带旧式
  `priorRevision.bundle` 的运行。前者的交付工作项输入只有 `bundleKey`，后者的会话输入只有 bundle；新的编解码
  与 Node 都无法执行它们。
- 不满足前提时系统不会进入不一致状态，但这些运行的成果会丢：交付无法结算，按 IssueRun 根决策 D5 的放弃窗口
  以 `deliveryState = failed` 释放。所以前提保护的是用户成果，运维在升级前确认这两类运行为零。

为什么不迁移进行中的工作项：要么同时保留两套编解码与 Node 路径，要么在迁移中替换已经固定的执行输入；前者与
“不向后兼容”矛盾，后者破坏“输入在放出工作项时固定”的幂等规则。

## 不变量

1. 每个新交付工作项的固定输入恰好含一个 `gitRepositoryUrl` 与一个 `deliveryRef = refs/ora/deliveries/<该工作项 ID>`，
   不含代码 bundle 的对象键；Controller 转交的输入与之逐字相同。
2. Cloud 只在 ref 查询确认交付 ref 指向声明的最终提交、且 JSONL 经 `HEAD` 核对一致之后登记 Revision。
3. ref 不存在或指向其他提交时不登记 Revision，记 `verification_failed`；Git Server 不可用时不接管、不确认，可重放。
4. 每个新登记的 Revision（包括 `unchanged`）都有 `delivery_ref`，且 bundle 列全为空。
5. 升级前的 Revision 行不删除、不改写。
6. 被选为前序的 Revision 有自己的交付 ref，且最终提交不等于其基础提交。
7. Git Server 只在运行处于 `delivering` 且交付执行尚无结果时授予写权限；只在输入带前序、运行处于
   `starting`/`running`、没有取消请求且 Thread 未在结束时授予会话执行读权限。
8. 上传授权只针对会话 JSONL 的键；不存在任何下载授权。
9. 执行输入、数据库与日志中没有 Git Server 访问材料或对象存储凭据，只有地址、ref 与对象键。
10. 缺少对象存储或 Git Server 配置时交付记为 `skipped`，不登记 Revision。

## 为什么不是这些替代方案

| 方案 | 取舍 |
| --- | --- |
| 保留 bundle 路径，以部署开关或回退兼容旧部署 | 升级平滑；但两套交付、核验、续作逻辑长期并存，`unchanged` 与续作语义分叉（一跳 vs 自有 ref），每处都要回答“这是哪种 Revision”，而唯一使用方本来就同步发布 |
| Node 上报 Git 交付能力，Cloud 按能力选择形态 | 允许新旧 Node 混部；但除了上一条的双路径成本，能力不足的工作项会在 `ClaimWork` 中排队阻塞后面的工作 |
| 开 `v2` 包 | 遵循 `buf.yaml` 的惯例，`buf breaking` 无需豁免；但唯一使用方同步发布，旧 `v1` 不会再被调用，双包只增加注册与编解码的维护（D6） |
| 保留 `GrantRevisionDownload` 与 `RevisionDelivered.bundle` 但标为废弃 | `buf breaking` 不需要豁免；但留下没有合法取值的契约，读者以为 bundle 路径仍在 |
| Controller 或 Node 配置 Git Server 地址 | Cloud 少一项配置；但地址不再受固定输入保护，Controller 还需知道 Project 才能拼路径，每个沙盒的配置都要随地址变化 |
| 只在 `delivered` 时推送，`unchanged` 不推 | 少一次推送；但续接后没有新提交的 Revision 又要间接引用前序的 ref，`unchanged` 也没有可核验的 Git 事实 |
| 把旧 bundle 导入 Git Server，使旧 Revision 可续作 | 旧成果不丢续作能力；但 bundle 不含基础提交，导入要先从用户上游取得基础提交，又回到 Git Server 想去掉的依赖 |
| 删除旧 `revisions` 行 | 表只剩一种形态；但丢掉已核验的历史，公开 API 中这些运行的 Revision 也随之消失 |
| 续作候选按字面“有交付 ref” | 规则更短；会选中停在基础提交的 Revision，把默认分支拨回旧提交（D7） |
| ref 查询失败按核验失败处理 | 交付更快收敛；Git Server 短暂不可用就会把成功的推送判失败（D3） |
| Git Server 推送成功后写收据，Cloud 不再查询 | 少一次调用；Git Server 根决策 D6 已否决：ref 与收据写入不是原子操作，登记方也不再亲自核验 |

## 本决策未解决的问题

- Git Server 中失败交付尝试留下的 ref 与对象、仓库配额与 GC、删除 Project 时的清理（Git Server 根决策未解决项）。
- 旧 bundle 对象与会话 JSONL 的过期清理（根决策未解决项）；对象存储是否在 JSONL 另有去处后下线。
- 同一 Issue 并发运行的协调（续接决策未解决项）。
- Git Server 多副本时 ref 查询与推送之间的一致性（Git Server 根决策“生产部署的多副本”）。

## 核心测试用例

本 ADR 获批后依据最终方案补充的核心用例如下。第 4 步尚未实现，新增义务的证据大多为 `Missing`，实现时按用例逐项
补充直接测试并更新状态。

- [Git 交付的输入、授权、核验与登记](../../../test-cases/cloud/revision/git-delivery-verification.md)：固定输入中的
  仓库地址与交付 ref；缺任一配置时 `skipped`；Git Server 授权的阶段条件；Revision 只在 ref 查询确认后登记
  （Git Server 根决策不变量 12，本文不变量 2），ref 不存在或不符记 `verification_failed`，Git Server 不可用不接管、
  可重放，`unchanged` 也登记交付 ref；`revisions` 的两种形态；公开投影保持形状。
- [从交付 ref 续作前序 Revision](../../../test-cases/cloud/revision/git-revision-resume.md)：续作候选只含有交付 ref
  且最终提交不等于基础提交的 Revision；会话输入的前序形状；取消的运行不标记不可续接。
- [Agent 运行执行、Thread 与上传授权](../../../test-cases/cloud/controller-integration/agent-run-executions-and-thread.md)：
  “成功交付等待 Cloud 验证”改为 ref 查询加 JSONL `HEAD`；“授权不持久化”中只剩 JSONL 上传授权。契约形状与
  Controller 行为的用例见 controller-integration 后续决策的“核心测试用例”一节。
- [Agent 运行的协议消息](../../../test-cases/node/protocol/agent-run-messages.md)：交付输入的 ref 与对象键校验改为
  `revision_ref`、交付 ref 与 JSONL 键；会话前序带仓库地址、交付 ref 与基础提交；没有下载授权消息。
- [个人模型访问](../../../test-cases/cloud/model-access/personal-model-access.md)：真实 CLI 的交付改为 JSONL 保存、
  交付 ref 核验。
- 端到端证据：cluster 的 `revision:acceptance` 与 `revision:resume:acceptance` 改走 Git，作为本组 ADR 的验收；验收只
  作补充证据，不替代上列义务的直接测试。

## 落地顺序

1. **specs**（已完成）：本文与 node/revision、controller-integration 两份后续决策于 2026-10-12 获批，核心用例随之补齐。
2. **cloud**：proto 修改与 `buf.yaml` 豁免、生成物、`internal/controlgrpc` 编解码（`DeliverRevisionSpec` 不再要求
   `bundle_key`）、`git_server` 配置与 ref 查询客户端、`deliveryWorkInput`、核验登记、迁移、续作选取、
   `GrantRevisionUpload` 只签 JSONL、删除 `GrantRevisionDownload`、前端交付展示去掉 bundle 大小（D8）；Git Server 的写授权改为读取固定输入中的
   `deliveryRef`。以真实 PostgreSQL、真实 Git Server 与 S3 兼容服务的集成测试证明不变量 2–4、6、7、10。
3. **desktop**：锁定新契约 commit；Controller 与 Node 按两份后续决策修改。
4. **cluster**：Node 配置模板启用 `git_access`，Compose 为 Cloud 配置 `git_server`，`revision:acceptance` 与
   `revision:resume:acceptance` 在 Git Server 上核对交付 ref、在 S3 兼容服务中核对 JSONL。
5. **cloud**：下一个 PR 移除 `buf.yaml` 豁免。

第 2–4 步分别评审，但必须同时发布（D9）。保持不变：会话 JSONL 的存放与校验、IssueRun 的阶段与重试、旧
Revision 的公开读取。从同时发布起改变：新交付的代码改动只在 Git Server，续作只从 Git Server 拉取，旧 Revision
不再被续作。
