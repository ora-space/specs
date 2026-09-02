# 插件 Registry 核心测试用例

本目录是插件分发与来源派生 Namespace 的 reviewer 风险导航图。它记录需要长期守住的系统承诺，
以及这些承诺是否已有直接、可信的可执行证据；它不是测试函数目录，也不负责登记所有单元测试、
表驱动输入或底层 adapter conformance 测试。

架构边界和不变量来自
[插件分发根决策](../../../../decisions/desktop/plugin/registry/0-registry-distribution.md) 与
[来源派生 Namespace 决策](../../../../decisions/desktop/plugin/registry/20260831-source-derived-namespace.md)。
namespace 相关的身份、绑定与去重规则以后者为准。

## 阅读与维护规则

- 核心用例按业务风险组织，不按单元测试、集成测试或 crate 组织。
- 一个核心用例可以由纯类型测试、repository 测试、跨 crate 集成测试与端到端测试共同证明。
- 只有在测试失败会直接说明某项验证义务不成立时，才把该测试列为代表性证据。
- 文件路径与用例标题共同构成稳定引用；代表性测试可以在 doc comment 中引用它们。
- 改变状态、转换条件、副作用顺序、ownership、恢复、持久化或兼容行为时，必须检查相关核心用例
  及其证据状态。

## 证据状态

| 状态 | 含义 |
|---|---|
| `Covered` | 已有测试直接证明完整验证义务。 |
| `Partial` | 已有测试证明部分路径或较低层规则，但尚未证明完整系统承诺。 |
| `Missing` | 尚未找到直接证明该验证义务的代表性测试。 |

`Partial` 和 `Missing` 是对当前证据边界的如实记录，不代表相关实现一定错误。新增测试后应更新
证据表；删除或弱化代表性测试时也必须同步更新。

## 当前基准用例

### [identity-and-namespace.md](identity-and-namespace.md)

- [Namespace Derivation From a Canonical Source URL Is a Frozen Compatibility Contract](identity-and-namespace.md#namespace-derivation-from-a-canonical-source-url-is-a-frozen-compatibility-contract)
- [A Source's Namespace Binding Survives Deletion and Re-Adding Under an Equivalent URL](identity-and-namespace.md#a-sources-namespace-binding-survives-deletion-and-re-adding-under-an-equivalent-url)
- [Installed Package Namespace Comes From the Install Directory, Never From the Manifest](identity-and-namespace.md#installed-package-namespace-comes-from-the-install-directory-never-from-the-manifest)
- [Same-Identifier Entries From Different Sources Must Coexist Without Shadowing](identity-and-namespace.md#same-identifier-entries-from-different-sources-must-coexist-without-shadowing)
- [Manifest Schema Must Ignore Unknown Fields While Still Rejecting Missing Required Ones](identity-and-namespace.md#manifest-schema-must-ignore-unknown-fields-while-still-rejecting-missing-required-ones)

### [ownership-continuity.md](ownership-continuity.md)

- [Uninstalling a Plugin Must Clear Every Ownership Row Its Full Plugin ID Wrote](ownership-continuity.md#uninstalling-a-plugin-must-clear-every-ownership-row-its-full-plugin-id-wrote)
- [Agent Identity and Session Resolution Must Key Off the Full Canonical Plugin ID](ownership-continuity.md#agent-identity-and-session-resolution-must-key-off-the-full-canonical-plugin-id)
