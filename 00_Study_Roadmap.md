# 00. 6-Week Study Roadmap & Senior Engineering Competency Grid

> **Navigation**: [Master Index](README.md) | [Next: Java Basics Foundations](01_Java_Basics_Foundations.md)

---

## 🎯 Strategic Study Blueprint for Senior / Lead Engineers (8+ Years Experience)

This roadmap is engineered to take you from core JVM & language mechanics through to large-scale distributed system design and technical leadership.

```
+-----------------------------------------------------------------------------------+
|                        6-WEEK SENIOR PREPARATION BLUEPRINT                        |
+-----------------------------------------------------------------------------------+
|  🟦 WEEK 1: Core Java, Memory, Concurrency & JVM Deep Dive (Modules 01 - 08)     |
|  🟩 WEEK 2: JVM Diagnostics, Dumps, Troubleshooting & Coding (Modules 09 - 11)   |
|  🟧 WEEK 3: SOLID, Spring Core, Boot 3 Internals & Security (Modules 12 - 16)    |
|  🟪 WEEK 4: Batch, JPA, Database Indexing & SQL Tuning (Modules 17 - 23)        |
|  🟫 WEEK 5: Kafka, RabbitMQ, Microservices & Cloud (Modules 24 - 30)              |
|  🟥 WEEK 6: System Design, Incidents, Leadership & AI (Modules 31 - 37)           |
+-----------------------------------------------------------------------------------+
```

---

## 📅 Week-by-Week Curriculum

### 🟦 Week 1: Core Java, Collections, Concurrency & Loom
- **[01 Java Basics & Core Foundations](01_Java_Basics_Foundations.md)**: Pass-by-value, memory layout, `equals()` & `hashCode()`, immutability.
- **[02 Strings & Memory Layout](02_Strings_Memory_Algorithms.md)**: Compact strings, String Constant Pool, string concatenation optimizations.
- **[03 Collections Framework & Generics](03_Collections_Framework_Generics.md)**: `HashMap` & `ConcurrentHashMap` internals, PECS rule, Sequenced Collections.
- **[04 Modern Java Features (8–21 LTS)](04_Modern_Java_Features_8_to_21.md)**: Streams, Records, Sealed Classes, Pattern Matching for switch.
- **[05 Java I/O, NIO & Netty](05_Java_IO_NIO_Netty.md)**: Channels, Buffers, Zero-Copy (`transferTo`), Netty EventLoops.
- **[06 Multithreading & JMM](06_Multithreading_JMM_Concurrency.md)**: JMM, Happens-Before, volatile, lock escalation, `ThreadLocal` leaks.
- **[07 Concurrency Primitives & Executors](07_Concurrency_Primitives_Executors.md)**: `CountDownLatch`, `Semaphore`, `ThreadPoolExecutor`, `CompletableFuture`.
- **[08 Virtual Threads (Project Loom)](08_Virtual_Threads_Loom.md)**: Carrier threads, thread pinning (`synchronized` vs `ReentrantLock`), unmounting.

### 🟩 Week 2: JVM Diagnostics, Dumps & Coding
- **[09 JVM Architecture, JIT & GC](09_JVM_Architecture_JIT_GC.md)**: ClassLoaders, Tiered Compilation, Escape Analysis, G1 GC vs ZGC.
- **[10 JVM Troubleshooting & Memory Dumps](10_JVM_Troubleshooting_Dumps_Profiling.md)**: 6 OOM types, Thread Dumps (BLOCKED vs WAITING), Heap Dumps in MAT, K8s flags.
- **[11 Coding, Data Structures & Concurrency Challenges](11_Coding_Data_Structures_Problems.md)**: Custom BlockingQueue, Thread-Safe Singleton, LRU Cache, Stream transformations.

### 🟧 Week 3: Software Design, Spring Framework & Security
- **[12 SOLID Principles & Design Patterns](12_SOLID_Principles_Design_Patterns.md)**: OCP with Strategy, GoF patterns in Spring.
- **[13 Spring Core & AOP Internals](13_Spring_Core_IoC_AOP_Internals.md)**: Bean lifecycle, 3-level cache circular dependencies, AOP self-invocation.
- **[14 Spring Boot 3 Internals & Tomcat](14_Spring_Boot_3_Internals_Actuator.md)**: Auto-configuration (`AutoConfiguration.imports`), Tomcat sizing, Micrometer metrics.
- **[15 Web, REST & Reactive APIs](15_Spring_Web_REST_APIs.md)**: RFC 7807 `ProblemDetail`, `RestClient`, `@HttpExchange`, WebFlux.
- **[16 Spring Security 6 & OAuth2](16_Spring_Security_6_OAuth2.md)**: Stateless JWT filter chain, Keycloak OIDC PKCE, `@PreAuthorize` SpEL, multi-tenancy.

### 🟪 Week 4: Persistence, SQL Tuning & Caching
- **[17 Spring Batch & ETL](17_Spring_Batch_ETL_Processing.md)**: Chunk-oriented processing, Step partitioning, skip/retry policies.
- **[18 Spring Modulith & Events](18_Spring_Modulith_Events.md)**: Modular monolith boundaries, `@TransactionalEventListener(AFTER_COMMIT)`.
- **[19 Hibernate & JPA Internals](19_Hibernate_JPA_Internals.md)**: Entity lifecycle, dirty checking, N+1 query 3 fixes, OSIV anti-pattern.
- **[20 Database Internals & SQL Tuning](20_Database_Internals_SQL_Tuning.md)**: B-Tree indexes, Leftmost Prefix rule, `EXPLAIN ANALYZE`, Seek pagination.
- **[21 Transactions & Locking](21_Transactions_Locking_Isolation.md)**: `@Transactional` rollback, Isolation anomalies (Write Skew), HikariCP pool sizing.
- **[22 Database Schema Migrations](22_Database_Migrations_Flyway_Liquibase.md)**: Expand-Contract zero-downtime migrations, Flyway vs Liquibase.
- **[23 Redis Caching & NoSQL](23_Redis_Caching_Patterns_NoSQL.md)**: Cache penetration, thundering herd, avalanche, Redis `ZSET`, Cassandra modeling.

### 🟫 Week 5: Messaging, Microservices & Cloud
- **[24 Apache Kafka Production Engineering](24_Apache_Kafka_Production_Engineering.md)**: Partitions, Cooperative Sticky Assignor, Exactly-Once Transactions, DLT.
- **[25 RabbitMQ vs. Apache Kafka](25_RabbitMQ_vs_Kafka_Messaging.md)**: AMQP exchanges, DLX, RabbitMQ vs Kafka decision matrix.
- **[26 Microservices Architecture & DDD](26_Microservices_DDD_Architecture.md)**: Tactical DDD (Aggregates, Value Objects), CQRS/Event Sourcing, Strangler Fig.
- **[27 Distributed Transactions & Outbox](27_Distributed_Transactions_Saga_Outbox.md)**: Saga Orchestration vs Choreography, Outbox + Debezium CDC, Redisson locks.
- **[28 Spring Cloud Gateway & Resilience4j](28_Spring_Cloud_Gateway_Resilience.md)**: Gateway filters, Circuit Breakers, Bulkheads, Rate Limiters.
- **[29 Observability & Integration Testing](29_Observability_Tracing_Testing.md)**: OpenTelemetry distributed tracing, MDC logs, Testcontainers.
- **[30 Docker & Kubernetes](30_Cloud_Docker_Kubernetes.md)**: Multi-stage builds, Spring Boot LayerTools, K8s Liveness/Readiness probes, graceful shutdown.

### 🟥 Week 6: System Design, Incidents, Leadership & AI
- **[31 Senior System Design Scenarios](31_System_Design_High_Scale_Scenarios.md)**: Flash Sale overselling prevention, Task Scheduler (`ZSET`), Payment Idempotency, gRPC vs REST.
- **[32 Production Incident Post-Mortems](32_Production_Incident_PostMortems.md)**: HikariCP connection starvation, 100% CPU thread dumps, Netty off-heap leaks.
- **[33 Enterprise Security & OWASP](33_Enterprise_Security_OWASP_Java.md)**: Insecure Deserialization, SSRF, SQLi, Vault secrets, mTLS.
- **[34 Build Tools & CI/CD](34_Build_Tools_CI_CD_DevOps.md)**: Maven BOM dependency convergence, Blue-Green/Canary deployments, Feature Flags.
- **[35 Engineering Leadership & ADRs](35_Engineering_Leadership_ADRs_Clean_Code.md)**: Writing ADRs, Monolith modernization, Blameless 5 Whys RCA.
- **[36 Spring AI & Generative AI](36_Spring_AI_Generative_AI_Guide.md)**: ChatClient, RAG with Vector Stores, Tool / Function Calling.
- **[37 Quick Revision Checklist](37_Quick_Revision_Checklist.md)**: 1-Page Pre-Interview Cheat Sheet, Top 30 Annotations, 15 Senior Gotchas.

---

> **Begin Day 1**: [01 Java Basics & Core Foundations](01_Java_Basics_Foundations.md)
