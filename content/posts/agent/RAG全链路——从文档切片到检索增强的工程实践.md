# RAG 全链路——从文档切片到检索增强的工程实践

> RAG (Retrieval-Augmented Generation) 是目前让 LLM 拥有"外部知识"最成熟的方式，也是 Agent 面试中最高频的话题。本文从 chunking 策略讲起，穿越 embedding、向量检索、reranker、一路到 HyDE 和 Self-RAG 等前沿增强技术，覆盖生产级 RAG 的完整链路。

---

## 一、RAG 解决了什么问题

### 1.1 大模型的三个"知识盲区"

1. **知识截止日期**：训练数据有截止时间，"今天发生了什么"模型不知道
2. **私有知识**：企业内部文档、代码库、客户数据——这些模型没见过
3. **可溯源**：LLM 输出的内容无法引用来源，"这句话从哪儿来的？"——答不上来

RAG 的核心思路：**回答问题之前，先检索相关文档，让 LLM 根据文档回答**。

### 1.2 RAG 的完整 Pipeline

```
┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
│ 文档加载  │ → │ 分块     │ → │ Embedding │ → │ 向量索引  │ → │ 检索     │
│ (Load)   │    │ (Chunk)  │    │ (Embed)   │    │ (Index)   │    │ (Search) │
└──────────┘    └──────────┘    └──────────┘    └──────────┘    └─────┬────┘
                                                                      │
                                                                      ▼
                                       ┌──────────┐    ┌──────────────┐
                                       │  生成     │ ← │  重排序       │
                                       │ (Generate)│    │  (Rerank)    │
                                       └──────────┘    └──────────────┘
```

---

## 二、分块策略 — RAG 的第一个关键决策

### 2.1 为什么分块如此重要

不分块直接 embed 整个文档 → 检索粒度太粗，查"怎么配置 Redis 超时"会返回整个 50 页运维手册。分得太细 → 语义被截断，一段话被切成两半，查什么都对不上。

### 2.2 主流分块方法

**固定大小分块 (Fixed-size Chunking)**：

```
chunk_size = 512 tokens, overlap = 64 tokens

文档: [token_0 ... token_575] → 两块
  Chunk 1: [0, 512)
  Chunk 2: [448, 575)  ← overlap 保证边界处不丢上下文
```

- overlap 的意义：防止关键语义恰好在边界处被截断。比如"Redis 超时配置参数是 timeout"恰好被切成"Redis 超时"和"配置参数是 timeout"，如果没有 overlap，两个 chunk 都不包含完整信息
- 缺点：不考虑语义边界，可能在句子中间断开

**语义分块 (Semantic Chunking)**：

不按固定 token 数，而是找语义断点——段落边界、标题、列表项。在 embedding 之后计算相邻句子的语义相似度，当相似度骤降时认为进入新话题，在此处切分。

```
句 1: "RAG 是一种结合检索和生成的技术。"
句 2: "它的核心优势是能让 LLM 访问外部知识。"         → 相似度 0.92 (继续，不分)
句 3: "接下来我们讨论向量数据库的选择。"               → 相似度 0.85 (继续)
句 4: "Redis 是一个内存数据库。"                      → 相似度 0.42 (骤降！在此切分)
```

**递归分块 (Recursive Chunking)**：

先用高级分隔符（两个换行 `\n\n`→ 段落 → 句子 → 词 → 字符），如果 chunk 还是太大，降级到更细的分隔符继续拆。LangChain 的 `RecursiveCharacterTextSplitter` 就是这个思路。

**Agentic Chunking**：让 LLM 自己决定在哪里切分。准确但昂贵，适合对质量要求极高的场景（比如法律文档）。

### 2.3 Chunk Size 怎么选

| 场景 | 推荐 chunk_size | 原因 |
|------|----------------|------|
| FAQ / 短问答 | 128-256 tokens | 答案通常是一两句话 |
| 技术文档 | 512-1024 tokens | 需要完整段落含代码示例 |
| 长文分析 / 论文 | 1024-2048 tokens | 需要更多上下文理解论证 |
| 多跳推理 | 256-512 tokens + 知识图谱 | 小块精确检索 + 图补全关系 |

**关键经验**：chunk_size 不是越大越好。小块检索精度高，但缺少上下文；大块上下文丰富，但检索噪音大。生产环境做法是**小块检索 + 大块返回**——用小块做检索（查得更准），返回时把相邻 chunk 也带上（补充上下文）。

---

##三、Embedding 与向量检索

### 3.1 Embedding 的本质

Embedding 模型把一段文本变成一个高维向量（如 1024 维的浮点数数组）。模型的训练目标是让"语义相近的文本"在向量空间中距离近。

做检索时，用户的 query 也被 embed 成一个向量，然后在向量数据库中找距离最近的 K 条记录：

```
similarity = cosine_similarity(query_vector, chunk_vector)
           = (A · B) / (|A| × |B|)

top-K = argmax(chunks, key=similarity, k=5)
```

### 3.2 向量数据库选择

| 数据库 | 类型 | 特点 | 适用规模 |
|--------|------|------|---------|
| Chroma | 嵌入式 | 轻量，适合 PoC | < 100K 向量 |
| Qdrant | 独立服务 | Rust 实现，高性能，过滤强 | 百万~千万级 |
| Milvus | 分布式 | 云原生，支持十亿级检索 | 亿级 |
| Weaviate | 独立服务 | 内置混合检索 + GraphQL | 百万~千万级 |
| Pinecone | 云服务 | 全托管，零运维 | 任意 |
| pgvector | PostgreSQL 扩展 | 利用现有 PG 基础设施 | 百万级 |

**选型关键考量**：
- 如果你的应用已经在用 PostgreSQL，pgvector 是最简单的方案——无需引入新组件
- 如果需要强过滤能力（metadata 过滤 + 向量检索），Qdrant 和 Weaviate 表现好
- 如果向量规模预期到亿级，Milvus 有最成熟的分布式方案

### 3.3 检索质量的关键优化

**1. Metadata 过滤**：

纯向量检索的一个经典失败：用户问"React 18 的新特性"，结果召回了 React 16、Vue 的文档。因为语义上它们都跟"前端框架"相关。

```
# 注入 metadata 过滤
检索时先做过滤，在缩小范围内做语义搜索:
  metadata.version = "18.x"  ← 精确过滤
  + 语义检索 "new features"
```

**2. 混合检索 (Hybrid Search)**：

```
最终分数 = α × 向量相似度 + (1-α) × BM25 关键词分

α 的取值:
- α = 0.7 偏向语义（适合自然语言问句）
- α = 0.3 偏向关键词（适合代码搜索、精确术语查询）
```

混合检索对"既需要语义理解又需要关键词精确匹配"的场景（如搜索"Kubernetes Pod CrashLoopBackOff 怎么排查"——每个词都很关键）有显著提升。

**3. 多向量策略**：

同一个文档，可以从不同角度生成多个 embedding：
- **内容 embedding**：文档本身的语义
- **摘要 embedding**：文档摘要的语义（检索时 query vs 摘要 有时比 query vs 原文更准）
- **假设问题 embedding (HyDE 的逆用)**：预处理时为每段文档生成"用户可能会怎么问这个问题"，检索时 query 跟这些假设问题匹配

---

## 四、检索增强技术

### 4.1 HyDE —— 假设答案驱动检索

经典 RAG 的问题是：用户的 query 很短，但相关文档很长。短向量 vs 长文本向量——语义空间不对齐。

HyDE 的做法：

```
1. 用户 query: "Redis 集群模式下 hash slot 怎么算"
2. 让 LLM 直接生成一个假设答案（不需要真实，只需要"看起来像答案"）:
   假设答案: "Redis 集群有 16384 个 hash slot，每个 key 通过
            CRC16(key) % 16384 计算所属 slot..."
3. 用这个假设答案去检索 →
   假设答案比原始 query 更接近真实文档的写作风格和内容密度
4. 用检索到的真实文档 + 原始 query 做最终生成
```

HyDE 的精妙之处：不是让"短 query"去匹配"长文档"，而是让"生成的伪答案"去匹配"真文档"——内容和风格都更接近。

### 4.2 Multi-Query —— 多角度召回

单条 query 可能召回不全。Multi-Query 让 LLM 把原始问题改写为多个不同角度的子问题：

```
原始: "怎么优化 React 渲染性能"

改写:
1. "React 性能优化有哪些常用技术"
2. "React.memo useMemo useCallback 最佳实践"
3. "React 虚拟列表实现方案"
4. "React Profiler 性能分析工具使用"
```

四个 query 分别检索，结果去重合并。召回率显著提升，但检索成本 ×4。

### 4.3 RAG-Fusion —— 互惠排名融合

Multi-Query 检索出的多组结果怎么合并？RAG-Fusion 用**互惠排名融合 (RRF)**：

```
RRF_score(doc) = Σ 1 / (k + rank_i(doc))

其中 k 是常数（通常取 60），rank_i 是文档在第 i 个搜索结果中的排名

文档在多个列表中排名靠前 → RRF 分高 → 最终排前面
```

RRF 的好处是无参数——不需要调权，只依赖排名信息。

### 4.4 Self-RAG —— 边检索边反思

Self-RAG 的核心思路：**不是检索了就用，而是检索后先判断"这个文档真的有用吗"**。

```
每篇检索到的文档，用 LLM 打四个标签:

- [Relevant]   文档与问题相关 → 用于生成
- [Irrelevant] 文档与问题无关 → 丢弃
- [Grounded]   生成的内容有文档支撑 → 可信
- [Ungrounded] 生成的可能是幻觉 → 需要再查

如果所有文档都被判 Irrelevant → 重新检索（改写 query 或换检索策略）
```

Self-RAG 的训练方式是给 LLM 做专门的微调，让它学会生成 `[Relevant]` / `[Irrelevant]` 这类"反思 token"。训练成本高，但对幻觉抑制效果显著。

### 4.5 GraphRAG —— 知识图谱 + 向量

向量检索擅长"这段话是关于什么的"，但不擅长"A 跟 B 是什么关系"。

GraphRAG 的做法：
1. 预处理阶段：LLM 从文档中提取实体和关系，构建知识图谱
2. 检索阶段：向量检索找到相关实体 → 在知识图谱中展开多跳邻居 → 同时返回原文和关系路径

```
用户: "Redis Cluster 的故障转移和 Sentinel 的有什么区别？"

向量检索 → 找到分别讲 Cluster 和 Sentinel 的文档
知识图谱 → 补充: Cluster (has_feature) 自动故障转移
                  Sentinel (has_feature) 自动故障转移
                  Cluster (mechanism) gossip协议
                  Sentinel (mechanism) 投票机制
                  → 这两者的区别在于"机制"而非"能力"
```

GraphRAG 在有丰富实体关系的领域（医药、法律、知识管理）效果提升最明显。

---

## 五、Reranker — 检索的最后一道关卡

### 5.1 为什么需要 Reranker

向量相似度高 ≠ 真正相关。Embedding 模型在检索时用的是双塔架构 (Bi-Encoder)——query 和 document 独立编码，最后的相似度只是一个内积。

Reranker 用的是 Cross-Encoder——把 query 和 document 拼在一起输入模型，做一次真正的"阅读理解"，判断相关程度。

```
Bi-Encoder (检索阶段):
  query → Embedding → [query_vector]  ─┐
  doc   → Embedding → [doc_vector]    ─┘→ cosine → 0.82

Cross-Encoder (重排阶段):
  [CLS] query [SEP] document [SEP] → 全连接 → relevance_score: 0.91
```

精度：Cross-Encoder >> Bi-Encoder
速度：Cross-Encoder << Bi-Encoder（因为不能预先算好 doc embedding）

所以生产中的做法：Bi-Encoder 粗召 top-100 → Cross-Encoder 精排 top-5 → 顶部 3-5 篇给 LLM。

### 5.2 常用 Reranker 模型

| 模型 | 特点 |
|------|------|
| Cohere Rerank v3 | API 服务，质量好，支持多语言 |
| bge-reranker-v2 | 开源，中文效果优秀 |
| cross-encoder/ms-marco-MiniLM | 轻量，延迟低 |
| Jina Reranker v2 | 支持 8K token 长文档 |

### 5.3 Reranking 的上下文窗口考虑

Reranker 选出的 top-3 文档，如果每篇 512 tokens，就是 1536 tokens 的上下文。再加上 system prompt 和对话历史，RAG 的"检索注入"实际上在跟其他内容抢 token 预算。

一个常见的优化：对 reranker top-N 文档做**压缩注入**。把 top-1 全文注入，top-2/3 只注入 LLM 生成的摘要。

---

## 六、RAG 评估

### 6.1 评估维度

RAG 系统需要从三个维度评估：

| 维度 | 评估问题 | 指标 |
|------|---------|------|
| 检索质量 | 召回的文档对吗？ | Recall@K, Precision@K, MRR, NDCG |
| 生成质量 | 用文档生成的回答对吗？ | Faithfulness, Answer Relevance |
| 端到端质量 | 整个系统能用吗？ | 人工评估, LLM-as-Judge (GPT Score) |

### 6.2 忠实度 (Faithfulness)

检查生成的答案是否完全基于检索到的文档——有没有 LLM "自由发挥"（幻觉）的部分。

一种自动评估方法：
1. 从生成的答案中分解出多个"断言"（atomic claims）
2. 用 NLI (自然语言推理) 检查每个断言是否被检索文档"支撑"或"矛盾"
3. Faithfulness = 被支撑的断言数 / 总断言数

### 6.3 RAGAS 框架

RAGAS 是目前最流行的 RAG 评估框架，它不需要 ground truth 答案，只需要 question + retrieved_contexts + answer + (可选的 ground_truth)：

```
from ragas import evaluate
from ragas.metrics import faithfulness, answer_relevancy, context_precision

result = evaluate(
    dataset,
    metrics=[faithfulness, answer_relevancy, context_precision]
)
```

关键指标：
- **Context Precision**：检索到的相关文档在排序中有多靠前
- **Context Recall**：ground truth 中需要的信息是否都被召回了
- **Faithfulness**：生成的答案是否可以归因于检索到的上下文
- **Answer Relevancy**：生成的答案是否聚焦于问题

---

## 七、RAG 常见陷阱

### 7.1 "检索到无关文档反而更糟"

如果 RAG 检索到的 top-3 文档中有 2 篇跟问题无关，LLM 可能被这些无关文档带偏——比不加 RAG 的结果还差。

解法：
- Reranker 阈值过滤：相关性分 < 阈值的文档直接丢弃，宁缺毋滥
- 如果全部文档都被过滤 → 告知用户"未找到相关信息"，而不是强行生成
- 在 prompt 中强调"如果检索到的文档与问题无关，请忽略它们并基于你自己的知识回答"

### 7.2 "Chunk 边界截断答案"

关键信息恰好被切在两个 chunk 之间，检索只召回了半个。

解法：
- 在检索到的 chunk 前后各取 1 个相邻 chunk 一起注入（context expansion）
- 使用 sentence-boundary aware 分块，避免在句子中间切开
- 较小的 chunk size + 检索后合并相邻 chunk

### 7.3 "信息冲突"

检索到的两篇文档给出了矛盾的信息（如不同版本的 API 文档）。

解法：
- 在 prompt 中注明每段文档的来源和时间
- 让 LLM 呈现矛盾而非选边站："文档 A (2024版) 说 XX，文档 B (2025版) 说 YY"
- Metadata 过滤确保只检索同一版本

---

RAG 不是一个"搭起来就能用"的系统。从分块策略到 reranker，每个环节的设计决策都会显著影响最终用户体验。下一篇将进入 Agent 系列最进阶的话题——**Multi-Agent 系统的设计与协作模式**。
