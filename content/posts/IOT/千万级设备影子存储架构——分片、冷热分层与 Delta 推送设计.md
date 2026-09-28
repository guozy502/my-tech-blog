---
title: "千万级设备影子存储架构——分片、冷热分层与 Delta 推送设计"
date: 2026-08-08
description: 从千万级 IoT 设备影子存储的分片路由（按 deviceId 一致性 Hash → 分片节点）、热冷数据分层（Redis 活跃期 → RocksDB 温数据 → S3 冷归档）、Delta 推送的 MQTT 削峰与版本冲突解决、到影子读写的 CAP 取舍（AP 优先 + 最终一致的 reported），拆解大规模设备影子平台的完整存储、同步与高可用架构设计。
tags: ["架构","IoT","设备影子","分片","Redis","Delta推送"]
categories: ["IOT"]
---

# 历史背景——影子从"一个 JSON"到"一个分布式系统"

AWS IoT Device Shadow 在 2015 年发布时，底层只是一个 DynamoDB 表。对于几万台设备，一个 JSON 文档 + 乐观锁版本控制就足够。

当设备规模冲破百万甚至千万级时，"一个 JSON"变成了"一个分布式存储系统"：
- 千万个影子，每个 1-10KB → 总数据 10-100GB（不算大）
- 但 QPS 完全不同：活跃设备每秒推送一次状态 → 10 万台活跃设备 = 10 万 writes/s
- 还有读负载：用户 App 频繁查询设备状态 → 缓存击穿风险
- 还有 Delta 推送：每个 desired 更新都可能触发一次 MQTT 推送 → 推送风暴

影子存储的瓶颈不在"数据存不存得下"，而在"读写能不能撑住"。这篇拆解从单影子到千万级影子的架构进化路径。

---

# 一、影子的访问模式分析——先搞清楚负载

## 1.1 影子的读写特征

```
写操作（设备 → 影子）：
  - 频率：活跃设备每 1-5 秒上报一次状态
  - 模式：增量更新（单个 shadow 一次只改 2-3 个字段，不是整个 JSON）
  - 热点：无热点——设备 ID 天然均匀分布（不像社交媒体有热点话题）

读操作（App/规则引擎 → 影子）：
  - 频率：用户打开 App → 查该用户所有设备的影子（1-5 次查询）
  - 模式：大多是"全量读"（读整个影子 JSON）
  - 热点：有热点——某些设备（如公共场所的监控设备）被大量用户同时查询
  - 时效性要求：1-5 秒的延迟可接受

Delta 推送（影子 → 设备）：
  - 频率：取决于应用更新 desired 的频率
  - 模式：按设备独立推送（没有"全量广播"的推送模式）
  - 关键：推送不能丢——设备依赖 Delta 来改变设置
```

## 1.2 数据量的量级估算

```
1000 万设备 × 每个影子 8KB = 80GB 总数据（完全可以放在内存用 Redis 存）
  但如果 100 万台"活跃"设备才有热查询
  → 不需要全量存 Redis → 冷数据可以下沉

QPS 估算：
  100 万活跃设备 × 0.2 writes/s = 20 万 writes/s
  50 万活跃用户 × 0.1 reads/s = 5 万 reads/s
  → 单机 Redis 扛不住，MySQL 也扛不住 → 必须分片
```

---

# 二、影子存储分片——一致性 Hash 路由

## 2.1 分片键的选择——天然的 deviceId

```
影子查询的特点：
  - 99% 的读写指定了 deviceId（GET / UPDATE shadow by deviceId）
  - 几乎没有"跨设备"的查询（不需要 JOIN 或聚合）

→ 分片键直接用 deviceId
→ 同一个设备的 desired 和 reported 永远在同一分片（本地事务即可保证一致性）
```

## 2.2 一致性 Hash——扩容友好

```
为什么不用 Hash(deviceId) % N？
  N 从 10 → 11 → 大部分数据的分片位置都变 → 全量搬迁

一致性 Hash 的解决方案：
  ① 环形空间 [0, 2^32-1]，10 个物理节点均匀分布在环上
  ② 每个物理节点有 150 个虚拟节点（均匀散列在环上）
  ③ 影子落在 hash(deviceId) 顺时针遇到的第一个虚拟节点 → 这个虚拟节点归属的物理节点

扩容（10 → 11）：
  - 新增节点插入环中，与它相邻的虚拟节点之间的数据迁移到新节点
  - 迁移量 ≈ 数据总量 / 新节点数 ≈ 10%
  - 不是 90%！
```

```java
// 一致性 Hash 路由的核心逻辑（简化）
public class ShadowRouter {
    private final TreeMap<Integer, ShadowShard> ring = new TreeMap<>();
    
    // deviceId → 分片
    public ShadowShard route(String deviceId) {
        int hash = hash(deviceId);
        // 顺时针找最近的虚拟节点
        Map.Entry<Integer, ShadowShard> entry = ring.ceilingEntry(hash);
        if (entry == null) entry = ring.firstEntry();  // 环尾 → 环头
        return entry.getValue();
    }
    
    // 扩容——新分片加入环中
    public void addShard(ShadowShard shard) {
        // 为每个虚拟节点插入环中
        for (int i = 0; i < 150; i++) {
            ring.put(hash(shard.getId() + "#" + i), shard);
        }
        // 新分片只需要从相邻分片迁移"自己负责"的影子
        // 不需要全量搬迁
    }
}
```

## 2.3 每个分片内部——两级存储

```
每个分片节点内部：

┌──────────────────────────────┐
│  L1: Redis Cluster (热数据)    │  ← 最近 15 分钟活跃的设备影子
│  - 持久化 + 主从 + 哨兵        │  ← 写 reported、读查询
│  - 15 分钟无活动 → L2          │
├──────────────────────────────┤
│  L2: RocksDB (温数据)          │  ← 15 分钟 - 24 小时未活跃
│  - SSD 持久化                  │  ← active flag = false
│  - 读时回 Redis + 续热          │
├──────────────────────────────┤
│  L3: S3/HDFS (冷归档)          │  ← 离线 > 24 小时的设备影子
│  - 极低成本                    │  ← 设备重新上线 → 从 S3 加载 → L1
└──────────────────────────────┘
```

**冷热迁移策略**：

```java
public class ShadowTierManager {
    
    // 活动超时检查（每分钟）
    @Scheduled(fixedRate = 60000)
    public void evictInactive() {
        // ① 找出 Redis 中超过 15 分钟未活跃的影子
        Set<String> inactiveKeys = redis.scan("shadow:*");
        for (String key : inactiveKeys) {
            long lastActive = redis.hget(key, "lastActive");
            if (now - lastActive > 900_000) {       // 15 分钟
                byte[] shadowData = redis.get(key);
                // ② 存入 RocksDB（异步）
                rocksDB.put(key, shadowData);
                // ③ 从 Redis 移除
                redis.del(key);
            }
        }
        
        // ④ 找出 RocksDB 中超过 24 小时未活跃的影子
        // ⑤ 存入 S3 → 从 RocksDB 移除
    }
    
    // 影子被访问时（主动续热）
    public ShadowDocument getShadow(String deviceId) {
        // ① 先查 Redis
        ShadowDocument shadow = redis.get(deviceId);
        if (shadow != null) return shadow;
        
        // ② 查 RocksDB
        shadow = rocksDB.get(deviceId);
        if (shadow != null) {
            // ③ 续热——放回 Redis
            redis.setex(deviceId, 900, shadow);  // 15 分钟 TTL
            return shadow;
        }
        
        // ③ 查 S3
        shadow = s3.get("shadows/" + deviceId);
        if (shadow != null) {
            // 回 L1 → 设备重新上线了
            redis.setex(deviceId, 900, shadow);
        }
        return shadow;
    }
}
```

---

# 三、读写分离与一致性

## 3.1 Desired 和 Reported 的两种读取路径

```
场景 A：App 读设备状态（99% 走缓存）：
  App → GET shadow by deviceId → Nginx → 影子服务 → Redis(L1)
    → Redis 命中有热数据 → 直接返回 ✓（< 1ms）
    → Redis 未命中有冷数据 → 回 RocksDB(L2) → 续热 → 返回（~5ms）

场景 B：设备写入 reported（直写主存储）：
  设备 → MQTT → 影子写服务 → 写 Redis(L1) + 异步写 RocksDB(L2)
    → 同时通知"该影子的缓存有更新"→ App 的后续读取会拿到新数据

场景 C：设备上报 reported 触发联动：
  写 Redis → 发 Event(ShadowUpdated) → 规则引擎消费 → 执行联动
```

## 3.2 一致性的权衡——AP 优先

```
影子读写的 CAP 分析：
  - P（网络分区）必然发生 → 不能完全避免
  - 读：App 读到"稍微旧一点"的 reported 是可接受的（1-2 秒的延迟）
  - 写：设备上报 reported 不能丢 → 需要确保写成功

→ 影子选择 AP（可用 + 分区容忍，写不丢，读可接受短暂不一致）
→ 不要 C（读到的一定是最新的）——代价太高，不值得

具体手段：
  - Redis(L1) 的主从异步复制 → 读可能命中旧数据（主已更新但从未同步完）
  - 对时效要求高的查询走 Master（write-through），普通查询走 Slave
  - 写 reported 走 Redis 主节点 → 写成功才返回 MQTT ACK
```

## 3.3 版本号与冲突解决——三维 version

```
单一 version 的问题：所有字段共用一个版本 → 读 color=red → 改 power=off → 版本冲突

字段级 version 的解法：

{
  "state": {
    "desired": {
      "color": "red",      ← version: 12
      "power": "on"        ← version: 8
    },
    "reported": {
      "color": "red",      ← version: 12
      "power": "on"        ← version: 8
    }
  },
  "shadowVersion": 15      ← 影子整体版本（用于全量恢复）
}

App A UPDATE desired.color="blue" (基于 version 12) → 比较 version == 12 → 是 → 更新为 13 → ✓
App B UPDATE desired.power="off" (基于 version 8) → 比较 version == 8 → 是 → 更新为 9 → ✓
  → 两个修改改不同字段 → 不冲突 → 两个都接受

App C UPDATE desired.color="green" (基于 version 12) → 比较 version == 12 → 否（13） → 拒绝 ✗
  → 基于旧版本修改 → 拒绝 → App C 重新读 → 重试
```

---

# 四、Delta 推送——"怎么告诉设备 desired 变了"

## 4.1 Delta 的计算逻辑

```java
public class DeltaCalculator {
    
    // 每次 desired 或 reported 更新后，重新计算 delta
    public Map<String, Object> computeDelta(String deviceId) {
        ShadowDocument shadow = shadowStore.get(deviceId);
        Map<String, Object> desired = shadow.getDesired();
        Map<String, Object> reported = shadow.getReported();
        
        Map<String, Object> delta = new HashMap<>();
        
        for (Map.Entry<String, Object> entry : desired.entrySet()) {
            String key = entry.getKey();
            Object desiredValue = entry.getValue();
            Object reportedValue = reported.get(key);
            
            // 只比较 desired 中实际存在的 key
            // reported 中不存在的 key → 肯定在 delta 中
            // reported 中的值和 desired 不同 → 在 delta 中
            if (!desiredValue.equals(reportedValue)) {
                delta.put(key, desiredValue);
            }
        }
        
        // Delta 为空 → desired == reported → 同步完成 ✓
        // Delta 非空 → 发布到 MQTT Topic → 推给设备
        return delta;
    }
}
```

## 4.2 推送的削峰——推送风暴的防护

```
场景：一个应用批量更新 10 万台设备的 desired（如"夜间模式"全开）
  
  如果直接推 → 10 万个 MQTT 消息瞬间喷出 → MQTT Broker 炸

削峰策略：
  ① 限制推送速率：每 100ms 最多推 1000 个设备
  ② 按优先级排队：紧急设备（如医疗设备）优先推送
  ③ 离线设备的 delta 不推——存影子，设备上线时自己拉

```java
public class DeltaPushScheduler {
    private final RateLimiter limiter = RateLimiter.create(10000);  // 每秒 1 万推送
    
    // 批量 desired 更新 → 非即时推
    public void scheduleBatchPush(List<String> deviceIds, Map<String, Object> desired) {
        // ① 写入影子（不阻塞）
        deviceIds.parallelStream().forEach(id -> {
            shadowStore.updateDesired(id, desired);
            deltaQueue.offer(new PushTask(id, computeDelta(id)));
        });
        
        // ② 后台线程按速率推送
        executor.submit(() -> {
            while (!deltaQueue.isEmpty()) {
                limiter.acquire();
                PushTask task = deltaQueue.poll();
                mqtt.publish(task.deviceId, task.delta);
            }
        });
    }
    
    // 单设备 desired 更新 → 即时推
    public void scheduleImmediatePush(String deviceId, Map<String, Object> desired) {
        shadowStore.updateDesired(deviceId, desired);
        mqtt.publish(deviceId, computeDelta(deviceId));  // 直接推，不排队
    }
}
```

## 4.3 Delta 推送的三次确认

```
① MQTT QoS 1 ：消息至少到达一次（MQTT Broker 确认收到）
② Delta 确认：设备收到 delta → 执行变更 → 更新 reported
   → 如果设备不更新 reported → 5 分钟后重推 delta
③ 收敛确认：desired == reported → delta 清空 → 推送完成
   → 如果 after 10 分钟 still desired ≠ reported → 设备可能离线
   → 返回步骤 ②，但不再重推（等设备上线自己拉）
```

---

# 五、高可用——分片故障怎么处理

```
场景：分片节点 3（负责 10% 的设备）宕机

影响范围：这 10% 的设备影子无法读写
恢复流程：
  ① 检测：分片节点心跳超时 → 从路由环中摘除故障节点
  ② 重新路由：原属于故障节点的影子重新映射到相邻节点
     使用一致性 Hash → 相邻分片（hash 环上的下一个节点）接管
  ③ 数据恢复：相邻分片从 RocksDB 的主从备份或 S3 冷归档加载数据
  ④ 设备感知：设备下一次 GET/PATCH 影子 → 自动路由到新分片
```

---

# 六、总结

| 问题 | 解法 | 为什么 |
|------|------|--------|
| **千万影子存在哪** | 一致性 Hash 分片 | deviceId 天然无热点 |
| **热冷数据差异** | Redis(L1) → RocksDB(L2) → S3(L3) | 99% 的查询只命中最近活跃的 10% 设备 |
| **Desired/Reported 冲突** | 字段级 version + 乐观换锁 | 不同字段的并发修改不应该冲突 |
| **批量推送** | RateLimiter 削峰 + 优先级排队 | 10 万推送瞬间喷出会炸 MQTT Broker |
| **分片故障** | 一致性 Hash 邻接接管 | 只影响对应分片的设备 |

# 延伸阅读

**Do——动手设计：**
- 用 Redis + RocksDB + 一致性 Hash 写一个最小影子存储（1000 个 device，2 个分片），测试分片迁移和冷热数据切换
- 模拟 10 万 desired 批量更新 → 观察 Delta 推送的速率控制和队列堆积
- 用 JMeter 对影子读接口做压测（10000 QPS），观察 Redis 命中率和 RocksDB 回源延迟

**Todo——深入方向：**
- 影子同步的离线补偿——设备离线 30 天后重新上线，如何增量同步影子变更
- 多租户影子隔离——不同企业客户的影子数据在同一集群中的逻辑/物理隔离策略
- AWS IoT Shadow 服务端源码级别的架构分析——对比自研影子和 AWS 实现的差异

*本文参考资料：*
- AWS IoT Device Shadow 文档: https://docs.aws.amazon.com/iot/latest/developerguide/iot-device-shadows.html
- 一致性 Hash 论文: "Consistent Hashing and Random Trees" (Karger et al., 1997)
- 你的博客: [设备影子详细设计——云端与设备的状态同步机制](/posts/iot/设备影子详细设计云端与设备的状态同步机制/)
