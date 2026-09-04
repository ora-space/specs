---
status: implemented
---

# 插件包获取：由 Marketplace Source 选择 Direct HTTPS 或 S3 SigV4

每个 Marketplace Source 独立选择插件包的获取方式：默认使用 **Direct HTTPS**，直接下载清单声明的
HTTPS 地址；需要访问私有对象存储时使用 **S3 SigV4**，把清单声明的对象 key 解析到该源配置的
S3 兼容 endpoint 与 bucket，并以 AWS Signature Version 4 签名下载。选择以一个带类型判别的 JSON
字段持久化，避免在 `plugin_marketplace_source` 中增加一组只对 S3 有意义的 nullable 列。

本决策已经落地。影响范围包括 `plugin_marketplace_source` 的持久化模型、Marketplace Source
契约、“管理市场 → 编辑插件源”界面、release 清单中 `url` 的可接受形态，以及安装/更新插件时的
下载器选择。市场元数据仍通过 Git 同步；本决策只改变 `.orax` 插件包的获取，不用 S3 替代 Git
checkout，也不改变索引构建、SHA-256 校验、解包或插件身份。

现有 source 行迁移为 Direct HTTPS，现有本地检出与安装目录不移动。切换获取方式不改变 source URL、
canonical URL、namespace 绑定或已安装插件 ID，因此不触碰本地文件系统兼容边界。

## 继承与修改

本决策继承
[插件分发根决策](0-registry-distribution.md) 与
[来源派生 Namespace 决策](20260831-source-derived-namespace.md)，并修改以下边界：

| 前序决策的约定 | 本决策 |
| --- | --- |
| Marketplace Source 是 Git 仓库，持久化 `url`、`branch`、`use_proxy`、`enabled` 与顺序 | 继承，并新增一个 source 级 `artifact_retrieval` JSON 配置 |
| release 清单的 `url` 是 HTTPS 地址 | 修改：Direct HTTPS 仍要求 HTTPS；S3 SigV4 还可使用对象 key 或匹配当前 endpoint/bucket 的 path-style HTTPS 地址 |
| 安装/更新按 source 的 `use_proxy` 下载清单指定的插件包 | 继承代理归属；下载器进一步由该 source 的 `artifact_retrieval` 选择 |
| 下载后必须通过清单声明的 SHA-256 才能进入安装目录 | 继承，不变；SigV4 只认证请求，不替代内容摘要校验 |
| source URL 决定 namespace 与本地检出目录 | 继承，不变；S3 endpoint、bucket 和凭据均不参与插件身份 |
| 清单解析是无 I/O 的纯函数 | 继承；解析器只区分 HTTPS locator 与 object-key locator，不读取 source 配置、不签名、不联网 |

## 背景与术语

“普通下载”和“S3 下载”混合了协议、认证与存储产品，不足以表达稳定边界。本决策使用以下领域术语：

| 领域名 | UI 中文名 | 含义 |
| --- | --- | --- |
| `Direct HTTPS` / `direct_https` | HTTPS 直接获取 | 对清单声明的绝对 HTTPS URL 发起普通 GET，不附加 S3 签名 |
| `S3 SigV4` / `s3_sigv4` | S3 签名获取 | 以 S3 兼容对象存储的 path-style URL 定位对象，并用 SigV4 对 GET 请求签名 |
| `Artifact retrieval` | 插件包获取方式 | Marketplace Source 获取其 `.orax` 发布资产的策略；不包括 Git 源同步 |
| `Release locator` | 发布资产定位符 | release 清单 `url` 字段承载的 HTTPS URL 或对象 key |

`S3 SigV4` 比“S3 下载”更准确：真正的网络传输仍是 HTTPS，差异在于对象寻址与 SigV4 请求认证。
`Direct HTTPS` 也比“普通下载”更明确，它排除了 HTTP、本地文件和隐式云厂商 SDK 凭据链。

## 决策

### D1 获取方式是 source 级互斥枚举

一个 Marketplace Source 在任一时刻恰好使用一种 `ArtifactRetrieval`：

```text
ArtifactRetrieval
├── DirectHttps
└── S3SigV4 { endpoint, bucket, region, credentials }
```

模式属于 source，而不属于全局应用设置，也不由每条插件清单自行选择。安装或更新
`<namespace>/<identifier>` 时，先由 namespace 定位唯一 source，再使用该 source 的获取方式解析
release locator。这样，同一台 Ora 可以同时从公开 GitHub source 直接下载，也可以从私有 source 的
S3 兼容对象存储签名下载，二者的凭据和失败互不影响。

这也是一个封闭状态枚举，不使用 `use_s3: bool`。布尔值无法承载 S3 的必填配置，也会产生
“`use_s3 = false` 但数据库仍残留一组有效 S3 凭据”的双重状态。

### D2 用一个判别式 JSON 列持久化完整状态

`plugin_marketplace_source` 新增：

```sql
artifact_retrieval TEXT NOT NULL
    DEFAULT '{"type":"direct_https"}'
    CHECK (
            CASE WHEN json_valid(artifact_retrieval)
                THEN json_type(artifact_retrieval) = 'object'
                    AND json_extract(artifact_retrieval, '$.type') IS NOT NULL
                ELSE 0
            END
    )
```

Direct HTTPS 的完整持久化形态只有：

```json
{
  "type": "direct_https"
}
```

S3 SigV4 的完整持久化形态为：

```json
{
  "type": "s3_sigv4",
  "endpoint": "https://s3.example.com",
  "bucket": "plugin-artifacts",
  "region": "region-1",
  "access_key_id": "example-access-key-id",
  "secret_access_key": "example-secret-access-key"
}
```

选择 JSON 而不是六个标量列，原因不是 S3 配置“无结构”，而是它本身就是一个互斥枚举的关联数据：

- Direct HTTPS 不拥有 S3 字段，数据库中不会出现五个无意义的 `NULL`。
- S3 SigV4 的 endpoint、bucket、region 和两项凭据必须同时存在，不允许半配置状态。
- repository 读写 JSON 时必须反序列化为带关联数据的 Rust enum；业务层不得把它展开成一组
  `Option<String>` 后再猜测当前模式。
- SQLite 的 `CHECK` 只守住合法 JSON、对象形态和 discriminator 存在，不枚举具体形态，避免以后新增
  variant 时必须重建列约束；字段完整性、字符串策略、未知 variant 与未知字段拒绝由 repository 的
  严格解码和验证类型守住。当前运行时遇到损坏或尚不认识的 JSON 必须作为 source 配置错误暴露，
  不得静默回退为 Direct HTTPS。

迁移把所有既有行保留为列默认值 `direct_https`。down migration 只移除本列，不改写其它 source
字段、namespace 绑定或文件系统内容。

### D3 S3 SigV4 只支持一组完整的静态凭据

S3 SigV4 形态的字段全部必填：

| 字段 | 规则 | 用途 |
| --- | --- | --- |
| `endpoint` | 绝对 HTTPS origin；禁止 userinfo、路径、query 与 fragment | S3 兼容服务地址，可含显式端口 |
| `bucket` | 非空、符合本实现接受的 bucket 名称上限；不含 `/` | path-style URL 的首个路径段 |
| `region` | 非空、有界、无首尾空白或控制字符 | SigV4 credential scope |
| `access_key_id` | 非空、有界、无首尾空白或控制字符 | SigV4 credential identity |
| `secret_access_key` | 非空、有界、无首尾空白或控制字符 | SigV4 签名密钥 |

当前版本固定使用 path-style 地址：

```text
{endpoint}/{bucket}/{object-key}
```

不增加 `path_style`、`virtual_hosted_style`、`service`、`session_token`、`key_prefix` 或 provider
开关。它们不是当前 S3 兼容静态 AK/SK 场景的必需状态；提前加入会重新制造大量 optional 字段和未经
验证的组合。将来需要临时凭据或 virtual-hosted-style 时，应新增可判别的配置形态或后续 ADR，不能
改变既有 `s3_sigv4` 的语义。

### D4 release locator 与获取方式必须相容

[插件清单根决策](../manifest/0-versioned-dual-form-manifest.md) D5、D6 中的 release `url` 从单一
`ReleaseUrl` 收窄类型修改为 `ReleaseLocator`：

```text
ReleaseLocator
├── Https(ReleaseUrl)
└── ObjectKey(ObjectKey)
```

解析仍是纯函数。以 `https://` 开头的值按现有 HTTPS 策略验证；包含其它 scheme 的值拒绝；无 scheme
的值按对象 key 验证。对象 key 必须非空、有长度上限、不是绝对路径、不含控制字符，也不得包含 `..`
路径段。字段名暂时保留为 `url`，避免为同一份 release 再增加 `url`/`key` 两个 optional 字段。

source 配置与 locator 的允许组合为：

| 获取方式 | HTTPS locator | Object-key locator |
| --- | --- | --- |
| Direct HTTPS | 直接、无 SigV4 签名下载 | 拒绝：缺少可解析对象 key 的 S3 配置 |
| S3 SigV4 | 仅当 URL 精确属于当前 endpoint/bucket 时提取 key 并签名；其它 URL 拒绝 | 拼成 path-style URL 并签名下载 |

S3 模式接受匹配的完整 path-style HTTPS URL，是为了兼容 source 仓库中已经写成
`https://{endpoint}/{bucket}/{key}` 的 listing；它与裸 key 最终进入同一条签名路径。匹配必须同时
比较 scheme、host、有效端口、bucket 首段，并要求其后至少有一个 key 段。

S3 模式不把“非本 bucket 的 HTTPS URL”降级成无签名 Direct HTTPS。用户选择 S3 SigV4，意味着信任
边界是配置的 endpoint/bucket；静默直连外部 URL 会让 source 内容绕过这一选择。若一个 marketplace
确实混合两种交付方式，应拆为不同 source，或由后续决策引入能明确表达混合策略的新枚举形态。

### D5 SigV4 与现有下载安全边界组合，而不是替代它

S3 下载使用 HTTPS GET 和 SigV4 header signing。签名输入至少包含 endpoint、region、bucket、对象
key、access key ID、secret access key 与协议要求的签名时间；签名生成和请求发送必须共享完全相同
的规范化 URL，避免“签名的是一个路径、下载的是另一个路径”。

无论采用哪种获取方式，以下行为保持一致：

- `use_proxy` 继续决定该 source 的插件包请求是否经过已配置代理；S3 不绕过代理策略。
- 下载仍写入既有临时/缓存路径，并在进入 `installed/` 前验证 release 清单声明的 SHA-256。
- 重试、超时、取消、进度和大小上限沿用统一 HTTP 下载能力。
- SigV4 认证成功不代表包可信；摘要不匹配仍然删除未验证产物并使安装/更新失败。
- 下载失败不修改当前安装版本；更新仍先遵守既有 stop、下载、校验和替换顺序。

### D6 编辑界面只展示当前形态拥有的字段

“管理市场 → 编辑插件源”在分支字段下新增“插件包获取方式”，选项为：

1. **HTTPS 直接获取（Direct HTTPS）**
2. **S3 签名获取（S3 SigV4）**

默认选中当前持久化形态；既有 source 因迁移默认值显示 Direct HTTPS。选择 S3 SigV4 后才追加
endpoint、bucket、region、Access Key ID 与 Secret Access Key 五项，全部为必填。切回 Direct HTTPS
时，这些字段从提交模型中消失；保存后数据库 JSON 只剩 direct 形态，不保留不可达凭据。

Secret Access Key 使用密码输入控件，不回显持久化明文。读取 source 列表的响应不得包含
`secret_access_key`；更新请求以一个明确的凭据操作枚举表达“保留既有凭据”或“用一对新的 AK/SK
替换”，不能用 `secretAccessKey: null` 同时表达保留、清空和未填写。首次从 Direct HTTPS 切到 S3
SigV4 时只允许“替换”，且 AK/SK 必须同时提交；已经处于 S3 形态时，留空的密码控件表示保留，
用户主动填写新凭据才原子替换 AK/SK。

source 读取契约可以返回 endpoint、bucket、region 与“已配置凭据”的状态，但不能返回 secret。
后端日志、错误链、Debug 输出和遥测也不得包含 access key ID、secret access key、签名 header 或完整
持久化 JSON。

### D7 凭据持久化是明确的当前阶段取舍

S3 静态凭据随 `artifact_retrieval` JSON 存入 Ora 的本地 SQLite。数据库必须沿用应用数据目录的访问
边界；备份或复制数据库也会复制凭据。本阶段没有独立 Credential Store，因此不能声称凭据已加密或
由操作系统密钥链保护。

接受这一风险是为了让 source 配置在重启后可用，并与当前本地配置的持久化能力保持同一边界；但
“本地明文可读”不等于“可通过契约或日志扩散”。将来迁移到系统 keychain 时，数据库 JSON 应改存
不可解释的 credential reference，并提供先写入 keychain、验证读取、再原子替换数据库引用的迁移，
不得先清空现有凭据。

## 不变量

1. 每个 Marketplace Source 恰好有一种插件包获取方式；不存在 bool 与 S3 nullable 字段拼成的非法
   中间状态。
2. 既有 source 行迁移后均为 Direct HTTPS，source URL、namespace 绑定、本地检出和安装目录不变。
3. Direct HTTPS 只接受绝对 HTTPS release locator，且永远不添加 SigV4 签名。
4. S3 SigV4 的 endpoint、bucket、region、AK 与 SK 同时存在；缺少任何一项都不能保存或下载。
5. S3 请求只发往配置的 HTTPS endpoint 和 bucket；其它 host、端口或 bucket 不得静默降级为无签名
   下载。
6. release locator 的解析不做 I/O；source 模式与 locator 的相容性在安装/更新解析 source 后判定。
7. 获取方式、S3 配置和凭据不参与 namespace、Plugin ID 或安装目录派生。
8. SigV4 不替代 SHA-256；任何插件包进入 `installed/` 前都必须通过清单摘要校验。
9. Secret Access Key、签名 header 与完整 S3 持久化 JSON 不得出现在 source 查询响应、日志、错误或
   Debug 输出中。
10. 从 S3 切回 Direct HTTPS 后，持久化 JSON 不再保留 S3 endpoint、bucket、region 或凭据。

## 为什么不是这些替代方案

| 方案 | 优势 | 否决理由 |
| --- | --- | --- |
| `use_s3` + 五个 nullable S3 列 | SQL 查询直观 | Direct 行永久携带无意义空列；数据库允许“开了 S3 但缺字段”和“关了 S3 但残留凭据”，非法状态需散落在调用点修补 |
| 独立 `retrieval_type` 列 + nullable JSON | SQL 可直接按模式筛选 | discriminator 存两份，列值与 JSON `type` 可以冲突；当前没有按模式查询 source 的需求 |
| 全局 S3 配置 | UI 与 wiring 最少 | 多 source 无法使用不同 bucket/region/凭据；启用一个私有源会改变其它公开源的下载行为 |
| 在 release 清单中写 endpoint、bucket 与凭据 | 每条 listing 自包含 | 把宿主凭据交给 source 仓库持久化并可能公开；重复配置，也让第三方内容决定用户的认证边界 |
| 只支持预签名 HTTPS URL | 客户端无需持久化 AK/SK | URL 会过期，Git 中的静态 registry listing 无法长期稳定引用；刷新签名需要额外控制面，不满足离线索引与按需安装的现有拓扑 |
| S3 模式对任意 HTTPS URL 自动回退直连 | 支持一个 source 混合公开与私有资产 | source 内容可绕过用户选择的 endpoint/bucket 信任边界，“S3 模式”不再是可验证承诺 |
| 同时支持 path-style、virtual-hosted-style、session token 与 SDK credential chain | 覆盖更多云厂商环境 | 当前需求没有这些状态；一次引入会增加 optional 字段、平台环境依赖和无法在 UI 中解释的凭据优先级 |
| 单独新增 `object_key` 清单字段 | 字段名比复用 `url` 准确 | 与 `url` 形成另一组互斥 optional 字段，旧宿主忽略新字段后会把条目视为无 release；保留 `url` 可让 locator 类型内部演进 |

## 风险与后果

- SQLite 中的静态 S3 凭据对能读取应用数据目录的本机主体可见；D6、D7 只防止二次扩散，不提供静态
  加密。
- source 维护者若把 Direct HTTPS listing 改成 object key，旧版 Ora 会拒绝或无法安装该 release；
  发布方需要把这一变化与支持本决策的 Ora 版本配套。
- S3 endpoint 的证书、时钟偏差、region 或签名实现错误会导致认证失败。错误应区分无效 source
  配置、locator 不相容、签名失败和远端 HTTP 拒绝，但不得携带凭据或 Authorization header。
- JSON 字段降低了临时 SQL 检查单个 S3 属性的便利性；这些字段没有排序、关联或筛选需求，这一代价
  小于让数据库长期承载多列非法组合。
- 严格拒绝 S3 source 中的外部 HTTPS URL 会限制混合交付 source；这是为保持模式语义和信任边界
  做出的有意选择。

## 本决策未解决的问题

- 系统 keychain/Credential Store、凭据轮换审计与临时 session token 不在本决策范围内。
- S3 上传、插件发布、bucket 创建、权限策略和对象生命周期不由 Ora Desktop 管理。
- virtual-hosted-style endpoint、HTTP endpoint、跨 region 重定向与云厂商 SDK credential chain 暂不
  支持。
- endpoint 连通性和凭据校验是否提供独立“测试连接”操作留给后续 UX 决策；保存时只做本地形状
  校验，不以一次远端请求作为持久化前置条件。

## 落地结果

1. **持久化与域模型**：增加带默认值和 JSON `CHECK` 的迁移；repository 使用判别式 enum 严格
   读写，并证明既有行成为 Direct HTTPS、非法 JSON 不会回退。
2. **契约与编辑 UI**：source 查询返回脱敏的获取配置，更新请求使用明确的保留/替换凭据动作；编辑
   界面按模式展示并验证字段。此阶段不改变实际下载行为。
3. **release locator**：把 release `url` 收窄为 HTTPS/ObjectKey 枚举，保持清单解析纯函数；Direct
   source 遇到 object key 必须在网络请求前失败。
4. **S3 下载器**：实现 path-style URL 构造、匹配 URL 的 key 提取、SigV4 签名及统一下载能力接入；
   保持代理、进度、取消、重试、限额与摘要校验行为。
5. **安装与更新接线**：namespace 定位 source 后同时返回其获取配置，按 source 构造下载器；验证不同
   source 的配置不会串用，失败不会破坏已有安装。

对应验证义务与当前证据收录在
[插件包获取核心测试用例](../../../../test-cases/desktop/plugin/registry/artifact-retrieval.md)。
