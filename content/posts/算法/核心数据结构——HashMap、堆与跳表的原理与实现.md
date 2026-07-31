---
title: "核心数据结构——HashMap、堆与跳表的原理与实现"
date: 2026-07-31
description: 深入HashMap扰动函数与扩容机制、二叉堆的siftUp/siftDown操作、跳表的随机层高与并发优势，给出Java和Python的最优实现与工程场景。
tags: ["算法","数据结构","HashMap","堆","跳表","SkipList","PriorityQueue"]
categories: ["算法"]
---

# 核心数据结构——HashMap、堆与跳表的原理与实现

> 这三个数据结构贯穿了几乎所有的系统设计：HashMap 是 O(1) 查找的基石，堆是 Top K 和优先调度的标配，跳表在 Redis 和并发场景中替代红黑树。本文从底层原理讲到最优实现，再到工程中真正用到它们的场景。

---

## 一、HashMap

### 1.1 核心原理

HashMap 的目标：**任意 key → O(1) 查找对应的 value**。

底层是 `数组 + 链表 + 红黑树`。先对 key 求 hash，定位到数组下标，如果该位置已有数据（哈希碰撞），在链表/红黑树中继续找。

**扰动函数——为什么 hash 值不能直接当数组下标？**

```java
// Java 8 HashMap 的 hash() 方法
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}

// 定位数组下标
int index = hash & (table.length - 1);  // 等价于 hash % length (length 是 2^n)
```

因为数组大小 n 通常远小于 2³²，取模 `& (n-1)` 只用了 hash 值的低几位（比如 n=16 时，只用低 4 位，高位全被忽略了）。如果直接拿 hashCode 取模，高位差异完全浪费——两个 key 的 hashCode 高 16 位完全不同但低 4 位相同，就会撞到同一个桶。

**扰动函数 `h ^ (h >>> 16)` 做的事**：把高 16 位的差异"折叠"到低 16 位，让高位也参与定位运算。比如 `n=16` 时，扰动后的低 4 位同时受原始 hashCode 高 16 位和低 16 位影响，碰撞率显著下降。

**为什么数组长度必须是 2 的幂？** `hash & (n-1)` 等价于取模运算的前提是 n=2^k。而且扩容时 `n<<=1`，老 slot 的节点只可能落在原位置或"原位置 + 老容量"两个地方——不需要重新 hash，直接按高低位拆分链表。

**树化阈值为什么是 8？**

```
链表长度概率（泊松分布，λ=0.5）:
  0: 60.6%    1: 30.3%    2: 7.6%    3: 1.3%
  4: 0.2%     5: 0.03%    6: 0.004%  7: 0.0003%
  8: 0.00000002%
```

链表长度达到 8 的概率不到千万分之一。如果真达到了，说明哈希函数设计有问题、或者有人恶意构造碰撞 key（哈希碰撞攻击）。此时转红黑树——不是因为"链表慢"，而是**防止攻击者利用大量碰撞把 HashMap 退化为 O(n)**。

**退化阈值为什么是 6 不是 8？** 如果在 8 和 8 之间来回振荡——扩容时红黑树拆成两个链表，如果正好一个 ≥8 一个 <8，就会频繁在树和链表之间转换。设 8→树、6→链，中间留 2 的缓冲区，避免振荡。

### 1.2 HashMap 的扩容

当 `size > threshold (= capacity × loadFactor)` 时触发扩容：

```
resize() 的核心流程:
1. newCap = oldCap << 1  (容量翻倍)
2. newThr = newCap * loadFactor
3. 遍历老数组每个 slot:
   - 单节点: 直接 rehash → 新位置
   - 链表: 按 hash & oldCap 是否为 0 拆成两条链表
       high 链 → 原位置 + oldCap
       low 链  → 原位置
   - 红黑树: 同样拆分，≤6 退化回链表
```

**为什么 1.7 扩容会死循环（头插法）？** 1.7 使用头插法将老链表倒序插入新数组。多线程并发 put 触发扩容时，两个线程同时操作同一条链表 → 节点的 next 指针形成环路 → CPU 100%。1.8 改为尾插法保留原始顺序，并在发现树节点时走树的拆分逻辑。

### 1.3 ConcurrentHashMap 的并发设计

```java
// 1.8 的 put 核心逻辑
final V putVal(K key, V value, boolean onlyIfAbsent) {
    // 1. 如果数组未初始化 → CAS 初始化（只有一个线程成功）
    // 2. 如果目标 slot 为空 → CAS 放入节点
    // 3. 如果正在扩容（slot 的 hash == MOVED） → 帮助扩容
    // 4. 否则 → synchronized 锁住该 slot 的头节点 → 插入
    // 5. 最后 addCount 增加元素个数（CAS + 必要时触发扩容）
}
```

关键演化：
- 1.7 Segment 分段锁（16 个 Segment，每个是一个小 ReentrantLock）→ 最大并发 16
- 1.8 CAS + synchronized 锁**单个 Node** → 细粒度锁，并发度 = 数组长度
- size 统计：1.7 三次不加锁统计 + 第四次加锁兜底；1.8 用 `CounterCell[]` 分片累加（LongAdder 原理）→ 无锁化

### 1.4 Java 最优实现

```java
// 构造时指定容量——避免扩容
// 预期 1000 个元素，loadFactor=0.75 → 1000/0.75 ≈ 1334 → 向上取 2^n = 2048
Map<String, Object> map = new HashMap<>(2048);

// 计数：getOrDefault + put 的原子替代
map.merge(key, 1, Integer::sum);       // key 不存在 → 1; 存在 → +1
map.computeIfAbsent(key, k -> load(k)); // 不存在时计算并放入

// 遍历最高效方式
for (Map.Entry<String, Object> entry : map.entrySet()) {
    entry.getKey();
    entry.getValue();
}
```

### 1.5 Python 实现

```python
# Python 3.6+ dict 是有序的（插入顺序保留）
d = {"a": 1, "b": 2}

# 计数
from collections import Counter, defaultdict
c = Counter(['a', 'b', 'a', 'c'])     # Counter({'a': 2, 'b': 1, 'c': 1})
d = defaultdict(list)                  # 不存在的 key 返回 []
d["x"].append(1)

# 获取或默认
val = d.get("key", "default")
val = d.setdefault("key", []).append(1)  # key 不存在时设 [] 并 append

# 合并 dict（Python 3.9+）
merged = d1 | d2
```

**Python dict 的底层实现（3.6+）** 采用紧凑有序设计：`indices` 稀疏数组（存 entries 下标）+ `entries` 密集数组（存 hash + key + value）。相比 Java HashMap（每个 slot 都是链表头指针），Python 的内存更紧凑——所有 entry 挤在密集数组里，缓存友好。开放寻址用伪随机探测序列 `(5*j+1) % 2^k`，避免线性探测的聚集问题。

### 1.6 实际场景

- **本地缓存**：`Map<String, Object> cache = new ConcurrentHashMap<>()` + 惰性加载 `computeIfAbsent`
- **请求去重**：`Set<String> idempotentSet = ConcurrentHashMap.newKeySet()` — 基于 id 判断是否已处理
- **计数统计**：`map.merge(eventType, 1, Integer::sum)` 统计各类事件发生次数
- **路由表**：`Map<String, ServiceInstance> routeTable` — O(1) 查找服务实例

---

## 二、堆（PriorityQueue）

### 2.1 核心原理

堆是一个**完全二叉树**，用数组存储。对于下标 i 的节点：
- 父节点：`(i-1) / 2`
- 左子节点：`2i + 1`
- 右子节点：`2i + 2`

**小顶堆**：每个节点 ≤ 其子节点。堆顶是全局最小值。**大顶堆**反之。

```
小顶堆数组表示：    [1, 3, 2, 7, 5, 4, 8]
                     │
                     ▼
                      1
                    /   \
                   3     2
                  / \   / \
                 7   5 4   8
```

**siftUp（上浮）——插入**：新元素放数组末尾 → 不断与父节点比较 → 如果比父节点小（小顶堆）就交换 → 直到满足堆性质。

**siftDown（下沉）——删除堆顶**：把最后一个元素移到堆顶 → 不断与两个子节点中的较小者比较 → 如果比子节点大就交换 → 直到满足堆性质。

### 2.2 heapify——构造堆的 O(n) 证明

```java
// O(n log n) 方式：逐个插入
for (int x : arr) pq.offer(x);

// O(n) 方式：从最后一个非叶子节点开始，逐个 siftDown
// 最后一个非叶子节点 = 最后一个元素的父节点 = (n-2)/2
for (int i = (n-2)/2; i >= 0; i--) siftDown(i);
```

O(n) 的直观解释：树中越靠近底层的节点越多。siftDown 的代价等于节点到叶子的距离。底层节点只需下浮 0-1 层，高层节点很少。全部节点的总下浮代价 = O(n)。数学推导：`sum(h - level) × 2^level = O(n)`。

### 2.3 Java 最优实现

```java
// Top K：大小为 K 的小顶堆
public int[] topK(int[] nums, int k) {
    PriorityQueue<Integer> pq = new PriorityQueue<>(k);  // 默认小顶堆
    for (int num : nums) {
        pq.offer(num);
        if (pq.size() > k) pq.poll();  // 弹出最小的，保留 K 个最大的
    }
    return pq.stream().mapToInt(Integer::intValue).toArray();
}

// 大顶堆：传入比较器
PriorityQueue<Integer> maxHeap = new PriorityQueue<>(Comparator.reverseOrder());

// 自定义比较器——按频率排序
PriorityQueue<Map.Entry<String, Integer>> pq = new PriorityQueue<>(
    (a, b) -> a.getValue() - b.getValue()  // 小顶堆，按频率升序
);

// 所有 K 个元素不需要有序 → 用堆
// 需要有序 → 堆弹出后再排序，或直接用快排 partition 找第 K 大
```

### 2.4 Python 最优实现

```python
import heapq

# Python 只有小顶堆。大顶堆 = 取负数入堆
nums = [3, 1, 4, 1, 5]
heapq.heapify(nums)          # O(n) 线性建堆

heapq.heappush(nums, 2)      # 插入
smallest = heapq.heappop(nums)  # 弹出最小

# Top K 最大：用小顶堆（堆顶是 "K 个最大中最小的那个"）
k = 3
heap = []
for x in nums:
    heapq.heappush(heap, x)
    if len(heap) > k:
        heapq.heappop(heap)
# heap 中就是 Top 3

# nlargest/nsmallest：内部用堆实现
top3 = heapq.nlargest(3, nums)  # 返回降序 [5, 4, 3]
```

### 2.5 实际场景

- **Top K**：排行榜前 100、热门搜索词 → 大小为 100 的小顶堆，O(n log 100)
- **定时任务调度**：堆顶 = 最早到期的任务，每次 peek 检查是否到期，到期则 poll 执行
- **Dijkstra 最短路径**：优先队列存 `(距离, 节点)`，每次取距离最小的节点扩展
- **合并 K 个有序链表/文件**：堆内存每条链表的当前节点，每次弹出最小的

---

## 三、跳表（SkipList）

### 3.1 核心原理

跳表 = **多层有序链表**。最底层包含所有元素，每向上一层元素数量减少（约 1/4）。查找时从最高层开始，水平向右 → 遇到更大的值就下降一层 → 继续向右。

```
Level 2:  1 ────────────── 9
Level 1:  1 ─── 3 ─── 5 ─── 9
Level 0:  1 → 2 → 3 → 4 → 5 → 6 → 7 → 8 → 9

查找 6:
  L2: 1→9(太大,降层) → L1: 3→5→9(太大,降层) → L0: 5→6 找到
```

**时间复杂度**：查找/插入/删除都是 O(log n)。每层最多跳 2 次就会降层。

**插入时的随机层高**（Redis 实现）：

```c
// Redis 源码: t_zset.c
int zslRandomLevel(void) {
    int level = 1;
    // 每次有 1/4 概率升级（ZSKIPLIST_P = 0.25）
    while ((random() & 0xFFFF) < (ZSKIPLIST_P * 0xFFFF))
        level += 1;
    return (level < ZSKIPLIST_MAXLEVEL) ? level : ZSKIPLIST_MAXLEVEL;
}
```

1/4 概率升级 → 平均每 4 个节点中有 1 个在 Level 2，每 16 个有 1 个在 Level 3 → 层数分布符合几何分布。最高 32 层可支持 2^64 个元素。

### 3.2 跳表 vs 红黑树——为什么 Redis 和 Java 并发集合选跳表

| 维度 | 跳表 | 红黑树 |
|------|------|--------|
| 实现复杂度 | 简单（无旋转、无色） | 复杂（旋转+变色，插入删除需修复） |
| 范围查询 | 天然支持（有序链表） | 需中序遍历（非局部操作） |
| 并发友好 | 只锁局部节点（上下左右指针修改范围小） | 旋转可能影响大范围节点 |
| 内存 | 指针多（每节点平均 1.33 个前向指针） | 紧凑（左/右/父三个指针+颜色） |

**Redis ZSet 用跳表的原因**：ZSet 需要支持 ZRANGE（范围查询）和 ZRANK（排名查询）——跳表的底层有序链表天然支持。同时跳表的插入/删除不需要平衡操作，在 Redis 单线程模型下实现简洁。

**ConcurrentSkipListMap 用跳表的原因**：红黑树的旋转会牵连多个节点，做并发控制时需要锁住可能被旋转影响的大范围路径。跳表的插入只修改前向指针，涉及的节点数是局部的（平均每层走两步，且大部分节点只有低层有指针），用 CAS 控制更容易实现高效并发。

### 3.3 Java 最优实现

```java
// ConcurrentSkipListMap：线程安全、有序、并发
ConcurrentSkipListMap<String, Integer> map = new ConcurrentSkipListMap<>();
map.put("c", 3);
map.put("a", 1);
map.put("b", 2);

// 范围查询
SortedMap<String, Integer> range = map.subMap("a", "c");   // [a, c)
Map.Entry<String, Integer> ceiling = map.ceilingEntry("b"); // >= b 的最小键
```

### 3.4 Python 实现

Python 无内置跳表。简化版实现：

```python
import random

class SkipNode:
    def __init__(self, val, level):
        self.val = val
        self.next = [None] * (level + 1)  # next[i] = 第 i 层的前向指针

class SkipList:
    MAX_LEVEL = 16
    
    def __init__(self):
        self.head = SkipNode(None, self.MAX_LEVEL)  # 哨兵头节点
        self.level = 0
    
    def _random_level(self):
        level = 0
        while random.random() < 0.25 and level < self.MAX_LEVEL:
            level += 1
        return level
    
    def search(self, target):
        cur = self.head
        for i in range(self.level, -1, -1):        # 从最高层向下
            while cur.next[i] and cur.next[i].val < target:
                cur = cur.next[i]                   # 水平前进
        cur = cur.next[0]                           # 降到第 0 层
        return cur is not None and cur.val == target
    
    def insert(self, val):
        update = [None] * (self.MAX_LEVEL + 1)      # 每层的前驱节点
        cur = self.head
        for i in range(self.level, -1, -1):
            while cur.next[i] and cur.next[i].val < val:
                cur = cur.next[i]
            update[i] = cur
        
        new_level = self._random_level()
        if new_level > self.level:
            for i in range(self.level + 1, new_level + 1):
                update[i] = self.head
            self.level = new_level
        
        node = SkipNode(val, new_level)
        for i in range(new_level + 1):
            node.next[i] = update[i].next[i]
            update[i].next[i] = node
```

### 3.5 实际场景

- **Redis ZSet**：元素 > 128 或成员长度 > 64 字节时，底层从 ziplist 切换到跳表。跳表 + 哈希表组合——哈希做 O(1) 按成员查分数，跳表做 O(log n) 范围查询和排名
- **ConcurrentSkipListMap**：需要并发且有序的场景（如在线玩家排行榜、延迟队列）
- **HBase MemStore**：内存中的数据用 ConcurrentSkipListMap 存储，保证 KeyValue 有序

---

这三者配合：HashMap 解决"快查"，堆解决"快排前 K 个"，跳表解决"有序 + 并发"——是工程中最常打交道的数据结构。
