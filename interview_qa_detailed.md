# 模拟面试题（详细版） — 按关键词逐条深挖

> 针对简历中 RAG + MCP 知识检索中枢项目，按关键词逐一设计面试问题与详细参考答案。
> 面试官视角：从简历描述出发，验证候选人是否真正理解并实现了所述内容。

---

## 一、RAG 数据流水线（整体架构）

### Q1: 简单介绍一下你这条 RAG 数据流水线的整体流程，每个阶段的输入输出是什么？

**参考答案：**

整条流水线分为 5 个阶段，遵循 Load → Split → Transform → Embed → Upsert 的线性流程：

| 阶段 | 输入 | 输出 | 关键操作 |
|------|------|------|---------|
| **Load** | 原始 PDF/文档文件 | Canonical Markdown + metadata（source_path, doc_type, title 等） | MarkItDown 将 PDF 转为标准 Markdown，同时提取元信息；**前置 SHA256 文件哈希去重**，已处理过的文件查询 `ingestion_history.db` 后直接跳过，避免重复消耗资源 |
| **Split** | Markdown 文本 | Chunk 列表，每个 chunk 携带 `chunk_index`（在原文中的序号）和 `start_offset`（在原文中的字符偏移量） | 使用 LangChain 的 `RecursiveCharacterTextSplitter`，按 Markdown 结构（标题/段落/代码块）语义优先切分，递归降级保证每个 chunk 不超过 `chunk_size` |
| **Transform** | 原始 Chunk 列表 | 增强后的 Chunk（正文可能被合并/去噪，并新增 Title/Summary/Tags 等 metadata 字段） | 三个 LLM 增强步骤串行执行：ChunkRefiner（合并被物理切断的语义段落、去除页眉页脚乱码）→ MetadataEnricher（LLM 为每个 chunk 生成 Title/Summary/Tags 注入 metadata）→ ImageCaptioner（Vision LLM 为图片生成文字描述，缝入 chunk 正文） |
| **Embed** | 增强后的 Chunk | Dense 向量（高维稠密向量）+ BM25 稀疏倒排索引条目 | **双路并行编码**：Dense 路调用 Embedding API 生成向量，Sparse 路更新 BM25 倒排索引。关键优化是**差异计算**——通过比对 chunk 内容哈希，只对内容有变更的 chunk 调用 Embedding API，未变更的直接复用已有向量，大幅节省 API 成本 |
| **Upsert** | 向量 + 索引数据 | 持久化写入存储 | 写入 ChromaDB 向量库（Dense 向量 + chunk 文本 + metadata）和 BM25 索引文件。**幂等设计**：chunk_id 由确定性哈希生成，Upsert 操作对已存在的 chunk 执行覆盖写入而非重复插入 |

**设计亮点总结**：整条流水线贯穿两个核心设计原则——（1）**幂等性**：文件级 SHA256 去重 + chunk 级确定性哈希保证重复执行不产生副作用；（2）**成本优化**：差异 Embed 避免未变更 chunk 重复调用 API。

---

### Q2: 这条流水线的幂等性是怎么保证的？如果同一个文件被重复提交会发生什么？

**参考答案：**

幂等性通过**两级去重机制**实现，两级分别防护不同粒度的重复：

**第一级：文件级去重（粗粒度，入口拦截）**
- 在 Load 阶段最前端，计算源文件的 SHA256 哈希值
- 查询 `ingestion_history.db`（SQLite 数据库），该库记录了每个已处理文件的 `source_path` + `SHA256 hash` + 处理时间
- 如果 SHA256 匹配（文件内容完全相同），直接跳过整个文件，不进入后续任何阶段
- **作用**：避免同一文件被重复投入整条流水线，节省 LLM 调用和 Embedding API 成本

**第二级：Chunk 级幂等（细粒度，存储层保障）**
- chunk_id 由 `hash(source_path + section_path + content_hash)` **确定性生成**
- 相同来源、相同位置、相同内容的 chunk 永远生成相同的 chunk_id
- Upsert 操作语义：如果 chunk_id 已存在则覆盖，不存在则插入
- **作用**：即使绕过了文件级检查（比如文件名变了但内容部分重叠），chunk 级也不会产生重复数据

**实际场景举例**：
- 同一 PDF 重复上传 → 第一级拦截，零 API 调用
- PDF 更新了部分内容重新上传 → 第一级放行（SHA256 变了），Split 后部分 chunk 内容未变 → Embed 阶段差异计算跳过未变更 chunk → Upsert 阶段幂等覆盖
- 同一内容以不同文件名上传 → 第一级放行（路径不同），但 chunk_id 中包含 content_hash，如果 section_path 也相同则 ID 相同，Upsert 幂等覆盖；如果 source_path 不同则视为不同来源的 chunk（这是合理的，因为可能需要分别追踪引用来源）

---

## 二、递归切分（RecursiveCharacterTextSplitter）

### Q3: 递归切分的 "递归" 体现在哪里？它按什么规则拆分？

**参考答案：**

RecursiveCharacterTextSplitter 的核心思想是**按语义层级递归降级切分**，具体机制：

**分隔符优先级列表**（以 Markdown 为例）：
```
["\n## ", "\n### ", "\n\n", "\n", ". ", " ", ""]
```

**递归过程**：
1. 首先尝试用最高优先级分隔符（如 `\n## ` 标题分隔）切分整篇文档
2. 检查每个切出的片段是否 ≤ `chunk_size`
3. 如果某个片段仍然超过 `chunk_size`，对该片段**递归降级**到下一个分隔符继续切分
4. 最终级别是空字符串 `""`——即按字符硬切，保证一定能满足大小限制

**"递归"的含义**：不是对整篇文档用单一规则一刀切，而是**逐层细化**。最终效果是：
- 能按标题分开的就按标题分 → 最大化语义完整性
- 标题段落太长就按段落分 → 次优保留
- 段落太长就按句子分 → 再次优
- 最差情况按字符硬切 → 保证不超限

**与简单 CharacterTextSplitter 的对比**：后者只用单一分隔符，无法适应文档结构的多样性。RecursiveCharacterTextSplitter 通过优先级列表和递归机制，在切分大小限制和语义完整性之间取得了更好的平衡。

---

### Q4: chunk_size 和 chunk_overlap 这两个参数你们是怎么定的？调参依据是什么？

**参考答案：**

**chunk_size 的选择逻辑**：

通常范围在 512–1024 tokens，我们的选择依据是：
- **太小**（如 128 tokens）：每个 chunk 信息量不足，上下文被过度碎片化。检索时可能命中了正确区域但 chunk 内容不足以支撑回答，导致 Faithfulness 指标下降
- **太大**（如 2048 tokens）：chunk 中混入大量不相关内容，引入噪声。Dense Embedding 对长文本的语义压缩会丢失细节，导致检索精度（Context Precision）下降
- **调参方法**：在 Golden Test Set 上分别跑 chunk_size = 256/512/768/1024/1536，对比 Hit Rate@K 和 MRR，选择指标最优的配置。在我们的风控文档场景中，512–768 区间表现最好

**chunk_overlap 的作用和选择**：

- overlap 的本质目的是**防止信息丢失在切分边界**。假如一段关键论述恰好被切分点一分为二，没有 overlap 的话两个 chunk 各持有一半信息，都不完整
- 通常设为 chunk_size 的 10%–20%（如 chunk_size=1000 时 overlap=100–200）
- **过大的 overlap**：chunk 间大量重复内容，存储膨胀，检索时同一段内容可能被多个 chunk 重复命中，浪费 Reranker 处理量
- **过小的 overlap**：无法有效防止边界切断

实际上 ChunkRefiner 在 Transform 阶段也能部分缓解切分边界问题（通过 LLM 智能合并），所以 overlap 不需要设得特别大。

---

## 三、Transform 智能增强

### Q5: ChunkRefiner 具体做了什么？为什么不直接调整 Splitter 参数而要用 LLM？

**参考答案：**

**ChunkRefiner 的两个核心功能**：

1. **语义合并**：识别被物理切分点切断但逻辑上属于同一语义单元的段落，进行合并。典型场景：
   - "问题描述"和"解决方案"被切到两个 chunk → 合并为一个完整的问题-方案对
   - 一个列表的前半部分和后半部分被切断 → 合并为完整列表
   - 表格的标题行和数据行被分离 → 合并为完整表格

2. **去噪清洗**：
   - 移除 PDF 转换过程中残留的页眉、页脚、页码
   - 清除乱码和无意义的格式字符（如 `\x00`、重复的分隔线 `---`）
   - 移除与正文无关的法律声明、版权信息等模板文本

**为什么不能只靠调 Splitter 参数解决**：

Splitter 是**基于规则的字符级操作**，存在根本性限制：
- 它无法理解"问题描述"和"解决方案"在语义上是一个整体——即使你把 chunk_size 调大，可能包含了完整的问题-方案对，但同时也引入了大量不相关内容
- 它无法区分页眉页脚和正文——规则无法判断"第 X 页"是正文内容还是页脚
- 这些判断本质上需要**语义理解能力**，只有 LLM 能胜任

**ChunkRefiner 的实现方式**：通过精心设计的 Prompt 告诉 LLM 当前 chunk 的上下文（前后相邻 chunk 的片段），让 LLM 判断是否需要合并以及哪些内容是噪声。LLM 输出精炼后的 chunk 文本。

**成本考量**：ChunkRefiner 需要对每个 chunk 调用一次 LLM，增加了 Ingestion 的时间和成本。但这是一次性的离线处理，换来的是检索阶段永久的质量提升，ROI 很高。

---

### Q6: 元数据注入增强（MetadataEnricher）产出的 Title/Summary/Tags 在检索时怎么用？

**参考答案：**

**MetadataEnricher 的产出和存储位置**：

调用 LLM 为每个 chunk 生成三个结构化字段，存入 Chroma 的 document metadata：
- **Title**：chunk 的简短标题（如"BM25 评分公式详解"）
- **Summary**：chunk 内容的一句话摘要（如"本节介绍 BM25 的 TF-IDF 改进及参数含义"）
- **Tags**：关键词标签列表（如 `["BM25", "TF-IDF", "检索算法"]`）

**在检索时的三种用途**：

1. **Metadata Filtering（Tags 过滤）**：
   - 查询时可通过 Chroma 的 `where` 参数按 Tags 过滤，先缩小搜索范围再做向量相似度计算
   - 例：用户查询限定了"风控规则"领域 → 先过滤 Tags 包含"风控"的 chunk → 再在子集上做 Dense/Sparse 检索
   - 效果：减少无关候选，提高 Precision，同时加速检索（搜索空间更小）

2. **检索文本增强（Summary 拼接）**：
   - 可选地将 Summary 拼接到 chunk 正文中一起做 Embedding
   - 效果：Summary 是 LLM 对 chunk 的语义提炼，包含的关键语义信号比原文更集中，能提升 Dense 路的召回率

3. **展示层（Title 用于 UI）**：
   - Dashboard 数据浏览器中显示 Title 而非原文片段，方便用户快速浏览和理解 chunk 内容
   - MCP 返回结果中也携带 Title，供下游 Agent 或用户界面展示

**不做 MetadataEnricher 的后果**：chunk 只有原始正文，没有结构化语义标签，无法做精细过滤，Dashboard 展示也只能显示正文截断片段，用户体验差。

---

## 四、双路 Embedding

### Q7: Dense Embedding 和 Sparse Embedding 分别是什么？为什么要双路并行而不是只用一种？

**参考答案：**

**Dense Embedding（稠密向量检索）**：
- 使用预训练语言模型（如 OpenAI text-embedding-ada-002）将文本编码为固定长度的高维稠密向量（如 1536 维，每个维度都有非零浮点值）
- 查询时将 query 也编码为向量，通过 Cosine Similarity 计算与所有 chunk 向量的相似度
- **核心优势**：语义理解。"用户异常行为"和"客户违规操作"虽然没有共同关键词，但在向量空间中距离很近
- **核心劣势**：对罕见专有名词容易"语义漂移"——模型可能没见过某个产品代号，将其映射到语义空间中的错误位置

**Sparse Embedding（稀疏向量/BM25 倒排索引）**：
- 基于词频统计，构建倒排索引：每个词 → 包含该词的文档列表 + 词频
- 查询时分词后查倒排索引，按 BM25 公式（考虑 TF、IDF、文档长度归一化）计算相关性分数
- **核心优势**：精确关键词匹配。产品代号"XR-7200"、API 名称"get_user_risk_score"这类专有名词，BM25 能精确命中
- **核心劣势**：无语义理解。query 写"异常行为"而文档写"违规操作"，BM25 一条都召回不了

**为什么必须双路并行**：

在风控场景中，用户查询的类型是多样的：
- 语义模糊型："这个产品有什么安全风险？" → Dense 路擅长
- 精确关键词型："XR-7200 的 KYC 规则编号" → Sparse 路擅长
- 混合型："XR-7200 用户异常行为处理方案" → 两路各发挥所长

单路方案在另一种查询类型上必然有盲区。Hybrid Search 双路并行 + RRF 融合，让两种能力互补，覆盖更多查询场景。这也是 Hit Rate 能达到 92% 的关键因素——单 Dense 约 78%，单 BM25 约 72%，Hybrid 融合后显著提升。

---

### Q8: Dense Embedding 用的是什么模型？向量维度是多少？存在哪里？

**参考答案：**

**Embedding 模型**：
- 默认使用 OpenAI 的 `text-embedding-ada-002`（1536 维）或更新的 `text-embedding-3-small`（可配置维度）
- 通过可插拔架构（`BaseEmbedding` 抽象类 + Factory 模式）支持多种 Provider：Azure OpenAI、Ollama 本地模型等
- 在 `config/settings.yaml` 中配置 `embedding.provider` 即可切换，零代码修改

**向量维度**：
- `text-embedding-ada-002`：固定 1536 维
- `text-embedding-3-small`：默认 1536 维，支持降维（如 512、256）以换取存储空间和检索速度

**存储位置和索引结构**：
- 向量存入 ChromaDB（`data/db/chroma/` 目录）
- Chroma 内部使用 **HNSW（Hierarchical Navigable Small World）** 算法构建近似最近邻索引
- HNSW 是一种图结构索引，查询时间复杂度 O(log n)，远优于暴力扫描的 O(n)，适合中等规模（百万级以下）的向量检索
- 选择 Chroma 而非 Pinecone/Qdrant 的原因：Local-First 设计原则，零外部服务依赖，`pip install` 即可运行，适合私有知识库场景

---

## 五、持久化存储

### Q9: 系统用了哪几类存储？各自存什么？为什么不能合并成一个？

**参考答案：**

系统使用 **4 类独立存储**，各自服务于不同的数据模型和访问模式：

| 存储 | 路径 | 存储内容 | 数据结构 | 为什么独立 |
|------|------|---------|---------|-----------|
| **ChromaDB** | `data/db/chroma/` | Dense 向量 + chunk 文本 + metadata（Title/Summary/Tags 等） | HNSW 图索引 + KV 存储 | 向量检索需要专用的 ANN 索引结构，BM25 的倒排索引完全不同 |
| **BM25 Index** | `data/db/bm25/` | 稀疏倒排索引（词 → 文档列表 + 词频） | 倒排索引（当前用 pickle 序列化） | 倒排索引的数据结构和查询算法与向量检索完全不同，硬塞进向量库会丧失 BM25 的查询效率 |
| **ingestion_history.db** | `data/db/` | 文件处理记录：source_path + SHA256 + 处理时间 | SQLite 关系表 | 这是**文件级**元数据，粒度与 chunk 不同。用于 Load 阶段快速判断文件是否已处理，不需要也不应该和 chunk 数据混在一起 |
| **image_index.db** | `data/db/` | image_id → 图片文件路径的映射 | SQLite 关系表 | 图片是二进制文件，不适合存入向量库。检索命中含图片的 chunk 后，通过此映射找到原始图片文件，编码为 Base64 通过 MCP 返回 `ImageContent` |

**为什么不能合并**：
- 技术层面：向量索引（HNSW）、倒排索引（BM25）、关系表（SQLite）是三种完全不同的数据结构，各自有最优的查询算法。强行统一会导致查询效率大幅下降
- 职责层面：遵循**单一职责原则**——每个存储只关心自己的数据模型，删除文档时需要协调四个存储的清理（通过 DocumentManager 的协调删除机制）

---

### Q10: chunk_id 是怎么生成的？为什么不用 UUID？

**参考答案：**

**chunk_id 的生成公式**：
```
chunk_id = hash(source_path + section_path + content_hash)
```
- `source_path`：源文件路径（标识来源）
- `section_path`：在文档结构中的位置（如 "第3章 > 3.2 节"，标识章节位置）
- `content_hash`：chunk 内容的哈希（标识具体内容）

这三个字段组合起来唯一标识一个 chunk 的"来源 + 位置 + 内容"，生成的 hash 是**确定性的**——相同输入永远产生相同输出。

**为什么不用 UUID**：

UUID（v4）是随机生成的，同一文件每次处理都会产生不同的 UUID，这会带来严重问题：
1. **重复堆积**：同一文档重新处理后，旧 chunk（旧 UUID）和新 chunk（新 UUID）共存，向量库中数据量不断膨胀
2. **无法幂等 Upsert**：Upsert 依赖 ID 判断"已存在则覆盖"，UUID 不同就永远是"新插入"
3. **无法做差异 Embed**：Embed 阶段需要判断某个 chunk 是否已经有向量了，UUID 每次不同就无法匹配

确定性哈希解决了以上所有问题：相同内容 → 相同 ID → Upsert 覆盖 → 幂等。

**追问：如果文件改了一行，受影响的 chunk_id 会变吗？**

只有**内容实际被修改的那些 chunk** 的 `content_hash` 会变，导致 chunk_id 变化。未修改的 chunk 保持相同 ID。这正是差异 Embed 的基础——通过比对 chunk_id 判断哪些是新的/变更的，只对这些调用 Embedding API。

---

## 六、Hybrid Search 混合检索架构

### Q11: Hybrid Search 的执行流程是什么？两路检索是串行还是并行？

**参考答案：**

**完整执行流程**（4 个阶段）：

```
         ┌→ Dense Retriever (ChromaDB Cosine Similarity) → Top-N 语义候选 ─┐
Query ──┤                                                                    ├→ RRF Fusion → Reranker → Top-K 最终结果
         └→ Sparse Retriever (BM25 倒排索引匹配) → Top-N 关键词候选 ────────┘
```

1. **Query 预处理**：QueryProcessor 对原始查询做标准化处理（去除多余空白、标点规范化等）
2. **双路并行召回**：
   - Dense 路：query → Embedding API 编码为向量 → ChromaDB HNSW 索引做 Cosine Similarity 搜索 → 返回 Top-N 语义相似候选
   - Sparse 路：query → 分词 → BM25 倒排索引匹配 → 按 BM25 公式计算分数 → 返回 Top-N 关键词匹配候选
   - **两路完全并行**，互不依赖，各自独立返回结果
3. **RRF 融合**：等两路都返回后，按 RRF 公式 `Score(d) = Σ 1/(k + rank_i(d))` 对两路结果按排名融合，输出统一排序的候选列表
4. **Rerank 精排**：融合结果送入 Cross-Encoder 或 LLM Reranker，对 Top-M 候选精细重排，输出最终 Top-K

**为什么并行而非串行**：两路使用的索引结构完全不同（HNSW vs 倒排索引），查询逻辑无数据依赖，并行执行可将延迟从"两路之和"降低到"两路取大"。

---

### Q12: ChromaDB 在语义检索中的角色是什么？它内部用什么索引结构？

**参考答案：**

**ChromaDB 的角色**：
ChromaDB 是系统中 Dense Retrieval 路的核心存储和检索引擎，负责：
- **存储**：chunk 的 Dense Embedding 向量（1536 维浮点数组）+ chunk 原始文本 + metadata 字段（Title/Summary/Tags/source_path 等）
- **检索**：接收 query 向量，执行近似最近邻（ANN）搜索，返回 Top-N 最相似的 chunk 及其相似度分数

**HNSW 索引结构**：
Chroma 内部使用 **HNSW（Hierarchical Navigable Small World）** 算法：
- 构建一个多层的图结构，底层包含所有向量节点，上层是底层的稀疏采样
- 查询时从顶层开始贪心搜索，逐层下降到底层，每层找到当前最近邻后跳到下一层继续精细搜索
- **时间复杂度**：O(log n)，比暴力扫描 O(n) 快几个数量级
- **空间复杂度**：额外存储图的边信息，约为原始向量大小的 1.5–2 倍
- **精度**：近似搜索，不保证绝对最优，但 recall@100 通常 >99%

**选择 Chroma 的原因**：
- Local-First：嵌入式数据库，无需额外部署服务进程
- 零依赖：`pip install chromadb` 即可，不需要 Docker/K8s
- 适合规模：万到十万级 chunk 性能优秀，契合私有知识库场景
- 局限：百万级以上可能需要考虑 Qdrant/Milvus 等分布式方案

---

### Q13: BM25 Index 的关键词检索具体怎么工作的？IDF 怎么算？

**参考答案：**

**BM25 的核心数据结构——倒排索引**：
```
倒排索引结构示例：
"风控" → [(chunk_1, tf=3), (chunk_5, tf=1), (chunk_12, tf=2)]
"规则" → [(chunk_1, tf=2), (chunk_3, tf=4), (chunk_12, tf=1)]
"KYC"  → [(chunk_5, tf=1)]
```
每个词映射到包含它的 chunk 列表，以及该词在每个 chunk 中的出现次数（term frequency）。

**查询流程**：
1. 对 query 进行分词（如"风控规则 KYC" → ["风控", "规则", "KYC"]）
2. 对每个词查倒排索引，获取匹配的 chunk 集合
3. 对每个匹配的 chunk，按 BM25 公式计算综合相关性分数
4. 按分数降序排列，返回 Top-N

**BM25 评分公式**：
```
Score(q, d) = Σ IDF(t) × (tf(t,d) × (k1 + 1)) / (tf(t,d) + k1 × (1 - b + b × dl/avgdl))
```

各参数含义：
- `tf(t,d)`：词 t 在文档 d 中的出现频率
- `dl`：文档 d 的长度（词数）
- `avgdl`：语料库中所有文档的平均长度
- `k1`（通常 1.2–2.0）：**词频饱和度参数**。控制 tf 的增长速度——k1 越大，高频词的贡献增长越慢（一个词出现 100 次不应该比出现 10 次重要 10 倍）
- `b`（通常 0.75）：**文档长度归一化参数**。b=1 完全归一化（长文档被惩罚），b=0 不归一化

**IDF 计算**：
```
IDF(t) = ln((N - df(t) + 0.5) / (df(t) + 0.5) + 1)
```
- `N`：语料库中总文档数
- `df(t)`：包含词 t 的文档数
- 效果：越稀有的词（df 越小），IDF 越高，对检索贡献越大。常见词（如"的""是"）df 接近 N，IDF 趋近于 0

**BM25 相比 TF-IDF 的改进**：引入了词频饱和（k1 参数）和文档长度归一化（b 参数），避免了 TF-IDF 中长文档和高频词被过度加权的问题。

---

## 七、RRF 融合

### Q14: RRF 公式是什么？k=60 是怎么来的？为什么不用线性加权？

**参考答案：**

**RRF 公式**：
```
Score_RRF(d) = Σ_{r ∈ Routes} 1 / (k + rank_r(d))
```
对于每个文档 d，遍历每一路检索结果，取该文档在该路中的排名 `rank_r(d)`，按公式累加得到最终融合分数。

**k=60 的来源**：
- 出自学术论文 Cormack, Clarke & Buettcher (2009) "Reciprocal Rank Fusion outperforms Condorcet and individual Rank Learning Methods"
- k 是**平滑因子**，防止排名第 1 的文档分数过度高估（如果 k=0，第 1 名得分 1.0，第 2 名得分 0.5，差距过大）
- k=60 是论文中经过大量实验验证的经验最优值，实践中通常无需调整
- **调大 k**（如 k=1000）：所有排名的分数差异被压缩，头部优势削弱，趋近于"每路各投一票"
- **调小 k**（如 k=1）：头部排名分数差异放大，接近"赢者通吃"

**为什么不用线性加权**：
线性加权公式：`Score = α × score_dense + (1-α) × score_sparse`

存在两个根本问题：
1. **分数尺度不一致**：Dense 路返回的 Cosine Similarity 范围是 [0, 1]，BM25 分数范围可能是 [0, 30+]。直接线性组合前必须做归一化（min-max? z-score?），归一化方式本身又引入了超参数
2. **权重 α 需要调参**：α 的最优值取决于查询类型（语义型查询 Dense 更重要，关键词型查询 BM25 更重要），固定 α 无法适应所有场景

RRF 的优雅之处：**只看排名不看分数**。排名是序数（第 1、第 2...），天然归一化，不受各路分数绝对值影响。而且不需要调 α 这样的权重超参数，只有一个稳定的 k=60。

---

### Q15: 如果两路检索结果完全不重叠，RRF 融合后会是什么效果？

**参考答案：**

**情景分析**：
假设 Dense Top-5 = [A, B, C, D, E]，Sparse Top-5 = [F, G, H, I, J]，完全不重叠。

RRF 计算（k=60）：
- A 的分数：1/(60+1) = 0.0164（只有 Dense 路贡献）
- F 的分数：1/(60+1) = 0.0164（只有 Sparse 路贡献）
- B 的分数：1/(60+2) = 0.0161
- G 的分数：1/(60+2) = 0.0161
- ...以此类推

**融合效果**：两路的 Top 结果**交错排列**——A, F, B, G, C, H, D, I, E, J。

**这是好事还是坏事**：
- **好事居多**：说明两路在互补地覆盖不同类型的相关文档。Dense 路捕获的是语义相关的文档，Sparse 路捕获的是关键词匹配的文档，两者不重叠说明它们各自发现了对方遗漏的相关结果
- **潜在风险**：如果某一路完全跑偏（返回的全是不相关文档），交错排列会把噪声混入最终结果。但这种情况很少见，而且后面还有 Reranker 精排做最后一道过滤

---

## 八、Cross-Encoder / LLM Rerank 精排

### Q16: Cross-Encoder 和 Bi-Encoder 的本质区别是什么？为什么 Cross-Encoder 不能做粗召回？

**参考答案：**

**Bi-Encoder（双塔模型，用于粗召回）**：
```
Query  → Encoder → Query Vector  ─┐
                                    ├→ Cosine Similarity → Score
Chunk  → Encoder → Chunk Vector  ─┘
```
- Query 和 Chunk **分别独立**通过同一个 Encoder 编码为向量
- Chunk 向量可以**离线预计算**并存入向量库
- 查询时只需编码 Query（一次 Encoder 调用），然后做向量检索（HNSW O(log n)）
- **适合大规模召回**：10 万个 chunk 也只需 1 次 Encoder + 1 次 ANN 搜索

**Cross-Encoder（交叉编码器，用于精排）**：
```
[Query + " [SEP] " + Chunk] → Encoder → Relevance Score
```
- Query 和 Chunk **拼接为一个序列**一起输入 Encoder
- 模型内部的 Self-Attention 能捕捉 Query 和 Chunk 之间的**交互特征**（如 Query 中的"安全"和 Chunk 中的"风险"的关联）
- **精度更高**：因为模型看到了完整的 Query-Chunk 上下文
- **必须实时推理**：每对 (Query, Chunk) 都要跑一次完整的 Encoder forward pass

**为什么 Cross-Encoder 不能做粗召回**：
- 假设有 10 万个 chunk，对每个都要跑一次 Cross-Encoder → 10 万次模型推理
- 即使每次 10ms，总耗时也要 1000 秒，完全不可接受
- Bi-Encoder 做同样的事只需 1 次编码 + 1 次 ANN 搜索 ≈ 100ms

所以架构设计是**两阶段**：Bi-Encoder 粗召回（快速从 10 万缩小到 30 条） → Cross-Encoder 精排（慢速对 30 条精细重排），把 Cross-Encoder 的高精度用在刀刃上。

---

### Q17: LLM Rerank 相比 Cross-Encoder 有什么优劣？什么场景下你会选 LLM Rerank？

**参考答案：**

| 维度 | Cross-Encoder | LLM Rerank |
|------|--------------|------------|
| **速度** | 较快（轻量模型如 MiniLM，CPU 可跑，30 条 ~200ms） | 较慢（调用 GPT-4 等大模型 API，30 条可能 2-5s） |
| **成本** | 低（本地推理，无 API 费用） | 高（每次 Rerank 消耗 tokens，按调用计费） |
| **精度（通用）** | 需要领域微调才能达到最佳 | Zero-shot 能力强，通用理解好，不需要微调 |
| **精度（专业）** | 微调后在特定领域可以非常强 | 对极其专业的领域可能不如微调模型 |
| **可解释性** | 只输出一个 0-1 的相关性分数 | 可以输出排序理由（"该文档讨论了 XX，与查询高度相关因为..."） |
| **上下文窗口** | 受限于模型最大长度（通常 512 tokens） | 大模型支持长上下文（如 128K tokens），可以处理更长的 chunk |

**选择 LLM Rerank 的典型场景**：
1. 候选文档需要**复杂推理**才能判断相关性（如需要理解因果关系、时间线、多步逻辑）
2. 没有领域内微调过的 Cross-Encoder 模型，LLM 的 Zero-shot 能力是更好的选择
3. 需要**可解释的排序理由**反馈给用户或下游系统
4. chunk 较长超出了 Cross-Encoder 的输入限制

**我们的默认选择**：优先 Cross-Encoder（`cross-encoder/ms-marco-MiniLM-L-6-v2`），成本低延迟小，适合大部分场景。LLM Rerank 作为可配置选项，在需要更高精度或可解释性时切换。

---

### Q18: 如果 Reranker 服务挂了，系统怎么处理？

**参考答案：**

系统实现了 **Graceful Fallback（优雅降级）** 机制：

**降级逻辑**：
1. Reranker 调用时设置**超时阈值**（如 5 秒）
2. 如果 Reranker 推理超时、模型加载失败、或抛出任何异常
3. 捕获异常后**自动回退到 RRF 融合结果直接返回**，不经过精排
4. 在 Trace 日志中记录降级事件（包括失败原因和时间戳），供后续排查

**设计思想**：
- Reranker 是**锦上添花的精排层**，不是核心链路的必要组件
- RRF 融合结果本身已经是两路互补的高质量候选，降级后检索质量略降但绝不会中断服务
- 这遵循了"**宁可降级也不能挂**"的可用性原则

**三种 Reranker 后端的降级链**：
```
Cross-Encoder 超时 → 尝试 LLM Rerank → 也失败 → 回退到 RRF 结果
                                                   （或直接配置 reranker: none）
```

实际上项目支持三种 Reranker 后端（Cross-Encoder / LLM Rerank / None），在 `config/settings.yaml` 中配置 `reranker.provider: none` 可以完全跳过精排阶段，适用于对延迟极度敏感的场景。

---

## 九、检索准确率 Hit Rate@10 达 92%

### Q19: Hit Rate@10 是怎么定义和计算的？

**参考答案：**

**严格定义**：
```
Hit Rate@K = |{q ∈ Q : Golden(q) ∩ Top-K(q) ≠ ∅}| / |Q|
```
即：在测试集 Q 的所有查询中，Top-K 检索结果中**至少包含一条**标注的正确答案（Golden Answer）的查询占比。

**直观理解**：
- 100 条测试 query，每条都有标注的"正确答案应该在哪个 chunk"
- 对每条 query 跑完整的 Hybrid Search + Rerank，拿到 Top-10 结果
- 检查这 10 条结果中是否包含标注的正确 chunk
- 92 条命中了 → Hit Rate@10 = 92%

**与其他指标的对比**：
- **Hit Rate@K**：只关心"有没有命中"，不关心正确结果排在第几名 → 衡量**召回能力**
- **MRR（Mean Reciprocal Rank）**：关心"命中的结果排在第几名"，`MRR = 平均(1/第一条正确结果的排名)` → 衡量**头部排序质量**
- 例：正确结果排在第 1 名 vs 第 10 名，Hit Rate 都算命中，但 MRR 分别贡献 1.0 和 0.1

---

### Q20: 这个 92% 是怎么测的？测试集是什么？基线是什么？

**参考答案：**

**测试集（Golden Test Set）**：
- 存储在 `tests/fixtures/golden_test_set.json`
- 格式：每条包含 `query`（测试查询）+ `golden_chunk_ids`（标注的正确 chunk ID 列表）+ `golden_answer`（期望的答案文本）
- 构建方式：基于实际的风控产品文档，人工设计查询场景并标注正确答案对应的 chunk

**评估流程**：
1. EvalRunner 加载 Golden Test Set
2. 对每条 query 执行**完整的 Hybrid Search + Rerank 流程**（与线上流程一致）
3. 检查 Top-K 结果中是否包含 golden_chunk_ids 中的任意一个
4. 汇总统计 Hit Rate@K 和 MRR
5. 同时可选地跑 Ragas 指标（Faithfulness、Answer Relevancy、Context Precision）

**基线对比实验**：

| 策略 | Hit Rate@10 | 说明 |
|------|------------|------|
| 纯 BM25 | ~72% | 缺少语义理解，同义词查询失效 |
| 纯 Dense | ~78% | 缺少精确关键词匹配 |
| Hybrid（RRF 融合，无 Rerank） | ~85% | 两路互补，显著提升 |
| Hybrid + Cross-Encoder Rerank | **92%** | 精排进一步优化头部排序 |

每次策略变更后必须重跑 Golden Test Set 对比，确保不退化。

---

### Q21: 如果一次策略变更（比如换了 Reranker 模型）导致指标下降，你们怎么处理？

**参考答案：**

**标准处理流程**：

1. **发现退化**：变更后自动跑 Golden Test Set 回归评估，对比变更前后的 Hit Rate@K、MRR、Ragas 指标
2. **定位原因**：
   - 是**整体退化**（所有类型 query 都变差）还是**局部退化**（只有特定类型 query 变差）
   - 分析退化的 query 共性：是语义型还是关键词型？是长 query 还是短 query？
   - 对比变更前后的 Rerank 排序差异，找到被错误降权的正确结果
3. **决策**：
   - 如果新模型在某类 query 上更好、另一类更差 → 分析 trade-off，可能保留新模型但调整其他参数补偿
   - 如果整体退化且无法补偿 → 回滚到旧模型
4. **防止未来退化**：将发现退化的 query 加入 Golden Test Set，扩大测试覆盖

**评估体系的多维支持**：
- CompositeEvaluator 支持多评估器并行执行，可以同时跑 Hit Rate、MRR、Ragas Faithfulness、Answer Relevancy、Context Precision
- 单一指标可能掩盖问题（如 Hit Rate 没变但 MRR 大幅下降，说明正确结果虽然在 Top-10 内但排名变差了）

---

## 十、MCP 标准（Model Context Protocol）

### Q22: MCP 是什么？和普通 REST API 有什么区别？

**参考答案：**

**MCP 定义**：
MCP（Model Context Protocol）是 Anthropic 提出的开放标准，专为 **AI Agent 与外部工具/数据源交互**而设计。基于 **JSON-RPC 2.0** 协议，定义了三种标准资源类型：
- **Tools**：Agent 可调用的功能（如 `query_knowledge_hub`）
- **Resources**：Agent 可读取的数据源
- **Prompts**：预定义的 Prompt 模板

**与 REST API 的核心区别**：

| 维度 | REST API | MCP |
|------|---------|-----|
| **设计目标** | 通用的 Web 服务接口 | 专为 AI Agent 设计的上下文协议 |
| **接口发现** | 需要阅读文档、写适配代码 | 标准的 `tools/list` 接口，Client 自动发现可用工具 |
| **集成成本** | 每个客户端需要单独适配（写 HTTP 调用、解析响应、处理认证） | 任何符合 MCP 规范的 Client（Claude Desktop、VS Code Copilot 等）**即插即用**，零适配代码 |
| **类比** | 每个外设自带独特的接口线 | USB 标准——所有设备用同一个接口 |
| **传输层** | HTTP/HTTPS | Stdio（stdin/stdout）或 HTTP+SSE |
| **状态管理** | 通常无状态 | 支持有状态的会话上下文 |

**在本项目中的价值**：
下游风控 Agent（可能是 Claude、GPT 或自研 Agent）只需在 MCP Client 配置中注册本 Server 的启动命令，就能自动发现 `query_knowledge_hub` 等工具并直接调用，**无需编写任何 API 适配代码**。这大大降低了知识检索能力嵌入风控流水线的集成成本。

---

### Q23: Stdio Transport 是怎么工作的？为什么选这种传输方式？

**参考答案：**

**Stdio Transport 工作机制**：
```
MCP Client（如 Claude Desktop）
   ├── 启动子进程：python -m src.mcp_server.server
   ├── stdin  → 向 Server 发送 JSON-RPC 请求
   ├── stdout ← 从 Server 接收 JSON-RPC 响应
   └── stderr ← Server 的日志输出（不影响协议通信）
```

关键要点：
1. Client 以**子进程方式**启动 Server（不是独立部署的服务）
2. 双方通过进程的 stdin/stdout 管道传输 JSON-RPC 消息（一行一条消息）
3. **stdout 只能输出合法的 MCP JSON-RPC 消息**——任何 `print()` 调试输出都会破坏协议，所以日志必须走 stderr
4. 进程生命周期由 Client 管理：Client 退出时子进程也随之终止

**选择 Stdio 的理由**：
1. **零网络依赖**：不需要开放端口、不需要配置 IP/域名、不需要 TLS 证书
2. **零鉴权开销**：进程间通信天然安全，不需要 API Key 或 OAuth
3. **部署极简**：用户只需配置一行启动命令（如 `python -m src.mcp_server.server`），无需 Docker/K8s
4. **天然适合私有知识库**：数据和服务都在本地，不经过网络，满足安全合规要求

**Stdio 的局限和替代方案**：
- **不支持远程调用**：Client 和 Server 必须在同一台机器上
- **不支持多客户端并发**：一个 Server 进程只服务一个 Client
- **替代方案**：MCP 也定义了 HTTP+SSE Transport，支持远程访问和多客户端。如果未来需要团队共享知识库或负载均衡，可以切换传输层而不改变 Tool 实现代码

---

## 十一、Tool Calling 模式嵌入

### Q24: 你们暴露了哪些 MCP Tools？主检索入口的输入输出是什么？

**参考答案：**

**三个核心 MCP Tools**：

| Tool | 功能 | 输入参数 | 输出 |
|------|------|---------|------|
| `query_knowledge_hub` | 主检索入口，执行完整的 Hybrid Search + Rerank 流程 | `query`（必填，查询文本）、`top_k`（可选，返回数量，默认 5）、`collection`（可选，指定文档集合） | 结构化结果列表，每条包含 chunk 内容 + Citation |
| `list_collections` | 列举所有可用的文档集合 | 无参数 | 集合名称列表 + 每个集合的 chunk 数量、文档数量等统计信息 |
| `get_document_summary` | 获取指定文档的摘要和元信息 | `doc_id`（文档标识） | 文档的 Title、摘要、chunk 数量、图片数量、入库时间等 |

**`query_knowledge_hub` 的返回结构（Citation 设计）**：
```json
{
  "results": [
    {
      "content": "chunk 的完整文本内容...",
      "score": 0.87,
      "citation": {
        "source": "产品分析_XR7200.pdf",
        "page": 12,
        "title": "XR-7200 风控规则概览",
        "summary": "本节描述了 XR-7200 产品的 KYC 和 AML 风控规则...",
        "chunk_id": "abc123..."
      }
    }
  ]
}
```

**Citation 设计的意义**：
- 每条检索结果都携带结构化引用信息（来源文件、页码、标题、摘要）
- 下游 Agent 可以直接引用这些信息构建"回答依据"，增强用户对 AI 输出的信任
- 支持用户溯源查证——点击来源可以定位到原始文档的具体位置

---

### Q25: 下游 Agent 是怎么调用这个知识库的？调用链路是什么样的？

**参考答案：**

**集成方式**：
下游 Agent（如风控规则生成 Agent）在其 MCP Client 配置文件中注册本 Server：
```json
{
  "mcpServers": {
    "knowledge_hub": {
      "command": "python",
      "args": ["-m", "src.mcp_server.server"],
      "cwd": "/path/to/rag-server"
    }
  }
}
```

**完整调用链路**：
```
1. 风控 Agent 判断需要查询产品知识
2. Agent → MCP Client: "请调用 query_knowledge_hub(query='XR-7200 KYC规则')"
3. MCP Client → stdin: {"jsonrpc":"2.0","method":"tools/call","params":{"name":"query_knowledge_hub","arguments":{"query":"XR-7200 KYC规则"}}}
4. MCP Server 接收请求，执行：
   a. QueryProcessor 预处理查询
   b. Dense Retriever 并行查 ChromaDB
   c. Sparse Retriever 并行查 BM25 Index
   d. RRF Fusion 融合两路结果
   e. Reranker 精排 Top-K
   f. Response Builder 构建带 Citation 的结构化响应
5. MCP Server → stdout: {"jsonrpc":"2.0","result":{"content":[{"type":"text","text":"...检索结果..."}]}}
6. MCP Client → Agent: 返回检索结果
7. Agent 基于检索结果 + Citation 生成风控规则
```

**关键设计优势**：
- Agent 只需要知道 Tool 的名称和参数，**完全不需要了解 RAG 内部实现**（不知道有 Hybrid Search、RRF、Reranker 这些概念）
- 更换 RAG 策略（如换 Reranker 模型）不影响下游 Agent 的任何代码
- 支持任何 MCP 兼容的 Agent 框架即插即用

---

## 十二、Skill 驱动全流程开发

### Q26: 什么是 Agent Skill？你们用了哪些 Skill 覆盖开发生命周期？

**参考答案：**

**Agent Skill 的定义**：
Skill 是预定义的**专业化 Agent 能力包**，封装了特定领域的完整工作流——包括操作步骤、工具调用序列、错误处理逻辑和领域知识。与简单的 Prompt 模板不同，Skill 是**可执行的自动化流程**。

**我们使用的核心 Skill 及其覆盖的开发生命周期阶段**：

| Skill | 阶段 | 具体能力 |
|-------|------|---------|
| **auto-coder** | 编码 | 读取 `DEV_SPEC.md` 开发规范，自动识别下一个待实现的任务，同步规范文档为章节参考文件，按照架构约定生成代码，自动运行测试并修复失败（最多 3 轮自动修复），成功后自动 commit |
| **qa-tester** | 测试 | 读取 `QA_TEST_PLAN.md` 测试计划，自动执行所有测试类型（CLI 命令测试、Dashboard UI 测试、MCP 协议端到端测试、Provider 切换测试），诊断失败原因，自动修复并重试（最多 3 轮），记录结果到 `QA_TEST_PROGRESS.md` |
| **setup** | 配置 | 交互式引导用户选择 LLM/Embedding Provider、配置 API Key、安装依赖、生成 `settings.yaml`、启动 Dashboard，如果启动失败自动诊断并修复 |
| **package** | 打包分发 | 清理 `__pycache__`、`.venv`、构建产物、数据缓存、日志文件，脱敏 `settings.yaml` 中的 API Key，生成可分发的干净项目包 |

**Skill 驱动开发的价值**：
1. **标准化**：每个 Skill 封装了最佳实践流程，确保开发过程一致性
2. **自动化**：减少人工重复操作，auto-coder 可以连续自主完成多个开发任务
3. **可复用**：Skill 定义与项目解耦，可以跨项目复用
4. **自愈能力**：qa-tester 和 setup 都有自动诊断和修复机制，减少人工干预

---

## 十三、综合深挖题（跨模块）

### Q27: 从用户发起一次查询到拿到结果，整个链路经过哪些组件？每个环节的延迟大概在什么量级？

**参考答案：**

**完整链路图**：
```
用户/Agent
  │
  ▼
MCP Client ──stdin──▶ MCP Server (protocol_handler.py)
                          │
                          ▼
                     QueryProcessor (query_processor.py)
                     - 查询预处理、标准化
                     - 耗时: ~1ms
                          │
                   ┌──────┴──────┐
                   ▼              ▼
           Dense Retriever   Sparse Retriever
           (dense_retriever)  (sparse_retriever)
           - Query Embedding  - BM25 分词+查索引
           - ChromaDB HNSW    - 评分排序
           - 耗时: 50-100ms   - 耗时: 10-30ms
                   │              │
                   └──────┬──────┘
                          ▼
                     RRF Fusion (fusion.py)
                     - 排名融合
                     - 耗时: ~1ms
                          │
                          ▼
                     Reranker (reranker.py)
                     - Cross-Encoder 精排 (30条)
                     - 耗时: 200-500ms (CPU)
                     - [失败时 Fallback 到 RRF 结果]
                          │
                          ▼
                     Response Builder
                     - 构建 Citation
                     - 格式化输出
                     - 耗时: ~1ms
                          │
                          ▼
MCP Client ◀──stdout── MCP Server
  │
  ▼
用户/Agent 拿到带 Citation 的结构化结果
```

**延迟分布总结**：

| 环节 | 典型延迟 | 占比 | 说明 |
|------|---------|------|------|
| Query 预处理 | ~1ms | <1% | 纯字符串操作 |
| Dense Retrieval | 50–100ms | 15–20% | 包含 1 次 Embedding API 调用 + HNSW 搜索 |
| Sparse Retrieval | 10–30ms | 5% | 纯内存操作（BM25 索引在内存中） |
| RRF Fusion | ~1ms | <1% | 纯排名计算 |
| Reranker | 200–500ms | **60–70%** | **延迟大头**，CPU 上跑 Cross-Encoder 对 30 条候选推理 |
| Response 构建 | ~1ms | <1% | JSON 序列化 |
| **总体 P50** | **300–600ms** | 100% | |

**优化思路**：Reranker 是瓶颈。可选优化：减少精排候选数（30→15）、使用 GPU 加速、或切换为 `reranker: none` 直接用 RRF 结果（延迟降至 ~100ms，精度略降）。

---

### Q28: 如果要支持中英文混合查询，现有架构需要改什么？

**参考答案：**

**逐组件分析改动**：

| 组件 | 当前状态 | 需要的改动 | 难度 |
|------|---------|-----------|------|
| **Dense Embedding** | OpenAI 模型已支持多语言 | 几乎不需要改动，`text-embedding-3-small` 原生支持中英文 | ★☆☆ |
| **BM25 分词器** | 当前可能只支持英文 tokenization | **改动最大**：需要集成中文分词器（如 jieba），且要处理中英混合文本的分词策略（识别中文段用 jieba、英文段用空格分词） | ★★★ |
| **Cross-Encoder** | 英文模型 `ms-marco-MiniLM-L-6-v2` | 需要换成多语言版本（如 `multilingual-e5-large`）或中英双语模型 | ★★☆ |
| **ChunkRefiner Prompt** | 可能是英文 Prompt | 改为中英双语 Prompt 或让 LLM 自适应语言 | ★☆☆ |
| **MetadataEnricher** | Prompt 语言 | 同上 | ★☆☆ |
| **BM25 IDF 统计** | 基于当前语料 | 中文文档加入后需要重建索引，IDF 分布会变化 | ★★☆ |

**关键挑战**：
1. **BM25 中文分词质量**：jieba 的默认词典可能不包含风控领域专有词汇，需要加载自定义词典
2. **中英混合分词**：同一个 query "XR-7200的KYC规则是什么" 包含英文产品代号 + 中文描述，分词器需要正确处理
3. **Cross-Encoder 语言一致性**：英文模型处理中文 query-chunk 对时精度会大幅下降

**可插拔架构的优势**：得益于工厂模式和 Base 抽象类设计，切换 Embedding Provider 和 Reranker 模型只需修改 `settings.yaml` 配置，不需要改业务代码。BM25 分词器需要实现一个新的分词策略并注册。

---

### Q29: 这个系统在什么情况下检索效果最差？你们怎么缓解？

**参考答案：**

**效果差的典型场景及缓解方案**：

| 场景 | 原因分析 | 缓解方案 |
|------|---------|---------|
| **纯专有名词查询**（如"XR-7200 规格"） | Dense Embedding 对罕见专有名词可能编码到错误的语义空间，导致语义漂移 | BM25 路精确匹配补偿——这正是 Hybrid Search 双路设计的核心价值 |
| **超长查询**（>512 tokens） | Embedding 模型有最大输入长度限制，超出部分被截断，丢失关键信息 | QueryProcessor 预处理阶段截断或摘要，或使用支持长上下文的 Embedding 模型 |
| **跨文档推理型问题**（"对比产品 A 和产品 B 的风控差异"） | 答案分散在多个文档中，单次 Top-K 检索难以同时覆盖所有相关 chunk | 当前架构局限——需要引入多轮 RAG（先检索产品 A 再检索产品 B）或知识图谱增强 |
| **文档质量差**（OCR 错误、扫描 PDF 排版混乱） | 垃圾输入导致 chunk 内容不可读，Embedding 无法提取有意义的语义 | ChunkRefiner 去噪 + MetadataEnricher 用 LLM 理解并补充语义摘要 |
| **极短查询**（单个词如"风控"） | 过于宽泛，缺乏上下文，Dense 和 Sparse 都会返回大量泛相关结果 | 可引入 Query Expansion（查询扩展），用 LLM 将短查询扩展为更具体的描述 |
| **语义歧义查询**（"苹果"——公司还是水果？） | Embedding 模型无法判断用户意图 | 可引入 Collection 级别的过滤（用户指定查哪个文档集合），缩小搜索范围消歧 |

**系统层面的缓解设计**：
1. Hybrid Search 本身就是最重要的缓解——双路互补覆盖更多场景
2. Transform 阶段的 ChunkRefiner + MetadataEnricher 在源头提升数据质量
3. 评估体系（Golden Test Set + Ragas 指标）持续监控效果，及时发现退化
4. 可插拔架构允许快速实验不同的 Embedding 模型、Reranker、chunk_size 等配置

---

## 十四、高频追问速查表

| 面试官可能追问 | 核心要点 |
|--------------|---------|
| "RRF 的 k 调大调小有什么影响？" | k 调大→分数分布更均匀，削弱头部优势，趋近等权投票；k 调小→头部文档优势放大，趋近赢者通吃 |
| "Cross-Encoder 用的哪个模型？" | `cross-encoder/ms-marco-MiniLM-L-6-v2`，6 层 MiniLM 架构，轻量适合 CPU 推理，在 MS MARCO 数据集上微调 |
| "BM25 索引存储格式？生产级怎么改？" | 当前用 pickle 序列化，优点是简单快速；缺点是不支持并发读写、文件损坏无法恢复。生产级可迁移至 SQLite 或 Redis |
| "删除文档要操作几个存储？" | 4 个：Chroma（按 metadata.source 删 chunk 向量）+ BM25 Index（移除倒排索引条目）+ image_index.db（删关联图片）+ ingestion_history.db（移除处理记录使文件可重新入库） |
| "测试怎么 mock LLM？" | `unittest.mock.patch` mock LLM 客户端的调用方法，返回预设 JSON 响应，测试关注业务逻辑而非外部依赖 |
| "Trace 数据存哪？为什么不用 LangSmith？" | JSON Lines 格式日志文件，零外部依赖。不用 LangSmith 是因为 Local-First 原则——pip install 即可运行，不需要注册第三方服务 |
| "Cosine Similarity 为什么比欧氏距离更适合？" | 高维空间中欧氏距离趋于"维度灾难"（所有点间距离趋同），Cosine 只看方向不看大小，对高维稠密向量更稳健 |
| "如果 Embedding API 挂了怎么办？" | 查询阶段 Dense 路失败可降级为纯 BM25 检索；Ingestion 阶段则需要等 API 恢复后重试（幂等设计保证安全重试） |
