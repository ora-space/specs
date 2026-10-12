# 推送执行权限

依据[推送执行权限](../../../decisions/node/repository/20261012-push-execution-authority.md)（`approved`，2026-10-12；
A–D、F 批准，E 不批准）。本文件登记写方向的长期承诺：推送只能由部署固定的执行器按 Cloud 的单次授权执行、
写入范围逐次限定且默认拒绝、秘密只经既有兑换路径到场并在执行结束时销毁、Agent 既请求不到也拿不到秘密，
以及"词汇表未扩展前推送不可被请求"。

**当前实现状态**：**推送路径不存在，这是三处判定的合力而不是巧合**——词汇表只有 `clone`、准入只承认
`clone`、执行器只构造读取命令。本 ADR 的获批**不改变任何运行时行为**：它只是写方向的权限模型，
`push` 仍不在词汇表中，`CanonicalCapabilities` 继续拒绝它。因此除"不可被请求"这一条外，本文件其余
用例的义务在实现与启用之前**全部为 `Missing`**；`not-run` 不得写成 passed，也不得用 clone 读向的证据
或
[推送交付 ref](../../../decisions/node/revision/20261012-push-delivery-ref-and-fetch-prior-revision.md)
通道的证据互相充当。

## Push Is Executed Only By The Deployment Fixed Executor And Never Requested By An Agent

### 风险

把写权限落到会话身份或 Agent 上，"能不能推"就取决于提示词与工具白名单，撤销与审计都没有稳定对象。

### 前置状态

存在部署固定的凭据执行器（身份 C）与固定的 helper；存在人（项目所有者或管理员）通过受控入口发起的
推送请求；Agent 会话可以发出 Thread 消息与工具调用。

### 触发

提交一次推送请求；另由一个 Agent 会话尝试以会话事件、Thread 消息或工具调用触发同一次推送。

### 必须成立

- 推送作业只由部署固定的执行器执行；业务请求不能指定执行者，也不能指定 helper。
- 请求方必须是已认证、已授权的人或产品自身受控的交付流程；Agent 的会话事件、消息与工具调用**不构成**
  推送请求，也不得被解释为对某次推送的批准。
- 远端具备写权限、网络可达、凭据可写都不构成授权。

### 禁止结果

由 Node 会话身份或 Agent 执行推送、把 Agent 的调用读成批准、把"远端允许"当成授权。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 只有部署固定的执行器执行推送，会话与 Agent 既不执行也不请求 | 尚无：执行器今天只构造读取命令，写方向不存在 | Missing |
| 远端许可不构成授权 | 尚无：需要一条"凭据可写但无 Cloud 授权"的真实拒绝证据 | Missing |

### 决策依据

D1、D2、D5 与不变量 1、2、9；继承[隔离决策](../../../decisions/node/repository/20261010-controlled-node-git-authentication-isolation.md)
D1、D2、D7、D9。

## A Push Authorization Binds Tenant Project Workspace Execution Generation Repository And Ref Scope

### 风险

授权不含写范围的精确绑定，一次授权就能改任意 ref；或代次被替换后旧执行仍在推。

### 前置状态

Cloud 具备签发能力（`internal/cloneauth` 的 `Issue`／`Verify`）；推送授权与 clone 授权同构并增加 ref 范围。

### 触发

签发一次推送授权，然后尝试用它推另一个仓库、另一个租户的执行、已被替换的运行代次，或超出范围的 ref。

### 必须成立

- 授权绑定租户、Project、Workspace、执行、运行代次与 Node 实例、规范化的仓库身份、凭据 id 与版本、
  ref 范围、一次性与有效期；**缺一项即拒绝**。
- 运行代次是 Node 的运行代次，不是 Controller 租约的 `epoch`。
- 仓库身份逐字节比对，不做前缀或模式匹配。

### 禁止结果

缺少任一绑定项仍被接受、用 Controller 租约 epoch 代替运行代次、按前缀匹配仓库。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 授权绑定九项且缺一即拒 | 相邻但**不可替代**：`cloneauth.TestIssueRejectsIncompleteBindings` 覆盖读向绑定，写向的 ref 范围与写能力成员尚未存在 | Missing |
| 被替换的代次不得继续推 | 尚无：写方向不存在 | Missing |

### 决策依据

D2、D3 与不变量 4；继承[隔离决策](../../../decisions/node/repository/20261010-controlled-node-git-authentication-isolation.md)
D3、D4。

## The Ref Scope Is Denied By Default And Never Allows Force Or Deletion

### 风险

写权限不带范围等于对仓库的任意写；force 与删除是最难恢复的两类写操作。

### 前置状态

授权写明允许更新的 ref 集合；存在受保护分支、tag 与远端权限规则。

### 触发

提交未在授权中写明的 ref、带 `+` 的 force 更新、删除 ref、删除 tag，以及一个只有前缀相同的 ref 名。

### 必须成立

- 未写明的 ref 一律不可更新；ref 名按完整名字比对，不承认 `refs/*`、`*` 之类的模式。
- force、删除 ref、删除 tag、改默认分支指向都在**准入处**拒绝，而不是依赖远端拒绝。
- Git host 自己的保护规则与审计仍是最终判据；被远端拒绝如实记为失败，不由 Cloud"修正"。

### 禁止结果

把范围交给远端判、按前缀通配放行、把远端拒绝改写成成功。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 未写明的 ref 在准入处被拒 | 尚无：写方向不存在 | Missing |
| force 与删除不在允许范围内 | 尚无 | Missing |

### 决策依据

D3 与不变量 5。

## The Secret Reaches Only The Executor And Is Destroyed When The Execution Ends

### 风险

凭据落进 URL、argv、环境、日志、业务库、执行器台账或 Workspace 卷；或执行结束后物化残留可被再次兑换。

### 前置状态

Store 兑换路径（mTLS 服务身份、Cloud 公钥验签、原子单次消费、重放拒绝）与部署固定的 helper 均在位；
物化根对 W／C 之外不可遍历。

### 触发

执行一次推送；在一次推送超时与中断处终止；在物化仍然存在的时刻由 Agent 会话尝试读取。

### 必须成立

- 秘密只存在于执行器进程内存与 helper 之间的一次管道；不出现在 URL、argv、环境变量、日志、错误串、
  业务库、业务事件、执行器台账与 Node 持久记录，也不写进 Workspace 卷。
- 兑换顺序固定为「校验 → 落消费记录 → 才交付」；已消费的授权即使未被交付也不得重放。
- 成功、失败、超时、中断四种结局都销毁物化。
- **helper 的作用域必须精确到 host＋path**：今天只比对 `host`，同一 host 下的另一个仓库借得到这份
  scope（D4 记录的现状缺口）。未补齐前不得启用推送。

### 禁止结果

helper 只按 host 放行、物化残留、执行结束后授权仍可兑换。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 秘密不进任何持久媒介与 Workspace 卷 | 相邻但**不可替代**：`cloneexec.TestExecuteClonesAndThenProvesTheCredentialGone` 等覆盖读向的销毁与不可达，写向不存在 | Missing |
| 四种结局都销毁物化 | 尚无：写方向不存在 | Missing |
| helper 精确到 host＋path 的作用域校验 | 尚无：现状是只比对 `host`，且 `internal/cloneexec/git.go` 的注释曾把它描述成 path 绑定。该收紧可在不启用推送的前提下先落地 | Missing |

### 决策依据

D4 与不变量 6–8；继承[隔离决策](../../../decisions/node/repository/20261010-controlled-node-git-authentication-isolation.md)
D3、D4、D6 与[凭据执行器归属](../../../decisions/node/repository/20261012-credential-executor-belongs-to-the-current-product.md)D1。

## A Spent Authorization Is Never A Retry Path And Partial Success Is Recorded As Partial

### 风险

用重放旧授权当作重试；把多 ref 的部分成功汇总成"成功"；失败分类泄露秘密或远端细节。

### 前置状态

一次推送授权已消费；远端只提供逐 ref 结果，不提供跨 ref 原子性。

### 触发

同一授权重放；一次多 ref 推送中部分 ref 被远端拒绝；一次网络超时。

### 必须成立

- 一次授权只允许一次执行尝试；重试必须重新签发、重新消费。
- 部分成功如实记录哪些 ref 更新了、哪些没有、错误码是什么。
- 失败分类能区分未授权／过期或已消费／ref 不在范围／远端拒绝／网络或超时／执行器不可用，且不泄露秘密。
- 审计记录含授权标识、请求方、执行 id、租户、Project、仓库、允许与被实际更新的 ref、结果码、时间；
  **不含秘密、不含 bearer 授权本身**。

### 禁止结果

重放旧授权当重试、把部分成功记成成功、审计里出现授权或秘密。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 授权不可重放、重试必须重新签发 | 相邻但**不可替代**：`cloneexec.TestTheAuthorizationIsConsumedOnceAndReplayIsRefused` 覆盖读向兑换 | Missing |
| 部分成功如实记录 | 尚无：写方向不存在 | Missing |

### 决策依据

D7 与不变量 3、11。

## Push Stays Unrequestable Until The Vocabulary Is Extended By Its Own Decision

### 风险

把本 ADR 的获批读成"推送已可用"，从而在词汇表之外偷偷放开一个写能力。

### 前置状态

`capabilities` 只有 `clone`；`push` 是词汇表外的成员。

### 触发

以任何路径请求一次推送；或以 `push` 作为绑定/执行计划里的能力值。

### 必须成立

- `push` 加入枚举需要另一篇 `approved` 决策，并同时变更公开契约枚举与准入、签发、执行三处强制点。
- 一次推送需要写能力成员，不得把 `clone` 读成写许可。
- 本 ADR 的获批不解除任何 fail-closed：带凭据的 clone 继续以 `repository_credential_executor_unavailable`
  逐位拒绝。

### 禁止结果

以"权限模型已批准"为由启用推送、把只读授权复用为写许可、在词汇表之外放行任何写路径。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 词汇表继续拒绝 `push`，且只有独立决策能扩展它 | `cloneauth.TestCapabilityVocabularyIsClosedToWhatTheDecisionFixes`（断言 `read`／`write`／`push`／`fetch`／`pull`／`admin` 都在词汇表之外）、`cloneauth.TestCanonicalCapabilitiesRefusesEverySetItCannotHaveIssued`（`["push"]` 被拒）、`cloneauth.TestHasCapabilityComparesTheCanonicalSpelling`（要求 `push` 时判 false）、`cloneauth.TestIssueRejectsIncompleteBindings`（`operation=push` 被拒）——四条共同固定"推送今天不可被请求" | Covered |
| 一次 clone 授权不得被当成写许可 | `cloneauth` 的验签只接受恰好 `["clone"]`；能力词汇表不变量 5 登记在[凭据能力词汇表](../../cloud/identity-access/repository-credential-capability-vocabulary.md) | Covered |

### 决策依据

D6 与不变量 10；[凭据能力词汇表](../../../decisions/cloud/identity-access/20261012-repository-credential-capability-vocabulary.md)
的不变量 5、7。

## The Push Acceptance Requires Real Remote Evidence And Never Borrows Another Channel

### 风险

用定向单测、mock 或内部交付 ref 通道的结果顶替真实私仓推送验收。

### 前置状态

存在真实私有仓库、真实远端与受控凭据；内部 Git Server 的交付 ref 通道是**另一条**通道。

### 触发

Issue #16 验收第 2 项（推送一侧）的取证。

### 必须成立

- 证据要求：真实私仓＋真实远端＋真实凭据在受控通道内，且"授权单次消费、ref 范围生效、Agent 拿不到秘密、
  执行后销毁"逐条成立；**缺一即 `Missing`**。
- 交付 ref 通道推的是内部 Git Server 的交付 ref，不是用户仓库；两侧证据不得互证，也不得用一侧的落地
  掩盖另一侧的缺口。

### 禁止结果

`not-run` 写成 passed、用 mock 顶替真实远端、用交付 ref 通道的结果充当用户仓库推送验收。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 真实私仓推送在受控通道内逐条成立 | 尚无：推送路径不存在，词汇表未扩展 | Missing |

### 决策依据

D8 与不变量 12；[推送交付 ref 与取回前序版本](../../../decisions/node/revision/20261012-push-delivery-ref-and-fetch-prior-revision.md)。
