# 37. Quick Revision Checklist & Pre-Interview Cheat Sheet

> **Navigation**: [Master Index](README.md) | [Previous: Spring AI Guide](36_Spring_AI_Generative_AI_Guide.md) | [Study Roadmap](00_Study_Roadmap.md)

---

## 📌 Chapter Overview
The **Ultimate 1-Page Pre-Interview Cheat Sheet** for fast review 1 hour before an interview. Contains essential formulas, core annotation tables, and **15 rapid-fire senior gotchas**.

---

## 1. Essential Formulas & Numbers Every Senior Engineer Must Know

```
+-----------------------------------------------------------------------------------+
|                        SENIOR ENGINEERING FORMULA CHEAT SHEET                     |
+-----------------------------------------------------------------------------------+
| 1. HikariCP Pool Sizing:   Pool Size = (CPU Cores * 2) + Effective Spindles (SSD=1)
| 2. CPU-Bound Thread Pool:  Pool Size = CPU Cores + 1                              |
| 3. I/O-Bound Thread Pool:  Pool Size = CPU Cores * (1 + WaitTime / ServiceTime)   |
| 4. Little's Law:           L = λ * W (Avg Items in System = Arrival Rate * Latency)
| 5. HashMap Index Formula:  Index = (Capacity - 1) & Hash                          |
| 6. HashMap Treeification:  Bucket Size >= 8 AND Total Capacity >= 64              |
+-----------------------------------------------------------------------------------+
```

---

## 2. Top 30 Spring Boot Annotations Quick Reference

| Annotation | Core Purpose & Scope |
| :--- | :--- |
| **`@SpringBootApplication`** | Combines `@Configuration`, `@EnableAutoConfiguration`, `@ComponentScan`. |
| **`@Component` / `@Service` / `@Repository`** | Stereotype annotations for Spring-managed singleton beans. |
| **`@ConfigurationProperties`**| Type-safe external configuration binding with validation. |
| **`@Transactional`** | Wraps method inside a database transaction with declarative propagation. |
| **`@TransactionalEventListener`** | Executes event listener ONLY after DB transaction commits (`AFTER_COMMIT`). |
| **`@PreAuthorize`** | Method-level security evaluation using Spring Expression Language (SpEL). |
| **`@RetryableTopic`** | Non-blocking topic-based Kafka retries with Dead Letter Topic (DLT). |
| **`@EntityGraph`** | Resolves JPA N+1 query problem by generating SQL dynamic joins. |
| **`@BatchSize`** | Batches Hibernate lazy loading queries using SQL `IN (?, ?, ...)` clauses. |
| **`@Version`** | Enables optimistic locking on JPA entities. |
| **`@Lock(PESSIMISTIC_WRITE)`**| Triggers database pessimistic locking (`SELECT ... FOR UPDATE`). |
| **`@RestClient` / `@HttpExchange`**| Declarative and fluent HTTP client interfaces in Spring Boot 3. |

---

## 3. 15 Rapid-Fire Senior Interview Gotchas

1. **Integer Cache Trap**: `Integer a = 100, b = 100; a == b` is `true`. `Integer x = 200, y = 200; x == y` is `false` (cached only from `-128` to `127`).
2. **Spring AOP Self-Invocation**: Calling `this.methodB()` inside the same class **bypasses the AOP proxy**, so `@Transactional` and `@Async` silently fail.
3. **`@Transactional` Rollback Default**: Rolls back **ONLY for Unchecked Exceptions (`RuntimeException`)**. Use `rollbackFor = Exception.class` for checked exceptions.
4. **ThreadLocal Memory Leaks**: Thread pools reuse threads; `ThreadLocalMap` entries have strong value references that stay in memory forever unless `.remove()` is called in a `finally` block.
5. **Virtual Thread Pinning**: Executing inside `synchronized` blocks pins OS carrier threads. Replace `synchronized` with `ReentrantLock`.
6. **Never Pool Virtual Threads**: Create a new virtual thread per task with `Executors.newVirtualThreadPerTaskExecutor()`.
7. **Open Session in View (OSIV) Anti-Pattern**: Keeps DB connections open across the entire HTTP request rendering, exhausting `HikariCP` connection pools. Disable it (`spring.jpa.open-in-view=false`).
8. **Leftmost Prefix Rule**: A composite index on `(A, B, C)` cannot be used for queries filtering *only* on `B` or `C`.
9. **Kafka Eager Rebalance**: Causes global Stop-The-World pauses. Use `CooperativeStickyAssignor` for non-disruptive rolling updates.
10. **Dual-Write Danger**: Never call `orderRepo.save()` and `kafkaTemplate.send()` directly in the same method. Use the **Transactional Outbox Pattern + Debezium CDC**.
11. **Deep Pagination**: `LIMIT 10 OFFSET 1000000` is slow because DB scans 1M rows. Use **Seek/Keyset Pagination** (`WHERE id > last_seen_id LIMIT 10`).
12. **Cache Avalanche**: Stagger key expiration with randomized TTL jitter.
13. **Cache Penetration**: Protect against queries for non-existent keys with a **Bloom Filter**.
14. **Java is Strictly Pass-by-Value**: Object references are copied by value; reassigning the reference does not change the caller's pointer.
15. **`finally` Block Return Trap**: A `return` statement in a `finally` block will silently swallow any exception thrown in the `try` block.

---

> **Return to Master Index**: [Master Repository Index](README.md)

