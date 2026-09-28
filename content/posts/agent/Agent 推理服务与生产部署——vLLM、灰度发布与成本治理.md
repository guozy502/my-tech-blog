---
title: "Agent 推理服务与生产部署——vLLM、灰度发布与成本治理"
date: 2026-08-08
description: 从推理引擎的底层原理（vLLM 的 PagedAttention + Continuous Batching 如何将 GPU 显存利用率从 30% 提升到 80%+）、TGI 与 Triton 多模型推理服务的架构差异、模型路由与降级策略（强/弱模型分层 → 小模型优先 + 缓存兜底）、到 Agent 系统的版本管理（Prompt/模型/工具/Skill/RAG 策略五层版本对象）、灰度发布（按用户/租户/任务类型 + Feature Flag 控制）与成本治理（Token Budget + 缓存体系 + 模型降级），拆解 Agent 从"能运行"到"生产级"的完整部署体系。
tags: ["AI Agent","vLLM","推理服务","灰度发布","成本治理","PagedAttention"]
categories: ["Agent"]
---

# 历史背景——Agent 的生产部署和传统微服务有什么不同？

传统微服务的部署：打包 Docker 镜像 → 推到容器仓库 → K8s 拉镜像 → 启动 → 健康检查 → 接入流量。整个过程是基于"代码是确定的"这个前提——同一个镜像在测试环境和生产环境的行为完全一样。

Agent 的部署打破了这个前提：**同一个 Docker 镜像、同一个代码版本、同一个配置——但因为底层模型的温度参数不同、RAG 检索策略不同、Prompt 版本不同——Agent 的行为可能完全不同。** Agent 的"版本"不是代码的版本，而是"模型版本 + Prompt 版本 + 工具版本 + RAG 策略版本"的组合。这四者任何一个变了，Agent 的行为都可能退化。

所以 Agent 的生产部署不只是"把代码部署到 K8s"，而是**五个版本对象 + 灰度发布 + Feature Flag + 成本治理的组合体系**。

---

# 一、推理引擎——Agent 的"CPU"

## 1.1 为什么不能直接用 OpenAI API？

大多数 Agent 开发从 `openai.ChatCompletion.create()` 开始——一行代码调一个模型，快速搭建 Demo。但生产环境有三个硬伤：

```
硬伤 1：延迟不可控。公有 API 的排队 + 模型推理 = P99 可能到 5-10 秒
硬伤 2：成本不可控。每次调用都按 Token 计费——Agent 一次执行可能 5-20 次模型调用 → 单个任务成本数美元
硬伤 3：数据安全。用户的代码、文档、业务数据经过公网传输到模型服务端

推理引擎（vLLM / TGI / Triton）就是"把模型部署在你自己的 GPU 上"——
  延迟可控（无外部排队）、成本可控（按 GPU 机时而非 Token 计费）、数据安全（内网传输）
```

## 1.2 vLLM——PagedAttention 与 Continuous Batching

vLLM 是 UC Berkeley 开源的 LLM 推理引擎。它的两个核心创新分别解决了 GPU 显存的浪费问题和推理延迟的浪费问题：

**PagedAttention——解决显存浪费**：

```
KV Cache 的存储问题：
  每次推理时，模型为上下文中每个 token 缓存 Key 和 Value 矩阵（避免重复计算）
  但 KV Cache 的大小随上下文长度变化——
  一次短请求只用 1K tokens = 小 KV Cache
  一次长请求要用 32K tokens = 大 KV Cache

传统的做法：为每个请求预分配最大可能的 KV Cache 空间
  → 短请求浪费了大量预分配的显存 → GPU 利用率只有 20-30%

PagedAttention 的解法：和操作系统的"分页内存"一样
  → 把 KV Cache 切分成固定大小的 block
  → 每个请求按需分配 block（不需要预分配最大空间）
  → 不同请求的 KV Cache block 可以交错存放在 GPU 显存中
  → 显存利用率从 30% → 80%+
```

**Continuous Batching——解决 GPU 空闲等待**：

```
传统批处理（Static Batching）：
  收集一批请求 → 一起推理 → 等所有请求都处理完 → 再收集下一批
  → 一个请求特别长 → 整个 batch 都在等它 → GPU 空闲

Continuous Batching：
  不是"等一批全部完成再开下一批"
  而是"一个请求刚完成，马上补一个新请求进来"
  → GPU 永远不空转 → 吞吐翻倍
```

**vLLM 的性能数据**：在相同 GPU（A100-80G）上，vLLM 的吞吐是 HuggingFace Transformers 的 **24 倍**，是 TGI 的 **2-3 倍**。

## 1.3 TGI 与 Triton——其他选项

| | vLLM | TGI (HuggingFace) | Triton (NVIDIA) |
|------|------|-------------------|----------------|
| **核心优化** | PagedAttention + Continuous Batching | Flash Attention + 流水线并行 | 多模型并发 + 动态批处理 |
| **模型支持** | 专注大语言模型 | HuggingFace 全生态 | 所有 NVIDIA 优化过的模型 |
| **适合 Agent？** | **是（单模型高吞吐）** | 是（需要 HuggingFace 模型） | 是（需要同时跑多个不同模型） |
| **部署复杂度** | 低（pip install + 一条命令） | 中 | 高（需要配置模型仓库和推理 pipeline） |

---

# 二、模型路由与降级——不是所有请求都需要最强模型

## 2.1 强模型 vs 弱模型分层

```
Agent 的一次执行中，不是每一次模型调用都需要 GPT-4 级别的能力：

强模型（Claude Opus / GPT-4）：
  任务规划（"这个复杂任务该拆成哪几个步骤？"）
  代码生成（"写一个完整的实现"）
  最终回答生成（"把分析结果整理成 Markdown 报告"）

弱模型（Claude Haiku / GPT-4o-mini）：
  简单工具参数生成（"查询天气，城市=北京"）
  文本摘要（"把这段 2000 字的文档摘要成 200 字"）
  格式校验（"这个 JSON 是否符合 Schema？"）

混合模型的成本差异：
  Opus: $15/1M input tokens, $75/1M output tokens
  Haiku: $0.80/1M input tokens, $4/1M output tokens
  → 将 70% 的调用从 Opus 切到 Haiku → 成本降到原来的 ~30%
```

## 2.2 模型路由策略

```python
class ModelRouter:
    def route(self, call_type: str, context: dict) -> str:
        """根据调用类型和上下文选择模型"""
        
        # ① 任务规划 → 必用强模型
        if call_type == "planning":
            return "claude-opus-4-6"
        
        # ② 代码生成 → 强模型
        if call_type == "code_generation":
            return "claude-sonnet-4-6"
        
        # ③ 简单的工具参数 → Haiku 足够
        if call_type == "tool_param_generation" and context["complexity"] == "simple":
            return "claude-haiku-4-5"
        
        # ④ 格式校验 → Haiku
        if call_type == "format_validation":
            return "claude-haiku-4-5"
        
        # ⑤ 兜底 → 中等模型
        return "claude-sonnet-4-6"
```

## 2.3 降级策略——模型挂了怎么办

```
降级链（Fallback Chain）：
  ① 主模型超时 → 重试 1 次（同模型）
  ② 再超时 → fallback 到 Haiku（弱模型，但至少能用）
  ③ Haiku 也挂了 → 返回缓存结果（如果这个请求和 5 分钟内的一个请求一样）
  ④ 缓存也没有 → 返回人工兜底消息"系统繁忙，请稍后重试"
```

```python
class ModelFallback:
    def call_with_fallback(self, prompt: str, preferred_model: str):
        models = [preferred_model, "claude-haiku-4-5"]  # fallback 链
        
        for model in models:
            try:
                result = llm.call(model, prompt, timeout=30)
                return result
            except TimeoutError:
                continue
            except RateLimitError:
                time.sleep(2)
                continue
        
        # 最终降级：缓存 or 兜底
        cached = cache.get(hash(prompt))
        if cached:
            return cached
        return "系统繁忙，请稍后重试"
```

---

# 三、缓存体系——Token 消耗的最大优化杠杆

## 3.1 Agent 的五层缓存

```
① Prompt Cache：
   System Prompt 在多次请求中完全一样（如工具定义、系统角色）
   → 缓存模型的 Key/Value 矩阵（不需要每次推理都重算）
   → Anthropic 原生支持，OpenAI 通过 "prompt caching" API
   → 节省：System Prompt 部分 90% 的 cost

② Semantic Cache（语义缓存）：
   两个请求虽然文字不同，但语义一样
   "查询北京的天气" = "北京今天天气怎么样"
   → 用 Embedding 相似度 > 0.95 判断为同一请求 → 返回缓存结果
   → 节省：相似查询 100% 的模型调用

③ Embedding Cache：
   同一段文本的 Embedding 在 RAG 检索中可能被多次查询
   → 缓存 embedding → 不需要每次都调 Embedding API
   → 节省：Embedding API 调用成本

④ RAG Cache：
   同一段检索结果在多轮对话或多次执行中重复出现
   "该项目的代码结构是怎样的？" → 同一个项目的两次调用，检索结果完全一样
   → 缓存检索结果 + chunks → 不需要每次都跑向量搜索
   → 节省：向量数据库查询成本

⑤ Tool Result Cache：
   同一个工具参数对同一个用户的调用结果在短时间内不变
   "查询仓库 guozy502/my-tech-blog 的 star 数量" → 10 秒内第二次调 → 缓存
   → 节省：工具调用成本 + 等待时间
```

## 3.2 缓存策略矩阵

| 缓存层 | 缓存什么 | 过期策略 | 预估节省 |
|--------|---------|---------|---------|
| **Prompt Cache** | System Prompt 的 KV 矩阵 | 由 LLM Provider 控制（~5 分钟） | 30-50% 的输入 token 成本 |
| **Semantic Cache** | Embedding + 模型输出 | TTL 5 分钟 to 1 小时 | 重复查询 100% 节省 |
| **Embedding Cache** | text → embedding 向量 | 永久（文本不变则 embedding 不变） | 20-40% 的 Embedding API 调用 |
| **Tool Result Cache** | 工具输入 → 工具输出 | TTL 10 秒 to 1 分钟 | 高：相同参数反复调用的场景 |
| **RAG Cache** | 检索 query → 相关文档 | TTL 至知识库更新 | 30-50% 的向量搜索量 |

---

# 四、版本管理与灰度发布——Agent 的"自动驾驶版本控制"

## 4.1 Agent 的五个版本对象

```
传统软件：代码版本 = 一切
Agent：五个独立的版本对象

① 模型版本：GPT-4-0613 vs GPT-4-0125 vs Claude Opus 4.0 vs 4.5
   → 同一 Prompt 在不同模型版本下的 tool_call 行为可能完全不同

② Prompt 版本：System Prompt / Task Prompt 的每次修改
   → 加一行"优先使用 read_file 而不是 search_code"→ 工具选择分布可能剧烈变化

③ 工具版本：工具定义的变更（新增参数/修改 Schema/新增工具）
   → 工具描述变了 → 模型可能开始或停止选择这个工具

④ Skill 版本：封装的 Skill 包的 I/O、Prompt 模板、质量标准
   → 研究方法论变了 → 输出的深度和引用质量都会变

⑤ RAG 策略版本：Chunking 策略、Embedding 模型、检索参数
   → Chunk 长度从 500 → 200 → 召回率上升，但引用准确率下降
```

## 4.2 Feature Flag 控制 Agent 行为

```python
# Feature Flag 控制：哪个用户用哪个版本的 Prompt、模型、RAG 策略
class AgentFeatureFlags:
    def __init__(self):
        self.flags = {
            # ① Prompt 版本
            "prompt_version": {
                "user_a": "v3.1",       # A 用最新的 Prompt
                "user_b": "v3.0",       # B 暂留旧版
                "default": "v3.0"       # 其他人用稳定版
            },
            # ② 模型版本
            "model": {
                "canary_group": "claude-opus-4-6",  # 金丝雀组用最新模型
                "default": "claude-sonnet-4-6"
            },
            # ③ RAG 策略
            "rag_strategy": {
                "experiment_group": "v2_semantic_chunking",  # 实验组
                "default": "v1_fixed_chunking"
            }
        }
    
    def get(self, flag: str, user_id: str) -> str:
        return self.flags.get(flag, {}).get(user_id, self.flags[flag]["default"])
```

## 4.3 灰度发布流程

```
① 1% 金丝雀流量：
   新版本 Agent 只开放给内部团队 → 跑 1 天 → 观察 Eval 指标

② 10% 灰度：
   开放给 10% 的外部用户 → 跑 3 天 → 对比成功率/延迟/成本 vs 旧版

③ 50% 灰度：
   观察指标没有退化 → 扩大 50% → 跑 1 周

④ 全量：
   100% 用户切换到新版本 → 继续监控 1 周

每步都有自动回滚条件：
  - Task Success Rate 下降 > 5% → 自动回滚
  - P99 延迟 > 2x → 自动回滚
  - Token 成本 > 1.5x → 告警 + 人工判断
```

---

# 五、成本治理——Token 消耗的透明化

```python
class CostTracker:
    def __init__(self):
        self.usage = defaultdict(lambda: {"input_tokens": 0, "output_tokens": 0, "cost": 0.0})
    
    def record(self, model: str, input_tokens: int, output_tokens: int):
        pricing = {
            "claude-opus-4-6":  (15.0, 75.0),   # $/1M tokens
            "claude-sonnet-4-6": (3.0, 15.0),
            "claude-haiku-4-5": (0.8, 4.0),
        }
        input_price, output_price = pricing.get(model, (0, 0))
        
        cost = (input_tokens / 1_000_000) * input_price + \
               (output_tokens / 1_000_000) * output_price
        
        self.usage[model]["input_tokens"] += input_tokens
        self.usage[model]["output_tokens"] += output_tokens
        self.usage[model]["cost"] += cost
    
    def daily_report(self) -> str:
        """每日成本报告"""
        total_cost = sum(m["cost"] for m in self.usage.values())
        by_model = "\n".join(
            f"  {model}: {stats['input_tokens']:,} in / {stats['output_tokens']:,} out = ${stats['cost']:.2f}"
            for model, stats in sorted(self.usage.items(), key=lambda x: x[1]["cost"], reverse=True)
        )
        return f"## Daily Cost: ${total_cost:.2f}\n{by_model}"
```

---

# 六、总结

| 组件 | 解决的问题 | 核心手段 |
|------|----------|---------|
| **推理引擎** | GPU 利用率低 + 延迟高 | vLLM PagedAttention + Continuous Batching |
| **模型路由** | 所有请求用最贵模型 | 强/弱模型分层 + 模型路由策略 |
| **降级策略** | 模型挂了怎么办 | fallback 链 + 缓存兜底 |
| **缓存体系** | 重复计算浪费 Token | 五层缓存（Prompt/Semantic/Embedding/Tool/RAG） |
| **版本管理** | Agent 行为不可复现 | 五层版本对象（模型/Prompt/工具/Skill/RAG 策略） |
| **灰度发布** | 全量上线风险 | Feature Flag + 金丝雀 → 10% → 50% → 全量 |
| **成本治理** | Token 消耗不可见 | 按模型/用户/任务类型的成本追踪 + 每日报告 |

# 延伸阅读

**Do——动手部署：**
- `pip install vllm && vllm serve meta-llama/Llama-3.1-8B-Instruct` 在本地 GPU 上部署 vLLM，对比和 OpenAI API 的延迟和成本
- 用 Feature Flag（如 LaunchDarkly）控制 Agent 的 Prompt 版本，做一次 10% 灰度的 A/B Test

**Todo——深入方向：**
- Speculative Decoding——用小模型"猜"下一个 token → 大模型验证 → 加速推理 2-3x
- LoRA Adapters——同一个基础模型上挂多个 LoRA adapter，实现"一个 GPU 跑多个定制的 Agent 模型"
- 多租户推理服务的资源隔离与安全——GPU 并发 + Token 级别的权限边界

*本文参考资料：*
- vLLM: "Efficient Memory Management for Large Language Model Serving with PagedAttention" (Kwon et al., SOSP 2023)
- vLLM 官方文档: https://docs.vllm.ai/
- TGI: https://huggingface.co/docs/text-generation-inference/
- Anthropic Prompt Caching 文档
