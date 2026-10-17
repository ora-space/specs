# workflow 插件资产导入：逐文档导入、跨版本幂等与出网边界

依据[workflow 插件资产决策](../../../decisions/cloud/plugin-marketplace/20261016-workflow-plugin-assets.md)
（`implemented`）与它的[workflow 接缝决策](../../../decisions/cloud/workflow/20261016-asset-import-and-published-snapshots.md)。

企业 HTTPS 取包的新增规则见[部署方共享出网 ADR D7](../../../decisions/cloud/model-access/20261017-deployment-owned-egress-and-explicit-resolution.md#d7共享网络准入保留模型与资产各自的业务边界)。
两调用方共用白名单/DNS、配置重启生效及企业内网取包义务见[共享出网用例](../model-access/deployment-owned-egress.md#model-forwarding-and-asset-downloads-must-share-one-deployment-policy)，
Cloud 共享准入、受控调用及离线部署更新的直接证据已逐项登记，尚缺的故障组合仍明确保留。下方 cloud#77 的 Covered/Partial 证据只覆盖其所列既有边界，不扩展为企业策略证据。

资产导入已随 [cloud#77](https://github.com/ora-space/cloud/pull/77) 实现：来源记录、取包与
逐文档导入、独立读取接口与前端展示。下表的直接证据以
[`integration/workflow_assets_test.go`](https://github.com/ora-space/cloud/pull/77/files) 与
[`internal/workflowasset`](https://github.com/ora-space/cloud/pull/77/files) 为准；`Partial` 行
列出仍未覆盖的那一段。既有安装不回填：只有生效后的安装或重装写入触发记录（ADR「落地」）。

## A Document Path Is Imported At Most Once Across Package Versions

- 风险：包升级或重装覆盖用户编辑过的副本；反过来，一次失败（例如名称冲突）被当成永久结论，用户
  修好冲突后同一路径再也导不进来；或用户删掉的副本在重装时被自动复活。
- 前置状态：来源记录里同一 `(source_namespace, identifier, document_path)` 可能已有 `imported` 或
  `failed` 结果；用户副本可能已被编辑、改名、归档或删除；包可能已升级到新版本。
- 触发：`v1` 安装成功后升级到 `v2`；重复安装同一版本；对上次 `failed` 的路径再次触发；删除或归档
  副本后重装插件；同一文档在新版本里内容变化。
- 必须成立：跳过判定只看 `(canonical Plugin ID, document_path)` 下是否已有 `imported`，与
  `package_version` / `package_sha256` 无关；`failed` 不占用判定键，下次触发重新尝试并刷新结果与
  时间；升级只导入从未成功导入过的路径；新版本删掉的路径不删副本；删除或归档不改写来源记录，重装
  不复活；结果按判定键独立持久化、可单独读取。
- 禁止结果：按包版本或 sha256 决定跳过、同一路径成功导入两次、失败一次后永久放弃、重装时把用户
  删除或归档的副本再导入一份、升级覆盖已编辑的副本。
- 验证义务与证据：

| 义务 | 所需直接证据 | cloud 侧状态 |
| --- | --- | --- |
| 判定键跨包版本，只有 `imported` 占用 | 真实 HTTP + PostgreSQL：`v1` 导入后升级 `v2`，同一路径的 `workflowId`、快照数与图/快照的 `jsonb` 文本都不变 | Covered |
| 重装使扫描后、领取前的旧待办作废 | 真实 PostgreSQL：同批扫描两个插件，第一个下载时重装第二个；第二个的旧包内容不导入，下一轮按新包导入 | Covered |
| `failed` 不占用判定键，下次触发重试并刷新记录 | 真实 PostgreSQL：名称冲突修复后再次触发，同一路径从 `failed` 转为 `imported`，文档结果与来源尝试的时间都前移 | Covered |
| 删除/归档副本后重装不复活 | 真实 PostgreSQL：软删除或归档副本后重装，`skip_key` 仍被占用，副本数不变 | Covered |
| 结果独立于安装结果持久化 | 真实 PostgreSQL：无 live workspace 的租户也有按 `skip_key` 可读的逐文档结果 | Covered |

- 决策依据：D1（来源记录与 `skip_key`）、D4（判定键跨版本且只认成功、升级只增不改、不复活用户
  处置过的副本），不变量 1、5；[circus#68](https://github.com/ora-space/circus/issues/68) 结论 6。
- 说明：本用例只管「跳过与重试」的判定，不重复「导入产物是已发布快照」那部分义务，后者记在
  [资产导入与已发布快照用例](../workflow/asset-import-and-published-snapshots.md)。
- 直接证据：**Covered** — cloud#77 的 `TestWorkflowAssetImportFromInstalledPlugin` 先导入
  `v1.0.0`，再安装 `v1.1.0`：`release.json` 的 `workflowId`、快照数与 `graph::text` 都不变，
  且升级前后活稿与快照的 `jsonb` 文本逐字节相同；只有新增的 `hotfix.json` 产生副本，同名路径在
  两个版本里内容不同也不覆盖。`TestWorkflowAssetImportDocumentFailuresAreIndependent` 让四份失败
  文档在修复后的包里重新导入，并让 `name_conflict` 在名称释放后转为 `imported`；`ok.json` 保持
  第一次的 `workflowId`。重试前把失败文档结果的 `updated_at` 与来源行的 `attempted_at` 回拨一小时，
  重试后两处都前移，证明失败会被重新尝试并刷新记录。扫描到领取之间的重装由
  `TestWorkflowAssetImportDiscardsAJobSupersededBeforeClaim` 覆盖：第二个插件的任务在第一个下载时
  被升级取代，旧包不导入、来源行保持 `pending`，下一轮按新包导入；下载期间的重装由
  `TestWorkflowAssetImportFencesAReinstallDuringDownload` 覆盖。同一用例把 `Release Kit` 归档后
  重装，live 列表里不再出现它，而 `workflows` 里的那一行仍在。零 workspace 的导入、成员读取、
  跨租户拒绝都在 `TestWorkflowAssetImportFromInstalledPlugin` 中直接断言。

## One Bad Document Fails Alone With A Bounded Reason

- 风险：一份坏 JSON、无法原样保存的文本或非法图让整包文档都不导入，用户丢掉其余可用工作流；
  或失败原因无界，用户无法分辨是 JSON、缺名字、图不可执行还是名称冲突。
- 前置状态：包内有多份 `assets/workflows/*.json`，其中至少一份非法；租户里可能已有同名 workflow；
  导入步骤在数据库事务外解析、事务内落库。
- 触发：导入含 `document_invalid_json`、`document_without_name`、`document_not_executable_graph`
  或 `name_conflict` 的文档；`document_invalid_json` 的文档可能能解析但含 PostgreSQL `jsonb` 拒绝
  的文本（转义 NUL、未配对代理项、非法 UTF-8）或超出 `numeric` 存储范围的数字（如 `1e-20000`、
  指数超过 `2^30-1` 的 `0e2147483648`）；同包其他文档合法。
- 必须成立：每份文档一个结果，坏文档只让自己失败，同包其他文档照常导入并各得一个已发布快照；
  失败码限于四个有界值；`document_invalid_json` 的 `reason` 区分 `unsupported_unicode_escape`、
  `invalid_utf8` 与 `numeric_out_of_range`，合法代理项对与可表示的边界数字照常导入；
  `document_not_executable_graph` 带门禁的问题列表（节点 id 与 kind、槽位、字段名），与编辑器
  发布同源；名称冲突不改名、不覆盖；下载、校验或解压失败是包级结果，不产生任何文档结果，下一次
  触发重试。
- 禁止结果：整包因一份文档失败、失败码之外的自由文本成为唯一原因、自动改名后导入、名称冲突时
  覆盖既有 workflow、包级失败仍留下部分文档结果、无法保存的文本或数字让结果事务回滚并把源行留在
  `importing`。
- 验证义务与证据：

| 义务 | 所需直接证据 | cloud 侧状态 |
| --- | --- | --- |
| 逐文档失败不牵连同包其他文档 | 真实 HTTP + PostgreSQL：混入坏文档的包导入后，其余文档各有 workflow 与快照，坏文档只有一条 `failed` | Covered |
| 失败码有界且可定位 | 真实 HTTP：四类失败分别返回四个错误码；`document_invalid_json` 按 `reason` 区分无法保存的文本与越界数字；图不可执行时问题列表点名节点/槽位/字段 | Covered |
| 无法原样保存的文本与数字在落库前被逐文档拒绝 | 真实 PostgreSQL：含转义 NUL、未配对代理项或非法 UTF-8 的文档，以及含越界数字（如 `1e-20000`、`0e2147483648`）的文档失败；合法代理项对、边界数字（`1e-16383`、`100e-16383`、`0e1073741823`）与正常文档在同一包内照常导入，结果事务提交 | Covered |
| 门禁拒绝与编辑器发布同源 | 同一份不可执行图在导入与发布两条路径得到同一组问题（逐项比较） | Covered |
| 名称冲突不改名、不覆盖 | 真实 PostgreSQL：同名文档失败为 `name_conflict`，既有 workflow 的名称与图不变 | Covered |

- 决策依据：D4（逐文档结果、有界失败码、名称冲突不改名不覆盖）；[workflow 接缝决策](../../../decisions/cloud/workflow/20261016-asset-import-and-published-snapshots.md)
  D1（导入走发布门禁）。
- 直接证据：**Covered** — cloud#77 的 `TestWorkflowAssetImportDocumentFailuresAreIndependent`
  把坏 JSON、缺名称、不可执行图与名称冲突放在同一个包里：包结果仍是 `completed`，`ok.json` 正常
  导入并发布快照，四份失败各得一个码，`graph.json` 带 `failureDetail.problems`，失败文档没有留下
  workflow 或快照。同一用例把 `graph.json` 的图再走一次编辑器发布，逐项比较导入的
  `failureDetail.problems` 与发布拒绝的 `params.problems`，证明两条路径同源；门禁本身的节点、
  字段、槽位矩阵仍由
  [`integration/workflow_admission_test.go`](https://github.com/ora-space/cloud/blob/df507f0/integration/workflow_admission_test.go)
  与 [`internal/core/workflow_graph_test.go`](https://github.com/ora-space/cloud/blob/df507f0/internal/core/workflow_graph_test.go)
  覆盖。`TestWorkflowAssetImportRejectsTextPostgresCannotStore` 把转义 NUL、未配对代理项与非法
  UTF-8 放进同一个包：三份文档各得 `document_invalid_json` 及各自的 `reason`，正常文档与合法
  代理项对照常导入，包结果为 `completed`，源行没有停在 `importing`。
  `TestWorkflowAssetImportRejectsNumbersPostgresCannotStore` 把 `1e-20000`、`0e2147483648` 与
  三份边界数字（`1e-16383`、`100e-16383`、`0e1073741823`）放在同一个包：两份越界文档各得
  `numeric_out_of_range`，三份边界文档与正常文档照常导入并各有 workflow 与快照，包结果为
  `completed`，源行没有停在 `importing`；去掉落库前检查时同一用例以 SQLSTATE 22003 失败，直接
  证明缺了它就会整包回滚。
  名称冲突用例预置了同名
  workflow，失败后该行的名称与图不变；名称释放后同一路径转为 `imported`，证明没有自动改名或覆盖。

## The Asset Fetch Stays Inside The Declared Outbound Boundary

- 风险：第三方仓库在目录快照里给出的 `url` 成为 SSRF 入口，打到云内私有地址；跟随重定向绕过刚做过
  的地址判定；或无界下载把远端字节变成内存与带宽风险。
- 前置状态：目录快照已同步，`universal.url` 通过 `validateLocator`，形态为无凭据 HTTPS URL（可带
  签名 query、无 fragment）或 S3 object key；导入步骤在数据库事务外取包。
- 触发：DNS 应答落在私有、环回、链路本地、未指定或文档/保留网段；服务端返回 3xx；响应体超过
  16 MiB；连接、响应头或整包超时；S3 object key 形态取包。
- 必须成立：只接受 `https`（S3 object key 走 Cloud 对象存储的私有端点，不出公网、不加凭据）；非企业
  模式 host 不设白名单，连接前解析全部 DNS 应答，任一答案落在上述网段即拒绝；企业模式改按共享
  白名单、部署 resolver/静态映射及不可覆盖基础设施禁区判断，新增证据见共享出网用例；不使用环境代理；不跟随重定向，
  任何 3xx 按下载失败处理；连接 10 秒、响应头 30 秒、整包 60 秒；边读边计数，超过 16 MiB 立即中断；
  sha256 校验先于任何解析；失败是包级结果，下一次触发重试。
- 禁止结果：非企业模式连接私网、企业模式访问未许可或绝对禁区地址、经重定向换目标后再下载、读完再判断大小、把下载失败降级成逐文档
  失败、S3 形态走公网或要求新凭据。
- 验证义务与证据：

| 义务 | 所需直接证据 | cloud 侧状态 |
| --- | --- | --- |
| 定位符只允许两种形态，`https` 之外一律拒绝 | 目录同步拒绝 `http`、`s3://`、带 fragment 或带凭据的 URL；S3 object key 被识别 | Covered |
| 非企业公网地址原语判定全部 DNS 应答，拒绝私有/环回/链路本地/保留网段 | 真实 HTTP：指向这些地址的包下载失败，且没有发出连接；不覆盖企业私网例外 | Partial |
| 不跟随重定向 | 真实 HTTP：3xx 被记为包级失败，目标地址没有被请求 | Covered |
| 超时与流式字节上界 | 真实 HTTP：慢响应在 10/30/60 秒边界被取消；超 16 MiB 的响应在读完前中断 | Partial |
| 失败是包级且下一次触发重试 | 真实 PostgreSQL：下载失败不产生文档结果，重试后成功 | Covered |

- 决策依据：D3（触发点与取包、出站请求的约束）、继承表「根决策 D3：目录快照由 Cloud git 同步，
  读取永远不出网 → 修改」、不变量 3、4、8。
- 直接证据：**Covered（1、3、5 行）、Partial（2、4 行）** — cloud#77 的
  [`internal/workflowasset/fetch.go`](https://github.com/ora-space/cloud/pull/77/files) 把
  定位符解析、sha256 先验、重定向拒绝与流式字节上界收在一个入口。`TestFetchRejectsHTTPSFailures`
  断言 `http`、带 fragment、带凭据的 URL 与超界声明都成为有界包级失败；
  `TestFetchUsesThePrivateObjectStoreForObjectKeys` 断言 S3 object key 走 Cloud 自己的对象存储端点；
  `TestFetchRefusesRedirects` 断言 3xx 失败且目标地址一次都没有被请求；`TestFetchWithoutATransportFailsClosed`
  断言没有注入受检传输时不会退回环境代理。包级失败的持久化与重试由
  `TestWorkflowAssetImportPackageFailureRetriesOnNextTrigger` 覆盖：摘要不匹配时导入行 `failed`、
  文档表为空，管理员重装同一版本后重试成功。
- Partial 的原因：DNS 应答判定由注入的 `modelgateway` 受检传输完成，导入侧只证明「接上了它、未接线
  时失败关闭」，没有一条从 `Fetch` 出发、目标解析到私有地址的端到端用例；60 秒整包超时是常量且
  没有慢响应测试，流式上界只有声明大小与读计数两条路径。原语本身的证据仍在
  [`internal/modelgateway/transport.go`](https://github.com/ora-space/cloud/blob/df507f0/internal/modelgateway/transport.go)
  与 [`internal/pluginmarket/manifest.go`](https://github.com/ora-space/cloud/blob/df507f0/internal/pluginmarket/manifest.go)。

## Import Results Are Persisted Independently Of The Install Outcome

- 风险：导入结果只作为插件安装结果的一个字段，没有 live workspace 的空间拿不到模板；或 `installed`
  被当成「有模板」，用户看不出导入失败。
- 前置状态：管理员把 `kind = "workflow"` 插件安装到某个 space；该 space 可能没有 live workspace；
  Node 安装可能成功、失败或在途。
- 触发：没有 live workspace 的 space 安装 workflow 插件；Node 安装失败但 Cloud 导入成功；导入失败
  但 Node 安装成功；卸载插件后读取副本。
- 必须成立：导入由 space 期望状态触发，不依赖 Node 安装结果，也不 fan-out、不建 operation；导入
  结果独立持久化并按 `skip_key` 单独读取；四种证据（安装、接管、发布可用、执行可用）可分别判断，
  不能合成一个布尔值；卸载插件不删除副本、不改写快照。
- 禁止结果：把导入挂在安装 fan-out 上、用 `installed` 宣称有模板、导入失败改写插件安装状态、卸载
  时回收副本。
- 验证义务与证据：

| 义务 | 所需直接证据 | cloud 侧状态 |
| --- | --- | --- |
| 无 live workspace 的 space 也会导入 | 真实 HTTP + PostgreSQL：零实例的 space 安装后出现 workflow 与快照 | Covered |
| 导入结果独立持久化并可单独读取 | 真实 PostgreSQL：导入结果按 `skip_key` 读取，不依赖插件执行结果行 | Covered |
| 安装结局与导入结局互不推导 | 真实 HTTP：Node 安装失败时导入结果仍为 `imported`，反之亦然 | Partial |
| 卸载不回收副本 | 真实 PostgreSQL：卸载后 workflow 与快照仍在，发布可用性不变 | Covered |

- 决策依据：D3（触发不依赖 Node 安装结果）、D5（四种证据分离）、不变量 1、2、7。
- 直接证据：**Covered（1、2、4 行）、Partial（3 行）** — `TestWorkflowAssetImportFromInstalledPlugin`
  在一个没有任何 workspace 的空间里安装插件后导入成功：`workspace_plugin_instances` 与
  `operations` 都是 0，workflow 与快照已经存在。结果从独立的 `GET .../plugins/assets` 读取，
  space member 得到 200，停用成员得到 403，他租户的同路径得到 404；前端
  `plugins-page.test.tsx` 把安装徽章与导入徽章分区断言。卸载插件后 live workflow 仍在、快照数不变。
- Partial 的原因：Cloud 集成测试不运行 Node，也就没有「Node 安装失败但导入成功」或反向的真实
  安装结局。用例证明的是导入不等待也不读取安装结果（零实例、零 operation），并证明界面不把两者
  合成一个状态；Node 侧失败结局的交叉断言仍缺。
