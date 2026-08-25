# Web, REST APIs & Reactive Programming Deep Dive

> **Navigation**: [Master Index](README.md) | [Previous: Spring Core & Boot](04_Spring_Core_Boot_Internals.md) | [Next: Persistence & Messaging](06_Persistence_Transactions_Messaging.md)

---

## 1. Spring MVC Architecture & Request Lifecycle

```
+-----------------------------------------------------------------------------------+
|                        SPRING MVC REQUEST LIFECYCLE                               |
+-----------------------------------------------------------------------------------+
|  1. Incoming HTTP Request                                                         |
|         |                                                                         |
|         v                                                                         |
|  2. [ DispatcherServlet ] (Front Controller)                                      |
|         |                                                                         |
|         +---> 3. [ HandlerMapping ] (Finds matching @RestController / Method)     |
|         |                                                                         |
|         +---> 4. [ HandlerInterceptor.preHandle() ] (Auth, Logging, Trace IDs)    |
|         |                                                                         |
|         +---> 5. [ HandlerAdapter ] (Invokes Controller Method)                   |
|         |           |                                                             |
|         |           +---> 6. [ HttpMessageConverter ] (Jackson JSON -> Java DTO)  |
|         |           |                                                             |
|         |           +---> 7. [ Jakarta Validator ] (@Valid validation)            |
|         |           |                                                             |
|         |           v                                                             |
|         |     [ @RestController Method Execution ]                                |
|         |           |                                                             |
|         +<----------+                                                             |
|         |                                                                         |
|         +---> 8. [ HandlerInterceptor.postHandle() ]                             |
|         |                                                                         |
|         +---> 9. [ HttpMessageConverter ] (Java Response DTO -> JSON payload)     |
|         |                                                                         |
|         +---> 10. [ HandlerInterceptor.afterCompletion() ] (Cleans MDC / ThreadLocals) |
|         |                                                                         |
|         v                                                                         |
|  11. Outgoing HTTP Response to Client                                             |
+-----------------------------------------------------------------------------------+
```

---

## 2. RESTful API Design & RFC 7807 Error Handling

### Q1. Explain REST Maturity Levels (Richardson Maturity Model).
**Answer:**
1. **Level 0 (The Swamp of POX)**: Single endpoint, single HTTP method (e.g., SOAP or XML-RPC posting all requests to `/api/service`).
2. **Level 1 (Resources)**: Individual endpoints for distinct resources (`/api/orders`, `/api/customers`), but typically only using POST.
3. **Level 2 (HTTP Verbs & Status Codes)**: Correct use of standard HTTP verbs (GET, POST, PUT, DELETE) and HTTP status codes (200, 201, 204, 400, 404, 409).
4. **Level 3 (Hypermedia Controls / HATEOAS)**: Responses include dynamic hypermedia links informing clients what actions can be performed next (Spring HATEOAS / HAL format).

---

### Q2. How do you implement production-grade Global Exception Handling using RFC 7807 `ProblemDetail`?
**Answer:**
Spring Boot 3 / Spring Framework 6 introduces full native support for **RFC 7807 Problem Details for HTTP APIs**.

```java
@RestControllerAdvice
public class GlobalExceptionHandler extends ResponseEntityExceptionHandler {

    @ExceptionHandler(OrderNotFoundException.class)
    public ProblemDetail handleOrderNotFound(OrderNotFoundException ex, HttpServletRequest request) {
        ProblemDetail problem = ProblemDetail.forStatusAndDetail(HttpStatus.NOT_FOUND, ex.getMessage());
        problem.setTitle("Order Not Found");
        problem.setType(URI.create("https://api.myapp.com/errors/not-found"));
        problem.setProperty("timestamp", Instant.now());
        problem.setProperty("orderId", ex.getOrderId());
        return problem;
    }

    @Override
    protected ResponseEntity<Object> handleMethodArgumentNotValid(
            MethodArgumentNotValidException ex, HttpHeaders headers, HttpStatusCode status, WebRequest request) {
        
        ProblemDetail problem = ProblemDetail.forStatus(HttpStatus.BAD_REQUEST);
        problem.setTitle("Validation Failed");
        
        Map<String, String> invalidFields = ex.getBindingResult().getFieldErrors().stream()
            .collect(Collectors.toMap(
                FieldError::getField,
                error -> (error.getDefaultMessage() != null) ? error.getDefaultMessage() : "Invalid value",
                (first, second) -> first
            ));
        
        problem.setProperty("invalidFields", invalidFields);
        return ResponseEntity.badRequest().body(problem);
    }
}
```

---

## 3. Input Validation with Jakarta Validation (JSR-380)

### Q3. How do you write a Custom Constraint Validator in Spring Boot?
**Answer:**

```java
// 1. Define Custom Annotation
@Target({ ElementType.FIELD, ElementType.PARAMETER })
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = ValidPhoneNumberValidator.class)
@Documented
public @interface ValidPhoneNumber {
    String message() default "Invalid international phone number format (+E.164)";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}

// 2. Define Validator Implementation
public class ValidPhoneNumberValidator implements ConstraintValidator<ValidPhoneNumber, String> {
    private static final Pattern PHONE_PATTERN = Pattern.compile("^\\+[1-9]\\d{1,14}$");

    @Override
    public boolean isValid(String phone, ConstraintValidatorContext context) {
        if (phone == null) return true; // Let @NotNull handle nullability
        return PHONE_PATTERN.matcher(phone).matches();
    }
}
```

---

## 4. Modern HTTP Clients in Spring Boot 3

### Q4. Compare `RestClient` (Boot 3) vs. `WebClient` vs. `RestTemplate`.
**Answer:**

| Feature | `RestClient` (Spring 6 / Boot 3) | `WebClient` (Spring WebFlux) | `RestTemplate` (Legacy) |
| :--- | :--- | :--- | :--- |
| **Model** | Synchronous, Blocking (Fluent) | Reactive, Non-blocking (Asynchronous) | Synchronous, Blocking (Template) |
| **Status** | **Recommended Standard** for sync apps | **Recommended Standard** for reactive apps | Maintenance Mode |
| **Dependencies** | Included in `spring-boot-starter-web` | Requires `spring-boot-starter-webflux` | Included in `spring-boot-starter-web` |
| **Virtual Threads Support** | Fully unmounts Virtual Threads seamlessly | Functional, but reactive paradigm is redundant | Supports, but dated API |

```java
// Modern Spring 6 RestClient Example
@Service
public class PaymentGatewayClient {
    private final RestClient restClient;

    public PaymentGatewayClient(RestClient.Builder builder, @Value("${gateway.base-url}") String baseUrl) {
        this.restClient = builder
            .baseUrl(baseUrl)
            .defaultHeader(HttpHeaders.CONTENT_TYPE, MediaType.APPLICATION_JSON_VALUE)
            .requestFactory(new JdkClientHttpRequestFactory()) // Java 11+ HttpClient
            .build();
    }

    public PaymentResponse charge(PaymentRequest request) {
        return restClient.post()
            .uri("/v1/payments")
            .body(request)
            .retrieve()
            .onStatus(HttpStatusCode::is4xxClientError, (req, res) -> {
                throw new PaymentClientException("Payment failed: " + res.getStatusCode());
            })
            .body(PaymentResponse.class);
    }
}
```

---

### Q5. Declarative HTTP Interfaces (`@HttpExchange` in Spring 6).
**Answer:**
Spring 6 allows defining HTTP client contracts as Java interfaces without writing boilerplate HTTP code (similar to Spring Cloud OpenFeign, but built directly into Spring Framework core).

```java
// 1. Declare Interface Contract
@HttpExchange("/api/users")
public interface UserClient {
    @GetExchange("/{id}")
    UserDTO getUserById(@PathVariable("id") Long id);

    @PostExchange
    UserDTO createUser(@RequestBody CreateUserRequest request);
}

// 2. Register Client Bean
@Configuration
public class ClientConfig {
    @Bean
    public UserClient userClient(RestClient.Builder builder) {
        RestClient restClient = builder.baseUrl("https://users.service.internal").build();
        RestClientAdapter adapter = RestClientAdapter.create(restClient);
        HttpServiceProxyFactory factory = HttpServiceProxyFactory.builderFor(adapter).build();
        return factory.createClient(UserClient.class);
    }
}
```

---

## 5. Reactive Programming Foundations (Spring WebFlux)

### Q6. When should you choose Spring WebFlux over Spring MVC?
**Answer:**

```
+------------------------------------+    +------------------------------------+
|        Spring MVC (Servlet)        |    |       Spring WebFlux (Netty)       |
+------------------------------------+    +------------------------------------+
|  Thread-per-Request model          |    |  Event Loop (Few worker threads)   |
|  Blocking I/O                      |    |  Non-blocking Asynchronous I/O     |
|  Large OS stack memory (~1MB/thd)  |    |  Backpressure stream control       |
|  * BEST for CPU-heavy / JDBC apps  |    |  * BEST for Streaming, Gateways,   |
|  * Greatly enhanced by Virtual Thd |    |    Microservice Aggregators        |
+------------------------------------+    +------------------------------------+
```

- **Choose Spring MVC** when:
  - You use relational databases with blocking JDBC / JPA / Hibernate drivers.
  - Team is experienced with imperative debugging and stack traces.
  - Using Java 21 Virtual Threads to handle massive concurrent I/O simply.
- **Choose Spring WebFlux** when:
  - Building high-throughput API Gateways, streaming proxies, or WebSocket aggregators.
  - Using fully reactive drivers end-to-end (e.g., R2DBC, Reactive Redis, Reactive Mongo).
  - Memory footprint per concurrent connection must be minimal.

---

> **Next Chapter**: [06 Persistence, Transactions & Messaging](06_Persistence_Transactions_Messaging.md)
