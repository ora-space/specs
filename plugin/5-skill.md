# Skill 插件

## 定位

Skill 插件是一种只分发 Skill 内容的纯数据插件。一个插件版本可以包含一个或多个 Skill；Ora 安装插件后解析这些 Skill，将其登记到统一 Skill 目录，插件启用后即可由所有 Agent 和 Workflow 使用。

Skill 插件不包含 `main.js`，不启动 Deno，也不使用 Ora SDK。它不拥有独立运行时，生命周期完全依附于所属插件。v1 不提供 Setting、Secret、宿主权限、插件依赖、工具依赖或 Agent 白名单，也不允许单独启用插件内的某一个 Skill。

Plugin Skill 与用户创建或从 SkillHub 导入的 local Skill 使用同一套 Skill 选择和运行语义，但所有权不同：local Skill 由用户管理，可以修改；Plugin Skill 由已安装插件版本管理，运行期间只读。

## 包结构

Skill 插件包必须包含插件级清单、说明、图标以及 `assets/skills/` 下的 Skill：

```text
plugin.orax
├── orax.toml
├── logo.svg
├── README.md
└── assets/
    └── skills/
        ├── review/
        │   ├── SKILL.md
        │   ├── scripts/
        │   ├── references/
        │   └── assets/
        └── testing/
            └── SKILL.md
```

`orax.toml`、`logo.svg` 和 `README.md` 均为必需文件。`README.md` 描述整个插件；每个 `SKILL.md` 描述一个具体 Skill。Skill 根目录可以包含脚本、参考资料、资源和其他辅助文件，这些文件与 `SKILL.md` 一起构成不可拆分的 Skill 包。

Skill 插件没有插件入口文件。Skill 内可以包含 JavaScript 等普通辅助脚本，但它们不会被当作插件 `main.js` 执行，也不会因此获得 Deno 权限。

## `orax.toml` 中的 Skill 声明

Skill 插件使用 `kind = "skill"`，并通过 `[[skills]]` 显式声明包内的全部 Skill：

```toml
resolver = 1
name = "ora-space.dev-tools"
namespace = "official"
kind = "skill"
version = "1.0.0"
description = "Development workflow skills"
license = "Apache-2.0"

[dependencies]
ora = ">=0.8.0"

[[skills]]
key = "review"
path = "assets/skills/review"

[[skills]]
key = "testing"
path = "assets/skills/testing"
```

每项声明包含：

- `key`：该 Skill 在插件内的稳定身份；
- `path`：该 Skill 根目录相对于当前插件版本根目录的路径。

Skill 的 `name` 和 `description` 只读取对应 `SKILL.md`，不在 `orax.toml` 中重复。这样插件级清单只负责声明“有哪些 Skill 以及它们在哪里”，Skill 自身清单继续作为内容元数据的唯一来源。

`key` 在同一个插件内必须唯一。插件升级时，即使 Skill 改名或移动目录，只要它仍表示同一个逻辑 Skill，就必须保留原 key；删除旧 key 并增加新 key 表示删除旧 Skill 并创建另一个 Skill。

`path` 必须是 `assets/skills/` 下的安全相对目录，不能使用绝对路径、`..` 或符号链接逃逸插件版本目录。不同 path 不得重复或互相嵌套；每个 path 根目录必须存在大小写精确的 `SKILL.md`，其内部不能再嵌套另一个 `SKILL.md`。

插件包中出现未声明的 `SKILL.md`、重复 key、重复 path、非法 Skill 清单或重复最终调用名时，整个插件版本均视为无效，不能只跳过其中一个 Skill 后继续安装。

## 代码与数据位置

Plugin Skill 的文件始终保留在所属插件的不可变安装版本中：

```text
~/.ora/plugins/installed/<namespace>/<plugin_name>/<version>/
└── assets/skills/<skill_path>/
```

Skill 模块从插件系统取得当前已安装版本的可信根目录，再使用 `orax.toml` 中经过校验的相对 path 定位 Skill。Skill 模块不自行选择插件版本，也不自行拼接插件安装根路径。

Plugin Skill 不复制到 local Skill 目录：

```text
~/.ora/atoms/skills/<name>/
```

后者只保存用户创建、手动导入、SkillHub 导入以及用户显式复制出来的可编辑 Skill。把 Plugin Skill 原件复制到这里会产生两个事实源，也会让插件升级、卸载和只读语义失去确定性。

## Skill 身份与目录记录

每个 Plugin Skill 都进入 Ora 统一的 Skill 目录并持久化到数据库，但数据库只保存其目录信息和逻辑定位，不保存插件安装绝对路径。

Plugin Skill 的 namespace 使用所属插件的 canonical plugin ID；用户创建和导入的 Skill 继续使用 `local`。Skill 来源必须作为明确类型保存，不能只根据 namespace 字符串推断所有权：

```text
Local

Plugin
├── plugin_id
├── contribution_key
└── relative_path
```

Plugin Skill 的 SkillId 由 canonical plugin ID 与 contribution key 稳定生成。同一个插件的同一个 key 在不同机器、插件升级、卸载重装后都得到相同 SkillId。name、description、path 和插件版本变化不能改变这个身份。

数据库中的 Plugin Skill 行是可查询、可选择的目录记录；实际内容仍以当前插件版本内的 Skill 包为准。Plugin Skill 的可编辑性由来源决定，不作为可随意切换的独立状态：

```text
Local  -> 可编辑
Plugin -> 插件管理、只读
```

## 安装与解析

插件安装完成后，插件系统把可信的插件身份、版本和版本根目录交给 Skill 模块。Skill 模块按以下顺序接入贡献：

```text
读取 orax.toml 的全部 [[skills]]
        ↓
校验 key、path 和完整目录边界
        ↓
解析每个 SKILL.md
        ↓
计算稳定 SkillId 和最终调用名
        ↓
将全部贡献登记到统一 Skill 目录
```

解析是整包操作。所有贡献全部有效时才能登记；任一贡献无效，整个插件版本都不能作为 Skill 插件使用。

首次安装后的插件默认禁用。此时 Skill 目录中可以保留已解析的贡献及其稳定身份，但这些 Skill 不进入可用列表，也不能被运行。

## 读取与物化

Skill 的调用方不应区分 local 和 plugin 路径。Skill 模块在统一的包访问 interface 后根据来源选择实际位置：

```text
Local Skill
    -> ~/.ora/atoms/skills/<name>/

Plugin Skill
    -> 当前插件版本根目录 + relative_path
```

Get、List、Workflow 物化和复制为 local 都通过这一个 interface 访问 Skill 包。调用方只提供 SkillId 和目标操作，不取得插件安装绝对路径，也不能自行拼接路径。

物化 Plugin Skill 时必须复制该 Skill 根目录的完整内容，而不是只复制 `SKILL.md`。物化副本可以根据 Agent 的发现规则重写 manifest name，但不能修改插件安装目录中的原件。

## 启用与调用名

启用 Skill 插件表示一次启用它声明的全部 Skill。v1 不保存每个 contribution 的独立启用状态。

最终调用名按 Ora 统一规则从 Skill name 规范化：

```text
lowercase(name).replace("_", "-")
```

所有当前可用 Skill 的最终调用名必须全局唯一。因此 `foo_bar` 与 `foo-bar` 会发生冲突，即使它们来自不同 namespace、不同插件，或者一个来自 local、一个来自插件。

启用前，Ora 必须同时检查插件内全部 Skill 与当前可用 Skill 的调用名。任一冲突都会使整个插件启用失败；Ora 不采用 local 优先、插件优先或后启用者覆盖等隐式规则。

普通聊天仍使用 `/name` 调用 Skill，不引入 `plugin/name` 形式。全局唯一约束保证同一个调用名在当前环境中只有一个明确目标。

## 生命周期

### 禁用

禁用插件时，它贡献的全部 Skill 同时变为不可用。数据库记录、稳定 SkillId 和 Workflow 引用继续保留；再次启用时重新校验插件包和调用名冲突。

### 升级

插件升级后，Ora 按 contribution key 同步 Skill：

```text
相同 key -> 更新 name、description 和 path，保留 SkillId
新增 key -> 创建新的 Skill
删除 key -> 将旧 Skill 标记为已删除
```

如果插件当前已启用，新版本在生效前必须重新检查全部调用名。新版本产生冲突时，不能让新旧 Skill 目录混合生效。

### 卸载

卸载插件时，它贡献的全部 Skill 从可用目录移除并软删除。Workflow 中的引用不被静默修改；以后重装相同 plugin ID 和 key 时恢复相同 SkillId。

### 损坏

如果当前插件版本中的任一声明 Skill 缺失、清单非法或越过目录边界，整个 Skill 插件的贡献都变为不可用。Ora 不能继续使用数据库中的旧元数据假装文件仍然有效，也不能只忽略坏 Skill 后继续使用其余部分。

## 只读与复制

Plugin Skill 的 Update 和 Delete 必须由 Ora 后端拒绝，不能只在界面上隐藏编辑按钮。插件升级是修改 Plugin Skill 原件的唯一正常途径。

用户需要修改 Plugin Skill 时，可以显式“复制为 local”。复制操作必须：

1. 要求用户确认一个新的、调用名不冲突的 local name；
2. 复制 Skill 的完整目录树；
3. 重写副本根 `SKILL.md` 中的 name；
4. 创建新的 local SkillId 和 `namespace = local` 记录。

复制后的 Skill 是独立的 local Skill，可以编辑和删除，不再跟随原插件升级、禁用或卸载，也不与原件维护差异或合并关系。

## Workflow 引用

新保存的 Workflow 使用稳定 SkillId 引用 Skill，而不是只保存 Skill name。这样插件升级时即使 Skill 改名或移动目录，只要 contribution key 不变，已有 Workflow 仍然指向同一个 Skill。

旧 Workflow 中已经保存的 name 或 `prefix:name` 继续兼容读取；用户下一次保存时可以转换为真实 SkillId，不需要在启动时批量改写历史配置。

插件被禁用、卸载、损坏或删除 contribution 时，Ora 保留 Workflow 引用，并在运行前报告该 Skill 不可用及其具体原因。禁用或卸载前可以提示受影响的引用，但不自动删除引用，也不自动复制为 local。

## 展示与管理

local 和 Plugin Skill 在同一个 Skill 列表中展示、搜索和选择。Plugin Skill 需要显示所属插件、插件版本和只读状态；不可用时应区分插件禁用、已卸载、包损坏、贡献被移除和调用名冲突。

Plugin Skill 不显示编辑和删除操作，改为提供“复制为 local”。插件本身的启用、禁用、升级和卸载仍由插件管理入口负责。

## 实现边界

插件系统负责：

- 安装和验证插件版本；
- 提供 canonical plugin ID、版本、启用状态和可信版本根目录；
- 在安装、启用、升级、禁用和卸载时通知 Skill 模块同步贡献。

Skill 模块负责：

- 解析和校验 `[[skills]]` 与 `SKILL.md`；
- 维护 Plugin Skill 的稳定身份、目录记录和可用性；
- 统一解析 local 与 plugin Skill 包；
- 强制 Plugin Skill 只读；
- 检查最终调用名冲突；
- 为 Workflow 物化 Skill，并支持复制为 local。

PluginManager 如何把安装布局调整为 `~/.ora/plugins/installed/<namespace>/<name>/<version>/`、如何迁移旧目录、如何原子安装和选择版本，属于独立设计与提交，不在本文展开。
