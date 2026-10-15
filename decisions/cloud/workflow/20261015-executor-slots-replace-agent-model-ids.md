---
status: proposed
---

# 工作流 agent 节点按执行器槽位引用执行器，图里不再存 agent CLI 与模型 ID

工作流图在顶层声明执行器槽位（`executorSlots`），agent 节点只用 `agentConfig.executorSlot` 按名字引用槽位，
不再写 `agentConfig.executor { agentCli, modelId }`。槽位名到具体执行器（agent CLI、模型连接、模型）的映射由
运行发起者提供，见 [model-access 后续决策](../model-access/20261015-executor-slot-mappings-and-per-slot-run-bindings.md)。
同一张图因此可以由作者以外的人运行，也可以在 cloud 上执行。

本决策状态为 `proposed`。涉及 cloud 的图 schema、编辑器、导入和发布校验；运行时的解析、启动门禁和锁定由
model-access 后续决策承担。desktop 的图行为（编辑器、默认执行器、启动门禁、旧图迁移）不在本决策范围，由
ora-space/circus#124 另行决定；过渡期 desktop 继续写旧的 `executor` 字段。cloud 自己已存的图不做数据迁移，
只在导入和读取时按 D5 处理旧字段。

## 继承与修改

| 前序约定 | 本文处理 |
|---|---|
| 规范图 schema 由 specs 拥有、两端共用（ora-space/circus#67 第 2 问的结论，所在 ADR 待写） | 继承，不变；本文在其中增加执行器槽位 |
| [desktop workflow 根决策](../../desktop/core/workflow/0-node-runtime-orchestration.md)：agent 节点用 `agentConfig.executor { agentCli, modelId }` 固定执行器 | 修改，仅限 cloud：agent 节点改为引用槽位，图里不出现 agent CLI 与模型 ID；desktop 保持现状，见 ora-space/circus#124 |
| agent 节点的 `roleId` 指向 agent definition，提供 system instructions | 继承，不变；槽位与 role 正交，节点同时引用两者 |
| [model-access 根决策](../model-access/0-personal-model-connections-and-run-scoped-proxy.md) D1：模型 ID 是不透明字符串 | 继承，不变；`hint` 里的模型 ID 同样不透明，也不参与解析 |

## 决策

### D1：槽位在图顶层声明，`description` 必填

图顶层增加 `executorSlots`，与 `globalVariables` 并列。每个槽位只有名字、说明和可选的 `hint`：

```json
{
  "executorSlots": [
    { "name": "strong", "description": "需要深度推理的审查与规划",
      "hint": { "agentCli": "official/ora-space.opencode", "modelId": "anthropic/claude-sonnet" } }
  ],
  "nodes": [
    { "id": "review", "data": { "nodeType": "agent",
        "agentConfig": { "executorSlot": "strong", "roleId": "reviewer", "prompt": "..." } } }
  ]
}
```

- `name` 是小写标识符（`^[a-z][a-z0-9_-]{0,63}$`），图内唯一。
- `description` 必填，去掉首尾空白后为 1–500 个字符。
- 槽位按模型需求命名（如 `strong`、`fast`），不按角色命名。

为什么这样定：

- **`name` 是映射的键。** 运行者的个人映射按槽位名查找（见 model-access 后续决策），名字一旦写进图就会被很多人配置。
  所以名字必须稳定、易比较：不区分大小写的变体、空格和 Unicode 都会让"同一个名字"出现多种写法，
  导致映射看起来配了却查不到。
- **`description` 是运行者唯一能看到的说明。** 运行者看不到作者的映射，判断"这个槽位该配什么模型"
  只能靠描述，因此必填。
- **按需求命名，是为了和 role 分开。** agent 节点已经有 `roleId`。如果槽位叫 `reviewer`，作者会分不清
  "reviewer 角色"和"reviewer 槽位"；按需求命名后，同一个 `reviewer` role 可以用 `strong` 做正式审查、用 `fast` 做预检。
  命名约定只是推荐，schema 不强制，编辑器负责推荐一组常用名和标准描述。

### D2：节点按名字引用；不写就是 `default`

- agent 节点用 `agentConfig.executorSlot` 引用槽位。
- **`default` 是隐式保留的槽位**：不用声明，任何图都能引用；节点不写 `executorSlot` 就等于 `default`。
  作者不能声明名为 `default` 的槽位。
- 引用了未声明的槽位，图非法，保存和发布时都拒绝，错误指出节点和槽位名。
- 声明了但没有节点引用的槽位只给警告，不拒绝：作者可能正在编辑，也可能刻意保留。
- iteration、loop 等区域内的节点同样引用顶层声明。槽位不设作用域。
- `agentConfig.executor` 不再是 cloud 图 schema 的一部分。新字段另起名字而不复用 `executor`，
  这样新旧图不靠字段内容就能区分，D5 的兼容处理才有确定的判断依据。

`default` 的含义由运行者决定：它解析为运行者的个人默认执行器。它是作者"不关心用什么模型"的显式表达，
而不是未配置时的回退。运行者没有个人默认执行器时，`default` 和其他槽位一样按未配置拒绝
（model-access 后续决策 D2）。

### D3：`hint` 只用于展示

`hint` 记录作者自己用的执行器，供运行者第一次配置时参考：

- `hint` 永远不参与解析，也不作兜底。运行者没配置槽位时，门禁照样拒绝，不会改用 `hint`。
- 只有作者在编辑器里显式操作（如"用我的映射填入"）才写入；作者可以清空。导入、保存、发布都不会自动写入。
- 运行者可以显式选择"采用作者的选择"，把 `hint` 写进自己的映射。这是运行者主动写个人配置，
  写入时照常校验运行者自己的连接里是否有该模型。
- `hint` 随图共享，可能暴露作者自定义的模型名。模型 ID 本身是不透明字符串，不是凭据，
  但编辑器在写入前要提示这一点。

`hint` 的结构是 `{ agentCli, modelId }`。它不包含连接：连接属于作者个人，对运行者没有意义。
`agentCli` 取 agent 插件 ID（如 `official/ora-space.opencode`），与 cloud 会话输入的 `agentPluginId` 一致；
desktop 现在用的是另一套标识（如 `open_code`），两端如何对齐由 ora-space/circus#124 决定。

### D4：cloud 首期只支持 opencode

cloud 首期只支持 `official/ora-space.opencode`。`hint` 和个人映射都保留 `agentCli` 字段，但首期固定为 opencode，
编辑器不提供 agent CLI 选择。保留这个字段，是为了以后支持更多 CLI 时不必再改图 schema。

### D5：导入 desktop 旧图时兼容，丢掉执行器配置

ora-space/circus#124 完成前，desktop 导出的图仍然带 `agentConfig.executor`。cloud 导入这类旧图时**接受，不拒收**：

- 丢掉节点上的 `agentConfig.executor`（agent CLI 和模型 ID），这些节点不写 `executorSlot`，因此使用 `default`。
- 导入结果要列出受影响的节点，并说明它们改用了 `default`。旧的 agent CLI 和模型 ID 可以在这份摘要里展示，
  但不写进 `hint`：D3 规定 `hint` 只能由作者显式写入，导入不算。
- 图里同时出现 `executor` 和 `executorSlot` 时，以 `executorSlot` 为准，丢掉 `executor`。

为什么丢掉而不是自动生成槽位：旧图里的模型 ID 来自作者的 desktop 或 cloud 演示目录，与导入者的个人连接无关，
自动生成槽位只会制造一批"必须配置、但没人知道该配什么"的槽位。改用 `default` 后，导入的图可以直接运行；
作者想区分模型，再在编辑器里声明槽位。

cloud 自己已存的图不做数据迁移。读取时按与导入相同的规则处理旧字段，下次保存时以新 schema 落盘。

## 不变量

1. cloud 图里的 agent 节点不包含 agent CLI 和模型 ID，只包含槽位名或省略槽位。
2. 每个被引用的槽位名都在 `executorSlots` 中声明，或者是 `default`。
3. `default` 不能被声明，也总能被引用。
4. `hint` 不影响任何运行使用哪个执行器。
5. 导入和读取旧图不会写入 `hint`，也不会因为存在 `executor` 字段而拒收。

## 为什么不是这些替代方案

| 方案 | 取舍 |
|---|---|
| 图里继续存 `(agentCli, modelId)`，运行者按原模型 ID 做个人重映射 | 图 schema 不变、没有迁移成本，这一点优于本方案；但映射的键是作者碰巧用的模型：两个节点同模型不同意图时无法分开，作者一换模型，所有运行者的映射都失效 |
| 执行器并进 role（运行者配置 role → 执行器） | 少一个概念；但 role 是跨工作流共享的定义，"某 role 用什么模型"会全局一刀切，没有 role 的节点还要另找地方定执行器 |
| 槽位打包 role（槽位 = role + 执行器） | 同一 role 要用不同模型时得声明两个槽位，槽位又退回成角色名 |
| 平台固定一组槽位，作者不能自定义 | 名字全局一致、映射最省事；但作者无法表达平台没预置的需求，固定名字也容易和 role 撞名 |
| 未配置的槽位回退到 `default` | 上手更顺；但作者声明的 `strong` 可能在运行者不知情时跑在弱模型上 |
| 导入旧图时拒收 | 规则最干净；但 ora-space/circus#124 完成前 desktop 用户无法把工作流带到 cloud |
| 导入旧图时把旧执行器自动写进 `hint` | 保留了作者原来的选择；但违背 `hint` 只能由作者显式写入的规则，导入者也不一定是作者 |

## 本决策未解决的问题

- **desktop 的图行为**：编辑器、默认执行器、启动门禁、旧图迁移，见 ora-space/circus#124。
- **按工作流覆盖个人映射**：首期只有个人全局映射，覆盖属于后续扩展。
- **本文的位置**：执行器槽位属于规范图 schema，应当和 ora-space/circus#67 第 2 问的 schema ADR 放在一起。若那份 ADR 成为
  `decisions/cloud/workflow/` 的根决策，本文作为它的后续决策；若规范图 schema 另设共享目录，本文随之移动。

## 落地顺序

1. **schema 与校验**：图 schema 增加 `executorSlots` 和 `executorSlot`，保存和发布时校验 D1、D2。
   完成条件：引用未声明槽位的图被拒绝；`default` 可以直接引用。
2. **编辑器**：槽位声明、常用名推荐、`hint` 写入和清空；删除演示模型目录（`DEMO_AGENT_MODELS`）。
3. **导入与读取兼容**：按 D5 处理旧字段，并在导入结果里列出受影响的节点。

第 1 步之后，新图就不再写 agent CLI 和模型 ID。运行时的解析和门禁按 model-access 后续决策的落地顺序接入，
在那之前，cloud 后端本来就不执行工作流，行为不变。
