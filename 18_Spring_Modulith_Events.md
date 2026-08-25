# 18. Spring Modulith & Event-Driven In-Process Decoupling

> **Navigation**: [Master Index](README.md) | [Previous: Spring Batch](17_Spring_Batch_ETL_Processing.md) | [Next: Hibernate & JPA Internals](19_Hibernate_JPA_Internals.md)

---

## 📌 Chapter Overview
This module explores **Spring Modulith**, modular monolith domain architectures, automated architectural boundary verification, and transaction-bound **in-process event publication (`@TransactionalEventListener`)**.

---

## 1. Modular Monolith vs. Microservices

```
       DISTRIBUTED MICROSERVICES                   SPRING MODULITH (Modular Monolith)
+---------------------------------------+    +-----------------------------------------------+
| [ Orders API ] -> Network -> [ Pay ]  |    |  +--------------------+  +------------------+ |
|       | Network              | Network|    |  |    orders module   |  |  payment module  | |
|       v                      v        |    |  +--------------------+  +------------------+ |
| [ Inventory ]         [ Notification ]|    |            \                     /            |
| * High network latency, distributed   |    |    In-Process Events (Zero Network Latency)   |
|   transactions, complex deployments   |    | * Strict package boundaries & single deploy   |
+---------------------------------------+    +-----------------------------------------------+
```

### Q1. What is Spring Modulith and what problem does it solve?
**Answer:**
**Spring Modulith** enables developers to build maintainable, deeply structured monoliths divided into clean, decoupled domain modules. It provides:
1. **Architectural Verification**: Automated unit tests that detect illegal cross-module dependencies or package leaks at build time.
2. **Event-Driven Decoupling**: Modules communicate via published domain events rather than direct bean autowiring.
3. **Durable In-Process Event Publication Registry**: Guarantees that internal events are not lost even if the application restarts during event processing.

---

## 2. Architectural Verification in Unit Tests

```java
public class ArchitectureTests {

    @Test
    void verifyModularStructure() {
        ApplicationModules modules = ApplicationModules.of(Application.class);
        
        // Fails test if module 'orders' illegally calls internal packages of module 'inventory'
        modules.verify();
    }
}
```

---

## 3. `@TransactionalEventListener` & Transaction Synchronization

### Q2. Why is sending events or emails inside `@Transactional` an anti-pattern? How does `@TransactionalEventListener` fix it?
**Answer:**
- **The Problem**: If a service publishes a Kafka message or sends a confirmation email *inside* a `@Transactional` method, and the database transaction subsequently fails/rolls back during DB commit, the email or Kafka message has already been dispatched $\rightarrow$ **Ghost Notification / Data Inconsistency**!

```java
// ❌ ANTI-PATTERN: Dispatches event before DB commits!
@Service
public class OrderService {
    @Transactional
    public void createOrder(OrderRequest req) {
        orderRepo.save(new Order(req));
        // If DB commit fails 2 lines later, user still receives confirmation email!
        notificationService.sendEmail("Order confirmed!"); 
    }
}
```

#### Production Solution: `@TransactionalEventListener(phase = AFTER_COMMIT)`

```java
@Service
public class OrderService {
    @Autowired private ApplicationEventPublisher eventPublisher;
    @Autowired private OrderRepository orderRepo;

    @Transactional
    public Order createOrder(OrderRequest req) {
        Order order = orderRepo.save(new Order(req));
        // Publish in-process event
        eventPublisher.publishEvent(new OrderCreatedEvent(order.getId(), order.getCustomerEmail()));
        return order;
    }
}

@Component
public class OrderNotificationListener {

    // Triggers ONLY after the database transaction successfully commits!
    @Async
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void handleOrderCreated(OrderCreatedEvent event) {
        // Safe to dispatch emails, push notifications, or Kafka events!
        emailService.sendConfirmation(event.customerEmail(), event.orderId());
    }
}
```

---

> **Next Chapter**: [19 Hibernate & Spring Data JPA Internals](19_Hibernate_JPA_Internals.md)

