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
权威绑定（`internal/core/repository_credentials.go`）。

**2026-10-13 更正：下面这句原先写成"授权的签发方还没有生产调用方"，它只对按凭据直连的 clone 成立。**
两条通道的签发方是分开的：`OperationClone` 的 `Issue`（`internal/core/credentialed_clone.go`）只在
`CLOUD_CLONE_DISPATCH_ENABLED=true` 时安装（该开关默认关闭），Node 侧认证通道未定义，因此**按凭据直连
的** clone 仍在每个部署上被拒绝；而镜像通道（`CLOUD_CLONE_DISPATCH_MIRROR_ENABLED`）有自己的签发方
`MirrorDispatcher.Prepare`（`internal/core/mirror_dispatch.go`，绑定 `OperationMirrorPrepare`），由
`cmd/server` 的镜像准备循环（`PrepareMirrorsOnce`）真实驱动。2026-10-13 的隔离验收里，它签发的授权由
凭据执行器在真实 Credential Service 上兑换（`ResolveMirror`）、抓取只承认 token 的私有上游、并销毁
材料——"签发 → 兑换 → 执行 → 销毁"这一段在镜像通道上已有端到端证据，见代码仓库
`docs/internal-git-server-mirror-acceptance.md`。

本文件中"元数据权威"一侧的义务多数已有直接证据；"执行绑定"一侧在 2026-10-12 由 `internal/core` 的
定向测试补上了 Cloud 侧的准入、结算与版本冻结证据，多数由 `Missing` 升为 `Partial`。**仍缺**的是
`OperationClone` 那一段（真实执行器执行 clone → 交付 checkout → 销毁）与协议冻结后的交接故障检查；
镜像通道那一段虽有端到端证据，但它只在验收驱动里被逐条断言，尚未把每一项登记进本文件的证据表。

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
- 带凭据引用的 clone 在引用**不允许本次操作所需能力**时也明确失败，错误码为稳定、可解释的
  `repository_credential_capability_unsupported`；空集合、未知成员、重复成员、大小写不同的拼写与
  非字符串成员一律读作"不允许任何操作"，不得被收窄成一个可用的集合。
- 部署级拒绝**优先于**能力拒绝：两者同时成立时报部署级错误码，因为它是用户无法通过换凭据绕开的那个。
- **不回退**匿名 clone，**不回退**部署静态凭据；受控 Node 上静态 PAT 不是用户凭据路径的回退。
- 拒绝**不改写**持久状态：`projects` 的归属与两列引用保持原样，不产生 `clone_executions` 行。
- 拒绝与规则侧获批相互独立：ADR 存在、Ingress 例外生效或测试文件存在都不解除该拒绝。
- 只有 [凭据 ADR D11-B](../../../decisions/cloud/identity-access/20261010-user-managed-git-credentials.md)
  与 Node 隔离 ADR D14-B 的**全部**条件有直接证据时，才允许移除该拒绝；移除必须由单独变更承载。

### 禁止结果

以匿名 Git "尽力而为"、以部署静态凭据代替用户凭据、把拒绝实现为静默排队、把凭据绑定写回或清空以
使请求通过、仅因 ADR 状态变为 `approved` 就删除拒绝分支、把空集合或含未知成员的能力集合读成"不受
限制"、把部署级拒绝与能力拒绝压成同一个错误码。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 缺执行器时拒绝且不产生 clone execution | `integration.repository_credentials_test.go` 的 `TestCredentialedCloneRefusesMissingExecutorAndAnonymousFallback` 用真实 HTTP、PostgreSQL 与生产 router 断言 `Drain` 失败且 `clone_executions` 为零；`integration.git_credential_binding_test.go` 的 `TestCredentialedCloneStaysRefusedOnEveryPath` 经 Controller 的 `Drain` 断言错误串携带 `repository_credential_executor_unavailable`，并用同一栈上的匿名 Project 作为正控制，证明拒绝来自凭据而不是栈本身不能 clone。`internal/core` 在准入层补上"不回退"的另一半：`TestAReferenceTheProjectCannotReachIsRefusedRatherThanFallingBackToAnonymous` 断言 schema 让 Project 无法绑定其他租户的引用（组合外键 `(credential_ref_id, tenant_id, owner_user_id)`），而被软删除的引用以 `repository_credential_unavailable` 被拒、执行器零调用且无 `clone_executions` 行——即"引用不可达"不会被读成"没有引用"从而退回匿名克隆；`TestARevokedReferenceIsRefusedAsRevokedBeforeTheDeploymentIsConsulted` 在**未安装执行器**的栈上断言已撤销引用得到 `repository_credential_revoked` 而不是部署级拒绝码（2026-10-14 更正：撤销此前与冻结共用 `repository_credential_frozen`，现按引用自身的 `availability` 取值分报，本行原先写的即旧码），`TestAFrozenReferenceIsRefusedBeforeItsScopeIsCompared` 在同一条记录上断言冻结先于作用域比较，两者共同固定了拒绝码的优先次序 | Covered |
| 拒绝不改写持久归属与引用 | 同上两个测试各自断言 Project 的 owner 与 `credential_ref_id` 未被改写；`internal/core` 的 `TestAReferenceTheProjectCannotReachIsRefusedRatherThanFallingBackToAnonymous` 断言拒绝后绑定仍在且 `clone_executions` 仍为零，`TestARevokedReferenceIsRefusedAsRevokedBeforeTheDeploymentIsConsulted` 断言撤销引用不产生执行记录 | Covered |
| 引用未声明所需能力时在准入处明确拒绝 | `internal/core` 的 `TestCredentialedCloneAdmissionRequiresTheReferencesCloneCapability` 用八行表断言：`["clone"]` 被准入并把规范集合交给执行器，而 `[]`、`["fetch"]`、`["push"]`、`["Clone"]`、`["clone","clone"]`、`["clone",7]`、`[""]` 各以 `repository_credential_capability_unsupported` 被拒且**从未到达执行器**；`TestADeploymentsOwnRefusalOutranksTheReferencesCapability` 断言部署级拒绝优先。执行器一侧的同一规则由 `internal/cloneexec` 的 `TestExecuteRefusesBeforeSpendingAnything` 覆盖（无能力、词汇表外能力、重复能力三种作业在花费任何资源前被拒） | Covered |
| 能力拒绝不改写持久状态并终结 Operation | `internal/core` 的 `TestAnUnsupportedCapabilitySettlesTheOperationToo` 断言拒绝码、执行器零调用、Operation 为 `failed` + 该码、`retry_at` 为空、无 `clone_executions` 行、绑定与 lifecycle 未被触碰 | Covered（**未裁决**：结算语义由 [把带凭据的拒绝结算为终态](../../../decisions/cloud/operation/20261012-credentialed-clone-refusal-is-terminal.md)（`proposed`）承载，批准前只作为"实现先行"证据，不计入验收结果） |
| 撤销终结 Operation，冻结只关闭新绑定而不终结 | `internal/core` 的 `TestARevokedReferenceSettlesTheOperationToo`（2026-10-14 补）用真实的 `WriteGitCredential` DELETE 路径撤销绑定中的引用后派发：断言拒绝码 `repository_credential_revoked` 逐字返回、执行器零调用、Operation 为 `failed` + 该码、`retry_at` 为空、无 `clone_executions` 行、绑定与 lifecycle 未被触碰；`TestRevokingACredentialClosesNewCloneBindings` 从撤销那一侧给出同一结论（真实撤销之后**新**的 clone 绑定以 `repository_credential_revoked` 在准入处被拒，本行原先写作 `repository_credential_frozen`）。冻结一侧的反向义务由 `TestSettlementIgnoresEveryRefusalButThePermanentOnes` 钉住：它逐条断言冻结与 scope 不匹配**不**写 `failed`——把 `repository_credential_frozen` 加进终态清单这一变异会让该用例失败，因此"暂时冻结被误当永久失效"是可证伪的 | Covered（**未裁决**：结算与撤销分报的语义由 [把带凭据的拒绝结算为终态](../../../decisions/cloud/operation/20261012-credentialed-clone-refusal-is-terminal.md)（`proposed`）承载，批准前只作为"实现先行"证据，不计入验收结果。真实栈上的撤销终态结算尚无证据，只有真实 PostgreSQL 支持的 core／DB 测试） |
| 无作用域的旧式 clone 准入被拒绝 | `integration.repository_credentials_test.go` 的 `TestProductionStoreRefusesUnscopedCloneAdmission` 断言 `runtime_scope_required` 且拒绝的请求未被持久化 | Covered |
| 解禁只能在 D11-B／D14-B 全部有证据之后 | 尚缺"解禁前后的门禁断言"测试；今天正确的行为正是继续拒绝，因此没有可指向的解禁证据。执行器与授权链路本轮已经落地，但 D11-B／D14-B 的**全部**义务仍不满足：checkout 尚未交付到 Workspace（D12 与验收义务 14 未实现），同一容器内的 W／C 分离不存在，"以 W 身份对 C 发起能力尝试"一类断言没有证据，超时与 `Interrupted` 两种结局的销毁也无直接证据。`repository_credential_executor_unavailable` 因此**在未安装任何通道的部署上**继续是带凭据 clone 的答案，且两个开关（`CLOUD_CLONE_DISPATCH_ENABLED` 与 `CLOUD_CLONE_DISPATCH_MIRROR_ENABLED`）都默认关闭。**2026-10-13 更正**：本行原先写作"继续保持无条件拒绝"，这对**启用镜像通道的部署**不成立——准入守卫的收尾判据是"两个通道都没有才拒绝"（`internal/core/repository_credentials.go`），因此被镜像通道服务的那一次 clone **会被准入**，并真的写下 `clone_executions` 行：`scripts/agent-read-acceptance.py` 在镜像栈上按 `workspace_id` 读回这次 clone 的 `result->>'path'`（`outcome='clone_ready'`）与 `input->>'repositoryUrl'`（内部 Git Server 的镜像地址）。这**不是**本项义务的例外：解禁所指的是"按凭据直连"的那条路径，镜像通道不把上游凭据交给 Node，因此 D11-B／D14-B 中任何以"执行器按凭据执行 clone 并把 checkout 交给 Workspace"为前提的义务它一条都不满足 | Missing |
| 能力词汇表本身是否扩展 | 词汇表今天只有 `clone`；Issue 验收要求的"读取与推送能力分别验证"与 P2 的候选验证能力都超出它，且被 approved 的凭据 ADR D2 明确排除。[仓库凭据能力词汇表](../../../decisions/cloud/identity-access/20261012-repository-credential-capability-vocabulary.md)已于 2026-10-12 **批准**（A–D 按推荐批准，E 有条件批准）：词汇表保持 `["clone"]`、不引入 `read`、不开放 `push`、候选验证复用受限 `clone`，旧式空集合引用走逐条核验后的显式迁移。词汇表自身的承诺登记在[凭据能力词汇表](repository-credential-capability-vocabulary.md)，本文件不重复；Issue 验收第 2 项的**推送一侧**因此继续为 `Missing`（由 [推送执行权限](../../../decisions/node/repository/20261012-push-execution-authority.md)（`approved` 2026-10-12，E 不批准）承载，词汇表扩展仍待独立决策，实现前不计入验收） | Missing |
| 部署托管引用（`capabilities` 为空）在执行器安装后的准入 | `internal/core` 的 `TestADeploymentManagedReferenceIsRefusedForItsEmptyCapabilityOnceAnExecutorExists`（2026-10-13 补上）在**已安装执行器**的栈上走真实 `clone_dispatch`：引用由 `core.Store.ConfigureCredential` **真的创建**（不是手工 UPDATE 出来的形状），因此读到的是部署路径实际写下的三列——`attribution='unknown'`、`capabilities` 取列默认的 `[]`、`repository_scope` 为 NULL，测试逐项读回而不假设。断言该引用以 `repository_credential_capability_unsupported` 被拒（**不是**部署级码，这正是本行原先缺的区分）、执行器零调用、`clone_executions` 零行；并先断言 owner 在场且活跃、执行器已安装，否则"被拒"可能出自冻结或部署级条款。对照是同一写入路径上加授 `["clone"]` 的引用，它被准入且执行器恰好一次调用——两处各自独立场景（受控租约与被结算为 `failed` 的 Operation 都不能复用）。**边界照实写明**：本行钉的是"引用侧拒绝在部署级拒绝不再命中时仍然成立"，它**不**表示解禁可以开始——解禁义务仍在下一行，且能力列的历史迁移义务（既有 `'[]'` 引用逐条核验）仍待解禁那次变更显式处理 | Covered |

### 决策依据

凭据 ADR D11-A、D11-B 与不变量 13；Node 隔离 ADR D14-A、D14-B；仓库凭据连续性 D4（"执行计划固定引用
身份、版本、仓库与所需能力……能力不支持时明确拒绝"）——"引用未声明能力即拒绝"这一条由**已批准**的连续
性 D4 承载，因此它的准入证据是正式验收结果。**"该拒绝终结 Operation"这一条不同**：它由
[把带凭据的拒绝结算为终态](../../../decisions/cloud/operation/20261012-credentialed-clone-refusal-is-terminal.md)
（`proposed`）承载，实现已先行，因此本文件只在证据表里如实登记并加 `未裁决` 标注，不把它记为正式验收
结果。能力词汇表已于 2026-10-12 批准并进入 `approved`（见
[仓库凭据能力词汇表](../../../decisions/cloud/identity-access/20261012-repository-credential-capability-vocabulary.md)）；
词汇表自身的承诺与其迁移义务登记在
[凭据能力词汇表](repository-credential-capability-vocabulary.md)，本条只登记今天的判定与证据。

## Credential Binding Is Frozen Per Clone Execution With Its Version

### 风险

一次 clone execution 在中途改用另一版本或另一条凭据（用户轮换、撤销或成员变更之后），使"这次执行
用的是哪条凭据"不可回答，并让轮换期间的结算无法解释。

### 前置状态

一次带凭据的 clone execution 已被受理并绑定了 `credential_ref_id` 与 `credential_ref_version`；
该凭据随后发生版本变化（轮换）或被冻结。

（2026-10-13：这个"已被受理并绑定"的前置状态现在由一个**真实派发**产生，而不是由夹具写入——
`workspaceCloneDispatch` 在登记执行行的同一个事务里写下准入所用的那一对
（`cloud/internal/core/workspace_clone.go`），值取自 `requireRepositoryAccess` 在这一事务里
记下的准入结果。）

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
| 绑定必须成对（id 与版本同时存在或同时为空） | `core.TestAnExecutionCannotRecordHalfACredentialBindingAndRevocationLeavesItAlone`：先写入一行同时带 id 与版本的执行（证明该形态可记录），再分别尝试只带 id 与只带版本的两行，两者都必须被拒，且断言拒绝来自 `execution_credential_binding` 这个具名约束（PostgreSQL 在错误里给出约束名），从而排除"因别的原因恰好失败"。约束被删除或放宽时该测试直接失败。**2026-10-13 补同一约束在真实派发上的两侧**：`core.TestACredentialedCloneRecordsTheCredentialVersionThatAdmittedIt` 的"the registration carries the reference and the version admission validated"一例从**真实 `clone_dispatch` 写下的执行行**读回两条都非空的绑定，"an anonymous clone records no binding at all"一例从同为真实派发的匿名执行行读回两条都为空——即两个合法形态各有一条由真实写入点产生的证据，而不是只有夹具能造出它们 | Covered |
| 轮换与冻结不重写已受理执行的绑定 | 同一测试的后半段：把引用推进版本并置为 `invalid`／`revoked_by_owner` 之后，原本记录 `credential_ref_version=1` 的执行行仍然匹配，引用侧的变更没有级联进执行记录。另有否定性核对：`clone_executions` 的全部写入点只有三处 `UPDATE`（`internal/core/clone.go`、`internal/core/credentialed_clone.go` 写 `result`，`internal/core/runtime_force_stop.go` 写 `terminated_by_force_stop_id`）与两处 `INSERT`，**没有任何代码路径或迁移改写已存在行的 `credential_ref_id`／`credential_ref_version`**。`core.TestRotatingACredentialLeavesTheAcceptedExecutionsBindingAndSettlementIntact`（2026-10-12 补，本轮实际执行）经**真实轮换 API**（`PUT` 把版本 1→2）驱动一条已受理执行，断言执行行的见证字节不变、`credential_ref_version` 仍为 1，并断言该执行随后正常结算。**2026-10-13 补齐原先缺的那一半（真实写入点，不再只有夹具）**：`core.TestACredentialedCloneRecordsTheCredentialVersionThatAdmittedIt` 的"a later rotation leaves the recorded binding alone"一例，先由真实 `clone_dispatch` 写下执行行，再走真实轮换 API，从持久状态读回绑定仍是准入那一对；且隔离栈上的端到端验收把同一条义务跑在**真实 Cloud 服务进程、真实凭据、真实执行器**上（`scripts/agent-read-acceptance.py` 的最后一项断言，证据 `.acceptance/private-mirror-agent-read.json`：轮换后凭据自身 `version=2`，而 Project 与 Agent 运行两次 clone 的执行登记都仍是版本 `1`，见 `docs/internal-git-server-mirror-acceptance.md` §2）。原先记为缺口的那半句——"生产上还没有真实执行器产出的带凭据执行行"——现已由镜像通道的真实派发填上；剩下的边界是这条通道在隔离栈上启用、生产开关仍关闭，以及"按凭据直连"的执行器通道仍未跑通（与本行的义务无关，属执行侧） | Covered |
| 在途执行按原版本结算 | **Cloud 结算侧已有直接证据（2026-10-12 补，本轮实际执行）**：`core.TestRevokingACredentialLeavesTheAcceptedExecutionAndItsSettlementIntact` 与 `core.TestRotatingACredentialLeavesTheAcceptedExecutionsBindingAndSettlementIntact` 分别在真实 `DELETE`（撤销）与真实 `PUT`（轮换）之后，断言已受理执行仍按它记录的 `credential_ref_version=1` 结算（推进基线、operation 置 `succeeded`），且执行行的结果字节与 `updated_at` 不被引用侧变更改写。**2026-10-13 更正**：本行原先写"真实 Store 与执行器那一段（兑换 → 执行 → 销毁）尚不存在"，这对**镜像通道**已不成立——隔离栈上真实 Cloud 服务进程签发、真实 Credential Service 校验并原子消费、真实执行器抓取只承认 token 的私有上游、随后销毁材料（`docs/internal-git-server-mirror-acceptance.md`）。**仍缺的一半**是"按凭据直连"的那条执行路径（`OperationClone` 的兑换 → clone → checkout 交付 → 销毁），它与本行的版本冻结义务不是同一段，因此状态仍为 Partial | Partial |
| 冻结后不再签发新授权、已签发授权随到期失效 | 冻结一侧由 `integration.repository_credentials_test.go` 的归属连续性测试覆盖状态迁移；`core.TestRevokingACredentialClosesNewCloneBindings`（2026-10-12 补，本轮实际执行）在真实撤销之后断言**新**的 clone 绑定以 `repository_credential_frozen` 在准入处被拒、执行器零调用、`clone_executions` 零行——即撤销确实关闭新绑定，且关闭发生在任何登记或派发之前。同类证据本轮补上"冻结"这一来源：`core.TestASweepFrozenReferenceIsRefusedAtAdmission` 用归属连续性扫描（而不是撤销）冻结一行，再走已装载执行器的 `clone_dispatch`，断言 `repository_credential_frozen`、执行器零调用、`clone_executions` 零行——扫描产出的冻结与撤销产出的冻结在准入处得到同一个拒绝码，谁冻结不影响结论。**签发侧要按绑定分开看（2026-10-13 更正，原先整句写作"Cloud 的 `Issue` 没有生产调用方"）**：`OperationClone` 的 `Issue` 确实没有生产调用方，但 `OperationMirrorPrepare` 的签发方有——`core.MirrorDispatcher.Prepare`（`internal/core/mirror_dispatch.go`）由 `cmd/server` 的 `PrepareMirrorsOnce` 循环真实驱动。因此"对已冻结凭据签发授权"在镜像通道上可断言，且已有断言：`core.TestOnlyADurableRefusalEndsTheOperation` 的 "a reference that is frozen" 一例走真实的 `PrepareMirrorsOnce`，断言**零次派发**（`len(preparer.calls)==0`）且不留准备行——冻结的引用没有走到签名那一步。隔离验收另给出一条端到端旁证：真实 Cloud 服务进程签发的授权被真实 Credential Service 兑换、抓取只承认 token 的私有上游成功。相邻证据是 `TestResolveRejectsARevokedCredential`：即便授权本身有效，Store 也拒绝兑换已撤销凭据的秘密——这是 Store 侧的纵深防御，不等于 Cloud 侧的冻结语义。**仍缺**："已签发授权随到期失效"仍无端到端演练（Cloud 侧只有"授权寿命不得超过准备租约"这条签发期拒绝，Store 侧只有按自己时钟判过期），且上述两条证据都属于镜像通道；`OperationClone` 一侧依旧无从断言。本行保持 Partial | Partial |

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
| 既有迁移不被改写 | **已有显式的校验和回归断言（2026-10-12 补）**：`core.TestTheAppliedCredentialMigrationsAreByteForByteImmutable` 把 `0001_core.sql`／`0023_repository_credential_continuity.sql`／`0033_user_git_credentials.sql` 的 sha256 钉在测试里，任何人改写这三个文件该测试即失败（本轮另用 `sha256sum` 独立核对了三点摘要，逐字一致）。仍是 `Partial`：它固定的是"当前检入的文件与钉子一致"，没有演练迁移器启动时对**已应用**迁移的校验和拒绝，且钉子与被钉的文件同仓同改——完整系统承诺仍缺这一半 | Partial |
| 新迁移不改变既有行语义 | `integration.runtime_upgrade_test.go` 的 `TestRuntimeUpgradeOnlyBackfillsUniqueCreationEvidenceAndKeepsUnknownReferences` 断言升级不猜测 `attribution`、不重写绑定；`0033` 只增加 `name`（`NOT NULL DEFAULT ''`）与 `revoked_at`（可空）两个可加列，不需要回填，`integration.git_credentials_test.go` 的 `TestGitCredentialRevokeIsAuthoritativeAndIdempotent` 断言撤销不写语义未定义的 `deleted_at`；覆盖面仍然有限 | Partial |

### 决策依据

凭据 ADR D10 与"为什么不能直接改写"；归属连续性 ADR D1、D7。
