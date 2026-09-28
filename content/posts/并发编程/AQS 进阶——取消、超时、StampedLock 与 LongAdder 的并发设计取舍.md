---
title: "AQS 进阶——取消、超时、StampedLock 与 LongAdder 的并发设计取舍"
date: 2026-08-07
description: 从 AQS 的 cancelAcquire() 节点取消逻辑（幽灵节点跳过 + SIGNAL 接力）、ConditionObject.awaitNanos() 的超时与 parkNanos 的虚假唤醒配合、StampedLock 为什么不用 AQS（乐观读的"版本号验证"需要全新队列设计）、到 AbstractQueuedLongSynchronizer 用 long 替代 int state 的场景需求，拆解 AQS 设计中四个"被省略的细节"背后的精妙考量。
tags: ["Java","并发","AQS","StampedLock","Condition","LockSupport"]
categories: ["Java并发"]
---

# 一、cancelAcquire()——队列中"不玩了"的线程怎么优雅退出

## 1.1 什么情况下节点需要取消？

```java
// 一个线程在 acquireQueued 中等待时，可能因为以下原因需要取消：
// ① 超时——tryAcquireNanos(1, TimeUnit.SECONDS) 等了 1 秒还没拿到
// ② 中断——另一个线程调了 t.interrupt()
// ③ 异常——tryAcquire 中抛了异常

// 此时该线程的 Node 还在 CLH 队列中——需要把它"摘掉"
// 但不能简单地 prev.next = next ——队列可能正在被其他线程并发修改
```

## 1.2 cancelAcquire 的精妙实现

```java
// AQS.cancelAcquire() 的完整逻辑（简化）：
private void cancelAcquire(Node node) {
    if (node == null) return;
    
    // ① 清空 node 的 thread 引用（方便 GC）
    node.thread = null;
    
    // ② 跳过前面已取消的节点——找到最近一个未取消的前驱
    Node pred = node.prev;
    while (pred.waitStatus > 0)  // waitStatus > 0 只有 CANCELLED=1
        pred = (node.prev = pred.prev);  // 向前跳
    
    // ③ 记录 pred 的后继（用于后续接力）
    Node predNext = pred.next;
    
    // ④ 把自己标记为 CANCELLED
    node.waitStatus = Node.CANCELLED;
    
    // ⑤ 如果自己是 tail → CAS 把 tail 指回 pred
    if (node == tail && compareAndSetTail(node, pred)) {
        // pred 是新的 tail → CAS 把 pred 的 next 清空
        pred.compareAndSetNext(predNext, null);
    } else {
        // ⑥ 不是 tail → 需要让 pred 的 SIGNAL 状态传递到后继
        //    如果 pred 不是 head、且 (pred.waitStatus == SIGNAL 或 CAS 设为 SIGNAL)
        //    且 pred.thread != null（pred 没被取消）
        //    → 把 pred.next 从自己改为自己的后继
        int ws;
        if (pred != head &&
            ((ws = pred.waitStatus) == Node.SIGNAL ||
             (ws <= 0 && pred.compareAndSetWaitStatus(ws, Node.SIGNAL))) &&
            pred.thread != null) {
            Node next = node.next;
            if (next != null && next.waitStatus <= 0)
                pred.compareAndSetNext(predNext, next);  // pred → next（跳过自己）
        } else {
            // ⑦ pred 是 head 或无法设置 SIGNAL → 唤醒后继
            unparkSuccessor(node);
        }
    }
}
```

**关键设计点**：

```
① 为什么不是直接"把前驱的 next 指向自己的后继"？
   因为 pred 可能也在被其他线程取消——直接设 pred.next 不安全
   必须先"跳掉前面的已取消节点"找到稳定的 pred，再小心地 CAS pred.next

② 为什么步骤⑥需要判断 pred != head？
   如果 pred 是 head → pred 代表当前持有锁的线程
   → 不能设 head 的 SIGNAL → 应该调 unparkSuccessor 让 head 自己唤醒后继

③ 为什么"跳过已取消节点"只用 prev 指针、不用 next 指针？
   因为 prev 的更新是"先 CAS 设 tail/prev → 再设 next"
   → 从后往前遍历（prev）一定不会丢失未取消的节点
   → 从前往后遍历（next）可能在 next 没设完之前就跳过去了
```

## 1.3 "幽灵节点"与 SIGNAL 接力

```
取消后 CLH 队列的变化：

取消前：head → A(waitStatus=SIGNAL) → B(waitStatus=CANCELLED) → C(waitStatus=-1)

B 执行 cancelAcquire 后：
  ① B.thread = null （B 变成"幽灵节点"——节点还在队里，但已没有关联的线程）
  ② B.waitStatus = CANCELLED
  
  如果 B 是 tail → CAS tail = A → A.next = null（B 从队尾被摘掉）
  
  如果 B 不是 tail → 需要"SIGNAL 接力"：
    A.next = C（跳过 B）
    A.waitStatus 仍然 = SIGNAL → A 释放锁时会唤醒 C
    → C 醒来后 acquireQueued 中的 shouldParkAfterFailedAcquire 检测到
      自己的 prev（B）的 waitStatus=CANCELLED → 向前跳 → prev = A
      
  next 更新是"可选"的——即使 A.next 没被成功更新为 C
  C 被唤醒之后也会通过 prev 指针跳过 B 找到 A
  → next 指针失效不影响正确性，只影响性能（多跳几次）
```

---

# 二、ConditionObject.awaitNanos()——超时怎么和 parkNanos 配合

## 2.1 完整的 awaitNanos 调用链

```java
// ConditionObject.awaitNanos(nanosTimeout) 的完整实现（简化）：
public final long awaitNanos(long nanosTimeout) throws InterruptedException {
    if (Thread.interrupted()) throw new InterruptedException();
    
    // ① 创建 CONDITION 节点，加入 Condition 队列（单向链表）尾部
    Node node = addConditionWaiter();
    
    // ② 完全释放锁（state 归零，记录重入次数 savedState）
    int savedState = fullyRelease(node);
    
    // ③ 计算超时的绝对时间点（deadline）
    final long deadline = System.nanoTime() + nanosTimeout;
    int interruptMode = 0;
    
    // ④ 核心等待循环
    while (!isOnSyncQueue(node)) {
        // ④a 超时 → 取消等待 → 把自己从 Condition 队列移到 CLH 队列
        if (nanosTimeout <= 0L) {
            transferAfterCancelledWait(node);  // 转移到 CLH，不成功就等一小会
            break;
        }
        
        // ④b 如果还剩 > 1000ns → parkNanos
        if (nanosTimeout >= SPIN_FOR_TIMEOUT_THRESHOLD) {  // 1000ns
            LockSupport.parkNanos(this, nanosTimeout);
        }
        // ≤ 1000ns → 不 park，直接自旋（park 的开销不值得小于 1μs 的等待）
        
        // ④c 醒来后检查是否被中断
        if (Thread.interrupted()) {
            interruptMode = ...;
            break;
        }
        
        // ④d 重新计算剩余时间
        nanosTimeout = deadline - System.nanoTime();
    }
    
    // ⑤ 被 signal 或超时后，在 CLH 队列中重新竞争锁
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
        interruptMode = REINTERRUPT;
    
    // ⑥ 如果是被 signal 后从 Condition 队列转移走的 → 清理后继
    if (node.nextWaiter != null) unlinkCancelledWaiters();
    
    // ⑦ 处理中断
    if (interruptMode != 0) reportInterruptAfterWait(interruptMode);
    
    // ⑧ 返回剩余时间（> 0 = 超时返回负数；≤ 0 = 时间到了）
    return deadline - System.nanoTime();
}
```

## 2.2 为什么要用"deadline"而不是"每次传入剩余时间"？

```java
// ❌ 错误做法：每次 parkNanos 后把剩余时间传给下一次 parkNanos
// 剩余 = 原始超时 - (当前时间 - 开始时间)
//    = 1000000000 - (System.nanoTime() - start) 
// → 如果 System.nanoTime() 在两次调用之间受 NTP 校时的影响
//   可能计算出错误的剩余时间

// ✅ 正确做法：用绝对时间点做 deadline
// deadline = System.nanoTime() + nanosTimeout（一开始就算好）
// 每次醒来 → nanosTimeout = deadline - System.nanoTime()
// → 即使 System.nanoTime() 被 NTP 调整，deadline 是绝对时间点，不受影响
```

## 2.3 虚假唤醒的处理

```java
// LockSupport.parkNanos() 可能在以下情况提前返回（不等到超时）：
// ① 真正的 signal() → isOnSyncQueue(node) = true → 退出循环 ✓
// ② 虚假唤醒（Spurious Wakeup） → isOnSyncQueue(node) = false → 继续等
// ③ 中断 → Thread.interrupted() = true → 处理中断

// 关键：
while (!isOnSyncQueue(node)) {   // ← 不是 if！是 while！
    // isOnSyncQueue(node) == false → 继续 park
    // isOnSyncQueue(node) == true → signal 已处理 → 退出循环
    LockSupport.parkNanos(this, nanosTimeout);
}
```

## 2.4 SPIN_FOR_TIMEOUT_THRESHOLD——1 微秒的"不值得 park"

```java
// 如果剩余等待时间 < 1000ns（1μs）→ 不 park，直接自旋
// 原因：park 是系统调用 → ~1μs 的开销
//      → 如果剩余等待时间 < 1μs，park 的开销大于等待本身
//      → 不如自旋等一小会

static final long SPIN_FOR_TIMEOUT_THRESHOLD = 1000L; // 纳秒

if (nanosTimeout >= SPIN_FOR_TIMEOUT_THRESHOLD)
    LockSupport.parkNanos(this, nanosTimeout);
// else → 自旋，不 park
```

---

# 三、StampedLock——为什么不用 AQS？

## 3.1 AQS 为什么不适合 StampedLock？

AQS 的核心是**一个 int state + CLH 队列**，所有同步器都围绕"争这一口 state"来设计。但 StampedLock 有三种访问模式：

```
① 写锁（Write）：独占，和 synchronized 一样
② 悲观读锁（Read）：共享，和 ReentrantReadWriteLock 一样
③ 乐观读锁（Optimistic Read）：完全不阻塞！拿一个 stamp（版本号），读完后验证 stamp 是否变化——没变 = 数据有效，变了 = 重试
```

**AQS 处理不了"乐观读"**，因为：
- AQS 的 `acquireShared` 就是"排队的共享模式"——乐观读不排队，它只是一次 `tryOptimisticRead()` 返回一个 stamp
- 乐观读不修改 state，不需要 CAS 去争——和 AQS 的"CAS 争 state"模型完全相反
- `validate(stamp)` 是纯读操作，不涉及任何 AQS 的队列、park/unpark 逻辑

## 3.2 StampedLock 的内部设计

```java
// StampedLock 的核心状态
private static final long ORIGIN = 0b00000000_00000000_00000000_00000000_00000000_00000000_00000000_00000001;

// 只有一个 volatile long state，编码了三个信息：
//   - 写锁：state & 0x1000000000000000L != 0  → 写锁被持
//   - 读锁：state & 0x0FFFFFFFFFFFFFFFL → 读锁计数
//   - 锁版本：state >>> 48 → 每次释放写锁时递增（这就是 stamp 的来源）

// 乐观读：
public long tryOptimisticRead() {
    long s = state;
    // 如果写锁被持 → 返回 0（乐观读失败）
    // 如果写锁未持 → 返回 s 的低位（版本号）
    return (s & WBIT) == 0L ? s & SBITS : 0L;
}

// 验证 stamp 是否仍然有效
public boolean validate(long stamp) {
    // 内存屏障：保证 load 顺序
    VarHandle.acquireFence();
    // stamp 的版本号 == state 中的版本号 → 写锁未被持 = 数据有效
    return (stamp & SBITS) == (state & SBITS);
}

// 乐观读的使用模式：
long stamp = lock.tryOptimisticRead();  // ① 拿版本号，没有锁！
String data = readSharedData();         // ② 无锁读数据
if (!lock.validate(stamp)) {            // ③ 验证版本号是否变了
    stamp = lock.readLock();            // ④ 版本号变了 → 退化到悲观读锁
    try {
        data = readSharedData();
    } finally {
        lock.unlockRead(stamp);
    }
}
```

## 3.3 StampedLock 的队列——自己实现，不用 AQS

```java
// StampedLock 维护自己的 CLH 队列（简化）

// 写节点（WNode）是双向链表：
static final class WNode {
    volatile WNode prev;
    volatile WNode next;
    volatile WNode cowait;  // ← 读节点链！读锁等待时挂成的"读链"
    volatile Thread thread;
    volatile int mode;       // RMODE = 读, WMODE = 写
    volatile int status;     // 0 = 等待, 1 = 取消
}

// StampedLock 的 CLH 队列有两种特殊的节点：
// ① 写节点：和 AQS 的 EXCLUSIVE 一样，FIFO 排队
// ② 读节点：多个读节点通过 cowait 字段形成"读链"挂在同一个写节点上
//   → 当锁释放时，唤醒队首的读链 → 所有读节点一起释放（共享特性）
```

**StampedLock 和 AQS 的队列差异本质**：

```
AQS 的队列：EXCLUSIVE + SHARED 两种节点，SHARED 节点释放时传播式唤醒
StampedLock 的队列：只有一个 state 编码所有信息 + 自己的队列，
  关键是"乐观读"完全不参与排队——它只是一个版本号的读和验证
  悲观读和写才排队——但这个排队逻辑是 StampedLock 自己实现的
  （不是复用 AQS），因为它需要在同一把锁上管理"乐观读 stamp"
  和"悲观读/写 CLH"两个维度的状态
```

---

# 四、AbstractQueuedLongSynchronizer——什么时候需要 long state？

## 4.1 AQS 的 int state 局限性

AQS 的 state 是 `volatile int`——这是刻意的，因为 `compareAndSetState` 底层用 `Unsafe.compareAndSwapInt`，32 位 CAS 在 32 位 JVM 上是一条 CPU 指令。

但 32 位 int 的表达范围是 [-2^31, 2^31-1]。某些场景下不够用：

## 4.2 什么时候需要 long state？

```java
// 场景 1：Semaphore 的许可证数 > 2^31-1
// → 如一个超大的连接池（虽然现实中心见的）

// 场景 2：多维度编码（把一个 long 分成多个 bit 段）
// ReentrantReadWriteLock 的 state 高 16 位= 读锁计数，低 16 位 = 写锁计数
// → 各只有 16 位的范围（65535），在极高并发下也不够
// → 如果用 long → 可放宽到 32 位各段

// 场景 3：自定义同步器中需要 64 位原子操作
// → 如需要原子地编码"时间戳 + 计数器"到一个 long
// → 如果 32 位 CAS 的竞争在这里不存在，long 能提供更大范围
```

## 4.3 AbstractQueuedLongSynchronizer 的实现——就是改了一个类型

```java
// AQS（int state）:
public abstract class AbstractQueuedSynchronizer {
    private volatile int state;
    protected final boolean compareAndSetState(int expect, int update) {
        return U.compareAndSetInt(this, STATE, expect, update);
    }
}

// AQLS（long state）——几乎一模一样的代码，就是 int → long：
public abstract class AbstractQueuedLongSynchronizer {
    private volatile long state;
    protected final boolean compareAndSetState(long expect, long update) {
        return U.compareAndSetLong(this, STATE, expect, update);
    }
}

// 代价：在 32 位 JVM 上，compareAndSetLong 需要"两次 compareAndSwapInt"（64 位原子操作不是原生的）
// → 如果部署在 32 位 JVM 上，性能会降低
// → 这是 AQS 默认用 int 而不是 long 的真实原因
```

**面试追问：AQLS 为什么几乎没人用？** 因为 int 在绝大多数场景下够用（Semaphore 许可数 > 21 亿的场景几乎不存在，ReentrantReadWriteLock 各 16 位也够用）。只有极端并发或自定义编码场景才需要 long。而且现代 JVM 几乎都是 64 位的，在 64 位 JVM 上 `compareAndSetLong` 是一条 CPU 指令（`CMPXCHG16B`），性能和 int 一样——但 AQS 的 int 设计要追溯到 2004 年，那时候 32 位 JVM 还是主流。

---

# 五、总结

| 问题 | 答案 |
|------|------|
| **cancelAcquire 为什么不能直接 `prev.next = next`？** | 队列正在被并发修改；必须先"跳过前面已取消的节点"，然后 CAS pred.next；next 指针更新失败不影响正确性（prev 是"主索引"） |
| **awaitNanos 为什么用 deadlin 而不用剩余时间？** | NTP 校时可能影响 `System.nanoTime()` → 用绝对时间点不受影响 |
| **parkNanos 前为什么要 SPIN_FOR_TIMEOUT？** | 剩余时间 < 1μs → 不值得 park（park 的开销 ~1μs ）→ 自旋更高效 |
| **StampedLock 为什么不用 AQS？** | 乐观读不参与排队——它只是一个 stamp 的读+验证，AQS 的 state+CLH 模型装不下"不排队的读" |
| **AQLS 为什么几乎没人用？** | int state 在 99.99% 场景下够用；32 位 JVM 上 long CAS 性能差；64 位 JVM 上 int 和 long CAS 性能一样——但 AQS 诞生年代 32 位还是主流 |

# 延伸阅读

**Do——动手验证：**
- 用两个线程构造一个"超时取消"的场景——线程 1 持锁 2 秒，线程 2 `tryAcquireNanos(1, TimeUnit.SECONDS)` → 观察 CLH 队列中线程 2 的节点如何被取消
- StampedLock 的乐观读和悲观读的性能对比——10 个读线程、持锁 1 秒、分别用 `tryOptimisticRead+validate` vs `readLock`，用 JMH 压测

**Todo——深入方向：**
- AQS 的 `doReleaseShared`——共享模式释放时如何传播式唤醒后继共享节点
- StampedLock 的写锁饥饿问题——高并发读下写锁永远拿不到怎么办
- `LockSupport.park/unpark` 与 `Object.wait/notify` 的底层实现差异——为什么 park 不需要先获得锁

*本文参考资料：*
- OpenJDK 源码: `AbstractQueuedSynchronizer.cancelAcquire()` / `ConditionObject.awaitNanos()`
- OpenJDK 源码: `StampedLock` / `AbstractQueuedLongSynchronizer`
- Doug Lea, "The java.util.concurrent Synchronizer Framework" (AQS 论文), 2004
