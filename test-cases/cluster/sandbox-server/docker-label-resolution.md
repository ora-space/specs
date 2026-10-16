# Sandbox Server 的 Docker 标签解析

依据[已实现的 Docker 标签解析决策](../../../decisions/cluster/sandbox-server/0-docker-label-resolver.md)。
本文件登记 Sandbox Server 把 `ate-target-actor` 解析为 Docker 容器中 Node 地址的承诺；中继的透传、
close code 与 ping/pong 语义见 cluster `README.md`，Controller 侧的失败分类见
[传输核心用例](../../node/protocol/frame-channel-transports.md)，本文不重复登记。证据来自 ora-space/cluster 仓库（核对基线 `aef10f3`），路径相对该仓库根目录。
Cloud 模式的 Controller 经此路由接入；lifecycle 前置检查、逐跳 TLS 与终止语义见
[路由接入用例](routed-node-connection.md)。本文的 ws 地址是 resolver 的发现结果，启用管理 TLS 时由中继以 wss 连接。

## Docker Target Resolves To Exactly One Running Node On The Sandbox Network

### 风险

解析把连接送到错误的 Node：同一 sandbox ID 的多个容器被任选其一，或者使用宿主机发布端口、其他网络
上的地址，使连接到达错误的后端；或缓存了容器重建前的旧地址。

### 前置状态

Sandbox Server 以 `-docker-network <沙盒网络>` 运行；Docker 中有带 `space.ora.sandbox.atespace`、
`space.ora.sandbox.id`、`space.ora.node.port` 标签的 Node 容器。

### 触发

Controller 以 `ate-target-actor: <atespace>/<sandbox-id>` 发起 WebSocket Upgrade。

### 必须成立

- 恰好一个匹配且运行中、已接入沙盒网络、端口标签合法时，Sandbox Server 拨号
  `ws://<该容器在沙盒网络上的 IP>:<端口>` 加原请求路径与 query。
- 容器重启或重建导致 IP 变化后，下一次 Upgrade 使用新地址。
- 解析只调用只读 Docker API。

### 禁止结果

同一 target 匹配多个容器时任选其一；使用宿主机发布端口或其他网络的地址；使用缓存的旧地址；
解析改变任何容器状态。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 唯一匹配解析为沙盒网络 IP 与标签端口 | `sandbox-server/internal/route/docker_test.go` 的 `TestDockerResolvesTheSingleRunningContainerOnTheSandboxNetwork`：容器同时接入沙盒网络与其他网络，断言只用沙盒网络 IP 与端口标签；`sandbox-server/internal/docker/docker_test.go` 的 `TestListContainersFiltersByLabelsAndDecodes` 在真实 unix socket 上断言 `/v1.41/containers/json`、`all=true` 与标签过滤条件 | Covered |
| 多个匹配一律拒绝 | `sandbox-server/internal/route/docker_test.go` 的 `TestDockerResolveFailures` 以两个同标签运行中容器断言 `ErrSandboxUnavailable`；解析失败时 server 在拨号前返回（`sandbox-server/internal/server/server_test.go` 的 `TestDiscoveryFailuresMapToFixedResponses`） | Covered |
| 不缓存 | `sandbox-server/internal/route/docker_test.go` 的 `TestDockerResolveReflectsEachQuery`：两次解析之间改变 IP，断言第二次返回新地址 | Covered |
| 只读 | 类型层：`route.NewDocker` 只接受 `ContainerLister`（仅 `ListContainers`），解析无法调用写 API；尚缺记录 Engine 请求方法的直接断言 | Partial |
| 真实 Docker 端到端路由 | cluster `sandbox-server/integration/routed_docker_test.go` 的 `TestDockerLifecycleRoutesRealNode` 经生产服务创建真实 Node，验证 routed hello、实际终止与后续 404；未单独注入全部 Docker 发现故障 | Partial |

### 决策依据

Docker 标签解析 D1–D3、D5，不变量 1、3、4、5；传输 D6。

## Resolution Failures Keep Not Found Distinct From Unavailable

### 风险

容器存在但暂时无法路由时报成 `404`，Controller 与未来的 Provider 会把它当作沙盒已销毁；或者把
Docker 错误原文、容器 ID、网络名写进响应体，泄露基础设施细节。

### 前置状态

Sandbox Server 以 Docker Resolver 运行。

### 触发

Controller 请求的 target 分别处于：无匹配容器；匹配容器未运行；匹配容器运行中但未接入沙盒网络或
端口标签非法；多个匹配；Docker API 不可达或报错。

### 必须成立

- 无匹配容器返回 `404`；存在匹配容器但无法路由（含多个匹配）返回 `503`；Docker 自身失败返回
  `502`。
- 所有情况都在拨号 Node 前以 HTTP 状态结束，不接受 Controller 的 Upgrade。
- 响应体是固定的简短原因；详情只进结构化日志。
- 启动时 Docker 不可达则拒绝启动；运行中 Docker 失联只影响当次请求。

### 禁止结果

仅凭容器暂时不可路由返回 `404`；响应体包含 Docker 错误原文、容器 ID 或网络名；Docker 失联导致进程退出。
启用 lifecycle 时，终止墓碑优先于存活容器并返回 `404`，这是路由接入决策的明确覆盖规则。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| D3 映射表的每一行 | `sandbox-server/internal/route/docker_test.go` 的 `TestDockerResolveFailures` 覆盖无匹配、未运行、不在网络、端口标签非法、多个匹配与 Docker 报错的错误分类；`sandbox-server/internal/server/server_test.go` 的 `TestDiscoveryFailuresMapToFixedResponses` 断言三类错误分别为 `404`／`503`／`502` | Covered |
| 响应体不含内部细节 | `sandbox-server/internal/server/server_test.go` 的 `TestDiscoveryFailuresMapToFixedResponses`：错误携带容器与网络细节，断言响应体等于固定短语 | Covered |
| 启动自检 | `cmd/sandbox-server` 已实现 `_ping` 自检与互斥校验，尚无测试 | Missing |
| 只接受 unix socket | `sandbox-server/internal/docker/docker_test.go` 的 `TestNewRejectsNonUnixHosts` 断言 `tcp://`、`http://`、空路径与裸路径都被拒绝 | Covered |

### 决策依据

Docker 标签解析 D3、D5、D6，不变量 2、6、7；传输 D7。
