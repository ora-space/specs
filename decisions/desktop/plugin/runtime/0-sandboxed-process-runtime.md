---
status: implemented
---

# 插件运行时：以一次性能力注册的沙箱子进程承载全部可执行插件

Ora 的可执行插件使用宿主以 Deno 启动的独立沙箱子进程运行：进程的 stdout 专用于帧协议（四字节大端长度、一字节帧类型、UTF-8 JSON，长度字段包含帧类型字节），插件启动后只做一次 `ora/register` 能力注册，随后本进程的协议面固定不变。`ora-plugin-lifecycle` 是后端唯一能够启动、停止、回收插件进程的仲裁者；`ora-plugin-runtime` 拥有单进程的生命周期与 stdio 协议；`ora-plugin-protocol` 持有双方共享的 wire 契约。`webview`、`skill`、`mcp`、`hook` 四类插件不启动进程，永久停留在 `stopped` 运行时状态。

该决策已经由 `crates/plugin-protocol`、`crates/plugin-runtime`、`crates/plugin-lifecycle` 与 `crates/backend` 落地。本文固定"哪些插件有进程、进程如何声明能力、宿主与插件如何通信、谁能决定进程生死"的系统承诺；插件分发、安装与 installed 目录布局属于 `desktop/plugin/registry` 领域，插件种类（如 Skill 静态包）属于 `desktop/plugin/category` 领域，agent 会话与 ACP 语义属于 agent-runtime 域。

## 状态与边界

`status: implemented`：`ora-plugin-runtime` 与 `ora-plugin-lifecycle` 已按本文描述落地并承担生产路径（agent、workbench 插件进程）。

兼容性边界：插件进程是硬边界，既有有进程插件必须走帧协议与注册流程，宿主不解析 stdout 之外的输出（stderr 仅用于日志排水）；本地文件系统布局是硬边界，installed 只读 package 与 per-plugin 可写 `data/` 角色不能互换；本 ADR 不承担 installed tree 迁移、`.orax` 包格式与 SDK 协议断点升级（resolver、snake_case 命名等），它们属于 registry 领域决策。

## 责任边界

| 能力 | 所有者 |
| --- | --- |
| 帧编解码、JSON-RPC 方法常量、`ora/register` 声明、DTO | `ora-plugin-protocol` |
| 单进程生命周期、stdio 协议、进程树回收 | `ora-plugin-runtime` |
| 进程仲裁、运行时状态、launcher 适配、kind 权限派生、storage 与 childprocess 宿主方法 | `ora-plugin-lifecycle` |
| 安装、release 选择、installed tree | `ora-plugin-manager`（分发域输入） |

## 决策

### D1：可执行性由 kind 决定，有进程的插件必定是受管沙箱子进程

`agent` 与 `workbench` 是仅有的两个可启动进程的 kind；`webview`、`skill`、`mcp`、`hook` 永远不启动进程，因此在运行时状态机中恒为 `stopped`。进程由宿主用 Deno 加载插件的 JavaScript entrypoint，`cwd` 固定为 package root，stdout 为帧协议专用通道。

- Deno 权限按 kind 在启动时派生：`workbench` 不获得任何 `--allow-*`；`agent` 沿用历史宽权限集合，收窄它是后续决策，不改变当前承诺。
- 宿主不向插件注入任何环境变量，插件不获知宿主路径；宿主授予插件的唯一受管持久化区域是绑定到其 `data/` 目录的 storage。
- 权限路径中含逗号时拒绝启动，因为 Deno 把逗号当作权限列表分隔符，不能静默产生与声明不同的权限面。

为什么这样选：进程边界把插件的 panic、内存与文件系统副作用隔离在宿主之外，并使每个插件享有独立的回收与失败面；kind 决定是否值得付出进程与握手成本，静态 kind 不进入进程面，避免为静态目录引入握手、权限和失败维面。

### D2：`ora/register` 一次性声明本进程的整个协议面

插件启动后必须发送恰好一次 `ora/register` 通知，声明 `methods`（宿主可以 invoke 的方法）、`emits`（插件可主动发送的通知）与可选的 `effectResources`（Workspace 相对路径的 Effect 资源）：

```jsonc
{
  "jsonrpc": "2.0",
  "method": "ora/register",
  "params": {
    "methods": ["agent/start", "effect/coordinate"],
    "emits": ["agent/acp"],
    "effectResources": [
      {
        "workspaceRelativePath": ".agents/skills",
        "materializationFormat": "ora/skill-directory.v1",
        "coordination": "quiesce_before_mutation"
      }
    ]
  }
}
```

- `methods` 是宿主 invoke 的完整白名单，invoke 未声明的方法在本地失败、不触达插件。
- `emits` 是插件主动通知的白名单，越界通知使连接失效。
- `effectResources` 的 locator、format 与 coordination 由运行时严格解析；语义校验与可信 consumer 身份来自被启动的插件本身，不来自该 payload。
- 注册对本进程生命周期固定：第二次注册是协议错误。
- 插件向宿主发起的请求（如 `ora/storage/*`）不需要声明：由启动时绑定的 `HostRequestHandler` 决定是否服务，未知方法回答 `method_not_found`（`-32601`）。

为什么这样选：把能力面固定在启动时，宿主在每次调用前即可判定方法存在性，也能发现越权 emit；声明缺失或越界即让连接失效，保证响应相关性在协议层可信，而不是依赖事后对账。

### D3：四种流量形态，相关性与背压各归其位

| 方向 | 形态 | 相关性 | 上限 |
| --- | --- | --- | --- |
| 宿主 → 插件 | `invoke` 请求/响应 | 运行时 request id | `call_timeout` |
| 宿主 → 插件 | `notify` 通知 | 无 | 无 |
| 插件 → 宿主 | 白名单通知 | 无 | 无（背压归消费者） |
| 插件 → 宿主 | 请求，由 handler 服务 | 插件 request id | handler 本身 |

- `call_timeout` 只覆盖 `invoke`：通知（如持续数分钟的 ACP 帧流）不因控制调用超时被切断。
- 连接级不做背压：一个 noisy 通知流不得阻塞其他流量，有界队列属于每个消费者。
- `invoke` 超时后，请求 id 移入有界 tombstone（256 条）；对已知请求的迟到有效响应被丢弃，真正未知的 id 仍是协议失败。

为什么这样选：相关性决定失败能否精确定位到 generation；有界 tombstone 让本地取消不会误杀健康插件，也不让过期 id 无界增长；按消费者划分背压，避免跨流相互拖累。

### D4：进程归属只有一个仲裁者

`ora-plugin-lifecycle` 是后端唯一能启动、停止、回收插件进程的组件；调用方需要与插件说话时通过 `ensure_running` / `connection` 获取 pin 到当前 generation 的连接，而不是自行启动进程。

- 生命周期打开时所有插件处于 `stopped`；显式扫描重建已安装 snapshot，保留有效 package 的运行时状态，新发现的 package 立即可用且 `stopped`。
- `agent`/`workbench` 进程按需启动，停止不改变可用性；每个状态转换镜像到 per-plugin `watch` 通道，`ensure_running` 在通道上等待。
- 卸载在 per-plugin 操作锁内先关闭 surface、停止进程，再原子化暂存并提交 `installed/`（可选 `data/`）删除；暂存失败按逆序回滚。
- 调用方只在一次交互期间持有连接：重启后的插件永远不能通过旧 generation 被寻址。

为什么这样选：让 settings 表面报告的运行时状态与实际进程一一对应，消除"进程存在但不可见"或"宣称停止仍存活"的两个失联方向；generation pinning 使重启、卸载与失败都收敛到同一个仲裁入口。

### D5：启动期一次性固定身份与宿主绑定

每次启动在同一 launch 中完成：创建 `<data-dir>/plugins/data/<namespace>/<name>/`（含 `downloads/`）→ 按 kind 派生 Deno 权限 → 以 package root 为 `cwd` → 把绑定该插件 data 目录的 storage handler 作为 `HostRequestHandler` 交给运行时 → 等待注册并按 manifest kind 校验。

- 身份来自启动绑定，不来自请求参数：handler 通过"被绑定到哪个进程"识别调用方。
- 注册按 manifest kind 校验：`workbench` 可以声明可 invoke 的方法但不得声明 emit 通知；`webview`/`skill`/`mcp`/`hook` 没有进程因此不能注册；agent 契约由 backend 的 agent runtime 验证。

为什么这样选：来源身份在进程创建时固化，任何运行中协商或参数注入都无法伪装来源，也无需在每次请求里携带身份。

### D6：退出必须回收完整进程树，失败从创建点起同一条回收边界

`shutdown` 只请求终止；需要防止 generation 重叠的一方使用 `shutdown_and_wait`，它在观察到优雅退出或超时 kill 并回收完整进程树后返回。缺 stdio、注册超时、契约拒绝、控制调用失败等任何创建后失败，都走同一条回收边界；意外退出会立即关闭插件发起的通知接收端，让连接 owner 判失败并应用重启策略。

插件通过 `ora/childprocess/*` 宿主方法衍生的子进程在 generation 退出时被尽力杀掉，`exit` 通知永远排在最后一块 `stdout`/`stderr` 之后；宿主衍生的进程不得存活于所属 generation 之外。

为什么这样选：进程树是幂等的失败单位——不完整回收会留下孤儿进程，或让新 generation 与旧进程并发写同一资源；把"创建后一切失败都回收"作为单一规则，避免按失败点枚举可回收路径。

## 为什么不是这些替代方案

| 方案 | 优势 | 否决理由 |
| --- | --- | --- |
| 进程内库模式（插件代码直接跑在宿主进程） | 调用延迟最低、实现最简 | 没有进程边界，插件的 panic、内存与文件系统副作用直接污染宿主；无法按 kind 收敛权限，也无法构成可独立回收的 generation |
| 每次调用即拉即弃子进程 | 无长驻状态泄漏、天然隔离 | agent ACP 会话、effect 资源协商、storage 缓存都是长生命周期状态；每次握手与进程创建的成本和失败面不可接受 |
| 无声明式的自由 stdout 管道 | 协议最简单、插件负担最小 | 宿主在调用前无法判定方法是否存在，也无法发现越权 emit；歧义输出让响应相关性失去可判定性，任何失败都难以定位 |
| 插件直接裸跑 agent CLI，宿主不做子进程管理 | 去掉中继与帧协议开销 | agent 子进程脱离 generation 回收面，Ora 无法保证其随插件退出被回收，也无法约束其数据目录与权限 |

## 不变量

1. 每个已安装插件同一时刻至多一个活跃 generation；连接只能 pin 到具体 generation，重启后旧 generation 不可寻址。
2. stdout 只承载帧协议；协议之外或无法解析的输出使该进程失败。
3. 每个进程恰好一次 `ora/register`，声明在本进程生命周期内固定；invoke 未声明方法在本地失败，emit 未声明通知使连接失效。
4. 只有 `ora-plugin-lifecycle` 能启动、停止或回收插件进程；settings 报告的运行时状态与实际进程一一对应。
5. `webview`、`skill`、`mcp`、`hook` 永不启动进程；`agent`、`workbench` 按需启动，停止不改变可用性。
6. 插件权限由 kind 在启动时派生，`cwd` 固定为 package root，宿主不注入环境变量；宿主授予插件的唯一受管持久化区域是绑定到其 `data/` 目录的 storage。
7. 插件发起请求的来源身份来自启动绑定，不由请求参数决定。
8. 退出必须回收完整进程树：正常退出、超时 kill、创建后失败与 `ora/childprocess` 子进程都不允许成为孤儿。
9. `invoke` 超时后已知请求 id 进入有界 tombstone，迟到有效响应被丢弃；未知 id 永远视为协议失败。
10. 停止只请求终止；需要避免 generation 重叠时必须使用 `shutdown_and_wait`。
11. 通知背压只属于消费者队列，连接级背压不得因单个 noisy 流阻塞其他流量。

## 当前未解决的问题

- agent 插件的宽权限集合是否收窄：留待专门决策，不改变本文的权限派生承诺。
- 多插件进程的资源配额（内存、CPU、并发上限）尚未定义。
- 协议/SDK 的断点升级策略（resolver、snake_case 命名）与 installed tree 迁移由 registry 领域决策承担。
- 若未来 `mcp`、`skill`、`hook` 需要进程形态，其注册面、权限面与失败语义由各自领域 ADR 单独定义。