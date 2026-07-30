---
title: "网络排查实战——tcpdump、Wireshark 与四个经典故障复盘"
date: 2026-07-29
description: 从网络排查四步法（ping→DNS→curl→traceroute）、tcpdump 的常见过滤表达式与输出解读、Wireshark 的 TCP 流追踪和时间序列分析，到四个经典生产故障（TIME_WAIT 过多/CLOSE_WAIT 泄漏/HTTPS 握手慢/TCP 重传风暴）的完整排查路径。
tags: ["网络","tcpdump","Wireshark","故障排查","TCP重传","CLOSE_WAIT"]
categories: ["网络"]
---

# 历史背景——网络排查的"工具箱"

网络问题的排查比其他层面的问题更让人头疼——数据库慢了你可以在查询上加索引，应用崩溃了你可以看堆栈。但网络问题往往表现为"有时候好、有时候坏"、"在这个机器上能通、那个机器上不通"、"昨天还是好的、今天突然不行了"。

网络排查的核心工具链 20 年来没有本质变化：`ping`（通不通）、`traceroute`（走哪条路）、`tcpdump`/`Wireshark`（把路上的包抓出来看）。变化的是问题场景——分布式系统让连接数的量级从几十变成了几万，给排查增加了量级级别的复杂度。

---

# 一、网络排查四步法——"这位患者哪里不舒服？"

```
第一步：ping —— 网络层通不通？
  ping 8.8.8.8
  → 通 → 说明 IP 层没问题 → 问题在 TCP/应用层
  → 不通 → 检查交换机/网线/防火墙/路由

第二步：DNS —— 域名解析是否正常？
  dig www.example.com
  → 有 IP 返回 → DNS 正常
  → 没有 → 检查 DNS 服务器配置或权威 DNS 记录

第三步：curl —— HTTP 通信是否正常？
  curl -v http://localhost:8080/api/health
  → 200 OK → HTTP 应用层健康
  → 连接拒绝 → 端口没开或应用没启动
  → 超时 → 防火墙拦截或路由不对

第四步：traceroute —— 走哪条路？
  traceroute -n 8.8.8.8
  → 看每一跳的 RTT 和丢包率
  → 某跳丢包突然变大 → 这一跳的链路有问题
```

---

# 二、tcpdump——Linux 网络排查的瑞士军刀

## 2.1 常用过滤表达式

```bash
# === 按主机过滤 ===
tcpdump host 192.168.1.100                        # 只看这个 IP 的包
tcpdump src 10.0.1.5                              # 只看这个 IP 发出的包
tcpdump dst 8.8.8.8                               # 只看发给这个 IP 的包

# === 按端口过滤 ===
tcpdump port 80                                   # 只看 80 端口
tcpdump portrange 8000-9000                       # 端口范围

# === 按协议过滤 ===
tcpdump tcp                                       # 只看 TCP
tcpdump 'tcp[tcpflags] & (tcp-syn|tcp-fin) != 0'  # 只看 SYN 或 FIN

# === 组合过滤 ===
tcpdump -i eth0 host 10.0.1.5 and port 3306       # 只看某台机器到 MySQL 的流量
tcpdump -i lo port 8080 -w local.pcap             # 抓本地回环并保存到文件

# === 只看握手包 ===
tcpdump 'tcp[tcpflags] & tcp-syn != 0'            # 只看 SYN
tcpdump 'tcp[tcpflags] == tcp-syn'                # 精确匹配 SYN

# === 只看 RST ===
tcpdump 'tcp[tcpflags] & tcp-rst != 0'

# === 只看重传包 ===
tcpdump 'tcp[tcpflags] & tcp-syn == 0 and tcp[tcpflags] != tcp-ack' -r file.pcap
# 更精确的方式：用 Wireshark 打开，看 tcp.analysis.retransmission
```

## 2.2 tcpdump 输出解读

```
16:30:00.123456 IP 10.0.1.5.54321 > 10.0.1.100.3306: Flags [S], seq 123456, win 64240, options [mss 1460], length 0
│           │       │     │       │     │       │        │          │                │           │
│           │       │     │       │     │       │        │          │                │           └ 数据长度
│           │       │     │       │     │       │        │          │                └ MSS=1460
│           │       │     │       │     │       │        │          └ TCP 窗口大小
│           │       │     │       │     │       │        └ 序列号
│           │       │     │       │     │       └ Flags: S=SYN, .=ACK, P=PUSH, F=FIN, R=RST
│           │       │     │       │     └ 目标端口
│           │       │     │       └ 目标 IP
│           │       │     └ 源端口
│           │       └ 源 IP
│           └ 时间戳(微秒精度)
└ IP
```

---

# 三、四个经典故障复盘

## 场景 1：TIME_WAIT 过多——Nginx 代理后端疯狂建连/断连

```bash
# 症状：Nginx 日志出现大量超时，ss -tan | grep TIME_WAIT | wc -l → 20000+

# 排查步骤：
ss -tan | awk '{print $1}' | sort | uniq -c | sort -rn
# → TIME_WAIT: 25000, ESTABLISHED: 30, ...
# → TIME_WAIT 占总连接的 99% → 短连接场景

# 根因：
# Nginx upstream 配置中没有 keepalive → 每次代理请求都是一次独立的 TCP 连接
# → 请求完成 Nginx 主动 close → 进入 TIME_WAIT → 2MSL(60s) 后释放
# → 2000 QPS × 60s = 120000 个 TIME_WAIT 同时存在 → 端口耗尽！

# 止血：
# ① Nginx upstream 加 keepalive 64 → 连接复用，不再频繁建连/断连
# ② tcp_tw_reuse = 1 → 允许 TIME_WAIT 端口被复用
# ③ 改变 Nginx → 后端的方向：让后端做主动关闭方（后端进入 TIME_WAIT）
```

## 场景 2：CLOSE_WAIT 泄漏——应用忘记 close 连接

```bash
# 症状：API 服务运行几天后无响应，ss -tan 发现几千个 CLOSE_WAIT

# 排查步骤：
ss -tan | grep CLOSE_WAIT | wc -l  # → 5200
ss -tanp | grep CLOSE_WAIT           # → 确认是哪个进程

# 根因：
# Java 应用代码：
HttpURLConnection conn = url.openConnection();
InputStream is = conn.getInputStream();
// ... 读取数据 ...
// 忘记了 conn.disconnect() 或 is.close()！
# → 远端关闭了连接（发 FIN）
# → 本地 TCP 栈收到 FIN → 进入 CLOSE_WAIT
# → 本地应用没有调 close() → TCP 永远不会进入 LAST_ACK → CLOSE_WAIT 永远存在

# 止血：
# ① 代码：finally { conn.disconnect(); }
# ② 用连接池 (HttpClient) 代替手动管理连接
# ③ 重启应用（CLOSE_WAIT 是 OS 层面的状态，应用重启后清空）
```

## 场景 3：HTTPS 握手耗时高——证书链太长

```bash
# 症状：某些用户反馈页面加载很慢（尤其是国外用户），但服务器 CPU 和带宽正常

# 排查步骤：
curl -w "TCP handshake: %{time_connect}s\nSSL handshake: %{time_appconnect}s\nTotal: %{time_total}s\n" \
     -o /dev/null -s https://example.com

# 输出：
# TCP handshake: 0.015s  ← TCP 三次握手很快（15ms）
# SSL handshake: 3.210s  ← TLS 握手花了 3.2 秒！
# Total: 3.450s

# 根因分析：
openssl s_client -connect example.com:443 -showcerts | grep "CN ="
# → 证书链 4 层（比普通 3 层多 1 层）→ 客户端需要验证 4 张证书
# → 而且没有开启 OCSP Stapling → 客户端还要额外查询 OCSP 服务器
# → 在跨国网络下，每次 OCSP 查询 + 额外证书验证 = 多了 2-3 秒

# 优化：
# ① 缩短证书链到 3 层（Root → Intermediate → Server）
# ② 开启 OCSP Stapling（服务器主动缓存 OCSP 响应，省掉客户端查询）
# ③ 开启 TLS Session Resumption（重用之前的加密协商，省掉一次握手）
```

## 场景 4：TCP 重传风暴——丢包率 10% 的"虚假网络"

```bash
# 症状：数据库查询时而快（5ms）时而慢（3000ms），所有监控都正常

# 排查步骤：
tcpdump -i eth0 host 10.0.1.100 and port 3306 -w mysql.pcap
# 用 Wireshark 打开 → Statistics → IO Graph → 观察 TCP Errors 曲线

# Wireshark 发现：
# tcp.analysis.retransmission: 每 100 个包里 8-10 个重传！
# tcp.analysis.duplicate_ack: 频发重复 ACK

# 根因：
# 不是"网络断了所以丢包"，而是"网络质量差导致 10% 丢包"
# → TCP 等待 RTO 重传（200ms-1s）
# → 每次重传都在等这么长时间
# → 所以偶尔一次查询要等 3 秒（3-4 次重传 × 200ms RTO 累积）

# 原因定位：
# ① 网线/光纤问题（误码率高）→ 检查交换机接口错误计数
# ② 网卡硬件故障 → dmesg 看网卡驱动错误
# ③ 中间路由器的 QoS 在限速/丢弃 > 某个阈值的包

# 止血：
# ① 检查物理层（换网线/换网卡/换交换机端口）
# ② 如果是跨地域链路 → 切换到 BBR 拥塞控制（BBR 在丢包率高时表现比 CUBIC 好得多）
```

---

# 四、排查工具速查

| 工具 | 一句话 | 核心参数 |
|------|--------|---------|
| **ping** | 网络通不通 | `-c 10`（发 10 次） |
| **traceroute** | 走哪条路 | `-n`（不解析域名）/ `-I`（ICMP 模式） |
| **dig** | DNS 解析 | `+trace`（追踪全链路） |
| **curl** | HTTP 通不通 | `-v`（详细）/ `-w`（自定义输出） |
| **tcpdump** | 包长什么样 | `host/port/and/or/write` |
| **Wireshark** | 图形化分析 | `tcp.analysis` 过滤 + Statistics |
| **ss** | 连接状态 | `-tanp`（所有 TCP + 进程） |
| **mtr** | ping+traceroute 结合 | 持续监测每一跳的丢包率 |

---

# 五、总结

| 故障 | 关键工具 | 止血 |
|------|---------|------|
| **TIME_WAIT 过多** | `ss -tan` | keepalive + tcp_tw_reuse |
| **CLOSE_WAIT 泄漏** | `ss -tanp` | 应用 close() |
| **HTTPS 握手慢** | `curl -w` | 缩短证书链 + OCSP Stapling |
| **TCP 重传风暴** | `tcpdump + Wireshark` | 换线/切换 BBR |

# 延伸阅读

**Do——动手模拟：**
- 用 `iptables -A OUTPUT -p tcp --dport 8080 -j DROP` 模拟网络不通，走一遍四步排查法
- 抓一次 `curl http://localhost:8080` 的完整 TCP 流（tcpdump 保存 pcap → Wireshark 打开 → Follow TCP Stream）
- 用 `tc qdisc` 模拟 10% 丢包，观察 TCP 重传行为（Wireshark → tcp.analysis.retransmission）

**Todo——深入方向：**
- Wireshark 的 TCP Expert Infos——自动标记重传/乱序/窗口为零等异常
- eBPF (BCC/bpftrace) 网络排查——在内核态 track TCP 重传/connect 失败，比 tcpdump 更精确
- 分布式追踪（Jaeger/Zipkin）与网络排查的配合——从应用层 trace 下钻到 TCP 层

*本文参考资料：*
- Brendan Gregg, "Linux Performance Tools" (2015)
- Wireshark User Guide: TCP Analysis
- tcpdump(1) Manual
