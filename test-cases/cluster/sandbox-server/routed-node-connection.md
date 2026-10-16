# Controller 经 Sandbox Server 路由连接 Node

依据[已批准的路由接入决策](../../../decisions/cluster/sandbox-server/20261016-controller-connects-through-sandbox-websocket-router.md)。
本文取代[历史直连用例](direct-node-connection.md)的接入与认证对端约定，继承 Node TLS 监听、证书用途及材料隔离义务。
Docker 标签解析见[解析用例](docker-label-resolution.md)；业务目标、Cloud 许可与 fencing 继续由运行时控制协议负责。
证据对应 desktop `3e7893f2`、cluster `23b775e`，路径相对各自仓库根目录；不代表配套 PR 已合并。

## Every Sandbox Uses The Configured Router Without Direct Fallback

- 风险：联调同网使直连分支仍能成功，掩盖真实部署不可达；统一入口丢失目标导致消息串线。
- 前置状态：Cloud Controller 配置 substrate 和管理 TLS；Cloud 提供两个不同沙盒的 ensure 结果。
- 触发：首次连接、断线重连或读取旧部署配置。
- 必须成立：每个目标使用同一 router_url，以 Ate-Target-Actor 唯一寻址；不构造 Node 域名/IP，不在失败后直连；Cloud 启动要求 HTTPS effects 与 WSS router，旧 direct_node_port 被拒绝；同网不改变应用路径。
- 禁止结果：回退直连、目标 A 的消息送到 B、只凭配置文本就宣称真实业务经过路由。

### 验证义务与证据

| 义务 | 直接证据与缺口 | 状态 |
|---|---|---|
| 多目标固定使用 router 与目标头 | desktop `apps/ora-controller/src/cloud/fleet.rs` 的 `tls_targets_share_the_router_without_deriving_node_hostnames` 验证带管理 TLS 的两个目标共用 URL、各自目标头和明文 URL 拒绝；`sandbox_targets_come_from_the_router_and_the_ensure_evidence` 验证 ensure 来源 | Covered |
| 多目标隔离、重连与二进制中继 | cluster `sandbox-server/internal/server/routed_tls_test.go` 的 `TestRoutedTLSSelectsNodeAndReconnectsWithoutDirectControllerAccess` 并发访问两个 TLS 后端，各重连一次并核对目标标记和 payload | Covered |
| Cloud 启动门禁及旧配置拒绝 | desktop `apps/ora-controller/src/main.rs` 和配置反序列化已有实现；尚无直接覆盖全部门禁与旧字段的测试 | Missing |
| 真实 Controller 业务路径 | cluster 的 runtime-control、plugin-step、agent-session、revision、revision-resume acceptance 在隔离栈使用真实 Controller/Node 通过；另以活动 TCP 对端和 relay opened 日志核对只访问 router/lifecycle 与 Cloud。路径核对仍为手工步骤，脚本未自动拒绝直连回归 | Partial |

决策依据：路由接入 D1、D3，不变量 1、3、4。

## Two TLS Hops Authenticate Distinct Management Peers

- 风险：把逐跳 TLS 当作端到端认证；接错沙盒证书；同 CA 其他主体冒充固定 peer；复制 Controller 私钥或开放明文来迁就部署。
- 前置状态：部署持有 CA、Controller 和独立 router 身份；当前许可指定 tenant/workspace/sandbox/generation。
- 触发：Controller 连接 router，router 连接目标 Node；另尝试错误服务端或客户端身份。
- 必须成立：Controller 按 URL 验证 Sandbox Server，入口固定接受 Controller；router 每次连接独立按 ora-sandbox-id 校验 Node；Node 固定接受独立 router 客户端证书。Node serverAuth 与 router clientAuth 分离，Controller 私钥不复制；Node TLS、业务围栏及材料隔离继续成立。
- 禁止结果：错误目标证书升级成功、工作负载身份冒充管理 peer、共享私钥、明文回退；把本机传输验证当作完整生产认证或证书轮换的证明。

### 验证义务与证据

| 义务 | 直接证据与缺口 | 状态 |
|---|---|---|
| 目标证书与并发 TLS 配置 | cluster `TestRoutedTLSSelectsNodeAndReconnectsWithoutDirectControllerAccess` 的后端只接受 router、拒绝 Controller；c 指向 a 的证书时返回 502；并发目标互不覆盖 ServerName | Covered |
| router 入口固定 Controller | cluster `sandbox-server/internal/management/tls_test.go` 的 `TestServerMutualTLSRequiresPinnedControllerEvenForAnotherTrustedCertificate` 验证固定证书、拒绝同 CA 其他证书 | Covered |
| Node 服务端用途、DNS 与模型身份隔离 | cluster `sandbox-server/internal/lifecycle/model_environment_test.go` 的 `TestSecuredEnsurePreservesNodeEnvAndIsolatesModelIdentity` 解析证书并核验用途、DNS、环境和独立模型身份 | Covered |
| 真实 Node 正向装配与错误 peer 拒绝 | desktop `crates/node-transport/tests/mtls.rs` 验证固定 peer 和工作负载证书拒绝；cluster routed TLS 的 real_node 子测试与 `TestDockerLifecycleRoutesRealNode` 验证生产装配的真实 Node hello。真实 Docker 装配下错误 Controller/工作负载证书的完整负向矩阵仍缺直接断言 | Partial |
| 带 TLS 端点不回退明文 | desktop `management_tls_never_falls_back_to_plaintext_even_without_headers` 及 Controller 多目标配置测试拒绝 ws URL | Covered |
| Node 可执行文件拒绝明文监听 | desktop `apps/ora-node/src/main.rs` 的拒绝分支尚无直接测试，保留原缺口 | Missing |
| ensure 材料不泄露 | cluster `TestSecuredEnsurePreservesNodeEnvAndIsolatesModelIdentity` 覆盖正常路径与重放；失败路径尚缺直接断言 | Partial |
| 独立 router 身份和幂等初始化 | `compose/management-setup.sh` 已手工连续执行并核验 clientAuth 和独立材料；尚无自动化部署断言 | Partial |

决策依据：路由接入 D3、不变量 3、4；继承前序 D2 的签发与 TLS 义务，修改固定客户端身份。

## Lifecycle Tombstones Fence New Routes Before Container Removal

- 风险：残存容器覆盖终止事实；创建中接入；把暂不可用误报终止；或写入墓碑就宣称已有执行责任结束。
- 前置状态：启用 lifecycle 的 Sandbox Server 持有创建 journal、终止墓碑和 Docker resolver。
- 触发：新 WS Upgrade，或已连接时执行正常终止；Docker 停止可能尚未完成。
- 必须成立：Docker 发现前校验 target、成功 ensure 与墓碑；未知/已终止 404，创建未完成或容器不可用 503，发现/连接错误 502，Node 拨号超时 504。每次重连重新发现唯一目标；实际停止 Node 后已有连接终结；基础设施错误详情只入日志。
- 禁止结果：存活容器绕过墓碑、创建中接受路由、以墓碑代替实际终止证据、缓存旧 IP 后串到新目标。

### 验证义务与证据

| 义务 | 直接证据与缺口 | 状态 |
|---|---|---|
| 成功 ensure、合法 target 与墓碑优先 | cluster `sandbox-server/internal/lifecycle/router_test.go` 的 `TestRouterRequiresSucceededLiveEnsureEvenWhileContainerRemains` 覆盖未知、创建中、成功、墓碑及非法 target | Covered |
| 唯一目标、重新发现与固定错误分类 | cluster `sandbox-server/internal/route/docker_test.go` 和 `internal/server/server_test.go` 的解析失败固定响应测试；详细义务见 Docker 标签解析用例 | Covered |
| 真实 Docker lifecycle 与连接收尾 | cluster `sandbox-server/integration/routed_docker_test.go` 的 `TestDockerLifecycleRoutesRealNode` 经生产服务 ensure、真实 Node hello、terminate 关闭已有连接并拒绝新连接为 404；仅显式配置隔离入口时启用，默认跳过不能记为实跑通过 | Covered |
| 断线与 shutdown 释放双方 | cluster `TestAbruptNodeLossClosesControllerWithInternalError`、`TestShutdownClosesBothSidesWithGoingAway`；完整故障交错与旧 fleet target 有界收敛未由这些测试证明 | Partial |

决策依据：路由接入 D2、不变量 2、5；继承生命周期的墓碑与实际终止证据顺序。

## Coordinated Upgrade Recreates Sandboxes Without Weakening Identity

- 风险：新旧版本混用导致不可达，或为复用旧实例而放宽身份检查、修改 journal、删除数据。
- 前置状态：存在按直连方案创建、固定 Controller 证书的旧沙盒。
- 触发：部署升级为路由方案。
- 必须成立：先正常停止/终止旧实例并保留数据，再协调更新 Controller、Sandbox Server、配置和独立 router 身份，最后正常创建新实例；旧 direct_node_port 配置明确拒绝。
- 禁止结果：复制私钥、改写 journal、删除用户数据、接受旧 peer 证书来兼容混合版本。

### 验证义务与证据

| 义务 | 直接证据与缺口 | 状态 |
|---|---|---|
| 新方案启停和数据保留 | cluster `runtime-control-acceptance.py` 验证启停、重启、Git 基线及强停保留数据；未直接覆盖旧直连版本升级 | Partial |
| 跨版本迁移失败关闭、数据恢复与回滚 | 正式步骤见 cluster `docs/sandbox-routing.md`；完整迁移/回滚演练尚无直接测试 | Missing |

决策依据：路由接入的升级与验证义务、不变量 3、4；完整生产认证与凭据轮换另行处理。
