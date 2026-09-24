---
status: proposed
---

# Controller 消费 Watch 信号流：信号驱动领取、断流退回周期领取、`Drain` 暂停领取直到新流建立

云端持久模式的 Controller 在持有 Cloud 租约期间维持一条 `ControlSignalService.Watch` 服务端流，并把它
作为领取的主要触发：流建立时与每个 `WorkAvailable` 到达时立即领取，另在每次租约续期后兜底领取一次；
流不可用时退回按 `claim_interval_ms` 周期领取；收到 `Drain` 或流正常结束时暂停领取，直到新流建立。
每次领取连续登记已接受的工作直到队列为空。Controller 到 Cloud 的连接启用 HTTP/2 保活，使失联的流被
及时发现并退回周期领取。信号仍只是加速器：工作归属继续由 `RecordDispatch` 事务决定，信号丢失只影响
延迟。

本决策处于 `proposed` 状态，是
[Controller 契约方向后续决策](20260922-cloud-owned-contract-and-controller-dial-out.md)的后续决策，修改
其 D2 中"当前 Controller 不消费该流，只做周期领取"的约定，以及
[持久协调双适配器](../persistence/20260922-coordination-store-with-sqlite-and-cloud-adapters.md) D2 中
`claim_interval_ms` 的含义。它只涉及 `ora-controller` Cloud 适配器的长驻任务（`CloudStore::serve`）与
Cloud 连接参数；契约消息、Cloud 的信号语义
（[Cloud 根决策](../../cloud/controller-integration/0-cloud-owned-internal-grpc-contract.md) D5）、本机
SQLite 模式、Node 会话与部署配置字段都不变。Cloud 监听端为保活所需的配套修改见
[Controller 控制监听允许 HTTP/2 保活](../../cloud/controller-integration/20260924-control-listener-keepalive.md)。
多 worker、`NodeAssignment` 与多实例下 `Drain` 的让位语义仍留给 scale 决策。

## 背景

现状下 Controller 每 `claim_interval_ms`（minicloud 生成 500 ms）调用一次 `ClaimWork`：空闲时每秒两次
PostgreSQL 查询，派发延迟的下限就是这个间隔。Cloud 已按根决策 D5 提供 `Watch`：订阅登记后才发送
响应头；接受事务提交后发送带 `operation_id` 的 `WorkAvailable`；每个订阅者缓冲 16 条、满时丢弃；关停
前发送 `Drain` 并正常结束流，排空期间新的 `Watch` 为 `UNAVAILABLE`；打开时只校验一次 epoch。

前序决策把消费 `Watch` 与 `Drain` 的排空语义一起留给 scale 决策。本文的判断是二者可以拆开：单实例下
`Drain` 唯一的发送时机是 Cloud 实例关停（`cmd/server` 收到 SIGTERM 后先 `Drain` 再 `GracefulStop`），
Controller 对它的义务"不再领取、不取消用户任务、不销毁沙盒"已由 Cloud 根决策 D5 规定，只缺"不再领取
到何时为止"；这个结束条件不依赖多 worker 设计。多实例相关部分仍保留在 scale 决策。

## 继承与修改

| 前序约定 | 本文处理 |
|---|---|
| dial-out D1：契约唯一源在 Cloud 仓库，desktop 以 submodule 锁定 commit 生成客户端 | 继承，不变；本文不需要新的契约消息，`Watch` 客户端已在生成物中 |
| dial-out D2：Controller 只拨出，Cloud 部署不打开监听 | 继承，不变 |
| dial-out D2：每 10 秒续期 30 秒的租约，按 `claim_interval_ms` 周期领取 | 修改：续期不变；领取节拍改由流状态决定（D2），`claim_interval_ms` 只在无流时生效 |
| dial-out D2：当前 Controller 不消费 `Watch`，只做周期领取 | 修改：Controller 在持有租约期间维持 `Watch` 并按信号领取（D1–D4） |
| dial-out D2、不变量 3：信号只是加速器，不构成归属，丢失只影响延迟 | 继承，不变 |
| dial-out D3：Cloud 裁决只在 `cloud/fault` 一处映射为接口错误分类 | 继承；打开 `Watch` 的失败按读取归类（D1） |
| dial-out D4：Controller 只以 `x-ora-controller-id` metadata 声明身份，不出示凭据 | 继承，不变；`Watch` 同样携带 |
| persistence D1：`serve` 是适配器与权威的长驻协调，停止时释放租约 | 继承；`serve` 额外持有流，停止时先关闭流再释放租约 |
| persistence D2：`{ kind: cloud, endpoint, claim_interval_ms }` | 字段与校验不变；修改含义：`claim_interval_ms` 是无流时的回退领取间隔 |
| Cloud 根决策 D5：`Drain` 后 Controller 不再领取，不取消用户任务、不销毁沙盒 | 继承；本文 D3 给出单实例下"不再领取"的结束条件 |

## 决策

### D1：持有租约期间维持一条 Watch 流，收到响应头才算建立

- Controller 持有租约且没有在线的流时，在每个续约节拍与领取节拍上以当前 epoch 打开 `Watch`，请求
  携带 `x-ora-controller-id`。
- 收到响应头才算流已建立。Cloud 只在订阅登记之后发送响应头，此后发布的信号都会进入该流；建立后
  立即领取一次，覆盖订阅之前已被接受、其信号因此无人接收的工作。
- 流在持有租约期间一直保留，续期与 epoch 变化不重开：Cloud 只在打开时校验 epoch，即使陈旧持有者
  仍收到信号，它的所有写入也会被 fencing 拒绝。
- 任一调用被裁决为资格失效、Controller 丢弃 epoch 时，关闭在线的流；重新获取租约后按本条重新打开。
- 等待响应头的上限与读取相同（10 秒）。打开失败按读取归类：`FAILED_PRECONDITION` 为资格失效，丢弃
  epoch；其余失败保持无流、继续周期领取。打开 `Watch` 不提交任何东西，超时不属于"未知"。

在两个节拍上都尝试打开：续约节拍保证即使 `claim_interval_ms` 很大，Controller 也在取得租约的同一轮
建立流；领取节拍让断流或 Cloud 重启后的重开延迟与回退领取一致，而不是等到下一次续期。

### D2：领取节拍由流状态决定

| 流状态 | 进入条件 | 领取节拍（`claim_interval_ms`） | 续约节拍（10 秒） | 信号 |
|---|---|---|---|---|
| 无流 | 启动；流以错误结束；在线时丢弃 epoch | 持有租约时先尝试打开流，然后领取一次 | 续约；持有租约时尝试打开流 | — |
| 在线 | 打开成功（任何状态） | 不领取 | 续约后兜底领取一次 | `WorkAvailable` 立即领取；`NodeAssignment` 见 D6 |
| 排空 | 收到 `Drain`；流正常结束 | 持有租约时尝试打开流，只有打开成功才领取 | 续约；持有租约时尝试打开流 | — |

打开成功从任何状态进入"在线"并立即领取（D1）；未持有租约时任何状态都不领取。

- 在线时不做周期领取：Cloud 在每次接受后发信号，周期领取只会重复空查询。兜底领取覆盖 Cloud 因订阅者
  缓冲满而丢弃的信号，以及上一次领取因不可用而未完成的工作；它随续约进行，不引入新的配置或计时器。
- `claim_interval_ms` 的字段与校验不变，含义收窄为"无流时的领取间隔"；minicloud 生成的 500 ms 不需要
  修改。
- 空闲时 Cloud 的领取查询从每个 `claim_interval_ms` 一次降为每 10 秒一次。

| 情形 | 派发延迟上界 |
|---|---|
| 流在线，信号送达 | 一次 `ClaimWork` 与 `RecordDispatch` 往返 |
| 流在线，信号被 Cloud 丢弃 | 10 秒：下一次续约后的兜底领取，并按 D4 一次领完积压 |
| 无流 | `claim_interval_ms` |
| Cloud 关停后重启 | Cloud 恢复后的第一个领取节拍：重开流并立即领取 |

### D3：`Drain` 与正常结束暂停领取，直到新流建立

收到 `Drain`，或流在没有错误的情况下结束，Controller 进入"排空"：

- 不领取，不登记新的派发；
- 继续续约，不释放租约；
- Node 会话、事件接管、查询结果与恢复读取照常进行，不取消用户任务、不销毁沙盒；
- 在领取与续约节拍上尝试打开新流，打开成功即回到"在线"并立即领取。排空中的 Cloud 以 `UNAVAILABLE`
  拒绝新订阅，因此只有新的或恢复的 Cloud 实例能结束排空；排空期间丢弃 epoch 不改变该状态。

流以错误状态结束（传输中断、Cloud 被强制停止、保活超时）时进入"无流"，按周期领取。

正常结束等同 `Drain`：Cloud 只在排空时正常结束 `Watch`（先向订阅者发送 `Drain` 再关闭其通道），而
`Drain` 消息与其他信号一样可能因缓冲满被丢弃；正常结束不会丢失，是比 `Drain` 消息更可靠的排空证据。

不释放租约：`Drain` 表示发送它的 Cloud 实例即将停止，而不是要求 Controller 让位。租约是 Cloud
PostgreSQL 中的事实，Cloud 重启后原租约在过期前仍然有效；此时向正在停止的实例发送 `ReleaseLease`
会被拒绝，即使成功也只会让 Controller 之后以新的 epoch 重新获取。`signals.proto` 中 `Drain` 注释的
"然后释放租约"与此不一致，由 Cloud 后续决策更正。

暂停而不是继续周期领取：排空后发往该实例的领取只会失败并反复告警；暂停遵守 Cloud 根决策 D5 的
"不再领取"，而持续的重开尝试保证 Cloud 恢复后第一时间继续。

### D4：每次领取连续登记，直到队列为空

一次领取触发（流建立、`WorkAvailable`、兜底或周期领取）重复"`ClaimWork` → 按 Node 命令校验 →
`RecordDispatch`"，直到 Cloud 返回无工作或某一步失败；单批最多 16 项，满批时先让关停与续约得到处理，
再立即继续下一批。

- 必须连续：在线时兜底每 10 秒一次，若一次只领一项，Cloud 丢弃 N 个信号造成的积压要 N×10 秒才能排空。
- 失败即停：队头无法登记时 `ClaimWork` 会一直返回同一项；停止并沿用"同一项只告警一次"的日志规则，
  等待下一次触发。
- 满批让出：长队列不应推迟续约直到租约过期。

`ClaimWork` 仍是纯读，`RecordDispatch` 仍决定归属；连续领取不改变任何归属或幂等规则。

### D5：Cloud 连接启用 HTTP/2 保活

Cloud 通道启用 HTTP/2 PING 保活：间隔 30 秒，等待回应 10 秒，连接空闲时也发送。未按时回应时连接被
关闭，在线的流以错误结束（进入"无流"，周期领取并重开），后续调用在新连接上进行。

- `Watch` 是长时间可能没有消息的流；经过 NAT 或中间设备时，连接可能被静默丢弃而两端都收不到关闭。
  没有保活，Controller 会一直认为流在线，派发延迟退化为 10 秒兜底；同一条失效连接还会让每个普通
  调用等满 10 秒超时，直到内核放弃 TCP 重传（Linux 默认约 15 分钟）。
- 选 HTTP/2 PING 而不是 TCP keepalive：PING 由 gRPC 对端回应，经过终结 TCP 的代理时探测的仍是 Cloud
  本身，且检测时间不依赖各主机的内核参数。
- 依赖：grpc-go 服务端默认要求客户端 PING 间隔不短于 5 分钟、无活动流时不允许 PING，累计三次违规后以
  `GOAWAY` 关闭整条连接，其上的普通调用一并中断。按默认策略，空闲的 30 秒保活约两分钟后即被断开。
  因此 Cloud 监听必须先放宽该策略（Cloud 后续决策），启用本条的 Controller 才能部署。

### D6：`NodeAssignment` 只记录

第一版 Cloud 不产生 `NodeAssignment`；Controller 收到时只记录日志，不改变 Node 会话或派发目标。其语义
留给 scale 决策。

## 不变量

1. 信号不产生归属：Controller 只在 `RecordDispatch` 成功后派发，任何信号都不跳过登记。
2. 未持有租约时不打开流、不领取；流只以当前持有的 epoch 打开，丢弃 epoch 时关闭在线的流。
3. 流在线时，领取只由流建立、信号与每次续约后的兜底触发；无流时按 `claim_interval_ms` 周期领取；排空
   时不领取。
4. 排空不释放租约，不影响 Node 会话、事件接管与恢复读取；只有新流建立才结束排空。
5. 每次领取触发都登记到队列为空或失败为止，单批有界。
6. Cloud 连接启用 HTTP/2 保活；失联的流以错误结束并退回周期领取。
7. 本机 SQLite 模式、部署配置字段与 Node 协议不变。

## 为什么不是这些替代方案

| 方案 | 取舍 |
|---|---|
| 维持现状，只做周期领取 | 最简单且已满足正确性；但延迟下限是 `claim_interval_ms`，空闲时持续查询 Cloud，已实现的 `Watch` 无人使用 |
| 接入 `Watch`，周期领取仍按 `claim_interval_ms` | 只降低延迟、不减负载；兜底频率由部署配置决定而与流是否健康无关 |
| 在线时的兜底使用新的配置项或独立计时器 | 可以单独调节；但多一个与 Cloud 缓冲策略耦合的部署参数，而续约节拍已是 Controller 与 Cloud 之间固定的最低频心跳 |
| 收到 `Drain` 释放租约（按现有 proto 注释） | 符合"让位"的直觉；但 `Drain` 来自正在停止的 Cloud 实例，释放调用无法到达，即使到达也只让 epoch 变化 |
| `Drain` 只当作断流，立即回退周期领取 | 状态更少；但违背 Cloud 根决策 D5 的"不再领取"，并在 Cloud 停止期间持续产生失败的领取与告警 |
| 每个信号只领一项 | 与信号一一对应、实现最小；但信号被丢弃后，积压只能按兜底节拍逐项排空 |
| 只用 TCP keepalive，不要求 Cloud 改动 | 没有跨仓库依赖；但探测的是最近的 TCP 对端而不是 gRPC 服务，默认内核参数下失联检测要十几分钟 |
| Cloud 经流推送工作并以推送为归属 | 少一次往返；Cloud 根决策已否决：至多一次投递使断线时的归属不可判定 |

## 本决策未解决的问题

- 多 worker 下 `Drain` 的让位与在途协调交接、`NodeAssignment` 与租约分区，仍属于 scale 决策。
- 流上的应用层心跳：当前依赖 HTTP/2 保活，契约没有心跳消息；需要穿越不转发 PING 的中间层时再议。
- Controller 认证与 TLS 不变，仍留给认证 ADR。

## 落地顺序

1. Cloud：监听放宽 PING 执行策略，更正 `Drain` 注释并重新生成（Cloud 后续决策）。完成条件：测试证明
   按新间隔的 PING 不触发 `GOAWAY`。
2. desktop：`ora-controller-proto` 以测试专用 feature 生成服务端桩，`ora-controller` 的测试据此建立
   内存假 Cloud；Cloud 适配器按 D1–D6 消费 `Watch` 并启用保活；文档中英文同步。完成条件：以假 Cloud 为
   对端的行为测试覆盖信号触发领取、订阅前积压、断流回退与重开、`Drain` 暂停与恢复、打开时资格失效、
   关停时关闭流并释放租约；状态迁移有单元测试；以真实 Cloud 为对端手工验证 clone 闭环与 Cloud 重启。
   实现与本地验证可与步骤 1 并行，合入必须在步骤 1 之后，否则保活会被旧策略断开。
3. desktop：submodule gitlink 移到包含步骤 1 的 Cloud 主干合并提交并重新生成客户端（只有注释变化）。
   同时更正前序决策中的一处事实：desktop#600 已关闭而未合并，当前 gitlink 仍是 cloud#19 的 PR head
   `535ca76c`；它可从 Cloud 主干到达，契约内容与主干只有注释差异。

步骤 2 合入之前 Controller 行为与现状一致；从步骤 2 起，云端模式的领取节拍按 D2 改变，本机模式不受
影响。核心用例在本文进入 `approved` 后补充。
