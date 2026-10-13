# 受控 Node 的仓库凭据执行身份隔离

依据[受控 Node 的仓库凭据执行使用独立 OS 身份](../../../decisions/node/repository/20261010-controlled-node-git-authentication-isolation.md)
（`approved`）与其下游契约修订
[最小 clone 执行契约 D2 的凭据来源修订](../../../decisions/node/repository/20261010-minimal-clone-execution-contract-credential-source-amendment.md)
（`approved`）。本文件登记 Node 侧需要长期守住的承诺：M／W／C 三个身份互不相同且不可互相冒用、
Agent 的既有合法功能保持可用而**能力**不可提升、匿名公开仓库 Clone 的可观察行为逐位不变、
凭据材料在执行结束后必被销毁，以及"缺少凭据配置只禁止凭据执行"这一启动语义。

**当前实现状态**：受控 Node 今天仍有 M（root 管理进程）与 W（`workload_uid`，沙箱内为 1000），
**Node 内没有** C。C 已按
[凭据执行器归属当前正式产品](../../../decisions/node/repository/20261012-credential-executor-belongs-to-the-current-product.md)
（`approved`）实现为**当前正式产品的独立部署单元**：独立镜像目标、独立非 root 进程身份（uid 10002）、
独立卷集与独立信任根，不是 Node 内的第三个进程。因此本文件各项义务的**形态**发生了变化：原来针对
"同一容器内三个 UID"的断言，今天落在容器边界与卷集合上；同一容器内的 M／W／C 三身份分离仍然**不存在**。

证据因此分两类：**已有直接证据**的是执行器一侧的边界（身份、物化根、helper 不可写性、单次授权、
销毁与清扫），以及真实私有仓库 Clone 全链路；**仍然 `Missing`** 的是"以 W 身份对 C 发起能力尝试"
这类需要同一文件系统内两个身份共存的断言。Cloud 侧的元数据与授权见
[凭据元数据](../../cloud/identity-access/repository-credential-metadata.md)，秘密承载见
[凭据 Secret Store](../../cloud/identity-access/repository-credential-secret-store.md)。

## Workload And Credential Identities Stay Distinct And Cannot Impersonate Each Other

### 风险

把"给不同工作负载传了不同的 UID 数值"当成隔离结论。真正要守住的是**能力边界**：W 读不到 C 的凭据、
授权载体、服务身份材料与物化文件；W 无法冒用 C 或 M；C 不拥有 M 的管理权限。若只断言数值不同，
那么 W 与 C 同 UID、或凭据文件对 W 可读时，测试仍然会通过。

### 前置状态

受控 Node 以 root 管理进程 M 运行，拥有受保护配置、Node 与 process-host journal；Agent 插件以 W 运行；
计划新增非 0、与 W 和 M 都不同的凭据执行身份 C，并按 ADR D11 准备 C 的目录与权限（含占位材料）；
M 拥有若干 `0600` 文件；容器以 `no-new-privileges:true` 与 `CapDrop ALL` 运行。

### 触发

以 W 的身份尝试：打开 C 的目录与占位材料、列举 C 的父目录、读取 M 的 `0600` 文件、以 C 或 M 的
身份取得能力（按 D16 的四类 IPC 判据逐一尝试）、读取另一个进程的环境与文件描述符、读取进程内存。
再以 C 的身份尝试执行 M 的管理操作。

### 必须成立

- 三个身份互不相同：`W ≠ C`、`W ≠ M`、`C ≠ M`，且这一互异性由运行配置强制，而不是由文档声明。
- **W 读不到 C 的凭据、授权载体、服务身份材料与物化文件**；W 列举不到 C 的父目录。
- **W 无法冒用 C 或 M**：不能以它们的身份取得能力，也不能经管理接口提升。
- **C 不拥有 M 的管理权限**：C 既不是管理身份的成员，也不能执行受保护的管理操作。
- M 属于**可信计算基**：UID 隔离不保证 M 被攻破后凭据仍然安全，因此不得把"M 也读不到"写成结论。
- 隔离断言的证据必须落在**文件、环境、文件描述符**三条路径上，而不是"配置了不同 UID"。

### 禁止结果

以"UID 数值不同"作为完整安全断言；把 W 与 C 配置为同一 UID 而仍然宣称隔离；让 C 持有 M 的管理能力
或让 C 成为 root；把凭据物化在 W 可读的路径下；用"M 也是同 UID 体系"为 W 的越权辩护。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| `W ≠ C`、`W ≠ M`、`C ≠ M` 由配置强制 | `desktop/apps/ora-node/src/managed.rs:29`（`workload_uid`）、`:38-56`（`isolate_workload` 经 `setpriv --reuid/--regid --clear-groups --no-new-privs`）与 `desktop/apps/ora-node/src/main.rs:39-46` 仍只覆盖 Node 内的 M／W。C 由部署固定为 uid 10002（`cloud/scripts/Dockerfile` 的 `executor` 目标），在运行栈上实测 `id` 为 `uid=10002(executor) gid=10002(executor)`，与 Node 镜像内 W 的 1000、Cloud 的 10001 三者数值互异；但三者今天分处不同容器，互异性来自部署，而不是来自同一文件系统内的配置强制 | Partial |
| W 读不到 C 的目录、占位材料与服务身份材料 | 执行器容器实测：`/var/lib/clone-executor` 及其 `material`／`checkout` 均为 `drwx------ 10002 10002`；C 的容器内**不存在** `/etc/ora-clone-signing`（Cloud 的签名私钥）与 `/etc/ora-clone-dispatcher`（Cloud 的派发身份），只挂载 `broker-resolver` 对（`0400` 私钥）与 Store 的 CA。这是"卷集合"路径的直接证据。**尚缺**以 W 身份对 C 的目录发起访问的否定断言——W 与 C 今天不在同一文件系统内，该断言无法在现有形态下成立 | Partial |
| W 无法冒用 C 或 M | 需要跨身份能力尝试测试；仅 P2 的 UID 断言不算证据。容器边界今天使 W 无法到达 C 的 IPC 面，但这是"不可达"，按本用例的判据**不构成**隔离证据 | Missing |
| C 不拥有 M 的管理权限 | C 以非 root 的 uid 10002 运行，其容器内不挂载任何管理材料（无管理 CA、无管理套接字、无 Cloud 配置）；实测 C 的容器只暴露执行器自己的监听端口。**尚缺**以 C 身份逐项尝试受保护管理操作的拒绝测试 | Partial |
| 文件、环境、文件描述符三条路径分别验证 | 文件路径已有上述卷集合与权限证据；**环境**与**文件描述符**两条路径均未建立 | Partial |

### 决策依据

Node 隔离 ADR D1（四条隔离承诺）、D9、D11、D14-B 第 2 项与不变量 1、2、4。

## Agent Interfaces Cannot Yield Credentials Or Identity Escalation

### 风险

用"Agent 无法连接 host／guardian socket"当作安全结论。这个结论既不成立也不必要：Agent 今天经
**stdio** 与 Node 通信，而且插件权限集包含 `--allow-read`（整个文件系统）、`--allow-run`、
`--allow-env`、`--allow-net`，所以"连不上"不是沙箱保证的性质。真正要守的是**能力**：Agent 的合法
业务 IPC 必须保持可用，同时不能经任何 IPC 取得 Git 凭据或提升权限。

### 前置状态

受控 Node 运行 Agent 会话；插件经 stdio 与 Node 通信；权限集为 `--allow-read`（全部）、`--allow-run`、
`--allow-env`、`--allow-net`；host 与 guardian 的流上存在 peer 凭据校验。

### 触发

按 D16 的四类接口分别尝试：

- **A 类**（Agent ↔ Node 的 stdio JSON-RPC 业务通道：`ora/childprocess/*`、`ora/storage/*`、
  `ora/register`、`ora/shutdown`、`agent/*`、`effect/*`）：正常驱动 Agent 的既有工作流；
- **B 类**（受保护的管理操作：host 的 `Inspect`／`CreateScope`／`Start`／`QueryRun`／`Stop`／
  `Close`／`QueryScope`／`Output`，guardian 的 `Bind`／`Inspect`／`Start`／`Query`／`Stop`／
  `Close`／`Scope`／`Output`）；
- **C 类**（Credential Broker 与 Resolver 通道）；
- **D 类**（可导致身份提升的接口）。

### 必须成立

- **A 类必须保持可用**：Agent 的既有合法功能不因本条而失效；这是与"阻止越权"同等强度的义务。
- B 类：Agent 无法以非管理身份执行受保护的管理操作；越权尝试被明确拒绝（既有 peer 凭据校验路径）。
- C 类：Agent 无法取得 Git Token、Broker 授权载体或 C 的 `broker-resolver` 服务身份材料。
- D 类：Agent 无法经任何接口提升身份；判据是"无法取得凭据或提升身份"，**不是**"无法连接 socket"。
- 结论必须按能力陈述，并记录"插件拥有宽权限"这一已知缺口；不得把 socket 不可达当作隔离证据。

### 禁止结果

以"Agent 连不上 host／guardian socket"作为安全结论；为了"更安全"而切断 A 类，使 Agent 的既有工作流
失效；把 `--allow-read` 之类的宽权限描述成已经收窄；用同一 UID 下的自查代替能力判据。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| A 类合法 IPC 在执行本条后仍可用 | 现有的 Agent 会话与插件测试覆盖 stdio 通道（`desktop/crates/plugin-runtime/src/lib.rs` 经 stdio 派生插件）；尚缺"隔离落地后 A 类逐项仍可用"的专门回归 | Partial |
| B 类管理操作对非管理身份被拒绝 | `desktop/crates/process-runtime/src/host/server.rs:73`、`desktop/crates/process-runtime/src/guardian.rs:38,136`、`desktop/crates/process-client/src/transport.rs:35` 校验对端 UID；尚缺以 W 身份逐项尝试 B 类接口的测试 | Partial |
| C 类不可达 | 尚未建立：C 类通道在 Node 侧不存在，Agent 与 clone 今天共用同一个 OS 身份（`uid=1000`），因此没有可断言的隔离面。授权半程已完整落地并有测试（`cloud/internal/cloneauth` 的签发与验签、`cloud/internal/credentialstore` 的 `Resolver` 兑换、`cloud/internal/cloneexec` 的 `broker-resolver` 客户端）；部署栈今天**已经**签发 `broker-resolver` 证书，但它只写入执行器容器挂载的卷，Node 侧没有任何进程能读到它。这是"Agent 拿不到 C 的身份材料"的部署侧证据，而**不是**"Agent 尝试后失败"的能力判据。**2026-10-13 更正**：本行原先写"Cloud 的 `Issue` 仍无生产调用方"，这对**镜像通道**不成立——`core.MirrorDispatcher.Prepare`（`internal/core/mirror_dispatch.go:120`）是 `cloneauth.Issue` 的真实生产调用方，由 `cmd/server/main.go:115,220` 的 `PrepareMirrorsOnce` 循环驱动，启用了镜像通道的隔离栈上已验证真实 Cloud 进程签发、真实 Store 兑换。仍然成立的是**按凭据直连**那一侧：`internal/core/credentialed_clone.go:125` 的 `Issue` 只在 `CLOUD_CLONE_DISPATCH_ENABLED=true` 时安装，该开关默认关闭，因此 Node 侧证据仍然不存在 | Missing |
| D 类不能提升身份 | 需要跨身份提升尝试的完整矩阵；均未建立 | Missing |
| 宽权限缺口被如实记录而非掩盖 | `desktop/crates/plugin-lifecycle/src/permissions.rs:106-113` 的 `agent_permissions()` 今天返回 `AllowRun`／`AllowRead(Everything)`／`AllowEnv`／`AllowNet`，文档中已有注释承认"roughly as privileged as the host"；这是记录，不是收窄证据 | Partial |

### 决策依据

Node 隔离 ADR D16、D4、不变量 3、4、6、8、9 与验收义务 17、19；`desktop/crates/plugin-lifecycle/src/permissions.rs`
的既有权限注释。

## Anonymous Clone Keeps Its Existing Identity And Observable Behavior

### 风险

为了让"所有 Git 都在 C 下运行"这一更简单的模型成立，把无凭据的匿名公开仓库 Clone 一并迁到 C；
于是公开仓库 Clone 的可观察行为发生变化（目录归属、权限、身份），且在没有凭据配置的 Node 上直接
不可用。

### 前置状态

受控 Node 配置了 `workload_uid`；某 Project 没有绑定任何凭据引用（匿名公开仓库路径）；
同一 Node 上同时存在 Agent 会话。

### 触发

对无凭据 Project 执行 Clone；检查执行身份、目标目录的归属与权限、结果与失败语义；
与本条落地前的行为逐项比较；同时在 `SessionWorkload::Shared`（未配置 Agent 工作负载目录或
`workload_uid`）与 `Separate` 两种配置下各触发一次。

### 必须成立

- **无凭据的受管 Git 长期继续使用既有的 `process.workload_uid`**，不迁移到 C。这是长期契约，
  不是过渡状态。
- 公开仓库匿名 Clone 的可观察行为**逐位不变**：目录分配、归属、权限、结果内容与失败语义都不变。
- 凭据执行身份 C 只用于**将来的**受控凭据 Clone；P1-M1 不启用任何凭据执行路径。
- Agent 会话与 Revision、Workspace 生命周期不因引入 C 而改变行为。

### 禁止结果

把全部 Git 操作统一迁到 C 以简化实现；把匿名 Clone 的目录归属或权限改为 C；因引入 C 而要求所有
Node 都必须配置凭据相关配置；以"未来会统一"为理由现在就改变公开仓库 Clone 的行为。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 匿名 Clone 仍以既有 `workload_uid` 执行 | `desktop/apps/ora-node/src/managed.rs:38-56` 今天对**所有**受管执行统一套用 `setpriv` 到 `workload_uid`；尚缺"身份在引入 C 之后逐位不变"的回归 | Partial |
| 公开仓库 Clone 的可观察结果逐位不变 | `desktop/apps/ora-node/tests/standalone/repository_policy.rs` 与 `apps/ora-node/tests/standalone/repository_commit.rs` 等覆盖 HTTPS 受管 Clone 的成功与失败路径；尚缺身份维度的前后比较 | Partial |
| Agent／Revision／Workspace 生命周期不受影响 | 既有 Agent 会话、Revision 与 Workspace 测试覆盖当前行为；尚缺"引入 C 后仍全绿"的回归 | Partial |
| C 只用于凭据 Clone（P1-M1 不启用） | 执行器按构造只做凭据 Clone：`cloud/internal/cloneexec` 的五步契约先验签、再兑换、才物化并克隆，没有授权就拒绝，因此它不会、也不能执行匿名 Clone（`TestExecuteRefusesBeforeSpendingAnything` 的 `no_authorization` 一项）。匿名 Clone 仍完全走 Node 既有路径，执行器不挂载任何 Workspace 卷、也不接收 Node 的 clone 请求。**尚缺**"无凭据配置的 Node 也能启动并匿名 Clone"的专门回归 | Partial |

### 决策依据

Node 隔离 ADR D17、D14-B 第 2 项、不变量 10；最小 clone 执行契约 D2 及不变量 4；权限仅面向
受控 Node 的凭据路径。

## Partial Credential Configuration Disables Only Credentialed Execution

### 风险

两种相反的失败：把"没有声明凭据 Clone"当成整个 Node 启动失败，使今天的匿名 Clone 与 Agent 工作流
不可用；或者把"声明了凭据 Clone 却无法建立身份边界"当成可以继续启动，让凭据路径在一个不成立的
身份模型上运行。

### 前置状态

分别构造：未声明凭据 Clone 的 Node；声明了凭据 Clone 且身份边界可建立；声明了凭据 Clone 但缺少
`workload_uid`（或 C 的目录与权限无法建立）；以及 `SessionWorkload::Shared`（Agent 与 Node 同身份）
的配置。

### 触发

在上述四种配置下各启动一次 Node；对未声明凭据的实例执行公开仓库匿名 Clone 与 Agent 会话；
对声明但无法建立边界的实例检查启动结果与拒绝原因。

### 必须成立

- **未声明凭据 Clone**：只禁止凭据执行，Node **正常启动**；匿名公开仓库 Clone 与 Agent 工作流保持可用。
- **声明凭据 Clone 且边界可建立**：正常启动，但凭据 clone 仍受 fail-closed 拒绝，直到 D14-B 有证据。
- **声明凭据 Clone 但无法建立身份边界**（缺 `workload_uid`、C 的目录与权限无法建立、
  `SessionWorkload::Shared` 使 W 与 M 同身份）：**整个 Node 启动失败**，不降级运行。
- 判断依据是"是否需要凭据执行"，而不是"是否配置了 C 的目录"；不得用"没配 C 就当没这功能"掩盖
  声明过凭据路径却边界不成立的情形。

### 禁止结果

把缺少凭据配置变成启动失败；把声明凭据却无边界变成静默降级或仅在 clone 时才失败；在
`SessionWorkload::Shared` 下运行凭据执行；以"环境变量没设"为由让边界检查通过。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 未声明凭据时 Node 正常启动且匿名 Clone 可用 | `desktop/apps/ora-node/src/main.rs:39-46` 今天只在网络管理且缺 `workload_uid` 时拒绝启动；凭据相关的启动判据尚未实现，需要四类配置各自的启动测试。执行器一侧的对应性质由部署天然满足：`CLOUD_CLONE_DISPATCH_ENABLED` 默认关闭，不配置执行器的栈照常运行匿名 Clone 与 Agent 工作流 | Missing |
| 声明凭据但边界不成立时整体启动失败 | 执行器一侧已有直接证据：`cloud/internal/config/clone_executor_test.go` 的 `TestCloneExecutorRefusesAnIncompleteDeployment` 用 18 项变异逐条断言"部署不完整就不启动"（缺物化根、物化根与 checkout 同目录、缺 helper、helper 含 shell 元字符、缺验证公钥、缺服务身份、缺 Store 端点等），且 `cloneexec.New` 在缺任一必需项时返回错误。**Node 一侧**的判据尚未实现 | Partial |
| `SessionWorkload::Shared` 下不得运行凭据执行 | `desktop/apps/ora-node/src/service/agents.rs:80-100` 今天在缺少 `workload_directory` 或 `workload_uid` 时返回 `SessionWorkload::Shared`（Agent 与 Node 同身份）；尚缺针对该配置的凭据拒绝断言。执行器不在 Node 内运行，因此该配置今天不会使它降级执行 | Missing |

### 决策依据

Node 隔离 ADR D17、D9、D14-A／D14-B、不变量 11、12、13 与验收义务 18。

## Credential Materialization Is Destroyed After Every Outcome

### 风险

凭据材料在成功、失败、超时、`Interrupted` 或崩溃之后留在磁盘、环境或进程表里，使"一次执行的凭据"
变成长期可用的残留；或者物化删除失败却把执行记为成功。

### 前置状态

一次凭据 clone 执行正在物化凭据与授权；具备可观察的物化目录；可以制造成功、失败、超时、`Interrupted`
与 Node 强杀五种结局。

### 触发

分别制造五种结局；在每种结局后检查物化目录、环境变量、命令行与进程表；再重启 Node 并检查启动清扫；
并制造一次物化删除失败。

### 必须成立

- 五种结局之后物化目录与授权载体都被销毁；执行结束后没有可复用的凭据残留。
- 物化删除失败**不得**使执行被记为成功；失败是可解释的失败。
- 启动清扫可证明崩溃后的物化目录为空；清扫先于接受任何新工作。
- 销毁接在既有的收尾屏障之后，不绕过 `Interrupted` 判定与 scope 收尾。

### 禁止结果

以"进程退出即回收"代替显式销毁、把删除失败吞掉并报成功、在崩溃重启后直接接受新工作而不清扫、
把凭据放进命令行参数或环境使其出现在进程表里。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 五种结局后无残留 | 已有**三种**结局的直接证据：成功（`cloud/internal/cloneexec/acceptance_test.go` 的 `TestCredentialedCloneOfARealPrivateRepository`——真实私有仓库 Clone 成功后物化根目录条目数为 0，且 Token 在物化根与 checkout 下均不出现）、失败（同文件 `TestAFailedCloneStillDestroysTheCredential`，以及 `executor_test.go` 的 `TestExecuteDestroysTheMaterialWhenTheCloneFails`）、崩溃后清扫（`acceptance_test.go` 的 `TestTheStartupSweepRemovesWhatACrashLeftBehind`、`executor_test.go` 的 `TestSweepRemovesMaterialLeftByAPreviousRun`）。**超时**与 **`Interrupted`** 两种结局尚无直接证据——执行器今天没有接入 Node 的 clone 状态机，因此 `Interrupted` 判定不适用于这条路径（见 `20261012` D4） | Partial |
| 删除失败不记为成功 | `cloud/internal/cloneexec/executor_test.go` 的 `TestExecuteReportsACleanupFailureRatherThanASuccess`：物化删除失败时以 `credential_executor_cleanup_failed` 失败，不返回成功 | Covered |
| 崩溃后启动清扫证明目录为空 | `cloud/internal/cloneexec/acceptance_test.go` 的 `TestTheStartupSweepRemovesWhatACrashLeftBehind` 与 `executor_test.go` 的 `TestSweepRemovesMaterialLeftByAPreviousRun` 覆盖"清扫移除上一次运行留下的物化目录"，且执行器在监听前先清扫（运行栈日志 `sweptEntries`）。**尚缺**真实强杀加重启的端到端清扫证据：现有测试构造的是遗留状态而不是真实崩溃 | Partial |
| 凭证不经 argv 或环境暴露 | `cloud/internal/cloneexec/executor_test.go:655-663` 读取 Git 替身记录的 argv，断言 Git 收到的是部署固定的 helper 路径与执行 id；`git.go` 的 `gitEnvironment` 是克隆进程的**全部**环境。Token 只在 helper 与 Git 之间的 `credential` 协议里流动，不进入 argv（argv 携带的是物化根与执行 id，helper 自己从磁盘读 Token）。**尚缺**对运行中进程表与 `/proc/<pid>/environ` 的直接检查 | Partial |

### 决策依据

Node 隔离 ADR D11、D13、不变量 5、7；Node 隔离 ADR D14-B 第 9 项；最小 clone 执行契约不变量 4。

## Credentialed Clone Uses One Single-Use Authorization And Never Falls Back

### 风险

一次 clone execution 重放旧授权、跨执行复用授权，或在凭据不可用时回退匿名 Git 或部署静态凭据；
于是"私有仓库必须以用户自己的身份访问"这一承诺失效，而失败又不可解释。

### 前置状态

一次凭据 clone 已被受理并绑定 `credential_ref_id` 与 `credential_ref_version`；Cloud 签发了单次授权；
Node 侧具备 C 身份的 `broker-resolver` 客户端；helper 是部署固定、root 拥有且不可写的白名单程序。

### 触发

正常兑换一次；重试同一执行；在授权到期后重试；在凭据被冻结后重试；让 helper 解析失败；让解析超时；
检查 Git 的 URL、重定向、代理、credential helper 与 checkout 内 `.git/config`。

### 必须成立

- 一次执行使用**一次**独立授权；失败、超时、`Interrupted`、崩溃或重试都申请新授权，不重放、不跨
  执行复用。
- 凭据来源不可用、授权到期或凭据被冻结时**明确失败**：不回退匿名 Git，也不回退部署静态凭据。
- 凭据不出现在 Git URL、argv、`.git/config`、重定向目标、代理配置或业务结果中。
- helper 与执行程序由部署固定，业务请求不能指定。

### 禁止结果

以部署静态 PAT 作为用户凭据路径的回退、把 Token 放进 URL 的 userinfo、允许请求指定 helper 或代理、
失败后静默改走匿名 Clone、把原始 stderr 存入业务结果。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 单次授权、不重放、不跨执行复用 | `cloud/internal/cloneexec/acceptance_test.go` 的 `TestTheAuthorizationIsConsumedOnceAndReplayIsRefused`：对着真实 Credential Service、真实卷兑换一次成功后，同一授权再次兑换被以 `authorization_consumed` 拒绝。跨执行复用在两处被直接证明：`executor_test.go` 的 `TestExecuteRefusesBeforeSpendingAnything` 覆盖"授权属于另一次执行"的拒绝，`credentialstore` 侧以 `O_CREATE\|O_EXCL` 消费记录作为原子单次判定，并在运行栈上实测重启 Store 后重放仍被拒绝（消费记录持久在 Store 自己的卷上，`0600`，不含秘密） | Covered |
| 不可用时拒绝而不是回退 | Cloud 侧既有证据不变（`integration.repository_credentials_test.go` 的 `TestCredentialedCloneRefusesMissingExecutorAndAnonymousFallback`、`integration.git_credential_binding_test.go` 的 `TestCredentialedCloneStaysRefusedOnEveryPath`）。执行器侧新增：授权缺失／格式错／已过期／绑定错配在执行前拒绝（`TestExecuteRefusesBeforeSpendingAnything`）；凭据已撤销、版本不可用、跨凭据分别以 `TestARedemptionIsRefusedAfterRevocation`、`TestARedemptionIsRefusedForAVersionTheServiceNoLongerHolds`、`TestARedemptionIsRefusedWhenTheAuthorizationNamesAnotherCredential` 拒绝；部署未配置验证公钥时 `TestTheDefaultDeploymentRefusesEveryRedemption` 证明每个兑换都被拒绝。`TestAFailedCloneStillDestroysTheCredential` 证明克隆失败以 `credential_executor_clone_failed` 结束，不会静默改走匿名 Clone | Covered |
| 凭据不进入 URL、argv、`.git/config`、重定向与代理 | `assertNoTokenUnder`（`acceptance_test.go`）与 `assertTokenGone`（`executor_test.go`）遍历执行器写下的全部内容断言 Token 不出现，覆盖 checkout 内的 `.git/config`；`git.go` 的克隆参数按 D6 固定 `credential.useHttpPath`／`http.followRedirects`／代理键，且 URL 中不含 userinfo。**重定向这一半本轮补上了交出点的直接证据**：helper 不再只比对主机，而是把 Git 请求的 host 与 path 经 `credentialmaterial.NormalizeScope` 规范化后与执行器物化的仓库范围**整体相等**比对，任一项不同即拒绝（`cmd/git-credential-ora/main_test.go` 的 `TestRunRefusesARequestThatDoesNotMatchTheScopedRepository`：同主机不同路径、子路径、父路径、`repo.git.git`、大小写不同的路径、把路径换成百分号编码的另一拼写、带 query、带 fragment、另一主机、共享后缀的主机、缺 path 全部拒绝；`TestRunMatchesTheHostCaseInsensitivelyButNotThePath` 与 `TestRunMatchesTheGitSuffixInBothDirections` 固定规范化只削减装饰；`TestRunRefusalDoesNotDiscloseTheStagedScope` 固定拒绝信息不回显范围），物化的范围本身取自**已签名绑定**里的仓库（`internal/cloneexec/executor.go:325` 的 `CanonicalRepository` 后再与 `binding.Repository` 比对），不是调用方可复述的值；旧格式（裸主机、无 `=`）的范围文件被 `ReadScope` 拒绝而不是按主机放行。这让"重定向不会带走 Token"不再只依赖 `http.followRedirects=false` 的文档承诺，而是在秘密真正交出的那一处强制。**仍缺**：一次真实重定向的端到端断言（今天证明的是"交出不匹配就不发生"，不是"Git 遇到 302 时的实际行为"），以及代理配置的独立断言 | Partial |
| helper 与执行程序由部署固定 | `executor_test.go` 的 `TestNewRefusesADeploymentThatCannotEstablishTheBoundary`、`TestHelperArgumentRefusesAPathGitCouldSplitOrExecute`、`TestCloneArgumentsRefuseAJobGitWouldRunSomethingElse` 覆盖部署期与作业期的路径拒绝；运行栈实测 `/app/git-credential-ora` 与 `/app/clone-executor` 为 root 拥有 `0755`，以 C 身份写入被拒（`Permission denied`），且 `CLONE_EXECUTOR_HELPER_PATH`／`CLONE_EXECUTOR_GIT_PATH` 是部署环境变量，作业不能指定 | Covered |

### 决策依据

凭据来源修订 D1、D2、D3 与不变量 1–7；Node 隔离 ADR D3、D5、D12；最小 clone 执行契约 D1、D2、
不变量 4；[`minimal-clone-execution.md`](minimal-clone-execution.md) 的
`Clone Input And Authentication Stay Noninteractive`。
