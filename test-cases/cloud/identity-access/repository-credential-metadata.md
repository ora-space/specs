# 用户托管仓库凭据的元数据与逐次授权

依据[用户自助 HTTPS Git 凭据](../../../decisions/cloud/identity-access/20261010-user-managed-git-credentials.md)
（`approved`）与其继承的
[仓库凭据归属连续性](../../../decisions/cloud/identity-access/20261010-repository-credential-ownership-continuity.md)。
本文件登记 Cloud 侧的凭据**元数据权威、逐次授权与 fail-closed**：谁是凭据归属的权威、`secret_ref`
为什么不是 bearer token、凭据引用如何绑定到一次 clone execution、以及
`repository_credential_executor_unavailable` 在什么条件下才允许移除。

秘密本身的承载与兑换在
[凭据 Secret Store](repository-credential-secret-store.md)，Token 的接收面在
[凭据 Ingress 受限例外](repository-credential-ingress-exception.md)，执行侧身份在
[Node 凭据执行身份隔离](../../node/repository/git-credential-execution-isolation.md)。

**当前实现状态（2026-10-12 本轮实现后）**：Cloud 侧已有凭据元数据（`internal/core/git_credentials.go`、
迁移 `0033_user_git_credentials.sql`）、用户自助 Ingress、独立 Credential Service
（`internal/credentialstore/`、`cmd/credential-store`）、单次授权的签发与兑换
（`internal/cloneauth/` 与 `internal/credentialstore/authorization.go`）与 `projects.credential_ref_id`
权威绑定（`internal/core/repository_credentials.go`）。仍然**没有**可用的凭据执行器：授权的签发方
（Cloud 的 `Issue`）还没有生产调用方，Node 侧认证通道未定义，因此凭据 clone 在每个执行路径上继续被
拒绝。本文件中"元数据权威"一侧的义务多数已有直接证据，"执行绑定"一侧仍然 `Missing`。

## Credential Metadata Is The Only Authority And Secret Reference Is Not A Bearer Token

### 风险

`secret_ref` 被当作可直接取用秘密的凭据，或凭据归属被复制出第二处权威（Store、Node、事件载荷），
使撤销与冻结在某个副本上失效。

### 前置状态

`credential_refs` 存在带 `owner_user_id`、`tenant_id`、`attribution`、`availability`、`version`、
`repository_scope`、`secret_ref` 的记录；`projects` 同时存在 `credential_ref_id`（`0001`，V1 权威列）
与 `repository_credential_ref_id`（`0023`，保留不删、V1 只读）。

### 触发

读取凭据元数据 API；读取 Project 的凭据引用；构造两列同时非空且不一致的 Project；用同一 `secret_ref`
字符串尝试在另一租户或另一用户下创建引用；在冻结之后查询作为凭据使用的可行性。

### 必须成立

- `credential_refs` 是凭据元数据、归属与状态的**唯一**权威记录；Store 不维护第二份归属结论。
- V1 的权威绑定列是 `projects.credential_ref_id`，两条读路径（准入判定与快照投影）结论一致；
  `repository_credential_ref_id` 只读、不参与回退、不被隐式取值。
- 两列同时非空且不一致时**显式拒绝**该次操作并返回可诊断原因码，不静默择一。
- `secret_ref` 是不透明定位符：持有它不构成取用秘密的能力，取用需要服务身份加 Cloud 签发的单次授权。
- 凭据归属与冻结只由 Cloud 的 PostgreSQL 状态与已验证 claim 决定，不来自调用方选择的字段；
  绑定与使用都强制 `owner_user_id` 与调用者一致。

### 禁止结果

把 `secret_ref` 当 PAT 使用或返回给客户端；在 Store 侧另建一份归属表；让"由哪一列决定凭据"取决于
读路径；以 `repository_credential_ref_id` 覆盖或回退到 `credential_ref_id`；靠应用层拦截替代
`credential_ref_id` 外键已有的 owner 强制。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| `secret_ref` 不出现在公开响应、业务事件或元数据 API 输出 | 公开响应：`integration.git_credentials_test.go` 的 `TestGitCredentialCreateHandsTheTokenOverAndKeepsOnlyMetadata`、`TestGitCredentialListIsScopedToTheOwningUser` 与 `TestGitCredentialRotateReplacesMaterialBeforeAdvancingTheVersion` 对响应**字节**断言既无 Token 也无服务生成的引用，且 `secretRef` 字段不存在；Controller 投影：`integration.git_credential_binding_test.go` 的 `TestCredentialedCloneStaysRefusedOnEveryPath` 断言 `secretRef` 与 `repositoryCredentialRefId` 在投影中显式为 null。业务事件路径尚无断言 | Partial |
| 持有 `secret_ref` 无法换取秘密 | `internal.credentialstore` 的 `TestServerEnforcesCapabilitySeparation`（`ingress-writer` 被拒绝 resolve，`broker-resolver` 被拒绝写入／轮换／撤销）、`TestServerRefusesClientWithoutCertificate`、`TestServerRefusesUnknownClientIdentity`、`TestPresentedIdentityRequiresVerifiedPeer` 与 `TestServerRefusesCallerSuppliedReference`（调用方自选的 `secret_ref` 被拒绝而不是被采用）。兑换侧本轮补齐：知道凭据标识并**不能**换取秘密，还需要一份 Cloud 用 Ed25519 签名的单次授权——`TestResolveRejectsAForgedAuthorization`（持有 `broker-resolver` 证书的一方自造授权被拒）、`TestResolveRejectsUnknownAndMalformedCredentialIds`（无授权、空授权、畸形授权一律拒绝）、`TestResolveIsUnavailableWithoutAVerificationKey`（未配置验签公钥的部署没有兑换能力）。见[凭据 Secret Store](repository-credential-secret-store.md) | Covered |
| 权威列统一且回退已删除 | `internal/core/repository_credentials.go` 的 `requireRepositoryAccess` 不再优先或回退 `repository_credential_ref_id`，`internal/core/control.go` 的 `snapshot()` 同样只读 `credential_ref_id`；`integration.git_credential_binding_test.go` 的 `TestProjectBindingAcceptsOnlyACredentialThatCouldClone` 断言绑定落在 `credential_ref_id`，`TestCredentialBindingConflictIsRefusedRatherThanResolved` 的"两列一致"子测试断言一致时不再产生冲突码，`TestCredentialedCloneStaysRefusedOnEveryPath` 断言两条读路径给出同一个结论 | Covered |
| 两列冲突显式拒绝 | `integration.git_credential_binding_test.go` 的 `TestCredentialBindingConflictIsRefusedRatherThanResolved`：两列指向不同凭据时以 `repository_credential_binding_conflict` 拒绝、不产生 `clone_executions` 行、两列都不被改写 | Covered |
| 双列在升级中不被猜测或改写 | `integration/runtime_upgrade_test.go` 的 `TestRuntimeUpgradeOnlyBackfillsUniqueCreationEvidenceAndKeepsUnknownReferences` 断言升级不猜测有效绑定列 | Partial |
| 归属只来自已验证 claim 与当前状态 | `integration.git_credential_ingress_test.go` 的 `TestGitCredentialIngressIsolatesOwnersAndTenants`（另一用户同租户为 404、另一租户为 403）与 `TestGitCredentialIngressAcceptsTheTokenInNoOtherPosition`（`tenantId`、`ownerUserId`、`userId`、`id`、`availability`、`capabilities`、`secretRef` 一律以 `unknown_field` 拒绝）；`TestGitCredentialListIsScopedToTheOwningUser` 断言列表只含调用者自己的 `personal` 凭据；冻结一侧由 `integration.repository_credentials_test.go` 的 `TestCredentialDepartureFreezesPersonalAndUnknownButRetainsTeamHistory` 断言 | Covered |

### 决策依据

凭据 ADR D1、D7–D10，不变量 3、4；`cloud/AGENTS.md` 关于授权只来自已验证 claim 的规则；
`internal/core/repository_credentials.go`（准入拒绝路径）与 `internal/core/control.go` 的 `snapshot()`
（快照投影）。

## Credentialed Clone Refuses A Missing Executor And Never Falls Back

### 风险

凭据 clone 在没有可用执行器或凭据执行身份时退回匿名 Git 或部署静态凭据，静默地以错误身份访问私有
仓库——既可能失败得不可解释，也可能在共享凭据下泄露他人仓库的访问。

### 前置状态

Project 通过 `credentialRefId` 绑定一条可用凭据；受控 Node 上没有实现受作用域凭据的执行器；
`repository_credential_executor_unavailable` 的无条件拒绝处于生效状态。

### 触发

经生产入口为该 Project 请求 clone 并推进受理；检查是否产生任何 `clone_executions` 行、是否改写
Project 的持久引用，以及对外错误形状。

### 必须成立

- 带凭据引用的 clone 在缺少执行器时**明确失败**，错误码为稳定、可解释的
  `repository_credential_executor_unavailable`。
- **不回退**匿名 clone，**不回退**部署静态凭据；受控 Node 上静态 PAT 不是用户凭据路径的回退。
- 拒绝**不改写**持久状态：`projects` 的归属与两列引用保持原样，不产生 `clone_executions` 行。
- 拒绝与规则侧获批相互独立：ADR 存在、Ingress 例外生效或测试文件存在都不解除该拒绝。
- 只有 [凭据 ADR D11-B](../../../decisions/cloud/identity-access/20261010-user-managed-git-credentials.md)
  与 Node 隔离 ADR D14-B 的**全部**条件有直接证据时，才允许移除该拒绝；移除必须由单独变更承载。

### 禁止结果

以匿名 Git "尽力而为"、以部署静态凭据代替用户凭据、把拒绝实现为静默排队、把凭据绑定写回或清空以
使请求通过、仅因 ADR 状态变为 `approved` 就删除拒绝分支。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 缺执行器时拒绝且不产生 clone execution | `integration.repository_credentials_test.go` 的 `TestCredentialedCloneRefusesMissingExecutorAndAnonymousFallback` 用真实 HTTP、PostgreSQL 与生产 router 断言 `Drain` 失败且 `clone_executions` 为零；`integration.git_credential_binding_test.go` 的 `TestCredentialedCloneStaysRefusedOnEveryPath` 经 Controller 的 `Drain` 断言错误串携带 `repository_credential_executor_unavailable`，并用同一栈上的匿名 Project 作为正控制，证明拒绝来自凭据而不是栈本身不能 clone | Covered |
| 拒绝不改写持久归属与引用 | 同上两个测试各自断言 Project 的 owner 与 `credential_ref_id` 未被改写 | Covered |
| 无作用域的旧式 clone 准入被拒绝 | `integration.repository_credentials_test.go` 的 `TestProductionStoreRefusesUnscopedCloneAdmission` 断言 `runtime_scope_required` 且拒绝的请求未被持久化 | Covered |
| 解禁只能在 D11-B／D14-B 全部有证据之后 | 尚缺"解禁前后的门禁断言"测试；今天正确的行为正是继续拒绝，因此没有可指向的解禁证据。执行器与授权链路本轮已经落地，但 D11-B／D14-B 的**全部**义务仍不满足：checkout 尚未交付到 Workspace（D12 与验收义务 14 未实现），同一容器内的 W／C 分离不存在，"以 W 身份对 C 发起能力尝试"一类断言没有证据，超时与 `Interrupted` 两种结局的销毁也无直接证据。`repository_credential_executor_unavailable` 因此**继续保持无条件拒绝**，且 `CLOUD_CLONE_DISPATCH_ENABLED` 默认关闭 | Missing |

### 决策依据

凭据 ADR D11-A、D11-B 与不变量 13；Node 隔离 ADR D14-A、D14-B。

## Credential Binding Is Frozen Per Clone Execution With Its Version

### 风险

一次 clone execution 在中途改用另一版本或另一条凭据（用户轮换、撤销或成员变更之后），使"这次执行
用的是哪条凭据"不可回答，并让轮换期间的结算无法解释。

### 前置状态

一次带凭据的 clone execution 已被受理并绑定了 `credential_ref_id` 与 `credential_ref_version`；
该凭据随后发生版本变化（轮换）或被冻结。

### 触发

在途执行继续推进；随后对同一条凭据执行轮换与冻结；再查询该执行的绑定与结算依据。

### 必须成立

- 执行绑定的是**具体版本**，不是"当前凭据"；`clone_executions` 的两列成对存在或成对为空
  （`execution_credential_binding` 约束），不存在只有 id 没有版本的绑定。
- 轮换期间的**在途**执行按原版本结算；轮换不重写已受理执行的绑定。
- 冻结与新授权：冻结后不再签发新授权；已经签发但未使用的授权随到期失效，不因冻结而被"续命"。
- 版本变化可被 fence：并发的旧版本写入被执行的身份与版本绑定拒绝。

### 禁止结果

执行时解析"当前版本"、轮换时改写在途执行的绑定、冻结后继续放行已签发的旧授权、把版本绑定实现为
仅记录而不参与校验。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 绑定必须成对（id 与版本同时存在或同时为空） | `0023` 的 `execution_credential_binding` 约束在 schema 上成立；尚缺直接断言该约束拒绝半绑定的测试 | Missing |
| 在途执行按原版本结算 | 需要真实 Store 与执行器，尚不存在 | Missing |
| 冻结后不再签发新授权、已签发授权随到期失效 | 冻结一侧由 `integration.repository_credentials_test.go` 的归属连续性测试覆盖状态迁移。签发与到期语义仍无直接证据：授权的签发与验签本轮已实现（`internal.cloneauth`）且 Store 侧的到期与单次消费已有测试（含跨真实进程的重放拒绝与"记录损坏即读作已消费"的失败语义），但 Cloud 的 `Issue` **没有生产调用方**——它唯一的合法调用点在 `requireRepositoryAccess` 的无条件拒绝之后，因此不存在"对已冻结凭据签发授权"这条可断言的路径，也没有"已签发授权随到期失效"的端到端演练。相邻证据是 `TestResolveRejectsARevokedCredential`：即便授权本身有效，Store 也拒绝兑换已撤销凭据的秘密——这是 Store 侧的纵深防御，不等于 Cloud 侧的冻结语义 | Missing |

### 决策依据

凭据 ADR D4、D5、D11-B 第 5 项，不变量 5、7、9；归属连续性 ADR D3。

## Credential Metadata Changes Do Not Rewrite Applied Migrations

### 风险

为引入凭据元数据的新语义而修改 `0001`／`0023`，使已部署实例与仓库文本分叉，并让 `credential_refs`
的既有行失去唯一解释。

### 前置状态

`0001_core.sql` 与 `0023_repository_credential_continuity.sql` 已在所有环境应用；仓库中存在解释其
语义的 ADR。

### 触发

需要为凭据引入新列、新状态值或新解读时；或有人以"顺便整理"为由编辑既有迁移。

### 必须成立

- 新语义只经**前向**迁移引入，并说明对既有行的处理；已应用迁移逐字不改。
- `credential_refs` 的既有列与约束的语义不变；不得为方便实现而放宽 `attribution`、`availability`
  的取值集合或删除既有约束。
- 新列若与既有列构成权威冲突（如两个有效绑定列），必须由 ADR 明确裁决并配拒绝路径，而不是隐式择一。

### 禁止结果

编辑 `0001`／`0023`、删除既有 CHECK 或外键、把新状态塞进既有 `availability` 取值而改变其含义。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 既有迁移不被改写 | 尚缺显式的迁移校验和回归断言；当前依赖评审与既有迁移约定。本轮新增的 `0033_user_git_credentials.sql` 只做前向追加，`0001`／`0023` 逐字未动，但这是差异评审而非测试证据 | Missing |
| 新迁移不改变既有行语义 | `integration.runtime_upgrade_test.go` 的 `TestRuntimeUpgradeOnlyBackfillsUniqueCreationEvidenceAndKeepsUnknownReferences` 断言升级不猜测 `attribution`、不重写绑定；`0033` 只增加 `name`（`NOT NULL DEFAULT ''`）与 `revoked_at`（可空）两个可加列，不需要回填，`integration.git_credentials_test.go` 的 `TestGitCredentialRevokeIsAuthoritativeAndIdempotent` 断言撤销不写语义未定义的 `deleted_at`；覆盖面仍然有限 | Partial |

### 决策依据

凭据 ADR D10 与"为什么不能直接改写"；归属连续性 ADR D1、D7。
