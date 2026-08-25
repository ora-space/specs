# Effect 状态

## 定位

Effect State 以单个 Workspace 为作用域。每个 Workspace 独立维护自己的 Desired State、Managed State、Observed State 和 Preserved State；一个 Workspace 的变更、reconcile 或清理不能修改另一个 Workspace 的状态或资源，即使两者使用相同的插件、Skill、MCP、目标名称或精确版本。

Effect 将 Ora 的声明物化到当前 Workspace、对应的 Agent 配置或运行环境中。本文件只定义物化过程中几类状态的边界，不定义 Effect 的执行、回滚、卸载和清理协议。

## 状态分类

### Desired State

Desired State 是 Ora 希望目标环境达到的状态。例如，某个 Workspace 选择启用哪些 Skill 和 MCP、使用哪些精确版本及配置的 revision。

Desired State 只表达意图，不证明配置已经写入，也不证明 Agent 或 MCP 已经加载。修改 Desired State 会产生新的 revision，并触发后续 reconcile。

### Managed State

Managed State 是 Ora 已经物化并拥有管理权的资源记录。它是 Ora 的 ownership ledger，用来判断哪些目标配置或运行资源可以由 Ora 更新和删除。

每个 Managed State 条目必须具有稳定的 `managed_identity`，并记录足以定位和校验目标资源的信息，例如资源种类、目标位置、已应用 revision 和配置 fingerprint。Skill 和 MCP 必须分别记录，不能仅依赖目标配置中的名称推断所有权。

Managed State 不等于 Desired State：资源可能仍在 Desired State 中但尚未成功物化，也可能已从 Desired State 移除但仍在等待安全清理。

### Observed State

Observed State 是 Ora 从目标环境读取或由运行时确认的实际状态，包括 Agent 配置文件当前内容、已应用 revision、Agent 是否正在运行，以及相关执行是否仍在输出或调用 MCP。

Observed State 不是所有权证明。目标环境中存在某个条目，不代表 Ora 有权修改它；只有与 Managed State 中的 `managed_identity` 匹配后，才能把它作为 Ora-managed 资源处理。

### Preserved State

Preserved State 是目标环境中不属于 Ora 的既有状态，包括项目原有配置、用户手动创建的 Skill 或 MCP，以及任何没有 Ora ownership record 的条目。

Preserved State 对 Ora 是不可变的：reconcile、停用和卸载都必须保留它，不能根据名称、插件 ID 或 Desired State 中不存在该条目而覆盖或删除。这里的“不可变”只约束 Ora；项目、用户或其他拥有者仍可修改这些配置。

已有条目默认属于 Preserved State。Ora 不能仅因它与待物化资源同名而接管；如果目标格式无法同时安全表达两者，reconcile 必须报告冲突。

## 关系

```text
Desired State
      |
      | reconcile
      v
Managed State -------- ownership --------> Ora 可更新或清理的资源
      |
      | observe and verify
      v
Observed State
      |
      +---- 没有匹配的 ownership record ----> Preserved State
```

reconcile 必须同时使用四类状态：以 Desired State 计算目标，以 Managed State 限定 Ora 的权限，通过 Observed State 判断实际差异，并在写回时完整保留 Preserved State。

## 不变量

1. Desired State 不能作为资源已经生效的证据。
2. Observed State 不能作为 Ora 拥有资源的证据。
3. 只有 Managed State 中有 ownership record 的资源才允许由 Ora 更新或删除。
4. 没有 ownership record 的既有资源一律作为 Preserved State 处理。
5. 名称、插件 ID、路径相同或配置内容相同，都不能代替 `managed_identity`。
6. 删除 Desired State 不等于立即删除资源；Managed State 必须保留到安全清理完成。
7. Agent 正在输出或仍有相关执行占用资源时，Observed State 必须阻止清理完成，直到输出自然结束或用户停止且运行时确认执行已经终止。
