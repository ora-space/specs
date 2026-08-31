# 插件身份与 Namespace

## Plugin Namespace Must Come From the Installing Source, Not the Package Manifest

### 风险

namespace 是插件身份的一半，并且直接构成三条文件系统路径、Skill 行主键与 Effect Consumer 的
storage key。如果它由第三方可编辑的清单内容决定，任何源都能声明任意 namespace，从而落进另一个
来源的目录、覆盖其配置、顶替其 Consumer 身份。清单内容是不可信输入，不能成为身份来源。

### 前置状态

- 至少两个已配置的 marketplace source，其中之一是默认源。
- 非默认源发布一个条目，其清单声明的 `identifier` 与官方源某条目相同。
- 宿主已为每个 source 绑定 namespace。

### 触发

从非默认源安装该条目，随后重启并重新 discover 已安装插件。

### 必须成立

- 安装后的 namespace 等于该 source 绑定的 namespace，与清单中出现过的任何 `namespace` 文本无关。
- namespace 是单个合法路径段（`[a-z0-9-.]`、非空、不超过 33 字节、不是 `.` 或 `..`）。
- 安装目录、data 目录、`store.json` 路径三者使用同一 namespace。
- 重启后的 discovery 从安装目录恢复出同一 namespace，而不是从包内清单推断。
- 包内清单声明的 name/version 与所在目录不一致时上报 discovery issue，插件不被静默归属。

### 禁止结果

- 清单中的 `namespace` 字段影响最终身份。
- 清单中的未知字段导致解析失败，使旧版本 Ora 因一个新增可选字段而丢弃整个条目。
- 未知字段被放宽的同时，必填字段缺失也被放宽为静默通过。
- 忽略残留的 `namespace` 字段时产生逐条告警。
- 派生出 `official` 或 `local` 这两个保留 namespace。
- 派生结果包含 `/`、`:`、大写字母或其他非法路径字符，使 `PluginId::parse` 与 `canonical()` 不再互逆。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
|---|---|---|
| `PluginId` 段拒绝非法字符，`parse` 与 `canonical` 互逆 | `ora_domain::plugin_id::tests::parses_canonical_form`、`ora_domain::plugin_id::tests::rejects_malformed_ids` | `Partial` |
| 安装后的 namespace 来自 source 绑定而非清单 | 尚无直接证据 | `Missing` |
| 携带未知字段的 marketplace 清单与安装包清单均可解析，该字段被忽略 | 尚无直接证据 | `Missing` |
| 必填字段缺失仍然是硬错误，并保留 TOML 路径 | `ora_plugin_manager::tests::reports_structural_errors_with_field_paths` 覆盖字段路径，但未覆盖放宽未知字段后的必填校验 | `Partial` |
| 派生结果始终是合法且不超过 33 字节的单一路径段 | 尚无直接证据 | `Missing` |
| 清单与安装目录不一致时上报 issue 而不猜测归属 | `ora_plugin_manager::tests::rejects_manifest_version_that_differs_from_directory` 证明 version 维度已覆盖，namespace 维度尚未 | `Partial` |

### 决策依据

- ADR 不变量 1：namespace 由宿主生成，永远不来自第三方可编辑的清单内容。
- ADR 不变量 4、6：namespace 必须是合法路径段；name/version/kind 与 namespace 的来源分离且不一致
  即为损坏。
- ADR D1：schema 增量由未知字段容忍承担，断裂性变化由 `resolver` 版本承担；必填字段不随之放宽。
- ADR 不变量 8：未知字段被忽略，必填字段缺失仍然是硬错误。

## Derived Namespace Must Stay Stable Across Equivalent URLs and Re-Adds

### 风险

安装目录冻结了已安装插件的 namespace，所以漂移不会改写任何持久化身份；但它会让插件与来源脱钩：
市场按新 namespace 生成条目，与已安装的旧 namespace 对不上，于是显示“未安装”、更新解析不到条目、
重新安装会多出一份。同一仓库的等价 URL 写法是最容易触发它的入口，因此归一与绑定必须成对存在。

### 前置状态

- 一个第三方 source 已绑定 namespace，并已从中安装至少一个插件。
- 该插件已写入配置（`store.json`）并产生 Skill 行与 Effect Consumer 状态。

### 触发

删除该 source，再用同一仓库的等价 URL 写法（大小写、`.git` 后缀、尾部斜杠、默认端口）重新添加。

### 必须成立

- 三种等价 URL 写法规范化到同一 canonical URL，派生同一 namespace。
- 等价写法重新添加后，已安装插件仍与其条目匹配，可以正常更新。
- 删除后重新添加同一 canonical URL，复用历史绑定，而不是生成第二个 namespace。
- 已安装插件的目录、配置路径、Skill 行与 Consumer storage key 在上述操作后保持不变。
- 派生函数是纯函数：同一输入在任何平台、任何版本得到同一输出。

### 禁止结果

- 每次读取都由当前 URL 现算 namespace，使绑定随规范化规则演进而漂移。
- 两个指向同一仓库的等价 URL 产生两个身份与两份安装。
- 为解决漂移而在启动时重命名已有目录或重写 DB 行。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
|---|---|---|
| 由 URL 派生本地位置是确定性的 | `ora_plugin_registry::source::tests::derives_checkout_dir_from_git_url` 证明检出目录派生确定，但未覆盖等价 URL 归一 | `Partial` |
| 等价 URL 写法规范化到同一 canonical URL | 尚无直接证据 | `Missing` |
| 绑定后的 namespace 不随规范化规则演进而改变 | 尚无直接证据 | `Missing` |
| 删除并重新添加同一 canonical URL 复用原绑定 | 尚无直接证据 | `Missing` |
| 派生函数跨平台稳定（含哈希后缀） | 尚无直接证据 | `Missing` |

### 决策依据

- ADR 不变量 2、3：派生函数是兼容性契约；绑定一旦写入即不可变。
- ADR D3：namespace 表达“首次引入该插件的来源”，不是“当前 URL”。

## Installed Plugin Namespace Must Never Change After It Lands on Disk

### 风险

namespace 是完整 plugin id 的一半，而 plugin id 是五处持久化状态的匹配键：安装目录、插件私有
数据目录（含 MCP 与全部插件配置）、`skills` 行、`effect_sources` 行、Effect Consumer identity。
其中 Skill 的写入与清理完全按 `namespace = <plugin id>` 匹配，一旦 namespace 改变，旧行既不会被
新 id 的写入覆盖，也不会被卸载清除，而 Skill 的 native identity 又不含 namespace —— 新旧两份
Desired Effect 会在同一 Workspace 撞上 native identity 冲突，使该 Workspace 的 Skill 收敛无法
到达干净的 Current。

### 前置状态

- 一个第三方 source 已绑定 namespace，并已从中安装至少一个插件。
- 该插件已写入配置、产生 Skill 行与 Effect Consumer 状态。

### 触发

改动 source 配置（删除后以等价 URL 重新添加）、规范化规则在新版本中扩展、以及进程重启。

### 必须成立

- 已安装插件的 namespace 由安装目录决定，不随 source 配置或规范化规则变化。
- 安装目录、插件私有数据目录、`skills` 行、`effect_sources` 行与 Consumer identity 在上述全部
  操作后保持不变。
- 配置确实变化时，最坏后果限于与来源脱钩：市场显示未安装、无法更新、可被重复安装一份。
- 派生函数是纯函数：同一 canonical URL 在任何平台、任何版本得到同一输出。

### 禁止结果

- 以就地 UPDATE 改写 `skills`、`effect_sources` 或 Consumer 行的 plugin id。
- 启动时重命名已有的安装目录或数据目录。
- 插件的 namespace 改变后，其私有数据目录被换成一个空目录。
- 旧 skills 行与 `effect_sources` 保持 active 且无法通过卸载清除。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
|---|---|---|
| 发现流程从安装目录而非清单确定 namespace | 尚无直接证据 | `Missing` |
| source 配置变化后已安装插件的持久化身份完全不变 | 尚无直接证据 | `Missing` |
| 卸载能清除该插件写下的全部 skills 行与 `effect_sources` | 尚无直接证据 | `Missing` |
| 同名 Skill 的 native identity 冲突被报告为 blocking condition 而非静默覆盖 | `ora_effect_skill` planner 冲突路径 | `Partial` |
| 派生函数跨平台稳定（含摘要后缀） | 尚无直接证据 | `Missing` |

### 决策依据

- ADR 不变量 7：已安装插件的 namespace 不因任何配置变化而改变。
- ADR D3 + D5：绑定不可变阻止身份在配置层漂移，目录权威保证身份落盘后不再由可变输入决定。
- ADR「namespace 一旦落盘就不可改变」：迁移的唯一正确形态是旧 id 完整卸载 + 新 id 重新安装。

## Listings From Different Sources Must Not Shadow Each Other

### 风险

当前索引合并按 `namespace/name` 去重且先出现的源胜出，而所有条目的 namespace 恒为 `official`：
排在前面的第三方源可以静默顶替官方条目，用户在市场里看不出条目来自哪个仓库，安装与后续更新都会
跟随顶替者。这是一条可被利用的供应链路径。

### 前置状态

- 两个 source 各自发布 `identifier` 相同的条目。
- 第三方源在优先级顺序上排在默认源之前。

### 触发

同步全部源、重建索引，随后打开市场并安装其中一个条目。

### 必须成立

- 两个条目以不同 `namespace/name` 同时出现在索引中，互不遮蔽。
- 每个条目可被单独定位到其来源；安装与更新始终跟随该条目自己的来源与代理策略。
- 展示层能区分同名不同源的条目（来源标识可见）。
- 同名 agent 插件在安装期被显式拒绝，而不是装上之后由 namespace 字母序决定谁被监督。
- 更新同一 agent 插件（相同 plugin id）不被该拒绝规则误伤。

### 禁止结果

- 一个源的条目因另一个源排在前面而从列表中消失。
- 用户安装 A 源的条目，后续更新却从 B 源拉取。
- 同名 agent 并存后，session 绑定到哪一个由启动顺序决定。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
|---|---|---|
| 跨源合并当前按 id 去重、先出现者胜出（本决策要替换的既有行为） | `ora_plugin_registry::index::tests::merges_multiple_sources_and_dedups_by_id` | `Covered`（针对旧语义；实现新决策时必须同步改写） |
| 安装解析遵循与索引一致的源顺序 | `ora_plugin_registry::index::tests::resolves_manifest_across_sources_in_source_order` | `Covered`（同上） |
| 同名不同源条目并存且互不遮蔽 | 尚无直接证据 | `Missing` |
| 索引条目携带归属来源，展示层据此区分同名条目 | 尚无直接证据 | `Missing` |
| 更新始终跟随条目自身来源 | 尚无直接证据 | `Missing` |
| 同名 agent 插件在安装期被拒绝，并指明冲突对象 | 尚无直接证据 | `Missing` |
| 相同 plugin id 的更新不触发该拒绝 | 尚无直接证据 | `Missing` |

### 决策依据

- ADR D6：不同源的同一 `identifier` 是两个身份，消歧发生在展示层而非索引合并层。
- ADR D8：agent 标识只携带 name 段，namespace 无法为其消歧，因此冲突必须在安装期被拒绝，而不是
  留给运行期按 namespace 字母序二选一。
