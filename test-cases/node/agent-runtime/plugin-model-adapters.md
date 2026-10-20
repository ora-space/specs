# 插件模型 adapter：输入、隔离与收尾

决策依据：[插件拥有 CLI 适配](../../../decisions/node/agent-runtime/20261017-plugin-owned-model-adapters.md)。
新增义务均为 `Missing`；[既有模型代理用例](model-proxy.md)只证明其已列范围，不覆盖新的 adapter 契约。

## Adapter startup must apply the frozen binding before the first prompt

- 风险：Agent 已启动但未应用配置便发首轮，或启动错包/错模型。
- 前置：固定发行/声明/adapter 版本、安装租约、模型绑定和可观察的 CLI。
- 触发：准备与启动；各阶段注入错摘要、缺能力、授权拒绝、provider 初始化失败。
- 必须成立：身份匹配、领取授权、应用上下文、就绪、prompt 顺序成立；失败停止启动并撤销已领取授权。
- 禁止结果：ready 标记单独当证明、首轮先于配置、使用 latest/共享凭据重试、部分准备资源遗留。
- 验证义务与证据：

| 义务 | 所需直接证据 | 状态 |
| --- | --- | --- |
| 生产启动顺序 | Node 生产入口经真实插件协议和可控 CLI 观察首轮前输入 | Missing |
| 失败收尾 | 分阶段故障注入核对进程、续期、grant、租约和目录，无仅靠 mock 调用次数的替代 | Missing |

## Runtime context must not leak credentials through durable channels

- 风险：adapter 上下文或错误把长期 Key、短期令牌或客户端私钥写入历史。
- 前置：带可检测合成秘密的授权服务，普通/错误/取消路径和真实日志/账本输出。
- 触发：控制输入、插件握手/provider 初始化、CLI 输出、续期、封口与成果保存。
- 必须成立：长期 Key 仅在凭据服务；短期令牌只在本执行敏感运行通路；公开 CA 与私钥分开。非秘密冻结身份/选项/版本可在控制账本中保留，并与执行输入一致。
- 禁止结果：令牌成为 prompt/Settings/控制账本/文件字面值，CLI 错误原样公开。
- 验证义务与证据：贯通协议、持久输出、运行目录、历史与成果的负向材料扫描，并检查访问权限；同时正向核对账本中的冻结非秘密配置足以校验重试，且不会重读当前默认值。当前 `Missing`。

## Concurrent adapters must not share provider state or discovery results

- 风险：两个用户同 cwd 被缓存或共享连接合并，产生额度/模型串用。
- 前置：同插件、相同 cwd 的两个不同执行上下文，个人连接、模型或 revision 不同。
- 触发：并发启动、发现模型、进行轮次；期间编辑一个人的配置或升级包。
- 必须成立：进程/连接/目录和缓存各自归属明确；旧执行保持冻结；新执行使用新 revision。
- 禁止结果：共享进程来回 providers/set、cwd-only 缓存、全局环境突变或热切换在途 provider。
- 验证义务与证据：真实宿主进程与受控模型端点并发核验每条请求的执行归属，当前 `Missing`。

## Project and ambient settings must not override platform model routing

- 风险：配置目录隔离仍被项目 env、登录文件或更高优先级规则覆盖。
- 前置：固定支持版本，checkout 中有冲突配置，宿主环境有其他 provider/登录态，包含特殊字符输入。
- 触发：初始化 adapter、读取项目配置、首轮与辅助调用。
- 必须成立：所有请求仍走授权代理和冻结模型；无可靠覆盖能力时准备失败；原项目文件不变。
- 禁止结果：删除用户文件、继承长效凭据、修改平台权限、把字符串当 shell/原生配置程序执行。
- 验证义务与证据：固定真实 CLI/adapter 的配置优先级、特殊字符串和出站观察矩阵；不能只测生成文本，当前 `Missing`。

## Aliases and discovery must remain within the single-model authorization

- 风险：模型选择器或辅助别名访问第二模型，发现过程先使用未授权凭据。
- 前置：Claude 单模型绑定以及无绑定的配置页面；adapter 声明可选发现能力。
- 触发：展示配置、运行内发现、主/辅助调用、会话模型切换和不支持的发现端点。
- 必须成立：配置页用个人目录；运行内发现有上下文；真实调用只到冻结模型，未支持的能力不冒充成功。
- 禁止结果：影子运行、原生默认模型回退、用配置选项绕过 Gateway 比对、扩大授权模型集合。
- 验证义务与证据：网页无启动副作用及固定 CLI/受控上游实际请求核对，当前 `Missing`。

## Revocation and termination must close the adapter resource lifecycle

- 风险：取消后仍续期/推理，或清理另一执行的目录。
- 前置：运行与续期进行中，另一执行并发存在。
- 触发：准备失败、普通撤销、已验证 ending、主动结束、Node 正常停止与重启。
- 必须成立：无新调用，续期停止、授权撤销、所管理进程结束后清理本执行目录和租约；ending 保留原原因。
- 禁止结果：重新解析当前配置复活授权、恢复旧令牌、跨执行清理、以正常退出证明孤儿后代已回收。
- 验证义务与证据：mTLS/网关与真实 Node 进程的故障矩阵，重启/孤儿覆盖分别记录，当前 `Missing`。

## Another compatible plugin must use the same platform integration seam

- 风险：所谓通用接口仍要求每个插件修改 Cloud UI/Node 授权分支。
- 前置：新 OpenCode/Claude adapter 路线和一个使用已有协议的最小测试插件。
- 触发：新增测试插件的声明、adapter 与包，完成个人配置和一次授权执行。
- 必须成立：平台无需增加该插件 ID 分支或专用页面；声明、配置、授权、取消、清理经同一接口。
- 禁止结果：测试硬编码特例、只测 JSON 不启动、把测试插件通过当作其他真实 CLI 已支持。
- 验证义务与证据：可审计变更范围与生产配置/启动接口的集成结果；同时新协议明确拒绝，当前 `Missing`。
