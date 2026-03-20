# Ingestion Pipeline - D2.1 Pipeline整体流程

> 学习日期: 2026-03-19
> 综合评分: 7/10 | 追问轮数: 3/4

---

**Q1: 在 `pipeline.py` 中，`IngestionPipeline.run()` 方法将文档摄取分为 6 个阶段（Stage 1 ~ Stage 6）。请你按顺序列出这 6 个阶段的名称，并说明每个阶段的输入是什么、输出是什么，以及它们之间的数据是如何流转的**

**标准答案**: `IngestionPipeline.run()` 分为 6 个阶段，数据流为：文件路径 → hash → Document → Chunks → 增强 Chunks → 向量+稀疏统计 → 持久化存储。Stage 1（File Integrity Check）：计算文件 SHA256 哈希，查询 SQLite `ingestion_history` 表判断是否已处理，已处理则早退跳过。Stage 2（Document Loading）：`PdfLoader` 将 PDF 解析为 `Document` 对象，包含全文 Markdown 文本和提取的图片列表。Stage 3（Chunking）：`DocumentChunker` 将 Document 切分为 `List[Chunk]`。Stage 4（Transform Pipeline）：对 Chunks 依次做三轮原地变换——`ChunkRefiner`（文本精炼）→ `MetadataEnricher`（元数据注入）→ `ImageCaptioner`（图片 Caption 生成）。Stage 5（Encoding）：`BatchProcessor` 协调 `DenseEncoder` 和 `SparseEncoder`，将增强后的 Chunks 编码为稠密向量和 BM25 稀疏统计。Stage 6（Storage）：分三路写入——稠密向量进 ChromaDB（`VectorUpserter`）、稀疏统计建 BM25 索引（`BM25Indexer`）、图片注册到 ImageStorage（SQLite），最后调用 `mark_success`（`pipeline.py:523`）标记文件已处理，保证幂等。

---

**Q2: Stage 4 的 ChunkRefiner 和 MetadataEnricher 都有 `use_llm` 配置项。如果 LLM 调用失败了，Pipeline 会怎样处理？这种设计叫什么原则？为什么在 Transform 阶段特别需要这种设计？**

**标准答案**: 采用 **Graceful Degradation（优雅降级）** 设计原则。ChunkRefiner 和 MetadataEnricher 都支持双模式：先执行 rule-based 处理作为保底结果，再尝试调用 LLM 增强，若 LLM 调用失败则直接使用保底结果输出，不阻塞 Pipeline。Transform 阶段特别需要这种设计，因为 LLM 增强是"锦上添花"——没有增强的 Chunk 仍可被编码和检索。注意：Stage 5 Encoding 也调用外部 API（Embedding Provider 如 `text-embedding-ada-002`），但 Encoding 阶段**不能降级**——没有向量就无法存储和检索，只能直接报错。两者的区别在于：Transform 的产出是可选增强，Encoding 的产出是必要数据。

---

**Q3: ChunkRefiner 的 rule-based 模式和 LLM 模式，分别做了哪些事情？它们在输出质量上有什么具体差异？**

**标准答案**: **rule-based 模式**只做模式固定的噪声清理——去除分隔线、HTML 标签等正则可匹配的内容，只能处理结构化的、可预测的噪声。**LLM 模式**能处理上下文相关的噪声——乱码修复、跨页断句合并、碎片段落智能重组等，使文本语义更连贯。质量差异直接影响下游：精炼后的文本语义越连贯，Embedding 向量质量越高，最终检索召回和排序效果越好。具体实现见 `src/ingestion/transform/chunk_refiner.py`。

---

**Q4: 如果 Pipeline 在执行过程中某个阶段（比如 Stage 5 Encoding）抛出异常，Pipeline 会怎么处理？`mark_failed` 和 `mark_success` 分别在什么时候调用？这种设计对"幂等性"有什么影响？**

**标准答案**: 整个 `run()` 方法（`pipeline.py:229-552`）使用**一个外层 try/except** 包裹所有 6 个阶段。任何阶段抛出异常都会跳到同一个 `except` 块（`pipeline.py:542`），调用一次 `mark_failed(file_hash, file_path, error_msg)`——不是每个阶段各自调用。`mark_success`（`pipeline.py:523`）只在 6 个阶段全部顺利完成后才执行。对幂等性的影响：失败的文件 hash 未被标记 success，下次运行时 `should_skip()` 返回 False，允许自动重试；已成功的文件返回 True，直接跳过避免重复处理。另外 `force=True` 参数可强制绕过跳过逻辑，重新处理已成功的文件。

---

## 📊 评价报告

**知识域**: D2 Ingestion Pipeline > **知识点**: D2.1 Pipeline 整体流程 — Ingestion Pipeline 6 阶段流程与设计原则
**追问轮数**: 3/4

### ✅ 回答亮点
- 6 个阶段的名称、输入输出、数据流转描述完整准确
- Stage 4 三个子步骤的执行顺序（精炼→enrichment→caption）记忆清晰
- Graceful Degradation 的机制（rule-based 保底 + LLM 增强）理解到位
- rule-based vs LLM 的质量差异分析有深度，且正确关联了下游 Embedding 质量影响
- 幂等性核心逻辑（失败可重试、成功则跳过）理解正确

### ⚠️ 需要加强
- Stage 5 Encoding 也调用外部 API（Embedding Provider），不仅 Transform 阶段调用外部服务
- `mark_failed` 的调用位置是外层 `except` 块的统一处理，不是每个阶段各自调用
- 可以多引用具体代码位置（如行号、类名）来增强回答的说服力

### 📈 评分明细

| 维度 | 分数 | 说明 |
|------|------|------|
| 准确性 | 7/10 | 6 阶段描述准确，两处细节有偏差（外部 API 范围、mark_failed 机制） |
| 深度 | 7/10 | Graceful Degradation 和质量差异分析有深度，错误处理部分偏表面 |
| 代码关联 | 6/10 | 理解了代码逻辑，但回答中较少引用具体文件/行号/类名 |
| 设计思维 | 8/10 | 降级设计的合理性分析优秀，理解了"锦上添花 vs 不可缺少"的区别 |

### 🏆 综合评分: 7/10

### 📊 学习进度: 0/45 知识点已掌握

---

## 📚 学习指南

### 📂 相关代码
- `pipeline.py:229-552` — `run()` 方法完整流程，重点看 try/except 结构和 `_notify` 回调
- `pipeline.py:120-195` — `__init__` 中 6 个阶段的组件初始化顺序
- `chunk_refiner.py` — rule-based 与 LLM 双模式的具体实现
- `file_integrity.py` — `should_skip` / `mark_success` / `mark_failed` 的 SQLite 实现

### 📄 相关文档
- `DEV_SPEC.md` — 第 2 节核心特点中的分块策略与混合检索设计理念

### 🔗 参考资料
- **Graceful Degradation** — 系统设计原则：核心功能保持可用，增强功能允许降级，常见于微服务架构

### 💡 建议学习路径
1. 先阅读 `file_integrity.py` 理解 `should_skip` / `mark_success` / `mark_failed` 的 SQLite 表结构
2. 再看 `chunk_refiner.py` 对比 rule-based 和 LLM 两条代码路径
3. 运行 `python scripts/ingest.py --file test.pdf --collection test` 实际体验 Pipeline 日志输出
4. 尝试将 `settings.yaml` 中 `chunk_refiner.use_llm` 设为 `false`，对比摄取结果差异
