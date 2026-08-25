# 28. Spring Cloud Gateway, Load Balancing & Resilience4j

> **Navigation**: [Master Index](README.md) | [Previous: Distributed Transactions](27_Distributed_Transactions_Saga_Outbox.md) | [Next: Observability & Tracing](29_Observability_Tracing_Testing.md)

---

## 📌 Chapter Overview
This module explores **Spring Cloud Gateway (Netty-based reactive gateway)** architecture, Custom Global Gateway Filters, **Load Balancing Mechanics (L4 vs. L7, Algorithms, Consistent Hashing, Client-side vs. Server-side)**, Redis Token Bucket Rate Limiting, and the complete **Resilience4j Fault Tolerance Suite** (Circuit Breakers, Bulkheads, Rate Limiters, Timeouts).

---

## 1. Spring Cloud Gateway Architecture

```
                                [ Client Request ]
                                        |
                                        v
                            [ Spring Cloud Gateway ]
                                        |
                +-----------------------+-----------------------+
                |                                               |
                v                                               v
        [ Route Predicates ]                           [ Filter Chain ]
    - Path=/api/v1/orders/**                       - GlobalAuthFilter (Validate JWT)
    - Method=POST                                  - RequestRateLimiter (Redis Token Bucket)
    - Header=X-Tenant-Id                           - Resilience4jCircuitBreaker
                |                                               |
                +-----------------------+-----------------------+
                                        |
                                        v
                            [ Downstream Microservice ]
```

### Q1. How do you implement a Custom Global Authentication Filter in Spring Cloud Gateway?
**Answer:**

```java
@Component
public class JwtAuthenticationGlobalFilter implements GlobalFilter, Ordered {

    @Autowired private JwtValidator jwtValidator;

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        ServerHttpRequest request = exchange.getRequest();

        // 1. Bypass public authentication endpoints
        if (request.getURI().getPath().startsWith("/api/v1/auth/")) {
            return chain.filter(exchange);
        }

        // 2. Extract Authorization Bearer header
        String authHeader = request.getHeaders().getFirst(HttpHeaders.AUTHORIZATION);
        if (authHeader == null || !authHeader.startsWith("Bearer ")) {
            exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
            return exchange.getResponse().setComplete();
        }

        String token = authHeader.substring(7);
        try {
            Claims claims = jwtValidator.validateAndExtractClaims(token);
            // 3. Mutate request headers to forward user info downstream
            ServerHttpRequest mutatedRequest = request.mutate()
                .header("X-User-Id", claims.getSubject())
                .header("X-User-Roles", String.join(",", claims.get("roles", List.class)))
                .build();

            return chain.filter(exchange.mutate().request(mutatedRequest).build());

        } catch (JwtException ex) {
            exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
            return exchange.getResponse().setComplete();
        }
    }

    @Override public int getOrder() { return -100; } // High filter priority
}
```

---

## 2. Load Balancing Architecture: L4 vs. L7 & Algorithms

```
+-----------------------------------------------------------------------------------+
|                        LOAD BALANCING ARCHITECTURE COMPARISON                     |
+-----------------------------------------------------------------------------------+
|  1. Layer 4 (Transport: TCP / UDP) -> AWS NLB, HAProxy                            |
|     - Fast packet routing without inspecting HTTP payloads or headers             |
|                                                                                   |
|  2. Layer 7 (Application: HTTP / HTTPS / gRPC) -> AWS ALB, Nginx, Envoy, Gateway   |
|     - Content-based routing via Path, Headers, Cookies, HTTP Verbs                |
|                                                                                   |
|  3. Client-Side vs Server-Side:                                                   |
|     - Server-Side: Client -> [ Central Load Balancer (ALB / Nginx) ] -> Pods      |
|     - Client-Side: [ Spring Cloud LoadBalancer ] -> Queries Eureka -> Calls Pod   |
+-----------------------------------------------------------------------------------+
```

### Q2. Detail the Core Load Balancing Algorithms & Consistent Hashing.
**Answer:**

| Algorithm | Mechanism | Best Use Case |
| :--- | :--- | :--- |
| **Round Robin** | Routes requests sequentially ($1 \rightarrow 2 \rightarrow 3 \rightarrow 1$) | Homogeneous servers with equal specs |
| **Weighted Round Robin** | Assigns higher request quotas to higher-capacity servers | Mixed cluster hardware (e.g. 8-core vs 16-core) |
| **Least Connections** | Routes to the server with fewest active TCP/HTTP connections | Long-running transactions / WebSocket connections |
| **IP Hash / Sticky Sessions**| Hashes client IP to always route same client to same pod | Stateful legacy session-based applications |
| **Consistent Hashing** | Maps nodes and request keys onto a $2^{32}-1$ hash ring with virtual nodes | **Distributed Caching (Redis), Distributed NoSQL (Cassandra), State partitioning** |

```
                 CONSISTENT HASHING RING (0 to 2^32 - 1)
                           [ Node A (v1) ]
                            /           \
               [ Key 1 ]   /             \   [ Node B (v1) ]
                 \        /               \       /
                  v      /                 \     v
             [ Node C (v1) ] ----------- [ Key 2 ]
  * Adding/Removing a node remaps only K/N keys, preventing total cache invalidation!
```

---

## 3. Distributed Rate Limiting with Redis Token Bucket in Gateway

### Q3. How do you configure Spring Cloud Gateway `RequestRateLimiter`?
**Answer:**

```java
@Configuration
public class RateLimiterConfig {

    // Rate limit per Authenticated User ID (or fallback to Client IP)
    @Bean
    public KeyResolver userKeyResolver() {
        return exchange -> {
            String userId = exchange.getRequest().getHeaders().getFirst("X-User-Id");
            if (userId != null) {
                return Mono.just(userId);
            }
            return Mono.just(Objects.requireNonNull(
                exchange.getRequest().getRemoteAddress()).getAddress().getHostAddress()
            );
        };
    }
}
```

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: order-service-route
          uri: lb://order-service
          predicates:
            - Path=/api/v1/orders/**
          filters:
            - name: RequestRateLimiter
              args:
                redis-rate-limiter.replenishRate: 10 # 10 tokens added per second (steady throughput)
                redis-rate-limiter.burstCapacity: 20 # Max 20 tokens (burst capacity)
                key-resolver: "#{@userKeyResolver}"
```

---

## 4. Resilience4j Circuit Breaker Architecture

```
+-----------------------------------------------------------------------------------+
|                        RESILIENCE4J CIRCUIT BREAKER STATE MACHINE                 |
+-----------------------------------------------------------------------------------+
|                                                                                   |
|  +----------------+      Failure Rate > 50%      +----------------+               |
|  |     CLOSED     | ---------------------------> |      OPEN      |               |
|  | (Normal Flow)  | <--------------------------- | (Fails Fast)   |               |
|  +----------------+     Success Rate > Threshold +----------------+               |
|          ^                                               |                        |
|          |                                               | Wait Duration Expired  |
|          |                                               v (e.g., 10 seconds)     |
|          |                                       +----------------+               |
|          +-------------------------------------- |   HALF-OPEN    |               |
|                                                  | (Tests N Calls)|               |
|                                                  +----------------+               |
+-----------------------------------------------------------------------------------+
```

### Q4. Production Resilience4j Configuration (`application.yml`).
**Answer:**

```yaml
resilience4j:
  circuitbreaker:
    instances:
      paymentService:
        sliding-window-type: COUNT_BASED
        sliding-window-size: 20              # Evaluates last 20 requests
        minimum-number-of-calls: 10          # Requires at least 10 calls before calculating failure rate
        failure-rate-threshold: 50           # Opens circuit if >= 50% requests fail
        slow-call-rate-threshold: 80         # Opens circuit if >= 80% calls take > 2s
        slow-call-duration-threshold: 2000ms
        wait-duration-in-open-state: 10000ms # Stays OPEN for 10s before transitioning to HALF_OPEN
        permitted-number-of-calls-in-half-open-state: 5 # Allows 5 trial requests in HALF_OPEN
        automatic-transition-from-open-to-half-open-enabled: true

  timelimiter:
    instances:
      paymentService:
        timeout-duration: 2500ms             # Hard timeout for downstream call

  bulkhead:
    instances:
      paymentService:
        max-concurrent-calls: 25             # Limits simultaneous in-flight calls to payment service
```

---

> **Next Chapter**: [29 Observability, Distributed Tracing & Integration Testing](29_Observability_Tracing_Testing.md)
