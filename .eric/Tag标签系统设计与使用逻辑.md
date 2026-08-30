# WeKnora 标签（Tag）系统：设计原理与使用逻辑

> 本文梳理 WeKnora 的**标签（Tag）系统**——它如何在一个知识库内对知识做「分类」，并让这个分类在**入库、检索、问题推荐**三个环节都发挥作用。
>
> 核心结论：WeKnora 的标签**不是**一套全局扁平的关键字表，而是一个**「按知识库 + 租户作用域」的二级分类体系**，并且对**两类知识**用了**两种完全不同的存储与落地方式**——文档（Document）走「知识-标签」多对多关联表，FAQ 条目走「chunk 上直接挂单标签字段」。理解这一点是理解整个标签系统的钥匙。
>
> 代码基准：`internal/`（Go 后端）。它属于**增强功能**——关掉标签，问答照常工作；标签只是叠在检索之上的一层「范围限定 + 内容组织」。

---

## 目录

1. [总览：一套标签、两类知识、两种存储](#1-总览一套标签两类知识两种存储)
2. [数据模型：Tag、Relation 与 AutoTagConfig](#2-数据模型tagrelation-与-autotagconfig)
3. [标签作用域与双标识：KB 内、seq_id + UUID](#3-标签作用域与双标识kb-内seq_id--uuid)
4. [两种存储与检索落地](#4-两种存储与检索落地)
5. [标签的增删改：分层删除语义](#5-标签的增删改分层删除语义)
6. [FAQ 标签：导入解析与两级优先检索](#6-faq-标签导入解析与两级优先检索)
7. [文档标签：手动打标与自动打标](#7-文档标签手动打标与自动打标)
8. [标签在检索与问题推荐中的使用](#8-标签在检索与问题推荐中的使用)
9. [关键设计原则（可移植）](#9-关键设计原则可移植)
10. [关键文件索引](#10-关键文件索引)
11. [附：标签系统全景图](#11-附标签系统全景图)

---

## 1. 总览：一套标签、两类知识、两种存储

```text
                    ┌──────────────────────────────────────────┐
                    │           KnowledgeTag（标签本体）          │
                    │   ID(UUID) · SeqID · TenantID · KBID ·     │
                    │   Name(库内唯一) · Color · SortOrder        │
                    └───────────────┬──────────────────────────┘
                                    │ 被引用
              ┌─────────────────────┴─────────────────────┐
              │                                           │
   ┌──────────▼──────────┐                 ┌───────────────▼────────────┐
   │ 文档 KB（多对多）      │                 │ FAQ KB（chunk 单标签）       │
   │ knowledge_tag_relations             │ chunk.tag_id (varchar36)  │
   │ (KnowledgeID, TagID) │                 │ 一条 FAQ = 一个 chunk = 一个标签│
   └──────────┬──────────┘                 └───────────────┬────────────┘
              │ 检索时: tag → knowledgeIDs                 │ 检索时: index tag_id 过滤
              │ (ListIDsByTagIDs, OR)                      │ (ContainsAny)
              ▼                                            ▼
          过滤 knowledge_id 字段                        过滤 tag_id 字段
```

一句话：**标签本体只有一份**（`KnowledgeTag`，挂在某个 KB 下），但**引用方式分两种**——文档通过关联表挂多个标签，FAQ 条目在 chunk 上直接存一个标签 ID。

---

## 2. 数据模型：Tag、Relation 与 AutoTagConfig

### 2.1 KnowledgeTag（标签本体）

`internal/types/tag.go`：

```go
type KnowledgeTag struct {
    ID              string    // UUID 主键
    SeqID           int64     // 自增整数 ID，供外部 API 使用（见 §3）
    TenantID        uint64    // 所属工作区
    KnowledgeBaseID string    // 所属知识库
    Name            string    // 标签名，同一 KB 内唯一
    Color           string    // 展示颜色（可选）
    SortOrder       int       // 排序，默认 0
    CreatedAt, UpdatedAt time.Time
}
```

- **作用域是「知识库」而非「全局」**：同一个名字「产品」可以在 KB-A 和 KB-B 各有一个，互不影响（`GetByName` 用 `tenant_id + knowledge_base_id + name` 三键）。
- **`SeqID` 是外部 API 用的自增整数**，与内部 UUID 并存。`BeforeCreate` 钩子只为 SQLite 手动补 SeqID（PostgreSQL/MySQL 交给 DB 序列，避免并发插入重复键）。

### 2.2 KnowledgeTagRelation（文档的关联表）

```go
type KnowledgeTagRelation struct {
    KnowledgeID string `gorm:"primaryKey"`
    TagID       string `gorm:"primaryKey"`
    CreatedAt   time.Time
}
func (KnowledgeTagRelation) TableName() string { return "knowledge_tag_relations" }
```

复合主键 `(KnowledgeID, TagID)`，天然去重、支持幂等 `ON CONFLICT DO NOTHING`。表名是显式指定的。

### 2.3 Chunk.TagID（FAQ 的单标签字段）

`internal/types/chunk.go:125`：`TagID string`（`varchar(36);index`）。**每个 chunk 最多一个标签**——FAQ 的「一条标准问 = 一个 chunk」，所以 FAQ 就是「一条 FAQ = 一个标签」。这个字段会同步进向量库（见 §4）。

### 2.4 统计与计数

- `KnowledgeTagWithStats` = `KnowledgeTag` + `KnowledgeCount` + `ChunkCount`，列表接口返回时带上「多少个文档、多少个 chunk 用了这个标签」。
- `TagReferenceCounts` 是批量计数的中间结果（`map[tagID]counts`）。
- 计数来源两处：`knowledge_tag_relations`（文档数）+ `chunks.tag_id`（chunk 数），用 `BatchCountReferences` 两条 SQL 一次算完所有标签（避免 2×N 查询）。

### 2.5 AutoTagConfig（自动打标配置）

`internal/types/knowledgebase.go:175`，挂在 **document 类型 KB** 上，**显式 opt-in**（默认关闭，升级不引入额外模型调用）：

```go
type AutoTagConfig struct {
    Enabled       bool
    ModelID       string   // 空则回退 kb.SummaryModelID
    MaxTags       int      // 默认 3，上限 10
    SkipIfTagged  *bool    // 默认 true：已有标签的文档跳过，不稀释人工分类
}
```

`SkipIfTagged` 用指针是为了区分「老数据字段缺失（nil）」→ 默认 true，而不是误翻成「总是追加」。

---

## 3. 标签作用域与双标识：KB 内、seq_id + UUID

**作用域**：标签归 `(TenantID, KnowledgeBaseID)` 所有，所有查询都带租户过滤。跨租户共享 KB 时，handler 层的 `g.KBAccessRead/Write` 路由守卫会把 context 里的 tenant 改写成**源租户**，所以下游按 `c.Request.Context()` 拿到的就是能命中数据的那一侧（`handler/tag.go:18-23` 注释明确）。

**双标识**：

| 标识 | 类型 | 用途 |
| --- | --- | --- |
| `ID` | UUID 字符串 | 内部主键、chunk.tag_id、关联表、索引过滤 |
| `SeqID` | 自增 int64 | 对外 API 的人类可读 ID（FAQ 导入、更新、检索接口用 int） |

`resolveTagID`（`handler/tag.go:52`）：`/tags/:tag_id` 的路径参数**既可以是 UUID 也可以是 seq_id**——先 `strconv.ParseInt` 试 int，成功就走 `GetBySeqID` 拿回 UUID；否则当作 UUID 直接返回。这层「宽容解析」让外部调用方（前端、SDK）可以用整数 ID 干活，内部始终用 UUID。

---

## 4. 两种存储与检索落地

这是整个系统的**核心设计**——同一个「标签」概念，对两类知识用了两套落地，代价是检索时要用两套过滤：

| 维度 | 文档（Document）KB | FAQ KB |
| --- | --- | --- |
| 存储 | `knowledge_tag_relations` 关联表（多对多） | `chunk.tag_id` 单字段 |
| 一个对象能挂几个标签 | 多个 | 一个 |
| 检索时如何落地 | tag → `ListIDsByTagIDs` 解出 knowledgeIDs，过滤 `knowledge_id` | 直接过滤索引里的 `tag_id` 字段 |
| 索引同步 | 不改 chunk，只改关联表 | `BatchUpdateChunkTagID` 同步向量库 |

**FAQ 侧**：`tag_id` 是向量库里的一个可过滤字段（`weaviate/repository.go:497`），检索时用 `ContainsAny` 过滤：

```go
if len(params.TagIDs) > 0 {
    operands = append(operands, filters.Where().
        WithPath([]string{fieldTagID}).
        WithOperator(filters.ContainsAny).
        WithValueText(params.TagIDs...))
}
```

`BatchUpdateChunkTagID`（`retriever/…`）在 FAQ 条目换标签时，把 `chunk.tag_id` 的新值批量写回向量库，保证索引里的 tag_id 与 DB 一致。

**文档侧**：标签只挂在 `knowledge_tag_relations`，chunk 上的 `tag_id` 字段是空的。检索时 `resolveSuggestionTagScopes`（§8）或查询装配阶段，先把「标签 ID」翻译成「knowledge ID 列表」（`ListIDsByTagIDs`，OR 语义，`knowledge.go:1027`），再按 `knowledge_id` 过滤。

> 为什么文档不也往 chunk 上挂 tag_id？因为文档是「一整篇文档」的分类，一篇文档可能拆成几十个 chunk、且可能属于多个标签；把标签铺到每个 chunk 会造成冗余和同步负担，关联表 + 运行时翻译更干净。FAQ 是「一条一条」的原子条目，天然适合单字段。

---

## 5. 标签的增删改：分层删除语义

CRUD 走 `knowledgeTagService`（`service/tag.go`）+ `TagHandler`（`handler/tag.go`），路由 `RegisterKnowledgeTagRoutes`（`routes_knowledge.go:264`）：

| 方法 | 路径 | 说明 |
| --- | --- | --- |
| GET | `/knowledge-bases/:id/tags` | 列表 + 批量统计（Viewer） |
| POST | `/knowledge-bases/:id/tags` | 建标签（OwnedKBOrAdmin） |
| PUT | `/knowledge-bases/:id/tags/:tag_id` | 改名字/颜色/排序（部分更新） |
| DELETE | `/knowledge-bases/:id/tags/:tag_id` | 删除（分层语义，见下） |

几个设计点：

1. **建标签防重**：`CreateTag` 先 `GetByName` 查同名，命中返回 409「标签名称已存在」；`name == "未分类"` 时强制 `SortOrder = -1`，让「未分类」永远排最前。
2. **删除的三级语义**（`DeleteTag` 的 `force` / `contentOnly` / `excludeIDs`）：
   - **默认**（无 force、有引用）→ 400「标签仍有知识或FAQ条目引用，无法删除」，防误删。
   - **`contentOnly=true`** → 只删标签下的内容，**保留标签本体**。
   - **`force=true`** → 先删内容再删标签。
   - **`excludeIDs`** → 排除掉不删的 chunk（seq_id），用于「从一批条目上摘掉这个标签、但保留其余」的场景。
   - 文档 KB 删内容 = 入队 `KnowledgeListDelete` 异步删整篇文档；FAQ KB 删内容 = `DeleteChunksByTagID` + 入队 `IndexDelete` 异步清向量索引。
3. **清理孤儿标签**：`DeleteUnusedTags` 批量删掉「既没有文档引用、也没有 chunk 引用」的标签。
4. **审计**：建/改/删都写 `AuditActionTagCreated/TagUpdated/TagDeleted`（含 name、force 等上下文）。

---

## 6. FAQ 标签：导入解析与两级优先检索

### 6.1 导入时把「标签名/标签 ID」解析成 UUID

`resolveTagID`（`knowledge_faq.go:1947`）的三级回退，是 FAQ 打标的关键：

```text
payload.TagID != 0   → GetBySeqID(seq_id) 拿回 UUID（精确指定已有标签）
payload.TagName != "" → FindOrCreateTagByName(name)（按名找，没有就建）
两者皆无             → FindOrCreateTagByName("未分类")（默认桶）
```

也就是说：FAQ 导入时可以**用整数 seq_id 精确指定**、**用标签名模糊匹配**，什么都不给就**自动落到「未分类」**这个内置默认标签。`UntaggedTagName = "未分类"`（`types/faq.go:385`）。

### 6.2 两级优先标签检索

FAQ 检索接口 `FAQSearchRequest`（`types/faq.go:376`）支持**两级标签优先级**：

```go
FirstPriorityTagIDs  []int64  // 一级优先标签，命中排最前
SecondPriorityTagIDs []int64  // 二级优先标签，命中次之
```

效果（`website-docs/03-features/17-faq.md:188`）：`FirstPriorityTagIDs` 命中的条目排最前，其次 `SecondPriorityTagIDs`。seq_id 在这里被 `GetBySeqIDs` 批量翻译回 UUID 参与过滤。

### 6.3 按标签批量改条目

`FAQEntryFieldsBatchUpdate.ByTag`（`types/faq.go:400`）支持「把这个标签下所有条目统一改成某字段」，key 是标签的 seq_id。批量摘标签时配 `ExcludeIDs` 排除指定条目。

---

## 7. 文档标签：手动打标与自动打标

文档侧有两种打标方式，都落在 `knowledge_tag_relations`。

### 7.1 手动打标（覆盖式）

`UpdateKnowledgeTag` / `UpdateKnowledgeTagBatch`（`knowledge.go:910,926`）→ `SetKnowledgeTags`（`knowledge_tag.go:15`）：

```text
事务内：先 DELETE 该 knowledge 的所有旧关联 → 再 INSERT 新关联（去空、去重）
```

是**整体覆盖**而非增量追加——用户在前端选中了哪些标签，就是最终结果。`UpdateKnowledgeTagBatch` 还会校验「所有 knowledge 都属于授权 KB」，防止越权改别的 KB 的文档。

### 7.2 自动打标（追加式，asynq 异步）

`KnowledgeAutoTagService`（`knowledge_auto_tag.go`），任务是 `TypeKnowledgeAutoTag`（`types/task.go:249`），在文档解析后处理阶段入队（`knowledge_post_process.go:492`，best-effort，失败不影响解析 finalize）。

流程：

```text
1. 读 AutoTagConfig（未启用/非 document KB → skip）
2. 列出 KB 现有标签（上限 maximumAutoTagCandidates=500，超出取稳定前缀并告警）
3. 若文档已有标签且 SkipIfTagged → skip（不浪费一次 LLM，不稀释人工分类）
4. 拼文档内容（文件名 + 摘要 + 文本/OCR/caption chunk，截断 16000 字）
5. classifyExistingTags：候选标签编号 1..N，让模型只回 {"matches":[{"index":1,"confidence":0.9}]}
6. validateAutoTagMatches：按 confidence 降序、去重、过滤 <0.75、限 max_tags
7. AddKnowledgeTagRelations：增量追加（ON CONFLICT DO NOTHING），不覆盖已有标签
```

三个精妙点（都写在代码注释里）：

1. **模型只回「序数」不回 UUID**：候选标签编号 `1. 产品 2. 技术…`，模型只需回 `index`，不用复现 UUID——既缩小 prompt（`maximumAutoTagCandidates=500` 也只是 500 行文字），又避免模型「悄悄篡改 UUID」。
2. **confidence 用指针**：模型常「忘了填 confidence 但明明选了」，`*float64` 区分「省略」和「显式 0」；省略视为 1（模型主动选了就该保留），0~100 的百分数自动 `/100` 归一到 0~1。
3. **prompt 语言中性**：只让模型回 `index + confidence`，**刻意不用** `{{language}}` 占位符——因为输出只有数字和序数，不涉及自然语言。

---

## 8. 标签在检索与问题推荐中的使用

### 8.1 检索：物理过滤与逻辑 scope 分离

`SearchTarget`（`types/search.go:26`）里三个字段各司其职：

| 字段 | 语义 | 落地 |
| --- | --- | --- |
| `TagIDs` | **物理索引过滤** | 直接进 `SearchParams.TagIDs` → 向量库 `tag_id` `ContainsAny`（FAQ chunk） |
| `KnowledgeIDs` | 文档标签解出的知识 ID | 过滤 `knowledge_id` 字段（document chunk） |
| `ScopeTagIDs` | **逻辑 scope**（用户点了哪些标签） | 仅作追踪/日志，检索完成后保留「用户选的是标签 X」这个事实 |

配套的 `DisableRecallThresholds`（`search.go:46`）：当检索范围是**用户显式选中的标签 scope** 时，把召回阈值临时归零，避免「阈值把整个显式 scope 过滤光、还没到重排就空了」——重排仍然排序，但向量/关键词阈值不能**抹掉整个 scope**。

在 `chat_pipeline/search.go:447`：只有「完整 KB 且无 TagIDs」的目标才合并成一次检索；带 TagIDs 或 KnowledgeIDs 的目标**逐个走 `searchSingleTarget`**（各自独立的过滤条件，不能合并）。

### 8.2 问题推荐：标签 scope → 具体来源

`resolveSuggestionTagScopes`（`custom_agent.go:835`）是「标签 scope 落地为可检索来源」的翻译器：

```text
输入: []TagScope{ {KnowledgeBaseID, TagIDs} }
1. 按 KB 分组，跨租户共享 KB 归到源租户
2. GetByIDs 校验标签确实属于该 KB（防越权/防脏 ID）
3. 文档标签 → ListIDsByTagIDs 解出 KnowledgeIDs（喂给文档来源采集）
4. FAQ 标签 → TagIDsByTenant（喂给 ListRecommendedFAQChunks 的 chunk 过滤）
输出: { KnowledgeBaseIDs, KnowledgeIDs, TagIDsByTenant }
```

于是开场推荐（Starters）里，用户「只看某标签下的 FAQ」就能精确落到「这个标签的 chunk」；「只看某标签下的文档」就落到「这些文档预生成的问题」。权限侧 `AuthorizeTenantAPIKeyOptionalTagIDs` 保证受限 API Key 不能越权传 tag ID。

---

## 9. 关键设计原则（可移植）

1. **作用域下沉到 KB + 租户，不做全局标签**：标签名只在一个 KB 内唯一，跨库隔离，天然避免「全局标签命名空间打架」。所有读写都带 `tenant_id` 过滤。
2. **同一概念、两种存储**：按「多对多（文档）vs 单值（FAQ）」的语义选择存储，检索时用「翻译成 knowledge_id」vs「索引字段过滤」两套落地。不为统一而强行统一。
3. **双标识（seq_id + UUID）**：对外用人类可读整数，对内用稳定 UUID；路径参数宽容解析两种 ID。
4. **删除语义分层**：默认防误删（有引用即拒绝）→ `contentOnly` 只清内容 → `force` 连内容带标签 → `excludeIDs` 精细摘除。把「危险操作」拆成显式的、可组合的开关。
5. **自动打标「存量标签分类」而非「自由生成标签」**：模型只从已有标签里选，`never invent a tag`，避免标签爆炸和失控。
6. **序数代替 UUID + 置信度下界 + skip_if_tagged**：三招让 LLM 分类「省 token、可过滤、不污染人工结果」。
7. **物理过滤与逻辑 scope 分离**：`TagIDs` 干检索的活，`ScopeTagIDs` 干可观测性的活；显式 scope 配 `DisableRecallThresholds` 防「阈值吃掉整个 scope」。
8. **批量计数与批量同步**：统计用 2 条 SQL 代替 2×N，FAQ 换标签用 `BatchUpdateChunkTagID` 批量写索引，避免 N+1。

---

## 10. 关键文件索引

| 关注点 | 文件 |
| --- | --- |
| 标签/关联/统计/自动打标配置模型 | `internal/types/tag.go`、`internal/types/knowledgebase.go:175` |
| TagScope / SearchTarget / SearchParams 的标签字段 | `internal/types/search.go:19,26,229` |
| chunk 单标签字段 | `internal/types/chunk.go:125` |
| FAQ 标签（UntaggedTagName / 两级优先） | `internal/types/faq.go:376,385` |
| 自动打标任务类型 + payload | `internal/types/task.go:249,530` |
| 服务/仓储接口契约 | `internal/types/interfaces/tag.go` |
| 标签仓储（CRUD/计数/删孤儿） | `internal/application/repository/tag.go` |
| 文档-标签关联（覆盖式/增量式） | `internal/application/repository/knowledge_tag.go` |
| tag → knowledgeID（OR 语义） | `internal/application/repository/knowledge.go:1027` |
| 标签服务（CRUD + 分层删除 + 索引删除任务） | `internal/application/service/tag.go` |
| 自动打标服务（分类 + 校验 + 追加） | `internal/application/service/knowledge_auto_tag.go` |
| 文档打标（手动覆盖/批量） | `internal/application/service/knowledge.go:886,910,926` |
| FAQ 打标（导入解析） | `internal/application/service/knowledge_faq.go:1947` |
| 自动打标入队点 | `internal/application/service/knowledge_post_process.go:492` |
| 数据源自动打标（按数据源名） | `internal/application/service/datasource_service.go:804` |
| 标签 scope → 检索/推荐来源翻译 | `internal/application/service/custom_agent.go:835` |
| 检索目标分组（含 TagIDs 拆分） | `internal/application/service/chat_pipeline/search.go:447,522` |
| 向量库 tag_id 过滤 | `internal/application/repository/retriever/weaviate/repository.go:497` |
| 索引 tag_id 批量同步 | `internal/application/service/retriever/*/BatchUpdateChunkTagID` |
| HTTP handler | `internal/handler/tag.go` |
| 路由注册 | `internal/router/routes_knowledge.go:264` |

---

## 附：标签系统全景图

```text
                          ┌─────────────────────────────────────────┐
                          │        标签本体 KnowledgeTag              │
                          │  (TenantID, KBID, Name唯一, Color, Sort)  │
                          └──────────────┬──────────────────────────┘
                                         │
              ┌──────────────────────────┼──────────────────────────┐
              │                          │                          │
        【手动/自动打标】              【FAQ 打标】               【检索/推荐】
              │                          │                          │
   文档: SetKnowledgeTags         FAQ: resolveTagID             SearchTarget.TagIDs
    (覆盖式, 关联表)               (seq_id→UUID)                (物理索引过滤)
   自动: KnowledgeAutoTagService    │ 缺省→"未分类"              SearchTarget.KnowledgeIDs
    (序数+置信度, 追加式)            │                            (文档标签翻译)
              │                   chunk.tag_id                 ScopeTagIDs (逻辑)
              │                          │                     DisableRecallThresholds
              ▼                          ▼                          │
   knowledge_tag_relations      chunks.tag_id (同步向量库)            ▼
   (KnowledgeID,TagID)           BatchUpdateChunkTagID          检索时过滤
              │                          │                    tag_id / knowledge_id
              └──────────┬───────────────┘                          │
                         ▼                                          │
                CountReferences / BatchCountReferences          问题推荐:
              (KnowledgeCount + ChunkCount 统计)            resolveSuggestionTagScopes
                                                              (文档→knowledgeIDs,
                                                               FAQ→chunk tag 过滤)
```
