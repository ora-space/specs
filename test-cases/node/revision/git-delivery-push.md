# Node 交付推送最终提交到交付 ref

依据[已批准的 node/revision 后续决策](../../../decisions/node/revision/20261012-push-delivery-ref-and-fetch-prior-revision.md)
（下称“本 ADR”）D2–D4。本 ADR 尚未实现：desktop 的 Node 仍生成 bundle 并上传对象存储，所以推送相关义务为
`Missing`；结果判定、冻结计划与 JSONL 上传沿用的规则以 `apps/ora-node/src/revision/tests/` 中经真实 Git 的测试为证据，
这些测试同时断言了将被删除的 bundle 部分，第 4 步改写时须保留其余断言。Git Server 一侧的应答来自 cloud
`internal/gitserver/service_test.go`，只证明服务端行为。cluster 的 `revision:acceptance` 改走 Git 后只作补充证据，不据此
标记 `Covered`。

## Delivery Pushes The Frozen Final Commit Even When Unchanged

### 风险

`unchanged` 不推送，Revision 没有可续作的交付 ref；推送了冻结计划之外的提交，或推送了其他 ref、tag；重启后重新
快照得到不同提交，重放推送失败；成功结果先于推送发出。

### 前置状态

会话已结束且进程已清理，交付执行正在运行，输入给出 `gitRepositoryUrl` 与 `deliveryRef`。

### 触发

最终提交分别等于基础提交、等于输入中前序的最终提交、两者都不等；首次推送之前与之后 Node 重启；同一执行重放推送；
远端 ref 已指向同一提交或其他提交；Controller 断开、Node 关闭或生命周期取消。

### 必须成立

- 三种情况都推送 `<最终提交>:<deliveryRef>`：前两种结果为 `unchanged`，第三种为 `delivered`；结果不带 bundle，
  也不重复声明交付 ref。
- 首次推送或首个 PUT 之前把最终提交、结果声明与 JSONL 的字节、大小、SHA-256 写入冻结计划；之前的重启重新准备，
  之后的重启沿用同一计划，不重新快照。
- 先推送、再上传 JSONL，两者都成功后才发送成功结果；发送前交付 ref 已在 Git Server 上指向冻结的最终提交。
- 重放推送同一提交时远端报告 up to date，按成功处理；远端 ref 已指向其他提交时按 `push_rejected` 处理。
- 只推送输入给出的那一个 ref，不推送其他 ref 或 tag；不生成 bundle。
- Controller 断开、Node 关闭与生命周期取消时终止进行中的 Git 进程，不写失败，重启后继续同一执行。

### 禁止结果

`unchanged` 没有交付 ref；成功结果先于推送；重启后推送不同的提交；推送其他 ref；生成或上传 bundle。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 结果判定 | desktop `apps/ora-node/src/revision/tests/snapshot.rs::unchanged_checkout_freezes_only_the_history`：无改动时为 `unchanged`；`::resumed_delivery_stopping_at_the_prior_final_commit_is_unchanged`：停在前序最终提交时为 `unchanged`；`::changes_become_a_hookless_snapshot_bundled_against_the_base`：有改动时为 `delivered`。三者都断言了将删除的 bundle 部分 | Partial |
| 三种情况都推送 | 尚无实现 | Missing |
| 冻结计划与重启 | `apps/ora-node/src/revision/tests/upload.rs::restart_after_partial_upload_reuses_frozen_bytes`、`::changed_frozen_bytes_fail_without_uploading`：首个 PUT 之后的重启沿用冻结字节，字节变化时失败而不上传。缺口：冻结计划覆盖推送、首次推送之后重启不重新快照尚无测试 | Partial |
| 先推送后上传、推送成功才报告 | 尚无实现 | Missing |
| 重放推送不失败 | cloud `internal/gitserver/service_test.go::TestDeliveryPushCreatesOnlyItsRefAndReplayIsUpToDate`：Git Server 对同一提交的重放推送报告 up to date；`::TestPushRulesRejectEveryOtherUpdate`：已存在的 ref 不能被更新。缺口：Node 把 up to date 当作成功、把被拒的 ref 归为 `push_rejected` 尚无测试 | Partial |
| 只推送一个 ref | 尚无实现 | Missing |

### 决策依据

本 ADR D2、D3，不变量 3、4、5、7；Git Server 根决策不变量 11。

## Push Failures Are Classified Retried And Bounded

### 风险

`403` 被反复重试，沙盒在 Cloud 已经放弃交付之后仍被占用；短暂的 `503` 被判为永久失败；没有总时限的推送越过
交付放弃窗口；无法识别的错误被误判为永久失败。

### 前置状态

交付执行正在推送。

### 触发

Git Server 返回 `401`、`403`、`429`、其他 `4xx`、`5xx`；`pre-receive` 拒绝或 ref 已存在；连接失败、DNS、TLS 握手
失败；传输低于 1 KiB/s 持续一分钟；单次推送超过 30 分钟；无法识别的错误；本地传输仓库无法建立或最终提交不在
checkout 中。

### 必须成立

- `401`、`403` 与 `429` 以外的其他 `4xx`，以及 `--porcelain` 中被拒的 ref：不重试，`failed{push_rejected}`。
- `5xx`、`429`、网络、DNS、TLS、低速中止、单次时限到期与无法识别的错误：共 3 次尝试，间隔 5 秒、30 秒；耗尽后
  `failed{push_failed}`。
- 单次推送 30 分钟总时限，另以 `http.lowSpeedLimit=1024`、`http.lowSpeedTime=60` 中止低速传输；最坏约 91 分钟，
  小于交付放弃窗口。
- 本地准备失败为 `failed{snapshot_failed}` 或 `failed{checkout_unavailable}`。
- JSONL 上传规则不变：授权缺失时等待，单对象 3 次失败为 `failed{upload_failed}`。

### 禁止结果

`403` 或被拒的 ref 被重试；可重试的失败没有上限；推送没有总时限；出现 `bundle_failed`。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 不重试的拒绝 | 尚无实现 | Missing |
| 可重试的失败与上限 | 尚无实现 | Missing |
| 总时限与低速中止 | 尚无实现 | Missing |
| JSONL 上传规则 | desktop `apps/ora-node/src/revision/tests/upload.rs::three_failed_attempts_fail_the_upload`：单对象 3 次失败为 `upload_failed`；`::grant_requests_carry_frozen_checksums_and_headers_travel_verbatim`：授权缺失或过期时以冻结的 checksums 重新请求 | Covered |

### 决策依据

本 ADR D4，不变量 6；IssueRun 根决策 D5 的交付放弃窗口。
