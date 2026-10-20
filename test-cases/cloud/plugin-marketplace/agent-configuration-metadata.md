# Agent 配置元数据的分发、版本与安装证据

决策依据：[静态元数据分发](../../../decisions/cloud/plugin-marketplace/20261017-agent-configuration-metadata.md)。
新增义务均为 `Missing`；已有 catalog/安装/工作流资产测试不证明 Agent 配置分发已接通。

## Configuration reads must not require agent execution or package downloads

- 风险：首次配置依赖一个无凭据 Agent，或页面请求执行发布者代码。
- 前置：已同步合法/非法声明的 catalog，空间没有运行 Workspace。
- 触发：同步以及浏览、编辑插件个人配置。
- 必须成立：同步只解析受控静态副本；页面读取已持久快照；错误条目无可消费声明，其他条目可用。
- 禁止结果：启动插件、创建沙盒、模型探测、下载整包或通过声明引用任意网络资源。
- 验证义务与证据：

| 义务 | 所需直接证据 | 状态 |
| --- | --- | --- |
| 静态同步与安全文件读取 | 生产扫描器对真实 checkout 的摘要、大小、链接 escape、非法声明矩阵 | Missing |
| 页面无执行副作用 | 无 Workspace 的 HTTP/浏览器配置流程，进程/网络观察证明确无相应调用 | Missing |
| 源快照失败边界 | PostgreSQL 集成验证整源失败保留旧快照、单条失败不借用旧声明 | Missing |

## Selected releases must retain their own declaration after catalog updates

- 风险：catalog 最新 schema 覆盖空间已选择的旧包。
- 前置：两个来源或空间使用同名插件的不同版本/target，用户打开旧页面。
- 触发：同步新版本、管理员升级、旧页面保存；为升级前已排队的运行创建 Workspace 并安装插件。
- 必须成立：声明跟随选定发行快照；保存重新校验关联；排队运行的准备使用原冻结包/声明，无法获取原包则准备失败并收尾。
- 禁止结果：按名称/最新版本合并、丢失旧关联、自动清空字段、选相近 target 或把 Space 新期望装入旧运行冒充原绑定。
- 验证义务与证据：

| 义务 | 所需直接证据 | 状态 |
| --- | --- | --- |
| 发行与声明持久绑定 | 真实 PG 源同步、Space 选择、配置保存/运行创建到 Workspace 安装的交错测试，覆盖原包可得/不可得 | Missing |
| 旧编辑器的冲突反馈 | 生产客户端保留草稿、展示版本冲突和显式重新加载的浏览器证据 | Missing |

## Installation evidence must bind the actual package declaration

- 风险：Cloud 接受正确版本号却携带另一声明的安装结果。
- 前置：固定安装计划、正确包及被篡改的声明/target/结果。
- 触发：Node 安装、结果接管、重试和启动前校验。
- 必须成立：真实包验证声明关联，Cloud 检查结果属于原计划；失败不产生新执行资格。
- 禁止结果：元数据有效等于 installed、结果缺项/重复/错摘要仍成功、重试读取 latest。
- 验证义务与证据：

| 义务 | 所需直接证据 | 状态 |
| --- | --- | --- |
| 包内与派发关联一致 | 真实归档经 Node 生产安装入口，篡改项分别拒绝且租约保护旧包 | Missing |
| 结果与计划一致 | Cloud 生产接管接口及 PG 约束验证，不仅序列化快照 | Missing |
| 离线发布一致性 | 同组静态元数据和离线包的发布/安装验证 | Missing |

## Space installation authority must not grant access to personal settings

- 风险：管理员用安装权限读取成员 Key，或用户配置把安装失败改成成功。
- 前置：同空间两用户、一个管理员；安装与个人配置分别处于不同状态。
- 触发：读取“我的配置”、保存个人连接、跨用户请求、安装/移除。
- 必须成立：共享安装和个人配置独立授权、独立展示；新建连接复用同一个人入口。
- 禁止结果：跨用户读写、catalog 存个人值、配置成功改变安装状态或反之。
- 验证义务与证据：HTTP+真实 PG 的跨角色矩阵及生产页面状态组合验证，当前 `Missing`。
