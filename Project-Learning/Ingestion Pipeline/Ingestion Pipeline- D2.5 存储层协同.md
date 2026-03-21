# Ingestion Pipeline - D2.5 存储层协同

> 学习日期: 2026-03-20
> 综合评分: 7.5/10 | 追问轮数: 2/3

---

**Q1: 在 Ingestion Pipeline 的 Stage 6 中，VectorUpserter、BM25Indexer、ImageStorage 三个存储组件各自的职责、数据来源，以及 Pipeline 如何协调写入顺序——特别是关键的 ID 对齐操作的作用和必要性？**

**标准答案**: Stage 6 包含三个存储组件，按 6a → ID 对齐 → 6b → 6c 的顺序执行：

1. **VectorUpserter**（6a）：将 chunks + dense embedding 向量写入 ChromaDB。数据来源是 Stage 3 DocumentChunker 产生的 chunks 和 Stage 5 DenseEncoder 产生的 dense_vectors。写入时通过 `_generate_chunk_id()` 为每个 chunk 生成确定性 ID（格式：`{source_hash}_{index:04d}_{content_hash}`），并返回所有 vector_ids。

2. **ID 对齐操作**（pipeline.py 第 443-444 行）：将 VectorUpserter 生成的 chunk_id 回写到 sparse_stats 的每条记录中（`stat["chunk_id"] = vid`）。这是因为 SparseEncoder 只做词频统计，不生成 chunk_id。此对齐操作是 Dense 和 Sparse 两条检索路径的桥梁——确保 BM25 倒排索引中的 posting 使用与 ChromaDB 相同的 ID 体系，使得 SparseRetriever 通过 BM25 命中某个 posting 后，能用该 chunk_id 到 ChromaDB 回查完整数据，从而支持 Reciprocal Rank Fusion 等分数融合策略。

3. **BM25Indexer**（6b）：使用对齐后的 sparse_stats 构建 BM25 倒排索引，存储每个 chunk 的词频(tf)、文档长度(doc_length)及全局 IDF 分数，支持关键词精确匹配检索。数据来源是 Stage 5 SparseEncoder 产生的 sparse_stats。

4. **ImageStorage**（6c）：在 SQLite 中注册图片元数据索引（调用 `register_image()` 而非 `save_image()`，不复制文件只建立指针）。数据来源是 document.metadata["images"]，图片文件已在 Stage 2 PdfLoader 阶段提取并保存到磁盘。

---

**Q2: 这三个存储组件选择了三种不同的持久化策略（ChromaDB / JSON / SQLite），各自的设计考量是什么？BM25Indexer 使用 `add_documents()` 而非 `build()` 的设计意图？**

**标准答案**:

**持久化策略选型**：
- **VectorUpserter → ChromaDB**：高维向量检索需要专属索引结构（如 HNSW），ChromaDB 提供 vector + metadata + text 的 All-in-One 存储，一条记录包含所有信息。
- **BM25Indexer → JSON 文件**：倒排索引本质是 `{term → [postings]}` 的嵌套字典，JSON 天然对应这种结构，无需额外序列化层。且 BM25 查询需要全局统计量（num_docs、avg_doc_length）和所有 posting 来计算分数，不存在"只查某一行"的场景，整体加载到内存最高效。持久化时使用原子写入策略（先写 `.tmp` 文件再 `rename`），防止写入中途崩溃导致索引损坏。
- **ImageStorage → SQLite**：图片索引需要按 collection、doc_hash、page_num 等多个维度灵活查询，SQLite 的 SQL + 索引天然支持多条件组合查询，且每条记录独立（与 BM25 全量加载形成对比）。初始化时启用 WAL 模式（`PRAGMA journal_mode=WAL`）支持并发读写。

**`add_documents()` vs `build()`**：Pipeline 逐文档处理，但 BM25 索引是跨文档共享的全局索引。直接调用 `build()` 会用当前文档的 term_stats 覆盖所有历史数据。`add_documents()` 解决三个问题：
1. **增量性**：加载现有索引，将新文档的 term_stats 合并到已有索引中
2. **幂等性**：通过 `doc_id` 参数先调用 `remove_document()` 删除该文档旧的 postings，再添加新的，同一文档重复 ingest 不产生重复
3. **IDF 一致性**：合并后内部调用 `build()` 重新计算所有 term 的 IDF，确保全局统计量反映完整语料库

---

**Q3: BM25Indexer 的 `_save()` 方法使用了什么写入策略？ImageStorage 的 WAL 模式作用？Stage 6 部分失败如何处理？**（未作答）

**标准答案**:
- **BM25Indexer `_save()` 原子写入**（bm25_indexer.py 第 535-548 行）：先将数据写入 `.tmp` 临时文件，写入成功后通过 `temp_path.replace(index_path)` 原子重命名。如果写入中途崩溃，原索引文件不受影响；如果 rename 失败，清理临时文件。
- **ImageStorage WAL 模式**（image_storage.py 第 109 行）：`PRAGMA journal_mode=WAL` 启用 Write-Ahead Logging，允许读写并发——多个线程可以同时读取图片索引，而写入不会阻塞读取。
- **Stage 6 部分失败处理**：整个 `run()` 方法在 try/except 中执行，任何存储步骤失败都会触发 `mark_failed()` 并返回失败的 PipelineResult。但已完成的写入不会回滚（例如 vector 已写入但 BM25 失败），这是有意的设计取舍——各存储的幂等性保证了重新运行 Pipeline 时可以安全覆盖。

---

## 📊 评价报告

**知识域**: Ingestion Pipeline > **知识点**: D2.5 存储层协同 — 三类存储的职责、协调与持久化策略
**追问轮数**: 2/3

### ✅ 回答亮点
- 三个存储组件的职责与数据来源描述清晰准确，`register_image`（只建指针不复制文件）的本质一句话点透
- ID 对齐操作的解释非常出色——准确定义为"Dense 与 Sparse 两条检索路径的桥梁"，并延伸到 RRF 融合层面
- 三种持久化策略的选型分析有理有据：ChromaDB 对应高维向量检索、JSON 对应全量加载的自包含字典结构、SQLite 对应多维度关系查询
- `add_documents()` vs `build()` 的分析完整覆盖了增量性、幂等性、IDF 一致性三个维度

### ⚠️ 需要加强
- PdfLoader 是 Stage 2 而非 Stage 1（Stage 1 是 File Integrity Check）
- 未提及 BM25Indexer `_save()` 的原子写入策略（先写 `.tmp` 文件再 `rename`）
- 未提及 ImageStorage 的 SQLite WAL 模式（`PRAGMA journal_mode=WAL`）
- 未分析 Stage 6 部分失败场景下 Pipeline 的处理策略

### 📈 评分明细

| 维度 | 分数 | 说明 |
|------|------|------|
| 准确性 | 8/10 | 核心概念准确，仅 Stage 编号有误 |
| 深度 | 7.5/10 | 选型分析和 add_documents 分析深入，原子写入/WAL/部分失败未涉及 |
| 代码关联 | 7/10 | 引用了具体方法名和数据结构，但缺少文件路径和行号级引用 |
| 设计思维 | 8/10 | JSON vs SQLite 的查询模式对比、ID 对齐的架构意义分析出色 |

### 🏆 综合评分: 7.5/10

---

## 📚 学习指南

### 📂 相关代码
- [vector_upserter.py:140-168](src/ingestion/storage/vector_upserter.py#L140-L168) — `_generate_chunk_id()` 确定性 ID 生成逻辑
- [bm25_indexer.py:311-362](src/ingestion/storage/bm25_indexer.py#L311-L362) — `add_documents()` 增量合并与幂等删除逻辑
- [bm25_indexer.py:518-548](src/ingestion/storage/bm25_indexer.py#L518-L548) — `_save()` 原子写入策略
- [image_storage.py:96-136](src/ingestion/storage/image_storage.py#L96-L136) — SQLite WAL 模式与 schema 初始化
- [pipeline.py:435-470](src/ingestion/pipeline.py#L435-L470) — Stage 6 三类存储协调写入与 ID 对齐

### 📄 相关文档
- [DEV_SPEC.md](DEV_SPEC.md) — 混合检索与存储层架构设计原理
- [config/settings.yaml](config/settings.yaml) — vector_store 配置项

### 🔗 参考资料
- SQLite WAL Mode — Write-Ahead Logging 允许读写并发
- Atomic File Write Pattern — 先写临时文件再 rename 防止数据损坏

### 💡 建议学习路径
1. 先阅读 pipeline.py:435-470 理解 Stage 6 完整流程
2. 重点关注第 443-444 行 ID 对齐操作
3. 阅读 bm25_indexer.py:518-548 理解原子写入实现
4. 运行 `python scripts/ingest.py --file documents/sample.pdf --collection test` 观察日志
