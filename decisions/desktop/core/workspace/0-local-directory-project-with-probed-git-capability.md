---
status: implemented
---

# Desktop 项目是本机目录，Git 是按目录实时探测的可选能力，而不是项目前提

Desktop 的项目是一个本机目录：添加项目只要求目录存在，会话、Agent 就地读写和文件浏览不依赖 Git。Git 是该目录
可能具备的一项能力，由后端在每次列出 Workspace 时对目录实时探测，得到 `repository` / `notRepository` /
`unknown` 三态之一，不从存储的项目数据推断、也不缓存。只有 Git 明确回答“不在任何仓库内”且目录及其祖先都没有
`.git` 条目时才判为 `notRepository`；其余无法确定的情况一律是 `unknown`，客户端按原有行为处理。依赖 Git 的入口
据此降级：变更面板改为说明、worktree 任务入口隐藏、Git 操作在后端返回带类型的 `workspace_not_git_repository`、
定时任务表单在 `notRepository` 时提前拦截后端必然拒绝的保存。

本 ADR 为 `implemented`：行为由 ora-space/desktop#812 与 ora-space/desktop#818 实现，按 desktop `c9a76541`
核对，核心用例维护在 `test-cases/desktop/core/workspace/`。涉及的模块是 Desktop 的 workspace/project（契约
`WorkspaceDirectory`、`WorkspaceGit`，后端 Workspace 列表与 Git 审阅操作，gitlancer 的工作树探测）以及使用它的
变更面板、worktree 任务入口和定时任务表单。兼容性边界：已有 Git 项目的行为不变；`unknown`、目录缺失、远程
Workspace 和列表失败都保持原有行为。本 ADR 只覆盖 Desktop 本机目录，不决定 Cloud 是否支持目录项目（见 D8）；
探测结果从不落盘，没有需要迁移的历史状态。

## 决策

### D1：项目是本机目录，Git 不是项目的前提

Desktop 项目以一个本机目录作为主 Workspace 的位置。添加项目只校验目录存在，不要求它是 Git 仓库，也不替用户执行
`git init`。在普通目录（没有 `.git`）上，以下能力与 Git 项目相同：开启会话、Agent 在目录中直接读写文件、文件
面板浏览与引用。依赖 Git 的能力（变更审阅、worktree 任务、暂存/提交/推送、定时任务）按 D6 降级，而不是阻止
项目存在。

这样选的原因：

- 用户把文档、资料、脚本目录交给 Agent 处理是正当场景；要求先建仓库会把“能不能用 Ora”绑在一个与任务无关的
  前提上。
- 自动 `git init` 会在用户目录里留下用户没要求的 `.git`，改变其他工具（编辑器、同步盘、外层仓库）对该目录的
  判断，且无法安全地自动撤销。
- 项目存储里没有可信的“这是不是 Git 项目”标记：所有 Desktop 项目的 `repository_kind` 都固定为 `"git"`，
  不描述目录的任何实际状态；即便存储真实标记，用户也可以随时在 Ora 之外 `git init` 或删除 `.git`，标记会过期
  （见 D2）。

### D2：Git 能力在读取时按目录探测，三态表达，不存储

`listWorkspaces` 对每个本机目录存在的 Workspace 返回 `directory: { kind: "present", git }`，`git` 即
[Workspace Git 能力](../../../../domain/glossary.md#workspace-git-capabilityworkspace-git-能力)，取值：

| `git` | 含义 | 客户端行为 |
| --- | --- | --- |
| `repository` | 从该目录运行的 Git 命令作用于一个工作树（含外层 checkout 与 linked worktree） | 原有行为 |
| `notRepository` | Git 明确回答目录不在任何仓库内，且目录与所有祖先都没有 `.git` 条目 | 按 D6 降级 |
| `unknown` | Git 无法给出确定回答 | 原有行为，让操作自身的错误照常暴露 |

规则：

- 探测对象是每个列出的 Workspace（项目主 Workspace 与 worktree 任务的隔离 Workspace）在列出时刻解析出的目录；
  目录不存在时为 `{ kind: "missing", path }`，非本机 checkout 为 `{ kind: "remote" }`，二者都不携带 Git 能力。
- 每次列出都重新探测，结果不缓存、不写库；同一次列出内各 Workspace 以有限并发探测。
- 单次探测有固定时限（5 秒），超时记为 `unknown`。时限约束 Git 进程的运行与等待其输出（含 Git 退出后仍占着
  输出管道的子进程）；不约束超时 kill 之后操作系统迟迟无法回收的 Git 进程（例如卡在不可中断的文件系统等待），
  也不约束 Git 之外的文件系统检查：探测前判断目录是否存在的 `stat`，以及 Git 回答之后的路径规范化与 `.git`
  检查。
- 探测在 Workspace 目录内启动 Git 进程，因此与其他 checkout 读取者一样持有 worktree 使用租约；但它只“尝试”取得
  共享租约：清理正持有或等待该 Workspace 时不等待，直接记为 `unknown`。列表不能被一次删除阻塞，正被清理的
  checkout 也没有值得报告的能力。

为什么在读取时探测而不存储：

- 能力的真实来源是磁盘上的目录与 Git 本身。项目表里的 `repository_kind` 对所有 Desktop 项目都是 `"git"`，
  不携带目录信息；任何存储标记与磁盘状态之间也没有同步机制，据此隐藏或展示入口会在用户 `git init` 或删 `.git`
  后给出错误判断。
- 每个 Workspace 一次轻量 `rev-parse` 的代价可控，且与 D5 的操作守卫共用同一判定，客户端看到的能力与操作实际
  的行为一致。
- 隔离 Workspace 也探测，而不是按“它是 Ora 建的 worktree”跳过：跳过就是从存储数据推断，正是本决策排除的做法。

### D3：只有明确证据才判为 `notRepository`，其余无法确定的情况都是 `unknown`

探测执行一次 `git rev-parse --is-inside-work-tree`，它与其他 Git 命令一样向上发现仓库。分类如下：

| Git 的结果 | 附加检查 | 分类 |
| --- | --- | --- |
| 退出 0，输出 `true` | — | `repository` |
| 退出 0，输出 `false`（裸仓库、元数据目录） | — | `unknown` |
| 非 0 退出，stderr 含 `not a git repository (or any`（即向上发现失败的两种措辞：`… (or any of the parent directories)` 与在文件系统边界停下的 `… (or any parent up to mount point <path>)`） | 规范化后的目录及其所有祖先都没有任何 `.git` 条目 | `notRepository` |
| 同上 | 任一层存在 `.git` 条目（含损坏的目录、悬空的符号链接） | `unknown` |
| 同上 | 规范化路径或检查 `.git` 时出现 IO 错误（如无权限） | `unknown` |
| 其他非 0 退出（所有权不受信任、配置损坏、`not a git repository: <path>` 形式的断链 `.git` 文件） | — | `unknown` |
| Git 未安装、超时、输出无法解析、清理进行中 | — | `unknown` |

要点与原因：

- **为什么还要查 `.git` 条目**：Git 跳过损坏或不可读的 `.git` 目录（缺 `HEAD`、无对象、无权限）时，打印的正是
  普通目录那条消息。只看消息会把损坏的仓库说成普通目录，界面随之提示用户“这不是 Git 仓库”，诱导对一个有历史
  的仓库重新 `git init`。
- **为什么沿规范化路径查祖先**：Git 沿真实路径向上发现。通过符号链接添加的项目若按字面路径检查，会漏掉目标
  上方损坏的 `.git`。
- **嵌套在外层 checkout 内的普通子目录判为 `repository`**：从该目录运行的 diff、status、commit 实际作用于外层
  checkout，能力报告必须与操作行为一致；把它说成普通目录会隐藏实际可用、且确实会修改外层仓库的操作。
- **消息分类不受用户语言影响**：gitlancer 的所有 Git 命令默认导出 `LANG=C`，但调用方环境里的 `LC_ALL` 会压过它。
  探测命令（以及同样按 stderr 文本分类的未跟踪文件 diff）额外设置 `LC_ALL=C`；其他命令不额外设置 `LC_ALL`，
  以免覆盖用户钩子与路径处理所依赖的 `LC_CTYPE`。
- **探测判定刻意比“能否尝试操作”的判定严格**：worktree 任务创建前的仓库校验与定时任务的 Git 资格校验（解析
  基准分支、校验 active 目标）都使用 `discover_repository` 这一粗判定，把任何非 0 退出都当作“不是仓库”，因为
  它们只决定是否尝试；探测的结果会展示给用户，必须只接受 Git 的明确回答。两者不得合并。已知局限：损坏或不受
  信任的仓库在这两处得到的是“需要 Git 仓库”一类的拒绝，而不是指出仓库损坏（见“本决策未解决的问题”）。

### D4：`unknown` 与任何缺少答案的状态都保持原有行为

客户端把后端报告折算成一个可用性：

| 可用性 | 来源 | 行为 |
| --- | --- | --- |
| `checking` | 共享 Workspace 列表尚未加载 | 依赖 Git 的读取暂不发出：需要 Git 数据的视图（变更面板、worktree 对话框的分支列表）显示加载态，会话中的 diff 链接等待结果；侧栏与输入框中的新建 worktree 入口保持可见 |
| `notRepository` | 列表中目标 Workspace 为 `present` 且 `git: notRepository` | 按 D6 降级 |
| `available` | 其余一切：`repository`、`unknown`、`missing`、`remote`、目标不在列表中、列表请求失败 | 原有行为，即与 Git 项目相同：入口照常显示、请求照常发出，失败时显示操作自身的错误；本文其他处的“原有行为”均指此义 |

决定可用性的 Workspace：给定 Workspace 时取该 Workspace；按项目判断（例如“能否新建 worktree”）时取该项目的
主 Workspace，因为 worktree 从它分出。

只有确定的 `notRepository` 会改变界面。理由是不对称的代价：把可用的 Git 功能藏起来，用户无从自救；而在少数
无法确定的情况下保留原有入口，最坏只是看到操作自身的错误。

### D5：Git 审阅操作在后端统一守卫，普通目录返回 `workspace_not_git_repository`

`getWorkspaceDiff`、`getWorkspaceStatus`、`stageWorkspaceChanges`、`unstageWorkspaceChanges`、
`commitWorkspaceChanges`、`pushWorkspaceBranch` 在解析出 Workspace 的实际 checkout 目录后、检查基准提交之前、
运行操作自身的 Git 命令之前，用 D3 的同一探测判定目录：

- `notRepository`：拒绝，返回公开错误 `workspace_not_git_repository`（分类为冲突：请求合法，但与目录当前状态
  不符），不运行任何操作 Git 命令。
- `repository` / `unknown`：照常执行；`unknown` 时操作自身的失败及其诊断照常暴露。

守卫位于这六个操作共用的 checkout 解析处，是“Git 审阅需要工作树”的唯一执行点。放在基准检查之前，是为了让
没有基准的范围（例如主 Workspace 请求 `Branch`）在普通目录上也先说明“不是 Git 仓库”，而不是报告与根因无关的
`workspace_diff_baseline_unavailable`。错误码与能力字段都是 Desktop 内部契约（`PublicError` 与 Workspace 契约），
不构成跨系统协议。

后端守卫不依赖前端隐藏：列表结果可能已过期（用户刚删掉 `.git`），也可能有旧状态打开的视图；无论界面状态如何，
操作都得到同一个可解释的错误，而不是内部错误。

### D6：各依赖 Git 的入口在 `notRepository` 上的降级规则

| 入口 | `notRepository` 时 | 依据 |
| --- | --- | --- |
| 变更面板（Changes） | 显示“该目录不是 Git 仓库”的说明与重新检查按钮，保留面板工具栏以便切到文件面板；不发出 diff 与 status 查询 | 没有可审阅的 diff：Agent 直接写入目录 |
| 会话中指向 diff 的链接 | 打开文件面板中的对应文件；列表仍在 `checking` 时等待，不提前跳走 | 普通目录没有 diff 可定位 |
| 新建 worktree 任务（侧栏新建菜单、输入框分支菜单） | 入口隐藏 | worktree 需要从项目目录用 Git 分出 |
| worktree 任务对话框（被旧状态打开时） | 显示说明，不列分支、不可提交；分支列表等探测结果出来后才请求 | 避免对普通目录发出 Git 分支查询 |
| 后端创建 worktree 任务 | 以 `worktree_requires_git_repository` 拒绝 | 创建前的仓库校验使用 D3 末条的粗判定 |
| Git 审阅操作 | 后端返回 `workspace_not_git_repository` | D5 |
| 定时任务 | 后端拒绝四类保存：新建任务；把任务换到普通目录项目；未冻结基准分支（`worktreeBaseBranch` 为空的旧任务）的任何保存；保存为启用状态（含已启用任务保持启用的编辑）。已冻结基准分支、项目不变且保持停用的任务可以保存。表单只在 `notRepository` 时、只拦截上述后端必然拒绝的保存，并在项目选择旁说明原因；列表上的启用开关对普通目录项目改为打开表单 | 见下 |

定时任务的 Git 资格来自[定时任务执行归属根决策](../scheduled-task/0-local-and-cloud-execution-ownership.md)。
其 D8 写的是“保存 local 任务时检查目标项目目录存在、是 Git 仓库且模型配置可用；每次接纳时再次检查”，没有限定
active；只有其 D4 与不变量 12 把“必须以 Git 仓库为目标”限定为保存为 active 的 local 任务。现有实现允许已冻结
基准分支、保持停用的任务在普通目录上保存，这是与 D8 字面不一致的既有行为；本决策只记录它，不修改，D8 措辞的
对齐留给 scheduled-task 叶子的后续决策或 ora-space/circus#1472。

普通目录下的表现：

- 后端拒绝时返回通用的 `invalid_request`，而不是因分支解析失败报告内部错误。这也是用户在 `unknown` 时看到的
  错误：资格校验用 D3 末条的粗判定，`unknown` 的目录可能同样被拒，此时表单不拦截，后端的拒绝照常显示。是否
  改用带类型的错误码由 scheduled-task 叶子的后续决策或 ora-space/circus#1472 承接。
- 表单只在 `notRepository` 时拦截，并且只拦截后端必然拒绝的保存：新建与换项目必然解析基准分支（当前 Desktop
  项目不持久化 `default_branch`，只能从目录解析），未冻结的旧任务保存时同样要解析，保存为启用状态（含已启用任务保持启用的编辑）则要校验 active 目标。
  表单与后端规则不一致时修正表单，不放宽后端。
- 已启用且未冻结基准分支的旧任务，在普通目录上目前不能通过保存或列表开关停用：停用也走保存，会先解析分支而被
  拒。当前可用的停止路径是撤销无人值守同意（按定时任务根决策 D8，撤销即自动停用，不解析分支）或删除任务。
  停用是否应跳过分支解析，见“本决策未解决的问题”，由 ora-space/circus#1472 承接。

不受影响的入口：会话、Agent 读写、文件面板等不依赖 Git 的功能在普通目录上照常工作，不读取 Git 能力。

### D7：能力变化通过重新列出被发现，Git 拒绝会触发重新探测

后端不监听目录变化，也不推送能力变化；客户端看到新能力的时机就是 Workspace 列表被重新获取的时机：

- 共享列表的常规刷新（窗口重新聚焦且缓存已过期、Workspace 事件）；
- 变更面板说明页的“重新检查”按钮，立即重新获取列表；
- 任一 Git 审阅请求被拒为 `workspace_not_git_repository` 时，立即重新获取列表，使已打开的变更面板从“重试”切到
  说明页，而不是反复提供必然失败的重试。

这保证 `git init` 或删除 `.git` 后无需重启即可生效；代价是两次列表之间界面可能短暂沿用旧判断，而 D5 的后端
守卫保证这段时间里的操作仍得到正确的错误。

### D8：本决策只覆盖 Desktop 本机目录，不预判 Cloud 是否支持目录项目

Cloud 方向已在 ora-space/circus#1444 定为暂不支持（C），Cloud v2 仍以外部 Git 仓库为前提：项目绑定一个外部
HTTPS 仓库分支，会话目录、回合交付与删除回收都以 Git 为前提（见
[外部仓库拉取决策](../../../cloud/project-git/0-external-repository-pull-and-encrypted-token.md)）。将来有需求时
优先考虑 B（导入后在受管副本上建内部 Git 种子），届时另立决策。这里只引用该结论，不构成 Cloud 规则。

本决策的边界：

- 只约束 Desktop 本机目录的 Workspace；非本机 checkout 报告为 `remote`，不探测、不降级，保持原有行为。
- `WorkspaceGit` 三态与 `workspace_not_git_repository` 是 Desktop 内部契约，不作为 Cloud 或 Node 协议的一部分，
  也不意味着 Cloud 要采用“读取时探测”的模型；若 Cloud 将来支持目录项目，其能力来源由届时的决策定义。
- D1 反对自动 `git init` 的理由针对的是用户自己的目录，不评价 #1444 选项 B（在受管副本上建内部 Git 种子）。
- Cloud 侧决策落地后若需要 Desktop 呈现 Cloud 目录项目，应另立后续 ADR，说明它与本决策三态的关系。

## 不变量

1. 添加 Desktop 项目不要求目录是 Git 仓库，Ora 从不自动在用户目录中执行 `git init`。
2. Workspace Git 能力只来自列出时刻对磁盘目录的探测，从不读取或写入存储的项目数据。（定时任务资格按其根决策
   读取存储的分支数据，不属于 Workspace Git 能力。）
3. 只有 Git 明确回答“不在任何仓库内”且规范化目录及其所有祖先都没有 `.git` 条目时，能力才是 `notRepository`。
4. 损坏、不可读或不受信任的仓库、IO 错误、超时、Git 缺失和清理进行中都报告为 `unknown`，从不报告为 `notRepository`。
5. 嵌套在外层 checkout 内的目录报告为 `repository`，Git 审阅操作作用于外层 checkout。
6. 只有确定的 `notRepository` 改变客户端行为；`unknown`、`missing`、`remote`、列表加载中或失败都保持原有行为
   （加载中只推迟 Git 读取，不降级）。
7. Git 审阅的六个操作在 `notRepository` 目录上返回 `workspace_not_git_repository`，且不运行操作自身的 Git 命令、
   不先报告基准缺失。
8. 列表探测从不等待 worktree 清理；正在清理的 checkout 报告为 `unknown`。
9. 在 `notRepository` 项目上不提供新建 worktree 任务的入口，变更面板不发出 diff 与 status 查询。
10. 定时任务表单只在 `notRepository` 时拦截，且只拦截后端必然拒绝的保存；`unknown` 时后端的拒绝照常显示。
    active 的 local 定时任务必须以 Git 仓库为目标的规则不变。
11. 探测结果的分类不受用户语言环境影响；只有按 stderr 文本分类的命令额外设置 `LC_ALL`。

## 为什么不是这些替代方案

| 方案 | 取舍 |
| --- | --- |
| 从存储的项目数据（如 `repository_kind`、默认分支）推断能力 | 零进程开销；但 `repository_kind` 对所有 Desktop 项目都是 `"git"`，不含目录信息，任何存储标记在用户于 Ora 外 `git init` 或删 `.git` 后也会失效，违反不变量 2。隔离 Workspace 按“Ora 建的 worktree”跳过探测同理 |
| 添加普通目录时自动 `git init` | 后续所有功能都能沿用 Git 链路，实现最省；但在用户目录留下未要求的 `.git`，改变外层仓库、编辑器与同步工具的行为，且无法安全撤销，违反不变量 1 |
| 嵌套在外层 checkout 内的目录当作普通目录 | 更贴合“我添加的是这个子目录”的直觉；但 diff、commit 实际作用于外层仓库，隐藏入口不能阻止 Agent 或用户在那里使用 Git，能力报告会与操作行为相悖，违反不变量 5 |
| 任何 Git 失败都当作普通目录 | 分类最简单，界面也更“干净”；但损坏或不受信任的仓库会被告知“不是 Git 仓库”，诱导重新 `git init`，并掩盖真正需要修复的错误，违反不变量 4 |
| 为所有 Git 命令全局设置 `LC_ALL=C` | 一处设置即可保证所有消息可分类；但 `LC_ALL` 同时覆盖 `LC_CTYPE`，会影响提交、推送触发的用户钩子与非 ASCII 路径处理。只对按文本分类的命令设置，违反面最小 |
| 用 `git worktree list` 或完整的仓库发现做探测 | 能拿到更多信息（分支、worktree 关系）；但每次列表的开销更大，且列表只需要“能否在此运行审阅操作”这一位信息 |
| 缓存探测结果、监听目录变化 | 列表更快，变化可即时推送；但跨平台目录监听（含网络盘）可靠性差，缓存还需失效策略。当前以“列表时探测 + 拒绝即重探”满足需求，并由后端守卫兜底 |
| 只在前端隐藏入口，后端不守卫 | 改动更小；但过期的列表或旧视图仍会触发操作并得到内部错误，违反不变量 7 |

## 本决策未解决的问题

- **Cloud 目录项目**：ora-space/circus#1444 定为 Cloud 暂不支持。将来 Cloud 支持目录项目时（优先 B）需另立决策，
  定义能力来源与降级，必要时另立 Desktop 后续 ADR 说明呈现方式。本决策不预置任何 Cloud 协议。
- **普通目录上的变更审阅替代**：普通目录目前没有“Agent 改了什么”的审阅视图。若需要基于文件快照的审阅，涉及
  快照存储与回收，属于后续决策。
- **定时任务 Git 资格的错误类型**：后端对普通目录项目（以及粗判定失败的 `unknown` 目录）的定时任务保存以通用的
  `invalid_request` 拒绝，前端在 `notRepository` 时依据能力提前说明，不依赖错误码。是否单列带类型的错误码，由
  scheduled-task 叶子的后续决策或 ora-space/circus#1472 承接。
- **停用未冻结的已启用任务**：普通目录上，已启用且未冻结基准分支的旧任务停用会因解析分支被拒，目前只能撤销同意
  或删除。停用是否应跳过分支解析，以及定时任务根决策 D8 的措辞如何与既有行为对齐，由 ora-space/circus#1472 承接。
- **区分“不是仓库”与“仓库损坏”**：worktree 任务创建与定时任务资格使用粗判定 `discover_repository`，损坏或不受
  信任的仓库得到“需要 Git 仓库”的拒绝。是否让这两处也区分损坏的仓库，留待后续决策。

## 核心测试用例

本 ADR 的核心用例维护在 [`test-cases/desktop/core/workspace/`](../../../../test-cases/desktop/core/workspace/README.md)，
证据钉在 desktop `c9a76541`：

- [探测分类](../../../../test-cases/desktop/core/workspace/probe-classification.md)：三态分类、损坏仓库不报普通目录、
  嵌套目录、重新探测、时限、清理租约与语言环境。
- [入口降级](../../../../test-cases/desktop/core/workspace/git-entry-degradation.md)：添加普通目录项目不初始化 Git、Git 审阅操作守卫、`unknown`
  保持原有行为、变更面板、worktree 入口、拒绝后重探与定时任务表单。

非 Git 目标不能保存为 active 的义务继续由
[Scheduled Task 接纳与授权](../../../../test-cases/desktop/core/scheduled-task/admission-and-approval.md#local-activation-must-require-a-git-project-and-explicit-consent)
承担，本目录只链接，不重复登记。

## 落地顺序

1. **desktop**：ora-space/desktop#812 实现 D1–D7（契约、后端探测与守卫、gitlancer 分类、前端各入口降级与文档），
   ora-space/desktop#818 补上未冻结旧任务的表单拦截；二者合并后的 desktop `c9a76541` 是本文的核对基线。已有 Git
   项目的行为不变，普通目录项目从该版本起按本决策降级。
2. **后续**：ora-space/circus#1472 处理普通目录上未冻结已启用任务的停用，其缺口在核心用例中标为 `Missing`。
   本决策的 Desktop 本机范围不随 Cloud 方向改变。
