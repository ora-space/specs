---
status: proposed
---

# 受控 Node 的仓库凭据执行使用独立 OS 身份，Agent 工作负载不得与之同身份

受控 Node 执行私有仓库 clone 时，凭据材料只在一个部署固定的**凭据执行身份 C** 下可见。C 与 Agent 工作
负载身份 W、Node 管理身份 M 三者互不相同；Broker 只把一次性的 clone 授权交付给 C。Agent 即使可以运行
`git`、读取文件系统与环境变量，也读不到凭据材料，也不能兑换 Broker 授权。

> **阅读入口（后续决策）**：本 ADR 的 D1 把 C 描述为受控 Node 容器内的第三个 OS 身份。
> [凭据执行器归属当前正式产品](20261012-credential-executor-belongs-to-the-current-product.md)（`proposed`）
> 把 C 的实现归属调整为**当前正式产品的独立部署单元**，并逐项说明 D1、D8、D12 与 `20260925` 的继承或
> 形态差异。D2–D7、D9–D11、D13–D17 的规则本身不变；凡本 ADR 以"容器内第三个 UID"表述的地方，当前
> 形态以该后续决策为准。

三个身份的记法在全文沿用，定义如下：

| 身份 | 是什么 | 特权 |
|---|---|---|
| **M** | 受信任管理身份：容器入口脚本、Node、host、guardian 及其同身份代码 | **当前以 root 运行**，是容器内的信任根 |
| **W** | Agent 工作负载身份：插件及由其派生的每个进程 | 非特权、非 root |
| **C** | 凭据执行身份：只执行带凭据的 Git 操作 | 非特权、非 root |

- 三者必须**互不相同**：`W ≠ C`、`W ≠ M`、`C ≠ M`。
- M 是 root **不是缺陷而是前提**：入口脚本需要它做 `chown`／`chmod`、准备挂载点并执行降权
  （`node-entrypoint.sh:26-27`、`managed.rs:116`）。本 ADR 因此**不**宣称"三个身份都是非 root"，
  也不把"M 非 root"当作目标状态。
- M 是信任根：本 ADR 的隔离承诺**不**覆盖"M 被攻破"这一情形（不变量 12）。

**隔离承诺的准确表述。**本 ADR 的隔离是**单向**的，必须按下面四条陈述，**不得**简化为
"M／W／C 互相完全不可读"或"三个身份彼此不可读"：

| # | 承诺 | 强制机制 |
|---|---|---|
| 1 | W **读不到** C 的凭据、授权载体、`broker-resolver` 服务身份材料（证书／私钥）与凭据物化文件 | 文件模式 + 目录可遍历性（D7、D11、D15） |
| 2 | W **无法冒用** C 或 M：不能以 C 的身份连接 Broker／Store，也不能以 M 的身份使用控制通道、管理接口或管理 socket | 控制通道证书由 M 拥有并置于 W 不可读的 `/run/ora`（D10）；管理 IPC 按内核 `peer_cred().uid()` 拒绝其他身份（D10、D16） |
| 3 | C **不拥有 M 的管理权限**：它是非特权身份，不 `chown`、不 `setpriv`、不读 `/run/ora` 的 mTLS 材料、不调用管理面，也不持有 `ingress-writer`／`kek-operator` 服务身份 | D1 的身份定义与 D10 的特权清单 |
| 4 | M **属于可信计算基**：容器内 root 在能力上可以读取 C 的文件；UID 隔离**不**保证"M 被攻破后凭据仍安全" | 不变量 11、12 |

这四条**只**约束 W→C／M 与 C→M 两个方向。M 对 W 与 C 始终保有 root 的读取能力（那是 D10 准备目录、
执行降权与清扫的前提），本 ADR 不承诺 C 与 W 之间、以及 M 与其他身份之间的完全互不可读。

当前部署**没有**这条边界：`process.workload_uid` 被设为 `1000`，而同一个值同时被用于受管 Git 命令
（`managed.rs:39-55`）和 Agent 插件（`service/agents.rs:87-95`），因此 Node 内 **W 与受管 Git 同值**。本 ADR
描述的是**目标状态**，不是既有事实。（D17 只把**带凭据**的受管 Git 迁到 C；无凭据的受管 Git 继续使用
这个值，因此"W == 1000"在目标状态下依然成立——变的是凭据执行不再用它。后续决策
[凭据执行器归属当前正式产品](20261012-credential-executor-belongs-to-the-current-product.md) 把 C 放到
Node 之外的独立部署单元，Node 内今天仍然没有 C。）

本 ADR（工作名 **Controlled Node Git Authentication Isolation**）只定义 Node 侧的身份边界、秘密数据流、
失败语义与验收义务；Token 的收集、Secret Store 与 Broker 授权的**签发**由 Cloud 侧决策负责
（[用户自助 HTTPS Git 凭据](../../cloud/identity-access/20261010-user-managed-git-credentials.md)、
[用户 Git Token 由独立 Credential Service 承载](../../cloud/identity-access/20261010-user-managed-git-secret-store.md)）。
涉及的模块是 `apps/ora-node`（`ProcessConfig`、受管执行、clone 状态机、session 工作负载）、
`crates/process-runtime` 与 `crates/process`（进程身份与 IPC），以及 sandbox-server 的容器创建约束；
`crates/gitlancer` 的 clone 命令行约束继续沿用。

本文的内容已由项目负责人在运行 issue `ora-space/circus#29` 的授权中确认方向，但**尚未经过 specs
的 PR 评审**，按 `specs/AGENTS.md` 保持 `proposed`，评审通过后再据此标记 `approved`。**该授权确认的是
决策内容与其安全边界，既不表示已经实现，也不解除任何 fail-closed**。核心用例已按本文的方向先行登记，
证据状态如实标注为 `Missing`——隔离、物化与销毁都还没有实现，因此本文的落地顺序第 2 步及之后都还没有
证据。

兼容性边界：本 ADR 不改变 clone 的对外协议（`CloneExecutionSpec` 仍只有 `node_id`／`repository`／
`branch`）、不改写 Node 数据库 schema、不改变已实现的 clone 收尾与 `Interrupted` 判定、不解除
`repository_credential_executor_unavailable`。它要求的是：在身份边界真正成立之前，凭据 clone 继续
被拒绝，而不是用"配置了不同 UID"的说法放行。

**关于 UID 边界强度的声明（贯穿全文）**：本 ADR 选择 UID 边界，是因为在当前交付中它是唯一由内核强制、
且已经存在的执行边界。**不同 UID 不是绝对隔离**：它不保护容器内的 root（本 ADR 中管理进程 M 就是容器
内的 root，可以读取任何 UID 的文件），不提供内核逃逸防护，不阻止同 UID 进程之间的 `/proc` 读取与
`ptrace`，也不阻止网络外发。把这些风险说成已被解决会构成虚假保证；它们的处置见 D10、D13 与"本决策未
解决的问题"。同理，**不得**把本 ADR 的隔离简化为"M／W／C 三者互相完全不可读"——承诺的准确内容只有
D1 表中的四条，其中并不包含"M 读不到 W 或 C"。

## 继承与修改

| 前序决策 | 处理 |
|---|---|
| [根决策](0-clone-selected-repository-branch.md)：目标、职责，以及"当前信任本机 Node…不据此认定私有 Git 仓库无需凭据" | 继承，不变 |
| [最小 clone 执行契约](20260918-minimal-clone-execution-contract.md) D2：「HTTPS 可使用部署已配置的非交互凭据来源」，凭据由 Node 部署提供 | **修改**：凭据来源扩展为按 clone 单次授权的动态来源；部署只提供固定的 helper 程序与配置引用。该修改的**权威正文**由 [最小 clone 执行契约 D2 的凭据来源修订](20261010-minimal-clone-execution-contract-credential-source-amendment.md)（proposed）承载，`20260918` 正文不改写 |
| 同上不变量 4：凭据不经业务消息、命令文本、日志或结果传播 | 继承，不变；本 ADR 把禁止面扩展到进程环境、文件描述符、磁盘、`.git/config` 与 Broker 授权载体 |
| 同上 D4：禁用 hooks 不等于沙盒，可信部署需明确允许的辅助程序 | 继承，不变；本 ADR 把它变成落地要求（helper 必须白名单化） |
| [clone 副作用移出受理 worker](20260925-clone-git-effects-run-outside-the-admission-worker.md)：执行器只做 host 交互 | 继承，不变 |
| [终止尝试判为 Interrupted](20260925-terminated-clone-attempt-fails-as-interrupted.md)：收尾屏障与失败码 | 继承，不变；本 ADR 把"凭据物化必须随收尾销毁"接在它上面 |
| [当前信任本机 Node，宿主代次只用于旧会话失效](../process/control/0-authorized-generations-and-leases.md) D2：「管理进程及同身份代码须可信」 | **修改**：仅就仓库凭据路径收紧——凭据不再依赖"同身份代码可信"这一前提 |
| [显式纳管与清理证据](../process/containment/guarantees/0-explicit-containment-and-cleanup-evidence.md) D3：纳管范围不扩展为所有外部副作用 | 继承，不变；本 ADR 不宣称阻止网络外发 |
| [Docker 沙盒生命周期经 Substrate 生效](../../cluster/sandbox-server/20260926-docker-sandbox-lifecycle-via-substrate-effects.md) 不变量 7：Sandbox Server 不执行 Git、不注入凭据 | 继承，不变；并因此否决 sidecar 形态 |
| [minicloud 本机 Web 客户端内嵌 Controller](../../minicloud/runtime/0-local-web-client-embedded-controller.md)：「私有仓库凭据仍由 Node 的既有部署提供，不在浏览器收集」 | **历史契约，不需 amendment**：该条限定在 minicloud 本机开发边界内（同段声明 minicloud 不投产、不建认证体系），而 minicloud 已整体退役（[退役决策](../../minicloud/runtime/20260928-retire-minicloud.md)，`implemented`，不变量 1：仓库中不再有浏览器直连 Controller 的入口）。它不约束当前 Cloud 的凭据功能；适用范围见 [Cloud ADR](../../cloud/identity-access/20261010-user-managed-git-credentials.md) 的继承表 |
| [用户 Git Token 由独立 Credential Service 承载](../../cloud/identity-access/20261010-user-managed-git-secret-store.md)（proposed）D3、D5、D10：解析需要 Store 侧 `broker-resolver` 服务身份**加上** Cloud 签发的单次授权，两者由 Store 自己校验；授权在签发时未使用、由 Store 在**兑换时原子消费**，消费记录持久在 Store 侧，重放一律拒绝 | 继承，不变；本 ADR 定义**消费**侧的校验与强制——凭据执行身份 C 持有解析服务身份并出示授权（D15），Cloud 的业务进程与 M 都不解析秘密；未拿到秘密的失败一律申请新授权（D5） |
| [仓库凭据归属连续性](../../cloud/identity-access/20261010-repository-credential-ownership-continuity.md)（proposed）D3、D4：成员／用户停用原子冻结 `personal`／`unknown`，重新加入不自动解冻 | 继承，不变；本文 D5 只定义消费侧的强制（冻结后拒绝签发新授权、已签发未使用的授权随到期失效），不新增冻结规则 |

## 背景与现状

### 已经就位、可以复用的机制

- **降权原语已存在，且已经被部署使用。** `ProcessConfig.workload_uid`（`apps/ora-node/src/managed.rs:29`）
  在 `isolate_workload` 中把受管程序包成
  `/usr/bin/setpriv --reuid=<uid> --regid=<uid> --clear-groups --no-new-privs -- <program>`
  （`managed.rs:39-55`）。启动闸门要求管理进程是 root、目标 UID 非 0、基础环境为空，且 `/usr/bin/setpriv`
  必须是 root 拥有的可信路径（`managed.rs:113-127`）。`main.rs:36-46` 已经把"网络管理必须配置独立工作
  负载身份"写成启动拒绝条件。部署模板 `sandbox-server/node-config.example.json` 的
  `process.workload_uid` 为 `1000`，容器镜像也以 `useradd --uid 1000 … node` 建立了同名用户
  （`desktop/docker/Dockerfile:60`）。
- **Agent 也走一条身份降权路径，但与 Git 是两套机制、同一个 UID。** `service/agents.rs:87-95` 用
  `process.workload_uid` 构造 `LinuxChildIdentity::new(uid, uid)`，`session/workload.rs:97-120` 通过
  `TokioProcessSpawner::running_as(*identity)` 应用它；`crates/process/src/tokio_process.rs:40-62` 在
  `pre_exec` 中降权并设置 `umask 0o077`。容器入口脚本的注释确认了这件事：
  *"Git workloads run as UID/GID 1000 through the scoped process boundary, and so do Agent plugins"*
  （`desktop/docker/node-entrypoint.sh:5-7`）。
- **对端 UID 校验已经贯穿本机 IPC。** host／guardian／helper 的 Unix socket 以 `0600` 建立，并在每条
  连接上按内核 `peer_cred().uid() == owner` 拒绝其他身份（`crates/process-runtime/src/guardian.rs:36-42,128-141`、
  `crates/process-runtime/src/host/server.rs:66-72`、`crates/process-client/src/transport.rs:33-37`、
  `crates/utils/src/local_ipc.rs:12-47`）。这是一条**由内核强制**的边界，可以原样复用于凭据通道。
- **Agent 与 Node 之间不是 socket，而是 stdio 管道。** Agent 是一个 Deno 插件进程，协议是 4 字节长度
  前缀 + 类型字节 + JSON 的 JSON-RPC，跑在 Node 持有的 stdin／stdout 上（`crates/plugin-runtime/src/lib.rs:147-170`）；
  Agent 请求的宿主能力（`ora/childprocess/*`、`ora/storage/*` 等）**复用在同一条流上**，由 Node 以 M
  的身份代它执行，且子进程仍以 W 的身份启动（`apps/ora-node/src/session/workload.rs:119`）。
  host／guardian 的 socket 客户端全部位于 M 侧（`apps/ora-node`、`crates/process-runtime/src/host*`），
  `crates/agent-runtime`、`crates/plugin-runtime`、`crates/plugin-lifecycle` 都不依赖 `process-client`。
  这就是 D16 把"能力"而非"连通性"作为判据的原因。
- **受管执行路径已有环境与文件描述符卫生。** `LinuxBestEffort::spawn` 先 `env_clear()` 再套用
  `spec.env`（`crates/process-runtime/src/linux_best_effort.rs:151-158`），随后
  `configure_linux_detached_child`（`crates/utils/src/process/linux_detached.rs:9-26`，由
  `linux_best_effort.rs:159-160` 调用）在 exec 前 `setsid()` 并
  `close_range(3, MAX, CLOSE_RANGE_CLOEXEC)`，注释写明"Workloads must not inherit guardian locks,
  SQLite descriptors or private endpoints"。**注意这条卫生只覆盖受管路径。**
- **Git 侧约束已经收紧。** clone 用固定环境（`GIT_CONFIG_NOSYSTEM=1`、`GIT_CONFIG_GLOBAL=<部署文件>`、
  `GIT_TERMINAL_PROMPT=0`、`GIT_ASKPASS=""`、`SSH_ASKPASS=""`、`GIT_LFS_SKIP_SMUDGE=1`、
  `GIT_OPTIONAL_LOCKS=0`，`repository/config.rs:103-137`），并按命令前置 `-c` 约束（`protocol.allow=never`、
  `protocol.https.allow=always`、`protocol.ssh.allow=never`、`core.hooksPath=/dev/null`、`core.askPass=`、
  禁用 submodule 与 LFS filter，`config.rs:140-161`）。受管 Git 进程**不继承** `ProcessConfig.environment`
  ——该字段在隔离模式下必须为空（`managed.rs:116`），且 clone 环境由 `CloneConfig::environment()` 显式构造。
- **clone 目标目录的所有权已按工作负载身份处理。** 目录以 `0700` 创建，并在设置了工作负载 UID 时
  `chown` 给它（`repository/drive.rs:261-263`）。
- **容器已经加固。** `CreateContainer` 的请求体现在包含 `Tmpfs{"/run/ora": "rw,noexec,nosuid,size=1m,mode=0700"}`、
  `CapDrop: ["ALL"]`、`CapAdd: ["CHOWN","DAC_OVERRIDE","FOWNER","SETUID","SETGID","KILL"]`、
  `SecurityOpt: ["no-new-privileges:true"]`（`sandbox-server/internal/docker/docker.go:168-178`）。注意这些
  键**硬编码在请求体里**，`ContainerSpec`（`docker.go:147-154`）本身仍没有 `User`／`ReadonlyRootfs`
  字段，因此加固不是可配置项，容器根文件系统仍可写。
- **控制通道的 mTLS 已经就位。** `securedContainerSpec`（`sandbox-server/internal/lifecycle/authorization.go:37-64`）
  把 `control.listen` 覆写为 `mutual_tls_websocket` 并写入 `/run/ora/{node-cert,node-key,ca,controller-cert}.pem`，
  再追加 `NodeIssuer.Environment(...)` 产生的四个证书环境变量；入口脚本把它们解码到 `/run/ora` 并
  `chmod 0600`（`node-entrypoint.sh:42-52`）。`sandbox-server/cmd/sandbox-server/main.go:111-114` 要求
  `-lifecycle-addr` 必须同时给出 `-management-dir`，否则拒绝启动（*"unauthenticated management is retired"*）。
  因此早前"部署模板与受控策略自相矛盾、受控 Node 未必可部署"的判断**已经失效**。

### 目前不存在的边界

- **Agent 与 Git 在同一个 OS 身份下运行。** 见上：`workload_uid = 1000` 同时施加于受管 Git 命令与 Agent
  插件。`managed.rs:116` 要求"隔离工作负载需要 root 管理进程、非 root 工作负载身份、空的基础环境"，
  这三条在当前部署中都已满足——**隔离开关是打开的，只是两边是同一个 UID**。因此"任何 git 能读到的
  凭据，Agent 代码也能读到"是当前部署的既成事实。
- **Agent 路径没有环境与文件描述符卫生。** `crates/process/src/spec.rs:29-39` 的 `ProcessSpec` 没有 UID
  字段，也没有 `env_clear`；`ProcessTree::configure_command`（`crates/process/src/tree.rs:69-83`）只设置
  `process_group(0)`，**没有** `close_range`。`session/host.rs:60-64` 的注释明确承认
  *"The rest of the host environment (model credentials, proxy settings) still reaches the agent: these
  variables are layered on top of it, never instead."* 于是 Agent 继承 Node 的完整环境（含容器环境变量）
  与 Node 持有的文件描述符。插件以 `--allow-run --allow-read --allow-env --allow-net` 运行，工作目录就是
  clone 出来的 checkout。
- **容器内管理进程 M 是 root。** 入口脚本以 root 执行 `chown root:root /var/lib/ora` 与 `chmod 0711`
  （`node-entrypoint.sh:26-27`），并把证书写到 root 拥有的 `/run/ora`；`managed.rs:116` 的启动闸门也要求
  `geteuid() == 0`。`desktop/docker/Dockerfile` 的 `node` 阶段没有 `USER` 指令（只有 `controller` 阶段有
  `USER controller`）。Dockerfile 第 54-55 行"the entrypoint runs the process host and the Node as the
  unprivileged `node` user"的注释与入口脚本的实际行为不一致，属于需要修正的文档偏差。
- **没有 namespace、cgroup、seccomp、AppArmor 或 userns。** 实际生效的适配器是 `LinuxBestEffort`，其
  `capabilities()` 自报 `PlatformCapabilities::BestEffortOnly`（`crates/process-runtime/src/linux_best_effort.rs:100-104`）；
  它只做 `setsid` 与 `close_range`。因此 D1 要求的 UID 边界**只能**由内核的 UID 判定与文件模式提供。
- **`/proc` 未受保护，且 UID 相同即等价。** 同 UID 进程在默认内核配置下可以读取彼此的
  `/proc/<pid>/environ`；`ptrace` 是否可用取决于宿主 `kernel.yama.ptrace_scope`，不能假定被禁用。这是
  W ≠ C 之后**仍然存在**的风险：它保护不了"W 与 C 之间的信息流"，只保证 C 的文件与环境不对 W 开放。
- **没有把凭据材料交给 C 的通道。** `RunSpec`（`crates/process-protocol/src/run.rs:34-47`）只有
  `program`／`args`／`cwd`／`env`／`descendants`／`output`／`lifetime`——没有文件描述符传递、没有 socket、
  没有额外文件挂载。受管路径的 `close_range(..., CLOEXEC)` 也会关掉任何想跨 exec 传递的 fd。
- **Git 配置的 validate-then-use 缺口与仓库本地配置未被收窄。** `git_config` 只在启动时按内容校验一次
  （`config.rs:32`），此后每次 clone 都经 `GIT_CONFIG_GLOBAL` 重新读取该文件（`config.rs:116`）；同时
  `constrain()`（`config.rs:140-161`）**没有**清除 `credential.helper`、`http.<url>.*`、`http.extraHeader`
  或代理键，而 checkout 内的 `.git/config` 对 Git 仍然有效（命令行 `-c` 优先级更高，但这些键没有被用上）。

结论：**今天不存在任何能把凭据材料与 Agent 代码隔开的边界。** 降权开关是打开的，容器也加固了，mTLS 也
就位了，但这些都不改变"W 与 C 是同一个 UID"这一事实；同一个 UID、同一个文件系统、同一组由对端 UID
校验保护的 socket，意味着 Agent 能读到的任何凭据，Git 也能读到，反之亦然。
`node/process/control/0-…` D2 已经写明"对端 UID 校验防止错接，不构成同 UID 恶意程序隔离"——本 ADR 要做
的第一件事就是把这条边界真正建立起来，而不是继续依赖"不同 UID"的说法，更不是把"已经配了 `workload_uid`"
当作边界已经存在。

## 决策

### D1：凭据路径必须有三个互不相同的 OS 身份，且不得把 UID 边界表述为绝对隔离

受控 Node 上同时存在管理身份 M（Node／host／guardian／入口脚本）、Agent 工作负载身份 W、凭据执行身份
C。要求 `W ≠ C`、`W ≠ M`、`C ≠ M`；**W 与 C 非特权且非 root，M 是受信任管理身份并当前以 root 运行**
（定义见开头）。三者互不相同是硬要求，但三者的特权并不相同：把 M 也表述为"非 root"或"非特权"是
与部署事实相反的措辞，本 ADR 不允许出现。

W 的权限**上限**是"非特权工作负载"：它不能直接取得 C 或 M 的权限（不同 UID + 文件模式 + socket 对端
UID 校验），也不能经管理接口间接提升（控制通道的 mTLS 证书由 M 拥有、位于 W 不可遍历的 `/run/ora`
`0600`，D10）。这条上限必须由验收义务 16 直接证明，而不是从"配了不同 UID"推断。

选择理由：平台真正强制执行的只有内核的 UID 判定——文件模式与 socket 对端校验。既然 namespace、
cgroup、seccomp 与 userns 都不存在，UID 边界就是**当前唯一可用**的执行边界；而 `main.rs:36-46`
已经确立了"网络管理必须配置独立工作负载身份"这一同类闸门，本决策把它扩展到凭据路径。

**边界强度必须如实表述**：`W ≠ C` 保证的是"W 不以 C 的身份读写文件、连接 C 的 socket、继承 C 的环境"。
它**不**保证：

- 容器内的 root（即 M）无法读取 C 的文件——M 是 root，`DAC_OVERRIDE` 也在 `CapAdd` 白名单里；
- 容器逃逸或内核漏洞被阻止——`CapDrop: ["ALL"]` 与 `no-new-privileges` 收窄了攻击面，但不构成隔离保证；
- 同 UID 进程之间的 `/proc` 读取与 `ptrace` 被阻止；
- 网络外发被限制。

因此本 ADR 不宣称"凭据对容器内任何进程不可读"，只宣称"凭据对 **W** 不可读"。M 必须被当作可信计算基
（D10 规定它的职责与禁止事项），它的可信度是 V1 的前提，不是本 ADR 的结论。**"M 被攻破之后凭据仍
安全"不是本 ADR 的承诺**：M 是 root，一旦 M 上的代码被攻破，它可以读取 C 的文件、冒充 C、杀死 C 的
进程；UID 隔离对此不提供任何保证。

注意 `workload_uid` 今天是一个**单一**值，被施加到每一个受管工作负载（`managed.rs:257` 与
`service/agents.rs:87-95` 共用同一份 `ProcessConfig`）。它无法同时表达"Agent = W、Git = C"，所以直接
复用现有字段不足以满足 D1；字段形态（新增身份字段，或把 Agent 也纳入受管路径）留给实现，但**语义要求
是明确的**。

`crates/gitlancer` 的直连 runner 不参与本决策：它不 `env_clear`、继承宿主环境（`exec/runner.rs:77-93`），
是 Desktop／backend 的路径，不是受控 Node 的 clone 路径。

### D2：凭据执行器是部署固定的单一组件，业务请求不能指定它

只有凭据执行器可以持有仓库凭据材料。它的程序路径、搜索路径与配置引用由部署固定，与 `git_program`
同级对待，**不能**由 Controller 消息、clone 规格或任何业务输入选择。这与
`20260918` D4「可信部署需明确允许的辅助程序，不能把任意环境继承称为执行隔离」一致。

因此凭据 helper 必须白名单化，而不是放开配置策略：

- 当前 `validate_cloud_policy`（`repository/config.rs:28-55`）只允许部署 gitconfig 出现 `[http] sslCAInfo`，
  并禁止一切非 `Disabled` 的 SSH。加入 credential helper 必须走**显式允许的键**，不能把该策略放宽为
  "接受任意 Git 配置"。
- 当前 `constrain`（`config.rs:140-161`）**没有**清除 `credential.helper`、`http.<url>.*`、
  `http.extraHeader` 与代理键。凭据路径必须显式固定它们，并补齐 `credential.useHttpPath` 一类决定
  "匹配粒度"的键（D6）。
- **仓库本地配置也必须被收窄**：`.git/config` 由 checkout 的属主（今天就是 W）可写，而 Git 会读取它。
  仅靠 `-c` 覆盖 `GIT_CONFIG_GLOBAL` 不足以防住它，必须在凭据路径上用命令行 `-c` 显式覆盖
  `credential.helper`、`http.*` 与代理键（命令行配置优先级高于仓库本地配置）。
- **validate-then-use 缺口必须在凭据路径上关闭**：`git_config` 只在启动时按内容校验一次
  （`config.rs:32`），此后 Git 每次 clone 都通过 `GIT_CONFIG_GLOBAL` 重新读取该文件（`config.rs:116`），
  中间的文件改动不会被重新检查。凭据路径不得依赖一个可变的磁盘配置来约束自己——要么在使用时重新
  校验，要么把全部与凭据相关的键都固定为命令行 `-c`。

### D3：凭据只经 Broker 单次授权取得，且只交付给凭据执行身份 C

每次 clone 使用一次独立授权，绑定 tenant、credential owner（用户）、project、workspace、
clone execution、credential ID、credential version、repository identity、`operation=clone`，
并带到期时间与单次使用约束。授权还必须绑定**接收它的 Node 实例与运行代次**，使被 fencing 掉的旧实例
不能兑换（见 D5）。

- **解析发生在凭据执行侧，且该服务身份由 C 持有**（D15）：它同时向 Store 出示自己的 `broker-resolver`
  服务身份与本次授权，由 Store
  自己校验两者后返回一次秘密（见
  [Secret Store ADR](../../cloud/identity-access/20261010-user-managed-git-secret-store.md) D3、D9）。
  Cloud 的业务进程不解析秘密，只签发授权与维护元数据；管理身份 M 也不解析。Agent 的 W 身份既没有
  服务身份也没有授权，因此既不能出示也不能兑换。
- 持有 credential ID 不等于取得秘密：ID 只是元数据主键，不是 bearer token。
- **授权载体本身是秘密**：按
  [Secret Store ADR](../../cloud/identity-access/20261010-user-managed-git-secret-store.md) D5，它是 bearer
  凭据，因此不得写入 Node 数据库、process journal、事件、日志或 `Debug` 输出。Node 侧沿用
  `revision/grants.rs` 的既有形态：只在内存、重启后重新申请。
- 授权有效期是短时的，且不长于 `command_timeout_ms`；短时授权不等于短时 Token——Token 本体在
  Secret Store 中，其轮换与冻结语义沿用既有归属连续性决策。
- Secret Store 或 Broker 不可用时**明确失败**，不回退匿名 clone，也不回退静态个人凭据。

### D4：秘密的物化形式与禁止面

凭据材料在 clone 期间只存在于 C 可读的位置，并在本次 clone 结束后销毁。明确禁止：

| 位置 | 要求 |
|---|---|
| Controller／Node 业务协议 | 不出现（现状已满足：`CloneExecutionSpec` 无凭据字段） |
| 命令文本（argv） | 不出现；clone URL 必须是**不含凭据**的地址 |
| 进程环境 | 不进入 W 可见的环境；M 的环境也不携带 |
| 文件描述符 | 不跨 exec 传给 W；受管路径的 `CLOEXEC` 语义必须保持 |
| Node 数据库、process journal、事件与结果 | 不出现，连可复原形式也不出现 |
| 日志（含 Git 错误输出） | 不出现；Git 输出本身也不能被当作秘密载体 |
| 磁盘上的 Git 配置 | 不写入 `credential.helper` 的密钥、不写入 `http.<url>.extraHeader` |
| checkout 的 `.git/config` | `remote.origin.url` 必须是无凭据 URL；不写入任何凭据键 |
| W 可读的目录 | 物化位置必须在 W 不可遍历的目录内，且不在 checkout 树内 |
| Broker 授权载体 | 不落任何持久介质（D3） |

**当前部署的物化位置不满足这一条**：`/run/ora` 是 root 拥有的 `0700` tmpfs（`docker.go:173` 与
`Dockerfile:61`），C（UID 1000）既不能遍历也不能在其中创建文件。因此凭据路径需要一个新的、
**由部署创建、对 C 开放而对 W 关闭**的物化位置，其权限模型见 D11。

### D5：单次 clone execution 绑定与重放防护

- 授权绑定到具体的 clone execution；Node 不能把它用于另一次执行。
- 失败、超时、`Interrupted` 或进程崩溃后的重试**必须申请新授权**，不得重放旧授权。
- 授权绑定接收 Node 的实例与运行代次：Runtime Control 的 fencing 一旦推进代次，旧实例持有的
  未使用授权即失效。这与 `node/process/control/0-…` D3「旧绑定不能重新取得执行资格」同向。
- 到期即失效，不续期；Node 侧不得缓存授权跨执行复用。

### D6：仓库范围由授权决定，重定向／代理／helper 不能扩大它

- 由**凭据执行器**（而非 Git）判断请求的 host 与 path 是否匹配授权中的 repository identity；
  不匹配时不返回任何材料。
- 沿用 `protocol.allow=never` 与仅 HTTPS 的现有约束（`config.rs:140-161`）。
- 跨 host 重定向不得携带凭据。这一点必须被**测试**证明，而不是按 Git 文档假定；并且凭据路径应显式
  固定 `http.followRedirects`，使"跟随重定向"不是一个依赖版本默认值的隐式行为。
- `http.proxy`、`http.<url>.extraHeader`、`http.<url>.sslCAInfo` 等键必须收窄，使恶意代理无法
  从凭据通道取得本不属于它的材料。
- **匹配粒度必须显式固定**：`credential.useHttpPath` 决定 Git 把 path 纳入匹配键，从而决定一个
  host 上的凭据能否被另一个 path 复用。凭据路径必须显式设定它，不能依赖默认值。
- `GIT_CONFIG_NOSYSTEM=1` 与 `GIT_CONFIG_GLOBAL` 继续固定配置来源；凭据路径继续保持无 `HOME`
  （受管 Git 进程的基础环境必须为空，`managed.rs:116`）。

### D7：Agent 与 Broker 的隔离靠 OS 身份，不靠一个 Agent 也能读到的令牌

Broker 端点是凭据执行身份 C 拥有的 `0600` Unix socket，按对端 UID 校验（复用
`crates/utils/src/local_ipc.rs:12-47` 与 `process-client/src/transport.rs:33-37` 的既有机制）。
W 的连接由内核拒绝。**不向 Agent 分发** Broker 地址加令牌式的调用凭据：任何 Agent 可读的 bearer
令牌都不是隔离，只是一个 Agent 也能偷走的秘密。

同理，凭据执行器不得复用 Agent 可达的通道（Agent 的 `--allow-read`／`--allow-run`／`--allow-env`
覆盖整个文件系统与进程环境）。

**隔离的判据是"能力"而不是"连通性"**：Agent 与 Node 之间的 stdio 插件通道必须保持可用（Agent 的
工作输入、结果与宿主能力调用都走那里），要挡住的是 W 取得凭据或提升身份，而不是 W 发起连接这件事
本身。四类通道的分类与各自的处理见 D16。

**C 的服务身份材料对 W 同样不可读**：`broker-resolver` 的客户端证书与私钥（D15）由 M 准备、只对 C
可读（`0600`，位于 W 不可遍历的目录），W 既读不到它，也不能用它去连接 Store。因此 W 的两条腿都是断的：
**没有授权**（授权不落 W 可达的介质），也**没有服务身份**（读不到 C 的证书）。

### D8：V1 不引入 sidecar 容器或外部 Git 服务

本 ADR 选择在 Node 容器内建立身份边界，而不是引入独立的 Git 执行容器或集群级 Git 服务。理由：

1. Sandbox Server 的模型是**一个 Workspace 一个容器**，并显式拒绝同一 Workspace 卷上的第二个容器
   （`effects.go:48-56`：两个容器会让两个 Node 对同一本账）。
2. `20260926` 不变量 7 明确"Sandbox Server 不执行 Git 操作，不注入任何凭据"。sidecar 会把 Git 与
   凭据注入带回 sandbox-server 的职责范围。
3. Workspace 卷是沙盒宿主机上的 Docker 命名卷（`effects.go:20,219`）。外部服务要写入它就必须取得
   宿主机卷访问权，等于把 Node 对 host 交互的所有权让出去，直接冲击已实现的收尾屏障与
   `Interrupted` 判定（`20260925` 两份 ADR）。
4. 已有实现把 clone 的 host 交互、`process_terminations`、只读核实都建立在"Node 拥有这次 host 观测"
   之上；D8 的选择保持这些不变量不动。

### D9：能力不足时拒绝，而不是降级

- 身份边界、Broker、Secret Store 或凭据执行器任一不可用或不可核实，clone 以明确失败结束；
  `repository_credential_executor_unavailable` 保持 fail-closed，不降级为匿名或静态个人凭据。
- 启动闸门：当部署要求执行凭据 clone 却**无法**建立三个互不相同的身份时，Node 必须拒绝该配置
  并拒绝受理凭据 clone，而不是在 Agent 的身份下执行。这与 `main.rs:36-46` 的现有拒绝条件同构，
  并应作为同一条启动校验实现。**"拒绝启动"与"只禁用凭据执行"的分界由 D17 的配置缺失表给出**：
  未声明要执行凭据 clone 的部署照常启动，只有"声明了却无法在正确身份下执行"才拒绝启动。
- 身份边界已建立但**未通过验收**（D14-B 的义务未全部满足）时，凭据 clone 同样必须被拒绝；"配置看起来
  正确"不构成放行条件。

### D10：受信任管理进程 M 的职责与容器特权风险

**M 是受信任管理身份，当前以 root 运行**（`node-entrypoint.sh:26-27`、`managed.rs:116`），因此它是
本方案的**可信计算基**。这不是过渡形态或待整改项，而是"降权执行 + 所有权交接"的前提：只有 root 才能
`chown`、准备挂载点并调用 `setpriv`。与之配套的约束是：**本 ADR 不得宣称 UID 隔离能抵御 M 被攻破**
（D1、不变量 12）。本 ADR 明确 M 的职责边界：

**M 必须做的事**：

- 在容器启动时准备并固定受保护路径：`/var/lib/ora`（`0711` root）、`repository_root`（`0755` root）、
  Agent 的 `workload_directory`（`0711` root，使 W 能按名字到达自己的会话目录而不能列举他人的），以及
  D11 规定的凭据物化目录（M 创建目录与 C 的服务身份，C 拥有叶子）。
- 持有并把 mTLS 证书写入 `/run/ora`（`0600` root），使控制通道不依赖 W 或 C 的权限。
- 在需要时降权：`setpriv` 的启动闸门（`managed.rs:113-127`）保证只有 root 能建立隔离工作负载。
- 为 C 准备并保护它的服务身份与位置：`broker-resolver` 的客户端证书与到 Store 的网络路径（D15）、
  物化目录及其属主（D11），以及把经控制通道到达的**授权载体**以文件形式交接给 C。M 只准备位置与
  身份，**不代替 C 解析**，也不读取解析结果。
- 在凭据路径上执行收尾：销毁物化材料、清理残留（D13）。

**M 不得做的事**：

- 不得把凭据材料写入自己的环境、命令行或日志；M 的环境与容器环境会被 W 继承（`session/host.rs:60-64`），
  因此**凭据材料绝不能经环境变量进入 Node 进程**。这条排除了"用 `-node-env-file` 或容器环境注入 Git
  凭据"的方案：该机制面向供 Agent 使用的模型凭据，用途相反。
- 不得把凭据材料保留在进程内存中超过一次 clone 所需；不得跨执行缓存。
- 不得把凭据材料交给 W 可达的通道（包括 Agent 的插件 IPC、session 目录与 checkout）。
- 不得兑换 Broker 授权，也不得持有 `broker-resolver` 服务身份（D15）。M 是控制通道的终点，因此必然
  经手**授权载体**（它本身也是 bearer 秘密，D3）；本 ADR 要求的是 M **不经手秘密本体**——不解析、
  不创建也不读取物化文件。把"任何秘密都不经过 M"写成承诺是虚假保证，本 ADR 不作此承诺。

**容器特权风险必须被承认**：M 是 root，且容器保留了 `DAC_OVERRIDE`、`SETUID`、`SETGID`、`CHOWN`、
`FOWNER`、`KILL`（`docker.go:175`）。这些能力是"降权执行 + 所有权交接"所必需的，但它们同时意味着
**容器内的任意 root 代码可以读取 C 的文件并杀死 C 的进程**。因此：

- 本 ADR 的隔离承诺只对 W 成立（D1）；
- 若未来要求"凭据对容器内任何进程不可读"，必须先交付更强的纳管（namespace／userns／seccomp）或
  把凭据执行移出容器，两者都超出 V1（D8、未解决问题）；
- 根文件系统当前可写、容器未设 `User`，属于可收窄项，但不改变上面的结论。

### D11：Git 子进程获取认证材料的具体通道与文件权限

**裁决：C 自己解析、自己物化。凭据材料只以"由 C 创建、C 拥有、W 不可遍历"的临时文件形式存在，不由
环境、argv 或跨 exec 的文件描述符传递；M 只准备位置与身份，不读取也不写入秘密内容。**

- 位置：一个由 M 创建的专用目录，位于 W 不可遍历的父目录下（例如与 `/run/ora` 同级的独立 tmpfs 挂载，
  或 Workspace 卷上 root 拥有的 `0711` 目录下的一个 C 拥有的 `0700` 叶子）。目录必须是
  `noexec`、`nosuid`，且不在 checkout 树内。
- 身份准备：M 创建该目录并把叶子 `chown` 给 C，同时准备 C 的 `broker-resolver` 客户端证书与到 Store
  的网络路径（D15）。**M 不代 C 解析**：授权载体经控制通道到达 M 之后，由 M 以文件形式放进该目录并
  `chown` 给 C，M 自己不出示它、不兑换它。
- 物化：物化文件由 **C** 在它自己的 `0700` 叶子里创建，模式 `0600`；解析结果（Git Token 明文）由 C
  写入并只由 C 读取。W 的 UID 与之不同，因此内核拒绝其打开；M 作为 root 在能力上仍可读取（D1 的边界
  声明），但**设计上不进入该路径**——它既不创建、也不读取该文件。
- 交付方式：Git 通过部署固定的、root 拥有且不可写的 helper 程序读取该文件（`GIT_ASKPASS` 或
  `credential.helper` 指向白名单内的固定路径），helper 本身不接收任何业务输入。**不使用**
  `http.<url>.extraHeader`，因为它会把秘密写进 Git 的配置求值路径与错误输出。
- 为什么不用文件描述符传递：受管路径在 exec 前执行 `close_range(3, MAX, CLOSE_RANGE_CLOEXEC)`
  （`crates/utils/src/process/linux_detached.rs:14-19`），跨 exec 的 fd 会被关闭；而 `RunSpec`
  （`crates/process-protocol/src/run.rs:34-47`）也没有 fd 传递字段。要支持 fd 传递就必须扩展
  host／guardian 协议，代价大于收益。承认其代价：文件路径多了一次磁盘物化，因此 D13 的销毁责任
  更重。
- **不使用环境变量**：受管 Git 进程的基础环境必须为空（`managed.rs:116`），且环境会被 Agent 继承的
  路径（`session/host.rs:60-64`）放大风险。
- **不使用 argv**：`git` 的 argv 会进入进程列表与错误输出。

### D12：checkout 所有权交接

clone 由 C 执行，但 checkout 最终要交给 W 使用（Agent 的会话工作目录就是它，`session/driver.rs` 的
`CheckoutDirectory`）。今天两者是同一个 UID，因此 `drive.rs:261-263` 的 `chown` 与
`session/workload.rs:64-93` 的 `own_tree_no_follow(checkout, uid, gid)` 都不产生交接语义。W ≠ C 之后
必须显式定义交接：

- clone **必须**在 C 拥有、W 不可写的目标上进行；不得让 C 直接写入 W 已拥有或 W 可写的目录，否则
  W 可以预置符号链接、`.git/config` 或 hooks 来影响 C 的行为。
- 交接发生在 clone 完成**之后**、Agent 会话开始**之前**，由 M 执行，使用既有的不跟随符号链接的
  所有权变更原语（`ora_utils::fs::own_tree_no_follow`）。
- 交接过程中**不得**出现"凭据材料对 W 可读"的窗口：凭据物化（D11）必须在交接之前销毁，而不是与
  交接并发。
- 交接之后 checkout 归 W，C 不再需要写它；后续的凭据化 Git 操作（若存在）必须重新走"C 拥有目标"
  的前提，不得在 W 拥有的树上进行。
- 交接失败（例如目录中出现不可变更的节点）必须以明确失败结束，不得跳过交接而让 Agent 以 W 身份
  进入一个仍由 C 拥有的树。

### D13：崩溃、超时与重试之后的秘密清理

- 正常路径：一次 clone 结束后（成功、失败或 `Interrupted`），M 在收尾屏障内删除物化文件。
  这与 `20260925` 的收尾屏障同一顺序，不新增第二个收尾阶段。
- 异常路径：Node 崩溃或容器被杀时无法执行收尾。因此必须有一个**启动清扫**：容器启动时（M 还是 root，
  在任何凭据 clone 被受理之前）清空凭据物化目录中的全部残留，并把"清扫完成"作为受理凭据 clone 的
  前置条件。
- 重试：重试是新执行、新授权（D5），因此也必须重新物化；不得复用上一次的残留文件。
- 超时：命令超时与授权到期都按失败处理并触发销毁；销毁失败不得使执行被记为成功。
- 销毁必须是"删除 + 目录可验证为空"，而不是"尽力而为"；`Interrupted` 判定与收尾证据沿用既有语义，
  本 ADR 只增加"凭据物化已销毁"这一条证据要求。
- **文件系统残留之外，还要考虑进程内存**：C 的进程被杀死后其内存由内核回收，这是可接受的；但 M 不得
  把凭据材料复制到可能被 core dump 的位置，容器也不应为 C 的进程开启 core dump。

### D14：开发准入与功能启用是两套门禁，验收义务只属于后者

与 [用户自助 HTTPS Git 凭据](../../cloud/identity-access/20261010-user-managed-git-credentials.md) D11
一致，本 ADR 把门禁拆成两套，两者**互不替代**：准入完成不解除任何 fail-closed，验收证据也不能在
准入之前产生。

**A. 开发准入**——进入实现所需，全部是规范与契约条件，**不依赖任何尚未实现的端到端测试结果**。
六项的规范工作已经做完，但结论要随相关 ADR 通过 specs 评审一并生效：

1. 相关 ADR 完成 specs 评审并标记 `approved`：本 ADR、Cloud 侧三份凭据决策与 `0023` 补录。方向已由
   `ora-space/circus#29` 确认，评审未走完，**六份现均为 `proposed`。待评审**。
2. Cloud Token Ingress 的仓库级规范例外生效（`cloud/AGENTS.md`）。**文本已随 cloud#64 落地**：英文例外
   段落已作为原条目续段追加，原句与测试密钥、日志脱敏两句逐字保留；承载它的 ADR 仍是 `proposed`，
   去留取决于 `ora-space/circus#135`。**规则尚未定案**。
3. Node D2 amendment 生效（最小 clone 执行契约的凭据来源修订）。方向已确认，但该 amendment 仍是
   `proposed`；该链阅读入口的注记按评审意见撤回，改由 `20260918` 标题下的 `> 历史决策：` 指向它。
   **待评审**。
4. Secret Store 的后端与密钥责任确定
   （[Secret Store ADR](../../cloud/identity-access/20261010-user-managed-git-secret-store.md) D1／D2／D8）。
   **已裁决，随该 ADR 的评审一并生效**：后端可替换是部署属性，具体选型仍是实现期决定，不阻塞准入。
5. M／W／C 信任边界确定：D1、D15 与 Cloud ADR D6 已按 `ora-space/circus#29` 的授权确认方向——三者互不
   相同、`broker-resolver` 由 C 持有而非 M、M 不解析秘密。**已裁决，随相关 ADR 的评审一并生效**。
6. API、数据兼容与 Broker 授权契约确定：授权绑定与**兑换时的原子消费**（D3、D5 与 Secret Store D10）、
   凭据审计列与权威绑定列的裁决（Cloud ADR D7／D8／D10）。**已裁决，随相关 ADR 的评审一并生效**。

**B. 功能启用**——解除 `repository_credential_executor_unavailable`：下节验收义务**全部**有可直接指向
的证据，且与 Cloud ADR D11-B 的每一项**同时**满足。**不得仅因为本文档存在、或测试文件存在，就认为
隔离已经成立。**

### D15：`broker-resolver` 服务身份由凭据执行身份 C 持有，M 只准备身份与位置

**裁决：解析用的 Store 服务身份 `broker-resolver`
（[Secret Store ADR](../../cloud/identity-access/20261010-user-managed-git-secret-store.md) D3）由 C
持有，不由 M 持有。**

- 解析客户端以 C 的身份运行，持有一套只对 C 可读的客户端证书与到 Store 的网络路径。它向 Store 出示
  服务身份与 Cloud 签发的授权，取回秘密并就地使用，**整个过程中秘密不经过 M**。
- M 的职责因此被限定为"准备与清理"：创建并保护 C 的身份材料与物化目录、把经控制通道到达的**授权
  载体**交接给 C（D11）、在收尾与启动清扫时删除残留（D13）。M **不解析**，也不读取解析结果。
- 如实记录边界：M 是控制通道的终点，因此**授权载体**（本身也是 bearer 秘密，D3）必然经过 M 的进程。
  本裁决消除的是"**Token 明文经过 M**"，不是"任何秘密都不经过 M"。把后者写成承诺会是虚假保证。

选择理由与代价：

- 若由 M 持有，Token 明文必然先进入作为可信计算基的 root 进程，再由它按 D11 物化给 C。这会把本就
  脆弱的性质进一步削弱：M 的进程内存、core dump 与调试路径都会成为明文经过点，而 M 是 root，任何
  M 侧缺陷都会同时暴露全部凭据。
- 代价是 C 需要自己的出网路径与证书，不能复用 M 的控制通道。这是部署必须承担的成本，也是本裁决对
  "最小改动"的明确取舍：宁可多一套 C 的服务身份，也不让明文路径穿过 root。
- 这与 W ≠ C 的边界互相印证：W 既没有授权，也读不到 C 的证书，两条腿都是断的（D7）。

不变量 13 与验收义务 2 覆盖本条。

### D16：受保护的是"能力"，不是 socket；Agent 的既有 IPC 保持可用

**裁决：本 ADR 在 IPC 上的承诺是"W 不能经 IPC 取得凭据或提升身份"，而不是"W 不能连接任何
host／guardian socket"。**后者既不是当前的实现事实，也不是有意义的判据。

现状（受控 Node，`workload_uid` 已配置、Agent 处于 `Separate` 放置）：

| 类别 | 具体通道 | 今天的客户端 | 本 ADR 的处理 |
|---|---|---|---|
| **A. Agent 合法业务 IPC** | Agent 与 Node 之间的 **stdio JSON-RPC 插件通道**（4 字节长度前缀 + 类型字节 + JSON）：`ora/childprocess/*`、`ora/storage/*`、`ora/register`、`ora/shutdown`、`agent/*`、`effect/*` | Agent（W）↔ Node（M） | **必须保持可用**。它不是 socket，而是一对由 Node 持有的管道；Agent 的工作输入、结果与宿主能力调用全部走这里 |
| **B. 受保护的管理操作** | host socket（`host.sock`／`host-io.sock`，`Inspect`／`CreateScope`／`Start`／`QueryRun`／`Stop`／`Close`／`QueryScope`／`Output`）与 guardian socket（`control.sock`／`events.sock`／`io.sock`，`Bind`／`Inspect`／`Start`／`Query`／`Stop`／`Close`／`Scope`／`Output`） | **只有 M 的进程**：`apps/ora-node` 与 `crates/process-runtime/src/host*` | W 不得直接调用；Node 也不得把 Agent 的请求**原样转发**成超出今天 `ora/childprocess` 语义的宿主操作 |
| **C. Credential Broker／Resolver 通道** | C 拥有的 `0600` Unix socket 与 C 的 `broker-resolver` 服务身份（D7、D15），以及到 Secret Store 的网络路径 | 只有 C | W 不得连接、不得持有其材料（D7） |
| **D. 可导致身份提升的接口** | 控制通道 mTLS 证书（`/run/ora`，`0600` root）、任何"以指定 UID 启动进程"或 `chown` 的能力、`setpriv` 等价入口 | 只有 M | 不得对 W 开放；不存在"以 W 身份调用管理面"的路径 |

依据与边界：

- A 类的存在意味着**"Agent 能请求宿主做事"是设计的一部分**：`ora/childprocess/*` 让 Agent 请 Node
  代它 spawn／写 stdin／kill 子进程，这是 Agent 今天运行 agent CLI 的正常方式，因此"Agent 无法与宿主
  通信"这个说法与事实相反。本 ADR 要保证的是这些请求**仍然以 W 的身份落地**（`TokioProcessSpawner`
  用同一个 `LinuxChildIdentity` 服务 Deno 进程与它请求的子进程），而不是变成 M 或 C 的能力。
- B／C／D 三类今天**已经**由内核对端 UID 校验保护：host／guardian／helper socket 以 `0600` 建立并在
  每条连接上按 `peer_cred().uid() == owner` 拒绝其他身份（`host/server.rs:73`、`guardian.rs:136`、
  `process-client/src/transport.rs:35`）。**这条边界不依赖路径保密**，因此无需新增机制即可原样复用。
- **如实记录**：Agent 今天持有 `--allow-read`（整个文件系统）、`--allow-run`、`--allow-env`、
  `--allow-net`，并继承 Node 的环境（`crates/plugin-lifecycle/src/permissions.rs:106-113`、
  `crates/plugin-runtime/src/lib.rs:155-157`）。因此它**可以尝试**连接 B／C 类的 socket 或读取其路径，
  被拒绝的原因是内核的对端 UID 与文件模式，不是它不知道路径。验收必须按"连接被拒"而非"路径不可见"
  来构造证据。
- **不得**把本条的判据写成"Agent 无法连接 host／guardian／管理 socket"：这个断言在当前的
  `--allow-read`／`--allow-net` 授权下无法由沙箱保证，只能由对端校验在"是否获得能力"这一层保证。

因此 D14-B 的义务 16、17 按"能力"而非"连通性"验证：以 W 身份发起 B／C／D 类连接必须被拒；
A 类通道与 Agent 的既有工作流必须照常工作。

### D17：首次实现只建立身份与启动检查，不改变无凭据 Git 的可观察行为

**裁决：受管 Git 按"本次执行是否需要凭据"分流身份，而不是把所有 Git 都迁到 C。**

| 执行 | 执行身份 | 何时生效 |
|---|---|---|
| **无凭据**的受管 Git（公开仓库匿名 Clone，以及今天所有未绑定凭据的 Clone） | **继续使用既有的受管执行身份**（今天的 `process.workload_uid`） | 长期契约，不是过渡态 |
| **带凭据**的受管 Git（未来的受控凭据 Clone） | **C** | P1-M1 **不**启用 |

- **为什么无凭据路径不迁到 C**：它不接触任何凭据。迁移只会扩大 C 的攻击面与故障半径，并让"公开仓库
  匿名 Clone 行为不变"多出可观察差异——checkout 目录今天按工作负载身份 `chown`（`repository/drive.rs:261-263`），
  换身份就要多一次交接。因此本 ADR 的"Agent 不得与凭据执行同身份"**只针对 C**：无凭据的受管 Git 与 W
  同 UID 是刻意保留的，它不承载秘密，因此不构成 D7／D9 所禁止的那条路径。
- **P1-M1 的范围**：建立 C 这个身份、按 D11 准备它的目录与权限、接上 D9 的启动闸门与 D13 的启动清扫。
  **不接入 Broker／Resolver，不物化任何凭据，不解除 `repository_credential_executor_unavailable`。**
  P1-M1 结束时，凭据 Clone 仍被拒绝——它建立的是"能力成立"的前提，不是能力本身。

**C 身份与目录的准备（由 M 在任何 clone 被受理之前完成）**：

- 固定 C 的 OS 身份：UID/GID 由部署给出，**非 0**，且与 W、M 都不同；三者中任意两个相等即按 D9 拒绝。
- 创建凭据物化目录：父目录 root 拥有 `0711`（C 能按名字到达，W 不能列举），叶子 `0700` 且 `chown`
  给 C；挂载为 `noexec`、`nosuid`，且**不在 checkout 树内**（D11）。
- 准备并保护 C 的服务身份材料（`broker-resolver` 证书／私钥，`0600`，只对 C 可读，D15）。P1-M1 不
  使用它，但要先建立"W 读不到"这一事实。

**如何在不使用真实 Token 的情况下验收身份隔离**：隔离证据只依赖"W 能否读到／冒用"，与凭据内容无关。
因此义务 1、2、3、10、16、17、18 与启动闸门都可以用**占位材料**完成：

- 在 C 的物化目录里放一个自造的占位文件（模式与所有权同 D11 一致），验证 W 打不开、W 列举不到父目录；
- 以 W 的身份尝试连接 C 的 socket 与管理 socket，验证被内核按对端 UID 拒绝；
- 以配置断言验收启动闸门：三者取值相同、C 为 0、非 root 无法降权——各自单独构造一次并确认拒绝。
- **不需要**真实 Token、不需要真实私有仓库。义务 12（真实私有仓库 Clone 成功）**必须**用真实凭据，
  属 D14-B 第 8 项，明确**不在** P1-M1 范围内。

**如何避免破坏既有生命周期**：

- 无凭据路径**完全不动**：clone 对外协议、目录分配、收尾屏障、`Interrupted` 判定、
  `process_terminations` 与只读核实保持原样。
- C 的目录位于 checkout 树之外，不改变 Workspace 卷的既有所有权与 `0711` 语义。
- 启动清扫只清 C 的物化目录，不触碰 Agent 会话目录，也不改变 Revision／Workspace 的既有清扫顺序。
- Agent／M 的身份字段若需要拆分，必须保持"未配置时与今天逐位一致"的行为。

**配置缺失的两种处置（这是 D9 启动闸门的操作化）**：

| 情形 | 结果 |
|---|---|
| 部署**没有声明**要执行凭据 Clone：未配置 C 的身份、Broker／Resolver 端点、C 的服务身份材料或物化目录 | **只禁止凭据执行**。Node 正常启动，公开仓库匿名 Clone、Agent 会话、Revision 与 Workspace 生命周期照常；凭据 Clone 以 `repository_credential_executor_unavailable` 拒绝 |
| 部署**声明**了要执行凭据 Clone，但身份边界无法建立：C 与 W 或 M 取值相同、C 为 0、不是 root 因而无法降权、受保护目录无法创建或无法固定权限、C 的服务身份材料不可读 | **整个 Node 启动失败**，不降级运行。这与 `main.rs:36-46` 的既有拒绝条件同构：配置要求了凭据执行却无法在正确身份下进行，那么任何一次 clone 都会在错误身份下执行 |

- 与 D9 一致：身份边界已建立但**未通过 D14-B 验收**时，Node 可以启动，但凭据 Clone 必须继续被拒绝。
- **一个必须显式判断的既有配置状态**：未设置 `workload_uid` 时 Agent 走 `SessionWorkload::Shared`
  （`apps/ora-node/src/service/agents.rs:81-100`），即 Agent 以 Node 自身的身份运行，此时 **W 与 M 是
  同一个身份**。这不是新增风险，而是今天在没有 `workload_uid` 的部署上的既有行为，因此：
  - 部署**没有**声明执行凭据 Clone → 照常启动，只禁止凭据 Clone（第一行规则）；
  - 部署**声明**了执行凭据 Clone → 必须按第二行规则**拒绝启动**，因为把凭据交给一个以管理身份运行的
    Agent 正是本 ADR 要消除的路径。
- **不得提前开放私有仓库 Clone**：P1-M1 不改变 `repository_credential_executor_unavailable` 的行为，
  D14-B 未全部满足前它一直是"拒绝"。

## 不变量

1. 管理身份 M、Agent 工作负载身份 W、凭据执行身份 C 三者互不相同；W 与 C 非特权且非 root，M 是
   受信任管理身份并当前以 root 运行。W 既不能直接取得、也不能经管理接口提升到 C 或 M 的权限；
   C 同样不拥有 M 的管理权限——它不 `chown`、不 `setpriv`、不读 `/run/ora` 的 mTLS 材料、不调用
   管理面，也不持有 `ingress-writer`／`kek-operator` 服务身份。
2. 凭据材料只对 C 开放；W 读不到其文件、环境、文件描述符与进程内存。M 作为 root 在能力上仍可读取
   （不变量 11、12），但**设计上不进入该路径**：它不解析、不创建也不读取物化文件（D15）。
3. 业务协议、Node 数据库、process journal、日志与结果中不出现凭据材料或其可复原形式；Broker 授权
   载体作为 bearer 凭据同样不落持久介质。
4. 每次 clone 使用一次独立授权，绑定 tenant／owner／project／workspace／clone execution／
   credential ID／credential version／repository identity／`operation=clone`，并带到期、单次使用与
   Node 实例／运行代次约束。
5. 授权到期、凭据冻结、成员停用或运行代次被 fencing 之后，该授权不可再用，且不再签发新授权。
6. 凭据执行只使用部署固定的程序与配置；业务请求不能指定 helper、代理或任意可执行文件；checkout 内的
   `.git/config` 也不能改变与凭据相关的键。
7. 凭据执行失败、超时、崩溃或 Node 停止后，不留下 W 可读的秘密残留；容器启动清扫完成是受理凭据
   clone 的前置条件。
8. 仓库范围只由授权中的 repository identity 决定；重定向、代理、helper 与匹配粒度的配置不能扩大该范围。
9. 凭据材料不经环境变量、argv 或跨 exec 的文件描述符传递；物化位置必须在 W 不可遍历的目录内。
10. checkout 的所有权交接发生在凭据物化销毁之后、Agent 会话开始之前，且交接期间不出现"材料对 W
    可读"的窗口。
11. UID 边界只对 W 成立，不构成对容器内 root、内核逃逸或网络外发的保证；任何文档、注释与提交说明
    都不得把它表述为绝对隔离，也不得表述为"M／W／C 三者互相完全不可读"。本 ADR 承诺的只有 D1 表中
    的四条：W 读不到 C 的秘密、W 无法冒用 C 或 M、C 不拥有 M 的管理权限、M 属于可信计算基。
12. M 是受信任管理身份并当前以 root 运行；它是本方案的信任根，UID 隔离**不**承诺抵御 M 被攻破，
    任何文档、注释与提交说明都不得把它表述为"三个身份都是非 root"或"M 被攻破后凭据仍安全"。
13. `broker-resolver` 服务身份由 C 持有，M 不持有；C 的服务身份材料只对 C 可读，W 既不能读取它，
    也不能用它连接 Store。M 只准备该身份与物化位置，不解析、不读取解析结果（D15）。

## 为什么不是这些替代方案

| 方案 | 取舍 |
|---|---|
| 只配置一个不同的 `workload_uid`，继续让 Agent 与 Git 同身份 | 复用现有开关，但 Agent 是同一个身份，等于没有边界；本 ADR 的核心否决对象。注意现状**已经**配了 `workload_uid`，因此"配了就安全"的说法尤其危险 |
| 把 Token 注入 Agent 环境或 checkout 供其自行 `git pull` | 最省事，且让 Agent 直接拿到长期 PAT；违反产品方向"不向 Agent 提供长期 PAT"，并会被 `session/host.rs:60-64` 的环境继承放大 |
| 继续用部署静态凭据（个人 PAT／SSH 身份） | 满足单租户自托管，但无法按用户归属、无法冻结、无法单次授权，且与"用户在 Web 管理 Token"冲突 |
| 只靠 `-c` 约束与 `core.hooksPath=/dev/null` | 约束的是 Git 的行为，不是 Agent 的读取能力；`20260918` D4 已说明这不构成执行隔离 |
| 声明 cgroup／namespace／seccomp 作为边界 | 这些机制在当前交付中不存在（`LinuxBestEffort` 自报 `BestEffortOnly`，只做 `setsid` + `close_range`），把未交付的能力当边界是虚假保证 |
| 把凭据物化目录放进 `/run/ora` | 复用现有 root `0700` tmpfs，但 C（非 root）既不能遍历也不能在其中创建文件；必须另建一个对 C 开放的目录（D11） |
| 经环境变量或 `-node-env-file` 交付 Git 凭据 | 复用既有机制，但该机制面向**供 Agent 使用**的模型凭据，且 Agent 继承 Node 环境；用途相反，方向错误 |
| 经跨 exec 的文件描述符交付 | 避免磁盘物化，但受管路径在 exec 前执行 `close_range(..., CLOEXEC)`，且 `RunSpec` 没有 fd 字段；需要扩展 host／guardian 协议，代价大于收益 |
| sidecar 容器（方案 B） | 隔离更强、Token 不进入 Agent 容器；但要求 amendment Sandbox Server 的"一个 Workspace 一个容器"与"不注入凭据"两条不变量，并新增跨容器交付通道，代价与风险超出 V1 |
| 外部可信 Git 服务（方案 C） | 凭据收敛最彻底；但需要跨主机写入 Workspace 卷，交出 Node 对 host 观测的所有权，直接冲击已实现的收尾屏障 |
| 让 Node 以非 root 运行、靠文件权限隔离 | 少一个特权进程，但入口脚本需要 root 做 `chown`／`chmod` 与挂载点准备，且 `managed.rs:116` 的降权闸门本身要求 root；先降权会同时移除 D1 需要的降权能力 |
| 声称 `W ≠ C` 已经提供强隔离 | 最省事，但违反 D1 与不变量 11：容器内的 root（M）仍可读 C 的文件，内核逃逸与同 UID `/proc` 风险依然存在 |
| 让管理身份 M 持有 `broker-resolver` 服务身份，由 M 解析后按 D11 物化给 C | 复用 M 已有的控制通道与证书，不必为 C 新造出网路径与服务身份；但 Token 明文必然先经过作为可信计算基的 root 进程——它的进程内存、core dump 与调试路径都成为明文经过点，而 M 是 root，任何 M 侧缺陷都会同时暴露全部凭据（D15）。改成"多一套 C 的服务身份"是明确取舍 |

## 失败与恢复语义

| 情形 | 行为 |
|---|---|
| Broker／Secret Store 不可用 | 明确失败；不匿名、不静态凭据；不留下部分物化 |
| 授权到期 | 本次 clone 失败；重试申请新授权 |
| 授权已被消费但响应丢失／超时 | 本次 clone 失败；**不得**重发同一授权，必须申请新授权（D5、Secret Store D10） |
| C 的 `broker-resolver` 服务身份缺失或不可用 | 凭据 clone 明确失败；不由 M 代解析（D15）；配置要求凭据能力而身份无法建立时按 D9 启动拒绝 |
| 凭据在 clone 前被冻结／成员停用 | 拒绝签发新授权；已签发未使用的授权随到期失效 |
| Git 进程被终止或 Node 停止 | 沿用 `Interrupted` 与收尾屏障；凭据物化在收尾时销毁（D13） |
| Node 崩溃或容器被杀 | 启动清扫在受理任何凭据 clone 之前清空物化目录；清扫未完成前拒绝受理 |
| 凭据执行器崩溃 | 不自动以 W 身份重跑；记录为未核实，按现有 host 观测规则结算 |
| 身份边界无法建立 | 启动即拒绝该配置（D9），不受理凭据 clone |
| 身份边界已建立但验收未通过 | 仍拒绝凭据 clone（D9、D14-B） |
| 重定向到授权范围外的 host | 凭据执行器不返回材料；clone 以认证失败结束 |
| checkout 所有权交接失败 | 明确失败；不得让 Agent 以 W 身份进入仍由 C 拥有的树（D12） |
| 物化文件删除失败 | 执行不得被记为成功；残留进入启动清扫（D13） |

## 验收义务

以下义务在 `approved` 后落到 `specs/test-cases/node/repository/`（本 ADR 不新建核心用例文件）。
本地受控 Git 服务器即可覆盖其中大部分；仅"真实私有仓库成功 clone"必须使用真实 GitHub／GitLab
集成证据，跨 host 重定向的**真实语义**也建议以真实主机复核一次。

1. Agent 不能读取 Git Token（文件、环境、文件描述符三条路径分别验证）。
2. Agent 不能调用 Credential Broker（对端 UID 拒绝），也不能读取 C 的服务身份材料（`broker-resolver`
   的客户端证书与私钥），因此既不能出示授权也不能出示服务身份。
3. Agent 不能读取 Git 子进程的秘密文件或环境。
4. Clone 只能访问授权的仓库。
5. HTTP 重定向不泄露 Token（不依赖 Git 文档，需实测）。
6. 恶意代理与 `http.extraHeader` 无法取得不应获得的认证材料。
7. Clone 重试不会复用旧授权。
8. Token 轮换后旧版本不能用于新 Clone。
9. 成员停用／凭据冻结后禁止新 Clone。
10. Git 进程崩溃不会留下秘密；Node 崩溃后启动清扫可证明目录为空。
11. 多租户与多 Workspace 之间凭据严格隔离。
12. 真实私有仓库 Clone 可以成功。
13. checkout 内的 `.git/config` 不能改变 `credential.helper`、`http.*` 或代理键的取值。
14. checkout 所有权交接之后，Agent 可正常读写工作目录，而凭据物化已不存在。
15. 身份边界不完整（例如三者中任意两个相同）时，Node 启动即拒绝并拒绝受理凭据 clone。
16. Agent 不能**经 IPC 提升身份或取得凭据**：以 W 的身份连接 Credential Broker／Resolver 通道、控制
    通道与受保护管理操作一律被拒（内核对端 UID 校验），控制通道证书对 W 不可读，且不存在任何"以 W
    身份调用管理面"的路径。**本项不要求、也不得表述为"Agent 无法连接任何 host／guardian socket"**：
    判据是"是否取得凭据或提升身份"，不是"socket 是否连通"（D16）。
17. Agent 的**既有合法功能不因本决策回归**：Agent 今天能够使用的业务 IPC、工作目录读写、命令执行与
    会话生命周期全部保持可用；D16 列出的四类 IPC 判据逐条验证，其中"合法业务 IPC"一类必须给出
    **仍然可用**的正向证据，而不只是负面拒绝证据。
18. C **不拥有 M 的管理权限**：以 C 的身份不能 `chown`／`setpriv`、不能读 `/run/ora` 的 mTLS 材料、
    不能调用管理面，也不持有 `ingress-writer`／`kek-operator` 服务身份（D1 表的第 3 条）。
19. **无凭据路径不回归**：公开仓库匿名 Clone 的对外行为与今天逐位一致——同样的协议、目录分配、收尾
    屏障与 `Interrupted` 判定，且其执行身份未被迁移到 C（D17）。

## 本决策未解决的问题

- **Agent 身份的落地形态**：新增一个 Agent 专用身份字段，还是把 Agent 也纳入受管执行路径？
  两者对 `workload_uid` 现有语义（单一值、施加于每个受管工作负载）的改动不同。
- **凭据物化目录的具体位置与创建方式**：新增一个 tmpfs 挂载、还是在 Workspace 卷上由 M 创建
  `0711` 父目录加 C 拥有的 `0700` 叶子。两者对"容器重启后残留是否自动消失"的答案不同（tmpfs 自动
  清空，卷不自动），直接影响 D13 的清扫必要性。
- **M 作为可信计算基的可接受度**：V1 承认容器内 root 能读 C 的文件。是否要求先交付更强的纳管
  （namespace／userns／seccomp）或只读根文件系统再启用凭据 clone，属于威胁模型裁决，超出本 ADR。
- **残余风险是否可接受**：同 UID `/proc` 读取与 `ptrace`、内核逃逸、网络外发都不在 UID 边界的作用
  范围内；是否要求额外缓解（例如固定 `ptrace_scope`、只读根、无 `CAP_KILL`）需要裁决。
- **sidecar 路线是否排期**：若未来需要"Token 不进入 Agent 容器"的更强保证，必须先 amendment
  Sandbox Server 的两条不变量，属于另一叶子的决策。
- **`Dockerfile:54-55` 的注释偏差**：该注释称入口脚本以非特权 `node` 用户运行 Node，与
  `node-entrypoint.sh:5-7,26-27` 的实际行为不一致，应在实现阶段一并修正。

## 落地顺序

**D14-A 开发准入的文本已就绪，但结论尚未生效**——六份 ADR 保持 `proposed`，等 specs 评审通过；
`cloud/AGENTS.md` 的例外文本已落地、D2 amendment 的方向已确认。**没有任何一步已经开始**。D14-B 的
验收义务只在对应步骤完成后才可能产生证据，不得倒过来用"还没跑端到端测试"阻止准入，也不得反过来因为
准入就绪就认为证据已经存在。

1. **身份边界与启动闸门（不接凭据，范围由 D17 限定）**：让 Agent 以非特权身份 W 运行，建立并固定
   凭据执行身份 C（非 0、与 W 和 M 都不同），按 D11 准备 C 的目录与权限；加入 D9／D14-B 的启动拒绝；
   加入 D13 的启动清扫。验收：W 打不开 C 的目录与占位材料、列举不到其父目录，W 无法以 C 或 M 的身份
   取得能力（D16 的四类 IPC 判据），M 拥有的 `0600` 文件对 W 不可读；**凭据 clone 仍被拒绝**。
   **这一步不改变任何 Git 执行路径的身份**：无凭据的受管 Git 继续使用既有身份，公开仓库匿名 Clone
   的可观察行为逐位不变（D17）。
2. **凭据执行器与 Broker 客户端（本地受控 Git 服务器）**：实现 D11 的物化与 helper、D15 的 C 侧
   `broker-resolver` 身份、单次授权的兑换时原子消费（Secret Store D10）、到期、冻结与 fencing 绑定，
   打通 D3／D5／D13。验收：义务 1、2、3、7、8、9、10。
3. **Git 侧约束收窄与所有权交接**：helper 白名单、URL 无凭据、关闭 validate-then-use 缺口、
   收窄仓库本地配置、重定向／代理／匹配粒度约束、D12 的交接（D2／D4／D6／D12）。验收：义务
   4、5、6、13、14。
4. **真实仓库集成**：以真实 GitHub／GitLab 私有仓库验证认证语义与重定向行为。验收：义务 12。
5. **解除 fail-closed 并同步 ADR**：在
   [Cloud 侧 D11-B](../../cloud/identity-access/20261010-user-managed-git-credentials.md) 与本 ADR D14-B
   的义务全部满足后，按最终实现更新本 ADR 状态、补核心用例。`20260918` D2 的 amendment 已由
   [最小 clone 执行契约 D2 的凭据来源修订](20261010-minimal-clone-execution-contract-credential-source-amendment.md)
   承载，**方向已由 `ora-space/circus#29` 确认，但仍是 `proposed`**（该链的阅读入口以
   `> 历史决策：` 指向它）；本步剩下的是在 D11-B／D14-B 满足后解除 fail-closed。`node/process/control/0-…` D2 的 amendment 仍待起草
   （minicloud 相关决策已随其退役成为历史契约，不在此列）。验收：义务 11、15、16。

每一步都保持已实现的行为不变：clone 的对外协议、目录分配、收尾屏障、`Interrupted` 判定、
以及"能力不足时拒绝"的 fail-closed 行为。第 1 步之后开始改变的只有**身份模型本身**——系统里出现
第三个互不相同的身份 C 并开始强制三者的互异性；**没有任何 Git 执行路径因此改变身份**，无凭据的
受管 Git（含公开仓库匿名 Clone）与 Agent 会话的行为保持逐位不变（D17）。
