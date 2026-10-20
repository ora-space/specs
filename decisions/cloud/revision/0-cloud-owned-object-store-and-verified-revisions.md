---
status: implemented
---

# Cloud 持有对象存储配置并签发限定上传授权，Revision 在 Cloud 校验对象后才登记

对象存储（S3 兼容）的端点、桶与访问凭据是 Cloud 的启动参数；Controller 与 Node 都不持有对象存储凭据。
Revision 交付时，Controller 向 Cloud 申请本次交付的**上传授权**（针对固定对象键、限定时长的预签名 PUT），
原样转交 Node；Node 直接上传增量 bundle 与会话 JSONL，报告对象键、大小与 SHA-256。Cloud 在数据库事务
之外用自己的凭据校验对象存在且摘要一致，然后在事务中登记 Revision。Revision 记录最终版本与对象位置，
是 IssueRun 释放运行 Workspace 的前提。

本 ADR 处于 `implemented`，是领域 `cloud/revision` 的根决策，描述 Cloud 当前的 bundle 交付实现。Revision 的
Git 语义（内部快照、累计增量 bundle、无改动时不生成 bundle）由 [Node revision 根决策](../../node/revision/0-internal-snapshot-and-incremental-bundle-upload.md)
承担；何时交付、失败如何重试由 [IssueRun 根决策](../issue-run/0-agent-run-in-disposable-isolated-workspace.md) D5
承担；gRPC 形状见 [gRPC 后续决策](../controller-integration/20260928-agent-run-executions-thread-and-upload-grants.md)。
[IssueRun 续接决策](../issue-run/20261010-resume-run-from-latest-issue-revision.md) 修改了本文 D4 的 `unchanged`
条件与 D5 的 `changed` 含义，正文已按修改后的行为描述。兼容性边界：新增配置项与表，不改变已有数据。本文
不设计 bundle 过期与推送到用户远端。

## 决策

### D1：对象存储是 Cloud 的启动配置，凭据按基础设施引用注入

Cloud 配置的 `object_store` 段，每个键都可由 `CLOUD_OBJECT_STORE_<KEY>` 环境变量覆盖：

| 字段 | 含义 |
|---|---|
| `enabled` | 是否启用；缺省或为 `false` 时视为未配置 |
| `endpoint` | S3 兼容端点 URL（当前 cluster 使用 RustFS） |
| `region` | 签名区域 |
| `bucket` | Revision 对象所在的桶 |
| `path_style` | 是否使用 path-style 寻址（当前 RustFS 联调开启） |
| `access_key_id_file`、`secret_access_key_file` | 凭据文件路径；凭据值不进入配置文件、日志或数据库 |
| `upload_grant_ttl` | 单个授权的有效期，启用时必须在 1 秒到 168 小时之间；随附配置为 15 分钟 |
| `public_endpoint` | 可选；签发给 Node 的 URL 所用端点，用于 Node 与 Cloud 看到的地址不同的部署 |

- 未配置时 Cloud 仍可启动，`GET /healthz` 的 `dependencies.objectStore` 报告 `unconfigured`（配置后为
  `configured`，不探测可达性）。Agent 运行进入 `delivering` 时不放出交付工作项，在同一事务中以
  `deliveryState = skipped`（原因 `object_store_unconfigured`）直接进入释放，每个运行只记录一次。这让不需要
  Revision 的开发环境不被阻塞，同时不把“未保存”伪装成“已保存”。
- 启用时，静态配置非法（凭据文件不可读或为空、TTL 越界、端点不是不带 userinfo/query/path 的 http(s) URL、
  区域或桶为空）拒绝启动；Cloud 以本地预签名校验私有与公开端点，不发网络请求。端点不可达、桶不存在等
  运行期错误不阻止启动，只让交付失败并按 IssueRun D5 重试；Cloud 不创建桶。

为什么放在 Cloud：Cloud 是 Revision 的权威登记方，也是唯一有能力校验对象的一方；Controller 不持凭据，
符合“Controller 无租户、不持有业务权威”的边界，也少一个需要轮换 S3 凭据的进程。

### D2：对象键由 Cloud 按运行决定，Node 不能选择写入位置

每个交付工作项（一次交付尝试）对应一组固定对象键与 Revision ref：

```text
revisions/{tenantId}/{runId}/{deliveryWorkId}/revision.bundle
revisions/{tenantId}/{runId}/{deliveryWorkId}/session.jsonl
refs/ora/revisions/{runId}
```

- 键在 Cloud 放出交付工作项时生成并写入工作项的输入（[controller-integration D6](../controller-integration/20260928-agent-run-executions-thread-and-upload-grants.md)
  的 `execution_work`）。此时执行 ID 尚不存在（它由 Controller 在 `RecordDispatch` 时生成），所以键使用
  工作项 ID；每次重试是新的工作项，不会覆盖上一次尝试的对象。
- 预签名 URL 只对单个键、`PUT` 方法和 `upload_grant_ttl` 有效。请求带上 Node 预先算出的 SHA-256 时，
  `x-amz-checksum-sha256` 签入请求，由 S3 在写入时校验内容与声明一致；不带摘要的旧版请求只签入
  `x-amz-sdk-checksum-algorithm: SHA256`，摘要头由上传方附加。
- 所有 PUT 授权（含旧版请求）将 `If-None-Match: *` 签入请求，只能首次创建对象；不能移除该条件或用
  尚未过期的另一张授权覆盖已验证对象。重复 PUT 返回 412 时仍须将原声明交由 Cloud 校验大小和摘要，
  不能凭状态码推断相同内容。这样外部 HEAD 与 SQL 提交之间及提交之后的对象均不会被上传能力改写。

### D3：上传授权不属于执行输入，可以随时重新申请

- 上传授权（URL、需要附带的头、过期时间）由 Controller 在向 Node 下发交付执行之后调用
  `GrantRevisionUpload(execution_id, checksums?)` 获取，并以非持久消息交给 Node；Node 过期后可请求新的授权。
- 可选的 checksum map 只能包含固定输入中的键，值为规范的 SHA-256；它属于本次授权请求，不属于持久执行
  输入。旧 v1 请求不带 map，仍为两个键各签发一张授权。
- 授权不写入执行登记、提交记录、Node 账本或日志；它是短期持有者凭据。执行输入只包含对象键，因此同一
  执行重传时输入不变，幂等规则不受授权刷新影响。签发不访问对象存储。
- 只对尚无结果、未被强停、且仍在交付的执行签发：Workspace 就绪且准入开放，运行未结束，运行时控制处于
  绑定本运行的 maintenance，当前 Node 与沙盒和工作项目标一致，对象存储已配置。错误映射：租约问题为
  `FAILED_PRECONDITION`（Controller 会重试）；执行不存在为 `NOT_FOUND`；checksum map 非法为
  `INVALID_ARGUMENT`（`invalid_upload_checksum`）；其余业务拒绝（包括 `object_store_unconfigured`）统一为
  `ABORTED`/`CONFLICT`。
- 续接运行读取前序 bundle 的只读授权由
  [恢复契约决策](../controller-integration/20261010-revision-restore-contract.md) 的 `GrantRevisionDownload` 承担，
  签发规则与本节相同。

### D4：Revision 在对象校验后登记，校验与事务分离

Node 报告交付结果（`delivered{finalCommit, baseCommit, ref, bundle{key, size, sha256}, history{key, size, sha256}}`
或 `unchanged{...}` 或 `failed{reason}`）后，Cloud 按以下顺序处理：

1. 租约、提交身份与事件序号（必须是上一个序号加一）的检查与其他执行结果相同。然后比对声明与交付工作项的
   固定输入：Node、`revisionRef`、`baseCommit` 与对象键必须与输入一致，提交与对象声明必须形状合法。不一致在
   接管**之前**拒绝：Node、基础提交、ref、最终提交、bundle 键与 bundle 形状的问题返回 `ABORTED`/`CONFLICT`
   （`result_conflict`）；会话记录键与会话记录形状的问题返回 `INVALID_ARGUMENT`/`INVALID_INPUT`
   （`invalid_result`）。`delivered` 必须带大小为正的 bundle 且最终提交不等于基础提交；`unchanged` 不能带
   bundle，最终提交等于基础提交，或等于续接运行输入中前序的最终提交。拒绝不访问对象存储，不写收据、持久
   结果、验证结论或 Revision，不推进序号，运行保持 `delivering`；该交付尝试随后按 IssueRun D5 的放弃窗口
   收敛（`GiveUpStaleDeliveriesOnce`）。
2. 事务外：对每个声明的对象以私有端点调用 `HEAD`（带 checksum 模式，不跟随重定向），确认存在、大小与
   SHA-256 与声明一致。
3. 事务内：再次检查租约、执行身份与输入、运行未结束、maintenance 绑定、Workspace 准入及当前 Node 与沙盒，
   然后把原始 Node 结果、收据、验证结论、`revisions` 行与 IssueRun 的交付结算（`OnDeliverySettled`，运行进入
   `releasing`）在同一事务中提交。
4. 对象不存在（404）或大小、摘要不符：接管原 Node 事件，在独立 Cloud 验证结论（`revision_verifications`）中记
   `failed{verification_failed}`，同事务调用失败的交付结算，按 IssueRun D5 重试新交付；不登记 Revision。
   原 Node 成功声明与原始事件保留在 `node_executions.result`，不以 Cloud 推断覆盖；Node 的失败码是封闭集合，
   不能在自己的结果中代替 Cloud 报告 `verification_failed`。其他 HTTP 状态、网络错误或对象存储在运行中被
   取消配置时返回 `UNAVAILABLE`，不接管、不 ACK，可重放。

重复接管不重新访问对象存储；已结算执行的重放只接受与原结果逐字节相同的事件。

实际决策与偏离：原决策按“与输入矛盾 / 形状不合法”分别返回 `CONFLICT` 与 `INVALID_INPUT`。实现按声明字段
归类：会话记录键与会话记录形状都返回 `INVALID_INPUT`，bundle 键与 bundle 形状都返回 `CONFLICT`。两种错误码
都在接管前拒绝、都不留记录，Controller 对二者的处理相同，系统承诺（不变量 3a）不受影响。bundle 路线不再
为此改代码：Git Server 交付删除 bundle 字段后，bundle 一侧的归类随之作废；剩下的 `history` 键与形状返回
`INVALID_INPUT`，其余矛盾返回 `CONFLICT`，由
[Git 交付决策](20261017-git-native-delivery-refs-and-resume.md) D3 沿用。

`revisions` 表：`id`、`tenant_id`、`run_id`（每个运行至多一行）、`execution_id`（对应的验证结论）、`workspace_id`、
`project_id`、`repository_url`、`base_commit`、`final_commit`、`revision_ref`、`bundle_key/bundle_size/bundle_sha256`
（本运行没有自有 bundle 时为空）、`history_key/history_size/history_sha256`、`created_at`、`expires_at`（目前恒为
空），以及续接决策增加的 `prior_revision_id`、`bundle_revision_id`、`resume_refused_at`、`resume_refused_reason`。
`revision_verifications` 记录每个交付执行的 Cloud 结论（`delivered`、`unchanged`、`failed`），
`revision_delivery_skips` 记录因未配置对象存储而跳过交付的运行。

- 无文件改动（`unchanged`）也登记 Revision，且没有自有 bundle；与“保存失败”区分。
- 为什么校验放在事务外：仓库规则禁止在事务中进行 HTTP 调用；先校验再写入，最坏情况是校验通过后事务失败，
  Controller 以同一提交身份重试即可，对象不变。
- 为什么由 Cloud 而不是 Controller 校验：Cloud 登记 Revision 就是在承诺“这些对象存在且可恢复”，承诺方应
  亲自验证；Controller 转述 Node 的“已上传”正是 Revision 方案否定的交付证据。
- 为什么与输入矛盾的声明在接管前拒绝，而不是接管后记 `verification_failed`：这类矛盾不需要对象存储就能判定，
  且说明结果不属于这次尝试（指向别的引用、基础提交或对象键）；接管它会把一份不可信的声明写成该执行的持久
  结果。控制面根决策已允许以 `CONFLICT` 表达结果冲突，拒绝后执行仍无结果，由放弃窗口而不是一份伪造的终态
  收敛。代价是这类尝试不会立即触发重试，要等放弃窗口到期。

### D5：公开读取只暴露元数据

Agent 运行的单个与列表读取都带 `revision{id, baseCommit, finalCommit, changed, bundleSize, historySize, priorRevisionId, createdAt}`，
未登记时为 `null`。`changed` 表示本运行存了自有 bundle；`bundleSize` 与 `priorRevisionId` 可为 `null`。运行的
`result` 中有 `deliveryState`（`saved`、`unchanged`、`failed`、`skipped`）与 `revisionId`（未登记时为 `null`），
续接运行另有 `resumeRevisionId`。对象键、ref、摘要与 URL 都不出现在公开 API 中；不提供下载或 diff 预览接口。
Issue 页的 Agent 会话面板据此显示保存结果。

## 不变量

1. 对象存储凭据只在 Cloud 进程内；Controller、Node、数据库与日志中没有对象存储凭据，也不持久化上传授权。
2. Node 只能写入 Cloud 为本次交付尝试（交付工作项）指定的对象键。
3. Revision 只在 Cloud 亲自校验对象存在且 SHA-256 一致后登记。
3a. 与交付输入矛盾或形状不合法的声明在接管前被拒绝，不访问对象存储、不留下任何持久记录。
4. 同一交付执行的输入在授权刷新后不变。
5. 无改动的运行也有 Revision 记录，且没有自有 bundle。
6. 未配置对象存储时交付明确记为 `skipped`，与运行的交付结算同事务、只记录一次，不登记 Revision。

## 为什么不是这些替代方案

| 方案 | 取舍 |
|---|---|
| S3 配置放在 Controller，由它签发授权 | Cloud 少一个依赖；但 Controller 要持有并轮换凭据，且 Cloud 仍需凭据才能校验 |
| 把 S3 凭据下发给 Node | 最简单；沙盒获得整个桶的写权限，违反“Node 不持凭据” |
| bundle 经 Controller–Node 长连接传回，由 Cloud 写入对象存储 | 无需预签名；但大文件会阻塞控制与 Thread 消息，Revision 方案已否决 |
| 预签名 URL 放进执行输入 | 省一次调用；URL 过期后重传同一执行会变成“输入不同”，破坏幂等 |
| 只信任 S3 的 checksum 头，不做 HEAD | 少一次请求；但无法区分“Node 声称上传了”与“对象确实存在”，例如上传请求从未发出 |

## 本决策未解决的问题

- bundle 与会话 JSONL 的过期与清理、失败尝试的残留对象清理任务。
- 下载、diff 预览与推送到用户远端（需要另行授权）。
- 生产对象存储的凭据轮换与加密策略。

## 验证

核心用例见 [Cloud 校验后登记的 bundle Revision](../../../test-cases/cloud/revision/verified-bundle-revisions.md)。
Cloud CI 以真实 RustFS 运行 PostgreSQL 集成测试：`integration/revision_test.go` 覆盖 `changed`/`unchanged` 登记、
对象缺失与大小摘要不符、hook 与提交回滚、网络失败可重放、外部校验后重新检查租约与状态、重试使用新对象键
并拒绝错误摘要与过期授权；`revision_immutability_test.go` 覆盖有效授权不能覆盖已验证对象；
`agent_run_delivery_test.go` 覆盖授权签发与拒绝、与输入矛盾的声明不访问对象存储、Node 不能报告 Cloud 的
验证结论、公开投影不泄漏对象信息；`agent_hook_atomicity_test.go` 覆盖未配置时只跳过一次且与业务同事务。
签名只绑定单个键由 `internal/objectstore` 的单元测试覆盖，尚无真实 S3 上“用 A 键授权写 B 键被拒”的用例。
