# OpenClaw Agent 记忆系统实现详解

本文档基于当前源码梳理 OpenClaw 的 agent 记忆系统实现，涵盖架构、数据结构、存储路径、索引与检索流程等关键细节。

---

## 目录

1. [架构概览](#1-架构概览)
2. [数据存储架构](#2-数据存储架构)
3. [记忆存储与索引](#3-记忆存储与索引)
4. [记忆检索机制](#4-记忆检索机制)
5. [记忆类型与来源](#5-记忆类型与来源)
6. [Agent 系统集成](#6-agent-系统集成)
7. [向量数据库与嵌入](#7-向量数据库与嵌入)
8. [记忆压缩与总结](#8-记忆压缩与总结)
9. [关键实现文件](#9-关键实现文件)
10. [配置选项](#10-配置选项)
11. [最佳实践](#11-最佳实践)

---

## 1. 架构概览

OpenClaw 的记忆系统围绕 **工作区 Markdown 文件 + SQLite 向量索引** 构建，主要特性如下：

### 核心原则

1. **文件即真相**
   - 记忆以 Markdown 文件存放在 agent 工作区中。
   - 支持 Git 版本控制，便于编辑和回溯。

2. **多层记忆来源**
   - **工作区记忆**：`MEMORY.md` / `memory.md` 与 `memory/*.md`。
   - **会话记忆（可选）**：session transcript 的文本摘要（实验性）。
   - **工具入口**：记忆工具由 memory 插件提供（默认 `memory-core`），可通过 `plugins.slots.memory = "none"` 禁用。

3. **增量索引与安全重建**
   - 文件哈希用于判断是否需要重建。
   - 索引配置变更触发全量重建，使用临时数据库安全替换。

4. **混合检索**
   - **向量相似度** + **FTS5 BM25**（可用时）
   - 支持权重融合与候选池扩展。

5. **Agent 隔离**
   - 每个 agent 的索引数据库、会话转录均独立存储。

---

## 2. 数据存储架构

### 2.1 存储位置

默认状态目录为 `~/.openclaw`（可通过 `OPENCLAW_STATE_DIR` 覆盖，兼容 legacy 目录）。默认工作区由 `OPENCLAW_PROFILE` 决定：

```
~/.openclaw/
├── workspace/                      # 默认 agent 工作区
│   ├── MEMORY.md                  # 长期记忆（可选）
│   ├── memory.md                  # MEMORY.md 不存在时的替代文件
│   └── memory/                    # 记忆日志目录（可选，按需创建）
│       ├── 2026-01-15.md
│       └── 2026-01-16.md
├── agents/
│   └── <agentId>/
│       └── sessions/
│           ├── sessions.json      # 会话元数据
│           └── <sessionId>.jsonl  # 会话转录（JSONL）
└── memory/
    └── <agentId>.sqlite           # 记忆索引数据库
```

补充说明：
- 会话文件名可能带 `-topic-<id>` 后缀（见 `resolveSessionTranscriptPath`）。
- 记忆索引路径可在配置中通过 `agents.defaults.memorySearch.store.path` 覆盖，支持 `{agentId}` 占位符。
- `MEMORY.md` / `memory.md` 会作为 bootstrap 文件注入上下文；`memory/YYYY-MM-DD.md` 不会自动注入，需要通过 `memory_search`/`memory_get` 获取。

### 2.2 SQLite 数据库架构

位于 `src/memory/memory-schema.ts`。核心表如下：

**1. `meta` 表**
```sql
CREATE TABLE meta (
  key TEXT PRIMARY KEY,
  value TEXT NOT NULL
);
```
- 使用 `memory_index_meta_v1` 记录：
  - `provider` / `model`
  - `providerKey`
  - `chunkTokens` / `chunkOverlap`
  - `vectorDims`（仅在 vector 可用时记录）

**2. `files` 表**
```sql
CREATE TABLE files (
  path TEXT PRIMARY KEY,
  source TEXT NOT NULL DEFAULT 'memory',
  hash TEXT NOT NULL,
  mtime INTEGER NOT NULL,
  size INTEGER NOT NULL
);
```
- `source` 为 `memory` 或 `sessions`。

**3. `chunks` 表**
```sql
CREATE TABLE chunks (
  id TEXT PRIMARY KEY,
  path TEXT NOT NULL,
  source TEXT NOT NULL DEFAULT 'memory',
  start_line INTEGER NOT NULL,
  end_line INTEGER NOT NULL,
  hash TEXT NOT NULL,
  model TEXT NOT NULL,
  text TEXT NOT NULL,
  embedding TEXT NOT NULL,
  updated_at INTEGER NOT NULL
);
```
- `embedding` 为 JSON 数组（字符串）。
- `start_line` / `end_line` 为 1-based 行号。

**4. `embedding_cache` 表**
```sql
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
```
- 可选 `maxEntries` 时按 `updated_at` 近似清理旧记录。

**5. `chunks_vec` 虚拟表（sqlite-vec）**
```sql
CREATE VIRTUAL TABLE chunks_vec USING vec0(
  id TEXT PRIMARY KEY,
  embedding FLOAT[<dimensions>]
);
```
- 仅在 sqlite-vec 可用时启用。
- 实际写入为 Float32Array BLOB（`Buffer.from(new Float32Array(...).buffer)`）。

**6. `chunks_fts` 虚拟表（FTS5）**
```sql
CREATE VIRTUAL TABLE chunks_fts USING fts5(
  text,
  id UNINDEXED,
  path UNINDEXED,
  source UNINDEXED,
  model UNINDEXED,
  start_line UNINDEXED,
  end_line UNINDEXED
);
```
- 仅在 FTS5 可用且 hybrid 启用时创建。

---

## 3. 记忆存储与索引

### 3.1 文件发现与同步

**位置**：`src/memory/internal.ts`, `src/memory/manager.ts`

- 默认扫描：
  - `MEMORY.md`
  - `memory.md`
  - `memory/**/*.md`
- 支持 `memorySearch.extraPaths` 追加目录或单个 `.md` 文件。
- 扫描与读取过程中会忽略符号链接。

**同步触发条件**
1. 会话启动时（`sync.onSessionStart`）
2. 搜索时（`sync.onSearch` 且索引标记为脏）
3. 文件监控器触发（chokidar）
4. 定时同步（`sync.intervalMinutes`）
5. 会话 transcript 变更达到阈值（见 3.4）

**文件监控器**
- 使用 chokidar，`ignoreInitial: true`。
- `awaitWriteFinish.stabilityThreshold` 与 `sync.watchDebounceMs` 一致。
- 事件触发后再由 `sync.watchDebounceMs` 二次去抖。

### 3.2 分块策略

**位置**：`src/memory/internal.ts` - `chunkMarkdown()`

核心逻辑：
- `maxChars = max(32, tokens * 4)`
- `overlapChars = overlap * 4`
- 以行切分文本，但 **单行过长时会按字符段落切开**。

要点：
- **可能在行中间切分**（当单行长度超过 `maxChars`）。
- `lineNo` 保持原行号；同一行被切分后，多个块可能共享同一行号。
- 重叠窗口通过保留末尾 `overlapChars` 的行片段实现。

### 3.3 嵌入生成与索引写入

**位置**：`src/memory/manager.ts`

流程概览：
1. 读取文件内容并分块
2. 查询缓存（`embedding_cache`）
3. 生成嵌入（批处理）
4. 写入 `chunks` / `chunks_vec` / `chunks_fts`
5. 更新 `files` 与 `meta`

注意：
- 估算 token 使用 **1 字符 ≈ 1 token**（`EMBEDDING_APPROX_CHARS_PER_TOKEN = 1`），偏保守。
- `EMBEDDING_BATCH_MAX_TOKENS = 8000` 用于切分 embedding 批次。

### 3.4 会话增量触发机制

**位置**：`src/memory/manager.ts`

- `onSessionTranscriptUpdate` 只用于 **触发** session reindex，不做增量内容解析。
- 触发条件基于 **文件大小增量** 和 **新增换行数**（`deltaBytes` / `deltaMessages`）。
- 达到阈值后会对 해당 session 文件执行 **完整重读与重建**（并非从 offset 增量读取）。
- 索引在后台异步执行，`memory_search` 不会阻塞等待；结果可能短暂滞后。

---

## 4. 记忆检索机制

### 4.1 检索流程

**位置**：`src/memory/manager.ts`, `src/memory/manager-search.ts`

```
1. 可选触发 sync（onSearch + dirty）
2. 生成查询 embedding
3. 并行检索：
   - 向量检索（sqlite-vec or JS cosine）
   - 关键词检索（FTS5 BM25，若可用）
4. 混合融合（可选）
5. 分数过滤与截断
```

### 4.2 向量检索

- sqlite-vec 可用时：
  ```sql
  SELECT c.id, c.path, c.start_line, c.end_line, c.text,
         c.source, vec_distance_cosine(v.embedding, ?) AS dist
  FROM chunks_vec v
  JOIN chunks c ON c.id = v.id
  WHERE c.model = ?
  ORDER BY dist ASC
  LIMIT ?;
  ```
- 不可用时：加载 `chunks.embedding` JSON，JS 内存计算余弦相似度。

向量得分：`score = 1 - dist`。

### 4.3 关键词检索（FTS5）

- 仅在 `hybrid.enabled = true` 且 FTS5 创建成功时启用。
- `buildFtsQuery()` 只保留 `[A-Za-z0-9_]` token，并用 `AND` 连接。

BM25 排名转分数：
```
textScore = 1 / (1 + max(0, rank))
```

### 4.4 混合融合

```
score = vectorWeight * vectorScore + textWeight * textScore
```

- `vectorWeight` / `textWeight` 会自动归一化。
- 候选池大小：`maxResults * candidateMultiplier`，上限 200。
- 当 hybrid 禁用时，只返回向量结果。
- 查询 embedding 失败会直接抛错，**不会自动降级为纯关键词搜索**；只有 embedding 返回全零向量时，才会走关键词结果（若 FTS 可用）。

### 4.5 Snippet 截断

`SNIPPET_MAX_CHARS = 700`，使用 `truncateUtf16Safe`。

---

## 5. 记忆类型与来源

### 5.1 工作区记忆

- `MEMORY.md` / `memory.md`：长期整理后的事实和偏好。
- `memory/*.md`：运行中的笔记、日志或每日条目。

**注意**：系统不会自动生成每日记忆文件，通常由人或 memory flush 写入。
文档层面建议“会话启动时读取今天+昨天”，但实现上并不会自动注入 daily files，只能通过 `memory_search` 拉取。
另外，bootstrap 注入只对 subagent 做了白名单过滤；当前实现并未按群组上下文额外排除 `MEMORY.md`（是否仅在主/私聊加载需由上层策略控制）。

### 5.2 会话记忆（实验性）

- 来源：`~/.openclaw/agents/<agentId>/sessions/*.jsonl`
- 只提取 `type: "message"` 的记录，并且角色为 `user` / `assistant`。
- 内容提取规则：仅保留 text block（`{ type: "text", text: "..." }`）。
- 索引文本格式：`User: ...` / `Assistant: ...`。

### 5.3 语义记忆

- 通过 embeddings 写入 `chunks` + `chunks_vec`。
- 向量维度取决于模型，系统不会假设固定维度。

---

## 6. Agent 系统集成

**工具实现**：`src/agents/tools/memory-tool.ts`（由 memory 插件提供，默认 `memory-core`）

- `memory_search`
  - 触发搜索，返回片段 + 行号 + 评分。
- `memory_get`
  - 读取指定文件行范围（安全限制在 memory 文件或 extraPaths）。
  
两者仅在 `memorySearch.enabled` 生效时注册。

`memory_get` 限制：
- 仅允许 `.md` 文件
- 必须在工作区记忆路径或 `extraPaths`
- 避免符号链接

---

## 7. 向量数据库与嵌入

### 7.1 sqlite-vec 加速

**加载方式**：`src/memory/sqlite-vec.ts`
- 默认通过 `sqlite-vec` 包加载扩展。
- 可通过 `memorySearch.store.vector.extensionPath` 覆盖路径。

### 7.2 嵌入提供商与选择逻辑

**位置**：`src/memory/embeddings.ts`

- `provider = "auto"`（默认）时：
  1) 若 `local.modelPath` 是本地文件且存在，则优先 local
  2) 否则依次尝试 OpenAI、Gemini
- `fallback` 默认为 `none`，可设置为 `openai` / `gemini` / `local`

默认模型：
- OpenAI: `text-embedding-3-small`
- Gemini: `gemini-embedding-001`
- Local: `hf:ggml-org/embeddinggemma-300M-GGUF/embeddinggemma-300M-Q8_0.gguf`

### 7.3 批量嵌入

- 常规批处理：将文本分批并调用 `embedBatch`，用于 **所有** provider。
- OpenAI / Gemini：可启用 Batch API（默认开启，需 provider 支持）
  - OpenAI: file upload + `batches` + 24h completion window
  - Gemini: `asyncBatchEmbedContent`
  - 失败时会自动退回非 Batch API。

关键限制：
- `EMBEDDING_BATCH_MAX_TOKENS = 8000`
- `BATCH_FAILURE_LIMIT = 2`
- 远程批量超时默认 2 分钟（针对非 Batch API），Batch API 轮询超时由 `timeoutMinutes` 控制（默认 60 分钟）

### 7.4 嵌入缓存

**缓存键**：`provider + model + provider_key + hash`

`provider_key` 计算（`src/memory/provider-key.ts`）：
- 对 OpenAI / Gemini：包含 baseUrl、model、header **名称** 的哈希（不包含 header 值）。
- 对 local：包含 providerId 与 model。

**缓存策略**：
- 默认启用
- 未设置 `maxEntries` 时不主动清理
- 设置 `maxEntries` 时按 `updated_at` 删除最旧记录

### 7.5 本地模型下载与缓存

- `provider = "local"` 且 `modelPath` 为 `hf:`/URL 时，`node-llama-cpp` 会在首次使用时解析并下载模型到缓存目录（`modelCacheDir` 或默认缓存）。
- `provider = "auto"` 时只有在本地文件路径存在的情况下才会优先 local；`hf:`/URL 不会触发自动选择。

---

## 8. 记忆压缩与总结

### 8.1 会话压缩

- 压缩逻辑由会话管理负责（见 `docs/concepts/compaction.md`）。
- 记忆索引本身不会主动总结内容。

### 8.2 Pre-compaction memory flush

- 当接近 compaction 阈值时，系统可触发一次 **memory flush** 的 agentic turn，提醒将持久信息写入 `memory/YYYY-MM-DD.md`。
- 逻辑位于 `src/auto-reply/reply/memory-flush.ts`。

---

## 9. 关键实现文件

### 核心模块

- `src/memory/manager.ts`：索引管理、同步、搜索主流程
- `src/memory/internal.ts`：分块、文件发现、辅助函数
- `src/memory/memory-schema.ts`：SQLite schema
- `src/memory/manager-search.ts`：向量 / 关键词检索
- `src/memory/hybrid.ts`：混合融合逻辑
- `src/memory/sqlite-vec.ts`：sqlite-vec 加载
- `src/memory/batch-openai.ts`：OpenAI Batch API
- `src/memory/batch-gemini.ts`：Gemini Batch API
- `src/memory/session-files.ts`：session transcript 读取与抽取
- `src/memory/sync-memory-files.ts` / `src/memory/sync-session-files.ts`

### 嵌入提供商

- `src/memory/embeddings.ts`
- `src/memory/embeddings-openai.ts`
- `src/memory/embeddings-gemini.ts`
- `src/memory/node-llama.ts`
- `src/memory/provider-key.ts`

### Agent 集成

- `src/agents/tools/memory-tool.ts`
- `src/agents/memory-search.ts`
- `src/auto-reply/reply/memory-flush.ts`

---

## 10. 配置选项

以下为与当前实现一致的简化示例（字段与默认值）：

```json5
{
  "agents": {
    "defaults": {
      "workspace": "~/.openclaw/workspace",
      "memorySearch": {
        "enabled": true,
        "provider": "openai",
        "model": "text-embedding-3-small",
        "fallback": "none",
        "remote": {
          "apiKey": "sk-...",
          "baseUrl": "https://api.openai.com/v1",
          "batch": {
            "enabled": true,
            "wait": true,
            "concurrency": 2,
            "pollIntervalMs": 2000,
            "timeoutMinutes": 60
          }
        },
        "local": {
          "modelPath": "hf:ggml-org/embeddinggemma-300M-GGUF/embeddinggemma-300M-Q8_0.gguf",
          "modelCacheDir": "~/.openclaw/models"
        },
        "chunking": {
          "tokens": 400,
          "overlap": 80
        },
        "query": {
          "maxResults": 6,
          "minScore": 0.35,
          "hybrid": {
            "enabled": true,
            "vectorWeight": 0.7,
            "textWeight": 0.3,
            "candidateMultiplier": 4
          }
        },
        "store": {
          "path": "~/.openclaw/memory/{agentId}.sqlite",
          "vector": {
            "enabled": true,
            "extensionPath": "/path/to/sqlite-vec/vec0.dylib"
          }
        },
        "cache": {
          "enabled": true,
          "maxEntries": 50000
        },
        "sync": {
          "watch": true,
          "watchDebounceMs": 1500,
          "onSessionStart": true,
          "onSearch": true,
          "intervalMinutes": 0,
          "sessions": {
            "deltaBytes": 100000,
            "deltaMessages": 50
          }
        },
        "extraPaths": ["../team-docs", "/srv/shared-notes/overview.md"],
        "sources": ["memory"],
        "experimental": {
          "sessionMemory": false
        }
      }
    }
  }
}
```

注意：
- `cache.maxEntries` 无默认上限，示例值仅用于说明。
- `memorySearch.provider` 支持 `openai | gemini | local`；未设置时按 `auto` 选择逻辑处理。

---

## 11. 最佳实践

1. **将长期事实写入 `MEMORY.md`**，把运行日志放入 `memory/YYYY-MM-DD.md`。
2. **启用 batch API** 可降低索引过程中的请求次数（但仍需关注 provider 支持与超时）。
3. **开启 sqlite-vec** 可显著提升向量检索性能。
4. **sessionMemory 仅在需要时启用**，会增加索引成本。
5. **避免巨型单行文本**，否则 chunking 会在行内切分并共享行号。

---

**文档版本**: v2.0
**最后更新**: 2026-02-02
**维护者**: OpenClaw Team
