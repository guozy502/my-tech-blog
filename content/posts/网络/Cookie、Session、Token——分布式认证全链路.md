---
title: "Cookie、Session、Token——分布式认证全链路"
date: 2026-07-29
description: 从 Cookie 的安全属性（HttpOnly/Secure/SameSite）与作用域、Session 的分布式存储一致性方案（粘滞/集中/复制）、JWT 的结构与无状态认证的安全争议、到 OAuth 2.0 授权码模式的完整授权链路，拆解 Web 应用中请求身份从"你是谁"到"你能做什么"的完整流程。
tags: ["网络","Cookie","Session","JWT","OAuth","认证"]
categories: ["网络"]
---

# 历史背景——HTTP 的"无状态"让认证变得必要

HTTP 是无状态协议——服务器处理完一个请求就忘了你是谁。下一个请求来了，服务器只能从零开始。这意味着"你登录了"这件事必须通过某种机制让服务器在后续每个请求中都能识别。

1994 年 Netscape 发明了 Cookie——一个小小的键值对，浏览器自动在每次请求中带上服务器给的 Cookie。之后演化出了 Session（服务器存身份、Cookie 只存 Session ID）、JWT（客户端自包含身份信息）、OAuth 2.0（第三方的"我允许这个 App 代表我访问我的数据"）。理解这一条线的演进，你就能回答"为什么有了 Cookie 还要 JWT，为什么有了 JWT 还要 OAuth"。

---

# 一、Cookie——身份的第一个载体

## 1.1 Set-Cookie 与自动回传

```http
# 服务器在响应中设 Cookie
HTTP/1.1 200 OK
Set-Cookie: session_id=abc123; Path=/; HttpOnly; Secure; SameSite=Lax

# 浏览器后续所有到该域名的请求自动带上
GET /api/profile HTTP/1.1
Cookie: session_id=abc123
```

## 1.2 Cookie 的安全属性

```
┌─────────────┬──────────────────────────────────────┐
│ 属性        │ 含义                                  │
├─────────────┼──────────────────────────────────────┤
│ HttpOnly    │ JS 不能通过 document.cookie 读取        │
│             │ → 防 XSS 攻击窃取 Cookie                │
├─────────────┼──────────────────────────────────────┤
│ Secure      │ 只在 HTTPS 连接上传输                   │
│             │ → 防中间人窃听                          │
├─────────────┼──────────────────────────────────────┤
│ SameSite    │ 控制跨站请求是否带 Cookie                │
│   Strict    │   绝对不带（跨站时完全不带 Cookie）       │
│   Lax       │   只在 GET 导航时带（安全默认）          │
│   None      │   全带（必须配合 Secure）               │
├─────────────┼──────────────────────────────────────┤
│ Domain      │ Cookie 的作用域名范围                    │
│ Path        │ Cookie 的作用路径范围                    │
│ Expires/Max │ Cookie 的过期时间                        │
└─────────────┴──────────────────────────────────────┘
```

**SameSite 是 CSRF 攻击的克星**：
```
CSRF 场景（没有 SameSite 保护时）：
  用户在 bank.com 已登录（Cookie 中有 session_id）
  用户访问 evil.com → evil.com 发一个 <img src="bank.com/transfer?to=hacker">
  → 浏览器自动带上 bank.com 的 Cookie → bank 以为用户本人发起了转账 → 钱被转走

SameSite=Lax：
  跨站请求（<img>/<form> POST）不会带 Cookie
  → CSRF 攻击请求中没有 session_id → 服务器直接拒绝
```

---

# 二、Session——服务端存身份，Cookie 只存 ID

```
登录流程：
  ① 用户 POST /login 发送 username/password
  ② 服务端验证通过 → 生成随机 Session ID → 存在 Redis/内存中
  ③ 服务端返回 Set-Cookie: session_id=abc123
  ④ 后续请求：客户端自动带 Cookie → 服务端用 session_id 查 Redis 拿到用户信息

Session 数据 = {"user_id": 123, "username": "alice", "role": "admin"}
```

## 2.1 分布式 Session 一致性方案

```
方案 1：粘滞 Session（Sticky Session）
  Nginx/负载均衡器根据 Cookie 值把请求固定路由到同一台机器
  → 不需要共享存储，简单
  → 某台机器挂了 → 这些用户的 Session 全丢

方案 2：集中式 Session（推荐）
  所有服务器都读写同一个 Redis 中的 Session
  → 任意一台服务器都能处理任意用户的请求
  → Redis 负责 Session 的持久化和高可用

方案 3：全复制（不推荐）
  每台服务器之间互相拷贝 Session
  → 网络开销随机器数平方增长
  → 一致性问题（A 服务器存了 Session，B 还没同步完）
```

---

# 三、JWT——客户端自包含身份信息

## 3.1 JWT 结构

```
JWT = Header.Payload.Signature
     = eyJhbGciOiJIUzI1NiJ9.eyJ1c2VyX2lkIjoxMjN9.abc123def456

Header:  {"alg": "HS256", "typ": "JWT"}
Payload: {"user_id": 123, "username": "alice", "exp": 1754223000}
Signature: HMAC-SHA256(base64(Header) + "." + base64(Payload), secret_key)
```

**关键纠正**：Payload 只是 Base64 编码，**不是加密**！任何人都可以解码看到内容。所以 Payload 中**绝对不能放密码、身份证号等敏感数据**。JWT 的"安全"不来自加密，而来自签名保证的防篡改性。

## 3.2 无状态 vs 有状态

```
Session（有状态）：
  优点：服务器完全控制，随时可主动失效
  缺点：每次请求都要查 Redis → 额外延迟 → 分布式 Session 存储需要维护

JWT（无状态）：
  优点：Token 自包含用户信息，不需要查 Redis → 服务器水平扩展零成本
  缺点：签发后就"不可撤销"——在过期之前，Token 就是有效的
       想强制"踢下线"？→ 额外维护黑名单（又回到"状态"了）
```

## 3.3 JWT 的安全性争议

```
争议 1：无法主动撤销
  Session 可以直接删 Redis key → 用户被踢下线
  JWT → 只能等 Token 过期 → 想提前撤销只能维护黑名单（Redis/TTL）

争议 2：Token 泄漏后危害大
  Cookie 有 HttpOnly + SameSite 保护
  JWT 存在 localStorage → XSS 可以直接读取

建议：
  - 敏感操作不要只依赖 JWT，结合短过期时间 + Refresh Token
  - 不要把 JWT 存在 localStorage，存在 HttpOnly Cookie 中
```

## 3.4 Access Token + Refresh Token 双 Token 模式

```
Access Token：短过期（15 分钟-1 小时），用来访问 API
Refresh Token：长过期（7-30 天），用来刷新 Access Token

流程：
  ① 登录 → 返回 Access Token (15m) + Refresh Token (7d)
  ② 用 Access Token 调 API
  ③ Access Token 过期 → 用 Refresh Token 换新的 Access Token
  ④ Refresh Token 也过期 → 重新登录

好处：
  - Access Token 短 → 泄漏后危害窗口小
  - 可以主动"废掉"Refresh Token（在服务端维护黑名单）
  - 用户不活跃 7 天后自动需要重新登录
```

---

# 四、OAuth 2.0——"让这个 App 代替我访问我的数据"

## 4.1 授权码模式——最安全的 OAuth 2.0 流程

```
OAuth 2.0 四个角色：
  - Resource Owner: 用户（你，拥有数据）
  - Client: 第三方应用（想要访问你的数据）
  - Authorization Server: 授权服务器（Google/GitHub/微信）
  - Resource Server: 资源服务器（Google Drive/GitHub API）

流程：
  ① 用户点击"用 Google 登录"
  ② Client → redirect → Authorization Server
     用户：看到 Google 的授权页面
     → "XXX App 想访问你的 Google 姓名和头像，允许吗？"
  ③ 用户允许 → Authorization Server → redirect → Client
     URL 中带有 Authorization Code（一次性、短期）
  ④ Client 拿 Code → 再次请求 Authorization Server → 换 Access Token
     （这一步是 Server → Server，Token 不经过浏览器 URL！）
  ⑤ Client 拿 Access Token → 调 Resource Server → 拿到用户数据

为什么需要第 ④ 步（用 Code 换 Token）？
  因为第 ③ 步的 Code 在 URL 中（浏览器历史记录/referer 头可能泄露）
  第 ④ 步是 Server → Server 的通信，Token 不经过浏览器
```

---

# 五、总结

| 机制 | 核心思想 | 适用 |
|------|---------|------|
| **Cookie + Session** | 服务器存状态，Cookie 只存 Session ID | 传统 Web 应用 |
| **JWT** | 客户端自包含，服务器不存状态 | 微服务/API/移动端 |
| **Access + Refresh Token** | 短 token 保护数据，长 token 控制验证频率 | JWT 场景的安全增强 |
| **OAuth 2.0** | 第三方授权，用户不交出密码 | 社交登录、第三方数据访问 |

# 延伸阅读

**Do——动手验证：**
- 用 `curl -c cookies.txt -b cookies.txt` 跟踪 Cookie 在请求间的传递
- 在 https://jwt.io 解码一个真实的 JWT Token，观察 Header/Payload/Signature
- 用 OAuth 2.0 Playground 模拟一遍授权码流转

**Todo——深入方向：**
- OpenID Connect（OIDC）——在 OAuth 2.0 之上加了一层"我是谁"的身份验证
- PKCE——移动端/SPA 的 OAuth 2.0 安全增强（防止 Authorization Code 拦截）
- MFA（多因素认证）——TOTP/WebAuthn 在登录流程中的集成

*本文参考资料：*
- RFC 6265: HTTP State Management Mechanism (Cookie)
- RFC 7519: JSON Web Token (JWT)
- RFC 6749: The OAuth 2.0 Authorization Framework
- OWASP: Session Management Cheat Sheet
