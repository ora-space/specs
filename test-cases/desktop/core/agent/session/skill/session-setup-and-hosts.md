# Skill 交付的 session setup、host 边界与旧协议兼容

决策依据：[Skill 按存储路径以会话清单交给 Agent](../../../../../../decisions/desktop/core/agent/session/skill/0-skill-catalog-from-storage-paths.md)
（`approved`）。实现尚未开始，本文件全部义务的证据状态为 `Missing`。

## Readable Roots Must Follow The Declared Capability Without Failing Setup

### 风险

如果不支持 `additionalDirectories` 的 Agent 也收到这个字段，或缺少能力时 setup 失败，Skill 交付会让本来能用的
会话打不开。如果 load 时只发部分列表，ACP 会把它当作完整结果，已授权的根就被收回。如果可读根按 Skill 目录逐个
传，会话中途安装的插件 Skill 不在授权范围内。

### 前置状态

- Agent A 在初始化时声明 `sessionCapabilities.additionalDirectories`，Agent B 没有声明。
- Catalog 中有本地 Skill 和插件 Skill。

### 触发

对两个 Agent 分别执行 `session/new`、冷启动 `session/load` 和 MCP refresh 的 `session/load`；会话建立后再安装一个
新的 Skill 插件。

### 必须成立

- Agent A 的每次 `session/new` 与 `session/load` 都在 `additionalDirectories` 中带完整的 Skill 可读根：
  `<应用数据目录>/atoms/skills/` 和 `<Ora 主目录>/plugins/installed/`。
- 会话中途安装的插件 Skill 位于已发送的插件安装根之下，不需要重建会话。
- Agent B 的请求不带 `additionalDirectories`，setup 照常成功，首条 prompt 照常带清单。
- 同一 setup 请求用 Ora 命名空间的 `_meta` 字段携带当时清单中各 Skill 的包目录；插件忽略该字段时 setup 不受
  影响。

### 禁止结果

- 因 Agent 缺少 `additionalDirectories` 能力而使 `session/new` 或 `session/load` 失败。
- 向未声明能力的 Agent 发送 `additionalDirectories`。
- load 时只发送部分可读根。
- 按单个 Skill 目录而不是存储根授予读取。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| 声明能力的 Agent 在每次 new 与 load 中收到完整的两个可读根 | 尚无实现 | `Missing` |
| 未声明能力的 Agent 不收到该字段，setup 成功且清单照常发送 | 尚无实现 | `Missing` |
| 会话中途安装的插件 Skill 位于已授权根之下 | 尚无实现 | `Missing` |
| `_meta` 携带 setup 时刻的 Skill 包目录，忽略它不影响 setup | 尚无实现；键名随插件协议改动确定 | `Missing` |

### 决策依据

- Skill 会话清单决策 D6。
- 不变量 9。

## Node Sessions Must Stay Unchanged With Empty Host Extension Points

### 风险

注入逻辑放在 Node 与 desktop 共享的 `ora-agent-runtime` 里。如果 Node host 的扩展点没有返回空，或 runtime 在清单
为空时仍改动请求，cloud v1 的 Node 会话会出现前缀、增量或额外目录，Node 行为因此改变，而这超出了本决策的范围。

### 前置状态

- ora-node 以 Node host 承载 Agent 会话。
- 同一 runtime 版本也被 desktop host 使用，desktop 上有已安装 Skill。

### 触发

Node 会话执行 `session/new`、`session/load`，并发送首条及后续用户 prompt。

### 必须成立

- Node host 的 `skill_catalog` 与 `extra_read_roots` 返回空。
- Node 会话的 ACP setup 请求不带 `additionalDirectories` 或 Skill `_meta` 字段，prompt 不带任何清单块，与本决策
  之前逐字节一致。
- ora-node 不再依赖 `ora-effect`。

### 禁止结果

- Node 会话出现清单前缀、增量或额外可读目录。
- desktop 的 Skill 状态影响 Node 会话。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| Node host 两个扩展点返回空 | 尚无实现 | `Missing` |
| Node 会话的 setup 请求与 prompt 与决策前一致 | 尚无实现 | `Missing` |
| ora-node 的依赖中没有 `ora-effect` | 尚无实现 | `Missing` |

### 决策依据

- Skill 会话清单决策 D5。
- 不变量 8。

## Legacy Effect Resource Registrations Must Be Ignored Without Failing The Connection

### 风险

尚未升级的 Agent 插件仍会在注册时声明 `effectResources`，也仍实现 `effect/*` handler。如果新 host 把这个字段当成
错误，旧插件就无法连接，用户必须先等插件发版才能继续使用 Agent。

### 前置状态

- 一个旧版 Agent 插件在注册中携带 `effectResources`，并实现 `effect/coordinate`、`effect/reactivate`、
  `effect/verify_ready`。
- 另一个新版 Agent 插件不携带该字段。

### 触发

desktop 连接两个插件，建立会话、发送 prompt，期间安装和卸载 Skill。

### 必须成立

- 两个插件都连接成功，会话可用，首条 prompt 都带清单。
- host 忽略 `effectResources`，不基于它创建任何 Workspace 写入或后台任务。
- host 从不调用任何 `effect/*` 方法。

### 禁止结果

- 因注册中带有 `effectResources` 使连接失败或进入 Failing。
- 调用 `effect/*` 方法，或因 Skill 变化重启旧插件。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| 带 `effectResources` 的注册连接成功，字段被忽略 | 尚无实现 | `Missing` |
| Skill 安装、卸载期间 host 不调用 `effect/*` | 尚无实现 | `Missing` |

### 决策依据

- Skill 会话清单决策 D1。
- 不变量 12。
