---
title: "Spring MVC请求处理全链路——从DispatcherServlet到返回值"
date: 2026-06-28
description: 从 DispatcherServlet 的 doDispatch、HandlerMapping 的路由匹配、HandlerAdapter 的反射调用、到 HttpMessageConverter 的请求/响应体序列化，全链路拆解一个 HTTP 请求在 Spring MVC 中的处理过程。
tags: ["Spring","Spring MVC","DispatcherServlet","HandlerMapping","源码"]
categories: ["Spring"]
---

```mermaid
flowchart TD
    REQ["HTTP 请求"] --> DS["DispatcherServlet.doDispatch()"]
    DS --> HM["HandlerMapping\n根据 URL 找 Handler"]
    HM --> HA["HandlerAdapter\n调用 Handler"]
    HA --> EXECUTE["执行业务逻辑\n返回 ModelAndView / @ResponseBody"]
    EXECUTE --> VIEW{"视图 or 数据?"}
    VIEW -->|"视图"| VR["ViewResolver\n渲染 JSP/Thymeleaf"]
    VIEW -->|"@ResponseBody"| MC["HttpMessageConverter\nJackson → JSON"]
    VR --> RESP["HTTP 响应"]
    MC --> RESP
    
    style DS fill:#e3f2fd,stroke:#1565c0
    style EXECUTE fill:#e8f5e9,stroke:#2e7d32
```

---

# 一、DispatcherServlet——请求的中央调度器

```java
protected void doDispatch(HttpServletRequest request, HttpServletResponse response) {
    HandlerExecutionChain mappedHandler = null;
    try {
        // ① 根据请求 URL 找到 Handler（Controller 方法）
        mappedHandler = getHandler(request);              // HandlerMapping
        // ② 找到能执行这个 Handler 的 Adapter
        HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());
        // ③ 执行拦截器 preHandle
        if (!mappedHandler.applyPreHandle(request, response)) return;
        // ④ 真正调用 Controller 方法
        ModelAndView mv = ha.handle(request, response, mappedHandler.getHandler());
        // ⑤ 执行拦截器 postHandle
        mappedHandler.applyPostHandle(request, response, mv);
        // ⑥ 处理视图或 @ResponseBody 返回值
        processDispatchResult(request, response, mappedHandler, mv);
    } finally {
        // ⑦ 执行拦截器 afterCompletion
        mappedHandler.triggerAfterCompletion(request, response);
    }
}
```

---

# 二、HandlerMapping——URL → Handler 的路由

```mermaid
flowchart LR
    REQ["GET /user/1"] --> RM["RequestMappingHandlerMapping\n扫描 @RequestMapping\n构建 URL → handler 映射表"]
    RM --> CHAIN["返回 HandlerExecutionChain\n(handler + N 个 interceptor)"]
```

```java
// RequestMappingHandlerMapping 初始化时扫描所有 @Controller 的方法
// 构建映射：GET /user/{id} → UserController.getById(Long id)
// 请求到达时：提取路径变量、匹配最佳 handler
```

---

# 三、HandlerAdapter——适配多种 Handler

```java
public interface HandlerAdapter {
    // 判断能否处理
    boolean supports(Object handler);
    // 真正执行
    ModelAndView handle(HttpServletRequest req, HttpServletResponse resp, Object handler);
}

// 实现类：
// RequestMappingHandlerAdapter    → @RequestMapping 方法（最常见）
// HttpRequestHandlerAdapter       → HttpRequestHandler 实现
// SimpleControllerHandlerAdapter  → Controller 接口实现（已废弃）
```

---

# 四、参数解析——你的参数是怎么到的 Controller

```java
// HandlerMethodArgumentResolver 体系
// 遍历所有 ArgumentResolver，找到能解析当前参数的那个
for (HandlerMethodArgumentResolver resolver : argumentResolvers) {
    if (resolver.supportsParameter(parameter)) {
        Object arg = resolver.resolveArgument(parameter, mavContainer, request, ...);
        args[i] = arg;
        break;
    }
}
```

| ArgumentResolver | 解析什么 |
|-----------------|---------|
| `@RequestParam` | `?name=xxx` query 参数 |
| `@PathVariable` | `/user/{id}` 路径变量 |
| `@RequestBody` | JSON 请求体 → 反序列成对象 |
| `@RequestHeader` | HTTP Header |
| 无注解的 Model | 按名称从 Model 中取值 |

---

# 五、返回值处理——@ResponseBody 如何变成 JSON

```java
// HandlerMethodReturnValueHandler 体系
for (HandlerMethodReturnValueHandler handler : returnValueHandlers) {
    if (handler.supportsReturnType(returnType)) {
        handler.handleReturnValue(returnValue, returnType, mavContainer, request, response);
        break;
    }
}
```

```mermaid
flowchart LR
    RET["Controller 返回\nUser{id:1, name:'Tom'}"] -->|"@ResponseBody"| CONV["HttpMessageConverter\nMappingJackson2HttpMessageConverter"]
    CONV -->|"ObjectMapper.writeValueAsString"| JSON['{"id":1,"name":"Tom"}']
    JSON --> RESP["HTTP 200\nContent-Type: application/json"]
```

---

# 六、拦截器（Interceptor）链

```mermaid
sequenceDiagram
    participant DS as DispatcherServlet
    participant I1 as Interceptor1
    participant I2 as Interceptor2
    participant C as Controller
    
    DS->>I1: preHandle → true
    DS->>I2: preHandle → true
    DS->>C: 执行业务方法
    C-->>DS: 返回结果
    DS->>I2: postHandle
    DS->>I1: postHandle
    DS->>DS: 渲染视图/序列化响应
    DS->>I2: afterCompletion
    DS->>I1: afterCompletion
```

---

# 七、总结

| 组件 | 职责 |
|------|------|
| **DispatcherServlet** | 中央调度器，请求入口 |
| **HandlerMapping** | URL → Handler 路由匹配 |
| **HandlerAdapter** | 适配调用 Handler |
| **ArgumentResolver** | 参数解析（@RequestParam/@RequestBody/...） |
| **ReturnValueHandler** | 返回值处理（@ResponseBody → JSON） |
| **Interceptor** | 请求前/后/完成时的切面拦截 |
