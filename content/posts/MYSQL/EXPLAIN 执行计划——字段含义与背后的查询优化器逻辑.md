---
title: "EXPLAIN 执行计划——字段含义与背后的查询优化器逻辑"
date: 2026-06-28
description: 从 EXPLAIN 每个字段的深度解读（type/rows/key_len/Extra/filtered）、EXPLAIN ANALYZE 的实际执行数据、到 OPTIMIZER_TRACE 追踪优化器决策过程，拆解 MySQL 查询优化器的成本模型与索引选择逻辑。
tags: ["MySQL","EXPLAIN","执行计划","优化器","索引"]
categories: ["MYSQL"]
---

# 历史背景——CBO 的"神谕"困境

MySQL 的查询优化器是典型的 CBO（Cost-Based Optimizer）——基于成本的优化器。它根据统计信息（索引基数、行数估算）计算每条可能的执行路径的成本，选成本最低的。

但 CBO 有一个"神谕困境"：它依赖统计信息做出判断，而统计信息是**采样估算**的，不是精确的。当统计信息不准时，优化器可能选择了一条成本看起来低、但实际执行巨慢的路径。更糟糕的是，优化器的成本模型是简化过的——它假设磁盘 IO 的成本和 CPU 成本是固定的权重，但在 SSD vs HDD、单机 vs 云盘之间，这个权重差异巨大。

所以 **EXPLAIN 是你的眼睛，不是你的答案**。它能告诉你优化器选择了什么执行计划，但不能直接告诉你这个计划为什么慢、为什么不选另一个索引。要回答这些问题，你还得理解 OPTIMIZER_TRACE（看优化器的思考过程）和实际执行数据（EXPLAIN ANALYZE）。

---

# 一、EXPLAIN 的完整字段

```sql
-- 基础用法
EXPLAIN SELECT * FROM users WHERE id = 1;

-- 查看更详细的信息（8.0.18+）
EXPLAIN FORMAT=TREE SELECT * FROM users u JOIN orders o ON u.id = o.user_id;

-- JSON 格式（包含成本数据）
EXPLAIN FORMAT=JSON SELECT * FROM users WHERE id = 1;

-- 实际执行分析（8.0.18+）
EXPLAIN ANALYZE SELECT * FROM users WHERE id = 1;
```

典型输出：
```sql
mysql> EXPLAIN SELECT * FROM users WHERE id = 1\G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: users
   partitions: NULL
         type: const
possible_keys: PRIMARY
          key: PRIMARY
      key_len: 4
          ref: const
         rows: 1
     filtered: 100.00
        Extra: NULL
```

---

# 二、type——访问类型（从垃圾到完美）

这是 EXPLAIN 中最重要的一个字段。它描述了 MySQL 以什么方式访问数据：

```
从最差到最好：
  ALL → index → range → index_subquery → index_merge → ref → eq_ref → const/system → NULL
```

## ALL（全表扫描）——你的敌人

```sql
EXPLAIN SELECT * FROM users WHERE name = 'Alice';
-- 如果 name 列没有索引：
+------+------+-------+------+------+-------------+
| type | key  | rows  | Extra                     |
+------+------+-------+------+------+-------------+
| ALL  | NULL | 10000 | Using where               |
+------+------+-------+------+------+-------------+
-- type=ALL：扫描了全表每一行
-- rows=10000：优化器估算 10000 行
-- Extra=Using where：在 Server 层做 WHERE 过滤
```

## index（索引全扫描）——比 ALL 略好

```sql
-- 假设 (name, age) 是联合索引
EXPLAIN SELECT name FROM users ORDER BY name;
+------+-------+------+-------------+
| type | key   | rows | Extra           |
+------+-------+------+-------------+
| index | idx_name | 10000 | Using index |
+------+-------+------+-------------+
-- type=index：扫描了整个索引（而不是全表），但因为索引比数据小，IO 更少
-- Extra=Using index：覆盖索引，不需要回表
```

## range（范围扫描）——起码用上了索引前缀

```sql
EXPLAIN SELECT * FROM users WHERE id > 100 AND id < 200;
+------+---------+------+------+-------------+
| type | key     | rows | Extra                     |
+------+---------+------+------+-------------+
| range | PRIMARY | 100  | Using where               |
+------+---------+------+------+-------------+
-- type=range：通过索引定位到范围的起点，然后沿索引扫描到终点
-- 常见触发：> / < / BETWEEN / IN / LIKE 'prefix%'
```

## ref（非唯一索引等值匹配）

```sql
EXPLAIN SELECT * FROM users WHERE age = 25;
-- age 列有索引但非唯一
+------+----------+------+------+
| type | key      | rows | ref  |
+------+----------+------+------+
| ref  | idx_age  | 200  | const|
+------+----------+------+------+
-- ref=const：索引列与一个常量值比较，可能返回多行
```

## eq_ref（唯一索引等值匹配，Join 中的黄金标准）

```sql
EXPLAIN SELECT * FROM users u JOIN orders o ON u.id = o.user_id;
+------+--------+---------+---------+
| type | key    | ref                |
+------+--------+---------+---------+
| eq_ref | PRIMARY | db.o.user_id     |
+------+--------+---------+---------+
-- eq_ref：驱动表每行过来 → 用主键/唯一索引查被驱动表 → 最多返回一行
-- 这是 Join 中最好的被驱动表访问方式
```

## const/system（常量级）

```sql
EXPLAIN SELECT * FROM users WHERE id = 1;
+------+---------+------+------+
| type | key     | ref  | rows |
+------+---------+------+------+
| const | PRIMARY | const | 1   |
+------+---------+------+------+
-- 主键等值查询 + 唯一值：优化器当常量处理，只在优化阶段计算一次
```

---

# 三、key_len——联合索引"用了多少列"的DNA检测

`key_len` 是联合索引使用情况的最佳诊断工具。它不是"索引用了几个列"，而是**索引实际生效了多少字节**。

```sql
-- 表：users
-- id INT PRIMARY KEY, name VARCHAR(50), age INT
-- 联合索引：(name, age)

-- 场景 A：只用到 name
EXPLAIN SELECT * FROM users WHERE name = 'Alice';
key_len = 153  -- 50*3(UTF8MB4) + 2(变长列长度) + 1(NULL标志) = 153
-- 索引生效：name 列 153 字节 ✓，age 列没用到 ✗

-- 场景 B：name + age 都用到了
EXPLAIN SELECT * FROM users WHERE name = 'Alice' AND age = 25;
key_len = 158  -- 153 + 5(INT 4B + 1B NULL标志)
-- 索引生效：name 153 字节 ✓，age 5 字节 ✓ → 两列都用到了！

-- 场景 C：等值 name + 范围 age → 仍然只用到了 name
EXPLAIN SELECT * FROM users WHERE name = 'Alice' AND age > 25;
key_len = 153  -- age 没用到！因为 name 之后的 age 在一个范围内不保证有序
```

**快速估算 key_len 的规则**：

| 列类型 | 基础大小 | + NULL 标记 | + 变长标记 |
|--------|---------|------------|-----------|
| INT | 4B | +1B | — |
| BIGINT | 8B | +1B | — |
| VARCHAR(N) UTF8MB4 | N × 4 | +1B | +2B |
| DATETIME | 5B (8.0 前 8B) | +1B | — |
| CHAR(N) UTF8MB4 | N × 4 | +1B | — |

---

# 四、rows + filtered——估算值的正确使用姿势

## 4.1 rows 是什么？

`rows` 是优化器**估算**的需要扫描的行数。它不是真实值，而是基于索引基数的统计估算：

```sql
-- 查看索引基数（Cardinality）
SHOW INDEX FROM users;
-- Cardinality 列：索引中不同值的数量（采样估算的！）
-- 例：Cardinality=950 → 约 950 个不同的 age 值
-- 表总行数 / Cardinality = 10000/950 ≈ 10.5 → EXPLAIN 的 rows
```

**rows 严重失准的常见原因：**

1. 很久没 `ANALYZE TABLE`——统计信息过时
2. `InnoDB` 默认随机采样 20 个页来计算基数——小表不够精确
3. 大量 DELETE 后 `information_schema` 没更新

```sql
-- 强制更新统计信息
ANALYZE TABLE users;

-- 增加采样页数（提高基数估算精度）
SET GLOBAL innodb_stats_persistent_sample_pages = 100;
```

## 4.2 filtered 的配合使用

`filtered` 是 rows 中预计通过 WHERE 条件的百分比：

```sql
EXPLAIN SELECT * FROM users WHERE age > 20 AND status = 'active';
-- type=range, key=idx_age, rows=1000, filtered=10.00
-- → 优化器估算：先通过 idx_age 扫描 1000 行
-- → 其中满足 status='active' 的约 10%=100 行
-- → 真正需要处理的只有 100 行
```

**rows × filtered% = 实际预估需要处理的行数**

联合索引 `(age, status)` 可以把这个 `filtered` 从 10% 提升到 100%:
```sql
-- 新建索引后
EXPLAIN SELECT * FROM users WHERE age > 20 AND status = 'active';
-- type=range, key=idx_age_status, rows=100, filtered=100.00
-- 索引直接过滤了两个条件 → 不需要二次过滤
```

---

# 五、Extra——执行计划的"秘密通道"

```sql
-- 各 Extra 值对应的真实含义

Using index           ← 覆盖索引，不需要回表（最优）
Using where           ← Server 层做 WHERE 过滤（数据可能多了）
Using index condition ← ICP 下推到引擎层过滤（比 Using where 好）
Using temporary       ← 需要临时表（❌ 性能警告！需要额外的内存/磁盘操作）
Using filesort        ← 需要额外排序（❌ 性能警告！ORDER BY 没走索引）
Using join buffer     ← Join 没有索引，走 BNLJ 或 Hash Join
Using MRR             ← 启用了 Multi-Range Read
```

**Using temporary + Using filesort = 特别需要关注**：

```sql
EXPLAIN SELECT status, COUNT(*) FROM users GROUP BY status ORDER BY COUNT(*) DESC;
+------+------+-----------------+---------------------------------+
| type | rows | Extra                                    |
+------+------+-----------------+---------------------------------+
| ALL  | 10000 | Using temporary; Using filesort          |
+------+------+-----------------+---------------------------------+
-- GROUP BY + ORDER BY 不同列 → 先建临时表分组 → 再排序 → 两次额外操作
```

---

# 六、EXPLAIN ANALYZE——不估算，实际执行

MySQL 8.0.18 引入的 `EXPLAIN ANALYZE` 是革命性的——它真的执行 SQL，输出**每一步的实际时间**：

```sql
mysql> EXPLAIN ANALYZE 
    -> SELECT u.name, COUNT(*) AS cnt
    -> FROM users u JOIN orders o ON u.id = o.user_id
    -> WHERE u.status = 'active'
    -> GROUP BY u.name\G

-> Table scan on <temporary>  (actual time=2.345..2.550 rows=500 loops=1)
    -> Aggregate using temporary table  (actual time=2.342..2.342 rows=500 loops=1)
        -> Nested loop inner join  (cost=1250.00 rows=1000) (actual time=0.123..1.890 rows=980 loops=1)
            -> Filter: (u.status = 'active')  (cost=250.00 rows=500) (actual time=0.056..0.345 rows=500 loops=1)
                -> Table scan on u  (cost=250.00 rows=1000) (actual time=0.045..0.250 rows=1000 loops=1)
            -> Index lookup on o using idx_user_id (user_id=u.id)  (cost=0.35 rows=2) (actual time=0.001..0.002 rows=2 loops=500)
```

阅读顺序：**从内向外，从下向上**。最内层先执行，结果传给外层。每一步都有 `(actual time=start..end rows=N loops=M)`：
- `start`：这一步第一行返回的时间（毫秒）
- `end`：这一步最后一行返回的时间
- `rows=N`：实际返回的行数
- `loops=M`：这一步被调用了多少次（Join 中被驱动表的次数）

**对比估算和实际**：
```
Table scan on u:    估算 rows=1000  → 实际 rows=1000 ✓（估算准）
Filter (status):    估算 rows=500   → 实际 rows=500  ✓
Index lookup on o:  估算 rows=2     → 实际 rows=2    ✓
```

当估算和实际严重偏离时（如估算 rows=1000 实际 rows=100000）→ 统计信息该更新了。

---

# 七、OPTIMIZER_TRACE——看优化器的"思考过程"

```sql
-- 开启 trace
SET optimizer_trace="enabled=on";
SET optimizer_trace_max_mem_size=1000000;

-- 执行你关心的 SQL
SELECT * FROM users WHERE name LIKE 'A%' AND age > 20;
SELECT * FROM information_schema.OPTIMIZER_TRACE\G

-- 关闭 trace（避免性能开销）
SET optimizer_trace="enabled=off";
```

trace 输出的四个阶段：
```json
{
  "join_preparation": {
    // 查询改写阶段：IN → EXISTS, 外连接 → 内连接等
    "select#": 1,
    "transformations": ["IN-to-EXISTS"]
  },
  "join_optimization": {
    // 优化核心阶段：评估所有可能的执行方案
    "considered_execution_plans": [
      {
        "plan_prefix": ["users"],
        "best_access_path": {
          "considered_access_paths": [
            {
              "access_type": "ref",        // 考虑用 ref 访问
              "index": "idx_age",
              "rows": 500,
              "cost": 125.5,               // 成本评估
              "chosen": true               // 选用！成本最低
            },
            {
              "access_type": "range",
              "index": "idx_name",
              "rows": 2000,
              "cost": 450.8,
              "chosen": false              // 不选，成本更高
            }
          ]
        }
      }
    ]
  },
  "join_execution": {
    // 实际执行（9/10 不显示细节）
    "select#": 1
  }
}
```

**实战场景：优化器为什么不选我的索引？**

```sql
-- 你发现 idx_my 没被使用，开 trace 找到原因
"cause": "index does not support the condition"
-- 索引列没有包含 WHERE 条件的第一列 → 最左前缀不满足

"cause": "cost is higher than full table scan" 
-- 优化器认为这个索引的回表成本 > 全表扫描
-- → 考虑加覆盖索引消除回表 → 如改 idx_my(a) 为 idx_my(a, b, c) 包含 SELECT 列
```

---

# 八、索引选择为什么不"准"？

诊断脚本：
```sql
-- 1. 检查统计信息是否过期
SELECT 
    table_name,
    table_rows,
    avg_row_length,
    update_time
FROM information_schema.tables
WHERE table_schema = 'mydb'
ORDER BY update_time;  -- 最近没更新的可能过期了

-- 2. 检查索引基数
SHOW INDEX FROM users;
-- Cardinality 列：如果是 NULL 或 0 → 索引从来没被统计过

-- 3. 强制更新
ANALYZE TABLE users;

-- 4. 如果优化器还是选错 → 考虑用 FORCE INDEX
SELECT * FROM users FORCE INDEX (idx_my) WHERE ...;
-- 但不建议硬编码到应用 SQL —— 优化器版本升级可能导致 FORCE INDEX 变坏事

-- 5. 更温和的方式：调整成本常量（不推荐生产盲改）
-- SET optimizer_search_depth = 0;  -- 限制优化器搜索深度（减少优化时间）
```

---

# 九、总结

| 关注点 | 关键字段 | 正常值 | 需关注 |
|--------|---------|--------|--------|
| 访问方式 | `type` | range/ref/eq_ref/const | ALL/index |
| 索引生效 | `key_len` | 和预期一致 | 比预期少（没用到后面的列） |
| 扫描行数 | `rows` + `filtered` | rows×filtered% ≈ 返回行数 | rows ≫ 返回行数（扫描多返回少） |
| 回表 | `Extra` | Using index | Using where |
| 排序 | `Extra` | 没有 | Using filesort |
| 临时表 | `Extra` | 没有 | Using temporary |

# 延伸阅读

**Do——动手练习：**
- 在测试库上对同一个查询分别用 `EXPLAIN` → `EXPLAIN FORMAT=JSON` → `EXPLAIN ANALYZE`，对比输出
- 用 OPTIMIZER_TRACE 找出优化器选择索引的成本依据
- 故意创建一个无索引的表，跑一个复杂查询，观察优化器在只有全表扫描路径时的"挣扎"

**Todo——深入方向：**
- MySQL 8.0 的 histogram（直方图统计）——`ANALYZE TABLE ... UPDATE HISTOGRAM`
- `optimizer_switch` 的各个 flag 含义——`derived_merge` / `semijoin` / `materialization` 等
- 多列索引的 `index dive` 和 `index statistics` 两种估算方式

*本文参考资料：*
- MySQL 官方文档: EXPLAIN Output Format / Optimizer Trace
- Øystein Grøvlen, "How to read MySQL EXPLAINs"
- 《高性能 MySQL》（第 4 版）——第 6 章（查询性能优化）
