---
title: "Nginx 架构深度拆解——Master-Worker 进程模型与事件驱动引擎"
date: 2026-07-29
description: 从 C10K 问题的历史背景出发，拆解 Nginx 的 Master-Worker 多进程模型、Worker 间负载均衡从 accept_mutex 到 EPOLLEXCLUSIVE 的演进、非阻塞 I/O + epoll 事件循环的处理流程，以及与 Apache prefork/event 模式的全维度对比。
tags: ["Nginx","架构","Master-Worker","epoll","事件驱动","C10K"]
categories: ["网络"]
---

# 历史背景——C10K 问题与 Nginx 的诞生

2002 年，Igor Sysoev 在俄罗斯一家门户网站做运维时，碰到了一个让他无法忍受的问题：Apache HTTP Server（当时的主流）在处理高并发连接时表现得"像牛车"。当时所有网站的服务器架构都差不多——Apache prefork 模式，一个连接一个进程。当同时在线用户超过 5000 人时，光是进程切换就让 CPU 跑满。这就是后来 Dan Kegel 在 1999 年文章中命名的 **C10K 问题**。

Igor Sysoev 花了两年时间，在 2004 年发布了 Nginx 的第一个版本。他的设计哲学和 Apache 截然相反：**不是给每个连接分配一个进程，而是用一个进程处理大量连接——通过事件驱动来区分"谁有数据可读/可写"**。这个架构差异使得 Nginx 在同样硬件上能处理的并发连接数是 Apache 的 10-50 倍。

理解 Nginx 的关键认知是：**Nginx 不是"网络编程厉害"，而是"操作系统机制用得好"。** 它把 Linux 内核的 `epoll`、`sendfile`、非阻塞 socket、共享内存等机制正确地组装在了一起，而不是发明了什么新的网络模型。

---

# 一、Master-Worker 进程模型——一个指挥官 + 一群干活的

## 1.1 What：Nginx 有哪些进程？

```bash
# 启动 Nginx，然后用 ps 查看
ps -ef | grep nginx

# 典型输出：
# root      1234     1  0 10:00 ?        00:00:00 nginx: master process
# www       1235  1234  0 10:00 ?        00:00:03 nginx: worker process
# www       1236  1234  0 10:00 ?        00:00:02 nginx: worker process
# www       1237  1234  0 10:00 ?        00:00:04 nginx: worker process
# www       1238  1234  0 10:00 ?        00:00:01 nginx: worker process

# 还有一个可选的 cache manager 进程（管理缓存文件）
# 和一个可选的 cache loader 进程（启动时加载磁盘缓存到内存索引）
```

```
┌────────────────────────────────────┐
│          Master Process            │
│  • 读取并验证配置文件                │
│  • 创建/绑定/监听共享 socket         │
│  • 启动/终止/管理 Worker 进程        │
│  • 处理信号（reload/reopen/stop）    │
│  • 不处理客户端请求！                │
└──────┬──────┬──────┬──────┬────────┘
       │      │      │      │
  ┌────┴┐  ┌──┴──┐ ┌─┴───┐ ┌┴────┐
  │ W1  │  │ W2  │ │ W3  │ │ W4  │   ← Worker 数量 = CPU 核数
  │epoll│  │epoll│ │epoll│ │epoll│      每个 Worker 是单线程的
  └─────┘  └─────┘ └─────┘ └─────┘      所有 Worker 共享监听同一组端口
```

## 1.2 Why：为什么不用多线程？

多线程模型（如 Apache worker 模式）存在三个冲突：

1. **锁竞争**：所有工作线程共享一个连接队列，每次 accept 都要抢锁。在高并发下，锁竞争本身消耗的 CPU 可能超过实际处理请求的 CPU
2. **上下文切换**：几百个线程 = 几百次调度/秒 = 上下文切换开销 5-15% CPU
3. **内存开销**：每个线程有自己的栈（默认 8MB），2 万个线程 = 160GB（超过物理内存）

Nginx 的 Worker 是**单线程**的——一个 Worker 进程 = 一个线程 = 一个事件循环。没有锁竞争、没有线程切换、没有共享数据。每个 Worker 独立调用 `epoll_wait` 从内核获取就绪的连接，互不干扰。

## 1.3 How：Master 和 Worker 如何通信？

```
Master ──(信号)──> Worker   ← Master 通过 kill 发信号给 Worker
Worker ──(共享内存)── Worker ← Worker 之间通过共享内存交换缓存/统计信息

信号机制：
  nginx -s reload  → Master 收到 SIGHUP
    → Master 启动新一批 Worker（加载新配置）
    → 旧 Worker 不再接受新连接，处理完现有请求后优雅退出
    → 实现了"不丢一个请求的配置热更新"

  nginx -s stop    → Master 收到 SIGTERM → 通知 Worker 退出
  nginx -s reopen  → Master 收到 SIGUSR1 → Worker 重新打开日志文件
```

---

# 二、Worker 如何抢连接——从 accept_mutex 到 EPOLLEXCLUSIVE

## 2.1 惊群问题（Thundering Herd）

Nginx 的所有 Worker 共享同一个监听 socket。当一个新连接到达时，内核会唤醒**所有**在 `epoll_wait` 上等待的 Worker。但只有一个 Worker 能 `accept` 到这个连接——其余被唤醒的 Worker 都是"空跑一趟"。

这就是**惊群效应（Thundering Herd）**，在高并发短连接场景下极其浪费 CPU。

## 2.2 两代解决方案

**第一代：accept_mutex（Nginx 默认，直到 1.11.3）**

```
accept_mutex on;  ← 默认开启

Worker 在调用 accept 之前，必须先获取 accept_mutex 锁：
  Worker 1 抢到锁 → 去 accept 连接 → 释放锁
  其余 Worker 没有锁 → 继续等 epoll_wait

缺点：在低延迟/高吞吐场景下，抢锁本身的开销不可忽略
      且锁只允许一个 Worker accept，在连接非常密集时成为瓶颈
```

**第二代：EPOLLEXCLUSIVE（Linux 4.5+，Nginx 1.11.3+）**

```
accept_mutex off;  ← 关闭用户态锁，让内核来解决

每个 Worker 在向 epoll 注册监听 socket 时，带上 EPOLLEXCLUSIVE 标志：
  epoll_ctl(epfd, EPOLL_CTL_ADD, listen_fd, 
            EPOLLIN | EPOLLEXCLUSIVE);
  
内核保证：同一个监听 socket 上，当有新连接到达时，
         只唤醒一个等待的 Worker（而不是全部）

优势：不需要用户态锁 → 零抢锁开销 → 吞吐更高
前提：Linux 内核 >= 4.5，glibc >= 2.24
```

```nginx
# nginx.conf
events {
    accept_mutex off;          # 关闭用户态锁（Nginx 1.11.3+）
    worker_connections 65535;   # 每个 Worker 最大连接数
    use epoll;                  # Linux 下显式指定 epoll
}
```

---

# 三、非阻塞 I/O + epoll 事件循环——Worker 的内部机制

## 3.1 非阻塞 I/O 的哲学

Nginx 把所有客户端 socket 都设置为 **非阻塞模式（O_NONBLOCK）**：

```c
// Nginx 连接处理的底层逻辑（简化伪代码）
int s = accept(listen_fd, ...);
fcntl(s, F_SETFL, O_NONBLOCK);  // ← 确保不会阻塞

// 非阻塞读：有数据就返回，没有立刻返回 EAGAIN
int n = recv(s, buf, size, 0);
if (n == -1 && errno == EAGAIN) {
    // 没数据 → 不等待 → 处理下一个连接
    return;
}

// 非阻塞写：能写多少写多少，写不完下次继续
int n = send(s, buf, len, 0);
if (n == -1 && errno == EAGAIN) {
    // socket 缓冲区满了 → 先等等 → 把剩的加回 epoll 等可写事件
    epoll_ctl(epfd, EPOLL_CTL_MOD, s, EPOLLOUT);
}
```

**核心思想**：**永远不阻塞在任何一个 socket 上**。如果一个 socket 暂时不可读/不可写，Worker 立刻转到下一个就绪的 socket。这保证了即使有几万个连接，只要大多数处于空闲状态，单个 Worker 就能轻松管理。

## 3.2 epoll 事件循环——Worker 的核心循环

```c
// Nginx Worker 核心循环（极度简化）
void worker_cycle() {
    for (;;) {
        // ① 等待事件（阻塞直到有事件就绪或超时）
        n = epoll_wait(epfd, events, MAX_EVENTS, timeout);
        
        for (i = 0; i < n; i++) {
            int fd = events[i].data.fd;
            
            // ② 监听 socket 可读 = 新连接到了 → accept
            if (fd == listen_fd) {
                int client = accept(listen_fd, ...);
                fcntl(client, F_SETFL, O_NONBLOCK);
                epoll_ctl(epfd, EPOLL_CTL_ADD, client, EPOLLIN | EPOLLET);
                // ↑ 边缘触发（ET）：只在状态变化时通知，减少 epoll_wait 返回次数
            }
            
            // ③ 客户端 socket 可读 = HTTP 请求到了 → 解析处理
            else if (events[i].events & EPOLLIN) {
                read_request_and_process(fd);
            }
            
            // ④ 客户端 socket 可写 = 响应准备好了 → 发送
            else if (events[i].events & EPOLLOUT) {
                send_response(fd);
            }
        }
        
        // ⑤ 处理定时器（超时的连接、缓存的到期等）
        process_timers();
    }
}
```

## 3.3 边缘触发 vs 水平触发

Nginx 默认使用 **EPOLLET（边缘触发）**：

```c
// 水平触发（Level-Triggered）：只要缓冲区有数据就通知
// 优点：简单可靠，不会丢事件
// 缺点：如果数据没读完，每次 epoll_wait 都返回这个 fd（重复通知）

// 边缘触发（Edge-Triggered）：只在"无数据→有数据"这个状态变化时通知一次
// 优点：减少重复通知，降低 epoll_wait 开销
// 缺点：必须一次性把数据读完（读到 EAGAIN），否则丢事件
epoll_ctl(epfd, EPOLL_CTL_ADD, fd, EPOLLIN | EPOLLET);

// Nginx 的处理：
int n;
do {
    n = recv(fd, buf, size, 0);
    // 一次性读到返回 EAGAIN（缓冲区空了）
} while (n > 0);
```

**为什么边缘触发在高并发时更优？** 因为减少了 `epoll_wait` 的返回次数——一次通知把所有数据读完，下次有数据时内核再通知，不会在数据还在缓冲区时就反复通知。

---

# 四、Nginx vs Apache——架构差异决定了性能上限

| | Apache prefork | Apache worker/event | Nginx |
|------|--------------|-------------------|-------|
| **进程模型** | 每个连接一个进程 | 多进程 + 多线程 | 多进程 + 单线程 epoll |
| **并发处理方式** | `select` / 阻塞 accept | `select` / 阻塞 accept | `epoll` / 非阻塞 边缘触发 |
| **内存占用** | ~10MB/连接（进程栈） | ~2MB/连接（线程栈） | ~2KB/连接（1 个连接状态 struct） |
| **5 万并发连接** | ~500GB（不可能） | ~100GB | ~100MB |
| **CPU 开销** | 上下文切换主导 | 线程切换 + 锁竞争 | 事件循环，几乎没有上下文切换 |
| **长连接/WebSocket** | ❌ 每个连接占一个进程 | ❌ 每个连接占一个线程 | ✅ 空闲连接几乎零开销 |
| **静态文件** | 慢（每次都要读磁盘） | 慢 | 极快（sendfile 零拷贝） |
| **动态内容** | 通过 mod_php 等模块 | 通过反向代理 | **反向代理到后端应用**（Nginx 本身不处理动态逻辑） |

**这个表解释了 Nginx 为什么适合做"反向代理/静态资源/负载均衡"三层工作**：

- 反向代理：大量空闲连接等后端响应 → Nginx 的并发成本极低
- 静态文件：`sendfile` 系统调用直接从磁盘到网卡，不经过用户态
- 负载均衡：内置 upstream 模块，不需要额外的 HAProxy

---

# 五、关键配置

```nginx
# 进程相关
worker_processes auto;          # Worker 数 = CPU 核数（auto 自动检测）
worker_cpu_affinity auto;       # Worker 绑定到固定 CPU 核（减少缓存失效）
worker_rlimit_nofile 65535;     # Worker 能打开的最大文件描述符数

# 事件相关
events {
    worker_connections 65535;    # 每个 Worker 的最大并发连接数
    use epoll;                   # Linux 下的事件模型
    accept_mutex off;            # 1.11.3+ 配合 EPOLLEXCLUSIVE
    multi_accept on;             # 一次 accept 尽可能多接收连接（减少 epoll_wait 次数）
}

# 查看 Nginx 编译信息和版本
# nginx -V
```

---

# 六、总结

| 组件 | 解决的问题 | 核心原理 |
|------|----------|---------|
| **Master 进程** | 配置管理 + Worker 生命周期 | 不处理请求，只管理配置和信号 |
| **Worker 进程** | 处理所有客户端请求 | 单线程 + epoll + 非阻塞 I/O |
| **多 Worker 共享端口** | 利用多核 CPU | SO_REUSEPORT（Linux 3.9+）或 accept_mutex/EPOLLEXCLUSIVE |
| **epoll 驱动** | 管理数万连接 | 只关注"有数据的连接"，空闲连接零开销 |
| **边缘触发** | 减少事件通知次数 | 只在状态变化时通知，一次性读完 |
| **非阻塞 I/O** | 避免阻塞单个连接卡死 Worker | 读不完/写不完就换下一个，不等待 |

> Nginx 架构的三个核心公式：**多进程（利用多核）+ 单线程 epoll（无锁 + 零切换开销）+ 非阻塞 I/O（不阻塞在任何连接上） = 单机百万并发连接的基础。**

# 延伸阅读

**Do——动手验证：**
- `nginx -V` 查看编译参数和启用的模块
- `strace -p <worker-pid>` 观察 Worker 进程的 `epoll_wait` / `accept4` / `recvfrom` 系统调用时序
- 修改 `worker_processes 1` vs `auto`，用 `wrk` 压测对比 QPS 和 CPU 利用率

**Todo——深入方向：**
- SO_REUSEPORT（Linux 3.9+）的多进程共享端口机制——每个 Worker 自己 `bind + listen`，内核做负载均衡
- Nginx 的定时器红黑树实现——为什么用红黑树而不是时间轮或最小堆
- Nginx 的内存池（`ngx_pool_t`）——为每个请求分配一整个内存池，请求结束一次性回收

*本文参考资料：*
- Nginx 官方文档: Development Guide (Core Architecture)
- Andrew Alexeev (Nginx, Inc.), "Inside NGINX: How We Designed for Performance & Scale" (2012)
- Nginx 源码 `src/os/unix/ngx_process_cycle.c`（Master-Worker 进程循环）
- Nginx 源码 `src/event/modules/ngx_epoll_module.c`（epoll 事件模块）
- Dan Kegel, "The C10K Problem" (1999): http://www.kegel.com/c10k.html
