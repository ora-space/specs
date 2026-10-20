---
status: implemented
---

# Node 通过专用模型身份领取临时授权，OpenCode 经平台代理调用冻结的模型

> 后续决策：[插件拥有模型 adapter](20261017-plugin-owned-model-adapters.md) 已批准将 CLI 配置
> 转换从通用授权模块交给插件，并增加冻结上下文与能力检查；尚待实现，本文既有证据不自动扩大。

本决策继承 [共享运行时根决策](0-shared-agent-runtime-crate-hosted-by-node.md)，将 D5 的部署共享凭据/默认模型改为用户的冻结连接与平台代理。
真实上游 Key 保留在平台；Node 只取得当前运行的临时授权。状态 `implemented`；证据与缺口见核心用例。

## 继承与修改

| 根决策约定 | 本文处理 |
|---|---|
| 共享 ACP 运行时及只启动指定插件 | 继承，不变 |
| D3：JSONL 与 Thread 顺序、D4：Git 环境身份、D6：结束收尾 | 继承，不变 |
| D5：沙盒部署提供真实模型 Key，使用 agent 默认模型 | 修改：Cloud 固定模型绑定，Node 通过 mTLS 领取临时授权并为 OpenCode 生成配置 |
| D2：忽略 stdio 关闭的孤儿进程尚未纳入 guardian | 继承根决策 D2 记录的偏离，本决策不声称修复 |

## 决策

### D1：绑定引用与能力显式协商

会话规格增加可选的 `model_binding_id`，不带 Key、代理令牌或可由调用者选择的上游。
部署配置 `agent.model_proxy` 时先验证专用 TLS 材料，再声明 `ModelProxy` 能力。
Controller、Cloud 登记与 Node 受理都检查模型绑定要求；缺能力时不能退回直连模型。
没有绑定的 Echo 和历史执行保持既有行为。

### D2：临时凭据只服务本次会话

Node 使用 model-access 客户端证书申请当前绑定/执行授权；原 WSS 服务端证书不承担客户端用途。
平台返回冻结协议、模型、固定代理地址与临时令牌；Node 在到期前延长同一授权。
OpenCode 配置引用进程环境中的临时令牌，分别使用 OpenAI 兼容或 Anthropic provider，并保留原模型 ID。
信任代理的公开 CA，正常验证 HTTPS，不使用关闭证书校验的开关。

### D3：运行数据与 checkout 分离

OpenCode 的 HOME、XDG、缓存及会话数据位于 Node 拥有的独立运行目录，不放在 checkout。
目录须允许部署的工作负载身份读写；代理公开 CA 位于可读的公开证书副本，客户端私钥仍只供管理身份读取。
模型绑定会话显式禁止 OpenCode 发现项目 `.opencode` 配置与依赖，保留平台生成的全局配置。
这样仓库配置、依赖安装、缓存和 Agent 私有会话数据不会混入 Revision；仓库任务本身仍可正常读写业务文件。
正常结束先停止续期、撤销授权并清理临时目录；续期被拒则取消对话。
专用 ending 响应只能在目标及身份验证通过后返回；Node 不再调用模型，并在有限时间内等待已有持久结束命令，保留用户结束原因。其他撤销立即失败关闭。
控制账本只保存绑定引用；临时令牌及客户端私钥内容不进入规格、JSONL 或 Revision。
Node 重启继续按根决策以 `interrupted` 收尾，不恢复原 agent 会话。

## 不变量与取舍

1. Node 不持有真实上游 Key；绑定不能指定另一个用户或模型。
2. 没有完整配置和已验证证书就不声明 `ModelProxy`。
3. 结束与取消停止续期，不能留下无生命周期归属的后台任务。

代理比直连多一次网络转发，但令个人 Key 的保护和撤销由平台统一承担；不另写 agent 客户端，继续使用已有 ACP 语义。
验证证据见 [个人模型访问](../../../test-cases/cloud/model-access/personal-model-access.md) 和现有
[会话执行用例](../../../test-cases/node/agent-runtime/session-execution.md)，进程隔离与生产轮换仍是后续边界。
