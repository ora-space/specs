---
status: approved
---

# Cloud 持久结算确定性派发拒绝，并让不可推进的 Operation 让出领取机会

Cloud 对尚未登记新执行的 Workspace 操作重新核验派发条件：确定性拒绝进入 `failed`，保存安全原因并
清空重试时间；暂时不可用保留原操作并延后。拒绝一次 RPC 不等于结算操作，Controller 也不能从一个
通用 `ABORTED` 推断业务失败。领取排序同时记住已经给予的机会，避免一个一直失败的旧操作挡住其他项目。

本文为 `approved`。owner 已按原文采纳下面的 D1–D4，Cloud 的另行实现（领取事务内结算与
`last_claimed_at` 轮转）已与本文对齐，因此状态不再停留在 `proposed`；对应的验证义务见
[确定性领取拒绝与轮转](../../../test-cases/cloud/operation/deterministic-claim-refusals.md)。
范围只含 Cloud 的 Operation 领取、派发前拒绝及 Controller 的对应行为。
Agent 工作项的持久延后、退役和调度版本围栏不在本文内，尚未批准，也不是本文的落地前置。
PostgreSQL 操作、执行、effect、资源与提交身份必须保留；本次不改变 Node 真实结果或自动放弃未知外部执行。

## 背景

`ClaimOperation` 能将一个操作置为 `running`，但随后的仓库访问守卫可能拒绝 `RecordDispatch`。该 RPC 的
事务回滚后，原操作仍是 `running`，没有持久错误或下次领取时间。下一次按创建时间领取又得到它，甚至
其他项目的正常操作也无法推进。单纯把 queued 排在 running 前面只能让后继获得第一次领取，不能让它
完成后续步骤。

另一个必须区分的事实是：Workspace clone 的已知失败已有最多三次的持久预算。派发前拒绝没有产生
`clone_executions`，因此永远消耗不到这三次，不能靠 clone 结果重试规则解决本问题。

## 继承与修改

| 前序约定 | 本文处理 |
|---|---|
| [Operation 根决策](0-workspace-runtime-follows-desktop-node.md) D1/D2：步骤、派发前登记、已知 clone 失败三次预算 | 继承；增加未登记的确定性拒绝分支，不重新定义 clone 结果分类 |
| [安全停止](20260927-safe-stop-and-administrative-force-stop.md)：未知执行不证明空闲或终止 | 继承；派发拒绝不能取消已有执行、清空账本或伪造删除 |
| [plugin 步骤与运行 Workspace](20260928-plugin-step-and-run-workspace-release.md) D4：Cloud 创建/删除运行 Workspace，业务以同事务钩子结算 | 继承；拒绝运行 Workspace 创建时复用失败钩子 |
| [Workspace 操作契约](../controller-integration/20260926-workspace-operations-and-controller-reported-nodes.md) D1：领取递增 version，写入受 epoch/version 保护 | 继承；候选选择增加持久的最近领取顺序，并可在同一领取事务中结算确定性拒绝 |
| 同契约 D3：真实 clone 结果接管先于阶段推进 | 继承；派发前拒绝不生成执行结果或收据 |
| 通用 RPC 冲突只说明该调用被拒绝 | 继承；新增业务结算必须以持久 Operation 为依据 |

## 决策

### D1：分类依据由 Cloud 持有，未知不归入确定性拒绝

“确定性”表示当前冻结请求在已核验的条件下明确不允许执行，不表示部署或授权永远不能改变。条件变化后
是否重新请求按原公开生命周期处理，不能后台重开已失败的操作。

| 证据 | 处理 | 不能据此推导的结论 |
|---|---|---|
| Cloud 确认请求引用的仓库凭据不存在/不可用、已冻结或与仓库范围不匹配 | 确定性拒绝，持久保留对应安全原因 | 不删除历史凭据引用或改为匿名 Git |
| Cloud 明确关闭或尚未提供该凭据类型的执行边界 | 确定性拒绝，`repository_credential_executor_unavailable` | 不把“已配置提供者暂时不可达”也解释成执行边界不存在 |
| 已配置提供者、Node 或 Substrate 暂时不可达，连接/容量/本地调度门暂不可用 | 保留原操作，按有限原因延后 | 不产生 Node 失败事实，不更换未知执行的身份 |
| 已登记执行或已计划外部 effect 的结果未知 | 按原身份查询/接管，遵循已有 blocked 与恢复规则 | 不以未收到回复断言未执行 |
| epoch/version 陈旧、提交冲突、输入与冻结值不符、协议形状不合法 | 拒绝当前调用，重新取得权威快照或报告部署错误 | 不把 Controller 的错误请求变成用户操作的永久失败 |

初期确定性拒绝集合仅包括上表的 Cloud 仓库凭据守卫。扩展到新的原因前必须说明权威证据与所属生命周期，
不能按错误字符串或所有 HTTP 409/gRPC ABORTED 批量判失败。数据库读取失败也不是“凭据不存在”。

Node 报告的 `clone_failed` 继续遵循根 D2：真实已知失败最多三次、5/10 秒退避、未知 blocked。认证失败与
网络失败的进一步细分另行评审，不借本 ADR 把所有失败改成零重试，也不取消现有上限。

### D2：未登记的确定性拒绝在同一事务结算

Cloud 在领取候选、准备交回其 clone 步骤前核验 D1；`RecordDispatch` 仍再次核验，防止领取后条件变化。
领取期间发现拒绝时，在持有现有事务锁并重新读取操作、Workspace、目标与执行登记后：

1. 只处理当前可推进、处于 clone 步骤的 create_project/create_workspace，且没有未结算 clone 或其他
   与本次操作相关的未知 effect/执行。已有成功 clone 先走原推进路径，不能因之后凭据冻结而改写成功事实。
2. 将 Operation 原子置为 `failed`、`error_code` 为确定性原因、`retry_at=NULL`，递增 version。关闭该目标
   的新执行准入，Workspace 为 `unavailable`；保留原阶段、请求、外部 effect 与所有资源身份。
3. 归还该操作的运行时维护控制，释放 Project 操作槽。主 Workspace 保留供核查；运行 Workspace 在同一
   事务调用 `runWorkspaceSettled(failed)`，沿已存在的失败/释放路径收尾。钩子或清理声明失败则全部回滚。
4. 只有事务提交成功后才返回结果/发出提示。不能更新后再抛出导致事务回滚的业务 Fault，宣称“已结算”。

`ClaimOperation` 在结算一个拒绝候选后继续检查下一个，单事务最多处理 16 个候选；达到上限可返回空，
下一次周期继续。这个界限限制事务时间，不代表队列已经永久为空；周期领取仍是正确性兜底。

若领取后才发生确定性拒绝，当前 `RecordDispatch` 保持拒绝且无新执行；Controller 回到下一次权威领取，
由同一核验/结算规则处理。不能靠解析一个 RPC 错误再直接调用“强制失败”接口，绕过并发和证据检查。
领取与登记之间恰有一次拒绝并不保证零等待，但不再永久占住队头。

### D3：给其他 Operation 可持续的推进机会

每次真正返回一个 Operation 时，Cloud 以数据库时间写入专用的 `last_claimed_at`。候选仍限于 queued、
到期的 retry_wait 与 running；blocked 不自动恢复，终态不参与。候选按
`COALESCE(last_claimed_at, created_at), created_at, id` 升序选择。

该字段表示领取机会，不是业务进度、失败次数、租约或超时起点。不能复用会被其他写入刷新、语义混杂的
`updated_at`。Controller 每次读取新 version；旧快照不能推进新的领取周期。重复 Claim 的响应丢失最多
消耗一次机会，不能更改冻结请求或复制副作用。

这提供有限候选集合下的轮转机会，尤其覆盖“未识别拒绝导致一轮直接退出”的保守路径；不承诺租户配额、
多 worker 公平性或长外部调用期间的并行推进。确定性拒绝仍必须结算，不能用轮转代替 D2。

暂时失败沿 `DeferOperation` 持久保存 retry_wait/blocked、有限原因和未来 retry_at；未到期的项不参与
轮转。控制连接整体不可用时停止本轮、按原租约协议恢复，不把所有操作都写成失败。

### D4：错误、恢复与资源边界

- 公开 Operation 继续通过现有 `state`、`errorCode`、`retryAt` 显示结果，无新增凭据正文、地址或日志细节。
- 原操作终态保留，公开 retry 不复活 failed；修正授权/部署后新请求遵循现有 Project/Workspace 约束。
- `GetDispatch`、真实结果与收据接管保留；释放 Project 槽不允许另一个操作删除仍有未知执行的资源。
- Agent 创建 blocked 的有界放弃属于独立生命周期决策。本 ADR 不给 provisioning 套用 Agent ready 之后
  的启动截止，也不新增未知 effect 的终止或物理回收授权。

## 不变量

1. RPC 拒绝、Operation 业务终态、Node 真实终态和物理释放分别有独立证据。
2. 确定性拒绝的 Operation 失败、错误码、维护控制归还及运行 Workspace 失败钩子共同提交或共同回滚。
3. 没有外部未知结果时才采用未登记拒绝结算；已登记/已计划身份的恢复义务不被擦除。
4. 临时失败、领取轮转、Controller 重启均不重置业务截止或 clone 持久次数。
5. 一条拒绝不会使其他项目永久失去领取机会；终态和未到期操作不被重新领取。

## 为什么不是这些替代方案

| 方案 | 取舍 |
|---|---|
| 所有 409/ABORTED 都改成 failed | 易实现，但会把陈旧快照、输入冲突和未知部署问题错误归给用户 |
| 只把 queued 排在 running 前面 | 后继能被领一次，进入 running 后仍会被旧项挡住 |
| 只按 updated_at 排序 | 可缓解阻塞，但心跳或无关写入会改变机会，不能解释公平性或结算 |
| 在 Controller 维护排除列表 | 无需数据库字段，但重启丢失、多个进程看法不一致，Cloud 仍显示 running |
| 在派发拒绝后无条件物理删除 Workspace | 释放快，但已有 effect/未知执行可能仍持有存储，破坏安全停止契约 |

## 兼容性、风险与落地

迁移只增加 nullable `last_claimed_at` 与合适索引，已有迁移不改写。存量 null 以 created_at 参与首次轮转，
不伪造过去的领取时间。旧 running 操作不批量标失败；下一次在 Cloud 当前状态下逐项核验。失败结算后
回滚程序版本不能重开终态，也不能删除留存的外部身份。

落地顺序：D1–D4 的条文按原文获批，Cloud 已实现领取事务内结算与 `last_claimed_at` 轮转，状态转为
`approved` 并登记[核心用例](../../../test-cases/cloud/operation/deterministic-claim-refusals.md)；
实现合并后再由落地运行把本 ADR 改为 `implemented`。之后固定 Cloud/Controller/Node 版本完成实际联合
验收。
核心用例逐项覆盖凭据守卫拒绝后其他项目推进、失败钩子回滚、旧版本/旧 epoch、领取后授权变化、未知执行
保护、重启与 clone 次数保持。实际 Rust Controller 的证据不能由 Go 模拟 Controller 的成功替代。

## 本决策未解决的问题

- Node clone 细分原因与凭据提供者实现；本次只明确已有 Cloud 守卫与已知结果之间的界限。
- 多 Controller 分区、租户配额、并行推进长操作与其吞吐 SLO；本次保留单全局租约。
- 未知外部结果的有界放弃与物理回收；继续由相应生命周期与残留资源决策承担。
