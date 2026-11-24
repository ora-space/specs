---
status: superseded
---

# Skill Effect：将 Plugin Skill 发布为不可变 Source 并经 ownership 收敛

> 已被取代：本决策由 [Skill 按存储路径以会话清单交给 Agent，取代 Effect 物化](../../agent/session/skill/0-skill-catalog-from-storage-paths.md) 取代，原因：Plugin Skill 不再发布为 Effect Source，Catalog 中的 Available Skill 直接按插件包内的存储位置进入会话清单，不再物化到 Workspace。当前系统承诺以取代者为准；取代者的实现合并之前，desktop 里仍在运行的 Effect 以本文为历史依据。Effect 的核心用例暂时保留，在取代者进入 `implemented` 时随 Effect 代码及其中的引用一并删除。

Ora 把 Skill 插件动态提供给 Catalog 的每个 Plugin Skill 发布为 immutable Effect Source revision，
再由通用 Effect Target/Resource 协调、ownership ledger 和 filesystem adapter 将完整 Skill package
收敛到 Consumer 声明的 Skill discovery Resource。发布只改变声明式 Desired State，不直接写入
Workspace；显式卸载插件时退休 Source，并通过后续 generation 清理由 Ora 管理的结果。

该决策已经由 SQLite Effect repository、`crates/backend` 与 `ora-effect-skill` 落地。它承接
`desktop/plugin/category/skill/0-static-skill-package.md` 对 Plugin Skill package、Catalog 身份和
受控生命周期的定义，并具体化 `desktop/core/effect/0-effect-system-foundation.md` 的通用 Source、
Revision、generation、Resource 与 ownership 规则。插件如何分发、安装和动态提供 Catalog Skill
不属于本决策。

## 术语与边界

- **Plugin Skill Source** 是由一个 active Plugin Skill Catalog projection 派生的稳定 Effect 来源。
- **Plugin Skill revision** 是绑定插件版本和已验证 package 内容的 immutable Effect Revision。
- **Publication** 是把已验证的 Catalog/package 事实原子写入 Effect repository 并更新 Scope Desired
  State；它不执行 filesystem materialization。
- **Materialization** 是 Effect worker 在取得 Resource authority 后，将 revision 的完整 package 内容
  收敛到 Consumer Resource。

## 决策

### D1：Catalog projection 与 Effect publication 是相邻但独立的领域转换

Plugin Skill 进入 Catalog 后，宿主为其发布 Effect Source；Catalog transaction 不直接复制 package
到 Ora 的本地可写 Skill 根或 Workspace。这样插件领域只负责证明“哪个插件当前提供哪个只读 Skill”，
Effect 领域负责“哪些 Scope 期望它，以及哪些 Consumer Resource 已安全收敛”。

Catalog metadata、package availability、Effect publication 和 Resource materialization 是不同事实。
其中任一项成立都不能替代下一项的持久化证据，也不能授权 filesystem mutation。

### D2：Source 身份与 revision 内容完整绑定 Plugin Skill

每个 Plugin Skill publication 使用以下稳定映射：

| Effect 字段 | Plugin Skill 来源 |
| --- | --- |
| source kind | `plugin` |
| source namespace | 完整 canonical Plugin ID |
| source identifier | canonical Skill name |
| revision key | 插件 SemVer |
| materialization input | 已验证的 plugin package root |

发布前计算并验证 `SKILL.md` digest 和完整 package fingerprint；revision 同时绑定这些内容证据、插件
版本与 package root。repository 只原子持久化已经验证的 publication，不在事务中通过 Resource Adapter
重新读取 source 目录。

同一插件的新版本沿用稳定 Source identity，并发布新的 immutable revision。新版本移除的 Skill 或
显式卸载会退休对应 active Source；旧 revision 行保持不可变，以供已有 projection、Managed State 和
恢复链路继续引用。文件系统路径、裸 Skill name 或 fingerprint 都不能单独替代 Source identity。

### D3：Source lifecycle 通过 Scope generation 表达

新 Source 发布时，Effect repository 为现有 Workspace Scope 引入该 Source 的 Desired Effect，并各自
只推进一次 generation；新 Workspace 创建时 seed 当前 active Source。重复发布相同 Source revision
是 no-op，不产生无意义 generation。

Source retirement 删除其当前 Desired intent，并令引用它的 Scope 进入下一 generation。retirement
不直接删除 Workspace 文件，也不抹除 immutable revision；清理由正常 reconcile 根据 Managed ledger
完成。外部工具绕过 Ora 删除或损坏插件安装目录不等价于受控 retirement，不能据此猜测 Desired 或
ownership 应被撤销。

### D4：完整 package 只能通过 Effect ownership 协议物化

`ora-effect-skill` 从 revision 绑定的 package root 读取完整 Skill package，包括 `SKILL.md`、scripts、
references、嵌套 assets 和其他普通文件，并把它作为一个贡献参与 Target projection 与共享 Resource
merge。写入前仍必须取得 fenced Resource claim，并遵守 journal、coordination、Managed ledger 和
adapter ownership evidence；Source publication 本身不授予 mutation authority。

已收敛的相同 generation 重放必须是 no-op。未经管理但占用相同 native identity 的内容必须保留并以
blocking Condition 报告；不能因为 Source、目录名或 fingerprint 相同就推断 Ora ownership。同名
Plugin Skill 若投影到同一 Resource native directory，其冲突由 Effect planner 在 Resource merge 时
阻塞，不通过改写 Catalog 身份、自动别名或静默覆盖解决。

## 为什么不是这些替代方案

| 方案 | 否决理由 |
| --- | --- |
| Catalog 同步时直接复制到每个 Workspace | 绕过 generation、Target/Resource 协调、journal 和 ownership，部分失败后也缺少可恢复证据 |
| 只发布 `SKILL.md` | Skill package 的 scripts、references、assets 等内容会丢失，revision fingerprint 也无法代表真实消费输入 |
| 用裸 Skill name 作为 Source identity | 不同 Plugin ID 的同名贡献会碰撞，更新与卸载无法按 owner 精确推进 Desired State |
| Source retirement 直接删除目标目录 | retirement 没有 Resource mutation authority，也无法区分 Ora 管理内容、共享贡献与用户内容 |
| 根据现存同名目录自动接管 | observed state 不能证明 ownership，会把用户或其他工具拥有的内容纳入 Ora 的删除权限 |

## 不变量

1. Plugin Skill Catalog projection 不等于 Effect publication，Effect publication 不等于 Resource 已收敛。
2. Plugin Skill revision 必须绑定发布时验证过的 package root、manifest digest、完整 fingerprint 和插件
   版本，发布后不可改写。
3. Source publication 与 retirement 只能通过 Desired State 和 Scope generation 驱动收敛，不能直接
   修改 Workspace filesystem。
4. 新 Workspace 必须 seed 当前 active Plugin Skill Sources；现有 Workspace 必须各自以一次 generation
   接收有语义变化的 publication 或 retirement。
5. 完整 Skill package 的任何 Resource mutation 都必须取得 Effect ownership authority，并遵守通用
   journal、coordination 与恢复协议。
6. Source retirement 必须保留仍被历史 projection、Managed State 或未终结 Operation 引用的 immutable
   revision。
7. 同名目录、Source、digest 或 fingerprint 都不能授权接管未经管理的外部内容。

## 当前未解决的问题

- installed tree 的持续监听和外部删除、损坏后的孤儿 Catalog projection 清扫属于插件生命周期边界；
  在插件领域定义受控 retirement 前，Effect 不自行推断 Source 已卸载。
- 同名 Plugin Skill 的用户侧别名或自动重命名协议尚未定义；当前冲突保持为 blocking Condition。

