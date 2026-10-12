# 最小 clone 执行契约

依据[已批准的执行契约](../../../decisions/node/repository/20260918-minimal-clone-execution-contract.md)，
以及修改其 D6 的[中断失败判定](../../../decisions/node/repository/20260925-terminated-clone-attempt-fails-as-interrupted.md)。
本文件登记首版行为义务。协议、schema v3 和 Linux 受管 clone 已接通，已有真实 HTTPS／SSH 与恢复证据；
已有 Worktree／process 测试不替代这些业务证据。端到端职责另见[结果边界](clone-and-result-boundaries.md)；
clone 的 Git 移出受理 worker 后的受理与在途义务见[受理与 Git 执行分离](admission-and-git-execution.md)。

**凭据来源已被修订（2026-10-10）**：执行契约 D2 的"凭据由部署提供"由
[凭据来源修订](../../../decisions/node/repository/20261010-minimal-clone-execution-contract-credential-source-amendment.md)
扩展为"部署静态来源**或** Cloud 侧 Credential Service 按 Project 逐次授权解析，且后者只交付给凭据执行
身份 C"。受控 Node 上的凭据执行与执行身份隔离见
[Node 凭据执行身份隔离](git-credential-execution-isolation.md)；下面第一个用例的"凭据由部署提供"
一项按修订后的契约读，其余子句不变。

## Clone Input And Authentication Stay Noninteractive

### 风险

地址或参数成为任意程序入口；认证等待交互，或诊断泄露凭据。

### 前置状态

Node 显式配置 Git、环境和可信凭据来源（部署静态来源，或受控 Node 上的逐次授权动态来源）；分别准备
HTTPS、显式 SSH 服务以及认证失败、未知主机场景。

### 触发

通过生产入口提交合法请求、非法传输、含秘密地址、tag／revision 及需要交互的认证请求。

### 必须成立

- 仅接受 HTTPS 与显式 ssh://，输入为源仓库分支；拒绝本地路径、file、scp 缩写和自定义 remote helper。
- 不允许请求传入任意 Git 参数、目标路径、shell、环境或秘密；SSH 用户名不被当作密码拒绝。
- 凭据由部署提供的静态来源、或受控 Node 上按 Project 逐次授权的动态来源提供（凭据来源修订 D1）；
  两种形态都不由请求指定。缺少凭据或需要确认主机时明确失败，不弹窗、自动信任或无限等待。
- 网络和认证子进程受期限及收尾管理；journal、业务结果和日志不传播秘密，诊断有上限且脱敏。

### 禁止结果

通过扩大生产入口为本地路径绕开网络验证，复制 Controller 的 HOME／密钥，持久保存原始敏感 stderr。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 输入限制与安全参数调用 | `crates/node-protocol/tests/protocol/repository.rs` 的 6 项测试覆盖请求 wire、分片往返、传输／凭据／分支及未知输入拒绝；尚缺实际 Git 参数派发检查 | Partial |
| 非交互认证与子进程收尾 | `apps/ora-node/tests/standalone/repository_policy.rs`、`repository_ssh.rs` 覆盖真实部署凭据、未知主机与成功；`repository_recovery.rs` 覆盖网络中 Node 强杀后的原 scope 收尾；尚缺专门认证子进程超时测试 | Partial |
| 秘密不进入持久记录或输出 | `private_https_uses_deployment_credentials_without_running_checkout_extensions` 检查 Node／host SQLite 不含凭据内容；clone 使用 Discard 输出；尚缺敏感 stderr 注入及 guardian／日志联合检查 | Partial |

### 决策依据

执行契约 D1、D2，不变量 4；凭据来源修订 D1–D3。

## Clone Reserves And Verifies Its Exclusive Destination

### 风险

预留被误当成目录所有权；恢复覆盖已有、被替换或属于其他执行的目录。

### 前置状态

注入 repository_root 和 Node／host／guardian 状态路径；准备相等／父子重叠、已有空目录、用户文件和符号链接。

### 触发

接受 clone，在持久预留、排他创建、Git 派发之间中断；替换目录或改变配置后恢复并重传。

### 必须成立

- 根目录与状态路径不得相等或父子重叠；目标由 Node 分配，不直接使用 URL、分支或原始请求字段作目录名。
- 输入、目标与预留先持久化，随后排他创建，再派发；clone 管理变更串行执行。
- 已有目录即使为空也不接管；重传保持原目标，同源新执行不复用已有 checkout。
- 创建记录和实际目录身份共同支持归属证明；不明、被替换或配置不兼容时保留冲突／未知和现场。
- 结果持久完成之前不将目标作为可用仓库发布。

### 禁止结果

先派发后预留，按路径存在推断归属，覆盖或递归删除未知目录，恢复时静默换目标。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 路径隔离、独占创建和持久顺序 | `ora-node-db::tests::repository` 覆盖预留事务；`repository_policy.rs` 覆盖状态根父子重叠；`repository_commit.rs` 覆盖创建证据写入失败；`repository_recovery.rs` 覆盖真实排他创建与派发 | Partial |
| 中断及目录替换不丢失保护 | `accepted_clone_never_adopts_an_existing_directory_or_replaced_root`、`clone_preserves_directory_when_creation_evidence_write_fails` 及 `ora_utils::path::identity` 覆盖已有文件、替换根与创建证据缺失；尚缺所有持久边界和叶子替换恢复组合 | Partial |

### 决策依据

执行契约 D3，不变量 1、2、3。

## Clone Checks Out The Requested Branch With Fixed Content Policy

### 风险

同名 tag 被当成分支，隐式浅克隆或初始化脚本改变承诺，外部资产被误称为已就绪。

### 前置状态

真实远端含非默认分支、分支完整历史、仅有同名 tag 的场景，以及 hooks、submodule、LFS 内容；部署辅助程序可观测。

### 触发

通过受管 Git 执行 clone 并检查仓库、命令副作用及资源访问。

### 必须成立

- 普通非 bare 仓库检出指定分支并具备其完整可达历史；不使用 shallow、partial、mirror 或共享外部对象库。
- 分支不存在明确失败，同名 tag、默认分支或 detached HEAD 不替代成功。
- 调用级禁用 hooks，不改用户持久配置；不运行项目初始化、构建或依赖安装。
- 不递归初始化 submodule，抑制 LFS smudge，不主动下载 LFS；允许指针和未初始化 submodule，公开此内容边界。
- 其他 filter／helper 遵守可信部署的明确策略，不将禁用 hooks 宣称为沙盒。

### 禁止结果

只检查退出码、默认递归获取资产、把 Git checkout 成功描述为项目运行环境已就绪。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 分支与仓库形态符合固定政策 | `managed_https_clone_selects_branch_and_replays_without_network`、`managed_https_failures_retain_targets_and_new_retry_uses_new_directory` 覆盖非默认分支、实际 commit、缺失分支及 tag-only；命令构造固定完整历史；尚缺外部 metadata 篡改组合 | Partial |
| hooks 和外部资产边界 | `private_https_uses_deployment_credentials_without_running_checkout_extensions` 使用可观测 hooks／LFS 程序、真实指针与 gitlink；尚缺部署持久配置前后逐项比较 | Partial |

### 决策依据

执行契约 D4，不变量 5。

## Clone Persists Actual Completion Before Replaying It

### 风险

分支移动导致历史结果变化，或只根据目录外观／进程退出就报告业务成功。

### 前置状态

远端分支可移动，Node 使用持久执行与结果记录；具备可独立检查的目标身份、来源、分支和 commit。

### 触发

接受后移动远端分支；执行完成后在结果持久化边界断开，重开并重传；成功后再修改本地 HEAD 或远端。

### 必须成立

- 结果保存实际获取并检出的 commit，不承诺接受时快照；包含原输入关联、Node 资源身份、实际路径和分支。
- 成功同时具备原尝试完成／收尾和有效 checkout 证据；目标归属、来源、非 bare 形态与元数据边界均核实。
- 先持久保存业务终态及待交付结果，再报告完成。
- 原身份不同输入返回冲突；已有终态重放原结果，不访问远端、重新 clone、reset 或刷新历史 commit。

### 禁止结果

接受即成功，仅凭目录或 remote 配置补成功，重传覆盖用户后续提交，Controller 将 Node 路径当本地路径。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 实际 commit 与完整成功证据 | `managed_https_clone_selects_branch_and_replays_without_network` 核实实际非默认分支 commit；`repository_commit.rs` 验证 Git 成功后的终态／outbox 回滚与离线补交；尚缺移动远端、错误来源／metadata 组合 | Partial |
| 持久完成后不可变重放 | DB 精确确认测试和 `managed_https_clone_selects_branch_and_replays_without_network` 覆盖真实结果重开、离线重传、用户编辑保留；尚缺结果交接处独立进程强杀 | Partial |

### 决策依据

执行契约 D1、D5、D6、D7，不变量 1、2。

## Clone Recovery Keeps Original Responsibility And Failed Residue

### 风险

丢回复后创建第二次 clone、旧进程仍写时重试，或删除残留来制造确定结果。

### 前置状态

执行已关联目标及 Scope／Run；可制造未派发、派发丢回复、网络失败、成功未落盘和 guardian 不可用状态。

### 触发

强杀 Node／guardian 或断连后重开，查询／重传原执行；明确失败后提交新的用户重试。

### 必须成立

- 每次变更派发前持久关联 Execution／目标／Scope／Run；丢回复查询原 Run，先核实旧 Scope 收尾再核实仓库。
- 尚未派发且前置有效时继续原执行／目标；旧尝试成功且收尾、checkout 证据完整时只补交原成功。
- 已知命令失败且收尾确认才保存失败与残留说明，不自动重跑或清理；用户重试使用新执行和新目录。
- 清理或目录／成功证据不足时保持 Unknown 与预留，查询恢复仍可用；数据库锁或超时不代替证据。
  被信号终止且收尾已确认不属于证据不足，按下一个用例以 `Interrupted` 结束。
- Unknown 不允许复用原目标；不共享可写资源的独立新目标不被一概阻塞。

### 禁止结果

续传部分下载、在残留中重新 clone、用完整外观猜测成功、强行终结 Unknown 或按超时递归删除。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 各故障边界恢复原尝试而非重复派发 | `killed_clone_recovers_original_run_without_redispatch` 使用独立 Node／host／guardian；`clone_recovers_completed_git_after_terminal_or_outbox_write_failure` 验证原唯一 Run 与离线补交；尚缺派发丢回复边界 | Partial |
| 失败残留和 Unknown 责任保持 | `repository_recovery.rs` 与 HTTPS 失败／新目录重试测试覆盖原目标保留及不重跑；尚缺 clone 专用 guardian 丢失与独立新请求组合 | Partial |

### 决策依据

执行契约 D3、D6，不变量 1、2、3；中断失败判定 D1。

## Terminated Clone Attempt Settles As Interrupted Failure

### 风险

Node 主动或被动终止的尝试被当作证据缺失，永久停留在 Unknown，调用方一直看到 `pending`；或者反过来，
收尾未确认就给出结论，在旧进程仍可能写入时解读目录。

### 前置状态

执行已派发，受管 Git 正在获取；host／guardian 可用，或被制造为不可用、无法回答。

### 触发

- Node 正常停止，Git 超过 `shutdown_grace_ms` 后被收尾；
- Git 超过 `command_timeout_ms` 后被收尾；
- 强杀 Node，guardian 按 owner 退出终止 Git，随后重启 Node 恢复；
- 以上任一情况下 host 的收尾或退出观察缺失。

### 必须成立

- Node 只在观测到原 Scope 已 Closed 之后解读原 Run 的结束事实或读取目标目录。
- 被信号终止且收尾完成时，执行以 `Interrupted` 失败结束，残留为 `Retained`，只有一个尝试；
  运行路径在本进程内落定该终态，恢复路径对同一观察得出相同结果。
- Git 在终止前已以退出码结束时，按退出码处理（0 则核实仓库），不被判为中断。
- 收尾未确认、退出结果未知、尚未启动或 host 无法回答时保持 Unknown 与预留。
- `Interrupted` 是不可变终态：原执行重传只重放它；用户重试使用新执行和新目录。
- 终态经 Controller 接管后，以 `interrupted` 交付到 Cloud 契约。

### 禁止结果

被终止的尝试停留在 Unknown、被判为成功、被自动重跑或在原目录续传；收尾未确认时给出 `Interrupted`；
删除保留目录。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 正常停止与命令超时以 `Interrupted` 结束 | `apps/ora-node/tests/standalone/repository_recovery.rs` 的 `stopped_node_settles_its_interrupted_clone_before_exit`（停止进程自身落定终态，不经恢复）与 `command_deadline_settles_clone_as_interrupted`，真实 host／guardian 与暂停的 HTTPS，检查完整终态、`Retained`、单一已清理尝试 | Covered |
| 强杀恢复以 `Interrupted` 结束 | `killed_clone_recovers_original_run_without_redispatch` 断言完整 `Interrupted` 终态与单一尝试 | Covered |
| 证据缺失仍为 Unknown | `guardian_loss_keeps_interrupted_clone_unknown` 覆盖 guardian 丢失；尚缺 host 无法回答 Run 查询的直接测试 | Partial |
| 一个 Run 只有一种结束事实，终止只支撑 `Interrupted` | `crates/node-db/src/tests/repository.rs` 的 `clone_interruption_requires_recorded_termination`、`clone_exit_code_cannot_back_an_interruption`；`version_four_upgrade_adds_termination_evidence_without_rewriting_clones` 覆盖 v4 升级不改写未完成 clone | Covered |
| 终态跨边界交付 | `apps/ora-node/tests/standalone/repository_websocket.rs` 的 `controller_session_takes_over_interrupted_clone_after_node_stop`：生产 Controller 会话经 WebSocket 接管正常停止后的完整中断终态，且不再保留待查询派发；`apps/ora-controller/src/cloud/mapping.rs` 失败码契约往返、Cloud `integration/clone_api_test.go` 公开 `interrupted` | Covered |

### 决策依据

中断失败判定 D1–D4，不变量 1–6；执行契约 D6；process integration D3。

## Clone Capability Coexists With Historical Worktree State

### 风险

第二能力被塞入 Worktree 终态，旧库被清空，或 clone-only Node 因缺少 Worktree 声明无法工作。

### 前置状态

存在旧 Node 数据库、Worktree 历史结果及用户文件；分别组合 clone-only 和保留 Worktree 能力的 Node。

### 触发

协商能力、派发 clone、重开升级后的数据库并查询旧执行；用不支持新 schema 的旧程序打开新库。

### 必须成立

- clone 有独立请求／结果及能力声明；派发按实际能力，不要求 clone-only Node 声明 Worktree。
- 共享执行查询、重放和精确确认，Controller 持久接管后确认；Client 经 Controller 观察状态。
- 保留 Worktree 历史语义，安全迁移存储，旧程序不能误读不支持的 schema；不清空旧库或用户文件。
- 不隐式切换 Backend 写入入口，不将业务结果扩大为任意 JSON。

### 禁止结果

WorktreeReady 冒充 clone 结果，重置历史身份或通过删除旧文件使升级启动成功。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 独立能力及旧结果共存 | `session::accepts_independent_and_combined_execution_capabilities`、`repository_results` 与 `worktree::completed` 覆盖能力声明、两种结果路径及历史归属；尚缺运行时派发和持久 clone 查询 | Partial |
| 文件兼容与安全升级 | `version_two_upgrade_preserves_live_process_responsibility_and_legacy_results` 与 v1 升级测试保留原执行／Run／事件；未知文件保护测试保留。旧版本门禁经静态核对，尚缺真实旧程序验收 | Partial |
| Controller 持久接管和 Client 状态 | 独立进程交接及确认丢失恢复测试；尚无 | Missing |

### 决策依据

执行契约 D7、文件兼容与风险；根决策 D2、D3。
