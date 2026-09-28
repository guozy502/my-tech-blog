---
title: "Agent Eval 评估体系——从单指标到可回归的评测平台设计"
date: 2026-08-08
description: 从 Agent Eval 为什么不是"跑通就行"（改 Prompt 可能退化和模型切换可能改变工具调用行为）、11 个核心评估指标（Task Success/Failure Recovery/Citation Accuracy/Hallucination Rate）、7 种评估方法（LLM-as-Judge/Golden Answer/Snapshot Test/Pairwise Comparison）、到 Trace Analysis 的 12 类失败根因分类与回归测试体系，拆解 Agent 质量治理的完整方法论。
tags: ["AI Agent","Eval","评估","LLM-as-Judge","Trace Analysis","回归测试"]
categories: ["Agent"]
---

# 历史背景——Agent Eval 为什么是"护城河"？

传统软件测试有一个明确的定义：输入 X，期望输出 Y。断言 `expected == actual`，测试通过。Agent 的问题在于——它的输出是自然语言，而且一次执行可能包含 5 次模型调用、3 次工具调用、2 次检索。你没法用 `assertEquals` 来判断"这个回答好不好"。

更深的问题是 Agent 的**非确定性退化**：
- 你改了 System Prompt 的一个短语 → 工具选择的准确率从 92% 掉到 87%，你不跑 Eval 永远不知道
- 你从 GPT-4 切到 Claude Sonnet → 代码生成质量提升了，但同一批测试中有 3 个 case 的工具参数生成格式变了
- 你优化了 RAG 的 Chunking 策略 → 检索召回率升了，但引用的准确率反而降了（因为每个 chunk 变短了，引用更难精确到行号）

Agent Eval 的存在价值是：**用数据证明 Agent 是否在变好——而不是凭感觉。**

---

# 一、Agent 和传统软件的测试有什么不同？

## 1.1 三个根本差异

```
传统软件：
  输入明确 → 输出可断言 → 一次执行 = 一个结果 → 回归测试 = 跑同一批 case → 全绿 = 通过

Agent：
  输入灵活（自然语言）→ 输出自然语言 → 一次执行 = N 次模型调用 + M 次工具调用
  → 修改 Prompt 可能破坏旧 case
  → 切换模型可能导致工具调用行为变化
  → 优化 RAG 可能降低引用准确率
  → "全绿"不存在——Agent 的 Eval 是一个"分数分布"而不是"通过/失败"
```

## 1.2 什么样的 Agent 需要 Eval

```
✅ 代码生成 Agent：生成代码的正确性、测试通过率
✅ 代码审查 Agent：Bug 检出率、误报率
✅ 研究分析 Agent：事实准确性、引用完整性
✅ 客服 Agent：答案正确率、用户满意度
✅ 安全扫描 Agent：漏报率、误报率

❌ 简单的 Chatbot：一句话回答，不需要工具调用，Eval 成本 > 收益
```

---

# 二、11 个核心评估指标

## 2.1 指标全景

```
                   ┌───────────────┐
                   │  Agent Eval   │
                   └───────┬───────┘
           ┌───────────────┼───────────────┐
    任务能力指标          质量指标          成本指标
    ┌──┴──────────┐   ┌────┴────────┐  ┌───┴──────────┐
    │Task Success  │   │Answer Corr. │  │Token Cost    │
    │Failure Recov.│   │Citation Acc.│  │Latency       │
    │Tool Call Acc.│   │Hallucination│  │Human Handoff │
    │Tool Fail Rate│   │Retrieval Rec│  │Loop Count    │
    └─────────────┘   └─────────────┘  └──────────────┘
```

## 2.2 11 个指标逐一解析

| 指标 | 计算方式 | 含义 |
|------|---------|------|
| **Task Success Rate** | 完成任务数 / 总任务数 | Agent 能否最终交付正确结果 |
| **Tool Call Accuracy** | 选对工具+参数正确的次数 / 总工具调用 | LLM 的工具选择能力 |
| **Answer Correctness** | 人工/LLM-Judge 判定正确的比例 | 答案的事实正确性 |
| **Citation Accuracy** | 引用支持结论的次数 / 总引用 | RAG 回答必须有证据支撑 |
| **Retrieval Recall** | 检索到相关文档数 / 相关文档总数 | RAG 检索质量 |
| **Hallucination Rate** | 包含编造信息的回答数 / 总回答数 | 模型的幻觉控制 |
| **Human Handoff Rate** | 需要人工接管的次数 / 总任务数 | Agent 的自主完成能力 |
| **Latency（P50/P99）** | 从任务开始到返回结果的时间 | 用户体验 |
| **Token Cost** | 每次任务的 Token 消耗 | 成本控制 |
| **Failure Recovery Rate** | 从工具失败/异常中恢复成功的次数 / 失败次数 | Agent 的鲁棒性 |
| **Regression Rate** | 新版本 Agent 比旧版本退化的 task 数 | 版本迭代质量 |

**优先级建议**：
- 第一层（必测）：Task Success Rate + Answer Correctness + Citation Accuracy
- 第二层（每周测）：Tool Call Accuracy + Hallucination Rate + Failure Recovery Rate
- 第三层（每个版本测）：Regression Rate + Latency P99 + Token Cost

---

# 三、7 种 Eval 方法

## 3.1 从简单到复杂

**① Rule-based Eval（基于规则）**

```python
# 最简单的 Eval：检查输出中是否包含特定关键词或模式
def rule_eval(result) -> bool:
    checks = [
        "PR" in result.summary,                    # 输出中提到了 PR
        result.pr_url.startswith("https://"),      # PR URL 是有效的
        '"title"' in result.json_output,            # JSON 输出有 title 字段
        result.tool_calls_count > 0,               # 至少调过一次工具
    ]
    return all(checks)
```
适用：结构化输出的格式校验、必须包含特定信息的任务。缺点：只能检查"有没有"，不能检查"对不对"。

**② Schema-based Eval（基于 Schema）**

```python
# 检查 JSON 输出是否符合预定义的 Schema
from pydantic import BaseModel, ValidationError

class PullRequestResult(BaseModel):
    pr_url: str
    title: str
    body: str
    files_changed: int
    status: str

def schema_eval(result) -> bool:
    try:
        PullRequestResult(**result.json_output)
        return True
    except ValidationError:
        return False
```
适用：所有有结构化输出的 Agent。这是自动化成本最低、ROI 最高的一种 Eval。

**③ Snapshot Test（快照测试）**

```
固定输入 + 固定上下文 → 期望输出不退化

不是"必须输出 X"，而是"之前输出 X，现在不能退化为更差的结果"

操作：
  ① 用旧版本 Agent 跑一批 Golden Task → 保存输出为 snapshot
  ② 用新版本 Agent 跑同一批 task
  ③ 对比新旧输出 → LLM-as-Judge 或人工判断是否退化
```
适用：回归测试的核心方法。每次改 Prompt/换模型/RAG 策略后必跑。

**④ Golden Answer（标准答案）**

```
为每个任务准备人工标注的标准答案。Agent 的输出和标准答案做比较。

例：
  任务："找出该项目中所有使用 SQLite 的代码位置"
  标准答案：["src/db/sqlite.py:42", "src/db/sqlite.py:89", "config/sqlite.ini:15"]
  Agent 输出：["src/db/sqlite.py:42", "src/db/sqlite.py:89"]
  → 比较：召回率 = 2/3 = 67%，精确率 = 2/2 = 100%
```

构造 Golden Dataset 的困难：需要人工标注、需要覆盖尽可能多的任务类型（代码理解/文档检索/长链推理）、需要定期更新。但它是**Eval 质量的天花板**——没有高质量的 Golden Dataset，任何评估方法都是空谈。

**⑤ LLM-as-Judge（用模型评估模型）**

```python
# 让另一个模型（Judge）判断 Agent 的输出是否正确
JUDGE_PROMPT = """
你是一个严格的评审。给定一个 Agent 的任务和输出，判断：
1. 任务是否完成？（是/否/部分）
2. 输出中的引用是否支持结论？（全部支持/部分支持/不支持）
3. 结论是否正确？（正确/基本正确/错误）
4. 是否存在幻觉？（无/轻微/严重）

任务：{task}
Agent 输出：{output}
上下文：{context}
"""
```

LLM-as-Judge 的可靠性取决于两个因素：

| 条件 | 说明 |
|------|------|
| **Judge 模型至少和 Agent 模型同级或更强** | 弱模型判断强模型容易"看不懂" |
| **评估维度清晰且可验证** | "有没有幻觉"比"答案好不好"更适合 LLM-as-Judge |
| **人工校验 10-20% 的 Judge 判断** | 建立 Judge 模型的可信度基线 |
| **逐条评估，不是总体给分** | "引用是否支持结论"比"答案好不好"更可靠 |

**什么时候 LLM-as-Judge 不可靠？**
- 任务本身需要领域专家才能判断（如医学、法律）
- 评估标准高度主观（"代码风格好坏"）
- Judge 模型的能力弱于 Agent 模型 → 可能"看不懂就说好"

**⑥ Human Eval（人工评估）**

最高质量、最高成本。适用：Golden Dataset 的初始标注、LLM-as-Judge 的校验样本（10-20% 采样）、争议 case 的仲裁。

**⑦ Pairwise Comparison（两两对比）**

```
不是打绝对分，而是比较："版本 A 和版本 B 的同一 task 输出，哪个更好？"

操作：
  ① 用版本 A 和版本 B 分别跑同一批 100 个 task
  ② LLM-as-Judge 或人工比较：A 更好 / B 更好 / 一样
  ③ 统计 A 赢了多少、B 赢了多少、平局多少
  
优点：比绝对分数更敏感——即使 A 和 B 都是"合格"，也能判断哪个更优
```

---

# 四、Trace Analysis——失败发生在哪一步？

Agent 的一次执行不是"一次模型调用"，而是"一次任务 → N 次评估 → M 次工具调用"。当 Agent 失败时，需要知道**失败发生在哪一步**。

## 4.1 12 类失败根因

| 失败类别 | 典型表现 | 定位方法 |
|---------|---------|---------|
| **Prompt 问题** | 指令不清晰、约束冲突、输出格式漂移 | 对比不同 Prompt 版本的同 task 成功率 |
| **上下文问题** | 关键信息缺失、无关信息过多、上下文污染 | Context Debug Report 分析 |
| **检索问题** | 召回到的文档不相关、排序错误、引用来源不可靠 | RAG recall/precision 监控 |
| **工具选择错误** | 模型选错了工具，或在错误的时机调用工具 | tool_call_accuracy 统计 |
| **参数错误** | 字段缺失、类型错误、值不合法 | Schema Validation 失败日志 |
| **权限拦截** | Agent 尝试执行超出权限的操作 | audit.log 审计日志 |
| **工具执行失败** | 超时、接口异常、外部服务不可用 | tool_failure_rate 监控 |
| **Loop 死循环** | 重复计划、重复调用工具、无法进入终止状态 | 最大循环次数限制 + loop_count 监控 |
| **子 Agent 委托错误** | 任务目标不清、上下文不足、权限配置错误 | 子 Agent 的 task_success_rate 独立追踪 |
| **模型能力不足** | 推理、代码理解或长上下文处理能力不足 | 对比同 task 在不同模型下的成功率 |
| **最终答案幻觉** | 结论没有证据支撑 | LLM-as-Judge 逐条验证引用 |
| **评测误判** | Eval 规则或 Judge 模型判断错误 | 人工抽样校验 10% Judge 判断 |

## 4.2 Trace 诊断流程

```
① Agent 执行失败 → ② 拉取这个 task 的完整 Trace
  → ③ 定位失败的阶段（Planning? 工具选择? 工具执行? 输出?）
  → ④ 归类失败类型
  → ⑤ 如果是新出现的失败类型 → 加到 Golden Dataset
  → ⑥ 如果是已知类型的退化 → 回滚最近的变更
```

---

# 五、Agent Eval 平台设计

## 5.1 平台核心结构

```
Eval Platform 的组件：

  Golden Dataset Manager（任务集管理）
    → 按任务类型分类（代码理解/代码修改/文档检索/长程任务）
    → 每个 task：输入、期望输出(golden answer)、评估方法、上下文
  
  Eval Runner（批量运行）
    → 对每个 task 跑 Agent → 记录 Trace → 记录输出
    → 支持并行运行、超时终止、失败重试
  
  Judge Engine（评估引擎）
    → Rule-based / Schema / LLM-as-Judge / Human Eval
    → 每个 task 可以指定不同的评估方法
  
  Report Generator（报告生成）
    → 成功率 / 成本 / 延迟 / 工具调用准确率 / 幻觉率
    → 回归对比：版本 A vs 版本 B
    → 失败样本聚类：哪些 task 类型容易失败
```

## 5.2 最小可用的 Eval 平台（~200 行 Python）

```python
class AgentEvalPlatform:
    def __init__(self, dataset: list[EvalTask], judge_model: str):
        self.dataset = dataset
        self.judge_model = judge_model
    
    def run_eval(self, agent_config: AgentConfig) -> EvalReport:
        """对给定的 Agent 配置跑全量评估"""
        results = []
        for task in self.dataset:
            # ① 跑 Agent
            trace, output = self.run_agent_with_trace(task, agent_config)
            
            # ② 评估
            eval_result = self.judge(task, output, trace)
            
            results.append({
                "task_id": task.id,
                "task_type": task.type,
                "output": output,
                "trace": trace,
                "eval": eval_result
            })
        
        # ③ 生成报告
        return self.generate_report(results)
    
    def run_regression(self, old_config, new_config) -> RegressionReport:
        """回归测试：旧版本 vs 新版本"""
        old_results = self.run_eval(old_config)
        new_results = self.run_eval(new_config)
        
        regressions = []
        for old, new in zip(old_results, new_results):
            if old.eval.success and not new.eval.success:
                regressions.append({
                    "task_id": old.task_id,
                    "old_output": old.output,
                    "new_output": new.output,
                    "reason": self.diagnose_regression(old, new)
                })
        
        return RegressionReport(
            old_score=old_results.avg_score,
            new_score=new_results.avg_score,
            regressions=regressions,
            improvements=[...]
        )
```

---

# 六、Eval 的工程节奏——什么频率测什么

```
每次代码变更（改 Prompt/工具/模型）：
  → Schema Eval（最快，2 分钟跑完）
  → Rule-based Eval
  
每天：
  → 全量 Golden Dataset Eval（100-500 个 task）
  → Task Success Rate + Tool Call Accuracy
  
每周：
  → Regression Test（旧版本 vs 新版本全量对比）
  → LLM-as-Judge（100-200 个 task 抽样）
  
每个版本发布前：
  → Human Eval（10-20% Golden Dataset 人工复核）
  → Pairwise Comparison（新旧版本逐条对比）
  → 回归报告 + 失败样本分析 + 新类型 case 录入 Golden Dataset
```

---

# 七、总结

| 问题 | 答案 |
|------|------|
| **Agent Eval 为什么不是"跑通就行"？** | 非确定性退化——改 Prompt 可能破坏旧 case，换模型可能改变工具调用行为 |
| **最核心的 3 个指标？** | Task Success Rate + Answer Correctness + Citation Accuracy |
| **最快的 Eval 方法？** | Schema-based（校验 JSON Schema）——2 分钟跑完 |
| **LLM-as-Judge 什么时候不可靠？** | 领域专家判断、高度主观标准、Judge 模型弱于 Agent 模型 |
| **回归测试怎么跑？** | Snapshot Test：新版本跑旧版本的 Golden Task → 对比输出是否退化 |

# 延伸阅读

**Do——动手搭建：**
- 准备 20 个 Golden Task（10 个代码理解 + 10 个文档检索），写 Schema Eval 和 Rule-based Eval 脚本
- 用 LLM-as-Judge 跑一遍 20 个 task，人工复核其中 5 个，计算 Judge 和人工判断的一致率
- 改 Agent 的 System Prompt（去掉一行约束），重新跑 Eval → 看成功率有没有退化

**Todo——深入方向：**
- RAGAS 框架——专门评估 RAG 系统的开源工具（Faithfulness / Answer Relevancy / Context Precision / Context Recall）
- Agent 的 A/B Test——生产环境中如何对真实用户流量做 A/B，而不是在离线 benchmark 上比较
- 失败样本的自动聚类——用 Embedding 把失败 task 的 Trace 向量化 → 相似失败聚类 → 定位系统性问题

*本文参考资料：*
- RAGAS: https://docs.ragas.io/
- Anthropic Eval Cookbook: https://github.com/anthropics/anthropic-cookbook/tree/main/eval
- OpenAI Evals: https://github.com/openai/evals
