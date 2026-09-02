---
status: implemented
---

# MCP 插件：以静态配置包声明可物化的 Server 模板

Ora 的 MCP 插件是一个不启动插件进程的静态配置包。插件用 `orax.toml` 声明
`kind = "mcp"`，并在唯一的 `assets/config.json` 中声明一个 `stdio` 或 Streamable HTTP
transport，以及可选的用户 Settings。宿主在发现阶段编译并整包验证该声明；只有配置完整的已安装
MCP 插件才发布 secret-free `ora/mcp` Effect Source，供 Agent Consumer 在 Workspace 中物化。

该决策已经由 `ora-plugin-manifest`、`ora-plugin-config`、`ora-plugin-manager`、
`ora-plugin-lifecycle`、`ora-effect-mcp`、SQLite Effect repository 与 `crates/backend` 落地。本文固定
MCP 插件从静态包声明、安装期验证到 Effect Source 发布的系统承诺。Agent 项目配置文件、ownership
sidecar、Target reconciliation、重启协调和运行时密钥注入由
`desktop/core/effect/mcp/0-plugin-mcp-source.md` 定义，不在本决策中重复。

## 术语与边界

- **MCP 插件**是 `orax.toml` 中 `kind = "mcp"` 的 `.orax` 插件包；身份是完整 canonical Plugin ID
  `<namespace>/<identifier>`。
- **MCP Configuration** 是 `assets/config.json` 编译出的静态声明，由可选 Settings 与唯一 transport
  组成。
- **Installed MCP Descriptor** 只证明包及声明在安装时有效；它不证明用户已经填写 Settings、远端
  endpoint 可访问或任何 Agent 已加载该 MCP。
- **Configured MCP Source** 是配置完整后由宿主发布的 immutable、secret-free `ora/mcp` Effect
  Revision。
- Ora 负责声明、配置、投影与启动边界，不代理 MCP 协议流量，也不在安装阶段连接 Server 做健康检查。

## 决策

### D1：MCP kind 是无插件进程的单一静态贡献

release 与 installed 两种 `orax.toml` schema 都接受 `kind = "mcp"`。MCP kind 不允许
`[workbench]` 或 `[webview]` 等进程/页面专属 section，也不得携带 `main.js`。它不创建 Deno runtime、
不参与 plugin handshake、不能注册 host methods，也不获得插件进程权限。

一个 MCP 包必须携带且只通过 `assets/config.json` 声明 MCP contribution；manifest 不再重复枚举
transport、Settings 或 Server。`assets/config.json` 缺失、不可读、无法编译或不是 MCP shape 时，整包
不进入 installed snapshot。这样 Plugin ID/version 始终只对应一个确定的 MCP contribution，不会出现
“插件已安装但 transport 被忽略”的部分状态。

### D2：`assets/config.json` 使用封闭且互斥的版本化 shape

MCP Configuration schema version 1 的顶层结构为：

```json
{
  "schemaVersion": 1,
  "settings": {},
  "transport": {}
}
```

`settings` 可省略，`transport` 必填。配置编译器以顶层 `transport` 识别 MCP shape；没有
`transport` 和 `hook` 的文件仍是普通 Settings-only shape，同时出现 `transport` 与 `hook` 则失败。
MCP shape 只能由 `kind = "mcp"` 的包携带，其他 kind 声明 transport 时也整包失败。

顶层、Settings、transport 及引用对象均拒绝未知字段、未知 schema version 和未知 transport type。
封闭 schema 的代价是新增协议能力需要显式升级编译器或 schema，而不是让旧宿主静默忽略作者以为已生效
的安全或连接字段。

### D3：Settings 与 transport binding 共用一份声明和持久化入口

MCP 的 `settings` 成员复用普通插件配置声明的字段、校验、编辑器与 host-owned `store.json`。显式空的
`settings` 对象非法；省略 `settings` 表示该 MCP 无需用户配置。schema version 1 不接受保留的
`secret`、`file` 和 `directory` Setting type，敏感值当前仍以受宿主保护的普通标量 Setting 保存，不能
因此被复制进静态插件包或 Effect Revision。

transport 中的 Setting 引用采用：

```json
{ "setting": "apiKey", "prefix": "Bearer ", "suffix": "" }
```

每个引用必须指向同一文件中声明的 Setting；`prefix` 与 `suffix` 以及所有可绑定文本均拒绝控制字符。
数字和布尔 literal 在编译时规范化为字符串，因为参数、环境变量和 header 的最终位置都是字符串。

配置记录的 revision 是可观察身份的一部分。带 Settings 的 MCP 只有在配置状态为 `Complete` 时才能发布
Source；配置修改、清除、损坏或变为不完整时必须重新发布新 revision 或退役当前 Source。无 Settings 的
MCP 使用稳定的配置 revision `0`，安装完成即可参与发布。

### D4：transport 用枚举隔离 `stdio` 与 Streamable HTTP

MCP Configuration 必须恰好声明一种 transport：

| transport | 必需字段 | 可选字段 | 禁止混入 |
| --- | --- | --- | --- |
| `stdio` | `command` | `args`, `env` | `url`, `headers` |
| `http` | `url` | `headers` | `command`, `args`, `env` |

编译结果使用互斥枚举表达两种 shape，调用方不需要通过多个 optional 字段重新判断非法组合。

`stdio.command` 必须是 `assets/` 下至少再包含一个路径段的 portable relative path；不允许 PATH lookup、
绝对路径、父目录穿越或仅指向 `assets/` 目录。发现阶段还必须针对当前 installed package version
canonicalize 并确认目标是包内普通文件，拒绝 symlink/reparse-point escape；Unix 上同时要求 executable
mode。`args` 可使用 scalar literal、Setting 引用或唯一的 `{ "context": "workspace" }`，`env` 名称必须
匹配 portable 环境变量语法。

HTTP transport 只接受 `https` URL，并拒绝 userinfo、query string 和 fragment。header 名必须是合法
HTTP token；header 值只能是 Setting 引用，不能把 token 作为 literal 固化进不可变插件包。当前协议仅
表示 Streamable HTTP，不把旧 SSE transport 当作别名接受。

### D5：安装期验证只证明静态安全，不提前执行或探测 Server

MCP 包发现采用 fail-closed 语义：manifest kind、配置 shape、Setting 引用、transport 结构和 stdio
command containment 中任一项非法，整个最高 installed version 都产生带稳定 field path 的 discovery
issue，不回退到同一 Plugin ID 的较低版本，也不产生部分 descriptor。

验证不会执行 stdio command、发起 HTTP 请求、读取用户 Setting 实值或判断 Agent 兼容性。这样安装
过程保持确定且无外部副作用；远端不可用、运行时鉴权失败和 Agent 消费失败由 Effect/Agent 运行边界
报告，不能反过来改变 installed package 的静态有效性。

### D6：只有配置完整的 installed MCP 才发布 secret-free Effect Source

宿主在启动发现、marketplace 安装/更新、本地 `.orax` import、配置保存/清除和显式卸载后同步 MCP
Source。Source identity 由完整 Plugin ID 拥有，revision key 由插件版本与配置 revision 共同派生：

```text
<plugin-version>:<configuration-revision>
```

发布的 Effect Definition 只包含 transport 模板、稳定 Server identity 和指向 Setting 的绑定元数据，
不包含 Setting 实值。相同 revision key 的定义内容必须保持 immutable；内容变化但 key 不变视为持久化
状态损坏，而不是原地覆盖。

配置不完整、插件不再可发现、贡献 shape 改变或显式卸载时，宿主退役该 Plugin ID 当前 Source，并推进
受影响 Workspace 的完整 Desired generation。启动同步还会依据完整 installed MCP snapshot 退役不再
存在的旧 Source。具体如何合并 Agent 配置、处理同名 Server ownership、协调一次重启和验证 ready
watermark，继续由 Effect 领域决策负责。

## 为什么不是这些替代方案

| 方案 | 否决理由 |
| --- | --- |
| 让 MCP 插件启动 `main.js` 并动态注册 Server | 为静态连接声明引入 handshake、权限和进程失败面，也使配置内容无法在安装期确定验证 |
| 在 `orax.toml` 中重复声明 transport 或配置清单 | 产生 manifest 与 `assets/config.json` 两份事实来源，需要定义漂移时的优先级 |
| 允许一个包同时贡献 MCP、Hook 或普通进程能力 | Plugin ID/version 不再对应单一生命周期与权限模型，宿主可能只理解或只执行部分贡献 |
| 允许 `npx`、`uvx` 或 PATH command | 安装结果依赖宿主 PATH 与可变网络解析，命令身份、版本和 package containment 都无法由插件包证明 |
| HTTP header 接受 literal | 凭据会进入发布包、registry checksum 覆盖内容和 immutable revision，无法安全轮换或按用户配置隔离 |
| 安装时连接 endpoint 或执行 stdio command | 给无副作用的包发现引入网络、代码执行和环境依赖；短暂不可用还会错误改变 installed snapshot |
| Settings 不完整时仍发布占位 Source | Agent 项目可能出现不可启动的半配置 Server，且后续无法区分“尚未配置”与“运行失败” |

## 不变量

1. 一个有效 MCP 插件必须声明 `kind = "mcp"`、不携带 `main.js`，并拥有唯一有效的
   `assets/config.json` MCP shape。
2. 一个 MCP Configuration 必须恰好包含一个 `stdio` 或 HTTP transport；MCP、Hook 和 Settings-only
   shape 不得在同一配置文件中混合。
3. 每个 Setting 引用必须解析到同一声明中的 Setting，任何用户 Setting 实值都不得进入插件包、
   Effect Revision 或 discovery issue。
4. `stdio.command` 必须是当前 installed package `assets/` 内的普通文件；PATH lookup、路径穿越和链接
   escape 不构成有效 command。
5. HTTP endpoint 必须使用 HTTPS 且不含 userinfo、query 或 fragment；header 值必须来自 Setting 引用。
6. Installed MCP Descriptor 只证明静态包有效，不能被解释为配置完整、Server 在线或 Agent 已加载。
7. 只有无 Settings 或 Settings 完整的 installed MCP 才拥有 active Effect Source；插件、版本或配置
   revision 的变化必须通过新的 immutable revision 或 Source retirement 表达。
8. MCP 插件不创建插件 runtime、不注册 host method，也不获得插件进程权限。

## 当前未解决的问题

- MCP Configuration schema version 1 不支持 `secret`、`file`、`directory` Setting type；引入专用秘密
  类型需要同时定义存储、编辑、redaction、revision 和升级语义。
- HTTP transport 当前不支持开发模式 localhost HTTP、query 参数或旧 SSE transport；这些能力不能由
  作者通过未知字段提前启用。
- stdio command 的安装期 canonical containment 不能消除验证后文件被外部替换的 TOCTOU 风险；更强的
  内容身份或执行时重新验证需要后续决策。
- 第三方 Agent 如何声明 MCP materialization contract、不同 Agent 格式如何扩展，以及运行中热重载
  是否可用，属于 Agent Consumer 与 Effect Target 领域。

## 核心测试依据

见 `specs/test-cases/desktop/plugin/category/mcp/static-package-and-source.md`；Agent 项目物化的跨 Resource
风险另见 `specs/test-cases/desktop/core/effect/mcp/plugin-mcp-materialization.md`。
