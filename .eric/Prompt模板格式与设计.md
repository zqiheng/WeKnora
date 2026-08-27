# WeKnora Prompt 模板：格式、设计与加载

> 本文梳理 WeKnora 的 prompt 模板系统的**数据结构、字段含义、设计动机、加载规则、占位符、fallback 的 mode 区分、i18n 的 locale 解析优先级**。
>
> 代码基准：`internal/config/config.go`（结构 + 加载）+ `internal/types/placeholder.go`（占位符）+ `config/prompt_templates/`（11 个模板 YAML）。

---

## 目录

1. [总览：统一的数据结构](#1-总览统一的数据结构)
2. [PromptTemplate 每个字段的含义](#2-prompttemplate-每个字段的含义)
3. [为什么这么设计](#3-为什么这么设计)
4. [加载规则：loadPromptTemplates](#4-加载规则loadprompttemplates)
5. [占位符系统](#5-占位符系统)
6. [prompt 的语言设计（中英文混用）](#6-prompt-的语言设计中英文混用)
7. [fallback.yaml 的 mode 区分](#7-fallbackyaml-的-mode-区分)
8. [i18n 的 locale 解析优先级](#8-i18n-的-locale-解析优先级)
9. [关键文件索引](#9-关键文件索引)

---

## 1. 总览：统一的数据结构

所有 prompt 模板（改写、意图、上下文、兜底、摘要、问题生成、图谱抽取…）都用**同一个结构** `PromptTemplate`，按类型分组存到 `PromptTemplatesConfig`。

```go
// PromptTemplatesConfig 提示词模板配置
type PromptTemplatesConfig struct {
    SystemPrompt         []PromptTemplate `yaml:"system_prompt"`
    ContextTemplate      []PromptTemplate `yaml:"context_template"`
    Rewrite              []PromptTemplate `yaml:"rewrite"`
    Fallback             []PromptTemplate `yaml:"fallback"`
    GenerateSessionTitle []PromptTemplate `yaml:"generate_session_title"`
    GenerateSummary      []PromptTemplate `yaml:"generate_summary"`
    KeywordsExtraction   []PromptTemplate `yaml:"keywords_extraction"`
    AgentSystemPrompt    []PromptTemplate `yaml:"agent_system_prompt"`
    GraphExtraction      []PromptTemplate `yaml:"graph_extraction"`
    GenerateQuestions    []PromptTemplate `yaml:"generate_questions"`
    IntentPrompts        []PromptTemplate `yaml:"intent_prompts"` // template ID = intent value
}
```

**每种 Prompt 类型对应一个 YAML 文件**，一个文件管一类：

| 配置字段 | YAML 文件 | 用途 |
| --- | --- | --- |
| `SystemPrompt` | `system_prompt.yaml` | 主回答 system prompt |
| `ContextTemplate` | `context_template.yaml` | 上下文组装模板 |
| `Rewrite` | `rewrite.yaml` | 改写 + 意图分类 |
| `Fallback` | `fallback.yaml` | 无结果兜底（fixed/model） |
| `IntentPrompts` | `intent_prompts.yaml` | 意图专用 system prompt（ID=意图值） |
| `GraphExtraction` | `graph_extraction.yaml` | 实体/关系抽取（遗留） |
| 其余 | `generate_*.yaml` 等 | 摘要/标题/问题生成 |

---

## 2. PromptTemplate 每个字段的含义

`internal/config/config.go:335`：

```go
type PromptTemplate struct {
    ID               string                        `yaml:"id"`
    Name             string                        `yaml:"name"`
    Description      string                        `yaml:"description"`
    Content          string                        `yaml:"content"`
    User             string                        `yaml:"user"`
    HasKnowledgeBase bool                          `yaml:"has_knowledge_base"`
    HasWebSearch     bool                          `yaml:"has_web_search"`
    Default          bool                          `yaml:"default"`
    Mode             string                        `yaml:"mode"`
    I18n             map[string]PromptTemplateI18n `yaml:"i18n"`
}
```

| 字段 | 含义 | 补充 |
| --- | --- | --- |
| `id` | 模板唯一标识 | `config.yaml` 用 `xxx_prompt_id` 引用它 |
| `name` / `description` | 名称 / 用途描述 | 前端展示用，可多语言（见 §8） |
| `content` | **系统侧 prompt** | LLM 的 system 消息主体，所有模板都用 |
| `user` | **用户侧 prompt** | 可选，只在需要 system+user 配对的模板（如 rewrite）用 |
| `has_knowledge_base` / `has_web_search` | 能力标记 | 标记模板是否引用 KB / 网页搜索上下文，前端据此过滤可选模板 |
| `default` | 默认标记 | 同一类型多个模板时，标哪个是运行时默认 |
| `mode` | 模式标记 | 见 §7，主要给 fallback 区分 fixed / model |
| `i18n` | 多语言 name/description | 键为 locale，值为 `{Name, Description}` |

**关键设计**（注释原文）：每个模板最多由两部分组成——系统侧（`content`）和用户侧（`user`）。`content` 是所有模板都用；`user` 只在需要 system+user 配对的模板（如 `rewrite`、`keywords_extraction`）用。

---

## 3. 为什么这么设计

1. **content + user 分离**——单消息任务（意图 prompt、兜底）只用 `content`；双消息任务（改写：system 给规则 + user 给当前问题）用 `content`+`user`。一个结构覆盖两类任务。

2. **i18n 多语言 name/description**——`name`/`description` 是给**人**（前端 UI）看的，所以多语言；`content`/`user` 是给 **LLM** 的（LLM 本身多语言，通过 `{{language}}` 占位符控制输出语言），不翻译。

3. **ID 引用解耦**——`config.yaml` 只存 `rewrite_prompt_id: "default_rewrite"` 这类 ID，不内联模板正文。模板内容可独立升级、前端可自选，配置不动。

4. **多模板 + default 标记**——同一类型多个模板（rewrite.yaml 有 3 个），`default: true` 标运行时用的，前端可让用户切换。

5. **按类型分文件**——11 个 YAML 文件各管一类，职责清晰，可独立维护。

---

## 4. 加载规则：loadPromptTemplates

`internal/config/config.go:1066`：

1. **目录检查**：`config/prompt_templates/` 目录不存在 → 返回 nil（让调用者用 config 里内联的模板）；
2. **文件映射**：硬编码 `文件名 → 配置字段` 的映射表（`system_prompt.yaml` → `SystemPrompt`，…共 11 项）；
3. **逐文件加载**：文件不存在 → 跳过；存在 → `os.ReadFile` + `yaml.Unmarshal` 到 `promptTemplateFile{Templates []PromptTemplate}`；
4. **赋给目标字段**：`*target = file.Templates`。

### 4.1 ID 回填（backfillConversationDefaults）

加载后，`config.yaml` 里的 `xxx_prompt_id` 通过 `FindTemplateByID` 解析成具体内容：

```go
if conv.RewritePromptID != "" {
    if t := FindTemplateByID(pt, conv.RewritePromptID); t != nil {
        conv.RewritePromptSystem = t.Content   // content → system prompt
        conv.RewritePromptUser = t.User         // user → user prompt
    } else {
        fmt.Printf("Warning: rewrite_prompt_id %q not found\n", conv.RewritePromptID)
    }
}
```

- `FindTemplateByID` **跨所有类型搜索**（遍历 11 个集合，按 `ID` 匹配）；
- 找不到 → 打印 `Warning: xxx_id not found`（不崩溃）；
- 意图 prompt 额外处理：把 `IntentPrompts` 按「模板 ID = 意图值」构建 `IntentSystemPrompts` 映射（如 `greeting` 意图 → greeting 模板）。

---

## 5. 占位符系统

`internal/types/placeholder.go`：模板正文用 `{{key}}` 占位符，运行时动态填充。

**已定义的占位符**：

| 占位符 | 含义 |
| --- | --- |
| `{{query}}` | 用户当前问题 |
| `{{contexts}}` | 检索到的相关内容列表 |
| `{{conversation}}` | 格式化历史对话（多轮改写） |
| `{{language}}` | 用户语言偏好（控制 LLM 回答语言） |
| `{{current_time}}` / `{{current_week}}` | 当前时间 / 星期（自动填充） |
| `{{yesterday}}` | 昨天日期（自动填充） |
| `{{kb_documents}}` | 知识库文档列表（fallback 用） |
| `{{knowledge_bases}}` / `{{web_search_status}}` | Agent 模式的知识库列表 / 网页搜索状态 |

**渲染规则**（`RenderPromptPlaceholders`）：

- `{{key}}` → 对应值；
- **未知占位符保持原样**（不报错）；
- `current_time` / `current_week` / `yesterday` 未显式提供时**自动填充**。

> 占位符是「模板可复用」的关键——同一个模板，运行时注入不同 query/contexts/language 服务不同请求。

---

## 6. prompt 的语言设计（中英文混用）

Prompt 模板里「中英文混用」是**有意的三分离设计**：

| 位置 | 语言 | 为什么 |
| --- | --- | --- |
| system prompt 正文（content/user） | **英文** | LLM 对英文指令理解最稳定 |
| 输出语言指令 | `{{language}}` 占位符 | 一份英文模板服务所有语言用户 |
| name/description | i18n 多语言 | 给前端 UI 的人看 |

### 6.1 为什么 system prompt 用英文

1. **英文指令遵循度最稳定**——LLM 训练语料里英文指令占主导，跨模型一致；
2. **英文 token 密度低**——同样语义 token 数更少，省钱、省上下文窗口；
3. **跨模型通用**——一份英文 prompt 换任何 LLM 都能稳定遵循。

### 6.2 {{language}} 的值：人类可读语言名，不是 locale 代码

`LanguageLocaleName`（`context_helpers.go`）把 locale 映射成人类可读的**英文语言名**：

```go
"zh-CN" → "Chinese (Simplified)"
"zh-TW" → "Chinese (Traditional)"
"en-US" → "English"
"ko-KR" → "Korean"
"ja-JP" → "Japanese"
"ru-RU" → "Russian"
```

所以模板里 `{{language}}` 实际渲染成 **"Chinese (Simplified)"**，LLM 看到的是：

```text
The rewritten question must be in Chinese (Simplified)
```

**设计巧思**：LLM 对 "Chinese (Simplified)" 这种**完整语言名**的理解，比 "zh-CN" 这种 locale 代码（LLM 可能不认识）更准确，也比「中文」（有歧义）更明确。

### 6.3 语言解析链（ResolveLanguage）

```go
// 优先级：显式 locale → ctx locale → DefaultLanguage()
if locale != "" { return locale }              // 1. 任务显式指定
if ctxLocale, ok := LanguageFromContext(ctx)   // 2. HTTP 语言中间件注入
if ... { return ctxLocale }
return DefaultLanguage()                        // 3. WEKNORA_LANGUAGE 环境变量，默认 "zh-CN"
```

用户语言从 HTTP 请求的语言中间件（`LanguageContextKey`）注入 ctx，异步任务用 `ResolveLanguage` 兜底（避免空 locale 插值成 "Write in ." 的坏 prompt）。

### 6.4 混用的例外（历史遗留）

`config.yaml` 的 `extract` 段存在不一致：`extract_graph` 模板 description 是**英文**，但 `extract_entity` 模板 description 是**中文**（"基于用户的问题，处理关键信息抽取任务..."）。这属于历史遗留，不是规范。主流做法是「system prompt 英文 + `{{language}}` 占位符 + name/description 走 i18n」。

---

## 7. fallback.yaml 的 mode 区分

`fallback.yaml` 同时放两类兜底模板，用 `mode` 字段区分：

| mode | 类型 | 用途 | 示例 |
| --- | --- | --- | --- |
| （无 mode，默认） | **固定回复**（fixed） | `content` 直接作为回复文本，不走 LLM | `default_fallback`、`polite_fallback`、`brief_fallback` |
| `mode: "model"` | **模型兜底**（model） | `content` 作为 LLM 的 prompt，带 `{{kb_documents}}`/`{{query}}` 占位符 | `model_fallback`、`default_fallback_prompt` |

**两类模板的区别**：

- **fixed 模板**：`content` 是给用户看的固定话术（如 "Sorry, I could not find content directly related to your question..."），直接返回；
- **model 模板**：`content` 是给 LLM 的兜底 prompt，含占位符 `{{kb_documents}}`（知识库文档列表）和 `{{query}}`（用户问题），让 LLM 基于「知识库有哪些文档」做兜底回答（如列出相关文档建议）。

**配置选择**（`config.yaml`）：

```yaml
fallback_strategy: "model"          # 用 model 类模板
fallback_response: "Sorry, ..."     # fixed 类兜底话术
fallback_prompt_id: "default_fallback_prompt"  # model 类兜底模板
```

- `fallback_strategy = "fixed"` → 用固定回复（`fallback_response` 或 fixed 模板）；
- `fallback_strategy = "model"` → 用模型兜底（`fallback_prompt_id` 指向的 model 模板）。

---

## 8. i18n 的 locale 解析优先级

有两套 locale 解析逻辑，**优先级略有不同**：

### 7.1 Prompt 模板（LocalizeTemplates，config.go:405）

```go
// Fallback chain: locale → primary language (e.g. "zh" from "zh-CN") → 原始 Name/Description
l10n, ok := out[i].I18n[locale]          // 1. 精确匹配 "zh-CN"
if !ok {
    if idx := strings.IndexByte(locale, '-'); idx > 0 {
        l10n, ok = out[i].I18n[locale[:idx]]  // 2. 主语言匹配 "zh"
    }
}
// 3. 都不匹配 → 保留原始 Name/Description（不替换）
```

**优先级**：精确匹配（`zh-CN`）→ 主语言（`zh`）→ **原始 Name/Description**（兜底，不替换）。

### 7.2 BuiltinAgent（resolveI18n，builtin_agent_config.go:184）

```go
// Priority: exact match → language-only match → "default" → first entry
// 1. 精确匹配 "zh-CN"
// 2. 语言匹配 "zh"（含前缀匹配 k 以 "zh" 开头）
// 3. "default" 键
// 4. 第一个条目
```

**优先级**：精确匹配 → 语言匹配（含前缀）→ `"default"` 键 → **第一个条目**。

### 两者对比

| 兜底链 | LocalizeTemplates（prompt 模板） | resolveI18n（BuiltinAgent） |
| --- | --- | --- |
| 1 | 精确 `zh-CN` | 精确 `zh-CN` |
| 2 | 主语言 `zh` | 主语言 `zh`（+ 前缀匹配） |
| 3 | **原始 Name/Description** | `"default"` 键 |
| 4 | — | 第一个条目 |

> 差异点：prompt 模板匹配失败时**保留原文**（不回退到 default 键）；BuiltinAgent 有更长的兜底链（default 键 + 第一个条目）。这是两类对象的定位不同——prompt 模板的 name/description 有原始值兜底，agent preset 没有。

---

## 9. 关键文件索引

| 关注点 | 文件 |
| --- | --- |
| PromptTemplate / PromptTemplatesConfig 结构 | `internal/config/config.go`（335 / 352 行） |
| 模板加载（loadPromptTemplates） | `internal/config/config.go:1066` |
| ID 回填（backfillConversationDefaults / FindTemplateByID） | `internal/config/config.go:937 / 1021` |
| prompt 模板 i18n 本地化（LocalizeTemplates） | `internal/config/config.go:405` |
| 占位符系统（RenderPromptPlaceholders） | `internal/types/placeholder.go` |
| 模板 YAML 文件 | `config/prompt_templates/*.yaml`（11 个） |
| 配置引用（xxx_prompt_id） | `config/config.yaml`（`conversation` 段） |
