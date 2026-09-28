---
title: "Agent 可观测性——Trace、Replay 与四维监控体系"
date: 2026-08-08
description: 从 Agent 的完整链路 Trace（用户请求→规划→检索→工具选择→执行→模型调用→子Agent→人工确认→输出）与分布式追踪的 Context Propagation、Replay 回放机制（保存失败任务的输入/上下文/工具结果/Agent状态→复现问题→验证修复→支撑回归测试）、四维监控指标体系（业务指标/模型指标/Agent指标/安全指标）与告警规则设计，拆解 Agent 生产可观测性的完整方案。
tags: ["AI Agent","可观测性","Trace","Replay","监控","告警"]
categories: ["Agent"]
---

# 历史背景——"Agent 卡住了"该怎么排查？

传统应用出问题：CPU 100%？查 top。内存泄漏？查 heap dump。数据库慢？查 slow query log + EXPLAIN。每条线索都有明确的方向。

Agent 出问题："Agent 没给我返回结果"——可能的原因有 15 种：模型超时、工具返回了错误格式、检索没召回到文档、LLM-as-Judge 误判导致循环、权限校验拦截了工具调用、上下文被压缩导致关键信息丢失……

Agent 的可观测性不是"加个 Prometheus 指标"就能解决的——你需要一条**完整的 Trace 链路**，从用户请求到最终输出的每一步都可以被回溯、被回放、被诊断。

---

# 一、Agent Trace——一次执行的所有决策链路

## 1.1 Agent Trace 和分布式 Tracing 的关系

```
分布式 Tracing（Jaeger/Zipkin）：
  关注跨服务的 RPC 调用链路
  Gateway → OrderService → MySQL / Redis / Kafka → 返回

Agent Trace：
  关注一次 Agent 执行的"决策链路"
  用户请求 → Planning → Context Build → Retrieval →
  Tool Selection → Tool Execution → Model Call →
  Sub-Agent → Human Confirmation → Policy Block → Final Output

差异：
  - 分布式 Tracing 的 span 之间的因果关系是显式的（调用 A 返回后才调用 B）
  - Agent Trace 的 span 之间的因果关系是"决策树"（Agent 决定了要不要调 B）
  - Agent Trace 需要记录更多"元信息"——Prompts、Token 消耗、检索相关性分数、工具参数
```

## 1.2 Agent Trace 的完整 Span 树

```
Trace-ID: agent-task-xyz-123
│
├── [Span] User Request Received
│     input: "帮我找出项目中所有未处理异常的位置并生成修复建议"
│     user: "alice"
│
├── [Span] Task Planning
│     model: claude-sonnet-4-6 | tokens: 1200 in / 300 out | cost: $0.003
│     plan: ["搜索异常处理代码", "分析未捕获的异常", "生成修复建议", "输出 Markdown"]
│
├── [Span] Context Build
│     system_prompt_tokens: 2500 | task_context_tokens: 800 | retrieval_budget: 8000
│
├── [Span] Retrieval: "未处理异常 catch Exception"
│     method: hybrid (vector + BM25) | chunks_returned: 12 | top_score: 0.92
│
├── [Span] Tool Call: search_code("catch Exception")
│     tool: codebase.search_code | params_valid: true | permission: READ
│     files_found: 8 | duration: 120ms | status: success
│
├── [Span] Sub-Agent: AnalyzeException
│     delegated_task: "分析这 8 个文件中的异常处理是否完整"
│     model: claude-sonnet-4-6 | tokens: 8500 in / 1500 out | cost: $0.03
│     result: "发现 5 处未妥善处理的异常"
│
├── [Span] Tool Call: write_file("exception-fix-report.md")
│     tool: file.write | permission: WRITE | risk: NORMAL
│     audit: user=alice, file=exception-fix-report.md, size=3200B
│     duration: 45ms | status: success
│
└── [Span] Final Output
    output_tokens: 800 | generation_time: 2.3s | total_cost: $0.05
    Task Success: ✅
```

## 1.3 Trace 上下文在 Agent 组件间传递

```python
# Agent 的 Context Propagation
class AgentTraceContext:
    trace_id: str       # 全局唯一，贯穿整个任务
    task_id: str        # 任务 ID（一个 trace 可以包含多个子任务）
    parent_span_id: str # 父 Span（子 Agent、工具调用、模型调用的上一级）
    
    def to_headers(self) -> dict:
        return {
            "x-agent-trace-id": self.trace_id,
            "x-agent-task-id": self.task_id,
            "x-agent-span-id": self.parent_span_id,
        }

# 每一次模型调用、工具调用、子 Agent 调用都携带这些头部
# → Trace Collector 收到 → 关联到同一棵 Span 树 → 可视化
```

---

# 二、四维监控指标体系

## 2.1 业务指标——"Agent 帮用户完成了多少事"

| 指标 | 计算方式 | 为什么重要 |
|------|---------|-----------|
| **任务成功率** | 完成任务数 / 总任务数 | **最核心的 Agent 指标**——直接反映"Agent 好不好用" |
| **自动化完成率** | 不需要人工干预就完成的任务比例 | 反映 Agent 的自主能力 |
| **人工接管率** | 触发 Human-in-the-loop 的比例 | 过高 → Agent 不能独立完成任务 |
| **用户满意度** | 用户评分 / 反馈 | 主观但不可替代——有些问题 Eval 指标测不出来 |
| **平均处理时长** | 从任务开始到结束的秒数 | 反映 Agent 效率 |

## 2.2 模型指标——"模型调用都正常吗"

| 指标 | 含义 | 告警规则 |
|------|------|---------|
| **Token 消耗** | 每次调用 + 每日累计 | 日环比 > 1.5x |
| **平均延迟 / P95 / P99** | 模型推理延迟 | P99 > 5s |
| **成本趋势** | 单任务成本 + 日总成本 | 日成本 > 预算 × 1.2 |
| **调用失败率** | 超时/限流/服务端错误 | > 1% |
| **Model Fallback 率** | 从主模型降级到备用的比率 | > 5% |

## 2.3 Agent 指标——"Agent 的决策和执行有多不靠谱"

| 指标 | 含义 | 告警规则 |
|------|------|---------|
| **工具调用准确率** | 选对工具 + 参数正确的比例 | < 90% |
| **工具失败率** | 工具执行失败（超时/错误）的比例 | > 5% |
| **检索命中率** | RAG 检索结果中有相关文档的比例 | < 80% |
| **引用准确率** | 引用内容支持其结论的比例 | < 85% |
| **Loop 中断次数** | Agent 因超过最大循环次数而被强制终止 | 1 天内 > 10 |
| **失败恢复成功率** | 从工具失败中成功恢复的比例 | < 70% |
| **子 Agent 成功率** | 委托给子 Agent 的任务完成率 | < 85% |

## 2.4 安全指标——"Agent 有没有越界"

| 指标 | 含义 | 告警规则 |
|------|------|---------|
| **高风险工具调用次数** | shell.exec / database.execute_ddl 等 | 异常突增 → 可能是 Prompt Injection |
| **越权调用拦截率** | 用户试图调用无权限工具被拦截的比例 | 异常突增 → 可能被攻击 |
| **敏感字段访问次数** | 密钥/密码/PII 相关工具调用 | 不为 0 → 立即告警 |
| **危险 SQL 拦截率** | DROP/TRUNCATE/DELETE 被拦截 | 不为 0 → 立即告警 |
| **人工确认触发率** | 高风险工具触发的"需要人工确认" | 偏离基线 ±50% |

---

# 三、Replay 回放——"这个失败的 case 我能在本地复现吗"

## 3.1 Replay 和传统日志有什么区别

```
传统日志：
  "Agent 调用 weather.get(city='Beijing') 失败了，原因是 API 超时"
  → 你知道了事实，但没法"重跑一遍"——因为当时的上下文已经没了

Replay：
  "保存失败任务的完整状态——输入/上下文/工具结果/Agent 状态/Span 树"
  → 你可以用同样的输入和上下文重新跑这个任务
  → 验证你的修复是否真的解决了这个 case
  → 把这个 case 加入 Golden Dataset → 永不再现
```

## 3.2 Replay 的数据保存

```python
class ReplayRecorder:
    def save_replay(self, trace: AgentTrace) -> str:
        """保存任务的全部状态，使它可以被重放"""
        replay_data = {
            "trace_id": trace.id,
            "task_input": trace.task_input,
            "user_id": trace.user_id,
            "system_prompt": trace.system_prompt,
            "tool_definitions": trace.tool_definitions,
            "tool_results": [r.to_dict() for r in trace.tool_results],
            "model_calls": [m.to_dict() for m in trace.model_calls],
            "retrieval_results": trace.retrieval_results,
            "context_snapshot": trace.context_snapshot,
            "agent_state": trace.agent_state,
            "final_output": trace.final_output,
            "error": trace.error,  # 如果失败了，错误信息
        }
        replay_id = f"replay-{trace.id}"
        self.storage.save(replay_id, json.dumps(replay_data))
        return replay_id
```

## 3.3 三种 Replay 的使用场景

```
① 调试一个特定的失败 case：
   从 Trace 列表中找到失败的任务 → 点击 Replay → 本地跑一遍
   → 对比原始 Trace 和 Replay Trace → 定位是哪一步出问题

② 验证修复：
   修改 Prompt / 工具 / RAG 策略 → 用 Replay 重跑失败 case
   → 如果原来失败的现在成功了 → 修复有效

③ 回归测试：
   把所有历史失败 case 的 Replay 加入 Golden Dataset
   → 每次 Agent 版本变更后批量重跑 → 确保旧 case 没有退化
```

---

# 四、日志体系——查什么、从哪查

## 4.1 模型调用日志

```json
{
  "trace_id": "agent-task-xyz-123",
  "model": "claude-sonnet-4-6",
  "model_version": "20241022",
  "prompt_version": "v3.1",
  "input_tokens": 8500,
  "output_tokens": 1500,
  "cost": 0.03,
  "latency_ms": 2340,
  "status": "success",
  "error": null
}
```

## 4.2 工具调用日志

```json
{
  "trace_id": "agent-task-xyz-123",
  "tool_name": "github.create_pr",
  "tool_version": "v2.0",
  "params": {"repo": "my-app", "title": "Fix NPE", "head": "fix/npe"},
  "permission": "WRITE",
  "result": "success",
  "pr_url": "https://github.com/my-app/pull/456",
  "duration_ms": 3200,
  "retry_count": 0,
  "audit_user": "alice",
  "audit_ip": "10.0.1.5"
}
```

---

# 五、总结

| 组件 | 解决的问题 | 一句话 |
|------|----------|--------|
| **Agent Trace** | Agent 出问题不知道从哪查 | 一次任务的全部决策链路可追溯 |
| **四维监控** | 不知道 Agent 健康还是不健康 | 业务/模型/Agent/安全——全覆盖 |
| **Replay** | 失败的 case 无法复现 | 保存完整状态 → 一键重跑 |
| **日志体系** | 查不到调了什么工具 | 模型调用 + 工具调用 + 权限检查全部留痕 |

# 延伸阅读

**Do——动手搭建：**
- 用 OpenTelemetry 的 Python SDK 给 Agent 加 Span——每个模型调用/工具调用/检索操作一个 Span
- 模拟一个工具调用失败，保存 Replay 数据 → 改修复 → Replay 重跑验证
- 接入 Prometheus + Grafana，搭建 Agent 的四维监控 Dashboard

**Todo——深入方向：**
- LangSmith / Arize / Phoenix——专业 Agent 可观测性平台的架构对比
- 失败样本的自动聚类——用 Embedding 把相似失败 case 聚类 → 定位系统性问题

*本文参考资料：*
- OpenTelemetry Specification: Traces
- Anthropic Trace 文档
- LangSmith 文档: https://docs.smith.langchain.com/
