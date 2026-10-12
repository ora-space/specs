# 仓库凭据的能力词汇表：只承认 `clone`、空集合继续被拒、迁移逐条授权

依据[仓库凭据能力词汇表](../../../decisions/cloud/identity-access/20261012-repository-credential-capability-vocabulary.md)
（`approved`，2026-10-12 批准：A–D 按推荐批准，E 有条件批准）。本文件登记**能力词汇表本身**的承诺：
词汇表只有 `clone` 且不隐含读或推、能力是引用（而不是请求方）的属性、允许不了 clone 的引用在准入处
被拒且不降级、以及既有 `'[]'` 引用凭什么、按什么规则才能获得一次显式授权。

凭据的元数据权威、`secret_ref` 的非 bearer 性质与逐次授权绑定登记在
[凭据元数据](repository-credential-metadata.md)；秘密的承载、兑换与原子消费登记在
[凭据 Secret Store](repository-credential-secret-store.md)；执行侧的 `clone` 语义与身份隔离登记在
[Node 凭据执行身份隔离](../../node/repository/git-credential-execution-isolation.md)。本文件**不重复**
登记这些义务，只登记词汇表一侧的判断依据。

**当前实现状态（2026-10-12 批准后）**：`clone` 的词汇表、规范化与三处强制（准入、签发、执行）都已
实现，证据在 `internal/cloneauth`（`capability.go` 的 `Capabilities`／`CanonicalCapabilities`）、
`internal/core/repository_credentials.go`、`internal/cloneexec/executor.go` 与各自的测试。仍然
`Missing` 的是**迁移一侧**：`ConfigureCredential` 的 `INSERT` 还不含 `capabilities` 列（列默认 `'[]'`
生效），因此"部署托管创建路径显式声明 `clone`"与"既有 `'[]'` 引用逐条核验后迁移"都还没有实现，
按 ADR 落地顺序第 3 步与解禁那次变更同批落地。批准本身**不解除任何 fail-closed**：带凭据的 clone 在
每个执行路径上继续被拒绝。

## The Capability Vocabulary Admits Only Clone And Never Implies Read Or Push

### 风险

为了让某个验收行"看起来满足"，或因为一个常量看起来少了一项，词汇表被顺手加宽：`read`、`push`、
`fetch`、`pull` 之类没有授权语义、没有记账规则、也没有强制执行点的成员进入签发路径。此后一次读取
授权可能被解释成无限次读取，一次写入授权可能被解释成"Agent 也能推"。

### 前置状态

`credential_refs.capabilities` 是 `jsonb`，列默认 `'[]'`；签发与执行两侧都通过
`cloneauth.CanonicalCapabilities` 归一化并比对。

### 触发

给引用写入未定义成员、重复成员、乱序集合、空集合或大小写不同的拼写；把 `read`／`push` 当作"显然
需要"的能力加入词汇表；让某条路径只比较集合的成员而不比较完整集合。

### 必须成立

- 词汇表恰为 `["clone"]`；`read`、`write`、`push`、`fetch`、`pull`、`admin` 与空字符串都不是已知能力。
- `clone` 是 D1 的六条语义（一次性、只读、单仓库、有期限、身份绑定、归属绑定），它不蕴含 fetch、
  pull、push，也不蕴含"可以读"这一无限次解释。
- 未知、重复、空集合与不按规范顺序的集合一律被拒绝，且在**签发**阶段就被拒绝，任何这样的集合都不能
  进入已签名的授权。
- 加宽词汇表是一次决策而不是一次编辑；公开契约的枚举随之一同变更。

### 禁止结果

在词汇表里新增第二个成员而不重新论证其授权语义；让 `KnownCapability` 接受未定义值；让某个未定义的
能力在签名载荷里出现；把"读取能力"当作 `clone` 的同义词。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 词汇表恰为 `["clone"]`，未定义成员不被接受 | `cloneauth.TestCapabilityVocabularyIsClosedToWhatTheDecisionFixes`：断言词汇表长度为 1 且为 `clone`，并逐个断言 `read`／`write`／`push`／`fetch`／`pull`／`admin`／空串不被 `KnownCapability` 接受。测试注释写明"加宽这份清单是一次决策而不是一次编辑" | Covered |
| 只有规范集合可通过，其余一律拒绝 | `cloneauth.TestCanonicalCapabilitiesRefusesEverySetItCannotHaveIssued` 的八组输入：`["clone"]` 通过；nil、空集合、越界成员、`["clone","fetch"]`、重复、大小写不同、空字符串成员全部返回 `ErrIncomplete` | Covered |
| `clone` 的六条语义作为此后成员的基准 | 语义本身是文字化承诺，其可测部分分别登记在[凭据元数据](repository-credential-metadata.md)（归属与版本绑定）与[凭据 Secret Store](repository-credential-secret-store.md)（一次性消费与重放拒绝），本文件不重复登记同一义务 | Partial |
| 加宽词汇表必须同时变更公开契约枚举 | 契约侧证据登记在[凭据元数据](repository-credential-metadata.md)的公开 API 义务；本文不重复 | Partial |

### 决策依据

能力词汇表 ADR 的 A、B 两项（批准记录），D1 与 D2；不变量 1、2、6。

## A Reference That Permits No Clone Is Refused Before The Executor Is Asked

### 风险

引用允许不了 clone 时，某条路径把它当作"没写限制"或降级成本次允许：执行器已经收到请求、已经花掉
预算甚至已经开始下载，拒绝才发生；或拒绝发生在执行之后，留给运营一个"看起来成功的派发"。

### 前置状态

Project 通过 `projects.credential_ref_id` 绑定一个引用；引用所在行给出能力集合；部署可能根本没有
可用的执行器。

### 触发

把引用的能力集合写成列默认值 `'[]'`、未定义成员、`["push"]`、大小写不同、重复成员或非字符串成员后
派发一次 clone；在一个没有执行器的部署上对带凭据的 Project 派发 clone。

### 必须成立

- 引用不允许所需能力时以 `repository_credential_capability_unsupported` 明确拒绝，**执行器一次都没有
  被调用**。
- 部署自己的拒绝（`repository_credential_executor_unavailable`）优先于引用侧的能力拒绝，并保持它
  原有的码与结算，不因能力条款改写已发布的行为。
- 该拒绝与其它确定性拒绝一样把 Operation 结算为终态。
- 能力集合随时间被改写（轮换、冻结、迁移）不改变已受理执行的身份与版本绑定。

### 禁止结果

在能力判定之前触发执行器；把空集合读成"不受限制"；以"部署还没装执行器"为由跳过能力判定；为让派发
成功而放宽集合判定。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 引用允许不了 clone 时拒绝，且执行器未被调用 | `core.TestCredentialedCloneAdmissionRequiresTheReferencesCloneCapability` 的八组输入：`["clone"]` 被准入且执行器被请求一次；列默认 `[]`、未定义成员、`push`、大小写不同、重复、非字符串成员、空串成员全部以 `repository_credential_capability_unsupported` 拒绝，并断言 `cloner.calls` 为空 | Covered |
| 部署自己的拒绝优先于能力条款 | `core.TestADeploymentsOwnRefusalOutranksTheReferencesCapability`：两个拒绝同时适用时返回的是部署侧的码；测试注释写明改这一顺序会把引用说成障碍、并改掉已发布的拒绝码 | Covered |
| 能力拒绝把 Operation 结算为终态 | `core.TestAnUnsupportedCapabilitySettlesTheOperationToo` | Covered |
| 拒绝不改写持久归属与引用、不留下 clone execution 行 | 同一批拒绝的持久面证据登记在[凭据元数据](repository-credential-metadata.md)的"引用未声明所需能力时在准入处明确拒绝"一行，本文件不重复 | Covered |

### 决策依据

D3、D5；不变量 3、4、6；[把带凭据的拒绝结算为终态](../../../decisions/cloud/operation/20261012-credentialed-clone-refusal-is-terminal.md)。

## The Required Capability Travels In The Signed Binding And Is Compared Again At The Executor

### 风险

能力只在准入处比对一次，之后的签发、传输或执行任一环节都能被替换：签名载荷里没有能力，执行器就
只能相信调用方；或执行器信任调用方自述的能力，于是"请求方自述不增加能力"这一条失守。

### 前置状态

准入判定通过后，所需能力随 `cloneauth.Binding` 被签名，随 `cloneexec.Job` 到达执行器；执行器持有
自己的配置与物化根。

### 触发

签发一个能力集合为空、含未定义成员或重复成员的绑定；篡改绑定里的能力字段；向执行器提交一个能力
集合为 nil、含 `push` 或重复的 `Job`。

### 必须成立

- 签发侧只接受规范集合：空、未定义、重复、越界的能力集合都无法获得签名。
- 执行计划携带准入判定所要求的能力（今天恒为 `["clone"]`），而不是由执行器或 Node 自行推断。
- 执行器在执行任何 Git 命令、花掉任何预算之前再比对一次，并拒绝非法集合。
- 执行器不因调用方自述、环境变量或配置而获得调用方未声明的能力。

### 禁止结果

把能力从签名范围内拿掉，让执行器相信未签名的自述；只在执行器一侧比对；把 `push` 之类未定义能力
透传给 Git。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 签发侧拒绝不可签发的能力集合 | `cloneauth.TestIssueRejectsIncompleteBindings`：Binding 的能力为 nil、空集合、`fetch`、重复、越界成员时 `Issue` 返回 `ErrIncomplete` | Covered |
| 执行计划携带准入所要求的能力 | `core.TestCredentialedCloneAdmissionRequiresTheReferencesCloneCapability`：断言被交给执行器的 `plan.Capabilities` 恰为 `["clone"]` | Covered |
| 执行器在花掉任何东西之前再比对一次 | `cloneexec.TestExecuteRefusesBeforeSpendingAnything`：`Job.Capabilities` 为 nil、`["push"]`、`["clone","clone"]` 时以 `ReasonCodeInvalidJob` 拒绝 | Covered |
| 三处强制都发生在真实进程与真实投递边界上 | 执行器的端到端证据登记在[Node 凭据执行身份隔离](../../node/repository/git-credential-execution-isolation.md)；本文件只登记比对规则本身 | Partial |

### 决策依据

D3；不变量 3、4。

## Migrating A Stored Empty Capability Set Is Per-Row And Verified, Never Blanket

### 风险

为了"让部署托管引用可用"，用一条语句把历史引用批量升级：`UPDATE credential_refs SET
capabilities='["clone"]'`。这会把一次归属猜测变成一次授权，且授权范围没有任何人逐条看过——在没有
授权依据的引用上，它等于凭空造出一份读取私有仓库的许可。

### 前置状态

`ConfigureCredential` 的 `INSERT` 不含 `capabilities` 列，因此它写入的引用落到列默认值 `'[]'`；
此类引用今天仍可绑定到 Project（D7 的兼容性），但在任何路径上都拿不到能力。

### 触发

准备一次解禁变更；需要让既有部署托管引用在解禁后可用；有人提出"这些都是部署托管的，一起放开"。

### 必须成立

- 部署托管创建路径**显式、可追踪地**声明 `clone`：值是字面量，不是从参数、环境或归属推导出来的；
  同一次创建在既有记录里可审计。
- 既有 `'[]'` 引用**逐条**核验四项——来源、Scope、所有权、原有授权依据——并留下具名清单（引用 id
  列表）；四项都能落成事实的引用才被迁移。
- 不存在无条件或按谓词批量的授权：`attribution='unknown'`、`capabilities='[]'`、租户、时间窗都不是
  授权判据；每条授权都能回溯到清单里具名的一行。
- `attribution='unknown'` 只能作为操作者核验时的线索，不得单独充当"原有授权依据"。

### 禁止结果

一条不带具名 `WHERE` 的 `UPDATE`；在 migration 文件里自动回填；改列默认值使既有行"自动可用"；
把"看起来是部署托管的"当作授权依据；把迁移与授权声明塞进一次 schema 变更而不留清单。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 部署托管创建路径显式声明 `clone` | 尚不存在：`core.Store.ConfigureCredential` 的 `INSERT` 今天不含 `capabilities` 列（列默认 `'[]'` 生效），因此没有可引用的测试。落地在解禁那次变更（ADR 落地顺序第 3 步） | Missing |
| 既有 `'[]'` 引用逐条核验四项后迁移，并留具名清单 | 尚不存在迁移，也没有断言清单内容的测试；本条是部署流程侧的义务，落地时须有可审计的迁移记录 | Missing |
| 无依据的引用不被迁移，且不得出现批量授权语句 | 今天不存在任何写入 `capabilities` 的迁移语句，因此没有测试可引；判据由迁移评审把守 | Missing |
| 列默认值不变、空集合不被读成"不受限制" | `core.TestCredentialedCloneAdmissionRequiresTheReferencesCloneCapability` 的"the column's own default"一例直接断言默认值下的引用不被准入；默认值本身由迁移保持 | Covered |

### 决策依据

D5 第 1–3 条；不变量 6、10、12；[仓库凭据归属连续性](../../../decisions/cloud/identity-access/20261010-repository-credential-ownership-continuity.md) D2（禁止推断归属）。

## A Reference Without Proven Authorization Stays Frozen And Never Falls Back

### 风险

核验不通过的引用被"临时放开"或被绕开：有人给它一个默认可用的解释，有人借用另一个引用的清单，有人
让它退回匿名 clone。任一做法都会把一次无依据的读取变成系统承诺。

### 前置状态

引用仍以 `'[]'` 存在（或已被部署显式删除／解绑）；准入判定在凭据路径上无条件拒绝。

### 触发

对一条四项核验没有全部通过的引用请求 clone；对该引用所在 Project 派发执行；有人主张"它就这一次，
先用匿名路径顶上"。

### 必须成立

- 冻结是合法结论：引用继续以 `'[]'` 被读作"不允许任何操作"，继续在准入处被
  `repository_credential_capability_unsupported` 拒绝（部署级拒绝先于它时按部署级码拒绝）。
- 显式拒绝或删除／解绑也是合法结论；两者都比一次无依据的授权安全。
- 不存在任何兜底：不放开空集合、不给"默认可用"的解释、不借用其他引用的清单或授权、不用匿名 clone
  代替该引用自己的授权。
- 冻结期间引用仍可被绑定、被列出（D7 的兼容性），但可绑定不等于有可用能力。

### 禁止结果

为空集合开一条兼容豁口；以匿名路径"顶上"一次带凭据的 clone；在拒绝之后仍产生 `clone_executions`
行或执行器调用；把冻结的引用从产品列表里悄悄改写以规避拒绝。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 允许不了 clone 的引用继续被拒绝 | `core.TestCredentialedCloneAdmissionRequiresTheReferencesCloneCapability` 的空集合两例；`core.TestADeploymentsOwnRefusalOutranksTheReferencesCapability`（部署托管引用 + 无执行器） | Covered |
| 不退回匿名 clone，不留下执行记录 | `integration.repository_credentials_test.go::TestCredentialedCloneRefusesMissingExecutorAndAnonymousFallback`；`integration.git_credential_binding_test.go::TestCredentialedCloneStaysRefusedOnEveryPath`（断言拒绝码、拒绝发生在注册 clone execution 之前且无 `clone_executions` 行） | Covered |
| 部署托管引用保持可绑定，但绑定不带来能力 | `integration.git_credential_binding_test.go::TestProjectBindingAcceptsOnlyACredentialThatCouldClone`：单独断言 `ConfigureCredential` 写出的引用仍可绑定（D7 兼容），同时被拒绝的绑定不留下 Project 行 | Covered |
| 四项核验的处置记录：不通过的引用被明确标记为冻结或删除 | 尚无迁移与处置记录，落地在解禁那次变更 | Missing |

### 决策依据

D5 第 4 条；不变量 6、9、11；[把带凭据的拒绝结算为终态](../../../decisions/cloud/operation/20261012-credentialed-clone-refusal-is-terminal.md)。
