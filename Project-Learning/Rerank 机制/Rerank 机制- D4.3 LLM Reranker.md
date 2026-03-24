# Rerank 机制 - D4.3 LLM Reranker

> 学习日期: 2026-03-23
> 综合评分: 8.5/10 | 追问轮数: 1/1

---

**Q1: LLMReranker 的打分机制与 CrossEncoder 有什么本质区别？rerank() 方法中需要处理哪些 LLM 输出特有的不确定性问题？**

**标准答案**: CrossEncoder 用专门训练的小模型（如 ms-marco-MiniLM）对 (query, passage) 对做前向推理，直接输出一个 float 分数，过程是纯数学计算，确定性高。LLMReranker 用通用大语言模型，通过 prompt 指令让 LLM 阅读 query 和所有 passages，以结构化 JSON 形式输出每个 passage 的 passage_id + score + reasoning。本质是把排序问题转化为文本生成任务。核心差异：CrossEncoder 是模型直接回归出分数，LLMReranker 是让 LLM 生成文本再解析出分数。

LLM 输出的 6 项不确定性及应对：
1. **格式不可控**（LLM 可能用 markdown 包裹）— `_parse_llm_response` 先剥离 ` ```json ` / ` ``` ` 包裹再解析
2. **JSON 不合法** — `json.loads` 捕获 `JSONDecodeError`，抛出含原始响应片段的错误
3. **结构不符预期**（可能返回 dict 而非 list，或缺少字段）— 逐层校验：必须是 list → 每项必须是 dict → 必须含 passage_id 和 score
4. **分数类型不对**（score 可能是字符串而非数字）— `isinstance(score, (int, float))` 校验
5. **passage_id 幻觉**（LLM 可能生成不存在的 id）— `_map_results_to_candidates` 用 `if passage_id in id_to_candidate` 静默跳过
6. **LLM 调用失败**（超时、API 错误）— `llm.chat()` 包在 try/except 中，统一包装为 `LLMRerankError`

补充要点：
- Prompt 使用 0-3 四档评分标尺（`config/prompts/rerank.txt`），档位少可降低 LLM 决策负担，减少分数漂移
- Prompt 要求输出 `reasoning` 字段但 `_parse_llm_response` 不校验它——容错设计，即使 LLM 省略 reasoning 排序功能仍正常
- `__init__` 同时支持注入 `llm` 和 `prompt_path`（双注入通道），测试时可同时 mock LLM + 自定义 prompt
- `len(candidates) == 1` 时直接短路返回，无需调用 LLM

---

**Q2: CrossEncoder 和 LLM Reranker 各有什么优劣势？什么场景下选择 LLM Reranker？**

**标准答案**:

| 维度 | CrossEncoder | LLM Reranker |
|------|-------------|--------------|
| 延迟 | 毫秒级（本地推理） | 秒级（API 调用） |
| 成本 | 低（小模型，可本地跑） | 高（按 token 计费） |
| 精度 | 通用检索任务上已很强 | 复杂语义/长文本/专业领域可能更好 |
| 可解释性 | 只输出分数 | 可输出 reasoning 解释 |
| 确定性 | 高，同输入同输出 | 低，需要层层解析和兜底 |
| 部署依赖 | 需要 GPU/CPU 推理环境 | 只需 API key |

选择 LLM Reranker 的场景：
1. **需要可解释性** — 业务要求知道"为什么排在前面"
2. **专业/垂直领域** — 通用 CrossEncoder（如 ms-marco 训练的）在法律、医疗等领域可能不够准，LLM 世界知识覆盖更广
3. **候选量很小** — 5-10 条候选时延迟和成本可接受
4. **已有 LLM 基础设施** — 加一个 rerank 调用边际成本低，不想额外维护模型推理服务

反之，高吞吐、低延迟、大批量场景优先选 CrossEncoder。

---

## 📊 评价报告

**知识域**: D4 Rerank 机制 > **知识点**: D4.3 LLM Reranker — 基于大语言模型的重排序方案与 Prompt 设计
**追问轮数**: 1/1

### ✅ 回答亮点
- 本质区别一句话到位
- 6 项 LLM 输出不确定性分析全面，每项准确对应代码防御措施
- 选型分析维度丰富，场景推荐有理有据
- "已有 LLM 基础设施时边际成本低"决策视角很实际

### ⚠️ 需要加强
- Prompt 0-3 四档评分标尺设计未分析
- reasoning 字段的"宽松校验"容错设计未提及
- 依赖注入双通道（llm + prompt_path）未展开
- 单候选短路未提及

### 📈 评分明细

| 维度 | 分数 | 说明 |
|------|------|------|
| 准确性 | 9/10 | 本质区别和不确定性处理都准确无误 |
| 深度 | 8/10 | 选型分析有深度，prompt 设计层面可再深入 |
| 代码关联 | 8.5/10 | 引用了具体方法名和处理逻辑 |
| 设计思维 | 9/10 | 架构选型维度全面，场景推荐可落地 |

### 🏆 综合评分: 8.5/10

---

## 📚 学习指南

### 📂 相关代码
- `src/libs/reranker/llm_reranker.py` L39-L65 — __init__ 的双注入通道
- `src/libs/reranker/llm_reranker.py` L86-L108 — _build_rerank_prompt 的 prompt 组装
- `src/libs/reranker/llm_reranker.py` L110-L166 — _parse_llm_response 层层防御
- `config/prompts/rerank.txt` — 0-3 四档评分标尺 + reasoning 字段

### 🔗 参考资料
- LLM-as-a-Judge — LLM Reranker 是这一范式在检索排序场景的应用
- Prompt Engineering for Structured Output — 让 LLM 稳定输出结构化 JSON

### 💡 建议学习路径
1. 对比 cross_encoder_reranker.py 和 llm_reranker.py 的 rerank() 方法
2. 修改 rerank.txt 评分标尺观察 LLM 输出稳定性变化
3. 阅读 CoreReranker.rerank() 的 fallback 逻辑
4. 下一步学习 D4.4 Rerank Pipeline 集成
