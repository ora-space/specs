# 个人 HTTPS Git Token 的受限 Ingress 例外

依据[凭据规则受限例外](../../../decisions/cloud/identity-access/20261010-user-managed-git-token-ingress-rule-exception.md)
（`approved`，2026-10-10 落地到 `cloud/AGENTS.md`）与
[`cloud/AGENTS.md`](../../../../cloud/AGENTS.md) 中的原条目。本文件登记**规则侧的窄例外**及其边界：
Token 只能从一个位置、以一个身份被接收，必须在同请求内转交，请求体不得进入任何诊断面，且原规则的
其余部分逐字继续有效。

**当前实现状态（2026-10-12 本轮实现后）**：Ingress 端点已存在
（`POST`／`GET /api/v1/tenants/{tid}/git-credentials`，`PUT`／`DELETE
/api/v1/tenants/{tid}/git-credentials/{credId}`，见 `internal/api/router/router.go`），独立 Credential
Service 已实现（`internal/credentialstore/`、`cmd/credential-store`），单次授权的签发与兑换本轮补齐
（`internal/cloneauth/` 与 `internal/credentialstore/authorization.go`）。条件 1–4 的接收面、同请求转交与
投影已有直接证据；仍然 `Missing` 的是**按凭据直连**的执行侧——`OperationClone` 的 `Issue`
（`internal/core/credentialed_clone.go`）只在 `CLOUD_CLONE_DISPATCH_ENABLED=true` 时安装，该开关默认关闭，
Node 侧的凭据执行身份与私有 clone 未实现。**2026-10-13 更正**：本段原先把签发侧整体写成"Cloud 的 `Issue`
还没有生产调用方"并说"部署尚未签发 `broker-resolver` 证书"，两句都不再成立——镜像绑定
（`OperationMirrorPrepare`）的签发方 `core.MirrorDispatcher.Prepare` 由 `cmd/server` 的 `PrepareMirrorsOnce`
循环真实驱动，`broker-resolver` 证书由 `compose/clone-executor-setup.sh` 真实签发，启用镜像通道的隔离栈上
真实 Cloud 进程签发、真实 Store 兑换（`docs/internal-git-server-mirror-acceptance.md`）。这**不**解除
任何拒绝：没有任何通道的部署上 `repository_credential_executor_unavailable` 依旧是带凭据 clone 的答案，
而镜像通道把凭据留在执行器手里用一次、Node 拿到的是内部地址，与本文件的接收面例外无关。
凭据元数据与 fail-closed 见[凭据元数据](repository-credential-metadata.md)，秘密承载见
[凭据 Secret Store](repository-credential-secret-store.md)。

## Ingress Accepts Only The Owner's Own Token At One Position

### 风险

接收面扩散到多个端点或多个位置（URL、query、header、cookie），或允许调用方为他人的归属写入秘密，
使"用户自助"变成"Cloud 代为保管他人秘密"。

### 前置状态

存在一个专用凭据 Ingress 端点；认证与授权来自已验证 claim 与当前 PostgreSQL 状态；
`tenant_memberships` 与 `users` 的活跃状态可判定。

### 触发

未认证请求；已认证但未授权请求；调用方提交属于他人的 `owner_user_id` 或另一租户的 `tenant_id`；
把 Token 放在 URL、路径、query、header 或 cookie 中；向其他端点提交 Token。

### 必须成立

- 只有专用端点接收 Token，且只在**请求体**中；其他端点一律不接收。
- 调用方只能写自己拥有的凭据（`tenant_id` + `owner_user_id` = 调用方）；越界归属在边界处拒绝。
- 不存在"从其他位置兜底读取"的回退：Token 出现在非请求体的位置即视为违反条件。
- 拒绝发生在边界，不产生任何元数据行。

### 禁止结果

在通用请求解码路径中嗅探 Token 字段、以 header 或 query 作为可接受的替代位置、为"兼容客户端"新增
第二个接收端点、以调用方提供的字段决定归属。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 唯一端点、唯一位置 | `integration.git_credential_ingress_test.go` 的 `TestGitCredentialIngressAcceptsTheTokenInNoOtherPosition` 断言只有 `token` 这一个字段名被接受，`secret`、`accessToken`、`password`、`credential`、`pat` 各自以 `unknown_field` 被拒绝并回传字段名；路由只在这两个路径上把请求体当成 Token 载体（`internal/api/router/router.go` 的 `credentialIngressPath`）。"其他端点一律不接收 Token" 由同一套严格解码保证，但没有针对性的跨端点测试 | Partial |
| 只允许写入自己的凭据 | `TestGitCredentialIngressIsolatesOwnersAndTenants`（同租户他人为 404、另一租户为 403，且被拒绝的调用不扰动它命名的凭据）；`TestGitCredentialIngressAcceptsTheTokenInNoOtherPosition` 断言 `tenantId`、`ownerUserId`、`userId`、`id`、`availability`、`capabilities`、`secretRef` 等服务端归属字段以 `unknown_field` 拒绝；`integration.git_credential_binding_test.go` 的 `TestProjectBindingAcceptsOnlyACredentialThatCouldClone` 断言另一成员的凭据不可绑定 | Covered |
| 非请求体位置一律拒绝（无兜底读取） | `TestGitCredentialIngressAcceptsTheTokenInNoOtherPosition` 覆盖了字段名与畸形 JSON（第二个 JSON 值以 `invalid_json` 拒绝且不回显 Token）；Token 出现在 URL、query、header 或 cookie 时没有可读取它的代码路径，但这四种位置没有逐一位置的拒绝测试 | Partial |

### 决策依据

例外 ADR D1 条件 1、2 与 D2 适用范围表，不变量 2、4；`cloud/AGENTS.md` 的严格解码与 owner 校验规则。

## Ingress Hands Off Within One Request And Persists Nothing

### 风险

Cloud 成为秘密的第二份持有者：Token 被排队、落盘、缓存，或写进业务库、业务事件、公开响应、业务消息、
持久文件与 Workspace 卷——即使加密，也使 Cloud 进程成为解密能力，并让失败语义失去可解释性。

### 前置状态

专用 Ingress 端点存在；Store 的写入侧服务身份（`ingress-writer`）可用或不可用，两种情形都要覆盖。

### 触发

在**同一次请求内**转交成功；Store 不可用；Store 写入成功但 Cloud 元数据事务失败；用户随后请求取回
已保存的 Token；检查数据库、事件、日志与卷中是否出现 Token 或其密文。

### 必须成立

- 转交发生在同一次请求内；Token 不被排队、不被落盘缓冲、不被缓存、不写入任何 Cloud 存储。
- 转交后 Cloud 只保留 `credential_refs` 元数据与不透明的 `secret_ref`，任何 API 都不得返回 Token。
- Store 不可用即整体失败：不返回 credential id、不产生元数据行，可安全重试。
- Store 写入成功而 Cloud 元数据事务失败时，对用户报保存失败，孤儿秘密由 Store 侧补偿清理。

### 禁止结果

先持久化再转交、把 Token 或密文写进业务库／事件／文件／卷、在失败时"先记下再补交"、
在错误响应或响应头中回显 Token、为用户提供"取回 Token"的能力。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 同请求转交且 Cloud 不持久化 | `integration.git_credentials_test.go` 的 `TestGitCredentialCreateHandsTheTokenOverAndKeepsOnlyMetadata` 经真实 HTTP 与 PostgreSQL 断言创建完成于一次请求内、`secrets.stored(id)` 在服务侧持有 Token 而 Cloud 只留元数据；辅助断言 `assertTokenAbsentFromCloudState` 对 `credential_refs`、`idempotency_records`（含整个 `response::text`）与 `operations.request` 三处做全文否定断言。`TestGitCredentialIngressValidatesNameScopeAndToken` 进一步断言被拒绝的请求从未到达服务 | Covered |
| 转交失败即整体失败、可安全重试 | `TestGitCredentialServiceRefusalsBecomeReasonCodes` 注入五类服务失败（不可达、未知码、`credential_conflict`、`credential_not_found`、`invalid_request`）并断言各自的公开码且不留元数据行；`TestGitCredentialIngressFailsClosedWithoutACredentialService` 断言未配置服务时 503、不写元数据、不记幂等记录；`TestGitCredentialCreateIsIdempotentOnItsKey` 断言重放只发生一次转交。"Store 写入成功而 Cloud 元数据事务失败"这一支已补上（2026-10-12）：`core.TestACredentialCreateThatLosesItsMetadataCommitReconcilesByTheSameIdentity` 用对抗性输入（调用前先占用推导出的 id）强制转交成功后的元数据 INSERT 失败，断言整体报错、**不推断成功**、转交只发生一次，并在清掉冲突后断言重试以**同一身份**收敛（同一 id、同一 `secret_ref`、仅一行、两次转交同一 id）；`core.TestACredentialCreateFailsClosedWhenTheStoreCannotBeReached` 同时固定 Store 不可达时得到 `credential_store_unavailable`/503 且零元数据行、零幂等行。仍是 `Partial`：这些用例走的是 `internal/core` 的转交缝，真实 credential-store 进程与真实元数据事务的组合仍未演练 | Partial |
| 任何 API 都不返回 Token | `TestGitCredentialCreateHandsTheTokenOverAndKeepsOnlyMetadata` 与 `TestGitCredentialRotateReplacesMaterialBeforeAdvancingTheVersion` 对响应字节断言不含 Token；`TestGitCredentialIngressFailsClosedWithoutACredentialService` 复用 `assertTokenAbsentFromCloudState`；契约侧 `api/openapi.json` 的凭据投影没有 Token 字段。尚无对全部响应做 Token 检索的仓库级检查 | Partial |
| 孤儿秘密由 Store 侧补偿清理 | 见[凭据 Secret Store](repository-credential-secret-store.md) 的一致性用例 | Missing |

### 决策依据

例外 ADR D1 条件 3，不变量 5、6；Secret Store ADR D4／D6 的失败语义。

## Request Bodies Never Reach Logs Or Error Context

### 风险

请求体进入访问日志、追踪 span、错误详情或崩溃报告，使明文 Token 落到最容易被检索、导出与长期保留
的地方——这是原规则 `never log` 子句针对的主要场景，例外**不**放宽它。

### 前置状态

Ingress 端点接入结构化日志与追踪；错误路径只返回原因码。

### 触发

正常接收、认证失败、Store 失败与 panic 恢复四条路径各触发一次；检查访问日志、追踪 span、
错误详情与崩溃报告的完整内容。

### 必须成立

- 请求体不进入访问日志、追踪 span、错误详情或崩溃报告；错误响应只携带原因码。
- 原条目中"结构化日志、排除秘密与不必要个人数据"的要求完整继续有效。
- 日志可用结构化字段与请求／资源 ID 定位这次调用，而不需要请求体。

### 禁止结果

以"调试开关"记录请求体、把请求体放进 span attribute、把 Token 放进 panic 上下文或错误链、
用请求体内容作为错误消息的一部分。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 四条路径都不泄露请求体 | 错误详情一侧：`internal/core/git_credentials_test.go` 的 `TestCredentialStoreFaultIsTotalAndCarriesNoDetail` 断言转交失败被折叠为总量化的原因码、不携带 Store 的文本，使 ingress 路径上的 `zap.Error(err)` 不可能写出请求体内容；`TestGitCredentialServiceRefusalsBecomeReasonCodes` 断言公开参数不引用传输层文本。**日志一侧仍无证据**：没有捕获 Cloud 访问日志与结构化日志并检索 Token 的测试，四条路径（正常接收、认证失败、Store 失败、panic 恢复）也没有各自的日志断言 | Partial |
| 错误响应只含原因码 | `integration.git_credential_ingress_test.go` 的 `failAs` 对每个拒绝断言确切的 `code` 而不是"非 200"，`TestGitCredentialServiceRefusalsBecomeReasonCodes` 断言五类服务失败各自的公开码且参数不含传输层文本，`TestGitCredentialIngressAcceptsTheTokenInNoOtherPosition` 断言的 `invalid_json` 响应体不含 Token | Covered |

### 决策依据

例外 ADR D1 条件 4，不变量 7；`cloud/AGENTS.md` 的结构化日志与脱敏规则。

## The Rule Revision Keeps Every Original Prohibition

### 风险

修订被当作"整条规则作废"的理由——原句被删除或弱化，于是对象存储凭据、部署私钥、服务账号令牌、
SSH 私钥与团队共享凭据失去了默认禁止来源；或者例外被静默扩张到 D2 表里"不适用"的行。

### 前置状态

`cloud/AGENTS.md` 中凭据条目 = 原句（逐字）+ 其后追加的例外段落；例外只覆盖用户个人 HTTPS Git Token。

### 触发

核对 `cloud/AGENTS.md` 的差异；尝试以"已经有接收端点"为由接收团队共享凭据、SSH 私钥或部署密钥；
尝试新增第二个接收端点。

### 必须成立

- 原句与测试密钥、日志脱敏两句**逐字保留**；修订是纯追加，不含删除或改写。
- 例外只覆盖经认证且已授权的终端用户提交的、属于自己的个人 HTTPS Git Token；`purpose='git'`。
- `cloud/AGENTS.md` 中其余规则（严格解码、服务与终端用户凭据相互独立、授权只来自已验证 claim 与
  当前 PostgreSQL 状态）继续有效，例外不放宽它们。
- 扩张例外范围或放宽 fail-closed 必须由**新的决策**承载，不能在规则文本处就地编辑。

### 禁止结果

删除或弱化 `never accept` 子句、把 `repository credentials` 收窄为 `static repository credentials`、
把例外写成无条件的"Cloud 可接收用户凭据"、为了新凭据类型复用同一个接收位置。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 修订为纯追加、原句逐字保留 | `git -C cloud diff --numstat AGENTS.md` 仍为 24 行新增、0 行删除（2026-10-10 复核）。该证据是差异评审，不是测试，因此这里不记为测试证据 | Partial |
| 例外范围未被扩张 | 尚无把 `cloud/AGENTS.md` 的例外边界与 D2 表逐行比对的自动化检查。相关事实证据：`internal/core/git_credentials.go` 的凭据写入路径把 `purpose` 固定为 `'git'`，`TestGitCredentialCreateHandsTheTokenOverAndKeepsOnlyMetadata` 断言该值，因此该接收面无法承载其他凭据类型 | Partial |
| 不放宽相邻规则 | 尚缺针对"例外不覆盖团队凭据与服务账号令牌"的否决测试 | Missing |
| 规则侧获批不解锁运行时 | 规则侧落地、Cloud 侧 ingress 与独立 Store 都已实现之后，`integration.git_credential_binding_test.go` 的 `TestCredentialedCloneStaysRefusedOnEveryPath` 仍断言每个执行路径返回 `repository_credential_executor_unavailable`，且受控 Project 不产生 `clone_executions` 行；`integration.repository_credentials_test.go` 的 `TestCredentialedCloneRefusesMissingExecutorAndAnonymousFallback` 给出同一结论。两个测试都不区分规则落地前后，因此不算完整证据 | Partial |

### 决策依据

例外 ADR D1 条件 5、D2 适用范围表、D3 与 D4，不变量 1、3、8、9；被修订的
`cloud/AGENTS.md` 凭据条目原文。
