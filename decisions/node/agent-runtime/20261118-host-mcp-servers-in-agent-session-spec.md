---
status: implemented
---

# Controller 经 AgentSessionSpec 交付宿主 MCP，Node 在每次 ACP setup 附加

Controller 在 `start_agent_session` 的 `AgentSessionSpec.host_mcp_servers` 中交付自己提供的 HTTP MCP
Server（名称、URL 与请求头），Node 把它们原样附加到该会话的每一次 ACP setup：`session/new`、
`session/load`、生命周期刷新的 load 以及 Agent 重启。它们是
[Host Session Capability](../../../domain/glossary.md#session-mcp) 在 Node 上的承载方式，不是插件 MCP：
不进入 `mcp` 显式选择，不受插件白名单、上限和刷新协议约束，也不改变“空选择不注入插件 MCP”。URL 与
请求头只在内存中存在；Node 不执行宿主命令、不探测宿主 URL。首个使用者是工作流交互节点的完成工具：
经 Node 执行的工作流也能获得与 Desktop 相同的
[Agent 完成信号](../../desktop/core/workflow/20261110-agent-completion-signal.md)。

本决策的 Node 侧已实现并验证，状态为 `implemented`；提供宿主 Server 一方的义务（D6）随使用方接入
另行验证。影响 `node-protocol` 的会话 spec 与能力枚举、Node 会话宿主对共享
运行时 `SessionSetup::host_mcp_servers` 的实现、Node 对执行输入的持久投影，以及提供宿主 Server 的
Controller 必须承担的义务（D6）；不改变 Desktop Backend 自己注入宿主能力的路径，不改变插件 MCP 的
解析、生命周期、刷新、健康探测或失败语义，也不改变共享运行时的 setup 等待窗口。旧 Node 的
`AgentSessionSpec` 使用 `deny_unknown_fields`，因此 Controller 只向声明新能力的 Node 交付非空列表；
省略或空列表的会话与现状完全相同。不迁移历史会话，不恢复 Node 进程重启前的 Agent。完成工具本身的
语义（attempt token、信号绑定、确认与审计）由提供它的 Controller 按完成信号决策实现，不在本 ADR；
新交互节点默认使用哪种完成策略也不在本 ADR（见末节“未解决”）。

## 继承与修改

承接本目录前序[会话 MCP 刷新与宿主健康](20261116-session-mcp-refresh-and-host-health.md)，经它继承
[显式 Session MCP](20261114-explicit-session-mcp-in-agent-session-spec.md)与
[共享运行时根决策](0-shared-agent-runtime-crate-hosted-by-node.md)；宿主能力的边界继承并修改
[Agent 完成信号](../../desktop/core/workflow/20261110-agent-completion-signal.md) D1。

| 前序约定 | 处理 |
| --- | --- |
| 宿主拥有进程、会话和授权生命周期 | 继承，不变；宿主能力的授权仍由提供它的宿主（这里是 Controller）负责，Node 只负责送达 |
| 显式 Session MCP D1：`mcp` 是有序完整的插件选择，最多 16 个 Server，空列表不注入插件 MCP；版本必须与 canonical 序列化往返相等 | 继承并扩展：`host_mcp_servers` 是独立字段，不计入 16 项，不参与插件 ID 排序与唯一性；URL 必须完整匹配 D1 文法（文法保证 WHATWG 不动点，不用解析再序列化的往返比较实现） |
| 显式 Session MCP D3：Setting 值只在运行时边界存在，持久输入只留身份 | 继承并扩展：宿主 Server 的 URL 与请求头同样只在内存中，持久输入只留 Server 名称（D3） |
| 显式 Session MCP D4：Node 与 Agent 能力门禁分离，能力错误在 `initialize` 后、`session/new` 前；非法选择是帧校验失败 | 继承并扩展：非空宿主列表要求 Agent 声明 ACP HTTP MCP，缺少时以新的稳定 code 失败；非法宿主输入同样按帧校验失败关闭连接 |
| 显式 Session MCP D5：stdio 命令必须属于已安装包的 `assets/`，不信任 Controller 提供的主机命令 | 继承，不变；宿主 MCP 只接受 HTTP，没有 stdio 形态 |
| 刷新 D1/D2：`mcp_lifecycle` 会话的绑定通过 `UpdateSessionMcp` 按 generation 更新 | 继承，不变；宿主 MCP 不在更新输入里，会话内不可更新 |
| 刷新 D4/D6：刷新只 `session/load` 原 provider ID；版本变化或第九次 load 时重启本 execution 的 Agent | 修改：每一次这类 ACP setup 都附加同一份宿主 MCP 列表 |
| 刷新 D7–D9：Host MCP Health 只针对已安装插件 MCP | 继承，不变；宿主 MCP 不被 Node 探测，也不产生健康事实 |
| 共享运行时：setup 等待窗口只按插件快照是否为空选择，有插件 MCP 时用 MCP 窗口 | 继承，不变；宿主 MCP 不放宽窗口，只有宿主 MCP 的会话使用普通 setup 窗口（D2） |
| 完成信号 D1：完成工具是独立宿主能力，不进入插件白名单；授权来自节点策略与单 attempt token；`session/new` 与 `session/load` 时按节点绑定与冻结图策略重新推导并注入 | 修改：经 Node 承载时，宿主值在 start 时给定、在整个 execution 内固定，不再每次 setup 重新推导或撤下；端点与凭据在 execution 内保持有效，节点离开 `Pending`/`Running` 后工具仍然附加，失效后的处理（例如返回 `not_awaiting`）完全由提供它的 Controller 负责（D6）；可见性只是工具可见性，不是保密边界（D5） |
| 完成信号 D7：不支持工具调用的 Agent 照常运行，靠 `force` 兜底，“不做能力探测或自动降级” | 修改，只限经 Node 承载的会话：非空宿主列表遇到不声明 HTTP MCP 的 Agent 时在启动时失败（D4）；仍不自动降级（D6）。Desktop 直接承载的会话不变 |
| 共享 checkout 会话：共享的是 checkout，不是会话身份、进程或历史 | 继承，不变；宿主 MCP 不改变每个 execution 独立插件进程的承诺 |

[协议根决策](../protocol/0-controller-node-protocol.md)的身份、帧上限及能力协商继续有效；
[会话执行与命令](../protocol/20260928-streamed-thread-events-and-session-commands.md)的先持久受理、
终态与重放继续有效，“不持久化输入”的例外在显式 Session MCP 的 Setting 之外再增加本文的宿主 MCP。

## 问题与约束

Desktop Backend 直接与 Agent 进行 ACP 通信，它在 `session/new`、`session/load` 时通过共享运行时的
`SessionSetup::host_mcp_servers()` 附加自己在本机 loopback 上提供的完成工具；每个 Backend 进程随机
绑定端口、生成进程 token，靠每次 setup 重新推导恢复，Backend 重启时它启动的 Agent 一并结束。经 Node
执行的会话由 Node 与 Agent 通信，Node 的会话宿主使用这个方法的默认空实现；`AgentSessionSpec.mcp`
只能选择 Node 已安装的插件包，其 HTTP transport 的 URL 写死在包内且必须是 https。Controller 因此
没有办法把自己提供的工具交给 Agent。

把宿主能力做成插件包会把它挂到租户插件的安装、市场和 MCP 生命周期上，违背完成信号 D1“它不是插件
MCP”的边界。要求 Node 解析工作流语义也不合适：Node 只负责送达，授权和工具语义留在宿主。宿主 Server
的 URL 可能把凭据放在路径里，请求头可能带 bearer token；Node 不持久化凭据的要求与执行输入整体持久化
仍然冲突，必须照 Setting 值的规则在首次数据库写入前分离。Node 上的 Agent 可能活得比 Controller 进程
久：Controller 重启时 Running 的 execution 不会因为重发 start 而重启，因此 Desktop“每次 setup 重新
推导”的写法搬到 Node 上会让在跑的会话拿着失效的 URL。

## D1：wire 形状与能力协商

`host_mcp_servers` 是可省略的列表；省略和空列表都表示没有宿主 MCP，行为与现状完全相同，空列表不被
序列化：

```json
"host_mcp_servers": [
  {
    "name": "ora-workflow-completion",
    "url": "http://127.0.0.1:41234/mcp/3f1c…",
    "headers": { "authorization": "Bearer …" }
  }
]
```

- 每项是 JSON 对象（按位置写成数组的条目拒绝），只有 `name`、`url`、`headers`；`headers` 必填，没有
  请求头时是空对象，未知字段拒绝。每项对象与
  `headers` 内出现重复的 JSON 键一律拒绝，不按“后者覆盖”处理。
- `name`：5–64 字节，以 `ora-` 开头且其后至少一个字符，只含小写 ASCII 字母、数字和 `-`，列表内唯一，按
  UTF-8 字节序严格递增。canonical Plugin ID 总是 `<namespace>/<name>`，而宿主名不含 `/`，因此**在 ACP
  `mcpServers` 的 Server 名这一层**宿主名不会与插件 MCP 的 Server 名相同，两类 Server 并列不需要额外的冲突
  检查。Agent 从 Server 名生成工具名时可能改写 `/`，Node 不承诺工具名层面不冲突；`ora-` 前缀只降低这种
  冲突的概率。
- `url`：最多 2 KiB，必须完整匹配下面的文法（ASCII，区分大小写）：

  ```text
  url        = scheme "://" host [ ":" port ] path
  scheme     = "https" / "http"
  host       = "127.0.0.1" / "[::1]" / ipv4 / dns-name   ; http 只允许前两种
  ipv4       = 四段十进制，每段 0–255，无前导 0
  dns-name   = *( label "." ) last-label                 ; 总长最多 253 字节
  label      = 1–63 个 [a-z0-9-]，首尾不是 "-"，不以 "xn--" 开头
  last-label = label，且以 [a-z] 开头
  port       = 1–65535 的十进制，无前导 0，不等于 scheme 的默认端口（http 80，https 443）
  path       = 1*( "/" segment )                         ; 不能为空，至少是 "/"
  segment    = *( [A-Za-z0-9] / "-" / "." / "_" / "~" / "!" / "$" / "&" / "'" / "(" / ")"
                / "*" / "+" / "," / ";" / "=" / ":" / "@" / "%" HEXUP HEXUP )
  HEXUP      = [0-9A-F]
  ```

  另外，段不能是 `.` 或 `..`，路径中不能出现 `%2E`。文法之外的一切都拒绝，包括 userinfo、`?` 查询与 `#`
  fragment（与插件 MCP 的 URL 规则一致）、大写的 scheme 或主机、空路径、默认端口、带前导 0 或越界的端口、
  点段、反斜杠、制表符与换行、空格与非 ASCII 等需要编码的原始字符、非 ASCII 或 `xn--` 的 IDNA 主机、
  `127.1`、`2130706433`、`0x7f.0.0.1` 这类 IPv4 简写、`[::1]` 以外的任何 IPv6 写法（包括
  `[::ffff:127.0.0.1]` 与 `[::FFFF:7f00:1]`），以及 `localhost`、其他主机名、`127.0.0.0/8` 内其他地址上的
  `http`。符合文法的字符串都是 WHATWG URL Standard 解析再序列化的不动点，Agent 侧的 WHATWG 解析器
  看到的就是原样的 URL。Node 与 Controller 都按这条文法逐字符校验；Controller 不得用 Go `net/url` 或其他
  不符合 WHATWG 的解析器的“解析再序列化相等”来代替，因为它们对端口、空路径、点段、主机大小写和查询
  字符的处理与 WHATWG 不同。
- `headers`：每个 Server 最多 8 项；名称是 1–64 字节的小写 HTTP token；值 1 字节以上，只含可见 ASCII 与
  空格，首尾不是空格，不含 CR、LF、制表符。拒绝由 Agent 的 HTTP 与 MCP 实现负责的请求头：`host`、
  `content-length`、`content-type`、`accept`、`mcp-session-id`、`mcp-protocol-version`、`last-event-id`，以及
  逐跳头 `connection`、`keep-alive`、`proxy-connection`、`te`、`trailer`、`transfer-encoding`、`upgrade`。
- 一次会话最多 **4** 个宿主 Server；所有 Server 的 URL、请求头名称与值的 UTF-8 字节数合计最多 8 KiB。
- 全部上限都是闭区间：恰好 4 个 Server、每个 Server 恰好 8 个请求头、恰好 2 KiB 的 URL、合计恰好 8 KiB
  都被接受，超过一字节或一项即拒绝。

Node 在 `HelloAccepted.capabilities` 广告 `session_host_mcp_servers`；Controller 只向同时声明它和
`agent_session` 的 Node 发送非空 `host_mcp_servers`。Controller 必须在发送前镜像上面的全部规则；
`node-protocol` 提供一份接受与拒绝样例（覆盖每条规则、上面列出的每类拒绝写法与每个上限的边界），
Node 与 Controller 的校验测试使用同一份样例，Node 一侧还用 WHATWG 解析器验证每个接受样例都是不动点。结构、上限或格式不合法是 spec 校验错误，按
显式 Session MCP D4 的帧校验失败处理：Node 以协议错误关闭控制连接，不受理该执行，不启动 Agent。

**兼容性**：Node 能力在 Rust 的 `NodeCapability` 中是封闭枚举；凡是把能力解码成封闭集合的
Controller（不论实现语言），解码带有新能力名的 `HelloAccepted` 都会失败。因此发布顺序是 Controller 先
升级（或两端同版本发布），再部署广告 `session_host_mcp_servers` 的 Node；新 Controller 面对不广告该能力的
旧 Node 时不发送字段，行为不变。

**为什么只放宽到字面 loopback 的明文 http**：插件 MCP 的 URL 由第三方包声明，要求 https 是为了不让
包把流量引到明文端点；宿主 Server 由 Controller 自己提供，loopback 明文只在 Controller 的端点与 Agent
进程处在同一网络命名空间时有意义，此时 https 只增加一套证书分发而不提高机密性。只认两个字面地址，
和用正向文法而不是“解析器往返相等”，是因为不同语言的 URL 解析器对 IPv4 简写、整数、映射地址、端口
与路径的处理不一致，而判断不一致会关闭整条控制连接，重连后 Controller 重发同一输入又被关闭。其他情况一律要求 https，远程部署不会退化为明文。

## D2：独立于插件 MCP，每次 ACP setup 原样附加

- Node 的会话宿主实现 `SessionSetup::host_mcp_servers()`，返回由 `host_mcp_servers` 映射出的 ACP HTTP
  Server（名称、URL、请求头），顺序与输入相同，放在插件 MCP 之后。
- 该会话的每一次 ACP setup 都附加同一份列表：首次 `session/new`、`session/load`、`mcp_lifecycle` 刷新时
  的 load、刷新 D6 的 Agent 重启。插件 MCP 刷新只改变插件部分，不重建、不增删宿主部分。
- 宿主 MCP 不进入 `mcp` 选择、Effective MCP Set、Desired/Active revision 或 `GetSessionMcpState` 快照，
  不计入 16 项上限，不受插件白名单约束；“空 `mcp` 不注入任何插件 MCP”的语义不变。
- 宿主 MCP 不改变共享运行时的 setup 等待窗口：窗口仍只按插件快照选择，只有宿主 MCP 的会话使用普通
  setup 窗口。Agent 若在回答 setup 前连接宿主 Server，而宿主 URL 不可达或无响应，setup 可能按现有
  超时规则超时（运行时报 `agent_timed_out`），会话按现有启动失败规则以 `agent_failed`、`detail` 为
  `agent_start_failed` 结束，无法启动；连接被立即拒绝时，是否让 setup 失败取决于 Agent。
- 会话内不可更新：`UpdateSessionMcp` 不携带宿主 MCP，也没有单独的更新命令。宿主 Server 的值在整个
  execution 内固定（D3、D6）。将来确实需要轮换，另立决策增加专门的更新消息。

**为什么不跟插件 MCP 共用刷新协议**：宿主能力的授权来自宿主自身（例如节点冻结策略与单 attempt
token），不是可变的插件绑定；为它引入 generation、blocked 状态和查询会把宿主语义带进 Node 的插件生命周期，
而目前的使用者在会话内不需要变化。一旦附加，每次 setup 都必须带上，否则一次刷新就会让 Agent 失去工具。

**为什么不放宽 setup 窗口**：MCP 窗口是共享运行时的行为，Desktop 也用它；按宿主 Server 放宽会改变
Desktop 附加完成工具时的等待时间。宿主 Server 由 Controller 自己提供，普通窗口足够完成一次正常的握手。
宿主不可达的结果取决于 Agent：无响应时 setup 可能超时、会话无法启动；连接被立即拒绝时，Agent 可能直接
丢掉这个 Server，会话照常启动，工具悄悄缺失。Node 不区分这两种情况；建议宿主 Server 把“该 execution 第一次
停住之前一直没有收到它的 `initialize` / `tools/list`”当作宿主能力缺失的信号，由使用方处理。

## D3：URL 与请求头只在运行时边界存在

- 宿主 Server 的 URL 与请求头只经 `start_agent_session` 控制帧进入 Node 内存，并在 ACP setup 请求中交给
  Agent。不得进入 ledger（包括 SQLite WAL）、会话 JSONL、Thread、日志、错误链、关闭原因或输入对象的
  `Debug`。
- 帧解码失败也属于错误链：JSON 解码器的错误文本可能回显字符串值，Node 记录或上报 `start_agent_session`
  的解码与校验失败时最多保留错误类别和字段路径，不保留解码器的原始错误文本。实现中解码失败只记录 JSON
  错误类别，校验失败只记录被拒绝的字段。
- 宿主插件日志持久化的 Agent 自写 stderr 不在 D3 范围内：Agent 把收到的值写进自己的 stderr 时，Node 作为
  插件日志宿主照常把它写入该 Agent 的插件日志；Node 不过滤 Agent 的输出（见“风险”中的“凭据到达 Agent”），
  按值的尽力脱敏见 circus#1294。
- 持久输入只保留 Server 名称：持久投影使用独立的形状 `host_mcp_server_names`（名称字符串的列表），
  不复用线上类型，因此没有 URL 与请求头的持久行可以正常解码；空列表不被序列化，旧行与旧 Node 的
  数据库不受影响。运行值与持久投影必须在首次数据库写入前分开。
- 重复 start 只比较名称列表，不比较 URL 与请求头；重发相同 execution 时携带的值被忽略，不替换正在
  使用的值。Controller 因此不得通过重发 start 轮换端点或凭据（D6）。
- 受理后 Node 进程崩溃时不从持久投影启动 Agent，按原规则结算 `interrupted`；不要求 Controller 重放宿主
  凭据。宿主在新的 execution 中重新提供。

**为什么 URL 也算运行时值**：Desktop 的完成 Server 把进程 token 放在 URL 路径里，HTTP MCP 常见的鉴权
也可能出现在查询或路径中。只持久化名称，Node 就不必区分 URL 的哪部分是秘密。

## D4：只接受 HTTP，能力不足明确失败

- 宿主 MCP 只有 HTTP 形态，没有 stdio：Node 不执行 Controller 指定的任何本地命令（继承显式 Session MCP D5）。
- 非空宿主列表要求 Agent 在 ACP `initialize` 响应中声明 HTTP MCP 能力。缺少时在 `initialize` 之后、
  `session/new` 之前以 `agent_failed` 结束会话，`detail` 为 `host_mcp_http_capability_missing`；失败路径
  不向 Agent 发送任何宿主值。`AgentSessionEnded` 不增加字段：终态不携带宿主 Server 名、URL 或请求头，
  新 code 只出现在 `detail` 中，不进入插件 MCP 的 `SessionMcpErrorCode`。
- 插件 MCP 的检查在先：会话同时带需要 HTTP 的插件 MCP 时，Agent 缺少 HTTP 能力以原有的
  `mcp_http_capability_missing`（连同插件身份字段）失败，插件 MCP 的失败语义不变；只有宿主列表需要 HTTP
  时才报 `host_mcp_http_capability_missing`。
- 刷新 D6 重启 Agent 时，新进程的 `initialize` 同样经过这项检查；新版本 Agent 缺少 HTTP MCP 能力时，
  该 execution 以同一 code 结束，不在没有宿主工具的情况下继续。
- 宿主门禁本身不要求 Agent 的 `session/load`；同时带非空插件 MCP 时按显式 Session MCP D4 的原规则要求。
  `mcp_lifecycle` 会话的 load 要求不因只有宿主 MCP 而豁免：Agent 不支持 load 时，重连与刷新照现有规则失败。
- Node 不探测宿主 URL、不做健康检查，也不把它计入 Host MCP Health。

**为什么失败而不是静默附加**：宿主能力缺失会改变工作流的完成方式（例如节点永远等不到完成信号）。Agent
不支持 HTTP MCP 时静默附加，工具就不存在，问题只能在节点卡住后才被发现；稳定 code 让 Controller 在启动时
得到可解释的失败。

**带来的行为变化**：Desktop Backend 当前不做这项检查，`agent` 完成策略的节点在不支持 HTTP MCP 的 Agent
上仍能运行，只是收不到信号、需要用户 `force`；经 Node 承载时，同样的节点在启动时就失败。本决策不改变
Desktop 的行为，Desktop 是否对齐另行决定。

## D5：工具可见性不是保密边界

OpenCode 把 ACP 送来的 MCP 添加到进程级注册表，后续 setup 的列表缩小也不会删除早先的注册项
（[工作流显式选择](../../desktop/core/agent/session/mcp/20260912-workflow-explicit-selection.md)）。在 Node 上，
[共享 checkout 会话](20261115-shared-checkout-agent-sessions.md)只共享 checkout，每个 execution 有自己的插件
lifecycle 和运行时，宿主 Server 不会出现在兄弟会话的 Agent 工具列表里；但同一 execution 的 Agent 进程会在
它的全部轮次和刷新中一直看得见它，Desktop 共享进程中的会话也会看见别的会话注册的 Server。

注册表的说法只管**工具可见性**，不是保密边界。一个 Node 的全部工作负载使用同一个 workload UID，兄弟
会话以同一身份运行，Shared 模式下 Agent 就以 Node 自身的身份运行；loopback 端口对本机任何进程都可连，
同一身份的进程也可能读到彼此进程内的值。因此 **Node 内同一 workload UID 是同一个信任域**：Node 不承诺
宿主 Server 的 URL、请求头或工具只对附加它的会话或当前 attempt 可达，宿主 Server 必须自行对每次调用
鉴权，不能把“请求到达了这个端点”或“工具出现在某个 Agent 里”当作“调用来自该会话的当前 attempt”。

Controller 与 Agent 属于同一个系统用户时（[Cloud v2 第一版](../../cloud/lifecycle/0-single-process-cloud-on-enterprise-intranet.md)
D2 的“不隔离的后果”正是如此），整个部署是一个信任域：按 execution 签发的凭据和 attempt token 都不构成
部署内的边界，只能防止误用与跨 execution、跨 attempt 的重放。鉴权义务见 D6。

## D6：提供宿主 Server 一方的义务

义务方是提供宿主 Server 的一方，即 Controller 侧；在 Cloud v2 中 Controller 就是
[Cloud 进程](../../cloud/lifecycle/0-single-process-cloud-on-enterprise-intranet.md)本身。这些义务在 Node 上
无法验证，在 Controller 侧维护核心用例：

- **按能力派发、启动前准入**：只向声明 `session_host_mcp_servers` 与 `agent_session` 的 Node 发送非空列表，
  发送前按 D1 的文法与共用样例校验。使用者需要宿主能力（例如 `agent` 完成策略的交互节点）而目标 Node
  不具备时，拒绝派发或另选具备能力的 Node，不静默降级为不需要宿主能力的行为：完成策略来自冻结的图，
  悄悄降级会让实际执行的策略与冻结的图不一致，又没有记录。使用方必须在 WorkflowRun 启动准入时检查（冻结
  图里有 `agent` 策略的交互节点时），以有名字的原因拒绝，而不是等前面的节点跑完、用掉模型额度后再在派发
  时拒绝；只有一个可用 Node 的部署（如 Cloud v2 一个项目一个 Node）没有“另选 Node”。
- **可达性由部署配置决定**：真正发起连接的是 Agent 进程，它可能以 workload UID 运行在自己的网络环境里，
  而不是在 Node 主机的网络命名空间里。每个 Node 使用哪种 URL 由 Controller 的部署配置决定：只有 Controller
  端点与 Agent 进程处在同一网络命名空间时才能用 D1 的 loopback 明文；远程沙箱只有在 Controller 提供 Agent
  可达的 https 端点时才能使用宿主 MCP，否则这类部署不提供宿主能力，完成信号的使用者只能使用 `end_turn`。
- **先持久化再发送，重发相同字节**：端点和凭据（或推导它们的输入）在第一次发送 start 之前就已持久化，
  每次重发都携带逐字节相同的宿主列表。否则 Controller 在“已发送、未持久化”时崩溃，重启后生成新凭据，
  Node 却保留第一次的值（D3），每次调用都被拒绝。因为重发时值被忽略，Controller 也不得借重发轮换。
- **凭据只在持久 execution 记录处于非终态时有效**：宿主 Server 的凭据（URL 或请求头中的秘密）按 execution
  签发，只对这个 execution 有效，并且只在 Controller 持久的 execution 记录处于非终态时有效；判断以持久状态
  为准，不依赖内存中的吊销名单，Controller 重启后已结束 execution 的凭据不会重新可用。记录在以下任一事件
  最先发生时离开运行态：Controller 决定结束它、发出 `EndSession`、收到 `AgentSessionEnded`、Controller
  把 execution 的持久记录标为中断（例如 Cloud v2 在自身重启或与 Node 断连时中断全部 execution；协议本身不要求
  因重启或断连中断）、经 `GetExecutionStatus` 观察到 `StopUnconfirmed`。被拒绝的调用不
  改变任何状态。
- **启动时检查可恢复性**：Controller 启动时、宿主 Server 开始提供服务之前，逐个检查仍处于运行态的
  execution 的端点和凭据能否恢复；不能恢复的，走正常的结束路径并记录有名字的原因，而不是让会话拿着
  失效的 URL 一直运行。重启时结束所有受影响的 execution 同样符合本条：Cloud v2 重启即中断全部 execution
  （同一 lifecycle 决策的 D3），天然满足。
- **attempt 级授权**：完成工具一类按 attempt 授权的工具，token 只在一个 attempt 内短期有效，绑定会话
  （execution）与 attempt。它只能把重放限制在这一个 attempt 和它绑定的会话上，挡不住同一身份下读取当前
  prompt 的进程：经 Node 承载时 token 随 prompt 写入 Node 持久的会话命令输入；Shared 模式下 Agent 以 Node 自身的身份
  运行，能读到 Node 数据库；使用 workload UID 时，兄弟会话共享该 UID，能读到彼此 provider 的存储。是否接受这一点，或者要求不持久化的承载方式，由使用方的决策
  规定。开启 `autoAcceptAgentCompletion` 的节点没有人工确认兜底，一次伪造的完成信号会直接让节点完成，影响
  因此放大。

## 不变量

1. 未声明 `session_host_mcp_servers` 的 Node 不会收到宿主列表；省略或空列表时会话行为与本决策之前相同。
2. 宿主 MCP 不进入插件 MCP 的选择、Effective MCP Set、revision、上限、刷新、健康探测或 setup 等待窗口的
   选择，也不改变它们的失败语义。
3. 同一 execution 的每一次 ACP setup 附加的宿主 Server 列表相同；会话内不更新，重发 start 不替换值。
4. 宿主 Server 名不含 `/`，在 ACP Server 名层面不会与任何插件 MCP 的 Server 名相同。
5. 合法 URL 完整匹配 D1 文法，因而是 WHATWG 不动点；明文 `http` 只出现在字面 `127.0.0.1` 或 `[::1]` 上；Controller
   与 Node 对同一输入的接受与拒绝判断相同。
6. URL 与请求头只驻留内存和成功的 ACP setup 请求，不属于任何持久或诊断数据（包括解码错误文本）；持久
   投影只有 Server 名称。
7. Node 不执行宿主提供的命令，不连接、不探测宿主 URL。
8. 非空宿主列表遇到不支持 HTTP MCP 的 Agent 时，会话以 `host_mcp_http_capability_missing` 失败（插件 MCP
   的同类失败在先），失败路径不向 Agent 发送宿主值。
9. 宿主 Server 在整个 execution 内可用，否则 Controller 在提供服务前经正常结束路径结束该 execution；凭据只在
   持久 execution 记录处于非终态时有效；每次调用由宿主 Server 按 execution 与 attempt 鉴权，Node 内同一
   workload UID、以及与 Agent 同一系统用户的 Controller 部署都不构成隔离。

## 为什么不是这些替代方案

| 方案 | 优势与未采用原因 |
| --- | --- |
| 把完成工具做成内置 `.orax` MCP 包，经 `mcp` 交付 | 不改 Node 协议，复用已落地的安装与刷新。但宿主能力会依赖租户插件的安装、市场和 MCP 生命周期，违背完成信号 D1；包内 HTTP URL 写死且必须 https，只能改用包内 stdio 中转程序，离线部署还要额外分发它 |
| Agent 在回答里输出约定的标记 | 不需要任何协议改动。但要解析和剥离文本，模型可能漏写或误写，与完成信号决策否决的方案相同，且不能在 turn 中途表达 |
| Node 自己在本地提供 MCP，把工具调用经控制协议转发给 Controller | 不要求 Agent 能访问 Controller 的地址，凭据也不必跨控制协议到达 Agent。代价是新增双向的调用和结果消息、超时与重放语义，Node 还要理解宿主工具的请求和响应。现在否决；Agent 无法访问 Controller 端点的部署按 D6 不提供宿主能力、使用 `end_turn`，若这类部署需要宿主能力，再作为新的决策另行评审 |
| 增加会话内更新宿主 Server 的消息，让 Controller 重启后推送新值 | 允许 Controller 像 Desktop 一样每次推导新值。但要新增消息、顺序与重放语义，而固定端点与持久凭据、或者重启时结束受影响的 execution，已能满足目前的使用者；确有轮换需求时另立决策 |
| 允许宿主提供 stdio 命令 | 不需要 HTTP 能力。但等于让 Controller 在 Node 主机上执行任意命令，违反显式 Session MCP D5 |
| 宿主 MCP 并入 `mcp`，由 Node 区分来源 | 少一个字段。但会让插件白名单、上限、revision 和刷新协议都要区分两类成员，也让“空选择不注入插件 MCP”失去直观含义 |

## 风险、本决策未解决的问题与落地顺序

- **凭据到达 Agent**：宿主请求头会出现在 Agent 进程内，与插件 MCP 的 Setting 值相同；本决策不提供对恶意
  Agent 回显凭据的内容过滤，Agent 写到 stderr 的回显会进入 Node 持久化的插件日志（D3）。宿主按 D6
  签发只对本 execution、且只在其持久记录处于非终态时有效的凭据。
- **同一信任域**：见 D5；宿主 Server 必须自行鉴权，Node 不提供跨会话的隔离。
- **可达性**：Node 不验证 URL 可达。宿主不可达时，setup 可能超时而让会话无法启动，也可能被 Agent 丢掉而让
  工具悄悄缺失，或在启动后表现为工具调用失败，取决于 Agent 何时、如何连接；宿主应从启动结果和自身是否
  收到该 execution 的连接判断（见 D2），不期待 Node 上报。
- **未解决**：
  - Agent 无法访问 Controller 端点时的转发方案（见替代方案第三行）与宿主值的会话内轮换（替代方案第四行），
    都留给后续决策。
  - 新建的交互节点默认使用 `end_turn` 还是 `agent` 不在本 ADR：由另一份工作流决策修改
    [Agent 完成信号](../../desktop/core/workflow/20261110-agent-completion-signal.md)落地结果 3（新建或重新保存的
    交互节点写入 `completionSignal: "agent"`）以及 D7 中“升级后新建或重新保存的交互节点默认 `agent`”一条，改为
    默认 `end_turn`，见[经 Node 承载的交互节点默认 `end_turn`](../../cloud/workflow/20261119-end-turn-default-until-host-completion-tool.md)。
  - Desktop 是否也对不支持 HTTP MCP 的 Agent 做 D4 的检查，另行决定。

落地顺序（第 1、2 步已在 Node 完成）：

1. Node 实现：`node-protocol` 增加 `host_mcp_servers` 字段、`session_host_mcp_servers` 能力与 D1 的共用
   接受/拒绝样例，Node 会话宿主实现 `host_mcp_servers()`，在首次数据库写入前分离运行值与持久名称，接入
   HTTP 能力门禁。以真实 ACP 帧证明 `session/new`、`session/load`、刷新与 Agent 重启都附加同一份列表，
   并证明值不出现在持久、日志与诊断表面（包括解码失败）。
2. 能力广告与实现和测试同批交付；先发布能解码新能力的 Controller，再部署广告新能力的 Node。
3. Controller 的使用者（工作流完成工具）在声明能力的 Node 上另行接入，按 D6 交付启动准入、先持久化再发送、
   以持久状态判定的凭据有效期与启动时检查、attempt 级授权，并使用同一份文法与样例校验。

从声明 `session_host_mcp_servers` 的 Node 开始，Controller 才能为会话附加宿主 MCP；Desktop Backend 的宿主
能力路径、插件 MCP 的全部行为、共享运行时的 setup 窗口、旧 spec 与重启中断规则保持不变。

## 核心测试用例

- Node 侧见[宿主 MCP 用例](../../../test-cases/node/agent-runtime/host-mcp-servers.md)：字段与上限的规范形式
  和共用样例、能力协商、每次 ACP setup 附加同一列表且不进入插件 MCP、运行值的持久与诊断隔离（含解码失败）、
  HTTP 能力门禁与失败优先级。
- Controller 侧见[Controller 交付宿主 MCP 用例](../../../test-cases/controller/session/host-mcp-servers.md)：
  按能力派发与文法校验、端点与凭据跨 Controller 重启有效或在提供服务前结束、凭据以持久状态判定有效期、attempt 级授权。

Node 侧证据按 desktop 固定合并提交登记，受理后崩溃重启的结算与 `mcp_lifecycle` 会话的 load 要求两项为
`Partial`，其余为 `Covered`；Controller 侧的第一个使用方是
[Cloud 经 Node 宿主 MCP 提供完成工具](../../cloud/workflow/20261124-host-completion-tool-for-node-hosted-interactive-nodes.md)，
证据与缺口登记在 Controller 侧用例。
