# Node 核心测试用例

本目录与 `specs/decisions/node/` 镜像。Node 各能力的根决策进入 `approved` 后，再在对应子目录
登记需要长期维护的跨进程、外部副作用、ownership、重试和恢复用例。

## 已批准的 Git 原生代码链路（内部线）

[Node 从快照 clone 决策](../../decisions/node/repository/20261016-clone-from-git-server-snapshot-and-ora-git.md)与
[Node 交付推送与 Git 恢复决策](../../decisions/node/revision/20261017-push-delivery-ref-and-restore-from-git-server.md)经 owner 批准为
`approved`。实现尚未合入 desktop 的 main，这些义务目前都是 Missing；以外部 clone 与 bundle 交付为依据的既有用例在实现合入时
改写或改挂。

| 领域 | 核心用例 | 重点风险 |
| --- | --- | --- |
| repository | [快照 clone 与 `ora-git`](repository/snapshot-clone-and-ora-git.md) | 访问外部仓库、checkout 留下 remote、带证书的 Git 读到 checkout 配置、拉取次数无界、冒充会话、推到别人的分支 |
| revision | [Git 交付与恢复](revision/git-delivery-push-and-restore.md) | 交付 ref 未写入就成功、`403` 被重试、快进失败拖垮交付、恢复访问外部、`401` 让前序永久不可续、缺证书仍启动 |

## 已批准的插件模型 adapter

[插件模型 adapter 决策](../../decisions/node/agent-runtime/20261017-plugin-owned-model-adapters.md)于 2026-10-17 经 owner
批准为 `approved`。实现尚未交付，这些义务目前都是 Missing；[既有模型代理用例](agent-runtime/model-proxy.md)的
证据保持原范围，不推导 adapter 路线已实现。

| 领域 | 核心用例 | 重点风险 |
| --- | --- | --- |
| agent-runtime | [插件模型 adapter](agent-runtime/plugin-model-adapters.md) | 首轮前准备失败残留授权、上下文泄入账本/日志、CLI 高优先级配置改路由、共享 provider 状态、别名与发现越权、收尾与撤销不收敛 |
