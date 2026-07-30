---
title: "TCP 拥塞控制——从 Reno 到 BBR，四个算法的博弈"
date: 2026-07-29
description: 从慢启动的指数增长、拥塞避免的线性爬坡、快速重传的 3 个重复 ACK 触发条件、到 BBR 如何用"测带宽而非探测丢包"彻底改变拥塞控制范式，拆解 TCP 四个核心算法的设计动机与进化历程。
tags: ["网络","TCP","拥塞控制","BBR","Reno","CUBIC"]
categories: ["网络"]
---

# 历史背景——1986 年的第一次"互联网大堵车"

1986 年，Lawrence Berkeley Lab 和 UC Berkeley 之间的网络吞吐从 32Kbps 暴跌到 40bps——下降了 1000 倍。Van Jacobson 调查后发现，这是 TCP 的"拥塞崩溃"（Congestion Collapse）：当网络有丢包时，发送方不断重传，但重传的包又加重了网络拥塞，导致更多丢包，循环恶化。整个网络在"传→丢→重传→更丢"的恶性循环中几乎完全瘫痪。

Van Jacobson 在 1988 年的经典论文中提出了 TCP 拥塞控制的四个算法——**慢启动、拥塞避免、快速重传、快速恢复**——统称 "TCP Reno"，至今仍是 TCP 拥塞控制的基础框架。

30 年后，Google 的 Neal Cardwell 等人提出 BBR（Bottleneck Bandwidth and Round-trip propagation time），不再"以丢包为拥塞信号"，而是**直接测量瓶颈带宽和最小 RTT**——从"被动反应"到"主动探测"，这是拥塞控制的第二次范式革命。

---

# 一、拥塞窗口（cwnd）——发送方的"油门"

TCP 的发送速率由**拥塞窗口（cwnd, Congestion Window）**控制。cwnd 是发送方在不收到 ACK 的情况下能发送的最大字节数（按 MSS 为单位）。

```
发送速率 ≈ cwnd / RTT

cwnd 增大 → 发送速率增大 → 网络吃不消 → 丢包
cwnd 减小 → 发送速率减小 → 网络缓解 → 丢包停止
```

TCP 的四个算法就是在"什么时候增大 cwnd、什么时候减小 cwnd、增大/减小多少"之间做博弈。

---

# 二、慢启动（Slow Start）——"先试探，别上来就猛冲"

## 2.1 原理

**名字是反的——慢启动其实很快**。"慢"是指"比连接建立前直接发整个接收窗口要慢"，但实际上 cwnd 是**指数增长**的：

```
初始: cwnd = 1 MSS (或 10 MSS, Linux 3.0+)
每收到 1 个 ACK: cwnd += 1 MSS
效果: 1 → 2 → 4 → 8 → 16 → 32 ... (每 RTT 翻倍)

这就是"慢启动"——以 2 的指数次方增长，实际上非常快。
```

## 2.2 ssthresh——慢启动的门槛

```
ssthresh (Slow Start Threshold): 慢启动上限

cwnd < ssthresh → 慢启动（指数增长）
cwnd >= ssthresh → 拥塞避免（线性增长）

初始 ssthresh 通常设得很高（如 64KB）
第一次丢包后 ssthresh 被设为当前 cwnd/2
```

---

# 三、拥塞避免（Congestion Avoidance）——"线性爬坡"

当 cwnd 达到 ssthresh 后，TCP 进入拥塞避免阶段：

```
每 RTT: cwnd += 1 MSS
效果: 线性增长（AIMD 的 AI 部分）

每收到 1 个 ACK: cwnd += 1/cwnd MSS
→ 需要收到 cwnd 个 ACK 才加 1 MSS
```

**AIMD 原则**：Additive Increase（线性加），Multiplicative Decrease（乘性减）。这是在网络中"公平分享带宽"的数学基础——每个流的 cwnd 线性增加、丢包时乘性减少，最终所有流趋向于平均分享瓶颈带宽。

---

# 四、快速重传（Fast Retransmit）——"不等超时就重来"

## 4.1 原理

传统 TCP 依赖**超时重传（RTO）**——如果 ACK 在 RTO 时间内没收到，重传数据。RTO 典型值 200ms-1s，这意味着一个丢包要等至少 200ms 才能被发现。

快速重传说：**如果收到 3 个重复 ACK（Dup ACK），说明后面的数据到了但中间缺了一个**——立刻重传，不等超时。

```
发送方发了 seq: 1, 2, 3, 4, 5
seq=2 丢了
接收方收到 seq=1 → ACK=2 ("我要 2")
接收方收到 seq=3 → ACK=2 ("我要 2")  ← 第一个重复 ACK
接收方收到 seq=4 → ACK=2 ("我要 2")  ← 第二个重复 ACK
接收方收到 seq=5 → ACK=2 ("我要 2")  ← 第三个重复 ACK → 触发快速重传！
发送方立刻重传 seq=2（不等超时）
```

**为什么是 3 个重复 ACK 而不是 1 个？** 因为 TCP 可能因为乱序而产生 1-2 个重复 ACK（后面的包先到了，前面的包后到）。3 个重复 ACK 意味着有足够证据判断是真的丢包，而不是乱序。

## 4.2 TCP Reno vs TCP Tahoe

```
Tahoe（1988 年，第一个版本）：
  丢包后 → cwnd = 1 → 慢启动（太保守）

Reno（1990 年，改进版）：
  丢包后 → ssthresh = cwnd/2 → cwnd = cwnd/2 → 拥塞避免（快速恢复）
  → 不回到慢启动！从 cwnd/2 开始线性增长
  → 这就是现在 Linux 的默认行为
```

---

# 五、CUBIC——高带宽网络的优化

Linux 2.6.19 之后，CUBIC 替代 Reno 成为默认拥塞控制算法。**Reno 的问题是**：在高带宽高延迟（BDP 大）网络中，线性增长太慢了——每次丢包减半后要爬很久才能回到满带宽。

**CUBIC 的创新**：不用线性增长，而用**三次函数**。cwnd 的增长曲线是与"上次丢包时间点"的三次函数：

```
cwnd = C × (t - K)³ + W_max

C: 缩放因子
t: 当前时间
K: 到达 W_max 需要的时间（根据 RTT 和 W_max 计算）
W_max: 上次丢包时的 cwnd

效果：
  - 刚丢包后：cwnd 快速反弹（曲线凹的部分）
  - 接近 W_max 时：cwnd 缓慢试探（曲线凸的部分）
  - 超过 W_max 后：继续爬（只有在检测到更多可用带宽时才继续）

这是把 AI 的部分从"线性"变成了"三次函数"——
在接近上次丢包点时更激进，超过后更保守。
```

---

# 六、BBR——Google 的范式革命

## 6.1 BBR 的核心思想

Reno/CUBIC 的共同前提是**"丢包 = 拥塞"**。但在有 buffer 的网络中（交换机/路由器有缓冲区），丢包是**最后发生的**——在丢包之前，缓冲区已经被填满，导致延迟飙升（Bufferbloat）。

BBR 不再以丢包为信号，而是**直接测量网络的两个物理参数**：

```
① RTprop (Round-Trip propagation time)：网络的最小 RTT（没有排队时的传播延迟）
② BtlBw (Bottleneck Bandwidth)：瓶颈链路的真实可用带宽

目标：发送速率 = BtlBw，且 inflight < BtlBw × RTprop
     → 恰好填满管道，不产生排队，不丢包
```

## 6.2 BBR 的四阶段状态机

```
BBR 循环探测网络：

① STARTUP（探测带宽）
  类似慢启动，指数增长，找到 BtlBw
  当吞吐不再增长 → 进入 DRAIN

② DRAIN（排空队列）
  降低发送速率，排空 STARTUP 期间在缓冲区积累的队列

③ PROBE_BW（稳态）
  大部分时间在这个状态
  以 8 个 RTT 为周期，持续估算 BtlBw 和 RTprop
  偶尔略微加大发送速率（探测是否有更多带宽可用）
  偶尔略微减小发送速率（探测 RTT 是否更小）

④ PROBE_RTT（探测最小延迟）
  每 10 秒进入一次
  cwnd 降到 4 MSS —— 强制排空所有队列 → 测量最小 RTprop
```

## 6.3 CUBIC vs BBR 对比

| | CUBIC | BBR |
|------|-------|-----|
| **拥塞信号** | 丢包 | 带宽 + RTT 测量 |
| **Bufferbloat** | 会把 buffer 填满 → 高延迟 | 尽量不产生排队 → 低延迟 |
| **丢包场景** | 丢包 = 减 cwnd | 丢包不等于拥塞（可能是链路质量差） |
| **适用** | 通用（特别是低 BDP） | 高 BDP + 有丢包（跨海链路、视频、4G/5G） |

```bash
# 查看当前 TCP 拥塞控制算法
cat /proc/sys/net/ipv4/tcp_congestion_control  # 输出: cubic (默认)

# 切换到 BBR
echo bbr > /proc/sys/net/ipv4/tcp_congestion_control
# 需要内核 4.9+ 且加载了 tcp_bbr 模块
```

---

# 七、总结

| 算法 | 年代 | 核心思想 | 问题 |
|------|------|---------|------|
| **Reno** | 1990 | AIMD：丢包减半，线性恢复 | 高 BDP 恢复太慢 |
| **CUBIC** | 2008 | 三次函数替代线性增长 | 仍然依赖丢包信号 |
| **BBR** | 2016 | 直接测量带宽和延迟 | 与其他算法共存时可能不公平 |

> TCP 拥塞控制的本质变迁：**从"丢包即拥塞"（Reno/CUBIC）到"延迟即拥塞"（BBR）**——前者在 buffer 填满后才反应，后者在 buffer 刚产生排队时就调控。

# 延伸阅读

**Do——动手验证：**
- `ss -tin` 查看当前连接的 cwnd、rtt、ssthresh 等 TCP 信息
- 用 `iperf3` 在两个节点间打流，同时观察 cwnd 的变化（`ss -tin` 每 0.5 秒采集）
- 切换 BBR 后用相同测试对比 CUBIC 的吞吐和延迟（`tc qdisc` 模拟丢包和延迟）

**Todo——深入方向：**
- Google BBRv2——修复 BBRv1 与 Reno/CUBIC 共存时"抢占过多带宽"的公平性问题
- QUIC 的拥塞控制——比 TCP 灵活在哪里（每个 Stream 独立控制 vs TCP 全局 cwnd）
- 移动网络下的 TCP 优化——高延迟 + 频繁切换 + 动态带宽

*本文参考资料：*
- Van Jacobson, "Congestion Avoidance and Control" (SIGCOMM 1988)
- Neal Cardwell et al., "BBR: Congestion-Based Congestion Control" (ACM Queue 2016)
- RFC 5681: TCP Congestion Control (Reno)
- Linux 内核源码 `net/ipv4/tcp_cubic.c` / `tcp_bbr.c`
