---
status: proposed
---

# 凭据执行器的正式归属：当前正式产品内的独立组件，旧 Desktop 保持只读，D8 的禁止面不变而形态不同

仓库凭据 clone 的执行能力**不再落到旧 Desktop 的 Node**：它成为当前正式产品（Cloud 仓库）内的一个
独立组件——`cmd/clone-executor`（进程与生命周期）、`internal/cloneexec`（受约束的 Git 调用、绑定校验、
兑换与销毁顺序）、`internal/credentialmaterial`（物化布局）、`cmd/git-credential-ora`（部署固定的
credential helper）、`internal/cloneauth`（授权签发与验签）与 `internal/credentialstore`（兑换时的
原子消费）。旧 Desktop 只作为**只读参考**保留 `ora-node`、`gitlancer` 与 process guardian 的既有安全
做法，不被修改、不被依赖、也不再承载凭据执行。

本文的方向已由项目负责人在运行 issue `ora-space/circus#29` 的授权中确认，但**尚未经过 specs 的 PR
评审**，按 `specs/AGENTS.md` 保持 `proposed`，评审通过后再据此标记 `approved`。那次授权的裁决内容
是：第 1 项要求"以当前正式 Cloud 产品及其部署架构为基础，为 Git Clone 执行能力确定一个长期维护的
正式代码归属，优先考虑在当前正式项目中建立独立执行组件，不得默认继续修改旧 Desktop"，第 2 项允许
只读参考 Desktop 的 `ora-node`、`gitlancer` 与 process guardian 而不修改它，第 3 项同意在正式执行器
中实现凭据执行身份 C、受保护的物化目录、部署固定的 helper、Broker Resolver mTLS 客户端与故障清理，
第 6 项要求以"最小范围的后续 ADR"承载实现归属的调整而不是改写历史决策。**本文只记录归属与形态**，
不新增秘密处理规则，也不改变任何已批准决策的安全边界。

上述裁决**不解除任何 fail-closed**：`repository_credential_executor_unavailable` 继续拒绝每一个带凭据引用的
Project 的 clone，本文不构成解禁依据（裁决第 8 项）。裁决第 8 项要求 D11-B 与 D14-B 的全部适用验收
义务取得直接证据之后才可解禁，而本文 D5 列出的缺口正是那些义务中尚未取得证据的部分。

**本文的去留取决于 `ora-space/circus#134`。** 审查指出本文与
[checkout 交付到 Workspace](20261012-checkout-delivery-into-the-workspace.md)描述的路径在当前默认部署
下走不通：配置层不允许 `git_server.sandbox_base_url` 与 `clone_dispatch.enabled` 同时启用，执行器也
不挂载 Workspace 卷。"凭据执行器通道是否属于本期交付"是 owner 的决策——结论若是"不属于本期"，本文
与实现一并按结论撤出；**在结论出来之前，本文不声称这条通道已经可用**。

涉及的模块：Cloud 仓库的上述六个包、`cmd/clone-executor`、Cloud 的 `cmd/server` 接线
（`internal/core` 的 `admitCredentialedClone`／`registerCredentialedClone` 与 `CloneDispatcher`）以及
部署仓库 `cluster/` 的 compose 与 setup 脚本。兼容性边界：clone 的对外协议
（`CloneExecutionSpec` 仍只有 `node_id`／`repository`／`branch`）、Cloud 的业务库 schema（不新增
migration）、旧 Desktop 仓库的内容、以及已实现的匿名 clone 与 Agent 会话链路，都按硬边界处理。本文
不承担把 checkout 交付到 Workspace 卷的迁移义务，也不承担修改 Desktop 的义务。

## 继承与修改

| 前序决策 | 处理 |
|---|---|
| [受控 Node 的仓库凭据执行使用独立 OS 身份](20261010-controlled-node-git-authentication-isolation.md)（proposed）D1：M／W／C 三者互不相同，W 与 C 非特权，M 是当前以 root 运行的受信任管理身份 | **修改实现归属**：三个身份的**语义与互异性要求不变**，但 C 不再由 Node 容器内的第三个 UID 承载，而是由独立部署单元的进程身份承载（D3）。D1 的隔离承诺表（W 读不到 C 的秘密、W 无法冒用 C 或 M、C 不拥有 M 的管理权限、M 属于可信计算基）四条**逐条继承** |
| 同上 D2：凭据执行器是部署固定的单一组件，业务请求不能指定它；helper 必须白名单化 | 继承，不变；`helper_path`／`git_path` 是执行器自己的配置项，clone 规格与 Controller 消息都不能选择它们 |
| 同上 D3、D5、D6、D7：单次授权、单次执行绑定与重放防护、仓库范围由授权决定、隔离靠 OS 身份而非 Agent 可读的令牌 | 继承，不变；D7 的"判据是能力而非连通性"在本形态下由容器边界与 mTLS 身份共同满足（D3） |
| 同上 D4、D9、D11、D13、D15：物化禁止面、能力不足时拒绝而不降级、C 自己解析自己物化、崩溃与超时后的清理、`broker-resolver` 由 C 持有 | 继承，不变；`internal/credentialmaterial` 与 `internal/cloneexec` 是这些规则的实现，规则本身不因归属变化而放松 |
| 同上 D8：V1 不引入 sidecar 容器或外部 Git 服务 | **继承禁止面，形态不同**：执行器不是 sidecar、不挂载 Workspace 卷、不使 Sandbox Server 执行 Git、不在同一 Workspace 卷上引入第二个容器（D4）。但"身份边界建立在 Node 容器内"这一**形态选择**不由本形态交付 |
| 同上 D12：checkout 所有权交接（C → W） | **未交付，且本形态下没有可交接的对象**：执行器把 checkout 写在自己的 `checkout_root`，不写 Workspace 卷。交接因此仍是未满足的义务，不是被本文改写的规则（D5） |
| 同上 D14-A／D14-B：开发准入与功能启用是两套门禁 | 继承，不变；本文属于开发准入侧的实现归属记录，**不产生任何启用证据** |
| 同上 D17：无凭据 Git 继续使用既有身份，只有带凭据的 Git 才迁到 C | 继承，不变；执行器只处理带凭据的 clone，匿名 clone 的路径与身份逐位不变 |
| [最小 clone 执行契约](20260918-minimal-clone-execution-contract.md) D2（经[凭据来源修订](20261010-minimal-clone-execution-contract-credential-source-amendment.md)扩展）：凭据来源为按 Project 逐次授权的动态来源，只交付给 C | 继承，不变；"只交付给 C"在本形态下由部署单元边界实现（D3） |
| [clone 副作用移出受理 worker](20260925-clone-git-effects-run-outside-the-admission-worker.md) 与 [终止尝试判为 Interrupted](20260925-terminated-clone-attempt-fails-as-interrupted.md)：clone 的 host 交互与收尾屏障、`Interrupted` 判定 | **形态差异，如实记录**：带凭据的 clone 今天不经过 Node 的 clone 状态机，因此这两份决策的收尾屏障与 `Interrupted` 判定对它不适用；Cloud 以"注册时结果已写入"记账，销毁由执行器自己保证（D4 末段、D5） |
| [用户 Git Token 由独立 Credential Service 承载](../../cloud/identity-access/20261010-user-managed-git-secret-store.md)（proposed）D3、D5、D10：`broker-resolver` 服务身份加上 Cloud 签发的单次授权，兑换时原子消费，消费记录持久在 Store 侧 | 继承，不变；执行器是这两个凭据的第一个正式持有者，本文只确定它属于哪个仓库 |
| [Sandbox Server 生命周期经 Substrate 生效](../../cluster/sandbox-server/20260926-docker-sandbox-lifecycle-via-substrate-effects.md) 不变量 7：Sandbox Server 不执行 Git、不注入凭据 | 继承，不变；执行器不是 Sandbox Server 的一部分，也不经它执行（D4） |

## 背景与现状

### 为什么需要这份 ADR

两份 Node 凭据决策把凭据执行写成"Node 容器内的第三个 OS 身份"：`process.workload_uid` 之外的
一个非 0、与 W 和 M 都不同的 UID，物化目录由入口脚本准备，helper 由部署固定在容器镜像里。那条路线
的实现落点是 `desktop/apps/ora-node` 与 `crates/*`。但 `desktop/` 已被裁决为**历史版本、严格只读**，
而当前正式开发项目是 `cloud/`。于是出现一个必须显式裁决的归属问题：**凭据执行器写在哪个仓库、以什么
形态运行**。项目负责人的裁决给出了方向（正式项目内的独立执行组件、Desktop 只读参考），并要求以最小
范围的后续 ADR 记录归属关系——本文即该 ADR。

### 已经交付的形态

| 组件 | 归属 | 承担什么 |
|---|---|---|
| `cmd/clone-executor` | 正式产品 | 执行器进程：自己的配置、自己的监听、自己的身份、启动闸门与启动清扫 |
| `internal/cloneexec` | 正式产品 | 受约束的 Git 调用、绑定校验、兑换与销毁顺序、mTLS 服务端与两个客户端 |
| `internal/credentialmaterial` | 正式产品 | 物化布局：`0700` 叶子、`0600` 文件、销毁与清扫 |
| `cmd/git-credential-ora` | 正式产品 | 部署固定的 credential helper：只按 host 作用域交付暂存的 token，不存储、不回写 |
| `internal/cloneauth` | 正式产品 | Cloud 侧的 Ed25519 授权签发与执行器侧的验签、仓库规范化 |
| `internal/credentialstore` | 正式产品 | Credential Service 侧的兑换：mTLS 身份、签名校验、以 `O_CREATE|O_EXCL` 为原子单次消费判定 |
| `internal/core` | 正式产品 | Cloud 侧的准入与记账：`admitCredentialedClone`、`CloneDispatcher`、`registerCredentialedClone` |

旧 Desktop 的 `ora-node`、`crates/gitlancer` 与 process guardian 在本形态下**没有被修改，也没有被
依赖**：执行器不 import 任何 Desktop 代码，构建与运行都不需要 Desktop 仓库。

## 决策

### D1：凭据执行器的正式归属是当前正式产品内的独立组件

凭据执行能力属于 `cloud/` 仓库，作为该仓库自己的命令与包存在（见上表），与 Cloud 服务共用构建产物
与镜像，但是**独立的部署单元**：自己的配置文件、自己的 `CLONE_EXECUTOR_` 环境前缀、自己的卷、自己的
身份、自己的信任根。

选择理由：

- **它必须能被独立授权、独立部署、独立审计。** 执行器持有 `broker-resolver` 服务身份——那是"能兑换
  一次凭据"的能力。把它放进 Cloud 服务的进程里意味着 Cloud 的业务进程持有该身份，与
  [Secret Store ADR](../../cloud/identity-access/20261010-user-managed-git-secret-store.md) D3 与
  [隔离决策](20261010-controlled-node-git-authentication-isolation.md) D15 明确否决的形态相同：
  任何 Cloud 侧的缺陷都会同时暴露全部凭据。
- **它不能与 Agent 工作负载同身份。** 正式产品的部署里，Agent 跑在 Workspace 的沙盒容器内，执行器跑
  在自己的容器内，两者是**不同的容器、不同的卷、不同的进程身份**（D3）。
- **配置前缀与加载器独立**：`CLONE_EXECUTOR_*` 不能被任何配置 Cloud 的组件设置，Cloud 也无法通过
  配置自己来继承执行器的物化根或 helper 路径。

### D2：旧 Desktop 保持只读参考，不修改、不依赖、不承载

- `desktop/` 仍然是**严格只读的历史版本**，本文不解除该限制，也不把它重新作为开发目标。
- 允许并已经发生的参考是**只读的**：`gitlancer` 的 clone 命令行约束集合、process guardian 的降权与
  收尾顺序、`ora-node` 的环境卫生做法，都是"读它的理由"的来源，不是被复制的代码。执行器的约束集合
  是独立写下的，并且比 Desktop 的版本更严（例如把 `credential.helper`、`http.proxy`、
  `http.followRedirects`、`credential.useHttpPath` 全部固定为命令行 `-c`）。
- **不依赖**：执行器的构建、测试与运行都不需要 Desktop 仓库存在。
- 因此"Desktop 是历史版本"与"凭据执行已经实现"不矛盾：实现落在正式项目，Desktop 只提供经验。

### D3：执行身份 C 由独立部署单元的进程身份承载，M／W／C 的映射随之确定

| 身份 | 在正式部署中是什么 | 与先前决策的对应 |
|---|---|---|
| **C** | 执行器容器内运行执行器的那个非 root 身份；它拥有物化根、helper 的执行权与 `broker-resolver` 客户端证书 | 与 D1 的"凭据执行身份"语义相同：只执行带凭据的 Git 操作，不 `chown`、不 `setpriv`、不持有 `ingress-writer`／`kek-operator` 身份、不调用任何管理面 |
| **W** | Agent 工作负载，运行在 Workspace 的沙盒容器内 | 与 D1 的 W 语义相同；它与 C **不在同一个容器**，因此不可能与 C 同身份 |
| **M** | 部署管理：生成并交付 C 的身份材料、创建并固定物化根与 checkout 根的属主、在启动时执行清扫前置条件 | 与 D1 的 M 语义相同（准备位置与身份、不代 C 解析、不读取解析结果） |

**形态差异必须如实表述**：先前决策选择的是"同一个容器内三个互不相同的 UID"，本形态选择的是
"两个容器 + 各自的进程身份"。前者靠内核对端 UID 校验与文件模式强制，后者靠容器边界、卷的挂载面与
mTLS 服务身份强制。**不能**把后者表述为"更强的隔离"：它没有减少容器内 root 的风险，也没有引入
namespace／seccomp／只读根；它改变的是边界所在的位置。D1 的隔离承诺表四条在本形态下的对应是：

| D1 的承诺 | 本形态的强制机制 |
|---|---|
| W 读不到 C 的凭据、授权载体与 `broker-resolver` 身份材料 | W 在另一个容器，C 的物化根与身份材料只挂载到执行器容器；W 的容器没有这些挂载 |
| W 无法冒用 C 或 M | 执行器与 Store 都要求并校验客户端证书链与 SPIFFE 身份；W 的容器里没有 C 的私钥，也没有通向执行器监听与 Store 的已配置路径 |
| C 不拥有 M 的管理权限 | C 是非 root 身份，不 `chown`、不 `setpriv`、不读管理卷、不持有 `ingress-writer`／`kek-operator` 身份 |
| M 属于可信计算基 | 不变：生成并交付 C 的身份材料与目录属主的是部署管理，它必然经手这些材料 |

**执行器只接受一个调用者身份**（`dispatcher_identity`，唯一值）。"能派发"等于"能花掉一次授权"，因此
它必须是单一值而不是一个集合；这一条与 D1 的"部署固定"同向。

### D4：D8 的禁止面继承不变，但"边界在 Node 容器内"这一形态选择不由本形态交付

D8 的四条理由逐条对照，说明本形态为什么不违反它、以及它到底约束了什么：

| D8 的理由 | 本形态 |
|---|---|
| 1. Sandbox Server 是"一个 Workspace 一个容器"，拒绝同一 Workspace 卷上的第二个容器 | **不适用，也不被违反**：执行器不挂载任何 Workspace 卷，因此不构成同一卷上的第二个容器 |
| 2. `20260926` 不变量 7：Sandbox Server 不执行 Git、不注入凭据 | **不适用，也不被违反**：执行器不是 Sandbox Server 的一部分，也不经它执行；Sandbox Server 的代码与不变量未被触碰 |
| 3. Workspace 卷是沙盒宿主机上的 Docker 命名卷，外部服务要写它就必须取得宿主机卷访问权，等于把 Node 对 host 交互的所有权让出去 | **不适用**：执行器不写 Workspace 卷，因此没有取得宿主机卷访问权，也没有让出任何 host 观测所有权 |
| 4. 已有实现把 clone 的 host 交互、`process_terminations` 与只读核实建立在"Node 拥有这次 host 观测"之上 | **部分适用，且是本形态的代价**：带凭据的 clone 不由 Node 观测，Node 的 clone 状态机、收尾屏障与 `Interrupted` 判定对它不参与（见下） |

因此本 ADR 的裁定是：**D8 的禁止面（sidecar 容器、集群级外部 Git 服务、让 Sandbox Server 执行 Git、
在同一 Workspace 卷上放第二个容器）全部继承不变；D8 没有覆盖的是"执行器完全位于 Node 之外、且不接触
Workspace 卷"这一形态**——它既不是 D8 的 sidecar，也不是 D8 方案 C 的"外部服务写 Workspace 卷"。

**必须如实记录的代价**：本形态下，一次带凭据的 clone 产出的是**一个 commit 与一个位于执行器
`checkout_root` 的目录**，而不是一个已经交给 Agent 会话的工作目录。Cloud 侧的记账方式是"注册这次
execution 时结果已经写好"（`clone_executions.result`），Node 的 clone 状态机不参与，`20260925` 的
收尾屏障与 `Interrupted` 判定因此对它不适用；"凭据物化已销毁"由执行器自己在应答之前保证，并由启动
清扫兜住崩溃残留。这条差异是**形态差异**，不是对 `20260925` 两份决策的改写：那两份决策描述的是 Node
的受管 clone，而带凭据的 clone 今天不走那条路径。

### D5：本形态明确不满足的义务，以及 fail-closed 的保持

以下义务在今天的实现里**没有证据**，本文如实记录而不是绕过：

| 义务 | 今天的实际状态 |
|---|---|
| D12 的 checkout 所有权交接（C → W），以及"交接后 Agent 可正常读写工作目录而物化已不存在"（D14-B 义务 14） | **未实现**：执行器把 checkout 写在自己的 `checkout_root`，不写 Workspace 卷，因此没有可交接的对象。跨容器把 checkout 交付给 Workspace 是一个**新的架构问题**，本文不裁决 |
| D14-B 义务 1、2、3、11、16、17（以 W 身份读不到 C 的秘密、不能冒用、不能经 IPC 提升身份、多租户与多 Workspace 隔离、Agent 既有 IPC 不回归） | **未取得证据**：这些义务是按"同一个容器内的三个 UID"写的。在容器边界形态下它们该由哪些直接证据满足，属于功能启用步骤的裁决，本文不改写义务清单，也不声称它们已成立 |
| D14-B 义务 12（真实私有仓库 Clone 成功） | **已有直接证据**：`internal/cloneexec` 的验收测试用真实私有 HTTPS 仓库（`git http-backend` + Basic 认证）、真实 Credential Service、真实 helper 二进制与真实 `git` 完成了端到端 clone，并核实 commit。这只是十九项义务中的一项 |
| D14-B 义务 15（身份边界不完整时拒绝启动） | **部分**：执行器在缺少验签公钥、Store 端点或物化根时拒绝启动（D17 的配置缺失表），但"三个身份互异"这一条在容器形态下由部署保证，尚无断言它的启动检查 |
| 匿名 clone 与既有业务链路的回归（裁决第 8 项末句） | **保持**：`clone_dispatch` 默认关闭，关闭时 `requireRepositoryAccess` 对带凭据的 Project 逐位返回 `repository_credential_executor_unavailable`，与本文之前完全相同 |

因此 `repository_credential_executor_unavailable` **继续无条件拒绝**，本文不解除它，也不授权任何部分
解禁。裁决第 8 项的条件（D11-B 与 D14-B 全部适用义务有直接证据）没有满足。

## 不变量

1. 凭据执行能力属于当前正式产品（Cloud 仓库），是独立的部署单元，不与 Cloud 业务进程、Agent 工作
   负载或 Sandbox Server 同进程、同容器、同身份。
2. 旧 Desktop 保持严格只读：不修改、不依赖、不承载凭据执行；参考只能是只读的。
3. C 是非 root 身份，只执行带凭据的 Git 操作；它不 `chown`、不 `setpriv`、不读管理材料、不持有
   `ingress-writer`／`kek-operator` 身份、不调用管理面。
4. 执行器只接受一个调用者身份；"能派发"等于"能花掉一次授权"，因此该身份是单一值。
5. D8 的禁止面不变：不引入 sidecar 容器、不引入写 Workspace 卷的外部 Git 服务、不让 Sandbox Server
   执行 Git、不在同一 Workspace 卷上放第二个容器。
6. 执行器不挂载任何 Workspace 卷；它不取得宿主机卷访问权，也不接管 Node 的 host 观测。
7. 带凭据的 clone 不经过 Node 的 clone 状态机，因此 `20260925` 的收尾屏障与 `Interrupted` 判定对它
   不适用；"物化已销毁"由执行器在应答前保证，崩溃残留由启动清扫兜住。
8. `repository_credential_executor_unavailable` 保持无条件拒绝；本文不产生任何解禁证据。
9. 本文不改写任何其他决策的正文：`20261010` 两份 Node 凭据决策与 Cloud 侧四份凭据决策的正文都
   保持不变，阅读它们的人应同时读到本文。

## 为什么不是这些替代方案

| 方案 | 取舍 |
|---|---|
| 继续在 `desktop/apps/ora-node` 内实现凭据执行 | 最贴近先前决策的原始落点；但 `desktop/` 已被裁决为历史版本并严格只读，把当前正式能力建在历史仓库上会让它随 Desktop 一起停止维护，与裁决第 1 项直接冲突 |
| 把执行器并入 Cloud 服务进程 | 部署最简单，少一个容器；但 Cloud 业务进程会持有 `broker-resolver` 身份，任何 Cloud 侧缺陷同时暴露全部凭据，正是 Secret Store D3 与隔离决策 D15 否决的形态 |
| 把执行器并入 Sandbox Server | 复用已有的容器管理面；直接违反 `20260926` 不变量 7（Sandbox Server 不执行 Git、不注入凭据），并让 Sandbox Server 成为凭据路径的可信计算基 |
| 把执行器作为 Workspace 的 sidecar 容器 | 隔离更强、checkout 天然落在 Workspace 卷上；违反 D8 理由 1 与 `20260926` 不变量 7，需要 amendment 两条不变量并新增跨容器交付通道，超出本次范围 |
| 让执行器克隆后把 checkout 写入 Workspace 卷 | 一步补齐 D12 与义务 14；但需要宿主机卷访问权或一条新的跨容器交付通道，等于让出 Node 对 host 交互的所有权（D8 理由 3）。本文把它留作未决问题，而不是顺手实现 |
| 复制 Desktop 的 helper 与 clone 约束代码到正式产品 | 省一次设计；但会把 Desktop 的 `validate-then-use` 缺口与"未清除 `credential.helper`／代理键"的问题一起带过来，而隔离决策 D2 要求这些键在凭据路径上被显式固定 |
| 让 Cloud 的业务进程签发授权的同时也持有解析能力 | 少一个组件；违反"Cloud 永远不解析秘密"（Secret Store D3）与 D15 |

## 失败与恢复语义

| 情形 | 行为 |
|---|---|
| 执行器未部署或未配置 | `clone_dispatch` 保持关闭；带凭据的 Project 逐位返回 `repository_credential_executor_unavailable` |
| 执行器声明了配置但缺验签公钥、Store 端点或物化根 | 启动即拒绝，不降级运行（D17 的配置缺失表） |
| 执行器崩溃或容器被杀 | 下一次启动的清扫在受理任何 job 之前清空物化根；清扫未完成即不绑定监听 |
| Cloud 无法连接执行器 | 本次 clone 记为失败，reason code 为执行器不可用；不回退匿名 clone，也不回退静态凭据 |
| 执行器返回拒绝码 | 按拒绝码记为失败；授权已被消费的，重试必须申请新授权 |
| Desktop 仓库不可用 | 不影响正式产品的构建、测试与运行：执行器不依赖它 |

## 本决策未解决的问题

- **checkout 如何交付到 Workspace**：带凭据的 clone 今天产出 commit 与执行器本地的目录，而 Agent 的
  会话工作目录在 Workspace 卷上。跨容器交付需要宿主机卷访问权或一条新的交付通道，两者都会触及
  D8 理由 3 与 `20260926` 不变量 7。**这是启用凭据 clone 之前必须由项目负责人裁决的新增架构冲突**，
  本文只记录它，不选择方案。
- **D14-B 义务在容器边界形态下的证据映射**：义务 1、2、3、11、16、17 是按容器内 UID 边界写的。
  哪些直接证据在容器形态下等价、哪些需要新增（例如"W 的容器里没有 C 的挂载与私钥"的断言式验收），
  属于功能启用步骤的裁决。
- **三个身份互异的启动检查**：容器形态下"W 与 C 不同容器"由部署保证，但尚无一条断言它的启动检查；
  是否需要一个显式的部署自检，与 D9／D17 的启动闸门一并裁决。
- **受控路径上是否需要 Node 侧的凭据执行**：若未来要求 checkout 落在 Workspace 卷上，Node 侧的身份
  边界（D1 的原形态）可能重新成为必需。这取决于上一条未决问题的裁决结果。
- **旧 Desktop 中尚未被参考的安全做法**：`crates/process-runtime` 的收尾与 process journal 语义是否
  有正式产品需要吸收的部分，本文不作判断。

## 落地顺序

1. **归属与形态记录——文本已就绪，结论待评审**：执行器的组件清单与部署形态已写定；本文保持
   `proposed`，评审通过后再据此标记 `approved`。本步不产生代码或测试证据，也不解除任何 fail-closed。
2. **正式执行器实现——代码已提交，端到端可达性未成立**：`cmd/clone-executor`、`internal/cloneexec`、
   `internal/credentialmaterial`、`cmd/git-credential-ora` 与 Cloud 侧的准入接线已提交，
   `internal/cloneexec` 的验收测试全绿；但**默认部署下这条通道走不通**：配置层不允许
   `git_server.sandbox_base_url` 与 `clone_dispatch.enabled` 同时启用，执行器也不挂载 Workspace 卷，
   clone 的 checkout 无从交付到 Workspace。因此本文**不声称本步已完成**——它的归属与去留由
   `ora-space/circus#134` 裁决。
3. **部署集成——代码已提交，默认未启用**：`cluster/` 增加了执行器服务、身份与目录的 setup 步骤、
   Cloud 侧签名密钥与验签公钥的交付；默认不启用 `clone_dispatch`，因此部署栈行为逐位不变。这一步
   同样随 `#134` 的结论处理。
4. **解禁——未开始，且不属于本文范围**：只有裁决第 8 项的条件满足后才可移除
   `repository_credential_executor_unavailable`；届时先裁决"checkout 如何交付到 Workspace"，再按最终
   实现同步本 ADR、隔离决策的 D14-B 与 Cloud 侧 D11-B。

保持不变的行为：clone 的对外协议（`CloneExecutionSpec` 的三个字段）、Cloud 业务库 schema（无新
migration）、匿名公开仓库 clone 与 Agent 会话链路的逐位行为、旧 Desktop 仓库的内容、以及"能力不足时
拒绝"的 fail-closed。从本文生效开始改变的行为：**凭据执行器的代码归属与部署形态**——它属于当前正式
产品，以独立部署单元运行，不再落到旧 Desktop。
