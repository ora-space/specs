# 本地 Pack 与隐藏条目的分发边界

决策依据为已批准的
[本地 pack 与隐藏条目索引 ADR](../../../../decisions/desktop/plugin/registry/20260915-local-pack-installation-and-hidden-listings.md)。
本文聚焦分发所有者的源、索引与读取边界；失败补偿和归属恢复见
[pack 核心用例](../category/pack/README.md)。新增行为的直接证据均为 `Missing`，既有组件证据
不代表这些组合义务已被覆盖。

## Hidden Listings Must Leave the Index Without Changing Source Resolution

**风险**：隐藏被误作无效插件或访问控制，过滤后索引与 resolve 选择同 ID 的不同源条目。

**前置状态**：真实源含可见 pack、隐藏普通插件、可见普通插件、非法清单，以及同源重复 ID
和跨源同 identifier；隐藏条目可有 README 与本地图标。

**触发**：构建索引，再按 canonical ID 解析并读取隐藏条目元数据。

**必须成立**：生成展示条目前过滤隐藏普通插件；隐藏本身不记为损坏清单，非法清单仍按既有
规则诊断。源内路径与去重顺序保持，不能因过滤让索引与 resolve 对同 ID 选择不同条目。
按 ID 读取清单、README、图标和成员安装不把索引当允许列表，已安装实体不被展示过滤隐藏。

**禁止结果**：过滤时删除源文件、将隐藏条目报告为损坏、跨源兜底或只因索引无条目而拒绝读取。

| 验证义务 | 直接证据与缺口 | 状态 |
| --- | --- | --- |
| 新索引无隐藏条目，合法隐藏与非法清单的诊断不同 | 待真实源构建、文件结果及诊断断言 | `Missing` |
| 过滤不改变路径/去重归属，隐藏条目仍可按 ID 读取 | 待同源重复与多源 fixture，经 resolve、README 和图标生产接口验证 | `Missing` |
| 隐藏成员可安装并进入已安装列表 | 待完整安装及列表契约验证 | `Missing` |

**决策依据**：registry 配套 ADR D1；不变量 1、3、6。

## Pack Installability Must Not Pretend That a Downloadable Artifact Exists

**风险**：无资产 pack 的安装入口被禁用，或为使入口可用伪造通用资产，导致误走下载分支。

**前置状态**：合法无资产 pack、普通无资产 listing、普通通用/target listing；pack 的成员
可缺失或不兼容，也可在本地已装任意版本。

**触发**：构建索引并投影前端 listing，随后通过生产安装入口执行 pack preflight。

**必须成立**：pack 由 kind 推导本地可安装性，原始资产状态仍为无资产；普通无资产仍不可安装，
普通 host 判定保持。前端使用领域投影，不独立按 `release_targets` 禁用 pack。索引不递归
解析成员或缓存聚合兼容性；成员错误在无副作用 preflight 报告，已装跳过成员不受源新版本
资产兼容性阻碍。

**禁止结果**：假 URL/摘要/通用 target、新的重复能力标志、仅入口可用便声称全部成员兼容。

| 验证义务 | 直接证据与缺口 | 状态 |
| --- | --- | --- |
| pack 与普通无资产条目的缓存语义和投影结果正确区分 | 待索引文件、AvailablePlugin 与前端入口验证 | `Missing` |
| 索引不求解成员，preflight 负责缺失/兼容性且无副作用 | 待不完整源、预装成员和实际安装入口验证 | `Missing` |

**决策依据**：registry 配套 ADR D2；不变量 2。

## Local Pack Creation Must Keep Source Retrieval and Asset Roots Isolated

**风险**：pack 意外请求制品或同步源，成员跨源使用凭据；复制条目目录带入外来状态，图标读取
回退到另一根或改写安装树。

**前置状态**：多个 source 配置不同获取方式，pack 本地源已存在，成员制品分别合法或校验失败；
registry 条目另含图标、任意文件和伪造状态，安装树已有宿主持久 staging。

**触发**：安装 pack 与成员，执行源扫描/缓存清理并分别读取 marketplace 和 installed 图标。

**必须成立**：pack 自身零 HTTP/S3 制品请求，已有检出上的 preflight 不隐式 Git 同步；成员
继承 pack namespace 及绑定 source 的认证、代理、摘要与 target 校验。复制只消费清单，不能
带入任意目录或源状态；扫描/缓存清理不改动安装树。两侧图标各读各的根，已安装 pack 无图标
可回退为空图标呈现，不回退读取 registry 或为图标写入文件。

**禁止结果**：新增本地 Source 获取模式、跨源认证兜底、跳过成员校验、递归复制或删除持久 staging。

| 验证义务 | 直接证据与缺口 | 状态 |
| --- | --- | --- |
| pack 本地创建无网络获取，成员走绑定 Source 并保留全部校验 | 待双源真实安装入口与受控网络/制品错误验证 | `Missing` |
| 复制及扫描/缓存清理不带入或修改无关文件和持久状态 | 待操作前后源、安装树和状态文件快照验证 | `Missing` |
| 图标根隔离与 installed 无图标回退遵守约定 | 待 registry 有图标而 installed 无图标的生产资产接口验证 | `Missing` |

**决策依据**：registry 配套 ADR D3；不变量 3、4、6。

## Visibility Refresh Must Preserve Readable Caches and Reject Incompatible Ones

**风险**：新可见性规则强制作废可读缓存，或允许隐藏条目暂留时连不兼容缓存也绕过校验。

**前置状态**：可读旧缓存仍有已隐藏条目；对照缓存版本不匹配、字段类型不符或内容损坏；
源刷新可失败或成功。

**触发**：打开市场、刷新、重启后再次读取。

**必须成立**：可见性不改 schema/版本、不为每次列表读取重扫源，可读旧缓存在成功刷新前
允许保留隐藏条目；失败不删除，成功原子替换。不可读缓存在既有加载规则下返回空列表与
`updated_at = 0`，等待显式同步。打开市场不隐式联网。

**禁止结果**：写入可见性索引字段、失败清空有效缓存、部分索引可见或接受错误版本/类型。

| 验证义务 | 直接证据与缺口 | 状态 |
| --- | --- | --- |
| 可读旧缓存经历失败刷新/成功刷新后分别保留/替换，重启一致 | 待真实缓存与同步入口故障注入验证 | `Missing` |
| 格式/版本不符继续降级，读取不重扫源、不触发网络 | 待缓存 fixture 与受控源读取/网络观察验证 | `Missing` |

**决策依据**：registry 配套 ADR D4；不变量 5；
[图标资产 ADR](../../../../decisions/desktop/plugin/manifest/20260909-plugin-logo-as-themed-asset.md)
的缓存失效边界。
