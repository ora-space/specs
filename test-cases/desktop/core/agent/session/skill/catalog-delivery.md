# Skill 会话清单的交付、增量与重置

决策依据：[Skill 按存储路径以会话清单交给 Agent](../../../../../../decisions/desktop/core/agent/session/skill/0-skill-catalog-from-storage-paths.md)
（`approved`）。实现尚未开始，本文件全部义务的证据状态为 `Missing`。

## First Prompt Must Carry The Full Catalog Of Available Skills

### 风险

如果首条 prompt 漏掉某个已安装 Skill、混入 Unavailable 的 Catalog 行，或给出的不是 `SKILL.md` 的绝对路径，
Agent 会不知道这个 Skill 存在，或按失效、相对于 `cwd` 的路径读取，Skill 就悄悄不可用。清单顺序不确定还会让相同
输入产生不同 prompt。

### 前置状态

- Skill Catalog 同时有：本地导入的 Available Skill，两个插件分别提供的同名 Available Skill，以及一个 package
  已缺失的 Unavailable Skill。
- 应用数据目录或 Ora 主目录的路径含空格或非 ASCII 字符。
- 新建一个交互式聊天 Live Session，另有一个工作流节点会话。

### 触发

两个会话各自发送第一条用户 prompt。

### 必须成立

- prompt 带一个清单块，列出全部 Available Skill，每项包含名称、描述和存储位置下 `SKILL.md` 的绝对路径，路径保持
  原样。
- 两个同名 Skill 各自是一个条目，按路径区分。
- 条目按名称的 ASCII 小写排序，名称相同再按路径排序；相同 Catalog 产生相同的清单块。
- 清单块说明 `SKILL.md` 中的相对路径按其所在目录解析。
- 清单在发送时刻读取，而不是在 `session/new` 时读取。
- Catalog 中没有 Available Skill 时，prompt 不带清单块。
- 工作流节点会话使用同一份清单逻辑。

### 禁止结果

- Unavailable 的 Catalog 行进入清单。
- 合并、改名或丢弃同名 Skill，或因同名而阻塞会话。
- 呈现 Workspace 相对路径、物化副本路径，或经转义、缩写的路径。
- 把 Skill 正文放进清单块。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| 首条 prompt 列出全部 Available Skill 的名称、描述与 `SKILL.md` 绝对路径，排除 Unavailable 行 | 尚无实现 | `Missing` |
| 同名 Skill 按路径并列，顺序按名称小写再按路径确定 | 尚无实现 | `Missing` |
| 含空格或非 ASCII 字符的路径原样呈现 | 尚无实现 | `Missing` |
| 清单为空时不带块 | 尚无实现 | `Missing` |
| 交互式聊天与工作流节点会话的首条 prompt 都带清单 | 尚无实现 | `Missing` |

### 决策依据

- Skill 会话清单决策 D2、D3：存储位置即交付位置，首条用户 prompt 携带全量清单。
- 不变量 2。

## Later Prompts Must Carry Only Path Deltas

### 风险

如果会话中途的安装、卸载、升级不告诉 Agent，它会继续用已删除的路径，或不知道新 Skill。反过来，如果每条 prompt
都重发全量，或者路径没变也发通知，prompt 前缀每轮都在变，Agent 侧的 prompt cache 失效，并白白消耗 token。

### 前置状态

- 一个 Live Session 已经在首条 prompt 中收到全量清单。
- 随后依次发生：安装一个本地 Skill；只改某个本地 Skill 的内容和描述，不改名；把一个插件从 1.0.0 升级到 1.1.0；
  卸载另一个插件；本地 Skill 改名。

### 触发

每次变化后发送下一条用户 prompt；另有一次在没有任何变化的情况下发送。

### 必须成立

- 新增路径：增量块列出新增条目的名称、描述和路径。
- 移除路径：增量块列出移除条目的名称和路径，并说明它们已不可用。
- 插件升级与本地 Skill 改名：同一个增量块同时列出旧路径的移除和新路径的新增。
- 路径集合不变（包括只改内容、描述或名称而路径不变）时，prompt 不带任何清单块。
- prompt 请求发给 Agent 之后，已告知清单才更新；prompt 在进入 Agent 前失败时不更新，下一次发送重新比较，增量
  不丢失。
- 首条全量之后各次增量累积得到的路径集合，等于最近一次发送时的当前清单路径集合。

### 禁止结果

- 会话中途重发全量清单。
- 路径不变时发出增量。
- 以 Skill ID 比较，导致插件升级后版本目录变化而没有通知。
- 一次失败的发送吞掉增量，使后续 prompt 不再告知该变化。
- 为了告知 Skill 变化而 reload 会话或重启 Agent。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| 新增、移除与升级（同时有移除和新增）分别产生对应的增量块 | 尚无实现 | `Missing` |
| 路径集合不变时不带清单块 | 尚无实现 | `Missing` |
| 进入 Agent 前失败的 prompt 不更新已告知清单，下一次发送补发增量 | 尚无实现 | `Missing` |
| 多次变化后累积路径集合与当前清单一致 | 尚无实现 | `Missing` |

### 决策依据

- Skill 会话清单决策 D2（以 `SKILL.md` 路径为条目身份）、D4（增量）、D7（不 reload、不重启）。
- 不变量 3、4、7。

## Lost Notification Records Must Resend The Full Catalog

### 风险

已告知清单只在内存。Live Session actor 重建后如果仍按增量发送，或 Agent 压缩上下文后清单已不在其中，Agent 手里
就没有可用的 Skill 列表。反过来，如果同一 actor 内为 MCP refresh 做的 `session/load` 也触发重发，会白白打破
prompt cache。

### 前置状态

- 一个 Live Session 已收到全量清单和至少一次增量。
- Agent 支持 `session/load`，并能报告上下文压缩。

### 触发

分别发生以下情况，之后发送下一条用户 prompt：

1. 应用重启后冷启动 `session/load`；
2. 停止的 Ora Session 再次打开；
3. Agent 插件报告该会话的上下文已压缩；
4. 同一 Live Session 因 MCP 配置变化完成一次 refresh `session/load`。

### 必须成立

- 情况 1–3：下一条用户 prompt 携带当前全量清单，已告知清单从这份全量重新开始。
- 情况 4：已告知清单保留，下一条 prompt 只按 D4 带增量，没有变化时不带块。
- 已告知清单不写入 Ora Session 或 SQLite。

### 禁止结果

- actor 重建后从持久化数据恢复已告知清单并继续发增量。
- 报告压缩后仍只发增量。
- MCP refresh 的 `session/load` 触发全量重发。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| 冷启动 `session/load` 与重新打开会话后，下一条 prompt 带全量 | 尚无实现 | `Missing` |
| 插件报告压缩后，下一条 prompt 带全量 | 尚无实现；压缩信号的 wire 形状尚未确定 | `Missing` |
| 同一 actor 内的 MCP refresh `session/load` 不清空已告知清单 | 尚无实现 | `Missing` |
| 已告知清单不进入 Ora Session 或 SQLite | 尚无实现 | `Missing` |

### 决策依据

- Skill 会话清单决策 D4：已告知清单只在内存，以及重置条件。
- 不变量 5。

## Catalog Blocks Must Follow A Leading Slash Command And Stay Out Of User Input

### 风险

Agent CLI 只在 prompt 以斜杠命令开头时才把它当作命令。清单块如果插在用户输入的 `/命令` 或工作流节点前导的
`/调用名` 前面，命令会被当成普通文本。清单块如果显示成用户消息，用户会看到自己没写过的内容。

### 前置状态

- Catalog 有 Available Skill。
- 一个交互式会话的首条用户输入以 `/review` 开头；另一个会话的首条输入是普通文本。
- 一个工作流节点的 prompt 以必需 Skill 的前导 `/调用名` 开头。
- 已开始的会话之后发生一次 Skill 安装，下一条用户输入以斜杠命令开头。

### 触发

发送上述 prompt，并在界面上查看这些会话的消息。

### 必须成立

- 首个文本块以斜杠命令开头时，该块保持在最前，清单块或增量块紧随其后。
- 首个文本块不以斜杠命令开头时，清单块位于用户内容之前。
- 增量块遵循同样的位置规则。
- 界面不把清单块或增量块显示为用户输入的一部分。

### 禁止结果

- 清单块或增量块出现在前导斜杠命令之前。
- 把清单块并入用户输入的文本块，改变用户原文。
- 在用户消息中显示清单内容。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| 前导斜杠命令块保持在最前，清单块紧随其后；否则清单块在用户内容之前 | 尚无实现 | `Missing` |
| 工作流节点前导 `/调用名` 后的块顺序同样成立 | 尚无实现 | `Missing` |
| 增量块遵循相同位置规则 | 尚无实现 | `Missing` |
| 界面不把清单块显示为用户输入 | 尚无实现 | `Missing` |

### 决策依据

- Skill 会话清单决策 D3 规则 1、2，D4 位置规则，D8 第 4 条。
- 不变量 6。

## Skill Delivery Must Not Write Into Workspaces Or Gate Prompts

### 风险

如果新方案仍向 Workspace 写入 Skill 目录、标记或 Git exclude，Windows 上的链接、junction 和句柄问题，以及副本
过期问题都会回来。如果发送仍等待某种 Skill 就绪状态，退役门禁后会话可能永远发不出去。

### 前置状态

- 存在项目 Workspace 与工作流 worktree。
- Skill 依次被导入、更新、改名、卸载，插件被安装、升级、卸载。

### 触发

在这些变化前后和过程中建立会话并发送 prompt。

### 必须成立

- 整个过程中 Workspace 与 worktree 的文件集合不因 Skill 而变化。
- 发送 prompt 不读取任何 Skill 状态，不等待；前端没有 Skill 同步门禁、冲突横幅或恢复入口。
- Skill 变化不占用 Agent Session Barrier，不重启 Agent，不 reload 任何会话，也不打断进行中的 turn。

### 禁止结果

- 在 Workspace 中创建、复制、链接或修改 Skill 目录、`.ora-managed.json` 或 Git exclude。
- 因 Skill 状态阻止或延迟发送。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| Skill 与插件生命周期全过程中 Workspace 文件集合不变 | 尚无实现 | `Missing` |
| 发送不等待 Skill 状态，前端不再有 Skill 门禁 | 尚无实现 | `Missing` |
| Skill 变化不占用 barrier、不重启 Agent、不 reload 会话 | 尚无实现；barrier 侧义务也登记在 [MCP Refresh Must Hold One Agent Session Barrier Per Agent](../mcp/acp-session-setup-delivery.md#mcp-refresh-must-hold-one-agent-session-barrier-per-agent) | `Missing` |

### 决策依据

- Skill 会话清单决策 D1、D2、D7。
- 不变量 1、7。
