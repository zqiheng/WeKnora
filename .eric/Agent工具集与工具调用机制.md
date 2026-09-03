# WeKnora Agent 工具集与工具调用机制

> 本文梳理两件事：**（一）LLM 是如何决定调用某个工具的**（tool-calling 的完整链路），**（二）每个工具的用途、调用场景与适用范围**。
>
> 代码基准：`internal/agent/`（ReAct 引擎）、`internal/agent/tools/`（工具实现与注册表）、`internal/application/service/agent_service.go`（工具注册与能力裁剪）、`config/prompt_templates/agent_system_prompt.yaml`（系统提示词）。
>
> 关联文档：[[问题检索与QA回答流程]]（AgentQA 路径）、[[Agent处理过程记录与前端实时显示]]（工具调用如何被前端实时渲染）。

---

## 目录

1. [总览：工具系统的分层](#1-总览工具系统的分层)
2. [LLM 如何判断调用工具](#2-llm-如何判断调用工具)
3. [工具注册与能力裁剪](#3-工具注册与能力裁剪)
4. [工具调用的一次完整生命周期](#4-工具调用的一次完整生命周期)
5. [知识检索类工具详解](#5-知识检索类工具详解)
6. [Wiki 类工具详解](#6-wiki-类工具详解)
7. [网页 / 记忆 / 通用 / 技能沙箱 / MCP 类工具详解](#7-其他类工具详解)
8. [关键文件索引](#8-关键文件索引)
9. [提示词体系：使用细节与写法](#9-提示词体系使用细节与写法)

---

## 1. 总览：工具系统的分层

WeKnora 的 Agent 工具不是写死的一组指令，而是一套「**可裁剪、可扩展、带作用域鉴权**」的工具系统，分四层：

```text
┌─────────────────────────────────────────────────────────────┐
│  ToolRegistry（注册表）  注册/执行/参数校验/输出截断            │
│    ├─ 知识检索类（8）   依赖 RAG KB（向量/关键词索引）           │
│    ├─ Wiki 类（10）     依赖 Wiki KB                           │
│    ├─ 网页类（2）       依赖 WebSearchEnabled                  │
│    ├─ 记忆/历史类（2）   依赖 memory 开关 / 会话历史             │
│    ├─ 通用类（2）       thinking / todo_write                  │
│    ├─ 技能/沙箱类（5）   依赖 skills / 沙箱后端能力              │
│    └─ MCP 类（动态）     外部 MCP server 注入                   │
└─────────────────────────────────────────────────────────────┘
```

每个工具都是 `name`（工具名）+ `description`（何时用/何时不用）+ JSON `schema`（参数）三要素构成，通过 `ToolRegistry.RegisterTool` 注册。`description` 是**决定 LLM 会不会正确调用该工具的关键**——它是写给 LLM 的「使用契约」。

---

## 2. LLM 如何判断调用工具

### 2.1 本质：OpenAI 兼容的 Function Calling

LLM 调用哪个工具**不是后端代码 `if/else` 决定的**，而是走标准的 **function calling**：

1. `AgentEngine.buildToolsForLLM()`（`internal/agent/observe.go:581`）把注册表里所有工具的 `FunctionDefinition`（`name + description + parameters`）序列化成 `chat.Tool` 列表，作为 `tools` 参数发给 LLM。
2. LLM 根据**系统提示词 + 对话历史 + 每个工具的 description/schema**，自主决定：返回「工具调用」（带工具名 + JSON 参数）还是「最终答案」。
3. 引擎只做三件事：**给工具清单、执行返回的工具调用、把结果喂回给 LLM**，循环直到自然停止或达到迭代上限。

### 2.2 LLM 决策的三重依据

LLM「该调哪个工具、调不调」依据三层信息，都在 prompt 里：

| 层 | 内容 | 来源 |
| --- | --- | --- |
| 系统提示词 | 「渐进式 RAG / 纯 Agent」的总体行为准则（先检索再回答、如何组织证据等） | `config/prompt_templates/agent_system_prompt.yaml`（`mode: rag` / `mode: pure`），`internal/agent/prompts.go` |
| 运行时上下文 `<runtime_context>` | 当前绑定的 KB 列表（`id/name/type/doc_count/capabilities`）与最近文档 | `formatKnowledgeBaseList`（`prompts.go:129`）+ `observe.buildRuntimeContextBlock` |
| 每个工具的 `description` | 明确写「何时用 / 何时不用 / 参数怎么填 / 输出是什么」 | 各工具文件的 `BaseTool.description` |

其中**工具 description 是最精准的「分诊」依据**。例如：

- `knowledge_search` 的 description 写「**Do NOT** search for specific named entities / literal lookup」——引导 LLM 在找精确词时不去调它；
- `grep_chunks` 的 description 写「locate candidate chunks by exact identifiers, error codes, product names」——引导 LLM 在找精确词时调它；
- `web_search` 的 description 写「**ABSOLUTE RULE**: 必须先跑 grep_chunks 和 knowledge_search，二者无结果才用」——用 description 直接硬编码了调用顺序。

### 2.3 ReAct 循环：think → analyze → act → observe

`internal/agent/engine.go` 的 `executeLoop` / `runReActIteration`：

```text
           ┌───────────────────────────────────────────┐
           │  for round < MaxIterations                 │
           │                                            │
           │  1. Think   callLLMWithRetry(messages, tools) │
           │       └─ LLM 返回 ToolCalls 或 纯文本          │
           │                                            │
           │  2. Analyze analyzeResponse                   │
           │       ├─ 无 ToolCalls 且自然 stop → 最终答案   │
           │       └─ 有 ToolCalls → 继续                   │
           │                                            │
           │  3. Act     executeToolCalls（可并行）        │
           │                                            │
           │  4. Observe appendToolResults（role:tool 回填）│
           │       └─ 进入下一轮                            │
           └───────────────────────────────────────────┘
```

关键控制逻辑（`runReActIteration`）：

- **结束条件**：LLM 返回纯文本且 `finish_reason` 自然停止、无工具调用 → `analyzeResponse` 判 `isDone`，产出 `FinalAnswer`。
- **空回复重试**：自然停止但内容为空 → 追加一句 `"Please provide your complete answer now as plain text."` 再重试，最多 `maxEmptyResponseRetries` 次。
- **卡死检测**：连续 `maxRepeatedResponseRounds` 轮返回相同内容且无工具调用 → 判定 stuck loop，提前终止。
- **迭代上限**：`MaxIterations` 用尽仍未给出最终答案 → `handleMaxIterations` 兜底生成。
- **上下文窗口管理**：每轮用「上一轮 API 用量 + 新增消息的 BPE 增量」估算 token，超阈值时触发消息压缩（`internal/agent/token/compress.go`）。
- **并行调用**：`ParallelToolCalls` 开启且一轮有 2+ 个工具调用时，用 errgroup 并发执行（`executeToolCallsParallel`）。

### 2.4 短 ID handle 机制

LLM 在工具参数里看到的是**短 ID**（`bN` 知识库 / `dN` 文档 / `cN` 分块 / `wN` 网页 / `res://` 等），由 `modelcontext.Registry` 在请求边界内解析成真实 ID：

- `buildToolsForLLM` 之外，`modelContext.ProtocolPrompt()` 会把 ID 约定写进系统提示词；
- 执行前 `act.go:478` 检查 `UnresolvedHandles`：**凡是没有解析成功的 handle（LLM 幻觉出的 ID）会在工具执行前被拦截**，绝不落到持久层或外部服务。

---

## 3. 工具注册与能力裁剪

`internal/application/service/agent_service.go:622` 的 `registerTools` 决定「这一次会话到底注册哪些工具」，分四步：

1. **白名单**：`config.AllowedTools`（用户可编辑的显式白名单）→ 为空则回退 `DefaultAllowedTools()`（默认 11 个）。
2. **能力检测**：从 `SearchTargets` 判断 `hasVectorKB`（向量/关键词索引）、`hasWikiKB`（Wiki）、`hasKnowledge`（是否绑定了 KB 作用域）。
3. **硬性裁剪**（防 LLM 调用运行时必报错的工具）：
   - 无 RAG KB → 剔除 `knowledge_search/grep_chunks/list_knowledge_chunks/query_knowledge_graph/get_document_info/database_query`；
   - 无 Wiki KB → 剔除全部 Wiki 工具；
   - 无 KB 作用域（纯聊天）→ 剔除上述所有 KB 工具，若也没开网页搜索则连 `todo_write` 也剔掉；
   - `WebSearchEnabled` → 追加 `web_search/web_fetch`；
   - memory 三开关（工作区/用户/Agent）全开 → 追加 `search_memory`。
4. **注册**：逐个 `switch` 构造工具实例并 `registry.RegisterTool`。

默认白名单（`definitions.go:96`）：`thinking`、`todo_write`、`knowledge_search`、`grep_chunks`、`list_knowledge_chunks`、`query_knowledge_graph`、`get_document_info`、`search_conversations`、`database_query`、`data_analysis`、`data_schema`。（`search_memory` 不在默认列表，随 memory 开关注入。）

---

## 4. 工具调用的一次完整生命周期

`ToolRegistry.ExecuteTool`（`registry.go:112`）→ `runToolCall`（`act.go:356`）的完整链路：

```text
LLM 返回 ToolCall {name, arguments}
  ├─ NormalizeToolCallID          归一化 tool_call_id
  ├─ JSON 解析（失败则 RepairJSON 修复）
  ├─ CastParams                   类型校正（"true"→true 等 LLM 常见笔误）
  ├─ ValidateParams               按 JSON schema 校验（不合法直接返回错误+hint）
  ├─ 检查 UnresolvedHandles       幻觉/过期 ID → 执行前拦截
  ├─ tool.Execute(ctx, args)      实际执行（带输出预算 WithOutputBudget）
  ├─ TruncateToolOutput           超限按 rune 截断（默认 24000 字符）
  └─ 返回 ToolResult{Success, Output, Data, Images}
        └─ appendToolResults      包成 role:tool 消息写回，进入下一轮
```

- 每个工具调用开始/结束都通过 EventBus 发 `EventAgentToolCall` / `EventAgentToolResult`，前端据此实时渲染「正在执行什么工具」（见 [[Agent处理过程记录与前端实时显示]]）。
- 部分工具接入**人工审批**（`internal/agent/approval/gate.go`）与**每工具超时**（`toolExecutionTimeout`）。
- 所有读 KB 内容的工具强制走 `authorizeKnowledgeInSearchTargets` / `authorizeChunkInSearchTargets`（`scope_authorization.go`），把访问限定在当前会话 `SearchTargets` 范围内。

---

## 5. 知识检索类工具详解

> 这类工具是 Agent 的主干，注册门槛：当前会话绑定了 RAG 型 KB（向量/关键词索引）。

### 5.1 `knowledge_search` — 语义搜索

| 项 | 说明 |
| --- | --- |
| 作用 | 用 embedding 做**语义/向量**检索，按「含义、意图、概念相关性」找内容（内部复用 `HybridSearch` + 重排 + MMR） |
| **调用场景** | 概念解释、主题概览、how/why 推理类、比较类、无法用字面关键词命中、措辞不同的同义表达 |
| **不调用场景** | 精确字符串/实体/错误码/产品名的字面查找（交给 `grep_chunks`）；不要传原始长文本当 query |
| 参数 | `queries`（必填，1–5 个语义问句/概念陈述）；`knowledge_base_ids`（可选，限定 `bN` KB） |
| 输出 | 按语义相似度排序的 chunk，每个带 `cN` 分块 ID 与所属 `dN` 文档 ID，可据此做文档级跟进 |
| 文件 | `internal/agent/tools/knowledge_search.go` |

### 5.2 `grep_chunks` — 关键词/正则搜索

| 项 | 说明 |
| --- | --- |
| 作用 | 用单个 POSIX 正则直接在数据库（PG `~*` / MySQL/SQLite `REGEXP`）上做大小写不敏感匹配，等价 `grep -E -i` |
| **调用场景** | 找精确标识符、错误码、产品名、专有名词、反复出现的术语；多个概念用 `\|` 合并成**一条**正则 |
| **不调用场景** | 语义/概念类检索（交给 `knowledge_search`）；不要把同义词拆成多次调用 |
| 参数 | `query`（必填，单个正则，如 `stardust\|skyvault`、`psionic.*engine`、`\\brag\\b`）；注意 JSON 里反斜杠要写 `\\` |
| 输出 | 命中 chunk（带 `cN`、父 `dN`、首个匹配的 `<match>` 片段） |
| 跟进 | FAQ 命中 → `list_knowledge_chunks(faq_id=cN)`；文档命中 → `list_knowledge_chunks(knowledge_id=dN)` 或 `get_document_info(knowledge_ids=[dN])` |
| 文件 | `internal/agent/tools/grep_chunks.go` |

### 5.3 `list_knowledge_chunks` — 查看文档分块

| 项 | 说明 |
| --- | --- |
| 作用 | 读某文档的全部 chunk 原文，或读单个 FAQ 条目/单个分块 |
| **调用场景** | `grep_chunks`/`knowledge_search` 命中后需要**精读**完整上下文时；对某个 `dN` 文档翻页逐块读 |
| **不调用场景** | 尚未定位到具体文档时（先搜索）；已拿到搜索结果且正文已足够时 |
| 参数 | `knowledge_id`（`dN`，配 `limit` 默认 20 / `offset`）/ `faq_id`（`cN`）/ `chunk_id`（`cN`），三者传其一 |
| 输出 | `<knowledge_chunks>` 结构化 XML，含 `total/fetched` 与分页剩余；FAQ 条目含 `<faq><answer>` |
| 文件 | `internal/agent/tools/list_knowledge_chunks.go` |

### 5.4 `get_document_info` — 获取文档元数据

| 项 | 说明 |
| --- | --- |
| 作用 | 批量查文档元数据：标题/描述/来源/文件名/类型/大小/解析状态/chunk 数/自定义元数据 |
| **调用场景** | 需要了解文档基本信息、确认文档是否存在、批量查元数据、了解解析状态 |
| **不调用场景** | 需要文档正文（用 `knowledge_search`/`list_knowledge_chunks`） |
| 参数 | `knowledge_ids`（`dN` 数组）或 `faq_ids`（`cN` 数组），至少传一个；支持并发批量 |
| 输出 | 每个文档一条 `[Entry #n]`，含 chunk 数；FAQ 条目返回标准问/答 |
| **注意** | **需要先拿到 ID**——本身不能「列出所有文档」，需配合 `database_query` 或检索结果使用 |
| 文件 | `internal/agent/tools/get_document_info.go` |

### 5.5 `database_query` — SQL 直查元数据库

| 项 | 说明 |
| --- | --- |
| 作用 | 对关系元数据库发只读 `SELECT`，可查 `knowledge_bases` / `knowledges` / `chunks` 三张表 |
| **调用场景** | **计数/枚举/聚合**类元数据问题：`COUNT(*)` 文档数、按状态分组、按 KB 统计、`SUM(storage_size)`、JOIN 出每个 KB 的文档数等 |
| **不调用场景** | 查正文语义内容（用检索工具）；非 SELECT、非白名单表 |
| 安全 | 自动注入 `tenant_id`、软删除过滤、隐藏 KB 过滤、chunk enabled 过滤、**当前会话作用域过滤**（`internal/utils/inject.go` 的 `WithSearchScopes`） |
| 参数 | `sql`（SELECT 语句；**不要**自己写 tenant_id 条件） |
| **适用范围** | 这是「当前 KB 有多少文档 / 涉及哪些文档 / 总结全库」这类**元数据聚合问题唯一精确的答案来源**（见 README 里的三类问题分析） |
| 文件 | `internal/agent/tools/database_query.go` |

### 5.6 `query_knowledge_graph` — 查询知识图谱

| 项 | 说明 |
| --- | --- |
| 作用 | 查询 Neo4j 实体-关系图谱，探索实体关系与知识网络 |
| **调用场景** | KB 开了图谱抽取（`NEO4J_ENABLE` + `ExtractConfig`）且问「A 和 B 有什么关系 / 谁关联谁」这类关系问题 |
| **不调用场景** | 图谱未开启（工具虽注册但查询会空）；普通语义/字面检索 |
| 参数 | 实体/关系查询参数（带知识作用域鉴权） |
| 文件 | `internal/agent/tools/query_knowledge_graph.go` |

### 5.7 `data_analysis` / `data_schema` — 表格数据分析

| 项 | 说明 |
| --- | --- |
| 作用 | `data_analysis`：把 CSV/Excel 载入 DuckDB，用 SQL 做数据分析/聚合；`data_schema`：查看表格文件（`dN`）的表名、列、行数 |
| **调用场景** | 知识库里有 CSV/Excel 文件，问「统计/求和/分组/趋势」等需要跑 SQL 的问题 |
| **不调用场景** | 非表格类文档；无 `dN` 表格文档时 |
| 参数 | `data_analysis`：SQL + 目标表格；`data_schema`：`knowledge_id`（`dN`） |
| 文件 | `internal/agent/tools/data_analysis.go`、`data_schema.go` |

---

## 6. Wiki 类工具详解

> 注册门槛：作用域内有 Wiki 型 KB（`kb.IsWikiEnabled()`）。Wiki 是 LLM 离线把文档蒸馏成的互联 Markdown 维基，是「全库概览/总结」的高效通道（见 [[LLM Wiki知识库生成与使用逻辑]]）。写类工具在 `SharedAgentReadOnly` 时会被过滤。

| 工具 | 作用 | 调用场景 | 关键参数 |
| --- | --- | --- | --- |
| `wiki_read_page` | 按 slug 读一页/多页完整 Markdown + 元数据 + 链接 | 已知 slug（如 `entity/acme-corp`、`index`）要精读 | `slugs`（数组） |
| `wiki_search` | 用正则搜 Wiki 页面（标题/正文/slug/摘要，`~*` 大小写不敏感） | 不知道精确 slug 时先定位页面 | `queries`（正则数组）、`limit`、`knowledge_base_id` |
| `wiki_read_source_doc` | 在某个源文档内检索/精读，补 Wiki 里被省略的细节 | Wiki 页不够细、要回到原始文档钻取 | 文档 ID + 检索词 |
| `wiki_write_page` | 新建或整体覆盖一页（自动处理出站链接） | 生成新知识页 / 重写错误页 | `slug/title/summary/content/page_type/aliases/source_refs` |
| `wiki_replace_text` | 局部替换页内精确文本 | 一致性小修正 | 目标文本 + 替换文本 |
| `wiki_rename_page` | 改 slug，自动级联更新所有入链 | 页面命名调整 | `slug`、`new_slug` |
| `wiki_delete_page` | 删页并清理入链防死链 | 页面作废 | `slug` |
| `wiki_flag_issue` | 标记页面存在事实错误/混入两个实体/过期 | 用户或模型发现页有问题 | 问题描述 |
| `wiki_read_issue` | 读某页 issue 详情或列待办 issue | 查看已标记的问题 | 页面 slug |
| `wiki_update_issue` | 更新 issue 状态（resolved/ignored 等） | 问题处理后销账 | issue + 新状态 |

文件：`internal/agent/tools/wiki_tools.go`（read/search）+ `wiki_read_source_doc.go` / `wiki_write_page.go` / `wiki_replace_text.go` / `wiki_rename_page.go` / `wiki_delete_page.go` / `wiki_flag_issue.go` / `wiki_read_issue.go` / `wiki_update_issue.go`；路由解析 `wiki_route_resolver.go`。

---

## 7. 其他类工具详解

### 7.1 网页类（需 `WebSearchEnabled`）

| 工具 | 作用 | 调用场景 | 适用范围 |
| --- | --- | --- | --- |
| `web_search` | 联网搜实时信息，结果自动 RAG 压缩并缓存到会话临时 KB | **仅当** `grep_chunks` + `knowledge_search` 都无结果后；需要实时/最新/外部信息时 | 结果带标题、`wN` 页 ID、摘要、正文（上限 `WebSearchMaxResults`） |
| `web_fetch` | 抓取 `web_search` 返回页面的全文并用 LLM 分析 | 摘要不够、需整页验证时 | 只在 `web_search` 之后用；失败则保留搜索证据并降低动态事实置信度 |

文件：`web_search.go`、`web_fetch.go`。

### 7.2 记忆 / 历史类

| 工具 | 作用 | 调用场景 | 适用范围 |
| --- | --- | --- | --- |
| `search_memory` | 查该用户的长期记忆（跨会话） | 需要用户背景/偏好/历史事实时（如「上次你给我的那个配置」） | 随 memory 三开关全开才注入；不在默认白名单 |
| `search_conversations` | 搜该用户自己的历史会话 | 需要回溯之前聊过的内容 | 默认可用；owner 从请求身份取，模型参数无法重定向他人会话 |

文件：`search_memory.go`、`search_conversations.go`。

### 7.3 通用推理 / 规划类

| 工具 | 作用 | 调用场景 | 适用范围 |
| --- | --- | --- | --- |
| `thinking`（`sequentialthinking`） | 动态、反思式分步思考 | 复杂多步问题需要显式推理时 | 默认可用 |
| `todo_write` | 创建/维护结构化任务清单 | 复杂检索/研究任务需要跟踪进度时 | 默认可用（纯聊天且无网页搜索时会被剔除） |

文件：`sequentialthinking.go`、`todo_write.go`。

### 7.4 技能 / 沙箱类（可选，需 skills / 沙箱后端）

| 工具 | 作用 | 调用场景 | 适用范围 |
| --- | --- | --- | --- |
| `read_skill` | 按需读取技能完整指令 | 系统提示词里的技能元数据命中用户意图时，先读再执行 | 渐进式披露：系统提示词只放技能名+描述，正文按需加载 |
| `execute_skill_script` | 在沙箱里执行技能自带的脚本 | 技能需要跑脚本（数据/图表等） | 支持 stdin 传数据 + 参数传路径；读 `/workspace/input`，写 `$WEKNORA_SKILL_OUTPUT_DIR` |
| `list_sandbox_files` | 列出会话沙箱已产出文件 | 链式技能之间要找到上一步产物路径 | 只读 |
| `read_sandbox_file` | 读会话沙箱里已生成的文件 | 查看技能产出的文件内容 | 只读 |
| `shell_exec` | 在**远程隔离沙箱**跑 shell 命令 | 装依赖、探测环境、用 shell 管道/脚本最直接地完成任务 | 命令绝不落在 WeKnora 主机上；非零退出码是正常结果而非错误 |

文件：`skill_read.go`、`skill_execute.go`、`sandbox_ls.go`、`sandbox_read.go`、`shell_exec.go`。

### 7.5 MCP 类（动态）

`RegisterMCPTools`（`mcp_tool.go:411`）把外部 MCP server 暴露的工具以 `mcp_{service}_{tool}` 命名动态注册进同一注册表，Agent 用统一机制调用第三方工具；支持 MCP OAuth（`mcp_oauth.go`）与人工审批。

---

## 8. 关键文件索引

| 关注点 | 文件 |
| --- | --- |
| ReAct 引擎 / 循环 / 工具调用执行 | `internal/agent/engine.go`、`internal/agent/act.go` |
| 工具列表序列化给 LLM | `internal/agent/observe.go`（`buildToolsForLLM`） |
| 系统提示词构建 | `internal/agent/prompts.go` |
| 工具名常量 / 默认白名单 / UI 清单 | `internal/agent/tools/definitions.go` |
| 工具注册表（注册/执行/截断） | `internal/agent/tools/registry.go` |
| 工具注册 + 能力裁剪 | `internal/application/service/agent_service.go`（`registerTools`） |
| 各工具实现 | `internal/agent/tools/*.go` |
| 作用域鉴权 | `internal/agent/tools/scope_authorization.go` |
| SQL 鉴权注入（database_query） | `internal/utils/inject.go` |
| 参数校验/转换/JSON 修复 | `param_validate.go` / `param_cast.go` / `json_repair.go` |
| 短 ID handle 机制 | `internal/modelcontext/`（`Registry`）、`internal/agent/tools/normalize_id.go` |
| 人工审批 | `internal/agent/approval/gate.go` |
| 上下文压缩 | `internal/agent/token/compress.go` |

---

## 9. 提示词体系：使用细节与写法

工具是「Agent 的手」，Prompt 是「Agent 的脑」。LLM 会不会正确调用工具、调用顺序对不对、答案准不准，**全由 prompt 决定**。这一节把 Agent 相关的 prompt 拆成五层，讲清楚每一层「写在哪、谁写的、写的是什么、怎么写出来的」。

### 9.1 五层 prompt 的分工

| 层 | 内容 | 来源 | 生命周期 | 静态/动态 |
| --- | --- | --- | --- | --- |
| ① 系统提示词 | 角色/使命/硬约束/工作流/工具准则/输出标准 | `config/prompt_templates/agent_system_prompt.yaml`（7 模板按 mode 或 system_prompt_id 选） | 每次 `Execute` 构建一次 | 静态骨架 + 占位符 |
| ② 协议提示词 | source handle 协议 + 引用协议 + res:// 资源协议 | `internal/modelcontext/`（代码内写死，不可配） | 追加在系统提示词末尾 | 静态 |
| ③ 运行时上下文 | `<runtime_context>`：绑定 KB、置顶文档、当前时间、communication/answer instruction | `observe.buildRuntimeContextBlock` | **每轮**注入当前 user 消息，**不落历史** | 动态 |
| ④ `<must_use>` 块 | @MCP/@Skill 强提醒 | `observe.buildMustUseBlock` | 每轮，仅当用户 @mention 时 | 动态 |
| ⑤ 工具 description/schema | 每个工具的「使用契约」 | 各 `tools/*.go` 的 `BaseTool`（代码内写死） | 作为 function calling 的 `tools` 参数 | 静态 |

一句话记忆：**①②⑤ 是「写在代码/模板里的静态契约」，③④ 是「每次请求现场拼的动态数据」**。系统提示词教 Agent「该怎么想」，工具 description 教 Agent「工具怎么用」，运行时上下文告诉 Agent「这一轮你面对的是什么」。

### 9.2 系统提示词模板：7 个模板 + 两种路由

`agent_system_prompt.yaml` 里有 7 个模板，每个有 `id`、`mode`、`content`（部分带 `default: true`）：

| id | mode | 用途 / 典型工具 | 核心设计 |
| --- | --- | --- | --- |
| `pure_agent` | `pure` | 无 KB 的通用助理 | Role / Mission / Tool Guidelines / 保密条款 |
| `progressive_rag_agent` | `rag`（**default: true**） | 默认渐进式 RAG（有 KB） | Evidence-First + Assess-Reconnaissance-Plan-Execute + 严格检索顺序 |
| `data_analyst` | `data_analyst` | 数据分析（`data_schema` + `data_analysis`） | Schema First、Read-Only、DuckDB SQL 规范 |
| `wiki_researcher` | `wiki_researcher` | Wiki 问答（只读 + 报障） | Search-Read-Expand 循环、`wiki_flag_issue` |
| `wiki_fixer` | `smart-reasoning` | Wiki 修订（含全部写操作） | 先读 issue 再定修复策略、一轮改完 |
| `hybrid_rag_wiki_agent` | `smart-reasoning` | 混合 RAG + Wiki（双表面） | `capabilities="..."` 属性路由、WIKI_KBS/CHUNK_KBS 分区 |
| `skill_installer` | `smart-reasoning` | 安装技能到沙箱镜像 | 隔离安装、镜像回写 |

**两种选模板的路由**：

1. **默认路由（按 mode）**：`BuildSystemPromptWithOptions`（`internal/agent/prompts.go:351`）内部三段式——

   ```go
   if 有自定义模板(systemPromptTemplate[0]) → 用它
   else if len(knowledgeBases)==0            → GetPureAgentSystemPrompt (mode="pure")
   else                                      → GetProgressiveRAGSystemPrompt (mode="rag")
   ```

   `Get*SystemPrompt` 最终调 `config.DefaultTemplateByMode(templates, mode)`：优先 `Mode==mode && Default`，否则 `Mode==mode`，再否则 `DefaultTemplate`。所以「有 KB → `progressive_rag_agent`，无 KB → `pure_agent`」是兜底默认。

2. **预置 Agent 路由（按 id）**：`data_analyst` / `wiki_researcher` / `wiki_fixer` / `hybrid_rag_wiki_agent` / `skill_installer` 这几个**不是靠 mode 选的**，而是通过 `builtin_agents.yaml` / `agent_type_presets.yaml` 里的 `system_prompt_id` 字段，用 `resolveBuiltinAgentPromptIDs`（`internal/config/config.go:1048`）按 `id` 反解出模板 `content` 填入 Agent 的 `SystemPrompt`。用户建 Agent 时选「数据分析师 / 维基问答 / 维基修订」等预设，就命中对应的 `id`。这里 `mode` 字段对这些模板只起分组/占位作用。

> 用户也可在 Agent 编辑器里写**完全自定义**的 `system_prompt`（不走任何模板），或指定 `system_prompt_id` 引用模板。自定义 prompt 优先级最高（见上三段式第一分支）。

### 9.3 系统提示词的组装链（谁拼出来的）

`engine.buildSystemPrompt`（`internal/agent/engine.go`）的最终产物 = 三块拼接：

```text
buildSystemPrompt
  = BuildSystemPromptWithOptions(...)          // ① 基础系统提示词（模板 + 占位符）
  + memoryPrompt                              // ② 长期记忆背景（MEMORY_RECALL 结果）
  + modelContext.ProtocolPrompt()             // ③ 协议提示词（handle/引用/res://）
```

其中 `BuildSystemPromptWithOptions` 内部还做了两件事：

1. **占位符渲染**（`renderPromptPlaceholdersWithStatus`，`prompts.go:308`）：

   | 占位符 | 值 |
   | --- | --- |
   | `{{knowledge_bases}}` | `formatKnowledgeBaseList` 渲染的 `<knowledge_bases>` XML（id/name/type/doc_count/capabilities） |
   | `{{web_search_status}}` | `"Enabled"` / `"Disabled"` |
   | `{{current_time}}` | 当前时间（RFC3339） |
   | `{{language}}` | 用户语言名（如 `Chinese (Simplified)`） |
   | `{{skills}}` | 置空——技能元数据单独追加，见下 |

2. **技能元数据追加**（Progressive Disclosure Level 1）：`options.SkillsMetadata` 非空时，用 `formatSkillsMetadata` 在 prompt 末尾追加技能「名字 + 一句话描述」，**不塞完整技能正文**（正文靠 `skill_*` 工具按需加载，即「渐进式披露」）。

### 9.4 每轮注入的运行时上下文（动态部分）

系统提示词是「静态骨架」，但 Agent 每一轮面对的知识库、置顶文档、沟通要求都不一样，这些**动态信息不写进系统提示词，而是拼进本轮 user 消息**，由 `observe.go` 生成：

```text
RenderUserTurnContent(query) =
      <runtime_context scope="this_turn">…</runtime_context>
    + <must_use>…</must_use>                  （仅当 @MCP/@Skill 时）
    + query                                    （用户原始提问）
```

`buildRuntimeContextBlock` 渲染的 `<runtime_context scope="this_turn">` 内含：

| 子块 | 含义 |
| --- | --- |
| `<current_time>` / session | 本轮时间与会话标识 |
| `<bound_knowledge_bases>` | 本轮绑定的知识库（用户 `@` 选中或 Agent 配置默认） |
| `<pinned_documents>` | 置顶/固定的文档 |
| `<communication_instruction>` | 沟通要求（如强制用某种语言/语气） |
| `<answer_instruction>` | 答案要求 |

`buildMustUseBlock` 渲染的 `<must_use>` 块，是在用户用 `@MCP` / `@Skill` 点名时，提示「本轮必须调用这些被 @ 的工具/技能」。

**关键设计**：`scope="this_turn"` 表示这些块**只注入当前这一轮**，不写进对话历史——下一轮会重新按当前状态生成。这避免了两类问题：① 历史被上一轮的绑定 KB 污染；② 多轮累积导致 token 膨胀。

### 9.5 协议提示词：短 ID handle 与引用协议

`modelContext.ProtocolPrompt()`（`internal/modelcontext/registry.go`）追加两块「协议」，是 Agent 正确使用 `cN`/`dN`/`wN` 短 ID 和引用 `<ref>` 的依据：

**① source handle 协议**（`citations.go` 的 `sourceHandleProtocolPrompt`）：检索结果里出现的 `cN` 是知识块、`wN` 是网页、`dN` 是文档、`bN` 是知识库、`iN` 是 issue、`res://NNNN` 是资源——这些是**请求局部的短 ID**，Agent 必须原样回传给需要 ID 的工具（如 `list_knowledge_chunks(chunk_ids=["c3"])`），不能自己编造。

**② 引用协议**（分开关两种）：

- 开启引用（`citationEnabledProtocolPrompt`）：要求用 `<ref id="cN"/>` 标注答案来源，后端再把它展开成前端可点的 `<kb/>` / `<web/>` 卡片。
- 关闭引用（`citationDisabledProtocolPrompt`）：要求**不要**输出引用标记。

配合 `registerRuntimeReferences`：每次工具返回结果时，把 handle → 实体 的映射登记进 `modelcontext.Registry`，最终答案里的 `<ref id="cN"/>` 才能被反向解析成真实来源。这就是「答案可溯源、前端可渲染引用卡片」的底层机制。

### 9.6 工具 description 怎么写（写给 LLM 的「使用契约」）

每个工具向 LLM 暴露的**不是代码，而是一段 description + JSON schema**（`buildToolsForLLM` 序列化 `chat.Tool`）。LLM 只靠这段文字判断「该不该用、怎么用」，所以 description 的写法直接决定调用准确率。WeKnora 的写法有固定套路：

1. **一句话说清做什么**（Purpose）——避免 LLM 误判；
2. **显式写「不做什么」**（Do NOT / This tool does NOT）——给 LLM 划边界，防止滥用；
3. **说明适用/不适用场景**——如 `grep_chunks` 强调「先于 knowledge_search 做精确关键词初筛」；
4. **参数逐项说明 + 类型约束**——JSON schema 本身即参数契约，`CastParams`/`ValidateParams` 再兜底；
5. **给调用示例**——部分工具 description 内嵌 example，帮 LLM 少走弯路。

典型例子：

- **`knowledge_search`**：声明「语义/混合检索 + 重排，返回按相关度排序的 chunk」，同时写清 `queries` 一次可传多个子查询；
- **`web_search`** 的 **KB-First 规则**：description 里写死「只有当知识库检索结果不足以回答、或用户明确要实时/外部信息时才用」，从源头阻止 Agent 一上来就联网；
- **`database_query`**：description 说明它是对**关系型元数据表**（`knowledge_bases`/`knowledges`/`chunks`）做**只读 SQL SELECT**，并列出可查的 schema 字段——这正是「知识库里有多少份文档」「涉及哪些项目」这类聚合问题能被回答的前提（详见 §5.5）。

**为什么 description 是「契约」而非「文档」**：它会被喂进每一轮 function calling 的上下文，既占 token 又影响决策。所以写 description 的原则是「**信息密度高、边界清晰、不含实现细节**」——LLM 不需要知道工具内部怎么实现，只需要知道「什么时候用、喂什么参数、得到什么」。

### 9.7 系统提示词的写作范式（「怎么写的」拆解）

以默认的 `progressive_rag_agent` 为样板，WeKnora 的系统提示词有清晰的七段结构，每段对应一个写作目的：

| 段落 | 目的 | 典型写法 |
| --- | --- | --- |
| Role | 定角色 | 「你是一个严谨的知识库问答助理」 |
| Critical Constraints | 兜底硬约束 | 「ABSOLUTE RULES」+ 编号：证据优先、强制 Deep Read、KB 优先、永不凭记忆答、保密条款 |
| Workflow | 定流程 | Assess → Reconnaissance → Plan → Execute → Synthesis 五阶段 |
| Core Retrieval Strategy | 定检索顺序 | 「Strict Sequence」：`grep_chunks → knowledge_search → list_knowledge_chunks(MANDATORY) → query_knowledge_graph(可选) → web_search(兜底)` |
| Tool Selection Guidelines | 定工具分工 | 「Index / Eyes / Manager / Conscience」比喻映射到具体工具 |
| Final Output Standards | 定输出 | 富媒体、图片用 ASCII 半角括号 |
| System Status | 注动态 | `{{web_search_status}}`、`{{language}}` |

**八条可复用的写作技巧**：

1. **用大写词强调不可违反的约束**：`ABSOLUTE RULES`、`MANDATORY`、`STRICT SEQUENCE`——LLM 对大写/强语气更敏感，能显著提高约束被遵守的概率。
2. **把「强制步骤」写进流程而非期望**：`list_knowledge_chunks` 标 `(MANDATORY)`，因为「检索到 chunk 后必须先 Deep Read 原文、不许只靠检索摘要回答」是这条 prompt 的命门。
3. **用比喻给工具分组**：`grep_chunks`/`knowledge_search` = "Index"（查目录）、`list_knowledge_chunks` = "Eyes"（读原文）、`todo_write` = "Manager"（列计划）、`thinking` = "Conscience"（反思）——让 LLM 在「选哪个工具」时按直觉对应，而非背工具名。
4. **显式写禁则/反例**：如「不要向用户暴露工具名、内部 ID、短 ID handle」——既保护用户体验，也防止 LLM 把 `c3` 这种内部标识吐给用户。
5. **用占位符承载运行时变量**：`{{web_search_status}}` 让同一条模板在「网页搜索开/关」两种部署下都成立，避免维护两份 prompt。
6. **结构化 XML 块承载数据**：`<knowledge_bases>`、`<runtime_context>` 把「机器可枚举的数据」和「自然语言指令」分层，指令稳定、数据可变。
7. **Prompt Confidentiality 条款**：明确要求「不要复述 prompt、不要暴露系统配置」，是面向 C 端产品的常见安全护栏。
8. **few-shot 正反例**（在 `rewrite.yaml` 这类非 Agent prompt 里更常见）：给「正确改写 vs 错误改写」让 LLM 照着学，比纯规则更稳。

> 这几条技巧里，**第 2、3 条是 WeKnora 区别于普通 RAG prompt 的关键**——它把「必须先精读再回答」这件事，从「期望」升级成了「流程里的强制步骤 + 工具分工比喻」，这是它检索质量明显更稳的主因之一。

### 9.8 prompt 相关关键文件

| 关注点 | 文件 |
| --- | --- |
| 系统提示词模板（7 个，含 mode/占位符） | `config/prompt_templates/agent_system_prompt.yaml` |
| 系统提示词构建 + 占位符渲染 + 技能元数据 | `internal/agent/prompts.go` |
| 运行时上下文 `<runtime_context>` / `<must_use>` | `internal/agent/observe.go`（`buildRuntimeContextBlock` / `buildMustUseBlock` / `RenderUserTurnContent`） |
| 协议提示词（handle / 引用 / res://） | `internal/modelcontext/registry.go`、`citations.go`、`sources.go` |
| 工具 description/schema（LLM 的工具契约） | `internal/agent/tools/*.go`（各工具的 `BaseTool`） |
| 工具序列化给 LLM | `internal/agent/observe.go`（`buildToolsForLLM`） |
| 预置 Agent 的 `system_prompt_id` | `config/builtin_agents.yaml`、`config/agent_type_presets.yaml` |
| `system_prompt_id` → 模板内容解析 | `internal/config/config.go`（`resolveBuiltinAgentPromptIDs`）、`internal/types/agent_type_preset.go` |
| 通用 prompt 模板格式（与本文互补） | 见 [[Prompt模板格式与设计]] |

> 通用 prompt 模板系统的字段结构、加载规则、i18n 已在 [[Prompt模板格式与设计]] 里讲，本文只聚焦「Agent 场景下 prompt 如何驱动工具调用」。

---

## 附：一次「工具调用决策」的时间线示例

```text
用户: "帮我查一下知识库里关于 RAG 的优势，以及最新的相关新闻"
  │
  1. AgentEngine.Execute
       ├─ buildSystemPrompt（渐进式 RAG prompt + 记忆 + 协议 prompt）
       ├─ buildToolsForLLM（所有已注册工具的 name/description/schema）
       └─ executeLoop 启动
  │
  2. Round 1: callLLMWithRetry
       LLM 读 description 判断：
         - 知识库内容 → 调 knowledge_search(queries=["RAG 的优势是什么"])
         - 精确/实时 → 需先检索 KB，再视结果决定是否 web_search
       └─ 返回 1 个 ToolCall {name: "knowledge_search", arguments: {...}}
  │
  3. Act: runToolCall
       ├─ CastParams + ValidateParams
       ├─ 作用域鉴权（SearchTargets）
       ├─ knowledge_search.Execute（HybridSearch + rerank + MMR）
       └─ 结果截断 → ToolResult
  │
  4. Observe: appendToolResults 把结果作为 role:tool 写回
  │
  5. Round 2: LLM 看到检索结果后：
       ├─ 内容足够且非实时 → 直接产出最终答案（无 ToolCall → 结束）
       └─ 若仍需实时新闻 → 再调 web_search → 下一轮 → 最终答案
  │
  6. EventBus 流式回传 thinking / tool_call / answer 到前端
```
