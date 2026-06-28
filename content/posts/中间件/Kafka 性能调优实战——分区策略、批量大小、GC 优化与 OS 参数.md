---
title: "Kafka 性能调优实战——分区策略、批量大小、GC 优化与 OS 参数"
date: 2026-06-28
description: 从分区数规划、生产者/消费者参数调优、JVM GC 选型到 OS 内核参数，覆盖 Kafka 性能调优的全链路实战清单。
tags: ["Kafka","中间件","性能优化","GC","调优"]
categories: ["中间件"]
---

# 一、分区数——第一道决策

| 场景 | 建议分区数 |
|------|-----------|
| 单 Broker |  ≤ 4000 |
| 集群总分区 | ≤ 20000 |
| 单分区吞吐 | ~10MB/s 写 / ~50MB/s 读 |

**分区过多的代价**：Controller 压力、ZK 元数据膨胀、文件句柄消耗、Leader 选举变慢。

---

# 二、生产者参数调优

```properties
# === 吞吐优先 ===
batch.size=262144              # 256KB（默认 16KB 太小）
linger.ms=10                   # 积攒 10ms 再发
compression.type=lz4           # LZ4 压缩：速度快，比 35%
buffer.memory=134217728        # 128MB 缓冲区
max.in.flight.requests.per.connection=5  # 最多 5 个未确认请求

# === 可靠性优先 ===
acks=all                       # 等所有 ISR 确认
retries=2147483647             # 无限重试
delivery.timeout.ms=120000     # 2 分钟超时
enable.idempotence=true        # 幂等生产者（防止重复）
```

| 参数 | 吞吐优先 | 可靠性优先 | 低延迟优先 |
|------|---------|-----------|-----------|
| `acks` | 0/1 | all | 1 |
| `linger.ms` | 10-50 | 10-50 | 0 |
| `compression` | lz4 | lz4/zstd | none |
| `retries` | 3 | MAX_INT | 0 |

---

# 三、Broker 参数调优

```properties
# === 网络线程 ===
num.network.threads=8           # CPU 核数相关
num.io.threads=16               # 2x network.threads

# === 日志存储 ===
log.segment.bytes=1073741824    # 1GB per segment
log.retention.hours=168         # 7 天
log.retention.bytes= -1         # 无大小限制

# === 副本拉取 ===
num.replica.fetchers=4          # 副本同步线程数
replica.fetch.max.bytes=10485760  # 10MB 每次拉取

# === Page Cache ===
# 直接依赖 OS 管理的 Page Cache
# 内存配比：Broker Heap 6GB + OS Cache 留 16GB+
```

---

# 四、JVM GC 优化

```bash
# Kafka Broker 推荐 GC 配置
KAFKA_HEAP_OPTS="-Xmx6g -Xms6g"
KAFKA_JVM_PERFORMANCE_OPTS="
  -XX:+UseG1GC
  -XX:MaxGCPauseMillis=20
  -XX:InitiatingHeapOccupancyPercent=35
  -XX:+DisableExplicitGC
"
```

---

# 五、OS 内核参数

```bash
# === 文件句柄 ===
ulimit -n 100000

# === 网络调优 ===
net.core.wmem_default=1048576
net.core.rmem_default=1048576
net.core.wmem_max=16777216
net.core.rmem_max=16777216

# === 虚拟内存（避免 OOM Killer） ===
vm.swappiness=1
vm.dirty_ratio=20

# === 文件系统 ===
# 挂载选项：noatime（不记录访问时间）
mount -o noatime /dev/sdb /data/kafka

# === 磁盘调度器 ===
echo noop > /sys/block/sdb/queue/scheduler
```

---

# 六、监控核心指标

| 指标 | 告警阈值 | 含义 |
|------|---------|------|
| `UnderReplicatedPartitions` | >0 | 有副本未同步 |
| `ActiveControllerCount` | ≠1 | Controller 故障 |
| `BytesInPerSec` | >80% 带宽 | 流量接近带宽上限 |
| `RequestQueueSize` | >1000 | Broker 处理不过来 |
| `ISRShrinkPerSec` | >0 | ISR 缩容，Follower 跟不上 |

---

# 七、总结

| 层级 | 核心调优 |
|------|---------|
| **生产者** | batch + linger + lz4 压缩 |
| **Broker** | G1 GC + 充足 Page Cache + noatime 挂载 |
| **OS** | 文件句柄 100k + 网络缓冲区 |
| **架构** | 分区数宁少勿多，单分区 10MB/s 够用 |

*本文参考资料：*
- Neha Narkhede et al.《Kafka: The Definitive Guide》（第 2 版）——第 1-6 章（架构、生产者、消费者、内部机制）
- Apache Kafka 官方文档 - Design 章节: https://kafka.apache.org/documentation/#design
- Apache ZooKeeper 官方文档: https://zookeeper.apache.org/doc/current/
- Flavio P. Junqueira, Benjamin Reed《ZooKeeper: Distributed Process Coordination》
- Linux 内核文档 - sendfile(2) / mmap(2) / Page Cache
- KIP-500: Replace ZooKeeper with a Self-Managed Metadata Quorum (KRaft)
