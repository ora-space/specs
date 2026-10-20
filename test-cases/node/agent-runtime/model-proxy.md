# Node 个人模型会话：临时授权与终止

后续[插件模型 adapter 用例](plugin-model-adapters.md)覆盖新上下文、进程/缓存隔离与扩展能力，
新义务均为 `Missing`，不以本文已有模型代理证据替代。

决策依据：[运行模型代理](../../../decisions/node/agent-runtime/20261010-run-scoped-model-proxy.md)。

## Model sessions must acquire scoped access without persisting tokens

- 风险：真实 Key 或临时令牌进入控制账本、checkout、JSONL 或成果。
- 前置：带模型绑定的执行，Node 已验证专用模型客户端证书。
- 触发：启动、续期、结束、Node 重启。
- 必须成立：授权目标固定；续期沿用令牌；配置只在会话运行目录和进程环境；结束撤销并清理；重启按既有中断规则收尾。
- 禁止结果：缺模型代理能力退回直连、WSS 证书替代客户端身份、后台续期脱离会话生命周期。
- 证据与义务：Desktop `model_proxy/tests` 的真实 mTLS、范围、续期/撤销及配置/清理测试；Node 协议兼容与能力拒绝测试。[两协议实际 CLI 证据](https://github.com/ora-space/cluster/pull/15#issuecomment-6107672919)补充无检测到的凭据材料的 Revision 和容器/卷释放。状态 `Partial`，同一真实上游的 OpenAI Completions / Anthropic Messages 各以生产 OpenCode 完成两轮、正常结束和 Workspace 释放；该脚本没有直接核验授权撤销。无密钥成果义务及真实 Key/临时令牌扫描缺口见 [个人模型访问](../../../test-cases/cloud/model-access/personal-model-access.md#real-opencode-must-modify-the-checkout-and-deliver-a-secret-free-revision)。全栈崩溃恢复仍缺直接证据，正常路径不替代故障矩阵。

## Model ending must preserve the durable end reason

- 风险：平台已经记录结束，但撤销先于结束命令到达，使 Node 错记为错误并丢弃成果。
- 前置：目标身份和当前绑定仍有效，会话已进入 ending。
- 触发：首轮授权或续期返回专用 `model_session_ending`。
- 必须成立：停止模型调用，在有限时间内等待已有持久结束命令，保留其结束原因并完成成果封装；未开始会话也可封装空记录。
- 禁止结果：把身份伪造/错误代次/其他撤销识别成正常 ending，或无限等待。
- 证据与义务：Core 的 ending 错误分类矩阵，Desktop `session/model` ending 竞态及未启动会话空 JSONL 封存测试。状态 `Partial`：分类与 Node 收尾有直接证据；初始授权已 ending 时的空记录 Revision 上传、登记及释放仍缺全栈直接证据。

## Project configuration must not become CLI state or Revision output

- 风险：OpenCode 自动发现 checkout 中的 `.opencode`，安装 CLI 依赖并把配置、缓存或私有会话状态纳入成果。
- 前置：官方 `ora-space.opencode@0.6.4`，用户仓库已提交 `.opencode/package.json` 和哨兵文件。
- 触发：两协议各启动全新会话，读取 LICENSE、写文件、追加一轮并结束。
- 必须成立：仅模型会话设置 `OPENCODE_DISABLE_PROJECT_CONFIG=true`；已有仓库 `.opencode` 不修改；HOME/XDG/CLI 配置在会话目录，公开 CA 可读而管理私钥不可读。结束后私有目录删除，Revision 仅含用户请求的仓库改动。
- 证据：`model_proxy/tests` 检查禁用标志；[官方 CLI 隔离与成果验证](https://github.com/ora-space/cluster/pull/15#issuecomment-6107672919)对两协议同 Run 的 bundle 恢复比较，只有用户请求的文件变化，已提交 `.opencode` 不变，并核对对象摘要、工作负载身份、公开 CA 权限及容器/卷释放。
- 状态：配置隔离与该受控链路 `Covered`；完整 guardian 后代纳管和全栈崩溃恢复仍 `Partial`，不从正常结束推定崩溃回收。
