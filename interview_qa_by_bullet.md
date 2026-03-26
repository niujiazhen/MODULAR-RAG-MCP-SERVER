# 模拟面试题（按 Bullet Point 综合提问版）

> 按简历中的每一条 bullet point 原文出发，模拟面试官的综合性提问。
> 每条 bullet point 下设 3–5 道综合题，不局限于单一关键词，而是围绕该条描述整体展开。

---

## Bullet 1：设计 RAG 数据流水线：递归切分→Transform 智能增强→双路 Embedding→持久化存储

### Q1: 介绍一下你设计的这条数据流水线，从原始文档到可检索的知识库，经历了哪些阶段？每个阶段解决什么问题？

**参考答案：**

整条流水线分 5 个阶段：Load → Split → Transform → Embed → Upsert。

- **Load**：用 MarkItDown 将 PDF 转为标准 Markdown，同时提取 metadata（来源路径、文档类型、标题等）。入口处做 **SHA256 文件级去重**——计算文件哈希查 `ingestion_history.db`，重复文件直接跳过，避免浪费后续所有阶段的计算资源。
- **Split**：使用 LangChain 的 `RecursiveCharacterTextSplitter`，按 Markdown 结构层级（标题 > 段落 > 句子 > 字符）递归切分。核心是**语义优先**——优先在标题/段落边界切，尽量保持每个 chunk 是一个完整的语义单元。
- **Transform**：三步 LLM 增强——ChunkRefiner 合并被切断的语义段落并去噪；MetadataEnricher 为每个 chunk 生成 Title/Summary/Tags 注入 metadata；ImageCaptioner 为图片生成文字描述缝入正文。
- **Embed**：双路并行编码——Dense 路调 Embedding API 生成稠密向量（用于语义检索），Sparse 路更新 BM25 倒排索引（用于关键词检索）。关键优化是**差异计算**——只对内容有变更的 chunk 调用 API。
- **Upsert**：幂等写入 ChromaDB 和 BM25 索引。chunk_id 由确定性哈希生成，保证重复执行不产生副作用。

**每个阶段解决的核心问题**：Load 解决格式统一和去重；Split 解决长文档无法直接检索的问题；Transform 解决切分质量和元数据缺失问题；Embed 解决将文本转化为可计算相似度的表示；Upsert 解决持久化和幂等性。

### Q2: 这条流水线里你觉得最有技术含量的设计是哪个环节？为什么？

**参考答案：**

我认为是 **Transform 阶段**，特别是 ChunkRefiner 的设计。原因有三：

1. **解决了规则切分的根本局限**：RecursiveCharacterTextSplitter 是业界标准的切分工具，但它本质是基于字符边界的规则操作，无法理解语义。当"问题描述"和"解决方案"被切到两个 chunk 时，两个 chunk 各自都不是完整的语义单元，检索时可能命中了"问题"但看不到"方案"。ChunkRefiner 用 LLM 理解语义，将这类碎片重组为完整的知识单元。

2. **投入产出比极高**：Transform 是离线处理（Ingestion 时一次性运行），增加的是入库时间和 LLM 成本。但换来的是检索阶段**永久的质量提升**——每次查询都受益于更高质量的 chunk。这个 ROI 是非常划算的。

3. **MetadataEnricher 打通了结构化检索能力**：为 chunk 注入的 Tags 支持 metadata filtering，使得检索可以先按标签过滤再做语义匹配，大幅提升了精准度。没有这一步，系统只能做"全量语义搜索"，无法利用结构化信息缩小搜索范围。

### Q3: 如果让你重新设计这条流水线，你会改什么？有没有你觉得可以简化的地方？

**参考答案：**

**可以改进的地方**：
1. **Transform 阶段的三步可以考虑合并**：目前 ChunkRefiner、MetadataEnricher、ImageCaptioner 串行执行，每步都调一次 LLM。可以设计一个 unified prompt，让 LLM 一次性完成去噪+元数据生成，减少 LLM 调用次数和延迟。但 trade-off 是 prompt 更复杂，可能影响各子任务的质量。
2. **BM25 索引存储可以升级**：当前用 pickle 序列化，不支持并发读写，生产环境下可以换成 SQLite 或 Redis，获得更好的可靠性和并发性能。
3. **增量更新可以更智能**：目前文件级去重是"全有或全无"——文件改了一行就要重走整条流水线。可以实现 chunk 级增量：对比新旧 chunk 列表，只处理新增/变更的 chunk，删除已不存在的 chunk。

**不会简化的地方**：
- 双路 Embedding 不能省——这是 Hybrid Search 的基础
- ChunkRefiner 不能省——它对检索质量的提升是实质性的
- 确定性 chunk_id 不能省——这是幂等性的基石

### Q4: 你提到了"持久化存储"，系统用了多种存储各司其职。如果删除一个文档，需要操作几个存储？漏删其中一个会怎样？

**参考答案：**

删除一个文档需要**协调操作 4 个存储**：

1. **ChromaDB**：按 `metadata.source` 匹配该文档的所有 chunk 向量并删除
2. **BM25 Index**：从倒排索引中移除该文档所有 chunk 的索引条目
3. **image_index.db**：删除该文档关联的图片映射记录和图片文件
4. **ingestion_history.db**：移除该文件的处理记录，使其可以被重新入库

**漏删的后果**：
- 漏删 ChromaDB：Dense 路还会召回已删除文档的 chunk，返回过时/错误信息
- 漏删 BM25：Sparse 路还会匹配已删除文档的关键词，但在 ChromaDB 中找不到对应 chunk，造成数据不一致
- 漏删 image_index.db：图片文件残留占用磁盘空间，且可能被错误关联
- 漏删 ingestion_history.db：重新上传同一文件时被认为"已处理"而跳过，无法重新入库

**当前实现**：采用**尽力删除策略**——各存储独立尝试删除，某个失败时记录错误但不阻塞其他存储的删除。生产级改进可引入事务记录或两阶段提交确保原子性。

---

## Bullet 2：Transform 智能增强——ChunkRefiner 重组 + 元数据注入增强

### Q5: ChunkRefiner 具体解决了什么问题？能举一个实际的例子吗？

**参考答案：**

ChunkRefiner 解决的核心问题是**物理切分破坏语义完整性**。

**实际例子**：假设一份风控文档中有这样的内容：

```
## 3.2 XR-7200 异常交易处理
问题：用户在非工作时间段连续发起大额交易，触发频率异常告警。
─── [chunk 切分点] ───
解决方案：系统自动冻结账户并发送短信通知，同时生成人工复核工单。
复核标准：单笔 > 50万 或 24小时内累计 > 200万。
```

RecursiveCharacterTextSplitter 在字符数达到 chunk_size 时切断，"问题"和"解决方案"被分到两个 chunk。检索"XR-7200 异常交易怎么处理"时：
- 可能命中"问题"chunk（有"异常交易"关键词），但看不到解决方案
- 或者命中"解决方案"chunk（有"处理"关键词），但缺少问题上下文

ChunkRefiner 让 LLM 审视前后 chunk 的片段，判断"这两段是一个问答对，应该合并为一个 chunk"，同时去除切分点附近可能残留的页码"第 12 页"之类的噪声。

### Q6: MetadataEnricher 生成的 Tags 在检索时是怎么用的？你觉得 Tags 和直接做全文语义搜索相比，优势在哪？

**参考答案：**

**Tags 的使用方式**：
存入 Chroma 的 document metadata 后，检索时可以通过 Chroma 的 `where` 过滤条件做**预过滤**：
```python
results = collection.query(
    query_embeddings=[query_vector],
    where={"tags": {"$contains": "风控"}},
    n_results=10
)
```
先缩小搜索范围到包含"风控"标签的 chunk，再在子集上做向量相似度计算。

**相比全文语义搜索的优势**：

1. **精准度提升**：全文搜索在 10 万 chunk 中找 Top-10，Tags 过滤后可能只在 2000 个相关 chunk 中找 Top-10，噪声大幅减少
2. **性能提升**：HNSW 搜索空间减小，查询更快
3. **可控性**：业务层可以传入 collection 或 tag 约束，精确控制检索范围（如"只搜 KYC 相关文档"），这是纯语义搜索无法做到的
4. **可解释性**：Tags 是人类可读的标签，方便 Dashboard 展示和调试

**Trade-off**：Tags 质量依赖 LLM 生成的准确性，如果 LLM 标注了错误的 Tag，反而会导致漏检。所以 Tags 过滤是**可选的增强手段**，不是必须使用的。

---

## Bullet 3：双路 Embedding——Dense 语义召回 + BM25 Sparse 关键词匹配

### Q7: 能用一个实际的查询例子说明为什么必须要双路？单路在什么情况下会失败？

**参考答案：**

**例子 1：Dense 路失败的场景**

用户查询："XR-7200 的 KYC 规则编号"

- "XR-7200" 是产品代号，Embedding 模型训练时大概率没见过，会将其映射到语义空间中的某个"不确定"区域
- Dense 检索可能返回其他产品的 KYC 规则（语义上"KYC 规则"很相似），但不是 XR-7200 的
- **BM25 可以精确匹配**"XR-7200"这个字符串，直接命中正确文档

**例子 2：Sparse 路失败的场景**

用户查询："用户异常行为的处理方案"

- 文档中实际写的是"客户违规操作的应对措施"
- BM25 逐词匹配：query 中没有"客户""违规""应对"这些词，BM25 完全无法匹配
- **Dense Embedding 理解语义**："异常行为"≈"违规操作"，"处理方案"≈"应对措施"，可以正确召回

这两个例子说明了**双路的必要性**：Dense 擅长语义，Sparse 擅长精确匹配，任何单路方案都会在另一种查询类型上失败。Hybrid Search 让两种能力互补。

### Q8: 双路 Embedding 的计算成本是单路的两倍吗？你们怎么控制 Ingestion 成本？

**参考答案：**

**不完全是两倍**，原因和成本控制策略：

1. **Embedding 成本（Dense 路）**：这是主要成本来源，调用外部 Embedding API 按 token 计费。**差异计算**是关键优化——通过比对 chunk 的 content_hash，只对内容有变更的 chunk 调用 API，未变更的复用已有向量。增量更新时可以节省 80%+ 的 API 调用。

2. **BM25 索引构建（Sparse 路）**：成本**几乎为零**。BM25 是本地计算（分词 + 构建倒排索引），不需要调用任何外部 API，纯 CPU 操作，千条 chunk 在毫秒级完成。

3. **LLM 增强成本（Transform 阶段）**：ChunkRefiner 和 MetadataEnricher 每个 chunk 各调一次 LLM，这其实是比 Embedding 更大的成本来源。但这是一次性的离线处理。

**总结**：双路的增量成本主要来自 Dense Embedding API 调用（Sparse 路几乎免费）。差异计算机制确保只为"新内容"付费，大幅降低了增量更新的成本。

### Q9: Dense Embedding 向量存在 ChromaDB 里，BM25 索引存在文件系统中。你有没有考虑过统一存储？为什么没这么做？

**参考答案：**

考虑过但最终没有统一，原因是**数据结构和查询算法完全不同**：

- **ChromaDB 存储的是稠密向量**（1536 维浮点数组），查询算法是 HNSW 近似最近邻搜索，时间复杂度 O(log n)
- **BM25 存储的是倒排索引**（词 → 文档列表 + 词频），查询算法是词条匹配 + 评分计算

如果强行统一：
- 把倒排索引存入 ChromaDB：Chroma 不支持倒排索引查询，需要自己在 metadata 上模拟，性能会很差
- 把向量存入 BM25 索引文件：文件系统不支持 ANN 搜索，只能暴力扫描
- 使用 Elasticsearch 统一：ES 确实同时支持 BM25 和向量检索，但引入了重量级外部依赖，违背 Local-First 原则

**当前方案的优势**：各存储各司其职，各自使用最适合自己数据模型的查询算法，性能最优。代价是删除文档时需要协调多存储，但这个复杂度通过 DocumentManager 统一封装。

---

## Bullet 4：Hybrid Search + RRF 融合 + Rerank 精排，Hit Rate@10 达 92%

### Q10: 完整描述一下从用户查询到返回 Top-K 结果的检索流程。如果中间某个环节挂了，系统会怎么表现？

**参考答案：**

**完整流程**：
1. QueryProcessor 对查询做预处理（标准化、去噪）
2. Dense Retriever 和 Sparse Retriever **并行执行**，各自返回 Top-N 候选
3. RRF Fusion 按 `Score(d) = Σ 1/(60 + rank_i(d))` 融合两路排名，输出统一排序列表
4. Reranker（Cross-Encoder/LLM）对融合结果的 Top-M 候选精排，输出最终 Top-K

**各环节故障的系统表现**：

| 故障环节 | 系统行为 | 用户感知 |
|---------|---------|---------|
| Dense Retriever 失败（Embedding API 不可用） | 降级为纯 BM25 检索结果 | 语义类查询效果下降，关键词类查询不受影响 |
| Sparse Retriever 失败（BM25 索引损坏） | 降级为纯 Dense 检索结果 | 精确关键词匹配能力丧失 |
| RRF Fusion | 不太可能失败（纯内存计算） | — |
| Reranker 超时/异常 | **Graceful Fallback**：直接返回 RRF 融合结果 | 检索结果质量略降但服务不中断 |
| MCP Server 进程崩溃 | Client 收到进程退出通知 | 需要重启 Server |

关键设计思想：**核心链路（BM25 + Dense 至少一路可用）保证基本可用，精排层是可降级的增强层**。

### Q11: 92% 的 Hit Rate@10 是一个好指标吗？你怎么判断这个数字够不够用？如果要提升到 95% 你会怎么做？

**参考答案：**

**92% 够不够用取决于业务场景**：
- 在风控知识检索场景中，92% 意味着每 100 次查询只有 8 次在 Top-10 中找不到正确答案。考虑到下游 Agent 还会基于检索结果做推理和生成，这个召回率是可接受的
- 但如果是医疗或法律场景，漏检可能造成严重后果，可能需要更高的阈值

**判断够不够用的方法**：
1. 看 MRR——92% 的 Hit Rate 如果 MRR 只有 0.3（正确结果平均排在第 3-4 名），说明排序还有优化空间
2. 看 Ragas Faithfulness——即使检索命中了，如果下游生成的答案"幻觉"严重，检索的价值就被打折
3. 看业务反馈——风控团队实际使用时是否频繁遇到检索不到的情况

**从 92% 提升到 95% 的可能方向**：

| 方向 | 预期收益 | 成本 |
|------|---------|------|
| 优化 chunk_size | 中等——找到当前数据集的最优切分粒度 | 低，只需跑评估实验 |
| 换更强的 Embedding 模型（如 text-embedding-3-large） | 中等——更好的语义理解 | API 成本增加 |
| Query Expansion（LLM 扩展查询） | 较高——覆盖更多语义变体 | 增加查询延迟和 LLM 成本 |
| 增加 Golden Test Set 覆盖度 | 间接——更准确地度量真实水平 | 人工标注成本 |
| 换更强的 Reranker | 中等——更精准的重排 | 可能增加延迟 |

分析当前 8% 失败 case 的共性（是什么类型的 query 失败？）是最重要的第一步，对症下药比盲目优化有效。

### Q12: RRF 融合你们有没有考虑过调权重？比如某些场景下 Dense 更重要，某些场景下 BM25 更重要？

**参考答案：**

**RRF 的设计哲学就是"无权重"**——它只看排名不看分数，两路天然等权。这既是优势也是局限：

**优势**：无需调参，不依赖各路分数的绝对尺度，鲁棒性好。

**局限**：确实无法根据查询类型动态调整两路的权重。如果已知某类查询 Dense 更准，没有办法让 Dense 路的贡献更大。

**如果要引入权重，可能的方案**：
1. **Weighted RRF**：`Score = α/(k + rank_dense) + (1-α)/(k + rank_sparse)`，但 α 如何根据查询动态选择是个难题
2. **Query Router**：用一个小模型判断查询类型（语义型 vs 关键词型），然后决定只走 Dense、只走 BM25 还是 Hybrid
3. **Learned Fusion**：训练一个轻量模型学习最优的融合策略，但需要大量标注数据

**我们的决策**：目前 RRF 等权已经达到了 92% 的 Hit Rate，投入产出比很好。引入动态权重会增加系统复杂度，且收益不确定。在当前阶段保持简单是正确的选择，等遇到瓶颈再考虑。

---

## Bullet 5：基于 MCP 标准构建知识检索 Agent Server，支持 Tool Calling 嵌入风控流水线

### Q13: 为什么选 MCP 而不是直接提供 REST API 给下游 Agent 调用？

**参考答案：**

**核心差异**：REST API 需要下游每个 Agent 框架单独写适配代码（HTTP 调用、响应解析、错误处理、认证），而 MCP 是**标准化协议**，任何 MCP Client 即插即用。

**类比**：REST API 像是每个外设自带独特的接口线（需要找适配器），MCP 像是 USB-C 标准（一根线通用）。

**具体优势**：
1. **零适配成本**：Claude Desktop、VS Code Copilot、Cursor 等 MCP Client 只需一行配置就能接入，不需要写任何适配代码
2. **自动发现**：MCP Client 通过 `tools/list` 接口自动发现 Server 提供的所有工具及其参数 schema，Agent 可以自主决定什么时候调用什么工具
3. **协议级 Citation**：返回结果可以携带结构化的引用信息（来源文件、页码），Client 侧有标准的展示方式
4. **生态兼容**：随着 MCP 生态扩大，新的 Agent 框架天然兼容，不需要我们额外做集成工作

**REST API 更合适的场景**：如果消费方不是 AI Agent 而是传统 Web 应用（如前端页面直接调用），REST API 更直接。但在我们的场景中，消费方是风控流水线中的 Agent，MCP 是更自然的选择。

### Q14: 你的 MCP Server 暴露了三个 Tool，它们之间是什么关系？下游 Agent 一般怎么组合使用？

**参考答案：**

**三个 Tool 的关系**：

```
list_collections ──▶ "有哪些知识库可查？"
                          │
                          ▼
query_knowledge_hub ──▶ "查询具体知识"（可指定 collection）
                          │
                          ▼
get_document_summary ──▶ "了解某文档的详细信息"
```

**典型使用流程**：

1. Agent 首次接入时调 `list_collections` 了解可用的文档集合（如"风控规则集""产品说明集"等）
2. Agent 根据用户意图调 `query_knowledge_hub` 做知识检索，可选地指定 collection 缩小范围
3. 如果需要了解某个文档的整体信息（如文档包含多少内容、什么时候入库），调 `get_document_summary`

**实际使用中**，90% 的调用都是 `query_knowledge_hub`。`list_collections` 通常在初始化阶段调一次缓存结果，`get_document_summary` 在需要展示文档概览时使用。

### Q15: Stdio Transport 有哪些局限？如果未来风控流水线要部署为微服务架构，你会怎么改？

**参考答案：**

**Stdio 的局限**：
1. **单机限制**：Client 和 Server 必须在同一台机器上（通过进程管道通信）
2. **单客户端**：一个 Server 进程只能服务一个 Client
3. **无负载均衡**：无法多实例部署做水平扩展
4. **无持久连接管理**：Client 退出时 Server 随之终止

**微服务架构改造方案**：

1. **传输层切换**：MCP 标准也定义了 **HTTP+SSE Transport**，将 Stdio 替换为 HTTP Transport 即可支持远程调用和多客户端并发。Tool 的实现代码（Hybrid Search、Reranker 等）**完全不需要改**，只改传输层。

2. **部署架构**：
```
Load Balancer
   ├── MCP Server Instance 1 (HTTP+SSE)
   ├── MCP Server Instance 2 (HTTP+SSE)
   └── MCP Server Instance 3 (HTTP+SSE)
        │
        ├── 共享 ChromaDB（独立部署或换成 Qdrant Cloud）
        ├── 共享 BM25 Index（换成 Redis 或 Elasticsearch）
        └── 共享 SQLite → PostgreSQL
```

3. **关键改动点**：
   - 存储层从 Local-First 改为共享存储（最大的改动）
   - BM25 从 pickle 文件换成支持并发访问的存储
   - 添加认证鉴权层（Stdio 不需要，HTTP 必须有）

得益于可插拔架构（Base 抽象类 + Factory 模式），更换存储后端只需新增 Provider 实现并修改配置，不需要改业务逻辑代码。

---

## Bullet 6：基于 Skill 驱动全流程开发

### Q16: Agent Skill 和普通的脚本/自动化工具（如 Makefile、CI Pipeline）有什么本质区别？

**参考答案：**

**核心区别在于"理解力"和"自愈能力"**：

| 维度 | 脚本/Makefile/CI | Agent Skill |
|------|-----------------|-------------|
| **执行方式** | 按固定步骤顺序执行 | AI Agent 理解上下文后自主决策执行路径 |
| **错误处理** | 预定义的 if-else 分支 | LLM 理解错误原因，自主生成修复方案 |
| **适应性** | 环境变化需要手动修改脚本 | 能适应未预见的情况（如新版本 API 变更） |
| **上下文理解** | 不理解代码含义 | 读懂代码、文档和错误日志的语义 |

**实际例子**：
- **qa-tester Skill**：如果某个测试失败，它不是简单地"重试 3 次"——它会读取错误日志，理解失败原因（如"配置文件路径错误"），修改代码或配置，然后重新运行测试。这种"诊断→修复→验证"的循环是脚本无法做到的。
- **setup Skill**：如果用户选择了一个尚未实现的 Provider（如 Gemini），它不会简单报错，而是按照可插拔架构的模式**自动生成 Provider 代码骨架**，然后继续配置流程。

### Q17: Skill 驱动开发这件事本身是如何体现工程能力的？面试官可能会质疑"这不就是用 AI 写代码吗？"

**参考答案：**

这是一个很好的问题。关键区别在于：**使用 AI 工具**和**设计 AI 驱动的开发体系**是完全不同的事情。

**工程能力的体现**：

1. **架构设计先行**：Skill 能高效工作的前提是项目有清晰的架构规范（`DEV_SPEC.md`）、可插拔的模块设计、完善的测试体系。这些是人设计的，不是 AI 生成的。auto-coder 能自动实现功能，是因为架构足够清晰，不是因为 AI 万能。

2. **Skill 本身的设计**：每个 Skill 的工作流（读取什么文件、执行什么步骤、如何处理失败、如何持久化进度）都需要精心设计。这是**元层面的工程——设计让 AI 高效工作的系统**。

3. **质量保障体系**：AI 生成的代码需要 qa-tester 验证，qa-tester 运行的测试用例（`QA_TEST_PLAN.md`）是人设计的。这是"人定义标准，AI 执行实现，人验证结果"的协作模式。

4. **可复用性设计**：Skill 与项目解耦，可以复用到其他项目。这种"将最佳实践封装为可复用能力包"的思维方式本身就是高级工程能力。

**直接回应质疑**：不是"用 AI 写代码"，而是"设计了一套让 AI 高效参与开发全生命周期的工程体系"。这需要深入理解软件工程原则、AI 的能力边界、以及两者如何最优配合。
