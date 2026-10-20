# 快照 clone 输入与执行授权

依据已批准的 [快照 clone 输入与执行授权决策](../../../decisions/cloud/controller-integration/20261016-snapshot-clone-input-and-execution-ref-grants.md)。
下文的 D 编号与不变量编号都指这篇 ADR；“契约”指
[代码中心契约决策](../../../decisions/cloud/git-server/20261016-project-repository-as-code-hub-and-ref-contract.md)，“Cloud 交付决策”指
[Cloud Revision Git 交付决策](../../../decisions/cloud/revision/20261017-git-native-delivery-refs-and-resume.md)。

现有的 [Revision 恢复契约](revision-restore-contract.md) 与 [Agent 运行执行与上传授权](agent-run-executions-and-thread.md) 仍以已实现的
bundle 契约为依据，实现合入并把本 ADR 改为 `implemented` 时再按本文改写或改挂。内部线的实现尚未合入 cloud 与 desktop 的
main，下列义务都是 **Missing**。

## The Clone Contract Carries Only The Snapshot Input

- 风险：`CloneSpec` 仍能表达外部仓库与分支；`snapshot_ref` 与 `base_commit` 不对应，Node clone 出的不是 Cloud 冻结的那份代码。
- 前置状态：重新生成的 `executions.proto` 与 `internal/controlgrpc`。
- 触发：解码字段 1、2 有值的旧形状；`snapshot_ref` 末段不等于 `base_commit`；`repository_url` 带用户信息或不是 `https`；`workspace_ref` 不在 `refs/heads/workspaces/<id>/` 下或短名为 `HEAD`。
- 必须成立：只有四个字段都非空且形状合法的输入被接受，落库为 `{kind: "clone", repositoryUrl, snapshotRef, baseCommit, workspaceRef}`；`BRANCH_NOT_FOUND` 不再是合法枚举值，`SNAPSHOT_UNAVAILABLE`、`BASE_COMMIT_MISMATCH` 计入三次尝试。
- 禁止结果：旧形状被接受；`buf breaking` 的豁免在下一次修改该文件的 PR 中仍然保留。
- 验证义务与证据：**Missing**。
- 决策依据：D1。

## The Controller Forwards The Frozen Clone Input Verbatim

- 风险：Controller 用 Project 与 Workspace 字段拼输入，或在 Cloud 冻结前替 Cloud 判断等待；外部仓库与凭据引用继续流向 Controller。
- 前置状态：operation 冻结了 `request.clone`；另有一个缺少 `request.clone` 的 clone 步骤（缺陷注入）。
- 触发：Controller 领取并派发；Controller 改写输入中的一个字段后 `RecordDispatch`。
- 必须成立：`OperationSnapshot.clone_input` 由 `request.clone` 投影，Controller 原样放进 `RecordDispatch` 与 Node 命令；`OperationProject` 只有 `id`；逐字不同的输入以 `ABORTED`/`CONFLICT` 拒绝；没有 `request.clone` 的 clone 步骤不派发，以 `blocked`、`external_failure` 停下。
- 禁止结果：Controller 读取 Project 的仓库字段或 `requested_ref`；契约出现等待快照的 defer 原因；Controller 访问 Git Server 或持有任何 Git 凭据。
- 验证义务与证据：**Missing**。
- 决策依据：D2，不变量 1、2、9。

## Only A Clone Result Matching The Base Commit Is Taken Over

- 风险：`CloneReady.commit` 与输入不一致却被接管，交付与续作此后都基于错误的基线提交。
- 前置状态：派发输入的 `base_commit` 为 `C`。
- 触发：Node 报告 `commit = C'`；Controller 重启后经 `RecordQueriedResult` 报告同一结果。
- 必须成立：Controller 不接管、不确认该事件；Cloud 的 `TakeOverNodeEvent` 与 `RecordQueriedResult` 以 `INVALID_ARGUMENT` 拒绝；`advanceClone` 再核对一次。
- 禁止结果：`workspaces.base_commit_id` 被写为 `C'`；结果覆盖已有事实。
- 验证义务与证据：**Missing**。
- 决策依据：D3，不变量 3。

## There Is No Clone Path Outside Workspace Operations

- 风险：`ClaimWork` 的 clone 分支或 Controller 的本机受理入口残留，Node 仍能被要求以部署凭据 clone 外部地址。
- 前置状态：新 Controller 与 Cloud。
- 触发：`ClaimWork`；调用 Controller 的本机受理接口；`RecordDispatch` 指向租户级请求的 `operation_id`。
- 必须成立：`ClaimWork` 只返回带 `target` 的 Agent 会话与交付工作；`CloneIntake` 与 `ControllerHandle` 的 `accept_clone`、`operations`、`operation` 不存在；`RecordDispatch` 拒绝该 `operation_id`。
- 禁止结果：`Watch` 因 clone 请求发出 `WorkAvailable`。
- 验证义务与证据：**Missing**。
- 决策依据：D4，不变量 8。

## Each Sandbox Execution Writes Only What Its Kind Allows

- 风险：clone 执行能写 ref；Agent 会话能写别的 Workspace 的分支或非快进改写；交付执行能写交付 ref 与本 Workspace 分支以外的 ref；结束的执行仍能访问。
- 前置状态：同一 Project 的 Workspace A、B，各有 clone、会话与交付执行；真实 `git` 客户端经 Git Server 访问。
- 触发：clone 执行在 clone 步骤内读与推送，operation 推进后再读；A 的会话推送 `workspaces/<A>/x`、`workspaces/<B>/main`、非快进；会话结束后推送；A 的交付新建自己的交付 ref、快进 `workspaceRef`、推送别的分支；插件执行访问；旧代次证书配上其余条件都成立的会话执行。
- 必须成立：通过判定的执行都能读整个 Project 仓库；clone 执行只读且只在 operation 处于 clone 步骤、执行无结果时有效（Project 可为 `provisioning`）；会话只能新建或快进 `refs/heads/workspaces/<证书中的 workspaceId>/` 下的分支；交付只能新建 `refs/ora/deliveries/<工作项 ID>` 并快进本 Workspace 的 `workspaceRef`；其余一律拒绝。
- 禁止结果：任何沙盒执行写 `refs/ora/upstreams/**`、别的 Workspace 的分支或表外 ref；旧代次证书获授权。
- 验证义务与证据：**Missing**。
- 决策依据：D5、D6，不变量 4–7；契约 D3、D4。

## WorkflowRun Agent Sessions Are Not Granted Git Access

- 风险：只按 IssueRun 写的有效性条件被误套到 WorkflowRun，WorkflowRun 的会话得到未经决策的写权限；或者它的 Workspace 混进目录，让 Agent 误以为可以协作。
- 前置状态：一个执行到 agent 节点的 WorkflowRun，它的 Workspace 与 `agent_session` 执行已登记。
- 触发：该会话执行经 Git Server 读取与推送；同一 Project 的 IssueRun 会话放出时生成 Workspace 目录。
- 必须成立：Git Server 对 WorkflowRun 的会话执行一律 `403`；IssueRun 会话的 Workspace 目录不含 WorkflowRun 的 Workspace。
- 禁止结果：WorkflowRun 的会话读到或写入任何 ref；目录出现没有定义的 `kind`。
- 验证义务与证据：**Missing**。
- 决策依据：D5、D7，“本决策未解决的问题”；Cloud 交付决策 D5。

## Delivery And Session Inputs Describe Only Git Locations

- 风险：bundle 字段或下载授权残留，读契约的人以为 bundle 路径仍在；交付输入缺少 Git 位置，Node 不知道推到哪里。
- 前置状态：重新生成的 `agent_executions.proto` 与 `agent_runs.proto`。
- 触发：放出交付与会话工作项；解码缺少 `git_repository_url`、`delivery_ref`、`workspace_ref` 或 `history_key` 的交付输入；交付输入中的前序带 Git 字段；会话前序只带部分 Git 字段；`GrantRevisionUpload` 的 `checksums` 含 JSONL 以外的键；结果带 `BUNDLE_FAILED`。
- 必须成立：交付输入带仓库地址、`refs/ora/deliveries/<工作项 ID>` 与取自 clone 输入的 `workspace_ref`；会话前序三个 Git 字段同时出现；上传授权只签发 `history_key`，其余以 `invalid_upload_checksum` 拒绝；`GrantRevisionDownload` 与 `DownloadGrant` 不存在；`BUNDLE_FAILED` 与 `BRANCH_NOT_FOUND` 都已 `reserved`。
- 禁止结果：Controller 改写或补全仓库地址与 ref；Controller 访问 Git Server。
- 验证义务与证据：**Missing**。
- 决策依据：D6、D9，不变量 12–14。

## The Workspace Directory Is Fixed At Release And Never Grants Access

- 风险：目录在会话期间被改写或来自 Node 的请求；目录被当作授权依据，Agent 借目录访问它本不能写的分支。
- 前置状态：同一 Project 有超过 100 个 Workspace，其中有已删除的 run Workspace。
- 触发：放出 IssueRun 会话工作项；Controller 改动目录后 `RecordDispatch`；会话期间新建 Workspace。
- 必须成立：目录按创建时间倒序取最多 100 个，含已删除的 run Workspace，随固定输入逐字核对；会话期间新建的 Workspace 不出现在目录中。
- 禁止结果：目录影响 Git Server 的任何判定；目录包含其他 Project 或租户的信息。
- 验证义务与证据：**Missing**。
- 决策依据：D7，不变量 11。

## A Resumed Run Combines The Snapshot Clone With The Prior Delivery Ref

- 风险：续作把基线改成前序的最终提交，交付与续作的判定要分情况；两份输入指向不同仓库，恢复拉取了别处的提交。
- 前置状态：同一 Issue 已有一个有交付 ref 与新提交的 Revision；当前快照为 `S`。
- 触发：再次运行该 Issue；放出会话工作项前改变 Git Server 地址配置。
- 必须成立：新 Workspace 的基线与交付输入的 `baseCommit` 都是 `S`；会话前序的仓库地址等于 clone 输入的 `repositoryUrl`，不同时不带 `priorRevision`；交付输入的前序只有 `revisionId` 与 `finalCommit`。
- 禁止结果：快照输入被替换为前序最终提交；前序的交付 ref 被改写。
- 验证义务与证据：**Missing**。
- 决策依据：D8，不变量 10。
