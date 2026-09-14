# Plugin Pack 核心测试用例

本目录是插件集合包领域的 reviewer 风险导航图，与
[决策目录](../../../../decisions/desktop/plugin/category/pack) 镜像。

架构边界和不变量来自
[插件集合包根决策](../../../../decisions/desktop/plugin/category/pack/0-plugin-pack.md)。

## 当前状态

根决策尚处于 `proposed`。按 ADR 生命周期，核心测试用例在该决策评审通过并标记为 `approved`
后，依据最终同意的不变量补充；在此之前本目录不登记任何用例。

预计首批核心用例将覆盖的风险面（以决策评审结论为准）：

- preflight 未通过时不发生任何下载，`installed/` 与 pack 数据目录均无变化。
- 成员安装失败后停止后续成员，已安装成员保留，pack 自身不落盘，不自动回滚。
- 已安装成员被跳过且不算失败，其既有版本不被改动或降级，也不新增登记。
- 由 pack 新安装的成员在安装提交后登记进 pack 数据目录 `data/<ns>/<identifier>/`。
- 卸载 pack 恰好卸载登记的成员，随后移除 pack 自身与数据目录；登记之外的已安装插件不受影响。
- `marketplace_visible = false` 的条目不进入派生索引，但仍可按 identifier 解析与安装。
- pack 条目在旧宿主上整条消失、隐藏成员在旧宿主上仍然可见的兼容行为。
- pack 包不携带 `assets/`、`main.js` 或任何运行时贡献物。
