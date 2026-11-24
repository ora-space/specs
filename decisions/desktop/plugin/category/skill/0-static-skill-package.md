---
status: implemented
---

# Skill 插件：以静态多 Skill 包动态提供 Catalog Skill

> 历史决策：[Skill 按存储路径以会话清单交给 Agent](../../../core/agent/session/skill/0-skill-catalog-from-storage-paths.md)（`approved`）修改本决策的下游边界：开头“Skill 进入 Catalog 后如何发布、选择和物化为 Effect，属于 Effect 领域决策”与“当前未解决的问题”最后一条不再适用。Catalog 中 Available 的 Plugin Skill 不发布为 Effect Source，直接以 package root 作为存储位置进入会话 Skill 清单；`desktop/core/effect/skill` 已改为 `superseded`。D1–D7 与不变量不变。

Ora 的 Skill 插件是一个不启动进程的静态插件包。插件用 `orax.toml` 声明
`kind = "skill"`，并在固定的 `assets/<skill-directory>/SKILL.md` 布局中携带一个或多个
Skill。宿主在插件发现阶段验证整包，在插件安装、更新和启动同步时把每个有效 Skill 投影到共享
Skill Catalog；显式卸载插件时撤销这些投影。

该决策已经由 `ora-plugin-manifest`、`ora-plugin-manager`、`ora-skill-package`、
`ora-domain`、SQLite Skill repository 与 `crates/backend` 落地。本文固定从静态插件包分发安装到
运行时向共享 Catalog 动态提供只读 Skill 的系统承诺；Skill 进入 Catalog 后如何发布、选择和物化为
Effect，属于 Effect 领域决策。普通本地 Skill 的文件/压缩包导入协议也不纳入 Skill 插件语义。

## 术语与边界

- **Skill 插件**是 `orax.toml` 中 `kind = "skill"` 的 `.orax` 插件包；它的身份是完整 canonical
  Plugin ID `<namespace>/<identifier>`。
- **Contributed Skill** 是该插件 `assets/` 下一个直接子目录拥有的完整 Skill package。
- **Plugin Skill projection** 是 Catalog 中来源为该插件的只读 Skill 行。
- 普通 Skill import 可以递归扫描文件夹或 `.zip`、`.skill`、`.tar.gz`、`.tgz` 中任意深度的
  `SKILL.md`，并按候选独立失败；这不是 Skill 插件的包布局或失败语义。

## 决策

### D1：`orax.toml` 只声明插件种类，不重复枚举 Skill

release 与 installed 两种 `orax.toml` schema 都接受 `kind = "skill"`。installed manifest 可以省略
`resolver`，宿主按当前 resolver 1 解析；身份、版本、描述等通用字段仍按插件 manifest 规则校验。

Skill kind 没有专属 TOML section，也不通过 manifest 数组声明每个 Skill。`[workbench]`、
`[webview]`、`[[targets]]` 和 `[artifact]` 均不属于 Skill kind：Skill 插件是平台无关的通用发布资产，
Skill 清单的数量和身份由安装包内固定目录布局决定。

这样只保留一个包内容事实来源：`orax.toml` 决定插件身份和版本，`SKILL.md` 决定每个 Skill 的公开
名称和描述，不存在 TOML 声明与文件树清单需要双向对账的问题。

### D2：插件包采用固定、浅层且整包校验的 Skill 布局

一个 Skill 插件必须满足：

```text
<plugin-package>/
├── orax.toml
└── assets/
    ├── <skill-a>/
    │   ├── SKILL.md
    │   ├── scripts/       # 可选，作为 package 内容保留
    │   ├── references/    # 可选，作为 package 内容保留
    │   └── assets/        # 可选，作为 package 内容保留
    └── <skill-b>/
        └── SKILL.md
```

规则如下：

1. `assets/` 必须存在、是目录，并至少包含一个直接子目录；`assets/` 下的普通文件不构成 Skill。
2. 每个直接子目录都是一个 Skill 根，必须包含大小写精确为 `SKILL.md` 的可读文件。
3. 不递归发现 `assets/<group>/<skill>/SKILL.md`；插件级目录结构故意保持为一层。
4. 任意一个 Skill 根缺失或包含非法 `SKILL.md`，整个插件都不进入已安装插件 snapshot；不会只接受
   其余有效 Skill。
5. `scripts/`、`references/`、嵌套 `assets/` 和其他普通文件属于 Skill package，供下游整体消费，但
   插件发现阶段不解释其语义。

整包校验使一个 Plugin ID 和版本始终对应确定的贡献集合，避免 UI 显示“插件已安装”但其中一部分
Skill 被静默舍弃。代价是一个 Skill 的打包错误会阻止同包所有 Skill 被发现。

### D3：`SKILL.md` 只管理公开身份字段，正文与扩展字段属于不透明内容

`SKILL.md` 必须从第一行开始使用 `---` 包围的 YAML front matter。当前解析器读取并验证：

- `name`：必填、trim 后非空，遵守共享 Skill 名称规则；即单个文件系统安全段，只允许 ASCII
  字母数字及 `.`, `_`, `-`，不允许以 `.` 开头，并受共享路径段长度限制。
- `description`：必填、trim 后非空，UTF-8 大小不超过 4096 bytes。
- manifest 文件：UTF-8 且不超过 1 MiB。

插件发现只消费以上两个字段。Markdown body 和未知 front-matter 字段不参与插件身份、Catalog
metadata 或有效性判断；它们仍留在原 package 中供消费端读取。YAML 不是 mapping、字段类型不是
string、front matter fence 不完整或 YAML 非法都使该 Skill manifest 非法。

`SKILL.md.name` 必须与其直接父目录名大小写不敏感地相等；同一个插件内的 Skill 名也按 ASCII
lowercase 后去重。目录负责稳定定位，manifest 负责公开名称，两者不能表达两个不同身份。

### D4：Skill 插件是无进程、无运行时权限的静态插件

Skill contribution 不携带 `main.js` entrypoint，不创建 Deno runtime，不参与 plugin handshake，不能
activate 或注册 host methods，也不会获得 `--allow-*` 权限。它仍然是可列出、可更新、可卸载的
Installed Plugin；“无进程”不是“无生命周期”。

Skill package 中的脚本只是随 Skill 一起交付的内容。Ora 插件宿主不执行、预检或授予这些脚本
插件运行时权限；脚本是否由消费 Skill 的 Agent 使用，属于 Agent/Skill 执行边界。

### D5：Catalog 身份由 Plugin ID 与 canonical Skill name 共同派生

每个 contributed Skill 的持久身份为：

```text
plugin:<canonical-plugin-id>:<canonical-skill-name>
```

其中 `<canonical-skill-name>` 是 `SKILL.md.name` 的 ASCII lowercase。例如插件
`official/review-pack` 中的 `Review` 对应 Skill ID
`plugin:official/review-pack:review`。Catalog 行同时保存：

| 字段 | 值 |
| --- | --- |
| `id` | `plugin:<plugin-id>:<canonical-skill-name>` |
| `namespace` | 完整 canonical Plugin ID，而不是 marketplace source namespace |
| `name` | `SKILL.md` 中通过验证的原始名称拼写 |
| `description` | `SKILL.md` 中 trim 后的描述 |
| `origin` | Plugin ID 与该 Skill 的 immutable package root |

同名 Skill 可以由不同插件贡献，因为 Plugin ID 是身份的一部分；它们在 Catalog 中不会碰撞。下游如何
选择或处理这些同名 Catalog 项不改变插件提供的身份，也不应通过提前合并 Catalog 行消除差异。

插件版本更新沿用同一 Skill ID：repository 在一个 SQLite transaction 中先隐藏该插件当前投影，
再按新版本贡献集合 upsert Skill。新版本移除的 Skill 会退出 active Catalog；其他插件贡献的同名
Skill 不受影响。

### D6：Plugin Skill 是只读 Catalog 项，availability 来自插件 package

应用层把 Plugin Skill 显式投影为 `SkillSource::Plugin { plugin_id }`。读取详情时直接从该 Skill 的
plugin package root 读取 `SKILL.md`；更新和用户删除接口拒绝修改 Plugin Skill，不能把插件拥有的目录
误当作本地可写 Skill storage。

Catalog 行存在不等于 package 当前可用。每次列出或读取 Skill 时，宿主重新读取 package root 的
`SKILL.md` 并运行相同 manifest 校验；文件缺失、不可读或失效时返回 `Unavailable`，而不是把数据库
metadata 当作内容仍存在的证明。修复或重新安装 package 后，同一 Catalog 身份可以恢复为
`Available`。

### D7：Catalog 同步由受控生命周期事件驱动，外部文件漂移只做有限观测

以下路径会同步 Plugin Skill projection：

- 后端启动：发现当前有效的已安装 Skill 插件并重建其投影；
- marketplace 安装、更新与本地 `.orax` import：package 落盘后同步该 Plugin ID，再刷新 lifecycle
  snapshot；
- 显式卸载：移除该 Plugin ID 的 active Catalog Skill。

当前实现不监听 installed tree，也不在启动同步中对数据库做“所有已投影 Plugin ID 减去当前发现集合”
的全量清扫。因此用户或外部工具绕过 Ora 删除插件目录、把最高版本破坏到无法发现，可能留下
`Unavailable` 的旧 Catalog projection；显式卸载和后续有效安装仍是受支持的收敛路径。这个限制不能
被描述成“启动时自动清理所有孤儿插件 Skill”。

## 为什么不是这些替代方案

| 方案 | 否决理由 |
| --- | --- |
| 在 `orax.toml` 中再声明 `[[skills]]` | 产生 manifest 与文件树两份贡献清单，更新时需要定义缺失、重名和路径漂移的优先级；固定目录已经能完整表达集合 |
| 递归扫描插件包内任意 `SKILL.md` | 作者无法从浅层包布局直接判断公开贡献集合，也会把 references 或 vendored 内容中的 manifest 意外暴露成插件 API |
| 单个 Skill 非法时接受其余 Skill | 同一个 Plugin ID/version 在不同宿主或解析版本上可能投影出不同集合，插件“安装成功”的含义变得不完整 |
| 把插件 Skill 复制进本地可写 Catalog 根 | 混淆 Plugin ownership 与用户 ownership，副本也无法随插件更新、卸载和 package availability 保持同一事实来源 |
| 用裸 Skill name 作为 ID | 不同插件不能合法贡献同名 Skill，插件更新与卸载也无法按 owner 精确替换或退休 |
| 让 Skill 插件启动进程提供内容 | 为静态目录引入 handshake、权限和进程失败面，且无法获得比 checksum-verified package 更稳定的 revision 输入 |

## 不变量

1. `orax.toml` 的 Plugin ID 和版本定义插件身份；`SKILL.md` 的 name 定义该插件内的 Skill 身份，两者
   不能互相替代。
2. 一个已发现 Skill 插件的每个 `assets/` 直接子目录都必须是完整有效的 Skill package；不存在部分
   接受的 installed snapshot。
3. Plugin Skill ID 必须稳定派生为
   `plugin:<canonical-plugin-id>:<canonical-skill-name>`，namespace 必须保存完整 canonical Plugin ID。
4. Plugin Skill package 由插件拥有且只读；Catalog API 不修改或删除其中内容。
5. Catalog metadata 与 package availability 是两个不同事实：Catalog 行不能证明 package 当前可读。
6. Plugin Skill 不创建 runtime、不注册方法，也不获得插件进程权限。
7. 显式卸载必须按 Plugin ID 撤销 active Skill projection；外部文件系统删除不等价于受控卸载。

## 当前未解决的问题

- installed tree 的持续监听、外部删除/损坏后的孤儿 projection 全量清扫与安全恢复策略尚未实现。
- `scripts/`、`references/` 和嵌套 `assets/` 的更细 schema、权限声明与宿主解释不属于当前版本；它们
  只是 Skill package 内容。
- 普通 Skill import 与 Plugin Skill 安装保持两条不同入口；本决策不统一它们的布局、事务或逐候选
  失败语义。
- Catalog 中的 Plugin Skill 如何成为 Effect Source、进入 Workspace Desired State 并收敛到 Consumer
  Resource，由 `desktop/core/effect/skill` 领域决策定义。
