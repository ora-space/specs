---
status: proposed
---

# 工作区内容经内部 Git Server 单向下发：读取授权由服务端按部署自有的容器状态判定，Node 不持有任何 Git 凭据

本 ADR 交付"真正的 Clone → Workspace → Agent"所需的唯一内容通道：由已获批准持有 `broker-resolver` 身份的凭据执行
单元 C 把用户上游仓库镜像进内部 Git Server；Cloud 在镜像发布并核验之后，才把 Workspace 的克隆输入指向该镜像中
指向目标提交的那个 ref；Node 用**既有的、匿名的 HTTPS clone 通道**执行 Clone（不出示证书、不出示凭据、不出示任何
自述字段）；Git Server 面向沙盒的监听不接受任何请求自述，而是在**每个请求内**把连接来源向部署自有的容器状态解析，
得到"这是哪一个运行中的沙盒容器"，再由该容器创建时由控制面写入的标签与 Cloud 的权威状态决定它只能读本 Workspace
的镜像。关键行为变化是：内部只读读取不再需要任何客户端秘密，因此 Node 与 Agent 工作负载都不持有 Git 凭据、客户端
证书私钥或长期令牌。影响面是 Cloud（新增 Git Server 程序与镜像发布）、cluster（Compose 接线、沙盒容器的 CA 引用、
来源解析接口）；`desktop/` **不修改**。

本 ADR 当前为 `proposed`：它记录的是尚待裁决的方案、其技术前提与替代方案，供 reviewer 在没有实现代码的情况下纠偏。
第 1–4 阶段的落地顺序在授权开启前不改变任何现有行为，是对应的评审对象。本 ADR 不承担迁移义务：今天带凭据的 Clone
在每条路径上都被 `repository_credential_executor_unavailable` 拒绝，不存在需要迁移的既有克隆路径，也不解除生产
fail-closed。兼容性边界：不修改已批准决策的正文——凭据执行归 C（`node/repository/20261012-credential-executor-belongs-to-the-current-product.md`）、
Desktop 严格只读、凭据物化不经 argv／环境（`node/repository/20261010-controlled-node-git-authentication-isolation.md` D2、D11）
全部保持原文不变；本文只替代 `cloud/git-server/0-sandbox-scoped-git-server-over-https.md` 中"由 Node 出示 git-access
客户端证书"的那一部分（该文同为 `proposed`）。

## 继承与修改

| 前序约定 | 本文处理 |
| --- | --- |
| [checkout 交付到 Workspace](../node/repository/20261012-checkout-delivery-into-the-workspace.md)（proposed）T4 裁决：不设中间通道、`desktop/` 保持严格只读、生产 fail-closed 不解除 | 继承 T4 的实质：仍然不修改 desktop、仍然 fail-closed、不引入临时过渡通道；修改其 R1／不变量 13 的措辞——"读取授权由受控客户端身份给出"改为"由服务端可证明的来源容器身份 + 逐请求服务端判定给出"（D3、D5） |
| [Git Server 根决策](0-sandbox-scoped-git-server-over-https.md) D1：Cloud 仓库中的独立程序，只提供 smart HTTP | 继承，不变（D1） |
| 同上 D2：认证只认沙盒的 git-access 证书 | **修改**：沙盒面向的读监听不要求也不接受客户端证书；认证目标由服务端解析来源容器得出，`git-access` 身份不再签发（D3）。该修改由本次裁决"Node 不持 git-access 客户端证书私钥"直接要求 |
| 同上 D3：每个请求按证书目标、声明的执行与 Cloud 当前状态判定 | **修改**：读方向不再接受 `Ora-Execution-Id` 一类声明执行的请求头作为授权输入；判定目标只由解析出的 Workspace 决定（D5） |
| 同上 D5：Node 执行 Git 访问，经 `-c http.sslCert/sslKey/sslCAInfo` 与 `-c http.extraHeader` 传入证书 | **修改**：Node 不执行任何带凭据或带身份的 Git 访问；内部只读通道不带凭据、不带证书、不带 header（D4、D7、D9） |
| 同上 D6：两个监听，仓库监听只面向沙盒网络，ref 查询监听只面向默认网络 | 继承（D2、D10） |
| 同上 D7：本阶段不改与用户上游仓库的关系 | **修改**：本文正是该文"本决策未解决的问题"里挂起的"镜像上游、Node 改从 Git Server clone"那一篇；上游凭据仍只由 C 持有，用户上游的写方向仍不开放（D1、D8） |
| [受控 Node 的 Git 认证隔离](../../node/repository/20261010-controlled-node-git-authentication-isolation.md)（approved）D1–D4、D11 与不变量 1–13：W／C／M 三身份、凭据只对 C 开放、物化不经环境与 argv、`repository identity` 决定范围 | 继承，原文不变。本文不新增任何凭据、任何物化位置、任何身份；内部通道不物化秘密（不变量 7） |
| [凭据执行器归属](../../node/repository/20261012-credential-executor-belongs-to-the-current-product.md)（approved）D1、D2：C 是当前产品内的独立部署单元；Desktop 严格只读、不承载凭据执行 | 继承，不变：镜像上游的带凭据 Git 执行仍然只属于 C，且发生在 C 自己的容器里（D1、D2） |
| [Secret Store](../identity-access/20261010-user-managed-git-secret-store.md)（approved）D3：只有服务身份可兑换，单次授权、原子消费 | 继承，不变 |
| [模型客户端身份](../../cluster/sandbox-server/20261010-model-client-identity-replaces-shared-model-key-injection.md)（approved）D1、D2：按租户、Workspace、代次另签 `clientAuth` 身份，材料只作 Docker 私有启动材料、由 entrypoint 存入管理 tmpfs | 该决策本身不变；本文**不使用**同一机制承载 Git 读取，理由见 D4 与"为什么不是这些替代方案"：该材料必须对执行 git 的身份可读，而该身份与工作负载同 uid |
| [仓库凭据连续性](../project/20260927-repository-credential-continuity.md)（approved）D4：执行端真正按引用选择，不回退全局静态凭据 | 继承：镜像路径同样不接受口令、静态 token 或匿名访问上游（不变量 9） |
| [Node 的最小 clone 执行契约](../../node/repository/20260918-minimal-clone-execution-contract.md)与 Node Cloud 策略：只允许 HTTPS，禁止 SSH 与静态凭据 helper | 继承，不变；本文不新增任何凭据来源，反而删去了一种（D9） |
| [沙盒网络与 Docker 标签解析器](../../cluster/sandbox-server/0-docker-label-resolver.md)、[Agent 运行沙盒环境](../../cluster/sandbox-server/20260928-agent-run-sandbox-environment.md)：沙盒网络是非 internal bridge，容器不发布端口，控制面按标签解析沙盒 | 继承，并把它从"用于路由"扩展到"用于读取授权"（D3、D10） |

## 决策

### D1：内容通道只有一条，方向单一

C 以受控凭据从用户上游仓库取得内容，发布为内部 Git Server 上的不可变镜像；Cloud 把 Workspace 的克隆输入指向该
镜像；Node 用现有 HTTPS clone 执行 Clone，内容落到 Workspace 卷；Agent 的会话工作目录就是该检出（`CheckoutDirectory::workspace_cwd`
把 workspace cwd 与 home 都指向 checkout）。除此之外没有第二条通道：不新增对象存储路径，不让 Agent 直接访问外部
私有仓库，不为本功能修改 `desktop/`。

为什么只要一条：读取授权需要且只需要一个判定点。通道越多，"谁能读什么"就要在越多地方保持一致；而今天唯一的
替代路径（把上游凭据交给工作负载）已被已批准决策禁止。代价如实记录：每个 Workspace 的 Clone 都要先经过一次
镜像，即使同一 Project 的第二个 Workspace 也要等镜像发布完成（缓解方式见 D6 的键复用）。

### D2：镜像的写入方只有 C；沙盒面向的接口没有写权限

- 镜像的创建与更新只由 C 完成——它已是被批准持有 `broker-resolver` 身份、独立部署、独立授权、独立审计的执行单元。
  Git Server 不持有上游凭据，也不代表用户对上游做任何写操作。
- 沙盒面向的监听只实现 `git-upload-pack`（读取）。写方向的实测行为**不是一个统一响应**，本文按测量如实记录，
  不假装两者一致：`info/refs?service=git-receive-pack` 走到与读方向同一个决定点，被判为"写方向不服务"而返回
  `403`；`POST .../git-receive-pack` 因为该监听**根本没有注册这条路径**由路由返回 `404`。两者都不是"写被放行"，
  也不泄露任何仓库状态。
- **"不存在"与"无权"不可区分只在读方向被要求**（D5 的表），由读方向那一个判定点统一承担。写方向不提供这条性质，
  因为它没有可服务的对象——"这个仓库存不存在"对一条不服务的写路径不是问题，把两者做成同一响应只会增加一个
  无信息量的分支。评审时不要把这条偏差读成"读方向也可以区分"。
- 内部交付接口不因为"来源是沙盒"而获得任何写权限，也不因为"来源是本 Project 的 Workspace"而获得任何写权限。
- 写方向（把成果推回内部仓库或用户上游）仍属 `node/repository/20261012-push-execution-authority.md`（proposed）
  的范围，本文不放行、不预判其身份方案。

为什么把写权限从内部通道彻底移出：只读接口一旦能写，D3 的来源判定就从"读范围"变成"写范围"，一次判定错误即可
改写镜像，而镜像又被当作可信的克隆源。把写面留给独立路径，使本文的失败后果上界是"读到不该读的内容"。

### D3：读取身份来自部署自有的容器状态，不来自请求

规则：请求到达沙盒面向的监听时，服务端取连接的对端地址，在**同一次请求内**向部署自有的容器状态解析三件事——
该地址当前是否属于一个 `running` 的沙盒容器、该容器在本网络上的地址、以及该容器创建时由控制面写入的标签
（`space.ora.workspace.id`、`space.ora.sandbox.id`、`space.ora.effect.id`、`space.ora.sandbox.atespace`）。
解析结果给出 Workspace；租户与 Project 由 Cloud 的权威状态给出（Workspace → Project → Tenant）。请求的路径、
`Host`、查询串与任何请求头都**不参与**身份判定（路径只用于在已授权的范围内定位资源，见 D5）。

标签字面量已收敛到 `sandbox-server/internal/labels` 一处：它们同时被路由解析、生命周期与本文的读取授权读取，
所以是授权输入而非装饰，重命名或增加必须是一次改动被所有读者看到。四个标签必须全部存在且非空——缺少任一个即
"这不是控制面创建的沙盒"，而不是"租户为空字符串"。注意 `space.ora.effect.id` 与 `space.ora.sandbox.id` 今天由
生命周期写入**同一个值**，因此两者不是彼此独立的证据。

为什么这在本次部署的具体形态下是可证明的，而不是"凭来源 IP 推断租户"（以下为只读核对的技术事实）：

1. **沙盒容器无法伪造源地址。** 容器由 `sandbox-server` 创建，创建体把能力集合固定为 `CapDrop: ["ALL"]` 加
   `CHOWN、DAC_OVERRIDE、FOWNER、SETUID、SETGID、KILL`，并设 `no-new-privileges:true`
   （`sandbox-server/internal/docker/docker.go` 的 `CreateContainer`）。集合中**没有 `NET_RAW` 与 `NET_ADMIN`**，
   因此容器内任何进程——包括以 root 身份运行的进程——都无法打开原始套接字、无法修改接口地址或 MAC、无法绑定
   非本地地址。对端地址是内核给出的、容器无法选择的值。
2. **容器不发布端口，且请求不经 NAT。** `ContainerSpec` 的注释即写明容器"never publish ports"，只在一个具名
   bridge 网络（`${ORA_SANDBOX_NETWORK:-ora-sandboxes}`，由 `Taskfile.yml` 的 `docker network create` 创建）上；
   同一网段的容器与服务之间是直接 L2 交付。因此服务端看到的是该容器自己的地址，而不是网关地址。
3. **标签是控制面写的，容器改不了。** 容器标签在创建时由生命周期写入（`sandbox-server/internal/lifecycle/effects.go`），
   容器自身没有 Docker API 访问权，改标签等价于取得部署 root。
4. **解析必须实时。** 地址由 Docker 从池中动态分配，容器重建后会被复用，因此每次请求都重新解析、不缓存、不落
   一张"地址 → Workspace"的表。`sandbox-server/internal/route/docker.go` 的既有解析器就是同一原则（每次查询都
   重新问 Engine，注释写明 "Nothing is cached"），本文沿用它的形态而不是新造一套。

必须拒绝（且不得静默放行）的情况：地址解析不到容器；解析到已停止的容器；解析到不是沙盒容器的部署服务；容器标签
的 Workspace 与 Cloud 状态不符；对端地址是网关地址（说明请求经过了 host 端口发布或 NAT 路径）。因此本判定**不依赖
L2 隔离**：即便沙盒网络今天是非 internal bridge、沙盒可达同网的其他服务，判定也只看"地址 → 运行中的容器"，并
在解析失败时拒绝。

已实测的两条具体事实（2026-10-12，隔离网络上的真实容器）：

- **已停止的容器永远不会按地址被解析到。** Engine 在容器离开网络时清空其地址，`/containers/json` 为停止的容器
  报告空地址；因此"解析到已停止的容器"在实现上表现为"解析不到容器"，两者都必须拒绝。
- **对端地址是网关地址时无法解析。** 从宿主或经 NAT 到达的连接，服务端看到的对端是网桥网关（实测为
  `172.24.0.1`），它是网络自身的地址而非任何容器的地址，因而必然落进"解析不到容器"。这条同时说明：为沙盒面向的
  监听发布 host 端口不会让请求"看起来合法"，它只会让请求变成不可解析。

已知残余，必须在评审中看见（不由本文解决）：

- **部署服务与沙盒容器共用同一条网桥，其中四个是被登记的部署服务。** 实测主栈的 `ora-sandboxes` 上除了 8 个
  沙盒容器，还有 `controller`、`sandbox-server`、`objectstore`、`plugin-fixture`（`premise-check` 按标签把它们
  与工作负载分成两类报告）。实测能力集合：8 个沙盒工作负载为 `CapEff=0x…eb`，即 D4 约定的六个能力，不含
  `NET_RAW`／`NET_ADMIN`；`controller` 与 `objectstore` 为 `CapEff=0`；而 **`sandbox-server` 为
  `CapEff=0x…a80425fb`（Docker 默认集合，含 `NET_RAW`）并按设计挂载 `/var/run/docker.sock`**，`objectstore`
  发布了端口。因此"同网桥上没有任何进程能伪造源地址"这条更强的性质在今天的部署里**不成立**——不是因为它不适用，
  而是因为网上确实有一个能伪造的进程；成立的只是"沙盒工作负载不能伪造"（8 个全部通过断言）。
- **`premise-check` 的两个退出码都必须保留在证据里。** 同一次只读运行：默认报 `exit=0`（工作负载全部合规），
  加 `-require-peers-constrained` 报 `exit=1`（4 个部署服务未受该前提约束）。**非零的那一次正是这条残余的度量**，
  不得因为"检查通过了"只记前者。它也不应被当成可修掉的缺陷：即便把 `sandbox-server` 的能力降到 `CapEff=0`，
  它仍持有 socket，strict 仍会失败——见下一条。
- **已修正的一项：未认证的 node relay 不再对工作负载可达。** 上述残余里唯一可直接被工作负载利用的是
  `sandbox-server` 的 node relay：修正前实测，从沙盒容器 `GET http://<sandbox-server>:18000/healthz` 返回 `200`
  ——明文、不认证任何调用方，仅凭请求头里的 `ate-target-actor` 就能把 WebSocket 路由到**任意**沙盒的 Node。
  修正方式不是给它加认证，而是把它从工作负载可达的网段上移开：两个监听的绑定地址改到一条工作负载**不在其上**的
  internal 控制段，同时把 `sandbox-server` 自己的能力集合降到 `cap_drop: [ALL]`。修正后实测
  （`scripts/network-isolation-acceptance.sh`；同一个镜像、同一份管理材料、同一个请求，只改监听地址与所在网段
  这一个变量）：`workload -> relay 200 before, unreachable after`，而持有 socket 的容器仍能创建容器、仍能拨入
  工作负载网段。**这条修正落在** cluster 栈的一个隔离 overlay（`compose.network-isolation.yaml`）与本轮验收脚本
  里，**主栈 `tt` 的 compose 未改动**；该 overlay 是否进入主栈属于部署裁决，不在本文范围。注意它改变的是"什么地址上
  能被连到"，不是"谁能伪造源地址"——下一项不受它影响。
- **不可消除的残余：Docker socket。** `sandbox-server` 按设计必须持有 Engine socket：它就是创建容器、写标签的
  那一方，而标签正是 D3 的授权依据。所以"同网桥上没有任何进程能伪造源地址"这条更强性质在**任何**满足 D10 的
  部署里都不成立——去掉 socket 等于放弃创建容器的能力。本文的可证明性因此以"控制面本身处于信任边界内"为前提；
  把解析放在已持有 Engine 客户端的 `sandbox-server`（D10）本就承认了这一点，因为能创建容器的组件已经能创建带
  任意标签的容器。这一残余必须被如实登记，不得表述为隔离。
- **被信任的调用方自述了"对端地址"。** 观测连接对端的是 Git Server，它把地址报告给 `sandbox-server`；身份不是
  调用方自述的（由部署自有状态解析），但**地址本身**是。一个被攻破或有缺陷的 Git Server 可以就任意地址提问，
  进而把别的容器的身份当成自己的。调用方已由**来源监听自己的**固定叶子 `git-server.pem` 限定（见 D10），因此
  这条残余等价于"Git Server 的完整性"，但它确实存在：唯一能彻底消除它的形态是让 `sandbox-server` 自己终结沙盒
  面向的连接（即把只读读取代理进它自己的进程），本文不采用，因为它把内容通道的吞吐与可用性也压进同一个信任根，
  代价更大。

### D4：不可伪造性是一条必须固化的前提，随容器能力集合一起被验证

- 规则：沙盒容器的能力集合（drop ALL 加上述六个，其中**不得**出现 `NET_RAW`／`NET_ADMIN`）、`no-new-privileges`，
  以及"不发布端口"，从"某一行创建参数"提升为**受保护前提**：任何放宽都必须先修订本文并同步修改 D3 的判定。
- 为什么：D3 的可证明性完全建立在这条之上。今天它不是不变量，只是一行参数；若某次改动为了别的功能把 `NET_RAW`
  加回，来源身份会在无人察觉的情况下变成可伪造。
- 验证义务：检查必须能对**实际创建的容器**断言该能力集合（例如对 `docker inspect` 的 `HostConfig.CapDrop`／`CapAdd`／
  `SecurityOpt` 断言），而不是只断言源码常量。该检查已实现为 `sandbox-server/cmd/premise-check`：它按网桥列出容器、
  逐个读取宿主配置，把"带沙盒标签的工作负载"与"部署服务"分成两类，工作负载违规即退出码 1。它断言的不止能力集合：
  还要求 `CapDrop` 含 `ALL`、`SecurityOpt` 设置 `no-new-privileges`、没有发布端口、没有挂载 Docker/containerd socket
  （挂载 socket 即可创建一个带任意标签且带 `NET_RAW` 的容器，等价于绕过本条）。读取真实容器立刻暴露了两件源码常量
  断言不会暴露的事：Engine 把能力名回读为 `CAP_CHOWN` 而创建请求写的是 `CHOWN`（直接比较字面量会把合规容器报成违规），
  以及部署服务与沙盒容器共用同一条网桥（见 D3 的已知残余）。
- **同一条前提也覆盖 `sandbox-server` 自己的监听。** 沙盒网络上的"未认证中继不可由工作负载访问"与容器能力集合
  是同一件事的两半：前者过去不成立（实测从沙盒容器访问 node relay 的 `:18000/healthz` 得到 `200`，见 D3 的已知
  残余），现已通过把该监听移出工作负载可达的网段修正，并由 `scripts/network-isolation-acceptance.sh` 逐项测量
  （before `200` / after `unreachable`）。这条性质由**地址位置**保证，不由认证保证：relay 仍然不认证调用方。
  因此它是必须与能力集合一起被检查的前提，而不是一次性的接线选择。
- 同时禁止：不得为沙盒面向的 Git Server 监听做 host 端口发布，也不得经 host 转发——那会把来源变成网关地址，
  使判定必然失败（拒绝而非放行）；同理，`sandbox-server` 的管理与中继监听不得绑定到工作负载所在的网段。
- 已实测的判据（2026-10-13，见 D3 的已知残余）：前提检查对**实际容器**的默认运行报 `exit=0`，而
  `-require-peers-constrained` 报 `exit=1`；后者不是本文的失败条件，而是本文刻意接受的前提边界——本文的可证明性
  以"控制面在信任边界内"为前提，不要求控制面自身受这条前提约束。
- 已知代价：本文因此把一个网络安全属性变成了一条产品级约束，未来任何"给沙盒加能力"的改动都多了一道评审门槛。
  这是此方案为"不需要任何客户端秘密"所付的代价。

### D5：逐请求授权，且判定只用到三样事实

Git Server 在不持事务的只读查询中，依据 Cloud 数据库的当前状态判定：

1. 解析出的容器是当前存活的沙盒，其沙盒实例是 Cloud 记录里该 Workspace 的当前实例（旧代次、已终止、已强停的实例
   拒绝）；
2. 请求定位到的 Project 属于该 Workspace——路径中的标识只用于**定位**，不用于授权：不匹配即拒绝，匹配也不因此
   获得任何额外权限。该 Project 还必须处于仍可服务自身内容的状态：`provisioning` 与 `active` 可读，`deleting`
   与 `deleted` 拒绝；
3. 要读的 ref 属于该 Workspace 本次克隆已核验的镜像与提交。

第 2 条把 `provisioning` 计为可读，是**必须**而不是放宽：带凭据 Project 的**第一个** Workspace 正是**在 Project
仍为 `provisioning` 时**克隆自己的镜像的——把 Project 置为 `active` 的 `openWorkspace` 写在同一次 operation 的末尾，
而这次 operation 的 clone 步骤就是该读取本身。只承认 `active` 会让读取被自己将要促成的那个状态拒绝，通道永远送不出
第一个 Workspace。隔离栈上实测过这一形态：Workspace、其沙盒实例与代次、钉住的提交、已发布的镜像全部匹配，Git Server
仍以 `container_not_live` 拒绝，唯一不同的是 `projects.lifecycle='provisioning'`。这不是身份边界的改动——调用方仍然
必须是该 Workspace 的存活沙盒，镜像键仍然由该 Workspace 的 Project 派生，因此能读到内容的 Project 仍然只有它自己。

| 情况 | 响应 |
| --- | --- |
| 判定通过 | 交给 `git http-backend`（只读） |
| 任一条件不满足 | `403`，不含细节；"仓库不存在"与"无权访问"返回相同响应 |
| 来源无法解析为运行中的沙盒容器 | `403`，不区分与上一条 |
| Cloud 状态不可读或查询超时 | `503`，不放行 |

为什么不再使用 `Ora-Execution-Id` 一类声明执行的请求头（对根决策 D3 的修改）：读方向的目标完全由解析出的
Workspace 决定，请求既不需要、也不得声明自己服务哪个执行；把授权建立在调用方自述的字段上，等于把授权交给调用方。

### D6：镜像的键、上限、保留、销毁、失败清理、孤儿回收与并发

- **键**：`(project, 上游仓库的规范化身份, 提交)`。镜像内容不可变，发布之后不再改写；同一 Project 的第二个
  Workspace 复用同一键的镜像，不重复搬运。
- **上限**：单个镜像在镜像阶段即有字节上限，超限是**终态失败**——不重试、不部分发布。存储卷总量与每租户配额
  同样在发布前判定。
- **保留与销毁**：镜像在"被存活 Workspace 或待结算的克隆执行引用"期间保留；最后一个引用消失后再经过宽限期才
  销毁。Workspace 终结时不立即删除镜像，避免同一 Project 的下一代会话重复搬运；但读取权限随容器消失立即关闭
  （不变量 13）。
- **失败清理**：镜像先写入暂存路径，核验（对象完整性、目标提交存在、ref 唯一且只指向该提交）通过后**原子发布**
  （rename 或等价的唯一性操作）。未核验的镜像永不进入可读集合；半成品不留成可读仓库。
- **孤儿回收**：回收器以 Cloud 的权威引用对账存储上的镜像，只删除超过宽限期且无引用的对象；对账绝不依据 Git
  Server 自己的记忆或本地记录。
- **并发**：同一键的发布以排他锁串行化；并发产生的第二个请求等待，或直接复用已核验的同一镜像，不产生两份可变副本。

**完成状态（2026-10-13，同日修订）——本节是承诺，不是现状。** 键、失败清理、原子发布、宽限销毁与撤销的
**机制**已在 `cloud/internal/mirror` 与 `cloud/internal/mirrorpublish` 内实现并有测试覆盖。**创建侧现在有生产
调用方**：`cloud/internal/core` 的镜像准备 worker（`PrepareMirrorsOnce` 配 `MirrorDispatcher`）领取 Operation、
签发 `mirror_prepare` 授权并派发给 C，由 C 抓取后经"暂存—核验—原子发布"写入镜像卷；开关
`CLOUD_CLONE_DISPATCH_MIRROR_ENABLED` 默认关闭。

**读者与回收者现在也有调用方，但只在可选的部署覆盖层里。** 覆盖层 `compose.git-server.yaml` 给
`git_server.sandbox_base_url` 赋值（`Store` 因此持有镜像端点）、打开 `CLOUD_CLONE_DISPATCH_MIRROR_ENABLED`，
并把**回收循环**交给 `cmd/git-server`：它是对账唯一可用的进程，因为只有它同时持有 Cloud 的数据库与镜像卷
（Cloud 有库无卷，执行身份 C 有卷无库）。生产 `compose.yaml` 不含该覆盖层，**未应用它的部署与本文写作时逐位
相同**：`Store` 不持有镜像端点、带凭据的 Workspace 在 `clone` 转换上以 `mirror_not_ready` 被拒，没有任何
Workspace 从镜像 clone，也没有任何镜像被回收。

**回收侧**因此有了唯一的调用方，且**默认关闭**（`-mirror-reclaim`；零值或负值的间隔在启动时被拒）。开启后它
按 `-mirror-reclaim-interval` 反复执行 `mirrorpublish.Reconciler.Maintain`：只读 Cloud 的权威引用、只撤销、只
清扫、只从存储上的记录回答"就绪"，**不发布**——发布要一枚 Cloud 为单次抓取签发的授权，而一次对账不是
Operation，签不出这样的授权。`-mirror-grace` 是同一开关的另一半：在循环存在之前它是惰性 flag，因为无人调用
清扫。据此，下列各项**从"无调用方"变为"有调用方"，但仍不记为完成**，因为各自的证据不在本轮：**每租户配额**
（没有任何代码按 tenant 记账；卷总量上限只由写入方在发布时强制，`Sweep` 从不按容量回收）、**撤销与销毁在
Cloud 侧的状态一致性**、**失败交付尝试留下的 ref 与对象在真实执行身份下的清理**。反之，**孤儿回收
（`Sweep`）、Project 删除后引用消失、标记跨进程重启的持久性、宽限期内外的保留与销毁**已在隔离栈上取得真实
端到端证据（见落地顺序第 3 步）。镜像记录的落点已按本次裁决定为受控持久卷上的文件系统记录，**不引入数据库
migration**（原先"很可能带来一次 migration"的推测作废）。

**裁决补充（2026-10-13）——镜像卷的属主是部署固定的一次性事实，每个写入方都必须保持它。** 镜像卷是**一个
目录、两个身份**：C 以执行身份（部署取值，默认 uid 10002）写入，Git Server 以 root 读取、回收并对账。部署在
两者启动之前把卷骨架与该属主一次性固定（`compose/git-server-setup.sh` 的 `chown -R "$MIRROR_OWNER_UID"
/mirror`），正是为了让**先到的进程决定不了后到的进程能否打开这个卷**。这条承诺并不止于"准备一次"：**写入方
新建的每一个文件都必须属于该属主，而不是属于写它的进程**。回收者是这条要求下的第一个新写入方——在它之前，
卷上唯一的写入者是 C 自己。

不满足时的失败模式已实测（隔离栈，2026-10-13），且比"一条记录读不到"严重得多：root 的回收循环把一条记录写成
了 `root:root 0600`，而 `Preparation.Publish` 经 `publishedBytes` → `Records()` **读取 records 目录下的每一条
记录**以核对卷总量上限，**一条不可读即整次调用失败**。于是一个键的记录不可读，会让此后**每一个键**的发布都
失败，且报错指向一个与本次发布无关的文件：`mirror store is unavailable: open /mirror/records/<digest>.json:
permission denied`。因此实现上的承诺是双向的：属于该属主的文件正常写入（对属主自身是空操作，对任何第三方身份
失败关闭），**不能取得该属主时写入失败**，而不是留下一个"自己写得出、另一个打不开"的文件。锁文件与记录文件
一视同仁——锁文件由 `flock` 创建，放宽目录权限救不了它。

**裁决补充（2026-10-13）——"不引入 migration"限的是镜像记录，不是操作状态。** 本轮批准的镜像准备**前置
Operation** 需要独立的、可审计、可重试的状态，绑定 tenant、project、workspace、operation、runtime
generation、credential ID/version、真实上游 repository 与精确 ref，因此**允许增加最小的 schema
migration**，且**只在隔离 PostgreSQL 上验证**；正式库不迁移。两者的分界是：**镜像内容与镜像记录**继续
落在受控持久卷上（本段上文），**镜像准备这一操作的权威状态**在 Cloud 的数据库里，因为它要参与 Cloud 的
操作状态机、要能被重试与结算，而一卷文件系统记录无法承担这些。

### D7：Cloud 只在镜像就绪后写克隆输入，并钉住精确提交

- Workspace 的克隆输入（记录中的 `repositoryUrl` 与 `requestedRef`）只在镜像已发布并核验之后写入；`requestedRef`
  钉到该镜像中指向目标提交的那个 ref。镜像不含其他 ref，因此"不存在的分支"与"已经移动的分支"都不可能被克隆，
  克隆结果对同一 Workspace 是确定的。
- 派发侧的既有不变量沿用、不改：`internal/core/workspace_clone.go` 的 `dispatch_conflict` 要求派发输入与实际记录的
  `repositoryUrl`／`branch` 逐字相等，`internal/controlgrpc/execution_codec.go` 的 `CloneSpec` 就从这条记录生成。
  于是 Node 只可能克隆 Cloud 记录过的镜像地址，**上游地址从不出现在这条记录里**，Node 无法被指向上游。
- **必须同步修改的一项**：`internal/core/credentialed_clone.go` 今天用 `w.S("repositoryUrl")` 规范化出"要绑定哪个
  仓库"（`normalizeRepositoryScope` 与 `cloneauth.CanonicalRepository`，同时是 scope 比对与授权绑定的来源）。
  一旦该字段变成镜像地址，凭据授权就会绑定到**镜像**而不是用户的上游仓库，授权会指向错误的仓库身份。因此 C 侧
  的授权与 scope 校验必须改为以 Project 的上游仓库身份为准。这是安全相关的必改项，不是可选项。
- **裁决（2026-10-13）**：本节承诺获批，并且"写克隆输入"这一步被明确限定为**在镜像 ready 且全部绑定重新核验
  通过之后**才执行。旧代次、重绑、撤销、轮换与迟到结果都不得推进到写输入；镜像 URL **不得**被用来替代真实上游
  repository 身份去授权。同时明确：**mirror ready 本身不等于 Workspace clone 成功**，两者是独立的状态，前者
  不蕴含后者，也不把后者记为已完成。
- **裁决补充（2026-10-13）——镜像准备不能作为独立的、Controller 可见的 step。** 本轮隔离实现原本把镜像准备
  实现成 `operations.step` 的一个新取值（`mirror`，由 migration 0034 插在 `node` 与 `clone` 之间），并已在
  隔离 PostgreSQL 上验证。**该形态被证据否决**：step 经 `internal/controlgrpc/operations.go` 的 `enum()` 转成
  protobuf 枚举发给 desktop Controller，而 `enum()` 是无 comma-ok 的 map 查表，未识别的名字**静默取零值**，
  于是 Cloud 实际在线上发出 `OPERATION_STEP_UNSPECIFIED`；Controller 把它当失败
  （`apps/ora-controller/src/cloud/operations.rs` 的 `OperationStep::Done | OperationStep::Unspecified =>
  Err(Error::Conflict)`），`drain()` 随即放弃该轮并重新认领。结果是**该 step 永远不会产生任何 effect，操作也
  永远不离开它**；`desktop/` 在本轮严格只读，无法为它补一条分支。因此：**镜像准备的真实状态记录在
  `mirror_preparations`（主键为 `operation_id`，自带 `state` 相位），运行在既有的 `node` → `clone` 转换之内**，
  由 Cloud 在该转换上把门——未 ready 或绑定未重新核验通过就不转换。migration 0035 把 `mirror` 从 step 词汇表
  撤回，使它不再是一个系统可以进入、却无人能驱动的状态。这**不改变**本节对顺序的承诺（镜像先于 Clone），
  只改变该承诺在何处被表达。本轮同时修掉了让这件事**无声**的那个映射：`controlgrpc` 现在对"合约未定义、
  且不属于退休集合"的 step 明确拒绝（`codes.Internal`，并在错误里带出该名字），不再静默降级为
  `OPERATION_STEP_UNSPECIFIED`。失败因此发生在 Cloud 自己的投影里并写明原因，而不是留给 Controller 去整轮
  失败。四个退休名（`storage`／`worktree`／`ready`／`storage_delete`）仍投影为零值，因为已应用的行还写着它们、
  而历史操作永远不会被 claim。

### D8：内部只读通道与外部 Push 是两条独立路径

- 内部：只读、无凭据，方向是 C → 沙盒。
- 外部：把成果推回用户上游仍是 proposed 的 push 执行授权范围，凭据由 C 持有，不经本文通道。
- 内部通道的任何"顺带"扩展（例如让沙盒直接 push 到内部仓库，或让内部仓库成为上游的写代理）都被拒绝：那会把
  只读接口变成写入面（D2），并把上游凭据的作用面扩大到容器。

### D9：部署接线——服务端信任只经既有的 `clone.gitconfig` CA 入口

- Node 侧现状（只读核对）：Cloud 策略只允许部署 gitconfig 出现 `[http] sslCAInfo=`（`desktop/apps/ora-node/src/repository/config.rs`
  的 `validate_cloud_policy`），镜像里该文件的默认内容只有注释，并明说"要加 HTTPS CA 引用就挂载替换件"
  （`desktop/docker/clone.gitconfig`）；`GIT_CONFIG_GLOBAL` 指向该文件。因此"信任内部 Git Server 的 CA"是**既有的、
  受支持的**接线方式，**不需要修改 desktop**。
- CA 必须是工作负载 uid 可读的普通文件（git 以工作负载 uid 运行、基础环境为空），不能用 `/run/ora`：该目录是
  root 拥有的 `0700` tmpfs，其内文件 `0600`，工作负载 uid 既不能遍历也不能读取。
- Node 的 clone 配置里不得出现任何凭据键、证书或 header；`sslCAInfo` 是唯一新增项。

### D10：来源解析有自己的监听、自己的 CA、一个固定的 git-server 调用方身份

- Git Server 不直接持有 Docker Engine 访问权。来源解析由 `sandbox-server` 提供：它本来就持有 Engine 客户端、本来
  就按标签解析沙盒（`internal/route/docker.go`），新增的只是一个"按地址解析"的只读查询。接口形态（已实现，
  `internal/origin`）：`GET /origin/v1/resolve?address=<IPv4>`，对运行中的沙盒容器返回身份，对已停止的容器、
  网关地址、无人持有的地址与非沙盒容器返回同一个 `403`，Engine 不可读返回 `503`。默认**关闭**：只有显式加
  `-origin-resolve`（并要求 `-origin-addr` 与 `-origin-tls-dir` 同时给出）才启动该监听；未启用时部署里根本
  不存在这个面，管理监听的行为与今天逐位一致。
- **独立监听。** 该查询有自己的绑定地址（`-origin-addr`），不挂在管理监听 `/effects/{id}` 上。理由：两个面认证的
  是**不同的调用方**（管理面认 Controller，来源面认 Git Server）。共用一个监听就只能共用一个固定叶子身份，等于把
  两个角色合并——一个能派发执行效果的身份，同时能就任意地址提问，而提问权就是身份判定权（见 D3 的已知残余
  "被信任的调用方自述了对端地址"）。分开之后，"能派发效果"不再蕴含"能解析任意地址"。
- **独立 CA。** 该监听的服务器证书、客户端 CA 与固定调用方叶子来自它自己的目录（`-origin-tls-dir` 下的
  `server.pem`、`server.key`、`ca.pem`、`git-server.pem`），与管理 mTLS 链的 CA **不是同一张**。理由：管理 CA 的
  信任面今天覆盖 Controller 与整个 effects 接口；让来源解析共享它，等于把"能就任意地址提问"发给任何一张管理链
  证书的持有者。独立 CA 让来源面的信任面恰好只有一张证书。
- **固定 git-server 调用方身份。** 该监听只接受 `git-server.pem` 这一张叶子：按**证书目标**固定比对，不按
  `CN`／`SAN` 匹配任意可信证书。调用方身份只从 `r.TLS` 读取，处理器自身再断言一次"连接确实带着已验证的客户端
  身份"，以免被误挂到没有 mTLS 的监听上。这与已经作废的 `controller.pem` 方案是两件事：Git Server 拿到的是它
  自己的身份，不是 Controller 的身份。
- 与 D4 的衔接：这两个监听必须绑定在工作负载**不在其上**的网段（D3 的已修正项、D4 的前提）。来源面因此有三层
  约束——网络位置、独立 CA、固定调用方叶子——缺任一层都不满足本文。
- 为什么：让一个新服务拿到 Engine 访问权，等于把"能创建容器、能给容器打标签"的能力再发一份；而标签正是 D3 的
  授权依据。把解析放在已经拥有该能力、且已被当作信任根的组件里，新增面最小。承认的代价：Git Server 因此依赖
  `sandbox-server` 的可用性，它不可用时读请求失败（拒绝，不放行）。

## 不变量

1. 沙盒容器不持有 `NET_RAW`／`NET_ADMIN`，保持 `no-new-privileges`，且不发布端口；这是来源身份可证明的前提（D4）。
2. 读取授权不来自 URL 的持有权，也不来自任何请求可自述的字段（路径、`Host`、查询串、请求头）。
3. 每次请求都重新解析来源；不缓存"地址 → Workspace"的映射，也不把它落库。
4. 沙盒面向的监听只提供 `git-upload-pack`；内部通道没有写权限。
5. 跨 Workspace、跨租户、旧代次、已停止容器、不可解析来源与匿名请求一律拒绝，且不区分"不存在"与"无权"。
6. 镜像在被读取期间不可变；被克隆的 ref 只指向已核验的目标提交。
7. 本通道不产生任何新的客户端秘密：Node 与工作负载都不持有 Git 凭据、客户端证书私钥或长期令牌。
8. 未核验或未完成发布的镜像永不对外可读；镜像销毁不早于其最后一个引用消失并经过宽限期。
9. 镜像的可用性不改变 fail-closed：镜像未就绪、Git Server 或 `sandbox-server` 不可用、Cloud 状态不可读时，相关
   克隆被拒绝，不退化为匿名访问上游，也不回退静态凭据。
10. 客户端证书、执行声明（`Ora-Execution-Id` 一类）、Basic／Bearer 与匿名访问都不是本通道的认证方式。
11. 来源身份只到容器粒度：工作负载与 git 共享网络命名空间与 uid，因此本容器内的任何进程都能读本 Workspace 的
    镜像。这是已知缺口（与凭据路径记录的同类缺口同源），不由本文解决，也不得在文档与提交说明中表述为隔离。
12. Git Server 重启不改变任何判定；它不保存授权副本，状态变化在下一个请求生效。
13. Workspace 终结（容器消失）即关闭读取；镜像的物理销毁另外遵循 D6 的保留与宽限期。
14. `sandbox-server` 的管理、中继与来源解析监听都不绑定在工作负载可达的网段上；未经认证的中继不因为来源是沙盒
    而可达（D4）。这条由网络位置保证，不由认证保证。
15. 镜像的发布授权只来自 Cloud 的数据行。能写镜像卷的进程仍然无法发布一个 Cloud 未绑定租户、仓库与克隆执行的
    镜像；Cloud 数据行未提及的键一律拒绝（D2、D6）。

## 为什么不是这些替代方案

| 方案 | 取舍 |
| --- | --- |
| Node 出示 git-access 客户端证书（根决策 D2／D5 的原方案） | 身份最强，且有既有的按代签发先例；但私钥必须对执行 git 的身份可读，而该身份就是工作负载 uid，因此工作负载可窃取（原 D5 已自认此缺口）；还要把证书经 `-c http.sslCert/sslKey` 传进 argv，与已批准 D11"不经 argv 或环境传递"直接冲突；并需要修改 `desktop/` 的 `validate_cloud_policy` 与 `constrain`。收益为零：粒度仍然是容器。否决。 |
| 复用 WSS 服务端证书 | 少签一张证书；但已批准的模型客户端身份决策已否决双用途，且同样要求把私钥交给工作负载 uid。否决。 |
| 请求头或调用方陈述的字段（如 `Ora-Execution-Id`） | 实现最省；但沙盒内任何进程都能自造，等于把授权交给调用方，本次裁决已明确禁止。否决。 |
| 只按来源地址判定，不约束容器能力集合 | 实现最省；但容器一旦拿到 `NET_RAW` 即可伪造源地址，属于"未经证明的来源 IP"。否决——这正是 D4 把它提升为前提的原因。 |
| 为每个 Workspace 建独立网络，身份来自网络成员关系 | 不依赖能力集合，网络层不可伪造；但要给沙盒生命周期增加网络创建与销毁、每容器多网络接入，Engine 与 iptables 状态随 Workspace 数增长，且仍需要一个实时解析步骤。作为 D4 前提被破坏时的**唯一回退**保留，本文不采用。 |
| 把镜像放进 Workspace 卷，Node 从本地路径克隆 | 完全不需要网络身份；但克隆目标只支持 HTTPS（`crates/node-protocol/src/domain/repository.rs` 的 `CloneRepositoryUrl::parse`），走本地路径必须修改 `desktop/`。否决。 |
| Git Server 直接读 Docker Engine 自行解析来源 | 少一个接口；但把 Engine 访问权多发一份，而标签正是授权依据，新增面更大（D10）。否决。 |
| 让 Agent 或沙盒直接访问外部私有仓库 | 少一次搬运；但等于把上游凭据交给工作负载，违反已批准隔离决策。否决。 |
| 在 Git Server 前面加一个 per-Workspace 代理，由代理以部署身份转发 | 代理可以携带部署凭据，容器不持秘密；但代理自己仍要回答"进来的连接是本 Workspace 的哪一个容器"，问题原样存在，只是多了一个进程与一条链路。否决。 |

## 本决策未解决的问题

- **写方向的身份与授权**：把成果推回内部仓库或用户上游仍属 `20261012-push-execution-authority.md`（proposed）；
  本文不放行任何写。该 ADR 同样需要把"声明执行 ID"改为服务端可判定的输入，否则会重复本文已否决的形态。
- **会话级读取窗口的收束**：Agent 会话期间镜像必须保持可读（否则 `git` 命令无法工作）；更细的"执行结束后立即
  关闭读取"留给后续 ADR，本文只做到"Workspace 终结即关闭"。
- **镜像 ref 与 Revision 交付 ref 的命名空间**：同一 Git Server 同时承载只读镜像与交付 ref 时，两套 ref 的命名
  空间、配额与 GC 策略如何合并。
- **镜像的深度**：是否保留完整历史、是否支持子模块与 LFS。本文只要求"目标提交及其所需对象可用"。
- ~~**镜像记录存放在哪里**~~ **已裁决（2026-10-13）**：权威记录暂留在**受控持久卷上的文件系统**里，不引入
  数据库 migration；镜像内容与记录同卷。这不排除将来把它移进数据库，但迁移必须另立决策，不得在本轮顺手做。
- ~~**镜像准备的操作状态放在哪里、C 用什么接口准备镜像、`mirror_prepare` 要不要新能力**~~ **已裁决
  （2026-10-13）**，三项分别如下，均**只在隔离完整栈内实施与验证**，不改生产行为：
  1. **新增 Cloud 权威的 mirror_prepare 前置 Operation**（状态见 D6 的裁决补充）：必须有独立、可审计、可重试的
     操作状态，绑定 tenant、project、workspace、operation、runtime generation、credential ID/version、真实
     上游 repository 与精确 ref；镜像准备在 Node Clone **之前**完成；Cloud 必须在镜像 ready 后**重新核验全部
     绑定**，并把 C 返回的完整 commit 与权威操作记录关联；旧代次、重绑、撤销、轮换与迟到结果一律不得推进状态。
     **mirror ready 不等于 Workspace clone 成功**；公开匿名仓库保留现有路径（不进入这条通道）。
  2. **C 增加独立镜像准备接口**，使用独立于 `POST /v1/clone-executions` 的路由、请求与结果类型，**不在原
     Clone Job 上新增 kind 开关**。镜像消费方**只能**接受 `operation=mirror_prepare`，Clone 消费方**只能**
     接受 `operation=clone`，两向都要测用途混淆。C 保持外部凭据独占与 C-only 私有临时材料规范，按"受权单仓库、
     单 ref"抓取，核验 commit，原子发布裸仓库，不向 Node／Agent／Cloud 业务进程暴露秘密。
  3. **`mirror_prepare` 继续使用 `capabilities=["clone"]`，不新增 capability。** 这只表示受限读取能力不扩大；
     operation 是**独立且强制校验**的用途维度。**不得**扩展到任意 ref/tag/submodule、其他仓库、Push、删除或
     强制更新。Store 可保持现有通用兑换逻辑，但必须记录并验证各可信消费方在使用材料前强制校验 operation 的
     安全不变量。
- **镜像准备是否需要一个 Controller 可见的独立相位**（2026-10-13 受阻于只读边界）：本轮把准备状态放在
  `mirror_preparations` 里、跑在既有 `node` → `clone` 转换内，因此**没有**新增 step 或 effect 类型，也就不需要
  改 `desktop/`。若将来要让镜像准备成为一个能被 Controller 观察与驱动的独立相位，就必须同时改 Cloud 的
  `controlpb` 与 Controller 的 step 分发，即一次跨 `desktop/` 的变更；它超出本轮边界，且必须先回答"新增的
  Controller 可见 step 如何被旧版本安全忽略"这一兼容性问题——本轮的证据正是：今天的答案不是"被忽略"，而是
  "整轮失败并停在那里"。
- **部署服务与工作负载共用网桥的残余**：见 D3 的已知残余。其中"未认证中继对工作负载可达"一条已在 cluster 栈的
  隔离 overlay 上修正并测量；**仍未决的是该 overlay 是否进入主栈**，以及是否愿意为 `ora-sandboxes` 上其余部署
  服务收紧能力集合。`sandbox-server` 的 Docker socket 无法收紧——它与"`sandbox-server` 必须持有 Engine 客户端"
  直接冲突，因此本文接受"控制面在信任边界内"这一前提，不自行放宽也不自行收紧。
- **调用方自述对端地址的残余**：见 D3 的已知残余。彻底消除需要把沙盒面向的读取终结在 `sandbox-server` 进程内，
  这是一个通道形态的改变，超出本文范围。
- 生产部署的多副本、存储后端与 GC 阈值；会话 JSONL 仍走对象存储（根决策不变量 14 不变）。
- ~~**`mirror_prepare` 授权既签不出、也兑换不了**~~ **已裁决（2026-10-13，签发侧已落地，兑换侧待落地）**：
  上一轮登记的两项阻塞（`cloneauth.Binding` 对任何 operation 都要求非空 `execution id`；Store 兑换要求出示方
  自述一组 Node／runtime 值，而 C 不运行在任何 Node 上）都改动签名载荷形态与 Store 的身份判定，因此当时未自行
  决定。裁决结果是**按 operation 区分签名绑定**，而不是任何兼容捷径：
  1. **operation 是强制校验的用途维度，不是标签。** `OperationClone` 与 `OperationMirrorPrepare` 各自**要求**
     本用途的身份字段并**禁止**对方的：clone 要求 `executionId` 与整组 Node 运行时值、禁止 `ref`／`operationId`／
     `preparationGeneration`／`executorIdentity`；镜像准备要求 `operationId`／`preparationGeneration`(≥1)／
     `executorIdentity`／`ref`(`refs/heads/...`)、禁止 `executionId` 与整组 Node 运行时值。两组**不得同时出现**，
     未知 operation 被拒而不是回落到 `clone`。**明确不允许**用假 Node ID 兼容，也**不允许**把 Operation ID 填进
     execution 字段——混用载荷读起来两种解释都成立，那样 operation 就退化成标签。
  2. **旧 Clone 协议逐位不变。** 上述规则不改变任何一份 clone 绑定的字节：clone 的字段集、签名输入与校验顺序都
     与上一轮相同，全部既有 clone 用例（含 `internal/cloneexec` 的验收路径）不改断言仍绿。
  3. **镜像用途的"谁可以花掉它"改由证书判定，而不是自述。** clone 绑定一个 Node 运行时，出示方能如实自述，
     Store 比对的是请求自述值；C 没有运行时可自述，因此镜像绑定的是 `executorIdentity`，由 Store 从**验过的
     mTLS 客户端证书**解析（`internal/credentialstore` 的 `Identities.PresentedIdentity`），即"部署签发了证书的
     那一方"，而不是请求体里写着的名字。上面那条"C 回述一个它并不运行于其上的 runtime 是否构成绑定"的疑问因此
     不再成立：C 不再回述任何 runtime 身份。
  **签发侧证据（2026-10-13）**：`internal/cloneauth` 的 `validateClone`／`validateMirrorPrepare` 双向强制；
  `purpose_test.go` 的 14 个"改一个字段"用例证明双向互斥（含把 Operation ID 填进 execution 字段），另有三个
  手工签名、只保留真实签名的用例证明**有效签名也不能让混用或未知 operation 的载荷通过 `Verify`**；
  `internal/cloneexec` 的 `TestExecuteRefusesAMirrorAuthorizationOnTheCloneRoute` 证明一份合法签发、且仓库／
  凭据／版本／代次都与 job 一致的镜像授权在 clone 路由上以 `binding_mismatch` 被拒，且 Store 一次都没有被询问。
  **仍待落地**："Store 不能实时查询 Cloud 时，授权有效期与撤销窗口如何如实声明并在发布与 ready 接收处重新校验"。
  **Store 侧的镜像专用兑换路径已落地（2026-10-13）**：`POST /v1/credentials/{id}/resolve-mirror` 只接受
  `operation=mirror_prepare` 的授权，出示方身份取自**已验证客户端证书**而非请求自述，兑换仍是一次认领、并校验
  凭据的当前版本与撤销状态。今日仍**没有**凭据会因此被交付：`CLOUD_CLONE_DISPATCH_MIRROR_ENABLED` 默认关闭，
  且 `broker-resolver` 证书在部署栈上按设计不签发。

## 落地顺序

1. **部署前提固化**：把"沙盒容器不持有 `NET_RAW`／`NET_ADMIN`、保持 `no-new-privileges`、不发布端口"变成可断言的
   检查，且对**实际创建的容器**断言而非源码常量。完成条件：检查在隔离栈上对真实容器通过，并在人为加回 `NET_RAW`
   时失败。行为不变。
   **已落地（2026-10-12）**：`cmd/premise-check` 在隔离网络上对真实容器通过；同一个检查在加回 `NET_RAW` 的容器上
   失败，且该容器内 `socket(AF_INET,SOCK_RAW,IPPROTO_RAW)` 与 `AF_PACKET` 原始套接字**确实打开成功**——违规不是
   纸面上的，而是可发送伪造源地址的。对主栈 `ora-sandboxes` 只读运行：8 个沙盒容器全部通过，4 个部署服务被登记
   为 D3 的已知残余。
2. **来源解析接口**：`sandbox-server` 增加"按地址解析沙盒"的只读查询，复用既有管理监听与既有调用方身份。完成
   条件：对运行中容器、已停止容器、非沙盒的部署容器分别返回正确结果或"不可解析"。此时无调用方，行为不变。
   **已落地（2026-10-12）**：`internal/origin` 提供解析器与处理器，默认关闭，由 `-origin-resolve` 显式开启。
   在隔离网络上以真实容器、真实 mTLS 验证：运行中的沙盒容器返回身份；已停止容器、网关地址、无人持有的地址、
   非沙盒容器一律 `403` 且响应体逐字相同；Engine 不可读为 `503`；无客户端证书、以及 CA 可信但未被固定的证书
   均在握手中被拒。生产 compose 未加该 flag，行为不变。
3. **Git Server 骨架与镜像发布**：只读 smart HTTP、镜像的暂存—核验—原子发布、容量上限与失败清理；沙盒面向监听
   的**授权尚未启用**（此时任何沙盒请求都被拒绝）。完成条件：用真实 `git` 客户端与真实 PostgreSQL 证明只读、
   不可写、未发布镜像不可读。行为不变。
   **代码已落地（2026-10-12），容器级测量见第 4 步**：`cloud/cmd/git-server` 默认不部署、不发布端口；沙盒面向的
   监听只注册 `GET /mirror/v1/{name}/info/refs` 与 `POST /mirror/v1/{name}/git-upload-pack` 两条读取路径，
   `info/refs?service=git-receive-pack` 走到同一个决定点后按"写方向不服务"被拒绝，而手写的
   `POST .../git-receive-pack` 因为**根本没有注册这条路径**由标准库以 `404` 作答——两者都不是"写被放行"；该偏差
   已按实测写进 D2，并把"不存在与无权不可区分"限定在读方向。`internal/mirror` 实现暂存—核验—原子发布的状态机
   （preparing/ready/failed/revoked/destroyed），容量上限、键级锁、失败回收与孤儿扫描在同一包内。
   **本步骤新增的写入方（2026-10-13）**：`cloud/internal/mirrorpublish` 实现对账器与写入方——它先读 Cloud 的
   数据行得到"应当存在哪些镜像"，再据此撤销、发布、清扫，因此"不发布 Cloud 未引用的镜像"对进入它的每条路径都
   成立（不变量 15）。**发布侧与被撤销／清扫侧分开看**：发布需要 `Fetcher` 与 `MaterialSource` 两个消费侧边界
   接到执行身份 C 所执行的带凭据抓取上，而那次抓取要兑换一枚 Cloud 为单个仓库签发的单次授权——一次对账不是
   Operation，签不出这样的授权，所以**对账器从不发布**，只撤销与清扫，这是能力上的限制而非取舍。**撤销与清扫
   现在有生产调用方**：`compose.git-server.yaml` 把 `mirrorpublish.Reconciler.Maintain` 交给 `cmd/git-server`
   的循环（`-mirror-reclaim`，默认关闭），因为只有该进程同时持有 Cloud 的数据库与镜像卷。生产 `compose.yaml`
   未使用该覆盖层，行为不变；循环存在之前的部署与本步骤写作时逐位相同（详见 D6 的完成状态）。
4. **部署接线**：Git Server 加入 cluster 栈（默认网络 + 沙盒网络，不发布端口）；为沙盒容器挂载只含 `[http] sslCAInfo=`
   的 `clone.gitconfig` 与工作负载 uid 可读的 CA 文件。完成条件：容器内 `git ls-remote` 能对 Git Server 完成 TLS
   握手并被**拒绝**（授权未启用），证明信任链与 Node 的 Cloud 策略校验都已就位。至此仍不改变任何现有行为。
   **隔离验收已落地（2026-10-13），部署接线未落地**：`scripts/git-read-acceptance.sh` 在三条 `--internal`
   Docker 网络上用真实容器、真实 PostgreSQL、真实 `git` 客户端与真实 Git Server 进程跑 **29 条断言，全部通过**
   （`29 passed, 0 failed`，退出码 0）。其中与第 4 步完成条件直接对应的是：客户端以工作负载 uid 1000 运行、
   用 Node 的同一套环境变量与 `--config` 参数前缀执行 clone；把 CA 配置换成空值时同一次 clone 以
   `certificate verification failed` 失败，证明生效的正是挂载进去的 CA；`http` 与 `ssh` 两种传输分别被 Node
   自身的策略以 `transport 'http' not allowed` / `transport 'ssh' not allowed` 拒绝。**信任链与 clone 策略已就位
   是实测结论**；把这两步接进 `compose.yaml` 属于改变部署边界，仍待裁决（见"本决策未解决的问题"）。
   本次验收在隔离栈上以显式参数开启了读取授权并写入了镜像与克隆输入，**这些都是验收容器内的状态**：生产
   `clone_dispatch` 仍默认关闭，Git Server 未接入生产流量，镜像存储只有单副本与本地卷（多副本见未决项）。
5. **开启读取授权与镜像搬运**（需本 ADR 批准后进入）：Cloud 在镜像就绪后写克隆输入并钉住提交；`credentialed_clone.go`
   的仓库身份改为以 Project 的上游为准。完成条件：在隔离栈上跑通真实的 Clone → Workspace → Agent，且负向矩阵
   （D5 表）全绿。
   **负向矩阵已在隔离验收中取得实测证据，第 5 步本身仍未进入**：授权绑定 tenant／workspace／容器／运行代次／
   真实上游 repository／操作类型／镜像 ready 状态，逐项被真实请求触发并拒绝——同网段无沙盒身份容器 `403`；
   另一租户访问 A 的镜像 `403`；**同租户另一 Workspace** 访问 `403`；容器与 Workspace 运行代次不符（3→4）`403`，
   代次恢复后同一容器又能读取；准备失败、已撤销、已清理的镜像分别 `403`，而同一 Workspace 的活跃镜像照常服务。
   写方向：客户端 push 在 receive-pack 通告处被拒 `403`，绕过通告直接 `POST git-receive-pack` 得到 `404`（该
   监听没有注册这条路径）。拒绝的**不可区分性**也被直接测量：一个格式合法但 Cloud 没有记录的镜像摘要，与
   "另一租户的镜像"得到**逐字节相同**的响应，因此拒绝原因只进日志、不进响应体。这里有一处**如实修正**：该
   未记录摘要走的是 `digest_mismatch` 而不是 `mirror_unknown`——状态查询取的是**调用方所在 Workspace 自己的**
   镜像键，摘要比对发生在容器存活与 Workspace 绑定之后、镜像状态之前，所以"库中没有这个摘要"与"这个摘要不是
   本 Workspace 的"在实现里是同一个判定；`mirror_unknown` 只在 Workspace 自身没有可解析的镜像键时出现。
   这不改变外部行为，但本文原先按错误码理解这处测量，现按实测更正。
   不可用：Cloud 权威状态不可读 `503`、来源解析器不可用 `503`、Git Server 不可用则 clone 失败，三者恢复后读数
   恢复。Git Server 日志中的拒绝原因计数为 `container_not_live 1 / digest_mismatch 5 / mirror_not_ready 2 /
   mirror_revoked 1 / operation_not_served 2 / origin_resolver_unavailable 1 / origin_unresolvable 1 /
   state_unavailable 1`（无 `mirror_unknown`，原因见上）。**缺的是执行侧**：真实的 Clone → Workspace → Agent
   需要 `desktop/` 侧执行器与部署接线，本轮未做，也未以模拟测试冒充。
   **本步骤的另一半——D7 的"Cloud 只在镜像就绪后写克隆输入"——尚未实现**：没有代码在镜像就绪之后才写
   `repositoryUrl`／`requestedRef`，也没有把该写入钉到核验过的提交上。就绪确认在**读取侧**已有调用方（Git Server
   服务前先问同一个存储，因此"未就绪即拒绝"是实测过的），缺的是写入侧的门控；在它落地之前，本步骤不得记为完成。
   **裁决补充（2026-10-13）——写入侧的门控已在 Cloud 侧落地，形态与本节原先设想的"改写记录"不同。**
   `cloud/internal/core` 现在只有**一处派生**（`mirror_preparation.go` 的 `cloneInputFor`）回答"这个 Workspace 能从
   哪里 clone"，三个使用点共用它：`node` → `clone` 转换上的门禁（`requireMirrorPrepared`）、Controller 快照的克隆
   输入（`projectMirrorInput`）、以及派发时对输入逐字相等的比对（`workspaceCloneDispatch`）。三点都拒绝读"上次写
   下的值"而就地重算，因为准备记录的 `ready` 只在它被写下的那一刻为真：租户／Project／Workspace／上游仓库／
   pinned ref／运行代次／凭据引用逐项与**实时行**重比，凭据**版本**只在门禁处比对（只有那里能从活动引用解析出权威
   版本）。**基线在转换时钉住**：`workspaces.base_commit_id` 由门禁在放行同一次写入中钉成镜像的提交，因为 Git
   Server 的读取授权要用这一列现算镜像键——写成"clone 成功之后再写"会让这次 clone 自己无处可读；`advanceClone`
   随后要求 Node 报回的提交与它逐字相等，报另一个提交是被拒而不是"重绑 Workspace"。

   **写入侧对 D7"必须同步修改的一项"的处理是让它在构造上不可能发生。** 本节原假设 Cloud 会把 Project 的
   `repository_url` 改写成镜像地址，因此要求 `credentialed_clone.go` 同步改为以上游为准。实际采用的形态是
   **只投影快照、不改任何行**：`projectMirrorInput` 改写的只有快照里的 `repository_url` 与 `requested_ref` 两个
   字段，数据库里 Project 的 `repository_url`、凭据绑定、镜像键与读取授权全都继续以上游为准，因此"凭据授权绑定到
   镜像"没有可发生的路径。这也让 D7 的"上游地址从不出现在这条记录里"成为可断言的：`clone_executions.input` 记的
   是快照给出的地址，而门禁与派发用的是同一张派生表，上游地址的派发以 `mirror_not_ready` 被拒且不留下执行行。

   **两个通道的关系同时被固定为"替代"而不是"先后"**：Cloud 是否命名内部 Git Server 是**一个部署值**
   （`git_server.sandbox_base_url` → `Store.SetMirrorEndpoint`，只在启动时装配，任何请求与调用方都打不开也关不上
   这条腿）。命名了它，Cloud 就不介入带凭据的 clone——内容由 Node 用既有匿名 HTTPS clone 从镜像取得；没命名它，
   才是 Cloud 自己执行带凭据 clone 的那种部署。两种能力同时声明会让其中一条永远派发不出去，`internal/config` 因此
   在启动时拒绝（命名了 Git Server 却开着 `clone_dispatch.enabled`；或命名了 Git Server 而
   `clone_dispatch.mirror_enabled` 关闭，即没有发布者）。**仍未取得的证据**：本步骤的完成条件是隔离栈上真实的
   Clone → Workspace → Agent，本轮**没有**做到——隔离栈尚未运行 Git Server、也没有把镜像卷接给执行器与读监听，
   因此上面的门控与投影只有进程内证据（真实 PostgreSQL），没有容器级端到端证据。本步骤继续不得记为完成。
6. **端到端验收与证据登记**：把 Clone → Workspace → Agent 的验收从 `Missing` 更新为实际证据，未运行项如实标注。
   **完成条件是 Clone → Workspace → Agent，本轮未达成**：已取得的证据是上述隔离读取链路（来源解析 → Cloud
   逐请求授权 → 只读 Git Clone）的容器级验收，执行侧的 Clone → Workspace → Agent 仍为 `Missing`。因为本 ADR
   仍是 `proposed`，按仓库规则其核心测试用例尚未建立，故本轮的证据登记在本文内如实记录，待状态变为 `approved`
   后再按核心用例重新登记。

第 1–4 步在授权未启用时不改变任何现有行为，因此安全边界可以先于业务调用方单独评审；任何一步失败都不影响现有的
对象存储路径与 fail-closed 语义。`desktop/` 在全部六个阶段中都不修改。

## 核心测试用例

按仓库规则，本 ADR 为 `proposed`，核心测试用例在其状态变为 `approved` 后补充到 `specs/test-cases/cloud/git-server/`。
至少需要覆盖：来源无法解析、来源是已停止容器、来源是部署服务时被拒；跨 Workspace 与跨租户读取被拒；沙盒面向
监听不存在写路由；未核验或半成品镜像不可读；同一键并发发布只产生一个不可变镜像；Cloud 状态不可读时 `503` 且
不放行；容器能力集合被人为放宽时前提检查失败；工作负载能读本 Workspace 镜像这一已知缺口被如实登记为缺口而非
被记为隔离。
