---
status: proposed
---

# Controller 经 Sandbox Server WebSocket 路由连接 Node，联调允许同网

提议将 cluster 的 Controller–Node 接入改为与 `ora-space/sandbox` 一致的受控 WebSocket
路由：Controller 连接统一入口并以沙盒 ID 选择目标，Sandbox Server 另建到 Node 的连接并中继。
Controller 不再直接拨号 Node。cluster 联调允许 Controller 与 Node 同网，不要求网络隔离。

本文为 `proposed`：方向与部署认证边界仍待 owner 审查，不宣告已批准或已完成真实业务验收。
涉及 cluster 的 Sandbox Server、desktop 的 Controller 与配置；不修改 Cloud 业务权威和 Node
运行时控制协议。它对齐 sandbox 的连接行为，不复制其尚未补齐的生产认证能力，也不以其 Go POC
runtime 替代真实 Rust Node。来源为 circus #160，preflight 实现由 circus #165 承接。

## 问题与约束

前序[双向 TLS 直连决策](20261014-controller-joins-sandbox-network-and-dials-nodes-over-mutual-tls.md)
依赖 Controller 可解析并访问每个 Node 的内部地址。真实 Agent Substrate 内部部署可能仅提供
受控 router/ingress，没有这样的可达性。仅在 cluster 验证直连会掩盖平台接入差异。

现有 Rust Node 的可执行文件要求 TLS 管理监听。本次只对齐路由接入，不扩展生产认证、授权、
签发和轮换能力，也不实现 TLS-over-WS 隧道。复用现有证书机制建立两段 TLS 连接，以便在不放宽
Node 监听的情况下验证真实执行链；这改变了直接认证的对端，不能声称端到端认证保持不变。

## 继承与修改

| 前序约定 | 本文处理 |
|---|---|
| 直连 D1：Cloud Controller 必须直连，不回退到 router | 修改：只使用配置的 router URL，无直连分支；Cloud 配置要求 HTTPS effects 与 WSS router |
| 直连 D2：Node 证书绑定 sandbox 与运行代次，签发权留在 Sandbox Server | 继承，不变；由 Sandbox Server 验证目标 Node 的证书 |
| 直连 D3：Node 固定接受 Controller 证书 | 修改：本机部署 Node 固定接受独立 router 客户端证书，Controller 私钥不交给 router |
| 直连 D4：WS router 只服务明文本机工具 | 修改：Cloud 联调使用两段 TLS WS；无 lifecycle 的静态表工具保留原行为 |
| 根 D1/D3：标签声明身份、唯一目标、固定错误分类 | 继承，并在 Docker 发现前增加 lifecycle 创建状态与终止墓碑检查 |
| 生命周期：先持久化终止墓碑，再执行 Docker 停止与移除 | 继承，墓碑生效后即使容器残存也拒绝新的路由连接 |

## D1：Controller 只向统一路由入口拨号

每个目标使用同一个配置的 `router_url`；`Ate-Target-Actor` 携带
`<atespace>/<sandboxInstanceId>`。Controller 不从 sandbox ID 构造主机名，不保留直连 fallback。
旧 `direct_node_port` 配置删除并明确拒绝，防止部署继续以为该字段影响连接路径。

这仅改变 Node 的连接入口，effects API、Cloud 存活列表、Node hello、业务消息与 fencing 继续
由原所有者负责。中继不解析、重写业务帧；不把身份请求头当成新的运行时授权。

## D2：Sandbox Server 负责发现和中继生命周期

启用 lifecycle 的部署在 Docker 标签发现之前检查创建 effect 和终止墓碑。未知/已终止为 404，
未完成创建或 Docker 不可用目标为 503，发现/连接失败为 502，拨号超时为 504。只有唯一存活
目标才建立上游；上游连接成功前不接受 Controller 的 WS 升级。

每次重连重新发现目标，不缓存旧 IP。终止实际停止 Node 后已有连接关闭；终止已登记而 Docker
停止尚未成功时，新连接先被墓碑拒绝，不能据此宣称所有既有执行责任已经结束。Node 的旧代次
拒绝仍由运行时控制契约负责。中继的 shutdown、close、背压和消息大小约束继承现有实现。

## D3：联调同网与逐跳 TLS

Compose 保留 Controller 与 Node 同网；应用路径由配置和 Controller 实现保证，不新增隔离或
防火墙前置条件。Controller 只认证 Sandbox Server；Sandbox Server 用独立 router 身份连接 Node，
按目标 sandbox 名验证 Node 服务端证书。Node 固定接受 router 身份。两条 WS 连接分别终止 TLS，
不是 Controller–Node 端到端 TLS。

这里复用本机管理证书，只为现有真实 Node 提供可运行的接入。它不决定真实 sandbox 的生产认证，
不承诺完成 Project/Workspace 入口授权、证书续期或跨部署信任治理。

## 不变量

1. Controller 的沙盒 Node 连接仅指向配置的 router，无直连或错误后回退分支。
2. 路由目标唯一；终止墓碑优先于残存容器，未知与暂不可用有不同结果。
3. 业务目标绑定、Cloud 许可与 Node fencing 不由中继重写或代替。
4. 联调同网不改变实际连接路径；Controller 私钥不复制给 Sandbox Server。
5. 路由 shutdown 和后端连接终结均有明确的资源释放责任。

## 为什么不是这些替代方案

| 方案 | 取舍 |
|---|---|
| 继续要求每 Node 内部地址可达 | cluster 已能工作，但无法覆盖仅暴露平台 ingress 的真实部署 |
| TLS-over-WS 隧道 | 可保留端到端认证，但增加两端流适配；超出本轮接入对齐范围 |
| 放宽真实 Node 为明文监听 | 改动小，但会改变 Node 可执行文件的安全约束；本轮没有必要这样做 |
| 用网络隔离禁止直连 | 可进一步收紧部署，但不能代替应用连接路径对齐，也不是联调前置条件 |

## 升级与验证义务

Controller、Sandbox Server 和配置需要协调升级。旧 Node 固定的客户端仍是 Controller 证书，
不能当成新路由后端继续使用。先通过 Cloud 正常停止/终止旧实例并保留数据，再生成 router 身份、
部署双方，最后正常启动 Workspace 创建新沙盒；不改 journal 或删除用户数据，也不为迁移放宽
身份验证。管理证书生成应幂等复用现有 CA，新增独立 router 身份。

评审需核对双沙盒路由与重连、错误目标、创建未完成、终止墓碑、真实 Node hello，以及受影响的
Cloud/插件/Agent/Revision 验收。子层传输测试不能代替完整业务链路；在 ADR 批准后再维护对应
核心测试用例。生产认证与过期证书轮换属于后续工作。
