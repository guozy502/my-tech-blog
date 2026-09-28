---
title: "CQRS、Event Sourcing 与六边形架构——三种架构模式的本质与适用边界"
date: 2026-08-07
description: 从 CQRS 的"为什么读写模型要分离"与四种实现层次、Event Sourcing 的"用不可变事件流替代当前状态"与快照+回溯机制、六边形架构的"依赖倒置在模块层面的应用——让领域层不依赖任何外部技术"，拆解三种架构模式各自解决什么问题以及什么场景下组合使用。
tags: ["架构","CQRS","Event Sourcing","六边形架构","DDD","端口适配器"]
categories: ["架构"]
---

# 历史背景——CQRS 是怎么从 DDD 中衍生出来的？

2004 年过后，DDD 社区发现了一个问题：**同一个领域模型既要做写操作（需要事务、业务规则校验），也要做读操作（需要复杂查询、JOIN、聚合）——两边都做不好。** 写模型关心"一个订单的金额不能大于用户余额"，读模型关心"展示最近 30 天按品类聚合的订单金额趋势"。这两个需求用的是同一套 Order 类，但写操作只需要几个字段，读操作却想 JOIN 一堆表。

Greg Young 和 Udi Dahan 在 2010 年左右开始推广 CQRS——**把领域模型的"命令"职责和"查询"职责拆成两个独立模型。** 写模型是纯粹的领域逻辑，读模型是为查询优化的数据结构。两者通过 Event 同步——写模型产生 Event，读模型消费并更新。

Event Sourcing 则更早——Martin Fowler 在 2005 年写了相关文章。它的核心是**不存"当前状态"，而存"发生了什么"——把每一次状态变更记录为不可变事件。** 这两种模式经常一起出现：CQRS 定义了"写和读分开"，Event Sourcing 定义了"写怎么记录"。六边形架构定义了"怎么写才能让领域不被技术污染"——三者讨论的是同一个问题的不同层次。

---

# 一、CQRS——读写分离在架构层面

## 1.1 传统 CRUD 的问题

```java
// 同一个 Order 类同时服务于"下单"和"查订单列表"
@Entity
public class Order {
    private Long id;
    private Long userId;
    private String status;
    private BigDecimal totalAmount;
    private LocalDateTime createdAt;
    
    // 写操作关心的字段：userId、status、totalAmount
    // 读操作额外需要的字段：
    private String userName;     // ← 从 user 表来
    private String productName;  // ← 从 product 表来
    private String productImg;   // ← 也是 product 表
    private Integer commentCount;// ← 从 comment 表来
    
    // 每次查询"订单列表"需要 JOIN 4 张表——这个模型对读太笨重
    // 每次下单——这个模型里 userName/productName 都是 null——无用字段
}
```

**根因**：单个模型既要做"写优化"（快速插入，事务保证），又要做"读优化"（多表 JOIN，聚合展示）——两边都不最优。

## 1.2 CQRS 的四种层次

```
层次 1——最简单的 CQRS：同一个数据库，不同的 DTO
  Command: INSERT INTO orders VALUES (...)
  Query: SELECT o.*, u.name, p.name FROM orders o 
         JOIN users u ON o.user_id = u.id 
         JOIN products p ON o.product_id = p.id
  → 只是代码层面分开 Service（OrderCommandService / OrderQueryService）
  → 存储还是同一张表，存储模型没有变化

层次 2——不同的存储，专为读写优化的表
  Command → orders 表（少量列，基于 PK 写优化）
  Query → order_detail_view 物化视图（预 JOIN 所有字段）
  → 写 SQL 简单快速，读 SQL 不用 JOIN（直接 SELECT * FROM order_detail_view）
  → 视图通过数据库触发器或代码同步更新

层次 3——不同的数据库，Event 驱动同步
  Command → MySQL (ACID 事务保证) → binlog → Kafka → ES (高性能搜索)
  写模型保证业务的正确性（库存不能为负、订单金额不能 > 余额）
  读模型保证查询的时效和灵活性（全文搜索、多维过滤、聚合统计）
  ES 中的文档结构和 MySQL 中的表结构完全不同——各自为各自的用途优化

层次 4——完整的 CQRS + Event Sourcing
  Command → Event Store（不可变事件流，追加写）
  Event → 投影到多个 Query DB（当前状态，物化视图）
  每个 Query DB 可以根据自己的需求从事件流中构建不同的物化视图
  → 订单列表需要 userName → 投影到 MySQL 的 order_view 表，包含 userName
  → 订单趋势图需要按天聚合 → 投影到 ClickHouse 中，按天聚合存储
```

**选择建议**：90% 的系统只需要层次 1（代码层面分开），层次 2 适合读压力已经大于写压力的场景，层次 3 适合需要全文搜索的场景，层次 4 适合审计/追溯/合规需求极强的场景（如银行交易、供应链追溯）。

## 1.3 CQRS 的一致性——读写之间有多长的延迟？

```
写操作完成 → 异步通知 → 读模型更新 → 用户马上刷新页面 → 可能看到旧数据

这不是 Bug，是 CQRS 的设计权衡：
  - 传统 CRUD：写后立刻读是实时的，但写性能受读查询拖累
  - CQRS：写是独立的，读模型异步更新，一致性窗口受 Event 传输延迟影响（通常毫秒到秒）

如何让用户不感到"数据延迟"？
  ① 写操作后立即返回成功页——用户看到"订单提交成功"即可，不需要马上看到订单列表中的新数据
  ② 乐观 UI 更新——前端先展示假数据，几毫秒后读模型更新时自动修正
  ③ 关键场景走强一致——如支付后必须"立刻"显示支付成功（这条 Read 走主库，不走读模型）
```

---

# 二、Event Sourcing——不存"当前"，存"发生了什么"

## 2.1 核心思想

```
传统数据库：存"当前状态"
  orders 表：id=123, status="paid", amount=100, paid_at="2026-08-07 10:00:00"
  → 你只知道订单"现在是已支付"，不知道"之前是什么状态、什么时候变的"

Event Sourcing：存"发生了什么"
  事件流：
    OrderCreated {orderId:123, amount:100, userId:456}
    OrderPaid {orderId:123, amount:100, paidAt:"2026-08-07"}
    OrderShipped {orderId:123, carrier:"SF", trackingNo:"SF123456"}
  
  → 当前状态 = 从事件流"投影"出来的
  → order 的状态 = OrderCreated + OrderPaid + OrderShipped → status="shipped"
  → 你可以知道订单的完整生命周期
```

## 2.2 Event Sourcing 的三个核心操作

```
① Append（追加）—— 产生事件
   新事件追加到 Event Store 尾部，不可修改，不可删除
   Event Store 可以是 PostgreSQL 的 append-only 表、Kafka Topic、或专门的 EventStoreDB

② Project（投影）—— 重建当前状态
   从 Event Store 读取该 Order 的所有事件 → 按顺序应用 → 得到当前状态
   Order currentState = events.reduce(initialState, (state, event) -> state.apply(event))
   
   为了性能，可以定期创建"快照"：
   快照 = 第 1000 个事件时的完整状态
   恢复时 → 加载快照 + 应用 1001~current 的事件 → 得到当前状态（不用从头回放全部事件）

③ Replay（回放）—— 重建历史状态
   从 Event Store 中读取"某个时间点之前的所有事件"→ 重建那个时间点的状态
   用于：审计、Bug 排查（"这个订单在下午 2:30 时的状态是什么？"）
```

## 2.3 Event Sourcing 的代价

```
代价 1：查询当前状态需要"回放"——慢
  解法 → CQRS + Event Sourcing：写走 Event Store，读走物化视图（从事件投影而来）
  → 这就是为什么这两个模式经常一起出现——Event Sourcing 擅长写，CQRS 擅长读

代价 2：事件 Schema 会演化——旧事件格式和新代码不兼容
  解法 → 事件的多版本处理：
    OrderCreated v1: {orderId, amount}
    OrderCreated v2: {orderId, amount, currency}  ← 加了币种字段
    → 代码中同时保留 v1 和 v2 的处理器（Upcaster 模式）

代价 3：一个事务只能写一个聚合
  经典的 ACID 事务只能应用在一个 Event Store 的范围内
  跨聚合的操作（订单完成→积分系统加积分）→ 通过 Saga 异步编排

代价 4：删除数据的矛盾
  GDPR 要求"用户可以要求删除个人数据"——但 Event Sourcing 的设计是不删除
  → 解法：加密个人数据 + 删除加密密钥 → 数据还在但不可恢复
```

## 2.4 Event Sourcing 的适用边界

| 适用 | 不适用 |
|------|--------|
| 需要完整审计追溯（金融交易、供应链） | 简单的 CRUD 系统 |
| 业务状态变化需要回溯和调试 | 状态本身就很简单（一个 flag 字段） |
| 多团队需要从同一事件流中构建各自的视图 | 数据量极大但不需要历史回溯（日志采集用 Kafka 就够了） |

---

# 三、六边形架构——让领域逻辑不被技术"污染"

## 3.1 传统三层架构的依赖问题

```java
// 传统分层：Controller → Service → Repository → DB
// 依赖方向：Controller → Service → Repository → MySQL Driver

@Service
public class OrderService {
    @Autowired
    private OrderRepository orderRepository;  // ← Service 知道 Repository
    
    public void cancel(Long orderId) {
        Order order = orderRepository.findById(orderId);  // ← 和数据库耦合
        order.cancel();
        orderRepository.save(order);  // ← 又是数据库操作
    }
}

// 问题：换存储（MySQL → Mongo）→ 不仅改 Repository 实现，还得改 Service 代码
//      → 因为 Service 中固化了"findById + save"这种数据库思维
```

**六边形架构的核心改变**：**让领域逻辑（Service）定义"我需要什么"，基础设施（Repository）去实现——依赖方向从"Service → Repository"变成"Service → 接口 ← Repository 实现"。**

## 3.2 端口-适配器模式

```
六边形内部（领域核心）：
  - 实体、值对象、聚合根
  - 领域服务（纯业务逻辑）
  - 端口（Port）= 接口，定义"领域需要什么"：
    OrderRepository（接口）：定义 save、findById
    PaymentGateway（接口）：定义 charge
    NotificationService（接口）：定义 notify

六边形外部（基础设施）：
  - 数据库适配器：MySQLOrderRepository implements OrderRepository
  - 支付适配器：StripePaymentGateway implements PaymentGateway
  - 消息适配器：KafkaNotificationService implements NotificationService

关键原则：领域代码不 import 任何基础设施的类
  → 领域层没有 @Autowired、没有 @Transactional、没有 JDBC Driver
  → 框架（Spring）在启动时负责"把适配器注入到端口"
```

```java
// 领域层——纯业务逻辑，不依赖框架
public class Order {
    private OrderId id;
    private Money totalAmount;
    private OrderStatus status;
    
    // 领域逻辑：取消订单
    public void cancel(LocalDateTime now) {
        if (status != OrderStatus.PAID) {
            throw new OrderException("只有已支付的订单可以取消");
        }
        if (now.isAfter(createdAt.plusHours(2))) {
            throw new OrderException("超过 2 小时的订单无法取消");
        }
        this.status = OrderStatus.CANCELLED;
    }
}

// 领域层——端口（接口，定义"我要什么"）
public interface OrderRepository {
    Order findById(OrderId id);
    void save(Order order);
}

// 基础设施层——适配器（实现"我怎么做到"）
@Repository
public class MySQLOrderRepository implements OrderRepository {
    private final JdbcTemplate jdbc;
    // ... 实现 MySQL 的查询和保存
}
```

## 3.3 六边形架构 vs 传统三层——本质差异

| | 传统三层 | 六边形架构 |
|------|---------|----------|
| **依赖方向** | Service → Repository → DB | Service → 端口(接口) ← 适配器(实现) |
| **谁定义接口** | Service 层没有接口，直接依赖具体实现 | **领域层**定义接口（端口），基础设施层实现 |
| **换存储的代价** | 改 Repository 实现 + 可能需要改 Service | 只改适配器，领域代码零修改 |
| **测试** | Service 测试需要 Mock Repository | Service 测试可以直接 new 实体类，不需要 Mock（领域逻辑纯 POJO） |
| **认知负担** | 低（三层，约定俗成） | 中（额外的接口抽象层） |

---

# 四、三者组合——什么场景下一起使用

```
Event Sourcing + CQRS 的组合最常见：
  写模型 = Event Store（追加不可变事件）
  读模型 = 物化视图（从事件流投影而来）
  MQ 同步两者

三者全组合的"完整架构"：
  六边形架构（组织代码，端口-适配器依赖倒置）
  Event Sourcing（存储方式，记录事件而非状态）
  CQRS（读写分离，各自独立的存储模型）

三者全组合的代价很高——只在"需要完整审计追溯 + 读写负载极不平衡 + 多团队多视图"
的场景下才值得。例：银行核心交易系统、交易所风控系统、电商供应链追溯。
```

---

# 五、总结

| 模式 | 解决什么问题 | 代价 | 常见搭配 |
|------|----------|------|---------|
| **CQRS** | 读写模型冲突 | 数据一致性窗口延迟 | 常与 Event Sourcing 一起用 |
| **Event Sourcing** | 需要完整状态追溯 | 查询当前状态需回放 | 常与 CQRS 一起用 |
| **六边形架构** | 领域逻辑被技术污染 | 额外的抽象层 | 独立使用，适合任何复杂业务 |

# 延伸阅读

**Do——动手实践：**
- 从你的项目中找一个 Service 类，把"纯业务判断"（如 if/else 的状态校验）从"技术操作"（如 repository.save）中抽出来，放到实体方法中
- 用 Event Sourcing 的思想记录一个"订单"的完整生命周期事件，然后写一个投影函数把事件流转成当前状态

**Todo——深入方向：**
- Saga 编排与 CQRS——多个聚合之间的事务怎么编排
- 事件存储（EventStoreDB / Kafka / PostgreSQL append-only）的选型对比
- 事件 Schema 演化——Upcaster、版本化事件的工程实践

*本文参考资料：*
- Greg Young, "CQRS and Event Sourcing" (2010)
- Martin Fowler, "Event Sourcing" (2005) / "CQRS" (2011)
- Alistair Cockburn, "Hexagonal Architecture" (2005)
- Vaughn Vernon《Implementing Domain-Driven Design》(2013)
