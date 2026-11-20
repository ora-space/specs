---
status: proposed
---

# 插件日志：Host 按已知运行时值尽力脱敏

本决策在已实现的 [插件日志根决策](0-host-owned-plugin-log.md) 之上，给宿主拥有的插件日志增加
「对已知运行时值尽力脱敏」：Host 在把 stderr 解码成日志行之后、进入有界队列之前，把会话层
登记的一组敏感明文（插件 MCP Settings 的有效值、宿主 MCP 的 URL 与请求头、模型 token 与代理
地址、legacy 模型 token 环境变量）替换为固定占位符 `[REDACTED]`。desktop 与 Node 共用同一套
实现。这是尽力而为的脱敏：只做原始字符串子串替换，不解析 JSON 转义、百分号编码或 base64，
跨 64 KiB 分片的超长值可能漏掉；它不把「已脱敏」宣称为「日志绝不泄露」。

本 ADR 目前是 `proposed`，实现落在 `ora-plugin-runtime`、`ora-plugin-lifecycle` 与会话层
（Node / desktop Backend），不改 Plugin Protocol 的对外字段，也不改变日志的归属、级别、队列、
容量与收尾语义。

## 继承与修改

继承 [插件日志根决策](0-host-owned-plugin-log.md) 的全部约定：日志归属独立 logs tree、单
writer、JSONL 身份与接收时间由 Host 填写、SDK 五个 console 方法共用 stderr、版本化 envelope
与 raw fallback、按 Plugin ID 的持久阈值、队列与 sink 故障隔离、收尾截止时间。本决策只修改
一处边界：

| 前序约定 | 处理 |
| --- | --- |
| 根决策开头「不承诺任意第三方输出自动脱敏」 | 修改：对 Host 已知的运行时值，改为尽力脱敏 |
| 根决策「日志可能含插件主动写出的敏感内容」作为后续决策 | 本决策承接其中的「已知运行时值脱敏」，其余（远端 sink、累计磁盘额度、更强进程隔离）仍各自后续决策 |

## 决策

### D1：脱敏发生在 Host 管道的唯一咽喉点，进入有界队列之前

stderr 字节经 `decode → filter(level) → render JSONL → 有界队列 → 写文件`。脱敏作用在
`render`（产出日志行字符串）之后、`queue.try_send` 之前——这是结构化路径与 raw fallback 的
共同咽喉点，任何路径都经它产出一行字符串再入队。

- 只改写将入队的这一行文本，不改变该行对应的级别、时间戳、Host 身份、入队顺序与容量核算。
- 脱敏用纯函数，输入是「已渲染的一行」与「本插件的脱敏值列表」，输出是替换后的一行；任何
  输入异常（空列表、空行）都必须照原样返回，不能让脱敏成为新的日志或协议故障源。

### D2：脱敏值按插件登记，由 Host 从会话层下发

每个插件实例的脱敏值列表由 Host 在会话层生成，随插件启动沿现有启动链下传到日志管道，插件
无法读取或覆盖它。

- `PluginLogSetup` 新增承载「脱敏值列表」的字段；列表在会话层（Node `driver` / desktop
  Backend 的 session setup）生成，经 `PluginLaunchRequest` → `PluginLogSetup` → pipeline
  `submit` 下发。
- 值随会话 setup 产生，并在 MCP Settings 或模型绑定刷新、宿主 MCP 变化时同步更新；更新的值
  只影响之后入队的行，不回改已落盘记录。
- 列表是明文子串集合，去重；空字符串与过短（长度不足以唯一标识一个秘密，易误替换常见子串）
  的值不登记。

### D3：脱敏是原始字符串子串替换，不解析语义

对每行日志，逐个登记值做 `replace(value, "[REDACTED]")`。只匹配明文子串，不尝试理解 JSON、
URL、HTTP 头或 base64 的结构。

- 占位符统一为 `[REDACTED]`，与现有 `SessionMcpBinding`、`HostMcpServer` 的 `Debug` 脱敏占位
  一致，避免在日志里引入新的可识别标记。
- 替换顺序不保证「先长后短」之外的语义：长度更长的值先替换，避免短值是长值前缀时漏掉长值。

### D4：覆盖的运行时值集合

脱敏值列表来自以下运行时值，desktop 与 Node 共用同一采集逻辑：

| 值 | 来源 |
| --- | --- |
| 插件 MCP Settings 的每个有效值 | `SessionMcpBinding.settings` 经 `resolve_mcp_transport` 绑进 stdio `env`/`args` 与 http `url`/`headers` 的 setting 值 |
| 宿主 MCP 的 `url` 与每个 `headers` 值 | `HostMcpServer`（`NodeSessionMcp` / `HostServers` 的输入） |
| 模型 token 与代理地址 | `AgentModelContext.token`、`AgentModelContext.proxy_base_url`（adapter 路由） |
| legacy 模型 token 环境变量值 | `ModelAccess.environment` 的 `ORA_MODEL_ACCESS_TOKEN` |

以上字段的明文值在会话层完整可得。Host 只登记「本身即秘密」的值，不登记由这些值派生的、
可能出现在日志中的其他形式（见 D5）。desktop Backend 路径不注入模型 token（走插件自身的
个人模型配置），因此该路径下模型 token 项为空；Node 路径才有 Host 注入的模型 token。

### D5：尽力而为的边界与已知局限

脱敏是尽力而为，明确以下不覆盖的情形，避免把「已脱敏」误读为「日志绝无秘密」：

- JSON 转义（如 `\u0041`）、百分号编码（如 `%41`）、base64 等非明文形式匹配不到；
- 跨 64 KiB 分段边界的超长值可能漏掉（超长内容被分片，分片边界不保证值完整落在一行内）；
- Agent 写到自己能访问的其他文件（非 plugin.log）的不归本决策管；
- 插件不经过 Host 管道、直接自行写文件的不归本决策管。

脱敏后日志仍可能包含插件主动写出的、Host 不认识的其他敏感内容。读取、导出与远端 sink 的
脱敏/授权是后续决策，不在本决策范围。

## 不变量

1. 脱敏只发生在 `render` 之后、`queue.try_send` 之前，不改写队列之前的任何状态。
2. 脱敏值由 Host 登记，插件不能读取或覆盖；空列表时管道行为与未实现脱敏时完全一致。
3. 脱敏不改变一条日志记录的级别、时间戳、Host 身份、入队顺序与容量核算。
4. 脱敏异常（空值、空行、匹配失败）不产生错误，不阻塞插件协议，不改变记录数量。
5. 脱敏是尽力而为的明文子串替换，不承诺覆盖 JSON 转义、百分号编码、base64 或跨分片值。

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

1. **ADR 评审**：本 ADR 由 `proposed` 评审到 `approved`，并补核心测试用例（脱敏咽喉点、值
   下发、尽力而为边界）。
2. **实现**：desktop 与 Node 共用一套实现——`PluginLogSetup` 新增脱敏值字段，会话层生成并
   沿启动链下发，pipeline `submit` 在进队列前做子串替换；值源接入 `SessionMcpBinding` 及
   刷新、`NodeSessionMcp`/`HostServers`、desktop Backend 交付的 Settings 与完整 token。
3. **落地同步**：实现验证后把 ADR 改为 `implemented`，按最终实现同步正文与核心用例证据。
