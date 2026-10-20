# Node 经 Git Server 交付与恢复前序 Revision

依据已批准的 [Node 交付推送与 Git 恢复决策](../../../decisions/node/revision/20261017-push-delivery-ref-and-restore-from-git-server.md)。
下文的 D 编号与不变量编号都指这篇 ADR；“clone 决策”指
[Node 从快照 clone 决策](../../../decisions/node/repository/20261016-clone-from-git-server-snapshot-and-ora-git.md)。

现有的 [Node 的快照与 bundle 交付](snapshot-bundle-delivery.md) 与 [Node 恢复前序 Revision](prior-revision-restore.md) 仍以已实现的
根决策与恢复决策为依据，实现合入并把本 ADR 改为 `implemented` 时再按本文改写或改挂。传输仓库与带证书 Git 的共同义务登记在
[clone 与 `ora-git` 用例](../repository/snapshot-clone-and-ora-git.md)（`Credentialed Git Never Reads The Checkout Configuration`）。
内部线的实现尚未合入 desktop 的 main，下列义务都是 **Missing**。

## Every Delivery Pushes Its Final Commit To The Delivery Ref

- 风险：`unchanged` 不推送，续作要一跳引用；交付 ref 未写入就报告成功；推送了 `refs/ora/revisions/*` 或其他 ref。
- 前置状态：会话结束、插件进程树已停止；三种最终提交：等于基础提交、等于前序最终提交、新提交。
- 触发：交付。
- 必须成立：三种情况都把最终提交推到输入的 `refs/ora/deliveries/<id>`，前两种为 `unchanged`、第三种为 `delivered`；成功结果不带 bundle；推送成功后才上传 JSONL 并发送成功结果。
- 禁止结果：生成 bundle；推送交付 ref 与 `workspaceRef` 以外的 ref 或 tag；`revision_ref` 离开 checkout。
- 验证义务与证据：**Missing**。
- 决策依据：D2，不变量 3、5、7。

## A Frozen Delivery Replays The Same Push After Restart

- 风险：重启后重新快照，推送另一个提交，被只能新建的交付 ref 拒绝；重放把 up to date 判为失败。
- 前置状态：冻结计划已写入后杀掉 Node；另一次在推送成功、JSONL 上传前杀掉。
- 触发：Node 重启后继续同一执行。
- 必须成立：沿用冻结的最终提交、声明与 JSONL；远端已指向该提交时按成功处理；冻结文件在终态持久化后删除。
- 禁止结果：重新快照；重放以 `push_rejected` 失败。
- 验证义务与证据：**Missing**。
- 决策依据：D3，不变量 4。

## Push Failures Are Classified And Retried Within Bounds

- 风险：`403` 或 ref 被拒被反复重试，延长沙盒占用；不可达被判永久失败；单次推送无时限，越过 Cloud 的放弃窗口。
- 前置状态：可注入应答的 Git Server 替身。
- 触发：`401`、`403`、其他 `4xx`；ref 已存在或 `pre-receive` 拒绝；`5xx`、`429`、连接失败、低速中止、单次时限到期、无法识别的错误。
- 必须成立：前两类不重试，以 `push_rejected` 失败；其余共 3 次尝试、间隔 5 秒与 30 秒，耗尽以 `push_failed` 失败；每次尝试 30 分钟时限，并在一分钟低于 1 KiB/s 时中止。
- 禁止结果：`403` 被重试；尝试次数超过 3。
- 验证义务与证据：**Missing**。
- 决策依据：D4，不变量 7。

## A Failed Workspace Branch Fast Forward Never Fails The Delivery

- 风险：Agent 改写了已推送的历史，快进被拒拖垮整个交付；账本未更新导致下一次推送误报冲突。
- 前置状态：交付 ref 已推送成功；Agent 推送过的分支不是最终提交的祖先；另一次远端已等于最终提交而账本未更新。
- 触发：交付快进 `workspaceRef`；快进时 Git Server 返回 `403` 或重试耗尽。
- 必须成立：被拒、旧值不符、`403`、重试耗尽都只记一条分类日志，交付结果不变、JSONL 照常上传；旧值不符而远端等于最终提交时按成功处理并更新账本；Agent 没推送时新建该分支。
- 禁止结果：交付因快进失败而失败；快进前做本地祖先检查；交付结果增加字段。
- 验证义务与证据：**Missing**。
- 决策依据：D2，不变量 6。

## Restore Moves The Workspace Branch To The Prior Final Commit From The Git Server

- 风险：恢复访问外部仓库；协商使用 Agent 可写的 ref；恢复后本地分支名或 HEAD 不对；外部历史被改写时没有提示，或不该提示时提示。
- 前置状态：续作的会话，`priorRevision` 指向 Git Server 上的交付 ref；前序 `baseCommit` 分别是、不是本 checkout 快照提交的祖先。
- 触发：会话开始、插件启动之前；恢复期间会话被结束或取消。
- 必须成立：只从输入的 Git Server 地址拉取交付 ref，取到的提交等于 `finalCommit`；协商只以 clone 记录的 `base_commit` 为起点；以工作负载身份取入后 `checkout --force -B <Workspace 分支短名>`，工作区干净；`baseCommit` 不是快照提交的祖先时才在首条提问末尾追加改写提示；会话结束或取消时在下一条 Git 命令前停止，插件不启动。
- 禁止结果：访问外部仓库；`refs/remotes/ora/**`、仓库配置或 clone 基线改变；恢复失败后插件启动。
- 验证义务与证据：**Missing**。
- 决策依据：D5，不变量 8、9、10。

## Restore Failures Separate Permanent Absence From Retryable Errors

- 风险：一次部署配置错误让 Cloud 把前序永久标为不可续接；缺陷导致的不一致让 Revision 永久放弃；Git Server 短暂不可用被判永久。
- 前置状态：可注入应答的 Git Server 替身；5 分钟恢复时限。
- 触发：交付 ref 不存在；`403`；`401`；`401`、`429` 以外的其他 `4xx`；取到的提交不等于 `finalCommit`；`5xx`、`429`、网络、时限到期；本地取入失败。
- 必须成立：交付 ref 不存在、`403` 与 `401`、`429` 以外的其他 `4xx` 为 `prior_revision_base_unavailable`；`401` 为 `prior_revision_unavailable` 且不重试；其余为 `prior_revision_unavailable`；可重试的失败在时限内最多 3 次；都以 `AGENT_FAILED` 加 `detail` 结束会话。
- 禁止结果：`401` 被归为 `base_unavailable`；提交不等被归为 `base_unavailable`；下一次退避越过时限仍等待。
- 验证义务与证据：**Missing**。
- 决策依据：D5、D6。

## A Cloud Mode Node Without Git Access Refuses To Start

- 风险：缺少 git-access 证书的 Node 照常启动，直到一整次会话之后才以交付失败暴露配置错误。
- 前置状态：配置了 `control.target` 的 Node。
- 触发：缺少 `git_access`；证书与私钥不匹配；CA 含私钥块；同时配置 `clone` 与 `git_access`。
- 必须成立：前三种拒绝启动，错误只给有限代码；齐全时声明 clone、`revision_delivery` 与 `revision_restore`。
- 禁止结果：缺少 `git_access` 的 Cloud 模式 Node 声明交付或恢复能力。
- 验证义务与证据：**Missing**。
- 决策依据：D7，不变量 11。

## Node Protocol Carries No Bundle Or Download Grant

- 风险：bundle 字段或下载授权消息残留，新旧 Node 与 Controller 混用时静默走旧路径。
- 前置状态：新 `node-protocol`。
- 触发：解码带 `bundle_key` 的 `DeliverRevision`、带 `bundle` 的 `PriorRevision`；交付输入的前序带 Git 字段；`UploadGrantNeeded` 含 JSONL 以外的键。
- 必须成立：未知字段被拒；交付输入带仓库地址、交付 ref 与 `workspace_ref`；失败码含 `push_rejected`、`push_failed`；`DownloadGrantNeeded` 与 `DownloadGrant` 不属于协议；Controller 按派发输入核对交付终态，不访问 Git Server。
- 禁止结果：新结果出现 `bundle_failed`。
- 验证义务与证据：**Missing**。
- 决策依据：D8。
