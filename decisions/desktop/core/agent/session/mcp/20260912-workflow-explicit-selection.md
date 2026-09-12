---
status: approved
---

# 工作流显式选择 MCP，并保留共享 Agent 进程

工作流 Agent 节点把冻结的 `mcps` 绑定作为显式 MCP 白名单。普通聊天继续自动选择当前合格插件；工作流只解析已启用的完整插件 ID，空集合表示不注入任何 MCP。Ora 在发送边界记录不含秘密的选择身份，便于区分 Host 选择错误与 Agent 未按会话落实 MCP 的错误。

本决策修改根决策的全局选择范围，但保留共享 Agent 进程、ACP wire contract、运行时刷新和 Workspace 文件边界。OpenCode 当前不能从共享进程中撤销早先 Session 注册的 MCP，这是尚未解决的 provider 一致性缺口；本决策不通过为每个 Session 启动长期子进程来规避该问题。

## 继承与修改

| 根决策约定 | 处理 | 当前规则 |
| --- | --- | --- |
| D1：全部配置完整插件构成 Effective MCP Set | 修改 | 普通聊天使用自动选择；工作流使用冻结节点的显式 ID 集合 |
| D2：所有 new/load 共用 resolver | 继承，扩展输入 | resolver 先按 Session 选择过滤，再解析配置和校验能力 |
| D3–D7：刷新、barrier、ACP 与全量成败 | 继承 | 显式集合在创建、恢复、重建和刷新中保持不变 |
| D8：Setting 实值不进入日志 | 继承 | INFO 日志只记录 secret-free revision identity |
| D9：不物化 Agent 原生 MCP 文件 | 继承 | 选择只存在于工作流图、运行快照和 Session Runtime |

## 决策

### D1：Session MCP Selection 区分自动选择与显式集合

内部 `SessionMcpSelection` 是封闭枚举：`Automatic` 表示普通聊天自动发现；`Explicit(Set<PluginId>)` 表示工作流的完整白名单。工作流图继续保存 `mcps: [{ mcpId, enabled }]`，运行时只把 `enabled: true` 的 ID 放入显式集合。缺少 `mcps`、空数组和全部禁用都产生空显式集合，不回退到自动发现。

显式选择在配置资格之前过滤。已选插件缺失、声明无效、配置不完整或 Agent 能力不足时，节点明确失败；未选择插件的安装、卸载和配置变化不影响该 Session。重复 ID、空 ID 和非法字段类型在冻结图解析时失败。

### D2：工作流运行冻结选择，Actor 恢复必须 fail closed

运行创建时冻结完整工作流图。每个 Agent 节点创建 Session 时，从该快照读取启用 ID；草稿后续修改只影响新运行。Session Actor 在首次创建、显式恢复、provider 重建、Agent 替换和运行时刷新中持有同一个选择策略。

Actor 被重新创建时，通过 Session 与节点运行关系回到运行引用的发布快照。关系缺失、节点不存在或图无效时返回错误，不能把无法恢复的显式选择扩大成 `Automatic`。

### D3：ACP 发送边界记录 secret-free 诊断身份

Host 在每次 `session/new` 或 `session/load` 之前记录 Ora Session ID、Agent、选择模式、成员数量，以及成员的 canonical Plugin ID、包版本、配置修订号和 transport。日志不包含 command、arguments、env、URL、headers 或 Setting 实值。

日志用于证明 Ora 实际发送的集合。它不证明 Agent 已建立对应 Server，也不证明 Agent 已撤销新列表中不存在的旧 Server。

## Provider 一致性缺口

ACP 把 `mcpServers` 定义为 Session setup 输入，Agent 应把请求中的列表落实为该 Session 的完整集合。OpenCode 1.18.30 仍把 ACP 传入的 Server 添加到进程级 MCP 注册表，后续空列表或缩小列表不会删除早先注册项。同一共享进程中的 Session 因而可能看到超出 Ora 当前选择的工具。

Ora 保留原有的单 Agent 共享进程模型。解决该缺口需要 OpenCode 修复 Session MCP ownership，或通过后续独立 ADR 设计资源有界的适配方案。在此之前，Host 侧的选择、冻结和发送已经实现，但 OpenCode 端到端隔离证据保持 `Missing`。

## 不变量

1. 普通聊天使用 `Automatic`；工作流使用冻结节点配置生成的 `Explicit` 集合。
2. 工作流空集合永远不回退到自动选择。
3. 创建、恢复、重建、刷新和 Agent 替换使用同一选择策略；恢复来源异常时 fail closed。
4. 草稿修改不改变已有运行的选择。
5. 日志只包含 secret-free 选择与 revision 身份。

## 核心测试依据

见 `specs/test-cases/desktop/core/agent/session/mcp/acp-session-setup-delivery.md`。
