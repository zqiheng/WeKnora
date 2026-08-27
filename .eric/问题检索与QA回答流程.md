# WeKnora 问题检索与 QA 回答完整流程

> 本文梳理 WeKnora 从「用户提出问题」到「流式返回答案」的端到端链路，覆盖两条主要路径：
>
> 1. **RAG 快速问答**（`KnowledgeQA`）：固定流水线，检索 + 重排 + 拼接上下文 + LLM 生成。
> 2. **ReAct Agent 问答**（`AgentQA`）：LLM 自主编排，循环调用工具（检索 / 图谱 / 网页等）。
>
> 代码基准：`internal/`（Go 后端）。所有路径均为仓库内相对路径。

---

## 目录

1. [总体架构](#1-总体架构)
2. [入口层：HTTP 请求 → 服务层](#2-入口层http-请求--服务层)
3. [RAG 流水线：整体阶段](#3-rag-流水线整体阶段)
4. [问题检索：混合检索的深入细节](#4-问题检索混合检索的深入细节)
5. [重排序（Rerank）](#5-重排序rerank)
6. [分块合并（Merge）](#6-分块合并merge)
7. [上下文组装与流式生成](#7-上下文组装与流式生成)
8. [ReAct Agent 问答路径](#8-react-agent-问答路径)
9. [关键数据结构](#9-关键数据结构)
10. [关键文件索引](#10-关键文件索引)

---

## 1. 总体架构

WeKnora 的问答能力由一个**事件驱动的插件流水线（Plugin Pipeline）** 承载。核心概念：

- **`EventType`**（`internal/types/chat_manage.go`）：流水线的阶段枚举，如 `QUERY_UNDERSTAND`、`CHUNK_SEARCH_PARALLEL`、`CHUNK_RERANK`、`CHUNK_MERGE`、`CHAT_COMPLETION_STREAM` 等。
- **`Plugin` 接口**（`internal/application/service/chat_pipeline/chat_pipeline.go`）：每个插件通过 `ActivationEvents()`声明自己处理哪些阶段，通过`OnEvent(ctx, eventType, chatManage, next)` 执行逻辑。
- **`EventManager`**：负责注册插件并按阶段分发事件；同一阶段的多个插件构成责任链（`next()` 调用链）。
- **`ChatManage`**（`internal/types/chat_manage.go`）：贯穿整个流水线的上下文对象，内嵌三部分：
  - `PipelineRequest` —— 不可变请求配置（知识库、阈值、模型 ID 等）；
  - `PipelineState` —— 各插件读写的中转状态（`RewriteQuery`、`SearchResult`、`RerankResult`、`MergeResult`、`UserContent` 等）；
  - `PipelineContext` —— 运行时句柄（`EventBus`、消息 ID 等）。

```text
用户问题
   │
   ▼
HTTP Handler (session/qa.go)
   │  KnowledgeQA (RAG)   /   AgentQA (ReAct)
   ▼
Service 层 (sessionService)
   │  组装 ChatManage + 构建流水线阶段列表
   ▼
EventManager.Trigger()  ──按序触发各阶段插件──►  最终通过 EventBus 流式回传答案
```

---

## 2. 入口层：HTTP 请求 → 服务层

入口文件：`internal/handler/session/qa.go`

- **`KnowledgeQA(c)`**（`qa.go:802`）—— RAG 快速问答入口，走 `executeQA(reqCtx, qaModeNormal, ...)`。
- **`AgentQA(c)`**（`qa.go:828`）—— ReAct Agent 问答入口，走 `executeQA(reqCtx, qaModeAgent, ...)`。
- **`SearchKnowledge(c)`**（`qa.go:701`）—— **纯检索 API**，只做召回、不生成答案（用于「仅检索」场景，如知识库检索预览）。
- **`executeQA(reqCtx, mode, generateTitle)`**（`qa.go:883`）—— 统一分发：
  - 解析请求（`parseQARequest`），处理 base64 内联图片、文件附件、临时附件；
  - 按 `qaMode`调用服务层的`KnowledgeQA`或`AgentQA`；
  - 建立 SSE 流（`setupSSEStream`）、停止事件监听、标题生成等。

服务层入口：

- **`sessionService.KnowledgeQA(ctx, req, eventBus)`**（`internal/application/service/session_knowledge_qa.go:23`）—— RAG 路径。
- **`sessionService.AgentQA(ctx, req, eventBus)`**（`internal/application/service/session_agent_qa.go:20`）—— Agent 路径。

---

## 3. RAG 流水线：整体阶段

`KnowledgeQA` 内部的装配过程（`session_knowledge_qa.go:23`）：

1. **解析知识库** `resolveKnowledgeBases`→ 得到`knowledgeBaseIDs`、`knowledgeIDs`。
2. **解析聊天模型** `resolveChatModelID`。
3. **构建检索目标** `buildSearchTargets`→`SearchTargets`（一次计算，贯穿全流水线；支持 KB 级 / 文档级 / 标签级作用域）。
4. **初始化 `ChatManage`**：从 `config.yaml` 读默认阈值（`VectorThreshold`、`KeywordThreshold`、`EmbeddingTopK`、`RerankTopK`、`RerankThreshold` 等），叠加自定义 Agent 的覆盖（`applyAgentOverridesToChatManage`）。
5. **动态组装流水线**（`PipelineBuilder`）：

```go
// RAG 路径（needsRAG = 有 KB 检索作用域 或 开启网页搜索）
pipeline = types.NewPipelineBuilder().
    AddIf(hasHistory, LOAD_HISTORY).        // 1. 加载历史（多轮时）
    Add(MEMORY_RECALL).                     // 2. 长期记忆召回
    Add(QUERY_UNDERSTAND).                  // 3. 查询改写 + 意图分类
    Add(CHUNK_SEARCH_PARALLEL).             // 4. 并行混合检索
    Add(CHUNK_RERANK).                      // 5. 重排序
    AddIf(webSearchEnabled, WEB_FETCH).     // 6. 网页抓取（可选）
    Add(CHUNK_MERGE).                       // 7. 分块合并
    Add(FILTER_TOP_K).                      // 8. Top-K 截断
    AddIf(dataAnalysisEnabled, DATA_ANALYSIS). // 9. 表格数据分析（可选）
    Add(INTO_CHAT_MESSAGE).                 // 10. 组装上下文 prompt
    Add(CHAT_COMPLETION_STREAM).            // 11. 流式 LLM 生成
    Build()
```

1. **触发流水线** `KnowledgeQAByEvent(ctx, chatManage, pipeline)`（`session_knowledge_qa.go:670`），内部逐个 `eventManager.Trigger(stageCtx, eventType, chatManage)`。

各阶段插件及其职责：

| 阶段 | 插件文件 | 职责 |
| --- | --- | --- |
| `LOAD_HISTORY` | `load_history.go` | 加载会话历史消息 |
| `MEMORY_RECALL` | `memory_recall.go`/`memory_affinity.go` | 长期记忆召回（按相关性），生成记忆上下文 |
| `QUERY_UNDERSTAND` | `query_understand.go` | LLM 改写查询 + 意图分类（`kb_search`/`web_search`/`greeting`/…） |
| `CHUNK_SEARCH_PARALLEL` | `search.go`/`search_parallel.go` | 向量 + 关键词混合检索（可并行跨 KB） |
| `CHUNK_RERANK` | `rerank.go` | 重排模型打分 + 阈值过滤 + MMR 去冗余 |
| `WEB_FETCH` | `web_fetch.go` | 抓取网页搜索结果全文 |
| `CHUNK_MERGE` | `merge.go` | 去重、父块解析、相邻块扩展、FAQ 答案填充 |
| `FILTER_TOP_K` | `filter_top_k.go` | 按 top-K 截断 |
| `DATA_ANALYSIS` | `data_analysis.go` | 用 DuckDB 对检索到的表格数据跑 SQL 分析 |
| `INTO_CHAT_MESSAGE` | `into_chat_message.go` | 把分块拼进上下文模板，生成最终用户消息 |
| `CHAT_COMPLETION_STREAM` | `chat_completion_stream.go` | 调用 LLM 流式生成答案，经 EventBus 回传 |

---

## 4. 问题检索：混合检索的深入细节

这是「问题检索」的核心。链路：

```text
PluginSearch.OnEvent (search.go)
  └─ searchByTargets()
       ├─ 按 embedding 模型分组（同一模型只算一次 query embedding）
       └─ 每组并发调用 knowledgeBaseService.HybridSearch()
```

`HybridSearch`（`internal/application/service/knowledgebase_search.go:93`）内部步骤：

1. **归一化 MatchCount**（`normalizedMatchCount`）：为 0 时回退 `DefaultRetrievalTopK`。
2. **确定检索 KB 集合**：`params.KnowledgeBaseIDs`（多 KB 合并检索）或单 KB `id`。
3. **批量加载 KB + 鉴权 + 校验 embedding 模型一致**：
   - `GetKnowledgeBaseByIDs`（跨租户共享 KB 也可加载）；
   - `authorizeKBAccess`（防止越权传任意 KB UUID）；
   - `validateSameEmbeddingModel`（多 KB 必须同 embedding 空间）。
4. **选定主 KB** `pickPrimary`（决定 embedding 模型与 FAQ 类型）。
5. **过度召回（Over-Retrieval）**：`matchCount = max(MatchCount*5, DefaultRetrievalTopK) * len(KBs)`，上限 `maxRetrievalPoolSize = 500` —— 先召回更大候选池，交给后续重排裁剪。
6. **计算 query embedding（仅一次）** `GetQueryEmbedding`→`embeddingModel.Embed(queryText)`（`knowledgebase_search.go:25`）。
7. **解析存储分组** `resolveStoreGroups`（`knowledgebase_search_storegroup.go:86`）：按 `(storeID, 租户)` 分组，每组绑定一个向量库引擎（pgvector / Milvus / Qdrant / Weaviate / Elasticsearch / OpenSearch / DuckDB / 腾讯云向量库 等）。
8. **构建检索参数** `buildRetrievalParams`（`knowledgebase_search.go:352`）：
   - 将 KB 按索引路由分区：FAQ 向量库、文档向量库、文档关键词库；
   - 生成 `VectorRetrieverType`（向量）与 `KeywordsRetrieverType`（关键词）两类 `RetrieveParams`。
9. **执行检索** `retrieveFromStores`（`knowledgebase_search_fanout.go:48`）→ 经 `retriever.CompositeRetrieveEngine.Retrieve`（`retriever/composite.go`）并发分发到各向量库引擎。
10. **结果分类** `classifyRetrievalResults`：按 retriever 类型分为向量结果 / 关键词结果。
11. **融合** `fuseOrDeduplicate`（`knowledgebase_search_fusion.go:33`）：
    - 仅向量或仅关键词 → `deduplicateByScore`（按 chunk ID 去重，保留最高分）；
    - 两者都有 → **RRF（Reciprocal Rank Fusion）融合**：

```text
      rrfScore = vectorWeight/(k + vectorRank) + keywordWeight/(k + keywordRank)
      ```

      `k`、`vectorWeight`、`keywordWeight`来自`RetrievalConfig`（有默认值）。

12. **FAQ 后处理** `applyFAQPostProcessing`：FAQ 类 KB 走迭代 TopK 增长的召回路径。
13. **截断** 到 `params.MatchCount`。
14. **上下文富化** `processSearchResults`（`knowledgebase_search_results.go:14`）：
    - 批量拉取 knowledge 元数据（标题、描述）与 chunk 正文；
    - `collectEnrichmentChunkIDs` 补充父块 / 相邻块 / 图片 OCR 文本等，供后续重排与上下文使用。

### 4.1 查询理解（QUERY_UNDERSTAND）

`query_understand.go`：

- 用 LLM（或视觉模型，若有图片）对原始 query 做 **改写 + 意图分类**，输出结构化 JSON：

  ```json
  {"rewrite_query": "...", "intent": "kb_search", "image_description": "..."}
  ```

- 意图类型（`internal/types/chat_manage.go:86`）：`kb_search`、`web_search`、`greeting`、`chitchat`、`follow_up`、`image_only`、`doc_only`、`summarize`、`clarification`。
- 改写后的 `RewriteQuery`会作为后续检索的实际查询文本；非检索意图（如`greeting`）会跳过检索阶段（`NeedsRetrieval()` 返回 false）。

### 4.2 并行检索（CHUNK_SEARCH_PARALLEL）

`search.go`+`search_parallel.go`：

- `search.go`的`PluginSearch` 同时启动 **KB 检索** 与 **网页检索** 两个 goroutine；
- `search_parallel.go`的`PluginSearchParallel` 支持多检索目标并行 fan-out；
- 召回不足时（结果数 < `EmbeddingTopK`）会触发 **查询扩展（query expansion）** 走关键词聚焦的补召回。

---

## 5. 重排序（Rerank）

文件：`internal/application/service/chat_pipeline/rerank.go`

`PluginRerank.OnEvent` 流程：

1. 取重排模型 `GetRerankModel`。
2. **构造 enriched passage** `getEnrichedPassage`：
   - `cleanPassageForRerank` 剥离 markdown / URL / 图片 / 表格分隔符等噪声，保留语义文本；
   - 追加图片 caption / OCR 文本、`ChunkMetadata` 中的生成问题（`GeneratedQuestions`）。
3. **调用重排模型** `rerankModel.Rerank(query, passages)`，得到每个候选的相关性分数。
4. **阈值过滤**（`rerank` 内）：
   - `RelevanceScore >= RerankThreshold` 才保留；
   - 无结果时先尝试 **阈值降级**（×0.7，下限 0.3）重试；
   - 仍无结果时按 **top-1 兜底**（最高分 ≥ `fallbackMinScore`时保留一条，显式作用域下`fallbackMinScore=0`）。
5. **复合分数** `compositeScore`：

```text
   composite = 0.6 * modelScore + 0.3 * baseScore + 0.1 * sourceWeight
   ```

   （`sourceWeight`：网页来源 0.95，其余 1.0）

1. **FAQ 提权**：若开启 FAQ 优先，`FAQScoreBoost` 乘到 FAQ 分块分数上。
2. **MMR（最大边际相关性）去冗余** `applyMMR`：`lambda = 0.7`，在相关性（relevance）与多样性（1 - 冗余度）之间平衡，选出 `RerankTopK` 个不重复的结果。

重排失败时兜底：直接用检索结果（`api_error_fallback`），保证流水线不中断。

---

## 6. 分块合并（Merge）

文件：`internal/application/service/chat_pipeline/merge.go`

`PluginMerge.OnEvent` 的合并顺序（`merge.go:44`）：

1. **选输入**：优先 `RerankResult`，为空则回退 `SearchResult`（按分降序）。
2. **初始去重**（ID + 内容签名）。
3. **注入历史引用** `injectHistoryResults`（把历史轮次引用过的知识片段并入）。
4. **解析父块** `resolveParentChunks`：子块 → 父块，补全上下文。
5. **按知识源 + 分块类型分组并合并相邻正文** `groupAndMergeCurrentContent`。
6. **填充 FAQ 答案** `populateFAQAnswers`。
7. **扩展短上下文** `expandShortContextWithNeighbors`：用相邻块补足过短上下文。
8. **再次合并 + 最终去重**（ID + 签名 + 部分内容重叠 `removePartialOverlaps`）。

最终结果写入 `chatManage.MergeResult`。

---

## 7. 上下文组装与流式生成

### 7.1 INTO_CHAT_MESSAGE（`into_chat_message.go`）

1. 若开启 FAQ 优先，先把 FAQ 分块（高优先级）与文档分块（补充）分开。
2. 校验用户 query 安全性（`utils.ValidateInput`）。
3. 生成 **文档头**（`buildDocumentHeader`）：按 knowledge 去重，列出标题 / 描述 / 自定义元数据。
4. 生成 **context 标签块**：

   ```xml
   <documents>...</documents>
   <context id="1">分块正文1</context>
   <context id="2">分块正文2</context>
   ...
   ```

5. 用 **上下文模板**（`SummaryConfig.ContextTemplate`，来自 `config.yaml`的`conversation.summary.context_template`）渲染占位符 `{query}`、`{contexts}`、`{language}`，得到 `chatManage.UserContent`。
6. 补充图片描述（非视觉模型时）、引用上下文、附件内容。
7. 异步把渲染后的内容写回用户消息（供后续轮次历史使用）。

### 7.2 CHAT_COMPLETION_STREAM（`chat_completion_stream.go`）

1. `prepareChatModel` 取聊天模型与参数（温度、max tokens 等）。
2. `prepareMessagesWithModelContext` 组装消息：系统 prompt + 历史 + 渲染后的用户消息。
3. `chatModel.ChatStream(...)` 发起流式调用。
4. 后台 goroutine 消费流，把 **思考内容** 与 **最终答案** 分别经 `EventBus`发`EventAgentThought`/`EventAgentFinalAnswer`事件（SSE 的`thinking`/`answer` 响应）。

### 7.3 分块数量与回答预算

**最终送入 LLM 的分块数量有硬限制**，经过三层收敛：

| 层级 | 参数 | 默认值 | 说明 |
| --- | --- | --- | --- |
| 检索候选 | `EmbeddingTopK` | 30（config.yaml）/ 50（租户级 RetrievalConfig） | 向量/关键词检索返回候选数 |
| 过度召回池 | `matchCount` | 上限 500（`maxRetrievalPoolSize`） | 先粗召回 5 倍候选 |
| **最终送入 LLM** | **`RerankTopK`** | **30**（config.yaml）/ **10**（租户级） | 重排 MMR 后 + `FILTER_TOP_K` 阶段截断 |

最终 chunk 数 = **`RerankTopK`**（默认 30），由 `FILTER_TOP_K` 阶段硬截断（`filter_top_k.go` 的 `searchResult[:topK]`）。

**回答预算**（区分「输出 token 上限」与「上下文字符」）：

- **输出 token 上限**：`MaxCompletionTokens = 2048`（`config.yaml` `summary.max_completion_tokens`）。这是回答生成的 max output tokens（≈中文 3000~4000 字），**流式回答无字符级截断**（`chat_completion_stream.go` 里无 truncate 逻辑）。
- **上下文字符**：QA 上下文无硬字符上限，由 `RerankTopK × ChunkSize` 决定（≈30 × 512 ≈ 15K 字符，含父块扩展），`into_chat_message.go` 不做字符截断。
- **`max_input_chars = 16384` 只用于「文档摘要」生成**（`getSummary` 里的 `sampleLongContent` 截断），**不是 QA 回答**。
- **Agent 场景**：工具输出上限 `DefaultMaxToolOutput = 24000` 字符；Agent 上下文窗口使用达 80%（`DefaultContextThresholdRatio = 0.8`）触发消息压缩（`agent/token/compress.go`）。

---

## 8. ReAct Agent 问答路径

文件：`internal/application/service/session_agent_qa.go`→`internal/agent/engine.go`

与固定流水线不同，Agent 路径由 LLM **自主决策**每一步：

1. `AgentQA`创建`AgentEngine`（`CreateAgentEngine`），可选注入记忆 prompt。
2. `engine.Execute(...)` 异步执行。
3. **ReAct 循环** `executeLoop`（`engine.go:362`）→ `runReActIteration`（`engine.go:468`），最多 `MaxIterations` 轮：
   - 组装消息与工具清单，调用 LLM；
   - LLM 返回「工具调用」或「最终答案」；
   - 若有工具调用，执行工具（检索类工具见下），把结果写回消息，进入下一轮；
   - 若为最终答案，流式输出并结束。
4. **检索相关工具**（`internal/agent/tools/`）：
   - `knowledge_search.go`—— 知识库混合检索（内部复用`HybridSearch` + 重排 + MMR）；
   - `grep_chunks.go` —— 对分块做关键词 / 正则检索（类似 grep）；
   - `query_knowledge_graph.go` —— 查询知识图谱（Neo4j）；
   - `list_knowledge_chunks.go` —— 列出某文档的分块；
   - `web_search.go` —— 网页搜索；
   - `search_memory.go`/`search_conversations.go` —— 检索长期记忆 / 历史会话。

---

## 9. 关键数据结构

- **`types.ChatManage`**（`internal/types/chat_manage.go:149`）：流水线上下文（内嵌 `PipelineRequest`/`PipelineState`/`PipelineContext`）。
- **`types.SearchParams`**：检索参数（`QueryText`、`QueryEmbedding`、`KnowledgeBaseIDs`、`VectorThreshold`、`KeywordThreshold`、`MatchCount`、`TagIDs`、`KnowledgeIDs` 等）。
- **`types.RetrieveParams`**：单次向量 / 关键词检索参数（`Query`、`Embedding`、`TopK`、`Threshold`、`RetrieverType`、`KnowledgeType` 等）。
- **`types.SearchResult`**：最终检索结果（`ID`、`Content`、`Score`、`MatchType`、`ChunkType`、`KnowledgeID`、`ImageInfo`、`ChunkMetadata` 等）。
- **`types.IndexWithScore`**：向量库返回的原始命中（`ChunkID`、`Score`）。
- **`types.QueryIntent`**（`chat_manage.go:84`）：意图枚举。
- **`types.EventType`**（`chat_manage.go:268`）：流水线阶段枚举。

---

## 10. 关键文件索引

| 关注点 | 文件 |
| --- | --- |
| HTTP 入口 / 分发 | `internal/handler/session/qa.go` |
| RAG 服务入口 + 流水线装配 | `internal/application/service/session_knowledge_qa.go` |
| Agent 服务入口 | `internal/application/service/session_agent_qa.go` |
| 流水线框架（Plugin / EventManager） | `internal/application/service/chat_pipeline/chat_pipeline.go` |
| 阶段枚举 / ChatManage / PipelineBuilder | `internal/types/chat_manage.go` |
| 查询改写 + 意图分类 | `internal/application/service/chat_pipeline/query_understand.go` |
| 检索插件（KB + 网页并发） | `internal/application/service/chat_pipeline/search.go` |
| 并行检索 | `internal/application/service/chat_pipeline/search_parallel.go` |
| 混合检索入口（HybridSearch） | `internal/application/service/knowledgebase_search.go` |
| 检索参数构建 | `internal/application/service/knowledgebase_search.go`（`buildRetrievalParams`） |
| 存储分组解析 | `internal/application/service/knowledgebase_search_storegroup.go` |
| 检索 fan-out | `internal/application/service/knowledgebase_search_fanout.go` |
| RRF 融合 / 去重 | `internal/application/service/knowledgebase_search_fusion.go` |
| 结果上下文富化 | `internal/application/service/knowledgebase_search_results.go` |
| 组合检索引擎 | `internal/application/service/retriever/composite.go` |
| 重排序插件 | `internal/application/service/chat_pipeline/rerank.go` |
| 分块合并插件 | `internal/application/service/chat_pipeline/merge.go` |
| 上下文组装 | `internal/application/service/chat_pipeline/into_chat_message.go` |
| 流式生成 | `internal/application/service/chat_pipeline/chat_completion_stream.go` |
| ReAct 引擎 | `internal/agent/engine.go` |
| 检索类工具 | `internal/agent/tools/knowledge_search.go`、`grep_chunks.go`、`query_knowledge_graph.go`、`web_search.go` |

---

## 附：一条完整 RAG 请求的时序

```text
用户: "WeKnora 如何配置向量数据库？"
  │

  1. handler/session/qa.go:KnowledgeQA
  2. sessionService.KnowledgeQA
       ├─ resolveKnowledgeBases / resolveChatModelID / buildSearchTargets
       └─ 组装 ChatManage + 构建 pipeline 阶段列表

  3. EventManager 按序触发：
       LOAD_HISTORY        ── 载入历史
       MEMORY_RECALL       ── 召回长期记忆
       QUERY_UNDERSTAND    ── 改写为"如何在 WeKnora 中配置向量数据库" + intent=kb_search
       CHUNK_SEARCH_PARALLEL ── HybridSearch:
                                   ├─ GetQueryEmbedding(query) 一次
                                   ├─ 向量检索 + 关键词检索（并发，多 KB）
                                   ├─ classifyRetrievalResults
                                   ├─ RRF 融合
                                   └─ processSearchResults（富化正文/父块/图片）
       CHUNK_RERANK        ── rerank 模型打分 → 阈值过滤 → 复合分 → MMR Top-K
       CHUNK_MERGE         ── 去重 → 父块解析 → 相邻块扩展 → FAQ 填充
       FILTER_TOP_K        ── 截断
       INTO_CHAT_MESSAGE   ── 拼 context 模板 → UserContent
       CHAT_COMPLETION_STREAM ── LLM 流式生成

  4. EventBus 流式回传 answer 到 SSE
```
