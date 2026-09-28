---
title: "Context Engineering——Token Budget、上下文压缩与 KV Cache 原理"
date: 2026-08-08
description: 从 Context Engineering 与 Prompt Engineering 的本质区别（前者决定"模型看到什么信息"，后者决定"模型怎么理解这些信息"）、Context Packing 的四项策略（哪些进上下文、哪些留外部、优先级排序、Token Budget 管理）、上下文压缩的三种技术（长文件截断+摘要、工具结果压缩、历史对话压缩）与 KV Cache 原理、到 Context Debug Report 的诊断方法，拆解 Agent 的能力上限如何由上下文决定。
tags: ["AI Agent","Context Engineering","Token Budget","KV Cache","上下文压缩","RAG"]
categories: ["Agent"]
---

# 历史背景——上下文窗口从 4K 到 128K，但"能存下"≠"会用"

2022 年，GPT-3 的上下文窗口只有 4K tokens——你需要小心翼翼地裁减输入，因为多放半篇文章就可能撑爆窗口。2024-2025 年，主流模型的上下文窗口扩展到 128K tokens 甚至 1M——可以把整本书塞进上下文了。

但窗口扩大带来了一个新问题：**模型虽然能看到 128K tokens，但它在长上下文中"找出关键信息"的能力并没有同比提升。** 把 100 页文档全塞进去，模型可能"读完"后给出的答案质量反而比只给 5 页精选内容更差——因为不相关的信息稀释了相关信息的注意力权重。这就是"Needle in a Haystack"问题。

Context Engineering 的出现，就是为了回答这个问题：**不是"模型能看多少就塞多少"，而是"哪些信息值得被模型看到"。**

---

# 一、Context Engineering 与 Prompt Engineering 的分工

| | Prompt Engineering | Context Engineering |
|------|-------------------|-------------------|
| **管的范围** | System Prompt + User Prompt | 整个 Request 中填入的所有信息 |
| **解决的问题** | "模型怎么理解这些信息" | "模型应该看到什么信息" |
| **操作对象** | 指令、Few-shot 示例、输出格式 | 工具调用结果、检索到的文档、历史对话、项目元数据 |
| **核心决策** | 角色定义、行为约束、输出格式 | 哪些进上下文、哪些留在外部、怎么压缩、优先级排序 |

**一个具体例子**：

```
任务：Agent 在修改一个 Java 项目的认证模块。

Context Engineering 负责回答：
  - 哪些代码文件应该放入上下文？（相关文件 vs 无关文件）
  - 工具调用的返回结果哪些需要保留？（成功的？失败的？）
  - 之前的对话历史哪些需要压缩？（很久之前且已完成的部分）
  - 这个项目的技术栈、构建命令等元数据需要放在上下文的哪个位置？

Prompt Engineering 负责回答：
  - Agent 的角色定义是什么？（"资深 Java 工程师"？）
  - 输出的格式要求？（"生成 Pull Request 描述 + 变更说明"？）
  - 修改原则？（"优先保持向后兼容、不引入新依赖"？）
```

---

# 二、Context Packing——Token Budget 管理

## 2.1 上下文的"四桶模型"

```
Agent 的上下文可以分成四个桶：

┌────────────── 上下文窗口（128K tokens）────────────────┐
│                                                          │
│ ┌──────────┐ ┌──────────────┐ ┌─────────┐ ┌──────────┐ │
│ │ ①系统指令 │ │ ②任务上下文   │ │ ③检索结果│ │ ④工作记忆│ │
│ │ System   │ │ 用户输入      │ │ 代码文档  │ │ 工具结果  │ │
│ │ Prompt   │ │ 项目元数据    │ │ 引用来源  │ │ 推理过程  │ │
│ │ 工具定义  │ │ 当前目标      │ │ 相关代码  │ │ 已完成步骤│ │
│ └──────────┘ └──────────────┘ └─────────┘ └──────────┘ │
│                                                          │
│ 约 2-5K     │ 约 2-10K       │ 约 5-50K   │ 约 5-50K   │
│ 固定不变     │ 随任务变化      │ 随检索变化  │ 随执行增长  │
└──────────────────────────────────────────────────────────┘
```

**Token Budget 的核心**：每一个桶的大小不是固定的，但**总预算**是固定的（由上下文窗口限制）。当"工作记忆"这个桶增长时，必须从其他桶中"挤出"空间。

## 2.2 Context Prioritization——什么优先进入上下文

```
优先级排序（从高到低）：

① 当前任务的直接目标（必入）
② 最近 3 轮的工具调用结果（必入——Agent 需要知道自己刚做了什么）
③ 与当前任务相关的代码片段 / 文档片段（检索到的，按相关性排序）
④ 项目元数据（技术栈、构建命令、编码规范——第一次放，后续除非变化不重复放）
⑤ 历史工具调用结果（超出 3 轮的部分——可以压缩后再入）
⑥ 已完成的子任务摘要（可以进一步压缩）

排序原则：
  - 与当前步骤直接相关的 → 优先级最高
  - 已经完成且不再需要的 → 优先级最低，可压缩或移出
  - 项目元数据 → 放在系统指令中，不占用任务上下文预算
```

## 2.3 Context Budget 的实际分配

```python
class ContextBudget:
    def __init__(self, max_tokens=128_000):
        self.max_tokens = max_tokens
        self.buckets = {
            "system": self.count_tokens(system_prompt + tool_definitions),
            "task": 0,
            "retrieval": 0,
            "scratchpad": 0
        }
    
    def allocate(self, bucket: str, tokens: int) -> bool:
        """尝试为某个桶分配 tokens，返回是否有足够的预算"""
        used = sum(self.buckets.values())
        if used + tokens > self.max_tokens:
            return False  # 预算不够 → 需要压缩或清理
        self.buckets[bucket] += tokens
        return True
    
    def remaining_budget(self) -> int:
        return self.max_tokens - sum(self.buckets.values())
```

---

# 三、上下文压缩——Token Budget 不够时的三道防线

## 3.1 长文件截断 + 摘要

```python
def compress_long_file(content: str, max_tokens: int) -> str:
    """长文件压缩：优先保留关键部分"""
    if count_tokens(content) <= max_tokens:
        return content
    
    # ① 提取文件级别的摘要（LLM 生成或规则提取）
    #    类名/函数签名/import/注释 → 保留
    #    函数体 → 截断 + 摘要
    lines = content.split("\n")
    signatures = extract_signatures(lines)        # 函数签名
    imports = extract_imports(lines)              # import 语句
    doc_comments = extract_doc_comments(lines)    # 文档注释
    
    # ② 用 LLM 生成函数体摘要
    body_summary = llm.summarize(content, max_tokens=200)
    
    # ③ 拼接压缩版本
    compressed = "\n".join(imports) + "\n" + body_summary
    return compressed[:max_tokens]
```

## 3.2 工具结果压缩

Agent 执行过程中产生的大量工具结果，不需要全部保留在上下文中：

```python
class ToolResultCompressor:
    def compress(self, results: list[ToolResult], budget: int) -> str:
        """压缩历史工具结果"""
        # ① 最近的工具结果 → 保留完整（Agent 需要知道自己刚做了什么）
        recent = results[-3:]
        
        # ② 更早的工具结果 → 只保留关键字段 + 摘要
        older = results[:-3]
        compressed_older = []
        for r in older:
            compressed_older.append(
                f"[{r.tool_name}] → {r.summary} (took {r.duration_ms}ms)"
                if r.success
                else f"[{r.tool_name}] FAILED: {r.error[:100]}"
            )
        
        # ③ 反复重试同一工具 → 进一步压缩为一行"尝试 3 次后成功"
        deduped = deduplicate_retries(compressed_older)
        
        return format_compressed(recent, deduped)[:budget]
```

## 3.3 历史对话压缩

```
每轮 Agent 循环产生的"思考 → 执行 → 观察"信息量远超单轮对话。

压缩策略：
  ① 已完成且成功的步骤 → 一行摘要（"步骤 2: 搜索代码 → 找到 5 个文件 → 成功"）
  ② 重试多次后成功的步骤 → 两行（"步骤 3: 运行测试 → 失败 2 次 → 修改 → 成功"）
  ③ 还在进行中的步骤 → 保留完整上下文（Agent 需要细节来继续决策）
  ④ 任务完全结束 → 生成"任务摘要"存入 Memory，上下文清空
```

---

# 四、KV Cache——为什么上下文越长推理越贵

## 4.1 KV Cache 的工作原理

```
Transformer 的自注意力机制：

  每次生成一个新 token 时，模型需要"回顾"上下文中所有之前的 token
  → 对这些 token 的 Key 和 Value 做矩阵乘法
  → 如果没有 KV Cache → 每次生成都要从头算一遍所有历史 token 的 K 和 V
  → 有了 KV Cache → 之前算过的 K 和 V 缓存在 GPU 显存中，只算新 token 的

所以"生成第 5000 个 token"和"生成第 5 个 token"的延迟差异可以差一个数量级——
因为前者需要 attend 到 4999 个之前的 token，后者只 attend 到 4 个。
```

## 4.2 KV Cache 对成本和延迟的影响

```
影响 1：计算成本随上下文长度超线性增长
  生成第一个 token → O(N)（N = 上下文长度）→ 中等
  生成后续每个 token → O(N)（每次都要 attend N 个之前 token）→ 超大！
  → 不缓存的位置每 step 都要重算，缓存的位置随 N 线性增长

影响 2：GPU 显存占用
  KV Cache 存储在 GPU 显存中 → 上下文越长 → KV Cache 越大 → 留给其他请求的显存越少
  → vLLM 的 PagedAttention 通过分页管理 KV Cache 来解决此问题

影响 3：延迟
  用户感知到的"思考时间" = 上下文越长，每次生成新 token 的延迟越明显
  → 这也是为什么应该"不要塞满 128K，只放真正相关的 10K"
```

---

# 五、Context Debug Report——"Agent 这次到底看到了什么"

当 Agent 输出不符合预期时，第一优先级不是改 Prompt，而是**看 Agent 的上下文中到底塞了什么**。

```python
class ContextDebugReport:
    def generate(self, trace: AgentTrace) -> str:
        report = []
        
        # ① Token 使用分布
        report.append(f"### Token Usage")
        report.append(f"- System Prompt: {trace.system_tokens} tokens")
        report.append(f"- Tool Definitions: {trace.tool_def_tokens} tokens")
        report.append(f"- Retrieved Context: {trace.retrieval_tokens} tokens")
        report.append(f"- Tool Results: {trace.tool_result_tokens} tokens")
        report.append(f"- History: {trace.history_tokens} tokens")
        report.append(f"- **Total: {trace.total_tokens}/{trace.max_tokens} tokens**")
        
        # ② 检索到的内容质量
        report.append(f"### Retrieved Context")
        for chunk in trace.retrieved_chunks:
            report.append(f"- [{chunk.source}] relevance={chunk.score:.2f}")
            report.append(f"  Content: {chunk.text[:200]}...")
        
        # ③ 工具结果中是否包含错误
        report.append(f"### Tool Results")
        for result in trace.tool_results:
            status = "✅" if result.success else f"❌ {result.error}"
            report.append(f"- [{result.tool_name}] {status}")
        
        # ④ 历史对话压缩效果
        report.append(f"### History Compression")
        report.append(f"- Original: {trace.history_original_tokens} tokens")
        report.append(f"- Compressed: {trace.history_compressed_tokens} tokens")
        report.append(f"- Ratio: {trace.history_compressed_tokens / max(trace.history_original_tokens, 1):.0%}")
        
        return "\n".join(report)
```

**Context Debug Report 的典型诊断场景**：

| 症状 | 从 Context Debug Report 中能发现 |
|------|-------------------------------|
| Agent 输出和用户期望不符 | 检索到的文档可能不相关（检索排序问题） |
| Agent 重复执行同一操作 | 历史对话压缩太激进，Agent 忘了自己做过 |
| Agent 输出的引用不准确 | 检索到的 chunk 可能没有精确的行号 |
| Agent 任务的成本超高 | Token Budget 分配不合理 → 系统指令占太多 |

---

# 六、总结

| 概念 | 解决的问题 | 核心方法 |
|------|----------|---------|
| **Context Packing** | 什么进上下文 | 四桶模型 + 优先级排序 + Token Budget 管理 |
| **Context Compression** | 上下文超限怎么办 | 长文件截断+摘要 / 工具结果压缩 / 历史对话压缩 |
| **KV Cache** | 为什么长上下文慢且贵 | 缓存 Key/Value 矩阵 → 显存占用 + 计算成本超线性增长 |
| **Context Debug Report** | Agent 输出错了怎么查 | Token 分布 + 检索质量 + 工具结果 + 压缩效果 |

# 延伸阅读

**Do——动手验证：**
- 在同一 Agent 任务中，分别用"全量上下文（100K tokens）"和"压缩后上下文（20K tokens）"运行，对比——成功率、延迟 P99、Token 成本
- 用 Anthropic/OpenAI 的 token counting API 统计你的 Agent 一次执行的 Token 分布（System / Task / Retrieval / Tool Results / History 各占多少）

**Todo——深入方向：**
- PagedAttention 与 vLLM——KV Cache 的分页管理如何把显存利用率从 20-30% 提升到 80%+
- Prompt Caching——同一个 System Prompt 在多轮对话中如何跨请求复用
- 长上下文中"注意力衰减"问题——中间位置的信息比开头和末尾更难被模型注意到

*本文参考资料：*
- Anthropic Context Engineering Guide
- vLLM PagedAttention 论文: "Efficient Memory Management for Large Language Model Serving with PagedAttention" (2023)
- "Lost in the Middle" 论文: "How Language Models Use Long Contexts" (Liu et al., 2023)
