# Canal 原理与实践——基于 MySQL binlog 的数据变更捕获与分发

> 缓存一致性、多存储同步、实时数仓、数据审计——这些场景都有一个共同需求：**知道数据库里发生了什么变更**。传统方案是在应用层埋点、发消息、双写——侵入代码且无法保证一致性。Canal 换个思路，把自己伪装成 MySQL Slave，"偷听" binlog，把数据库变成事件源。

---

## 一、Canal 解决了什么问题

### 1.1 缓存双写困境

上一篇文章《分布式系统面试四连问》详细分析了 Cache-Aside 模式——更新 DB 后删除缓存的极端时序窗口可能导致数据不一致。这个问题的根因是：**应用层同时操作 DB 和缓存，无法保证原子性。**

Canal 的思路彻底不同：应用层只管更新 DB，完全不碰缓存。Canal 监听 binlog，发现变更 → 异步删除 Redis。DB 和 Redis 的操作被完全解耦，应用代码里一行缓存操作都不用写。

### 1.2 多存储数据同步

当数据需要同时存在 MySQL、ES、Redis、Mongo 中时，传统方案是应用层在事务中做双写或多写。问题很明显：一旦链路变长，任何一个目标存储失败，整个链路的一致性就崩了。

Canal 的方案：MySQL 是唯一真相源。所有变更通过 binlog 流出 → Kafka → 各存储系统独立消费。写路径极短（只写 MySQL），读路径各自优化。

### 1.3 实时数仓与审计

实时数据同步到数仓（MySQL → Kafka → Flink → ClickHouse/StarRocks）、数据变更审计（谁、什么时间、改了哪个字段、旧值是什么）——这些需求本质上都是"我需要实时感知数据库的变更"。

---

## 二、MySQL binlog 基础——理解 Canal 的前提

### 2.1 binlog 是什么

binlog (Binary Log) 是 MySQL Server 层记录的**所有数据变更**的二进制日志。注意是 Server 层，不是存储引擎层——所以 InnoDB 的 redo log 和 binlog 是完全不同的东西：

| | redo log | binlog |
|---|---------|--------|
| 层面 | InnoDB 引擎层 | MySQL Server 层 |
| 记录内容 | 物理日志：对数据页的物理修改 | 逻辑日志：SQL 语句或行数据变更 |
| 作用 | crash-safe：崩溃恢复 | 主从复制、数据恢复、CDC |
| 写入方式 | 循环写（固定大小，写满覆盖） | 追加写（文件写满后切新文件） |

### 2.2 三种 binlog 格式——Canal 为什么只能用 ROW

Canal 依赖 binlog ROW 模式。`binlog_format` 三种格式的区别：

**STATEMENT 模式**：记录执行的 SQL 语句原文。

```
SET @@session.binlog_format=STATEMENT;
UPDATE user SET age = age + 1 WHERE id = 1;
-- binlog 记录: UPDATE user SET age = age + 1 WHERE id = 1
```

问题很明显：`age = age + 1` 这条 SQL 在 Slave 上重放时，如果 Slave 的 age 跟 Master 不一致，结果就错了。还有 `NOW()`、`UUID()` 这类非确定性函数，每次执行结果不同。STATEMENT 下主从数据可能不一致，所以生产基本不用。

**ROW 模式**：记录每行数据的实际变更。

```
UPDATE user SET age = 25 WHERE id = 1;
-- binlog 记录:
--   TableMap: test.user (id INT, name VARCHAR, age INT)
--   UpdateRows: before_image={id=1, age=24} → after_image={id=1, age=25}
```

每行变更被精确记录：改了哪一行、改之前是什么、改之后是什么。Canal 拿到这些精确信息才能做后续处理。

**MIXED 模式**：默认 STATEMENT，非确定性语句自动切 ROW。Canal 场景不适用——你不知道什么时候切 ROW，且 Canal 依赖 ROW 的稳定格式。

**关键配置**：

```ini
# my.cnf
[mysqld]
binlog_format = ROW
binlog_row_image = FULL  # 必须！MINIMAL 只记录变更字段，Canal 拿不到完整行
log-bin = mysql-bin       # 开启 binlog
server-id = 1             # 必须唯一
```

`binlog_row_image = FULL` 是关键——如果设为 MINIMAL，UpdateRows 只包含变更的列（age），没有 id 等其他列。Canal 下游按"以 id 为主键处理这行数据"的逻辑就断了——比如同步到 ES，你只知道 age 变了，不知道这行对应的 doc id，因为 id 列没在变更记录里。

### 2.3 binlog 事件类型

ROW 模式下，一次 DML 操作产生 2-3 条事件：

```
BEGIN
  TableMap:    描述表结构（列名、类型）        → 一次 DML 只产生一条 TableMap
  WriteRows:   每行 insert 数据                 → 多行 = 多条
  Xid:         事务提交
```

INSERT、UPDATE、DELETE 的行事件分别是 `WriteRowsEvent`、`UpdateRowsEvent`、`DeleteRowsEvent`。UpdateRows 包含 **before_image** 和 **after_image** 两份数据——"改之前"和"改之后"。这是 Canal 能做数据审计的根本原因。

**DDL 事件**（`ALTER TABLE`、`CREATE INDEX` 等）也会进入 binlog。Canal 解析到 DDL 会发送一个特殊类型的 CanalEntry，但 DDL 不包含行数据——需要下游代码做特殊处理。

### 2.4 binlog 的位点

两个维度定位 binlog 中的位置：

**文件 + 偏移量**：
```
mysql-bin.000003:4567890
```
表示第 3 个 binlog 文件的第 4567890 字节处。主从切换时，不同 Master 的 binlog 文件命名不同——所以这个定位方式在主从切换后会失效。需要手动在 MHA/Orchestrator 的回调脚本中告知 Canal 新 Master 的 binlog 位置。

**GTID (Global Transaction ID)**：
```
3E11FA47-71CA-11E1-9E33-C80AA9429562:1-100
```
每个事务全局唯一标识，由 server_uuid + 递增序号组成。主从切换时 GTID 不变，Canal 直接拿上次消费到的 GTID 去新 Master 续消费——这才是生产环境的正确姿势。

---

## 三、Canal 的核心原理——伪装成 MySQL Slave

### 3.1 整体架构

```
┌───────────────────────────────────────────┐
│              Canal Server (JVM)            │
│                                           │
│  ┌─────────────────────────────────────┐  │
│  │         Canal Instance               │  │
│  │  ┌──────────┐  ┌──────┐  ┌───────┐  │  │
│  │  │  Parser  │→ │ Sink │→ │ Store │  │  │
│  │  │ 解析binlog│  │过滤归并│  │环形缓冲│  │  │
│  │  └──────────┘  └──────┘  └───┬───┘  │  │
│  └───────────────────────────────┼──────┘  │
│                                  │         │
└──────────────────────────────────┼─────────┘
                                   │
                    ┌──────────────┼──────────────┐
                    ▼              ▼              ▼
               TCP Client   Kafka Producer   RocketMQ
               (原生协议)    (生产常用)      (阿里系)
```

关键概念：
- **Canal Server**：一个 JVM 进程，可以管理多个 Canal Instance
- **Canal Instance**：一个 binlog 订阅任务，对应一个 MySQL 实例（或一个数据库）
- **EventParser**：打开 binlog 连接，解析二进制流为 Java 对象
- **EventSink**：过滤（只看指定库表）、归并（将 TableMap 和 RowData 合并为一条完整记录）
- **EventStore**：RingBuffer 存储解析后的事件，提供 get/ack/rollback 的消费语义
- **HA 机制**：基于 ZooKeeper（或 MySQL），同一 Instance 同一时刻只有一个 Server 活跃。发生故障时，各 Server 竞争 Instance 的 ZK 锁——谁拿到锁谁接管。ZK 中同时保存消费位点，接管后从位点继续

### 3.2 伪装成 MySQL Slave——最巧妙的设计

MySQL 主从复制的协议是公开的。Canal 利用这个协议，把自己伪装成一个从库：

```
Canal 启动 → ① 连接 MySQL，发送 COM_BINLOG_DUMP 命令
                 参数: binlog_file, binlog_position
             → ② MySQL 把 Canal 当成 Slave
                 将 binlog 事件流推送给 Canal
             → ③ Canal 持续接收事件流
                 Master 有新数据写入 → 立刻推给 Canal
```

这个过程和真正的 MySQL 主从复制完全一致——MySQL 无法区分对方是真的 Slave 还是 Canal。优势在于：不需要在 MySQL 上安装任何插件或 Agent（不像 Debezium 基于 Kafka Connect 需要在 Kafka 和 MySQL 之间额外部署 Worker），只需要给 Canal 一个拥有 REPLICATION SLAVE 权限的账号。

```sql
CREATE USER 'canal'@'%' IDENTIFIED BY 'canal';
GRANT SELECT, REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'canal'@'%';
FLUSH PRIVILEGES;
```

**EventParser 内部的工作**：
1. 连接到 MySQL，发送 `COM_BINLOG_DUMP`
2. 接收 binlog 二进制流
3. 解析 binlog event header（事件类型、时间戳、server_id）
4. 根据事件类型调用对应解析器：`WriteRowsEventParser`、`UpdateRowsEventParser`、`DeleteRowsEventParser`
5. 解析后的内部对象 → `CanalEntry.Entry`（Protobuf 封装）

### 3.3 EventSink —— 这个阶段的核心价值

Parser 产出的原始 Entry 是零散的：TableMap 和 RowData 是分开的 Event。Sink 做的事：

**归并**：将 TableMap + RowData 合并。例如 UpdateRows 事件 + 之前缓存的 TableMap → 完整记录 `{table: user, columns: [id, name, age], before: {id=1, age=24}, after: {id=1, age=25}}`。归并不是简单的字节拼接——TableMap 携带了列名和类型信息，归并时按 TableMap 给 RowData 的每个字段赋值列名，下游才能用 `row.get("age")` 而不是 `row.get(2)`。

**过滤**：`canal.instance.filter.regex = .*\\..*`（正则匹配 db.table）。生产环境要精确指定，避免全库全表的 binlog 流量冲击。

**路由**：一个 Instance 只连一个 MySQL，但如果下游有 Kafka，可以通过配置将不同表的事件路由到不同的 Kafka Topic：

```
canal.mq.dynamicTopic = true
canal.mq.topic = example
# user 表 → example_user topic
# order 表 → example_order topic
```

### 3.4 EventStore —— RingBuffer 的 get/ack/rollback

EventStore 使用 **Disruptor RingBuffer** 存储事件。消费模型：

```
get(batchSize)     → 获取 N 条事件（不删除，可重复读取）
ack(batchId)       → 确认消费完成，RingBuffer 移动读取指针
rollback(batchId)  → 回滚到指定 batch，下次 get 重新读
```

这个设计的好处：消费者失败时可以不丢数据、重新处理。但 bad case 是如果消费者永远不 ack——RingBuffer 满了之后 Parser 阻塞，Canal 停止拉取 binlog。生产环境的 Canal 延迟很多就是这个原因。

---

## 四、部署模式与消费方式

### 4.1 单机 vs 集群

**单机**：开发测试，一个 Canal Server、一个 Instance、直连 MySQL。

**集群**：生产环境，多个 Canal Server + ZooKeeper 协调。同一 Instance 只有一个 Server 活跃——其他 Server 监听 ZK 节点作为热备，主 Server 故障时竞争锁接管。这个设计简单有效——既保证了高可用，又避免了"多 Server 同时解析同一 binlog 位点"的冲突。

### 4.2 四种消费出口

```
Canal Server
    │
    ├── TCP Client (原生 Protocol Buffers)
    │       │
    │       └── 自定义 Java 客户端
    │
    ├── Kafka (生产最常用)
    │       │
    │       └── 解析 → JSON → 投递到 Kafka topic
    │
    ├── RocketMQ
    │       │
    │       └── 阿里系标配
    │
    └── Canal Adapter
            │
            ├── ES Adapter (MySQL → ES)
            ├── HBase Adapter
            ├── Redis Adapter
            └── RDB Adapter (MySQL → 另一 MySQL)
```

**为什么生产环境一律走 Canal → Kafka？**

TCP Client 有两个致命问题：
- 耦合 Canal Server——消费者必须跟 Canal 部署在一起或用专线，无法独立扩缩容
- 消费组不支持——只有一个消费者能消费一个 Instance，不能多消费者并行
- 消费位点存 Canal 内存——Canal 挂了消费位点就丢了

Kafka 天然解决了这三个问题：多消费组各自独立消费、消费位点永久持久化、消费者可以用任何语言、可以独立扩容。

### 4.3 Canal → Kafka 的配置

```properties
# canal.properties
canal.serverMode = kafka
kafka.bootstrap.servers = kafka1:9092,kafka2:9092

# instance.properties
canal.mq.topic = mysql_binlog

# 动态 Topic：不同表 → 不同 Topic
canal.mq.dynamicTopic = true

# 消息格式：FlatMessage（扁平化 JSON，易消费）
canal.mq.flatMessage = true
```

FlatMessage 格式：

```json
{
  "id": 12345,
  "database": "ecommerce",
  "table": "order",
  "type": "UPDATE",
  "ts": 1751412000000,
  "sql": "",
  "data": [{"id": "1001", "status": "paid", "amount": "99.00"}],
  "old": [{"status": "pending"}],
  "es": 1751412000000
}
```

`type` 字段是 `INSERT` / `UPDATE` / `DELETE`，消费者根据它决定操作类型。`old` 字段只在 UPDATE 和 DELETE 时有值（DELETE 时 `data` 为空，`old` 有删除前的行数据）——这里有个经验性的坑：`old` 在非 ROW 模式或 binlog_row_image 配置错误时可能为空，但 SQL 解析模式下"老值是什么"是算不出来的，所以必须确保 ROW+FULL。

---

## 五、核心实战场景

### 5.1 缓存一致性——Canal 的经典场景

架构对比：

```
传统 Cache-Aside（应用层双写）:
  App → ① 更新 MySQL
     → ② 删除 Redis
  → ①和②之间没有原子性保证 → 存在微小但真实的不一致窗口

Canal 方案:
  App → 更新 MySQL（只这一件事）
  Canal 监听 binlog → 发现 user 表变更 → 发 Kafka
  Consumer 消费 → 删除 Redis 对应 key
  → App 什么都不用做，缓存自动失效
```

Java 消费端实现：

```java
@Component
public class CacheInvalidationConsumer {
    
    @Autowired
    private RedisTemplate<String, Object> redisTemplate;
    
    @KafkaListener(topics = "mysql_binlog_user", groupId = "cache-invalidator")
    public void onMessage(String messageJson) {
        FlatMessage msg = JSON.parseObject(messageJson, FlatMessage.class);
        
        if (!"user".equals(msg.getTable())) return;
        
        String type = msg.getType();
        for (Map<String, String> row : msg.getData()) {
            String userId = row.get("id");
            // 删除该用户的所有缓存 key
            String cacheKey = "user:" + userId;
            redisTemplate.delete(cacheKey);
            
            // 如果 UPDATE 影响了关联字段，也删相关缓存
            if ("UPDATE".equals(type) && row.containsKey("role_id")) {
                redisTemplate.delete("user_role:" + row.get("role_id"));
            }
        }
    }
}
```

注意：消费者不重新构建缓存，只删除。重建责任交给下一次读请求（Cache-Aside 的读路径）。为什么不在消费端重建？因为重建需要查 DB，而消费者的职责应该是轻量的——不应该在 binlog 消费链路中引入 DB 查询，否则会把 binlog 的吞吐优势直接拉下来。

### 5.2 MySQL → ES 搜索引擎同步

全量 + 增量双链路：

```
启动时:
  1. DataX / 自定义 scan 程序：SELECT * FROM order → 批量 write ES
  2. 记录 scan 开始时的 binlog position
  3. Canal 从该 position 开始消费增量变更

运行中:
  Canal → Kafka → ES Consumer:
    INSERT → ES.index(doc)
    UPDATE → ES.update(doc)   // 部分更新
    DELETE → ES.delete(doc)
```

Java 消费端实现：

```java
@KafkaListener(topics = "mysql_binlog_order", groupId = "es-syncer")
public void syncToES(String messageJson) {
    FlatMessage msg = JSON.parseObject(messageJson, FlatMessage.class);
    
    switch (msg.getType()) {
        case "INSERT":
        case "UPDATE":
            for (Map<String, String> data : msg.getData()) {
                IndexRequest request = new IndexRequest("order_index")
                    .id(data.get("id"))
                    .source(data, XContentType.JSON);
                bulkProcessor.add(request);
            }
            break;
            
        case "DELETE":
            for (Map<String, String> old : msg.getOld()) {
                DeleteRequest request = new DeleteRequest("order_index", old.get("id"));
                bulkProcessor.add(request);
            }
            break;
    }
}
```

UPDATE 时为何用 data 而非 old+data 拼接：ES 的 `IndexRequest` + 指定 `id` 就是 upsert 语义（有则覆盖，无则插入）。data 已经是这行的完整最新状态（binlog_row_image=FULL），不需要跟 old 对比——直接覆盖写即可。

### 5.3 数据变更审计

监听所有敏感表，记录每一次变更：

```java
@KafkaListener(topics = "mysql_binlog_order", groupId = "audit-logger")
public void auditLog(String messageJson) {
    FlatMessage msg = JSON.parseObject(messageJson, FlatMessage.class);
    
    for (int i = 0; i < msg.getData().size(); i++) {
        AuditLog log = AuditLog.builder()
            .tableName(msg.getTable())
            .rowId(msg.getData().get(i).get("id"))
            .operationType(msg.getType())
            .newData(JSON.toJSONString(msg.getData().get(i)))
            .oldData(msg.getOld() != null ? 
                     JSON.toJSONString(msg.getOld().get(i)) : null)
            .timestamp(new Date(msg.getTs()))
            .build();
        auditLogService.save(log);  // 写入审计表或 Kafka topic
    }
}
```

---

## 六、生产避坑

### 6.1 位点管理是命门

消费位点丢失 = Canal Server 重启后从 binlog 起点（`mysql-bin.000001:4`）重新消费 = 海量历史数据瞬间涌入下游 = 下游被打爆。

**你必须持久化消费位点**：

- Canal Server 侧：ZK 集群（或 MySQL 表）存储各 Instance 的消费位点
- 如果走 Kafka：消费位点天然在 Kafka 的 `__consumer_offsets` 中。Canal Server 挂了重启无所谓——消费者从上次提交的 offset 续消费。这也是 Kafka 出口优于 TCP Client 的关键原因
- 如果走 RocketMQ：同理
- 如果走原生 TCP：你必须在 Client 端把 batchId/position 刷到外部存储

### 6.2 大事务

一个大事务 UPDATE 几百万行，MySQL 事务提交后这些 binlog 事件需要在 Canal 内部一次性处理。内存压力极大：

- EventParser 缓冲：解析出的几百万行 RowData → 堆内存
- EventSink 归并：每行都要关联 TableMap → 大量临时对象
- EventStore RingBuffer：默认 16384 个 slot → 大事务的行数可能远超 slot 数

这是 Canal 当前的工程局限——它假设 binlog 事件可以按"行"为粒度处理，大事务打破了这一点。生产环境的缓解办法不多，更多的是预防：业务侧避免在单事务中操作超过 10 万行；或者定期检查 `INNODB_TRX` 表看是否有长时间未提交的大写事务。

### 6.3 DDL 变更

`ALTER TABLE user ADD COLUMN phone VARCHAR(20)` 导致的问题：

- TableMap 变更：`user` 表的列从 `[id, name, age]` 变成 `[id, name, age, phone]`
- 下游如果按列索引取值（`row.get(2)`）→ `phone` 是第 4 列 → 下标错位
- 如果按列名取值（`row.get("phone")`）→ 新列只在 DDL 后的 RowData 中出现 → 老 INSERT/UPDATE 事件的 data 中没有 "phone" → `row.get("phone")` 返回 null

所以必须：下游消费代码一律按列名取值。并在发布 DDL 时通知下游团队。DDL 事件本身也会被 Canal 捕获——你可以监听 DDL 类型事件，自动发告警。

### 6.4 MySQL 主从切换

Master 宕机 → 新 Master 的 binlog 文件序列重新开始 → file+position 定位失效。

**必须用 GTID 模式**：

```properties
# instance.properties
canal.instance.master.address = 192.168.1.100:3306
canal.instance.master.journal.name =    # 留空，用 GTID
canal.instance.master.position =        # 留空
canal.instance.gtid = true              # 开启 GTID 模式
```

Canal 在主从切换后自动连接到新 Master，从上次消费的 GTID 位置继续。

### 6.5 监控

三个核心指标：

| 指标 | 含义 | 告警阈值 |
|------|------|---------|
| 消费延迟 | 事件产生时间 vs Canal 消费时间的差值 | > 5s 告警 |
| RingBuffer 使用率 | Store 中待消费的事件占比 | > 80% 告警 |
| GC 停顿 | Canal Server 的 GC 时间 | > 500ms 告警 |

消费延迟是最关键的——一旦延迟持续增长，说明消费端处理不过来。排查方向：下游消费代码慢（查 ES/写 DB 的 RPC 耗时）、Kafka 消费者不足、或大事务导致一次性涌入海量事件。

### 6.6 配置检查清单

```properties
# 必须验证的 MySQL 配置
SHOW VARIABLES LIKE 'binlog_format';       -- 必须是 ROW
SHOW VARIABLES LIKE 'binlog_row_image';    -- 必须是 FULL
SHOW VARIABLES LIKE 'log_bin';             -- 必须是 ON

# Canal Instance 配置
canal.instance.filter.regex = ecommerce\\.order,ecommerce\\.user  -- 精确指定，避免全库
canal.instance.gtid = true                                         -- 生产必须
canal.mq.flatMessage = true                                        -- 扁平 JSON 优于 Protobuf
```

### 6.7 常见错误排查

**启动后没有收到任何事件**：先确认 MySQL binlog 是否开启且格式为 ROW。再看 filter 是否正确——很多情况是 filter 正则匹配错了库名或表名。

**下游消费到的事件 data 字段缺少列**：检查 `binlog_row_image = FULL` 是否生效（某些 MySQL 版本的某些情况下，即使配了 FULL 也可能降级为 MINIMAL）。

**Canal 频繁重连**：MySQL 的 `wait_timeout` 设得太短。Canal 长时间没有事件时连接空闲，MySQL 主动掐断。需要调大 `wait_timeout` 或在 Canal 侧配置心跳保活。

**消费延迟持续增长**：大概率是下游消费太慢。先看消费者 GC 情况，再看消费逻辑中的外部调用耗时——最常见的问题是消费端写 ES/Redis 时没用批量写入，逐条刷导致吞吐极低。

---

## 七、与其他 CDC 方案对比

| 方案 | 数据源 | 消费协议 | 输出 | 运维复杂度 |
|------|--------|---------|------|-----------|
| Canal | 仅 MySQL | 伪装 Slave | TCP/Kafka/RocketMQ | 中 |
| Debezium | MySQL/PG/Mongo/Oracle | Kafka Connect | Kafka | 中高 |
| Flink CDC | MySQL/PG/Mongo | 伪装 Slave | Flink DataStream | 中（依赖 Flink） |
| Maxwell | 仅 MySQL | 伪装 Slave | Kafka | 低 |
| Databus | 仅 MySQL | 伪装 Slave | 自定义 | 高 |

**Canal vs Debezium 的选择**：
- Debezium 的优势是**数据库覆盖面**和 **Kafka Connect 生态标准化**——单表变更自动路由到 `{server}.{db}.{table}` 结构的 topic，schema 变更自动注册到 Schema Registry
- Canal 的优势是**阿里技术栈集成**——跟 RocketMQ、Nacos 天然配合，Adapter 直接对接 ES/HBase，国内文档和社区丰富
- 如果你的基础设施不在阿里云，且需要支持 Postgres 等多种数据源 → Debezium 更合适
- 如果已经是阿里云全家桶，且只需要 MySQL → Canal 是原配

**Flink CDC** 是最近的强势方案——它把 binlog 捕捉和流计算合二为一，不需要中间 Kafka。但 Flink CDC 目前 3.x 版本的稳定性还待验证，Canal 在稳定性方面有更长的生产验证历史。

---

## 八、总结

Canal 的核心价值不在于"能做数据同步"——写个定时任务扫描 `updated_at` 也能做——而在于：

- **对业务代码零侵入**：应用层只管写 MySQL，缓存更新、ES 同步、审计记录全部由 Canal + 下游消费完成
- **变更捕获的实时性和完整性**：binlog 是数据库的原生能力，不存在"漏扫"或"扫描延迟"的问题
- **将数据库变成事件源**：Canal 连接了"数据库的世界"和"消息队列的世界"，让 MySQL 的每一次变更成为 Kafka 上的一条消息——从此，任何需要"感知数据变更"的系统都可以订阅 Kafka，而不需要侵入应用的业务代码

架构上唯一增加的复杂度是 Canal Server 本身的运维——位点、延迟、大事务、DDL——但只要配好监控和告警，这些是可控的。
