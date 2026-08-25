# 29. Observability, Distributed Tracing & Integration Testing

> **Navigation**: [Master Index](README.md) | [Previous: Spring Cloud Gateway](28_Spring_Cloud_Gateway_Resilience.md) | [Next: Docker & Kubernetes](30_Cloud_Docker_Kubernetes.md)

---

## 📌 Chapter Overview
This module covers **Distributed Tracing with OpenTelemetry (OTel)**, Micrometer Tracing in Spring Boot 3, structured MDC logging, Prometheus/Grafana metric alerts, and **Integration Testing with Testcontainers**.

---

## 1. Distributed Tracing: OpenTelemetry & Micrometer Tracing

```
+-----------------------------------------------------------------------------------+
|                        DISTRIBUTED TRACE CONTEXT PROPAGATION                      |
+-----------------------------------------------------------------------------------+
|  [ Trace ID: 4bf92f3577b34da6a3ce929d0e0e4736 ] (Shared across entire request)    |
|                                                                                   |
|  1. API Gateway:        [ Span A: Gateway Auth & Routing ] (Span ID: 00f067aa0ba) |
|         |                                                                         |
|         | HTTP Header: traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f06...  |
|         v                                                                         |
|  2. Order Service:      [ Span B: Process Order & DB Write ] (Span ID: 5fb3f0a91) |
|         |                                                                         |
|         | Kafka Record Header: traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736...|
|         v                                                                         |
|  3. Payment Service:    [ Span C: Payment Authorization ] (Span ID: a2c18d990)    |
+-----------------------------------------------------------------------------------+
```

### Q1. How do you implement Distributed Tracing with MDC Logging in Spring Boot 3?
**Answer:**
In Spring Boot 3, Spring Cloud Sleuth was replaced by **Micrometer Tracing with OpenTelemetry (OTel)**.

```xml
<!-- pom.xml Dependencies -->
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-tracing-bridge-otel</artifactId>
</dependency>
<dependency>
    <groupId>io.opentelemetry</groupId>
    <artifactId>opentelemetry-exporter-otlp</artifactId>
</dependency>
```

#### Logback Pattern (`logback-spring.xml`):
```xml
<property name="LOG_PATTERN"
    value="%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level [TraceId: %X{traceId:-}, SpanId: %X{spanId:-}] %logger{36} - %msg%n"/>
```

---

## 2. Integration Testing with Testcontainers & `@ServiceConnection`

### Q2. Why is H2 an anti-pattern for integration tests? How does Testcontainers solve it?
**Answer:**
- **Why H2 Fails**: H2 does not support PostgreSQL/MySQL-specific features (e.g., `JSONB`, CTEs, `FOR UPDATE NOWAIT`, Triggers, Sequence generation, full locking semantics). Tests pass on H2 in CI but fail in production on real databases!
- **Testcontainers Solution**: Spins up real, lightweight Docker containers (PostgreSQL, Kafka, Redis) for the duration of the test.
- **Spring Boot 3.1+ `@ServiceConnection`**: Automatically configures all Spring datasource, Kafka, and Redis connection properties without manual `@DynamicPropertySource` boilerplate!

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@Testcontainers
class OrderServiceIntegrationTests {

    @Container
    @ServiceConnection // Automatically sets spring.datasource.url/username/password!
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine");

    @Container
    @ServiceConnection // Automatically sets spring.kafka.bootstrap-servers!
    static KafkaContainer kafka = new KafkaContainer(DockerImageName.parse("confluentinc/cp-kafka:7.5.0"));

    @Autowired private OrderRepository orderRepo;
    @Autowired private TestRestTemplate restTemplate;

    @Test
    void shouldCreateOrderAndPersistToRealPostgres() {
        OrderRequest request = new OrderRequest(100L, 50.0);
        ResponseEntity<OrderDto> response = restTemplate.postForEntity("/api/v1/orders", request, OrderDto.class);

        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.CREATED);
        assertThat(orderRepo.findById(response.getBody().id())).isPresent();
    }
}
```

---

> **Next Chapter**: [30 Cloud-Native Deployments: Docker & Kubernetes](30_Cloud_Docker_Kubernetes.md)

