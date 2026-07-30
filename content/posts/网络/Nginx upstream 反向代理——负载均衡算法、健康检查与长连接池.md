---
title: "Nginx upstream 反向代理——负载均衡算法、健康检查与长连接池"
date: 2026-07-29
description: 从 upstream 的六种负载均衡算法（round-robin/least_conn/ip_hash/hash/random/least_time）的源码级实现原理、被动健康检查（max_fails + fail_timeout）与主动健康检查（health_check）的差异、到反向代理连接池的 keepalive 默认短连接陷阱，拆解 Nginx 作为反向代理的完整负载均衡机制。
tags: ["Nginx","反向代理","upstream","负载均衡","keepalive","健康检查"]
categories: ["网络"]
---

# 历史背景——为什么 Nginx 的 upstream 是"隐形的负载均衡器"？

在 Nginx 出现之前，负载均衡是独立部署的硬件或软件：F5 BIG-IP、HAProxy、LVS。部署架构通常是 `Web Server（Apache）→ LB（HAProxy）→ 后端应用服务器`——三层结构，LB 自己就有单点故障风险。

Nginx 把"Web Server"和"Load Balancer"两个角色合并了。`upstream` 模块让 Nginx 处理静态资源的同时，把动态请求按规则转发到后端应用服务器，省掉了一层独立的 LB。对于绝大多数中小型场景，Nginx 的 upstream 负载均衡能力已经足够——这就是为什么"Nginx 反向代理"几乎是每个后端项目的标配。

但 `upstream` 不是"随便写几个 server 就完了"。它的平滑加权轮询算法有精妙的数学设计，默认的"短连接"行为在高并发场景背后是 TIME_WAIT 爆炸，健康检查默认是被动的（等请求失败了才发现），很多配置细节直接影响生产可靠性。

---

# 一、upstream 的六种负载均衡算法

## 1.1 round-robin——平滑加权轮询（默认）

```
upstream backend {
    server 10.0.1.1:8080 weight=3;
    server 10.0.1.2:8080 weight=1;
    server 10.0.1.3:8080 weight=2;
}
```

这个算法的精妙之处在于**平滑**——不是简单的"按权重分配请求比例"，而是均匀分布每一次请求：

```
传统加权（不均匀）：
  高权重服务器连续收到 N 个请求 → 低权重服务器饿死一段时间 → 不均匀

Nginx 平滑加权（均匀）：
  weight=3, weight=2, weight=1
  分配序列：A B C A B A（3:2:1 比例，但分布在大时间范围内是均匀的）
```

**核心实现**：Nginx 为每个 upstream 中的 server 维护一个 `current_weight`。每次选取时，所有 `current_weight` 都增加各自的有效权重，然后选择 `current_weight` 最大的 server；被选中的 server 的 `current_weight` 减去总权重。

```c
// Nginx round-robin 选择算法（简化伪代码）
for (i = 0; i < peers->number; i++) {
    peers[i].current_weight += peers[i].effective_weight;  // 加各自的权重
    total += peers[i].effective_weight;
    
    if (best == NULL || peers[i].current_weight > best->current_weight)
        best = &peers[i];  // 选最大的
}

best->current_weight -= total;  // 被选中的减去总权重
return best;
```

**三轮示例**（A:weight=3, B:weight=2, C:weight=1）：

```
初始: A=0, B=0, C=0

第 1 次: A=3, B=2, C=1 → 选 A → A=3-6=-3
第 2 次: A=0, B=4, C=2 → 选 B → B=4-6=-2
第 3 次: A=3, B=0, C=3 → 选 A → A=3-6=-3 (C 也是 3，A 在前所以选 A)
第 4 次: A=0, B=2, C=4 → 选 C → C=4-6=-2
第 5 次: A=3, B=4, C=-1→ 选 B → B=4-6=-2
第 6 次: A=6, B=0, C=1 → 选 A → A=6-6=0

结果: A→B→A→C→B→A (3A:2B:1C，完全均匀)
```

## 1.2 least_conn——最小连接数

```nginx
upstream backend {
    least_conn;
    server 10.0.1.1:8080;
    server 10.0.1.2:8080;
}
```

每一次新请求选择**当前活跃连接数最少**的 server。适用场景：后端服务器处理能力相当，但请求处理时间差异很大（有些请求 10ms 完成，有些 2s）。round-robin 可能会把耗时长的请求集中分发到某个服务器，least_conn 通过"看谁空闲"来动态负载。

**权重与 least_conn 可以配合**：

```nginx
upstream backend {
    least_conn;
    server 10.0.1.1:8080 weight=3;  # 权重高的可以承担更多连接
    server 10.0.1.2:8080 weight=1;
}
```

## 1.3 ip_hash——基于客户端 IP 的会话保持

```nginx
upstream backend {
    ip_hash;
    server 10.0.1.1:8080;
    server 10.0.1.2:8080;
}
```

hash 客户端 IP 的前三个八位组（`$remote_addr` 的前三段），同一 IP 的请求总是落到同一个后端。适用场景：后端服务有本地缓存（session 数据），不希望请求分散到不同实例导致缓存穿透。

**局限性**：
- 某个 server 下线 → 原来 hash 到它的用户被重分配到剩余 server → 缓存丢失
- 用 IPv4 前三段意味着同一网段的用户大概率落到同一 server → 可能不均衡

## 1.4 hash——自定义键值的一致性哈希

```nginx
upstream backend {
    hash $request_uri consistent;  # 按请求 URI 哈希 + 一致性哈希
    server 10.0.1.1:8080;
    server 10.0.1.2:8080;
}
```

`consistent` 参数启用**一致性哈希**——增删 server 时只影响相邻哈希节点的数据，不导致全面重分配。适用于：后端 server 是缓存节点，希望同一 URI 始终被同一 server 处理。

## 1.5 random——随机选择（Nginx 1.15.1+）

```nginx
upstream backend {
    random two;          # 随机选 2 个，从中选最少连接的
    server 10.0.1.1:8080;
    server 10.0.1.2:8080;
}
```

`random two`：从所有 server 中随机选 2 个，再从这 2 个中按 least_conn 选一个。这个设计融合了随机（减少选择开销）和最少连接（保证均匀性）。

## 1.6 least_time——最小平均响应时间（Nginx Plus 商业版）

```nginx
upstream backend {
    least_time header;   # 按响应头返回时间
    least_time last_byte; # 按完整响应返回时间
    server 10.0.1.1:8080;
    server 10.0.1.2:8080;
}
```

Nginx Plus 专有，跟踪每个 server 的最近平均响应时间，选最快的。这是最符合"后端负载"语义的算法，但只有商业版可用。

## 1.7 六种算法速查

| 算法 | 原理 | 适用 |
|------|------|------|
| **round-robin** | 平滑加权轮询（默认） | 通用，后端同类 |
| **least_conn** | 选当前连接数最少的 | 请求耗时差异大 |
| **ip_hash** | 按客户端 IP 前三段哈希 | 需要会话保持 |
| **hash** | 自定义 key 哈希 + 可配一致性哈希 | 缓存节点路由 |
| **random** | 随机选 N 个再挑最少连接 | 减少大集群选 server 的开销 |
| **least_time** | 选平均响应最快的（Plus） | 后端异构性能 |

---

# 二、健康检查——Nginx 怎么知道后端死了？

## 2.1 被动健康检查（默认）

```nginx
upstream backend {
    server 10.0.1.1:8080 max_fails=3 fail_timeout=30s;
    server 10.0.1.2:8080 max_fails=3 fail_timeout=30s;
}

# 规则：
#  - 在 fail_timeout 内，失败次数达到 max_fails → 标记为 DOWN（不再转发）
#  - fail_timeout 之后 → 尝试一次请求（如果成功 → 重新标记为 UP）
```

这叫做**被动**健康检查，因为 Nginx 只在"正常请求过程中发现失败"时才标记后端为 down，**不会主动探测**。后端还没恢复这段时间内，没有请求发给它，Nginx 永远不知道它已经恢复了。

```nginx
# 生产建议配置
upstream backend {
    server 10.0.1.1:8080 max_fails=2 fail_timeout=10s;
    server 10.0.1.2:8080 max_fails=2 fail_timeout=10s;
    server 10.0.1.1:8080 backup;  # 备份 server（主 server 全挂时启用）
}
```

## 2.2 主动健康检查（Nginx Plus）

Nginx Plus 支持主动向每个后端 server 发送健康检查请求（类似 HAProxy 的 health check）：

```nginx
upstream backend {
    zone backend 64k;  # 共享内存，存储健康状态
    server 10.0.1.1:8080;
    server 10.0.1.2:8080;
}

server {
    location / {
        proxy_pass http://backend;
        health_check interval=5s fails=3 passes=2 uri=/health;
        # 每 5 秒 GET /health，连续 3 次失败 → unhealthy
        # 连续 2 次成功 → healthy
    }
}
```

**被动 vs 主动的区别**：被动是"下游正常流量中失败才发现"，主动是"周期性主动探测"。前者零额外开销但发现慢，后者发现快但有健康检查流量开销。

---

# 三、反向代理的连接池——keepalive 的隐蔽陷阱

## 3.1 默认：每次代理请求新建一个 TCP 连接

```
Nginx 默认代理行为：
  客户端请求 1 到来 → Nginx 新建一条 TCP 连接到 backend:8080
  → 后端处理后返回 → Nginx 马上关闭这条 TCP 连接（TIME_WAIT）
  
  高并发下：
  1000 QPS = 1000 次 TCP 建连/秒 + 1000 次 TIME_WAIT/秒
  → 端口耗尽 + 大量 TIME_WAIT socket 占用内存
```

## 3.2 keepalive 开启长连接池

```nginx
upstream backend {
    server 10.0.1.1:8080;
    server 10.0.1.2:8080;
    keepalive 32;  # ← 每个 Worker 进程最多保持 32 个空闲长连接
}

server {
    location /api/ {
        proxy_pass http://backend;
        
        # 必须同时设置这些 keepalive 相关指令（否则 keepalive 不起作用！）
        proxy_http_version 1.1;               # HTTP/1.1 才支持 keepalive
        proxy_set_header Connection "";        # 清空原始 Connection 头
        proxy_set_header Host $host;
    }
}
```

**关键点**：
- `keepalive 32` 是每个 Worker 进程对每个 upstream 的最大空闲连接数（不是总数）
- 如果 Worker 都是 4 核 → 4 × 32 = 128 个空闲连接（同一时刻最多 128 个请求复用连接）
- 超出 128 的并发请求仍然新建 TCP 连接 → 请求完成后如果池有空位，加入连接池；没有 → 直接关闭

**生产建议值**：`keepalive` = 后端 server 数 × 每个 server 期望的最大并发数 × 1.5

```nginx
# 后端 4 台 server，每台并发 100
upstream backend {
    server 10.0.1.1:8080;
    server 10.0.1.2:8080;
    server 10.0.1.3:8080;
    server 10.0.1.4:8080;
    keepalive 64;  # 4 × 16 ≈ 64
}
```

## 3.3 代理连接的超时控制

```nginx
server {
    location /api/ {
        proxy_pass http://backend;
        
        # 连接池相关
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        
        # 超时控制
        proxy_connect_timeout 3s;     # 与后端建立连接的超时
        proxy_read_timeout 30s;       # 等后端响应的超时（重要！）
        proxy_send_timeout 30s;       # 发请求体给后端的超时
        
        # 长连接在池中的空闲时间（后端断开前 Nginx 先主动关）
        proxy_socket_keepalive on;     # TCP keepalive 探测（防防火墙踢连接）
    }
}
```

`proxy_read_timeout` 是最关键的参数——过短导致正常的长响应被截断（后端还在处理，Nginx 等不及了）；过长导致异常后端长时间占用连接。根据后端 99 分位耗时 × 1.5 设置是一个经验起点。

---

# 四、完整的生产级 reverse proxy 配置

```nginx
upstream app_backend {
    # 负载均衡算法
    least_conn;
    
    # 后端列表
    server 10.0.1.1:8080 weight=3 max_fails=2 fail_timeout=10s;
    server 10.0.1.2:8080 weight=3 max_fails=2 fail_timeout=10s;
    server 10.0.1.3:8080 weight=2 max_fails=2 fail_timeout=10s;
    server 10.0.1.1:8080 backup;  # 备份
    
    # 连接池
    keepalive 64;
    keepalive_timeout 60s;
    keepalive_requests 1000;  # 每条连接最多复用 1000 次
}

server {
    listen 80;
    server_name api.example.com;
    
    location / {
        # 代理目标
        proxy_pass http://app_backend;
        
        # HTTP 协议 + keepalive
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        
        # 转发客户端真实信息
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        # 超时
        proxy_connect_timeout 3s;
        proxy_read_timeout 30s;
        proxy_send_timeout 30s;
        
        # 缓冲
        proxy_buffering on;
        proxy_buffer_size 4k;
        proxy_buffers 8 16k;
        proxy_busy_buffers_size 32k;
        
        # 错误降级（后端不可用时返回缓存内容）
        proxy_cache_use_stale error timeout updating http_500 http_502 http_503 http_504;
    }
}
```

---

# 五、总结

| 配置维度 | 关键指令 | 建议 |
|---------|---------|------|
| **算法** | `least_conn` | 后端处理时长不一致时用 |
| **权重** | `weight` | 后端异构时按 CPU/内存比例分配 |
| **健康检查** | `max_fails` + `fail_timeout` | 生产设 2/10s，快速摘除故障节点 |
| **后备 server** | `backup` | 至少留一台备份，主 server 全挂时自动启用 |
| **连接池** | `keepalive` | 必须配合 `proxy_http_version 1.1` + `Connection ""`！ |
| **超时** | `proxy_read_timeout` | 按 P99×1.5 设，过短截断长响应，过长拖累故障恢复 |

# 延伸阅读

**Do——动手验证：**
- 用 Docker Compose 启动 3 个不同响应耗时的后端 + Nginx upstream，配置 `least_conn`，观察连接分布
- 对比 `keepalive 0` vs `keepalive 32` 的压测吞吐（`wrk -c100 -t4`），观察 TIME_WAIT 数量（`ss -s`）
- 故意停掉一个后端，观察 `max_fails=2 fail_timeout=10s` 的摘除速度和恢复检测

**Todo——深入方向：**
- Nginx 反向代理的 `proxy_cache` 缓存命中逻辑——如何精确控制 `proxy_cache_key` 包含/排除哪些头
- `ngx_http_upstream_check_module`（第三方模块）——开放源码版的主动健康检查方案
- Nginx 的 DNS 动态解析——`resolver` + `set $backend "..."` + `proxy_pass $backend` 的变量驱动方案

*本文参考资料：*
- Nginx 官方文档: ngx_http_upstream_module
- Nginx 源码 `src/http/ngx_http_upstream_round_robin.c`（平滑加权轮询）
- Nginx 源码 `src/http/ngx_http_upstream_least_conn_module.c`
- Igor Sysoev, "Upstream Keepalive Connections in NGINX" (2015)
