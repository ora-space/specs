# Host 按已知运行时值脱敏插件日志

决策依据：[插件日志脱敏 ADR](../../../../decisions/desktop/plugin/logging/20261120-redact-known-runtime-values.md)。
本文记录长期验证义务及代表性直接测试。`Missing` 表示尚无对应直接证据，不能以相邻测试替代。

## Known Runtime Values Must Be Registered Before Delivery and Never Persist

- **风险：** 敏感值在登记前就交付给插件，插件回显后未脱敏落盘；或会话结束/刷新后登记被撤销，旧值仍被回显而泄露；或插件的值与注册表互相干扰。
- **前置状态：** 一个插件进程代已启动，Host 即将交付插件 MCP Settings、宿主 MCP url/headers、模型 token、desktop Skill secret 或工作流完成工具 URL；存在刷新后新值与旧值并存。
- **触发：** Host 通过 spawn 环境、`agent/start` 的 `AgentModelContext`、ACP 会话 setup/load/刷新交付任一敏感值；插件把该值写入 stderr。
- **必须成立：** Host 在交付前先把值登记进接收该值的 Agent 插件的注册表并发布；同一进程代内值一经登记成功只增不减，刷新替换不撤销旧值；该进程代写 stderr 的任何含该值的完整帧，落盘记录的插件可控字段中都不含该值（Host 写入字段按不变量 8 保留）。
- **禁止结果：** 值先交付后登记导致首条诊断泄露；刷新替换导致旧值失去覆盖；已登记值残留在落盘记录中；插件能读取或改写注册表。
- **验证义务与证据：**

| 验证义务 | 代表性证据 | 状态 |
| --- | --- | --- |
| 交付前登记：sentinel 作为 Setting / 宿主请求头 / 模型 token / Skill secret / 完成工具 URL 被测试 Agent 打印到 stderr，`plugin.log` 不出现原文 | 待实现（pipeline 匹配 + 交付点采集 + 生命周期登记表的端到端 sentinel 测试） | Missing |
| Node 路径覆盖：插件 MCP Setting（参与传输绑定的，含刷新后新值）、宿主 MCP URL 与请求头、模型 token（adapter token 与 legacy 环境 token）都脱敏 | 待实现（Node 交付点各自回显 sentinel 的测试） | Missing |
| desktop 路径覆盖：插件 MCP Setting（参与传输绑定的）、Skill secret、工作流完成工具 URL 都脱敏 | 待实现（desktop 交付点各自回显 sentinel 的测试） | Missing |
| 注册表只增不减、条数与总字节有界、超限不阻塞交付；刷新替换不撤销旧值 | 待实现（同进程代内刷新前后两值都回显的测试） | Missing |
| desktop 同一进程代并发交付多个会话的值互不覆盖，单会话结束后该值仍保留在并集中 | 待实现（desktop 同代多会话交付后各自回显、会话结束后仍回显的测试） | Missing |
| 插件不能读取或覆盖注册表；注册表为空时管道行为与未实现脱敏时完全一致 | 待实现（空注册表下管道逐字节不变的测试） | Missing |

- **决策依据：** D2、D4，不变量 2、6、7。

## Redaction Must Not Alter Host-Written Fields or Record Accounting

- **风险：** 脱敏改写 Host 写入的 JSONL 字段或 JSON 结构，导致身份、级别、时间戳失真或整行不再是合法 JSON；或字节核算与脱敏后长度不符，破坏队列容量边界。
- **前置状态：** 一个登记值等于或包含于 `message`、`plugin.stderr`、`2026-11-24T`、插件 ID 本身、Host 身份/时间戳等 ≥8 字节的固定内容；或登记值含 `"`、`\` 等需 JSON 转义的字符。
- **触发：** Host 在 `decode_frame` 之后、`to_json_line` 之前对插件可控字段做脱敏，然后渲染入队。
- **必须成立：** Host 写入的 `timestamp`、`level` 以及 `context` 顶层保留键逐字节不变；每条记录脱敏后仍是合法 JSON；队列字节核算按脱敏后渲染长度计算。
- **禁止结果：** 替换改写 Host 字段、顶层 Host 保留键或 JSON 结构；`Number`/`Boolean` 值或过短值被登记导致误伤或破坏行；队列字节核算与实际入队的脱敏后 JSONL 长度不符。
- **验证义务与证据：**

| 验证义务 | 代表性证据 | 状态 |
| --- | --- | --- |
| Host 字段与 JSON 结构在脱敏后逐字节不变 | 待实现（登记值命中 `message`/`level`/`plugin_id` 等固定字段时仍保持 Host 字段不变的测试） | Missing |
| 只登记 `String` 且长度 ≥ 8 字节；`Number`/`Boolean` 不登记 | 待实现 | Missing |
| 队列字节核算按脱敏后渲染长度 | 待实现 | Missing |

- **决策依据：** D1、D3，不变量 1、3、8。

## Redaction Must Remain Best Effort at Fragment and Encoding Boundaries

- **风险：** 脱敏被误读为「日志绝无秘密」，而编码形式、跨分片、短值、超限/构建失败后的新值或 Host 拥有之外的文件仍会泄露。
- **前置状态：** 登记值在 raw 行以字面转义序列（`\u0041` 等）、百分号编码（`%41`）、base64 形式出现；值跨 64 KiB 分片边界；值短于 8 字节；注册表超限或匹配器构建失败；Agent 写 Host 拥有之外的其它文件。
- **触发：** 插件把上述形式的敏感内容写入 stderr。
- **必须成立：** 每个分片各自脱敏——完整落在一个分片内的登记值被替换为 `[REDACTED]`；跨分片的值两侧分片各保留一部分明文（部分泄露，不是整体绕过）；转义字节的 raw 记录同样执行明文子串匹配并替换命中的明文；重叠/相邻命中合并后无残留字节；短值、编码形式、Host 拥有之外的文件明确不覆盖。
- **禁止结果：** 把「已脱敏」宣称为「日志绝无秘密」；对编码形式/跨分片/短值给出错误的覆盖承诺；构建失败或超限后登记的新值被误认为已脱敏。
- **验证义务与证据：**

| 验证义务 | 代表性证据 | 状态 |
| --- | --- | --- |
| 完整落在一个分片内的登记值被替换，重叠/相邻命中合并后无残留字节 | 待实现（重叠值如 `abcdWXYZ` 与 `WXYZefgh` 在 `abcdWXYZefgh` 中不残留字节的测试） | Missing |
| 每个分片各自脱敏，跨 64 KiB 分片明确为部分泄露 | 待实现 | Missing |
| 转义字节的 raw 记录同样脱敏 | 待实现 | Missing |
| 短于 8 字节不登记、raw 行字面转义序列/百分号编码/base64 不覆盖、Host 拥有之外的文件不受影响 | 待实现 | Missing |
| 匹配器构建失败或注册表超限：该值及后续新值不登记、每个进程代只告警一次、不丢弃不阻塞记录 | 待实现 | Missing |

- **决策依据：** D3、D5，不变量 4、5。