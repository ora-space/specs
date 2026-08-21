# Skill 类型插件设计

## 文档状态

本文定义 Ora Skill 类型插件的 v1 目标设计，用于约束后续的清单、领域模型、数据库、Skill 包访问、插件生命周期接入和用户界面实现。

本文描述的是目标行为，不表示相关能力已经在当前代码中实现。

## 范围

Skill 类型插件用于随一个 Ora 插件版本分发一个或多个只读 Skill。插件安装后，Ora 校验并登记这些 Skill；插件启用后，它们进入统一 Skill 目录，可被所有 Agent 和 Workflow 选择。

v1 包含：

- Skill 插件包格式与校验规则；
- Plugin Skill 的稳定身份、来源和只读语义；
- Plugin Skill 到统一 Skill 目录的登记方式；
- 安装、启用、禁用、升级、卸载和损坏恢复语义；
- 来源感知的 Skill 包访问 interface；
- Workflow、聊天、用户界面和错误语义；
- Plugin Skill 显式复制为 local Skill。

v1 不包含：

- Skill 插件的 `main.js`、Deno 运行时或 Ora SDK 调用；
- Setting、Secret、宿主权限或配置模板替换；
- 插件依赖和本机工具依赖的自动解析；
- 按单个 Skill 启用或禁用；
- Agent 白名单或按 Agent 类型限制 Skill；
- 区分用户手写、手动导入和 SkillHub 下载的 local Skill 来源。

## 前置条件

插件系统必须能够向 Skill 模块提供当前已安装插件版本的可信描述，包括：

```text
canonical plugin_id
plugin namespace
plugin name
plugin version
当前版本根目录
enabled 状态
```

已安装版本的规范位置为：

```text
~/.ora/plugins/installed/<namespace>/<name>/<version>/
```

Skill 模块不自行扫描多个版本、不选择当前版本，也不拼接上述安装根路径。PluginManager 如何安装、切换或回滚版本不属于本文范围。

## 核心原则

1. **纯数据插件：**Skill 插件只分发内容，不运行插件代码。
2. **插件包是事实源：**Plugin Skill 始终从当前已安装的不可变插件版本读取，不复制到 local Skill 根目录。
3. **数据库是目录索引：**数据库保存稳定身份、来源和逻辑定位，不保存插件安装绝对路径。
4. **原件只读：**Plugin Skill 不能被通用 Update/Delete 修改；用户只能将其复制成独立 local Skill 后编辑。
5. **整包一致：**一个 Skill 插件内任一贡献无效，整个版本均无效，不允许部分安装或部分启用。
6. **稳定引用：**插件升级时，只要 `plugin_id` 和 contribution `key` 不变，SkillId 和已有 Workflow 引用就不变。
7. **可执行名唯一：**所有当前可用 Skill 的最终 `/alias` 全局唯一，不使用隐式覆盖或来源优先级。

## 插件包规范

### 标准布局

一个 Skill 插件包必须采用以下布局：

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
            ├── SKILL.md
            └── references/
```

根目录文件规则：

- `orax.toml`、`logo.svg`、`README.md` 均为必需文件；
- Skill 插件不得以 `main.js` 作为入口，也不声明或启动 Deno 运行时；
- `logo.svg` 必须通过 Ora 统一的 SVG 安全校验；
- `README.md` 是插件级说明，不替代各 Skill 的 `SKILL.md`；
- 所有 Skill 根目录必须位于 `assets/skills/` 下。

### `orax.toml`

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

`kind = "skill"` 表示该插件是排他的纯数据 Skill 插件。v1 不允许同一个插件同时作为 Agent、Workbench 或其他运行时插件。

每个 `[[skills]]` 条目只声明：

- `key`：插件内部稳定 contribution 身份；
- `path`：该 Skill 根目录相对于插件版本根目录的安全相对路径。

Skill 的 `name` 和 `description` 只从对应的 `SKILL.md` front matter 读取，不在 `orax.toml` 中重复。

### contribution key

`key` 必须满足：

- 在同一个插件内唯一；
- 使用规范化、大小写不敏感的 ASCII slug；
- 不包含路径分隔符、父目录引用或空白；
- 发布后作为稳定身份使用，不因 Skill 的显示名或目录移动而改变。

作者如果删除旧 key 并新增另一个 key，Ora 必须将其解释为删除旧 Skill 并创建新 Skill，而不是重命名。

### contribution path

`path` 必须满足：

- 是规范化安全相对路径；
- 位于 `assets/skills/` 下；
- 指向普通目录，不能通过绝对路径、`..` 或符号链接逃逸插件版本目录；
- 在同一个插件内唯一，且不同 contribution path 不得重叠；
- 根目录存在大小写精确的 `SKILL.md`；
- 根目录内部不得再出现另一个 `SKILL.md`。

Skill 根目录可以包含 `scripts/`、`references/`、`assets/` 及其他普通辅助文件。完整目录树属于该 Skill 包，后续物化或复制时必须整体处理，不能只复制 `SKILL.md`。

### `SKILL.md` 与 alias

每个 `SKILL.md` 必须通过现有 Skill manifest 规则，包括合法的 `name`、非空 `description`、大小限制和 YAML front matter 校验。

最终运行 alias 按统一规则从 `name` 计算：

```text
runtime_alias = lowercase(name).replace("_", "-")
```

例如：

```text
SFMEA_Review -> sfmea-review
foo_bar      -> foo-bar
foo-bar      -> foo-bar
```

同一个插件包内的 `key`、`path`、Skill `name` 和最终 `runtime_alias` 必须分别唯一。`foo_bar` 与 `foo-bar` 因最终 alias 相同而发生冲突。

### 整包校验

安装或升级前，Ora 必须先完成以下校验：

1. 插件根必需文件完整；
2. `orax.toml` 是受支持的 `kind = "skill"` 清单；
3. 所有 `key` 和 `path` 合法且唯一；
4. 所有 path 均位于允许的目录内且互不重叠；
5. 每个 `SKILL.md` 均可解析；
6. 没有嵌套或未声明的 `SKILL.md`；
7. 包内 Skill name 和最终 alias 无冲突；
8. 文件数、文件大小、清单大小和总解压量符合统一资源限制。

任一检查失败，整个插件版本均返回 `invalid_plugin_skill_manifest` 或 `plugin_skill_package_invalid`，不能只跳过坏 Skill 后继续安装。

## 领域模型

### Skill 来源

Skill 必须具有一等来源模型：

```text
SkillSource
├── Local
└── Plugin
    ├── plugin_id
    ├── contribution_key
    └── relative_path
```

来源决定管理策略：

```text
Local  -> Editable
Plugin -> PluginManaged
```

`editable` 不作为可以独立修改的布尔状态保存，避免出现 `source_kind = plugin` 但 `editable = true` 的非法组合。

### Namespace

local Skill 使用：

```text
namespace = local
source_kind = local
```

Plugin Skill 使用：

```text
namespace = canonical plugin_id
source_kind = plugin
owner_plugin_id = canonical plugin_id
```

`namespace` 用于表达可信所有者；`source_kind` 和 `owner_plugin_id` 用于强类型校验和生命周期查询。实现不能只根据 namespace 字符串猜测来源。

用户手写、手动导入和 SkillHub 下载的 Skill 在 v1 中都属于 `local`，保持现有可编辑语义。

### 稳定 SkillId

Plugin Skill 的 SkillId 使用固定 UUID namespace 下的 UUIDv5，根据以下稳定输入生成：

```text
canonical plugin_id + contribution_key
```

实现必须使用无歧义编码组合两个字段，并永久冻结 UUID namespace 常量和编码规则。相同插件和 key 在不同机器、重装和升级后必须产生相同 UUID。

以下变化不能改变 SkillId：

- `SKILL.md` 中的 name 或 description 修改；
- contribution path 移动；
- 插件版本升级；
- 插件卸载后重新安装。

改变 `plugin_id` 或 contribution key 表示创建新的 Skill 身份。

## 数据库存储

### 统一 Skill 目录

Plugin Skill 继续进入统一 `skills` 表，不建立独立的 `plugin_skills` 目录表。目标字段为：

```text
id
namespace
name
description
source_kind          local | plugin
owner_plugin_id      Plugin Skill 必填
contribution_key     Plugin Skill 必填
relative_path        Plugin Skill 必填
runtime_alias        当前可用时的规范化执行名
created_at
updated_at
is_deleted
```

约束：

- local 行的插件来源字段必须为空；
- plugin 行的 `owner_plugin_id`、`contribution_key`、`relative_path` 必须同时存在；
- `(owner_plugin_id, contribution_key)` 唯一，包括软删除记录；
- Plugin Skill 的 namespace 必须等于其 canonical owner plugin ID；
- `relative_path` 只能保存已校验的包内逻辑路径，禁止保存安装绝对路径；
- 当前可用且未删除记录的 `runtime_alias` 全局唯一；
- 首次安装但尚未启用的 Plugin Skill 已有目录记录，但没有可用 runtime alias。

数据库行是用于查询、选择和引用的目录索引。插件版本根目录、当前版本选择和安装完整性仍由插件系统负责。

### 软删除与恢复

卸载插件或升级删除一个 contribution 时，Ora 软删除对应 Skill 行，不删除稳定身份。

以后重新安装相同 `plugin_id + key` 时，Ora 必须恢复原行并复用相同 SkillId，而不是插入一个新的随机身份。

## 公共合约

公共 Skill 合约必须暴露强类型来源和管理策略，概念结构为：

```text
source:
  Local
  Plugin {
    pluginId,
    pluginVersion,
    contributionKey
  }

management:
  Editable
  PluginManaged
```

前端需要知道：

- Skill 属于 local 还是某个插件；
- owning plugin 的显示身份和当前版本；
- Skill 是否可编辑；
- Skill 当前是否可用以及不可用原因。

前端不需要也不能获得：

- 插件安装绝对路径；
- contribution relative path；
- PluginManager 内部版本选择信息。

## 来源感知的 Skill Package 模块

### Seam

Skill 包访问必须集中在一个来源感知的深模块 interface。Get/List、Workflow 物化、复制为 local 和启动 reconciliation 都通过该 interface 访问包，调用方不能自行判断来源或拼接路径。

模块内部根据领域来源选择 adapter：

```text
Local adapter
  -> ~/.ora/atoms/skills/<name>/

Plugin adapter
  -> 插件系统提供的版本根目录 + relative_path
```

这是一个真实 seam：local 和 plugin 是两个具有不同所有权、可写性和定位规则的 adapter。

### Interface 行为

interface 至少提供以下高层行为，而不是暴露原始文件系统句柄：

```text
load(skill_id)
  -> 已解析的 manifest、正文、来源、可用性

materialize(skill_id, destination)
  -> 校验来源、复制完整包、按最终 alias 重写物化副本的 manifest name

copy_as_local(skill_id, new_name)
  -> 复制完整包、重写根 manifest、创建独立 local Skill

reconcile_plugin(plugin_descriptor)
  -> 校验整包贡献并幂等同步统一 Skill 目录
```

模块必须隐藏：

- local/plugin 路径解析；
- 插件身份与相对路径归属验证；
- `SKILL.md` 解析；
- 包可用性和损坏判断；
- 完整目录复制；
- 物化时的 name 重写；
- Plugin Skill 复制为 local 时的原子性。

模块的 interface 同时是主要测试 surface。测试应断言可观察结果，不能依赖调用方知道内部目录布局。

## 生命周期

### 安装

插件系统向 Skill 模块提供已安装版本的可信描述和版本根目录。Skill 模块：

1. 解析 `orax.toml` 中全部 `[[skills]]`；
2. 校验完整插件包和全部 Skill；
3. 为每个贡献计算稳定 SkillId；
4. 在统一 Skill 目录中幂等 upsert；
5. 软删除该插件旧目录中已不再声明的 contribution；
6. 保持新安装插件默认禁用，因此贡献初始为 unavailable。

安装不会把 Skill 文件复制进 `~/.ora/atoms/skills/`。

### 启用

Skill 插件只支持插件级整体启用：

1. 重新确认当前插件包完整有效；
2. 计算全部 contribution 的最终 runtime alias；
3. 与所有当前可用 local Skill 和其他 Plugin Skill 做全局冲突检查；
4. 在一次一致性操作中启用全部 alias；
5. 任一冲突则整个启用失败，不允许部分 Skill 可用。

冲突错误必须列出冲突 alias 及现有 Skill 的来源，不能采用 local 优先、插件优先或后启用者覆盖。

### 禁用

禁用插件时：

- 全部 contribution 同时变为 unavailable；
- 数据库行、稳定 SkillId 和用户引用保留；
- 不删除或改写 Workflow；
- 再次启用时重新校验包完整性和 alias 冲突。

### 升级

升级后的 contribution 按 key reconciliation：

```text
相同 key -> 更新 name、description、relative_path，保留 SkillId
新增 key -> 创建新 Skill
删除 key -> 软删除旧 Skill
```

如果插件当前已启用，新版本的全部 alias 必须在生效前重新检查。任一新冲突都必须拒绝新版本的 Skill 目录生效，并保留旧版本语义。插件版本切换和回滚的具体实现不属于本文范围。

### 卸载

卸载时：

- owning plugin 的全部 Skill 行软删除；
- 全部 contribution 立即不可用；
- Workflow 等持久引用保留；
- 重装相同 plugin ID 和 key 时恢复相同 SkillId。

### 损坏

如果当前插件包中任一已声明 Skill 缺失、非法、路径逃逸或完整性失效：

- 整个 Skill 插件的全部 contribution 标记为 unavailable/invalid；
- 不继续使用数据库中的旧 manifest 内容冒充可用包；
- 不只隔离单个坏 Skill；
- 用户界面提示重新安装或修复插件。

修复后必须重新执行整包校验和 alias 冲突检查，成功后才恢复可用性。

## 只读与复制

### 后端强制只读

通用 Skill Update/Delete 对 `source_kind = plugin` 必须返回 `plugin_skill_read_only`。只在前端隐藏按钮不能构成不可变保证。

插件升级是修改 Plugin Skill 的唯一正常入口。

### 复制为 local

用户可以显式将 Plugin Skill 复制成 local Skill：

1. 用户确认一个新的 local name，界面默认建议 `<name>-copy`；
2. 新名称必须通过 Skill name 校验；
3. 规范化后的 runtime alias 必须全局唯一；
4. Ora 复制完整 Skill 目录树到 local Skill staging；
5. Ora 将副本根 `SKILL.md` 的 name 改为新名称；
6. Ora 创建新的 local SkillId 和 `namespace = local` 记录；
7. 文件和数据库按现有 local Skill 原子写入语义提交。

复制后的 local Skill：

- 可编辑、可删除；
- 不再属于原插件；
- 不跟随插件升级、禁用或卸载；
- 不保留自动合并或差异覆盖关系。

## Workflow 与聊天

### Workflow 引用

新保存的 Workflow 必须持久化真实稳定 Skill UUID，不能继续把 Skill name 写进名为 `skillId` 的字段。

读取现有 Workflow 时采用惰性兼容：

1. 优先把值解释为真实 SkillId；
2. 找不到时兼容旧 name；
3. 继续兼容旧 `prefix:name` 形式的后缀解析；
4. 用户下一次成功保存该 Workflow 时写回真实 UUID；
5. 不在启动时批量改写历史 Workflow JSON。

插件改名或移动 path 时，只要 key 不变，旧 Workflow 继续指向同一 Skill。

### 不可用引用

插件被禁用、卸载、损坏或删除 contribution 时：

- 不自动删除 Workflow 中的引用；
- 不自动复制为 local；
- 运行前返回明确的 `plugin_skill_unavailable`；
- 错误携带 plugin ID、SkillId 和 disabled/uninstalled/invalid/removed 等稳定原因。

禁用或卸载前，Ora 应显示受影响的 Workflow 引用数量并警告用户，但用户确认后仍允许继续。

### 聊天 alias

普通聊天继续使用 `/name` 形式。因为所有当前可用 Skill 的最终 runtime alias 全局唯一，所以不引入 `plugin/name` 限定语法，也不存在隐式来源优先级。

## 用户界面

local 和 Plugin Skill 在同一 Skill 列表中展示、搜索和选择。

Plugin Skill 显示：

- owning plugin 名称；
- 当前插件版本；
- 只读标记；
- available/unavailable 状态及原因。

Plugin Skill 不展示编辑和删除操作，改为展示“复制为 local”。插件启用、禁用和卸载仍由插件管理界面负责。

至少区分以下不可用原因：

```text
plugin_disabled
plugin_uninstalled
plugin_invalid
contribution_removed
alias_conflict
```

`NeedsConfiguration` 不适用于 v1 Skill 插件，因为它不支持 Setting。

## 错误语义

稳定错误类别：

```text
invalid_plugin_skill_manifest
plugin_skill_read_only
plugin_skill_alias_conflict
plugin_skill_unavailable
plugin_skill_package_invalid
plugin_skill_copy_name_conflict
```

### `invalid_plugin_skill_manifest`

用于 `orax.toml` 的 skills 声明、key、path、必需根文件或 `SKILL.md` 结构不符合规范。

### `plugin_skill_read_only`

用于调用通用 Update/Delete 修改 Plugin Skill。

### `plugin_skill_alias_conflict`

用于启用、升级、local 创建/导入或复制时出现全局最终 alias 冲突。错误应返回冲突 alias 和占用者来源。

### `plugin_skill_unavailable`

用于持久引用存在，但 owning plugin disabled、uninstalled、invalid 或 contribution 已移除。

### `plugin_skill_package_invalid`

用于已安装插件包缺失、损坏、越界或完整性失效。

### `plugin_skill_copy_name_conflict`

用于复制为 local 时，新名称或最终 alias 已被占用。

## 验收场景

### 包解析

- 正确解析包含多个显式 contribution 的 Skill 插件；
- 完整保留每个 Skill 的 scripts、references、assets 和其他普通文件；
- 拒绝缺少 `orax.toml`、`logo.svg` 或 `README.md` 的包；
- 拒绝重复或非法 key；
- 拒绝重复、重叠、越界或不在 `assets/skills/` 下的 path；
- 拒绝 path 根缺少大小写精确 `SKILL.md` 的贡献；
- 拒绝嵌套或未声明的 `SKILL.md`；
- 拒绝非法 Skill manifest；
- 拒绝包内重复最终 alias；
- 任一贡献无效时拒绝整个包。

### 身份与数据库

- 相同 plugin ID 和 key 在不同机器生成相同 UUID；
- 插件升级修改 name、description 或 path 后 SkillId 不变；
- 删除 key 后对应行软删除；
- 同 key 重新出现或重装插件后恢复原 SkillId；
- local 与 Plugin Skill 的来源字段满足各自不变量；
- 数据库不保存插件安装绝对路径。

### 生命周期

- 安装后目录记录存在，但插件默认禁用且 Skill unavailable；
- 启用后一次激活全部 contribution；
- local、其他插件以及 `_`/`-` 规范化冲突都阻止整体启用；
- 禁用保留 Skill 行和 Workflow 引用；
- 升级新增、更新和删除 key 的结果符合 reconciliation 规则；
- 新版本 alias 冲突时旧版本 Skill 语义保持不变；
- 卸载软删除全部贡献；
- 任一已安装 Skill 损坏时整包不可用；
- 修复后只有通过完整校验和冲突检查才恢复。

### 读取、物化与复制

- Get/List 通过来源感知模块读取 local 和 plugin 包；
- Workflow 从插件版本根目录物化完整 Skill 包；
- 调用方不自行拼接插件路径；
- Plugin Skill Update/Delete 被后端拒绝；
- 复制为 local 后文件完整、manifest name 正确、SkillId 独立且可编辑；
- 复制后的 local Skill 不受原插件升级、禁用和卸载影响。

### Workflow 与界面

- 新 Workflow 保存真实 Skill UUID；
- 旧 name 和 `prefix:name` 引用仍可解析，并在下一次保存时转为 UUID；
- 不可用 Plugin Skill 返回具体原因，不被静默删除；
- 禁用和卸载前展示受影响引用警告；
- Skill 列表正确展示插件来源、版本、只读和不可用状态；
- 聊天 `/alias` 在所有可用 Skill 中保持唯一。

## 实现边界总结

本文只定义 Skill 类型插件及其进入统一 Skill 模块后的语义。

以下工作应由独立 PluginManager 变更负责，不能混入本设计的实现提交：

- 将安装布局调整为 `~/.ora/plugins/installed/<namespace>/<name>/<version>/`；
- 旧 `installed/<name>/` 数据迁移；
- 下载、staging、原子安装和目录切换；
- 当前版本选择与版本回滚；
- 插件安装根的统一解析。

Skill 插件实现只依赖插件系统提供的可信当前版本根目录和插件描述，不复制、猜测或重新实现这些职责。
