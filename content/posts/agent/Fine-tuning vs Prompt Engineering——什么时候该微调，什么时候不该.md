---
title: "Fine-tuning vs Prompt Engineering——什么时候该微调，什么时候不该"
date: 2026-08-09
description: 从微调的本质（LoRA 如何在不改变原有权重的前提下通过旁路低秩矩阵学习新行为、全量微调 vs QLoRA 的显存与成本差异）、数据准备的完整流程（收集→清洗→格式标准化→质量审查→切分训练/验证集）、到 Fine-tuning 与 Prompt Engineering 的七维度选型矩阵（成本/延迟/可控性/泛化/维护/数据需求/适用场景），帮助 Agent 开发者做出正确的"调还是不调"决策。
tags: ["AI Agent","Fine-tuning","LoRA","Prompt Engineering","微调","QLoRA"]
categories: ["Agent"]
---

# 历史背景——Prompt Engineering 不能解决所有问题

2023 年，行业共识是"Prompt Engineering 就够了——不用微调"。到了 2024 年，共识变成了"有些场景 Prompt 打不过微调"。

**Prompt 能解决的问题**：告诉模型"做什么"和"怎么做"——角色设定、工具调用规则、输出格式、标准操作流程。这些问题本质上是"给模型更好的指令"。

**Prompt 解决不了的问题**：模型的**行为风格**不符合你的场景——它生成的代码总是 Java 1.4 的风格而不是 Java 21、它拒绝回答某些类型的问题、它的 JSON 输出不稳定。这些不是"指令清晰度"的问题，是模型在预训练阶段形成了某种行为模式，Prompt 没法完全覆盖。

微调（Fine-tuning）就是**用你的数据改变模型的"行为模式"**——不是教它新知识，而是调整它的输出风格、格式控制、领域偏好。

---

# 一、微调的本质——LoRA 怎么"只改一点点"

## 1.1 全量微调 vs LoRA

```
全量微调（Full Fine-tuning）：
  把整个模型的所有 70B 参数都在你的数据上再训练一轮
  → 需要 8×A100 80GB GPU，训练几小时到几天
  → 显存：模型参数 × 4（Adam 优化器） ≈ 280GB+
  → 成本：数千到数万美元
  → 适用：需要从根本上改变模型行为的场景

LoRA (Low-Rank Adaptation)：
  不改变原有权重——在 Transformer 的某些层旁边"接"一个小的低秩矩阵
  只有这个小矩阵被训练（原有权重冻结）
  → 可训练参数：全量的 0.1%-1%
  → 显存：原模型 + LoRA 参数（几 MB 到几十 MB）
  → 成本：数十到数百美元
  → 适用：**Agent 场景的标准选择**
```

**LoRA 的可插拔特性**：你可以在同一个基础模型上挂多个 LoRA adapter——一个用于代码生成风格调整，一个用于 JSON 格式优化，一个用于客服话术。切换 adapter 不需要重新加载基础模型。

## 1.2 QLoRA——消费级 GPU 也能微调

```
QLoRA = LoRA + 4-bit 量化

把基础模型从 16-bit 量化到 4-bit（精度略降，显存降低 4 倍）
然后在量化的模型上做 LoRA 微调

效果：
  Llama-3-70B 全量微调 → 需要 8×A100 (280GB 显存)
  Llama-3-70B QLoRA   → 2×RTX 4090 (48GB 显存) 就能跑

代价：训练速度慢 ~30%，最终精度几乎无差异
```

---

# 二、数据准备——微调的核心瓶颈不是算力，是数据质量

## 2.1 需要多少数据？

```
Prompt Engineering：0 条（不需要数据）
Few-shot：5-50 条
微调（最小可用）：50-100 条高质量数据
微调（稳定效果）：500-2000 条
微调（生产级）：5000+ 条
```

**数据质量远比数据量重要**——50 条精心挑选、人工审核过的数据比 5000 条机洗数据效果好。

## 2.2 数据格式——Chat 格式是标准

```json
{
  "messages": [
    {"role": "system", "content": "你是一个代码审查助手..."},
    {"role": "user", "content": "审查这份代码"},
    {"role": "assistant", "content": "{\"issues\": [{\"severity\": \"warning\", ...}]}"}
  ]
}
```

## 2.3 数据准备的五个步骤

```
① 收集：从你的 Agent 历史 trace 中提取
   - 人工标注"好"的回复 → 作为训练数据
   - 人工标注"差"的回复 → 分析为什么差 → 修成好的 → 也作为训练数据

② 清洗：去掉质量差的样本
   - 输出格式不符合 Schema → 要么修正，要么丢弃
   - 引用不准确 → 丢弃
   - 回复过于简略/过于啰嗦 → 修正

③ 格式标准化：所有数据统一为 Chat 格式

④ 质量审查：人工或强模型审查
   - 每一条数据都"值得被模型学习"吗？

⑤ 切分：80% 训练 / 20% 验证
```

---

# 三、七维度选型矩阵——什么时候微调，什么时候不调

| 维度 | Prompt Engineering | Fine-tuning |
|------|-------------------|-------------|
| **成本** | ~$0（工程时间） | $50-$500（GPU 机时）+ 数据标注成本 |
| **延迟** | 不变（推理时没有额外开销） | 基本不变（LoRA adapter 很小） |
| **可控性** | 中——Prompt 改了就变 | **高**——模型行为模式被改变 |
| **泛化能力** | **高**——模型保留全部能力 | 有"灾难性遗忘"风险——微调后可能丢失部分通用能力 |
| **维护成本** | **低**——改 Prompt 即可 | 高——需要持续标注新数据 + 重训 |
| **数据需求** | 0-50 条 | **50-5000 条**高质量数据 |
| **适用场景** | 指令/规则/流程/格式控制 | **输出风格/格式稳定性/领域偏好/拒绝行为修正** |

## 什么时候微调：

```
✅ Agent 的 JSON 输出格式不稳定（频繁不符合 Schema）
✅ Agent 生成的代码风格不符合团队规范（老是 Java 8 风格）
✅ Agent 对某些合理请求过度拒绝（"我不能帮你分析这个代码"）
✅ 你有很多高质量的历史对话数据（500+ 条人工审核过的）
```

## 什么时候不要微调：

```
❌ 改用 Prompt 就能解决的问题（先用 Prompt 优化一轮）
❌ 数据量不到 50 条（太少了，模型不会学到稳定行为）
❌ 数据质量不确定（机洗数据、未经人工审查）
❌ 你希望保留模型的全部通用能力（微调可能导致"灾难性遗忘"）
❌ 你的需求频繁变化（每次变化都需要新数据 + 重新训练）
```

---

# 四、微调的最小可用流程

```python
# 用 HuggingFace + QLoRA + Unsloth 微调 Llama-3-8B
# 50-100 条数据，1 小时，1 张 RTX 4090

from unsloth import FastLanguageModel
from datasets import Dataset

# ① 加载模型（4-bit 量化）
model, tokenizer = FastLanguageModel.from_pretrained(
    model_name="unsloth/Llama-3.1-8B-Instruct",
    max_seq_length=2048,
    load_in_4bit=True,
)

# ② 配 LoRA
model = FastLanguageModel.get_peft_model(
    model, r=16, lora_alpha=16, lora_dropout=0,
    target_modules=["q_proj", "k_proj", "v_proj", "o_proj"],
)

# ③ 加载数据
dataset = Dataset.from_json("agent_training_data.jsonl")
dataset = dataset.map(lambda x: tokenizer.apply_chat_template(
    x["messages"], tokenize=True
))

# ④ 训练（~30 分钟 for 100 条数据）
trainer = SFTTrainer(model=model, dataset=dataset, max_seq_length=2048)
trainer.train()

# ⑤ 保存 adapter（只有 10MB！）
model.save_pretrained("agent-lora-adapter")
```

---

# 五、总结

| 问题 | 答案 |
|------|------|
| **微调能替代 Prompt Engineering 吗？** | 不能——Prompt 控制"做什么"，微调控制"怎么做" |
| **什么时候该微调？** | 输出格式不稳定、行为风格不符、有 200+ 条高质量数据 |
| **微调需要多少钱？** | LoRA：~$50-200（单 GPU 几小时）；QLoRA：~$10-50（消费级 GPU） |
| **微调后模型会变笨吗？** | 有风险——用 LoRA + 混合数据（领域数据 + 通用数据）可以缓解 |

# 延伸阅读

**Do——动手实践：**
- 收集 50 条你的 Agent 的"好回复"和"差回复"→ 把差的修成好的 → 作为微调数据
- 用 Unsloth + QLoRA 在 Colab（免费 T4 GPU）上微调 Llama-3-8B → 对比微调前后的输出格式稳定性
- 对比同一个 task 在微调前后的表现——工具参数生成的准确率有没有提升？

**Todo——深入方向：**
- DPO (Direct Preference Optimization) ——不训生成、只训偏好：给模型看"好→坏"对比 → 强化好的行为
- 多 LoRA adapter 的热切换——同一个基础模型上挂多个 adapter，按任务类型切换
- 微调数据的自动生成——用强模型（GPT-4）生成训练数据，喂给弱模型（Llama-3-8B）微调

*本文参考资料：*
- LoRA 论文: "LoRA: Low-Rank Adaptation of Large Language Models" (Hu et al., 2021)
- QLoRA 论文: "QLoRA: Efficient Finetuning of Quantized LLMs" (Dettmers et al., 2023)
- Unsloth: https://github.com/unslothai/unsloth
