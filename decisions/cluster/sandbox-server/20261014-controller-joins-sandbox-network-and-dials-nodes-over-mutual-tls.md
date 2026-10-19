---
status: implemented
---

# Controller 接入沙盒网络并以双向 TLS 直连 Node，Sandbox Server 的 WS 路由退为本机传输工具

> 历史决策：D1 的直连要求、D3 固定的 Controller 对端及 D4 的路由用途已由
> [Controller 经 Sandbox Server WebSocket 路由连接 Node](20261016-controller-connects-through-sandbox-websocket-router.md)
> 修改。Node 证书签发、TLS 监听与业务目标绑定继续继承；本文保留原直连方案的历史依据。

Cloud 模式下，Controller 不再经 Sandbox Server 的 WS 路由连接沙盒 Node，而是自己接入沙盒网络，以
`wss://ora-sandbox-<sandboxId>:<port>/ora-node/v1` 直连 Node。连接两端双向认证：Node 出示 Sandbox Server 按运行
代次签发的服务端证书，Controller 按容器名校验它；Node 只接受部署固定的那一张 Controller 证书。WS 路由
（[根决策](0-docker-label-resolver.md)的标签解析与中继）照常运行，但只服务不启用 TLS 的本机传输工具，到达不了
启用双向 TLS 的 Node。

本 ADR 处于 `implemented`，记录 cluster#6（2026-09-29 合并）已经落地、此前没有决策记录的拓扑。它落实
[运行时控制同步交付](../../cloud/controller-integration/20260927-fenced-runtime-control-delivery.md)对“Controller–Node
管理通道使用部署提供的双向 TLS 身份”的要求，本文只决定 cluster 这一侧怎样让这条通道成立：网络拓扑、Node 身份
的签发、Node 监听方式。涉及 cluster 的 `compose.yaml`、`compose/controller.json`、`compose/management-setup.sh`、
`sandbox-server/internal/management/tls.go` 与 `sandbox-server/internal/lifecycle/authorization.go`，以及 desktop 的
`apps/ora-controller/src/cloud/fleet.rs`、`crates/node-transport/src/mtls.rs`、`crates/node-transport/src/websocket.rs`
与 `docker/node-entrypoint.sh`。验证见
[Node 直连核心用例](../../../test-cases/cluster/sandbox-server/direct-node-connection.md)。兼容性边界：没有配置
`direct_node_port` 的 Controller（本机传输工具）仍经 WS 路由，行为不变；Cloud 模式的 Controller 不支持回退到路由。

## 继承与修改

| 前序约定 | 本文处理 |
|---|---|
| 根 D2：Node 的 WS 入口只经 Sandbox Server 到达，解析地址只用沙盒网络 IP | 修改：Cloud 模式下 Controller 也在沙盒网络上，直接连接 Node；Node 入口不再靠“只有路由能到达”保护，而靠双向 TLS（D2、D3）。路由自身的解析规则不变 |
| 根“对 T8 的约束”：沙盒网络只接入 Sandbox Server 与 Node，Controller 不在沙盒网络上 | 修改：Controller 接入沙盒网络；Cloud 与浏览器 Gateway 仍不接入（D1） |
| 根 D1、D3：标签声明路由身份，`404`/`503`/`502` 映射 | 继承，不变；只对经路由的连接成立 |
| 生命周期 D2：容器名 `ora-sandbox-<effectId>` 是稳定外部身份 | 继承并扩展：容器名同时是 Controller 拨号的主机名与 Node 服务端证书的 DNS 名（D2） |
| 生命周期 D4：Node 配置 `control.listen` 为 `websocket`，配置不含凭据 | 修改：启用管理身份时为 `mutual_tls_websocket`，证书材料经私有启动环境传入，配置只引用路径（D3） |
| 沙盒部署环境 D3、不变量 3：除 Sandbox Server 外只有 RustFS 接入沙盒网络；Cloud 与 Controller 不接入 | 修改：Controller 接入（本文），model-gateway 接入（模型客户端身份决策）；Cloud 仍不接入 |

## 决策

### D1：Controller 接入沙盒网络，Cloud 模式只用直连

Compose 中 Controller 同时接入默认服务网络与沙盒网络。Controller 配置 `substrate.direct_node_port` 时，沙盒 Node
的端点为 `wss://ora-sandbox-<sandboxId>:<direct_node_port>/ora-node/v1`，主机名由 Docker 内置 DNS 在沙盒网络上
解析；`ate-target-actor` 请求头照常发送，Node 不使用它。Controller 以 Cloud 持久化启动时，若没有
`direct_node_port`、或 effects 接口不是 `https://`，直接拒绝启动，不回退到明文路由。

沙盒网络上的成员因此是：Sandbox Server、Controller、RustFS（别名 `ora-revisions`）、model-gateway（别名
`ora-model-gateway`）与各沙盒容器。Cloud、PostgreSQL 与浏览器 Gateway 不在沙盒网络上。

为什么直连而不是继续经路由：WS 路由在 Sandbox Server 处终止 WebSocket，Controller 与 Node 之间无法建立端到端
的 TLS，Node 只能信任“能连进来的就是路由”。运行时控制要求 Node 认证 Controller、Controller 认证目标 Node，
这只有在 TLS 终止于 Node 本身时才成立。

### D2：Sandbox Server 按运行代次签发 Node 服务端证书

签发 CA 在 `management-setup` 一次性任务中生成，私钥只放在 Sandbox Server 的管理卷；Controller 只持有公开 CA
与自己的证书，不能签发 Node 身份。每次 `sandbox_ensure` 创建容器时，Sandbox Server 为该容器签一张新证书：

| 字段 | 值 |
|---|---|
| 用途 | 只有 `serverAuth` |
| DNS SAN | 容器名 `ora-sandbox-<sandboxId>` |
| URI SAN | `spiffe://ora.local/tenant/<tenant>/runtime/<workspace>/generation/<generation>` |
| 有效期 | 24 小时 |

租户、Workspace 与代次取自 Cloud 为本次 ensure 签发的许可。证书、私钥、CA 与 Controller 证书经
`ORA_NODE_CERT`、`ORA_NODE_KEY`、`ORA_NODE_CA`、`ORA_CONTROLLER_CERT` 私有启动环境传入，不进入 effect 请求、
journal 或结果；Node 入口脚本把它们写入管理 tmpfs（`/run/ora`，权限 `0600`）后从环境中清除。

Controller 校验 Node 证书链与主机名：DNS SAN 必须等于拨号的容器名。主机名来自 Cloud 记录的 sandbox ID，所以
一张为别的沙盒签发的证书无法冒充目标 Node。

### D3：Node 只接受固定的 Controller 证书

Node 监听 `mutual_tls_websocket`：要求客户端证书，先按 CA 校验，再要求它与部署提供的 Controller 证书逐字节
相同，不符时在 WebSocket 升级前断开。同一 CA 签发的其他证书（例如 model-access 客户端证书）因此也进不来。
Node 配置的 `control.target` 记录本代沙盒的租户、Workspace、sandbox 与运行代次；Controller 在会话中提交的运行时
绑定与它不一致时，Node 拒绝。

为什么固定证书而不是只校验 CA：同一 CA 还签发 model-access 与 git-access 等工作负载身份，只校验 CA 会让持有
任何一张工作负载证书的进程连上 Node 的管理入口。

### D4：WS 路由保留为本机传输工具

Sandbox Server 继续在 `-addr` 上提供根决策的 WS 路由与中继，供不启用 TLS 的 Node（静态表、回环传输夹具）使用。
它只拨 `ws://`，不能到达监听 `mutual_tls_websocket` 的 Node；不为它增加 TLS 透传。Node 的生产可执行文件拒绝
明文 `websocket` 监听，明文变体只留给传输测试夹具，所以沙盒 Node 不会因配置错误退回到可经路由到达的明文入口。

## 不变量

1. Cloud 模式的 Controller 只经双向 TLS 直连沙盒 Node，配置不满足时拒绝启动，不回退到明文路由。
2. Node 的管理入口只接受部署固定的 Controller 证书；同一 CA 签发的其他证书一律拒绝。
3. Node 服务端证书只有 `serverAuth` 用途，DNS SAN 是本沙盒的容器名，每代沙盒重新签发。
4. Node 身份的签发 CA 私钥只在 Sandbox Server 的管理卷中，Controller 没有签发权。
5. 证书与私钥不出现在 effect 请求、journal、结果或 Node 配置正文中。
6. Cloud 不接入沙盒网络。

## 为什么不是这些替代方案

| 方案 | 取舍 |
|---|---|
| 继续经 WS 路由，在路由上做认证 | Node 只能信任路由，无法独立认证 Controller，运行时控制要求的端到端身份不成立；优势是 Controller 不进沙盒网络、沙盒网络成员更少 |
| 路由做 TLS 透传（按 SNI 转发 TCP） | 能保住端到端 TLS 与 Controller 不进沙盒网络，但要在路由里新增一条 TCP 转发路径和按名解析；本地只有一个 Controller，直连更简单 |
| Node 主动回连 Controller | 需要 Node 知道并能到达 Controller，反转了现有协议的拨号方向（传输决策 D3），改动面大 |
| 只校验 CA，不固定 Controller 证书 | 同一 CA 签发的工作负载证书也能连上 Node 管理入口（D3） |
| Controller 持有 CA 私钥自行签发 | Controller 被攻破即可伪造任意 Node；签发留在只做生命周期的 Sandbox Server（D2） |

## 本决策未解决的问题

- 证书刷新：沙盒运行超过 24 小时后 Node 证书过期，尚无刷新机制。
- 生产形态：受控签发、证书轮换与生产 Substrate 的网络方案不在本地部署范围内。
- Controller 只校验 Node 证书的 DNS SAN，没有比对 URI SAN 中的租户与代次；当前由主机名唯一对应 sandbox ID
  保证目标正确。

## 落地

cluster#6 与 desktop#632 一起落地：Compose 生成管理证书与私有卷，Controller 接入沙盒网络，Sandbox Server 在
ensure 时签发 Node 身份，Node 入口脚本落盘证书并以 `mutual_tls_websocket` 监听，Controller 在 Cloud 模式下改为
直连。真实链路由 cluster 的 `scripts/runtime-control-acceptance.py` 验收（见 cluster `docs/testing/runtime-control.md`）。
保持不变：WS 路由与静态表的行为、生命周期 effect 语义。开始改变：Cloud 模式的 Controller–Node 连接不再经过
Sandbox Server。
