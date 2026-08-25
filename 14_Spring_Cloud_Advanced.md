# Advanced Spring Cloud, Gateway & Resilience4j

> **Navigation**: [Master Index](README.md) | [Previous: Advanced Security](13_Spring_Security_Advanced.md) | [Next: Microservices Architecture](15_Microservices_Architecture.md)

---

## 1. Spring Cloud Gateway Architecture

```
+-----------------------------------------------------------------------------------+
|                           SPRING CLOUD GATEWAY                                    |
+-----------------------------------------------------------------------------------+
|  [ Client Request ]                                                               |
|         |                                                                         |
|         v                                                                         |
|  [ Netty EventLoop ] ---> [ Route Predicate Handler ] (Matches Path, Host, Method)|
|                                  |                                                |
|                                  v                                                |
|                     [ Gateway Filter Chain ]                                      |
|                     1. Global Authentication Filter (JWT Validation & Token Relay)|
|                     2. Distributed Rate Limiter Filter (Redis Token Bucket)       |
|                     3. Request Header Transformation (X-User-Id, X-Trace-Id)      |
|                                  |                                                |
|                                  v                                                |
|                     [ Downstream Microservice ]                                   |
+-----------------------------------------------------------------------------------+
```

---

### Q1. How do you implement a Custom Global Authentication Filter in Spring Cloud Gateway?
**Answer:**

```java
@Component
public class AuthenticationGatewayFilterFactory extends AbstractGatewayFilterFactory<AuthenticationGatewayFilterFactory.Config> {

    private final JwtUtils jwtUtils;

    public AuthenticationGatewayFilterFactory(JwtUtils jwtUtils) {
        super(Config.class);
        this.jwtUtils = jwtUtils;
    }

    public static class Config {}

    @Override
    public GatewayFilter apply(Config config) {
        return (exchange, chain) -> {
            ServerHttpRequest request = exchange.getRequest();

            if (!request.getHeaders().containsKey(HttpHeaders.AUTHORIZATION)) {
                return onError(exchange, HttpStatus.UNAUTHORIZED, "Missing Authorization Header");
            }

            String authHeader = request.getHeaders().getFirst(HttpHeaders.AUTHORIZATION);
            if (authHeader == null || !authHeader.startsWith("Bearer ")) {
                return onError(exchange, HttpStatus.UNAUTHORIZED, "Invalid Bearer Token");
            }

            String token = authHeader.substring(7);
            if (!jwtUtils.validateToken(token)) {
                return onError(exchange, HttpStatus.UNAUTHORIZED, "Expired or Invalid Token");
            }

            // Extract claims and mutate request downstream with authenticated user metadata
            Claims claims = jwtUtils.extractAllClaims(token);
            ServerHttpRequest mutatedRequest = request.mutate()
                .header("X-User-Id", claims.getSubject())
                .header("X-User-Roles", String.join(",", claims.get("roles", List.class)))
                .build();

            return chain.filter(exchange.mutate().request(mutatedRequest).build());
        };
    }

    private Mono<Void> onError(ServerWebExchange exchange, HttpStatus status, String message) {
        ServerHttpResponse response = exchange.getResponse();
        response.setStatusCode(status);
        response.getHeaders().setContentType(MediaType.APPLICATION_JSON);
        byte[] bytes = String.format("{\"error\": \"%s\"}", message).getBytes(StandardCharsets.UTF_8);
        return response.writeWith(Mono.just(response.bufferFactory().wrap(bytes)));
    }
}
```

---

## 2. Resilience4j Circuit Breaker & Fault Tolerance

```
+-----------------------------------------------------------------------------------+
|                        RESILIENCE4J CIRCUIT BREAKER STATES                        |
+-----------------------------------------------------------------------------------+
|                              +------------------+                                 |
|                              |      CLOSED      |  <--- Normal operation (Passes all calls)
|                              +------------------+                                 |
|                                 |            ^                                    |
|   Failure Rate > Threshold (50%)|            | Success Rate >= 80%                |
|                                 v            | in Half-Open probe                 |
|                              +------------------+                                 |
|                              |       OPEN       |  <--- Fast-Fail (Returns Fallback immediately)
|                              +------------------+                                 |
|                                 |                                                 |
|          waitDurationInOpenState| (10 seconds)                                    |
|                                 v                                                 |
|                              +------------------+                                 |
|                              |    HALF_OPEN     |  <--- Permits test calls to probe downstream health
|                              +------------------+                                 |
+-----------------------------------------------------------------------------------+
```

---

### Q2. Production Resilience4j Configuration in `application.yml`.
**Answer:**

```yaml
resilience4j:
  circuitbreaker:
    instances:
      paymentService:
        slidingWindowType: COUNT_BASED
        slidingWindowSize: 20              # Measure last 20 requests
        minimumNumberOfCalls: 10          # Require at least 10 calls before calculating failure rate
        failureRateThreshold: 50.0        # Trip to OPEN if 50% of calls fail
        slowCallRateThreshold: 50.0       # Trip to OPEN if 50% of calls take > 2s
        slowCallDurationThreshold: 2000ms
        waitDurationInOpenState: 10000ms  # Stay in OPEN for 10s before probing HALF_OPEN
        permittedNumberOfCallsInHalfOpenState: 5
        automaticTransitionFromOpenToHalfOpenEnabled: true
  retry:
    instances:
      paymentService:
        maxAttempts: 3
        waitDuration: 500ms
        enableExponentialBackoff: true
        exponentialBackoffMultiplier: 2
        retryExceptions:
          - java.io.IOException
          - java.util.concurrent.TimeoutException
        ignoreExceptions:
          - com.exception.BusinessValidationException
```

```java
// Service using Circuit Breaker & Fallback
@Service
public class OrderCheckoutService {

    @CircuitBreaker(name = "paymentService", fallbackMethod = "paymentFallback")
    @Retry(name = "paymentService")
    public PaymentResponse chargeOrder(Order order) {
        return paymentClient.executeTransaction(order);
    }

    // Fallback must have same return type and append Throwable as final parameter
    public PaymentResponse paymentFallback(Order order, Throwable ex) {
        logger.error("Payment Service unavailable. Routing to async pending queue. Reason: {}", ex.getMessage());
        return PaymentResponse.deferred(order.getId());
    }
}
```

---

> **Next Chapter**: [15 Microservices Architecture](15_Microservices_Architecture.md)
