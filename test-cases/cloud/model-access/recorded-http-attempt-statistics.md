# 模型请求统计：发送意图、不可变收尾与未知用量

决策依据：[已登记 HTTP 尝试统计](../../../decisions/cloud/model-access/20261016-recorded-http-attempt-statistics.md)（`approved`）。
直接代码证据固定在 [Cloud PR #69](https://github.com/ora-space/cloud/pull/69) 的
[`feae6e39f6cd9647784e6ff923778e65fa8f7f92`](https://github.com/ora-space/cloud/tree/feae6e39f6cd9647784e6ff923778e65fa8f7f92)。
WorkflowRun 登记与 `0044` 的代码证据来自 [Cloud PR #93](https://github.com/ora-space/cloud/pull/93)，
固定在合并提交 [`090bded8505cbf39fc3ec866eaae826cd6436c95`](https://github.com/ora-space/cloud/tree/090bded8505cbf39fc3ec866eaae826cd6436c95)。
`Covered` 只证明该提交所列测试边界内的义务，不表示已合并或真实部署已验收。
Gateway 故障注入使用替身持久化；Core 使用真实 PostgreSQL；贯通测试使用真实 PostgreSQL 和受控 TLS
上游，不能代替真实供应商、Node 或浏览器验收。

## Model dispatch must follow a persisted and authoritative attempt identity

- 风险：未登记就发送模型请求，或客户端伪造统计所属租户、Run、模型。
- 前置状态：有效 grant、冻结 binding/execution；也可能撤销授权或无法写入台账。
- 触发：网关准备发送一次上游 HTTP 请求。
- 必须成立：当前数据库关系决定归属；先提交意图，再发送；登记失败返回 `503 model_metrics_unavailable`，上游调用数为零；每次客户端重发是独立尝试。
- 禁止结果：凭客户端字段选择归属、失效授权创建意图、登记错误导致无台账调用、跨 HTTP 持有 SQL 事务。
- 决策依据：D1、不变量 1、2。
- 验证义务与证据：

| 义务 | 代表性直接证据 | 状态 |
| --- | --- | --- |
| 归属和模型来自持久绑定，重复相同意图幂等，撤销后拒绝新意图 | [Core 台账测试][core]：`TestModelAttemptsPersistFrozenScopeAndImmutableCompletion` | Covered |
| WorkflowRun 绑定按 `workflow_run_id` 登记且 `run_id` 为空 | [WorkflowRun 台账测试][workflow-attempt]：`TestWorkflowModelAttemptRegistersUnderWorkflowRun`，真实 PostgreSQL 驱动冻结 grant/binding/execution 与最终收尾 | Covered |
| 登记失败返回 503 且不发送、不泄漏诊断 | [Gateway 尝试测试][attempt]：`TestModelAttemptsPersistenceFencesAndIdempotentFinalization` 的失败登记分支；注入 Store 失败并计数真实 TLS 请求 | Covered |
| 503 的精确公开错误码 | 上述测试断言 HTTP 状态和调用数，未直接断言 `model_metrics_unavailable` 字段 | Missing |
| 真实提交先于 HTTP，发送期间释放数据库连接，每次请求独立计数 | [贯通测试][gateway-http]：`TestModelGatewayPersistsActualAttemptsThroughStatisticsHTTP`，两协议、单 SQL 连接、受控 TLS 上游 | Covered |
| 实际 PostgreSQL 故障时整个网关的拒绝边界 | Store 故障注入与正常真实数据库贯通分别验证，尚无真实数据库故障注入的组合证据 | Partial |

## Attempt finalization must be bounded immutable and independent of request cancellation

- 风险：完成写入不确定时重复生成，取消导致丢失观察，重试改变结果或重复计量。
- 前置状态：已提交 `started`；上游成功、失败、流中断、请求取消或 grant 撤销；收尾可能连续失败。
- 触发：结束转发并持久化观察，或两个收尾竞争同一 ID。
- 必须成立：取消后使用独立的 3 秒上下文，最多三次、同 ID 同结果写入；相同结果幂等、不同结果冲突；全部失败保留未知 `started`；收尾不得重发上游。
- 禁止结果：复活授权、无界重试、把未完成观察解释为成功/运行中/零用量、泄漏上游诊断正文。
- 决策依据：D2、不变量 2、3、4。
- 验证义务与证据：

| 义务 | 代表性直接证据 | 状态 |
| --- | --- | --- |
| 收尾最多三次，ID/内容不变、上下文有界，失败保留 started 且不重发 | [Gateway 尝试测试][attempt]：`TestModelAttemptsPersistenceFencesAndIdempotentFinalization` | Covered |
| 取消/撤销后仍记录有限分类与部分观察 | [Gateway 尝试测试][attempt]：`TestModelAttemptsCancellationPersistsAfterRequestEnds`；[贯通测试][gateway-http]：实际 HTTP 客户端取消后读回 PostgreSQL 中的 cancelled/partial | Covered |
| grant 撤销后可关闭既有意图，相同重试幂等，并发不同结果只有一个成功 | [Core 台账测试][core]：`TestModelAttemptsPersistFrozenScopeAndImmutableCompletion`、`TestModelAttemptConcurrentFinalizationHasOneWinner` | Covered |
| 错误/中断使用安全分类，非法指标不能关闭意图 | [Gateway 故障测试][failure]：`TestModelAttemptsRecordSafeFailureOutcomes`、`TestModelAttemptsTransportAndDownstreamInterruptions`；[Core 台账测试][core]：`TestModelAttemptRejectsInvalidMetricsWithoutClosingIntent` | Covered |
| 真实提交成功但响应丢失、进程被杀后的跨进程恢复 | 同结果重试与持久 started 有局部证据，没有真实故障窗口的进程级验收；长期未完成行恢复属于后续设计 | Partial |

## Usage observations must preserve provider semantics and missing values

- 风险：累计快照被相加、缓存/推理重复计量、无终止证据却宣称完整、费用或耗时被推算。
- 前置状态：OpenAI/Anthropic JSON 或 SSE，含重复快照、缺失/非法/零计数、无终止事件和空输出帧。
- 触发：观察响应、统计 token 与网关耗时、展示请求趋势。
- 必须成立：快照覆盖而非累加；input/output/provider total/缓存/推理分别保留；完整性需终止和最终输入/输出；未知与零分开；首有效输出排除空帧，不擅改 stream_options；摘要和当前页趋势各有范围。
- 禁止结果：缺失补零、估账单、与 ACP 相加、将网关耗时称为模型计算时间、跨缺失点伪造观测。
- 决策依据：D3、D4、不变量 4。
- 验证义务与证据：

| 义务 | 代表性直接证据 | 状态 |
| --- | --- | --- |
| 两协议快照、不安全数值、零和终止语义 | [用量解析测试][usage]：`TestUsageSnapshotsPreserveCountersWithoutSumming`、`TestOpenRouterFinalUsageRequiresAccountingFrameAndTerminal`、`TestNonStreamedUsageAndSSEFraming` | Covered |
| 首有效输出与时长、协议字节不改写、不覆盖流选项 | [用量解析测试][usage]：`TestSemanticOutputIgnoresEnvelopeAndEmptyChunks`；[Gateway 尝试测试][attempt]：`TestModelAttemptsObserveSuccessfulProtocolsWithoutChangingBytes`、`TestModelAttemptsNeverOverrideStreamOptions` | Covered |
| 真实数据库保存两协议 complete/partial/unavailable 与逐字段覆盖率 | [贯通测试][gateway-http]：`TestModelGatewayPersistsActualAttemptsThroughStatisticsHTTP` | Covered |
| 全 Run 摘要与页趋势分开，真实零与缺失点分开，缓存口径与 total 单列 | [统计组件测试][metrics-ui]：`keeps complete-run summaries separate from the current page and states every coverage boundary`、`plots actual zero and marks unknown fields separately without zero-filled points`、`connects known observations across consecutive missing requests with explicit gap counts`、`reports an excludes-cache summary and separately retained upstream totals` | Covered |
| 真实供应商的协议兼容与观测计数 | 仅有合成协议响应与受控 TLS；不能据此声称真实供应商计量已验收，账单一致性不属本决策承诺 | Missing |

## Statistics reads must be independently authorized and preserve uncollected history

- 风险：没有 Revision 就无法读统计、跨 Run 游标泄漏、旧会话被标为零消耗、撤权后继续展示缓存。
- 前置状态：当前成员与非成员；Run 有完整、部分、started 或完全没有台账；历史归档可用性独立。
- 触发：读取 statistics/model-calls，分页或刷新权限。
- 必须成立：按当前租户/Issue/Agent Run 授权，只读且无需 Revision；来源为 model_gateway/recorded_http_attempts；明细升序分页；每项含覆盖率；无记录为未采集；成功响应禁止共享缓存。
- 禁止结果：身份写入、跨范围游标、未知变零、将摘要与分页视为同一原子快照。
- 决策依据：D4、不变量 1、4。
- 验证义务与证据：

| 义务 | 代表性直接证据 | 状态 |
| --- | --- | --- |
| 已知/总数覆盖率、started 未知、零保留、升序分页和只读事务 | [Core 台账测试][core]：`TestSessionModelStatisticsCoveragePaginationAndReadOnly` | Covered |
| HTTP 范围和查询校验、旧会话未采集、不建身份、删除后拒绝 | [统计 HTTP 测试][statistics-http]：`TestSessionStatisticsHTTPUnknownHistoryAuthorizationAndQueries` | Covered |
| 无归档仍可读统计，撤权/刷新失败清理缓存 | [统计页面测试][metrics-page]：`reads gateway statistics despite unavailable history and pages attempts with a tenant-scoped cursor`、`hides cached metrics and other views when a statistics refresh loses authorization`、`hides old attempt rows after a failed request-page refresh instead of retaining stale success` | Covered |
| 成功 HTTP 响应直接断言 `private, no-store` | 实现已有设置，但所列 HTTP 测试没有对应头部断言 | Missing |
| 真实浏览器多身份刷新与撤权全链路 | 组件和真实 HTTP 授权分别有证据，尚未在一个真实部署中贯通 | Partial |

## Migration must add the attempt ledger without fabricating historical usage

- 风险：改写已应用迁移、升级破坏历史数据、服务启动隐式迁移、旧会话补造用量。
- 前置状态：全新数据库或已应用 0034 的数据库；旧身份/运行应保持原有语义。
- 触发：用迁移身份运行 cloudctl migrate，再启动新服务；重复执行迁移。
- 必须成立：追加 0035，迁移历史/checksum 不改写；新表有范围外键和索引；重复迁移安全；不回填台账；旧会话明确未采集。
- 禁止结果：启动服务执行 DDL、推测旧 token、以全新库测试冒充升级数据保真。
- 决策依据：兼容性、验收与落地；D4。
- 验证义务与证据：

| 义务 | 代表性直接证据 | 状态 |
| --- | --- | --- |
| 全新与 0034 升级、重复迁移、最终台账形状（两组所有者外键/两个所有者索引）及既有身份保留 | [迁移测试][statistics-http-93]：`TestMigration0035ModelAttemptsFreshAndUpgradeWithoutBackfill`，真实 PostgreSQL | Covered |
| 0044 的全新与 0043 升级、重复迁移：`run_id` 可空、`workflow_run_id` 新增、恰有其一，运行与绑定各按所有者引用 | [迁移测试][migration-upgrade]：`TestMigration0044ModelAttemptWorkflowRunAppliesFreshAndUpgrades`，真实 PostgreSQL | Covered |
| 旧会话 HTTP 与页面显示未采集 | [统计 HTTP 测试][statistics-http]：`TestSessionStatisticsHTTPUnknownHistoryAuthorizationAndQueries`；[统计组件测试][metrics-ui]：`shows a historical-session empty state without claiming zero consumption` | Covered |
| 升级时历史 Run/Revision/Thread 全量保真 | 迁移测试保留一条身份，未在 0034 阶段植入完整历史 Run/Revision/Thread；旧会话测试也不是升级夹具 | Partial |
| 真实部署的先迁移后启动及迁移身份隔离 | 迁移单测不证明实际发布顺序或身份配置；需部署验收 | Missing |

[core]: https://github.com/ora-space/cloud/blob/feae6e39f6cd9647784e6ff923778e65fa8f7f92/internal/core/model_attempts_db_test.go
[attempt]: https://github.com/ora-space/cloud/blob/feae6e39f6cd9647784e6ff923778e65fa8f7f92/internal/modelgateway/attempt_test.go
[failure]: https://github.com/ora-space/cloud/blob/feae6e39f6cd9647784e6ff923778e65fa8f7f92/internal/modelgateway/attempt_failure_test.go
[gateway-http]: https://github.com/ora-space/cloud/blob/feae6e39f6cd9647784e6ff923778e65fa8f7f92/integration/model_attempt_gateway_test.go
[usage]: https://github.com/ora-space/cloud/blob/feae6e39f6cd9647784e6ff923778e65fa8f7f92/internal/modelgateway/usage_test.go
[statistics-http]: https://github.com/ora-space/cloud/blob/feae6e39f6cd9647784e6ff923778e65fa8f7f92/integration/session_statistics_test.go
[statistics-http-93]: https://github.com/ora-space/cloud/blob/090bded8505cbf39fc3ec866eaae826cd6436c95/integration/session_statistics_test.go
[migration-upgrade]: https://github.com/ora-space/cloud/blob/090bded8505cbf39fc3ec866eaae826cd6436c95/integration/migration_upgrade_path_test.go
[workflow-attempt]: https://github.com/ora-space/cloud/blob/090bded8505cbf39fc3ec866eaae826cd6436c95/internal/core/model_attempt_workflow_db_test.go
[metrics-ui]: https://github.com/ora-space/cloud/blob/feae6e39f6cd9647784e6ff923778e65fa8f7f92/frontend/src/features/session-metrics/session-metrics.test.tsx
[metrics-page]: https://github.com/ora-space/cloud/blob/feae6e39f6cd9647784e6ff923778e65fa8f7f92/frontend/src/features/sessions/session-metrics-page.test.tsx
