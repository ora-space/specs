# Skill 配置与启动环境

以下源码证据统一固定到 Desktop 合并提交 `8218ef060614f8fdada03bdffbea4a654ed7c223`。`Covered` 只表示该行完整义务有直接测试；组合路径缺口保留 `Partial`。

## Secret Settings Must Preserve Values Without Returning Them

### 风险

编辑器响应泄露凭据，或保存公开字段时清空未编辑的 secret。

### 前置状态

v1 Skill 配置包含字符串、数字、布尔 secret、公开字段、默认值和存储覆盖；另有 MCP 与 v2 Agent 声明。

### 触发

读取、保存、重置配置，或编译声明。

### 必须成立

secret 默认值、存储值、有效值不回传，只返回来源与 redacted；未编辑保存保留原值，重置删除覆盖；公开字段可见；MCP 全量脱敏。v1 省略 secret 等于 false；v2 拒绝 secret（包括 false），保持既有编译输出和指纹契约。secret 参与 v1 指纹，revision CAS 与声明变化冲突不变。

### 禁止结果

任何响应泄露 secret；将界面脱敏解释为 store.json 加密；放宽 v2 声明契约。

### 验证义务与证据

| 验证义务 | 代表性证据 | 状态 |
| --- | --- | --- |
| 三种类型的 secret 在读取、保存、重置响应中均脱敏，公开字段可见，未编辑保留值 | [配置 API 与三类型脱敏测试](https://github.com/ora-space/desktop/blob/8218ef060614f8fdada03bdffbea4a654ed7c223/crates/backend/src/plugin_configuration.rs)；三类型完整读取/保存/重置组合尚无逐类型直接证据 | `Partial` |
| v1 指纹、CAS、默认值、必填、重置及 MCP 全量脱敏保持一致 | [默认值、revision CAS、声明升级与持久化测试](https://github.com/ora-space/desktop/blob/8218ef060614f8fdada03bdffbea4a654ed7c223/crates/plugin-config/src/service.rs)；[配置 API 与三类型脱敏测试](https://github.com/ora-space/desktop/blob/8218ef060614f8fdada03bdffbea4a654ed7c223/crates/backend/src/plugin_configuration.rs)；[MCP 绑定字段强制脱敏投影](https://github.com/ora-space/desktop/blob/8218ef060614f8fdada03bdffbea4a654ed7c223/crates/backend/src/plugin_configuration.rs) | `Partial` |
| v2 拒绝 secret true/false，现有编译输出与指纹向量不变 | [keeps_skill_secret_metadata_out_of_agent_settings 与指纹/契约向量](https://github.com/ora-space/desktop/blob/8218ef060614f8fdada03bdffbea4a654ed7c223/crates/plugin-config/src/agent/tests.rs) | `Covered` |

### 决策依据

[配置 ADR D1](../../../../../decisions/desktop/plugin/category/skill/20261016-secret-settings-and-startup-environment.md#d1配置沿用通用存储增加敏感字段声明)。

## Readable Environment Names Must Reject Every Colliding Plugin

### 风险

有损规范化、大小写或拼接歧义导致覆盖其他凭据；继承或调用方环境伪造宿主配置。

### 前置状态

完整有效配置包含大小写、标点、namespace、非 ASCII 字符、连续下划线；环境和调用方 env 含混合大小写的 ORA_SKILL_ 键。

### 触发

建立快照并通过两条 Agent 启动路径创建进程。

### 必须成立

仅 ASCII 字母数字保留，ASCII 小写转大写，其他每个 Unicode 码点（标量值）换一个 `_`，不做 NFC/NFD 规范化，不合并或裁剪；完整 Plugin ID 含 namespace。按已安装有效 Skill 插件的有效声明中的全部字段检测跨插件、同插件字段及拼接边界撞名，与字段填值无关；配置值无效被跳过的插件仍参与，插件或声明无效而没有有效字段集合时不参与。涉及插件整包不注入，其余插件正常注入；检测不依赖枚举顺序。Agent 插件和宿主 `childprocess/spawn`（含无 cwd）对继承环境及调用方 env 的所有保留前缀变量先移除再注入，空快照也清理。字符串原样、数字 JSON、布尔 true/false。

### 禁止结果

静默覆盖、任选胜者、仅删除冲突字段、诊断含配置值、无前缀别名、复活被跳过插件的变量。

### 验证义务与证据

| 验证义务 | 代表性证据 | 状态 |
| --- | --- | --- |
| `a/b` + `api-token` → `ORA_SKILL_A_B_API_TOKEN`，`a//b` → `A__B`，`é`（U+00E9）→ `_`，`e` + U+0301 → `E_`；三种值类型序列化 | [规范化、标量、整包跳过与别名拒绝测试](https://github.com/ora-space/desktop/blob/8218ef060614f8fdada03bdffbea4a654ed7c223/crates/plugin-config/src/skill.rs) ；尚缺上述全部 Unicode 示例的逐例直接断言 | `Partial` |
| 大小写、标点、同插件字段及 `x/a_b` + `c` / `x/a` + `b_c` 撞名均整包拒绝；可选字段未填及配置值无效插件仍参与，插件或声明无效无字段集合时不参与；逆序枚举结果一致，其他插件不受影响 | [声明撞名矩阵、脱敏诊断与冻结快照测试](https://github.com/ora-space/desktop/blob/8218ef060614f8fdada03bdffbea4a654ed7c223/crates/backend/src/plugin/skill_environment.rs)；[规范化、标量、整包跳过与别名拒绝测试](https://github.com/ora-space/desktop/blob/8218ef060614f8fdada03bdffbea4a654ed7c223/crates/plugin-config/src/skill.rs) ；拼接边界 `x/a_b` + `c` / `x/a` + `b_c` 尚无独立输入测试 | `Partial` |
| 两条启动路径在继承及调用方环境中先清理保留前缀，再注入；空快照、冲突、缺配置均不能复活伪造值 | [真实 Deno Agent、后代与 host spawn 测试](https://github.com/ora-space/desktop/blob/8218ef060614f8fdada03bdffbea4a654ed7c223/crates/plugin-lifecycle/src/environment_tests.rs)；[空快照与大小写保留前缀过滤测试](https://github.com/ora-space/desktop/blob/8218ef060614f8fdada03bdffbea4a654ed7c223/crates/plugin-lifecycle/src/childprocess_tests.rs)；[进程继承环境前缀过滤测试](https://github.com/ora-space/desktop/blob/8218ef060614f8fdada03bdffbea4a654ed7c223/crates/plugin-runtime/src/tests.rs) ；冲突/缺配置后经真实进程尝试复活变量的组合尚未直接覆盖 | `Partial` |
| 无前缀通用别名不可声明，冲突诊断不含配置值 | [规范化、标量、整包跳过与别名拒绝测试](https://github.com/ora-space/desktop/blob/8218ef060614f8fdada03bdffbea4a654ed7c223/crates/plugin-config/src/skill.rs)；[声明撞名矩阵、脱敏诊断与冻结快照测试](https://github.com/ora-space/desktop/blob/8218ef060614f8fdada03bdffbea4a654ed7c223/crates/backend/src/plugin/skill_environment.rs) | `Covered` |

### 决策依据

[配置 ADR D3](../../../../../decisions/desktop/plugin/category/skill/20261016-secret-settings-and-startup-environment.md#d3变量名采用可读规范化检测撞名并保留宿主前缀)。

## Agent Process Trees Must Share A Restart Bound Configuration Snapshot

### 风险

两条启动路径使用不同配置，配置失效阻断 Skill 交付，或重置被误认为立即撤销凭据。

### 前置状态

多个工作区和 Agent；有效、缺失必填、损坏、非法、含 NUL 的 Skill 配置并存。

### 触发

初始化 Backend，经 Agent 插件及 `ora/childprocess/spawn`（含不指定 cwd）启动 Agent/子进程；随后保存、重置、安装、更新、卸载，重新启动 Agent，再重启 Desktop。

### 必须成立

两条路径共享同一冻结快照，所有 Agent 进程树可读取全部完整有效配置，非 Agent 插件不获得注入；可选缺失不产生变量，无效配置整包跳过且不阻断其他插件、会话或 Skill 交付。配置变更仅在 Desktop 重启后影响注入，设置页提示重启，不自动中断会话；单独重启 Agent 仍使用旧快照。

### 禁止结果

配置值进入宿主交付路径的工作区、Effect、ACP 或日志；声称能防止脚本自行输出泄漏；声称卸载、重置或重启可以吊销外部凭据。

### 验证义务与证据

| 验证义务 | 代表性证据 | 状态 |
| --- | --- | --- |
| 两条路径、无 cwd、跨工作区和后代进程共享快照，非 Agent 插件不注入 | [真实 Deno Agent、后代与 host spawn 测试](https://github.com/ora-space/desktop/blob/8218ef060614f8fdada03bdffbea4a654ed7c223/crates/plugin-lifecycle/src/environment_tests.rs)；[声明撞名矩阵、脱敏诊断与冻结快照测试](https://github.com/ora-space/desktop/blob/8218ef060614f8fdada03bdffbea4a654ed7c223/crates/backend/src/plugin/skill_environment.rs)；真实进程测试使用固定 provider，跨工作区完整串联尚无合并测试 | `Partial` |
| 缺必填、损坏、非法、NUL 整包跳过；可选缺失不注入，Skill 仍交付，诊断无值 | [声明撞名矩阵、脱敏诊断与冻结快照测试](https://github.com/ora-space/desktop/blob/8218ef060614f8fdada03bdffbea4a654ed7c223/crates/backend/src/plugin/skill_environment.rs)；[规范化、标量、整包跳过与别名拒绝测试](https://github.com/ora-space/desktop/blob/8218ef060614f8fdada03bdffbea4a654ed7c223/crates/plugin-config/src/skill.rs)；配置无效时仍完成 Skill 物化的组合路径未直接覆盖 | `Partial` |
| 保存、重置、安装、更新、卸载后重启 Agent 不刷新，重启 Desktop 才刷新；不自动停止会话，界面提示重启 | [声明撞名矩阵、脱敏诊断与冻结快照测试](https://github.com/ora-space/desktop/blob/8218ef060614f8fdada03bdffbea4a654ed7c223/crates/backend/src/plugin/skill_environment.rs)（保存、重置、卸载及重新 capture）；[配置页保留、清空与重启提示测试](https://github.com/ora-space/desktop/blob/8218ef060614f8fdada03bdffbea4a654ed7c223/packages/app-shell/src/features/settings/plugins-settings.test.tsx)；安装/更新/活动会话不中断的完整组合仍缺直接证据 | `Partial` |
| 宿主交付不写工作区、Effect、ACP、日志；存储明文与脚本输出不在防泄漏承诺内 | [配置 API 与三类型脱敏测试](https://github.com/ora-space/desktop/blob/8218ef060614f8fdada03bdffbea4a654ed7c223/crates/backend/src/plugin_configuration.rs)；[真实 Deno Agent、后代与 host spawn 测试](https://github.com/ora-space/desktop/blob/8218ef060614f8fdada03bdffbea4a654ed7c223/crates/plugin-lifecycle/src/environment_tests.rs)；宿主全部工作区/Effect/ACP/日志交付面尚无统一直接断言，不能据此承诺第三方脚本防泄漏 | `Partial` |

### 决策依据

[配置 ADR D2、D4 与已知边界](../../../../../decisions/desktop/plugin/category/skill/20261016-secret-settings-and-startup-environment.md)。
