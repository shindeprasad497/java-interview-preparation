# Microservices Architecture, DDD & Event Sourcing

> **Navigation**: [Master Index](README.md) | [Previous: Advanced Spring Cloud](14_Spring_Cloud_Advanced.md) | [Next: Spring AI Guide](16_Spring_AI_Interview_Guide.md)

---

## 1. Domain-Driven Design (DDD) in Microservices

```
+-----------------------------------------------------------------------------------+
|                        DOMAIN-DRIVEN DESIGN (DDD) TAXONOMY                        |
+-----------------------------------------------------------------------------------+
|  [ Bounded Context: Order Management ]                                            |
|  +-----------------------------------------------------------------------------+  |
|  |  AGGREGATE ROOT: Order (Enforces business invariants)                      |  |
|  |  +-----------------------------------------------------------------------+  |  |
|  |  |  - Entity: OrderItem (Has identity: item_id)                         |  |  |
|  |  |  - Value Object: Money / Address (Immutable, equality by attributes) |  |  |
|  |  |  - Domain Event: OrderPlacedEvent (Published on state change)       |  |  |
|  |  +-----------------------------------------------------------------------+  |  |
|  +-----------------------------------------------------------------------------+  |
|                                       | (Published via Kafka)                     |
|                                       v                                           |
|  [ Bounded Context: Billing & Invoicing ] ---> Translates via Anti-Corruption Layer|
+-----------------------------------------------------------------------------------+
```

---

### Q1. Deep Dive: Entities vs. Value Objects vs. Aggregates.
**Answer:**

| DDD Concept | Definition | Equality Comparison | Java Implementation |
| :--- | :--- | :--- | :--- |
| **Entity** | Object with distinct continuous identity throughout its lifecycle | By unique identifier (`id`) | Class with private ID (`Order`, `Customer`) |
| **Value Object** | Immutable object defined strictly by the combination of its attributes | By value of all attributes | Java `record` (`Money(amount, currency)`, `Address`) |
| **Aggregate** | Cluster of domain objects treated as a single transactional unit | Via Aggregate Root ID | Root Entity containing child entities and value objects |
| **Aggregate Root**| Only member of the Aggregate that outside objects are allowed to reference directly | By Root ID | `Order` is Root; `OrderItem` cannot be updated without going through `Order` |

```java
// Immutable Value Object in Java
public record Money(BigDecimal amount, Currency currency) {
    public Money {
        if (amount.compareTo(BigDecimal.ZERO) < 0) {
            throw new IllegalArgumentException("Money cannot be negative");
        }
    }
    public Money add(Money other) {
        if (!this.currency.equals(other.currency)) {
            throw new CurrencyMismatchException("Cannot add different currencies");
        }
        return new Money(this.amount.add(other.amount), this.currency);
    }
}
```

---

## 2. CQRS (Command Query Responsibility Segregation) & Event Sourcing

```
+-----------------------------------------------------------------------------------+
|                          CQRS & EVENT SOURCING ARCHITECTURE                       |
+-----------------------------------------------------------------------------------+
|  [ Client (Write Command) ]                           [ Client (Read Query) ]     |
|              |                                                    ^               |
|              v                                                    |               |
|   [ Command Controller ]                                 [ Query Controller ]     |
|              |                                                    |               |
|              v                                                    v               |
|   [ Write Model / Aggregate ]                            [ Read Model (Elastic/   |
|              |                                             Postgres View / Redis)]|
|              v (Appends Event)                                    ^               |
|   [ Event Store (DB / Kafka) ]                                    |               |
|   [ OrderCreated, ItemAdded, OrderPaid ]                          |               |
|              |                                                    |               |
|              +---> (Kafka Event Stream) ---> [ Read Projector ] --+               |
+-----------------------------------------------------------------------------------+
```

---

### Q2. Compare Traditional CRUD vs. Event Sourcing.
**Answer:**

- **Traditional CRUD**: Overwrites database rows (`UPDATE orders SET status = 'PAID' WHERE id = 123`). The previous state is permanently destroyed; historical audit trails require separate audit tables.
- **Event Sourcing**: Never updates or deletes state. It stores state as an **immutable append-only sequence of domain events** (`OrderCreated`, `PaymentReceived`, `ItemShipped`).
  - **State Reconstruction**: The current state of an entity is derived by replaying all historical events from timestamp zero.
  - **Snapshots**: To optimize replay performance for aggregates with thousands of events, periodic snapshots (e.g., every 100 events) are saved to cache.

---

## 3. Monolith Decomposition Strategies

### Q3. How do you migrate a legacy Monolith to Microservices with Zero Downtime?
**Answer:**
Use the **Strangler Fig Pattern** combined with **Branch by Abstraction**:

1. **Step 1: Identify Bounded Context**: Extract a discrete domain with low coupling (e.g., `NotificationService` or `PricingService`).
2. **Step 2: Place an API Gateway / Reverse Proxy**: Route all traffic to the Monolith initially.
3. **Step 3: Build New Microservice**: Implement the extracted capability in the new Spring Boot microservice.
4. **Step 4: Dual-Run / Dark Launch**: Gateway duplicates traffic (shadow traffic) to the microservice to verify performance and correctness without serving responses to users.
5. **Step 5: Cutover & Strangle**: Switch the API Gateway route to direct 100% of traffic to the microservice. Deprecate and remove old code from the monolith.

---

> **Next Chapter**: [16 Spring AI Interview Guide](16_Spring_AI_Interview_Guide.md)
