# Persistence, Transactions & Messaging Deep Dive

> **Navigation**: [Master Index](README.md) | [Previous: Web, REST & Reactive](05_Web_REST_Reactive.md) | [Next: Security & Observability](07_Security_Testing_Observability.md)

---

## 1. Hibernate & Spring Data JPA Internals

### Q1. Trace Hibernate Entity Lifecycle States.
**Answer:**

```
               [ New Entity () ]
                       |
                       | persist() / save()
                       v
       +-------------------------------+
       |       MANAGED / PERSISTENT    | <=======> [ 1st-Level Cache / PersistenceContext ]
       +-------------------------------+           (Dirty checking active, auto-synced on flush)
          /            |            \
 detach()|             |remove()     | close() / clear()
         v             v             v
   +----------+   +---------+   +----------+
   | DETACHED |   | REMOVED |   | DETACHED |
   +----------+   +---------+   +----------+
         |
 merge() |
         v
   [ MANAGED ]
```

1. **Transient (New)**: Instantiated with `new Object()`, has no primary key ID, not associated with a JPA `EntityManager`.
2. **Managed (Persistent)**: Associated with the active `PersistenceContext` (1st-level cache) and has a database ID. Any modification to its fields is automatically detected by **Dirty Checking** and persisted on transaction commit without calling `save()`.
3. **Detached**: Had a database ID and was managed, but the `PersistenceContext` closed, cleared (`em.clear()`), or detached (`em.detach()`).
4. **Removed**: Marked for deletion in the database (`em.remove()`), removed from the persistence context during flush.

---

### Q2. Deep Dive: What is the N+1 Query Problem and how do you resolve it?
**Answer:**
- **The Problem**: When fetching N parent entities with a lazily-loaded `@OneToMany` or `@ManyToOne` relationship, accessing the child collection triggers **1 initial query** for the parents followed by **N additional queries** for each parent's children (1 + N queries).

```java
// Triggers 1 query for Orders + 100 queries for Customers if there are 100 orders!
List<Order> orders = orderRepository.findAll();
for (Order o : orders) {
    System.out.println(o.getCustomer().getName()); // Lazy loading N queries!
}
```

#### The 3 Production Fixes:

```java
public interface OrderRepository extends JpaRepository<Order, Long> {

    // Fix 1: JPQL JOIN FETCH (Single SQL INNER/LEFT JOIN)
    @Query("SELECT o FROM Order o JOIN FETCH o.customer c JOIN FETCH o.items")
    List<Order> findAllWithCustomerAndItems();

    // Fix 2: Spring Data @EntityGraph (Generates SQL JOIN dynamically)
    @EntityGraph(attributePaths = { "customer", "items" })
    List<Order> findByStatus(OrderStatus status);
}

// Fix 3: Hibernate @BatchSize (Batches lazy fetches using SQL IN clauses: 1 + (N/batchSize) queries)
@Entity
public class Order {
    @OneToMany(mappedBy = "order", fetch = FetchType.LAZY)
    @BatchSize(size = 50) // Fetches children in batches of 50 using 'WHERE order_id IN (?, ?, ...)'
    private List<OrderItem> items;
}
```

---

## 2. Spring Transactions & Database Locking

### Q3. Explain `@Transactional` Propagation Types and Rollback Mechanics.
**Answer:**

#### 1. Propagation Types:
- **`REQUIRED` (Default)**: Executes within the active transaction. If none exists, creates a new one.
- **`REQUIRES_NEW`**: Always creates a new, independent transaction, suspending the current active transaction until the inner one completes.
- **`NESTED`**: Executes within a nested transaction using database **Savepoints**. If the nested transaction fails, it rolls back to the savepoint without rolling back the outer transaction.
- **`MANDATORY`**: Requires an existing transaction; throws `TransactionRequiredException` if none exists.
- **`SUPPORTS`**: Runs in transaction if one exists; runs non-transactionally if none exists.
- **`NOT_SUPPORTED`**: Always runs non-transactionally, suspending any active transaction.
- **`NEVER`**: Throws an exception if an active transaction exists.

#### 2. Rollback Rules:
- By default, Spring rolls back transactions **ONLY on Unchecked Exceptions (`RuntimeException` and `Error`)**.
- Checked exceptions (`IOException`, `SQLException`, custom checked exceptions) do **NOT** trigger rollback unless explicitly configured:

```java
@Transactional(rollbackFor = Exception.class, propagation = Propagation.REQUIRED)
public void executeFinancialTransfer(TransferRequest req) throws PaymentException {
    accountRepo.debit(req.sourceId(), req.amount());
    accountRepo.credit(req.targetId(), req.amount());
}
```

---

### Q4. Optimistic Locking vs. Pessimistic Locking: When to use which?
**Answer:**

```java
// 1. OPTIMISTIC LOCKING: Best for low-contention, read-heavy workloads
@Entity
public class InventoryItem {
    @Id private Long id;
    private int stockQuantity;

    @Version // Automatically increments on each UPDATE. Throws OptimisticLockException if conflict!
    private Long version;
}

// 2. PESSIMISTIC LOCKING: Best for high-contention, critical concurrency (Banking / Flash Sales)
public interface InventoryRepository extends JpaRepository<InventoryItem, Long> {
    // Generates SQL: 'SELECT * FROM inventory_item WHERE id = ? FOR UPDATE'
    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query("SELECT i FROM InventoryItem i WHERE i.id = :id")
    Optional<InventoryItem> findByIdForUpdate(@Param("id") Long id);
}
```

---

### Q5. How do you size the HikariCP Database Connection Pool?
**Answer:**
A common mistake is making connection pools too large (e.g., 200 connections), which causes intense CPU context switching and disk spindle thrashing on the database server.

#### PostgreSQL / HikariCP Pool Sizing Formula:
```
Pool Size = (CPU Cores * 2) + Effective Spindle Count (SSD = 1)
```
*Example: A 4-core database server performs best with ~10 to 15 connections!*

```properties
# Optimal Production HikariCP Settings
spring.datasource.hikari.maximum-pool-size=15
spring.datasource.hikari.minimum-idle=10
spring.datasource.hikari.idle-timeout=300000
spring.datasource.hikari.connection-timeout=20000 # 20s wait before failing
spring.datasource.hikari.max-lifetime=1800000     # 30m connection recycling
spring.datasource.hikari.leak-detection-threshold=20000 # Warns if connection held > 20s
```

---

## 3. Distributed Messaging with Apache Kafka

### Q6. Deep Dive: Apache Kafka Partitioning, Consumer Groups & Exactly-Once Semantics.
**Answer:**

```
+-----------------------------------------------------------------------------------+
|                                 KAFKA TOPIC (4 Partitions)                        |
+-----------------------------------------------------------------------------------+
|  Partition 0: [msg 0][msg 1][msg 2][msg 3] -----> Consumer A (Group: "order-grp") |
|  Partition 1: [msg 0][msg 1][msg 2] ------------> Consumer A                      |
|  Partition 2: [msg 0][msg 1] -------------------> Consumer B (Group: "order-grp") |
|  Partition 3: [msg 0][msg 1][msg 2][msg 3][msg 4]-> Consumer C (Group: "order-grp")|
+-----------------------------------------------------------------------------------+
```

1. **Partitioning Key**:
   - Messages with the **same partition key** (e.g., `userId`, `orderId`) hash to the **exact same partition**, guaranteeing strict FIFO ordering for that entity.
   - Messages without a key use sticky round-robin batching.
2. **Consumer Groups & Scaling**:
   - Each partition is consumed by **exactly one consumer instance** within a consumer group.
   - Max parallel consumers for a topic = Number of Partitions. Extra consumers in the group remain idle.
3. **Reliability Configurations**:

```properties
# Producer Reliability (Zero Data Loss)
spring.kafka.producer.acks=all                # Leader + all in-sync replicas must ACK
spring.kafka.producer.properties.enable.idempotence=true # Prevents duplicate broker writes
spring.kafka.producer.retries=10

# Consumer Reliability
spring.kafka.consumer.enable-auto-commit=false # Manual offset commit after business logic succeeds
spring.kafka.consumer.auto-offset-reset=earliest
```

---

### Q7. Non-Blocking Retries and Dead Letter Queues (DLT) in Spring Kafka.
**Answer:**
Blocking retries (`Thread.sleep()`) stall the entire Kafka partition for all other messages. Spring Kafka provides **non-blocking topic-based retries** using `@RetryableTopic`.

```java
@Service
public class OrderEventConsumer {
    private static final Logger log = LoggerFactory.getLogger(OrderEventConsumer.class);

    @RetryableTopic(
        attempts = "4",
        backoff = @Backoff(delay = 1000, multiplier = 2.0, maxDelay = 10000),
        dltStrategy = DltStrategy.FAIL_ON_ERROR,
        include = { TransientNetworkException.class, TimeoutException.class }
    )
    @KafkaListener(topics = "orders.created", groupId = "inventory-service")
    public void processOrderEvent(OrderCreatedEvent event, @Header(KafkaHeaders.RECEIVED_TOPIC) String topic) {
        log.info("Received event {} from topic {}", event.orderId(), topic);
        inventoryService.reserveStock(event);
    }

    // Handles terminal failures after all 4 retry topics are exhausted
    @DltHandler
    public void handleDlt(OrderCreatedEvent event, @Header(KafkaHeaders.EXCEPTION_MESSAGE) String error) {
        log.error("Event {} routed to DEAD LETTER TOPIC (DLT). Root cause: {}", event.orderId(), error);
        alertService.sendPagerDutyAlert("DLT Poison Pill", event);
    }
}
```

---

> **Next Chapter**: [07 Security, Testing & Observability](07_Security_Testing_Observability.md)
