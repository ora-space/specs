# 个人模型访问：归属、冻结、撤销和密钥隔离

后续[个人 Agent 配置用例](personal-agent-configurations.md)单独记录 Claude、插件配置及升级回归
的新义务，当前均为 `Missing`；本文保留原 OpenCode/模型访问证据的适用范围。

决策依据：[个人连接与运行代理](../../../decisions/cloud/model-access/0-personal-model-connections-and-run-scoped-proxy.md)。
状态对应直接证据范围；生产 CLI 与真实上游的联调不能由下层替身测试代替。
两协议的受控上游浏览器/API 证据见 [PR 检视与验证记录](https://github.com/ora-space/cluster/pull/15#issuecomment-6107672919)，实施阶段记录见 [Specs PR 归档](https://github.com/ora-space/specs/pull/76#issuecomment-6107673463)。
两种协议的正常路径连接同一真实上游，不代表多供应商兼容。证据只覆盖各自明确的义务；多身份浏览器、全栈崩溃和灾备仍保留 Partial/Missing。

## Personal model configuration must remain owner scoped

- 风险：另一个成员或管理员读取、修改个人连接或 Key。
- 前置：两个有效用户，各有个人连接。
- 触发：公开 CRUD、默认选择、凭据写入及跨用户重试。
- 必须成立：归属来自已验证身份；响应仅含公开字段；资源版本和幂等性由 PostgreSQL 保障。
- 禁止结果：请求体选定用户、读取 Key/密文、重复写入或覆盖并发修改。
- 证据：`cloud/internal/core/model_connections_db_test.go` 真实 PostgreSQL；HTTP 边界及严格结构测试；两协议真实连接的浏览器路径覆盖元数据保存、凭据已配置状态和默认模型选择；真实 Key 经 Gateway API 直接写入，未经过浏览器 Key 输入；页面输入 Key 的路径仍仅有受控上游记录覆盖。状态 `Partial`，多身份及完整配置矩阵的浏览器路径仍待直接验收。

## Model bindings must freeze inputs and never revive after revocation

- 风险：排队期间编辑配置改变已创建运行，或重新启用连接复活旧授权。
- 前置：OpenCode 运行已绑定个人连接；之后替换 Key、停用连接或清除 Key。
- 触发：编辑配置或密钥后再次取得/使用旧授权，或重新启用连接。
- 必须成立：编辑仅影响新运行；撤销绑定永久失效；缺默认配置的评论和运行原子回滚。
- 禁止结果：旧运行读取新 Key、缺配置运行无限等待、重新启用后旧绑定再次获权。
- 证据：`cloud/internal/core/model_connections_db_test.go` 冻结、原子回滚、清除/停用后重新配置矩阵。状态 `Covered`（数据库义务）。

## Model grants must follow confirmed runtime scope and current authorization

- 风险：陈旧或错误证书目标申请、续期或使用另一个运行的授权。
- 前置：已登记会话执行、当前运行绑定及确认过的 Node。
- 触发：申请、续期和使用临时授权。
- 必须成立：证书用途/租户/Workspace/代次和数据库目标匹配；当前账号、成员、能力、会话和维护权均有效。
- 禁止结果：调用者选定 Node 身份、未确认绑定获得授权、失效账号/成员续期、仅持 WSS 证书调用。
- 证据：Cloud 的真实 PostgreSQL 授权矩阵；Desktop 的真实 mTLS grant/renew/revoke 测试；cluster 证书用途测试。状态 `Partial`，全栈故障恢复尚缺直接证据。

## Forwarding must preserve protocol streams and cancel revoked requests

- 风险：代理泄露认证头、改用另一模型、重试生成或撤销后继续调用。
- 前置：一条有效运行授权，上游可持续发送 SSE。
- 触发：调用冻结模型、上游错误、撤销绑定或超时。
- 必须成立：两种协议和工具数据持续转发；加入冻结的认证与模型；撤销取消上游；HTTP 失败不回显诊断正文。
- 禁止结果：跨模型请求、客户端指定目标、跟随重定向、绕过当前部署地址策略或把真实 Key 返回客户端。非企业模式仍禁止私网；企业模式的新增许可与不可覆盖禁区见[部署方模型出口用例](deployment-owned-egress.md)，其新增义务和直接证据单独维护。
- 证据：`cloud/internal/modelgateway/service_test.go`、`transport_test.go` 使用真实 TLS HTTP 边界验证转发、撤销、错误过滤与地址策略；`events_test.go`、`framing_test.go`、`completion_test.go` 直接覆盖 HTTP 200 内错误、SSE 格式及阻塞写入/正常结束竞态。`dns_test.go` 验证真实 HTTPS DNS 双栈、混合私网/基准地址拒绝、重定向及畸形响应；`diagnostics_test.go`、`diagnostics_classification_test.go` 验证有限分类和诊断过滤。可选 `TestLiveTrustedDNSAndVerifiedUpstreamTLS` 只证明无凭据连接与 TLS。状态 `Partial`，受控 CLI 两协议证据见 [PR 记录](https://github.com/ora-space/cluster/pull/15#issuecomment-6107672919)，真实供应商正常路径已由官方 OpenCode 两协议多轮工具会话验证；撤销、错误和故障的完整真实上游矩阵仍缺直接证据。

## Personal model usage must remain controlled by the run initiator

- 风险：协作成员通过追加消息消耗另一个用户的个人模型额度。
- 前置：有模型绑定的 Thread；发起者、其他有效成员和当前管理员。
- 触发：读取、追加消息和结束运行。
- 必须成立：仅发起者追加消息；其他成员可读取；发起者及管理员可结束；终态均不可追加。
- 禁止结果：界面禁用但服务端仍受理、管理员默认获得 Key 权限、Echo 历史权限被一并改变。
- 证据：Cloud PostgreSQL 权限矩阵与前端 Thread 测试。两协议真实 CLI/API 正常路径补充发起者的追加和结束，状态 `Covered`（规则与界面义务）；多身份浏览器路径仍为 `Partial`。

## Real OpenCode must modify the checkout and deliver a secret-free Revision

- 风险：只有 Echo 或模拟器跑通，实际 CLI、工具调用、成果保存及释放未接通。
- 前置：独立真实 Compose 栈，安装固定版本官方 OpenCode；真实上游路径通过页面保存连接元数据与默认模型，Key 经 Gateway API 写入。受控上游另有页面输入 Key 的证据，两条路径分别判断。
- 触发：登录、创建项目并 clone、@OpenCode、追加轮次、结束会话。
- 必须成立：真实 CLI 读取/修改 checkout，JSONL 与 bundle 保存并校验，Git 作者归发起者，运行 Workspace 释放。
- 禁止结果：以替身代替 CLI、Key/临时令牌进入 Git/日志/截图、未登记成果就宣称完成。
- 验证义务与证据：

| 义务 | 代表性直接证据 | 状态 |
| --- | --- | --- |
| 受控上游页面配置、官方 CLI 两轮读写、结束和同 Run 成果恢复 | [受控上游 CLI 验证记录](https://github.com/ora-space/cluster/pull/15#issuecomment-6107672919)，包括 Git 作者、对象摘要及容器/卷释放 | Covered |
| 同一真实上游的两协议正常运行、成果和 Workspace 释放 | [cluster 固定版本验收脚本](https://github.com/ora-space/cluster/blob/aef10f3b6de16bf0bed6797a37bf0b1b3d9bc7c5/scripts/opencode-session-acceptance.py) 的 `--real-default-subject` 路径；使用页面已保存的默认模型，完成模型/工具调用、追加、结束、同 Run Revision 恢复/摘要/Git 作者和容器/卷释放 | Covered |
| 真实 Key 与临时令牌不进入 Git、日志、截图及 Revision | 上述脚本的 `assert_no_credential_material` 只匹配 fixture Key 和私钥头；另有 JSONL 与恢复结果文件的历史真实 Key 字节核对，但未覆盖临时令牌、全部 Git/日志/截图，也没有可复跑的真实凭据排除检查 | Partial |
| 真实上游的浏览器 Key 输入、默认模型到运行闭环 | 元数据保存、已配置状态和默认模型选择有页面证据；真实 Key 绕过浏览器输入，经 Gateway API 写入 | Partial |

正常结束证据不证明授权已撤销；撤销、全栈崩溃、多身份和灾备仍按对应用例验证。
