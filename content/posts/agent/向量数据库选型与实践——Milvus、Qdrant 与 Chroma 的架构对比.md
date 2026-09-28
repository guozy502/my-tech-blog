---
title: "向量数据库选型与实践——Milvus、Qdrant 与 Chroma 的架构对比"
date: 2026-08-09
description: "从 RAG 系统中向量数据库的角色（Embedding 存储→相似度搜索→Metadata Filter 过滤）、三种主流向量数据库的架构差异（Milvus 分布式高吞吐/Qdrant 单机高性能+丰富 Filter/Chroma 轻量级开发者友好）、索引类型选型（IVF_FLAT/HNSW 的精度-速度权衡）、到多租户隔离与性能调优，拆解向量数据库在 Agent 检索栈中的选型依据。"
tags: ["AI Agent","向量数据库","Milvus","Qdrant","Chroma","RAG","Embedding"]
categories: ["Agent"]
---

# 历史背景——为什么 Elasticsearch 做不了 RAG？

ES 的 BM25 关键词检索对于"精确匹配关键词"很强——"Nginx 反向代理 timeout 配置"搜得准。但 RAG 需要的是**语义相似度**——用户问"反向代理怎么设置超时"，ES 搜不到"proxy_read_timeout"这个词，因为用户没提到。向量数据库把文本变成向量——"反向代理怎么设置超时"和"proxy_read_timeout 配置"在向量空间中距离很近——即使它们没有共享任何关键词。

这不是"向量数据库比 ES 好"——是他们的任务不同。BM25 做精确匹配，向量做语义匹配。Agent 的 RAG 系统通常**两者一起用**。

---

# 一、向量数据库在 Agent 中的角色

```
Agent 执行 RAG 检索的流程：

用户问题："这个项目中 OAuth 是怎么实现的？"
  ↓
① Embedding 模型 → 问题向量 [0.12, -0.34, 0.78, ...]
  ↓
② 向量数据库——相似度搜索
   在数百万个文档向量中找出最相似的 Top-K（如 K=5）
  ↓
③ Metadata Filter——过滤
   只看 *.py 文件，只看最新的 commit，只看技术文档
  ↓
④ 返回 Top-5 文档片段 → Agent 基于这些文档回答
```

**向量数据库在 Agent 架构中的位置**：

```
Agent ←→ 向量数据库（RAG 检索层）
  │
  └── 写入路径：文档解析 → Chunking → Embedding → 存入向量库
  └── 读取路径：Query → Embedding → 相似度搜索 → 返回文档
```

---

# 二、三种向量数据库对比

## 2.1 选型速查

| | Milvus | Qdrant | Chroma |
|------|--------|--------|--------|
| **定位** | **分布式**，支持十亿级向量 | **单机高性能**，丰富过滤 | **轻量级**，开发者友好 |
| **架构** | 存算分离：Proxy+Query Node+Data Node+Index Node+Meta Store | 单机 Rust 实现，内存+磁盘 | 嵌入式 Python 库，背后是 SQLite |
| **索引类型** | IVF_FLAT / IVF_PQ / HNSW / DiskANN | **HNSW**（默认）| HNSW |
| **Filter** | 标量过滤（表达式） | **强大的 Payload Filter** | 基础 Metadata Filter |
| **部署复杂度** | 高（需要 etcd + MinIO + Pulsar） | **中**（单二进制，也可 Docker） | **低**（`pip install chromadb`） |
| **适用数据量** | **千万-十亿级** | 百万-千万级 | 千-十万级 |
| **适合 Agent?** | 中大型 RAG 系统 | **最佳 Agent 单机方案** | 原型开发 / Demo |

## 2.2 架构差异

```
Milvus（分布式）：
  存算分离——Proxy（入口）→ Query Node（查询）→ Data Node（存储+索引）→ MinIO（持久化）+ etcd（元数据）+ Pulsar（消息）
  → 完全分布式，组件多，运维复杂

Qdrant（单机高效）：
  纯 Rust 实现，一个二进制即可运行
  → 极低的资源消耗（1GB 内存/百万向量）
  → 极强的 Payload Filter（不只是 Metadata 相等，支持范围、嵌套、全文搜索）

Chroma（轻量级）：
  本质是 SQLite + HNSW 索引
  → pip install 就能用，API 最简洁
  → 做 Demo 和原型的最佳选择
```

**Agent 场景建议**：
- 做 Demo / 原型 / 个人项目 → Chroma（安装最简单）
- 生产环境单机 RAG → Qdrant（高性能 + Payload Filter）
- 企业级大规模 RAG（亿级文档）→ Milvus（分布式水平扩展）

---

# 三、索引类型——速度与精度的权衡

## 3.1 两种核心索引

```
① IVF_FLAT（倒排文件 + 暴力搜索）：
   ① 先把所有向量聚成 N 个簇（K-Means 聚类）
   ② 查询时：找到最近的 M 个簇 → 只在这 M 个簇的向量中暴力搜索
   → 速度提升 N/(选择的簇数) 倍，精度略微下降
  
  参数：nlist = 簇的数量（越大越精确越慢）
  适合：不想建复杂索引，需要快速开始

② HNSW（Hierarchical Navigable Small World）：
   构建多层"小世界图"——每个向量连接几个"邻居"
   查询时从高层粗找 → 降到低层细找 → 找到最近向量
   → 查询速度极快，精度极高，但内存占用比 IVF_FLAT 多
  
  参数：M = 每条边连接的邻居数（越大越精确越占内存）
        ef_construction = 建索引时的搜索宽度
        ef_search = 查询时的搜索宽度
  适合：**RAG 场景的标准选择**
```

**Agent 开发者建议**：使用默认 HNSW 索引即可，不需要手动调参。M=16, ef_construction=200, ef_search=100 是 Qdrant 的默认值，对 99% 的场景足够。

---

# 四、多租户隔离——不同用户的数据存在哪

```python
# Chroma 做法：每个用户一个 Collection
collection = chroma_client.create_collection(f"docs-user-{user_id}")

# Qdrant 做法：Payload 中标记 user_id + 查询时 Filter
qdrant_client.search(
    collection_name="documents",
    query_vector=embedding,
    query_filter={"must": [{"key": "user_id", "match": {"value": user_id}}]},
    limit=5
)

# Milvus 做法：Partition Key（物理隔离）
# 在 Collection 中设置 partition_key_field="user_id"
# → Milvus 自动把不同 user_id 的数据分到不同的物理分区
# → 查询时自动只扫描该用户的分区
```

---

# 五、性能调优——RAG 中的向量库怎么配

```yaml
# Qdrant 配置建议（Agent 场景）
qdrant:
  storage:
    on_disk: true          # 向量存磁盘（百万级必须）
    memmap_threshold_kb: 20000  # 20MB 以下的热数据常驻内存
  optimizers:
    default_segment_number: 2   # 分段数（越大写入并行度越高）
    memmap_threshold: 20000
  hnsw_index:
    m: 16                       # 精度-内存权衡（默认即可）
    ef_construct: 100           # 建索引时不追求极致（省内存）
```

```python
# 搜索时动态调整精度-速度：
# ef 越大 → 更精确但更慢
qdrant_client.search(..., search_params={"hnsw_ef": 128})  # 默认 128
qdrant_client.search(..., search_params={"hnsw_ef": 64})   # 追求速度
qdrant_client.search(..., search_params={"hnsw_ef": 256})  # 追求精度
```

---

# 六、总结

| 场景 | 推荐 | 理由 |
|------|------|------|
| **原型/Demo** | Chroma | `pip install` 即可，API 最简洁 |
| **生产单机 RAG** | **Qdrant** | Rust 高性能 + 丰富 Payload Filter |
| **企业级大规模** | Milvus | 分布式水平扩展 + 十亿级向量 |
| **与 ES 互补** | ES(BM25) + Qdrant(向量) | Hybrid Search：BM25 精确 + 向量语义 |

# 延伸阅读

**Do——动手验证：**
- Chroma 5 分钟上手：`pip install chromadb` → 创建 Collection → 加 100 个文档向量 → 检索 10 个 query
- 同一个检索场景：对比纯向量检索 vs Hybrid Search（BM25 + 向量）的 Top-5 结果差异
- Qdrant 的 Payload Filter：按文件类型、创建时间、标签过滤后 + 向量相似度搜索

**Todo——深入方向：**
- 量化索引（Product Quantization）——如何用更少的内存存更多向量
- DiskANN——微软的磁盘向量索引，十亿级向量只需要 64GB RAM
- 向量数据库的多模态——同一个向量空间存入文本+图片+代码的 embedding

*本文参考资料：*
- Milvus 官方文档: https://milvus.io/
- Qdrant 官方文档: https://qdrant.tech/
- Chroma 官方文档: https://docs.trychroma.com/
