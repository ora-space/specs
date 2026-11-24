---
status: implemented
---

# 经 Node 承载的交互节点：完成工具送达前新节点默认 `end_turn`，按产品版本恢复 `agent`

> 历史决策：D4 由[Cloud 经 Node 宿主 MCP 提供完成工具](20261124-host-completion-tool-for-node-hosted-interactive-nodes.md)
> （`approved`）兑现。恢复版本起，D2 的默认 `end_turn` 与“`agent` 按 `end_turn` 执行并标为暂不生效”不再适用，新节点的默认值与
> 显式 `agent` 的执行以该决策为准；风险一节留给使用方的“Agent 到不了端点的部署”也见该决策 D1。D3、D4 的
> 不改写与升级前会话保持 `end_turn` 不变。

[Agent 完成信号](../../desktop/core/workflow/20261110-agent-completion-signal.md)（下称完成信号 ADR）让新建的
交互节点默认 `completionSignal: "agent"`，前提是宿主能把完成工具交给 Agent。经 Node 承载的 Controller
（目前是 Cloud v2 的 v2-server）还做不到：[宿主 MCP](../../node/agent-runtime/20261118-host-mcp-servers-in-agent-session-spec.md)
（下称宿主 MCP ADR）只定了 Node 怎样送达宿主 Server，Controller 还没有提供完成工具。本决策规定：在这类
Controller 的产品版本接入完成工具之前，为它写图的编辑器给新的交互节点写 `completionSignal: "end_turn"`。
已经写在图里的 `agent` 照常保存，运行时按 `end_turn` 执行，并在编辑器和运行记录里标为“暂不生效”。从
Controller 接入完成工具、且它锁定的 Node 声明 `session_host_mcp_servers` 的那个产品版本起，编辑器恢复默认
写 `agent`。之前保存的 `end_turn` 是显式值，不改回，已冻结的图也不动。

本决策处于 `implemented` 状态：D2 由 [cloud#403](https://github.com/ora-space/cloud/pull/403)（merge `2b80c023`）在
Cloud v2 的工作流编辑器和 v2-server 实现，[cloud#417](https://github.com/ora-space/cloud/pull/417)（merge `43ad6eb2`）补齐了
暂不生效提示与运行测试，D3 不需要额外实现，D4 的恢复版本尚未出现。本决策只定默认值，以及显式
`agent` 在恢复版本前后怎样执行；验证义务与证据见
[完成工具送达前的默认完成方式用例](../../../test-cases/cloud/workflow/end-turn-default-before-completion-tool.md)，
仍为 `Partial` / `Missing` 的缺口如实保留。涉及 Cloud v2 的工作流
编辑器和 v2-server 的交互节点执行，desktop 的行为和代码都不变。完成工具怎样经 Node 送达（宿主 MCP ADR
D6 的 Controller 义务，以及 [Controller 交付宿主 MCP 用例](../../../test-cases/controller/session/host-mcp-servers.md)）
不在本 ADR，由之后接入完成工具的使用方决策承接。本决策没有数据迁移义务：它不改写任何已保存或已冻结的值。

## 继承与修改

本文是本目录[工作流执行根决策](0-cloud-side-scheduler-owns-workflow-execution.md)的后续决策，修改完成信号
ADR 的两处默认值规定，并限定宿主 MCP ADR D6 的适用时点。

| 前序约定 | 处理 | 当前规则 |
| --- | --- | --- |
| 根决策的历史决策：正式 Cloud 的发布门禁拒绝非 null 的 `agentConfig.completion` 和 `schemaVersion: 4` | 继承，不变 | 正式 Cloud 的图没有 `completionSignal`，按完成信号 ADR D7 读作 `end_turn`；本决策只约束接受 `schemaVersion: 4`、经 Node 承载交互节点的 Controller |
| 完成信号 ADR D1：只有 `completionSignal: agent` 的交互节点注入完成工具；`end_turn` 节点不注入 | 继承，不变 | 显式 `end_turn` 在任何宿主上都不注入完成工具 |
| 完成信号 ADR D5、D7：`completionSignal: end_turn` 与没有该字段的旧图按 legacy 规则执行 | 继承，不变 | v2 写的 `end_turn` 在 desktop 上按 legacy 执行（D3） |
| 完成信号 ADR D7：“升级后新建或重新保存的交互节点默认 `agent`；编辑器明示，允许显式改回 `end_turn`” | 修改，只限经 Node 承载 | 恢复版本（D4）之前，这类 Controller 的编辑器给新交互节点默认写 `end_turn`，允许显式选 `agent`，并提示它暂不生效；desktop 不变 |
| 完成信号 ADR 落地结果 3：前端给新建或重新保存的交互节点写 `completionSignal: "agent"` 与 `schemaVersion: 4` | 修改，只限经 Node 承载 | 恢复版本之前写 `completionSignal: "end_turn"` 与 `schemaVersion: 4`；version-4 边界和关闭交互模式时清理策略字段的规则不变 |
| 完成信号 ADR D7：不支持工具调用的 Agent 照常运行，靠 `force` 兜底，不做能力探测或自动降级 | 继承，不变 | 恢复版本之前 `agent` 按 `end_turn` 执行，取决于产品版本，不取决于对 Agent 或 Node 的探测，见 D2 |
| 宿主 MCP ADR D6：使用方需要宿主能力而 Node 不具备时，在 WorkflowRun 启动准入时拒绝，不静默降级 | 修改适用时点 | 从恢复版本起完整适用。恢复版本之前 Controller 不提供完成工具，`agent` 按 `end_turn` 执行，并在编辑器和运行记录里标为“暂不生效”：降级可见、有记录，不是 D6 禁止的静默降级（D2） |
| 宿主 MCP ADR D4：非空宿主列表遇到不声明 HTTP MCP 的 Agent 时，以 `host_mcp_http_capability_missing` 失败 | 继承，不变 | 恢复版本起，显式 `agent` 的节点在这类 Agent 上启动即失败，不再按 `end_turn` 运行（D4 后果） |
| [插件资产导入](20261016-asset-import-and-published-snapshots.md)：已发布快照不被改写，运行只冻结 `snapshotId` | 继承，不变 | 恢复默认值不改写任何快照或已保存的图 |

## 问题与约束

- **默认值必须是宿主能执行的策略**：`agent` 节点要等 Agent 调用完成工具才会 ready。宿主没有完成工具时，
  默认 `agent` 只能按 `end_turn` 执行，用户看到的默认值却和实际行为不一致；等完成工具上线，所有按默认值
  建出的节点会一起改变行为，没有人主动选过。
- **默认值在写图时决定**：编辑器写图时不知道这张图之后在哪个 Node 上运行，也不知道那个 Node 的能力；
  同一张图可以在 desktop 和 Cloud v2 之间来回导入。所以默认值只能取决于编辑器所在的产品版本。
- **显式值不能被猜**：图里的 `end_turn` 可能是默认值，也可能是用户的选择，两者在存储上无法区分；
  `agent` 也一样。任何把旧值批量改成新默认值的做法都会覆盖用户的选择。
- **宿主值在 start 时固定**：宿主 MCP ADR D2、D6 规定宿主列表在 execution 启动时给定、整个 execution
  内不变；升级前启动的会话拿不到完成工具。

## D1：只适用于经 Node 承载的交互节点

- 适用范围：Controller 通过 `AgentSessionSpec` 把会话交给 Node 承载的交互式 Agent 节点，目前只有 Cloud v2
  的 v2-server。为这类 Controller 写图的编辑器（目前是 Cloud v2 的工作流编辑器）按 D2 和 D4 决定默认值。
- desktop Backend 直接承载的工作流会话已经有完成工具（完成信号 ADR D1），默认 `agent` 不变，desktop 不改代码。
- 正式 Cloud 的发布门禁继续拒绝 `schemaVersion: 4`（根决策的历史决策），本决策不打开它。正式 Cloud 以后
  接受这些字段时，同样按本决策决定默认值。

**为什么不改 desktop**：desktop 能执行 `agent`，改默认值只会让新建的节点退回“提问回合被当成完成”的误判，
这正是完成信号 ADR 要消除的问题。

## D2：恢复版本之前，新节点默认 `end_turn`，显式 `agent` 按 `end_turn` 执行

- **写图**：新建交互节点，或为没有完成策略的节点打开交互模式时，编辑器写 `completionSignal: "end_turn"`
  与 `schemaVersion: 4`。节点已有的 `completionSignal`（无论 `end_turn` 还是 `agent`）原样保留。没有该字段的
  旧节点仍按完成信号 ADR D7 读作 `end_turn`，编辑器不替它补写。
- **显式选择 `agent`**：用户仍可以选 `agent` 和 `autoAcceptAgentCompletion`，值照常保存。编辑器在这两个选项
  旁提示“暂不生效”，说明现在按“回合正常结束即可”执行、发起人仍需确认，并说明完成工具上线后会开始生效。
- **执行**：Controller 把 `agent` 节点按 `end_turn` 的完成门禁执行，不附加宿主 MCP，也不签发 attempt token；
  `autoAcceptAgentCompletion` 不生效。运行记录把这两项列为未应用的设置，编辑器和运行页据此显示“暂不生效”。
- **判断依据是产品版本**：Controller 不探测 Agent 是否支持 HTTP MCP，也不探测 Node 是否声明
  `session_host_mcp_servers`；同一产品版本下，`agent` 节点在所有部署上都按 `end_turn` 执行。

**为什么这不是宿主 MCP ADR D6 禁止的静默降级**：D6 禁止的是 Controller 已经提供宿主能力、目标 Node 却不具备时
悄悄按 `end_turn` 执行，冻结的图和实际执行不一致，也没有记录。这里在写图时、运行时都明确标出“暂不生效”，
运行记录列出未应用的设置；用户在保存前就知道 `agent` 现在不会生效。

## D3：跨端导入时保持显式值

图在 desktop 和 Cloud v2 之间导入时，`completionSignal` 原样保留，不按目标端的默认值改写：

| 来源 | 值 | 在 desktop 上 | 在 Cloud v2 上（恢复版本之前） | 在 Cloud v2 上（恢复版本起） |
| --- | --- | --- | --- | --- |
| v2 编辑器的默认值 | `end_turn` | 显式 `end_turn`，按 legacy 执行，不注入完成工具 | `end_turn` | `end_turn` |
| desktop 编辑器的默认值 | `agent` | 按完成信号 ADR 执行 | 按 `end_turn` 执行，提示“暂不生效” | 按完成信号 ADR 执行 |
| 升级前的旧图 | 无字段 | legacy `end_turn` | legacy `end_turn` | legacy `end_turn` |

**为什么不在导入时改写**：存储上分不清默认值和用户选择（见“问题与约束”）。导入时改写，用户在一端显式选的
`end_turn` 或 `agent` 会被另一端的默认值覆盖，而且图在两端之间来回一次就会变。

## D4：恢复 `agent` 的条件是产品版本

- **恢复条件**：从同时满足下面两条的 Controller 产品版本起，编辑器给新的交互节点默认写 `agent`：
  1. Controller 接入完成工具，即按宿主 MCP ADR D6 提供完成工具的宿主 Server；
  2. 该版本锁定的 Node 声明 `session_host_mcp_servers`。
- **不改回旧值**：恢复版本之前保存的 `end_turn` 是显式值，不自动改成 `agent`；已发布或已冻结的图、快照和运行
  都不改写。
- **显式 `agent` 开始生效**：恢复版本起，图里写着 `agent` 的节点（包括恢复版本之前用户显式选的、从 desktop
  导入的）按完成信号 ADR 执行：Agent 必须发出完成信号或由人 `force`，`autoAcceptAgentCompletion` 开始生效，
  “暂不生效”的提示随之消失。WorkflowRun 启动准入按宿主 MCP ADR D6 检查目标 Node 的宿主能力，不具备时以有名字
  的原因拒绝；Agent 不支持 HTTP MCP 时，节点会话按宿主 MCP ADR D4 以 `host_mcp_http_capability_missing` 失败。
  两种情况都不再按 `end_turn` 运行。
- **已启动的会话不变**：升级前已经启动的节点会话没有附加完成工具，在会话结束前继续按 `end_turn` 执行；节点在
  恢复版本下启动的下一个会话才按 `agent` 执行。

**为什么按产品版本而不是运行时状态**：默认值写进图的那一刻就固定了，而写图时不知道之后由哪个 Node 运行。
按运行时状态决定默认值，同一个编辑动作会因为当时连着哪个 Node 而写出不同的图；按产品版本决定，同一版本的
编辑器总是写同一个值，图的含义也不随部署变化。

**接受的后果**：恢复版本之前显式选了 `agent` 的节点，在升级后会自动开始要求完成信号；开了自动接受的节点
会开始自动接受。这些都是用户在“暂不生效”提示下作出的显式选择，可以接受；D2 的提示必须事先说明这一点。

## 不变量

1. 恢复版本之前，经 Node 承载的 Controller 的编辑器给新交互节点写 `completionSignal: "end_turn"`；节点已有的
   `completionSignal` 原样保留。
2. 恢复版本之前，`agent` 节点按 `end_turn` 执行、不附加宿主 MCP；`completionSignal` 与
   `autoAcceptAgentCompletion` 在编辑器和运行记录中标为暂不生效。
3. 是否按 `agent` 执行只取决于 Controller 的产品版本，不取决于对 Agent 或 Node 能力的运行时探测。
4. 跨端导入不改写 `completionSignal`；没有该字段的旧图在所有宿主上都是 legacy `end_turn`。
5. 恢复默认值不改写任何已保存的 `end_turn`、已发布的快照或已冻结的运行。
6. 恢复版本起，显式 `agent` 的节点按完成信号 ADR 执行，宿主能力不足时按宿主 MCP ADR D4、D6 明确失败。
7. desktop 直接承载的会话默认 `agent`，行为和代码不变。

## 为什么不是这些替代方案

| 方案 | 取舍 |
| --- | --- |
| 照旧默认 `agent`，只提示“暂不生效” | 默认值和实际行为不一致；完成工具上线时，所有按默认值建出的节点一起开始要求完成信号，而这些节点没有人主动选过 `agent`。优点是两端默认值一致、少一份 ADR，但代价落在不知情的用户身上 |
| 按运行时状态决定默认值（写图时探测 Node 能力） | 写图时不知道之后由哪个 Node 运行；同一个编辑动作在不同部署上写出不同的图 |
| 恢复版本时把已保存的 `end_turn` 批量改回 `agent` | 分不清默认值和用户的显式选择；改写已发布快照违反插件资产导入 ADR 的快照不改写约定 |
| desktop 也改为默认 `end_turn` | desktop 已有完成工具，改了只会让新节点退回“提问回合被当成完成”的误判 |
| 恢复版本之前拒绝保存 `agent` | 从 desktop 导入的图无法保存，用户的选择也会丢；保存后标为“暂不生效”可以保留意图 |
| 未生效的 `agent` 节点在启动准入时拒绝 | 等于在完成工具上线前禁用从 desktop 导入的交互节点；`end_turn` 门禁仍保留首轮确认和 `force`，按它执行并提示足以避免误判被掩盖 |

## 风险、本决策未解决的问题与落地顺序

- **用户忽略提示**：用户在恢复版本之前选了 `agent`，升级后节点开始等完成信号。缓解：D2 的提示说明之后会生效，
  恢复版本的发布说明列出这一变化。
- **恢复版本无法提供宿主能力的部署**：宿主 MCP ADR D6 规定，Agent 连不到 Controller 端点的部署不提供宿主能力。
  这类部署即使在恢复版本上，显式 `agent` 的节点也会在启动准入时被拒绝；它的默认值如何处理留给接入完成工具的
  使用方决策。
- **未解决**：完成工具怎样经 Node 送达、attempt token 的承载方式、宿主 MCP ADR 的 Controller 用例，都由之后
  接入完成工具的使用方决策承接；本决策只定默认值。

落地顺序：

1. **本 ADR**：`approved`。同一变更在完成信号 ADR 的落地结果 3 与 D7 加 `> 历史决策：`，在宿主 MCP ADR 的
   “未解决”一节链到本 ADR，并在 `test-cases/cloud/workflow/` 建立核心用例。desktop 不改代码。
2. **`implemented`**：Cloud v2 的工作流编辑器和 v2-server 按 D2 实现（cloud#403，merge `2b80c023`）：新节点写
   `end_turn`，`agent` 与自动接受按 `end_turn` 执行并标为暂不生效；证据登记在核心用例里。
3. **恢复**：由接入完成工具的使用方决策落地，它的产品版本按 D4 把编辑器默认值改回 `agent`。使用方的 ADR 在
   “继承与修改”表里写明“兑现本 ADR 的 D4”；本 ADR 届时只在标题下加一条 `> 历史决策：` 指向它，不改写正文。
