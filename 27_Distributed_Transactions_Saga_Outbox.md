# 27. Distributed Transactions: Saga Pattern, Outbox & Distributed Locks

> **Navigation**: [Master Index](README.md) | [Previous: Microservices & DDD](26_Microservices_DDD_Architecture.md) | [Next: Spring Cloud Gateway & Resilience](28_Spring_Cloud_Gateway_Resilience.md)

---

## 📌 Chapter Overview
This module explores **Distributed Transactions** in microservices, comparing **Saga Orchestration vs. Choreography**, solving the Dual-Write problem using the **Transactional Outbox Pattern with Debezium CDC**, and **Redisson Distributed Locks**.

---

## 1. The Saga Pattern: Orchestration vs. Choreography

```
+-----------------------------------------------------------------------------------+
|                         SAGA PATTERN: ORCHESTRATION VS CHOREOGRAPHY               |
+-----------------------------------------------------------------------------------+
|  1. SAGA ORCHESTRATION (Central State Machine Controller):                        |
|                                                                                   |
|                   +-------------------------------+                               |
|                   |   Order Saga Orchestrator     |                               |
|                   +-------------------------------+                               |
|                     /            |            \                                   |
|       1. Reserve   / 2. Process  | 3. Reserve  \                                  |
|         Payment   /     Credit   |    Inventory \                                 |
|                  v               v               v                                |
|          [ Payment Svc ]  [ Account Svc ]  [ Inventory Svc ]                      |
|                                                                                   |
|  2. SAGA CHOREOGRAPHY (Event-Driven Pub-Sub):                                     |
|                                                                                   |
|  [ Order Svc ] ---> (OrderCreated) ---> [ Payment Svc ] ---> (PaymentSuccess)     |
|                                                                     |             |
|  [ Shipping Svc ] <--- (StockReserved) <--- [ Inventory Svc ] <-----+             |
+-----------------------------------------------------------------------------------+
```

### Q1. Compare Saga Orchestration vs. Saga Choreography.
**Answer:**

| Dimension | Saga Orchestration | Saga Choreography |
| :--- | :--- | :--- |
| **Control Model** | Central coordinator orchestrates workflow | Decentralized; services react to domain events |
| **Visibility / Tracing**| **High** (Single state machine reveals progress) | Low (Distributed event tracing required) |
| **Service Coupling** | Orchestrator knows about all participants | **Loosely coupled** via message topics |
| **Compensating Actions**| Orchestrator triggers explicit rollbacks | Services listen for failure events and rollback |
| **Recommended For** | **Complex multi-step enterprise workflows** | Simple 2–3 step workflows |

---

## 2. The Dual-Write Problem & Transactional Outbox Pattern

```
                                     [ Order Service ]
                                             |
                         +-------------------+-------------------+
                         | Single ACID Database Transaction       |
                         | 1. INSERT INTO orders (...)           |
                         | 2. INSERT INTO outbox_table (event)   |
                         +-------------------+-------------------+
                                             |
                                             v
                                  [ PostgreSQL Database ]
                                  (orders & outbox_table)
                                             |
                                             | (Reads WAL / binlog)
                                             v
                                 [ Debezium CDC Connector ]
                                             |
                                             v
                                 [ Apache Kafka Topic ]
```

### Q2. How does the Transactional Outbox Pattern solve the Dual-Write Problem?
**Answer:**
- **The Dual-Write Problem**: If you save to database (`orderRepo.save()`) and then publish to Kafka (`kafkaTemplate.send()`) in the same method, a network crash before sending to Kafka causes the DB to have the order while downstream services never hear about it.
- **The Outbox Solution**:
  1. Write the business entity (`orders`) AND the event payload (`outbox_table`) inside the **same local ACID database transaction**.
  2. A Change Data Capture tool (**Debezium CDC**) tails the database transaction write-ahead log (**WAL** in Postgres / **binlog** in MySQL).
  3. Debezium automatically captures committed outbox records and streams them into Apache Kafka with **Guaranteed At-Least-Once Delivery**.

---

## 3. Distributed Locking with Redisson

```java
@Service
public class InventoryReservationService {

    @Autowired private RedissonClient redissonClient;
    @Autowired private InventoryRepository inventoryRepo;

    public boolean reserveItem(Long itemId, int quantity) {
        // Distributed Lock key scoped per inventory item
        RLock lock = redissonClient.getLock("lock:inventory:" + itemId);

        try {
            // Wait up to 5s to acquire; auto-renew lease every 10s via Redisson Watchdog
            boolean acquired = lock.tryLock(5, 10, TimeUnit.SECONDS);
            if (!acquired) {
                log.warn("Could not acquire lock for item: {}", itemId);
                return false;
            }

            // Critical section: Protected across all distributed microservice instances!
            Inventory item = inventoryRepo.findById(itemId).orElseThrow();
            if (item.getStock() >= quantity) {
                item.setStock(item.getStock() - quantity);
                inventoryRepo.save(item);
                return true;
            }
            return false;

        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            return false;
        } finally {
            // Guarantee lock is released only if held by current thread
            if (lock.isHeldByCurrentThread()) {
                lock.unlock();
            }
        }
    }
}
```

---

> **Next Chapter**: [28 Spring Cloud Gateway, Resilience4j & Fault Tolerance](28_Spring_Cloud_Gateway_Resilience.md)

