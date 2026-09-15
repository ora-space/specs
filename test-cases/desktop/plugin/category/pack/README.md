# Plugin Pack 核心测试用例

本目录是插件集合包领域的 reviewer 风险导航图，与
[决策目录](../../../../../decisions/desktop/plugin/category/pack) 镜像。

架构边界和不变量来自
[插件集合包根决策](../../../../../decisions/desktop/plugin/category/pack/0-plugin-pack.md)。

## 当前状态

根决策已为 `approved`，尚未实现。以下用例记录已批准的能力承诺；当前尚未找到 pack 行为的
直接可执行证据，验证义务均为 `Missing`。既有单插件测试不代替 pack 编排、恢复或前端通知证据。

清单与 registry 配套 ADR 均已批准。字段和分发细节分别见
[清单核心用例](../../manifest/pack-manifest-and-visibility.md)与
[registry 核心用例](../../registry/local-packs-and-hidden-listings.md)；本目录保留跨领域的能力验收。

## 用例导航

### [安装、恢复与整体卸载](installation-and-recovery.md)

- [Preflight and Local Preparation Must Precede Every Member Installation](installation-and-recovery.md#preflight-and-local-preparation-must-precede-every-member-installation)
- [Only Same Namespace Installed Members May Be Skipped](installation-and-recovery.md#only-same-namespace-installed-members-may-be-skipped)
- [Pack Success Requires Durable Ownership of Every Newly Installed Member](installation-and-recovery.md#pack-success-requires-durable-ownership-of-every-newly-installed-member)
- [Exhausted Retries Must Roll Back New Members Before Removing the Pack](installation-and-recovery.md#exhausted-retries-must-roll-back-new-members-before-removing-the-pack)
- [Cancellation and State Write Failures Must Preserve Compensation](installation-and-recovery.md#cancellation-and-state-write-failures-must-preserve-compensation)
- [Recovery Must Reconcile Commits Without Resetting Retry Budgets](installation-and-recovery.md#recovery-must-reconcile-commits-without-resetting-retry-budgets)
- [Concurrent Operations Must Not Share or Steal Member Ownership](installation-and-recovery.md#concurrent-operations-must-not-share-or-steal-member-ownership)
- [Pack Uninstall Must Preserve Unowned and Independently Reinstalled Plugins](installation-and-recovery.md#pack-uninstall-must-preserve-unowned-and-independently-reinstalled-plugins)
- [Persistent Staging Must Survive Version Cleanup and Reject Unknown State](installation-and-recovery.md#persistent-staging-must-survive-version-cleanup-and-reject-unknown-state)

### [清单、展示与分发边界](listing-and-distribution.md)

- [Pack Manifests Must Remain Non Runtime and Reject Marketplace Visibility](listing-and-distribution.md#pack-manifests-must-remain-non-runtime-and-reject-marketplace-visibility)
- [Hidden Members Must Remain Installable After Index Filtering](listing-and-distribution.md#hidden-members-must-remain-installable-after-index-filtering)
- [Readable Old Caches May Retain Hidden Listings Until Successful Refresh](listing-and-distribution.md#readable-old-caches-may-retain-hidden-listings-until-successful-refresh)

## 证据维护

每项义务须通过生产接口取得直接证据，标记为 `Covered`、`Partial` 或 `Missing`；不能仅凭
生成类型、代码存在或依赖组件测试通过升级证据状态。安装与卸载结果的后端断言和前端通知分别
验证，文件系统恢复需包含重建宿主后的观察，避免只验证同一进程中的内存状态。
