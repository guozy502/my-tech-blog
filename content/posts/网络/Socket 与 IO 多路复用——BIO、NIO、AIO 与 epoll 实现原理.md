---
title: "Socket 与 IO 多路复用——BIO、NIO、AIO 与 epoll 实现原理"
date: 2026-07-29
description: 从阻塞 IO 的"一个连接一个线程"到非阻塞 IO 的轮询浪费、IO 多路复用的"一个线程监听所有连接"、再到 epoll 的底层回调机制（红黑树+rdllist），拆解 BIO/NIO/AIO 的演进逻辑与 select/poll/epoll 的性能差异。
tags: ["网络","Socket","epoll","IO多路复用","BIO","NIO"]
categories: ["网络"]
---

# 历史背景——C10K 催生了 epoll

1999 年，Dan Kegel 在《The C10K Problem》中描述了一个问题：如何让一台服务器同时处理 10000 个客户端连接？当时的主流方案是"一个连接一个线程"——每个连接分配一个线程（或进程），线程阻塞在 `read()` 上等数据。当连接数到达几千时，操作系统光是线程上下文切换就耗光了 CPU。

答案在 Linux 2.6 内核（2003 年）中——`epoll`。它不是"让 IO 处理更快"，而是**改变了 IO 模型**——不再需要"为每个连接分配一个线程"，而是"用一个线程监听所有连接"。这个模型今天几乎所有高性能服务器（Nginx、Redis、Netty、Go netpoller）都在用。

---

# 一、BIO（Blocking IO）——一个连接一个线程

```c
// 经典 BIO 模型：accept + read 都是阻塞的
void bio_server() {
    int listen_fd = socket(AF_INET, SOCK_STREAM, 0);
    bind(listen_fd, ...);
    listen(listen_fd, 10);
    
    while (1) {
        int client_fd = accept(listen_fd, ...);  // ← 阻塞，等新连接
        // 为每个连接创建一个线程
        pthread_create(&thread, NULL, handle_client, &client_fd);
    }
}

void* handle_client(void* arg) {
    int fd = *(int*)arg;
    char buf[1024];
    while (1) {
        int n = read(fd, buf, sizeof(buf));  // ← 阻塞，等数据
        if (n <= 0) break;
        write(fd, buf, n);
    }
    close(fd);
}
```

```
BIO 的问题：
  连接数 10000 → 线程数 10000
  每个线程默认栈 8MB → 80GB 内存（仅栈空间！）
  线程上下文切换 ~1μs/次 → 10000 线程的切换开销吞噬 CPU
```

---

# 二、NIO（Non-blocking IO）——轮询的浪费

```c
// 非阻塞 IO：read 不阻塞，没数据立刻返回
void nio_server() {
    int listen_fd = socket(AF_INET, SOCK_STREAM, 0);
    fcntl(listen_fd, F_SETFL, O_NONBLOCK);  // ← 设为非阻塞
    bind(listen_fd, ...);
    listen(listen_fd, 10);
    
    int fds[10000];  // 存储所有客户端 fd
    int fd_count = 0;
    
    while (1) {
        // ① 非阻塞 accept
        int client_fd = accept(listen_fd, ...);
        if (client_fd != -1) {
            fcntl(client_fd, F_SETFL, O_NONBLOCK);
            fds[fd_count++] = client_fd;
        }
        
        // ② 遍历所有 fd，逐个尝试 read（轮询！）
        for (int i = 0; i < fd_count; i++) {
            int n = read(fds[i], buf, sizeof(buf));
            if (n > 0) process(fds[i], buf, n);
        }
    }
}
```

```
NIO 的问题：
  每次循环都要遍历所有 fd 并尝试 read → 10000 个 fd 中可能只有 5 个有数据
  → 9995 次无效的系统调用（read 返回 EAGAIN）
  → CPU 空转在"检查没数据的连接"上
```

---

# 三、IO 多路复用——select / poll / epoll

## 3.1 select——"把 10000 个 fd 打包发给内核，内核告诉你哪些就绪了"

```c
void select_server() {
    fd_set read_fds;         // 位图：用 1 bit 表示 1 个 fd
    int max_fd = listen_fd;
    
    while (1) {
        FD_ZERO(&read_fds);
        FD_SET(listen_fd, &read_fds);      // 把 listen_fd 加入集合
        for (int i = 0; i < fd_count; i++)
            FD_SET(fds[i], &read_fds);      // 把每个客户端 fd 加入集合
        
        // select 阻塞，直到有 fd 就绪
        int ready = select(max_fd + 1, &read_fds, NULL, NULL, NULL);
        
        for (int i = 0; i <= max_fd; i++) {
            if (FD_ISSET(i, &read_fds)) {   // 检查哪些 fd 就绪
                if (i == listen_fd) accept(...);
                else process(i, ...);
            }
        }
    }
}
```

**select 的三个致命缺陷**：
1. `fd_set` 是位图，**最大只能存 1024 个 fd**（`FD_SETSIZE`）
2. 每次调用 select 都要**把整个 fd_set 从用户态拷贝到内核态**——高并发下拷贝开销大
3. 内核遍历 fd_set 的方式是**全量扫描**——O(N)，N 是所有注册的 fd 数量

## 3.2 poll——select 的改良版（去掉 1024 限制）

```c
// poll 用链表替代位图，没有 1024 限制
struct pollfd fds[10000];
fds[0].fd = listen_fd;
fds[0].events = POLLIN;

int poll(struct pollfd *fds, nfds_t nfds, int timeout);

// 改进：没有 fd 数量限制
// 没改：每次还是传入整个数组（拷贝开销），内核还是全量扫描
```

## 3.3 epoll——真正解决 C10K 的答案

```c
void epoll_server() {
    // ① 创建 epoll 实例（内核分配一个 eventpoll 对象）
    int epfd = epoll_create1(0);
    
    // ② 把 listen_fd 注册到 epoll（只注册一次！）
    struct epoll_event ev;
    ev.events = EPOLLIN;
    ev.data.fd = listen_fd;
    epoll_ctl(epfd, EPOLL_CTL_ADD, listen_fd, &ev);
    
    struct epoll_event events[MAX_EVENTS];
    
    while (1) {
        // ③ 等待事件——内核只把就绪的 fd 返回，不需要遍历全部
        int nfds = epoll_wait(epfd, events, MAX_EVENTS, -1);
        
        for (int i = 0; i < nfds; i++) {
            int fd = events[i].data.fd;
            if (fd == listen_fd) {
                int client = accept(listen_fd, ...);
                ev.events = EPOLLIN;
                ev.data.fd = client;
                epoll_ctl(epfd, EPOLL_CTL_ADD, client, &ev);  // 新连接注册
            } else {
                process(fd, ...);
            }
        }
    }
}
```

## 3.4 epoll 为什么快？——三个关键数据结构

```
epoll 在 Linux 内核中的实现：

① 红黑树（rbtree）—— 存储所有注册的 fd
   epoll_ctl(ADD/MOD/DEL) 操作 O(logN)
   只在添加/修改/删除时用到，不影响 epoll_wait 性能

② 就绪链表（rdllist）—— 存储所有就绪的 fd
   当内核收到数据时：网卡驱动 → 内核协议栈 → 找到 fd → 
   调用 epoll 的回调函数 → 把该 fd 加入 rdllist
   epoll_wait 只遍历 rdllist → O(1) 就绪的 fd 数

③ 回调机制（callback）
   每个 fd 注册到 epoll 时，epoll 在 fd 的等待队列上挂一个回调
   当 fd 变为可读/可写时，内核调用这个回调 → 把 fd 放入 rdllist
   → 不需要 epoll_wait 主动轮询任何 fd
```

```
select/poll 的工作方式：
  每次 select/poll(传送整个fd集合) → 内核全量扫描(fds) → 找到就绪的 → 返回
  复杂度：O(N)，N = 所有 fd

epoll 的工作方式：
  注册时(只一次)：epoll_ctl 把 fd 加入红黑树，挂回调
  等待时：epoll_wait 从 rdllist 取就绪 fd
  内核收到数据时：回调自动把 fd 放入 rdllist
  复杂度：epoll_wait = O(1) 就绪数，与总 fd 数无关
```

## 3.5 三兄弟对比

| | select | poll | epoll |
|------|--------|------|-------|
| **fd 上限** | 1024 | 无限制 | 无限制 |
| **每次扫描方式** | 内核全量扫描 O(N) | 内核全量扫描 O(N) | 回调驱动 O(1) 就绪数 |
| **每次传入数据** | 整个 fd_set 拷贝 | 整个 pollfd 数组拷贝 | 不需要每次传（红黑树常驻内核） |
| **适用场景** | 极少量 fd (<100) | 少量 fd (<1000) | **任何规模** |

---

# 四、边缘触发 vs 水平触发

```c
// 水平触发（Level-Triggered, LT）—— 默认模式
// epoll_wait 返回后，如果没读完数据，下次 epoll_wait 还会返回这个 fd
ev.events = EPOLLIN;  // 默认 = LT

// 边缘触发（Edge-Triggered, ET）
// epoll_wait 只在"无数据→有数据"状态变化时通知一次
// 必须一次性读完（读到 EAGAIN），否则丢事件
ev.events = EPOLLIN | EPOLLET;

// ET 模式配合非阻塞 IO（必须！因为要读到 EAGAIN 为止）
int n;
while ((n = read(fd, buf, sizeof(buf))) > 0) {
    process(buf, n);
}
if (n == -1 && errno == EAGAIN) {
    // 读完了，下次有数据时 epoll 会再次通知
}
```

---

# 五、总结

| 模型 | 连接:线程 | 内核/用户交互 | 适用 |
|------|----------|------------|------|
| **BIO** | 1:1 | read/write 阻塞 | 极低并发 |
| **NIO** | 1:1 | read 非阻塞 + 轮询 | 同 BIO，加了非阻塞 |
| **select/poll** | N:1 | 每次传入全量 fd 集合 | <1000 fd |
| **epoll** | N:1 | 红黑树常驻内核 + 回调驱动 | **高并发标准答案** |

> **epoll 的哲学：把"找谁就绪"的工作从用户态搬到内核态——不是"用户一个个问内核"，而是"内核主动告诉用户谁就绪了"。**

# 延伸阅读

**Do——动手验证：**
- `strace -e epoll_create,epoll_ctl,epoll_wait nginx` 启动 Nginx 并观察 epoll 系统调用
- 用 Python 分别写 select/poll/epoll 版本的 echo server，对比 1000 连接时的 CPU 使用率
- `cat /proc/sys/fs/epoll/max_user_watches` 查看系统 epoll 最大监控数

**Todo——深入方向：**
- `io_uring`（Linux 5.1+）——比 epoll 更激进的异步 IO，连系统调用次数都省了
- Go 语言的 netpoller——如何在 goroutine 层面封装 epoll/kqueue，实现"同步写法、异步执行"
- `EPOLLEXCLUSIVE`（Linux 4.5+）——解决多进程 accept 惊群问题的内核级方案

*本文参考资料：*
- Dan Kegel, "The C10K Problem" (1999)
- Linux 内核源码 `fs/eventpoll.c`（epoll 实现）
- W. Richard Stevens《UNIX Network Programming, Volume 1》
