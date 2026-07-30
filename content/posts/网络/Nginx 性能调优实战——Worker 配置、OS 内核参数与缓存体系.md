---
title: "Nginx 性能调优实战——Worker 配置、OS 内核参数与缓存体系"
date: 2026-07-29
description: 从 Worker 进程数与 CPU 亲和性绑定、sendfile/tcp_nopush/tcp_nodelay 的 OS 级零拷贝优化、gzip 压缩的 CPU/带宽 tradeoff、open_file_cache 文件描述符缓存到 proxy_buffer 的代理缓冲区调优，覆盖 Nginx 从应用层到 OS 层的全链路性能调优和实践配置。
tags: ["Nginx","性能优化","sendfile","gzip","Worker","TCP"]
categories: ["网络"]
---

# 历史背景——Nginx 配置的"默认值陷阱"

Nginx 的配置以简单著称——十几行就能跑一个反向代理。但这份简单是双刃剑：**很多默认值在 2004 年的硬件上是合理的，在今天的 SSD、万兆网卡、64 核 CPU 上却成了性能瓶颈。**

几个例子：
- `sendfile off`（默认 on，但很多老配置模板里没开）
- `gzip off`（默认关闭压缩——2004 年 CPU 比带宽贵）
- `worker_connections 512`（默认 512，但今天一个 Worker 能处理 65535+ 并发）
- `proxy_buffer_size 4k|8k`（默认小 buffer 适合小响应，大 JSON 响应会导致频繁磁盘 IO）

Nginx 的性能调优不是"改几个参数 CPU 就降一半"的魔法，而是**让 OS 内核替你干活**。sendfile 让内核从磁盘直接拷贝到网卡，TCP_NODELAY 让内核立刻发送小包——你的 Nginx 进程只需要专注"决定转发给谁"这一件事。

---

# 一、Worker 进程调优——多核利用的第一步

## 1.1 Worker 数量

```nginx
# ✅ 正确：Worker 数 = CPU 核数
worker_processes auto;
# Nginx 自动检测 CPU 核数（getconf _NPROCESSORS_ONLN）
# 设为 auto 是配合容器化部署的最优解

# ❌ 常见错误
worker_processes 1;    # 16 核服务器只用一个核——其余 15 个核闲置
worker_processes 16;   # 8 核服务器设 16——多余的 Worker 在争抢 CPU 时间片
```

**为什么 Worker = CPU 核数？** 每个 Worker 是单线程的。N 核的 CPU → N 个 Worker 各绑定到一个核 → 没有上下文切换→ 没有缓存失效→ 每个 Worker 跑满它的核。

## 1.2 CPU 亲和性绑定

```nginx
worker_processes auto;
worker_cpu_affinity auto;  # 1.9.10+ 支持 auto
# 等价于把每个 Worker 固定到不同 CPU 核心

# 老版本的显式配置（了解即可）
# worker_cpu_affinity 0001 0010 0100 1000;  # 4 核
```

**为什么要绑定 CPU？** 如果不绑定，Linux 调度器可能把 Worker 1 从 Core 0 调到 Core 3——Core 0 的 L1/L2 缓存里 Worker 1 的数据全废了，到了 Core 3 要重建。频繁的 CPU 迁移导致缓存命中率暴跌。

## 1.3 连接数上限

```nginx
# 总并发连接数 = worker_processes × worker_connections
# 例：4 Worker × 10240 = 40960 并发连接

events {
    worker_connections 10240;  # 每个 Worker 最大并发连接数
    multi_accept on;           # 一次 accept 尽可能多取连接
    accept_mutex off;          # 1.11.3+ 配合 EPOLLEXCLUSIVE
    use epoll;                 # Linux 显式用 epoll
}

# 同时修改 OS 文件描述符限制
worker_rlimit_nofile 65535;  # 设置每个 Worker 可打开的文件描述符数
```

**worker_connections 和 worker_rlimit_nofile 的关系**：每个客户端连接至少用 1 个 fd。如果你还代理了后端，每个客户端连接对应 1 个上游连接 = 再用 1 个 fd。所以实际需要的 fd = worker_connections × 2（加上日志、DNS 查询等额外开销）。`worker_rlimit_nofile` 必须 **≥ worker_connections × 2**。

---

# 二、TCP/IO 调优——让内核替你拷贝

## 2.1 sendfile——从"用户态拷两次"到"内核直接拷"

```nginx
http {
    sendfile on;  # 启用 sendfile 零拷贝
    
    # sendfile 开启后，静态文件的发送路径：
    # 磁盘 DMA → Page Cache → 网卡 DMA（不经过用户态！）
    # 
    # sendfile 关闭时（传统方式）：
    # 磁盘 DMA → Page Cache → 内核→用户态拷贝 → 用户态→Socket Buffer 拷贝 → 网卡 DMA
    # 多了 2 次 CPU 拷贝 + 2 次上下文切换
}
```

**什么时候不能开 sendfile？** 如果请求需要修改响应内容（如 `add_before_body` / `add_after_body` / `sub_filter`），Nginx 必须把数据读到用户态做处理 → sendfile 自动被绕过。

## 2.2 tcp_nopush 与 tcp_nodelay——两个互补的优化

```nginx
http {
    sendfile on;
    tcp_nopush on;   # ① 攒够 MSS 或等 sendfile 完成再发 → 减少 TCP 包数
    
    # 只有在 sendfile on 时 tcp_nopush 才有效
    # 原理：设置 TCP_CORK，让 TCP 尽量攒到 MSS (1460B) 再发
}

server {
    location / {
        tcp_nodelay on;  # ② 对于 keepalive 连接，立刻发送小包（不攒）
    }
}
```

```
tcp_nopush vs tcp_nodelay：
  tcp_nopush = "攒够了再发" → 适合发静态文件（大块数据）
  tcp_nodelay = "有就立刻发" → 适合发 keepalive 上的小响应（降低延迟）
  
两者可以同时开启，作用时机不同：
  - tcp_nopush：在 HTTP 响应头发送前设置 TCP_CORK
  - tcp_nodelay：在 HTTP 响应头发送后取消 TCP_NODELAY
```

## 2.3 OS 内核参数

```bash
# /etc/sysctl.conf —— 高并发 Nginx 的 OS 层配置

# TCP 连接的快速回收（减少 TIME_WAIT）
net.ipv4.tcp_tw_reuse = 1        # 允许 TIME_WAIT socket 被复用
net.ipv4.tcp_fin_timeout = 30    # FIN_WAIT 超时（默认 60s）
net.ipv4.tcp_max_tw_buckets = 5000  # TIME_WAIT 最大数量

# TCP 缓冲区
net.core.rmem_max = 16777216     # 最大接收缓冲 16MB
net.core.wmem_max = 16777216     # 最大发送缓冲 16MB
net.ipv4.tcp_rmem = 4096 87380 16777216  # 接收缓冲（最小/默认/最大）
net.ipv4.tcp_wmem = 4096 65536 16777216  # 发送缓冲

# 连接队列（应对突发流量）
net.core.somaxconn = 65535       # 监听队列最大长度
net.ipv4.tcp_max_syn_backlog = 8192  # SYN 队列最大长度

# 本地端口范围（多 upstream 时避免端口不够用）
net.ipv4.ip_local_port_range = 1024 65000

# 文件描述符
fs.file-max = 655350             # 系统级文件描述符上限

# 应用配置
sysctl -p  # 生效
```

---

# 三、压缩与缓冲——CPU 和带宽的 tradeoff

## 3.1 gzip 压缩

```nginx
http {
    gzip on;
    gzip_min_length 1000;         # 小于 1000B 不压缩（压了反而变大）
    gzip_comp_level 5;            # 1-9，5 是 CPU/压缩率的平衡点
    gzip_types text/plain text/css application/json 
               application/javascript text/xml application/xml
               application/xml+rss text/javascript;
    gzip_vary on;                 # 添加 Vary: Accept-Encoding 头
    gzip_disable "msie6";         # IE6 不压缩（兼容性问题）
    gzip_proxied any;             # 对代理请求也压缩
    gzip_buffers 16 8k;           # 压缩缓冲区
}
```

**gzip_comp_level 的选择**：

| 级别 | 压缩率（JSON） | CPU 开销 | 适用 |
|------|-------------|---------|------|
| 1 | ~50% | 很低 | 纯吞吐场景 |
| 5 | ~70% | 中 | **通用推荐** |
| 9 | ~80% | 高 | 带宽极贵/BGP 跨地域传输 |

**关键**：`gzip` 是 CPU 密集操作。如果后端响应已经是压缩的（如 `Content-Encoding: gzip`），Nginx 的 `gzip` 不会再次压缩（已经标记了 `gzip_proxied` 需要配合 `proxy_set_header Accept-Encoding`）。

## 3.2 proxy_buffer——代理响应的缓冲与磁盘 offload

```nginx
server {
    location /api/ {
        proxy_pass http://backend;
        
        proxy_buffering on;          # 默认 on
        proxy_buffer_size 4k;        # 响应头的缓冲区
        proxy_buffers 8 16k;         # 响应体的缓冲区（8 × 16KB = 128KB）
        proxy_busy_buffers_size 32k;  # 在缓冲还未读完时就能开始发的量
        proxy_max_temp_file_size 256m; # 响应超过缓冲大小时存磁盘的临时文件上限
        
        # 当响应 > proxy_buffers 总容量时：
        # → 超出的部分写临时文件（磁盘 IO）
        # → 如果不想写磁盘，加大 proxy_buffers 数量/大小
    }
}
```

**缓冲区大小的经验公式**：

```
你的 API 的典型响应大小 = X

X < 128KB → 默认的 proxy_buffers (8 16k = 128KB) 足够
X > 128KB → 考虑加大 proxy_buffers 或设置为 X × 1.5
X 变化很大 → 保持默认 + 监控 proxy_temp_file 的 IO 量
```

## 3.3 open_file_cache——文件描述符缓存

```nginx
http {
    open_file_cache max=10000 inactive=60s;  # 缓存 1 万个文件的 fd
    open_file_cache_valid 30s;               # 每 30s 验证一次文件是否仍然有效
    open_file_cache_min_uses 2;              # 至少被访问 2 次才进入缓存
    open_file_cache_errors on;               # 缓存"文件不存在"的结果（减少磁盘查询）
}
```

这个优化是**静态文件场景专用**。如果 Nginx 主要代理动态请求，`open_file_cache` 几乎没有作用。对静态资源密集型站点（如图片 CDN 源站），缓存 1 万个文件描述符可以减少大量磁盘元数据查询。

---

# 四、完整的生产性能配置

```nginx
# === 全局段 ===
worker_processes auto;
worker_cpu_affinity auto;
worker_rlimit_nofile 65535;
error_log /var/log/nginx/error.log warn;
pid /var/run/nginx.pid;

# === events 段 ===
events {
    worker_connections 10240;
    multi_accept on;
    accept_mutex off;
    use epoll;
}

# === http 段 ===
http {
    # 基础优化
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    server_tokens off;           # 不暴露 Nginx 版本号
    keepalive_timeout 65;        # 客户端 keepalive
    keepalive_requests 100;      # 一条连接最多处理 100 个请求
    types_hash_max_size 2048;    # MIME 类型哈希表大小
    server_names_hash_bucket_size 64;
    
    # 文件缓存（静态文件场景）
    open_file_cache max=10000 inactive=60s;
    open_file_cache_valid 30s;
    open_file_cache_min_uses 2;
    open_file_cache_errors on;
    
    # 压缩
    gzip on;
    gzip_comp_level 5;
    gzip_min_length 1000;
    gzip_types text/plain text/css application/json application/javascript;
    gzip_vary on;
    
    # 日志
    access_log /var/log/nginx/access.log main buffer=32k flush=5s;
    # buffer=32k: 攒够 32KB 再写盘；flush=5s: 5s 不攒够也强刷（防丢日志）
    
    include /etc/nginx/conf.d/*.conf;
}
```

---

# 五、总结

| 层级 | 配置 | 一句话 |
|------|------|--------|
| **CPU** | `worker_processes auto` | Worker 数 = CPU 核数 |
| **CPU** | `worker_cpu_affinity auto` | Worker 绑核，减少缓存失效 |
| **连接** | `worker_connections 10240+` | 一个 Worker 能处理的并发连接 |
| **IO** | `sendfile on` | 静态文件从磁盘 DMA 直发网卡 |
| **TCP** | `tcp_nopush on` | 攒够再发，减少包数 |
| **TCP** | `tcp_nodelay on` | keepalive 小包立刻发，降延迟 |
| **压缩** | `gzip_comp_level 5` | CPU 和带宽的最佳平衡 |
| **缓冲** | `proxy_buffers 16 16k` | 响应超 256KB 触磁盘 IO → 加大 |
| **文件** | `open_file_cache` | 静态文件场景省磁盘元数据查询 |

# 延伸阅读

**Do——动手验证：**
- 用 `strace -c -p <worker-pid>` 观察 sendfile 开启/关闭时 `sendfile` 和 `read/write` 系统调用数量的差异
- 对比 `gzip_comp_level 1` vs `5` vs `9` 的 CPU 使用率和响应体大小（用 `curl -H "Accept-Encoding: gzip" -w "%{size_download}"` 测压缩后大小）
- 用 `ss -s` 观察 TIME_WAIT 数量，对比配置 `tcp_tw_reuse=1` 前后的差异

**Todo——深入方向：**
- Nginx 的 `aio` 指令——异步 I/O 在线程池中的实现（`thread_pool` 指令）
- HTTP/2 的 `http2_push_preload`——Server Push 在 Nginx 中的性能影响
- TLS 调优——`ssl_session_cache` / `ssl_session_tickets` / OCSP stapling 对 HTTPS 连接建立延迟的影响

*本文参考资料：*
- Nginx 官方文档: ngx_core_module / ngx_http_core_module / ngx_http_proxy_module
- Jeff Dean, "The Tail at Scale" (CACM 2013) —— 与缓冲/降级相关的尾部延迟问题
- Linux 内核文档: tcp(7) —— tcp_nodelay / tcp_cork 的语义
