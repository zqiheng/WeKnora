# WeKnora Langfuse 链路追踪

> 本文梳理 WeKnora 的 **Langfuse 可观测性（Tracing）** 实现——它如何把一次「用户提问 → 检索 → LLM 回答」的完整调用链，跨 HTTP 与 asynq 异步任务、跨多个模型调用，缝合成一棵可在 Langfuse UI 里浏览的 trace 树。
>
> 核心结论：WeKnora **没有直接调 Langfuse 的 REST/上报协议**（Langfuse 官方没有 Go SDK），而是站在 **OpenTelemetry Go SDK** 之上，用 OTLP/HTTP exporter 直写 Langfuse v3+ / LiteFuse 的 OTel 端点，并在请求头里**伪装成 langfuse-python v4** 以通过服务端校验。
>
> 代码基准：`internal/tracing/langfuse/`（核心包）+ `internal/types/tracing.go`（跨进程载体）+ `internal/models/*/langfuse_wrapper.go`（模型装饰器）。

---

## 目录

1. [总览：OTel SDK + 伪装 Python SDK 直写 Langfuse](#1-总览otel-sdk--伪装-python-sdk-直写-langfuse)
2. [配置：全走环境变量，默认关闭](#2-配置全走环境变量默认关闭)
3. [核心架构：Manager + 三种观测类型](#3-核心架构manager--三种观测类型)
4. [五个精妙机制](#4-五个精妙机制)
5. [跨进程传播：HTTP 与 asynq 两段拼接](#5-跨进程传播http-与-asynq-两段拼接)
6. [模型层：5 个 Generation 装饰器](#6-模型层5-个-generation-装饰器)
7. [业务埋点：Span 覆盖问答全链路](#7-业务埋点span-覆盖问答全链路)
8. [输出摘要：避免 trace 爆炸](#8-输出摘要避免-trace-爆炸)
9. [生命周期：初始化与优雅关闭](#9-生命周期初始化与优雅关闭)
10. [关键文件索引](#10-关键文件索引)

---

## 1. 总览：OTel SDK + 伪装 Python SDK 直写 Langfuse

WeKnora 是 Go 项目，Langfuse 没有一等公民的 Go SDK，所以它**没有手写 Langfuse 的上报协议**，而是：

```text
业务代码 → StartTrace/StartSpan/StartGeneration
              ↓
        OpenTelemetry Go SDK (TracerProvider + BatchSpanProcessor)
              ↓  OTLP/HTTP exporter
        POST /api/public/otel/v1/traces  (Langfuse OTel 端点)
```

为了通过服务端「必须是 Python SDK >= 4.0」的校验，exporter 在请求头里**伪装成 langfuse-python v4**（`internal/tracing/langfuse/exporter.go:26`）：

```go
"Authorization":                "Basic " + base64(publicKey:secretKey),
"x-langfuse-ingestion-version": "4",     // ← 关键 gate，缺了服务端返回 400
"x-langfuse-sdk-name":          "python",
"x-langfuse-sdk-version":       "4.0.0",
```

一句话：**OTel SDK 负责 span 采集/缓冲/采样 + OTLP/HTTP 直写 Langfuse 的 OTel 端点 + 用 langfuse-python 的 attribute 语义约定让 LiteFuse 正确索引**。

---

## 2. 配置：全走环境变量，默认关闭

`internal/tracing/langfuse/config.go` 的 `LoadConfigFromEnv` 读 `LANGFUSE_*` 环境变量，**完全 opt-in**：

| 环境变量 | 默认 | 作用 |
| --- | --- | --- |
| `LANGFUSE_HOST` | `https://cloud.langfuse.com` | 服务地址 |
| `LANGFUSE_PUBLIC_KEY` / `SECRET_KEY` | 空 | 项目凭据（Basic Auth） |
| `LANGFUSE_ENABLED` | 凭据存在即 `true` | 总开关 |
| `LANGFUSE_RELEASE` / `ENVIRONMENT` | 空 | 附加到每个 trace，供 UI 过滤 |
| `LANGFUSE_FLUSH_AT` / `FLUSH_INTERVAL` | 15 条 / 3s | 批量冲刷 |
| `LANGFUSE_QUEUE_SIZE` | 2048 | 内存缓冲上限（防止端点不可达时内存爆炸） |
| `LANGFUSE_SAMPLE_RATE` | 1.0 | 采样率 |
| `LANGFUSE_REQUEST_TIMEOUT` | 10s | 单批上报超时 |

设计要点：

- **凭据不进 YAML**（避免密钥泄露到配置文件），只走环境变量；
- **关闭时零成本 no-op**——所有公开入口点都容忍 `*Manager == nil`，调用方可以无条件接线，无需在业务里写 `if enabled`。

---

## 3. 核心架构：Manager + 三种观测类型

`internal/tracing/langfuse/manager.go` 的 `Manager` 是门面单例，内部持有一个 OTel `TracerProvider`：

```text
Manager (单例)
  └─ TracerProvider
       ├─ Resource: service.name=weknora, public_key, environment, release
       ├─ Sampler: ParentBased(TraceIDRatioBased(sampleRate))
       └─ SpanProcessor
            ├─ 生产: BatchSpanProcessor → OTLP/HTTP exporter → Langfuse
            └─ 测试: SimpleSpanProcessor → InMemoryExporter（确定性断言）
```

三种观测类型（`tracer.go`，都是 OTel span 的薄封装）：

| 类型 | 含义 | 例子 |
| --- | --- | --- |
| **Trace** | 根观测，= 一次「请求」/「对话轮次」，ID 就是 W3C trace id | HTTP 请求、asynq 独立任务 |
| **Span** | 非 LLM 的工作单元 | pipeline 阶段、检索、重排、Agent 轮次 |
| **Generation** | 一次模型调用 | chat / embedding / rerank / VLM / ASR |

它们在 Langfuse UI 里的父子关系**靠 OTel 的 span context 自动建立**（`trace.SpanFromContext` 决定 parent），不靠自绘的 ID 拼接。

**attribute 命名**（`events.go:24`）直接镜像 langfuse-python v4 的语义约定：

```go
"langfuse.observation.type"           // trace | span | generation
"langfuse.observation.input/output/metadata/model.name/model.parameters/usage_details"
"langfuse.observation.completion_start_time"   // TTFB
"langfuse.trace.name/input/output/metadata/tags"
"user.id" / "session.id" / "langfuse.environment" / "langfuse.release"
```

结构化字段（input/output/metadata/usage）统一 `jsonAttr` 序列化成 **JSON 字符串**塞进 span attribute，和 langfuse-python 存法一致。

---

## 4. 五个精妙机制

### 4.1 `reestablishParentSpan`（`tracer.go:186`）

Go 里一个坑：`logger.CloneContext` 每请求会重建一个瘦身 context，可能丢掉 OTel 的活跃 span。于是当 ctx 上带着 `*Trace` 但 OTel span 没了，这个方法就把 root span 重新塞回 context 当 parent——否则下游的 generation 会各开一个独立 root，**一个 HTTP 请求碎成 N 条不相关的 trace**。

### 4.2 `autoTrace`（`tracer.go:207,264`）

如果 `StartSpan`/`StartGeneration` 时 ctx 上没有任何 trace，就自动开一个浅 root，并持有句柄，`Finish` 时把它一并 End 掉——否则 child span 的 parent 会指向一个「从未导出的幽灵 span」。

### 4.3 `ResumeTrace`（`tracer.go:153`）

从外部给的 W3C trace id 恢复一个 `*Trace`（不新建 root），供异步任务「嫁接」到已存在的 trace 上。

### 4.4 `MarkCompletionStart`（`tracer.go:312`）

流式生成收到第一个 token 时打点，Langfuse 把它展示为 time-to-first-token（TTFB）。

### 4.5 `mergeMetadata`（`tracer.go:324`）

`Finish` 时的 metadata **合并**（而非覆盖）到 `Start` 时的 metadata——这样「打开时才知道的关联字段（request_id、http.method）」和「结束时才知道的结果字段（outcome、duration）」能共存。

---

## 5. 跨进程传播：HTTP 与 asynq 两段拼接

### 5.1 HTTP 侧（`middleware.go`）

`GinMiddleware` 在 `router.go:193` 注册（Auth 之后）：

1. `propagator.Extract` 从请求头提取 W3C `traceparent`——**如果上游调用方（如 sop3）带了 traceparent，WeKnora 的 trace 就继承它的 trace id**，两段运行在 LiteFuse 里落到同一棵树。
2. 按 `shouldTrace` **白名单**过滤（`:85`）——只 trace 会触发 LLM 工作的路径（`/knowledge-chat`、`/agent-chat`、`/knowledge-search`、`generate_title`、各种 ingestion 的 POST/PUT、initialization 测试、evaluation、wiki auto-fix…），**故意排除纯 GET 列表/健康检查**以保持信噪比。
3. trace name = `"METHOD /full/path"`，metadata 带 `http.method/path/query/request_id`。
4. 请求结束后 `trace.Finish({status, response.size})`。

### 5.2 asynq 异步侧（`asynq.go` + `types/tracing.go`）

这是把「HTTP 请求」和「它入队的异步处理任务」缝成同一棵 trace 的关键：

- **入队时 `InjectTracing`**（`asynq.go:28`）：把当前 W3C `traceparent` + user/session 标签写到 payload。靠 `types.TracingContext`（`types/tracing.go:20`）内嵌进各 payload struct，字段统一 `lf_` 前缀 + `omitempty`，**向后兼容旧 payload**。散布在几十处入队点（`knowledge_create.go`、`knowledge_process.go`、`wiki_ingest.go` 等）。
- **worker 端 `AsynqMiddleware`**（`asynq.go:82`，注册于 `router/task.go:260`）：
  - 有 `traceparent` → `propagator.Extract` 恢复上游 trace（worker 的 span 变成 HTTP trace 的孩子）；
  - 无 → 开独立 trace（`asynq.<task_type>`）；
  - 再包一层 span（带 `task_id/queue/retry/payload_bytes`），handler 里的 embedding/VLM/chat 等 generation 自动挂到它下面。

`types.TracingContext` 放在 types 包而非 langfuse 包，是为了让 asynq payload 不用 import langfuse，保持 langfuse 是叶子依赖。

---

## 6. 模型层：5 个 Generation 装饰器

每个模型类型都有一个 `langfuse_wrapper`，在模型构造函数里**条件安装**（`GetManager().Enabled()` 时才包）：

| 模型 | wrapper | 安装点 |
| --- | --- | --- |
| Chat | `chat/langfuse_wrapper.go` | `chat.go:154` `wrapChatLangfuse` |
| Embedding | `embedding/langfuse_wrapper.go` | `embedder.go:104` |
| Rerank | `rerank/langfuse_wrapper.go` | `reranker.go` |
| VLM | `vlm/langfuse_wrapper.go` | `vlm.go:94` |
| ASR | `asr/langfuse_wrapper.go` | `asr.go:64` |

以 chat 为例（`chat/langfuse_wrapper.go`）：

- `Chat`：包一个 `chat.completion` generation，记录 prompt/response/token usage/`call_purpose`。
- `ChatStream`：包 `chat.completion.stream`，起 goroutine 消费流，**累积 content/reasoning/tool_calls/usage**，第一个 token 时 `MarkCompletionStart`，流结束后一次性 `Finish`。`tool_calls` 会先 `snapshot` 一份（因为下游会在原地解码参数，不能污染观测）。
- 额外通过 `types.LLMCallMetadataFromContext(ctx)` 带出 `call_purpose`（这次 LLM 调用是干嘛的）和 `prompt_prefix_fingerprint`（prompt 前缀指纹，用于在 UI 里聚合同类调用）。

---

## 7. 业务埋点：Span 覆盖问答全链路

业务代码用 `langfuse.GetManager().StartSpan(...)` 埋点（`GetManager()` 返回 nil 时是 no-op，业务无感知）：

| 埋点 | 位置 | span 名 |
| --- | --- | --- |
| QA 请求装配 | `session_knowledge_qa.go:39` | `qa.setup` |
| pipeline 各阶段 | `session_knowledge_qa.go:706,899` | 各 stage |
| 混合检索 | `knowledgebase_search.go:202` | `retrieve`（Finish 用 `SummarizeRetrieveOutput`） |
| 网页搜索 | `chat_pipeline/search.go:611` | web |
| 重排 | `chat_pipeline/rerank.go:91` | rerank（Finish 用 `SummarizeSearchResults`） |
| Agent 执行 | `agent/engine.go:238` | `agent.execute` |
| Agent 单轮 | `agent/engine.go:483` | round |
| 长期记忆召回 | `memory/service.go:114,910`、`recall_trace.go`、`search.go` | recall / conditioning / search |

于是一个完整 Agent 问答的 span 树大致是：

```text
Trace: "POST /api/v1/agent-chat/:session_id"
 ├─ Span: qa.setup
 ├─ Span: agent.execute
 │    ├─ Generation: chat.completion.stream (LLM 规划)
 │    ├─ Span: round
 │    │    ├─ Span: retrieve  (知识库检索)
 │    │    │    └─ Generation: embedding (query 向量化)
 │    │    ├─ Span: rerank    (重排)
 │    │    │    └─ Generation: rerank
 │    │    └─ ...
 │    └─ Generation: chat.completion.stream (最终答案)
 └─ (asynq 入队的后处理任务 → 跨进程挂回同一 trace)
```

---

## 8. 输出摘要：避免 trace 爆炸

`retrieval_obs.go` / `memory_obs.go` 提供一组 `Summarize*` 助手，把结果压缩成**预览**而非全量：

- `SummarizeRetrieveOutput` → 按 retriever/engine 分组计数 + top 25 hits（截断 160 runes）
- `SummarizeSearchResults` → 带 `composite_score/match_type/faq_boosted` 的排名预览
- `SummarizeRankScores` / `SummarizePassagePreviews` → 重排输入输出对齐
- `SummarizeMemoryRecallOutput` → 召回的记忆条目预览（topic/importance 截断）

`asynq.go:163` 的 `spanInputFromPayload` 同理，只发 payload 前 ~1KB 预览，防止 FAQ 导入这类大 payload 塞爆 trace。

---

## 9. 生命周期：初始化与优雅关闭

- **初始化**：`container.go:545` `initLangfuse()` → `LoadConfigFromEnv` + `Init`；无凭据返回 disabled manager，永不报错。
- **优雅关闭**：`container.go:1496` `registerLangfuseCleanup`，5 秒超时 `Shutdown` 冲刷缓冲中的 span。

---

## 10. 关键文件索引

| 关注点 | 文件 |
| --- | --- |
| 配置（env 读取/校验） | `internal/tracing/langfuse/config.go` |
| 单例 Manager / TracerProvider | `internal/tracing/langfuse/manager.go` |
| 三种观测类型 + 生命周期 | `internal/tracing/langfuse/tracer.go` |
| OTLP exporter（伪装 Python SDK） | `internal/tracing/langfuse/exporter.go` |
| attribute 语义常量 | `internal/tracing/langfuse/events.go` |
| context 传递 | `internal/tracing/langfuse/context.go` |
| 跨进程 payload 载体 | `internal/types/tracing.go` |
| HTTP 中间件 + 白名单 | `internal/tracing/langfuse/middleware.go` |
| asynq 注入/恢复 | `internal/tracing/langfuse/asynq.go` |
| 检索/记忆输出摘要 | `internal/tracing/langfuse/retrieval_obs.go`、`memory_obs.go` |
| 模型装饰器 | `internal/models/{chat,embedding,rerank,vlm,asr}/langfuse_wrapper.go` |
| 初始化/清理 | `internal/container/container.go:545,1496` |
| 中间件注册 | `internal/router/router.go:193`、`internal/router/task.go:260` |

---

## 附：设计意图小结

这是一套相当工程化的实现，几个可复用的点：

1. **借力 OTel 而非重造轮子**——span 采集/缓冲/采样/导出全部复用成熟 SDK，项目只写观测语义封装；
2. **伪装 Python SDK 过直写 gate**——用 `x-langfuse-ingestion-version: 4` 等请求头骗过服务端校验，比手写 Langfuse REST 上报省一个协议层的维护成本；
3. **W3C traceparent 做跨进程/跨服务关联**——HTTP 与 asynq 用标准 trace context 缝合，天然支持上游 sop3 的 trace id 透传；
4. **白名单控噪、摘要控体积**——只 trace 会触发 LLM 的路径，输出只发预览，避免 trace 爆炸与信噪比下降；
5. **处处 nil-safe 的 no-op 降级**——凭据缺失时整条链路零成本，业务代码无需 `if enabled` 判断。
