# WeKnora LLM Wiki 详细实现细节（移植参考手册）

> 本文是《LLM Wiki知识库生成与使用逻辑.md》（架构概述）的**配套实现手册**，聚焦「可直接照着写的细节」：完整字段、SQL DDL、函数签名、常量值、算法与并发控制。
>
> 代码基准：`internal/`（Go 后端，GORM + Postgres/SQLite 双方言）。核心实现约 15K 行（不含测试与飞书连接器）。
>
> 阅读建议：如果你要移植，先读架构文档建立心智模型，再用本文按模块落地；每个模块末尾都有「可移植要点」清单。

---

## 目录

1. [数据模型与持久层](#1-数据模型与持久层)
2. [入队、认领与调度编排](#2-入队认领与调度编排)
3. [Map/Reduce 主流程](#3-mapreduce-主流程)
4. [抽取子管线](#4-抽取子管线cite--dedup--taxonomy--slug-handles)
5. [页面 CRUD 与链接维护](#5-页面-crud-与链接维护)
6. [在线消费：Agent 工具 / 重排 / 权限 / 句柄](#6-在线消费agent-工具--重排--权限--句柄)
7. [Prompt 模板清单](#7-prompt-模板清单)
8. [移植路线图：从最小可行版到完整版](#8-移植路线图从最小可行版到完整版)
9. [关键文件索引](#9-关键文件索引)

---

## 1. 数据模型与持久层

### 1.1 类型别名

三个自定义 JSON 类型，实现 `driver.Valuer` / `sql.Scanner` 直接映射 JSON 列：

```go
// internal/types/session.go
type StringArray []string   // 存 JSON 数组，如 ["a","b"]

// internal/types/json.go
type JSON json.RawMessage    // 存自由 JSON 对象

// internal/types/json_map.go
type JSONMap map[string]any  // nil -> SQL NULL（而非 "null"）
```

### 1.2 枚举（全部取值，注意区分字符串值）

```go
// WikiPageType
const (
    WikiPageTypeSummary    = "summary"    // 文档摘要页
    WikiPageTypeEntity     = "entity"     // 实体页
    WikiPageTypeConcept    = "concept"    // 概念页
    WikiPageTypeIndex      = "index"      // 索引页
    WikiPageTypeSynthesis  = "synthesis"  // Agent 综合页（非 ingest 创建）
    WikiPageTypeComparison = "comparison" // Agent 对比页（非 ingest 创建）
)

// WikiPageStatus
const (
    WikiPageStatusDraft     = "draft"
    WikiPageStatusPublished = "published"
    WikiPageStatusArchived  = "archived"
)

// WikiEditSource（版本作者类型；空值归一化为 pipeline）
const (
    WikiEditSourcePipeline = "pipeline"
    WikiEditSourceAgent    = "agent"
    WikiEditSourceUser     = "user"
    WikiEditSourceRevert   = "revert"
)

// WikiExtractionGranularity（抽取粒度）
const (
    WikiExtractionFocused    = "focused"
    WikiExtractionStandard   = "standard"   // 默认
    WikiExtractionExhaustive = "exhaustive"
)
```

### 1.3 WikiPage 结构体（逐字段）

```go
type WikiPage struct {
    ID              string         `gorm:"type:varchar(36);primaryKey"`
    TenantID        uint64         `gorm:"index"`
    KnowledgeBaseID string         `gorm:"type:varchar(36);index"`
    // KB 内唯一的 URL 友好 slug，如 "entity/acme-corp"、"summary/<uuid>"
    Slug            string         `gorm:"type:varchar(255);uniqueIndex:idx_kb_slug"`
    Title           string         `gorm:"type:varchar(512)"`
    PageType        string         `gorm:"type:varchar(32);index"`
    Status          string         `gorm:"type:varchar(32);default:'published'"`
    Content         string         `gorm:"type:text"`   // 完整 Markdown（含 [[slug]] 链接）
    Summary         string         `gorm:"type:text"`   // 单行摘要（索引列表用）
    Aliases         StringArray    `gorm:"type:json"`   // 别名/缩写/翻译名
    ParentSlug      string         `gorm:"type:varchar(255);index"` // 语义父页（≠ 目录）
    // folder_id 是目录定位唯一真源；CategoryPath/WikiPath/Depth 是反规范化缓存，每次写重算
    FolderID        string         `gorm:"column:folder_id;type:varchar(36);index;default:''"`
    CategoryPath    StringArray    `gorm:"type:json"`   // 面包屑，如 ["AI","LLM应用","RAG"]
    WikiPath        string         `gorm:"type:varchar(1024);index"` // 派生可排序路径
    Depth           int            `gorm:"default:0;index"`  // = len(CategoryPath)
    SortOrder       int            `gorm:"default:0;index"`
    // 来源知识，格式 "knowledge_id|doc_title"，用 "|" 切分（旧格式是裸 uuid）
    SourceRefs      StringArray    `gorm:"type:json"`
    ChunkRefs       StringArray    `gorm:"type:json"`   // 逐 chunk UUID（引用根基）
    InLinks         StringArray    `gorm:"type:json"`   // 指向本页的 slug
    OutLinks        StringArray    `gorm:"type:json"`   // 本页指向的 slug（Content 派生）
    PageMetadata    JSON           `gorm:"column:page_metadata;type:json"` // 自由 JSON
    // 乐观锁 + 用户可见修订号（仅真实内容变化才 bump）
    Version         int            `gorm:"default:1"`
    LastEditSource  string         `gorm:"type:varchar(16);default:''"`
    LastEditorID    string         `gorm:"type:varchar(64);default:''"`
    CreatedAt       time.Time
    UpdatedAt       time.Time
    DeletedAt       gorm.DeletedAt `gorm:"index"`       // 软删除
}
```

### 1.4 WikiFolder / WikiPageRevision

```go
type WikiFolder struct {
    ID              string `gorm:"type:varchar(36);primaryKey"`
    TenantID        uint64 `gorm:"index"`
    KnowledgeBaseID string `gorm:"type:varchar(36);index"`
    ParentID        string `gorm:"column:parent_id;type:varchar(36);index;default:''"` // 邻接表，""=根
    Name            string `gorm:"type:varchar(255)"`
    Path            string `gorm:"type:varchar(1024)"` // 物化 "/" 连接路径
    Depth           int    `gorm:"default:0"`
    SortOrder       int    `gorm:"default:0"`
    CreatedAt       time.Time
    UpdatedAt       time.Time
    DeletedAt       gorm.DeletedAt `gorm:"index"`
}

type WikiPageRevision struct {  // 不可变快照
    ID              string      `gorm:"type:varchar(36);primaryKey"`
    TenantID        uint64      `gorm:"index"`
    KnowledgeBaseID string      `gorm:"type:varchar(36);index:idx_wiki_page_revisions_kb_slug"`
    PageID          string      `gorm:"type:varchar(36);uniqueIndex:idx_wiki_page_revisions_page_version"`
    Slug            string      `gorm:"type:varchar(255);index:idx_wiki_page_revisions_kb_slug"`
    Version         int         `gorm:"uniqueIndex:idx_wiki_page_revisions_page_version"`
    Title           string      `gorm:"type:varchar(512)"`
    PageType        string      `gorm:"type:varchar(32)"`
    Status          string      `gorm:"type:varchar(32)"`
    Content         string      `gorm:"type:text"`
    Summary         string      `gorm:"type:text"`
    Aliases         StringArray `gorm:"type:json"`
    EditSource      string      `gorm:"type:varchar(16);default:''"` // 该版本作者
    EditorID        string      `gorm:"type:varchar(64);default:''"`
    EditedAt        time.Time   // 该版本作为当前版本时的 updated_at
    CreatedAt       time.Time
}
```

**关键语义**：当前版本只存在于 `wiki_pages`，不进快照表；每次编辑替换版本 V 时，把编辑前状态以 `(page_id, V)` 在同一事务插入快照表。

快照保留策略常量：

```go
const (
    WikiMaxRevisionsPerPage = 50   // 软上限（机器作者快照）
    WikiMaxRevisionsHardCap = 200  // 硬上限（所有来源）
)
var WikiPrunableEditSources = []string{"", WikiEditSourcePipeline} // "" + "pipeline" 可被软上限删
```

### 1.5 WikiConfig

```go
type WikiConfig struct {
    SynthesisModelID        string                    // 页面生成 LLM 模型
    MaxPagesPerIngest       int                       // 每次 ingest 页数上限，0=无限
    ExtractionGranularity   WikiExtractionGranularity // focused/standard/exhaustive
    ContentInstructions     string                    // 正文补充指令
    ExtractionInstructions  string                    // 抽取补充指令
    IngestBatchSize       int // 单批领取 op 数，0->5
    IngestMapParallel     int // Map 并发，0->10
    IngestReduceParallel  int // Reduce 并发，0->10
    IngestMaxInflight     int // 每 KB 在途批数上限，0->4
}
// 每个字段都有 OrDefault(fallback) 方法；WikiConfig 实现 driver.Valuer/sql.Scanner（存 JSONB）
```

是否启用 wiki 由 `knowledge_bases.indexing_strategy.wiki_enabled` 控制，`WikiConfig` 只承载参数。

### 1.6 表结构 DDL（生产以 migration 为准）

```sql
-- wiki_pages
CREATE TABLE wiki_pages (
    id                VARCHAR(36) PRIMARY KEY,
    tenant_id         BIGINT NOT NULL,
    knowledge_base_id VARCHAR(36) NOT NULL,
    slug              VARCHAR(255) NOT NULL,
    title             VARCHAR(512) NOT NULL DEFAULT '',
    page_type         VARCHAR(32) NOT NULL DEFAULT 'summary',
    status            VARCHAR(32) NOT NULL DEFAULT 'published',
    content           TEXT NOT NULL DEFAULT '',
    summary           TEXT NOT NULL DEFAULT '',
    parent_slug       VARCHAR(255) NOT NULL DEFAULT '',
    folder_id         VARCHAR(36) NOT NULL DEFAULT '',
    category_path     JSONB DEFAULT '[]'::JSONB,
    wiki_path         VARCHAR(1024) NOT NULL DEFAULT '',
    depth             INT NOT NULL DEFAULT 0,
    sort_order        INT NOT NULL DEFAULT 0,
    source_refs       JSONB DEFAULT '[]'::JSONB,
    chunk_refs        JSONB DEFAULT '[]'::JSONB,
    in_links          JSONB DEFAULT '[]'::JSONB,
    out_links         JSONB DEFAULT '[]'::JSONB,
    page_metadata     JSONB DEFAULT '{}'::JSONB,
    aliases           JSONB DEFAULT '[]'::JSONB,
    version           INT NOT NULL DEFAULT 1,
    last_edit_source  VARCHAR(16) NOT NULL DEFAULT '',
    last_editor_id    VARCHAR(64) NOT NULL DEFAULT '',
    created_at        TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at        TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    deleted_at        TIMESTAMPTZ
);

-- 关键索引（注意：slug 用「部分唯一索引」而非普通 uniqueIndex）
CREATE UNIQUE INDEX idx_wiki_pages_kb_slug ON wiki_pages (knowledge_base_id, slug) WHERE deleted_at IS NULL;
CREATE INDEX idx_wiki_pages_kb_id        ON wiki_pages (knowledge_base_id);
CREATE INDEX idx_wiki_pages_page_type    ON wiki_pages (knowledge_base_id, page_type);
CREATE INDEX idx_wiki_pages_parent_slug  ON wiki_pages (knowledge_base_id, parent_slug);
CREATE INDEX idx_wiki_pages_tree         ON wiki_pages (knowledge_base_id, page_type, wiki_path, sort_order, title);
CREATE INDEX idx_wiki_pages_folder       ON wiki_pages (knowledge_base_id, folder_id);
CREATE INDEX idx_wiki_pages_tenant_id    ON wiki_pages (tenant_id);
CREATE INDEX idx_wiki_pages_deleted_at   ON wiki_pages (deleted_at);
CREATE INDEX idx_wiki_pages_fulltext     ON wiki_pages USING GIN (to_tsvector('simple', coalesce(title,'') || ' ' || coalesce(content,'')));
CREATE INDEX idx_wiki_pages_source_refs      ON wiki_pages USING GIN (source_refs jsonb_path_ops);
CREATE INDEX idx_wiki_pages_source_refs_text ON wiki_pages USING GIN (to_tsvector('simple', source_refs::text));
CREATE INDEX idx_wiki_pages_title_trgm       ON wiki_pages USING GIN (lower(title) gin_trgm_ops);
```

```sql
-- wiki_folders
CREATE TABLE wiki_folders (
    id                VARCHAR(36) PRIMARY KEY,
    tenant_id         BIGINT NOT NULL DEFAULT 0,
    knowledge_base_id VARCHAR(36) NOT NULL,
    parent_id         VARCHAR(36) NOT NULL DEFAULT '',
    name              VARCHAR(255) NOT NULL,
    path              VARCHAR(1024) NOT NULL DEFAULT '',
    depth             INT NOT NULL DEFAULT 0,
    sort_order        INT NOT NULL DEFAULT 0,
    created_at        TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at        TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    deleted_at        TIMESTAMPTZ
);
CREATE UNIQUE INDEX idx_wiki_folders_parent_name ON wiki_folders (knowledge_base_id, parent_id, name) WHERE deleted_at IS NULL;
CREATE INDEX idx_wiki_folders_parent ON wiki_folders (knowledge_base_id, parent_id);
```

```sql
-- wiki_page_revisions
CREATE TABLE wiki_page_revisions (
    id                VARCHAR(36) PRIMARY KEY,
    tenant_id         BIGINT NOT NULL,
    knowledge_base_id VARCHAR(36) NOT NULL,
    page_id           VARCHAR(36) NOT NULL,
    slug              VARCHAR(255) NOT NULL,
    version           INT NOT NULL,
    title             VARCHAR(512) NOT NULL DEFAULT '',
    page_type         VARCHAR(32) NOT NULL DEFAULT 'summary',
    status            VARCHAR(32) NOT NULL DEFAULT 'published',
    content           TEXT NOT NULL DEFAULT '',
    summary           TEXT NOT NULL DEFAULT '',
    aliases           JSONB DEFAULT '[]'::JSONB,
    edit_source       VARCHAR(16) NOT NULL DEFAULT '',
    editor_id         VARCHAR(64) NOT NULL DEFAULT '',
    edited_at         TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    created_at        TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE UNIQUE INDEX idx_wiki_page_revisions_page_version ON wiki_page_revisions (page_id, version);
CREATE INDEX idx_wiki_page_revisions_kb_slug ON wiki_page_revisions (knowledge_base_id, slug);
```

### 1.7 任务持久化队列

```go
type TaskPendingOp struct {
    ID         int64           `gorm:"primaryKey;autoIncrement"`
    TenantID   uint64          `gorm:"index"`
    TaskType   string          `gorm:"type:varchar(64)"`  // "wiki:ingest" / "wiki:finalize"
    Scope      string          `gorm:"type:varchar(32)"`  // "knowledge_base"
    ScopeID    string          `gorm:"type:varchar(64)"`  // kbID
    Op         string          `gorm:"type:varchar(32)"`  // "ingest" / "retract"
    DedupKey   string          `gorm:"type:varchar(128);default:''"` // 服务自定义去重键（= knowledgeID）
    Payload    json.RawMessage `gorm:"type:jsonb;default:'{}'"`
    FailCount  int             `gorm:"default:0"`
    EnqueuedAt time.Time
    ClaimedAt  *time.Time      // 领取时间戳
}

type TaskDeadLetter struct {
    ID        int64           `gorm:"primaryKey;autoIncrement"`
    TenantID  uint64          `gorm:"index"`
    TaskType  string          `gorm:"type:varchar(64)"`
    Scope     string          `gorm:"type:varchar(32)"`
    ScopeID   string          `gorm:"type:varchar(64)"`
    RelatedID string          `gorm:"type:varchar(64);default:''"` // wiki ingest 放 knowledge_id
    Payload   json.RawMessage `gorm:"type:jsonb"`
    LastError string          `gorm:"type:text;default:''"`
    FailCount int
    FailedAt  time.Time
}
```

```sql
CREATE TABLE task_pending_ops (
    id          BIGSERIAL PRIMARY KEY,
    tenant_id   BIGINT NOT NULL,
    task_type   VARCHAR(64) NOT NULL,
    scope       VARCHAR(32) NOT NULL,
    scope_id    VARCHAR(64) NOT NULL,
    op          VARCHAR(32) NOT NULL,
    dedup_key   VARCHAR(128) NOT NULL DEFAULT '',
    payload     JSONB NOT NULL DEFAULT '{}'::JSONB,
    fail_count  INT NOT NULL DEFAULT 0,
    enqueued_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    claimed_at  TIMESTAMPTZ
);
CREATE INDEX idx_task_pending_ops_scope ON task_pending_ops (task_type, scope, scope_id, id); -- PeekBatch 游标
CREATE INDEX idx_task_pending_ops_tenant ON task_pending_ops (tenant_id);
```

**核心语义**：队列身份 = `(TaskType, Scope, ScopeID)`；**不强制唯一**（同 DedupKey 多行可共存，消费端「最后写入胜出」折叠）；`FailCount` 由 `IncrFailCount` 原子自增，超限进死信。

### 1.8 路径清洗（移植必须复刻，保证存储与查询一致）

```go
const WikiCategoryMaxDepth = 3  // prompt 只求 ≤2 层，存储层留 1 层防御

// 全角分隔符归一化
var wikiCategorySeparatorReplacer = strings.NewReplacer("／", "/", "｜", "/", "|", "/")

func CleanWikiCategoryPart(part string) []string   // 去空格、替换分隔符、按"/"切、去引号括号、剔除类型标签
func CleanWikiCategoryPath(parts []string) []string // 全路径清洗+去重+截断到 3 层
func SplitWikiPageTypes(raw string) []string        // "entity,concept" -> ["entity","concept"]
func isWikiTypeCategoryLabel(label string) bool     // 归一化匹配 entity/实体/concept/概念/summary/摘要/wiki/页面
func WikiSourceKnowledgeID(ref string) string       // 从 "uuid|title" 提取 uuid（按第一个 | 切）
```

---

## 2. 入队、认领与调度编排

### 2.1 完整常量表

| 常量 | 值 | 用途 |
| --- | --- | --- |
| `maxContentForWiki` | 32768 字节 | 送 LLM 的文档内容上限 |
| `wikiClaimStaleAfter` | 90m | 僵尸 claim 判定阈值（> asynq Timeout 60m） |
| `wikiSlugLockTTL` | 5m | per-slug 锁 TTL |
| `wikiSlugLockWait` | 2m | reduce 抢锁最长等待 |
| `wikiSlugLockPoll` | 50ms | 抢锁轮询 |
| `wikiInflightDefault` | 4 | 每 KB 在途批次数默认 |
| `wikiInflightTTL` | 90s | 槽位无续期存活 |
| `wikiInflightRenew` | 30s | 槽位续期周期 |
| `wikiInflightBackoff` | 10s | 被上限拒绝后的重触发延迟 |
| `wikiIngestDelay` | 30s | 上传后去抖窗口 |
| `wikiFollowUpDelay` | 5s | 正常 follow-up 延迟 |
| `wikiRateLimitBackoff` | 60s | 429 时 follow-up 退避 |
| `wikiMaxDocsPerBatch` | 5 | 单批最大文档数 |
| `wikiMaxFailRetries` | 5 | 单文档 op 重试上限 |
| `wikiIngestMaxRetry` | 10 | asynq 层 MaxRetry |
| `wikiDeletedTTL` | 1h | 删除墓碑存活（> ingestDelay） |
| `wikiLLMMaxAttempts` | 3 | 每次 LLM 调用尝试数 |
| `wikiLLMMaxTokens` | 32768 | completion token 预算 |
| `wikiLLMBackoffBase` | 2s | LLM 重试指数退避基数 |
| `wikiFinalizeDelay` | 20s | finalize 去抖延迟 |
| `wikiFinalizeMaxRows` | 5000 | 单次 finalize 排空行数上限 |
| `wikiFinalizeLockTTL` | 60s | finalize 锁 TTL |
| `wikiFinalizeLockRenew` | 20s | finalize 锁续期 |

类型常量：`wikiTaskType = "wiki:ingest"`、`wikiFinalizeTaskType = "wiki:finalize"`、`WikiOpIngest = "ingest"`、`WikiOpRetract = "retract"`、`TypeWikiIngest = "wiki:ingest"`、`TypeWikiFinalize = "wiki:finalize"`、`QueueWiki = "wiki"`、`DefaultWikiWorkerConcurrency = 8`。

### 2.2 入队流程

```go
func EnqueueWikiIngest(ctx, task, pendingRepo, tenantID, kbID, knowledgeID string) (bool, error)
```

1. `newWikiIngestPendingOp`：`TaskType="wiki:ingest"`、`Scope="knowledge_base"`、`ScopeID=kbID`、`Op="ingest"`、`DedupKey=knowledgeID`、`Payload=WikiPendingOp{Op, KnowledgeID, Language}`。
2. `enqueueWikiPendingOp`：若 repo 实现了 `TaskPendingOpsKnowledgeBaseGuard`，走 `EnqueueIfKnowledgeBaseActive`（**Postgres 上 SHARE 锁串行 check+insert**，防 KB 已删仍入队）；否则普通 `Enqueue`。
3. `enqueueWikiIngestTrigger`：asynq 任务 `Queue("wiki")`、`MaxRetry(10)`、`Timeout(60m)`、`ProcessIn(30s)`。**普通 ingest trigger 不带 TaskID**，靠 30s 窗口内「先到者排空，后到者空队列退出」自然合并。

`EnqueueWikiRetract` 类似，但 `ProcessIn(5s)`（删除是单次事件无需去抖），payload 额外含 `DocTitle/DocSummary/PageSlugs/FolderIDs`。

**TaskID 去抖（asynq.TaskID 按 KB 合并，与 dedup_key 是两套机制）**：

| 场景 | TaskID | ProcessIn |
| --- | --- | --- |
| finalize 去抖 | `"wiki-finalize-"+kbID` | 20s |
| 被 in-flight 上限拒绝 | `"wiki-ingest-capped-"+kbID` | 10s |
| stale claim 安全网 | `"wiki-ingest-recheck-"+kbID` | 90m5s |

冲突统一处理：`ErrTaskIDConflict` / `ErrDuplicateTask` → 静默返回（已合并）。

### 2.3 finalizing 占位（防「处理完成」提前置位）

`willSpawnWiki := kb.IndexingStrategy.WikiEnabled && len(textChunks) > 0`。wiki 占一个 subtask 槽位。若 `willSpawnWiki`，走 `SeedKnowledgeFinalizingWithPendingOp`，**在一个事务里**：
1. `SELECT id FROM knowledge_bases ... FOR SHARE` 锁 KB 行；
2. `UPDATE knowledge SET parse_status='finalizing', pending_subtasks_count=expected WHERE parse_status='processing'`；
3. `INSERT task_pending_ops` 行。

三者原子，关闭「先入 finalizing 后丢 op」的崩溃窗口。释放用 `FinalizeSubtask`（原子减计数 + `parse_status='finalizing' AND pending=0` 条件 promote，**无前置 SELECT**，规避读写副本延迟）。

### 2.4 认领（claim）—— 并发安全核心

`ClaimBatch(ctx, taskType, scope, kbID, limit, staleBefore)` 在一个事务内三步（Postgres）：

**步骤 1**：`FOR UPDATE SKIP LOCKED` 锁每个 dedup_key 的锚点行，映射回 key，`NOT IN` 剔除仍持 fresh claim 的 key：

```sql
SELECT dedup_key FROM task_pending_ops
WHERE id IN (
    SELECT id FROM (
        SELECT id, ROW_NUMBER() OVER (PARTITION BY dedup_key ORDER BY id) AS rn
        FROM task_pending_ops
        WHERE task_type=? AND scope=? AND scope_id=?
          AND (claimed_at IS NULL OR claimed_at < ?)
          AND dedup_key NOT IN (
            SELECT dedup_key FROM task_pending_ops
            WHERE task_type=? AND scope=? AND scope_id=?
              AND claimed_at IS NOT NULL AND claimed_at >= ?
          )
    ) anchors WHERE anchors.rn = 1
)
ORDER BY id ASC LIMIT ? FOR UPDATE SKIP LOCKED
```

**步骤 2**：按选中 key 解析 eligible 行 id（`claimed_at IS NULL OR < staleBefore`，`Order id ASC`），按 id 认领。

**步骤 3**：`UPDATE task_pending_ops SET claimed_at=now WHERE id IN ids`，再 SELECT 返回。

关键设计：
- `limit` 计数 **distinct dedup_key（文档数）**，不是行数——保证「一个文档的多个 op 永不拆到两个并发 batch」。
- 僵尸 claim：`claimed_at < now-90m` 可回收；**fresh claim（`claimed_at >= staleBefore`）整体屏蔽其整个 dedup_key**，串行化同文档并发 op。

`decodePendingRows` 按 `knowledge_id` **last-write-wins 去重**（`[ingest, retract] → [retract]`），但 `peekedIDs` 仍含被折叠行的 dbID（供 trim 一次性删除）。

### 2.5 重试与死信

`requeueFailedOps`：对每个失败 op：
1. `IncrFailCount`（`UPDATE ... RETURNING fail_count` 原子自增）；
2. `count <= 5` → `ReleaseByIDs`（`claimed_at=NULL`）释放 claim，让行立即 eligible（无需等 90m）；
3. `count > 5` → 若 `Op==ingest` 先 `finalizeWikiSubtask` 释放槽位；`deadLetterRepo.Insert` 写死信；`DeleteByIDs` 移除。

服务级死信：`LastError="exceeded wikiMaxFailRetries=5 (in-batch retries)"`、`RelatedID=op.KnowledgeID`。

asynq 级死信（中间件 `asynqdl`）：handler 返回非 nil 且 `GetRetryCount >= GetMaxRetry`（最终尝试）时写一行，`truncateError(err, 8192)`。

### 2.6 启动恢复 `recover_pending_wiki_tasks.go`

1. 清理已删 KB 的遗留行（fail-closed：清理失败直接 return）；
2. 列出 `DISTINCT (tenant_id, task_type, scope_id)` 的待恢复 lane；
3. 每 lane 重建一个 trigger（finalize 带 TaskID 对齐 `scheduleFinalize`）。

不检查 `claimed_at`（僵尸判定在消费时 `ClaimBatch` 完成）；重复 trigger 无害。

### 2.7 每 KB 在途上限（Redis sorted set + Lua）

```lua
-- wikiInflightReserveScript（purge + count + add 原子执行）
local now = tonumber(ARGV[1])
local expiry = tonumber(ARGV[2])
local maxInflight = tonumber(ARGV[3])
local token = ARGV[4]
local ttl = tonumber(ARGV[5])
redis.call('ZREMRANGEBYSCORE', KEYS[1], 0, now)   -- 清过期槽位
if redis.call('ZCARD', KEYS[1]) >= maxInflight then return 0 end
redis.call('ZADD', KEYS[1], expiry, token)
redis.call('PEXPIRE', KEYS[1], ttl)
return 1
```

成功则起后台 goroutine 每 30s 续期 `ZAdd`；返回 release 闭包（`ZRem`）。Redis 错误 **fail-open**（返回 no-op + true）。

---

## 3. Map/Reduce 主流程

### 3.1 ProcessWikiIngest 主流程

```go
func (s *wikiIngestService) ProcessWikiIngest(ctx context.Context, t *asynq.Task) error
```

1. 解 payload → 注入 tenant/language ctx。
2. **Lite 模式**：`liteLocks.LoadOrStore(kbID)` 抢进程内锁，抢不到返回 `ErrWikiIngestConcurrent`（asynq 15s 固定重试）。
3. 取 KB，校验 `IsWikiEnabled`；取 synthesis 模型（`WikiConfig.SynthesisModelID` → fallback `SummaryModelID`）。
4. 解析 tunable：`batchSize(5)`、`mapParallel(10)`、`reduceParallel(10)`、`maxInflight(4)`。
5. **in-flight 槽位**：`reserveInflightSlot`；被拒 → `scheduleCappedRetry` 后 return（不认领）。
6. 认领（标准）或 peek（Lite）`batchSize` 个 op；load 失败向上抛。
7. 空结果 → `scheduleStaleClaimRecheck` 后 return。
8. **异常退出释放网**：defer 中若 `!claimsSettled`，用 detached ctx `ReleaseByIDs(peekedIDs)`。
9. **MAP**：`errgroup.SetLimit(mapParallel)`；retract op 运行时重查 `ListPagesBySourceRef`；ingest op 调 `mapOneDocument`；失败进 `failedOps`（不 fail 整批）。
10. **Taxonomy**：`planBatchTaxonomy` → `resolvePlannedFolders`（Reduce 前规划一次）。
11. **REDUCE**：`errgroup.SetLimit(reduceParallel)`，按 slug 分组，每组 `withSlugLock` 串行 RMW；锁超时/错误收集 `unappliedSlugKIDs`。
12. 尾部：`sanitizeDeadSummaryLinks` → `RecordWikiContentActivity` → `publishDraftPages` → `enqueueFinalize`。
13. 关 span + `finalizeWikiSubtask`（unapplied 的 doc 除外）。
14. **settlement**：`failedOps` 走 `requeueFailedOps`；trim 集 = `peekedIDs - failedIDs`；**先 trim 再 requeue**（成功兄弟先删，避免 follow-up 重拾）。
15. follow-up：`rateLimited ? 60s : 5s`，`PendingCount > 0` 才排。

### 3.2 Map 阶段 `mapOneDocument`

```go
func (s *wikiIngestService) mapOneDocument(ctx, chatModel, payload, op, batchCtx) (*docIngestResult, []SlugUpdate, error)
```

1. 开 span。
2. **guard 删除竞态**：`isKnowledgeGone`（Redis 墓碑 `wiki:deleted:{kb}:{kid}` + DB + ParseStatus ∈ {Deleting,Cancelled}）。
3. 加载 chunks。
4. `reconstructEnrichedContent`：纯文本 chunk 拼接（`MergeTextChunks` 重叠去重）+ 图片子 chunk OCR/caption 内联。
5. 截断 32KB（按 rune）。
6. `hasSufficientTextContent`：剥图片标记后 ≥10 rune。
7. 确定 docTitle（优先 DB Title，否则首个 chunk 首行）；`sourceRef = knowledgeID`（故意用 kid 而非文件名）。
8. 旧 slug 快照：`ListSlugsBySourceRef`（过滤 index）。
9. **Pass 0**：`extractCandidateSlugs`（失败回退 `extractEntitiesAndConceptsNoUpsert`）。
10. 构造 summary 的 wiki-link 输入；`summarySlug = "summary/"+slugify(knowledgeID)`。
11. **并行两条 goroutine**：build summary（`WikiSummaryPrompt`）+ classify citations（Pass 0 失败则跳过）。
12. `mergeCitationsIntoItems` 回填 `SourceChunks`。
13. summaryErr 非 nil → 整个 op 失败重试；否则拆 `sumLine/sumBody`。
14. 追加 entity/concept 的 `SlugUpdate`。
15. **新旧 slug 对账（三态）**：
    - `oldSlug ∉ new` → `retractStale`（`RetractDocContent`=新 content）；
    - `oldSlug ∈ new` 且 entity/concept → `reparse swap`（额外发 `retract` 带 `priorContribution`）；
    - `oldSlug ∈ new` 且 `summary/` → 不发（summary 整体覆盖）。

### 3.3 中间契约：SlugUpdate

```go
type SlugUpdate struct {
    Slug        string        // "entity/xxx" | "summary/<uuid>"
    Type        string        // "entity" | "concept" | "summary" | "retract" | "retractStale"
    Item        extractedItem // 仅 entity/concept
    DocTitle    string
    KnowledgeID string
    SourceRef   string        // "knowledgeID" 或 "knowledgeID|title"
    Language    string
    SummaryBody string        // summary 正文（SUMMARY 行之后）
    SummaryLine string        // summary 单行标题
    RetractDocContent string  // retract/retractStale 携带的上下文
    SourceChunks []string     // 支撑更新的 chunk IDs
    DocSummary string         // 文档级 summary body（写 <source_context>）
}

type extractedItem struct {
    Name, Slug, Description, Details string
    Aliases, SourceChunks            []string
}
```

### 3.4 Reduce 阶段 `reduceSlugUpdates`

```go
func (s *wikiIngestService) reduceSlugUpdates(ctx, chatModel, kbID, slug, updates, tenantID, batchCtx, kidToWikiSpan) (changed bool, affectedType string, additionFailed bool, err error)
```

1. `filterLiveUpdates`：retract/retractStale 保留；其余若 `isKnowledgeGone` 丢弃。
2. `GetPageBySlug`；不存在且只有 retract → return。
3. 分桶：summary / retracts / additions。
4. **summary 分支**：直接覆盖正文（`Content=SummaryBody`、`Summary=SummaryLine`、`ChunkRefs=[]`）。
5. **entity/concept 分支**：构造三块上下文：
   - `<deleted_documents>`（每个 retract 的 title+content）；
   - `<remaining_source_documents>`（存活 SourceRefs 的 summary body）；
   - `<new_information>`（每个 addition 的「Name: Description + 逐字 cited chunk 内容」，无 cited 回退 Details）；
   - `<shared_source_contexts>`（DocSummary，放最前保证前缀缓存）。
6. **slugHandles 编解码**：`OutLinks` → `ref-N` 句柄；`stripWikiInlineChunkCitations` 去 `[cNNN]`；生成后 `decodeContent` 还原。
7. `WikiPageModifyUserPrompt` 填充 15 个模板变量（含 `HasAdditions`/`HasRetractions` 开关）。
8. taxonomy 应用：`FolderID==""` 时取 `PlannedFolderID[slug]`。
9. `mergeChunkRefs` + `UpdatePage`/`CreatePage`。

### 3.5 Finalize `ProcessWikiFinalize`

1. per-KB finalize 锁（`SetNX "wiki:finalize:active:"+kbID`，fail-closed）。
2. `PeekBatch(finalize lane, limit=5000)`。
3. 聚合 rows（slug/change/folder 三类）。
4. 重建 index intro（`WikiIndexIntroPrompt` / `WikiIndexIntroUpdatePrompt`）。
5. `cleanDeadLinks`（死链改写）。
6. `injectCrossLinks`（交叉链接注入）。
7. `PruneEmptyFolderChains`（需 ingest lane 空，否则 defer + 1m 重试）。
8. trim 已处理行；仍有行则 reschedule。

### 3.6 并发与锁

- **Map/Reduce errgroup**：`SetLimit(10)`，goroutine 永不返回非 nil（失败进 `failedOps`/`unappliedSlugKIDs`）。
- **withSlugLock**：Redis `SetNX` 5m TTL，2m 超时让出（上层 collectUnapplied），50ms 轮询；Redis 错误 **fail-open**；Lite 模式直接执行。
- **单飞合并**：`generateWithTemplate` 用 `singleflight.DoChan` 合并字节相同请求；`WikiPageModifyUserPrompt` 有前缀预热机制。
- **LLM 重试**：`wikiLLMMaxAttempts=3`，仅 `isTransientLLMError`（408/429/5xx/超时）指数退避 `2s << (attempt-1)`。

---

## 4. 抽取子管线（cite / dedup / taxonomy / slug-handles）

### 4.1 统一 LLM 调用入口 `generateWithTemplate`

```go
func (s *wikiIngestService) generateWithTemplate(ctx, chatModel, promptTpl string, data map[string]string) (string, error)
```

- Go `text/template` 解析；`maskTemplateDataImageURLs` 遮蔽图片 URL；
- **单 user 消息（无 system）**；`Temperature=0.3`、`Thinking=false`、`MaxTokens=32768`；
- 指数退避重试，仅 transient LLM error；
- `cleanLLMJSON` 剥 ` ```json ``` ` 围栏 + `sanitizeJSONString` 转义控制字符。

### 4.2 chunk 引用管道（`wiki_ingest_cite.go`）

```go
const (
    maxRunesPerCitationBatch  = 12000  // 单批 rune 预算
    maxCitationBatchConcurrency = 4    // 跨批并行度
)
```

`splitChunksIntoCitationBatches`：过滤（只留 `ChunkTypeText`/空）、按 `ChunkIndex`+`StartAt` 排序、贪心装批（超 12000 且非空则 flush；**单 chunk 超预算也单独成批**）。每批局部 `HandleTable("c", 3, 0)` → `c000`。

两个渲染函数（决定 prompt 变量形状）：

```xml
<!-- CandidateSlugs：行列表 -->
- slug: entity/zhang-san, type: entity, name: "张三", aliases="zs, zhang", description: ...

<!-- ChunksXML：c 标签 -->
<c id="c000" index="0">
第一个 chunk 正文……
</c>
```

`classifyChunkCitations`：`errgroup.SetLimit(4)` 并行各批；**单批失败不中断**；解析后 `batch.handles.Resolve(handle)` 反查真 UUID，写 `citationSet[slug][realID]=true`；最终按 `ChunkIndex` 稳定排序。

**降级**：无 citation 的候选 `SourceChunks` 为空 → Reduce 回退到 `Item.Details`（Pass 0 的 1–3 句兜底）。

**前缀缓存**：块顺序固定为「静态规则 → 输出 schema → candidate_slugs（文档内稳定）→ chunks（逐批变化）」，同文档多批共享长前缀。

### 4.3 去重（`wiki_ingest_dedup.go`）

```go
const (
    dedupCandidateTopK       = 5     // 每个 query 词相似页 LIMIT
    dedupCandidateScoreFloor = 0.08  // Jaccard 下界
    dedupSmallCorpusBypass   = 25    // corpus ≤25 页跳过预筛
)
```

**粗筛**（双信号）：
- DB 侧 `FindSimilarPages`：pg_trgm `similarity(lower(title), q)` + `%` 操作符（默认阈值 0.3），`ORDER BY sim DESC LIMIT 5`。
- 内存侧 `selectDedupCandidatePages`：slug 分词 Jaccard + 字符 bigram Jaccard（取 max）；`score >= 0.08` 必选，否则 top-K 且 `score>0` 才选，`score==0` 直接 break。

`surfaceGrams`：小写、去非字母数字 → 相邻 rune 双字 bigram，单字符退化 1-gram（对 CJK 和 Latin 都有效）。

**精判** `deduplicateExtractedBatch`：每个新 item 的 `[Name]+Aliases` 作为 query 集 `FindSimilarPages`；candidates 按 item 分组写 `<item><candidates>` XML；`WikiDeduplicationPrompt` 判定，输出 `{"merges": {"newSlug": "existingSlug"}}`。

**硬校验** `dedupMergeRejectReason`（独立于 LLM 的确定性 gate）：

```go
// ① 目标必须是 src item 自身候选集成员（防跨 item 幻觉配对）
// ② 必须带 "/" 前缀
// ③ 前缀（含 "/"）必须一致（entity↔entity，concept↔concept）
```

合并后存活 slug = 已有页 slug，新 item 的 `Slug` 被改写。

### 4.4 目录规划（`wiki_ingest_taxonomy.go`）

```go
const (
    wikiTaxonomyPromptMaxPaths    = 150
    wikiTaxonomyFolderPoolMax     = 400
    wikiTaxonomyFeedAllMaxFolders = 60   // ≤60 整体喂入
    wikiTaxonomyRelevantTopK      = 3
    wikiTaxonomyPlanChunkSize     = 60
)
```

`planBatchTaxonomy`：collect items → `ListDistinctCategoryPaths` 拉 pool → `selectRelevantFolders` 缩小 → 按 60 分块，逐块 `WikiTaxonomyPlanPrompt` + `parseTaxonomyAssignments`，**新目录前馈进后续块**（收敛到同一棵树）。

`selectRelevantFolders`（embedding 选 top3）：
- pool ≤60 → 整体返回；
- `EmbeddingModelID==""` 或 embedding 失败 → `capFolders(pool, 150)` 截断；
- 否则 level-1 全保留 + 每 item 取余弦相似度 top-3 deeper，`capFolders(150)`。**无相似度阈值**，只按 top-K。

`resolvePlannedFolders`：`pathCache` 去重 + **negative cache**（失败记 `""`）；`FindOrCreateFolderPath` 顺序建目录（Reduce 前串行建好，Reduce 只赋 ID）。

### 4.5 slug 句柄（`wiki_slug_handles.go` + `modelcontext`）

```go
// slug 句柄：prefix="ref-", width=0, start=1 → ref-1, ref-2, ...
func newWikiSlugHandles() *wikiSlugHandles {
    return &wikiSlugHandles{table: modelcontext.NewHandleTable("ref-", 0, 1)}
}
```

`HandleTable`（泛型 `handleTable[M]`）核心：

```go
type handleTable[M any] struct {
    mu            sync.RWMutex
    prefix        string
    width         int
    next          int
    handleByKey   map[string]string
    entryByHandle map[string]*handleEntry[M]  // 含 wordBounded 正则（注册时编译一次）
}
```

- `Register(value)`：首次分配句柄；重复返回旧句柄。**entry 永不删除**（编号稳定）。
- `Resolve(handle)`：句柄→真值。
- 命名：`width>0` 零填充（`c000`），`width==0` 纯十进制（`ref-1`）。
- 生命周期：**调用局部、不持久化**。

`encodeContent`/`decodeContent` 委托 `rewriteDeadWikiLinks`（正则 `\[\[([^\]]+)\]\]`），只替换 known 集合内的链接，未映射句柄原样保留（防误替换 `[[ref-99]]`）。

---

## 5. 页面 CRUD 与链接维护

### 5.1 CreatePage / UpdatePage

`CreatePage` 顺序：默认 `Status=published`/`Version=1` → 记录作者 → `stripWikiPageInlineChunkCitations` → `parseOutLinks` → `applyFolderToPage`（从 FolderID 反查重算 CategoryPath）→ `normalizeWikiHierarchy` → `repo.Create` → `updateInLinks`（建立反向链接）。

`UpdatePage` 关键逻辑：

```go
contentChanged := existing.Title != page.Title ||
    existing.Content != page.Content || existing.Summary != page.Summary ||
    existing.PageType != page.PageType || existing.Status != page.Status ||
    !slices.Equal(existing.Aliases, page.Aliases)
```

- `contentChanged` → `repo.UpdateWithRevision`（事务：快照 + 版本化更新）→ `pruneRevisions`；
- 否则 → `repo.UpdateMeta`（bookkeeping 不 bump version）；
- 两种情况都执行链接维护 `removeInLinks(oldOutLinks)` + `updateInLinks(newOutLinks)`。

### 5.2 乐观锁 SQL

```go
func updateWikiPageRow(db *gorm.DB, page *types.WikiPage) error {
    expectedVersion := page.Version
    page.Version = expectedVersion + 1
    result := db.Model(page).
        Where("id = ? AND version = ?", page.ID, expectedVersion).
        Updates(map[string]interface{}{ /* 显式列 map，含零值字段 */ })
    if result.RowsAffected == 0 {
        // 查 id 区分 ErrWikiPageNotFound vs ErrWikiPageConflict
    }
}
```

要点：用**显式 map**（非 GORM struct Updates）保证零值字段能落库；`WHERE id=? AND version=?` 用自增前版本；失败回滚 `page.Version`。

HTTP 层另有前置乐观锁：`req.Version > 0 && req.Version != existing.Version` → 409 `{"error":"Wiki page was modified by someone else","current_version":...}`。

### 5.3 三类写法的区别（哪些不 bump version）

| 方法 | bump version | 重解析 OutLinks | 更新 InLinks | 用途 |
| --- | --- | --- | --- | --- |
| `UpdatePage` | 仅 contentChanged | 是 | 是 | 真实编辑 |
| `UpdatePageMeta` | 否 | 否 | 否 | status/source_refs 等 |
| `UpdateAutoLinkedContent` | 否 | 是 | 是 | 交叉链接注入/死链清理 |

`UpdateMeta` 的列（不含 version）：in_links/out_links/aliases/status/source_refs/chunk_refs/page_metadata/parent_slug/folder_id/category_path/wiki_path/depth/sort_order/updated_at。

`UpdateAutoLinkedContent` 只写 `content/out_links/updated_at`（`WHERE id=?`，无 version）。

### 5.4 版本快照与回滚

`UpdateWithRevision` 事务内 `ON CONFLICT (page_id, version) DO NOTHING` 插快照 + 版本化更新（并发写同版本幂等）。

`pruneRevisions`：软上限 50 只删 `edit_source IN ('','pipeline')`，硬上限 200 无差别删；触发点只在 contentChanged 写之后。

`RevertPageToVersion`：取快照 → 覆盖 title/content/summary/page_type/status/aliases → 走 `UpdatePage`（`WikiEditSourceRevert`）。folder/sort_order/refs 保持当前值（回滚只针对内容）。

### 5.5 双向链接（核心不变量：正文是唯一真相源）

- `OutLinks` 永远 `parseOutLinks(Content)` 重解析，正则 `\[\[([^\]]+)\]\]`，处理 `[[slug|display]]` 取 `|` 左边，`normalizeSlug`（lower + 空格→`-`）+ 去重。
- `InLinks` 由 `updateInLinks`/`removeInLinks` 按「旧 OutLinks vs 新 OutLinks 差集」增量维护。
- `RebuildLinks` 全量重算：先清空所有 InLinks，再遍历重算，全部走 `UpdateMeta`（不 bump）。

### 5.6 交叉链接注入（`wiki_linkify.go`）

`linkifyContent(content, refs, selfSlug)`：
1. 过滤空 ref、跳过 `slug == selfSlug`；
2. 按 matchText rune 数降序（长名优先）；
3. `computeForbiddenSpans` 扫禁区（fenced code / inline code / 已有 `[[...]]` / markdown 链接 / 引用式链接 / autolink）；
4. 每个 ref：若 slug 已在 `used`（正文已链接）跳过；`findFirstSafeMatch` 找首个安全匹配；插入 `[[slug|matchText]]`；更新 forbidden + used。

`findFirstSafeMatch` 的 word-boundary：仅 ASCII 字母/数字边界的 needle 才做词边界检查；**非 ASCII（CJK）一律视为边界**——所以 `"北京"` 能匹配进 `"北京邮电大学"`（由「按长度降序」另行解决）。

### 5.7 图片 URL 掩码（opaque token）

```go
mdImageURLRE   = regexp.MustCompile(`!\[[^\]]*\]\(([^)]*)\)`)
imageURLAttrRE = regexp.MustCompile(`(?i)<image\b[^>]*\surl\s*=\s*(?:"([^"]*)"|'([^']*)')`)
```

- 掩码：`maskImageURLs` 把 URL 换成 `wkimg:0001`（低熵不透明 token），按 URL 长度降序替换；
- 还原：`unmaskImageURLs` 用 `urlMap`（token→URL）反向映射，未知 `wkimg:` 占位符直接丢弃（防 LLM 杜撰落库）。

### 5.8 行内引用剥离

```go
var wikiInlineChunkCitationRegex = regexp.MustCompile(`[ \t]*\[c\d{3,}(?:\s*[,;]\s*c\d{3,})*\]`)
```

写库前（CreatePage/UpdatePage/UpdateAutoLinkedContent）与读路径（GetPageBySlug/GetPageByID/ListPages）都 strip，双保险；稳定引用只存 `ChunkRefs`。

### 5.9 死链处理（三层，层层保守）

`resolveDeadSlug`（`slug_fuzzy.go`）三级解析：
1. display 反查（`titleToSlug`，最准）；
2. 归一化相等（lower + 去 `-`/`_`）；
3. 字符 bigram Jaccard ≥ `slugResolveBigramThreshold=0.8`。

`stripDeadWikiLinks`：优先 `resolveDeadSlug` rewrite，否则 strip 成纯文本（display 优先，否则 humanize slug 末段）。

`RepairContentLinks`（rewrite-only，**永不 strip**）：`ExistsSlugs` 分类 live/dead → 按 `slugNamespace`（首个 `/` 前）收集候选 → `rewriteDeadWikiLinks` 只对「存在安全候选」的 dead 链接 rewrite。

### 5.10 软删除 vs 归档

`DeletePage`：`removeInLinks`（摘除给别人的引用）→ `repo.Delete`（软删 `DeletedAt`）→ `DeleteRevisionsByPage`（硬删快照）→ `deleteChunkForPage`（删同步 chunk）。

`archived` 状态与 `DeletedAt` 都通过 `status <> 'archived'` / gorm 默认过滤让读路径看到一致视图。

### 5.11 健康检查 / lint（`wiki_lint.go`）

Issue 类型：`orphan_page`(warning) / `broken_link`(error,可修复) / `stale_ref`(error,可修复) / `missing_cross_ref`(info) / `empty_content`(warning,可修复)。

两遍游标遍历（`ListPagesCursor`，200/批，`id ASC`）：
- Pass 1：orphan（非 index 且 InLinks 空）、broken link（OutLinks 不在 live set）、empty（Content <50）、stale ref（SourceRefs 软删）；
- Pass 2：missing cross-ref（正文含 entity/concept 标题但未链接）。

健康分 100 起扣：orphan>50% 扣 25、orphan>25% 扣 10、broken×5、totalLinks==0 且 pages>2 扣 15、empty×3。

`AutoFix`：broken link → strip；empty → archive；stale ref → 去 ref（空则 DeletePage）。

---

## 6. 在线消费：Agent 工具 / 重排 / 权限 / 句柄

### 6.1 工具清单（10 个）

工具名常量（`definitions.go`）：

```go
ToolWikiReadPage      = "wiki_read_page"
ToolWikiWritePage     = "wiki_write_page"
ToolWikiReplaceText   = "wiki_replace_text"
ToolWikiRenamePage    = "wiki_rename_page"
ToolWikiDeletePage    = "wiki_delete_page"
ToolWikiSearch        = "wiki_search"
ToolWikiReadSourceDoc = "wiki_read_source_doc"
ToolWikiFlagIssue     = "wiki_flag_issue"
ToolWikiReadIssue     = "wiki_read_issue"
ToolWikiUpdateIssue   = "wiki_update_issue"
```

（`wiki_link_mutation` 不是工具，是 rename/delete 内部调用的辅助函数。）

### 6.2 wiki_search（POSIX 正则）

```sql
SELECT *,
  CASE
    WHEN title   ~* :q THEN 4
    WHEN slug    ~* :q THEN 3
    WHEN summary ~* :q THEN 2
    WHEN content ~* :q THEN 1
    ELSE 0 END AS match_rank
FROM wiki_pages
WHERE knowledge_base_id = :kb
  AND (title ~* :q OR content ~* :q OR summary ~* :q OR slug ~* :q)
  AND status != 'archived'
ORDER BY match_rank DESC, updated_at DESC
LIMIT :limit;
```

返回 `<search_results>` XML，含 `<page>`（link/type/aliases/summary/match_snippet）。snippet 用 Go `regexp.Compile("(?i)"+query)` 找首个匹配，前后各取 60 runes。

### 6.3 wiki_read_page（`<wiki_page>` XML）

```xml
<wiki_page>
<metadata><knowledge_base_id>kb-1</knowledge_base_id><link>[[entity/acme-corp|Acme Corp]]</link><type>entity</type><aliases>ACME, Acme</aliases></metadata>
<relationships><links_to>...</links_to><linked_from>...</linked_from></relationships>
<sources><source knowledge_id="uuid-1">Doc Title</source></sources>
<summary>one-line summary</summary>
<content>full markdown body</content>
</wiki_page>
```

**输出预算截断**（`renderWikiPagesWithinBudget`）：预算 `OutputBudget(ctx)`（默认 24000 runes）；`usable = budget - 600`；超出时减少保留页数（`fixedCost + keep*400 > usable`），再对保留页用 max-min fair 分配 body 配额；被砍页进 `<omitted_pages>`，被截页标记 truncated。

### 6.4 wiki_write_page（slug 校验）

`normalizeAndValidateWikiSlug`：lower + 空格→`-`；禁止 `//`/前导`/`/后缀`/`；每 rune 只允许 `a-z/0-9/-/` + CJK。

- summary 命名空间保护：`summary/` 前缀或 `page_type==summary` 且页不存在 → 拒绝（只能更新已存在的 summary 页）；
- `RepairContentLinks`：死链只 rewrite 不 strip；
- 写来源标记 `WikiEditSourceAgent`。

### 6.5 权限与路由

`WikiScope`：`{KnowledgeBaseID, KnowledgeIDs[], TagIDs[]}`。

`pagePassesWikiScope`（fail-closed）：
1. 无过滤直接通过；
2. **结构性页面（index）与无来源页（SourceRefs 空）在有过滤 scope 下一律拒绝**；
3. `extractSourceKnowledgeIDs` 拆 SourceRefs 的 uuid；
4. KnowledgeIDs 白名单命中 → 通过；
5. 否则查 source tags 与 TagIDs 交集。

`WikiRouteResolver`：`bySlug map[string]map[string]struct{}`（slug→kbID 集合）；`remember`/`forget` 登记；`scopesForSlug` 返回缓存 owner 中仍属 scope 的 KB（只影响顺序）。

`resolveUniqueWikiPage`（写/改/删/issue 共享边界）：命中 0 → not found；1 → 返回；>1 → **歧义拒绝**（绝不静默写第一个 KB）。

`KBCapability CapWiki = "wiki"`，映射到 `kb.IsWikiEnabled()`（=`IndexingStrategy.WikiEnabled`）。运行时两级过滤：无 wiki KB 时 10 个工具全剔除；AgentShare 只读场景再剥掉 6 个写工具。

### 6.6 重排加权（`wiki_boost.go`）

```go
const wikiBoostFactor = 1.3
// 挂 CHUNK_RERANK 阶段；先 next() 跑正常 rerank
// 快速路径1：无 ChunkTypeWikiPage chunk 直接 return
// 快速路径2：无 KB IsWikiEnabled 直接 return
// 对 wiki chunk Score *= 1.3，sort.SliceStable 降序
```

`ChunkTypeWikiPage ChunkType = "wiki_page"`。

### 6.7 chunk 同步（缺口提醒）

`deleteChunkForPage`：chunk ID 约定 `"wp-" + page.ID`；删除用 `DeleteChunk(tenantID, chunkID)`。**「写侧」（页面正文落成 `wiki_page` chunk）不在后端 Go 代码中**，若目标系统需要「wiki 页进向量库」需自行补写侧。

---

## 7. Prompt 模板清单

全部在 `internal/agent/prompts_wiki.go`，9 个模板 + 3 档 granularity 指引。详见架构文档 §5.6（已单独展开）。此处只列速查表：

| Prompt 常量 | 阶段 | 关键约束 |
| --- | --- | --- |
| `WikiSummaryPrompt` | Map·summary | 不喂文件名；空内容规则 |
| `WikiCandidateSlugPrompt` | Map·Pass 0 | granularity；slug 连续性 |
| `WikiChunkCitationPrompt` | Map·Pass 1..N | 只准从 chunks 块照抄句柄 |
| `WikiTaxonomyPlanPrompt` | Taxonomy | 复用目录字符级一致 |
| `WikiPageModifySystemPrompt` | Reduce | compiler not writer、逐字接地 |
| `WikiPageModifyUserPrompt` | Reduce | CRITICAL CONFLICT CHECK |
| `WikiIndexIntroPrompt` | Finalize 首建 | 不生成目录链接 |
| `WikiIndexIntroUpdatePrompt` | Finalize 增量 | 保持语气格式 |
| `WikiDeduplicationPrompt` | 去重 | related ≠ same |

granularity 三档：`focused`(3–7 个主干) / `standard`(默认，主干+实质讨论) / `exhaustive`(所有具名项)。

---

## 8. 移植路线图：从最小可行版到完整版

### 第 0 步：身份模型（必须最先做）

每个知识对象有稳定 key（对应 slug）+ 类型（entity/concept/summary）+ 状态（draft/published/archived）+ `version`。**别用随机 ID 当身份**。slug 用「KB 内部分唯一索引（`WHERE deleted_at IS NULL`）」。

### 第 1 步：「引用 + 编译」两段式（先跳过 taxonomy/去重）

- 引用段：对每个候选让 LLM 标「哪些 chunk 讨论它」（§4.2，接地根基）；
- 编译段：让 LLM 做编译器式合并（§3.4，新增逐字来自引用 chunk）；
- 这一步做好，「幻觉」和「删文档撤不掉内容」两大痛点解决一大半。

### 第 2 步：slug 句柄间接层（几乎免费的保险，尽早加）

只花一个映射表（§4.5），把「LLM 抄错 ID」归零。

### 第 3 步：去重「廉价预筛 + LLM 裁决」

先 pg_trgm/Jaccard 收窄（§4.3），LLM 只判 top-k，硬校验 gate 独立于 LLM。

### 第 4 步：链接存一份、索引派生

正文是唯一真相，正反链随写重建（§5.5）。这让你以后加「改名自动更新」「死链清理」几乎零成本。

### 第 5 步：可靠性基建（按需选）

单机先跳过分布式锁/死信；但**「待办持久化到 DB 而非内存队列」**哪怕单机也值得做（§2）——异步流水线最怕「入队后进程挂」丢任务。

### 第 6 步：检索侧二选一

- 轻量：专用读工具（`wiki_search`/`wiki_read_page`，直查 DB，不碰向量管道）；
- 重量：wiki 页也落成 chunk 进混合检索 + 重排加权 ×1.3（记得补写侧，§6.7）。

### 移植时必须一起带上的硬约束

1. **slug 部分唯一索引**（软删后复用 slug）。
2. **乐观锁** `UPDATE ... WHERE id=? AND version=?`，当前版本排除在快照表外。
3. **TaskPendingOp 覆盖索引** `(task_type, scope, scope_id, id)` 支撑 `id ASC` 游标认领。
4. **认领 `limit` 计 distinct dedup_key**，保证同文档多 op 不拆分。
5. **版本号语义收敛**：bump 唯一条件 = 内容字段变化；其余全走 UpdateMeta/UpdateAutoLinkedContent。
6. **权限 fail-closed**：index/无源页在受限 scope 下拒绝；多 KB 歧义拒绝而非静默写第一个。
7. **死链处理三层保守**：resolve → rewrite → strip（RepairContentLinks 永不 strip）。
8. **大 KB 全量操作走游标或投影**：`ListPagesCursor`/`ListAllSlugs`/`ExistsSlugs`，禁 `ListAll` 全量物化。

---

## 9. 关键文件索引

| 关注点 | 文件 |
| --- | --- |
| 数据模型 / 常量 / 清洗 | `internal/types/wiki_page.go` |
| 接口契约 | `internal/types/interfaces/wiki_page.go` |
| 仓储实现（SQL/乐观锁/快照） | `internal/application/repository/wiki_page.go` |
| 全部 prompt 模板 | `internal/agent/prompts_wiki.go` |
| 入队/认领/常量/重试/死信 | `internal/application/service/wiki_ingest.go` |
| Map/Reduce 主体 + Finalize | `internal/application/service/wiki_ingest_batch.go` |
| chunk 引用管道 | `internal/application/service/wiki_ingest_cite.go` |
| 去重 | `internal/application/service/wiki_ingest_dedup.go` |
| 目录规划 | `internal/application/service/wiki_ingest_taxonomy.go` |
| 页面 CRUD + 链接维护 | `internal/application/service/wiki_page.go` |
| 交叉链接注入 | `internal/application/service/wiki_linkify.go` |
| slug 句柄 | `internal/application/service/wiki_slug_handles.go` |
| 死链模糊修复 | `internal/application/service/slug_fuzzy.go` |
| 健康检查 / lint | `internal/application/service/wiki_lint.go` |
| 重排加权 ×1.3 | `internal/application/service/chat_pipeline/wiki_boost.go` |
| Agent 工具集 | `internal/agent/tools/wiki_tools.go` + `wiki_*.go` |
| 路由解析 | `internal/agent/tools/wiki_route_resolver.go` |
| 句柄原语 | `internal/modelcontext/handles.go` + `handle_table.go` |
| HTTP handler | `internal/handler/wiki_page.go` |
| 启动恢复 | `internal/container/recover_pending_wiki_tasks.go` |
| 队列/死信类型 | `internal/types/task_pending_op.go` / `task_dead_letter.go` |
| migration DDL | `migrations/versioned/000037` / `000041` / `000061` / `000075` |
