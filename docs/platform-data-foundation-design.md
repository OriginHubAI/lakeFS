# 平台数据底座设计文档（一期）

> 基于 lakeFS 架构的数据集管理与版本控制系统设计

## 1. 需求概述

| 编号 | 需求 | 优先级 |
|------|------|--------|
| R1 | 数据集 CRUD（创建、删除、改名）及属性查看（创建者、创建时间、最近修改时间、占用空间） | P0 |
| R2 | 数据集中存储各种结构化和非结构化文件 | P0 |
| R3 | 文件操作（添加、删除、更新、列表含属性） | P0 |
| R4 | 每次操作生成版本信息 | P0 |
| R5 | 查看数据集的所有版本 | P0 |
| R6 | 回滚数据集到指定版本 | P0 |
| R7 | Data Lineage（任务-数据集关系追踪、输入输出追踪） | P1 |

## 2. lakeFS 能力与需求匹配分析

### 2.1 核心概念映射

| 平台概念 | lakeFS 概念 | 说明 |
|----------|------------|------|
| 数据集 (Dataset) | Repository | 一个 lakeFS repository 对应一个数据集，天然隔离 |
| 文件 (File) | Object | lakeFS 的 object 支持任意类型文件的存储和管理 |
| 版本 (Version) | Commit | 每次 commit 生成不可变的版本快照 |
| 版本列表 | Commit Log | lakeFS 的 commit log 提供完整的版本历史 |
| 版本回滚 | Revert / Reset | lakeFS 原生支持 revert（生成逆向 commit）和 hard reset |
| 数据集属性 | Repository + Repository Metadata | 创建时间原生支持，其他属性通过 metadata 扩展 |

### 2.2 逐项可行性分析

| 需求 | 可行性 | lakeFS 原生支持 | 需要扩展 |
|------|--------|----------------|---------|
| R1 数据集 CRUD | ✅ 完全可行 | 创建/删除 repository、repository metadata | 改名(metadata)、占用空间(聚合计算)、最近修改时间(commit log) |
| R2 存储文件 | ✅ 完全可行 | object storage 支持任意文件类型 | 无 |
| R3 文件操作 | ✅ 完全可行 | upload/delete/list/stat object | 无 |
| R4 版本生成 | ✅ 完全可行 | commit 机制 | 自动 commit 封装 |
| R5 版本列表 | ✅ 完全可行 | log commits API | 无 |
| R6 版本回滚 | ✅ 完全可行 | revert commit / hard reset branch | 无 |
| R7 Data Lineage | ⚠️ 需要扩展 | commit metadata 可携带自定义信息 | 需要新建 lineage 服务层 |

**结论：7 项需求中前 6 项 lakeFS 原生架构可以直接支持或通过轻量封装实现，第 7 项 Data Lineage 需要在 lakeFS 之上构建额外的元数据服务层。**

## 3. 整体架构设计

```
┌─────────────────────────────────────────────────────────────────┐
│                     Platform API Gateway                        │
│                   (RESTful API / gRPC)                          │
├────────────┬──────────────┬──────────────┬──────────────────────┤
│  Dataset   │    File      │   Version    │     Lineage          │
│  Service   │   Service    │   Service    │     Service          │
├────────────┴──────────────┴──────────────┴──────────────────────┤
│                    Platform Service Layer                       │
│          (业务逻辑封装、自动版本管理、权限控制)                       │
├─────────────────────────────────────┬──────────────────────────-┤
│         lakeFS SDK / API            │     Lineage Store         │
│   (Repository, Object, Commit,      │   (PostgreSQL / KV)       │
│    Branch, Tag, Diff, Merge)        │                           │
├─────────────────────────────────────┤                           │
│            lakeFS Server            │                           │
│  ┌──────────┬──────────┬─────────┐  │                           │
│  │ Catalog  │ Graveler │  Auth   │  │                           │
│  │ (对象管理)│(版本引擎) │(认证授权) │  │                           │
│  ├──────────┴──────────┴─────────┤  │                           │
│  │     KV Store (PostgreSQL)     │  │                           │
│  │     Block Storage (S3/MinIO)  │  │                           │
│  └───────────────────────────────┘  │                           │
└─────────────────────────────────────┴───────────────────────────┘
```

### 3.1 分层职责

| 层级 | 职责 |
|------|------|
| **Platform API Gateway** | 对外暴露统一的 REST API，请求路由、认证鉴权、限流 |
| **Platform Service Layer** | 封装 lakeFS 操作为业务语义，实现自动版本管理、属性聚合、Lineage 记录 |
| **lakeFS SDK/API** | 调用 lakeFS 原生 API 实现数据存储和版本控制 |
| **Lineage Store** | 独立存储 lineage 元数据（任务、数据集关系、DAG 信息） |

## 4. 详细设计

### 4.1 R1: 数据集管理

#### 4.1.1 概念映射

一个**数据集**对应一个 lakeFS **repository**。每个 repository 使用独立的存储命名空间，天然实现数据隔离。数据集的扩展属性存储在 repository metadata 中。

#### 4.1.2 数据模型

```
Dataset:
  id:                string       # lakeFS repository name (唯一标识)
  display_name:      string       # 数据集显示名称 (存储在 repository metadata)
  creator:           string       # 创建者 (存储在 repository metadata)
  creation_time:     timestamp    # 创建时间 (lakeFS repository.creation_date)
  last_modified_time: timestamp   # 最近修改时间 (从最新 commit 的 creation_date 获取)
  size_bytes:        int64        # 占用空间 (聚合计算所有 object 的 size)
  description:       string       # 数据集描述 (存储在 repository metadata)
  storage_namespace: string       # lakeFS storage namespace
  default_branch:    string       # 默认分支 (默认 "main")
```

#### 4.1.3 API 设计

##### 创建数据集

```
POST /api/v1/datasets

Request:
{
  "name": "image-training-set",
  "display_name": "图像训练集",
  "creator": "user-001",
  "description": "用于模型训练的图像数据集",
  "storage_namespace": "s3://bucket/datasets/image-training-set"
}

Response: 201 Created
{
  "id": "image-training-set",
  "display_name": "图像训练集",
  "creator": "user-001",
  "creation_time": "2026-03-04T10:00:00Z",
  "last_modified_time": "2026-03-04T10:00:00Z",
  "size_bytes": 0,
  "storage_namespace": "s3://bucket/datasets/image-training-set",
  "default_branch": "main"
}
```

**实现流程：**

```
1. 调用 lakeFS: POST /api/v1/repositories
   body: {
     name: "image-training-set",
     storage_namespace: "s3://bucket/datasets/image-training-set",
     default_branch: "main"
   }
2. 调用 lakeFS: POST /api/v1/repositories/{repo}/metadata  (internal API)
   body: {
     metadata: {
       "platform.display_name": "图像训练集",
       "platform.creator": "user-001",
       "platform.description": "用于模型训练的图像数据集",
       "platform.created_by_platform": "true"
     }
   }
3. 返回组合后的 Dataset 对象
```

##### 删除数据集

```
DELETE /api/v1/datasets/{dataset_id}

Response: 204 No Content
```

**实现流程：** 直接调用 lakeFS `DELETE /api/v1/repositories/{repository}`。

##### 修改数据集名称

```
PATCH /api/v1/datasets/{dataset_id}

Request:
{
  "display_name": "新的数据集名称"
}

Response: 200 OK
{
  "id": "image-training-set",
  "display_name": "新的数据集名称",
  ...
}
```

**实现流程：** 调用 lakeFS `POST /api/v1/repositories/{repo}/metadata` 更新 `platform.display_name`。

> 注意：lakeFS 的 repository name 创建后不可变（用作存储路径标识），因此"改名"操作修改的是 metadata 中的 `display_name`，不影响底层存储路径。

##### 查看数据集属性

```
GET /api/v1/datasets/{dataset_id}

Response: 200 OK
{
  "id": "image-training-set",
  "display_name": "图像训练集",
  "creator": "user-001",
  "creation_time": "2026-03-04T10:00:00Z",
  "last_modified_time": "2026-03-10T15:30:00Z",
  "size_bytes": 1073741824,
  "description": "用于模型训练的图像数据集",
  "file_count": 1500,
  "version_count": 12
}
```

**实现流程：**

```
并行调用:
1. lakeFS: GET /api/v1/repositories/{repo}           → creation_date
2. lakeFS: GET /api/v1/repositories/{repo}/metadata   → display_name, creator, description
3. lakeFS: GET /api/v1/refs/{ref}/commits?amount=1    → 最新 commit 时间 = last_modified_time
4. lakeFS: GET /api/v1/refs/{ref}/objects/ls           → 遍历计算总 size 和 file_count
   (或使用缓存/预计算值，见 4.1.4)

组装返回 Dataset 对象
```

#### 4.1.4 数据集大小计算策略

遍历所有对象计算大小在数据量大时开销较高，采用**写时更新 + 异步校准**策略：

1. **写时更新**：每次文件添加/删除/更新操作时，在 commit metadata 中记录增量变化 `delta_size_bytes`，Platform Service 维护一个 `cached_size_bytes` 存储在 repository metadata 中。
2. **异步校准**：定时任务（如每小时一次）遍历数据集对象列表，精确计算 size 并更新缓存。
3. **读取时**：优先返回 `cached_size_bytes`，如不存在则触发一次全量计算。

##### 列出数据集

```
GET /api/v1/datasets?prefix=&after=&amount=20

Response: 200 OK
{
  "results": [
    {
      "id": "image-training-set",
      "display_name": "图像训练集",
      "creator": "user-001",
      "creation_time": "2026-03-04T10:00:00Z",
      "size_bytes": 1073741824
    },
    ...
  ],
  "pagination": {
    "has_more": true,
    "next_offset": "image-training-set"
  }
}
```

**实现流程：** 调用 lakeFS `GET /api/v1/repositories`，过滤出含有 `platform.created_by_platform` metadata 标记的 repository，逐个获取 metadata 组装返回。

---

### 4.2 R2 & R3: 文件存储与操作

#### 4.2.1 概念映射

数据集中的文件直接对应 lakeFS 中的 **Object**。lakeFS 的 object 存储不限制文件类型，支持任意二进制文件，天然满足"各种非结构化文件和结构化文件"的需求。

#### 4.2.2 API 设计

##### 添加/上传文件

```
POST /api/v1/datasets/{dataset_id}/files?path=data/images/cat_001.jpg
Content-Type: multipart/form-data

body: (文件二进制内容)

Response: 201 Created
{
  "path": "data/images/cat_001.jpg",
  "size_bytes": 204800,
  "checksum": "abc123def456",
  "content_type": "image/jpeg",
  "created_at": "2026-03-04T10:05:00Z",
  "version_id": "commit-sha-xxxx"
}
```

**实现流程：**

```
1. lakeFS: POST /api/v1/repositories/{repo}/branches/main/objects?path=data/images/cat_001.jpg
   content: (文件内容)
2. lakeFS: POST /api/v1/repositories/{repo}/branches/main/commits
   body: {
     message: "Add file: data/images/cat_001.jpg",
     metadata: {
       "platform.operation": "add_file",
       "platform.file_path": "data/images/cat_001.jpg",
       "platform.operator": "user-001"
     }
   }
3. 更新 repository metadata 中的 cached_size_bytes (增量)
4. 返回文件信息 + commit ID 作为 version_id
```

##### 批量添加文件

```
POST /api/v1/datasets/{dataset_id}/files/batch
Content-Type: multipart/form-data

files[0]: (path=data/a.csv, content=...)
files[1]: (path=data/b.csv, content=...)

Response: 201 Created
{
  "files": [...],
  "version_id": "commit-sha-xxxx"
}
```

**实现流程：** 上传多个 object 后执行一次 commit，生成单一版本。

##### 删除文件

```
DELETE /api/v1/datasets/{dataset_id}/files?path=data/images/cat_001.jpg

Response: 200 OK
{
  "version_id": "commit-sha-yyyy"
}
```

**实现流程：**

```
1. lakeFS: DELETE /api/v1/repositories/{repo}/branches/main/objects?path=data/images/cat_001.jpg
2. lakeFS: POST /api/v1/repositories/{repo}/branches/main/commits
   body: {
     message: "Delete file: data/images/cat_001.jpg",
     metadata: {
       "platform.operation": "delete_file",
       "platform.file_path": "data/images/cat_001.jpg",
       "platform.operator": "user-001"
     }
   }
3. 更新 cached_size_bytes (减量)
4. 返回 version_id
```

##### 更新文件

```
PUT /api/v1/datasets/{dataset_id}/files?path=data/images/cat_001.jpg
Content-Type: multipart/form-data

body: (新文件内容)

Response: 200 OK
{
  "path": "data/images/cat_001.jpg",
  "size_bytes": 210000,
  "checksum": "new-checksum",
  "version_id": "commit-sha-zzzz"
}
```

**实现流程：** 与添加文件相同（lakeFS 的 upload object 是 upsert 语义），commit message 标记为 `update_file`。

##### 查看文件列表

```
GET /api/v1/datasets/{dataset_id}/files?prefix=data/images/&after=&amount=100

Response: 200 OK
{
  "results": [
    {
      "path": "data/images/cat_001.jpg",
      "size_bytes": 204800,
      "checksum": "abc123def456",
      "content_type": "image/jpeg",
      "last_modified": "2026-03-04T10:05:00Z",
      "metadata": {
        "label": "cat"
      }
    },
    ...
  ],
  "pagination": {
    "has_more": true,
    "next_offset": "data/images/cat_100.jpg"
  }
}
```

**实现流程：** 调用 lakeFS `GET /api/v1/repositories/{repo}/refs/main/objects/ls?prefix=data/images/`，返回 `ObjectStats` 列表，包含 `path`, `size_bytes`, `mtime`, `checksum`, `content_type`, `metadata`。

##### 下载文件

```
GET /api/v1/datasets/{dataset_id}/files/content?path=data/images/cat_001.jpg

Response: 200 OK
Content-Type: image/jpeg
(文件二进制内容)
```

**实现流程：** 调用 lakeFS `GET /api/v1/repositories/{repo}/refs/main/objects?path=data/images/cat_001.jpg`。

##### 按版本下载文件

```
GET /api/v1/datasets/{dataset_id}/files/content?path=data/images/cat_001.jpg&version=commit-sha-xxxx

Response: 200 OK
(该版本的文件内容)
```

**实现流程：** 调用 lakeFS `GET /api/v1/repositories/{repo}/refs/{commit_id}/objects?path=...`。lakeFS 的 ref 可以是 branch 名、tag 名或 commit ID，因此历史版本的文件访问是原生支持的。

---

### 4.3 R4: 自动版本管理

#### 4.3.1 设计原则

lakeFS 基于 Git 模型，需要显式调用 commit 来创建版本。为满足"每次操作都生成版本"的需求，Platform Service Layer 在每次文件操作后自动执行 commit。

#### 4.3.2 版本生成策略

```
用户调用 Platform API
  → Platform Service 执行 lakeFS 文件操作 (stage changes)
  → Platform Service 自动执行 lakeFS commit (生成版本)
  → 返回 version_id (commit ID) 给用户
```

**commit 信息规范：**

```json
{
  "message": "[{operation}] {description}",
  "metadata": {
    "platform.operation": "add_file | delete_file | update_file | batch_add | batch_delete",
    "platform.operator": "user-id",
    "platform.file_paths": "path1,path2,...",
    "platform.timestamp": "2026-03-04T10:05:00Z",
    "platform.client_ip": "10.0.0.1",
    "platform.task_id": "task-xxx (可选，关联 lineage)"
  }
}
```

#### 4.3.3 批量操作的版本策略

对于批量操作（如同时上传多个文件），先完成所有 object 的 stage，然后执行一次 commit，保证批量操作的原子性：

```
批量上传 10 个文件:
  → stage object 1
  → stage object 2
  → ...
  → stage object 10
  → commit (生成 1 个版本，包含这 10 个文件的变更)
```

---

### 4.4 R5: 版本列表查看

#### 4.4.1 API 设计

```
GET /api/v1/datasets/{dataset_id}/versions?after=&amount=20

Response: 200 OK
{
  "results": [
    {
      "version_id": "abc123",
      "message": "[add_file] Add file: data/images/cat_001.jpg",
      "operator": "user-001",
      "operation": "add_file",
      "timestamp": "2026-03-04T10:05:00Z",
      "file_paths": ["data/images/cat_001.jpg"],
      "parent_versions": ["def456"]
    },
    {
      "version_id": "def456",
      "message": "[batch_add] Batch upload 50 training images",
      "operator": "user-002",
      "operation": "batch_add",
      "timestamp": "2026-03-03T14:20:00Z",
      "file_paths": ["data/images/img_001.jpg", "..."],
      "parent_versions": ["ghi789"]
    }
  ],
  "pagination": {
    "has_more": true,
    "next_offset": "def456"
  }
}
```

**实现流程：** 调用 lakeFS `GET /api/v1/repositories/{repo}/refs/main/commits`，从返回的 `CommitLog` 中提取 commit ID、message、metadata、creation_date 和 parents，组装为版本列表。

#### 4.4.2 版本详情

```
GET /api/v1/datasets/{dataset_id}/versions/{version_id}

Response: 200 OK
{
  "version_id": "abc123",
  "message": "[add_file] Add file: data/images/cat_001.jpg",
  "operator": "user-001",
  "operation": "add_file",
  "timestamp": "2026-03-04T10:05:00Z",
  "changes": [
    {
      "type": "added",
      "path": "data/images/cat_001.jpg",
      "size_bytes": 204800
    }
  ],
  "parent_versions": ["def456"]
}
```

**实现流程：**

```
1. lakeFS: GET /api/v1/repositories/{repo}/commits/{commit_id}   → commit 信息
2. lakeFS: GET /api/v1/repositories/{repo}/refs/{commit_id}/diff/{commit_id~1}
   → 获取该版本相对于上一版本的变更列表 (added/removed/changed)
3. 组装返回
```

---

### 4.5 R6: 版本回滚

#### 4.5.1 回滚策略

提供两种回滚方式：

| 方式 | lakeFS 操作 | 语义 | 适用场景 |
|------|------------|------|---------|
| **安全回滚** (revert) | `POST .../branches/main/revert` | 创建一个新的 commit 来撤销目标 commit 的变更，保留完整历史 | 生产环境、需要审计追踪 |
| **强制重置** (reset) | `PUT .../branches/main/hard_reset` | 将 branch 指针直接回退到目标 commit，丢弃之后的变更 | 开发/测试环境、需要彻底回退 |

一期推荐使用**安全回滚**，确保所有操作可追溯。

#### 4.5.2 API 设计

##### 安全回滚（推荐）

```
POST /api/v1/datasets/{dataset_id}/versions/{version_id}/rollback

Request:
{
  "strategy": "revert",
  "operator": "user-001"
}

Response: 200 OK
{
  "new_version_id": "new-commit-sha",
  "message": "Rollback to version abc123",
  "rolled_back_to": "abc123",
  "changes": [
    {"type": "reverted", "path": "data/images/new_file.jpg"},
    {"type": "restored", "path": "data/images/old_file.jpg"}
  ]
}
```

**实现流程（安全回滚 — 回滚到指定版本）：**

lakeFS 的 `revert` 仅撤销单个 commit。要回滚到指定历史版本，需要按逆序 revert 从当前到目标版本之间的所有 commit，或者使用 `hard_reset`：

```
方案A: 逐个 Revert（保留完整历史）
  1. 获取从目标版本到当前最新版本之间的所有 commit 列表
  2. 从最新的 commit 开始，逆序逐个 revert
  3. 每个 revert 生成一个新的 commit

方案B: Hard Reset + 重新 Commit（推荐，简洁）
  1. 获取目标版本的文件快照
  2. lakeFS: PUT /api/v1/repositories/{repo}/branches/main/hard_reset
     body: { ref: "target_commit_id" }
  3. lakeFS: POST /api/v1/repositories/{repo}/branches/main/commits
     body: {
       message: "Rollback to version {target_version_id}",
       metadata: {
         "platform.operation": "rollback",
         "platform.target_version": "target_commit_id",
         "platform.operator": "user-001"
       }
     }
  注意：hard_reset 是实验性 API，需评估稳定性

方案C: 基于 Diff 的精确回滚（最稳健，一期推荐）
  1. lakeFS: GET /api/v1/repositories/{repo}/refs/{current_head}/diff/{target_commit_id}
     → 获取当前版本与目标版本之间的差异
  2. 根据 diff 结果，执行对应操作：
     - added (当前多出的文件): DELETE
     - removed (当前缺少的文件): 从目标版本复制回来
     - changed (内容变化的文件): 从目标版本复制回来
  3. commit 所有变更，message 标记为 rollback
```

---

### 4.6 R7: Data Lineage

#### 4.6.1 概念模型

Data Lineage 需要追踪以下关系：

```
┌──────────┐     produces      ┌──────────┐
│   Task   │──────────────────>│ Dataset  │
│  (任务)   │                   │ (数据集)  │
└──────────┘                   └──────────┘
     │ consumes                      │
     │                               │
     ▼                               ▼
┌──────────┐                   ┌──────────┐
│ Dataset  │                   │ Version  │
│ (输入)    │                   │ (版本)    │
└──────────┘                   └──────────┘
```

核心实体关系：

- **Task（任务）**：数据处理的执行单元，可包含子任务
- **TaskRun（任务执行）**：任务的一次具体执行
- **Dataset（数据集）**：已由 R1 定义
- **DatasetVersion（数据集版本）**：已由 R4 定义
- **LineageEdge（血缘边）**：记录任务输入/输出关系

#### 4.6.2 数据模型

Lineage 数据存储在独立的 PostgreSQL 表中（复用 lakeFS 的 PostgreSQL 实例）：

```sql
-- 任务定义
CREATE TABLE lineage_tasks (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    task_type       VARCHAR(50),           -- 'training', 'preprocessing', 'evaluation', etc.
    parent_task_id  UUID REFERENCES lineage_tasks(id),  -- 父任务（支持子任务）
    creator         VARCHAR(255) NOT NULL,
    created_at      TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMP NOT NULL DEFAULT NOW(),
    metadata        JSONB DEFAULT '{}'     -- 扩展属性
);

-- 任务执行记录
CREATE TABLE lineage_task_runs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    task_id         UUID NOT NULL REFERENCES lineage_tasks(id),
    status          VARCHAR(20) NOT NULL DEFAULT 'running',  -- running, completed, failed
    started_at      TIMESTAMP NOT NULL DEFAULT NOW(),
    finished_at     TIMESTAMP,
    error_message   TEXT,
    metadata        JSONB DEFAULT '{}'
);

-- 血缘边：任务执行与数据集版本的关系
CREATE TABLE lineage_edges (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    task_run_id     UUID NOT NULL REFERENCES lineage_task_runs(id),
    dataset_id      VARCHAR(255) NOT NULL,     -- lakeFS repository name
    dataset_version VARCHAR(255),              -- lakeFS commit ID
    direction       VARCHAR(10) NOT NULL,      -- 'input' or 'output'
    created_at      TIMESTAMP NOT NULL DEFAULT NOW(),
    
    UNIQUE (task_run_id, dataset_id, direction)
);

-- 索引
CREATE INDEX idx_lineage_edges_task_run ON lineage_edges(task_run_id);
CREATE INDEX idx_lineage_edges_dataset ON lineage_edges(dataset_id);
CREATE INDEX idx_lineage_tasks_parent ON lineage_tasks(parent_task_id);
CREATE INDEX idx_lineage_task_runs_task ON lineage_task_runs(task_id);
```

#### 4.6.3 API 设计

##### 注册任务

```
POST /api/v1/lineage/tasks

Request:
{
  "name": "image-preprocessing-pipeline",
  "description": "图像预处理流水线",
  "task_type": "preprocessing",
  "creator": "user-001",
  "parent_task_id": null,
  "metadata": {
    "framework": "pytorch",
    "version": "2.0"
  }
}

Response: 201 Created
{
  "id": "task-uuid-001",
  "name": "image-preprocessing-pipeline",
  ...
}
```

##### 注册子任务

```
POST /api/v1/lineage/tasks

Request:
{
  "name": "resize-images",
  "description": "将图像缩放到 224x224",
  "task_type": "preprocessing",
  "parent_task_id": "task-uuid-001",
  "creator": "user-001"
}

Response: 201 Created
{
  "id": "task-uuid-002",
  ...
}
```

##### 开始任务执行

```
POST /api/v1/lineage/tasks/{task_id}/runs

Request:
{
  "metadata": {
    "gpu": "A100",
    "batch_size": 64
  }
}

Response: 201 Created
{
  "run_id": "run-uuid-001",
  "task_id": "task-uuid-001",
  "status": "running",
  "started_at": "2026-03-04T10:00:00Z"
}
```

##### 记录输入/输出数据集

```
POST /api/v1/lineage/runs/{run_id}/edges

Request:
{
  "edges": [
    {
      "dataset_id": "raw-images",
      "dataset_version": "commit-sha-aaa",
      "direction": "input"
    },
    {
      "dataset_id": "processed-images",
      "dataset_version": "commit-sha-bbb",
      "direction": "output"
    }
  ]
}

Response: 201 Created
```

##### 完成任务执行

```
PATCH /api/v1/lineage/runs/{run_id}

Request:
{
  "status": "completed"
}

Response: 200 OK
```

##### 查看数据集的血缘关系

```
GET /api/v1/lineage/datasets/{dataset_id}?direction=both&depth=3

Response: 200 OK
{
  "dataset_id": "processed-images",
  "upstream": [
    {
      "dataset_id": "raw-images",
      "dataset_version": "commit-sha-aaa",
      "produced_by": {
        "task_run_id": "run-uuid-001",
        "task_name": "image-preprocessing-pipeline",
        "task_type": "preprocessing",
        "status": "completed",
        "started_at": "2026-03-04T10:00:00Z",
        "finished_at": "2026-03-04T10:30:00Z"
      }
    }
  ],
  "downstream": [
    {
      "dataset_id": "trained-model",
      "dataset_version": "commit-sha-ccc",
      "consumed_by": {
        "task_run_id": "run-uuid-002",
        "task_name": "model-training",
        "task_type": "training",
        "status": "completed"
      }
    }
  ]
}
```

##### 查看任务的输入输出

```
GET /api/v1/lineage/tasks/{task_id}?include_runs=true

Response: 200 OK
{
  "task": {
    "id": "task-uuid-001",
    "name": "image-preprocessing-pipeline",
    "task_type": "preprocessing",
    "sub_tasks": [
      {
        "id": "task-uuid-002",
        "name": "resize-images"
      },
      {
        "id": "task-uuid-003",
        "name": "normalize-images"
      }
    ]
  },
  "runs": [
    {
      "run_id": "run-uuid-001",
      "status": "completed",
      "inputs": [
        {"dataset_id": "raw-images", "version": "commit-sha-aaa"}
      ],
      "outputs": [
        {"dataset_id": "processed-images", "version": "commit-sha-bbb"}
      ]
    }
  ]
}
```

##### 查看完整 DAG

```
GET /api/v1/lineage/dag?root_dataset_id=final-model&depth=5

Response: 200 OK
{
  "nodes": [
    {"type": "dataset", "id": "raw-images", "name": "原始图像"},
    {"type": "dataset", "id": "processed-images", "name": "处理后图像"},
    {"type": "dataset", "id": "final-model", "name": "最终模型"},
    {"type": "task", "id": "task-001", "name": "预处理流水线"},
    {"type": "task", "id": "task-002", "name": "模型训练"}
  ],
  "edges": [
    {"from": "raw-images", "to": "task-001", "type": "input"},
    {"from": "task-001", "to": "processed-images", "type": "output"},
    {"from": "processed-images", "to": "task-002", "type": "input"},
    {"from": "task-002", "to": "final-model", "type": "output"}
  ]
}
```

#### 4.6.4 与 lakeFS 的集成

Lineage 与 lakeFS 的集成点：

1. **commit metadata 关联**：在通过 lineage API 创建的文件操作中，commit metadata 中记录 `platform.task_id` 和 `platform.run_id`，实现版本与任务执行的双向关联。

2. **Hooks 自动记录**：利用 lakeFS 的 post-commit hook，在 commit 时自动检查 metadata 中是否包含 task 信息，如果包含则自动创建 lineage edge。

```yaml
# _lakefs_actions/lineage-tracker.yaml
name: lineage-tracker
on:
  post-commit:
    branches: ["*"]
hooks:
  - id: record-lineage
    type: webhook
    properties:
      url: "http://platform-service:8080/internal/lineage/on-commit"
```

---

## 5. 技术实现方案

### 5.1 技术栈

| 组件 | 技术选型 | 版本 | 说明 |
|------|---------|------|------|
| Web 框架 | FastAPI | >=0.110 | 高性能异步框架，自带 OpenAPI 文档生成 |
| lakeFS 集成 | lakefs-sdk (官方 Python SDK) | >=1.0 | lakeFS 官方 Python 客户端 |
| ORM | SQLAlchemy | >=2.0 | Lineage 元数据持久化，支持异步 |
| 数据库迁移 | Alembic | >=1.13 | 数据库 Schema 版本管理 |
| Lineage 存储 | PostgreSQL | >=14 | 复用 lakeFS 的 PostgreSQL 实例 |
| 数据校验 | Pydantic | >=2.0 | 请求/响应模型校验（FastAPI 内置集成） |
| 异步 HTTP | httpx | >=0.27 | 异步调用 lakeFS REST API（SDK 不支持的场景） |
| 缓存 | cachetools | >=5.3 | 内存缓存（数据集大小等高频查询） |
| 任务调度 | APScheduler | >=3.10 | 异步校准任务（数据集大小定时重算） |
| 测试 | pytest + pytest-asyncio | - | 单元测试和集成测试 |
| 容器化 | Docker + docker-compose | - | 开发与部署 |
| 代码质量 | ruff + mypy | - | Lint + 类型检查 |

### 5.2 部署架构

```
┌─────────────────┐   HTTP/SDK   ┌────────────────────┐  lakefs-sdk  ┌──────────┐
│  Python Client  │─────────────>│  Platform Service  │─────────────>│  lakeFS  │
│  (SDK/Notebook)  │              │  (FastAPI + Uvicorn)│              │  Server  │
└─────────────────┘              └────────┬───────────┘              └────┬─────┘
                                          │                               │
                                          │ SQLAlchemy (async)            │ KV + Block
                                          ▼                               ▼
                                   ┌──────────────┐               ┌──────────────┐
                                   │ PostgreSQL   │               │ PostgreSQL   │
                                   │ (lineage DB) │               │ (lakeFS KV)  │
                                   └──────────────┘               │ S3/MinIO     │
                                                                  └──────────────┘
```

Platform Service 作为独立 Python 服务部署，通过 `lakefs-sdk` 调用 lakeFS API，Lineage 数据存储在独立的 PostgreSQL schema 中。

### 5.3 项目结构

```
platform-data-service/
├── pyproject.toml                      # 项目元数据 & 依赖管理 (PEP 621)
├── alembic.ini                         # Alembic 迁移配置
├── Dockerfile
├── docker-compose.yml                  # 本地开发：lakeFS + PostgreSQL + MinIO + 本服务
├── .env.example                        # 环境变量示例
│
├── migrations/                         # Alembic 数据库迁移
│   ├── env.py
│   └── versions/
│       └── 001_create_lineage_tables.py
│
├── src/
│   └── platform_data_service/          # 主 Python 包
│       ├── __init__.py
│       ├── main.py                     # FastAPI app 创建 & 启动入口
│       ├── config.py                   # 配置管理 (pydantic-settings)
│       ├── dependencies.py             # FastAPI 依赖注入 (DB session, lakeFS client 等)
│       │
│       ├── models/                     # Pydantic 模型 (API schema)
│       │   ├── __init__.py
│       │   ├── dataset.py              # DatasetCreate, DatasetResponse, DatasetUpdate ...
│       │   ├── file.py                 # FileUploadResponse, FileInfo, FileListResponse ...
│       │   ├── version.py              # VersionInfo, VersionDetail, RollbackRequest ...
│       │   ├── lineage.py              # TaskCreate, TaskRunResponse, LineageEdge, DAGResponse ...
│       │   └── common.py               # PaginatedResponse, ErrorResponse ...
│       │
│       ├── db/                         # 数据库层 (Lineage 持久化)
│       │   ├── __init__.py
│       │   ├── engine.py               # SQLAlchemy async engine & session factory
│       │   ├── tables.py               # SQLAlchemy Table 定义 (lineage_tasks, lineage_task_runs, lineage_edges)
│       │   └── repository.py           # 数据访问层 (CRUD for lineage tables)
│       │
│       ├── lakefs/                     # lakeFS 客户端封装层
│       │   ├── __init__.py
│       │   ├── client.py               # LakeFSClient: 封装 lakefs-sdk，提供高层操作
│       │   └── errors.py               # lakeFS 异常映射为平台异常
│       │
│       ├── services/                   # 业务逻辑层
│       │   ├── __init__.py
│       │   ├── dataset_service.py      # DatasetService: 数据集 CRUD + 属性聚合
│       │   ├── file_service.py         # FileService: 文件操作 + 自动 commit
│       │   ├── version_service.py      # VersionService: 版本列表、详情、回滚
│       │   └── lineage_service.py      # LineageService: 任务管理、血缘记录、DAG 查询
│       │
│       ├── routers/                    # FastAPI 路由层
│       │   ├── __init__.py
│       │   ├── datasets.py             # /api/v1/datasets/...
│       │   ├── files.py                # /api/v1/datasets/{id}/files/...
│       │   ├── versions.py             # /api/v1/datasets/{id}/versions/...
│       │   └── lineage.py              # /api/v1/lineage/...
│       │
│       ├── tasks/                      # 后台任务
│       │   ├── __init__.py
│       │   └── size_calibration.py     # 数据集大小异步校准定时任务
│       │
│       └── exceptions.py               # 全局异常定义 & FastAPI exception handler
│
├── tests/
│   ├── conftest.py                     # pytest fixtures (test DB, mock lakeFS)
│   ├── unit/
│   │   ├── test_dataset_service.py
│   │   ├── test_file_service.py
│   │   ├── test_version_service.py
│   │   └── test_lineage_service.py
│   └── integration/
│       ├── test_dataset_api.py
│       ├── test_file_api.py
│       ├── test_version_api.py
│       └── test_lineage_api.py
│
└── sdk/                                # Python SDK (供外部用户使用)
    ├── pyproject.toml
    └── src/
        └── platform_data_sdk/
            ├── __init__.py
            ├── client.py               # PlatformClient 主入口
            ├── models.py               # SDK 数据模型
            └── exceptions.py           # SDK 异常
```

## 6. Python 实现框架详细设计

本节详细说明 Python 实现框架的核心模块设计，包括关键类、接口定义和模块间交互方式。

### 6.1 配置管理 (`config.py`)

使用 `pydantic-settings` 管理配置，支持环境变量和 `.env` 文件：

```python
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    model_config = {"env_prefix": "PLATFORM_"}

    # FastAPI
    app_title: str = "Platform Data Service"
    debug: bool = False
    host: str = "0.0.0.0"
    port: int = 8080

    # lakeFS
    lakefs_endpoint: str = "http://localhost:8000"
    lakefs_access_key: str = ""
    lakefs_secret_key: str = ""
    lakefs_default_branch: str = "main"
    lakefs_storage_namespace_prefix: str = "s3://platform-data"

    # PostgreSQL (Lineage)
    database_url: str = "postgresql+asyncpg://postgres:postgres@localhost:5432/platform_lineage"

    # 缓存
    cache_ttl_seconds: int = 300
    size_calibration_interval_minutes: int = 60
```

### 6.2 lakeFS 客户端封装层 (`lakefs/client.py`)

封装 `lakefs-sdk`，将 lakeFS 底层操作抽象为平台语义：

```python
import lakefs_sdk
from lakefs_sdk.client import LakeFSClient as _SDKClient

class LakeFSClient:
    """封装 lakefs-sdk，对外提供数据集/文件/版本语义的操作接口。"""

    METADATA_PREFIX = "platform."

    def __init__(self, endpoint: str, access_key: str, secret_key: str):
        configuration = lakefs_sdk.Configuration(
            host=endpoint,
            username=access_key,
            password=secret_key,
        )
        self._client = _SDKClient(configuration)

    # ---- Repository (Dataset) 操作 ----

    def create_repository(
        self, name: str, storage_namespace: str, default_branch: str = "main"
    ) -> lakefs_sdk.Repository:
        return self._client.repositories_api.create_repository(
            repository_creation=lakefs_sdk.RepositoryCreation(
                name=name,
                storage_namespace=storage_namespace,
                default_branch=default_branch,
            )
        )

    def delete_repository(self, name: str) -> None:
        self._client.repositories_api.delete_repository(name)

    def get_repository(self, name: str) -> lakefs_sdk.Repository:
        return self._client.repositories_api.get_repository(name)

    def list_repositories(
        self, prefix: str = "", after: str = "", amount: int = 100
    ) -> lakefs_sdk.RepositoryList:
        return self._client.repositories_api.list_repositories(
            prefix=prefix, after=after, amount=amount
        )

    def set_repository_metadata(self, repo: str, metadata: dict[str, str]) -> None:
        self._client.internal_api.set_repository_metadata(
            repo, lakefs_sdk.RepositoryMetadataSet(metadata=metadata)
        )

    def get_repository_metadata(self, repo: str) -> dict[str, str]:
        resp = self._client.internal_api.get_repository_metadata(repo)
        return resp.metadata or {}

    # ---- Object (File) 操作 ----

    def upload_object(
        self, repo: str, branch: str, path: str, content: bytes | str,
    ) -> lakefs_sdk.ObjectStats:
        return self._client.objects_api.upload_object(
            repository=repo, branch=branch, path=path, content=content,
        )

    def delete_object(self, repo: str, branch: str, path: str) -> None:
        self._client.objects_api.delete_object(
            repository=repo, branch=branch, path=path,
        )

    def get_object(self, repo: str, ref: str, path: str) -> bytes:
        return self._client.objects_api.get_object(
            repository=repo, ref=ref, path=path,
        )

    def stat_object(self, repo: str, ref: str, path: str) -> lakefs_sdk.ObjectStats:
        return self._client.objects_api.stat_object(
            repository=repo, ref=ref, path=path,
        )

    def list_objects(
        self, repo: str, ref: str, prefix: str = "",
        after: str = "", amount: int = 100, delimiter: str = "",
    ) -> lakefs_sdk.ObjectStatsList:
        return self._client.objects_api.list_objects(
            repository=repo, ref=ref, prefix=prefix,
            after=after, amount=amount, delimiter=delimiter,
        )

    # ---- Commit (Version) 操作 ----

    def commit(
        self, repo: str, branch: str, message: str, metadata: dict[str, str] | None = None,
    ) -> lakefs_sdk.Commit:
        return self._client.commits_api.commit(
            repository=repo, branch=branch,
            commit_creation=lakefs_sdk.CommitCreation(
                message=message, metadata=metadata or {},
            ),
        )

    def get_commit(self, repo: str, commit_id: str) -> lakefs_sdk.Commit:
        return self._client.commits_api.get_commit(
            repository=repo, commit_id=commit_id,
        )

    def log_commits(
        self, repo: str, ref: str, after: str = "", amount: int = 100,
    ) -> lakefs_sdk.CommitList:
        return self._client.refs_api.log_commits(
            repository=repo, ref=ref, after=after, amount=amount,
        )

    # ---- Diff & Revert ----

    def diff_refs(
        self, repo: str, left_ref: str, right_ref: str,
        after: str = "", amount: int = 100,
    ) -> lakefs_sdk.DiffList:
        return self._client.refs_api.diff_refs(
            repository=repo, left_ref=left_ref, right_ref=right_ref,
            after=after, amount=amount,
        )

    def revert_commit(
        self, repo: str, branch: str, ref: str, parent_number: int = 0,
    ) -> None:
        self._client.branches_api.revert_branch(
            repository=repo, branch=branch,
            revert_creation=lakefs_sdk.RevertCreation(
                ref=ref, parent_number=parent_number,
            ),
        )

    def hard_reset_branch(self, repo: str, branch: str, ref: str) -> None:
        self._client.branches_api.hard_reset_branch(
            repository=repo, branch=branch, ref=ref,
        )

    def copy_object(
        self, repo: str, branch: str, dest_path: str, src_ref: str, src_path: str,
    ) -> lakefs_sdk.ObjectStats:
        return self._client.objects_api.copy_object(
            repository=repo, branch=branch, dest_path=dest_path,
            object_copy_creation=lakefs_sdk.ObjectCopyCreation(
                src_ref=src_ref, src_path=src_path,
            ),
        )

    def diff_branch(
        self, repo: str, branch: str, after: str = "", amount: int = 100,
    ) -> lakefs_sdk.DiffList:
        return self._client.branches_api.diff_branch(
            repository=repo, branch=branch, after=after, amount=amount,
        )
```

### 6.3 Pydantic 数据模型 (`models/`)

定义 API 请求/响应的数据结构：

```python
# models/dataset.py
from datetime import datetime
from pydantic import BaseModel, Field

class DatasetCreate(BaseModel):
    name: str = Field(..., pattern=r"^[a-z0-9][a-z0-9\-]{2,62}$",
                      description="数据集唯一标识，只允许小写字母、数字和连字符")
    display_name: str = Field(..., max_length=255)
    creator: str
    description: str = ""
    storage_namespace: str | None = None

class DatasetUpdate(BaseModel):
    display_name: str | None = None
    description: str | None = None

class DatasetResponse(BaseModel):
    id: str
    display_name: str
    creator: str
    description: str
    creation_time: datetime
    last_modified_time: datetime | None = None
    size_bytes: int = 0
    file_count: int = 0
    version_count: int = 0
    storage_namespace: str
    default_branch: str = "main"

class DatasetListResponse(BaseModel):
    results: list[DatasetResponse]
    has_more: bool = False
    next_offset: str = ""
```

```python
# models/file.py
from datetime import datetime
from pydantic import BaseModel

class FileInfo(BaseModel):
    path: str
    size_bytes: int
    checksum: str
    content_type: str = ""
    last_modified: datetime | None = None
    metadata: dict[str, str] = {}

class FileUploadResponse(BaseModel):
    path: str
    size_bytes: int
    checksum: str
    content_type: str = ""
    version_id: str

class FileListResponse(BaseModel):
    results: list[FileInfo]
    has_more: bool = False
    next_offset: str = ""
```

```python
# models/version.py
from datetime import datetime
from pydantic import BaseModel

class VersionInfo(BaseModel):
    version_id: str
    message: str
    operator: str = ""
    operation: str = ""
    timestamp: datetime
    parent_versions: list[str] = []

class DiffEntry(BaseModel):
    type: str  # "added" | "removed" | "changed"
    path: str
    size_bytes: int = 0

class VersionDetail(VersionInfo):
    changes: list[DiffEntry] = []

class RollbackRequest(BaseModel):
    strategy: str = "diff_based"  # "diff_based" | "hard_reset"
    operator: str = ""

class RollbackResponse(BaseModel):
    new_version_id: str
    message: str
    rolled_back_to: str
    changes: list[DiffEntry] = []

class VersionListResponse(BaseModel):
    results: list[VersionInfo]
    has_more: bool = False
    next_offset: str = ""
```

```python
# models/lineage.py
from datetime import datetime
from uuid import UUID
from pydantic import BaseModel

class TaskCreate(BaseModel):
    name: str
    description: str = ""
    task_type: str = ""
    parent_task_id: UUID | None = None
    creator: str = ""
    metadata: dict = {}

class TaskResponse(BaseModel):
    id: UUID
    name: str
    description: str
    task_type: str
    parent_task_id: UUID | None
    creator: str
    created_at: datetime
    sub_tasks: list["TaskResponse"] = []

class TaskRunCreate(BaseModel):
    metadata: dict = {}

class TaskRunResponse(BaseModel):
    run_id: UUID
    task_id: UUID
    status: str  # "running" | "completed" | "failed"
    started_at: datetime
    finished_at: datetime | None = None

class LineageEdgeCreate(BaseModel):
    dataset_id: str
    dataset_version: str | None = None
    direction: str  # "input" | "output"

class LineageEdgeBatchCreate(BaseModel):
    edges: list[LineageEdgeCreate]

class DatasetLineageNode(BaseModel):
    dataset_id: str
    dataset_version: str | None = None
    task_run_id: UUID | None = None
    task_name: str = ""
    task_type: str = ""

class DatasetLineageResponse(BaseModel):
    dataset_id: str
    upstream: list[DatasetLineageNode] = []
    downstream: list[DatasetLineageNode] = []

class DAGNode(BaseModel):
    type: str  # "dataset" | "task"
    id: str
    name: str

class DAGEdge(BaseModel):
    source: str
    target: str
    edge_type: str  # "input" | "output"

class DAGResponse(BaseModel):
    nodes: list[DAGNode]
    edges: list[DAGEdge]
```

### 6.4 数据库层 (`db/`)

使用 SQLAlchemy 2.0 + asyncpg 实现 Lineage 数据持久化：

```python
# db/engine.py
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker, AsyncSession
from platform_data_service.config import Settings

def create_db_engine(settings: Settings):
    engine = create_async_engine(settings.database_url, echo=settings.debug, pool_size=10)
    session_factory = async_sessionmaker(engine, class_=AsyncSession, expire_on_commit=False)
    return engine, session_factory
```

```python
# db/tables.py
import uuid
from datetime import datetime, timezone
from sqlalchemy import Column, String, Text, DateTime, ForeignKey, Index, UniqueConstraint
from sqlalchemy.dialects.postgresql import UUID, JSONB
from sqlalchemy.orm import DeclarativeBase, relationship

class Base(DeclarativeBase):
    pass

class LineageTask(Base):
    __tablename__ = "lineage_tasks"

    id = Column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    name = Column(String(255), nullable=False)
    description = Column(Text, default="")
    task_type = Column(String(50), default="")
    parent_task_id = Column(UUID(as_uuid=True), ForeignKey("lineage_tasks.id"), nullable=True)
    creator = Column(String(255), nullable=False, default="")
    created_at = Column(DateTime(timezone=True), default=lambda: datetime.now(timezone.utc))
    updated_at = Column(DateTime(timezone=True), default=lambda: datetime.now(timezone.utc),
                        onupdate=lambda: datetime.now(timezone.utc))
    metadata_ = Column("metadata", JSONB, default=dict)

    sub_tasks = relationship("LineageTask", backref="parent_task", remote_side=[id])
    runs = relationship("LineageTaskRun", back_populates="task")

class LineageTaskRun(Base):
    __tablename__ = "lineage_task_runs"

    id = Column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    task_id = Column(UUID(as_uuid=True), ForeignKey("lineage_tasks.id"), nullable=False)
    status = Column(String(20), nullable=False, default="running")
    started_at = Column(DateTime(timezone=True), default=lambda: datetime.now(timezone.utc))
    finished_at = Column(DateTime(timezone=True), nullable=True)
    error_message = Column(Text, nullable=True)
    metadata_ = Column("metadata", JSONB, default=dict)

    task = relationship("LineageTask", back_populates="runs")
    edges = relationship("LineageEdge", back_populates="task_run")

class LineageEdge(Base):
    __tablename__ = "lineage_edges"
    __table_args__ = (
        UniqueConstraint("task_run_id", "dataset_id", "direction"),
        Index("idx_lineage_edges_dataset", "dataset_id"),
    )

    id = Column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    task_run_id = Column(UUID(as_uuid=True), ForeignKey("lineage_task_runs.id"), nullable=False)
    dataset_id = Column(String(255), nullable=False)
    dataset_version = Column(String(255), nullable=True)
    direction = Column(String(10), nullable=False)  # "input" | "output"
    created_at = Column(DateTime(timezone=True), default=lambda: datetime.now(timezone.utc))

    task_run = relationship("LineageTaskRun", back_populates="edges")
```

```python
# db/repository.py
from uuid import UUID
from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy.orm import selectinload
from .tables import LineageTask, LineageTaskRun, LineageEdge

class LineageRepository:
    """Lineage 数据访问层，所有 SQL 查询集中在此。"""

    def __init__(self, session: AsyncSession):
        self._session = session

    # ---- Task ----

    async def create_task(self, task: LineageTask) -> LineageTask:
        self._session.add(task)
        await self._session.flush()
        return task

    async def get_task(self, task_id: UUID) -> LineageTask | None:
        stmt = (
            select(LineageTask)
            .options(selectinload(LineageTask.sub_tasks))
            .where(LineageTask.id == task_id)
        )
        result = await self._session.execute(stmt)
        return result.scalar_one_or_none()

    async def list_tasks(
        self, parent_id: UUID | None = None, offset: int = 0, limit: int = 100,
    ) -> list[LineageTask]:
        stmt = select(LineageTask).where(LineageTask.parent_task_id == parent_id)
        stmt = stmt.offset(offset).limit(limit).order_by(LineageTask.created_at.desc())
        result = await self._session.execute(stmt)
        return list(result.scalars().all())

    # ---- TaskRun ----

    async def create_task_run(self, run: LineageTaskRun) -> LineageTaskRun:
        self._session.add(run)
        await self._session.flush()
        return run

    async def get_task_run(self, run_id: UUID) -> LineageTaskRun | None:
        stmt = (
            select(LineageTaskRun)
            .options(selectinload(LineageTaskRun.edges), selectinload(LineageTaskRun.task))
            .where(LineageTaskRun.id == run_id)
        )
        result = await self._session.execute(stmt)
        return result.scalar_one_or_none()

    async def update_task_run_status(
        self, run_id: UUID, status: str, error_message: str | None = None,
    ) -> None:
        run = await self.get_task_run(run_id)
        if run:
            run.status = status
            if error_message:
                run.error_message = error_message
            if status in ("completed", "failed"):
                from datetime import datetime, timezone
                run.finished_at = datetime.now(timezone.utc)

    # ---- Edge ----

    async def create_edges(self, edges: list[LineageEdge]) -> list[LineageEdge]:
        self._session.add_all(edges)
        await self._session.flush()
        return edges

    async def get_edges_by_dataset(
        self, dataset_id: str, direction: str | None = None,
    ) -> list[LineageEdge]:
        stmt = (
            select(LineageEdge)
            .options(selectinload(LineageEdge.task_run).selectinload(LineageTaskRun.task))
            .where(LineageEdge.dataset_id == dataset_id)
        )
        if direction:
            stmt = stmt.where(LineageEdge.direction == direction)
        result = await self._session.execute(stmt)
        return list(result.scalars().all())

    async def get_edges_by_task_run(self, run_id: UUID) -> list[LineageEdge]:
        stmt = select(LineageEdge).where(LineageEdge.task_run_id == run_id)
        result = await self._session.execute(stmt)
        return list(result.scalars().all())
```

### 6.5 业务逻辑层 (`services/`)

Service 层是核心业务逻辑所在，封装 lakeFS 操作并实现自动版本管理和 Lineage 追踪。

```python
# services/dataset_service.py
from datetime import datetime
from cachetools import TTLCache
from platform_data_service.lakefs.client import LakeFSClient
from platform_data_service.models.dataset import DatasetCreate, DatasetResponse, DatasetUpdate

_size_cache = TTLCache(maxsize=1024, ttl=300)

class DatasetService:
    METADATA_PREFIX = "platform."

    def __init__(self, lakefs: LakeFSClient):
        self._lakefs = lakefs

    async def create_dataset(self, req: DatasetCreate, storage_ns_prefix: str) -> DatasetResponse:
        storage_ns = req.storage_namespace or f"{storage_ns_prefix}/{req.name}"
        repo = self._lakefs.create_repository(req.name, storage_ns)
        self._lakefs.set_repository_metadata(req.name, {
            f"{self.METADATA_PREFIX}display_name": req.display_name,
            f"{self.METADATA_PREFIX}creator": req.creator,
            f"{self.METADATA_PREFIX}description": req.description,
            f"{self.METADATA_PREFIX}managed": "true",
        })
        return DatasetResponse(
            id=repo.id, display_name=req.display_name, creator=req.creator,
            description=req.description, creation_time=repo.creation_date,
            storage_namespace=storage_ns, default_branch=repo.default_branch,
        )

    async def get_dataset(self, dataset_id: str) -> DatasetResponse:
        repo = self._lakefs.get_repository(dataset_id)
        metadata = self._lakefs.get_repository_metadata(dataset_id)

        last_modified = repo.creation_date
        try:
            commits = self._lakefs.log_commits(dataset_id, repo.default_branch, amount=1)
            if commits.results:
                last_modified = commits.results[0].creation_date
        except Exception:
            pass

        size_bytes, file_count = await self._get_dataset_size(dataset_id, repo.default_branch)

        return DatasetResponse(
            id=repo.id,
            display_name=metadata.get(f"{self.METADATA_PREFIX}display_name", dataset_id),
            creator=metadata.get(f"{self.METADATA_PREFIX}creator", ""),
            description=metadata.get(f"{self.METADATA_PREFIX}description", ""),
            creation_time=repo.creation_date,
            last_modified_time=last_modified,
            size_bytes=size_bytes,
            file_count=file_count,
            storage_namespace=repo.storage_namespace,
            default_branch=repo.default_branch,
        )

    async def update_dataset(self, dataset_id: str, req: DatasetUpdate) -> DatasetResponse:
        updates = {}
        if req.display_name is not None:
            updates[f"{self.METADATA_PREFIX}display_name"] = req.display_name
        if req.description is not None:
            updates[f"{self.METADATA_PREFIX}description"] = req.description
        if updates:
            self._lakefs.set_repository_metadata(dataset_id, updates)
        return await self.get_dataset(dataset_id)

    async def delete_dataset(self, dataset_id: str) -> None:
        self._lakefs.delete_repository(dataset_id)
        _size_cache.pop(dataset_id, None)

    async def list_datasets(
        self, prefix: str = "", after: str = "", amount: int = 100,
    ) -> tuple[list[DatasetResponse], bool, str]:
        repo_list = self._lakefs.list_repositories(prefix=prefix, after=after, amount=amount)
        results = []
        for repo in repo_list.results:
            try:
                meta = self._lakefs.get_repository_metadata(repo.id)
                if meta.get(f"{self.METADATA_PREFIX}managed") != "true":
                    continue
            except Exception:
                continue
            results.append(DatasetResponse(
                id=repo.id,
                display_name=meta.get(f"{self.METADATA_PREFIX}display_name", repo.id),
                creator=meta.get(f"{self.METADATA_PREFIX}creator", ""),
                description=meta.get(f"{self.METADATA_PREFIX}description", ""),
                creation_time=repo.creation_date,
                storage_namespace=repo.storage_namespace,
                default_branch=repo.default_branch,
            ))
        return results, repo_list.pagination.has_more, repo_list.pagination.next_offset

    async def _get_dataset_size(self, dataset_id: str, branch: str) -> tuple[int, int]:
        cached = _size_cache.get(dataset_id)
        if cached:
            return cached
        total_size = 0
        file_count = 0
        after = ""
        while True:
            objs = self._lakefs.list_objects(dataset_id, branch, after=after, amount=1000)
            for obj in objs.results:
                total_size += obj.size_bytes
                file_count += 1
            if not objs.pagination.has_more:
                break
            after = objs.pagination.next_offset
        _size_cache[dataset_id] = (total_size, file_count)
        return total_size, file_count
```

```python
# services/file_service.py
from platform_data_service.lakefs.client import LakeFSClient
from platform_data_service.models.file import FileInfo, FileUploadResponse, FileListResponse

class FileService:
    def __init__(self, lakefs: LakeFSClient, default_branch: str = "main"):
        self._lakefs = lakefs
        self._branch = default_branch

    async def upload_file(
        self, dataset_id: str, path: str, content: bytes,
        operator: str = "", task_id: str | None = None,
    ) -> FileUploadResponse:
        stats = self._lakefs.upload_object(dataset_id, self._branch, path, content)
        metadata = {
            "platform.operation": "add_file",
            "platform.file_path": path,
            "platform.operator": operator,
        }
        if task_id:
            metadata["platform.task_id"] = task_id
        commit = self._lakefs.commit(
            dataset_id, self._branch,
            message=f"[add_file] {path}", metadata=metadata,
        )
        return FileUploadResponse(
            path=path, size_bytes=stats.size_bytes,
            checksum=stats.checksum, content_type=stats.content_type or "",
            version_id=commit.id,
        )

    async def upload_files(
        self, dataset_id: str, files: list[tuple[str, bytes]],
        operator: str = "", task_id: str | None = None,
    ) -> list[FileUploadResponse]:
        results = []
        for path, content in files:
            stats = self._lakefs.upload_object(dataset_id, self._branch, path, content)
            results.append(FileUploadResponse(
                path=path, size_bytes=stats.size_bytes,
                checksum=stats.checksum, content_type=stats.content_type or "",
                version_id="",
            ))
        paths = [f[0] for f in files]
        metadata = {
            "platform.operation": "batch_add",
            "platform.file_paths": ",".join(paths),
            "platform.operator": operator,
        }
        if task_id:
            metadata["platform.task_id"] = task_id
        commit = self._lakefs.commit(
            dataset_id, self._branch,
            message=f"[batch_add] {len(files)} files", metadata=metadata,
        )
        for r in results:
            r.version_id = commit.id
        return results

    async def delete_file(
        self, dataset_id: str, path: str, operator: str = "",
    ) -> str:
        self._lakefs.delete_object(dataset_id, self._branch, path)
        commit = self._lakefs.commit(
            dataset_id, self._branch,
            message=f"[delete_file] {path}",
            metadata={
                "platform.operation": "delete_file",
                "platform.file_path": path,
                "platform.operator": operator,
            },
        )
        return commit.id

    async def get_file_content(
        self, dataset_id: str, path: str, version: str | None = None,
    ) -> bytes:
        ref = version or self._branch
        return self._lakefs.get_object(dataset_id, ref, path)

    async def list_files(
        self, dataset_id: str, prefix: str = "",
        after: str = "", amount: int = 100, version: str | None = None,
    ) -> FileListResponse:
        ref = version or self._branch
        objs = self._lakefs.list_objects(
            dataset_id, ref, prefix=prefix, after=after, amount=amount,
        )
        results = [
            FileInfo(
                path=obj.path, size_bytes=obj.size_bytes,
                checksum=obj.checksum, content_type=obj.content_type or "",
                last_modified=obj.mtime, metadata=obj.metadata or {},
            )
            for obj in objs.results
            if obj.path_type == "object"
        ]
        return FileListResponse(
            results=results,
            has_more=objs.pagination.has_more,
            next_offset=objs.pagination.next_offset,
        )
```

```python
# services/version_service.py
from platform_data_service.lakefs.client import LakeFSClient
from platform_data_service.models.version import (
    VersionInfo, VersionDetail, DiffEntry, RollbackResponse, VersionListResponse,
)

class VersionService:
    def __init__(self, lakefs: LakeFSClient, default_branch: str = "main"):
        self._lakefs = lakefs
        self._branch = default_branch

    async def list_versions(
        self, dataset_id: str, after: str = "", amount: int = 100,
    ) -> VersionListResponse:
        commits = self._lakefs.log_commits(dataset_id, self._branch, after=after, amount=amount)
        results = [self._commit_to_version_info(c) for c in commits.results]
        return VersionListResponse(
            results=results,
            has_more=commits.pagination.has_more,
            next_offset=commits.pagination.next_offset,
        )

    async def get_version_detail(self, dataset_id: str, version_id: str) -> VersionDetail:
        commit = self._lakefs.get_commit(dataset_id, version_id)
        info = self._commit_to_version_info(commit)
        changes = []
        if commit.parents:
            parent = commit.parents[0]
            diffs = self._lakefs.diff_refs(dataset_id, parent, version_id, amount=1000)
            changes = [
                DiffEntry(
                    type=d.type, path=d.path,
                    size_bytes=d.size_bytes if hasattr(d, "size_bytes") else 0,
                )
                for d in diffs.results
            ]
        return VersionDetail(**info.model_dump(), changes=changes)

    async def rollback(
        self, dataset_id: str, target_version: str, operator: str = "",
    ) -> RollbackResponse:
        """基于 Diff 的精确回滚：对比当前 HEAD 与目标版本，执行差异操作后 commit。"""
        after = ""
        all_diffs = []
        while True:
            diffs = self._lakefs.diff_refs(
                dataset_id, self._branch, target_version, after=after, amount=1000,
            )
            all_diffs.extend(diffs.results)
            if not diffs.pagination.has_more:
                break
            after = diffs.pagination.next_offset

        for diff in all_diffs:
            if diff.type == "removed":
                self._lakefs.copy_object(
                    dataset_id, self._branch, diff.path, target_version, diff.path,
                )
            elif diff.type == "changed":
                self._lakefs.copy_object(
                    dataset_id, self._branch, diff.path, target_version, diff.path,
                )
            elif diff.type == "added":
                self._lakefs.delete_object(dataset_id, self._branch, diff.path)

        commit = self._lakefs.commit(
            dataset_id, self._branch,
            message=f"[rollback] Rollback to version {target_version}",
            metadata={
                "platform.operation": "rollback",
                "platform.target_version": target_version,
                "platform.operator": operator,
            },
        )
        changes = [DiffEntry(type=d.type, path=d.path) for d in all_diffs]
        return RollbackResponse(
            new_version_id=commit.id, message=commit.message,
            rolled_back_to=target_version, changes=changes,
        )

    @staticmethod
    def _commit_to_version_info(commit) -> VersionInfo:
        meta = commit.metadata or {}
        return VersionInfo(
            version_id=commit.id, message=commit.message,
            operator=meta.get("platform.operator", commit.committer or ""),
            operation=meta.get("platform.operation", ""),
            timestamp=commit.creation_date,
            parent_versions=list(commit.parents) if commit.parents else [],
        )
```

```python
# services/lineage_service.py
from uuid import UUID
from collections import deque
from sqlalchemy.ext.asyncio import AsyncSession
from platform_data_service.db.repository import LineageRepository
from platform_data_service.db.tables import LineageTask, LineageTaskRun, LineageEdge
from platform_data_service.models.lineage import (
    TaskCreate, TaskResponse, TaskRunCreate, TaskRunResponse,
    LineageEdgeCreate, DatasetLineageResponse, DatasetLineageNode,
    DAGNode, DAGEdge, DAGResponse,
)

class LineageService:
    def __init__(self, session: AsyncSession):
        self._repo = LineageRepository(session)
        self._session = session

    async def create_task(self, req: TaskCreate) -> TaskResponse:
        task = LineageTask(
            name=req.name, description=req.description, task_type=req.task_type,
            parent_task_id=req.parent_task_id, creator=req.creator, metadata_=req.metadata,
        )
        task = await self._repo.create_task(task)
        await self._session.commit()
        return self._task_to_response(task)

    async def get_task(self, task_id: UUID) -> TaskResponse | None:
        task = await self._repo.get_task(task_id)
        return self._task_to_response(task) if task else None

    async def start_run(self, task_id: UUID, req: TaskRunCreate) -> TaskRunResponse:
        run = LineageTaskRun(task_id=task_id, metadata_=req.metadata)
        run = await self._repo.create_task_run(run)
        await self._session.commit()
        return TaskRunResponse(
            run_id=run.id, task_id=run.task_id,
            status=run.status, started_at=run.started_at,
        )

    async def finish_run(
        self, run_id: UUID, status: str, error_message: str | None = None,
    ) -> None:
        await self._repo.update_task_run_status(run_id, status, error_message)
        await self._session.commit()

    async def add_edges(self, run_id: UUID, edges: list[LineageEdgeCreate]) -> None:
        db_edges = [
            LineageEdge(
                task_run_id=run_id, dataset_id=e.dataset_id,
                dataset_version=e.dataset_version, direction=e.direction,
            )
            for e in edges
        ]
        await self._repo.create_edges(db_edges)
        await self._session.commit()

    async def get_dataset_lineage(
        self, dataset_id: str, depth: int = 3,
    ) -> DatasetLineageResponse:
        upstream = await self._traverse(dataset_id, "output", depth)
        downstream = await self._traverse(dataset_id, "input", depth)
        return DatasetLineageResponse(
            dataset_id=dataset_id, upstream=upstream, downstream=downstream,
        )

    async def get_dag(
        self, root_dataset_id: str, depth: int = 5,
    ) -> DAGResponse:
        """BFS 遍历构建 DAG 图。"""
        nodes: dict[str, DAGNode] = {}
        edges: list[DAGEdge] = []
        visited: set[str] = set()
        queue: deque[tuple[str, int]] = deque([(root_dataset_id, 0)])

        while queue:
            dataset_id, current_depth = queue.popleft()
            if dataset_id in visited or current_depth > depth:
                continue
            visited.add(dataset_id)
            nodes[f"ds:{dataset_id}"] = DAGNode(type="dataset", id=dataset_id, name=dataset_id)

            for direction in ("input", "output"):
                db_edges = await self._repo.get_edges_by_dataset(dataset_id, direction)
                for edge in db_edges:
                    task = edge.task_run.task
                    task_key = f"task:{task.id}"
                    nodes[task_key] = DAGNode(type="task", id=str(task.id), name=task.name)

                    if direction == "output":
                        edges.append(DAGEdge(
                            source=task_key, target=f"ds:{dataset_id}", edge_type="output",
                        ))
                        input_edges = await self._repo.get_edges_by_task_run(edge.task_run_id)
                        for ie in input_edges:
                            if ie.direction == "input" and ie.dataset_id not in visited:
                                queue.append((ie.dataset_id, current_depth + 1))
                                edges.append(DAGEdge(
                                    source=f"ds:{ie.dataset_id}", target=task_key,
                                    edge_type="input",
                                ))
                    elif direction == "input":
                        edges.append(DAGEdge(
                            source=f"ds:{dataset_id}", target=task_key, edge_type="input",
                        ))
                        output_edges = await self._repo.get_edges_by_task_run(edge.task_run_id)
                        for oe in output_edges:
                            if oe.direction == "output" and oe.dataset_id not in visited:
                                queue.append((oe.dataset_id, current_depth + 1))
                                edges.append(DAGEdge(
                                    source=task_key, target=f"ds:{oe.dataset_id}",
                                    edge_type="output",
                                ))

        return DAGResponse(nodes=list(nodes.values()), edges=edges)

    async def _traverse(
        self, dataset_id: str, direction: str, depth: int,
    ) -> list[DatasetLineageNode]:
        """沿指定方向遍历血缘关系。direction='output' 表示查上游，'input' 表示查下游。"""
        results = []
        db_edges = await self._repo.get_edges_by_dataset(dataset_id, direction)
        for edge in db_edges:
            task = edge.task_run.task
            opposite = "input" if direction == "output" else "output"
            related_edges = await self._repo.get_edges_by_task_run(edge.task_run_id)
            for re in related_edges:
                if re.direction == opposite:
                    results.append(DatasetLineageNode(
                        dataset_id=re.dataset_id, dataset_version=re.dataset_version,
                        task_run_id=edge.task_run_id, task_name=task.name,
                        task_type=task.task_type,
                    ))
        return results

    @staticmethod
    def _task_to_response(task: LineageTask) -> TaskResponse:
        sub_tasks = [
            TaskResponse(
                id=st.id, name=st.name, description=st.description or "",
                task_type=st.task_type or "", parent_task_id=st.parent_task_id,
                creator=st.creator or "", created_at=st.created_at,
            )
            for st in (task.sub_tasks or [])
        ]
        return TaskResponse(
            id=task.id, name=task.name, description=task.description or "",
            task_type=task.task_type or "", parent_task_id=task.parent_task_id,
            creator=task.creator or "", created_at=task.created_at, sub_tasks=sub_tasks,
        )
```

### 6.6 FastAPI 路由层 (`routers/`)

```python
# routers/datasets.py
from fastapi import APIRouter, Depends, HTTPException
from platform_data_service.dependencies import get_dataset_service
from platform_data_service.models.dataset import (
    DatasetCreate, DatasetResponse, DatasetUpdate, DatasetListResponse,
)
from platform_data_service.services.dataset_service import DatasetService

router = APIRouter(prefix="/api/v1/datasets", tags=["datasets"])

@router.post("", response_model=DatasetResponse, status_code=201)
async def create_dataset(
    req: DatasetCreate, svc: DatasetService = Depends(get_dataset_service),
):
    return await svc.create_dataset(req, storage_ns_prefix="s3://platform-data")

@router.get("", response_model=DatasetListResponse)
async def list_datasets(
    prefix: str = "", after: str = "", amount: int = 100,
    svc: DatasetService = Depends(get_dataset_service),
):
    results, has_more, next_offset = await svc.list_datasets(prefix, after, amount)
    return DatasetListResponse(results=results, has_more=has_more, next_offset=next_offset)

@router.get("/{dataset_id}", response_model=DatasetResponse)
async def get_dataset(
    dataset_id: str, svc: DatasetService = Depends(get_dataset_service),
):
    return await svc.get_dataset(dataset_id)

@router.patch("/{dataset_id}", response_model=DatasetResponse)
async def update_dataset(
    dataset_id: str, req: DatasetUpdate,
    svc: DatasetService = Depends(get_dataset_service),
):
    return await svc.update_dataset(dataset_id, req)

@router.delete("/{dataset_id}", status_code=204)
async def delete_dataset(
    dataset_id: str, svc: DatasetService = Depends(get_dataset_service),
):
    await svc.delete_dataset(dataset_id)
```

```python
# routers/files.py
from fastapi import APIRouter, Depends, UploadFile, File, Query
from fastapi.responses import StreamingResponse
from platform_data_service.dependencies import get_file_service
from platform_data_service.models.file import FileUploadResponse, FileListResponse
from platform_data_service.services.file_service import FileService
import io

router = APIRouter(prefix="/api/v1/datasets/{dataset_id}/files", tags=["files"])

@router.post("", response_model=FileUploadResponse, status_code=201)
async def upload_file(
    dataset_id: str, path: str = Query(...), file: UploadFile = File(...),
    operator: str = Query(""), svc: FileService = Depends(get_file_service),
):
    content = await file.read()
    return await svc.upload_file(dataset_id, path, content, operator)

@router.get("", response_model=FileListResponse)
async def list_files(
    dataset_id: str, prefix: str = "", after: str = "",
    amount: int = 100, version: str | None = None,
    svc: FileService = Depends(get_file_service),
):
    return await svc.list_files(dataset_id, prefix, after, amount, version)

@router.get("/content")
async def download_file(
    dataset_id: str, path: str = Query(...), version: str | None = None,
    svc: FileService = Depends(get_file_service),
):
    content = await svc.get_file_content(dataset_id, path, version)
    return StreamingResponse(io.BytesIO(content), media_type="application/octet-stream")

@router.delete("")
async def delete_file(
    dataset_id: str, path: str = Query(...), operator: str = Query(""),
    svc: FileService = Depends(get_file_service),
):
    version_id = await svc.delete_file(dataset_id, path, operator)
    return {"version_id": version_id}
```

```python
# routers/versions.py
from fastapi import APIRouter, Depends
from platform_data_service.dependencies import get_version_service
from platform_data_service.models.version import (
    VersionDetail, RollbackRequest, RollbackResponse, VersionListResponse,
)
from platform_data_service.services.version_service import VersionService

router = APIRouter(prefix="/api/v1/datasets/{dataset_id}/versions", tags=["versions"])

@router.get("", response_model=VersionListResponse)
async def list_versions(
    dataset_id: str, after: str = "", amount: int = 100,
    svc: VersionService = Depends(get_version_service),
):
    return await svc.list_versions(dataset_id, after, amount)

@router.get("/{version_id}", response_model=VersionDetail)
async def get_version(
    dataset_id: str, version_id: str,
    svc: VersionService = Depends(get_version_service),
):
    return await svc.get_version_detail(dataset_id, version_id)

@router.post("/{version_id}/rollback", response_model=RollbackResponse)
async def rollback(
    dataset_id: str, version_id: str, req: RollbackRequest,
    svc: VersionService = Depends(get_version_service),
):
    return await svc.rollback(dataset_id, version_id, req.operator)
```

```python
# routers/lineage.py
from uuid import UUID
from fastapi import APIRouter, Depends
from platform_data_service.dependencies import get_lineage_service
from platform_data_service.models.lineage import (
    TaskCreate, TaskResponse, TaskRunCreate, TaskRunResponse,
    LineageEdgeBatchCreate, DatasetLineageResponse, DAGResponse,
)
from platform_data_service.services.lineage_service import LineageService

router = APIRouter(prefix="/api/v1/lineage", tags=["lineage"])

@router.post("/tasks", response_model=TaskResponse, status_code=201)
async def create_task(
    req: TaskCreate, svc: LineageService = Depends(get_lineage_service),
):
    return await svc.create_task(req)

@router.get("/tasks/{task_id}", response_model=TaskResponse)
async def get_task(
    task_id: UUID, svc: LineageService = Depends(get_lineage_service),
):
    result = await svc.get_task(task_id)
    if not result:
        from fastapi import HTTPException
        raise HTTPException(status_code=404, detail="Task not found")
    return result

@router.post("/tasks/{task_id}/runs", response_model=TaskRunResponse, status_code=201)
async def start_run(
    task_id: UUID, req: TaskRunCreate,
    svc: LineageService = Depends(get_lineage_service),
):
    return await svc.start_run(task_id, req)

@router.patch("/runs/{run_id}")
async def update_run(
    run_id: UUID, status: str, error_message: str | None = None,
    svc: LineageService = Depends(get_lineage_service),
):
    await svc.finish_run(run_id, status, error_message)
    return {"status": "ok"}

@router.post("/runs/{run_id}/edges", status_code=201)
async def add_edges(
    run_id: UUID, req: LineageEdgeBatchCreate,
    svc: LineageService = Depends(get_lineage_service),
):
    await svc.add_edges(run_id, req.edges)
    return {"status": "ok"}

@router.get("/datasets/{dataset_id}", response_model=DatasetLineageResponse)
async def get_dataset_lineage(
    dataset_id: str, depth: int = 3,
    svc: LineageService = Depends(get_lineage_service),
):
    return await svc.get_dataset_lineage(dataset_id, depth)

@router.get("/dag", response_model=DAGResponse)
async def get_dag(
    root_dataset_id: str, depth: int = 5,
    svc: LineageService = Depends(get_lineage_service),
):
    return await svc.get_dag(root_dataset_id, depth)
```

### 6.7 依赖注入 (`dependencies.py`)

```python
# dependencies.py
from fastapi import Depends
from sqlalchemy.ext.asyncio import AsyncSession
from platform_data_service.config import Settings
from platform_data_service.db.engine import create_db_engine
from platform_data_service.lakefs.client import LakeFSClient
from platform_data_service.services.dataset_service import DatasetService
from platform_data_service.services.file_service import FileService
from platform_data_service.services.version_service import VersionService
from platform_data_service.services.lineage_service import LineageService

_settings: Settings | None = None
_lakefs_client: LakeFSClient | None = None
_session_factory = None

def get_settings() -> Settings:
    global _settings
    if _settings is None:
        _settings = Settings()
    return _settings

def get_lakefs_client(settings: Settings = Depends(get_settings)) -> LakeFSClient:
    global _lakefs_client
    if _lakefs_client is None:
        _lakefs_client = LakeFSClient(
            endpoint=settings.lakefs_endpoint,
            access_key=settings.lakefs_access_key,
            secret_key=settings.lakefs_secret_key,
        )
    return _lakefs_client

async def get_db_session(settings: Settings = Depends(get_settings)) -> AsyncSession:
    global _session_factory
    if _session_factory is None:
        _, _session_factory = create_db_engine(settings)
    async with _session_factory() as session:
        yield session

def get_dataset_service(
    lakefs: LakeFSClient = Depends(get_lakefs_client),
) -> DatasetService:
    return DatasetService(lakefs)

def get_file_service(
    lakefs: LakeFSClient = Depends(get_lakefs_client),
    settings: Settings = Depends(get_settings),
) -> FileService:
    return FileService(lakefs, settings.lakefs_default_branch)

def get_version_service(
    lakefs: LakeFSClient = Depends(get_lakefs_client),
    settings: Settings = Depends(get_settings),
) -> VersionService:
    return VersionService(lakefs, settings.lakefs_default_branch)

async def get_lineage_service(
    session: AsyncSession = Depends(get_db_session),
) -> LineageService:
    return LineageService(session)
```

### 6.8 应用入口 (`main.py`)

```python
# main.py
from contextlib import asynccontextmanager
from fastapi import FastAPI
from platform_data_service.config import Settings
from platform_data_service.db.engine import create_db_engine
from platform_data_service.routers import datasets, files, versions, lineage

@asynccontextmanager
async def lifespan(app: FastAPI):
    settings = Settings()
    engine, _ = create_db_engine(settings)
    yield
    await engine.dispose()

def create_app() -> FastAPI:
    app = FastAPI(
        title="Platform Data Service",
        description="数据集管理与版本控制平台，基于 lakeFS 构建",
        version="1.0.0",
        lifespan=lifespan,
    )
    app.include_router(datasets.router)
    app.include_router(files.router)
    app.include_router(versions.router)
    app.include_router(lineage.router)
    return app

app = create_app()
```

### 6.9 Docker Compose 开发环境

```yaml
# docker-compose.yml
services:
  postgres:
    image: postgres:16
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data
      - ./init-db.sql:/docker-entrypoint-initdb.d/init-db.sql

  minio:
    image: minio/minio
    command: server /data --console-address ":9001"
    environment:
      MINIO_ROOT_USER: minioadmin
      MINIO_ROOT_PASSWORD: minioadmin
    ports:
      - "9000:9000"
      - "9001:9001"
    volumes:
      - miniodata:/data

  lakefs:
    image: treeverse/lakefs:latest
    depends_on:
      - postgres
      - minio
    ports:
      - "8000:8000"
    environment:
      LAKEFS_DATABASE_TYPE: postgres
      LAKEFS_DATABASE_POSTGRES_CONNECTION_STRING: postgres://postgres:postgres@postgres:5432/lakefs?sslmode=disable
      LAKEFS_AUTH_ENCRYPT_SECRET_KEY: "some-secret-key-at-least-16-ch"
      LAKEFS_BLOCKSTORE_TYPE: s3
      LAKEFS_BLOCKSTORE_S3_ENDPOINT: http://minio:9000
      LAKEFS_BLOCKSTORE_S3_FORCE_PATH_STYLE: "true"
      LAKEFS_BLOCKSTORE_S3_CREDENTIALS_ACCESS_KEY_ID: minioadmin
      LAKEFS_BLOCKSTORE_S3_CREDENTIALS_SECRET_ACCESS_KEY: minioadmin

  platform-service:
    build: .
    depends_on:
      - lakefs
      - postgres
    ports:
      - "8080:8080"
    environment:
      PLATFORM_LAKEFS_ENDPOINT: http://lakefs:8000/api/v1
      PLATFORM_LAKEFS_ACCESS_KEY: ${LAKEFS_ACCESS_KEY}
      PLATFORM_LAKEFS_SECRET_KEY: ${LAKEFS_SECRET_KEY}
      PLATFORM_DATABASE_URL: postgresql+asyncpg://postgres:postgres@postgres:5432/platform_lineage
      PLATFORM_LAKEFS_STORAGE_NAMESPACE_PREFIX: s3://platform-data

volumes:
  pgdata:
  miniodata:
```

### 6.10 pyproject.toml

```toml
# pyproject.toml
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[project]
name = "platform-data-service"
version = "1.0.0"
description = "数据集管理与版本控制平台服务"
requires-python = ">=3.11"
dependencies = [
    "fastapi>=0.110",
    "uvicorn[standard]>=0.27",
    "lakefs-sdk>=1.0",
    "sqlalchemy[asyncio]>=2.0",
    "asyncpg>=0.29",
    "alembic>=1.13",
    "pydantic-settings>=2.0",
    "httpx>=0.27",
    "cachetools>=5.3",
    "apscheduler>=3.10",
]

[project.optional-dependencies]
dev = [
    "pytest>=8.0",
    "pytest-asyncio>=0.23",
    "pytest-cov>=4.0",
    "ruff>=0.3",
    "mypy>=1.8",
    "httpx",  # for TestClient
]

[tool.ruff]
target-version = "py311"
line-length = 100

[tool.mypy]
python_version = "3.11"
strict = true
```

## 7. Python SDK 设计

独立发布的 Python SDK 包，供用户在脚本和 Notebook 中使用：

```python
from platform_data_sdk import PlatformClient

client = PlatformClient(endpoint="http://platform-service:8080")

# ---- 数据集操作 ----
dataset = client.create_dataset(
    name="training-images",
    display_name="训练图像集",
    description="用于目标检测模型的训练图像",
)

datasets = client.list_datasets(prefix="training")

info = client.get_dataset("training-images")
print(f"大小: {info.size_bytes}, 文件数: {info.file_count}")

client.update_dataset("training-images", display_name="新名称")

client.delete_dataset("training-images")


# ---- 文件操作 ----
client.upload_file("training-images", "data/img_001.jpg", local_path="/tmp/img_001.jpg")

client.upload_files("training-images", [
    ("data/img_001.jpg", "/tmp/img_001.jpg"),
    ("data/img_002.jpg", "/tmp/img_002.jpg"),
])

files = client.list_files("training-images", prefix="data/")
for f in files:
    print(f"{f.path}  {f.size_bytes}  {f.last_modified}")

client.download_file("training-images", "data/img_001.jpg", local_path="/tmp/download.jpg")

client.delete_file("training-images", "data/img_001.jpg")


# ---- 版本管理 ----
versions = client.list_versions("training-images")
for v in versions:
    print(f"{v.version_id}  {v.message}  {v.timestamp}")

version_detail = client.get_version("training-images", "commit-sha-xxx")
print(version_detail.changes)

client.rollback("training-images", target_version="commit-sha-xxx")


# ---- Data Lineage ----
task = client.create_task(
    name="preprocess-pipeline",
    task_type="preprocessing",
    description="图像预处理流水线",
)

sub_task = client.create_task(
    name="resize-step",
    task_type="preprocessing",
    parent_task_id=task.id,
)

run = client.start_task_run(task.id)

run.add_input("raw-images", version="commit-sha-aaa")

# ... 执行数据处理 ...
client.upload_file("processed-images", "data/result.csv", local_path="/tmp/result.csv")

run.add_output("processed-images", version="latest")

run.complete()

# 查看血缘
lineage = client.get_dataset_lineage("processed-images", depth=3)
print(lineage.upstream)
print(lineage.downstream)

# 查看 DAG
dag = client.get_lineage_dag(root_dataset="final-model", depth=5)
```

## 8. 一期交付范围与排期

### 8.1 里程碑规划

| 阶段 | 时间 | 交付内容 |
|------|------|---------|
| M1: 核心功能 | 3月4日 - 3月14日 | 数据集 CRUD (R1)、文件操作 (R2/R3)、自动版本管理 (R4) |
| M2: 版本管理 | 3月15日 - 3月21日 | 版本列表 (R5)、版本回滚 (R6)、Python SDK 基础版 |
| M3: Data Lineage | 3月22日 - 3月28日 | Lineage 数据模型、任务管理 API、血缘查询 API (R7) |
| M4: 集成测试 | 3月29日 - 3月31日 | 端到端测试、文档完善、Bug 修复 |

### 8.2 一期详细任务分解

**M1: 项目搭建与核心功能（2周）**
- [ ] 初始化 Python 项目骨架（pyproject.toml、src layout、docker-compose）
- [ ] 实现 `config.py`（pydantic-settings 配置管理）
- [ ] 实现 `lakefs/client.py`（lakefs-sdk 封装层）
- [ ] 实现 `db/` 层（SQLAlchemy tables、engine、Alembic migration）
- [ ] 实现 `services/dataset_service.py` + `routers/datasets.py`（数据集 CRUD）
- [ ] 实现 `services/file_service.py` + `routers/files.py`（文件上传/下载/删除/更新/列表）
- [ ] 实现自动 commit 机制（文件操作后自动生成版本）
- [ ] 实现 `dependencies.py`（FastAPI 依赖注入）
- [ ] 单元测试 (pytest + mock lakeFS)

**M2: 版本管理与 SDK（1周）**
- [ ] 实现 `services/version_service.py` + `routers/versions.py`（版本列表、详情、回滚）
- [ ] 实现 Diff-based 回滚策略
- [ ] 开发 `sdk/` Python SDK 包（PlatformClient，覆盖数据集/文件/版本操作）
- [ ] 集成测试（FastAPI TestClient + 真实 lakeFS）

**M3: Data Lineage（1周）**
- [ ] Alembic migration: 创建 lineage_tasks / lineage_task_runs / lineage_edges 表
- [ ] 实现 `db/repository.py`（Lineage 数据访问层）
- [ ] 实现 `services/lineage_service.py`（任务管理、血缘记录、DAG BFS 查询）
- [ ] 实现 `routers/lineage.py`（Lineage API 路由）
- [ ] Python SDK 增加 Lineage 部分
- [ ] 血缘与 lakeFS commit metadata 联动

**M4: 集成测试与收尾（3天）**
- [ ] 端到端测试场景覆盖（docker-compose up 一键测试）
- [ ] FastAPI 自动生成 OpenAPI 文档校验
- [ ] Dockerfile 优化 & 部署文档
- [ ] Bug 修复与性能优化

## 9. 风险与应对

| 风险 | 影响 | 应对策略 |
|------|------|---------|
| lakeFS API 调用延迟 | 用户体验下降 | 批量操作合并 commit；引入缓存层 |
| 数据集大小计算开销 | 大数据集属性查询慢 | 采用写时增量更新 + 异步校准策略 |
| hard_reset API 稳定性 | 回滚功能受限 | 一期优先使用 diff-based 回滚方案 |
| Lineage 数据一致性 | 血缘记录不完整 | 利用 lakeFS Hook 自动记录；SDK 封装确保操作原子性 |
| 开发周期紧张 | 功能不全 | Lineage 可适当简化，先实现核心 CRUD 和查询，DAG 可视化后续迭代 |

## 10. 后续演进（二期规划）

- **数据集权限管理**：基于 lakeFS RBAC，实现数据集级别的读写权限控制
- **数据集分支**：利用 lakeFS branch 支持数据集的多版本并行开发和合并
- **数据集标签 (Tag)**：利用 lakeFS tag 支持为重要版本打标签（如 v1.0、production）
- **数据质量检查**：利用 lakeFS pre-commit hook 实现文件格式校验、Schema 检查
- **全文搜索**：结合 lakeFS Metadata Search 实现文件内容和属性的搜索
- **数据集克隆**：利用 lakeFS 的零拷贝分支特性实现数据集的快速克隆
- **Lineage 可视化 UI**：DAG 图形化展示
- **与调度系统集成**：Airflow / Argo Workflow 自动记录 lineage
