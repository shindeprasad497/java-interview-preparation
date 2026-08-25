# 12. SOLID Principles & Gang of Four (GoF) Design Patterns

> **Navigation**: [Master Index](README.md) | [Previous: Coding Challenges](11_Coding_Data_Structures_Problems.md) | [Next: Spring Core & AOP](13_Spring_Core_IoC_AOP_Internals.md)

---

## 📌 Chapter Overview
This module explores the **SOLID design principles** applied to real-world Spring Boot enterprise architectures and maps the classic **Gang of Four (GoF) Design Patterns** directly to Spring Framework implementations.

---

## 1. SOLID Principles Applied in Spring Boot

```
+-------------------------------------------------------------------------------+
|                             SOLID DESIGN PRINCIPLES                           |
+-------------------------------------------------------------------------------+
|  S - Single Responsibility   -> A class should have only one reason to change |
|  O - Open / Closed           -> Open for extension, closed for modification   |
|  L - Liskov Substitution     -> Subtypes must be substitutable for base types |
|  I - Interface Segregation   -> Clients shouldn't depend on unused methods    |
|  D - Dependency Inversion    -> Depend on abstractions, not concretions       |
+-------------------------------------------------------------------------------+
```

### Q1. How do you implement the Open/Closed Principle (OCP) elegantly in Spring Boot?
**Answer:**
**The Problem**: A payment service using `if-else` or `switch` statements violates OCP because adding a new payment provider (e.g., Apple Pay) requires modifying existing, tested code.

**The Solution**: Combine the **Strategy Pattern** with Spring's dependency injection to dynamically route requests without modifying existing classes.

```java
// Step 1: Strategy Interface
public interface PaymentStrategy {
    PaymentProvider getProvider();
    PaymentResponse process(PaymentRequest request);
}

// Step 2: Individual Strategy Implementations
@Component
public class StripePaymentStrategy implements PaymentStrategy {
    @Override public PaymentProvider getProvider() { return PaymentProvider.STRIPE; }
    @Override public PaymentResponse process(PaymentRequest req) { /* Stripe logic */ return new PaymentResponse(); }
}

@Component
public class PayPalPaymentStrategy implements PaymentStrategy {
    @Override public PaymentProvider getProvider() { return PaymentProvider.PAYPAL; }
    @Override public PaymentResponse process(PaymentRequest req) { /* PayPal logic */ return new PaymentResponse(); }
}

// Step 3: Payment Factory / Dispatcher (Open for extension, closed for modification!)
@Service
public class PaymentService {
    private final Map<PaymentProvider, PaymentStrategy> strategies;

    // Spring automatically injects all PaymentStrategy implementations into a List!
    public PaymentService(List<PaymentStrategy> strategyList) {
        this.strategies = strategyList.stream()
            .collect(Collectors.toMap(PaymentStrategy::getProvider, Function.identity()));
    }

    public PaymentResponse pay(PaymentRequest request) {
        PaymentStrategy strategy = strategies.get(request.provider());
        if (strategy == null) {
            throw new UnsupportedOperationException("Unsupported provider: " + request.provider());
        }
        return strategy.process(request);
    }
}
```
*(To add ApplePay, simply create `ApplePayStrategy` with `@Component`. Zero existing code is modified!)*

---

## 2. Gang of Four (GoF) Design Patterns in Spring Framework

### Q2. How are GoF Patterns implemented natively across Spring Framework?
**Answer:**

| Pattern Category | Design Pattern | Spring Framework Implementation |
| :--- | :--- | :--- |
| **Creational** | **Factory Method** | `BeanFactory.getBean()`, `FactoryBean<T>` |
| **Creational** | **Singleton** | Default Spring Bean Scope (`@Scope("singleton")`) |
| **Creational** | **Builder** | `RestClient.builder()`, `UriComponentsBuilder`, `SecurityFilterChain` |
| **Structural** | **Proxy** | Spring AOP Dynamic Proxies (`@Transactional`, `@Async`, `@Cacheable`) |
| **Structural** | **Adapter** | `HandlerAdapter` (Adapts custom controllers to `DispatcherServlet`) |
| **Structural** | **Decorator** | `HttpHeadResponseDecorator`, `WebSocketHandlerDecorator` |
| **Behavioral** | **Template Method** | `JdbcTemplate`, `TransactionTemplate`, `RestTemplate`, `JmsTemplate` |
| **Behavioral** | **Strategy** | `ResourceLoader`, `AuthenticationProvider`, `TaskExecutor` |
| **Behavioral** | **Observer** | Spring Application Events (`ApplicationEventPublisher`, `@EventListener`) |
| **Behavioral** | **Chain of Responsibility**| Spring MVC `HandlerInterceptorChain`, `SecurityFilterChain` |

---

### Q3. Implement the Chain of Responsibility Pattern for Request Validation.
**Answer:**

```java
public interface OrderValidationStep {
    void validate(OrderRequest request);
    void setNext(OrderValidationStep next);
}

public abstract class AbstractValidationStep implements OrderValidationStep {
    private OrderValidationStep next;
    @Override public void setNext(OrderValidationStep next) { this.next = next; }
    protected void checkNext(OrderRequest req) { if (next != null) next.validate(req); }
}

@Component
public class InventoryCheckStep extends AbstractValidationStep {
    @Override
    public void validate(OrderRequest request) {
        if (!hasInventory(request.itemId())) throw new ValidationException("Out of stock");
        checkNext(request);
    }
    private boolean hasInventory(Long id) { return true; }
}

@Component
public class FraudCheckStep extends AbstractValidationStep {
    @Override
    public void validate(OrderRequest request) {
        if (isFraudulent(request.userId())) throw new ValidationException("Fraud detected");
        checkNext(request);
    }
    private boolean isFraudulent(Long id) { return false; }
}
```

---

> **Next Chapter**: [13 Spring Core, IoC Container & AOP Internals](13_Spring_Core_IoC_AOP_Internals.md)

