---
status: proposed
---

# Windows 创建时加入 Job，并独立核实清空结果

Windows 强纳管在创建进程时使用 `PROC_THREAD_ATTRIBUTE_JOB_LIST` 加入 Job，禁止后代
breakaway；guardian 不在其清理的工作负载 Job 内。句柄归属与退出证据共同决定清理结果。

本根决策为 `proposed`，承接[保证契约](../guarantees/0-explicit-containment-and-cleanup-evidence.md)，
尚未证明 Job 层级和故障恢复链。它不自动接管旧 reaper 的进程或承诺所有外层 Job 配置均可运行。

## 背景与约束

Job 可以约束进程组，但创建与分配窗口、嵌套 Job、breakaway 和管理句柄都会改变实际保证。
Guardian 死亡不必然关闭最后一个 Job 句柄，因此不能直接等同于全体工作负载退出。

## 决策

### D1：创建时加入且禁止 Breakaway

通过创建属性加入 Job，业务代码执行前完成归属。检查部署环境的外层 Job 和嵌套规则，禁止
允许后代 breakaway 的配置；无法满足时按请求拒绝或启动前降级。
参见 [创建属性文档](https://learn.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-updateprocthreadattribute)
和 [Microsoft 创建时加入 Job 的说明](https://devblogs.microsoft.com/oldnewthing/20230209-00/?p=107812)。

### D2：管理句柄必须有完整责任归属

业务进程不得继承管理句柄。`KILL_ON_JOB_CLOSE` 依赖最后一个句柄关闭；宿主或恢复程序额外
持有句柄时必须承担相应清理责任，不能假定 guardian 退出就是最后句柄关闭。
Guardian 保持在工作负载 Job 外，避免清理自身导致事实无法保存。
参见 [Job Objects](https://learn.microsoft.com/en-us/windows/win32/procthread/job-objects)。

### D3：清理后独立核实活动进程归零

先封闭启动入口，再终止并核实活动进程归零。完成端口通知只唤醒检查，不能作为唯一退出证据。
单 Run 清理需对应明确 Job 层级和归属，不能通过清空 scope 来伪装 Run 隔离。
Guardian 死亡且 Job 已消失时，仍需可靠证据链；证据不足保持未知。

## 不变量

1. 强模式进程执行前进入目标 Job，声明范围内后代不能 breakaway。
2. 管理句柄不泄漏给业务进程，每个额外持有者承担清理责任。
3. 通知、guardian 死亡或 Job 名称消失均不单独替代清空证据。

## 为什么不是这些替代方案

| 方案 | 取舍 |
|---|---|
| 进程启动后再加入 Job | 易包装旧入口，但引入启动和纳管之间的窗口 |
| 只依赖 `KILL_ON_JOB_CLOSE` | 自动清理简单，但额外句柄改变最后关闭时点 |
| 只等完成端口通知 | 事件驱动方便，但不足以独立证明所需退出事实 |

## 落地与验收

先验证创建属性与嵌套 Job，再验证句柄继承、额外持有、单 Run 清理及 guardian 外部强杀。
覆盖 Job 已消失和通知缺失的恢复路径，分别记录实际 Windows 与部署配置，不从 Linux 测试推导通过。

## 本决策未解决的问题

Scope／Run Job 层级、完整句柄所有权及 Job 消失后的证据链必须在实现前冻结并通过真实平台测试。
不满足这些条件不能标记强模式已实现。
