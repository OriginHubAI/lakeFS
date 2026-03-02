# lakeFS 整体架构分析

## 1. 项目概述

**lakeFS** 是一个开源的数据版本控制系统，为数据湖提供 Git-like 的版本控制功能。它将对象存储（S3、Azure Blob Storage、GCS）转换为版本化存储库，支持分支、提交、合并、回滚等 Git 风格操作。

### 核心特性
- **版本控制**: 分支、提交、标签、合并、回滚
- **数据一致性**: 原子性操作，保证数据完整性
- **S3 兼容**: 提供 S3 Gateway，兼容 S3 API
- **多存储支持**: AWS S3、Azure Blob、Google Cloud Storage
- **Hooks 机制**: 支持预/后提交钩子进行数据验证

---

## 2. 整体架构图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              API 层                                          │
├─────────────────────────────────────────────────────────────────────────────┤
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐ │
│  │ REST API     │  │ S3 Gateway   │  │  Web UI      │  │   CLI/Clients    │ │
│  │ (OpenAPI)    │  │ (S3兼容)      │  │  (React)     │  │   (Go/Python)    │ │
│  └──────────────┘  └──────────────┘  └──────────────┘  └──────────────────┘ │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                            服务层 (Service Layer)                            │
├─────────────────────────────────────────────────────────────────────────────┤
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │                      Catalog (目录服务)                                │ │
│  │  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌──────────────────┐  │ │
│  │  │ Repository  │ │   Branch    │ │   Commit    │ │     Object       │  │ │
│  │  │   管理      │ │   管理      │ │   管理      │ │    管理          │  │ │
│  │  └─────────────┘ └─────────────┘ └─────────────┘ └──────────────────┘  │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │                      Graveler (核心版本控制引擎)                        │ │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────┐    │ │
│  │  │ RefManager   │  │StagingManager│  │CommittedMgr  │  │  Hooks   │    │ │
│  │  │ (引用管理)    │  │ (暂存区管理)  │  │ (提交管理)   │  │(钩子处理)│    │ │
│  │  └──────────────┘  └──────────────┘  └──────────────┘  └──────────┘    │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐    │ │
│  │ Auth Service │  │Action Service│  │  GC Service  │  │Import Service│    │ │
│  │ (认证授权)   │  │ (动作钩子)   │  │ (垃圾回收)   │  │ (数据导入)   │    │ │
│  └──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘    │ │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           存储层 (Storage Layer)                             │
├─────────────────────────────────────────────────────────────────────────────┤
│  ┌──────────────────────────┐      ┌──────────────────────────────────────┐ │
│  │    KV Store (元数据)      │      │         Object Store (数据)           │ │
│  │  ┌────────────────────┐  │      │  ┌─────────────────────────────────┐  │ │
│  │  │ PostgreSQL/DynamoDB│  │      │  │  S3 / Azure Blob / GCS         │  │ │
│  │  │                    │  │      │  │                                 │  │ │
│  │  │ 存储: 分支、提交、   │  │      │  │  存储: 实际数据对象              │  │ │
│  │  │       标签、配置    │  │      │  │       MetaRange/Range文件      │  │ │
│  │  └────────────────────┘  │      │  └─────────────────────────────────┘  │ │
│  └──────────────────────────┘      └──────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. 核心组件详解

### 3.1 API 层

#### 3.1.1 REST API (`pkg/api/`)
- **实现**: [`pkg/api/controller.go`](pkg/api/controller.go)
- **规范**: OpenAPI 3.0 ([`api/swagger.yml`](api/swagger.yml))
- **功能**: 完整的 HTTP REST API，包含以下模块：
  - **认证**: Login、User/Group/Policy 管理
  - **仓库**: Repository CRUD、元数据管理
  - **分支**: Branch CRUD、保护规则
  - **对象**: Object 上传/下载/删除/复制
  - **提交**: Commit、Revert、Cherry-pick、Merge
  - **标签**: Tag 管理
  - **Actions**: Hooks 执行管理
  - **GC**: 垃圾回收管理

#### 3.1.2 S3 Gateway (`pkg/gateway/`)
- **实现**: [`pkg/gateway/handler.go`](pkg/gateway/handler.go)
- **功能**: 提供 S3 兼容的 API 接口
- **支持的操作**:
  - `GetObject`, `PutObject`, `DeleteObject`
  - `ListObjects`, `HeadObject`
  - `CreateBucket`, `DeleteBucket`
  - `Multipart Upload`
- **映射**: S3 请求映射到 lakeFS 的分支和路径

#### 3.1.3 客户端 SDK
- **Go**: `clients/go/` - 官方 Go 客户端
- **Python**: `clients/python/` - Python 客户端
- **Java**: `clients/java/` - Java/Scala 客户端
- **Rust**: `clients/rust/` - Rust 客户端
- **HadoopFS**: `clients/hadoopfs/` - Hadoop 文件系统适配器

---

### 3.2 服务层核心

#### 3.2.1 Graveler - 版本控制引擎
**位置**: [`pkg/graveler/graveler.go`](pkg/graveler/graveler.go)

Graveler 是 lakeFS 的核心版本控制引擎，负责：

##### 核心概念
```go
// 基本类型定义
Repository    // 仓库，对应一个数据湖
Branch        // 分支，包含当前 commit 和 staging 区域
Commit        // 提交，包含元数据和 MetaRangeID
Tag           // 标签，指向特定 commit
MetaRange     // 提交的快照，包含多个 Range
Range         // 数据分片，存储有序的 KV 记录
StagingToken  // 暂存区标识
```

##### 主要组件

**1. RefManager** ([`pkg/graveler/ref/`])
- 管理分支、标签、提交的引用
- 解析引用（如 `main~2`, `feature-branch`）
- 计算 merge-base
- 提交历史遍历 (Log)

**2. StagingManager** ([`pkg/graveler/staging/`])
- 管理分支的暂存区 (staging area)
- 处理未提交的变更
- 支持多个 sealed tokens 和当前 staging token

**3. CommittedManager** ([`pkg/graveler/committed/`])
- 管理已提交的数据 (MetaRange/Range)
- 实现 diff、merge、compare 操作
- 处理数据去重和压缩

**4. Hooks 系统**
- **Pre-hooks**: PreCommit、PreMerge、PreRevert 等
- **Post-hooks**: PostCommit、PostMerge 等
- 支持 Lua 脚本 ([`examples/hooks/`](examples/hooks/))

##### 核心操作接口

```go
// VersionController 接口 (graveler.go:656)
type VersionController interface {
    // 仓库管理
    CreateRepository(ctx, repoID, storageNS, branchID) (*RepositoryRecord, error)
    DeleteRepository(ctx, repoID) error
    
    // 分支管理
    CreateBranch(ctx, repo, branchID, ref) (*Branch, error)
    DeleteBranch(ctx, repo, branchID) error
    
    // 提交操作
    Commit(ctx, repo, branchID, params) (CommitID, error)
    Revert(ctx, repo, branchID, ref) (CommitID, error)
    CherryPick(ctx, repo, branchID, ref) (CommitID, error)
    
    // 合并操作
    Merge(ctx, repo, destBranch, sourceRef) (CommitID, error)
    FindMergeBase(ctx, repo, from, to) (*Commit, error)
    
    // Diff 操作
    Diff(ctx, repo, left, right) (DiffIterator, error)
    Compare(ctx, repo, left, right) (DiffIterator, error)
    DiffUncommitted(ctx, repo, branchID) (DiffIterator, error)
    
    // KV 操作
    Get(ctx, repo, ref, key) (*Value, error)
    Set(ctx, repo, branchID, key, value) error
    Delete(ctx, repo, branchID, key) error
    List(ctx, repo, ref) (ValueIterator, error)
}
```

#### 3.2.2 Catalog 层
**位置**: [`pkg/catalog/`](pkg/catalog/)

Catalog 是 API 层与 Graveler 之间的适配层，提供更高层次的抽象：

```go
// Catalog 核心功能
type Catalog struct {
    BlockAdapter  // 对象存储适配器
    Store         // Graveler 实例
    ...
}

// 主要方法
func (c *Catalog) GetEntry(ctx, repo, ref, path) (*DBEntry, error)
func (c *Catalog) CreateEntry(ctx, repo, branch, entry) error
func (c *Catalog) DeleteEntry(ctx, repo, branch, path) error
func (c *Catalog) ListEntries(ctx, repo, ref, params) ([]*DBEntry, bool, error)
func (c *Catalog) Commit(ctx, repo, branch, message, metadata) (*CommitLog, error)
```

#### 3.2.3 认证授权系统
**位置**: [`pkg/auth/`](pkg/auth/)

##### 组件
- **Authenticator**: 用户认证（Basic Auth、OIDC、SAML、LDAP）
- **Service**: 用户/组/策略管理
- **Policy**: RBAC 权限模型
- **Credentials**: Access Key / Secret Key 管理

##### 权限模型
```yaml
# 权限结构
Permission:
  Action: "fs:WriteObject"  # 操作类型
  Resource: "arn:lakefs:fs:::repository/branch/path"  # 资源ARN

# 策略示例
Policy:
  Statement:
    - Effect: "Allow"
      Action: ["fs:ReadObject", "fs:ListObjects"]
      Resource: "arn:lakefs:fs:::my-repo/main/*"
```

#### 3.2.4 Block Adapter 存储适配器
**位置**: [`pkg/block/`](pkg/block/)

提供统一的存储接口，支持多种后端：

```go
// Block Adapter 接口
type Adapter interface {
    Get(ctx, objPointer) (io.ReadCloser, error)
    Put(ctx, objPointer, size, reader, opts) (*UploadResult, error)
    Delete(ctx, objPointer) error
    CreateMultiPartUpload(ctx, objPointer) (*MultipartUpload, error)
    GetPreSignedURL(ctx, objPointer, mode) (string, time.Time, error)
    ...
}
```

**支持的存储后端**:
- **S3**: AWS S3 及兼容服务（MinIO、Ceph）
- **Azure**: Azure Blob Storage
- **GCS**: Google Cloud Storage
- **Local**: 本地文件系统（开发/测试）

---

### 3.3 数据存储层

#### 3.3.1 KV 存储（元数据）
**位置**: [`pkg/kv/`](pkg/kv/)

存储所有元数据：仓库、分支、提交、标签、用户、策略等。

**支持的后端**:
- PostgreSQL
- DynamoDB
- CosmosDB
- Local (内存/BadgerDB，开发测试)

#### 3.3.2 对象存储（数据）

##### MetaRange / Range 文件格式
```
Storage Namespace
├── _lakefs/
│   └── data/
│       ├── meta_range/           # MetaRange 文件
│       │   └── <meta_range_id>
│       └── range/                # Range 文件
│           └── <range_id>
└── <user-data>/                  # 用户实际数据
```

##### 数据模型
```go
// Value 存储结构
type Value struct {
    Identity []byte  // 内容哈希，用于去重
    Data     []byte  // 实际数据或元数据
}

// Commit 结构
type Commit struct {
    Version      CommitVersion
    Committer    string
    Message      string
    MetaRangeID  MetaRangeID    // 指向提交的快照
    CreationDate time.Time
    Parents      CommitParents  // 父提交
    Metadata     Metadata       // 自定义元数据
    Generation   CommitGeneration
}

// Branch 结构
type Branch struct {
    CommitID                 CommitID
    StagingToken            StagingToken
    SealedTokens            []StagingToken
    CompactedBaseMetaRangeID MetaRangeID
    Hidden                  bool
}
```

---

## 4. 核心流程详解

### 4.1 写入流程

```
用户请求 (PUT /repos/{repo}/branches/{branch}/objects/{path})
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ 1. 接收请求，生成物理存储路径                                │
│    - 使用 PathProvider 生成随机路径                          │
│    - 格式: data/{随机ID}/{文件名}                            │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ 2. 写入对象存储                                              │
│    - 调用 BlockAdapter.Put()                                 │
│    - 数据写入底层存储（S3/Azure/GCS）                        │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ 3. 创建 Entry 并写入 Staging                                 │
│    - 构建 DBEntry（path, physical_address, checksum, size）  │
│    - 调用 Graveler.Set() 写入 staging                        │
│    - 实际写入 KV 的 staging 区域                             │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
              返回成功
```

### 4.2 提交流程 (Commit)

```
用户请求 (POST /repos/{repo}/branches/{branch}/commits)
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ 1. PreCommit Hook                                            │
│    - 执行预提交钩子（如数据验证）                            │
│    - 失败则中止提交                                          │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ 2. 合并 Staging → Committed                                  │
│    - 读取分支的 SealedTokens                                 │
│    - 合并所有 staging 变更                                   │
│    - 调用 CommittedManager.Commit()                          │
│      - 创建新的 MetaRange                                    │
│      - 写入 Range/MetaRange 文件到对象存储                   │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ 3. 创建 Commit 记录                                          │
│    - 生成 Commit 对象                                        │
│    - 写入 KV Store                                           │
│    - 更新分支的 CommitID                                     │
│    - 清理 SealedTokens                                       │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ 4. PostCommit Hook                                           │
│    - 执行后提交钩子（如通知、导出）                          │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
              返回 CommitID
```

### 4.3 读取流程

```
用户请求 (GET /repos/{repo}/refs/{ref}/objects/{path})
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ 1. 解析引用 (Resolve Ref)                                    │
│    - 将 ref (branch/tag/commit) 解析为 CommitID              │
│    - 处理修饰符（~2, ^, @ 等）                               │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ 2. 查找 Entry                                                │
│    - 如果是分支，先查 Staging 区域                           │
│    - 再查 Committed (通过 MetaRange)                         │
│    - 返回 Entry 元数据                                       │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ 3. 返回数据                                                  │
│    - 从 Entry 获取 PhysicalAddress                           │
│    - 调用 BlockAdapter.Get() 读取对象存储                    │
│    - 流式返回数据                                            │
└─────────────────────────────────────────────────────────────┘
```

### 4.4 合并流程 (Merge)

```
用户请求 (POST /repos/{repo}/refs/{source}/merge/{dest})
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ 1. 验证和准备                                                │
│    - 检查目标分支是否干净（无未提交变更）                    │
│    - 执行 PreMerge Hook                                      │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ 2. 计算 Merge Base                                           │
│    - FindMergeBase: 找到两个分支的最近公共祖先               │
│    - 使用图遍历算法（基于提交历史）                          │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ 3. 三向合并 (3-way Merge)                                    │
│    - 比较: Base vs Source, Base vs Dest                      │
│    - 应用策略处理冲突：                                       │
│      - dest-wins: 目标分支优先                               │
│      - source-wins: 源分支优先                               │
│      - none: 遇到冲突报错                                    │
│    - 生成新的 MetaRange                                      │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────────┐
│ 4. 创建 Merge Commit                                         │
│    - 有两个父提交（源和目标）                                │
│    - 更新目标分支指向新提交                                  │
│    - 执行 PostMerge Hook                                     │
└─────────────────────────────────────────────────────────────┘
                    │
                    ▼
              返回 CommitID
```

---

## 5. 模块接口定义

### 5.1 Graveler 核心接口

```go
// 文件: pkg/graveler/graveler.go

// KeyValueStore - 基本的 KV 操作
type KeyValueStore interface {
    Get(ctx context.Context, repository *RepositoryRecord, ref Ref, key Key) (*Value, error)
    Set(ctx context.Context, repository *RepositoryRecord, branchID BranchID, key Key, value Value) error
    Delete(ctx context.Context, repository *RepositoryRecord, branchID BranchID, key Key) error
    List(ctx context.Context, repository *RepositoryRecord, ref Ref, batchSize int) (ValueIterator, error)
}

// VersionController - 版本控制操作
type VersionController interface {
    KeyValueStore
    
    // 仓库管理
    GetRepository(ctx context.Context, repositoryID RepositoryID) (*RepositoryRecord, error)
    CreateRepository(ctx context.Context, repositoryID RepositoryID, storageID StorageID, 
                     storageNamespace StorageNamespace, branchID BranchID, readOnly bool) (*RepositoryRecord, error)
    DeleteRepository(ctx context.Context, repositoryID RepositoryID) error
    
    // 分支管理
    CreateBranch(ctx context.Context, repository *RepositoryRecord, branchID BranchID, 
                 ref Ref, opts ...SetOptionsFunc) (*Branch, error)
    DeleteBranch(ctx context.Context, repository *RepositoryRecord, branchID BranchID) error
    GetBranch(ctx context.Context, repository *RepositoryRecord, branchID BranchID) (*Branch, error)
    ListBranches(ctx context.Context, repository *RepositoryRecord) (BranchIterator, error)
    
    // 提交操作
    Commit(ctx context.Context, repository *RepositoryRecord, branchID BranchID, 
           commitParams CommitParams) (CommitID, error)
    Revert(ctx context.Context, repository *RepositoryRecord, branchID BranchID, 
           ref Ref, parentNumber int) (CommitID, error)
    CherryPick(ctx context.Context, repository *RepositoryRecord, branchID BranchID, 
               ref Ref, parentNumber *int) (CommitID, error)
    
    // 合并操作
    Merge(ctx context.Context, repository *RepositoryRecord, destination BranchID, 
          source Ref, commitParams CommitParams, strategy string) (CommitID, error)
    FindMergeBase(ctx context.Context, repository *RepositoryRecord, from Ref, to Ref) 
                  (*CommitRecord, *CommitRecord, *Commit, error)
    
    // Diff 操作
    Diff(ctx context.Context, repository *RepositoryRecord, left, right Ref) (DiffIterator, error)
    Compare(ctx context.Context, repository *RepositoryRecord, left, right Ref) (DiffIterator, error)
    DiffUncommitted(ctx context.Context, repository *RepositoryRecord, branchID BranchID) (DiffIterator, error)
    
    // 标签管理
    CreateTag(ctx context.Context, repository *RepositoryRecord, tagID TagID, commitID CommitID) error
    DeleteTag(ctx context.Context, repository *RepositoryRecord, tagID TagID) error
    GetTag(ctx context.Context, repository *RepositoryRecord, tagID TagID) (*CommitID, error)
    
    // GC 管理
    GetGarbageCollectionRules(ctx context.Context, repository *RepositoryRecord) (*GarbageCollectionRules, error)
    SetGarbageCollectionRules(ctx context.Context, repository *RepositoryRecord, rules *GarbageCollectionRules) error
    SaveGarbageCollectionCommits(ctx context.Context, repository *RepositoryRecord) (*GarbageCollectionRunMetadata, error)
    
    // 分支保护
    GetBranchProtectionRules(ctx context.Context, repository *RepositoryRecord) (*BranchProtectionRules, *string, error)
    SetBranchProtectionRules(ctx context.Context, repository *RepositoryRecord, 
                            rules *BranchProtectionRules, lastKnownChecksum *string) error
}

// Plumbing - 内部维护接口
type Plumbing interface {
    GetMetaRange(ctx context.Context, repository *RepositoryRecord, metaRangeID MetaRangeID) (MetaRangeAddress, error)
    GetRange(ctx context.Context, repository *RepositoryRecord, rangeID RangeID) (RangeAddress, error)
    WriteRange(ctx context.Context, repository *RepositoryRecord, it ValueIterator) (*RangeInfo, error)
    WriteMetaRange(ctx context.Context, repository *RepositoryRecord, ranges []*RangeInfo) (*MetaRangeInfo, error)
}
```

### 5.2 Block Adapter 接口

```go
// 文件: pkg/block/adapter.go

type Adapter interface {
    // 基础操作
    Get(ctx context.Context, objPointer ObjectPointer) (io.ReadCloser, error)
    GetRange(ctx context.Context, objPointer ObjectPointer, start, end int64) (io.ReadCloser, error)
    Put(ctx context.Context, objPointer ObjectPointer, size int64, reader io.Reader, opts PutOpts) (*UploadResult, error)
    Delete(ctx context.Context, objPointer ObjectPointer) error
    
    // 分片上传
    CreateMultiPartUpload(ctx context.Context, objPointer ObjectPointer, r *http.Request, 
                         opts CreateMultiPartUploadOpts) (*MultipartUploadCreationResponse, error)
    UploadPart(ctx context.Context, objPointer ObjectPointer, size int64, reader io.Reader, 
               uploadID string, partNumber int) (*UploadPartResponse, error)
    AbortMultiPartUpload(ctx context.Context, objPointer ObjectPointer, uploadID string) error
    CompleteMultiPartUpload(ctx context.Context, objPointer ObjectPointer, uploadID string, 
                           multipartList *MultipartUploadCompletion) (*CompleteMultipartUploadResponse, error)
    
    // 预签名 URL
    GetPreSignedURL(ctx context.Context, objPointer ObjectPointer, mode PreSignMode, 
                    contentDisposition string) (string, time.Time, error)
    
    // 元数据
    HeadObject(ctx context.Context, objPointer ObjectPointer) (*ObjectStats, error)
    GetStorageNamespaceInfo(storageID string) *StorageNamespaceInfo
}
```

### 5.3 Auth 服务接口

```go
// 文件: pkg/auth/service.go

type Service interface {
    // 用户管理
    CreateUser(ctx context.Context, user *model.User) (*model.User, error)
    DeleteUser(ctx context.Context, username string) error
    GetUser(ctx context.Context, username string) (*model.User, error)
    ListUsers(ctx context.Context, params *model.PaginationParams) ([]*model.User, *model.Paginator, error)
    
    // 组管理
    CreateGroup(ctx context.Context, group *model.Group) (*model.Group, error)
    DeleteGroup(ctx context.Context, groupID string) error
    GetGroup(ctx context.Context, groupID string) (*model.Group, error)
    ListGroups(ctx context.Context, params *model.PaginationParams) ([]*model.Group, *model.Paginator, error)
    AddUserToGroup(ctx context.Context, username, groupID string) error
    RemoveUserFromGroup(ctx context.Context, username, groupID string) error
    ListGroupUsers(ctx context.Context, groupID string, params *model.PaginationParams) ([]*model.User, *model.Paginator, error)
    ListUserGroups(ctx context.Context, username string, params *model.PaginationParams) ([]*model.Group, *model.Paginator, error)
    
    // 策略管理
    WritePolicy(ctx context.Context, policy *model.Policy, overwrite bool) error
    DeletePolicy(ctx context.Context, policyID string) error
    GetPolicy(ctx context.Context, policyID string) (*model.Policy, error)
    ListPolicies(ctx context.Context, params *model.PaginationParams) ([]*model.Policy, *model.Paginator, error)
    AttachPolicyToUser(ctx context.Context, policyID, username string) error
    DetachPolicyFromUser(ctx context.Context, policyID, username string) error
    AttachPolicyToGroup(ctx context.Context, policyID, groupID string) error
    DetachPolicyFromGroup(ctx context.Context, policyID, groupID string) error
    
    // 授权
    Authorize(ctx context.Context, req *AuthorizationRequest) (*AuthorizationResponse, error)
}
```

---

## 6. 目录结构

```
lakeFS/
├── api/                    # OpenAPI 定义
│   └── swagger.yml         # API 规范
├── cmd/                    # 命令行入口
│   └── lakefs/            # 主程序
├── contrib/               # 贡献组件
│   └── auth/acl/          # ACL 认证实现
├── pkg/                   # 核心代码
│   ├── api/               # REST API 实现
│   ├── auth/              # 认证授权
│   ├── block/             # 存储适配器
│   ├── catalog/           # 目录服务
│   ├── config/            # 配置管理
│   ├── gateway/           # S3 Gateway
│   ├── graveler/          # 版本控制引擎
│   ├── httputil/          # HTTP 工具
│   ├── ident/             # 标识生成
│   ├── kv/                # KV 存储抽象
│   ├── logging/           # 日志工具
│   ├── permissions/       # 权限定义
│   ├── stats/             # 统计收集
│   └── upload/            # 上传处理
├── clients/               # 客户端 SDK
│   ├── go/               # Go 客户端
│   ├── java/             # Java 客户端
│   ├── python/           # Python 客户端
│   ├── rust/             # Rust 客户端
│   ├── hadoopfs/         # Hadoop 文件系统
│   └── spark/            # Spark 集成
├── docs/                 # 文档
├── esti/                 # 集成测试
├── examples/             # 示例代码
│   └── hooks/            # Hooks 示例
├── modules/              # 模块化组件
├── scripts/              # 脚本
├── test/                 # 测试工具
└── webui/                # Web UI (React)
```

---

## 7. 依赖关系

```
┌────────────────────────────────────────────────────────────┐
│                      API 层                                 │
│  pkg/api/, pkg/gateway/, webui/                            │
└────────────────────────────────────────────────────────────┘
                          │
                          ▼
┌────────────────────────────────────────────────────────────┐
│                     Service 层                              │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
│  │ Catalog  │  │ Actions  │  │   Auth   │  │   GC     │   │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘   │
└────────────────────────────────────────────────────────────┘
                          │
                          ▼
┌────────────────────────────────────────────────────────────┐
│                  Core Engine 层                             │
│              pkg/graveler/ (版本控制引擎)                    │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │
│  │  RefManager  │  │StagingManager│  │CommittedMgr  │     │
│  └──────────────┘  └──────────────┘  └──────────────┘     │
└────────────────────────────────────────────────────────────┘
          │                      │
          ▼                      ▼
┌──────────────────┐  ┌────────────────────────────────────┐
│   KV Store       │  │         Object Store               │
│ pkg/kv/          │  │      pkg/block/                    │
│ • PostgreSQL     │  │  • S3 / Azure / GCS / Local        │
│ • DynamoDB       │  │  • Range/MetaRange 文件           │
│ • CosmosDB       │  │  • 用户数据对象                    │
│ • Local          │  │                                    │
└──────────────────┘  └────────────────────────────────────┘
```

---

## 8. 总结

lakeFS 的架构设计体现了以下核心思想：

1. **分层架构**: 清晰的 API 层、服务层、存储层分离
2. **Git-like 版本控制**: Graveler 引擎实现分支、提交、合并等核心功能
3. **存储解耦**: 元数据与数据分离，支持多种 KV 和对象存储后端
4. **S3 兼容**: Gateway 层提供无缝的 S3 API 兼容
5. **可扩展性**: Hooks 机制和模块化设计支持丰富的扩展场景

通过 Graveler 的核心版本控制引擎，lakeFS 在对象存储之上构建了强大的数据版本管理能力，使数据湖具备了与代码仓库类似的版本控制、协作和治理能力。
