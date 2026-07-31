---
title: "海量数据处理与工程算法——BitMap、布隆过滤器、HyperLogLog、一致性Hash与限流"
date: 2026-07-31
description: 深入BitMap的节省内存原理、布隆过滤器的误判率推导、HyperLogLog的概率计数、一致性Hash的虚拟节点机制、雪花算法的时钟回拨应对、四种限流算法对比，给出Java/Python工程级实现。
tags: ["算法","BitMap","布隆过滤器","HyperLogLog","一致性Hash","雪花算法","限流算法"]
categories: ["算法"]
---

# 海量数据处理与工程算法——BitMap、布隆过滤器、一致性 Hash 与限流

> 这些算法有一个共同特征：它们不是"解 LeetCode 题"用的，而是在真实系统中处理海量数据时节省内存、提升性能和保证可靠性的基础设施。理解它们的原理，是架构师和资深工程师的基本功。

---

## 一、BitMap——1 bit 存一个数

### 1.1 核心原理

用一个 bit 数组标记某个数是否存在。第 i 位 = 1 表示数 i 出现过。

```
40 亿个非负整数（范围 0 ~ 2³²-1），去重后有多少个不重复的数？
→ BitMap 需要 2³² bit = 512MB 内存
→ 如果用 HashSet<Integer>（每个 Integer 占 16+ 字节）= 64GB+
→ BitMap 节省 100 倍+ 内存
```

**定位公式**：

```java
// 将数 num 映射到 BitMap
int byteIndex = num / 8;           // 在哪个字节
int bitIndex  = num % 8;           // 在字节中哪个 bit

// 置位
bitmap[byteIndex] |= (1 << bitIndex);

// 查询
boolean exists = (bitmap[byteIndex] & (1 << bitIndex)) != 0;
```

### 1.2 RoaringBitmap——稀疏 BitMap 的压缩

如果数据很分散（比如只有 1 和 10 亿两个数），512MB BitMap 中只有两个 1 位——极度浪费。RoaringBitmap 是大多数大数据引擎使用的 BitMap 实现：

```
思路：把 32 位整数分成高 16 位（桶号）和低 16 位（桶内偏移）

高 16 位的 2¹⁶ = 65536 个桶，每个桶内三种存储方式:
  - Array Container: 桶内 ≤ 4096 个元素 → 用 short[] 存
  - Bitmap Container: 桶内 > 4096 个元素 → 用 8KB BitMap
  - Run Container: 连续递增序列 → 用 (start, length) 压缩

每个桶根据自身密度自动切换最优存储方式
```

Spark、Hive、ClickHouse 的去重和计数优化大量依赖 RoaringBitmap。

### 1.3 实际场景

- **用户签到统计**：365 天的签到记录 = 365 bit ≈ 46 字节/用户
- **布隆过滤器的底层存储**：BitMap 的数组就是 Bloom 的位向量
- **去重**：IP 去重、日志 ID 去重

---

## 二、布隆过滤器

### 2.1 核心原理

布隆过滤器 = **K 个哈希函数 + 一个 BitMap**。插入时对元素计算 K 个哈希值，将 BitMap 中对应位置设为 1。查询时同样算 K 个哈希值，检查所有位是否都为 1——有一个是 0 就"一定不存在"。

```
插入 "hello":
  hash1("hello") = 3  → bit[3] = 1
  hash2("hello") = 7  → bit[7] = 1
  hash3("hello") = 11 → bit[11] = 1

查询 "hello":
  bit[3]&bit[7]&bit[11] = 1&1&1 = 1 → "可能存在"

查询 "world":
  bit[5] = 0 → "一定不存在"
```

**为什么是"可能存在"而不是"一定存在"？** 查询 "world" 的三个哈希位可能恰好被 "hello" 和 "foo" 的哈希位置中了——这就是假阳性（false positive）。数学上：位数组大小 m、哈希函数数 k、插入元素数 n，误判率 p ≈ `(1 - e^(-kn/m))^k`。

### 2.2 参数选择

```python
import math

# 给定预期元素数 n 和目标误判率 p，计算最优参数
def bloom_params(n, p):
    m = -n * math.log(p) / (math.log(2) ** 2)  # bit 数组大小
    k = (m / n) * math.log(2)                  # 哈希函数数量
    return int(m), int(k)

# 例：1 亿元素，1% 误判率
# m = -10^8 * ln(0.01) / (ln2)^2 ≈ 9.6 亿 bit ≈ 114MB
# k = 9.6 * 0.693 ≈ 7
```

### 2.3 Java 实现（Guava）

```java
import com.google.common.hash.BloomFilter;
import com.google.common.hash.Funnels;

// 预期 100 万元素，0.01 误判率
BloomFilter<String> filter = BloomFilter.create(
    Funnels.stringFunnel(StandardCharsets.UTF_8),
    1_000_000,
    0.01
);

filter.put("user_12345");
boolean mightContain = filter.mightContain("user_12345"); // true
```

### 2.4 Redis 布隆过滤器

```bash
# RedisBloom 模块
BF.RESERVE user_filter 0.01 1000000   # 误判率 1%，100 万容量
BF.ADD user_filter "user_12345"
BF.EXISTS user_filter "user_12345"    # 1 = 可能存在，0 = 一定不存在
```

### 2.5 实际场景

- **防止缓存穿透**：恶意查询不存在的商品 ID → 先查布隆 → 不存在直接返回，不打 DB
- **爬虫 URL 去重**：已抓取的 URL 放布隆 → 新 URL 命中就跳过
- **HBase Get 快速过滤**：每个 HFile 有 Bloom Filter → 快速跳过不存在该行键的文件
- **黑名单**：用户 ID/设备指纹黑名单快速判断

---

## 三、HyperLogLog——12KB 统计 10 亿 UV

### 3.1 核心原理

**问题**：统计网站今天有多少独立访客（UV）。HashSet 存每个用户 ID → 1 亿用户 = 几 GB 内存。HLL 只需要 **12KB 内存**，误差 0.81%。

**原理**——基于哈希值的前导零：

```
随机哈希值的前导零:
  hash("user_1234") = 0b00001... → 前导有 4 个 0
  hash("user_5678") = 0b0000001... → 前导有 6 个 0

概率直觉:
  → 出现 1 个前导零的概率 ≈ 1/2 → 大约每 2 个元素出现一个
  → 出现 4 个前导零的概率 ≈ 1/16 → 大约每 16 个元素出现一个
  → 看到"最多 6 个前导零" → 大约有 2^6 = 64 个不同元素
```

HLL 将这个直觉做了两个工程优化：分桶（16384 个桶，每个记录该桶的最大前导零数）和调和平均（减少极值的影响）。最终公式：基数 `≈ constant × 16384² / Σ(2^(-M[j]))`。

### 3.2 使用方式

```bash
# Redis
PFADD uv:2026-07-31 "user_1234" "user_5678"
PFCOUNT uv:2026-07-31                     # 返回近似 UV
PFMERGE uv:this_week uv:day1 uv:day2 ...  # 合并多天（去重！）
```

```java
// Java
HyperLogLog hll = new HyperLogLog(14); // 2^14 = 16384 个桶，12KB
hll.add("user_1234");
hll.add("user_5678");
long cardinality = hll.cardinality();  // 近似基数
```

### 3.3 实际场景

- **UV / DAU 实时统计**：12KB 存一天全站 UV，误差 < 1%
- **大数据集的 distinct count 近似**：Spark 的 `approx_count_distinct` 内部就是 HLL
- **不需要精确值的 Count Distinct**：广告曝光人数、商品浏览人数

---

## 四、一致性 Hash

### 4.1 核心原理

传统 `hash(key) % N` 的问题：N 变化（加节点/删节点）→ 几乎所有 key 的映射都会变 → 缓存雪崩。

一致性 Hash 把 0 ~ 2³²-1 映射成一个环：

```
环上分布节点 A、B、C（每个节点在环上有一个或多个位置）
key 映射到环上 → 顺时针找到第一个节点 → 该节点负责这个 key

删除节点 C:
  → 只有 C 负责的 key 重新分配到 D（C 的下一个节点）
  → 其余 key 不受影响
```

**虚拟节点**：物理节点太少时，环上分布不均（有的节点负责一大片 key，有的负责极少）。为每个物理节点创建 100-200 个虚拟节点，打散分布到环上。虚拟节点越多分布越均匀，但路由表越大——100-200 在均匀性和内存之间取得了好的平衡。

### 4.2 Java 实现

```java
public class ConsistentHash {
    private final TreeMap<Integer, String> ring = new TreeMap<>();
    private final int virtualNodes;
    
    public ConsistentHash(int virtualNodes) {
        this.virtualNodes = virtualNodes;
    }
    
    public void addNode(String node) {
        for (int i = 0; i < virtualNodes; i++) {
            int hash = hash(node + "#" + i);
            ring.put(hash, node);
        }
    }
    
    public String getNode(String key) {
        if (ring.isEmpty()) return null;
        int hash = hash(key);
        // 顺时针找到第一个 ≥ hash 的节点
        Map.Entry<Integer, String> entry = ring.ceilingEntry(hash);
        if (entry == null) {
            entry = ring.firstEntry();  // 超过最大值 → 回到环起点
        }
        return entry.getValue();
    }
    
    private int hash(String key) {
        // 可以用 MD5 或 MurmurHash
        return Math.abs(key.hashCode());
    }
}
```

### 4.3 实际场景

- **Redis Cluster**：16384 个 hash slot，CRC16(key) % 16384 → 每个节点负责一部分 slot。本质是特化的一致性 Hash
- **Dubbo 负载均衡**：`consistenthash` 策略——同样的参数总是路由到同一个 Provider
- **CDN 请求路由**：用户请求 → 一致性 Hash 选边缘节点
- **分布式存储**：Cassandra/DynamoDB 的数据分片

---

## 五、雪花算法（Snowflake）——分布式 ID 生成

### 5.1 原理

```
64 位整数:
  [1 位保留] [41 位时间戳] [10 位机器ID] [12 位序列号]
                     │               │            │
                     │               │            └ 同毫秒内递增，最多 4096/ms
                     │               └ 1024 个节点
                     └ 从自定义起始时间（epoch）开始算，约 69 年

容量：每节点每毫秒 4096 个 → 全集群每秒约 400 万个 ID
```

### 5.2 时钟回拨的四种应对

雪花算法的大敌是**时钟回拨**（NTP 校正、虚拟机迁移导致时钟倒退了几秒）。时钟回拨意味着同一个机器 ID + 同一个毫秒时间戳下可能生成重复 ID。

应对方案：

1. **等待**：检测到回拨后 sleep 到追回之前的时间。回拨 1 秒以内适用，不能用于大范围回拨
2. **抛异常**：回拨时拒绝生成 → 上层业务重试 → 适合能容忍短暂不可用的场景
3. **使用历史序列号**：每次生成 ID 时记录 `(timestamp, sequence)`。回拨时使用回拨时间点对应的历史序列号 `+1`
4. **启动时同步时钟**：每次启动时先 NTP 同步，且如果发现本地时间比上次记录的时间低 → 等时间追平后再生成

**美团 Leaf-snowflake 方案**：`workerId` 不再手动指定，而是通过 ZooKeeper 持久顺序节点自动分配——每个 Leaf 节点启动时在 ZK 创建一个顺序节点，节点序号就是 workerId。这就避免了手动配置导致的 workerId 重复。

### 5.3 Java 实现（Hutool）

```java
// Hutool 内置的雪花算法实现
import cn.hutool.core.lang.Snowflake;
import cn.hutool.core.util.IdUtil;

Snowflake snowflake = IdUtil.getSnowflake(workerId=1, datacenterId=1);
long id = snowflake.nextId();  // 1389238400001234567
```

### 5.4 实际场景

- **订单 ID**：分布式系统下替代自增 ID → 各服务独立生成
- **消息 ID**：MQ 消息的全局唯一标识
- **日志 traceId**：全链路追踪的唯一标记

---

## 六、限流算法

### 6.1 四种算法的对比与选择

| 算法 | 原理 | 优点 | 缺点 |
|------|------|------|------|
| 计数器 | 统计 1 秒内请求数 | 实现最简单 | 临界突刺（00:59 和 01:00 各放满流量） |
| 滑动窗口 | 过去 1 秒内请求数（精确到时刻） | 精准，无突刺 | 内存重（存每个请求时间戳） |
| 漏桶 | 请求进桶 → 固定速率流出 | 流量整形（平滑输出） | 不能处理突发流量 |
| 令牌桶 | 固定速率放令牌 → 请求取令牌 | 允许 burst（取完令牌） | 实现稍复杂 |

**生产环境推荐**：网关层用令牌桶（允许短时 burst，体验好），核心数据层用滑动窗口（精确控制，不超 DB 承载）。

### 6.2 Java 实现——Guava RateLimiter（令牌桶）

```java
import com.google.common.util.concurrent.RateLimiter;

// SmoothBursty：允许 burst（存 1 秒的令牌）
RateLimiter limiter = RateLimiter.create(100.0);  // 每秒 100 个令牌

if (limiter.tryAcquire(200, TimeUnit.MILLISECONDS)) {
    // 拿到令牌，处理请求
    doProcess();
} else {
    // 200ms 内拿不到 → 限流
    throw new RateLimitException("too many requests");
}

// SmoothWarmingUp：冷启动，逐渐增加速率
RateLimiter warmupLimiter = RateLimiter.create(100, 3, TimeUnit.SECONDS);
```

### 6.3 Python 实现——令牌桶

```python
import time
from collections import deque

class TokenBucket:
    def __init__(self, rate, capacity):
        self.rate = rate          # 每秒放入的令牌数
        self.capacity = capacity  # 桶容量（允许的最大 burst）
        self.tokens = capacity    # 当前令牌数
        self.last_time = time.monotonic()
    
    def try_acquire(self, tokens=1, timeout=0):
        """尝试获取 tokens 个令牌，timeout 内没拿到返回 False"""
        deadline = time.monotonic() + timeout
        while time.monotonic() < deadline:
            # 补充令牌
            now = time.monotonic()
            elapsed = now - self.last_time
            self.tokens = min(self.capacity, self.tokens + elapsed * self.rate)
            self.last_time = now
            
            if self.tokens >= tokens:
                self.tokens -= tokens
                return True
            
            # 令牌不够 → 等到下一个令牌产生
            wait_time = (tokens - self.tokens) / self.rate
            time.sleep(min(wait_time, 0.01))
        
        return False
```

### 6.4 实际场景

- **API 网关限流**：`/api/user` QPS 限制 → 令牌桶，单个 IP 限制 → 滑动窗口
- **MQ 消费限流**：消费者处理能力有限 → 漏桶，消息匀速消费
- **Sentinel** 的默认限流算法 = 滑动窗口。统计每个时间窗口内的请求数，超阈值拒绝
- **Nginx `limit_req_zone`** = 漏桶（rate 限制平均速率，burst 允许排队等待）

---

## 七、总结

| 算法 | 目标 | 空间代价 | 精确度 |
|------|------|---------|--------|
| BitMap | 去重/存在性 | O(range/8) | 精确 |
| 布隆过滤器 | 存在性判断 | O(n) 极省 | 假阳性 |
| HyperLogLog | Count Distinct | 12KB | 误差 0.81% |
| 一致性 Hash | 数据分片 | O(vNodes) | 精确 |
| 雪花算法 | 分布式 ID | O(1) | 精确 |
| 令牌桶 | 限流 | O(1) | 精确 |

这些算法的共同哲学：**用小的、可控的误差或代价，换来量级级别的资源节省**。
