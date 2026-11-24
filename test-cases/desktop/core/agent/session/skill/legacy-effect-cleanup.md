# 升级时清理旧受管 Skill 目录

决策依据：[Skill 按存储路径以会话清单交给 Agent](../../../../../../decisions/desktop/core/agent/session/skill/0-skill-catalog-from-storage-paths.md)
（`approved`）D9。实现尚未开始，本文件全部义务的证据状态为 `Missing`。

## Upgrade Cleanup Must Delete Only Exact Ledger Matches And Be Idempotent

### 风险

升级会删除 Effect 表，账本是判断目录归属的唯一依据。如果按目录名、路径或内容相似度删除，会删掉用户或其他工具放入的
内容，或用户改过的副本。如果迁移先于清理执行，账本已不存在，旧副本永远留在原处，与清单重复甚至冲突。如果一次删除
失败就阻止启动，Windows 上的句柄占用会让用户无法使用应用。

### 前置状态

升级前的 Effect 账本中有若干受管条目，对应的 Workspace 状态分别为：

1. 目录存在，`.ora-managed.json` 标记身份、native identity 和当前 fingerprint 都与账本一致；
2. 目录存在，标记一致但内容被用户修改，fingerprint 漂移；
3. 目录存在，没有标记或标记与账本不一致；
4. 目录已不存在，或 Workspace 根目录已被删除；
5. 路径中含符号链接或 junction；
6. 目录完全匹配，但删除时文件被占用。

另外，同一发现目录下有不在账本中的用户目录，以及一个未终结 Operation 留下的 staging 或 backup Artifact。

### 触发

首次启动移除 Effect 的版本。另做一次：清理完成部分条目后、迁移执行前进程中断，然后再次启动。

### 必须成立

- 只有状态 1 的目录被删除。
- 状态 2–5 的目录、不在账本中的用户目录、Artifact，以及变空的父目录（如 `.claude/skills`）都保持不变。
- 状态 6 的删除失败时，记录 Workspace 相对路径并继续，启动不被阻止，失败的目录不重试。
- 清理在删除 Effect 表的迁移之前执行；迁移完成后不再清理。
- 中断后再次启动时清理重做，已删除的目录被跳过，最终结果与一次完整执行相同。

### 禁止结果

- 删除标记、native identity 或 fingerprint 任一项与账本不一致的目录。
- 沿符号链接或 junction 删除。
- 删除 Artifact、用户目录或父目录。
- 迁移先于清理执行。
- 因单个目录删除失败而阻止启动。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| 只删除三项都与账本一致的目录，其他情况全部保留 | 尚无实现 | `Missing` |
| 链接、junction 与缺失根目录被跳过 | 尚无实现 | `Missing` |
| 单个目录删除失败时记录并继续，不阻止启动 | 尚无实现 | `Missing` |
| 清理先于删除 Effect 表的迁移 | 尚无实现 | `Missing` |
| 中断后重做的结果与一次完整执行相同 | 尚无实现 | `Missing` |

### 决策依据

- Skill 会话清单决策 D9 与“风险与迁移”中的 Effect 表顺序。
- 不变量 1、11。
