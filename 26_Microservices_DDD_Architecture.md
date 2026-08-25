# 26. Microservices Architecture, DDD & CQRS / Event Sourcing

> **Navigation**: [Master Index](README.md) | [Previous: RabbitMQ vs Kafka](25_RabbitMQ_vs_Kafka_Messaging.md) | [Next: Distributed Transactions](27_Distributed_Transactions_Saga_Outbox.md)

---

## 📌 Chapter Overview
This module explores **Domain-Driven Design (DDD)** in microservices, Tactical DDD patterns (Entities, Value Objects, Aggregate Roots), **CQRS**, **Event Sourcing**, and the **Strangler Fig Monolith Decomposition Pattern**.

---

## 1. Domain-Driven Design (DDD) Tactical Building Blocks

```
+-----------------------------------------------------------------------------------+
|                            AGGREGATE ROOT BOUNDARY                                |
+-----------------------------------------------------------------------------------+
|  +-----------------------------------------------------------------------------+  |
|  |             ORDER AGGREGATE ROOT (Consistency Boundary)                     |  |
|  |  - OrderId id (Entity Identifier)                                           |  |
|  |  - OrderStatus status                                                       |  |
|  |  - Address shippingAddress (Value Object)                                   |  |
|  |  - Money totalAmount (Value Object)                                         |  |
|  |                                                                             |  |
|  |  +---------------------------+       +---------------------------+          |  |
|  |  |  OrderItem (Entity)       |       |  OrderItem (Entity)       |          |  |
|  |  |  - itemId, quantity, price|       |  - itemId, quantity, price|          |  |
|  |  +---------------------------+       +---------------------------+          |  |
|  +-----------------------------------------------------------------------------+  |
|  * Outside services can ONLY access internal entities through the Aggregate Root! |
+-----------------------------------------------------------------------------------+
```

### Q1. Compare Entities vs. Value Objects vs. Aggregate Roots.
**Answer:**

| Concept | Definition | Identity | Mutability | Example |
| :--- | :--- | :--- | :--- | :--- |
| **Entity** | Object with distinct operational identity that persists over time | Explicit ID (`id = 101`) | Mutable state | `User`, `Order`, `Product` |
| **Value Object** | Object that describes characteristics without distinct identity | Identified by **all its attributes** | **100% Immutable** | `Money(amount, currency)`, `Address`, `GeoPoint` |
| **Aggregate Root**| Primary entity that guards a cluster of associated objects and enforces transactional business invariants | Root Entity ID | Controls mutations | `Order` (controls `OrderItem` collections) |

---

## 2. CQRS & Event Sourcing Architecture

```
                                [ User Action ]
                                       |
                     +-----------------+-----------------+
                     |                                   |
                     v (Command: Write / Update)         v (Query: Read)
             [ Command Service ]                  [ Query Service ]
                     |                                   ^
                     v                                   |
             [ Write Database ]                   [ Read Database ]
         (Normalized / PostgreSQL)            (Denormalized / Elasticsearch)
                     |                                   ^
                     +-----> [ Kafka Event Sync ] -------+
```

### Q2. Compare Traditional CRUD vs. Event Sourcing.
**Answer:**
- **Traditional CRUD**: Mutates records in-place (`UPDATE accounts SET balance = 500 WHERE id = 1`). Overwrites historical state, destroying audit trails.
- **Event Sourcing**: Instead of storing current state, you store an **immutable sequence of domain delta events** (`AccountCreated`, `MoneyDeposited(1000)`, `MoneyDebited(500)`).
  - The current state of an entity is reconstructed by **replaying all past events** from the event store.
  - Provides a 100% accurate, tamper-proof financial audit trail and temporal queries (e.g., *"What was the account state on Nov 15th at 3 PM?"*).

---

## 3. Monolith Decomposition: Strangler Fig Pattern & ACL

```
 [ Client / Mobile ] ---> [ API Gateway / Reverse Proxy ]
                                /                     \
             (90% Legacy Routes)                       (10% New Microservice Routes)
                      v                                             v
          [ Legacy Monolith App ]                       [ New Order Microservice ]
                      |                                             ^
                      +=======> [ Anti-Corruption Layer (ACL) ] ====+
                                (Translates legacy data models)
```

### Q3. How do you migrate a legacy Monolith to Microservices with Zero Downtime?
**Answer:**
1. **Strangler Fig Pattern**: Introduce an API Gateway in front of the monolith. Gradually carve out domain modules (e.g., `OrderService`) into microservices, routing matching endpoint URLs to the new service while proxying remaining traffic to the legacy monolith.
2. **Anti-Corruption Layer (ACL)**: Implement a translation adapter between the legacy monolith and new microservice to prevent the legacy system's technical debt or messy schemas from polluting the new microservice's clean domain model.

---

> **Next Chapter**: [27 Distributed Transactions: Saga & Transactional Outbox](27_Distributed_Transactions_Saga_Outbox.md)

