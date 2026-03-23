# Hybrid Search & Retrieval - D3.4 QueryProcessor

> 学习日期: 2026-03-22
> 综合评分: 8.5/10 | 追问轮数: 1/1

---

**Q1: `QueryProcessor.process()` 方法将用户的原始查询转化为结构化的 `ProcessedQuery`。请描述这个方法的处理流程，并重点解释为什么 QueryProcessor 中的分词器（jieba）必须和 Ingestion 阶段 `SparseEncoder` 中使用的分词器保持一致？如果两端不一致会导致什么问题？**

**标准答案**: `QueryProcessor.process()` 的四步处理流程：

1. **规范化**（`_normalize`）：合并多余空格、去首尾空白，统一输入格式
2. **提取过滤语法**（`_extract_filters`）：用正则 `(\w+):([^\s]+)` 匹配 `key:value` 语法（如 `collection:docs`），返回 filters 字典和去除过滤语法后的纯查询文本
3. **分词**（`_tokenize`）：使用 `jieba.lcut()` 进行中文分词（同时保留英文原样），过滤掉纯标点和空白 token
4. **关键词过滤**（`_filter_keywords`）：去停用词（双语：`CHINESE_STOPWORDS | ENGLISH_STOPWORDS`）→ 大小写不敏感去重（保留原始大小写）→ 最小长度过滤 → 截断至 `max_keywords`（默认 20）

最终返回 `ProcessedQuery(original_query, keywords, filters)`。

**分词器一致性的重要性**：BM25 是基于精确词项匹配的。倒排索引的 key 是 ingestion 阶段 SparseEncoder 分词产生的 token，查询时的 token 必须和索引端的 token 完全一致才能命中。

具体例子：如果 SparseEncoder 用 jieba 把"机器学习"分成 `["机器", "学习"]`，但 QueryProcessor 用 simple split 保留为整体 `"机器学习"`，那么查询 token `"机器学习"` 不匹配索引中的 `"机器"` 和 `"学习"`，BM25 完全无法命中该文档。这就是 IR 领域的基本原则：**Query-Index Tokenizer Alignment**。

---

**Q2: `QueryProcessor` 支持从查询字符串中解析过滤语法。请描述支持哪些 key 及别名，解析结果是如何传递给下游 Dense 和 Sparse 检索的？两条路径对 filters 的使用方式有什么不同？**

**标准答案**:

**支持的 filter key 及别名**（`query_processor.py:186-202`）：

| 标准 key | 别名 | 存入 filters 的 key |
|----------|------|-------------------|
| collection | col, c | collection |
| type | doc_type, t | doc_type |
| source | src, s | source_path |
| tag | tags | tags（列表，支持逗号分隔累加） |
| 其他任意 key | 无 | 原样保留 |

**filters 的下游传递**：

在 `HybridSearch.search()` 中，QueryProcessor 提取的 filters 与调用方显式传入的 filters 通过 `_merge_filters` 合并（显式 filters 优先覆盖），然后传给 `_run_retrievals`。

**两条路径对 filters 的使用方式完全不同**：

- **Dense 路径**：`filters` 直接传给 `dense_retriever.retrieve(filters=merged_filters)`，最终传给 `vector_store.query(filters=...)`——这是向量数据库层面的元数据过滤，在向量相似度搜索时同时施加 metadata 条件。所有 filter key 都可以生效。

- **Sparse 路径**：不传通用 filters，而是从 filters 中**仅提取 `collection` 字段**（`hybrid_search.py:558`：`collection = filters.get('collection')`），作为 BM25 索引文件的选择器。BM25 索引按 collection 分文件存储（`{collection}_bm25.json`），不支持通用的 metadata 过滤。因此 doc_type、source_path 等 filter key 在 Sparse 路径中**不生效**。

本质上：Dense 端的 filters 是**查询时的动态过滤条件**，Sparse 端的 collection 是**索引选择器**——两者机制完全不同。

补充：为了弥补 Sparse 路径无法做通用 metadata 过滤的短板，HybridSearch 在 fusion 之后还有一层 `_apply_metadata_filters`（由 `config.metadata_filter_post=True` 控制），对融合后的所有结果应用 doc_type、source_path、tags 等过滤条件，作为兜底保障。

---

## 📊 评价报告

**知识域**: D3 Hybrid Search & Retrieval > **知识点**: D3.4 QueryProcessor — 查询预处理与过滤语法传递机制
**追问轮数**: 1/1

### ✅ 回答亮点
- 四步处理流程完整准确，正则模式和 jieba.lcut() 分词细节到位
- 分词器一致性原因解释精炼
- Filter key 别名映射表完整无遗漏
- 精准区分 Dense 路径（动态元数据过滤）和 Sparse 路径（索引选择器）的本质差异
- 主动指出"doc_type、source_path 在 Sparse 路径中不生效"

### ⚠️ 需要加强
- 分词不一致的具体后果可举例说明
- Post-fusion 过滤补偿机制（_apply_metadata_filters）未提及
- 双语停用词合并设计（DEFAULT_STOPWORDS = CHINESE | ENGLISH）未展开

### 📈 评分明细

| 维度 | 分数 | 说明 |
|------|------|------|
| 准确性 | 9/10 | 流程和 filter 传递路径均准确无误 |
| 深度 | 8.5/10 | Dense vs Sparse filter 区分出色，可补充 post-fusion 过滤兜底 |
| 代码关联 | 8/10 | 引用了具体行号（186-202、558） |
| 设计思维 | 9/10 | "索引选择器 vs 动态过滤条件"展现了深层架构理解 |

### 🏆 综合评分: 8.5/10

---

## 📚 学习指南

### 📂 相关代码
- `src/core/query_engine/query_processor.py:117-149` — process() 四步流程入口
- `src/core/query_engine/query_processor.py:168-208` — _extract_filters 过滤语法解析
- `src/core/query_engine/query_processor.py:210-237` — _tokenize jieba 分词
- `src/core/query_engine/query_processor.py:239-280` — _filter_keywords 停用词/去重/截断
- `src/core/query_engine/query_processor.py:26-74` — 双语停用词定义
- `src/core/query_engine/hybrid_search.py:677-747` — _apply_metadata_filters post-fusion 过滤

### 🔗 参考资料
- **jieba 中文分词** — 精确模式 vs 全模式，自定义词典
- **Query-Index Tokenizer Alignment** — IR 基本原则

### 💡 建议学习路径
1. 对比 query_processor.py 的 _tokenize 和 sparse_encoder.py 的分词逻辑
2. 手动测试 QueryProcessor 的 filter 提取和关键词输出
3. 阅读 _apply_metadata_filters 理解 post-fusion 过滤兜底
4. 思考扩展：支持日期范围过滤需要修改哪些地方
