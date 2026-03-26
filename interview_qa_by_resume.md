# 模拟面试题 — 基于简历 Bullet Points 逐条深挖

> 针对简历中 RAG + MCP 知识检索中枢项目，按关键词逐一设计面试问题与参考答案。
> 面试官视角：从简历描述出发，验证候选人是否真正理解并实现了所述内容。

---

## 一、RAG 数据流水线（整体架构）

### Q1: 简单介绍一下你这条 RAG 数据流水线的整体流程，每个阶段的输入输出是什么？

**参考答案：**

流水线分为 5 个阶段：

| 阶段 | 输入 | 输出 | 关键操作 |
|------|------|------|---------|
| **Load** | 原始 PDF/文档 | Canonical Markdown + metadata | MarkItDown 转换，SHA256 文件级去重 |
| **Split** | Markdown 文本 | Chunk 列表（带 chunk_index, start_offset） | RecursiveCharacterTextSplitter 递归切分 |
| **Transform** | 原始 Chunk | 增强后 Chunk（含 Title/Summary/Tags） | ChunkRefiner + MetadataEnricher + ImageCaptioner |
| **Embed** | 增强 Chunk | Dense 向量 + BM25 稀疏索引 | 双路并行编码，差异计算跳过未变更 chunk |
| **Upsert** | 向量 + 索引 | 持久化存储 | Chroma 向量库 + BM25 索引，幂等写入 |

### Q2: 这条流水线的幂等性是怎么保证的？如果同一个文件被重复提交会发生什么？

**参考答案：**

幂等性通过**两级去重**实现：
1. **文件级去重**：计算文件 SHA256 哈希，查询 `ingestion_history.db`，已处理过的文件直接跳过
2. **Chunk 级幂等**：chunk_id 由 `hash(source_path + section_path + content_hash)` 确定性生成，相同内容永远生成相同 ID，Upsert 操作天然幂等

因此同一文件重复提交时，第一级就会拦截；即使绕过文件级检查，chunk_id 的确定性哈希也保证 Upsert 是覆盖写入而非重复插入。

---

## 二、递归切分（RecursiveCharacterTextSplitter）

### Q3: 递归切分的 "递归" 体现在哪里？它按什么规则拆分？

**参考答案：**

RecursiveCharacterTextSplitter 维护一个**分隔符优先级列表**（如 `\n\n` > `\n` > `. ` > ` ` > `""`），切分时优先用高优先级分隔符（如段落间双换行），如果切出的 chunk 仍超过 `chunk_size`，则递归降级到下一个分隔符继续切分，直到满足大小限制。这样可以尽量保留语义完整性——优先按段落切，段落太长再按句子切，句子太长再按词切。

### Q4: chunk_size 和 chunk_overlap 这两个参数你们是怎么定的？调参依据是什么？

**参考答案：**

- **chunk_size**：通常设为 512–1024 tokens。太小会丢失上下文，太大会引入噪声降低检索精度。我们通过评估指标（Hit Rate@K）在不同 chunk_size 下跑 Golden Test Set 对比选定。
- **chunk_overlap**：通常设为 chunk_size 的 10%–20%（如 chunk_size=1000 时 overlap=200）。作用是让相邻 chunk 有重叠内容，防止关键信息恰好落在切分边界而被两个 chunk 都无法完整覆盖。

---

## 三、Transform 智能增强

### Q5: ChunkRefiner 具体做了什么？为什么不直接调整 Splitter 参数而要用 LLM？

**参考答案：**

ChunkRefiner 让 LLM 做两件事：
1. **合并逻辑相关但被物理切断的段落**（如 "问题描述" 和 "解决方案" 被切到两个 chunk）
2. **去噪**：移除页眉页脚、乱码、无意义的格式字符

为什么不直接调 Splitter 参数：Splitter 是基于字符边界的规则切分，无法理解语义。比如一段连续文本中"问题"和"方案"逻辑上是一个语义单元，但字符数已经超过 chunk_size，Splitter 必须切断。只有 LLM 能判断"这两段应该合并"或"这段乱码应该删除"。

### Q6: 元数据注入增强（MetadataEnricher）产出的 Title/Summary/Tags 在检索时怎么用？

**参考答案：**

MetadataEnricher 调用 LLM 为每个 chunk 生成：
- **Title**：chunk 的简短标题，用于 Dashboard 数据浏览器展示
- **Summary**：chunk 的摘要，可以拼入检索文本提升召回率
- **Tags**：关键词标签，存入 Chroma 的 document metadata，支持 metadata filtering（如按标签过滤特定领域的 chunk）

这些字段存入 chunk 的 metadata，检索时可组合使用：Tags 做过滤缩小搜索范围，Summary 拼接增强语义匹配。

---

## 四、双路 Embedding

### Q7: Dense Embedding 和 Sparse Embedding 分别是什么？为什么要双路并行而不是只用一种？

**参考答案：**

| | Dense Embedding | Sparse Embedding (BM25) |
|---|---|---|
| **原理** | 将文本编码为高维稠密向量（如 1536 维），通过 Cosine Similarity 计算语义相似度 | 基于词频统计（TF-IDF 改进版），构建倒排索引，精确匹配关键词 |
| **优势** | 语义理解强，能处理同义词、模糊表达 | 精确关键词匹配强，对专有名词（产品代号、API 名称）效果好 |
| **劣势** | 对不常见专有名词可能"语义漂移" | 无法理解语义，同义词替换就失效 |

两者互补：Dense 捕捉语义，Sparse 锚定关键词。风控场景中既有语义模糊的描述（"用户异常行为"），也有精确的产品代号和规则编号，单路都有盲区。

### Q8: Dense Embedding 用的是什么模型？向量维度是多少？存在哪里？

**参考答案：**

使用 OpenAI 的 Embedding 模型（如 `text-embedding-ada-002`，1536 维），通过可插拔架构也支持 Azure OpenAI / Ollama 等 Provider。向量存储在 ChromaDB 中，Chroma 内部使用 HNSW 索引实现近似最近邻搜索。

---

## 五、持久化存储

### Q9: 系统用了哪几类存储？各自存什么？为什么不能合并成一个？

**参考答案：**

| 存储 | 路径 | 内容 | 为什么独立 |
|------|------|------|-----------|
| **ChromaDB** | `data/db/chroma/` | Dense 向量 + chunk 文本 + metadata | 向量检索需要专用索引结构（HNSW） |
| **BM25 Index** | `data/db/bm25/` | 稀疏倒排索引 | BM25 基于词频统计，数据结构与向量完全不同 |
| **ingestion_history.db** | `data/db/` | 文件处理记录（SHA256） | 文件级去重，与 chunk 粒度不同 |
| **image_index.db** | `data/db/` | image_id → 文件路径映射 | 图片二进制不适合存入向量库 |

这四类存储服务于不同的数据模型和访问模式，强行合并会导致接口复杂且性能下降。

### Q10: chunk_id 是怎么生成的？为什么不用 UUID？

**参考答案：**

chunk_id = `hash(source_path + section_path + content_hash)`，是**确定性哈希**。

不用 UUID 的原因：UUID 是随机的，同一文件重复处理会产生不同 ID，导致重复 chunk 堆积。确定性哈希保证相同内容永远生成相同 ID，Upsert 时自然覆盖，实现幂等性。

---

## 六、Hybrid Search 混合检索架构

### Q11: Hybrid Search 的执行流程是什么？两路检索是串行还是并行？

**参考答案：**

并行执行：
1. **Dense 路**：Query → Embedding → ChromaDB Cosine Similarity → Top-N 语义候选
2. **Sparse 路**：Query → BM25 倒排索引匹配 → Top-N 关键词候选
3. **RRF 融合**：两路结果按排名融合
4. **Rerank 精排**：融合结果送入 Cross-Encoder/LLM 精排，输出最终 Top-K

两路检索是并行的，互不依赖，融合阶段等两路都返回后再执行。

### Q12: ChromaDB 在语义检索中的角色是什么？它内部用什么索引结构？

**参考答案：**

ChromaDB 是轻量级向量数据库，负责存储 chunk 的 Dense Embedding 向量和 metadata。查询时接收 query 向量，通过内置的 **HNSW（Hierarchical Navigable Small World）** 索引做近似最近邻（ANN）搜索，返回 Top-N 最相似的 chunk。选择 Chroma 是因为 Local-First、零外部依赖、pip install 即可运行。

### Q13: BM25 Index 的关键词检索具体怎么工作的？IDF 怎么算？

**参考答案：**

BM25 基于倒排索引：每个词映射到包含它的文档列表及词频。查询时：
1. 对 query 分词
2. 对每个词查倒排索引，获取匹配文档
3. 按 BM25 公式计算相关性分数：`Score = Σ IDF(t) × (tf × (k1+1)) / (tf + k1 × (1 - b + b × dl/avgdl))`

其中 `IDF(t) = ln((N - df + 0.5) / (df + 0.5) + 1)`，N 是总文档数，df 是包含词 t 的文档数。k1 控制词频饱和度，b 控制文档长度归一化。

---

## 七、RRF 融合

### Q14: RRF 公式是什么？k=60 是怎么来的？为什么不用线性加权？

**参考答案：**

RRF 公式：`Score(d) = Σ 1/(k + rank_i(d))`，对每路检索结果按排名（而非分数）融合。

- **k=60**：来自学术论文（Cormack et al. 2009）的经验推荐值，平滑因子防止排名靠前文档的分数过度高估
- **为什么不用线性加权**：线性加权需要对不同路的分数做归一化（BM25 分数范围和 Cosine Similarity 范围完全不同），且权重比例需要调参。RRF **只看排名不看分数**，天然免去归一化问题，对排名稳健，不依赖各路分数的绝对尺度。

### Q15: 如果两路检索结果完全不重叠，RRF 融合后会是什么效果？

**参考答案：**

每个文档只从一路获得分数贡献（另一路排名视为无穷大，贡献趋近于 0）。效果是两路的 Top 结果交错排列，各自的头部候选都能进入融合结果。这实际上是好事——说明两路在互补地覆盖不同类型的相关文档。

---

## 八、Cross-Encoder / LLM Rerank 精排

### Q16: Cross-Encoder 和 Bi-Encoder 的本质区别是什么？为什么 Cross-Encoder 不能做粗召回？

**参考答案：**

- **Bi-Encoder**（如 Dense Embedding）：Query 和 Document **分别**编码为向量，可以预计算 Document 向量，查询时只需一次向量对比，复杂度 O(1)，适合大规模召回
- **Cross-Encoder**：Query 和 Document **拼接**后一起输入模型，能捕捉交互特征，精度更高，但必须对每对 (Query, Chunk) 实时推理，复杂度 O(n)，不适合大规模召回

所以架构是两阶段：Bi-Encoder 粗召回（快但粗） → Cross-Encoder 精排（慢但准），精排只处理 10–30 条候选。

### Q17: LLM Rerank 相比 Cross-Encoder 有什么优劣？什么场景下你会选 LLM Rerank？

**参考答案：**

| | Cross-Encoder | LLM Rerank |
|---|---|---|
| **速度** | 较快（轻量模型） | 较慢（调用大模型） |
| **成本** | 低（本地推理） | 高（API 调用费用） |
| **精度** | 在特定领域微调后很强 | 通用理解能力强，Zero-shot 效果好 |
| **可解释性** | 只输出分数 | 可以输出排序理由 |

选 LLM Rerank 的场景：候选文档需要复杂推理才能判断相关性（如需要理解上下文逻辑），或者没有领域微调过的 Cross-Encoder 模型时。

### Q18: 如果 Reranker 服务挂了，系统怎么处理？

**参考答案：**

系统实现了 **Graceful Fallback**：精排失败或超时时自动回退到 RRF 融合结果直接返回。这保证了系统可用性——Reranker 是锦上添花的精排层，不是必要链路，降级后检索质量略降但服务不中断。

---

## 九、检索准确率 Hit Rate@10 达 92%

### Q19: Hit Rate@10 是怎么定义和计算的？

**参考答案：**

Hit Rate@K = 在 Top-K 检索结果中，**至少有一条命中 Golden Answer 的查询占总查询数的比例**。

例如 100 条测试 query，其中 92 条的 Top-10 结果包含了标注的正确答案，则 Hit Rate@10 = 92%。

### Q20: 这个 92% 是怎么测的？测试集是什么？基线是什么？

**参考答案：**

- **测试集**：Golden Test Set（`tests/fixtures/golden_test_set.json`），包含 query + 标注的正确答案 chunk
- **评估流程**：EvalRunner 遍历测试集，对每条 query 执行完整 Hybrid Search + Rerank 流程，检查 Top-10 是否命中
- **基线对比**：纯 BM25 / 纯 Dense / Hybrid 无 Rerank 分别跑同一测试集对比，Hybrid + Rerank 的 92% 显著优于单路检索
- **其他指标**：同时关注 MRR（Mean Reciprocal Rank）衡量头部排序质量

### Q21: 如果一次策略变更（比如换了 Reranker 模型）导致指标下降，你们怎么处理？

**参考答案：**

每次策略变更后必跑 Golden Test Set 回归评估，对比变更前后的 Hit Rate@K 和 MRR。如果指标下降，分析原因：是特定类型 query 退化还是整体退化，然后针对性调整。评估体系支持 Ragas 指标（Faithfulness、Answer Relevancy、Context Precision）做多维度分析。

---

## 十、MCP 标准（Model Context Protocol）

### Q22: MCP 是什么？和普通 REST API 有什么区别？

**参考答案：**

MCP（Model Context Protocol）是专为 AI Agent 设计的上下文协议，基于 **JSON-RPC 2.0**，定义了标准的 `tools`/`resources`/`prompts` 接口。

与 REST API 的区别：
- REST API 需要每个客户端单独适配接口，MCP 是标准协议，任何符合规范的 MCP Client（Claude Desktop、Copilot 等）都能即插即用
- MCP 通过协议标准化消除了集成成本，类似于 USB 标准化了外设接口
- MCP 使用 Stdio Transport（stdin/stdout 通信），零网络依赖，天然适合私有知识库场景

### Q23: Stdio Transport 是怎么工作的？为什么选这种传输方式？

**参考答案：**

Client 以**子进程方式**启动 MCP Server，双方通过 stdin/stdout 传输 JSON-RPC 消息，日志走 stderr 避免污染协议通道。

选择 Stdio 的原因：零网络依赖、无需端口/鉴权配置，天然适合本地私有知识库场景。局限是不支持远程调用和多客户端并发，如需远程访问需切换到 HTTP+SSE Transport。

---

## 十一、Tool Calling 模式嵌入

### Q24: 你们暴露了哪些 MCP Tools？主检索入口的输入输出是什么？

**参考答案：**

三个核心 Tools：

| Tool | 功能 | 关键参数 |
|------|------|---------|
| `query_knowledge_hub` | 主检索入口（Hybrid Search + Rerank） | `query`（必填），`top_k`、`collection`（可选） |
| `list_collections` | 列举可用文档集合 | 无 |
| `get_document_summary` | 获取文档摘要与元信息 | `doc_id` |

`query_knowledge_hub` 返回结构化结果，每条包含 chunk 内容 + **Citation（来源文件名、页码、chunk 摘要）**，方便下游 Agent 展示"回答依据"。

### Q25: 下游 Agent 是怎么调用这个知识库的？调用链路是什么样的？

**参考答案：**

下游 Agent（如风控规则生成 Agent）在其 MCP Client 配置中注册本 Server。当 Agent 需要检索产品知识时：
1. Agent 发起 Tool Call：`query_knowledge_hub(query="XX产品的风控规则")`
2. MCP Client 通过 Stdio 将 JSON-RPC 请求发送给 Server 子进程
3. Server 执行完整 Hybrid Search + Rerank 流程
4. 返回带 Citation 的结构化结果
5. Agent 基于检索结果生成风控规则

全程通过标准 MCP 协议通信，Agent 无需了解 RAG 内部实现细节。

---

## 十二、Skill 驱动全流程开发

### Q26: 什么是 Agent Skill？你们用了哪些 Skill 覆盖开发生命周期？

**参考答案：**

Agent Skill 是预定义的专业化 Agent 能力包，封装了特定领域的工作流。我们用 Skill 驱动开发的完整生命周期：

| Skill | 覆盖阶段 | 作用 |
|-------|---------|------|
| **auto-coder** | 编码 | 读取 DEV_SPEC.md 开发规范，自动识别待实现任务，生成代码并自动修复测试 |
| **qa-tester** | 测试 | 自动执行 QA 测试计划，诊断失败并重试修复 |
| **setup** | 配置 | 交互式引导 Provider 选择、API Key 配置、依赖安装 |
| **package** | 打包 | 清理缓存、脱敏 API Key、生成可分发的项目包 |

这种模式的价值是将重复性的开发流程标准化、自动化，每个 Skill 都是可复用的工作流封装。

---

## 十三、综合深挖题（跨模块）

### Q27: 从用户发起一次查询到拿到结果，整个链路经过哪些组件？每个环节的延迟大概在什么量级？

**参考答案：**

```
Query → QueryProcessor(预处理) → Dense Retriever(向量召回) ──┐
                                → Sparse Retriever(BM25召回) ─┤
                                                              ├→ RRF Fusion → Reranker → Response Builder → MCP 返回
```

延迟分布（典型值）：
- Dense Retrieval：~50–100ms（Chroma HNSW 查询）
- Sparse Retrieval：~10–30ms（BM25 内存索引）
- RRF Fusion：~1ms（纯排名计算）
- Reranker（Cross-Encoder）：~200–500ms（CPU，30 条候选）
- 总体 P50：~300–600ms

Reranker 是延迟大头，这也是为什么设置超时回退的原因。

### Q28: 如果要支持中英文混合查询，现有架构需要改什么？

**参考答案：**

主要改动点：
1. **BM25 分词器**：需要同时支持中文分词（如 jieba）和英文 tokenization，或使用多语言分词器
2. **Embedding 模型**：选用多语言模型（如 `text-embedding-3-small` 已支持多语言）或中英双语模型
3. **Cross-Encoder**：需要选用多语言版本，避免英文模型处理中文时精度下降

Dense 路对多语言天然友好（Embedding 模型本身支持即可），Sparse 路（BM25）改动最大，因为分词质量直接影响倒排索引准确性。

### Q29: 这个系统在什么情况下检索效果最差？你们怎么缓解？

**参考答案：**

效果差的场景：
1. **纯专有名词查询**（如产品代号）：Dense 语义匹配可能漂移 → BM25 路互补
2. **超长查询**（>512 tokens）：Embedding 模型截断风险 → Query 预处理截断/摘要
3. **跨文档推理型问题**（答案分散在多个文档）：单次检索无法覆盖 → 需要多轮 RAG 或知识图谱增强
4. **文档质量差**（扫描 PDF、OCR 错误多）：ChunkRefiner 去噪 + MetadataEnricher 补充语义

Hybrid Search 本身就是对场景 1 的缓解方案，双路互补覆盖更多 case。

---

## 十四、高频追问速查表

| 面试官可能追问 | 核心要点 |
|--------------|---------|
| "RRF 的 k 调大调小有什么影响？" | k 调大→分数分布更均匀，削弱头部优势；k 调小→头部文档优势更大 |
| "Cross-Encoder 用的哪个模型？" | `cross-encoder/ms-marco-MiniLM-L-6-v2`，轻量适合 CPU 推理 |
| "BM25 索引存储格式？" | 当前用 pickle 序列化存储，生产级可迁移至 SQLite |
| "删除文档要操作几个存储？" | 4 个：Chroma + BM25 Index + image_index.db + ingestion_history.db |
| "测试怎么 mock LLM？" | `unittest.mock.patch` mock LLM 客户端，返回预设响应 |
| "Trace 数据存哪？" | JSON Lines 格式日志，零外部依赖 |
| "为什么不用 LangSmith？" | Local-First 原则，零外部依赖，pip install 即可运行 |
