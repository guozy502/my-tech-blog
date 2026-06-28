---
title: "ZooKeeper 核心机制——ZAB 协议与 Leader 选举全解"
date: 2026-06-28
description: 从 ZAB 协议的崩溃恢复、消息广播两个模式出发，拆解 ZooKeeper 如何用两阶段提交变体实现强一致性，以及 FLE 选举算法的 zxid 优先级设计。
tags: ["ZooKeeper","中间件","ZAB","共识算法","Leader选举"]
categories: ["中间件"]
---

```mermaid
flowchart TD
    ZAB["ZAB 协议\n(ZooKeeper Atomic Broadcast)"]
    ZAB --> RECOVERY["崩溃恢复模式\n选举 Leader + 数据同步"]
    ZAB --> BROADCAST["消息广播模式\n两阶段提交变体\nProposal → Ack → Commit"]
    
    RECOVERY --> FLE["FLE 选举\n优先级: zxid > myid"]
    BROADCAST --> PROP["Leader 接收写请求\n生成 Proposal"]
    PROP --> ACK["Follower 收到 Proposal\n写本地事务日志 → ACK"]
    ACK --> COMMIT["过半数 ACK\nLeader 发 Commit\n应用到内存"]
    
    style RECOVERY fill:#ffebee,stroke:#c62828
    style BROADCAST fill:#e8f5e9,stroke:#2e7d32
```

---

# 一、ZAB 协议的两个模式

## 1.1 崩溃恢复（Crash Recovery）

```
触发条件：
  - Leader 宕机 / 网络分区 / 集群刚启动
  
目标：
  1. 选举出新的 Leader
  2. 所有节点同步到一致状态（已提交的不丢、未提交的回滚）
```

## 1.2 消息广播（Message Broadcast）

```
正常运行时：
  Leader 接收客户端写请求
  → 生成 Proposal（带递增 zxid）
  → 广播给所有 Follower
  → Follower 写事务日志 + ACK
  → 过半 ACK → Leader 发 Commit
  → 所有节点应用到内存
```

---

# 二、FLE——ZooKeeper 的 Leader 选举

## 2.1 选举规则

```
每个节点投票：(myid, zxid)

比较规则：
  ① 先比 zxid：zxid 大的优先（数据最新）
  ② zxid 相同比 myid：myid 大的优先（确定性裁决）

结果：数据最新的节点成为 Leader
```

## 2.2 选举过程

```mermaid
sequenceDiagram
    participant S1 as Server 1 (myid=1, zxid=100)
    participant S2 as Server 2 (myid=2, zxid=100)
    participant S3 as Server 3 (myid=3, zxid=99)
    
    Note over S1,S3: Leader 宕机，触发选举
    
    S1->>S1: 第一轮投票：投自己 (1, 100)
    S2->>S2: 投自己 (2, 100)
    S3->>S3: 投自己 (3, 99)
    
    S1->>S2: 交换投票信息
    S2-->>S1: (2, 100) vs (1, 100) → zxid 平，myid=2 更大
    S1->>S1: 改投 S2
    
    Note over S1,S3: S2 收到 3 票（过半）→ 成为 Leader
```

## 2.3 FLE 选举轮次

每轮选举增加 `logicalclock`（选举纪元），确保不会混入旧轮投票。节点收到更高逻辑时钟的投票 → 更新自己的时钟 → 重新参与新一轮。

---

# 三、消息广播——两阶段提交变体

```mermaid
sequenceDiagram
    participant L as Leader
    participant F1 as Follower 1
    participant F2 as Follower 2
    
    L->>L: 接收写请求 → 生成 Proposal(zxid=101)
    L->>F1: Proposal(101)
    L->>F2: Proposal(101)
    
    F1->>F1: 写事务日志
    F1-->>L: ACK(101)
    F2->>F2: 写事务日志
    F2-->>L: ACK(101)
    
    Note over L: 2/3 >= 半数 → 提交
    L->>F1: Commit(101)
    L->>F2: Commit(101)
    F1->>F1: 应用到内存
    F2->>F2: 应用到内存
```

**关键设计**：
- Follower 的 ACK 顺序必须和 Proposal 顺序一致（FIFO 队列保证）
- Commit 可以乱序到达（异步应用）

---

# 四、数据同步——新 Leader 如何追齐？

新 Leader 当选后，需要确认所有节点有相同的已提交数据：

```
1. Leader 取最新 zxid（记为 newEpoch）
2. 向所有 Follower 发送 NEWLEADER 包（含 newEpoch）
3. Follower 对比自己的 zxid：
   - zxid <= Leader → 接受，等待同步
   - zxid > Leader → 拒绝（不应该发生，因为 FLE 保证了 zxid 最大者当选）
4. Leader 从自己的事务日志中找到 Follower 的 zxid 之后的所有操作
5. 通过 Proposal + Commit 方式同步给 Follower
6. 同步完成 → 服务可用
```

---

# 五、ZAB vs Raft

| | ZAB（ZK） | Raft（etcd） |
|------|-----------|------------|
| **选举优先级** | zxid > myid | Term + Log Index 更完整 |
| **日志修复** | Leader 从自己日志复制差异 | Leader 强制覆盖 Follower |
| **读一致性** | Follower 可读（sync 保证） | 默认 Leader 读（linearizable） |
| **协议复杂度** | 无正式论文 | 有完整论文和动画 |

*本文参考资料：*
- Neha Narkhede et al.《Kafka: The Definitive Guide》（第 2 版）——第 1-6 章（架构、生产者、消费者、内部机制）
- Apache Kafka 官方文档 - Design 章节: https://kafka.apache.org/documentation/#design
- Apache ZooKeeper 官方文档: https://zookeeper.apache.org/doc/current/
- Flavio P. Junqueira, Benjamin Reed《ZooKeeper: Distributed Process Coordination》
- Linux 内核文档 - sendfile(2) / mmap(2) / Page Cache
- KIP-500: Replace ZooKeeper with a Self-Managed Metadata Quorum (KRaft)
