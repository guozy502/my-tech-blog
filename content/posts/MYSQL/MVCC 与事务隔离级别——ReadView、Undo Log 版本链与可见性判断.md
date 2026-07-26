---
title: "MVCC 与事务隔离级别——ReadView、Undo Log 版本链与可见性判断"
date: 2026-06-28
description: 从 InnoDB 的 Undo Log 版本链结构、ReadView 三要素的可见性判断算法、RC 与 RR 的根本差异（ReadView 生成时机）、快照读与当前读的本质区别，到幻读为什么在 RR 下是"部分解决"而非"完全解决"，拆解 MySQL MVCC 机制的完整逻辑。
tags: ["MySQL","InnoDB","MVCC","事务","隔离级别","Undo Log"]
categories: ["MYSQL"]
---

# 历史背景——ANSI SQL 标准遇到了 InnoDB

1992 年，ANSI SQL-92 标准定义了四种事务隔离级别，用"禁止哪些异常现象"来区分：READ UNCOMMITTED（允许脏读）→ READ COMMITTED（禁止脏读）→ REPEATABLE READ（禁止不可重复读）→ SERIALIZABLE（禁止幻读）。

这套定义从教学角度很清晰，但有一个致命问题：**它只描述了"异常现象"，没有规定"实现方式"**。不同数据库厂商用完全不同的机制实现了同一套隔离级别——Oracle 用基于 Undo 的 MVCC，PostgreSQL 用多版本元组，MySQL 用 MVCC + 锁的混合方案。

InnoDB 的实际行为和 SQL 标准有几个微妙的偏差：
- **RR 下快照读没有幻读，但当前读仍有幻读**（标准说 RR 不能有幻读 → InnoDB 用 Next-Key Lock 来弥补当前读的幻读）
- **InnoDB 的 RR 实际上防止了不可重复读**（标准说 RR 只保证读同一行相同，但 InnoDB 连新插入的行都挡住了）
- **RU 在 InnoDB 中其实不会脏读快照读**（因为 ReadView 机制，`SELECT` 仍然看不到未提交的数据——但 `SELECT ... FOR UPDATE` 当前读能读到）

知道"标准怎么说"和"InnoDB 实际怎么做"之间的差距，是你排查线上数据一致性问题的关键。

---

# 一、事务到底是什么？

## 1.1 从 ACID 开始

| 字母 | 含义 | 工程上的具体实现 |
|------|------|----------------|
| **A** Atomicity | 原子性 | Undo Log 提供回滚能力——事务中任何操作失败，Undo 负责反向操作 |
| **C** Consistency | 一致性 | 约束检查（NOT NULL / UNIQUE / FOREIGN KEY）+ 触发器确保数据合法 |
| **I** Isolation | 隔离性 | MVCC（快照读）+ 锁（当前读）提供不同级别的隔离 |
| **D** Durability | 持久性 | Redo Log + Doublewrite Buffer 保证已提交事务不丢失 |

**"C 不是数据库的责任"**这点很关键：约束是你 DBA 定义的，数据库只负责检查。如果你把 `age INT` 存了 -1，数据库拦不住——除非你加了 `CHECK (age >= 0)`。

## 1.2 事务 ID 的分配时机

```sql
BEGIN;   -- 这时候还没有 trx_id
SELECT * FROM users WHERE id = 1;  -- 仍然是快照读，但还没有分配 trx_id
UPDATE users SET name = 'Alice' WHERE id = 1;  -- 第一次写操作 → 分配 trx_id
COMMIT;
```

InnoDB 的事务 ID（`trx_id`）只在**事务第一次执行写操作**时分配（INSERT/UPDATE/DELETE，SELECT ... FOR UPDATE 也算写操作）。只读事务没有 trx_id——这一设计节省了 trx_id 的消耗，也意味着"只读事务不会出现在活跃事务列表中"。

---

# 二、Undo Log——MVCC 的"胶卷底片"

## 2.1 两种 Undo Log

InnoDB 的 Undo Log 有两种，用途不同：

| | Insert Undo | Update Undo |
|------|------------|------------|
| **对应操作** | INSERT | UPDATE / DELETE |
| **回滚方式** | DELETE 该行 | 反向操作（UPDATE 回原值 / DELETE → INSERT 回） |
| **事务提交后** | **立即删除**（不需要给其他事务看） | **保留**（因为其他事务的 ReadView 可能需要旧版本） |
| **Purge 线程** | 不涉及 | 后台 Purge 线程判断"没有任何事务需要了"后删除 |

为什么 Insert Undo 可以提交后立即删？因为 INSERT 的数据在提交前对其他事务不可见——没有其他事务的 ReadView 引用了它。而 UPDATE 的数据在被修改前就存在，其他活跃事务的快照需要看到"修改前的版本"。

## 2.2 Undo Log 版本链

每一行数据（在聚簇索引的叶子页中）有三个隐藏列：

| 隐藏列 | 大小 | 作用 |
|--------|------|------|
| `DB_TRX_ID` | 6B | 最后一次修改这行的事务 ID |
| `DB_ROLL_PTR` | 7B | 回滚指针——指向 Undo Log 中这行的"上一个版本" |
| `DB_ROW_ID` | 6B | 如果没有主键，用这个当聚簇索引键 |

**版本链的形成**：

```
当前行（聚簇索引页中）：
  id=1, name='C', DB_TRX_ID=300, DB_ROLL_PTR ──┐
                                                  │
  Undo Log 链表：                                 ↓
  版本2: id=1, name='B', DB_TRX_ID=200, DB_ROLL_PTR ──┐
                                                       │
                                                       ↓
  版本1: id=1, name='A', DB_TRX_ID=100, DB_ROLL_PTR → NULL（链尾）
```

每个 UPDATE 操作都在 Undo Log 中创建一个新版本，`DB_ROLL_PTR` 像链表的 next 指针一样串联起所有历史版本。这条链就是 MVCC 的"时间旅行"能力——通过沿链回溯，你可以看到这行数据在任意历史时间点的状态。

---

# 三、ReadView——快照读的"时间戳"

## 3.1 ReadView 的三要素

ReadView 是 InnoDB 实现"快照读"的核心数据结构。源码在 `read0types.h` 中定义：

```
struct ReadView {
    trx_id_t  m_low_limit_id;    // 活跃事务中最小的 trx_id
    trx_id_t  m_up_limit_id;     // 系统下一个要分配的 trx_id（= max_id + 1）
    trx_ids_t m_ids;             // 当前活跃的读写事务 ID 集合
    trx_id_t  m_creator_trx_id;  // 创建这个 ReadView 的事务的 trx_id
};
```

**通俗理解**：
- `m_ids` = "在这个时间点，有哪些事务还没提交？"
- `m_low_limit_id` = 这些未提交事务中，ID 最小的是多少？
- `m_up_limit_id` = 系统的"下一个事务 ID"——大于等于此值的都还没开始

## 3.2 可见性判断算法

当快照读需要读取一行数据时，InnoDB 调用 `changes_visible()` 函数判断"这行数据的当前版本对我可见吗？"。核心逻辑：

```
输入：这行数据的 DB_TRX_ID、当前 ReadView

① if (DB_TRX_ID == creator_trx_id):
       → 这行是我自己改的 → 可见 ✓

② if (DB_TRX_ID < m_low_limit_id):
       → 修改这行的事务在我创建 ReadView 前就提交了 → 可见 ✓

③ if (DB_TRX_ID >= m_up_limit_id):
       → 修改这行的事务在我创建 ReadView 之后才开始 → 不可见 ✗
       → 沿 DB_ROLL_PTR 回溯到上一个版本，重新判断

④ if (DB_TRX_ID is in m_ids):
       → 修改这行的事务在我创建 ReadView 时还未提交 → 不可见 ✗
       → 沿 DB_ROLL_PTR 回溯

⑤ else:
       → DB_TRX_ID 不在活跃列表中 → 已提交 → 可见 ✓
```

**本质**就一个判断：**"改这行的那个事务，在我开始看的时候提交了没有？"**

如果没有 → 沿 Undo 版本链回溯到"我出生之前"的版本 → 那个版本应该是可见的（它是某个更早提交的事务留下的）。

---

# 四、RC vs RR——ReadView 的生成时机决定一切

## 4.1 READ COMMITTED：每次快照读生成新 ReadView

```
事务 A（RC）：
  BEGIN;
  SELECT * FROM t WHERE id=1;  -- 生成 ReadView_1，读到 name='old'
  -- 此时事务 B UPDATE t SET name='new' WHERE id=1; COMMIT;
  SELECT * FROM t WHERE id=1;  -- 生成 ReadView_2，读到 name='new'
  -- 同一个事务内，两次读看到了不同结果 → 不可重复读
```

RC 下每次 `SELECT`（严格说是每次快照读）都新建 ReadView。效果是：**你总是读到最新的已提交数据**。代价是：同一事务内两次读可能不同（不可重复读）。

## 4.2 REPEATABLE READ：事务开始时生成唯一的 ReadView

```
事务 A（RR）：
  BEGIN;
  SELECT * FROM t WHERE id=1;  -- 生成 ReadView_1，读到 name='old'
  -- 此时事务 B UPDATE t SET name='new' WHERE id=1; COMMIT;
  SELECT * FROM t WHERE id=1;  -- 复用 ReadView_1，仍然读到 name='old'
  -- 同一事务内，两次读看到一样的结果 → 可重复读
```

RR 下，事务中**第一次快照读**生成的 ReadView 被整个事务期间复用。效果是：**你看到的是事务开始那一刻的数据快照**，后续其他事务的提交不影响你。

**关键的细节**：如果事务开头没有 `SELECT`，而直接 `UPDATE` 呢？
- `UPDATE` 是当前读，不是快照读 → 会读到最新的已提交版本并加锁
- 但第一次 `SELECT`（快照读）时仍然生成 ReadView，后续快照读都复用

---

# 五、快照读 vs 当前读——这是两套完全不同的机制

许多人学 MVCC 时最大的迷惑来自：**不是说 RR 下数据不会变吗？为什么我 UPDATE 的时候能读到别人刚改的值？**

答案很简单：InnoDB 有**两套读取机制**，它们走完全不同的代码路径：

| | 快照读（Snapshot Read） | 当前读（Current Read / Locking Read） |
|------|----------------------|----------------------------------|
| **SQL 类型** | 普通 `SELECT` | `SELECT ... FOR UPDATE` / `SELECT ... LOCK IN SHARE MODE` / `UPDATE` / `DELETE` / `INSERT` |
| **读取机制** | 走 ReadView → 可见性判断 → 可能回溯 Undo 版本链 | 读最新的已提交版本 → 加锁 |
| **是否加锁** | 不加锁 | 加行锁（Record Lock 等） |
| **能不能被其他事务阻塞** | 不阻塞（MVCC 无锁） | 可能被阻塞（等锁释放） |

**经典误解**：RR 下当前读仍然可能看到"不可重复的"数据——

```sql
-- 事务 A（RR）
SELECT * FROM t WHERE id=1;  -- 快照读 → 返回 'A'
-- 事务 B 偷偷修改 id=1 为 'B' 并提交
SELECT * FROM t WHERE id=1 FOR UPDATE;  -- 当前读 → 返回 'B'！（因为 B 已提交）
-- RR 下快照读仍是 'A'，但当前读看到了 'B'
```

**理解这个区别**，你才能理解为什么 RR 下需要 Next-Key Lock——它保护的不是快照读（已经有 MVCC 保证了），而是当前读（加锁读/写操作）。

---

# 六、幻读——RR 下是"部分解决"

## 6.1 幻读的准确定义

```sql
-- 事务 A（RR）
SELECT * FROM t WHERE age > 20;  -- 返回 5 行
-- 事务 B 插入一行 age=25 的新记录并提交
SELECT * FROM t WHERE age > 20;  -- 快照读：仍返回 5 行（MVCC 挡住了）
SELECT * FROM t WHERE age > 20 FOR UPDATE;  -- 当前读：返回 6 行！（幻读发生了）
```

## 6.2 为什么快照读挡住了？

RR 下 ReadView 不变 → 事务 B 插入的行的 `DB_TRX_ID` 必然 ≥ ReadView 的 `m_up_limit_id` → 不可见性判断算法拦截 → 快照读看不到新行。

## 6.3 为什么当前读挡住了？（Next-Key Lock）

**快照读不需要锁，MVCC 天然挡幻读。当前读怎么档？**

答案：Next-Key Lock（临键锁）。当 `SELECT ... FOR UPDATE WHERE age > 20` 执行时，InnoDB 对满足条件的**记录和它们之间的间隙**都加了锁——其他事务无法在这些间隙中 INSERT 新行。

然而，这只能档住"范围有间隙"的幻读。如果 `WHERE` 条件没有合适的索引，Next-Key Lock 无法有效加锁——这种情况下 RR 并不能完全解决幻读。

## 6.4 为什么 MySQL 默认 RR 而不是 RC？

这主要是历史原因——主从复制：

- MySQL 5.5 及之前，Binlog 默认是 STATEMENT 格式——记录 SQL 语句本身
- RC 下，`DELETE FROM t WHERE status=0 LIMIT 1000` 可能主从删除的行不同（因为没有 ORDER BY）
- RR + Gap Lock 保证了主从的语句式复制结果一致

对于大多数互联网应用，如果不是必须用 Gap Lock 防幻读，**RC 可能比 RR 更合适**——省掉 Gap Lock 的开销（Gap 锁不互斥但累积开销不容忽视）。

---

# 七、总结

| 概念 | 核心机制 | 一句话 |
|------|---------|--------|
| **MVCC** | Undo Log 版本链 + ReadView | 通过回溯"历史版本"实现无锁快照读 |
| **ReadView** | 记录"我出生时谁还没提交" | 唯一定义了快照读的"时间点" |
| **RC vs RR** | ReadView 的生成时机 | RC 每次生成（看最新），RR 第一次生成（看历史快照） |
| **快照读** | ReadView 可见性判断 | 走版本链，不加锁 |
| **当前读** | 读最新版本 + 加锁 | `SELECT FOR UPDATE` / `UPDATE` / `DELETE` |
| **幻读** | RR 快照读解决了，当前读还需 Gap Lock | "部分解决" |

# 延伸阅读

**Do——动手验证：**
- 开两个会话，A 用 RR，B 用 RC，观察同一条数据被 B 修改后 A 的快照读和当前读的结果差异
- `SHOW ENGINE INNODB STATUS` 查看 `TRANSACTIONS` 段，观察活跃事务列表和它们的 Undo 使用量
- 开启一个长事务不提交，观察 `information_schema.INNODB_TRX` 中的 `trx_rows_modified`

**Todo——深入方向：**
- Purge 线程的调度逻辑——如何判断一个 Undo Log "没人需要了"（`purge_sys` + `read view` 链）
- 长事务导致的 Undo 膨胀和慢查询日志的关联分析
- MySQL 8.0 的 `instant ADD COLUMN` 和 Undo Log 元数据的关系
- PostgreSQL 的 MVCC（基于 tuple 多版本 + VACUUM）与 InnoDB MVCC 的架构对比

*本文参考资料：*
- MySQL 官方文档: InnoDB Multi-Versioning / Transaction Isolation Levels
- MySQL 源码 `storage/innobase/include/read0types.h` (ReadView struct)
- Jeremy Cole, "InnoDB Transaction Isolation Levels" blog series
- 姜承尧《MySQL 技术内幕：InnoDB 存储引擎》——第 6 章（事务）
