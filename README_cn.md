# Context 工作原理 - 本地搜索和 Chunk 机制

用法

```sh
➜  context add https://github.com/uniquejava/icm-tracing/tree/heartbeat

➜  context list
Installed packages:

  ant-design@6.3.5           1.4 MB    567 sections
  icm-tracing@heartbeat     68.0 KB    18 sections
  next@15.0.0                3.1 MB    1174 sections
  react@latest               5.9 MB    1662 sections

Total: 4 packages (10.4 MB)

➜  context query 'icm-tracing@heartbeat'
error: missing required argument 'topic'
➜  context query 'icm-tracing@heartbeat'  'some topic'
{
  "library": "icm-tracing@heartbeat",
  "version": "heartbeat",
  "results": [],
  "message": "No documentation found. Try a shorter query using just the API or function name, for example 'cors' instead of 'CORS middleware configuration'."
}
➜   context query 'icm-tracing@heartbeat'  'HeartBeat'
{
  "library": "icm-tracing@heartbeat",
  "version": "heartbeat",
  "results": [
    {
      "title": "how-to-add-temporal-tracing > Goal",
      "content": "After these changes, the project should:\n\n- emit HTTP and application spans from Spring Boot,\n- emit Temporal workflow and activity spans,\n- export traces through OTLP,\n- optionally keep very long-running workflow traces visible with custom heartbeat spans.",
      "source": "docs/how-to-add-temporal-tracing.md"
    },
```

## 问题 1：它是怎么在本地搜索的？是精确匹配 topic 吗？

**答案：不是精确匹配！是使用 BM25 全文检索（Full-Text Search）！**

看 `search.ts:43-74`：
- 使用 **SQLite FTS5** 全文搜索
- **BM25 评分算法**：标题比内容更重要（权重 5.0, 10.0, 1.0）
- 关键词是 **隐含 AND 关系**：比如搜索 "temporal tracing"，会找同时包含 "temporal" 和 "tracing" 的内容
- 不是精确匹配！是模糊搜索！

### 搜索流程（`search.ts:183-196`）：
1. **searchFts()** → BM25 搜索，取 Top 20
2. **filterByRelevance()** → 只保留分数 > 最高分 50% 的结果
3. **applyTokenBudget()** → 总 tokens 不超过 2000
4. **assembleResults()** → 合并相邻的 chunk，按文档排序

---

## 问题 2：保存到 SQLite 时会做 chunk 吗？

**答案：会！按 Markdown 的 Section（章节）做 chunk！**

看 `build.ts` 和 `package-builder.ts`：

### Chunk 策略：

| 配置项 | 值 | 说明 |
|--------|-----|------|
| MAX_CHUNK_TOKENS | 800 | 理想 chunk 大小 |
| HARD_LIMIT_TOKENS | 1200 | 绝对最大，超过就强制分割 |
| MIN_CHUNK_TOKENS | 5 | 过滤掉太 trivial 的 chunk |

### Chunk 方法：

1. **按 Markdown 标题分割**：每个 `## 标题` 或 `### 子标题` 就是一个 chunk
2. **如果单个 chunk 太大**（> 1200 tokens）：
   - 优先在代码块（```）边界分割
   - 如果还是太大，按行分割
3. **过滤掉 TOC（目录）**：如果 50% 以上是链接，就跳过
4. **去重**：内容完全一样的 chunk（MD5 前 16 位相同）只留一个
5. **合并相邻 chunk**：搜索时如果相邻 chunk 都匹配，会合并在一起返回

### SQLite 表结构（`package-builder.ts:55-70`）：
```sql
CREATE TABLE chunks (
  id INTEGER PRIMARY KEY,
  doc_path TEXT NOT NULL,
  doc_title TEXT NOT NULL,
  section_title TEXT NOT NULL,
  content TEXT NOT NULL,
  tokens INTEGER NOT NULL,
  has_code INTEGER DEFAULT 0
);

-- 还创建了 FTS5 虚拟表用于全文搜索！
CREATE VIRTUAL TABLE chunks_fts USING fts5(
  doc_title, section_title, content,
  content='chunks', content_rowid='id',
  tokenize='porter unicode61'
);
```

---

## 总结

| 问题 | 答案 |
|------|------|
| 搜索方式？ | BM25 全文检索，不是精确匹配 |
| 会做 chunk 吗？ | 会！按 Markdown 章节，800 tokens 理想大小，最大 1200 tokens |

设计得相当不错！
