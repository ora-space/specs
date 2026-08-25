# Agent 插件

## 定位

Agent 是 Ora 插件体系中最重要的插件类型。大部分插件能力与 Ora 产品能力都围绕 Agent 的创建、配置、运行和协作展开。Agent 插件负责把某一种 Agent CLI 或 Agent Runtime 适配为 Ora 统一的 Agent 协议，并管理由自己启动的全部 Agent 实例。

Ora 定义稳定的实例与 Session 语义，Agent 插件负责将这些语义映射到具体 Agent 的进程、连接和原生 Session。Ora 不规定一个 Agent 实例必须对应一个操作系统进程，也不规定一个底层进程只能服务一个 Session。

## Agent 实例与 Session

关系模型如下：

```text
AgentPlugin
└── AgentInstance (agent_instance_id, cwd, sandbox)
    ├── Session A
    ├── Session B
    └── Session …
```

必须满足以下不变量：

1. Agent 插件必须管理由其启动的全部 Agent 实例，并能通过 Ora 提供的 `agent_instance_id` 唯一定位实例。
2. 一个 Agent 实例可以承载一个或多个 Session。
3. 同一 Agent 实例下的所有 Session 必须使用相同的 `cwd`。需要不同 `cwd` 时，Ora 必须创建不同的 Agent 实例。
4. 一个进程服务一个还是多个 Session、是否为每个 Session 建立独立连接，以及如何映射具体 Agent CLI 的原生 Session ID，均由插件自行决定。
5. 每个 Agent 实例都必须运行在独立的宿主沙盒中。Agent 插件的 `main.js` 无权直接创建 Agent 进程；进程启动、资源授权和生命周期控制必须通过 Ora SDK 交由宿主执行。

`agent_instance_id` 是 Ora 分配的稳定运行期身份，不由插件生成，也不应替换为某个 Agent CLI 自己的 Session ID 或进程 ID。插件可以在内部维护这些底层身份与 `agent_instance_id`、Ora Session 之间的映射。

## 宿主调用插件的方法

Agent 插件必须实现以下方法：

### `ora/agent/start_agent`

启动或初始化一个 Agent 实例。

参数至少包含：

```text
agent_instance_id
cwd
```

- `agent_instance_id` 由 Ora 创建并传入；
- `cwd` 是该实例及其全部 Session 的权威工作目录；
- 插件必须记录实例状态，并保证之后的操作可通过该 ID 路由；
- 插件可以立即启动底层进程，也可以只完成惰性初始化，在首次 Session 操作时再启动，具体策略由插件决定；
- 对同一 `agent_instance_id` 的重复调用不得意外创建两个互不受控的实例。插件应返回明确的幂等结果或状态冲突错误。

### `ora/agent/stop_agent`

停止并释放一个 Agent 实例。

参数：

```text
agent_instance_id
```

插件必须结束或释放该实例拥有的全部 Session、进程、连接和临时资源。方法完成后，该 ID 不再接受新的 Session 操作。重复停止应安全处理，不得影响其他 Agent 实例。

### `ora/agent/list_models`

返回指定 Agent 实例在其 `cwd` 上下文中可用的模型列表。

参数：

```text
agent_instance_id
```

模型列表可能受 Agent 配置、工作目录、项目配置、登录状态或 Agent CLI 版本影响，因此它属于实例上下文中的查询结果，而不是插件级静态常量。

## ACP 下的模型列表

在 ACP 语义中，模型列表只有在完成 `session/new` 后才能取得，而 Ora 的统一接口仍要求插件通过 `ora/agent/list_models` 提供模型列表。两者之间的适配责任属于 Agent 插件。

插件应缓存 `cwd` 到模型列表的映射：

```text
cwd -> model list
```

调用 `ora/agent/list_models` 时：

1. 若当前 `cwd` 已有仍然有效的缓存，插件可以直接返回缓存结果；
2. 若没有缓存，插件负责完成取得模型列表所需的 ACP 流程，包括在必要时执行 `session/new`；
3. 获取成功后，插件更新该 `cwd` 的缓存并返回统一的模型列表；
4. 缓存何时因配置变化、认证变化、Agent 重启或插件判断而失效，由插件实现，但不得把其他 `cwd` 的结果错误复用到当前实例。

为获取模型列表而创建的底层 Agent CLI Session ID 是否继续用于后续 Ora Session，由插件自行决定。插件可以复用该 Session，也可以将其视为临时探测 Session 并创建新的业务 Session；Ora 只依赖返回的模型列表和统一 Session 语义，不依赖该底层 ID 的生命周期。

## 实现边界

Ora 负责：

- 分配 `agent_instance_id`；
- 指定并维护实例的权威 `cwd`；
- 为每个实例建立和管理宿主沙盒；
- 通过统一协议调用 Agent 插件；
- 管理 Ora Session 与 Agent 实例的归属关系。

Agent 插件负责：

- 管理全部 Agent 实例及其内部资源；
- 适配具体 Agent CLI 或 Runtime 的进程与协议；
- 维护 Ora 身份与底层进程、连接、原生 Session ID 之间的映射；
- 实现 ACP 等具体协议所需的启动顺序与状态转换；
- 获取、缓存并失效按 `cwd` 区分的模型列表；
- 在停止实例时完整释放其拥有的资源。
