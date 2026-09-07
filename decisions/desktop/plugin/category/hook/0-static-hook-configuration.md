---
status: implemented
---

# Hook 插件：以静态配置包声明可解析的命令改写协议

Ora 的 Hook 插件是一个不启动插件进程、却携带一个包内原生可执行文件的静态配置包。插件用
`orax.toml` 声明 `kind = "hook"`，并在唯一的 `assets/config.json` 中声明一个版本化的 Hook Protocol
descriptor、一个包内 `executable` 路径、一个 bare command alias 和一个嵌入式 tool version，以及可选
用户 Settings。宿主在发现阶段编译并整包验证该声明，确认 `executable` 是 package `assets/` 内的真实
普通文件、且 targeted 包自述的 `[artifact]` target 与所选 release target 一致；只有通过整包验证的
已安装 Hook 插件才进入 installed snapshot，且生命周期恒为 `stopped`。安装另一个同名 command alias 的
Hook 不禁用任何包，而是报告 `installed_with_command_conflict`，把 PATH 歧义交给未来 consumer。

该决策已经由 `ora-plugin-manifest`、`ora-plugin-config`、`ora-plugin-manager`、`ora-plugin-lifecycle`
与 `crates/backend` 落地。本文固定 Hook 插件从静态包声明、安装期验证到 command alias 冲突观测的系统
承诺。Hook 如何被一个 Agent Plugin 解析为运行时 `rtk rewrite` 调用、如何协商协议能力，以及如何在
运行中会话热更新，属于 Agent Session 领域，不在本决策中重复。

## 术语与边界

- **Hook 插件**是 `orax.toml` 中 `kind = "hook"` 的 `.orax` 插件包；身份是完整 canonical Plugin ID
  `<namespace>/<identifier>`。
- **Hook Configuration** 是 `assets/config.json` 编译出的静态声明，由可选 Settings 与唯一 Hook Protocol
  descriptor 组成。
- **Hook Protocol** 是版本化、强类型的契约，标识一个 Agent Plugin 如何集成一个 Hook Plugin。Ora 当前
  只接受 `rtk-rewrite-v1`；它调用 `rtk rewrite`，并保留 exit status 与 stdout 表示的命令决策。新协议
  作为显式枚举变体加入，而不是接受任意字符串。
- **Installed Hook Descriptor** 只证明包与声明在安装时有效；它不证明用户已填 Settings、可执行文件
  当前可读、或任何 Agent Plugin 已消费该 Hook。
- **command alias** 是 descriptor 中归一化的 bare command 名（如 `rtk`），用于让未来 consumer 在 PATH
  上定位 Hook；它不参与插件身份，也不被宿主解析为可执行路径。
- Ora 负责声明、配置、包内可执行文件 containment 与 host target 兼容性判断，不代理 Hook 协议流量，
  也不在安装阶段执行 `executable` 做健康检查。

## 决策

### D1：Hook kind 是无插件进程、却携带原生可执行文件的单一静态贡献

release 与 installed 两种 `orax.toml` schema 都接受 `kind = "hook"`。Hook kind 不允许 `[workbench]` 或
`[webview]` 等进程/页面专属 section，也不得携带 `main.js`。它不创建 Deno runtime、不参与 plugin
handshake、不能注册 host methods，也不获得插件进程权限。

与无进程的 MCP 和 Skill 不同，Hook 包本身就是一个原生二进制：descriptor 中的 `executable` 指向
package `assets/` 内的一个真实可执行文件。因此 Hook kind 是少数 `may_ship_targeted_artifact` 的 kind
之一，它用 `[[targets]]`（release）或 `[artifact]`（installed）自述编译目标，以便宿主在下载或导入前
独立证明主机兼容。一个 Hook 包必须携带且只通过 `assets/config.json` 声明 Hook contribution；manifest
不再重复枚举 protocol、command 或 tool version。`assets/config.json` 缺失、不可读、无法编译或不是 Hook
shape 时，整包不进入 installed snapshot。这样 Plugin ID/version 始终只对应一个确定的 Hook contribution，
不会出现“插件已安装但 executable 被忽略”的部分状态。

### D2：`assets/config.json` 使用封闭且互斥的版本化 shape

Hook Configuration schema version 1 的顶层结构为：

```json
{
  "schemaVersion": 1,
  "settings": {},
  "hook": {
    "protocol": "rtk-rewrite-v1",
    "executable": "assets/rtk.exe",
    "command": "rtk",
    "toolVersion": "0.45.0"
  }
}
```

`settings` 可省略，`hook` 必填。配置编译器以顶层 `hook` 识别 Hook shape；没有 `transport` 和 `hook`
的文件仍是普通 Settings-only shape，同时出现 `transport` 与 `hook` 则失败。Hook shape 只能由
`kind = "hook"` 的包携带，其他 kind 声明 `hook` 时也整包失败。

顶层、Settings、descriptor 及引用对象均拒绝未知字段、未知 schema version 和未知 protocol。封闭
schema 的代价是新增协议能力需要显式升级编译器或 schema，而不是让旧宿主静默忽略作者以为已生效的
连接或权限字段。

### D3：descriptor 的四个字段各自封闭校验，且 tool version 独立于插件版本

Hook Protocol descriptor 按声明顺序编译以下字段，任一非法即整包失败：

| 字段 | 约束 | 失败 field path |
| --- | --- | --- |
| `protocol` | 必须解析为已知枚举变体；当前仅 `rtk-rewrite-v1` | `unsupported Hook protocol` |
| `executable` | 必须是 package `assets/` 内的安全相对路径，canonicalize 后为普通非链接文件；Windows 上必须以 `.exe` 结尾 | `hook.executable` |
| `command` | bare command alias；禁止空、路径分隔符 `/` `\`、控制字符、空白与非 ASCII | `hook.command` |
| `toolVersion` | 必须是合法 SemVer | `hook.toolVersion` |

`command` 必须是 bare name：一个路径分隔符会让 Hook 伪装成任意文件系统路径，破坏确定性的 PATH 解析。
命令名由构建脚本产生、由 consumer 原样消费，所以控制字符、空白或非 ASCII 字节都是打包错误。命令别名
不参与插件身份，也不同名去重：两个不同插件可以贡献同名 command alias，冲突由安装期观测而非强制
唯一性处理（见 D7）。

`toolVersion` 是内嵌工具版本，独立于 Hook Plugin 自身的版本。descriptor 单独声明它，是因为一个
`rtk-rewrite-v1` Hook 包的协议兼容性取决于内嵌的 `rtk` 工具版本，而不是 `.orax` 包的 SemVer；插件
版本升级可以只换底层二进制而不改协议，反之亦然。把两者建模成不同字段，避免把“包版本”误当成“协议
兼容性”的唯一输入。

### D4：executable containment 在安装期一次性证明，不提前执行

Hook 包发现采用 fail-closed 语义：manifest kind、配置 shape、descriptor 字段、executable containment 与
`[artifact]` target 一致性中任一项非法，整个最高 installed version 都产生带稳定 field path 的
discovery issue，不回退到同一 Plugin ID 的较低版本，也不产生部分 descriptor。

`validate_executable_containment` 确认编译出的 `executable` 解析为 package `assets/` 树内的真实普通
非 symlink 文件，拒绝 symlink 或 reparse-point escape。Windows 上 milestone 要求 `.exe` 后缀，但 PE
header 不在安装期解析。验证不会执行 `executable`、发起任何子进程、读取用户 Setting 实值或判断 Agent
兼容性。这样安装过程保持确定且无外部副作用；Agent capability 不匹配、可执行文件运行时缺失依赖、
TOCTOU 替换和 Agent 消费失败由 Session setup、refresh 或 Agent 运行边界报告，不能反过来改变 installed
package 的静态有效性。安装期 canonical containment 不能消除验证后文件被外部替换的 TOCTOU 风险；
session 是否以及如何重复验证属于 Agent Session 领域。

### D5：target 选择精确匹配，不跨架构回退

Hook 的 release 形态与 Agent 对称：release manifest 用 `[[targets]]` 列出精确 target 资产，installed
manifest 用 `[artifact]` 自述编译目标。一次安装或本地 `.orax` 导入按以下顺序证明主机兼容：

1. marketplace install：`select_release` 用 `HostTarget` 在 `[[targets]]` 中找精确匹配；host 不在
   `CANONICAL_RUST_TARGET_TRIPLES` 内时返回 `UnsupportedHost`，host 三元组无匹配资产时返回
   `NoArtifactForTarget`。
2. 解压后：installer 重新读取包内 `orax.toml`，确认 targeted 包携带 `[artifact]`，且其 target 与所选
   release target 逐字节相等；不等则 `TargetMismatch`，缺失自述则 `MissingArtifactTarget`。
3. 本地 `.orax` import：对任何自述 `[artifact]` 的包按当前 `HostTarget` 校验；installed Hook 缺
   `[artifact]` 直接失败。`HostTarget::Unsupported` 拒绝 Hook import，但放行 universal 包。

target 匹配是精确的：不跨架构、操作系统、libc 或 ABI 回退。`x86_64-unknown-linux-gnu` 不能替代
`x86_64-unknown-linux-musl`，因为两者二进制不可互换；一次“相近即可”的回退会把打包错误推迟成运行时
崩溃。`HookTarget` 解析器只接受 `CANONICAL_RUST_TARGET_TRIPLES` 内的已知三元组，未知字符串如
`not-a-real-triple` 不解析，以免把打包笔误当成一个可安装的独立架构。这样 Hook 作为原生二进制，在
下载或导入路径上都能独立证明主机兼容，不依赖 marketplace 元数据。

### D6：Settings 复用普通插件配置声明，且 schema version 1 不接受保留类型

Hook 的 `settings` 成员复用普通插件配置声明的字段、校验、编辑器与 host-owned `store.json`。显式空的
`settings` 对象非法；省略 `settings` 表示该 Hook 无需用户配置。schema version 1 不接受保留的
`secret`、`file` 和 `directory` Setting type，敏感值当前仍以受宿主保护的普通标量 Setting 保存，不能
因此被复制进静态插件包或其他声明式持久化状态。

配置记录的 revision 是一次运行时解析的精确输入。带 Settings 的 Hook 只有在配置状态为 `Complete` 时
其配置才参与编辑器；配置修改、清除、损坏或变为不完整时，后续 setup 必须观察到新的 revision。无
Settings 的 Hook 使用稳定的配置 revision `0`，安装完成即具备配置入口资格（RTK v0.1.0 声明 none，
因此当前不向 UI 暴露 Configure 动作）。宿主判断 Hook 的配置完整性时只读声明与 revision，不读取或
暴露 Setting 实值。

### D7：command alias 冲突以观测而非强制唯一性处理

一次成功安装总会保留包：安装是 append-only 的，Ora 没有 enablement 状态。当一个新安装的 Hook 与另一
个已安装 Hook 共享同一 bare command alias 时，`detect_hook_command_conflict` 在刷新 installed snapshot
后返回冲突方的 canonical Plugin ID，安装结果带上
`InstallOutcome::InstalledWithCommandConflict { conflict_plugin_id }`。两个包都保持 installed 且
available；唯一性被推迟给未来 consumer。

之所以把冲突建模成一个显式 outcome 而不是禁用或拒绝安装，是因为：宿主没有 enablement 状态可供
“禁用”，强行卸载任一方都会丢失用户意图；PATH 解析的歧义只在未来某个 consumer 按 command alias
调用时才真正发生，提前消除它只会隐藏信息。冲突 outcome 让调用方可以把冲突身份暴露给用户或未来
consumer，由它们拒绝歧义的 PATH 解析，而不是静默选择错误的 Hook。重新安装同一插件不会与自身冲突：
检测显式排除新安装的 Plugin ID。

### D8：Hook 是全局可用、运行时恒为 stopped 的 processless 插件

Installed Hook 不创建 runtime、不注册方法、也不获得 `--allow-*` 权限。它在 installed snapshot 中始终
全局可见，但 `PluginRuntimeStatus` 恒为 `Stopped`：没有 `activate`/`stop` 会让一个 Hook 进入
`Starting`/`Running`/`Failed`。“无进程”不是“无生命周期”：Hook 仍是可列出、可更新、可卸载的 Installed
Plugin，只是它的进程边界由未来 consumer（Agent Plugin）在调用 `executable` 时建立，而不是由 Ora 的
plugin runtime 建立。

frontend 只接收一个扁平的 `InstalledPluginContribution::Hook`，携带 `protocol`、`command`、`target`
（absent 表示 universal release）和 `toolVersion`，用于审计展示。它永远不学习 `executable` 路径、
origin allow list 或下载规则，这正是把这些宿主策略保持不可协商的边界。卸载按 Plugin ID 完整移除
package 与持久化 lifecycle 状态，与其他 kind 对称。

## 为什么不是这些替代方案

| 方案 | 否决理由 |
| --- | --- |
| 让 Hook 插件启动 `main.js` 并动态注册协议 | 为静态二进制声明引入 handshake、Deno 权限和进程失败面，也使协议能力无法在安装期确定验证 |
| 在 `orax.toml` 中重复声明 protocol 或 command | 产生 manifest 与 `assets/config.json` 两份事实来源，需要定义漂移时的优先级 |
| 允许一个包同时贡献 Hook 与 MCP/Skill | Plugin ID/version 不再对应单一生命周期与权限模型，宿主可能只理解或只执行部分贡献 |
| 允许 `command` 携带路径分隔符或 PATH lookup | 让 Hook 伪装成任意文件系统路径，破坏确定性 PATH 解析；package containment 也无法证明一个 PATH 命令的身份与版本 |
| 安装时执行 `executable` 做健康检查 | 给无副作用的包发现引入代码执行与环境依赖；短暂不可用还会错误改变 installed snapshot |
| 把 `toolVersion` 合并进插件版本 | 一个 `rtk-rewrite-v1` 包换底层二进制但不改协议时，协议兼容性无法独立判定；反之协议升级与换包也会被一个 SemVer 混淆 |
| target 选择跨架构回退 | `gnu`/`musl`、`msvc`/`gnu` 二进制不可互换，回退把打包错误推迟成运行时崩溃 |
| command alias 冲突时禁用或拒绝安装 | 宿主无 enablement 状态可供禁用；强制卸载任一方丢失用户意图；PATH 歧义只在 consumer 调用时才发生，提前消除只会隐藏信息 |
| 让 frontend 学习 `executable` 路径 | 把 origin allow list 与下载规则暴露给页面侧，使这些宿主策略变得可协商 |
| Settings 不完整时仍赋予运行资格 | consumer 只能制造缺值或占位调用，且无法区分“尚未配置”与“运行失败” |

## 不变量

1. 一个有效 Hook 插件必须声明 `kind = "hook"`、不携带 `main.js`，并拥有唯一有效的 `assets/config.json`
   Hook shape。
2. 一个 Hook Configuration 必须恰好包含一个 Hook Protocol descriptor；MCP、Hook 和 Settings-only shape
   不得在同一配置文件中混合。
3. `hook.protocol` 必须解析为已知枚举变体（当前仅 `rtk-rewrite-v1`），未知字符串整包失败。
4. `hook.executable` 必须是当前 installed package `assets/` 内的普通非链接文件；Windows 上必须以
   `.exe` 结尾；PATH lookup、路径穿越和链接 escape 不构成有效 executable。
5. `hook.command` 必须是 bare ASCII 名，不含路径分隔符、控制字符或空白；它不参与插件身份。
6. `hook.toolVersion` 必须是合法 SemVer，且独立于 Hook Plugin 版本表达协议兼容性。
7. targeted Hook release 必须经 `[[targets]]` 精确匹配 host target，installed 包必须自述 `[artifact]` 且
   与所选 release target 逐字节相等；不跨架构、OS、libc 或 ABI 回退。
8. Installed Hook Descriptor 只证明静态包有效，不能被解释为配置完整、可执行文件当前可读或 Agent 已
   消费。
9. Hook 插件不创建插件 runtime、不注册 host method、不获得 `--allow-*` 权限，运行时恒为 `Stopped`。
10. 同名 command alias 的多个 Hook 共存；冲突以 `InstalledWithCommandConflict` 观测而非强制唯一性，
    任何 consumer 都必须按 Plugin ID 重新解析，不能把旧声明与新 Setting 拼接为有效输入。

## 当前未解决的问题

- Hook Configuration schema version 1 不支持 `secret`、`file`、`directory` Setting type；引入专用秘密
  类型需要同时定义存储、编辑、redaction、revision 和升级语义。
- 安装期 canonical containment 不能消除验证后 `executable` 被外部替换的 TOCTOU 风险；session 是否
  以及如何重复验证由 Agent Session 决策约束。
- `CANONICAL_RUST_TARGET_TRIPLES` 当前是一个封闭 allowlist；新增主机目标需要显式扩展解析器，作者
  不能通过未知字符串提前启用。
- Agent Plugin 如何消费 Hook、`rtk rewrite` 的 exit code 协议（0 allow / 1 passthrough / 2 deny / 3 ask）
  如何在 Agent 边界表达，以及运行中会话如何热更新，属于 Agent Session 领域，由
  `desktop/core/agent/session` 下的 Hook 决策定义。
- command alias 冲突的最终 consumer 解析策略与 PATH 唯一性尚未定义；当前仅观测冲突，不在宿主侧消除。

## 核心测试依据

见 `specs/test-cases/desktop/plugin/category/hook/static-package-and-configuration.md`。
