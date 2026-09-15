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
