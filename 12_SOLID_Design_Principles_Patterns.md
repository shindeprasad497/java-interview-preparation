# SOLID Design Principles & GoF Patterns in Spring Boot

> **Navigation**: [Master Index](README.md) | [Previous: Strings Deep Dive](11_Strings_Deep_Dive.md) | [Next: Advanced Spring Security](13_Spring_Security_Advanced.md)

---

## 1. SOLID Principles Applied in Spring Boot

```
+-----------------------------------------------------------------------------------+
|                            SOLID PRINCIPLES MATRIX                                |
+-----------------------------------------------------------------------------------+
|  S - Single Responsibility : A class should have one, and only one, reason to change|
|  O - Open / Closed         : Open for extension, but closed for modification      |
|  L - Liskov Substitution   : Subtypes must be substitutable for their base types  |
|  I - Interface Segregation : Clients should not depend on methods they do not use |
|  D - Dependency Inversion  : Depend on abstractions, not on concrete implementations|
+-----------------------------------------------------------------------------------+
```

---

### Q1. How do you implement the Open/Closed Principle (OCP) elegantly in Spring Boot?
**Answer:**
Use the **Strategy Pattern with Spring Map/List Dependency Injection**.

```java
// 1. Define Common Strategy Interface
public interface PaymentProcessor {
    String getPaymentMethod(); // "STRIPE", "PAYPAL", "CRYPTO"
    PaymentResponse process(PaymentRequest request);
}

// 2. Implement Concrete Strategies as Spring @Component
@Component
public class StripePaymentProcessor implements PaymentProcessor {
    @Override public String getPaymentMethod() { return "STRIPE"; }
    @Override public PaymentResponse process(PaymentRequest req) { /* Stripe logic */ return null; }
}

@Component
public class PaypalPaymentProcessor implements PaymentProcessor {
    @Override public String getPaymentMethod() { return "PAYPAL"; }
    @Override public PaymentResponse process(PaymentRequest req) { /* PayPal logic */ return null; }
}

// 3. Open for Extension: Adding a new payment method requires ZERO changes to existing service!
@Service
public class PaymentService {
    private final Map<String, PaymentProcessor> processors;

    // Spring automatically injects all PaymentProcessor beans into the list!
    public PaymentService(List<PaymentProcessor> processorList) {
        this.processors = processorList.stream()
            .collect(Collectors.toMap(PaymentProcessor::getPaymentMethod, Function.identity()));
    }

    public PaymentResponse executePayment(String method, PaymentRequest request) {
        PaymentProcessor processor = processors.get(method.toUpperCase());
        if (processor == null) {
            throw new UnsupportedPaymentMethodException("No processor found for: " + method);
        }
        return processor.process(request);
    }
}
```

---

## 2. Gang of Four (GoF) Design Patterns in Spring Boot

### Q2. How are GoF Patterns implemented natively across Spring Framework?
**Answer:**

| GoF Pattern | Pattern Type | Where It is Used in Spring Framework / Spring Boot |
| :--- | :--- | :--- |
| **Singleton** | Creational | Default Spring Bean Scope (`@Component`, `@Service`) |
| **Factory Method**| Creational | `BeanFactory`, `FactoryBean`, `ResponseEntity.ok()`, `ProblemDetail.forStatus()` |
| **Builder** | Creational | `RestClient.builder()`, `WebClient.builder()`, `SecurityFilterChain` |
| **Prototype** | Creational | `@Scope("prototype")` (creates a new instance on every injection) |
| **Adapter** | Structural | `HandlerAdapter` in Spring MVC, `JpaVendorAdapter` |
| **Proxy** | Structural | Spring AOP (`@Transactional`, `@Async`, `@Cacheable` proxies) |
| **Decorator** | Structural | `ServerHttpRequestDecorator`, `TransactionAwareCacheDecorator` |
| **Facade** | Structural | `JdbcTemplate`, `RestTemplate`, `KafkaTemplate` |
| **Strategy** | Behavioral | Spring Map-injection, `ResourceLoader`, `AuthenticationProvider` |
| **Observer** | Behavioral | Spring Application Events (`ApplicationEventPublisher` & `@EventListener`)|
| **Chain of Resp.**| Behavioral | Servlet Filters, `SecurityFilterChain`, `HandlerInterceptor` |
| **Template Method**| Behavioral | `JdbcTemplate.execute()`, `AbstractBeanFactory` |

---

### Q3. Implement the Chain of Responsibility Pattern for Request Validation.
**Answer:**

```java
public abstract class OrderValidationHandler {
    private OrderValidationHandler next;

    public OrderValidationHandler setNext(OrderValidationHandler next) {
        this.next = next;
        return next;
    }

    public void handle(Order order) {
        validate(order);
        if (next != null) {
            next.handle(order);
        }
    }

    protected abstract void validate(Order order);
}

@Component
public class InventoryCheckHandler extends OrderValidationHandler {
    @Override
    protected void validate(Order order) {
        if (order.getItems().isEmpty()) {
            throw new ValidationException("Order must have at least one item");
        }
    }
}

@Component
public class CustomerCreditCheckHandler extends OrderValidationHandler {
    @Override
    protected void validate(Order order) {
        if (order.getCustomer().getCreditBalance().compareTo(order.getTotalAmount()) < 0) {
            throw new InsufficientCreditException("Customer credit limit exceeded");
        }
    }
}
```

---

> **Next Chapter**: [13 Advanced Spring Security & OAuth2](13_Spring_Security_Advanced.md)
