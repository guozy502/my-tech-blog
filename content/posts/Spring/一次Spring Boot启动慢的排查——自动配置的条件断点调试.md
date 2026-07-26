---
title: "一次Spring Boot启动慢的排查——自动配置的条件断点调试"
date: 2026-06-28
description: 从 -Ddebug 查看自动配置报告、ConditionEvaluationReport 的条件匹配日志、到 Arthas trace 定位启动瓶颈和优化启动时间的四个方向，还原一次线上 Spring Boot 启动慢的完整排查过程。
tags: ["Spring Boot","启动优化","自动配置","Arthas","故障排查"]
categories: ["Spring"]
---

# 一、现象——启动 120 秒

```
Started Application in 120.345 seconds (JVM running for 122.1)
```

正常应该 10-20 秒。发生了什么？

---

# 二、排查第一步——自动配置报告

## 2.1 开启 debug 模式

```bash
# 启动时加 --debug
java -jar app.jar --debug

# 或在 application.properties
debug=true
```

## 2.2 解读自动配置报告

```
============================
CONDITIONS EVALUATION REPORT
============================

Positive matches:        ← 这些自动配置**生效**了
---------------------
  DataSourceAutoConfiguration matched:
    - @ConditionalOnClass found required class 'javax.sql.DataSource'
    - @ConditionalOnMissingBean (type: DataSource) no DataSource bean found

Negative matches:        ← 这些自动配置**被跳过**了
---------------------
  KafkaAutoConfiguration:
    Did not match:
      - @ConditionalOnClass did not find required class 'org.apache.kafka.clients.producer.KafkaProducer'
```

**关键信息**：
- **Positive matches**：看有没有意料之外的自动配置被激活（比如引入了 starter 但不需要的组件）
- **Negative matches**：看有没有该生效却没生效的

---

# 三、排查第二步——ConditionEvaluationReport

```java
// 在启动完成后手动输出条件评估日志
@Component
public class ConditionReportDumper implements ApplicationRunner {
    @Autowired
    private ConfigurableApplicationContext context;
    
    @Override
    public void run(ApplicationArguments args) {
        ConditionEvaluationReport report = 
            context.getBeanFactory().getBean(ConditionEvaluationReport.class);
        for (Map.Entry<String, ConditionEvaluationReport.ConditionAndOutcomes> entry 
             : report.getConditionAndOutcomesBySource().entrySet()) {
            if (entry.getValue().isFullMatch()) continue;  // 只看未匹配的
            System.out.println("NOT matched: " + entry.getKey());
            entry.getValue().forEach(c -> System.out.println("  " + c.getOutcome()));
        }
    }
}
```

---

# 四、排查第三步——定位瓶颈代码

## 4.1 StartupInfoLogger

```java
@Component
public class BeanInitTimer implements BeanPostProcessor {
    private Map<String, Long> startTimes = new ConcurrentHashMap<>();
    
    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) {
        startTimes.put(beanName, System.currentTimeMillis());
        return bean;
    }
    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) {
        Long start = startTimes.remove(beanName);
        long cost = System.currentTimeMillis() - start;
        if (cost > 500) {  // 超过 500ms 的 Bean 初始化
            System.out.println("SLOW BEAN: " + beanName + " → " + cost + "ms");
        }
        return bean;
    }
}
```

## 4.2 Arthas trace 启动瓶颈

```bash
# 启动 Arthas
curl -O https://arthas.aliyun.com/arthas-boot.jar
java -jar arthas-boot.jar

# 追踪 SpringApplication.run()
trace org.springframework.boot.SpringApplication run -n 5 --skipJDKMethod false

# 输出调用链路中的每个耗时节点：
# +---[95.34% 18234ms ] SpringApplication:run()
#     +---[60.12% 10962ms ] AbstractApplicationContext:refresh()
#         +---[35.21% 3859ms ] DefaultListableBeanFactory:preInstantiateSingletons()
#             +---[12.33% 476ms  ] DataSourceInitializer:init()      ← 瓶颈！
```

---

# 五、常见慢的原因与解法

| 原因 | 排查方法 | 解法 |
|------|---------|------|
| **无用自动配置** | debug 报告看正例 | 排除不必要的 starter 或 `@SpringBootApplication(exclude=...)` |
| **数据库连接池初始化慢** | BeanInitTimer + Arthas | 检查网络延迟、DNS、连接超时 |
| **太多 @Bean 用 `initMethod`** | BeanInitTimer | 延迟初始化：`spring.main.lazy-initialization=true` |
| **扫描路径太广** | `@ComponentScan` 检查 | 缩小扫描包范围 |
| **XML/Validation 解析** | Arthas trace Bean | HibernateValidator 校验慢 → 关闭自动校验 |
| **代理生成** | CGLIB 创建 Bean 开销 | 按接口编程，让 JDK 代理可用 |

---

# 六、四个优化方向

```bash
# ① 排除不需要的自动配置
spring.autoconfigure.exclude=\
  org.springframework.boot.autoconfigure.kafka.KafkaAutoConfiguration

# ② 延迟初始化（慎用，可能把问题延迟到运行时）
spring.main.lazy-initialization=true

# ③ 缩小组件扫描范围
@SpringBootApplication(scanBasePackages = "com.example.order")

# ④ 关闭 JMX 注册
spring.jmx.enabled=false
# 或启动参数：-Dspring.jmx.enabled=false
```

---

# 七、总结

| 步骤 | 工具 | 做什么 |
|------|------|--------|
| **①** | `--debug` | 看自动配置报告，排除不需的配置 |
| **②** | ConditionEvaluationReport | 确认 @Conditional 行为符合预期 |
| **③** | BeanPostProcessor 计时 | 定位哪个 Bean 初始化慢 |
| **④** | Arthas trace | 精确定位方法调用链耗时 |
