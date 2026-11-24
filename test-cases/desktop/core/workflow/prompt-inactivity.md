# 工作流 prompt 静默策略

以下用例依据[工作流有限与无限等待决策](../../../../decisions/desktop/core/workflow/20261123-workflow-agent-wait-limit.md)。冻结配置与会话隔离的基础规则见[静默策略](../../../../decisions/desktop/core/workflow/20261008-prompt-inactivity-policy.md)。下列测试链接固定到 [Desktop 合并提交 6489e9d1](https://github.com/ora-space/desktop/commit/6489e9d1cafc2a392f71332156425236cf50a83f)。

## A Silent Waiting Prompt Remains Cancellable And Session Local

**风险与前置状态**　同一连接承载显式 `unbounded` 工作流与默认策略的普通会话。长时间静默可能触发意外重发，或使其他会话也失去超时保护。

**触发与预期**　让两个回合静默跨过 30 分钟及普通会话的重试窗口。无限等待回合应保持单次发送，随后可正常完成。分别触发手动停止、释放 owning stream、工作流停止、provider 断连和宿主关闭，每条路径都须收敛并释放本轮资源。普通会话应独立执行其超时策略。禁止跨会话续期、静默触发节点重试或终止后遗留路由。

| 验证义务 | 证据与边界 | 状态 |
| --- | --- | --- |
| 显式 `unbounded` 长静默后完成或取消，同连接默认会话独立超时 | 同一连接的默认回合独立重试，无限等待回合跨过两小时仍可正常完成、取消并复用。这里的取消是 owner stream 的 `cancel_and_wait()`，不是用户停止端点。未直接贯通从已发布图到 provider 的两小时静默。 [并行会话](https://github.com/ora-space/desktop/blob/6489e9d1cafc2a392f71332156425236cf50a83f/crates/agent-runtime/src/prompt_policy_tests.rs#L534)、[取消及复用](https://github.com/ora-space/desktop/blob/6489e9d1cafc2a392f71332156425236cf50a83f/crates/agent-runtime/src/prompt_policy_tests.rs#L837) | Partial |
| 按各类终止事件清理显式无限等待回合及订阅者 | 释放 owning stream 后，断言已发出取消且排在其后的停止请求完成。释放宿主和 stream 后，断言已发出取消且 generation 任务退出。provider 断连路径断言 owner 收到错误，资源释放和路由摘除仍缺直接断言。三条路径均未建立 follower，订阅者终态仍缺直接证据。手动停止和工作流停止在 `unbounded` 下没有证据。 [释放 owning stream](https://github.com/ora-space/desktop/blob/6489e9d1cafc2a392f71332156425236cf50a83f/crates/agent-runtime/src/prompt_policy_tests.rs#L893)、[provider 断连](https://github.com/ora-space/desktop/blob/6489e9d1cafc2a392f71332156425236cf50a83f/crates/agent-runtime/src/prompt_policy_tests.rs#L1340)、[宿主关闭](https://github.com/ora-space/desktop/blob/6489e9d1cafc2a392f71332156425236cf50a83f/crates/agent-runtime/src/prompt_policy_tests.rs#L1367) | Partial |

**决策依据**　D1、D3。

## Every Workflow Turn Uses The Frozen Inactivity Policy

**风险与前置状态**　已有图、含新取值的图和已启动的运行并存。编辑草稿或升级 Desktop 时，后续发送可能误读草稿，非法值也可能被静默降级。

**触发与预期**　从保存、发布、导入和导出的图开始运行，随后编辑草稿并打开运行详情。首轮、追问、节点重试及 Loop/Iteration 的每次实际发送均采用运行的冻结字段。缺失或 `null` 使用默认策略，打开旧图不会补写字段。升级重启服务进程会结束在途回合，恢复后的下一次实际发送按恢复后运行快照的值解释（没有切换快照时即原冻结值）。禁止因读取、改无关字段或升级而自动选择无限等待。

| 验证义务 | 证据与边界 | 状态 |
| --- | --- | --- |
| 接受缺失、`null`、`timeout`、`wait`、`unbounded`，拒绝未知字符串和 `{"waitMinutes": N}` 并定位节点与字段 | 五种合法输入均解析为预期策略；非法字符串、分钟对象及其他类型的错误含节点和字段。桌面导入在预览前拒绝分钟对象。 [合法输入](https://github.com/ora-space/desktop/blob/6489e9d1cafc2a392f71332156425236cf50a83f/crates/application/src/workflow_run/engine/agent_config.rs#L511)、[非法输入](https://github.com/ora-space/desktop/blob/6489e9d1cafc2a392f71332156425236cf50a83f/crates/application/src/workflow_run/engine/agent_config.rs#L581)、[文件导入](https://github.com/ora-space/desktop/blob/6489e9d1cafc2a392f71332156425236cf50a83f/packages/app-shell/src/features/workflow-editor/workflow-transfer.test.ts#L146) | Covered |
| 各轮冻结、保存发布及导入导出保留策略 | 草稿修改后的人类追问和节点重试仍使用冻结的 `wait`。记录型 executor 逐轮收到 Loop 的 `Wait` 和 Iteration 的默认 `Timeout`。Iteration 的显式 `wait` 和复合节点实际发送到 provider 时的策略均缺直接断言。快照导入导出覆盖五种合法值。`unbounded` 在 backend 分发的首轮、追问、重试及 Loop/Iteration 各路径均无断言。 [人类追问](https://github.com/ora-space/desktop/blob/6489e9d1cafc2a392f71332156425236cf50a83f/crates/backend/src/workflow/run/interactive/inactivity_tests.rs#L19)、[节点重试](https://github.com/ora-space/desktop/blob/6489e9d1cafc2a392f71332156425236cf50a83f/crates/backend/src/workflow/run/executor/inactivity_tests.rs#L58)、[Loop/Iteration 逐轮分发](https://github.com/ora-space/desktop/blob/6489e9d1cafc2a392f71332156425236cf50a83f/crates/backend/src/workflow/run/mixed_scope_tests.rs#L48)、[快照往返](https://github.com/ora-space/desktop/blob/6489e9d1cafc2a392f71332156425236cf50a83f/packages/app-shell/src/features/workflow-editor/workflow-transfer.test.ts#L161) | Partial |
| 编辑器提供固定 30 分钟和明确标示需手动停止的无限等待。只读运行详情展示冻结值，有限等待显示 30 分钟 | 编辑器要求显式选择无限等待，说明固定窗口并移除分钟输入。运行详情按快照展示 30 分钟及无限等待的手动停止提示。 [编辑器](https://github.com/ora-space/desktop/blob/6489e9d1cafc2a392f71332156425236cf50a83f/packages/app-shell/src/features/workflow-editor/workflow-agent-inactivity-field.test.tsx#L191)、[提示文案](https://github.com/ora-space/desktop/blob/6489e9d1cafc2a392f71332156425236cf50a83f/packages/app-shell/src/features/workflow-editor/workflow-agent-inactivity-field.test.tsx#L213)、[运行详情](https://github.com/ora-space/desktop/blob/6489e9d1cafc2a392f71332156425236cf50a83f/packages/app-shell/src/features/workflow-run/run-act-inspector.test.tsx#L443) | Covered |
| 升级重启结束在途回合，外层节点在显式恢复时开启新的节点尝试 | 启动清理测试用默认策略的外层节点，断言它以 `interrupted_by_restart` 失败，显式 `resume_from_failure` 后进入第二次尝试。该测试未执行实际升级，也没有真实回合和 `wait` 策略；复合区域内被中断的轮次由复合节点吸收、结算为失败轮，没有直接证据。 [启动清理与恢复](https://github.com/ora-space/desktop/blob/6489e9d1cafc2a392f71332156425236cf50a83f/crates/backend/src/workflow/run/resume_gap_tests.rs#L453) | Partial |
| 升级重启并恢复后，下一次实际发送按恢复后运行快照的策略执行（未切换快照时即原冻结值） | 跨版本恢复后首次实际发送的策略缺直接断言，由 [#1421](https://github.com/ora-space/circus/issues/1421) 验证。 | Missing |

**决策依据**　D1。

## Only Attributable Progress Renews A Bounded Wait

**风险与前置状态**　已发送 `wait` 回合，同时存在其他会话、同轮多个委托及权限应答。遗漏真实进展会过早取消，把重复或无关更新当作进展会让回合永久挂起。

**触发与预期**　用可控时钟在窗口内注入下列事件。正例从事件到达时刻重新计满 30 分钟，反例保留原截止。工具执行和权限写入期间计时继续。禁止重复事件延长窗口，或父子来源之间共用计划比较状态。

| 信号 | 可续期 | 保留原截止 | 直接证据 |
| --- | --- | --- | --- |
| 模型内容 | 本轮非空消息或思考 | 空内容、其他会话内容 | Partial。非空消息按到达时刻续期，空消息和空思考保持原截止。非空思考和其他会话内容未见成对的有限窗口验证。 [消息到达](https://github.com/ora-space/desktop/blob/6489e9d1cafc2a392f71332156425236cf50a83f/crates/agent-runtime/src/prompt_policy_tests.rs#L956)、[空内容](https://github.com/ora-space/desktop/blob/6489e9d1cafc2a392f71332156425236cf50a83f/crates/agent-runtime/src/prompt_liveness.rs#L492) |
| 计划 | 同一来源计划条目内容、顺序、优先级或状态变化 | 重复计划、仅 `_meta` 变化。交替发送父子计划后分别重复也不续期 | Partial。父子交替报告后，重复父计划及仅变更元数据的子计划均保持原截止。条目重排、优先级及状态变化的负例与正向续期均未分别断言。 [按来源比较计划](https://github.com/ora-space/desktop/blob/6489e9d1cafc2a392f71332156425236cf50a83f/crates/agent-runtime/src/prompt_liveness.rs#L527) |
| 父工具 | 首次报告状态、状态变化、同一工具的 `content` 或 `rawOutput` 有非空变化 | 重复状态或内容、空输出、仅修改 `title`、`kind`、`locations` | Partial。运行中工具重复状态不延时；非空 rawOutput 可续期，重复值、仅改标题和恰在截止时的输出保持原截止。其他输出字段及纯展示字段缺逐项断言。 [运行中工具](https://github.com/ora-space/desktop/blob/6489e9d1cafc2a392f71332156425236cf50a83f/crates/agent-runtime/src/prompt_liveness.rs#L436)、[输出变化](https://github.com/ora-space/desktop/blob/6489e9d1cafc2a392f71332156425236cf50a83f/crates/agent-runtime/src/prompt_liveness.rs#L467) |
| 子代理 | 本次发送仍在等待的具体委托通过父工具 ID 关联，所报更新符合相同进展判据 | 只知道父会话、委托不匹配或已结束、上一轮委托、重复输出 | Partial。活跃委托与父会话各自比较计划。委托完成后子消息保持原截止，但子代理有效更新的正向续期未单独断言。跨发送复用委托号及重复子工具输出未见有限窗口直接对照。 [活跃委托](https://github.com/ora-space/desktop/blob/6489e9d1cafc2a392f71332156425236cf50a83f/crates/agent-runtime/src/prompt_liveness.rs#L527)、[已完成委托](https://github.com/ora-space/desktop/blob/6489e9d1cafc2a392f71332156425236cf50a83f/crates/agent-runtime/src/prompt_liveness.rs#L492) |
| 权限 | 确认归属本轮父工具，且自动应答成功写入 | 尚在写入、写入失败、子代理或来源不明 | Partial。活跃父工具应答可续期，未知、已完成的工具不能续期。写入挂起时未决请求仍在；写入失败及投影子权限未在有限等待下逐项断言。 [权限归属](https://github.com/ora-space/desktop/blob/6489e9d1cafc2a392f71332156425236cf50a83f/crates/agent-runtime/src/prompt_liveness.rs#L492)、[挂起写入](https://github.com/ora-space/desktop/blob/6489e9d1cafc2a392f71332156425236cf50a83f/crates/agent-runtime/src/support.rs#L344)、[续期恢复应答](https://github.com/ora-space/desktop/blob/6489e9d1cafc2a392f71332156425236cf50a83f/crates/agent-runtime/src/prompt_policy_tests.rs#L1045) |
| 其他流量 | 无 | 会话级通知、`usage_update`、load 回放、stderr、其他会话、未知或未列入清单的扩展通知 | Partial。usage_update 和重复工具状态保持旧截止；回放、stderr、其他会话及未知通知未逐类在有限窗口下验证。 [使用量通知](https://github.com/ora-space/desktop/blob/6489e9d1cafc2a392f71332156425236cf50a83f/crates/agent-runtime/src/prompt_liveness.rs#L436) |

**决策依据**　D2。

## A Bounded Wait Settles Once Before Node Failure Handling

**风险与前置状态**　已发出的有限等待回合带有订阅者和未决权限。静默截止临近时，正常响应、显式取消和进展可能与定时器同时就绪。旧发送尚未收敛就重试会重复产生工具副作用。

**触发与预期**　推进时钟到截止，分别控制进展的到达时刻、终态响应、取消命令和 provider 取消回执。核对 owner、订阅者、节点状态与实际发送次数。禁止双重结算、迟到更新进入下一次发送，或把用户停止记录成静默超时。

| 验证义务 | 证据与边界 | 状态 |
| --- | --- | --- |
| 正常响应与到期同时就绪时正常完成。显式取消先受理时记录用户停止，到期已结算后迟到的取消或响应不能改写终态 | 同一截止批次里已就绪的终态先结算，超时处理中的迟到取消保留原归因。普通回合有用户停止归因断言；有限窗口到期前显式取消的对照尚不完整。 [已就绪的终态](https://github.com/ora-space/desktop/blob/6489e9d1cafc2a392f71332156425236cf50a83f/crates/agent-runtime/src/prompt_policy_tests.rs#L1084)、[迟到取消](https://github.com/ora-space/desktop/blob/6489e9d1cafc2a392f71332156425236cf50a83f/crates/agent-runtime/src/prompt_policy_tests.rs#L1223) | Partial |
| 截止前到达、调度延后处理的有效进展按到达时刻续期。恰在截止和截止后到达均不续期 | 截止前一秒到达、截止后才处理的进展从到达时刻续期；工具输出恰在截止时不能续期。严格晚于截止的进展缺少单独的负例。 [路由到达时刻](https://github.com/ora-space/desktop/blob/6489e9d1cafc2a392f71332156425236cf50a83f/crates/agent-runtime/src/prompt_policy_tests.rs#L956)、[截止时的工具输出](https://github.com/ora-space/desktop/blob/6489e9d1cafc2a392f71332156425236cf50a83f/crates/agent-runtime/src/prompt_liveness.rs#L467) | Partial |
| 到期取消请求及未决权限。取消终态获确认时允许复用；宽限耗尽则隔离旧发送。旧发送结束或隔离后才交给节点失败策略，owner 和订阅者只结算一次 | 确认取消后可复用；未确认时先排空已入队输出并摘除旧路由，再交付本轮和订阅者的超时。取消未确认后，同一连接重新挂载原 provider 会话时的迟到更新和权限请求由 [#1392](https://github.com/ora-space/circus/issues/1392) 承接。 [取消未决权限](https://github.com/ora-space/desktop/blob/6489e9d1cafc2a392f71332156425236cf50a83f/crates/agent-runtime/src/prompt_policy_tests.rs#L1018)、[确认后复用](https://github.com/ora-space/desktop/blob/6489e9d1cafc2a392f71332156425236cf50a83f/crates/agent-runtime/src/prompt_policy_tests.rs#L923)、[尾部输出与订阅者](https://github.com/ora-space/desktop/blob/6489e9d1cafc2a392f71332156425236cf50a83f/crates/agent-runtime/src/prompt_policy_tests.rs#L1133)、[隔离顺序](https://github.com/ora-space/desktop/blob/6489e9d1cafc2a392f71332156425236cf50a83f/crates/agent-runtime/src/prompt_policy_tests.rs#L1223) | Partial |
| 本轮不因静默重发，保留 stale-load 恢复。非交互节点只按自身失败策略决定重试 | 非交互节点配置一次重试，两次节点尝试合计发送两次；取消结算期间已有输出会阻止重复发送。`wait` 下 stale-load 首轮失败后的重建重发尚无直接断言，现有重建门禁用默认 `Timeout` 策略且只是负向门禁。 [节点重试和发送计数](https://github.com/ora-space/desktop/blob/6489e9d1cafc2a392f71332156425236cf50a83f/e2e/tests/desktop/core/session/tests/abnormal_first_turn.rs#L244)、[重建门禁](https://github.com/ora-space/desktop/blob/6489e9d1cafc2a392f71332156425236cf50a83f/crates/agent-runtime/src/prompt_policy_tests.rs#L2597) | Partial |
| 交互自动首轮失败并释放执行会话，无节点重试、无停靠快照。人类回合保留会话和历史，以 `prompt_timed_out` 异常停靠，不自动重试 | 发布图中自动首轮单次发送后失败且未创建停靠；人类回合保留原会话、异常停靠并拒绝默认完成。节点会话视图保留回放和回合错误。自动首轮释放会话在该超时用例中没有独立断言；两个 e2e 都设了 `retry.enabled = false`，「无节点重试」未被测试证伪。 [自动首轮](https://github.com/ora-space/desktop/blob/6489e9d1cafc2a392f71332156425236cf50a83f/e2e/tests/desktop/core/session/tests/abnormal_first_turn.rs#L221)、[人类回合](https://github.com/ora-space/desktop/blob/6489e9d1cafc2a392f71332156425236cf50a83f/e2e/tests/desktop/core/session/tests/abnormal_human_turn.rs#L193)、[会话视图](https://github.com/ora-space/desktop/blob/6489e9d1cafc2a392f71332156425236cf50a83f/packages/app-shell/src/features/workflow-run/run-node-session-chat.test.tsx#L396) | Partial |
| 可观察策略、会话和发送归属、尝试、窗口、静默时长、运行及待执行工具数、终止原因和取消结果，诊断不含模型内容或工具参数 | 结构化日志测试断言字段全集，包括发送归属、策略、窗口、静默时长、工具计数及取消结果。未覆盖运行中的工具取值，也未覆盖 `grace_expired` 分支。 [超时诊断](https://github.com/ora-space/desktop/blob/6489e9d1cafc2a392f71332156425236cf50a83f/crates/agent-runtime/src/prompt_policy_tests.rs#L1686) | Partial |

**决策依据**　D3。
