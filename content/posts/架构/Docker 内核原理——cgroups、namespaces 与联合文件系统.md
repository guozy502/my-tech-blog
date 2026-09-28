---
title: "Docker 内核原理——cgroups、namespaces 与联合文件系统"
date: 2026-08-07
description: 从容器"不是轻量级虚拟机"的本质出发，拆解 Docker 依赖的三个 Linux 内核机制——cgroups 的资源限制（CPU/内存/IO）、namespaces 的视图隔离（PID/NET/MOUNT/UTS/IPC/USER）、联合文件系统 OverlayFS 的分层镜像与写时复制，以及 JVM 在容器中 OOM 的根因与解决方案。
tags: ["架构","Docker","容器","cgroups","namespaces","OverlayFS"]
categories: ["架构"]
---

# 历史背景——容器不是"轻量级虚拟机"

2013 年 Docker 发布时，很多人把它理解为"更快的虚拟机"。这个认知是错误的——Docker **没有在宿主机上虚拟出一个新的 OS 内核**。每个容器和宿主机共享同一个 Linux 内核。容器隔离靠的是三个 Linux 内核特性：

1. **cgroups**（Control Groups）：限制容器能用多少 CPU、内存、磁盘 IO
2. **namespaces**：让每个容器看到自己被隔离的进程树、网络栈、文件系统、用户
3. **联合文件系统（OverlayFS）**：镜像是分层存储的——每层只读，容器层通过写时复制（Copy-on-Write）来修改

这三个机制是 Linux 2.6（2003-2007 年）就有的，比 Docker 早了约十年。Docker 的创新不是发明了容器，而是**把这三个机制打包成了一个开发者友好的工具**——Dockerfile 写镜像、镜像仓库共享、一行命令启停，降低到了普通程序员的日常工具范畴。

理解这三个机制，才能回答"JVM 为什么会在容器中 OOM"、"容器和 VM 的 IO 性能差在哪"、"为什么 Docker 镜像要分层写"这些问题。

---

# 一、cgroups——限制资源使用

## 1.1 为什么需要 cgroups？

在没有 cgroups 的系统中，一个进程理论上可以用满所有 CPU 和内存。在容器化部署中这是灾难性的——某个容器的内存泄漏不仅会杀死自己，还可能导致同一台宿主机上的其他容器被 OOM Killer 杀掉。

cgroups 的解决方案是：**给每个容器定义一个"资源使用上限"**——容器内的进程以为自己有机器上所有的 CPU 和内存，但实际上被 cgroups 硬限制在配置的范围内。

## 1.2 cgroups v2 的核心文件系统接口

```bash
# 查看某个容器的 cgroup 限制
ls /sys/fs/cgroup/system.slice/docker-<container-id>.scope/

# CPU 限制
cat /sys/fs/cgroup/.../cpu.max
# 输出: 200000 100000
# 含义: 每 100ms（100000µs）周期中，最多使用 200ms（200000µs）的 CPU
# → 等价 "2 个 CPU 核"

# 内存限制
cat /sys/fs/cgroup/.../memory.max
# 输出: 536870912
# 含义: 最多使用 512MB 内存

# 内存当前使用量
cat /sys/fs/cgroup/.../memory.current
```

```bash
# cgroups v2 的六个核心控制器（subsystem）
cpu       → 限制 CPU 使用率（CFS 调度器）
memory    → 限制内存 + swap 使用量
cpuset    → 绑定容器到特定 CPU 核心和 NUMA 节点
io        → 限制块设备 IOPS 和 BPS
pids      → 限制容器内可创建的进程数
hugetlb   → 限制大页内存的使用
```

## 1.3 JVM 在容器中被 OOM Kill 的经典问题

```
场景：
  Docker 启动 Java 应用 → docker run --memory=512m
  JVM 运行时参数：-Xmx1024m（JVM 以为自己有 1024MB 可用）
  但 cgroups 把该容器的内存硬限制在 512MB
  → JVM Heap 用到了 512MB → memory.current 达到 memory.max
  → Linux OOM Killer 被触发 → 容器直接被 kill
  → 日志里不是 OutOfMemoryError，而是容器进程突然消失

根因：
  JDK 8 的 JVM 默认读的是宿主机总内存，不是 cgroup 限制的内存
  → JVM 以为自己有 16GB → 设 Heap 为 4GB → 实际只能用 512MB → OOM Kill

解决方法：
  JDK 8u131+：-XX:+UnlockExperimentalVMOptions -XX:+UseCGroupMemoryLimitForHeap
  JDK 8u191+：-XX:InitialRAMPercentage=50 -XX:MaxRAMPercentage=75
  JDK 10+：UseContainerSupport 默认开启 → JVM 自动读 cgroup 限制而不是宿主机内存
```

---

# 二、namespaces——六个"视觉隔离"的维度

## 2.1 每个 namespace 是"你在哪个世界看东西"

namespaces 不是"创建了一个新世界"，而是"你看到的世界被限制在了一个子集"。Linux 有 6 种 namespace：

| namespace | 隔离什么 | 容器内看到的效果 | 不加隔离的后果 |
|-----------|---------|---------------|-------------|
| **PID** | 进程 ID 编号空间 | 容器内 `ps aux` 只看到自己的进程，PID 从 1 开始 | 容器内 `kill -9 1` 会杀掉宿主机的 init |
| **NET** | 网卡、IP、路由表、端口 | 容器内 `ifconfig` 只看到分配给自己虚拟网卡 | 两个容器用同一个端口会冲突 |
| **MOUNT** | 文件系统挂载点 | 容器内 `/` 是容器的根文件系统，看不到宿主机 `/` | 容器内 `rm -rf /` 会删掉宿主机根目录 |
| **UTS** | 主机名和域名 | 容器内 `hostname` = 容器 hostname | 容器改 hostname 会影响宿主机 |
| **IPC** | 信号量、消息队列、共享内存 | 容器不能通过 IPC 读取其他容器的共享内存 | 不安全 |
| **USER** | 用户和用户组 ID 映射 | 容器内 UID=0(root) 映射到宿主机 UID=1000(普通用户) | 容器内的 root 就是宿主机的 root |

## 2.2 PID namespace——进程树的隔离

```bash
# 宿主机上看（PID namespace 0）
ps aux | grep java
# root  12345  java -jar app.jar   ← PID=12345

# 同一个容器内看（自己 PID namespace）
ps aux
# root  1  java -jar app.jar        ← PID=1！
```

**为什么容器内的 init 进程 PID=1？** 因为内核为每个 PID namespace 维护了独立的进程编号空间。容器内第一个进程被映射为 PID 1（这是该 namespace 的 init 进程）。它特殊在——如果 PID 1 退出，内核会杀掉该 namespace 中的所有其他进程（这是 PID namespace 的"孤儿进程清理"规则）。这就是为什么 Dockerfile 中 CMD 启动的 Java 进程要用 exec（`exec java -jar app.jar`）替代 shell 脚本——否则 shell 成为 PID 1，你的 Java 应用是 PID 7，shell 退出后系统感知不到应该由谁带头。

## 2.3 NET namespace——网络的隔离

```bash
# 查看某个容器的网络 namespace（Docker 背后做的事）
# 创建一对 veth（虚拟网卡）
# 一端在容器内：eth0 → IP=172.17.0.2
# 一端在宿主机：vethxxxx → 接在 docker0 网桥上

# 宿主机上查看
ip link show | grep veth
# vethAB1234@if5: ... master docker0 ...

# 容器内查看
docker exec <container-id> ip addr
# eth0@if6: ... inet 172.17.0.2/16 ...
```

**容器网络的本质**：每个容器有自己的 NET namespace → 有独立的网卡、IP、路由表、端口号空间 → 两个容器可以同时在 8080 端口监听而不冲突（因为它们的端口号空间是独立的）。容器之间的通信通过宿主机的网桥（docker0）做 L2 转发。

---

# 三、联合文件系统（OverlayFS）——分层的镜像

## 3.1 为什么镜像是分层的？

```
Dockerfile 的每一行 RUN/COPY/ADD 产生一层：

FROM ubuntu:22.04          ← 层 1: ubuntu 基础文件系统（只读）
RUN apt-get update          ← 层 2: 更新 apt 索引（只读）
RUN apt-get install -y java ← 层 3: 安装 JDK（只读）
COPY app.jar /app/          ← 层 4: 应用 jar（只读）
CMD ["java", "-jar", "app.jar"]

实际运行时，Docker 在这些只读层之上创建一个可写层（容器层）。

拉取镜像时："Layer already exists" → Docker 检测到某些层已在本地缓存过，
不需要重新下载。这就是为什么 FROM ubuntu:22.04 → 
继续 pull 其他应用镜像时基础层叠加很快。
```

## 3.2 OverlayFS 的目录结构

```
/var/lib/docker/overlay2/
  ├── <layer-id-1>/              ← 层 1 的文件（ubuntu 基础）
  │     └── diff/                ← 这个层的文件系统变化
  ├── <layer-id-2>/              ← 层 2 的文件（apt update）
  │     └── diff/
  ├── <merged>/                  ← OverlayFS 合并视图（容器内看到的整个文件系统）
  │     ├── bin/
  │     └── ...
  └── <work>/                    ← OverlayFS 的工作目录（内部使用）

容器内执行 ls / 时，其实是在读取 <merged> 目录。
<merged> 是 OverlayFS 把所有层叠加在一起后的"统一视图"。
```

```bash
# 查看某个容器的 OverlayFS 挂载信息
docker inspect <container-id> | grep -A 5 "GraphDriver"
# "LowerDir": "层4:层3:层2:层1"（只读层从左到右优先级递增）
# "UpperDir": "容器可写层"（修改和新增都在这里）
# "MergedDir": "合并视图"（所有层叠在一起的结果）
```

## 3.3 写时复制（Copy-on-Write）

```
当容器中需要修改文件时（例如改 /etc/hosts）：
  ① 这个文件存在于某个只读层（LowerDir）中
  ② OverlayFS 不能直接修改只读层 → 把它拷贝到可写层（UpperDir）
  ③ 在可写层中修改这个文件
  ④ 容器下次读这个文件 → UpperDir 中的版本（覆盖 LowerDir 的版本）

当容器中新增文件时：
  直接写在可写层（UpperDir）中

当容器中删除文件时：
  在可写层创建一个"whiteout"文件（标记为"已删除"）→ 
  OverlayFS 合并时把这个文件隐藏掉

性能影响：
  - 首次修改大文件（如 500MB 的日志文件）时会触发 COW 拷贝 → 额外延迟
  - 频繁修改大量小文件 → 可写层膨胀 → 需要清理
  - 读不触发 COW，只有写触发
```

**这和 Docker 的最佳实践直接相关**：

```
为什么 Dockerfile 要把变化频率低的放在前面？
  FROM ubuntu        ← 几乎不变
  RUN apt install    ← 偶尔变
  COPY app.jar       ← 经常变（每次发版变）
  → 代码变更时只需要重新构建最后一层 → push/pull 极快

为什么不要在容器中频繁写大文件？
  COW 会把大文件从只读层拷到可写层 → 磁盘空间从 500MB 变 1GB
  → 如果要写大量数据 → 用 Volume（绕过 OverlayFS，直接写在宿主机目录）
```

---

# 四、容器 vs 虚拟机的本质差异

| | 虚拟机 | 容器 |
|------|--------|------|
| **内核** | 每个 VM 有独立的 Guest OS 内核 | 所有容器共享宿主机的 Linux 内核 |
| **隔离机制** | Hypervisor + 硬件虚拟化 | cgroups + namespaces |
| **启动速度** | 分钟级（启动 Guest OS） | 秒级（启动进程即可） |
| **镜像大小** | GB 级（虚拟磁盘文件） | MB 级（分层叠加的最小差异） |
| **内存开销** | 每个 VM 有独立的 OS 内存开销（~500MB+） | 每个容器只多一个进程（~几MB） |
| **安全隔离** | 强（内核级物理隔离） | 弱（共享内核，存在逃逸风险） |

---

# 五、总结

| 机制 | 解决的问题 | Docker 中的体现 |
|------|---------|---------------|
| **cgroups** | 限制资源使用 | `docker run --memory=512m --cpus=2` |
| **PID namespace** | 隔离进程树 | 容器内 PID 从 1 开始，`ps aux` 只看到自己 |
| **NET namespace** | 隔离网络 | 每个容器独立 IP、端口空间 |
| **MOUNT namespace** | 隔离文件系统 | 容器内 `/` 是 OverlayFS 的 merged 层 |
| **OverlayFS** | 分层存储 | 镜像层只读，容器层 COW 可写 |
| **veth pair** | 容器连接外部网络 | 虚拟网卡对：容器内 eth0 ↔ 宿主机 veth |

# 延伸阅读

**Do——动手验证：**
- `docker run --memory=100m openjdk:17 java -XshowSettings:vm -version` 观察 JVM 读到的内存是容器限制还是宿主机总内存
- `ls /sys/fs/cgroup/system.slice/docker-$(docker inspect -f '{{.Id}}' <container>).scope/` 查看容器的 cgroup 参数
- `docker run -d nginx && nsenter -t $(docker inspect -f '{{.State.Pid}}' <container>) -n ifconfig` 进入容器的 NET namespace 查看网络

**Todo——深入方向：**
- Seccomp 与 Linux Security Modules（LSM）——容器安全的三层防护（cgroups 资源 + namespace 隔离 + seccomp 系统调用过滤）
- runc 与 OCI 运行时规范——Docker 是如何调用 runc 来创建和启动容器的
- Kubernetes 的 CNI 插件（Calico / Cilium）——如何用 eBPF 替代 iptables 做容器网络

*本文参考资料：*
- Linux Kernel Documentation: cgroups v2 / namespaces
- Docker 官方文档: Storage Driver (OverlayFS)
- Red Hat, "Containers are Linux" (2016)
