---
status: proposed
---

# checkout 交付到 Workspace 的通道：Cloud 的只读交付端点先落地，最终并入 Git Server 的只读取回——第一阶段不修改 `desktop/`

带凭据的 clone 今天止步于执行器：C 在自己私有的 `checkout_root` 里产出**一个 commit 与一个目录**，
Workspace 卷上什么也没有，因此 Agent 的会话工作目录（Node 的 checkout）不存在，
`repository_credential_executor_unavailable` 继续无条件拒绝（
[凭据执行器归属](20261012-credential-executor-belongs-to-the-current-product.md) D5 与"本决策未解决的问题"）。
本文裁决**那一步怎么走**：跨容器/跨卷把一个已经生成的 checkout 交付到 Workspace 卷、并在交接后让
W（`process.workload_uid`，1000）能正常读写它。

本文的 `status` 是 `proposed`：**推荐已定、批准未到**。前序 ADR 把这一个问题明确留给了项目负责人
（[凭据执行器归属](20261012-credential-executor-belongs-to-the-current-product.md) 的 D5 表与"本决策
未解决的问题"），本文即那份裁决材料。它**不再比较方案**：D2 选定一条通道，其余各节给出该通道的全部
代价、最小修改清单与逐条待批准事项。

因此本文的目的不是自行选择通道，而是把**候选、代价与不可绕过的硬约束**摆到可以直接裁决的粒度。
D2 节是需要裁决的唯一一项；其余各节记录无论选哪条通道都成立的要求与不变量。

## 评审裁决（2026-10-12）：第一阶段不可实现——不修改 `desktop/` 就没有可用通道

**结论**：在 `desktop/` 冻结、且不违反已批准决策的前提下，本文原来推荐的第一阶段（Cloud 的只读交付端点
+ Node 用自己的 clone 取回）**不可实现**。原因不是端点难写，而是 Node 的 fetch **无法出示任何受控身份**：
今天 Node 侧可用的 Git 取回面只允许「把秘密放进 URL」这一种鉴权形态，而这正是评审要求 1 与 3 明确拒绝的
形态。据此**撤回原 D2 的第一阶段**，并按评审要求 1–7 重列约束；`repository_credential_executor_unavailable`
继续逐位拒绝，本裁决**不授权实施任何新信任边界**。

### 评审提出的七条要求（作为本通道的准入约束）

| # | 要求 |
|---|---|
| R1 | 不以 URL 持有权单独授权读取；采用受控客户端身份、逐请求服务端鉴权，或可证明等效的机制 |
| R2 | 授权绑定 Tenant、Project、Workspace、Execution、Runtime Generation、仓库与有效期 |
| R3 | Cloud 与 Node 的台账都不得保存可直接兑换代码的秘密 URL |
| R4 | 服务端不可列举、不可跨执行访问；执行结束后关闭读取并清理镜像 |
| R5 | C 的导出目录与 Token 物化根完全隔离 |
| R6 | Node 必须实际完成正常 Checkout，不得绕过已有 Workspace 生命周期、结果结算与收尾规则 |
| R7 | 必须给出与正式 Git Server 的兼容／迁移计划；不得依赖未来迁移掩盖当前漏洞 |

### 最小不可行证据（只读 `desktop/` 得出，逐条给出位置）

| # | 事实 | 位置 | 推论 |
|---|---|---|---|
| E1 | 每条命令注入**固定**的 `-c` 前缀：`protocol.allow=never`、`protocol.https.allow=always`、`protocol.ssh.allow=never`／`always`、`core.hooksPath=/dev/null`、`core.askPass=`、LFS 与 submodule 键 | `apps/ora-node/src/repository/config.rs:135-152` `constrain()` | 没有 `http.extraHeader`、`http.sslCert`、`http.sslKey`、`credential.helper`——**无法按执行注入 Authorization 头、客户端证书或凭据助手** |
| E2 | Cloud 部署的 Git 配置只允许 CA 引用：非 `[http]` 段、或 `[http]` 下除 `sslCAInfo` 以外的任何键都是配置错误；SSH 身份一律禁止 | `config.rs:30-51` `validate_cloud_policy()` | 部署侧也**无法**为交付主机配置客户端证书 |
| E3 | 环境是固定的：`PATH`、`GIT_CONFIG_NOSYSTEM=1`、`GIT_CONFIG_GLOBAL=<策略校验过的文件>`、`GIT_TERMINAL_PROMPT=0`、`GIT_ASKPASS=""`、`SSH_ASKPASS=""`、`GIT_LFS_SKIP_SMUDGE=1`、`GIT_OPTIONAL_LOCKS=0` | `config.rs:106-124` `environment()` | 没有环境变量位可放一次性秘密，交互与 askpass 取凭据都被关闭 |
| E4 | E1+E2 之下 SSH 为 `never`，剩余取回形态只有 `https://`，而 HTTPS 的鉴权手段已被 E1–E3 全部关闭 | E1–E3 | **唯一可用的鉴权就是把秘密放进 URL**，即 R1／R3 拒绝的 bearer URL |
| E5 | 只有一个 `ProcessConfig.workload_uid`，被 `isolate_workload`（`setpriv --reuid/--regid --clear-groups --no-new-privs`）应用到**每一个** RunSpec；clone 与 Agent 会话用同一个 uid | `apps/ora-node/src/managed.rs:25-56`；clone：`apps/ora-node/src/repository/drive.rs:261`；会话：`apps/ora-node/src/service/agents.rs:86-95` | 不存在「第三个身份、W 不可遍历的物化根」：任何交给 fetch 的秘密，**Agent 都读得到**。这不是「可证明等效」的机制，而是已批准隔离决策要求先解决的前提 |
| E6 | `repository_root` 被冻结且必须与受保护状态／checkout 不重叠；Node 只承认自己独占创建的目录 | `config.rs:53-90` | 「别的组件写好目录、Node 直接采用」不成立——这也是 β／γ 被否决的同一依据 |

**合并结论**：第一阶段的形态只剩两种可能——把读取权放进 URL（R1／R3 拒绝），或让 fetch 出示受控身份
（必须修改 E1–E5 中的至少一条，也就是**必须改 `desktop/`**）。**不存在「不改 `desktop/` 且满足 R1–R7」的
第三条路。**

### 必须调整的约束（裁决项，本文不代选）

| # | 必须调整的约束 | 最小改动面 | 对 R1–R7 的判定 |
|---|---|---|---|
| T1 | 放开「Cloud Git 配置只允许 CA 引用」，允许**部署为交付主机**配置客户端证书（`[http "https://<delivery-host>/"] sslCert/sslKey`），私钥固定在 Agent 不可遍历的路径 | `config.rs:30-51` 的白名单、`constrain()` 的固定前缀、部署编排；不改命令形状与 proto | 满足 R1（mTLS 身份 + 逐请求服务端鉴权，URL 只带 execution id，不构成读取权）、R2、R3（台账无秘密 URL）、R4、R5、R6。**但 E5 仍在**：证书对 Agent 可读，Agent 可冒名访问该端点；只要服务端把授权绑定到五元组而非「持证即授权」，跨执行读取仍被拒绝，冒名的审计与限流面需要单独裁决 |
| T2 | 允许逐执行注入 `http.extraHeader: Authorization` | `constrain()` 增加一条按执行传入的键 | **建议否决**：一次性秘密必须经环境或配置进入与 Agent 同 uid 的进程（E5），违反已批准隔离决策；「受控客户端身份」不成立 |
| T3 | 授权 desktop 侧**受控取回身份**：第三个 OS 身份、W／C 不可遍历的物化根、部署固定的 broker 客户端，按执行与代次兑换一次读取授权 | 前序[隔离决策](20261010-controlled-node-git-authentication-isolation.md)已写定的那份工作（clone 执行器形态、`managed.rs` 的身份模型、部署编排） | 完整满足 R1–R7；代价最大，且正是本 Issue 执行侧那一半的既有阻塞 |
| T4 | 不设第一阶段：保持无条件拒绝，等正式 Git Server 通道 | `desktop/` 0 个文件；等 [Git Server 决策](../../cloud/git-server/0-sandbox-scoped-git-server-over-https.md)获批并落地（其 D5 自带 Node 出示客户端证书的工作） | 满足 R1–R7（R7 天然成立：不存在需要退役的过渡通道）。代价是本 Issue 的真实 Clone → Workspace → Agent 验收在该通道落地前**不可能**取得 |

**建议**：若本轮仍坚持不修改 `desktop/`，唯一自洽的裁决是 **T4**——撤回第一阶段、fail-closed 保持、
本文只保留 D3–D7 作为「任何通道都必须满足的约束」，并把 R1–R7 记为不变量。若要在本 Issue 内取得真实
Checkout 验收，必须先在 **T1 与 T3** 之间裁决；**T2 应否决**。

### B′ 裁决（2026-10-12）：选 T4

项目负责人选择 **T4**：不设任何中间通道，`desktop/` 保持严格只读，生产 fail-closed 不解除。据此：

- 本 ADR 在 B′ 被裁决后**不改变任何运行时行为**：`repository_credential_executor_unavailable` 继续逐位
  拒绝，第 2–4 步不开始。R1–R7 记为不变量 13–19，是**任何**未来通道的准入约束。
- **Issue #16 的真实 Clone → Workspace → Agent 验收在本通道落地前不可能取得**，该项保持
  `Missing`／not-run，不得用别处的证据替代（原文该代价在 T4 行已写明，此处再次确认它是本次裁决的已知
  且已接受的代价）。
- T1 与 T3 未被授权：它们都要求修改 `desktop/`（T1 改 `apps/ora-node/src/repository/config.rs` 的配置
  白名单与 `constrain()` 前缀，T3 改 clone 执行器的身份模型与物化根），在授权之前不得实施。T2 否决。

### 目标通道自身的前置（2026-10-12 只读调查所得，待裁决）

T4 把取得验收的希望放在正式 [Git Server 通道](../../cloud/git-server/0-sandbox-scoped-git-server-over-https.md)
上。该通道**自身也不是"不碰 `desktop/`"的通道**，这一节把它今天的前置如实记下，避免 T4 被读成
"验收迟早自动可用"：

- 该通道的 D5 明确由 **Node 进程**执行 `git push`／`fetch`，并把 git-access 证书以
  `-c http.sslCert=… -c http.sslKey=…`、执行 id 以 `-c http.extraHeader=…` 传入。这些是 `desktop/` 侧
  的能力，今天的白名单与固定前缀都不允许（本 ADR 的 E1、E2 即该事实）。
- 该通道与已批准决策存在两处需要裁决的关系：① [凭据执行器归属](20261012-credential-executor-belongs-to-the-current-product.md)
  （approved）D2 与不变量 2 写的是"旧 Desktop 严格只读：不修改、不依赖、**不承载凭据执行**"，而按
  git-server D5 执行带凭据 Git 的正是 Node；② [隔离决策](20261010-controlled-node-git-authentication-isolation.md)
  （approved）D11 写的是认证材料"**不由环境、argv 或跨 exec 的文件描述符传递**"，而 git-server D5 正是
  经 argv 传证书与私钥。两者都需要一篇明确的裁决或修订，不能靠沉默通过。
- 该通道的 ADR 仍为 `proposed`，且其"未解决的问题"里包含上游镜像与 Node 从 Git Server 取回这两项——
  没有它们，Issue #16 需要的**带凭据 workspace clone 仍然不存在**。

结论：**在 `desktop/` 只读不变的前提下，今天不存在可交付的真实 checkout 通道**；T4 是这一前提下唯一
自洽的选择，但它把"取得验收"推迟到了需要上述裁决与实现之后。


## 推荐：选 α——Cloud 的交付端点发布只读镜像，Node 用自己的 clone 取回

> **已被 2026-10-12 评审取代**：本节推荐的 α 第一阶段（端点 + URL 里的逐次能力值）因违反 R1／R3
> 被撤回，可行性判定见上文「评审裁决」。本节保留为决策历史与代价对照，**不构成当前推荐**。


三条硬约束合起来只剩下一条通道不需要修改 `desktop/`。决定性依据是**约束二的方向性**：Agent 会话的
工作目录只能由 Node 自己的 clone 台账解析出来（`apps/ora-node/src/session/driver.rs:176` 的
`CheckoutDirectory` 取自 `shared.checkouts.checkout(spec.checkout_execution_id)`），而 Node 只承认
它独占创建的目录（`repository/drive.rs:242-249`）。

| 候选 | 能否产出 Node 认可、会话可用的 checkout | 是否改 `desktop/` | 代价落在哪一侧 |
|---|---|---|---|
| **α（本文推荐）** | 能——Node 执行一次自己的 clone，台账、独占创建、交接与 `delivery_checkout` 证据全部落在**已实现且有证据的匿名 clone 路径**上 | **否（0 个文件）** | 改一条**范围性禁令**（D8 标题级） |
| β | 需要 Node 新增受控命令，并在物化后**收养**一个不是它自己创建的目录 | 是 | 改**所有权不变量**（约束二的目录证据与 D12） |
| γ | 需要 Node 侧承认外部写入的目录，且额外把宿主卷写入权交给 Substrate | 是 | 改所有权不变量 **+** 让出 host 交互所有权 |

α 的取舍据此固定：**把代价从"改 Node 的所有权不变量"换成"限定性地修改一条范围禁令"**。它不触碰 D8
的另外三条、不触碰 D12、不触碰约束一与约束二；`20260925` 的收尾屏障与 `Interrupted` 判定在 α 下
**重新适用**，因为交付最终仍由 Node 的 clone 状态机产出。

### 与 Git Server 通道的关系：这是第一阶段，目标通道已由另一份决策给出

`specs` 的工作分支落后 `origin/main` 62 个提交，其中包含一份本文先前没有读到的根决策：
[Git Server 以 HTTPS 接收沙盒推送](../../cloud/git-server/0-sandbox-scoped-git-server-over-https.md)（`proposed`）。
它已经把"Cloud 仓库内的独立 Git Server 程序"这个**服务面**写定：smart HTTP over mTLS、按代签发的
`git-access` 客户端证书只回答身份、授权逐请求读 Cloud 数据库、两个监听按网络分开、不接受口令／静态 token／匿名。
它的 D5 规定由 **Node** 执行 Git 访问并以 `-c http.sslCert=… -c http.sslKey=… -c http.sslCAInfo=…` 出示证书，
因此"Node 出示客户端证书"是那份决策自带的工作，不是本文新引入的信任边界。

更关键的是它明确留下的下一步：D7 与"本决策未解决的问题"把**镜像上游、Node 改从 Git Server clone**
列为需要另立 ADR 的事项，并写明它"涉及上游凭据与仓库凭据连续性"——**那正是本 Issue 的工作**。

因此本文的推荐分两段，两段的边界必须写清楚：

> **已被 2026-10-12 评审取代**：下表中「第一阶段」一列不再成立（该形态要求把读取权放进 URL）。
> 第一阶段的当前状态是「不存在」：`repository_credential_executor_unavailable` 继续逐位拒绝，直到 T1／T3
> 之一被明确授权、或目标通道落地（T4）。

| 阶段 | 通道 | 授权形态 | 今天能否落地 | 前提 |
|---|---|---|---|---|
| **第一阶段（本文推荐先落地）** | Cloud 的只读交付端点（D2 的四条条件） | URL 中的逐次能力值 + 网络可达性 | 能：零 `desktop/` 变更、无 proto 变更、无新 effect kind | A–C 获批 |
| **目标通道（必须并入）** | Git Server 的仓库监听：Node 以 `git-access` 证书与 `Ora-Execution-Id` 取回交付 ref | 证书回答身份、授权逐请求取 Cloud 当前状态——**不留 bearer 秘密** | 不能：等待 git-server 决策获批及其落地顺序第 1、4 步（含 Node 侧证书支持） | git-server 决策获批；其 D7 的"镜像上游"由后续 ADR 补齐 |

两段的差别是**实质性的，必须摆明**：第一阶段的 URL 本身就是一个 bearer 秘密，它会落进 Node 的 clone 台账与
Cloud 的执行记录；目标通道没有任何这一类秘密，因为它用证书回答身份、用数据库回答授权。第一阶段的缓解全部是
结构性的（能力值只对应本 Workspace 自己的镜像、只覆盖本次执行、端点不列举、用后即删），但它**仍然是"把一份
可读内容编译成一个可传递的秘密"**。因此本文的推荐不是"A 与 A′ 二选一"，而是：

**批准第一阶段，并把"并入 Git Server 通道"作为它的完成条件（F 项）**：当 Git Server 的仓库监听能在同一次
`(tenantId, projectId, workspaceId, executionId, runtimeGeneration)` 绑定下只读地服务同一份交付内容时，交付端点
**退役**，而不是长期并存。若 F 项改为直接等目标通道，那么本 Issue 的真实 Clone → Workspace → Agent 验收
就要排在 git-server 决策获批与其实施之后——这是同一项选择的另一面，必须一起权衡，不能只批一半。

## 待批准事项（逐条）

| # | 事项 | 本文的建议 | 批准后才可执行的变更 |
|---|---|---|---|
| A′ | 是否批准以 URL 承载读取权的第一阶段 | **不批准**（已按 2026-10-12 评审撤回） | 无。原 A 项（限定修改凭据隔离 D8 的标题级禁令）随之作废：本通道不再要求引入新的 Git 服务面 |
| B′ | 在 T1／T3／T4 之间裁决 Node 的取回身份 | 坚持不改 `desktop/` → **T4**；要在本 Issue 内取得真实 Checkout 验收 → 在 **T1 与 T3** 之间裁决，本文不代选。**2026-10-12 已裁决：T4**（`desktop/` 严格只读、生产 fail-closed 不解除） | T1 改 `apps/ora-node/src/repository/config.rs` 的配置白名单与固定前缀（+ 部署编排）；T3 改 clone 执行器的身份模型与物化根。两者都**必须由新决策承载，不得改写已批准正文** |
| C′ | 是否解除 fail-closed | 不解除（D7 无条件不变） | 无 |
| D′ | 目标通道与退役判据 | 正式 Git Server 的只读取回是**唯一**目标通道；任何中间通道都必须写明退役判据，且不得依赖未来迁移掩盖今天的漏洞（R7） | 无 |
| E′ | D14-B 义务在容器边界下的证据映射 | 延后，且不得被声称已成立 | 无 |

## 最小修改清单

| 仓库 | 改动 | 关键约束 |
|---|---|---|
| `cloud/` | 新增交付端点部署单元（`cmd/` + `internal/`）：只读地服务 C 发布的镜像（dumb HTTP，与 `internal/cloneexec/acceptance_test.go` 已用真实私有仓库验证过的服务形态同源），按路径能力值逐次授权 | 不持有凭据，不解析秘密，不列举 |
| `cloud/` | `internal/cloneexec`：C 在成功 clone 与物化销毁之后，把结果固化为只读镜像并发布到导出区 | D13 的"销毁失败即失败"不变；发布先于销毁的写法一律拒绝 |
| `cloud/` | `internal/core`：带凭据的 clone 派发时把 `CloneExecutionSpec.repository` 指向本次执行的交付 URL；`clone_executions` 继续记录真实仓库 | 值变更，无 schema 变更、无 proto 变更 |
| `cluster/` | compose 增加交付端点服务与 C 导出卷（端点不发布宿主端口，只对 Node 容器可达）；setup 提供 CA 与固定配置键 | 不给 Substrate 任何卷或 Git 职责 |
| `specs/` | 本文 + 一份承载 A 项的新决策；`20260925` 两份决策的适用性按 D2 恢复 | 不改写已批准正文 |
| `cloud/` | **目标通道**：交付并入 Git Server 的仓库监听——镜像由 Git Server 作为只读 ref 服务，Node 以 `git-access` 证书与 `Ora-Execution-Id` 取回 | 逐条继承 git-server 决策的不变量 1–14；本文不重复设计它的鉴权 |
| `desktop/` | **第一阶段：0 个文件**。**目标通道**需要 Node 侧出示客户端证书，那是 git-server 决策 D5 自带的工作 | 这里的 0 个文件只在第一阶段成立；目标通道的 Desktop 变更不由本文引入，也不由本文免除 |

**无论选哪条通道，本文的不变量 1～12 都成立**，且**都不解除 fail-closed**：`repository_credential_executor_unavailable`
继续逐位拒绝，直到 D11-B 与 D14-B 的适用义务取得直接证据（见 D7）。

涉及的模块：Cloud 的 `internal/cloneexec`（发布只读镜像）、`cmd/clone-executor`、新增的交付端点部署
单元、Cloud 的 `internal/core`（clone 步骤的交付状态与记账）、`cluster/` 的 compose 与 setup。
**第一阶段不涉及 `desktop/`**：Node 的 clone 状态机、验证、所有权交接、`Interrupted` 判定与 `delivery_checkout`
证据全部按既有实现复用。目标通道（F 项第 6 步）需要的 Node 侧证书支持属于 git-server 决策的落地范围，不由本文引入。

兼容性边界：Node 的 clone 源协议（`https://`／`ssh://`）、`clone_executions.result` 与
`baseCommitId` 的既有语义、Cloud 业务库 schema（不新增 migration 除非交付状态需要）、旧 Desktop 仓库的
内容、匿名 clone 与 Agent 会话链路，都按硬边界处理。本文**不承担**修改已批准决策正文的义务，也不
承担"凭据执行器与 W 同容器"形态的恢复。

## 继承与修改

| 前序决策 | 处理 |
|---|---|
| [受控 Node 的仓库凭据执行使用独立 OS 身份](20261010-controlled-node-git-authentication-isolation.md)（approved）D12：clone 必须在 C 拥有、W 不可写的目标上进行；交接在 clone 完成**之后**、会话开始**之前**由 M 执行；交接前必须销毁物化；不得让 W 进入仍由 C 拥有的树 | **继承规则，形态由 D2 固定**：C 侧仍按 D12 在 C 拥有、W 不可写的目标上完成 clone，并在交接（= 发布镜像）之前销毁物化（D1）；Workspace 侧的取回是 Node 既有的匿名 clone 路径，不新增持卷者、不收养外来目录。因此 D12 在两侧都按原样成立，D6 只需重申其判据 |
| 同上 D8：V1 不引入 sidecar 容器或外部 Git 服务；禁止同一 Workspace 卷上的第二个容器；禁止让 Sandbox Server 执行 Git 或注入凭据；禁止把 Node 的 host 交互所有权让出去 | **逐条继承，其中标题级的一条在 D2 中被限定修改**：交付端点触碰"V1 不引入外部 Git 服务"，因此本文要求一份承载该修改的新决策（A 项），且把新服务限定为 D2 的四条条件（无凭据、逐次授权、无列举、用后即删）。其余三条——同一 Workspace 卷上不出现第二个容器、Sandbox Server 不执行 Git 也不注入凭据、不让出 Node 的 host 交互所有权——**全部保持**；γ 因触碰第三条被否决，β 因需要收养外来目录被否决 |
| 同上 D10：M 是受信任管理身份（root），负责准备挂载点、`chown` 与降权；M 不读秘密本体 | **继承，不变**：α 不新增持卷者，Workspace 侧的物化就是 Node 既有的 clone 与其后的交接触发点；M 的职责与"不读秘密本体"都逐条不变（C 的物化从不出现在 Node 容器里） |
| 同上 D11、D13：凭据材料只存在于 C 拥有、W 不可遍历、`noexec`／`nosuid`、**不在 checkout 树内**的叶子；崩溃后由启动清扫兜底 | 继承，并在容器形态下**加强为结构性要求**：物化根不得与任何会被交付通道触及的卷共卷（不变量 6） |
| 同上 D14-A／D14-B：开发准入与功能启用是两套门禁；D14-B 义务 14 是"交接后 Agent 可正常读写工作目录而物化已不存在" | 继承，不变；义务 14 的直接证据只有在 D2 裁决并实现后才可能取得，本文不产生启用证据 |
| [凭据执行器归属](20261012-credential-executor-belongs-to-the-current-product.md)（approved）不变量 5、6：C 不引入 sidecar、不挂载任何 Workspace 卷、不让 Sandbox Server 执行 Git | 继承，不变；本文不要求 C 取得卷访问权，交付通道必须在不给 C 挂载 Workspace 卷的前提下成立 |
| 同上 D5、落地顺序第 4 步：先裁决"checkout 如何交付到 Workspace"，再按最终实现同步本 ADR、D14-B 与 Cloud 侧 D11-B | 继承；本文即那一步的裁决材料 |
| [最小 clone 执行契约](20260918-minimal-clone-execution-contract.md) D2（经[凭据来源修订](20261010-minimal-clone-execution-contract-credential-source-amendment.md)扩展）与[用户 Git Token 由独立 Credential Service 承载](../../cloud/identity-access/20261010-user-managed-git-secret-store.md) D3、D10：凭据按 Project 逐次授权，只交付给 C，Cloud 从不保存密钥值 | 继承，不变；交付物是**已去凭据的结果**，交付通道上不存在任何秘密 |
| [Sandbox Server 生命周期经 Substrate 生效](../../cluster/sandbox-server/20260926-docker-sandbox-lifecycle-via-substrate-effects.md) 不变量 7：Sandbox Server 不执行 Git，不注入任何凭据 | 继承，不变：交付端点是 **Cloud 的独立部署单元**，不是 Sandbox Server；α 不让 Substrate 执行 Git，也不给它任何新的卷权限或卷职责 |
| [把带凭据的拒绝结算为终态](../../cloud/operation/20261012-credentialed-clone-refusal-is-terminal.md)（proposed）：能力不足时以该拒绝自己的 reason code 结算 Operation | 继承，不变；本文不改变"确定性拒绝即终态"的语义。能力**词汇表**的裁决不属于本文（见 [仓库凭据能力词汇表](../../cloud/identity-access/20261012-repository-credential-capability-vocabulary.md)），交付通道是否成立与"哪些能力存在"是两个独立问题 |
| [clone 副作用移出受理 worker](20260925-clone-git-effects-run-outside-the-admission-worker.md)、[终止尝试判为 Interrupted](20260925-terminated-clone-attempt-fails-as-interrupted.md) | **已由 D2 裁决：重新适用**。α 下交付最终仍由 Node 的 clone 状态机产出，因此这两份决策的收尾屏障与 `Interrupted` 判定逐条恢复；C 侧"销毁先于发布"的既有规则（D1）与它们不冲突 |
| [Git Server 以 HTTPS 接收沙盒推送](../../cloud/git-server/0-sandbox-scoped-git-server-over-https.md)（proposed）D1–D6：Cloud 仓库内的独立 Git Server 程序、`git-access` 证书只认证身份、授权逐请求读 Cloud 当前状态、两个监听按网络分开 | **继承为目标通道，本文不重复设计它的鉴权**：交付最终并入该通道的只读取回；第一阶段的端点是**限时的**（F 项），其 URL 能力值形态在并入后消失。该决策 D7 把"镜像上游、Node 改从 Git Server clone"留给后续 ADR，并写明它涉及上游凭据与仓库凭据连续性——本工作线（本文与[仓库凭据能力词汇表](../../cloud/identity-access/20261012-repository-credential-capability-vocabulary.md)）即那份后续 ADR 的组成部分 |

## 背景：为什么这是一个绕不过去的裁决，而不是一次实现选择

三条硬约束各自独立地在今天的代码里成立，合起来排除了"顺手实现"的所有路径。

### 约束一：Node 只接受 `https://` 与 `ssh://` 的 clone 源

`crates/node-protocol/src/domain/repository.rs:28-31` 对 clone 源做 scheme 白名单：

```rust
let (scheme, rest) = value.split_once("://").ok_or(InvalidCloneRepositoryUrl)?;
if !matches!(scheme, "https" | "ssh") || rest.starts_with('/') {
    return Err(InvalidCloneRepositoryUrl);
}
```

并有显式反例测试（`crates/node-protocol/tests/protocol/repository.rs:97`；`file:///tmp/repo` 在 `:103`）：
`file:///tmp/repo`、`http://example.com/repo`、`git://example.com/repo`、`/tmp/repo`、带 userinfo 的 https
一律拒绝；`apps/ora-node/tests/standalone/https.rs:14` 记下这条边界是**有意**的："no widening to `file://` in
production"。**"让 C 把目录放到某个路径、Node 从那个路径 clone"不成立。**

### 约束二：Node 不会接管一个不是它自己创建的 checkout

`apps/ora-node/src/repository/drive.rs:242-249` 用 `fs::DirBuilder::new().mode(0o700).create(&target.path)`
**独占创建**目标目录，**目标已存在时走 `AlreadyExists` 分支把这次 clone 记为 `Unknown`，而不是收养它**：

```rust
match fs::DirBuilder::new().mode(/*mode*/ 0o700).create(&record.target.path) {
    Ok(()) => {}
    Err(error) if error.kind() == io::ErrorKind::AlreadyExists => {
        return self.unknown_clone(&record).map(|()| None);
    }
```

创建成功后 `:260-262` 才把它 `chown` 给 `process.workload_uid` 并落目录证据；
`docs/node/repository-clone.md:26` 明确"Missing or replaced ownership evidence remains Unknown;
**a database reservation alone never owns a directory**"。因此**外部把文件预放到 checkout 路径不会被采纳**：
Node 既不认领也不采信这份目录，Agent 的工作目录仍然解析不到。约束一与约束二一起意味着：**能进入
Agent 会话的 checkout 必须由 Node 的受管 clone 产出，或者必须新增一条 Node 侧的物化路径。**

### 约束三：今天的部署形态里，没有第二个身份能同时满足"能写 Workspace 卷"与"被 Node 承认"

- Workspace 卷是宿主机上的 Docker 命名卷，由 `cluster/sandbox-server` 创建（`internal/lifecycle/effects.go:20,57`），
  只挂进该 Workspace 的 Node 容器（`:219`），并且显式拒绝同一卷上的第二个容器（`:48-56`）。
- Cloud 是集群服务，不挂载任何 Workspace 卷；C 按不变量 6 也不挂载。
- Controller 的 effect 请求映射对 proto oneof **穷尽**（`apps/ora-controller/src/cloud/substrate.rs:211-240`，
  且 plugin 两支已被显式拒绝），因此新增 effect kind 必然是一次 Desktop 变更。

于是三条约束的合力是：**要么改 `desktop/`，要么引入一个新的 Git 服务面，要么把宿主机卷写入权交给
Substrate。**三者都超出"实现细节"的范围，这正是前序 ADR 把它标成"新增架构冲突"的原因。

## 编号决策

### D1：交付物是"已去凭据的只读结果"，不是一个仍然活着的凭据执行现场

**规则**：交付通道上流动的只能是 C 已经完成 clone、已经完成凭据物化销毁、并已把结果固化为只读交付物
之后的**结果**。交付物必须携带可独立校验的身份：目标 commit、以及交付内容的摘要。任何时刻交付通道
上都不得出现授权载体、Git Token、`credential.helper` 配置或 Store 的解析结果。

**为什么**：D13 已经把"销毁"定义为结果的一部分（销毁失败即使 clone 成功也记为失败），D12 要求交接
**在销毁之后**而不是与销毁并发。把交付物定义成"销毁之后的结果"，使"交付通道泄漏秘密"在结构上不可能，
而不是靠交付通道自己小心。**代价**：交付物与执行现场解耦后，"重试"必须重新执行 C 的 clone 并重新
产出交付物（D5 的新执行、新授权），不能复用上一条残留。

> **已被 2026-10-12 评审取代**：下面的通道形态要求「URL 中的能力值不可猜测」作为授权载体，违反 R1／R3。
> D2 的**当前有效内容**是：在 T1／T3 被明确授权之前，本通道不存在，`repository_credential_executor_unavailable`
> 保持逐位拒绝（T4）。本节其余文字保留为代价对照与不变量来源。
>

### D2：交付通道——Cloud 的只读交付端点，Node 用自己的 clone 取回

**规则**：C 完成 clone 与物化销毁之后，把结果固化为**只读镜像**并发布到自己的导出区；Cloud 的交付端点
（一个独立的部署单元，**不属于 Sandbox Server、不持有任何凭据**）在本次执行的有效期内只读地服务该
镜像；Cloud 在派发时把 Node 的 clone 源指向该端点的、按本次执行逐次授权的 URL；Node 用**已有的 clone
状态机**取回，产出自己的台账记录与 `delivery_checkout` 证据。交接与 `Interrupted` 判定按 `20260925`
恢复适用。

交付端点必须同时满足四条条件，缺一不可：

1. **无凭据**：端点上不存在任何凭据材料或解析结果，镜像里也不含（D1 已保证镜像只是"结果"）。
2. **逐次授权**：一次执行一个 URL，URL 中的能力值不可猜测；端点只服务与本次
   `(tenantId, projectId, workspaceId, executionId, runtimeGeneration)` 绑定的那一个镜像，不提供列举，
   不提供跨执行访问。
3. **只对该 Workspace 的 Node 可达**：端点不发布宿主端口，只接受来自该 Workspace Node 容器的请求；
   compose 网络的扁平拓扑允许按对端地址再校验一次，作为能力值之外的纵深防御。
4. **用后即删**：镜像在交接完成后销毁，D4 的崩溃清理覆盖端点与导出区两侧，不得成为长期留存。

**D2 是第一阶段，不是最终形态**（F 项）：当 Git Server 的仓库监听能在同一次
`(tenantId, projectId, workspaceId, executionId, runtimeGeneration)` 绑定下只读地服务同一份交付内容时，Node 的
取回改走那条通道——证书回答身份、授权逐请求取 Cloud 当前状态——本节的 URL 能力值随之**消失而不是并存**。
**在那一刻之前，"逐次授权只能靠 URL 里的能力值"是 Node 今天唯一可用的取回方式，而不是本文认为它更优。**

**为什么是这条**：它把"一次带凭据的 clone 如何到达 Agent 的工作目录"还原成**已经实现、已经有证据的
匿名 clone 路径**——Node 用自己的台账、自己独占创建的目录、自己完成的交接。因此它不需要新增 Node
命令，不需要收养外来目录（β、γ 都必须放宽约束二依赖的所有权证据），也不需要新 effect kind（γ）。

**为什么"逐次授权"只能靠 URL 里的能力值**：Node 对 clone 源的策略是硬边界——scheme 白名单只有
`https`／`ssh`，带 userinfo／query／fragment 的 URL 一律拒绝（`crates/node-protocol/src/domain/repository.rs:28-47`），
`git_config` 的云端策略允许键只有 `[http] sslCAInfo`，且 `credential.helper`、`http.extraHeader` 与
`http.<url>.*` 都不在允许集内。**因此 Node 无法向交付端点出示任何凭据**，端点的隔离只能建立在"不可
猜测的执行绑定能力值 + 网络可达性"上。这条代价必须写进批准意见、不得含糊：**任何读到该 URL 的进程
都能取到那一份镜像**，而该 URL 会落在 Node 的 clone 台账与 Cloud 的执行记录里。缓解是结构性的——
能力值只对应**该 Workspace 自己的**仓库镜像、有效期只覆盖本次执行、端点不列举且用后即删，因此暴露面
止于"这个 Workspace 本来就有权获得的内容"；它不会让一次交付成为跨租户或跨 Workspace 的读取许可。

**代价（如实记录，不隐藏）**：

- 同一次内容被取两次（C 的网络 clone + Node 对镜像的 clone），网络与磁盘开销翻倍；
- 新增一个集群内的 Git 服务面，与 D8 标题级约束冲突，需要一次 amendment（A 项）；
- Node 的 clone 源不再是 Project 仓库，provenance 必须由 Cloud 侧记账补齐（B 项）：
  `clone_executions` 继续记录 Project 的真实仓库，交付记录记录镜像与端点 URL 的映射，`baseCommitId`
  仍来自 C 的真实 clone；
- 镜像在导出区与端点侧各有一份中间副本，两者的过期与删除路径都必须显式（D4）。

**若 A 项被否决**（即不接受为交付端点修改 D8）：本文的选择退化为 β，其最小 Desktop 修改清单是——
`node-protocol` 增加一条受控命令的请求／响应形状；`apps/ora-node` 增加命令处理、摘要校验、独占创建
与从交付物物化 checkout 的路径；`session/ledger.rs` 与 `repository/drive.rs` 的所有权证据必须为
"收养外部物化的树"新增一条可判定的判据；并配套 Desktop 侧自己的边界测试。**这份清单是代价对照，不是
本文的选择。**

**不列为候选**：让 C 直接写 Workspace 卷或给它宿主卷访问权（违反不变量的 6、9 与 D8 理由 3）；在
Node 容器里加 sidecar（D8 理由 1）；把凭据执行搬回 Node 容器（前序 ADR 已裁决其归属）；让交付端点
由 Sandbox Server 承载（违反 `20260926` 不变量 7）。

### D3：交付绑定到 tenant／project／workspace／execution／runtime generation

**规则**：一次 delivery 只能由 `(tenantId, projectId, workspaceId, executionId, runtimeGeneration)`
唯一确定，并且只能被该 Workspace 当前代次的 Node 取得。Cloud 侧的 delivery 记录必须与
`clone_executions` 的这次 execution 一对一；另一个 Workspace、另一个 project、或者同一 Workspace 的
**旧 runtime generation** 都必须被拒绝，而不是被"最近一次交付"这种模糊匹配放行。

**为什么**：`runtimeGeneration` 是运行时围栏的既有单位（`sandbox-server/internal/lifecycle/runtime_fence.go`、
`authorization.go`），而 Workspace 卷跨 sandbox 代次保留（`20260926` 不变量 6）。若交付只按 workspace 绑定，
一次旧代次的迟到交付就能改写新代次的 Agent 工作目录；绑定到 generation 与 execution 才是可判断的。

### D4：交付必须幂等、可恢复、可取消、可崩溃清理

**规则**：

- **幂等**：同一次 execution 的交付重复执行不产生第二份 checkout，也不改写已交接的树；重复请求返回
  同一个结果（沿用 `clone_executions.result` 的"注册时结果已写"口径，或等价的交付记录）。
- **可恢复**：交付进行到一半（下载中断、物化中断、交接中断）后重试必须能从可判定的状态继续或重来，
  不得留下 Agent 可以进入的半成品。判据必须是持久事实（目标目录的存在性、摘要、交接完成的证据），
  不是进程内状态。
- **可取消**：Operation 取消或 Workspace 停止时，尚未交接的交付必须可被丢弃；已经交接给 W 的树属于
  Workspace 数据，按既有的 `workspace_data_delete` 语义处理，不由交付阶段删除。
- **崩溃清理**：C 侧沿用启动清扫；持有交付物的中间副本（对象存储中的 bundle）必须有明确的过期或
  删除路径，**不得成为长期留存**——它是这次 execution 的结果，不是需要长期保存的产物。

**为什么**：交付是"跨越容器边界的外部副作用"，Cloud 的既有规则要求"在先持久化外部副作用计划与稳定
幂等键，再派发变更；结果不明确时按稳定外部身份对账"（`cloud/AGENTS.md`）。半成品被 Agent 进入是
本决策最严重的失败模式，因此 D4 与 D6 一起要求"交接完成"是可判定的证据，而不是尽力而为。

### D5：交付物的边界强化

**规则**：交付物与其物化过程必须对**来源仓库的内容**免疫：

- 来源仓库里的**符号链接**不得使交付或物化走出目标树：物化与所有权交接必须使用不跟随符号链接的
  原语（D12 已指定 `ora_utils::fs::own_tree_no_follow` 一类语义），且不得对交付物做会跟随链接的解包。
  这里有一条**已经成立、可以引用**的既有性质：C 的 clone 命令行把 `core.symlinks=false` 与
  `core.hooksPath=/dev/null` 一并固定（`cloud/internal/cloneexec/git.go:165,167`，与
  `protocol.allow=never`、`credential.helper=` 清空后在 `:169` 重新指定同处一条命令行），因此
  **C 产出的 checkout 目录由构造保证不含符号链接、不含可执行的 hooks**。但若交付物以 bundle 承载，
  则 Node 侧重新 checkout 时仍可能按对象里的 mode `120000` 创建符号链接，所以物化侧的原语要求不能
  因为这条性质而放松。
- 来源仓库里的 **`.git/config`、`hooks/`、模板、子模块、LFS filter 与 `credential.helper`／代理键**
  不得进入交付路径。以 `git bundle` 承载交付物在结构上排除了它们（bundle 只含 objects 与 refs）；
  Node 侧物化后产生的仓库本地配置必须由部署显式固定，沿用受控 clone 已有的收尾约束。
  这正是"把 Desktop 的 helper 与 clone 约束代码复制过来"这条替代方案被否决的同一个理由。
- 交付物不得覆盖**已存在的用户文件**：目标目录的创建必须是独占的（约束二已经如此），已存在即明确
  失败，不得合并、不得覆盖。

**为什么**：来源仓库是**不受信任的输入**，而交付是它进入 Workspace 的唯一通道。今天匿名 clone 的
等价保护由 Node 的受管 clone 提供（模板禁用、hooks 禁用、子模块不递归、验证干净工作树）；交付通道
必须提供同等强度的保护，无论它由谁执行。

### D6：交接的判据——"交接完成"必须是可判定的证据，且物化先于交接

**规则**：Agent 会话开始之前，必须已经存在以下证据：交付物的摘要与目标 commit 匹配；目标 checkout
由本次 execution 独占创建；树的所有权已变更为 W（`process.workload_uid`）；C 的凭据物化在本代次
**已经销毁**（且**不是**与交接并发）。任何一条不成立，本次交付以明确失败结束，**不得让 Agent 以 W
身份进入一个仍由其他身份拥有、或可能被其他身份继续写入的树**。

**为什么**：这是 D12 的最后两条规则（交接失败必须明确失败；不得跳过交接）在容器形态下的等价表述。
容器形态下 C 与 W 的文件系统不相交，因此"物化已销毁"与"所有权已交给 W"是两条独立的、都必须单独
取证的结论。

### D7：fail-closed 保持，启用门禁不变

**规则**：本文不解除任何拒绝，也不产生任何启用证据。

- `clone_dispatch` 继续保持默认关闭；关闭时带凭据引用的 Project 的 clone 继续逐位返回
  `repository_credential_executor_unavailable`。
- 只有 D11-B 与 D14-B 的适用义务取得**直接证据**之后才可解禁（前序 ADR 裁决第 8 项）。交付通道落地
  **不等于**这些义务成立：它只补齐 D14-B 义务 14 的一部分。
- D14-B 义务 1、2、3、11、16、17 是按"同一容器内的三个 UID"写的，本文不改写它们，也不声称它们在
  容器形态下已成立；这些义务的容器形态证据映射仍是未决项。

## 不变量

1. 交付通道上不出现任何秘密：授权载体、Git Token、解析结果与 `credential.helper` 配置都不进入交付物、
   交付请求、交付响应与任何日志。
2. 交付物是 C 完成 clone 且完成物化销毁之后的**结果**；交付与销毁不并发。
3. 交付的绑定是 `(tenantId, projectId, workspaceId, executionId, runtimeGeneration)` 五元组；另一个
   Workspace、另一个 project 或同一 Workspace 的旧代次一律拒绝。
4. 同一次 execution 重复交付不产生第二份 checkout，也不改写已交接的树。
5. 目标 checkout 由本次 execution 独占创建；已存在即明确失败，不合并、不覆盖用户文件。
6. **凭据物化根不与任何会被交付通道触及的卷共卷**：物化根只挂进 C 的进程，交付通道触及的卷里不存在
   物化目录。这是 D11 的"不在 checkout 树内"在容器形态下的结构性等价物。
7. 交付物与其物化过程不跟随符号链接走出目标树；来源仓库的 `.git/config`、`hooks/`、模板、子模块、
   LFS filter 与凭据相关配置键不进入交付路径。
8. Agent 会话开始的前置条件是"摘要与 commit 匹配 + 独占创建的 checkout + 所有权已交给 W + 本代次
   物化已销毁"，缺一不可；缺任一条则以明确失败结束。
9. C 不取得任何 Workspace 卷的访问权；Substrate 不执行 Git、不注入凭据；同一 Workspace 卷上不出现
   第二个容器。这三条在任一候选通道下都成立。
10. 本文不解除 `repository_credential_executor_unavailable`，也不产生任何启用证据。
11. 交付端点不持有任何凭据，只服务与某一次 `(tenantId, projectId, workspaceId, executionId,
    runtimeGeneration)` 绑定的镜像，不提供列举，交接完成后销毁副本。
12. 在 T1／T3 被明确授权之前，本通道**不修改 `desktop/`**：Node 的 clone 源策略（scheme 白名单与
    `git_config` 允许键）、目标目录的独占创建与所有权证据、会话 checkout 的解析路径都逐位不变。
    2026-10-12 评审已判定：正是这条把第一阶段挡死（见「评审裁决」）——它不是本通道的优点，而是它的成本。

**2026-10-12 评审的准入要求（R1–R7）——任何最终通道都必须同时满足，本文原 D3–D7 是它们的具体化：**

13. 读取授权不来自 URL 持有权；由受控客户端身份与逐请求服务端鉴权给出，或给出可证明等效的机制（R1）。
14. 授权绑定 Tenant、Project、Workspace、Execution、Runtime Generation、仓库与有效期；缺一项即拒绝（R2）。
15. Cloud 与 Node 的任何台账、日志、审计记录都不保存可直接兑换代码的秘密 URL（R3）。
16. 服务端不可列举、不可跨执行访问；执行结束后关闭读取并清理镜像（R4）。
17. C 的导出目录与 Token 物化根完全隔离，不同卷、不共挂载点（R5）。
18. Node 必须实际完成正常 Checkout：走既有的 Workspace 生命周期、结果结算与收尾规则，不绕过、不新建旁路（R6）。
19. 必须给出与正式 Git Server 的兼容与迁移计划，且不得依赖未来迁移掩盖当前漏洞（R7）。

## 为什么不是这些替代方案

| 方案 | 否决理由 |
|---|---|
| 外部把文件预放到 checkout 路径，期望 Node 采纳 | 违反约束二：Node 独占创建目标目录并以目录证据（device／inode／birth time）判断所有权，"仅凭数据库预约不拥有目录"。Agent 的工作目录仍然解析不到 |
| 让 Node 从 `file://` 路径 clone 交付结果 | 违反约束一：scheme 白名单只有 `https`／`ssh`，并有显式反例测试与"no widening to `file://`"的记录 |
| 给 C 挂载 Workspace 卷，让它直接写 | 违反不变量 9 与 D8 理由 1、3：同一 Workspace 卷上的第二个写入者，且把 Node 的 host 交互所有权让出去；前序 ADR 不变量 6 明确禁止 |
| 让 Cloud 直接写 Workspace 卷 | Cloud 是集群服务，不挂载任何 Workspace 卷；给它卷访问权等于把凭据路径的信任边界与宿主卷操作合并到一个进程 |
| 把交付并入 `sandbox_ensure` effect（用既有 kind 承载） | 语义错位：`ensure` 的生命周期是**沙箱代次**，交付的生命周期是**一次 execution**；两者在重试与 generation 上的单位不同，合并会让"重试 clone"与"替换沙箱"互相污染，且无法满足不变量 3 |
| 复制 Desktop 的 helper 与 clone 约束代码到正式产品来实现物化 | 会把 Desktop 已有的 `validate-then-use` 缺口与"未清除 `credential.helper`／代理键"的问题一起带过来（前序 ADR 已就同一理由否决过一次） |
| 让 Substrate 执行 Git 来完成交付 | 违反 `20260926` 不变量 7：Sandbox Server 不执行 Git、不注入凭据 |
| 把交付推迟到"Agent 需要时再拉取"（懒交付） | Agent 的工作目录在会话开始前必须已由 Node 解析为 checkout（`session/driver.rs` 的 `CheckoutDirectory`），懒交付会把失败暴露在会话中途，且让 W 获得一条主动取内容的路径，与 D6 的前置条件冲突 |
| β：对象存储交付 + Node 就地物化（原建议） | 最贴合 D8 原文，但要求 Node **收养**一个不是它独占创建的目录，而约束二的目录证据（device／inode／birth time）与 D12 的所有权交接正是为禁止这一动作而存在；代价落在"改所有权不变量"上，比 α 的"限定修改一条范围禁令"更重，且必须新增 Desktop 命令面。**它不是坏方案，只是在 A 项被否决时才成为唯一可选项**（清单见 D2） |
| γ：Substrate 一次性 helper 容器写入卷 | 需要新 effect kind（`substrate.rs` 对 proto oneof 穷尽，属 Desktop 变更），同时把宿主卷写入权交给 Substrate，且仍需 Node 侧承认外部目录——既不省 Desktop 变更，又额外触碰 D8 理由 3 |
| 把交付端点交给 Sandbox Server 承载 | 违反 `20260926` 不变量 7（Sandbox Server 不执行 Git、不注入凭据），并让宿主卷的所有者同时成为 Git 服务面 |

## 失败与恢复语义

| 情形 | 行为 |
|---|---|
| C 的 clone 成功但物化销毁失败 | 本次 clone 记为失败（D13 既有规则），不产出交付物 |
| 交付物上传失败或不完整 | 本次交付失败；Operation 按 clone 步骤的既有 defer 语义重试，重试产出新的 execution 与新的交付物 |
| 摘要不匹配 | 交付明确失败，不物化、不交接；不得以"内容看起来对"放行 |
| 目标目录已存在 | 明确失败，不合并、不覆盖 |
| 物化中断或交接中断（含 Node 崩溃） | 启动恢复必须以持久证据判定半成品并清理或重来；Agent 会话不得在证据不全时开始 |
| 交付迟于 Workspace 代次替换到达 | 按不变量 3 拒绝（旧 runtime generation），不得改写新代次的 checkout |
| Operation 取消或 Workspace 停止 | 未交接的交付可丢弃；已交接的树按 `workspace_data_delete` 语义处理 |
| 执行器未部署或 `clone_dispatch` 关闭 | 与今天逐位相同：带凭据的 clone 以 `repository_credential_executor_unavailable` 结算为终态 |

## 本决策未解决的问题

- **D14-B 义务在容器边界形态下的证据映射**（义务 1、2、3、11、16、17）：哪些直接证据等价、哪些需要
  新增断言式验收，仍属功能启用步骤的裁决。本文只要求它们不被声称已成立。
- **镜像与端点 URL 的记账字段**：B 项批准后，"真实仓库 → 镜像 → 端点 URL"的映射记在哪里（执行日志、
  交付记录，还是 `clone_executions` 的既有列）由实现变更一并决定；本文只固定一条——`clone_executions`
  继续记录 Project 的真实仓库，`baseCommitId` 仍来自 C 的真实 clone。
- **交付端点与导出区的保留时长**：镜像的过期时长与删除责任的归属，以及它是否需要在 Cloud 的 schema
  里有一条交付记录（若需要，是一次 forward migration）。今天的 `clone_executions.result` 只记 outcome
  与 commit，不承载一个可对账的外部对象。
- **交付是否应覆盖 `restart` 与 `stop/start` 后的会话恢复路径**：本文只要求会话开始的前置条件，未
  定义已交接的树在 Workspace 代次替换后的重放行为。
- **并入 Git Server 通道所需的"上游镜像"**（F 项）：git-server 决策的 D7 把"镜像上游、Node 改从
  Git Server clone"留给后续 ADR，并写明它涉及上游凭据与仓库凭据连续性；本工作线是那份 ADR 的一部分。
  在该 ADR 获批、且 T1／T3 之一被明确授权之前，**没有任何取回通道**：原来的第一阶段已按 2026-10-12 评审撤回，`repository_credential_executor_unavailable` 继续逐位拒绝。

## 落地顺序

1. **裁决取回身份（材料齐备，2026-10-12 评审后重写）**：D2 的第一阶段已撤回，需要裁决的是
   B′——在 T1（放开交付主机的客户端证书配置）、T3（desktop 侧受控取回身份）、T4（不设第一阶段、等
   Git Server 通道）之间选一条，并同时确认 C′（不解除 fail-closed）与 D′（唯一目标通道与退役判据）。
   本步不产生代码、测试或启用证据；**在 B′ 被批准之前，第 2–4 步一律不得开始**，第 5、6 步的前置条件
   按各自的决策执行。原第 1 步的其余文字随第一阶段的撤回一并作废。
2. **镜像与导出区（C 侧，可在不启用 `clone_dispatch` 的条件下独立验收）**：把成功的 clone 固化为
   只读镜像并发布到导出区，销毁先于发布，启动清扫覆盖残留。完成条件：镜像可被独立校验（commit 与
   摘要），发布路径上不存在凭据材料。
3. **交付端点（Cloud + `cluster/` 侧）**：实现只读服务、逐次授权、无列举、可达性限制与用后即删。
   完成条件：D2 的四条条件各有直接证据，含"跨执行取用被拒绝"与"列举不可用"。
4. **记账与派发（Cloud 侧）**：把交付建成 clone 步骤之后一个可判定、幂等、绑定五元组的阶段，并把
   Node 的 clone 源指向本次交付 URL。完成条件：失败恢复与重复交付有直接证据；不变量 3、5、8 在真实
   容器与真实文件系统权限下有证据。
5. **启用评估——不属于本文范围**：只有 D11-B 与 D14-B 的适用义务全部取得直接证据后才可移除
   `repository_credential_executor_unavailable`；届时先按本文实现同步前序 ADR、D14-B 与 Cloud 侧
   D11-B，再评估解禁。
6. **并入 Git Server 通道并退役交付端点（F 项，独立验收）**：在 git-server 决策获批并落地其第 1、4 步
   （`git-access` 身份与 Node 侧证书支持）之后，把同一份交付内容改为由 Git Server 的仓库监听只读服务、
   Node 以证书与执行 ID 取回；**完成条件是交付端点被删除**，且 `clone_executions` 的记账口径随之收敛到
   一个通道。这一步之前，第一阶段的 URL 能力值一直在用；这一步之后，它不存在。

保持不变的行为：`clone_dispatch` 默认关闭、带凭据的 clone 一律以
`repository_credential_executor_unavailable` 结算、匿名 clone 与 Agent 会话链路逐位不变、旧 Desktop
仓库内容不变（第一阶段不改 `desktop/` 一个文件；目标通道的 Desktop 变更由 git-server 决策承担）、Cloud
业务库 schema 不变（除第 4 步若确需一条交付记录）。
从本文生效开始改变的行为：**没有**——2026-10-12 评审撤回第一阶段之后，本 ADR 在 B′ 被批准之前
不改变任何运行时行为，`repository_credential_executor_unavailable` 逐位拒绝照旧。B′ 选 T1 或 T3 时，
行为按那时的新决策改变；选 T4 时，行为直到 Git Server 通道落地才改变，且本 ADR 不引入任何中间通道。

原第 7 步之后的文字（保留为历史）：

> 从本文生效开始改变的行为：**交付通道由 Cloud 的交付端点承担**——但仅在 A–F 批准之后按本文实现；
第 6 步生效后，它改由 Git Server 承担，交付端点退役。
