# 插件来源展示与市场标签

依据[已批准的插件来源展示决策](../../../../decisions/desktop/plugin/registry/20261127-plugin-origin-presentation.md)。
实现尚未合入 desktop 的 main，下表义务的证据状态一律如实标为 `Missing`，证据列一律为“尚无”；
具体测试留到落地运行按合并后的提交填写，ADR 转为 `implemented` 时按合入后的最终测试逐项核对并回填状态。
[身份与 namespace 用例](identity-and-namespace.md)的证据保持各自范围，不推导本文新义务。

## Plugin Origin Is a Plugin-Level Attribute Carried by Every Origin-Bearing Response

### 风险

来源由 namespace 判定，语义是“首次引入该插件的来源”，不是当前市场源配置。如果实现按当前源列表判定，
或让前端从 namespace、plugin ID 推断，市场源被删除、禁用后来源会漂移，同一插件在不同界面显示不同来源；
如果来源挂在 Skill 上，同一插件的两个 Skill 有机会显示不同来源。本地 Skill 不属于任何插件，误归入插件
来源会让用户以为卸载某个插件能移除它。

### 前置状态

- 已安装插件覆盖三类 namespace：`official`（官方市场）、`local`（本地导入的 `.orax`）、派生 namespace
  （第三方市场）。
- 存在用户本地 Skill（`SkillSource::Local`）与各插件提供的 Skill。
- 市场列表含官方源与第三方源的条目。

### 触发

返回 `Skill`、`SkillDetails`、`InstalledPlugin`、`AvailablePlugin` 的各查询：Skill 列表与详情、已安装
插件列表及扫描、激活和停止响应、市场列表和索引同步响应。

### 必须成立

- `local` 判为本地包，`official` 判为官方，其余 namespace 判为第三方市场；判定只看 namespace，不看当前
  市场源配置。
- 插件提供的 Skill 与所属插件来源相同；本地 Skill 仍由 `SkillSource::Local` 表达，不是 `PluginOrigin`
  的变体。
- 所有上述响应都由后端填好 `origin`，复用同一套来源判定与标签规则。
- `AvailablePlugin` 只出现官方与第三方市场两类；顶层 `source_url` 保留，继续用于搜索。

### 禁止结果

- 市场源被删除、禁用或插件在当前市场里找不到时，已安装插件的来源改变。
- 前端从 namespace 或 plugin ID 推断来源，或显示原始 namespace。
- 同一插件的两个 Skill 来源不同；`AvailablePlugin` 出现本地包。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 保留与绑定的 namespace 按四类判定，第三方带 URL 与标签 | 尚无 | Missing |
| 插件 Skill 继承所属插件的来源 | 尚无 | Missing |
| 市场条目装配 origin，顶层 `source_url` 保留 | 尚无 | Missing |
| origin 的前后端序列化形状固定（kind 判别、camelCase 字段） | 尚无 | Missing |
| 扫描、激活/停止、索引同步等其余响应同样装配 origin | 尚无 | Missing |

### 决策依据

D1、D2 规则 1–3；不变量 1、2。来源语义继承来源派生 Namespace 决策 D3（首次引入该插件的来源）。

## Marketplace Labels Are Computed at Query Time and Unique Within the Comparison Set

### 风险

标签是用户区分不同来源同名插件的唯一手段。比较集合必须是全部绑定加默认 marketplace 的 canonical URL：
少了已删除源的绑定，两个已安装插件可能撞标签；少了默认源，官方市场的 fork 只显示 `marketplace`，和“官方”
放在一起看不出是谁的 fork。计算若理解 URL 结构、错位比较或撞名不回落，同样会让两个来源显示相同标签。

### 前置状态

比较集合为绑定表全部绑定的 canonical URL 加默认 marketplace 的 canonical URL；D3 例表的每一行是一组输入。

### 触发

后端在查询时对比较集合计算标签。

### 必须成立

- 仓库名唯一时直接取最后一段：`github.com/acme/awesome-plugins` → `awesome-plugins`。
- 同名 fork 用相邻段区分：`github.com/acme/marketplace`、`github.com/obsismc/marketplace` →
  `acme/marketplace`、`obsismc/marketplace`。
- 默认源 fork 行：默认源 `github.com/ora-space/marketplace` 与 `github.com/obsismc/marketplace` 并存时，
  fork 显示 `obsismc/marketplace`；默认源自身仍显示“官方”，不产生第三方标签。
- 同路径不同 host 带出 host：`github.com/acme/tools`、`gitlab.com/acme/tools` →
  `github.com/…/tools`、`gitlab.com/…/tools`。
- 非默认端口参与比较：`git.example.com:8443/scm/infra/registry`、`git.example.com/scm/infra/registry` →
  `git.example.com:8443/…/registry`、`git.example.com/…/registry`。
- 无单个区分段时回落完整 URL：`github.com/a/x/m`、`github.com/a/y/m`、`github.com/b/x/m` →
  `github.com/a/x/m`、`y/m`、`b/…/m`。
- 规则 5：不同位置选中的同名段渲染成相同文本时整组回落：`github.com/p/x/a/m`、`github.com/x/b/a/m`、
  `github.com/q/b/c/m` 前两个按规则 2 都得 `x/…/m`，整组回落为各自的完整 URL。
- 平台字面量照样显示：`dev.azure.com/contoso/a/_git/registry`、`dev.azure.com/contoso/a/scm/registry` →
  `_git/registry`、`scm/registry`。
- 段数不同的 URL 按从末尾对齐的位置比较；本来源在某位置没有段时跳过该位置继续往前。

### 禁止结果

- 同一比较集合内两个来源的显示标签相同。
- 把某一段解释为组织、项目或平台字面量而跳过或特殊处理。
- 默认源产生第三方标签，或其 canonical URL 缺席比较集合。
- 标签写入任何表或缓存。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 唯一仓库名取最后一段 | 尚无 | Missing |
| 同名 fork 按相邻段区分 | 尚无 | Missing |
| 默认源 fork 行：fork 显示 `obsismc/marketplace`，默认源自身“官方” | 尚无 | Missing |
| 同路径不同 host 带出 host | 尚无 | Missing |
| 非默认端口保留在标签中 | 尚无 | Missing |
| 无单个区分段回落完整 URL（规则 4） | 尚无 | Missing |
| 显示文本撞名整组回落（规则 5） | 尚无 | Missing |
| 平台字面量按普通段比较、照样显示 | 尚无 | Missing |
| 不同深度按末尾对齐，本来源缺段的位置跳过 | 尚无 | Missing |

### 决策依据

D3 分段、比较集合、计算规则 1–5 与例表；不变量 3、5。段比较不理解 URL 结构继承来源派生 Namespace 决策 D2。

## All Origin-Bearing Responses Agree on the Label for the Same Source

### 风险

`/` 菜单、技能页、市场卡片与已安装插件列表若各自计算，或读到不同的绑定集合，同一来源会在不同界面显示
不同标签，用户无法把三处的同名插件对应起来。标签若被缓存，新增绑定后不会立即更新。

### 前置状态

同一份绑定集合（加默认 marketplace 的 canonical URL）；同一来源的插件同时出现在 Skill 列表、已安装插件
列表与市场列表中。

### 触发

分别发起各带 `origin` 的查询。

### 必须成立

- 每个响应读取一次全部绑定，补入默认源的 canonical URL，构造一份快照，响应内所有条目用这份快照计算。
- 同一份比较集合下，所有带 `origin` 的响应对同一来源给出相同标签。
- 标签在每次查询时计算：新增绑定后的下一次查询即用新的比较集合，不需要重启。

### 禁止结果

- 同一份比较集合下，同一来源在不同响应中标签不同。
- 响应使用过期快照，或把标签写入表、缓存供后续响应复用。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 各响应对同一来源装配相同标签 | 尚无 | Missing |
| 标签不持久化，新增绑定后下一次查询即生效 | 尚无 | Missing |

### 决策依据

D3 时机与身份边界；不变量 3。

## Missing or Unreadable Bindings Degrade Origin Without Failing the Query

### 风险

绑定表只增不删、安装前必已写入，但损坏状态仍可能出现：某条绑定的 namespace 查不到，或读取绑定表本身
失败。若让一条损坏条目失败整个列表查询，用户看不到任何插件和 Skill；若降级时直接展示 namespace，摘要
后缀会进入界面，违反“不展示 namespace”的文案约束。

### 前置状态

一个派生 namespace 在绑定表中查不到；或读取绑定表本身失败。`official` 与 `local` 的插件不依赖绑定表。

### 触发

返回带 `origin` 条目的查询。

### 必须成立

- 绑定缺失：记录告警，该条目降级为第三方市场，`source_url` 为空，标签取 namespace 的 slug 段（`.` 之前
  的部分）；其余条目不受影响，列表查询不失败。
- 绑定表读取失败：记录告警，按空绑定集合计算（比较集合只剩默认 marketplace 的 canonical URL），所有派生
  namespace 都按上述规则降级，响应不因此失败。
- `official` 与 `local` 的判定不受绑定缺失或读取失败影响。

### 禁止结果

- 因一条损坏绑定或一次绑定表读取失败而让整个列表查询失败。
- 降级标签显示含摘要后缀的完整 namespace。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 绑定缺失降级为 slug 标签、`source_url` 为空 | 尚无 | Missing |
| 绑定表读取失败时全部派生 namespace 降级、记录告警、响应不失败 | 尚无 | Missing |

### 决策依据

D2 规则 4；不变量 8。

## Adding a Same-Name Source Changes Labels but Never Plugin Identity

### 风险

标签依赖整个比较集合，新增同名来源会让已有标签变长。如果标签参与身份匹配或被持久化，新增来源会破坏已有
匹配；如果前端在市场源变更（含多步操作后一步失败回滚）后不失效查询，用户会一直看到旧标签，误以为消歧没有
生效。删除市场源后绑定仍在，标签不应变回去。

### 前置状态

一个第三方源已绑定并安装了插件；随后新增另一个仓库名相同的源，或删除一个市场源（绑定仍在）。

### 触发

市场源添加、编辑或删除操作结束后，再次查询 Skill 列表、已安装插件列表、市场列表与市场源列表。

### 必须成立

- 新增同名来源后，已有来源的标签从 `marketplace` 变为 `acme/marketplace` 一类的消歧形式；namespace、
  Canonical Plugin ID、安装目录与 `skills` 行不变。
- 删除市场源不让标签变回去，因为绑定还在。
- 市场源操作结束后无论成功还是失败，前端都让 Skill 列表、已安装插件列表、市场列表和市场源列表的查询失效；
  不重启即可看到消歧后的标签。
- 任何匹配、存储和日志继续用 namespace 或 Canonical Plugin ID，不用标签。

### 禁止结果

- 标签参与身份匹配，或被持久化后与当前比较集合不一致。
- 新增绑定改写已有 namespace；删除市场源后标签回退。
- 多步操作后一步失败回滚配置后，前端仍显示未刷新的旧标签。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 新增同名来源后标签变化而 namespace 与 Canonical Plugin ID 不变 | 尚无 | Missing |
| 市场源变更（含失败）后查询失效、不重启看到新标签 | 尚无 | Missing |

### 决策依据

D3 时机与身份边界、D5 规则 4；不变量 3、4。来源语义继承来源派生 Namespace 决策 D3。