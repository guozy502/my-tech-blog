---
title: "可观测性体系——Prometheus、分布式追踪与 SLI/SLO 方法论"
date: 2026-08-07
description: 从 Prometheus 的 Pull 模型与四种指标类型（Counter/Gauge/Histogram/Summary）、Histogram 的分位数可聚合性原理、PromQL 的核心语法与告警规则设计、分布式追踪的 Trace/Span/Context Propagation 传递机制、到 Google SRE 的 SLI→SLO→SLA 层次与错误预算驱动发布决策的方法论，拆解现代可观测性体系的三大支柱。
tags: ["架构","可观测性","Prometheus","分布式追踪","SLI","SLO","Grafana"]
categories: ["架构"]
---

# 历史背景——从"监控"到"可观测性"

2010 年代，运维团队管"监控"——配置一堆告警规则，CPU 过 90% 发通知。问题是：**告警告诉你出事了，但不告诉你为什么出事了。** 你还需要登录机器、查日志、分析调用链、对比指标。

2018 年，Google 的《Site Reliability Engineering》一书普及了"可观测性"的概念——**系统应该具备让外部观测者理解其内部状态的能力**。这不是一个功能的堆叠，而是三个维度的协同：

- **Metrics（指标）**：发生了什么？——Prometheus + Grafana
- **Traces（追踪）**：发生在哪里？——OpenTelemetry + Jaeger
- **Logs（日志）**：为什么发生？——ELK / Loki

三者的交集是：**Metric 告诉你延迟指标突然涨了，Trace 告诉你延迟发生在调用链的哪个节点上，Log 告诉你那个节点的应用日志里有一条异常堆栈。**

---

# 一、Prometheus——Metrics 的标准答案

## 1.1 Pull 模型——Prometheus 主动去拉，不是应用推给它

```
Prometheus 的 Pull 模型：
  应用暴露 /metrics 端点（HTTP 文本格式）
    → Prometheus Server 每 15s-1min 向这个端点拉取一次指标
    → 存储在本地的时序库（TSDB）中
    → Grafana 从 Prometheus 查数据并可视化

Push 模型（InfluxDB / Graphite）：
  应用主动把指标推给 Push Gateway 或消息队列
  → 适合短生命周期任务（CronJob/Pod 可能在拉取窗口之前就退出了）

Pull 模型的优势：
  - Prometheus 控制拉取频率——不需要应用操心"多久推一次"
  - 不需要中间 Broker——减少一个故障点
  - 每个 Target 的 /metrics 端点可以独立验证（curl 就行）
```

## 1.2 四种指标类型——不仅仅是 Counter

| 类型 | 行为 | 示例 | 适用 |
|------|------|------|------|
| **Counter** | 只增不减（重启归零） | `http_requests_total`、`errors_total` | 任何"累计"的东西 |
| **Gauge** | 可增可减的瞬时值 | `jvm_memory_used_bytes`、`cpu_usage_percent` | 瞬时快照 |
| **Histogram** | 按 bucket 分布的时间/大小 | `http_request_duration_seconds_bucket{le="0.1"}` = 耗时≤100ms 的请求数 | **延迟分布、可聚合分位数** |
| **Summary** | 客户端计算分位数 | `http_request_duration_seconds{quantile="0.99"}=320ms` | 不需要聚合的单实例场景 |

**Histogram vs Summary——面试最常问的选型**：

```
Histogram 的优点：分桶数据可以跨实例聚合
  三个实例各自暴露 bucket 数据：
    实例A: http_duration_bucket{le="0.1"} 100 ← 100 个请求 ≤100ms
    实例B: http_duration_bucket{le="0.1"} 80
    实例C: http_duration_bucket{le="0.1"} 120
    → Prometheus 可以把它们加起来：100+80+120=300
    → 从汇总的 bucket 计算全局的 P50/P95/P99

Summary 的缺点：计算好的分位数无法聚合
  实例A: P99=200ms
  实例B: P99=150ms
  实例C: P99=300ms
  → 你无法从这三个 P99 得到全局的 P99！
  → (200+150+300)/3 = 216ms 不是全局 P99，无意义

工程建议：用 Histogram。Summary 只在不需要跨实例聚合时用（极少场景）。
```

## 1.3 核心 PromQL——你在生产中最需要会的几条

```promql
# ① 过去 5 分钟的 QPS（每秒请求数）
rate(http_requests_total[5m])

# ② 过去 5 分钟的错误率
sum(rate(http_requests_total{status=~"5.."}[5m]))
  /
sum(rate(http_requests_total[5m]))

# ③ 过去 5 分钟的 P99 延迟（前提是用了 Histogram）
histogram_quantile(0.99,
  sum(rate(http_request_duration_seconds_bucket[5m])) by (le))

# ④ CPU 使用率百分比
100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# ⑤ 内存使用率
(1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100

# ⑥ 过去 5 分钟的请求延迟中位数
histogram_quantile(0.50,
  rate(http_request_duration_seconds_bucket[5m]))
```

**告警规则设计——不是"超过阈值就告警"**：

```yaml
groups:
  - name: api_alerts
    rules:
      # 好规则：P99 延迟 > 500ms 持续 5 分钟
      - alert: HighLatency
        expr: histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m])) > 0.5
        for: 5m             # ← "持续 5 分钟才告警"——防止瞬时波动误报
        labels:
          severity: warning
        annotations:
          summary: "API P99 延迟超过 500ms 持续 5 分钟"
          
      # 好规则：错误率 > 5% 持续 5 分钟
      - alert: HighErrorRate
        expr: (sum(rate(http_requests_total{status=~"5.."}[5m])) / sum(rate(http_requests_total[5m]))) > 0.05
        for: 5m
        labels:
          severity: critical
```

---

# 二、分布式追踪——一个请求的"旅行地图"

## 2.1 Trace 和 Span 是什么？

```
一次 HTTP 请求经过多个微服务：

  Trace（全局唯一的追踪链）：
    Trace-ID: abc123 ← 从 API 网关生成，贯穿整个调用链
    │
    ├── Span A: API Gateway (2ms)
    │     └── Span B: OrderService (45ms)
    │           ├── Span C: MySQL Query (40ms)  ← B 的子 Span
    │           └── Span D: Redis GET (1ms)      ← B 的子 Span
    │     └── Span E: UserService (30ms)
    │           └── Span F: MySQL Query (28ms)   ← E 的子 Span

  每个 Span 记录：操作名称、开始时间、结束时间、标签（HTTP URL/DB语句等）
```

## 2.2 Context Propagation——Trace-ID 怎么跨服务传递？

```
W3C Trace Context 标准（HTTP Header）：
  traceparent: 00-<trace-id>-<parent-span-id>-01

Gateway 收到请求（没有 traceparent）→ 生成 Trace-ID = abc123
→ 调用 OrderService 时，在 HTTP Header 中带上：
  traceparent: 00-abc123-<spanA-id>-01

OrderService 收到 → 提取 Trace-ID = abc123，创建自己的 Span B
→ 调用 MySQL 时，在 DB 语句的注释或协议字段中传递 Trace-ID

调用 Redis 时同样的逻辑。

整个过程对业务代码透明——OpenTelemetry Agent 自动注入和提取 Trace 信息
（基于 Bytecode Instrumentation 拦截 HTTP/gRPC/DB 调用）。
```

## 2.3 从慢 Trace 诊断——定位性能瓶颈

```
用户反馈"订单列表查询很慢" → 在 Jaeger 中查出这个请求的 Trace：

Trace 视图：
  Span: OrderListHandler    → 1800ms（全链路耗时）
    ├── Span: MySQL-ListOrders → 1700ms（这里花了 94% 的时间！）
    │     └── SQL: SELECT * FROM orders WHERE user_id=123
    └── Span: Redis-UserInfo → 1ms

诊断结论：
  瓶颈在 MySQL 查询。可能是因为：
    - orders 表 user_id 列没有索引
    - 或者该用户的订单数量极大（数万条），需要分页
    - 写"最近 30 天"条件来限制范围 + 加(user_id, created_at)联合索引
```

---

# 三、SLI / SLO / SLA——Google SRE 的可量化质量框架

## 3.1 三者的层次关系

```
SLI（Service Level Indicator，服务等级指标）
  可量化的服务质量测量值
  例："请求延迟 ≤ 300ms 的比例"、"成功率"、"可用性"

SLO（Service Level Objective，服务等级目标）
  基于 SLI 设定的目标值
  例："99.9% 的请求在 300ms 内完成"
  注意：SLO 不能是 100% — 必须为未知风险和变更留余地

SLA（Service Level Agreement，服务等级协议）
  具有商业约束力的 SLO — 不满足时有赔偿、罚款
  例："每月可用性 ≥ 99.99%，不满足退 10%"
  SLA 应该比 SLO 更宽松 — 如果 SLA = SLO，就没有"内部警告"的空间了
```

## 3.2 错误预算——SLO 的实际用途

```
错误预算 = 1 - SLO

例：SLO = 99.9% → 错误预算 = 0.1%
    一个月有 30 × 24 × 60 = 43200 分钟
    允许的不达标时间 = 43200 × 0.1% = 43 分钟

错误预算的作用：
  ① 发布决策：错误预算用完 → 暂停所有功能发布 → 集中做稳定性和可靠性改进
  ② 风险评估：新功能上线前，如果错误预算有剩余 → 可以承担"上线带来的故障风险"
  ③ 团队目标：可靠性团队的目标是"保护错误预算不被耗尽"，而不是"消灭所有故障"
```

**错误预算在工程中的实际例子**：

```
月初，错误预算 = 43 分钟

第 1 周：一次部署导致数据库连接池不足，服务瘫痪 10 分钟
  → 错误预算剩余：43 - 10 = 33 分钟

第 2 周：一个缓存配置错误导致 API 返回 500 错误，持续 15 分钟
  → 错误预算剩余：33 - 15 = 18 分钟

第 3 周：团队计划发布一个新功能 → 风险评估
  → 错误预算仅剩 18 分钟 → CTO 建议暂缓，"先把可靠性搞回来"

这就是"错误预算驱动发布"——不是在"随便发"和"禁发"中二选一，
而是用数学量化"发或不发"的风险边界。
```

## 3.3 怎么设计你的第一个 SLO

```
Step 1: 确定关键用户旅程（Critical User Journey）
  例：电商系统 → "用户搜索商品 → 看到结果 → 加购物车 → 支付"

Step 2: 选择 SLI（从用户视角定义指标）
  延迟 SLI：P99 搜索返回结果 ≤ 500ms
  成功率 SLI：下单成功率 ≥ 99.9%（故障/系统错误/拒绝）
  可用性 SLI：API 健康检查成功率 ≥ 99.99%

Step 3: 设定 SLO（不要 100%，至少留 0.1% 的错误预算）
  延迟 SLO = 99.9% 的搜索请求 ≤ 500ms
  成功率 SLO = 99.95% 的订单创建成功

Step 4: 计算错误预算
  成功率 SLO = 99.95% → 错误预算 = 0.05%
  一个月 = 43200 分钟 × 0.05% = 22 分钟
  → 每个月允许订单创建失败的累计时间为 22 分钟

Step 5: 构建错误预算仪表盘（Grafana）
  错误预算消耗率 = 已消耗的错误预算分钟 / 已过去的月份分钟
  → 如果消耗率 > 100%，说明错误预算已用完
  → 如果消耗率 > 150%，说明团队在迅速逼近 SLA 红线
```

---

# 四、总结

| 支柱 | 回答的问题 | 核心工具 |
|------|----------|---------|
| **Metrics** | 发生了什么？ | Prometheus + Grafana |
| **Traces** | 发生在哪里？ | OpenTelemetry + Jaeger |
| **SLI/SLO** | 有多好才算好？ | 错误预算驱动发布决策 |

> **可观测性的底线：出问题后 2 分钟知道出了事、5 分钟定位到根因、15 分钟止血。** 把这串时间变成从上到下的 SLO："故障发现时间 ≤ 2 分钟" → 对应告警规则设计；"根因定位时间 ≤ 5 分钟" → 对应 Trace/Log 的可用性；"止血时间 ≤ 15 分钟" → 对应回滚/降级能力。

# 延伸阅读

**Do——动手搭建：**
- Spring Boot 应用的 `/actuator/prometheus` 暴露指标 → Docker Compose 部署 Prometheus + Grafana → 配置第一个 Dashboard
- 用 `curl` 发一个请求 → 在 `traceparent` 传自定义 Trace-ID → 在 Jaeger 中跟踪全链路
- 选一个系统的核心 API → 定义延迟 SLI → 设 SLO → 在 Grafana 中创建错误预算仪表盘

**Todo——深入方向：**
- Prometheus 的 TSDB 存储引擎——内存 head block + WAL + 压缩管理
- OpenTelemetry Collector 的 pipeline 架构——Receiver → Processor → Exporter
- 多租户可观测性——将指标/追踪/日志按租户隔离和聚合

*本文参考资料：*
- Google SRE Team《Site Reliability Engineering》(2016)
- Prometheus 官方文档: Metric Types
- OpenTelemetry Specification: Context Propagation (W3C TraceContext)
