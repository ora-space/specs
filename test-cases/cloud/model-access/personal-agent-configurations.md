# 个人 Agent 配置、冻结与授权

决策依据：[个人 Agent 配置](../../../decisions/cloud/model-access/20261017-personal-agent-configurations.md)。
以下新增验证义务均为 `Missing`。既有 OpenCode/个人连接的 Covered/Partial 证据保持原范围，
不能推导 Claude、新 schema 或 adapter 路线已实现；供应商确认不是 Ora 实测证据。

## Plugin configuration must reuse owner-scoped connections without exposing keys

- 风险：插件另建 Key 库，或空间管理员继承他人个人配置权。
- 前置：两用户同空间、管理员、同名不同来源插件，以及可被两插件选择的连接。
- 触发：网页内联创建/选择连接、配置读写、Key 写入和跨用户重试。
- 必须成立：所有连接来自同一用户库；Settings 与秘密接口分开；读取只给允许字段和已配置状态。
- 禁止结果：请求体决定归属、普通 Setting 存 Key、管理员读取个人值、删除插件导致共用连接被删。
- 验证义务与证据：

| 义务 | 所需直接证据 | 状态 |
| --- | --- | --- |
| 用户/来源隔离 | 真实 PG + HTTP 的跨用户、管理员、canonical ID 与幂等矩阵 | Missing |
| 网页秘密通路 | 生产页面和 Gateway/model-gateway 联合验证，含前端缓存/错误/日志不留 Key | Missing |
| 连接单一事实来源 | 两插件共用连接、一次轮换/删除与各引用状态的集成验证 | Missing |

## Configuration writes must reject stale revisions without a declaration change

- 风险：只检查 schema 指纹，同一声明下后写覆盖先写，或清除重建产生旧 revision 再次有效。
- 前置：两个编辑器读到同一用户/插件、发行、声明指纹和 revision；另备无配置的首次创建状态。
- 触发：保存/保存、保存/清除竞争，同一幂等请求重放，清除重填后提交旧页面；OpenCode 同时存在全局默认版本。
- 必须成立：不同修改只有一个提交成功，陈旧请求冲突并保留草稿；清除推进 revision，重填不复用旧 revision；合法重放无额外推进。
- 禁止结果：自动重试覆盖胜者、并发建两份默认、清除重置版本、以插件 revision 绕过全局默认 version。
- 验证义务与证据：

| 义务 | 所需直接证据 | 状态 |
| --- | --- | --- |
| CAS 与幂等 | 生产 HTTP + 真实 PG 的并发首次创建、保存/清除、重复与不同输入重放矩阵 | Missing |
| 草稿与后续运行 | 两编辑器冲突保留草稿，随后运行冻结胜者配置；旧页面在清除重填后仍失败 | Missing |
| OpenCode 双资源版本 | 全局默认与插件选项各按自身版本处理，不以其中一方版本覆盖另一方 | Missing |

## Explicit Claude selection must not change or inherit the global default

- 风险：Claude 误用 OpenCode default，或两个默认记录漂移。
- 前置：已存在 OpenCode 默认值、不同 Claude 配置和工作流 default 槽位。
- 触发：分别编辑两种插件配置并从普通任务与原工作流入口创建运行；包括 Claude 缺配置。
- 必须成立：OpenCode 插件页和全局默认读写同一选择；Claude 独立解析；缺配置明确拒绝。
- 禁止结果：复制 default、切换全局 CLI、缺配置回退其他人/槽位/宿主凭据。
- 验证义务与证据：三入口/默认值读写的生产 HTTP + PG 和浏览器行为测试，当前 `Missing`。

## Claude thread permissions must protect the initiator's model access

- 风险：Claude 接入沿用普通 Thread 成员追加权，导致其他成员消耗发起者的个人额度。
- 前置：带个人绑定的 Claude Thread，发起者、其他成员、非发起者租户管理员，以及无绑定历史 Thread 对照。
- 触发：各角色读取、追加、结束；随后撤销成员/账号或模型绑定，进入 ending/ended，再重放旧界面请求。
- 必须成立：以下当前资格与状态由服务器检查，界面据同一能力显示；无绑定历史 Thread 保持原规则。
- 禁止结果：管理员结束权变成追加权、撤销后仍可追加、关闭后无法读取历史、前端禁用而 HTTP 仍受理。

| 场景 | 读取 | 追加 | 结束 |
| --- | --- | --- | --- |
| 可受理状态、有效发起者 | 允许 | 允许 | 允许 |
| 可受理状态、其他有效成员 | 允许 | 拒绝 | 拒绝 |
| 可受理状态、非发起者当前租户管理员 | 允许 | 拒绝 | 允许 |
| 成员或账号资格已失效 | 拒绝 | 拒绝 | 拒绝 |
| 绑定撤销、仍有效的发起者/租户管理员 | 允许 | 拒绝 | 按现有状态允许收尾 |
| ending/ended、当前有效成员 | 允许历史读取 | 拒绝 | 继承 Thread 状态/幂等规则，不重开运行 |

- 验证义务与证据：生产 Thread HTTP + 真实 PG 的角色/状态/重放矩阵，浏览器 canAppend/canEnd 与直接接口结果一致；当前 `Missing`。旧 OpenCode 权限证据不能替代。

## Run creation must freeze the complete execution configuration atomically

- 风险：配置编辑或 catalog 更新改变排队运行，失败留下半个运行。
- 前置：合法发行声明和个人配置；并发升级包、编辑 Settings/模型、替换 Key。
- 触发：创建运行、延迟派发、准备重试；提交错误版本/协议/目标/引用。
- 必须成立：发起者、包/target/摘要、声明、adapter 版本、选项 revision 与模型绑定一次冻结；失败原子回滚。
- 禁止结果：客户端自报快照被信任、旧运行重读默认值、事务中访问网络、配置失败仍有可派发运行。
- 验证义务与证据：

| 义务 | 所需直接证据 | 状态 |
| --- | --- | --- |
| 并发冻结与回滚 | 真实事务交错测试核对评论/交互/Run/绑定及实际派发内容 | Missing |
| 版本及能力拒绝 | 生产准入接口的协议/target/声明/安装准备矩阵 | Missing |
| 共享展示脱敏 | Thread/运行详情 HTTP 只含允许元数据，不泄露个人选项、上游地址或凭据引用 | Missing |

## Editing and revocation must have distinct effects on existing runs

- 风险：普通配置编辑热切换，或撤销因 DNS 失败/重新启用失效。
- 前置：运行处于 pending/active/idle，并持有可持续流式的授权。
- 触发：编辑选项、改模型、替换 Key、清插件选项、清 Key、停用/删除连接、失去成员资格、结束。
- 必须成立：普通编辑只影响新运行；真正撤销拒绝后续和在途访问；纯降权不依赖出网可用。
- 禁止结果：复活旧绑定、将配置清除误作共用连接删除、回退其他凭据、静默 provider 切换。
- 验证义务与证据：PG 授权状态 + 真实受控 HTTP/SSE、DNS 不可用与 Node 续期的联合矩阵，当前 `Missing`。

## Upgrades must preserve history and require explicit repair of changed declarations

- 风险：旧编辑器覆盖新配置，升级迁移改写历史输入或丢值。
- 前置：既有 OpenCode 用户/历史运行、旧插件无声明、新包同指纹或不同指纹。
- 触发：数据库升级、旧页面保存、卸载重装、来源或 target 改变、仅排版变化的声明升级（字节不同、语义相同）、新旧运行分别启动。
- 必须成立：旧连接/默认/Key 引用不复制；历史不回填；冲突保留值/草稿并拒绝；仅排版变化也按指纹不匹配处理并要求显式修复，不做兼容猜测；新值只用于新 run。
- 禁止结果：插件迁移脚本在 Cloud 执行、未知字段静默删除、旧包被原地修改、旧绑定升级为新语义。
- 验证义务与证据：真实新建/升级数据库、部署混合版本和浏览器修复路径测试，当前 `Missing`。

## Changed declarations must permit explicit repair without overwriting concurrent edits

- 风险：所有指纹不匹配的保存均拒绝，用户无法修复；或修复跳过 CAS、自动丢弃字段并改写在途配置。
- 前置：已保存旧指纹/值/revision，发布的新声明增加必填项或移除字段，旧运行仍持旧快照。
- 触发：旧编辑器保存；重新加载并显式修复；修复过程中另一编辑器保存或管理员再次升级。
- 必须成立：旧保存冲突；重新加载区分新声明与旧值；合法修复按当前发行、新指纹及预期旧 revision 原子提交。
- 禁止结果：只能删除重建、无条件覆盖、隐藏丢字段、默认值自动批准、改写 Key 或已有运行。
- 验证义务与证据：生产 HTTP/PG 验证修复成功、缺必填/类型错误和双重竞争拒绝；浏览器保留草稿且只有显式确认才移除旧有效项；当前 `Missing`。

## One personal configuration must resolve predictably across spaces with different declarations

- 风险：两个空间选定不同声明的发行，单份配置在空间间来回失配，或被静默回退成某个空间的版本。
- 前置：同一用户的空间 A、B 选定同一插件的不同声明版本（指纹不同），配置按 A 的指纹保存且有效。
- 触发：在 A、B 分别打开个人配置页与创建运行；随后在 B 的发行上下文显式修复，再回 A 创建运行。
- 必须成立：配置页按所在空间解析发行；修复前 A 可创建、B 明确拒绝并说明声明指纹失配；在 B 修复后 B 可创建、A 变为明确拒绝并提示修复；拒绝原因可与其他缺口（权限、安装、凭据）区分。
- 禁止结果：按 latest 或任一空间版本静默回退、两个空间同时声称可用、把失配报成权限或安装错误、修复后另一空间无提示复用旧值启动。
- 验证义务与证据：真实 PG + HTTP 的双空间交错矩阵（保存/修复/创建运行的顺序组合），当前 `Missing`。

## Claude model aliases must resolve to the single authorized model

- 风险：辅助或深度调用隐式访问第二模型，兼容标签误判协议。
- 前置：一个 Messages 连接和明确模型；另有 Completions 连接、未授权模型和主/辅助别名配置。
- 触发：启动 Claude、调用主/辅助路径、会话切模型及越权请求。
- 必须成立：实际每个推理请求都使用冻结模型；不兼容配置启动前拒绝；不能满足单模的固定版本不开放。
- 禁止结果：网关任意改写模型、授权第二模型、以同名供应商判断协议或回退直连。
- 验证义务与证据：固定 CLI/adapter + 受控网关对真实请求的模型/协议核验及拒绝矩阵，当前 `Missing`。

## Real Claude runs must complete with personal credentials and released resources

- 风险：只验证表单或供应商声明，实际 CLI、成果与撤销未贯通。
- 前置：无部署共享 Key/宿主登录态的真实栈，固定插件/CLI/adapter，用户经网页设置个人连接。
- 触发：真实两轮任务和工具操作、正常结束；独立验证取消/失败及授权撤销。
- 必须成立：同一 Run 的流式/工具调用、成果/Git 身份、历史和 Workspace 释放可核验，敏感材料不进入成果。
- 禁止结果：以模拟器/HTTP 200/另一 Run 替代、把上游 Messages 确认当端到端通过、把正常退出当撤销证据。
- 验证义务与证据：

| 义务 | 所需直接证据 | 状态 |
| --- | --- | --- |
| 受限协议兼容 | 明确路径/方法/query/头的合成供应商测试，覆盖未知项拒绝、SSE、工具数据和不自动重发 | Missing |
| 真实网页正常链路 | 固定版本、同组资源的两轮/成果/身份/释放证据，实际 Key 经网页受控入口写入 | Missing |
| 隔离与收尾 | 双用户并发、错误/取消/撤销、临时令牌及真实 Key 的安全排除核验 | Missing |
| 旧 OpenCode 回归 | 升级后的两协议默认模型与真实任务回归，不仅引用升级前记录 | Missing |
