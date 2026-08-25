# 15. Web, REST, Spring Validation & Reactive APIs (Spring WebFlux)

> **Navigation**: [Master Index](README.md) | [Previous: Spring Boot 3 Internals](14_Spring_Boot_3_Internals_Actuator.md) | [Next: Spring Security & OAuth2](16_Spring_Security_6_OAuth2.md)

---

## 📌 Chapter Overview
This module covers the **Spring MVC request lifecycle**, REST API design standards (Richardson Maturity Model), **Spring Validation (Built-in + Custom Constraint Annotations)**, global error handling with **RFC 7807 `ProblemDetail`**, modern HTTP clients (**`RestClient`**, Declarative `@HttpExchange`), and **Spring WebFlux**.

---

## 1. Spring MVC Architecture & Request Lifecycle

```
                                [ Incoming HTTP Request ]
                                            |
                                            v
                                 [ DispatcherServlet ]
                                            |
                         +------------------+------------------+
                         |                                     |
                         v                                     v
                [ HandlerMapping ]                    [ HandlerExecutionChain ]
        (Finds Controller matching URL)               - Interceptor preHandle()
                         |                            - Controller Execution
                         v                            - Interceptor postHandle()
                 [ HandlerAdapter ]                   - Interceptor afterCompletion()
                         |
                         v
                [ @RestController ]
             (Executes business logic)
                         |
                         v
             [ HttpMessageConverter ]
             (Jackson serializes Java DTO to JSON)
                         |
                         v
                 [ HTTP 200 JSON ]
```

---

## 2. Spring Validation: Built-in Annotations & `@Valid` vs. `@Validated`

```
+-----------------------------------------------------------------------------------+
|                        JAKARTA BEAN VALIDATION IN SPRING                          |
+-----------------------------------------------------------------------------------+
|  1. @NotNull:  Value must not be null (allows empty String "" or empty List [])   |
|  2. @NotEmpty: Value must not be null AND size/length must be > 0                 |
|  3. @NotBlank: Value must not be null, length > 0, and not all whitespace ("   ") |
|  4. @Size:     Checks min/max boundary on Strings, Collections, Maps, Arrays      |
|  5. @Pattern:  Validates string against RegEx (e.g. ^[A-Z]{5}[0-9]{4}[A-Z]{1}$)   |
+-----------------------------------------------------------------------------------+
```

### Q1. `@Valid` vs. `@Validated`: What is the difference?
**Answer:**

| Feature | `@Valid` (Jakarta Validation / JSR-380) | `@Validated` (Spring Framework) |
| :--- | :--- | :--- |
| **Origin** | Standard Java Spec (`jakarta.validation.Valid`) | Spring specific (`org.springframework.validation.annotation.Validated`)|
| **Placement** | Method parameters, fields, nested object properties | Class level (Service/Controller) & method parameters |
| **Nested Validation**| **Yes** (Required on nested child DTOs) | No (Cannot be placed on member fields for nested cascades) |
| **Validation Groups**| No | **Yes** (Supports conditional groups e.g. `@Validated(OnCreate.class)`)|
| **Service Validation**| Requires `@Validated` on the Service class | Enables AOP method validation interceptor on `@Service` beans |

---

## 3. How to Create Custom Validation Annotations

```
 [ DTO Field: @ValidTaxId ] ---> Hibernate Validator Engine ---> TaxIdValidator.isValid()
                                                                         |
                                                 +-----------------------+-----------------------+
                                                 | (true)                                        | (false)
                                                 v                                               v
                                        Request Proceeds                        Validation Exception (400 Bad Request)
```

### Q2. How do you implement a Custom Validation Annotation in Spring Boot?
**Answer:**

#### Step 1: Define the Custom Constraint Annotation
```java
package com.example.validation;

import jakarta.validation.Constraint;
import jakarta.validation.Payload;
import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Target({ ElementType.FIELD, ElementType.PARAMETER })
@Retention(RetentionPolicy.RUNTIME)
@Documented
// Links annotation to the validator implementation!
@Constraint(validatedBy = TaxIdValidator.class)
public @interface ValidTaxId {

    String message() default "Invalid Tax Identification Number format (Expected: ABCDE1234F)";

    Class<?>[] groups() default {};

    Class<? extends Payload>[] payload() default {};
}
```

#### Step 2: Implement the `ConstraintValidator`
```java
package com.example.validation;

import jakarta.validation.ConstraintValidator;
import jakarta.validation.ConstraintValidatorContext;
import java.util.regex.Pattern;

public class TaxIdValidator implements ConstraintValidator<ValidTaxId, String> {

    private static final Pattern TAX_ID_REGEX = Pattern.compile("^[A-Z]{5}[0-9]{4}[A-Z]{1}$");

    @Override
    public boolean isValid(String value, ConstraintValidatorContext context) {
        if (value == null) {
            return true; // Let @NotNull handle null checks if required
        }
        return TAX_ID_REGEX.matcher(value).matches();
    }
}
```

#### Step 3: Usage on Request DTO
```java
public record CreateVendorRequest(
    @NotBlank(message = "Vendor name cannot be blank")
    String vendorName,

    @ValidTaxId
    String taxId,

    @NotNull
    @Valid // Cascades validation into nested objects!
    AddressDto address
) {}
```

---

## 4. REST API Standards & RFC 7807 `ProblemDetail`

### Q3. How do you implement production Global Exception Handling using RFC 7807?
**Answer:**
Spring Boot 3 natively supports **RFC 7807 (Problem Details for HTTP APIs)** via the `ProblemDetail` class.

```java
@RestControllerAdvice
public class GlobalExceptionHandler extends ResponseEntityExceptionHandler {

    // 1. Handle Business Entity Not Found
    @ExceptionHandler(ResourceNotFoundException.class)
    public ProblemDetail handleNotFound(ResourceNotFoundException ex) {
        ProblemDetail problem = ProblemDetail.forStatusAndDetail(HttpStatus.NOT_FOUND, ex.getMessage());
        problem.setTitle("Resource Not Found");
        problem.setType(URI.create("https://api.example.com/errors/not-found"));
        problem.setProperty("timestamp", Instant.now());
        return problem;
    }

    // 2. Handle Jakarta DTO Validation Failures
    @Override
    protected ResponseEntity<Object> handleMethodArgumentNotValid(
            MethodArgumentNotValidException ex, HttpHeaders headers, HttpStatusCode status, WebRequest request) {
        
        ProblemDetail problem = ProblemDetail.forStatusAndDetail(HttpStatus.BAD_REQUEST, "Validation failed for request payload");
        problem.setTitle("Invalid Request Content");

        Map<String, String> fieldErrors = new HashMap<>();
        for (FieldError fe : ex.getBindingResult().getFieldErrors()) {
            fieldErrors.put(fe.getField(), fe.getDefaultMessage());
        }
        problem.setProperty("invalidFields", fieldErrors);

        return ResponseEntity.status(HttpStatus.BAD_REQUEST).body(problem);
    }
}
```

---

## 5. Modern HTTP Clients in Spring Boot 3

```
+-------------------------------------------------------------------------------+
|                       SPRING BOOT 3 HTTP CLIENT COMPARISON                    |
+-------------------------------------------------------------------------------+
|  1. RestClient (Spring 6 / Boot 3) -> Modern Fluent Synchronous (Recommended) |
|  2. WebClient                      -> Reactive Non-Blocking (WebFlux)         |
|  3. @HttpExchange                  -> Declarative Interface Proxy (Spring 6)  |
|  4. RestTemplate                   -> Legacy (Maintenance Mode)               |
+-------------------------------------------------------------------------------+
```

### Q4. Compare `RestClient` (Boot 3) vs. Declarative `@HttpExchange`.
**Answer:**

#### 1. Modern Fluent `RestClient` (Spring Boot 3):
```java
@Service
public class PaymentClientService {
    private final RestClient restClient;

    public PaymentClientService(RestClient.Builder builder) {
        this.restClient = builder
            .baseUrl("https://api.payment-gateway.com")
            .defaultHeader("Authorization", "Bearer secret-token")
            .build();
    }

    public PaymentResponse initiatePayment(PaymentRequest req) {
        return restClient.post()
            .uri("/v1/charges")
            .contentType(MediaType.APPLICATION_JSON)
            .body(req)
            .retrieve()
            .body(PaymentResponse.class);
    }
}
```

#### 2. Declarative HTTP Interfaces (`@HttpExchange` in Spring 6):
```java
// 1. Declare the HTTP Interface contract
@HttpExchange("/users")
public interface UserHttpClient {
    @GetExchange("/{id}")
    UserDto getUserById(@PathVariable("id") Long id);

    @PostExchange
    UserDto createUser(@RequestBody CreateUserRequest request);
}

// 2. Register Client Proxy Bean
@Configuration
public class HttpClientConfig {
    @Bean
    public UserHttpClient userHttpClient(RestClient.Builder builder) {
        RestClient restClient = builder.baseUrl("https://users.service.internal").build();
        HttpServiceProxyFactory factory = HttpServiceProxyFactory
            .builderFor(RestClientAdapter.create(restClient)).build();
        return factory.createClient(UserHttpClient.class);
    }
}
```

---

## 6. Reactive Programming: Spring WebFlux vs. Spring MVC

### Q5. When should you choose Spring WebFlux over Spring MVC with Virtual Threads?
**Answer:**

| Architectural Criteria | Spring MVC + Java 21 Virtual Threads | Spring WebFlux (Project Reactor) |
| :--- | :--- | :--- |
| **Programming Paradigm**| Standard Imperative (Sequential code) | **Functional / Reactive Streams** (`Mono`, `Flux`) |
| **Concurrency Model** | Millions of Virtual Threads on Carrier Threads | **Non-blocking Event Loop** (Netty) |
| **Backpressure Support**| Relies on TCP/OS socket flow control | **Native Reactive Backpressure** |
| **Database Driver Support**| Full JDBC / Hibernate JPA support | Requires reactive drivers (**R2DBC**, Mongo Reactive) |
| **Ideal Use Case** | 90%+ enterprise REST CRUD microservices | **Server-Sent Events (SSE), Streaming APIs, WebSockets** |

---

> **Next Chapter**: [16 Spring Security 6, JWT & OAuth2 Architecture](16_Spring_Security_6_OAuth2.md)
