# 21. Transactions, Propagation Types, Locking & Isolation Levels

> **Navigation**: [Master Index](README.md) | [Previous: Database Internals](20_Database_Internals_SQL_Tuning.md) | [Next: Database Migrations (Flyway)](22_Database_Migrations_Flyway_Liquibase.md)

---

## 📌 Chapter Overview
This module explores **Spring Transaction Management**, the complete breakdown of all **7 `@Transactional` Propagation Types**, **4 Database Isolation Levels** & concurrency anomalies, **Optimistic vs. Pessimistic Locking**, `readOnly = true` optimizations, and **HikariCP pool sizing**.

---

## 1. The 7 Spring `@Transactional` Propagation Types

```
+-----------------------------------------------------------------------------------+
|                        SPRING TRANSACTION PROPAGATION MATRIX                      |
+-----------------------------------------------------------------------------------+
| Propagation Type   | If Transaction Exists?             | If NO Transaction Exists?|
| ------------------ | ---------------------------------- | ------------------------ |
| **REQUIRED** (Def) | **Joins** current transaction      | **Creates** a new one    |
| **REQUIRES_NEW**   | **Suspends** current; creates NEW  | **Creates** a new one    |
| **NESTED**         | Creates a **Savepoint** in current | **Creates** a new one    |
| **MANDATORY**      | **Joins** current transaction      | **Throws Exception!**    |
| **SUPPORTS**       | **Joins** current transaction      | Runs non-transactionally |
| **NOT_SUPPORTED**  | **Suspends** current transaction   | Runs non-transactionally |
| **NEVER**          | **Throws Exception!**              | Runs non-transactionally |
+-----------------------------------------------------------------------------------+
```

### Q1. Deep Dive: Compare `REQUIRED`, `REQUIRES_NEW`, and `NESTED`.
**Answer:**

#### 1. `REQUIRED` (Default):
Both parent and child method share the **same physical transaction**. If child throws an uncaught exception, the entire transaction is marked as `rollback-only`. The parent *cannot* catch the exception to commit its own work!

#### 2. `REQUIRES_NEW`:
Parent transaction is **suspended**. Child method starts an **independent physical transaction** with its own database connection.
- If child commits, its data is permanently committed regardless of whether parent subsequently fails!
- *Common Use Case*: Audit logging, security attempt logging (must be persisted even if business transaction fails).

```
 [ Parent Tx 1 (Suspended) ] ----> [ Child Tx 2 (Active on separate DB conn) ]
                                          |
                                    Child Commits!
                                          |
 [ Parent Tx 1 (Resumed) ] <--------------+
```

#### 3. `NESTED`:
Uses a **Single physical transaction with JDBC Savepoints**.
- If the nested method fails, execution rolls back **only to the savepoint**. The parent can catch the exception and commit other work!
- *Common Use Case*: Optional order bonus points or non-critical secondary billing attempts.

---

## 2. Database Isolation Levels & Concurrency Anomalies

```
+-----------------------------------------------------------------------------------+
|                        TRANSACTION ISOLATION LEVEL MATRIX                         |
+-----------------------------------------------------------------------------------+
| Isolation Level   | Dirty Read | Non-Repeatable Read | Phantom Read | Write Skew  |
| ----------------- | :--------: | :-----------------: | :----------: | :---------: |
| READ UNCOMMITTED  |    YES     |         YES         |     YES      |     YES     |
| READ COMMITTED    |     NO     |         YES         |     YES      |     YES     |
| REPEATABLE READ   |     NO     |          NO         |   NO (MVCC)  |     YES     |
| SERIALIZABLE      |     NO     |          NO         |      NO      |      NO     |
+-----------------------------------------------------------------------------------+
```

### Q2. Define the 4 Concurrency Anomalies.
**Answer:**
1. **Dirty Read**: Transaction A reads data modified by uncommitted Transaction B. If B rolls back, A processed corrupt/phantom data.
2. **Non-Repeatable Read**: Transaction A reads a row. Transaction B **updates** that row and commits. Transaction A re-reads the row and observes changed values.
3. **Phantom Read**: Transaction A executes a range query (`SELECT WHERE age > 30`). Transaction B **inserts** a new row matching the range and commits. Transaction A re-runs query and sees new "phantom" rows.
4. **Write Skew (Snapshot Isolation)**: Two concurrent transactions read overlapping data sets, make decisions based on what they read, and perform disjoint writes that violate a business invariant (e.g., both doctors request off-call simultaneously).

---

## 3. The `readOnly = true` Performance Optimization

### Q3. Why should you always use `@Transactional(readOnly = true)` for query methods?
**Answer:**
1. **Hibernate Snapshot Optimization**: Hibernate **disables dirty checking snapshots**, reducing JVM memory consumption and CPU comparison overhead.
2. **FlushMode.MANUAL**: Hibernate sets FlushMode to `MANUAL`, ensuring `em.flush()` is never triggered unnecessarily.
3. **Database Read-Replica Routing**: In master-replica database architectures (AWS Aurora / Postgres Replicas), Spring's routing DataSource routes `readOnly=true` transactions to read-only follower nodes, offloading the primary master database!

---

## 4. Rollback Rules & Transaction Gotchas

### Q4. What is the rollback contract of Spring `@Transactional`?
**Answer:**
- By default, Spring rolls back transactions **ONLY for Unchecked Exceptions (`RuntimeException` and `Error`)**.
- Checked exceptions (`IOException`, `SQLException`, custom checked exceptions) do **NOT** trigger rollback unless explicitly declared via `rollbackFor`:

```java
@Transactional(rollbackFor = Exception.class, propagation = Propagation.REQUIRED)
public void executeTransfer(TransferRequest req) throws PaymentException {
    accountRepo.debit(req.sourceId(), req.amount());
    accountRepo.credit(req.targetId(), req.amount());
}
```

---

## 5. Optimistic vs. Pessimistic Locking

### Q5. Compare Optimistic vs. Pessimistic Locking with Spring Data JPA.
**Answer:**

```java
// 1. OPTIMISTIC LOCKING: Best for low-contention, read-heavy workloads
@Entity
public class ProductInventory {
    @Id private Long id;
    private int stockQuantity;

    @Version // Automatically increments version on every UPDATE. Throws OptimisticLockException on conflict!
    private Long version;
}

// 2. PESSIMISTIC LOCKING: Best for high-contention, high-value operations (Flash sales / Banking)
public interface InventoryRepository extends JpaRepository<ProductInventory, Long> {
    
    // Generates SQL: 'SELECT * FROM product_inventory WHERE id = ? FOR UPDATE'
    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query("SELECT p FROM ProductInventory p WHERE p.id = :id")
    Optional<ProductInventory> findByIdWithLock(@Param("id") Long id);
}
```

---

## 6. HikariCP Connection Pool Sizing Formula

### Q6. How do you size HikariCP database connection pool for production?
**Answer:**
A common pitfall is configuring huge connection pools (e.g., 200 connections), which overwhelms database disk I/O and causes CPU context-switching degradation.

$$\text{Optimal Pool Size} = (\text{DB CPU Cores} \times 2) + \text{Effective Spindles (SSD = 1)}$$

*Example: A 4-core database server performs best with only ~10 to 15 connections!*

```properties
# Optimal Production HikariCP Settings
spring.datasource.hikari.maximum-pool-size=15
spring.datasource.hikari.minimum-idle=10
spring.datasource.hikari.idle-timeout=300000
spring.datasource.hikari.connection-timeout=20000     # 20s timeout before failing
spring.datasource.hikari.max-lifetime=1800000         # 30m connection recycling
spring.datasource.hikari.leak-detection-threshold=20000 # Warns if connection held > 20s
```

---

> **Next Chapter**: [22 Database Schema Migrations (Flyway & Liquibase)](22_Database_Migrations_Flyway_Liquibase.md)
