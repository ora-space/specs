# 沙盒 Node 直连用例的历史入口

现行管理通道采用[已批准的路由接入决策](../../../decisions/cluster/sandbox-server/20261016-controller-connects-through-sandbox-websocket-router.md)。
直接验证义务与证据已改挂到[路由接入核心用例](routed-node-connection.md)。本文件保留既有路径和锚点，
不再把旧直连方案作为当前系统承诺；原始内容可从 Git 历史查阅。

## Sandbox Node Accepts Only The Pinned Controller Over Mutual TLS

此历史锚点曾要求 Node 固定接受 Controller，并由 Controller 直拨 Node。后续决策改为 Node 固定接受
router，Controller 认证 Sandbox Server，两条连接各自使用 TLS。现行身份义务见
[Two TLS Hops Authenticate Distinct Management Peers](routed-node-connection.md#two-tls-hops-authenticate-distinct-management-peers)。
Node 证书用途、材料不泄露、明文拒绝等继承义务及其原有缺口一并保留在新用例中。
