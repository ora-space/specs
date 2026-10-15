---
status: implemented
---

# Node 非交互 clone 指定分支，独占目标目录并按原执行恢复

首版由调用方提供仓库地址和分支，Node 在显式配置的仓库根目录下分配新的独立目录，
受管执行 clone 并检出指定分支。成功返回实际路径、分支和 commit；重传读取原执行，
失败残留不自动删除或被新请求接管。

> 历史决策：D6 中“被终止的尝试保留 Unknown”已被[中断失败判定](20260925-terminated-clone-attempt-fails-as-interrupted.md)
> 收窄：被信号终止且收尾已确认的尝试判为中断失败。受理与 Git 执行的线程划分见
> [Git 副作用移出受理 worker](20260925-clone-git-effects-run-outside-the-admission-worker.md)。其余规则不变。

> 历史决策：D2 中"HTTPS 可使用部署已配置的非交互凭据来源"已被[最小 clone 执行契约 D2 的凭据来源修订](20261010-minimal-clone-execution-contract-credential-source-amendment.md)
> 扩展为"部署静态来源**或** Cloud 侧 Credential Service 按 Project 逐次授权解析，且后者只交付给凭据
> 执行身份 C"。D2 的其余每一句、以及本文其余决策都未被修订，仍然有效；该修订自身仍是 `proposed`。

本文继承[根决策](0-clone-selected-repository-branch.md)，状态为 `implemented`：实现位于 desktop 的
`crates/node-protocol`（`CloneRepositoryMessage`、`CloneExecutionResult` 与源地址校验）、`crates/node-db`（clone 执行
记录，schema v3）、`crates/gitlancer/src/git/branch_clone.rs` 与 `apps/ora-node/src/repository/`。验证见
[最小 clone 执行契约用例](../../../test-cases/node/repository/minimal-clone-execution.md)。不迁移既有 Worktree 数据，
不切换 Backend 写入入口，也不安装凭据、SSH 服务或特权 helper。

## 继承与修改

| 根决策约定 | 本文处理 |
|---|---|
| 仓库和分支显式指定，Node 实际执行 | 继承，不变；不需要已有 Main Workspace |
| Controller 协调并接管结果，Client 可观察 | 继承，不变；执行接受不等于 clone 成功 |
| 稳定身份、派发前持久责任和查询／确认 | 继承，不变；细化原执行与进程尝试的关系 |
| 目标目录尚未决定 | 细化：Node 在注入根目录下分配专用目录，不由请求提供任意绝对路径 |
| 分支移动语义尚未决定 | 细化：保存本次成功获取并检出的实际 commit，不承诺请求接受时的远端快照 |
| clone 参数和凭据尚未决定 | 细化为下述固定首版政策，不开放任意 Git 参数或凭据随请求传递 |
| 旧文件与历史执行保留 | 继承，不变；不将 WorktreeReady 重新解释为 clone 结果 |

## 背景与约束

网络获取、Git 修改文件、进程退出和 SQLite 提交不是一个原子事务。
命令可能已产生完整仓库但结果尚未保存，也可能只留下部分对象或目录。
重试同一请求不能据此重新 clone，更不能通过清空目标目录消除不确定性。

另一方面，首版不需要任意 Git 工作流。固定参数、非交互执行和独占目标可减少恢复分支，
避免为了一个 clone 能力先建设通用任务引擎、下载续传系统或资源调度器。

## 决策

### D1：输入只表达仓库获取意图

业务输入为仓库地址和一个明确的分支名，外层沿用 operation／execution／目标 Node 身份。
分支指向源仓库的分支引用，不接受 tag、任意 revision 表达式或省略后回退默认分支。
同一执行身份必须匹配原输入和目标 Node；同身份不同输入返回冲突。

首版支持 HTTPS URL 与显式 ssh:// URL。原始本地路径、file URL、scp 风格缩写、
自定义 remote helper 和其他传输暂不对业务入口开放，测试不能靠扩大生产入口替代网络边界验证。
仓库地址不能含密码／token，拒绝携带秘密的 userinfo、query 或 fragment；SSH 用户名可保留。
不开放任意 Git 选项、shell 字符串、目标路径或逐请求环境变量。
使用结构化参数调用，拒绝无效输入；不以字符串拼接执行 shell。

实现比上述规则更严：HTTPS 地址不允许任何 userinfo（含只有用户名的形式），SSH 地址可带非空、不含 `:` 的
用户名；`ssh://` 后紧跟 `/` 的无主机形式、含空白、控制字符或反斜杠的地址同样拒绝（URL 解析器会“修复”这些写法，而 Git 收到的是原文）。HTTPS 的用户名对认证没有用处，一律拒绝比逐个判断是否
含秘密更简单。

地址等价化只做明确且无损的处理，不把不同输入静默合并为一个执行。
持久保存经校验的源地址用于去重和来源检查，不在日志回显完整地址或认证诊断。

### D2：非交互访问，凭据由 Node 部署提供

Node 使用显式配置的 Git 程序、运行环境及可信凭据配置；不从 Controller 机器复制其 HOME 或密钥。
HTTPS 可使用部署已配置的非交互凭据来源；SSH 可使用部署已准备的身份和主机校验配置（`BatchMode=yes`、
`StrictHostKeyChecking=yes`）。部署没有配置 SSH 时，Node 以调用级配置 `protocol.ssh.allow=never` 禁用 SSH 传输。
凭据缺失、需要询问密码／口令或确认主机时明确失败，不弹窗、不无限等待，不自动接受未知主机。
管理的网络及认证子进程同样受运行时收尾和执行期限约束。

协议不接收秘密，Node／process journal 不复制凭据内容；可记录非秘密配置引用。
Git 输出可能包含认证敏感信息，持久诊断必须限制和脱敏，不将原始 stderr 直接存入业务结果。
是否采用具体 credential helper 或 SSH agent 属部署配置，不由业务请求指定任意可执行程序。
当前信任 Node 的管理程序，不等于仓库不需要认证，也不保证抵御恶意同 UID 代码。

### D3：每个执行使用独立、受预留保护的目标

Node 显式接收 repository_root，与 Node／host／guardian 状态目录隔离，拒绝相等或父子重叠。
目标位于该根下的 Node 分配目录；目录名不直接取仓库 URL、分支或未经编码的请求字段。
Node 先持久保存输入、目标和资源预留，再以排他方式创建目录，随后才派发 Git。
同一 Node 的 clone 管理变更串行执行，不因同源仓库而复用某个已有 checkout。

不自动使用已有目录，即使它为空或看起来是同一仓库。重复原执行只使用已保存目标；
发现目录被替换、成为符号链接、归属不明或配置根发生不兼容变化时保留冲突／未知，不覆盖。
预留记录本身不能证明任意后来出现在该路径的目录都归 Node 所有；实施须结合排他创建记录及
可核实的目录身份。提交预留与创建目录之间中断、无法证明实际创建归属时，不能删除现场。

目录可以在执行期间存在，但只有结果持久完成后才对业务发布为可使用仓库。
这不是文件系统隔离：用户或其他程序仍可能访问目录，检测到相关改变时保守处理。
首版不提供 clone 目录删除、自动残留 GC、目录迁移或跨 Node 共享分配协议。

### D4：固定获取指定分支并 checkout，不执行初始化脚本

使用普通非 bare 仓库，获取指定单分支的完整可达历史，不使用 depth、partial clone、
mirror、共享对象库或引用外部 object store。远端分支在实际获取时不存在就失败，不使用同名 tag 替代。
获取后检出该分支，不能以 detached HEAD 或仅有 bare 仓库冒充成功。
实际命令为 `git clone --no-local --reject-shallow --single-branch --no-tags --no-recurse-submodules --template=
--origin origin --branch <branch> -- <source> <target>`：不带 tag、不读本地模板、拒绝浅仓库作为来源。单分支和分支
选择的 Git 语义见 [git-clone](https://git-scm.com/docs/git-clone)。

Node 管理命令以调用级配置禁用 hooks（`core.hooksPath=/dev/null`），不修改用户仓库持久 hooks 配置；这与
[core.hooksPath 的单次调用用法](https://git-scm.com/docs/git-config#Documentation/git-config.txt-corehooksPath)一致。
不自动运行项目初始化、构建或依赖安装，不递归初始化 submodule，不主动下载 LFS 对象。
LFS smudge 必须被抑制（`GIT_LFS_SKIP_SMUDGE=1`，并以调用级配置清空 `filter.lfs.*`）；指针文件与未初始化 submodule 是本版明确允许的内容边界，
“可使用 checkout”只表示 Git 工作树可访问，不表示项目依赖或全部外部资产已经就绪。
禁用 hooks 不等于沙盒，也不能单独阻止配置的其他 Git filter／helper 执行；可信部署需明确
允许的辅助程序，不能把任意环境继承称为执行隔离。

### D5：实际获取的 commit 是结果，不承诺接受时的远端快照

接受请求只冻结源地址、分支及目标，不先把一次远端查询当作不可变快照。
本次 clone 获取并检出哪个 commit，就核实并持久保存哪个 commit；远端随后移动不更新历史结果。
若需要“请求接受时就锁定 commit”，应另行引入以明确 commit 为输入的能力，不暗中增加两阶段远端协议。

成功判定至少核实：

- 目标身份与持久预留一致，是该执行创建的有效非 bare 仓库，管理元数据没有指向未知外部目录；
- 来源配置符合原请求，当前分支是指定分支，HEAD 指向可解析 commit，checkout 已完成；
- 属于本次执行的变更 Run 已完成收尾，不存在未核实的旧写入；
- 已有的进程完成和仓库检查证据共同成立，不只依据目录存在、remote 配置或退出码。

Node 先持久保存业务终态及待交付结果，再向调用方报告完成。
成功结果包含原输入关联、Node 侧仓库资源身份、实际路径、指定分支及实际 commit。
该 commit 描述完成时的事实；后续用户提交不会被重传请求重置，也不会把历史结果改成当前 HEAD。

### D6：重传恢复原执行，未知不自动重跑或删除

每个有副作用的 Git 尝试先持久关联 Execution／目标／Scope／Run，再通过 host 派发。
连接丢失时查询原 Run；恢复先核实旧 Scope 收尾，再检查仓库。取得 Node 数据库锁不替代进程证据。

| 可核实事实 | 首版处置 |
|---|---|
| 已有持久业务终态 | 重放原结果，不重新访问远端或修改目录 |
| 尚未派发，且原目标／归属仍符合启动前条件 | 沿原执行继续，不重新分配目标 |
| 旧尝试已收尾，成功执行及完整 checkout 证据足够，但业务结果未落盘 | 按原执行补交成功；不重新 clone 或 reset |
| 已知命令失败且原尝试已收尾 | 保存失败和残留说明，保留目标记录，不自动清理重试 |
| guardian／旧进程清理证据不可用，或目录／成功事实无法核实 | 保留 Unknown 及预留，查询和恢复核实继续可用 |

仅从“目录看起来完整”不能断言成功；缺少必要完成证据时不把推测封为成功或失败。
首版不自动恢复部分对象下载、不在已有残留内重新 clone，也不自动反复创建新尝试。
明确失败后的用户重试使用新的业务执行和新目录，旧失败记录及残留保留；未知执行不能由调用方
复用原目标绕过阻塞。独立新目标可在不共享可写资源的前提下继续，不把所有未知扩大为全 Node 门禁。
残留清理、强行放弃未知责任及原目录重用须后续决策，不以超时授权递归删除。

### D7：clone 是独立能力，不伪装为 Worktree 终态

Node 协议有独立的 clone 请求（`CloneRepositoryMessage`）及结果（`CloneExecutionResult`），沿用通用执行状态、
查询、结果重放和精确确认。Worktree 历史结果和校验保留原语义，不将新字段塞进 WorktreeReady 或依赖已有
Main Workspace；Worktree 请求本身已被 Node 拒绝（根决策 D3）。

Controller 持久记录原输入、目标 Node 和执行关联，接管结果后确认；调用方通过 Controller 观察接受、进行中、
失败／未知与成功，不直接管理 host。Cloud 模式下调用方是 Cloud，clone 作为 Workspace 操作的 clone 步骤派发。
Node 目录事实不等于 Controller 本地路径。结果是有界的结构化形状，不扩大为任意 JSON。

### D8：网络 Git 采用独立且有上限的预算

2026-10-11 根据 M4 检视修正执行配置：本地 Git 检查继续使用 `command_timeout_ms`，
clone、fetch 等网络命令使用 `network_timeout_ms`，缺省十分钟，必须为正且不得超过二十分钟。
旧配置无需迁移，原命令意图决定预算，业务消息不能临时指定超时。网络慢于本地命令预算不再提前终止 clone；
达到网络预算仍通过原 process host / guardian 收尾和核实终态。Cloud 的业务重试计数不由 Node 的超时配置决定。
该修正不解除进程围栏，也不解决忽略 stdio 关闭的 Agent 后代纳管问题。

## 不变量

1. 原执行身份固定输入和目标，重传不再次 clone、不追随移动分支更新历史结果。
2. 持久接受先于业务副作用；进程收尾和仓库事实分别核实后才能完成业务结果。
3. 未知残留、用户文件或被替换目录不因恢复／重试被覆盖或自动删除。
4. 凭据不经业务消息、命令文本、日志或结果传播；认证不依赖人机交互。
5. hooks／submodule／LFS 的首版限制公开，不把 Git checkout 当作项目运行环境就绪。

## 为什么不是这些替代方案

| 方案 | 取舍 |
|---|---|
| 请求传任意目标绝对路径 | 更灵活，但引入已有用户目录接管与跨执行冲突；首版由 Node 分配 |
| 首版只支持公开 HTTPS | 认证边界更简单，但不能覆盖已有私有开发仓库；本决策复用可信部署的非交互凭据，不建设凭据产品 |
| 先查询分支 tip，再要求 clone 精确该 commit | 更接近接受时快照，但增加移动分支与远端可获取性协调；当前承诺实际获取结果 |
| 失败就删除目录重试 | 使用体验简单，但不能证明未知目录或旧运行已安全解除责任 |
| 部分 clone 自动续传与修复 | 节约传输，但需额外确定副作用和恢复边界；首版保留失败现场 |
| 浅克隆并自动拉取所有外部资源 | 初始 Git 传输较小，却增加深度、LFS／submodule 凭据与网络失败组合；本版固定单分支完整历史 |

## 文件兼容与风险

不改变现有 ora-node.sqlite3、host.sqlite、guardian.sqlite 或 Worktree 文件的含义。clone 记录经 node-db 的 schema
迁移引入，旧库按迁移升级，不靠清空旧库开始新闭环。

主要代价是完整历史可能耗时较长、失败残留占用磁盘、缺少进程证据时执行可能长期 Unknown。
运行期限、输出上限及诊断保留由部署显式给出，不能用超时制造成功或推断无副作用。
Linux BestEffort 不能排除逃逸后代；远端服务和外部程序副作用不由本地清理自动撤销。

## 本决策未解决的问题

后续协议／存储细化负责具体消息、错误分类、资源身份编码、目录创建证据、schema 迁移、
credential 配置引用与日志脱敏边界。自动清理、GC、原目录复用、按 commit 获取、
其他传输、资产拉取和交互认证另开 ADR；跨平台和 Strong 沿用既有延期边界。

## 落地

clone 专用协议、持久记录与源地址校验，受管 Git 执行与仓库事实核实，独立进程的派发、强杀与重传恢复，以及
Controller 持久接管依次落地。真实 HTTPS／SSH 与恢复路径由 `apps/ora-node/tests/standalone/` 下的独立进程测试
验证。保持不变：已有 Node／process 状态文件的含义。开始改变：Node 可以受理 clone，并以原执行身份恢复和重放结果。
