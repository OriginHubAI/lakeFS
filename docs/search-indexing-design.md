# lakeFS 全文索引与向量索引扩展设计文档

> 创建时间：2026-03-03
> 状态：设计阶段
> 作者：PSF Team

---

## 一、背景与目标

### 1.1 问题现状

lakeFS 当前的对象检索能力仅限于：
- **前缀列表**：`ListObjects` 按路径前缀 + 分隔符遍历
- **精确查找**：`StatObject` 按完整路径获取元数据
- **仓库搜索**：`SearchString` 仅支持仓库 ID 子串匹配

缺失的能力：
- 无法按文件内容搜索（"哪些 CSV 包含 `user_id=12345`"）
- 无法按元数据模糊搜索（"所有带 `pii=true` 标签的文件"）
- 无法做语义搜索（"找到与这张图片相似的数据集"）

### 1.2 设计目标

| 目标 | 说明 |
|------|------|
| **全文搜索** | 支持对象路径、元数据、文本内容的全文检索 |
| **向量搜索** | 支持基于 embedding 的语义相似度搜索 |
| **版本感知** | 搜索结果绑定到特定 ref（branch/commit/tag）|
| **最小侵入** | 不修改 Graveler/SSTable 核心存储层 |
| **渐进实施** | 分阶段交付，每阶段独立可用 |

### 1.3 参考设计

Hugging Face Hub 的做法：
- 全文搜索：DuckDB + Parquet + BM25，服务端按需计算
- 向量搜索：不内建，交给 FAISS/Lance/DuckDB 用户侧处理
- 元数据搜索：预计算 + MongoDB 缓存

本设计借鉴 HF 思路但适配 lakeFS 的版本控制特性。

---

## 二、整体架构

```
┌─────────────────────────────────────────────────────────────────┐
│                         lakeFS Server                           │
│                                                                 │
│  ┌──────────┐   ┌──────────────┐   ┌─────────────────────────┐  │
│  │ OpenAPI  │   │  Controller  │   │    Catalog (existing)   │  │
│  │ swagger  │──→│  /search     │──→│  ListEntries / GetEntry │  │
│  │ .yml     │   │  /vector     │   └─────────────────────────┘  │
│  └──────────┘   └──────┬───────┘                                │
│                        │                                        │
│                        ▼                                        │
│              ┌─────────────────────┐                            │
│              │   Search Engine     │  ← 新增模块                 │
│              │   pkg/search/       │                            │
│              │                     │                            │
│              │  ┌───────────────┐  │                            │
│              │  │ FullText      │  │  DuckDB + FTS extension    │
│              │  │ Engine        │  │                            │
│              │  └───────────────┘  │                            │
│              │  ┌───────────────┐  │                            │
│              │  │ Vector        │  │  DuckDB vss / 外部引擎      │
│              │  │ Engine        │  │                            │
│              │  └───────────────┘  │                            │
│              │  ┌───────────────┐  │                            │
│              │  │ Index         │  │  异步构建 + 缓存管理         │
│              │  │ Manager       │  │                            │
│              │  └───────────────┘  │                            │
│              └──────────┬──────────┘                            │
│                         │                                       │
│                         ▼                                       │
│              ┌─────────────────────┐                            │
│              │  Block Adapter      │  读取对象内容                │
│              │  (existing)         │                            │
│              └─────────────────────┘                            │
└─────────────────────────────────────────────────────────────────┘
                          │
                          ▼
              ┌─────────────────────┐
              │  Index Storage      │
              │  _lakefs_search/    │  索引文件存储在对象存储中     │
              │  ├── ft_index/      │  全文索引                   │
              │  └── vec_index/     │  向量索引                   │
              └─────────────────────┘
```

### 核心设计原则

1. **索引与数据分离**：索引存储在 `_lakefs_search/` 命名空间下，不影响用户数据
2. **按需构建**：索引通过 post-commit hook 或手动 API 触发构建
3. **ref 绑定**：每个索引快照关联到具体的 commit ID
4. **DuckDB 作为执行引擎**：进程内执行，无外部依赖
5. **渐进增强**：全文搜索先行，向量搜索后续扩展

---

## 三、Phase 1：全文搜索

### 3.1 功能范围

| 搜索类型 | 搜索目标 | 优先级 |
|----------|---------|:------:|
| **路径搜索** | 对象路径的模糊匹配 | P0 |
| **元数据搜索** | 用户自定义 metadata key/value | P0 |
| **内容搜索** | 文本类文件内容（CSV, JSON, TXT, Markdown 等） | P1 |
| **系统元数据搜索** | size, content_type, last_modified | P1 |

### 3.2 索引架构

#### 索引数据模型

全文索引采用 DuckDB 的 FTS (Full-Text Search) 扩展，底层基于 BM25 算法。

索引表结构：

```sql
-- 对象元数据索引表
CREATE TABLE object_index (
    path            VARCHAR NOT NULL,    -- 对象路径
    size            BIGINT,              -- 文件大小
    last_modified   TIMESTAMP,           -- 最后修改时间
    etag            VARCHAR,             -- ETag
    content_type    VARCHAR,             -- MIME 类型
    metadata_json   VARCHAR,             -- 用户 metadata (JSON 序列化)
    content_preview VARCHAR,             -- 文本内容预览 (前 64KB)
    commit_id       VARCHAR NOT NULL,    -- 关联的 commit ID

    PRIMARY KEY (path)
);

-- 创建全文搜索索引
INSTALL fts;
LOAD fts;

PRAGMA create_fts_index(
    'object_index',
    'path',                              -- document ID 列
    'path', 'metadata_json', 'content_preview', 'content_type',
    stemmer = 'porter',
    stopwords = 'english',
    ignore = '(\\.|/)[^/]*',
    strip_accents = true,
    lower = true
);
```

#### 索引存储

```
<storage_namespace>/_lakefs_search/
├── <repository_id>/
│   ├── ft_index/
│   │   ├── <commit_id_short>.duckdb       -- DuckDB 数据库文件
│   │   └── manifest.json                   -- 索引元信息
│   └── vec_index/
│       └── ...
```

`manifest.json` 结构：

```json
{
  "version": 1,
  "repository": "my-repo",
  "commit_id": "abc123def456",
  "created_at": "2026-03-03T10:00:00Z",
  "index_type": "fulltext",
  "engine": "duckdb-fts",
  "stats": {
    "total_objects": 150000,
    "indexed_objects": 148500,
    "content_indexed": 42000,
    "index_size_bytes": 52428800,
    "build_duration_ms": 35000
  },
  "config": {
    "stemmer": "porter",
    "content_max_bytes": 65536,
    "content_types": ["text/*", "application/json", "application/csv"]
  }
}
```

### 3.3 索引构建流程

#### 方式一：API 手动触发（推荐初期使用）

```
POST /api/v1/repositories/{repository}/search/index
{
  "ref": "main",
  "index_type": "fulltext",
  "config": {
    "include_content": true,
    "content_types": ["text/*", "application/json"],
    "content_max_bytes": 65536
  }
}

Response:
{
  "task_id": "idx-abc123",
  "status": "in_progress"
}
```

#### 方式二：Post-Commit Hook 自动触发

```yaml
# _lakefs_actions/search_index.yaml
name: auto-index
on:
  post-commit:
    branches: ["main"]
hooks:
  - id: rebuild_search_index
    type: webhook
    properties:
      url: "http://localhost:8000/api/v1/repositories/{{.RepositoryID}}/search/index"
      method: POST
      body: |
        {
          "ref": "{{.CommitID}}",
          "index_type": "fulltext",
          "mode": "incremental"
        }
```

#### 方式三：增量构建

增量模式通过 diff 计算变更：

```
1. 获取上次索引的 commit_id (from manifest.json)
2. diff(last_indexed_commit, current_commit) → 变更列表
3. 对 added/modified 的对象：读取内容并更新索引
4. 对 deleted 的对象：从索引中移除
5. 保存新的 DuckDB 文件和 manifest
```

#### 构建流程详细步骤

```
┌──────────────┐     ┌─────────────────┐     ┌──────────────────┐
│  触发构建     │────→│  解析 Ref        │────→│  获取 Diff       │
│  (API/Hook)  │     │  commit_id      │     │  (增量) 或        │
└──────────────┘     └─────────────────┘     │  ListEntries     │
                                             │  (全量)          │
                                             └────────┬─────────┘
                                                       │
                                                       ▼
                                              ┌──────────────────┐
                                              │  迭代对象列表      │
                                              │  for each entry: │
                                              │  1. 提取 metadata │
                                              │  2. 判断内容类型   │
                                              │  3. 读取内容预览   │
                                              │    (BlockAdapter)│
                                              └────────┬─────────┘
                                                       │
                                                       ▼
                                              ┌──────────────────┐
                                              │  写入 DuckDB      │
                                              │  1. INSERT/UPDATE│
                                              │  2. 重建 FTS 索引 │
                                              │  3. 导出 .duckdb  │
                                              └────────┬─────────┘
                                                       │
                                                       ▼
                                              ┌──────────────────┐
                                              │  上传到对象存储    │
                                              │  _lakefs_search/ │
                                              │  + manifest.json │
                                              └──────────────────┘
```

### 3.4 查询 API

#### 搜索端点

```
GET /api/v1/repositories/{repository}/search
```

参数：

| 参数 | 类型 | 必填 | 说明 |
|------|------|:----:|------|
| `ref` | string | 是 | 分支名、commit ID 或 tag |
| `query` | string | 是 | 搜索关键词 |
| `scope` | string | 否 | 搜索范围：`path`, `metadata`, `content`, `all` (默认 `all`) |
| `prefix` | string | 否 | 限定路径前缀 |
| `content_type` | string | 否 | 过滤 MIME 类型 |
| `size_min` | int64 | 否 | 最小文件大小 |
| `size_max` | int64 | 否 | 最大文件大小 |
| `after` | string | 否 | 分页游标 |
| `amount` | int | 否 | 返回数量 (默认 20, 最大 100) |

响应：

```json
{
  "results": [
    {
      "path": "data/users/profiles.csv",
      "score": 12.45,
      "size": 1048576,
      "last_modified": "2026-03-01T08:00:00Z",
      "content_type": "text/csv",
      "metadata": {"department": "analytics", "pii": "true"},
      "highlights": {
        "path": null,
        "metadata": null,
        "content": "...user_id,name,email...<mark>user_id=12345</mark>..."
      }
    }
  ],
  "pagination": {
    "has_more": true,
    "next_offset": "data/users/profiles.csv",
    "results": 20,
    "max_per_page": 100
  },
  "index_info": {
    "commit_id": "abc123def456",
    "indexed_at": "2026-03-03T10:00:00Z",
    "total_indexed": 148500
  }
}
```

#### 查询执行流程

```
1. 解析 ref → 定位最近的索引快照
   - 精确匹配 commit_id
   - 或找到 ref 对应 commit 的最近祖先索引
2. 从对象存储下载 DuckDB 文件到本地缓存
3. 打开 DuckDB 连接，执行搜索查询
4. 返回结果

SQL 内部执行：

SELECT
    path,
    fts_main_object_index.match_bm25(path, ?) AS score,
    size,
    last_modified,
    content_type,
    metadata_json,
    content_preview
FROM object_index
WHERE score IS NOT NULL
  AND (? IS NULL OR path LIKE ? || '%')        -- prefix filter
  AND (? IS NULL OR content_type LIKE ?)        -- content_type filter
  AND (? IS NULL OR size >= ?)                  -- size_min filter
  AND (? IS NULL OR size <= ?)                  -- size_max filter
ORDER BY score DESC
LIMIT ? OFFSET ?;
```

### 3.5 Go 实现设计

#### 包结构

```
pkg/search/
├── engine.go           -- SearchEngine 接口定义
├── config.go           -- 搜索配置
├── fulltext/
│   ├── indexer.go      -- 全文索引构建器
│   ├── searcher.go     -- 全文搜索执行器
│   ├── duckdb.go       -- DuckDB 操作封装
│   └── cache.go        -- 索引文件本地缓存
├── vector/
│   ├── indexer.go      -- 向量索引构建器
│   ├── searcher.go     -- 向量搜索执行器
│   └── embedder.go     -- Embedding 生成接口
├── manager.go          -- 索引生命周期管理
└── store.go            -- 索引存储 (对象存储交互)
```

#### 核心接口

```go
package search

import "context"

// SearchEngine 统一搜索引擎接口
type SearchEngine interface {
    // Search 执行搜索
    Search(ctx context.Context, params SearchParams) (*SearchResult, error)
    // BuildIndex 构建/更新索引
    BuildIndex(ctx context.Context, params BuildIndexParams) (*BuildIndexResult, error)
    // GetIndexStatus 获取索引状态
    GetIndexStatus(ctx context.Context, repo, ref string) (*IndexStatus, error)
    // DeleteIndex 删除索引
    DeleteIndex(ctx context.Context, repo, commitID string) error
}

type SearchParams struct {
    Repository  string
    Ref         string
    Query       string
    Scope       SearchScope   // path | metadata | content | all
    Prefix      string
    ContentType string
    SizeMin     *int64
    SizeMax     *int64
    After       string
    Amount      int
}

type SearchScope string

const (
    ScopePath     SearchScope = "path"
    ScopeMetadata SearchScope = "metadata"
    ScopeContent  SearchScope = "content"
    ScopeAll      SearchScope = "all"
)

type SearchResult struct {
    Results   []SearchHit
    HasMore   bool
    NextAfter string
    IndexInfo IndexInfo
}

type SearchHit struct {
    Path        string
    Score       float64
    Size        int64
    LastModified time.Time
    ContentType string
    Metadata    map[string]string
    Highlights  map[string]string   // field → highlighted snippet
}

type BuildIndexParams struct {
    Repository     string
    Ref            string
    IndexType      IndexType
    Mode           BuildMode         // full | incremental
    IncludeContent bool
    ContentTypes   []string          // glob patterns for content indexing
    ContentMaxBytes int64
}

type IndexType string

const (
    IndexTypeFullText IndexType = "fulltext"
    IndexTypeVector   IndexType = "vector"
)

type BuildMode string

const (
    BuildModeFull        BuildMode = "full"
    BuildModeIncremental BuildMode = "incremental"
)

type IndexStatus struct {
    CommitID       string
    IndexedAt      time.Time
    TotalObjects   int64
    IndexedObjects int64
    IndexSize      int64
    IndexType      IndexType
    State          IndexState         // building | ready | stale | error
}

type IndexState string

const (
    IndexStateBuilding IndexState = "building"
    IndexStateReady    IndexState = "ready"
    IndexStateStale    IndexState = "stale"
    IndexStateError    IndexState = "error"
)
```

#### DuckDB 集成

使用 Go 的 DuckDB 驱动 `github.com/marcboeker/go-duckdb`：

```go
package fulltext

import (
    "database/sql"
    _ "github.com/marcboeker/go-duckdb"
)

type DuckDBEngine struct {
    cacheDir string
    catalog  catalog.Interface
    block    block.Adapter
}

func (e *DuckDBEngine) createIndex(ctx context.Context, dbPath string) (*sql.DB, error) {
    db, err := sql.Open("duckdb", dbPath)
    if err != nil {
        return nil, err
    }

    // 安装并加载 FTS 扩展
    _, err = db.ExecContext(ctx, "INSTALL fts; LOAD fts;")
    if err != nil {
        return nil, fmt.Errorf("load fts extension: %w", err)
    }

    // 创建索引表
    _, err = db.ExecContext(ctx, `
        CREATE TABLE object_index (
            path            VARCHAR PRIMARY KEY,
            size            BIGINT,
            last_modified   TIMESTAMP,
            etag            VARCHAR,
            content_type    VARCHAR,
            metadata_json   VARCHAR,
            content_preview VARCHAR
        )
    `)
    return db, err
}

func (e *DuckDBEngine) buildFTSIndex(ctx context.Context, db *sql.DB) error {
    _, err := db.ExecContext(ctx, `
        PRAGMA create_fts_index(
            'object_index', 'path',
            'path', 'metadata_json', 'content_preview', 'content_type',
            stemmer = 'porter',
            strip_accents = true,
            lower = true
        )
    `)
    return err
}
```

### 3.6 配置项

```yaml
# deploy/config/config.yaml 新增部分

search:
  enabled: true

  # 索引文件本地缓存
  cache:
    dir: "/tmp/lakefs-search-cache"
    max_size_bytes: 1073741824  # 1GB
    ttl: 1h

  fulltext:
    enabled: true
    # 内容索引配置
    content_indexing: true
    content_max_bytes: 65536          # 每文件最大索引 64KB 内容
    content_types:                     # 哪些 MIME 类型索引内容
      - "text/*"
      - "application/json"
      - "application/csv"
      - "application/x-ndjson"
    stemmer: "porter"
    # 索引构建并发
    build_concurrency: 4
    # 内容读取超时
    content_read_timeout: 10s

  vector:
    enabled: false                     # Phase 2 启用
    # 详见 Phase 2 配置
```

对应 Go config struct：

```go
// pkg/config/types.go 新增

type SearchConfig struct {
    Enabled  bool              `mapstructure:"enabled"`
    Cache    SearchCacheConfig `mapstructure:"cache"`
    FullText FullTextConfig    `mapstructure:"fulltext"`
    Vector   VectorConfig      `mapstructure:"vector"`
}

type SearchCacheConfig struct {
    Dir          string        `mapstructure:"dir"`
    MaxSizeBytes int64         `mapstructure:"max_size_bytes"`
    TTL          time.Duration `mapstructure:"ttl"`
}

type FullTextConfig struct {
    Enabled            bool          `mapstructure:"enabled"`
    ContentIndexing    bool          `mapstructure:"content_indexing"`
    ContentMaxBytes    int64         `mapstructure:"content_max_bytes"`
    ContentTypes       []string      `mapstructure:"content_types"`
    Stemmer            string        `mapstructure:"stemmer"`
    BuildConcurrency   int           `mapstructure:"build_concurrency"`
    ContentReadTimeout time.Duration `mapstructure:"content_read_timeout"`
}
```

### 3.7 OpenAPI 定义

在 `api/swagger.yml` 中新增：

```yaml
# 路径定义
/repositories/{repository}/search:
  get:
    tags: ["search"]
    operationId: searchObjects
    summary: Search objects in a repository
    parameters:
      - $ref: "#/components/parameters/RepositoryParam"
      - name: ref
        in: query
        required: true
        schema:
          type: string
      - name: query
        in: query
        required: true
        schema:
          type: string
      - name: scope
        in: query
        schema:
          type: string
          enum: [path, metadata, content, all]
          default: all
      - name: prefix
        in: query
        schema:
          type: string
      - name: content_type
        in: query
        schema:
          type: string
      - name: size_min
        in: query
        schema:
          type: integer
          format: int64
      - name: size_max
        in: query
        schema:
          type: integer
          format: int64
      - name: after
        in: query
        schema:
          type: string
      - name: amount
        in: query
        schema:
          type: integer
          default: 20
          maximum: 100
    responses:
      200:
        description: search results
        content:
          application/json:
            schema:
              $ref: "#/components/schemas/SearchResult"
      404:
        description: index not found for this ref

/repositories/{repository}/search/index:
  post:
    tags: ["search"]
    operationId: buildSearchIndex
    summary: Build or rebuild search index
    parameters:
      - $ref: "#/components/parameters/RepositoryParam"
    requestBody:
      required: true
      content:
        application/json:
          schema:
            $ref: "#/components/schemas/BuildIndexRequest"
    responses:
      202:
        description: index build started
        content:
          application/json:
            schema:
              $ref: "#/components/schemas/BuildIndexResponse"
  get:
    tags: ["search"]
    operationId: getSearchIndexStatus
    summary: Get search index status
    parameters:
      - $ref: "#/components/parameters/RepositoryParam"
      - name: ref
        in: query
        required: true
        schema:
          type: string
    responses:
      200:
        content:
          application/json:
            schema:
              $ref: "#/components/schemas/IndexStatus"
```

### 3.8 实施计划

| 阶段 | 内容 | 工作量 |
|------|------|--------|
| P1-1 | 路径 + 元数据全文搜索 (不含内容索引) | 1.5 周 |
| P1-2 | 文件内容索引 (文本类文件) | 1 周 |
| P1-3 | 增量索引构建 | 0.5 周 |
| P1-4 | Hook 自动触发 + 索引缓存优化 | 0.5 周 |
| **合计** | | **3.5 周** |

详细任务拆分：

```
Phase 1-1 (路径 + 元数据搜索)
├── [ ] pkg/search/ 骨架代码和接口定义
├── [ ] DuckDB Go 驱动集成 (go-duckdb)
├── [ ] 索引构建：遍历 ListEntries → 写入 DuckDB
├── [ ] 索引存储：上传/下载到 _lakefs_search/
├── [ ] 本地缓存管理 (LRU, TTL)
├── [ ] 搜索查询执行 (BM25)
├── [ ] API 端点：POST /search/index, GET /search
├── [ ] swagger.yml 定义 + codegen
├── [ ] Controller 实现
└── [ ] 单元测试 + 集成测试

Phase 1-2 (内容索引)
├── [ ] 内容读取：BlockAdapter.Get + 截断
├── [ ] MIME 类型过滤
├── [ ] 文本提取（JSON/CSV/纯文本）
├── [ ] content_preview 写入索引
├── [ ] 搜索高亮 (snippet 提取)
└── [ ] 测试

Phase 1-3 (增量索引)
├── [ ] Diff 计算：上次索引 commit → 当前 commit
├── [ ] 增量 INSERT/UPDATE/DELETE
├── [ ] FTS 索引重建优化
└── [ ] 测试

Phase 1-4 (自动化)
├── [ ] Hook 模板 YAML
├── [ ] 缓存预热策略
├── [ ] 索引过期清理
└── [ ] 文档
```

---

## 四、Phase 2：向量索引

### 4.1 功能范围

| 搜索类型 | 说明 | 优先级 |
|----------|------|:------:|
| **文本语义搜索** | 自然语言查询 → 匹配相关文件 | P0 |
| **相似文件查找** | 给定一个文件 → 找到内容相似的文件 | P1 |
| **多模态搜索** | 图片 → 找类似图片（需 CLIP 等模型） | P2 |

### 4.2 架构设计

向量搜索比全文搜索复杂，因为需要：
1. **Embedding 生成**：需要 ML 模型（本地或远程 API）
2. **向量存储**：高维浮点数组，存储开销大
3. **近似最近邻 (ANN)**：精确 kNN 太慢，需要 ANN 索引

#### 设计方案对比

| 方案 | 优点 | 缺点 | 推荐度 |
|------|------|------|:------:|
| **A: DuckDB vss 扩展** | 零外部依赖，与 Phase 1 统一技术栈 | 性能有限，10M+ 向量时慢 | ⭐⭐⭐⭐ |
| **B: 内嵌 Hnswlib** | 高性能 HNSW 索引，Go 绑定成熟 | 需要额外依赖，CGO | ⭐⭐⭐ |
| **C: 外部向量数据库** | 性能最好，功能最全 | 增加运维复杂度 | ⭐⭐ |

**推荐方案 A（DuckDB vss）**，理由：
- 与 Phase 1 共用 DuckDB 引擎，降低复杂度
- 对 lakeFS 典型规模（数十万到百万对象）的向量搜索足够
- 无额外运维依赖
- 如果性能不足，可后续迁移到方案 B/C

#### 向量搜索架构

```
┌──────────────────────────────────────────────────────────────┐
│                    向量搜索流程                                │
│                                                              │
│  ┌─────────────┐    ┌──────────────┐    ┌─────────────────┐  │
│  │ 用户查询     │    │ Embedding    │    │ DuckDB vss      │  │
│  │ "财务报表"   │───→│ Generator    │───→│ 向量相似度搜索     │  │
│  └─────────────┘    │              │    │                 │  │
│                      │ 支持:        │    │ cosine_sim()    │  │
│                      │ • 本地模型   │    │ + metadata       │  │
│                      │ • OpenAI API │    │   过滤          │  │
│                      │ • HF API    │    │                  │  │
│                      └──────────────┘    └─────────────────┘  │
│                                                                │
│  索引构建：                                                    │
│  ┌─────────┐    ┌──────────────┐    ┌─────────────────────┐  │
│  │ 对象列表 │───→│ 内容提取      │───→│ Embedding 批量生成 │  │
│  │ (catalog)│    │ (text/image) │    │ + 写入 DuckDB      │  │
│  └─────────┘    └──────────────┘    └─────────────────────┘  │
└──────────────────────────────────────────────────────────────┘
```

### 4.3 Embedding 生成

#### Embedder 接口

```go
package vector

import "context"

// Embedder 生成向量 embedding
type Embedder interface {
    // Embed 为文本生成 embedding
    Embed(ctx context.Context, texts []string) ([][]float32, error)
    // Dimensions 返回 embedding 维度
    Dimensions() int
    // ModelName 返回模型名称
    ModelName() string
}
```

#### 支持的 Embedding 后端

| 后端 | 配置名 | 说明 |
|------|--------|------|
| **OpenAI** | `openai` | text-embedding-3-small/large，需 API key |
| **Hugging Face API** | `huggingface` | Inference API，支持多种模型 |
| **本地 ONNX** | `local` | 内嵌 all-MiniLM-L6-v2 等小模型 |
| **自定义 HTTP** | `custom` | 用户自定义 embedding 服务 |

```go
// OpenAI Embedder 实现
type OpenAIEmbedder struct {
    client     *openai.Client
    model      string
    dimensions int
}

func (e *OpenAIEmbedder) Embed(ctx context.Context, texts []string) ([][]float32, error) {
    resp, err := e.client.CreateEmbeddings(ctx, openai.EmbeddingRequest{
        Model: openai.EmbeddingModel(e.model),
        Input: texts,
    })
    if err != nil {
        return nil, err
    }
    result := make([][]float32, len(resp.Data))
    for i, d := range resp.Data {
        result[i] = d.Embedding
    }
    return result, nil
}

// HTTP Embedder (通用)
type HTTPEmbedder struct {
    endpoint   string
    dimensions int
    client     *http.Client
}
```

### 4.4 向量索引表结构

```sql
-- 安装 vss 扩展
INSTALL vss;
LOAD vss;

-- 向量索引表
CREATE TABLE vector_index (
    path            VARCHAR PRIMARY KEY,
    embedding       FLOAT[384],          -- 维度取决于模型
    content_type    VARCHAR,
    size            BIGINT,
    last_modified   TIMESTAMP,
    metadata_json   VARCHAR,
    content_snippet VARCHAR              -- 用于生成 embedding 的原始文本
);

-- 创建 HNSW 索引 (ANN)
CREATE INDEX vec_hnsw_idx ON vector_index
USING HNSW (embedding)
WITH (metric = 'cosine');
```

### 4.5 向量搜索 API

#### 文本查询搜索

```
POST /api/v1/repositories/{repository}/search/vector
{
  "ref": "main",
  "query": "财务报表季度收入",
  "top_k": 20,
  "min_score": 0.7,
  "filters": {
    "prefix": "data/finance/",
    "content_type": "text/*",
    "metadata": {
      "department": "finance"
    }
  }
}
```

响应：

```json
{
  "results": [
    {
      "path": "data/finance/q4_2025_revenue.csv",
      "score": 0.945,
      "size": 2048576,
      "content_type": "text/csv",
      "metadata": {"department": "finance", "quarter": "Q4"},
      "snippet": "季度收入汇总表，包含各产品线收入..."
    }
  ],
  "pagination": {
    "has_more": false,
    "results": 15
  },
  "index_info": {
    "commit_id": "def789",
    "model": "text-embedding-3-small",
    "dimensions": 384,
    "total_vectors": 42000
  }
}
```

#### 相似文件搜索

```
POST /api/v1/repositories/{repository}/search/vector/similar
{
  "ref": "main",
  "source_path": "data/finance/q3_2025_revenue.csv",
  "top_k": 10,
  "min_score": 0.8
}
```

### 4.6 索引构建 API

```
POST /api/v1/repositories/{repository}/search/index
{
  "ref": "main",
  "index_type": "vector",
  "config": {
    "embedder": "openai",
    "model": "text-embedding-3-small",
    "content_types": ["text/*", "application/json"],
    "content_max_bytes": 8192,
    "batch_size": 100
  }
}
```

### 4.7 配置项

```yaml
search:
  vector:
    enabled: true

    # Embedding 后端配置
    embedder:
      type: "openai"            # openai | huggingface | local | custom

      openai:
        api_key: "${OPENAI_API_KEY}"
        model: "text-embedding-3-small"
        dimensions: 384
        max_tokens: 8191
        batch_size: 100
        rate_limit_rpm: 3000

      huggingface:
        api_key: "${HF_API_KEY}"
        model: "sentence-transformers/all-MiniLM-L6-v2"
        endpoint: "https://api-inference.huggingface.co"

      local:
        model_path: "/models/all-MiniLM-L6-v2.onnx"
        dimensions: 384

      custom:
        endpoint: "http://embedding-service:8080/embed"
        dimensions: 768
        batch_size: 64
        timeout: 30s

    # 内容提取
    content_types:
      - "text/*"
      - "application/json"
      - "application/csv"
    content_max_bytes: 8192

    # 索引构建
    build_concurrency: 2
    build_batch_size: 100
```

对应 Go config struct：

```go
type VectorConfig struct {
    Enabled          bool           `mapstructure:"enabled"`
    Embedder         EmbedderConfig `mapstructure:"embedder"`
    ContentTypes     []string       `mapstructure:"content_types"`
    ContentMaxBytes  int64          `mapstructure:"content_max_bytes"`
    BuildConcurrency int            `mapstructure:"build_concurrency"`
    BuildBatchSize   int            `mapstructure:"build_batch_size"`
}

type EmbedderConfig struct {
    Type        string            `mapstructure:"type"`
    OpenAI      OpenAIConfig      `mapstructure:"openai"`
    HuggingFace HuggingFaceConfig `mapstructure:"huggingface"`
    Local       LocalConfig       `mapstructure:"local"`
    Custom      CustomConfig      `mapstructure:"custom"`
}

type OpenAIConfig struct {
    APIKey     string `mapstructure:"api_key"`
    Model      string `mapstructure:"model"`
    Dimensions int    `mapstructure:"dimensions"`
    MaxTokens  int    `mapstructure:"max_tokens"`
    BatchSize  int    `mapstructure:"batch_size"`
    RateLimitRPM int  `mapstructure:"rate_limit_rpm"`
}
```

### 4.8 实施计划

| 阶段 | 内容 | 工作量 |
|------|------|--------|
| P2-1 | Embedder 接口 + OpenAI/HF 实现 | 1 周 |
| P2-2 | DuckDB vss 集成 + 向量索引构建 | 1 周 |
| P2-3 | 向量搜索 API + 查询执行 | 1 周 |
| P2-4 | 相似文件搜索 + 增量更新 | 0.5 周 |
| P2-5 | 本地 ONNX Embedder (可选) | 1 周 |
| **合计** | | **4.5 周** |

---

## 五、Python SDK 扩展

Phase 1 和 Phase 2 完成后，需要更新 Python 客户端。

### 5.1 自动生成

搜索 API 在 `swagger.yml` 中定义后，通过 `make client-python` 自动生成 SDK 代码。
生成的类包括：
- `SearchApi` - 搜索操作
- `SearchResult`, `SearchHit` - 响应模型
- `BuildIndexRequest`, `IndexStatus` - 索引管理模型

### 5.2 高层封装

在 `clients/python-wrapper/` 中添加便捷封装：

```python
# clients/python-wrapper/lakefs/search.py

class SearchIndex:
    """lakeFS 搜索索引管理"""

    def __init__(self, repository: Repository):
        self._repo = repository
        self._client = repository._client

    def build(
        self,
        ref: str = "main",
        index_type: str = "fulltext",
        include_content: bool = True,
        content_types: list[str] | None = None,
        mode: str = "incremental",
    ) -> str:
        """构建搜索索引，返回 task_id"""
        ...

    def status(self, ref: str = "main") -> dict:
        """获取索引状态"""
        ...

    def search(
        self,
        query: str,
        ref: str = "main",
        scope: str = "all",
        prefix: str | None = None,
        content_type: str | None = None,
        limit: int = 20,
    ) -> list[dict]:
        """全文搜索"""
        ...

    def vector_search(
        self,
        query: str,
        ref: str = "main",
        top_k: int = 20,
        min_score: float = 0.0,
        filters: dict | None = None,
    ) -> list[dict]:
        """向量语义搜索"""
        ...

    def find_similar(
        self,
        source_path: str,
        ref: str = "main",
        top_k: int = 10,
    ) -> list[dict]:
        """查找相似文件"""
        ...


# 使用示例
repo = lakefs.Repository("my-repo")

# 构建索引
repo.search.build(ref="main", index_type="fulltext")

# 全文搜索
results = repo.search.search("user_id=12345", scope="content")

# 向量搜索
results = repo.search.vector_search("季度财务报表")

# 相似文件
similar = repo.search.find_similar("data/q3_report.csv")
```

---

## 六、lakectl CLI 扩展

### 6.1 新增命令

```bash
# 构建全文索引
lakectl search index build lakefs://my-repo/main --type fulltext --include-content

# 构建向量索引
lakectl search index build lakefs://my-repo/main --type vector --embedder openai

# 查看索引状态
lakectl search index status lakefs://my-repo/main

# 全文搜索
lakectl search query lakefs://my-repo/main "user_id=12345" --scope content

# 向量搜索
lakectl search vector lakefs://my-repo/main "财务报表" --top-k 20

# 查找相似文件
lakectl search similar lakefs://my-repo/main data/report.csv --top-k 10
```

---

## 七、版本化索引策略

### 7.1 核心问题

lakeFS 的核心是版本控制，搜索索引必须与版本语义兼容。

### 7.2 策略选择

**采用"最近祖先索引"策略：**

```
commit A (有索引) → commit B → commit C (查询)
                                    ↓
                               使用 commit A 的索引
                               + 实时 diff 补偿
```

规则：
1. 索引只在显式触发时构建（不自动为每个 commit 构建）
2. 查询时，找到 ref 对应 commit 链上最近的已索引 commit
3. 如果 diff 较小（< 阈值），实时补偿差异
4. 如果 diff 较大，提示用户重建索引

### 7.3 分支隔离

```
main:    C1(indexed) → C2 → C3(indexed)
              \
feature:       C4 → C5 → C6 (查询)
               ↓
           使用 C1 的索引 + diff(C1, C6) 补偿
```

- 每个分支独立查找其祖先链上的最近索引
- 分支之间不共享索引（避免合并冲突）
- merge 后的新 commit 可触发新索引构建

### 7.4 索引生命周期

```
创建 → 就绪 → 过时(stale) → 清理(GC)
               ↑
           新 commit 产生
```

- 索引文件存储在 `_lakefs_search/` 下，不受 lakeFS GC 管理
- 需要独立的索引清理机制（删除过旧的索引快照）
- 保留策略：每个分支保留最近 N 个索引快照

---

## 八、性能预估

### 8.1 全文搜索

| 对象数量 | 索引构建时间 | 索引大小 | 搜索延迟 (P95) |
|----------|-------------|---------|----------------|
| 10K | ~5s | ~5MB | <50ms |
| 100K | ~30s | ~50MB | <100ms |
| 1M | ~5min | ~500MB | <200ms |
| 10M | ~50min | ~5GB | <500ms |

(含内容索引时，构建时间取决于对象存储读取速度)

### 8.2 向量搜索

| 向量数量 | 维度 | 索引构建 | 索引大小 | 搜索延迟 |
|----------|------|---------|---------|---------|
| 10K | 384 | ~2min (含 embedding) | ~15MB | <50ms |
| 100K | 384 | ~20min | ~150MB | <100ms |
| 1M | 384 | ~3h | ~1.5GB | <300ms |

(embedding 生成是瓶颈，OpenAI API ~3000 RPM)

### 8.3 瓶颈与优化

| 瓶颈 | 优化策略 |
|------|---------|
| 对象存储读取 | 并发读取 + 只读 content_preview 部分 |
| Embedding API 调用 | 批量请求 + 速率控制 + 本地缓存 |
| DuckDB 文件下载 | 本地 LRU 缓存 + 预热 |
| 大索引搜索 | DuckDB HNSW 近似搜索 + 分区 |

---

## 九、总体里程碑

```
Phase 0: 基础设施                         1 周
├── pkg/search/ 包结构
├── DuckDB Go 驱动集成
├── 索引存储抽象
├── 配置系统集成
└── 基础测试框架

Phase 1: 全文搜索                         3.5 周
├── P1-1: 路径 + 元数据搜索              1.5 周
├── P1-2: 内容索引                       1 周
├── P1-3: 增量构建                       0.5 周
└── P1-4: 自动化 + 缓存优化             0.5 周

Phase 2: 向量搜索                         4.5 周
├── P2-1: Embedder 接口                  1 周
├── P2-2: DuckDB vss 索引               1 周
├── P2-3: 搜索 API                       1 周
├── P2-4: 相似搜索 + 增量更新           0.5 周
└── P2-5: 本地 ONNX (可选)              1 周

Phase 3: 客户端与 CLI                     1.5 周
├── Python SDK 封装
├── lakectl 命令
└── 文档

──────────────────────────────────────────────
总计                                      ~10.5 周
```

---

## 十、风险与缓解

| 风险 | 影响 | 缓解措施 |
|------|------|---------|
| DuckDB Go 驱动 CGO 兼容性 | 编译/部署 | 使用 Docker 多阶段构建；备选纯 Go 方案 (Bluge) |
| 大仓库索引构建超时 | 可用性 | 异步 Task + 进度上报 + 超时控制 |
| Embedding API 成本 | 运营 | 增量更新 + 本地缓存 + 可选本地模型 |
| 索引存储膨胀 | 成本 | 保留策略 + 索引 GC + 压缩 |
| DuckDB vss 性能瓶颈 | 大规模场景 | 预留迁移到 Hnswlib/外部向量库的接口 |
| 版本化索引一致性 | 正确性 | 最近祖先策略 + diff 补偿 + 明确的过时提示 |

---

## 附录 A：替代方案评估

### A.1 为什么不用 Elasticsearch

| 考虑 | 分析 |
|------|------|
| 运维复杂度 | ES 需要独立集群，与 lakeFS "单二进制" 哲学冲突 |
| 版本化 | ES 不天然支持数据版本化，需要额外索引管理 |
| 成本 | ES 集群成本高，对小规模部署不友好 |
| 结论 | 不推荐 |

### A.2 为什么不用 SQLite FTS5

| 考虑 | 分析 |
|------|------|
| 性能 | FTS5 在大数据集上比 DuckDB FTS 慢（列式 vs 行式）|
| 向量支持 | SQLite 无原生向量搜索（需 sqlite-vss 扩展，不够成熟）|
| 分析能力 | DuckDB 支持更丰富的 SQL 分析（聚合、窗口函数等）|
| 结论 | DuckDB 更适合 |

### A.3 为什么不用 Bleve/Bluge (纯 Go)

| 考虑 | 分析 |
|------|------|
| 优点 | 纯 Go，无 CGO 依赖 |
| 缺点 | 无向量搜索，无法与 Phase 2 统一；SQL 查询能力弱 |
| 结论 | 如果 CGO 是硬性障碍可作为 Phase 1 备选 |

---

## 附录 B：与现有功能的兼容性

### B.1 与 Metadata Search（企业版）的关系

本方案是**开源版**的搜索能力，与企业版 Metadata Search 互补：

| 维度 | 企业版 Metadata Search | 本方案 |
|------|----------------------|--------|
| 搜索目标 | 系统/用户元数据 | 路径 + 元数据 + 文件内容 + 语义 |
| 实现方式 | Iceberg 表 + 外部引擎 | DuckDB 进程内 |
| 外部依赖 | 需要 DuckDB/Spark/Trino | 无外部依赖 |
| 版本语义 | 通过 Iceberg 快照 | 通过索引快照 + diff 补偿 |
| 适用场景 | 大规模数据治理 | 开发者快速搜索 |

### B.2 与 Actions/Hooks 的集成

搜索索引可通过现有 hook 机制自动化，不需要修改 hook 引擎。

### B.3 与 S3 Gateway 的关系

搜索不影响 S3 Gateway。搜索功能仅通过 OpenAPI 端点暴露。
