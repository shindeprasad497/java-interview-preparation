# 24. Apache Kafka Production Engineering & Streaming

> **Navigation**: [Master Index](README.md) | [Previous: Redis Caching](23_Redis_Caching_Patterns_NoSQL.md) | [Next: RabbitMQ vs Kafka](25_RabbitMQ_vs_Kafka_Messaging.md)

---

## 📌 Chapter Overview
This module explores **Apache Kafka 3.x+ (KRaft mode)** internals, partition ordering guarantees, **Cooperative Sticky Rebalance protocols**, Exactly-Once Semantics (EOS), Non-blocking retry topics with **Dead Letter Topics (DLT)**, and **Consumer Lag tuning**.

---

## 1. Apache Kafka Architecture & KRaft Mode

```
+-----------------------------------------------------------------------------------+
|                        APACHE KAFKA TOPIC (4 Partitions)                          |
+-----------------------------------------------------------------------------------+
|  Partition 0: [msg 0][msg 1][msg 2][msg 3] -----> Consumer Instance A             |
|  Partition 1: [msg 0][msg 1][msg 2] ------------> Consumer Instance A             |
|  Partition 2: [msg 0][msg 1] -------------------> Consumer Instance B             |
|  Partition 3: [msg 0][msg 1][msg 2][msg 3][msg 4]-> Consumer Instance C           |
|                                                  (Consumer Group: "orders-group") |
+-----------------------------------------------------------------------------------+
```

### Q1. How does KRaft replace ZooKeeper in modern Kafka (Kafka 3.x+)?
**Answer:**
- **Legacy Kafka with ZooKeeper**: Stored cluster metadata, broker registries, and partition state in an external ZooKeeper cluster. This caused slow metadata synchronization, partition limits (~200k partitions), and dual-system operational overhead.
- **Modern Kafka with KRaft (Kafka Raft Metadata Mode)**:
  - Metadata is stored directly inside an internal, replicated Kafka topic (`@metadata`).
  - Dedicated controller brokers form a **Raft consensus quorum**.
  - Scales to **millions of partitions** with near-instantaneous controller failover recovery.

---

## 2. Consumer Rebalance Protocols: Eager vs. Cooperative Sticky

```
 EAGER REBALANCE (Legacy Stop-the-World):
 [ Consumer A Revokes ALL ] [ Consumer B Revokes ALL ] ---> Reassignment Storm ---> Resume

 COOPERATIVE STICKY ASSIGNOR (Incremental):
 [ Consumer A keeps P0, P1 ] [ Consumer B gives up P3 ONLY ] ---> No global pause!
```

### Q2. Why is `CooperativeStickyAssignor` recommended for production?
**Answer:**
- **Eager Assignor (Default in old versions)**: When a consumer joins/leaves, **all consumers in the group revoke all assigned partitions** and pause consumption (*Stop-The-World* event).
- **Cooperative Sticky Assignor (`CooperativeStickyAssignor`)**:
  - Rebalances partitions incrementally in 2 phases.
  - Unaffected consumers continue processing their existing partitions without interruption.
  - Prevents massive **Rebalance Storms** during Kubernetes pod rolling restarts!

```properties
# application.properties
spring.kafka.consumer.properties.partition.assignment.strategy=org.apache.kafka.clients.consumer.CooperativeStickyAssignor
```

---

## 3. Zero Data Loss Producer & Consumer Configurations

### Q3. What configurations guarantee Zero Data Loss and Exactly-Once Semantics?
**Answer:**

#### 1. Producer Reliability:
```properties
# Minimum replicas that must acknowledge write
spring.kafka.producer.acks=all
spring.kafka.producer.properties.enable.idempotence=true # Prevents broker duplicate writes via sequence numbers
spring.kafka.producer.retries=2147483647
spring.kafka.producer.properties.max.in.flight.requests.per.connection=5
# Broker side setting:
# min.insync.replicas=2 (Topic with replication factor 3 requires leader + 1 follower ACK)
```

#### 2. Consumer Reliability:
```properties
spring.kafka.consumer.enable-auto-commit=false # Manual ACK only after DB write succeeds
spring.kafka.consumer.properties.isolation.level=read_committed # Read only committed transactional messages
spring.kafka.consumer.auto-offset-reset=earliest
```

---

## 4. Non-Blocking Retries & Dead Letter Topics (DLT)

```java
@Service
public class OrderEventConsumer {
    private static final Logger log = LoggerFactory.getLogger(OrderEventConsumer.class);

    // Non-blocking topic-based retry: orders.created-retry-1000, orders.created-retry-2000, orders.created-dlt
    @RetryableTopic(
        attempts = "4",
        backoff = @Backoff(delay = 1000, multiplier = 2.0, maxDelay = 10000),
        dltStrategy = DltStrategy.FAIL_ON_ERROR,
        include = { TransientNetworkException.class, TimeoutException.class }
    )
    @KafkaListener(topics = "orders.created", groupId = "inventory-service")
    public void processOrder(OrderCreatedEvent event, Acknowledgment ack) {
        inventoryService.reserveStock(event);
        ack.acknowledge(); // Commit offset manually
    }

    // Terminal poison-pill handler
    @DltHandler
    public void handleDlt(OrderCreatedEvent event, @Header(KafkaHeaders.EXCEPTION_MESSAGE) String error) {
        log.error("Event {} routed to DEAD LETTER TOPIC (DLT). Error: {}", event.orderId(), error);
        alertService.triggerPagerDuty("Kafka DLT Alert", event);
    }
}
```

---

> **Next Chapter**: [25 RabbitMQ vs. Apache Kafka Messaging](25_RabbitMQ_vs_Kafka_Messaging.md)

