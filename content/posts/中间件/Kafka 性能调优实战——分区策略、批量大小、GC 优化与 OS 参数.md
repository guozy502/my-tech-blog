---
title: "Kafka 性能调优实战——分区策略、批量大小、GC 优化与 OS 参数"
date: 2026-06-28
description: 从分区数规划、生产者/消费者参数调优、JVM GC 选型到 OS 内核参数，覆盖 Kafka 性能调优的全链路实战清单。
tags: ["Kafka","中间件","性能优化","GC","调优"]
categories: ["中间件"]
---

# 历史背景——Kafka 调优为什么是"全栈"的？

很多开发者拿到 Kafka 后第一件事就是调参数，但大多数人的思路局限在"改个 `batch.size`"或者"加个 `compression.type`"。实际上，LinkedIn 早期的 Kafka 运维经验表明：**Kafka 的性能瓶颈从来不在单一组件上，而是一条从应用层到底层硬件的全链路管道**。

应用层（批量大小、压缩算法）→ JVM 层（GC 选型、堆大小）→ OS 层（Page Cache、文件句柄、网络缓冲区）→ 硬件层（磁盘调度器、文件系统、网卡）。这里面任何一环的配置失误，都会让其他层的优化付诸东流。比如：你把 `batch.size` 拉到 512KB 提高了吞吐，结果 JVM Heap 设 32GB 用 CMS GC，频繁的 Full GC 卡住写入路径，吞吐不升反降。

本文按照**从应用层到硬件层**的顺序，逐个拆解每个节点的调优原理和推荐值。它们不是孤立的数字，而是一套需要匹配的配置组合。

---

# 一、分区数——第一道决策，也是最容易后悔的

## 1.1 What：分区数影响了什么？

分区是 Kafka 并行度的最小单位。分区数决定了：
- **写入并行度**：每个分区只能被一个消费者线程消费（同一消费组内）
- **消费者数量上限**：消费组内有效消费者数量 ≤ 分区数
- **Controller 压力**：分区越多，Controller 在 Broker 变化时需要处理的元数据越多
- **文件句柄数**：每个分区的每个 Segment 需要至少 3 个文件 (.log + .index + .timeindex)

## 1.2 Why：为什么"分区宁少勿多"？

很多团队的习惯是"分区先设大，以后好扩展"。这个思路的代价是：

1. **Controller 压力**：分区数从 1000 涨到 10000，Controller 处理一次 Broker 宕机的 Leader 选举耗时可能从 1 秒涨到 10 秒
2. **文件句柄爆炸**：10000 个分区 × 3 个 Segment 文件 × 3 副本 = 90000 个文件句柄（还没算网络连接和其他开销）
3. **元数据膨胀**：每次 `UpdateMetadataRequest` 的 payload 与分区数成正比，在大集群中可能成为网络亮点

| 场景 | 建议分区数 | 原因 |
|------|-----------|------|
| 单 Broker | ≤ 4000 | 文件句柄和 Controller 开销可控 |
| 集群总分区 | ≤ 20000 | 超过此数建议拆分集群 |
| 单分区吞吐 | ~10MB/s 写 / ~50MB/s 读 | 普通商用机磁盘上限 |
| 预估分区数 | `max(目标吞吐/单分区吞吐, 消费者数)` | 取这两者的较大值 |

**规划方法**：如果目标总吞吐是 100MB/s 写，单分区能跑 10MB/s → 至少 10 个分区。如果消费者最多 20 个 → 至少 20 个分区。最终取 20。再加 20% 冗余 → 24。**不要一上来就设 128**。

---

# 二、生产者参数调优

## 2.1 What：三个维度三种配置

生产者调优按照**业务目标**可以分为三种配置模板。没有"万能参数"，只有匹配需求的组合。

**吞吐优先**——适用于日志收集、埋点上报、批量数据同步：
```properties
batch.size=262144              # 256KB 批量
linger.ms=10                   # 等待 10ms 攒批
compression.type=lz4           # LZ4 压缩速度最快
buffer.memory=134217728        # 128MB 发送缓冲区
max.in.flight.requests.per.connection=5  # 允许 5 个未确认请求
acks=1                         # Leader 确认即可（不要求所有副本）
```

**可靠性优先**——适用于订单、支付、账户数据：
```properties
acks=all                       # 等待所有 ISR 确认
retries=2147483647             # 无限重试（配合 delivery.timeout.ms）
delivery.timeout.ms=120000     # 2 分钟超时
enable.idempotence=true        # 幂等生产者，防止网络重试导致消息重复
max.in.flight.requests.per.connection=5  # 幂等模式下自动 <=5
```

**低延迟优先**——适用于实时推荐、在线交易：
```properties
linger.ms=0                    # 有数据就发，不等
batch.size=16384               # 小批量，减少积攒等待
compression.type=none          # 不压缩，省 CPU
acks=1                         # 快速确认
```

## 2.2 为什么这三者不能兼得？

```
吞吐        可靠性        延迟
  ↑            ↑            ↑
  │     ╲      │      ╱     │
  │        ╲   │   ╱        │
  │           ╲│╱           │
  └────────────┼────────────┘
            三者关系

吞吐 ↑ → 批量大 + linger 长 → 延迟 ↑
可靠性 ↑ → acks=all → 吞吐 ↓ + 延迟 ↑
延迟 ↓ → 批量小 + 不等 → 吞吐 ↓
```

这个不可能三角意味着你必须明确**当前业务的优先级**。建议是：大部分生产系统选"可靠优先 + 合理批量"（`acks=all + linger.ms=10`），这样在保证消息不丢的前提下吞吐也不会太差。

## 2.3 对比速查表

| 参数 | 吞吐优先 | 可靠性优先 | 低延迟优先 |
|------|---------|-----------|-----------|
| `acks` | 0/1 | all | 1 |
| `linger.ms` | 10-50 | 10-50 | 0 |
| `compression` | lz4 | lz4/zstd | none |
| `retries` | 3 | MAX_INT | 0 |
| `enable.idempotence` | false | true | false |
| `max.in.flight.requests` | 5 | 5（幂等限制） | 1 |

---

# 三、Broker 参数调优

```properties
# === 网络线程 ===
num.network.threads=8           # 处理网络请求（accept + read/write）
num.io.threads=16               # 处理磁盘 IO（通常 2x network.threads）

# === 日志存储 ===
log.segment.bytes=1073741824    # 1GB per Segment
log.retention.hours=168         # 7 天保留
log.retention.bytes= -1         # 不按大小限制（按时间）
log.index.interval.bytes=4096   # 索引间隔 4KB

# === 副本拉取 ===
num.replica.fetchers=4          # Follower 同步线程数
replica.fetch.max.bytes=10485760  # 每次拉取最多 10MB
replica.lag.time.max.ms=30000   # Follower 超过 30 秒落后 → 踢出 ISR

# === Page Cache 内存配比 ===
# 核心原则：Heap 够用就行，剩余内存全给 OS 做 Page Cache
# 典型 32GB 物理机：Heap 6GB + Page Cache ~20GB + OS 开销 6GB
```

**网络线程与 IO 线程**的职责分离：`network.threads` 处理 TCP 连接和协议解析，`io.threads` 处理磁盘读写。两者分离是因为网络操作是 CPU 密集（协议编解码），磁盘操作是 IO 密集——分开以后 IO 线程等待磁盘时不会阻塞网络线程。

---

# 四、JVM GC 优化

## 4.1 Why：Kafka 对 GC 有多敏感？

Kafka Broker 的很多关键路径依赖于 JVM 堆上的对象——`ProducerRequest` 对象、`FetchRequest` 对象、分区元数据缓存等。如果 GC 停顿 500ms，所有等待该 Broker 上 Leader 分区的生产者和消费者都会被卡住 500ms。在吞吐 100MB/s 的场景下，500ms 意味着约 **50MB 的写入被暂停**，TCP 发送缓冲区很可能被打满。

## 4.2 How：G1 GC 配置

Kafka 官方推荐 G1 GC，目标是**控制停顿时间在 20ms 以内**：

```bash
KAFKA_HEAP_OPTS="-Xmx6g -Xms6g"  # Heap 大小固定，避免动态调整
KAFKA_JVM_PERFORMANCE_OPTS="
  -XX:+UseG1GC
  -XX:MaxGCPauseMillis=20          # 目标：每次 GC 不超过 20ms
  -XX:InitiatingHeapOccupancyPercent=35  # 堆占用 35% 就开始并发标记
  -XX:+DisableExplicitGC           # 禁止 System.gc()
  -XX:G1HeapRegionSize=4M          # Region 大小 4MB（6GB Heap 适用）
"
```

- **`Xmx = Xms`** 避免堆动态伸缩，伸缩过程中 JVM 会触发 Full GC
- **`InitiatingHeapOccupancyPercent=35`**：Kafka 对消息分配的内存生命周期很短（请求处理完即可回收），设置较低的 IHOP 让 G1 更早开始并发标记，减少 Young GC 停顿时间
- **`DisableExplicitGC`**：某些库（如 NIO DirectBuffer）可能调用 `System.gc()`，禁止后避免意外 Full GC

---

# 五、OS 内核参数

## 5.1 What：OS 层在控制什么？

```bash
# === 文件句柄 ===
ulimit -n 100000
# Kafka 的分区多、Segment 多、网络连接多，文件句柄需求远超默认 1024

# === 网络缓冲区 ===
net.core.wmem_default=1048576     # 默认写缓冲 1MB
net.core.rmem_default=1048576     # 默认读缓冲 1MB
net.core.wmem_max=16777216        # 最大写缓冲 16MB
net.core.rmem_max=16777216        # 最大读缓冲 16MB
# 缓冲太小 → TCP 窗口缩死 → 带宽上不去

# === 虚拟内存 ===
vm.swappiness=1                   # 尽量不用 swap，避免磁盘换页拖垮延迟
vm.dirty_ratio=20                 # 脏页占 20% 物理内存时开始强制刷盘
# dirty_ratio 默认 10%，调高到 20% 可以给 Kafka 更大的写缓冲空间

# === 文件系统挂载 ===
# noatime：不更新文件访问时间，减少磁盘写操作
mount -o noatime /dev/sdb /data/kafka

# === 磁盘调度器 ===
echo noop > /sys/block/sdb/queue/scheduler
# 对于 SSD，noop/deadline 最合适（SSD 不需要磁头调度）
```

## 5.2 Why：vm.swappiness=1 而不是 0？

很多教程说设为 0 禁止 swap，但实际上 Linux 3.5+ 后 `swappiness=0` 的含义变了（只在 OOM 时才换出）。Kafka 社区推荐 `swappiness=1`：允许极少量 swap 换出那些真正不用的匿名页（堆上的一些冷对象），但不会主动换出 Page Cache。

---

# 六、监控核心指标

调优必须搭配监控，否则就是盲调：

| 指标 | 告警阈值 | 含义 | 根因方向 |
|------|---------|------|---------|
| `UnderReplicatedPartitions` | > 0 | 有副本未同步 | Broker 宕机/网络抖动/磁盘满 |
| `ActiveControllerCount` | ≠ 1 | Controller 不正常 | Controller 切换或故障 |
| `BytesInPerSec` | > 80% 带宽 | 流量接近网卡上限 | 分区扩容或限流 |
| `RequestQueueSize` | > 1000 | Broker 处理不过来 | 线程数不够或磁盘瓶颈 |
| `ISRShrinkPerSec` | > 0 | ISR 缩容，Follower 掉队 | 磁盘慢/网络慢/GC 停顿 |
| `NetworkProcessorAvgIdle` | < 0.3 | 网络线程忙不过来 | 增加 network.threads |
| `RequestHandlerAvgIdle` | < 0.3 | IO 线程忙不过来 | 增加 io.threads 或排查慢磁盘 |

---

# 七、总结

| 层级 | 核心调优 | 一句话 |
|------|---------|--------|
| **生产者** | batch + linger + lz4 | 用 10ms 延迟换 10x 吞吐 |
| **Broker JVM** | G1 GC + Heap 6GB | Full GC 停顿是最好的吞吐杀手 |
| **Broker OS** | Page Cache > Heap | 32GB 物理机：6GB Heap + 20GB Cache |
| **OS 内核** | 文件句柄 + 网络缓冲 + noatime | 内核参数是最后也是最容易被忽略的环节 |
| **架构规划** | 分区数宁少勿多 | 分区多的开销全是 Controller 跑不掉 |

# 延伸阅读

**Do——动手验证：**
- 在测试集群上做一次双盲对比：`linger.ms=0` vs `linger.ms=20`，用 `kafka-producer-perf-test.sh` 压测
- `jstat -gcutil <kafka-pid> 1000` 观察 G1 GC 的频率和停顿时间
- 用 `cat /proc/<kafka-pid>/limits | grep "open files"` 确认文件句柄生效
- 模拟一次磁盘写满的场景：`dd if=/dev/zero of=/data/kafka/bigfile bs=1G count=100`，观察 Broker 日志和监控面板

**Todo——深入方向：**
- [ ] G1 GC 的 Mixed GC 和 Concurrent Mark 对 Kafka 请求延迟的影响分析
- [ ] Linux io_uring（Kernel 5.1+）替换 epoll 对 Kafka 网络层的潜在收益
- [ ] KRaft 模式下的元数据 Topic `@metadata` 与普通数据 Topic 的资源隔离策略
- [ ] 多租户隔离场景下的 Broker 参数差异化（不同 Topic 不同 `retention.ms`）

*本文参考资料：*
- Neha Narkhede et al.《Kafka: The Definitive Guide》（第 2 版）——第 3 章（生产者）、第 5 章（内部机制）
- Apache Kafka 官方文档 - Operations: https://kafka.apache.org/documentation/#operations
- Oracle G1 GC Tuning Guide: https://www.oracle.com/technical-resources/articles/java/g1gc.html
- Linux Kernel Documentation - sysctl/vm.txt, sysctl/net.txt
