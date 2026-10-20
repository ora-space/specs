# 插件市场:选择状态、fan-out 与 effect 证据

新增静态 Agent 声明分发见[配置元数据用例](agent-configuration-metadata.md)，新义务为 `Missing`，
不以既有目录或安装成功证明个人配置与实际包的一致性。

决策依据:[插件市场:cloud 权威选择状态、目录快照与自闭合 effect 载荷](../../../decisions/cloud/plugin-marketplace/0-plugin-marketplace.md)。
以下用例通过真实 PostgreSQL 集成测试(`integration/plugins_*.go`)验证。

后续已实现的[管理员选择与空闲维护](admin-selection-and-idle-maintenance.md)收紧修改权限，并将忙时整体冲突改为持久等待。下列证据按原固定版本、隔离、effect 和聚合义务保留；管理员门控与维护独占已有直接局部证据，整体为 `Partial`。真实执行及故障缺口分别见该用例与[Node 插件执行用例](node-executed-plugin-installs.md)，不将局部通过扩大为完整矩阵。

## Tenant membership grants plugin access and revocation rejects new reads

- 风险：插件仍依赖已删除的空间成员表，或错误复用另一个租户的空间权限。
- 前置状态：用户是租户普通成员，部署已有插件目录，另有独立租户及空间。
- 触发：读取目录和选择列表，使用当前租户路径请求另一个空间，再停用成员后重试读取。
- 必须成立：有效成员可读；错误租户与空间组合返回 404；停用后新请求返回 403。不同空间的选择状态仍隔离，跨空间并发安装可分别完成。
- 禁止结果：要求第二份空间成员关系、跨租户泄漏、停用后继续读取。
- 验证义务与证据：**Covered** — `TestPluginCatalogAPIAndGating`、`TestPluginIsolationSerializationAndRecovery`、`TestPluginConcurrentInstallsAcrossSpaces`；隔离测试中的每个空间现在对应独立租户。
- 决策依据：插件市场的选择隔离，以及一租户一空间 D1。

## Install writes authoritative state and fans out one instance plus one operation per live workspace

- 风险:desired 与执行事实分裂提交,或 fan-out 遗漏 live workspace,聚合永远到不了终态。
- 前置状态:已同步目录的 space + 一个 live(ready)运行时 workspace;已订阅 SSE。
- 触发:`POST /spaces/:sid/plugins {identifier}`。
- 必须成立:同一事务后 `space_plugins` desired=installed、版本固定;每个 live workspace 各有一行
  pending 实例与一条 `install_plugin` operation(workspace 绑定);SSE `space.plugins_updated`
  已广播。
- 禁止结果:实例与 operation 数量不匹配、operation request 缺 pluginId/version、无 SSE。
- 验证义务与证据:**Covered** — `integration TestPluginInstallAPI`(IT-3.3)断言行/实例/operation
  计数与 SSE 事件;`TestPluginMigrationAndCatalogPersistence` 覆盖同库约束。

## Replaying an install is idempotent and a busy project serializes fan-out

- 风险:同 Key 重放重复 fan-out;同 project 并发 install 违反 `one_project_operation`。
- 前置状态:已安装插件(operation 在途)。
- 触发:同 Key 重放 POST;新 Key 重装同版本;operation 未 drain 时安装第二个插件。
- 必须成立:重放返回原响应;重装同版本不重复实例执行，但已有待办仍继续；管理员的新期望持久保存，在途 project 的实例维护等待 operation 槽位和独占条件，不要求再次点击。
- 禁止结果:重复实例/operation、忙时丢弃已接受的期望、跨 project 串行失败(不同 project 可并行)。
- 验证义务与证据:**Partial** — `TestPluginInstallAPI`(IT-3.4)、`TestPluginIsolationSerializationAndRecovery`(IT-5.2)证明前序幂等/项目串行；旧整体 409 断言须按新语义调整，持久等待及自动继续为 `Missing`，见新增核心用例。

## plugin_ensure payload is self-contained and the digest is mandatory

- 风险:载荷缺 release 信息导致 Node 被迫自行同步市场;sha256 跳过校验即破坏供应链不变量。
- 前置状态:目录含 universal 与 targets 两种发布;workspace ready。
- 触发:claim → plan `plugin_ensure`;断言 effect request;模拟 Node 下载错/对工件。
- 必须成立:载荷含 pluginId/version 与 universal{url,sha256} 或完整 targets 三元组;sha256 不匹配
  的下载失败并把实例置 failed + install_error(聚合 failed);修正后按稳定 effect id 重试收敛
  installed。
- 禁止结果:载荷引用外部索引、失败下载静默通过。
- 验证义务与证据:**Covered** — `TestPluginEffectChainAndEvidence`(IT-4.1/4.2)真实 sha256 校验
  (simulator `MapArtifact` 只换传输,不换校验)。

## Effect success evidence is strictly validated

- 风险:Node 上报不完整证据被当作安装成功,聚合与真实状态漂移。
- 前置状态:effect planned。
- 触发:effect_result 携带 `{installed: true}`(缺 version)或错误 version。
- 必须成立:400 `invalid_plugin_evidence`,实例状态不被改写。
- 禁止结果:证据缺失仍收敛。
- 验证义务与证据:**Covered** — `TestPluginEffectChainAndEvidence`(IT-4.6)。

## Aggregation folds instance states deterministically

- 风险:聚合只在内存或只在个别路径重算,恢复后状态不可重算。
- 前置状态:两个 live workspace 的两实例 fan-out。
- 触发:顺序完成两个 install(plan→execute→advance),中间断言聚合;无 live workspace 的 space
  安装后立即断言终态。
- 必须成立:一安装中一 pending → installing;一成功一 pending → installing;全成功 → installed;
  无 live workspace → installed。
- 禁止结果:聚合依赖进程内存或事件顺序。
- 验证义务与证据:**Covered** — `TestPluginAggregationMatrix`(IT-4.5)+
  `TestPluginAggregateRuleMatrix`(纯函数全矩阵单测)。

## Controller takeover reconciles an abandoned plugin effect by stable id

- 风险:planned 后 controller 丢失,新 controller 重复执行或放弃该实例。
- 前置状态:install planned 但未 dispatch;旧 controller release 租约。
- 触发:新 controller acquire(epoch+1)→ Drain。
- 必须成立:按稳定 effect id 与 journal 协调后收敛 installed;实例与聚合状态正确。
- 禁止结果:重复下载/重复实例、实例卡在 installing。
- 验证义务与证据:**Covered** — `TestPluginIsolationSerializationAndRecovery`(IT-5.4)。

## Catalog reads never leave the database and a failed sync keeps the previous snapshot

- 风险:目录 API 直连市场导致宕机传播;同步失败清空目录。
- 前置状态:一次成功同步后,删除/不可达市场源;构造非 fast-forward 更新。
- 触发:GET catalog;再次 Sync。
- 必须成立:GET 返回快照(含 indexed/synced 新鲜度);失败 Sync 后旧目录保留、`sync_error` 记录。
- 禁止结果:同步失败清空目录、目录读触发网络。
- 验证义务与证据:**Covered** — `TestPluginMigrationAndCatalogPersistence`(IT-2.2/2.3);
  单测 `TestSyncNonFastForwardKeepsCatalog`/`TestSyncScanFailureKeepsCatalog`(UT-3.4/3.5)。
