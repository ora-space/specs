---
status: approved
---

# 插件日志：Host 按已知运行时值尽力脱敏

本决策在已实现的 [插件日志根决策](0-host-owned-plugin-log.md) 之上，给宿主拥有的插件日志增加
「对已知运行时值尽力脱敏」：Host 在把 stderr 解码成记录、通过级别过滤之后，到渲染成 JSONL
之前，对记录中**插件可控的字符串字段**做替换，把一组 Host 已知的敏感明文（插件 MCP Settings
的有效值、宿主 MCP 的 URL 与请求头、模型 token、desktop Skill secret、desktop 工作流完成
工具 URL）替换为固定占位符 `[REDACTED]`。desktop 与 Node 共用同一套采集与匹配实现。这是尽力而为
的脱敏：只匹配明文子串，不解析 JSON 转义、百分号编码或 base64；跨 64 KiB 分片的超长值会部分
泄露；它不把「已脱敏」宣称为「日志绝不泄露」。

本 ADR 目前是 `approved`。实现落在 `ora-plugin-runtime`（匹配）、`ora-plugin-lifecycle`
（登记表订阅）、共享 `ora-agent-runtime`（交付点采集）与各宿主的 spawn 环境提供者，不改
Plugin Protocol 的对外字段，也不改变日志的归属、级别、队列、容量与收尾语义。

核心验证义务见 [核心用例](../../../../test-cases/desktop/plugin/logging/redact-known-runtime-values.md)。

## 继承与修改

继承 [插件日志根决策](0-host-owned-plugin-log.md) 的全部约定：日志归属独立 logs tree、单
writer、JSONL 身份与接收时间由 Host 填写、SDK 五个 console 方法共用 stderr、版本化 envelope
与 raw fallback、按 Plugin ID 的持久阈值、队列与 sink 故障隔离、收尾截止时间。本决策只修改
两处边界：

| 前序约定 | 处理 |
| --- | --- |
| 根决策开头「不承诺任意第三方输出自动脱敏」 | 修改：对 Host 已知的运行时值，改为尽力脱敏 |
| 根决策「日志可能含插件主动写出的敏感内容」，读取、导出、远端 sink、rotation/累计磁盘额度、更强进程隔离需各自后续决策（根决策「后果与边界」末段） | 本决策只承接其中的「已知运行时值脱敏」；读取、导出、远端 sink、rotation/累计磁盘额度、更强进程隔离仍各自后续决策 |

## 决策

### D1：脱敏发生在「解码之后、渲染之前」，只作用于插件可控字段

stderr 字节经 `decode → filter(level) → render JSONL → 有界队列 → 写文件`。脱敏作用在
`decode_frame` 产出记录、级别过滤通过之后、`to_json_line` 渲染之前，作用于记录中所有插件
可控的字符串：`message`、`target`、`method`、`error`（递归），以及 `context`（递归，含键名；
`context` 顶层 Host 保留键除外）。

- Host 写入的顶层字段不参与匹配：`timestamp`、`level`。`target` 只在 envelope 显式提供时才
  匹配；raw 记录的 `target`（`plugin.stderr`）和 envelope 缺省的 `plugin` 都是 Host 填的，
  不匹配，避免登记值等于 `plugin` 或 `stderr` 时改写 Host 身份。
- `context` 的**顶层**保留键不参与匹配（`RESERVED_CONTEXT_KEYS` 的 9 项：`plugin_id`、
  `host_session_id`、`generation`、`request_id`、`trace_id`、`span`、`encoding`、`fragment`、
  `format_failure`），这些由 Host 清除并拥有；嵌套对象内与保留键同名的键属于插件可控字段，
  仍参与递归脱敏。
- raw 记录（含无效 UTF-8 的转义文本）与 v1 envelope 走同一处理：JSON 转义发生在明文子串匹配
  **之后**，由 `to_json_line` 序列化器自然完成，不发生在匹配之前。
- `context` 键名被替换后若两个键同名（`serde_json::Map` 按键排序），保留字典序靠前的键，其余
  同名键丢弃，使结果与记录内的键顺序无关。
- 队列字节核算按脱敏后的渲染长度计算，不按脱敏前的行长预留。

### D2：脱敏值按 Plugin ID 登记进 Host 的实时注册表，交付前先登记

每个 Plugin ID 有一份 Host 持有的脱敏注册表，和持久日志级别一样以订阅形式进入
`PluginLogSetup`（`redactions: watch::Receiver<…>`），在 plugin-lifecycle 的 `complete_launch`
中注入。插件无法读取或改写它。

- **交付前登记**：Host 把任何敏感值交给插件进程之前，必须先把它登记进该插件的注册表并发布：
  spawn 环境（legacy token、desktop Skill 环境）、`agent/start` 的 `AgentModelContext`、ACP
  会话 setup / load / 刷新携带的 `mcpServers`（插件 MCP 与宿主 MCP）。发布完成之后才发送。
- **登记点**：采集放在 desktop 与 Node 共用的 `ora-agent-runtime`（MCP 映射 `mcp_resolver`、
  `host_mcp_servers()`、`start_agent` 的 `model_context`），外加各宿主的 spawn 环境提供者，使
  两端共用一套采集与匹配实现。`mcp_resolver` 只收集候选值（其解析函数要求无副作用，且被健康
  探测与 revision 比较复用），登记由持有接收方 Agent Plugin ID 的交付调用方完成，避免在「没有
  交付」的路径上登记。
- **只增不减**：同一进程代内，值一经登记成功就保留到该代结束；会话结束、刷新替换都不撤销旧
  值，因为 Agent 仍可能回显旧值。desktop 的一个 Agent 进程代长期服务多个会话，各会话的值不同，
  注册表就是这一代已交付给其中任何一个会话的值的并集，不会随某个会话结束而收回。新进程代从
  仍在交付的值重新登记。Node 的模型 grant 在 renew 时不换 token（spawn 环境沿用同一临时
  token），所以会话中不存在模型 token 轮换需撤销的问题。
- **每代重置**：注册表生命周期与进程代一致，代结束后整体丢弃；spawn 环境的值（legacy token、
  desktop Skill secret 等）在每次 spawn 前、由 launcher / 环境提供者按该次声明的敏感子集
  登记。注册表的键是接收这些值的 Agent 插件 ID，所以只覆盖真正收到该值的插件。每个进程代持有
  独立的匹配器订阅：旧代 stderr reader 保留本代匹配器直到读完或收尾超时被中止，新一代的重置
  与登记不影响旧代收尾。
- **有界**：注册表最多 256 条、总字节最多 64 KiB。超限或匹配器构建失败后，新值不再登记，每个
  进程代第一次记一条只含计数、不含值的 Host 告警。超限不阻塞交付，也不改变插件协议。登记顺序
  上 spawn 环境（legacy token、Skill secret）与模型 token 先于会话级 MCP Setting，同一预算下
  后到的会话级值先被拒绝。
- 更新只影响之后入队的行，不回改已落盘记录。

### D3：匹配语义与代价

只登记 `String` 类型的值，Number / Boolean 永不登记。最小长度定为 **8 个 UTF-8 字节**：低于
此长度的值误伤率高，且作为秘密本身已不可靠；短于阈值不脱敏，并列进 D5 的已知局限。登记值是
`[REDACTED]` 的子串（如 `REDACTED`、`[REDACTED`、`REDACTED]`）时也不登记，否则替换后记录里
仍残留占位符片段。

- **匹配语义**：在原字符串上一次找出所有登记值的全部出现位置（允许重叠，如 `abcdWXYZ` 与
  `WXYZefgh` 同时出现在 `abcdWXYZefgh` 中），合并相交或相邻的区间，每个合并后的区间替换为一个
  `[REDACTED]`。结果与登记顺序无关，任何登记值的任何字节都不会残留在被匹配的区间之外。
- **代价**：匹配器（如 Aho-Corasick 自动机）在注册表变化时构建一次；每条记录的匹配为
  O(记录长度 + 匹配数)，与登记值个数无关；注册表为空时不做任何工作。
- **失败语义**：只有包含该值的匹配器发布之后，值才算「登记成功」，登记成功前不算覆盖。首次
  超限或匹配器构建失败即拒绝本次新增的值，并停止该进程代后续新值登记，不将其视为登记成功；
  日志继续用上一版已成功发布的匹配器覆盖已登记值。每个进程代第一次记一条只含计数、不含值的
  Host 告警，不丢弃、不阻塞记录。
- 占位符统一为 `[REDACTED]`，与 `SessionMcpBinding` 的 `Debug` 占位一致，避免在日志里引入
  新的可识别标记。

### D4：覆盖的运行时值集合

| 值 | 来源与登记时点 |
| --- | --- |
| 插件 MCP Settings 的 `String` 有效值 | 共享 `mcp_resolver` 从本次实际参与传输绑定（stdio `args`/`env`、HTTP `headers`）的 `EligibleMcpMember.values` 采集拼接前后缀之前的原始 `SettingValue::String` 有效值，在携带 `mcpServers` 的 ACP `session/new` / `session/load` / 刷新发出前登记 |
| 宿主 MCP 的 `url` 与每个 `headers` 值 | Node 的 `HostMcpServer`（`NodeSessionMcp` / `HostServers` 输入的 `url` 与 `headers`），以及 desktop Backend 交付的宿主 HTTP MCP Server 及其工作流完成工具 URL，在对应 ACP setup / load / 刷新发出前登记 |
| 模型 token | `AgentModelContext.token`（adapter 路由）与 legacy 环境的 `ORA_MODEL_ACCESS_TOKEN` |
| desktop Skill Settings 中 `secret` 为真的 `String` 值 | 在 `SkillEnvironment::capture` 阶段由它保留 `secret` 子集的原始 `String` 值（含默认值；命名冲突被拒绝的包不注入也就不登记），spawn 环境提供者只按接收方 Agent 插件 ID 登记 |

**不登记 `proxy_base_url`**：它是网关主机加固定路径（`/runtime/openai/v1`、`/runtime/anthropic/v1`），
同一部署的所有会话都相同，且 [模型 adapter 决策](../../../node/agent-runtime/20261017-plugin-owned-model-adapters.md)
已写明「公开 CA 和固定代理地址不是秘密」。登记它只会抹掉 Agent 的正常网络诊断；模型 token 才
是应脱敏的秘密。

**不登记 `attemptToken`**：desktop 在每次发送（含重试）时撤销旧 token、签发新 UUID，而 Agent
进程代跨会话长期存活；在 D2 的「只增不减 + 256 条上限」下，注册表会被持续轮换的 token 占满，
导致之后的新 Setting 值不再登记。已撤销的 token 泄露无影响，所以不登记它，只保留稳定交付的
完成工具 URL。

desktop 路径没有 Host 注入的模型 token，但有 Skill secret 环境、会话级插件 MCP Settings 与
工作流完成工具 URL；Node 路径有插件 MCP Settings、宿主 MCP URL 与请求头、模型 token。为什么
MCP 只登记**参与传输绑定的**全部 `String` 有效值、Skill 只登记 `secret` 值：没参与绑定的
Setting 不会经 `mcpServers` 交付给插件，登记它只会白占 256 条 / 64 KiB 的预算（与 Node 一次
setup 的 `MAX_SESSION_MCP_SETTINGS_BYTES` 同量级，desktop 还是多会话并集，一次 setup 就可能
吃满）；参与绑定的 `String` 值 Host 无从判断哪些会被插件视为敏感，全部登记才能覆盖。Skill
值量大且多为非敏感配置，按 `secret` 标记只登记真正需要脱敏的子集。

### D5：尽力而为的边界与已知局限

脱敏是尽力而为，只覆盖 Host 拥有的 Plugin log（`<home>/plugins/logs/<namespace>/<name>/plugin.log`
及其 D10 恢复文件）；Ora runtime log（含 debug 构建的 TRACE）、会话 JSONL、ACP 消息不在本
决策范围。明确以下不覆盖的情形：

- raw 行里以字面转义序列（`\u0041` 等）、百分号编码（`%41`）、base64 出现的值不覆盖；envelope
  的 JSON 转义在 `parse_envelope` 解码时已由 `serde_json::from_str` 还原，不在本条边界内。
- 值跨 64 KiB 分片边界时，两侧分片各自保留该值的一部分明文（部分泄露，不是整体绕过）。
- 宿主 MCP 只打印 token（去掉 `Bearer ` 前缀的凭据部分）或经 URL 解析器规范化后的 URL（加
  尾斜杠、host 小写）会漏，因为 D4 只登记完整 `url` / header 值，不登记派生形式。
- 短于 8 字节的秘密、`Number`/`Boolean` 类型的值不脱敏（后者永不登记，D3）。
- 超限或匹配器构建失败之后登记的新值不脱敏（D2/D3 的失败语义）。
- 转义字节的 raw 记录里，任何含 `\` 的值都匹配不上（渲染前整条消息里的所有 `\` 都被加倍成
  `\\`）；匹配还可能落在 `\xNN` 转义中间，破坏可追溯编码。这两类都只影响 raw 记录的局部，
  不改变记录结构。
- Agent 写到自己能访问的其他文件（非 Host 拥有的 Plugin log）的不归本决策管；插件不经过
  Host 管道、直接自行写文件的不归本决策管。

## 不变量

1. 脱敏只改写 `decode_frame` 之后、`to_json_line` 之前记录中插件可控的字段，不改写帧、解码
   计数与过滤结果。
2. 脱敏值由 Host 登记，插件不能读取或覆盖；注册表为空时管道行为与未实现脱敏时完全一致。
3. 脱敏不改写 Host 写入的字段、记录条数与入队顺序；队列字节核算按脱敏后的渲染长度计算；
   脱敏后单条记录长度增长有上界（由最小长度与占位符长度决定）。
4. 匹配器构建失败或注册表超限时，本次新增的值不算登记成功，并停止该进程代后续新值登记，日志
   继续用上一版已成功发布的匹配器；每个进程代第一次记一条 Host 告警，不丢弃、不阻塞记录，
   不改变记录数量。
5. 脱敏是尽力而为的明文子串替换，不承诺覆盖 JSON 转义、百分号编码、base64 或跨分片值。
6. 对任一登记成功的值 S（满足登记规则），Host 把 S 交付给插件之后，该插件进程代写到 stderr
   的任何包含 S 的完整帧（raw 行或 v1 envelope 的任何插件可控字段），落盘到该 Plugin ID 日志
   树记录的**插件可控字段**中都不含 S；Host 写入字段按不变量 8 逐字节保留，D5 所列跨 64 KiB
   分片、`escaped-bytes` 含 `\` 的边界除外。
7. 在第 6 条中，S 覆盖：Node 路径的插件 MCP Setting（参与传输绑定的，含刷新后新值）、宿主
   MCP URL 与请求头、模型 token（adapter token 与 legacy 环境 token）；desktop 路径的插件
   MCP Setting（参与传输绑定的）、Skill secret、工作流完成工具 URL。
8. 脱敏后的每条记录仍是合法 JSON，Host 字段与未脱敏时逐字节相同。

## 为什么不能直接改写已有日志

本决策不改写已落盘的 `plugin.log`、其 D10 恢复文件与既有明文历史：改写会破坏根决策 D2 的
单 writer 与 D10 的残缺尾部保留，且历史文件没有轮转、要等卸载才删除。清除历史明文的安全
路径是卸载并删除数据（根决策 D6/D8 的删除数据卸载），或等后续 history/retention 决策；导出
仍是原样复制文件、不再次脱敏，这些边界与「上线前历史文件仍含明文」一起，随 ADR 落地同步
记录到受影响文档。

## 为什么不是这些替代方案

| 方案 | 更好的维度及最终取舍 |
| --- | --- |
| 要求插件或 SDK 自己脱敏后再写 stderr | 值离插件最近，但第三方依赖和原生组件直接写 stderr，Host 无法强制；仍会有未脱敏内容。 |
| 把含敏感值的整行丢弃 | 不会泄露，但会丢掉同一行里的正常诊断，且需要可靠的「含敏感值」判定，误判代价高。 |
| 用正则或语义解析脱敏（URL、JSON、header 结构） | 覆盖更多编码形式，但值的形态多样、跨语言，维护成本高；明文子串替换已覆盖主要泄露路径，剩余边界在 D5 明确排除。 |
| 加密整个 plugin.log | 从文件层面杜绝泄露，但破坏现有可读性与诊断工具链，且仍需密钥管理；与「Host 拥有、可导出诊断」的定位冲突。 |
| 只靠文件权限与信任域，不做脱敏 | 现状。但诊断导出会把 plugin.log 带出信任域，且同工作区用户/进程可读，泄露面仍然存在。 |

## 本决策未解决的问题

- 远端 sink、日志导出、rotation/累计磁盘额度的脱敏与授权，留各自后续决策。
- 对编码形式（JSON 转义、百分号、base64）的覆盖，如需要更强的保证再另开决策。
- 跨 64 KiB 分片边界的完整匹配，需要改动分帧语义，暂不承诺。

## 落地顺序

1. **ADR 评审**：本 ADR 已评审到 `approved`，核心用例在同一 PR 补齐（证据先 `Missing`）。
2. **实现**：`ora-plugin-lifecycle` 登记表订阅 + `ora-agent-runtime` 交付点采集 + 宿主 spawn
   环境提供者 + `ora-plugin-runtime` 管道匹配（三段各自可验收）。
3. **落地同步**：实现验证后把 ADR 改为 `implemented`，并按最终实现同步正文与核心用例证据；
   同时把以下既有文档里「按已知运行时值的尽力脱敏，circus#1294 承接」的引用改指向本 ADR，并
   在 Skill ADR 的已知边界里注明「插件日志已有尽力脱敏」：
   - `decisions/node/agent-runtime/20261114-explicit-session-mcp-in-agent-session-spec.md`
   - `decisions/node/agent-runtime/20261118-host-mcp-servers-in-agent-session-spec.md`
   - `test-cases/node/agent-runtime/session-mcp.md`
   - `test-cases/node/agent-runtime/host-mcp-servers.md`
   - `decisions/desktop/plugin/category/skill/20261016-secret-settings-and-startup-environment.md`