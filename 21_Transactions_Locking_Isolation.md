# 21. Transactions, Locking & Database Isolation Levels

> **Navigation**: [Master Index](README.md) | [Previous: Database Internals](20_Database_Internals_SQL_Tuning.md) | [Next: Database Migrations (Flyway)](22_Database_Migrations_Flyway_Liquibase.md)

---

## 📌 Chapter Overview
This module explores **Database Transaction Isolation Levels**, concurrency anomalies (Dirty Read, Non-Repeatable Read, Phantom Read, Write Skew), **Optimistic vs. Pessimistic Locking**, Spring `@Transactional` propagation rules, and **HikariCP pool sizing**.

---

## 1. Database Isolation Levels & Concurrency Anomalies

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

### Q1. Define the 4 Concurrency Anomalies.
**Answer:**
1. **Dirty Read**: Transaction A reads data modified by uncommitted Transaction B. If B rolls back, A processed corrupt/phantom data.
2. **Non-Repeatable Read**: Transaction A reads a row. Transaction B **updates** that row and commits. Transaction A re-reads the row and observes changed values.
3. **Phantom Read**: Transaction A executes a range query (`SELECT WHERE age > 30`). Transaction B **inserts** a new row matching the range and commits. Transaction A re-runs query and sees new "phantom" rows.
4. **Write Skew (Snapshot Isolation)**: Two concurrent transactions read overlapping data sets, make decisions based on what they read, and perform disjoint writes that violate a business invariant (e.g., both doctors request off-call at the same time thinking the other is on duty).

---

## 2. Optimistic vs. Pessimistic Locking

### Q2. Compare Optimistic vs. Pessimistic Locking with Spring Data JPA.
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

## 3. Spring `@Transactional` Propagation Rules

```
+-------------------------------------------------------------------------------+
|                       SPRING TRANSACTION PROPAGATION TYPES                    |
+-------------------------------------------------------------------------------+
| 1. REQUIRED (Default) -> Join active transaction; create new if none exists.  |
| 2. REQUIRES_NEW       -> Always suspend active transaction & create new one.  |
| 3. NESTED             -> Execute inside nested savepoint (partial rollback).  |
| 4. MANDATORY          -> Must run in active transaction; else throw exception.|
| 5. SUPPORTS           -> Run in transaction if present; else non-transactional|
| 6. NOT_SUPPORTED      -> Always run non-transactionally; suspend if present.  |
| 7. NEVER              -> Throw exception if transaction exists.               |
+-------------------------------------------------------------------------------+
```

### Q3. What is the rollback contract of Spring `@Transactional`?
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

## 4. HikariCP Connection Pool Sizing Formula

### Q4. How do you size HikariCP database connection pool for production?
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

