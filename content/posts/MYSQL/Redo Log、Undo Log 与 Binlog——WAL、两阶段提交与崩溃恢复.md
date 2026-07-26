---
title: "Redo Log、Undo Log 与 Binlog——WAL、两阶段提交与崩溃恢复"
date: 2026-06-28
description: 从 WAL 机制的设计动机、Redo Log 的物理日志与 LSN 递增体系、Undo Log 的 MVCC 版本回溯、Binlog 的三种格式与组提交，到 Redo-Binlog 两阶段提交的四种崩溃恢复判断逻辑，拆解 MySQL 三套日志的协作机制。
tags: ["MySQL","InnoDB","Redo Log","Binlog","Undo Log","WAL","崩溃恢复"]
categories: ["MYSQL"]
---

# 历史背景——MySQL 为什么需要三套日志？

这个问题的真相不太好看：三套日志**不是精心设计的分层架构，而是历史架构分层的产物**。MySQL 最初由瑞典公司 MySQL AB 开发，Server 层（含 Binlog）是它自己的代码。而 InnoDB 是芬兰公司 Innobase Oy 的产品，自带 Redo Log 和 Undo Log。2001 年 InnoDB 被引入 MySQL，两套日志体系从此并存。

这意味着：MySQL 每次提交事务，既要写 Server 层的 Binlog（给复制用），也要写 InnoDB 层的 Redo Log（给崩溃恢复用）——两层之间没有共享的写入路径。为了保证两层之间的提交命运一致（要么都成功，要么都不成功），MySQL 引入了一个"胶水协议"——**两阶段提交（2PC）**。

对比 PostgreSQL：PG 只有一套 WAL（Write-Ahead Log），一台机器写一次日志，管崩溃恢复也管复制。干净利落。MySQL 的"三日志"模式是技术债——但这份技术债撑起了 MySQL 生态的兼容性和灵活性（你可以换存储引擎，Binlog 依然正常工作）。

---

# 一、WAL 的设计哲学

## 1.1 为什么是"日志先行"？

假设没有 WAL——每次事务提交时直接修改磁盘上的数据页。那将意味着：

- 每个 UPDATE 要等磁盘寻道（随机 IO，~10ms HDD）
- 一个事务修改了 10 个页 → 可能在磁盘的不同位置 → 10 次寻道 → 100ms+
- TPS 天花板 = 10 个事务/秒

**WAL 的解法**：数据和索引的"最终位置"在磁盘数据文件中，但修改不是直接写那里。而是先把修改顺序追加到日志文件中——日志文件是顺序写的（几个 GB 的大文件连续追加），速度可达 100-600MB/s。

```
事务提交流程（WAL）：
  ① 写 Log Buffer（内存）
  ② Log Buffer → Redo Log 文件（顺序写，可能 fsync）
  ③ 返回客户端 "OK"（数据页还没动！）
  ④ 后台 page cleaner 线程慢慢把脏页刷到数据文件（随机写，异步）
```

**核心权衡**：用"延迟写数据页"换取"事务快速提交"。代价是——崩溃恢复时必须重放日志才能重建数据页。

## 1.2 FORCE vs STEAL 策略

数据库领域描述 Buffer Pool 管理策略有两个术语：

```sql
-- InnoDB 的策略：No-FORCE + STEAL
-- 
-- No-FORCE：事务提交时，不必强制立即把数据页刷到磁盘
--           （只保证 Redo Log 落盘，脏页可以慢慢刷）
-- STEAL：Buffer Pool 空间不够时，可以把未提交事务的脏页刷到磁盘
--         （然后再通过 Undo Log 回滚未提交事务的数据）
--
-- 这个组合让 Buffer Pool 的空间管理非常灵活，
-- 代价是崩溃恢复需要 Redo（重做已提交）+ Undo（回滚未提交）
```

---

# 二、Redo Log——crash-safe 的保证

## 2.1 Redo Log 的物理结构

```bash
# 查看 Redo Log 配置
SHOW VARIABLES LIKE 'innodb_log%';

# 典型配置
# innodb_log_file_size = 512M    ← 每个 redo log 文件大小
# innodb_log_files_in_group = 2  ← 文件数量（组内轮换循环写）
# innodb_log_buffer_size = 16M   ← 内存缓冲区
```

Redo Log 文件在磁盘上是固定大小的，**循环写**：

```
┌──────────────┐  ┌──────────────┐
│ ib_logfile0  │  │ ib_logfile1  │
│   512MB      │  │   512MB      │
└──────────────┘  └──────────────┘
     ↑ write pos (当前写到哪里）
     ↑ checkpoint pos (这个点之前的日志已经刷到数据页，可以覆盖）

当 write pos 追上 checkpoint pos → Redo Log 满了 → 必须等待 checkpoint 推进
（强制刷脏页 → 释放 Redo Log 空间 → write pos 才能往前写）
```

## 2.2 Redo Log 的格式

Redo Log 是**物理日志**，记录的是"页 x 的偏移量 y 处把 a 改成 b"：

```sql
-- 一个 UPDATE t SET name='Bob' WHERE id=1 产生的 Redo（简化描述）：
-- ① 修改 undo log 页（为 UPDATE 生成 undo 数据）
--    → redo: page 100, offset 50, old_data → new_undo_data
-- ② 修改聚簇索引的数据页
--    → redo: page 200, offset 100, 'Alice' → 'Bob'
-- ③ 如果有二级索引
--    → redo: page 300, offset 80, old_index_entry → new_index_entry
```

一条 SQL 可能产生多个 Redo 记录（MTR, Mini-Transaction 概念）。每个 MTR 是原子单位——崩溃恢复时要么全部重放，要么全部回滚。

## 2.3 LSN——贯穿一切的递增序列号

LSN（Log Sequence Number）是 8 字节的全局单调递增数字。它贯穿 Redo Log、数据页、Checkpoint：

```sql
-- 查看当前 LSN 位置
SHOW ENGINE INNODB STATUS\G
-- 在 LOG 段：
-- Log sequence number          123456789000
-- Log buffer assigned up to    123456789500
-- Log buffer completed up to   123456789450
-- Log written up to            123456789450
-- Log flushed up to            123456789450
-- Pages flushed up to          123456780000  ← checkpoint LSN
-- Last checkpoint at           123456780000
```

**LSN 在多个位置出现，每个含义不同**：

| LSN 位置 | 含义 |
|---------|------|
| `Log sequence number` | 当前已分配的最新 LSN |
| `Log flushed up to` | 已经 fsync 到磁盘的 LSN |
| `Pages flushed up to` | 数据页已经刷到磁盘的 LSN（checkpoint 点） |
| 数据页的 `FIL_PAGE_LSN` | 这个数据页最后一次被修改时的 LSN |

崩溃恢复时，InnoDB 比较数据页的 `FIL_PAGE_LSN` 和 Redo Log 中的记录——如果页的 LSN 小于日志记录的 LSN，说明这个修改还没刷到数据页 → 需要重放。

## 2.4 组提交——多个 fsync 合并为一次

```bash
# 控制组提交的关键参数
innodb_flush_log_at_trx_commit = 1
sync_binlog = 1
binlog_group_commit_sync_delay = 0  # 微秒（0=不等，不延迟）
binlog_group_commit_sync_no_delay_count = 0  # 攒够多少个事务就刷

# 适度的组提交延迟可以大幅提升高并发下的写入性能
# 代价是单个事务的提交延迟增加
# 例如：
# binlog_group_commit_sync_delay = 100   # 等 100 微秒
# binlog_group_commit_sync_no_delay_count = 10  # 或攒够 10 个事务
```

组提交的核心思想：高并发下，100 个事务几乎同时提交 → 每个事务的 fsync 花费约 1ms → 如果串行 fsync 需要 100ms → 组提交把这 100 个 fsync 合并成 1 次 → 1ms 完成 100 个事务的提交。

---

# 三、Undo Log——事务回滚与 MVCC 的载体

## 3.1 Undo Log 存了什么？

Undo Log 是**逻辑日志**，记录"如何撤销这个操作"：

```sql
-- INSERT INTO t VALUES (1, 'Alice');
-- Undo: DELETE FROM t WHERE id=1;  (标志这条记录是被 INSERT 创建的)

-- UPDATE t SET name='Bob' WHERE id=1;
-- Undo: UPDATE t SET name='Alice' WHERE id=1;  (回原值)

-- DELETE FROM t WHERE id=1;
-- Undo: INSERT INTO t VALUES (1, 'Bob');  (逆向插入回原值)
```

## 3.2 Undo Log 的持久化——Undo 本身也有 Redo 保护

```
修改数据行 → 写 Undo Log 页 → Undo 页的修改也产生 Redo Log
                                  ↑
                            这就是"双层保障"
```

这意味着崩溃恢复时，连 Undo Log 也能被恢复——先用 Redo 把 Undo Log 恢复出来，再用 Undo Log 回滚未提交的事务。

## 3.3 Purge——清理没人要的老版本

```sql
-- 查看 Purge 线程状态
SHOW ENGINE INNODB STATUS\G
-- 在 TRANSACTIONS 段：
-- Purge done for trx's n:o < 0x12345  ← 已经清理到这个 LSN 之前的事务
-- History list length 23               ← 还有 23 个旧版本等待清理
```

`History list length` 是一个重要监控指标。如果它持续增长，说明有**长事务**在阻止 Purge 工作（长事务的 ReadView 能看到太老的版本 → Purge 线程不确定"有没有人还需要这个老版本"→ 不敢清理）。

```sql
-- 找出最老的事务
SELECT 
    trx_id, trx_started,
    TIMESTAMPDIFF(SECOND, trx_started, NOW()) AS duration_sec,
    trx_mysql_thread_id
FROM information_schema.INNODB_TRX
ORDER BY trx_started
LIMIT 5;
-- 如果有一个事务已经运行了 3600 秒 → 过去一小时的 Undo Log 都没法清理
```

---

# 四、Binlog——Server 层的复制日志

## 4.1 三种格式

```sql
-- 查看当前 Binlog 格式
SHOW VARIABLES LIKE 'binlog_format';

-- 查看 Binlog 内容
SHOW BINARY LOGS;
SHOW MASTER STATUS;
mysqlbinlog --base64-output=decode-rows -v binlog.000001
```

| 格式 | 记录内容 | 示例（一个 UPDATE 改 3 行） |
|------|---------|--------------------------|
| **STATEMENT** | 原始 SQL 语句 | `UPDATE t SET c=c+1 WHERE id>5`（一行） |
| **ROW** | 每行变更的前后值 | 3 个 Row Event，每个含 before/after 行数据 |
| **MIXED** | 默认 STATEMENT，不确定时切 ROW | 取决于 SQL 的类型和上下文 |

**STATEMENT 的陷阱**：

```sql
-- 这个 SQL 在主从可能产生不同结果！
UPDATE users SET views = views + 1 WHERE id IN (SELECT id FROM hot_users LIMIT 100);
-- LIMIT 没有 ORDER BY → 主从可能选中不同的 100 行 → 结果不一致
```

**ROW 格式的优点和代价**：

```sql
-- ROW 格式：每条被修改的行都记录 before & after
-- 一个 UPDATE 批处理更新了 100 万行
-- → Binlog 中产生 100 万个 Row Event
-- → 瞬间膨胀到几百 MB
-- → 磁盘 IO 和网络传输压力都炸了
```

```bash
# ROW 格式下的优化：减少 Binlog 体积
binlog_row_image = MINIMAL  # 只记录变更列（默认 FULL 记录整行）
# FULL: before_image + after_image 都是完整行
# MINIMAL: 只记录主键列 + 变更列
```

## 4.2 Binlog 的写入与 fsync

```bash
sync_binlog = 1  # 每次提交事务都 fsync Binlog（最安全）
# sync_binlog = 0  # 交给 OS 决定（可能丢 Binlog）
# sync_binlog = N  # 每 N 次提交 fsync 一次（批量）

# 生产建议：
# 数据不能丢 → sync_binlog = 1
# 数据可以丢 1 秒 → sync_binlog = 0 或 N（性能更好）
```

---

# 五、两阶段提交——Redo 和 Binlog 的一致性协议

## 5.1 为什么需要 2PC？

```
假设没有 2PC，先写 Redo 再写 Binlog：

  事务提交
    → 写 Redo Log 成功 ✓
    → 写 Binlog 时宕机 ✗（Binlog 没有这条记录）
      → Slave 没有收到这个事务
      → 但崩溃恢复时，Redo 里有 → Master 恢复了这条数据
        → 主从数据不一致！
```

反过来（先写 Binlog 再写 Redo）同样问题：

```
  事务提交
    → 写 Binlog 成功 ✓
    → 写 Redo 时宕机 ✗
      → Slave 收到了这个事务 → 执行了
      → Master 崩溃恢复 → Redo 没有 → 没恢复
        → 主从数据又不一致！
```

## 5.2 2PC 流程

```java
// MySQL 内部 2PC 流程（伪代码）
public void commit(Transaction trx) {
    // === Phase 1: Prepare ===
    // ① 写 Redo Log，标记为 TRX_PREPARED 状态
    //    此时 Redo Log 中该事务处于"PREPARE"状态（未提交）
    innodb_xa_prepare(trx);
    // ② fsync Redo Log（确保 PREPARE 记录落盘）
    if (flush_log_at_trx_commit == 1) {
        fsync(redo_log);
    }
    
    // === Phase 2: Commit ===
    // ③ 写 Binlog
    write_binlog(trx);
    // ④ fsync Binlog（确保 Binlog 落盘）
    if (sync_binlog == 1) {
        fsync(binlog);
    }
    
    // ⑤ 写 Redo Log COMMIT 标记
    //    这一步写的是"这个小事务已经提交了"，很短，但标志着该事务已最终提交
    innodb_commit(trx);
    // 实际上这一步不需要立刻 fsync，可以等下次提交一起
}
```

**核心保障**：在 Binlog 写入并 fsync 之后（步骤④），才能写 Redo 的 COMMIT 标记（步骤⑤）。因为从崩溃恢复的角度——如果 Binlog 里没有这个事务，Redo 应该回滚它；如果 Binlog 里有，Redo 应该提交它。

## 5.3 崩溃恢复的四种判断

```
MySQL 启动 → 扫描 Redo Log → 找出所有 TRX_PREPARED 状态的事务

对每个 PREPARE 的事务：
  
  ① 去 Binlog 中查找该事务的 XID
  ② if (Binlog 里没有该 XID)：
       → 这个事务还没被外部系统"知道"
       → 回滚（Undo Log 反向操作）
  ③ if (Binlog 里有该 XID && Redo 里是 PREPARE)：
       → Binlog 有了说明客户端收到了 "OK"
       → Binlog 有了也说明 Slave 会执行这个事务
       → 提交（Redo Log 补上 COMMIT 标记 + 应用到数据页）
  ④ if (Redo 里已经是 COMMITTED)：
       → 正常提交完成，不需要额外动作
```

**这就是为什么"双 1 配置缺一不可"：**

```bash
# ✅ 安全配置
innodb_flush_log_at_trx_commit = 1   # 每次提交时 Redo MUST 在磁盘
sync_binlog = 1                       # 每次提交时 Binlog MUST 在磁盘

# 如果 sync_binlog = 0：
#   步骤④ 的 Binlog 可能在 OS 缓存中还没到磁盘
#   断电 → Binlog 丢了 → 崩溃恢复误判 → 回滚了"客户端认为已提交"的事务
#   → 主从数据不可能一致

# 如果 innodb_flush_log_at_trx_commit = 0：
#   步骤② 的 Redo 可能在 OS 缓存中
#   断电 → Redo PREPARE 标记丢了 → 这个事务的页面修改在 Buffer Pool 中也没了
#   → 客户端收到了 "OK" → 但数据不见了！
```

---

# 六、崩溃恢复的完整流程

MySQL 启动时的后台恢复步骤：

```
1. 扫描 Redo Log，从上次 Checkpoint LSN 开始，重放所有日志到最新
   → 此时数据页恢复到 crash 前一刻的状态
   → 包括已提交和未提交事务的修改都在数据页里了

2. Undo 阶段：扫描 Undo Log，找出所有 TRX_ACTIVE（未提交）的事务
   → 对每个未提交事务，用其 Undo Log 反向操作，回滚数据页

3. 对每个 TRX_PREPARED 的事务（2PC 挂起的事务）：
   → 去查 Binlog 判断该提交还是回滚（见 5.3 节）

4. 恢复完成 → 数据库可以接受连接
```

```sql
-- 监控：上次崩溃恢复的信息
SHOW ENGINE INNODB STATUS\G
-- 在 LOG 段：
-- "Log sequence number xxxx ... Last checkpoint at yyyy"
-- 如果 xxxx >> yyyy → 说明 checkpoint 落后很多 → 下次崩溃恢复可能很慢
```

**为什么 checkpoint 不能太频繁？**
checkpoint 意味着把脏页刷到磁盘（随机 IO）。太频繁 → 写性能下降。太不频繁 → Redo Log 容易写满（write pos 追上 checkpoint pos）→ 而且崩溃后需要重放太多 Redo → 恢复时间过长。

---

# 七、总结

| 日志 | 层级 | 格式 | 用途 | 关键配置 |
|------|------|------|------|---------|
| **Redo Log** | InnoDB 引擎层 | 物理日志（页内偏移+内容） | 崩溃恢复，保证持久性 | `innodb_flush_log_at_trx_commit=1` |
| **Undo Log** | InnoDB 引擎层 | 逻辑日志（反向操作） | 事务回滚 + MVCC 版本链 | Purge 调度（`innodb_purge_threads`） |
| **Binlog** | Server 层 | 逻辑日志（SQL/ROW） | 主从复制 + 数据恢复 | `sync_binlog=1` |
| **2PC** | 两层之间 | 协议（XID 关联） | 保证 Redo 和 Binlog 一致 | 依赖双 1 配置 |
| **Doublewrite** | InnoDB 磁盘 | 页副本 | 防止页断裂 | `innodb_doublewrite=ON` |

# 延伸阅读

**Do——动手验证：**
- `mysqlbinlog --base64-output=decode-rows -v binlog.000001` 查看一次 UPDATE 在 ROW 格式下产生的完整 binlog 记录
- 在测试库上模拟崩溃恢复：`kill -9 <mysqld-pid>` → 重启 MySQL → `SHOW ENGINE INNODB STATUS` 观察恢复日志
- 对比 `innodb_flush_log_at_trx_commit=1` vs `=0` 的 `sysbench` 写入 sysbench 结果

**Todo——深入方向：**
- MTR（Mini-Transaction）的内部并发控制——多个 MTR 如何并发写同一个 Redo Log Buffer
- MySQL 8.0 的 Redo Log 禁用（`ALTER INSTANCE DISABLE INNODB REDO_LOG`）——什么场景可以用？
- Binlog 的加密（`binlog_encryption=ON`）和压缩（`binlog_transaction_compression=ON`，8.0.20+）
- 逻辑时钟（Logical Clock）在并行复制中的应用——每个事务的 `sequence_number` 如何从 Binlog 传递到 Slave

*本文参考资料：*
- MySQL 官方文档: InnoDB Redo Log / Undo Log / Binary Log
- Jeremy Cole, "The InnoDB Redo Log" / "InnoDB Crash Recovery"
- 姜承尧《MySQL 技术内幕：InnoDB 存储引擎》——第 3 章（日志文件）
- MySQL 源码 `storage/innobase/log/`（Redo Log 实现）
