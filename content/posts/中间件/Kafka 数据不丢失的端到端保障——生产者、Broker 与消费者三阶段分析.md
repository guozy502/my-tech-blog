# Kafka 数据不丢失的端到端保障——生产者、Broker 与消费者三阶段分析

> Kafka 的"数据不丢失"不是一个配置项就能解决的——它需要生产端、Broker 端、消费端三方协同。任何一个环节配置错了，数据就丢了。本文从每个阶段的丢数据场景出发，倒推需要做什么配置、为什么有效、以及配置之间的制约关系。

---

## 一、数据可能在三个环节丢失

```
Producer ──→ Broker ──→ Consumer
   ↓           ↓           ↓
 发送失败    副本故障    消费失败
 确认丢失    数据损坏    偏移量错误
```

发出去之前、存下来的那一刻、消费提交的时候——每个环节都有丢掉数据的窗口。理解每个环节"到底怎么丢的"，才能理解配置"为什么这么设"。

---

## 二、Producer 端：消息发出去不等于到了 Broker

### 2.1 `acks`——Producer 的核心配置

`acks` 控制 Producer 认为一条消息"发送成功"需要多少确认：

| 值 | 行为 | 丢失风险 | 延迟 |
|----|------|---------|------|
| `acks=0` | 发出去就算成功，不等待任何确认 | **极高**：网络断、Broker 挂了都不知道 | 最低 |
| `acks=1` | Leader 写入本地日志即返回成功 | **中**：Leader 刚写完就宕机，Follower 没同步 = 数据丢失 | 较低 |
| `acks=all` (或 -1) | 所有 ISR 中的副本都确认后才返回成功 | **极低**：只要 ISR 中至少还有一个副本活着，数据就不丢 | 最高 |

**`acks=1` 丢数据的典型场景**：

```
1. Producer 发送 msg-100 → Leader 写入成功 → 返回 ACK 给 Producer
2. Leader 在将 msg-100 复制给 Follower 之前宕机
3. Follower 成为新 Leader → msg-100 不在新 Leader 的日志中
4. Producer 以为 msg-100 发送成功，但实际上丢了
```

这个窗口很短——Leader 写完本地日志到 Follower 拉取复制之间通常只有几毫秒——但对于"绝对不能丢"的数据，这个窗口必须用 `acks=all` 关掉。

### 2.2 `min.insync.replicas`——跟 acks=all 配合的底线

`acks=all` 只要求"所有 ISR 中的副本确认"，但如果 ISR 里只有一个副本（本身就是 Leader），acks=all 退化成了 acks=1——仍然可能丢数据。

**这就是为什么 `min.insync.replicas` 必须配置**：

```
如果 ISR = [Broker A(Leader)]，min.insync.replicas = 1:
  Producer → Leader 写入成功 → 只有 Leader 有数据 → Leader 宕机 → 数据丢失

如果 ISR = [Broker A(Leader), Broker B(Follower)]，min.insync.replicas = 2:
  Producer → Leader 写入 → 等待 B 确认 → 两个副本都有数据 → Leader 宕机 → B 接任 → 数据不丢
  ISR 数量不足 2 时（如 B 宕机），Leader 拒绝写入 → Producer 收到异常 → 不丢数据（宁可不可用也不丢）
```

**典型推荐配置**：

```properties
acks=all
min.insync.replicas=2
replication.factor=3
```

解释：3 副本 + 至少 2 个 ISR（包括 Leader）同步确认 + Producer 等所有 ISR 确认。允许 1 个副本宕机不影响写入。2 个副本同时宕机时写入被拒绝——这是"可用性"换"可靠性"的明确取舍：宁可暂时写不进去，也不丢数据。

为什么 replication.factor=3 不设为 2？因为 min.insync.replicas=2 时，3 副本允许 1 台宕机而写入不受影响；2 副本的情况下一旦有 1 台宕机，ISR 不足 2，写入立刻被阻塞——运维上没有缓冲空间。

### 2.3 `enable.idempotence`——精确一次 + 不丢消息的最后一层保护

即使 `acks=all`，Producer 也可能在 Broker 写入成功后、返回 ACK 之前，网络中断——Producer 没收到 ACK，会重试。重试的消息跟之前成功的消息重复。

开启幂等性后，Producer 给每条消息分配 `<ProducerId, SequenceNumber>`。Broker 发现 sequence number 重复 → 直接丢弃 → 返回成功（不写盘，但告诉 Producer 成功了）。不会有重复消息进入日志。

```properties
enable.idempotence=true          # 开启幂等（Kafka 3.0+ 默认）
max.in.flight.requests.per.connection=5  # 幂等开启后可安全设为 5
```

**注意**：`enable.idempotence=true` 会自动设置 `acks=all`、`retries=Integer.MAX_VALUE`、`max.in.flight.requests.per.connection=5`（单个连接最多 5 个未完成的请求，在高吞吐场景下比设为 1 好得多）。这是 Kafka 的默认推荐配置——除非你知道自己在做什么，不要改为手动覆盖。

### 2.4 Producer 端的三道保险总结

```
acks=all               ← 第一道：等所有 ISR 确认
min.insync.replicas=2  ← 第二道：ISR 不够就拒绝写入（宁可不可用也不丢）
enable.idempotence=true ← 第三道：重试不产生重复，防止 ACK 丢失导致的重试覆盖
```

---

## 三、Broker 端：数据存下来还要保证坏不了

### 3.1 ISR 机制——丢了 Follow 也不丢数据

Kafka 的 ISR (In-Sync Replicas) 是动态的：Follower 同步足够快 → 留在 ISR；Follower 落后太多 → 从 ISR 踢出。只有 ISR 中的副本有资格在 Leader 宕机时接任。

```
Leader: msg-0, msg-1, msg-2, msg-3
Follower A: msg-0, msg-1, msg-2         → 落后 1 条，在 ISR 中
Follower B: msg-0, msg-1                → 落后 2 条，超过阈值 → 被踢出 ISR

如果 acks=all, min.insync.replicas=2 → Producer 写入 msg-4：
  Leader 写成功 → 等 Follower A 确认 → OK（2 个副本确认）
  Follower B 不在 ISR 中 → 不参与确认
```

**ISR 的边界条件**：`replica.lag.time.max.ms`（默认 30s）控制 Follower 何时被踢出 ISR。这个值是"多长內没同步"的时间阈值，不是"落后了多少条消息"的数量阈值——因为消息流量可能波动很大，按条数算不准确。如果 Follower 能在 30s 内追上，就留在 ISR 中。

### 3.2 `unclean.leader.election.enable`——宁可不可用，也不丢数据

当 ISR 中**没有任何副本**存活时（比如三副本的 Broker 全挂了，且没来得及恢复），Kafka 面临一个选择：是等 ISR 中的 Broker 恢复，还是从 ISR 之外的"落后了很多数据"的副本中选一个当 Leader？

```properties
# 生产环境必须设 false（Kafka 2.4+ 默认 false）
unclean.leader.election.enable=false
```

- `true`：选 ISR 外的 Follower 当 Leader → 服务恢复速度快 → 但这个 Follower 磁盘上的数据不完整 → 数据丢失且消费者感知不到
- `false`：拒绝选举 → 该分区不可用 → 持续报错 → 等到 ISR 中的副本恢复 → 数据不丢

**推荐**：核心数据链路设为 `false`。日志、监控类的非核心数据可以设为 `true` 优先级在可用性。

### 3.3 日志刷盘——Kafka 不靠它保证数据安全

很多从数据库转 Kafka 的工程师会问："Kafka 怎么保证数据刷盘不丢？"

答案是：**Kafka 不保证刷盘**。Kafka 的数据安全不依赖于把数据刷到磁盘——依赖的是**多副本复制**。副本刷盘间隔由 OS 的 page cache 管理（`flush.messages` 和 `flush.ms` 默认都不设，交给 OS）。

```
数据库思维: 写 WAL → fsync → 确认 → 单机故障可恢复（靠 fsync 保证单机持久化）
Kafka 思维:  写 page cache → 多副本确认 → 返回成功 → 多机同时故障才会丢数据（靠多副本保证）
```

**为什么 Kafka 不依赖 fsync？**

fsync 极慢——物理磁盘一次 fsync 的延迟是毫秒级的，高频 fsync 直接打穿 Kafka 的吞吐优势。Kafka 选择用多副本替代单机 fsync：三个 Broker 同时 page cache 丢失（三台同时断电）的概率远小于单机 fsync 后磁盘损坏的概率。

**一个常见的配置错误**：把 Kafka 的 `flush.messages=1`（每条消息都 fsync）——这会让 Kafka 的写入吞吐暴跌到几千条/秒，同时并没有提高数据安全性（因为不解决多副本问题）。这不是 Kafka 设计的使用方式。

### 3.4 Broker 端数据安全配置总结

```properties
# 副本配置
default.replication.factor=3              # 至少 3 副本
min.insync.replicas=2                     # 至少 2 个 ISR 确认（Topic 级别覆盖）

# 日志保留
log.retention.hours=168                   # 7 天——给数据恢复留足时间窗口
log.segment.bytes=1073741824              # 1GB 一个 segment

# 不靠谱的 Leader
unclean.leader.election.enable=false      # ISR 外不允许选主

# 不要手动设置 flush.messages 和 flush.ms
# 让 OS 的 page cache 管理刷盘
```

---

## 四、Consumer 端：消息被处理了不代表被安全提交了

### 4.1 自动提交的坑——最容易丢数据的地方

```java
// ❌ 自动提交：消息可能还没处理完 offset 就提交了
props.put("enable.auto.commit", "true");
props.put("auto.commit.interval.ms", "5000");

// 典型丢失场景：
// 1. poll() 拿到 100 条消息 → 自动提交 offset=100
// 2. 处理到第 50 条时消费者宕机
// 3. 重启后从 offset=100 开始消费
// 4. 第 51-100 条——丢了！
```

**自动提交的二阶丢失**：消息被 poll 出来、还没处理完、但定时提交的钟到了，offset 被提交到"已拉取的位置"而不是"已处理完的位置"。Consumer 挂了重启后从提交位置续消费，中间那批"拉了但没处理完"的消息永远消失了。

### 4.2 手动提交的正确姿势

```java
// ✅ 手动提交：处理完再提交
props.put("enable.auto.commit", "false");

while (true) {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(1000));
    
    for (ConsumerRecord<String, String> record : records) {
        try {
            processRecord(record);  // 真正处理完成
        } catch (Exception e) {
            log.error("处理失败: offset={}", record.offset(), e);
            // 不提交 offset！
            // 可以塞到死信队列，也可以直接抛出让消费者退出
            break;  
        }
    }
    
    // 所有消息处理成功后才提交
    // 提交的是 offset+1（下一次该从哪开始）
    consumer.commitSync();  // 同步提交：阻塞直到 Broker 确认
}
```

**`commitSync` vs `commitAsync`**：

- `commitSync`：阻塞直到 Broker 确认 offset 提交成功。提交失败会自动重试。可靠但慢。
- `commitAsync`：发提交请求后立刻返回，不等待 Broker 确认。提交失败不会自动重试（因为重试可能乱序）。快但有丢失风险。
- 生产实践：正常循环用 `commitAsync`（快速不阻塞），关闭时用 `commitSync` 做最后一次兜底提交：

```java
// 正常循环：异步提交，不阻塞
consumer.commitAsync((offsets, exception) -> {
    if (exception != null) {
        log.error("异步提交失败: {}", offsets, exception);
        // 记录失败，但不重试（防止乱序）
    }
});

// JVM 关闭钩子：最后一次同步提交
Runtime.getRuntime().addShutdownHook(new Thread(() -> {
    consumer.commitSync();  // 阻塞直到提交完成
    consumer.close();
}));
```

### 4.3 消息处理失败的应对——不提交 offset 的代价

消费者处理某条消息失败时怎么办？核心原则：**处理失败就不提交 offset**。

但这会导致两个后果：
- 重平衡后别的消费者会重新拉这批消息 → 消息至少被处理一次（at-least-once）→ **业务层必须做幂等**
- 如果失败是永久性的（消息格式损坏、业务规则不允许），同一个消费者会反复卡在同一偏移量——消息本身没问题但业务处理永远失败。此时需要**死信队列**

```java
// 死信队列模式
for (ConsumerRecord<String, String> record : records) {
    try {
        processRecord(record);
    } catch (NonRetriableException e) {
        // 永久性错误 → 不阻塞整个消费流
        deadLetterQueue.send(record.topic(), record.key(), record.value());
        log.error("消息进入死信队列: key={}, error={}", record.key(), e.getMessage());
        // 继续处理下一条（这条会在死信队列中被单独处理）
    } catch (RetriableException e) {
        // 临时性错误 → 停止当前批次处理
        log.error("临时错误，停止消费: {}", e.getMessage());
        break;  // 不提交 offset，下次重新拉取
    }
}
consumer.commitSync();
```

**死信队列不能自动处理**——死信中的消息需要人工介入判断是否修复后重新投递，或者确认可以丢弃。自动丢弃死信就失去了它作为最后防线的意义。

### 4.4 消费者组 Rebalance 导致的数据丢失

Rebalance 期间，原消费者持有的分区被分配给新消费者。如果原消费者还没处理完当前批次就失去了分区：

```
1. Consumer A 正在处理 partition-0 的 offset 100-200
2. Consumer A 因 GC 停顿被踢出消费者组（max.poll.interval.ms 超时）
3. Rebalance 触发：partition-0 分配给 Consumer B
4. Consumer B 从上次提交的 offset=100 开始消费
5. Consumer A 已经处理完的 101-150 被 Consumer B 重复处理

→ 消息至少被处理一次（如果 Consumer A 在 GC 前已经提交了 150 就不会重复）
```

**应对**：
```properties
# Consumer 配置
max.poll.interval.ms=300000        # 两次 poll 之间最大间隔，超过触发 rebalance
session.timeout.ms=45000           # Consumer 心跳超时
heartbeat.interval.ms=3000         # 心跳间隔
max.poll.records=500               # 每次 poll 最多拉多少条——不设太大避免处理时间过长
```

关键：`max.poll.records` 不能设得太大（很多人设 5000+），`max.poll.interval.ms` 要大于最坏情况下的单批处理时间。`max.poll.interval.ms` 的倒计时是从 `poll()` 调用返回时开始的，不是从消息开始处理时开始的——所以如果你的处理逻辑中有外部 RPC 调用，RPC 的超时必须远小于 `max.poll.interval.ms`。

### 4.5 Consumer 端数据安全配置总结

```properties
enable.auto.commit=false                    # 手动提交
auto.offset.reset=earliest                  # 无 offset 时从最早开始（防止 gap）
max.poll.records=500                        # 控制单批次量
max.poll.interval.ms=300000                 # 处理超时上限
```

---

## 五、端到端不丢消息的配置模板

```properties
# ========== Producer ==========
acks=all
enable.idempotence=true
max.in.flight.requests.per.connection=5
retries=2147483647                          # 最大重试（Integer.MAX_VALUE）
delivery.timeout.ms=120000                  # 累计重试总时间上限

# ========== Broker ==========
default.replication.factor=3
min.insync.replicas=2
unclean.leader.election.enable=false
log.retention.hours=168

# ========== Consumer ==========
enable.auto.commit=false
auto.offset.reset=earliest
max.poll.records=500
max.poll.interval.ms=300000
```

这套配置的代价是**性能**：
- `acks=all` 比 `acks=1` 延迟高 ~30%（多等一次副本确认的网络 RTT）
- `min.insync.replicas=2` 在副本故障时拒绝写入——可用性下降
- 手动提交比自动提交更阻塞——消费吞吐下降

**取舍原则**：核心数据链路（订单、支付、账户）用全保配置；日志、监控、点击流用低代价配置（`acks=1` + 自动提交 + 1 副本）。

---

## 六、排查"数据到底丢在哪了"

### 6.1 检查清单

```
□ Producer 端
  □ acks 配置
  □ enable.idempotence 是否开启
  □ send() 返回值或 Callback 是否检查了异常
  □ retries 是否足够

□ Broker 端
  □ Topic 的 replication.factor
  □ min.insync.replicas 值
  □ ISR 是否有持续缩容（Follower 被频繁踢出）
  □ unclean.leader.election.enable 是否 false

□ Consumer 端
  □ enable.auto.commit 是否关了
  □ max.poll.records 是否过大
  □ 异常时是否处理了（而不是吞掉）
```

### 6.2 Kafka 自带的监控指标

```
# Producer（检查发送端是否有异常丢消息）
kafka.producer:type=producer-metrics,client-id=*
  record-error-rate      → 发送失败率 > 0 说明有消息没发出去
  record-retry-rate      → 重试率异常高说明 Broker 端有问题

# Broker（检查 ISR 健康度）
kafka.server:type=ReplicaManager,name=UnderReplicatedPartitions
  → 任何分区的副本数 > 1 且长时间未复制完成 → ISR 异常缩小

# Consumer（检查消费端是否有漏消费）
kafka.consumer:type=consumer-fetch-manager-metrics,client-id=*
  records-lag-max       → 最大 lag（消费延迟）
  fetch-rate            → 消费速率
```

### 6.3 命令行排查

```bash
# 查看 Topic 的 ISR 情况
kafka-topics --describe --bootstrap-server localhost:9092 --topic <topic>
# 关注 Isr 列：副本数与 min.insync.replicas 的关系
# 如果 ISR 数量不确定地波动 → Follower 可能有性能瓶颈

# 查看消费者组 offset 和 lag
kafka-consumer-groups --bootstrap-server localhost:9092 --group <group> --describe
# CURRENT-OFFSET: 消费者当前消费到的位置
# LOG-END-OFFSET: 分区最新消息位置
# LAG: 差值 = 还有多少条消息等待消费
# LAG 持续增长 → 消费端处理不过来

# 查看 Under Replicated Partitions（持续 >5min 要警惕）
kafka-topics --describe --bootstrap-server localhost:9092 --under-replicated-partitions
```

---

数据不丢不是某一端的责任。Producer 配置 `acks=all` 但 Consumer 自动提交——一样丢。Broker 三副本但 `unclean.leader.election.enable=true`——一样丢。只有 Producer、Broker、Consumer 三端都配置正确，且业务消费逻辑做了幂等，才算真正做到了"端到端的数据不丢失"。
