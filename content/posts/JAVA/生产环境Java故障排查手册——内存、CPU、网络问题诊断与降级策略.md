# 生产环境 Java 故障排查手册——内存、CPU、网络问题诊断与降级策略

> 凌晨三点，告警群弹出一条"服务可用率下降到 99.7%"。你打开监控发现 GC 频率飙升、CPU 打满、线程池队列排到几千——该从哪下手？本文梳理 Java 生产环境最常见的四类问题：内存、CPU、网络、磁盘 IO，每类都给出现象→诊断路径→解决方案→降级兜底。

---

## 一、故障诊断工具箱速览

生产环境不能打断点、不能加日志重启。你只能靠 JVM 自带的工具和在线诊断工具。

| 工具 | 作用 | 典型命令 |
|------|------|---------|
| `top -H` | 找 CPU 最忙的线程 | `top -H -p <pid>` |
| `jstack` | 线程快照，看死锁、阻塞、等待 | `jstack <pid>` |
| `jstat` | GC 统计，各代内存使用率 | `jstat -gcutil <pid> 1000` |
| `jmap` | 堆内存 dump，分析对象分布 | `jmap -dump:live,file=heap.hprof <pid>` |
| `vmstat` | 系统级：上下文切换、CPU、内存 | `vmstat 1` |
| `iostat` | 磁盘 IO 使用率 | `iostat -x 1` |
| `netstat/ss` | 网络连接状态 | `ss -s` |
| Arthas | 在线诊断利器 | `dashboard`/`thread`/`trace`/`watch`/`vmtool` |

Arthas 是阿里的开源项目，不侵入进程，attach 上去就能用。是所有工具里唯一能在线看方法耗时、热替换代码、追踪调用链的工具。生产环境排障时，`dashboard`（全局看板）和 `thread -n 3`（最忙的三个线程）是最常用的两个入口命令。

---

## 二、内存问题

### 2.1 Heap OOM

**现象**：应用日志出现 `java.lang.OutOfMemoryError: Java heap space`，服务假死或频繁 Full GC 最终崩掉。

**诊断路径**：

```
1. 确认 OOM 类型
   → 日志搜 "OutOfMemoryError"，确认是 Heap space 还是其他

2. 保留现场
   → -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/tmp/heap.hprof
   → 如果没配置 OOM 自动 dump，用 jmap -dump:live,file=heap.hprof <pid>

3. MAT / JProfiler 分析 dump
   → 看 Dominator Tree：哪些对象占了 > 10% 堆内存？
   → 看 Leak Suspects：自动报告可疑泄漏点
   → 看 Path to GC Roots：为什么它不能被回收？
```

**常见根因与解决**：

| 根因 | 特征 | 解决方案 |
|------|------|---------|
| 静态集合类无限增长 | 某个 ConcurrentHashMap 或 ArrayList 占了 60% 堆 | 换成 LRU 缓存（Caffeine）或定期清理 |
| ThreadLocal 没 remove | 线程池 + ThreadLocal → 线程复用导致 value 永不被 GC | try-finally 中 `threadLocal.remove()` |
| 流式查询没关闭 | ResultSet/FileInputStream 忘记 close | try-with-resources 或查连接池泄漏 |
| 大对象频繁创建 | char[] 或 byte[] 占满堆 | 减小对象大小、复用、用堆外内存 |
| 死循环拼接字符串 | StringBuilder 在循环中无限 append | 加长度上限或分批处理 |
| -Xmx 配太小 | 正常业务对象总数超过堆上限 | 扩容 -Xmx + 检查业务量是否异常增长 |

**线上应急处置**：

```
1. jmap -dump 保留现场（必须，否则重启后证据消失）
2. 重启服务（最快止血）
3. 暂时调大 -Xmx（治标，给分析争取时间）
4. 分析 dump 找根因
```

注意：`jmap -dump:live` 会触发一次 Full GC（live 参数的含义是先做 GC 再 dump），对大堆来说这次 FGC 可能持续几十秒。如果服务已经在 OOM 边缘，这次 FGC 可能是压死骆驼的最后一根稻草——服务直接不可用。权衡：如果服务还能勉强跑，用 `jmap -dump`（不带 live）先快速 dump，虽然 dump 文件更大但不会触发 GC。

### 2.2 Metaspace OOM

**现象**：`java.lang.OutOfMemoryError: Metaspace`，常发生在应用启动一段时间后或大量动态类加载场景。

**根因**：

- 动态生成大量 Class（CGLIB 代理、Groovy 脚本、JSP 编译）
- `-XX:MaxMetaspaceSize` 设得太小
- 类加载器泄漏（ClassLoader 无法被 GC，导致其加载的类也常驻 Metaspace）

Metaspace OOM 比 Heap OOM 更难排查——因为 Metaspace 不在堆内，MAT 看不到。需要 `jstat -gc <pid>` 看 MU (Metaspace Used) 是否持续增长；或者 `jcmd <pid> VM.classloader_stats` 查看 ClassLoader 数量和加载类数。

**解决方案**：

```bash
# 查看 Metaspace 使用情况
jstat -gcutil <pid> 1000
# 关注 M (Metaspace usage)：如果持续增长，有泄漏

# 排查 ClassLoader 泄漏（Arthas）
classloader -t       # 列出所有 ClassLoader 及其加载的类数
# 找类数量异常大的 ClassLoader

# JVM 参数
-XX:MaxMetaspaceSize=256m     # 生产必须设上限，默认无上限
-XX:+TraceClassLoading         # 开启类加载日志（排查用，性能影响大）
-XX:+TraceClassUnloading       # 开启类卸载日志
```

常见修复：CGLIB 动态代理重复创建 → 缓存代理对象；Groovy 脚本每次执行 new 一个 GroovyClassLoader → 复用 ClassLoader。

### 2.3 直接内存 OOM

**现象**：`java.lang.OutOfMemoryError: Direct buffer memory`，Heap 使用率看着不高，但进程 RSS 远超 -Xmx。

**根因**：
- Netty 的 DirectByteBuffer 没 release（最常见）
- NIO 的 `ByteBuffer.allocateDirect()` 忘记回收
- `-XX:MaxDirectMemorySize` 没设或设太小

诊断上，直接内存不在堆内、不在 jmap 中、MAT 看不到——只能通过进程 RSS（`ps aux | grep java` 的 RSS 列）和 `-Xmx` 的差值推断。直接内存的回收机制是靠 `DirectByteBuffer` 的 Cleaner 虚引用在 GC 时触发，所以如果堆内存没压力、GC 不触发 → DirectByteBuffer 永远不回收 → 直接内存 OOM。解决手段是主动调用 `System.gc()`（不推荐）或者在 Netty 中用 `ReferenceCountUtil.release()`。

**解决方案**：

```java
// ❌ 忘记释放
ByteBuf buf = ctx.alloc().directBuffer();
ctx.writeAndFlush(buf);
// buf 没有 release！

// ✅ try-finally 或使用 SimpleChannelInboundHandler（自动释放）
ByteBuf buf = ctx.alloc().directBuffer();
try {
    ctx.writeAndFlush(buf);
} finally {
    ReferenceCountUtil.release(buf);
}

// JVM 参数
-XX:MaxDirectMemorySize=512m
```

**监控**：`jcmd <pid> VM.native_memory summary`（需开启 `-XX:NativeMemoryTracking=summary`）。找到 Internal 行中的 direct buffer 实际占用量。

### 2.4 GC 问题

**现象分类与诊断**：

| 现象 | 诊断命令 | 常见根因 |
|------|---------|---------|
| Young GC 频率过高 | `jstat -gc <pid> 1000` 看 YGC 增长速度 | 朝生夕死的临时对象太多；Eden 区太小 |
| Full GC 频繁 | `jstat -gcutil <pid> 1000` 看 FGC 次数 | 老年代持续增长又无法回收 → 内存泄漏 |
| 单次 GC 停顿太长 | GC 日志 `-Xlog:gc*` | 堆太大（>32GB）；Remark 阶段耗时长；使用 Serial GC |
| GC 后内存回落很多 | `jstat` 每列的变化量 | 正常，对象被回收 |
| GC 后内存几乎不动 | `jstat` 中 OU（老年代）持续增长 | 内存泄漏 |

**GC 调优参数速查**：

```bash
# 通用推荐（JDK 11+/17+）
-XX:+UseG1GC
-XX:MaxGCPauseMillis=200     # 期望停顿目标（G1 尽力而为，不是硬约束）
-XX:G1HeapRegionSize=4m      # Region 大小，堆 8GB 以下用 2m，16GB 以上用 4-8m
-XX:ConcGCThreads=2          # 并发 GC 线程数
-XX:InitiatingHeapOccupancyPercent=45  # 老年代占 45% 时触发并发标记周期
-Xms4g -Xmx4g                # 生产强烈建议 -Xms = -Xmx，避免动态扩容
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/data/logs/

# 大堆(>32GB) 或 极低延迟(<10ms) 场景
-XX:+UseZGC
-XX:ZCollectionInterval=0    # ZGC 可以跑得极其频繁，停顿极小
```

**Full GC 频繁的应急处置**：

```
1. jmap -dump 保留现场
2. 如果服务已不可用 → 直接重启
3. 如果还能用但 FGC 越来越密 →
   - 调大 -Xmx（给内存泄漏更多缓冲时间）
   - 排查泄漏源
4. 修复后灰度上线
```

### 2.5 内存问题诊断命令速查

```bash
# 查看堆内存使用和各代占用
jstat -gcutil <pid> 1000

# 查看堆外内存（需要 NMT 开启）
jcmd <pid> VM.native_memory summary

# 线程数量趋势（每个线程默认占 1MB 栈外内存）
jstack <pid> | grep "^Thread" | wc -l

# 查看 GC 次数和时间
jstat -gc <pid>
# YGC: Young GC 次数  YGCT: Young GC 总耗时
# FGC: Full GC 次数   FGCT: Full GC 总耗时

# 系统内存概览
free -h
cat /proc/<pid>/status | grep VmRSS     # 进程实际物理内存
```

---

## 三、CPU 问题

### 3.1 CPU 飙升——如何定位到代码行

这是生产环境最常见的告警。CPU > 80% 持续几分钟，你需要知道**哪个线程、哪段代码在消耗 CPU**。

**标准诊断流程**：

```bash
# 1. 找到 Java 进程 PID
top                 # 或 jps -l

# 2. 找到进程中 CPU 最高的线程
top -H -p <pid>
# 记录 PID 列中 CPU 最高的几个线程的 TID（十进制）

# 3. 十进制 TID → 十六进制（jstack 用十六进制）
printf "%x\n" 12345
# 输出: 3039

# 4. jstack 看线程堆栈
jstack <pid> | grep 0x3039 -A 30
# 找到线程当前执行的代码位置

# 或者用 Arthas 一步到位：
thread -n 3          # 打印 CPU 最高的 3 个线程的堆栈
```

**常见 CPU 飙升根因**：

| 根因 | 特征 | 解决 |
|------|------|------|
| 死循环 | 一个线程 CPU 100%，长时间不退 | 加循环边界条件或超时保护 |
| 正则表达式灾难回溯 | 线程卡在 `Pattern.compile` 或 `matcher.find()` | 限制正则复杂度 + 超时 `(?^...)`；或预编译 + 换匹配算法 |
| 频繁 Full GC | GC 线程 CPU 高，应用线程正常 | 排查内存泄漏 |
| 大量 JSON 序列化 | 线程卡在 `writeValueAsString`，对象特别大 | 只序列化需要的字段、分页传输 |
| HashMap 严重冲突 | JDK7 及以下，恶意构造 key 让链表退化为 O(n) | 升级 JDK8+（红黑树优化）或限制入参大小 |
| 日志异步队列满 | 线程卡在 `logback AsyncAppender` → 不停重试 | 调大队列容量或降级为同步写 |
| 线程池拒绝策略 | 线程卡在 RejectedExecutionHandler 的循环重试 | 选择合适的拒绝策略 |

**正则回溯的坑**：

```java
// ❌ 灾难性回溯：对长字符串做复杂匹配
Pattern pattern = Pattern.compile("(a+)+b");
pattern.matcher("aaaaaaaaaaaaaaaaaaaaaaaaaaaaa!").matches();  // CPU 爆炸

// ✅ 限制
Pattern pattern = Pattern.compile("(a+)+b");
Matcher matcher = pattern.matcher(input);
// Java 没有内置正则超时，需要自己包装 Future + timeout
```

### 3.2 死锁

**现象**：部分请求超时，但 CPU 不高，`jstack` 看到 `BLOCKED` 状态。

**诊断**：

```bash
# jstack 自动检测死锁
jstack <pid> | grep -A 200 "deadlock"

# Arthas
thread -b          # 找出当前阻塞其他线程的线程
```

**jstack 会自动在输出末尾列出检测到的死锁**——线程 A 持锁 L1 等 L2，线程 B 持锁 L2 等 L1。不需要手动分析。

**死锁排查顺序**：
1. 确认涉及哪些锁 → 找到对应的 `synchronized` 块或 `ReentrantLock`
2. 分析锁的获取顺序是否有循环依赖 → 统一加锁顺序（所有线程按相同顺序获取锁）
3. 短锁：缩小同步块范围，不把 I/O 操作放在锁内

### 3.3 线程池耗尽

**现象**：请求大量超时，`jstack` 中大量线程 `WAITING` 等待获取线程池的任务队列。

```bash
# 查看线程状态分布
jstack <pid> | grep "java.lang.Thread.State" | sort | uniq -c | sort -nr
```

输出中 `TIMED_WAITING` 和 `WAITING` 占绝大多数是正常的（线程大部分时间在等待任务）。但如果大量线程是 `BLOCKED` 或 `RUNNABLE` 但一直在处理同一段代码——前者是死锁/锁竞争，后者是 CPU 热点或死循环。

**根因**：
- 线程池配置过小 → 任务排队等待
- 下游服务超时 → 线程阻塞等待响应 → 新请求拿不到线程
- 线程池队列无界 → 排队任务堆积 → OOM

**线程池配置原则**：

```java
// IO 密集型：线程数 = CPU 核数 × (1 + 阻塞系数)
// 阻塞系数 = 阻塞时间 / (CPU时间 + 阻塞时间)，取 0.8-0.9
int ioThreads = Runtime.getRuntime().availableProcessors() * 2;

ThreadPoolExecutor pool = new ThreadPoolExecutor(
    ioThreads,
    ioThreads * 2,
    60L, TimeUnit.SECONDS,
    new LinkedBlockingQueue<>(2000),     // 有界队列
    new ThreadPoolExecutor.CallerRunsPolicy()  // 反压
);

// CPU 密集型：线程数 ≈ CPU 核数 + 1
int cpuThreads = Runtime.getRuntime().availableProcessors() + 1;
```

### 3.4 CPU 问题应急处置

```
1. 定位到 CPU 热点代码（Arthas thread -n 3）
2. 如果是死循环 / 正则回溯 → 重启服务（最快止血）
3. 如果是线程池耗尽 →
   - 临时扩容 pod
   - 下游服务熔断（防止线程继续阻塞）
   - 限流（拒绝新增请求）
4. 如果是 GC 导致 → 按内存问题处理
```

### 3.5 CPU 诊断命令速查

```bash
# 系统整体 CPU
vmstat 1 5
# 关注 r(运行队列)、us(用户态CPU)、sy(内核态CPU)、cs(上下文切换)

# 上下文切换频繁 → 可能线程数太多或锁竞争严重
pidstat -w -p <pid> 1
# cswch/s: 自愿上下文切换（等 I/O、等锁、sleep）
# nvcswch/s: 非自愿切换（CPU 时间片用完被抢走）

# perf 看 CPU 热点（Linux 原生工具，比 arthas 开销更低）
perf top -p <pid>
```

**内核态 CPU (sy) 持续高** → 频繁系统调用、频繁线程切换、或者 GC 线程正在密集工作。前者需要优化代码减少 syscall（比如批量 read/write 替代逐字节操作）；后者需要排查 GC。**cs 极高**（>10 万/秒）→ 线程数太多或锁竞争严重。

---

## 四、网络问题

### 4.1 连接超时与连接池耗尽

**现象细分**：

- 调用下游服务大量超时 → 下游慢或挂了
- 连接建立超时（connect timeout）→ 下游端口不通或防火墙拦截
- 连接池耗尽 → 从池中获取不到连接，"无法获取连接"的报错通常是获取超时（borrow timeout）而非连接建立超时
- TCP 连接数飙升 → ss -s 看到大量 ESTABLISHED → 可能是连接泄漏

**连接池配置审查**：

```java
// HttpClient 连接池（Apache HttpClient）
PoolingHttpClientConnectionManager cm = new PoolingHttpClientConnectionManager();
cm.setMaxTotal(200);                          // 最大连接数
cm.setDefaultMaxPerRoute(50);                 // 每个路由（目标 host:port）最大连接数

CloseableHttpClient client = HttpClients.custom()
    .setConnectionManager(cm)
    .setDefaultRequestConfig(RequestConfig.custom()
        .setConnectTimeout(3000)              // 建立连接超时 3s
        .setSocketTimeout(5000)               // 读取超时 5s
        .setConnectionRequestTimeout(2000)    // 从连接池获取连接超时 2s
        .build())
    .setRetryHandler(new DefaultHttpRequestRetryHandler(1, false))  // 只重试 1 次
    .build();
```

三个超时是不同含义：连接建立超时是 TCP 三次握手的等待上限；读取超时是对方回包的第一个字节必须在多长时间内到；获取连接超时是等待连接池中有可用连接的时间。很多线上故障是因为只关注了前两个而忽略了第三个——下游一慢，连接全部被占用，新请求拿不到连接直接报错。

### 4.2 TIME_WAIT 风暴

**现象**：`netstat -an | grep TIME_WAIT | wc -l` 看到几万个 TIME_WAIT 连接，新连接建立失败或变慢。

**根因**：服务作为 TCP 客户端频繁创建和关闭连接（如每次 HTTP 请求都 new 一个连接），连接关闭后进入 TIME_WAIT 状态（默认持续 60 秒）。这个状态是 TCP 协议保证"延迟的包不会污染新连接"的机制，关不掉也不该关。解决方案不是调内核参数 (`tcp_tw_reuse`)，而是在应用层复用连接。

**解决方案**：
- **连接池复用**：不用短连接，用 HttpClient 连接池 → 从根本上减少连接的创建和关闭
- 如果用短连接是不可避免的（比如调用方太多太分散），调整 `tcp_tw_reuse = 1` 让内核重用处于 TIME_WAIT 的 socket
- `tcp_tw_recycle` 在 NAT 环境下会导致连接 RST——**永远不要开启**

### 4.3 TCP 半连接队列/全连接队列溢出

**现象**：`ss -s` 或 `netstat -s | grep -i listen` 中出现 `SYNs to LISTEN sockets dropped` 或 `times the listen queue of a socket overflowed`。

这就是 TCP 的 **SYN Queue（半连接队列）** 和 **Accept Queue（全连接队列）** 满了——三次握手完成了，但 `accept()` 线程来不及处理，新连接被丢弃。

**根因**：`accept()` 线程（Netty 的 BossGroup 或 Tomcat 的 Acceptor 线程）处理不过来。两个可能：连接涌入速度超过 accept 能力（瞬时峰值），或者 accept 线程被阻塞在做非 accept 的事情上。

**解决方案**：
```bash
# Linux 内核参数
net.core.somaxconn = 4096        # 全连接队列最大长度
net.ipv4.tcp_max_syn_backlog = 8192  # 半连接队列最大长度
```

应用层：
- Netty：`.option(ChannelOption.SO_BACKLOG, 4096)`
- Tomcat：`server.tomcat.accept-count=200`
- 如果持续溢出 → 水平扩容（增加 pod 数量分担连接压力）

### 4.4 网络问题诊断命令速查

```bash
# 连接状态统计
ss -s                     # 每种状态的 socket 数量

# TOP 10 连接数最多的目标 IP/端口
ss -tan | awk '{print $5}' | cut -d: -f1 | sort | uniq -c | sort -nr | head -10

# 查看是否丢包
netstat -s | grep -i retrans       # TCP 重传统计
netstat -s | grep -i "listen"      # 队列溢出统计
ping -c 100 <host>                 # 简单看是否有持续丢包

# 查看指定 IP 的连接数（排查单个下游连接泄漏）
ss -tan | grep 192.168.1.100 | wc -l

# 网络延迟
# 注意：ping 只测 ICMP 延迟，TCP 应用延迟可能完全不同
curl -o /dev/null -s -w "connect: %{time_connect}s, TTFB: %{time_starttransfer}s, total: %{time_total}s\n" <url>
```

---

## 五、磁盘 IO 问题

### 5.1 IO Wait 高

**现象**：`top` 中 `wa` (IO Wait) 列持续 > 30%。

**常见根因**：
- GC 日志同步写（`-Xloggc` 没有异步缓冲）
- 业务日志同步落盘
- 文件读写没有缓冲
- 磁盘性能本身不行（HDD 慢、云盘 IOPS 配额耗尽）

**解决方案**：
```bash
# 定位高 IO 的进程
iotop -o                    # 按 IO 排序

# 定位高 IO 的文件
lsof -p <pid> | grep -E "REG.*w"   # 进程打开的写文件

# GC 日志改异步（关键！）
-Xlog:gc*:file=/data/logs/gc.log:time,level,tags   # JDK 11+

# Logback 使用异步 Appender
<appender name="ASYNC" class="ch.qos.logback.classic.AsyncAppender">
    <queueSize>4096</queueSize>
    <discardingThreshold>0</discardingThreshold>  <!-- 队列满时不丢日志 -->
    <appender-ref ref="FILE"/>
</appender>
```

### 5.2 磁盘空间满

**现象**：日志写入失败，服务异常。

**快速排查**：
```bash
df -h                           # 各挂载点使用率
du -sh /data/logs/* | sort -rh | head -10   # 最大的 10 个目录
find /data/logs -name "*.log" -mtime +7 -delete  # 删 7 天前的日志
lsof | grep deleted              # 已删除但进程还持有的文件（僵尸文件）
```

`lsof | grep deleted` 是最容易被忽略的：你 `rm` 了文件但进程的文件句柄还开着，磁盘空间不会释放——内核要等到所有引用该 inode 的 fd 都关闭后才真正回收磁盘空间。必须重启进程才能真正释放空间。

---

## 六、系统性故障的降级策略

单点问题好排查，系统性故障时才考验架构的容灾能力。

### 6.1 降级决策树

```
流量暴涨 / 下游大面积超时
      │
      ▼
┌──────────────┐
│ 1. 限流       │  入口层拦截，保护核心链路
│    - Nginx    │  全局限流：限制总 QPS
│    - Sentinel │  服务限流：按接口/资源限流
└──────┬───────┘
       │ 还不够
       ▼
┌──────────────┐
│ 2. 熔断       │  下游不可用 → 快速失败，不阻塞
│    - Sentinel │  熔断后 fallback 返回降级结果
│    - 方法级   │  读接口→返回缓存数据；写接口→返回重试提示
└──────┬───────┘
       │ 还不够
       ▼
┌──────────────┐
│ 3. 功能降级   │  关闭非核心功能
│    - 关推荐   │  首页推荐 → 返回固定热门列表
│    - 关详情   │  商品详情 → 返回缓存快照
│    - 关日志   │  日志级别动态调高 → ERROR only
└──────┬───────┘
       │ 还不够
       ▼
┌──────────────┐
│ 4. 服务降级   │  整个非核心服务下线
│    - 停营销   │  营销、积分、推荐 直接 503 → 前端屏蔽入口
│    - 只保留   │  下单、支付 核心链路
└──────────────┘
```

### 6.2 Sentinel 降级配置

```java
// 方法级熔断降级
@SentinelResource(
    value = "getOrderDetail",
    fallback = "getOrderDetailFallback",     // 异常时降级
    blockHandler = "getOrderDetailBlockHandler"  // 限流/熔断时降级
)
public OrderDetail getOrderDetail(String orderId) {
    return orderService.query(orderId);
}

// 降级方法：返回缓存数据
public OrderDetail getOrderDetailFallback(String orderId, Throwable e) {
    log.warn("订单详情降级: orderId={}", orderId, e);
    OrderDetail cached = cacheService.get(orderId);
    if (cached != null) {
        cached.setFromCache(true);  // 前端可据此展示"数据可能不是最新"
        return cached;
    }
    throw new BizException("服务繁忙，请稍后重试");
}
```

**Fallback 和 BlockHandler 的区别**：Fallback 是业务异常时的兜底（比如下游 HTTP 超时、抛出异常）；BlockHandler 是流量控制触发时的兜底（QPS 超了、线程数超了被 Sentinel 拦下来）。两者分开定义可以让你在监控上区分"是下游挂了导致降级"还是"流量太大被限流了"。

### 6.3 降级开关设计

```java
// 配置中心（Nacos/Apollo）动态控制降级开关
@Component
public class DegradeSwitch {
    
    @Value("${degrade.recommend.enabled:true}")
    private boolean recommendEnabled;
    
    @Value("${degrade.cacheOnly.enabled:false}")
    private boolean cacheOnlyMode;
    
    public List<Product> getRecommend(String userId) {
        if (!recommendEnabled) {
            return getDefaultRecommend();  // 降级：返回固定推荐
        }
        if (cacheOnlyMode) {
            return cacheService.get("recommend:" + userId);  // 降级：只查缓存
        }
        return recommendService.calculate(userId);  // 正常：个性化推荐
    }
}
```

关键：降级开关必须能**动态生效**（配置中心 push）、**有兜底**（配置中心不可用时降级到关闭状态）、**可监控**（开关状态上报到监控系统）。

---

## 七、故障响应 SOP

```
告警触发
    │
    ▼
[0-2 分钟]  确认：是真的故障还是误告警？
              → 看监控大盘：QPS、RT、错误率
              → 看最近变更：谁发版了？
    
[2-5 分钟]  止血：让服务先活着
              → CPU 高 → 重启
              → 内存高 → dump + 重启
              → 下游超时 → 熔断降级
              → 流量暴涨 → 限流 + 扩容
    
[5-15 分钟] 排查根因
              → Arthas 在线诊断
              → 看错误日志
              → 看慢 SQL / GC 日志
    
[15+ 分钟]  修复与复盘
              → 修复代码/配置
              → 灰度验证
              → 写复盘报告（时间线 + 根因 + 改进措施）
```

**第一条铁律**：先止血，再排查。服务每多不可用 1 分钟，影响范围都在扩大。不要为了保留现场而拖延重启——dump 可以在重启前快速拿走（`jmap -dump:live` 几秒到几十秒），在此之后应该立刻重启恢复服务。

---

生产环境的故障排查，本质是**快速定位 + 快速止血 + 事后根除**三项能力的组合。工具是手段，不是目的。真正值钱的是"看到 CPU 飙升 → 脑子里自动浮现五种可能根因 → 逐一排除"的经验积累。
