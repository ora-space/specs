---
status: approved
---

# 模型用量按网关 HTTP 尝试独立登记，保留部分观测与未知值

Model Gateway 为每次已授权的上游 HTTP 发送意图建立独立、无正文的持久台账，供会话分析读取。
不能从 Thread 消息数推断调用数，也不能把缺失用量解释为零或账单。本决策已获批准，状态为
`approved`；配套实现仍需审查合并与验收。接受台账与模型请求共用 Cloud 已依赖的 PostgreSQL
可用性边界：登记不可用时返回 503，不发送上游请求。

## 继承与修改

| 前序决策 | 本文处理 |
| --- | --- |
| [个人模型连接与代理](0-personal-model-connections-and-run-scoped-proxy.md) D2–D5 | 继承：冻结绑定、当前授权、凭据隔离、流式转发与不自动重发生成请求。新增发送前登记与结束观察 |
| [执行器槽位绑定](20261015-executor-slot-mappings-and-per-slot-run-bindings.md) | 继承请求必须使用所持 binding/execution 的冻结归属；本决策不实现槽位配置或工作流执行 |
| [Thread 根决策](../thread/0-durable-agent-thread-with-user-turns.md) | 继承原始记录和 Thread 状态机；网关台账是独立来源，不把模型文本或 turnEnded 当 HTTP 次数 |

## D1：先持久化发送意图，每次尝试有独立身份

网关为一次 HTTP 尝试生成 UUID，在发送上游之前由 Core 重新校验 grant、binding 和 execution，
按数据库关系保存 tenant、运行归属、model 与 protocol。运行归属跟随绑定的所有者判别列：IssueRun
绑定写 `run_id`，WorkflowRun 绑定写 `workflow_run_id`，两者恰有其一，与 `run_model_bindings` 的
`0040` 形状一致；两种绑定都只能登记由其冻结 execution 发起的尝试。客户端 header 和 body 不能指定
统计归属。登记失败返回 `503 model_metrics_unavailable`，不发送模型请求；这是相对原代理新增的
可用性依赖。登记后到实际发送前崩溃可能留下 `started`，因此记录数表示已登记尝试，不保证每条都被
供应商接收。客户端重新发起的请求是新尝试；网关不因统计失败重发生成请求。

## D2：结束观察不可变，取消后仍可有界提交

正常、失败、取消、流中断和未观察到终止均保存有限 outcome/error 分类。
完成不重新授予调用权：已经存在的意图在 grant 失效后仍可收尾。提交使用脱离请求取消的
3 秒上下文，最多三次写入同一 ID、同一结果；相同重试幂等，不同完成结果冲突。
所有写入失败时保留 `started`；它表示缺少完成观察，不等于仍在运行，也不等于成功或零用量。

保存范围仅为白名单整数 token、状态、协议、冻结模型和耗时，不含请求/响应正文、工具参数、
认证头、原始 Key、临时 token 或上游诊断。网络 I/O 始终在数据库事务之外。

## D3：供应商字段独立，覆盖率逐指标表达

保存输入、输出、provider total、缓存读/写和推理 token；缺失、负数、非整数或超过浏览器安全整数
范围的值保持未知。累计 SSE 快照替换旧值，不相加；错误或缺少终止事件不能形成完整用量。
完整用量要求独立的终止证据和最终快照中已知的输入/输出，其他情况是 partial/unavailable。

OpenAI input 包含缓存，Anthropic input 不含缓存读写；推理是输出明细，不能与输出重加。
缺 provider total 不猜总量；不推算费用。兼容上游没有 usage 时仍保留时间与状态，
不默认注入 `include_usage`，不覆盖客户端现有 `stream_options`。

计时由网关单调时钟观测，从登记后发送点开始。首有效输出排除 ping、空 role/text 和 usage-only 帧；
非流式 JSON 只有完整收到后才可观察，不能称模型内部计算时长或 token 级 TTFT。

## D4：统计与明细按当前授权独立读取

`statistics` 聚合一个 Agent Run 的全部已登记尝试，`model-calls` 按 `(started_at,id)` 升序分页，
默认 50、上限 100；来源和范围分别为 `model_gateway`、`recorded_http_attempts`。
所有请求验证当前租户成员和 Issue/Run 归属，不依赖 Revision，不 JIT 建号，成功响应 `private, no-store`。
每项数值附已知数、总数和可用性；无记录的旧会话是“未采集”，不是零消耗。

摘要与明细是不同读取快照，活动请求期间可以短暂不一致。前端趋势只描述当前页观察：
真实零保留，缺失标记与数值轴分开，虚线只能连接已知端点并说明中间缺失，不插值或累计。
未来 Agent/ACP 报告仍为另一来源，不与 Gateway 相加；本决策不依赖 Desktop token_usage 扩展。

本决策的读取范围仍是 IssueRun：`statistics` 与 `model-calls` 按 IssueRun 的 `run_id` 读取。
WorkflowRun 的尝试已经登记，但它的统计读取需要单独定义聚合范围、游标与授权，在另行批准前不开放
（决策见 circus#285，由 circus#286 落地）；WorkflowRun 登记行的 `run_id` 为空，不会进入上述
IssueRun 读取。

## 不变量

1. 统计归属由验证过的持久绑定决定，跨租户或跨 Run 游标不可读取。
2. 未成功登记的请求不会发送；统计收尾永远不会重发上游调用。
3. 已完成观察不可变，同一 ID 收尾重试不能增加次数或用量。
4. 未知、部分和测得零保持可区分，正文及凭据不进入台账。

## 为什么不是这些替代方案

| 方案 | 取舍 |
| --- | --- |
| 尽力记录，登记失败仍发送 | 可用性更高，但发生未登记调用时无法表达缺失范围；本方案选择显式失败 |
| 从归档文本估 token/调用次数 | 无法识别内部重试和供应商计量口径，也会把日志事件当请求 |
| 完成时首次插入台账 | 崩溃会让进行中的调用完全消失，无法保留未完成观察 |
| 汇总 ACP 与 Gateway | 两者计量范围可能不同，直接相加会重复计量 |

## 兼容性、验收与落地

追加 `0035_model_attempt_statistics.sql`；完整文件名和 checksum 是迁移身份，不改写旧迁移或历史运行。
部署先用迁移身份执行 `cloudctl migrate`，再启动新 Cloud 与 Model Gateway；服务只校验 schema。
新旧历史无回填，台账以外的运行不补造计量。新表含复合外键、状态/数值/完成形状约束和 Run 时间索引。

后续追加 `0044_model_attempt_workflow_run.sql`：`model_request_attempts` 增加可空的
`workflow_run_id`，`run_id` 改为可空，两者恰有其一；运行与绑定各按所有者引用
`issue_runs`/`workflow_runs` 的 `(id, tenant_id)` 与 `run_model_bindings` 的
`(id, run_id)`/`(id, workflow_run_id)`。它只扩展登记范围，不改变读取范围，也不回填历史行。

验收需覆盖全新与升级及重复迁移、登记前拒绝、WorkflowRun 绑定登记、实际 HTTP 与数据库贯通、
两协议 JSON/SSE、取消后收尾、并发不同完成结果、故障不重发、未知/零/部分覆盖率、越权与旧会话。
长期未完成行的恢复、台账保留期限、增量汇总及大 Run 查询成本留给后续设计；当前全 Run 聚合
仍受 Core 事务串行边界约束。真实供应商账单一致性不由合成协议测试证明。
核心验证义务与直接证据见[已登记 HTTP 尝试统计](../../../test-cases/cloud/model-access/recorded-http-attempt-statistics.md)。
测试覆盖以各义务声明的边界为准，不据合成供应商响应宣布真实供应商或部署验收完成。
