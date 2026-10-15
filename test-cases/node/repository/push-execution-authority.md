# 推送执行权限

依据[推送执行权限](../../../decisions/node/repository/20261012-push-execution-authority.md)（`proposed`，2026-10-12；A–D、F 同意，E 不同意，那次裁决未走 specs 的 PR 评审）。本文件登记写方向的长期承诺：推送只能由部署固定的执行器按 Cloud 的单次授权执行、
写入范围逐次限定且默认拒绝、秘密只经既有兑换路径到场并在执行结束时销毁、Agent 既请求不到也拿不到秘密，
以及"词汇表只能由它自己那篇独立决策扩展"（末一条的"未扩展前"那一半已于 2026-10-13 按其自身条款办完，
见下节）。

**当前实现状态（2026-10-13 重写）**：本文件原先那句「**推送路径不存在，这是三处判定的合力而不是巧合**」
——词汇表只有 `clone`、准入只承认 `clone`、执行器只构造读取命令——**自 2026-10-13 起不再成立**。当天
按本 ADR 落地顺序的第 3、4 步依次发生三件事：

| 日期 | 发生了什么 | 落点 |
|---|---|---|
| 2026-10-13 | 词汇表由**它自己那篇**独立决策扩展为 `["clone","push"]`，同一次裁决还修订了本 ADR 的两处绑定项与结果语义 | [`20261013-push-capability-vocabulary-extension.md`](../../../decisions/cloud/identity-access/20261013-push-capability-vocabulary-extension.md)（`proposed`，A–G 全部按建议同意；`specs` `aebe21d` 新建、`682cd78` 补正；那次裁决**未走 specs 的 PR 评审**，因此正文里的裁决结论待评审、`approved` 落款未写） |
| 2026-10-13 | 公开契约枚举、准入、签发、执行四处强制点在同一次变更里就位 | cloud `16e0c40` |
| 2026-10-13 | 写向实现：单次授权的签发、受控执行与三终态结算、对账 | cloud `2bd610d` |
| 2026-10-14 | 容器级验收实测到的一处缺陷被修复：执行器失联时的派发错误不再被映射成带原因码的 `Fault`，结算落到 `uncertain`／`push_result_unknown` | cloud `d49edea` |

因此本文件与写方向有关的义务状态**按实测重写**，不再沿用「实现前一律 `Missing`」这一整块结论。可引用的
实测是**容器级 D9 验收**：`cluster/scripts/push-acceptance.py` 在专用隔离栈 `ora-private-mirror` 上起真实
Cloud、真实 Credential Store、真实执行器 C，以及一个由验收自己启动的**真实 HTTPS Git 远端**（凭据由该远端
逐请求裁决，非 mock、非内部交付 ref 通道）。它有**三次**运行，三次都留在各自的证据里：

- **修复前（cloud `2bd610d`，2026-10-13）**：**完整的 58 项**断言、**57 项成立、退出码 1**，证据
  `cluster/.acceptance/push-acceptance/push-acceptance.json`。唯一失败项
  `interruption/an-interrupted-write-is-not-a-concluded-failure` 指向一处**真实产品缺陷**，不是脚本问题。
- **修复后·单场景复测（cloud `d49edea`，2026-10-14）**：重建两个镜像后只复测中断场景及其依赖
  （`--only interruption`，`environment` 与 `baseline` 两组照常运行），**20/20 项成立、退出码 0**，证据
  `cluster/.acceptance/push-acceptance-fixed/push-acceptance.json`。
- **修复后·完整矩阵（cloud `d49edea`，2026-10-14）**：在**已经验证过的那两个镜像上**（不再重建）重跑
  **完整的 58 项**，八个阶段全跑、无 `--only`，**58/58 项成立、退出码 0**，证据
  `cluster/.acceptance/push-acceptance-full-fixed/push-acceptance.json`（镜像标签与摘要见同目录
  `image-digests.txt` 与 `provenance.json`）。这一轮把"修复后的代码在完整 D9 矩阵上全绿"从推断变成了实测。

三次运行各自独立、互不顶替：修复前那一次是**历史结论，未被覆盖**；单场景那一次说明缺陷确实被修掉了；
完整矩阵那一次说明**修复没有把别的东西弄坏**。下文凡引用实测处都写明引自哪一次。

**三个口径不许混用**：`Implemented`（写向代码已存在，cloud `16e0c40`＋`2bd610d`＋`d49edea`）、
`Integrated`／`End-to-End Passed`（**仅限隔离栈** `ora-private-mirror`，即上面这三次运行）、
`Production Enabled`（**否**——见文末红线①）。"隔离环境验收通过"不等于"生产已启用"。

**两条仍然成立的红线**：①**生产继续 fail-closed**——`CLOUD_CLONE_DISPATCH_PUSH_ENABLED` 与
`CLONE_EXECUTOR_PUSH_ROOT` 只出现在验收专用的 `cluster/compose.push-acceptance.yaml` 里，`compose.yaml`
中没有它们，部署上推送依旧不可用，四处强制点在生产配置下逐位不变；②**`not-run` 不得写成 passed**，
clone 读向的证据与[推送交付 ref](../../../decisions/node/revision/20261012-push-delivery-ref-and-fetch-prior-revision.md)
通道的证据仍不得互相充当。D9 的九条中仍有未取得直接证据的部分，逐条列在文末。

**2026-10-13 更正一处**：本文件原先写「除"不可被请求"这一条外，其余义务全部为 `Missing`」。这对
**helper 的 host＋path 作用域校验**不成立——该 ADR 的落地顺序第 2 步（「仅限读向收紧；不依赖前方
任何决策」）已经落地，该义务现在有直接且可证伪的证据，已改为 `Covered`（见下表与下节）。本条更正**不改变**
任何写方向的结论：推送仍不可被请求，其余义务仍为 `Missing`。同时更正：本文件与 D4 都把该缺口描述成
「helper 今天只比对 `host`」，这在当前代码上已不成立；D4 那段现状描述属于**决策文本的正文**，
不在此处单方面改写，见文末[决策文本里一段过期现状描述的更正](#决策文本里一段过期现状描述的更正2026-10-13-已裁决并落地)
（该处已按"只追加前向注记"更正，原文逐字保留）。

**2026-10-13 再更正一处（本轮）**：上一段末尾那句「推送仍不可被请求，其余义务仍为 `Missing`」写在
词汇表扩展决策走完评审之前，**当天即被那篇决策自己的裁决所取代**（那篇同样保持 `proposed`）——`push` 已成为词汇表的第二个成员，写向
实现也已落地。本文件据此逐行重估证据状态：**新取得直接证据的行按实测改写，没有证据的行继续如实保留
`Missing`／`Partial`，容器级未通过的项一条都没有写成 `Covered`**。上一段的更正理由（helper 的
host＋path 收紧与写向启用互相独立）不受影响，仍然成立。**2026-10-14 补一句**：当时唯一未通过的那一项
（D9 第 7 条的中断分支）已不再是"未通过"——它指向的产品缺陷被修掉（cloud `d49edea`），并在重建镜像上
重测成立，因此该行按**新的实测**改为 `Covered`（见下文那一节与 D9 表第 7 条）。这不是把未通过的项写成
通过，而是那一次不通过被当成缺陷修完后又测了一次。

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
| 只有部署固定的执行器执行推送，会话与 Agent 既不执行也不请求 | **已取得（2026-10-13 由 `Missing` 转为 `Covered`）**。执行侧：容器级验收的 `write/the-write-was-one-receive-pack`（一次批准恰好一次 receive-pack）与 `write/the-store-delivered-to-the-executor-identity`（Store 把秘密交付给 `spiffe://ora.local/credential-store/broker-resolver`，即 C 的兑换身份）。请求侧（负向控制）：`identity/a-peer-that-is-not-the-dispatcher-cannot-ask-for-a-write`——一个出示**另一个身份**证书的对端连 TLS 握手都过不去（状态 0，`remote error: tls: certificate required`），根本没有到达作业解析；`identity/a-forged-authorization-cannot-cause-a-write`（伪造作业被 403 拒绝、没花费也没写）；`identity/the-ingress-identity-cannot-redeem`（写向 ingress 身份被 403 拒绝）。业务侧：批准只能由工作区的已授权成员经认证会话发出，`core.TestAPushApprovalRequiresTheProjectRuleAndNotOnlyTheWorkspaceOne` 与 `core.TestAPushApprovalIsRefusedForACallerOutsideTheTenant` 分别钉住"只认 Workspace 规则不够"与"租户外的人不可批准"——Agent 不持有成员会话，因此不构成推送请求。**边界照实写明**：这一条的直接证据是**身份闸门的负向控制**与批准路由的成员要求，验收本身没有驱动一个 Agent 会话来尝试请求推送（该会话不是 Cloud 的 API 客户端） | Covered |
| 远端许可不构成授权 | **已取得（2026-10-13 由 `Missing` 转为 `Covered`）**。容器级验收在**推送开关关闭**的部署上跑：远端可写、可达、凭据在位，`baseline/push-route-status` 是 500 `credential_executor_unavailable`，批准结算为 `failed`／`push_executor_unavailable`，而 `baseline/no-authorization-was-signed` 断言 Store 卷里**没有**该批准的消费记录、`baseline/nothing-reached-the-upstream` 断言 **0 个请求到达远端且 `main` 仍在 `fa9e902e…`**。第二条同向证据：`refusal/a-clone-only-reference-is-refused-by-capability`——同一份 token 在这台远端上确实能写，但那份引用只声明 `["clone"]`，推送被 `repository_credential_capability_unsupported` 拒绝，**没有签出任何授权** | Covered |

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

> **2026-10-13 修订（只在此处登记，标题与 ADR 原文逐字保留）**：本节标题里的 "Execution Generation"
> 与本节的"执行／运行代次"两项**已被那篇词汇表扩展决策的裁决一并修订**（那篇同样 `proposed`）——推送**不**绑定 clone 的
> `ExecutionID`，**不**绑定 Node 运行代次与 Node 实例，也**不**绑定 `PreparationGeneration`。代之以由
> Store 从**已校验的客户端证书**解析的 `ExecutorIdentity`（不接受请求体自述），加 `OperationID`、单次
> 消费与短有效期。决策文本的正文已按它自己的前向注记约定同步（见
> [推送执行权限](../../../decisions/node/repository/20261012-push-execution-authority.md) 裁决结论前向注记 1）。
> **拒绝形状不变**：租户、Project、Workspace、仓库身份、凭据 id 与版本、ref 与提交、有效期继续必填，
> 缺一项继续拒绝。下一节的三条"必须成立"与"禁止结果"中，只有"运行代次"一项按此修订读。

- 授权绑定租户、Project、Workspace、执行、运行代次与 Node 实例、规范化的仓库身份、凭据 id 与版本、
  ref 范围、一次性与有效期；**缺一项即拒绝**。
- 运行代次是 Node 的运行代次，不是 Controller 租约的 `epoch`。
- 仓库身份逐字节比对，不做前缀或模式匹配。

### 禁止结果

缺少任一绑定项仍被接受、用 Controller 租约 epoch 代替运行代次、按前缀匹配仓库。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 授权绑定九项且缺一即拒 | **已取得（2026-10-13 由 `Missing` 转为 `Covered`，绑定项按上节修订读）**。签发侧：`cloneauth.TestIssueRejectsIncompleteBindings`（表驱动，逐项省略即拒；写向的 ref 与提交现已在该表内）、`cloneauth/push_test.go` 的 `TestPushBindingRoundTrip` 与 `TestPushRefusesTheOtherOperationsFields`（clone 的执行 id 与运行时代字段在 push 载荷里是**禁止字段**）。兑换侧：`credentialstore/push_authorization_test.go`（Store 以自己的状态独立复核绑定与用途，不依赖调用方已校验）。容器级：`identity/a-forged-authorization-cannot-cause-a-write`（**伪造的授权被 403 拒绝**，Store 记下这次 resolve 与原因）、`refusal/a-counterfeit-authorization-is-refused-by-the-store`（**有权兑换的那个身份**出示一份未签名的授权，403 `authorization_invalid`）、`identity/a-job-naming-more-than-one-ref-is-refused`（一个 ref 字段里写两条 ref 被 400 拒绝）。这四条合起来是"缺项或错配就走不到写入"的直接证据 | Covered |
| 被替换的代次不得继续推 | **按修订改写后已取得（2026-10-13 由 `Missing` 转为 `Covered`）**。修订把 fencing 的意图交给"执行者身份＋单次授权＋有效期"，本行据此改为"**别人的、过期的、已消费的授权都不能写入**"：`identity/a-peer-that-is-not-the-dispatcher-cannot-ask-for-a-write`（另一个身份连握手都过不去）、`identity/the-resolving-identity-is-admitted-and-refused-for-the-authorization`（过了身份闸门，仍因授权不是给它的而被 403）、`cloneexec.TestARedemptionCannotBeSpentOnAnotherPurpose`、`cloneexec.TestOneApprovalCannotWriteTwice`（同一份授权不能被花两次）、`credentialstore` 的过期与重放用例。**边界**：这证明的是"出示方拿不到写入权限"，不等价于 clone 那条"运行代次被替换"的 fencing——后者在推送里**不存在**，因为 C 不运行在任何 Node 上 | Covered |

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

> **2026-10-13 收窄（只在此处登记）**：那篇扩展决策把写入面收窄为"**一次授权恰好一个 upstream、一个完整
> ref 名、一对提交**"，因此本节的"允许更新的 ref 集合"在实现里是**一元**的：不存在"授权内的一部分 ref"，
> 也就不存在"授权外的 ref 但仍在同一个批准里"这种状态——换一条 ref 就是另一份批准、另一份授权。默认拒绝
> 因此是结构性的，而不是靠一张白名单比对。

- 未写明的 ref 一律不可更新；ref 名按完整名字比对，不承认 `refs/*`、`*` 之类的模式。
- force、删除 ref、删除 tag、改默认分支指向都在**准入处**拒绝，而不是依赖远端拒绝。
- Git host 自己的保护规则与审计仍是最终判据；被远端拒绝如实记为失败，不由 Cloud"修正"。

### 禁止结果

把范围交给远端判、按前缀通配放行、把远端拒绝改写成成功。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 未写明的 ref 在准入处被拒 | **已取得（2026-10-13 由 `Missing` 转为 `Covered`）**。准入处：`cloneauth.ValidateRef`（要求 `refs/heads/` 的完整拼写，拒绝通配、force 前缀 `+`、空白与穿越拼写）与 `cloneauth.ValidateCommit`（要求完整小写 id，拒绝空对象 id）在 `core/push_request.go` 的接受路径上逐项强制，失败即 400 `invalid_push_ref`／`invalid_push_commit`——发生在**签出任何授权之前**。测试：`cloneauth.TestValidateRef`、`core.TestARequestThatNamesAnythingButOneBranchUpdateIsRefusedAtTheBoundary`。容器级：`identity/a-job-naming-more-than-one-ref-is-refused`（一个 ref 字段写两条 ref → 400）与 `write/only-the-named-ref-moved`（同一次写入后另外两条分支逐字节未动） | Covered |
| force 与删除不在允许范围内 | **已取得（2026-10-13 由 `Missing` 转为 `Covered`）**。`cloneexec.TestTheWriteCommandIsOneRefOneCompareAndSwap` 钉住写命令的形状——一次一个 ref、带 `--force-with-lease=<ref>:<old>` 的比较交换、**不含** force 前缀与删除语义；`cloneexec.TestTheWriteCommandRefusesAnAddressOrRefItWillNotDial` 钉住命令构造对地址与 ref 的自拒；空的 old／new 对象 id（删除与"从无到有"的拼写）被 `ValidateCommit` 在准入处拒绝。容器级同向证据：`write/the-remote-was-asked-for-exactly-one-ref-update`——远端钩子日志里恰好一条更新行，没有删除行，也没有 `+` 前缀 | Covered |

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
- **helper 的作用域精确到 host＋path**：helper 把物化 scope 的 `host` 与 `path` 两行与 Git 请求
  规范化后的 host＋path 逐字节比对，因此同一 host 下的另一个仓库借不到这份 scope。这是 D4 要求的收紧，
  **已落地并单独验收**（见下表的第三行；2026-10-13 更正：本行原先写「今天只比对 `host`」，已不成立）。
  它与推送的启用互相独立——收紧读向不启用写向。

### 禁止结果

helper 只按 host 放行、物化残留、执行结束后授权仍可兑换。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 秘密不进任何持久媒介与 Workspace 卷 | **已取得（2026-10-13 由 `Missing` 转为 `Covered`）**。容器级验收的隔离阶段在真实栈上逐个容器读：`isolation/the-token-is-in-no-container-environment`（扫过 19 个容器的环境，命中 0）、`isolation/the-token-is-in-no-container-log`（同样 19 个容器的完整日志，命中 0）、`isolation/the-store-holds-no-plaintext-token`（Store 卷里只有密封凭据、没有明文）、`isolation/no-runtime-mounts-the-store-or-the-executors-material`（70 个 Node 容器没有一个挂载了 Store 卷或执行器物化卷）。Cloud 侧同向：`refusal/the-token-is-not-in-clouds-credential-metadata` 对 `credential_refs` 与 `idempotency_records` 的全部文本做否定断言。写向的销毁侧由 `cloneexec.TestAPushThatCannotReachTheUpstreamIsReapedAndLeavesNoSecretBehind` 在进程内覆盖 | Covered |
| 四种结局都销毁物化 | **部分取得（2026-10-13 由 `Missing` 转为 `Partial`，不写成 `Covered`）**。已有直接证据的两条：**成功**——容器级 `write/the-staged-secret-did-not-outlive-the-write`（一次完成的写入之后物化根与暂存根都是空的）；**失败与超时**——`cloneexec.TestAPushThatCannotReachTheUpstreamIsReapedAndLeavesNoSecretBehind`。**中断这条的实测与义务不符**：`interruption/a-killed-executor-leaves-the-staged-secret-behind` 在写已落到远端之后 `docker kill` 执行器 C，物化根里**实测留下 11 个路径，其中 1 个是明文 token**（`/var/lib/clone-executor/material/<授权 id>/token`，检索退出码 0，即搜索确实执行过）。被杀的进程不执行收尾，残留由**下一次启动的清扫**销毁——`interruption/the-next-start-recovers-the-residue` 读回启动行 `sweptEntries: 1, sweptFetches: 0, sweptPushes: 1`，清扫后残留为 **0**，进程内对应 `cloneexec.TestTheStartupSweepRemovesAnAbandonedScratchRepository`。因此这一行的准确说法是：**成功、失败、超时三条物化即刻不存在；中断那条的明文在磁盘上存活到下一次启动**，残留已如实记录 | Partial |
| helper 精确到 host＋path 的作用域校验 | **已落地并单独验收**（2026-10-13 由 `Missing` 转为 `Covered`）。物化不再是裸 host，而是 `host=`／`path=` 两行记录（`cloud/internal/credentialmaterial/material.go` 的 `Stage`／`formatScope`，旧格式在 `ReadScope` 处被**拒绝**而不是按 host 解读）；两端的比对都走同一个 `NormalizeScope`（host 小写并去掉 `:443`，path 保留大小写、去掉一个前导 `/`、一个尾随 `/` 与一个尾随 `.git`，不做百分号解码），执行器把**规范化仓库**而非请求自述拆成 host 与 path 交给 `Stage`（`cloud/internal/cloneexec/executor.go` 的 `stage`／`repositoryScope`）。直接证据：`cmd/git-credential-ora.TestRunRefusesARequestThatDoesNotMatchTheScopedRepository` 的 14 个用例——其中「a different repository on the same host」（`ora/other-repo.git`）与子路径、父路径、大小写不同、百分号编码、带 query、带 fragment、另一 host、同名后缀 host、无 path 各自被拒，另有三个同仓库的不同拼写被放行作为对照；`TestRunMatchesTheHostCaseInsensitivelyButNotThePath`、`TestRunRefusesAScopeFileInAnOldOrCorruptFormat`、`TestNormalizeScopeReducesOnlyTheDecorations`、`TestStageRefusesAScopeItCannotNormalize` 分别钉住大小写规则、旧格式拒绝、归一化边界与无法归一化的输入。**可证伪**：把 helper 的比对从「host 与 path 都相等」改成「只比 host」，同一用例有 **8 个子用例失败**（含 `a different repository on the same host`），`go test -count=1 -run TestRunRefusesARequestThatDoesNotMatchTheScopedRepository ./cmd/git-credential-ora/` 退出码 1；恢复后退出码 0，文件 sha256 与改动前逐字节相同。**边界照实写明**：helper 是读写两个方向共用的同一个组件，而这条证据测的是**共用 helper 自身的行为**，不区分这一次请求会走向读还是写——这与 D4 把该收紧定性为「同时收紧读向、可在不启用推送的前提下先落地」一致，它**不**构成任何写方向**执行**的证据（写向执行的证据登记在文末那张 D9 表里，其中最相关的是 `refusal/a-clone-only-reference-is-refused-by-capability` 与 `refusal/a-counterfeit-authorization-is-refused-by-the-store`） | Covered |

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
| 授权不可重放、重试必须重新签发 | **已取得（2026-10-13 由 `Missing` 转为 `Covered`）**。容器级：`baseline/a-repeat-of-the-request-returns-the-recording`（开关关闭时重发同一个 requestId 只回放记录、**不派发**）、`refusal/a-replayed-approval-is-not-dispatched-again`（结算之后重发同一 requestId 返回同一条批准与同一个终态、远端 0 个新请求、Store 卷记录逐字节不变）、`write/a-second-approval-moves-only-its-own-ref`（要再写一次必须**再批一次**）。进程内（真实 Store 卷与真实 mTLS）：`credentialstore` 的 `TestResolveConsumesTheAuthorizationOnce`（`O_CREAT\|O_EXCL` 即原子单次判定）、`TestResolveSurvivesARestart`（**重启后重放仍被拒**，对应 D9 第 2 条）、`cloneexec.TestOneApprovalCannotWriteTwice`。**边界照实写明**：容器级验收**没有**测量"出示一份已花费但真实的授权被拒"——Cloud 从不持久化 bearer 授权，验收侧也无法在不知道部署私钥的前提下造一份，因此那一格由 `internal/credentialstore` 的进程内真实 mTLS 用例承担；容器级测量的是"重发同一个批准不会再兑换、不会再写入" | Covered |
| 部分成功如实记录 | **改写后成立（2026-10-13 由 `Missing` 转为 `Covered`）**。那次收窄（"一次授权恰好一个 ref"）使"部分成功"**结构性不可达**：没有多 ref 的一次推送，也就没有"一部分 ref 成功、一部分失败"可记。可证伪的直接证据是这一元性本身——容器级 `write/the-remote-was-asked-for-exactly-one-ref-update`（远端钩子日志恰好一条更新行）、`write/only-the-named-ref-moved`（另两条分支未动），进程内 `cloneexec.TestTheWriteCommandIsOneRefOneCompareAndSwap`。**这一行不是"暂时没测到"，而是义务的对象不存在**；若将来放宽为多 ref，本行必须重新开张 | Covered |

### 决策依据

D7 与不变量 3、11。

## Push Stays Unrequestable Until The Vocabulary Is Extended By Its Own Decision

### 风险

把本 ADR 的裁决读成"推送已可用"，从而在词汇表之外偷偷放开一个写能力。

### 前置状态

> **2026-10-13 后的实际前置状态（只在此处登记，标题逐字保留）**：本节标题所说的那个条件
> ——"直到词汇表由它自己那篇决策扩展"——**已经在 2026-10-13 被满足**：`capabilities` 现在是
> `["clone","push"]`，`push` 已不是词汇表外的成员。因此标题所命名的"不可被请求"**是一个已经结束的状态**，
> 而不是一条今天的承诺。下面两条义务在这件事上因此**一分为二**：标题里的那半（"只能由独立决策扩展"）
> 已按规则办完并留证；另一半（"一次读授权永远不是写许可"）是**不随词汇表变化的长期承诺**，继续按下表登记。
> 上一次修订（把 `["push"]` 从"被拒"改成"可被签发"）本身就是那篇决策的落地，所以下面两条的证据引用全部
> 换成了今天真实存在的测试。

### 触发

以任何路径请求一次推送；或以 `push` 作为绑定/执行计划里的能力值。

### 必须成立

- `push` 加入枚举需要另一篇 `approved` 决策，并同时变更公开契约枚举与准入、签发、执行三处强制点。
- 一次推送需要写能力成员，不得把 `clone` 读成写许可。
- 本 ADR 的裁决不解除任何 fail-closed：带凭据的 clone 继续以 `repository_credential_executor_unavailable`
  逐位拒绝。

### 禁止结果

以"权限模型已裁决"为由启用推送、把只读授权复用为写许可、在词汇表之外放行任何写路径。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 词汇表只能由它自己那篇独立决策扩展，且扩展必须同时落在四处强制点 | **已按规则办完（保持 `Covered`，引用全部改写）**。扩展决策是 [`20261013-push-capability-vocabulary-extension.md`](../../../decisions/cloud/identity-access/20261013-push-capability-vocabulary-extension.md)（`proposed` 2026-10-13，A–G 全部按建议同意）。四处强制点在**同一次变更**里就位（cloud `16e0c40`：公开契约枚举、准入、签发、执行），并由今天真实存在的测试钉住："`push` 是词汇表的第二个成员且顺序为 `clone` 在前"由 `cloneauth.TestCapabilityVocabularyIsClosedToWhatTheDecisionFixes` 断言（它同时断言 `read`／`write`／`fetch`／`pull`／`admin` **仍然**在词汇表之外——把它们写成测试常量而不是"看起来完整"的枚举，正是为了让下次扩展又得先改决策）；`cloneauth.TestCanonicalCapabilitiesRefusesEverySetItCannotHaveIssued` 断言 `["push"]` 与 `["clone","push"]` 可被签发，而空集、重复、非规范顺序、大小写变体与词汇表外的成员一律被拒；`cloneauth.TestIssueRejectsIncompleteBindings` 新增 "a set the clone operation does not permit"（`["clone","push"]` 作为 clone 的授权被拒）；`cloneauth.TestHasCapabilityComparesTheCanonicalSpelling` 断言 `["clone"]` 对 `push` 判 false、`["push"]` 对 `clone` 判 false | Covered |
| 一次 clone 授权不得被当成写许可 | **已取得（保持 `Covered`，引用全部改写为写向真实存在的测试）**。签发与验签侧：`cloneauth.TestPushRefusesTheOtherOperationsFields`（clone 的执行 id、Node 实例与运行代次在 push 载荷里是禁止字段，反向亦然）、`cloneauth.TestPushBindingRoundTrip`（一份 push 授权携带的恰好是 `["push"]`）、`cloneauth.TestAPushAuthorizationIsNotReadableAsEitherReadOperation`、`cloneauth.TestAnUnknownOperationIsNotReadAsAPush`、`cloneauth.TestTheTwoPurposesRefuseEachOthersFields`。兑换侧：`cloneexec.TestARedemptionCannotBeSpentOnAnotherPurpose`、`credentialstore/push_authorization_test.go`（用途在签名载荷内，由 Store 验签后比对）。容器级：`refusal/a-clone-only-reference-cannot-be-pushed-with`（那份引用声明的能力恰为 `["clone"]`）、`refusal/a-clone-only-reference-is-refused-by-capability`（同一份 token 在远端正可写，推送仍被 `repository_credential_capability_unsupported` 拒绝，且**没有签出任何授权**）。能力词汇表不变量 5 登记在[凭据能力词汇表](../../cloud/identity-access/repository-credential-capability-vocabulary.md) | Covered |

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
| 真实私仓推送在受控通道内逐条成立 | **部分取得（2026-10-13 由 `Missing` 转为 `Partial`；2026-10-14 仍为 `Partial`，不写成 `Covered`）**。已有三次真实组件上的端到端运行：`cluster/scripts/push-acceptance.py` 在专用隔离栈 `ora-private-mirror` 上驱动真实 Cloud、真实 Credential Store 与真实执行器 C，写入一个由验收自己启动的**真实 HTTPS Git 远端**（该远端的 `push-git-fixture` 逐请求裁决 token，并具备 receive-pack 前后的钩子）。修复前（cloud `2bd610d`）：58 项断言、**57 成立**、退出码 1，证据 `cluster/.acceptance/push-acceptance/push-acceptance.json`；修复后（cloud `d49edea`，重建镜像、`--only interruption`）：**20/20 成立**、退出码 0，证据 `cluster/.acceptance/push-acceptance-fixed/push-acceptance.json`；同日完整矩阵（同一批镜像、不再重建）**58/58 成立**、退出码 0，证据 `cluster/.acceptance/push-acceptance-full-fixed/push-acceptance.json`。逐条对照见下节。**两条仍未达成**：D9 第 1 条的"真实私有仓库"在这里是**验收自己的受控 fixture**，不是第三方私仓——授权 §2 明令不使用真实 GitHub PAT、不访问真实外部仓库，因此这一格**只能**这样取；D9 第 5 条的中断残留仍是"被杀时留在卷里、由下一次启动清扫"。第 7 条原先被实测**违反**，已于 cloud `d49edea` 修复并在重建镜像上重测通过。也正因第 1、5 条如此，本行不写成 `Covered` | Partial |

### D9 九条在容器级验收里的逐条实测（2026-10-13）

下表把 D9 的九条判据逐条对到实际测量。**没有任何一条因为"实现存在"而被记为有证据**；未达成的写未达成。

| D9 | 判据 | 直接证据 | 状态 |
|---|---|---|---|
| 1 | 真实私有仓库、真实远端、受控凭据，不得用 mock／内部交付 ref 通道／定向单测顶替 | 真实远端与真实凭据裁决都有（见上）。**但仓库是验收自己的 fixture**，不是第三方私仓：授权禁止访问真实外部仓库与真实 PAT。边界照实写明，不用它顶替"真实私仓"四个字 | Partial |
| 2 | 一次授权消费一次即失效，重放被拒（含重启后仍被拒） | 容器级：`baseline/a-repeat-of-the-request-returns-the-recording`、`refusal/a-replayed-approval-is-not-dispatched-again`（Store 卷记录逐字节不变、远端 0 个新请求）、`interruption/the-authorization-was-spent-exactly-once`（被杀的 C 那次也恰好消费一次）。重启：`credentialstore.TestResolveSurvivesARestart` 在同一**真实卷**上关掉再重开 Store，已消费的授权仍被拒。**边界**：重启的是进程内的 Store 对象，不是容器 | Covered |
| 3 | ref 与提交严格按授权执行：授权外的 ref、old 不匹配、非后代、force／delete／wildcard 都被**准入处**拒绝 | `cloneauth.ValidateRef`／`ValidateCommit` 与 `core.TestARequestThatNamesAnythingButOneBranchUpdateIsRefusedAtTheBoundary`、`cloneauth.TestPushRefusesTheOtherOperationsFields`（短名、tag、`refs/heads/*`、缩写提交逐项被拒）；非后代与 old 不匹配由 C 在自己的裸仓上先判（`merge-base --is-ancestor`），`cloneexec.TestAWriteTheUpstreamDoesNotCarryIsRefusedWithoutReachingTheNetwork`。容器级：`identity/a-job-naming-more-than-one-ref-is-refused`（400）、`write/the-remote-was-asked-for-exactly-one-ref-update` | Covered |
| 4 | Agent（身份 W）读不到秘密：**负向控制**证明物化根与兑换响应都不可达 | 容器级：`isolation/node-containers` 扫过 **70** 个 Node 容器，`isolation/no-runtime-mounts-the-store-or-the-executors-material`（没有一个挂载这两卷）、`isolation/the-runtime-identity-cannot-read-the-executors-material` 与 `...-the-store-volume`（以 uid 1000 —— 正是会话与 Agent 的那个身份 —— 读两卷都是 `denied`，mode 0700、属主是别的 uid）、`identity/the-resolving-identity-is-admitted-and-refused-for-the-authorization`（有资格兑换的身份拿到的是拒绝，不是秘密）。这是**负向控制**（走到底、读到拒绝），不是"没看到" | Covered |
| 5 | 执行结束（成功、失败、超时、中断）后物化被销毁，残留被如实记录 | 成功 `write/the-staged-secret-did-not-outlive-the-write`；失败与超时 `cloneexec.TestAPushThatCannotReachTheUpstreamIsReapedAndLeavesNoSecretBehind`。**中断不成立**：`interruption/a-killed-executor-leaves-the-staged-secret-behind` 实测残留 11 个路径、其中 1 个是明文 token，由下一次启动的清扫销毁（`sweptEntries: 1, sweptPushes: 1` → 0） | Partial |
| 6 | 撤销在写入前生效即阻止写入，撤销后对账不重放 | `core.TestAPushIsNotDispatchedWhenItsAuthorityIsGoneByTheTimeTheWriteWouldRun`：在"批准已提交、尚未派发"的窗口里逐项改一处活状态（凭据被撤销／被轮换／Project 被重绑／凭据被摘除／Project 被删），每例断言**确切的稳定码**且**执行器一次都没被调用**；`core.TestAPushIsStillDispatchedWhenNothingAboutItsAuthorityChanged` 是对照，防止"一律拒绝"也能过。写没发生，因此不存在"撤销后对账重放"；重放本身见第 2 条 | Covered |
| 7 | 不确定终态被记为不确定，并能按远端当前值对账出三种结果 | 三格里的两格有实测：`refusal/a-target-reached-by-someone-else-is-not-attributed`（远端 = new → `uncertain`／`push_attribution_unknown`）、`refusal/a-refused-update-settles-not-applied`（远端 = old → `failed`／`push_not_applied`，对账读只读不写）。`refusal/a-lost-answer-settles-unknown` 与 `refusal/a-lost-answer-is-not-followed-by-a-read` 证明"答案丢失 → `uncertain`／`push_result_unknown` 且**不做**第二次读"。**中断这一格：先被实测违反，修复后重测成立**。修复前（cloud `2bd610d`）`interruption/an-interrupted-write-is-not-a-concluded-failure` 失败——写入被远端接受之后 C 被杀，Cloud 记的是 `failed`／`credential_executor_unavailable`。该缺陷在 cloud `d49edea` 修掉（无原因码的错误**原样**透传，不再被替换成 `ReasonCodeUnavailable`），并在**重建后**的镜像上重测：`interruption/an-interrupted-write-is-not-a-concluded-failure` 成立，报文 "Cloud recorded state=uncertain code=push_result_unknown for a write the remote accepted"，读数 `interruption/state = uncertain`、`interruption/result-code = push_result_unknown`、`interruption/observed = null`（未把只读对账错记成观察值），同一次运行 20/20 项成立、退出码 0，证据 `cluster/.acceptance/push-acceptance-fixed/push-acceptance.json`（镜像 `ora-cloud:d49edea`／`ora-clone-executor:d49edea`）。同日又在**同一批镜像**上重跑了**完整的 58 项**（八阶段全跑、无 `--only`）：**58/58 成立、退出码 0**，这一格在其中同样是 `uncertain`／`push_result_unknown`、`observed` 为空、远端 `refs/heads/main` 为新提交、Store 恰好一次 `delivered`，证据 `cluster/.acceptance/push-acceptance-full-fixed/push-acceptance.json`。同一提交还留下这一格最小的进程内复现：`core.TestAnInterruptedWriteIsNotSettledAsAConcludedFailure` 经真实受理路由、真实 `dispatchPush` 与真实结算驱动三个子用例；**它在没有 `TEST_DATABASE_URL` 时是 `Skip` 而不是通过**，因此 2026-10-14 特意在真库上跑过一次确认三个子用例都执行且通过。修复前的 57/58 与退出码 1 **原样保留**在 `cluster/.acceptance/push-acceptance/{push-acceptance,run6,run7}.json`，未被覆盖。**边界**：三格里被"按远端当前值对账出三种结果"覆盖的仍是前两格（远端 = new、远端 = old）；第三格（远端不可读）由 `a-lost-answer-settles-unknown` 与上面这次中断实测承担 | Covered（2026-10-14 由 `Partial`（含一处实测违反）转 `Covered`） |
| 8 | 秘密不进入 Node、Agent、浏览器、Cloud 业务库、日志、存储卷或 Revision | 容器级：`isolation/the-token-is-in-no-container-environment`（19 个容器的环境，命中 0）、`...-the-token-is-in-no-container-log`（同 19 个容器的完整日志，命中 0）、`isolation/the-store-holds-no-plaintext-token`；Cloud 侧 `refusal/the-token-is-not-in-clouds-credential-metadata`。部署级日志扫描见 `cluster/docs/issue-16-acceptance-matrix.md` §B 末行 | Covered |
| 9 | 只读证据与一次推送成功都不得被复用为写许可（T7） | `refusal/a-clone-only-reference-cannot-be-pushed-with` 与 `refusal/a-clone-only-reference-is-refused-by-capability`（同一份可写 token，只有 `clone` 能力 → 推送被拒且**没签出授权**）；成功之后要再写必须**再批一次**：`write/a-second-approval-moves-only-its-own-ref`；结算过的批准不会被重新派发：`core.TestASettledPushIsNeverDispatchedAgain` | Covered |

### 实测到的一处产品缺陷：发现、修复与重测（2026-10-14 已修，重测通过）

`interruption/an-interrupted-write-is-not-a-concluded-failure` 失败，报文为
"Cloud recorded `state=failed code=credential_executor_unavailable` for a write the remote accepted"。
当场事实齐备：远端确实接受了这次写入（`interruption/the-write-landed-before-the-executor-died` 读回
`refs/heads/main` = 新提交）、授权恰好消费一次（`result: delivered`）、没有第二次写入
（`interruption/no-second-write-was-made`），而 Cloud 把它结算成**已确认的失败**。

成因在代码里可定位：`cloud/internal/cloneexec/client.go` 的 `Client.Push` 在传输失败时返回的是**普通
error**（`clone executor is unreachable: %w`），而 `cloud/internal/core/push_dispatch.go` 的
`pushDispatchFault` 把任何**没有原因码**的错误一律映射成 `&Fault{Status: 409, Code:
cloneexec.ReasonCodeUnavailable}`——那是一个**带码**的故障，于是 `dispatchPush` 结算为
`failed`／`credential_executor_unavailable`。这与 `pushDispatchFault`、`PushDispatcher.Execute` 与
`dispatchPush` 三处的文档注释（都承诺"不可达的执行器可能已经写了"，因此应当**不确定**）以及本 ADR
2026-10-13 修订 2 的"网络中断、响应丢失、进程被杀、执行器失联记为**不确定**"直接冲突。

**发现时的处置是"如实登记、不修"**：改它要动 cloud 的生产代码并再走一次提交审批，所以 2026-10-13 那一次
运行把它原样留在记录里（57/58、退出码 1），本文件与
`cluster/docs/issue-16-acceptance-matrix.md` 的 §C／§E 以同一口径登记。

**2026-10-14 已修并重测**。修复提交是 cloud `d49edea`（`fix(push): settle an interrupted dispatch as
uncertain rather than failed`，6 个文件），改动落在上面定位到的同一处：
`internal/cloneexec/client.go` 的 push 路由在**读不出原因码**时不再替换成
`ReasonCodeUnavailable`，而是返回一个**不带 refusal** 的普通 error；
`internal/core/push_dispatch.go` 的 `pushDispatchFault` 相应地把无码错误**原样**交还，于是
`dispatchPush` 按"未知"结算。另两条路由（clone／mirror）保留原替换，因为那两条**不可能已经写过**。
`PushUnknown`（Cloud 对**远端**的判词）与无码的派发错误（Cloud 读不到**自己的执行器**）仍可区分，两者
都结算为 `uncertain` 但记录不同的组件。

**重测口径**：重建 `ora-cloud:d49edea` 与 `ora-clone-executor:d49edea`（源码摘要校验：新镜像内含
`with no reason code`，旧镜像不含），在同一个隔离栈 `ora-private-mirror` 上以 `--image-tag d49edea
--only interruption` 重跑，**20/20 项成立、退出码 0**，新证据
`cluster/.acceptance/push-acceptance-fixed/push-acceptance.json`。除本例外同一次运行还复测了
`environment`（4）与 `baseline`（8，开机前 fail-closed 的一半），两组全绿。修复前的那一次运行**未被覆盖**：
`cluster/.acceptance/push-acceptance/push-acceptance.json` 仍是 58 项、**57 成立、退出码 1**，
`run6.json`／`run7.json` 原样保留。镜像标签与摘要记在
`cluster/.acceptance/push-acceptance-fixed/images.json`。

**同日再跑一次完整矩阵**：为了不把"修复没有弄坏别的东西"留成推断，同一天在**同一批已经验证过的镜像上**
（`ora-cloud:d49edea`／`ora-clone-executor:d49edea`，**未重建**）重跑了完整的 58 项——八个阶段全跑、不带
`--only`。结果 **58/58 项成立、退出码 0**，证据
`cluster/.acceptance/push-acceptance-full-fixed/push-acceptance.json`（标签与摘要见同目录 `image-digests.txt`，
命令、阶段与结果见 `provenance.json`）。这一轮没有改动过任何断言：驱动、`--image-tag` 的语义、栈脚本的
`ORA_IMAGE_TAG` 默认值都与修复前那一次相同，变的只有 `--evidence` 指向的新目录与栈跑的镜像标签。修复前那次
运行的证据目录（`.acceptance/push-acceptance/`）与单场景复测的目录（`.acceptance/push-acceptance-fixed/`）
**逐字节未动**。

**仍然照实写明的边界**：D9 第 1 条的"真实私有仓库"依旧只是**验收自己的受控 fixture**（受控 HTTPS 远端，
不是第三方真实私有仓库——授权禁止真实 PAT 与真实外部仓库，这一格只能这样取）；第 5 条的中断残留依旧是
"被杀时留在卷里、由下一次启动清扫"（实测 11 个路径、其中 1 个明文 token，清扫后归零），不是即时销毁。
本节的修复只改第 7 条的结算方向，第 1、5 两条的边界不因它而改变。

### 决策依据

D8 与不变量 12；[推送交付 ref 与取回前序版本](../../../decisions/node/revision/20261012-push-delivery-ref-and-fetch-prior-revision.md)。

## 决策文本里一段过期现状描述的更正（2026-10-13 已裁决并落地）

上文把 helper 的 host＋path 校验记为 `Covered`，依据是当前代码与可证伪的测试；而
[推送执行权限](../../../decisions/node/repository/20261012-push-execution-authority.md)（`proposed`）
里关于该缺口的**现状描述**已与代码不符。这一处已按"只追加前向注记"的方式更正，原文逐字保留：

| ADR 位置 | ADR 原文的说法 | 今天的实际 |
|---|---|---|
| D4「必须同时收紧的一点（现状缺口）」 | 「helper 今天只比对 Git 报出的 `host`」「物化的 scope 文件只存 host」 | 物化写的是 `host=`／`path=` 两行记录（`cloud/internal/credentialmaterial/material.go:48-56` 记录了格式变更与旧格式的拒绝口径），helper 比对 host **与** path（`cmd/git-credential-ora/main.go:84-92`） |
| 同上，引用的行号 | `internal/credentialmaterial/material.go:157` 是 scope 只存 host 的证据 | 该行落在 `ValidExecutionID` 的函数体内，与 scope 无关；scope 的写入点是同文件的 `Stage`（`:211`）与 `formatScope`（`:241`），读取点是 `ReadScope`（`:310`） |
| 同上，引用的行号 | `internal/cloneexec/git.go:126-129` 把 `useHttpPath` 描述成 path 绑定 | 该注释现在在 `internal/cloneexec/git.go:201-207`，且**已经**如实写成「比对 path 与 host」，与代码一致 |
| 裁决边界第 2 条、D4 前置说明 | 「未补齐前不得启用推送」 | 前置条件已满足，但该边界**仍不是唯一的门**：启用推送还受 E 那篇独立的词汇表决策约束 |

**更正方式**：在 D4 与该边界处各追加一段前向注记（日期 2026-10-13），只更正**现状描述与证据引用**。
原文、裁决结论、D4 的决策、不变量 8 与"未补齐前不得启用推送"这一安全边界**逐字未变**。

**末句的更正（同日稍后）**：这一节原先收尾于「E 未被追认，`push` 仍在词汇表之外，本文与那篇 ADR 的其余
`Missing` 项一条都没有转状态」。前两句当时为真，**当天晚些时候即被 E 自己的裁决所取代**——E 没有被"追认"
进本文，它是**照流程走完并经裁决同意**的：`push` 已成为词汇表的第二个成员，本文与那篇 ADR 的其余证据项也据此
逐行重估（见上表）。本节更正的对象从头到尾只是**现状描述与证据引用**，不涉及任何决策内容；E 的裁决、
两处修订的落地与本文的状态改写，都不改变"未补齐前不得启用推送"这条边界——它今天的形态是
**生产配置里两处开关都不存在**，见文首那张表与 `cluster/docs/issue-16-acceptance-matrix.md` §H。
