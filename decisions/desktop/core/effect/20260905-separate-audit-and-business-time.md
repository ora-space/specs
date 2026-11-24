---
status: superseded
---

# Effect 的行审计时间由写事务拥有，业务时间独立保存

> 已被取代：本决策由 [Skill 按存储路径以会话清单交给 Agent，取代 Effect 物化](../agent/session/skill/0-skill-catalog-from-storage-paths.md) 取代，原因：Effect 表由前向迁移删除，本文的审计时间与业务时间规则失去对象。当前系统承诺以取代者为准；取代者的实现合并之前，desktop 里仍在运行的 Effect 以本文为历史依据。Effect 的核心用例暂时保留，在取代者进入 `implemented` 时随 Effect 代码及其中的引用一并删除。

Effect repository 在取得写事务后采样注入时钟，同一事务共享该采样值；每行独立保留自己的时间下界。
源数据版本、Operation 阶段时间和调度截止时间各自保留业务语义，不再作为跨行审计时间传播。
本决策已实现并通过回归验证，覆盖 Effect 持久化与 worker；Local Skill 的版本、文件 journal 和目录布局不变。

## 继承与修改

继承 [Effect 系统基础](0-effect-system-foundation.md)。

| 约定 | 本决策 |
| --- | --- |
| generation、status version、fencing、ownership | 继承，不变；审计时间不授予权威 |
| journal-first、事务原子性、恢复隔离 | 继承，不变 |
| 调用方传入各行 updated_at | 修改：由写事务采样并按本行下界持久化 |
| Operation 用 updated_at 保存恢复检测时间 | 修改：独立 detected_at |
| Workspace trigger 复制历史时间创建 Scope | 修改：repository 同一事务显式创建、seed、唤醒 |

## D1：审计时间不能表示源数据版本或事务因果顺序

新行的 created_at 与 updated_at 使用写事务时间；更新不改写 created_at，updated_at 取当前行值、
写事务采样值和 CHECK 所要求业务时间下界的最大值。时间允许相等，时钟回拨不能阻止合法状态转换。
旧 Skill 回填不会把历史时间写入新建 Target；Scope 的更新时间也不能代表 Target 的时间下界。

生产 Effect 写入使用 Ora 已初始化的本地时钟，测试注入可控时钟。独立 migration CLI 需要在日志初始化
之前运行，因此保留原有系统时钟；迁移记录不承担 Effect 行审计时间的语义。

## D2：在阶段发生时记录业务时间

同一 Attempt 的 Operations 共享 prepared_at；apply 返回后记录 applied_at，完成验证和 readiness
后记录 finalized_at，恢复扫描记录 detected_at。阶段时间以前序阶段为下界。worker 不保留整批任务
的开始时间用于后续阶段或重试；重试从失败处理时刻计算。receipt 的 received_at 表示本地持久接收，
Condition 的 first_observed_at 保持稳定，last_observed_at 不倒退。

## D3：保留既有数据库和恢复证据

新增迁移 0010，不改写旧迁移。RecoveryRequired 行的 detected_at 从原 updated_at 回填；其他行为空。
数据库校验确保检测时间只属于 RecoveryRequired 且不早于 prepared_at。回滚先恢复旧字段承载的检测
时间，再删除新增字段，并恢复 Workspace trigger。既有身份、generation、request、journal、ownership
不重建；不试图恢复历史记录中已经丢失的实际写入时刻。

显式受控恢复以 exact planned 或 exact expected 证明关闭 Operation 时，`detected_at` 随阶段离开
RecoveryRequired 而清空，不能把它留在 Finalized / FinalizedWithoutEffect 行里。同一个关闭事务必须
把每个 Operation 的 identity、原始 `detected_at` 与关闭结果保存到 `recovery_finalized` audit 事件；
audit 的 `occurred_at` 表示关闭事务时间，不替代原检测时刻。审计写入失败则关闭整体回滚。不可变
journal 意图、prepared_at 和已知 applied_at 继续保留；历史检测证据由 audit 承载，阶段字段不承担
跨阶段的历史存档职责。

## 为什么不是这些替代方案

| 方案 | 否决理由 |
| --- | --- |
| 仅增加 MAX，保留调用方历史时间 | 能保护部分 CHECK，但审计时间来源仍不正确，容易遗漏跨行路径 |
| 删除 CHECK | 将非法时间顺序持久化，失去损坏检测能力 |
| 所有字段统一使用 SQL 当前时间 | Skill 版本和业务阶段语义会丢失，时钟回拨仍需保护 |
| 本次重构 Skill 版本号 | 涉及独立文件 journal 兼容义务，超出 Effect 改进范围 |

## 不变量

1. Effect 行审计时间不由旧业务对象或 worker snapshot 决定。
2. 每行 updated_at 不倒退且满足既有 CHECK；合法时钟回拨不会制造业务版本。
3. 状态转换权威始终来自现有版本、generation 和 claim。
4. Scope 初始化与 Workspace 写入共同提交或共同回滚。
5. 升级、回滚和再升级保留精确恢复证据。

## 落地与验收

迁移 0010、repository 写事务时钟、状态读取 metadata 和 reconciler 阶段采样已落地。历史数据回填、
交错提交、时钟回拨、恢复和迁移往返已有直接回归证据，见
[时间语义核心用例](../../../../test-cases/desktop/core/effect/time.md)。端到端测试验证 Skill 导入后收敛
到 Workspace，并在删除后移除产物。
