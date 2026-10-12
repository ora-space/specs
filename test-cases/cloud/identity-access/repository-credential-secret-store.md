# 独立 Credential Service 的秘密承载与单次授权

依据[用户 Git Token 由独立可替换的 Credential Service 承载](../../../decisions/cloud/identity-access/20261010-user-managed-git-secret-store.md)
（`approved`）。本文件登记 Store 侧需要长期守住的承诺：服务身份分权、Token 的加密与密钥归属、
Cloud 与 Store 跨两个存储的一致性、**兑换时的原子消费**，以及审计与备份不泄露秘密。

**当前实现状态（2026-10-10 本轮实现后）**：独立可替换的 Credential Service 已实现，位于
`cloud/internal/credentialstore/`（`store.go` 卷与记录、`keystore.go` DEK／KEK 包装、`record.go`
持久格式、`identity.go` 能力表、`server.go` mTLS 端点、`authorization.go` 兑换、`client.go` Cloud 的
ingress-writer 客户端）并由 `cloud/cmd/credential-store` 承载，部署门禁与配置在
`cloud/internal/config/credential_store.go`。服务身份分权、密文与 KEK 分离、创建／轮换／撤销的写入面与
审计脱敏已有直接证据。

**本轮（2026-10-12）新增了兑换侧的 Cloud+Store 半程**：单次授权的载体与验签在
`cloud/internal/cloneauth/`，Store 侧的兑换、原子消费与重放拒绝在
`cloud/internal/credentialstore/authorization.go`。两份 ADR 都没有规定授权的密码学形态，而 D10 又禁止
签发方在 Store 预登记消费记录，因此唯一可行的形态是 Cloud 用 Ed25519 签名、Store 只持公钥验签——否则
任何持有 `broker-resolver` 证书的进程都能自造授权去兑换任意租户的凭据（D14-B 第 11 项）。

**本轮后续（同日）补齐了三项授权安全验收**：消费记录在崩溃、写入失败与"记录损坏"下的语义（损坏即读作
已消费、写不下即不交付、交付前失败留下终态 `consumed_not_delivered`）、Cloud 签名私钥的归属与轮换／
失效机制（验签文件可同时携带多把公钥，Store 在启动时一次性读入，`MaxLifetime` 使"移除 + 重启"这种失效
方式足够），以及**跨进程验收**：`cloud/internal/credentialstore/process_test.go` 编译真实的
`cmd/credential-store`，用真实配置文件、真实 PKI、真实 KEK keyring 与真实 Ed25519 密钥对启动子进程，
在真实双向认证监听上验证"一份授权恰好换到一个 Token、重放被拒、**进程重启后仍被拒**、审计日志不含
Token 与授权载体"。

仍然 `Missing` 的是：**签发侧没有生产调用方**（唯一合法调用点在 `requireRepositoryAccess` 的无条件拒绝之后，
本轮 fail-closed 不变，因此也不配置 Cloud 签名私钥，`Issue` 只由测试驱动；跨进程验收里的"签发方"是 Cloud
的真实签发代码而不是真实的 Cloud 服务进程）、**部署尚未签发 `broker-resolver` 证书**（部署栈上
`resolve` 依旧不可达，未配置验签公钥的部署继续以 `authorization_verification_unavailable` 拒绝每一次
兑换）、**Node 侧执行身份 C 与凭据清理**、以及**备份／恢复演练**。Cloud 侧的元数据权威与 fail-closed 见
[凭据元数据](repository-credential-metadata.md)，接收面见
[凭据 Ingress 受限例外](repository-credential-ingress-exception.md)，消费侧强制见
[Node 凭据执行身份隔离](../../node/repository/git-credential-execution-isolation.md)。

## Store Accepts Only Service Identities And Splits Write From Resolve

### 风险

Store 接受终端用户身份或 Cloud 的业务身份直接读写秘密，或同一身份既能写入又能解析，使"写入面"与
"取用面"合并成一个可以被单点攻破的能力。

### 前置状态

Store 定义了 `ingress-writer` 与 `broker-resolver` 两个服务身份，各有独立的客户端凭据；
Cloud 的业务进程与 Node 的管理身份 M 都不持有任何解析身份。

### 触发

以终端用户身份调用 Store；以缺失或伪造的服务身份调用；用 `ingress-writer` 尝试解析；用
`broker-resolver` 尝试写入或撤销；检查 Cloud 业务进程与 M 的凭据集合。

### 必须成立

- Store 只接受服务身份，不接受终端用户身份；身份校验在 Store 侧完成，不依赖调用方声明。
- 写入／撤销与解析分权：`ingress-writer` 不能解析，`broker-resolver` 不能写入或撤销。
- `broker-resolver` 由凭据执行身份 **C** 持有，不由 M 持有；Cloud 的业务进程与 M 都不解析秘密。
- 解析必须同时满足"服务身份正确"**且**"持有 Cloud 签发的单次授权"；任一缺失即拒绝。

### 禁止结果

让 Cloud 业务进程持有解析身份、让 M 持有 `broker-resolver`、以网络位置或共享密钥代替服务身份、
把"知道 `secret_ref`"当作解析权限。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 只接受服务身份且写入／解析分权 | `internal.credentialstore` 的 `TestServerEnforcesCapabilitySeparation`（七个组合：ingress 可写、ingress 被拒绝 resolve、resolver 被拒绝写入／轮换／撤销、keys 身份被拒绝写入、resolver 的 resolve 在未配置验签公钥的部署上返回 `authorization_verification_unavailable`）、`TestServerRefusesClientWithoutCertificate`、`TestServerRefusesUnknownClientIdentity`、`TestPresentedIdentityRequiresVerifiedPeer`、`TestServerDoesNotRequireTLSToEmitAnAnonymousRefusal`、`TestIdentitiesCapabilityMapping`、`TestNewIdentitiesValidatesConfiguration`、`TestClientHoldingTheWrongIdentityIsRefused`；兑换能力所在的 `Resolver` 是**独立类型**且只能经 `Store.Resolver()` 取得（`TestResolveIsUnavailableWithoutAVerificationKey`、`internal.config` 的 `TestCredentialStoreServiceResolveCapabilityFollowsItsKey`）。分权在**真实进程**上另有一次验收：`process_test.go` 的 `TestCredentialStoreProcessEnforcesCapabilitySeparation`（一份有效授权对 `ingress-writer` 身份毫无价值且不消耗它，`broker-resolver` 身份的写入被拒）与 `TestCredentialStoreProcessRefusesAnAnonymousCaller`（无证书的客户端在握手阶段就被拒，根本到不了能力判定） | Covered |
| `broker-resolver` 由 C 持有而非 M | 需要 Node 侧身份与文件权限的联合断言，见[Node 凭据执行身份隔离](../../node/repository/git-credential-execution-isolation.md)；Cloud 侧不产生该证据，本轮也未在部署中签发 `broker-resolver` 证书。跨进程验收证明了"持有该证书才能兑换"，但没有证明"只有 C 能持有它" | Missing |
| 解析需服务身份加授权双条件 | 两侧都有直接证据。服务身份一侧：`TestServerEnforcesCapabilitySeparation`（`ingress-writer` 被拒绝 resolve，`broker-resolver` 通过身份校验），`TestResolveIsUnavailableWithoutAVerificationKey`（Store 没有验签公钥就没有兑换能力，每一次兑换都被拒绝）。授权一侧：`TestResolveRejectsAForgedAuthorization`（他人密钥签发的授权被拒）、`TestResolveRejectsATamperedAuthorization`、`TestResolveRejectsAnExpiredAuthorization`、`TestResolveRejectsUnknownAndMalformedCredentialIds`（无授权、空授权、畸形授权），以及跨真实 TLS 的 `TestServerResolveRefusalCodesOverTransport`（过期／伪造／绑定错配／已撤销／版本不可用各返回独立原因码）。"授权正确即交付"由 `TestResolveDeliversTheBoundToken` 与 `TestServerResolveDeliversThenRefusesReplay` 证明。真实进程上的拒绝矩阵见 `process_test.go` 的 `TestCredentialStoreProcessRefusesEveryInvalidAuthorization`（不可信密钥、错的运行时、已过期、根本不是授权），且其后一份有效授权仍能兑换——证明这些拒绝没有消耗任何东西 | Covered |

### 决策依据

Secret Store ADR D3、D9，不变量 2、8；Node 隔离 ADR D15。

## Token Is Encrypted With A Delivery-Only Key And Unreadable Without It

### 风险

单独导出数据库或数据目录即可解密 Token；密钥与密文存放在一起，使"加密"只是形式；或缺少 KEK 时
Store 静默以降级方式启动。

### 前置状态

Store 具备独立的持久卷布局与密钥交付介质；KEK 经部署侧交付，不出现在数据目录中。

### 触发

分别导出数据库与数据目录并尝试解密；在缺少 KEK 的情况下启动 Store；检查日志与错误输出；
在密钥轮换期间读取既有记录。

### 必须成立

- Token 以加密形式持久化；密文与 KEK 分离存放，单独导出数据库或数据目录都无法解密。
- 缺少 KEK 时**启动拒绝**，不降级、不以明文或空密钥继续运行。
- 密钥轮换有定义的行为：既有记录在轮换后仍可读，轮换过程不产生可读的中间明文残留。
- 日志与错误中不出现 Token、密文、密钥或授权载体。

### 禁止结果

以环境变量之外的方式把 KEK 写进数据目录、缺少 KEK 时以空密钥启动、轮换时把明文写入临时文件、
把密文当作可安全记录的内容。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 单独导出数据无法解密 | `internal.credentialstore` 的 `TestVolumeExportNeverYieldsToken` 把 Store 卷里每个文件的字节连同 base64（四种变体）与 hex 编码形式一起检索，断言 Token 与明文都不出现；`TestOpeningVolumeWithForeignKeyringFailsClosed` 断言拿另一份密钥环打开同一卷会失败而不是读出明文 | Covered |
| 缺少 KEK 时启动拒绝 | `TestOpenRefusesMissingKeyFileAsErrKeyMissing`（`errors.Is(err, ErrKeyMissing)` 且卷内不留任何记录文件）、`TestOpenRequiresVolumeAndKeyFile`、`TestLoadKeyringRefusesMalformedConfigurations`、`TestOpenRefusesNonPrivateVolume`；部署侧门禁见 `internal.config` 的 `TestCredentialStoreServiceRefusesAnIncompleteDeployment`（缺少 KEK 文件即拒绝启动） | Covered |
| 日志与错误无秘密 | `TestServerAuditLinesExcludeSecrets` 捕获 Store 的结构化审计日志，断言其中不含写入与轮换的 Token、密文、wrapped DEK、nonce 与 KEK 材料，同时含操作名、credential id、服务身份与结果；拒绝路径不回显材料的断言见 `TestClientRefusalsAreStableCodesWithoutStoreText` | Covered |

### 决策依据

Secret Store ADR D1、D2、D8，不变量 1、3、9。

## Cloud And Store Stay Consistent Across Create Rotate Revoke

### 风险

两个存储的写入不构成一个可解释的整体：Cloud 有元数据而 Store 无秘密（悬空引用）、Store 有秘密而
Cloud 无元数据（孤儿秘密）、撤销在 Cloud 生效而 Store 仍可解析——尤其是**Store 侧的删除失败不得使
凭据重新可用**。

### 前置状态

Cloud 的 `credential_refs` 与 Store 各自的持久化都在；写入使用幂等键；对账任务可运行。

### 触发

在元数据写入与秘密写入之间注入失败（两个方向各一次）；重复提交同一幂等键；执行轮换；
在 Store 删除失败的情况下执行撤销；随后尝试解析。

### 必须成立

- 创建路径要么两者都成功，要么对用户报失败；不产生"看似成功"的半成品。
- 幂等键使重复提交返回同一结果，不产生第二条秘密或第二行元数据。
- 撤销的**权威**固定在 Cloud 元数据上：Store 备份恢复不能复活凭据，Store 侧删除失败也不得使凭据
  重新可用。
- 悬空引用与孤儿秘密都能被对账发现，并且有明确的收敛方向。

### 禁止结果

对用户报成功而 Store 未写入、Store 删除失败时把状态改回 `available`、以 Store 的存在与否决定凭据
可用性、用重试掩盖不一致。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 双向失败都不产生半成品 | "Store 侧失败 → Cloud 不写元数据"这一向：`integration.git_credentials_test.go` 的 `TestGitCredentialServiceRefusalsBecomeReasonCodes`（五类服务失败均不留 `credential_refs` 行）与 `TestGitCredentialIngressFailsClosedWithoutACredentialService`（未配置服务时不写元数据、不记幂等记录）；"Store 删除失败 → 撤销仍生效"由 `TestGitCredentialRevokeIsAuthoritativeAndIdempotent` 覆盖。反方向（Cloud 元数据事务失败而 Store 已写入，孤儿秘密的补偿清理）没有注入测试 | Partial |
| 幂等键不产生重复秘密 | `TestGitCredentialCreateIsIdempotentOnItsKey`（重放只发生一次转交、只有一行个人凭据，同键异体返回 `idempotency_conflict`）；`internal.credentialstore` 的 `TestWriteIsIdempotentPerCredentialID`、`TestServerWriteIsIdempotentOverTransport`、`TestClientRoundTripsTheCredentialLifecycleOverMutualTLS`（重放的 Write 返回原引用而不是再封一份） | Covered |
| 撤销权威在 Cloud，Store 删除失败不复活凭据 | `TestGitCredentialRevokeIsAuthoritativeAndIdempotent`：服务不可达时撤销仍返回 `availability='invalid'`／`revoked_by_owner`、不写 `deleted_at`、凭据仍留在所有者列表中；`internal.credentialstore` 的 `TestRevokeIsTerminalTombstone` 与 `TestRevokedIsDistinguishableFromNeverExisted` 断言 Store 侧的墓碑使撤销后的重放写入被稳定拒绝 | Covered |
| 对账发现孤儿与悬空 | 尚无对账任务，也没有不一致注入测试 | Missing |

### 决策依据

Secret Store ADR D4、D6，不变量 4、5、7；归属连续性 ADR D4。

## Authorization Is Consumed Atomically At Redemption And Replay Is Refused

### 风险

一次性授权被重放：同一次授权的第二次兑换成功，或 Store 重启后已消费的授权重新可用，或在响应丢失
后旧授权被重发——这会让"一次 clone execution 一次授权"名存实亡。

### 前置状态

Cloud 已签发一次授权，状态为"未使用"；Store 具备自己的独立消费记录存储（既不在 Cloud DB，
也不在 Node DB／journal）；一次受控的兑换正在进行。

### 触发

正常兑换一次；用同一授权再兑换一次；在 Store 重启后再次尝试已消费的授权；制造响应丢失或超时后重发
旧授权；在消费记录写入与秘密交付之间崩溃或断电，并检查重启后的状态与记录内容；让消费记录无法写入或
无法解析；跨执行、跨实例、跨代次复用同一授权；检查消费记录的持久位置。

### 必须成立

- 兑换是**原子消费**：先持久消费记录，再交付秘密；不存在"已交付但未消费"的窗口。
- 已消费的授权永久不可用；重放一律拒绝，Store 重启后结论不变。
- 消费记录持久在 Store 侧自己的存储中，**不**写在 Cloud DB，也**不**写在 Node DB 或 journal。
- 响应丢失或超时后不得重发旧授权；失败一律申请新授权。
- 授权绑定 tenant／owner／project／规范化仓库／credential id／version／workspace／execution，
  任一项不匹配即拒绝。

### 禁止结果

先交付后记录、把消费记录写进 Cloud DB 或 Node journal、以"响应丢失"为由重发旧授权、允许跨执行复用、
用内存集合代替持久消费记录。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 兑换时原子消费且重放被拒绝 | `internal.credentialstore` 的 `TestResolveDeliversTheBoundToken`（交付的正是绑定的那个版本，且卷上留下结果为 `delivered` 的消费记录）、`TestResolveConsumesTheAuthorizationOnce`（同一授权第二次兑换返回 `ErrAuthorizationConsumed`）、`TestResolveConsumesAtMostOnceUnderConcurrency`（八个 goroutine 并发兑换，恰好 1 次成功、其余 7 次 `ErrAuthorizationConsumed`）、`TestResolveLeavesTheAuthorizationUnspentWhenChecksFail`（校验失败不留消费记录，且同一授权随后仍可正常兑换——顺序只能是"先记录后交付"）、跨真实 mTLS 的 `TestServerResolveDeliversThenRefusesReplay` 与 `TestServerResolveRefusalCodesOverTransport`。消费判定是消费记录文件的 `O_CREATE|O_EXCL` 创建：原子、不需要锁、不依赖进程内存。**崩溃与写入失败的语义**各有专门用例：`TestResolveTreatsAnUnparseableConsumptionRecordAsSpent`（记录存在却无法解析时读作已消费，而不是"不存在"——一个损坏的卷不能把已消费的授权变回可用）、`TestResolveRefusesWhenTheConsumptionRecordCannotBeWritten`（卷写不下消费记录时不交付任何秘密，也不返回 Token）、`TestResolveRecordsConsumedNotDeliveredWhenTheSecretCannotBeOpened`（秘密打不开时记录保持 `consumed_not_delivered`，第二次尝试仍是 `ErrAuthorizationConsumed`，即 D10 允许的唯一中间态是终态）。"交付前必须落盘"的顺序由 `syncDirectory` 在 `openToken` 之前返回固定，因此"丢失记录"的那次断电同时意味着秘密从未离开进程。**部署栈上的直接证据**：在 `ora-cluster` 真实容器上，以部署签发的 `broker-resolver` 身份向 `POST https://credential-store:8443/v1/credentials/{id}/resolve` 出示一份用部署自己的签名密钥签出的授权，第一次兑换返回 200 与 Token，同一授权立即重放返回 409 `authorization_consumed`；伪造签名的同一请求返回 403 `authorization_invalid`（而不是未配置公钥时的 `authorization_verification_unavailable`），因此该证据同时证明部署确实把 Cloud 的公钥交给了 Store、验签真实发生 | Covered |
| Store 重启后已消费授权仍被拒绝 | `TestResolveSurvivesARestart`：在同一卷、同一密钥材料上关闭并重开 `Store`，取回 `Resolver` 后用同一授权再次兑换，返回 `ErrAuthorizationConsumed`——消费记录是持久状态而不是进程内存。**跨真实进程**的同一结论由 `process_test.go` 的 `TestCredentialStoreProcessDeliversAndConsumesOneAuthorization` 给出：以 SIGTERM 停止真实 `cmd/credential-store` 进程、在同一卷上重新启动，同一授权仍返回 409 `authorization_consumed`，而随后一份新授权照常交付。**部署栈上的同一结论**：重启 `credential-store` 容器后，用同一授权标识、重新签名且有效期更长的授权重放，仍返回 409 `authorization_consumed`——消费记录跨进程重启存活，且重放拒绝不依赖载荷里的任何自述 | Covered |
| 消费记录不落在 Cloud DB 或 Node DB／journal | 落点有直接证据：`TestResolveDeliversTheBoundToken`、`TestResolveConsumesTheAuthorizationOnce` 经 `fileStore.authorizationPath` 从 `<Store 卷>/authorizations/<authorizationID>.json` 读回消费记录，记录内容只有授权标识、消费时刻、出示它的 Node 实例与运行代次和结果；跨进程验收 `TestCredentialStoreProcessDeliversAndConsumesOneAuthorization` 在真实进程写的卷上读回同一条记录，并断言其中不含 Token。Cloud DB 一侧是**结构性**排除而非测试：`internal/credentialstore` 不导入 `internal/core`、不持有任何数据库句柄，本轮也没有新增 migration，兑换路径没有可写入 Cloud 业务库的通道。Node DB／journal 一侧要等 Node 侧执行身份落地后才有可断言的介质，见[Node 凭据执行身份隔离](../../node/repository/git-credential-execution-isolation.md)。**部署栈上的直接证据**：在运行中的 `credential-store` 容器里读 `<Store 卷>/authorizations/`，记录为 `-rw------- ora ora`，内容只有 `authorizationId`／`credentialId`／`credentialVersion`／`consumedAt`／`nodeId`／`nodeInstanceId`／`nodeIncarnationId`／`runtimeGeneration`／`controlEpoch`／`expiresAt`／`result`，不含 Token；该卷只挂载给 Store 自己的容器 | Partial |
| 绑定的每一项都参与校验 | 有独立比对来源的那些项都有直接证据：`TestResolveRejectsBindingMismatches` 六项（路径上的 credential id、`nodeId`、`nodeInstanceId`、`nodeIncarnationId`、`runtimeGeneration`、`controlEpoch` 各自错配即 `ErrAuthorizationBinding`，且不留下消费记录）、`TestResolveRefusesAnIncompleteRuntimeDeclaration`（出示方什么都不声明时同样拒绝且不消耗授权，所以"自述字段"不会因为缺失而退化成通配）、`TestResolveRejectsAnUnavailableVersion`（版本对照 Store 自己的记录：从未存在的版本、保留窗口已过的版本都拒绝）、`TestResolveServesARetainedVersion`（窗口内的旧版本仍可兑换）、`TestResolveRejectsARevokedCredential`；`internal.cloneauth` 的 `TestIssueRejectsIncompleteBindings` 另外保证这 20 个字段不可能为空缺地签出去。tenant／owner／project／workspace／execution／repository 这六项被签名覆盖因此**不可篡改**，但 Store 侧没有独立来源可以比对它们（它不知道一次请求"本应"属于哪个租户），所以它们产生不了"错配即拒绝"——这是绑定语义的真实边界，不是遗漏。同理，运行时五项是出示方的自述，只能做一致性与 fencing 比对，不构成身份证明：唯一被独立核实的因子是 mTLS 的 `broker-resolver` 身份，而部署固定的三身份表让每个 Node 拿到同一张证书，因此 Store 分不出不同的 Node 实例。要让它成为密码学结论需要 per-Node 身份，而身份表目前没有这一项 | Partial |

### 决策依据

Secret Store ADR D5、D10，不变量 6、10；凭据 ADR D4、D11-B 第 4 项；Node 隔离 ADR D3。

## Audit And Backup Never Expose Secrets Or Ciphertext

### 风险

审计、备份或恢复路径成为秘密的旁路：审计记录写入 Token 或密文，备份被导出即可离线解密，
或恢复演练把旧秘密带回到可用状态。

### 前置状态

Store 具备审计记录与备份／恢复流程；备份介质与 KEK 的存放位置已定义。

### 触发

执行创建、轮换、撤销与兑换并检查审计记录与结构化日志；导出备份并在缺少 KEK 的环境尝试读取；
从备份恢复后检查凭据可用性与消费记录。

### 必须成立

- 审计与日志只记录操作与结果（谁、何时、对哪条凭据、结果如何），不含秘密、密文、密钥或授权载体。
- 备份在缺少 KEK 时不可解密；备份介质本身不携带 KEK。
- 恢复不得复活已撤销或已冻结的凭据，也不得使已消费的授权重新可用；恢复后 Cloud 元数据仍是撤销与
  冻结的权威。

### 禁止结果

审计记录写入 `secret_ref` 之外的秘密材料、把 KEK 与备份放在同一介质、以恢复为手段绕过撤销、
把消费记录排除在备份之外导致恢复后授权可重放。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 审计与日志不含秘密、密文与授权载体 | `TestServerAuditLinesExcludeSecrets`（同上一用例）：捕获的审计行不含 Token、密文、wrapped DEK、nonce 与 KEK 材料，且成功后与拒绝后都记录操作、credential id、身份与结果。兑换路径另由 `TestServerResolveKeepsSecretsOutOfTheAuditLog` 覆盖：一次成功的 `resolve` 之后，审计行含 `"operation":"resolve"` 与 `"result":"succeeded"`，同时**既不含授权载体也不含 Token**——两者都是 bearer 秘密。真实进程一侧由 `TestCredentialStoreProcessDeliversAndConsumesOneAuthorization` 覆盖：它读的是 Service 落盘的审计文件（JSON 编码器，部署实际会检视的那一份），断言其中含成功的 `resolve` 与重放拒绝记录，且不含 Token 与任何授权载体；stdout 的 console 编码器同样被断言不含秘密。尚无审计落库／导出面的证据 | Partial |
| 备份不可离线解密 | 需要备份导出与解密尝试的测试。`TestVolumeExportNeverYieldsToken` 只覆盖"卷本身"这一半；"卷 + 包装元数据、不含 KEK"的备份约定还没有演练证据 | Missing |
| 恢复不复活凭据、不复活消费 | 需要恢复演练加解析／重放尝试的测试 | Missing |

### 决策依据

Secret Store ADR D7、D8，不变量 9；归属连续性 ADR D4；`cloud/AGENTS.md` 的日志脱敏规则。
