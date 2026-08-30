# WeKnora 问答 SSE 流式传输与前端实时渲染

> 本文梳理 WeKnora 从「后端处理结果产出」到「前端页面实时渲染」的完整链路，聚焦两块：
>
> 1. **后端**：处理流水线各阶段的结果，如何变成一条条 SSE 事件流。
> 2. **前端**：如何消费 SSE 事件流，并实时（打字机式）渲染到问答页面。
>
> 本文与《问题检索与QA回答流程.md》互补——那份讲「处理流水线内部」，这份讲「结果如何出得去、前端如何接得住」。
>
> 代码基准：后端 `internal/`（Go），前端 `frontend/src/`（Vue 3 + TypeScript）。

---

## 目录

1. [总体架构：一条被刻意解耦的事件链](#1-总体架构一条被刻意解耦的事件链)
2. [后端：处理结果如何变成事件](#2-后端处理结果如何变成事件)
3. [SSE 传输层](#3-sse-传输层)
4. [前端：实时消费与渲染](#4-前端实时消费与渲染)
5. [response_type 完整映射表](#5-response_type-完整映射表)
6. [一条完整请求的实时数据流](#6-一条完整请求的实时数据流)
7. [关键文件索引](#7-关键文件索引)
8. [工具结果 display_type 与前端渲染器映射](#8-工具结果-display_type-与前端渲染器映射)

---

## 1. 总体架构：一条被刻意解耦的事件链

WeKnora 的实时显示**不是**「后端 handler 直接往连接里写 SSE」，而是经过**三层缓冲解耦**：

```text
LLM 生成 / 各插件阶段
      │  EventBus.Emit(Event{Type: "final_answer", ...})    ← 进程内发布/订阅
      ▼
AgentStreamHandler（订阅 EventBus）                           ← 进程内
      │  把事件翻译成 StreamEvent{Type: "answer", ...}
      │  streamManager.AppendEvent(...)                     ← 写入中间存储
      ▼
StreamManager（内存 或 Redis，append-only 列表）               ← 跨进程/可重放
      │  GetEvents(offset) 增量读取
      ▼
SSE Handler（100ms 轮询 + c.SSEvent("message", ...)）        ← HTTP 长连接
      │  text/event-stream
      ▼
前端 fetchEventSource → onmessage → processStreamChunk → 响应式渲染
```

### 为什么中间要插一层 StreamManager？

从代码注释与设计可以归纳出三点：

1. **支持断线续传/重放**：`ContinueStream`（`internal/handler/session/stream.go:35`）可按 `offset` 从 StreamManager 重放已发生的事件，再继续轮询新事件。
2. **分布式部署**：StreamManager 有 Redis 实现（`internal/stream/redis_manager.go`），事件不依赖单机进程内存。
3. **解耦「生成」与「推送」**：SSE handler 用 100ms 定时器**拉**（pull），而非 EventBus 直接**推**。客户端断开时生成不会中断，生成方也无需关心连接状态。

### StreamManager 的抽象

`internal/types/interfaces/stream_manager.go` 定义了一个极简的 append-only 接口：

```go
type StreamManager interface {
    // 追加单条事件（Redis RPush，O(1)）
    AppendEvent(ctx, sessionID, messageID string, event StreamEvent) error
    // 从 offset 起增量读取（Redis LRange），返回 (events, nextOffset)
    GetEvents(ctx, sessionID, messageID string, fromOffset int) ([]StreamEvent, int, error)
}
```

**Key 是 `(sessionID, messageID)`**，即每条 assistant 消息对应一条独立的事件流。

---

## 2. 后端：处理结果如何变成事件

### 2.1 两个写入源

事件有**两个来源**写进 StreamManager：

| 来源 | 谁写 | 写什么 |
| --- | --- | --- |
| **业务事件**（EventBus） | `AgentStreamHandler`（订阅 EventBus） | thinking / answer / tool_call / references / complete 等 |
| **Handler 直接写** | `Handler` 层 | `agent_query`（开场）、`stop`（停止）、`artifacts_pending`（沙箱产物） |

### 2.2 核心翻译器：AgentStreamHandler

文件 `internal/handler/session/agent_stream_handler.go`。

它在 `setupSSEStream`（`internal/handler/session/qa.go:678`）里被创建并 `Subscribe()`，订阅 **14 种 EventBus 事件**，每种事件映射成对应的 SSE `response_type`（见 [第 5 节](#5-response_type-完整映射表) 完整对照）。

关键设计点（均在此文件）：

- **流式 chunk 不累积**：`handleFinalAnswer`（`:461`）与 `handleThought`（`:131`）每收到一个 chunk 就 `AppendEvent` 一条 `Content = data.Content`（单个片段）。**前端负责按 `event_id` 累积**（见第 4 节）。`done` 标志标记该事件流结束。
- **answer 的「开场白回撤」机制**：`handleToolCall`（`:179`）里，若 Agent 先流式输出了一段文字（"让我先搜索…"）之后才发现要调工具，会把之前已流式的 answer segment 标记为 `superseded`，从**持久化到数据库**的最终答案里剔除。前端流里也有对应的 retract 逻辑（见 4.3）。
- **TTFB 埋点**：`handleFinalAnswer` 在第一个 answer chunk 时打 `TTFB:first_answer_chunk` 日志（`:478`），与前端 `streame.ts` 的 `[TTFB]` 日志按 `X-Request-ID` 对齐，用于端到端延迟定位。
- **兜底 answer**：`handleComplete`（`:695`）里，若整个流程没有任何 answer 事件被流式输出（例如模型自然停止、或 Ollama 非增量返回工具调用），会把 `FinalAnswer` 整体作为一条 answer 事件补发，保证前端一定能渲染出答案。

### 2.3 RAG 快速问答也走同一条路

RAG 路径（`KnowledgeQA`）在 `chat_completion_stream.go` 里同样发 `EventAgentThought` / `EventAgentFinalAnswer`。因此 **RAG 模式与 Agent 模式在前端是同一套 SSE 协议**，区别只在前端根据 `isAgentStreamSession()` 决定走哪种渲染分支（见 4.2）。

---

## 3. SSE 传输层

### 3.1 建立连接

`internal/handler/session/qa.go:614` 的 `setupSSEStream`：

1. `setSSEHeaders`（`helpers.go:182`）：写
   - `Content-Type: text/event-stream`
   - `Cache-Control: no-cache`
   - `Connection: keep-alive`
   - `X-Accel-Buffering: no`（禁用 Nginx 缓冲，保证逐块下发）
2. 立即写一条 `agent_query` 事件（`writeAgentQueryEvent`，`helpers.go:418`）——前端靠它**创建 assistant 占位消息**（在收到第一条 answer 前先显示加载态），并携带持久化的 user/assistant 时间戳。
3. 创建独立 `EventBus` + 可取消 context；注册 stop 事件处理、启动 stop watcher、注册 `setupStreamHandler`（创建 `AgentStreamHandler` 并订阅）。
4. 需要时异步生成会话标题（`GenerateTitleAsync`）。

### 3.2 推送循环（pull-based）

`internal/handler/session/stream.go:330` 的 `handleAgentEventsForSSE` 是真正的推送循环：

```go
ticker := time.NewTicker(100 * time.Millisecond)
for {
    select {
    case <-c.Request.Context().Done():   // 客户端断开 → 优雅退出
        return
    case <-ticker.C:
        events, newOffset := streamManager.GetEvents(ctx, sessionID, msgID, lastOffset)
        for _, evt := range events {
            emitStreamEvent(ctx, c, evt, requestID, rewriter)  // c.SSEvent("message", resp); c.Writer.Flush()
        }
        lastOffset = newOffset
        if streamCompleted { sendCompletionEvent(...); return }
    }
}
```

要点：

- **100ms 轮询**：这是「拉」模型的核心。所有事件先进 StreamManager，再由这个 ticker 增量取出推给浏览器。
- **`emitStreamEvent`**（`resource_urls.go:113`）→ `buildStreamResponse`（`helpers.go:190`）把 `StreamEvent` 组装成 `StreamResponse`（`chat.go:252`）。
  - `references` 事件特殊处理：把 `data["references"]` 反序列化进 `KnowledgeReferences` 字段。
  - `agent_query` 事件提取 `session_id` / `assistant_message_id` 到顶层字段。
- **`complete` 是终态信号**：`sendCompletionEvent` 是**空函数**（`helpers.go:242`），真正的完成信号就是 `complete` 事件本身。注释明确说明「多发一个 `done:true` 的 answer 会搞乱前端状态」。
- **`resource_urls` 重写**：`emitStreamEvent` 里 `rewriter` 会在 `public` 模式下把资源 URL 重写为可加载直链；若流在资源未就绪时提前结束，`flushHeldStreamContent` 会释放暂存的尾部片段，避免引用/尾文本丢失（`resource_urls.go:128`）。

### 3.3 停止（stop）

`StopSession`（`stream.go:219`）往 StreamManager 写一条 `stop` 事件，随后：

1. `startStopWatcher`（`helpers.go:364`）——**独立于 SSE 连接的 300ms 轮询** goroutine 捕获到 `stop` 事件，经 EventBus 触发 context 取消（即使客户端已断开也能停）。
2. `handleAgentEventsForSSE` 同样检测到 `stop` 事件，flush 缓冲后向前端发 `stop` 并返回。

### 3.4 续传（ContinueStream）

`stream.go:35` 的 `ContinueStream`（`GET /sessions/continue-stream/{session_id}?message_id=...`）：

1. 从 offset 0 重放 StreamManager 里已有事件。
2. 若已含 `complete`，直接发完成事件返回。
3. 否则继续 100ms 轮询后续事件。

用于客户端断线重连后补齐错过的流。

---

## 4. 前端：实时消费与渲染

### 4.1 发起流式请求：streame.ts

文件 `frontend/src/api/chat/streame.ts`，基于 `@microsoft/fetch-event-source`：

- `startStream` 组装请求（session_id、query、knowledge_base_ids、agent_enabled、images、attachments…），携带 `Authorization`（embed 用 `Embed <token>`，web 用 `Bearer <token>`）、`X-Tenant-ID`、`X-Request-ID`。
- `onmessage`（`:175`）：**每个 SSE 消息 `JSON.parse` 后 push 进 buffer，同时调用 `chunkHandler(parsed)`**。真正的渲染逻辑通过 `onChunk` 注册的回调（在 `useChatStreamHandler.ts`）执行。
- `onclose` → `stopStream`；`stopStream` 里 `streamGeneration++` 做**代际隔离**（`:176` 的 `myGeneration !== streamGeneration` 判断），防止旧请求的迟到消息污染新请求。

### 4.2 分发中枢：useChatStreamHandler.ts 的 processStreamChunk

`index.vue` 里 `onChunk((data) => { ... processStreamChunk(data) })`（`index.vue:954`）。`processStreamChunk` 定义于 `frontend/src/composables/useChatStreamHandler.ts:897`。

它先处理特例，再决定走 Agent 分支还是非-Agent 分支：

```text
processStreamChunk(data):
  ├─ session_title     → 更新侧边栏会话标题（index.vue:955 直接处理，不进入）
  ├─ agent_query       → 创建/续接 assistant 占位消息（含 _eventMap/_pendingToolCalls）
  ├─ references        → applyKnowledgeReferences()
  ├─ memory_recalled   → applyUsedMemories()
  ├─ 判断 shouldHandleAsAgent
  │      （isAgentOnlyResponse || isAgentMode || isAgentAnswerChunk || isAgentCompleteChunk）
  │      └─ true  → handleAgentChunk(data)     ← Agent 模式
  └─ false → 非-Agent 分支：fullContent 累积 + <think> 标签解析 → updateAssistantSession
```

**判断是否走 Agent 分支的关键**（`:974-991`）：

- `isAgentOnlyResponse`：`thinking / tool_call / tool_result / reflection / artifacts_pending` 这些**只有 Agent 会发**。
- `isAgentAnswerChunk`：`answer` 且 `isAgentStreamSession()`（当前会话是 agent 模式）或 `data.id` 匹配当前 assistant message。

### 4.3 Agent 模式渲染：handleAgentChunk

`handleAgentChunk` 里的 `switch(responseType)`（`:554`）逐事件累积到一个响应式 `message.agentEventStream` 数组：

| response_type | 处理 |
| --- | --- |
| `thinking` | 按 `event_id` 找/建 thinking 事件，累积 `content`；`done` 时算 `duration_ms`（`:555-599`） |
| `tool_call` | 按 `tool_call_id` 建 pending 工具调用事件，**并把之前已流的 answer 标记 `superseded`**（前端回撤，`:668-683`） |
| `tool_result` / `error` | 匹配 pending 工具调用，填 `output/error/duration/display_type`（`:732-799`） |
| `answer` | 按 `event_id` 累积到 answer 事件，同步 `recomposeAgentAnswer` 更新 `message.content`（`:801-837`） |
| `artifacts_pending` | 设置 `artifactsCollecting`，显示「产物上传中」占位 |
| `complete` | 标记完成，hydrate `artifacts`（沙箱生成的文件下载按钮，`:847-874`） |
| `stop` | 标记完成 |

**渲染组件**是 `frontend/src/views/chat/components/AgentStreamDisplay.vue`——遍历 `message.agentEventStream`，把 thinking / tool_call / tool_result 渲染成可折叠的步骤卡片；工具结果再分发到 `tool-results/` 下的专用渲染器（`KnowledgeChunksList.vue`、`WebFetchResults.vue`、`GrepResults.vue`、`GraphQueryResults.vue`、`PlanDisplay.vue` 等）。

### 4.4 RAG 模式渲染（非 Agent）

走 `processStreamChunk` 末尾（`:1040-1079`）：

1. `fullContent.value += data.content`（**累积所有 answer chunk 到一个字符串**）。
2. 解析 `<think>...</think>` 标签：有开头无结尾 → 思考中；有完整标签 → 拆分 `thinkContent`（思考）与 `content`（正文）。
3. `updateAssistantSession(obj)` 响应式更新消息。
4. `done` 时触发 `onReplyComplete` / `onTurnComplete`。

渲染组件是 `botmsg.vue`（markdown 正文 + `deepThink.vue` 折叠思考过程）。

### 4.5 两条关键前端技巧

1. **`_eventMap` / `_pendingToolCalls`**：流式 chunk 是「每 chunk 一条 SSE」，前端用 `event_id`（Map）做 O(1) 累积定位，用 `tool_call_id`（Map）把 `tool_result` 关联回 `tool_call`。
2. **引用（references）单独走 `applyKnowledgeReferences`**，写入 `message.knowledge_references`，由 `ChatReferencesDrawer` / 引用角标渲染，不混进 answer 文本。

---

## 5. response_type 完整映射表

后端 `response_type` 常量定义在 `internal/types/chat.go:200`；EventBus 事件定义在 `internal/event/event.go`。

| EventBus 事件 | SSE `response_type` | 前端处理（useChatStreamHandler） |
| --- | --- | --- |
| `EventAgentThought` | `thinking` | 累积思考内容（Agent） |
| `EventAgentToolCall` | `tool_call` | 建 pending 工具卡片（Agent） |
| `EventAgentToolResult` | `tool_result`（失败→`error`） | 填工具结果（Agent） |
| `EventAgentReferences` | `references` | `applyKnowledgeReferences`（两种模式共用） |
| `EventMemoryRecalled` | `memory_recalled` | `applyUsedMemories`（两种模式共用） |
| `EventAgentFinalAnswer` | `answer` | Agent：累积 answer 事件；非-Agent：fullContent 累积 |
| `EventAgentReflection` | `reflection` | 反思卡片（Agent） |
| `EventError` | `error` | 错误卡片 / 报错（两种模式） |
| `EventSessionTitle` | `session_title` | 更新侧边栏标题 |
| `EventAgentComplete` | `complete` | 标记完成 + hydrate artifacts/usage |
| `EventToolApprovalRequired/Resolved` | `tool_approval_required/resolved` | MCP 危险工具人工审批卡片 |
| `EventMCPOAuthRequired/Resolved` | `mcp_oauth_required/resolved` | MCP OAuth 授权卡片 |
| （Handler 直接写） | `agent_query` | 创建 assistant 占位消息 |
| （Handler 直接写） | `artifacts_pending` | 「产物上传中」占位 |
| （Handler 直接写） | `stop` | 标记停止 |

---

## 6. 一条完整请求的实时数据流

对照《问题检索与QA回答流程.md》的时序，补上「事件如何实时流出」：

```text
用户提问
  │  POST /api/v1/sessions/{id}   (fetchEventSource)
  ▼
setupSSEStream: 写 SSE 头 + 立即写 agent_query 事件 ──────────► 前端建占位消息（加载态）
  │
  ▼
KnowledgeQA/AgentQA 流水线执行（检索→重排→…→LLM 流式生成）
  │  各阶段经 EventBus.Emit 发事件
  ▼
AgentStreamHandler（订阅）: 每个事件 → AppendEvent 到 StreamManager
  │
  ▼
SSE 轮询循环（100ms）: GetEvents(offset) → c.SSEvent("message", StreamResponse) → Flush
  │  典型顺序：agent_query → thinking → tool_call → tool_result
  │           → references → answer(chunk1..N) → complete
  ▼
前端 onmessage: JSON.parse → processStreamChunk → 按 response_type 分发
  │  agentEventStream[] 累积 → AgentStreamDisplay 响应式重绘
  ▼
complete 事件 → 前端标记 is_completed，停止加载态，展示引用/产物/usage
```

---

## 7. 关键文件索引

| 关注点 | 文件 |
| --- | --- |
| SSE 推送循环（轮询/续传/停止） | `internal/handler/session/stream.go` |
| SSE 头、buildStreamResponse、createAgentQueryEvent | `internal/handler/session/helpers.go` |
| SSE 事件写出发射（emitStreamEvent） | `internal/handler/session/resource_urls.go` |
| **EventBus 事件 → SSE 事件翻译** | `internal/handler/session/agent_stream_handler.go` |
| SSE 连接建立（setupSSEStream） | `internal/handler/session/qa.go:614` |
| EventBus 定义 | `internal/event/event.go` |
| 事件数据结构 | `internal/event/event_data.go` |
| StreamManager 接口 | `internal/types/interfaces/stream_manager.go` |
| StreamManager 实现（内存/Redis） | `internal/stream/memory_manager.go`、`redis_manager.go` |
| `response_type` / `StreamResponse` 定义 | `internal/types/chat.go:200` |
| 前端流式请求（fetchEventSource） | `frontend/src/api/chat/streame.ts` |
| **前端事件分发中枢** | `frontend/src/composables/useChatStreamHandler.ts` |
| Agent 事件流渲染 | `frontend/src/views/chat/components/AgentStreamDisplay.vue` |
| RAG 答案渲染 | `frontend/src/views/chat/components/botmsg.vue`、`deepThink.vue` |
| 工具结果专用渲染器 | `frontend/src/views/chat/components/tool-results/*.vue` |

---

## 8. 工具结果 display_type 与前端渲染器映射

> 本节是第 4.3 节「Agent 模式渲染」的展开，聚焦 tool_result 事件内部的富渲染契约。

Agent 模式下，每个工具结果该「画成什么卡片」，由后端写在 `ToolResult.Data["display_type"]` 里的一个字符串契约决定；前端 `ToolResultRenderer.vue` 据此分发到专用组件。它是前后端之间关于工具结果展示的**唯一对齐点**。

### 8.1 数据流转

```text
工具执行（internal/agent/tools/*.go）
  │  result.Data["display_type"] = "search_results"（例）
  ▼
agent_stream_handler.go handleToolResult
  │  clientData := SanitizeToolResultForClient(toolName, result)   ← persist.go:76
  │      遍历 result.Data，清理大字段后合并进 metadata（display_type 保留）
  ▼
StreamEvent.Data = metadata  →  SSE 的 StreamResponse.Data
  │
  ▼
前端 useChatStreamHandler.ts（tool_result case）
  │  toolCallEvent.tool_data = dataPayload          ← = data.data
  │  toolCallEvent.display_type = dataPayload.display_type
  ▼
AgentStreamDisplay.vue  resolveToolDisplayType(event)   ← 读 event.display_type
  ▼
ToolResultRenderer.vue  v-if 链按 displayType 分发
  ▼
tool-results/*.vue 专用渲染组件
```

### 8.2 完整映射表

工具名常量在 `internal/agent/tools/definitions.go`；`display_type` 设置在各自工具文件。

| 工具（ToolName） | display_type | 前端渲染器 | display_type 设置处 |
| --- | --- | --- | --- |
| `knowledge_search`（语义搜索） | `search_results` | `SearchResults.vue` | `knowledge_search.go:1143` |
| `web_search` | `web_search_results` | `WebSearchResults.vue` | `web_search.go:289` |
| `web_fetch` | `web_fetch_results` | `WebFetchResults.vue` | `web_fetch.go:226` |
| `grep_chunks`（关键词搜索） | `grep_results` | `GrepResults.vue` | `grep_chunks.go:220` |
| `list_knowledge_chunks` | `knowledge_chunks_list` | `KnowledgeChunksList.vue` | `list_knowledge_chunks.go:264,317` |
| `get_document_info` | `document_info` | `DocumentInfo.vue` | `get_document_info.go:322` |
| `query_knowledge_graph` | `graph_query_results` | `GraphQueryResults.vue` | `query_knowledge_graph.go:381` |
| `database_query` | `database_query` | `DatabaseQuery.vue` | `database_query.go:246` |
| `data_analysis` | `data_analysis` | **（无 → fallback）** | `data_analysis.go:296` |
| `shell_exec` | `shell_exec` | `ShellExecResult.vue` | `shell_exec.go:515` |
| `todo_write` | `plan` | `PlanDisplay.vue` | `todo_write.go:197` |
| `thinking`（sequentialthinking） | `thinking` | `ThinkingDisplay.vue` | `sequentialthinking.go:217` |
| `wiki_write_page` | `wiki_write_page` | `WikiEditResult.vue` | `wiki_write_page.go:237` |
| `wiki_replace_text` | `wiki_replace_text` | `WikiEditResult.vue` | `wiki_replace_text.go:139` |
| `wiki_rename_page` | `wiki_rename_page` | `WikiEditResult.vue` | `wiki_rename_page.go:169` |
| `wiki_delete_page` | `wiki_delete_page` | `WikiEditResult.vue` | `wiki_delete_page.go:121` |
| `wiki_read_source_doc`（精读源文档） | `knowledge_chunks_list` | `KnowledgeChunksList.vue`（复用） | `wiki_read_source_doc.go:361` |

补充说明：

- **wiki 四件套共用 `WikiEditResult.vue`**，靠 `display_type` 区分显示字段（`AgentStreamDisplay.vue` 的 `formatWikiEditResultContent`）。
- **`wiki_read_source_doc` 复用 `knowledge_chunks_list`**：精读源文档返回的也是「一组分块」。
- **`data_analysis` 无对应渲染器**：后端写了 `display_type: "data_analysis"`，但前端没有这个分支，会落到 fallback（见 8.4）。

### 8.3 前端分发链

`ToolResultRenderer.vue` 的模板是一条 `v-if / v-else-if` 链（`:4-64`）：

```text
search_results          → SearchResults.vue
chunk_detail            → ChunkDetail.vue
related_chunks          → RelatedChunks.vue
knowledge_base_list     → KnowledgeBaseList.vue
document_info           → DocumentInfo.vue
graph_query_results     → GraphQueryResults.vue
thinking                → ThinkingDisplay.vue
plan                    → PlanDisplay.vue
database_query          → DatabaseQuery.vue
web_search_results      → WebSearchResults.vue
web_fetch_results       → WebFetchResults.vue
grep_results            → GrepResults.vue
knowledge_chunks_list   → KnowledgeChunksList.vue
wiki_write_page |
wiki_replace_text |
wiki_rename_page |
wiki_delete_page        → WikiEditResult.vue
shell_exec              → ShellExecResult.vue
（其它任何值）           → fallback：直接显示原始 output 文本
```

组件 Props（`:105-117`）：`displayType`（分发依据）、`toolData`（结构化结果，= `event.tool_data`）、`output`（原始文本，fallback 用）、`arguments`（工具入参）。

### 8.4 前后端不一致点与兜底

**前端有、后端当前不设置（疑似遗留/预留）**——不会命中：

| display_type | 渲染器 |
| --- | --- |
| `chunk_detail` | `ChunkDetail.vue` |
| `related_chunks` | `RelatedChunks.vue` |
| `knowledge_base_list` | `KnowledgeBaseList.vue` |

**后端有、前端不识别（走 fallback）**：

| 工具 | display_type | 后果 |
| --- | --- | --- |
| `data_analysis` | `data_analysis` | 无分支 → fallback 显示原始文本 |
| `data_schema` | （未设置） | 无分支 → fallback |
| `attachment_parsing` | `attachment_parsing`（`qa.go:1227`，handler 层，非 Agent 工具） | 无分支 → fallback |

`data_analysis` 还能用是因为分析结论就在 `output` 文本里，fallback 直接显示也够用，只是失去了富卡片。

**前端 tool_name 兜底**：`AgentStreamDisplay.vue:1072` 的 `resolveToolDisplayType` 优先信 `display_type`，仅 `shell_exec` 有 tool_name 兜底（历史数据可能无 display_type）：

```ts
if (event?.display_type) return event.display_type as DisplayType
if (event?.tool_name === 'shell_exec') return 'shell_exec'
return undefined
```

此外 `AgentStreamDisplay.vue` 还有一批**绕过 ToolResultRenderer、直接按 `event.tool_name` 判断**的轻量摘要（`:154-173`），例如 `web_search` 显示「找到 N 条结果」、`grep_chunks` 显示匹配摘要、`todo_write` 显示任务计划等；同时兼容 `search_knowledge` 与 `knowledge_search` 两个名字（历史改名）。

### 8.5 display_type 的「双轨」机制

`display_type` 除了驱动前端渲染，还驱动后端对工具输出的**省略与摘要**（`internal/agent/tools/persist.go`）：

1. **`ShouldOmitRawToolOutput`**（`:36`）：只要 `Data` 有 `display_type`，就把原始 `Output` 从 SSE 回放和持久化的 `agent_steps` 里省略——前端已能靠 `display_type + tool_data` 渲染富卡片，无需再传冗长原始文本。
2. **`compactToolSummary`**（`:263`）：被省略时生成一句人类可读摘要，用于**历史记录和下一轮模型上下文**，对不同 `display_type` 有定制话术：

   ```text
   knowledge_chunks_list → "Listed N/M chunks from <title> (content omitted from history)"
   grep_results          → "Keyword search found N matching chunks across M document(s)"
   search_results        → "Semantic search returned N result(s)"
   shell_exec            → 重建 "shell_exec exit=N command=..."
   attachment_parsing    → "Parsed N attachment(s)"
   ```

3. **`persistStripFields`**（`:11`）：对 `knowledge_chunks_list` 删 `chunks`、对 `grep_results` 删 `chunk_results`，避免大块内容进 SSE / DB。
4. **`persistStripFieldsByTool` / `clientStripFieldsByTool`**（`:18-29`）：对 `shell_exec` / `read_sandbox_file` 删 `content` / `content_base64`（二进制/重复 blob），但保留 `stdout` / `stderr`（前端终端卡片需要）。

> 一句话：**`display_type` 既是前端「怎么画」的契约，也是后端「存什么、传什么」的裁剪开关。**

---

## 附：可进一步深入的点

- `resource_urls.go` 里 `public` / `handle` 两种资源 URL 重写模式，以及 `StreamRewriter` 的 holdback buffer 机制。
- `recomposeAgentAnswer` 与 `superseded` 回撤的完整状态机（前端 `useChatStreamHandler.ts` + 后端 `agent_stream_handler.go` 的 `answerSegment`）。
- `streame.ts` 的 TTFB 埋点与后端 `TTFB:first_answer_chunk` 日志的端到端对齐方式。
