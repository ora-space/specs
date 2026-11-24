# 登录与会话：登录方式开关、IDaaS 认人、邀请链接、网页会话与 desktop 令牌

依据 [登录与会话根决策](../../../decisions/cloud/login/0-idaas-or-email-login-with-shared-web-and-desktop-sessions.md)（`approved`）。
下文的 D 编号与不变量编号都指这篇根决策。

证据读取的是 cloud `0ca0a9a0` 的 `v2-server/`（路径相对该目录）；“Exactly One Login Method Is Open” 的 `V2_LOGIN` 必填部分读取的是
cloud `85bc9a5f`（含 cloud#362）。IDaaS、邀请链接与移出成员已由 I2
ora-space/circus#1052（ora-space/cloud#399、ora-space/cloud#400、ora-space/cloud#402）实现，证据由 ora-space/circus#1292 登记，在那之前这些用例保持 Missing；开发版已有的行为引用现有测试。
desktop 令牌的三个用例（ora-space/circus#1084）读取的是合并后的 cloud `1635cc7d`（ora-space/cloud#432）。

## Exactly One Login Method Is Open

- 风险：内网部署忘了配置，开放了无密码的邮箱登录；部署版里还能经邮箱接口拿到会话。
- 前置状态：分别以不配 `V2_LOGIN`、`V2_LOGIN=email`、`V2_LOGIN=idaas` 启动 Cloud。
- 触发：启动；读取 `GET /auth/providers`；在部署版用 `provider=email` 调 `POST /auth/login`；用不支持的提供方登录。
- 必须成立：不配置或配错值时拒绝启动；`/auth/providers` 只返回当前的那一种；部署版拒绝邮箱登录（`400 unknown_provider`）；
  不支持的提供方一律拒绝。
- 禁止结果：没有 `V2_LOGIN` 时按某个默认值启动；部署版里存在不经 IDaaS 得到会话的路径。
- 验证义务与证据：**Partial** — `internal/server/server_test.go` 的 `TestLoginRejectsOtherProvidersAndForeignRedirects`；
  `deployment_test.go` 的 `TestLoginPreflightBeforeEffects`。
  已覆盖：不支持的提供方被拒绝，`returnTo` 只接受站内路径，`V2_LOGIN` 缺失或非法值时在接触数据目录前拒绝启动（当前构建亦拒绝
  `V2_LOGIN=idaas` 且不回落到 `email`）。仍缺：`/auth/providers` 只返回一种、部署版拒绝邮箱登录（证据由 ora-space/circus#1292 登记）。
- 决策依据：D1，不变量 1、2。

## The IDaaS Exchange Keeps Every Credential Out Of Storage And Logs

- 风险：回调被伪造或重放；client secret、授权码、access token 进了数据库、日志或浏览器；IDaaS 的异常响应被当成登录成功。
- 前置状态：部署版配置了 IDaaS；一个替身 IDaaS 可以返回成功、OAuth 错误、遗留 `errorCode`、超过 64 KiB 的响应、缺
  `uuid` 的 userinfo、429、5xx 和超时。
- 触发：正常登录；用不匹配的 `state` 或缺失的尝试 Cookie 回调；同一次尝试回调两次；替身返回上述各种异常；发起登录的
  `Origin` 不是公开地址。
- 必须成立：只有 Cookie 与 `state` 属于同一次、未过期、未用过的尝试才继续；换 token 用 `client_secret_post`，不发
  PKCE；明确拒绝和非法响应返回 `401 login_failed`，网络错误、429 和 5xx 返回 `503 login_unavailable`；不跟随重定向；
  回调地址只取自 `V2_PUBLIC_URL`；成功或失败都清掉尝试 Cookie。
- 禁止结果：授权码、access token、client secret 或完整 userinfo 出现在数据库、日志或应答里；异常响应建出账号或会话。
- 验证义务与证据：**Missing** — 已由 I2 ora-space/circus#1052 实现，证据由 ora-space/circus#1292 登记。
- 决策依据：D2，不变量 6。

## An IDaaS Account Is Identified Only By Its UUID

- 风险：同一个人登录两次得到两个账号；显示名或其他字段变化后被对成另一个人；停用的账号被复活。
- 前置状态：部署版；一个已登录过的员工；一个已停用的账号。
- 触发：同一个 `uuid` 再次登录，显示名变了，`uuid` 两端带空白；新的 `uuid` 第一次登录；已停用账号的 `uuid` 登录。
- 必须成立：同一个 `uuid`（去掉首尾空白）总是同一个账号，显示名随登录刷新；新 `uuid` 建出一个账号；停用的账号返回
  `403`。
- 禁止结果：按显示名、邮箱或工号匹配或合并账号；为没登录过的人预先建出账号；停用账号重新可用。
- 验证义务与证据：**Missing** — 已由 I2 ora-space/circus#1052 实现，证据由 ora-space/circus#1292 登记。
- 决策依据：D3，不变量 3。

## Logging In Grants No Team Membership

- 风险：任何能登录的人都看到团队数据；开发版里没被邀请的邮箱也能进来。
- 前置状态：部署版一个从未被邀请的员工；开发版一个未预设、未邀请的邮箱和一个被邀请的邮箱。
- 触发：部署版员工第一次登录后读取团队列表和团队资源；开发版两个邮箱分别登录。
- 必须成立：部署版员工登录成功、得到账号，但团队列表为空，团队资源不可见；开发版未邀请的邮箱返回
  `403 email_not_invited` 且不建账号，被邀请的邮箱登录后看到团队。
- 禁止结果：登录本身让任何账号成为团队成员。
- 验证义务与证据：**Partial** — `internal/server/members_test.go` 的 `TestEmailLoginAndOwnerInvite`；
  `internal/store/invite_test.go` 的 `TestInviteMemberRequiresOwnerAndCreatesTheAccount`。已覆盖：开发版未邀请的邮箱被拒绝且
  不建账号，只有管理员能邀请，被邀请的邮箱登录后看到团队。仍缺：部署版登录后没有团队成员身份。
- 决策依据：D4，不变量 4。

## An Invite Link Admits One Account Once

- 风险：一条链接被转发后多人加入；过期或吊销的链接仍然有效；打开链接（`GET`）就被加入团队；令牌明文落库。
- 前置状态：部署版；管理员生成了一条带小队的链接，另有一条已过期、一条已吊销的链接。
- 触发：员工 A 打开链接并点“加入”；员工 B 再用同一条链接；两人同时点“加入”；已是成员的人打开一条新链接；用过期、
  吊销的链接；只 `GET` 链接页不点加入；用跨站 `Origin` 发“加入”；非管理员生成或吊销链接。
- 必须成立：A 成为团队成员并加入所选小队，链接记下使用人；B 和并发中的后到者被拒绝，提示链接已失效；已是成员的人直接
  进入团队、链接不消耗；过期和吊销的链接被拒绝；`GET` 不改变状态；跨站 `POST` 被拒绝；只有管理员能生成和吊销；
  数据库只有令牌的 SHA-256。
- 禁止结果：一条链接让两个账号加入；过期、吊销或用过的链接让人加入；角色高于成员。
- 验证义务与证据：**Missing** — 已由 I2 ora-space/circus#1052 实现，证据由 ora-space/circus#1292 登记。
- 决策依据：D4，不变量 4、5、6。

## The First Administrator Comes From A Machine-Local Invite

- 风险：部署版没有可用的管理员，团队无法使用；或者不在部署机器上的人也能拿到管理员身份。
- 前置状态：部署版的空库。
- 触发：在部署机器上运行管理员链接命令；第一个人打开链接加入；再次使用这条链接；24 小时后使用另一条未用的管理员链接。
- 必须成立：命令打印一条一次性、24 小时有效、角色为管理员的链接；第一个使用者成为管理员；链接用过或过期后不能再用；
  没有任何 HTTP 接口能生成管理员链接。
- 禁止结果：经网页或 API 生成管理员链接；同一条管理员链接让两个人成为管理员。
- 验证义务与证据：**Missing** — 已由 I2 ora-space/circus#1052 实现，证据由 ora-space/circus#1292 登记。
- 决策依据：D4，不变量 4、5。

## Removing A Member Ends Access But Keeps History

- 风险：离职的人移不出团队，或者移出后仍能读到团队数据；移出时删掉了历史；团队被移得没有管理员。
- 前置状态：一个团队有两个管理员和一个普通成员，成员在一个小队里、建过任务和评论。
- 触发：管理员移出普通成员，随后该成员带着原来的会话访问团队资源；管理员依次移出另一个管理员和自己；非管理员尝试移出别人。
- 必须成立：被移出的人下一个请求起看不到团队，小队编组结束，他的任务、评论和小队历史保留，账号仍能登录；最后一个
  管理员不能被移出；非管理员不能移出别人。
- 禁止结果：移出后仍能访问团队；历史被删除；团队没有管理员。
- 验证义务与证据：**Missing** — 已由 I2 ora-space/circus#1052 实现，证据由 ora-space/circus#1292 登记。
- 决策依据：D4。

## A Web Session Is Hashed, Expires After Seven Days And Logs Out Alone

- 风险：会话令牌明文落库，数据库泄露即可冒充；会话因使用被无限延长；登出影响别的会话。
- 前置状态：同一个账号在两个浏览器各有一个网页会话，另有一个 desktop 令牌。
- 触发：在一个浏览器登出；把时钟推到登录后第 7 天；部署版检查 Cookie 属性；跨站 `Origin` 发起登录或登出。
- 必须成立：数据库只存令牌的 SHA-256；登出只删除这一个会话，另一个浏览器和 desktop 令牌不受影响；从登录起 7 天后过期，
  使用不延长；部署版 Cookie 带 `Secure`、`HttpOnly`、`SameSite=Lax`；跨站的登录和登出被拒绝。
- 禁止结果：明文令牌落库；过期会话仍能使用；登出一处、处处失效。
- 验证义务与证据：**Partial** — `internal/server/server_test.go` 的 `TestLoginSessionLogout`。已覆盖：登录后能读到自己和团队，
  再次登录不产生重复账号或成员，登出后会话失效。仍缺：只存哈希、7 天绝对过期、登出只影响本会话、`Secure` 与 `Origin`
  校验。
- 决策依据：D5，不变量 6。

## A Desktop Token Is Issued Only Through An Approved Loopback Exchange

- 风险：已登录的浏览器被诱导悄悄给 desktop 签发 30 天令牌；授权码被截获后换成令牌；回调被指向其他主机。
- 前置状态：浏览器里有网页会话；desktop 在回环地址上监听。
- 触发：打开 desktop 登录页但不点“允许”；点“允许”后用正确的 verifier 和回调地址换令牌；重复使用同一个授权码；
  用错误的 verifier 或不同的回调地址；回调地址是非回环主机或自定义 scheme；授权码超过 2 分钟后使用。
- 必须成立：不点“允许”就没有授权码；授权码只能用一次，三项都匹配才能换成令牌，否则作废；回调地址只接受回环地址；
  2 分钟后授权码失效；数据库只存令牌的 SHA-256，令牌 30 天不用即过期。
- 禁止结果：未经确认签发令牌；同一个授权码换出两个令牌；令牌发往非回环地址。
- 验证义务与证据：**Partial** — `internal/server/desktop_login_test.go` 的 `TestDesktopLoginIssuesATokenForTheApprovingAccount`、
  `TestDesktopCodeIsSingleUseAndBound`、`TestDesktopApprovalAcceptsOnlyLoopbackAndS256`、`TestDesktopApprovalNeedsTheWebSession`；
  `desktop_login_internal_test.go` 的 `TestDesktopCodesExpireAfterTwoMinutes`、`TestDesktopCodesStayBounded`；
  `internal/store/desktop_tokens_test.go` 的 `TestDesktopTokenExpiresAfterThirtyIdleDays`；`frontend-v2` 的
  `src/features/desktop-authorize/page.test.tsx`（「允许」之前不调用批准接口）。已覆盖：只有网页会话、本站 `Origin` 且不带
  `Bearer` 的批准才产生授权码；授权码第一次兑换即作废，verifier 或回调地址不符一律 `invalid_grant`；回调地址只接受带端口和
  路径的 `127.0.0.1` / `[::1]`，拒绝 `localhost`、自定义 scheme、查询串、fragment、用户信息和其他主机；2 分钟后授权码失效；
  令牌 30 天不用过期、使用后顺延、过期后不复活。仍缺：没有测试直接断言数据库只存令牌的 SHA-256；“一小时内不再写库”没有断言。
- 决策依据：D6，不变量 6、9。

## A Declined Desktop Sign-In Only Returns To The Loopback

- 风险：确认页的「取消」被当成开放跳转，把浏览器送到任意地址；取消路径被跨站页面或 desktop 令牌调用；取消也签出授权码。
- 前置状态：浏览器里有网页会话；desktop 在回环地址上等回调。
- 触发：在确认页点「取消」（`POST /auth/desktop/deny`）；用非回环、带 fragment、`localhost` 的回调地址或空 `state` 调它；
  不带网页会话、带别的 `Origin`、或带 `Bearer` 调它。
- 必须成立：只接受网页会话且 `Origin` 是本站；回调地址与批准用同一条回环校验；回答是回调地址加 `error=access_denied` 和
  原样的 `state`；不签发授权码或令牌。
- 禁止结果：跳往非回环地址；无会话、跨站或带 `Bearer` 的调用成功；取消后存在可兑换的授权码。
- 验证义务与证据：**Partial** — `internal/server/desktop_login_test.go` 的 `TestDesktopDenialReturnsToTheLoopback`；`frontend-v2`
  的 `src/features/desktop-authorize/page.test.tsx`（「取消」只带 `redirectUri` 和 `state` 调 `deny` 并跳回；服务器拒绝时页面停住）；
  desktop 一侧的 Windows 11 真机实测（ora-space/circus#1379：确认页「取消」后 desktop 显示已取消、不写条目）。已覆盖：回答恰好是
  回调地址加 `error=access_denied` 和原样的 `state`、不带授权码；非回环、`localhost`、带空 fragment 的回调地址与空 `state` 被拒绝；
  无会话 `401`、带 `Bearer` `403 web_session_required`。仍缺：`deny` 本身的跨站 `Origin` 用例（同一个会话路由包装在批准接口上有
  测试）。
- 决策依据：D6；[desktop Cloud 连接第一版](../../../decisions/desktop/cloud-connection/20261121-first-version-loopback-login-and-host-held-token.md)
  的“继承与修改”（对 A6 D6 Cloud 契约的新增）与 D2。

## Web Session And Desktop Token Resolve To The Same Account And Lose Access Together

- 风险：desktop 令牌拿到和网页不同的身份或更大的权限；账号停用后某种会话还能用。
- 前置状态：同一个人有一个网页会话和一个 desktop 令牌，是团队成员。
- 触发：分别用 Cookie 和 Bearer 读取自己的身份、团队和资源；同一个请求同时带 Cookie 和另一个账号的 Bearer；停用或
  删除这个账号后再用两种会话各发一个请求；desktop 登出后再用网页会话。
- 必须成立：两种会话读到同一个账号、同样的团队和权限；带 Bearer 时只按令牌判定；账号停用或删除后，两种会话的下一个
  请求都返回 `401`；desktop 登出只吊销自己的令牌。
- 禁止结果：desktop 专用权限；停用后任一会话仍能访问；desktop 登出让网页会话失效。
- 验证义务与证据：**Partial** — `internal/server/desktop_login_test.go` 的 `TestDesktopLoginIssuesATokenForTheApprovingAccount`、
  `TestBearerHeaderAloneDecidesTheCaller`、`TestDesktopLogoutAndAccountState`；`internal/server/session_test.go` 的
  `TestDisablingAnAccountEndsItsSession`；`internal/store/desktop_tokens_test.go` 的 `TestDesktopTokenIsItsOwnKindOfCredential`、
  `TestDesktopTokenFollowsTheAccount`。已覆盖：令牌与批准它的网页会话是同一个账号，用令牌能读到团队列表；带 `Bearer` 时只按令牌
  判定（别人的 Cookie 不能顶替，坏令牌不回落到 Cookie）；desktop 登出只吊销自己的令牌，网页登出不影响 desktop 令牌；账号停用后
  desktop 令牌和网页会话的下一个请求都是 `401`，停用前批准的授权码也换不出令牌。仍缺：账号删除后 desktop 令牌失效的用例；
  两种会话读到的团队列表相同、以 `Bearer` 访问团队资源与权限的逐项对照。
- 决策依据：D5、D6，不变量 7、8。
