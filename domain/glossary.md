# Glossary

## Session MCP

`Effective MCP Set` 是一次 Agent session setup 或 refresh 必须完整选择的 MCP 集合，来源是当前
installed snapshot 中静态有效且配置完整的 MCP 插件。它不表示 Agent 已接收、Server 已连接或工具
可用。

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
