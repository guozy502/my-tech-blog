---
title: "Redis 分布式锁实现原理——从SETNX到RedLock的完整推导"
date: 2026-07-31
description: 从SETNX死锁到RedLock五阶段演进推导，逐层讲清原子性陷阱、Watch Dog看门狗续期机制、Lua脚本可重入实现、主从切换丢锁与RedLock争议。
tags: ["Redis","分布式锁","Redisson","RedLock","Watch Dog","Lua","可重入锁"]
categories: ["中间件"]
---

# Redis 分布式锁实现原理——从 SETNX 到 RedLock 的完整推导

> 面试问你 Redis 分布式锁，答"SETNX + 过期时间 + Lua 删锁"只能算及格。真正考察的是你能不能讲清楚：为什么不能先 SETNX 再 EXPIRE？看门狗怎么续期？Redis 主从切换时锁为什么丢了？RedLock 到底有没有用？本文从零推导 Redis 分布式锁的五次演进，逐层讲清每一步解决了什么问题、又引入了什么新问题。

---

## 一、为什么需要分布式锁

单机 `synchronized` 或 `ReentrantLock` 只在同一个 JVM 内有效。当你部署了 3 个 pod 同时处理扣库存的请求：

```
Pod A: 库存 10 → 读 → 减 1 → 写 9
Pod B: 库存 10 → 读 → 减 1 → 写 9   ← 跟 A 同时读到 10
→ 库存应该是 8，实际是 9。超卖了一条。
```

你需要一把**所有 JVM 共享的锁**——这就是分布式锁。Redis 因为单线程、高性能、原子命令，天然适合实现。

---

## 二、第一次演进：SETNX 单条命令

最简单的实现：

```bash
SETNX lock_key unique_value
```

`SETNX` = **SET** if **N**ot e**X**ists。返回 1 表示拿到锁，返回 0 表示锁已被别人持有。unique_value 是解锁时的"钥匙"——只有锁的持有者才能解锁。最简单的 value 是 UUID，复杂场景可以用"线程 ID + 递增计数器"来支持同一线程内可重入。

**问题一目了然**：进程拿到锁之后崩了——锁永远不释放——死锁。

---

## 三、第二次演进：SETNX + EXPIRE（两条命令的原子性陷阱）

给锁加一个自动过期时间，防止死锁：

```bash
SETNX lock_key unique_value
EXPIRE lock_key 30
```

**致命缺陷**：SETNX 和 EXPIRE 是两条命令。如果在 SETNX 成功之后、EXPIRE 执行之前，进程崩了——`EXPIRE` 没执行 → 锁没有过期时间 → 死锁。

**不是说不会发生**：进程在 SETNX 和 EXPIRE 之间 OOM、被 kill -9、或者宿主机掉电——中间那几十微秒就是死锁窗口。高频锁竞争场景下，这个概率乘以调用次数就不小了。

---

## 四、第三次演进：SET ... NX PX（原子化）

Redis 2.6.12 开始，`SET` 命令支持扩展参数——将"加锁 + 设过期"合并为一条原子命令：

```bash
SET lock_key unique_value NX PX 30000
```

| 参数 | 含义 |
|------|------|
| `NX` | Not eXists——只有当 key 不存在时才设置 |
| `PX 30000` | 过期时间 30000 毫秒 |

**一条命令、一次网络往返、原子执行**。这是 Redis 分布式锁的基础实现。

**但过期时间设多少？**

- 设得太短：业务还没执行完，锁过期了。其他线程拿到锁 → 并发安全问题
- 设得太长：持有锁的进程崩了，锁要等很久才释放 → 服务不可用

**这个矛盾催生了"看门狗"**。

---

## 五、第四次演进：锁续期（Watch Dog 看门狗）

### 5.1 核心思想

不是"设一个足够大的过期时间"，而是**在业务执行期间持续续期**。业务完成后主动释放 + 停止续期。

Redisson 的看门狗机制是标准实现。它的核心思路是：加锁时不指定过期时间，由后台定时任务定期刷新 TTL。锁的"持有权"和"有效期"被解耦——只要业务还在跑，锁就一直有效；业务结束或进程挂了，锁的续期停止，TTL 到期自动释放。

### 5.2 Redisson 加锁流程

Redisson 的加锁命令不是简单的 `SET lock_key uuid NX PX 30000`，而是用 **Hash 结构 + Lua 脚本**。用 Hash 是为了支持**可重入**——Hash 的 key 是锁名，field 是"线程标识"，value 是重入次数。

```lua
-- Redisson 加锁 Lua 脚本（简化版）
-- KEYS[1] = lock_key
-- ARGV[1] = 锁的过期时间 (默认 30000ms)
-- ARGV[2] = 锁持有者标识 (UUID:threadId)

-- 1. 如果锁不存在 → 加锁成功
if (redis.call('exists', KEYS[1]) == 0) then
    redis.call('hincrby', KEYS[1], ARGV[2], 1);   -- 重入次数=1
    redis.call('pexpire', KEYS[1], ARGV[1]);        -- 设过期时间
    return nil;   -- 加锁成功
end;

-- 2. 如果锁存在，且持有者是自己 → 可重入
if (redis.call('hexists', KEYS[1], ARGV[2]) == 1) then
    redis.call('hincrby', KEYS[1], ARGV[2], 1);   -- 重入次数+1
    redis.call('pexpire', KEYS[1], ARGV[1]);        -- 刷新过期时间
    return nil;   -- 重入成功
end;

-- 3. 锁被其他人持有 → 返回剩余过期时间（调用方据此自旋等待）
return redis.call('pttl', KEYS[1]);
```

**为什么用 Hash 而不是简单的 String？** String 只能是"有锁/没锁"二态。但分布式锁需要可重入——同一线程多次获取同一把锁不能把自己阻塞。Hash 结构：field 存线程 ID、value 存重入计数——自然支持可重入，且字段间操作在 Lua 脚本中原子化。

**返回值详解**：返回 `nil` = 加锁成功；返回 `pttl` (剩余过期时间的毫秒数) = 锁被别人持有，调用方可以根据这个值设置等待超时，而不是无限自旋。

### 5.3 看门狗续期机制

```java
// Redisson 看门狗核心逻辑（简化版）
// 这不是 Redisson 源码，是核心逻辑的提炼

private void scheduleWatchDog(String lockKey, String threadId) {
    // 定时任务：每 internalLockLeaseTime/3 = 10 秒执行一次
    ScheduledFuture<?> future = executor.scheduleAtFixedRate(() -> {
        // 续期 Lua 脚本：只有"锁还存在且持有者是我"时才续期
        String lua = """
            if (redis.call('hexists', KEYS[1], ARGV[1]) == 1) then
                redis.call('pexpire', KEYS[1], ARGV[2]);
                return 1;
            end;
            return 0;
            """;
        
        Long result = eval(lua, lockKey, threadId, "30000");
        if (result == 0) {
            // 锁已经不属于我了 → 停止续期
            future.cancel(false);
        }
    }, internalLockLeaseTime / 3, internalLockLeaseTime / 3, TimeUnit.MILLISECONDS);
}

// 解锁时停止看门狗
private void unlock(String lockKey, String threadId) {
    // 先取消续期任务（必须在删锁之前，否则 key 已于续期中被刷新）
    cancelWatchDog(lockKey);
    
    // 再删除锁
    eval(unlockLua, lockKey, threadId);
}
```

**关键细节**：加锁 leaseTime = -1（默认）时，锁默认 30 秒过期，看门狗每 10 秒续期到 30 秒。如果 leaseTime > 0（你自己指定了过期时间），看门狗不启动——因为"自定义过期时间"意味着你不需要续期，锁到期自然释放。

### 5.4 解锁——为什么必须用 Lua 脚本

```lua
-- Redisson 解锁 Lua 脚本（简化版）
-- 必须是原子操作：检查持有者 + 递减计数 + 删除 key
if (redis.call('hexists', KEYS[1], ARGV[1]) == 0) then
    return nil;   -- 锁不是我的，什么都不做
end;

local counter = redis.call('hincrby', KEYS[1], ARGV[1], -1);
if (counter > 0) then
    -- 重入计数 > 0 → 还在重入层数中，不删锁
    redis.call('pexpire', KEYS[1], ARGV[2]);
    return 0;
else
    -- 重入计数 = 0 → 完全释放锁
    redis.call('del', KEYS[1]);
    return 1;
end;
```

**为什么必须 Lua**：如果不用 Lua，你需要两步——`GET lock_key` 检查是否是自己的锁，然后 `DEL lock_key`。两步之间有并发窗口：A 检查锁是自己的 → B 的锁刚好过期 → B 拿到锁 → A 的 DEL 删了 B 的锁。Lua 原子化后这个窗口不存在。

**删除锁之后 redis 才收到看门狗续期请求怎么办？** 这个顺序是无法保证的——网络延迟可以让"先发的删除"在"后发的续期"之后到达 Redis。如果发生这种情况，续期请求会创建一个新的锁——这显然不对。所以续期脚本必须加 `hexists` 判断（如果锁已经不存在或者持有者已经变了，不执行续期）。

---

## 六、第五次演进：RedLock —— 解决单点 Redis 的数据安全问题

### 6.1 主从切换时锁为什么丢了

前面所有的方案，数据只存在一个 Redis 节点上（即使有主从也只有一个主节点接受写入）：

```
1. Client A 在 Master 拿到锁
2. Master 还没来得及将锁同步给 Slave → Master 宕机
3. Slave 提升为新 Master
4. Client B 在新 Master 拿到同一把锁（因为 Slave 上没有锁的数据）
5. Client A 和 Client B 同时持有同一把锁 → 并发安全问题
```

**这个场景必须用异步复制来触发**——因为 Redis 的复制是异步的（同步复制会严重拉低写入性能）。锁的写入在 Master 确认后立刻返回，此时锁还没到 Slave 上。异步复制是 Redis 高性能的基础，但也是锁在主从切换时丢失的根源。

### 6.2 RedLock 算法——Redis 作者提出的方案

RedLock 的核心思路：**不在一个 Redis 实例上加锁，而是向 N 个独立 Redis 实例（奇数个，不是主从关系，不是 Cluster Shard，是各自独立的单机）分别获取锁**。过半节点成功才算成功。

```
Client 向 5 个独立 Redis 实例发起加锁请求
  → Redis 1: ✓ 拿到锁
  → Redis 2: ✓ 拿到锁
  → Redis 3: ✓ 拿到锁
  → Redis 4: ✗ 超时
  → Redis 5: ✗ 超时
  
  成功 3/5 > 半数 → 加锁成功
  锁的有效时间 = 原始过期时间 - 获取锁的总耗时
```

**RedLock 的核心保障**：任何时刻，同一把锁只能被一个 Client 持有。因为任意两个 Client 如果要同时获得锁，它们各自需要拿到过半节点的锁——但两个"过半集合"必然有交集（5 个节点中两个拿到 3 个的集合至少共享一个节点），而这个交集节点不可能同时给两个人加锁。

### 6.3 加锁与解锁流程

```
加锁:
  1. 获取当前时间 t1
  2. 依次向 N 个 Redis 实例请求加锁（SET NX PX）
     - 每个请求有独立超时（通常远小于锁的过期时间，比如 50ms）
     - 某个实例超时 → 跳过，继续下一个
  3. 获取当前时间 t2
  4. 计算总耗时 = t2 - t1
  5. 成功获取的实例数 >= N/2+1 且总耗时 < 锁的有效时间
     → 加锁成功，实际有效时间 = 锁的过期时间 - 总耗时
     → 加锁失败，向所有实例（包括成功的）发送解锁请求（用 Lua 脚本，只删自己的锁）

解锁:
  向所有 N 个 Redis 实例发送解锁 Lua 脚本
  不管该实例有没有锁，全部请求一遍
```

**为什么解锁要发给所有实例，不只是发给成功加锁的那几个？** 因为加锁阶段可能存在这样的情况：向某实例发了 SET NX PX 请求，请求确实成功执行了，但回复在网络中丢了。Client 数出来的成功数不包含它，但那个实例上确实有一把锁。如果不给所有实例发解锁，那把"幽灵锁"会一直占用到自然过期。

### 6.4 RedLock 的争议

Martin Kleppmann（《DDIA》作者）对 RedLock 的核心批评：

**时钟依赖问题**：Redis 的过期机制依赖于每个实例自身的时钟。如果某个 Redis 实例的时钟向前跳跃（NTP 校正、虚拟机迁移），锁的有效期被提前终止。而依赖时钟的系统无法在分布式环境下提供安全保证。

**更好的替代方案**：
- **Fencing Token**：每次获取锁时返回一个单调递增的 token。写回共享存储时带上 token，存储层校验 token 是否是最新的——如果不是，拒绝写入。这把"锁的有效性校验"从 Redis 移到了共享存储（如 ZooKeeper 的 zxid）。
- **ZooKeeper / etcd**：基于 Raft/ZAB 的强一致性系统，不容忍脑裂。锁数据必须通过共识写入，不会出现"Master 有但 Slave 没有"的情况。代价是吞吐低于 Redis。

### 6.5 生产环境怎么选

```
Redis 单实例（SET NX PX + 看门狗）
  → 适用: 90% 的场景
  → 风险: 极端情况下可能锁丢失（概率极低但理论存在）
  → 如果你能容忍"百万分之一的并发冲突"→ 这个就够了

RedLock
  → 适用: 需要更高的安全性，但不愿意引入 ZK/etcd
  → 风险: 依赖时钟，运维多套 Redis 实例成本增加

ZooKeeper / etcd
  → 适用: 锁的正确性不能有任何妥协（金融交易、分布式任务调度）
  → 风险: 吞吐远低于 Redis（ZooKeeper 是写路径经过共识的单 Leader 模型）
```

**大部分场景不需要 RedLock**：Redisson 单实例 + 看门狗的方案，在绝大多数情况下足够安全。如果你真的遇到了主从切换丢锁的场景——先检查是不是你的业务逻辑有问题，再考虑 RedLock。

---

## 七、分布式锁的正确使用姿势

### 7.1 锁的粒度

```
❌ 锁住整个用户模块
  lock("user:module")

✅ 锁住具体资源
  lock("user:123:update")

✅ 锁住具体操作
  lock("order:1001:pay")
```

锁粒度越粗 → 并发性能越差 → 锁竞争越多 → 排队等待越多。**锁名包含资源 ID** 是最小粒度，保证不同用户/订单之间完全并发。

### 7.2 加锁的代码模板

```java
public void updateUser(String userId) {
    String lockKey = "lock:user:" + userId;
    RLock lock = redissonClient.getLock(lockKey);
    
    try {
        // tryLock(waitTime, leaseTime, unit)
        // waitTime = 等待获取锁的最长时间
        // leaseTime = 锁的过期时间（-1 = 走看门狗自动续期）
        if (lock.tryLock(3, -1, TimeUnit.SECONDS)) {
            try {
                // 执行业务逻辑
                doUpdateUser(userId);
            } finally {
                // 确保解锁
                lock.unlock();
            }
        } else {
            // 获取锁超时 → 返回业务错误或重试
            throw new BizException("系统繁忙，请稍后重试");
        }
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
        throw new BizException("获取锁被中断");
    }
}
```

**关键细节**：`tryLock` 的 timeout 和 `unlock` 之间永远有一个 `try-finally`。`leaseTime=-1` 表示让 Redisson 的看门狗自动续期——你不用猜"业务需要执行多久"，看门狗帮你保证锁不会在业务完成前过期。

### 7.3 常见错误

**1. 在 finally 中解锁不检查持有者**

```java
// ❌ 直接 unlock → 如果锁已过期被他人持有 → 删了别人的锁
finally {
    lock.unlock();
}

// ✅ 检查是否自己持有
finally {
    if (lock.isHeldByCurrentThread()) {
        lock.unlock();
    }
}
```

Redisson 的 `unlock()` 实际上内部会做持有者检查（Lua 脚本中的 `hexists`），如果锁不是当前线程持有的，会抛出 `IllegalMonitorStateException`——所以推荐用 `isHeldByCurrentThread()` 预检，或直接 catch 异常。

**2. 业务时间 > 锁过期时间，且没用看门狗**

```java
// ❌ 锁 5 秒过期，但业务要跑 10 秒 → 锁提前释放 → 并发冲突
lock.tryLock(3, 5, TimeUnit.SECONDS);
slowOperation();  // 跑了 10 秒

// ✅ leaseTime = -1 → 看门狗自动续期
lock.tryLock(3, -1, TimeUnit.SECONDS);
slowOperation();  // 看门狗每 10 秒续期一次，锁不会过期
```

**3. 在锁内调用了远程接口**

```java
lock.tryLock(3, -1, TimeUnit.SECONDS);
try {
    // 这个 HTTP 调用可能耗时 30 秒 → 看门狗虽然续着期
    // 但其他等待锁的线程会一直阻塞
    String result = restTemplate.getForObject("http://slow-api/data", String.class);
    processData(result);
} finally {
    lock.unlock();
}
```

锁的持有时间 = 所有争用这把锁的线程被阻塞的时间。锁内的网络调用会把高并发退化为串行——**锁应该只保护"真正需要互斥的临界代码"，不应该保护 I/O**。

**4. 不设 waitTime**

```java
// ❌ 无限等待 → 下游服务慢 → 所有请求线程都在排队等锁 → 线程池耗尽
lock.lock();  // 没有 timeout！

// ✅ 必须设 waitTime
if (!lock.tryLock(3, -1, TimeUnit.SECONDS)) {
    throw new BizException("系统繁忙");
}
```

---

## 八、演进总结

```
第一代: SETNX                           → 死锁风险
第二代: SETNX + EXPIRE                   → 原子性问题，极小概率死锁
第三代: SET key value NX PX               → 解决了原子性，解决了死锁
第四代: + Watch Dog 看门狗 + Lua 解锁     → 解决了过期时间不确定，支持可重入
第五代: RedLock (多实例过半确认)           → 解决了主从切换丢锁

大多数场景停在第四代就够了。
```

Redis 分布式锁的设计是一个经典的"逐层发现缺陷 → 逐层修复"的工程演进案例。理解每一代的"为什么不够"，比记住最终的 Lua 脚本更重要。
