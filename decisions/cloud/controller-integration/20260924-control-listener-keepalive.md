---
status: proposed
---

# Controller 控制监听允许 HTTP/2 保活：放宽 PING 执行策略，并更正 `Drain` 的持有者义务表述

Cloud 的 Controller gRPC 监听（`control.grpc_addr`）改为接受客户端的 HTTP/2 PING 保活：两次 PING 间隔
不短于 5 秒即不计违规，连接上没有活动流时也允许 PING。`signals.proto` 中 `Drain` 的注释改为与根决策
D5 一致的表述：它表示发送它的 Cloud 实例即将停止，持有者停止对它领取直到新的 `Watch` 建立，不要求
释放租约，也不取消用户任务。两项都不改变任何消息、状态码、事务或授权。

本决策处于 `proposed` 状态，是
[Cloud 拥有 Internal Control gRPC 契约](0-cloud-owned-internal-grpc-contract.md)的后续决策，修改其 D7
（监听）的传输参数，并澄清 D5 中 `Drain` 的持有者义务。它只涉及 `internal/controlgrpc` 的服务端选项与
`proto/ora/cloud/internal/v1/signals.proto` 的注释；Controller 侧的消费方式与保活参数见
[Controller 消费 Watch 信号流](../../controller/api-boundary/20260924-controller-consumes-watch-signals.md)。

## 继承与修改

| 根决策约定 | 本文处理 |
|---|---|
| D1：proto 是唯一契约源；`v1` 只接受非破坏性修改；生成物提交并检查漂移 | 继承；注释修改是非破坏性变更，`internal/controlpb` 随之重新生成 |
| D5：`Watch` 至多一次、不持久化、缓冲有界；关停前 `Drain`，之后流正常结束，新订阅为 `UNAVAILABLE` | 继承，不变 |
| D5：持有者收到 `Drain` 后不再领取，不取消用户任务、不销毁沙盒 | 继承并澄清：不再领取持续到新流建立；不要求释放租约（D2） |
| D7：独立地址监听、不经 Gateway、只能位于回环或私网；`x-ora-controller-id` 声明身份 | 继承，不变 |
| D7：监听使用 grpc-go 默认传输参数（未明示） | 修改：显式放宽 keepalive 执行策略（D1） |

## 决策

### D1：接受间隔不短于 5 秒的客户端 PING，允许无活动流时 PING

`controlgrpc.New` 构造服务端时设置执行策略 `MinTime = 5s`、`PermitWithoutStream = true`。

- grpc-go 默认策略为 `MinTime = 5min`、`PermitWithoutStream = false`：间隔更短的 PING，或无活动流时的
  PING，各计一次违规；超过两次即以 `GOAWAY(ENHANCE_YOUR_CALM, "too_many_pings")` 关闭整条连接，连接上
  的 unary 调用与 `Watch` 一并中断。按默认策略，Controller 在空闲 `Watch` 上的 30 秒保活约两分钟后被
  断开，并在之后反复发生。
- 5 秒低于任何合理的客户端间隔（grpc-go 客户端的下限是 10 秒，Controller 使用 30 秒），为网络抖动留出
  余量，同时仍能拒绝真正的 PING 洪泛。
- 允许无活动流时 PING：Controller 在 `Watch` 断开后的回退期间同样需要保活，才能在下一个普通调用前
  发现失联连接。
- 监听仍只能位于回环或私网（D7 不变），放宽 PING 策略不扩大暴露面。

服务端不主动发起保活 PING：Go 的 TCP 监听默认对接受的连接启用 TCP keepalive，失联 Controller 的连接
最终会被关闭，其 `Watch` 上下文随之取消并退订；在此之前订阅者缓冲有界，失联订阅只会丢弃自己的信号，
不阻塞发布者，也不影响其他订阅者。

### D2：`Drain` 的注释与 D5 一致

`signals.proto` 中 `Drain` 现有注释为"要求持有者停止领取并排空在途协调，然后释放租约"。改为：发送它
的 Cloud 实例即将停止；持有者停止对其领取，直到新的 `Watch` 建立；它不要求释放租约，从不取消用户
任务或销毁沙盒。

- 根决策 D5 与 `cmd/server` 的关停顺序都把 `Drain` 定义为实例关停前的提示；此时释放租约的调用会被
  正在停止的服务端拒绝，而租约是 PostgreSQL 中的事实，Cloud 重启后在过期前仍然有效。
- 注释是契约的一部分，消费方据此实现；注释与决策不一致会迫使实现者二选一。
- 多实例下"让位"意义上的排空若有需要，由 scale 决策以新的信号或字段表达，不复用这条注释。

## 不变量

1. 监听接受间隔不短于 5 秒的客户端 PING，包括连接上没有活动流时；按此间隔保活的 Controller 不会被
   `GOAWAY` 断开。
2. 保活参数不改变任何 RPC 的消息、状态码、事务或授权；监听仍只能位于回环或私网。
3. `Drain` 只表示发送它的实例即将停止；它不改变归属，不要求释放租约，不取消用户任务。

## 为什么不是这些替代方案

| 方案 | 取舍 |
|---|---|
| 保持默认策略，Controller 以 5 分钟以上的间隔保活 | Cloud 无需改动；但失联检测超过 5 分钟，且间隔贴近 `MinTime` 时网络抖动仍会累计违规 |
| Controller 只用 TCP keepalive | Cloud 无需改动；但探测的是最近的 TCP 对端，经过代理时不能证明 Cloud 仍在服务 |
| 服务端同时主动 PING 客户端 | 能更快清理失联订阅；但 TCP keepalive 已保证最终清理，而缓冲有界使失联订阅不影响他人 |
| `MinTime` 取 10 秒 | 更保守；但与 grpc-go 客户端下限相同，没有余量，一次抖动即可计入违规 |
| 保留 `Drain` 注释中的"释放租约" | 为多实例预留"让位"语义；但与当前唯一的发送时机矛盾，让位应由 scale 决策显式定义 |

## 本决策未解决的问题

- TLS、Controller 认证与非私网监听，仍留给认证 ADR。
- 多实例下 `Drain` 的让位语义与 `NodeAssignment`，仍留给 scale 决策。

## 落地顺序

1. `internal/controlgrpc` 设置执行策略；`signals.proto` 更正注释并重新生成 `internal/controlpb`；
   `internal/controlgrpc` 的中英文 README 同步。完成条件：测试在无活动流的连接上以略长于 5 秒的间隔
   连续发送 PING，服务端全部回应且不发送 `GOAWAY`（同一测试在默认策略下会在第四次 PING 收到
   `GOAWAY`）；lint、breaking 与生成漂移检查通过。
2. desktop 把 submodule 移到包含步骤 1 的主干合并提交，Controller 启用保活（Controller 后续决策）。
   Controller 的保活必须在步骤 1 部署之后才能上线。

核心用例在本文进入 `approved` 后补充。
