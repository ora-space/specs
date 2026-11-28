# Glossary

## Workspace Collaboration

`Collaboration Space`（协作空间，产品 UI 称工作区）是用户共同组织工作的空间，当前与一个租户
一一对应；它不同于承载执行目录的[运行时 Workspace](#workspace)。

`Workspace Membership`（工作区成员资格）是一个真实用户在工作区中的有效身份、角色与状态，
是该用户进入空间的依据；它不表示已经参加某个小队。

`Space Agent`（空间 Agent）是工作区选择的 Agent 插件所提供的协作身份；它不是人类账号、
一次运行中的 Agent Session，或某位用户的私人模型配置。

`Squad`（小队）是同一工作区内人类成员与 Space Agent 的协作编组。既有协作协议的 `team`
引用指向这一身份，不表示另一个组织层级。小队不等于私有空间或已经启用的自动执行能力。

`Squad Membership`（小队成员关系）是人或 Space Agent 显式参加一个小队的关系，携带其在队中
的职责；与 Workspace Membership 分开，同一个对象可参加多个小队。

`Squad Maintainer`（小队维护者）是有权维护本队的有效人类队员；该身份不等于工作区管理员，
不提供对外邀请、私人模型或运行时权限。创建者的历史身份不等于永久维护权。

`Squad Leader`（小队协调者）是小队可选的协调 Agent。配置了协调者不表示已授权或启用自动协作，
也不将该 Agent 变为工作区管理员。

`Targeted Joining Intent`（定向加入意图）是邀请或申请中明确的工作区与小队目标集合，表达接受者
同意加入的范围；待接受意图本身不是成员资格或小队成员关系。

领域决策见[小队](../decisions/cloud/squad/0-workspace-scoped-human-agent-squads.md)与
[定向加入](../decisions/cloud/tenancy/20261027-squad-targeted-joining-and-inviter-revocation.md)，
两份决策均为 approved（尚待实现与直接验证），术语存在不表示能力已开放。

## Desktop Cloud Connection

`Desktop Token`（desktop 令牌）是 v2-server 在用户于浏览器中点「允许」后为一台 desktop 签发的不透明令牌（`ora_dt_`
前缀），以 `Authorization: Bearer` 认人，指向与网页会话相同的账号；只存在于 desktop 宿主内存和系统安全存储，只发往签发
它的 Cloud origin。任何带它的请求都算使用，连续 30 天未用即过期，没有绝对上限、不轮换。它不是网页 Cookie、IdP token 或
Agent Session。根决策与 Gateway ADR 里绑定部署身份、可轮换的 `Desktop Session`（Desktop 会话）第一版暂停，不实现。

`Desktop Login Transaction`（desktop 登录事务）是宿主拥有的一次登录：本机回环监听、随机 state、PKCE verifier 和目标
Cloud origin，5 分钟有效，同一安装同时至多一个；回调不能改变换令牌的目标。第一版不绑定部署身份。

`Authorization Code`（授权码）是 Cloud 在用户点「允许」后经浏览器送到回环地址的一次性码，2 分钟有效，绑定批准的账号、
PKCE challenge 与回调地址，只能换一次 desktop 令牌；它不是可调用 Cloud 的凭据，也不是 IdP 的 OAuth code。根决策与
Gateway ADR 称之为 `Handoff Code`（交接码）。

`Deployment Identity`（部署身份）是 Gateway 部署的稳定 deployment ID 与签名公钥身份，随部署数据迁移；不是 URL、TLS
证书或仅一个自声明 ID。相同 ID 换密钥不证明旧身份。v2-server 没有部署身份（A6 D7），desktop 第一版不校验它。

`Reported Activity`（上报活动）是正常客户端汇总的前台使用事实，用于推进空闲期限；Gateway 无法验证真伪，它不构成用户在场
证明或恶意客户端的安全边界。desktop 第一版不上报活动。

职责由 [Desktop 宿主连接根 ADR](../decisions/desktop/cloud-connection/0-host-owned-cloud-connection-and-session.md) 定义，
第一版的修改见 [Desktop Cloud 连接第一版](../decisions/desktop/cloud-connection/20261121-first-version-loopback-login-and-host-held-token.md)；
desktop 与 Cloud 之间的协议以 [登录与会话 A6](../decisions/cloud/login/0-idaas-or-email-login-with-shared-web-and-desktop-sessions.md) D6
为准，[Gateway 原生会话 ADR](../decisions/cloud/identity-access/20261019-deployment-bound-desktop-sessions.md) 定义暂停的完整版。

## Desktop Release

### Release Notes（更新日志）

一个 Desktop 正式版本面向用户的中英文变更说明，手写来源是 desktop `main` 里该版本的 `release-notes/<版本>.json`
（见[更新日志决策](../decisions/desktop/release-notes/0-release-notes-in-main-bundled-at-packaging.md)）。它不是 GitHub Release
的正文，也不是提交或 PR 列表。

### Update Manifest（更新清单）

Desktop 检查更新时读取的 `latest.json`，列出最新版本及各平台的安装包与签名。签名只覆盖安装包，不覆盖清单本身。

_Avoid_: 与 Plugin Manifest、Release Manifest（插件的清单）混用

### Yellow-Zone Build（黄区包）

同时满足下列三条的 Desktop 安装包：用 `scripts/package-desktop.ts --tag` 打包；源码与所用的 desktop 提交相比，只替换首次
安装数据（现为 `FIRST_INSTALL_SQL`，公开包为空）；只在本地交付，不上传 GitHub Release。“黄区”只指这类安装包的交付环境，
不表示该环境能否访问某个公网地址。

## Agent Conversation

`Ora Session`（Ora 会话）是 Workspace 中一条对话的稳定身份，切换 Agent 不改变该身份与历史归属。
它不同于登录用的 desktop 令牌，也不同于 Cloud IssueRun 或一次 Workflow 节点执行。

`Agent Session`（插件会话）是 Agent 插件通过 ACP 返回的 provider 会话，其 id 是不透明字符串；
定位时必须保留插件身份和所属 Ora 会话，不能将它当作全局 Ora 身份或控制资格。

`Agent Binding`（Agent 绑定）是一条 Ora 会话当前关联的完整插件引用与 Agent Session ID。
`Binding Segment`（绑定段）是这条对话中一次连续绑定的历史片段；同一插件再次绑定不合并旧段。
绑定记录时间不等于插件服务端的创建时间。

身份与诊断读取边界见 [Session 根 ADR](../decisions/desktop/core/agent/session/0-session-identity-bindings-and-diagnostics.md)
（`implemented`）。

`Session Options`（会话选项）是有 provider 的会话由 Agent 报告的全部可配置项和当前值（模型、推理强度等），每次报告都是
完整集合。它属于当前 `Agent Binding`，不写入会话历史；没有 provider 时用户的选择只是下次附着的意图，不是会话选项。规则见
[会话选项的唯一来源](../decisions/desktop/core/agent/session/20261125-session-options-single-source.md)（`approved`）。

## Session MCP

`Session MCP Selection` 是一次 Agent Session 的 MCP 选择策略。普通聊天使用 `Automatic` 自动发现；
工作流使用冻结节点配置生成 `Explicit` 完整插件 ID 集合，空集合表示不注入任何插件 MCP。host 提供的
会话能力（如 workflow 完成工具）不属于 `Session MCP Selection`，也不进入 `Effective MCP Set`；
它作为独立的宿主能力在 ACP 发送边界附加，授权来自节点策略。

`Effective MCP Set` 是一次 Agent session setup 或 refresh 必须完整选择的 MCP 集合，先由 Session MCP
Selection 过滤，再从当前 installed snapshot 中解析静态有效且配置完整的 MCP 插件。它不表示 Agent 已
接收、Server 已连接或工具可用。

`Host Session Capability` 是 Host 在 ACP 发送边界附加的会话能力（例如 workflow 完成工具）；它不属于
插件 MCP，不进入 `Explicit` 集合或 `Effective MCP Set`，授权来自节点冻结策略与仅在单个 attempt 内有效的 token。
经 Node 执行的会话由 Controller 在 `AgentSessionSpec.host_mcp_servers` 中交付这类能力的 HTTP MCP Server（能力
`session_host_mcp_servers`，见[Controller 交付宿主 MCP](../decisions/node/agent-runtime/20261118-host-mcp-servers-in-agent-session-spec.md)）；
它与 `Host MCP Health` 无关，后者只针对已安装的插件 MCP。

`Desired Session MCP Revision` 是 Effective MCP Set 的 secret-free 内容身份，绑定成员的 canonical
Plugin ID、精确 package version 与 Configuration revision。它用于合并变化通知并判断 Live Session
是否过期，不是持久化 ownership 或 Server 健康证据。

`Session MCP Snapshot` 是 Host 针对一个 Agent、一个 Workspace `cwd` 和一次 ACP `session/new` 或
`session/load` 解析出的有序 `mcpServers`。它是只在内存与该次 setup 帧中存在的运行输入，不是持久化
Session 属性，也不写入 Workspace。

`Active Session MCP Revision` 是 Live Session 最近一次成功 `session/new` 或 `session/load` 接受的
Desired Session MCP Revision。停止的 Ora Session 没有 provider channel，因而没有 Active revision。

`Live Session MCP Convergence` 是在不打断当前 prompt 的前提下，通过标准 ACP `session/load` 把 Live
Session 从旧 Active revision 切换到最新 Desired revision。变化提交后开始的 prompt 必须先完成该收敛，
或在进入 Agent 前明确失败。

`Agent Session Barrier` 是同一 Agent 上 MCP refresh 的协调 seam。它等待已经开始的 prompt 到达安全点，
并阻止新 prompt 穿过未完成的变更窗口。Skill 变化不占用它；它曾同时服务 Effect Resource mutation 与 Effect
引起的 Agent replacement，二者随 Effect 退役删除（见下文 Session Skill）。

`Host MCP Health` 是 Host 对配置完整的已安装 MCP 执行有界握手得到的运行健康事实。它不是配置完整、
不是 Active Session MCP Revision，也不是 Agent 在会话内已经连上 Server 或工具已经可用。会话横幅只
展示该 Session Effective MCP Set 内的健康结果。

`MCP Health Probe` 是一次完整的 MCP 客户端会话：按该成员的 transport 完成 `initialize` 与
`tools/list`，然后拆除连接。它只证明 Host 视角下该绑定产物此刻可以握手。

## Session Skill

Desktop 不再把 Skill 物化到 Workspace，而是按存储路径以会话清单告诉 Agent。规则见
[Skill 会话清单决策](../decisions/desktop/core/agent/session/skill/0-skill-catalog-from-storage-paths.md)（`implemented`）。
此前的 Effect（Workspace 内 Skill 目录的声明式收敛）已整体退役，相关决策保留为 `superseded` 历史记录。

`Skill Storage Location`（Skill 存储位置）是 Skill package 的权威目录：本地导入的 Skill 在应用数据目录的
`atoms/skills/<name>/`，插件提供的 Skill 在插件安装目录的 `assets/<skill>/`。它也是交付位置，Ora 不复制或
链接到 Workspace。

`Session Skill Catalog`（会话 Skill 清单）是某一时刻全部 Available Skill 的条目集合，每项只有名称、描述和
`SKILL.md` 的绝对路径，以该路径为条目身份。同名 Skill 按路径区分。它不是 Skill Catalog 本身：Unavailable 的
Catalog 行不进入清单。

`Notified Skill Catalog`（已告知清单）是 runtime 在内存中为一个 Live Session 记录的、已经告知该 Agent 的条目
集合。它不持久化；Live Session actor 新建或 Agent 报告上下文压缩后视为不存在，下一条用户 prompt 携带全量清单。

`Skill Catalog Delta`（清单增量）是当前清单与已告知清单按 `SKILL.md` 路径比较得出的新增与移除条目。路径不变的
原地更新不产生增量。

`Skill Read Roots`（Skill 可读根）是 desktop host 在 ACP `session/new` 与 `session/load` 中作为
`additionalDirectories` 交给 Agent 的稳定存储根：本地 Skill 根与插件安装根。它授予读取便利，不是安全边界。

`Legacy Managed Skill Directory`（旧受管 Skill 目录）是 Effect 曾在 Workspace 中物化、带 `.ora-managed.json`
标记并登记在 Effect 账本中的目录。升级时只删除标记与账本完全匹配的这类目录，其余内容保留。

_Avoid_: 用“物化”“同步”描述当前的 Skill 交付；用“Skill 就绪”描述发送前提。

## Agent Plugin Discovery

规则见[Agent 插件发现契约](../decisions/desktop/plugin/agent/0-discovery-contract.md)（`approved`，尚未实现）。

`Pre-session Discovery`（会话前发现）是在没有 Ora 会话 provider 时，由 Agent 插件回答一个 Workspace 中该 Agent 提供
哪些模型和命令。它不创建 Ora 会话，不写会话历史。

`Discovery Process`（发现进程）是插件为一次会话前发现启动的、独立的一次性 Agent 进程及其中的临时 ACP 会话。它不是
Ora 会话，不使用 Ora 正在使用的连接，与 Host 对 MCP 的健康探测无关。

`Agent Command`（命令）是 Agent 通过 ACP 上报的条目，按上报渠道界定。Agent 自带的 skill 和插件接入原生机制后的
Ora Skill 都可能以命令上报；界面不按名称把命令与 Ora Skill 合并。

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

### Plugin Origin（插件来源）

Plugin Origin 是插件面向用户展示的来源类别，由宿主按插件的 namespace 判定：`local` 是本地包，`official`
是官方，其余是第三方市场。插件提供的 Skill 继承所属插件的来源；本地 Skill 不属于任何插件，来源为“用户”。
第三方市场的标签由后端在查询时计算，只用于展示，可能随新增来源变化，不是身份，也不替代 namespace。定义来源是
[插件来源展示决策](../decisions/desktop/plugin/registry/20261127-plugin-origin-presentation.md)。

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

### Retired Target（退役 target）

退役 target 是 Ora 已经不再发布、但仍可能出现在已发布插件清单里的平台 target triple，当前是 `x86_64-apple-darwin`
与 `aarch64-apple-darwin`（见[支持平台决策](../decisions/desktop/platform/0-windows-and-linux-only.md) D4）。它不是未知
拼写：release 清单里的退役项被丢弃，不让整份清单失效；installed 清单自述退役 target 时被拒绝。任何宿主都不会选中
退役 target。

_Avoid_: 不支持的 target（指白名单外的未知 triple，仍是清单错误）

## Personal Model Access

本节统一个人模型设置与 Agent 运行中的领域语言；接口和实现决策见对应 ADR。

### Model Connection（模型连接）

用户拥有的一组模型服务设置，包含一个访问入口、协议和可选择的多个模型。
_Avoid_: 把连接称为 API Key，或把每个模型都称为一条连接

### Default Model（默认模型）

用户选定的全局模型连接与该连接内的一个模型，供 OpenCode 普通运行和工作流 `default` 槽位使用。
显式 Claude 的个人配置不继承或改变该选择。
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

个人 Agent 配置、冻结执行配置、模型访问上下文与 adapter 的区别见
[Agent 模型配置术语](#agent-model-configuration)。这些术语复用本节的连接和授权概念，
不另建插件级 Key 或第二份全局默认值。

## Agent Model Configuration

本节术语复用[个人模型术语](#personal-model-access)中的 Model Connection（模型连接）、Default Model、
Model Binding 和 Model Grant；“个人模型连接”是同一连接概念，不新增一种插件或空间所有的连接。

### Agent Configuration Declaration

Agent 配置声明是插件发行版本对可配置选项和模型访问能力的静态约定，不包含任何用户的配置值。
声明有效、个人配置完整、插件已安装和模型调用成功是相互独立的事实。

### Personal Agent Configuration

个人 Agent 配置是某个用户对一个完整插件身份的默认模型选择与专属选项。
它引用个人模型连接，不拥有另一份上游凭据；空间的插件安装选择不代表成员已完成个人配置。

### Frozen Agent Execution Configuration

冻结 Agent 执行配置是一次运行创建时确定的执行器、配置契约、选项和模型选择。
它包含执行器/选项与 Model Binding 的关联，描述这次运行实际选择了什么；普通配置编辑不会改变它，
它也不替代 Model Grant 的当前授权判断。

### Model Access Context

模型访问上下文是一次会话消费冻结模型选择所需的受控运行输入。它只授予该会话当前有效的模型访问，
不代表用户全部连接的访问权，也不是可再次使用的个人配置。

### Agent Model Adapter

Agent 模型 adapter 负责把平台模型访问上下文解释为某个 Agent 所需的配置与交互行为。
它不拥有个人凭据、授权范围或平台执行策略，不能由配置转换自行扩大用户授予的能力。

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

### Delivery Business Settlement（交付业务结算）

Cloud 对一次交付停止业务等待的持久结论；它不表示 Node 已受理、完成或停止，也不证明资源已释放。

### Execution Non-Acceptance Proof（执行未受理证明）

执行端对精确原执行没有受理、并已持久阻挡其未来受理的证明；普通 Unknown 查询或缺少回报不构成该证明。
责任分工见[交付执行结算与未受理证明](../decisions/cloud/issue-run/20261026-delivery-execution-settlement-and-non-acceptance.md)（`approved`）。

### Revision

一次运行结束时经平台核验保存的成果。当前 Git 交付使用指向最终提交的不可变交付 ref 与会话历史，
没有新改动也有自身可核验的 Revision；它不同于可移动的 Workspace 分支，也不表示项目校验通过。
当前契约见 [Git Revision](../decisions/cloud/revision/20261017-git-native-delivery-refs-and-resume.md)。

### Workflow History Manifest（工作流历史清单）

一份工作流运行的已建立节点会话及尝试的归档目录，保留各段历史的来源身份，不把多段原始历史
解释为同一 Thread。它不是工作流最终输出或自动摘要。定义来源是[整次成果决策](../decisions/cloud/workflow/20261025-run-owned-git-delivery-and-history.md)。

### Internal Snapshot（内部快照）

Node 用临时 index、`write-tree`、`commit-tree` 为剩余未提交改动创建的提交，不运行提交钩子、不改动工作区、
暂存区与分支；它是平台的成果保存机制，不代表项目校验通过。

### Upload Grant（上传授权）

Cloud 为一次交付执行签发的、针对固定对象键和限定时长的预签名上传请求。它是短期持有者凭据，不属于执行
输入，任何一方都不持久化。

## Scheduled Task

### Scheduled Task（定时任务）

用户创建的周期任务持久定义：把随版本冻结的 Prompt、执行目标与无人值守同意组合为一个持久定义，
执行归属（`local` 或 `cloud`）在创建时固定且不可切换。`local` 端的 future occurrence 只由 Desktop
进程内 scheduler 产生；`cloud` 端的 future occurrence 只由服务端持久调度器产生。任务定义与运行
历史的存储位置和可见范围由执行归属一端决定。

### Scheduled Occurrence（计划触发）

一次计划触发的稳定身份，由 `(taskId, scheduledAt)` 唯一标识，`scheduledAt` 是 UTC 时间点。同一
occurrence 至多接纳一个 Scheduled Run；重启、重复评估与迟到重放都复用同一身份。

### Scheduled Run（定时运行）

被接纳的一次计划触发的记录：冻结接纳时的任务 revision 与执行输入，引用由此产生的目标运行；
运行终态从目标运行的持久事实投影，调度层不复制运行状态机。

### Scheduled Run Verdict（定时运行业务结论）

一次定时运行独立于执行层状态、由执行 Agent 在收尾时通过文档化哨兵表达的机器可读业务结论
（`pass / warn / fail / blocked / idle / range_unresolved`）。它不与 `status` / `failure_reason` 互推，
`status = succeeded` 且 `verdict = fail` 可合法并存；无结论即「未报告」。
_Avoid_: execution status, failure reason

### Scheduled Run Artifact（定时运行产物）

定时运行产出、经固定目录（`.ora-scheduled-artifacts/`，Ora 保留名）与类型白名单声明后持久化的文件
（报告、测试结果、日志）。产物生命周期独立于隔离 worktree / 一次性运行 Workspace，按 Run ID 索引。
本地产物只落本机、随历史裁剪、不离开本机；Cloud 产物由 Cloud 生成对象键、在 Revision 之外按运行登记，
按运行级上限与每任务保留数约束，读取跟随 issue 可见性。产物目录不进入 Revision 的 Git 快照。
_Avoid_: worktree file, user file, Revision snapshot

### Unattended Consent（无人值守同意）

用户为定时任务授予的显式授权：在执行归属一端规定的隔离执行环境中全自动批准工具调用（`local` 为
独立 worktree——worktree 不是沙箱；`cloud` 为一次性运行 Workspace），并知晓额度消耗与执行身份
代价。策略名与同意时间随任务持久化；撤销同意自动停用任务，既有运行事实不改写。

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
只以它为基准创建 task worktree。Desktop 普通目录项目没有 main worktree，见
[Workspace Git Capability](#workspace-git-capabilityworkspace-git-能力)。
_Avoid_: Controller checkout, local worktree, task worktree

### Task Worktree

`task worktree` 是 Task 所属的 Isolated Workspace 在目标 Node 上对应的、由 Ora 创建并管理生命
周期的隔离 Git checkout。Task Workspace 与 Main Workspace 分别拥有独立的 `WorkspaceId`、
`WorkspaceLocation` 和生命周期；删除 task worktree 不得删除或重置 main worktree。
_Avoid_: main worktree, workspace root, task checkout as a separate identity

### Workspace Git Capability（Workspace Git 能力）

Desktop 列出一个目录存在的本机 Workspace 时，对该目录做一次 Git 工作树探测得到的结果，取值
`repository`（从该目录运行的 Git 命令作用于某个工作树）、`notRepository`（Git 明确回答不在任何仓库内，且目录
及其祖先都没有 `.git` 条目）或 `unknown`（无法确定，不等于否定）。它属于这次列出的 Workspace 目录，不是项目
属性，也不存储。定义来源是
[Desktop 本机目录项目决策](../decisions/desktop/core/workspace/0-local-directory-project-with-probed-git-capability.md)。
_Avoid_: repository kind, Git project flag, Desktop plain folder as a project type

## Node Process

本节用于区分进程运行责任与上层业务身份，决策入口见
[Node 进程运行](../decisions/node/process/README.md)。ProcessScope、ProcessRun、ProcessSession、Process Host、
Scope Guardian 和 Scope Control Generation 属于已被取代的 host／guardian 体系，只用于阅读历史决策；Node 自己的
Git 现在使用下一节 Node Git Execution 的术语。

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

## Node Git Execution

本节是 Node 直接执行自己的 Git 时使用的术语，决策入口见
[Node 直接执行自己的 Git](../decisions/node/git-execution/0-node-runs-git-directly-with-attempt-locks.md)。

### Attempt Lock（尝试锁）

`Attempt Lock` 是一条有副作用的 Git 命令在派发前取得的文件锁，由 git 和它的所有后代继承。它只回答"原命令的
进程树里是否还有进程"，不记录是哪些进程，也不用于终止进程。
_Avoid_: pid file, repository lock, Git index lock

### Stop Evidence（停止证据）

`Stop Evidence` 是"原命令已不再运行"的事实依据：重新对尝试锁加锁成功。它不区分从未启动和已经结束，也不说明
命令是否成功。超时、已发出终止信号或等待期限到达都不是停止证据。
_Avoid_: kill sent, timeout elapsed, exit status

### Kill Clue（终止线索）

`Kill Clue` 是派发后记下的直接子进程身份，只在 Node 重启后用来找到并终止残留进程。它的缺失或错误不改变
停止证据的结论。
_Avoid_: Stop Evidence, process attempt record

## Node Session Stop

本节是 Node 停止 Agent 会话尝试、以及保留会话前停掉一轮新起进程时使用的术语。尝试锁和终止线索沿用上一节的定义，
决策入口见[会话尝试的进程带环境标记](../decisions/node/agent-runtime/20261120-attempt-mark-in-session-stop-evidence.md)
与[保留会话前先停掉这一轮新起的进程](../decisions/node/protocol/20261123-turn-process-fence-before-keeping-session.md)。

### Attempt Environment Mark（尝试环境标记）

`Attempt Environment Mark` 是 Linux 上 Node 给一次 Agent 尝试或 stdio 探测尝试派发的每个进程设置的环境项
`ORA_SESSION_ATTEMPT=<attempt>`，后代经继承得到它；嵌套 Node 写成 `外层:attempt`。尝试登记前持久写下的
**范围记录**（尝试开始的 boot tick 与进程用户）限定它能认领的进程：只认领尝试开始后启动、带标记或父进程已被
认领的存活进程。它用来在停止时找到并杀掉脱离进程组、关闭了锁 fd 的后代，并作为停止证据的一部分（扫描干净），
不是秘密，也不构成强纳管：清空环境后成为孤儿的进程不在其内。
_Avoid_: session id, process group, secret token, cgroup

### Turn Process Fence（轮次进程栅栏）

`Turn Process Fence` 是 Linux 上一轮以取消或失败结束、Agent 已经回应之后，Node 在决定保留会话之前对这一轮的轮次范围
执行的冻结、杀与判定：复用尝试环境标记的扫描、冻结与 pidfd 杀，连续两次扫描既无范围内的被认领进程、也无范围内的待定
进程才算干净；静默窗口之后、写 `stopped` 之前，再对同一范围只扫描不杀地复查一次。栅栏或复查不干净时，会话以
`turn_processes_unstopped` 结束。它不停止尝试，不终止进程组，一轮正常完成后不运行。取消一轮时可依赖的轮次边界由
Agent 回应与这个栅栏共同构成，按（插件，版本，平台）登记。
_Avoid_: session stop, attempt stop, process group kill

### Turn Scope（轮次范围）

`Turn Scope` 是栅栏每一遍扫描时从本尝试的认领结果中收窄出的进程：被本尝试认领、启动时刻不早于这一轮开始、且所在会话
id 不属于豁免会话的进程。轮次开始是准入提交之后、prompt 写出之前在内存中记下的 boot tick，不写账本。范围对后代封闭；
认领仍用整个尝试的范围，所以范围外的父进程照样能借出它的子进程。它不同于尝试的范围记录（尝试开始时刻与进程用户）。
_Avoid_: attempt scope record, process group, all processes started during the turn

### Exempt Session（豁免会话）

`Exempt Session` 是同一遍扫描中，某个在这一轮开始前启动、此刻仍存活且被本尝试认领的进程所在的会话 id（sid）。它在
栅栏的每一遍扫描时现取，不在轮次开始时快照；在豁免会话中启动的进程（例如 Agent 懒启动的 LSP、MCP）不在轮次范围内，
栅栏不向它们发信号。这里的会话是内核的进程会话，与 Agent 会话、ACP session 无关。
_Avoid_: ACP session, agent session, session snapshot

### Turn Failure（一轮失败）

`Turn Failure` 是一个被准入的轮次因运行时先观察到的失败而结束：Agent 以 ACP 错误回应、无进展看门狗放弃、Agent 进程
退出或 ACP 连接断开，或 Ora 自己的内部错误。它记在这一轮的 `TurnEnded.failure` 中，带封闭集合里的原因码；原因来自 ACP
错误回应时另带 ACP 错误码与 Agent 报告的、经过清洗且有长度上限的文字。`stop_reason` 仍是 `cancelled`。它不同于调用方取消
与会话结束时的取消（`TurnEnded.ended_by`），也不同于 MCP 门禁中没有被准入就失败的轮次（`SessionTurnFailed`）。选择了
`keep_session_on_failure` 的会话，Node 在判定之后以 `TurnFailureSettled` 告知会话保留（`kept`）还是不会保留
（`session_ending`）；这条事件只说判定，不带原因。
_Avoid_: cancelled turn, session failure, agent_failed

## Cloud Deployment

正式 Cloud 的部署形态见 [Cloud 单进程根决策](../decisions/cloud/lifecycle/0-single-process-cloud-on-enterprise-intranet.md)。

### Interrupted Execution

`Interrupted Execution`（中断）是执行因 Cloud 重启或 Node 退出而结束的终态。
基础设施层不自动重放或续跑被中断的执行，由业务层决定重做或对账。
_Avoid_: retry_wait, parked operation, failed

### Turn Delivery

`Turn Delivery`（回合交付）是 Agent 每一轮结束时，Cloud 把工作目录相对会话起点做成的一个提交，挂在项目镜像里只新建的
交付 ref 上，并在 PostgreSQL 记一条结果为 `delivered`、`empty` 或 `failed` 的记录。变更视图、验收和 patch 下载读它。
它不是 v1 的 Revision：不经 Node 上传、不进对象存储、不承载会话历史。见
[回合交付](../decisions/cloud/project-git/20261112-single-machine-turn-delivery.md)。
_Avoid_: Revision, delivery settlement

### Workspace Reclaim

`Workspace Reclaim`（工作目录回收）是用户删除普通对话时，Cloud 在同一事务里为这个对话的工作目录登记、再按精确归属删除
这个目录的过程，状态为 `pending`、`deleting`、`reclaimed` 或 `kept`。只有成果能由运维从回合交付恢复、或用户明确丢弃时
才删除；它不删除交付、镜像、数据库记录或会话历史。见
[删除对话时回收工作目录](../decisions/cloud/project-git/20261119-reclaim-deleted-conversation-workspace.md)。
_Avoid_: workspace cleanup, garbage collection
