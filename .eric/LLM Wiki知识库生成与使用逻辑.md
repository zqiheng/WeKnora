# WeKnora LLM Wiki：生成与使用逻辑

> 本文梳理 WeKnora 的 **LLM Wiki** 子系统——一个由 LLM 从原始文档中「读+写」出来的、持久化且可检索的**互联 Markdown 维基**。它和上一篇《知识图谱生成与使用逻辑》（Neo4j 实体-关系图谱）是**两套完全不同的东西**：Neo4j 图谱存的是结构化的「实体+关系」，而 LLM Wiki 存的是**有页、有目录、有双向链接、有修订历史的成段文章**。
>
> 核心思想：文档入库后不直接回答，而是让 LLM 先「消化」成一层**合成知识**（summary/entity/concept 等页面），这些页面之间用 `[[slug]]` 互链成网，检索时既作为普通 chunk 参与召回，又通过一组专用 Agent 工具被精准读取。
>
> 代码基准：`internal/`（Go 后端）。这条链路约 **16K 行**，是 WeKnora 里最复杂的离线子系统之一。开启方式：知识库配置里启用 Wiki（`WikiConfig`），文档入库后自动触发 ingest。

---

## 目录

1. [总体架构：从文档到互联维基](#1-总体架构从文档到互联维基)
2. [与 Neo4j 知识图谱的边界](#2-与-neo4j-知识图谱的边界)
3. [数据模型：WikiPage / WikiFolder / 修订](#3-数据模型wikipage--wikifolder--修订)
4. [触发与入队：Postgres 持久化队列 + asynq 去抖](#4-触发与入队postgres-持久化队列--asynq-去抖)
5. [生成管线：chunk-cited Map/Reduce](#5-生成管线chunk-cited-mapreduce)
6. [八个关键设计原则（可移植的核心）](#6-八个关键设计原则可移植的核心)
7. [检索与使用：重排加权 + Agent 工具](#7-检索与使用重排加权--agent-工具)
8. [并发与可靠性设计](#8-并发与可靠性设计)
9. [移植到自研知识库的落地建议](#9-移植到自研知识库的落地建议)
10. [关键文件索引](#10-关键文件索引)

---

## 1. 总体架构：从文档到互联维基

LLM Wiki 是一条**离线异步流水线**，把「一批文档」转换成「一批互联的 Wiki 页面」，再在检索侧消费这些页面。

```text
┌────────────────────────── 生成（离线异步，Map/Reduce） ──────────────────────────┐
│                                                                                  │
│  文档入库完成 → KnowledgePostProcess → EnqueueWikiIngest                          │
│     → 写 Postgres task_pending_ops（持久化待办，非 Redis list）                    │
│     → 去抖 asynq 触发 → ProcessWikiIngest（一批最多 5 篇文档）                     │
│          │                                                                       │
│          ├─ Map 阶段（每篇文档，并行）                                             │
│          │     Pass 0: 抽候选 slug（轻量骨架：name/slug/aliases/描述）             │
│          │     Pass 1..N: chunk 引用（哪些 chunk 实质讨论了该候选）                 │
│          │     └ 并行：生成 summary 页                                             │
│          │                                                                       │
│          ├─ Taxonomy 规划（整批一次 LLM 分配目录路径）                              │
│          │                                                                       │
│          └─ Reduce 阶段（每个 slug，并行 + per-slug 锁）                           │
│                编译器式 LLM：合并新增引用 / 撤回失效内容（逐字接地）                  │
│                                                                                  │
│     → Finalize（去抖 KB 全局收敛）：重建 index 页 + 清死链 + 注入交叉链接 + 剪空目录  │
└──────────────────────────────────────────────────────────────────────────────────┘

┌────────────────────────── 使用（在线检索） ──────────────────────────────────────┐
│  对话 query                                                                      │
│    ├─ 混合检索：wiki 页面作为 ChunkTypeWikiPage chunk 参与召回 → 重排后 ×1.3 加权   │
│    └─ Agent 工具：wiki_search / wiki_read_page / wiki_write_page / ...（精准读写） │
└──────────────────────────────────────────────────────────────────────────────────┘
```

一句话：**用 LLM 把「原始 chunk」蒸馏成「成体系的 wiki 页面」，让知识从「碎片」升级成「有结构、可导航、可被 LLM 直接读的二次文档」。**

---

## 2. 与 Neo4j 知识图谱的边界

两者都叫「知识」，但解决的问题不同，**不要混淆**（这也是上一篇文档容易让读者误以为「知识图谱」只有一个的原因）：

| 维度 | Neo4j 实体-关系图谱 | LLM Wiki |
| --- | --- | --- |
| 存储内容 | 实体 + 关系（图结构） | 成段 Markdown 页面 |
| 生成方式 | LLM 抽 `(实体, 关系)` 三元组 | LLM 逐 chunk 引用 + 编译器式合并 |
| 粒度 | 三元组级别 | 页面级别（含正文、摘要、别名、目录） |
| 组织 | 图遍历（一跳/二跳邻居） | 文件夹树 + `[[slug]]` 双向链接网 |
| 检索消费 | 查图谱补 chunk | 作为 chunk 召回（×1.3）+ 专用工具 |
| 版本/修订 | 无 | 有（50 条机器快照 / 200 硬上限） |
| 代码规模 | ~2K 行 | ~16K 行 |

结论：**Neo4j 图谱是「结构化补召回」，LLM Wiki 是「合成一篇可导航的百科」**。你在自研知识库里可以把二者独立看待、按需只移植其中一个。

---

## 3. 数据模型：WikiPage / WikiFolder / 修订

核心类型在 `internal/types/wiki_page.go`。

### 3.1 WikiPage（页面 = 一篇文章）

```go
type WikiPage struct {
    ID, TenantID, KnowledgeBaseID string
    Slug       string        // KB 内唯一；命名空间化：summary/<kid>、entity/…、concept/…
    Title      string        // 显示标题
    PageType   WikiPageType  // 见下
    Status     WikiPageStatus // draft / published / archived
    Content    string        // Markdown 正文（含 [[slug]] 链接）
    Summary    string        // 一句话摘要（summary 页正文第一行强制 "SUMMARY: …"）
    Aliases    StringArray   // 别名（用于 linkify 匹配）
    ParentSlug string        // 逻辑父页（≠ 文件夹）
    FolderID   string        // 归属目录（→ WikiFolder）
    CategoryPath StringArray // 面包屑缓存（FolderID 的派生，最大深度 3）
    WikiPath   string        // 拼接好的路径字符串（缓存）
    Depth      int
    SortOrder  int
    SourceRefs StringArray   // "uuid|title"，上游源文档
    ChunkRefs  StringArray   // 支撑本页内容的源 chunk ID（引用根基）
    InLinks    StringArray   // 指向本页的 slug 集合
    OutLinks   StringArray   // 本页指向的 slug 集合（Content 的派生，随写随解析）
    PageMetadata JSONMap     // 扩展字段
    Version    int           // 乐观锁 + 用户可见修订号
    LastEditSource WikiEditSource // pipeline/agent/user/revert
    LastEditorID   string
}
```

**页面类型 `WikiPageType`**：`summary`（每篇文档一份总览）、`entity`（实体）、`concept`（概念）、`index`（目录页）、`synthesis`（Agent 合成的综合页）、`comparison`（Agent 合成的对比页）。

**页面状态 `WikiPageStatus`**：`draft` → `published` → `archived`（ingest 生成时先 draft，整批收敛后 publish）。

### 3.2 WikiFolder（目录节点）

一等公民的目录树，邻接表 `ParentID`；`WikiPage.CategoryPath` 是它的面包屑缓存。目录用于把整批页面落到一棵**连贯的分类树**上（见 §5.3）。

### 3.3 WikiPageRevision（不可变快照）

每次「用户可见内容变更」（title/content/summary/page_type/status/aliases 任一变化）才 `version+1` 并落一条快照；纯 bookkeeping（刷新 source_refs、交叉链接注入没替换任何东西）只走 `UpdateMeta` 不 bump 版本。保留策略：**机器作者快照 50 条、总硬上限 200 条**。`RevertPageToVersion` 支持回滚。

### 3.4 WikiConfig（知识库级配置）

```go
type WikiConfig struct {
    SynthesisModelID        string  // 用什么模型跑 ingest
    MaxPagesPerIngest       int
    ExtractionGranularity   WikiExtractionGranularity // focused/standard/exhaustive
    ContentInstructions     string  // 业务层对「正文」的补充指令
    ExtractionInstructions  string  // 业务层对「抽取」的补充指令
    IngestBatchSize / IngestMapParallel / IngestReduceParallel
    IngestMaxInflight       int     // 每 KB 并发上限（默认 4）
}
```

`ExtractionGranularity`（`focused/standard/exhaustive`）直接注入 Pass 0 的 prompt，控制候选抽取的**激进程度**——这是「抽取粒度」的单一旋钮。

---

## 4. 触发与入队：Postgres 持久化队列 + asynq 去抖

这是整个子系统**最值得抄的工程点之一**：它没有用 Redis list 做任务队列，而是用 **Postgres 表 `task_pending_ops` 当持久化队列**，asynq 只负责「去抖触发」。

```text
文档处理完成
   → KnowledgePostProcess.SetFinalizing(willSpawnWiki=true)   // 占位，防「处理完成」提前置位
   → EnqueueWikiIngest / EnqueueWikiRetract
        ├─ 写 task_pending_ops（op=ingest|retract, knowledge_id, payload, fail_count）
        └─ asynq 入队 TypeWikiIngest（ProcessIn(wikiIngestDelay=30s) 去抖）
   → Handle 分发 TypeWikiFinalize vs TypeWikiIngest
   → worker 触发 ProcessWikiIngest（一批）
```

关键常量（`wiki_ingest.go`）：

| 常量 | 值 | 含义 |
| --- | --- | --- |
| `wikiMaxDocsPerBatch` | 5 | 一批最多处理几篇文档 |
| `wikiIngestDelay` | 30s | 去抖：连续入库合并成一次触发 |
| `wikiMaxFailRetries` | 5 | 批内失败重试上限，超限进死信表 |
| `wikiClaimStaleAfter` | 90m | 行被 claim 后超过此时间视为「僵尸 claim」可被抢占 |
| `wikiSlugLockTTL` | 5m | per-slug 锁的 TTL |
| `wikiInflightDefault` | 4 | 每 KB 默认在途批次数 |
| `maxContentForWiki` | 32768 | 喂给 LLM 的单文档重构正文上限（32KB） |

**为什么不用 Redis list？** 因为「待办」必须**可靠**：文档入库是异步的，如果进程在「已入队但未处理」之间挂了，Redis list 里的任务就丢了。用 Postgres 行，重启后可以 `recover_pending_wiki_tasks.go` 恢复，失败可以 `fail_count` 计数、超限进 `task_dead_letters` 死信表。asynq 只当一个「定时器」用，真正的账本在 Postgres。

---

## 5. 生成管线：chunk-cited Map/Reduce

管线主体在 `wiki_ingest_batch.go`（`ProcessWikiIngest`）+ `wiki_ingest_cite.go` + `wiki_ingest_taxonomy.go` + `wiki_ingest_dedup.go`。

### 5.0 整体流程（`ProcessWikiIngest`）

```text
1. 模式检测：redis 模式（分布式） vs lite 模式（单机内存锁）
2. 在途并发上限预占（Redis sorted set / 内存）
3. claimPendingList：不相交地认领一批待办行（有序，防并发重读）
4. Map 阶段：errgroup + mapParallel，每篇文档 mapOneDocument
5. Taxonomy 规划：planBatchTaxonomy（整批一次 LLM）→ resolvePlannedFolders
6. Reduce 阶段：errgroup + reduceParallel，每个 slug withSlugLock + reduceSlugUpdates
7. 死链清理 / publish 草稿页 / 入队 finalize / 关 span
8. requeueFailedOps（失败重试）/ trimPendingList（删除已消费行）
```

### 5.1 Map 阶段（`mapOneDocument`，每篇文档）

这是「把一篇文档拆成一组更新指令」的地方：

```text
guard 删除竞态
  → 加载该文档 chunks
  → reconstructEnrichedContent：拼接文本 + 图片 OCR/说明（image_multimodal）
  → 截断到 32KB（maxContentForWiki）
  → hasSufficientTextContent 检查（内容太少直接跳过）
  → Pass 0: extractCandidateSlugs  —— 轻量骨架（name/slug/aliases/description/details 兜底）
        （失败则回退到旧式 extractEntitiesAndConceptsNoUpsert）
  → 并行两条 goroutine：
        ├─ build summary（WikiSummaryPrompt，正文首行必须 "SUMMARY: …"）
        └─ classifyChunkCitations（Pass 1..N，chunk 引用，见 5.2）
  → mergeCitationsIntoItems：把 chunk 引用合进候选
  → 新旧 slug 对账：retract（已不存在）/ retractStale（源文档被删）/ reparseOverlap
  → 产出 []SlugUpdate（Map 阶段的输出契约）
```

`SlugUpdate` 是 Map 与 Reduce 之间的**中间契约**：

```go
type SlugUpdate struct {
    Slug         string
    Type         string   // entity/concept/summary/retract/retractStale
    Item         *extractedItem
    DocTitle     string
    KnowledgeID  string
    SourceRef    string   // "uuid|title"
    Language     string
    SummaryBody  string   // summary 页正文
    SummaryLine  string   // 一句话摘要
    SourceChunks []string // 支撑该 item 的源 chunk ID（引用根基）
    DocSummary   string
}
```

### 5.2 chunk 引用管道（`wiki_ingest_cite.go`）——「逐字接地」的关键

这是全系统**最有价值的设计**：LLM 不凭空写正文，而是**先指出「哪些 chunk 实质讨论了它」，Reduce 阶段再把这些 chunk 逐字搬进来**。

```text
Pass 0（extractCandidateSlugs）：抽候选骨架
Pass 1..N（classifyChunkCitations）：
    splitChunksIntoCitationBatches  —— 用 modelcontext.HandleTable 把 chunk 映射成
                                        "c000"/"c001" 短句柄，分批（maxRunesPerCitationBatch=12000）
    WikiChunkCitationPrompt  —— 对每个候选 slug，列出实质讨论它的 chunk ID，
                                还允许补 new_slugs（遗漏的候选）
    errgroup + maxCitationBatchConcurrency=4 并行多批，合并结果
    （prompt 的块顺序固定，保证前缀缓存命中，见 prompts_wiki_test.go 的 StablePrefixAcrossBatches）
resolveCitedChunks → collectCitedChunkContent → mergeCitationsIntoItems
```

**为什么用句柄 `c000`？** 源 chunk ID 是 UUID（高熵），LLM 复制时容易插/删字符；换成短句柄后 LLM 只要照抄 `c000`，回来再反查真 ID。这与 §6.4 的 slug 句柄 `ref-1` 是同一套思想。

### 5.3 Taxonomy 规划（`wiki_ingest_taxonomy.go`）

整批页面要落到一棵连贯的目录树，用**单次 LLM 调用**给整批分配 `category_path`：

```text
planBatchTaxonomy：
    WikiTaxonomyPlanPrompt —— 一次给整批（>60 个时按 wikiTaxonomyPlanChunkSize=60 分块，
                              前面的目录「前馈」给后面，保证收敛到同一棵树）
    parseTaxonomyAssignments
resolvePlannedFolders：
    顺序创建目录（在 Reduce 之前串行建好，Reduce 并行阶段只做「赋 ID」不建目录，避免竞态）
    目录 > 60 个时 selectRelevantFolders 用 embedding 选最相关的 top3
```

Reduce 阶段只给「还没被人工归置（`FolderID==""`）」的页面赋目录，**不覆盖用户手动归类**。

### 5.4 Reduce 阶段（`reduceSlugUpdates`，每个 slug）

按 slug 聚合所有文档的 `SlugUpdate`，一次性决定该页面的最终形态：

```text
reduceSlugUpdates：
  filter 存活更新 → GetPageBySlug
  summary 页：直接覆盖正文
  entity/concept：构造三块上下文
      <new_information>      —— 逐字引用的 chunk 内容（grounding）
      <deleted_documents>    —— 本次被删的源文档
      <remaining_source_documents> —— 仍存活的源文档
    → WikiPageModifyUserPrompt（编译器式合并，见 §6.3）
  slugHandles 编解码（§6.4）→ applyFolderToPage（赋目录）
  mergeChunkRefs → CreatePage / UpdatePage
```

Reduce 的 LLM 是「**compiler not writer**」：它不重新创作，只做三件事——**保留**仍在源里的内容、**新增**新文档带来的信息、**撤回**随源文档一起消失的内容，且新增必须逐字来自 `SourceChunks`。

### 5.5 Finalize（去抖 KB 全局收敛，`ProcessWikiFinalize`）

Map/Reduce 是「每批」的，而以下工作要「整 KB 一次性」做，于是用第二条去抖队列（`wikiFinalizeTaskType`，延迟 `wikiFinalizeDelay=20s`，`asynq.TaskID` 按 KB 合并，最多 `wikiFinalizeMaxRows=5000` 行一次拉取）：

```text
1. 重建 index 页 intro（WikiIndexIntroUpdatePrompt，按 added/removed 变更描述增量更新）
2. 清理死链（rewriteDeadWikiLinks）
3. 注入交叉链接（InjectCrossLinks：给正文里的裸词自动补 [[slug]]）
4. 剪空目录链（PruneEmptyFolderChains）
```

Finalize 有**独立的 per-KB 锁**（`wiki:finalize:active:{kb}`，与 ingest 锁分离，互不阻塞），fail-closed：拿锁失败就报错让 asynq 重试，**绝不裸奔**（否则两个 finalize 会双重建 index 页产生 lost update）。

### 5.6 蒸馏用的是 Prompt，不是「直接提取」——全部 Prompt 模板清单

**先回答一个最常见的疑问：LLM Wiki 到底有没有用 prompt 提示词？答案是有，而且用得非常重。** 它**不是**「把原文档丢给模型无脑提取」，而是把原文档切成 chunk 作为**数据**喂进 prompt，再由 prompt 定义**「怎么结构化地抽、引、并」**。原文档仍是唯一事实来源，prompt 是约束与脚手架——尤其是一套防幻觉的「逐字接地」规则。

所有模板集中在一个文件里：`internal/agent/prompts_wiki.go`，共 **9 个 prompt 模板 + 3 档 granularity 指引**。下表按管线阶段一一对应：

| Prompt 常量 | 所在阶段 | 输入（数据块） | 输出 | 核心约束（防幻觉 / 防漂移） |
| --- | --- | --- | --- | --- |
| `WikiSummaryPrompt` | Map·summary | `<content>` 文档正文 + `<available_wiki_pages>` | `SUMMARY:` 首行 + Markdown 摘要 | **故意不喂文件名/标题**；空内容必须输出 "No textual content..." 不得脑补 |
| `WikiCandidateSlugPrompt` | Map·Pass 0 | `<content>` + `<previous_slugs>` + granularity | `entities`/`concepts` 轻量骨架 | slug 连续性；granularity 控制抽取激进程度；`details` 仅作兜底 |
| `WikiChunkCitationPrompt` | Map·Pass 1..N | `<candidate_slugs>` + `<chunks>`（短句柄 `c000`） | `citations` + `new_slugs` | 只准从 `<chunks>` 块里、用 `id` 原样照抄句柄 |
| `WikiTaxonomyPlanPrompt` | Taxonomy | `<existing_folders>` + `<items>` | `assignments`（每项一个 path） | 复用已有目录、**字符级一致**，不造同义词目录 |
| `WikiPageModifySystemPrompt` | Reduce | 纯规则（**无**页面身份/源数据） | 编译器式合并规则 | compiler not writer、逐字接地、无幻觉 |
| `WikiPageModifyUserPrompt` | Reduce | source_contexts + page_metadata + existing + new/deleted | `SUMMARY:` 首行 + 更新正文 | CRITICAL CONFLICT CHECK（新信息是否**真的**关于本页） |
| `WikiIndexIntroPrompt` | Finalize（首建） | `<document_summaries>` | 标题 + 简介段 | 不生成目录/页面链接 |
| `WikiIndexIntroUpdatePrompt` | Finalize（增量） | current_intro + changes + summaries | 更新后简介 | 保持原有语气/标题格式 |
| `WikiDeduplicationPrompt` | 去重 | `<items>`（每个自带 top-k candidates） | `merges` map | **related ≠ same**；硬约束（只能并进同 item 的候选、类型必须匹配） |

#### 逐条展开：几个 prompt 里「防幻觉」的硬设计

**① `WikiSummaryPrompt` —— 故意不喂文件名，杜绝「由文件名脑补」。**

> 见源码注释（第 54–59 行）：文件名和标题**刻意不传给 LLM**，因为用户上传的文档常带无语义文件名（如扫描件 `MX5280.pdf`），一旦喂进去，正文很薄时模型就会顺着文件名编摘要。

配套的硬规则：输出第一行必须是 `SUMMARY: {15–40 词一句话}`；第 82 行规定**空内容规则**——若 `<content>` 为空或只剩图片无文字，必须输出 `"SUMMARY: No textual content was extractable from this document."`，**明确禁止**「invent a topic / guess from any other clue」。

**② `WikiCandidateSlugPrompt` —— Pass 0 只抽「骨架」，granularity 是唯一旋钮。**

这个 prompt 明确告诉模型「后面还有一 pass 会补齐 chunk 引用，你**不需要**写详尽事实」——所以它很便宜，即使长文档也扛得住。`{{.Granularity}}` 会把知识库配置的 `ExtractionGranularity` 对应的一段指引注入进来（见下方 granularity 三档表）。`details` 字段被降级成「1–3 句、300 字符内」的兜底摘要，只在 chunk 引用失败时才用。

**③ `WikiChunkCitationPrompt` —— 引用阶段「只标 ID，不抄正文」。**

模型的任务是：对每个候选 slug，从 `<chunks>` 块里选出**实质讨论它**的 chunk ID（如 `c003`），用 `<c>` 元素的 `id` 属性**原样照抄**。正文的「事实」保持在 chunk 原文里，而不是让 LLM 转述。它还允许在 `new_slugs` 里补上漏掉的候选，但**禁止**重新发现已列出的 slug。这条 prompt 的「块顺序固定」是 §6.4 前缀缓存命中的前提（见下方「prompt 工程共性」）。

**④ `WikiPageModifySystemPrompt` + `WikiPageModifyUserPrompt` —— Reduce 的「编译器契约」。**

System prompt 只放**所有页面共享的规则**，刻意不放页面身份和源数据（这样整个 reduce 批次能共享一个长的 byte-stable 前缀，见 §5.4）。三条最关键：
- **Mandatory Grounding**：每一条新增事实断言/数值，都必须能直接由提供的 `new source chunks` 支撑；
- **No Hallucination**：不得杜撰/推断 chunk 里没有明说的内容；冲突时优先更新且加 "Contradictions / Updates" 段，模糊则只加段不改正文；
- **COMPILER, not a creative writer**：贴近源文字面，允许轻量重排/去重/连接，但**不得**为风格改写、扩写短句、或编造过渡。

User prompt 则装**每批每页的数据**，其中第 391 行的 **CRITICAL CONFLICT CHECK** 尤其值得注意：先校验 `<new_information>` 是否**真的关于本页**（例如本页是「Hunyuan Model」而新信息是「Qwen3」，或本页是「居民身份证」而新信息是「工作居住证」），不对就要**拒绝**这部分新增、不加进去。这是防「同名相近实体的信息被误合并到别人页」的关键闸门。

**⑤ `WikiTaxonomyPlanPrompt` —— 目录不并行发散，一次收敛。**

单次调用给整批（>60 项按 60 分块）分配 path，且要求「复用已有目录时**字符级一致**，不造同义词目录（不要已有『春节 / 传统习俗』再造『春节习俗』）」。目录是「事物本质的稳定书架」而非「它在某篇文档里的角色」。

**⑥ `WikiDeduplicationPrompt` —— 去重的「related ≠ same」。**

prompt 里塞了**十几条正反例**（正确合并：`Acme Corp → Acme Corporation`；错误合并：`Hunyuan Model → Qwen Model`、`居民身份证 → 工作居住证`、`Machine Learning → Neural Networks` 等），并把核心原则写成一句话：**「related ≠ same」**。硬约束：只能并进**同一个 item 内**列出的候选 page、类型必须匹配（entity 只能并 entity）；「不确定就不并」，宁可两个页面也不误并两个不同的东西。

#### granularity 三档指引（注入 `WikiCandidateSlugPrompt`）

`WikiGranularityGuidance()` 把 `WikiConfig.ExtractionGranularity` 映射成一段英文指引文本，直接控制 Pass 0 抽多少候选：

| 档位 | 取值 | 抽取范围 | 典型用途 |
| --- | --- | --- | --- |
| focused | `focused` | **激进剪枝**：只抽文档主干，entities+concepts 合计 **3–7 个**；排除技术栈、背景提及、一句带过的项 | 简历/公告/产品页等「要干净不要全」 |
| standard（默认） | `standard` | 主干 + **实质讨论**项（有独立段落/多条要点/2–3 句上下文）；排除「逗号列表里仅点名的技术」 | 通用文档，默认档 |
| exhaustive | `exhaustive` | **最大召回**：每个具名的实体/概念都抽（哪怕只提到一次），仅排除纯泛词（"database"/"function"） | 技术 glossary / 词典型知识库 |

三档是一个**单调递增**的谱系：档位越往下，候选 slug 数越多 → 下游 chunk 引用成本越高 → index 噪声比越大。`WikiConfig.ExtractionInstructions`（业务补充抽取指令）也会合入这条 prompt。

#### prompt 工程里的三个共性技巧（跨模板）

1. **Byte-stable 前缀缓存**：`WikiChunkCitationPrompt` 把「静态规则 + 候选 slug」放在「逐批变化的 `<chunks>`」**之前**，同文档内只有 `ChunksXML` 在批间变化，于是除第一批外的每批都能命中 provider 前缀缓存、不重复计费静态规则（§5.2 已提，源码第 255–259 行注释）。`WikiPageModifySystemPrompt` 也刻意把「共享规则」与「每页数据」拆成 system/user 两条消息，同理。
2. **句柄间接层**：高熵 ID（UUID chunk ID、`summary/<uuid>` slug）LLM 极易抄错，于是用 `modelcontext.HandleTable` 把 chunk 映射成 `c000`、把 slug 映射成 `ref-1` 短句柄，进 prompt 前 `encode`、出 prompt 后 `decode`（§6.4）。
3. **JSON 格式硬规则**：每个要返回 JSON 的 prompt 都重复一条 `Do NOT use literal newline inside JSON string values`，并 `Output ONLY valid JSON, no preamble`——把「模型输出结构松垮」这类问题在 prompt 层就压掉，配合 `prompts_wiki_test.go` 里的 `StablePrefixAcrossBatches` 保证块顺序稳定。

---

## 6. 八个关键设计原则（可移植的核心）

这一节是给你「搬到自己的知识库」准备的——剥掉 WeKnora 的 Go/asynq/Postgres 细节，剩下的这些**设计原则**才是真正值钱的部分。

### 6.1 slug 稳定性 / 连续性（Slug 是页面的身份证）

页面用 `Slug`（KB 内唯一、命名空间化）做身份，而不是用自增 ID。文档更新时**复用上一版的 slug**（通过 `PreviousSlugs` 对账），这样：
- 页面 URL、其他页面的 `[[slug]]` 链接、外部引用都不会断；
- 增量更新才可能——新文档「加到」已有实体页，而不是每次重建一个新页。

**移植要点**：给每个实体/概念一个**稳定的、语义化的 key**（不是随机 UUID），更新时按 key 合并而非新建。

### 6.2 chunk-cited 逐字接地（防幻觉的根基）

正文不是 LLM 凭空发挥，而是**先让 LLM 标出「哪些源 chunk 讨论了我」，再把那些 chunk 逐字喂给 Reduce**。这样：
- 新增内容的每个字都有 `ChunkRefs` 溯源；
- 幻觉被物理上限制在「chunk 里没有的东西写不进来」；
- 文档删除时能**精确撤回**（retract）对应内容，而不是整页重建。

**移植要点**：把「生成」拆成「引用（cite）+ 编译（compile）」两步；正文永远携带 chunk 引用列表。

### 6.3 编译器式 Reduce（compiler not writer）

Reduce 的 prompt（`WikiPageModifySystemPrompt`）明确要求模型扮演**编译器**：输出结构化的 `HasAdditions`/`HasRetractions` + `valid_wiki_links`，做的是「合并增量」，而不是「重写全文」。这让**多篇文档的更新可以安全地幂等合并**，也让「源文档被删 → 对应内容撤回」成为可能。

### 6.4 slug 句柄间接层（`ref-1`/`ref-2`）

高熵 slug（尤其 `summary/<uuid>`）LLM 极易复制错（插/删十六进制位）。于是 `wiki_slug_handles.go` 用 `modelcontext.HandleTable` 把真 slug 映射成 `ref-1`、`ref-2` 短句柄，进 prompt 前 `encodeContent` 换成句柄，出 prompt 后 `decodeContent` 换回真 slug。**从源头消除「复制错」的机会**，而不是事后修补（事后修补对应 `RepairContentLinks`，是兜底）。

### 6.5 pg_trgm + Jaccard 预筛 + LLM 硬去重

去重分两级（`wiki_ingest_dedup.go`）：
- **粗筛**：`pg_trgm` 三元组相似度 + Jaccard，挑 top5（`dedupCandidateTopK=5`，`dedupCandidateScoreFloor=0.08`，小语料 <25 页直接跳过）；
- **精判**：LLM `WikiDeduplicationPrompt` 做「严格合并判定」——**相关 ≠ 相同**（related ≠ same），只有真重复才合并，且有 `dedupMergeRejectReason` 的硬校验（类型不匹配、缺前缀、非候选）。

**移植要点**：去重别全靠 LLM（贵且慢），先用廉价相似度收窄候选，LLM 只做最后裁决。

### 6.6 双向链接（OutLinks / InLinks）

`Content` 里的 `[[slug|显示名]]` 是唯一真相源；`OutLinks` 每次写时从正文重新 `parseOutLinks` 解析（派生字段，永不手工维护）；`InLinks` 由 `updateInLinks`/`removeInLinks` 在目标页上反向维护。于是：
- 改名/删除时，反向链接能自动重写（`wiki_link_mutation.go` / `rename` 工具）；
- 死链能被 `cleanDeadLinks` 发现并改写。

**移植要点**：链接只存一份（正文），正反两个索引都是**派生的、随写随重建**的，不要试图手工同步两份。

### 6.7 版本号语义：只有「用户可见变化」才 bump

`version` 字段的语义是「用户可见内容修订」，不是「每次行更新」。bookkeeping（刷 source_refs、交叉链接注入没换任何东西、index 页同目录重建）走 `UpdateMeta` **不 bump 版本**。这让消费方能放心地把 `version` 变化当作「真的编辑信号」。配套的是不可变快照 `WikiPageRevision`（50 机器 / 200 硬上限）+ 回滚。

### 6.8 内容净化：图片 URL 掩码 + 行内引用剥离

- **图片 URL 掩码**：正文里的图片 URL 在进/出 LLM 时替换成不透明 token，防止模型复写 URL 出错、也防把对象存储 URL 泄露进 prompt；
- **行内 chunk 引用剥离**（`stripWikiInlineChunkCitations`）：citation 阶段产生的 `[c003]` 短句柄只对机器有意义，写库前用正则 `[ \t]*\[c\d{3,}([;,]\s*c\d{3,})*\]` 从正文和摘要里剥掉，**不泄露到读者可见的 Markdown**。

---

## 7. 检索与使用：重排加权 + Agent 工具

### 7.1 重排加权（`chat_pipeline/wiki_boost.go`）

`PluginWikiBoost` 挂在 `CHUNK_RERANK` 阶段：把 `ChunkType == ChunkTypeWikiPage` 的 chunk 分数 × **1.3**（`wikiBoostFactor`），再 `SliceStable` 重排。

```go
if chatManage.RerankResult[i].ChunkType == types.ChunkTypeWikiPage {
    chatManage.RerankResult[i].Score *= 1.3
}
```

设计意图：**Wiki 页面是「LLM 合成、交叉引用过」的知识，比原始文档 chunk 更连贯，应当被优先**。快速路径：先扫一遍结果集有没有 wiki chunk，没有就直接 return，避免每个非 wiki 查询都去 hit KB 服务。

### 7.2 Agent 工具（`internal/agent/tools/wiki_*.go`）

Wiki 的读写是通过一组**专用 Agent 工具**暴露的（能力由 `KBCapability CapWiki` 门控，`capabilities.go`）：

| 工具 | 作用 |
| --- | --- |
| `wiki_read_page` | 按 slug 读页面，渲染成 `<wiki_page>` XML（元数据/关系/来源/摘要/正文），带输出预算截断 |
| `wiki_search` | Postgres POSIX 正则 `~*` 搜索，返回 `<search_results>` |
| `wiki_read_source_doc` | 精读某个上游源文档 |
| `wiki_write_page` | 创建/整页覆盖（带 slug 校验，`RepairContentLinks` 修 LLM 弄坏的 slug） |
| `wiki_replace_text` | 局部替换正文（精确匹配，找不到不落库） |
| `wiki_rename_page` / `wiki_delete_page` | 重命名（自动更新关联链接）/ 删除（自动清理死链） |
| `wiki_flag_issue` / `wiki_read_issue` / `wiki_update_issue` | 标记/查看/更新「事实错误或合并冲突」问题 |
| `wiki_link_mutation` | 机器维护的链接改写（不 bump 用户可见版本） |

配套的**权限**机制（`WikiScope` + `pagePassesWikiScope`）做「源文档/标签范围」校验，`WikiRouteResolver` 做 slug→KB 路由——多 KB 场景下 Agent 不会读错库。

### 7.3 一个需要留意的现状：chunk 同步「创建侧」未接线

检索加权（`wiki_boost.go`）和删除侧（`wiki_page.go:deleteChunkForPage`，chunk ID 约定 `"wp-" + page.ID`）都已接线，`ChunkTypeWikiPage`（`"wiki_page"`）类型也已定义。但我在当前代码快照里**没找到对称的「创建侧」**——即「页面 Create/Update 时把正文落成一个 `wiki_page` 类型的 chunk」这一步没有调用点（grep `ChunkTypeWikiPage` 只出现在类型定义、boost 两处、测试；`wp-` 只出现在 deleteChunkForPage）。

含义：**Wiki 页面目前主要通过 §7.2 的专用 Agent 工具被消费**（`wiki_search`/`wiki_read_page` 直接走 Postgres，不依赖向量索引）；「作为普通 chunk 参与混合检索并 ×1.3」这条路径的**创建侧半成品**。你移植时如果想走「wiki 页也进向量库」这条路，记得补上创建侧（见 §9 建议 6）。

---

## 8. 并发与可靠性设计

| 机制 | 实现 | 解决什么问题 |
| --- | --- | --- |
| 持久化待办 | Postgres `task_pending_ops` 表（非 Redis list） | 进程挂了任务不丢，重启可恢复 |
| 不相交认领 | `claimPendingList`（按 id ASC 有序 claim） | 多 worker 并发不重复读同一行 |
| per-slug 锁 | `withSlugLock`（Redis/内存，`wikiSlugLockTTL=5m`） | 两个 batch 同时改同一 slug 不打架 |
| per-KB 在途上限 | Redis sorted set 计数（`wikiInflightDefault=4`） | 一个 KB 不会被海量文档打爆 |
| 去抖触发 | asynq `ProcessIn(wikiIngestDelay=30s)` + `TaskID` 合并 | 连续入库合并成一次处理 |
| 失败重试 | `fail_count` 计数，≤5 留在队里下次重试，>5 进死信表 | 偶发 LLM 失败不丢任务 |
| 僵尸 claim 抢占 | `wikiClaimStaleAfter=90m` | worker 挂掉后行不会被永久锁死 |
| finalize 锁 | `wiki:finalize:active:{kb}`，fail-closed | 双跑 finalize 导致 index 页 lost update |
| 乐观锁 | `wiki_pages.version` + `WHERE id=? AND version=?` | 并发写页面不覆盖 |
| 死信审计 | `task_dead_letters` 表 | 终态失败可查可恢复 |
| Lite/Redis 双模式 | `redisClient != nil` 分支，内存锁降级 | 单机部署无需 Redis |

---

## 9. 移植到自研知识库的落地建议

如果你要把这套设计搬进自己的知识库，建议**从最小可行版开始，按下面的顺序加**：

1. **先定义身份模型**：每个可增量更新的「知识对象」有一个稳定 key（对应 slug）+ 类型（entity/concept/summary）+ 状态（draft/published/archived）+ `version`。别用随机 ID 当身份。

2. **先做「引用 + 编译」两段式，跳过 taxonomy/去重**：
   - 引用段：对每个候选，让 LLM 标出「哪些 chunk 讨论它」（这是接地根基，§6.2）；
   - 编译段：让 LLM 做编译器式合并（§6.3），新增内容逐字来自引用 chunk；
   - 这一条做好了，幻觉和「删文档撤不掉内容」两大痛点就解决了一大半。

3. **slug 句柄间接层**（§6.4）几乎是免费的保险，**尽早加**——它只花一个映射表，却能把「LLM 抄错 ID」这类最恶心的问题归零。

4. **去重用「廉价预筛 + LLM 裁决」**（§6.5）：先 `pg_trgm`/余弦/Jaccard 收窄，LLM 只判 top-k 候选。别一上来全量 LLM 比两两。

5. **链接存一份、索引派生**（§6.6）：正文是唯一真相，正反链随写重建。这让你以后加「改名自动更新」「死链清理」几乎零成本。

6. **检索侧二选一，不必都做**：
   - 轻量：走「专用读工具」（对应 `wiki_search`/`wiki_read_page`，直查 DB/向量库，不看 chunk 管道）；
   - 重量：把 wiki 页也落成普通 chunk 进混合检索 + 重排加权（对应 `wiki_boost` ×1.3，记得补创建侧）。

7. **可靠性基建**（§8）按需选：单机先跳过分布式锁/死信；但**「待办持久化到 DB 而非内存队列」**这条哪怕单机也值得做——异步流水线最怕「入队后进程挂」丢任务。

---

## 10. 关键文件索引

| 关注点 | 文件 |
| --- | --- |
| 数据模型（WikiPage/Folder/Revision/Config） | `internal/types/wiki_page.go` |
| 服务/仓储接口契约 | `internal/types/interfaces/wiki_page.go` |
| 全部 prompt 模板（设计契约） | `internal/agent/prompts_wiki.go` |
| 入队/触发/常量/分发 | `internal/application/service/wiki_ingest.go` |
| Map/Reduce 主体 + Finalize | `internal/application/service/wiki_ingest_batch.go` |
| chunk 引用管道（Pass 0/1..N） | `internal/application/service/wiki_ingest_cite.go` |
| 去重（pg_trgm + LLM 硬校验） | `internal/application/service/wiki_ingest_dedup.go` |
| 目录规划（单次 LLM + embedding 选目录） | `internal/application/service/wiki_ingest_taxonomy.go` |
| 页面 CRUD + 链接维护 + chunk 删除 | `internal/application/service/wiki_page.go` |
| 正文 `[[slug]]` 链接触发（word-boundary 安全） | `internal/application/service/wiki_linkify.go` |
| slug 句柄 `ref-1` 间接层 | `internal/application/service/wiki_slug_handles.go` |
| 健康检查 / lint（orphan/broken link/stale ref） | `internal/application/service/wiki_lint.go` |
| 检索重排加权（×1.3） | `internal/application/service/chat_pipeline/wiki_boost.go` |
| Agent 读写工具 | `internal/agent/tools/wiki_tools.go` 及 `wiki_*.go` |
| HTTP handler | `internal/handler/wiki_page.go` |
| 重启后待办恢复 | `internal/container/recover_pending_wiki_tasks.go` |

---

## 附：一条文档从入库到 Wiki 可查的时序

```text
文档入库完成
   │
   ├─ KnowledgePostProcess.SetFinalizing(willSpawnWiki=true)   // 占位
   ├─ EnqueueWikiIngest → task_pending_ops 插入一行 (op=ingest)
   ├─ asynq ProcessIn(30s) 去抖触发
   │
   ▼
ProcessWikiIngest（一批 ≤5 篇）
   ├─ claimPendingList 认领待办
   ├─ Map: 每篇文档
   │     Pass 0 抽候选 → Pass 1..N chunk 引用 + 并行生成 summary
   │     → 新旧 slug 对账 → []SlugUpdate
   ├─ Taxonomy: 整批一次 LLM 分配目录 → 顺序建目录
   ├─ Reduce: 每 slug 编译器式合并（逐字接地）→ Create/Update 页面
   ├─ 死链清理 → publish 草稿 → enqueueFinalize
   └─ trimPendingList 删已消费行
   │
   ▼ (去抖 20s)
ProcessWikiFinalize（整 KB 一次）
   ├─ 重建 index intro（增量）
   ├─ InjectCrossLinks 补交叉链接
   ├─ 清死链
   └─ 剪空目录链
   │
   ▼
可查：Agent 工具 wiki_search/wiki_read_page（直查 Postgres）
        （或：wiki 页作为 chunk 参与混合检索 → 重排 ×1.3）
```
