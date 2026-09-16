# Node 进程运行决策

`process/` 只做导航，ADR 位于叶子目录；平台部分再分公共保证与三个 OS，避免将全体系决策误排成同一主题的时间演进链。

原讨论已确认架构方向、范围和实施顺序，尚未实施或完成平台验证。本次转入 specs 后统一保持
`proposed`，供正式评审；不因整理文档自动标记 `approved` 或 `implemented`。每个叶子目录后续
按 `YYYYMMDD-...md` 追加 ADR，并显式继承或修改该目录的前序决策。

## 阅读顺序与目录边界

| 目录／根 ADR | 负责的决策 |
|---|---|
| [architecture](architecture/0-node-host-and-scope-guardians.md) | 逻辑 Node 宿主、Scope guardian、权威分工、范围与 crate 结构 |
| [control](control/0-authorized-generations-and-leases.md) | 稳定身份、Controller 授权、代次、租约、撤权 |
| [lifecycle](lifecycle/0-durable-run-attempts-and-scope-closure.md) | 一次尝试、幂等启动、未知事实、Run／Scope 结束与停止 |
| [recovery](recovery/0-single-authority-and-failure-recovery.md) | 故障层级、单实例、独占恢复和逐 scope 放行 |
| [containment/guarantees](containment/guarantees/0-explicit-containment-and-cleanup-evidence.md) | 请求与实际保证、启动前降级、强／尽力清理证据 |
| [containment/linux](containment/linux/0-cgroup-v2-containment.md) | cgroup v2、委派、受限 Linux／gVisor 能力证明 |
| [containment/windows](containment/windows/0-creation-time-job-containment.md) | 创建时加入 Job、句柄归属和清空核实 |
| [containment/macos](containment/macos/0-native-best-effort-containment.md) | 原生尽力纳管、跟踪缺口与身份复用 |
| [io](io/0-bounded-output-and-fenced-input.md) | 输出政策、续读、输入交付与业务会话边界 |
| [protocol](protocol/0-independent-local-streams-and-state-recovery.md) | 本地三流、编码、快照订阅与结果接管 |
| [persistence](persistence/0-independent-journals-and-responsibility-retirement.md) | 独立 SQLite、耐久顺序、事实汇合、退休与回收 |
| [upgrade](upgrade/0-version-coexistence-and-safe-migration.md) | 三个兼容边界、版本共存、目录并存与安全迁移 |
| [integration](integration/0-shared-process-entry-and-resource-handoff.md) | Client 句柄、Git／插件入口、资源交接与业务对账 |

先阅读 architecture、control、lifecycle，理解身份与责任，再按主题进入其他 ADR。平台实现依赖
公共保证；资源放行同时依赖控制权、生命周期、清理证据和业务对账。本地 process 协议与上层
[Controller–Node Protocol](../protocol/0-controller-node-protocol.md)属于不同边界。
术语统一见[领域词汇表](../../../domain/glossary.md#node-process)。

## 原方案覆盖映射

编号沿用原方案第 2 节，用于核对拆分完整性，不充当本目录 ADR 编号。

| 原决策编号 | 归属 |
|---|---|
| 1、13、14、25、30 | architecture；其中失权策略见 control，第 25 项明确不采纳运行配额 |
| 3、5、20 | control |
| 2、4、7、19、23、24、26 | lifecycle；资源冲突另见 integration |
| 6、15、28 | recovery |
| 12、18 | containment/guarantees；macOS、受限 Linux 的机制分别归平台 ADR |
| 16 | containment/linux |
| 17 | containment/windows |
| 9、10 | io |
| 11、34 | protocol |
| 21、22、33 | persistence |
| 27 | upgrade |
| 8、29、31 | integration |
| 32、35 | 各 ADR 的验收约束及下述实施顺序、故障验证 |

原第 15 节 REVIEW 的修正已融入相应决策：fencing 与资源交接属于根契约；各平台分别证明保证；
宿主崩溃由 guardian 存续覆盖；Execution／Scope／Run 分离；断线、停止、退出分开；Git／插件
统一入口；保留接管与多观察者基础，但不引入无限历史或资源配额。原第 16 节待冻结项分配到各篇
“本决策未解决的问题”，没有因迁移变成已验证事实。

## 实施顺序

| 阶段 | 闭环 | 完成条件与涉及 ADR |
|---|---|---|
| 1 | 平台启动与收尾 | containment 各篇＋lifecycle／recovery；真实创建、后代处理、停止、guardian 强杀，分别证明强／尽力保证 |
| 2 | 持久启动与宿主恢复 | lifecycle、persistence、recovery；回复丢失不重复启动，关键窗口强杀不伪造未执行 |
| 3 | 控制权与资源交接 | control、integration；旧 Node 恢复不能夺权，迟到输入失效，冲突资源按政策放行 |
| 4 | 完整 I/O 与观察 | io、protocol、persistence；输出政策、输入交付、快照订阅、结果接管，拥塞时控制仍推进 |
| 5 | Git 与插件接入 | integration；真实入口、持久关联、业务对账与会话恢复，消除已迁移调用方的启动旁路 |
| 6 | 升级与退休 | upgrade、persistence；版本共存、迁移、回收和旧请求拒绝，完成全故障矩阵 |

每个切片包含必要协议、状态、持久化和测试，降低同时改动所有边界的风险。阶段 1 的最小入口
不发展成另一套生产实现；阶段 4 才补齐完整 I/O 不代表此前可以忽略管道读取或撤权。
各阶段保持原身份去重、责任不自动释放和实际保证可观察；行为变化只在相应闭环验收并迁移调用方后启用。

## 故障验证

以下是 proposed 方案的验收约束，尚不是已批准的核心用例或测试通过记录。

| 故障／并发点 | 必须保持的承诺 | 主要归属 |
|---|---|---|
| 接受启动后回复丢失 | 同 RunId 恢复原尝试，无第二次启动 | lifecycle |
| 落盘、纳管、启动、回复之间强杀 | 未知不当作未执行，核实或阻塞／收尾 | lifecycle、persistence |
| 旧 Node 暂停、接管后恢复 | 旧变更、续约和未交付输入失效，已交付命令按业务事实恢复 | control、io |
| Node 强杀后立即重启 | 冲突工作遵守恢复政策，无冲突工作独立推进 | recovery、integration |
| 宿主强杀或升级 | guardian 保持所选存续政策，新宿主不重复启动 | recovery、upgrade |
| Guardian 强杀或暂时断线 | 确认死亡才关闭；不明时不补启，结果与清理分别报告 | recovery |
| 直接退出、后台化或新会话 | 按策略管理后代，不提前完成或误报强保证 | lifecycle、containment |
| 单 Run 终止与其他 Run 并行 | 不隐式杀整个 scope，不提前释放共同资源 | lifecycle、integration |
| 启动与关闭／撤权并发 | 拒绝或纳入收尾，无漏管窗口 | lifecycle、control |
| 输出积压、I/O／Events 独立断线 | 政策生效、缺口可见，控制和租约不受输出阻塞 | io、protocol |
| stdin 写入后记录／回复丢失 | 暴露交付未知，不自动重发业务命令 | io、persistence |
| 快照与订阅并发、游标回收 | 不静默漏变化，显式重新同步 | protocol |
| 完成记录／确认丢失 | 接管幂等，未接管结果不被普通轮转删除 | protocol、persistence |
| 退休后旧启动到达 | 拒绝旧身份，不隐式创建或重执行 | persistence |
| 强能力缺失、受限 Linux／gVisor | 拒绝或启动前降级，保证与恢复政策可查询 | containment |
| 清理失败、磁盘满、权限错误、记录损坏 | 不伪造完成，责任、错误及阻塞可观察 | containment、persistence |
| 新旧协议／数据不兼容 | 不误解语义、不覆盖旧文件、不回退授权与去重事实 | upgrade |
| 并发宿主启动、guardian 启动回复丢失 | 单一有效权威，取得锁不等于旧运行退出 | recovery |
| Git 已改文件但结果未保存 | 不重跑原尝试，业务对账后决定新尝试 | integration |
| 插件本地退出但外部任务继续 | 本地静止不解释为所有外部副作用结束 | integration、containment/guarantees |

纯状态机测试控制时钟、授权、存储和平台事实；平台测试启动真实进程及后代；业务测试跨独立进程、
持久存储和真实 Git／测试插件验证最终事实。使用显式同步点与外部强杀，不用 sleep 碰概率、正常
Drop、正常退出或单进程 panic 代替。既证明不错误放行，也证明条件恢复后不会永久卡住；基础矩阵
稳定后再增加顺序探索与随机故障。

证据记录保证、场景、测试入口、实际平台／能力、结果和未验证范围。强／尽力、macOS、受限
Linux／gVisor 与版本组合分别验收，不从其他环境通过推导支持。测试遵循 pretty_assertions、
TRACE subscriber 隔离及不修改进程环境等仓库规范。

## 评审与落地门槛

对应[测试目录](../../../test-cases/node/process/README.md)镜像本目录，目前只有导航与占位。
ADR 评审通过后先按反馈更新正文和 `approved` 状态，再维护核心用例与证据；实现验证后同步
最终约束并标记 `implemented`。本次仅迁移决策，不编写核心用例，不宣称完成运行验证。
实施时运行相应格式、lint 和测试，仓库级代码变更完成前运行 `task test`；涉及 `docs/` 时同步中英版本。
