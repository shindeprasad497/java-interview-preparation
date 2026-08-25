# 14. Spring Boot 3 Internals, Auto-Configuration & Embedded Tomcat

> **Navigation**: [Master Index](README.md) | [Previous: Spring Core & AOP](13_Spring_Core_IoC_AOP_Internals.md) | [Next: Spring Web & REST APIs](15_Spring_Web_REST_APIs.md)

---

## 📌 Chapter Overview
This module explores **Spring Boot 3 internal mechanics**, the Auto-Configuration resolution engine (`AutoConfiguration.imports`), `SpringApplication.run()` lifecycle phases, **Embedded Tomcat production tuning**, and **Actuator / Micrometer metrics**.

---

## 1. Auto-Configuration Mechanics & Conditionals

```
+-----------------------------------------------------------------------------------+
|                        SPRING BOOT AUTO-CONFIGURATION ENGINE                      |
+-----------------------------------------------------------------------------------+
|  1. @SpringBootApplication                                                        |
|     -> @EnableAutoConfiguration                                                   |
|        -> AutoConfigurationImportSelector                                         |
|           -> Reads: META-INF/spring/org.springframework.boot.autoconfigure.       |
|                     AutoConfiguration.imports                                     |
|                                                                                   |
|  2. Condition Evaluation:                                                         |
|     - @ConditionalOnClass(DataSource.class)       -> Is class on classpath?       |
|     - @ConditionalOnMissingBean(DataSource.class)  -> Has user defined bean?      |
|     - @ConditionalOnProperty("spring.datasource") -> Is property enabled?        |
|                                                                                   |
|  3. Register Auto-Configured Beans in ApplicationContext                          |
+-----------------------------------------------------------------------------------+
```

### Q1. How does Auto-Configuration work in Spring Boot 3 vs Spring Boot 2.x?
**Answer:**
- **In Spring Boot 2.x**: Auto-configurations were listed in `META-INF/spring.factories` under `org.springframework.boot.autoconfigure.EnableAutoConfiguration`.
- **In Spring Boot 3.x (Spring Framework 6.x)**: Switched to the more efficient file location:
  `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`

#### Core Conditional Annotations:
- **`@ConditionalOnClass`**: Loads configuration only if specified classes exist on the classpath.
- **`@ConditionalOnMissingBean`**: Registers a default bean *only if* the user has not defined their own custom bean (enabling easy user override).
- **`@ConditionalOnProperty`**: Activates configuration based on application configuration keys (`prefix = "feature", name = "enabled", havingValue = "true"`).

---

## 2. Embedded Tomcat High-Concurrency Tuning

### Q2. How do you size and tune Embedded Tomcat for high-throughput production?
**Answer:**

```
                                 [ Incoming HTTP Traffic ]
                                             |
                                             v
                           +-----------------------------------+
                           |  OS TCP Backlog Queue             |
                           |  (server.tomcat.accept-count=100) |
                           +-----------------------------------+
                                             |
                                             v
                           +-----------------------------------+
                           |  Tomcat Poller / Max Connections  |
                           |  (server.tomcat.max-connections=  |
                           |   8192 NIO Sockets)               |
                           +-----------------------------------+
                                             |
                                             v
                           +-----------------------------------+
                           |  Tomcat Worker Thread Pool        |
                           |  (min-spare=20, max=200)          |
                           +-----------------------------------+
```

#### Production Configuration (`application.yml`):
```yaml
server:
  port: 8080
  tomcat:
    # 1. Active Worker Threads
    threads:
      min-spare: 20       # Minimum idle worker threads always kept alive
      max: 200            # Maximum concurrent worker threads (Default: 200)
    
    # 2. Connection Buffers
    max-connections: 8192 # Max simultaneous TCP sockets the NIO connector accepts
    accept-count: 100     # OS TCP queue length when all max-threads are busy
    
    # 3. Timeouts
    connection-timeout: 5000ms # Time to wait for request headers before closing socket
    keep-alive-timeout: 60000ms
```

---

## 3. Spring Boot Actuator & Micrometer Custom Metrics

### Q3. How do you implement custom business metrics in Spring Boot 3 with Micrometer?
**Answer:**

```java
@Service
public class PaymentProcessingService {

    private final Counter paymentSuccessCounter;
    private final Counter paymentFailureCounter;
    private final Timer paymentProcessingTimer;

    public PaymentProcessingService(MeterRegistry meterRegistry) {
        // 1. Counter: Monotonically increasing metric
        this.paymentSuccessCounter = Counter.builder("payments.processed.success")
            .description("Total successful payment transactions")
            .tag("env", "production")
            .register(meterRegistry);

        this.paymentFailureCounter = Counter.builder("payments.processed.failed")
            .description("Total failed payment transactions")
            .tag("env", "production")
            .register(meterRegistry);

        // 2. Timer: Measures execution latency and throughput distribution
        this.paymentProcessingTimer = Timer.builder("payments.processing.latency")
            .description("Latency distribution for payment processing")
            .publishPercentiles(0.5, 0.95, 0.99) // p50, p95, p99 latency SLA tracking
            .register(meterRegistry);
    }

    public PaymentResponse processPayment(PaymentRequest req) {
        return paymentProcessingTimer.record(() -> {
            try {
                PaymentResponse response = gateway.execute(req);
                paymentSuccessCounter.increment();
                return response;
            } catch (Exception ex) {
                paymentFailureCounter.increment();
                throw ex;
            }
        });
    }
}
```

---

> **Next Chapter**: [15 Web, REST & Reactive APIs (Spring WebFlux)](15_Spring_Web_REST_APIs.md)

