---
status: proposed
---

# Cloud 以生成的 Go client 调用 Controller gRPC：契约由 Controller 拥有、以 tag 锁定、编译即对齐

Ora Cloud（API Server）通过 gRPC 调用 Controller 的 `ora.controller.v1.ControllerService`。契约的
唯一来源是代码仓库 `ora-space/desktop` 的 `proto/` 目录，由 Controller 作为服务端拥有；Cloud 不复制
`.proto`，而是在自己的 `buf` 配置里以该仓库的 git tag（`controller-proto/vX.Y.Z`）为输入生成
`internal/controllerpb` 并提交，再由薄封装 `internal/controllerclient` 拨号、设置截止时间并把
Controller 的错误分类解成 Go 类型。契约变更的对齐信号是"Cloud 更新 tag 后能否编译"，不做人工
逐字段比对。这条调用链服务于 API Server 主动发起的交互；Controller 领取与推进操作的既有拉取
方向不变。

本决策处于 `proposed` 状态，覆盖 Cloud 仓库的 proto 生成链、生成包与 client 包的边界、错误映射、
Controller 地址配置和契约测试义务。它不定义 Controller 侧的服务实现（由
[Controller Client API 决策](../../controller/api-boundary/0-controller-hosted-client-api.md)承担）、
不引入认证或 TLS、不改变公开 HTTP API，也不把 clone 接入 Project／Operation 状态机——那是
后续决策。

## 继承与修改

| 来源与约定 | 本文处理 |
|---|---|
| Cloud 仓库 `docs/core-contract.md`：Cloud 是唯一业务权威存储；Controller 通过 `/internal/v1` 领取和推进操作，不持有 PG 连接 | 继承，不变；本文新增的是反方向的调用（Cloud → Controller），两者并存，各自服务不同交互 |
| [Controller Client API 决策](../../controller/api-boundary/0-controller-hosted-client-api.md) D4：`.proto` 放 desktop 仓库、两侧生成并提交、`buf lint`／`buf breaking` | 继承；本文只固定 Cloud 侧的消费方式 |
| [错误携带后续决策](../../controller/api-boundary/20260922-status-code-with-error-detail.md)：规范状态码 + `ErrorDetail`，映射由 Controller 一处产生 | 继承；本文 D3 描述 Cloud 侧如何解读，不重复定义映射 |
| 同决策不变量 7：契约不含租户、用户或成员字段 | 继承，不变；租户授权在 Cloud 公开边界完成后才调用 Controller，Controller 返回的记录由 Cloud 重新归属 |
| Cloud 现有 `internal/simulator` 的 Controller 替身 | 继承，不变；替身继续服务阶段一的操作推进测试，与本文的真实 Controller 调用不互相替代 |
| Cloud 的 OpenAPI／orval 生成与 CI 漂移检查 | 继承其模式：proto 生成物同样提交并由 CI 漂移检查守护 |

## 决策

### D1：契约输入是 desktop 仓库的 git tag，生成物提交

`proto/buf.gen.yaml` 以 `git_repo: ora-space/desktop`、`subdir: proto`、`ref: controller-proto/vX.Y.Z`
为输入；`managed` 模式为 `ora/controller/v1/controller.proto` 指定 `go_package`
`github.com/wanglongan587/cloud/internal/controllerpb`，因此契约文件本身不写 Go 包路径，不与
Cloud 的模块路径耦合。生成插件（`protocolbuffers/go`、`grpc/go`）固定版本。

生成的 `internal/controllerpb` 随仓库提交；CI 重新生成后 diff 必须为空。开发者可用
`CONTROLLER_PROTO_INPUT=<本地 desktop/proto 路径>` 覆盖输入在 tag 之前联调，但提交的生成物
必须来自配置中的 ref。

为什么用 tag 而不是分支：tag 不可变，"Cloud 当前对齐到哪个契约版本"是仓库里可读的事实；升级是
一次显式的 ref 变更 + 重新生成 + 编译，评审看到的就是接口差异本身。

### D2：`internal/controllerclient` 只做传输与错误分类

client 包只负责：把部署地址（`tcp://host:port` 或 `unix:///absolute/path`）转换为 grpc-go target、
建立明文连接（`tcp` 上为 h2c，`unix` 上为 h2；与 Controller 当前无认证的回环／socket 传输一致）、
为每次调用设置截止时间、把响应原样返回。它不含 clone 的业务含义，不缓存，不重试——重试语义
（同 `request_id` 重传返回原回执）由调用方按业务决定。

拨号只接受上述两种地址形式，其他形式在建立连接前拒绝，避免把 `http://` 之类的误配置变成一次
运行时的连接失败。

### D3：错误的 Cloud 侧解读

状态码与 `ErrorDetail` 的映射由
[Controller 侧决策](../../controller/api-boundary/20260922-status-code-with-error-detail.md)固定；
Cloud 只定义如何解读：

| `ErrorCode` | gRPC 状态 | Cloud 侧含义 |
|---|---|---|
| `CONFLICT` | `ABORTED` | 身份或输入与 Controller 持久记录冲突；重试同一调用不可能成功 |
| `INVALID_INPUT` | `INVALID_ARGUMENT` | 请求不合法 |
| `UNAVAILABLE` | `UNAVAILABLE` | Controller 持久化暂时不可用，没有任何接受发生，可重试 |
| `NOT_FOUND` | `NOT_FOUND` | 没有该执行身份的接受记录，区别于"已接受但无终态" |

client 把带 gRPC 状态的错误解成 `controllerclient.Error{Code, Status, Message}`：`Code` 来自
detail，没有 detail 时为 `UNSPECIFIED` 且 `Status` 仍可用（例如传输层的 `UNAVAILABLE` 与
Controller 自报的 `UNAVAILABLE` 由 `Code` 是否存在区分）。截止时间与取消不是 Controller 的裁决，
原样透传，不包装为 `Error`。两侧都不解析错误文本。

### D4：契约测试义务

生成只证明结构一致。行为一致由 Cloud 仓库的集成测试保证：以真实 `ora-controller --transport unix`
为对端，用生成的 client 覆盖接受、幂等重传、冲突、缺失与 Node 离线时的列表。二进制路径由
`ORA_CONTROLLER_BIN` 提供，未设置时测试跳过，与现有 `REQUIRE_POSTGRES` 的门控方式一致。该测试
在 Controller 侧 gRPC 服务落地后登记；在此之前 client 包的单元测试以进程内 stub 验证错误映射与
截止时间。

## 不变量

1. Cloud 仓库不含任何手写或复制的 `.proto`；`internal/controllerpb` 与配置中 ref 指向的契约无漂移。
2. Cloud 对齐的契约版本是仓库中可读的单一 ref；升级是显式的 ref 变更。
3. `internal/controllerclient` 不含业务规则、缓存或隐式重试；地址形式在拨号前校验。
4. Controller 错误按 `ErrorDetail` 枚举分类；截止时间与取消不伪装为 Controller 裁决。
5. Cloud → Controller 调用只在 Cloud 完成租户授权之后发生；契约与 client 不出现租户字段。
6. Controller 领取与推进操作的既有拉取方向不因本文改变。

## 为什么不是这些替代方案

| 方案 | 取舍 |
|---|---|
| 把 `.proto` 复制进 Cloud 仓库 | 两份来源必然漂移，人工对齐正是本文要消除的 |
| 独立的 proto 仓库 | 每次接口变更需要三处 PR；当前只有一个消费者，Controller 拥有契约更直接 |
| 以分支而不是 tag 为输入 | 生成结果随时间变化，"当前对齐到哪个版本"不再是仓库可读事实 |
| `.proto` 里直接写 Cloud 的 `go_package` | 服务端契约耦合调用方模块路径，第二个消费者出现即冲突 |
| 用 JSON over HTTP 复用现有 Gin 体系 | 可行但需要自定义流生命周期与错误封装，且 Controller 侧的双向流需求已选定 gRPC |

## 本决策未解决的问题

- Controller 地址在 Cloud 配置中的位置（`controller.address`）与多 Controller 部署的寻址，随首个
  真实调用方端点决定。
- 认证与 TLS：Controller 与 Cloud 之间的凭据由认证 ADR 决定，本文只承诺明文传输仅用于回环与
  Unix socket。
- clone 与 Project／Workspace 领域的关系，以及是否经 operation 状态机推进。
- PTY 等流式交互的 Cloud 侧消费方式。

## 落地顺序

1. 生成链与 client：`buf.gen.yaml`、生成包、`controllerclient` 与其 stub 测试、CI 漂移检查。完成
   条件：`go build ./...` 与 client 单元测试通过，CI 在 ref 变更外的任何生成物改动上失败。
2. Controller 侧 gRPC 服务落地后：登记 D4 的契约测试，并首次以 tag 而不是提交哈希锁定 ref。
3. 首个真实调用方端点（镜像 minicloud 语义的公开端点）作为独立决策提出，届时确定
   `controller.address` 配置与租户到 Controller 记录的归属规则。

进入 `approved` 后，在 `specs/test-cases/cloud/controller-integration/` 登记生成物无漂移、地址校验、
错误映射与契约测试的核心用例。
