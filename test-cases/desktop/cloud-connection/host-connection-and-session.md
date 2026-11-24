# 宿主连接、回调与本地独立

依据[宿主连接根 ADR](../../../decisions/desktop/cloud-connection/0-host-owned-cloud-connection-and-session.md) 与
[第一版后续 ADR](../../../decisions/desktop/cloud-connection/20261121-first-version-loopback-login-and-host-held-token.md)
（下称“第一版 ADR”）。根决策中暂停的义务（部署身份、TOFU 与能力协商、证书固定与内网 HTTP、换地址迁移、活动上报与
轮换、60 秒发现吊销、deep link）不在本文；它们恢复时由新的后续 ADR 补回用例。
证据读取的是合并后的 desktop `7fc4544f`（ora-space/desktop#806）与 cloud `1635cc7d`（ora-space/cloud#432）。真实环境验证都用开发版
（`V2_LOGIN=email`）的 debug 构建，正式安装包没有验证过，目前也没有承接的事项。Linux 真实栈跑在 desktop `e17c4f82`，之后
`65fc2552`、`c4788689`、`8d03a4df`、`4ce28c05` 改写了协调器与安全存储，没有在最终代码上重跑；Windows 11 真机跑在 desktop
`63aee9b8`、cloud `15161217`，代码与合并提交相同。

## Local Work Remains Available Across Every Cloud Failure

- 风险：Cloud 状态错误地成为本地启动、编辑或 Agent 运行的准入条件。
- 前置状态：本地工作区、编辑和模型连接；分别处于未配置 Cloud、未登录、已登录、需要重新登录、安全存储不可用。
- 触发：启动与回到前台的检查遇到断网、超时、TLS 错误、3xx、5xx、`401`；登录事务进行中退出程序。
- 必须成立：本地启动、Agent、编辑和工作区照常可用；检查异步进行，不阻塞本地就绪；`401` 只清令牌，不动本地文件、
  工作区和未发送内容。
- 禁止结果：本地等待 Cloud；为本地操作弹出登录；被动失效删除或改动本地现场。
- 决策依据：根决策 D1、D6；第一版 ADR D5，不变量 1、7。

### 验证义务与证据

| 验证义务 | 代表性证据 | 状态 |
| --- | --- | --- |
| Windows 与 Linux 正式包在各种 Cloud 状态下启动、本地编辑与 Agent 运行正常，检查不阻塞就绪 | Linux 真实栈（GNOME、gnome-keyring，desktop `e17c4f82`）与 Windows 11 真机（debug 构建，ora-space/circus#1379）：未登录、已登录、需要重新登录时应用照常启动；`app_lib::cloud_connection::tests::an_unusable_secure_store_turns_cloud_off`（启动时安全存储失败只表现为 Cloud 状态）。仍缺：正式包，以及在这些状态下实际运行 Agent 与编辑 | `Partial` |
| `401` 与安全存储不可用只改变 Cloud 状态，不触碰本地现场 | `app_lib::cloud_connection::tests::a_rejected_token_is_deleted`；`an_unusable_secure_store_turns_cloud_off`（只改变 Cloud 会话与条目）。协调器只持有安全存储、Cloud 接口、浏览器和地址设置这几个端口，碰不到本地文件；没有测试直接断言本地现场不变 | `Partial` |

## Only The Host Holds The Desktop Token

- 风险：令牌进入 webview、插件、日志、本地配置或 URL；安全存储失败后以明文或内存方式“将就”保持登录。
- 前置状态：主 webview、插件 webview、可注入失败的系统安全存储；Windows 凭据管理器与 Linux Secret Service；同一操作系统
  用户下的开发版（自己的 `ORA_DATA_DIR`）与正式包。
- 触发：登录、重启读回、检查、退出、改地址；安全存储写入或删除失败；漫游配置文件的 Windows 用户。
- 必须成立：令牌只在宿主内存和安全存储，条目作用范围是“操作系统用户 + Cloud origin”，Windows 条目只存本机、不漫游；
  开发版与正式包共用同一 origin 的条目、不同时运行；前端只拿到状态、地址和显示名，Cloud 连接命令只授权给主 webview；写入失败时尽力吊销刚签发的令牌并显示安全存储不可用；
  退出时删除失败照样吊销并显示安全存储不可用。
- 禁止结果：插件或页面读到令牌；令牌出现在 SQLite 配置、日志、诊断或 URL；安全存储不可用时仍显示已登录。
- 决策依据：根决策 D2；第一版 ADR D3，不变量 2、6。

### 验证义务与证据

| 验证义务 | 代表性证据 | 状态 |
| --- | --- | --- |
| Windows 与 Linux 真实安全存储的写入、读回、删除，Windows 条目为本机持久化 | Linux 真实栈（GNOME、gnome-keyring，desktop `e17c4f82`，早于安全存储改写）：登录写入、重启读回、停用后删除、退出删除；Windows 11 真机（debug 构建，ora-space/circus#1379）：`CredReadW` 读到 GENERIC、`Persist=2`（`CRED_PERSIST_LOCAL_MACHINE`），重启读回，停用与退出后读回为 not found；`app_lib::cloud_connection::secure_store::platform::tests::blob_round_trips_as_utf16`（只在 Windows 上编译，CI 不跑，见 ora-space/circus#1483）。仍缺：Linux 在最终代码上的真实安全存储；“不漫游”只有 Persist 位作证 | `Partial` |
| 写入、删除失败的各条路径（尽力吊销、状态、不在内存保持登录） | `app_lib::cloud_connection::tests::an_unusable_secure_store_turns_cloud_off`（写入失败：吊销刚签发的令牌、不存条目、显示安全存储不可用；启动读取失败同样显示）；`signing_out_with_a_failing_store_still_revokes_and_says_so`（退出时删除失败仍吊销并显示）。仍缺：改地址时和收到 `401` 后删除失败这两条 | `Partial` |
| 前端与插件拿不到令牌，日志、配置与 URL 中没有令牌 | `xtask::export_desktop::tests::public_catalog_is_complete_and_plugin_grants_remain_restricted`（插件 webview 仍只有 `plugin_webview_invoke`）；`ora_backend::settings::tests::cloud_server_round_trips_and_clears`（本地配置只存 origin）。令牌类型的 `Debug` 只打印占位符、契约里的连接状态没有令牌字段，但没有测试直接断言日志与 URL 中没有令牌 | `Partial` |

## Callbacks Cannot Change The Initiating Server Or Transaction

- 风险：本机其他程序或旧页面借回环地址打断登录、换走令牌或让令牌发往别处。
- 前置状态：一个进行中的登录事务；本机其他程序可访问回环端口；Cloud 确认页已打开。
- 触发：错误路径、错误或缺失 `state`、带 `error`（含确认页「取消」送回的 `access_denied`）、重复投递、超时后投递；
  事务中再点「登录」、取消、改地址、退出程序。
- 必须成立：监听只绑定回环地址的系统分配端口；只有 `GET /callback` 且 `state` 一致的第一次投递结束事务，错误路径或
  `state` 回 `404` 且事务继续；`access_denied` 立即以取消结束；换令牌只发往事务记录的 origin、带同一回调地址和 verifier；
  事务以任何方式结束都关闭监听、丢弃 verifier；收到授权码后才取消、改地址或再点「登录」时，换到的令牌被删除并吊销；
  宿主不自动重新打开浏览器；同一时刻至多一个事务。
- 禁止结果：回调改变换令牌目标；错误请求结束或劫持事务；回调页在换令牌前声称已登录；事务结束后监听仍在；取消后留下
  没人吊销的令牌或删掉更新一次登录的条目。
- 决策依据：根决策 D3；第一版 ADR D2，不变量 5。

### 验证义务与证据

| 验证义务 | 代表性证据 | 状态 |
| --- | --- | --- |
| 回调状态机：错误路径与 `state`、`error`、重复、超时、取消与替换事务 | `app_lib::cloud_connection::loopback::tests::ignores_strays_and_returns_the_matching_code`（错误路径与 `state` 回 `404`、事务继续，合法投递后监听关闭）；`error_redirect_is_denied`；`app_lib::cloud_connection::tests::a_denied_approval_signs_nothing_in`；`an_unanswered_sign_in_times_out_and_stops_listening`；`cancelling_releases_the_listener`；`a_new_sign_in_replaces_the_previous_one` | `Covered` |
| 收到授权码后取消、改地址或替换事务：令牌删除并吊销，新登录的条目不受影响 | `app_lib::cloud_connection::tests::cancelling_during_the_exchange_revokes_the_issued_token`；`cancelling_during_the_secure_store_write_leaves_nothing_stored`；`changing_the_server_during_the_exchange_revokes_the_issued_token`；`a_new_sign_in_during_the_exchange_revokes_the_old_token` | `Covered` |
| 监听范围与所有结束路径的资源释放 | `app_lib::cloud_connection::loopback::tests::ignores_strays_and_returns_the_matching_code`（只绑定 `127.0.0.1`，成功后关闭）；`app_lib::cloud_connection::tests::an_unanswered_sign_in_times_out_and_stops_listening`；`cancelling_releases_the_listener`；`a_new_sign_in_replaces_the_previous_one`；Windows 11 真机（debug 构建，ora-space/circus#1379）：登录中只监听 `127.0.0.1:<随机端口>`，没有防火墙提示。仍缺：被拒绝结束和退出程序这两条路径没有断言监听已关闭 | `Partial` |
| Windows 与 Linux 正式包经真实系统浏览器完成登录与确认页取消 | Windows 11 真机（debug 构建，ora-space/circus#1379）：宿主用系统接口打开默认 Chrome，登录与确认页「取消」都完成；Linux 真实栈（GNOME、gnome-keyring，desktop `e17c4f82`）：真实 Chromium 完成登录。仍缺：正式包，以及 Linux 上的确认页「取消」 | `Partial` |

## The Token Only Goes To The Origin That Issued It

- 风险：令牌经重定向、代理或改过的地址发给别的服务器，或以明文经网络传输。
- 前置状态：已登录的 Cloud origin；设置了 `HTTP_PROXY` / `HTTPS_PROXY` 与 `NO_PROXY`；回环开发地址与 HTTPS 地址。
- 触发：Cloud 回答 3xx；改地址；回环地址或命中 `NO_PROXY` 的地址；系统不信任的证书；用户输入各种形式的地址。
- 必须成立：地址规范化为 origin（补 `https://`、小写、去默认端口，拒绝用户信息、查询串、fragment 和路径）；`http` 只给
  回环主机；客户端不跟随重定向，3xx 按连不上处理；回环地址与 `NO_PROXY` 命中时直连、不取环境代理；TLS 只信系统信任库；
  改地址后旧令牌不再发出。
- 禁止结果：令牌跟随重定向到另一个 origin；回环地址的令牌经代理明文发出；非回环地址接受 `http`；为连上而关闭证书校验。
- 决策依据：根决策 D4（重定向部分）、不变量 3；第一版 ADR D1、D4，不变量 3。

### 验证义务与证据

| 验证义务 | 代表性证据 | 状态 |
| --- | --- | --- |
| 地址规范化与拒绝清单 | `app_lib::cloud_connection::server::tests::normalizes_to_one_origin`；`refuses_addresses_that_are_not_a_bare_origin` | `Covered` |
| 3xx 不跟随、回环与 `NO_PROXY` 直连、系统信任库校验 | `app_lib::cloud_connection::api::tests::only_unauthorized_rejects_the_token`（带 `Location` 的 `302` 按连不上处理）；`ora_utils::http::client::tests::environment_proxy_is_not_used_for_loopback`；`loopback_request_goes_direct`；`ora_utils::http::proxy::tests::no_proxy_bypasses_environment_proxy`（规则级：命中 `NO_PROXY` 时不取环境代理）；Windows 11 真机（debug 构建，ora-space/circus#1379）：设了死端口的 `HTTP(S)_PROXY` / `ALL_PROXY` 后仍能连上回环地址的 Cloud。仍缺：非回环主机命中 `NO_PROXY` 的直连、只信系统信任库 | `Partial` |
| 改地址后旧令牌不发往新地址 | `app_lib::cloud_connection::tests::changing_the_server_signs_out_of_the_old_one`（改地址后新地址为未登录、旧条目已删、吊销只发往旧地址）；`changing_the_server_during_the_exchange_revokes_the_issued_token`。没有断言新地址从未收到旧令牌 | `Partial` |

## Only A 401 For The Current Token Ends The Session

- 风险：网络故障被误当作登出；迟到的 `401` 删掉新登录的令牌；失效后仍继续使用令牌。
- 前置状态：已登录；检查在途时发生退出、改地址或重新登录。
- 触发：启动与回到前台（节流一分钟）的检查，以及任何带令牌的请求，得到 `2xx`、`401`、其他 `4xx`、3xx、`5xx`、断网、
  超时、TLS 错误；Cloud 停用账号或令牌 30 天未用。
- 必须成立：只有对当前令牌的 `401`、且发出后代未变，才删除条目和内存令牌并转为需要重新登录；网络错误、超时、TLS 错误、
  3xx 和 `5xx` 保留令牌并显示连不上；其他 `4xx` 不改变会话；停用账号后下一次检查提示重新登录；删除失败时状态照样转为
  需要重新登录，残留条目下次启动再检查删除。
- 禁止结果：断网或 5xx 清掉令牌；迟到的 `401` 影响新的登录；`401` 后仍发送旧令牌。
- 决策依据：根决策 D5（网络与 5xx 部分）、D6（迟到响应）；第一版 ADR D5，不变量 4。

### 验证义务与证据

| 验证义务 | 代表性证据 | 状态 |
| --- | --- | --- |
| 各类回答到会话状态的映射 | `app_lib::cloud_connection::api::tests::only_unauthorized_rejects_the_token`（`401` 为拒绝；`500`、`503`、`302`、端口关闭为连不上；`404` 为请求错误）；`app_lib::cloud_connection::tests::a_rejected_token_is_deleted`；`an_unreachable_server_keeps_the_token`。仍缺：超时、TLS 错误，以及其他 `4xx` 在协调器里不改变会话 | `Partial` |
| 迟到 `401` 与新登录、退出、改地址的竞争 | `app_lib::cloud_connection::tests::a_late_rejection_leaves_a_newer_sign_in_alone`（旧令牌的 `401` 在退出并重新登录后到达：新会话和新条目都不受影响）。仍缺：改地址后迟到的 `401` | `Partial` |
| 真实 v2-server 上停用账号后重启提示重新登录、条目被删 | Linux 真实栈（GNOME、gnome-keyring，desktop `e17c4f82`，早于协调器改写）：停用后重启为需要重新登录、钥匙环条目被删；Windows 11 真机（debug 构建，ora-space/circus#1379）：停用后回到前台的检查得到 `401`，提示重新登录、条目被删；`app_lib::cloud_connection::tests::end_to_end::signs_in_restarts_notices_revocation_and_signs_out`（真实 HTTP 客户端与回环，桩 Cloud）。仍缺：Linux 在最终代码上重跑 | `Partial` |

## Signing Out Or Changing The Address Only Affects This Desktop

- 风险：退出不彻底（本机仍留令牌）、影响网页会话或其他设备、改动本地身份与数据。
- 前置状态：已登录；同一账号另有网页会话和其他 desktop；Cloud 可达或不可达。
- 触发：退出登录；改地址或清空地址；登出请求失败或超时；安全存储删除失败。
- 必须成立：先删除本机条目和内存令牌，再用内存里的令牌吊销；吊销失败也清本地并提示“服务端撤销未确认”，改地址同样提示；
  不保留令牌作重试队列；同时取消进行中的登录；网页会话、其他设备的令牌、本地文件、工作区、模型连接、Agent 凭据和
  Git 身份不变。改地址时删除旧条目失败：旧条目按旧 origin 记，新地址读不到它，新地址保持未登录；失败写日志，
  仍用内存里的令牌请旧服务器吊销，吊销结果照常经 `SetCloudServerResponse.revocation` 回报。
- 禁止结果：退出后本机仍可用旧令牌；为重试保留令牌；退出影响网页会话或其他设备；改动本地 Git 或模型身份。
- 决策依据：根决策 D6；第一版 ADR D1、D6，不变量 7。

### 验证义务与证据

| 验证义务 | 代表性证据 | 状态 |
| --- | --- | --- |
| 退出与改地址的顺序、吊销确认与未确认两种结果、取消进行中的登录 | `app_lib::cloud_connection::tests::signing_out_offline_is_local_and_unconfirmed`（连不上时本机已清、结果为未确认；可达时为已确认）；`changing_the_server_signs_out_of_the_old_one`（改地址删旧条目、吊销未确认如实回报）；`cancelling_during_the_exchange_revokes_the_issued_token`。仍缺：“先删条目再吊销”的顺序和退出时取消进行中的登录没有直接断言 | `Partial` |
| 真实 v2-server 上 desktop 退出后令牌 `401`、网页会话仍有效 | cloud `internal/server/desktop_login_test.go` 的 `TestDesktopLogoutAndAccountState`（真实 v2-server 与数据库：登出后该令牌 `401`，同账号另一个令牌和网页会话仍有效）；Linux 真实栈（GNOME、gnome-keyring，desktop `e17c4f82`）与 Windows 11 真机（debug 构建，ora-space/circus#1379）：退出后吊销已确认、条目删除，服务端只剩别的令牌 | `Covered` |
| 改地址时删除旧条目失败：新地址未登录、旧令牌仍请求吊销并回报结果 | 尚无 | `Missing` |
