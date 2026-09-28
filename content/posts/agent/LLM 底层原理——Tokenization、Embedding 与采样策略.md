---
title: "LLM 底层原理——Tokenization、Embedding 与采样策略"
date: 2026-08-09
description: 从 LLM 输入输出的源头出发，拆解 Tokenization 的编码原理（BPE vs SentencePiece，为什么中英文 token 消耗差异 3 倍）、Embedding 向量化的数学本质（余弦相似度、不同 embedding 模型对检索质量的影响）、与采样策略的三参数控制（Temperature/Top-P/Top-K 如何控制生成结果的随机性与确定性），建立 Agent 开发者对"模型为什么这样回答"的底层理解。
tags: ["AI Agent","LLM","Tokenization","Embedding","采样策略","Temperature"]
categories: ["Agent"]
---

# 历史背景——Agent 开发者需要理解"模型的边界"

大多数 Agent 开发者从"一行 API 调用"开始——`openai.ChatCompletion.create()` 就够了。但当你在生产环境中遇到这些问题时，调 API 的经验派不上用场：

- "为什么同样的中文 prompt，GPT-4 消耗的 token 是 Claude 的 1.5 倍？" → 需要理解不同模型的 tokenizer 差异
- "为什么换了 embedding 模型后，RAG 的召回率从 92% 掉到了 78%？" → 需要理解 embedding 向量空间
- "为什么工具调用时 temperature 应该设为 0，但创意生成时应该设为 0.8？" → 需要理解采样策略。

理解这三个底层机制，你才能从"调 API 的人"变成"诊断 Agent 行为异常的工程师"。

---

# 一、Tokenization——"一个字"对于模型不是"一个 token"

## 1.1 Token ≠ 字

```
"Hello World" → GPT-4 tokenizer → ["Hello", " World"] → 2 tokens
"你好世界"   → GPT-4 tokenizer → ["你好", "世界"]   → 2 tokens  
"你好世界"   → Claude tokenizer → ["你", "好", "世界"] → 3 tokens

同一个中文句子，不同模型的 token 数可以差 1.5-2 倍！
```

**Agent 开发者需要关心的 token 影响**：

| 影响 | 具体表现 |
|------|---------|
| **成本** | Prompt 越长 → Token 越多 → 每次调用的成本越高 |
| **上下文窗口** | 中文模型可能因为 tokenizer 效率低，导致同样的中文文本更早触达上下文窗口上限 |
| **工具定义** | Function Call 的工具描述也消耗 token → 工具越多，System Prompt 中的工具定义占用的 token 就越多 |

## 1.2 BPE——最常见的 Tokenization 算法

```
BPE (Byte Pair Encoding) 的基本原理：

① 从单字符开始：'你', '好', '世', '界'
② 统计所有训练文本中相邻字符对的频率 → 找到最常见的组合
③ 把最常见的组合合并成一个新的 token
④ 重复 N 轮 → 得到最终的 token 词汇表

例：训练中 "你好" 出现了 10 万次，"世界" 出现了 8 万次
  → 第一轮：合并 "你" + "好" → "你好"（新 token）
  → 第二轮：合并 "世" + "界" → "世界"（新 token）
  → 最终词汇表：['你', '好', '世界', '你好', ...]

高频词 = 1 个 token（"你好"）
低频词 = 多个字符 token（"䶮" → 可能被拆成多个 byte token）
```

## 1.3 Token 优化——Agent 开发者能做什么？

```python
import tiktoken

# ① 估算 Prompt 的 token 消耗
def estimate_tokens(text: str, model: str = "gpt-4") -> int:
    encoding = tiktoken.encoding_for_model(model)
    return len(encoding.encode(text))

# ② 工具描述——用最少的字数描述参数
# ❌ 冗长版：18 tokens
"Retrieve the current weather conditions for a specified city, including temperature, humidity, and wind speed"

# ✅ 精简版：9 tokens
"Get current weather for a city (temp, humidity, wind)"

# ③ 同一个意思，中英文 token 差异
print(estimate_tokens("查询北京的天气"))  # ~7 tokens (中文)
print(estimate_tokens("Get weather for Beijing"))  # ~5 tokens (英文)

# ④ 对高频操作——考虑用英文 Prompt（省钱）
# 如果 Agent 的大部分 Prompt 是工具调用和系统指令（非用户面文本）
# 英文 Prompt 通常比中文省 30-50% token
```

---

# 二、Embedding——"用向量表示一段文字"

## 2.1 Embedding 在 Agent 中的作用

```
RAG 检索流程：
  用户问题："如何配置 Nginx 反向代理"
    → Embedding 模型 → [0.12, -0.34, 0.78, ...]  ← 1536 维的向量
    → 在向量数据库中找余弦相似度最高的文档向量
    → 返回最相关的文档片段

Embedding 的质量决定了 RAG 检索的命中率
```

## 2.2 余弦相似度——"两个文本有多接近"

```python
import numpy as np

def cosine_similarity(a: list[float], b: list[float]) -> float:
    """计算两个向量的余弦相似度 [-1, 1]，1 表示完全相同"""
    a = np.array(a)
    b = np.array(b)
    return np.dot(a, b) / (np.linalg.norm(a) * np.linalg.norm(b))

# 例：
q = embed("如何配置 Nginx 反向代理")      # 查询向量
d1 = embed("Nginx reverse proxy 配置")   # 相关文档 → 余弦相似度 0.92
d2 = embed("Apache 安装教程")            # 不相关文档 → 余弦相似度 0.31
```

## 2.3 Embedding 模型的选择影响检索质量

| 模型 | 维度 | 中文质量 | 说明 |
|------|------|:---:|------|
| `text-embedding-3-small` (OpenAI) | 512/1536 | 中 | 便宜、快，适合通用场景 |
| `text-embedding-3-large` (OpenAI) | 256/1024/3072 | 好 | 更准但更贵 |
| `bge-large-zh-v1.5` (BAAI) | 1024 | **最好** | 中文开源最佳选择 |
| `multilingual-e5-large` | 1024 | 好 | 多语言场景 |

**Agent 开发者决策**：用同一个 Embedding 模型索引所有文档 → 同一个 Embedding 做查询向量化 → 否则向量空间不一致，相似度毫无意义。

---

# 三、采样策略——控制生成的"随机性"

## 3.1 Temperature（温度）

```
Temperature 控制生成概率分布的"平坦度"：

Temperature = 0（确定性）：
  模型永远选择概率最高的下一个 token
  "我明天要去" → 100% 概率 → "上班"
  适合：JSON 生成、工具参数生成、代码生成

Temperature = 1.0（平衡）：
  模型按概率分布采样
  "我明天要去" → 40% "上班", 30% "学校", 20% "医院", 10% "公园"
  适合：通用对话、摘要

Temperature = 1.5（高随机性）：
  概率分布被"拉平"——低概率 token 也有更高的机会被选中
  "我明天要去" → 可能生成 "月球"、"火星"、"外太空"
  适合：创意写作、头脑风暴
```

**Agent 开发者需要牢记**：工具调用时 **temperature 应该接近 0**——你不想因为温度太高而导致工具参数中的 `city` 变成 `Beijingg` 或 `!!Beijing!!`。

## 3.2 Top-P（核采样）与 Top-K

```
Top-K（经典方法）：
  只从概率最高的 K 个 token 中采样
  K=50 → 只考虑前 50 个 token → 其余 49950 个直接淘汰

Top-P（核采样，Nucleus Sampling）：
  只考虑累积概率达到 P 的最小 token 集合
  P=0.9 → 概率前几的 token 累积到 0.9 → 从这些中采样 → 动态 K

推荐的 Agent 调用参数：
  工具调用：temperature=0, top_p=1  （不采样，确定性输出）
  创意生成：temperature=0.7, top_p=0.9
  严格编码：temperature=0.1, top_p=1
```

## 3.3 同一个 task，不同采样参数的效果

```python
# 工具参数生成 — 必须确定！
response = llm.call(
    prompt="Extract the city name from: '查询北京的天气'",
    temperature=0.0,  # ← 关键
    response_format={"type": "json_object"}
)
# 输出 100%：{"city": "北京"}
# 如果 temperature=1.0 → 可能输出 {"city": "北京", "province": "北京"} 或其他格式

# 最终报告生成 — 可以有创意
response = llm.call(
    prompt="根据以上分析，生成一份代码审查报告",
    temperature=0.7,   # ← 允许一点变化
)
```

---

# 四、总结

| 概念 | Agent 开发者需要记住什么 |
|------|---------------------|
| **Tokenization** | 中文比英文消耗更多 token；不同模型的 tokenizer 不同 → 选模型时考虑 token 成本 |
| **Embedding** | 用同一个模型索引和查询 → 向量空间对齐；中文 RAG 首选 bge-large-zh |
| **Temperature** | 工具调用 = 0（确定）；创意生成 = 0.7-1.0；代码生成 = 0-0.2 |
| **Top-P** | 和 Temperature 不冲突；工具调用时 P=1（不限制），温度已经够"冷"了 |

# 延伸阅读

**Do——动手验证：**
- 用 `tiktoken` 对比同一个 prompt 中英文的 token 消耗差异
- 同一 query 用 `text-embedding-3-small` vs `bge-large-zh-v1.5` 做向量化 → 检索同一批文档 → 对比 Top-5 结果的差异
- 同一个 code generation prompt → temperature=0 vs 0.8 → 跑 10 次 → 统计结果的变异程度

**Todo——深入方向：**
- KV Cache 的显存管理——为什么长上下文贵且慢
- LoRA 微调的原理——不改变原始权重，旁路加 trainable adapter
- Transformer 的注意力机制——Self-Attention / Multi-Head / Flash Attention

*本文参考资料：*
- OpenAI Tokenizer: https://platform.openai.com/tokenizer
- BAAI Embedding (bge-large-zh): https://huggingface.co/BAAI/bge-large-zh-v1.5
- "The Illustrated Transformer" by Jay Alammar
