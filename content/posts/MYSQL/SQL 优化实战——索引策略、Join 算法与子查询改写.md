---
title: "SQL 优化实战——索引策略、Join 算法与子查询改写"
date: 2026-06-28
description: 从联合索引的列序设计与索引失效场景、三种 Join 算法（NLJ/BNLJ/Hash Join）的触发条件、子查询到 Semi-Join 的自动改写、到大 offset 分页的性能灾难与游标分页解法，覆盖 SQL 优化的核心方法论。
tags: ["MySQL","SQL优化","索引","Join","子查询"]
categories: ["MYSQL"]
---

# 历史背景——SQL 优化是"艺术"还是"工程"？

早期的数据库调优靠 DBA 的经验和直觉——"这个查询慢了？加个索引看看。"这种方式在数据量小的时候勉强有效，但当表涨到几千万行、Join 七八张表时，凭直觉的优化方式完全失灵。

MySQL 5.6 之后，优化器变得足够智能，OPTIMIZER_TRACE 可以告诉你它为什么选 A 不选 B。SQL 优化从"玄学"逐步变成了"工程"——你可以系统性地诊断瓶颈、评估索引设计、改写查询结构。但有一个前提：你必须理解"优化器能做什么"和"它做不了什么"之间的边界。

本文的优化方法论基于一个原则：**先确定瓶颈在哪一层（索引/Join/排序/网络）→ 再用具体手段解决那一层的问题**。而不是上来就改 SQL 语句。

---

# 一、索引策略——创建什么索引，为什么它能用上

## 1.1 联合索引列顺序的设计方法

假设有一张订单表，最常见的查询是：
```sql
SELECT * FROM orders 
WHERE user_id = ? AND status = ? 
ORDER BY created_at DESC 
LIMIT 20;
```

**索引设计的思考过程**：

```
第一步：列出 WHERE 条件中的等值列和范围列
  等值: user_id, status
  范围: 无（created_at 在 ORDER BY 中）

第二步：确定索引列的顺序
  ① 等值列在前 → (user_id, status)
  ② ORDER BY 的列紧随等值列 → (user_id, status, created_at)
  ③ 如果 SELECT 列少且固定，考虑把 SELECT 列也加进去 → 覆盖索引

最终索引: (user_id, status, created_at)
```

验证：
```sql
EXPLAIN SELECT * FROM orders WHERE user_id = 123 AND status = 'paid' ORDER BY created_at DESC LIMIT 20;
-- key_len = 4(user_id) + 16(status) + 5(created_at) → 三列全部生效
-- Extra = NULL → 没有 Using filesort！（索引已排序）
```

**一个常见的反例**：
```sql
-- 索引: (status, user_id, created_at) ← 把区分度低的列放在前面
-- 查询: WHERE user_id = 123 AND status = 'paid'

-- 效果: status='paid' 占了 90% 的行，索引第一个列就过滤掉 90% 的数据
--       但接下来只能用 user_id 做二次过滤 → 索引效果打折
--
-- 对比 (user_id, status, created_at):
--       user_id = 123 只占 0.1% 行 → 马上定位到精确范围 → 效果极好

-- 经验法则：区分度高的列放前（等值列优先的前提下）
```

## 1.2 索引失效的典型场景

```sql
-- 索引: (name, age, created_at)

-- ❌ 失效 1: 对索引列做函数操作
SELECT * FROM users WHERE DATE(created_at) = '2026-01-01';
-- 优化器看不到索引列 → 走全表扫描
-- ✅ 改为范围查询:
SELECT * FROM users WHERE created_at >= '2026-01-01' AND created_at < '2026-01-02';

-- ❌ 失效 2: 隐式类型转换
SELECT * FROM users WHERE phone = 13800138000;  -- phone 是 VARCHAR
-- MySQL 把 phone 列转换为数字再比较 → 索引失效！
-- ✅ 改为:
SELECT * FROM users WHERE phone = '13800138000';

-- ❌ 失效 3: LIKE 前缀模糊
SELECT * FROM users WHERE name LIKE '%Alice%';
-- 开头是通配符 → B+Tree 无法确定起点 → 索引失效（走全表扫描或全索引扫描）
-- ✅ 如果是后缀模糊:
SELECT * FROM users WHERE name LIKE 'Alice%';
-- B+Tree 可以定位到 'Alice' 开头的键 → 索引有效

-- ❌ 失效 4: OR 条件中有一个无索引列
SELECT * FROM users WHERE id = 1 OR age = 25;
-- id 有主键索引，age 没有索引 → 优化器可能选择全表扫描
-- ✅ 改为 UNION:
SELECT * FROM users WHERE id = 1
UNION
SELECT * FROM users WHERE age = 25;
-- 每个子查询都能高效用各自的索引

-- ❌ 失效 5: NOT IN / != 在大范围中出现
SELECT * FROM users WHERE status != 'deleted';
-- status 只有 3 个值: active/inactive/deleted
-- != 'deleted' = active + inactive = 95% 的数据 → 走索引不如走全表
-- 优化器判断回表成本 > 全表扫描 → 放弃索引
```

## 1.3 冗余索引的检测

```sql
-- 检查冗余索引（MySQL 内置方法）
SELECT * FROM sys.schema_redundant_indexes\G
-- 例: 
-- table: orders
-- redundant_index: idx_user_status (user_id, status)
-- dominant_index:  idx_user_status_time (user_id, status, created_at)
-- → idx_user_status 是 idx_user_status_time 的前缀 → 冗余 → 可删

-- 检查从未使用的索引
SELECT * FROM sys.schema_unused_indexes;
-- 注意：这个结果从 Performance Schema 采集，P_S 重启后数据重置
-- 建议至少运行一个月后再据此做删除决策
```

---

# 二、三种 Join 算法——什么时候 Join 会爆炸

## 2.1 Index Nested-Loop Join（NLJ）——有索引时的最佳路径

```sql
-- INLJ 的执行逻辑（伪代码）:
for each row in 驱动表:
    用该行的 Join 列值 → 通过索引查被驱动表 → 拿到匹配行
```

```sql
-- 检查 Join 是否走了 INLJ
EXPLAIN SELECT * FROM orders o JOIN users u ON o.user_id = u.id;
-- users（被驱动表）:
--   type: eq_ref  ← 说明走了主键索引关联，每行只匹配一行
--   key: PRIMARY
--
-- 这是 Join 的最好情况
```

**INLJ 的核心要求**：被驱动表的 Join 列有索引。

## 2.2 Block Nested-Loop Join（BNLJ）——无索引且不能用 Hash Join

```sql
-- BNLJ 的执行逻辑:
-- 驱动表数据分块(每次 join_buffer_size 大小) → 缓存到 join buffer
-- 每块数据一次全表扫描被驱动表做匹配

-- 例子：被驱动表没有索引
EXPLAIN SELECT * FROM orders o JOIN order_log l ON o.id = l.order_id;
-- order_log:
--   type: ALL
--   Extra: Using join buffer (Block Nested Loop)
-- ← 性能差！每次全表扫被驱动表匹配一块驱动表数据
```

**join_buffer_size 的影响**：
```sql
SHOW VARIABLES LIKE 'join_buffer_size';  -- 默认 256KB
-- 调大 → BNLJ 每块缓存更多驱动表行 → 减少全表扫描轮次
-- 但不建议超过 4MB（内存占用 × 同时执行的查询数）
SET SESSION join_buffer_size = 4194304;  -- 4MB
```

## 2.3 Hash Join（MySQL 8.0.18+）——无索引的大 Join 救星

```sql
-- Hash Join 的执行逻辑:
-- ① 将驱动表的所有行加载到内存 → 建哈希表 (Join 列 → 行数据)
-- ② 一次扫描被驱动表 → 用 Join 列哈希查找 → O(M+N)

EXPLAIN FORMAT=TREE
SELECT * FROM orders o JOIN order_detail d ON o.id = d.order_id;
-- 输出:
-- -> Inner hash join (d.order_id = o.id)  (cost=... rows=...)
--    -> Table scan on d
--    -> Hash
--       -> Table scan on o
```

Hash Join vs BNLJ 的性能差异：
```sql
-- 场景: 两张 100 万行的表 Join，被驱动表无索引
-- BNLJ: join_buffer=256KB → 驱动表切换 ~3900 块 → 3900 次全表扫被驱动表 = 灾难
-- Hash Join: 驱动表建哈希表(内存) → 1 次全表扫被驱动表 = 可控

-- MySQL 8.0 自动选择 BNLJ 还是 Hash Join（等值 Join + 无索引 → Hash Join）
-- 但如果 Join 条件包含范围比较 → 只能 BNLJ
```

## 2.4 驱动表的选择

```sql
-- 查看优化器的 Join Order
EXPLAIN FORMAT=TREE SELECT ... ;
-- 缩进最深的表 = 驱动表（最先读）

-- 如果想强制指定驱动表:
SELECT * FROM users u STRAIGHT_JOIN orders o ON u.id = o.user_id;
-- users 一定是驱动表（STRAIGHT_JOIN = 按书写顺序 Join）
```

**驱动表选择的代价差异**：
```sql
-- users: 1000 行, orders: 1000 万行
-- users 驱动: 1000 次 INDEX lookup → 1000 次回表 → 可控
-- orders 驱动: 1000 万次 INDEX lookup → 1000 万次回表 → 慢两个数量级

-- "小表驱动大表"的真谛: drive_rows × log_fanout(driven_rows) 最小化
```

---

# 三、子查询改写——优化器能帮你多少？

## 3.1 IN 子查询的 Semi-Join 优化

```sql
-- MySQL 5.5 时代（优化器不会自动改写）
SELECT * FROM orders WHERE user_id IN (SELECT id FROM users WHERE status = 'active');
-- 执行: 先跑子查询 → 结果存临时表 → 外层用 IN 查临时表
-- 问题: 两个独立查询，无索引关联

-- MySQL 5.6+（Semi-Join 自动改写）
-- 优化器内部改写为:
-- SELECT o.* FROM orders o SEMI JOIN users u ON o.user_id = u.id AND u.status = 'active'
-- 效果: 直接走 Join 流程，用上索引！
```

验证 Semi-Join 是否生效：
```sql
EXPLAIN SELECT * FROM orders WHERE user_id IN (SELECT id FROM users WHERE status = 'active');
-- 如果 Extra 中有 "Using where; FirstMatch(orders)" → Semi-Join 生效
-- 如果没有 → 子查询被物化为临时表 → 检查 optimizer_switch
```

## 3.2 EXISTS vs IN 的现代观点

在 MySQL 5.5 前，`EXISTS` 通常比 `IN` 快（因为 `IN` 会先算子查询）。在 MySQL 8.0，优化器通常能自动互转：
```sql
-- 这两个查询在现代 MySQL 上等效:
SELECT * FROM orders o WHERE EXISTS (SELECT 1 FROM users u WHERE u.id = o.user_id);
SELECT * FROM orders o WHERE o.user_id IN (SELECT id FROM users);
-- 优化器都会选择最优执行路径，不需要手动改写
```

## 3.3 Derived Table——派生表的 Merge 优化

```sql
-- MySQL 5.6 前:
SELECT * FROM 
  (SELECT user_id, COUNT(*) AS cnt FROM orders GROUP BY user_id) AS tmp
WHERE tmp.cnt > 10;
-- 子查询结果先物化为临时表（无索引）→ 外层全表扫临时表

-- MySQL 5.7+ derived_merge:
-- 优化器自动合并到外层:
-- SELECT user_id, COUNT(*) AS cnt FROM orders GROUP BY user_id HAVING cnt > 10;
-- 消除临时表！
```

```sql
-- 查看 derived_merge 是否启用
SELECT @@optimizer_switch LIKE '%derived_merge=on%';  -- 应该返回 1

-- 某些情况下即使开启也不会 merge（GROUP BY + LIMIT 同在一个子查询中等）
```

---

# 四、大 offset 分页——越翻页越慢的根因和根治

## 4.1 为什么 LIMIT 1000000, 20 会慢？

```sql
SELECT * FROM orders ORDER BY id LIMIT 1000000, 20;
-- MySQL 必须扫描 1000020 行，扔掉前 1000000 行，只返回最后 20 行
-- 扔掉的 1000000 行不是免费的——它们也需要被读出来、比较、丢弃
```

## 4.2 三种解法

**方案 1：游标分页（推荐）**
```sql
-- 上一页返回的最大 id 是 1234567
SELECT * FROM orders WHERE id > 1234567 ORDER BY id LIMIT 20;
-- 走主键索引 → 定位到 1234567 之后 → 扫描 20 行 → O(logN + 20)
-- 代价：不能跳页（用户只能"下一页"不能"第 100 页"）
```

**方案 2：覆盖索引 + 延迟关联**
```sql
-- 分两步:
-- ① 子查询只取主键（覆盖索引，不回表）
-- ② 再用主键关联获取完整数据
SELECT o.* FROM orders o
INNER JOIN (
    SELECT id FROM orders 
    ORDER BY id LIMIT 1000000, 20
) AS tmp ON o.id = tmp.id;
-- 子查询: type=index, Extra=Using index（只扫描索引，不回表！速度远快于全表扫描）
-- 关联: type=eq_ref（主键关联，O(1)）
```

性能测试（1000 万行表）:
```sql
-- 传统 LIMIT:          1000000,20 → ~800ms
-- 覆盖索引+延迟关联:   1000000,20 → ~200ms
-- 游标分页 WHERE id>:  1000000,20 → ~1ms
```

**方案 3：业务层限制深度分页**
```java
// 产品层面解决：只显示前 100 页
if (page > 100) {
    throw new IllegalArgumentException("不支持超过 100 页的分页，请使用筛选条件");
}
```

---

# 五、总结

| 问题 | 诊断方式 | 解法 |
|------|---------|------|
| 索引失效 | EXPLAIN type=ALL | 修正 SQL（避免函数包裹/隐式转换/LIKE前缀） |
| Join 慢 | EXPLAIN Extra: Using join buffer | 给被驱动表加索引（走 NLJ）或升级 8.0（走 Hash Join） |
| 子查询慢 | EXPLAIN 看是否 semi-join | 确认 `derived_merge=on` / `semijoin=on` |
| 分页大 offset 慢 | 时间随页码线性增长 | 游标分页或覆盖索引+延迟关联 |
| GROUP BY 慢 | Extra: Using temporary; Using filesort | 索引覆盖 GROUP BY 列顺序 |

# 延伸阅读

**Do——动手练习：**
- 写一个 3 表 Join 查询，分别不建索引、只建一个索引、全建索引，对比 EXPLAIN
- 用 `EXPLAIN FORMAT=TREE` 查看 Join Order 和使用的 Join 算法
- 对比同一个分页查询下，传统 LIMIT vs 覆盖索引+延迟关联 的 `rows` 差异

**Todo——深入方向：**
- MySQL 8.0 的 CTE（WITH 递归查询）——组织架构树、BOM 展开等嵌套数据查询
- `optimizer_switch` 的 `materialization` vs `semijoin` 开关对子查询性能的影响
- 反范式化（Denormalization）在查询优化中的位置——什么时候应该打破范式获取性能

*本文参考资料：*
- 《高性能 MySQL》（第 4 版）——第 6-7 章
- MySQL 官方文档: Optimizing SELECT Statements
- 知数堂（叶金荣）：MySQL 优化实战系列
