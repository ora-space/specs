# Node 核心测试用例

本目录与 `specs/decisions/node/` 镜像。Node 各能力的根决策进入 `approved` 后，再在对应子目录
登记需要长期维护的跨进程、外部副作用、ownership、重试和恢复用例。

用户托管的仓库凭据在 Node 侧的承诺于 2026-10-10 随
[受控 Node 的仓库凭据执行使用独立 OS 身份](../../decisions/node/repository/20261010-controlled-node-git-authentication-isolation.md)
进入 `approved`，其核心用例登记在
[repository/git-credential-execution-isolation.md](repository/git-credential-execution-isolation.md)：
M／W／C 身份互异与不可冒用、Agent 合法 IPC 保持可用而能力不可提升、匿名公开仓库 Clone 的行为逐位
不变、按"是否需要凭据执行"决定启动拒绝，以及凭据物化的销毁与单次授权。**凭据执行身份 C 尚未建立，
因此这些用例的多数义务今天为 `Missing`**；Cloud 侧对应用例见
`specs/test-cases/cloud/identity-access/` 的四份凭据文件。

写方向的权限模型于 2026-10-12 随
[推送执行权限](../../decisions/node/repository/20261012-push-execution-authority.md)进入 `approved`
（A–D、F 批准，E 不批准：`push` 仍不在词汇表中），其核心用例登记在
[repository/push-execution-authority.md](repository/push-execution-authority.md)：执行者只能是部署固定的
执行器、授权绑定九项、ref 范围默认拒绝、秘密只经既有兑换路径并在执行结束销毁、以及**推送在词汇表扩展
之前不可被请求**。推送路径今天不存在，因此除最后一条外全部为 `Missing`。
