---
title: "Multi-Agent 系统——多Agent协作的架构模式与实践"
date: 2026-07-29
description: 从顺序、层级、辩论、投票四大协作模式拆到LangGraph、AutoGen、CrewAI三大框架对比，覆盖生产环境中错误级联、对话循环、token爆炸等真实陷阱。
tags: ["Multi-Agent","LangGraph","AutoGen","CrewAI","协作模式","Agent"]
categories: ["agent"]
---

# Multi-Agent 系统——多Agent协作的架构模式与实践

> 单 Agent 能做的事有限。真正复杂的任务需要拆解、分工、辩论、互相校验。Multi-Agent 是 Agent 领域的进阶话题，也是面试中能让你脱颖而出的深度考点。本文从协作模式拆到框架选型，覆盖顺序、层级、辩论、投票四大模式，以及生产环境中真正会遇到的坑。

---

## 一、为什么要 Multi-Agent

### 1.1 单 Agent 的天花板

单 Agent（即使配备了工具和记忆）面对复杂任务有几个结构性瓶颈：

- **上下文窗口承载不了整个任务链**：一个复杂任务可能涉及数十个子步骤，每个的中间结果都塞进上下文，很快 token 爆炸
- **一块 LLM 无法同时做好所有角色**：理解用户意图、规划任务、写代码、检查质量——让同一个 prompt 同时做好这些，prompt 会变得臃肿且互相冲突
- **缺乏独立校验**：单 Agent 的输出只有自己能检查，但"自己查自己"本身就不可靠

### 1.2 Multi-Agent 解决什么

Multi-Agent 的核心思想：**把复杂任务拆给多个专门化的 Agent，各司其职 + 互相校验**。

典型的价值场景：

| 场景 | 单Agent 做法 | Multi-Agent 做法 |
|------|------------|-----------------|
| 代码 Review | Agent 写代码后自查 | Coder Agent 写 → Reviewer Agent 审 → 反馈修改 |
| 复杂研究报告 | 一个 Agent 硬写 | Researcher 收集资料 → Analyst 分析 → Writer 撰写 → Editor 润色 |
| 事实核查 | 依赖 LLM 自身不稳定的判断 | FactChecker Agent 逐一验证关键断言，标记存疑内容 |
| 多方案决策 | 一步到位出方案 | 多个 Agent 分别从成本/技术/风险角度出方案 → 汇总比较 |

---

## 二、四大协作模式

### 2.1 顺序流水线 (Sequential Pipeline)

最基础的模式：A 的输出是 B 的输入，像工厂流水线。

```
用户输入 → [Agent A: 收集信息] → [Agent B: 分析整合] → [Agent C: 生成输出] → 结果
```

**适合**：步骤固定、有明确上下游依赖的任务。如"给定一个 GitHub 仓库，生成一份架构分析报告"——克隆代码 → 分析结构 → 写报告 → 排版。

**关键设计**：
- Agent 之间的传递格式要标准化。Agent A 输出 JSON 或结构化文本，Agent B 的 prompt 里明确写"输入格式：..."
- 上下文隔离：Agent A 的所有中间推理不应该进入 Agent B 的上下文——只传**结构化输出**（如搜索结果列表），不传 Thought 过程

### 2.2 层级委派 (Hierarchical Delegation)

类似管理结构：一个 Manager Agent 拆分任务，分配给多个 Worker Agent，最后汇总结果。

```
                 ┌─────────────┐
                 │   Manager   │  ← 理解目标，拆解任务，协调结果
                 └──┬───┬───┬──┘
                    │   │   │
          ┌─────────┘   │   └─────────┐
          ▼             ▼             ▼
    ┌──────────┐ ┌──────────┐ ┌──────────┐
    │ Worker A │ │ Worker B │ │ Worker C │  ← 各自独立执行子任务
    └──────────┘ └──────────┘ └──────────┘
```

**适合**：任务可并行拆分的场景。如"对比三家云厂商的 Kubernetes 服务价格"——Manager 把任务拆成三个独立工作包，三个 Worker 并行查询各厂商，Manager 汇总对比。

**Manager 的关键职责**：
1. 任务拆解 (Decomposition)：把大目标拆成互相独立的子任务
2. 分配 (Assignment)：根据 Worker 的能力描述匹配合适的 Agent
3. 整合 (Synthesis)：合并 Worker 的结果，识别冲突，给出最终方案
4. 异常处理：Worker 失败时的补偿策略（重试 / 换人 / 降级）

**Worker 的设计原则**：
- 每个 Worker 有明确的 **角色 profile** + **能力清单**，Manager 据此做分配
- Worker 之间不应直接通信——所有协调经 Manager 中转，避免"网状"通信
- Worker 的上下文干净：只看到 Manager 分配的子任务，看不到其他 Worker 的工作

### 2.3 辩论模式 (Debate)

多个 Agent 持不同立场或方法论，各自论证后达成共识。

```
       ┌──────────┐
       │  Agent A │ → 观点 1 + 论据
       └────┬─────┘
            │ 互相看到对方的论证
       ┌────┴─────┐
       │  Agent B │ → 观点 2 + 论据（可能是对观点 1 的反驳）
       └────┬─────┘
            │
       ┌────┴─────┐
       │  Judge   │ → 汇总双方论证，做出最终决策
       └──────────┘
```

**适合**：需要多角度审视的决策型任务，如技术方案评审、法律风险评估。

**效果的核心前提**：Agent 之间必须**有信息不对称**——如果两个 Agent 用同一个 prompt、同一套知识，它们的"辩论"只是表演。真正的辩论价值来自于不同的角色设定：
- Agent A："你是极度关注性能的架构师"
- Agent B："你是极度关注可维护性的 Tech Lead"
- Judge："你是 CTO，需要综合考虑后做决策"

### 2.4 并行投票 (Parallel Voting)

同一个问题给多个 Agent，各自独立生成答案，最终投票或择优。

```
用户问题 → ┬─ Agent 1 → 答案 A ─┐
           ├─ Agent 2 → 答案 B ─┼─→ 投票/评估 → 最终答案
           └─ Agent 3 → 答案 C ─┘
```

**两种主要变体**：

**多数投票 (Majority Voting)**：
- 适合有"客观答案"的问题（数学题、代码选择题）
- 3 或 5 个 Agent 各自解题，取最多人同意的答案
- 实操中通常用**相同 LLM + 不同 temperature + 不同 few-shot 示例**制造多样性

**择优 (Best-of-N with LLM Judge)**：
- 适合"质量判断"类任务（写作、翻译、创意）
- N 个 Agent 各自输出，一个 Judge Agent 评估并选出最优
- 关键：Judge 必须有清晰的评估维度和打分标准

---

## 三、框架对比

### 3.1 LangGraph —— 有向图编排

LangGraph 把 Multi-Agent 系统建模为一个**有向图**：节点是 Agent 或函数，边是数据流 + 条件路由。

```python
from langgraph.graph import StateGraph

graph = StateGraph(AgentState)

graph.add_node("researcher", research_agent)
graph.add_node("analyst", analysis_agent)
graph.add_node("writer", writer_agent)

graph.add_edge("researcher", "analyst")          # 顺序
graph.add_conditional_edges(
    "analyst",
    decide_next,                                  # 条件分支
    {"continue": "writer", "research_more": "researcher"}
)

graph.set_entry_point("researcher")
app = graph.compile()
```

**核心优势**：
- **状态管理**：State 在整个图中流转，每个节点可以读写 State（对记忆系统非常友好）
- **灵活性**：因为有向图是通用抽象，可以实现任何协作模式
- **Human-in-the-loop**：原生支持中断 + 人工审批
- **可观察性**：图的每次执行都是可追踪的

**适合**：需要精确控制执行流程的生产级 Agent 系统。

### 3.2 AutoGen —— 对话驱动协作

微软的 AutoGen 把 Agent 之间的协作建模为**对话**。Agent 之间通过发消息来协调工作。

核心概念：
- **ConversableAgent**：能收发消息的 Agent 基类
- **GroupChat**：多个 Agent 在同一个群聊中，轮流发言
- **GroupChatManager**：决定下一个该谁发言（基于规则或 LLM 决策）

```python
from autogen import AssistantAgent, GroupChat, GroupChatManager

researcher = AssistantAgent("researcher", system_message="你是研究员...")
analyst = AssistantAgent("analyst", system_message="你是数据分析师...")
user_proxy = UserProxyAgent("user", human_input_mode="TERMINATE")

groupchat = GroupChat(
    agents=[user_proxy, researcher, analyst],
    messages=[],
    max_round=10,
    speaker_selection_method="auto"  # LLM 自动决定下一个发言者
)
manager = GroupChatManager(groupchat)
```

**核心优势**：
- **简单直观**：对话是所有 Agent 都能理解的通信协议
- **灵活的发言权控制**：可以手动指定"下一轮谁说话"，也可以让 LLM 自动选择
- **渐进式复杂度**：从两个 Agent 对话到 N 个 Agent 群聊都是同一套 API

**局限**：
- 对话历史增长极快（所有 Agent 的消息都共享上下文）
- 群聊中可能出现"两个 Agent 来回聊，其他 Agent 插不上话"的问题
- 不适合需要严格流程控制的场景（对话天然是灵活的，图是精确的）

### 3.3 CrewAI —— 角色化层级协作

CrewAI 把 Multi-Agent 协作抽象为 **Crew (团队) + Task (任务) + Agent (成员)** 三层模型：

```python
from crewai import Agent, Task, Crew

researcher = Agent(
    role="研究员",
    goal="收集市场数据",
    backstory="你是经验老道的市场分析师，擅长从公开数据中发现趋势"
)

analyst = Agent(
    role="数据分析师",
    goal="从数据中提炼洞察",
    backstory="..."
)

# Task 之间有依赖声明
research_task = Task(description="收集三家竞品的定价信息", agent=researcher)
analysis_task = Task(description="基于收集的信息做对比分析", agent=analyst,
                     context=[research_task])  # ← 依赖声明

crew = Crew(agents=[researcher, analyst], tasks=[research_task, analysis_task])
result = crew.kickoff()
```

**核心优势**：
- **角色设定极其自然**：role + goal + backstory 的模板让非技术人员也能定义 Agent
- **任务依赖显式声明**：Task 的 context 参数表达依赖，框架自动处理执行顺序
- **开箱即用的层级结构**：Crew 默认按 Manager → Worker 模式运行

**局限**：
- 定制化程度不如 LangGraph（如果一种协作模式框架没提供，就很难实现）
- 黑盒较多，Debug 不如 LangGraph 直观

### 3.4 框架选择指南

| 场景 | 推荐框架 | 原因 |
|------|---------|------|
| 复杂流程图、Human-in-the-loop | LangGraph | 图模型 + 状态管理 + 可中断 |
| 快速原型、对话式协作 | AutoGen | 对话驱动，上手简单 |
| 角色分工明确、层级任务 | CrewAI | 角色化抽象，任务依赖天然 |
| 多 Agent + RAG + 工具多 | LangGraph | 组件可插拔，定制性最强 |
| 非技术人员定义 Agent 流程 | CrewAI | Backstory 写法自然 |

**一个常见的组合**：用 LangGraph 做整体编排（因为它抽象能力最强），在具体节点内嵌入 CrewAI 或 AutoGen 风格的 Agent 定义。

---

## 四、Multi-Agent 的坑

### 4.1 错误级联放大

A Agent 给 B Agent 传了一个看起来合理但实际错误的结果。B 基于这个错误结果继续工作，错误在链条上不断放大。

```
Agent A (搜索): "竞品 X 的价格是 1000 元/月" (实际是 100 元，多看了个零)
Agent B (分析):  "竞品 X 是高价定位，我们的价格优势明显"
→ 最终报告基于一个根本不成立的事实
```

**应对**：
- **交叉验证 Agent**：关键节点（如数据收集）后跟一个 Validator Agent，抽样核实
- **不确定性标注**：Agent 输出时标注置信度，"这条价格信息来自一个搜索结果，置信度中等"
- **溯源链**：每个 Agent 的输出附带来源信息，下游 Agent 可追溯验证

### 4.2 对话循环

两个 Agent 陷入无限修正循环：

```
Agent A: "代码有个 bug 在第 15 行"
Agent B: "修复了"
Agent A: "第 15 行改后第 23 行又出问题了"
Agent B: "修复了第 23 行"
Agent A: "第 15 行的修法不对，改回来"
Agent B: "改回来了，第 23 行呢？"
... (无限循环)
```

**应对**：
- **最大轮次限制**：硬性上限，Manager 在达到限制后汇总当前状态做最终决策
- **改进度追踪**：每次修改后评估"距离目标更近了吗"，如果连续 N 轮没有进展，触发升级或切换策略
- **角色边界清晰化**："A 只负责审查和标记问题，B 负责修复，修完后 A 最多审查一轮，必须给出通过/不通过的最终意见"

### 4.3 Token 消耗爆炸

Multi-Agent 的成本是单 Agent 的 N 倍起。每个 Agent 都有自己的上下文，自己的 LLM 调用，如果有辩论或投票，成本还要翻倍。

**实际数字**：
- 单 Agent ReAct 完成一次任务：~10-30 次 LLM 调用
- 3 Agent 层级协作完成同样任务：~30-80 次调用
- 辩论模式（2 Agent + Judge + 3 轮辩论）：~50-100 次调用

**应对**：
- 能用单 Agent 的不用 Multi-Agent——多花钱要有充分理由
- **小模型做 Worker**：Worker Agent 用 gpt-4o-mini / Claude Haiku，Manager 和 Judge 用大模型
- **减少 Agent 间通信的冗余信息**：A 传给 B 的应该是结构化输出，不是完整上下文
- **提前终止**：如果 Manager 判断 2/3 的 Worker 结果一致，不需要等最后一个

### 4.4 上下文管理

所有 Agent 共享一个超大上下文 vs 每个 Agent 独立上下文——两种极端。

共享上下文 → 任何 Agent 的输出都进入全局上下文 → token 爆炸 + 注意力稀释
独立上下文 → Agent 之间缺乏沟通 → 各自为战，结果不一致

**折中方案**：
- **共享 Memory + 隔离 Working Memory**：每个 Agent 有独立的对话上下文，但共享长期记忆（用户偏好、关键事实）
- **消息总线**：Agent 间的通信通过一个结构化消息队列，而非直接塞进上下文
- 只有"最终输出"跨 Agent 传递，Not "思考过程"

---

## 五、生产级的 Multi-Agent 架构

### 5.1 典型生产架构

```
┌──────────────────────────────────────────────────────┐
│                     Orchestrator                      │
│                 (LangGraph 编排引擎)                   │
│                                                      │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐         │
│  │ Router   │   │ Task     │   │ Result   │         │
│  │ Agent    │──▶│ Planner  │──▶│ Merger   │──▶ 输出  │
│  │ (意图路由)│   │ (任务规划)│   │ (结果整合)│         │
│  └──────────┘   └────┬─────┘   └──────────┘         │
│                      │                               │
│           ┌──────────┼──────────┐                    │
│           ▼          ▼          ▼                     │
│     ┌────────┐ ┌────────┐ ┌────────┐                 │
│     │Worker A│ │Worker B│ │Worker C│  (可扩展)        │
│     └────────┘ └────────┘ └────────┘                 │
│                                                      │
│  ┌──────────────────────────────────────────────┐    │
│  │ 共享组件: 记忆服务 | 工具注册中心 | 审计日志  │    │
│  └──────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────┘
```

### 5.2 关键设计决策

**1. 通信协议**

Agent 之间的信息传递需要标准化：

```json
{
  "from": "researcher_agent",
  "to": "analyst_agent",
  "type": "task_result",
  "content": {...},
  "metadata": {
    "confidence": 0.85,
    "sources": ["url1", "url2"],
    "time_cost_ms": 3200,
    "tool_calls_count": 4
  }
}
```

**2. 共享记忆 vs 隔离记忆**

共享的：用户画像、领域知识、关键事实
隔离的：任务执行中间态、Agent 的角色 prompt

**3. Human-in-the-loop 断点**

```
Router → 识别意图
   ↓
Planner → 生成执行计划
   ↓ [人工确认] ← 断点 1: 计划是否正确？
Worker Group → 并行执行
   ↓
Validator → 校验结果
   ↓ [人工确认] ← 断点 2: 关键决策点
Merger → 输出最终结果
```

Human-in-the-loop 不是每步都要人——只在两个关键节点介入：**计划确认**和**关键决策**。

### 5.3 可观测性

多 Agent 系统的 Debug 极其困难。你必须有：
- **每个 Agent 的执行轨迹**：完整的Thought → Action → Observation 日志
- **Agent 间通信记录**：谁给谁发了什么消息
- **工具调用链**：哪个 Agent 调了哪个工具，参数是什么，结果是什么
- **成本追踪**：每个 Agent、每个步骤消耗了多少 token，花了多少钱

OpenTelemetry + LangSmith / LangFuse / Arize 是目前的主流方案。

---

Multi-Agent 不是银弹。大多数场景下，**一个设计良好的单 Agent + 好的 prompt + 好的工具定义**已经足够。Multi-Agent 的价值在任务本身就有天然分工、或需要独立校验的场景。架构的复杂度应当匹配问题的复杂度——不要为了"用 Multi-Agent"而引入不必要的复杂度。

---

## 系列总结

本系列五篇文章覆盖了 AI Agent 的核心知识体系：

1. **Agent 架构与设计模式**：理解 Agent 的底层循环和 ReAct / Plan-Execute / Reflection 三大范式
2. **Function Calling 深度解析**：掌握 Agent "动手" 的完整机制，从 JSON Schema 到并行调用到安全设计
3. **Agent 记忆系统**：构建工作记忆、短期记忆、长期记忆的三层架构
4. **RAG 全链路**：从分块策略到检索增强到评估体系的生产级实践
5. **Multi-Agent 系统**：四种协作模式 + 三大框架对比 + 生产环境的设计决策

这些知识不仅覆盖 Agent 面试的高频考点，更是从零构建一个生产级 Agent 系统时真正需要决策的核心技术问题。
