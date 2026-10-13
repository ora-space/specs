---
status: approved
---

# 交付与会话输入改为携带 Git Server 仓库地址与交付 ref，Controller 原样转交；删除 bundle 对象键与下载授权（修改说明）

本文是简短的修改说明：Internal Control 契约随
[Cloud Revision 后续决策](../revision/20261012-git-server-delivery-and-verified-delivery-refs.md)（主 ADR）改为只支持 Git
交付。`DeliverRevisionSpec` 删除 `bundle_key`，增加 `git_repository_url` 与 `delivery_ref`；会话输入的 `PriorRevision`
删除 `bundle`，增加仓库地址、交付 ref 与基础提交；`RevisionDelivered` 删除 `bundle`；`GrantRevisionDownload` 与
`DownloadGrant` 删除；`GrantRevisionUpload` 只签发会话 JSONL 的授权。Controller 原样转交 Cloud 固定的输入，不签发、
不持有、不转交任何 Git 访问凭据，推送与拉取的授权由 Git Server 按 Cloud 当前状态逐请求判定。决策依据、proto
`v1` 破例与不开 `v2` 的理由都在主 ADR，本文只列契约与 Controller 行为的变化。

本 ADR 于 2026-10-12 经 owner 审核通过，状态为 `approved`：规则是第 4 步 cloud 契约与 desktop Controller 修改的开发
依据，实现与验收尚未完成；核心用例见文末“核心测试用例”一节，直接证据逐项登记为 `Covered`、`Partial` 或
`Missing`。本文修改 [Agent 运行执行与上传授权决策](20260928-agent-run-executions-thread-and-upload-grants.md)
与 [Revision 恢复契约决策](20261010-revision-restore-contract.md)。涉及 Cloud 的 proto 与 `internal/controlgrpc`，以及
desktop 的 Controller。兼容性边界：不兼容，与主 ADR 相同，Cloud 与 Controller 同时发布。

## 继承与修改

| 前序约定 | 本文处理 |
| --- | --- |
| 根 D1：proto 是唯一契约源 | 继承，不变 |
| 20260928 不变量 1：`v1` 已发布的消息与方法不变，`buf breaking` 必须通过 | 修改：本次一次性破例，按主 ADR D6 在本次 PR 中以 `ignore_only` 豁免 `FIELD_NO_DELETE`、`MESSAGE_NO_DELETE`、`RPC_NO_DELETE`，下一个 PR 移除；此后恢复只增不改 |
| 20260928 D1：`DeliverRevision{session_execution_id, checkout_execution_id, base_commit, revision_ref, bundle_key, history_key}` | 修改：删除 `bundle_key`（`reserved 5`），增加 `git_repository_url = 8`、`delivery_ref = 9`（D1） |
| 20260928 D1：`RevisionDelivered` 带 bundle 与 history | 修改：删除 `bundle`（`reserved 4`），只带 history |
| 20260928 D1：`RevisionFailed.reason` 的封闭集合 | 修改：增加 `PUSH_REJECTED`、`PUSH_FAILED`；`BUNDLE_FAILED` 枚举值保留，新结果中不再接受 |
| 20260928 D4：`GrantRevisionUpload` 为交付输入中的两个对象键签发 PUT | 修改：只为 `history_key` 签发；`checksums` 只能含该键（D3） |
| 20260928 D6：`revisions` 表见 Revision D4 | 修改：见主 ADR D4 |
| 20261010 D1：`PriorRevision{revision_id, final_commit, bundle}`，`bundle` 只在会话输入中 | 修改：删除 `bundle`（`reserved 3`），增加 `git_repository_url = 4`、`delivery_ref = 5`、`base_commit = 6`，三者只在会话输入中设置（D1） |
| 20261010 D2：`GrantRevisionDownload` 签发前序 bundle 的预签名 GET | 删除（D3） |
| 20261010 D3：恢复失败以 `AGENT_FAILED` 加 `detail` 报告 | 继承；`detail` 的判定见 node/revision 后续决策 D6 |
| 20261010 D4：`RevisionUnchanged` 可以等于前序最终提交 | 继承，不变 |
| 20261010 D5：Controller 中继下载授权；带前序的会话只派给声明 `revision_restore` 的 Node | 修改：删除下载授权中继；能力检查继承（D2、D3） |
| [Controller 会话根决策](../../controller/session/0-controller-relays-agent-sessions.md) D4：交付终态先核对基础提交、ref 与对象键再交给 Cloud | 修改：核对基础提交、`revision_ref` 与 JSONL 键（D4） |
| [Git Server 根决策](../git-server/0-sandbox-scoped-git-server-over-https.md)：Controller 不向 Git Server 登记或撤销授权 | 继承（D2） |

## 决策

### D1：输入形状

```proto
message PriorRevision {
  reserved 3;
  reserved "bundle";
  string revision_id = 1;
  string final_commit = 2;
  // Session spec only: where the Node fetches the prior Revision and what it was based on.
  string git_repository_url = 4;
  string delivery_ref = 5;
  string base_commit = 6;
}

message DeliverRevisionSpec {
  reserved 5;
  reserved "bundle_key";
  // … fields 1-4, 6, 7 unchanged
  // Fixed by Cloud when the work item was released; the Node pushes the final commit here.
  string git_repository_url = 8;
  string delivery_ref = 9;
}
```

- 两份输入都在 Cloud 放出工作项时固定（主 ADR D1、D7），`RecordDispatch` 校验 Controller 转交的输入与之逐字相同，
  规则同 20260928 D1。`internal/controlgrpc` 的解码不再要求 `bundle_key`，改为要求交付输入的
  `git_repository_url`、`delivery_ref` 与 `history_key` 非空。
- 交付输入中的 `prior_revision` 仍只有 `revision_id` 与 `final_commit`。

### D2：Controller 原样转交，不参与 Git 授权

- Controller 不改写、不补全仓库地址与交付 ref，也不需要 Git Server 的任何配置。
- Controller 不签发、不申请、不持有、不转交任何 Git 访问凭据；Node 用自己的 git-access 证书访问 Git Server，授权由
  Git Server 读取 Cloud 当前状态决定。执行结束、运行取消或沙盒换代在 Git Server 的下一个请求生效，不需要 Controller
  通知任何一方。
- 带 `prior_revision` 的会话仍只派给声明了 `revision_restore` 的 Node。

### D3：授权方法

- `GrantRevisionDownload`、`GrantRevisionDownloadRequest`、`GrantRevisionDownloadResponse` 与 `DownloadGrant` 删除；Node 不再
  发送下载授权请求，Controller 删除对应的中继。
- `GrantRevisionUpload` 的形状不变，只签发 `history_key` 的授权；请求的 `checksums` 含其他键时以
  `INVALID_ARGUMENT`（`invalid_upload_checksum`）拒绝。Controller 的上传授权中继（Controller 会话根决策 D4）只涉及
  JSONL，其余规则不变。

### D4：交付终态的核对

Controller 在 `TakeOverNodeEvent` 之前核对：`base_commit`、`revision_ref` 与 `history.key` 等于派发输入；`unchanged` 的
最终提交等于基础提交或输入中前序的最终提交，`delivered` 二者都不等。交付 ref 与最终提交的对应关系只由 Cloud
经 ref 查询核验（主 ADR D3），Controller 不访问 Git Server。

## 不变量

1. Controller 转交的交付与会话输入与 Cloud 固定的逐字相同，包括仓库地址与交付 ref。
2. Controller 不持有、不申请、不转交任何 Git 访问凭据。
3. `AgentRunService` 没有下载授权方法；上传授权只针对会话 JSONL。
4. 本次破例之后，`v1` 恢复只增不改。

## 为什么不是这些替代方案

| 方案 | 取舍 |
| --- | --- |
| Controller 按自己的配置注入 Git Server 地址 | Cloud 少一项配置；但地址脱离固定输入，`RecordDispatch` 无法核对，Controller 还要知道 Project 才能拼出仓库路径 |
| 保留 `GrantRevisionDownload` 为废弃方法 | `buf breaking` 不需要豁免；但留下一个永远拒绝或无人调用的方法，读者以为下载路径仍在 |
| 新开 `v2` 包承载新形状 | 不需要豁免；见主 ADR D6 的否决理由 |

## 本决策未解决的问题

- Controller 遇到能力不足的工作项时的队头阻塞（20261010 未解决项，不变）。

## 核心测试用例

本 ADR 获批后依据最终方案补充的核心用例如下。本 ADR 尚未实现，新增义务的证据大多为 `Missing`，实现时按用例逐项
补充直接测试并更新状态。

- [Git 交付输入、授权方法与交付终态](../../../test-cases/cloud/controller-integration/git-delivery-inputs.md)：仓库地址、
  交付 ref 与会话前序逐字转交，Controller 不持有 Git 凭据；没有下载授权，上传授权只针对 JSONL；交付终态在接管前
  核对；`buf breaking` 豁免只在本次修改中存在。
- [Agent 运行执行、Thread 与上传授权](../../../test-cases/cloud/controller-integration/agent-run-executions-and-thread.md)：
  “成功交付等待 Cloud 验证”改为 ref 查询加 JSONL `HEAD`，“授权不持久化”中只剩 JSONL 上传授权。

## 落地

随主 ADR 落地顺序第 2、3 步：cloud 的 proto、生成物与 `internal/controlgrpc` 编解码；desktop 锁定新契约 commit，
Controller 删除下载授权中继、上传授权只涉及 JSONL、交付终态按 D4 核对，测试覆盖输入逐字转交与新的终态核对。
