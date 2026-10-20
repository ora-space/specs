---
status: implemented
---

# Workflow 插件资产由 Cloud 自行取包、逐文档导入为可编辑副本

> 后续决策：[部署方共享出网策略](../model-access/20261017-deployment-owned-egress-and-explicit-resolution.md)
> 后续决策的 D3 企业 HTTPS 取包已有部分实现，完整共享出网验证仍按后继用例逐项核对。
> 本文的 `implemented` 只对应原有工作流资产导入；既有证据不自动扩大到企业出网，非企业模式仍沿用原公网行为。

`kind = "workflow"` 的插件包是「工作流文档的投递载体」，不是插件代码。Cloud 在**资产导入步骤**
里自己下载该插件版本的 `.orax`、校验 sha256、限定大小，把 `assets/workflows/` 下的每份 `*.json`
导入为该租户的一份普通工作流并发布首个快照。导入产物是**用户数据**（可编辑、可改名、可删除），
来源身份另存为一条独立记录，只服务于幂等、对账与展示，不构成所有权。

这明确扩展了根决策「Cloud 不下载包、读取永远不出网、Node 不自行解析市场」的边界：扩展**仅限
资产导入**，安装落盘仍只由 Node 完成，Cloud 不把包写进任何 Workspace，取到的字节用完即弃。
四种证据——安装成功、接管成功、发布可用、执行可用——是四个可以分别判断的事实，任何一条都不能
推导出另一条。

本 ADR 已 `implemented`：Cloud 侧实现见 [cloud#77](https://github.com/ora-space/cloud/pull/77)，
方案、边界与不变量经 owner 复审确认，核心用例见
[workflow 插件资产核心用例](../../../test-cases/cloud/plugin-marketplace/workflow-plugin-assets.md)。
资产导入与运行契约的接缝写在
[workflow 后续决策](../workflow/20261016-asset-import-and-published-snapshots.md)；包内容格式由
[workflow 插件类别根决策](../../desktop/plugin/category/workflow/0-workflow-package-format.md)
拥有，本文不重复。本 ADR 不承担既有租户的回填迁移义务：只对生效后新触发的导入负责。

## 继承与修改

| 前序约定 | 本文处理 |
|---|---|
| [根决策](0-plugin-marketplace.md) D1：`space_plugins` 选择状态与 `workspace_plugin_instances` 执行实例两级分离，聚合规则 | 继承，不变：导入结果不参与插件实例聚合，也不改变 `installed` 的含义（D5） |
| 根决策 D2：fan-out 到 live workspace，受 `one_project_operation` 串行 | 继承，不变：资产导入不 fan-out、不建 operation、不占项目独占（D3） |
| 根决策 D3：目录快照由 Cloud git 同步，读取永远不出网 | **修改**：目录读取仍不出网，但资产导入会按目录快照记录的 `url` 自行下载 `.orax`，这是新增的出网点，仅限资产导入；scheme/host、SSRF、重定向与超时在 D3 逐条固定（D3） |
| 根决策 D4：`plugin_ensure` 载荷与 `installed=true` 证据 | 继承，不变；资产导入的证据不是插件安装的证据（D5） |
| 根决策不变量 2：外部副作用前权威状态与 operation 已落库；网络/下载绝不在数据库事务内 | 继承，并扩展：下载、校验与解析都在事务外，导入结果落库在事务内（D3） |
| 根决策不变量 4：sha256 校验必选 | 继承，不变：导入前先校验，且先于任何解析（D3） |
| [管理员选择决策](20260927-admin-selection-and-idle-maintenance.md) D1：只有当前管理员可以改变期望 | 继承，不变：导入由管理员改变期望触发，不新增入口（D6） |
| 管理员选择决策不变量 5：共享汇总不暴露受限日志，Node 结果才证明实例安装结局 | 继承，不变：导入结果与安装结局分别可见（D5） |
| [Node 执行安装决策](20260928-node-executes-plugin-installs.md) 标题与不变量 3：安装与移除由 Node 执行，实例状态只由 Node 执行结果写入 | 继承，不变：**安装落盘仍然只由 Node 完成**，本文其余部分不触碰它（D3） |
| Node 执行安装决策 D1：执行结果逐项、失败码有界 | 形式沿用（逐文档结果同样有界失败码），语义不同：那是安装落盘，这是资产导入（D4） |
| Node 执行安装决策不变量 1：载荷由 Cloud 从目录快照拼装，Node 不自行解析市场 | **修改**：Cloud 自己取包，但**仅用于资产导入**；仍然「Node 不自行解析市场」，且 Cloud 不把包交给 Node（D3） |
| [desktop plugin registry 决策](../../desktop/plugin/registry/20260831-source-derived-namespace.md)：`workflows`、`agents` 的 namespace 恒为 `local` 且插件不向它们贡献任何行 | 继承，不变：导入副本是 `local` 用户数据，插件身份只存在于来源记录中，不进 workflow 行的身份（D1） |
| 根决策 v1 边界：pack 禁装（目录可列出） | 继承，不变：pack 携带文档的选择语义属于 pack 自己的决策，不在本文（「本决策未解决的问题」） |

## 决策

### D1：资产模式 A —— 导入为可编辑副本，来源另存

每份文档导入为当前租户的一份普通 workflow（沿用编辑器同一套创建规则）与一个已发布快照；导入产物
与用户在编辑器里新建、发布的工作流在存储与运行契约上没有区别。

来源记录至少包含：

```text
source_record = (source_namespace, identifier, document_path, package_version, package_sha256)
                + 结果 + 触发 space + 时间
skip_key      = (source_namespace, identifier, document_path)
```

- `(source_namespace, identifier)` 是 canonical Plugin ID，`document_path` 是包内路径
  （`assets/workflows/<name>.json`）。canonical 身份用显式列对保存，不塞进 JSON。
- `package_version` 与 `package_sha256` 说明这条结果出自哪个包版本，只用于对账与展示，**不参与
  「是否跳过」的判定**：判定键是 `skip_key`，跨包版本成立（D4）。
- 记录还保存触发它的 space、最近一次尝试的结果与时间；用户对副本改名、编辑、归档都不改写这条记录。
- 来源不构成所有权：副本可编辑、可改名、可归档；任何地方都不能用「它来自某个插件」阻止用户改动，
  也不能因为插件卸载、来源删除或市场下架而回收副本。

选择模式 A 同时让 registry 决策继续成立：插件不以插件身份向 `workflows` 贡献行，导入副本的身份是
`local` 用户数据，来源只在独立记录里。这与 desktop 已实现的语义一致
（`ora-plugin-manifest::PluginKind::Workflow`：包是文档投递载体，宿主把文档按用户数据导入工作流库，
被导入的工作流比投递它的包活得久；卸载插件从不移除它）。

### D2：归属保持 tenant 级，空间只记入来源

- 不给 `workflows` 加 space 列，也不把 workflow 变成空间私有资源：它继续是租户级、编辑器与 `@`
  共用同一份。
- 「哪个空间的期望触发了这次导入」只是来源记录的一段事实，不决定可见性。
- 本决定依赖[一租户一空间](../tenancy/20260923-one-space-per-tenant.md)不变量。引入多空间时必须
  重新定义跨空间可见性：同一份副本不能同时是「空间 A 的模板」和「空间 B 不可见」，来源里的 space
  字段无法承担这个语义。
- 名称唯一性沿用现有 tenant 级规则（`workflows(tenant_id, name) WHERE deleted_at IS NULL`），
  因此跨空间同名文档按 D4 逐文档失败，而不是并存两份。

### D3：触发点与取包 —— Cloud 在资产导入步骤自行下载 `.orax`

**触发**：space 的期望状态出现某个 `kind = "workflow"` 插件时，由 Cloud 侧的资产导入步骤执行。
触发**不依赖 Node 安装结果**：没有 live workspace 的新空间也要拿到文档，不能出现「空间已聚合
`installed` 却没有任何模板」。这也意味着导入不与安装 fan-out 配对，不需要 `one_project_operation`
的独占。

每次触发把来源记录的 revision 加一，并且**一次扫描出的待办只在 revision 未变时才能领取**：管理员
在扫描之后、领取之前重装插件，旧待办按新的 revision 做 CAS 失败，本轮丢弃，下一轮按新选择重新
解析包身份。结果写入同样以领取时的 revision 围栏，下载期间的重装丢弃整次尝试。因此「已经选中的包
身份」与「正在写入的 revision」不会错配，旧包内容不会占用新 revision 的判定键。

**取包**：载荷与 effect 一样来自目录快照，但资产导入是 Cloud 新增的一个出网点。根决策的「读取永远
不出网」说的是目录读取，T1b 取包不受它覆盖，因此这里逐条固定边界：

- workflow kind 是平台无关的通用发布资产，永远只需 `universal{url, sha256}`。`workflow` 不允许
  `[[targets]]` / `[artifact]`（`may_ship_targeted_artifact()` 对它为假），因此导入不需要任何
  target 选择与主机兼容性判断。
- 边界：sha256 必选且先于任何解析；只接受 `assets/workflows/` 直接子层的 `*.json`（README、
  子目录、符号链接忽略），份数 1..=256，与 desktop 的 `MAX_WORKFLOWS_PER_PACKAGE` 一致。
- 大小上界：压缩包 ≤ 16 MiB、解压后总量 ≤ 64 MiB、归档条目数 ≤ 512。超限以
  `asset_package_too_large` / `asset_package_expansion_too_large` 拒绝整个包。包体只承载 JSON
  文本，上界要在「覆盖真实包」与「限制 Cloud 从远端取字节的风险面」之间取；解压上界才是真正的
  控制（防 zip bomb），压缩上界只是提前失败。三条各自依据如下：
  - **条目数 512 是从本文的文档上限推出的**：每份文档占一个条目，份数上限 256（与 desktop 的
    `MAX_WORKFLOWS_PER_PACKAGE` 一致），余量留一倍给 `orax.toml`、README、Logo 这类辅助文件。
    desktop 的通用默认是 5000 个条目（`crates/utils/src/archive/limits.rs` 的
    `ExtractLimits::default()`），插件安装只上调字节预算、条目与路径上限仍沿用这个默认
    （`crates/plugin-manager/src/limits.rs` 的 `package_extract_limits`）。workflow 包只有文档，
    用不到那个量级，而条目上限正是防「大量空目录/空文件」的那一层。
  - **两个字节上界是取值，不是推导来的**：desktop 的插件安装自报 512 MiB 压缩 / 1024 MiB 解压
    （`crates/plugin-manager/src/limits.rs` 的 `MAX_PACKAGE_ARCHIVE_BYTES`、
    `MAX_PACKAGE_TOTAL_BYTES`），通用默认是 50 MiB / 200 MiB——前者为运送 CLI 二进制而放宽，后者
    为文本包而设。Cloud 从远端取字节且只读 JSON，取比通用默认更紧的一档：16 MiB 意味着 256 份
    文档平均每份可占 64 KiB，远超一份图 JSON 的量级（真实包通常只带几份文档），64 MiB 是解压后的
    硬顶。这两个数值是取值，经 owner 复审确认。
- **出站请求的约束**（T1b 是新增出网点的代价，逐条固定）：
  - **定位符**：`universal.url` 在目录同步时已按 `validateLocator` 校验，只有两种形态——无凭据
    的 HTTPS URL（允许带签名的 query，不允许 fragment），或 S3 object key。前者由 Cloud 直接
    HTTPS GET；后者不使用公网，按 Cloud 自己的对象存储配置读取（与 `objectstore.Verify` 同一条
    私有端点）。因此「出网」只发生在 HTTPS 形态。S3 object key 形态本轮不变：Cloud 的插件之后会
    统一放在 cluster 内部的插件专用 S3 上，该形态既不经过公网，也不新增凭据。
  - **scheme 与 host**：HTTPS 形态只接受 `https`（目录扫描已拒绝 `http`、`s3://` 与其他 scheme）。
    非企业模式 host 不设白名单。企业模式按[共享出网后续决策 D1–D7](../model-access/20261017-deployment-owned-egress-and-explicit-resolution.md)
    使用与 model-gateway 相同的 `deployment-egress.json`、精确 host:port 集合及部署 DNS；未列出的
    公网/私网均拒绝，目录快照受信不能代替网络准入。签名 query 继续有效，不套用模型 baseUrl 的无 query 限制。
  - **SSRF**：非企业模式解析与连接沿用 Cloud 既有的公网出网策略（`internal/modelgateway/transport.go` 的
    `publicAddress` 与 `secureDialer`）：连接前解析全部 DNS 应答，任一落在私有、环回、链路本地、
    未指定或文档/保留网段即拒绝；不使用环境代理。判定在连接时做，所以被篡改的目录条目也打不到
    内网。企业模式按共享策略显式许可私网，但 cluster/宿主/回环/link-local 拒绝不可覆盖；两者均
    禁用环境代理、直接拨验证后的数字地址、按原 hostname 校验 TLS。企业 DNS 使用显式 resolver 或
    静态映射，无系统回退；非企业使用共同部署 DoH 配置，下载器不能单独写死 Cloudflare。
  - **重定向**：不跟随（`CheckRedirect` 返回 `ErrUseLastResponse`），任何 3xx 按下载失败处理。
    跟随重定向会让刚做过的地址判定作废，且与 `objectstore.Verify`、模型网关客户端的既有做法不一致。
  - **超时与流式上界**：连接 10 秒、等待响应头 30 秒、从开始到读完整个包 60 秒；下载边读边计数，
    超过 16 MiB 立即中断，不等读完再判断。三个超时是取值，经 owner 复审确认。
  - **失败归属**：下载、校验或解压失败是包级结果，不产生任何文档结果；下一次触发重试。
- 下载、校验、解压与解析全部在数据库事务外；只有逐文档结果在事务内落库（根决策不变量 2）。

**明确扩展与明确不扩展**：本文扩展「Cloud 不下载包」与「读取永远不出网」两条，仅限资产导入这一
处——目录本身仍只在 git 同步里读取、不出网，这一点不变。不扩展的是「安装落盘只由 Node 完成」与
「Node 不自行解析市场」：Cloud 取到的字节只用于解析文档，不写入任何 Workspace，也不经过 Node。

### D4：逐文档导入、有界失败码与幂等升级

一份文档一个结果：

| 结果 | 内容 |
| --- | --- |
| `imported` | `workflow_id`、`snapshot_version` |
| `failed` | 有界错误码：`document_invalid_json`、`document_without_name`、`document_not_executable_graph`、`name_conflict` |

`snapshot_version` 是 Cloud 发布规则分配的下一个整数，不是文档里写的版本，所以没有「版本不可寻址」
这一类失败（见 [workflow 后续决策](../workflow/20261016-asset-import-and-published-snapshots.md) D1）。
`document_not_executable_graph` 覆盖发布门禁的全部拒绝：首期不执行的节点 kind、首期不执行的 agent
配置字段、以及节点引用了未声明的槽位；失败结果带上门禁的问题列表（节点、槽位或字段），与编辑器
发布时的报错同源。

`document_invalid_json` 同时覆盖「能解析、但 Cloud 无法原样保存」的文档：PostgreSQL 的 `jsonb`
输入拒绝转义 NUL（`\u0000`）、未配对的代理项转义、非法 UTF-8，以及超出 `numeric` 存储范围的数字
（最多 131072 位整数位、16383 位小数位，指数不超过 `2^30-1`）；Go 的 JSON 解码器不会拒绝后者中的
小幅值，`1e-20000` 会解成 0 并通过门禁；指数超过 `2^30-1` 时，即使尾数为零也会被 PostgreSQL
拒绝。文档按原样保存，所以判定必须在落库前基于原始字节，而不是解析后的图（解码器已把未配对代理项
替换成 U+FFFD、把越界的小数字变成 0）；否则一份这样的文档会让整包结果事务回滚，源行停在
`importing`。失败的 `reason` 用 `unsupported_unicode_escape`、`invalid_utf8` 与
`numeric_out_of_range` 区分三种原因；合法的代理项对（一个高代理项紧跟一个低代理项）与可表示的边界
数字照常导入。

- **逐文档失败，不整包失败**：一份坏 JSON、无法原样保存的文本或数字、或非法图只让该文档失败，同包其他
  文档照常导入。这与 skill 插件的整包校验相反，理由不同：skill 贡献集合是插件身份的一部分，而这里
  每份文档都是彼此独立、且最终属于用户的资产；让一份坏文档连带丢弃其余可用工作流，损失的是用户
  数据。包完整性由 sha256 保证，不再由「每份文档都合法」承担。
- **名称冲突不改名、不覆盖**：按现有 tenant 级唯一规则让该文档失败（`name_conflict`）。自动改名
  会让同一份来源在不同租户导入出不同名字，来源记录无法复核。
- **判定键跨版本，且只认成功**：是否跳过一份文档，只看 `skip_key`
  `(canonical Plugin ID, document_path)` 下是否已有一条 `imported` 记录，与 `package_version` /
  `package_sha256` 无关。`failed` 不占用这个键：上次失败（例如 `name_conflict`）的路径，在导入
  步骤下次被触发（重复安装、重装、新版本、显式重试）时重新尝试，并把记录里的结果与时间刷新为这一
  次。这样 [circus#68](https://github.com/ora-space/circus/issues/68) 结论 6 的「失败按来源身份
  幂等重试」与「同一路径最多导入一次」是同一句话的两面——幂等约束的是成功的结果，不是尝试本身。
- **升级只增不改**：包升级时，判定键已被占用的路径**不再导入**（用户的副本可能已被编辑，覆盖就是
  数据损失），只导入从未成功导入过的路径：新版本新增的路径，以及此前失败、这次可能成功的路径；
  新版本删掉的路径不删副本。包内版本提升导致的文档内容变化，只能由用户在副本上显式操作。
- **不复活用户处置过的副本**：用户删除或归档副本都不改写来源记录，判定键仍被占用，因此重新安装
  插件不会把它再导入一次。删除与归档是用户对这份数据的明确处置，导入不能替用户撤销。要重新拿到
  同一份文档，用户走编辑器的复制/新建，与来源无关；显式的「从来源重新导入」入口留二期（#76），
  且必须是用户对该路径的主动操作，不能由重装或升级自动触发。
- 导入结果必须独立持久化并可单独读取（按 `skip_key`），不能只作为插件安装结果的一个字段。

### D5：四种证据分离

| 证据 | 由谁产生 | 能证明什么 |
| --- | --- | --- |
| 安装成功 | Node 执行结果 → 实例行与 space 聚合（既有） | 包已落到某个运行 Workspace；不证明有任何模板 |
| 接管成功 | 本文的逐文档导入结果 | 该文档已成为本租户的一份 workflow；不证明当下仍可用、也不证明能跑 |
| 发布可用 | 该 workflow 未被归档且有快照，并进入 `@` 目录投影 | 现在可以被引用；不证明运行能启动 |
| 执行可用 | 既有运行状态与启动门禁 | 运行真的起来了 |

四者互相不能推导：`installed` 不宣称有模板；导入成功不宣称可执行；导入失败不改变插件安装状态；
用户事后归档副本会让「发布可用」失效而不回退「接管成功」（后者是历史事实）。因此导入结果与安装
结果必须分别展示，不能合成一个「装好了」的布尔值。

### D6：预置只做管理员显式安装

首期只有一条入口：管理员把 workflow 插件显式安装到某个 space（沿用既有管理员门控与期望状态）。
市场源登记、目录可列出、`marketplace_visible` 都不等于预置；新租户不会被自动改成安装了任何包。
官方示例与默认模板入口留二期。

## 不变量

1. 导入副本是用户数据：可编辑、可改名、可归档；来源记录既不阻止改动，也不暗示所有权。
2. 卸载插件、包升级与来源消失都不删除或改写已导入的副本与既有快照。
3. 下载、校验、解压与解析都在数据库事务外；导入结果落库在事务内。
4. sha256 校验先于任何解析，且下载与解压都有上界。
5. 是否跳过一份文档只由 `(canonical Plugin ID, document_path)` 决定：同一路径最多成功导入一次，
   跨包版本成立；`failed` 不占用这个键，因此失败可以反复重试。
6. 插件身份不出现在 `workflows` 行的身份里；tenant 级名称唯一性不变，冲突逐文档失败。
7. 资产导入不改变插件安装状态；插件安装状态也不宣称任何资产可用。
8. 资产导入的出站请求只发往目录快照记录的定位符：HTTPS 形态在非企业模式只连公网，企业模式受共享部署白名单/地址策略约束且不跟随重定向，S3
   object key 形态走 Cloud 自己的对象存储；超时与字节上界先于解析。

## 为什么不是这些替代方案

| 方案 | 否决理由 |
| --- | --- |
| 模式 B：引用包内文档，不复制 | 包升级或卸载会改变用户资产，用户编辑无处安放；与 desktop 已实现的「用户数据」语义分叉；`workflows` 行会变成由插件身份拥有，破坏不变量 6 |
| 每个 Workspace 的安装都触发导入 | 同一租户会重复建文档（#73 的范围已排除）；没有 live workspace 的空空间拿不到模板 |
| Cloud 把包写入 Workspace，或让 Node 解析文档后上报 | 与「安装落盘只由 Node 完成」「Node 不自行解析市场」冲突；同一份包在两处解析，格式必然分叉；空空间没有 Node |
| 按包升级覆盖副本 | 破坏用户编辑，也与「已发布快照不可变」矛盾 |
| 名称冲突时自动改名 | 同一来源在不同租户导入出不同名字，来源记录无法复核；用户也看不到冲突这件事 |
| 直接 GET 目录里的 `url`，不做地址判定 | 目录条目可由第三方仓库提供，`url` 就成了 SSRF 入口；把判定放在连接时，代价只是一次 DNS 解析与一条既有出网策略的复用（D3） |
| 跟随下载重定向 | 重定向目标不会重新经过地址判定；Cloud 既有的出站客户端都拒绝重定向，这里不破例（D3） |
| 一份文档非法就整包不导入 | 用用户数据为包的完整性买单；sha256 已经承担了包完整性 |
| 导入结果只写在插件安装结果里 | 无 live workspace 的空间没有安装结果，模板却已经存在；四证据合成一个布尔值就再也分不开 |

## 本决策未解决的问题

- pack（extension-pack）是否也能携带 workflow 文档、以及成员插件如何被选中：属于 pack 自己的
  决策，本文不预先规定。
- 官方示例包的制包与默认预置入口（#76）属于二期。
- 「用户副本与来源的差异」如何对账展示（用户改过哪一份、来源已升级到哪个版本）：需要 README 与
  界面决策，超出本次契约。

## 落地

- 阶段 1（#73）：已实现——来源记录与逐文档结果落库、资产导入步骤（下载/校验/解压/逐文档导入）、
  幂等与升级只增不改。触发记录只由生效后的安装或重装写入；既有安装不做回填。证据是真实 registry
  `.orax` 加 HTTP 与 PostgreSQL 的集成测试，不以 Desktop 本地 SQLite import 或 `Installed(version)`
  单独验收。
- 阶段 2（#76）：预置入口与官方示例包，以及用户显式的「从来源重新导入」入口。
- 两个阶段都不改动插件安装与移除的既有路径；插件实例的状态仍只由 Node 执行结果的接管写入。
- 核心用例见
  [workflow 插件资产核心用例](../../../test-cases/cloud/plugin-marketplace/workflow-plugin-assets.md)。
