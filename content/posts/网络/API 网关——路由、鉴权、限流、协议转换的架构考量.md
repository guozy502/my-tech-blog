---
title: "API 网关——路由、鉴权、限流、协议转换的架构考量"
date: 2026-07-29
description: 从 API 网关的核心功能（路由转发/鉴权校验/限流熔断/协议转换/日志采集）、插件链设计模式（类比 Nginx 11 阶段）、Kong/APISIX/ShenYu 三种主流网关的架构对比，到网关无状态化+配置中心的高可用部署，拆解统一入口在微服务架构中的角色与选型依据。
tags: ["网络","API网关","Kong","APISIX","路由","鉴权"]
categories: ["网络"]
---

# 历史背景——微服务多了，网关就成了必需品

单体应用时代没有网关——前端直接调后端 API，一个 `nginx.conf` 搞定路由。微服务拆分后，后端变成了几十甚至几百个独立部署的服务。每个服务有自己的鉴权、限流、日志、协议格式——前端面对的不是一个"后端"而是一个"微服务动物园"。

API 网关（API Gateway）就是在这个背景下成为微服务标配的——**把每一个微服务都需要处理的通用功能（鉴权、限流、日志、协议转换）上提到网关层统一处理**。服务端只需要专注业务逻辑，公共能力由网关统一兜底。

---

# 一、API 网关的核心功能——不止是"转发"

```
请求 → API 网关
  ├── ① 路由转发：根据 URL/Header/Query 分发到不同微服务
  ├── ② 鉴权校验：JWT 验证 / OAuth Token / IP 白名单
  ├── ③ 限流熔断：令牌桶 / 漏桶 + 熔断降级
  ├── ④ 协议转换：HTTP → gRPC / REST → GraphQL
  ├── ⑤ 日志采集：每个请求的耗时/状态码/来源
  ├── ⑥ 灰度发布：根据 Header/Cookie 路由到不同版本的服务
  └── ⑦ 请求聚合：把多个微服务的响应组合为一个响应返回给客户端
```

---

# 二、网关的插件链设计——类比 Nginx 11 阶段

网关的请求处理不是"查路由→转发→返回"这么简单，而是一个多阶段的处理流水线：

```
请求到达网关
  ├── ① 解析阶段：解码 URL、解析 Header
  ├── ② 路由阶段：根据路由表找到背后的后端服务
  ├── ③ 前置过滤阶段：
  │     ├── 限流插件 (limit-req)
  │     ├── 鉴权插件 (JWT / key-auth)
  │     ├── IP 黑白名单插件
  │     └── 请求改写插件 (proxy-rewrite)
  ├── ④ 代理阶段：将请求转发到后端微服务
  ├── ⑤ 后置过滤阶段：
  │     ├── 响应改写插件
  │     └── 日志插件 (记录请求耗时/状态码)
  └── ⑥ 日志阶段：写入 access log
```

**为什么要插件化？** 因为每个团队对网关的需求不同——A 团队需要 JWT 鉴权，B 团队需要 OAuth，C 团队不需要鉴权。如果网关把你的鉴权逻辑硬编码在框架里，B 和 C 团队就得 fork 代码。插件化让网关保持内核的稳定，同时让每个租户通过自由组合插件来适配自己的需求。

---

# 三、主流网关对比——Kong vs APISIX vs ShenYu

| | Kong | APISIX | ShenYu |
|------|------|--------|--------|
| **基础** | OpenResty (Nginx + LuaJIT) | OpenResty (Nginx + LuaJIT) | Java (Netty + WebFlux) |
| **插件语言** | Lua / Go (PDK) / JS | Lua / Java / Go / WASM | Java |
| **配置存储** | PostgreSQL + Cassandra | etcd | ZooKeeper / etcd / Nacos / Consul |
| **性能** | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ |
| **生态** | 最大（200+ 插件） | 国内增长快 | Spring Cloud 集成 |
| **选型建议** | 需要最成熟的生态 | 需要高性能+国产替代 | 纯 Java 团队，不想写 Lua |

**Kong vs APISIX 的关键差异**：
```
Kong 的配置存储在 PostgreSQL 中，更新配置 = 写 DB + 通知 Worker
APISIX 的配置存储在 etcd 中，更新配置 = 写 etcd → etcd Watch 推送变化
→ etcd 的推送模式比 DB 的轮询模式延迟更低
```

---

# 四、网关的部署——无状态 + 水平扩展

```nginx
# 网关的核心部署原则：网关本身是无状态的
# 配置存在外部存储（etcd/DB），网关实例只加载配置并处理流量

# 高可用部署：
  负载均衡器 (L4/LB) → 网关节点 1/2/3 → 后端微服务
                       └────── 配置中心 (etcd/Nacos/DB)
```

---

# 五、总结

| 功能 | 解决的问题 | 实现方式 |
|------|---------|---------|
| **路由** | 前端只需要知道一个入口 | URL/Header 匹配后端服务 |
| **鉴权** | 每个服务自己写鉴权？不！ | 在网关统一拦截 JWT/Token |
| **限流** | 防止单个用户/服务打垮后端 | 令牌桶/漏桶 + 配置中心动态调 |
| **协议转换** | REST API 网关 + gRPC 微服务 | 网关内部 HTTP↔gRPC 转换 |
| **灰度** | 新版本上线只让小部分用户看到 | 按 Header/Cookie 路由不同版本 |

# 延伸阅读

**Do——动手搭建：**
- Docker 一键部署 APISIX + etcd，配置一个上游服务 + 路由规则
- 在 APISIX 上配置 limit-req 插件和 JWT 鉴权插件，验证限流与鉴权是否生效
- 用 wrk 压测网关代理 vs 直连后端，观察网关层增加的延迟

**Todo——深入方向：**
- 网关的灰度方案——金丝雀（Canary）+ 蓝绿（Blue-Green）+ A/B Testing
- API 网关的 Service Mesh 化——网关与 Sidecar 代理（Envoy）的关系
- 自定义网关插件开发——用 Lua/Go/Java 写一个 APISIX/Kong 插件

*本文参考资料：*
- Kong 官方文档: https://docs.konghq.com/
- APISIX 官方文档: https://apisix.apache.org/docs/
- ShenYu 官方文档: https://shenyu.apache.org/docs/
