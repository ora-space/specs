# Project、Workspace、Session 与远程工作环境

## 1. 定位

本文定义 Project、Workspace、Worktree、Session 和 WorkflowRun 的持久化领域边界，
以及本地 checkout、SSH、Dev Container 和 remote agent sandbox 的统一扩展方式。

本文不定义 Effect State，也不包含任何 `workspace_effect_*` 表。Effect State 后续
以 Workspace 为外键独立设计。

## 2. 核心术语

### 2.1 Project

Project 是逻辑仓库或项目。它描述仓库身份、来源和默认分支，不拥有某一个具体
checkout 的权威路径。

一个 Project 至少有一个 main Workspace。Workspace 始终拥有独立的 Workspace ID；
Project 通过 `project_id` 和 `workspace_kind = main` 找到自己的 main Workspace。

### 2.2 Workspace

Workspace 是一个可以被 Session 或 WorkflowRun 使用的实际工作环境。它可以是：

- 本地文件系统目录；
- 本地 Git main checkout；
- 本地 Git worktree；
- SSH 主机上的目录；
- Dev Container 内的目录；
- remote agent sandbox 内的目录。

Workspace 是 Agent 执行和未来 Effect 收敛的隔离边界。Session、WorkflowRun 和后续
Effect State 都通过 `workspace_id` 关联 Workspace，而不是通过 Project 推导
工作目录。

Workspace 的位置不是必然的本地 `PathBuf`。远程位置必须由显式的 tagged union 和
对应的 backend adapter 表达，不能伪装成本地路径。

### 2.3 Worktree

Worktree 只是本地 Git worktree 的实现记录，不是 Workspace 的父级抽象。

Worktree 记录 branch、base commit 等 Git 特有信息；Workspace location 记录实际位置。
SSH checkout、Dev Container 和 remote agent sandbox 不应创建伪造的 Worktree 行。

### 2.4 Session

Session 是一次 Agent 会话，直接绑定一个 Workspace。

### 2.5 WorkflowRun

WorkflowRun 是一次已发布 Workflow snapshot 的执行，直接绑定一个 Workspace。它自己保存展示名称。

## 3. 领域关系

最终关系如下：

```text
Project
  └── Workspace (main)
        ├── Session
        ├── WorkflowRun
        └── Worktree (optional, local Git only)

Project
  └── Workspace (isolated)
        ├── Session
        ├── WorkflowRun
        └── Worktree (optional, local Git only)
```

一个 Project 可以拥有多个 Workspace。

一个 Workspace 可以被多个 Session 和 WorkflowRun 使用。删除一个 WorkflowRun 或
Session 不得隐式删除仍被其他运行使用的 Workspace。

Project-root 使用的 Workspace 是 Project 的 main Workspace，不创建重复的 Workspace。
因此不会出现两个 Workspace 同时管理同一个物理目录的问题。

Workspace 的 `display_name` 用于 UI 展示；本地 Git Worktree 的 branch name 由 Worktree保存。

## 4. ID 规则

### 4.1 Project main Workspace

Project main Workspace 使用独立的 Workspace ID：

```text
workspace.id = WorkspaceId
workspace.project_id = project.id
workspace.workspace_kind = main
```

Project 不保存 `main_workspace_id`；通过 `project_id` 和 `workspace_kind = main` 的
唯一索引查找 main Workspace。这样未来 main checkout 迁移到 SSH、Dev Container 或
remote agent sandbox 时，不需要改变 Project 的身份。

### 4.2 Isolated Workspace

独立 Workspace 使用独立的 Workspace ID。它不依赖 Task 身份。

```text
workspace.id = WorkspaceId
workspace.kind = isolated
workspace.project_id = ProjectId
```

### 4.3 强类型边界

即使底层都是 SQLite `TEXT`，Rust 领域层仍然保留独立的：

```text
ProjectId
WorkspaceId
SessionId
WorkflowRunId
WorktreeId
```

ProjectId 和 WorkspaceId 始终是独立类型，不能因为它们都使用 SQLite `TEXT` 就隐式互换。

## 5. Project Schema

Project 只保存逻辑仓库信息：

```sql
CREATE TABLE projects (
    id              TEXT PRIMARY KEY,
    name            TEXT NOT NULL,
    repository_kind TEXT NOT NULL DEFAULT 'git',
    repository_url  TEXT,
    default_branch  TEXT,
    created_at      INTEGER NOT NULL,
    updated_at      INTEGER NOT NULL,
    is_deleted      INTEGER NOT NULL DEFAULT 0
        CHECK (is_deleted IN (0, 1))
);
```

字段语义：

- `repository_kind`：当前支持 `git`，保留未来 VCS 扩展点；
- `repository_url`：仓库来源，不保存认证信息；
- `default_branch`：Project main checkout 的默认分支；

## 6. Workspace Schema

Workspace 保存稳定身份、所属 Project、checkout 类型和生命周期，不直接保存远程
provider 的全部细节：

```sql
CREATE TABLE workspaces (
    id              TEXT PRIMARY KEY,
    project_id      TEXT NOT NULL REFERENCES projects(id),
    display_name    TEXT NOT NULL,

    workspace_kind  TEXT NOT NULL
        CHECK (workspace_kind IN ('main', 'isolated')),

    location_id     TEXT NOT NULL REFERENCES workspace_locations(id),

    -- Known values: provisioning, active, unavailable, retiring, deleted.
    -- New lifecycle values are validated by the supported domain model, not SQLite.
    lifecycle       TEXT NOT NULL DEFAULT 'provisioning',

    created_at      INTEGER NOT NULL,
    updated_at      INTEGER NOT NULL,
    is_deleted      INTEGER NOT NULL DEFAULT 0
        CHECK (is_deleted IN (0, 1))
);

CREATE UNIQUE INDEX workspaces_active_project_main_unique
    ON workspaces(project_id)
    WHERE workspace_kind = 'main' AND is_deleted = 0;
```

额外不变量：

- 一个 active Project 最多有一个 main Workspace；
- main Workspace 的 ID 与 Project ID 无关；
- Workspace 的 `project_id` 在生命周期内不可改变；
- Workspace location 改变时不得原地修改旧 location 的语义，应创建新 location 或
  通过显式 relocation 流程完成切换；
- `retiring` Workspace 不再接受新的 Session 或 WorkflowRun，但必须完成已有资源的
  安全收尾；
- `deleted` Workspace 不再作为新的外键目标。

创建 main Workspace 和 Project 的 repository 必须在同一个 write transaction 中校验
“Project 存在且最多一个 active main Workspace”这一不变量。

`display_name` 是 Workspace 的用户可见名称。对于 Project main Workspace，默认取
Project name；对于本地 Git worktree（工作树任务），使用创建时用户输入的名称。它不代表 branch
或 revision，Git 事实仍由 Worktree 保存。

## 7. Workspace Location Schema

位置使用独立表。核心只区分三类 location：本地文件系统、SSH 和由插件解释的
`remote_target`。AWS、Azure、GCS 以及未来其他远程环境不是核心数据库枚举值，而是
remote-target plugin 提供的 target provider。

```sql
CREATE TABLE workspace_locations (
    id              TEXT PRIMARY KEY,

    -- Known values: local_filesystem, ssh, remote_target.
    -- New location kinds are validated by the supported domain model, not SQLite.
    location_kind   TEXT NOT NULL,

    plugin_id       TEXT,

    locator_version INTEGER NOT NULL DEFAULT 1
        CHECK (locator_version > 0),
    locator_json    TEXT NOT NULL,

    created_at      INTEGER NOT NULL,
    updated_at      INTEGER NOT NULL,
    CHECK (
        (location_kind = 'remote_target' AND plugin_id IS NOT NULL)
        OR
        (location_kind <> 'remote_target' AND plugin_id IS NULL)
    )
);
```

这些已知值只是当前版本领域模型支持的集合，SQL 注释不是数据库约束。Repository
读取未知的 `location_kind` 时必须返回 typed unsupported/unknown location 错误，不能
把未知值当成本地路径或静默降级。新增核心支持类型时只需要更新领域模型和对应
adapter，不需要修改这张表。

领域层使用强类型 tagged union：

```text
WorkspaceLocation =
    LocalFilesystem {
        path
    }

    Ssh {
        connection_ref,
        path
    }

    RemoteTarget {
        plugin_id,
        target_ref,
        locator
    }
```

`locator_json` 只能由对应的 database repository 和 remote-target plugin 编码、校验和
解析，不能直接暴露给普通 domain 或 application 调用方。Core 只根据 `plugin_id` 选择
插件，不解析插件专属 locator。

`connection_ref`、`target_ref` 和 `plugin_id` 是外部资源引用或插件身份，不是凭据本身。
密码、私钥、token 和其他 Secret 不得写入 Workspace、Location 或普通错误诊断。

remote-target plugin 可以将 Dev Container、云端开发环境或 remote agent sandbox
统一实现为 RemoteTarget。新增这类环境时只需新增插件，不修改 Workspace 表的枚举。

remote-target plugin 至少负责：

- 校验和解析自己的 locator；
- 创建、观察、恢复和销毁 target；
- 将 target 转换为可供 Session/Agent 使用的执行与文件访问能力；
- 隔离 provider 专属状态和错误；
- 保证 credentials 只通过外部 secret/connection 引用访问。

## 8. Workspace Provisioning

Workspace 的位置和 Workspace 的创建过程是两个不同事实。未来的远程 Workspace 可能
尚未创建，或创建后暂时不可访问，因此使用独立 provisioning 表：

```sql
CREATE TABLE workspace_provisioning (
    workspace_id        TEXT PRIMARY KEY REFERENCES workspaces(id),

    -- Known values: local_git, ssh, remote_target.
    -- New provisioners are validated by the supported domain model, not SQLite.
    provisioner_kind    TEXT NOT NULL,

    plugin_id           TEXT,

    requested_revision  TEXT,
    requested_branch    TEXT,

    actual_revision     TEXT,
    actual_branch       TEXT,

    requested_locator_json TEXT,
    actual_locator_json    TEXT,

    -- Known values: pending, provisioning, ready, failed, destroying, destroyed.
    -- New provisioning states are validated by the supported domain model, not SQLite.
    state               TEXT NOT NULL,

    last_error_code     TEXT,
    created_at          INTEGER NOT NULL,
    updated_at          INTEGER NOT NULL,

    CHECK (
        (provisioner_kind = 'remote_target' AND plugin_id IS NOT NULL)
        OR
        (provisioner_kind <> 'remote_target' AND plugin_id IS NULL)
    )
);
```

这些已知值只是当前版本 provisioner 支持的集合，SQL 注释不是数据库约束。Repository
读取未知的 `provisioner_kind` 或 `state` 时必须返回 typed unsupported/unknown 状态，
不能继续执行可能破坏远程资源的操作。新增值时由领域模型、provisioner 和 adapter
共同升级，不需要修改这张表。

`requested_*` 表示用户或上层服务的意图，`actual_*` 表示 provisioner 已经确认的结果。
`requested_locator_json` 和 `actual_locator_json` 只在 `remote_target` 下由对应插件
解释。不能在没有 provisioner 确认的情况下把 Workspace 标记为 `active`。

三类 provisioner 的边界是：

- `local_git`：在 `local_filesystem` Location 中创建或管理本地 Git checkout；
- `ssh`：在 SSH Location 中准备和管理远程目录；
- `remote_target`：委托给 `plugin_id` 对应的 remote-target plugin。

如果未来插件需要大量专有配置，可以由插件维护版本化 configuration，但必须保持：

- plugin 专有配置不进入 Project、Session 或 WorkflowRun；
- provisioning state 不伪装成 Session 状态；
- provisioning failure 不改变 WorkflowRun 的业务结果；
- target 的销毁操作必须可以重试，并保留明确的失败状态。

## 9. Worktree Schema

Worktree 只保存本地 Git worktree 的实现信息：

```sql
CREATE TABLE worktrees (
    workspace_id    TEXT PRIMARY KEY REFERENCES workspaces(id),

    branch_name     TEXT,
    base_commit_id  TEXT CHECK (
        base_commit_id IS NULL OR base_commit_id <> ''
    ),

    created_at      INTEGER NOT NULL,
    updated_at      INTEGER NOT NULL,
    is_deleted      INTEGER NOT NULL DEFAULT 0
        CHECK (is_deleted IN (0, 1))
);
```

实际路径由 `workspace_locations` 提供。对于 `local_git` Workspace，location 的
backend 必须是 `local_filesystem`，provisioning provider 可以是 `local_git`。

不是所有 Workspace 都有 Worktree：

```text
Project main checkout        → 无 Worktree
独立本地 Git worktree        → 有 Worktree
SSH workspace                → 无 Worktree
Dev Container workspace      → 无 Worktree
Remote agent sandbox         → 无 Worktree
```

## 10. Session Schema

Session 直接绑定 Workspace：

```sql
CREATE TABLE sessions (
    id                      TEXT PRIMARY KEY,
    workspace_id            TEXT NOT NULL REFERENCES workspaces(id),
    title                   TEXT,
    agent_cli               TEXT NOT NULL,
    agent_session_id        TEXT NOT NULL,
    history_degraded_reason TEXT,
    status                  INTEGER NOT NULL DEFAULT 0,
    created_at              INTEGER NOT NULL,
    updated_at              INTEGER NOT NULL,
    is_deleted              INTEGER NOT NULL DEFAULT 0
        CHECK (is_deleted IN (0, 1))
);
```

Session 的 admission 和 status 更新必须检查 Workspace 与 Project 都仍然可见：

```text
Session.is_deleted = 0
Workspace.is_deleted = 0
Workspace.lifecycle = active
Project.is_deleted = 0
```

Session 的实时 Idle、Working 和 StoppingTurn 仍然是运行时观察结果，不会变成数据库事实。

## 11. WorkflowRun Schema

WorkflowRun 直接绑定 Workspace，并直接保存展示名称：

```sql
CREATE TABLE workflow_runs (
    id              TEXT PRIMARY KEY,
    workspace_id    TEXT NOT NULL REFERENCES workspaces(id),
    workflow_id     TEXT NOT NULL REFERENCES workflows(id),
    snapshot_id     TEXT NOT NULL REFERENCES workflow_snapshots(id),

    name            TEXT NOT NULL,
    run_status      INTEGER NOT NULL,
    state           TEXT,
    input           TEXT,
    output          TEXT,
    error           TEXT,
    payload         TEXT,
    started_at      INTEGER,
    finished_at     INTEGER,
    created_at      INTEGER NOT NULL,
    updated_at      INTEGER NOT NULL,
    is_deleted      INTEGER NOT NULL DEFAULT 0
        CHECK (is_deleted IN (0, 1))
);

CREATE INDEX idx_workflow_runs_workspace
    ON workflow_runs(workspace_id, created_at, id);
```

WorkflowRun 的删除只影响自身、node runs 和该运行创建的 Session。不能因为删除
WorkflowRun 就删除一个仍被其他 Session 或 WorkflowRun 使用的 Workspace。

WorkflowRun 不拥有 Workspace。WorkflowRun 可以在 Project main Workspace、isolated
Workspace 或任意 remote Workspace 中执行；Workspace 的创建和销毁由 Workspace
provisioning/lifecycle 单独负责。

## 12. Workflow Node Run

Workflow node run 保留 `run_id` 和可选 `session_id`：

```text
WorkflowRun
└── WorkflowNodeRun
      └── Session (optional for agent/prompt nodes)
```

Node session 的 Workspace 必须与 WorkflowRun 的 Workspace 相同。这个约束在 workflow
engine 的 transaction 中校验，避免一个 node 绑到其他 Workspace：

```sql
SELECT 1
FROM workflow_runs AS wr
JOIN sessions AS s ON s.id = ?1
WHERE wr.id = ?2
  AND wr.workspace_id = s.workspace_id;
```

## 13. 生命周期和事务边界

### 13.1 创建 Project

Project 创建和 main Workspace 创建必须在同一个 database transaction 中完成：

```text
insert Project
insert main Workspace(id = new workspace id)
insert Location
insert Provisioning
commit
```

如果本地目录或远程环境尚未创建，Workspace 状态保持 `provisioning`，不能让 Session
或 WorkflowRun 进入执行阶段。

### 13.2 创建 isolated Workspace

创建过程分为：

```text
创建 Workspace
创建 Location
创建 Provisioning(pending)
调用 provider
记录 actual locator/revision
标记 Workspace active
```

数据库 transaction 不能与远程 provider 或文件系统操作合并。必须通过 durable
provisioning state 支持崩溃恢复和重试。

### 13.3 创建 Session

Session 只接受 `workspace_id`。创建前必须验证 Workspace active 且所属 Project 可见。

### 13.4 创建 WorkflowRun

WorkflowRun 创建前必须验证：

- Workspace active；
- Workflow 和 snapshot 可见；
- snapshot 属于指定 Workflow；
- name 已经由调用方确定；
- 需要 checkout 时，provisioning 已经 ready。

创建不需要先插入 Worktree。调用方只需提供一个 active 且 provisioning ready
的 Workspace；WorkflowRun 不会因为执行方式而隐式创建或占用一个 Worktree。

### 13.5 删除 Project

Project 删除必须在一个 immediate transaction 中软删除：

```text
Project
  → Workspace
      → Session
      → WorkflowRun
      → WorkflowNodeRun
      → Worktree
```

远程 provider 的实际销毁不能在 SQLite transaction 内执行。数据库先记录
`retiring/destroying` 状态，后台清理成功后再进入 `deleted/destroyed`。

### 13.6 删除 Workspace

Workspace 只有在以下条件满足后才可以进入最终 deleted：

- 没有 active Session；
- 没有运行中的 WorkflowRun；
- 没有待完成的 provisioning operation；
- local Git cleanup 或 remote provider cleanup 已成功，或者进入明确的
  `manual_attention` 状态。

删除 Workspace 不得只删除 SQLite 行而遗留本地 worktree、容器或远程 sandbox。

## 14. Remote Backend 设计原则

### 14.1 Backend 与 Workspace 分离

Workspace 表达“哪个工作环境”，backend 表达“如何访问或创建它”：

```text
Workspace
  ├── Location: 在哪里
  └── Provisioning: 如何创建
```

未来可以新增 remote-target plugin，例如：

- AWS 开发环境或 sandbox；
- Azure 开发环境或 sandbox；
- GCS-backed 工作环境；
- Dev Container；
- remote agent sandbox；
- 其他专用开发环境 provider。

这些 provider 都使用 `location_kind = remote_target`，不应新增核心 location 枚举值，
也不应修改 Session 或 WorkflowRun 表。

### 14.2 凭据隔离

Workspace 数据库只保存引用：

```text
connection_ref
plugin_id
target_ref
```

凭据由独立 credential/secret 系统管理。数据库、Effect state、WorkflowRun payload、
Session history 和错误消息都不能保存 Secret 明文。

### 14.3 Agent sandbox 不等于 Session

Remote agent sandbox 是 Workspace backend，不是某一个 Session 的临时字段。

一个 sandbox 可以支持多个 Session 或 WorkflowRun；Agent session 的生命周期不能
决定 sandbox 的身份。反过来，sandbox 销毁也必须经过 Workspace lifecycle，而不是
简单删除 Session。

### 14.4 Adapter 边界

Application 和 domain 只依赖抽象的 Workspace location/provisioning port。具体的
SSH、Docker、Dev Container 或 remote agent provider 实现属于 infrastructure adapter。

Effect 后续应以 Workspace location adapter 访问目标 surface，不把 remote location
转换成本地路径。

## 15. 设计不变量

1. Project 是逻辑仓库，不拥有 checkout 路径。
2. Workspace 是 Session、WorkflowRun 和 Effect 的工作环境边界。
3. 所有 Workspace 使用独立的 Workspace ID。
4. Project main Workspace 通过 `project_id + workspace_kind` 唯一定位。
5. Project-root Session 直接使用 Project main Workspace。
6. Workspace 的 `display_name` 是用户可见名称，不是 Git branch 或 revision 的事实。
7. WorkflowRun 可以在任意 active Workspace 中执行，不隐式拥有 Worktree。
8. WorkflowRun 直接保存 `workspace_id` 和 `name`。
9. Worktree 只表示本地 Git worktree，不表示所有 Workspace。
10. Remote location 不能伪装成本地路径。
11. 删除 Session 或 WorkflowRun 不得隐式删除共享 Workspace。
12. Workspace provisioning 的期望状态和实际状态必须分开保存。
13. 远程凭据只能通过引用访问，不能进入普通业务表。
