# Spring Core & Spring Boot Internals Deep Dive

> **Navigation**: [Master Index](README.md) | [Previous: Coding & Problem Solving](03_Coding_Problem_Solving.md) | [Next: Web, REST & Reactive](05_Web_REST_Reactive.md)

---

## 1. Spring Framework Core Architecture

### Q1. Trace the complete Spring Bean Lifecycle from start to finish.
**Answer:**

```
+-----------------------------------------------------------------------------------+
|                            SPRING BEAN LIFECYCLE                                  |
+-----------------------------------------------------------------------------------+
|  1. Bean Definition Loading (@Component, @Bean, XML)                              |
|                               |                                                   |
|  2. Instantiation (Constructor call via reflection / CGLIB)                       |
|                               |                                                   |
|  3. Populate Properties / Dependency Injection (Autowired fields/setters)         |
|                               |                                                   |
|  4. Aware Interfaces Callback (BeanNameAware, BeanFactoryAware, AppContextAware)  |
|                               |                                                   |
|  5. BeanPostProcessor.postProcessBeforeInitialization()                           |
|                               |                                                   |
|  6. Initialization Callbacks:                                                     |
|     -> @PostConstruct                                                             |
|     -> InitializingBean.afterPropertiesSet()                                      |
|     -> Custom init-method defined in @Bean(initMethod="...")                      |
|                               |                                                   |
|  7. BeanPostProcessor.postProcessAfterInitialization()                            |
|     *** CRITICAL: THIS IS WHERE SPRING AOP CREATES DYNAMIC PROXIES! ***          |
|                               |                                                   |
|  8. Bean Ready for Use (Injected into application context)                        |
|                               |                                                   |
|  9. Destruction Callbacks (On Context Shutdown):                                  |
|     -> @PreDestroy                                                                |
|     -> DisposableBean.destroy()                                                   |
|     -> Custom destroy-method                                                      |
+-----------------------------------------------------------------------------------+
```

---

### Q2. How does Spring AOP work? Explain the Self-Invocation Problem.
**Answer:**

1. **Proxy Creation Mechanism**:
   Spring wraps target beans in a dynamic proxy using either:
   - **JDK Dynamic Proxy**: When the target class implements an interface (creates a proxy implementing the interface).
   - **CGLIB Proxy (Default in Spring Boot 2 & 3)**: Dynamically generates a subclass of the target bean at runtime using bytecode generation.
2. **The Self-Invocation Bug**:
   Spring AOP works **only when a method is called from outside the bean through the proxy**. If method `A()` inside bean `Service` calls `this.B()` (which has `@Transactional` or `@Async`), the call is executed directly on the raw instance, completely **bypassing the AOP proxy wrapper**!

```
[ Caller ] ----> [ Spring AOP Proxy ] ----> [ Target Bean ]
                        |                          |
               (Intercepts @Transactional)         |
                        |                          |
                        +---> calls methodA() ---->|
                                                   |-- (this.methodB()) --+
                                                   |                      | (BYPASSES PROXY!)
                                                   |<---------------------+
```

#### How to Solve Self-Invocation:
- **Solution 1 (Recommended)**: Refactor and extract `methodB()` into a separate collaborator bean/service.
- **Solution 2 (Self-Injection)**: Inject the bean into itself via `@Autowired private Service self;` (or `ObjectProvider<Service>`).
- **Solution 3 (AopContext)**: Use `((Service) AopContext.currentProxy()).methodB();` (Requires `@EnableAspectJAutoProxy(exposeProxy = true)`).

---

### Q3. How does Spring resolve Circular Dependencies?
**Answer:**
Spring's `DefaultSingletonBeanRegistry` uses a **Three-Level Cache (Three Maps)** to resolve circular dependencies between singleton beans with setter/field injection:

1. **`singletonObjects` (1st Level Cache)**: Holds fully initialized, ready-to-use singleton beans.
2. **`earlySingletonObjects` (2nd Level Cache)**: Holds early references to instantiated beans (dependencies not yet injected, before initialization).
3. **`singletonFactories` (3rd Level Cache)**: Holds `ObjectFactory` lambdas capable of creating early proxy references if AOP is involved.

> [!WARNING]
> Circular dependencies with **Constructor Injection CANNOT be resolved** (fails with `BeanCurrentlyInCreationException`) because the bean cannot even be instantiated without its dependency. Circular dependencies should be refactored or broken using `@Lazy`.

---

## 2. Spring Boot 3 Ecosystem & Internals

### Q4. What happens behind `@SpringBootApplication`? How does Auto-Configuration work?
**Answer:**

`@SpringBootApplication` is a meta-annotation composed of:
1. `@SpringBootConfiguration`: Marks the class as a configuration source (`@Configuration`).
2. `@ComponentScan`: Scans for components, services, and repositories in the current package and sub-packages.
3. `@EnableAutoConfiguration`: The engine that automatically configures beans based on classpath libraries.

#### How Auto-Configuration Resolves:
1. At startup, Spring Boot scans `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` (in Boot 3) or `META-INF/spring.factories` (in Boot 2).
2. It evaluates conditional annotations on each auto-configuration class:
   - `@ConditionalOnClass({ DataSource.class, EmbeddedDatabaseType.class })`: Configures DataSource only if classes exist on classpath.
   - `@ConditionalOnMissingBean(DataSource.class)`: Registers the auto-configured bean only if the user hasn't defined their own.
   - `@ConditionalOnProperty(prefix = "app.feature", name = "enabled", havingValue = "true")`.
3. Auto-configuration runs **after** user-defined beans are registered, ensuring developer-defined beans always take priority.

---

### Q5. `@ConfigurationProperties` vs. `@Value`: Best Practices.
**Answer:**

```java
// BEST PRACTICE: Type-Safe, Immutable Configuration with Validation
@Validated
@ConfigurationProperties(prefix = "app.payment")
public record PaymentConfig(
    @NotBlank String gatewayUrl,
    @Min(1) @Max(30) int timeoutSeconds,
    @NotNull RetryPolicy retry
) {
    public record RetryPolicy(int maxAttempts, Duration backoff) {}
}
```

| Feature | `@ConfigurationProperties` | `@Value` |
| :--- | :--- | :--- |
| **Type Safety** | Strong (Binds to POJOs / Records) | Weak (String evaluation via SpEL) |
| **Validation** | Supports JSR-380 (`@NotNull`, `@Min`, `@Validated`) | No direct validation support |
| **Relaxed Binding**| Yes (`kebab-case`, `camelCase`, `SNAKE_CASE` map seamlessly) | No (Requires exact property name match) |
| **IDE Support** | Full autocomplete & documentation metadata | Basic / None |

---

## 3. Embedded Tomcat Tuning & Production Sizing

### Q6. How do you tune embedded Tomcat for high-concurrency production workloads?
**Answer:**

```properties
# application.properties Tomcat Thread Pool Sizing
server.tomcat.threads.max=200         # Max worker threads handling concurrent requests (Default: 200)
server.tomcat.threads.min-spare=20    # Minimum worker threads kept warm (Default: 10)
server.tomcat.max-connections=8192    # Max TCP connections server accepts before queuing (Default: 8192)
server.tomcat.accept-count=100        # OS-level TCP backlog queue size when threads are maxed out (Default: 100)
server.tomcat.connection-timeout=5000 # Timeout in ms before closing idle client connection (5s)

# Graceful Shutdown (Spring Boot 2.3+)
server.shutdown=graceful
spring.lifecycle.timeout-per-shutdown-phase=30s
```

#### Connection Flow & Backpressure:
1. Up to `max-connections` ($8192$) TCP sockets are held open by Tomcat's NIO connector.
2. Worker threads ($200$) process requests from active connections concurrently.
3. If all $200$ threads are busy, incoming requests queue up in the OS TCP backlog (`accept-count = 100`).
4. If the backlog overflows, Tomcat rejects additional requests with **`Connection Refused`** (protecting the JVM from crashing).

---

### Q7. Spring Boot Actuator: Securing & Custom Metrics.
**Answer:**

```java
@Component
public class OrderMetrics {
    private final Counter orderCounter;
    private final Timer orderTimer;

    public OrderMetrics(MeterRegistry registry) {
        this.orderCounter = Counter.builder("orders.placed.total")
            .description("Total number of successfully placed orders")
            .tag("channel", "web")
            .register(registry);

        this.orderTimer = Timer.builder("orders.placement.latency")
            .description("Time taken to process and persist order")
            .publishPercentiles(0.5, 0.95, 0.99)
            .register(registry);
    }

    public void recordOrder(Runnable orderProcessingLogic) {
        orderCounter.increment();
        orderTimer.record(orderProcessingLogic);
    }
}
```

```properties
# Production Actuator Security Configuration
management.endpoints.web.exposure.include=health,info,metrics,prometheus
management.endpoint.health.show-details=when_authorized
management.endpoint.health.probes.enabled=true # Kubernetes Liveness & Readiness probes
```

---

> **Next Chapter**: [05 Web, REST & Reactive APIs](05_Web_REST_Reactive.md)
