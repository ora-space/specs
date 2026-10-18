# Node 核心测试用例

本目录与 `specs/decisions/node/` 镜像。Node 各能力的根决策进入 `approved` 后，再在对应子目录
登记需要长期维护的跨进程、外部副作用、ownership、重试和恢复用例。

用户托管的仓库凭据在 Node 侧的承诺由
[受控 Node 的仓库凭据执行使用独立 OS 身份](../../decisions/node/repository/20261010-controlled-node-git-authentication-isolation.md)
承载。该篇与它的下游契约修订都是 `proposed`（`approved` 落款来自运行 issue `ora-space/circus#29` 的授权
记录，specs 的 PR 评审未走完，见 `specs/test-cases/cloud/identity-access/README.md` 的评审状态一节），
其核心用例登记在
[repository/git-credential-execution-isolation.md](repository/git-credential-execution-isolation.md)：
M／W／C 身份互异与不可冒用、Agent 合法 IPC 保持可用而能力不可提升、匿名公开仓库 Clone 的行为逐位
不变、按“是否需要凭据执行”决定启动拒绝，以及凭据物化的销毁与单次授权。**Node 内没有 C**：C 已按
[凭据执行器归属当前正式产品](../../decisions/node/repository/20261012-credential-executor-belongs-to-the-current-product.md)
（`proposed`）实现为当前正式产品的独立部署单元，因此原来针对“同一容器内三个 UID”的那些断言今天
**不存在**，同一容器内的三身份分离仍为 `Missing`；已有的直接证据在执行器一侧的边界上。Cloud 侧对应
用例见 `specs/test-cases/cloud/identity-access/` 的四份凭据文件。

写方向的权限模型由
[推送执行权限](../../decisions/node/repository/20261012-push-execution-authority.md)承载，同样是
`proposed`（A–D、F 同意，E 未同意），其核心用例登记在
[repository/push-execution-authority.md](repository/push-execution-authority.md)：执行者只能是部署固定的
执行器、授权绑定九项、ref 范围默认拒绝、秘密只经既有兑换路径并在执行结束销毁、以及**推送在词汇表扩展
之前不可被请求**。**2026-10-13 起写方向已有实现与实测**，该文件已按实测逐行重估：多数义务由 `Missing`
转为 `Covered`，两行仍是 `Partial`——中断后的明文要等下一次启动清扫，以及“真实私有仓库”那一格用的是
验收自建的受控远端而不是第三方私仓。按上一条规则，这些证据在承载它的 ADR 变为 `approved` 之前只算
“实现先行”证据。
