# Distributed Systems, Microservices Patterns & Cloud

> **Navigation**: [Master Index](README.md) | [Previous: Security & Observability](07_Security_Testing_Observability.md) | [Next: System Design & Senior Scenarios](09_System_Design_Senior_Scenarios.md)

---

## 1. Distributed Transaction Patterns

### Q1. Deep Dive: Saga Pattern — Orchestration vs. Choreography.
**Answer:**
Traditional ACID 2-Phase Commit (2PC / XA) creates tight coupling, locks database rows across network calls, and creates single points of failure. The **Saga Pattern** manages distributed transactions as a sequence of local database transactions.

```
[ CHOREOGRAPHY SAGA (Event-Driven) ]
[ Order Service ] --(OrderCreated)--> [ Payment Service ] --(PaymentProcessed)--> [ Inventory Service ]
      ^                                     | (Fails!)
      +--------------(PaymentFailed)--------+ (Triggers Compensating Transaction: Cancel Order)

[ ORCHESTRATION SAGA (Central Coordinator) ]
                     +---------------------------+
                     | Order Saga Orchestrator   |
                     +---------------------------+
                       /           |           \
           1. Create  /  2. Charge |  3. Reserve\
                     v             v             v
             [Order Service] [Payment Service] [Inventory Service]
```

| Dimension | Choreography Saga (Event-Driven) | Orchestration Saga (State Machine) |
| :--- | :--- | :--- |
| **Coordination** | Decentralized (Services listen to Kafka events) | Central orchestrator (State machine service) |
| **Complexity** | Simple for 2–3 services; confusing at scale | Easy to track, visualize, and debug complex flows |
| **Coupling** | Loosely coupled | Orchestrator knows all participating services |
| **Deadlock Risk** | Cyclic event loops possible | Controlled centralized state machine |

---

### Q2. Deep Dive: The Transactional Outbox Pattern + Debezium CDC.
**Answer:**
- **The Dual-Write Problem**: When an application writes to a SQL database AND publishes an event to Kafka, one operation can fail while the other succeeds, causing catastrophic data inconsistency.
- **The Solution (Transactional Outbox)**: Save the business entity and the outgoing event in the **same local database transaction**.

```
+-----------------------------------------------------------------------------------+
|                        TRANSACTIONAL OUTBOX PATTERN                               |
+-----------------------------------------------------------------------------------+
|  [ Spring Boot Service ]                                                          |
|         |                                                                         |
|         | 1. Begin Local DB Transaction (@Transactional)                          |
|         +---> INSERT INTO orders (...)                                            |
|         +---> INSERT INTO outbox_events (id, aggregate_type, payload, status)    |
|         | 2. Commit Transaction (Guaranteed Atomic!)                              |
|         v                                                                         |
|  [ PostgreSQL Database ]                                                          |
|         | (WAL - Write Ahead Log)                                                 |
|         v                                                                         |
|  [ Debezium CDC Engine ] (Reads DB WAL asynchronously)                            |
|         |                                                                         |
|         v                                                                         |
|  [ Apache Kafka Topic: orders.events ]                                            |
+-----------------------------------------------------------------------------------+
```

---

### Q3. Distributed Locking with Redis & Redisson.
**Answer:**
In a clustered microservice deployment, Java's `synchronized` and `ReentrantLock` only protect a single JVM. To coordinate across multiple server nodes, use a **Distributed Lock**.

```java
@Service
public class FlashSaleService {
    private final RedissonClient redisson;
    private final InventoryRepository inventoryRepo;

    public FlashSaleService(RedissonClient redisson, InventoryRepository inventoryRepo) {
        this.redisson = redisson;
        this.inventoryRepo = inventoryRepo;
    }

    public boolean purchaseItem(String itemId, String userId) {
        RLock lock = redisson.getLock("lock:item:" + itemId);
        try {
            // waitTime = 5s (wait to acquire), leaseTime = 10s (auto-release after 10s watchdog)
            if (lock.tryLock(5, 10, TimeUnit.SECONDS)) {
                try {
                    // Critical section: Guaranteed single-node execution across cluster
                    Inventory item = inventoryRepo.findById(itemId).orElseThrow();
                    if (item.getAvailableStock() > 0) {
                        item.decrementStock();
                        inventoryRepo.save(item);
                        return true;
                    }
                    return false;
                } finally {
                    lock.unlock();
                }
            } else {
                throw new RateLimitException("Server busy, please try again");
            }
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            throw new RuntimeException("Lock interrupted", e);
        }
    }
}
```

---

## 2. Docker Multi-Stage Builds & Spring Boot LayerTools

### Q4. How do you write an optimal, secure Dockerfile for Spring Boot?
**Answer:**
Using Spring Boot **LayerTools** separates changing application classes from static third-party JARs, allowing Docker to cache 95% of the image layers during rebuilds!

```dockerfile
# Stage 1: Extract JAR layers
FROM eclipse-temurin:21-jre-alpine AS builder
WORKDIR /builder
ARG JAR_FILE=target/*.jar
COPY ${JAR_FILE} app.jar
RUN java -Djarmode=layertools -jar app.jar extract

# Stage 2: Minimal, secure runtime container
FROM eclipse-temurin:21-jre-alpine
WORKDIR /app

# Run as non-root user for security compliance
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser:appgroup

# Copy layers in order of least-frequent to most-frequent changes
COPY --from=builder /builder/dependencies/ ./
COPY --from=builder /builder/spring-boot-loader/ ./
COPY --from=builder /builder/snapshot-dependencies/ ./
COPY --from=builder /builder/application/ ./

# Memory-aware JVM container settings
ENV JAVA_OPTS="-XX:+UseContainerSupport -XX:MaxRAMPercentage=75.0 -XX:+UseG1GC"

EXPOSE 8080
ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS org.springframework.boot.loader.launch.JarLauncher"]
```

---

## 3. Kubernetes Deployment & Health Probes

### Q5. Kubernetes Deployment Manifest with Liveness, Readiness, and Resource Limits.
**Answer:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
  labels:
    app: order-service
spec:
  replicas: 3
  selector:
    matchLabels:
      app: order-service
  template:
    metadata:
      labels:
        app: order-service
    spec:
      containers:
        - name: order-service
          image: myregistry.io/order-service:1.0.0
          ports:
            - containerPort: 8080
          resources:
            requests:
              memory: "1Gi"
              cpu: "500m"
            limits:
              memory: "2Gi"
              cpu: "2000m"
          # Liveness Probe: Restarts container if deadlocked/hung
          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 10
            failureThreshold: 3
          # Readiness Probe: Routes network traffic only when warm
          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: 8080
            initialDelaySeconds: 20
            periodSeconds: 5
            failureThreshold: 2
          env:
            - name: SPRING_PROFILES_ACTIVE
              value: "prod"
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: db-secret
                  key: password
```

---

> **Next Chapter**: [09 System Design & Senior Scenarios](09_System_Design_Senior_Scenarios.md)
