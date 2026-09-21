---
status: proposed
---

# Controller 空闲心跳：会话保活责任由 Controller 承担，Node 的帧期限只撤销无响应对端

`ControllerToNodeMessage` 增加 `Heartbeat`。Controller 在一条会话上没有待协调命令时，按查询间隔
发送心跳；有命令时查询本身即保活，不额外发心跳。Node 在读循环内就地校验心跳的 `controller_id`
并丢弃它，不进入执行 worker 队列，不回复。Node 的帧期限 `frame_timeout_ms` 语义不变：只有在期限内
一帧都没收到的对端才被撤销。改变的是空闲会话不再每个帧期限断开重连一次，日志中的会话关闭警告
只对应真实的对端无响应。

本决策处于 `proposed` 状态，基于[协议根决策](0-controller-node-protocol.md)，涉及
`ora-node-protocol` 的消息集合、`ora-controller` 的会话 tick 与 `ora-node` 的会话读循环。它不改变
编码、帧格式、协议版本号、单会话撤销规则、查询与事件确认的职责划分，也不引入 Controller
拨出或多会话。

## 背景

[本机归属决策](../../controller/persistence/0-local-controller-node-ownership.md) D2 规定 Node 同时只有
一个有效控制会话，且不以"最后连接者获胜"抢占仍有效的旧连接；旧会话只有在关闭或经有限超时撤销
后才能被替换。因此 Node 必须能在有限时间内判定对端无响应：本机 socket 在对端进程死亡时给出 EOF，
但对端进程存活却卡死或写了半帧时不会有任何信号。`frame_timeout_ms` 就是这个撤销期限，统一约束
读、写与准入回复。

现有文档把保活责任放在 Controller 的查询上（"An idle Controller should periodically query its
executions"），但查询需要执行身份；Controller 没有待协调命令时无东西可查，实现上什么都不发。
结果是空闲会话每个 `frame_timeout_ms` 被 Node 撤销一次，两侧各记录一条警告，`reconnect_ms` 后重连。
持久接受、结果接管与"断连不取消执行"不受影响，但空闲和故障在日志里不可区分，Controller 管理的
Node 越多、空闲越久，这种噪音越多。

## 继承与修改

| 来源与约定 | 本文处理 |
|---|---|
| 协议根决策 D2：统一消息帧；D6：控制和心跳优先于大量日志 | 继承；心跳是控制类消息，与其他控制消息共用同一连接与优先级 |
| 协议根决策 D5：第一版最小消息集合 | 修改：`Controller → Node` 增加 `Heartbeat`，无执行身份关联 |
| 协议根决策 D4：断开不把执行标记为失败；重连后按执行身份对账 | 继承，不变；心跳不参与对账，也不携带执行证据 |
| 本机归属决策 D2：单一有效会话、不抢占、有限超时撤销 | 继承，不变；撤销期限继续由 Node 的 `frame_timeout_ms` 定义，本文只让守规矩的 Controller 不再触发它 |
| 本机归属决策 D2：执行与会话收发解耦，长时间 clone 不阻塞心跳 | 继承并对称化：Node 对 Controller 心跳的处理同样不经过被 Git 占用的 worker |
| 现有文档"空闲 Controller 应定期查询执行"、"查询兼做保活" | 修改：保活由心跳承担，查询只在有待协调命令时发送 |

## 决策

### D1：`Heartbeat` 是 Controller 的会话保活消息

```text
Controller → Node: Heartbeat { controller_id }
```

它只携带 `controller_id`，与 Node → Controller 的 `Heartbeat { node }` 对称；不携带 `operation_id`、
`execution_id` 或任何执行状态，不要求回复，不影响事件重放、确认或查询语义。编码校验只要求
`controller_id` 非空。

选择新增消息而不是复用 `GetExecutionStatus`：查询语义上要求一个存在的执行身份，用历史执行或
占位身份"假查询"会在 Node 制造无意义的状态回复和日志，并把保活与对账两种职责混在一起。

### D2：保活责任在 Controller，频率复用查询间隔

Controller 每个会话有独立的 tick（`query_interval_ms`）。tick 时若该 Node 有待协调命令，发送查询
（现有行为）；否则发送 `Heartbeat`。任何一帧都能重置 Node 的读期限，因此有查询时不再叠加心跳。

不新增配置项：Controller 并不知道 Node 的 `frame_timeout_ms`，派生不出更合适的值；现有部署规则
已要求 `query_interval_ms` 小于 Node 的帧期限，这条规则的含义从"查询间隔"扩展为"会话发帧间隔"。
minicloud 的 `1000 ms` 对 `10000 ms` 帧期限留有十倍余量。

每条会话独立发送心跳是长连接固有的开销：一个 Controller 管理 N 个 Node 就有 N 条会话，每条会话
每秒一帧几十字节。连接数才是规模变量，而"每个 Node 一条会话"由单会话决策确定，不由本文改变。

### D3：Node 就地处理心跳，不进入 worker 队列

Node 读循环收到 `Heartbeat` 时，校验 `controller_id` 等于会话绑定的 Controller，然后丢弃；不回复，
不生成 `Request`，不经过执行 worker。`controller_id` 不匹配按现有"冲突消息关闭会话"规则处理。

必须就地处理的原因：现有实现把每条消息交给 worker 并套用 `frame_timeout_ms` 的等待期限，Git 占用
worker 时排队的查询可能超时并关闭会话（文档已承认这一代价）。如果心跳也排队，空闲保活反而会在
Node 忙碌时制造断连，与本文目的相反。

### D4：兼容性

协议版本号不变。旧版本 Node 收到未知 `message_type` 会解码失败并关闭会话；新 Controller 对旧 Node
会退化为每个 `query_interval_ms` 断开重连一次，比现状更频繁。Controller 与 Node 由同一启动器或
`--single-node` 成对部署，不单独升级；接受该退化并在文档中注明，不为此引入版本协商分支。

## 不变量

1. 空闲会话在 Controller 存活且能发帧时不会被 Node 因帧期限撤销。
2. Node 的帧期限语义不变：期限内无任何帧到达的对端仍被撤销，撤销后新连接可以建立会话。
3. `Heartbeat` 不携带执行身份，不产生回复，不改变查询、重放与确认的任何顺序或幂等规则。
4. Node 对心跳的处理不依赖执行 worker 的可用性；worker 忙碌不导致心跳超时。
5. 有待协调命令时 Controller 发查询而不是心跳；两者不在同一 tick 叠加。
6. `controller_id` 与会话绑定不一致的心跳导致会话关闭，与其他冲突消息一致。

## 为什么不是这些替代方案

| 方案 | 取舍 |
|---|---|
| Node 空闲时不因读超时断开，只要自己的心跳写成功 | 写成功只证明内核缓冲区未满，卡死的 Controller 仍可被写很久；Node 失去撤销僵死会话的能力，正是读期限要保护的东西 |
| 来了新连接才检查旧会话是否静默超期（按需撤销） | 空闲时无需发帧，但把"不抢占仍有效连接"改成"静默够久即可被抢占"，同 `ControllerId` 误启动的第二个实例会顶掉空闲的第一个，违背单会话决策的初衷 |
| 空闲时重复查询最近一条已完成执行 | 无需新消息，但用语义消息当保活，在 Node 产生无意义的状态回复与日志，并让查询同时承担两种职责 |
| 调大 `frame_timeout_ms` | 只改配置，但只是降低频率，同时把僵死对端的撤销拖慢同样倍数 |
| 依赖传输层保活（TCP keepalive、HTTP/2 PING） | 协议根决策要求同一消息帧跑在 Unix socket、SSH 与 TLS 上，本机 Unix socket 与 SSH 流没有这一层；消息级心跳是协议内的长期答案 |
| 为心跳新增独立间隔配置 | 可独立调节，但 Controller 无法据 Node 的期限派生它，只是多一个要人工对齐的参数 |

## 本决策未解决的问题

- Controller 侧对"对端 EOF 关闭"与"读写超时"的日志级别区分，本文只消除空闲触发的关闭。
- 网络 Transport 下的心跳间隔与丢包判定，留给 Transport 决策。
- Node 对心跳到达间隔的主动监测（例如据此判定 Controller 半死），本文只用心跳重置既有读期限。

## 落地顺序

1. 协议与两侧实现：新增消息、Controller tick 分支、Node 就地处理；更新协议、Node IPC 与
   Controller 运行时文档。完成条件：Controller framed-session 测试证明空闲时在查询间隔内发出心跳、
   有命令时发出查询；Node 真实 IPC 测试证明只发心跳的空闲会话跨过多个帧期限仍保持，停止发帧后
   在期限内被撤销且替换连接可以建立会话；Node 忙碌（HTTPS 暂停中的 clone）期间心跳不关闭会话。
   此阶段调用方可观察行为只有一处改变：空闲时不再出现周期性的会话关闭警告与重连。
2. 进入 `approved` 后在 `specs/test-cases/node/protocol/` 登记上述验证义务。
