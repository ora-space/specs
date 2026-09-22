---
status: proposed
---

# Controller gRPC 错误以规范状态码为主分类，`ErrorDetail` 枚举随 status details 附带

Controller 的 gRPC 调用失败时，以 gRPC 规范状态码（`ABORTED`、`INVALID_ARGUMENT`、`UNAVAILABLE`、
`NOT_FOUND`）表达分类，并把 `ora.controller.v1.ErrorDetail{ErrorCode}` 作为 `google.rpc.Status`
的 detail 一并返回。调用方按状态码或按枚举分支都得到同一分类，两者由 Controller 同一处映射产生，
不解析错误文本。这修改了[根决策](0-controller-hosted-client-api.md) D4 中"错误分类以 proto 枚举
携带在 gRPC status details 中"的表述：枚举仍在 details 中，但状态码不再是无意义的占位值。

本决策处于 `proposed` 状态，只覆盖这一项：Controller gRPC 失败的携带方式及其映射。契约归属、
生成链、Transport、租户边界与过渡 JSON 接口均不变。Cloud 侧如何消费该分类由
[Cloud 消费 Controller gRPC 决策](../../cloud/controller-integration/0-cloud-consumes-controller-grpc.md)
承担。

## 继承与修改

| 根决策约定 | 本文处理 |
|---|---|
| D4：契约由本仓库 `.proto` 拥有，两侧生成并提交，`buf lint`／`buf breaking` | 继承，不变 |
| D4：错误分类以 proto 枚举携带在 gRPC status details 中，Rust 映射只写一次，Go 侧按枚举分支 | 修改：枚举仍附在 details，但 gRPC 状态码同时承载分类；Rust 侧一处映射同时产出状态码与枚举 |
| 不变量 5：同一 DTO 不存在 serde、proto 与手写转换三份定义 | 继承，不变；错误映射是 `ora_controller::Error` 到 proto 的单向投影，不引入第二份错误类型 |
| 不变量 7：契约不含租户、用户或成员字段 | 继承，不变 |
| 过渡 JSON 接口的 HTTP 状态映射（400／404／409／503） | 继承，不变；两个边界各自一份映射，语义对应见下表 |

## 决策

### D1：状态码与枚举的固定映射

| `ora_controller::Error` | gRPC 状态码 | `ErrorCode` | 过渡 JSON 接口 |
|---|---|---|---|
| `Conflict`、`AlreadyRunning` | `ABORTED` | `CONFLICT` | 409 |
| `Validation` | `INVALID_ARGUMENT` | `INVALID_INPUT` | 400 |
| `Io`、`Sql`、`Encoding`、`InvalidStorage`、`Injected`、`Configuration` | `UNAVAILABLE` | `UNAVAILABLE` | 503 |
| 查询无记录 | `NOT_FOUND` | `NOT_FOUND` | 404 |

映射在 Controller 的 gRPC 适配层只写一次，同时产出状态码与 `ErrorDetail`；不存在"状态码说 A、
枚举说 B"的可能。`ErrorCode` 的取值集合与状态码一一对应，新增分类必须同时新增枚举值与状态码
映射，`buf breaking` 保护枚举只增不改。

选 `ABORTED` 而不是 `ALREADY_EXISTS` 表达冲突：冲突包含"同身份不同输入"与"另一运行时持有
所有者"，都不是"资源已存在可直接读取"，而是调用方必须改变输入或等待的前置条件失败。

### D2：为什么状态码必须承载分类

- grpc-go 的服务端配置重试、拦截器、指标与链路追踪都以状态码为键：`UNAVAILABLE` 可重试、
  `ABORTED`／`INVALID_ARGUMENT` 不可重试，这些策略无需了解本契约即可正确工作。全部失败共用一个
  占位码会让这些通用能力失效。
- grpcurl 等通用工具与未注册 `ErrorDetail` 类型的调用方也能读懂分类。
- 枚举仍然保留，因为状态码的粒度不足以承载未来更细的分类（例如区分不同的冲突原因），而枚举可以
  在不改状态码的前提下细分。

### D3：调用方的解读规则

调用方以 detail 中的枚举为准；没有 detail 的 gRPC 状态（例如传输层产生的 `UNAVAILABLE`）不是
Controller 的裁决，调用方须能区分"Controller 自报不可用"与"连接失败"。截止时间超出与取消由
调用方自身产生，Controller 不为其附加 `ErrorDetail`。

## 不变量

1. Controller 返回的每个失败 gRPC 状态都携带 `ErrorDetail`，其枚举与状态码按 D1 一一对应。
2. 状态码与枚举由同一处映射产生；不存在两者不一致的响应。
3. 传输层失败与截止时间不携带 `ErrorDetail`，调用方据此区分 Controller 裁决与连接问题。

## 为什么不是这些替代方案

| 方案 | 取舍 |
|---|---|
| 仅 `ErrorDetail`，状态码统一占位（根决策原表述） | 严格符合原文，但 grpc 生态的重试、观测、通用工具全部失去分类信息，而选 gRPC 的理由正是复用该生态 |
| 仅状态码，不附枚举 | 通用工具友好，但分类不可枚举细分，未来细分冲突原因时只能改状态码或加消息文本 |
| 用 `google.rpc.ErrorInfo.reason` 字符串承载枚举名 | 与标准类型兼容，但 reason 是自由字符串，两侧对齐回到人工约定 |

## 本决策未解决的问题

- 流式调用中途失败的携带方式，随流式交互决策一起定。
- 认证失败（`UNAUTHENTICATED`／`PERMISSION_DENIED`）的枚举与来源，留给认证 ADR。

## 落地顺序

1. 契约：`ErrorDetail` 与 `ErrorCode` 已在 `proto/ora/controller/v1/controller.proto` 中定义，本文
   固定其与状态码的对应关系。
2. Controller 侧 gRPC 服务落地时按 D1 实现一处映射，测试断言冲突、无效输入、缺失、持久化失败四类
   响应的状态码与 detail 同时正确。
3. 进入 `approved` 后，在 `specs/test-cases/controller/api-boundary/` 的现有用例中登记"状态码与
   detail 一致"的验证义务。
