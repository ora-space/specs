---
status: approved
---

# Git 凭据由独立的 Credential Store 保管：浏览器直接提交 token，Cloud 只管引用与能力，执行器一次性兑换

用户托管的外部 Git 凭据（第一阶段是 HTTPS token）由一个独立的 Credential Store 保管。浏览器凭 Cloud 签发的一次性提交授权，
把 token 直接提交给 Credential Store 的专用入口，token 不经过 Cloud 和 Gateway。Cloud 只保存凭据引用：归属（团队、个人、未知）、
作用的仓库、可用状态、当前版本，以及 `read` 和 `publish` 两项能力是否已经验证。明文只在 Repo Executor 执行一次任务时，凭
Cloud 签发的一次性授权兑换一次。`read` 由第一次成功的同步证明；`publish` 必须经过一次真实的推送能力验证：请求外部仓库
`git-receive-pack` 的 ref 广告，不写入任何内容。

本 ADR 已经 owner 批准，处于 `approved`（核心测试用例见[Git 凭据存储用例](../../../test-cases/cloud/identity-access/git-credential-store.md)），是 `cloud/identity-access` 领域在[用户 Git 提交身份](20260928-user-git-commit-identity.md)之后的后续决策。
它按评审中推荐的方案回答两项安全规则：个人 token 的入口（D3）与推送能力的验证（D4）；第三项“发布的审批人”的使用规则在
D5 给出约束，审批流程由 [发布 ADR](../project/20261017-explicit-fast-forward-publish.md) 承担。涉及 Cloud（凭据引用、提交授权、
能力验证、设置页）、Credential Store、Repo Executor 与边缘反向代理。

兼容性边界：

- 不给 `cloud/AGENTS.md` 的硬规则开例外：Cloud 仍然不接收、不保存、不记录、不返回任何 token。
- **破坏性切换**（契约范围边界）：不设过渡开关，不做数据迁移。`credential_refs` 直接改为本文的形状，已有的开发环境数据在切换后
  重建；部署侧预置凭据的路径不再保留。
- 凭据连续性 ADR 的归属与冻结规则不变；本文只把“真实凭据由部署侧保管”改为“由 Credential Store 保管”。

## 继承与修改

| 前序约定 | 本文处理 |
| --- | --- |
| [identity-access 根决策](0-gateway-mediated-external-identity.md)：外部 provider token 不持久化、不进入 Cloud；Cloud 是授权权威 | 继承，不变。用户托管的 Git token 同样不进入 Cloud（D3）；谁能使用哪个凭据由 Cloud 判定（D5） |
| [用户 Git 提交身份](20260928-user-git-commit-identity.md)：git 身份是自述的提交署名，不是认证身份，不参与授权 | 继承，不变。提交身份与访问外部仓库的凭据无关，凭据的所有者也不由提交作者推断 |
| [仓库凭据连续性](../project/20260927-repository-credential-continuity.md) D1、D2：团队、个人、未知三类；关联成员离开后个人和未知引用冻结，重新加入不自动解冻 | 继承，不变（D1） |
| 凭据连续性开头：“Ora 管理凭据引用，真实凭据仍由部署侧保管”；D3：管理员选择部署侧已配置的团队引用 | 修改：真实凭据由 Credential Store 保管，用户和管理员经浏览器直接提交（D2、D3）。部署侧预置凭据的路径删除 |
| 凭据连续性 D3：“只验证读取不能被记录为已经拥有推送权限” | 继承并具体化：`read` 与 `publish` 分别验证，验证方式见 D4 |
| 凭据连续性 D4：执行端按引用选择，临时提供，不回退全局凭据 | 继承。执行端是 Repo Executor，兑换方式见[执行器根决策](../repo-executor/0-repo-executor-sole-external-git-boundary.md) D3 |
| `cloud/AGENTS.md`：凭据只是基础设施引用，Cloud 不接收、不保存、不记录、不返回 token | 继承，不开例外（D3） |

## 决策

### D1：凭据引用

Cloud 的 `credential_refs` 是凭据的唯一元数据权威，保留现有列（`attribution`、`availability`、`repository_scope`、`version` 等），
增加：

| 字段 | 含义 |
| --- | --- |
| `secretState` | `awaiting_secret`（已创建，等待浏览器提交）、`stored`（Credential Store 中有当前版本）、`revoked` |
| `storeVersion` | Credential Store 中的当前版本号；每次提交新 token 加一 |
| `capabilities` | 已对 `storeVersion` 验证的能力集合，取值 `read`、`publish`；列名沿用，取值从旧的 `clone`/`push` 改为新词表 |
| `requestedCapabilities` | 创建时用户声明需要的能力，只决定是否自动发起 `publish` 验证，不授予任何权限 |
| `capabilityEvidence` | 每项能力的证据：验证时间、`storeVersion`、来源（同步 Operation ID 或验证记录 ID） |

规则：

- **作用范围**：`repositoryScope` 必填，且是一个具体仓库的 `https://` 地址（规范化：host 小写，去掉 `:443`、末尾 `/` 和 `.git`）。
  第一阶段一个凭据只作用于一个仓库。推送能力只能对具体仓库验证（D4）；按 host 作用的凭据无法说明它在哪些仓库上能推送。
- **归属**：个人凭据由用户为自己创建，`owner_user_id` 是本人；团队凭据只能由租户 `admin`/`owner` 创建，`attribution = team`，
  `attribution_basis` 记录创建的管理员和时间。归属在创建时确定，之后不能改；需要换归属就新建一个凭据。
- **版本变化**：提交新 token 后 `storeVersion` 加一，`capabilities` 清空，重新验证。旧版本的能力不能延续到新版本：两个 token
  的权限可能不同。
- **可用状态**：沿用凭据连续性的 `available`、`frozen`、`invalid`。外部仓库以认证失败拒绝时，该版本标为 `invalid`；提交新 token
  后回到 `available`（冻结的引用除外，冻结仍需管理员处理）。
- `secretState` 不是 `stored` 的引用不能签发兑换授权，使用它的同步和发布以 `credential_not_in_store` 拒绝。

### D2：Credential Store

Credential Store 是独立程序，参考未合并的旧实现中 `internal/credentialstore` 的设计新写（不依赖其代码），不与 Cloud 共享数据库或卷：

- **加密**：信封加密。每个秘密一个随机 32 字节 DEK，以 AES-256-GCM 加密 token，附加数据是凭据 ID；DEK 再以 KEK 用 AES-256-GCM
  包裹，附加数据是 KEK 的 ID。KEK 是部署挂载的只读 keyring 文件，有一个主密钥，可以保留旧密钥用于解包；KEK 缺失时拒绝启动。
- **存储**：凭据卷上每个凭据一个文件，原子写入（临时文件、fsync、rename、目录 fsync）；卷的权限不允许组和其他用户读取。
- **版本**：每次提交写入新版本，旧版本保留 1 小时，只用于兑换在此之前签发的授权，之后销毁。
- **撤销**：写入撤销标记，同时销毁所有版本的密文。撤销幂等。
- **一次性兑换**：验证授权签名（Cloud 的 Ed25519 公钥集合）、用途、绑定的凭据与版本、过期时间（以 Credential Store 自己的时钟），
  再以授权 ID 独占创建兑换记录，记录落盘之后才解密。同一授权的第二次兑换被拒绝。
- **身份**：

  | 入口 | 调用方 | 能做什么 |
  | --- | --- | --- |
  | 兑换接口（mTLS） | 只认 `spiffe://ora.local/repo-executor` | 凭 `sync`、`publish`、`probe` 授权兑换明文；授权中的 `executorIdentity` 必须等于证书身份 |
  | 管理接口（mTLS） | 只认 `spiffe://ora.local/cloud/credential-manager` | 查询元数据（版本、提交记录 ID、是否撤销）、撤销；**不能写入明文，也不能读取明文** |
  | 密钥接口（mTLS） | 只认 KEK 运维身份 | KEK 轮换 |
  | 浏览器入口（HTTPS，无客户端证书） | 持有一次性提交授权的浏览器 | 提交一个新版本（D3） |

  每个身份只有一项能力，同时持有两项能力的证书在启动时被拒绝（旧实现的规则）。

相对旧实现设计的不同：去掉 Cloud 写入明文的 `ingress-writer` 能力，改为浏览器入口；兑换身份从 `broker-resolver` 改为执行器的
服务身份；兑换的用途从 `clone`、`mirror_prepare`、`push` 改为 `sync`、`publish`、`probe`。

**撤销的时效**：Credential Store 不读 Cloud 的数据库，无法在兑换时知道 Operation 是否仍然有效。撤销的顺序是：Cloud 先在事务内
把引用置为 `revoked`（此后不再签发授权），再调用管理接口撤销（此后不能兑换）。两步之间已经签发的授权，最多在其有效期
（不超过 2 分钟）内仍可兑换；撤销的界面说明这一点。管理接口调用失败时 Cloud 重试，直到成功。

### D3：浏览器直接提交 token

```text
浏览器 ──(1) POST /git-credentials {元数据，不含 token}──▶ Gateway ──▶ Cloud
       ◀─(1') {credentialId, submissionGrant, ingressUrl}────────────────
浏览器 ──(2) PUT ingressUrl {token}，Authorization: Ora-Submission <grant>──▶ Credential Store 浏览器入口
       ◀─(2') {storeVersion}────────────────────────────────────────────
浏览器 ──(3) POST /git-credentials/{id}/confirm──▶ Gateway ──▶ Cloud ──(3') 管理接口查询元数据──▶ Credential Store
```

1. **创建或轮换**：Cloud 校验调用者的权限（D1 的归属规则；轮换个人凭据只限所有者，轮换团队凭据只限管理员），创建或更新
   引用为 `awaiting_secret`，签发提交授权。请求体按严格解码，带 `token` 字段的请求直接拒绝。
2. **提交授权**：Ed25519 签名，使用与执行授权**不同的密钥和用途**（`submit`），字段为 `submissionId`、`tenantId`、`credentialId`、
   `ownerUserId`、`attribution`、`repositoryScope`、`expectedStoreVersion`、`issuedAt`、`expiresAt`（不超过 5 分钟）。Credential
   Store 验证签名、用途和有效期，以 `submissionId` 独占认领，写入 `expectedStoreVersion` 版本；版本不符（并发轮换）时拒绝。
3. **确认**：浏览器把结果告诉 Cloud，但 Cloud 不信任浏览器的报告，经管理接口查询该凭据的当前版本和对应的 `submissionId`，
   一致才把引用置为 `stored` 并更新 `storeVersion`，然后发起能力验证（D4）。浏览器没有回来确认时，Cloud 的定时任务在授权过期后
   查询一次，按同样规则补上确认或把引用标为已放弃。

**浏览器入口**：

- 独立的 origin（例如 `credentials.<部署域名>`），由边缘反向代理直接转发到 Credential Store 的浏览器监听，不经 Gateway 和 Cloud；
- 只有一个路由：`PUT /v1/credentials/{credentialId}/versions`；请求体上限 16 KiB；
- CORS 只允许 Ora 前端的 origin 和 `PUT`；不使用、也不接受 cookie，认证只靠提交授权，所以不存在 CSRF；
- 响应只有版本号，从不回显 token；入口按来源地址限速。

为什么不经 Cloud 转交：转交需要给硬规则开例外，Cloud 的请求处理、错误路径和日志都要证明不碰 token，而且这份证明要随 Cloud 的
每次修改重新成立。直接提交让 token 只出现在浏览器和 Credential Store 两处。代价是 Credential Store 多一个公开入口，并要自己
校验提交授权；入口只接受写入，没有任何读出明文的路径。

为什么也不经 Gateway：Gateway 持有浏览器会话、记录请求日志，同样是 Cloud 仓库中处理所有用户请求的组件；让 token 绕过它，
硬规则的边界就是一个进程，而不是一组路由。

### D4：能力词表与验证

| 能力 | 含义 | 怎样验证 | 何时失效 |
| --- | --- | --- | --- |
| `read` | 能用这个版本 fetch `repositoryScope` | 第一次用该版本成功完成的同步（[同步 ADR](../project/20261016-repository-binding-and-snapshot-sync.md) D5），在登记同步结果的事务中记录 | 提交新版本；同步以认证失败结束 |
| `publish` | 能用这个版本向 `repositoryScope` 推送 | 执行器以 `probe` 授权请求 `git-receive-pack` 的 ref 广告（执行器根决策 D7），平台返回广告即验证通过；不发送任何更新命令 | 提交新版本；发布被平台以权限不足拒绝 |

- `publish` 验证在以下时机发起：引用确认时 `requestedCapabilities` 包含 `publish`；用户在界面点击“验证推送权限”。验证记录
  `credential_verifications` 保存 `storeVersion`、仓库、结果分类和时间，不保存平台的原始输出。
- 平台以 401/403 拒绝 receive-pack 广告时，记录 `push_denied`，`publish` 不进入能力集合；网络等其他错误记录 `verification_failed`，
  可以重试。
- `publish` 验证通过不保证每次发布都会成功：保护分支、必需的状态检查、hook 只在真正推送时才判定，那时以 `rejected` 结束
  （发布 ADR）。各平台对“有读无写”的 token 请求 receive-pack 广告时的响应，在落地时逐一验证并留下测试。
- 发布 Operation 只能使用 `capabilities` 包含 `publish` 的引用；`read` 不是同步的前提，而是同步成功的结果。

为什么不接受用户自述：自述的推送权限要到第一次发布时才被证伪，那时用户已经完成审批、等待结果，失败的位置离原因很远。
receive-pack 广告是 Git 协议在写入前就会做的认证与授权检查，可以在不写入的前提下提前暴露最常见的错误（只读 token、错误的仓库）。

### D5：谁能使用凭据

| 动作 | 团队凭据 | 个人凭据 |
| --- | --- | --- |
| 创建、轮换 | 租户 `admin`/`owner` | 只有所有者本人 |
| 绑定到 Project（用于同步读取） | 租户 `admin`/`owner` | 只有所有者本人 |
| 批准用它发布 | 拥有 Project 发布权限的人（发布 ADR D2） | 只有所有者本人 |
| 撤销 | 租户 `admin`/`owner` | 所有者本人，或租户 `admin`/`owner` |
| 查看元数据 | Project 的成员 | Project 的成员 |

- 租户管理员**不能**使用他人的个人凭据：不能用它绑定、不能批准用它发布。管理员能做的是撤销或等待冻结，然后绑定团队凭据
  （凭据连续性 D3）。撤销是收紧方向的操作，允许管理员执行。
- 个人凭据被所有者绑定到 Project，等于所有者同意该 Project 的同步用它读取；同步不逐次批准。发布逐次批准，因为发布以所有者的
  名义改变外部仓库。
- 所有者离开租户时，个人凭据按凭据连续性冻结；之后的发布需要改绑团队凭据。这是推荐方案接受的代价。

### D6：设置页

- 凭据列表：名称、仓库、归属、状态（等待提交、可用、冻结、失效、已撤销）、`read` 与 `publish` 是否已验证及时间。
- 新建与轮换：token 输入框的内容只发往 Credential Store 的浏览器入口，提交后立即从页面状态中清除；Cloud 的请求不含 token。
- “验证推送权限”按钮，显示最近一次验证的结果分类。
- 团队凭据的新建入口只对管理员显示。

## 不变量

1. Cloud 和 Gateway 从不接收、保存、记录或返回 token 明文。
2. Credential Store 只向 `spiffe://ora.local/repo-executor` 返回明文，每份授权至多兑换一次。
3. 提交授权与执行授权使用不同的签名密钥和用途，互相不能替代。
4. 浏览器入口只能写入新版本，不存在读出明文的路径。
5. 引用只有在 Cloud 经管理接口确认 Credential Store 中存在对应版本后才成为 `stored`。
6. 能力只对一个 `storeVersion` 成立；新版本的能力集合为空。
7. `publish` 只能由一次成功的 receive-pack 广告请求证明；`read` 只能由一次成功的同步证明。
8. 租户管理员不能绑定他人的个人凭据，也不能批准用它发布。
9. 撤销之后不再签发授权；Credential Store 的撤销完成后不能再兑换。
10. 只有 `secretState = stored` 的引用能被兑换。

## 为什么不是这些替代方案

| 方案 | 否决理由 |
| --- | --- |
| token 经 Cloud 转交给 Credential Store（旧实现） | 实现最简单，前端只调一个 API；但要给硬规则开例外，并持续证明 Cloud 的所有路径不泄露 token（D3） |
| token 经 Gateway 转交 | Gateway 也是 Cloud 仓库中处理全部用户请求的组件，同样需要例外；边界从一个进程扩大为一组路由 |
| Cloud 数据库加密保存 token | Cloud 失陷即可解密；违反不变量 1 |
| 部署侧保管、管理员在配置文件中预置（凭据连续性的现状） | 用户无法自助；个人凭据无从提交 |
| 推送能力由用户自述 | 错误要到发布时才暴露（D4） |
| 用一次 `--dry-run` 推送验证推送能力 | 需要本地有可推送的对象，并且仍要连接 receive-pack；只请求广告就足够，而且明确不写 |
| 管理员可以批准使用他人的个人凭据 | 所有者离开后仍可继续发布，但这是以离开者的外部身份写入，没有外部平台的授权依据（凭据连续性的替代方案表） |
| 一个凭据作用于整个 host | 用户少建几个凭据；但推送能力无法对“所有仓库”验证，也无法在冻结时说明影响面 |

## 本决策未解决的问题

- **GitHub App、SSH key 等凭据类型**：需要 Credential Store 保存私钥并由执行器换取短期 token；词表与验证方式可以沿用。
- **多仓库作用范围**：一个 token 在多个仓库上的能力需要逐仓库验证，留给多 Binding 的后续决策。
- **KEK 的外部 KMS**：第一阶段是挂载的 keyring 文件。
- **Credential Store 多副本与备份**：与 Git Server 一样第一阶段单实例。
- **兑换时核对仓库作用域**（纵深安全加固，契约“本决策未解决的问题”）：第一阶段 Credential Store 兑换时只核对授权中的凭据与版本，
  不核对授权中的仓库是否等于凭据的 `repositoryScope`；作用域由 Cloud 签发授权前检查，并由 `git-credential-ora` 按 host 与 path
  限制凭据只发给作用域内的请求。让 Credential Store 自己也核对，留给后续决策。

## 落地顺序

1. **Credential Store**：参考旧实现的设计新写，包含浏览器入口、提交授权、管理接口、执行器的兑换身份与兑换用途。完成条件：
   提交、确认、兑换、撤销的测试；同一提交授权、同一执行授权都只能使用一次。
2. **Cloud 的引用与授权**：`credential_refs` 的新形状、提交授权签发、确认、撤销顺序、能力字段。完成条件：Cloud 的 API 拒绝带
   `token` 的请求。
3. **能力验证**：`read` 接入同步登记，`publish` 接入执行器的 `probe`。
4. **设置页**：改为直接提交。

保持不变：凭据连续性的归属与冻结规则、Gateway 的身份认证。从第 2 步开始改变：凭据的真实值只在 Credential Store。

## 外部线的独立验收

不依赖内部线和 Node。起栈同[执行器根决策](../repo-executor/0-repo-executor-sole-external-git-boundary.md)的独立验收，加边缘
反向代理和一个前端（或用脚本模拟浏览器的三步）。

| 场景 | 必须成立 |
| --- | --- |
| 个人凭据的提交与确认 | token 只出现在发往浏览器入口的请求中；引用成为 `stored`；Cloud 与 Gateway 的访问日志中没有 token |
| Cloud 收到带 `token` 字段的请求 | `400`，不创建引用 |
| 提交授权重放、过期、被改动 | Credential Store 拒绝 |
| 用执行授权调用浏览器入口，或用提交授权兑换 | 都被拒绝 |
| 浏览器谎报版本后确认 | Cloud 经管理接口发现不一致，不置为 `stored` |
| 轮换 | 新版本的能力为空；旧版本在 1 小时后不可兑换 |
| `publish` 验证：`push-git-fixture` 的可写 token | 验证通过，`publish` 进入能力集合；fixture 的请求日志中只有 receive-pack 的广告请求，没有更新命令 |
| `publish` 验证：只读 token | `push_denied`；用它创建发布 Operation 被拒绝 |
| `read` 验证 | 第一次同步成功后 `read` 进入能力集合；匿名或错误 token 的同步不会产生 `read` |
| 撤销 | 撤销后的同步与发布被拒绝；Credential Store 中该凭据的密文已销毁 |
| 管理员与个人凭据 | 管理员绑定他人的个人凭据、批准用它发布，都被拒绝；管理员撤销它成功 |
| 成员离开 | 个人凭据冻结；重新加入后仍冻结 |
| 秘密扫描 | canary token 经浏览器提交、兑换、同步、发布、撤销全程后，在所有日志、Cloud 数据库导出、Git Server 仓库、执行器任务目录、Credential Store 卷的明文部分中都找不到 |

**最终验收**：用 owner 提供的 GitHub 私有测试仓库与真实 token，经浏览器提交、确认，完成 `publish` 验证；再用一个只读的
fine-grained token 证明 `push_denied`。
