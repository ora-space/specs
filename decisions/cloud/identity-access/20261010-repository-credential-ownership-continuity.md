---
status: proposed
---

# 仓库凭据归属连续性：把 `0023` 已交付的 attribution／availability／冻结与审计语义补录为 Cloud 的历史契约

`credential_refs` 的归属与可用性语义已经随 `0023_repository_credential_continuity.sql` 交付并在真实
PostgreSQL 上生效，但 `specs/` 内没有任何 Cloud ADR 承载它——规则只写在迁移正文、迁移 README 与
`cloud/docs/runtime-control.md` 里。本文是这份**历史契约的补录**：逐条确认 `0023` 引入的六个列、两个
冻结触发器、恢复扫描与两处凭据引用列的既有语义，把已经生效的行为写成可评审、可逐条核对的决策。

**补录的方向已由项目负责人在运行 issue `ora-space/circus#29` 的授权中确认，且该确认不改写迁移。**
本文**尚未经过 specs 的 PR 评审**，按 `specs/AGENTS.md` 保持 `proposed`，评审通过后再据此标记
`approved`；而 `0023` 早已应用、校验和不可改写，其行为不因本文的状态而变化，评审也不重写迁移正文。
本文**只做记录与确认**：
不修改任何已应用的 migration，不为既有语义新增写入者或读路径，不授权实现变更，也不解除任何
fail-closed 门禁（`repository_credential_executor_unavailable` 继续有效）。若今后认为某条既有语义应当
改变，那属于前向决策，不在本文范围内。

涉及模块：Cloud（`internal/core` 的凭据准入、运行时控制与迁移）。兼容性边界：`credential_refs`、
`projects`、`clone_executions` 的现有 schema、已应用迁移 `0001`／`0023` 及其后的 `0024`–`0032`、
以及迁移 README 记录的历史决策都按硬边界处理。本文**不承担**的迁移义务：`0023` 之后新增的任何
schema、Runtime Control 的运行时／强停／创建者语义、凭据秘密本身的存储与解析（见
[用户 Git Token 由独立 Credential Service 承载](20261010-user-managed-git-secret-store.md)），以及
凭据的产品化写入路径（见 [用户自助 HTTPS Git 凭据](20261010-user-managed-git-credentials.md)）。

## 继承与修改

| 前序决策 | 处理 |
|---|---|
| [20260928 用户 Git 提交身份](20260928-user-git-commit-identity.md)（proposed，本叶子前序决策）：用户级 Git 材料只经专用端点写入，不进入公开响应与普通读取路径 | 继承，不变；本文补录的凭据元数据同样只经专用管理路径写入，公开 `Project` schema 不含 `secret_ref` |
| [cloud/operation 根决策](../operation/0-workspace-runtime-follows-desktop-node.md)（implemented）D2：Cloud 只提供 credential reference，由基础设施注入，Cloud 不保存密钥值 | 继承，不变；本文确认 `credential_refs.secret_ref` 自 `0001` 起就是"引用"而不是密钥值或密文 |
| 凭据只保存引用、类型、范围、能力、版本及可用性；成员离开原子冻结 `personal`／`unknown`；`team` 不因旧 owner 离开冻结；重新加入不自动解冻 | 出处是 `cloud/docs/runtime-control.md:29` 与 `0023` 迁移正文，**而不是任何 ADR**——这正是本文要补的缺口 |

本叶子此前的最后一篇决策是 `20260928`，因此它是本文的形式前序；本文对该决策只做继承，不修改其结论。

## 背景与现状

### `0023` 之前的既有形态

`credential_refs` 由 `0001_core.sql:18-22` 建立，列固定为 `id`、`tenant_id`、`owner_user_id`、
`purpose`、`secret_ref`、`version`、`created_at`、`deleted_at`：

```sql
CREATE TABLE credential_refs (
 id uuid PRIMARY KEY, tenant_id uuid NOT NULL, owner_user_id uuid NOT NULL, purpose text NOT NULL CHECK(purpose='git'), secret_ref text NOT NULL CHECK(length(secret_ref)>0),
 version bigint NOT NULL DEFAULT 1 CHECK(version>0), created_at timestamptz NOT NULL DEFAULT now(), deleted_at timestamptz,
 FOREIGN KEY(tenant_id,owner_user_id) REFERENCES tenant_memberships(tenant_id,user_id), UNIQUE(id,tenant_id,owner_user_id)
);
```

它已经表达了"凭据属于一个 tenant 内的一个成员，且只服务 Git"。它**没有**表达归属类别、可用性、范围
与能力，因此无法回答"成员离开后这条凭据还能不能用"。

### `0023` 引入的内容

`0023_repository_credential_continuity.sql` 是一个纯前向迁移，包含五组改动：

- 给 `credential_refs` 追加六个列：`attribution`（默认 `'unknown'`）、`attribution_basis`、
  `availability`（默认 `'available'`）、`unavailable_reason`、`repository_scope`、`capabilities`
  （`jsonb`，默认 `'[]'`，带数组类型 CHECK）。
- 增加 `CONSTRAINT credential_reference_scope UNIQUE(id,tenant_id)`。
- 增加 `projects.repository_credential_ref_id` 及其复合外键 `(repository_credential_ref_id, tenant_id)`
  → `credential_refs(id, tenant_id)`（后者的可行性正来自上一条唯一约束）。
- 给 `clone_executions` 追加 `credential_ref_id`（外键 `REFERENCES credential_refs(id)`）与
  `credential_ref_version`，并加配对 CHECK `(credential_ref_id IS NULL)=(credential_ref_version IS NULL)`。
- 建立两个冻结触发器与一次恢复扫描。

`0023` **不是**最后一个迁移：其后还有 `0024`–`0032`。`0023` 自身不因后续迁移而可改写——迁移文件的
校验和被启动校验，只能前向追加。

### 已经生效、可核对的行为

- **唯一的写入路径是部署 CLI。** `ConfigureCredential`（`internal/core/store.go:550-558`）由
  `cloudctl -command credential-ref` 调用，执行
  `INSERT INTO credential_refs(id,tenant_id,owner_user_id,purpose,secret_ref) VALUES($1,$2,$3,'git',$4)`。
  **该 INSERT 没有列出 `attribution`**，因此今天经它创建的每一条凭据都取列默认值
  `attribution='unknown'`——不是 `personal`。这是"为什么触发器把 `unknown` 与 `personal` 并列"的直接
  解释，也是将来产品化写入路径必须显式写 `personal` 的原因。
- **`attribution_basis` 与 `capabilities` 在 Go 侧零命中。** 全仓检索 `internal/` 与 `cmd/` 的
  `attribution_basis`／`attributionBasis`／`capabilities`：0 处。它们是只由测试用裸 SQL 赋值的文档性
  列，今天不参与任何判定。
- **准入读的是 `availability`、`attribution` 与 `repository_scope`。**
  `requireRepositoryAccess`（`internal/core/repository_credentials.go:6-26`）要求
  `availability='available'`（第 18 行），对 `attribution <> 'team'` 的行要求 owner 仍是
  `tenant_memberships` 与 `users` 双 active 的成员（第 19-21 行），并要求
  `repository_scope` 为空或与 `projects.repository_url` 全等（第 22 行）。**空 scope 即通配**，
  这是既有语义。
- **恢复扫描有两份实现，条件不完全相同。** 迁移尾部的 `UPDATE` 只判
  `attribution IN ('personal','unknown') AND NOT EXISTS(活跃成员)`，**没有** `availability='available'`
  前置条件；Go 侧的 `freezeInactiveRepositoryCredentials`（`repository_credentials.go:30-32`）多一个
  `AND c.availability='available'`，因此不会重写已冻结行的 `unavailable_reason`。Go 侧扫描由
  `refreshRuntimeControls`（`internal/core/runtime_control.go:15`）在每次运行时控制刷新时调用，所以它
  不只在升级时跑一次。两份实现对"成员已停用"的结论一致；差异只在已冻结行的原因码是否会被重写。
- **两处引用列都没有完整语义。** `projects.repository_credential_ref_id` 没有任何写入者；
  `clone_executions.credential_ref_id`／`credential_ref_version` 同样没有写入者——
  `workspace_clone.go:38` 的 `INSERT INTO clone_executions(...)` 列清单里没有它们。

结论：`0023` 交付的是一套**已经生效、但只被部分读取**的语义。补录的价值不在于新增规则，而在于把这套
语义从"迁移正文 + 运维文档"提升为可评审的 Cloud 决策，并如实记录它今天**没有**覆盖到的部分。

## 决策

### D1：`credential_refs` 是凭据元数据、归属与可用性的唯一权威记录

凭据的归属根是 `(tenant_id, owner_user_id)`，`purpose` 恒为 `'git'`。凭据的可用性、归属类别、仓库
范围与能力都表达在同一张表的列上，不另建并行的凭据主表。`secret_ref` 是"秘密的定位引用"，不是密钥
值、不是密文。

六个列的语义按 `0023` 的既有定义记录如下，**不新增取值、不放宽 CHECK**：

| 列 | 允许取值 | 默认 | 权威含义 |
|---|---|---|---|
| `attribution` | `unknown`／`personal`／`team` | `'unknown'` | 凭据归属类别；`unknown` 表示历史行的归属不可知 |
| `attribution_basis` | 自由文本 | `NULL` | 归属判定的依据说明；今天无读写者 |
| `availability` | `available`／`frozen`／`invalid` | `'available'` | 能否用于**新的**远程 Git 工作 |
| `unavailable_reason` | 自由文本 | `NULL` | 不可用的原因码；今天只有两个取值来源（D3） |
| `repository_scope` | 自由文本 | `NULL` | 允许匹配的仓库；**空值即通配** |
| `capabilities` | JSON 数组 | `'[]'` | 允许的操作集合；今天无读写者 |

`version` 在每次冻结时 `+1`，因此"凭据是否被冻结过"有单调的版本证据；`deleted_at` 自 `0001` 起只有
`IS NULL` 读取、没有写入者，不承担删除职责。

为什么把这些写进 ADR 而不是留在迁移里：迁移正文回答"当时改了什么"，不回答"系统现在承诺什么"。
规范要求状态转换、ownership 与持久化模型的变化有对应决策；冻结语义正是这三者的交集。

### D2：归属只有三个取值，历史行是 `unknown`，不猜测

- `personal`：凭据属于一个具体用户；成员或用户停用会冻结它（D3）。
- `team`：凭据属于租户而非个人；**不因旧 owner 离开而冻结**。
- `unknown`：`0023` 升级时对既有行的判定——迁移首行注释写明
  *"Existing owner keys and secret references are retained; historical attribution is unknown."*
  迁移 README 对应地写 *"team attribution and effective binding are never guessed"*。

因此 `unknown` **不是**一个可以新指派的类别，而是"无法证明是 `team`，因此按 `personal` 的方式保护"
的保守取值。这一保守方向是刻意的：把 `unknown` 与 `personal` 并列冻结，宁可误冻也不误放。

**`unknown` 的保守语义不得被静默改写**：在归属未经核实的情况下，任何路径都不得把既有 `unknown` 行
自动转换为 `personal`。批量转换会把"无法证明"变成"已证明"，同时改变冻结与解冻的适用范围以及
`version` 的语义（它今天单调记录冻结次数）。产品化写入路径必须对**新**凭据显式写 `personal`
（见 [用户自助 HTTPS Git 凭据](20261010-user-managed-git-credentials.md) D2）；既有 `unknown` 行的
重新归属必须是一次显式、可审计、带依据（`attribution_basis`）的前向变更，而不是一次 `UPDATE`。

需要如实记录的是：今天**没有任何**路径会写出 `personal` 或 `team`。`ConfigureCredential` 不列出
`attribution`，所以新建凭据落在 `unknown`。把新凭据标为 `personal`、以及为 `team` 定义授权与成员离开
规则，都是尚未交付的产品能力，不在本文范围（见
[用户自助 HTTPS Git 凭据](20261010-user-managed-git-credentials.md) D2）。

### D3：成员停用与用户停用原子冻结 `personal`／`unknown`，`team` 不因旧 owner 离开冻结

两个触发器都定义在 `0023`，都是"停用即冻结"的原子动作，方向单一、不可逆：

| 触发器 | 监听 | 触发条件 | 冻结范围 | 写入的 `unavailable_reason` |
|---|---|---|---|---|
| `departed_repository_credentials` | `AFTER UPDATE OF status ON tenant_memberships` | `NEW.status <> 'active'` | 与该成员同一 tenant、`owner_user_id` 相同、`attribution IN ('personal','unknown')`、当前 `available` 的行 | `associated_member_departed` |
| `disabled_user_repository_credentials` | `AFTER UPDATE OF status, deleted_at ON users` | `NEW.status <> 'active' OR NEW.deleted_at IS NOT NULL` | **该用户在全部 tenant 中**的 `personal`／`unknown` 且当前 `available` 的行 | `associated_user_disabled` |

两条规则的共同点与边界：

- 都只改 `availability`、`unavailable_reason` 与 `version`，不改 `secret_ref`、不改 `owner_user_id`、
  不改 `repository_scope`，因此**保留原 owner 外键与旧引用**。
- 都只作用于 `availability='available'` 的行，因此不会覆盖已经冻结的行。
- **`attribution='team'` 在两条规则里都被排除**：租户凭据不因某一个成员离开而失效。
- 触发器是 `AFTER UPDATE`：把凭据创建在停用状态成员名下不会触发冻结，但那条路径今天不存在——
  `ConfigureCredential` 先经 `membership(t, tid, owner, false)` 要求成员有效。
- 冻结是"能否用于**新的**远程 Git 工作"的判定。已完成执行的结算、结果查询、restart 与 cleanup 不走
  `requireRepositoryAccess`（`repository_credentials.go:3-4` 的注释即为此），因此不受冻结影响。

为什么是数据库触发器而不是应用层：成员停用与冻结必须在同一个事务里成立，否则会出现"成员已停用但
凭据仍可用"的窗口。把规则放在约束层，是让任何写入者（包括将来的产品路径）都无法绕过它。

### D4：重新加入不自动解冻；恢复扫描是补偿，不是第二条规则

- 两个触发器的 `IF` 只在**离开**方向成立：`NEW.status='active'` 时不做任何事。因此成员重新加入、
  用户被重新启用都不会把 `availability` 改回 `available`。解除冻结需要显式的、未来的前向决策，
  今天没有这条路径。
- `0023` 尾部的 `UPDATE` 是**升级时的补偿扫描**：它把"在 `0023` 应用之前就已经停用"的历史行补冻为
  `associated_member_departed`。迁移注释写明 *"Members already inactive at upgrade are frozen as well.
  Rejoining does not unfreeze a reference."*
- Go 侧的同名扫描（`freezeInactiveRepositoryCredentials`）在每次 `refreshRuntimeControls` 时运行，
  用于捕获"触发器建立之后、部署侧状态变化"的遗漏。它的判定与迁移扫描一致，只是多了
  `availability='available'` 前置条件，因此不会重写已冻结行的原因码。

因此"重新加入不自动解冻"有两层保证：触发器不反向动作，补偿扫描也不反向动作。这条边界必须保持——
自动解冻会让一个已经离开组织的成员的个人凭据在重新加入时静默恢复可用。

### D5：`repository_scope` 与 `capabilities` 已就位，但今天没有产品写入路径

- `repository_scope` 是准入实际使用的列：`requireRepositoryAccess` 要求它为空或与
  `projects.repository_url` **全等**（`repository_credentials.go:22`，不匹配返回
  `409 repository_credential_scope_mismatch`）。**空值即通配**——今天经 `ConfigureCredential` 创建的
  凭据都不带 scope，因此在准入层面等价于"对该 host 上任意仓库可用"。
- `capabilities` 没有读写者，`'[]'` 的默认值不表达任何授权；今天它既不放行也不阻止任何操作。
- 本文**不改变**这两列的语义：不把空 scope 收紧为必需（那会改变既有行的可判定结果），也不给
  `capabilities` 增加判定。把它收紧为"第一版必需的单仓库许可"，属于
  [用户自助 HTTPS Git 凭据](20261010-user-managed-git-credentials.md) D2 的前向要求。

为什么要在补录里点明"空 scope 即通配"：它是今天真实生效的边界，而直觉上"没有 scope"会被读成"没有
权限"。把实际方向写清楚，才能让后续"scope 必需"的收紧被当作**收紧**而不是**修正笔误**来评审。

### D6：两处凭据引用列与审计列的历史来源

`0023` 同时引入了两组引用列，它们的历史意图与今天的地位不同：

| 列 | 出处 | 外键 | 是否强制 owner 一致 | 今天的写入者 | 本文的记录 |
|---|---|---|---|---|---|
| `projects.credential_ref_id` | `0001_core.sql:25,28` | `(id, tenant_id, owner_user_id)` → `credential_refs` | **是** | `createProject`（`internal/core/public.go:549-556`） | 最早、唯一有写入者的 Project 绑定列 |
| `projects.repository_credential_ref_id` | `0023` | `(repository_credential_ref_id, tenant_id)` → `credential_refs(id, tenant_id)` | **否** | 无 | `0023` 为"仓库凭据"预留的第二条绑定，靠新增的 `credential_reference_scope UNIQUE(id, tenant_id)` 才成立 |
| `clone_executions.credential_ref_id` ＋ `credential_ref_version` | `0023` | `credential_ref_id` 单列 → `credential_refs(id)`（**不带 tenant 范围**） | 不适用 | 无（`workspace_clone.go:38` 的 INSERT 未列出） | 为"某次 clone 用了哪个凭据的哪个版本"预留的审计事实；配对 CHECK 保证两者同时有值或同时为 NULL |

本文只记录它们的**历史来源**：`repository_credential_ref_id` 的引入意图是承载"仓库维度的有效凭据
绑定"，但 `0023` 没有交付任何写入者，也没有交付 owner 强制；审计列同理，只有 schema 与公开契约声明
（`internal/contract/openapi.go:279-280`）而无写入。

V1 的权威绑定列**不由本文裁决**——[用户自助 HTTPS Git 凭据](20261010-user-managed-git-credentials.md)
D7 已裁决 V1 以 `projects.credential_ref_id` 为准，另一列保留不删、不参与回退，两列冲突必须显式拒绝
（同文 D10、不变量 9）。本文确认这一裁决与补录的历史事实不矛盾：权威列正是"已有写入者 + 外键已强制
owner"的那一列，而另一列今天既无写入者也无 owner 强制。启用审计列写入是前向要求（同文 D8），
不是本文的补录内容。

### D7：本文不改变任何已实现行为

补录的性质决定了它的边界，逐条列出以便评审核对：

- 不改写 `0001`／`0023` 及其后的任何迁移文件，不新增 migration。
- 不改 `requireRepositoryAccess` 的判定顺序、错误码或无条件拒绝；`repository_credential_executor_unavailable`
  继续有效。
- 不新增或删除冻结规则的触发条件，不引入自动解冻，不改变 `unknown` 与 `personal` 的并列处理。
- 不给 `attribution_basis`、`capabilities`、`repository_credential_ref_id`、
  `clone_executions.credential_ref_id`／`credential_ref_version` 增加写入者。
- 不收紧 `repository_scope` 的空值语义；不改变 `deleted_at` 的既有读取方式。
- 公开 API 形状、匿名公共 Git 路径、既有 Workspace 的恢复与清理都不变。

## 不变量

1. `credential_refs` 是凭据元数据、归属与可用性的唯一权威记录；`secret_ref` 不是密钥值或密文。
2. 凭据的归属根是 `(tenant_id, owner_user_id)`，`purpose` 恒为 `'git'`；归属类别只有
   `unknown`／`personal`／`team` 三个取值。
3. `unknown` 是历史行的保守取值，与 `personal` 一并受冻结保护；不得被解释为"无归属因而无约束"，
   也不得在归属未经核实时被自动改写为 `personal`。
4. 成员停用与用户停用会原子冻结相应 `personal`／`unknown` 凭据，并保留 `secret_ref`、`owner_user_id`
   与 `repository_scope` 不变。
5. `attribution='team'` 的凭据不因某个成员离开组织而冻结。
6. 重新加入组织或被重新启用**不会**自动解冻既有的冻结凭据；解除冻结需要显式的前向决策。
7. 冻结只影响能否用于**新的**远程 Git 工作；已完成执行的结算、结果查询、restart 与 cleanup 不受影响。
8. `repository_scope` 为空表示不限制仓库；该语义不得被静默收紧为"无权限"。
9. 已应用的 migration 不可改写；本文不新增、不修改任何迁移。
10. `projects.repository_credential_ref_id` 与 `clone_executions` 的凭据审计列今天没有写入者；
    在权威列裁决（Cloud ADR D7）与审计写入要求（Cloud ADR D8）落地之前，不得假定它们承载事实。

## 为什么不是这些替代方案

| 方案 | 优势 | 否决理由 |
|---|---|---|
| 不补录，继续让规则只存在于迁移与 `cloud/docs/runtime-control.md` | 零文档改动，`0023` 已经是事实 | 规范要求 ownership 与持久化模型的变化有对应决策；没有 ADR 就无法判断某次改动是"修正笔误"还是"改变已承诺的语义"，而冻结语义是有损且不可逆的 |
| 补录同时把 `unknown` 收紧为"必须显式指派" | 归属更明确 | 会改变既有行的可判定结果——今天经部署 CLI 创建的凭据都是 `unknown` 且必须继续受保护；那是前向产品决策，不是补录 |
| 借补录把空 `repository_scope` 收紧为必需 | 消除"意外通配" | 同样改变既有行的判定结果；且收紧是前向要求，已在 Cloud ADR D2 承载 |
| 借补录给 `attribution_basis`／`capabilities` 加上判定 | 让预留列"有用" | `capabilities` 一旦参与判定就会放宽或收紧今天的行为；预留列在补录里只应记录为"无读写者" |
| 用一次 `UPDATE` 把既有 `unknown` 批量改写为 `personal`（"归属其实已经明确"） | 让语义立刻统一 | 未核实归属就改写，会把"无法证明是 `team`"变成"已证明属于某个用户"，并顺带改变冻结与解冻的适用范围与 `version` 的语义；重新归属必须是显式、可审计、带 `attribution_basis` 的前向变更（D2、[用户自助 HTTPS Git 凭据](20261010-user-managed-git-credentials.md) D2） |
| 在补录里顺手修正迁移尾部扫描缺少 `availability` 前置条件的差异 | 两份实现更一致 | 迁移文件不可改写（校验和），改写它等于改变已应用的 DDL；差异本身不影响停用判定，只需如实记录 |
| 把补录直接标为 `approved`，因为它描述的是既有事实 | 少一轮评审 | 补录固定的是"系统从此承诺什么"，与新建决策一样需要纠偏；规范明确不能仅通过修改状态字段代替评审 |
| 把 `0023` 的全部历史设计（含 Runtime Control）一并补录 | 一次写完 | 会把运行时、强停与创建者语义拖进来，超出"凭据归属连续性"这一件事，也超出本叶子领域 |

## 失败与恢复语义

| 情形 | 对外结果 | 持久事实与恢复 |
|---|---|---|
| 成员离开 tenant | 该成员的个人凭据不再可用于新的远程 Git 工作 | 触发器在同事务内改 `availability='frozen'`、原因码 `associated_member_departed`、`version+1`；`secret_ref` 与 owner 外键保留 |
| 用户被停用或软删 | 该用户**在所有 tenant 中**的个人凭据同样冻结 | 原因码 `associated_user_disabled`；不按 tenant 区分 |
| `team` 凭据的旧 owner 离开 | 凭据状态不变 | 触发器排除 `team`；成员离开不改变租户凭据 |
| 成员重新加入 | 凭据仍为冻结，不恢复可用 | 触发器只在离开方向动作；没有解冻写入者 |
| `0023` 升级时已存在停用成员 | 其历史凭据被补冻 | 回复扫描写入 `associated_member_departed`；注释明确重新加入不解冻 |
| 部署侧状态变化绕过触发器 | 由运行时控制刷新时的 Go 扫描补齐 | 判定与升级扫描一致，且不重写已冻结行的原因码 |
| 凭据 `repository_scope` 为空 | 准入不因 scope 拒绝 | 空值即通配（不变量 8） |
| 凭据引用的仓库与 Project 的 `repository_url` 不同 | `409 repository_credential_scope_mismatch` | 准入拒绝，不修改凭据 |
| 存储引用不可解析（Store 未交付） | `409 repository_credential_executor_unavailable` | 保持 fail-closed，与 `0023` 无关，也不因本文改变 |

## 本决策未解决的问题

- **迁移尾部扫描与 Go 扫描的原因码差异是否要收敛**：迁移的 `UPDATE` 缺少 `availability='available'`
  前置条件，因此可能把一条已按 `associated_user_disabled` 冻结的行改写成
  `associated_member_departed`（当该用户同时也不在该 tenant 的活跃成员中）。两份实现对"是否冻结"的
  结论一致，差异只在原因码。是否需要一个前向决策来统一原因码语义，留待人工裁决——**不在本文改**。
- **`unknown` 的最终去向**：产品化写入路径落地后，`unknown` 是否成为只读的历史值，还是保留为"归属
  判定失败"的当前取值，需要与 `personal`／`team` 的写入规则一并裁决。
- **`attribution_basis` 的用途**：是记录产品侧的归属判定依据，还是仅作历史审计留存。今天它没有读写者，
  也无法从既有数据推断设计意图。
- **解冻的管理路径**：不自动解冻已经明确，但是否需要（以及由谁发起）一条显式的解冻或凭据重建路径，
  属于运维与安全裁决。
- **审计列的启用时机**：`clone_executions` 的凭据审计列由 Cloud ADR D8 要求启用；启用属于前向变更，
  其验收证据（与执行记录同事务写入、失败时不缺失）需要单独设计。

## 落地顺序

本文是补录，**没有实现步骤**：它记录的行为已经生效并持续有效。可独立验收的动作只有评审动作本身。
**三步的核对工作已经做过，但结论随本文的评审一并生效**：本文保持 `proposed`，评审通过并标记
`approved` 后才算完成。这些步骤都不产生代码或测试证据，因此不构成实现或解禁的依据。

1. **评审并确认既有语义（核对已完成，结论待评审确认）**：D1–D6 与真实 `0023`、`requireRepositoryAccess`、
   `freezeInactiveRepositoryCredentials` 逐条核对一致，每条都能在迁移正文或现有 Go 源码中直接指出出处。
2. **确认补录不改变行为（核对已完成，结论待评审确认）**：按 D7 的清单逐项核对，没有列被新增写入者或读路径；
   `git -C cloud diff` 对 `internal/core/migrations/` 与 `internal/core/*.go` 为空。
3. **与本叶子其余凭据决策对齐（核对已完成，结论待评审确认）**：本文与
   [用户自助 HTTPS Git 凭据](20261010-user-managed-git-credentials.md)、
   [用户 Git Token 由独立 Credential Service 承载](20261010-user-managed-git-secret-store.md)、
   [凭据规则受限例外](20261010-user-managed-git-token-ingress-rule-exception.md) 对
   `availability`／`attribution`、冻结与归属的表述一致。

保持不变的行为：本文列出的全部内容——六个列的语义与取值、两个冻结触发器的条件与方向、
"重新加入不自动解冻"、空 scope 的通配语义、两处引用列与审计列的"无写入者"现状、以及
`repository_credential_executor_unavailable` 的 fail-closed 行为。从本文开始改变的行为：**没有**。
