# Ingestion Pipeline - D2.2 Chunking策略

> 学习日期: 2026-03-20
> 综合评分: 7.5/10 | 追问轮数: 2/3

---

## Q1: RecursiveSplitter 的分割逻辑与 DocumentChunker 的业务增强

**标准答案**: 本项目的 Chunking 实现分为两层：`src/libs/splitter/`（纯文本分割工具层）和 `src/ingestion/chunking/`（业务适配层）。

**RecursiveSplitter** 封装了 LangChain 的 `RecursiveCharacterTextSplitter`，核心思想是按分隔符优先级列表递归降级切分。`DEFAULT_SEPARATORS` 定义了从高到低的优先级：`"\n\n"`（段落）→ `"\n"`（行）→ `". "`（句子）→ `"! "` / `"? "` / `"; "` / `", "`（标点）→ `" "`（空格）→ `""`（字符）。分割时先按最高优先级分隔符切割，若某个片段仍超过 `chunk_size`，则降级到下一个分隔符继续切分，直至所有片段满足长度约束。这样在满足长度要求的同时，尽可能保留高层级的语义边界（段落 > 句子 > 词）。

**DocumentChunker** 是业务适配层（Adapter Pattern），在纯文本分割之上做了以下增强：
1. **确定性 ID 生成**：`_generate_chunk_id` 采用 `{doc_id}_{index:04d}_{content_hash}` 三段式格式
2. **元数据继承**：将 Document 的 metadata 浅拷贝到每个 Chunk，并添加 `chunk_index`、`source_ref` 字段
3. **图片引用提取**：通过正则匹配 `[IMAGE: xxx]` 占位符，提取 `image_refs` 并关联完整图片元数据
4. **类型转换**：将纯文本字符串转换为符合 `core.types.Chunk` 契约的业务对象

---

## Q2: chunk ID 三段式设计的必要性与 chunk_overlap 的作用

**标准答案**: 三段式 ID `{doc_id}_{index:04d}_{content_hash}` 中，`index` 和 `content_hash` 缺一不可：
- **只有 index**：内容变了但位置没变时，ID 不变，导致静默覆盖旧数据，无法感知内容更新
- **只有 hash**：同文档内重复段落的 hash 相同，导致 ID 冲突，丢失 chunk
- **两者组合**：index 保证唯一性，hash 保证内容感知，共同实现幂等 upsert 和增量更新检测

`chunk_overlap`（默认 200 字符）使相邻 chunk 共享一段重叠文本，防止语义在切分边界处断裂。例如，一个完整句子被切到两个 chunk 的边界时，重叠区域可以让两个 chunk 都包含这个句子，避免检索时因为边界截断而丢失关键信息。若设为 0，相邻 chunk 完全不共享文本，边界处的信息会被硬切断，导致语义断裂，降低检索召回质量。但 overlap 也不宜过大，否则会增加冗余存储和 embedding 计算成本。

---

## Q3: 如何新增 SemanticSplitter 扩展分割策略

**标准答案**: 得益于工厂模式 + 基类抽象的可插拔设计，新增 `SemanticSplitter` 只需三步：

1. **实现接口**：创建 `src/libs/splitter/semantic_splitter.py`，继承 `BaseSplitter`，实现 `split_text()` 方法。SemanticSplitter 的实现需要引入 Embedding 模型来计算相邻句子/段落的语义相似度，当相似度低于阈值时作为切分点
2. **注册工厂**：在 `splitter_factory.py` 的 `_register_builtin_providers()` 中添加 `SplitterFactory.register_provider("semantic", SemanticSplitter)`
3. **配置切换**：`settings.yaml` 中设置 `ingestion.splitter: "semantic"` 即可启用

关键设计优势：**`DocumentChunker` 完全不需要修改**——它只依赖 `SplitterFactory.create()` 返回的 `BaseSplitter` 接口，与具体分割策略解耦。这正是适配器模式 + 工厂模式组合的价值：上层业务代码对底层策略变化完全透明。

---

## 📊 评价报告

**知识域**: Ingestion Pipeline > **知识点**: D2.2 Chunking 策略 — RecursiveSplitter 分割逻辑与 DocumentChunker 业务增强
**追问轮数**: 2/3

### ✅ 回答亮点
- 对 RecursiveSplitter 递归降级分割机制的描述精准——"在满足长度约束下尽可能保留高层级语义边界"
- 对 chunk ID 三段式设计的分析非常出色，清晰指出了只用 index 或只用 hash 各自的缺陷
- 扩展新 Splitter 的步骤简洁且完整，体现了对工厂模式的理解

### ⚠️ 需要加强
- 对 `DEFAULT_SEPARATORS` 的具体层级（段落→行→句→词→字符）可以更明确地展开
- chunk_overlap 的回答偏概念化，可以结合具体场景举例说明
- 扩展性回答中未提到 DocumentChunker 无需修改这一关键设计优势，也未提及 SemanticSplitter 的实现挑战

### 📈 评分明细

| 维度 | 分数 | 说明 |
|------|------|------|
| 准确性 | 8/10 | 三轮回答均无事实错误，核心逻辑正确 |
| 深度 | 7/10 | ID 设计分析有深度，但 overlap 和扩展性部分可以更深入 |
| 代码关联 | 7/10 | 提及了关键文件和方法，但未引用具体代码行或细节 |
| 设计思维 | 8/10 | index+hash 互补分析体现了优秀的工程取舍思维 |

### 🏆 综合评分: 7.5/10

---

## 📚 学习指南

### 📂 相关代码
- `src/libs/splitter/recursive_splitter.py:42-52` — DEFAULT_SEPARATORS 分隔符优先级列表
- `src/libs/splitter/recursive_splitter.py:109-117` — LangChain splitter 初始化参数
- `src/ingestion/chunking/document_chunker.py:140-169` — _generate_chunk_id 三段式 ID 生成
- `src/ingestion/chunking/document_chunker.py:171-249` — _inherit_metadata 元数据继承 + image_refs
- `src/libs/splitter/splitter_factory.py:32-113` — SplitterFactory 注册与创建机制

### 📄 相关文档
- `DEV_SPEC.md` — 3.1.1 数据摄取流水线设计
- `config/settings.yaml:98-112` — ingestion 配置项

### 🔗 参考资料
- LangChain RecursiveCharacterTextSplitter — 底层分割实现
- Chunking 对 RAG 检索质量的影响 — chunk_size/overlap 调优

### 💡 建议学习路径
1. 先阅读 `src/libs/splitter/base_splitter.py` 理解接口契约
2. 再看 `src/libs/splitter/recursive_splitter.py` 掌握分隔符降级实现
3. 尝试修改 `config/settings.yaml` 中的 chunk_size 和 chunk_overlap，重新 ingest 后观察变化
