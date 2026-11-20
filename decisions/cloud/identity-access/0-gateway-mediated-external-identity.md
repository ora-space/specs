---
status: implemented
---

# 外部身份由 Gateway 归一，Cloud 保持授权权威

> 历史决策：Gateway 验证、Cloud 授权以及 IDaaS `uuid` 登录键仍适用；下文忽略 `globalUserId` 的规则已由 [20260923-huawei-global-id-alias](20260923-huawei-global-id-alias.md) 修改。当前经验证的 `globalUserId` 用于关联天舟人员，冲突时拒绝自动合并。

Ora 的公开认证入口由独立 Gateway 承担，即 RFC 10017 §6.1 的 Backend-For-Frontend 模式：浏览器不持有
任何 OAuth 或内部凭据。Gateway 通过可插拔的 Authenticator 验证外部身份，把结果归一为稳定的 `source`、
`subject` 和非权威的 `displayName`，再用现有的短期内部 service/user JWT 调用 Cloud。Cloud 不直接接入
GitHub、密码或其他身份协议，也不持有浏览器会话；它继续以 PostgreSQL 中的 `users`、`user_identities`、
membership 和资源 owner 关系作为用户与授权的唯一权威。

首个适配器是 GitHub OAuth Authorization Code Flow。浏览器会话由 Gateway 存入 PostgreSQL，所有会话都有
非空的绝对过期时间：默认 30 天，配置上限 90 天，可提前退出或吊销。外部 provider token、OAuth code、PKCE
verifier 和浏览器 session token 明文均不持久化，也不进入 Cloud。

内网部署增加华为 IDaaS 2.0 Authorization Code（`client_secret_post`）适配器。未登录前端自动发起 Gateway Login Attempt，
由 IDaaS 跳转华为 W3 统一登录；成功后 Gateway 建立 12 小时绝对期限的本地会话并返回登录开始时保存的
Cloud 站内目标。身份键固定为 `huawei-corp + uuid`，provider token 与完整用户资料不持久化、不进入 Cloud。

本决策已实现并验证：`cmd/gateway`、`internal/gateway`、`internal/gateway/github` 与 migration
`internal/core/migrations/0005_gateway_auth.sql` 已落地，`internal/gateway/idaas` 随 ora-space/cloud#15
落地，运行文档为 `docs/gateway.md`，核心测试用例登记于 `test-cases/cloud/identity-access/`。兼容性边界：Cloud 已发布的公开 API、`user_identities`
唯一性、用户 JIT 创建语义、短期 JWT 上限和 tenant/owner 授权规则均未改变；变化只发生在 Cloud 之前新增
一个认证与会话边界。前端登录页面、UI 路由和交互不属于本决策。

## 问题与现有边界

Cloud 接受两类相互独立的内部凭据：`Authorization` 中的 service JWT 证明调用服务，`X-Ora-User-Token`
中的 user JWT 证明最终用户；公开 API 只允许 `role=gateway` 的服务调用，并要求 `user.caller == service.sub`。
内部 JWT 的有效期上限是 5 分钟，不能被改造成浏览器长期会话。

`user_identities(source, subject)` 已经表达“外部身份映射到 Ora 用户”的关系，并通过联合唯一约束保证并发
首次登录只创建一个用户。这个模型刻意不依赖协议名、用户名、展示名或 email。本决策补上此前缺失的部分：
谁负责完成外部登录、浏览器如何保持登录而不延长内部 JWT、会话如何保存/过期/吊销/清理、新增认证方式如何
避免复制会话与代理逻辑，以及各方在失败时保留什么事实。

## 术语与信任边界

- **Authenticator**：验证一种外部认证证明并返回 `VerifiedIdentity` 的适配器。只有适配器解释 provider
  code、token 和用户响应；它不能授予 Ora tenant、role 或 owner 权限。
- **VerifiedIdentity**：Gateway 内部的短生命周期值，包含稳定的 `source`、稳定的 `subject` 和可选
  `displayName`。它表示外部认证已成功，不表示 Cloud 用户启用、membership 有效或资源可访问。
- **Login Attempt**：一次有时限、一次性的外部登录尝试，绑定 provider、`state`、PKCE、浏览器和同源
  `return_to`。
- **Browser Session**：Gateway 接受浏览器请求时使用的长期、不透明凭据，权威状态在 PostgreSQL，与内部
  JWT 的生命周期分离。
- **Internal Credentials**：Gateway 为一次上游调用签发的 service JWT 和 user JWT。Cloud 只信任经过现有
  密钥、issuer、audience、kind、role 和 caller 校验的内部凭据。
- **Absolute Session Lifetime**：从 session 创建到 `expires_at` 的固定期限。请求活动不会滑动延长它。

## 决策

### D1：Gateway 是公开认证边界，Cloud 不直接实现外部认证协议

Gateway 是独立进程 `cmd/gateway`，只做配置、依赖装配、生命周期与退出；认证编排、provider 适配、会话策略
和 HTTP 边界在 `internal/gateway`。它可以与 Cloud 一起部署，也可以独立扩缩容；进程边界不因同仓库而消失。

Gateway 只承担：发起和完成外部登录；创建、解析、吊销浏览器会话；为通过会话校验的请求签发短期内部
凭据；移除浏览器提交的内部认证 header 并把 `/api/v1/*` 转发给固定配置的 Cloud upstream；对改变状态的
浏览器请求执行同源校验并返回稳定、脱敏的错误。它不转发 `/internal/v1/*`（该前缀没有路由），不接受
请求参数选择 upstream，不把客户端提供的 `Authorization`、`X-Ora-User-Token`、`Cookie` 或
`Forwarded`/`X-Forwarded-*` 传给 Cloud。Cloud 继续负责用户启用状态、identity 映射、JIT 用户创建、tenant
membership、role、owner 和所有业务访问判定；Cloud 的 401/403 原样透传，有效 session 不能绕过它们。

生产环境可以在受信 ingress 终止 TLS，但浏览器看到的公开 origin 和 OAuth callback 必须是 HTTPS。Gateway
使用显式配置的 `public.base_url`，callback 固定为 `<base_url>/auth/callback/<provider>`，不从 `Host`、
`Forwarded` 或 `X-Forwarded-*` 推导。只有显式 `public.development` 配置允许 `localhost`/`127.0.0.1`/`::1`
上的 HTTP，其他 host 上的 HTTP 使启动失败。

### D2：认证方式通过 `VerifiedIdentity` 收敛，会话与 Cloud 转发保持 provider-neutral

编排层依赖小型 `Authenticator` 接口：适配器只生成 provider 授权 URL（编排层提供 `state`、PKCE
challenge 和固定 callback URL）和用 code 加 verifier 交换出 `VerifiedIdentity`。编排层拥有 Login Attempt、
`state`、PKCE、`return_to`、失败映射和 session 创建；Session Store 只接受已验证的 `VerifiedIdentity`；
代理只接受已解析的 Session。HTTP handler 只做输入限制与协议翻译。

增加新的认证方式只需新增一个产生 `VerifiedIdentity` 的适配器及其配置和 callback 路径，不复制 session
表、Cookie 策略、内部 JWT 签发或代理。账户关联、MFA、组织准入和高风险操作的再认证需另行决策；
Authenticator 不能通过伪造新的 `source/subject` 合并已有 Ora 用户。

部署配置 `login.provider` 选择默认适配器；`POST /auth/login` 省略 provider 时使用该默认值，保留显式字段以
兼容既有调用方。命令只读取被选择 provider 的 secret：选择 IDaaS 不要求 GitHub secret，选择 GitHub 也不
要求 IDaaS client secret。所有副本必须共享 PostgreSQL、PKCE 派生密钥以及所选 provider 的 client 配置。

### D3：GitHub OAuth 是首个适配器，并使用 GitHub 数字用户 ID 作为稳定 subject

适配器使用 GitHub OAuth App 的 Authorization Code Flow，启用随机 `state` 与 PKCE `S256`。GitHub 在
authorize 请求携带 `code_challenge` 后要求 token exchange 携带匹配的 `code_verifier`，因此 PKCE 在 GitHub
上是被 provider 强制校验的保护。不请求任何 scope。callback 用 code 换取 access token，读取
authenticated-user 后立即丢弃该 token。

| 字段 | 取值 | 理由 |
| --- | --- | --- |
| `source` | `github.com`；GitHub Enterprise Server 配置为该实例 host | 稳定、可读的 provider 身份空间；不同 host 的数字 ID 不会混淆 |
| `subject` | 数字 `id` 的十进制字符串；缺失、非数字或非正使登录失败 | 数字 ID 不随 login 改名；login、name、email 都不能作身份键 |
| `displayName` | 非空 `name`，否则 `login`；按字节截断到 200，截断落在 rune 边界 | 只用于展示，可变化，不参与查找、合并或授权 |

字段长度以 Cloud identity contract 为准：`source` ≤ 128、`subject` ≤ 512、`displayName` ≤ 200 **字节**
（Cloud 用 Go `len` 度量）。GitHub 的 `name` 最长 255 字符，因此适配器结果离开编排层前必须截断，否则
Cloud 会返回用户无法理解的 `401 invalid_identity`。GitHub token、OAuth code、完整用户文档和 email 不写入
数据库、日志、错误、JWT 或浏览器。GitHub 组织准入若有需要是独立的授权策略，不塞进 `source`/`subject`。

### D3a：华为 IDaaS 使用 2.0 `client_secret_post` Authorization Code，并以 UUID 作为稳定 subject

下文三个路径是默认值。部署方可以分别配置 authorize、token、userinfo 的完整 HTTPS URL，
用于同一企业身份权威在不同环境中的端点差异；浏览器请求不能选取这些地址。覆盖值不得包含 URL
userinfo、query 或 fragment，参数仍由适配器按本节协议构造，token/userinfo 继续禁止跟随重定向。
应用凭据以挂载文件提供，地址和 Client ID 由运行时配置提供；更换它们不要求重建镜像，但需重新创建
Gateway 容器。此配置能力不改变 `huawei-corp`、UUID、scope、身份关联、会话撤销或 Cloud 授权规则。

内网部署通过 `login.provider=huawei-idaas` 显式选择适配器。Gateway 使用 IDaaS 2.0 Authorization Code
的 `client_secret_post` 机密客户端认证；PKCE 模式默认关闭，且面向无 secret 的公共客户端，因此不启用。
authorize 地址为 `/saaslogin1/oauth2/v1/authorize`，只发送 `client_id`、`response_type=code`、固定
callback、`scope=base.profile` 和随机 `state`。不发送 `display`（可选，页面已按分辨率自适应），也不发送
`code_challenge` / `code_challenge_method`（仅 PKCE 模式必填）。认证方法不匹配时 IDaaS 返回
`error=invalid_request`。

回调后以 `application/x-www-form-urlencoded` POST 调用 `/saaslogin1/oauth2/v1/token`，请求体仅包含
`grant_type=authorization_code`、`client_id`、`client_secret` 和 `code`；不发送 `Authorization: Basic`，
也不发送 PKCE-NULL 的 `code_verifier` 或 `redirect_uri`。IDaaS 2.0 示例把同一组字段写在 query，但把
`Content-Type: application/x-www-form-urlencoded` 列为必填，该头只约束 entity-body。Gateway 按 RFC 6749
§2.3.1 `client_secret_post` 与 GitHub 适配器把它们放在请求体，避免 secret 进入 URL 或访问日志。随后以
`access_token` 表单 POST `/saaslogin1/oauth2/v1/userinfo`，完成身份归一后立即丢弃 token。
授权码截获后的换 token 保护依赖机密客户端的 `client_secret`、IDaaS 对 code 的一次性消费，以及 Gateway
的 attempt Cookie 与 `state` 绑定。编排层仍派生 PKCE verifier 以满足 provider-neutral `Authenticator`
接口，但 IDaaS 适配器不得把它放到线上请求里。上线前须按 `docs/gateway.md` 上线检查以 beta 环境实测
v1 token/userinfo 接受请求体中的表单字段（query 为空），以及 userinfo 默认返回已申请的 `uuid`。

| 字段 | 取值 | 理由 |
| --- | --- | --- |
| `source` | 固定 `huawei-corp`，不可配置 | 避免环境或命名漂移把同一员工拆成多个身份空间 |
| `subject` | IDaaS 顶层 `uuid` 去掉首尾空白后的非空字符串 | UUID 是稳定企业身份键，由 IDaaS 分配并对应员工工号，而非工号明文或 W3 账号；文档成功示例含尾部空格，空白不承载区分度。§2.1 默认返回列表与 §2.4 是否必含 `uuid` 互相矛盾，上线前必须在控制台确认/申请该字段，不能改用 `globalUserID`。部署预置管理员（如 cloudctl bootstrap）必须使用真实 IDaaS uuid，切勿填写工号 |
| `displayName` | 配置指定的顶层字符串；缺失或空时回退 `uuid` | 只用于首次 JIT（Just-In-Time，首次访问即时创建）用户快照，不参与匹配或授权；强烈建议在控制台申请真实姓名（如 `userName`）并配置，避免前端展示 uuid 字符串 |

authorization code、verifier、access/refresh token、client secret 和完整 userinfo 只存在于当前请求内存；
userinfo 中除 `uuid` 和显示名称外的字段立即丢弃。IDaaS 明确拒绝、OAuth `error`（以及遗留 `errorCode`）、
超大或非法 JSON 响应、缺失 UUID 收敛为 `401 login_failed`；网络、超时、取消、429 与 5xx 收敛为
`503 login_unavailable`。响应有 64 KiB 上限，HTTP 调用传播取消、使用独立超时且禁止跟随重定向。

### D4：Login Attempt 使用 PostgreSQL 保存一次性、短期的登录关联事实

登录由浏览器向 `POST /auth/login` 发起同源 JSON 请求触发（`provider`、可选 `returnTo`），响应
`authorizationUrl` 由前端导航。该请求接受 D8 的同源校验并受限流约束，没有可被 `<img>`、预取或第三方
顶层导航触发的 `GET` 形式：start 会无条件写入一行 attempt，没有同源校验与限流就是登录 CSRF 和无界写入。

Gateway 为每次登录生成至少 256 bit 的随机 attempt secret 和独立随机 `state`。浏览器只在 attempt Cookie
中持有 attempt secret；PostgreSQL 只保存两者的 SHA-256 digest。PKCE verifier 由 attempt secret 和仅
Gateway 可读的密钥通过带域分离的 HMAC-SHA256 确定性派生，不落库；所有副本必须共享同一把派生密钥。

attempt Cookie 使用 `HttpOnly`、`SameSite=Lax`、`Path` 限定为 callback 路径、`Max-Age` 等于 attempt
有效期，生产 HTTPS 下 `Secure` 并使用 `__Secure-` 前缀。必须是 `Lax` 而非 `Strict`：provider 回跳是来自
`github.com` 的跨站顶层 `GET` 导航，`Strict` Cookie 不会随之发送。callback 无论成功还是失败都清除
attempt Cookie；因此一次失败或被注入的 callback 会结束这个 attempt，用户需重新发起登录，而不是让
attempt 悬空可用。

`gateway_login_attempts` 保存 `id`、唯一的 `secret_hash` 与 `state_hash`（定长 32 字节）、`provider`、
已验证的同源相对路径 `return_to`（数据库层也拒绝绝对、`//` 和 `/\` 形式）、`created_at`/`expires_at`
（有效期为分钟级配置，1 分钟到 1 小时）和 `consumed_at`。

callback 必须同时匹配 attempt Cookie、`state`、provider、未过期和未消费状态；这次预检不加锁。对 provider
的 HTTP 调用不在数据库事务或行锁内。exchange 成功后，Gateway 开启短事务 `FOR UPDATE` 锁定 attempt，再次
验证其未消费和未过期，并在同一事务中写入 `consumed_at` 与新 session；这两个写入只能一起提交。provider 已
消费 code 但事务失败时登录明确失败，不推断 session 已创建。重复、过期、已消费、错误 state、错误 attempt
secret、provider 拒绝对外统一为 `401 login_failed`（JSON，不重定向），不暴露 attempt 是否存在。
基础设施故障（数据库、provider 网络/超时）与之区分为 `503 login_unavailable`。

### D5：Browser Session 存在 PostgreSQL，数据库只保存不透明 token 的 digest

Session Store 以 PostgreSQL 为唯一权威存储，没有内存、SQLite 或仅签名 Cookie 的并行实现。创建 session
时生成至少 256 bit 的随机 token，明文只返回一次写入 Cookie，数据库只保存 SHA-256 digest，按唯一索引解析。

`gateway_sessions` 保存 `id`（审计用 UUID，不作凭据）、唯一 `token_hash`、与 Cloud contract 等长的
`source`/`subject`、可空且不超过 200 字节的 `display_name`、`created_at`、非空 `expires_at`（晚于
`created_at` 且不超过其后 90 天）、`revoked_at` 与 `revoked_reason`（同时为空或同时非空，reason 限于
`logout`/`identity_revoked`/`administrative`）。同一 identity 可以有多个并发 session；session 不外键关联
Cloud `users`，因为外部身份可能在首次 Cloud 请求前已登录，且 Gateway 不读写 Cloud 业务表。

Store 提供 Create、Resolve、Revoke、RevokeIdentity 和 Cleanup。Resolve 只在 `revoked_at IS NULL` 且
`expires_at > clock_timestamp()` 时成功，不续期；不存在、过期和已吊销统一表现为 `401 unauthenticated`。
Revoke 与 RevokeIdentity 幂等，并且只作用于尚未过期的 session：过期行保留其过期事实作为审计，不再获得
吊销标记。吊销保证后续新进入 Gateway 的请求失败，不撤回已经通过 admission 并正在 Cloud 执行的请求。

session TTL 是正数配置：GitHub 默认 30 天，华为 IDaaS 默认 12 小时，硬上限均为 90 天；显式配置仍可
覆盖默认值，零、负数或超过上限使启动失败。`created_at` 与
`expires_at` 由 PostgreSQL 时间确定。session Cookie 使用 `HttpOnly`、`SameSite=Lax`、`Path=/`、无
`Domain`，生产 HTTPS 下 `Secure` 且使用 `__Host-` 前缀，由浏览器强制上述属性并阻止同站其他 origin 覆盖；
它是持久 Cookie，`Max-Age` 等于数据库 `expires_at` 与当前时间之差。仅开发 loopback HTTP 可以省略前缀与
`Secure`。退出（`POST /auth/logout`，同源）只吊销当前 session 并清除 Cookie；RevokeIdentity 服务“退出
所有设备”和管理员吊销，其公开管理 API 不在本决策范围。

同一 identity 的 session 数量不设上限：每次成功登录都创建新 session，累积直到各自到期或被吊销，数据库
增长由绝对期限与 D6 的有界清理约束。

### D6：Gateway 与 Cloud 共用 PostgreSQL 集群，但数据库权限和表所有权分离

`gateway_login_attempts` 与 `gateway_sessions` 通过 `internal/core/migrations` 下的有序 forward migration
`0005_gateway_auth.sql` 创建，由单独授权的 `cloudctl migrate` 应用；Gateway 启动时复用 Cloud 的
`CheckSchema` 只校验全部 migration 的校验和，绝不执行 DDL。

运行时应使用独立的 Gateway 数据库角色，只授予上述两张表的 `SELECT/INSERT/UPDATE/DELETE` 与
`schema_migrations` 的 `SELECT`。Gateway 代码不查询 `users`、`user_identities`、membership、tenant、
project、workspace 或 operation 表；Cloud 也不依赖 session 表完成业务授权。共享集群是部署复用，不是
共享数据所有权。角色的创建与授权是部署步骤，不写入不可变的 migration 文件。

表约束在数据库层保证 digest 长度、identity 字段边界、`expires_at > created_at`、90 天/1 小时上限、
吊销 reason 与时间一致、`return_to` 形式，并为 token lookup、identity 吊销、过期清理建立索引。时间字段
使用 `timestamptz`，有效性与清理边界使用数据库时间。

过期或吊销 session 与过期 attempt 保留一个可配置审计窗口（默认 7 天）后由 Gateway 生命周期拥有的清理
goroutine 按批删除（默认每表每批 500 行，循环直到一批未满）。清理不删除尚未过期且未吊销的活跃 session，
不用无界 `DELETE` 阻塞生产表，失败只记录并等待下一轮。

### D7：有限期浏览器会话只换取短期内部凭据，不改变 Cloud 的 JWT 安全上限

每次代理已认证请求时，Gateway 先 Resolve 当前 session，再用两把按用途分离的 Ed25519 私钥签发：service
签名密钥签发 `kind=service`、`role=gateway` 的 service JWT，`sub` 是本副本稳定的服务身份；user 签名密钥
签发 `kind=user` 的 user JWT，`source`、`sub`、`displayName` 来自 session，`caller` 等于本次 service JWT
的 `sub`。必须是两把密钥：Cloud 按 `kid` 登记的 key purpose 选择公钥，一把私钥同时签两种 kind 会让
purpose 绑定失去意义，也无法分别轮换。

两种 JWT 的 issuer、audience、`kid`、kind、role、caller 和最长有效期服从 Cloud 现有验证规则；配置的
凭据期限默认 60 秒、上限 5 分钟，绝不按 browser session 的期限签发。内部 JWT 只存在于 Gateway 到 Cloud
的请求中，不写入 session 表、Cookie、重定向 URL、响应 body 或日志。两把签名私钥与 PKCE 派生密钥是三个
独立 secret，从文件读入进程内存，不存入 PostgreSQL；Cloud 只配置对应公钥及用途。多副本共享 session
数据，每个副本用自己的 service subject 和密钥签发配对的 JWT，因此 caller binding 天然成立。

### D8：浏览器安全依赖同源证明、CSRF 防护和严格 redirect，而不只依赖 Cookie 属性

认证路由与代理路由同源，不开放 credentialed cross-origin CORS。对改变状态的浏览器请求（login start、
logout 和代理的非 `GET/HEAD/OPTIONS` 请求），`Origin` 必须与配置的 public origin 完全一致；`Origin` 缺失
时接受 `Sec-Fetch-Site: same-origin` 作为等价证据；两者都缺失或不匹配返回 `403 origin_forbidden`。
`POST /auth/login` 还要求 `application/json` 且严格解码（未知字段、尾随内容、超过 4 KiB 均拒绝）。OAuth
callback 是跨站顶层导航的例外，但完整验证 Login Attempt、`state` 和固定 callback URL。GitHub 额外由
provider 强制 PKCE；IDaaS 额外以 `client_secret_post` 校验 client secret，且不向 provider 发送 PKCE 或
`display` 参数。

`return_to` 在写入 attempt 前规范化：以单个 `/` 开头，第二个字符不是 `/` 或 `\`，不含反斜杠、控制字符，
没有 scheme、host、userinfo；回调后只从可信 attempt 行读取，请求参数不能直接成为 `Location`。

前端以 `/api/v1/me` 作为唯一会话事实：401 保存当前站内路径并自动 `POST /auth/login`，随后用
`location.replace` 导航 authorization URL；403 表示用户已停用，不重复登录；5xx 停止自动循环并提供显式
重试。已有有效会话访问登录页直接返回目标页面。浏览器不在 localStorage、sessionStorage、Zustand 或模拟
handler 中保存 token；退出只调用 `POST /auth/logout` 吊销 Cloud 本地 session，不退出 W3/IDaaS SSO。

login start 与 callback 是未认证入口，按客户端传输地址限流（每进程令牌桶，键表有界），限流拒绝不写入
attempt。认证错误不回显 provider token、code、state、attempt secret、session token、SQL 或上游响应；结构化
日志记录 request ID、provider、阶段、稳定错误码和 session UUID，不记录上述秘密、完整 Cookie 或不必要的
个人信息。请求体、provider 响应（64 KiB）与代理响应（8 MiB）有大小上限，向 provider 与 Cloud 的调用有
可取消 context 与超时并关闭 response body。provider 暂时失败不创建 session；Cloud 暂时失败返回
`502 upstream_unavailable`，不吊销 session。以 `text/event-stream` 应答的已授权连接是例外（D10）。

### D9：本地开发登录 provider 只存在于 loopback development origin

`login.development_provider: true` 在 `login.provider` 旁边额外注册一个 `dev` Authenticator（`internal/gateway/devlogin`）：
Gateway 自己扮演外部 provider，浏览器被送到同源表单 `GET /auth/dev/authorize`（携带 `state` 与 `code_challenge`），
开发者输入任意 `source` / `subject` / `display_name`，表单把身份与 challenge 用进程内随机密钥的 HMAC-SHA256
封装成 5 分钟有效期的 code，`303` 回普通的 `/auth/callback/dev`。之后的 attempt、PKCE、session、Cookie 与凭据
签发与 GitHub/IDaaS 完全相同；`Exchange` 的任何失败都是 `ErrProviderRejected`。

这项能力看起来违反 D2 的“Authenticator 不能通过伪造新的 `source/subject` 合并已有 Ora 用户”（输入已有身份
即可以其登录），因此它被硬性约束在开发边界内，作为该规则的显式豁免而不是静默例外：

- `gateway.Config.Validate` 只在同时开启 `public.development`（loopback HTTP）时接受 `login.development_provider`，
  生产配置无法开启；`New` 额外拒绝非 loopback 的 `http://` origin。
- `dev` 永远不会被隐式选中：`POST /auth/login` 省略 `provider` 时用 `login.provider`（外部 provider），仅当
  `login.development_provider` 开启且 `login.provider` 留空时 `dev` 才是唯一选项。
- provider 不授予任何权限：Cloud 仍按 `(source, subject)` JIT 创建或解析用户，membership 与授权判定与
  GitHub/IDaaS 身份完全相同。
- 签名密钥只存在于进程内存，每次启动重新生成。

表单路由自身的错误响应（缺失 `state`/`code_challenge`、跨站提交、身份非法时重新渲染表单）是面向人的纯文本/HTML，
不走 JSON fault 形状；`/auth/callback/dev` 的失败仍统一为 fault，编排层不因 provider 而异。

### D10：已授权的 SSE 事件流豁免逐连接的超时与体积上限

Cloud 以 `text/event-stream` 应答的空间事件流是无限期流：Gateway 对该已授权连接解除 `cloud.timeout`、server
`write_timeout` 与 8 MiB 代理响应体上限，逐帧转发直到任一方关闭；限制只对这一个已授权连接放开，未授权或普通
bounded 响应仍受全部约束。上游不可达或超时仍为 `502 upstream_unavailable`。

## 失败与恢复语义

| 失败点 | 对外结果 | 持久事实与恢复 |
| --- | --- | --- |
| 创建 attempt 失败或被限流 | `503 login_unavailable` / `429 rate_limited`，不跳转 provider | 没有可用 attempt；可安全重试 start |
| provider 拒绝、state 不匹配、GitHub PKCE 不匹配、attempt Cookie 缺失、过期或已消费 | `401 login_failed`，attempt Cookie 被清除 | 不创建 session；attempt 保持未消费直到过期后清理 |
| provider exchange 成功、session 事务失败 | `401 login_failed` 或 `503 login_unavailable` | code 可能已不可重用，但没有 session；必须重新开始登录 |
| session Cookie 不存在、未知、过期或已吊销 | `401 unauthenticated`，Cookie 被清除 | 不泄露 token 是否曾存在；可重新登录 |
| Gateway 签名失败 | `503 credential_unavailable`，请求不发给 Cloud | session 保持有效；修复密钥配置后重试 |
| Cloud 网络或 5xx 失败 | `502 upstream_unavailable` 或透传 Cloud 响应 | 不吊销 session，不推断业务请求成功；幂等仍由 Cloud contract 决定 |
| 用户在 Cloud 被禁用或 membership 被移除 | 透传 Cloud 的 `403`/`404` | session 可以存在，但不能绕过当前业务状态 |
| logout 与业务请求并发 | logout 幂等 `204` | 已 admission 的请求可完成，后续 Resolve 必须失败 |

## 为什么不是这些替代方案

| 方案 | 优势 | 否决理由 |
| --- | --- | --- |
| Cloud 直接实现 GitHub/IDaaS 协议 | 少一个部署进程和一次代理 | 把外部协议、Cookie、重定向和业务授权耦合到权威服务，后续认证方式扩大 Cloud 攻击面并复制边界逻辑 |
| 浏览器直接持有 provider token 调 Cloud | Gateway 更薄 | Cloud 被迫信任 provider token 并联网验证；token 暴露给浏览器和业务 API，破坏 service/user 双凭据边界 |
| 把内部 user JWT 做成长期 token | 无需 session 查询 | 破坏 Cloud 的短期凭据上限，禁用、logout 和泄露处置无法及时生效，并把内部凭据暴露给浏览器 |
| 纯签名/加密 Cookie，不存 session | 无数据库读取 | 无法可靠 logout、按身份吊销或观察泄露影响；长期凭据只能等密钥全局轮换 |
| session 外键关联 `users` | 可在数据库中直接绑定 Ora user | Gateway 必须跨越业务表所有权，首次 JIT 创建前无法建 session，并把外部认证成功误读为 Cloud 授权成功 |
| 用 GitHub login 或 email 作 `subject` | 人类可读 | login 可改名，email 可缺失、变化或复用；会错误合并或分裂 durable identity |
| 用 IDaaS 姓名、邮箱或工号作 `subject` | 便于人工识别 | 这些字段可变、可复用或缺失；且 IDaaS 默认 userinfo 并不返回工号或姓名；只有 `huawei-corp + uuid` 能维持稳定且不随部署命名漂移 |
| 向 IDaaS 发送 PKCE S256 或 `display=page` | 与 GitHub 适配器对称，且 RFC 7636 可防授权码截获 | IDaaS 2.0 PKCE 默认关闭，且仅 PKCE 模式必填；Gateway 是机密客户端，用 `client_secret_post` 而非 PKCE-NULL。多发 PKCE 字段会按错误认证方式被拒。`display` 可选且页面已自适应 |
| 使用 `client_secret_basic` | 与部分 OAuth 服务器的 Basic 头一致 | 与 GitHub 表单换票形态不同，要新增 `Authorization` 头；IDaaS 在方法不匹配时返回 `error=invalid_request` |
| 使用 PKCE-NULL | 公共客户端不必持有 `client_secret` | Gateway 已持有 client secret；PKCE-NULL 示例不含 secret，会放弃机密客户端认证 |
| 把 token/`access_token` 放在 query | 与 IDaaS 2.0 示例逐字相同 | 示例同时把 `Content-Type: application/x-www-form-urlencoded` 列为必填，该头只约束 body。`client_secret` 与 `access_token` 会进入 URL、反向代理和访问日志；RFC 6749 `client_secret_post` 要求放在请求体 |
| 用默认返回的 `globalUserID`/`UID` 作 `subject` | 避开 §2.1 默认字段列表可能不含 uuid 的矛盾 | 身份键不能随文档版本漂移；缺 uuid 时在控制台申请该字段，而不是改写已选定的 `huawei-corp + uuid` |
| 持久化 GitHub access token | 减少 provider 调用 | 只需登录时读取身份；长期保存扩大泄露、加密、scope、撤销和轮换责任 |
| IDaaS refresh/revoke 或 Cloud 退出联动 W3 全局退出 | 可延长 provider 会话或一次退出全部系统 | Cloud 只拥有自己的 session；保存 refresh token 扩大凭据责任，全局退出会意外影响同浏览器中的其他华为系统 |
| 每种 provider 单独建 session 表和 handler | 单个适配器局部直观 | 复制安全策略、清理、Cookie、JWT 和代理，使不同认证方式的权限与失败语义不一致 |
| 单独部署 Redis session | 低延迟、天然 TTL | 唯一权威持久化是 PostgreSQL；PostgreSQL 已能表达有限 TTL、即时吊销和清理审计，不值得新增运维与恢复边界 |
| login start 用 `GET` 链接或返回 `303` | 更接近传统 OAuth 入口 | `GET` 可被跨站导航、预取和图片触发，无法要求同源证明；fetch 跟随到 provider 的 `303` 会被 CORS 打断。同源 JSON `POST` 返回 URL 由前端导航两者都避免 |
| callback 失败重定向到前端错误页 | 用户体验更顺 | 需要在本决策发明前端路由；JSON `401 login_failed` 稳定可测，前端接入时可增加失败跳转配置 |

## 不变量

1. 只有 Gateway 解释外部认证协议；Cloud 不接收或验证 GitHub/IDaaS token、OAuth code、密码或浏览器 Cookie。
2. Authenticator 成功只产生 `VerifiedIdentity`，不能授予 Ora tenant、role、membership、owner 或业务权限。
3. 外部身份只由稳定的 `(source, subject)` 查找；`displayName`、login、name 和 email 永不参与身份合并或授权。
4. GitHub 的 `subject` 必须来自数字用户 ID，`source` 必须为 provider host（公共 GitHub 为 `github.com`）；
   华为 IDaaS 必须使用固定 `source=huawei-corp` 和顶层 `uuid` subject；`VerifiedIdentity` 的字段长度在
   离开 Authenticator 前已满足 Cloud 的 128/512/200 字节上限。
5. 原始 session token、attempt secret、OAuth code、provider access token 和 PKCE verifier 不得持久化、记录或返回给 Cloud。
6. Login Attempt 必须有时限、绑定浏览器与 provider、验证 `state`，并且最多成功创建一个 session。
   GitHub 的 token 交换由 provider 强制 PKCE；IDaaS 不发送 PKCE 参数，改由 `client_secret_post` 与一次性
   code 保护。login start 只接受经过同源校验和限流的 `POST`；attempt Cookie 只发送到 callback 路径，并在
   callback 结束时清除。
7. Browser Session 的有效性只由 PostgreSQL 当前状态和权威时间决定；未知、过期和已吊销统一为未认证。
8. 每个 session 的 `expires_at` 必须非空、晚于 `created_at` 且不超过创建时间 90 天；Resolve 不得滑动延长期限。
9. 同一 identity 可以有多个 session；session 不外键关联 Cloud user，也不证明该用户当前可访问任何资源。
10. Gateway 运行时只访问自己拥有的认证表，不读写 Cloud 业务表，不在启动时执行 DDL。
11. Gateway 必须移除浏览器提供的内部认证 header，只向固定 Cloud upstream 代理允许的公开 API，绝不代理内部 API。
12. Gateway 到 Cloud 的 service/user JWT 始终短期有效并满足现有 issuer、audience、kind、role、caller 与
    key-purpose 校验；service JWT 与 user JWT 由两把按 kind 分离的私钥签发。
13. 生产公开 origin 和 OAuth callback 必须使用 HTTPS；redirect URI 来自显式配置，不能由不受信请求 header 决定。
14. 外部 HTTP 调用不得发生在数据库事务或行锁内；ambiguous failure 不得被推断为登录或业务操作成功。
15. 有效 session 不能覆盖 Cloud 对 disabled user、membership、tenant 和 owner 的当前授权判定。
16. `dev` provider 只能与 `public.development`（loopback HTTP）同时开启，生产配置无法启用；它不会被
    `POST /auth/login` 的省略 provider 隐式选中，也不会授予任何 GitHub/IDaaS 身份之外的权限。
17. SSE 豁免只作用于单个已授权的 `text/event-stream` 连接；其余请求的超时、体积上限与失败语义不变。

## 核心测试用例

- [登录尝试与 provider 交换](../../../test-cases/cloud/identity-access/login-attempt.md)
- [浏览器会话](../../../test-cases/cloud/identity-access/browser-session.md)
- [代理与内部凭据](../../../test-cases/cloud/identity-access/proxy-and-credentials.md)

## 风险与代价

- 每个已认证请求增加一次 PostgreSQL session lookup。接受这项成本以换取即时 logout 和吊销；只有在保持
  数据库为权威、证明失效上限并另行决策后才能增加缓存。
- GitHub 的 30 天默认绝对期限仍给 Cookie 被盗留下较长暴露窗口；IDaaS 以 12 小时默认值缩短员工状态变化
  的暴露窗口，但不会主动感知 W3 账号变化。HttpOnly、Secure、`__Host-`、同源/CSRF、digest-only 存储、
  90 天硬上限和吊销降低但不能消除风险；正式支持高价值操作前应增加空闲期限、再认证和会话管理。
- Gateway 成为公开可用性的关键路径，需要独立 readiness、限流、超时、日志脱敏和密钥轮换演练。共享
  PostgreSQL 集群降低运维成本，但必须通过数据库角色防止横向读取业务数据。
- 限流是每进程的，多副本下的全局公平性不是目标；它保证 attempt 写入有界，不保证精确配额。
- 同一 identity 的 session 数量无上限；表体积由 30/90 天期限、start 限流和有界清理约束，若观察到异常
  累积，按 identity 限制并发 session 数是后续决策。
- GitHub 或 IDaaS 故障会阻止对应部署的新登录，但不使已有 session 或 Cloud 内部授权失效。

## 本决策未解决的问题

- provider callback 自身失败后的品牌化错误落地页；当前 start 失败由前端展示并显式重试，callback 失败仍返回
  稳定 JSON Fault，避免把不可信 provider 参数带回前端路由。
- 账户主动关联/解绑、多个 external identity 合并、误绑定恢复和 identity 管理审计。
- GitHub organization/team 准入、IDaaS 组织/群组授权、MFA、passkey、step-up authentication 和风险评分。
- 管理员 session 列表与吊销 API、用户“退出所有设备”公开接口和完整安全事件审计产品；`RevokeIdentity`
  已在 Store 层可用。
- session token 自动轮换、空闲过期、续期策略、设备信息以及高价值操作的重新认证策略。
- Cloud 是否在后续登录时同步 `users.display_name`：当前 JIT 只在创建时写入，user JWT 每次携带最新快照。
- IDaaS refresh token、token revoke、W3 全局退出、资料持续同步与自动租户授权。
- ingress、域名、证书和 secret manager 的具体产品选择；本文只固定公开 HTTPS 与可信配置边界。

## 参考

- Cloud 认证契约：`docs/authentication.md`；Gateway 运行文档：`docs/gateway.md`
- Cloud identity schema：`internal/core/migrations/0001_core.sql`；Gateway 表：`0005_gateway_auth.sql`
- OAuth 2.0 Security Best Current Practice：[RFC 9700](https://www.rfc-editor.org/rfc/rfc9700)
- OAuth 2.0 for Browser-Based Applications：[RFC 10017](https://www.rfc-editor.org/rfc/rfc10017)
- GitHub OAuth web application flow：<https://docs.github.com/en/apps/oauth-apps/building-oauth-apps/authorizing-oauth-apps>
