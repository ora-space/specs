# Node 的 git-access 凭据隔离与 Cloud 模式配置

依据[已批准的 node/revision 后续决策](../../../decisions/node/revision/20261012-push-delivery-ref-and-fetch-prior-revision.md)
（下称“本 ADR”）D1、D7，承担 [Git Server 根决策](../../../decisions/cloud/git-server/0-sandbox-scoped-git-server-over-https.md)
D5 在 Node 一侧的凭据规则。本 ADR 尚未实现：desktop 的 Node 只接收并校验 git-access 材料（`apps/ora-node/src/git_access.rs`），
还没有用它推送或拉取，所以传输仓库相关义务为 `Missing`。cluster 的 Revision 验收改走 Git 后只作补充证据，不据此
标记 `Covered`。

## Credentialed Git Runs Only In The Node Private Transport Repository

### 风险

Node 身份的 Git 进程读取 Agent 写过的 checkout 配置：`credential.helper`、`core.askPass` 以 Node 身份执行任意命令，
`http.<url>.sslVerify` 关闭证书校验，`url.<base>.insteadOf` 把推送改写到别处；git-access 私钥进入工作负载进程、
仓库配置、日志或 journal。

### 前置状态

Node 配置了 `git_access`；checkout 由工作负载拥有，Agent 可能写过其中的 `.git/config`、钩子与 ref。

### 触发

交付推送与恢复拉取；checkout 的 `.git/config` 预置恶意的 `credential.helper`、`core.askPass`、
`http.<url>.sslVerify=false`、`http.<url>.extraHeader`、`url.<base>.insteadOf`，以及恶意钩子；执行结束、失败或 Node
重启。

### 必须成立

- 带证书的 Git 命令只在 Node home 下的私有裸传输仓库（`0700`，属主 Node）中以 Node 身份运行；传输仓库经
  `objects/info/alternates` 借用 checkout 的对象，不枚举 checkout 的 ref，从不读取 checkout 的配置与钩子。
- 证书、私钥、CA 与 `Ora-Execution-Id` 只以命令行 `-c` 传入，并带 `credential.helper=`、`http.followRedirects=false`、
  `GIT_CONFIG_NOSYSTEM=1`、`GIT_CONFIG_GLOBAL=/dev/null`、`GIT_TERMINAL_PROMPT=0` 与只允许 `https` 的协议策略；这些值
  不写入任何仓库配置，不进入日志、journal 或结果。
- 地址只接受不带用户信息、查询与片段的 `https`，ref 只接受 `refs/ora/deliveries/` 下的名字；按 URL 访问，不新增
  remote。
- 传输仓库在交付的终态证据持久化后或恢复结束后删除；Node 启动时清空整个传输目录。

### 禁止结果

checkout 中的任何配置或钩子以 Node 身份生效；推送去往输入之外的地址；私钥出现在工作负载可读的文件、进程参数或
环境之外的任何持久位置。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| checkout 配置不生效 | 尚无实现。需要以真实 Git 在预置恶意 `credential.helper`、`core.askPass`、`http.<url>.sslVerify`、`url.<base>.insteadOf` 与钩子的 checkout 上推送和拉取，断言它们都未执行或生效 | Missing |
| 凭据只经命令行传入且不落盘 | 尚无实现。需要断言传输仓库与 checkout 的配置、Node 日志、journal 与结果中都不出现私钥内容、证书参数或 `Ora-Execution-Id` 请求头，且工作负载身份的进程读不到私钥 | Missing |
| 地址与 ref 的形状 | 尚无实现；Node 协议一侧的输入校验见[Agent 运行的协议消息](../protocol/agent-run-messages.md#upload-grants-stay-in-memory-and-deliveries-stay-in-their-namespace) | Missing |
| 传输仓库的生命周期 | 尚无实现。需要断言执行结束、失败与 Node 重启后传输目录被清空 | Missing |

### 决策依据

本 ADR D1，不变量 1、2；Git Server 根决策 D5，不变量 9。

## Cloud Mode Node Refuses To Start Without Git Access

### 风险

Cloud 模式的 Node 缺少 git-access 材料仍然启动，一整次 Agent 会话之后才以交付失败暴露配置错误；Node 在没有
`git_access` 时仍声明交付能力。

### 前置状态

Node 配置文件中有或没有 `control.target`、`clone` 与 `git_access`。

### 触发

Node 启动；Node 向 Controller 声明能力。

### 必须成立

- 配置了 `control.target` 而缺少 `git_access` 时拒绝启动，只给有限代码 `git_access_required`。
- `git_access` 的材料按现有规则校验：绝对路径、可读、CA 不含私钥块、证书与私钥匹配；错误只给有限代码，不含
  路径或内容。
- 只有同时配置 `clone` 与 `git_access` 时声明 `revision_delivery`；`revision_restore` 仍要求会话与交付都可用。
- 没有 `control.target` 的本地部署与测试中 `git_access` 可选。

### 禁止结果

Cloud 模式的 Node 没有 `git_access` 却开始接收工作；缺少 `git_access` 的 Node 声明 `revision_delivery`。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| Cloud 模式必填 | 尚无实现：`git_access` 目前在任何模式下都可选 | Missing |
| 材料校验 | desktop `apps/ora-node/src/git_access/tests.rs::issued_material_validates`、`::unusable_material_fails_closed_with_bounded_codes`：相对路径、不可读、CA 含私钥块、证书与私钥不匹配都以有限代码失败，代码不含路径或内容；`::configuration_accepts_only_material_paths`：未知字段与部署给出的服务端地址被拒绝 | Covered |
| 能力声明 | 尚无实现：`apps/ora-node/src/service/worker.rs` 声明 `revision_delivery` 时还不要求 `git_access` | Missing |

### 决策依据

本 ADR D7，不变量 10。
