---
title: "UDP 与 QUIC——不可靠协议的可靠用法"
date: 2026-07-29
description: 从 UDP 的"不保证到达/不保证顺序/不保证不重复"三大特性出发，拆解 DNS 的无连接查表、音视频的"丢帧好过卡顿"设计取舍、QUIC 如何在应用层基于 UDP 实现 0-RTT 握手的可靠传输，以及 HTTP/3 如何彻底消除 TCP 队头阻塞。
tags: ["网络","UDP","QUIC","HTTP/3","DNS","音视频"]
categories: ["网络"]
---

# 历史背景——UDP 为什么"活下来了"？

1980 年，David P. Reed 设计了 UDP。和 TCP 同年诞生，但 UDP 几乎没有获得什么关注——人们认为"不可靠的协议"在网络中没什么用。TCP 独占了几乎所有互联网应用：HTTP、FTP、SMTP、SSH……

但这个局面在 2010 年代被三个趋势打破：
1. **实时通信（音视频）**——丢几帧画面没关系，但卡顿 200ms 用户就能感知
2. **物联网/MQTT**——传感器数据量极小，TCP 的三次握手开销占比太大
3. **Google QUIC（2013）**——在 UDP 上实现了完整的可靠传输 + 加密 + 多路复用，直接催生了 HTTP/3

UDP 的"什么都不保证"从一个劣势变成了优势——**因为什么都不保证，所以应用层可以按需定制**。QUIC 正是利用了这个空白，在应用层实现了比 TCP 更激进的优化。

---

# 一、UDP 的三不保证——具体是什么？

## 1.1 不保证到达

```
发送方 UDP socket.send("hello")
  → 网络层打包 → 路由器 → ... 可能丢了 → 接收方没收到
  → 没有任何 ACK 机制，发送方永远不知道丢了
```

## 1.2 不保证顺序

```
发送方发送: A, B, C
接收方可能收到: C, A, B（乱序）
  → TCP 有序列号保证顺序，UDP 没有
```

## 1.3 不保证不重复

```
发送方发送: A
网络中出现了一个重复的 A（路由器缓存/重复分组）
接收方收到: A, A（重复）
  → TCP 有序列号去重，UDP 没有
```

## 1.4 UDP 头部——8 字节的极简设计

```
 0      16      31
┌───────┬───────┬───────┐
│源端口  │目的端口│ 长度   │
├───────┴───────┼───────┤
│     校验和     │ 数据   │
└───────────────┴───────┘

只有 8 字节！对比 TCP 最小头部 20 字节。
这就是 UDP 的哲学："我不会给你保证，所以也不收你什么管理开销。"
```

UDP 的伪头部（计算校验和时临时拼接）包含 IP 的源/目的地址——这保证了校验和不是"只校验端口正确"，而是连"IP 包没有被错误路由"也一并检测了。

---

# 二、DNS——UDP 最经典的应用

DNS 查询是 UDP 的完美场景：**一问一答，数据量极小（通常几百字节），偶尔丢包重试就行，不需要连接。**

```
DNS 解析 www.example.com：
  ① 客户端用 UDP 发送 DNS Query 到 8.8.8.8:53（几十字节）
  ② Google DNS 返回 A 记录（93.184.216.34，几十字节）
  ③ 总共 2 个 UDP 包，没有连接建立，没有挥手

如果丢包？→ DNS 客户端超时重试（通常 2-5 秒超时，重试 2 次）
```

**DNS 什么时候用 TCP？**
- 响应超过 512 字节（UDP 的 DNS 限制）→ 切换到 TCP
- 区域传输（Zone Transfer, AXFR/IXFR）——数据传输量大，需要 TCP

---

# 三、QUIC——在 UDP 上重建 TCP（但更好）

## 3.1 为什么 Google 要自己造 QUIC？

2013 年，Google 的 Chrome 团队面临一个困境：**TCP 在 Linux 内核中，想改进太慢**。一个新内核版本需要 2-3 年才能普及到大部分终端用户。但 Google 想优化的东西很多：

- TCP 握手 1-RTT + TLS 握手 1-RTT = 建立 HTTPS 连接需要 2-RTT
- TCP 队头阻塞——一个 stream 丢包会影响所有 stream（HTTP/2 的痛点）
- TCP 在丢包率高（4G/5G/WiFi 切换）的移动网络下表现不好

QUIC 的答案：**把传输层搬到用户态（UDP），用应用层代码实现可靠传输 + 加密 + 多路复用**。

## 3.2 QUIC 的 0-RTT 握手

```
传统 HTTPS (TCP + TLS 1.2)：
  ① TCP 三次握手 (1-RTT)
  ② TLS 1.2 ClientHello + ServerHello + 密钥交换 (1-RTT)
  ③ HTTP 请求 (2-RTT 后才能发送!)
  总延迟：2-RTT + Server 处理

QUIC：
  首次连接（1-RTT）：
    ① QUIC ClientHello（在 UDP 包中携带 TLS 1.3 密钥交换参数 + 自己的公钥）
    ② Server 回复 ServerHello（已完成密钥交换 + 可以立刻发送 HTTP 响应）
    总延迟：1-RTT（比 HTTPS 快 1-RTT）
  
  重连（0-RTT）：
    如果客户端之前和这个 Server 有过连接，QUIC 可以用之前协商的密钥
    → 第一个 UDP 包就携带应用数据（HTTP 请求）！
    总延迟：0-RTT
```

## 3.3 QUIC 解决 TCP 队头阻塞

**TCP 的队头阻塞**（HTTP/2 最大的痛点）：

```
TCP 连接
  ┌─── Stream 1 (CSS)  ──┐
  ├─── Stream 2 (HTML) ──┤  ← 如果 Stream 2 的一个 TCP 包丢了
  ├─── Stream 3 (JS)   ──┤  → TCP 必须等重传完成才能继续
  └─── Stream 4 (图片) ──┘  → 所有 Stream 都被阻塞！
```

**QUIC 的解法**：

```
QUIC 连接
  ┌─── Stream 1 (CSS)  ──┐  ← 独立 ACK，独立重传
  ├─── Stream 2 (HTML) ──┤  ← 丢了？只影响 Stream 2
  ├─── Stream 3 (JS)   ──┤  ← 不受影响，正常到达
  └─── Stream 4 (图片) ──┘  ← 不受影响，正常到达
```

QUIC 在同一个 UDP 连接中通过 **Stream ID** 区分不同的流。每个流独立 ACK，独立重传。一个流丢包只影响它自己，不影响其他流。

## 3.4 QUIC 的其他优化

**连接迁移（Connection Migration）**：
```
传统 TCP：连接由 (src_ip, src_port, dst_ip, dst_port) 唯一确定
  手机从 WiFi 切到 4G → src_ip 变了 → TCP 连接断开 → 必须重新握手

QUIC：连接由 Connection ID 唯一确定（一个 64 位随机数）
  WiFi 切到 4G → src_ip 变了 → 但 Connection ID 不变 → 连接无缝迁移！
```

**前向纠错（FEC, Forward Error Correction）**：
```
TCP：丢一个包 → 必须重传（增加延迟）

QUIC（可选）：发送 N 个数据包 + 1 个 FEC 冗余包
  任何一个包丢了 → 可以用其余包恢复 → 不需要重传！
  代价：20% 的带宽冗余
```

## 3.5 QUIC 的现状

- **Google 内部**：YouTube、Search、Gmail 的 50%+ 流量走 QUIC
- **IETF 标准化**：RFC 9000（2021 年）→ HTTP/3 = HTTP over QUIC
- **浏览器支持**：Chrome/Firefox/Edge 默认启用 QUIC + HTTP/3
- **服务端**：Nginx（ngx_http_v3_module，1.25+）、Caddy、LiteSpeed 原生支持

---

# 四、QUIC vs TCP 全维度对比

| | TCP + TLS 1.2 | TCP + TLS 1.3 | QUIC |
|------|-------------|-------------|------|
| **握手延迟** | 2-RTT | 1-RTT | 1-RTT（首次）/ 0-RTT（重连） |
| **队头阻塞** | TCP 层阻塞 | TCP 层阻塞 | 无（独立 Stream ACK） |
| **连接迁移** | ❌ | ❌ | ✅ Connection ID |
| **内核依赖** | 内核 | 内核 | 用户态（UDP） |
| **加密** | TLS 可选 | TLS 可选 | **强制加密**（TLS 1.3 内置） |
| **部署** | 所有 OS 原生 | 所有 OS 原生 | 需要应用支持 |

---

# 五、总结

| 协议 | 哲学 | 适用 |
|------|------|------|
| **UDP** | 什么都别管，我最快 | DNS、音视频、游戏、物联网 |
| **TCP** | 可靠第一，慢点没关系 | HTTP/1.1、文件传输、邮件 |
| **QUIC** | 在应用层做 TCP 该做的事，顺便改进它 | 移动互联网、直播、HTTP/3 |

# 延伸阅读

**Do——动手验证：**
- 用 `ncat -u` 发送 UDP 包，用 `tcpdump udp port` 抓包，观察 UDP 是否做三次握手
- 在 Chrome 的 `chrome://net-internals/#quic` 查看 QUIC 连接和 Stream 状态
- 用 `curl --http3 https://www.google.com` 测试 HTTP/3 连接

**Todo——深入方向：**
- QUIC 的拥塞控制——BBR 在 QUIC 上如何工作，和 TCP BBR 有何异同
- HTTP/3 的 QPACK 头部压缩——如何替代 HTTP/2 的 HPACK（解决队头阻塞问题）
- WebTransport（基于 QUIC）——WebSocket 的下一代替代方案

*本文参考资料：*
- RFC 768: User Datagram Protocol
- RFC 9000: QUIC: A UDP-Based Multiplexed and Secure Transport
- RFC 9114: HTTP/3
- Google QUIC Design Document: https://www.chromium.org/quic/
