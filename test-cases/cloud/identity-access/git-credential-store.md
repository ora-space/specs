# Git 凭据存储：浏览器直接提交、引用与能力、一次性兑换

依据已批准的 [Git 凭据存储决策](../../../decisions/cloud/identity-access/20261016-git-credentials-in-independent-credential-store.md)。
下文的 D 编号与不变量编号都指这篇 ADR。Credential Store、提交授权与能力验证尚未合入 cloud 的 main，下列义务都是 **Missing**。

## Git Tokens Reach Only The Browser And The Credential Store

- 风险：token 经 Cloud 或 Gateway 转交，出现在它们的请求处理、日志或响应中。
- 前置状态：边缘反向代理把浏览器入口直接转发到 Credential Store；带 canary token。
- 触发：浏览器三步提交；Cloud 收到带 `token` 字段的请求；提交、兑换、同步、发布、撤销全程。
- 必须成立：token 只出现在发往浏览器入口的请求中；Cloud 拒绝带 `token` 的请求（`400`，不创建引用）；浏览器入口只有写入新版本的路由，响应只有版本号。
- 禁止结果：Cloud、Gateway 的日志、数据库导出或响应中出现 token；浏览器入口存在读出明文的路径。
- 验证义务与证据：**Missing** — 需要全程秘密扫描，带阳性对照。
- 决策依据：D3，不变量 1、4。

## Submission And Execution Authorizations Are Distinct And Single-Use

- 风险：提交授权被重放或改动；执行授权被当作提交授权使用，反之亦然；并发轮换覆盖版本。
- 前置状态：Cloud 用不同的密钥和用途签发提交授权与执行授权。
- 触发：重放、过期、改动提交授权；用执行授权调用浏览器入口；用提交授权兑换；两个轮换并发写入同一 `expectedStoreVersion`。
- 必须成立：Credential Store 以 `submissionId` 独占认领、以授权 ID 独占兑换，兑换记录落盘后才解密；版本不符时拒绝；两类授权互相不能替代。
- 禁止结果：同一授权使用两次；跨用途的授权被接受；并发轮换都成功。
- 验证义务与证据：**Missing**。
- 决策依据：D2、D3，不变量 2、3。

## A Reference Becomes Stored Only After Cloud Confirms The Version

- 风险：Cloud 相信浏览器报告的版本，把引用置为 `stored`；浏览器没有回来确认时引用永远停在 `awaiting_secret`。
- 前置状态：`awaiting_secret` 的引用。
- 触发：浏览器谎报版本后确认；浏览器提交后不确认。
- 必须成立：Cloud 经管理接口查询当前版本与 `submissionId`，一致才置 `stored`；授权过期后定时任务补上确认或标为已放弃；非 `stored` 的引用不能签发兑换授权。
- 禁止结果：版本不一致仍置 `stored`；`awaiting_secret` 或 `revoked` 的引用被兑换。
- 验证义务与证据：**Missing**。
- 决策依据：D1、D3，不变量 5、10。

## Capabilities Hold For One Store Version And Are Proven, Not Declared

- 风险：旧版本的能力延续到新 token；推送能力由用户自述，错误到发布时才暴露；验证推送能力时写入了外部仓库。
- 前置状态：可写 token、只读 token；`push-git-fixture`。
- 触发：首次同步成功；`publish` 验证；轮换 token；同步以认证失败结束。
- 必须成立：`read` 只由一次成功的同步证明；`publish` 只由一次成功的 receive-pack 广告请求证明，只读 token 得到 `push_denied`；轮换后能力集合为空；能力证据记录 `storeVersion` 与来源。
- 禁止结果：新版本继承旧能力；`publish` 验证向 fixture 发出更新命令；只读 token 能创建发布。
- 验证义务与证据：**Missing**。
- 决策依据：D1、D4，不变量 6、7。

## Administrators Cannot Use Another Member's Personal Credential

- 风险：租户管理员以他人的个人凭据绑定 Project 或批准发布，以离开者的外部身份写入。
- 前置状态：个人凭据与团队凭据；成员离开后重新加入。
- 触发：管理员绑定他人的个人凭据；管理员批准用它发布；管理员撤销它；所有者离开。
- 必须成立：绑定与批准被拒绝；撤销成功；所有者离开后个人凭据冻结，重新加入仍冻结。
- 禁止结果：管理员借他人的个人凭据完成任何读取绑定或发布。
- 验证义务与证据：**Missing**。
- 决策依据：D5，不变量 8。

## Revocation Stops New Authorizations And Destroys Ciphertext

- 风险：撤销后仍签发授权或仍能兑换；管理接口调用失败后撤销停在一半。
- 前置状态：`stored` 的引用和已签发、未兑换的授权。
- 触发：撤销；管理接口暂时失败；撤销后同步与发布。
- 必须成立：Cloud 先在事务内置 `revoked`、再调用管理接口撤销，失败时重试直到成功；撤销后不再签发授权，Credential Store 撤销完成后不能兑换，所有版本的密文已销毁；撤销幂等。
- 禁止结果：撤销后签发新授权；Credential Store 撤销后仍能兑换；密文残留。
- 验证义务与证据：**Missing**。
- 决策依据：D2，不变量 9。
