# 12. SOLID Principles & Gang of Four (GoF) Design Patterns

> **Navigation**: [Master Index](README.md) | [Previous: Coding Challenges](11_Coding_Data_Structures_Problems.md) | [Next: Spring Core & AOP](13_Spring_Core_IoC_AOP_Internals.md)

---

## 📌 Chapter Overview
This module provides a complete deep dive into **all 5 SOLID Principles** with real-world enterprise Java / Spring Boot code examples and maps the classic **Gang of Four (GoF) Design Patterns** directly to Spring Framework implementations.

---

## 1. The 5 SOLID Principles with Real-World Spring Boot Code Examples

```
+-----------------------------------------------------------------------------------+
|                             SOLID DESIGN PRINCIPLES                               |
+-----------------------------------------------------------------------------------+
|  S - Single Responsibility   -> A class should have one, and only one, reason to change |
|  O - Open / Closed           -> Open for extension, closed for modification       |
|  L - Liskov Substitution     -> Subtypes must be substitutable for base types     |
|  I - Interface Segregation   -> Clients should not depend on interfaces they don't use |
|  D - Dependency Inversion    -> High-level modules should depend on abstractions  |
+-----------------------------------------------------------------------------------+
```

---

### 1. Single Responsibility Principle (SRP)
> *"A class should have only one reason to change, meaning it should have only one job or responsibility."*

#### ❌ Bad Example (SRP Violation - "God Class"):
```java
// ❌ BAD: Handles order processing, payment charging, database saving, and email notifications!
@Service
public class OrderService {
    public void processOrder(Order order) {
        // 1. Validate order
        if (order.getItems().isEmpty()) throw new IllegalArgumentException();
        // 2. Charge payment via Stripe API
        StripeClient.charge(order.getAmount());
        // 3. Save to database
        jdbcTemplate.update("INSERT INTO orders VALUES (...)");
        // 4. Send confirmation email
        JavaMailSender.sendEmail(order.getUserEmail(), "Order Confirmed");
    }
}
```

#### ✅ Good Example (Adhering to SRP):
```java
// ✅ GOOD: Each service has exactly ONE distinct responsibility!
@Service
public class OrderService {
    private final OrderRepository orderRepository;
    private final PaymentProcessor paymentProcessor;
    private final NotificationService notificationService;

    public OrderService(OrderRepository repo, PaymentProcessor payment, NotificationService notify) {
        this.orderRepository = repo;
        this.paymentProcessor = payment;
        this.notificationService = notify;
    }

    @Transactional
    public Order createOrder(CreateOrderRequest req) {
        paymentProcessor.charge(req.amount());
        Order savedOrder = orderRepository.save(new Order(req));
        notificationService.sendConfirmation(savedOrder);
        return savedOrder;
    }
}
```

---

### 2. Open/Closed Principle (OCP)
> *"Software entities (classes, modules, functions) should be open for extension, but closed for modification."*

#### ✅ Good Example (Spring Strategy Pattern + Factory Map):
```java
// Step 1: Extensible Strategy Interface
public interface PaymentStrategy {
    PaymentProvider getProvider();
    PaymentResponse process(PaymentRequest request);
}

// Step 2: Individual Strategy Implementations
@Component
public class StripePaymentStrategy implements PaymentStrategy {
    @Override public PaymentProvider getProvider() { return PaymentProvider.STRIPE; }
    @Override public PaymentResponse process(PaymentRequest req) { return new PaymentResponse("STRIPE_SUCCESS"); }
}

@Component
public class PayPalPaymentStrategy implements PaymentStrategy {
    @Override public PaymentProvider getProvider() { return PaymentProvider.PAYPAL; }
    @Override public PaymentResponse process(PaymentRequest req) { return new PaymentResponse("PAYPAL_SUCCESS"); }
}

// Step 3: Open for Extension, Closed for Modification Dispatcher
@Service
public class PaymentDispatcherService {
    private final Map<PaymentProvider, PaymentStrategy> strategyMap;

    // Spring auto-injects all strategies into a Map by provider key!
    public PaymentDispatcherService(List<PaymentStrategy> strategies) {
        this.strategyMap = strategies.stream()
            .collect(Collectors.toMap(PaymentStrategy::getProvider, Function.identity()));
    }

    public PaymentResponse executePayment(PaymentRequest request) {
        PaymentStrategy strategy = strategyMap.get(request.provider());
        if (strategy == null) {
            throw new IllegalArgumentException("Unsupported payment provider: " + request.provider());
        }
        return strategy.process(request); // Zero if-else or switch statements!
    }
}
```
*(Adding ApplePay requires creating a new class `ApplePayStrategy` with `@Component`. Existing classes are never modified!)*

---

### 3. Liskov Substitution Principle (LSP)
> *"Subtypes must be substitutable for their base types without altering the correctness of the program."*

#### ❌ Bad Example (LSP Violation):
```java
// ❌ BAD: ReadOnlyAccount breaks the contract of Account by throwing UnsupportedOperationException
public class Account {
    public void deposit(double amount) { /* ... */ }
    public void withdraw(double amount) { /* ... */ }
}

public class FixedTermDepositAccount extends Account {
    @Override
    public void withdraw(double amount) {
        // Violates LSP: Client expecting an Account crashes at runtime!
        throw new UnsupportedOperationException("Withdrawals not allowed on fixed term accounts!");
    }
}
```

#### ✅ Good Example (Adhering to LSP):
```java
// ✅ GOOD: Segregate base behavior into valid substitutable abstractions
public interface Account {
    double getBalance();
    void deposit(double amount);
}

public interface WithdrawableAccount extends Account {
    void withdraw(double amount);
}

public class SavingsAccount implements WithdrawableAccount {
    @Override public void deposit(double amount) { /* ... */ }
    @Override public void withdraw(double amount) { /* ... */ }
    @Override public double getBalance() { return 1000.0; }
}

public class FixedTermDepositAccount implements Account {
    @Override public void deposit(double amount) { /* ... */ }
    @Override public double getBalance() { return 5000.0; }
    // No withdraw() method exists, preserving 100% type safety and substitution!
}
```

---

### 4. Interface Segregation Principle (ISP)
> *"Clients should not be forced to depend upon interfaces that they do not use."*

#### ❌ Bad Example (Fat "Polluted" Interface):
```java
// ❌ BAD: Forces all workers to implement unrelated operations
public interface Worker {
    void writeCode();
    void reviewCode();
    void designArchitecture();
    void manageDatabase();
}
```

#### ✅ Good Example (Role-Specific Segregated Interfaces):
```java
// ✅ GOOD: Fine-grained, focused role interfaces
public interface CodeWriter { void writeCode(); }
public interface CodeReviewer { void reviewCode(); }
public interface Architect { void designArchitecture(); }
public interface DatabaseAdmin { void manageDatabase(); }

// Senior Engineer implements only relevant capabilities:
public class SeniorSoftwareEngineer implements CodeWriter, CodeReviewer, Architect {
    @Override public void writeCode() { /* ... */ }
    @Override public void reviewCode() { /* ... */ }
    @Override public void designArchitecture() { /* ... */ }
}
```

---

### 5. Dependency Inversion Principle (DIP)
> *"High-level modules should not depend on low-level modules. Both should depend on abstractions (interfaces)."*

#### ❌ Bad Example (DIP Violation):
```java
// ❌ BAD: High-level OrderService directly depends on concrete low-level SendGridEmailClient
public class OrderService {
    private SendGridEmailClient emailClient = new SendGridEmailClient(); // Hardcoded dependency!
}
```

#### ✅ Good Example (Spring Inversion of Control & DIP):
```java
// Step 1: High-level abstraction
public interface NotificationChannel {
    void send(String recipient, String message);
}

// Step 2: Low-level concrete implementations
@Component
public class SendGridEmailNotification implements NotificationChannel {
    @Override public void send(String to, String msg) { /* Send via SendGrid */ }
}

@Component
public class TwilioSmsNotification implements NotificationChannel {
    @Override public void send(String to, String msg) { /* Send via Twilio SMS */ }
}

// Step 3: High-level service depends solely on the abstraction!
@Service
public class OrderNotificationService {
    private final NotificationChannel notificationChannel;

    // Injected via constructor - Decoupled from concrete implementation!
    public OrderNotificationService(@Qualifier("sendGridEmailNotification") NotificationChannel channel) {
        this.notificationChannel = channel;
    }
}
```

---

## 2. Gang of Four (GoF) Design Patterns in Spring Framework

```
+-------------------------------------------------------------------------------+
|                        GOF DESIGN PATTERNS IN SPRING FRAMEWORK                |
+-------------------------------------------------------------------------------+
| Category   | Pattern            | Spring Framework Implementation             |
| ---------- | ------------------ | ------------------------------------------- |
| Creational | **Factory Method** | `BeanFactory.getBean()`, `FactoryBean<T>`   |
| Creational | **Singleton**      | Default Spring Bean Scope                   |
| Creational | **Builder**        | `RestClient.builder()`, `SecurityFilterChain`|
| Structural | **Proxy**          | Spring AOP Dynamic Proxies (`@Transactional`)|
| Structural | **Adapter**        | `HandlerAdapter` in Spring MVC              |
| Structural | **Decorator**      | `WebSocketHandlerDecorator`                 |
| Behavioral | **Template Method**| `JdbcTemplate`, `TransactionTemplate`       |
| Behavioral | **Strategy**       | `AuthenticationProvider`, `TaskExecutor`    |
| Behavioral | **Observer**       | `ApplicationEventPublisher`, `@EventListener`|
| Behavioral | **Chain of Resp**  | `HandlerInterceptorChain`, `SecurityFilter` |
+-------------------------------------------------------------------------------+
```

---

> **Next Chapter**: [13 Spring Core, IoC Container & AOP Internals](13_Spring_Core_IoC_AOP_Internals.md)
