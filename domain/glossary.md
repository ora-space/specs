# Glossary

## Session MCP

`Session MCP Selection` 是一次 Agent Session 的 MCP 选择策略。普通聊天使用 `Automatic` 自动发现；
工作流使用冻结节点配置生成 `Explicit` 完整插件 ID 集合，空集合表示不注入 MCP。

`Effective MCP Set` 是一次 Agent session setup 或 refresh 必须完整选择的 MCP 集合，先由 Session MCP
Selection 过滤，再从当前 installed snapshot 中解析静态有效且配置完整的 MCP 插件。它不表示 Agent 已
接收、Server 已连接或工具可用。

`Desired Session MCP Revision` 是 Effective MCP Set 的 secret-free 内容身份，绑定成员的 canonical
Plugin ID、精确 package version 与 Configuration revision。它用于合并变化通知并判断 Live Session
是否过期，不是持久化 ownership 或 Server 健康证据。

`Session MCP Snapshot` 是 Host 针对一个 Agent、一个 Workspace `cwd` 和一次 ACP `session/new` 或
`session/load` 解析出的有序 `mcpServers`。它是只在内存与该次 setup 帧中存在的运行输入，不是 Effect
Desired State、持久化 Session 属性或 Workspace Resource。

`Active Session MCP Revision` 是 Live Session 最近一次成功 `session/new` 或 `session/load` 接受的
Desired Session MCP Revision。停止的 Ora Session 没有 provider channel，因而没有 Active revision。

`Live Session MCP Convergence` 是在不打断当前 prompt 的前提下，通过标准 ACP `session/load` 把 Live
Session 从旧 Active revision 切换到最新 Desired revision。变化提交后开始的 prompt 必须先完成该收敛，
或在进入 Agent 前明确失败。

`Agent Session Barrier` 是 Effect Resource mutation、Agent replacement 与 MCP refresh 共用的协调
seam。它等待已经开始的 prompt 到达安全点，并阻止新 prompt 穿过未完成的变更窗口。

`Host MCP Health` 是 Host 对配置完整的已安装 MCP 执行有界握手得到的运行健康事实。它不是配置完整、
不是 Active Session MCP Revision，也不是 Agent 在会话内已经连上 Server 或工具已经可用。会话横幅只
展示该 Session Effective MCP Set 内的健康结果。

`MCP Health Probe` 是一次完整的 MCP 客户端会话：按该成员的 transport 完成 `initialize` 与
`tools/list`，然后拆除连接。它只证明 Host 视角下该绑定产物此刻可以握手。

## Plugin

插件相关术语必须区分“插件叫什么”“插件属于谁”和“插件的完整身份”。除非明确讨论某个旧版
schema 或外部字段，否则不要用 `name` 泛指这些概念。

### Plugin Manifest（插件清单）

`Manifest`（清单）是插件通过 `orax.toml` 声明自身 identifier、版本、kind、元数据与 kind 专属
内容的领域文档。清单有 release 和 installed 两种形态，共用 schema 与语义校验，但发布侧字段
的允许范围不同；清单本身不等同于插件安装包、派生索引或宿主记录的安装状态。

### Release Manifest（发布清单）

`release` 形态是 Marketplace Source 的 registry 中使用的插件清单，描述对外发布的插件版本，
可按插件 kind 的规则包含发布资产信息与 marketplace 展示声明。它不表示插件已经安装，也不必然
意味着存在可下载制品；例如 Plugin Pack 决策中的 pack 从 registry 本地创建安装态，无需发布下载制品。

### Installed Manifest（安装清单）

`installed` 形态是插件包内及安装后版本目录中使用的插件清单，自述这一份物理插件的身份、版本
与 kind 专属内容，不承载可下载 release 的语义。这里的 `installed` 指清单形态，不等同于
Installed Plugin 实体或整体安装操作的成功状态；例如 pack 已有安装清单，不代表成员编排已完成。

### Plugin Identifier

`identifier` 是插件声明的、用于标识该插件的稳定名称，也是 Plugin ID 的 name 段。它应当在同一
namespace 内保持稳定；版本升级不会改变 identifier。

`name` 不是 `identifier` 的领域同义词。历史清单或接口中仍然存在的 `name` 字段，必须根据其
实际语义明确标注为旧字段或序列化字段；新的领域模型、文档和接口优先使用 `identifier`。

`identifier` 也不是面向用户展示的名称。展示文案可以随产品或翻译变化，不参与插件身份匹配。

### Plugin Namespace

`namespace` 是插件身份的来源归属段，由宿主根据 Marketplace Source 绑定并持久化；它不是插件清单
可以自行声明的内容，也不是插件当前所在的 URL。已安装插件的 namespace 一旦绑定就不改变。

### Canonical Plugin ID

Canonical Plugin ID 是插件的完整、可持久化身份，格式为：

```text
<namespace>/<identifier>
```

例如：

```text
acme-plugins.3f9a1c2d/image-tools
```

其中 `namespace` 表示来源归属，`identifier` 表示该来源中的插件名称。只有两者组合起来，才能唯一
定位一个插件；单独的 `identifier` 不能代替 Plugin ID。版本、展示名称、当前下载 URL 和安装包文件
名都不是 Plugin ID 的组成部分。

同名插件可以来自不同 source，并因此拥有不同的 canonical Plugin ID；它们不是同一个插件。一个
source URL 的等价写法可以归一化到同一个 source 身份，但 URL 写法的变化不能改写已经落盘的
canonical Plugin ID。

## Personal Model Access

本节统一个人模型设置与 Agent 运行中的领域语言；接口和实现决策见对应 ADR。

### Model Connection（模型连接）

用户拥有的一组模型服务设置，包含一个访问入口、协议和可选择的多个模型。
_Avoid_: 把连接称为 API Key，或把每个模型都称为一条连接

### Default Model（默认模型）

用户选定的模型连接与该连接内的一个模型，供之后发起的运行使用。
_Avoid_: 空间共享默认值、平台统一模型

### Model Binding（模型绑定）

创建一次运行时固定的发起者、连接版本、凭据引用和模型。
_Avoid_: 运行中随设置变化的默认值

### Model Grant（模型授权）

允许特定运行在限定时间内使用其模型绑定的临时资格。
_Avoid_: 用户长效 API Key、登录会话

### Run Initiator（运行发起者）

触发该次 Agent 运行的用户，其个人模型连接承担本次模型调用。
_Avoid_: 项目拥有者、后来追加评论的人

## Agent Issue Run

### Agent Run（Agent 运行）

在 Issue 评论中 @ 一个 Agent 并给出任务所产生的 IssueRun（`executor_type = agent`）。它由 Cloud 编排，
拥有一个运行专属的 isolated Workspace，经过 `provisioning → starting → running → delivering → releasing → done`
阶段；执行结果（`status`）与成果保存（Revision）分别记录。

### Run Workspace（运行 Workspace）

为一次 Agent Run 创建、只属于该运行的 isolated Workspace。它由 Cloud 以系统身份建删，公开 API 不能操作；
Revision 登记后删除，释放沙盒与数据。

### Thread

一次 Agent Run 中 Agent 会话的对话记录，也是用户与 Agent 对话的入口。Thread 条目是 Ora 会话记录
（`ora-history` 的定型记录）的在线副本，另有用户轮次；会话 JSONL 是随 Revision 保存的归档。

### User Turn（用户轮次）

Thread 中交给 Agent 执行的一条用户输入，身份 `turn_id` 由 Cloud 生成。首条 prompt 由 Cloud 按 Issue
上下文渲染，后续轮次由用户在 Thread 中追加；状态为 `queued → delivered`，会话结束时未执行的为 `discarded`。

### 启动意图

平台已准备启动一次 Agent 会话的计划，尚不证明执行实例承接了它。
_Avoid_: 把准备好的首条提问称为 Agent 已启动

### 派发登记

平台已将一次执行绑定给特定实例，之后的执行结果可能尚未确认。
_Avoid_: 以没有回复或请求超时推断执行没有发生

### 运行受理

执行实例已承诺处理这次执行；它与 Agent 已产出回复是不同的事实。
_Avoid_: 将连接、登记和真实回复统称为启动成功

### Revision

一次运行结束时保存的成果：最终提交（Agent 提交之后可能追加的内部快照提交）、相对 clone 基础提交的
增量 bundle 与会话 JSONL。没有改动时也有 Revision，但没有 bundle。

### Internal Snapshot（内部快照）

Node 用临时 index、`write-tree`、`commit-tree` 为剩余未提交改动创建的提交，不运行提交钩子、不改动工作区、
暂存区与分支；它是平台的成果保存机制，不代表项目校验通过。

### Upload Grant（上传授权）

Cloud 为一次交付执行签发的、针对固定对象键和限定时长的预签名上传请求。它是短期持有者凭据，不属于执行
输入，任何一方都不持久化。

## Node Worktree

### Workspace

`Workspace` 是一个可被 Ora 绑定、定位和使用的工作上下文，具有独立的 `WorkspaceId`、
`WorkspaceKind`、生命周期和 `WorkspaceLocation`。同一个 Project 下，Main Workspace 与每个
Task 的 Isolated Workspace 都是独立的 Workspace；Git worktree 是该 Workspace 在 Node 上的
物理 checkout。当前 Main Workspace 只通过 `workspaces` 与 `workspace_locations` 表示，不在
`worktrees` 表中建立记录；`Worktree` 记录是 Isolated Workspace 的 Git 元数据扩展，不是另一个
领域身份。
_Avoid_: worktree as the domain identity, checkout path as Workspace identity

### Main Worktree

`main worktree` 是 Project 的 Main Workspace 在目标 Node 所在环境中对应的主 Git checkout。
Main Workspace 本身是一个独立的 Workspace，具有自己的 `WorkspaceId` 和 `WorkspaceLocation`；
当前实现不为它建立 `worktrees` 表记录；第一版要求该 Workspace 及其 checkout 预先存在，Ora
只以它为基准创建 task worktree。
_Avoid_: Controller checkout, local worktree, task worktree

### Task Worktree

`task worktree` 是 Task 所属的 Isolated Workspace 在目标 Node 上对应的、由 Ora 创建并管理生命
周期的隔离 Git checkout。Task Workspace 与 Main Workspace 分别拥有独立的 `WorkspaceId`、
`WorkspaceLocation` 和生命周期；删除 task worktree 不得删除或重置 main worktree。
_Avoid_: main worktree, workspace root, task checkout as a separate identity

## Node Process

本节用于区分进程运行责任与上层业务身份，决策入口见
[Node 进程运行](../decisions/node/process/README.md)。

### ProcessScope

`ProcessScope` 是共同控制、失联处置、资源关联和收尾的一组运行，其 `ScopeId` 不可复用。
它不固定等于一个 Node、Workspace、worktree 路径或业务 Execution。
_Avoid_: process group as Scope identity, workspace path as Scope identity

### ProcessRun

`ProcessRun` 是一次启动尝试及其受管后代，`RunId` 永久标识该次尝试。一个业务 Execution
可以包含多个 Run；业务重试属于新的尝试。
_Avoid_: ExecutionId as RunId, automatically restarted workload

### ProcessSession

`ProcessSession` 是 Node 实例以可验证授权访问运行宿主的会话，承载其观察资格与控制资格。
它不等于插件业务会话，也不以某条物理连接的存活定义运行是否结束。
_Avoid_: plugin session, physical connection as running state

### Process Host

`Process Host`（运行宿主）是一个逻辑 Node 的进程管理权威，拥有控制权、管理意图和恢复协调责任。
它不同于承载业务编排的 Node 实例。
_Avoid_: Node instance as Process Host, machine-wide supervisor

### Scope Guardian

`Scope Guardian` 是一个活跃 ProcessScope 的运行存续责任持有者，拥有该组运行的启动、I/O、
观察和收尾事实。它不解释 Git 结果或插件命令是否达成业务目标。
_Avoid_: legacy reaper as Scope Guardian, business result interpreter

### Privileged Process Helper

`Privileged Process Helper` 是部署授予有限平台管理权限的进程纳管辅助角色，隔离管理权限与工作负载身份。
它不拥有业务解释权，不替代 Process Host 或 Scope Guardian，也不是通用提权入口。
_Avoid_: machine-wide supervisor, arbitrary root command executor

### Scope Control Generation

`Scope Control Generation`（Scope 控制权代次）区分某个 scope 前后两次变更资格；旧代次撤销后
不再有效。它不同于授权 Node 实例更替顺序的代次，也不同于运行事实版本。
_Avoid_: Node authorization generation, fact version

### Confirmed Quiescence

`Confirmed Quiescence`（已确认静止）是在声明纳管范围内取得完整退出证据，并封闭后续启动／写入
入口的清理结果。它不证明外部服务的副作用已经结束或业务操作成功。
_Avoid_: direct process exit, business success

### Best-effort Cleanup Complete

`Best-effort Cleanup Complete`（尽力清理完成）表示承诺的跟踪与清理已经完成，仍不能排除未跟踪
残留。它不同于已确认静止，也不由重试次数或等待期限到达自动产生。
_Avoid_: Confirmed Quiescence, cleanup timeout
