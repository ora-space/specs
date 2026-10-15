---
status: proposed
---

# Cloud 仓库级凭据硬规则为用户个人 HTTPS Git Token Ingress 建立受限例外，原规则其余部分全部保留

`cloud/AGENTS.md:119-123` 今天禁止 Cloud 接受、持久化、记录、返回或提交任何原始 repository
credentials。用户自助 HTTPS Git 凭据的产品方向要求 Cloud 在**一次请求内**接收用户提交的个人 Token
并立即转交给独立 Credential Service，因此需要对该条规则做**范围受限的正式修订**。本文承载这次修订的
权威正文：在原规则之后追加一段仅覆盖"经认证与授权的用户个人 HTTPS Git Token Ingress"的例外，并逐条
写明例外的生效条件。**原规则不删除、不重写、不弱化**，其对象存储凭据、部署私钥、服务账号令牌、
SSH 私钥、团队共享凭据与测试密钥约束继续完整有效。

本文的方向已由项目负责人在运行 issue `ora-space/circus#29` 的授权中确认，但**尚未经过 specs 的 PR
评审**，按 `specs/AGENTS.md` 保持 `proposed`，评审通过后再据此标记 `approved`。**规则修订的文本已经
随 cloud#64 提交**：D1 的五个条件已作为追加段落落在 `cloud/AGENTS.md` 的原规则之后，原规则逐字保留；
但承载这段正文的**本文仍是 `proposed`**，该例外是否长期保留取决于 `ora-space/circus#135` 的裁决。
**规则文本的落地不等于运行时解禁**：`requireRepositoryAccess` 的
`repository_credential_executor_unavailable` 无条件拒绝继续有效，凭据仍不得进入任何实现路径，直到
Cloud ADR D11-B 与 Node ADR D14-B 的全部条件满足。本文不授权任何实现变更，不修改已应用的 migration，
不修改其他 ADR 的状态或决策。

本文的**修订对象**是 Cloud 仓库的规则文件，不是 Cloud 的运行时行为；**正文承载者**是本文，而不是
[用户自助 HTTPS Git 凭据](20261010-user-managed-git-credentials.md) D3——后者只提出需求并链接到本文，
以免同一段例外文本出现两个权威副本。

涉及模块：`cloud/AGENTS.md`（仓库级硬规则）、Cloud 的 Credential Ingress 端点与其请求解码路径、
日志／追踪／错误上下文边界、Secret Store 的服务调用（见
[用户 Git Token 由独立 Credential Service 承载](20261010-user-managed-git-secret-store.md)）。
兼容性边界：原规则的**全部**既有限制按硬边界处理，例外只能收窄不能扩张。

## 继承与修改

| 前序决策 | 处理 |
|---|---|
| [用户 Git Token 由独立 Credential Service 承载](20261010-user-managed-git-secret-store.md)（proposed，本叶子前序决策）D3：Store 只接受服务身份、写入／撤销与解析分权；D9：`secret_ref` 不构成取用秘密的能力 | 继承，不变；本文规定 Cloud 侧接收 Token 的那一次调用必须走该 ADR 的 `ingress-writer` 服务身份，且接收后 Cloud 只保留元数据与 `secret_ref` |
| 同上不变量 1：Token 明文只存在于 Store 的加密存储与一次受限写入路径；Cloud 业务库、公开响应、业务事件、Controller／Node 常规消息、持久日志、审计记录与 Workspace 文件都不含 Token | 继承，不变；本文的例外条件 2–3 是这条不变量的规则侧投影 |
| 同上不变量 9：审计与日志记录操作与结果，不记录秘密、密文、密钥或授权载体 | 继承，不变；例外**不**放宽日志与审计边界 |
| [用户自助 HTTPS Git 凭据](20261010-user-managed-git-credentials.md)（proposed）D3：Token 只经受限 Ingress 进入 Store，并随该文提出范围受限的硬规则修订建议 | **修改**：例外的权威正文由本文承载，该文 D3 改为链接本文；需求与理由仍由该文陈述 |
| [仓库凭据归属连续性](20261010-repository-credential-ownership-continuity.md)（proposed）D1–D6：`credential_refs` 的归属、可用性与冻结语义 | 继承，不变；例外不改变凭据的归属与可用性判定 |
| `cloud/AGENTS.md:119-123`：Credentials remain infrastructure references. Never accept, persist, log, return, or commit raw … repository credentials | **修改（范围受限）**：追加一段只覆盖用户个人 HTTPS Git Token Ingress 的例外；原句与其后的测试密钥、日志脱敏要求全部保留 |

## 背景与现状

### 现有规则原文

`cloud/AGENTS.md:119-123`：

> Credentials remain infrastructure references. Never accept, persist, log, return, or commit raw
> deployment private keys, access tokens, passwords, or repository credentials. Generated test keys
> may exist only in isolated process memory or temporary paths and must never reach logs or durable
> fixtures. Logs use structured fields and request/resource IDs while excluding secrets and
> unnecessary personal data.

这条规则把"凭据"整体归入基础设施引用，包含四类禁止（accept／persist／log／return／commit）和两类
对象（部署材料与 repository credentials），并在同一段里独立约束测试密钥与日志脱敏。

### 为什么不能靠实现绕过

产品方向要求**用户**提交自己的 HTTPS Git Token，并由 Cloud 接收后转交 Credential Service。这与
"never accept … repository credentials" 直接冲突：冲突点不在措辞，而在规则范围。任何"端点不进日志
就不算 accept"或"只存密文就不算 persist"的解读都是把一条仓库级硬规则解释成可以局部失效，而这正是
该规则要防止的事。**因此必须走正式修订，并让修订后的规则仍然可以逐条判断。**

### 规则今天在保护什么（不能被例外削弱的既有约束）

- 其他三类秘密对象：部署私钥、对象存储凭据、服务账号令牌与 SSH 私钥，都由原句直接覆盖；例外的条件
  4 明确排除它们。
- 同类对象中的团队共享凭据：`attribution='team'` 今天没有任何产品写入路径
  （见[仓库凭据归属连续性](20261010-repository-credential-ownership-continuity.md) D2），例外的条件 4
  把它排除在范围之外。
- 测试密钥与日志脱敏：原句后两句独立成立，例外不涉及，也不放宽。
- 服务与终端用户凭据相互独立、授权只来自已验证 claim 与当前 PostgreSQL 状态：同文件 115-118 行的相邻
  规则继续有效，例外的条件 1 依赖它。

### 例外的目标边界

例外要覆盖的是**恰好一种**输入：一个已经通过认证的用户，在自己的会话里提交**属于自己**的 HTTPS Git
Token，用于绑定到自己的 Project。除此之外，Cloud 不得因为本文而获得接受任何其他原始凭据的能力。

## 决策

### D1：例外正文（权威文本，已于 2026-10-10 追加到 `cloud/AGENTS.md`）

以下英文段落即 `cloud/AGENTS.md` 中实际生效的例外文本，位于原条目之后；原句未改动：

> Exception — user-managed personal HTTPS Git tokens only. Cloud may accept one raw HTTPS Git token
> from an authenticated and authorized end user through a dedicated credential-ingress endpoint,
> subject to every one of the following, with no exceptions:
> 1. **Authenticated and authorized end user, own credential only.** The request is authenticated and
>    authorization derives only from verified claims and current PostgreSQL state — never from
>    caller-selected headers or body fields. The caller may only write a credential owned by itself
>    (`tenant_id` + `owner_user_id` = the caller); supplying another user's or another tenant's
>    ownership is rejected at the boundary.
> 2. **Dedicated endpoint, single position.** Only the dedicated endpoint accepts a token, and only in
>    the request body. A token must never appear in a URL, path, query, header, cookie, or any other
>    position, and no other endpoint may accept one.
> 3. **Same-request handoff, no persistence in Cloud.** Cloud must hand the token to the independent
>    Credential Service within the same request. The token must not be queued, buffered to disk, cached,
>    or written to any Cloud store — no business database column, business event, public response,
>    business message, durable file, or workspace volume. After the handoff Cloud retains only
>    `credential_refs` metadata and an opaque `secret_ref`; **no API may return the token**.
> 4. **No request bodies in logs or error context.** Request bodies must not enter access logs, tracing
>    spans, error details, or crash reports; error responses carry reason codes only. The structured-log
>    and secret-exclusion rules above remain fully in force.
> 5. **No scope expansion.** This exception does **not** apply to deployment private keys, object-storage
>    credentials, service-account tokens, SSH private keys, generated test keys, or team-shared
>    credentials. For those, the original rule above continues to apply in full, including its
>    `never accept` clause. Widening this exception requires a new decision, not an edit here.

中文对照（`cloud/AGENTS.md` 当前为英文，仓库只写入上面的英文段落；本对照仅供评审，不作为规则文本）：

> 例外——仅限用户个人托管的 HTTPS Git Token。Cloud **可以**经专用凭据入口端点接收一个来自**已认证且
> 已授权**终端用户的原始 HTTPS Git Token，但必须同时满足下列全部条件，无一例外：
> 1. **已认证且已授权的终端用户，且只能是自己拥有的凭据。** 请求必须通过认证，授权只来自已验证的
>    claim 与当前 PostgreSQL 状态，绝不来自调用方选择的 header 或 body 字段。调用方只能写入自己拥有
>    的凭据（`tenant_id` + `owner_user_id` = 调用方）；提交他人或其他租户的归属在边界处即被拒绝。
> 2. **专用端点，位置唯一。** 只有专用端点接收 Token，且只在请求体内。Token 不得出现在 URL、路径、
>    query、header、cookie 或任何其他位置，其他端点一律不得接收。
> 3. **同请求转交，Cloud 不持久化。** Cloud 必须在同一次请求内把 Token 交给独立的 Credential
>    Service，不得排队、不得落盘缓冲、不得缓存、不得写入任何 Cloud 存储——包括业务数据库列、业务事件、
>    公开响应、业务消息、持久文件与 Workspace 卷。转交后 Cloud 只保留 `credential_refs` 元数据与不透明
>    的 `secret_ref`；**任何 API 都不得返回 Token**。
> 4. **请求体不进日志与错误上下文。** 请求体不得进入访问日志、追踪 span、错误详情或崩溃报告；错误
>    响应只携带原因码。上面的结构化日志与排除秘密的规则完整继续有效。
> 5. **不得扩张范围。** 本例外**不**适用于部署私钥、对象存储凭据、服务账号令牌、SSH 私钥、生成的
>    测试密钥或团队共享凭据。对它们，上面的原规则继续完整适用，包括其 `never accept` 子句。扩张本
>    例外需要新的决策，而不是在此处编辑。

条件 1 比"接收用户 Token"这一需求本身更严：它把例外限定在"用户写自己的凭据"。理由是这条规则原本
保护的是"凭据不进入 Cloud"，而例外唯一可以辩护的形态是"用户自愿把自己的秘密交给一个托管自身仓库
访问的服务"。一旦允许写入他人的归属，例外就从"用户自助"变成"Cloud 代为保管他人秘密"，两者不能用同
一条规则覆盖。

### D2：例外的适用面与不适用面

| 对象 | 是否适用例外 | 依据 |
|---|---|---|
| 用户个人的 HTTPS Git Token（`purpose='git'`） | **适用**，须满足 D1 全部五个条件 | 例外标题与条件 1 |
| 部署私钥、部署 access token、密码 | 不适用 | D1 条件 5；原规则 `never accept` 继续有效 |
| 对象存储 access／secret key | 不适用 | D1 条件 5；既有形态仍是管理面写入共享卷、使用方按 `*_FILE` 读取，不经本例外 |
| 服务账号令牌（含控制通道 mTLS 证书与私钥） | 不适用 | D1 条件 5 |
| SSH 私钥与 SSH 主机密钥 | 不适用 | D1 条件 5；V1 只覆盖 HTTPS |
| 团队共享凭据（`attribution='team'`） | 不适用 | D1 条件 5；今天也没有产品写入路径 |
| 生成的测试密钥 | 不适用 | 原句后两句独立成立，且已经允许"仅在隔离的进程内存或临时路径"存在 |
| 日志、追踪、错误上下文、崩溃报告 | 不适用（即"例外不放宽它们"） | D1 条件 4 与原句最后一句 |

### D3：修订的落地形态：追加，而不是替换或删除

- 修订动作是**在原条目之后追加 D1 的段落**，原句与测试密钥、日志脱敏两句逐字保留。只有在原规则仍然
  成立的前提下，例外才是一条可判断的窄规则；删除或改写原句会让"什么被禁止"失去默认值。实际形态是
  **同一列表条目的续段**（与原文同缩进两格，不新增顶层条目），`git -C cloud diff AGENTS.md` 为
  24 行纯新增、零删除；该追加已于 2026-10-10 完成。
- 修订必须与本文的五个条件逐条对应，评审时可以逐条判定：条件 1 对应用户与会话边界，条件 2 对应
  端点与位置，条件 3 对应同请求转交与不持久化，条件 4 对应日志与错误上下文，条件 5 对应对象范围。
- 修订**不**要求、也**不**授权为实现便利而放宽任何其他仓库规则：请求严格解码、服务与终端用户凭据
  相互独立、授权只来自已验证 claim 与当前 PostgreSQL 状态等相邻规则都继续有效。
- 修订文本已随 cloud#64 提交进 `cloud/AGENTS.md`，但它**不是运行时的放行**：
  `repository_credential_executor_unavailable` 继续拒绝所有带凭据引用的 Project 的 clone，本文的一切
  条件都不构成实现前提。

### D4：例外不得被静默扩张，且与 fail-closed 相互独立

- **不得扩张**：任何把例外范围扩到 D2 表中"不适用"行的改动，都必须由新的决策承载，不能以"原例外已经
  允许接收 Token"为理由顺手放宽。同样，**不得**以"已经有一个接收端点"为理由新增第二个接收位置。
- **不得以已存在之物推导生效**：本文正文已写定、`cloud/AGENTS.md` 也已有例外段落，但这两件事都不构成
  例外**已定案**或 fail-closed 已解除的证据。ADR 的状态、规则文本、测试文件的存在、或端点代码的存在，
  都不等于 D11-B／D14-B 的证据已经产生——这与
  [用户自助 HTTPS Git 凭据](20261010-user-managed-git-credentials.md) D11-B 的
  "不得仅因为 ADR 存在就移除 fail-closed" 是同一条原则。本文的落地是那道门禁的 **A2 开发准入条件**，
  属于"可以开始实现"的前提，不是"可以放行"的依据。
- **独立性**：即使本文通过评审且 `cloud/AGENTS.md` 保留该例外，凭据 clone 的解禁仍然要满足 Cloud ADR D11-B
  与 [受控 Node 的仓库凭据执行使用独立 OS 身份](../../node/repository/20261010-controlled-node-git-authentication-isolation.md)
  D14-B 的**全部**条件。规则侧定案与运行时解禁是两件事，不能互相替代。

## 不变量

1. `cloud/AGENTS.md` 中"凭据是基础设施引用"的原句与测试密钥、日志脱敏两句在任何时候都不得被删除或
   弱化；例外只能是其后追加的窄段落。
2. 例外只覆盖经认证且已授权的终端用户提交的、**属于自己**的个人 HTTPS Git Token。
3. 例外不覆盖部署私钥、对象存储凭据、服务账号令牌、SSH 私钥、生成的测试密钥与团队共享凭据。
4. Token 的接收位置唯一（专用端点的请求体）；不得出现在 URL、路径、query、header、cookie 或其他端点。
5. Cloud 必须在同一次请求内把 Token 转交 Credential Service；不得在 Cloud 侧排队、落盘、缓存或写入
   任何持久介质。
6. 任何 API 都不得返回 Token 明文；转交后 Cloud 只保留 `credential_refs` 元数据与不透明的
   `secret_ref`。
7. 请求体不得进入访问日志、追踪 span、错误详情或崩溃报告；错误响应只携带原因码。
8. 例外的扩张必须由新决策承载；ADR 或代码的存在不构成例外已生效或 fail-closed 已解除的证据。
9. 规则侧定案不解锁运行时：本文的落地只是 Cloud ADR D11-A 的一个准入条件；凭据 clone 的解禁仍须满足
   Cloud ADR D11-B 与 Node ADR D14-B 的全部条件。

## 为什么不是这些替代方案

| 方案 | 优势 | 否决理由 |
|---|---|---|
| 删除 `never accept … repository credentials` 子句 | 规则变简单，不必逐条判断 | 该子句是对象存储凭据、服务账号令牌与 SSH 私钥的默认禁止来源；删掉它等于同时放开这些对象，范围变化远超本需求 |
| 把原句的 "repository credentials" 收窄为 "static repository credentials" | 一行改动即可容纳用户 Token | 用形容词区分"静态"与"用户"在评审与测试中都不可判定；团队共享凭据同样是动态的，却必须继续被禁止 |
| 不改规则，靠实现层的端点不走日志来"事实上"满足 | 零规范改动 | 正是该规则要防止的解读方式：把仓库级硬规则解释成可以局部失效，且不留可评审的范围声明 |
| 不设独立端点，在已有的 Project 创建请求里带上 Token 字段 | 少一个端点，前端更省事 | Token 会进入通用请求的解码、校验与错误路径，违反条件 2 与条件 4；专用端点是把接收面收成一点的唯一方式 |
| 允许 Cloud 先持久化 Token（哪怕是密文），稍后再交给 Store | 解耦 Cloud 与 Store 的可用性 | 与 "never persist" 直接冲突，并使 Cloud 进程成为解密能力；失败语义也失去可解释性（见 Secret Store ADR D4／D6） |
| 把例外写成"Cloud 可接收用户凭据"而不附条件 | 正文更短 | 条件就是范围；没有条件的例外在评审中无法判断某次实现是否越界，等于把原规则整条作废 |
| 让例外同时覆盖团队共享凭据 | 减少将来的修订次数 | 团队凭据的授权与成员离开规则尚未定义（Cloud ADR D2 明确另立决策）；在没有归属规则时放开接收会先制造无人负责的秘密 |
| 由本文直接修改 `cloud/AGENTS.md` | 一步到位 | 已否决：在评审通过前改动仓库级硬规则，等于用未评审的文本替换已生效的约束。实际采用的是两步路径——本文承载权威正文，评审通过后再据其修订 `cloud/AGENTS.md`（文本已随 cloud#64 提交，评审尚未完成） |

## 失败与恢复语义

| 情形 | 对外结果 | 持久事实与恢复 |
|---|---|---|
| 请求未经认证或授权 | 拒绝，不接收 Token | 边界处拒绝；不产生任何元数据 |
| 调用方提交他人的 `owner_user_id`／`tenant_id` | 拒绝（边界校验失败） | 不产生元数据；与既有 `createProject` 的 owner 校验同向 |
| Token 出现在 URL、query 或 header | 拒绝，不接收 | 视为违反条件 2；不得有"从其他位置兜底读取"的回退 |
| Token 经专用端点到达但 Store 不可用 | 保存失败，不返回 credential id | 无元数据行；可安全重试（同请求内转交失败即整体失败） |
| Store 写入成功但 Cloud 元数据事务失败 | 保存失败 | 产生孤儿秘密，由 Store 侧补偿清理；不得对用户报成功 |
| 请求体意外进入日志或错误上下文 | 视为实现缺陷 | 必须由测试与检索证据证明不发生；这是条件 4 的验收义务 |
| 用户请求取回已保存的 Token | 拒绝（无此能力） | 不变量 6；响应只说明无法取回 |
| 有人以"规则里已有例外"为由要求放行凭据 clone | 拒绝该主张 | 规则侧生效 ≠ 运行时解禁；`repository_credential_executor_unavailable` 继续拒绝，除非 D11-B／D14-B 的每一项都能指向直接证据 |
| 有人试图用本文扩张到 D2 表的"不适用"行 | 拒绝该改动 | 不变量 8：扩张需要新决策 |

## 本决策未解决的问题

- **例外的措辞终稿——已裁决（2026-10-10）**：D1 的英文段落即 `cloud/AGENTS.md` 中生效的规则文本；
  该文件当前为英文，**只有英文段落进入仓库**，中文对照保留在本文中供评审，不写入规则文件。
- **例外的验收义务归属**：条件 4（请求体不进日志与错误上下文）需要可直接指向的测试或检索证据。
  这类证据应落在核心用例、还是仅由 Cloud 的集成测试承担，需要与 Cloud ADR D11-B 的验收清单对齐。
  当前证据状态见
  [核心用例](../../../test-cases/cloud/identity-access/repository-credential-ingress-exception.md)，
  全部为 `Missing`。
- **修订后规则的名称——已裁决，随本文评审生效**：例外按原条目的**续段**落地，不单独命名、不新增顶层
  条目；后续引用以本文 D1 为准，代码注释引用本文第 D1 节而非 `cloud/AGENTS.md` 的段落编号，以免行号
  漂移。
- **多个接收端点的未来需求**：若将来需要为其他凭据类型（例如 SSH 私钥）建立接收面，必须走新的决策；
  是否会有一个统一的多类型接收端点，本 ADR 不做预判。

## 落地顺序

本文是规范修订，**没有实现步骤**；步骤 1、2 的文本都已就绪，但结论要随本文的评审一并生效，步骤 3
仍需等待。

1. **确认例外文本（方向已确认，结论待评审）**：项目负责人确认了 D1 的五个条件与 D2 的适用范围表，
   逐条可判断，且 D2 表中每一行的归属都有明确依据；评审通过并标记 `approved` 后才算生效。
2. **按该文本修订 `cloud/AGENTS.md`（文本已随 cloud#64 提交，去留随评审定）**：在原条目之后追加了
   英文段落，原句与测试密钥、日志脱敏两句逐字保留；`git -C cloud diff AGENTS.md` 只包含追加，不含
   删除或改写。承载它的本文仍是 `proposed`，这段例外是否保留取决于 `ora-space/circus#135`。
   本步同样不产生代码或测试证据，因此不解除任何 fail-closed。
3. **与运行时解禁解耦地推进**：规则侧定案**不**解除任何 fail-closed——它是 D11-A 准入条件 A2，
   而不是 D11-B 的启用证据。凭据 clone 的解禁仍须满足
   [用户自助 HTTPS Git 凭据](20261010-user-managed-git-credentials.md) D11-B 与
   [受控 Node 的仓库凭据执行使用独立 OS 身份](../../node/repository/20261010-controlled-node-git-authentication-isolation.md)
   D14-B 的全部条件。完成条件：解禁那次变更单独验收，且能指出 D11-B／D14-B 每一项的直接证据。
   **本步尚未开始。**

保持不变的行为：原规则对部署私钥、对象存储凭据、服务账号令牌、SSH 私钥、团队共享凭据与测试密钥的
全部约束；日志与审计排除秘密的要求；`repository_credential_executor_unavailable` 的 fail-closed 行为。
从本修订生效开始改变的行为：Cloud **可以**在满足五个条件的前提下，经专用端点接收用户个人的
HTTPS Git Token 并在同请求内转交 Credential Service。
