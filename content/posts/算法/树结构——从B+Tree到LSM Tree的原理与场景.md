---
title: "树结构——从B+Tree到LSM Tree的原理与场景"
date: 2026-07-31
description: 深入B+Tree的IO优化逻辑、红黑树的旋转与变色、LSM Tree的写放大与读放大tradeoff、Trie的前缀匹配，给出每个结构的Java/Python实现与数据库选型的底层解释。
tags: ["算法","数据结构","B+Tree","红黑树","LSM Tree","Trie","二叉树"]
categories: ["算法"]
---

# 树结构——从 B+Tree 到 LSM Tree 的原理与场景

> MySQL 为什么选 B+Tree？Kafka 为什么能写那么快？MQTT 通配符怎么匹配？——这些问题都指向同一个数据结构家族：树。本文从 B+Tree 的 IO 优化推起，经红黑树的平衡哲学、LSM Tree 的写优化思路，直到 Trie 的前缀匹配应用。

---

## 一、B+Tree

### 1.1 核心原理——为什么要用 B+Tree

**关键约束**：磁盘 IO 的单位是"页"（InnoDB 默认 16KB），不是字节。让每次 IO 带到内存的数据尽可能有用，是 B+Tree 设计的核心目标。

**B+Tree 的三个关键设计**：

1. **非叶子节点只存 key**（不存数据行/data 指针）。同样 16KB 的页，B+Tree 的非叶子能存的 key 数量远超 B-Tree。扇出更大 → 树更矮 → 从根到叶的 IO 次数更少。InnoDB 中一个 16K 页约能存 1200 个 key（假设 key 为 bigint），3 层就能索引 `1200³ ≈ 17 亿`行——根在 buffer pool 常驻，实际找到一行只需 1-2 次磁盘 IO。

2. **叶子节点形成有序双向链表**。范围查询 `WHERE id BETWEEN 100 AND 200` 在 B+Tree 中只需定位到 100 的叶子 → 沿着链表向右遍历到 200。B-Tree 的叶子不相连，范围查询需要中序遍历——不断上下移动在父节点和叶子之间。

3. **数据只存叶子节点**。所有查询的路径长度相同（稳定），非叶子节点的修改不需要考虑数据迁移。

### 1.2 聚簇索引与二级索引

InnoDB 中表数据按主键顺序存储在聚簇索引的叶子节点中。`SELECT * FROM user WHERE id = 100` 只走一棵树。

二级索引（如 `idx_name`）的叶子节点不存完整行，只存**该二级索引列 + 主键**。`SELECT * FROM user WHERE name = 'Alice'` 的流程是：

```
1. 在 idx_name 的 B+Tree 中按 "Alice" 查找 → 找到主键 id=100
2. 用 id=100 去聚簇索引再查一次 → 拿到完整行
   ↑ 这就是"回表"
```

`SELECT id FROM user WHERE name = 'Alice'` 不需要回表——`id` 已经在二级索引的叶子节点里了。这就是**覆盖索引**的优化意义。

### 1.3 Java/Python 实现（简化版）

只实现核心的三个方法：search/insert/range_search：

```java
// 简化版 B+Tree 节点结构（仅 key，不含 data）
class BPlusNode {
    boolean isLeaf;                       // 是否叶子节点
    List<Integer> keys;                   // 有序 key 列表
    List<BPlusNode> children;             // 内部节点：子节点指针
    BPlusNode next;                       // 叶子节点：指向下一叶子的链表指针
    BPlusNode prev;                       // 叶子节点：指向上一叶子
}

// search：从根开始，每层二分定位 → 内部节点去子节点 → 叶子节点二分查找
// insert：找到叶子 → 插入 key → 若溢出则分裂（中间 key 上提到父节点）→ 递归向上
// range_search：找到起始叶子 → 向右遍历链表直到超过 end
```

```python
class BPlusNode:
    def __init__(self, is_leaf=False):
        self.is_leaf = is_leaf
        self.keys = []
        self.children = []     # 内部节点
        self.next = None       # 叶子节点链表

class BPlusTree:
    def __init__(self, order=4):  # order = 每个节点的最大子节点数
        self.root = BPlusNode(is_leaf=True)
        self.order = order
    
    def search(self, key):
        node = self.root
        while not node.is_leaf:
            # 找到第一个 >= key 的位置
            i = 0
            while i < len(node.keys) and key >= node.keys[i]:
                i += 1
            node = node.children[i]
        # 在叶子节点中查找
        for k in node.keys:
            if k == key:
                return True
        return False
    
    def range_search(self, start, end):
        """返回 [start, end] 范围内的所有 key"""
        node = self.root
        while not node.is_leaf:
            i = 0
            while i < len(node.keys) and start >= node.keys[i]:
                i += 1
            node = node.children[i]
        
        result = []
        while node:
            for k in node.keys:
                if start <= k <= end:
                    result.append(k)
                elif k > end:
                    return result
            node = node.next
        return result
```

### 1.4 实际场景

- **MySQL InnoDB 的所有索引**都是 B+Tree（除了全文索引和空间索引使用其他结构）。理解 B+Tree 才能理解索引的左前缀原则：联合索引 `(a,b,c)` 走 `WHERE a=1 AND b=2` 时，B+Tree 先按 a 定位 → 在 a 的范围内按 b 二分——这等价于二级索引的查找过程，所以能用到索引
- **MongoDB 的默认索引**也是 B-Tree 变体，WiredTiger 存储引擎内部用 B-Tree
- **PostgreSQL 的 B-Tree 索引**本质上也是 B+Tree（叶子链接成有序链）

---

## 二、红黑树

### 2.1 核心原理——五个性质的博弈

红黑树不是"严格的平衡树"（AVL 要求左右子树高度差 ≤ 1），而是一个**近似平衡树**。五个性质：

1. 节点是红色或黑色
2. 根节点是黑色
3. 叶子（NIL）是黑色
4. 红色节点的两个子节点必须是黑色（红不相邻）
5. 从任意节点到其所有叶子（NIL）的路径上，黑色节点数相同

**性质 5 的强大之处**：它只约束了"黑色节点数"，对红色节点没有限制。红色节点是"免费"的——可以挂在两个黑色节点之间而不增加路径长度差。这给了红黑树比 AVL 更多的灵活性：插入删除时更少旋转，代价是可能最长路径 = 最短路径 × 2（红-黑-红-黑 交替 vs 全黑）。实际场景中插入远多于查找，这个 tradeoff 值得。

### 2.2 插入修复——三种情况

插入的节点设为**红色**（不破坏性质 5，只可能破坏性质 4"红不相邻"）。修复看"叔父"的颜色：

```
设新节点为 N（红），父节点 P（红），祖父 G（黑），叔父 U

Case 1: U 是红色
  → P 和 U 染黑，G 染红 → G 作为新的 N 向上递归修复

Case 2: U 是黑色，N 和 P 不在同一侧（N 是左子但 P 是右子，反之亦然）
  → 旋转 P，变成 Case 3

Case 3: U 是黑色，N 和 P 在同一侧（N 和 P 都是左子，或都是右子）
  → 旋转 G，P 染黑，G 染红 → 修复完成
```

最多 2 次旋转（Case 2 + Case 3），远少于 AVL 的最坏 O(log n) 次。

### 2.3 Java 实现——TreeMap 的红黑树

```java
// TreeMap 的核心定位：有序 Map
TreeMap<Integer, String> map = new TreeMap<>();
map.put(3, "c");
map.put(1, "a");
map.put(2, "b");

// 范围查询
SortedMap<Integer, String> sub = map.subMap(1, 3);   // [1, 3)
Integer first = map.ceilingKey(2);                    // >= 2 的最小键 = 2
Integer last  = map.floorKey(2);                      // <= 2 的最大键 = 2
```

TreeNode（HashMap 树化节点）内部也是红黑树实现：

```java
// HashMap 树化的条件：链表长度 >= 8 且数组长度 >= 64
// TreeNode 的 putTreeVal 内部调用 balanceInsertion（插入后修复）
// 删除时走 balanceDeletion（删除后修复）
```

### 2.4 Python 实现（手写核心操作）

Python 无内置红黑树。手写插入修复：

```python
class RBNode:
    def __init__(self, val, color="RED"):
        self.val = val
        self.color = color
        self.left = None
        self.right = None
        self.parent = None

class RedBlackTree:
    def _rotate_left(self, x):
        y = x.right
        x.right = y.left
        if y.left:
            y.left.parent = x
        y.parent = x.parent
        if not x.parent:
            self.root = y
        elif x == x.parent.left:
            x.parent.left = y
        else:
            x.parent.right = y
        y.left = x
        x.parent = y
    
    def _insert_fixup(self, z):
        while z.parent and z.parent.color == "RED":
            if z.parent == z.parent.parent.left:       # 父是左子
                y = z.parent.parent.right               # 叔父
                if y and y.color == "RED":               # Case 1
                    z.parent.color = "BLACK"
                    y.color = "BLACK"
                    z.parent.parent.color = "RED"
                    z = z.parent.parent
                else:
                    if z == z.parent.right:               # Case 2
                        z = z.parent
                        self._rotate_left(z)
                    z.parent.color = "BLACK"               # Case 3
                    z.parent.parent.color = "RED"
                    self._rotate_right(z.parent.parent)
            else:                                        # 对称
                # (镜像处理)
                pass
        self.root.color = "BLACK"
```

### 2.5 实际场景

- **TreeMap/TreeSet**：需要有序遍历的 key-value 或去重集合
- **HashMap 树化**：桶内碰撞过多时退化防护
- **Linux CFS 调度器**：vruntime（虚拟运行时间）用红黑树存储，每次 schedule 选 vruntime 最小的进程——就是红黑树的最左叶子。插入和删除都在 O(log n) 内完成
- **epoll 的事件管理**：监控的 fd 集合用红黑树存储，O(log n) 增删查找

---

## 三、LSM Tree

### 3.1 核心原理——写入最快的树

B+Tree 和红黑树的设计目标都是"查找快"。但很多场景（日志、消息队列、时序数据）是**写多读少**。LSM Tree 的核心洞察：不追求每次查找都快，而是**把随机写入全部转成顺序写入**。

```
写入路径:
  1. 数据先写入 MemTable（内存有序结构，通常是跳表/红黑树）
  2. MemTable 满了 → flush 成 SSTable 文件（Sorted String Table，不可变）
  3. 后台 compaction 线程合并多个 SSTable 文件：
      按 key 排序去重 → 生成新的 SSTable → 删除旧的

读取路径:
  1. 先查 MemTable
  2. 再查 SSTable（从新到旧）
  3. 因为数据可能存在多个 SSTable 中，需要合并结果
```

**写放大和读放大的 tradeoff**：

- 写放大：一条数据被 compaction 多次合并 → 实际写磁盘量 = 原始数据 × 写放大倍数
- 读放大：一次查询要检查 MemTable + 多个 SSTable → 读磁盘量 = 原始数据 × 读放大倍数
- Leveled Compaction vs Size-Tiered Compaction 就是在两者之间取舍

### 3.2 Kafka 为什么是 LSM 的思想

Kafka 不直接使用 LSM Tree 数据结构，但它的设计哲学完全一致：

- 消息追加到 segment 文件（顺序写 = LSM 的 SSTable 写入）
- 消息不可变，不删除（SSTable 也不修改，只生成新文件）
- 靠 page cache 缓存最近写入的热数据（LSM 靠 Bloom Filter 和缓存加速读）
- compaction = Kafka 的 log cleanup（按 retention 删除旧 segment 或按 key compaction 保留最新值）

Kafka 的写入吞吐能达到百万条/秒的根本原因：**纯顺序写盘 + page cache**。没有 B+Tree 的随机查找开销，没有索引维护开销。

### 3.3 Java 场景——RocksDB

```java
// RocksDB 是 LSM Tree 的工业级实现（Facebook 基于 LevelDB 打造）
// Flink/ Spark/ Kafka Streams 的状态后端都用它
RocksDB.loadLibrary();
try (Options options = new Options().setCreateIfMissing(true);
     RocksDB db = RocksDB.open(options, "/data/rocksdb")) {
    
    // 写入
    db.put("key1".getBytes(), "value1".getBytes());
    
    // 读取
    byte[] value = db.get("key1".getBytes());
    
    // 批量写入（WriteBatch 保证原子性）
    try (WriteBatch batch = new WriteBatch()) {
        batch.put("k1".getBytes(), "v1".getBytes());
        batch.put("k2".getBytes(), "v2".getBytes());
        db.write(new WriteOptions(), batch);
    }
}
```

### 3.4 实际场景

- **Kafka 的日志存储**：顺序写 segment 文件（虽然不是完整 LSM Tree，但思想一致）
- **RocksDB / LevelDB**：LSM Tree 的标准实现，Flink 状态后端、TiKV 存储引擎
- **HBase / Bigtable**：MemStore(内存) + HFile(磁盘 SSTable) + compaction
- **ClickHouse**：MergeTree 引擎，按分区写 part → 后台 merge

---

## 四、Trie（前缀树）

### 4.1 核心原理

Trie 的每个节点代表一个字符，从根到某节点的路径拼起来就是该节点代表的字符串。所有公共前缀共享节点。

```
插入 "abc", "abd", "ace":

      root
       │
       a
       │
    ┌──┴──┐
    b      c
    │      │
 ┌──┴──┐   e
 c     d
(*)   (*)
      ↑ * = isEnd, 表示一个完整单词的结束
```

查找 "abc" → 沿着 a→b→c 走，看 c 是否标记为 isEnd。时间复杂度 O(len(key))，与树中有多少 key 无关。

### 4.2 Java/Python 实现

```java
class TrieNode {
    Map<Character, TrieNode> children = new HashMap<>();
    boolean isEnd;
}

class Trie {
    TrieNode root = new TrieNode();
    
    public void insert(String word) {
        TrieNode cur = root;
        for (char ch : word.toCharArray()) {
            cur = cur.children.computeIfAbsent(ch, k -> new TrieNode());
        }
        cur.isEnd = true;
    }
    
    public boolean search(String word) {
        TrieNode cur = root;
        for (char ch : word.toCharArray()) {
            cur = cur.children.get(ch);
            if (cur == null) return false;
        }
        return cur.isEnd;
    }
    
    public boolean startsWith(String prefix) {
        TrieNode cur = root;
        for (char ch : prefix.toCharArray()) {
            cur = cur.children.get(ch);
            if (cur == null) return false;
        }
        return true;
    }
}
```

```python
class TrieNode:
    def __init__(self):
        self.children = {}
        self.is_end = False

class Trie:
    def __init__(self):
        self.root = TrieNode()
    
    def insert(self, word: str):
        cur = self.root
        for ch in word:
            if ch not in cur.children:
                cur.children[ch] = TrieNode()
            cur = cur.children[ch]
        cur.is_end = True
    
    def search(self, word: str) -> bool:
        cur = self.root
        for ch in word:
            if ch not in cur.children:
                return False
            cur = cur.children[ch]
        return cur.is_end
    
    def starts_with(self, prefix: str) -> bool:
        cur = self.root
        for ch in prefix:
            if ch not in cur.children:
                return False
            cur = cur.children[ch]
        return True
```

### 4.3 实际场景

- **MQTT Topic 通配符匹配**：`sensor/+/temperature` 匹配 `sensor/floor1/temperature`。用 Trie 存 Topic，`+` 匹配单层任意值，`#` 匹配多层任意值。订阅时将 Topic Pattern 插入 Trie，消息发布时在 Trie 中查找匹配
- **ES 的 Term Dictionary**：倒排索引中的 Term（词条）用 FST（有限状态转换器）存储——Trie 的压缩版，共享前缀和后缀
- **搜索联想**：输入"算法" → 在 Trie 中找 `算法*` 的所有单词 → 按频率取 Top 10
- **IP 路由最长前缀匹配**：`192.168.1.0/24` 优先于 `192.168.0.0/16`

---

## 五、二叉树遍历（高频模板）

### 5.1 递归三行——遍历的骨架

```java
// 前序：根 → 左 → 右
void preorder(TreeNode root) {
    if (root == null) return;
    visit(root);
    preorder(root.left);
    preorder(root.right);
}

// 中序：左 → 根 → 右  （BST 中序 = 有序输出）
void inorder(TreeNode root) {
    if (root == null) return;
    inorder(root.left);
    visit(root);
    inorder(root.right);
}

// 后序：左 → 右 → 根  （表达式树求值、删除文件夹）
void postorder(TreeNode root) {
    if (root == null) return;
    postorder(root.left);
    postorder(root.right);
    visit(root);
}
```

### 5.2 迭代写法——用栈模拟递归

```python
# 中序迭代（面试常考）
def inorder(root):
    res, stack = [], []
    cur = root
    while cur or stack:
        while cur:              # 一直往左走到底
            stack.append(cur)
            cur = cur.left
        cur = stack.pop()       # 弹出最左节点
        res.append(cur.val)
        cur = cur.right         # 转向右子树
    return res

# 层序遍历（BFS）
from collections import deque
def level_order(root):
    if not root: return []
    res, q = [], deque([root])
    while q:
        level = []
        for _ in range(len(q)):   # 按层处理
            node = q.popleft()
            level.append(node.val)
            if node.left: q.append(node.left)
            if node.right: q.append(node.right)
        res.append(level)
    return res
```

### 5.3 实际场景

- **前序**：二叉树的序列化与反序列化——前序 + 空节点标记可唯一确定一棵树
- **中序**：BST 验证（中序序列必须严格递增）、找到 BST 中第 K 小的元素
- **后序**：删除文件夹（先递归删除所有子文件夹再删自己）、表达式树求值
- **层序**：二叉树的右视图（每层最右节点）、序列化（跟 LeetCode 的序列化格式一致）

---

B+Tree 优化的是 IO，红黑树平衡的是旋转次数，LSM Tree 极致于写入，Trie 牺牲内存换前缀匹配——四种树对应四种完全不同的优化目标。理解"为什么场景 X 选树 Y"，比会写旋转代码更重要。
