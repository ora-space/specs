---
status: proposed
---

# 内网原生 HTTP 与受信 HTTPS 浏览器认证分离，双端点仍绑定同一部署

建议首个 HTTP 交付使用两类明确配置的入口：Desktop 的原生 API 可在用户确认风险后走
内网 HTTP，系统浏览器认证始终走同一部署的受信 HTTPS 入口。网页登录、Cookie 和 provider
callback 保持 HTTPS；HTTPS 登录页不能保护之后 HTTP 传输的会话秘密，不宣称端到端安全。

本文为 `proposed`。双端点协议与“完全 HTTP 浏览器认证是否纳入首轮”仍待 owner 选择，
不构成已批准的范围缩减或 Cookie 例外。本文先给出推荐方案的完整行为，替代方案单独比较。
仅涉及原生 HTTP/浏览器认证接缝；不扩展原生账号权限、设备管理或同步，也不改变本地使用资格。

## 问题与约束

Desktop 允许用户选择内网 HTTP，并不自动使 Gateway 浏览器认证或第三方 provider 支持它。
现有 Gateway 的 production public origin 是 HTTPS，网页登录用 Secure/HttpOnly Cookie，
development 例外只限 loopback。把该开关开放给任意内网同时会耦合 Cookie 降级与 dev provider，
不是一个可以仅靠客户端风险提示解决的变更。

浏览器 Secure Cookie 不会因此可用于普通 HTTP，Cookie 也不按端口隔离。
[RFC 6265 §4.1.2.5、§8](https://www.rfc-editor.org/rfc/rfc6265#section-4.1.2.5)
描述了传输与 Cookie 的相关边界；这些规则不证明明文浏览器会话可安全使用。
provider redirect URI 的接受范围还取决于部署配置和该 provider，不能从 loopback 原生回调
例外推导成“任意远端 HTTP OAuth callback 都支持”。

## 继承与修改

直接前序是[部署绑定 Desktop 会话 ADR](https://github.com/ora-space/specs/blob/0eddd50e25ebb0e6408631b4653e4787f61e116c/decisions/cloud/identity-access/20261019-deployment-bound-desktop-sessions.md)，
本文细化其中 D7 的 HTTP/浏览器门禁，并承接同域
[轮换恢复提案](https://github.com/ora-space/specs/blob/0273e78667e7a56a4cab3f1af862ad6643193159/decisions/cloud/identity-access/20261019-desktop-session-rotation-and-recovery.md)
保留的 HTTP 例外边界：不改变其 D5 有限恢复规则。两份提案均待独立评审，轮换提案本身不为
本文提供已批准的 HTTP 传输许可。

| 前序约定 | 本提案处理 |
| --- | --- |
| [认证根决策](0-gateway-mediated-external-identity.md) 的生产浏览器 HTTPS、Cookie、同源和 provider callback | 推荐方案继承，不改全局 Cookie 或 OAuth 入口为 HTTP |
| 用户显式内网 HTTP，经风险确认并持续标识 | 保留原生 API/会话使用范围，部署显式启用；不把“仅可匿名探测”冒充完整 HTTP 支持 |
| TLS 和 deployment ID/公钥分别验证，首次 TOFU | 继承；两个端点分别验证，同 ID 换密钥不能算同一部署 |
| 交接绑定发起 server，回调不能改换票目标 | 具体化为原生端点与浏览器入口都进入事务绑定 |
| 本地始终独立、单 server/账号、无隐式同步 | 继承，部署配置失败只使 Cloud 连接不可用 |

## 决策提案

### D1：部署明确声明原生端点与浏览器入口

默认仍是单一可信 HTTPS 入口。新增显式生产配置，分别表达 `nativeApiBaseURL` 与
`browserAuthBaseURL`；前者可在原生 HTTP opt-in 下为 HTTP，后者在推荐方案必须为 HTTPS。
这两个名称是协议语义，不是已有配置键；实现前正式契约统一校验和生成消费端类型。

地址只从部署可信配置产生，禁止从 Host、Forwarded、请求体任意 URL 或相对重定向推导。
保留部署路径前缀，规范化后拒绝用户名密码、fragment 和无关 query。两个入口都必须映射到
同一部署 ID/签名身份及同一会话权威，不能只在显示名上相同。

部署配置同时要求明确的 HTTP 风险 opt-in，默认关闭；它与 development/dev provider、
普通浏览器 Secure Cookie 规则独立。给 HTTPS-only 部署增配 HTTP 地址不会自动生效。
不能仅凭私网 IP 或 DNS 名称推断网络可信，也不自动从 HTTPS 回退到 HTTP。

### D2：发现给出候选，信任验证决定能否使用

1. 宿主规范化用户输入的原生地址，展示协议/风险。HTTP 首次或地址变化须明确确认，
   之后持续标识 HTTP，提示会话凭据、交接信息和业务内容可能被窃听或篡改。
2. 不带旧资格发现候选入口。`browserAuthBaseURL` 仍是不可信输入：只允许正常 HTTPS URL，
   不接受本地文件、自定义执行 scheme、用户名密码或任意 query 作为浏览器认证入口。
3. 对浏览器入口验证系统 TLS 信任；若宿主为自签/私有 CA 固定指纹，系统浏览器仍需自行
   信任证书，失败时指导安装可信 CA/证书后重试，不绕过浏览器安全页。
4. 对两个端点分别做新鲜 nonce 签名挑战，校验同 deployment ID、公钥和协议上下文；
   签名响应覆盖规范化端点对与本次 nonce/有效期，防止把无关已签信息当新配置授权。
5. 首次 TOFU 展示同一部署指纹并确认，旧连接则匹配已记录身份；失败保留原配置，不发旧秘密。

同签名私钥能证明同一身份信任域，不能证明 HTTP 链路未被透明转发；攻击者仍能窃听或
篡改 HTTP 会话。双端点挑战不能把明文网络升级成 HTTPS。发现声明、地址重定向或
“HTTPS 浏览器入口已验证”任一单项都不能替代原生目标的身份/风险检查。

### D3：登录事务绑定端点对，换票目标始终固定

宿主事务绑定已验证 deployment ID、公钥、原生 API 端点、浏览器认证端点、state、
PKCE S256、loopback/deep link 回调目标和有效期。Gateway 的批准页显示部署、账号以及
“将授权的原生 HTTP 地址和风险”，不能只显示 HTTPS 页面的锁形图标。

外部 provider callback 只回配置的 HTTPS 浏览器入口，由 Gateway 完成现有 provider 验证。
浏览器批准仍有同源/CSRF 保护。Gateway 在共享权威存储把已验证账号绑定到同一原生事务，
交接码短时/一次性，并绑定上述端点对与身份。外部 provider code/token 不转给 Desktop。

回调 URL 仅携带原事务允许的交接信息，不得携长期资格或替换端点。Desktop 只向原事务
记录的、已确认的原生目标换票；此步若为 HTTP，秘密可能泄露，必须如实提示，不能因为
网页登录走 HTTPS 就隐藏风险。事务取消、超时、端点编辑或账号切换均使旧事务失效。

`browserAuthBaseURL` 不是给浏览器跨源 fetch 原生 HTTP API 的许可。系统浏览器页面不
访问该 API，也不把 Cookie 导出给宿主。登录页面只导航批准后的回调，业务 HTTP 请求由
宿主自己的客户端发送，不能为绕过 mixed-content 限制而全局放开浏览器安全策略。

### D4：原生 HTTP 不成为浏览器 Cookie 边界

| 入口/凭据 | 推荐方案行为 |
| --- | --- |
| HTTPS 浏览器登录及 provider callback | 现有 Secure、HttpOnly、SameSite、同源与 callback 路径限制保持 |
| HTTP 原生 API | 不设置/接受浏览器认证 Cookie，不能凭 Cookie 建立身份；原生 bearer 按会话权威校验 |
| 宿主网络客户端 | 不导入浏览器 Cookie jar，不转发 Cookie，不自动跨源跟随带资格重定向 |
| 内部 Gateway→Cloud | 固定 upstream、短期分用途内部 JWT；原生 HTTP opt-in 不修改内部 TLS/密钥政策 |

端口差异不是 Cookie 隔离证据；优先为浏览器使用独立 HTTPS 主机名，浏览器从不导航到
原生 HTTP 认证页面。实现必须测试同主机不同端口、相邻路径、同名 Cookie 注入及跨 origin
请求，不能假设另一个端口自动拥有另一份 Cookie。

用户点击 HTTP 风险确认不允许 Gateway 接受任意身份或打开开发 provider。
原生接口严格验证 bearer 用途、会话状态与当前授权，不从 Cookie、Origin 或用户选择的
账号 ID 推导身份。HTTP 请求体与资格不得进入反向代理访问日志或诊断。

### D5：失败保留本地与旧信任，不静默降级

| 失败 | 可观察行为 |
| --- | --- |
| 原生 HTTP 未由部署启用 | 连接不兼容，提示管理员启用受支持配置；不改 development |
| 无受信 HTTPS 浏览器入口 | 可以保留 HTTP 地址设置和本地工作，不能完成新的网页登录；明确部署所缺前提 |
| 宿主已固定证书但浏览器不信任 | 提示安装受信 CA/证书后重试，不伪称宿主确认已授权浏览器 |
| 两个端点部署 ID/公钥或签名端点绑定不符 | 停止交接，保留旧配置；旧资格不发送 |
| provider 拒绝配置 callback | 明确登录配置错误，不改为任意 HTTP callback 或回退另一 provider |
| 网络/5xx、迁移挑战失败 | 保留现场与旧信任，不自动开循环登录或将 HTTP 当修复 HTTPS 的默认路径 |

同身份地址迁移仍须分别验证新传输与部署身份，旧地址不必在线；同时变更浏览器/原生
端点对须取消旧事务并重新验证。不同身份按新 server 重新登录，不能仅凭 URL 对相似沿用资格。
HTTP 连接上会话被动失效或账号停用仍只限制云端能力，不能影响本地文件、Agent 或编辑。

### D6：完全 HTTP 浏览器认证是待选范围，不隐式生效

替代方案可考虑另建纯 HTTP 浏览器入口，但必须由 owner 明确选入，不能把本提案标 approved
后顺手删 Secure。若选入，至少需要独立生产 opt-in、专用不安全 Cookie 名称/作用域与清理、
严格配置 origin、同源/CSRF、浏览器风险确认、provider 对明确 HTTP callback 的支持验证，
以及 HTTPS 部署不退化的回归义务。HttpOnly/SameSite 和私网地址都不能防链路窃听或主动篡改。

该选项适应没有 HTTPS 的内网，但从浏览器认证开始就暴露身份劫持风险，且不能承诺所有
provider 接受它。推荐先批准 D1–D5 的双端点方案；纯 HTTP 浏览器是否纳入首轮由 owner
明确决定，若要求纳入则先把其完整状态机/异常/Cookie 生命周期补齐再批准该部分。
不能将“纯 HTTP 仍未定”改写为“内网 HTTP 原生访问已经取消”。

## 不变量

1. 原生 HTTP 是部署与用户分别显式选择，不是 HTTPS 出错后的自动降级。
2. 两个端点都经过独立传输检查和同部署签名身份校验，发现声明不直接授予信任。
3. 事务、交接码与换票绑定原端点对，回调或重定向不能替换目标。
4. HTTPS 网页登录不被宣传为保护之后的 HTTP 会话，签名也不替代加密。
5. Cookie/development/provider 的既有默认边界不因原生 HTTP opt-in 被全局放松。
6. 任意失败不锁本地，不清旧信任，不把未发送内容传给新 server/账号。

## 替代方案与代价

| 方案 | 收益与代价 |
| --- | --- |
| 原生 HTTP + 浏览器 HTTPS（推荐） | 保留网页登录安全默认，支持明文原生访问；但部署需受信浏览器入口与双端点身份绑定 |
| 完全 HTTP 浏览器认证 | 适应纯内网；需新 Cookie/CSRF/部署契约，外部 provider 有条件，主动攻击风险更大 |
| HTTP 仅匿名探测，登录后全走 HTTPS | 边界小，但不满足已批准的原生 HTTP 资格访问，不能当作等价交付 |
| 全局 development/关闭 TLS 校验 | 接入快，但扩大任意身份/Cookie/TLS 攻击面，不采用 |

## 兼容、风险与批准后落地

现有 HTTPS 配置和网页用户不变，新增配置默认关闭。迁移不能静默把既有原生 HTTPS 凭据
发到 HTTP：地址变更、传输风险及同身份检查全部通过才沿用仍有效会话。
没有签名身份/双端点能力的旧 Gateway 必须报告升级需要，不假装发现已足够。

身份挑战签名编码与端点元数据属于首次交接协议的正式契约，需与第一阶段协议评审保持
一致；本提案只约束必须签什么、验证什么，不声称任意编码互通。
签名密钥丢失/自动轮换、组织下发配置、自动迁移声明及多 server 并行仍在范围外。

owner 选择后按结论纠偏并转 approved，再维护核心用例；推荐方案需要双端点、同主机
多端口、错误身份/证书、HTTP 风险、浏览器/系统证书信任差异、provider callback 和迁移
失败的直接证据。第三阶段产品交付前不能只用配置单测代替真实系统浏览器与正式包验收。
