---
status: proposed
---

# 用户自助 HTTPS Git 凭据用于私有仓库 Clone：Cloud 只保管引用与授权，秘密由独立 Secret Store 承担，Node 认证通道获批前保持 fail-closed

用户可以在 Web 设置页添加、轮换、撤销个人 HTTPS Git Token，创建 Project 时绑定其中一个凭据，使私有
仓库能被 Node 非交互 Clone。Cloud 继续只持有凭据的**元数据与引用**（`credential_refs`），Token 明文由
独立 Secret Store 以加密形式保存；Cloud 业务数据库、公开响应、业务事件、Controller/Node 常规消息、
日志与 Workspace 文件都不持久化 Token。Clone 许可按 tenant／owner／Project／规范化仓库／credential
version／Workspace／execution 逐次签发，持有 credential ID 本身不取得秘密。

本文的**新增内容**是三处此前缺失的契约：(1) Credential Ingress 的接收边界，以及为它提出的、范围受限的
`cloud/AGENTS.md` 正式修订建议；(2) `secret_ref` 到 Controller 的投影裁决；(3) 解除
`repository_credential_executor_unavailable` 的**全部**前置条件。秘密的存储载体、服务间认证与跨存储
一致性由 [用户 Git Token 由独立 Credential Service 承载](20261010-user-managed-git-secret-store.md)
（`proposed`）定义；Node 侧的身份边界、秘密数据流与验收义务由
[受控 Node 的仓库凭据执行使用独立 OS 身份](../../node/repository/20261010-controlled-node-git-authentication-isolation.md)
（`proposed`）定义。本文不重复它们的结论。

本文的内容已由项目负责人在运行 issue `ora-space/circus#29` 的授权中确认方向，但**尚未经过 specs
的 PR 评审**，按 `specs/AGENTS.md` 保持 `proposed`，评审通过后再据此标记 `approved`。**该授权确认的是
决策内容，既不表示已经实现，也不解除任何 fail-closed 门禁**。Node 认证通道的进程隔离方案方向已同时
确认，但
`requireRepositoryAccess` 的 `repository_credential_executor_unavailable` 拒绝路径继续有效，直到
D11-B 的全部条件满足为止。本文不授权实现变更，不改写任何已应用的 migration，不修改任何 approved ADR
的决策。

涉及模块：Cloud（`internal/core`、`internal/contract`、`internal/api/router`、`internal/core/migrations`）、
Web 设置页、Secret Store／Ingress（新组件，尚未实现）、desktop Controller 与 Node、cluster 部署（Compose）。
兼容性边界：`credential_refs` 表、已应用的 `0001`／`0023` 迁移、已发布的公开 API 形状，以及
`projects.credential_ref_id` 的既有语义都按硬边界处理，只做前向兼容演进。本 ADR **不承担** SSH 凭据、
GitHub App、fetch/pull/push、团队共享凭据与 ownership transfer 的迁移义务。

## 继承与修改

| 前序约定 | 出处 | 本文处理 |
|---|---|---|
| `credential_refs` 的归属、可用性与冻结语义已由 `0023` 交付；`attribution`／`availability`／`repository_scope`／`capabilities` 六列、两个冻结触发器、恢复扫描与两处引用列的历史来源与既有行为 | [仓库凭据归属连续性](20261010-repository-credential-ownership-continuity.md)（proposed，本叶子前序决策）D1–D6 | 继承，不变：本文不改变冻结、归属与可用性语义，只在其上追加"秘密的持久化载体、逐次授权通道与 Node 侧隔离"。本文 D2 对 `attribution='personal'` 的收窄与 D5 的撤销语义都必须与补录的既有判定相容 |
| 用户 Git 提交身份由用户自助维护（`GET/PUT/DELETE /api/v1/me/git-identity`），且用户级 Git 材料不得随 `users` 整行流向无关读者 | [20260928 用户 Git 提交身份](20260928-user-git-commit-identity.md)（proposed） | 继承，不变：本文的凭据同样是"用户自助的 Git 材料"，同样只经专用端点写入、不进入公开响应与普通读取路径。该 ADR 的存储形态与快照时点与当前实现存在偏差，已在 P0 报告中另行提出修订建议，本文不据此改变任何结论 |
| Cloud 只提供 credential reference，由基础设施注入，Cloud 不保存密钥值 | [cloud/operation 根决策](../operation/0-workspace-runtime-follows-desktop-node.md)（implemented）D2 | 继承，不变；本文细化"基础设施"的形态、授权粒度与失败语义 |
| Cloud 只持有凭据引用、不持有密钥 | [cloud/operation 20260926](../operation/20260926-require-concrete-default-branch.md)（implemented） | 继承，不变 |
| 仓库地址不能含密码／token，拒绝携带秘密的 userinfo、query 或 fragment；不开放任意 Git 选项、shell 字符串、目标路径或逐请求环境变量 | [node/repository 20260918](../../node/repository/20260918-minimal-clone-execution-contract.md)（approved）D1 | 继承，不变；凭据不得回到 URL，也不得由请求指定 helper |
| 非交互访问，凭据由 Node 部署提供；协议不接收秘密；凭据不经业务消息、命令文本、日志或结果传播；认证不依赖人机交互 | [node/repository 20260918](../../node/repository/20260918-minimal-clone-execution-contract.md)（approved）D2、不变量 4 | **修改**：凭据来源由"部署静态配置"扩展为"Cloud 侧 Secret Store 按 Project 逐次授权解析"。D2 其余约束（协议不接收秘密、非交互、不由业务请求指定任意可执行程序）全部保留。此修改须经 D2 的正式 amendment 生效 |
| Sandbox Server 不执行 Git 操作，不注入任何凭据 | [cluster/sandbox-server 20260926](../../cluster/sandbox-server/20260926-docker-sandbox-lifecycle-via-substrate-effects.md)（approved）不变量 7 | **条件性修改**：仅当最终认证通道选择"经沙盒注入"才需要修订；选择沙盒内通道则保持不变。当前 Node 侧方案（D6）选择容器内身份边界，因此本条**保持不变** |
| 私有仓库凭据仍由 Node 的既有部署提供，不在浏览器收集或另建凭据管理功能 | [minicloud/runtime 根决策](../../minicloud/runtime/0-local-web-client-embedded-controller.md)（implemented）D3 | **历史契约，不需 amendment**：该句位于 D3「仅本机访问，不建设认证和安全体系」之内，与同段的"minicloud 不投入生产环境""不增加登录、用户／租户、令牌、权限、TLS"同属对 minicloud 这个本机开发应用的约束。minicloud 已整体退役（[退役决策](../../minicloud/runtime/20260928-retire-minicloud.md)，`implemented`：D1 删除 client／启动器／任务／文档，不变量 1 声明"仓库中不再有浏览器直连 Controller 的入口"），该句因此**已无适用对象**，不构成本文的强制 amendment。适用范围仅限已删除的 minicloud client；当前 Web 客户端是 Cloud 前端，不由该句约束 |
| 凭据只保存引用、类型、范围、能力、版本及可用性；成员离开原子冻结 `personal`／`unknown`；`team` 不因旧 owner 离开冻结；重新加入不自动解冻 | `cloud/docs/runtime-control.md`、`0023` 迁移 | 继承，不变；本文不改变冻结与归属连续性语义 |
| Credentials remain infrastructure references. Never accept, persist, log, return, or commit raw … repository credentials | `cloud/AGENTS.md`（仓库级硬约束，119-123 行） | **修改**：接受用户 PAT 直接改变该规则，必须走正式修订；**权威修订正文由 [凭据规则受限例外](20261010-user-managed-git-token-ingress-rule-exception.md) 承载**（本文 D3 只陈述需求与理由，并链接该文）。原规则其余部分全部保留 |
| 仓库凭据引用属于 Project 聚合 | `decisions/cloud/README.md`（`project` 领域范围） | **已裁决（人工）：本叶子三份凭据决策保留在 `identity-access`。** 该裁决优先于 README 对 `project` 领域的措辞；是否同步调整 README 的领域描述另见"本决策未解决的问题" |
| `secret_ref` 是不透明定位标识，不是 bearer token；解析需同时具备 Cloud 逐次授权与 Store 服务身份；写入／撤销与解析分权 | [用户 Git Token 由独立 Credential Service 承载](20261010-user-managed-git-secret-store.md)（proposed）D3、D9 | 继承，不变；本文只决定 Cloud 侧是否把该引用投影给 Controller（D9） |
| Broker 授权短时、单次、绑定执行与 Node 实例，且自身是秘密，不落持久介质；授权在签发时**未使用**，由 Store 在兑换时**原子消费**，消费记录持久在 Store 侧，重放一律拒绝 | 同上 D5、D10 | 继承，不变；消费侧的校验与强制由 node/repository 20261010 定义，`broker-resolver` 服务身份由凭据执行身份 C 持有（Node ADR D15）。Cloud 侧只负责签发，并在每次签发前重新校验归属、冻结、版本与执行状态 |

## 背景与现状

当前实现并非从零开始，但也没有可用的私有仓库路径。事实（以现有代码为准）：

- `credential_refs`（`0001_core.sql:18-22`）保存 `secret_ref`——一个**明文的不透明引用字符串**，不是
  密钥值，也不是密文。`0023_repository_credential_continuity.sql` 追加 `attribution`、`availability`、
  `repository_scope`、`capabilities` 与两个冻结触发器。
- 创建 Project 时可绑定凭据：`createProject`（`internal/core/public.go:549-556`）校验引用属于
  `tenant_id + owner_user_id=调用者 + purpose='git' + deleted_at IS NULL`，写入
  `projects.credential_ref_id`。`PATCH /projects/:pid` 只接受 `name`／`version`，不接受凭据变更。
- **没有凭据 CRUD API**：唯一的写入口是部署 CLI `cloudctl -command credential-ref`（`cmd/cloudctl/main.go:55`），
  其调用目标 `ConfigureCredential`（`internal/core/store.go:550-558`）的 doc comment 明确
  *"deliberately a deployment-only management path, never a public secret API"*，并且把部署方提供的
  任意非空、≤1024 字符串直接写进 `secret_ref`。公开 `Project` schema 不含 `secretRef`。
- `requireRepositoryAccess`（`internal/core/repository_credentials.go:6-26`）校验 tenant、可用性、
  owner 仍为 active 成员、`repository_scope` 匹配，然后**无条件** `reject(409, "repository_credential_executor_unavailable")`
  （`repository_credentials.go:25`）。代码注释写明：*"never substitute a global Git/SSH credential or issue
  an anonymous clone for a credentialed project."* 因此今天**任何带凭据引用的 Project 都无法完成 clone**，
  这是有意的硬拒绝，不是未完成。
- **`secret_ref` 已被投影给 Controller，且投影了两次，而 Controller 完全不用它。**
  `snapshot()` 的 `SELECT p.*,c.secret_ref FROM projects p LEFT JOIN credential_refs c …`（`control.go:174`）
  进入 `ControllerProject`（`internal/contract/openapi.go:269-270`：`secretRef`、`credentialRefId` 与
  `repositoryCredentialRefId` 都在可选集内），并由 `controlgrpc/operations.go:174` 映射到 gRPC 的
  `OperationProject.CredentialRef`（`cloud/proto/ora/cloud/internal/v1/operations.proto:92`，
  `optional string credential_ref = 4`）。
  该 gRPC 字段名读起来像"凭据 id"，实际承载的是 `secret_ref`，而真正的凭据 id 是 JSON 侧的
  `credentialRefId`——命名与内容不一致本身就是一个易误用的点。`desktop/` 全仓检索只有两处命中：生成的
  `credential_ref` 绑定与一个把它设为 `None` 的测试夹具，**没有生产消费方**。
- **两条读路径对权威绑定列的选择不一致。** `requireRepositoryAccess`（`repository_credentials.go:8-12`）
  优先 `repositoryCredentialRefId`、为空再回退 `credentialRefId`；而 `snapshot()`（`control.go:174`）的
  LEFT JOIN 只读 `p.credential_ref_id`。裁决见 D7。
- 协议层没有凭据字段：`CloneExecutionSpec { node_id, repository, branch }`，doc 为
  *"Clone intent has no Main Workspace, destination, credentials or caller-selected Git flags."*
- controlled Node 禁止任何凭据机制：`validate_cloud_policy`（`apps/ora-node/src/repository/config.rs:28-55`）
  拒绝非 `Disabled` 的 SSH，且 git config 只允许 `[http] sslCAInfo` 一行。该策略在 `worker.rs:50-52` 只在
  `controlled` 时生效。
- **Node 侧 W == C 已经是部署事实，不再是假设。** `sandbox-server/node-config.example.json` 设置
  `process.workload_uid: 1000`；`managed.rs:39-55` 用它包装**每一个**受管 Git 命令
  （`setpriv --reuid --regid --clear-groups --no-new-privs`），`service/agents.rs:87-95` 又把**同一个**
  `workload_uid` 作为 Agent 的 `LinuxChildIdentity`。容器入口脚本的注释直接写明
  *"Git workloads run as UID/GID 1000 through the scoped process boundary, and so do Agent plugins"*
  （`desktop/docker/node-entrypoint.sh:5-7`）。因此"Agent 读得到 Git 的凭据"是当前部署的**既成事实**。
- **Git 的配置来源存在 validate-then-use 缺口，且仓库本地配置未被收窄。** `git_config` 只在启动时按内容
  校验一次（`config.rs:32`），此后每次 clone 都经 `GIT_CONFIG_GLOBAL` 重新读取该文件（`config.rs:116`）；
  同时 `constrain()`（`config.rs:140-161`）**没有**清除 `credential.helper`、`http.<url>.*` 或代理键，
  而 checkout 内的 `.git/config` 对 Git 仍然有效（`-c` 优先级更高，但未被用于这些键）。凭据路径不能
  依赖一个可变的磁盘配置来约束自己。
- 现有冻结语义已实现并有真实 PostgreSQL 证据；`attribution='team'`、`capabilities`、
  `repository_scope` 目前**没有任何产品写入路径**（仅测试用裸 SQL 设置）。
- `clone_executions.credential_ref_id` / `credential_ref_version` 列与配对 CHECK 已存在
  （公开契约中已声明为可选：`internal/contract/openapi.go:279-280`），但**从未被写入**，所以没有
  "某次 clone 用了哪个凭据的哪个版本"的持久事实。

结论：本 ADR 要补的不是"凭据表"，而是**秘密的持久化载体、逐次授权通道与 Node 侧隔离**这三段，外加
Ingress 的接收边界、引用投影的裁决与解除门禁的完整条件。

## 决策

### D1：凭据是 Cloud 的用户级资源，`credential_refs` 是唯一业务主表

凭据的归属根是 `tenant_id + owner_user_id`，`purpose` 恒为 `'git'`。新增能力只扩展这张表的**元数据**
与状态，不另建并行的 Git Credential 业务主表；`secret_ref` 只表示"秘密的定位引用"，永远不是 PAT 本身。

为什么不是新建一张表：`0001`／`0023` 已把归属、冻结触发器、`projects` 外键与 `clone_executions` 审计列
都建在这张表上；另建主表会让冻结语义、owner 校验和外键完整性分叉成两套。代价是 `credential_refs`
继续承载"引用"与"新语义"两种角色，实现时必须保证旧列（`secret_ref`）的语义不被静默重解释。

### D2：第一版只覆盖 personal 归属、单仓库 scope、`clone` 能力，且仅 HTTPS

- 只允许 `attribution='personal'`。`team` 的授权与成员离开规则另立决策（与
  [one-space-per-tenant](../tenancy/20260923-one-space-per-tenant.md) 的未决项一致）。
- **产品写入路径必须显式写 `attribution='personal'`，不得依赖列默认值。** 既有部署 CLI 路径不列出该列
  （见"背景与现状"），因此新建凭据落在 `unknown`；产品路径必须显式声明归属，而不是继承这个默认值。
- **禁止在归属未经核实的情况下把既有 `unknown` 自动改写为 `personal`。** 批量转换会把"无法证明是
  `team`、因此按个人保护"静默变成"已证明属于某个用户"，同时改变冻结与解冻的适用范围与 `version`
  的语义（见 [仓库凭据归属连续性](20261010-repository-credential-ownership-continuity.md) D2、不变量 3）。
  确需重新归属时，必须是一次显式、可审计、带依据（`attribution_basis`）的前向变更，而不是一次 `UPDATE`。
- `repository_scope` 第一版是**必需的单仓库许可**，匹配策略必须显式定义：协议、主机、路径、
  大小写、末尾 `.git`、HTTP 重定向。输入只接受 D1 许可的 HTTPS 形态，拒绝 URL 内密码／token／
  敏感 query／fragment，也不接受用户指定的 helper 程序、环境变量或 shell 片段。
- `capabilities` 第一版只有 `["clone"]`；它不蕴含 fetch／pull／push。

为什么 scope 必需：`requireRepositoryAccess` 已经在用 `repositoryScope` 做等值比对
（`repository_credentials.go:22`），但它是可空列且当前无人写入；把"空即通配"留给用户凭据会让一个
Token 意外获得对该 host 上任意仓库的访问。单仓库许可把爆炸半径限制在绑定范围内。

### D3：Token 只经受限 Credential Ingress 进入 Secret Store，并随本文提出范围受限的硬规则修订建议

用户提交的 Token 只能通过一个受限入口写入 Secret Store。该入口的契约是：

- **接收面唯一**：只有专用端点接收 Token；它不接受 Token 出现在 URL、query、header 或任何其他位置。
- **同请求内转交**：Cloud 必须在处理该请求的过程中把 Token 交给 Credential Service，不排队、不落盘、
  不写入中间缓存。
- **不得持久化、不得回显、不得取回**：Cloud 业务数据库、公开响应、业务事件、Controller／Node 常规消息、
  持久日志、审计记录与 Workspace 文件都不得含 Token；写入成功后**不能经任何 API 取回明文**。
- **访问日志与错误上下文同样排除**：请求体不得进入访问日志、追踪 span、错误详情或崩溃报告；错误响应
  只返回原因码。
- **成功后的一致性**：API 只在元数据与秘密都持久化一致后才向用户报告保存成功（跨存储一致性见
  [Secret Store ADR](20261010-user-managed-git-secret-store.md) D4）。

#### 对 `cloud/AGENTS.md` 的正式修订需求（已于 2026-10-10 落地）

`cloud/AGENTS.md:119-123` 的 *"Credentials remain infrastructure references. Never accept, persist,
log, return, or commit raw … repository credentials."* 与本功能的产品方向直接冲突：本功能要求 Cloud
**接受**用户提交的个人 HTTPS Git Token。这不是措辞问题，而是规则范围的实质变化，因此必须走正式修订，
不能靠实现绕过。

**权威修订正文不在此处**：[凭据规则受限例外](20261010-user-managed-git-token-ingress-rule-exception.md)
（proposed）承载这次修订的正文——一段只覆盖"已认证且已授权的用户提交的、**属于自己**的个人 HTTPS
Git Token"的窄例外，附五个缺一不可的条件（用户与归属边界、端点与位置唯一、同请求转交且 Cloud 不持久化、
请求体不进日志与错误上下文、不得扩张到其他秘密对象），并逐条列出例外**不**覆盖的对象（部署私钥、
对象存储凭据、服务账号令牌、SSH 私钥、生成的测试密钥、团队共享凭据）。本文不复制该正文，以免同一段
规则出现两个权威副本。

本文只固定这条约束与本功能的关系：**在修订通过评审并落地之前，该例外不构成任何实现前提**，本功能的
fail-closed 门禁保持有效；规则侧落地也**不**解锁运行时——凭据 clone 的解禁仍需满足 D11-B 的全部条件。

### D4：Clone 许可逐次签发，持有 credential ID 不等于取得秘密

每次进入新的远程 Git 工作（`workspaceCloneDispatch` 与 `runtime_permit` 的两条分支）都必须重新校验
owner／tenant、冻结状态、仓库 scope 与执行器可用性，并签发一次绑定
`tenant + owner + project + 规范化仓库 + credential_id + credential_version + workspace + execution + clone`
的授权。授权本身是**短时**的，但被授权的 PAT 不是——这两者不能混同。

为什么强调这一点：把"短时授权"当成"短时 Token"会让设计者以为泄露窗口只有几秒。实际上只要 PAT 在
GitHub／GitLab 侧仍然有效，泄露后的可用期由该 PAT 自身的有效期决定。Ora 的撤销不等于自动吊销上游 PAT。

### D5：冻结、撤销与轮换沿用 `0023` 语义，撤销只关闭新授权

- 冻结：继续由 `0023` 的触发器与恢复扫描负责，`personal`／`unknown` 在成员或用户停用后原子冻结，
  重新加入不自动解冻。本文不新增第二套冻结规则。
- 撤销：用户的显式撤销映射到 `availability`（`frozen` 或 `invalid`）。注意 `credential_refs.deleted_at`
  目前**没有任何写入者**，只有 `IS NULL` 读取；因此"删除"若要落地必须显式决定它是软删还是不可用，
  不能假定它已经可用（见 D10）。
- 轮换：产生新的 `credential_ref_version`。已完成或正在进行的执行按原版本结算，新的远程 Git 工作必须
  用新版本重新授权，不得静默复用旧凭据能力。
- 撤销后已经派发、正在执行的 clone 如何处置，以及上游 PAT 是否/何时真正失效，属于安全设计范围
  （见"本决策未解决的问题"）。

### D6：Node 侧认证通道必须与 Agent 执行身份隔离，否则本 ADR 不落地

这是本 ADR 的**阻塞性决策**。现状是（见"背景与现状"的对应条目）：

- `ProcessConfig.workload_uid`（`apps/ora-node/src/managed.rs:29`）是一个**单一**可选 UID，通过
  `setpriv --reuid=<uid> --regid=<uid> --clear-groups --no-new-privs` 包装**每一个**受管 workload
  （`managed.rs:39-55`）。
- 该字段在部署模板里**已经**被设为 `1000`，因此 Git clone 与 Agent 会话共用同一个 workload 身份
  （`service/agents.rs:87-95` 用同一值构造 Agent 的 `LinuxChildIdentity`；容器入口脚本的注释也确认
  Git 与 Agent 插件同为 UID/GID 1000）。
- 结论：**任何 git 能读到的凭据，Agent 代码也能读到**。今天不存在"给 git 一个秘密而 Agent 读不到"
  的机制——这不是配置疏漏，而是当前身份模型的结构性结果。

因此本 ADR 明确要求：在 Node 侧隔离方案获评审通过并验证之前，不得解除
`repository_credential_executor_unavailable`。Node 侧的身份边界、秘密数据流与失败语义由
[受控 Node 的仓库凭据执行使用独立 OS 身份](../../node/repository/20261010-controlled-node-git-authentication-isolation.md)
（`proposed`）定义；本 ADR 只负责 Cloud 侧的凭据元数据、逐次授权与冻结语义，不重复其结论。
该 Node 侧决策的方向已由 `ora-space/circus#29` 确认，但本节的门禁**不因方向确认而解除**——D11-B
要求的是可直接指向的证据，而证据尚未产生。

无论最终采用哪种隔离形态，都必须额外验证 Git 的 HTTP 重定向、`http.proxy`、子进程环境继承、
`GIT_CONFIG_GLOBAL`／`core.askPass` 继承与 `credential.helper` 匹配等边界，并证明普通
Workspace／Agent 进程无法读取认证材料。**注意**：`constrain()` 目前并未清除 `credential.helper` 与
`http.<url>.*`，且 checkout 内的 `.git/config` 对后续 Git 调用仍然有效；因此"收窄 Git 配置"是凭据路径
的必做项，不是可选加固。

为什么不能靠"解除 controlled Node 限制"完成设计：`validate_cloud_policy` 的存在正是因为 controlled
Node 的部署方无法保证静态凭据的作用域。简单放开它，等于把"部署可信"这一前提替换成"每个用户提交的
PAT 都可信"，而后者不成立。

### D7：V1 以 `projects.credential_ref_id` 为权威绑定列，另一列保留但不回退

**裁决：V1 的权威绑定列是 `projects.credential_ref_id`（`0001`）。**
`projects.repository_credential_ref_id`（`0023`）**保留不删**，但在 V1 只读、不参与回退、不隐式取值。

| 列 | 出处 | 外键 | 是否强制 owner 一致 | 当前写入者 |
|---|---|---|---|---|
| `credential_ref_id` | `0001_core.sql:25,28` | `(id, tenant_id, owner_user_id)` → `credential_refs(id, tenant_id, owner_user_id)` | **是** | `createProject`（`internal/core/public.go:549-556`） |
| `repository_credential_ref_id` | `0023:9-10` | `(id, tenant_id)` → `credential_refs(id, tenant_id)` | **否** | 无 |

选择 `credential_ref_id` 的三条理由：

1. **它已经在生产代码中实际写入**：`createProject` 校验 `tenant_id + owner_user_id=调用者 + purpose='git'
   + deleted_at IS NULL` 后写入该列；`repository_credential_ref_id` 没有任何写入者。
2. **它的外键在数据库层强制 owner 一致**，不需要额外的应用层拦截；改选另一列会削弱这一强制。
3. **对现有 schema 的改动范围最小**：V1 因此不产生任何 schema 变更，也不需要回填。

**已知代价（必须记录）**：`credential_ref_id` 的外键把 `owner_user_id` 写进约束，因此**结构上无法绑定
owner 与 Project owner 不同的凭据**。这与 V1 的 `personal` 单仓库范围（D2）一致；将来支持
`attribution='team'` 的共享凭据时必须重新裁决（见"本决策未解决的问题"）。

**与当前源码的直接冲突（报告，不静默套用）**：今天两条读路径对权威列的选择并不一致——

- `requireRepositoryAccess`（`internal/core/repository_credentials.go:8-12`）**优先**
  `repositoryCredentialRefId`，为空时才回退 `credentialRefId`；
- `snapshot()`（`internal/core/control.go:174`）的 `SELECT p.*,c.secret_ref FROM projects p LEFT JOIN
  credential_refs c ON c.id=p.credential_ref_id` 只读 `credential_ref_id`，**从不**看另一列。

也就是说，"由哪一列决定凭据"今天取决于读路径，而不是由数据模型决定。本裁决要求实现阶段把
`requireRepositoryAccess` 改为以 `credential_ref_id` 为准并删除回退（冲突拒绝规则见 D10），使两条读路径
一致。**本 ADR 只作裁决，不改代码。**

**未来若切换到 `repository_credential_ref_id`**：必须是前向、可重试的迁移，且必须同时新增等价的 owner
约束（例如把该外键扩展为三列）或明确接受应用层强制；不得修改已应用的 `0001`／`0023` 及其校验和；回填
必须可验证；切换期间旧客户端不得读到半迁移状态。

### D8：`clone_executions` 的凭据审计字段在派发登记时写入

`clone_executions.credential_ref_id` 与 `credential_ref_version` 及其配对 CHECK 已经存在，但从未写入。
本 ADR 要求派发登记时写入二者（只写引用与版本，不写 Token），使"某次 clone 用了哪个凭据的哪个版本"
成为可查询的持久事实。这是幂等、审计与轮换控制的共同前提，且**不需要新的 migration**。该字段是长期
审计义务，不是可选遥测：它必须与执行记录同事务写入，且不得因为授权失败、冻结或执行中止而缺失。

### D9：`secret_ref` 停止投影给 Controller，并按契约版本规则渐进兼容

**裁决：停止把 `secret_ref` 投影给 Controller。** 现状是双重投影，且两条通道都没有消费方。

| 通道 | 位置 | 承载的字段 | Controller 侧消费 |
|---|---|---|---|
| JSON 快照 | `internal/core/control.go:174`（`SELECT p.*,c.secret_ref FROM projects p LEFT JOIN credential_refs c …`） | `ControllerProject.secretRef`，另有 `credentialRefId`、`repositoryCredentialRefId`（`internal/contract/openapi.go:269-270`） | 无 |
| gRPC | `internal/controlgrpc/operations.go:174` → `OperationProject.CredentialRef`（`cloud/proto/ora/cloud/internal/v1/operations.proto:92`，`optional string credential_ref = 4`） | `secret_ref` | 无 |

事实核对（`desktop/` 全仓大小写不敏感检索 `credentialref`／`secretref`，排除 `target/`）：只有两处命中——
`crates/controller-proto/src/gen/…/ora.cloud.internal.v1.rs:1109` 的生成字段 `credential_ref`，以及
`apps/ora-controller/tests/support/workspace_cloud.rs:430` 把它显式设为 `None` 的测试夹具。
**没有任何生产消费方。**

为什么它不是秘密、但仍不该投影：`secret_ref` 按
[Secret Store ADR](20261010-user-managed-git-secret-store.md) D9 不是 bearer token，单靠它取不到秘密；
但它仍是**秘密的唯一定位标识**。把它放进没有消费方的消息里，等于为将来某次"顺手解析一下"留下入口，
而整套设计的前提正是"引用只在 Cloud 与 Store 之间流动"（不变量 8）。

**命名歧义独立于安全风险**：gRPC 的 `OperationProject.CredentialRef` 承载的是 `secret_ref`，而 JSON 的
`ControllerProject.credentialRefId` 才是凭据元数据主键。两个名字都读作"凭据引用"，内容却一个是定位
标识、一个是主键。即使决定保留投影，也必须先消除这个歧义。

**谁可以解析**：只有**凭据执行身份 C** 的部署固定组件（运行身份不是 Agent 的 W，也不是管理身份 M），
凭 Cloud 签发的一次逐次授权（D4）与 Store 侧的 `broker-resolver` 服务身份
（[Secret Store ADR](20261010-user-managed-git-secret-store.md) D3；持有者裁决见
[Node D15](../../node/repository/20261010-controlled-node-git-authentication-isolation.md)）解析单条
引用，且两个条件由 Store
自己校验。**Cloud 的业务进程不解析秘密**——它只签发授权与维护元数据。
**谁不可以**：Cloud 的业务进程与 Controller、Node 的业务逻辑（含 Agent 会话与插件）、Web 前端、任何只
持有 `credential_ref_id` 或 `secret_ref` 的调用方。

**如何防止引用变成无界秘密访问**：解析接口按单条引用授权、无列举能力（Secret Store ADR 不变量 4），
加上本 ADR D2 的单仓库 scope——前者限制"一次能取几条"，后者限制"能对哪个仓库用"。

#### 渐进兼容方案（不得未经版本化评估就删除已发布字段）

1. **旧版本 Controller 的行为**：gRPC 侧 `credential_ref` 是 `optional string`，旧 Controller 反序列化
   得到 `None`，与字段缺失不可区分，且没有生产消费方（唯一引用是把它设为 `None` 的测试夹具）；JSON 侧
   `secretRef` 同为可选字段，同样无消费方。因此**停止填充值对旧 Controller 是零行为变化**。
2. **何时停止填充值**：在落地顺序第 4 步（引用投影与数据兼容收口）**立即停止填充**。它是纯 Cloud 侧
   改动，不需要 Controller 先行升级，也不改变 clone 语义（`CloneExecutionSpec` 本来就没有凭据字段）。
3. **字段保留还是删除**：**先保留声明并置空，不立即删除**。gRPC 字段号 `4` 已发布，删除属于破坏性契约
   变更，必须走契约版本规则（`controller-proto` 与 Cloud 同步升级）后再做。保留期间必须在 proto 注释与
   契约中标注"始终为空、不再填充"，避免读者把它当成可用输入；JSON 的 `secretRef` 同理。
4. **版本兼容与回归测试**：置空后必须有回归证据，断言 Cloud→Controller 的 JSON 快照与 gRPC 快照中
   `secretRef`／`credential_ref` 均为空，且 Controller 在字段为空时行为不变（旧、新 Controller 各跑一次）。
   真正删除字段的那次变更必须另行评估并单独验收。
5. **`credentialRefId` 与 `secret_ref` 的语义区分**：`credentialRefId` 是 `credential_refs.id`，即凭据
   **元数据主键**，是 UUID、不是秘密，投影它不构成泄露；`secret_ref` 是 Secret Store 命名空间内的
   **定位标识**，是秘密的唯一定位符，不应离开 Cloud 与 Store。同一次收口应一并处理 JSON 侧的
   `repositoryCredentialRefId`：按 D7 的裁决它不具权威性，投影它只会让 Controller 看到两列并存，
   因此应与 `secretRef` 一起置空或删除，使 Controller 只看到权威列 `credentialRefId`。

### D10：项目凭据绑定的读写兼容与冲突拒绝

- **权威列**：D7 已裁决 V1 的权威列是 `projects.credential_ref_id`。读写都以它为准；
  `repository_credential_ref_id` 在 V1 只读、不参与回退、不隐式取值。实现阶段必须删除
  `requireRepositoryAccess` 现有"优先 `repositoryCredentialRefId`、为空再回退"的选择
  （`repository_credentials.go:8-12`），使两条读路径与裁决一致。
- **冲突拒绝**：两列同时非空且不一致时，必须**显式拒绝**该次操作并返回可诊断的原因码，不得静默选择
  其一。回退逻辑（`repository_credentials.go:9-12` 现为 `repository_credential_ref_id` 优先、
  再取 `credential_ref_id`）在过渡期是权宜之计，不是长期语义。
- **owner 强制不得降级**：绑定与使用都必须强制 `owner_user_id` 与调用者一致。V1 的权威列
  `credential_ref_id` 的外键 `(id, tenant_id, owner_user_id)` 已在数据库层强制这一点，**不需要**额外的
  应用层拦截；若未来切换到 `repository_credential_ref_id`，必须同时新增等价的 owner 约束。
- **`secret_ref` 的语义**：继续是"秘密的定位引用"，不是密钥值、不是密文、不是 bearer token；其形态
  收口（不透明、唯一、不可枚举）由 [Secret Store ADR](20261010-user-managed-git-secret-store.md) D1 承担。
- **审计列**：`clone_executions.credential_ref_id` / `credential_ref_version` 是长期审计义务（D8）。
- **`deleted_at` 与 `availability`**：撤销经 `availability` 表达；`deleted_at` 在明确其语义之前不承担
  "删除"职责。若将来启用软删，必须同时定义"软删后已发放授权如何处置"与"能否被重新绑定"，不能只加
  一个写入者。
- **schema 演进**：不修改已应用的 migration（`0001`／`0023` 及其校验和）；所有变更经前向迁移，
  且必须可重试、可验证回填。

### D11：开发准入与功能启用是两套门禁，且互不替代

`repository_credential_executor_unavailable` 何时可以移除，与"何时可以开始写代码"是两个不同的问题。
把两者合成一个清单，要么让实现被"还没有端到端证据"卡死，要么让解禁被"ADR 已经写了"提前放行。因此
这里明确拆成两套，**A 组全部满足也不解除任何 fail-closed，B 组的证据也不可能在准入之前产生**。

#### A. 开发准入（进入实际开发所需）

全部是规范与契约条件，**不依赖任何尚未实现的测试结果**。**六项的规范工作已经做完，但结论要随相关
ADR 通过 specs 评审一并生效**；满足准入不代表任何一项功能启用条件（B 段）已经成立：

| # | 条件 | 判定方式 | 状态 |
|---|---|---|---|
| A1 | 相关 ADR 完成 specs 评审并标记 `approved` | 本叶子四份（本文、Secret Store、`0023` 补录、凭据规则例外）与 Node 隔离 ADR 的**方向**已由 `ora-space/circus#29` 确认，但都没有走完 specs 的 PR 评审，现全部保持 `proposed` | **未满足：待评审** |
| A2 | Cloud Token Ingress 的仓库级规范例外生效 | [凭据规则受限例外](20261010-user-managed-git-token-ingress-rule-exception.md) 的例外文本已随 cloud#64 落进 `cloud/AGENTS.md`（英文例外段落作为原条目续段追加，原句逐字保留），但承载它的 ADR 仍是 `proposed`，这段例外的去留取决于 `ora-space/circus#135` 的裁决 | **部分满足：文本已落地，规则尚未定案** |
| A3 | Node D2 amendment 生效 | [最小 clone 执行契约 D2 的凭据来源修订](../../node/repository/20261010-minimal-clone-execution-contract-credential-source-amendment.md) 方向已确认，但仍是 `proposed`；该链阅读入口的注记按评审意见撤回，改由 `20260918` 标题下的 `> 历史决策：` 指向本文 | **未满足：待评审** |
| A4 | Secret Store 后端与密钥责任确定 | Secret Store ADR D1／D2／D8 的后端选型、独立持久卷布局与 KEK 交付介质已裁决 | **已裁决，随该 ADR 的评审一并生效**（具体选型仍是实现期决定） |
| A5 | M／W／C 信任边界确定 | Node 隔离 ADR D1、D15 与本文 D6 已按 `ora-space/circus#29` 的授权确认方向：三者互不相同、`broker-resolver` 由 C 持有而非 M、M 不解析秘密 | **已裁决，随相关 ADR 的评审一并生效** |
| A6 | API、数据兼容与 Broker 授权契约确定 | 凭据元数据 API 形状、权威绑定列与两列冲突拒绝（D7／D10）、`secret_ref` 投影收口（D9）、授权的绑定项与**兑换时原子消费**语义（Secret Store D5／D10）已按上述授权写定 | **已裁决，随相关 ADR 的评审一并生效** |

#### B. 功能启用（解除 fail-closed）

`repository_credential_executor_unavailable` 的无条件拒绝**只能**在下列条件**全部**满足后移除：

1. Node 侧隔离方案与本文（含 [Secret Store ADR](20261010-user-managed-git-secret-store.md)）都已
   `approved`，且 A2／A3 两项必要的规范修订都已生效并落地到各自文件。minicloud 根决策
   **不在**此列：minicloud 已整体退役，该句已无适用对象（见"继承与修改"）。
2. 有**直接**的负面隔离证据：M／W／C 三者互不相同，且 **W 读不到 C 的凭据、授权载体、服务身份材料
   与物化文件**、**W 无法冒用 C 或 M**、**C 不拥有 M 的管理权限**（Node 隔离 ADR D1 的四条），
   Git 执行身份与 Agent 身份之间的文件、环境、文件描述符三条路径分别验证（不是"配置了不同 UID"
   的断言）。**不得**把这一项表述为"M／W／C 互相完全不可读"：M 属于可信计算基，UID 隔离不保证
   M 被攻破后凭据仍安全。同时必须有**正向**证据表明 Agent 的既有合法 IPC 与工作流保持可用；判据是
   "无法取得凭据或提升身份"，**不是**"无法连接 host／guardian socket"（Node 隔离 ADR D16）。
3. 有直接证据证明 Agent 既不能获取 Git Token、也不能获取 Broker 授权载体、也不能获取 C 的
   `broker-resolver` 服务身份材料（证书／私钥），且无法冒充 Broker、无法兑换授权。
4. 有直接证据证明**单次授权防重放**：兑换是一次性原子消费，Store 重启后已消费的授权仍被拒绝，
   响应丢失或超时之后旧授权不可重发（Secret Store D10）。
5. 凭据的**范围、版本、冻结、轮换与撤销**已验证：轮换期间在途执行按原版本结算，冻结与撤销后不再签发
   新授权，`repository_scope`、重定向与代理都不能扩大范围。
6. 有直接证据证明 Git 的 **URL、重定向、代理、credential helper 与 checkout 内 `.git/config`** 都不会
   泄露凭据（含 `constrain()` 未覆盖的键）。
7. 真实 PostgreSQL 集成测试通过（元数据、冻结、scope、版本绑定、审计列写入）。
8. 真实私有 Git 仓库的端到端 Clone 通过。
9. **崩溃、重启、超时与凭据残留清理**有直接证据：Node 崩溃后启动清扫可证明物化目录为空，物化删除
   失败不得使执行被记为成功。

**截至 2026-10-10，只有第 1 项满足，第 2–9 项一项都没有证据**，因此
`repository_credential_executor_unavailable` 继续无条件拒绝。

**不得仅因为 ADR 存在、或测试文件存在，就移除 fail-closed。** 上述每一项都要求可直接指向的证据；
`integration/repository_credentials_test.go` 现有的三个测试只证明 Cloud 的权威判定，其注释本身写明
"A real scoped Git credential provider remains unavailable"——它们不构成第 2–6、8、9 项的证据。

## 不变量

1. Token 明文只存在于 Secret Store 与一次性的受限 Ingress 路径；Cloud 业务数据库、公开响应、业务事件、
   Controller／Node 常规消息、持久日志、审计记录与 Workspace 文件都不含 Token。
2. 秘密不出现在 Git URL、`git` argv、`.git/config`、错误响应或普通 Workspace 卷中。
3. `credential_refs` 是凭据元数据、归属与状态的唯一权威记录；`secret_ref` 不是 PAT 本身，也不是
   bearer token。
4. 第一版凭据的归属是 `personal`，绑定与使用都必须通过 owner 与 tenant 校验；V1 的权威绑定列是
   `projects.credential_ref_id`（D7），其数据库外键强制 owner 一致，切换列不得削弱这一强制。
5. 持有 credential ID 或 `secret_ref` 不构成取得秘密的授权；每次远程 Git 工作都重新签发一次许可。
6. 冻结、撤销与轮换只影响新的远程 Git 工作；已完成的执行、结果结算、restart 与 cleanup 不受影响。
7. `credential_refs.deleted_at` 在明确其语义之前不承担"删除"职责；撤销经 `availability` 表达。
8. `secret_ref` 只在 Cloud 与 Credential Service 之间流动：不进入公开响应、不由客户端提交、不投影给
   Controller（按 D9 停止填充并置空）、不作为 URL 或 query 的一部分。
9. 两个绑定列冲突时必须显式拒绝，不得静默选择其一。
10. 在 Node 侧认证材料对 Agent 进程不可读的隔离获评审通过并验证之前，
    `repository_credential_executor_unavailable` 保持有效；移除它必须满足 D11-B 的全部条件。
11. 公开仓库匿名 Clone、既有 Workspace 恢复与清理行为不因本决策回归。
12. 不修改已应用的 migration；所有 schema 演进是前向、可重试的 SQL。
13. 开发准入（D11-A）与功能启用（D11-B）是两套独立门禁：满足 A 不解除任何 fail-closed，B 的每一项
    都必须有可直接指向的证据；`unknown` 不得在归属未经核实时被自动改写为 `personal`（D2）。

## 为什么不是这些替代方案

| 方案 | 优势 | 否决理由 |
|---|---|---|
| Cloud 业务库直接加密保存 PAT | 少一个组件，事务简单，本地开发最省事 | 密钥必须与密文同库或同进程才自洽，数据库泄露即等于 PAT 泄露；且直接违反 `cloud/AGENTS.md` 的"never accept, persist … repository credentials"，修订代价最大 |
| 独立 Credential Service／Secret Store | 密钥与业务库分离，可独立审计与轮换，能对 Node 做单次授权 | 需要定义授权协议、崩溃恢复与本地开发形态；在 D6 未决时不解决 Agent 可读问题，只是把秘密挪了位置。故本文与 Secret Store ADR 都必须落地 |
| 外部 Secret Manager／Vault 类服务 | 生产形态成熟，轮换与审计现成 | 本地 Docker Compose 无法承载，需要额外凭据引导（"谁来解锁 Vault"）；作为**可替换后端**优于作为第一版前提 |
| 把 PAT 放进 Project 的 URL 或 `.git/config` | 改动最小，Git 原生支持 | 直接违反 D1 的"拒绝携带秘密的 userinfo"，且必然落入 Workspace 卷、日志与 `git` argv |
| 放开 controlled Node，使用部署 credential helper | 复用已验证的 standalone 路径（`repository_policy.rs` 证明 helper 可用） | 该路径的前提是"凭据由部署提供且作用域由部署保证"；换成用户 PAT 后该前提消失，且 helper 与 Agent 同 UID（D6） |
| 经沙盒环境变量注入 PAT | 复用已交付的 `-node-env-file` 机制（该机制已由 M4 实现，其 ADR [沙盒为 Agent 运行提供模型凭据环境变量](../../cluster/sandbox-server/20260928-agent-run-sandbox-environment.md) 状态仍为 `proposed`） | 该机制面向**供 Agent 使用**的模型凭据；Git PAT 必须对 Agent **隐藏**，用途相反。且 Agent 继承 Node 环境（`session/host.rs:60-64` 明确"其余宿主环境仍会到达 Agent"），复用会把秘密直接交给 Agent |
| 用短时上游 token 代替长期 PAT | 泄露窗口短，最接近"短时授权" | GitHub／GitLab 不提供"由 Ora 按次签发、绑定单仓库、数秒有效"的 token；GitHub App installation token 属另一条产品线，不在第一版范围 |
| 改用 `projects.repository_credential_ref_id` 作为 V1 权威列 | 名字更贴近"仓库凭据"，约束也更宽松，将来容纳 `team` 凭据更容易 | 它没有任何写入者，外键不强制 owner（需另行补约束或接受应用层强制），且会把改动扩到 schema 与回填；V1 以"已在写入 + 数据库层已强制 owner + 零 schema 变更"的 `credential_ref_id` 为准（D7） |
| 继续把 `secret_ref` 投影给 Controller，只是"暂时没人用" | 零改动，保持契约现状 | 没有消费方的投影只增加暴露面与"顺手解析"的诱惑，且违反不变量 8；停止填充是前向的且不影响 clone 语义（D9） |
| 只加一个写入者来启用 `deleted_at` 软删 | 让"删除"看起来可用 | 未定义软删后已发放授权如何处置、能否被重新绑定；撤销已有 `availability` 承担，不需要第二套语义 |
| 首版只支持公开仓库，不做凭据管理 | 零新增攻击面 | 用户明确要求私有 HTTPS Clone；且 `credential_refs` 与 `requireRepositoryAccess` 的存在说明方向已定，只是通道缺失 |

## 失败与恢复语义

| 失败点 | 对外结果 | 持久事实与恢复 |
|---|---|---|
| Ingress 写入失败（Store 侧） | 保存失败，不返回 credential ID | 无元数据行；可安全重试 |
| Store 成功、元数据事务失败 | 保存失败 | 产生孤儿秘密；由 Store 侧补偿清理，不得对用户报成功 |
| 元数据成功、秘密不可读（Store 不可用） | 绑定可读，Clone 明确失败，不降级匿名 | 引用保留；Store 恢复后重试是新执行 |
| Ingress 请求体进入日志或错误上下文 | 视为实现缺陷 | 必须由测试与检索证据证明不发生（[凭据规则受限例外](20261010-user-managed-git-token-ingress-rule-exception.md) 条件 4；证据归属见 D11-B） |
| Clone dispatch 时凭据已冻结／已撤销 | `409 repository_credential_frozen` | 不派发；已发放的许可与在途执行按 D5 处置 |
| 凭据版本在授权与执行之间变化 | 明确失败，不静默复用旧版本 | 重试使用新版本重新授权 |
| 两个绑定列同时非空且不一致 | 显式拒绝，返回可诊断原因码 | 不静默选择其一（不变量 9） |
| Node 认证通道不可用（D6 未决） | `409 repository_credential_executor_unavailable` | 保持 fail-closed，不回退匿名或静态个人凭据 |
| 用户在 Git 托管平台吊销了 PAT，但 Ora 侧未撤销 | Clone 失败，`credential_refs` 状态不自动变化 | Ora 无法感知上游吊销；不得据此推断凭据可用或不可用 |
| `secret_ref` 仍出现在 Controller 投影中 | 不构成功能失败 | 记为待整改项：按 D9 停止填充并置空；字段的删除另行走契约版本评估 |

## 本决策未解决的问题

- **Node 侧隔离机制**（D6）：git 与 Agent 的身份分离、受信任 helper、受限 IPC 的具体选择，
  以及 Git HTTP 重定向、代理、子进程环境与配置继承的边界验证。这是本 ADR 落地的阻塞项。
- **领域归属已裁决，README 措辞是否同步待定**：本叶子的三份凭据决策保留在 `identity-access`（人工裁决，
  见"继承与修改"）。但 `decisions/cloud/README.md` 仍把"凭据引用"划归 `project`，与该裁决的措辞不一致；
  是否调整 README 的领域描述、以及跨叶子引用是否需要标注，留待人工裁决。
- **`0023` 的持久语义已补录为独立决策（待评审）**：`attribution`／`availability`／`repository_scope`／
  `capabilities` 与两个冻结触发器的规则原先只存在于 `cloud/docs/runtime-control.md` 与迁移正文中，现已
  由 [仓库凭据归属连续性](20261010-repository-credential-ownership-continuity.md)（proposed）承载。
  该补录仍需评审后才能推进状态；本文的 D2／D5 与它的既有判定必须保持相容。
- **Secret Store 的产品形态**：本地 Compose 与未来生产如何替换后端；密钥由谁持有、如何轮换与备份；
  服务重启与 Store 不可用时的确定失败语义。已由
  [Secret Store ADR](20261010-user-managed-git-secret-store.md) 承接，本文不再展开。
- **撤销的追溯力**：已发放授权是否必须强制终止；上游 PAT 的有效期与 Ora 撤销语义的差距如何在产品上
  向用户解释。
- **第一版 Provider 范围**：仅 GitHub 还是通用 HTTPS Git；精确仓库身份匹配与重定向策略。
- **权威列的 owner 约束与 team 凭据**：`credential_ref_id` 的外键把 `owner_user_id` 写进约束，因此结构上
  无法绑定 owner 与 Project owner 不同的凭据。V1 只支持 `personal`（D2），但支持 `attribution='team'`
  共享凭据时必须重新裁决（可能需要切换到 `repository_credential_ref_id` 并补齐等价的 owner 约束）。
- **团队凭据**：`attribution='team'` 目前没有任何产品写入路径，其授权与成员离开规则另立决策。
- **`cloud/AGENTS.md` 例外条款的措辞终稿——方向已确认，去留待裁决**：项目负责人确认了 D3 的文本方向，
  该文本已作为原条目续段写入 `cloud/AGENTS.md`；仓库文件为英文，只有英文段落进入规则文件。这段例外
  是否保留，取决于 `ora-space/circus#135` 的裁决。

### `0023` 持久语义的最小补录范围（已按此范围起草）

补录一份 Cloud ADR 承载"仓库凭据归属连续性"这一件事即可，**不把 Runtime Control 的历史设计一并纳入**。
草案见 [仓库凭据归属连续性](20261010-repository-credential-ownership-continuity.md)（proposed）。范围仅限：

1. `credential_refs` 的 `attribution`／`attribution_basis`／`availability`／`unavailable_reason`／
   `repository_scope`／`capabilities` 六列的语义与允许取值。
2. 两个冻结触发器的触发条件与不可逆性（成员／用户停用 → 原子冻结 `personal`／`unknown`；`team` 不因
   旧 owner 离开冻结；重新加入不自动解冻）。
3. 恢复扫描的判定与"重新加入不自动解冻"的边界。
4. `projects.repository_credential_ref_id` 与 `clone_executions.credential_ref_id`／`credential_ref_version`
   两处外键与审计列的引入意图（D7 已裁决 V1 的权威列是 `credential_ref_id`，补录只需记录另一列的引入
   意图与"保留不删、V1 不参与回退"的现状）。
5. 明确**不在**范围内的内容：Runtime Control 的 runtime／force stop／creator 相关表与语义、`0023` 之外
   的迁移、以及任何实现细节。

## 落地顺序

每一步都可独立验收。第 1 步即 **D11-A 开发准入**，它不要求任何实现或测试证据；第 6 步的解除
fail-closed 属于 **D11-B 功能启用**，两者不可互相替代。**D11-A 的文本已就绪，但结论尚未生效**——
相关 ADR 保持 `proposed`，等 specs 评审通过；**没有任何一步已经开始，D11-B 的九项也一项都没有证据。**

1. **规范与补录（= D11-A 开发准入）——文本已就绪，等 specs 评审**：领域归属（保留在 `identity-access`）
   与权威列（D7）两项裁决不变。六份决策——本文、[Secret Store ADR](20261010-user-managed-git-secret-store.md)、
   [仓库凭据归属连续性](20261010-repository-credential-ownership-continuity.md)（`0023` 补录）、
   [凭据规则受限例外](20261010-user-managed-git-token-ingress-rule-exception.md)（`cloud/AGENTS.md`）、
   [最小 clone 执行契约 D2 的凭据来源修订](../../node/repository/20261010-minimal-clone-execution-contract-credential-source-amendment.md)
   与 [受控 Node 的仓库凭据执行使用独立 OS 身份](../../node/repository/20261010-controlled-node-git-authentication-isolation.md)
   ——方向已由 `ora-space/circus#29` 确认，但**都还没有走完 specs 的 PR 评审，现全部保持 `proposed`**；
   两项 amendment 的文本已落到目标文件（`cloud/AGENTS.md` 追加例外段落；`20260918` 标题下的
   `> 历史决策：` 指向本文），其去留随评审结论定。本步不产生任何代码或测试证据，**因此它的完成不构成
   任何实现或解禁的依据**；A4–A6 三项契约（Secret Store 后端与密钥责任、M／W／C 信任边界、API 与授权
   契约）已按上述授权写定。
2. **Cloud 侧元数据与授权（不触碰秘密）**：凭据元数据 CRUD（不含 Token 读取）、scope 规范化、
   D8 的审计字段写入、D4 的逐次授权。完成条件：真实 PostgreSQL 覆盖 owner／tenant 隔离、冻结、
   不自动解冻、scope 不匹配、版本绑定、重放与失败回滚。此时 `repository_credential_executor_unavailable`
   **仍然有效**，因为秘密通道尚未存在。
3. **Secret Store 与 Ingress**：按选定的形态实现加密持久化、写入一致性与孤儿清理（详见
   [Secret Store ADR](20261010-user-managed-git-secret-store.md) 的落地顺序）。完成条件：
   泄露检查（日志、错误、审计、Workspace 卷均无 Token）、Store 不可用时的确定失败。
4. **引用投影与数据兼容收口**：按 D9 停止填充 `secret_ref`（保留声明并置空，标注"始终为空"），并同步
   处理 JSON 侧 `repositoryCredentialRefId`；按 D7／D10 把 `requireRepositoryAccess` 改为以
   `credential_ref_id` 为准、删除回退，并加入两列冲突拒绝。完成条件：Cloud→Controller 的 JSON 与 gRPC
   快照中 `secretRef`／`credential_ref` 均为空且有回归证据（旧、新 Controller 各一次）；Controller 只
   看到权威列 `credentialRefId`；两列冲突有直接测试证据。
5. **Node 认证通道**：按 D6 定下的隔离方案实现，并同步修订 Node D2 与相关既有契约。
   完成条件：受控私有 Git fixture 的真实端到端 Clone，且证明普通 Agent 进程无法读取认证材料。
   注意 Node 侧的第一阶段只建立身份边界与启动闸门（
   [受控 Node 的仓库凭据执行使用独立 OS 身份](../../node/repository/20261010-controlled-node-git-authentication-isolation.md)
   D17）——它不物化凭据、不接入 Broker、不改变无凭据 Git 的执行身份，因此**不产生本步所需的证据，
   也不解除 fail-closed**。
6. **解除 fail-closed（= D11-B 功能启用）**：在 D11-B 的全部条件都满足后，移除
   `repository_credential_executor_unavailable` 的无条件拒绝，保留其前置校验。

保持不变的行为：公开仓库匿名 Clone、既有 Workspace 的恢复与清理、冻结与归属连续性语义、
所有已应用的 migration 与已发布的公开 API 形状。
