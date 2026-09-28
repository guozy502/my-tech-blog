---
title: "Tomcat 与 Jetty——从容器嵌套到 Handler 链的架构差异"
date: 2026-08-07
description: 从 Tomcat 的四层容器嵌套模型（Engine→Host→Context→Wrapper）与 NIO 线程模型（Acceptor→Poller→Worker）、Jetty 的 Handler 责任链模型与 Continuations 的异步先驱、Tomcat 与 Jetty 在类加载器隔离策略上的差异（先自身后委托 vs 先委托后自身）、到 maxThreads 的调优公式与生产环境选型依据，拆解两大 Servlet 容器在架构哲学上的根本分歧。
tags: ["架构","Tomcat","Jetty","Servlet","类加载","NIO"]
categories: ["架构"]
---

# 历史背景——为什么会有两种 Servlet 容器？

1999 年，Sun 发布了 Servlet 2.2 规范。同年，Apache 基金会启动 Tomcat 项目作为 Servlet 规范的**参考实现（Reference Implementation）**。Tomcat 的目标是"完整、正确地实现 Servlet 规范"，架构设计以规范为纲——容器嵌套、web.xml 配置、标准的类加载层级。

1995 年，Greg Wilkins 创建了 Jetty（比 Tomcat 更早！）。Jetty 的定位是"轻量级、可嵌入的 HTTP 服务器和 Servlet 容器"。它不是作为 Servlet 规范的参考实现，而是作为一个**对开发者友好的 HTTP 基础设施**。这个定位差异决定了两个容器在后来的分化：Tomcat 服务于传统的 Web 应用部署场景（war 包、独立运维），Jetty 服务于嵌入式场景（Spring Boot 内嵌、微服务、自动化测试）。

Spring Boot 之所以同时支持 Tomcat 和 Jetty 作为内嵌容器，就是因为 Jetty 的轻量级设计让它在启动速度、内存占用、嵌入友好度上比 Tomcat 更有优势。但 Tomcat 的社区规模、文档完善度、Servlet 规范的跟进速度是 Jetty 比不了的。

---

# 一、Tomcat 核心架构——四层容器嵌套

## 1.1 架构全景

Tomcat 把"处理请求"这件事抽象成了一个嵌套的**容器（Container）层级**：

```
Server（代表整个 Tomcat 实例，一个 JVM 一个 Server）
  └── Service（一个 Server 可以有多个 Service）
        ├── Connector（接收请求，字节流 → HttpServletRequest）
        │     ├── ProtocolHandler
        │     │     ├── Endpoint（网络 IO：NIO/NIO2/APR）
        │     │     └── Processor（协议解析：HTTP/1.1→Http11Processor）
        │     └── Adapter（CoyoteAdapter：Tomcat Request → Servlet Request）
        │
        └── Engine（处理所有请求的顶层容器）
              └── Host（虚拟主机：根据 Host 头路由）
                    └── Context（一个 Web 应用 = 一个 WAR）
                          └── Wrapper（一个 Servlet 实例）
```

**面试追问：为什么分这么多层？** 每一层是一级路由——Host 按域名路由（`a.example.com` → Host A），Context 按 URL 前缀路由（`/app1` → Context A），Wrapper 按 URL 精确匹配（`/users` → UserServlet）。这个设计把路由决策分散到各自职责明确的层级，而不是一个中央路由器。每一层只管"找到下一个包含我的孩子"。

## 1.2 Connector 的 IO 模型——三种实现

| 模型 | 实现类 | 工作方式 | 适用 |
|------|--------|---------|------|
| **BIO** | `Http11Protocol` | 每个连接一个线程，`accept` 阻塞 + `read` 阻塞 | Tomcat 7 及之前的默认 |
| **NIO** | `Http11NioProtocol` | 单线程 accept → Poller ←→ epoll 管理连接 → Worker 线程池处理 | **Tomcat 8+ 默认** |
| **NIO2** | `Http11Nio2Protocol` | JDK 7 `AsynchronousChannel` 异步 IO | 理论上更高效但代码复杂 |
| **APR** | `Http11AprProtocol` | OS 原生 sendfile/epoll（Apache Portable Runtime） | 性能最高 |

**NIO 模式的三类线程**：

```
┌──────────┐     ┌─────────────┐     ┌──────────────────┐
│ Acceptor │     │   Poller    │     │  Worker 线程池    │
│  (1 个)  │     │  (1-2 个)   │     │  (maxThreads=200) │
└────┬─────┘     └──────┬──────┘     └────────┬─────────┘
     │                  │                     │
  ① accept          ② epoll_wait         ③ 读 HTTP 请求
  新连接注册         检测就绪事件          解析 → 调用 Servlet
  到 Poller           提交给 Worker        返回响应
```

**面试追问：Acceptor 为什么只需要 1 个线程？** `accept()` 操作的开销极低——只是从内核的 accept queue 中取出一个已建好的连接（三次握手已在内核层完成）。一个线程循环 `while(true) accept()` 足以处理数万 QPS 的建连速率。

## 1.3 完整请求处理链路

```
① 客户端 TCP 连接到达 → Acceptor accept → 封装为 NioSocketWrapper
② 注册到 Poller 的 Selector 上 → Poller 在 epoll_wait 中检测到可读事件
③ Poller 创建 SocketProcessor 任务 → 提交给 Worker 线程池
④ Worker 线程：
   a. Http11Processor 解析 HTTP 请求行（GET /api/users HTTP/1.1）
   b. 解析 HTTP 请求头（Host/Content-Type/Authorization...）
   c. 解析 HTTP 请求体（如有）
   d. 生成 Tomcat 内部的 Request 对象
⑤ CoyoteAdapter：把 Tomcat Request 转成 HttpServletRequest（Servlet 规范）
⑥ Engine → Host → Context → Wrapper → 调用 Servlet.service()
⑦ Servlet 返回 → Filter 链包装响应
⑧ 长连接 → Socket 归还 Poller；短连接 → 关闭 Socket
```

---

# 二、Jetty 核心架构——Handler 责任链

## 2.1 Handler 链——和 Tomcat 容器嵌套的本质差异

```
Tomcat 模型：容器嵌套（Container）
  Engine 包含 Host，Host 包含 Context，Context 包含 Wrapper
  → 每一层找到下一层，最终找到 Servlet
  → 像"俄罗斯套娃"，层层传递

Jetty 模型：Handler 链（Chain of Responsibility）
  Server → HandlerCollection（一组 Handler）
    ├── ContextHandler："URL 匹配 /app？归我 → 交给 ServletHandler"
    ├── ResourceHandler："URL 匹配 /static/*？归我 → 返回文件"
    └── DefaultHandler："都不匹配 → 返回 404"
  → 不是"层层传递"，而是"链式调用"，每个 Handler 自己判断"归我吗"
```

```java
// Jetty 内嵌启动——代码说话
Server server = new Server(8080);

// Handler 链就是"谁遇到谁处理"
HandlerList handlers = new HandlerList();
handlers.setHandlers(new Handler[]{
    new ResourceHandler() {{           // Handler 1: 静态文件
        setResourceBase("/www/public");
    }},
    new ServletHandler() {{            // Handler 2: Servlet
        addServletWithMapping(MyServlet.class, "/api/*");
    }},
    new DefaultHandler()               // Handler 3: 兜底 404
});

server.setHandler(handlers);
server.start();
```

**Spring Boot 为什么内置了 Jetty 作为替代选择？** 因为 Jetty 的 Handler 模型让它更容易"嵌入"——你只需要 `new Server(port)` + `server.setHandler(myHandler)` 就能启动。没有 Container/Wrapper 的层级概念需要理解，启动极快。

## 2.2 Continuations——Servlet 3.0 异步之前的先驱

Servlet 3.0（2009 年）才引入 `AsyncContext` 支持异步处理。但 Jetty 6（2006 年）就有了自己的 **Continuations** 机制：

```java
// Jetty 6 Continuations（在 Servlet 3.0 之前！）
Continuation cont = ContinuationSupport.getContinuation(request);
// 暂停当前请求处理，释放 Worker 线程，等数据就绪后再恢复
// → 允许一个 Worker 线程同时服务多个等待中的请求

// 现在全部改用标准的 AsyncContext
AsyncContext ctx = request.startAsync();
// 同样的效果，但这是标准的 Servlet 3.0+ API
```

---

# 三、类加载器——隔离策略的差异

## 3.1 为什么需要隔离？

```
场景：同一台服务器上部署了两个 Web App
  WebApp A：Spring 5.0（依赖 spring-core-5.0.jar）
  WebApp B：Spring 6.0（依赖 spring-core-6.0.jar）
  → 两个 jar 的同名类不能共用！A 和 B 需要各自加载自己的版本
```

**每个 Web App 有独立的 WebappClassLoader** ——不同的 ClassLoader 加载同一个类名产生的两个 Class 对象，JVM 认为是不同的类。

## 3.2 Tomcat 的类加载器——打破双亲委派

```
Bootstrap ClassLoader（java.lang.* / java.util.*）
    ↓
System ClassLoader（CLASSPATH / bin/bootstrap.jar）
    ↓
Common ClassLoader（Tomcat 公共库：$CATALINA_HOME/lib/*.jar）
    ↓
WebappClassLoader（每个 Web App 独立的加载器）

关键：WebappClassLoader 的加载顺序和双亲委派相反！
  普通双亲委派：先委托父加载器 → 父找不到 → 自己加载
  Tomcat WebappClassLoader：
    ① 先自己在 WEB-INF/lib 中找（优先本地！）
    ② 找不到 → 再委托给 Common ClassLoader（共享 Tomcat lib）

这样 Webapp A 的 Spring 5.0（在 WEB-INF/lib 里）不会被 Common 的 Spring 6.0 覆盖
```

**把 jar 放在 WEB-INF/lib vs Tomcat lib 的区别**：

| 位置 | 加载器 | 可见范围 |
|------|--------|---------|
| `WEB-INF/lib` | WebappClassLoader | 只有这个 Web App 自己 |
| `$CATALINA_HOME/lib` | Common ClassLoader | **所有 Web App 共享**（如 MySQL Driver 应该放这） |

## 3.3 Jetty 的类加载器——优先委托系统

Jetty 也是每个 Web App 独立的 WebAppClassLoader。但有一个关键差异——**它优先委托给 System ClassLoader（不委托给 Common Loader）**：

```
Tomcat：先自身 WEB-INF/lib → 再委托 Common
Jetty：先委托 System → 再自身 WEB-INF/lib
  → javax.servlet 等 API 由 System ClassLoader 加载，所有 Web App 天然共享
  → 不需要像 Tomcat 那样在 Common 和 Webapp 之间显式配置类的共享
```

## 3.4 类加载导致的常见故障

| 故障 | 现象 | 根因 |
|------|------|------|
| **ClassNotFoundException** | `java.lang.ClassNotFoundException: com.example.MyService` | `WEB-INF/lib` 缺 jar |
| **NoClassDefFoundError** | 编译时有这个类，运行为时找不到 | 静态初始化时依赖类不存在 → 类加载失败 |
| **LinkageError** | `java.lang.LinkageError: loader constraint violation` | 同一个类被两个 ClassLoader 各加载了一次 → JVM 不知道用哪个 |
| **ClassCastException** | `$Proxy32 cannot be cast to com.example.User` | 同上——`com.example.User` 被两个不同 ClassLoader 加载，JVM 认为它们不是同一个类 |

---

# 四、性能与调优——两个方向的优化

## 4.1 Tomcat 线程池调优——maxThreads 怎么算？

```xml
<Connector port="8080" 
           protocol="org.apache.coyote.http11.Http11NioProtocol"
           maxThreads="200"          <!-- Worker 线程池最大值 -->
           minSpareThreads="25"      <!-- 空闲线程池最小值 -->
           acceptCount="100"         <!-- 线程全忙时，TCP accept 队列长度 -->
           connectionTimeout="20000" <!-- 连接超时 ms -->
           maxConnections="10000"    <!-- 最大并发连接数 -->
/>
```

**maxThreads 不是越大越好——IO 密集型场景的估算公式**：

```
maxThreads ≈ CPU核数 × (1 + 平均等待时间 / 平均计算时间)

例：4 核 CPU，每个请求数据库等待 50ms，实际计算 5ms
  → maxThreads ≈ 4 × (1 + 50/5) = 4 × 11 = 44

通用推荐：
  中小型应用：200-500
  大型 IO 密集型（大量等后端/等数据库）：500-1000
  超过 1000 → 考虑异步 Servlet 而不是继续加线程（线程调度开销 > 收益）
```

## 4.2 Tomcat vs Jetty 调优差异

| 维度 | Tomcat | Jetty |
|------|--------|-------|
| **启动速度** | 较慢（容器层级初始化多） | **极快**（Handler 链直接组装，比 Tomcat 快 2-4 倍） |
| **默认内存** | 100-150MB | 50-80MB |
| **默认 IO** | NIO（8+） | NIO |
| **嵌入友好度** | `Tomcat tomcat = new Tomcat()` → 但理解容器层级是前提 | **优秀**——`new Server()` 即可，不需要理解容器层级 |
| **WebSocket** | 较弱 | **强**（Jetty WebSocket 是公认的参考实现） |
| **Servlet 规范跟进** | **最快**（参考实现） | 次之 |

---

# 五、选型——什么时候该用哪个？

```
✅ 选 Tomcat：
  - 传统的 Servlet 应用（Spring MVC / JSP / 旧 war 包部署）
  - 需要最广泛的社区支持和文档
  - 需要最快跟进的 Servlet 规范支持（如 Servlet 6.0）
  - 有专门的运维团队管理容器配置

✅ 选 Jetty：
  - 嵌入式部署（Spring Boot starter-jetty 替代 starter-tomcat）
  - 微服务/容器化环境（启动快 + 内存小 → Pod 启动时间短）
  - WebSocket 密集型（实时聊天、协作编辑、游戏服务器）
  - 自动化测试（内嵌启动极快，每个测试类一个 Server 也没有负担）

✅ 选 Undertow：
  - 追求极致性能（直接操作 ByteBuffer，越过 Servlet 流的抽象层）
  - 需要 HTTP/2 + WebSocket 同一条物理连接
```

---

# 六、总结

| 维度 | Tomcat | Jetty |
|------|--------|-------|
| **架构哲学** | 容器嵌套（Container），层层包含 | **Handler 责任链**，链式调用 |
| **启动速度** | 慢 | **快**（2-4 倍） |
| **内存占用** | 100-150MB | **50-80MB** |
| **嵌入友好度** | 需要理解容器层级 | **一行 `new Server()` 就够** |
| **IO 模型** | NIO（默认） | NIO（默认） |
| **类加载顺序** | 先自身 → 后委托 Common | 先委托 System → 后自身 |
| **WebSocket** | 较弱 | **最强（公认的参考实现）** |
| **Servlet 规范** | **最快跟进（参考实现）** | 次之 |
| **生态规模** | **最大** | 较小但专用 |
| **适用** | 传统 Web 应用、独立运维 | **嵌入式、微服务、WebSocket、测试** |

# 延伸阅读

**Do——动手验证：**
- 启动一个 Spring Boot 应用（分别用 Tomcat 和 Jetty）→ `spring-boot-starter-web` 排除 Tomcat → 加 `spring-boot-starter-jetty`，对比启动时间的差异
- `jstack -l <tomcat-pid>` 观察 Acceptor / Poller / Worker 三类线程的命名和状态
- 在 Tomcat 的 `server.xml` 中把 `maxThreads` 从 200 调到 10，用 wrk 压测观察请求排队和拒绝的日志

**Todo——深入方向：**
- Tomcat 的 `Nio2Endpoint`——JDK 的 `AsynchronousChannel` 异步 IO 在 Tomcat 中的实际表现
- Jetty 的 `QueuedThreadPool`——为什么 Jetty 的线程池比 Tomcat 的 `ThreadPoolExecutor` 更高效
- Servlet 6.0 / Jakarta EE 10——`javax.servlet.*` 改名为 `jakarta.servlet.*` 的原因和迁移方案

*本文参考资料：*
- Apache Tomcat 9/10 官方文档: Architecture / Class Loading
- Eclipse Jetty 官方文档: Architecture / Embedding Jetty
- Servlet 3.0 Specification: Asynchronous Processing
