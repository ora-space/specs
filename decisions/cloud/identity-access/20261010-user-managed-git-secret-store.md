---
status: approved
---

# 用户 Git Token 由独立可替换的 Credential Service 承载：Cloud 只经受限 Ingress 写入并按 clone 单次签发 Broker 授权，`secret_ref` 本身不构成取用秘密的能力

[用户自助 HTTPS Git 凭据](20261010-user-managed-git-credentials.md) 把秘密的持久化交给"独立 Secret Store"
之后，本文把这一句展开为可评审的契约：Store 的信任边界、Token 的加密与密钥归属、Cloud 与它的服务级
认证、跨两个存储的创建／轮换／撤销一致性、后端不可用时的失败语义、审计与脱敏、备份与密钥轮换，以及
`secret_ref` 为什么不是 bearer token。Broker 授权是 Cloud 与 Node 之间的交付载体：**签发**规则在本文，
**消费**规则在 [受控 Node 的仓库凭据执行使用独立 OS 身份](../../node/repository/20261010-controlled-node-git-authentication-isolation.md)
（`approved`）。

本文于 2026-10-10 经项目负责人批准，状态为 `approved`；**批准确认的是决策内容与其安全边界，既不表示
已经实现，也不解除任何 fail-closed 门禁**。`requireRepositoryAccess` 的
`repository_credential_executor_unavailable` 拒绝路径继续有效；本文不授权实现变更，不改写任何已应用的
migration，不修改任何 approved ADR 的状态或决策。Store、Ingress 与 Broker 都尚未实现，本文落地顺序的
第 2 步及之后都还没有证据。

涉及模块：Cloud（`internal/core` 的凭据元数据与授权、`internal/core/migrations`）、Secret Store／
Credential Service（新组件，尚未实现）、Credential Ingress（新组件，尚未实现）、Web 设置页、desktop
Controller 与 Node、cluster 部署（Compose）。兼容性边界：`credential_refs` 表、已应用的 `0001`／`0023`
迁移、已发布的公开 API 形状，以及 `projects.credential_ref_id` 的既有语义都按硬边界处理，只做前向兼容
演进。本 ADR **不承担** SSH 凭据、GitHub App、fetch／pull／push、团队共享凭据、ownership transfer 与
上游平台令牌自动吊销的迁移义务。

## 继承与修改

| 前序约定 | 出处 | 本文处理 |
|---|---|---|
| 凭据的归属、可用性与冻结语义已由 `0023` 交付；冻结只影响新的远程 Git 工作，且不因撤销而修改 `secret_ref` | [仓库凭据归属连续性](20261010-repository-credential-ownership-continuity.md)（approved）D1–D6 | 继承，不变；本文据此把"撤销的权威"固定在 Cloud 元数据上，并要求 Store 侧的删除失败不得使凭据重新可用（D4） |
| Cloud 只持有凭据引用、不持有密钥值；`credential_refs` 是唯一业务主表 | [用户自助 HTTPS Git 凭据](20261010-user-managed-git-credentials.md)（approved）D1 | 继承，不变；本文细化 `secret_ref` 的定位语义与解析权归属 |
| Token 只经受限 Secret Ingress 进入 Secret Store；Cloud 业务库、公开响应、业务事件、Controller／Node 常规消息、持久日志、审计记录与 Workspace 文件都不得持久化 Token | 同上 D3 | 继承并展开：本文定义 Store 的信任边界、加密、服务认证与跨存储一致性 |
| Clone 许可逐次签发，绑定 tenant／owner／project／规范化仓库／credential id／version／workspace／execution，持有 credential ID 不等于取得秘密 | 同上 D4 | 继承；本文把"许可"具体化为 Broker 授权的载体、单次使用与 fencing 规则 |
| 撤销经 `availability` 表达；`credential_refs.deleted_at` 在明确语义前不承担删除职责 | 同上 D5、不变量 7 | 继承，不变；本文据此把"撤销的权威"固定在 Cloud 元数据上，使 Store 备份恢复不能复活凭据 |
| Token 明文只存在于 Secret Store 与一次性受限 Ingress 路径；`secret_ref` 不是 PAT 本身 | 同上 不变量 1、3 | 继承，不变 |
| Cloud 只提供 credential reference，由基础设施注入，Cloud 不保存密钥值 | [cloud/operation 根决策](../operation/0-workspace-runtime-follows-desktop-node.md)（implemented）D2 | 继承，不变；本文把"基础设施"具体化为可替换的 Credential Service，并规定它的服务边界 |
| 凭据不经业务消息、命令文本、日志或结果传播；认证不依赖人机交互 | [最小 clone 执行契约](../../node/repository/20260918-minimal-clone-execution-contract.md)（approved）D2、不变量 4 | 继承，不变；本文把禁止面扩展到 Store 的审计、备份与 Broker 授权载体 |
| bearer 凭据只在内存，不进 ledger、日志或 `Debug`；重启后重新申请 | `apps/ora-node/src/revision/grants.rs:3`（upload grant 的既有形态） | 继承其形态；Broker 授权沿用同一形态，因此授权本身按秘密处理 |
| Credentials remain infrastructure references. Never accept, persist, log, return, or commit raw … repository credentials | `cloud/AGENTS.md`（仓库级硬约束，119-123 行） | **修改**：范围受限的正式 amendment 由 [凭据规则受限例外](20261010-user-managed-git-token-ingress-rule-exception.md)（approved）承载正文，并已于 2026-10-10 落地到 `cloud/AGENTS.md`；本文只承担"Cloud 不持久化 Token"这一侧的约束，并复用该例外的条件 3、4 作为 Ingress 与日志边界 |

## 背景与现状

事实（以现有代码为准）：

- **今天没有任何 Store。** 仓库内不存在 Secret Store／Credential Service：`cloud/internal`、`cloud/cmd`
  中 `secretstore`／`credentialservice` 零命中，`compose.yaml` 也没有对应服务。`secret_ref` 因此不是
  "某个存储中的键"，只是一个**部署方手填的字符串**。
- **唯一的写入路径是部署 CLI。** `ConfigureCredential`（`internal/core/store.go:550-558`）由
  `cloudctl -command credential-ref -tenant -owner -secret`（`cmd/cloudctl/main.go:55-56`）调用，把调用者
  提供的任意非空、长度 ≤1024 的字符串直接写进 `credential_refs.secret_ref`。它的 doc comment 写明
  *"deliberately a deployment-only management path, never a public secret API"*。**该字符串没有唯一性、
  没有不可枚举性、也没有任何存储与之对应**——这是 V1 必须收口的地方，否则"不透明标识"这一前提不成立。
- **没有凭据 CRUD API，也没有任何秘密解析能力。** 公开 `Project` schema 不含 `secretRef`；Cloud 侧没有
  任何向外部服务解析秘密的代码路径。
- **引用已经被投影给 Controller，而 Controller 完全不用它。** `ControllerProject` 把 `secretRef` 放进
  可选集（`internal/contract/openapi.go:269`），`controlgrpc/operations.go:174` 又把它映射到 gRPC 的
  `OperationProject.CredentialRef`（字段名读起来像"凭据 id"，实际承载的是 `secret_ref`）；而
  `apps/ora-controller/src/` 中 `credential_ref` 零命中。
- **仓库已有"管理面初始化秘密、使用方按文件读取"的先例。** `objectstore-credentials`
  （`compose.yaml:19-22`）由管理面服务把对象存储的 access／secret key 写入共享卷，使用方以
  `RUSTFS_ACCESS_KEY_FILE`／`RUSTFS_SECRET_KEY_FILE` 读取（`compose.yaml:24-31`）。本地 Compose 形态的
  Store 与 KEK 交付可以沿用同一形状，不必新造机制。
- **Node 侧已经存在"bearer 凭据只在内存"的实现先例。** upload grant 的注释（`revision/grants.rs:3-6`）
  写明它"never enters the ledger, a log line or a `Debug` print"，并且 `Drop` 时从内存移除
  （`grants.rs:190-195`）。Broker 授权沿用该形态即可复用既有审查结论。
- **`credential_refs` 的冻结与归属连续性语义已经实现并有真实 PostgreSQL 证据**（`0023` 迁移的触发器与
  恢复扫描），本文不新增第二套冻结规则，只在其上增加"秘密侧"的一致性要求。

结论：本文要补的不是"再建一张表"，而是**秘密的存储载体、服务间认证与跨存储一致性**这三段；它们今天
都不存在，而 `secret_ref` 的现有写入方式（部署方手填任意字符串）不足以支撑"引用不是能力"这一前提。

## 决策

### D1：Secret Store 是独立可替换的 Credential Service，`secret_ref` 是它命名空间内的不透明标识

- Cloud 不实现 Token 的存储。`credential_refs.secret_ref` 是 Cloud 侧对 Store 中一条秘密的**定位标识**：
  不是秘密、不是密文、也不是可兑换的凭据。
- Store 以独立进程、**独立持久卷**与独立数据存储部署，与 Cloud 业务库不共享数据库实例、不共享连接、
  不共享凭据、不共享卷。V1 允许先用独立持久卷上的内置加密存储，它的备份、快照与保留策略都与 Cloud
  业务库彼此独立（D8）。
- **后端可替换是部署属性，不是协议属性**：内置加密存储与外部 Secret Manager 都只暴露同一组服务接口，
  Cloud 与 Node 不依赖某一后端的内部形态。
- `secret_ref` 的形态要求：高熵、不透明、非用户可读可猜，且全局唯一。**生成权已裁决归 Secret Store**
  （人工裁决）：由 Store 在写入成功时生成并返回，Cloud 只负责持久化它。**浏览器、终端用户与部署方都
  不得自行指定 `secret_ref`**：客户端提交的引用一律不接受，`ConfigureCredential` 当前"接受任意 ≤1024
  字符串"的形态在 V1 必须收口——一个可由部署方随意填写的字符串无法保证不重复、不可枚举。
- 因此创建是"Cloud 决定身份、Store 决定引用"的两段式：Cloud 先生成 `credential_refs.id` 并把它作为
  幂等键（D4），Store 生成 `secret_ref` 并在写入响应中返回，Cloud 在同一个元数据事务里持久化该引用。
  这样既保证重试不产生第二条秘密，也保证引用不可被外部预先指定或猜测。
- 本地 Compose 形态复用 `objectstore-credentials` 的形状：由管理面初始化服务生成并写入共享卷，使用方
  按 `*_FILE` 读取；不新造交付机制。

为什么要求独立进程与独立存储：把秘密留在 Cloud 业务库内，密钥就必须与密文同库或同进程才自洽，于是
"数据库导出"等价于"Token 泄露"；而分离之后，单次泄露面从"整个业务库"收缩为"某一层"。代价是多一个
组件、多一次网络往返、多一组失败模式——这些正是 D4／D6 要显式处理的。

### D2：Token 以 AEAD 加密保存，密钥不与被加密数据同库

- 每条秘密一个数据密钥（DEK），密文与 DEK 的封装（wrapped DEK）一起存放；KEK 只用于封装 DEK，不直接
  加密 Token。
- **KEK 的归属是部署，不是 Cloud、也不是 Store 的运行期。** KEK **不进入** Cloud 业务库、**不进入**
  Store 的数据表、**不进入**任何备份；由部署在启动时经文件或卷交付（D1 的 `*_FILE` 形态）。Store
  **不生成** KEK，也**不持久化** KEK 本身——它持久化的只有密文、wrapped DEK 与密钥标识。KEK 的轮换与
  备份／恢复由 `kek-operator` 身份执行（D3），不由解析或写入路径执行。
- 因此"拿到 Cloud 数据库导出"或"拿到 Store 数据目录"都不足以解密；只有同时拿到 KEK 才构成泄露。
- 封装中必须带密钥标识，使 KEK 轮换期间新旧封装都能被解开（D8）。

明确否决"用 Cloud 主密钥直接加密 Token"：那会让 Cloud 进程本身成为解密能力，与 Cloud ADR 的"Cloud
只保管引用与授权、不持久化 Token"直接冲突——即使密文不在业务库里，解密能力仍在业务进程内。

### D3：Store 只接受服务身份，且三个身份按能力分权；解析必须同时验证服务身份与授权

Store 只接受服务身份（mTLS 加固定服务证书，与 Node 侧 `securedContainerSpec` 的
`mutual_tls_websocket` 同形），**不接受**终端用户令牌，也不接受来自浏览器的请求。服务身份固定为三个，
能力互不重叠：

| 服务身份 | 持有方 | 允许的能力 | 明确没有的能力 |
|---|---|---|---|
| `ingress-writer` | Cloud 的 Credential Ingress 路径 | 写入一条新秘密、轮换、撤销（按 `credential_refs.id`） | 解析、列举 |
| `broker-resolver` | **凭据执行身份 C**（人工裁决，见 [Node D15](../../node/repository/20261010-controlled-node-git-authentication-isolation.md)）：解析客户端由部署固定、以 C 的身份运行，运行身份**不是** Agent 的 W，**也不是**管理身份 M。C 因此需要自己的服务证书与到 Store 的网络路径，不复用 M 的控制通道 | 按一次授权解析**一条** `secret_ref` | 列举、遍历、按 host 批量解析、写入 |
| `kek-operator` | 部署操作路径（不在 Cloud 与 Node 的运行路径上） | KEK 轮换、备份／恢复相关操作 | 解析单条秘密（不构成运行期取用能力） |

**解析是双因子校验，由 Store 自己完成，不依赖调用方已经校验过授权**：

1. **服务身份**：mTLS 对端必须是 `broker-resolver`；其他身份调用解析接口一律拒绝。
2. **逐次授权**：调用方必须同时出示一个由 Cloud 签发、绑定 tenant／owner／project／workspace／
   clone execution／credential id／credential version／规范化 repository identity／`operation=clone`、
   带到期与单次使用约束、并绑定接收 Node 实例与运行代次的授权（D5）。Store 校验其绑定与实际请求一致、
   未过期、**且尚未被消费**；校验通过后，Store 在**同一个原子步骤**里先持久化"该授权已被消费"这一事实，
   成功之后才把秘密交出去（D10）。顺序不可颠倒：先交付、后记录会让"响应在途中丢失"变成一次可重放的
   成功。

两个条件缺一不可：只有服务身份而没有授权，解析被拒绝；只有授权而没有服务身份，解析同样被拒绝。

**授权在签发时是"未使用"状态**：Cloud 签发时不做任何"已用"标记，Store 也不在写入、缓存或收到授权时
预占它。消费只发生在兑换的那一刻（D10）。

**写入路径只需要服务身份，不需要授权**——理由必须写清：用户级别的授权在 Cloud 的 Ingress 处完成
（请求已认证、归属为调用者自身、凭据归属校验通过），Store **不重复验证用户身份**，它信任
`ingress-writer` 这一服务身份代表 Cloud 已完成的判定。因此写入路径的授权边界是"Cloud 的 Ingress 逻辑
＋ `ingress-writer` 身份"，而解析路径的授权边界是"`broker-resolver` 身份 ＋ 单次授权"。两者不共享
任何一组凭据，单点被攻破不等于全量泄露。

`kek-operator` 单独存在的理由：KEK 一旦可被解析路径使用，`broker-resolver` 被攻破就等于"全部秘密可
解密"，而不仅仅是"能多解析几条"。把批量性的密钥操作与单条解析分开，是为了让单点泄露的影响面保持
在"一条秘密"这一量级。

### D4：创建、轮换与撤销是跨两个存储的操作，必须有幂等键与显式补偿

- **创建**：Cloud 先生成 `credential_refs.id`，再以该 id 作为幂等键向 Store 写入；**Store 生成
  `secret_ref` 并在写入响应中返回**（D1 的裁决——引用不可由外部预先指定），Cloud 在同一个元数据事务里
  持久化它。元数据事务在秘密写入成功后才提交；向用户报告保存成功必须同时满足两者。重试携带同一个
  `credential_refs.id` 时必须返回同一条秘密与其原 `secret_ref`，不得生成第二条。
- **孤儿与悬空都必须可识别、可对账**：孤儿（秘密存在、元数据不存在）由 Store 侧按"未被任何元数据引用
  且超过 TTL"清扫；悬空（元数据存在、秘密不可读）在授权时**明确失败**，不降级匿名。
- **轮换**：先写新版本、后推进元数据指针。旧版本在指针推进后仍可解析一个有界的宽限期，让在途执行按原
  版本结算；宽限期结束后删除。**不得**先删旧版本再写新版本。
- **撤销**：先改 Cloud 侧 `availability`（权威），再尽力删除 Store 中的秘密。Store 删除失败**不得**使
  凭据重新可用。这把"撤销的权威"固定在 Cloud 元数据上，因此从 Store 的旧备份恢复不会让已撤销凭据复活
  （D8）。
- 这三条是跨两个存储的分布式一致性问题，不能用"重试"掩盖；每条都必须有幂等键与可对账的中间状态。

### D5：Broker 授权由 Cloud 签发、由凭据执行侧消费，是短时、单次、绑定具体执行的载体，且它自身是秘密

- **签发方是 Cloud**：绑定内容、到期判定与签发都在 Cloud 侧完成。**消费方是凭据执行侧**的
  `broker-resolver`（D3），由它在解析时向 Store 出示。Cloud **不消费自己签发的授权**，也不解析秘密。
- 授权绑定 tenant／owner／project／workspace／clone execution／credential id／credential version／
  规范化 repository identity／`operation=clone`，带到期时间与单次使用约束，并绑定接收它的 Node 实例
  与运行代次（fencing）。
- 载体形态沿用既有先例：`revision/grants.rs` 的 upload grant 是"只在内存、不进 ledger／日志／`Debug`、
  重启后重新申请"的 bearer 凭据。Broker 授权沿用同一形态——**授权是 bearer，因此它自己就是秘密**，
  不得写入 Node 数据库、process journal、日志或事件。
- 授权有效期不长于一次 clone 的命令超时。**授权短时不等于 Token 短时**：Token 的有效期由上游托管平台
  决定，Ora 的撤销不等于自动吊销上游 PAT。
- **单次使用的强制点在 Store，且消费发生在兑换时**：授权在**签发时是未使用状态**——Cloud 签发它时
  不做任何"已用"标记，Store 也不在写入或缓存时预占它。只有在一次解析请求真正到来、双因子校验通过之后，
  Store 才**原子地**消费它（D10）。因此同一授权最多成功兑换一次，不能跨执行、跨 Node 实例、跨运行
  代次复用。消费方同样不得重放（失败与重试必须申请新授权），但**强制点在 Store**——只靠消费方自律
  无法察觉重放。
- **未拿到秘密的失败一律换新授权**：超时、连接中断、响应丢失、Git 失败、`Interrupted`、Node 崩溃与
  重试都不得重发同一个授权；新授权必须重新经过 Cloud 的归属、冻结、版本与执行状态校验（D10）。

### D6：后端不可用、不可核实或服务身份未建立时明确失败

- Store 不可用、服务认证失败、密文完整性校验失败、KEK 缺失或授权校验失败，一律以明确失败结束；不回退
  匿名 clone、不回退静态个人凭据、不把"引用存在"当作"秘密可用"。
- **启动闸门**：Cloud 在配置了凭据能力却无法建立 Store 服务身份或无法取得 KEK 时，必须拒绝启动该能力，
  而不是在运行期逐次失败；这与 `main.rs` 已有的"网络管理必须配置独立工作负载身份"式启动拒绝同构。
- 这与 Cloud 侧既有的 fail-closed 行为同向（`internal/core/repository_credentials.go:25` 无条件拒绝）。

### D7：审计与脱敏有固定边界

- 审计记录：谁、何时、对哪个 tenant／owner／project／credential id／version 执行了创建／轮换／撤销／
  解析，结果成功或失败，以及失败原因码。
- 审计**不记录**：Token、密文、wrapped DEK、KEK、Broker 授权载体，以及任何可复原这些值的片段。
- 日志与错误响应同样适用；Git 的错误输出也不得作为秘密载体（与 `20260918` 不变量 4 一致）。
- 失败原因码必须区分"未授权""已撤销""版本不匹配""Store 不可用"，但**不泄露秘密内容或长度**。

### D8：备份、恢复与密钥轮换

- 备份包含密文与封装元数据，**不包含** KEK；没有 KEK 的备份不可解密。恢复后必须失败关闭，而不是降级
  为明文或空 Store。
- **KEK 轮换**只重新封装 DEK，不重新加密密文；**DEK 轮换**重新加密密文。两者都必须在线、幂等，且失败
  可重入。
- 恢复后撤销的权威仍来自 Cloud 元数据（D4），因此恢复旧备份不会让已撤销凭据重新可用。
- 密钥丢失是不可恢复事件：Store 必须让"无 KEK"表现为**启动拒绝**，而不是表现为一个空 Store——后者会
  把"无法解密"误报为"从未存在"。

### D9：`secret_ref` 不构成取用秘密的能力

- 解析一条 `secret_ref` 需要**同时**具备 Cloud 签发的逐次授权（D5）与 Store 侧的 `broker-resolver`
  服务身份（D3），且两者都由 Store 自己校验。因此解析发生在**凭据执行侧**，而不是 Cloud 的业务进程里
  ——Cloud 只签发授权与元数据，不解析秘密。持有引用字符串本身、只持有服务身份、或只持有授权，都得不到
  任何秘密。
- 因此 `secret_ref` 不得出现在公开 API 响应、不得由客户端提交、不得作为 URL 或 query 的一部分；它只能
  存在于 Cloud 与 Store 之间的服务调用与 Cloud 的业务库列中。
- 引用不得成为"无界秘密访问"的入口：解析接口按单条引用授权，**没有列举接口**，也没有"按 host 批量
  解析"的形态。这与 Cloud ADR 的单仓库 scope 互为纵深：scope 限制"能对哪个仓库用"，引用权限限制
  "一次能取几条"。
- 现存的 Controller 投影问题（`secretRef` 出现在 Cloud→Controller 的 JSON 快照与 gRPC 投影中，而
  Controller 零使用）由 [用户自助 HTTPS Git 凭据](20261010-user-managed-git-credentials.md) D9 的渐进
  兼容方案处理：停止填充、保留已发布字段声明并置空，字段的删除另行走契约版本评估。本文只固定"引用不是
  能力"这一条，并要求任何投影都不得把它变成可兑换的凭据。

### D10：单次授权在兑换时原子消费，消费记录持久，重放一律拒绝

授权的生命周期必须分成三个**互不重叠**的阶段。任何文档、注释、提交说明或实现描述都不得把它们的顺序
写反，尤其不得出现"签发时即标记已用"这类描述：

| 阶段 | 发生在哪 | 状态与动作 |
|---|---|---|
| 签发 | Cloud | **未使用**。Cloud 只做绑定与签发，不做任何"已用"标记，也不预留消费记录 |
| 兑换 | Store（一次解析请求内） | 校验服务身份与授权绑定后**原子消费**：先持久化"已消费"事实，成功之后才交付秘密 |
| 交付后 | Store → C | 秘密只交给出示该授权的 `broker-resolver`；该授权自此永久不可再用 |

- **为什么必须否定"签发时即标记已用"**：那会让一条刚签发、尚未兑换的授权立即失效，等于把授权做成
  一次性的空凭据；更糟的是它会把真正的强制点从兑换挪到签发，使"同一授权被出示两次"这件事不再有
  可判定的结果。签发与兑换之间必然存在时间差（交付、fencing、排队都可能占掉它），这段时间里授权
  必须是可用的。
- **消费记录由 Store 自己持久化**，落在 Store 的独立数据存储里（D1），**不落在 Cloud 业务库、不落在
  Node 数据库或 process journal**。理由是：Cloud 不是消费方（D5），Node 侧的持久介质按不变量 7 本就
  不得承载授权载体；只有 Store 能同时看到"这条授权被出示过"与"它是否已经兑换"。
- **原子性要求**：校验通过 → 写入消费记录 → 交付秘密。前两步必须在同一次存储事务内完成，且消费记录
  的写入**必须先于**秘密离开 Store。若 Store 在写入消费记录之后、交付之前崩溃，结果必须是"该授权已被
  消费、秘密未交付"（可由消费方申请新授权恢复），**不得**是"秘密已交付、消费记录缺失"——后者会让
  一次崩溃本身构成重放窗口。
- **重放防护**：消费记录是**持久**的，因此
  - Store 重启、滚动升级或从崩溃中恢复后，已消费的授权**不得**被重新接受；
  - 同一授权在并发请求下最多有一个成功，其余以"已被消费"拒绝；
  - 跨执行、跨 Node 实例、跨运行代次的复用，以及把授权搬到另一个 host 使用，一并以"绑定不符"或
    "已被消费"拒绝。
- **失败恢复**：授权不可重放，因此**任何**未拿到秘密的失败路径都只能申请**新授权**，包括请求超时、
  连接中断、响应在途中丢失、Git 进程失败、`Interrupted`、Node 崩溃与重试。响应丢失时 Store 可能已经
  消费了该授权——这正是不得重发的理由，而不是重发的理由。
- **新授权必须重新经过 Cloud 的全部校验**：Cloud 不得因为"刚签发过"而跳过或复用判定。每一次签发都
  重新校验调用方的 tenant／owner 归属、成员与用户是否 active、凭据 `availability` 与
  `repository_scope`（[凭据归属连续性](20261010-repository-credential-ownership-continuity.md) D3–D5）、
  credential version 与执行状态（D5 的绑定项），并据此决定签发或拒绝；冻结、停用或版本推进之后一律
  不再签发。已签发但未兑换的授权不因冻结被"撤回"，它按到期失效（消费侧规则见
  [Node D5](../../node/repository/20261010-controlled-node-git-authentication-isolation.md)）。
- **消费记录的最小内容**：授权标识、消费时刻、出示它的 Node 实例与运行代次、结果（已交付／已消费未
  交付）。它**不记录**秘密本身，也不记录任何可复原秘密的片段（D7）。

## 不变量

1. Token 明文只存在于 Store 的加密存储与一次受限写入路径；Cloud 业务库、公开响应、业务事件、
   Controller／Node 常规消息、持久日志、审计记录与 Workspace 文件都不含 Token。
2. KEK 不与密文同库、不进备份、不经 Cloud 业务库传递；由部署交付，Store 不生成、不持久化 KEK 本身。
3. `secret_ref` 是不透明定位标识，不是秘密、不是密文、不是 bearer token；持有它不构成解析权；
   它由 Store 生成并返回，浏览器、终端用户与部署方都不得自行指定。
4. 解析权、写入／撤销权与密钥操作权分属三个不同服务身份（`broker-resolver`／`ingress-writer`／
   `kek-operator`）；解析身份没有列举能力；解析还需同时出示 Cloud 签发的单次授权，由 Store 自己校验。
5. 撤销的权威在 Cloud 元数据；Store 侧删除失败或从旧备份恢复都不能使凭据重新可用。
6. 创建、轮换与撤销是跨两个存储的操作，必须有幂等键与显式补偿；孤儿与悬空都可对账。
7. Broker 授权短时、单次，绑定 tenant／owner／project／workspace／execution／credential id／version／
   repository identity／Node 实例与运行代次，且自身是秘密，不落任何持久介质。授权在**签发时是未使用
   状态**，由 Store 在**兑换时原子消费**；消费记录持久在 Store 侧，重放一律拒绝，未拿到秘密的失败
   只能申请新授权。
8. Store 不可用、不可核实或密钥缺失时明确失败，不降级匿名、不降级静态凭据。
9. 审计与日志记录操作与结果，不记录秘密、密文、密钥或授权载体。
10. 后端可替换是部署属性；Cloud 与 Node 只依赖 Store 的服务接口与本文的语义。

## 为什么不是这些替代方案

| 方案 | 优势 | 否决理由 |
|---|---|---|
| Cloud 业务库直接加密保存 Token | 少一个组件，事务简单，本地开发最省事 | 密钥必须与密文同库或同进程才自洽，数据库导出即等于 Token 泄露；且与 Cloud 只保管引用与授权的既有边界冲突，修订代价最大 |
| 外部 Secret Manager 作为第一版唯一形态 | 生产形态成熟，轮换与审计现成 | 本地 Docker Compose 无法承载，需要额外的凭据引导（"谁来解锁"）；作为**可替换后端**优于作为第一版前提 |
| 用 Cloud 主密钥直接加密 Token | 少一层封装 | 让 Cloud 进程本身成为解密能力，与"Cloud 不持久化 Token"冲突；不变量 2 因此要求 KEK 独立交付 |
| 让 `secret_ref` 成为可兑换的引用（"有引用就能取"） | 接口最少，调用方无需再申请授权 | 引用会出现在业务库、消息与日志中，等于把秘密的定位标识升级成能力；与不变量 3 冲突，且无法回答"一次能取几条" |
| 把 Broker 授权做成可持久缓存、跨执行复用 | 减少往返，实现简单 | 授权是 bearer，持久化等于把秘密写进 Node 数据库与 process journal；并破坏单次使用与 fencing（不变量 7） |
| 在**签发时**就把授权标记为已使用（"一次性空凭据"） | 表面上最省事，不需要消费记录 | 签发与兑换之间必然有时间差（交付、fencing、排队），这样会让尚未兑换的授权立即失效；更严重的是把强制点从兑换挪到签发，"同一授权被出示两次"不再有可判定结果（D10） |
| 在内存里记"已用过"，不落持久介质 | 不增加存储，实现最快 | Store 重启后已消费的授权会被重新接受，等于把重启本身变成重放窗口；与 D10 的持久消费记录冲突 |
| 先交付秘密、后写消费记录 | 少一次写事务 | "响应丢失"会变成一次可重放的成功——秘密已经泄露而记录不存在；顺序不可颠倒（D10） |
| 撤销时先删 Store、后改元数据 | 直觉上"先断秘密" | 删除失败会留下"元数据仍可用、秘密已消失"的不确定窗口；反过来的失败方向是安全的（不可用但秘密可能残留，交给清扫） |
| 用 Store 的备份恢复"找回"被误删的凭据 | 运维方便 | 会让已撤销凭据复活，破坏 D4 的撤销权威与不变量 5 |
| 让 Cloud 的解析组件同时具备列举能力 | 便于运维排查与对账 | 单点被攻破即全量泄露；与不变量 4 冲突，对账需求改由 D4 的幂等键与清扫满足 |
| 在 Cloud 与 Store 之间直接传递 Token 明文而不加密存储 | 实现最简单 | 违反不变量 1；且失去"数据库导出不足以解密"这一性质，D2 的存在正是为了它 |

## 失败与恢复语义

| 失败点 | 对外结果 | 持久事实与恢复 |
|---|---|---|
| Store 写入失败 | 保存失败，不返回 credential id | 无元数据行；可安全重试 |
| Store 成功、元数据事务失败 | 保存失败 | 产生孤儿秘密；由 Store 侧按"未被引用 + 超过 TTL"清扫，不对用户报成功 |
| 元数据成功、秘密不可读（悬空） | 绑定可读，Clone 明确失败 | 引用保留；Store 恢复后重试是新执行 |
| KEK 缺失或不可用 | Store 启动拒绝 | 不表现为空 Store；不降级为明文或明文回退（D8） |
| 密文完整性校验失败 | 解析明确失败 | 记为不可核实；不重试解密，不返回部分内容 |
| Store 服务认证失败 | 写入／解析明确失败 | 不重试为匿名，不回退其他身份 |
| 撤销时 Store 删除失败 | 撤销对用户仍然成功 | 元数据已不可用；残留秘密进入清扫队列 |
| 从旧备份恢复 Store | 已撤销凭据仍不可用 | 撤销权威在元数据；恢复不改变 `availability` |
| Broker 授权过期或被重放 | 本次 clone 失败 | 重试申请新授权；不缓存、不续期 |
| 授权兑换成功、但响应在途中丢失或请求超时 | 本次 clone 失败 | 该授权**已被消费**，不得重发；重试必须由 Cloud 重新签发并通过全部校验（D10） |
| 消费记录已写入、秘密交付前 Store 崩溃 | 本次 clone 失败 | 结果为"已消费未交付"；重试申请新授权（D10） |
| Store 重启、滚动升级或从崩溃恢复 | 已消费的授权仍不可用 | 消费记录持久在 Store 数据存储中，不随重启丢失（D10） |
| 凭据在授权签发后、兑换前被冻结 | 已签发的授权不撤回，随到期失效 | 冻结后不再签发新授权；不改变已签发授权的可用性（D10） |
| 授权出示两次（并发或串行） | 至多一次成功，其余以"已被消费"拒绝 | 原子消费保证唯一成功（D10） |
| KEK 轮换进行中 | 新旧封装都可解 | 轮换幂等，失败可重入 |
| Cloud 配置了凭据能力但无法建立服务身份 | 该能力启动拒绝 | 不进入运行期逐次失败（D6） |

## 本决策未解决的问题

- **本地 Compose 与生产后端的替换边界（部分已定，具体选型待裁决）**：已定的是"后端可替换是部署属性
  而非协议属性"（D1、不变量 10）——本地 Compose 用内置加密存储并经 `*_FILE` 交付 KEK，生产可换成外部
  Secret Manager，两者只暴露同一组服务接口，Cloud 与 Node 的协议不随之改变。待裁决的是**具体选型与
  拓扑**：内置存储的进程与卷布局、本地 Compose 的服务拓扑，以及 KEK 交付的具体介质（卷／docker secret／
  宿主文件）。
- **`secret_ref` 的生成权已裁决，收口形态待设计**：生成权归 Store，浏览器、终端用户与部署方都不得指定
  （D1、不变量 3）。仍然开放的是**工程收口方式**：`ConfigureCredential` 这条部署 CLI 路径是改为"由 CLI
  向 Store 申请引用"，还是整体退役改由 Ingress 承担，需要与 §`0023` 补录中记录的"唯一写入路径"现状一并
  设计——在它收口之前，经部署 CLI 创建的凭据仍然带着可被部署方随意填写的 `secret_ref`。
- **孤儿与悬空的清扫时限**：TTL 取值、清扫由谁执行（Store 侧还是 Cloud 侧），以及清扫本身是否需要审计，
  属于运维与合规裁决。
- **撤销的追溯力**：已发放的 Broker 授权是否必须在撤销时立即失效（主动吊销），还是仅随到期失效。前者
  需要 Cloud 能触达在途执行，后者留出一个窗口；这与 Cloud ADR 的同名未决项是同一问题。
- **审计的留存与外部化**：审计是否必须写入独立的、Cloud 不能改写的介质，还是留在 Cloud 业务库即可。
- **多后端语义一致性**：可替换后端是否必须实现同一组语义（尤其是"删除失败"与"无列举能力"），还是允许
  后端差异由适配层吸收。
- **领域归属已裁决**：本叶子的三份凭据决策保留在 `identity-access`（人工裁决）。`decisions/cloud/README.md`
  对"凭据引用"划归 `project` 的措辞因此与裁决不一致，是否同步调整 README 留待人工裁决（与 Cloud ADR 的
  同名未决项一致）。

## 落地顺序

每一步都可独立验收。第 1 步即**开发准入**（[Cloud ADR](20261010-user-managed-git-credentials.md) D11-A
的定义适用于本叶子全部凭据决策）；第 6 步的解除 fail-closed 属于**功能启用**（D11-B），两者不可互相
替代——准入完成不解除任何 fail-closed。**准入已于 2026-10-10 满足，Node 侧隔离方案也已 `approved`，
因此下列步骤可以开始；但没有任何一步已经开始。**

1. **规范与补录——已完成（2026-10-10）**：领域归属与 `secret_ref` 生成权已裁决（保留 `identity-access`；
   生成权归 Store），后端替换边界已定（可替换是部署属性），具体选型仍待裁决；`cloud/AGENTS.md` 的
   范围受限 amendment 正文由
   [凭据规则受限例外](20261010-user-managed-git-token-ingress-rule-exception.md)（approved）承载并已落地，
   `0023` 的既有语义由
   [仓库凭据归属连续性](20261010-repository-credential-ownership-continuity.md)（approved）承载。本步
   不产生任何代码或测试证据，**因此不构成实现或解禁的依据**；第 2 步尚未开始。
2. **Store 骨架与密钥边界（不接用户 Token）**：服务身份、KEK 交付、加密封装、启动拒绝与审计骨架。
   完成条件：无 KEK 时启动拒绝；单独导出数据库或数据目录都无法解密；日志与错误中无秘密。
3. **Ingress 与元数据一致性**：受限写入、幂等键、孤儿与悬空对账。完成条件：写入一致性与补偿在真实
   PostgreSQL 加真实 Store 上有证据。
4. **Broker 授权签发与兑换**：D5 的绑定、到期与 fencing，以及 D10 的兑换时原子消费、持久消费记录与
   重放拒绝。完成条件：重放、跨执行、跨实例、跨代次全部被拒绝；Store 重启后已消费的授权仍被拒绝；
   响应丢失或超时后旧授权不可重发。
5. **轮换、撤销、备份恢复与 KEK 轮换**：D4 与 D8。完成条件：轮换不中断在途执行；撤销后从旧备份恢复
   不复活凭据；KEK 轮换期间新旧封装都可解。
6. **解除 fail-closed**：在 Cloud ADR 的解除条件全部满足后，移除 `repository_credential_executor_unavailable`
   的无条件拒绝，保留其前置校验。

保持不变的行为：公开仓库匿名 Clone、既有 Workspace 的恢复与清理、冻结与归属连续性语义、所有已应用的
migration 与已发布的公开 API 形状。
