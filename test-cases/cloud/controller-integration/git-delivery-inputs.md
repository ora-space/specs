# Git 交付输入、授权方法与交付终态的契约

依据[已批准的 controller-integration 后续决策](../../../decisions/cloud/controller-integration/20261012-git-delivery-inputs-without-bundle-grants.md)
（下称“本 ADR”），决策依据与 proto `v1` 破例见
[Cloud Revision 后续决策](../../../decisions/cloud/revision/20261012-git-server-delivery-and-verified-delivery-refs.md)（下称“主 ADR”）。
本 ADR 尚未实现：cloud 的 proto 仍带 `bundle_key`、`PriorRevision.bundle` 与 `GrantRevisionDownload`，desktop 的 Controller
仍中继下载授权，所以契约形状相关义务为 `Missing`。逐字转交、能力检查与终态核对的现有证据来自 desktop
`apps/ora-controller` 的单元与集成测试，以及 cloud `integration` 包；它们用的是旧字段，第 4 步改写时须保留规则断言。
cluster 的 Revision 验收改走 Git 后只作补充证据，不据此标记 `Covered`。

## Controller Relays Git Delivery And Resume Inputs Verbatim

### 风险

Controller 改写、补全或自行注入 Git Server 地址与交付 ref，Node 推送到 Cloud 没有固定的位置；Controller 为此持有
或转交 Git 访问凭据；带前序的会话被派给不能恢复的 Node。

### 前置状态

Cloud 放出带 `git_repository_url`、`delivery_ref` 的交付工作项，或带前序 Revision 的会话工作项。

### 触发

Controller 领取工作项、转换为 Node 消息并 `RecordDispatch`；Controller 重启后从执行记录重建 Node 消息；Node 握手
时有或没有 `revision_restore` 能力。

### 必须成立

- `DeliverRevisionSpec` 的 `git_repository_url = 8`、`delivery_ref = 9` 与会话 `PriorRevision` 的
  `git_repository_url = 4`、`delivery_ref = 5`、`base_commit = 6` 原样进入 Node 消息与 `RecordDispatch`，与 Cloud 固定的
  输入逐字相同；交付输入的 `prior_revision` 只有 `revision_id` 与 `final_commit`。
- `internal/controlgrpc` 解码要求交付输入的 `git_repository_url`、`delivery_ref` 与 `history_key` 非空，不再要求
  `bundle_key`。
- Controller 不需要 Git Server 配置，不签发、不申请、不持有、不转交任何 Git 访问凭据，也不访问 Git Server。
- 带 `prior_revision` 的会话只派给声明了 `revision_restore` 的 Node。

### 禁止结果

Node 消息中的地址或 ref 与 Cloud 固定的输入不同；Controller 配置或日志中出现 Git Server 地址或 git-access 材料；
带前序的会话派给不能恢复的 Node。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 逐字转交与重建 | desktop `apps/ora-controller/src/cloud/deliveries/mapping.rs::records_rebuild_the_delivery_for_the_target_node`：执行记录原样重建为目标 Node 的交付消息，目标不符为冲突、`revision_ref` 越界为校验错误；`::resumed_deliveries_accept_an_unchanged_prior_final_commit`：交付输入的前序只转交 `revision_id` 与 `final_commit`。缺口：`git_repository_url`、`delivery_ref` 与会话前序的新字段尚无测试 | Partial |
| 编解码要求的字段 | cloud `internal/controlgrpc` 仍要求 `bundle_key`；`internal/controlgrpc/session_models_restore_test.go::TestSessionWirePreservesModelBindingAndPriorRevision` 证明的是带 `bundle` 的旧 `PriorRevision` | Missing |
| 不持有 Git 凭据 | Controller 现在没有任何 Git Server 配置，但没有测试断言其配置、日志与转交的消息中不出现 git-access 材料 | Missing |
| 能力检查 | desktop `apps/ora-controller/tests/workspaces/restores.rs::resumed_sessions_wait_for_a_restore_capable_handshake`：Node 不能恢复时带前序的会话留在 Cloud，出现声明 `revision_restore` 的握手后才派发 | Covered |

### 决策依据

本 ADR D1、D2，不变量 1、2；主 ADR D1、D7。

## Agent Run Service Has No Download Grant And Upload Grants Cover Only The Session JSONL

### 风险

下载授权或 bundle 的上传授权残留在契约中，读者与实现以为 bundle 路径仍在；上传授权被用于 JSONL 之外的对象键。

### 前置状态

交付执行已登记；运行处于 `delivering`。

### 触发

Node 经 Controller 请求上传授权，`checksums` 只含 JSONL 键或含其他键；Node 或 Controller 试图申请下载授权。

### 必须成立

- `GrantRevisionUpload` 形状不变，只为交付输入的 `history_key` 签发 PUT；`checksums` 含其他键时以 `INVALID_ARGUMENT`
  （`invalid_upload_checksum`）拒绝。
- 签发规则不变：每个授权只对应一个键与一个方法，只对尚无结果的交付签发，Cloud 不留下任何授权记录。
- `AgentRunService` 没有 `GrantRevisionDownload`，proto 中没有 `DownloadGrant`；Controller 删除下载授权中继，Node 不再
  发送下载授权请求。
- Controller 的上传授权中继只涉及 JSONL，授权不进入日志。

### 禁止结果

为 bundle 键或其他键签发授权；存在任何下载授权方法或消息；授权出现在数据库或日志中。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 签发规则不变 | cloud `integration/agent_run_delivery_test.go::TestGrantRevisionUploadSignsOneGrantPerObjectKey`：每个授权只含一个键与一个方法，拒绝顺序按 D3，授权只在回复中；`integration/agent_run_control_test.go::TestThreadCommandsAndUploadGrantsAreNotLostOrStored`：终态后拒绝，数据库中没有授权 | Covered |
| 只签 JSONL | `TestGrantRevisionUploadSignsOneGrantPerObjectKey` 现在为 bundle 与 JSONL 各签一个 PUT，需改为只签 `history_key`、其他键以 `invalid_upload_checksum` 拒绝 | Missing |
| 没有下载授权 | `GrantRevisionDownload` 仍在 proto 中，Controller 仍中继（desktop `apps/ora-controller/tests/workspaces/restores.rs::download_grants_are_relayed_unlogged_and_refusals_answered`）；删除后需由契约与 Controller 测试共同证明不存在 | Missing |
| Controller 上传授权中继 | desktop `apps/ora-controller/tests/workspaces/deliveries.rs::delivery_waits_for_its_permit_and_relays_checksum_bound_grants_unlogged`：授权按冻结的 checksums 中继，Controller 日志中没有存储主机与签名。缺口：只含 JSONL 键尚无断言 | Partial |

### 决策依据

本 ADR D3，不变量 3；主 ADR D6，不变量 8。

## Delivery Terminal Results Match The Dispatched Input Before Takeover

### 风险

Controller 把与派发输入矛盾的交付终态交给 Cloud；去掉 bundle 后核对被放松；Controller 试图自行确认交付 ref。

### 前置状态

交付执行已派发到 Node。

### 触发

Node 报告 `RevisionDelivered`、`RevisionUnchanged` 或 `RevisionFailed`：`base_commit`、`revision_ref`、`history.key` 与输入
一致或不一致；`unchanged` 停在基础提交、前序最终提交或其他提交；`delivered` 的最终提交等于基础提交或前序最终
提交；失败原因为 `PUSH_REJECTED`、`PUSH_FAILED`。

### 必须成立

- `TakeOverNodeEvent` 之前核对 `base_commit`、`revision_ref` 与 `history.key` 等于派发输入；`unchanged` 的最终提交等于
  基础提交或输入中前序的最终提交，`delivered` 二者都不等。
- 交付 ref 与最终提交的对应只由 Cloud 经 ref 查询核验，Controller 不访问 Git Server。
- 失败结果没有可比内容，照常交给 Cloud；`PUSH_REJECTED`、`PUSH_FAILED` 是合法的失败原因。

### 禁止结果

矛盾的成功结果被交给 Cloud；`delivered` 停在基础提交或前序最终提交仍被接受。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 成功结果的核对 | desktop `apps/ora-controller/src/cloud/deliveries/mapping.rs::resumed_deliveries_accept_an_unchanged_prior_final_commit`：`unchanged` 只接受基础提交或前序最终提交；`::results_project_onto_the_contract`：结果按原样投影到契约，Node 不能自报核验失败。缺口：`delivered` 现在核对的是 bundle 键，未核对最终提交不等于基础提交与前序最终提交 | Partial |
| 新的失败原因 | `PUSH_REJECTED`、`PUSH_FAILED` 尚无契约与测试 | Missing |

### 决策依据

本 ADR D4；主 ADR D3、D6。

## The V1 Breaking Exemption Is Scoped To This Change

### 风险

`buf breaking` 的豁免范围过大或长期保留，此后对 `v1` 的破坏性修改不再被 CI 拦住；删除的字段号或名字以后被赋予
别的含义。

### 前置状态

cloud 的 `proto/buf.yaml` 使用 `FILE` 规则。

### 触发

本次 proto 修改的 PR；下一个 PR；之后任何删除 `v1` 字段、消息或方法的修改。

### 必须成立

- 本次 PR 的 `ignore_only` 只含 `FIELD_NO_DELETE`（`agent_executions.proto`）、`MESSAGE_NO_DELETE`（`agent_executions.proto`、
  `agent_runs.proto`）、`RPC_NO_DELETE`（`agent_runs.proto`）；删除的字段写上 `reserved` 的字段号与名字。
- 下一个 PR 移除豁免，`buf breaking` 以包含本次删除的基线恢复全部 `FILE` 规则，此后 `v1` 只增不改。

### 禁止结果

豁免覆盖其他文件或规则；豁免在下一个 PR 之后仍然存在；删除的字段没有 `reserved`。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 豁免范围与 `reserved` | 本次 proto 修改尚未提出 | Missing |
| 豁免移除后恢复只增不改 | cloud `.github/workflows/backend.yml` 在每个 PR 上以目标分支为基线执行 `buf breaking`（见[Contract Changes Stay Additive And Generated Code Never Drifts](cloud-owned-internal-grpc-contract.md#contract-changes-stay-additive-and-generated-code-never-drifts)），豁免移除后由它继续拦截；移除豁免的 PR 尚未提出 | Missing |

### 决策依据

本 ADR 不变量 4；主 ADR D6。
