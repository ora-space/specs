# Repo Executor：一次性授权、凭据隔离、出网与结果分类

依据已批准的 [Repo Executor 根决策](../../../decisions/cloud/repo-executor/0-repo-executor-sole-external-git-boundary.md)。
下文的 D 编号与不变量编号都指这篇根决策；“契约”指
[代码中心契约决策](../../../decisions/cloud/git-server/20261016-project-repository-as-code-hub-and-ref-contract.md)，
“共享出网决策”指[部署方共享出网决策](../../../decisions/cloud/model-access/20261017-deployment-owned-egress-and-explicit-resolution.md)。

本文登记执行器自身的义务。同步与发布的业务流程见 [仓库同步用例](../project/repository-binding-and-snapshot-sync.md) 与
[显式发布用例](../project/explicit-fast-forward-publish.md)；Git Server 一侧的授权与执行器出网的契约义务见
[代码中心契约用例](../git-server/project-repository-code-hub-contract.md)。执行器的实现尚未合入 cloud 的 main，下列义务都是
**Missing**，实现合入后按直接测试逐项更新。

## Every Executor Task Matches A Single-Use Authorization Field By Field

- 风险：执行器执行与授权不一致的任务，或同一授权被用两次；只读的 `probe` 授权被用来推送。
- 前置状态：Cloud 签发 `sync`、`publish`、`probe` 三种授权；Credential Store 记录兑换。
- 触发：改动任务中的任一绑定字段；同一授权提交两次；授权过期；用 `probe` 授权提交发布任务；旧用途 `clone`、`mirror_prepare`、`push`。
- 必须成立：签名、有效期和每个绑定字段都一致才兑换凭据；Credential Store 以授权 ID 独占兑换；`probe` 只读取 ref 广告；授权只约束开始任务与兑换，任务时长跟随租约。
- 禁止结果：字段不一致仍兑换；第二次兑换成功；`probe` 授权发出更新命令；兑换后授权过期让长任务失败。
- 验证义务与证据：**Missing**。
- 决策依据：D2，不变量 3。

## Credential Plaintext Never Leaves The Task Material Directory

- 风险：明文 token 出现在进程参数、环境变量、URL、Git 配置、日志或返回给 Cloud 的字段里；任务结束后材料残留。
- 前置状态：私有仓库的同步与发布任务；带 canary token 的凭据。
- 触发：成功、失败、超时、取消、panic 各路径；外部平台返回包含 token 的错误；仓库 URL 带用户信息。
- 必须成立：明文只写入 `0600` 材料文件，只经 `git-credential-ora` 交给作用域内的请求；stderr 截断前先脱敏；任务结束后材料目录不存在。
- 禁止结果：`/proc/<pid>/cmdline` 或环境中出现 token；带用户信息的 URL 被执行；清理失败没有被报告。
- 验证义务与证据：**Missing** — 需要按根决策验收的秘密扫描（字面、base64、base64url、百分号编码、共享前缀，带阳性对照）。
- 决策依据：D3，不变量 4。

## Executor Egress Uses The Shared Policy And Pins Checked Addresses For Git

- 风险：执行器另设放行规则或允许列表，比共享出网策略宽；`git` 自己再解析一次域名，DNS rebinding 绕过检查；`probe receive-pack` 走了另一条拨号路径。
- 前置状态：公网模式与企业模式的 `deployment-egress.json`；测试 resolver 可以在两次查询之间改变答案；隔离开发栈的精确测试 host 例外。
- 触发：同步、发布、`probe` 访问外部仓库；第一次解析得到允许的地址、第二次得到 cluster 地址；企业配置与开发例外同时给出；配置文件不合法。
- 必须成立：准入与解析复用共享出网决策的实现；每个访问外部仓库的 `git` 子进程启动前重新解析并检查全部地址，以 `http.curloptResolve` 只把检查过的地址交给 `git`，TLS 按原 hostname 校验；`probe receive-pack` 使用同一拨号器；企业配置与开发例外冲突、配置不合法时执行器不启动。
- 禁止结果：存在仓库专用的允许列表；`git` 连接到未经检查的地址；cluster、宿主、回环或链路本地地址被连接；跟随重定向或使用环境代理。
- 验证义务与证据：**Missing**。契约一侧的出网义务见代码中心契约用例 `Repo Executor Egress Stays Within The Deployment Allowlist And Resolver`。
- 决策依据：D4、D7，不变量 9；契约 D2“执行器出网”、不变量 15；共享出网决策 D1–D5。

## Executor Joins Egress Config Changes And Digest Verification

- 风险：更换出网配置后执行器仍按旧快照放行；执行器的配置与 Cloud、model-gateway 不一致而没有被发现。
- 前置状态：cluster 部署，执行器、Cloud、model-gateway 只读挂载同一份配置。
- 触发：部署入口更换配置；只替换执行器看到的文件；执行器加载失败。
- 必须成立：更换配置时执行器与 Cloud、model-gateway 一起停止、换文件、再启动；执行器在健康检查报告规范化配置的摘要和 schema；`ora verify` 发现摘要缺失或不一致时报错；运行中的同步与发布按 Operation 规则收尾。
- 禁止结果：执行器热加载配置；加载失败时退回公网模式；摘要不一致仍报告部署校验成功。
- 验证义务与证据：**Missing**。
- 决策依据：D9；共享出网决策 D6。

## Sync Writes Snapshot And Tracking Refs Atomically Under A Lease

- 风险：`+` refspec 让 lease 失效，覆盖别人推进的跟踪 ref；快照 ref 与跟踪 ref 只写了一个；`pre-receive` 拒绝被误判为 `superseded`。
- 前置状态：Binding 已有跟踪 ref；预置工具可以改写它。
- 触发：以旧的 `expectedTrackingOid` 推送；首次同步（全零）；同一提交重放；Operation 结束后用同一 ID 推送。
- 必须成立：refspec 不带 `+`；两个 ref 在一次 `--atomic` 推送中写入；`stale info` 报告 `superseded`（同一结果已写入时按成功），`hook declined` 报告 `failed(git_server_rejected)`；重放是空操作。
- 禁止结果：lease 不符时跟踪 ref 被覆盖；只写入一个 ref；两种拒绝不可区分。
- 验证义务与证据：**Missing**。
- 决策依据：D5，不变量 5、6。

## Publish Pushes Only A Fast-Forward Of A Git Server Commit And Classifies The Outcome

- 风险：发布推送了源 ref 不可达的提交或非快进；推送结果不明时执行器重推；后置核验失败被误判为确定结果。
- 前置状态：Git Server 上的 Workspace 分支或交付 ref；`push-git-fixture` 的 `normal`、`refuse`、`moved`、`slow-post`、`drop` 模式。
- 触发：源提交不在源 ref 历史中；`expectedOldOid` 不是源提交的祖先；外部已前进；平台拒绝；响应丢失；平台拒绝而后置 `ls-remote` 读取失败。
- 必须成立：快进只用 Git Server 取回的对象在本地判定；前置 `ls-remote` 与 `--force-with-lease` 都比对 `expectedOldOid`；推送报告与后置核验按 D8 的表分类，平台拒绝且后置读取失败为 `uncertain`；`uncertain` 不重推。
- 禁止结果：非快进写入外部；`conflict` 时发出更新命令；`uncertain` 后再次推送；读取失败被判为 `rejected`。
- 验证义务与证据：**Missing**。
- 决策依据：D6、D8，不变量 7、8。

## Executor Processes Are Bounded And Leave Nothing Behind

- 风险：Git 子进程挂起或输出过量拖住执行器；大仓库占满任务目录；超时后遗留进程和目录。
- 前置状态：大仓库、慢速 fixture、`docker pause` 的 fixture。
- 触发：任务超时；输出超过缓冲上限；任务目录超过大小上限；执行器容量已满。
- 必须成立：超时向整个进程组发 `SIGKILL`；输出有界且被截断时结果不作为事实采用；超限报告 `too_large` 且没有新 ref；容量已满按 `executor_busy` 拒绝；任务结束后任务目录与材料目录都不存在。
- 禁止结果：遗留 Git 进程；截断的 stdout 被当作结果；超限后留下部分 ref。
- 验证义务与证据：**Missing**。
- 决策依据：D1、D4，不变量 2。
