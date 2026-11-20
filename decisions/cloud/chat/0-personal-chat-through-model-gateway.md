---
status: proposed
---

# 独立聊天是租户内个人的私有会话：Cloud 持久化消息并转发回复，模型请求经 model-gateway 代理

Cloud 网页端新增独立的轻量聊天：会话和消息由 Cloud 持久化，只属于创建它的用户；每一轮由 Cloud 向
model-gateway 发起一次上游 HTTP 尝试，把网关返回的流式内容转成 Cloud 自己的 SSE 事件推给浏览器。
真实 Key 不进入 Cloud，聊天的临时授权只在 Cloud 内存中；聊天不创建 IssueRun、Workspace、Node 执行、
Revision 或 Git 引用，也不进入 Space 事件流。网页端"聊天"入口由此从暂不可用引导页变成真实可用的会话。

本 ADR 处于 `proposed`，是领域 `cloud/chat` 的根决策。它覆盖独立聊天页及其公开 API、PostgreSQL
持久化，以及聊天向 model-gateway 取授权和登记台账的边界；不覆盖 Agent Thread、IssueRun、WorkflowRun
与任何 Node 侧执行。聊天与 [Thread 根决策](../thread/0-durable-agent-thread-with-user-turns.md) 是两条
独立契约：Thread 属于一次 Issue 运行，由 Node 会话产生定型记录并随 Revision 归档；聊天没有运行、
没有 Agent 会话，也没有可交付的代码。兼容性上本决策只新增表、列、路由与前端模块，不改既有持久化语义，
也不回填历史数据（聊天的 Mock 数据从未落库）。

## 继承与修改

| 前序决策 | 本文处理 |
| --- | --- |
| [个人模型连接与运行级代理](../model-access/0-personal-model-connections-and-run-scoped-proxy.md)：连接与 Key 归属个人、网关独占解密、运行冻结绑定、代理只转发冻结的模型与协议、出网由网关裁决 | 继承 D1、D2、D5、D6 与不变量 1、2、4、6；修改 D3、D4：冻结发生在会话创建而不是运行创建，授权范围新增"聊天会话"，授权持有者是 Cloud 而不是 Node |
| [执行器槽位绑定](../model-access/20261015-executor-slot-mappings-and-per-slot-run-bindings.md)：`default` 就是个人默认模型、不是映射表；绑定必须属于被授权的对象 | 继承"`default` 即个人默认模型"和"绑定必须属于授权对象"；修改：绑定的所有者从 `run_id` / `workflow_run_id` 二选一扩为三选一，新增聊天会话 |
| [已登记 HTTP 尝试统计](../model-access/20261016-recorded-http-attempt-statistics.md)：发送前登记、登记失败不发送上游、不可变收尾、未知与零可区分 | 继承全部登记与收尾规则；修改 D1：台账归属新增聊天会话，与 WorkflowRun 归属在同一次结构设计里完成；读取首期不开放 |
| [部署方出网与显式解析](../model-access/20261017-deployment-owned-egress-and-explicit-resolution.md) | 继承，不变：聊天不解析模型域名、不选择出网路径，全部由网关按部署策略决定 |
| [Space API 与 Space 级失效事件流](../api-boundary/0-space-api-and-event-stream.md)：REST 是权威、SSE 只做失效通知 | 继承双凭据、严格解码、分页、错误外形与"事件不是权威状态"；修改：聊天回复的 token 级流是**请求作用域的响应流**，不是 Space 事件流，它的增量只是提示，落库消息才是权威 |
| [Thread 根决策](../thread/0-durable-agent-thread-with-user-turns.md)：Issue 运行的会话记录、用户轮次与结束语义 | 不继承：没有运行、没有 Node 记录、没有 Revision，Thread 的表、状态机与事件都不复用 |
| [IssueRun 根决策](../issue-run/0-agent-run-in-disposable-isolated-workspace.md)：一次运行一个隔离 Workspace，交付 Revision 后释放 | 不适用：聊天不创建运行与 Workspace |

## 决策

### D1：会话与消息只属于创建者，只追加不编辑

- 表 `chat_sessions`：`id`、`tenant_id`、`user_id`、`title`、`binding_id`、`system_prompt_version`、
  `created_at`、`updated_at`、`last_message_at`、`deleted_at`。
- 表 `chat_messages`：主键 `(session_id, seq)`，`seq` 从 1 连续递增、由 Cloud 在写入事务中分配；
  另列 `id`、`role`（`user | assistant`）、`content jsonb`（文本块数组）、`status`、`error_code`、
  `model_id`、`created_at`、`finished_at`。
- 消息状态只有 `streaming | completed | failed | cancelled | interrupted`；助手消息在开始时以
  `streaming` 插入，结束时进入其余四种之一。
- 可见性只按 `user_id`：同一租户的其他成员与管理员、Space 成员都不可读；跨租户或不属于自己的
  会话统一 `404 not_found`，不区分"不存在"和"无权访问"。
- 所有聊天请求都要求有效的浏览器会话、活跃的租户成员身份和未停用的账号；`tenant_id` 与 `user_id`
  只来自凭据，不接受请求体或请求头自称。
- 删除是软删除：`DELETE` 写 `deleted_at`，会话从列表与读取中消失、不再接受新消息，并撤销进行中的
  授权；消息与台账行保留，使已登记的尝试仍然可归属。
- 标题由 Cloud 从第一条用户消息派生（去掉多余空白、截断到 80 字），不额外发起一次模型调用，也因此
  不产生额外的一次台账尝试和计费。
- 为什么不是评论或 Thread 表：评论有编辑、删除、通知和 Space 可见性，Thread 属于运行且内容来自 Node
  记录；聊天需要的是"个人私有、只追加、可流式"的最小集合，混用会把两套权限与生命周期绑在一起。
- 为什么标题不让客户端选，也不让模型生成：标题只是列表的可读标签，交给云端的确定性规则就不会引入
  额外的模型费用、失败路径和"标题生成也算一次调用"的统计歧义。

### D2：会话创建时冻结模型绑定，没有可用模型就不建会话

- 创建请求可以带 `connectionId` 与 `modelId`；不带时使用调用者的个人默认模型（`personal_model_defaults`）。
- 创建事务里冻结一行模型绑定：`chat_session_id` 归属、`slot_name = 'default'`、连接名、连接版本、
  凭据引用、协议、`baseUrl`、认证方式和模型快照。冻结列与运行绑定同形，只是所有者不同。
- 准入复用运行入口的同一套判定（个人默认模型可用性、连接是否启用、Key 是否配置、模型是否仍在）：
  缺默认模型、连接停用或删除、Key 未配置、模型已移除分别给出稳定原因码，整个请求回滚，不留下空会话。
- 之后编辑连接、替换或清除 Key、删除连接只影响新会话；已冻结的会话继续使用它冻结的引用。清除 Key、
  停用或删除连接会永久撤销该连接已签发的授权，重新配置只能用于新会话。
- 因此一个会话不能换模型；要换就新建会话。原因：授权、台账归属和上下文都按会话固定，允许中途换模型
  会把两条模型配置拼进同一段上下文，也会让"这次调用记在谁名下"变得不可回答。
- 表名 `run_model_bindings` 保持不变：它现在绑定的是"有模型调用权的对象"，改名要动已经持久化的
  绑定、授权与台账引用，收益只有名字。UI 与文档按"模型绑定"称呼它。

### D3：每轮是一次独立的单轮请求，上下文由 Cloud 从已持久化消息重建

- Chat Completions 与 Messages 都是无状态接口。每轮发送：Cloud 拥有的系统指令，然后是能装进预算的
  最近若干条完整历史消息，最后是本次用户消息。
- 上下文预算按完整消息和字节计：默认最多 24 条消息、96 KiB 文本（配置键
  `chat.context_max_messages`、`chat.context_max_bytes`），系统指令与新消息计入预算。总是丢最旧的完整
  消息，从不截断单条消息，也不拆分工具内容——本决策没有工具内容。
- 只有 `completed` 的助手消息进入后续上下文；`failed`、`cancelled`、`interrupted` 的消息保留可读，
  但不再作为历史喂给模型，避免把半截回答当作已经说完。
- 不做 token 计数：不同协议与不同供应商的分词不同，Cloud 既不引入 tokenizer 依赖，也不假装能精确
  匹配模型窗口。真正超出模型窗口时由上游拒绝，Cloud 归一为稳定错误码（见 D7），保留历史并让用户
  新建会话。
- 系统指令由 Cloud 拥有、随部署发布、带版本号，会话创建时冻结版本；正文不包含租户、用户、Space、
  Issue、仓库或 Git 数据，也不作为消息出现在会话里，因此不占 `seq`、不进聊天记录。
- 为什么不让前端回传整段历史：那会让客户端决定模型看到什么，等于把上下文权威交给浏览器；也让刷新后
  的续聊依赖客户端内存。

### D4：一轮一条 SSE 响应流，同一会话同时至多一轮

- `POST .../chat/sessions/:sid/messages` 返回 `text/event-stream`，因为发送动作带请求体，浏览器用
  `fetch` 读取响应流（`EventSource` 只能发 GET）。
- 事件是 Cloud 自己的契约，不转发供应商的 chunk 形状：

  ```text
  event: start   data: {"sessionId":…,"userMessageId":…,"assistantMessageId":…,"seq":…}
  event: delta   data: {"assistantMessageId":…,"text":"…"}
  event: done    data: {"assistantMessageId":…,"status":"completed|failed|cancelled|interrupted",
                        "finishReason":…?,"error":{…}?}
  ```

  `done` 之后关闭响应。`error.code` 只取白名单错误码，供应商诊断正文不进入事件和聊天记录。
- Cloud 解析网关转发的协议流，只提取文本增量和终止原因，再按上面的形状发出事件；网关仍是唯一接触
  供应商字节的服务，Cloud 不把供应商的帧或错误正文透传给浏览器。这样前端不随协议分叉，切换
  `openai-completions` 与 `anthropic-messages` 不需要第二套界面。
- 落库时机：新轮次在一个事务里插入用户消息（`completed`）和助手消息（`streaming`），分配连续 `seq`；
  生成过程中按"距上次写入满 1 秒或累计满 2 KiB，先到者"把累计文本写回这一行；结束事务写终态、
  `finishReason`、`error_code` 与 `finished_at`。
- 单轮约束由部分唯一索引保证：一个会话至多一条 `streaming` 助手消息。并发发送返回
  `409 chat_turn_in_progress`，不排队。
- 取消：`POST .../messages/:mid/cancel`（`:mid` 是进行中的助手消息 ID）打断当前一轮，终态 `cancelled`；
  客户端直接断开连接同样打断上游，终态记为 `interrupted`。两种情况都保留已落库的部分文本。
- 刷新与重连没有续传：读取消息接口返回已落库的部分文本与状态，前端显示"生成中"并继续轮询或等待用户
  重新发送。崩溃留下的 `streaming` 行由启动时和打开会话时的核对任务按"开始超过 15 分钟"标为
  `interrupted`（`error.code = chat_turn_abandoned`），会话不会永久卡住。
- 为什么不在后台继续生成：生成任务必须有明确的所有者、上限和回收路径；脱离浏览器请求继续跑就要再建
  一套后台执行与回收模型。聊天的价值不足以换来第二套执行模型，以后需要"关页面也继续生成"时另立决策。
- 为什么不排队多条用户消息：与 Thread 不同，聊天没有 Agent 会话承载排序；排队会让"我发的这一条到底
  什么时候回"无法从界面上解释，也让上下文顺序依赖队列状态。
- 为什么不用 WebSocket：一次回复天然是一个请求—响应流，SSE 经现有 Gateway 代理即可工作，不需要新增
  长连接协议、鉴权路径与错误语义。

### D5：Cloud 为聊天签发会话范围的授权，令牌不出 Cloud

- 每一轮开始时，Core 生成 32 字节随机令牌，明文只在进程内存，数据库只存摘要；授权行绑定
  `(tenant_id, chat_session_id, binding_id, model, protocol)`，有效期 15 分钟作为硬上限，轮次结束即撤销，
  不续期。
- Cloud 用该令牌调用 model-gateway 的模型数据监听（`/runtime/openai/v1/chat/completions` 或
  `/runtime/anthropic/v1/messages`），固定 `stream: true`。网关照常从绑定读出目标、认证方式和密文，
  解密后转发；Cloud 不接触明文 Key，也不解析模型域名。
- 授权不走网关的授权监听：那条入口要求 model-access 客户端证书，证书绑定 Workspace 与运行代次，
  聊天没有这些。聊天授权由 Core 直接写入摘要，网关用同一套 `ResolveModelGrant` 解析。
- 网关在转发前和长时间流式转发期间重新核验：会话是否已删除、绑定是否已撤销、连接是否仍启用、
  Key 是否仍在、用户与租户成员是否仍有效。任一项失效都会打断上游并结束这一轮，终态为 `failed`，
  错误码取网关的稳定分类。
- 为什么令牌不交给浏览器：浏览器直连网关要暴露内部地址，令牌也会进入 XSS、扩展和 DevTools 的可见范围；
  留在 Cloud 内，浏览器只使用既有的浏览器会话凭据，网关的模型数据监听仍只被可信服务端调用。
- 为什么不是 Cloud 直连供应商：那必然把 Key 或代理逻辑搬进 Cloud，破坏"Key 只在网关解密、目标与出网
  只在网关裁决"的边界，owner 已否决，本文不采用 issue 正文的方案 A。

### D6：台账给聊天会话第三种归属，与 WorkflowRun 归属同一次结构设计

- `model_request_attempts` 的归属改为 `run_id` / `workflow_run_id` / `chat_session_id` 恰有其一；
  聊天行的 `execution_id` 为空，`binding_id` 与 `grant_id` 按归属分支各自建立外键，沿用
  `UNIQUE(id, owner)` 加复合外键的既有写法，避免"任一列为空就整条外键不校验"的漏判。
- 发送前登记不变：登记失败返回 `503 model_metrics_unavailable`，不发送上游；已登记尝试的收尾不可变，
  取消与中断也保留部分观测；一次聊天轮次就是一次网关 HTTP 尝试，Cloud 不重试，用户重发是新的一轮、
  新的一次尝试。
- 读取首期不开放：只登记，不提供聊天的统计聚合与明细（与 WorkflowRun 只登记的处理一致）。读取的接口
  形状、归属与权限留给后续决策，不在本决策里假装决定。
- 落地顺序：本迁移必须建立在"台账支持 WorkflowRun 归属"之后；两者同时在开发时，在同一次迁移里一次
  做成三归属，不允许连续两次修改同一个归属约束。

### D7：模型不可用时明确引导，不静默降级

- `GET .../chat/availability` 返回 `{"ready": bool, "reason": …, "settingsPath": …}`，`reason` 取既有准入
  原因码：`ready`、`default_model_missing`、`connection_unavailable`、`credential_missing`、
  `model_unavailable`。不新造一套词表。
- 创建会话或发送消息遇到不可用模型时返回 `409 model_not_configured`（`params.reason`）、
  `409 model_credential_required` 或 `409 model_connection_unavailable`；不写消息、不建会话、不签发授权。
- 进行中的一轮遇到授权撤销或凭据不可用，终态 `failed`，`error.code` 为网关稳定码（例如
  `model_credential_unavailable`、`model_grant_revoked`、`model_authentication_failed`、
  `model_rate_limited`）；前端保留已生成的部分文本并提供重试，重试就是新的一轮。
- 不降级到其他用户的连接、别人的默认模型或"部署级模型"——今天不存在部署级模型，也不会退回直连。
- 引导只指向设置页的模型连接入口，不在聊天页复制 Key 管理界面。

### D8：公开 API 只做个人会话的增删读与一轮发送

| 路由 | 语义 |
| --- | --- |
| `GET /api/v1/tenants/:tid/chat/availability` | 当前用户能否开始聊天，以及需要补什么 |
| `GET /api/v1/tenants/:tid/chat/sessions` | 本人会话列表，按 `(lastMessageAt, id)` 倒序，游标分页 |
| `POST /api/v1/tenants/:tid/chat/sessions` | 创建会话并冻结绑定；必须带 `Idempotency-Key` |
| `GET /api/v1/tenants/:tid/chat/sessions/:sid` | 会话元数据与进行中状态 |
| `DELETE /api/v1/tenants/:tid/chat/sessions/:sid` | 软删除并撤销授权；必须带 `Idempotency-Key` |
| `GET /api/v1/tenants/:tid/chat/sessions/:sid/messages` | 按 `seq` 读消息，`after`/`before` 互斥，默认 200、上限 500 |
| `POST /api/v1/tenants/:tid/chat/sessions/:sid/messages` | 发送一条用户消息并返回 SSE；必须带 `Idempotency-Key` |
| `POST /api/v1/tenants/:tid/chat/sessions/:sid/messages/:mid/cancel` | 取消进行中的一轮；必须带 `Idempotency-Key` |

- 双凭据、严格解码、错误外形、分页口径、`404` 不泄露存在性都沿用现有 HTTP 约定；聊天路由进
  `router.Routes()` 白名单并进入 OpenAPI 与生成客户端。
- 请求体只接受 `content: [{type: "text", text}]`，文本 ≤ 64 KiB；未知块类型返回
  `400 invalid_field_type`，超出返回 `400 content_too_large`（路由体上限放宽到 256 KiB 以便区分两种情况）。
- 服务端拥有的字段（`userId`、`tenantId`、`model`、`bindingId`、`seq`、`status`）一律不接受请求体注入。
- 幂等：同 key 同内容重放原响应；发送消息的重放只返回终态 `done`（已结束）或
  `409 chat_turn_in_progress`（仍在生成），不重放增量；同 key 不同内容返回 `409 idempotency_conflict`。

### D9：聊天不创建运行、Workspace、Revision，也不进入 Space 事件流

- 明确不创建：`issue_runs`、`workflow_runs`、`workspaces`、`node_executions`、`revisions`、Git 引用与对象
  存储对象；不产生评论、活动、通知或 Space 成员可见的痕迹。
- 不发布 Space 事件：会话列表与消息只经 REST 读取，生成中状态由发送请求承载，刷新由读取取得。Space
  事件流派生自 Space 成员的失效通知，聊天不是 Space 资源。
- 前端替换现有 `chat` 页（当前是 `UnavailablePage`），不复活 Mock 联系人、示例消息或演示请求；侧边栏
  入口保留。
- 为什么这么严：轻量的定义就是不走容器与交付链路。任何"顺手建一个 Workspace"或"顺手留一个 Revision"
  的写法都会把沙盒配额、生命周期、清理和 Git 授权一并拉进来，聊天也就不再是聊天。

## 不变量

1. 会话与消息只属于创建者的 `user_id`；其他成员、管理员和跨租户访问一律 `404`。
2. 一个会话同一时刻至多一条 `streaming` 助手消息；`seq` 在会话内连续、不重排。
3. 会话冻结的绑定不随后续连接编辑、Key 替换或删除而改变；已撤销的授权不能因为重新启用连接而复活。
4. 每轮先登记台账再发送上游；未成功登记不发送；一轮至多一次上游 HTTP 尝试。
5. 原始 Key 与聊天授权令牌不进入浏览器、聊天记录、Cloud 持久化或日志。
6. 助手消息只追加；部分内容与终态一起保留，Cloud 不重写已结束的消息。
7. 聊天不创建 Workspace、Node 执行、运行或 Revision，也不产生 Git 副作用。
8. 模型请求不在数据库事务或数据库锁内执行。

## 为什么不是这些替代方案

| 方案 | 取舍 |
| --- | --- |
| Cloud 直连模型提供商（issue 正文方案 A） | 少一个服务跳数；但 Key 或代理逻辑必然进入 Cloud，破坏"Key 只在网关解密、目标与出网只在网关裁决"的边界，owner 已否决 |
| 浏览器直连 model-gateway | Cloud 只转发元数据；但令牌必须交给浏览器（XSS、扩展、DevTools 可读），还要把内部网关地址暴露给浏览器 |
| 每次聊天拉起 Node 沙盒，或复用 Thread 契约 | 复用已有的会话记录与前端；但启动成本、沙盒配额、清理与 Revision 交付都与"轻量"相反，也覆盖不了"不建任务、不绑仓库"的场景 |
| 复用 `issue_comments` 存聊天消息 | 表与权限现成；但评论有编辑、删除、通知和 Space 可见性，与"个人私有、只追加"冲突 |
| 新建独立的聊天绑定表与授权表 | 归属更直白；但网关要多一条解析与撤销路径，台账也多一种外键形状，等于复制一套模型访问边界 |
| 同一会话排队多个轮次 | 不丢用户输入；但回复与上下文顺序难以向用户解释，也看不到某一条何时开始 |
| 生成任务脱离请求在后台跑完 | 刷新不丢回答；但需要第二套后台执行、所有者与回收路径；本决策选择随请求取消 |
| 按 token 精确裁剪上下文 | 更贴近模型窗口；但要么引入 tokenizer 依赖并假设各供应商一致，要么假装算得准；本决策按完整消息与字节预算 |
| 首期就开放聊天统计读取 | 复用 Dashboard；但会话分析读取的归属仍是 IssueRun，读取形状与权限需要单独决定，先只登记 |
| 让模型生成会话标题 | 标题更自然；但会多一次模型尝试、多一条台账记录和一条失败路径，收益只是标签 |

## 本决策未解决的问题

- 部署级/集群默认模型：今天不存在。是否由部署方统一提供、以什么凭据边界提供，属于 model-access 的
  后续决策；本决策首期只使用调用者个人的模型配置。
- 保留期限与配额：会话、消息与台账的保留期，以及单用户会话数、消息数与存储上限。
- 统计读取：聊天尝试的汇总与明细接口、归属与权限。
- 图片/文件输入、工具调用、会话分享、跨会话搜索、限流与计费口径。
- 刷新或断线后继续接收未完成的回复（续传或后台生成）。

## 落地

按两个可独立验收的阶段推进：

1. **后端契约**：迁移（`chat_sessions`、`chat_messages`，以及绑定、授权、台账的三归属扩展）、会话与
   消息的 REST 读写、availability、SSE 发送与取消、台账登记与授权撤销。验收覆盖全新库与升级库的迁移、
   越权与跨租户读取、并发轮次、取消与断连的终态、登记失败不发送、以及模型不可用时的引导码。
2. **前端接入**：替换 `chat` 页，接通会话列表、创建（含模型选择）、流式收发、取消、历史回溯与未配置
   引导；刷新后从 REST 恢复已落库内容。

保持不变：Thread、IssueRun、WorkflowRun、Space 事件流、模型连接设置与模型网关的三个监听。开始改变：
网页端"聊天"入口有真实的个人会话，聊天的模型调用进入同一份模型请求台账。
