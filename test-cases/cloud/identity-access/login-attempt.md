# 登录尝试与 provider 交换

## Login Start Requires Same-Origin Proof And Writes One Bound Attempt

- 风险：login start 会无条件写入一行 attempt。若它能被跨站导航、`<img>`、预取或无同源证明的请求触发，
  任何页面都能让受害者浏览器发起登录 CSRF，或无界灌入 `gateway_login_attempts`；若 `return_to` 未在
  写入前规范化，callback 会把用户重定向到攻击者控制的 origin。
- 前置状态：Gateway 已配置 public origin、至少一个 provider 与限流阈值；数据库中没有该浏览器的 attempt。
- 触发：浏览器向 `POST /auth/login` 提交 `{provider, returnTo}`，分别带正确 `Origin`、错误/缺失 `Origin`、
  各种形式的 `returnTo`、未知 provider、未知字段，或超过限流阈值连续发起。
- 必须成立：只有同源、JSON、限流内的请求写入一行 attempt 并返回 `authorizationUrl`；attempt Cookie 为
  `SameSite=Lax`、`HttpOnly`、`Path=/auth/callback`、`Max-Age` 等于 attempt 有效期；`returnTo` 被规范化为
  同源相对路径或在写入前被拒绝；authorize URL 携带 `state` 和固定 `redirect_uri`，并遵守所选 provider
  的固定 scope（GitHub 不请求 scope 但携带 S256 `code_challenge`；IDaaS 只发送 2.0 普通模式必填参数
  `client_id`/`response_type`/`redirect_uri`/`scope=base.profile`/`state`，不发送 `display` 或 PKCE）。
- 禁止结果：`GET /auth/login` 存在；无同源证明或被限流的请求写入 attempt；绝对、`//`、`/\` 或带反斜杠/
  控制字符的 `returnTo` 被保存；attempt secret 或 `state` 明文进入数据库。
- 决策依据：ADR D4、D8，不变量 6、13。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
|---|---|---|
| 无 `Origin`、外来 `Origin` 或 `GET` 的 start 被拒绝且不写 attempt；`Sec-Fetch-Site: same-origin` 可替代缺失的 `Origin` | `integration.TestGatewayLoginAttemptReplayStateAndConcurrentConsume`、`gateway.TestSameOriginRequiresExactOriginOrSameOriginFetchMetadata` | `Covered` |
| 绝对、`//`、`/\`、反斜杠、控制字符与超长 `returnTo` 在写入前失败 | `integration.TestGatewayLoginAttemptReplayStateAndConcurrentConsume`、`gateway.TestNormalizeReturnToRejectsEveryOffOriginForm` | `Covered` |
| 超过限流阈值的 start 返回 `429` 且 attempt 行数不增加 | `integration.TestGatewayProviderAndUpstreamFailuresKeepFactsStraight`、`gateway.TestRateLimiterBurstRefillAndBoundedKeys` | `Covered` |
| attempt Cookie 的 `SameSite=Lax`、`Path`、`HttpOnly`、`Max-Age` 与前缀策略 | `integration.TestGatewayLoginAttemptReplayStateAndConcurrentConsume`（startLogin）、`gateway.TestCookiePolicyEnforcesPrefixesScopeAndDatabaseBoundedMaxAge` | `Covered` |
| authorize URL 绑定 `state`、固定 callback，且 scope 不可由浏览器选择；GitHub 另绑 S256 challenge，IDaaS 不发送 PKCE/`display` | `github.TestAuthorizationURLBindsStateChallengeAndCallbackWithoutScopes`、`idaas.TestAuthorizationURLBindsDocumentedIDaaSParameters` | `Covered` |
| 未知 provider 与未知 JSON 字段被拒绝 | `integration.TestGatewayLoginAttemptReplayStateAndConcurrentConsume` | `Covered` |
| 数据库只出现 secret/state 的定长 digest，attempt secret 明文不出现 | `integration.TestGatewayLoginAttemptReplayStateAndConcurrentConsume` | `Covered` |

## Callback Consumes An Attempt At Most Once

- 风险：同一 attempt 被重放、被错误 `state` 或缺失 Cookie 的请求完成，或两个并发 callback 各自创建一个
  session，会让攻击者用注入的 code 登录受害者浏览器，或让一次授权产生多份凭据。
- 前置状态：一个未消费、未过期的 attempt，浏览器持有对应 attempt Cookie，provider 已为该 attempt 签发
  code。
- 触发：分别以缺失 Cookie、错误 `state`、正确参数、再次重放正确参数，以及两个同时到达的正确 callback
  请求 `GET /auth/callback/github`。
- 必须成立：缺失 Cookie 与错误 `state` 统一返回 `401 login_failed`；一次失败的 callback 清除 attempt
  Cookie，使同一 attempt 之后的合法回跳也失败，用户必须重新 start；正确 callback `303` 到 attempt 中保存
  的 `return_to`；重放返回 `401`；并发 callback 恰好一个 `303`、另一个 `401`，数据库中只多出一个有效
  session；`consumed_at` 与 session 在同一事务提交。
- 禁止结果：failure 语义区分“attempt 不存在”与“state 错误”；重放或并发产生第二个 session；`Location`
  取自请求参数而非 attempt 行；provider 调用发生在事务或行锁内。
- 决策依据：ADR D4、D8，不变量 6、14。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
|---|---|---|
| 缺失 attempt Cookie、错误 `state`、重放均为统一 `401`，且不产生 session | `integration.TestGatewayLoginAttemptReplayStateAndConcurrentConsume` | `Covered` |
| 数据库中的 attempt 已过期而浏览器 Cookie 仍有效时，callback 在兑换 provider code 前拒绝，清除 Cookie 且不创建 session | `integration.TestGatewayExpiredLoginAttemptRejectsCallbackBeforeProviderExchange` | `Covered` |
| 失败 callback 清除 attempt Cookie 并使原 attempt 不再可用 | `integration.TestGatewayLoginAttemptReplayStateAndConcurrentConsume` | `Covered` |
| 成功 callback 重定向到 attempt 保存的 `return_to`，attempt 被标记消费 | `integration.TestGatewayLoginProxyLogoutAndCredentialBoundaries` | `Covered` |
| provider 允许 code 重用时，两个并发 callback 只有一个创建 session | `integration.TestGatewayLoginAttemptReplayStateAndConcurrentConsume` | `Covered` |
| IDaaS 的错误 Cookie/state、重放和并发 callback 不产生第二个 session，并恢复原目标 | `integration.TestGatewayIDaaSLoginIdentityAndSessionLifetime` | `Covered` |
| provider exchange 成功但 session 事务失败时没有 session、返回失败而非推断成功 | 没有数据库提交失败的故障注入 | `Missing` |
| provider 调用不在数据库事务或行锁内 | 结构上由 `Login.Callback` 的调用顺序保证，没有直接检测锁持有的测试 | `Missing` |

## Provider Rejection Never Creates A Session

- 风险：provider 拒绝（错误 code、verifier 不匹配、用户文档非法）若被当作成功，或 provider 超时被当作
  拒绝而丢弃可重试的 attempt，会产生无效身份的 session 或错误的用户提示。
- 前置状态：有效 attempt 与 Cookie；GitHub 会校验 `code_verifier` 与 challenge、拒绝未签发的 code；
  IDaaS 按 `client_secret_post` 校验 client secret 与一次性 code，拒绝未签发的 code 以及超大或非法 JSON。
- 触发：callback 携带 provider 从未签发的 code；callback 落到一个 PKCE 派生密钥不同的 GitHub 副本；
  provider 返回错误对象、非 bearer token、非 200 用户响应、缺失稳定 ID、超过 64 KiB 或非法 JSON；
  provider 超时或 context 取消。
- 必须成立：上述拒绝均为 `401 login_failed`，attempt 保持未消费，没有新 session；超时与取消是基础设施
  错误（`503 login_unavailable`），与拒绝区分；access token 与完整用户文档不离开适配器。
- 禁止结果：任一拒绝路径创建 session；超时被归类为 `login_failed`；provider 错误文本、token 或 code 回显
  给浏览器。
- 决策依据：ADR D3、D4、D8，不变量 5、6。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
|---|---|---|
| 未签发的 code 与 GitHub PKCE verifier 不匹配都得到 `401`，attempt 未消费，session 数不变 | `integration.TestGatewayProviderAndUpstreamFailuresKeepFactsStraight` | `Covered` |
| GitHub 错误对象（含 HTTP 200）、非 bearer、用户端点非 200、非法/缺失 `id`、超大或非法 JSON 均为 `ErrProviderRejected` | `github.TestExchangeFallsBackToLoginAndRejectsBadProviderAnswers` | `Covered` |
| 超时与取消是基础设施错误而非 provider 拒绝 | `github.TestExchangeReportsInfrastructureFailuresDistinctly` | `Covered` |
| 经过完整 HTTP 路径时 provider 超时返回 `503 login_unavailable` 且 attempt 保持可用 | 只有适配器层证据，没有通过 Gateway HTTP 的超时验收 | `Partial` |
| exchange 表单携带 `code`、`code_verifier`、`client_secret`、`redirect_uri`，用户读取使用换得的 token | `github.TestExchangeUsesVerifierReadsNumericIDAndDiscardsToken` | `Covered` |
| IDaaS token 表单只携带 `grant_type`/`client_id`/`client_secret`/`code`，userinfo 表单只携带 `access_token`；OAuth `error`、遗留错误码、超大或非法 JSON 为 `ErrProviderRejected`，取消、超时与重定向按契约分类 | `idaas.TestExchangeUsesClientSecretPostAndReturnsCorporateIdentity`、`idaas.TestExchangeFallsBackToUUIDAndRejectsProviderAnswers`、`idaas.TestExchangeBoundsResponsesAndPropagatesCancellation`、`idaas.TestHTTPClientTimesOutAndRefusesProviderRedirects` | `Covered` |

## GitHub Identity Is Keyed By Numeric ID Within Cloud Field Limits

- 风险：用 login 或 email 做身份键会在改名后分裂或错误合并 Ora 用户；`displayName` 超过 Cloud 的 200
  字节上限会让 Cloud 返回 `401 invalid_identity`，把一次成功的外部登录变成无法解释的未认证。
- 前置状态：Cloud 中不存在该 GitHub 用户；provider 返回数字 `id`、`login`、`name`（可能超长或为空）。
- 触发：首次登录；provider 改变 `login`/`name` 后再次登录；provider 返回 255 字符或多字节超长 `name`。
- 必须成立：`user_identities` 中出现 `(github.com, <数字 id>)` 且仅一行；改名后仍映射同一 Cloud user；
  `displayName` 在离开 Authenticator 前按字节截断在 rune 边界；`name` 为空时使用 `login`。
- 禁止结果：以 login/email 作为 `subject`；超长 name 到达 Cloud；截断切断 UTF-8 序列；`users` 出现第二行。
- 决策依据：ADR D3，不变量 3、4。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
|---|---|---|
| 首次登录建立 `github.com/<id>` identity 并 JIT 创建用户 | `integration.TestGatewayLoginProxyLogoutAndCredentialBoundaries` | `Covered` |
| 改名后再次登录不产生第二个用户 | `integration.TestGatewayProviderAndUpstreamFailuresKeepFactsStraight` | `Covered` |
| 超长多字节 name 被截断到 200 字节的 rune 边界，登录与 `/api/v1/me` 成功 | `integration.TestGatewayProviderAndUpstreamFailuresKeepFactsStraight`、`gateway.TestNormalizeMatchesCloudByteLimits` | `Covered` |
| `source`/`subject` 超过 128/512 字节或为空时拒绝而非截断 | `gateway.TestNormalizeMatchesCloudByteLimits` | `Covered` |
| `name` 为空时回退到 `login`；数字 `id` 以十进制字符串作为 subject | `github.TestExchangeFallsBackToLoginAndRejectsBadProviderAnswers`、`github.TestExchangeUsesVerifierReadsNumericIDAndDiscardsToken` | `Covered` |

## Development Provider Sign-In And Its Loopback Boundary

- 风险：本地开发登录 provider 允许输入任意 `source`/`subject`，若它在生产 origin 上可用，或在省略
  `provider` 时被隐式选中，就在 D2 “Authenticator 不得伪造 `source/subject` 合并已有用户”之外打开了一个
  生产可用的身份伪造入口；伪造的 code 若被接受，同样会签发真实 session。
- 前置状态：`login.development_provider` 与 `public.development` 同时开启（loopback HTTP），Gateway 注册
  `dev` 与外部 provider 两个适配器。
- 触发：`GET /auth/providers`；省略 `provider` 的 `POST /auth/login`；选择 `dev` 的完整表单流；携带伪造
  code 的 `/auth/callback/dev`；生产 origin 上开启该 provider 的配置。
- 必须成立：provider 列表同时含 `dev` 与外部 provider，`default` 是外部 provider；省略 provider 的 start 永远
  指向外部 provider；表单仅同源提交，封装的 code 只在 5 分钟内、签名、challenge 匹配时有效；伪造 code 统一
  `401 login_failed` 且不创建 session；真实流程后 `/api/v1/me` 返回输入的显示名，`user_identities` 恰好落在
  输入的 `(source, subject)` 上；配置校验拒绝生产 origin 上的 `login.development_provider`。
- 禁止结果：生产配置开启 `dev`；省略 provider 时落到 `dev`；伪造 code 创建 session；provider 授予任何
  Cloud 内权限。
- 决策依据：ADR D2、D9，不变量 16。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
|---|---|---|
| 表单流程登录输入的典型身份，伪造 code 统一 `401 login_failed` 且不产生 session | `integration.TestGatewayDevelopmentProviderSignsInTypedIdentity` | `Covered` |
| `GET /auth/providers` 同时列出 `dev` 与外部 provider，`default` 为外部 provider，省略 provider 的 start 指向外部 provider | `integration.TestGatewayDevelopmentProviderSignsInTypedIdentity` | `Covered` |
| `devlogin` 表单仅同源提交、code 由进程内密钥 HMAC 封装且过期/篡改拒绝 | `devlogin.TestAuthenticator`（表单、封印与校验） | `Partial` |
| 开启 `login.development_provider` 只允许显式 development 的 loopback HTTP；HTTPS（含 loopback）、非 loopback 与未开启 development 均在配置阶段拒绝 | `gateway.TestDevelopmentProviderRequiresExplicitLoopbackHTTPOrigin` | `Covered` |

## Huawei IDaaS Identity Is Keyed Only By Corporate UUID

- 风险：把姓名、邮箱或工号作为身份键，或允许部署自由配置 source，会因字段变化、复用和环境漂移错误合并或
  拆分员工身份；把 userinfo 或 provider token 保存下来会扩大个人信息与凭据泄露面。
- 前置状态：Gateway 选择 `huawei-idaas`；IDaaS 返回顶层 `uuid`（对应员工工号而非工号明文，可能含文档示例中的首尾空白）、配置指定的
  显示名称字段以及其他个人字段；Cloud 中尚无该 identity。
- 触发：首次登录、显示名称变化后的再次登录、显示名称缺失、UUID 缺失或全空白，以及 token/userinfo 响应异常。
- 必须成立：identity 固定为 `(huawei-corp, trim 后的 uuid)`；文档示例 `uuid~dGVzdDE = ` 归一为
  `uuid~dGVzdDE =`；uuid 对应员工工号但非工号明文；显示名称缺失时回退 trim 后的 UUID；全空白或缺失 UUID 拒绝登录；首次 `/api/v1/me` 以
  JIT 方式创建用户，后续登录不覆盖用户显示名称；登录不创建 membership；code、verifier、access/refresh
  token、client secret、完整 userinfo、邮箱和工号不进入数据库、Cloud、JWT、日志或公开错误。
- 禁止结果：source 可配置漂移；按姓名、邮箱、工号或 `globalUserID` 合并；缺失或全空白 UUID 仍创建
  session；因首尾空白拒绝本可识别的 uuid；provider token 被用作 Browser Session 或发给 Cloud。
- 决策依据：ADR D3a、D4，不变量 2—6、15。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
|---|---|---|
| adapter 固定 `huawei-corp + uuid`，去掉首尾空白，显示名称字段缺失时回退 UUID，并拒绝缺失/全空白 UUID 与错误响应 | `idaas.TestExchangeUsesClientSecretPostAndReturnsCorporateIdentity`、`idaas.TestExchangeFallsBackToUUIDAndRejectsProviderAnswers` | `Covered` |
| 部署覆盖 HTTPS authorize/token/userinfo 后保留身份与表单协议；非法 URL 不泄漏配置值 | `idaas.TestDeploymentEndpointsUseTLSAndPreserveProtocol`、`idaas.TestEndpointOverridesRejectUnsafeURLs`、`gateway.TestLoadConfigReadsFileAndEnvironmentOverrides` | `Covered`（受控 TLS 服务与配置边界；真实企业认证仍需现场验收） |
| 全链路只建立一条 identity（假 IDaaS 返回带首尾空白的 uuid），显示名称保持首次 JIT 快照，登录不授予 membership | `integration.TestGatewayIDaaSLoginIdentityAndSessionLifetime` | `Covered` |
| access/refresh token 不成为 session token，session 只保存最小归一身份字段 | `integration.TestGatewayIDaaSLoginIdentityAndSessionLifetime` | `Covered` |
| 日志与公开错误不包含 code、token、refresh token、client secret、邮箱或工号 | 适配器错误不携带响应正文；尚无覆盖所有日志输出的捕获扫描 | `Partial` |
