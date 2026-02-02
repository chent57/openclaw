# OpenClaw Agent 记忆系统实现详解

本文档深入剖析 OpenClaw 项目中 agent 记忆系统的完整实现细节，包括架构设计、数据结构、存储机制、检索流程等核心技术。

---

## 目录

1. [架构概览](#1-架构概览)
2. [数据存储架构](#2-数据存储架构)
3. [记忆存储机制](#3-记忆存储机制)
4. [记忆检索机制](#4-记忆检索机制)
5. [记忆类型](#5-记忆类型)
6. [Agent 系统集成](#6-agent-系统集成)
7. [向量数据库与嵌入](#7-向量数据库与嵌入)
8. [记忆压缩与总结](#8-记忆压缩与总结)
9. [关键实现文件](#9-关键实现文件)
10. [配置选项](#10-配置选项)
11. [最佳实践](#11-最佳实践)

---

## 1. 架构概览

OpenClaw 的记忆系统采用**混合架构**，结合了以下核心设计理念：

### 核心原则

1. **文件即真相 (Files as Source of Truth)**
   - 记忆以 **纯 Markdown 文件** 形式存储
   - 无专有格式，易于编辑、备份和版本控制
   - 支持 Git 版本控制，团队协作友好

2. **多层记忆架构**
   - **短期记忆**: 会话内上下文（Session Context）
   - **长期记忆**: 持久化 Markdown 文件
   - **情节记忆**: 会话历史记录（Session Transcripts）
   - **语义记忆**: 向量嵌入索引

3. **增量索引策略**
   - 只处理变更的文件
   - 基于内容哈希的去重
   - 增量式会话日志索引

4. **混合检索**
   - 向量相似度搜索（语义匹配）
   - BM25 关键词搜索（精确匹配）
   - 加权融合策略

5. **智能缓存机制**
   - 嵌入向量缓存避免重复调用
   - LRU 淘汰策略
   - 跨文件/会话的去重

6. **Agent 隔离**
   - 每个 agent 拥有独立的索引和会话
   - 独立的工作空间和存储路径

---

## 2. 数据存储架构

### 2.1 存储位置

```
~/.openclaw/
├── workspace/                      # Agent 工作空间（默认）
│   ├── MEMORY.md                  # 长期记忆（可选）
│   └── memory/                    # 日志目录
│       ├── 2024-01-15.md         # 每日日志
│       └── 2024-01-16.md
├── agents/
│   └── <agentId>/
│       └── sessions/              # 会话数据
│           ├── sessions.json      # 会话元数据
│           └── <SessionId>.jsonl  # 会话转录（JSONL）
└── memory/
    └── <agentId>.sqlite           # 记忆索引数据库
```

### 2.2 SQLite 数据库架构

位于: `src/memory/memory-schema.ts`

#### 核心表结构

**1. `meta` 表 - 索引元数据**
```sql
CREATE TABLE meta (
  key TEXT PRIMARY KEY,
  value TEXT NOT NULL
);
```
存储内容：
- `memory_index_meta_v1`: JSON 对象包含
  - `provider`: 嵌入提供商（openai/gemini/local）
  - `model`: 模型名称
  - `providerKey`: 提供商指纹（端点+配置哈希）
  - `chunkTokens`: 分块大小（默认 400 tokens）
  - `chunkOverlap`: 重叠大小（默认 80 tokens）
  - `vectorDims`: 向量维度（由首次嵌入确定）

**2. `files` 表 - 文件索引**
```sql
CREATE TABLE files (
  path TEXT PRIMARY KEY,              -- 相对路径
  source TEXT NOT NULL DEFAULT 'memory', -- 来源: 'memory' | 'sessions'
  hash TEXT NOT NULL,                 -- SHA-256 内容哈希
  mtime INTEGER NOT NULL,             -- 修改时间戳（毫秒）
  size INTEGER NOT NULL               -- 文件大小（字节）
);
```

**3. `chunks` 表 - 文本块与嵌入**
```sql
CREATE TABLE chunks (
  id TEXT PRIMARY KEY,                -- UUID
  path TEXT NOT NULL,                 -- 文件相对路径
  source TEXT NOT NULL DEFAULT 'memory',
  start_line INTEGER NOT NULL,        -- 起始行号（1-based）
  end_line INTEGER NOT NULL,          -- 结束行号（包含）
  hash TEXT NOT NULL,                 -- 文本块哈希
  model TEXT NOT NULL,                -- 嵌入模型
  text TEXT NOT NULL,                 -- 原始文本
  embedding TEXT NOT NULL,            -- JSON 编码的向量数组
  updated_at INTEGER NOT NULL         -- 更新时间戳
);

CREATE INDEX idx_chunks_path ON chunks(path);
CREATE INDEX idx_chunks_source ON chunks(source);
```

**4. `chunks_vec` 虚拟表 - 向量加速（sqlite-vec）**
```sql
CREATE VIRTUAL TABLE chunks_vec USING vec0(
  id TEXT PRIMARY KEY,
  embedding FLOAT[<dimensions>]       -- 原生向量存储
);
```
- 使用 sqlite-vec 扩展提供原生向量操作
- 支持快速余弦相似度计算
- 动态维度（根据首个嵌入确定）
- 不可用时自动降级到 JS 内存计算

**5. `chunks_fts` 虚拟表 - 全文搜索（FTS5）**
```sql
CREATE VIRTUAL TABLE chunks_fts USING fts5(
  text,                               -- 索引的文本内容
  id UNINDEXED,                       -- 块 ID（不索引）
  path UNINDEXED,                     -- 文件路径（不索引）
  source UNINDEXED,                   -- 来源（不索引）
  model UNINDEXED,                    -- 模型（不索引）
  start_line UNINDEXED,               -- 起始行（不索引）
  end_line UNINDEXED                  -- 结束行（不索引）
);
```
- 使用 SQLite FTS5 提供 BM25 排序
- 仅 `text` 字段被索引
- 其他字段作为元数据存储（UNINDEXED）

**6. `embedding_cache` 表 - 嵌入缓存**
```sql
CREATE TABLE embedding_cache (
  provider TEXT NOT NULL,             -- openai/gemini/local
  model TEXT NOT NULL,                -- 模型名称
  provider_key TEXT NOT NULL,         -- 提供商指纹
  hash TEXT NOT NULL,                 -- 文本哈希（SHA-256）
  embedding TEXT NOT NULL,            -- JSON 编码的向量
  dims INTEGER,                       -- 向量维度
  updated_at INTEGER NOT NULL,        -- 更新时间戳
  PRIMARY KEY (provider, model, provider_key, hash)
);

CREATE INDEX idx_embedding_cache_updated_at ON embedding_cache(updated_at);
```

### 2.3 会话文件格式

**位置**: `~/.openclaw/agents/<agentId>/sessions/`

**sessions.json** - 会话元数据
```json
{
  "sessionKey": {
    "sessionId": "uuid",
    "totalTokens": 12500,
    "inputTokens": 8000,
    "outputTokens": 4500,
    "compactionCount": 2,
    "memoryFlushAt": 1,
    "memoryFlushCompactionCount": 1,
    "createdAt": "2024-01-15T10:00:00Z",
    "updatedAt": "2024-01-15T11:00:00Z"
  }
}
```

**<SessionId>.jsonl** - 会话转录（每行一个 JSON 对象）
```jsonl
{"role":"user","content":"...", "timestamp":"2024-01-15T10:00:00Z"}
{"role":"assistant","content":"...", "timestamp":"2024-01-15T10:00:05Z"}
{"role":"tool_result","tool_call_id":"...", "content":"..."}
```

---

## 3. 记忆存储机制

### 3.1 文件监控与同步

**位置**: `src/memory/manager.ts` - `MemoryIndexManager` 类

**文件监控器 (File Watcher)**
```typescript
// 使用 chokidar 监控文件变化
private watcher?: FSWatcher;

// 监控目标
- MEMORY.md
- memory/**/*.md
- extraPaths 中配置的路径
- 会话转录目录（可选）

// 防抖设置
- 默认 1.5 秒防抖
- 避免频繁重索引
```

**同步触发条件**
1. **会话启动时**: `memorySearch.sync.onSessionStart = true`
2. **搜索时**: `memorySearch.sync.onSearch = true` 且索引标记为脏
3. **文件变更**: 文件监控器检测到变化（防抖后）
4. **定时同步**: `memorySearch.sync.intervalMinutes` 配置
5. **会话增量同步**: 超过阈值时触发
   - `deltaBytes >= 100KB`
   - `deltaMessages >= 50 条`

### 3.2 分块策略 (Chunking)

**位置**: `src/memory/internal.ts` - `chunkMarkdown()`

**分块算法**
```typescript
// 默认配置
const DEFAULT_CHUNK_TOKENS = 400;      // ~1600 字符
const DEFAULT_CHUNK_OVERLAP = 80;      // ~320 字符

// 工作原理
1. 按行分割 Markdown 文本
2. 累积行直到达到 maxChars（tokens * 4）
3. 创建块并记录 startLine/endLine
4. 保留 overlapChars 到下一个块（滑动窗口）
5. 计算每个块的 SHA-256 哈希
```

**分块特点**
- **保持行边界**: 不会在行中间切分
- **重叠窗口**: 避免关键信息被切分在边界
- **哈希去重**: 相同内容的块不会重复嵌入
- **行号追踪**: 精确定位源文件位置

**示例**
```markdown
# 文档标题
这是第一段内容。
这是第二段内容。
这是第三段内容。

## 第二部分
更多内容...
```

分块后:
```
Chunk 1: lines 1-3 (with overlap)
Chunk 2: lines 2-5 (overlap from previous + new content)
Chunk 3: lines 4-7 (overlap from previous + new content)
```

### 3.3 嵌入生成流程

**位置**: `src/memory/manager.ts` - 索引管道

**同步流程**
```
1. 文件发现
   ├─ 扫描 MEMORY.md, memory/*.md
   ├─ 扫描 extraPaths
   └─ 可选: 扫描会话转录

2. 变更检测
   ├─ 计算文件 SHA-256 哈希
   ├─ 对比数据库中的 files 表
   └─ 跳过未变更文件

3. 文本分块
   ├─ 按配置的 tokens/overlap 切分
   ├─ 保持行边界
   └─ 计算块哈希

4. 缓存查找
   ├─ 查询 embedding_cache 表
   ├─ 键: (provider, model, provider_key, text_hash)
   └─ 命中: 跳过嵌入 API 调用

5. 批量嵌入
   ├─ OpenAI: Batch API (异步，折扣价)
   ├─ Gemini: Async Batch Embeddings API
   ├─ Local: node-llama-cpp (同步)
   └─ 并发控制 + 重试逻辑

6. 存储
   ├─ 插入/更新 chunks 表
   ├─ 更新 embedding_cache
   ├─ 填充 chunks_vec 虚拟表
   └─ 填充 chunks_fts 虚拟表

7. 清理
   └─ 删除过时的块和文件记录
```

**批量处理 (OpenAI Batch API)**

位置: `src/memory/batch-openai.ts`

```typescript
// 批量配置
const BATCH_MAX_TOKENS = 8000;          // 单批最大 tokens
const BATCH_CONCURRENCY = 2;            // 并发批任务数
const BATCH_TIMEOUT_REMOTE = 2 * 60_000; // 2 分钟超时
const BATCH_FAILURE_LIMIT = 2;          // 失败容忍次数

// 工作流
1. 分组: 将文本块按 token 估算分组
2. 提交: 并发提交多个批任务到 OpenAI
3. 轮询: 定期检查批任务状态
4. 下载: 批任务完成后下载结果
5. 降级: 超过失败限制后降级到同步模式
```

**优势**
- **成本优化**: Batch API 提供折扣价格（~50% off）
- **高吞吐**: 单个批任务可处理数千个嵌入
- **异步处理**: 不阻塞主流程

**Gemini 批量处理**

位置: `src/memory/batch-gemini.ts`

- 类似 OpenAI，使用 Gemini Batch API
- 异步提交和轮询
- 自动降级到同步模式

### 3.4 增量会话索引

**位置**: `src/memory/manager.ts` - 会话同步逻辑

**Delta 触发机制**
```typescript
// 触发阈值
const SESSION_DELTA_BYTES = 100_000;    // 100 KB
const SESSION_DELTA_MESSAGES = 50;       // 50 条消息

// 会话文件监听
onSessionTranscriptUpdate((event) => {
  if (event.deltaBytes >= threshold ||
      event.deltaMessages >= threshold) {
    scheduleDeltaSync(event.sessionKey);
  }
});
```

**Delta 读取策略**
```typescript
const SESSION_DELTA_READ_CHUNK_BYTES = 64 * 1024; // 64 KB

// 只读取新增内容
1. 从 files 表获取上次索引的文件大小
2. 使用 fs.createReadStream 从 offset 开始读取
3. 解析新增的 JSONL 行
4. 仅对新内容进行分块和嵌入
5. 更新 files 表的大小和哈希
```

**特点**
- **非阻塞**: 后台异步处理
- **增量式**: 只处理新增内容
- **最终一致性**: 搜索结果可能略微滞后
- **可配置**: 可禁用会话索引（默认禁用）

---

## 4. 记忆检索机制

### 4.1 混合搜索架构

**位置**: `src/memory/hybrid.ts`, `src/memory/manager-search.ts`

**核心理念**: 结合语义理解（向量）和精确匹配（关键词）的优势

**向量搜索 vs BM25 比较**

| 方面 | 向量搜索 | BM25 全文搜索 |
|------|----------|---------------|
| **适用场景** | 语义相似、改写查询 | 精确 token 匹配 |
| **示例强项** | "Mac Studio gateway" vs "运行网关的机器" | IDs、代码符号、错误字符串 |
| **弱点** | 对精确标识符敏感度低 | 对改写/同义词不敏感 |
| **技术** | 余弦相似度 | BM25 排序算法 |

### 4.2 混合搜索流程

**位置**: `src/memory/manager.ts` - `search()` 方法

```typescript
async search(query: string, options?: SearchOptions): Promise<MemorySearchResult[]>
```

**步骤详解**

```
1. 查询嵌入
   ├─ 使用配置的 provider 生成查询向量
   ├─ 超时保护 (远程: 60s, 本地: 5min)
   └─ 失败降级到纯关键词搜索

2. 并行检索
   ├─ 向量搜索
   │  ├─ sqlite-vec: 原生 SQL 查询
   │  ├─ 降级: JS 内存计算余弦相似度
   │  └─ 返回 top (maxResults * candidateMultiplier)
   │
   └─ BM25 关键词搜索
      ├─ 查询预处理: buildFtsQuery()
      ├─ FTS5 查询: MATCH '...'
      ├─ BM25 rank 转 score: 1 / (1 + rank)
      └─ 返回 top (maxResults * candidateMultiplier)

3. 混合融合
   ├─ 合并结果集（按 chunk id）
   ├─ 计算加权分数:
   │  score = vectorWeight * vectorScore + textWeight * textScore
   ├─ 排序: 分数从高到低
   └─ 应用 minScore 阈值过滤

4. 截断与返回
   ├─ 取 top maxResults 结果
   ├─ 截断 snippet 到 ~700 字符
   └─ 返回: { path, startLine, endLine, score, snippet, source }
```

**配置示例**
```json5
{
  "memorySearch": {
    "query": {
      "maxResults": 6,
      "minScore": 0.35,
      "hybrid": {
        "enabled": true,
        "vectorWeight": 0.7,        // 70% 语义权重
        "textWeight": 0.3,          // 30% 关键词权重
        "candidateMultiplier": 4    // 每侧检索 6*4=24 个候选
      }
    }
  }
}
```

### 4.3 查询预处理

**位置**: `src/memory/hybrid.ts` - `buildFtsQuery()`

**FTS5 查询构建**
```typescript
// 输入: "memorySearch.query.hybrid vectorScore"
// 输出: "memorySearch" AND "query" AND "hybrid" AND "vectorScore"

function buildFtsQuery(raw: string): string | null {
  const tokens = raw
    .match(/[A-Za-z0-9_]+/g)   // 提取字母数字 tokens
    ?.map(t => `"${t}"`)        // 引号包裹
    .filter(Boolean);

  return tokens.length > 0
    ? tokens.join(" AND ")      // AND 连接
    : null;
}
```

**特点**
- 自动提取有效 tokens
- 转义特殊字符
- AND 逻辑（所有词都必须存在）
- 无效查询返回 null

### 4.4 分数融合策略

**当前实现** (简单加权平均)

```typescript
// 向量分数: 余弦相似度 (0..1)
vectorScore = cosineSimilarity(queryEmbedding, chunkEmbedding);

// BM25 分数转换
textScore = 1 / (1 + max(0, bm25Rank));  // rank 越小越好

// 最终分数
finalScore = vectorWeight * vectorScore + textWeight * textScore;
```

**归一化**: `vectorWeight + textWeight = 1.0`（配置时自动归一化）

**未来改进方向**
1. **Reciprocal Rank Fusion (RRF)**
   ```
   score = Σ [ 1 / (k + rank_i) ]  for each retrieval method
   ```
2. **分数归一化**
   - Min-Max 归一化
   - Z-score 标准化
3. **动态权重**
   - 根据查询类型自动调整权重

### 4.5 向量相似度计算

**SQLite-vec 模式** (首选)

位置: `src/memory/sqlite-vec.ts`

```sql
SELECT
  id, path, start_line, end_line, source, text,
  vec_distance_cosine(embedding, ?) AS distance
FROM chunks_vec
ORDER BY distance ASC
LIMIT ?;
```

- 原生 C 扩展，性能优异
- 直接在 SQL 中计算距离
- 不需要加载所有向量到 JS 内存

**JS 降级模式**

```typescript
function cosineSimilarity(a: number[], b: number[]): number {
  let dotProduct = 0;
  let normA = 0;
  let normB = 0;

  for (let i = 0; i < a.length; i++) {
    dotProduct += a[i] * b[i];
    normA += a[i] * a[i];
    normB += b[i] * b[i];
  }

  return dotProduct / (Math.sqrt(normA) * Math.sqrt(normB));
}
```

- sqlite-vec 不可用时使用
- 从 chunks 表读取所有 embedding JSON
- 在 JS 内存中计算相似度
- 性能较低，但功能完整

---

## 5. 记忆类型

OpenClaw 实现了多种记忆类型，对应认知心理学中的记忆分类。

### 5.1 短期记忆 (Working Memory)

**实现**: 会话上下文（Session Context）

**位置**:
- `src/config/sessions.ts` - 会话管理
- `src/agents/context.ts` - 上下文构建

**特征**
- **易失性**: 仅存在于当前会话
- **容量有限**: 受 LLM 上下文窗口限制
- **实时追踪**: 动态计算 token 使用量
- **自动压缩**: 接近限制时触发总结

**数据结构**
```typescript
type SessionContext = {
  messages: Array<{
    role: "user" | "assistant" | "system" | "tool_result";
    content: string;
    timestamp?: string;
    tool_call_id?: string;
  }>;
  totalTokens: number;
  inputTokens: number;
  outputTokens: number;
};
```

**上下文修剪策略**
```typescript
// 位置: src/agents/context.ts

// 移除旧的工具调用结果（保留最近几轮）
- 保留用户/助手消息
- 移除 N 轮之前的 tool_result
- 不重写会话历史文件
```

### 5.2 长期记忆 (Long-term Memory)

**实现**: 持久化 Markdown 文件

**文件类型**

**1. MEMORY.md - 精炼知识库**
- **用途**: 手动整理的长期事实、偏好、决策
- **特点**:
  - 用户精心策划
  - 跨会话持久
  - 仅在主会话加载（不在群组上下文）
- **示例内容**:
  ```markdown
  # 用户偏好
  - 喜欢简洁的代码风格
  - 使用 TypeScript strict 模式
  - 优先使用函数式编程

  # 项目决策
  - 2024-01-15: 决定使用 SQLite 作为记忆存储
  - 原因: 零配置，跨平台兼容
  ```

**2. memory/YYYY-MM-DD.md - 每日日志**
- **用途**: 自动/半自动记录每日上下文
- **特点**:
  - 追加式写入
  - 按日期组织
  - 自动在会话启动时加载今天+昨天
- **示例内容**:
  ```markdown
  # 2024-01-15

  ## 10:30 - 修复 bug #123
  - 问题: 嵌入缓存未命中
  - 解决: 添加 provider_key 到缓存键

  ## 14:00 - 实现混合搜索
  - 添加 BM25 支持
  - 配置权重: 70% 向量, 30% 文本
  ```

### 5.3 情节记忆 (Episodic Memory)

**实现**: 会话转录（Session Transcripts）

**位置**: `~/.openclaw/agents/<agentId>/sessions/<SessionId>.jsonl`

**格式**: JSONL（每行一个 JSON）
```jsonl
{"role":"user","content":"帮我实现记忆搜索","timestamp":"2024-01-15T10:00:00Z"}
{"role":"assistant","content":"好的，让我先探索现有实现...","timestamp":"2024-01-15T10:00:02Z"}
{"role":"tool_call","name":"memory_search","arguments":"{\"query\":\"memory implementation\"}","tool_call_id":"call_123"}
{"role":"tool_result","tool_call_id":"call_123","content":"[{\"path\":\"src/memory/manager.ts\",\"snippet\":\"...\"}]"}
```

**特征**
- **完整性**: 保留所有对话细节
- **时间序列**: 按时间戳排序
- **工具调用**: 包含工具使用记录
- **可选索引**: 实验性功能，可启用语义搜索

**实验性会话记忆索引**

配置:
```json5
{
  "memorySearch": {
    "experimental": {
      "sessionMemory": true
    },
    "sources": ["memory", "sessions"]
  }
}
```

特点:
- 会话日志被分块和嵌入
- 可通过 `memory_search` 工具检索
- 增量索引（delta 触发）
- 隔离性: 仅索引当前 agent 的会话

### 5.4 语义记忆 (Semantic Memory)

**实现**: 向量嵌入索引

**本质**: 将文本转换为高维向量表示，捕获语义关系

**向量空间特性**
```
"猫" ≈ "小猫" ≈ "feline"
"国王" - "男人" + "女人" ≈ "女王"
"编程" ≈ "代码" ≈ "开发"
```

**存储位置**:
- `chunks` 表的 `embedding` 列（JSON）
- `chunks_vec` 虚拟表（原生 FLOAT 数组）
- `embedding_cache` 表（缓存）

**维度**
- OpenAI text-embedding-3-small: **1536 维**
- Gemini embedding-001: **768 维**
- 本地 embeddinggemma-300M: **256 维**

**跨会话共享**
- 所有会话共享同一个索引
- 新会话可立即访问历史记忆

---

## 6. Agent 系统集成

### 6.1 工具注入

**位置**: `src/agents/tools/memory-tool.ts`

**两个核心工具**

**1. `memory_search` - 语义搜索**
```typescript
{
  name: "memory_search",
  description: "语义搜索 MEMORY.md + memory/*.md (及可选的会话转录)；返回带路径+行号的片段",
  parameters: {
    query: string;         // 搜索查询
    maxResults?: number;   // 最大结果数（默认 6）
    minScore?: number;     // 最小分数阈值（默认 0.35）
  },
  returns: {
    results: MemorySearchResult[];
    provider: string;      // 嵌入提供商
    model: string;         // 模型名称
    fallback?: string;     // 是否降级
  }
}
```

**2. `memory_get` - 精确读取**
```typescript
{
  name: "memory_get",
  description: "读取特定记忆文件内容（可选行范围）",
  parameters: {
    path: string;          // 文件相对路径
    from?: number;         // 起始行号（可选）
    lines?: number;        // 读取行数（可选）
  },
  returns: {
    path: string;
    text: string;          // 文件内容
    from?: number;
    lines?: number;
  }
}
```

**工具可用性条件**
```typescript
// 位置: src/agents/tools/memory-tool.ts

function createMemorySearchTool(options): AnyAgentTool | null {
  // 1. 配置存在
  if (!options.config) return null;

  // 2. memorySearch.enabled 为 true
  const config = resolveMemorySearchConfig(cfg, agentId);
  if (!config) return null;

  // 3. 返回工具定义
  return { name: "memory_search", ... };
}
```

### 6.2 Agent 运行时集成

**位置**: `src/agents/pi-embedded.ts`, `src/auto-reply/reply/`

**工具集构建**
```typescript
// Agent 启动时
const tools = [
  ...defaultTools,
  createMemorySearchTool({ config, agentSessionKey }),
  createMemoryGetTool({ config, agentSessionKey }),
].filter(Boolean);

// 注入到 Agent 上下文
agent.setTools(tools);
```

**Agent 隔离机制**
```typescript
// 每个 agent 独立的索引路径
const indexPath = resolveMemoryIndexPath({
  agentId: "agent-123",
  basePath: "~/.openclaw/memory/{agentId}.sqlite"
});

// 每个 agent 独立的工作空间
const workspace = resolveAgentWorkspaceDir({
  agentId: "agent-123",
  basePath: "~/.openclaw/workspace"
});
```

### 6.3 自动记忆刷新 (Memory Flush)

**位置**: `src/auto-reply/reply/memory-flush.ts`

**触发机制**: 预压缩 Ping (Pre-compaction Ping)

**工作原理**
```
上下文接近限制
      ↓
检查是否需要刷新
      ↓
是否已在本轮刷新？ → 是 → 跳过
      ↓ 否
工作空间可写？ → 否 → 跳过
      ↓ 是
触发静默 Agent 回合
      ↓
系统提示: "预压缩记忆刷新。将持久记忆写入磁盘。"
用户提示: "立即存储持久记忆（使用 memory/YYYY-MM-DD.md）；如无需存储，回复 NO_REPLY。"
      ↓
Agent 执行（写入文件或返回 NO_REPLY）
      ↓
标记 memoryFlushCompactionCount
      ↓
继续正常压缩流程
```

**触发条件检查**
```typescript
// 位置: src/auto-reply/reply/memory-flush.ts

function shouldRunMemoryFlush(params: {
  entry?: SessionEntry;
  contextWindowTokens: number;
  reserveTokensFloor: number;
  softThresholdTokens: number;
}): boolean {
  // 1. 会话有 token 统计
  if (!params.entry?.totalTokens) return false;

  // 2. 计算阈值
  const threshold =
    contextWindowTokens
    - reserveTokensFloor
    - softThresholdTokens;

  // 示例: 200,000 - 20,000 - 4,000 = 176,000

  // 3. 超过阈值？
  if (params.entry.totalTokens < threshold) return false;

  // 4. 本轮压缩周期已刷新？
  const lastFlush = params.entry.memoryFlushCompactionCount;
  const currentCycle = params.entry.compactionCount;
  if (lastFlush === currentCycle) return false;

  // 5. 触发刷新
  return true;
}
```

**配置**
```json5
{
  "agents": {
    "defaults": {
      "compaction": {
        "reserveTokensFloor": 20000,  // 底部保留空间
        "memoryFlush": {
          "enabled": true,             // 启用刷新
          "softThresholdTokens": 4000, // 提前触发阈值
          "prompt": "将持久笔记写入 memory/YYYY-MM-DD.md；如无需存储，回复 NO_REPLY。",
          "systemPrompt": "会话接近压缩。立即捕获持久记忆到磁盘。"
        }
      }
    }
  }
}
```

**默认值**
```typescript
// src/auto-reply/reply/memory-flush.ts

const DEFAULT_MEMORY_FLUSH_SOFT_TOKENS = 4000;
const SILENT_REPLY_TOKEN = "NO_REPLY";

const DEFAULT_MEMORY_FLUSH_PROMPT = [
  "Pre-compaction memory flush.",
  "Store durable memories now (use memory/YYYY-MM-DD.md; create memory/ if needed).",
  `If nothing to store, reply with ${SILENT_REPLY_TOKEN}.`,
].join(" ");

const DEFAULT_MEMORY_FLUSH_SYSTEM_PROMPT = [
  "Pre-compaction memory flush turn.",
  "The session is near auto-compaction; capture durable memories to disk.",
  `You may reply, but usually ${SILENT_REPLY_TOKEN} is correct.`,
].join(" ");
```

**会话元数据追踪**
```typescript
// sessions.json
{
  "compactionCount": 2,              // 压缩次数
  "memoryFlushAt": 1,                // 首次刷新时的压缩计数
  "memoryFlushCompactionCount": 2    // 最近一次刷新的压缩计数
}

// 逻辑
if (memoryFlushCompactionCount === compactionCount) {
  // 本轮已刷新，跳过
}
```

**静默特性**
- 提示中包含 `NO_REPLY` token
- Agent 通常回复 `NO_REPLY`
- 用户不可见此回合
- 仅在需要写入时执行文件操作

---

## 7. 向量数据库与嵌入

### 7.1 嵌入提供商架构

**位置**: `src/memory/embeddings.ts`

**提供商抽象**
```typescript
type EmbeddingProvider = {
  provider: "openai" | "gemini" | "local";
  model: string;
  providerKey: string;  // 端点+配置指纹

  // 单次嵌入
  embed(text: string): Promise<number[]>;

  // 批量嵌入（可选）
  embedBatch?(texts: string[]): Promise<number[][]>;
};
```

### 7.2 OpenAI 嵌入

**位置**: `src/memory/embeddings-openai.ts`

**默认模型**: `text-embedding-3-small`

**特点**
- **维度**: 1536
- **速度**: 快
- **成本**: $0.02 / 1M tokens
- **Batch API**: 50% 折扣（$0.01 / 1M tokens）

**配置示例**
```json5
{
  "memorySearch": {
    "provider": "openai",
    "model": "text-embedding-3-small",
    "remote": {
      "apiKey": "sk-...",
      "baseUrl": "https://api.openai.com/v1/",  // 可选
      "headers": {                                // 可选
        "X-Organization": "org-..."
      }
    }
  }
}
```

**自定义端点支持**
- OpenRouter: `https://openrouter.ai/api/v1/`
- vLLM: `http://localhost:8000/v1/`
- 任何 OpenAI 兼容端点

**批量处理**

位置: `src/memory/batch-openai.ts`

```typescript
// Batch API 流程
1. 创建批任务
   POST /v1/embeddings/batches
   Body: { input: ["text1", "text2", ...], model: "..." }

2. 轮询状态
   GET /v1/embeddings/batches/{batch_id}
   Status: validating → processing → completed

3. 下载结果
   GET /v1/embeddings/batches/{batch_id}/output
   返回: JSONL 格式的嵌入结果

4. 解析和存储
   提取每个 embedding 向量
   存储到 chunks 和 embedding_cache
```

**配置**
```json5
{
  "memorySearch": {
    "remote": {
      "batch": {
        "enabled": true,           // 启用批量
        "wait": true,              // 等待完成
        "concurrency": 2,          // 并发批任务数
        "pollIntervalMs": 5000,    // 轮询间隔
        "timeoutMinutes": 30       // 超时时间
      }
    }
  }
}
```

### 7.3 Gemini 嵌入

**位置**: `src/memory/embeddings-gemini.ts`

**默认模型**: `gemini-embedding-001`

**特点**
- **维度**: 768
- **速度**: 快
- **成本**: 免费（有配额限制）
- **Batch API**: 支持异步批量

**配置示例**
```json5
{
  "memorySearch": {
    "provider": "gemini",
    "model": "gemini-embedding-001",
    "remote": {
      "apiKey": "AIza...",
      "baseUrl": "https://generativelanguage.googleapis.com"  // 可选
    }
  }
}
```

**批量处理**

位置: `src/memory/batch-gemini.ts`

- 使用 Gemini Embeddings Batch API
- 异步提交和轮询
- 类似 OpenAI Batch 流程

### 7.4 本地嵌入

**位置**: `src/memory/embeddings-local.ts`

**使用 node-llama-cpp**

**默认模型**:
```
hf:ggml-org/embeddinggemma-300M-GGUF/embeddinggemma-300M-Q8_0.gguf
```
- 大小: ~600 MB
- 维度: 256
- 量化: Q8_0（8-bit）

**特点**
- **离线**: 无需 API 密钥
- **隐私**: 数据不离开本地
- **成本**: 零（除硬件）
- **速度**: 取决于硬件（CPU/GPU）

**配置示例**
```json5
{
  "memorySearch": {
    "provider": "local",
    "local": {
      "modelPath": "hf:ggml-org/embeddinggemma-300M-GGUF/embeddinggemma-300M-Q8_0.gguf",
      "modelCacheDir": "~/.openclaw/models",  // 可选
      "gpu": true                              // 启用 GPU（如果可用）
    }
  }
}
```

**自动下载**
- 首次使用时自动从 HuggingFace 下载
- 断点续传支持
- 下载到 `modelCacheDir` 或 `node-llama-cpp` 默认缓存

**原生构建要求**
```bash
# 1. 批准原生构建
pnpm approve-builds

# 2. 选择 node-llama-cpp

# 3. 重新构建
pnpm rebuild node-llama-cpp
```

### 7.5 自动选择与降级

**位置**: `src/memory/embeddings.ts` - `createEmbeddingProvider()`

**自动选择逻辑 (`provider: "auto"`)**
```typescript
async function createEmbeddingProvider(config): Promise<EmbeddingProvider> {
  // 1. 尝试本地（如果配置了 modelPath 且文件存在）
  if (config.local?.modelPath) {
    try {
      const exists = await fs.access(resolvedPath);
      return await createLocalProvider(config);
    } catch {}
  }

  // 2. 尝试 OpenAI（如果有 API key）
  const openaiKey = resolveOpenAiKey(config);
  if (openaiKey) {
    return createOpenAiProvider(config);
  }

  // 3. 尝试 Gemini（如果有 API key）
  const geminiKey = resolveGeminiKey(config);
  if (geminiKey) {
    return createGeminiProvider(config);
  }

  // 4. 失败
  throw new Error("No embedding provider available. Configure API keys or local model.");
}
```

**降级链 (Fallback Chain)**

配置:
```json5
{
  "memorySearch": {
    "provider": "local",
    "fallback": "openai"  // local 失败时降级到 openai
  }
}
```

支持的降级选项:
- `"openai"`: 降级到 OpenAI
- `"gemini"`: 降级到 Gemini
- `"local"`: 降级到本地
- `"none"`: 不降级（失败即失败）

**降级触发条件**
- 提供商初始化失败
- 嵌入 API 调用超时
- 批量处理失败次数超限

**降级记录**
```typescript
// MemoryIndexManager.status()
{
  provider: "openai",           // 当前使用的提供商
  model: "text-embedding-3-small",
  fallback: {
    from: "local",              // 从哪个提供商降级
    reason: "Model file not found: /path/to/model.gguf"
  }
}
```

### 7.6 SQLite-vec 向量加速

**位置**: `src/memory/sqlite-vec.ts`

**什么是 sqlite-vec？**
- SQLite 的 C 扩展
- 提供原生向量操作
- 支持余弦、欧几里得、点积距离
- 零依赖（扩展库内置）

**虚拟表结构**
```sql
CREATE VIRTUAL TABLE chunks_vec USING vec0(
  id TEXT PRIMARY KEY,
  embedding FLOAT[1536]  -- 维度动态确定
);
```

**插入向量**
```typescript
// TypeScript
const vectorBlob = Buffer.from(new Float32Array(embedding).buffer);

db.prepare(`
  INSERT INTO chunks_vec (id, embedding)
  VALUES (?, ?)
`).run(chunkId, vectorBlob);
```

**查询相似向量**
```sql
SELECT
  id,
  vec_distance_cosine(embedding, ?) AS distance
FROM chunks_vec
WHERE distance IS NOT NULL
ORDER BY distance ASC
LIMIT 24;
```

**优势**
- **性能**: 原生 C 实现，比 JS 快 10-100x
- **内存**: 不需要加载所有向量到 JS 内存
- **可靠**: 基于 SQLite，事务安全

**降级到 JS**

当 sqlite-vec 不可用时:
```typescript
// 从 chunks 表读取所有 embedding JSON
const allChunks = db.prepare(`
  SELECT id, path, start_line, end_line, source, text, embedding
  FROM chunks
`).all();

// 在 JS 中计算余弦相似度
const results = allChunks
  .map(chunk => ({
    ...chunk,
    embedding: JSON.parse(chunk.embedding),
  }))
  .map(chunk => ({
    ...chunk,
    score: cosineSimilarity(queryEmbedding, chunk.embedding),
  }))
  .sort((a, b) => b.score - a.score)
  .slice(0, maxResults);
```

**扩展加载**
```typescript
// 位置: src/memory/sqlite-vec.ts

function loadSqliteVecExtension(db: DatabaseSync, config): boolean {
  const extensionPath =
    config.store.vector.extensionPath ||
    path.join(__dirname, "../../native/sqlite-vec/vec0.dylib");  // macOS
    // 或 vec0.so (Linux), vec0.dll (Windows)

  try {
    db.loadExtension(extensionPath);
    return true;
  } catch (err) {
    log.warn("sqlite-vec 扩展加载失败，降级到 JS 计算", err);
    return false;
  }
}
```

**动态维度处理**
```typescript
// 首次嵌入时确定维度
const firstEmbedding = await provider.embed("test");
const dimensions = firstEmbedding.length;

// 创建/重建虚拟表
db.exec(`DROP TABLE IF EXISTS chunks_vec`);
db.exec(`
  CREATE VIRTUAL TABLE chunks_vec USING vec0(
    id TEXT PRIMARY KEY,
    embedding FLOAT[${dimensions}]
  )
`);

// 存储维度到 meta 表
setMeta(db, "memory_index_meta_v1", {
  ...meta,
  vectorDims: dimensions
});
```

**维度变更检测**
```typescript
// 如果模型变更导致维度不同
const storedMeta = getMeta(db, "memory_index_meta_v1");
if (storedMeta.vectorDims !== currentDimensions) {
  // 触发完全重索引
  log.info("向量维度变更，重建索引", {
    old: storedMeta.vectorDims,
    new: currentDimensions
  });
  await rebuildIndex();
}
```

### 7.7 嵌入缓存机制

**位置**: `src/memory/manager.ts` - 缓存逻辑

**缓存键设计**
```typescript
type EmbeddingCacheKey = {
  provider: "openai" | "gemini" | "local";
  model: string;
  provider_key: string;  // 端点+配置哈希
  hash: string;          // 文本内容的 SHA-256
};
```

**为什么需要 `provider_key`？**
- 同一模型在不同端点可能产生不同嵌入
- 示例: OpenAI 官方 vs OpenRouter 代理
- 确保缓存不跨端点污染

**provider_key 生成**
```typescript
function computeProviderKey(provider: EmbeddingProvider): string {
  const parts = [
    provider.provider,
    provider.model,
    provider.baseUrl || "",
    JSON.stringify(provider.headers || {}),
  ];
  return crypto.createHash("sha256")
    .update(parts.join(":"))
    .digest("hex")
    .slice(0, 16);  // 取前 16 字符
}
```

**缓存查询**
```typescript
// 嵌入前检查缓存
function getCachedEmbedding(
  db: DatabaseSync,
  provider: string,
  model: string,
  providerKey: string,
  textHash: string
): number[] | null {
  const row = db.prepare(`
    SELECT embedding FROM embedding_cache
    WHERE provider = ? AND model = ?
      AND provider_key = ? AND hash = ?
  `).get(provider, model, providerKey, textHash);

  return row ? JSON.parse(row.embedding) : null;
}
```

**缓存存储**
```typescript
function cacheEmbedding(
  db: DatabaseSync,
  provider: string,
  model: string,
  providerKey: string,
  textHash: string,
  embedding: number[]
): void {
  db.prepare(`
    INSERT INTO embedding_cache
      (provider, model, provider_key, hash, embedding, dims, updated_at)
    VALUES (?, ?, ?, ?, ?, ?, ?)
    ON CONFLICT(provider, model, provider_key, hash)
    DO UPDATE SET
      embedding = excluded.embedding,
      dims = excluded.dims,
      updated_at = excluded.updated_at
  `).run(
    provider,
    model,
    providerKey,
    textHash,
    JSON.stringify(embedding),
    embedding.length,
    Date.now()
  );
}
```

**LRU 淘汰**
```typescript
// 配置
const MAX_CACHE_ENTRIES = 50000;  // 默认无限制

// 淘汰旧条目
function evictOldCacheEntries(db: DatabaseSync, maxEntries: number): void {
  const count = db.prepare(`
    SELECT COUNT(*) as count FROM embedding_cache
  `).get().count;

  if (count > maxEntries) {
    const toDelete = count - maxEntries;
    db.exec(`
      DELETE FROM embedding_cache
      WHERE rowid IN (
        SELECT rowid FROM embedding_cache
        ORDER BY updated_at ASC
        LIMIT ${toDelete}
      )
    `);
  }
}
```

**缓存命中率统计**
```typescript
// 索引统计
type IndexStats = {
  totalChunks: number;
  cacheHits: number;
  cacheMisses: number;
  hitRate: number;  // cacheHits / (cacheHits + cacheMisses)
};

// 日志示例
// [memory] 索引完成: 150 块, 缓存命中 120 (80%), API 调用 30
```

---

## 8. 记忆压缩与总结

### 8.1 会话压缩 (Compaction)

**概念**: 当会话上下文接近 token 限制时，自动总结旧对话以释放空间。

**位置**: `docs/concepts/compaction.md`（推断）

**压缩流程**
```
检测: totalTokens > (contextWindow - reserveTokens)
      ↓
触发: 自动或手动 (/compact)
      ↓
预刷新: Memory Flush (可选，见 6.3)
      ↓
总结: LLM 总结旧消息
      ↓
替换: 用总结替换原始消息
      ↓
更新: compactionCount++
      ↓
继续: 恢复正常对话
```

**压缩策略**
- **保留最近消息**: 最新 N 轮完整保留
- **总结中间历史**: 旧消息被压缩成摘要
- **保留系统消息**: 重要的系统提示保持原样

**配置**
```json5
{
  "agents": {
    "defaults": {
      "compaction": {
        "enabled": true,
        "reserveTokensFloor": 20000,    // 底部保留空间
        "targetCompressionRatio": 0.5,  // 目标压缩率
        "preserveRecentTurns": 10       // 保留最近 N 轮
      }
    }
  }
}
```

### 8.2 文本分块即隐式压缩

**原理**
- 400 token 的块 << 完整文件
- 重叠保证关键信息不丢失
- 搜索只返回相关块，而非全文

**示例**
```
原始文件: 10,000 tokens
分块后: 25 个块 × 400 tokens = 10,000 tokens (存储)
搜索返回: 6 个块 × 400 tokens = 2,400 tokens (加载到上下文)

压缩率: 76% (只加载 24% 的内容)
```

### 8.3 Snippet 截断

**位置**: `src/memory/manager.ts`

```typescript
const SNIPPET_MAX_CHARS = 700;

function truncateSnippet(text: string): string {
  if (text.length <= SNIPPET_MAX_CHARS) {
    return text;
  }
  return text.slice(0, SNIPPET_MAX_CHARS) + "...";
}
```

**为什么 700 字符？**
- ~175 tokens (假设 4 chars/token)
- 足够提供上下文
- 避免污染主上下文窗口
- 用户可通过 `memory_get` 获取完整内容

### 8.4 无显式记忆总结

**当前状态**: OpenClaw 不实现自动记忆总结

**原因**
1. **文件即真相**: 记忆由用户管理
2. **搜索即压缩**: 只检索相关部分
3. **手动精炼**: 用户可将 `memory/YYYY-MM-DD.md` 内容提炼到 `MEMORY.md`

**未来可能方向**
- 周期性总结每日日志
- 自动提取关键事实到 MEMORY.md
- 记忆去重和合并

---

## 9. 关键实现文件

### 核心模块

| 文件路径 | 行数 | 功能描述 |
|---------|------|----------|
| `src/memory/manager.ts` | 2399 | 核心 `MemoryIndexManager` 类，协调所有记忆操作 |
| `src/memory/embeddings.ts` | ~500 | 嵌入提供商抽象和自动选择 |
| `src/memory/hybrid.ts` | 116 | 混合搜索融合逻辑 |
| `src/memory/internal.ts` | ~400 | 分块、文件发现、工具函数 |
| `src/memory/memory-schema.ts` | 97 | SQLite 表结构定义 |
| `src/memory/manager-search.ts` | ~600 | 向量和关键词搜索实现 |
| `src/memory/sqlite-vec.ts` | ~300 | SQLite-vec 扩展加载和向量操作 |
| `src/memory/batch-openai.ts` | ~600 | OpenAI Batch API 批量嵌入 |
| `src/memory/batch-gemini.ts` | ~500 | Gemini Batch API 批量嵌入 |

### 嵌入提供商

| 文件路径 | 功能 |
|---------|------|
| `src/memory/embeddings-openai.ts` | OpenAI embeddings API 实现 |
| `src/memory/embeddings-gemini.ts` | Gemini embeddings API 实现 |
| `src/memory/embeddings-local.ts` | node-llama-cpp 本地嵌入 |

### Agent 集成

| 文件路径 | 功能 |
|---------|------|
| `src/agents/tools/memory-tool.ts` | `memory_search` 和 `memory_get` 工具定义 |
| `src/agents/memory-search.ts` | 记忆搜索配置解析 |
| `src/auto-reply/reply/memory-flush.ts` | 预压缩记忆刷新逻辑 |
| `src/auto-reply/reply/agent-runner-memory.ts` | Agent 运行时记忆刷新集成 |

### 会话管理

| 文件路径 | 功能 |
|---------|------|
| `src/config/sessions.ts` | 会话元数据管理 |
| `src/config/sessions/paths.ts` | 会话路径解析 |
| `src/sessions/transcript-events.ts` | 会话转录事件监听 |

### 文档

| 文件路径 | 内容 |
|---------|------|
| `docs/concepts/memory.md` | 用户面向文档 |
| `docs/concepts/compaction.md` | 压缩机制文档（推断）|

---

## 10. 配置选项

### 完整配置示例

```json5
{
  "agents": {
    "defaults": {
      "workspace": "~/.openclaw/workspace",  // 工作空间路径

      "memorySearch": {
        // 基础开关
        "enabled": true,

        // 嵌入提供商
        "provider": "auto",  // "auto" | "openai" | "gemini" | "local"
        "model": "text-embedding-3-small",
        "fallback": "openai",  // "openai" | "gemini" | "local" | "none"

        // 远程配置 (OpenAI/Gemini)
        "remote": {
          "apiKey": "sk-...",
          "baseUrl": "https://api.openai.com/v1/",
          "headers": {
            "X-Organization": "org-..."
          },
          "batch": {
            "enabled": true,
            "wait": true,
            "concurrency": 2,
            "pollIntervalMs": 5000,
            "timeoutMinutes": 30
          }
        },

        // 本地配置
        "local": {
          "modelPath": "hf:ggml-org/embeddinggemma-300M-GGUF/embeddinggemma-300M-Q8_0.gguf",
          "modelCacheDir": "~/.openclaw/models",
          "gpu": true
        },

        // 分块配置
        "chunking": {
          "tokens": 400,
          "overlap": 80
        },

        // 搜索配置
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

        // 存储配置
        "store": {
          "path": "~/.openclaw/memory/{agentId}.sqlite",
          "vector": {
            "enabled": true,
            "extensionPath": "/path/to/sqlite-vec/vec0.dylib"
          },
          "fts": {
            "enabled": true
          }
        },

        // 缓存配置
        "cache": {
          "enabled": true,
          "maxEntries": 50000
        },

        // 同步配置
        "sync": {
          "watch": true,
          "debounceMs": 1500,
          "onSessionStart": true,
          "onSearch": true,
          "intervalMinutes": 0,  // 0 = 禁用定时同步
          "sessions": {
            "deltaBytes": 100000,
            "deltaMessages": 50
          }
        },

        // 额外路径
        "extraPaths": [
          "../team-docs",
          "/srv/shared-notes/overview.md"
        ],

        // 记忆来源
        "sources": ["memory"],  // ["memory", "sessions"]

        // 实验性功能
        "experimental": {
          "sessionMemory": false  // 启用会话转录索引
        }
      },

      // 压缩配置
      "compaction": {
        "enabled": true,
        "reserveTokensFloor": 20000,
        "targetCompressionRatio": 0.5,
        "preserveRecentTurns": 10,

        // 记忆刷新
        "memoryFlush": {
          "enabled": true,
          "softThresholdTokens": 4000,
          "prompt": "将持久笔记写入 memory/YYYY-MM-DD.md；如无需存储，回复 NO_REPLY。",
          "systemPrompt": "会话接近压缩。立即捕获持久记忆到磁盘。"
        }
      }
    }
  }
}
```

### 关键配置项说明

**Provider 选择**
- `"auto"`: 自动选择（本地 → OpenAI → Gemini）
- `"openai"`: 强制 OpenAI
- `"gemini"`: 强制 Gemini
- `"local"`: 强制本地

**Fallback 策略**
- `"openai"`: 主提供商失败时降级到 OpenAI
- `"gemini"`: 降级到 Gemini
- `"local"`: 降级到本地
- `"none"`: 不降级

**Hybrid Search 权重**
- `vectorWeight + textWeight` 会自动归一化到 1.0
- 推荐: 70% vector, 30% text
- 纯向量: `vectorWeight: 1.0, textWeight: 0.0`
- 纯关键词: `vectorWeight: 0.0, textWeight: 1.0`

**Sync 触发时机**
- `onSessionStart`: 会话启动时同步
- `onSearch`: 搜索时检查并同步（如果脏）
- `intervalMinutes`: 定时同步（0 = 禁用）
- `watch`: 文件监控（debounce）
- `sessions.delta*`: 会话增量同步阈值

---

## 11. 最佳实践

### 11.1 记忆文件组织

**MEMORY.md**
```markdown
# 用户偏好
- 编码风格: TypeScript strict, functional
- 文档: 中文 + 英文
- 工具: VSCode, pnpm, Bun

# 项目规范
- 使用 ESM 模块
- 测试: Vitest
- Lint: Oxlint

# 关键决策
## 2024-01-15: 选择 SQLite 作为记忆存储
- 原因: 零配置，跨平台，ACID
- 备选: PostgreSQL (过重), JSON 文件 (无索引)
```

**memory/2024-01-15.md**
```markdown
# 2024-01-15

## 10:00 - 实现混合搜索
- 添加 BM25 关键词搜索
- 融合策略: 70% 向量 + 30% 文本
- 测试: 精确 ID 查询性能提升 80%

## 14:30 - Bug 修复: 嵌入缓存
- 问题: 跨端点缓存污染
- 解决: 添加 provider_key 到缓存键
- PR: #456

## 待办
- [ ] 实现会话记忆索引
- [ ] 优化批量嵌入超时处理
```

### 11.2 提示 Agent 使用记忆

**显式提醒**
```
User: 我之前让你记住什么？
Assistant: [调用 memory_search("previous user instructions")]
```

**上下文切换**
```
User: 继续昨天的工作
Assistant: [调用 memory_search("yesterday work context")]
```

**偏好查询**
```
User: 帮我写个函数
Assistant: [调用 memory_search("coding style preferences")]
[使用记忆中的风格规范编写代码]
```

### 11.3 性能优化技巧

**1. 使用 Batch API**
```json5
{
  "memorySearch": {
    "remote": {
      "batch": {
        "enabled": true,  // 大幅降低成本
        "wait": true      // 等待完成（推荐）
      }
    }
  }
}
```

**2. 启用缓存**
```json5
{
  "memorySearch": {
    "cache": {
      "enabled": true,
      "maxEntries": 50000  // 根据内存调整
    }
  }
}
```

**3. 调整分块大小**
```json5
{
  "memorySearch": {
    "chunking": {
      "tokens": 400,   // 默认，平衡精度和召回
      "overlap": 80    // 20% 重叠
    }
  }
}

// 大文件优化
{
  "chunking": {
    "tokens": 600,   // 更大块，减少 API 调用
    "overlap": 100
  }
}

// 精细检索优化
{
  "chunking": {
    "tokens": 200,   // 更小块，更精确
    "overlap": 50
  }
}
```

**4. 选择合适的提供商**
```
大规模索引 (>1000 文档):
  → OpenAI Batch API (最快，最便宜)

隐私敏感:
  → Local (node-llama-cpp)

小规模 + 免费额度:
  → Gemini
```

### 11.4 调试和监控

**启用详细日志**
```bash
# 设置环境变量
export OPENCLAW_LOG_LEVEL=debug

# 运行 CLI
openclaw gateway run
```

**查看索引状态**
```bash
# 使用 SQLite 命令行
sqlite3 ~/.openclaw/memory/<agentId>.sqlite

# 查询统计
SELECT
  COUNT(*) as total_chunks,
  COUNT(DISTINCT path) as total_files,
  SUM(LENGTH(text)) as total_chars
FROM chunks;

# 查看缓存命中率
SELECT COUNT(*) as cached_embeddings FROM embedding_cache;
```

**监控嵌入 API 使用**
```typescript
// 代码中添加日志
log.info("Embedding API call", {
  provider: "openai",
  model: "text-embedding-3-small",
  texts: batch.length,
  tokens: estimatedTokens
});
```

### 11.5 常见问题排查

**问题 1: 记忆搜索返回空结果**

排查步骤:
```bash
1. 检查索引是否存在
   ls -lh ~/.openclaw/memory/<agentId>.sqlite

2. 查看数据库内容
   sqlite3 ~/.openclaw/memory/<agentId>.sqlite "SELECT COUNT(*) FROM chunks;"

3. 检查配置
   openclaw config get agents.defaults.memorySearch.enabled

4. 手动触发同步
   [在会话中] "请同步记忆索引"
```

**问题 2: sqlite-vec 不可用**

解决方法:
```bash
# 1. 检查扩展是否存在
ls -lh /path/to/sqlite-vec/vec0.dylib  # macOS
ls -lh /path/to/sqlite-vec/vec0.so     # Linux

# 2. 手动指定路径
openclaw config set agents.defaults.memorySearch.store.vector.extensionPath "/path/to/vec0.dylib"

# 3. 或禁用 sqlite-vec（使用 JS 降级）
openclaw config set agents.defaults.memorySearch.store.vector.enabled false
```

**问题 3: 本地嵌入失败**

排查步骤:
```bash
# 1. 检查原生构建
pnpm approve-builds
pnpm rebuild node-llama-cpp

# 2. 验证模型文件
ls -lh ~/.openclaw/models/

# 3. 测试加载
node -e "const { getLlama } = require('node-llama-cpp'); getLlama().then(console.log)"

# 4. 降级到远程
openclaw config set agents.defaults.memorySearch.fallback "openai"
```

**问题 4: 批量嵌入超时**

解决方法:
```json5
{
  "memorySearch": {
    "remote": {
      "batch": {
        "timeoutMinutes": 60,  // 增加超时
        "concurrency": 1       // 降低并发
      }
    }
  }
}

// 或禁用批量模式
{
  "memorySearch": {
    "remote": {
      "batch": {
        "enabled": false
      }
    }
  }
}
```

---

## 附录: 技术术语表

| 术语 | 英文 | 解释 |
|------|------|------|
| 嵌入 | Embedding | 文本的向量表示，捕获语义信息 |
| 向量 | Vector | 高维数值数组，表示语义空间中的点 |
| 余弦相似度 | Cosine Similarity | 衡量两个向量夹角的相似度指标 |
| BM25 | Best Matching 25 | 概率检索模型，基于 TF-IDF 的改进 |
| FTS5 | Full-Text Search 5 | SQLite 的全文搜索扩展 |
| JSONL | JSON Lines | 每行一个 JSON 对象的文件格式 |
| 分块 | Chunking | 将大文本分割成小块以便处理 |
| 重叠 | Overlap | 相邻块之间共享的内容，避免边界信息丢失 |
| 降级 | Fallback | 主方法失败时切换到备用方法 |
| 压缩 | Compaction | 总结旧上下文以释放空间 |
| LRU | Least Recently Used | 最近最少使用缓存淘汰算法 |
| Delta | Delta | 增量，自上次同步以来的变化量 |

---

## 总结

OpenClaw 的 agent 记忆系统是一个**生产级、深思熟虑的混合架构**，具备以下亮点：

1. **文件为本**: Markdown 文件作为真相源，无专有格式
2. **智能索引**: 增量同步、哈希去重、批量嵌入
3. **混合检索**: 语义向量 + 关键词搜索，优势互补
4. **多层记忆**: 短期/长期/情节/语义记忆完整实现
5. **性能优化**: SQLite-vec 加速、嵌入缓存、批量 API
6. **Agent 友好**: 自动记忆刷新、工具集成、隔离管理
7. **灵活配置**: 本地/远程提供商、降级链、可调参数
8. **成本意识**: Batch API 折扣、缓存去重、delta 索引

此系统展示了如何在实际产品中**平衡性能、成本、隐私和用户体验**，是 LLM 应用中记忆管理的优秀参考实现。

---

**文档版本**: v1.0
**最后更新**: 2024-01-16
**维护者**: OpenClaw Team
