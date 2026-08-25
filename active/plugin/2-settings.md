# 插件配置

## 目标

插件可以声明需要用户填写的配置项。配置声明属于插件代码，随安装版本发布且不可变；用户填写的配置值属于运行数据，在安装完成后按需产生，并独立于插件版本保存。

配置模块在以下 seam 上提供统一接口：插件只声明 Setting，调用方只请求已解析配置；配置文件定位、持久化、类型校验、默认值、Secret 和原子写入均隐藏在模块实现中。

## 代码与数据布局

```text
~/.ora/plugins/
├── installed/<namespace>/<plugin_name>/<version>/
│   └── assets/config.json
└── data/<namespace>/<plugin_name>/
    └── store.json
```

- `assets/config.json` 由插件作者提供，声明配置结构及使用方式，属于不可变安装制品。
- `store.json` 由 Ora 管理，只保存用户配置后的值，属于插件私有运行数据。
- 新安装的插件没有 `store.json`。用户第一次保存配置时，Ora 才创建它。
- 插件升级会安装新的 `assets/config.json`，但不同版本共享同一插件数据目录，因此用户配置不会随代码版本一起被替换。

`assets/config.json` 不得通过相对路径或绝对路径引用 `store.json`。配置中的 Setting ID 是两者之间唯一稳定的关联。Ora 根据当前插件身份解析实际数据目录，插件作者和 Agent 插件都不会得到 `store.json` 的路径。

## Setting 声明

`assets/config.json` 使用 `settings` 对象声明配置项。对象键是包内唯一、跨版本尽量稳定的 Setting ID：

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
    "readOnly": {
      "type": "boolean",
      "title": "Read only",
      "description": "Disable write operations",
      "required": false,
      "default": true
    },
    "token": {
      "type": "secret",
      "title": "Access token",
      "description": "Token used to authenticate requests",
      "required": true
    }
  }
}
```

第一版支持以下类型：

| `type` | 值 | 说明 |
| --- | --- | --- |
| `string` | JSON string | 普通文本 |
| `number` | JSON number | 有限数值，不接受 `NaN` 或无穷值 |
| `boolean` | JSON boolean | 布尔开关 |
| `secret` | Secret reference | 敏感值，明文不进入普通文件或插件协议 |
| `file` | 规范化绝对路径 | 由 Ora 文件选择器产生并校验 |
| `directory` | 规范化绝对路径 | 由 Ora 目录选择器产生并校验 |

通用字段：

- `title`：用户可见名称，必填。
- `description`：用途与获取方式，必填。
- `required`：是否必须在使用插件前解析出值，缺省为 `false`。
- `default`：可选的非敏感默认值，必须与 `type` 匹配。

`secret` 禁止声明 `default`。配置声明、示例、错误信息和日志中均不得包含 Secret 明文。

## Setting 引用

插件配置的其他字段通过结构化表达式引用 Setting：

```json
{ "setting": "repository" }
```

需要在值外添加固定文本时可以声明前后缀：

```json
{
  "setting": "token",
  "prefix": "Bearer ",
  "suffix": ""
}
```

结构化引用优先于 `${...}` 字符串模板，因为它没有转义语言，Ora 可以在安装时验证引用存在、类型适用并保持 SecretRef，不必先把 Secret 拼成普通字符串。

引用规则：

1. `setting` 必须引用同一份 `config.json` 中已声明的 ID。
2. 普通 JSON 字符串、数字和布尔值表示字面值。
3. `prefix` 和 `suffix` 缺省为空字符串，只能用于最终目标为字符串的位置。
4. Secret Setting 只能出现在该插件 kind 明确允许的敏感位置；例如 MCP stdio 的环境变量或 MCP HTTP 的 Header。
5. Secret Setting 禁止用于可执行文件、命令参数、URL、文件路径和日志字段。

## `store.json`

用户保存上述配置后，Ora 创建：

```json
{
  "schemaVersion": 1,
  "values": {
    "repository": "ora-space/desktop",
    "readOnly": true,
    "token": {
      "secretRef": "plugin/official/github-mcp/token"
    }
  }
}
```

`store.json` 只保存值，不复制 `title`、`description`、`required` 或 `default` 等声明。声明的权威来源始终是当前安装版本的 `assets/config.json`。

持久化规则：

1. 普通值按其 JSON 类型保存。
2. Secret 明文保存到操作系统 Credential Store；`store.json` 只保存 Ora 生成的 opaque `secretRef`。
3. `secretRef` 只能由 Ora 解释，不能作为文件路径、环境变量名或明文值使用。
4. Ora 使用临时文件、刷新和原子替换更新 `store.json`，不能原地写入半份 JSON。
5. 文件权限限制为当前用户可读写。
6. 删除一个配置值后，如果没有默认值且该项为必填，插件进入 `NeedsConfiguration`。
7. 当前 `config.json` 未声明的旧值不提供给插件，并在用户下一次成功保存时移除。
8. 已保存值与新版本声明类型不匹配时不做隐式类型转换，保留原文件并将对应 Setting 标记为需要重新配置。

## 解析语义

配置值按以下顺序解析：

```text
store.json 中的用户值
        ↓ 不存在
config.json 中的非敏感 default
        ↓ 不存在且 required = true
NeedsConfiguration
```

安装只要求 `assets/config.json` 结构合法，不要求用户已经配置。使用插件能力之前，Ora 才解析当前值：

```text
compile(package_root)
    -> CompiledPluginConfig

resolve(compiled_config, plugin_store, runtime_context)
    -> ResolvedPluginConfig | NeedsConfiguration
```

`compile` 隐藏 JSON 解析、严格字段校验、Setting ID、引用关系和默认值验证。`resolve` 隐藏 store 定位、SecretRef、类型检查和运行上下文替换。调用方与测试都通过这两个接口观察结果，不直接解析文件。

## 运行上下文

用户配置与 Ora 提供的运行上下文是不同概念。当前 Workspace 不应作为用户值写入 `store.json`，配置可以显式引用 Ora 提供的上下文：

```json
{ "context": "workspace" }
```

Ora 将其解析为当前 Agent 实例的权威 `cwd`。第一版只提供规范定义过的上下文名称；未知名称安装失败。插件不能通过上下文引用获得任意宿主路径或环境变量。

## 作用域

第一版的 `store.json` 是插件全局配置：同一插件在不同 Workspace 中默认使用同一组用户值。Workspace 选择哪些插件，与插件自身如何配置，是两个独立状态。

若以后需要 Workspace 覆盖，应新增独立的 Workspace Setting 存储，并定义明确的覆盖优先级；不能把绝对 Workspace 路径作为 `store.json` 的键，也不能悄悄改变现有全局配置语义。

## 状态与生命周期

配置状态至少包括：

```text
Unconfigured        # store.json 尚不存在，且没有需要保存的值
Ready               # 所有必填 Setting 均可解析
NeedsConfiguration  # 缺值、类型失配或 SecretRef 不可用
InvalidDeclaration  # assets/config.json 不符合当前 schema
```

- `InvalidDeclaration` 是安装制品错误，该版本不能启用。
- `NeedsConfiguration` 是有效安装的用户状态，Ora 应展示配置入口而不是把插件标记为损坏。
- 卸载插件时是否删除 `store.json` 与 Credential Store 中的 Secret，必须由卸载交互明确决定。
