# Identity & Access 核心测试用例

本目录是 Cloud identity-access 领域的 reviewer 风险导航图，镜像
`specs/decisions/cloud/identity-access/`。它记录 Gateway 外部身份归一、浏览器会话与内部凭据边界需要
长期守住的系统承诺，以及这些承诺是否已有直接、可信的可执行证据；它不是测试函数目录。

架构边界和不变量来自
[外部身份由 Gateway 归一，Cloud 保持授权权威](../../../decisions/cloud/identity-access/0-gateway-mediated-external-identity.md)。

用户托管的仓库凭据（HTTPS Git Token）于 2026-10-10 随四份 ADR 进入 `approved`，其核心用例登记在
[凭据元数据](repository-credential-metadata.md)、[凭据 Ingress 受限例外](repository-credential-ingress-exception.md)、
[凭据 Secret Store](repository-credential-secret-store.md) 与
[仓库凭据归属连续性](repository-credential-ownership-continuity.md)。能力词汇表（`clone` 的语义、空集合
的拒绝与迁移规则）于 2026-10-12 获批，登记在
[凭据能力词汇表](repository-credential-capability-vocabulary.md)。签名载荷**覆盖到哪些输入**、以及
"决定克隆什么的值必须被签名或被权威行复核"这一条，连同带凭据 clone 的分支未进签名这一已登记缺口，
见[授权载荷的覆盖范围](repository-credential-authorization-payload-coverage.md)。执行侧的身份隔离见
[Node 凭据执行身份隔离](../../node/repository/git-credential-execution-isolation.md)。

**当前实现状态（2026-10-12 本轮实现后）**：Cloud 侧的凭据元数据、用户自助 Ingress 与独立
Credential Service 已经落地，证据集中在 `integration.git_credentials_test.go`、
`integration.git_credential_ingress_test.go`、`integration.git_credential_binding_test.go`、
`internal.credentialstore.*` 与 `internal.config.*`。上一轮补上了**单次授权的 Cloud+Store 半程**：
授权载体与验签在 `internal.cloneauth`，Store 侧的兑换、原子消费与重放拒绝在
`internal.credentialstore`（`Resolver` 类型与 `authorization.go`）。本轮把**所需能力并入执行计划**：
准入要求引用声明 `clone`，规范集合随签名的 `cloneauth.Binding` 进入 `cloneexec.Job`，执行器再比对
一次；引用不允许该能力时以新的 `repository_credential_capability_unsupported` 明确拒绝，并与其他
确定性拒绝一样结算 Operation（见
[把带凭据的拒绝结算为终态](../../../decisions/cloud/operation/20261012-credentialed-clone-refusal-is-terminal.md)，
`proposed`）。

仍然 `Missing` 的是**执行侧**与**迁移一侧**：Cloud 的 `Issue` 还没有生产调用方（唯一合法调用点在
`requireRepositoryAccess` 的无条件拒绝之后），部署尚未签发 `broker-resolver` 证书，Node 侧的凭据
执行身份 C 与 clone 后清理未实现，因此私有仓库 clone 在所有路径上继续保持
`repository_credential_executor_unavailable` 拒绝。能力词汇表已于 2026-10-12 裁决并进入 `approved`：
词汇表保持 `["clone"]`，本次不引入 `read`、不开放 `push`，候选验证复用受限 `clone`；旧式
`capabilities=[]` 引用走**逐条核验 + 逐条显式迁移**（不得批量授权，无法证明授权依据者保持冻结），
实现与既有行的迁移落在解禁那次变更里。Issue 验收要求的"读取与推送能力分别验证"因此仍超出今天的
词汇表：推送一侧由独立的
[推送执行权限](../../../decisions/node/repository/20261012-push-execution-authority.md)（`approved` 2026-10-12，
E 不批准：`push` 仍不在词汇表中）承载，其核心用例登记在
[推送执行权限核心用例](../../node/repository/push-execution-authority.md)，除"推送不可被请求"一条外全部为
`Missing`。规则侧获批、Cloud 侧落地与新能力码都不解除任何拒绝。

## 阅读与维护规则

- 核心用例按业务风险组织，不按 Go package、单元测试或集成测试组织。
- 代表性证据写作 `包.测试函数`：`integration.*` 是 `integration/gateway_test.go` 中使用真实 HTTP、
  PostgreSQL、真实 Cloud router 与会校验 PKCE 的假 provider 的验收测试；`gateway.*` 与 `github.*` 是
  `internal/gateway` 及其 `github` 子包的单元测试。
- 只有在测试失败会直接说明某项验证义务不成立时，才把该测试列为代表性证据。
- `decisions/` 中状态为 `proposed` 的 ADR **不构成验收依据**：以它为决策依据的义务可以登记"实现先行"
  的证据（测试存在且在跑），但状态必须显式标注 `未裁决` 并写明承载它的 proposed ADR，且在该 ADR 变为
  `approved` 之前不得被当作正式验收结果引用，也不得计入 Issue 验收。ADR 状态变化时同步复核这些标注。
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
