---
title: "HTTP 演进史——从 HTTP/0.9 到 HTTP/3 QUIC"
date: 2026-07-29
description: 从 HTTP/0.9 的单行请求、HTTP/1.1 的 keep-alive 与管线化、HTTP/2 的多路复用与 HPACK 头部压缩、到 HTTP/3 QUIC 的 0-RTT 与无队头阻塞，拆解 HTTP 协议三十年演进中每一次"为什么不兼容旧版本"的设计决策与技术权衡。
tags: ["网络","HTTP","HTTP/2","HTTP/3","QUIC","协议"]
categories: ["网络"]
---

# 历史背景——Tim Berners-Lee 顺手写了个协议

1991 年，CERN 的 Tim Berners-Lee 需要一个让研究者们能分享超文本文档的协议。他设计了三个东西：HTML（内容格式）、URL（定位符）、HTTP（传输协议）。最早的 HTTP 只有一种方法——`GET`——整个协议一句话就能描述："客户端发 `GET /page.html`，服务器返回 HTML 文档。

30 年后，HTTP 已经从"一句话协议"变成了支撑全球互联网的核心传输层。这 30 年的演进路径可以概括为：**每次升级都是为了解决上一代协议在高并发环境下的性能瓶颈**。

```
HTTP/0.9 (1991) → 只有 GET，只返回 HTML
HTTP/1.0 (1996) → 加了 POST/HEAD，Content-Type，状态码
HTTP/1.1 (1997) → keep-alive，pipeline，Host 头（虚拟主机）
HTTP/2   (2015) → 多路复用，头部压缩，Server Push
HTTP/3   (2022) → QUIC 替代 TCP，0-RTT，无 TCP 队头阻塞
```

---

# 一、HTTP/1.0 → HTTP/1.1——"一个连接反复用"

## 1.1 HTTP/1.0 的痛点：每个请求都要建新连接

```http
GET /index.html HTTP/1.0    ← Client → Server: TCP 三次握手
200 OK                       ← Server 返回 HTML → TCP 四次挥手 ◀ 连接关闭！
GET /style.css HTTP/1.0     ← 又一次 TCP 三次握手...
200 OK                       ← 又一次四次挥手 ◀

一个网页 + 5 个资源文件 = 6 次 TCP 握手 + 6 次挥手的 RTT 开销
```

## 1.2 HTTP/1.1 keep-alive——连接复用

```http
GET /index.html HTTP/1.1
Connection: keep-alive
200 OK                       ← 连接保持！

GET /style.css HTTP/1.1      ← 复用同一条 TCP 连接
200 OK

GET /script.js HTTP/1.1      ← 还是同一条
200 OK
```

**keep-alive 是 HTTP/1.1 最重要的改进**，省掉了 6 次 TCP 连接的高昂建立开销。

## 1.3 Pipeline——理论加速，现实失败

```
非管线化：
  Client → GET /a → Server → 200 OK → Client → GET /b → Server → 200 OK

管线化（理想）：
  Client → GET /a → GET /b → GET /c → Server → 200 → 200 → 200
  省掉了请求之间的 RTT 等待

现实：存在队头阻塞（HOL Blocking）
  GET /a 卡住了（后端处理慢）→ GET /b 和 GET /c 的响应也被堵在后面
  → 浏览器几乎全禁用了 pipeline
```

## 1.4 HTTP/1.1 的其他关键特性

```
Host 头：同一 IP 部署多个网站
  GET / HTTP/1.1
  Host: www.example.com  ← 服务器根据 Host 区分不同站点

Chunked Transfer：不预知总长度的流式传输
  Transfer-Encoding: chunked
  7\r\nMozilla\r\n     ← 一段一段的发，最后 0\r\n\r\n 结束
  9\r\nDeveloper\r\n
  0\r\n\r\n
```

---

# 二、HTTP/2——多路复用 + 二进制帧

## 2.1 HTTP/1.1 的三个性能天花板

```
问题 1：同一连接的请求串行（即使开了多个连接也就 6-8 个）
问题 2：Header 冗余——每个请求都带相同的 Cookie/User-Agent/... 
问题 3：请求/响应只能由客户端发起（Server 不能主动 Push）
```

## 2.2 多路复用——同一连接上并行传输

```
HTTP/2 在一个 TCP 连接上建立多个 Stream（流），每个 Stream 独立收发：

  TCP 连接
  ┌─── Stream 1: GET /index.html  ──┐
  ├─── Stream 3: GET /style.css   ──┤  ← 并行发送，并行接收！
  ├─── Stream 5: GET /script.js   ──┤    不用等前一个完成
  └─── Stream 7: GET /image.png   ──┘

  Client 和 Server 通过 Frame 在同一个 TCP 连接上交错传输，
  每个 Frame 标记自己属于哪个 Stream，接收端按 Stream ID 组装。
```

**HTTP/2 用"二进制帧"替代了 HTTP/1.1 的"纯文本协议"**：

```
HTTP/1.1（文本）：
  GET /index.html HTTP/1.1\r\nHost: example.com\r\n\r\n
  → 字符串解析，空格和换行有歧义

HTTP/2（二进制帧）：
  Frame Header (9 字节) + Frame Payload
  ┌─────────┬───────────┬────────────┐
  │ Length  │  Type     │  Stream ID │
  │(24 bit) │ (8 bit)   │  (31 bit)  │
  ├─────────┴───────────┴────────────┤
  │         Frame Payload            │
  └──────────────────────────────────┘
  → 固定格式：TYPE 区分 HEADERS/DATA/PRIORITY/RST_STREAM/SETTINGS/PING/GOAWAY...
```

## 2.3 HPACK——头部压缩

```
HTTP/1.1 每次请求都带：
  Cookie: session_id=abc123def456... (可能 500+ 字节)
  User-Agent: Mozilla/5.0 ... (200+ 字节)

HTTP/2 HPACK：
  ① 两端维护相同的"头部索引表"（静态表 61 个常见头 + 动态表 用户自定义头）
  ② 第一次发 Cookie → 存入动态表（索引 62）
  ③ 第二次发 Cookie → 只发 "索引 62"（2 字节替代 500 字节）
```

## 2.4 Server Push——服务器主动推送

```
传统 HTTP/1.1：
  Client 请求 index.html → Server 返回
  Client 解析 HTML → 发现需要 style.css → Client 额外请求 → Server 返回

HTTP/2 Server Push：
  Client 请求 index.html
  → Server 返回 index.html + 主动推送 style.css（PUSH_PROMISE 帧）
  → Client 在还没解析 HTML 之前就已经拿到了 style.css
  → 省掉一次 RTT
```

> **但 Server Push 在 Chrome 106 中被移除**——因为它浪费带宽（可能推送了客户端已缓存的内容）。实践中，`<link rel="preload">` 是更好的选择。

## 2.5 HTTP/2 的问题：TCP 层的队头阻塞

HTTP/2 解决了 HTTP 层的队头阻塞（多路复用），但 TCP 层的队头阻塞仍然存在：

```
TCP 连接上 Stream 1 丢了一个包 → TCP 必须等重传 → 整个 TCP 窗口停止滑动
→ Stream 2,3,4,5... 全部被卡住（即使它们的包没丢）
→ HTTP/2 的"多路复用"并发优势在丢包时几乎全废
```

这个问题的根本原因是：**TCP 是字节流，它不理解 HTTP frame 的边界**。对 TCP 来说，Stream 之间只是连续的字节，丢一个字节所有 Stream 都等。HTTP/3 的 QUIC 解决了这个问题。

---

# 三、HTTP/3——QUIC 替代 TCP

HTTP/3 的核心变化：**底层从 TCP 切换到 QUIC（UDP）**。HTTP 的语义（GET/POST/状态码/Header）保持不变，但传输层彻底变样。

```
HTTP/1.1: HTTP 语义 → TLS (可选) → TCP → IP
HTTP/2:   HTTP 语义 → TLS → TCP → IP
HTTP/3:   HTTP 语义 → QUIC (强制 TLS 1.3) → UDP → IP
                              ↑
                    内置加密 + 内置多路复用
```

对比表格：

| | HTTP/1.1 | HTTP/2 | HTTP/3 |
|------|---------|--------|--------|
| **传输** | TCP | TCP | QUIC (UDP) |
| **多路复用** | ❌ | ✅（但有 TCP 队头阻塞）| ✅（天然无阻塞） |
| **头部压缩** | ❌ | HPACK | QPACK |
| **加密** | 可选 TLS | 可选 TLS | 强制 TLS 1.3 |
| **握手延迟** | 1-RTT TCP + 1-RTT TLS | 1-RTT TCP + 1-RTT TLS | 1-RTT（首次）/0-RTT（重连） |
| **连接迁移** | ❌ | ❌ | ✅ |

---

# 四、总结

| 版本 | 核心改进 | 为什么需要 |
|------|---------|-----------|
| **0.9** | 单行 GET | 分享 HTML 文档 |
| **1.0** | POST/状态码/Content-Type | 不只是 GET 了 |
| **1.1** | keep-alive + Host + chunked | 省掉重复的 TCP 握手 |
| **2** | 多路复用 + 二进制帧 + HPACK | 省掉 HTTP 层的串行 |
| **3** | QUIC + 0-RTT + 无 TCP 队头阻塞 | 根治 TCP 队头阻塞 |

> **HTTP 30 年的演进方向：省 RTT。** 因为每一次 RTT 的减少都直接等于用户感知到的页面加载速度的提升。

# 延伸阅读

**Do——动手验证：**
- 用 Chrome DevTools Network 面板右键勾选 "Protocol" 列，看当前网页的请求用的是 h2 还是 h3
- `curl --http2 -v https://nghttp2.org` 观察 HTTP/2 的 ALPN 协商和二进制帧
- `docker run -p 443:443 nginx:1.25` 配合 `http3 on;` 测试 HTTP/3 连接

**Todo——深入方向：**
- HTTP/2 的流优先级——Stream Dependency 与 Weight 对资源加载顺序的影响
- QPACK 如何解决 HPACK 的队头阻塞——同一连接上多个 Stream 需要各自独立的头部压缩状态
- gRPC 如何利用 HTTP/2 的 Stream 实现双向流式调用

*本文参考资料：*
- RFC 1945: HTTP/1.0 / RFC 2616: HTTP/1.1
- RFC 7540: HTTP/2
- RFC 9114: HTTP/3
- Google Chrome: Intent to Remove HTTP/2 Server Push (2022)
