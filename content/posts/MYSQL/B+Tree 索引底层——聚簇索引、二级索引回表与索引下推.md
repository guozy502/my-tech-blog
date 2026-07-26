---
title: "B+Tree 索引底层——聚簇索引、二级索引回表与索引下推"
date: 2026-06-28
description: 从 InnoDB 页结构的 Page Directory 槽位二分查找、聚簇索引与二级索引的回表机制、ICP 下推和 MRR 批量回表的优化原理到页分裂与页合并的触发条件，拆解 MySQL B+Tree 索引从页内查询到跨页查询的完整链路。
tags: ["MySQL","InnoDB","索引","B+Tree","执行计划"]
categories: ["MYSQL"]
---

# 历史背景——为什么数据库索引都长成 B+Tree？

1970 年，Bayer 和 McCreight 发明了 B-Tree，目标很明确：**在磁盘上高效地查找、插入、删除数据**。磁盘比内存慢几个数量级，核心指标不是比较次数，而是"磁盘 IO 次数"。

二叉树（如 AVL、红黑树）每个节点存一个 key + 两个子节点指针，树的高度 = log₂(N)。1000 万条数据 = 约 24 层 = 24 次磁盘 IO——不可接受。

B-Tree 的改进：每个节点存几十到几百个 key + 等量的子节点指针。高度 = log₁₀₀(N) ≈ 3-4 层。B+Tree 进一步优化：**只在叶子节点存数据**，非叶子节点只存 key 做"路标"——非叶子节点能装更多 key，树更矮。叶子节点之间用**链表连接**——范围查询时找到起点后沿链表走就行，不用返回父节点。

这就是为什么所有主流关系型数据库（MySQL、PostgreSQL、Oracle）的默认索引都是 B+Tree。不是因为它"最好"，而是因为它是"磁盘 IO 次数最优解"。

---

# 一、InnoDB 的页（Page）结构——16KB 里的乾坤

## 1.1 页是 IO 的最小单位

InnoDB 读写磁盘从来不是"读几行数据"，而是"读一个 16KB 的页"。即使你只查一条记录，也是把包含这条记录的整个页加载到 Buffer Pool。

```
┌─────────────────── 16KB Page ───────────────────┐
│ File Header (38B)         页类型、校验和、LSN     │
│ Page Header (56B)         槽数量、记录数、层级    │
│ Infimum + Supremum        最小/最大伪记录          │
│ User Records              实际的行数据（从底往上长）│
│ Free Space                空闲空间（中间）         │
│ Page Directory (槽数组)   每组 4-8 条记录一个槽     │
│ File Trailer (8B)         页尾校验和 + LSN        │
└──────────────────────────────────────────────────┘
```

**Infimum 和 Supremum** 是两个虚拟的"哨兵记录"。Infimum 表示"比所有真实记录小"，Supremum 表示"比所有真实记录大"。它们在页内形成了一个**逻辑链表**：Infimum → 记录1 → 记录2 → ... → Supremum。这个链表是按主键顺序排列的单向链表。

## 1.2 Page Directory——页内二分查找的秘密

如果页内有 100 条记录，查找一条记录不能从头扫到尾（O(N)）。Page Directory 解决的就是"如何在页内快速定位"：

- 每 **4-8 条记录**分配一个**槽（slot）**
- 每个槽指向这组记录中的**最大记录**
- 查找时：先对槽数组做**二分查找**（槽数通常几十个），找到目标所在的槽
- 然后在该槽指向的记录开始顺序扫描（最多 4-8 条）

这就是为什么 `EXPLAIN` 的 `type=ref` 只需要几次 IO——第一次 IO 把页加载到内存，页内二分找到数据。

---

# 二、聚簇索引——数据本身也是索引

## 2.1 什么是聚簇索引？

"聚簇"的意思是：**数据按照索引键的顺序物理存储**。在 InnoDB 中，聚簇索引 = 主键索引 = B+Tree 的叶子节点存储了**完整的行数据**。

```
B+Tree 结构（聚簇索引）
    
            [Key≤100 | Key≤200 | Key≤300]  ← 非叶子节点（只存主键值+子页指针）
            /          |          \
    [1..100]        [101..200]      [201..300]  ← 叶子节点
    完整行数据       完整行数据        完整行数据
      ↓                ↓               ↓
    链表 ←→ 链表 ←→ 链表（叶子之间双向连接）
```

**如果没有显式定义主键，InnoDB 会怎么做？**
1. 找第一个 `UNIQUE NOT NULL` 索引，用它当聚簇索引
2. 如果没有这样的索引 → InnoDB 自动创建一个 6 字节的隐藏列 `ROW_ID`，单调递增
3. 这隐藏列对用户透明，但对性能有影响——额外的 6 字节开销 + 无法人为控制

## 2.2 为什么推荐自增主键？

**自增主键 = 顺序插入**。新记录的主键总是大于已有记录 → 每次插入都加在 B+Tree 的最右边叶子节点 → 页满了才分裂，且分裂也很简单（把一半数据移到新页）。

**UUID 主键 = 随机插入**。新记录的主键值随机分布在整个 B+Tree 中 → 每次插入可能落在任意位置 → 页分裂频繁发生 → 碎片化 + 页利用率低（可能只有 50-75%）。更重要的是：聚簇索引的叶子节点存完整行数据，页分裂的代价很高（移动真实的行数据）。

此外，**主键越大，二级索引越大**——因为二级索引的叶子节点存储的是主键值。BIGINT(8B) pk INT(4B)，一个 1000 万行的表，仅二级索引区别就差 40MB。

---

# 三、二级索引——回表的代价

## 3.1 二级索引存了什么？

二级索引（普通索引、唯一索引）的 B+Tree 中：

- **非叶子节点**：存索引列的值 + 子页指针
- **叶子节点**：存索引列的值 + **主键值**（不是整行数据！）

```
二级索引 B+Tree（以 age 列为例）

    [age≤25 | age≤30]
    /        \
[20:pk=100] [20:pk=105] [22:pk=200] ...  ← 叶子存 (age, 主键)
```

## 3.2 回表是什么？

```sql
SELECT * FROM users WHERE age = 25;
```

1. 在 `age` 二级索引中找到所有 `age=25` 的记录 → 拿到对应的主键值列表
2. 拿着这些主键值 → 回到聚簇索引 → 查出完整的行数据

步骤 2 就是**回表**——一次额外的随机 IO。如果 `age=25` 匹配了 1000 个主键，就是 1000 次回表 = 1000 次随机 IO。

## 3.3 覆盖索引——消除回表

```sql
-- 不覆盖：需要回表取 *
SELECT * FROM users WHERE age = 25;

-- 覆盖：查询的列全在索引中，无需回表
SELECT age, id FROM users WHERE age = 25;
-- age 在索引列中，id 是主键（也在叶子节点中）→ 不回表
```

`EXPLAIN` 中 `Extra: Using index` 就是"覆盖索引"的标志——这是性能最好的查询模式。

**联合索引的覆盖技巧**：把 SELECT 列也加进索引

```sql
-- 查询：SELECT name, age FROM users WHERE age = 25 ORDER BY created_at

-- 索引设计：(age, created_at, name)
-- age 走等值匹配 → created_at 走排序 → name 覆盖（不回表）
-- Extra: Using index
```

---

# 四、最左前缀——联合索引的核心规则

## 4.1 联合索引的排序机制

联合索引 `(a, b, c)` 的 B+Tree 中，数据先按 `a` 排序，`a` 相同按 `b` 排序，`b` 相同按 `c` 排序。类似电话簿：先按姓排，同姓按名排。

## 4.2 什么情况下能用上索引？

```sql
-- 索引 (a, b, c)

-- ✅ 等值 a → 可以用索引
SELECT * FROM t WHERE a = 1;

-- ✅ 等值 a + 等值 b → 可用索引的前两列
SELECT * FROM t WHERE a = 1 AND b = 2;

-- ✅ 等值 a + 范围 b → 可用 (a, b) 两列
SELECT * FROM t WHERE a = 1 AND b > 2;

-- ❌ 跳过了 a → 索引用不上（因为索引按 a 排序，没有 a 就没法定位）
SELECT * FROM t WHERE b = 2;

-- ❌ 等值 a + 等值 c（跳过了 b）→ 只能用 a（c 不按顺序了）
SELECT * FROM t WHERE a = 1 AND c = 3;

-- ✅ 范围 a + 等值 b → 只能用 a（a 是范围后，b 在这个范围内不保证有序）
SELECT * FROM t WHERE a > 1 AND b = 2;
```

**最左前缀的本质**：索引列的顺序决定了"走到哪一列之后就不再有序了"。一旦遇到范围查询（`>`/`<`/`BETWEEN`），后面的列就不再有顺序保证。

## 4.3 联合索引的列顺序怎么设计？

三条经验法则（按优先级排序）：

1. **等值条件的列放前面，范围条件的列放后面**
2. **区分度高的列放前面**（但不是绝对的——范围条件即使区分度高也要放后面）
3. **覆盖查询需求**（把 SELECT 中需要的列也放进索引，消除回表）

```sql
-- 最常见的查询：WHERE user_id = ? AND status = ? ORDER BY created_at DESC LIMIT 20
-- 索引设计：(user_id, status, created_at)
-- user_id 等值 → status 等值 → created_at 排序（索引中已经有序，不需要 filesort）
```

---

# 五、ICP——把过滤推给引擎层

## 5.1 ICP 之前的问题

MySQL 5.6 之前，索引的过滤逻辑是这样的：

```
引擎层：用索引找出所有符合索引条件的行 → 返回给 Server 层
Server 层：用 WHERE 条件中的其余条件（索引用不上的部分）再过滤
```

问题：引擎层返回了太多行给 Server 层，其中很多在 Server 层被丢弃——白白传输数据。

## 5.2 ICP 做了什么？

**Index Condition Pushdown**：把 WHERE 条件中**能用索引前导列判断的部分**下推到引擎层，在引擎层多做一层过滤。

```sql
-- 索引：(name, age)
SELECT * FROM users WHERE name LIKE '张%' AND age = 25;
```

传统方式：引擎层用 `name LIKE '张%'` 找到所有姓张的人 → 返回 Server → Server 再筛 `age=25`。

ICP 方式：引擎层找到姓张的记录后，**在引擎层直接检查 `age=25`**（因为 `age` 在索引中！）→ 只返回同时满足两个条件的行给 Server。

`EXPLAIN` 中 `Extra: Using index condition` 就是 ICP 生效的标志。

---

# 六、MRR——把随机回表变顺序

## 6.1 回表的随机 IO 问题

```sql
-- 索引：(age)
-- 聚簇索引按 id 排序，数据物理存储也是按 id 顺序
SELECT * FROM users WHERE age = 25;
-- age=25 的主键列表：[1003, 56, 8902, 247, ...]
-- 不按顺序 → 回表时磁盘磁头来回跳 → 随机 IO 的寻道时间累积
```

## 6.2 MRR 的优化

**Multi-Range Read**：
1. 把 age=25 匹配到的所有主键值收集起来
2. **排序**
3. 按聚簇索引的物理顺序批量回表

排序后：`[56, 247, 8902, 1003, ...]` → 回表时磁头从低到高顺序走 → 接近顺序 IO。

`EXPLAIN` 中 `Extra: Using MRR` 是 MRR 生效的标志。

```bash
# 开启 MRR（默认基于成本评估）
optimizer_switch='mrr=on,mrr_cost_based=off'
# 设为 off 强制启用（不计成本），但只在优化器"算错了"时用
```

---

# 七、页分裂与页合并——索引维护的代价

## 7.1 页分裂

当新记录插入时，如果它所属的页已满：
1. 分配一个新页
2. 把旧页的一半记录移到新页（对半分裂）
3. 更新父节点的指针（"我多了一个子节点"）

**随机插入（UUID 主键）的后果**：每次插入都可能触发分裂，分裂后页利用率低（每次剩一半空），磁盘碎片不断积累。

## 7.2 页合并

当两个相邻页的数据总量降到某一阈值以下（`MERGE_THRESHOLD` 默认 50%），InnoDB 将两页合并为一块。DELETE 或 UPDATE（缩短行）都可能导致页利用率降低。

```sql
-- 查看表的碎片率
SELECT TABLE_NAME, 
       DATA_FREE / 1024 / 1024 AS fragmented_mb
FROM information_schema.TABLES 
WHERE TABLE_SCHEMA = 'mydb' AND DATA_FREE > 0;

-- 碎片整理 = 重建表
OPTIMIZE TABLE users;  -- = ALTER TABLE ... ENGINE=InnoDB（重建）
```

`OPTIMIZE TABLE` 本质是创建一个新的紧凑表 → 把旧表数据逐行拷贝过去 → 原子 rename。这个过程会锁表（根据算法不同，可能用 `ALGORITHM=INPLACE` 减少阻塞时间）。

---

# 八、总结

| 概念 | 一句话 | EXPLAIN 标志 |
|------|--------|------------|
| **聚簇索引** | 叶子节点存完整行，按主键物理排序 | — |
| **二级索引** | 叶子节点只存主键值，需要回表 | `type=ref` |
| **覆盖索引** | 查询列全在索引中，不回表 | `Extra: Using index` |
| **最左前缀** | 遇到范围查询后，后面的列不保证有序 | `key_len` 看实际用了多少字节 |
| **ICP** | WHERE 条件推到引擎层过滤 | `Extra: Using index condition` |
| **MRR** | 回表前主键排序，随机 IO 变顺序 IO | `Extra: Using MRR` |
| **页分裂** | 页满了对半分，随机插入导致碎片 | `DATA_FREE > 0` |

# 延伸阅读

**Do——动手验证：**
- 创建 `(a, b, c)` 联合索引，用 `key_len` 判断不同 WHERE 条件实际用了几列
- 对比 `SELECT * FROM t WHERE age=25` vs `SELECT id, age FROM t WHERE age=25` 的 `Extra` 差异
- 故意插入 UUID 主键，用 `information_schema.TABLES` 的 `DATA_FREE` 观察碎片增长

**Todo——深入方向：**
- InnoDB 的 FST（Fast Index Creation）——CREATE INDEX 时内部的临时索引构建流程
- 降序索引（MySQL 8.0+）的内部实现——`DESC` 索引真的倒着存吗？
- 空间索引（SPATIAL INDEX）的 R-Tree 与 B+Tree 的对比
- 全文索引（FULLTEXT）的倒排索引结构与 `ngram` 分词器

*本文参考资料：*
- MySQL 官方文档: InnoDB Indexes / CREATE INDEX Statement
- 姜承尧《MySQL 技术内幕：InnoDB 存储引擎》——第 4-5 章
- Jeremy Cole, "B+Tree Index Structures in InnoDB"
- 《高性能 MySQL》（第 4 版）——第 5 章（索引）
