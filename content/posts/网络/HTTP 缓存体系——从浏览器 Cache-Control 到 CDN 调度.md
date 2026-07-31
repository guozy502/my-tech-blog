---
title: "HTTP 缓存体系——从浏览器 Cache-Control 到 CDN 调度"
date: 2026-07-29
description: "从浏览器强缓存（Cache-Control: max-age）+ 协商缓存（ETag/Last-Modified）的两层缓存策略、缓存决策树、CDN 的分层缓存架构（边缘→区域→源站），到缓存击穿/穿透/雪崩的根因与解决方案，拆解 HTTP 缓存在整个网络数据链路中的完整角色。"
tags: ["网络","HTTP","缓存","Cache-Control","CDN","ETag"]
categories: ["网络"]
---

# 历史背景——缓存是 HTTP 中唯一被低估的优化

HTTP 的许多进化都被广泛讨论——HTTP/2 的多路复用、HTTP/3 的 0-RTT。但缓存的改进相对安静：从 HTTP/1.0 的 `Expires` 到 HTTP/1.1 的 `Cache-Control` 和 `ETag`，再到 CDN 的多层缓存架构——缓存是 HTTP 中唯一一个"不看它也不会出错，但管好它能让性能翻 10 倍"的机制。

**缓存的省钱逻辑很简单**：每一次"文件从源站传输到客户端"都有带宽成本和延迟成本。如果在浏览器/网关/CDN 各层缓存了相同的内容，后续请求就不需要再走整条链路。理解 HTTP 缓存，就是理解**如何在整个网络链路中层层减少传输量**。

---

# 一、强缓存——"别问了，直接用，还没过期"

## 1.1 Cache-Control: max-age (HTTP/1.1)

```http
# 响应头
Cache-Control: max-age=3600  ← 缓存 1 小时

# 效果：
# 第一次请求 → 下载 → 缓存到本地 + 记录过期时间
# 第二次请求（1 小时内）→ 直接用缓存，不问服务器
# 第二次请求（1 小时后）→ 缓存过期，重新问服务器
```

## 1.2 Cache-Control 指令速查

```http
Cache-Control: public           ← 可以被任何中间节点（CDN/代理）缓存
Cache-Control: private          ← 只能被浏览器缓存（不能 CDN 缓存）
Cache-Control: no-cache         ← 可以缓存，但每次使用前必须验证（走协商缓存）
Cache-Control: no-store         ← 完全不缓存（每次都拉新的）
Cache-Control: max-age=3600     ← 缓存 3600 秒
Cache-Control: s-maxage=600     ← CDN 缓存 600 秒（比浏览器自己的 max-age 短）
Cache-Control: must-revalidate  ← 过期后必须重新验证，不能用过期缓存
```

## 1.3 Expires (HTTP/1.0, 已淘汰)

```http
Expires: Wed, 21 Oct 2026 07:28:00 GMT  ← 过期绝对时间
```
用 `max-age` 替代 `Expires`——因为 `Expires` 依赖客户端和服务器时间同步，而 `max-age` 是相对时间，不依赖时钟。

---

# 二、协商缓存——"我有缓存，但它过期了，服务器你看看还能不能用？"

当强缓存过期时，浏览器不直接重新下载，而是**问服务器"过期归过期，内容变了吗？没变的话我用旧的就行"。**

## 2.1 ETag / If-None-Match——内容指纹

```http
# 第一次请求
GET /style.css HTTP/1.1
HTTP/1.1 200 OK
ETag: "abc123def456"           ← 服务器返回内容指纹（通常是内容的哈希）
Cache-Control: max-age=0       ← 立即过期，每次都要验证

# 第二次请求（带协商）
GET /style.css HTTP/1.1
If-None-Match: "abc123def456"  ← 告诉服务器"我有这个版本的缓存"
HTTP/1.1 304 Not Modified      ← 服务器回复："没改，用你的缓存"
                                ← 响应体为空，省下了整个文件的传输！
```

## 2.2 Last-Modified / If-Modified-Since——时间戳

```http
# 第一次请求
HTTP/1.1 200 OK
Last-Modified: Mon, 27 Jul 2026 10:00:00 GMT

# 第二次请求
GET /style.css HTTP/1.1
If-Modified-Since: Mon, 27 Jul 2026 10:00:00 GMT
HTTP/1.1 304 Not Modified           ← 没改过
```

## 2.3 ETag vs Last-Modified

| | ETag | Last-Modified |
|------|------|-------------|
| **精度** | 内容级别的哈希 | 秒级时间戳 |
| **秒内多次修改** | ✅ 能检测 | ❌ 检测不到（时间戳只到秒） |
| **内容不变但 mtime 变** | ✅ 不受影响 | ❌ 误判为修改 |
| **计算代价** | 需要算哈希 | 几乎零代价（读文件元数据） |

**两者都设置时，ETag 优先级更高。**

---

# 三、缓存决策树——浏览器问自己："我该怎么处理这个请求？"

```
浏览器发起请求前的决策：

① 有缓存且 Cache-Control: no-store?
   是 → 直接发请求取最新

② 有缓存且 Cache-Control: no-cache?
   是 → 直接走协商缓存（带 If-None-Match 去问服务器）

③ 有缓存且 max-age 还没过期?
   是 → 直接用缓存（强缓存命中）

④ 有缓存但 max-age 过期了?
   是 → 带 If-None-Match / If-Modified-Since 去问
      → 服务器返回 304 → 用缓存
      → 服务器返回 200 → 下载新内容

⑤ 没有缓存?
   是 → 直接下载
```

---

# 四、CDN 缓存——在离用户最近的地方缓存

## 4.1 CDN 的层次缓存架构

```
用户 (北京)
  │
  ├──→ CDN 边缘节点 (北京) ← 最热内容缓存在这里（命中率 ~90%）
  │       │
  │       └──→ CDN 区域节点 (华北) ← 较热内容在这里
  │               │
  │               └──→ 源站 (杭州) ← 全量内容，访问量少
  │
  ├──→ CDN 边缘节点 (上海) ...
  └──→ CDN 边缘节点 (广州) ...
```

## 4.2 CDN 缓存控制

```http
# 利用 HTTP 头控制 CDN 缓存
Cache-Control: public, max-age=3600, s-maxage=1800
# s-maxage=1800 → CDN 缓存 30 分钟
# max-age=3600 → 浏览器缓存 1 小时
# → CDN 更频繁地回源检查（热点内容及时更新）

# CDN 缓存 Key：通常 = 域名 + URI + 部分头部
# 例：cdn-key = "example.com" + "/images/logo.png" + "Accept-Encoding"
```

## 4.3 CDN 刷新与预热

```bash
# 刷新（Purge）：主动清除 CDN 上的缓存
# → 下次请求时 CDN 回源重新取
curl -X PURGE https://cdn.example.com/images/logo.png

# 预热（Prefetch）：提前把内容加载到 CDN
# → 在大促前把热门商品的图片提前推到边缘节点
```

---

# 五、缓存击穿/穿透/雪崩——三个面试最爱问的灾难

| | 现象 | 根因 | 解法 |
|------|------|------|------|
| **击穿** | 单个热点 key 过期，大量请求瞬间穿透到源站 | key 同时过期 + 高并发 | 互斥锁（同时只有 1 个请求回源）+ 永远不过期（后台异步刷新） |
| **穿透** | 查询不存在的数据，缓存永远不命中 | 恶意请求/业务逻辑 Bug | 缓存空值 + 布隆过滤器预筛 |
| **雪崩** | 大量 key 同时过期，流量全打到源站 | 批量 key 的过期时间相同 | 过期时间加随机偏移 + 多级缓存 |

---

# 六、总结

| 层级 | 机制 | 解决的问题 |
|------|------|----------|
| **浏览器** | 强缓存 (max-age) | 相同资源不重复下载 |
| **浏览器** | 协商缓存 (ETag/304) | 过期了但内容没变就不重新下载 |
| **CDN 边缘** | 离用户最近 | 减少 RTT + 分担源站压力 |
| **Nginx** | proxy_cache | 反向代理层拦截重复回源 |
| **应用** | Redis/Memcached | 数据库保护 |

# 延伸阅读

**Do——动手验证：**
- Chrome DevTools Network 面板，观察请求的 `Size` 列：`disk cache`（强缓存命中）vs `memory cache` vs 真实大小
- 用 `curl -I https://example.com/image.png` 观察响应头中的 `Cache-Control` / `ETag` / `Last-Modified`
- 用 `curl -H "If-None-Match: \"abc123\"" -I https://example.com/resource` 模拟协商缓存

**Todo——深入方向：**
- Service Worker 的 Cache API——在浏览器端用 JS 完全控制缓存策略
- 缓存的 Key 设计——`Vary` 头、`CacheKey` 粒度对命中率的影响
- `stale-while-revalidate`——后台异步刷新的"永远新鲜"缓存模式

*本文参考资料：*
- RFC 7234: HTTP/1.1 Caching
- Google Web Fundamentals: HTTP Caching
- Fastly / Cloudflare CDN 文档: Cache-Control
