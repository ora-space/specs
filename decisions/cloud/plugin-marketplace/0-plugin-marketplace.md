---
status: implemented
---

# 插件市场:cloud 权威选择状态、目录快照与自闭合 effect 载荷

> 后续决策：[Agent 配置元数据](20261017-agent-configuration-metadata.md) 已批准增加静态声明的
> 目录分发与安装一致性验证，尚待实现。配置读取仍不出网、不启动插件，旧安装证据不覆盖该扩展。

> 后续已实现决策：[管理员选择与空闲维护](20260927-admin-selection-and-idle-maintenance.md)限制空间插件修改并让实例维护持久等待独占资格；[Node 插件执行](20260928-node-executes-plugin-installs.md)拥有实际执行和新实例补装，两者均为 `implemented`。完整联合故障验收仍按其核心用例保留缺口；本文保留原实现状态。

cloud 自己维护插件市场目录快照(每 5 分钟 git fetch + 扫描 `registry/**/orax.toml`,单事务整源
替换 `plugin_catalog_entries`,读取永远不出网);插件选择状态挂在工作区
(`space_plugins`,与技能、智能体同级),执行经 `workspace_plugin_instances` fan-out 到该空间下
每个 live 运行时 workspace,每个实例对应一条 `install_plugin`/`remove_plugin` operation。
`plugin_ensure` effect 载荷由 cloud 从目录快照自包含拼装(url/sha256/targets,与 desktop
`plugin-manager` 的 `DownloadRequest` 字段对应),Node 不自行同步市场、不自己决定装什么;
sha256 校验为必选项。desktop 仓本期零改动,其本地同步与安装运行时保持原样。

本决策已实现:`status: implemented` 表示规则与实现同步。desktop 侧执行器接线属于后续工作
(desktop 仓另立项),本期由 cloud 仓 simulator 扮演 Node 下载/安装。

## 行为变化与不变边界

| 约定 | 本决策 |
| --- | --- |
| 市场同步只在 desktop 本地形态发生 | 修改:cloud 部署形态由 cloud server 自己同步,desktop 代码不动 |
| operation/effect 先持久化 plan 再派发外部副作用 | 继承,不变;install/remove 在同一事务写权威状态+实例行+operation,再经既有通道派发 |
| tenant/owner/project/workspace 复合约束 | 继承,不变;两张新表以复合外键继承完整归属 |
| 迁移线性 + checksum 不可变 | 继承,不变;新增 append-only `0015_plugins.sql` |
| 公开路由严格解码 + space 成员门控 + 幂等键 | 继承,不变 |

## D1:选择状态与执行实例两级分离

- `space_plugins`(主键 `(space_id, source_namespace, identifier)`):desired 是用户意图
  (安装时固定版本,无自动升级),observed 是 fan-out 聚合;乐观 `version` 防并发。
- `workspace_plugin_instances`(主键 `(workspace_id, source_namespace, identifier)`):
  执行事实,effect 回写 observed 后同事务重算聚合。canonical identity 是显式
  `(source_namespace, identifier)` 列对,不塞进 `operations.result` JSON。
- 聚合规则:任一实例 failed → failed;否则任一非终态 → installing/removing;全部终态或没有
  live workspace → installed/removed(无事可做,立即收敛)。

## D2:fan-out 与 one_project_operation 串行

- 安装/移除时 fan-out 到该空间所有 live 运行时 workspace(每实例一个 operation,workspace
  绑定、step=plugin);目标 project 有在途 operation 时整个请求 409
  `operation_in_progress`(与既有 `idleProject` 语义一致),不同 project 可并行。
- 同一 project 多 live workspace 时,单次安装每个 project 只放行一个 operation,其余实例保持
  pending,由下次安装重新 fan-out 收敛(含 pending/failed 实例的补偿路径)。

## D3:git 同步语义与失败策略

- go-git 纯 Go 传输:首次 clone 到临时目录原子 rename;之后 fetch + checkout +
  pull --ff-only(go-git 只支持 fast-forward merge,非 ff 报错,与 desktop gitlancer 同语义)。
- 单飞准入:并发 Sync 直接返回(不排队);失败保留上一版目录快照,只写
  `plugin_sources.sync_error`,下个 tick 重试。
- 扫描镜像 desktop 校验(≤1MiB、resolver=1、8 种 kind（`workbench`、`agent`、`webview`、
  `skill`、`mcp`、`hook`，以及 [Extension Pack 决策](../../desktop/plugin/extension-pack/0-extension-pack.md)
  加入的 `pack` 与 [Workflow 插件类别根决策](../../desktop/plugin/category/workflow/0-workflow-package-format.md)
  加入的 `workflow`）、slug 标识符、文本策略、sha256、
  url/targets 互斥、pack 无 release、canonical 三元组白名单);损坏条目单独跳过,同 canonical
  id 按路径序首个胜出;`marketplace_visible=false` 不进目录;README 截断、logo 只存组合描述。

## D4:effect 契约与证据

- `plugin_ensure` 载荷:`{kind, projectId, workspaceId, pluginId, version,
  universal{url,sha256} | targets[{target,url,sha256}]}`;准入与 node 步骤同门控
  (workspace ready 才派发)。`plugin_delete` 幂等卸载。
- 成功证据:`installed=true` 且 `version` 与计划一致(否则 400 `invalid_plugin_evidence`);
  失败回写实例 failed + `install_error`。

## D5:目录 API 与 SSE

- v1 全量下发(`GET …/plugins/catalog` 带 `syncedAt` 新鲜度),过滤在前端。
- `space.plugins_updated`(选择状态与执行回写)与 `plugins.catalog_updated`(同步提交后
  `PublishAll` 广播到所有订阅者);事件只带失效信息,权威状态永远经 REST 重取。

## v1 边界(明确不做)

- pack 禁装(目录可列出);激活/停止/配置/日志属于 Node 运行时平面。
- 新建 workspace 的自动补装(desired-state 收敛)为后续 phase。
- 真实 Node 执行器接线为后续工作;本期 simulator 保证载荷对 desktop `DownloadRequest` 自闭合。

## 不变量

1. 目录读取永远不出网;一次成功同步整源替换,读者看不到中间态。
2. 外部副作用前权威状态与 operation 已落库;网络/下载绝不在数据库事务内。
3. 实例行复合外键在 SQL 级杜绝跨租户引用;聚合不依赖内存,可随时由实例行重算。
4. sha256 校验必选,即使载荷来自受信目录快照。
5. 幂等:同 Key 重放返回原响应;重装同版本不重复 fan-out;移除/变更过期 version 409、缺失 428。

## 为什么不是这些替代方案

| 方案 | 否决理由 |
| --- | --- |
| Node 自行同步市场解析 release | 需求 3/4 直接推论:cloud 目录是唯一生产者,载荷闭合、可幂等重试;Node 增加网络与缓存复杂度 |
| 选择状态挂运行时 workspace | 用户确认插件与技能、智能体同级,是工作区级资源;执行层 fan-out 覆盖运行时归属 |
| 跟随 latest 自动升级 | 与 desktop 语义一致(升级是显式操作),desired 确定性可对账 |
| 目录服务端分页/过滤 | 市场条目量级小;契约层 query param 支持是额外工程(v1 前端过滤) |
| 同步排队合并 | 5 分钟 tick 下重叠概率极低,desktop 同语义("turn away, don't queue") |
| 目录行直接存 logo 字节 | 索引记录组合而非字节,与 desktop `PluginLogoVariants` 语义一致 |

## 落地与验收

- 证据:`integration/plugins_test.go`(迁移约束、同步入 PG、失败保留旧快照、目录读不出网)、
  `integration/plugins_lifecycle_test.go`(门控、目录形状、安装/移除 API、幂等/冲突/严格解码)、
  `integration/plugins_effect_test.go`(effect 载荷自闭合、sha256 必校验、准入、聚合矩阵、
  隔离/串行/恢复、并发)。
- 核心测试用例见 `test-cases/cloud/plugin-marketplace/`。
