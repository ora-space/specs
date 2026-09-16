---
status: proposed
---

# 每个逻辑 Node 使用独立宿主，每个 Scope 使用独立 Guardian

运行身份与生命周期独立于 Node 实例。每个逻辑 Node 配置一个独立运行宿主，每个活跃
`ProcessScope` 配置一个 guardian：宿主协调持久意图与控制权，guardian 持有进程、I/O 和运行事实。
宿主重启可以恢复管理；guardian 确认失效后关闭对应 scope，不接管其原活进程。

本根决策为 `proposed`，承接已讨论确认但未实现、未完成平台验证的方向。适用于新进程体系；
保留现有 `ora-process`、`ora-reaper`，允许重构 Git／插件接口，不增加旧 API 兼容包装。
文件系统布局仍是硬兼容边界，具体并存与迁移由[升级决策](../upgrade/0-version-coexistence-and-safe-migration.md)负责。

## 背景与约束

现有 `crates/process/src/reaper.rs` 使用进程级 reaper 状态和父进程存活管道；运行管理依附于
Desktop 进程。三面架构需要在 Node 更替、宿主故障和通信恢复时继续解释已经接受的运行责任。
一次 RPC 成功不能承担完整生命周期，需要分别保存意图、事实与协调进度。

## 决策

### D1：按权威划分运行层和业务层

```text
Controller：授权逻辑 Node 的实例更替
  └─ Node：Execution、资源协调、Git 对账、插件协议
      └─ process-client：Session / Scope / Run
          └─ 运行宿主：授权、控制权、意图、查询投影、恢复协调
              ├─ Scope A guardian：Run A1、Run A2 及其受管后代
              └─ Scope B guardian：Run B1 及其受管后代
```

Controller 授权不证明旧进程退出；宿主未收到回复不证明 guardian 未执行；guardian 的退出码
不证明业务目标完成。Node 持久记录本地执行关联，业务权威状态仍遵循
[Controller–Node Protocol](../../protocol/0-controller-node-protocol.md) 的部署边界。

一个业务 Execution 可以包含多个 Run。Scope 按共同控制、失联处置与收尾责任划分，不固定等于
Node、worktree 路径或 Execution。不同 scope 仍可能争用同一资源，由
[业务接入决策](../integration/0-shared-process-entry-and-resource-handoff.md)处理。

### D2：以独立 Guardian 承担存续责任

Guardian 持续纳管、读写 I/O、记录事实和收尾；宿主集中协调并汇合事实。共用代码不能变成
共用进程、数据库或内存权威。Scope 可以声明宿主失效时终止，但检测与执行必须在宿主之外。
每 scope 一个 guardian 带来进程和存储成本，换取明确的故障与收尾边界；本版不做机器级统一宿主。

### D3：三个 Library Crate 与两个薄 App

| 位置／名称 | 责任与依赖 |
|---|---|
| `crates/process-protocol` / `ora-process-protocol` | 身份、策略、状态、消息、版本和编码；无 Tokio、数据库、Git 或 OS 进程操作 |
| `crates/process-client` / `ora-process-client` | 远程句柄和连接管理；依赖 protocol，不依赖 runtime |
| `crates/process-runtime` / `ora-process-runtime` | 私有 host、guardian、platform 模块，各自关联持久化和测试 |
| `apps/ora-process-host`、`apps/ora-process-guardian` | 参数、日志与启动入口 |

Platform seam 区分 Linux、Windows、macOS 和 fake，不以 Unix 抹平保证差异。优先静态分发、
私有实现和显式导出；与 Ora 领域、transport、runtime 无关的通用能力归 `ora-utils`。

### D4：范围服务于可恢复运行

纳入稳定身份、幂等启动、控制权交接、宿主恢复、平台纳管、双向有界 I/O、查询订阅、结果接管、
责任回收及 Git／插件接入。不纳入运行数量准入、并发配额、CPU／内存／进程数预算、机器级管理、
guardian 崩溃后的活进程接管、Run 内自动业务重试、无限事件历史或恶意程序安全沙盒。
有界队列、输出政策和磁盘故障处理属于正确性，不能据此重新引入资源配额模型。

## 不变量

1. 运行身份跨 Node 实例和宿主实例保持稳定，实例更替不创建新的业务尝试。
2. 意图、运行事实、业务解释各有唯一所有者，通过显式协议组合。
3. 旧进程体系与新体系并存时，同一业务尝试只交给其中一套执行。

## 为什么不是这些替代方案

| 方案 | 取舍 |
|---|---|
| 继续依附 Node 内存管理 | 进程少，但 Node 崩溃后无法独立承担已经接受的责任 |
| 一个宿主直接持有全部进程和 I/O | 装配简单，但宿主升级和崩溃同时破坏所有运行存续 |
| 每 Run 一个 guardian | 隔离更细，但共同交接与收尾仍需 Scope 协调；当前按 Scope 隔离已满足责任边界 |
| 机器级统一管理器 | 可共享管理成本，但超出逻辑 Node 的授权、部署和恢复边界 |

## 落地与验收

先按[实施顺序](../README.md#实施顺序)证明平台启动与收尾，再接入持久恢复和控制权。
验证宿主强杀不丢失 guardian 已承担的责任、单个 scope 故障不误伤无冲突工作，并检查 crate 依赖
和真实业务入口。物理结构检查不能替代跨进程行为证据。

## 本决策未解决的问题

各平台机制、授权协议、wire schema、存储布局与发布兼容窗口分别由本目录其他根 ADR 约束；
具体类型与默认参数在对应决策评审时冻结。本 ADR 不授权新增已排除的运行配额或安全沙盒。
