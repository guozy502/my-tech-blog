---
title: "主从复制深度解析——Binlog 格式、并行复制与 GTID"
date: 2026-06-28
description: 从 Binlog 三种格式的适用场景与陷阱、主从延迟的根因与并行复制三代演进（5.6 库级 → 5.7 LOGICAL_CLOCK → 8.0 WRITESET）、GTID 的全局事务标识体系到半同步复制的数据安全边界，拆解 MySQL 主从复制的完整链路。
tags: ["MySQL","主从复制","Binlog","GTID","并行复制","半同步"]
categories: ["MYSQL"]
---

# 历史背景——读写分离的黄金时代

2008-2015 年是 MySQL 主从复制的黄金时代。那时大多数 Web 应用的瓶颈不在"写得多快"而在"读得太多"——一个论坛首页的查询可能 Join 十几张表，但写入只是用户发帖，每秒几次。主从复制完美适配这个场景：Master 处理写入，2-10 个 Slave 分摊读取，成本极低。

但随着业务演进，"写"不再是瓶颈而"主从延迟"成了新痛点。支付场景要求写入后立刻能读到——读写分离不能用了。促销场景的批量更新让 Slave SQL 线程单线程回放跟不上。社区的回答是 GTID（解决切换的自动化问题）和并行复制（解决 Slave 回放慢的问题）——但这些方案本身也引入了新的复杂度。

---

# 一、主从复制的三层架构

## 1.1 搭建主从复制

```sql
-- Master 配置 (my.cnf)
[mysqld]
server-id = 1
log_bin = /var/log/mysql/mysql-bin
binlog_format = ROW
sync_binlog = 1
innodb_flush_log_at_trx_commit = 1

-- Slave 配置 (my.cnf)
[mysqld]
server-id = 2
relay_log = /var/log/mysql/mysql-relay-bin
read_only = ON

-- Master 上创建复制账号
CREATE USER 'repl'@'%' IDENTIFIED BY 'password';
GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%';

-- 获取 Master 当前 Binlog Position
SHOW MASTER STATUS;
-- +------------------+----------+--------------+------------------+
-- | File             | Position | Binlog_Do_DB | Binlog_Ignore_DB |
-- +------------------+----------+--------------+------------------+
-- | mysql-bin.000001 |      872 |              |                  |
-- +------------------+----------+--------------+------------------+

-- Slave 配置复制
CHANGE MASTER TO
    MASTER_HOST = '10.0.1.1',
    MASTER_USER = 'repl',
    MASTER_PASSWORD = 'password',
    MASTER_LOG_FILE = 'mysql-bin.000001',
    MASTER_LOG_POS = 872;

START SLAVE;
SHOW SLAVE STATUS\G
```

## 1.2 三个线程的角色

```
Master:
  Binlog Dump Thread ── 读取 Binlog → 通过网络发给 Slave

Slave:
  I/O Thread ── 接收 Binlog → 写入 Relay Log
  SQL Thread ── 读取 Relay Log → 逐条执行 → 应用到数据
```

## 1.3 SHOW SLAVE STATUS 的关键字段

```sql
SHOW SLAVE STATUS\G
```

| 字段 | 含义 | 健康值 |
|------|------|--------|
| `Slave_IO_Running` / `Slave_SQL_Running` | IO 和 SQL 线程是否正常 | 都 = Yes |
| `Seconds_Behind_Master` | Slave SQL 落后 Master 的时间 | < 5 秒 |
| `Master_Log_File` / `Read_Master_Log_Pos` | IO 线程读到 Master Binlog 的哪个位置 | 接近 Master 最新 Position |
| `Relay_Master_Log_File` / `Exec_Master_Log_Pos` | SQL 线程回放到哪个位置 | 与 Read 位置接近 |
| `Last_IO_Error` / `Last_SQL_Error` | 最后一次 IO 或 SQL 错误 | 为空 |

---

# 二、Binlog 三种格式的深度对比

## 2.1 STATEMENT——省空间但有陷阱

```sql
SET binlog_format = STATEMENT;

-- 危险 SQL 示例：这些在主从可能产生不同结果！
SELECT ... ORDER BY created_at LIMIT 100;          -- 没有确定性排序 → 不同的 100 行
UPDATE ... WHERE id IN (SELECT id FROM ... LIMIT 10); -- 子查询的 LIMIT 不保证确定
INSERT INTO t VALUES (UUID(), NOW());                -- UUID 和 NOW 主从不同
```

## 2.2 ROW——安全但爆炸式增长

```sql
SET binlog_format = ROW;

-- 一条 UPDATE 更新 100 万行 → Binlog 产生 100 万个 Row Event
UPDATE orders SET status = 'expired' WHERE created_at < '2026-01-01';

-- 查看 ROW 格式 Binlog 内容
mysqlbinlog --base64-output=decode-rows -v mysql-bin.000001 | tail -100
-- 输出示例:
-- ### UPDATE `mydb`.`orders`
-- ### WHERE
-- ###   @1=12345     ← id=12345
-- ### SET
-- ###   @3='expired' ← status 变成 expired
```

## 2.3 ROW 格式下的体积优化

```sql
-- ROW 格式默认记录完整行（FULL），改为 MINIMAL 只记录变更列
SET GLOBAL binlog_row_image = MINIMAL;

-- 效果: FULL 记录所有列的前后值 → 每行数据量 > 行大小
--       MINIMAL 只记录主键 + 变更列 → Binlog 体积减少 30-70%

-- 8.0.20+ 支持 Binlog 压缩
SET GLOBAL binlog_transaction_compression = ON;
-- 压缩率约 50-70%（批量 UPDATE/INSERT 和高重复值场景）
```

---

# 三、主从延迟——根因与解法

## 3.1 延迟的根因

```
Master: 50 个客户端并发写入 → Binlog 记录 50 个并发事务
         ↓ 通过网络传输到 Slave
Slave:  SQL 线程 1 个线程 → 串行回放 50 个事务 → 赶不上
         ↑
    单线程回放是瓶颈的根源
```

## 3.2 并行复制的三代演进

**第一代：MySQL 5.6 库级并行**
```sql
-- Slave 配置
slave_parallel_workers = 4
slave_parallel_type = DATABASE  -- 默认

-- 不同库的事务可以并行回放
-- 效果：Master 上有 db1, db2, db3, db4 四个库都在写入 → 4 个 worker 各管各的
-- 局限：单库多表的写入 → 所有事务仍在同一个 worker 上串行
```

**第二代：MySQL 5.7 LOGICAL_CLOCK 并行**
```sql
-- Slave 配置
slave_parallel_type = LOGICAL_CLOCK
slave_parallel_workers = 4

-- 同一组提交（Group Commit）的事务共享相同的 last_committed 值
-- 只要是同一组提交的事务 → 它们在 Master 上没有修改冲突 → 可以并行回放！
-- 效果：单库多表、甚至单表不同行的写都能并行
```

原理：
```
Master 上的 Group Commit：
  last_committed=100  ← 这个时间点之前的事务都已提交
  sequence_number=101: 事务 A（修改 id=1）
  sequence_number=102: 事务 B（修改 id=2）← 与 A 同行冲突？不 → 可以并行
  sequence_number=103: 事务 C（修改 id=1）← 改了 A 刚改的行 → sequence_number 不同 → 不能并行

Slave 上：sequenc_num 相同的事务 → 并行回放；不同 → 按顺序回放
```

**第三代：MySQL 8.0 WRITESET 并行**
```sql
-- Slave 配置
slave_parallel_type = WRITESET

-- 基于行数据的主键/唯一索引计算 writeset
-- 即使不同 sequence_number，只要 writeset 不冲突 → 可以并行！
-- 效果：单库单表场景的并行度大幅提升
```

```sql
-- 监控并行复制的实际效果
SELECT * FROM performance_schema.replication_applier_status_by_worker;
-- 查看每个 worker 处理的最后一个事务和延迟
```

---

# 四、GTID——全局事务 ID 体系

## 4.1 GTID 解决了什么？

```sql
-- 传统复制：基于 Binlog 文件名 + 偏移位置
CHANGE MASTER TO
    MASTER_LOG_FILE = 'mysql-bin.000005',
    MASTER_LOG_POS = 12345678;

-- 问题：Master 故障后，新 Master 的 Binlog 文件和位置和旧 Master 不同
--      你无法知道 "旧 Master 的 000005:12345678 = 新 Master 的 000003:9876543"
--      需要手工计算新 Master 对应的位置——容易错
```

```sql
-- GTID 复制：基于全局事务 ID
CHANGE MASTER TO
    MASTER_AUTO_POSITION = 1;  -- 自动！Slave 自己知道它需要哪些事务

-- GTID 格式：server_uuid:transaction_id
-- 例：3E11FA47-71CA-11E1-9E33-C80AA9429562:1-2345

-- Slave 启动时：
-- 1. 告诉 Master："我执行过了哪些 GTID"
-- 2. Master 回复："我多出来的是这些 GTID"→ 发送对应的事务
-- 3. 不需要任何人算 Binlog Position —— GTID 是全集群唯一且有序的
```

## 4.2 GTID 运维操作

```sql
-- 查看已执行的 GTID 集合
SHOW MASTER STATUS\G  -- Executed_Gtid_Set: 3E11...:1-2345

-- 新增 Slave 时跳过已执行 GTID（从 Master 全量备份恢复后）
RESET MASTER;
SET GLOBAL gtid_purged = '3E11FA47...:1-2345';  -- 这之前的已在你备份中
-- 然后 Slave 启动复制 → 从 2346 开始追

-- 查出 Executed GTID 与 Purged GTID 的差异
-- Executed: 这个节点执行过的所有 GTID
-- Purged:   这些 GTID 对应的 Binlog 已经从磁盘删除了
SELECT @@gtid_executed, @@gtid_purged;
```

## 4.3 GTID 的限制

```sql
-- 以下操作在 GTID 模式下会报错：
CREATE TABLE t1 SELECT * FROM t2;  
-- ERROR: Statement is not safe for GTID-based replication
-- 原因：一个语句产生两个 GTID（CREATE TABLE + INSERT...SELECT）

-- 解法：拆为两步
CREATE TABLE t1 LIKE t2;
INSERT INTO t1 SELECT * FROM t2;

-- 临时表在事务中使用也有限制
-- 建议应用代码层面规避
```

---

# 五、半同步复制——性能与安全的平衡

## 5.1 异步 vs 半同步

```sql
-- 安装半同步插件（Master 和 Slave 都执行）
INSTALL PLUGIN rpl_semi_sync_master SONAME 'semisync_master.so';
INSTALL PLUGIN rpl_semi_sync_slave SONAME 'semisync_slave.so';

-- Master 配置
SET GLOBAL rpl_semi_sync_master_enabled = ON;
SET GLOBAL rpl_semi_sync_master_timeout = 1000;   -- 超时 1 秒自动退化为异步
SET GLOBAL rpl_semi_sync_master_wait_for_slave_count = 1;  -- 至少等 1 个 Slave 确认

-- Slave 配置
SET GLOBAL rpl_semi_sync_slave_enabled = ON;
STOP SLAVE IO_THREAD; START SLAVE IO_THREAD;  -- 重启 IO 线程使配置生效
```

**异步 vs 半同步 vs 增强半同步的时间线区别**：

```
异步复制：
  ① Master commit → ② 写 Binlog → ③ 返回客户端 OK
                                            ↑ 这时 Slave 可能还没收到

半同步 (AFTER_COMMIT, 5.5/5.6)：
  ① Master commit → ② 写 Binlog → ③ 等 Slave IO 线程确认收到 → ④ 返回客户端 OK
    问题: ③ 确认收到前其他客户端已经看到数据了（已 commit）
    如果 Master 在 ③ 后宕机 → 数据已对外可见但 Slave 没有 → 切主后丢了

增强半同步 (AFTER_SYNC, 5.7+)：
  ① Master 写 Binlog → ② 等 Slave IO 线程确认收到 → ③ Master commit → ④ 返回 OK
    核心: 先确认 Slave 收到 Binlog → 再让其他客户端看到数据
    即使 Master 在 ③ 后宕机 → Slave 有 Binlog → 新 Master 有这份数据
```

---

# 六、总结

| 技术 | 解决的问题 | 代价 |
|------|----------|------|
| **ROW Binlog** | STATEMENT 不确定性导致主从不一致 | Binlog 体积增大 5-50 倍 |
| **并行复制 5.7 LOGICAL_CLOCK** | Slave 单线程回放太慢 | 增加 Slave CPU 和 worker 线程管理开销 |
| **并行复制 8.0 WRITESET** | 单库单表场景并行度不够 | 更多 CPU 计算 writeset 冲突检测 |
| **GTID** | 主从切换需要手动找同步点 | 部分操作不支持（CREATE...SELECT、临时表） |
| **半同步** | 异步复制失败时可能丢数据 | 每次事务多一次网络 RTT |

# 延伸阅读

**Do——动手搭建：**
- Docker Compose 搭建 1 Master + 2 Slave 环境，验证 ROW 格式下 Binlog 内容
- 模拟 Slave 延迟——在 Master 上执行 10 万次 UPDATE，观察 `Seconds_Behind_Master`
- 用 GTID 方式搭建复制并模拟主从切换（`CHANGE MASTER TO MASTER_AUTO_POSITION=1`）

**Todo——深入方向：**
- 8.0.22 的异步复制连接故障转移（Replication Channel 多源）
- `binlog_transaction_dependency_tracking=WRITESET` 在 Master 端的冲突检测原理
- Clone Plugin（8.0.17+）替代 xtrabackup 做 Slave 初始化的流程

*本文参考资料：*
- MySQL 官方文档: Replication
- 《高性能 MySQL》（第 4 版）——第 9 章
- Jean-François Gagné, "MySQL 8.0 Replication: New Features"
