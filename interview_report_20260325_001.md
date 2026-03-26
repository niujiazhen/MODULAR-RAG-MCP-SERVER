# 模拟面试报告

**项目**：Modular RAG MCP Server
**面试时间**：2026-03-25
**面试官风格**：🎯 速攻广度型 (FAST)
**掷骰结果**：4
**评分**：8.0 / 10

---

## 一、面试记录

> ✅ 答对核心要点 | ⚠️ 方向正确但细节缺失 | ❌ 未能答出或方向错误

### 方向 1：项目综述

| 轮次 | 问题 | 候选人回答（原文） | 评估 | 参考答案 |
|-----|------|------------------|------|---------|
| Q1 | 这个系统里有哪几类存储？它们各自负责什么？为什么不能只用一个？ | 系统里存储类型如下：1. ChromaDB向量数据库：用于语义检索，存储Dense Embedding向量+原文+元数据，负责语义相似度的召回，底层采用的是HNSW索引做近似最近邻搜索。2. BM25 Index采用Json文件：用于关键词检索，存倒排索引term → [chunk_id, tf, doc_len] + 全局 IDF 统计。负责关键词精确匹配召回。用 JSON 是因为倒排索引本质是嵌套字典结构，而且 BM25 打分需要全量加载计算 IDF，不需要随机访问。其余还包含SQLite下的Image Storage和FileIntegrityChecker，不过这不在我专门为面试准备的项目里，就不提及了。不能用一个存储的原因是：ChromaDB查询模式是向量相似度top-k检索，需要专用索引HNSW，关系型DB做不了。BM25 Json查询模式是全量加载和词频打分，需要全局IDF统计，必须整体加载，向量DB无法做到精准关键词匹配。 | ✅ | [→ 查看](#a-存储架构) |

### 方向 2：简历深挖

| 轮次 | 问题 | 候选人回答（原文） | 评估 | 露馅 | 参考答案 |
|-----|------|------------------|------|-----|---------|
| Q2 | 你简历里写了"实现 ChunkRefiner 重组+元数据注入增强，显著降低冗余干扰"——这个 ChunkRefiner 的 Prompt 是你自己写的吗？迭代了几版？最终定稿的依据是什么？ | ChunkRefiner的Prompt实现逻辑是这样：第一版：在Prompt里写让LLM把文本里的噪声（包含一些页眉页脚，乱码，多余空行）等清理掉，但实际调用出现如下问题：1. LLM过度总结，导致500字chunk只剩下200字，信息丢失过多。2. LLM会自动加入markdown格式的符号（例如##，\*\*等），导致后续Embedding的时候格式符号又变成噪声。第二版：针对上述问题，在Prompt模版中加入负面约束Do Not，提示LLM不要做summary以及添加任何markdown format。第三版：还未实现，但我在考虑使用few shot的方式，给LLM一组"噪声原文->理想输出"的示例，让LLM更精准的理解清理但不压缩的边界，目前全靠Prompt的instruction，效果不稳定。定稿依据是跑了对比试验，主要参考如下两个因素：1. 长度保持率：Refined chunk 与原文长度比应在 85%-105% 之间，低于说明过度压缩。2. 下游检索命中率：对比 Refine 前后的 Hit Rate@10，Refine 后从约 85% 提升到 92%，说明降噪有效。这个Chunk Refiner还采用了优雅降级的设计策略，由于LLM的调用有随机性，也有失败率，所以设计了一个基础的rule-based的本地清理模式，采用正则表达式清理，0成本；LLM增强作为可选选项，若执行失败，则直接返回rule-based清理的内容，也保证了当LLM调用失败的时候，pipeline不会挂。 | ✅ | 否 | [→ 查看](#a-chunkrefiner) |

### 方向 3：技术深挖

| 轮次 | 问题 | 候选人回答（原文） | 评估 | 参考答案 |
|-----|------|------------------|------|---------|
| Q3 | Cross-Encoder 和 Bi-Encoder 的本质区别是什么？为什么 Cross-Encoder 不能用来做粗排召回？ | Cross-Encoder的打分原理是：把query和passage拼接为一个序列，然后送入Transformer，模型在完整的双向注意力交互下直接计算相似度，输出相关性分数，每个query token和passage token都有完整的双向Attention交互，这是为什么Cross-Encoder更精准的原因。Bi-Encoder则是先各自送入Transformer中进行编码，再计算点积来计算相似度，而且passage的编码向量已经在Dense Embedding中存入ChromaDB了，所以查询的时候只需要计算query的向量编码即可，因此Bi Encoder速度更快。Cross-Encoder不能用来粗排的原因是：粗排阶段，必须将每一个passage和原始query拼接后送入Transformer前向传播计算相似度得分，太慢了，而Bi-Encoder只需要计算一次query编码，然后和每个passage计算点积相似度即可。 | ✅ | [→ 查看](#a-cross-encoder) |
| Q4 | 你们的测试分几层？Unit、Integration、E2E 各自测什么，分别不测什么？ | 项目测试分3层。1. 第一层Unit测试：测逻辑，不测服务。类型契约：Document 必须有 source_path，缺了要抛 ValueError。工厂路由：EmbeddingFactory.create("openai") 能正确返回对应实例。数据变换：Chunk ID 生成的确定性、metadata 继承、序列化 roundtrip。边界条件：空文档、无效 batch_size、未知 provider。2. 第二层Integration：测组件协作，用真实服务。真实 API 调用：Azure LLM（gpt-4o）、Azure Embedding（ada-002）、Azure Vision。真实存储：ChromaDB upsert/query roundtrip、BM25 索引构建。完整 Pipeline：PDF → 切分 → Transform → Embedding → Storage，验证 chunk_count > 0、dense_dimension == 1536。容错路径：一路检索挂了，HybridSearch 自动 fallback 到另一路。3. 第三层E2E：测用户可见行为，全部真实。CLI 完整流程：subprocess.run(["python", "scripts/ingest.py", "--path", pdf]) 作为子进程执行，验证 returncode 和 stdout。MCP 协议合规：启动 Server 子进程，通过 stdin/stdout 发 JSON-RPC，验证 initialize → tools/list → tools/call 完整握手。Dashboard 冒烟：Streamlit 启动是否正常。 | ✅ | [→ 查看](#a-测试体系) |

---

## 二、参考答案

### <a id="a-存储架构"></a>Q: 这个系统里有哪几类存储？它们各自负责什么？为什么不能只用一个？

**参考答案**：
系统共有 **4 类存储**，各自查询模式不同，无法合一：

| 存储 | 位置 | 用途 | 查询模式 |
|------|------|------|---------|
| **Chroma 向量库** | `data/db/chroma/` | Dense Embedding 向量 + 原文 + metadata | 向量相似度 Top-K（HNSW 近似最近邻） |
| **BM25 倒排索引** | `data/db/bm25/` | term → 词频 + IDF 统计 | 全量加载 + 词频打分 |
| **ingestion_history.db** | `data/db/` | 文件 SHA256 处理记录 | 精确哈希查询（去重） |
| **image_index.db** | `data/db/` | image_id → 文件路径映射 | 精确 ID 查找 |

不能合并的根本原因：向量检索需要专用索引（HNSW），BM25 需要全局 IDF 统计，SQLite 做关系型精确查询——三种查询范式各需专用引擎。

---

### <a id="a-chunkrefiner"></a>Q: ChunkRefiner 做了什么？为什么需要额外的 LLM 步骤？

**参考答案**：
`RecursiveCharacterTextSplitter` 按字符边界物理切分，会将语义连续的段落切断（如"问题背景"和"解决方案"分入不同 Chunk），导致检索命中的 Chunk 缺乏上下文。

ChunkRefiner 的工作：
1. **合并语义断裂的段落**：LLM 判断相邻 Chunk 是否逻辑连续，若是则合并
2. **去噪清理**：移除 PDF 转换产生的页眉页脚乱码、重复标题

使每个 Chunk 成为 **Self-contained 的语义单元**，提升检索精度和 LLM 生成质量。

Prompt 迭代要点：需要通过负面约束（Do Not summarize / Do Not add formatting）防止 LLM 过度总结或注入格式噪声。定稿依据应包含长度保持率和下游检索指标的对比实验。

---

### <a id="a-cross-encoder"></a>Q: Cross-Encoder 和 Bi-Encoder 的区别？为什么不能做粗排召回？

**参考答案**：

| | Bi-Encoder | Cross-Encoder |
|--|-----------|--------------|
| 编码方式 | Query 和 Document **分别**编码为向量，算相似度 | Query 和 Document **拼接**一起输入模型，联合建模 |
| Document 向量 | 可**离线预计算**，查询时 O(1) | 每对 (Query, Chunk) 必须**实时推理**，O(n) |
| 精度 | 较低（无交互） | 更高（充分建模交互特征） |
| 适合场景 | 粗排召回（大规模） | 精排（10-30 条小候选集） |

**Cross-Encoder 不能做粗排**：5000+ 文档场景每次查询需推理 5000 次，延迟不可接受、成本极高。必须先用 Bi-Encoder 粗召回 Top-N，再用 Cross-Encoder 精排。

---

### <a id="a-测试体系"></a>Q: 测试分几层？单元测试怎么 mock LLM？

**参考答案**：
三层金字塔：
- **Unit（单元测试）**：1198+ 个，只测业务逻辑，用 `unittest.mock.patch` 替换 LLM 客户端返回预设响应，避免真实 API 调用
- **Integration（集成测试）**：验证模块间协作（Ingestion Pipeline / Chroma 存储读写），使用真实组件
- **E2E（端到端测试）**：`test_mcp_client.py` 启动真实 MCP Server 子进程，发送 JSON-RPC 消息验证完整链路；`test_dashboard_smoke.py` 用 Streamlit AppTest 无头渲染验证

---

## 三、简历包装点评

### 包装合理 ✅

- **"实现ChunkRefiner重组+元数据注入增强，显著降低冗余干扰"**：候选人详细描述了 Prompt 三版迭代过程（过度总结 → 负面约束 → few-shot 规划），并给出长度保持率（85%-105%）和 Hit Rate 提升（85%→92%）两个定稿依据，且主动提到 Graceful Fallback（rule-based 降级）。细节程度强烈表明亲手实现。

- **"采用Dense Embedding语义召回+BM25 Sparse关键词匹配，双路并行编码"**：存储问题中准确描述了 ChromaDB（HNSW 索引）和 BM25 JSON（倒排索引 term → [chunk_id, tf, doc_len] + IDF 统计）的存储细节和查询模式差异，超出表面描述。

- **"检索准确率（Hit Rate@10）达92%"**：在 ChunkRefiner 回答中自然给出了 baseline（85%）和提升后数据（92%），有对比实验支撑，数字非凭空捏造。

- **"基于Skill驱动全流程开发"**：测试回答中展示了对三层测试体系的深入理解，具体到函数名（`EmbeddingFactory.create`）、模型名（gpt-4o、ada-002）、CLI 命令、MCP 握手流程，说明确实经历了完整开发周期。

### 露馅点 ❌

- **"其余还包含SQLite下的Image Storage和FileIntegrityChecker，不过这不在我专门为面试准备的项目里，就不提及了"**：在问存储类型时主动提到了这两个组件的存在，但随即以"不在面试准备范围"为由回避展开。一个真正"设计"了系统的人不应该需要"准备"才能讲清自己设计的存储。措辞暴露出对这两个组件的理解可能仅停留在知道名字层面。**严重性：低**（知道组件存在且能命名，只是未深入，可能是表达策略问题而非认知缺失）

### 改进建议

1. **不要在面试中说"这个不在我准备范围内"**——即使确实没准备到，也应该尽量回答你知道的部分。面试官听到"没准备"会比听到"不完整的回答"更扣分
2. **SQLite 两库建议补齐**：`ingestion_history.db` 存文件 SHA256 用于去重、`image_index.db` 存 image_id→路径映射用于图片返回——这两个只需两句话就能说清，且能展示你对四路协调删除的理解

---

## 四、综合评价

**优势**：
- **ChunkRefiner Prompt 迭代（Q2）是本场最强回答**：三版演进（过度总结→负面约束→few-shot 规划）、两个量化定稿依据（长度保持率 + Hit Rate）、Graceful Fallback 设计，逻辑完整且细节丰富，面试官确信候选人亲自做过此模块
- **测试体系（Q4）回答具体到代码级别**：每层不是泛泛而谈，而是给出了具体测试场景（类型契约、工厂路由、CLI subprocess、MCP JSON-RPC 握手），展示了工程化素养
- **Cross-Encoder vs Bi-Encoder（Q3）从 Attention 机制层面解释**：不是背定义，而是从"双向注意力交互"和"预计算向量"角度讲清本质差异，说明理解了底层原理

**薄弱点**：
- **存储回答（Q1）刻意回避了 SQLite 两库**：四路存储是系统一致性设计的核心，回避两库意味着无法自然引出"跨存储协调删除"这一高分话题
- **FAST 模式覆盖面有限**：由于速攻风格，MCP 协议实现细节、RRF 公式、可插拔架构新增 Provider 流程等简历中的关键技术点未被考察

**面试官建议**：
1. 补齐 SQLite 两库（`ingestion_history.db` + `image_index.db`）的作用和四路协调删除逻辑，这是高频追问点
2. 建议用 DEEP 或 HARD 风格再跑一轮，覆盖 RRF 公式细节、MCP Stdio Transport 原理、可插拔架构三步流程等本场未触及的高价值考点
3. 面试表达上避免"不在准备范围"类措辞，改为主动回答已知部分

---

## 五、评分

| 维度 | 分数（满分 10）| 评分依据 |
|-----|--------------|---------|
| 项目架构掌握 | 8 | ChromaDB + BM25 双路存储讲清了查询模式差异和不能合并的原因；提到了 SQLite 两库但未展开；三层测试体系完整 |
| 简历真实性 | 9 | ChunkRefiner 三版迭代 + 量化定稿 + Graceful Fallback，强烈支持"亲手实现"；Hit Rate 92% 有 baseline 对比；唯一瑕疵是回避 SQLite 两库（低严重性） |
| 算法理论深度 | 8 | Cross-Encoder vs Bi-Encoder 从 Attention 机制层面解释到位；但 FAST 模式未考察 RRF 公式、评估指标（MRR/Faithfulness）等，样本有限 |
| 实现细节掌握 | 8 | BM25 存储结构（term→[chunk_id, tf, doc_len]+IDF）、测试具体场景（EmbeddingFactory.create、subprocess.run、JSON-RPC 握手）细节充分；缺少 MCP Tool 参数、chunk_id 生成等细节（未被问到） |
| 表达清晰度 | 8 | 回答结构清晰，善用编号分层；ChunkRefiner 回答尤其出色（版本演进→问题→解决方案→量化验证→降级设计）；扣分点："不在面试准备范围"这种措辞在真实面试中是减分项 |
| **综合** | **8.0** | 在 FAST 广度模式下，4 道题全部答出核心要点（4/4 ✅），无严重露馅。ChunkRefiner 的 Prompt 迭代经历和测试体系的代码级细节是本场亮点。主要提升空间在 SQLite 存储和面试表达策略上。建议换 DEEP/HARD 风格深度检验未覆盖的考点。 |

---

## 附录：本场考察题目清单

| 序号 | 方向 | 题目来源 | 题目 |
|-----|------|---------|------|
| Q1 | 方向 1 项目综述 | 开场题池 #7 | 这个系统里有哪几类存储？它们各自负责什么？为什么不能只用一个？ |
| Q2 | 方向 2 简历深挖 | P2 强动词池 #4（融合简历原文） | ChunkRefiner 的 Prompt 迭代过程与定稿依据 |
| Q3 | 方向 3 技术深挖 | B1 精排 Reranker | Cross-Encoder 和 Bi-Encoder 的本质区别 |
| Q4 | 方向 3 技术深挖 | F1 测试体系 | 测试分几层？各层测什么、不测什么？ |
