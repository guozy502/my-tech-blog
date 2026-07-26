---
title: "慢查询诊断工具链——Performance Schema、sys schema 与 pt-query-digest"
date: 2026-06-28
description: 从慢查询日志的配置与解析、Performance Schema 的 SQL 指纹聚合统计、sys schema 的冗余索引/全表扫描诊断视图到 pt-query-digest 的离线分析与 pt-online-schema-change 的无阻塞 DDL 原理，构建 MySQL 慢查询诊断的全套工具链。
tags: ["MySQL","慢查询","Performance Schema","sys schema","pt-query-digest","DDL"]
categories: ["MYSQL"]
---

# 历史背景——MySQL 可观测性的三个时代

**石器时代（MySQL 5.0 之前）**：只有慢查询日志。纯文本文件，不能结构化查询。想看"过去一周最慢的 10 个查询是什么？"——你得用 Linux 的 grep/awk/sort 手工分析。

**青铜时代（MySQL 5.5-5.6）**：Performance Schema 诞生。MySQL 终于有了类似 Oracle AWR 的动态性能视图。所有指标在数据库内就能查。但 P_S 的 SQL 写法极其复杂（涉及多表 JOIN 和晦涩的 instrument 表名），DBA 门槛极高。

**现代（MySQL 5.7-8.0）**：sys schema 出现——把复杂的 P_S 查询封装成人类友好的视图。同时 Percona Toolkit 作为外部工具链补充了 MySQL 内置工具覆盖不到的场景（在线 DDL、慢查询离线分析、从库延迟检测等）。今天，这三套工具各司其职：**P_S 管实时数据，sys 管便捷可读性，pt 管运维任务**。

---

# 一、慢查询日志——最基础也最可靠

## 1.1 开启和配置

```sql
-- 查看慢查询配置
SHOW VARIABLES LIKE 'slow_query%';
SHOW VARIABLES LIKE 'long_query_time';

-- 生产配置建议
SET GLOBAL slow_query_log = ON;
SET GLOBAL long_query_time = 0.1;           -- 100ms（不是默认 10s！）
SET GLOBAL log_queries_not_using_indexes = ON;  -- 没走索引的 SQL 也记录
SET GLOBAL log_slow_admin_statements = ON;  -- DDL 也记录
SET GLOBAL log_slow_extra = ON;             -- 8.0.14+ 记录更多字段
```

## 1.2 慢查询日志的格式

```sql
-- 一条典型的慢查询日志
# Time: 2026-07-27T10:30:00.123456+08:00
# User@Host: app_user[root] @ [10.0.1.5] Id: 12345
# Query_time: 2.345678  Lock_time: 0.000123  Rows_sent: 10  Rows_examined: 500000
# Thread_id: 12345  Errno: 0  Killed: 0  Bytes_received: 256  Bytes_sent: 520
SET timestamp=1754223000;
SELECT * FROM orders WHERE user_id = 123 ORDER BY created_at DESC;
```

**关键字段解读**：

| 字段 | 含义 | 诊断价值 |
|------|------|---------|
| `Query_time` | SQL 执行总时间 | 是否 > 阈值 |
| `Rows_examined / Rows_sent` | 扫描行数 / 返回行数 | **比值 >> 1** → 全表扫描或索引无效 |
| `Lock_time` | 等待锁的时间 | 显著 > 0 → 锁竞争严重 |
| `Bytes_sent` | 返回给客户端的数据量 | 异常大 → 可能是大字段（TEXT/BLOB） |
| `Thread_id` | 执行该 SQL 的线程 ID | 可用于 `SHOW PROCESSLIST` 或 `EXPLAIN FOR CONNECTION` |

## 1.3 mysqldumpslow 分析

```bash
# MySQL 自带的慢查询分析工具
# 按执行时间排序（前 10）
mysqldumpslow -s t -t 10 slow.log

# 按出现次数排序
mysqldumpslow -s c -t 10 slow.log

# 限制只看特定数据库
mysqldumpslow -s at -t 10 slow.log | grep "mydb"

# 输出示例:
# Count: 2345  Time=2.34s (5489s)  Lock=0.00s (2s)  
# Rows_sent=10.0 (23450)  Rows_examined=500000.0 (1172500000)
# SELECT * FROM orders WHERE user_id = N ORDER BY created_at DESC
#   ↑ N 表示参数被替换为抽象值，同一模式的 SQL 聚合在一起
```

---

# 二、Performance Schema——数据库内的"诊断仪表盘"

## 2.1 P_S 的基本概念

Performance Schema 是 MySQL 内置的性能诊断引擎。它以内存表形式存储，记录所有 SQL 执行、锁等待、IO 操作等事件：

```sql
-- 查看 P_S 是否启用
SHOW VARIABLES LIKE 'performance_schema';  -- ON/OFF

-- 5.7+ 默认开启，开销 < 5% CPU
-- 如果嫌开销大，可以关闭部分 instrument:
-- UPDATE performance_schema.setup_instruments 
-- SET ENABLED = 'NO' WHERE NAME LIKE 'wait/synch/%';  -- 关闭互斥锁统计
```

## 2.2 核心表：按 SQL 指纹聚合的统计

`events_statements_summary_by_digest` 是 P_S 中最重要的一张表——它按 SQL 的"指纹"（参数化的 SQL 文本）聚合了所有执行统计：

```sql
-- 找出执行最频繁的 10 个 SQL
SELECT 
    DIGEST_TEXT AS 'SQL 指纹',
    COUNT_STAR AS '执行次数',
    ROUND(AVG_TIMER_WAIT / 1000000000, 2) AS '平均耗时(ms)',
    ROUND(SUM_TIMER_WAIT / 1000000000, 2) AS '总耗时(ms)',
    ROUND(SUM_ROWS_EXAMINED / COUNT_STAR, 0) AS '平均扫描行数',
    ROUND(SUM_ROWS_SENT / COUNT_STAR, 0) AS '平均返回行数',
    ROUND(SUM_ROWS_EXAMINED / SUM_ROWS_SENT, 1) AS '扫描/返回比'
FROM performance_schema.events_statements_summary_by_digest
WHERE DIGEST_TEXT IS NOT NULL 
ORDER BY SUM_TIMER_WAIT DESC
LIMIT 10;
```

输出示例：
```
+--------------------------------------+----------+------------+----------+-------------+-------------+------------+
| SQL 指纹                              | 执行次数 | 平均耗时ms | 总耗时ms | 平均扫描行数 | 平均返回行数 | 扫描/返回比 |
+--------------------------------------+----------+------------+----------+-------------+-------------+------------+
| SELECT * FROM orders WHERE ...        |  2345    |  2340.12   | 5487000  |  500000     |  10         |  50000.0   |
| UPDATE users SET last_login = ...     |  890000  |  0.05      | 44500    |  1          |  0          |  0.0       |
+--------------------------------------+----------+------------+----------+-------------+-------------+------------+
```

第一条 SQL：平均扫描 50 万行返回 10 行 → 严重缺少索引 → 优化价值最大。

## 2.3 按表维度的 IO 统计

```sql
-- 每张表的 IO 统计
SELECT 
    OBJECT_SCHEMA AS '库',
    OBJECT_NAME AS '表',
    COUNT_READ AS '读次数',
    COUNT_WRITE AS '写次数',
    SUM_TIMER_WAIT / 1000000000 AS '总耗时(ms)'
FROM performance_schema.table_io_waits_summary_by_table
WHERE OBJECT_SCHEMA NOT IN ('mysql', 'performance_schema', 'sys')
ORDER BY SUM_TIMER_WAIT DESC
LIMIT 10;
-- 找出 IO 最密集的表
```

## 2.4 实时监控连接的 SQL

```sql
-- 查看当前所有连接正在执行的 SQL
SELECT 
    THREAD_ID,
    PROCESSLIST_ID AS '连接ID',
    PROCESSLIST_INFO AS '执行的SQL',
    TIMER_WAIT / 1000000000 AS '已执行ms'
FROM performance_schema.events_statements_current
WHERE PROCESSLIST_INFO IS NOT NULL 
  AND PROCESSLIST_STATE = 'executing'
ORDER BY TIMER_WAIT DESC;

-- 对一个正在运行的 SQL 看它的执行计划（8.0+）
EXPLAIN FOR CONNECTION 12345;
-- 不需要等 SQL 执行完就能看它走什么索引！
```

---

# 三、sys schema——把 P_S 封装成易用视图

sys schema 是建立在 P_S 和 `information_schema` 之上的视图集合。它解决的核心痛点是——P_S 的表名和列名太难记了。

```sql
-- 检查 sys schema 是否可用
SELECT * FROM sys.version;
```

## 3.1 最常用的六个 sys 视图

```sql
-- ① 从未使用的索引（可以直接删！）
SELECT * FROM sys.schema_unused_indexes
WHERE object_schema = 'mydb';
-- 注意：P_S 重启后数据清空，建议累计至少一个月再删

-- ② 冗余索引
SELECT * FROM sys.schema_redundant_indexes
WHERE table_schema = 'mydb';
-- redundant_index: idx_a_b_c_d, dominant_index: PRIMARY
-- → idx_a_b_c_d 完全被主键覆盖（问题索引的设计包含主键）

-- ③ 有全表扫描的表
SELECT * FROM sys.schema_tables_with_full_table_scans
WHERE object_schema = 'mydb';
-- rows_full_scanned: 历史累计全表扫行数 → 关注增速

-- ④ 导致全表扫描的 SQL
SELECT 
    query, db, exec_count,
    rows_sent_avg, rows_examined_avg
FROM sys.statements_with_full_table_scans
WHERE db = 'mydb'
ORDER BY total_latency DESC
LIMIT 10;

-- ⑤ 锁等待（当前！谁阻塞了谁）
SELECT 
    waiting_pid, waiting_query,
    blocking_pid, blocking_query,
    wait_age
FROM sys.innodb_lock_waits;
-- 立刻知道是谁持锁导致的阻塞

-- ⑥ 磁盘 IO 最密集的文件
SELECT * FROM sys.io_global_by_file_by_bytes
WHERE file LIKE '%mydb%'
ORDER BY total DESC
LIMIT 10;
-- 找出哪些表文件的 IO 量最大
```

## 3.2 关于 schema_unused_indexes 的注意事项

```sql
-- 误删常见场景：
-- 1. Unique 约束的索引即使没被查询用到，也不能删
--    → 检查 schema_unused_indexes 中的 NON_UNIQUE=0
-- 2. 外键列的索引即使没被查询用到，也不能删（外键需要）
-- 3. P_S 刚重启 → 数据没累积够 → 可能把热门索引标记为"未使用"

-- 安全做法：用 pt-duplicate-key-checker 辅助判断
pt-duplicate-key-checker h=localhost,u=root --ask-pass
```

---

# 四、pt-query-digest——离线分析的首选

## 4.1 分析慢查询日志

```bash
# 基本用法
pt-query-digest slow.log > slow_report.txt

# 分析指定时间范围
pt-query-digest slow.log \
  --since '2026-07-27 10:00:00' \
  --until '2026-07-27 11:00:00'

# 分析 tcpdump（在数据库不在本机时有用）
tcpdump -i eth0 port 3306 -s 65535 -w mysql.pcap
pt-query-digest --type tcpdump mysql.pcap
```

## 4.2 报告解读

```
# Profile
# Rank Query ID           Response time    Calls  R/Call V/M  
# ==== ================== ================ ====== ====== =====
#    1 0x1234...ABCDEF    5487.00 65.2%    2345   2.34   0.01 SELECT orders
#    2 0x5678...FEDCBA    2010.00 23.9%  890000   0.002  0.00 UPDATE users
#    3 0x9ABC...123456     920.00 10.9%   12345   0.07   0.03 SELECT products
#
# 排名 #1: 占总耗时 65.2%，调优优先级最高
# V/M: 方差/均值 → 值越大说明这个 SQL 的执行时间波动越大
```

## 4.3 监控 processlist 快照

```bash
# 定时抓 processlist 可用于事后分析（不需要预先开慢查询日志）
pt-query-digest --type processlist \
  --processlist h=localhost,u=root --ask-pass
```

---

# 五、DDL 优化——在线变更的武器库

## 5.1 MySQL 官方 DDL 能力矩阵

```sql
-- 查看 DDL 是否支持 INPLACE
-- 很多 DDL 操作需要 COPY 整表（= 锁表时间与表大小成正比）
ALTER TABLE orders ADD COLUMN tags VARCHAR(200) DEFAULT '', 
  ALGORITHM=INPLACE, LOCK=NONE;
-- 如果失败 → MySQL 会报错并提示不支持 → 必须走 COPY

-- 8.0.12+ 支持 INSTANT 加列（只改元数据不碰数据页）
ALTER TABLE orders ADD COLUMN comments VARCHAR(200) DEFAULT '', 
  ALGORITHM=INSTANT;
-- 只要改元数据字典 → 无需等数据拷贝 → 1 毫秒完成
```

## 5.2 pt-online-schema-change——当官方 DDL 不够用

```bash
# 原理：创建新表 → 在原表建触发器(捕获增量) → 分批拷贝数据 → 原子 swap
pt-online-schema-change \
  --alter "ADD COLUMN tags VARCHAR(200) DEFAULT ''" \
  --alter-foreign-keys-method auto \
  --execute \
  h=localhost,D=mydb,t=orders

# 关键流程:
# 1. CREATE TABLE orders_new LIKE orders     -- 建新表
# 2. ALTER TABLE orders_new ADD COLUMN ...   -- 在新表上执行 DDL（锁新表几秒，不锁原表）
# 3. CREATE TRIGGER AFTER INSERT/UPDATE/DELETE ON orders -- 捕获增量写入
# 4. INSERT INTO orders_new SELECT ... FROM orders -- 分批拷贝(chunk-size 1000)
# 5. RENAME TABLE orders TO orders_old, orders_new TO orders -- 原子 swap
# 6. DROP TABLE orders_old                   -- 清理
```

**pt-osc 的注意事项**：
- 需要至少 2 倍磁盘空间（新旧表各一份）
- 有外键的表处理复杂（建议配合 `alter-foreign-keys-method rebuild_constraints`）
- 必须在表上有主键或唯一索引（用于分块 SELECT）

---

# 六、工具链选择决策

```
实时诊断当前问题:
  → sys.innodb_lock_waits (锁阻塞)
  → EXPLAIN FOR CONNECTION (正在跑的SQL)
  → SHOW PROCESSLIST / sys.processlist

日常巡检慢查询:
  → P_S.events_statements_summary_by_digest (按 SQL 指纹聚合)
  → sys.statements_with_full_table_scans (全表扫描)

离线深度分析:
  → pt-query-digest slow.log

索引优化:
  → sys.schema_redundant_indexes (冗余索引)
  → sys.schema_unused_indexes (未使用索引)
  → EXPLAIN + OPTIMIZER_TRACE (单条 SQL 优化)

在线 DDL:
  → ALGORITHM=INPLACE / INSTANT (优先用官方)
  → pt-online-schema-change (官方不支持 INPLACE 时的兜底)
```

---

# 七、总结

| 工具 | 适用场景 | 一句话 |
|------|---------|--------|
| **慢查询日志** | 记录所有慢 SQL | 最基础，最可靠，但离线分析需要额外工具 |
| **P_S** | 在线实时诊断 | 全维度数据，但 SQL 查询语法复杂 |
| **sys schema** | 日常巡检 | 把 P_S 封装成易用视图 |
| **pt-query-digest** | 离线深度分析 | 慢查询日志/processlist/tcpdump 全支持 |
| **pt-osc** | 在线无阻塞 DDL | 大表变更的救星 |

# 延伸阅读

**Do——动手搭建：**
- 把 `long_query_time=0.1` 和 `log_queries_not_using_indexes=ON` 设为生产默认值
- 用 `sys.innodb_lock_waits` 配合 cron 脚本实现锁等待自动告警
- 在测试库模拟一个在线 DDL 场景（100 万行表在执行 `ALTER TABLE ADD COLUMN` 的过程中持续 insert）

**Todo——深入方向：**
- P_S 的 memory instruments——`memory_summary_*` 表分析 MySQL 内存使用
- MySQL 8.0 的 `log_slow_extra=ON` 新增字段的使用（`Bytes_sent`、`Thread_id` 等）
- pt-table-checksum + pt-table-sync 用于主从数据一致性校验和修复

*本文参考资料：*
- MySQL 官方文档: Performance Schema / sys schema / Slow Query Log
- Percona Toolkit 官方文档
- 《高性能 MySQL》（第 4 版）——第 3-4、8 章
