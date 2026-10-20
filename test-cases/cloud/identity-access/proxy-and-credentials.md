# 代理与内部凭据

## Cloud Only Receives Gateway-Issued Short-Lived Credentials

- 风险：若浏览器提供的 `Authorization`、`X-Ora-User-Token`、`Cookie` 或 `X-Forwarded-*` 被原样转发，或
  内部 JWT 按 session 期限签发、用同一把密钥签两种 kind，Cloud 的 service/user 双凭据边界与 5 分钟上限
  就被 Gateway 绕过。
- 前置状态：Cloud 只信任 Gateway 的 service 公钥（`kind=service, role=gateway`）与 user 公钥
  （`kind=user`）；浏览器持有有效 session Cookie。
- 触发：带伪造内部 header 与转发 header 的请求；不带 Cookie 但带 `Authorization` 的请求；用 Cloud 验证器
  校验 Gateway 签出的两种凭据，并互换 kind 校验；时间前进超过凭据期限。
- 必须成立：伪造 header 被剥离，请求按 session 身份成功；无 Cookie 的请求为 `401 unauthenticated`；
  service JWT `role=gateway`、user JWT `caller` 等于 service `sub`、`source`/`sub`/`displayName` 来自
  session；两种凭据不能互换用途；期限等于配置值且过期后 Cloud 拒绝；一把私钥或重复 `kid` 在启动时被拒。
- 禁止结果：任何浏览器 header 到达 Cloud；凭据期限超过 5 分钟；service 与 user 共用私钥。
- 决策依据：ADR D1、D7，不变量 11、12。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
|---|---|---|
| 伪造 `Authorization`/`X-Ora-User-Token`/`X-Forwarded-*` 不影响结果；无 Cookie 为 `401` | `integration.TestGatewayLoginProxyLogoutAndCredentialBoundaries` | `Covered` |
| Cloud 验证器接受 Gateway 签发的两种凭据，caller 绑定成立，kind 不可互换，过期后拒绝 | `gateway.TestIssuedCredentialsSatisfyCloudVerification` | `Covered` |
| 同一私钥、重复 `kid`、超过 5 分钟的期限或空 service subject 使 Issuer 构造失败 | `gateway.TestNewIssuerRejectsSharedKeysAndLongLifetimes`、`gateway.TestConfigDefaultsAndSessionLifetimeBounds` | `Covered` |
| 私钥只接受 PKCS#8 Ed25519 PEM | `gateway.TestLoadPrivateKeyAcceptsOnlyPKCS8Ed25519` | `Covered` |
| 内部 JWT 不出现在 session 表、Cookie、重定向、响应 body 或日志 | 没有日志捕获或响应扫描测试 | `Missing` |

## Browser Authentication Uses Only The HttpOnly Session And Current User Probe

- 风险：前端把 service/user JWT、provider token 或模拟 token 放入 localStorage、sessionStorage、Zustand 或
  JavaScript header，会让脚本注入直接窃取长期认证材料，并形成与 Gateway session 不一致的第二事实来源。
- 前置状态：前端与 Gateway 对浏览器表现为同一公开 origin；Gateway 代理 `/api/v1/*` 并签发内部短期 JWT。
- 触发：未登录访问任意 Cloud 站内目标、已登录访问登录页、账号被停用、Gateway 暂时失败和用户退出。
- 必须成立：前端只用 `/api/v1/me` 判断状态；401 保存经过约束的当前路径并自动 POST `/auth/login`，随后用
  `location.replace` 跳转；403 停止重登并显示停用；5xx 停止循环并提供重试；有效会话直接返回目标；退出调用
  `/auth/logout`。Vite 只把 `/auth`、`/api`、`/healthz` 代理到 Gateway，不暴露 `/internal`。
- 禁止结果：前端持久化或注入任何认证 token；模拟 auth handler 响应登录；callback 新参数覆盖 attempt 的
  `return_to`；401/403/5xx 形成自动跳转循环。
- 决策依据：ADR D1、D7、D8，不变量 5、11—15。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
|---|---|---|
| 401 自动启动登录并恢复包含 query/hash 的站内目标；外部 URL 通过校验后用 replace 跳转 | `login-page.test.tsx`、`return-to.test.ts`、`dashboard-layout.test.tsx` | `Covered` |
| 已登录、403、登录 start 失败和暂时故障停止循环并提供正确恢复入口 | `login-page.test.tsx`、`dashboard-layout.test.tsx` | `Covered` |
| 共享客户端不注入 `Authorization`/`X-Ora-User-Token`，模拟业务 client 与真实认证分离 | `api-client.test.ts` | `Covered` |
| 开发代理只暴露 `/auth`、`/api`、`/healthz` 到 Gateway | `vite.config.ts` 为静态配置，尚无自动化配置断言 | `Partial` |

## Internal Routes And Cross-Site Mutations Never Reach Cloud

- 风险：Gateway 若代理 `/internal/v1/*`，Controller 内部契约就暴露给浏览器；改变状态的请求若不要求
  同源证明，持有 session Cookie 的浏览器会被第三方页面驱动执行 mutation。
- 前置状态：浏览器持有有效 session Cookie。
- 触发：`POST /internal/v1/access`；不带 `Origin`、带外来 `Origin`、只带 `Sec-Fetch-Site: same-origin`
  的 `POST /api/v1/...`；`GET /api/v1/...` 不带任何同源 header。
- 必须成立：内部路由为 `404 not_found`；无同源证明或外来 origin 的 mutation 为 `403 origin_forbidden`
  且不到达 Cloud；`Sec-Fetch-Site: same-origin` 被接受；`GET` 无需同源证明。
- 禁止结果：内部 API 响应经 Gateway 返回；跨站 mutation 到达 Cloud；同源 mutation 被误拒。
- 决策依据：ADR D1、D8，不变量 11。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
|---|---|---|
| `/internal/v1/*` 为 `404`；跨站或无证明 mutation 为 `403`；`Sec-Fetch-Site` 替代；`GET` 直接转发 | `integration.TestGatewayLoginProxyLogoutAndCredentialBoundaries` | `Covered` |
| `Origin` 精确匹配规则（scheme、host、子域、`null`） | `gateway.TestSameOriginRequiresExactOriginOrSameOriginFetchMetadata` | `Covered` |
| 请求体与代理响应大小上限生效 | 没有针对 64 KiB 请求体与 8 MiB 响应上限的测试 | `Missing` |

## Cloud Authorization Decisions Pass Through Unchanged

- 风险：有效 session 若被 Gateway 解释为“已授权”，或 Cloud 的 401/403 被改写，disabled user 与被移除的
  membership 就能继续访问。
- 前置状态：浏览器持有有效 session；Cloud 中该用户或其 tenant membership 随后被置为 `disabled`；另一个已登录用户尚不是该租户成员。
- 触发：经 Gateway 请求 `/api/v1/me` 或租户资源；管理员邀请和停用成员；成员重新外部登录。
- 必须成立：Cloud 基于当前数据库状态返回 `403 user_disabled` 或 `403 membership_required`，Gateway 原样透传；session 行不因授权拒绝而被吊销。租户创建者为 `admin`，邀请只授予普通 `member` 角色；停用在所有 Gateway 副本的下一次请求立即生效，重新登录不能恢复成员权限。
- 禁止结果：Gateway 缓存授权结果；把 Cloud 拒绝改写为成功或吊销 session。
- 决策依据：ADR D1，不变量 15。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
|---|---|---|
| disabled user 经 Gateway 得到 Cloud 的 `403 user_disabled` | `integration.TestGatewayLoginProxyLogoutAndCredentialBoundaries` | `Covered` |
| 创建者角色、非成员拒绝、邀请角色、成员停用后两个 Gateway 副本立即拒绝，重新登录不恢复权限 | `integration.TestGatewayTenantAuthorizationUsesCurrentMembership`（真实 HTTP、PostgreSQL、Cloud 与两个 Gateway；外部 provider 为受控测试服务） | `Covered` |
| membership 移除或 owner 不匹配的拒绝透传 | membership 停用已有经 Gateway 的直接证据；移除和 owner 路径仍由 Cloud 自身测试覆盖，尚无对应 Gateway 验证 | `Partial` |

## Authorized Event Streams Outlive Proxy Timeouts

- 风险：空间事件流（SSE）是无限期连接，若 Gateway 的 upstream timeout、server write timeout 或 8 MiB 响应体
  上限同样作用于它，流会在超时后被静默切断，前端进入无休止重连；若豁免被扩大到普通 bounded 响应，卡死的
  upstream 会永久占住连接。
- 前置状态：已登录用户经 Gateway 订阅 `GET /api/v1/tenants/{tid}/spaces/{sid}/events`，Cloud 以
  `text/event-stream` 应答；Gateway 配置了远短于流寿命的 upstream/write timeout。
- 触发：订阅后等待两个超时窗口之和以上，再提交一次空间更新；另外让 upstream 对普通请求永不应答。
- 必须成立：事件流在两个超时都过期后仍保持打开并投递后续事件；永不响应的普通请求仍被 upstream timeout
  切断并返回 `502 upstream_unavailable`。
- 禁止结果：事件流在超时后断开；豁免作用于未授权连接或 bounded 响应。
- 决策依据：ADR D10，不变量 17。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
|---|---|---|
| 已授权事件流超过 upstream 与 write timeout 后仍打开并投递事件；bounded 请求仍被超时切断 | `integration.TestGatewayRelaysEventStreamsBeyondTimeouts` | `Covered` |

## Cloud Outage Is A Stable Failure That Keeps Sessions

- 风险：Cloud 不可达时若 Gateway 吊销 session、返回不稳定错误或泄露上游细节，用户会在 Cloud 恢复后被
  迫重新登录，或看到内部信息。
- 前置状态：浏览器持有有效 session；Gateway 的 upstream 不可达。
- 触发：经该 Gateway 请求 `/api/v1/me`；再用同一 Cookie 经 upstream 正常的 Gateway 请求。
- 必须成立：故障 Gateway 返回 `502 upstream_unavailable`；同一 session 在健康 Gateway 上仍然 `200`。
- 禁止结果：session 被吊销或清除；上游错误文本进入响应。
- 决策依据：ADR D8，失败与恢复语义表。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
|---|---|---|
| upstream 拒绝连接时返回 `502 upstream_unavailable`，session 在其他副本仍有效 | `integration.TestGatewayProviderAndUpstreamFailuresKeepFactsStraight` | `Covered` |
| Gateway 签名失败返回 `503 credential_unavailable` 且不发请求给 Cloud | 没有签名失败的故障注入 | `Missing` |
