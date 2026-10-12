# 授权载荷的覆盖范围：决定"克隆什么"的值必须被签名或被权威行复核

依据[仓库凭据的能力词汇表](../../../decisions/cloud/identity-access/20261012-repository-credential-capability-vocabulary.md)
（`approved`）与[用户托管的 Git 凭据](../../../decisions/cloud/identity-access/20261010-user-managed-git-credentials.md)
D4（`approved`）。本文件登记**签名载荷覆盖到哪些输入**这一条承诺，以及一个已登记但尚未闭合的缺口：
带凭据 clone 的分支经 `cloneexec.Job.Branch` 传输而**不在签名内**。

能力词汇表本身登记在[凭据能力词汇表](repository-credential-capability-vocabulary.md)；秘密的承载、兑换与
原子消费登记在[凭据 Secret Store](repository-credential-secret-store.md)；执行侧的 `clone` 语义与身份隔离
登记在 [Node 凭据执行身份隔离](../../node/repository/git-credential-execution-isolation.md)。本文件不重复
这些义务，只登记"载荷够不够"这一侧。

**缺口来源与本次登记的缘起（2026-10-13）**：D4 固定了绑定项的清单——
`tenant + owner + project + 规范化仓库 + credential_id + credential_version + workspace + execution + clone`
——**其中没有分支或 ref**。因此 clone 路径的分支未被签名不是一次实现疏漏，而是该决策没有覆盖到的输入。
2026-10-13 追加 `mirror_prepare` 用途时，镜像侧的 `ref` 被签进 `cloneauth.Binding.Ref`（见本文件第二个
核心用例），同一份载荷里于是出现了一边签名、一边不签名的两个用途，这个不对称必须登记而不是沉默。

## A Value That Decides What Is Cloned Is Either Signed Or Compared Against Cloud's Row

### 风险

某个决定"这次执行到底读写什么"的输入既不在签名载荷里，也没有在执行方与 Cloud 权威行之间被复核，于是
它能被传输路径上的第三方改写，而执行方无从分辨。载荷看起来绑定得很完整（租户、仓库、凭据版本、代次全
都有），被漏掉的那一项却恰好是决定对象的那一项。

### 前置状态

- 准入判定（`internal/core/credentialed_clone.go` 的 `admitCredentialedClone`）在 Cloud 自己的事务里读
  `workspaces.requested_ref` 与 `projects.repository_url`，并把 `Branch: w.S("requestedRef")` 放进计划。
- `cloneauth.Binding` 被签名；`cloneexec.Job` 携带 `Repository`、`Branch`、`Capabilities`、
  `CredentialID`、`CredentialVersion`、`Authorization`、`Runtime`，经 mTLS 投递到 C。
- 匿名（无凭据）clone 路径另有一条权威复核：`internal/core/workspace_clone.go:34` 断言
  `input.S("branch") == w.S("requestedRef")`，不一致时以 `dispatch_conflict` 拒绝。

### 触发

- 在 Cloud 的派发方与 C 之间改写一次投递体里的 `branch`（该投递是 mTLS、在内部控制网段上）。
- 或在 C 侧构造一条 `Job`，其 `Authorization` 合法但其 `Branch` 不是签发时 Cloud 读到的那个。
- 或新增一种用途时只把新的输入写进 `Job` 而忘了放进 `Binding`。

### 必须成立

- **已闭合的一半**：`mirror_prepare` 用途的 ref 在签名内。`cloneauth.Binding.Ref` 为该用途必填、为
  `clone` 必须为空，取值限定在 `refs/heads/` 下；`Issue` 与 `Verify` 都经 `validate()`，因此签名验证
  通过后该规则仍然成立，且 ref 的字节被签名覆盖。
- **未闭合的一半（本条缺口）**：`clone` 用途的分支不在签名内。C 在执行前只按 Git 的 ref 规则校验其
  **语法**（`internal/cloneexec/git.go` 的 `validateBranch`），不校验其**权威**；Store 的 `Resolver.Resolve`
  也不检查 operation 或 ref。因此 C 会克隆投递体里写的那个分支。
- 缺口的影响有界，登记时必须一并写明，不得夸大为任意仓库读写：凭据 token 按仓库（host+path）限定 scope，
  因此**不产生跨仓库提权**；可被改写的是同一仓库内的分支，其后果是该次执行的 `clone_ready`
  commit 记录成了另一个分支的提交。
- 利用前提是能在 Cloud 与 C 之间的 mTLS 投递上改写请求体，即攻击者已在信任边界以内。这是纵深防御
  缺口，不是可远程触发的问题。
- 关闭它需要在**决策**层面选择其一：把 ref 并入 `clone` 的绑定（改变 clone 载荷），或在执行方把
  `Job.Branch` 与绑定比对（需要载荷里有可比对的权威值）。两者都不是可以顺手加的实现细节。

### 禁止结果

- 新增用途时只把决定对象的输入放进 `Job`，不放进 `Binding`，且不留登记。
- 让执行方以"调用方自述"为权威：C 不得因为 `Job` 里写了某个分支就认为它已被授权。
- 把"token 是仓库 scope 的"当作"分支被改写无所谓"的理由——记录到执行行的 commit 会因此指向另一分支。
- 以匿名路径的 `dispatch_conflict` 复核为凭据 clone 路径已有同等复核的证明。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| `mirror_prepare` 的 ref 在签名内，且两个方向的该有／不该有都被强制 | `cloneauth.TestMirrorPreparationRoundTrip`（往返后 `Operation` 与 `Ref` 原样保留）；`cloneauth.TestTheRefIsInsideTheSignedPayload`（只改写载荷中 ref 的字节、保留原签名，验签失败；测试自检载荷确实含 ref）；`cloneauth.TestIssueRejectsIncompleteBindings` 新增四例覆盖"clone 带 ref""mirror 无 ref""mirror 用短名""mirror 指向 tag" | Covered |
| ref 的取值被限在分支命名空间且按 Git ref 规则校验 | `cloneauth.TestValidateRef`：22 组输入，含短名、tag、remote-tracking、空组件、`-` 前缀（选项注入）、`..`、`@{`、`~^:?*[\\`、`.lock` 与 `.` 后缀 | Covered |
| `clone` 用途的分支被签名或被权威行复核 | 尚不存在。D4 的绑定清单不含分支，`Binding` 亦无该字段；需要一个决策来选择关闭方式 | Missing |
| 执行方不以调用方自述为权威 | 今天的直接证据只覆盖能力集合（`cloneexec.TestExecuteRefusesBeforeSpendingAnything`），不覆盖分支；`Job.Branch` 在 C 侧只有语法校验 | Partial |
| 匿名路径的权威复核继续成立 | `core` 的 `workspaceCloneDispatch` 断言 `input.branch == w.requested_ref`，证据登记在 [clone 与结果边界](../../node/repository/clone-and-result-boundaries.md)，本文件不重复 | Covered |

### 决策依据

D4（绑定项清单，其中不含分支）；`mirror_prepare` 的 ref 绑定以本文件登记的工作区变更引入，待并入
一份 `proposed` 决策；[授权载荷可表达 mirror_prepare](repository-credential-capability-vocabulary.md)
（能力词表不因该用途扩宽）。
