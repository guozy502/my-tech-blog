---
title: "InnoDB 锁机制——记录锁、间隙锁、临键锁与死锁分析"
date: 2026-06-28
description: 从意向锁、记录锁、Gap Lock、Next-Key Lock 到插入意向锁，拆解 InnoDB 七种锁的加锁规则；从死锁日志的三段式阅读法到经典死锁场景的主动规避，覆盖 MySQL 锁机制的全链路。
tags: ["MySQL","InnoDB","锁","死锁","Next-Key Lock","Gap Lock"]
categories: ["MYSQL"]
---

# 历史背景——从"MySQL 只有表锁"到 InnoDB 行锁

MySQL 早期的 MyISAM 引擎只有表锁。`LOCK TABLES users WRITE` 一执行，整个表上所有读写全部阻塞——并发能力约等于零。但公平地说，2000 年代初期的大多数 Web 应用并不需要高并发写入——一个论坛的帖子写入每秒可能只有几次。

InnoDB 引入行级锁（Row-Level Locking）后，MySQL 的并发写作能力有了质的飞跃。多个事务可以同时修改同一张表的不同行，互不阻塞。但这套锁机制的代价是**复杂度的大幅增加**：锁不只是锁"行"，还要锁"行与行之间的间隙"来防止幻读——间隙锁（Gap Lock）就是 InnoDB 独有的概念，在 Oracle 或 PostgreSQL 中没有对应物。

更麻烦的是，**加锁规则不是"对查询的所有记录加排他锁"这么简单**。加的锁取决于：索引类型（唯一/非唯一）、查询条件（等值/范围）、隔离级别（RC/RR）、甚至 SQL 中是否用了 `LIMIT`。在生产环境排查死锁时，如果对这套规则没有清晰的心智模型，`SHOW ENGINE INNODB STATUS` 的输出看起来就像天书。

---

# 一、InnoDB 的七种锁速览

```sql
-- 查看当前所有锁（MySQL 8.0+）
SELECT * FROM performance_schema.data_locks\G

-- 查看锁等待关系
SELECT * FROM performance_schema.data_lock_waits\G

-- 或者用 sys schema 更友好的视图
SELECT * FROM sys.innodb_lock_waits;
```

| 锁类型 | 锁粒度 | 互斥关系 | 存在目的 |
|--------|--------|---------|---------|
| **意向共享锁（IS）** | 表级 | 与 X/IX 互斥 | 表锁只需要检查表的意向锁标志位，无需遍历所有行锁 |
| **意向排他锁（IX）** | 表级 | 与 S/IS/X 互斥 | 同上 |
| **记录锁（Record Lock）** | 行级 | 同记录的 X 锁互斥 | 锁定具体的索引记录 |
| **间隙锁（Gap Lock）** | 间隙 | **Gap 之间不互斥** | 防止其他事务在间隙中 INSERT（防幻读） |
| **临键锁（Next-Key Lock）** | 记录+前间隙 | Record 部分互斥，Gap 部分不互斥 | RR 下的默认加锁方式 |
| **插入意向锁（Insert Intention）** | 间隙 | 与 Gap Lock 互斥 | 多个事务可同时在同间隙等待插入，被 Gap Lock 阻塞 |
| **自增锁（AUTO-INC）** | 表级 | 依赖 `innodb_autoinc_lock_mode` | 保证自增 ID 的唯一性 |

**意向锁的核心价值**：假设要加表锁，如果没有意向锁，需要遍历所有行看有没有行锁——这在百万行的表上是灾难。有了意向锁，只需要检查表的 IX 标志位——O(1)。

---

# 二、记录锁（Record Lock）——锁的到底是"行"还是"索引"？

## 2.1 记录锁加在索引上

```sql
-- 创建测试表
CREATE TABLE t (
    id INT PRIMARY KEY,
    name VARCHAR(50),
    age INT,
    INDEX idx_age (age)
) ENGINE=InnoDB;

INSERT INTO t VALUES (1, 'Alice', 25), (5, 'Bob', 30), (10, 'Carol', 25);

-- 事务 A：对 id=5 加锁
BEGIN;
SELECT * FROM t WHERE id = 5 FOR UPDATE;

-- 查看锁情况
SELECT 
    lock_type, lock_mode, lock_status, lock_data
FROM performance_schema.data_locks
WHERE object_name = 't';
```

输出：
```
lock_type  | lock_mode     | lock_status | lock_data
-----------|---------------|-------------|----------
TABLE      | IX            | GRANTED     | NULL      ← 表级意向排他锁
RECORD     | X,REC_NOT_GAP | GRANTED     | 5         ← 记录锁，锁在 id=5 的聚簇索引记录上
```

**关键洞察**：锁加在**索引**上，不是加在"行"上。如果表根本没有索引，InnoDB 用什么加锁？

```sql
-- 创建一个没有索引的表，插入数据后测试
CREATE TABLE t_noindex (id INT, val VARCHAR(10)) ENGINE=InnoDB;
INSERT INTO t_noindex VALUES (1, 'a'), (2, 'b'), (3, 'c');

-- 事务 A
BEGIN;
SELECT * FROM t_noindex WHERE id = 1 FOR UPDATE;

-- 查看锁
SELECT lock_type, lock_mode, lock_data 
FROM performance_schema.data_locks
WHERE object_name = 't_noindex';
```

结果：锁住了表中所有行（因为没有索引 → 无法定位具体行 → InnoDB 只能锁全表所有聚簇索引记录 + 所有间隙）。**没有索引的表 = 所有行锁退化为表级锁**。

---

# 三、间隙锁（Gap Lock）——锁住的是"空无"

## 3.1 Gap Lock 的存在理由

Gap Lock 锁的是**索引记录之间的空隙**，目的是防止其他事务插入新记录到这些空隙中。

```sql
-- 准备数据
DELETE FROM t;
INSERT INTO t VALUES (5, 'Alice', 25), (10, 'Bob', 30), (15, 'Carol', 25);

-- 事务 A（RR 隔离级别）
BEGIN;
SELECT * FROM t WHERE id = 7 FOR UPDATE;  -- 等值查询一个不存在的记录

-- 事务 B
INSERT INTO t VALUES (6, 'Dave', 28);  -- ❌ 被阻塞！即使 id=6 和 id=7 不同
INSERT INTO t VALUES (12, 'Eve', 32);  -- ❌ 也被阻塞！
INSERT INTO t VALUES (3, 'Frank', 22);  -- ✅ 成功（不在间隙范围）
```

查看事务 A 的锁：
```sql
SELECT lock_mode, lock_data 
FROM performance_schema.data_locks
WHERE object_name = 't';
```

```
lock_mode    | lock_data
-------------|----------
X,GAP        | 5         ← 间隙锁：锁住 (5, 10) 之间的空隙
```

**等值查询一个不存在的记录 → InnoDB 锁住"该记录应该在的位置"的间隙**：`WHERE id = 7`，7 在 5 和 10 之间 → Gap Lock 锁住 (5, 10) 间隙。

## 3.2 Gap Lock 之间的特殊性质

```sql
-- 事务 A
BEGIN;
SELECT * FROM t WHERE id = 7 FOR UPDATE;  -- 锁住 (5, 10) 间隙

-- 事务 B
BEGIN;
SELECT * FROM t WHERE id = 8 FOR UPDATE;  -- ✅ 成功！也锁住 (5, 10) 间隙
-- Gap Lock 之间不互斥！两个事务可以同时持有同一个间隙的 Gap Lock

-- 事务 A
INSERT INTO t VALUES (6, 'test', 20);  -- ❌ 被阻塞！
-- 事务 B 的 Gap Lock 阻止了 A 的插入

-- 事务 B
INSERT INTO t VALUES (6, 'test', 20);  -- ❌ 也被阻塞！
-- 这就有可能导致死锁！
```

**Gap Lock 之间的不互斥是死锁的重要来源**——两个事务各自持有同一个间隙的 Gap Lock，都想往这个间隙插入，互相等待对方释放 Gap Lock → 死锁。

---

# 四、临键锁（Next-Key Lock）——RR 下的默认模式

## 4.1 什么是 Next-Key Lock？

Next-Key Lock = 记录锁（Record Lock）+ 该记录**前方**的间隙锁（Gap Lock）= "左开右闭"区间。

```sql
-- 数据：id = 5, 10, 15
-- 事务 A
BEGIN;
SELECT * FROM t WHERE id > 8 AND id < 15 FOR UPDATE;
```

加的锁（简化为区间表示）：
```
Next-Key Lock 范围：(5, 10] + (10, 15]
                    ↑           ↑
                gap+record   gap+record
```

**为什么是"前间隙"而不是"后间隙"？**
每个记录锁住它前面的空隙。下一条记录会锁住它前面的空隙（包括这条记录的位置）。这样整个索引被无缝覆盖——索引的最小值和最大值之间没有锁不到的"盲区"。

## 4.2 降级规则：什么时候 Next-Key Lock 退化为 Record Lock？

```sql
-- 规则：唯一索引 + 等值查询 + 记录存在 → 退化为 Record Lock
BEGIN;
SELECT * FROM t WHERE id = 10 FOR UPDATE;
-- 聚簇索引 id 唯一 → 只锁 id=10 这一行（无 Gap Lock）

-- 但如果等值查询的记录不存在呢？
BEGIN;
SELECT * FROM t WHERE id = 7 FOR UPDATE;
-- 退化为 Gap Lock（不是 Next-Key Lock）→ 锁住 (5, 10) 间隙
```

**为什么记录存在时可以退化？** 因为唯一索引保证同一个值只对应一行 → 不存在"别人插一个同样值的行导致幻读"的可能 → Gap Lock 是多余的。

---

# 五、加锁规则的完整推演

以下推演基于 **RR 隔离级别**（RC 下 Gap Lock 不生效，更简单）。

准备数据：
```sql
CREATE TABLE t_lock (
    id INT PRIMARY KEY,
    name VARCHAR(50),
    age INT,
    INDEX idx_age (age)
) ENGINE=InnoDB;

INSERT INTO t_lock VALUES 
    (5, 'Alice', 25), 
    (10, 'Bob', 30), 
    (15, 'Carol', 25),
    (20, 'Dave', 35);
```

## 场景 1：唯一索引等值查询 + 记录存在

```sql
-- 事务 A
BEGIN;
SELECT * FROM t_lock WHERE id = 10 FOR UPDATE;

-- 加的锁：
-- TABLE: IX
-- RECORD: X,REC_NOT_GAP on id=10（聚簇索引）
-- 结论：只锁 id=10 一行，无间隙锁
```

## 场景 2：唯一索引等值查询 + 记录不存在

```sql
-- 事务 A
BEGIN;
SELECT * FROM t_lock WHERE id = 12 FOR UPDATE;

-- 加的锁：
-- TABLE: IX  
-- RECORD: X,GAP on id=15（间隙锁在 (10, 15)）
-- 结论：锁住 (10, 15) 间隙，阻止插入 id=11,12,13,14
```

## 场景 3：唯一索引范围查询

```sql
-- 事务 A
BEGIN;
SELECT * FROM t_lock WHERE id >= 10 AND id < 20 FOR UPDATE;

-- 加的锁（推演）：
-- TABLE: IX
-- RECORD: X,REC_NOT_GAP on id=10  ← 等值命中，退化为 Record Lock
-- RECORD: X on id=15              ← Next-Key Lock (10, 15]
-- RECORD: X on id=20              ← Next-Key Lock (15, 20]
-- RECORD: X,GAP on supremum       ← 锁住 (20, +∞) 间隙
```

验证：
```sql
-- 事务 B 测试各种插入：
INSERT INTO t_lock VALUES (9, 'test', 20);   -- ❌ 阻塞（在 (5, 10] 间隙？不对，id=10 只锁了 record，实际可插入）
INSERT INTO t_lock VALUES (11, 'test', 20);  -- ❌ 阻塞（在 (10, 15] 间隙）
INSERT INTO t_lock VALUES (22, 'test', 20);  -- ❌ 阻塞（在 (20, +∞) 间隙）
INSERT INTO t_lock VALUES (3, 'test', 20);   -- ✅ 可以（在 (supremum伪记录, 5) 间隙，未被锁）
```

## 场景 4：非唯一索引等值查询

```sql
-- 事务 A
BEGIN;
SELECT * FROM t_lock WHERE age = 25 FOR UPDATE;
-- age=25 的记录有：id=5 和 id=15（两条）

-- 加的锁推演：
-- idx_age 索引上：
--   RECORD: X on (age=25, id=5)   ← Next-Key Lock
--   RECORD: X,GAP on (age=30, id=10) ← Gap Lock！锁住 (25, 30) 间隙，阻止插入新的 age=25
-- 聚簇索引上：
--   RECORD: X,REC_NOT_GAP on id=5
--   RECORD: X,REC_NOT_GAP on id=15
```

验证：为什么 idx_age 上还要锁一个 (25, 30) 的间隙？
```sql
-- 如果不锁 (25, 30)，其他事务可以：
INSERT INTO t_lock VALUES (12, 'New', 25);  -- idx_age 的 B+Tree 中，(age=25, id=12) 排在 (age=25, id=5) 之后
-- 这个新插入的 age=25 就变成了"幻行"，当前读会看到！
```

## 场景 5：LIMIT 的锁范围缩减

```sql
-- 事务 A
BEGIN;
DELETE FROM t_lock WHERE age = 25 LIMIT 1;
-- 只删一条 age=25 的记录（id=5 被删除）

-- 加的锁：
-- 比不加 LIMIT 少锁了一个 (25, 30) 的 Gap！
-- 因为 InnoDB 在找到并删除 id=5 后，检查 LIMIT 已满足 → 停止继续扫描 → 不再加后面的锁
```

**`LIMIT` 对锁的优化非常重要**：`DELETE ... LIMIT 1` 意味着"找到第一条就停"，所以后面的间隙不需要锁——这大大减少了锁冲突范围。

---

# 六、死锁分析

## 6.1 死锁日志的三段式阅读

```sql
-- 模拟一个死锁 ↓

-- 会话 A
BEGIN;
SELECT * FROM t_lock WHERE id = 10 FOR UPDATE;  -- A 持 id=10 的 X 锁

-- 会话 B
BEGIN;
SELECT * FROM t_lock WHERE id = 15 FOR UPDATE;  -- B 持 id=15 的 X 锁

-- 会话 A（继续）
INSERT INTO t_lock VALUES (12, 'New', 25);  -- A 等 B 释放 (10, 15) 的 Gap Lock → 阻塞

-- 会话 B（继续）
INSERT INTO t_lock VALUES (12, 'Other', 30); -- B 等 A 释放 (10, 15) 的 Gap Lock → 死锁！
-- ERROR 1213 (40001): Deadlock found when trying to get lock; try restarting transaction
```

查看死锁日志：
```sql
SHOW ENGINE INNODB STATUS\G
-- 滚动到 LATEST DETECTED DEADLOCK 段
```

死锁日志的结构化阅读法：

```
------------------------
LATEST DETECTED DEADLOCK
------------------------
2026-07-27 10:30:00 0x7f1234567890

*** (1) TRANSACTION:                          ← 第一个事务的身份
TRANSACTION 4212345, ACTIVE 15 sec inserting  ← 事务 ID、活跃时间、当前操作
mysql tables in use 1, locked 1
LOCK WAIT 3 lock struct(s), heap size 1136    ← 正在等锁
MySQL thread id 8, OS thread handle 12345     ← 对应 SHOW PROCESSLIST 的哪个连接
INSERT INTO t_lock VALUES (12, 'New', 25)     ← 这个事务执行的 SQL

*** (1) HOLDS THE LOCK:                       ← 这个事务持有什么锁
RECORD LOCKS space id 5 page no 4 n bits 72 
index PRIMARY of table `test`.`t_lock` 
trx id 4212345 lock_mode X locks rec but not gap
Record lock, heap no 4 PHYSICAL RECORD       ← 持有 id=10 的记录锁

*** (1) WAITING FOR THIS LOCK TO BE GRANTED:  ← 这个事务在等什么锁
RECORD LOCKS space id 5 page no 4 n bits 72 
index PRIMARY of table `test`.`t_lock` 
trx id 4212345 lock_mode X locks gap before rec insert intention waiting
Record lock, heap no 5 PHYSICAL RECORD       ← 在等 (10, 15) 间隙的插入意向锁
                                              ← 但被事务 2 的 Gap Lock 阻塞

*** (2) TRANSACTION:                          ← 第二个事务
TRANSACTION 4212346, ACTIVE 10 sec inserting
...
*** (2) HOLDS THE LOCK:
lock_mode X locks gap before rec              ← 持有 (10, 15) 的 Gap Lock！

*** (2) WAITING FOR THIS LOCK TO BE GRANTED:
lock_mode X locks gap before rec insert intention waiting  ← 也在等插入意向锁
                                              ← 被事务 1 的 Gap Lock 阻塞
                                              → 两边互相等待 → 死锁！

*** WE ROLL BACK TRANSACTION (2)              ← InnoDB 选择回滚事务 2（代价较小）
```

**阅读心法**：先看 "WAITING FOR"（它在等什么），再看 "HOLDS THE LOCK"（它持有什么），最后看另一个事务的 "HOLDS" → 形成"互相持有对方需要的资源"的闭环。

## 6.2 三个经典死锁场景

**场景 1：两个事务互抢同间隙**

```sql
-- 数据：id = 5, 10
-- T1: SELECT * FROM t WHERE id=7 FOR UPDATE;  -- 锁 (5,10) Gap
-- T2: SELECT * FROM t WHERE id=8 FOR UPDATE;  -- 锁 (5,10) Gap（Gap之间不互斥！）
-- T1: INSERT INTO t VALUES (6, ...);           -- 等 T2 释放 Gap Lock
-- T2: INSERT INTO t VALUES (7, ...);           -- 等 T1 释放 Gap Lock → 死锁！
```

**解法**：等值查询不存在的记录改走乐观处理（先查后插，不加 Gap Lock），或改用 RC 隔离级别（RC 下不加 Gap Lock）。

**场景 2：先 Update 后 Insert**

```sql
-- 数据：id = 5, 10
-- T1: UPDATE t SET name='X' WHERE id=10;       -- 持 id=10 的 X,REC_NOT_GAP
-- T2: UPDATE t SET name='Y' WHERE id=5;        -- 持 id=5 的 X,REC_NOT_GAP  
-- T1: INSERT INTO t VALUES (6, ...);           -- 等 (5,10) 间隙 → 正常等
-- T2: INSERT INTO t VALUES (6, ...);           -- 也等 (5,10) 间隙 → 互相等 → 死锁
```

**解法**：按统一顺序操作（如都按主键升序），或拆成小批量操作+捕获死锁重试。

**场景 3：共享锁升级**

```sql
-- T1: SELECT * FROM t WHERE id=5 LOCK IN SHARE MODE;  -- 持 S 锁
-- T2: SELECT * FROM t WHERE id=5 LOCK IN SHARE MODE;  -- S 锁不互斥 → 成功
-- T1: UPDATE t SET name='X' WHERE id=5;               -- S → 升 X → 等 T2 释放 S
-- T2: UPDATE t SET name='Y' WHERE id=5;               -- S → 升 X → 等 T1 释放 S → 死锁
```

**解法**：如果最终要修改，直接用 `SELECT ... FOR UPDATE`（一开始就拿 X 锁）。

## 6.3 死锁应对策略

```sql
-- 1. 应用层：死锁重试（标准做法）
int retries = 3;
while (retries-- > 0) {
    try {
        // 执行事务
        break;
    } catch (DeadlockLoserDataAccessException e) {
        if (retries == 0) throw e;
        Thread.sleep(50);  // 短暂等待后重试
    }
}

-- 2. 数据库层：监控死锁频率
SHOW ENGINE INNODB STATUS\G  -- 看 "LATEST DETECTED DEADLOCK"

-- 3. 用 pt-deadlock-logger 持续记录死锁
-- pt-deadlock-logger h=localhost --user=root --ask-pass

-- 4. 高并发下关闭死锁检测（MySQL 8.0.18+）
SET GLOBAL innodb_deadlock_detect = OFF;
-- 配合 innodb_lock_wait_timeout（默认 50s，建议 5-10s）
SET GLOBAL innodb_lock_wait_timeout = 5;
-- 关闭检测后不再自动回滚死锁，需要依赖锁等待超时
-- 适合极高并发场景（检测的 O(N²) 开销太大）
```

---

# 七、锁监控速查

```sql
-- 当前所有锁
SELECT 
    object_schema, object_name, 
    lock_type, lock_mode, lock_status, lock_data
FROM performance_schema.data_locks;

-- 当前锁等待（谁在等谁）
SELECT 
    waiting_trx_id, waiting_thread_id, 
    blocking_trx_id, blocking_thread_id
FROM performance_schema.data_lock_waits;

-- sys schema 的易读版本（谁阻塞了谁）
SELECT 
    waiting_pid AS '等待的线程',
    waiting_query AS '等待的SQL',
    blocking_pid AS '阻塞者的线程',
    blocking_query AS '阻塞者的SQL',
    wait_age AS '等待时长'
FROM sys.innodb_lock_waits;

-- 查看活跃事务
SELECT 
    trx_id, trx_state, trx_started,
    trx_rows_locked, trx_rows_modified,
    TIMESTAMPDIFF(SECOND, trx_started, NOW()) AS '运行秒数'
FROM information_schema.INNODB_TRX
WHERE trx_started < DATE_SUB(NOW(), INTERVAL 60 SECOND)  -- 运行超过 60 秒的
ORDER BY trx_started;
```

---

# 八、总结

| 场景 | 加的锁 | 关键判断 |
|------|--------|---------|
| 唯一索引等值命中 | Record Lock (无 Gap) | 唯一性保证不需要 Gap |
| 唯一索引等值不命中 | Gap Lock | 锁住"应该在的间隙" |
| 唯一索引范围查询 | Next-Key Lock 全区间 + supremum Gap | 可能锁到 max 值之后的间隙 |
| 非唯一索引等值命中 | Next-Key Lock 命中行 + 向后 Gap Lock | 非唯一性必须锁后面，防止插入相同值 |
| 非唯一索引等值不命中 | Gap Lock | 与唯一索引类似 |
| `LIMIT` 存在时 | 找到足够行后停止加锁 | 显著缩小锁范围 |

# 延伸阅读

**Do——动手验证：**
- 用 `performance_schema.data_locks` 验证本文每个加锁场景的锁输出
- 模拟两种死锁场景，阅读 `SHOW ENGINE INNODB STATUS` 的死锁日志
- 对比 RC 和 RR 下同样 SQL 的锁差异（RC 下 Gap Lock 不生效）
- 在生产库从库上用 `sys.innodb_lock_waits` 定期巡检锁等待

**Todo——深入方向：**
- InnoDB 的锁内存结构（`lock_t` struct 和 `lock_rec_t` 位图）——一个页上的多个行锁如何用位图高效表示
- MySQL 8.0 的 `NOWAIT` 和 `SKIP LOCKED` 语义——队列场景的最佳实践
- 元数据锁（MDL Lock）——`ALTER TABLE` 被 `SELECT` 阻塞的原因和排查方法
- 全局读锁（`FLUSH TABLES WITH READ LOCK`）与备份工具的一致性快照原理

*本文参考资料：*
- MySQL 官方文档: InnoDB Locking
- 何登成（淘宝丁奇）,"MySQL 加锁处理分析"系列
- Bill Karwin《SQL Antipatterns》——第 7 章（并发控制）
- MySQL 8.0 Release Notes: `SKIP LOCKED` / `NOWAIT`
