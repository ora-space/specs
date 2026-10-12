# Identity & Access 核心测试用例

本目录是 Cloud identity-access 领域的 reviewer 风险导航图，镜像
`specs/decisions/cloud/identity-access/`。它记录 Gateway 外部身份归一、浏览器会话与内部凭据边界需要
长期守住的系统承诺，以及这些承诺是否已有直接、可信的可执行证据；它不是测试函数目录。

架构边界和不变量来自
[外部身份由 Gateway 归一，Cloud 保持授权权威](../../../decisions/cloud/identity-access/0-gateway-mediated-external-identity.md)。

用户托管的仓库凭据（HTTPS Git Token）于 2026-10-10 随四份 ADR 进入 `approved`，其核心用例登记在
[凭据元数据](repository-credential-metadata.md)、[凭据 Ingress 受限例外](repository-credential-ingress-exception.md)、
[凭据 Secret Store](repository-credential-secret-store.md) 与
[仓库凭据归属连续性](repository-credential-ownership-continuity.md)。执行侧的身份隔离见
[Node 凭据执行身份隔离](../../node/repository/git-credential-execution-isolation.md)。

**当前实现状态（2026-10-12 本轮实现后）**：Cloud 侧的凭据元数据、用户自助 Ingress 与独立
Credential Service 已经落地，证据集中在 `integration.git_credentials_test.go`、
`integration.git_credential_ingress_test.go`、`integration.git_credential_binding_test.go`、
`internal.credentialstore.*` 与 `internal.config.*`。本轮补上了**单次授权的 Cloud+Store 半程**：
授权载体与验签在 `internal.cloneauth`，Store 侧的兑换、原子消费与重放拒绝在
`internal.credentialstore`（`Resolver` 类型与 `authorization.go`）。仍然 `Missing` 的是**执行侧**：
Cloud 的 `Issue` 还没有生产调用方（唯一合法调用点在 `requireRepositoryAccess` 的无条件拒绝之后），
部署尚未签发 `broker-resolver` 证书，Node 侧的凭据执行身份 C 与 clone 后清理未实现，因此私有仓库
clone 在所有路径上继续保持 `repository_credential_executor_unavailable` 拒绝。规则侧获批与 Cloud 侧
落地都不解除该拒绝。

## 阅读与维护规则

- 核心用例按业务风险组织，不按 Go package、单元测试或集成测试组织。
- 代表性证据写作 `包.测试函数`：`integration.*` 是 `integration/gateway_test.go` 中使用真实 HTTP、
  PostgreSQL、真实 Cloud router 与会校验 PKCE 的假 provider 的验收测试；`gateway.*` 与 `github.*` 是
  `internal/gateway` 及其 `github` 子包的单元测试。
- 只有在测试失败会直接说明某项验证义务不成立时，才把该测试列为代表性证据。
- 文件路径与用例标题共同构成稳定引用；集成测试的 doc comment 引用它们。
- 改变 attempt/session 状态、事务顺序、Cookie 策略、凭据签发、代理过滤或数据库约束时，必须检查相关
  核心用例及其证据状态。

## 证据状态

| 状态 | 含义 |
|---|---|
| `Covered` | 已有测试直接证明完整验证义务。 |
| `Partial` | 已有测试证明部分路径或较低层规则，但尚未证明完整系统承诺。 |
| `Missing` | 尚未找到直接证明该验证义务的代表性测试。 |

## 当前基准用例

- [Login Start Requires Same-Origin Proof And Writes One Bound Attempt](login-attempt.md#login-start-requires-same-origin-proof-and-writes-one-bound-attempt)
- [Callback Consumes An Attempt At Most Once](login-attempt.md#callback-consumes-an-attempt-at-most-once)
- [Provider Rejection Never Creates A Session](login-attempt.md#provider-rejection-never-creates-a-session)
- [GitHub Identity Is Keyed By Numeric ID Within Cloud Field Limits](login-attempt.md#github-identity-is-keyed-by-numeric-id-within-cloud-field-limits)
- [Session Validity Is Decided Only By PostgreSQL State And Time](browser-session.md#session-validity-is-decided-only-by-postgresql-state-and-time)
- [Revocation Is Immediate On Every Replica And Idempotent](browser-session.md#revocation-is-immediate-on-every-replica-and-idempotent)
- [Cleanup Is Bounded And Preserves Live Rows](browser-session.md#cleanup-is-bounded-and-preserves-live-rows)
- [Schema Constraints Reject Invalid Session And Attempt Rows](browser-session.md#schema-constraints-reject-invalid-session-and-attempt-rows)
- [Cloud Only Receives Gateway-Issued Short-Lived Credentials](proxy-and-credentials.md#cloud-only-receives-gateway-issued-short-lived-credentials)
- [Internal Routes And Cross-Site Mutations Never Reach Cloud](proxy-and-credentials.md#internal-routes-and-cross-site-mutations-never-reach-cloud)
- [Cloud Authorization Decisions Pass Through Unchanged](proxy-and-credentials.md#cloud-authorization-decisions-pass-through-unchanged)
- [Cloud Outage Is A Stable Failure That Keeps Sessions](proxy-and-credentials.md#cloud-outage-is-a-stable-failure-that-keeps-sessions)

用户托管仓库凭据（2026-10-10 批准；Cloud 侧元数据、Ingress 与 Credential Service 已实现，
执行侧仍未实现）：

- [Departed Member Freezes Personal Credentials Without Rewriting Ownership](repository-credential-ownership-continuity.md#departed-member-freezes-personal-credentials-without-rewriting-ownership)
- [Rejoining Never Unfreezes A Frozen Credential](repository-credential-ownership-continuity.md#rejoining-never-unfreezes-a-frozen-credential)
- [Applied Migration Semantics Are Records, Not Editable Drafts](repository-credential-ownership-continuity.md#applied-migration-semantics-are-records-not-editable-drafts)
- [Credential Metadata Is The Only Authority And Secret Reference Is Not A Bearer Token](repository-credential-metadata.md#credential-metadata-is-the-only-authority-and-secret-reference-is-not-a-bearer-token)
- [Credentialed Clone Refuses A Missing Executor And Never Falls Back](repository-credential-metadata.md#credentialed-clone-refuses-a-missing-executor-and-never-falls-back)
- [Credential Binding Is Frozen Per Clone Execution With Its Version](repository-credential-metadata.md#credential-binding-is-frozen-per-clone-execution-with-its-version)
- [Credential Metadata Changes Do Not Rewrite Applied Migrations](repository-credential-metadata.md#credential-metadata-changes-do-not-rewrite-applied-migrations)
- [Ingress Accepts Only The Owner's Own Token At One Position](repository-credential-ingress-exception.md#ingress-accepts-only-the-owners-own-token-at-one-position)
- [Ingress Hands Off Within One Request And Persists Nothing](repository-credential-ingress-exception.md#ingress-hands-off-within-one-request-and-persists-nothing)
- [Request Bodies Never Reach Logs Or Error Context](repository-credential-ingress-exception.md#request-bodies-never-reach-logs-or-error-context)
- [The Rule Revision Keeps Every Original Prohibition](repository-credential-ingress-exception.md#the-rule-revision-keeps-every-original-prohibition)
- [Store Accepts Only Service Identities And Splits Write From Resolve](repository-credential-secret-store.md#store-accepts-only-service-identities-and-splits-write-from-resolve)
- [Token Is Encrypted With A Delivery-Only Key And Unreadable Without It](repository-credential-secret-store.md#token-is-encrypted-with-a-delivery-only-key-and-unreadable-without-it)
- [Cloud And Store Stay Consistent Across Create Rotate Revoke](repository-credential-secret-store.md#cloud-and-store-stay-consistent-across-create-rotate-revoke)
- [Authorization Is Consumed Atomically At Redemption And Replay Is Refused](repository-credential-secret-store.md#authorization-is-consumed-atomically-at-redemption-and-replay-is-refused)
- [Audit And Backup Never Expose Secrets Or Ciphertext](repository-credential-secret-store.md#audit-and-backup-never-expose-secrets-or-ciphertext)
