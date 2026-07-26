---
title: "分库分表——水平拆分策略与 ShardingSphere 实战"
date: 2026-06-28
description: 从垂直拆分与水平拆分的选型依据、分片键与分片算法的设计方法论（Hash/一致性Hash/虚拟槽）、ShardingSphere-JDBC vs Proxy 的架构选型，到分库分表后分布式主键、跨分片分页、分布式事务的解决方案。
tags: ["MySQL","分库分表","ShardingSphere","分布式","水平拆分"]
categories: ["MYSQL"]
---

# 历史背景——单表多大该拆？

行业里流传着一条经验："MySQL 单表 5000 万行就该拆了"。这条经验的物理依据是：InnoDB 的 B+Tree 在 3 层时约存 2000 万行（主键 BIGINT + 行 1KB），4 层时约 1 亿行。每多一层，每次索引查询多一次随机 IO → 从亚毫秒级退化到数毫秒级。

但 **"该拆"不是因为 MySQL 装不下了，而是因为运维开始痛苦了**：
- 备份：一张 500GB 的表跑 `mysqldump` 可能要几个小时
- DDL：`ALTER TABLE ADD COLUMN` 500GB 表 → 拷贝整表 → 可能锁几小时
- 查询：虽然单行查询仍快（B+Tree 4 层也就 4 次 IO），但范围扫描和 Join 的效果会显著变差

分库分表解决的不是"能不能"的问题，而是"运维起来有多痛苦"的问题。

---

# 一、垂直拆分 vs 水平拆分

## 1.1 垂直拆分——按业务域拆库

```
微服务架构的标准模式：每个服务一个独立数据库

┌──────────┐  ┌──────────┐  ┌──────────┐
│ 用户服务   │  │ 订单服务   │  │ 商品服务   │
│ user_db   │  │ order_db  │  │ product_db│
└──────────┘  └──────────┘  └──────────┘

优点：服务解耦、独立扩缩、故障隔离
缺点：订单要关联用户信息？ → 以前一个 JOIN，现在需要两次 RPC
```

## 1.2 水平拆分——按行分表

```
单表 5000 万行 → 拆成 10 张表，每表 500 万行

order_0  (id % 10 = 0)
order_1  (id % 10 = 1)
...
order_9  (id % 10 = 9)

进一步可以跨库:
db_0: order_0, order_1, order_2, order_3, order_4
db_1: order_5, order_6, order_7, order_8, order_9
```

```sql
-- 分库分表后不能这样写了：
SELECT * FROM orders WHERE user_id = 123;
-- 你不知道 order_0 ~ order_9 哪个是这个用户的数据 → 需要分片中间件路由

-- 也不能跨分片 JOIN：
SELECT * FROM orders o JOIN order_items i ON o.id = i.order_id;
-- 如果 o 和 i 不在同一个分片 → 需要中间件拉数据到内存做关联
```

---

# 二、分片键——最重要的设计决策

## 2.1 分片键选错了意味着什么？

分片键一旦确定，后续几乎无法更改。因为改了意味着**所有数据需要按新规则重新分布**——全量数据迁移。

## 2.2 选择原则

```java
// 原则 1：大多数查询条件都带这个字段
// ✅ user_id：查订单、查收货地址、查收藏 → 99% 的查询都带 user_id
// ❌ order_id：查一个订单时可以，但查"用户的所有订单"就需要扫描所有分片

// 原则 2：数据分布均匀
// ✅ user_id（Hash 后）：均匀分布到各分片
// ❌ create_time：最新一天的订单全在同一个分片 → 热点分片 → "拆了等于没拆"

// 原则 3：不可变
// ✅ order_id：一旦创建不会改变
// ❌ status：订单状态会变 → 状态变了数据要迁移到另一个分片 → 不可能在线完成
```

**典型案例分析**：
```sql
-- 场景：订单表，最常见的查询是"查用户的所有订单"
-- 分片键选 user_id → 大部分查询只走一个分片 → 优秀
-- 分片键选 order_id → 查用户订单需要广播到所有分片 → 灾难

-- 但！如果有"按日期查所有订单"的运维需求呢？
-- user_id 分片 → 查日期需要扫描所有分片 → 也灾难
-- 解法：建一张按日期分片的"订单索引表"（只存 order_id + 日期）
--       或者把"按日期查"的操作走 ES/Hive 等分析型系统
```

---

# 三、分片算法

## 3.1 Hash 取模——最均衡但扩容灾难

```java
// 优点：数据分布最均匀
shard = hash(shard_key) % 4;  // 4 个分片

// 扩容：4 → 5 个分片
// hash % 4 → hash % 5
// hash=100: 100%4=0(分片0) → 100%5=0(分片0) ✓ 刚好不动
// hash=101: 101%4=1(分片1) → 101%5=1(分片1) ✓ 刚好不动
// hash=102: 102%4=2(分片2) → 102%5=2(分片2) ✓ 刚好不动
// hash=103: 103%4=3(分片3) → 103%5=3(分片3) ✓ 刚好不动
// hash=104: 104%4=0(分片0) → 104%5=4(分片4) ✗ 需要迁移！
// ...
// 迁移比例 = (N_new - N_old) / N_new = 1/5 = 20% ... 实际上远大于此
// 因为 hash % 4 → hash % 5 时，大部分数据的分片编号都会变
// 实际迁移量 ≈ (1 - old_count/new_count) × 总数据量
// 4→5: 约 80% 的数据需要迁移！不是 20%！
```

## 3.2 一致性 Hash——扩容友好但不够均匀

```java
// 环形哈希空间：[0, 2^32-1]
// 4 个节点在环上，数据按 hash 值顺时针走到最近的节点

// 扩容（加一个节点）：
// 新节点 N5 插入环中 → 只有 N5 和它顺时针方向的下一个节点之间的数据需要迁移
// 迁移量 ≈ 1 / (N+1) 的总数据量 → 5 个节点 ≈ 1/5 = 20%

// 问题：节点数少时分布可能很不均匀（有的节点占 40% 数据，有的只占 5%）
// 解法：虚拟节点 → 每个物理节点映射为 150 个虚拟节点分布在环上 → 基本均匀
```

## 3.3 ShardingSphere 的内置算法

```yaml
# ShardingSphere 配置示例（YAML 格式）
rules:
  - !SHARDING
    tables:
      orders:
        actualDataNodes: ds_${0..1}.orders_${0..4}  # 2 库 × 5 表 = 10 分片
        databaseStrategy:
          standard:
            shardingColumn: user_id
            shardingAlgorithmName: database_inline
        tableStrategy:
          standard:
            shardingColumn: user_id
            shardingAlgorithmName: table_inline
    
    shardingAlgorithms:
      database_inline:
        type: INLINE
        props:
          algorithm-expression: ds_${user_id % 2}   # 按 user_id 分库
      table_inline:
        type: INLINE
        props:
          algorithm-expression: orders_${user_id % 5}  # 按 user_id 分表
```

---

# 四、ShardingSphere 实战

## 4.1 JDBC vs Proxy 架构对比

```
ShardingSphere-JDBC:
  应用 → (内嵌 ShardingSphere-JDBC jar) → 改写 SQL → 直连各 MySQL 分片
  优点：无额外部署，性能最高（无中间网络转发）
  缺点：只支持 Java，升级需重新发布应用

ShardingSphere-Proxy:
  应用 → ShardingSphere-Proxy (独立进程，伪装成 MySQL) → 各 MySQL 分片
  优点：语言无关（Go/Python/Java 都能连）
  缺点：多一层网络延迟（~0.1-0.3ms），需要维护 Proxy 集群本身的 HA
```

**选型建议**：纯 Java 微服务 → JDBC；多语言环境或不能改代码（接入已有 MySQL 客户端）→ Proxy。

## 4.2 Spring Boot 集成 ShardingSphere-JDBC

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.apache.shardingsphere</groupId>
    <artifactId>shardingsphere-jdbc-core</artifactId>
    <version>5.4.0</version>
</dependency>
```

```yaml
# application.yml
spring:
  shardingsphere:
    datasource:
      names: ds0,ds1
      ds0:
        type: com.zaxxer.hikari.HikariDataSource
        driver-class-name: com.mysql.cj.jdbc.Driver
        jdbc-url: jdbc:mysql://10.0.1.1:3306/order_db_0
      ds1:
        type: com.zaxxer.hikari.HikariDataSource
        driver-class-name: com.mysql.cj.jdbc.Driver
        jdbc-url: jdbc:mysql://10.0.1.2:3306/order_db_1
    
    rules:
      sharding:
        tables:
          orders:
            actual-data-nodes: ds$->{0..1}.orders_$->{0..4}
            database-strategy:
              standard:
                sharding-column: user_id
                sharding-algorithm-name: db-sharding
            table-strategy:
              standard:
                sharding-column: user_id
                sharding-algorithm-name: table-sharding
        
        sharding-algorithms:
          db-sharding:
            type: INLINE
            props:
              algorithm-expression: ds$->{user_id % 2}
          table-sharding:
            type: INLINE
            props:
              algorithm-expression: orders_$->{user_id % 5}
    
    props:
      sql-show: true  # 开发环境可开启，显示实际执行的 SQL
```

```java
// 业务代码：完全不用关心分片！按普通 JPA/MyBatis 写就行
@Repository
public class OrderRepository {
    @Autowired
    private JdbcTemplate jdbcTemplate;
    
    public List<Order> findUserOrders(Long userId) {
        // ShardingSphere 自动根据 user_id 路由到正确的分片
        return jdbcTemplate.query(
            "SELECT * FROM orders WHERE user_id = ?",
            new Object[]{userId},
            new OrderRowMapper()
        );
    }
}
```

## 4.3 广播表与绑定表

```yaml
# 广播表（每个分片存一份全量数据，如字典表）
rules:
  sharding:
    broadcastTables: t_config, t_district
    
# 绑定表（避免笛卡尔积 Join）
rules:
  sharding:
    bindingTables:
      - orders, order_items  # 这两张表按相同分片键分片 + 分片算法一致
      # 这样 orders JOIN order_items → 只在一个分片内完成，不需要跨分片
```

---

# 五、分库分表后的"新问题"

## 5.1 分布式主键

```sql
-- 分库后不能用 AUTO_INCREMENT → 两个分片可能生成相同的 id
-- 解法：Snowflake 算法（Twitter）
```

```
Snowflake ID 结构 (64-bit Long):
┌─┬──────────────────────┬─────────────┬────────────────┐
│0│ 41 位 时间戳(ms)      │ 10 位 机器ID │ 12 位 序列号    │
└─┴──────────────────────┴─────────────┴────────────────┘

生成规则：当前时间 - 自定义纪元 → 左移 22 位 → 加上机器 ID 左移 12 位 → 加上序列号
单机每毫秒 4096 个 ID → 10 台机器每秒 4000 万+ ID
```

```yaml
# ShardingSphere 内置 Snowflake
rules:
  sharding:
    keyGenerators:
      snowflake:
        type: SNOWFLAKE
        props:
          worker-id: 1  # 每台机器不同
```

```java
// Leaf（美团开源）—— 两种模式：
// ① Segment 模式：从 DB 预取号段（如 1001-2000），用完后取下一个
//    优点：高性能、避免 Snowflake 的时钟回拨问题
// ② Snowflake 模式：基于 ZooKeeper 的 worker-id 分配 + 时钟回退检测
```

## 5.2 跨分片分页

```sql
-- 问题：SELECT * FROM orders ORDER BY created_at LIMIT 100000, 20
-- 中间件需要从每个分片取 100020 条 → 聚合 → 全局排序 → 选 100000-100020
-- 10 个分片 = 10 × 100020 = 100 万行数据拉到中间件 → 排序 → 只返回 20 行！
```

```java
// 解法 1：禁用深度分页（产品层面）
// 解法 2：游标分页 + 二次查询
// 第一次：从各分片取 TOP N+offset，只取主键和排序列
// 第二次：全局归并排序后，拿到目标 20 条的主键，再回各分片查完整数据
// 解法 3：ES/HBase 等外部系统做排序和分页，MySQL 只存数据
```

## 5.3 分布式事务

```java
// Seata AT 模式：自动代理 SQL，生成 Undo SQL，两阶段提交
@GlobalTransactional  // 一个注解开启分布式事务
public void createOrder(Order order) {
    orderService.create(order);          // 操作订单库
    inventoryService.deduct(order);      // 操作库存库（另一个分片）
    accountService.debit(order);         // 操作账户库（再一个分片）
}
// ① 执行各分支事务 + 生成 Undo SQL
// ② 全部成功 → 提交（删 Undo SQL）
// ③ 任何失败 → 回滚（用 Undo SQL 反向操作）
```

```yaml
# Seata AT 模式配置
seata:
  tx-service-group: my_tx_group
  service:
    vgroup-mapping:
      my_tx_group: default
```

```java
// 大多数业务实际上不需要强分布式事务
// → 本地事务 + 最终一致性（MQ 消息 + 补偿调度）:
orderService.create(order);  // 本地事务
mq.send(new OrderCreatedEvent(order));  // 发消息
// 消费者收到消息 → 扣库存 → 扣款 → 失败则重试 + 补偿
```

---

# 六、总结

| 决策 | 关键问题 | 建议 |
|------|---------|------|
| **什么时候拆** | 单表 > 5000 万行 / > 500GB | 先考虑归档旧数据、只读副本等更简单的方案 |
| **怎么拆** | 查询模式决定分片键 | 99% 查询带的分片键 = 正确选择 |
| **分片算法** | 是否未来需要扩容 | 预期会扩容 → 一致性 Hash；预估够用 → Hash 取模 |
| **中间件** | Java + 不需多语言 | ShardingSphere-JDBC |
| **分布式主键** | 已有 ZK 或不需要时钟回拨保护 | ShardingSphere 内置 Snowflake；否则 Leaf |
| **分布式事务** | 真的必须强一致吗 | 90% 场景不需要 → 最终一致性够用 |

# 延伸阅读

**Do——动手搭建：**
- Spring Boot + ShardingSphere-JDBC 搭建 2 库 4 表的分库分表 Demo
- 插入 100 万条数据验证分片均匀性（`SELECT COUNT(*)` 到各分片）
- 用 Seata AT 模式实现跨分片的 `INSERT order + UPDATE inventory`

**Todo——深入方向：**
- ShardingSphere 的 SQL 改写细节（分页改写、聚合改写、Join 改写）
- 数据迁移方案：停机迁移 vs 双写 + 灰度切流 vs 增量同步
- Vitess（YouTube 开源）与 ShardingSphere 的架构对比

*本文参考资料：*
- ShardingSphere 官方文档: https://shardingsphere.apache.org/
- 美团技术团队，Leaf——美团分布式 ID 生成服务
- 阿里巴巴 Seata 官方文档: https://seata.io/
