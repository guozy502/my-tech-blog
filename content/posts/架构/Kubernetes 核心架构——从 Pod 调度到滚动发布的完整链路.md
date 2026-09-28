---
title: "Kubernetes 核心架构——从 Pod 调度到滚动发布的完整链路"
date: 2026-08-07
description: 从一个 Pod 从提交到运行的完整调度流程、Deployment/ReplicaSet/Pod 的层级控制关系、Service 的 ClusterIP（iptables/IPVS 路由）+ Ingress L7 路由、StatefulSet 的有状态标识管理、到滚动更新（RollingUpdate）与回滚的精确实时状态同步，拆解 Kubernetes 声明式 API 与控制器模式的核心设计。
tags: ["架构","Kubernetes","K8s","Pod","Service","调度"]
categories: ["架构"]
---

# 历史背景——Google Borg 的"平民化"版本

2003 到 2015 年，Google 内部的所有服务都跑在一个叫 Borg 的集群管理系统上。Borg 调度了 Google 搜索、Gmail、MapReduce 等几十万台机器的作业。2014 年，Google 基于 Borg 的经验发布了 Kubernetes（开源版本）——只保留了 Borg 中最通用、最成熟的特性，去掉了 Google 特有的部分。

理解 K8s 的关键在于理解它的核心哲学——**声明式 API + 控制器模式**。你在 YAML 中声明"我想要 3 个 Pod"（期望状态），一堆控制器在后台不断地比较"期望状态"和"实际状态"，发现差异就自动调整。你不是告诉 K8s"怎么创建 Pod"，而是告诉它"我想要什么状态"。

---

# 一、一个 Pod 从提交到运行——控制面 + 数据面

## 1.1 控制面的四个组件

```
kubectl apply -f pod.yaml
  ↓
┌──────────────────────────────────────────────┐
│  API Server（唯一的"门面"，所有操作通过它）     │
│  - 认证（你是谁）→ 鉴权（你能做什么）→ 准入控制器   │
│  - 把 Pod 的定义写入 etcd                     │
└──────────────────────────────────────────────┘
  ↓ watch（API Server 通知"Scheduler 有新 Pod 没分配节点"）
┌──────────────────────────────────────────────┐
│  Scheduler（调度器："这个 Pod 放到哪个 Node 上"）│
│  - 过滤（Filter）：哪些 Node 满足 Pod 的资源需求  │
│  - 打分（Score）：哪个 Node 最优               │
│  - 绑定：把 Pod 绑定到选中的 Node → 写入 etcd    │
└──────────────────────────────────────────────┘
  ↓ watch（kubelet 检测到"有个 Pod 被分配到我身上了"）
┌──────────────────────────────────────────────┐
│  kubelet（每台 Node 上的"节点代理"）            │
│  - 调 CRI → containerd → runc → 创建容器       │
│  - 调 CNI → 给 Pod 分配 IP，设置网络            │
│  - 调 CSI → 挂载存储卷                          │
└──────────────────────────────────────────────┘
  ↓ 汇报状态
┌──────────────────────────────────────────────┐
│  Controller Manager（一堆控制器在后台循环工作）    │
│  - ReplicaSet Controller："这个 Deployment 期   │
│     望 3 个 Pod，现在只有 2 个 → 创建一个"       │
└──────────────────────────────────────────────┘
```

**面试追问：API Server 为什么是"无状态的"？它把数据存哪了？** 数据存在 etcd 中（一个基于 Raft 的强一致 KV 存储）。API Server 本身无状态，可以部署多副本——多个 API Server 实例都是 etcd 的"看门人"，不存储数据。

## 1.2 Scheduler 的分配决策机制

```
过滤（Filter）——淘汰不合格的 Node：
  ✗ Node 的 CPU/内存剩余不足 → 淘汰
  ✗ Node 上有 Pod 指定的端口冲突 → 淘汰
  ✗ Node 的标签不匹配 nodeSelector/nodeAffinity → 淘汰
  ✗ Node 上的污点（Taint）无法被 Pod 的容忍（Toleration）匹配 → 淘汰

打分（Score）——给剩余 Node 排名：
  ✓ LeastRequestedPriority：选择"最空闲"的节点（剩余资源最多）
  ✓ BalancedResourceAllocation：CPU 和内存的使用比例均衡
  ✓ NodeAffinityPriority：优先选符合亲和性的节点
  ✓ ImageLocalityPriority：镜像已在节点上 → 省去拉镜像的时间
  ✓ InterPodAffinityPriority：选择"离它相关的 Pod 最近的节点"（如 cache Pod 和数据 Pod）
```

## 1.3 Pod 是什么——最小调度单元

```
Pod = 一组共享网络和存储的容器：

┌─────────── Pod ────────────┐
│                             │
│  ┌───────────┐ ┌─────────┐ │
│  │ Container A│ │Container B│  ← 共享同一个 NET namespace
│  │  (主应用)   │ │ (Sidecar)│     共享同一个 IPC namespace
│  │            │ │         │     共享同一个存储卷
│  └───────────┘ └─────────┘ │
│                             │
│  共享 IP: 10.244.1.5        │  ← 容器间通过 localhost 通信
│  共享 Volume: /var/log      │  ← 容器间共享文件
└─────────────────────────────┘

Pod 是"不可变的"——不会"修改 Pod 的镜像"，而是"创建新 Pod 替代旧 Pod"
```

---

# 二、Deployment ——声明式控制 Pod 的生命周期

## 2.1 三层控制关系

```
Deployment（"我想要 N 个这样的 Pod"）
    │
    ├── 管理 ReplicaSet（"Pod 的版本控制"——创建/回滚到哪个版本）
    │     │
    │     ├── ReplicaSet v1 (镜像: app:v1, 期望 3 个 Pod)
    │     │     ├── Pod (app-v1-abc)
    │     │     ├── Pod (app-v1-def)
    │     │     └── Pod (app-v1-ghi)
    │     │
    │     └── ReplicaSet v2 (镜像: app:v2, 期望 3 个 Pod) ← 滚动更新时创建
    │           ├── Pod (app-v2-xyz)
    │           └── Pod (app-v2-uvw)
    │
    └── 滚动更新时，v1 逐步缩容，v2 逐步扩容
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:               # ← Pod 模板
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: app
        image: my-app:v1
        resources:
          requests:
            cpu: "200m"
            memory: "256Mi"
          limits:
            cpu: "500m"
            memory: "512Mi"
```

## 2.2 滚动更新（RollingUpdate）的精确流程

```
Deployment 从 v1 → v2：

① kubectl set image deployment/my-app app=my-app:v2
② Deployment Controller 检测到 Pod 模板变了
③ 创建新的 ReplicaSet（v2），初始 replicas=0
④ 逐步操作：
     v2 扩容 +1 → v2 Pod 变成 Ready → v1 缩容 -1
     v2 扩容 +1 → v2 Pod 变成 Ready → v1 缩容 -1
     v2 扩容 +1 → v2 Pod 变成 Ready → v1 缩容 -1
⑤ v1 的 ReplicaSet 缩小到 0（但保留历史记录）
⑥ 滚动更新完成 → 所有流量切换到 v2

回滚 = kubectl rollout undo deployment/my-app
  → Deployment 找到上一个 ReplicaSet（v1）→ 反向滚动回去
  正是因为老的 ReplicaSet 被保留，才不需要"重新创建 v1"
```

**面试追问：滚动更新时怎么保证零宕机？** 需要两个条件：(1) `readinessProbe` 等待新 Pod 完全 Ready 后才标记为可用，Service 此时才把流量切过来；(2) `terminationGracePeriodSeconds` 给旧 Pod 足够的时间处理完现有请求后再被 SIGTERM。

---

# 三、Service——给短暂的 Pod 一个稳定的访问入口

## 3.1 为什么需要 Service？

```
Pod 是临时的——重启、升级、扩缩，IP 都会变
  Pod v1-abc → IP=10.244.1.5  → 被删除
  Pod v1-def → IP=10.244.2.3  → 新的 Pod，IP 完全不一样

Service 给这组 Pod 一个"不变的名字和虚拟 IP"
  my-app-service → ClusterIP=10.96.0.10 → 永远不变
  → kube-proxy 自动更新转发规则，保持 ClusterIP 指向当前存活的后端 Pod
```

## 3.2 ClusterIP 的工作原理——iptables/IPVS 规则

```bash
# ClusterIP 模式默认用 iptables 规则做负载均衡

# 1. Service 创建后，分配 ClusterIP=10.96.0.10
# 2. kube-proxy 在每台 Node 上写 iptables 规则
iptables -t nat -L KUBE-SERVICES -n
# 规则逻辑：
#   如果 目标IP=10.96.0.10 且 端口=80
#     → 随机选后端 Pod（10.244.1.5:8080、10.244.2.3:8080、10.244.3.1:8080）
#     → DNAT 到选中的 Pod IP:Port

# IPVS 模式（性能更好，K8s 1.11+ GA）：
#   在每台 Node 上创建 IPVS 虚拟服务器 → 直接在内核态做负载均衡
#   支持多种调度算法（rr/wrr/lc/wlc/sed/nq/dh/sh）
```

## 3.3 四种 Service 类型

| 类型 | 可访问范围 | 实现方式 |
|------|----------|---------|
| **ClusterIP**（默认） | 集群内部 | 虚拟 IP + iptables/IPVS 规则 |
| **NodePort** | 集群外部（通过 `<NodeIP>:<30000-32767>`） | 在每台 Node 上开通一个端口，转发到 ClusterIP |
| **LoadBalancer** | 外部负载均衡器（云厂商外部 LB） | 云 LB → NodePort → ClusterIP |
| **ExternalName** | 返回外部域名 | DNS CNAME 记录 |

## 3.4 Ingress——L7 HTTP 路由

```
为什么需要 Ingress？
  NodePort 只能用一个端口映射一个 Service

  Ingress = "HTTP 路由器"：
    同一个 IP/端口，根据 Host 和 URL Path 分发到不同 Service

  api.example.com/orders  → order-service (ClusterIP: 10.96.0.20)
  api.example.com/users   → user-service  (ClusterIP: 10.96.0.30)
  static.example.com/*    → static-service(ClusterIP: 10.96.0.40)

Ingress Controller（如 nginx-ingress/traefik）是实际处理 Ingress 规则的组件
→ 它自己就是一个 Pod，运行 nginx/traefik，监听 Host/Path 规则的变化
→ 动态生成 nginx.conf 并 reload（或通过 Lua 动态，无需 reload）
```

---

# 四、StatefulSet——有状态的 Pod

```
Deployment 和 StatefulSet 的区别：

Deployment：
  Pod 名 = my-app-<random-string>（如 my-app-7d5f8b9c6-x2k3j）
  → Pod 是可互换的，删掉一个，重建的新 Pod 名字是新的随机串
  → 存储：PVC 没有固定绑定关系（可以用共享存储或空目录）

StatefulSet：
  Pod 名 = my-app-0, my-app-1, my-app-2（有稳定编号）
  → 每个 Pod 有"身份"——my-app-0 一直是 my-app-0，即使被删掉重建
  → 每个 Pod 有独立的 PVC（my-app-0 的存储 ≠ my-app-1 的存储）
  → 启动顺序可控（0 启好后才能启动 1；删除时倒序删除 2→1→0）

适用：
  - 数据库集群（MySQL/Mongo/ES 的主从节点需要稳定网络标识）
  - 消息队列（Kafka Broker 需要稳定的 broker.id）
  - 任何需要"这个 Pod 的数据在它自己的磁盘上"的应用
```

---

# 五、总结

| 组件 | 解决的问题 | 一句话 |
|------|---------|--------|
| **Pod** | 最小调度单元 | 一组共享网络和存储的容器 |
| **Deployment** | 声明式管理无状态 Pod | "我想要 3 个 Pod，你帮我看着" |
| **ReplicaSet** | 维护 Pod 数量 | Deployment 背后实际管理 Pod 数量的控制器 |
| **Service** | 给 Pod 稳定的访问入口 | 虚拟 IP → iptables/IPVS → 后端 Pod |
| **Ingress** | L7 HTTP 路由 | 同一个 IP/端口，按域名+路径分发到不同 Service |
| **StatefulSet** | 有状态 Pod | 每个 Pod 有稳定编号、独立存储、有序启停 |

# 延伸阅读

**Do——动手验证：**
- `kubectl describe pod <pod-name>` 观察 Scheduler 选择的 Node 和分配的 IP
- `kubectl get endpoints <service-name>` 查看 Service 当前路由到的后端 Pod IP:Port
- 模拟滚动更新：`kubectl set image deployment/my-app app=my-app:v2 && kubectl rollout status deployment/my-app`

**Todo——深入方向：**
- Kubernetes 的 CNI（容器网络接口）——Calico 的 BGP 路由 vs Flannel 的 VXLAN 覆盖网络
- HPA（水平 Pod 自动缩放）——CPU/内存指标 → Metrics Server → HPA Controller → 改 Deployment replicas
- Operator 模式——把运维知识编码为自定义控制器（如 MySQL Operator 自动主从切换）

*本文参考资料：*
- Brendan Burns et al.《Kubernetes: Up and Running》
- Kubernetes 官方文档: Concepts (Architecture / Workloads / Services)
- Google Borg Paper (2015): "Large-scale cluster management at Google"
