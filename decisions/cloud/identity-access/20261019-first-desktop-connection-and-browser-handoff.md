---
status: proposed
---

# 首次 Desktop 连接以签名认可 TLS 端点，经浏览器单次交接激活预备会话

建议 Gateway 用独立 Ed25519 部署密钥签名配置修订、端点公钥绑定和新鲜 nonce，宿主校验
实际 HTTPS 对端的公钥后再登录。宿主预存首次会话秘密，浏览器只批准绑定事务、返回一次
交接码，Gateway 原子激活其 digest；换票响应丢失不需要重新返回未知的原始秘密。

本文为 `proposed`，包括签名格式、TLS 端点绑定、60 秒身份证明、10 分钟登录事务与恢复/
取消语义，均待评审。已有交接码默认 2 分钟、服务端会话政策和本地独立不因本文重定。
第一阶段只开放系统信任 HTTPS 与 loopback；deep link、自签指纹交互与 HTTP 由后续阶段
协议批准后开放，不自动降级。本文不是现有 Gateway API 或产品已实现能力。

## 问题、继承与修改

直接依据为[部署绑定会话 ADR](https://github.com/ora-space/specs/blob/0eddd50e25ebb0e6408631b4653e4787f61e116c/decisions/cloud/identity-access/20261019-deployment-bound-desktop-sessions.md)
以及[轮换恢复提案](https://github.com/ora-space/specs/blob/ae8046ba9da7237454085f271fd77c35b1e50743/decisions/cloud/identity-access/20261019-desktop-session-rotation-and-recovery.md)。
后者解决已有资格的换代，尚未定义首次发现、签名与浏览器交接的 wire 边界。

| 前序约定 | 本提案处理 |
| --- | --- |
| [认证根 ADR](0-gateway-mediated-external-identity.md)：Gateway 验证外部身份、Cloud 当前授权、PG 会话权威 | 继承；不复制 provider、Cookie 或内部 JWT 的权威 |
| 部署身份为稳定 ID 与签名公钥，首次 TOFU、TLS 分别验证 | 具体化为受限 JWS 和部署配置认可的 TLS SPKI 集合 |
| 事务绑定 server、state、PKCE、回调与期限，交接单次消费 | 具体化为固定端点、字段、PG 状态机与安全取消 |
| 独立 Desktop 会话，长期秘密不进 URL/Cloud/明文配置 | 具体化为宿主预备初始秘密、服务端只激活 digest |
| 原有网页登录、provider callback、浏览器绝对 TTL 不变 | 继承；新增原生事务不重用外部 OAuth code/PKCE/state |
| 轮换、政策发布与已认证退出 | 按最终批准的轮换协议消费，本文不批准其中的新选择 |

## 决策提案

### D1：严格规范化目标，发现只产生未可信候选

所有路径均追加到用户确认的 base URL 路径前缀，不能由响应或回调任意替换目标。
v1 的 wire base URL 使用 HTTPS、ASCII DNS A-label 或规范化 IP、显式非默认端口及绝对
路径前缀：scheme/主机小写、默认 443 省略，IPv6 使用压缩表示并保留括号；路径大小写保持。
拒绝用户名密码、query、fragment、反斜线、zone ID、非标准整数/八进制 IPv4、尾点域名和
`.`/`..` 段。v1 路径前缀各段只用 ASCII unreserved 字符，不接受百分号编码或重复 `/`，
根以空前缀表达，其他去尾部 `/`。UI 展示最终地址再确认，不能有损转写后不告知。
这是避免不同 URL 库歧义的首版限制，不声称支持所有合法 URI 路径；更广编码另经协议演进。

`GET /auth/desktop/v1/metadata` 返回协议版本和候选原生/浏览器 base URL、公开部署键、
能力与配置修订，但未验签前不授予信任。不带旧资格探测，不跨源自动跟随重定向，拒绝
HTML/SPA 冒充元数据；旧 Gateway 缺能力只提示升级，不导出网页 Cookie 或改用长期内部 JWT。

原生与浏览器地址来自部署可信配置，不能从 Host/Forwarded/请求任意 URL 推导。
第一阶段二者均为 HTTPS；不同地址也必须在同一部署/配置修订下分别验证。
首版不自动改写别名地址；输入不属于已声明端点时展示差异并重新确认目标，不偷偷迁移。

### D2：受限 JWS，明确部署身份、域分离与验证字节

挑战为 `POST /auth/desktop/v1/discovery/challenge`，请求仅含客户端新生成的 `nonce`。
nonce 为 32 字节 CSPRNG、无 padding base64url；服务端不能按调用者给的证书公钥、URL 或
转发头制造任意端点认可。部署 ID 是稳定 UUID，签名私钥独立于内部 JWT、TLS 与 CA 密钥，
公钥为 `OKP/Ed25519` JWK，仅含公开字段，不含 `d`。

响应为 JWS Compact Serialization，受保护 header 固定 `alg=Ed25519`、
`typ=ora-desktop-discovery+jws`、`kid=SHA256(原始32字节部署公钥)` 的小写十六进制值。
按 [RFC 7515](https://www.rfc-editor.org/rfc/rfc7515) 验证收到的签名输入字节，不重序列化
JSON 再验签。JWK 形状依据 [RFC 8037](https://www.rfc-editor.org/rfc/rfc8037)；算法名采用
[RFC 9864](https://www.rfc-editor.org/rfc/rfc9864) 的明确 `Ed25519`，不协商降级到多义
`EdDSA`、`none` 或其他算法。库未支持时须升级/适配并有互通证据，不偷换已签 header。

header 不接受远程 `jku/x5u`、嵌入取键扩展、未知 `crit`、非编码载荷或非受保护 header。
header/载荷 JSON 均拒绝重复键、未知必需语义、无效 UTF-8 和额外 JSON 值；JWS 整体最多
32 KiB；除 D5 的 `POST /login/start` 外，普通原生 JSON 请求最多 4 KiB。
base URL/回调 URI 各自最多 1024 字节。KiB 为 1024 字节；请求上限按实际传输的
UTF-8 JSON body 字节数计算，包含字段名、引号、转义和分隔符，不按解码后的字段长度计算。
原生端点不接受压缩请求体，避免压缩前后预算歧义。

部署发布配置前必须以最长合法 nonce、时间、修订及该配置的全部端点、SPKI、能力和版本
字段检查序列化、base64url 编码和签名后的完整 JWS 不超过 32 KiB，并检查 D5 的 start
组合预算；不满足时拒绝发布该配置，不能发布后再截断字段或删减已声明能力。生成挑战
响应时再次检查实际 JWS 大小，超限返回有界配置故障，不签发不可用证明。宿主在信任确认
和准备登录材料前检查发现证明及 start 的可编码性，超限作为不兼容配置停止，不先接受
信任再进入必然失败的 start。

| 已签载荷 | 含义 |
| --- | --- |
| domain、protocolVersion | 固定 ora.desktop.discovery.v1 与版本 1，不能拿来当用户 JWT 或批准票据 |
| deploymentId、deploymentPublicKey | 稳定 ID 与 JWK；kid 必须对应该公钥，已有记录还必须匹配记录中的键 |
| nonce、issuedAt、expiresAt | 本次 nonce 与 PG 时间，建议有效 60 秒；时间用 RFC3339 UTC 秒格式，UI 按本地时间显示 |
| configurationRevision | PG 权威配置修订，uint64 十进制规范字符串，不接受溢出或前导零歧义 |
| nativeApiBaseURL、browserAuthBaseURL | 一次配置修订共同声明的规范化端点对 |
| nativeTlsSpkiSha256、browserTlsSpkiSha256 | 各入口被部署可信配置认可的端点密钥集合，不由请求填入 |
| capabilities、callbackKinds | 签名覆盖的协议能力、回调方式；第一阶段只声明实际交付的 loopback |
| serverVersion、clientChannels | 可解释升级方向的发行版本，以及允许的正式/开发客户端通道；通道不授予任何账号权限 |

版本1必需能力为 `discovery-jws-ed25519-spki-v1`、`desktop-handoff-prepared-v1` 与
`desktop-session-v1`；缺任一能力不开始交接。能力名必须对应实际交付协议，不能仅因
草案存在就声明支持。扩展所需能力必须协商，不能把未知必需语义静默忽略。

首次公钥自包含只证明持钥，不证明预期组织；用户明确 TOFU，可核对管理员指纹。
相同 ID 换公钥按新 server 展示并重登，不自动轮换信任。签名/解码/目标不符不覆盖旧记录。

### D3：认可实际 TLS 端点密钥，不把 nonce 当防转发证明

每个 SPKI 值为 `SHA256(DER SubjectPublicKeyInfo)` 的无 padding base64url。宿主先正常
校验 TLS 链/主机名，再把实际对端公钥与已签允许集合比较；不同 base URL 分别用独立 nonce
取得证明，二者 ID、公钥、配置修订和完整端点对必须一致。证明已过期或修订改变时重新探测。
每个携资格的新 TLS 连接均需检查实际对端键，不能只验证发现请求后换另一连接盲发秘密。

该绑定适应 TLS 在可信 ingress 终止：管理员预先将入口公钥登记到部署权威配置；
Gateway 不从客户端或未验证 header 学习认可集合。多副本读同一已发布配置修订，旧配置副本
不能签出声称当前的新修订。一个修订中的每个端点公钥集合建议最多 4 项，以容纳受控轮换，
不接受通配符/空集合/任意 CA 证书作为“端点公钥”。

换证书但公钥不变无需改部署身份；换 TLS 密钥须先发布新绑定、部署新证书，再收回旧绑定。
签名更新不能豁免系统浏览器证书信任，也不能替代后续自签指纹变化的用户确认。
签名键与端点键均须随部署受控备份；不能把不同签名身份或丢失会话权威数据的副本当作
可沿用旧会话的同一恢复。

nonce 拒绝重放，端点公钥认可用于排除用另一 TLS 密钥在线转发挑战的代理；这不是 TLS
exporter，不唯一绑定每条连接，也不能防持有受信端点私钥者或被攻破的可信 ingress。
可缓存的证明最长 60 秒，移除端点公钥并非对所有客户端瞬间生效；到缓存期限必须重验，
失败停止发送云端资格而保留本地。未选定的 HTTP 透明转发边界不由本 HTTPS 提案解决。

宿主从发挑战开始用本地单调经过时间限制 60 秒，匹配 nonce 并核对已签时间间隔；不靠
客户端墙钟漂移延长有效性。Gateway 受理登录事务仍以 PG 当前时间检查证明到期。
证书系统时钟验证失败按连接故障处理，不绕过 TLS 或阻止本地工作。

### D4：宿主先准备可恢复事务，再允许浏览器参与

宿主唯一协调器生成 attempt UUID `L`、独立随机 state、PKCE verifier、浏览器入场秘密
`B`、事务管理秘密 `M` 与初始会话秘密 `U`，每个秘密均为独立 32 字节 CSPRNG。
PKCE 采用 [RFC 7636](https://www.rfc-editor.org/rfc/rfc7636) 的 S256，无 padding base64url，
verifier 为 43 字符；原生 PKCE 与 Gateway 外部 provider 的材料完全分开。

先将材料、端点/部署身份/配置修订、回调 URI、当前选择 epoch 与期限可靠写系统安全存储的
PendingLogin 记录，再发 start；写失败不打开浏览器。未确认账号时记录以部署身份+L命名，
认证后原子转到部署身份+账号的正式 Current 记录，不以未知账号猜测归属。

loopback 先绑定 `127.0.0.1` 或 `[::1]` 的系统动态端口，路径含不可猜的事务分量；
仅允许 `http`、明确回环 IP、实际端口与固定路径，不接受 localhost、任意域名、userinfo、
fragment、附加 query、zone ID 或替换地址。参考 [RFC 8252](https://www.rfc-editor.org/rfc/rfc8252#section-7.3)。
Gateway 校验回调种类与规范形状，端口在创建时固定，后续不能替换；正式/development 包与
多实例事务所有权隔离，后续 deep link 必须先协商再按已批准 allowlist 开启。

### D5：start 登记 digest，提前分配无权限的会话 ID

所有端点位于已验证 base 前缀下的 `/auth/desktop/v1`。普通请求严格解码，secret 不进
query/Cookie/日志/Cloud；浏览器入场与一次性交接码的短时 URL 例外见 D6。

| 方法/路径 | 输入与语义 |
| --- | --- |
| POST /login/start | L、完整证明、配置修订/端点绑定、callbackURI、stateHash、PKCE S256 challenge、browserSecretHash、managementSecretHash、initialSecretHash |
| GET /browser/enter | L、B、state；一次入场并绑定安全 Cookie，不直接批准账号 |
| GET /browser/continue | L；经现有 provider 登录返回或恢复确认页，必须有本事务浏览器 Cookie |
| POST /browser/approve | L、state、confirmationNonce；同源与本事务 Cookie + 当前已验证网页账号，批准后返回绑定回调 |
| POST /login/exchange | L、code、state、codeVerifier、initialSecret、callbackURI、已记录配置/身份绑定；原子消费并激活 |
| POST /login/status | L、managementSecret；只返回事务阶段、期限和预分配 ID，不授予会话或返回代码/秘密 |
| POST /login/cancel | L、managementSecret；D8 的受限终结操作，不需要网页/Cloud/provider 继续可用 |
| POST /login/complete | 当前bearer与L；D8的完成确认，终结M权限，不续期/计活动 |

`POST /login/start` 使用独立的 **48 KiB** JSON body 上限，不能继承普通请求的 4 KiB
限制；Gateway 与部署 ingress 的该路由均须允许此上限。完整 JWS 放在唯一 `proof`
字符串字段中，其 ASCII 内容最多 32 KiB，逐字传送、不转义或重编码签名输入。
删除 `proof` 成员后，其余对象按 UTF-8 紧凑 JSON（无无意义空白、仅必要转义）序列化
最多 **8 KiB**；这一组合预算包含 L、配置修订、完整端点/身份绑定、callbackURI、
clientChannel 和所有 digest/challenge，而不只是各字段值之和。

v1 编码器须能在该预算内同时容纳两个各 1024 字节的 base URL、1024 字节 callbackURI、
UUID/部署公钥、uint64 最大修订、正式/development 通道及全部固定长度 digest/challenge；
每入口 4 个 SPKI 和全部已声明能力计入完整证明预算。部署发布检查须使用上述最长组合，
不得只用常见短地址测算；任何新增绑定字段或能力扩展也必须满足同一组合预算，否则另经
协议演进。宿主使用该紧凑编码：32 KiB proof 加 8 KiB 其余对象及 `proof` 成员的 JSON
包装最多 40971 字节，小于 48 KiB，因而所有满足字段与组合预算的配置都可发送。
服务端仍按收到的完整 body 执行 48 KiB 硬上限，并在解码后复核 proof 与其余对象预算；
额外空白或转义不能绕过上限。总 body 超限在解析前拒绝，字段/组合超限在严格解码后、
写入事务前拒绝；两者均不分配 S、不创建 attempt。
不得通过截断证明、去掉绑定或分批发送来适配。宿主发请求前也检查完整 body，重试沿用
同一材料及预算，不为缩小请求替换证明。

start同时绑定已登记的clientChannel，回调路径和本地Pending命名按通道隔离；改变通道
属于不同绑定。clientChannel既不能开启dev provider，也不能替代正式包/回调所有权验证。
原生JSON端点只接受application/json且不接受浏览器Cookie作身份，不为任意网页开放携
资格CORS；响应no-store，网络层不跨源自动重定向。浏览器路由另走D6的Cookie/同源保护。

hash 为 SHA-256 原始随机字节/按 PKCE 定义的 challenge；协议明确每种字段编码，不能
一处哈希 base64 文本而另一处哈希原字节。state/秘密本体以无 padding base64url 传输，
hash 字段也用解码后正好32字节的 base64url；PKCE 的输入例外严格按 RFC7636 的 ASCII verifier。

Gateway 以 PG 校验证明当前配置与期限，绑定首次受理时刻 `createdAt`，建议
`attemptExpiresAt=createdAt+10分钟`。分配随机 session UUID `S`，仅保存在 attempt，
不创建活跃会话。start 响应返回 L/S、期限与固定浏览器路径；B/state 由宿主按协议构造
入场 URL，响应不能给任意 browser URL。相同 L 与完整绑定重试返回相同 S/期限；不同
绑定返回409，不能覆盖已有行或重置 TTL。过期/终结 L 不复用，新登录用新 L/秘密。

start 成功后宿主先保存 S 再开浏览器；若响应丢失且原证明仍有效，用原材料重试 start，
不新建副作用。原60秒证明已过期、10分钟事务仍可能有效时，以M调用login/status取回
原S/期限，不为旧L替换证明再创建。请求前仍须重新验证当前TLS/部署端点，不把已到期的
原创建证明当作当前传输信任；配置修订变化时旧事务失效，不能换新证明给旧L续命。
限流在写入前执行，部署须有正数且有限的待处理事务/浏览器 claim 上限与有界清理，
不能靠无限接受匿名 start 或无限增加 Cookie 消耗资源。

### D6：浏览器先绑定事务，再明确确认当时显示的账号

首次 enter 验证 B/state 的 digest、期限与 pending 状态，原子绑定随机浏览器 Cookie `D`
的 digest，B 此后不能给另一个浏览器 claim。Cookie 使用事务区分名称、HttpOnly、Secure、
SameSite=Lax、host-only 和浏览器子路径，期限不超过 attempt；不覆盖现有网页/provider
attempt Cookie。同一 D 可恢复该事务，其他/缺失 D 不可接管。首次响应丢失导致 D 未保存时
明确失败，回 Desktop 取消后重开，不凭已消费 B 签发第二个浏览器身份。

入场页设置 no-store、no-referrer、无第三方资源，移除含 B/state 的可见 URL；state 只在
同源页面的会话级存储中按 L/期限短暂保留，用于 approve 后回显，不能持久化原始秘密到 PG。
provider 返回路径仅含 L，不把 B/state 塞进持久 return_to；会话级存储不可恢复时明确重试，
不改用任意请求 state 或循环登录。退出/完成/期限到达清理页面状态与本事务 Cookie。

未登录时复用现有 Gateway provider 流程，外部 callback 仍回 Gateway，return_to 为
已验证同源 continue 路径。外部 token/code/verifier 不进入原生事务或宿主，不存 provider
refresh token。已有网页会话可以复用，但 Desktop 会话与网页会话独立。

确认页展示部署、端点与当前账号，从已验证网页会话取得稳定 `(source,subject)`，生成
确认 nonce `Q`，PG 只存 Q 的 digest 与显示的身份快照/版本。approve 必须同时验证
Origin/同源证明、D、Q、state、未到期阶段，以及当前网页会话仍是所显示身份。
Cookie 被其他窗口切成另一账号时返回 account_changed 并重新展示确认，不用请求自报账号。

身份启用查询在 PG 锁外，以已验证身份调用 Cloud；失败不批准，业务授权仍由 Cloud 判定。
批准事务锁定 attempt 再次校验全部绑定，将状态置 Approved、冻结身份、生成一次交接码 C
并只存 digest；`codeExpiresAt=min(批准时PG时间+2分钟,attemptExpiresAt)`。
原始 C 只在该响应导航到原回调，state 来自本次已验证 approve 请求，不从数据库还原秘密。
重复 approve 不能再次签发 C/重设期限；响应丢失且回调无法恢复时由宿主取消并显式重开。

### D7：单次换票激活已准备资格，丢响应只验证同一会话

宿主只接收匹配 L/state/路径/回调目标的 GET，拒绝重复参数、附加不明 query、错误方法、
过期/未知/跨实例/跨部署/已取消回调；请求行上限2KiB，不反射 C 或 state 到页面/日志。
收到有效 C 后先保存在该 PendingLogin 的系统安全存储，再确认接收、关闭监听并换票。
取消/超时也关闭监听；监听 HTTP 仅限本机回环，不是对远端 HTTP 的隐式批准。

exchange 在已验证目标提交，检查 C/state/PKCE、U digest、L/S、部署签名身份、配置修订、
端点对与原 callbackURI。配置变更使未完成事务失效，不能自动换目标继续。先在锁外从
已冻结身份取得当前 Cloud 账号结果，失败不创建会话；锁内重查全部状态、期限与绑定，
原子消费 C、将 attempt 置 Consumed 并创建唯一 S 的 `generation=0/currentDigest=hash(U)`。
不得在锁内调 provider/Cloud；取消和消费同一行锁，只有一个顺序。

成功响应只返非秘密 S/代次/账号上下文/当前政策与期限，不返回 U；宿主已知 bearer 为
`d1.S.U`。session ID 预分配不授予权限，身份取冻结的已验证记录，不取客户端账号字段。
digest 的唯一性与约束在 PG 强制，失败整体回滚，不留“消费成功但无会话”的半状态。

同一 C 的重复 exchange 必须拒绝，不因为要幂等恢复而再建第二个会话或回放资格。
网络/5xx/重复消费后，宿主用已保存 S/U 查询已认证 session/status；成功后通过既有
Cloud `GET /api/v1/me`取得可信账号上下文，再原子保存 Current并清理 PendingLogin，
不假定轮换提案的status响应已经包含用户资料。账号请求失败同样保留恢复状态；用户名/
展示名不作为身份匹配键，正式存储按部署签名身份+Cloud权威用户ID命名。
Current保留非秘密的待完成L及选择epoch，
以便D8的complete在崩溃后重试，不为此保留M/B/PKCE/C。若首次 status 与提交交错返回401，先用
M 查询事务结果再重查 S/U，不能凭一次401判断全失败或重新创建会话。

| 恢复观察 | 处理 |
| --- | --- |
| S/U status与当前账号读取成功，且本地选择epoch仍匹配 | 激活该上下文；即使登录事务期限已过，有效会话仍按其独立政策使用 |
| S/U未激活，事务Approved且C尚在安全存储/期限有效 | 重试原exchange；不得生成另一S/U或替换绑定 |
| 事务Consumed、S/U先前401 | 再查S/U；明确失效才重登，503保留Pending并停止云端操作 |
| Approved但C响应丢失且无本地C | 不从digest恢复C；取消后显式新登录，可复用网页SSO |
| 配置/选择epoch变化、取消、明确到期/停用 | 不能激活旧上下文；尽力取消或以已知S/U退出，保留本地现场 |
| 结果不确定/网络故障 | 有界退避、保留恢复材料，不循环开浏览器、不重放业务任务/消息 |

Pending 中的 U/verifier/M/B/C 不能写明文恢复日志或成为通用离线队列；这只是该有限事务
的系统安全存储记录。后续凭据轮换按最终批准的独立协议，不由本次换票隐式完成。

### D8：取消/换票竞争只影响本事务，不能取消已完成登录的后续会话

M 只证明该 L 的事务管理权，不能访问业务、取秘密、选择别的账号或撤销其他会话。
cancel 不查 provider/Cloud；M 有效且 attempt 尚在其10分钟管理期内时，在同一 PG 行锁下：

- Pending/BrowserBound/Approved：置 Cancelled，C/浏览器批准不再可用。
- exchange 已先赢：只撤销该 L 创建的 S，不能返回“取消完成”却留已激活资格。
- unknown/失效/重复取消：统一幂等204，不泄露事务是否曾存在。

M 的管理权必须在宿主完成可靠 Current 保存后明确终结。宿主随后以当前 S/U 调
`POST /login/complete`（L），Gateway 仅以PG校验当前bearer、验证S确为L产物，并在锁内置
Completed、销毁M的授权；它不需要provider/Cloud仍可用。已换代的有效当前bearer也可完成
该S的同一L，不要求仍持generation0秘密，不能以此访问另一个会话。
这不续期、不计活动、不改变网页会话。Completed 后即便持有旧M也不能取消会话，只有
当前会话的正常 logout/管理性撤销有效；complete 重试幂等。
attempt已按期限清理时，可凭有效当前S及其保留的来源L确认完成，不重建attempt或M授权；
无关联的L不能影响该会话，也不能用新建记录恢复已终结管理权。

Current 落盘后 complete 响应丢失，重启可从 Current 恢复并重试，不重开登录。完成确认前
有崩溃时 M 仍至多有效到原10分钟期限；不无限延长。会话秘密泄露/攻击者抢complete仍属于
bearer妥协风险，不宣称M或Cookie能证明真实设备。

若 cancel 与 complete 并发，先取得锁者确定结果：cancel先赢使资格失效，complete不能
复活；complete先赢使M失权，cancel不影响当前会话。无论何者胜出，本地选择epoch检查都
不得把旧登录写进新的server/账号上下文。离线本机取消清材料并说明远端撤销未确认，
不为事后取消保存完整访问资格；若S/U已确认有效可正常logout，不把未知204当全局撤销证明。

### D9：故障、限流、兼容与资源都有明确边界

| 情况 | 结果 |
| --- | --- |
| metadata/签名/端点公钥/配置不匹配 | 不发资格、不覆盖信任，说明不兼容/连接修复，本地继续可用 |
| malformed、重复键/参数、超限、非法callback/PKCE | 400 invalid_request，不推进事务 |
| 错误secret/state/code、过期/已消费、跨事务绑定 | 统一401 login_failed，不回显原值；宿主用受限状态/已知资格恢复 |
| 配置或绑定冲突、显示账号变化 | 409 configuration_changed/attempt_conflict/account_changed，重新验证/确认，不默选其他目标 |
| 已验证账号停用 | 可区分的user_disabled；普通资源403不作为全局停用；取消仍可进行 |
| 限流/数据库/Cloud/provider故障 | 429或有界503，已有恢复事实不清，退出/取消只受所需PG可用性限制 |

期限由 PG 当前时间决定，不能靠清理任务是否跑过。匿名事务/签名请求限流，清理按有限批
保留必要的终态/会话关联，浏览器Cookie按attempt到期/完成清理。只删除过期事务记录不能
复活Cancelled或Consumed；有效重试期内不能因清理重新受理同一L，墓碑至少覆盖原证明/
事务的最大有效期。原start绑定固定的证明与期限，重试不能刷新proof给旧L续命；旧请求
即使行已清理也因签名证明到期被拒绝。客户端始终生成新L，不承诺用有限墓碑永久检测恶意
客户端以新证明复用已删除的UUID；仍存活S的来源L保持唯一，旧code/M/state也不能用于新绑定。
完整retention/上限配置须在实现契约中明确且可验证，不能凭无限保存回执/秘密达成恢复。

新增持久表采用向前迁移，不改变已应用SQL、网页session或provider TTL。不同版本缺能力
只指明升级方向，最低版本按首次实际发布确定。第一阶段的本地启动、Agent、编辑与运行
在任意故障/未登录/停用下保持独立，不新增同步/隐式上传，也不主动取消已启动远端任务。

## 不变量

1. 未经TOFU/已记身份匹配及TLS端点公钥认可，宿主不发送任何既有/初始会话资格。
2. 签名只认可部署可信配置，不认可客户端自报公钥或转发头；证明有域/nonce/期限/配置绑定。
3. L/端点对/回调/state/PKCE/U从创建到激活不可替换，一次交接至多激活一个S。
4. 页上显示的账号与批准的账号相同；Cookie变化必须再次确认，身份不由请求体选择。
5. 原始U/B/M/PKCE/C不落PG/日志/Cloud；宿主只在系统安全存储持有有限恢复材料。
6. 网络/5xx/一次401不等于明确登录失败，恢复不能重复创建会话或重发业务副作用。
7. 取消与换票/完成以同一事务序列化；Completed后旧M不能撤销已经完成的会话。
8. 任意登录/连接状态都不锁本地；旧选择epoch的结果不能进入新上下文。

## 为什么不是这些替代方案

| 方案 | 收益与代价 |
| --- | --- |
| 只签nonce/URL | 简单且防重放，但另一TLS密钥的在线代理仍可转发挑战 |
| TLS exporter绑定每条连接 | 更细的通道绑定，但TLS终止于ingress时需可信材料传递/代理支持，首版选择部署认可SPKI |
| 自定义canonical JSON或复用内部JWT密钥 | 少一个格式/密钥管理面，但增加跨语言歧义或混淆签名用途，不采用 |
| Gateway生成初始秘密、仅存digest | 客户端简单，但丢响应不能原样返回；加密回放另增密钥和保留责任 |
| approve只看当时最新Cookie | 状态少，但其他窗口切账号可能批准与显示不同的身份 |
| 无期限M一直可取消 | 容易收尾，但泄露的旧事务能力可长期撤销有效会话，不采用 |

## 风险、未决选择与落地

端点私钥/可信ingress被攻破、首次TOFU误认与恶意宿主不由本提案解决；SPKI不是硬件/设备
身份或端到端TLS保证。受控轮换必须先发布绑定，部署方为此承担配置/证书一致性责任。
v1严格URL路径限制、JWS新算法名的库支持、浏览器会话级存储与安全Cookie恢复失败都会
影响接入体验，应有明确错误和三平台互通证据，不能静默放宽到不同协议。

owner需选择签名/SPKI方案、新的60秒/10分钟时限及初始秘密预备/完成确认机制；与轮换/
HTTP各自最终结论有出入时先纠偏。批准后转approved并维护核心用例，再做Gateway/宿主/
正式包实现；最终implemented须覆盖TLS代理转发、证书/配置轮换、账号切换、浏览器入场
丢响应、双窗口、换票/取消/完成竞争、崩溃/重启、重复回调及已有网页回归的直接证据。
批准后核心用例还须覆盖请求预算：最长端点对/回调、每入口 4 个 SPKI 与必需能力的合法
组合可完成 start；32 KiB proof 加 8 KiB 其余对象的边界可发送；JWS、其余对象或总 body
分别超限，以及空白/转义造成编码后超限时均拒绝且不写事务；部署配置发布与发现阶段
提前拒绝不可编码组合，不能让宿主完成信任确认后才发现合法配置无法 start。
