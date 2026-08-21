# MCP 插件

## 定位

MCP 是纯配置型插件。它描述一个可供 Agent 使用的 MCP Server，但自身不运行 `main.js`，不启动 Deno 插件进程，也不调用 Ora SDK。

MCP Server 由目标 Agent CLI 按自身配置启动、连接和管理。Ora 负责安装与解析 MCP 包；Agent 插件负责把 Ora 的规范化描述转换成 Claude Code、Codex、OpenCode 等目标 Agent 的配置格式。

MCP kind 使用 Ora 自有配置，不兼容 MCPB `manifest.json` 或 MCP Registry `server.json`，也不承诺与它们互相转换。

## 职责

| 模块 | 职责 |
| --- | --- |
| MCP 包 | 声明唯一 transport、启动或连接信息、Setting 及附带文件 |
| Ora MCP 配置模块 | 校验安装包、编译声明、解析 Target、Setting、SecretRef、包内路径和 Workspace 上下文 |
| Agent 插件 | 将 `ResolvedMcp` 渲染、合并并写入目标 Agent 的配置，只管理 Ora 创建的条目 |
| Agent CLI | 创建 MCP Server 进程或连接远程 MCP，并管理实际连接生命周期 |

Agent 插件不能直接读取 MCP 的 `assets/config.json`、`store.json` 或 Ora 数据库。MCP 配置模块隐藏这些实现，只把当前 Agent 实例所需的强类型 `ResolvedMcp` 交给 Agent 插件。

## 包结构

```text
plugin.orax
├── orax.toml
├── README.md
├── logo.svg
└── assets/
    ├── config.json
    └── server            # stdio 类型的包内可执行文件
```

`orax.toml` 示例：

```toml
resolver = 1
name = "github-mcp"
namespace = "official"
kind = "mcp"
version = "1.0.0"
description = "GitHub MCP Server"
homepage = "https://github.com/example/github-mcp"
license = "MIT"
```

`kind = "mcp"` 时，Ora 强制要求包内存在 `assets/config.json`。路径是规范的一部分，不在 `orax.toml` 中重复声明。

MCP 包没有 `main.js`，也不能在 `orax.toml` 中声明供 Deno 胶水进程使用的 Capability。stdio Server 的执行条件由当前 Agent 实例及其沙盒决定。

## 配置格式

`assets/config.json` 是严格 JSON。根对象包含：

- `schemaVersion`：第一版必须为 `1`；
- `settings`：可选的用户配置声明，语义见 [2-settings.md](./2-settings.md)；
- `transport`：恰好一个 stdio 或 HTTP 连接描述。

未知字段、未知版本和未知 transport 一律拒绝安装，不能静默忽略。

## stdio

```json
{
  "schemaVersion": 1,
  "settings": {
    "repository": {
      "type": "string",
      "title": "Repository",
      "description": "Repository in owner/name format",
      "required": true
    },
    "token": {
      "type": "secret",
      "title": "GitHub token",
      "description": "Token used to access GitHub",
      "required": true
    }
  },
  "transport": {
    "type": "stdio",
    "command": "assets/server",
    "args": [
      "--repository",
      { "setting": "repository" },
      "--workspace",
      { "context": "workspace" }
    ],
    "env": {
      "GITHUB_TOKEN": {
        "setting": "token"
      }
    }
  }
}
```

### `command`

第一版只允许执行当前 `.orax` 包内 `assets/` 下的文件：

1. `command` 必须是以 `assets/` 开头的规范化包内相对路径。
2. 禁止绝对路径、盘符、UNC、空路径和父目录穿越。
3. 路径解析后必须仍位于当前精确安装版本内。
4. 禁止符号链接、junction 或 reparse point 逃逸。
5. 目标必须是普通文件，并满足当前平台的可执行文件规则。
6. 不允许通过 PATH 查找 `npx`、`uvx`、`node`、`python`、shell 或其他系统命令。
7. Ora 安装时验证路径和文件类型；使用时从不可变安装记录解析为绝对路径并再次确认版本归属。

不同平台需要不同二进制时，由 registry manifest 的 Target 资产提供不同 `.orax`，但各 Target 包中的逻辑 `command` 路径应保持一致。

### `args`

`args` 缺省为空数组。每项只能是：

- JSON string、number 或 boolean 字面值；
- `{ "setting": "<id>" }` 普通 Setting 引用；
- `{ "context": "workspace" }` 运行上下文引用。

Ora 将 number 和 boolean 转换为规范字符串。Secret Setting 禁止出现在 `args` 中。

### `env`

`env` 缺省为空对象。键必须是当前平台接受的环境变量名，值可以是普通字面值、普通 Setting 引用或 Secret Setting 引用。

MCP Server 不继承 Ora 的完整环境。Agent 插件接收结构化的环境绑定和 SecretRef，而不是 Secret 明文。Agent adapter 必须把它转换成目标 Agent 支持的安全环境引用；不能安全表达时返回 `SecretInputUnsupported`，不得把明文写入 Workspace 配置。

### 工作目录

stdio MCP Server 的工作目录固定为当前 Agent 实例的权威 `cwd`，不在 `config.json` 中增加可配置 working directory。包内资源使用 Ora 解析后的绝对路径访问，避免依赖启动目录。

## HTTP

`type = "http"` 专指 MCP Streamable HTTP，不包括 deprecated HTTP+SSE：

```json
{
  "schemaVersion": 1,
  "settings": {
    "token": {
      "type": "secret",
      "title": "Access token",
      "description": "Token used to authenticate with the MCP server",
      "required": true
    }
  },
  "transport": {
    "type": "http",
    "url": "https://mcp.example.com/v1",
    "headers": {
      "Authorization": {
        "setting": "token",
        "prefix": "Bearer "
      }
    }
  }
}
```

HTTP 规则：

1. `url` 必须解析为一个绝对 URL，不得包含用户名或密码。
2. 默认只允许 HTTPS。
3. `http://localhost`、`http://127.0.0.1` 和 `http://[::1]` 只在明确的开发模式下允许。
4. `url` 可以是普通 Setting 引用，但不能引用 Secret。
5. `headers` 缺省为空对象；Header 名和值必须通过严格校验，禁止换行和控制字符。
6. Secret 只允许通过 Header Setting 引用出现，不得写成字面值或拼入 URL。
7. 不支持 deprecated SSE、多个 endpoint、stdio fallback 或由 Ora 启动的 bundled local HTTP Server。
8. HTTP transport 不能声明 `command`、`args` 或 `env`。

## 排他领域模型

Ora 将 JSON 编译为以下排他结构，而不是一个 transport 字符串配合大量可选字段：

```text
McpTransport
├── Stdio {
│     command: PackageExecutable,
│     args: [ValueExpression],
│     env: Map<EnvironmentName, ValueExpression>
│   }
└── Http {
      url: HttpEndpointExpression,
      headers: Map<HeaderName, ValueExpression>
    }
```

该模型使以下非法状态不可表达：

- stdio 携带 URL 或 Header；
- HTTP 携带 command、args 或 env；
- 同一 MCP 同时声明两个 active transport；
- remote HTTP 伪装成本地 executable；
- stdio 使用任意系统命令。

## 安装期编译

安装 `.orax` 时，Ora 完成：

1. registry ID、kind、version、Target 和摘要交叉校验；
2. archive 路径与展开安全校验；
3. `orax.toml` 严格解析；
4. `assets/config.json` schema、Setting 和引用校验；
5. transport 排他性与 kind-specific 规则校验；
6. stdio command containment、文件类型和平台可执行性校验；
7. 编译并持久化不可变 `InstalledMcpDescriptor`。

安装成功只证明包和声明静态合法，不证明用户已完成配置、远程 endpoint 可达或 MCP initialize 成功。

## 使用期解析

使用 MCP 前，Ora 根据精确安装版本、当前插件 `store.json` 和 Agent Workspace 生成：

```text
ResolvedMcp {
  id,
  exact_version,
  transport: ResolvedStdio | ResolvedHttp,
  managed_identity
}
```

解析必须满足：

- 所有 required Setting 均可解析；
- Secret 保持为 SecretRef，不转成普通字符串；
- `context: workspace` 只解析为当前 Agent 实例的权威 `cwd`；
- stdio command 解析为当前精确安装版本中的绝对路径；
- 不把原始 `config.json`、整个 `store.json` 或插件数据目录交给 Agent 插件。

缺少用户配置时返回 `NeedsConfiguration`，不得调用 Agent 插件物化一个不完整条目。

## Workspace 选择

MCP 选择属于 Workspace，而不是单个 Session。同一 Agent 实例下的所有 Session 使用相同 `cwd`，目标 Agent 的 MCP 配置通常也以该目录为作用域。

```text
Workspace / cwd
├── desired MCP set
├── configuration revision
└── Agent instances and Sessions
```

共享同一 `cwd` 的 Session 不能拥有互相冲突的 MCP 集合。修改 Workspace 的 desired MCP set 会产生新 revision；已经运行的 Agent 实例需要重新配置并重启后才使用新集合，第一版不承诺热插拔。

## Agent 配置时序

Ora 在创建 Agent 实例 ID 后、启动 Agent 进程前完成 MCP 配置：

```text
读取 Workspace desired MCP set
        ↓
解析每个 MCP 为 ResolvedMcp
        ↓
ora/agent/configure_agent {
  agent_instance_id,
  cwd,
  revision,
  mcps: [ResolvedMcp]
}
        ↓ 成功
ora/agent/start_agent {
  agent_instance_id,
  cwd
}
```

`configure_agent` 接收完整期望集合并进行幂等 reconcile，不使用可能丢失的 `add_mcp`/`remove_mcp` 增量事件。Agent 插件必须：

1. 添加或更新当前 desired set 中的 Ora-managed 条目；
2. 删除不再存在于 desired set 的 Ora-managed 条目；
3. 保留用户自己创建的配置项；
4. 使用稳定 `managed_identity` 区分 Ora 条目与用户条目；
5. 先规划完整变更，再原子替换目标配置；
6. 返回实际应用的 revision、managed identity 和配置 fingerprint。

任一 selected MCP 无法解析或安全物化时，`configure_agent` 失败，Ora 不调用 `start_agent`。用户修复配置或从 Workspace 选择中移除失败项后重试。

## Agent 沙盒

stdio MCP Server 是 Agent CLI 创建的子进程，因此继承 Agent 实例沙盒。Ora 创建 Agent 沙盒时，将当前选择的 MCP 精确安装版本作为只读、可执行资源附加进去：

```text
Agent sandbox
├── authoritative cwd
├── Agent executable
└── selected MCP package versions
```

Agent 插件不能任意读取其他插件安装目录。只有本次 `ResolvedMcp` 引用的精确包版本进入该 Agent 实例的资源集合，升级后的新版本也不会替换运行实例正在使用的旧路径。

由于 MCP Server 由 Agent CLI 创建，第一版不承诺为每个 MCP 子进程建立不同于 Agent 的独立网络或文件权限。MCP 包不应声明 Ora 无法执行的逐 MCP 沙盒权限。

## 状态与错误

MCP 至少区分以下状态：

```text
Installed             # 包和 config.json 静态合法
NeedsConfiguration    # required Setting 或 Secret 不可用
Ready                 # 可生成 ResolvedMcp
UnsupportedByAgent    # Agent adapter 无法表达 transport 或 SecretRef
ConfigurationFailed   # 目标 Agent 配置读取、合并或原子写入失败
```

`Installed`、`Ready` 和“运行中的 Agent 已加载”是不同事实。安装新 MCP 或修改 `store.json` 不会自动改变已经运行的 Agent；Ora 必须推进 Workspace configuration revision，并要求重新配置和重启 Agent。
