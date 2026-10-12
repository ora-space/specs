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
