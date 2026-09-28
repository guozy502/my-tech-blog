---
title: "Agent 面试高频考点——从 LLM 基础到生产落地的完整问答框架"
date: 2026-08-09
description: 从 LLM 基础（Tokenization/Embedding/Temperature）、Prompt Engineering（分层设计/Few-shot/CoT）、Function Calling 与 Tool Runtime（注册/校验/权限/MCP）、RAG 检索链路（Chunking/Hybrid Search/Rerank）、Agent Loop 与状态机、记忆系统三层架构、Multi-Agent 协作模式、Agent Eval 评估体系、到生产部署与安全防护，按 10 个模块梳理 Agent 开发工程师面试最高频考点与标准回答框架。
tags: ["AI Agent","面试","Agent","RAG","Function Calling","Prompt","Eval"]
categories: ["Agent"]
---

## 模块一：LLM 基础

**Q1: Token 是什么？为什么中英文的 token 消耗不一样？**

Token 是模型理解和生成文本的最小单位——不是"字"也不是"词"。BPE（Byte Pair Encoding）算法通过统计训练文本中相邻字符对的频率，把高频组合合并成一个 token。"苹果"在中文中可能是 1-2 个 token，英文 "apple" 是 1 个 token，但"你好世界"在 Claude tokenizer 中可能被拆成 3 个 token（"你"/"好"/"世界"），在 GPT-4 中可能是 2 个（"你好"/"世界"）。同一个中文句子，不同模型的 token 消耗可能差 1.5-2 倍。

**追问：Agent 开发中什么时候需要关心 token 消耗？**

工具定义的描述、System Prompt、历史对话——这些每次请求都携带。工具越多 → System Prompt 中工具定义越大 → 每轮 token 消耗越多。优化手段：精简工具描述、压缩历史对话、对高频操作考虑英文 Prompt（比中文省 30-50% token）。

---

**Q2: Temperature 和 Top-P 的区别？Agent 工具调用时应该用多少？**

Temperature 控制概率分布的"平坦度"——0 = 确定性（永远选择概率最高的 token），1.0 = 按原始概率采样，>1.0 = 拉平分布让低概率 token 也有机会。Top-P（核采样）从累积概率达到 P 的最小 token 集合中采样。两者不冲突——通常 Temperature=0 时 Top-P 不起作用，Temperature > 0 时 Top-P 控制候选范围。

Agent 工具调用时必须设 Temperature=0 或接近 0——你需要工具参数确定性地正确，不能因为温度导致 `city` 变成 `Beijingg`。

---

**Q3: Embedding 模型的选择对 RAG 有什么影响？**

Embedding 模型决定了"语义相似度"的向量空间。换了 Embedding 模型 → 整个向量空间变了 → 已有的索引全部失效，必须重建。中文 RAG 场景首选 `bge-large-zh-v1.5`（BAAI），多语言场景用 `multilingual-e5-large`。同一个 Embedding 模型索引和查询——如果索引用 A 模型、查询用 B 模型，相似度结果毫无意义。

---

## 模块二：Prompt Engineering

**Q4: 一个 Agent 的 System Prompt 应该包含哪些部分？**

六层分层设计：① 角色定义（200 tokens）——"你是一个资深 Java 代码审查助手"；② 行为约束（300 tokens）——"永远引用源代码行号、优先 read_file 再 search_code"；③ 工具定义（~1000 tokens）——可用工具的名称/用途/参数/返回格式；④ 操作流程（500 tokens）——代码审查的标准步骤；⑤ 输出格式（200 tokens）——"最终输出是 JSON，包含 summary/severity/findings"；⑥ 安全边界（200 tokens）——"不修改配置、不删除文件、不暴露密钥"。

**追问：如果 Prompt 改了一句话后 Agent 的行为退化了，你怎么排查？**

用 Snapshot Test——改 Prompt 之前跑 50 个 Golden Task，保存输出快照。改 Prompt 后用同样的 50 个 task 重跑，逐条对比输出的差异和成功率。超过 10% 的退化 → 回滚 Prompt。

---

**Q5: Few-shot 示例怎么选？Chain-of-Thought 什么时候用？**

Few-shot 三条原则：① 覆盖多样性——简单 case + 复杂 case + 边界 case；② 展现边界行为——"如果输入模糊，默认假设 X"；③ 从历史失败样本中提取——模型上次在哪里出错了，就把错误 case 的正确写法做成 Few-shot。

CoT 适用于需要多步推理的任务——代码分析、Bug 诊断、复杂报告生成。Agent 工具调用场景用的是 ReAct（Reasoning + Acting），不是纯 CoT——思考一步 → 调一个工具 → 观察结果 → 思考下一步，而不是"先想好全盘再行动"。

---

## 模块三：Function Calling 与 Tool Runtime

**Q6: Function Calling 的工作机制是什么？模型怎么"知道"该调哪个工具？**

你把工具定义（名称、描述、参数 JSON Schema）放在 System Prompt 中。模型在生成回复时，如果判断"这个问题需要调用工具才能回答"，会生成一个特殊的 `function_call` 字段，包含 `tool_name` 和 `arguments`（符合你定义的 JSON Schema）。关键：模型不是"自己知道答案"，而是"知道该调哪个工具"——工具执行完返回结果后，模型再把结果和原始问题一起理解，生成最终回复。

**追问：模型选错了工具或者生成了不存在的参数，怎么办？**

参数校验在 Runtime 层拦截——Pydantic 校验 → 不符合 Schema → 把错误反喂给模型 → 模型修正参数后重新调用。工具选错的情况——增大工具的 description 和参数描述，使模型更清楚地理解工具的适用场景。

---

**Q7: Tool Runtime 需要哪些治理能力？**

八件套：① Tool Registry——工具注册、发现、版本管理；② Schema Validation——参数校验 + Pydantic 模型；③ Permission Control——RBAC 四级权限（READ/WRITE/HIGH_RISK/NEEDS_APPROVAL）；④ Risk Assessment——只读/写/高风险分级；⑤ Audit Log——每次调用的输入/输出/身份/TraceID；⑥ Timeout & Retry——超时 → exponential backoff → 降级 fallback；⑦ Trace——全链路追踪；⑧ MCP 协议——工具接入标准化。

---

**Q8: MCP 协议和 Function Calling 是什么关系？**

分工不同：Function Calling 管"模型如何选择工具"——在 System Prompt 中声明工具定义，模型生成 `function_call`。MCP 管"工具如何标准化接入"——工具提供方按 MCP 协议暴露工具，Agent 通过 MCP Client 统一接入，不需要为每个工具写单独的 Adapter。FC 关心"选哪个"，MCP 关心"怎么连"。

---

## 模块四：RAG 检索增强

**Q9: RAG 的完整链路是什么？每一步为什么要这么做？**

文档解析（PDF/Markdown/HTML 清洗）→ Chunking（切分——固定长度/语义/AST 分块，Overlap 防截断）→ Embedding（向量化——同一模型索引+查询）→ 存储（向量数据库——相似度搜索）→ 检索（Query 向量化 → Top-K 相似文档）→ Rerank（Cross Encoder 二次精排）→ 生成（LLM 基于检索结果回答 + 引用来源）。每一步的"为什么"：Chunking 决定检索粒度；Embedding 决定语义对齐；Rerank 决定最终精度；引用来源决定可信度。

**追问：检索到的文档和问题不相关，可能是什么原因？怎么排查？**

四个可能：① Embedding 模型不适合该领域（换模型）；② Chunk 太大或太小（调整 chunk size）；③ Query 措辞和文档措辞差异太大（Query Rewrite 改写查询）；④ 向量相似度 ≠ 语义相关性（加 Metadata Filter 或 Hybrid Search 结合 BM25 关键词过滤）。

---

**Q10: 为什么需要 Hybrid Search？Rerank 的作用是什么？**

向量检索找"语义相似"的文档，BM25 找"精确关键词匹配"的文档——两者互补。向量检索可能因为训练数据偏差而漏掉关键词匹配的文档，BM25 可能因为表述差异而漏掉语义相关的文档。合并后 Rerank（Cross Encoder 或 LLM Rerank）对候选集做二次精排——不是"从数百万文档中找 Top-5"，而是"从 20 个候选文档中精挑 Top-5"。

---

## 模块五：Agent Loop 与状态机

**Q11: Agent Loop 的 ReAct 模式是怎么工作的？**

ReAct = Reasoning + Acting——观察（Observe）当前状态 → 思考（Think）下一步做什么 → 行动（Act）调用工具 → 观察工具返回结果 → 循环，直到达成目标或触发终止条件。不是"先想好全盘计划再执行"，而是"边想边做、根据工具结果调整下一步"。终止条件：目标达成 / 最大循环次数 / 连续失败 / 用户取消。

**追问：Agent 陷入无限循环怎么办？**

三道防线：① 硬限制——max_loops=15，超过强制 finalize；② 循环检测——同一个工具连续调超过 5 次 → 注入警告上下文"你已经搜索了 5 次，请基于现有结果开始执行"；③ 状态机 Checkpoint——每 5 步保存一次状态，中断后从 Checkpoint 恢复，不需要从头重跑。

---

**Q12: Agent 什么时候需要 Planning？什么时候不需要？**

简单任务（一句话能完成、不需要多步骤、不需要工具调用结果来决定下一步）→ 不需要 Planner。复杂任务（需要多个步骤、每个步骤依赖前一步的结果、需要 Sub-Agent 委托）→ 需要 Planner。判断标准：你可以一眼看出怎么执行 → 固定 Workflow；你不确定怎么执行，需要 Agent 自己推理 → Agent Loop + Planner。

---

## 模块六：记忆系统

**Q13: Agent 的记忆系统分哪几层？**

三层：① 工作记忆（Working Memory）——当前任务的目标、已完成步骤、工具结果、未解决问题，存在上下文中，任务结束清空；② 短期记忆——当前会话/对话的上下文，由上下文窗口承载；③ 长期记忆——用户偏好、历史任务、常用工具、项目约定，存外部存储（向量库/关系库），任务开始时检索注入上下文。项目级记忆——目录结构、技术栈、构建命令、编码规范、常见故障。

**追问：Agent 的上下文超出了 Token 预算怎么办？**

三道防线：① 工具结果只保留最近 3 轮——更早的压缩为一行摘要；② 检索结果只保留 Top-5 最高相关度的 chunk；③ 已完成不再需要的部分——生成摘要注入 Memory。目标：不是"能塞多少就塞多少"，而是"哪些信息值得被模型看到"。

---

## 模块七：Multi-Agent

**Q14: Multi-Agent 的常见协作模式有哪些？**

四种模式：① Supervisor Pattern——一个主控 Agent 拆解任务、调度、结果收集和最终决策；② Planner/Executor——Planner 制定计划，Executor 负责具体执行；③ Reviewer Pattern——引入审查 Agent 对结果复核，降低幻觉和遗漏；④ 并行+串行——独立子任务并行执行，有依赖的串行。隔离机制：每个子 Agent 需要上下文隔离、工具隔离、权限隔离、超时隔离。

**追问：多个 Agent 的结论互相冲突，怎么仲裁？**

三级仲裁：① Supervisor Agent 比较证据质量 → 选更强的一方；② Reviewer Agent 只复核不分析 → 基于证据质量判断哪方更可靠；③ 如果前两级无法裁决 → Human-in-the-loop → 把两个结论和证据一起提交给用户决策。

---

## 模块八：Agent Eval

**Q15: Agent 和传统软件在测试上有什么根本差异？**

三个根本差异：① Agent 输出是自然语言，不能用 `assertEquals` ——需要语义评估；② Agent 的一次执行包含 N 次模型调用 + M 次工具调用——需要全链路 Trace 才能定位失败根因；③ 非确定性退化——改 Prompt/换模型/优化 RAG 都可能导致旧 case 退化——需要回归测试。

**追问：LLM-as-Judge 什么时候可靠？什么时候不可靠？**

可靠条件：Judge 模型至少和 Agent 模型同级或更强；评估维度清晰且可验证（"引用是否支持结论"比"答案好不好"更可靠）；人工抽查 10-20% 的 Judge 判断校验可信度。不可靠条件：任务需要领域专家判断；评估标准高度主观；Judge 模型弱于 Agent 模型。

---

**Q16: Agent 最核心的 5 个评估指标是什么？**

① Task Success Rate——任务是否最终完成；② Answer Correctness——答案的事实正确性；③ Citation Accuracy——引用是否支持结论；④ Hallucination Rate——包含编造信息的回答比例；⑤ Tool Call Accuracy——工具选择+参数正确的比例。每周观测：Failure Recovery Rate、Latency P99、Token Cost。

---

## 模块九：生产部署

**Q17: Agent 的生产部署和传统微服务有什么不同？**

传统微服务：代码版本 = 一切。Agent：五个独立的版本对象——① 模型版本（GPT-4-0613 vs Claude Opus 4.5）；② Prompt 版本（System Prompt 每次修改）；③ 工具版本（Schema/参数/权限变更）；④ RAG 策略版本（Chunking/Embedding/检索参数）；⑤ Skill 版本（封装的能力包）。任何一个变了 → Agent 的行为可能退化。灰度发布流程：金丝雀(1%) → 10% → 50% → 全量，每步跑 Eval 对比指标，自动回滚条件（成功率降>5% / P99延迟>2x / Token成本>1.5x）。

---

**Q18: 怎么降低 Agent 的模型调用成本？**

四层优化：① 模型路由——强模型做规划和代码生成，弱模型做工具参数生成和格式校验；② 五层缓存——Prompt Cache（System Prompt KV 复用）、Semantic Cache（相似查询复用）、Embedding Cache（同一文本的向量复用）、RAG Cache（相同检索结果复用）、Tool Result Cache（同参数短时间复用）；③ 上下文压缩——工具结果只保留最近 3 轮，检索截断 Top-5；④ 降级链——主模型超时 → 弱模型 fallback → 缓存兜底。

---

## 模块十：安全

**Q19: Prompt Injection 为什么在 Agent 中比在 ChatBot 中更危险？**

ChatBot 没有工具——注入只能让它"说错话"。Agent 有工具——注入可能诱导它执行 `shell.exec("rm -rf /")`、把文件内容发到外部 URL、调用 API 修改数据库。安全防线不能依赖 Prompt——"在 System Prompt 中写不要被注入"本身就是 Prompt，也可能被覆盖。真正的防线在 Runtime 层——权限控制 + 工具白名单 + 参数清理 + 审计日志。

**追问：Agent 安全的三层纵深防御是什么？**

Prompt 层（分隔符隔离 + 输入清理 + 角色约束）→ Runtime 层（工具白名单 + 权限控制 + SQL/路径注入防护 + 高风险工具人工确认）→ 输出层（敏感信息脱敏 + URL 白名单 + 内容安全过滤）。Prompt 是建议，Runtime 是强制，输出是兜底。

---

## 面试考察能力矩阵

| 级别 | 考察重点 | 典型问题 |
|------|---------|---------|
| **入门** | LLM 基础 + Prompt + Function Calling | Token 是什么？Temperature 设多少？Function Calling 怎么工作？ |
| **进阶** | RAG + Agent Loop + 记忆系统 | RAG 的全链路是什么？Agent Loop 的终止条件怎么设计？ |
| **资深** | Eval + 安全 + 平台架构 | Agent 怎么评估？提示注入怎么防？五个版本对象怎么管理？ |
| **专家** | 自定义 Agent 平台设计 + 推理优化 + 成本治理 | 从零设计一个 Agent 平台需要哪些组件？怎么降低 70% 的 Token 成本？ |

---

*本文覆盖了 Agent 面试中最核心的 19 个考点。需要深入某个模块的内容，可参考本系列对应文章。*
