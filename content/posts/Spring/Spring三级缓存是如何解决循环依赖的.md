---
title: "Spring三级缓存是如何解决循环依赖的"
date: 2026-06-28
description: 从一级缓存的成品 Bean、二级缓存的早期引用、三级缓存的 ObjectFactory，层层拆解 Spring 用三级缓存解决循环依赖的设计精妙——以及为什么构造器注入无解。
tags: ["Spring","循环依赖","三级缓存","AOP","Bean生命周期"]
categories: ["Spring"]
---

```mermaid
flowchart LR
    CACHE1["一级缓存 singletonObjects\n(成品 Bean)"] 
    CACHE2["二级缓存 earlySingletonObjects\n(半成品 Bean 引用)"] 
    CACHE3["三级缓存 singletonFactories\n(ObjectFactory lambda)"]
    
    CIRCLE["A 依赖 B\nB 依赖 A"] --> CACHE3
    CACHE3 -->|"getObject()"| CACHE2
    CACHE2 -->|"属性填充完成"| CACHE1
    
    style CACHE1 fill:#e8f5e9,stroke:#2e7d32
    style CACHE2 fill:#fff3e0,stroke:#f57c00
    style CACHE3 fill:#e3f2fd,stroke:#1565c0
```

---

# 一、什么是循环依赖？

```java
@Component
class A {
    @Autowired B b;  // A 依赖 B
}
@Component
class B {
    @Autowired A a;  // B 依赖 A
}
// A 创建时发现需要 B → 去创建 B → B 发现需要 A → 死循环
```

---

# 二、三级缓存解决了什么？

| 缓存 | 类型 | 存什么 |
|------|------|--------|
| **一级** `singletonObjects` | `Map<String, Object>` | 成品 Bean（属性注入完毕、初始化完成） |
| **二级** `earlySingletonObjects` | `Map<String, Object>` | 半成品 Bean 的引用（已实例化但未填充属性） |
| **三级** `singletonFactories` | `Map<String, ObjectFactory<?>>` | 能生成半成品 Bean 引用的工厂 lambda |

---

# 三、完整流程——A 和 B 互相依赖

```mermaid
sequenceDiagram
    participant C1 as 一级缓存
    participant C2 as 二级缓存
    participant C3 as 三级缓存
    participant A as Bean A
    participant B as Bean B
    
    Note over A: 创建 A
    A->>A: ① 实例化 A（构造器）
    A->>C3: ② A 的 ObjectFactory 放入三级缓存
    A->>A: ③ 填充 A 的属性 → 发现需要 B
    
    Note over B: 创建 B
    B->>B: ④ 实例化 B
    B->>C3: ⑤ B 的 ObjectFactory 放入三级缓存
    B->>B: ⑥ 填充 B 的属性 → 发现需要 A
    
    B->>C1: ⑦ 从一级缓存拿 A？→ 没有
    B->>C2: ⑧ 从二级缓存拿 A？→ 没有
    B->>C3: ⑨ 从三级缓存拿 A → 触发 ObjectFactory.getObject()
    C3->>C2: ⑩ A 的早期引用放入二级缓存，三级缓存删除
    
    B->>B: ⑪ B 拿到 A 的引用 → 完成属性填充 → B 初始化完成
    B->>C1: ⑫ 成品 B 放入一级缓存
    
    A->>C1: ⑬ A 拿到成品 B → 完成属性填充 → A 初始化完成
    A->>C1: ⑭ 成品 A 放入一级缓存
```

---

# 四、三级缓存的源码核心

```java
// DefaultSingletonBeanRegistry.getSingleton()
protected Object getSingleton(String beanName, boolean allowEarlyReference) {
    Object singletonObject = this.singletonObjects.get(beanName);      // 一级
    if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
        singletonObject = this.earlySingletonObjects.get(beanName);    // 二级
        if (singletonObject == null && allowEarlyReference) {
            ObjectFactory<?> factory = this.singletonFactories.get(beanName); // 三级
            if (factory != null) {
                singletonObject = factory.getObject();  // ← 关键：触发 lambda
                this.earlySingletonObjects.put(beanName, singletonObject); // 升级二级
                this.singletonFactories.remove(beanName);                // 删除三级
            }
        }
    }
    return singletonObject;
}
```

**三级缓存里的 lambda 干了什么？**

```java
// AbstractAutowireCapableBeanFactory.doCreateBean()
addSingletonFactory(beanName, () -> 
    getEarlyBeanReference(beanName, mbd, bean)  // ← 生成早期引用
);
// getEarlyBeanReference() 会经过 SmartInstantiationAwareBeanPostProcessor
// 如果有 AOP 代理，这里返回代理对象！
```

---

# 五、为什么三级缓存需要存在？

> **如果只是解决循环依赖，二级缓存就够了。三级之所以存在，是为了兼容 AOP——在早期引用阶段就把代理对象创建好。**

如果没有三级缓存：
- A 的早期引用是原始对象
- A 初始化后需要 AOP 代理 → 代理对象 ≠ 原始对象
- B 拿到的原始对象引用就错了

有了三级缓存：
- `getEarlyBeanReference()` 在三级转二级时触发 → 先判断要不要代理
- 要代理 → 直接生成代理对象放入二级缓存
- B 拿到的就是正确的代理引用

---

# 六、构造器注入为什么无解？

```java
@Component
class A {
    B b;
    A(B b) { this.b = b; }  // 构造器注入
}
```

构造器注入的依赖在**实例化阶段**就需要——此时连三级缓存都还没放进去。`new A(b)` 这行代码本身就死循环了，缓存帮不上忙。

**解决方案**：构造器循环依赖 → 加 `@Lazy` 让注入的只是一个代理。

```java
A(@Lazy B b) { this.b = b; }  // b 是一个代理对象，不会触发真正的 B 创建
```

---

# 七、总结

| 问题 | 答案 |
|------|------|
| **一级缓存** | 存成品 Bean，getBean 的默认来源 |
| **二级缓存** | 存半成品引用，已经过 getEarlyBeanReference |
| **三级缓存** | 存 ObjectFactory lambda，三级转二级时触发 |
| **三级为什么存在** | 兼容 AOP——早期引用阶段就确定代理关系 |
| **构造器注入为什么不行** | 实例化阶段还没放缓存，依赖就卡死了 |
