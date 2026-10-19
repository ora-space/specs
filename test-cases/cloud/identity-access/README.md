# Identity & Access 核心测试用例

本目录是 Cloud identity-access 领域的 reviewer 风险导航图，镜像
`specs/decisions/cloud/identity-access/`。它记录 Gateway 外部身份归一、浏览器会话与内部凭据边界需要
长期守住的系统承诺，以及这些承诺是否已有直接、可信的可执行证据；它不是测试函数目录。

架构边界和不变量来自
[外部身份由 Gateway 归一，Cloud 保持授权权威](../../../decisions/cloud/identity-access/0-gateway-mediated-external-identity.md)。

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
- [Only The Owner Reads And Writes A Git Identity](user-git-commit-identity.md#only-the-owner-reads-and-writes-a-git-identity)
- [Session Start Fixes The Trigger User's Identity](user-git-commit-identity.md#session-start-fixes-the-trigger-users-identity)

## Git 凭据存储用例

依据 [Git 凭据存储决策](../../../decisions/cloud/identity-access/20261016-git-credentials-in-independent-credential-store.md)，
实现尚未合入，证据均为 `Missing`。

- [Git Tokens Reach Only The Browser And The Credential Store](git-credential-store.md#git-tokens-reach-only-the-browser-and-the-credential-store)
- [Submission And Execution Authorizations Are Distinct And Single-Use](git-credential-store.md#submission-and-execution-authorizations-are-distinct-and-single-use)
- [A Reference Becomes Stored Only After Cloud Confirms The Version](git-credential-store.md#a-reference-becomes-stored-only-after-cloud-confirms-the-version)
- [Capabilities Hold For One Store Version And Are Proven, Not Declared](git-credential-store.md#capabilities-hold-for-one-store-version-and-are-proven-not-declared)
- [Administrators Cannot Use Another Member's Personal Credential](git-credential-store.md#administrators-cannot-use-another-members-personal-credential)
- [Revocation Stops New Authorizations And Destroys Ciphertext](git-credential-store.md#revocation-stops-new-authorizations-and-destroys-ciphertext)
