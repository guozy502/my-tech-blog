---
title: "DNS 与 CDN——从域名解析到就近分发"
date: 2026-07-29
description: 从 DNS 的递归/迭代查询机制、A/CNAME/NS 记录类型与智能解析（GeoDNS）、CDN 的 DNS 调度→边缘节点→回源的完整工作链路，到 DNS 污染与 HTTPDNS 绕过劫持，拆解互联网"域名→IP→就近访问"的数据传输起点。
tags: ["网络","DNS","CDN","域名解析","GeoDNS","HTTPDNS"]
categories: ["网络"]
---

# 历史背景——HOSTS.TXT 撑不住的时候

1970 年代，ARPANET 上只有几十台主机。每台主机的名称和 IP 地址记录在一个叫 `HOSTS.TXT` 的文件里，所有人手动下载这个文件来同步。到了 1983 年，主机数突破 500 台——没人能每天手动更新这个文件了。Paul Mockapetris 设计了 DNS（Domain Name System）——一个分布式、层次化的命名系统。

1998 年 Akamai 成立时，DNS 有了新用途——不只是"域名→IP"，还可以"根据请求来源返回离用户最近的服务器 IP"。这就是 CDN 的核心——DNS 调度。

---

# 一、DNS 解析——递归 vs 迭代

## 1.1 递归查询——"你帮我一查到底"

```bash
dig +trace www.example.com

# 解析链路：
# ① 浏览器 → 本地 DNS 解析器（/etc/resolv.conf）
# ② 本地 DNS → 根 DNS（.）
# ③ 根 DNS → "我不知道，你去问 .com 的 NS"
# ④ 本地 DNS → .com TLD DNS
# ⑤ .com DNS → "我不知道，你去问 example.com 的 NS"
# ⑥ 本地 DNS → example.com 的权威 DNS
# ⑦ 权威 DNS → "www.example.com 的 A 记录是 93.184.216.34"
```

```
层次结构：
  . (Root DNS, 13 个根服务器集群)
  └── .com / .org / .cn (Top-Level Domain, TLD)
        └── example.com (Authoritative DNS)
              └── www.example.com → 93.184.216.34 (A 记录)
```

## 1.2 迭代查询 vs 递归查询

```
递归查询："你帮我问，你告诉我结果"
  Client → Local DNS（Local DNS 替 Client 一层一层问）

迭代查询："你问谁，我去找他"
  Client → Root DNS → "去问 .com" → Client → .com DNS → "去问 example.com" → ...
```

实际中浏览器对 Local DNS 发递归查询，Local DNS 对上级 DNS 发迭代查询。

## 1.3 记录类型速查

| 类型 | 含义 | 示例 |
|------|------|------|
| **A** | IPv4 地址 | `www.example.com. A 93.184.216.34` |
| **AAAA** | IPv6 地址 | `www.example.com. AAAA 2606:2800:220:1::` |
| **CNAME** | 别名（指向另一个域名） | `www.example.com. CNAME example.com.` |
| **NS** | 权威 DNS 服务器 | `example.com. NS ns1.example.com.` |
| **MX** | 邮件服务器 | `example.com. MX 10 mail.example.com.` |
| **TXT** | 任意文本（SPF/DKIM 验证） | `example.com. TXT "v=spf1 include:_spf.google.com ~all"` |
| **SRV** | 指定服务的端口和主机 | `_sip._tcp.example.com. SRV 10 5 5060 sipserver.example.com.` |

---

# 二、CDN——DNS 调度的魔法

## 2.1 CDN 的工作流程

```
① 用户在浏览器输入 www.example.com
② Local DNS 向 example.com 的权威 DNS 查询
③ 权威 DNS 已把 www.example.com CNAME 到 example.cdn.com
④ Local DNS 继续向 cdn.com 的 DNS 查询
⑤ cdn.com 的 DNS 看到请求来自北京电信 IP
   → 返回北京电信边缘节点的 IP（最近的！）
⑥ 用户请求导向北京电信 CDN 节点
⑦ 节点有缓存 → 直接返回（命中）
   节点没有缓存 → 回源站（example.com）→ 缓存 + 返回
```

## 2.2 CDN 的三个核心技术

```
① DNS 调度（GeoDNS）：根据 Local DNS 的 IP 判断地理位置 → 返回最近节点的 IP
   精度 ≈ 城市级（因为 Local DNS 可能不在用户同一城市）

② 缓存策略：边缘节点缓存静态资源（图片/CSS/JS）和可缓存的动态内容（API JSON）
   命中率通常 90-98%

③ 回源控制：CDN 节点到源站的连接数控制 + 合并回源（多个边缘节点请求同一资源时
   只允许第一个请求回源，其余等待）
```

---

# 三、DNS 劫持与 HTTPDNS——绕过运营商篡改

## 3.1 DNS 劫持/UDP 劫持

```
现象：你在浏览器输入 www.example.com
      → Local DNS 返回的不是正确 IP，而是一个广告页/钓鱼页的 IP

原因：
  ① 运营商层面的 DNS 劫持：UDP 53 端口无加密无校验 → 运营商中间人篡改 DNS 响应
  ② 路由器/Local DNS 被篡改：改了 /etc/hosts 或 DNS 服务器地址
```

## 3.2 HTTPDNS——绕过 UDP，走 HTTPS 做 DNS

```
传统 DNS：UDP 53 → 明文 → 容易被劫持

HTTPDNS：
  客户端 → HTTPS → httpdns.example.com/resolve?domain=www.baidu.com
  服务器 → JSON: {"ip": "110.242.68.4", "ttl": 60}

优点：
  - HTTPS 防劫持（加密 + 服务器身份验证）
  - 服务器可以直接看到客户端真实 IP（不是 Local DNS IP）
  - 精确到用户级别的调度（可以实现灰度、A/B、精准降级）

代价：
  - 比 UDP 慢（多一次 HTTPS 握手）
  - 需要客户端嵌入 HTTPDNS SDK
```

---

# 四、DNS 常见问题

```bash
# 1. DNS 缓存导致的"改了域名解析但没生效"
dig www.example.com @8.8.8.8  # 指定用 Google DNS
dig www.example.com +noall +answer  # 只显示答案部分

# 2. CNAME 与 A 记录不能共存
# www.example.com 要么是 A（直接指向 IP），要么是 CNAME（指向另一个域名）
# 不能同时设置 A 和 CNAME

# 3. CNAME 展平（Flattening）——Cloudflare 的解决方案
# 根域名 example.com 理论上不能用 CNAME
# Cloudflare 在权威 DNS 层面自动把 CNAME 展开为 A 记录

# 4. TTL —— 权衡"变更速度"和"解析负担"
# TTL=60s → 变更生效快（最多 60s 后全网更新）
# TTL=3600s → Local DNS 缓存 1 小时 → 变更慢但 DNS 查询量小
```

---

# 五、总结

| 概念 | 作用 | 一句话 |
|------|------|--------|
| **DNS 递归** | 域名→IP 解析 | "你帮我查，你告诉我答案" |
| **CNAME** | 域名别名 | "别问我，去问他" |
| **GeoDNS** | 地理定位解析 | "你在北京？去北京服务器" |
| **HTTPDNS** | 防劫持 DNS | "走 HTTPS，不走 UDP 53" |
| **CDN** | 就近缓存分发 | "在你家门口放个备份" |

# 延伸阅读

**Do——动手验证：**
- `dig +trace www.example.com` 从根开始追踪 DNS 解析的全路径
- `dig www.example.com @1.1.1.1` vs `@8.8.8.8` 对比不同 Public DNS 的解析结果
- 用 `curl -v http://example.com -H "Host: www.example.com"` 模拟 CDN 的 HTTP 调度

**Todo——深入方向：**
- Anycast DNS——同一个 IP 在多个地方同时广播，天然就近
- DNSSEC——给 DNS 响应加数字签名，防篡改（正在推广中）
- EDNS Client Subnet——DNS 服务器传递用户子网信息给权威 DNS，解决 CDN 调度精度

*本文参考资料：*
- RFC 1034/1035: Domain Names
- Cloudflare: What is Anycast? / How CDN Works?
- 腾讯云 HTTPDNS 文档
