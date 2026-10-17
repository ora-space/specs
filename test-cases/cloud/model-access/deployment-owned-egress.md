# 部署方共享出网：模型转发、资产下载、解析与策略切换

决策依据：[部署方出口与显式解析 ADR](../../../decisions/cloud/model-access/20261017-deployment-owned-egress-and-explicit-resolution.md)
（`implemented`）。Cloud 共享策略与离线部署入口已有下列受控证据，剩余故障矩阵逐项保留。
状态逐项区分 `Covered`、`Partial` 与 `Missing`，不把受控服务或原有测试扩展为真实企业部署验收。原公网、凭据、冻结和撤销的已有证据继续见
[个人模型访问](personal-model-access.md)；这些证据不能替代企业策略的组合验证。

Cloud 实现与验证入口：[cloud#80](https://github.com/ora-space/cloud/pull/80)。本次证据来自
`internal/egress/{policy,dns,health}_test.go`、`internal/modelgateway/egress_test.go` 和
`integration/deployment_egress_test.go`，后者使用真实 HTTP/PostgreSQL/TCP DNS；模型转发的上游为
受控 TLS 服务，不是现场供应商。部署实现见 [cluster#25](https://github.com/ora-space/cluster/pull/25)：
`egress_config.py`、`egress_topology.py`、`egress_health.py`；直接验收为 `scripts/offline/smoke_egress.py`
及其受控 DNS/TLS/公共 API 场景，控制流故障与双栈拓扑矩阵见 `test_egress.py`。
整包正常路径另由 `smoke.py` 覆盖真实 Node 两轮、成果与释放、认证配置轮换；它不证明真实企业模型已验收。


## Public deployments must retain the existing HTTPS and trusted DNS policy

- 风险：企业扩展悄悄改变默认公网规则或启用系统解析。
- 前置状态：未配置策略路径，或显式 `mode: public`，已有公网连接、有效 grant 及 HTTPS 工作流资产源。
- 触发：保存连接、两协议请求、资产取包和解析故障。
- 必须成立：原公网 HTTPS/DoH 与数字 bootstrap 保持；全部答案校验、数字拨号、原 hostname TLS、无代理和无重定向。
- 禁止结果：默认允许私网、系统/fake-IP 回退，或未启用 enterprise 却要求用户白名单。
- 决策依据：D1、D4、D5。
- 验证义务与证据：`Missing`；需要无配置与 public 配置两条真实保存/受控 TLS 转发及资产下载回归，两调用方使用同一部署 DoH 配置，双栈混合保留地址与 DoH 故障均拒绝。

## Enterprise targets must match an exact canonical host and effective port

- 风险：仅在拨号检查，或 URL/别名/端口规范化差异绕过许可。
- 前置状态：enterprise 仅允许 `model.example.invalid:443`；已有一条不在新许可内的历史冻结绑定。
- 触发：保存/更新连接及使用历史绑定请求；测试无凭据的受控域名解析。
- 必须成立：大小写、单末尾根点与默认/显式 443 等价；同 authority 不同 base path 可用；需要准入的保存解析失败不提交新版本；每次请求重新检查。
- 禁止结果：通配/后缀/别名匹配、8443 自动放行、userinfo/query/fragment 或数字 IP literal 绕过；在 SQL 事务中等待 DNS。
- 决策依据：D1、D2、D5。
- 验证义务与证据：

| 义务 | 直接证据要求 | 状态 |
| --- | --- | --- |
| 共享规范化和精确匹配 | 配置与保存/转发边界覆盖根点、端口、Unicode、畸形 host、IP literal 与不同路径。直接证据：`egress.TestStrictStartupConfiguration` 覆盖规范化、不同路径、端口/Unicode/畸形 host/IP/query；HTTP 保存与代理测试覆盖主要入口，全部变体的入口组合仍待补齐 | Partial |
| 未列出的所有目标均拒绝 | 未许可 host 的公网（含 `7.x` 借用公网编号夹具）、RFC1918/ULA；断言 DNS/上游零调用。直接证据：`TestPolicyTCPRejectsMixedAnswersAndUnlistedHostsBeforeLookup` 计数 DNS；`TestEnterpriseModelAndAssetShareTransportPolicy` 验证模型/Fetch 未许可目标无上游调用；借用公网地址的保存路径见 `TestDeploymentEgressSaveReplayAndPermissionReduction` | Partial |
| 保存与转发独立且一致 | 真实保存接口、版本/幂等冲突、DNS 失败不提交；旧绑定不能绕过收紧许可。直接证据：`TestDeploymentEgressSaveReplayAndPermissionReduction` 用真实 HTTP/PG 与单连接池证明 DNS 在事务外、失败不写新版本、重放不查询；`TestEnterpriseModelAndAssetShareTransportPolicy` 拒绝未许可冻结目标。并发变更矩阵仍待补齐 | Partial |

## Revocation must remain available when egress admission is unavailable

- 风险：DNS 或策略拒绝阻止停用，导致旧绑定不能被永久撤销；纯停用成为修改目标的旁路。
- 前置状态：服务运行且连接有有效绑定，之后 host 被移出当前许可、地址进入禁区或 DNS 故障。
- 触发：分别提交纯 `enabled=false`、删除连接、清除 Key，以及重试、并发冲突和重新启用。
- 必须成立：身份/归属/资源版本/幂等约束继续校验；纯停用接受原样历史配置且零 DNS/上游调用；三种降低权限操作均持久撤销绑定；重新启用重新准入且旧绑定永久失效。
- 禁止结果：因旧 URL 不再符合准入而拒绝纯停用、借停用附带改 URL/模型、幂等重放重新执行出口检查或复活旧授权。
- 决策依据：D2、D6、不变量 6、7；根决策 D1–D4。
- 验证义务与证据：

| 义务 | 直接证据要求 | 状态 |
| --- | --- | --- |
| 降低权限独立于准入 | 真实 HTTP/PostgreSQL/Gateway 覆盖三种操作与三种不可准入状态，断言永久撤销且 DNS/上游零调用。直接证据：`TestDeploymentEgressRevocationSurvivesRemovedTarget` 真实 PG/HTTP 验证目标移除后三种操作永久撤销；`TestDeploymentEgressSaveReplayAndPermissionReduction` 覆盖 DNS 故障。全部状态与凭据 HTTP 入口组合仍待补齐 | Partial |
| 纯停用只降低权限 | 历史 URL 原样可停用，夹带修改配置拒绝，跨用户/版本冲突仍拒绝；启用后旧绑定继续无效。直接证据：`TestDeploymentEgressSaveReplayAndPermissionReduction` 验证纯停用不查询 DNS、夹带改名不能绕过、跨用户拒绝；`TestDeploymentEgressRevocationSurvivesRemovedTarget` 验证重新启用后旧绑定仍撤销 | Partial |
| 幂等不受之后网络变化影响 | 保存成功后改变 DNS/策略，合法相同重试返回原结果；原有幂等冲突仍成立。直接证据：`TestDeploymentEgressSaveReplayAndPermissionReduction` 在真实 DNS 从成功变为 SERVFAIL 后重放原 POST，断言相同 ID 且 DNS 计数不变；幂等冲突与策略变更的组合仍待补齐 | Partial |

## Private exceptions must never override infrastructure and special address denies

- 风险：白名单或宽 CIDR 使模型代理访问 cluster 数据库、宿主或元数据服务。
- 前置状态：同 target 分别省略/开启 `allowPrivate`，可选限定 CIDR；自动采集生成双栈 topology，可有管理员补充项。
- 触发：受控 DNS 或静态映射返回私网、特殊地址或混合 A/AAAA。
- 必须成立：私网默认拒绝；显式允许后仍逐地址检查目标 CIDR、绝对禁区及保留类别；mapped IPv6 同义。
- 禁止结果：跨 target 合并 CIDR、只检查首个答案、跳过坏答案、只支持 IPv4 的宿主拒绝。
- 决策依据：D1、D3。
- 验证义务与证据：

| 义务 | 直接证据要求 | 状态 |
| --- | --- | --- |
| 私网许可与可选 CIDR 独立 | RFC1918/ULA 的缺省拒绝、显式通过、CIDR 缺省和越界；另一个 target 许可无效。直接证据：`TestStrictStartupConfiguration` 与 `TestDenyPrecedenceAndPerTargetRanges` 直接覆盖缺省私网拒绝、显式许可、可选 CIDR 与越界；真实 TLS 私网成功见 `TestEnterpriseModelAndAssetShareTransportPolicy` | Partial |
| 不可覆盖拒绝 | 两调用方在白名单+allowPrivate+覆盖 CIDR 下，cluster 网络、宿主接口/网关/VIP、loopback、link-local 仍拒绝。直接证据：`TestDenyPrecedenceAndPerTargetRanges` 与配置加载测试覆盖基础设施优先拒绝；两类业务入口逐地址类别的组合证据仍待补齐 | Partial |
| 其他保留类别和全答案检查 | unspecified/multicast/documentation/benchmark/转换地址，A 允许+AAAA 拒绝及反向组合，mapped IPv6。直接证据：`TestDenyPrecedenceAndPerTargetRanges` 覆盖 mapped/保留类别，`TestPolicyTCPRejectsMixedAnswersAndUnlistedHostsBeforeLookup` 使用真实 TCP DNS 的 A 安全/AAAA 禁区组合；反向组合仍待补齐 | Partial |
| 拓扑自动采集与可选补充 | `TopologyTests` 覆盖默认桥、多网段、双栈接口/网关、配置池与补充的并集及失败拒绝；`smoke_egress.py` 在真实 Docker/host namespace 核对运行地址属于自动拒绝表。真实双栈 Docker 与外部 NAT 别名变化仍待补齐 | Partial |

## Enterprise resolution must use only the configured resolver or static mapping

- 风险：隐藏的系统、公网回退或 CNAME/附加记录绕过地址检查。
- 前置状态：显式 TCP resolver 数字地址，或 target 非空 staticAddresses；系统 DNS 返回一个可计数诱饵。
- 触发：A/AAAA 查询、TCP 超时、SERVFAIL、NODATA、畸形响应、CNAME 链和静态映射请求。
- 必须成立：动态查询只发往显式 resolver，按原 target 校验最终全部地址；一族 NODATA 可接受，错误不可忽略；静态映射零 DNS。
- 禁止结果：DNS 失败改走系统/公网、静态失败改动态、DNS 控制通路放开同地址模型 HTTP，或从 additional 区取拨号地址。
- 决策依据：D3、D4。
- 验证义务与证据：

| 义务 | 直接证据要求 | 状态 |
| --- | --- | --- |
| 显式解析及无回退 | 真实受控 TCP DNS 服务与系统诱饵计数，超时/失败/单族错误拒绝，NODATA 语义。直接证据：`TestTCPDNSFamiliesCNAMEAndFailureBoundaries` 真实 TCP DNS 覆盖双族、单族 NODATA/失败；`TestDeploymentEgressAssetDNSRejectionLeavesNoDocuments` 证明实际 Fetch 与模型调用使用该 resolver。系统诱饵计数及真实超时矩阵尚缺 | Partial |
| 有界协议校验 | CNAME 深度/循环、响应关联、超长/畸形/附加记录；地址验证发生在任何 dial 前。直接证据：`TestTCPDNSFamiliesCNAMEAndFailureBoundaries` 覆盖查询 ID、截断、additional-only、CNAME 跟随/循环；深链、超长报文与所有畸形响应仍待直接证据 | Partial |
| 静态映射与 TLS 身份 | 无 resolver 的全静态配置、混合配置、静态全地址检查；HTTP Host/SNI/证书仍为原域名。直接证据：`TestEnterpriseModelAndAssetShareTransportPolicy` 使用全静态配置、真实非回环 TLS 上游，验证原 SNI、正常证书和无信任证书拒绝；混合配置组合仍待补齐 | Partial |

## Verified dialing and connection reuse must preserve the current target boundary

- 风险：新拨号未验证全部地址，或连接无限复用、HTTP/2 合并、代理与重定向绕过目标边界。
- 前置状态：真实受控双栈 DNS/TLS 上游，有效授权，已建立 HTTP/1 或 HTTP/2 连接。
- 触发：新请求、新拨号、达到连接最大存活时间或空闲超时、后续解析地址变化、证书错误、3xx 和环境代理配置。
- 必须成立：每请求按启动快照检查目标；新拨号完整验证后只拨数字地址；连接按实现固定的有界存活时间/空闲超时回收，回收后新拨号重新检查；原 hostname TLS 不变。
- 禁止结果：验证后系统再次解析、无限复用旧连接、跨 host HTTP/2 合并、TLS 降级、跟随跳转、向环境代理发送认证头或回收后自动重发生成。
- 决策依据：D3、D5。
- 验证义务与证据：`Partial`；`TestConnectionLifetimeClosesActiveAndIdleSockets` 通过真实到期回调关闭活跃/空闲连接，`TestPolicyPreventsReusedConnectionToUnlistedAuthority` 验证复用后的逐请求门禁，受控 TLS 测试覆盖代理/重定向/证书。仍需计数真实 DNS/dial/HTTP 边界，覆盖新 dial、keep-alive/HTTP/2 的最大存活与空闲回收、回收后混合坏地址拒绝、错误证书、3xx、HTTP/SOCKS 代理与有限脱敏错误；纯匹配函数单测不够。

## Policy configuration must take effect through restarting both services

- 风险：仅改文件却误以为策略已生效，旧进程/池仍发送，或加载失败退回宽许可、摘要不一致未被发现。
- 前置状态：旧快照已加载且存在活跃 SSE、HTTPS 资产下载与空闲连接；新快照移除目标或收紧 CIDR/topology。
- 触发：运行中仅改文件，或经部署入口停止两服务、换文件、再启动；配置加载失败、摘要不一致及显式回滚。
- 必须成立：文件变动不热生效；确认两个服务旧进程退出后才换文件/启动，旧流与所有池已终止；有效新配置在启动时加载，失败进程不启动；`ora verify` 比对两端摘要，缺失/不一致报错。
- 禁止结果：只关 idle 池或不确认退出就替换文件、无效配置自动回退 public/旧配置、摘要不一致仍宣称部署校验成功、自动重发中断的生成。
- 决策依据：D1、D3、D6。
- 验证义务与证据：

| 义务 | 直接证据要求 | 状态 |
| --- | --- | --- |
| 配置加载失败不启动 | Cloud 配置单元/二进制启动探针与 `ApplicationTests` 覆盖无效配置；`smoke_egress.py` 验证未知输入字段不动旧服务/快照，启动失败后双服务均停止。完整错误类别的双进程矩阵仍待补齐 | Partial |
| 文件仅在启动加载 | `TestSnapshotAndCanonicalDigest` 与实际 `smoke_egress.py`：输入修改后 verify 拒绝未应用状态，经 apply 改为静态解析并移除旧目标，两个容器身份变化而镜像身份不变，新请求按新快照判定 | Covered |
| 停止旧流和池 | `smoke_egress.py` 实际移除两个已停止的旧容器、重建同一镜像，受控慢资产下载取消且零文档结果；`ApplicationTests` 断言停止失败不换文件。活跃模型 SSE 的进程停止组合仍待补齐 | Partial |
| 摘要比对 | `TestHealthExposesOnlyPolicyIdentity`、`ApplicationTests` 与 `VerificationTests`；实际 `smoke_egress.py` 注入单端不同策略使 ora verify 失败，再经真实 apply 恢复，覆盖同代成功/不一致拒绝及输入/生成快照变更识别；多副本真实部署矩阵待补齐 | Partial |
| 回滚与拓扑变更 | 工具使用同一采集/停机/替换/启动路径，`ApplicationTests` 连续应用不同输入且未设 complete 门槛；实际 smoke 显式修复失败配置后恢复。现场拓扑变更与完整旧配置回滚的实测组合仍待补齐 | Partial |

## Current policy must constrain frozen bindings without reviving revoked grants

- 风险：冻结被误读为永久网络许可，或策略重新允许后复活已撤销凭据。
- 前置状态：有效冻结绑定和另一条已永久撤销的绑定；之后修改个人 URL/Key/槽位或收紧部署策略。
- 触发：旧运行请求、清除/停用/删除连接，再重新允许目标和启用连接。
- 必须成立：有效旧绑定仍用原 URL/模型/Key 并受当前出口限制；准入失败不改写绑定；恢复许可只适用于仍有效的授权；永久撤销永不复活。
- 禁止结果：旧运行自动换新 Key/URL/槽位、把临时网络拒绝误写为永久撤销、部署配置读取或注入真实 Key 给 Cloud/Node。
- 决策依据：D5、D6；根决策 D1–D4 与槽位后续 ADR。
- 验证义务与证据：`Partial`；`TestDeploymentEgressRevocationSurvivesRemovedTarget` 验证企业目标移除后永久撤销及重新启用不复活，`TestDeploymentEgressAssetDNSRejectionLeavesNoDocuments` 验证网络拒绝不写永久撤销。仍需完整真实数据库+Gateway 贯通修改/撤销/恢复许可矩阵、原绑定字段核对和解密边界断言，不能只引用原永久撤销数据库测试。

## Model forwarding and asset downloads must share one deployment policy

- 风险：资产下载器保留硬编码公网 DoH/独立白名单而无法取内网包，或借签名 query、对象存储/池复用绕过共同准入。
- 前置状态：Cloud 与 Gateway 启动时加载同一 `deployment-egress.json`，`ora verify` 摘要一致；受控 HTTPS 模型与资产源返回双栈地址，目录含带签名 query 的合法 `.orax` URL 和 object key。
- 触发：模型 POST 与后台资产 GET 分别访问白名单内/外目标，更换部署 resolver/静态映射，返回混合禁区地址、3xx、错误证书或超限包，并在取包期间应用新策略。
- 必须成立：同一 host:port 集合、目标私网/CIDR、DNS 与拓扑规则用于两者；资产原路径/签名 query 被保留，模型 baseUrl 仍拒绝 query；无个人模型认证头进入资产 GET。资产大小/摘要/事务与包级失败语义保持，切换关闭同时停止两类出网。
- 禁止结果：只有模型遵守策略、资产默认放行未许可公网/7.x、使用系统或写死 Cloudflare DNS、忽略 AAAA 坏答案、cluster HTTPS 源特例、把拒绝的 HTTPS 改为 object-key 取包、泄漏 query/Key 或取消后留下部分导入。
- 决策依据：D1、D3–D7、不变量 1、5、8、9；[工作流资产 ADR D3–D5](../../../decisions/cloud/plugin-marketplace/20261016-workflow-plugin-assets.md)。
- 验证义务与证据：

| 义务 | 直接证据要求 | 状态 |
| --- | --- | --- |
| 两者使用同一集合与解析来源 | Cloud 真实 handler/Fetch 测试加 `smoke_egress.py` 的实际 Cloud 保存与后台资产导入：共同 TCP DNS、许可内 HTTPS 取包成功、未许可目标在 DNS 前拒绝；改为静态解析/移除资产目标后两服务按同一新摘要生效。部署栈内有效 grant 模型转发完整矩阵仍待补齐 | Partial |
| 全答案与不可覆盖拒绝 | 两入口分别覆盖 A/AAAA 混合禁区、cluster/宿主/loopback/link-local；新拨号完整校验，连接存活/空闲回收后重新解析，不建立禁止地址连接。直接证据：共享准入的全答案/拒绝优先测试及真实 Fetch/模型 DNS 拒绝已有直接证据；两入口全部地址类别和连接回收后的 DNS 变化矩阵尚缺 | Partial |
| URL 与凭据边界不同但准入一致 | Cloud TLS 测试保留签名 query/原 HTTP authority、无模型认证头、TLS/3xx/摘要失败；实际 `smoke_egress.py` 的源端再次断言签名 query 和无个人模型认证头，导入生成真实工作流。错误 hostname 与完整日志矩阵待补齐 | Partial |
| 资产原完整性与失败边界 | 真实 Fetch+PostgreSQL 验证 16 MiB/10–30–60 秒、摘要失败先于解析、包级失败零文档结果及下次触发重试；导入 revision 围栏不被策略切换绕过。直接证据：`TestDeploymentEgressAssetDNSRejectionLeavesNoDocuments` 实际 Fetch+PG 证明 DNS 拒绝为包级失败且零文档结果；受控 TLS Fetch 校验摘要。原大小/超时和导入 revision 证据仍见资产用例，不扩大为新策略全矩阵 | Partial |
| 内部对象存储不是 HTTPS 旁路 | object key 只走固定私有客户端且不能携带 host；同一 cluster 服务若作为 HTTPS URL 则被拒，无自动降级 | Missing |
| 共同配置重启生效 | 实际 `smoke_egress.py` 验证目标/DNS 改变、两个旧容器消失、镜像一致、未完成资产无文档结果、单端摘要错配拒绝、启动失败停止两端和显式恢复；新主机、多副本及活跃模型流组合仍待补齐 | Partial |
