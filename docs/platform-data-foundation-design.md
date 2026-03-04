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
│                    Platform Service Layer                        │
│          (业务逻辑封装、自动版本管理、权限控制)                     │
├─────────────────────────────────────┬──────────────────────────-┤
│         lakeFS SDK / API            │     Lineage Store         │
│   (Repository, Object, Commit,     │   (PostgreSQL / KV)       │
│    Branch, Tag, Diff, Merge)        │                           │
├─────────────────────────────────────┤                           │
│            lakeFS Server            │                           │
│  ┌──────────┬──────────┬─────────┐  │                           │
│  │ Catalog  │ Graveler │  Auth   │  │                           │
│  │ (对象管理)│(版本引擎) │(认证授权)│  │                           │
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

| 组件 | 技术选型 | 说明 |
|------|---------|------|
| Platform API | Go (chi router) | 与 lakeFS 技术栈一致，可直接复用代码 |
| lakeFS 集成 | lakeFS Go SDK / REST API | 直接调用 lakeFS 的 Go catalog 包或 REST API |
| Lineage 存储 | PostgreSQL | 复用 lakeFS 的 PostgreSQL 实例 |
| 缓存 | 本地缓存 (go-cache / ristretto) | 缓存数据集属性、大小等频繁查询的数据 |
| API 文档 | OpenAPI 3.0 | 与 lakeFS 保持一致 |

### 5.2 部署方式

#### 方式一：独立服务（推荐一期）

```
┌─────────────┐    REST API    ┌──────────────┐    REST API    ┌──────────┐
│   Client    │───────────────>│  Platform    │───────────────>│  lakeFS  │
│  (SDK/CLI)  │                │  Service     │                │  Server  │
└─────────────┘                └──────┬───────┘                └────┬─────┘
                                      │                             │
                                      │ SQL                         │ KV + Block
                                      ▼                             ▼
                               ┌──────────────┐             ┌──────────────┐
                               │ PostgreSQL   │             │ PostgreSQL   │
                               │ (lineage)    │             │ (lakeFS KV)  │
                               └──────────────┘             │ S3/MinIO     │
                                                            └──────────────┘
```

优点：与 lakeFS 解耦，独立开发部署，不影响 lakeFS 稳定性。

#### 方式二：lakeFS 插件（后续考虑）

将 Platform Service 的逻辑作为 lakeFS 的内置模块，直接调用 Go 层面的 catalog 包，减少网络开销。适合长期演进。

### 5.3 项目结构

```
platform-data-service/
├── cmd/
│   └── server/
│       └── main.go                 # 服务入口
├── api/
│   └── swagger.yml                 # OpenAPI 定义
├── pkg/
│   ├── api/
│   │   ├── handler_dataset.go      # 数据集 API handler
│   │   ├── handler_file.go         # 文件操作 API handler
│   │   ├── handler_version.go      # 版本管理 API handler
│   │   └── handler_lineage.go      # Lineage API handler
│   ├── service/
│   │   ├── dataset.go              # 数据集业务逻辑
│   │   ├── file.go                 # 文件操作业务逻辑
│   │   ├── version.go              # 版本管理业务逻辑
│   │   └── lineage.go              # Lineage 业务逻辑
│   ├── lakefs/
│   │   └── client.go               # lakeFS API 客户端封装
│   ├── lineage/
│   │   ├── store.go                # Lineage 数据存储
│   │   └── dag.go                  # DAG 构建与查询
│   ├── model/
│   │   ├── dataset.go              # 数据模型定义
│   │   ├── version.go
│   │   └── lineage.go
│   └── config/
│       └── config.go               # 配置管理
├── migrations/
│   └── 001_create_lineage_tables.sql
├── deploy/
│   └── docker-compose.yml
├── go.mod
└── go.sum
```

## 6. Python SDK 设计

为用户提供 Python SDK，方便在数据处理脚本和 Notebook 中使用：

```python
from platform_data_sdk import PlatformClient

client = PlatformClient(endpoint="http://platform-service:8080", token="xxx")

# ---- 数据集操作 ----
dataset = client.create_dataset(
    name="training-images",
    display_name="训练图像集",
    description="用于目标检测模型的训练图像"
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
    description="图像预处理流水线"
)

sub_task = client.create_task(
    name="resize-step",
    task_type="preprocessing",
    parent_task_id=task.id
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

## 7. 一期交付范围与排期

### 7.1 里程碑规划

| 阶段 | 时间 | 交付内容 |
|------|------|---------|
| M1: 核心功能 | 3月4日 - 3月14日 | 数据集 CRUD (R1)、文件操作 (R2/R3)、自动版本管理 (R4) |
| M2: 版本管理 | 3月15日 - 3月21日 | 版本列表 (R5)、版本回滚 (R6)、Python SDK 基础版 |
| M3: Data Lineage | 3月22日 - 3月28日 | Lineage 数据模型、任务管理 API、血缘查询 API (R7) |
| M4: 集成测试 | 3月29日 - 3月31日 | 端到端测试、文档完善、Bug 修复 |

### 7.2 一期详细任务分解

**M1: 核心功能（2周）**
- [ ] 搭建 Platform Service 项目骨架
- [ ] 实现 lakeFS 客户端封装
- [ ] 实现数据集 CRUD API
- [ ] 实现文件上传/下载/删除/更新 API
- [ ] 实现文件列表查询 API
- [ ] 实现自动 commit 机制
- [ ] 单元测试

**M2: 版本管理（1周）**
- [ ] 实现版本列表 API
- [ ] 实现版本详情 API（含 diff）
- [ ] 实现版本回滚 API
- [ ] Python SDK 基础版（数据集、文件、版本操作）
- [ ] 集成测试

**M3: Data Lineage（1周）**
- [ ] Lineage 数据库 migration
- [ ] 实现任务 CRUD API
- [ ] 实现任务执行管理 API
- [ ] 实现血缘边记录 API
- [ ] 实现血缘查询 API（上下游、DAG）
- [ ] Python SDK Lineage 部分
- [ ] lakeFS Hook 集成

**M4: 集成测试与收尾（3天）**
- [ ] 端到端测试场景覆盖
- [ ] API 文档最终版
- [ ] 部署文档
- [ ] Bug 修复与性能优化

## 8. 风险与应对

| 风险 | 影响 | 应对策略 |
|------|------|---------|
| lakeFS API 调用延迟 | 用户体验下降 | 批量操作合并 commit；引入缓存层 |
| 数据集大小计算开销 | 大数据集属性查询慢 | 采用写时增量更新 + 异步校准策略 |
| hard_reset API 稳定性 | 回滚功能受限 | 一期优先使用 diff-based 回滚方案 |
| Lineage 数据一致性 | 血缘记录不完整 | 利用 lakeFS Hook 自动记录；SDK 封装确保操作原子性 |
| 开发周期紧张 | 功能不全 | Lineage 可适当简化，先实现核心 CRUD 和查询，DAG 可视化后续迭代 |

## 9. 后续演进（二期规划）

- **数据集权限管理**：基于 lakeFS RBAC，实现数据集级别的读写权限控制
- **数据集分支**：利用 lakeFS branch 支持数据集的多版本并行开发和合并
- **数据集标签 (Tag)**：利用 lakeFS tag 支持为重要版本打标签（如 v1.0、production）
- **数据质量检查**：利用 lakeFS pre-commit hook 实现文件格式校验、Schema 检查
- **全文搜索**：结合 lakeFS Metadata Search 实现文件内容和属性的搜索
- **数据集克隆**：利用 lakeFS 的零拷贝分支特性实现数据集的快速克隆
- **Lineage 可视化 UI**：DAG 图形化展示
- **与调度系统集成**：Airflow / Argo Workflow 自动记录 lineage
