# 已安装身份的持久化归属

## Uninstalling a Plugin Must Clear Every Ownership Row Its Full Plugin ID Wrote

### 风险

`skills` 表的活跃行以 `namespace = <完整 canonical Plugin ID>` 匹配，`effect_sources` 表的
`UNIQUE(effect_kind, source_kind, namespace, identifier)` 同样以完整 plugin id 为键。若卸载流程
只删除安装目录、不按同一把键清理这两张表，旧行会永远保持 active：卸载后重装同名插件会与旧行的
native identity（技能名本身，不含 namespace）冲突，导致该 Workspace 的 Skill 收敛拿不到干净的
Current；而如果旧行恰好不再冲突，则是另一种后果——无主的 ownership 永久残留，任何人都清不掉。

### 前置状态

- 一个 Skill 插件已安装，其 `skills` 行与对应的 `effect_sources` 行已写入，且已通过 Effect 收敛
  在某 Workspace 中物化出实际文件。
- 该插件的完整 canonical Plugin ID（`<namespace>/<identifier>`）是这些行的匹配键。

### 触发

对该插件调用卸载（`uninstall_plugin`），且 `data_disposition` 要求清理插件数据。

### 必须成立

- 卸载完成后，`skills` 表中该 plugin id 名下的全部行被清除。
- 卸载完成后，`effect_sources` 表中该 plugin id 名下的全部行被清除（连同其驱动的 Desired
  Projection 收敛为空，物化产物被移除）。
- 清理只按完整 plugin id 匹配，不影响其他 namespace 下同名插件或同 identifier 不同 namespace
  的插件残留的行。

### 禁止结果

- 卸载后安装目录已消失，但 `skills` 或 `effect_sources` 中仍保留该 plugin id 的行。
- 卸载清理误删了其他 namespace 下同 identifier 插件的行。
- 卸载不经过 Effect 收敛就直接 `DELETE` 持久化行，绕开 ownership 账本直接留下无主的物化产物。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
|---|---|---|
| 在 repository 层直接调用移除操作时，给定 plugin id 的 `skills` 行与其 Desired Effect 计数归零 | `ora_db::repository_tests::plugin_skill_projection_round_trips_and_is_removed_with_its_plugin` | `Partial` |
| 通过真实的 `uninstall_plugin` 后端流程（而非直接调用 repository 方法）触发上述清理 | 尚无直接证据；现有端到端卸载测试（`ora_backend::plugin::tests::rtk_hook_plugin_clean_data_directory_e2e`）针对 Hook 插件，不产生 `skills`/`effect_sources` 行 | `Missing` |
| 卸载清理后，`effect_sources` 表中该 plugin id 名下的行本身被移除（而不仅是其 Desired Effect 计数归零） | 尚无直接证据；现有测试只查询 `effect_desired_effects`/`effect_scopes`，未直接断言 `effect_sources` 行被删除 | `Missing` |
| 卸载清理不影响其他 namespace 下同 `identifier` 插件残留的行 | 尚无直接证据 | `Missing` |

### 决策依据

- 来源派生 Namespace 决策「namespace 一旦落盘就不可改变」一节：列出 `skills` 表、`effect_sources`
  表是完整 plugin id 匹配键的两处持久化位置，并说明匹配失败的具体后果。
- 来源派生 Namespace 决策不变量 6、7。

## Agent Identity and Session Resolution Must Key Off the Full Canonical Plugin ID

### 风险

`agent` kind 的运行期标识与 `sessions.agent_cli` 在本决策之前只使用 `identifier`（历史上的 name
段）。两个 source 各自发布一个同 `identifier` 的 agent 插件时，若运行期监督表、agent 选择器或
会话绑定中任何一处仍按 `identifier` 而不是完整 plugin id 匹配，两个包会在该处静默塌缩成一个：
安装期可能拒绝其中一个、监督表只留下按遍历顺序活下来的那个、已绑定某个 source 的会话可能在另一
同名 agent 被安装或监督顺序变化后被悄悄接管。

### 前置状态

- 两个不同的 marketplace source 各自发布一个 `identifier` 相同、namespace 不同的 `agent` 插件。
- 至少一个已安装并被监督；至少一个既有会话的 `sessions.agent_cli` 已绑定到其中一个的完整
  plugin id。

### 触发

- 场景一：两个包都已安装，宿主按已安装包目录的遍历顺序构建监督集合。
- 场景二：在已有会话绑定其中一个 agent 之后，安装另一个同 `identifier` 的 agent。

### 必须成立

- 两个包在监督集合、agent 选择器与会话绑定中始终是两个独立条目，键是各自完整的 canonical
  Plugin ID。
- 已安装集合内不会出现「同 identifier 两个包竞争一个监督槽位」的仲裁；监督集合恒等于已安装 agent
  集合。
- 已绑定某会话的 `sessions.agent_cli`（完整 plugin id）在同 `identifier` 的另一 agent 被安装、
  卸载或监督顺序变化后，仍然解析回原来的包，不被重新绑定到新包。
- 会话解析必须能独立于已安装包被遍历、发现或注册的顺序，稳定地找到 `agent_cli` 所命名的那个
  完整 plugin id 对应的包。

### 禁止结果

- 监督集合大小小于已安装 agent 数量（说明发生了按 identifier 的塌缩）。
- 已有会话在安装同 `identifier` 的另一 agent 后，`agent_cli` 解析到的实现发生变化。
- 会话对同一 `agent_cli` 值的解析结果随已安装包的发现顺序而不同。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
|---|---|---|
| agent 运行期标识使用完整 canonical Plugin ID，namespace 参与标识 | `ora_backend::agent_runtime::connection::tests::identifies_an_agent_by_its_whole_plugin_id` | `Covered` |
| 两个 source 发布的同 `identifier` agent 在标识函数层面保持不同标识 | `ora_backend::agent_runtime::connection::tests::keeps_same_identifier_agents_from_different_sources_distinct` | `Covered` |
| 会话按 `sessions.agent_cli` 命名的完整 plugin id 解析到正确的包，且结果与已安装包的发现/遍历顺序无关 | 尚无直接证据；现有测试只验证标识函数本身的纯计算结果，未验证真实会话解析路径 | `Missing` |
| 安装另一同 `identifier` agent 不会使既有会话的 `agent_cli` 解析改指向新包 | 尚无直接证据 | `Missing` |

### 决策依据

- 来源派生 Namespace 决策 D8：agent 标识使用 canonical Plugin ID，不再因 name 段相同而在安装期
  拒绝或产生身份仲裁。
- 来源派生 Namespace 决策「与 approved 决策的偏离」：agent 监督不再做身份仲裁，监督集合恒等于
  已安装的 agent 集合，这一推论本身需要独立的运行期证据，不能只由标识函数的纯计算测试代替。
- 来源派生 Namespace 决策不变量 10。
