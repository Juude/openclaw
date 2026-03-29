# OpenClaw 记忆系统架构详解

> 基于源码分析，覆盖存储、检索、上下文控制、压缩、生命周期全链路。

---

## 目录

1. [设计哲学](#一设计哲学)
2. [文件记忆层 — 存储](#二文件记忆层--存储)
3. [索引检索层 — 如何找到正确的记忆](#三索引检索层--如何找到正确的记忆)
4. [Agent 工具接口 — 如何存取](#四agent-工具接口--如何存取)
5. [会话压缩层 — 防止记忆占据上下文](#五会话压缩层--防止记忆占据上下文)
6. [记忆生命周期 — 是否永久存在](#六记忆生命周期--是否永久存在)
7. [插件与扩展机制](#七插件与扩展机制)
8. [配置参考](#八配置参考)
9. [源码索引](#九源码索引)

---

## 一、设计哲学

OpenClaw 的记忆系统遵循一个核心原则：

> **Markdown 文件是唯一的真实来源（source of truth）。模型只"记住"被写入磁盘的内容。**

所有索引（SQLite 向量表、FTS5 全文索引、嵌入缓存）都是从 Markdown 文件**派生**出来的，可以在任何时候从源文件完整重建。这使得系统具有极强的可审计性和可恢复性。

整个记忆系统由三个正交的子系统组成：

| 子系统         | 职责                          | 核心源码                                      |
| -------------- | ----------------------------- | --------------------------------------------- |
| **文件记忆层** | 持久化存储 — 纯 Markdown 文件 | `MEMORY.md`, `memory/YYYY-MM-DD.md`           |
| **索引检索层** | 向量 + 全文搜索索引与检索     | `src/memory/manager.ts`, SQLite DB            |
| **会话压缩层** | 上下文窗口管理 + 记忆冲洗     | `src/agents/compaction.ts`, `memory-flush.ts` |

---

## 二、文件记忆层 — 存储

### 2.1 文件布局

记忆文件位于 agent workspace 目录下（默认 `~/.openclaw/workspace/`），结构如下：

```
~/.openclaw/workspace/
├── MEMORY.md                    # 长期策划的核心记忆（偏好、决策、重要事实）
├── memory.md                    # MEMORY.md 的备选文件名
└── memory/
    ├── 2026-03-25.md            # 每日追加日志
    ├── 2026-03-26.md
    └── 2026-03-27.md
```

**文件识别逻辑**（`src/memory/internal.ts` → `isMemoryPath()`）：

- `MEMORY.md` / `memory.md` — 被识别为"常青"记忆文件（evergreen），**永不受时间衰减影响**
- `memory/YYYY-MM-DD.md` — 每日日志，通过文件名中的日期解析后施加时间衰减
- `memorySearch.extraPaths` 配置可以指向额外的 `.md` 文件或目录

文件扫描函数 `listMemoryFiles()` 遍历以下位置：

1. `workspace/MEMORY.md`
2. `workspace/memory.md`
3. `workspace/memory/` 下所有 `.md` 文件（递归）
4. 所有 `extraPaths` 指向的 `.md` 文件或目录

只收集 `.md` 后缀的普通文件，跳过 symlinks。每个文件通过 `buildFileEntry()` 计算 SHA-256 hash 用于增量同步。

### 2.2 写入策略 — 两条路径

**路径 1：显式写入（用户/agent 主动操作）**

当用户说"记住这个"，或 agent 判断需要存储信息时，通过文件操作工具将内容写入：

- `MEMORY.md` — 长期事实（偏好、决策、规则）
- `memory/YYYY-MM-DD.md` — 日常笔记（追加模式）

**路径 2：自动记忆冲洗（Pre-compaction Memory Flush）**

这是最关键的自动写入路径，在**会话即将被压缩之前**触发。详见 [第五节](#53-记忆冲洗memory-flush--压缩前的关键步骤)。

### 2.3 可选：会话转录索引

当 `memorySearch.experimental.sessionMemory = true` 时，系统还会索引会话转录文件：

- 位置：`~/.openclaw/agents/<agentId>/sessions/*.jsonl`
- 处理：`src/memory/session-files.ts` → `buildSessionEntry()` 将 JSONL 中的对话消息扁平化为纯文本
- 然后像 Markdown 文件一样进行分块和嵌入
- 增量同步：通过 `deltaBytes` (默认 100KB) 和 `deltaMessages` (默认 50) 阈值控制重新索引频率

---

## 三、索引检索层 — 如何找到正确的记忆

### 3.1 SQLite 索引存储

索引存储在 per-agent 的 SQLite 数据库中，默认路径 `~/.openclaw/memory/<agentId>.sqlite`。

Schema 定义在 `src/memory/memory-schema.ts`：

```sql
-- 元信息表：记录当前索引的 model/provider/chunking 参数
CREATE TABLE meta (key TEXT PRIMARY KEY, value TEXT NOT NULL);

-- 已索引文件的清单（通过 hash 追踪变更）
CREATE TABLE files (
  path TEXT PRIMARY KEY,
  source TEXT NOT NULL DEFAULT 'memory',  -- 'memory' | 'sessions'
  hash TEXT NOT NULL,                     -- SHA-256 of file content
  mtime INTEGER NOT NULL,
  size INTEGER NOT NULL
);

-- 文本分块 + 嵌入向量
CREATE TABLE chunks (
  id TEXT PRIMARY KEY,                    -- UUID
  path TEXT NOT NULL,                     -- 相对路径
  source TEXT NOT NULL DEFAULT 'memory',  -- 'memory' | 'sessions'
  start_line INTEGER NOT NULL,            -- 块起始行
  end_line INTEGER NOT NULL,              -- 块结束行
  hash TEXT NOT NULL,                     -- SHA-256 of chunk text
  model TEXT NOT NULL,                    -- 嵌入模型标识（如 text-embedding-3-small）
  text TEXT NOT NULL,                     -- 原始文本内容
  embedding TEXT NOT NULL,                -- JSON 序列化的 float[] 向量
  updated_at INTEGER NOT NULL             -- 时间戳
);

-- 嵌入缓存（避免重复 API 调用，按 hash 去重）
CREATE TABLE embedding_cache (
  provider TEXT NOT NULL,
  model TEXT NOT NULL,
  provider_key TEXT NOT NULL,
  hash TEXT NOT NULL,
  embedding TEXT NOT NULL,
  dims INTEGER,
  updated_at INTEGER NOT NULL,
  PRIMARY KEY (provider, model, provider_key, hash)
);

-- FTS5 全文检索虚拟表（关键词搜索，BM25 排序）
CREATE VIRTUAL TABLE chunks_fts USING fts5(
  text,
  id UNINDEXED, path UNINDEXED, source UNINDEXED,
  model UNINDEXED, start_line UNINDEXED, end_line UNINDEXED
);

-- sqlite-vec 向量搜索虚拟表（余弦距离排序）
CREATE VIRTUAL TABLE chunks_vec USING vec0(
  id TEXT PRIMARY KEY,
  embedding FLOAT[<dimensions>]  -- 维度由嵌入模型决定
);
```

### 3.2 分块策略

`chunkMarkdown()` 函数（`src/memory/internal.ts`）实现**固定窗口 + 重叠**的 Markdown 分块：

```
文件内容（按行）
│
├─ 块 1: 行 1-25  ──────────────────────────┐
│                          ≤ 1600 chars      │ overlap
├─ 块 2: 行 20-45 ──────────────────────────┤ 320 chars
│                                            │
├─ 块 3: 行 40-60 ──────────────────────────┘
│
...
```

**默认参数**：

| 参数               | 默认值              | 含义                               |
| ------------------ | ------------------- | ---------------------------------- |
| `chunking.tokens`  | 400                 | 每块最大 token 数                  |
| `chunking.overlap` | 80                  | 块间重叠 token 数                  |
| `maxChars`         | `tokens × 4 = 1600` | 字符级限制（粗略 token→char 换算） |
| `overlapChars`     | `overlap × 4 = 320` | 字符级重叠                         |

**分块逻辑**：

1. 按行遍历文件内容
2. 累积字符数；当超过 `maxChars` 时 flush 当前块
3. `carryOverlap()` 保留上一个块末尾的 `overlapChars` 字符到新块开头
4. 每个块记录 `startLine` / `endLine`（1-indexed），用于后续精确读取
5. 对超长单行（如 base64 数据），按 `maxChars` 进行二次切割
6. 每个块计算 SHA-256 hash 用于缓存命中判断

### 3.3 嵌入生成

`MemoryManagerEmbeddingOps`（`src/memory/manager-embedding-ops.ts`）管理嵌入向量的生成。

**支持的嵌入提供商**：

| 提供商       | 默认模型                 | 配置方式                                              |
| ------------ | ------------------------ | ----------------------------------------------------- |
| OpenAI       | `text-embedding-3-small` | `OPENAI_API_KEY` 或 auth profile                      |
| Gemini       | `gemini-embedding-001`   | `GEMINI_API_KEY` 或 `models.providers.google.apiKey`  |
| Voyage       | `voyage-4-large`         | `VOYAGE_API_KEY`                                      |
| Mistral      | `mistral-embed`          | `MISTRAL_API_KEY`                                     |
| Ollama       | `nomic-embed-text`       | 本地部署，不需要 API key                              |
| Local (GGUF) | 用户指定路径             | `memorySearch.local.modelPath`, 依赖 `node-llama-cpp` |

**自动选择逻辑**（`provider = "auto"` 时，`src/agents/memory-search.ts`）：

```
1. 如果配置了 local.modelPath 且文件存在 → 使用 "local"
2. 如果有 OpenAI key 可解析 → 使用 "openai"
3. 如果有 Gemini key 可解析 → 使用 "gemini"
4. 如果有 Voyage key 可解析 → 使用 "voyage"
5. 如果有 Mistral key 可解析 → 使用 "mistral"
6. 否则，记忆搜索保持禁用（降级到 FTS-only 模式）
```

**嵌入缓存**：

- 嵌入结果按 `(provider, model, provider_key, hash)` 四元组缓存在 `embedding_cache` 表中
- 相同内容（相同 hash）不会重复调用 API
- LRU 淘汰：`pruneEmbeddingCacheIfNeeded()` 当条目数超过 `cache.maxEntries` 时，按 `updated_at ASC` 删除最旧条目

**批量嵌入**：

- `buildEmbeddingBatches()` 将多个 chunk 打包成不超过 `EMBEDDING_BATCH_MAX_TOKENS = 8000` 的批次
- 支持 OpenAI Batch API 和 Gemini Batch API 的异步批处理
- 批次失败重试 3 次，指数退避（500ms → 8s）
- 超过 `BATCH_FAILURE_LIMIT = 2` 次批次失败后停止批处理

**提供商降级**：

当主嵌入提供商出错（如配额耗尽）时，`activateFallbackProvider()` 自动切换到配置的 `fallback` 提供商，并触发完整重建索引。

### 3.4 搜索管道 — 完整流程

当 agent 调用 `memory_search` 工具时，`MemoryIndexManager.search()` 执行以下流程：

```
用户查询 (query: string)
    │
    ▼
╔══════════════════════════════════════════╗
║  步骤 1：触发同步（如有脏数据）            ║
║  - dirty 标记（文件变更）                 ║
║  - sessionsDirty 标记（会话变更）          ║
║  - warmSession() 确保会话文件已索引        ║
╚══════════════════════════════════════════╝
    │
    ▼
╔══════════════════════════════════════════╗
║  步骤 2：计算候选数                       ║
║  candidates = min(200, maxResults × 4)   ║
║  candidateMultiplier 默认为 4             ║
╚══════════════════════════════════════════╝
    │
    ├──── 无嵌入提供商？──→ FTS-only 路径（步骤 2a）
    │
    ▼
╔══════════════════════════════════════════╗
║  步骤 3：并行执行两路检索                  ║
║                                          ║
║  ┌────────────────────────────────────┐  ║
║  │ 向量搜索 (Vector Search)            │  ║
║  │                                    │  ║
║  │ query → embedQueryWithTimeout()    │  ║
║  │      → float[] 查询向量            │  ║
║  │                                    │  ║
║  │ 首选：sqlite-vec                   │  ║
║  │   vec_distance_cosine(v.embedding, │  ║
║  │     queryVec) → ORDER BY dist ASC  │  ║
║  │   score = 1 - distance             │  ║
║  │                                    │  ║
║  │ 降级：JS cosineSimilarity()        │  ║
║  │   加载所有 chunks 到内存，逐一计算   │  ║
║  └────────────────────────────────────┘  ║
║                                          ║
║  ┌────────────────────────────────────┐  ║
║  │ 全文搜索 (FTS5 Keyword Search)      │  ║
║  │                                    │  ║
║  │ query → buildFtsQuery()            │  ║
║  │   分词 → "token1" AND "token2"     │  ║
║  │                                    │  ║
║  │ chunks_fts MATCH ftsQuery          │  ║
║  │   bm25(chunks_fts) → rank         │  ║
║  │                                    │  ║
║  │ bm25RankToScore():                 │  ║
║  │   rank < 0: score = -rank/(1-rank) │  ║
║  │   rank ≥ 0: score = 1/(1+rank)    │  ║
║  └────────────────────────────────────┘  ║
╚══════════════════════════════════════════╝
    │
    ▼
╔══════════════════════════════════════════╗
║  步骤 4：混合融合 (Hybrid Merge)          ║
║                                          ║
║  mergeHybridResults():                   ║
║  - 按 (path, startLine, endLine) 去重    ║
║  - 融合分数计算：                         ║
║    finalScore = vectorWeight × vecScore  ║
║               + textWeight × textScore   ║
║  - 默认权重：0.7 (向量) : 0.3 (关键词)    ║
╚══════════════════════════════════════════╝
    │
    ▼
╔══════════════════════════════════════════╗
║  步骤 5：可选 — MMR 多样性重排            ║
║  （默认关闭）                             ║
║                                          ║
║  applyMMRToHybridResults():              ║
║  - Carbonell & Goldstein (1998) 算法     ║
║  - λ=0.7：70% 相关性 + 30% 多样性       ║
║  - 使用 Jaccard similarity 衡量冗余      ║
║  - 贪心迭代：每次选 MMR 值最高的候选      ║
╚══════════════════════════════════════════╝
    │
    ▼
╔══════════════════════════════════════════╗
║  步骤 6：可选 — 时间衰减                  ║
║  （默认关闭）                             ║
║                                          ║
║  applyTemporalDecayToHybridResults():    ║
║  - 指数衰减：multiplier = e^(-λ × age)  ║
║  - λ = ln(2) / halfLifeDays             ║
║  - 默认半衰期：30 天                      ║
║  - memory/YYYY-MM-DD.md → 按日期衰减     ║
║  - MEMORY.md → 永不衰减（evergreen）      ║
║  - 非日期文件 → 按 mtime 衰减            ║
╚══════════════════════════════════════════╝
    │
    ▼
╔══════════════════════════════════════════╗
║  步骤 7：过滤 + 截断                     ║
║                                          ║
║  - filter: score >= minScore (0.35)      ║
║  - slice: maxResults (6)                 ║
║  - snippet 截断: SNIPPET_MAX_CHARS (700) ║
║                                          ║
║  特殊处理：                               ║
║  当 hybrid 融合后所有结果 < minScore，     ║
║  但存在纯关键词命中时，放宽阈值到          ║
║  min(minScore, textWeight) 以保留精确      ║
║  词法匹配。                               ║
╚══════════════════════════════════════════╝
    │
    ▼
返回 MemorySearchResult[]
```

### 3.5 FTS-only 模式（无嵌入提供商时的降级方案）

当没有嵌入提供商可用时（所有 API key 都未配置），系统自动降级到纯全文搜索模式。此时使用 `src/memory/query-expansion.ts` 的**多语言查询扩展**：

1. **分词**：使用正则 `[\s\p{P}]+` 按空格和标点分割
2. **中文处理**：提取 unigram + bigram（因为没有分词器）
3. **日语处理**：按书写系统（汉字/片假名/平假名/ASCII）分割后分别处理
4. **韩语处理**：剥离尾部助词（`은/는/이/가/을/를/...`）提取词干
5. **停用词过滤**：内置 7 种语言的停用词表（英/中/日/韩/西/葡/阿）
6. **对每个关键词独立执行 FTS 搜索**，然后合并去重取最高分

### 3.6 QMD 替代后端

当 `memory.backend = "qmd"` 时，整个索引和检索委托给 [QMD](https://github.com/tobi/qmd)：

- 运行在 `~/.openclaw/agents/<agentId>/qmd/` 下独立的 XDG home 中
- 结合 BM25 + 向量 + 重排序（本地 GGUF 模型）
- 搜索通过子进程 `qmd search --json` 进行
- 定期执行 `qmd update` + `qmd embed` 保持索引新鲜
- `FallbackMemoryManager`（`src/memory/search-manager.ts`）确保 QMD 失败时自动降级到内置 SQLite 索引

### 3.7 索引同步机制

`MemoryManagerSyncOps`（`src/memory/manager-sync-ops.ts`）实现多种同步触发：

| 触发方式       | 何时生效                                  | 配置项                                                    |
| -------------- | ----------------------------------------- | --------------------------------------------------------- |
| **文件监控**   | `MEMORY.md`/`memory/` 任何 `.md` 文件变更 | `sync.watch = true`, `watchDebounceMs = 1500`             |
| **搜索时同步** | `memory_search` 调用且存在脏数据          | `sync.onSearch = true`                                    |
| **会话开始**   | 新会话首次搜索                            | `sync.onSessionStart = true`                              |
| **定时同步**   | 固定间隔重新索引                          | `sync.intervalMinutes`                                    |
| **会话增量**   | JSONL 文件增长超阈值                      | `sync.sessions.deltaBytes = 100000`, `deltaMessages = 50` |

**增量更新**：比较 `files` 表中的 `hash` 与文件当前 SHA-256，只重新索引变更过的文件。

**安全重建**（`runSafeReindex()`）：当检测到 model/provider/chunking 参数变更时：

1. 在临时数据库中完整重建索引
2. 从旧数据库迁移嵌入缓存（`seedEmbeddingCache()`）
3. 原子交换数据库文件（rename）
4. 确保搜索服务不中断

**只读数据库恢复**（`runSyncWithReadonlyRecovery()`）：检测到 `SQLITE_READONLY` 错误时，自动关闭并重新打开数据库连接。

---

## 四、Agent 工具接口 — 如何存取

记忆通过两个 agent 工具暴露（`src/agents/tools/memory-tool.ts`）。

### 4.1 `memory_search` — 语义搜索（召回）

```typescript
{
  name: "memory_search",
  description: "Mandatory recall step: semantically search MEMORY.md + memory/*.md
    (and optional session transcripts) before answering questions about prior work,
    decisions, dates, people, preferences, or todos; returns top snippets with
    path + lines.",
  parameters: {
    query: string,         // 搜索查询
    maxResults?: number,   // 默认 6
    minScore?: number,     // 默认 0.35
  }
}
```

**关键设计**：工具描述包含"**Mandatory recall step**"——指导模型在回答涉及过往决策、日期、人物、偏好或待办事项的问题时，**必须**先调用此工具进行检索。

**返回示例**：

```json
{
  "results": [
    {
      "path": "memory/2026-03-25.md",
      "startLine": 12,
      "endLine": 18,
      "score": 0.82,
      "snippet": "Decided to use PostgreSQL...\n\nSource: memory/2026-03-25.md#L12-L18",
      "source": "memory"
    }
  ],
  "provider": "openai",
  "model": "text-embedding-3-small",
  "mode": "hybrid"
}
```

**上下文控制**：

- snippet 最大 700 字符（`SNIPPET_MAX_CHARS`）
- 最多返回 6 条结果（`maxResults`）
- QMD 后端额外有 `maxInjectedChars` 总字符预算

### 4.2 `memory_get` — 精确读取

```typescript
{
  name: "memory_get",
  description: "Safe snippet read from MEMORY.md or memory/*.md with optional
    from/lines; use after memory_search to pull only the needed lines and keep
    context small.",
  parameters: {
    path: string,      // 相对路径（如 "memory/2026-03-25.md"）
    from?: number,     // 起始行号（1-indexed）
    lines?: number,    // 读取行数
  }
}
```

**设计意图**：`memory_search` 返回短 snippet + 位置信息，`memory_get` 用于**按需精确拉取**特定片段的完整内容。这种两步模式确保上下文消耗最小化。

**安全约束**：

- 只允许读取 `MEMORY.md`、`memory.md`、`memory/` 目录下或 `extraPaths` 配置的 `.md` 文件
- 禁止读取其他路径（返回 "path required" 错误）
- 文件不存在时返回空文本 `{ text: "", path }` 而不是报错——agent 可以正常处理"尚无记录"

### 4.3 引用 (Citations)

`memory.citations` 配置控制搜索结果中是否附加来源引用：

| 模式             | 行为                                                   |
| ---------------- | ------------------------------------------------------ |
| `"auto"`（默认） | 私聊中显示引用，群聊/频道中隐藏                        |
| `"on"`           | 始终显示 `Source: path#L12-L18` 尾注                   |
| `"off"`          | 不显示尾注（路径仍在结构化数据中供 `memory_get` 使用） |

---

## 五、会话压缩层 — 防止记忆占据上下文

这是系统中最复杂的部分，解决核心问题：**长对话如何在有限上下文窗口中持续运行，同时不丢失重要信息**。

### 5.1 上下文窗口管理

每个模型有固定的上下文窗口大小（如 Claude 200K tokens, GPT-4 128K tokens）。随着对话进行，消息和工具调用结果不断累积。OpenClaw 通过以下层次管理上下文使用：

```
┌──────────────────────────────────────────────────────────┐
│                      Context Window                       │
│                                                          │
│  ┌────────────────┐  系统提示 + Bootstrap 文件            │
│  │ System Prompt   │  (bootstrapMaxChars 限制)            │
│  └────────────────┘                                      │
│                                                          │
│  ┌────────────────┐  如有压缩历史                         │
│  │ Compaction      │  → 简短摘要替代旧消息                 │
│  │ Summary         │                                     │
│  └────────────────┘                                      │
│                                                          │
│  ┌────────────────┐                                      │
│  │ Recent Messages │  最近的对话消息（完整保留）            │
│  │                 │                                     │
│  └────────────────┘                                      │
│                                                          │
│  ┌────────────────┐  memory_search 返回的 snippet         │
│  │ Memory Recalls  │  (按需拉取，最多 6×700 chars)         │
│  └────────────────┘                                      │
│                                                          │
│  ┌────────────────┐                                      │
│  │ Reserve Floor   │  reserveTokensFloor (默认 20000)     │
│  │                 │  为新回复保留的空间                    │
│  └────────────────┘                                      │
└──────────────────────────────────────────────────────────┘
```

**关键点**：记忆文件**不会**被全量注入上下文。`MEMORY.md` 在主会话启动时读取（仅限私聊），`memory/YYYY-MM-DD.md` 只读取今天和昨天的。其余记忆通过 `memory_search` **按需拉取**。

### 5.2 自动压缩 (Auto-compaction)

当会话 token 估计值接近 `contextWindow - reserveTokensFloor` 时，`src/agents/compaction.ts` 的压缩系统自动启动。

**Token 估算**：`estimateMessagesTokens()` 使用 `chars / 4` 的粗略换算（`estimateTokens` from pi-coding-agent）。因为这种方法对多字节字符、特殊 token 等估算不足，应用了 **20% 安全余量**（`SAFETY_MARGIN = 1.2`）。

**压缩流程**：

```
当前会话消息 [M1, M2, M3, ..., Mn]
    │
    ▼
splitMessagesByTokenShare(messages, parts=2)
    │
    ├─→ 块 1 (旧消息，约 50% token)  ─→  LLM 摘要 (generateSummary)
    │                                      │
    │                                      ├── 保留：活跃任务 + 当前状态
    │                                      ├── 保留：批量操作进度（如 "5/17 完成"）
    │                                      ├── 保留：最后一个请求及处理情况
    │                                      ├── 保留：决策及理由
    │                                      ├── 保留：待办、开放问题、约束
    │                                      ├── 保留：承诺和后续跟进
    │                                      └── 保留：所有不透明标识符
    │                                           (UUID, hash, IP, URL, 文件名...)
    │
    └─→ 块 2 (新消息，约 50% token)  ─→  完整保留

结果 = [Compaction Summary] + [块 2 的完整消息]
     → 持久化到会话 JSONL 文件
```

**自适应分块**（`computeAdaptiveChunkRatio()`）：

当消息平均大小超过上下文窗口的 10% 时，自动缩小分块比率：

```
avgRatio = (avgTokens × SAFETY_MARGIN) / contextWindow
if avgRatio > 0.1:
  reduction = min(avgRatio × 2, BASE_CHUNK_RATIO - MIN_CHUNK_RATIO)
  chunkRatio = max(0.15, 0.4 - reduction)
```

**超大消息处理**（`summarizeWithFallback()`）：

当单条消息超过上下文窗口 50%（`isOversizedForSummary()`）：

1. 先尝试完整摘要
2. 失败则分离超大消息，只摘要小消息
3. 超大消息以 `[Large <role> (~NNK tokens) omitted from summary]` 标注
4. 如果仍然失败，生成最基本的统计描述

**多阶段摘要**（`summarizeInStages()`）：

对于非常长的历史，先按 token 份额分成 N 个部分，各自独立摘要，然后用 `MERGE_SUMMARIES_INSTRUCTIONS` 提示将部分摘要合并为一份连贯摘要。

**标识符保留策略**：

默认 `identifierPolicy: "strict"`，摘要指令中包含：

> "Preserve all opaque identifiers exactly as written (no shortening or reconstruction), including UUIDs, hashes, IDs, tokens, API keys, hostnames, IPs, ports, URLs, and file names."

可配置为 `"off"`（不保留）或 `"custom"`（自定义指令）。

### 5.3 记忆冲洗 (Memory Flush) — 压缩前的关键步骤

这是整个系统最巧妙的设计。**在自动压缩执行之前**，系统插入一次**静默的 agent turn**，提醒模型将当前上下文中的持久信息写入磁盘：

```
会话 token 接近上限
    │
    ▼
shouldRunMemoryFlush() 判断：
  totalTokens >= contextWindow - reserveTokensFloor - softThresholdTokens
  且本压缩周期尚未冲洗过
    │
    ▼ YES
╔══════════════════════════════════════════════╗
║  静默 Memory Flush Turn                       ║
║                                              ║
║  System Prompt:                              ║
║  "Pre-compaction memory flush turn.           ║
║   The session is near auto-compaction;        ║
║   capture durable memories to disk.           ║
║   Store durable memories only in              ║
║   memory/YYYY-MM-DD.md.                       ║
║   If memory/YYYY-MM-DD.md already exists,     ║
║   APPEND new content only.                    ║
║   Treat MEMORY.md, SOUL.md, TOOLS.md,        ║
║   AGENTS.md as read-only during this flush.   ║
║   If nothing to store, reply with NO_REPLY."  ║
║                                              ║
║  → Agent 将重要信息写入 memory/YYYY-MM-DD.md  ║
║  → 或回复 NO_REPLY（用户不可见）               ║
╚══════════════════════════════════════════════╝
    │
    ▼
正常执行 Auto-compaction
（此时磁盘上的记忆已经被保存，即使旧消息被摘要替代，信息也不会丢失）
```

**触发条件**（`shouldRunMemoryFlush()` in `src/auto-reply/reply/memory-flush.ts`）：

```
threshold = contextWindow - reserveTokensFloor - softThresholdTokens
          = contextWindow - 20000 - 4000

if totalTokens >= threshold
   && !hasAlreadyFlushedForCurrentCompaction(entry)
   → 触发 flush
```

**双重触发机制**：

| 触发方式     | 阈值                                  | 说明                        |
| ------------ | ------------------------------------- | --------------------------- |
| Token 基础   | `contextWindow - 24000`               | 基于 prompt token 估计      |
| 转录文件大小 | `2 MB`（`forceFlushTranscriptBytes`） | 当 JSONL 文件过大时强制触发 |

**一次性保证**：通过 `memoryFlushCompactionCount` 字段追踪，确保每个压缩周期最多触发一次 flush。

**安全约束**：

- Flush 提示中强制包含"memory/YYYY-MM-DD.md 追加模式"和"核心文件只读"的安全提示
- 如果用户自定义了 flush 提示，`ensureMemoryFlushSafetyHints()` 会自动追加缺失的安全提示
- 沙箱模式 `workspaceAccess: "ro"` 或 `"none"` 时跳过 flush

### 5.4 手动压缩

用户可通过 `/compact` 命令强制触发压缩，可选附加指令：

```
/compact Focus on decisions and open questions
```

### 5.5 上下文裁剪 (pruneHistoryForContextShare)

`pruneHistoryForContextShare()` 用于更激进的上下文缩减：

- 限制历史消息占上下文的 `maxHistoryShare`（默认 50%）
- 分块丢弃最旧的消息，直到 token 数在预算内
- 丢弃后执行 `repairToolUseResultPairing()`，修复 tool_use/tool_result 配对断裂（避免 API "unexpected tool_use_id" 错误）

---

## 六、记忆生命周期 — 是否永久存在

### 6.1 Markdown 记忆文件 — 永久保留

| 类型                   | 生命周期       | 删除方式                       |
| ---------------------- | -------------- | ------------------------------ |
| `MEMORY.md`            | **永久**       | 只有用户或 agent 主动编辑/删除 |
| `memory/YYYY-MM-DD.md` | **永久**       | 只有用户或 agent 主动删除      |
| `extraPaths` 文件      | **跟随源文件** | 文件删除后，下次同步时清理索引 |

**没有内置的 TTL 或自动过期机制**。这是有意的设计选择：Markdown 文件作为用户的个人知识库，不应被自动删除。

### 6.2 SQLite 索引 — 跟随源文件

| 组件              | 生命周期     | 清理方式                                                 |
| ----------------- | ------------ | -------------------------------------------------------- |
| `files` 表记录    | 跟随源文件   | `syncMemoryFiles()` 检测到文件不存在时删除对应记录       |
| `chunks` 表记录   | 跟随所属文件 | 文件记录删除时级联删除                                   |
| `chunks_vec` 行   | 跟随 chunk   | chunk 删除时通过子查询级联删除                           |
| `chunks_fts` 行   | 跟随 chunk   | chunk 删除时手动删除对应 FTS 行                          |
| `embedding_cache` | LRU 淘汰     | `pruneEmbeddingCacheIfNeeded()` 按 `maxEntries` 上限淘汰 |

当参数（model/provider/chunking）变更时，通过 `runSafeReindex()` 完整重建索引。

### 6.3 会话转录 — 有维护策略

会话转录文件（`sessions/*.jsonl` 和 `sessions.json`）有独立的维护机制：

| 维护参数            | 默认值 | 说明                       |
| ------------------- | ------ | -------------------------- |
| `pruneAfter`        | 30 天  | 超龄条目被清理             |
| `maxEntries`        | 500    | 最大条目数                 |
| `rotateBytes`       | 10 MB  | 单文件大小上限             |
| `maxDiskBytes`      | 无限制 | 总磁盘预算（可配置）       |
| QMD `retentionDays` | 可配置 | QMD 后端的转录导出保留天数 |

维护由 `src/config/sessions/store-maintenance.ts` 和 `src/config/sessions/disk-budget.ts` 执行，但**只影响会话数据**，不会删除 workspace 中的 `MEMORY.md` 或 `memory/*.md`。

### 6.4 记忆"压缩"的本质

OpenClaw **没有**对 Markdown 记忆文件本身进行压缩或摘要。文件始终保持原始全文。"压缩"发生在两个不同的层面：

| 层面             | 机制                             | 影响对象                              |
| ---------------- | -------------------------------- | ------------------------------------- |
| **会话压缩**     | LLM 摘要旧消息                   | 会话历史（JSONL），不影响记忆文件     |
| **记忆冲洗**     | 压缩前将关键信息从上下文写入磁盘 | 上下文 → 记忆文件（**扩展**而非压缩） |
| **检索裁剪**     | snippet 截断 + maxResults        | 搜索结果注入上下文时                  |
| **嵌入缓存淘汰** | LRU by maxEntries                | 缓存表，不影响实际索引                |

---

## 七、插件与扩展机制

### 7.1 Memory Plugin Slot

记忆系统通过插件槽 `plugins.slots.memory` 加载，默认为 `memory-core`：

- **`extensions/memory-core/`** — 注册 `memory_search` / `memory_get` 工具和 CLI 命令
- **`extensions/memory-lancedb/`** — 替代实现，使用 LanceDB + OpenAI 嵌入的独立向量存储

可通过 `plugins.slots.memory = "none"` 禁用记忆插件。

### 7.2 LanceDB 替代方案

`extensions/memory-lancedb/` 实现了一个完全不同的存储后端：

- 使用 LanceDB（基于 Arrow 的向量数据库）
- 独立的 `memories` 表（不复用 Markdown 文件）
- 通过 agent hooks 自动存储消息

这是与主记忆系统**并行**的方案，不是替代 `MEMORY.md` 索引的后端。

---

## 八、配置参考

### 8.1 核心配置路径

所有记忆相关配置位于 `openclaw.json` 的以下路径：

```jsonc
{
  // 顶层记忆后端配置
  "memory": {
    "backend": "builtin", // "builtin" | "qmd"
    "citations": "auto", // "auto" | "on" | "off"
    "qmd": {
      /* QMD 后端配置 */
    },
  },

  "agents": {
    "defaults": {
      // 记忆搜索配置
      "memorySearch": {
        "enabled": true,
        "provider": "auto", // "auto"|"openai"|"gemini"|"voyage"|"mistral"|"ollama"|"local"
        "fallback": "none", // 降级提供商
        "model": "", // 自定义模型（留空用默认）
        "sources": ["memory"], // ["memory"] 或 ["memory", "sessions"]
        "extraPaths": [], // 额外的 .md 文件路径

        "store": {
          "driver": "sqlite",
          "path": "~/.openclaw/memory/{agentId}.sqlite",
          "vector": { "enabled": true },
        },

        "chunking": {
          "tokens": 400,
          "overlap": 80,
        },

        "sync": {
          "onSessionStart": true,
          "onSearch": true,
          "watch": true,
          "watchDebounceMs": 1500,
          "intervalMinutes": 0,
          "sessions": {
            "deltaBytes": 100000,
            "deltaMessages": 50,
          },
        },

        "query": {
          "maxResults": 6,
          "minScore": 0.35,
          "hybrid": {
            "enabled": true,
            "vectorWeight": 0.7,
            "textWeight": 0.3,
            "candidateMultiplier": 4,
            "mmr": { "enabled": false, "lambda": 0.7 },
            "temporalDecay": { "enabled": false, "halfLifeDays": 30 },
          },
        },

        "cache": {
          "enabled": true,
          "maxEntries": null, // 无上限
        },

        "experimental": {
          "sessionMemory": false,
        },
      },

      // 压缩配置
      "compaction": {
        "reserveTokensFloor": 20000,
        "model": null, // 自定义压缩模型
        "identifierPolicy": "strict",

        "memoryFlush": {
          "enabled": true,
          "softThresholdTokens": 4000,
          "forceFlushTranscriptBytes": "2MB",
          "prompt": "Pre-compaction memory flush. Store durable memories only in memory/YYYY-MM-DD.md...",
          "systemPrompt": "Pre-compaction memory flush turn...",
        },
      },
    },
  },
}
```

### 8.2 关键默认值速查

| 参数                              | 默认值   | 说明                      |
| --------------------------------- | -------- | ------------------------- |
| `memorySearch.enabled`            | `true`   | 记忆搜索总开关            |
| `memorySearch.provider`           | `"auto"` | 自动选择嵌入提供商        |
| `chunking.tokens`                 | 400      | 分块大小                  |
| `chunking.overlap`                | 80       | 分块重叠                  |
| `query.maxResults`                | 6        | 搜索返回最大条数          |
| `query.minScore`                  | 0.35     | 最低分数阈值              |
| `hybrid.vectorWeight`             | 0.7      | 向量搜索权重              |
| `hybrid.textWeight`               | 0.3      | 关键词搜索权重            |
| `cache.enabled`                   | `true`   | 嵌入缓存开关              |
| `compaction.reserveTokensFloor`   | 20000    | 为新回复预留的 token 数   |
| `memoryFlush.softThresholdTokens` | 4000     | flush 触发的 token 余量   |
| `SNIPPET_MAX_CHARS`               | 700      | 搜索结果 snippet 最大字符 |

---

## 九、源码索引

### 核心索引与搜索

| 文件                                  | 职责                                                    |
| ------------------------------------- | ------------------------------------------------------- |
| `src/memory/manager.ts`               | `MemoryIndexManager` — 索引生命周期、搜索编排、Watcher  |
| `src/memory/manager-sync-ops.ts`      | 文件/会话同步、安全重建、文件监控                       |
| `src/memory/manager-embedding-ops.ts` | 嵌入生成、批处理、缓存淘汰                              |
| `src/memory/manager-search.ts`        | 向量搜索（sqlite-vec / JS fallback）和关键词搜索 SQL    |
| `src/memory/memory-schema.ts`         | SQLite DDL（meta/files/chunks/cache/fts/vec 表）        |
| `src/memory/internal.ts`              | 文件扫描、hash、`chunkMarkdown()`、`cosineSimilarity()` |
| `src/memory/hybrid.ts`                | 向量 + FTS 融合、BM25 转分数                            |
| `src/memory/mmr.ts`                   | MMR 多样性重排（Jaccard similarity）                    |
| `src/memory/temporal-decay.ts`        | 时间衰减（指数衰减、日期解析、evergreen 识别）          |
| `src/memory/query-expansion.ts`       | 多语言停用词、关键词提取（EN/ZH/JA/KO/ES/PT/AR）        |
| `src/memory/search-manager.ts`        | 后端分派（QMD vs builtin）、`FallbackMemoryManager`     |
| `src/memory/session-files.ts`         | JSONL 转录 → 纯文本、行号映射                           |
| `src/memory/sqlite-vec.ts`            | sqlite-vec 扩展加载                                     |
| `src/memory/embeddings.ts`            | 嵌入提供商工厂                                          |
| `src/memory/embeddings-openai.ts`     | OpenAI 嵌入 API                                         |
| `src/memory/embeddings-gemini.ts`     | Gemini 嵌入 API                                         |
| `src/memory/embeddings-voyage.ts`     | Voyage 嵌入 API                                         |
| `src/memory/embeddings-mistral.ts`    | Mistral 嵌入 API                                        |
| `src/memory/embeddings-ollama.ts`     | Ollama 嵌入 API                                         |
| `src/memory/node-llama.ts`            | 本地 GGUF 嵌入（node-llama-cpp）                        |

### Agent 工具与压缩

| 文件                                          | 职责                                    |
| --------------------------------------------- | --------------------------------------- |
| `src/agents/tools/memory-tool.ts`             | `memory_search` / `memory_get` 工具定义 |
| `src/agents/memory-search.ts`                 | 配置解析、默认值合并、store 路径解析    |
| `src/agents/compaction.ts`                    | Token 估算、分块、LLM 摘要、多阶段压缩  |
| `src/auto-reply/reply/memory-flush.ts`        | Flush 提示、阈值判断、安全提示注入      |
| `src/auto-reply/reply/agent-runner-memory.ts` | Flush turn 执行、token 预投影           |

### QMD 后端

| 文件                             | 职责                        |
| -------------------------------- | --------------------------- |
| `src/memory/qmd-manager.ts`      | QMD 进程管理、索引、搜索    |
| `src/memory/qmd-process.ts`      | QMD CLI 子进程封装          |
| `src/memory/qmd-scope.ts`        | QMD 通道作用域控制          |
| `src/memory/qmd-query-parser.ts` | QMD JSON 输出解析           |
| `src/memory/backend-config.ts`   | 后端配置解析（builtin/qmd） |

### 配置 Schema

| 文件                                     | 职责                                   |
| ---------------------------------------- | -------------------------------------- |
| `src/config/types.memory.ts`             | `MemoryConfig`, `MemoryQmdConfig` 类型 |
| `src/config/zod-schema.ts`               | 顶层 `memory:` Zod schema              |
| `src/config/zod-schema.agent-runtime.ts` | `MemorySearchSchema` agent 级配置      |

### 会话维护

| 文件                                       | 职责                             |
| ------------------------------------------ | -------------------------------- |
| `src/config/sessions/store-maintenance.ts` | 会话条目清理（age/count/rotate） |
| `src/config/sessions/disk-budget.ts`       | 会话目录磁盘预算                 |

### 插件

| 文件                                 | 职责                           |
| ------------------------------------ | ------------------------------ |
| `extensions/memory-core/index.ts`    | 默认记忆插件（注册工具 + CLI） |
| `extensions/memory-lancedb/index.ts` | LanceDB 替代方案               |

### 文档

| 文件                                              | 内容                               |
| ------------------------------------------------- | ---------------------------------- |
| `docs/concepts/memory.md`                         | 面向用户的记忆使用指南             |
| `docs/concepts/compaction.md`                     | 上下文窗口与压缩                   |
| `docs/reference/session-management-compaction.md` | 会话管理深度参考                   |
| `docs/reference/token-use.md`                     | Bootstrap vs 记忆工具的 token 使用 |

---

## 附录：架构图

```
┌─────────────────────────────────────────────────────────────────────┐
│                         Agent Workspace (磁盘)                      │
│                                                                     │
│   MEMORY.md          memory/2026-03-25.md    memory/2026-03-26.md  │
│   (长期记忆)          (每日日志)               (每日日志)             │
│                                                                     │
│   extraPaths/*.md    sessions/*.jsonl (可选)                         │
└───────────────────────────────┬─────────────────────────────────────┘
                                │
                    ┌───────────┴───────────┐
                    │  文件监控 (chokidar)    │
                    │  + 增量同步             │
                    └───────────┬───────────┘
                                │
                    ┌───────────▼───────────────────────────────────┐
                    │        SQLite 索引 (~/.openclaw/memory/)       │
                    │                                               │
                    │  ┌─────────┐ ┌──────────┐ ┌───────────────┐  │
                    │  │  files   │ │  chunks  │ │ embedding_    │  │
                    │  │  (hash)  │ │  (text + │ │ cache         │  │
                    │  │          │ │  embedding│ │ (LRU)        │  │
                    │  └─────────┘ │  vector)  │ └───────────────┘  │
                    │              └─────┬─────┘                    │
                    │         ┌──────────┼──────────┐               │
                    │  ┌──────▼──────┐ ┌─▼────────┐                │
                    │  │ chunks_vec  │ │chunks_fts │                │
                    │  │ (sqlite-vec)│ │ (FTS5)    │                │
                    │  └─────────────┘ └──────────┘                │
                    └───────────────────────┬───────────────────────┘
                                            │
                    ┌───────────────────────▼───────────────────────┐
                    │               搜索管道                         │
                    │                                               │
                    │  query → [Vector Search] + [FTS Search]       │
                    │       → Hybrid Merge (0.7:0.3)                │
                    │       → [MMR] → [Temporal Decay]              │
                    │       → filter(minScore) → top(maxResults)    │
                    └───────────────────────┬───────────────────────┘
                                            │
                    ┌───────────────────────▼───────────────────────┐
                    │            Agent 工具层                        │
                    │                                               │
                    │  memory_search(query) → snippets[]            │
                    │  memory_get(path, from, lines) → text         │
                    └───────────────────────┬───────────────────────┘
                                            │
                    ┌───────────────────────▼───────────────────────┐
                    │          会话上下文管理                         │
                    │                                               │
                    │  ┌─────────────────────────────────────────┐  │
                    │  │ Memory Flush: 压缩前写入持久记忆         │  │
                    │  │ → memory/YYYY-MM-DD.md                  │  │
                    │  └─────────────────────────────────────────┘  │
                    │                     ↓                         │
                    │  ┌─────────────────────────────────────────┐  │
                    │  │ Auto-compaction: 旧消息 → LLM 摘要      │  │
                    │  │ → 持久化到 JSONL                        │  │
                    │  └─────────────────────────────────────────┘  │
                    └───────────────────────────────────────────────┘
```
