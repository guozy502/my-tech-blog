---
title: "限流实战——从Guava单机到Redis分布式，框架选型与代码落地"
date: 2026-07-28
description: 从 Guava RateLimiter 的单机令牌桶、Bucket4j 的多维度限流、Sentinel 的集群限流架构，到 Redis 实现限流的五种方案（INCR 计数器/ZSET 滑动窗口/Lua 令牌桶/CL.THROTTLE/Redisson RRateLimiter），结合 API 网关、秒杀、数据库保护三个生产场景的代码落地，覆盖限流从单机到分布式的完整技术选型。
tags: ["分布式","限流","RateLimiter","Redis","Sentinel","Bucket4j","令牌桶"]
categories: ["分布式系统"]
---

# 历史背景——限流为什么从"算法题"变成了"架构题"？

十年前说到限流，答案只有一个：Guava RateLimiter，令牌桶，一行 `limiter.acquire()` 搞定。但今天的微服务架构变化了游戏的规则：

1. **单机限流不够了**：你有 10 个实例，每个实例限制 100 QPS，全局就是 1000 QPS——而数据库只能承受 500。单机限流在这里是盲目的。
2. **限流的粒度变细了**：不是"整个 API 限流 1000 QPS"，而是"每个用户 100 次/秒 + 每个 IP 500 次/秒 + 全 API 总共不超过 5000 QPS"。多维度的限流规则叠加。
3. **框架不只是 Guava**：Sentinel 带来了"限流 + 熔断 + 系统保护"的联动能力；Redis 从单纯的计数器变成了一个限流平台（CL.THROTTLE 命令、Lua 脚本原子执行、Redisson 封装）。

本文的核心观点是：**限流的选型不是"哪个算法更快"，而是"你的瓶颈在哪一层"。** 瓶颈在本地 → Guava 足够；瓶颈在数据库 → 需要全局限流；瓶颈在多维度叠加 → Sentinel 的多规则匹配。

# 一、限流的层次模型——你需要限在哪一层？

```
                    ┌─────────────┐
                    │  API 网关层   │  ← Nginx/Kong/APISIX 的全局限流
                    │  (全局限流)   │     限制某个 API 被调用的总量
                    └──────┬──────┘
                           │
           ┌───────────────┼───────────────┐
      ┌────┴────┐    ┌────┴────┐    ┌────┴────┐
      │ 服务 A   │    │ 服务 B   │    │ 服务 C   │  ← 分布式限流
      │(Sentinel)│    │(Sentinel)│    │(Sentinel)│     集群整体 QPS 上限
      └────┬────┘    └──────────┘    └──────────┘
           │
    ┌──────┴──────┐
    │ 本地方法级限流│  ← Guava RateLimiter
    │ (保护DB连接) │     单机单方法的 QPS 上限
    └─────────────┘
```

**这三层分别对应不同的工具选择**：
- **网关层**：Nginx `limit_req_zone` / Kong rate-limiting 插件 / APISIX `limit-req`，用 Redis 做计数器
- **服务层**：Sentinel / 自研 Redis 限流框架，支持动态规则和全局限流
- **方法层**：Guava RateLimiter / Bucket4j，本地保护特定资源

---

# 二、单机限流——最快的路径，最低的复杂度

## 2.1 Guava RateLimiter——Java 世界的标准选择

```java
// 引入
// implementation 'com.google.guava:guava:33.0-jre'

// ============ SmoothBursty（默认，允许突发）============
RateLimiter limiter = RateLimiter.create(100.0);  // 稳态 100 QPS

// 阻塞式获取令牌
public void handleRequest() {
    double waitSeconds = limiter.acquire();  // 阻塞直到有令牌
    doWork();
}

// 非阻塞式
public boolean tryHandle(long timeout, TimeUnit unit) {
    if (limiter.tryAcquire(timeout, unit)) {
        doWork();
        return true;
    }
    return false;  // 拿不到令牌，触发限流
}

// ============ SmoothWarmingUp（冷启动，禁止突发）============
RateLimiter warmupLimiter = RateLimiter.create(
    100.0,   // 稳态速率
    3,       // 3 秒预热期
    TimeUnit.SECONDS
);
// 适用：刚启动的系统需要预热（连接池建立、缓存预热等）
// 前 3 秒令牌生成速率从 0 线性增长到 100/s

// ============ 实际案例：保护数据库连接池 ============
class DatabaseGuard {
    // 不管外面多大 QPS，放给数据库的不超过 500 QPS
    private final RateLimiter dbLimiter = RateLimiter.create(500);
    
    public List<Order> queryOrders(Long userId) {
        if (!dbLimiter.tryAcquire(100, TimeUnit.MILLISECONDS)) {
            throw new RateLimitException("数据库查询限流");
        }
        return orderDao.findByUserId(userId);
    }
}
```

**Guava RateLimiter 的局限性**：
1. **纯单机**——10 个实例各有自己的 RateLimiter，互相不知道对方用了多少配额
2. **不支持动态调整**——rate 在构造时固定，不支持运行时修改（需要重建对象）
3. **只支持 QPS 维度**——不能按"用户/IP/API"多维度叠加限流

## 2.2 Bucket4j——多维度 + 动态规则的 Java 限流库

[Bucket4j](https://github.com/vladimir-bukhtoyarov/bucket4j) 比 Guava 更灵活：支持**多维度**限流、**运行时动态调整**规则、**分布式扩展**（基于 Redis/Hazelcast）。

```java
// implementation 'com.github.vladimir-bukhtoyarov:bucket4j-core:8.7.0'

// ============ 基础令牌桶 ============
Bandwidth limit = Bandwidth.classic(100, Refill.greedy(100, Duration.ofSeconds(1)));
Bucket bucket = Bucket.builder().addLimit(limit).build();

if (bucket.tryConsume(1)) {
    doWork();
} else {
    reject();
}

// ============ 多维度限流：同一个 API 对不同用户有不同的配额 ============
class MultiDimensionalRateLimiter {
    // 每个用户一个独立的 bucket
    private final ConcurrentHashMap<String, Bucket> userBuckets = new ConcurrentHashMap<>();
    
    private Bucket createUserBucket(String userId) {
        // VIP 用户 1000 QPS，普通用户 100 QPS
        long qps = isVip(userId) ? 1000 : 100;
        Bandwidth limit = Bandwidth.classic(qps, Refill.greedy(qps, Duration.ofSeconds(1)));
        return Bucket.builder().addLimit(limit).build();
    }
    
    public boolean tryConsume(String userId) {
        return userBuckets.computeIfAbsent(userId, this::createUserBucket).tryConsume(1);
    }
}

// ============ 分布式扩展（基于 Redis）============
// 引入 bucket4j-redis
// Bucket bucket = BucketProxyManager 创建的分布式 bucket
// bucket.tryConsume(1) 内部走 Redis 做全局限流
```

**Bucket4j 的优势**：
- 支持分钟/小时/天级别的长时间窗口限流（不只是秒级 QPS）
- 支持 `Refill.intervally()`（固定间隔补充令牌）和 `Refill.greedy()`（平滑补充）
- 分布式模式下自动通过 Redis/JCache 同步令牌状态

## 2.3 Guava vs Bucket4j vs 手写——选型建议

| | Guava RateLimiter | Bucket4j | 自实现（ScheduledThreadPool） |
|------|-----------------|---------|-----------------------------|
| **难度** | ⭐ | ⭐⭐ | ⭐⭐⭐ |
| **动态调整速率** | ❌ | ✅ | ✅ |
| **多维度** | ❌ | ✅ 多 bucket | ✅ |
| **分布式扩展** | ❌ | ✅ 通过 Redis | ❌（需另外设计） |
| **适合** | 简单场景 | 灵活/复杂场景 | 不想引入依赖 |

---

# 三、分布式限流——集群视角下的全局限流

## 3.1 为什么单机限流不够？

```
10 个服务实例，每个 Guava RateLimiter(100/s)
→ 全局可以通过 1000 QPS

但你的数据库只能承受 500 QPS
→ 单机限流完全不解决问题
→ 需要"所有实例加起来不超过 500 QPS"的全局视野
```

分布式限流的本质是**把"配额"从本地内存搬到全局共享存储（Redis）中**——所有实例共同争用同一份配额。

## 3.2 Sentinel——阿里开源的"限流+熔断+系统保护"全家桶

[Sentinel](https://github.com/alibaba/Sentinel) 是目前国内使用最广泛的分布式限流框架。它的核心价值不止限流——**限流、熔断降级、系统保护是联动的**。QPS 超了先限流，RT 高了触熔断，系统负载升了启自适应降级。

```xml
<!-- pom.xml -->
<dependency>
    <groupId>com.alibaba.csp</groupId>
    <artifactId>sentinel-core</artifactId>
    <version>1.8.7</version>
</dependency>
```

```java
// ============ 基础用法：直接埋点 ============
// ① 定义规则（通常在启动时加载）
private void initRules() {
    List<FlowRule> rules = new ArrayList<>();
    
    // 规则 1：getOrder API 限流 100 QPS
    FlowRule orderRule = new FlowRule("getOrder");
    orderRule.setGrade(RuleConstant.FLOW_GRADE_QPS);  // QPS 模式
    orderRule.setCount(100);                           // 100 QPS
    rules.add(orderRule);
    
    // 规则 2：createOrder API 限流 50 QPS（基于调用方）
    FlowRule createRule = new FlowRule("createOrder");
    createRule.setGrade(RuleConstant.FLOW_GRADE_QPS);
    createRule.setCount(50);
    createRule.setLimitApp("app-A,app-B");  // 只针对这两个调用方
    rules.add(createRule);
    
    FlowRuleManager.loadRules(rules);
}

// ② 被保护的业务代码
@GetMapping("/order/{id}")
public Order getOrder(@PathVariable Long id) {
    // 埋点：entry 通过 = 没被限流
    try (Entry entry = SphU.entry("getOrder")) {
        return orderService.findById(id);
    } catch (BlockException e) {
        // 被限流 → 走降级返回
        return Order.fallback();
    }
}
```

```java
// ============ 注解方式（推荐）============
@RestController
public class OrderController {
    
    @GetMapping("/order/{id}")
    @SentinelResource(value = "getOrder", 
                      blockHandler = "getOrderBlocked")
    public Order getOrder(@PathVariable Long id) {
        return orderService.findById(id);
    }
    
    // 被限流后的降级方法
    public Order getOrderBlocked(Long id, BlockException e) {
        return Order.fallback();
    }
}
```

**Sentinel 的集群限流**是其最重要的分布式能力：

```
┌──────────────────────────────────────┐
│         Sentinel Dashboard           │
│     (规则管理 + 实时监控界面)           │
└────┬──────────┬──────────┬───────────┘
     │          │          │
┌────┴──┐  ┌────┴──┐  ┌────┴──┐
│实例 A  │  │实例 B  │  │ 实例 C │
│Sentinel│  │Sentinel│  │Sentinel│
│ + 规则  │  │ + 规则  │  │ + 规则  │
└───────┘  └───────┘  └───────┘

集群限流模式：
  ① Token Server 模式：集群选一个实例作为"令牌服务器"
     其余实例申请令牌都通过它，全局只有一份配额
  ② 嵌入式模式：不选单独的 Token Server，各实例之间直接通信协商
```

```java
// 集群限流规则
FlowRule clusterRule = new FlowRule("api-total");
clusterRule.setCount(500);           // 整个集群总共 500 QPS
clusterRule.setClusterMode(true);    // ← 集群模式
clusterRule.setClusterConfig(new ClusterFlowConfig()
    .setFlowId(1001L)
    .setThresholdType(ClusterRuleConstant.FLOW_THRESHOLD_GLOBAL) // 全局阈值
);
```

**Sentinel vs Guava 的关键差异**：

| | Guava RateLimiter | Sentinel |
|------|-----------------|----------|
| **限流范围** | 单 JVM | 集群（可单机可全局） |
| **规则管理** | 硬编码 | Dashboard 可视化 + 动态下发 |
| **熔断降级** | ❌ | ✅ （慢调用/异常比例/异常数） |
| **系统保护** | ❌ | ✅ （CPU/LOAD/RT 自适应） |
| **热点参数限流** | ❌ | ✅ （如"userId=123 的特权用户"降低 QPS） |
| **运维复杂度** | 零 | 中（需部署 Dashboard） |

---

# 四、Redis 实现限流的五种方案

Redis 是分布式限流最常用的共享存储。从最简单的 `INCR` 计数器到内置的 `CL.THROTTLE`，有五种不同精度和代价的实现方案。

## 4.1 方案一：INCR 固定窗口计数器（最简单，不精确）

```java
/**
 * Redis INCR 实现固定窗口计数器
 * 优点：1 次 Redis 操作，性能最高
 * 缺点：临界双倍问题（见 限流算法全解 的详细分析）
 */
public class FixedWindowRateLimiter {
    private final StringRedisTemplate redis;
    
    public boolean isAllowed(String key, int limit, int windowSeconds) {
        String redisKey = "rate_limit:" + key + ":" + 
                          LocalDateTime.now().truncatedTo(ChronoUnit.SECONDS);
        
        Long count = redis.opsForValue().increment(redisKey);
        
        if (count == 1) {
            redis.expire(redisKey, Duration.ofSeconds(windowSeconds * 2));  // 给 2x 兜底
        }
        
        return count <= limit;
    }
}
// 调用: isAllowed("api:order:" + userId, 100, 1)
// 含义: 每个用户每秒最多 100 次
```

**适用**：粗糙的计数场景（如统计埋点、非精确的防刷），**生产环境不建议用来保护数据库**。

## 4.2 方案二：ZSET 滑动窗口（精确但操作多）

```java
/**
 * Redis ZSET 实现滑动窗口
 * 优点：精确到毫秒，窗口平滑滑动
 * 缺点：3 次 Redis 操作，ZSET 内存占用随请求量线性增长
 */
public class SlidingWindowRateLimiter {
    private final StringRedisTemplate redis;
    
    public boolean isAllowed(String key, int limit, int windowSeconds) {
        long now = System.currentTimeMillis();
        long windowStart = now - windowSeconds * 1000L;
        
        // ① 清除窗口外的旧请求
        redis.opsForZSet().removeRangeByScore(key, 0, windowStart);
        
        // ② 统计窗口内的请求数
        Long count = redis.opsForZSet().zCard(key);
        
        if (count != null && count < limit) {
            // ③ 记录本次请求（用纳秒+随机数确保唯一性）
            redis.opsForZSet().add(key, String.valueOf(now + ThreadLocalRandom.current().nextInt(1000)), now);
            redis.expire(key, Duration.ofSeconds(windowSeconds));
            return true;
        }
        return false;
    }
}
```

**适用**：需要精确计数的 API 限流（如"1 分钟内最多 10 次"的短信验证码接口）。

**痛点**：高 QPS 下每个请求 3 次 Redis 操作，ZSET 的 score 清理在大窗口下可能成为瓶颈。生产建议用 Lua 脚本把三步原子化（见方案三）。

## 4.3 方案三：Lua 令牌桶——原子 + 精确 + 可定制

```lua
-- rate_limiter.lua：令牌桶算法在 Redis 中的原子实现
-- KEYS[1]: 令牌桶的 key
-- ARGV[1]: 令牌投放速率（个/秒）
-- ARGV[2]: 桶容量上限（最大突发量）
-- ARGV[3]: 当前时间（毫秒）
-- ARGV[4]: 请求的令牌数

local key = KEYS[1]
local rate = tonumber(ARGV[1])       -- 每秒生成多少个令牌
local capacity = tonumber(ARGV[2])   -- 桶最多存多少令牌
local now = tonumber(ARGV[3])        -- 当前时间（ms）
local requested = tonumber(ARGV[4])  -- 要拿几个令牌

-- 取出上一次请求的时间和桶里的令牌数
-- 数据结构：hash { "tokens": 100, "last_refreshed": 1234567890 }
local bucket = redis.call("HMGET", key, "tokens", "last_refreshed")
local tokens = tonumber(bucket[1])
local last_refreshed = tonumber(bucket[2])

if tokens == nil then
    -- 第一次请求，桶满
    tokens = capacity
    last_refreshed = now
end

-- 计算从上次补充到现在生成了多少新令牌
local elapsed_ms = now - last_refreshed
local new_tokens = elapsed_ms * rate / 1000  -- 新增令牌数
tokens = math.min(capacity, tokens + new_tokens)  -- 不能超过桶容量
last_refreshed = now

-- 判断够不够
local allowed = tokens >= requested
if allowed then
    tokens = tokens - requested
end

-- 更新 Redis
redis.call("HMSET", key, "tokens", tokens, "last_refreshed", last_refreshed)
redis.call("EXPIRE", key, math.ceil(capacity / rate) * 2)  -- 自动过期

return {allowed and 1 or 0, tokens}
```

```java
// Java 调用侧
public class LuaTokenBucketRateLimiter {
    private final StringRedisTemplate redis;
    private final DefaultRedisScript<List> script;
    
    public LuaTokenBucketRateLimiter(StringRedisTemplate redis) {
        this.redis = redis;
        this.script = new DefaultRedisScript<>();
        script.setLocation(new ClassPathResource("rate_limiter.lua"));
        script.setResultType(List.class);
    }
    
    public boolean isAllowed(String key, int rate, int capacity) {
        List<?> result = redis.execute(
            script,
            List.of("token_bucket:" + key),
            String.valueOf(rate),
            String.valueOf(capacity),
            String.valueOf(System.currentTimeMillis()),
            "1"  // 每次请求 1 个令牌
        );
        return result != null && Long.parseLong(result.get(0).toString()) == 1L;
    }
}

// 使用示例：对 API 限流 100 QPS，允许突发 200
limiter.isAllowed("api:/order/create", 100, 200);
```

**这个 Lua 脚本的优势**：
- **原子性**：Redis 单线程执行整个脚本，不会出现"查了但还没扣"的竞态
- **空间效率**：一个 hash key 存 {tokens, last_refreshed}，比 ZSET 的内存占用小两个数量级
- **可定制**：rate、capacity、requested tokens 都是参数，一个脚本适用所有场景

## 4.4 方案四：Redis Stack CL.THROTTLE——一行命令搞定

Redis Stack（Redis 7.2+ 或 Redis Stack Docker 镜像）内置了限流命令 `CL.THROTTLE`，基于 **Generic Cell Rate Algorithm（GCRA）**——滑动窗口的变体。

```bash
# CL.THROTTLE <key> <max_burst> <tokens_per_period> <period> <cost_tokens>
# max_burst: 最大突发量（桶容量）
# tokens_per_period: 每个 period 生成的令牌数
# period: 时间窗口（秒）
# cost_tokens: 本次请求消耗的令牌数

# 例子：限制 60 次/分钟，最大突发 100 次
redis-cli CL.THROTTLE api:order:user123 100 60 60 1
# 返回：
# 1) (integer) 0        ← 0=允许通过, 1=被限流
# 2) (integer) 101      ← 最大突发量（你的 max_burst）
# 3) (integer) 99       ← 桶里还剩多少令牌
# 4) (integer) -1       ← 被限流时需要等多少秒才能重试（-1=没被限流）
# 5) (integer) 2        ← 桶恢复到满需要多少秒
```

```java
// Java 调用（Lettuce 客户端支持自定义命令）
public class RedisCellRateLimiter {
    private final RedisCommands<String, String> sync;
    
    public boolean isAllowed(String key, long maxBurst, long rate, long periodSeconds) {
        // CL.THROTTLE key max_burst rate period 1
        List<Object> result = sync.dispatch(
            CommandType.valueOf("CL.THROTTLE"),
            new StatusOutput<>() {},
            new CommandArgs<>(codec)
                .addKey(key)
                .add(maxBurst)
                .add(rate)
                .add(periodSeconds)
                .add(1)  // cost_tokens
        );
        return result != null && (Long) result.get(0) == 0L;
    }
}
```

**CL.THROTTLE 对比手写 Lua 的优势**：
- 单命令，无需加载脚本
- 返回信息丰富（包括重试时间和恢复到满的时间）
- 使用 GCRA 算法（滑动窗口思想的优化实现），比 ZSET 滑动窗口更省内存

## 4.5 方案五：Redisson RRateLimiter——Java 友好的分布式令牌桶

[Redisson](https://github.com/redisson/redisson) 把 `CL.THROTTLE` 封装成了 Java 对象，且**支持异步刷新令牌**（不需要每次调用都走 Redis）。

```java
// 引入 redisson
// implementation 'org.redisson:redisson-spring-boot-starter:3.30.0'

@Autowired
private RedissonClient redisson;

public void initRateLimiter() {
    // 创建限流器：每秒 100 个令牌
    RRateLimiter limiter = redisson.getRateLimiter("api:order");
    
    // trySetRate(RateType, rate, rateInterval, RateIntervalUnit)
    // RateType.OVERALL: 所有客户端共享的总速率（全局限流）
    // RateType.PER_CLIENT: 每个客户端独立的速率
    limiter.trySetRate(RateType.OVERALL, 100, 1, RateIntervalUnit.SECONDS);
    
    // 实际使用
    limiter.acquire();  // 阻塞获取
    // 或
    if (limiter.tryAcquire()) {  // 非阻塞
        doWork();
    }
}

// 动态调整速率（无须重建）
public void adjustRate(long newRate) {
    RRateLimiter limiter = redisson.getRateLimiter("api:order");
    limiter.setRate(RateType.OVERALL, newRate, 1, RateIntervalUnit.SECONDS);
}
```

**Redisson RRateLimiter 的内部实现**（关键点）：
1. 底层使用 `CL.THROTTLE`（Redis Stack）或回退到 Lua 脚本
2. 支持 `trySetRate` + 运行时 `setRate` 修改（比 Guava 灵活）
3. 支持 FAIL_FAST（拿不到立刻返回）和 BLOCKING（阻塞等待）两种模式

---

# 五、经典生产场景与技术选型

## 场景 1：API 网关——全局限流 + 多维度叠加

```
需求：每个用户 100 次/秒，每个 IP 500 次/秒，全 API 总共 5000 QPS
方案：Redis ZSET 滑动窗口（精确）+ INCR 计数器（粗糙）

优先用 ZSET 对"用户维度"精确限流，IP 维度用 INCR 兜底。
总 QPS 限制用 Sentinel 集群限流或 CL.THROTTLE。
```

```java
// 多级限流组合
public class ApiGatewayRateLimiter {
    private final SlidingWindowRateLimiter userLimiter;   // ZSET 精确限流
    private final FixedWindowRateLimiter ipLimiter;       // INCR 粗略防刷
    private final RRateLimiter globalLimiter;             // Redisson 全局总控
    
    public RateLimitResult check(String userId, String ip) {
        // 第一层：总 QPS（5000/s 全 API）
        if (!globalLimiter.tryAcquire()) {
            return RateLimitResult.reject("全局限流");
        }
        // 第二层：IP 维度（500/s）
        if (!ipLimiter.isAllowed("ip:" + ip, 500, 1)) {
            return RateLimitResult.reject("IP 限流");
        }
        // 第三层：用户维度（100/s）
        if (!userLimiter.isAllowed("user:" + userId, 100, 1)) {
            return RateLimitResult.reject("用户限流");
        }
        return RateLimitResult.allow();
    }
}
```

## 场景 2：秒杀/抢购——Sentinel 集群限流 + 热点参数

```
需求："每个用户只能抢到 1 次" + "总库存 10000 件"
方案：Sentinel 集群限流 + 热点参数限流

Sentinel 的热点参数限流可以识别 "skuId=hot-item" 的值是高热点的，
针对这个特殊值降低 QPS 限制，防止某个热门商品抢垮系统。
```

```java
// Sentinel 热点参数限流规则
ParamFlowRule rule = new ParamFlowRule("seckill")
    .setParamIdx(0)              // 第 0 个参数（skuId）是限流依据
    .setGrade(RuleConstant.FLOW_GRADE_QPS)
    .setCount(1000);             // 默认 1000 QPS

// 针对特定热点值降低限制
ParamFlowItem hotItem = new ParamFlowItem()
    .setObject("hot-sku-123")    // 最火爆的商品
    .setClassType(String.class.getName())
    .setCount(100);              // 只给 100 QPS（保护系统）
rule.addParamFlowItem(hotItem);
ParamFlowRuleManager.loadRules(List.of(rule));

// 业务代码
@SentinelResource(value = "seckill", blockHandler = "seckillBlocked")
public OrderResult seckill(@RequestParam String skuId, @RequestParam Long userId) {
    // Sentinel 根据 skuId 的值自动匹配限流规则
    return seckillService.execute(skuId, userId);
}
```

## 场景 3：数据库连接保护——Guava 单机 + 少量 Redis 兜底

```
需求：数据库最多承受 500 QPS（全局 10 个实例）
方案：每个实例 Guava RateLimiter(50/s) + Redis Lua 令牌桶(500/s) 兜底

Guava 作为第一道防线（零网络开销），Redis Lua 令牌桶作为第二道防线
（纠正各实例 Guava 独立导致的全局偏差）。
```

```java
public class DatabaseGuard {
    // 第一道防线：本地令牌桶（零延迟）
    private final RateLimiter localLimiter = RateLimiter.create(50);  // 每实例 50/s
    
    // 第二道防线：Redis 全局令牌桶（兜底）
    private final LuaTokenBucketRateLimiter globalLimiter;
    
    public boolean tryExecute(Runnable task) {
        // ① 本地快速通道
        if (!localLimiter.tryAcquire(10, TimeUnit.MILLISECONDS)) {
            return false;
        }
        // ② 全局二次检查（异步？同步？取决于你对精度的要求）
        // 同步模式：每次都查 Redis → 精度高但多一次 RTT
        // 异步模式：本地 batch 处理 → 如每 100ms 从 Redis 批量预取令牌
        if (!globalLimiter.isAllowed("db-guard:" + instanceId, 500, 500)) {
            return false;  // 全局配额用完了
        }
        task.run();
        return true;
    }
}
```

---

# 六、五种 Redis 限流方案对比

| 方案 | Redis 操作次数 | 精度 | 内存占用 | 突发支持 | 生产建议 |
|------|--------------|------|---------|---------|---------|
| **INCR 计数器** | 1 次 | ❌ 临界双倍 | 极低 | ❌ | 粗糙防刷、埋点统计 |
| **ZSET 滑动窗口** | 3 次 | ✅ 精确 | 高（与 QPS 成正比） | ✅ | 精确 API 限流、短信接口 |
| **Lua 令牌桶** | 1 次（整个脚本） | ✅ 令牌桶精度 | 极低（一个 hash） | ✅ 可配置 | **生产首选**的通用方案 |
| **CL.THROTTLE** | 1 次 | ✅ GCRA 精度 | 极低 | ✅ | Redis Stack 环境首选 |
| **Redisson RRateLimiter** | 1 次 | ✅ | 极低 | ✅ | Java 生态最友好 |

---

# 七、选型决策树

```
你的 Redis 是 Redis Stack 或 7.2+?
  ├── 是 → CL.THROTTLE（一行命令，零维护）
  └── 否 → 你用的是 Java?
            ├── 是 → Redisson RRateLimiter（封装好，开箱即用）
            └── 否或其他语言 → Lua 令牌桶脚本（通用方案）

你需要集群限流 + 熔断 + 系统保护联动?
  ├── 是 → Sentinel（不止限流，是整个流量治理体系）
  └── 否 → 继续往下

你只需要单机限流?
  ├── 是 → Guava RateLimiter（最简单，最快）
  └── 否 → 你的限流有多维度需求?（用户/IP/API 叠加）
            ├── 是 → Bucket4j 多 bucket 模式 或 Lua 令牌桶 + 多 key
            └── 否 → Redis Lua 令牌桶（一次搞定）
```

---

# 八、总结

| 维度 | 单机 | 分布式 | 两者结合（推荐） |
|------|------|--------|----------------|
| **工具** | Guava / Bucket4j | Sentinel / Redis Lua / Redisson | Guava(本地) + Redis(全局) |
| **延迟** | ~10ns | ~0.1-0.5ms（Redis RTT） | 本地 ~10ns，全局兜底 |
| **精度** | 精确 | 近似（网络延迟导致） | 本地精确 + 全局校准 |
| **适用** | 保护本地资源 | 集群统一配额 | **生产通用方案** |

> **限流的选型原则：保护什么资源，就在离它最近的地方限。** 保护数据库 → 在 DAO 层用 Guava；保护 API → 在网关用 Redis；保护整个集群 → 在服务入口用 Sentinel。不是为了炫技而选复杂的方案，而是每一层限流都保护了不同的资源。

# 延伸阅读

**Do——动手验证：**
- 用 JMeter 对 Guava RateLimiter 限流的 API 发起 200 QPS 持续 30 秒，用 Grafana 观察通过率
- 在本地 Docker 中启动 Redis Stack（`docker run -p 6379:6379 redis/redis-stack-server`），试跑 CL.THROTTLE 命令
- 用 Sentinel Dashboard（Docker 一键部署）给一个 Spring Boot 应用配置集群限流规则

**Todo——深入方向：**
- Nginx `limit_req_zone` + `limit_req` 的漏桶实现与 burst/nodelay 参数组合
- Kong/APISIX 的限流插件与 Redis Cluster 的集成
- 自适应限流（Adaptive Limiting）——根据系统实时负载（CPU/RT）动态调整限流阈值

*本文参考资料：*
- Guava RateLimiter 源码: `com.google.common.util.concurrent.RateLimiter` (SmoothRateLimiter)
- Bucket4j 官方文档: https://bucket4j.com/
- Sentinel 官方文档: https://sentinelguard.io/
- Redis CL.THROTTLE 文档: https://redis.io/docs/latest/commands/cl.throttle/
- Redisson RRateLimiter 文档: https://redisson.org/docs/
