# 4-Week Study Roadmap & Senior Engineering Readiness

> **Navigation**: [Master Index](README.md) | [Java Basics Foundations](00_Java_Basics_QA.md) | [Next: Modern Java & JVM](01_Modern_Java_JVM_Language.md)

---

## 🎯 Strategic Study Blueprint

This 4-week structured preparation roadmap is designed for engineers preparing for **Senior (L5/L6)** and **Tech Lead** technical rounds. It transitions systematically from foundational language mechanics and JVM internals to enterprise distributed architectures and live production debugging.

```
┌───────────────────────────────────────────────────────────────────────────┐
│  WEEK 1: Java Core, Modern Language (8-21), JVM Internals & Concurrency   │
│  WEEK 2: Spring Core, Spring Boot 3 Internals, REST APIs & Validation     │
│  WEEK 3: Data Persistence, Hibernate/JPA, Caching & Event-Driven Systems │
│  WEEK 4: Microservices Architecture, Cloud, Security & System Design      │
└───────────────────────────────────────────────────────────────────────────┘
```

---

## 📅 Week-by-Week Action Plan

### 🟦 Week 1: Java Foundations, JVM Architecture & Concurrency

> **Goal**: Master memory semantics, garbage collection tuning, concurrent synchronization, and problem solving in pure Java.

- [ ] **Day 1: OOP & Modern Java Syntax**
  - Four pillars of OOP, equals/hashCode contract, immutability principles.
  - Java 14–17: Records, Sealed Classes, Pattern Matching, Switch Expressions.
  - 📖 *Reading*: [00 Java Basics QA](00_Java_Basics_QA.md) & [01 Modern Java & JVM](01_Modern_Java_JVM_Language.md)

- [ ] **Day 2: Functional Programming & Streams Deep Dive**
  - Lambdas, Functional Interfaces (`Function`, `Consumer`, `Predicate`, `Supplier`).
  - Stream pipeline mechanics (Intermediate vs. Terminal, `Collectors.groupingBy`).
  - Dangers of Parallel Streams in web containers (Common ForkJoinPool starvation).
  - 📖 *Reading*: [01 Modern Java & JVM](01_Modern_Java_JVM_Language.md)

- [ ] **Day 3: JVM Architecture & Garbage Collection Tuning**
  - Memory Areas: Heap (Eden, Survivor, Old), Metaspace, Thread Stacks, Code Cache.
  - ClassLoaders: Delegation hierarchy, Loading, Linking, and Initialization.
  - JIT Compiler: C1, C2 Server Compiler, Tiered Compilation, Escape Analysis.
  - Garbage Collectors: G1 GC region layout vs. ZGC low-latency colored pointers.
  - 📖 *Reading*: [01 Modern Java & JVM](01_Modern_Java_JVM_Language.md)

- [ ] **Day 4: Collections & Generics Internals**
  - `HashMap` internals: Bitwise indexing `(n-1) & hash`, collision chaining, Red-Black treeification (threshold 8/6).
  - `ConcurrentHashMap` internals: CAS node insertion + synchronized bin head locking.
  - `LinkedHashMap` access-order and building an LRU cache.
  - Generics: Type erasure and the PECS rule (`Producer Extends, Consumer Super`).
  - 📖 *Reading*: [02 Collections, Generics & Concurrency](02_Collections_Generics_Concurrency.md)

- [ ] **Day 5: Multithreading Foundations & Java Memory Model (JMM)**
  - JMM: CPU hardware caches, memory visibility, instruction reordering.
  - `volatile` keyword: Memory barriers and Happens-Before guarantees.
  - `synchronized` monitor locks and lock inflation (Biased $\rightarrow$ Lightweight $\rightarrow$ Heavyweight).
  - 📖 *Reading*: [02 Collections, Generics & Concurrency](02_Collections_Generics_Concurrency.md)

- [ ] **Day 6: Advanced Locks, Executors & Virtual Threads**
  - `ReentrantLock` vs. `StampedLock` (Optimistic read validation).
  - `ThreadPoolExecutor` sizing, bounded queues, and `CallerRunsPolicy`.
  - `CompletableFuture` asynchronous pipelines (`thenCompose`, `thenCombine`).
  - Java 21 Virtual Threads (Project Loom): Carrier threads and pinning caveats.
  - 📖 *Reading*: [02 Collections, Generics & Concurrency](02_Collections_Generics_Concurrency.md)

- [ ] **Day 7: Live Coding & Concurrency Practice**
  - Implement Thread-Safe Double-Checked Locking Singleton.
  - Implement Bounded Producer-Consumer with `ReentrantLock` & `Condition`.
  - Implement Custom LRU Cache ($O(1)$ `get`/`put`).
  - 📖 *Coding Practice*: [03 Coding Problem Solving](03_Coding_Problem_Solving.md)

---

### 🟩 Week 2: Spring Framework Core, Spring Boot 3 & Web Layer

> **Goal**: Understand Spring container internals, AOP proxy mechanics, auto-configuration resolution, and modern RESTful API design.

- [ ] **Day 8: Spring Core & Bean Lifecycle**
  - Inversion of Control (IoC) & Dependency Injection (Constructor vs. Field injection).
  - Complete 9-step Bean Lifecycle, `BeanPostProcessor` callbacks.
  - Bean Scopes (`singleton`, `prototype`, `request`, `session`) & Scoped Proxies.
  - 📖 *Reading*: [04 Spring Core & Boot Internals](04_Spring_Core_Boot_Internals.md)

- [ ] **Day 9: Spring AOP, Proxies & Event System**
  - AOP: Pointcuts, Advice, JointPoints.
  - JDK Dynamic Proxy (interfaces) vs. CGLIB (subclassing).
  - Self-invocation limitation (`this.method()` proxy bypass) and resolutions.
  - Application Events & `@TransactionalEventListener` (`AFTER_COMMIT`).
  - 📖 *Reading*: [04 Spring Core & Boot Internals](04_Spring_Core_Boot_Internals.md)

- [ ] **Day 10: Spring Boot 3 Auto-Configuration Internals**
  - `@SpringBootApplication` breakdown.
  - Auto-configuration resolution: `META-INF/.../AutoConfiguration.imports`.
  - Conditional Annotations (`@ConditionalOnClass`, `@ConditionalOnMissingBean`).
  - Type-safe `@ConfigurationProperties` with `@Validated` vs. `@Value`.
  - 📖 *Reading*: [04 Spring Core & Boot Internals](04_Spring_Core_Boot_Internals.md)

- [ ] **Day 11: Actuator, Tomcat Tuning & Observability**
  - Actuator endpoints: `/health`, `/metrics`, `/env`, `/threaddump`.
  - Embedded Tomcat tuning: `max-threads`, `accept-count`, `max-connections`.
  - Spring Boot 3 baseline: `jakarta.*` packages, GraalVM AOT native images.
  - 📖 *Reading*: [04 Spring Core & Boot Internals](04_Spring_Core_Boot_Internals.md)

- [ ] **Day 12: RESTful API Design & RFC 7807 Error Handling**
  - Richardson Maturity Model (Level 0 to Level 3 / HATEOAS).
  - HTTP Verbs, Idempotency, Safety, and correct Status Code mapping.
  - Global Exception Handling with `@RestControllerAdvice` & RFC 7807 `ProblemDetail`.
  - 📖 *Reading*: [05 Web, REST & Reactive](05_Web_REST_Reactive.md)

- [ ] **Day 13: Validation & Modern HTTP Clients**
  - Jakarta Bean Validation (JSR-380): Custom `@ConstraintValidator`.
  - Spring 6 `RestClient` vs. `WebClient` vs. `RestTemplate`.
  - Declarative HTTP Interfaces (`@HttpExchange`, `@GetExchange`).
  - 📖 *Reading*: [05 Web, REST & Reactive](05_Web_REST_Reactive.md)

- [ ] **Day 14: SOLID Principles & Design Patterns in Spring**
  - SOLID principles with modern Java examples.
  - GoF patterns in Spring: Factory, Builder, Strategy (Map injection), Decorator.
  - 📖 *Reading*: [12 SOLID & Design Patterns](12_SOLID_Design_Principles_Patterns.md)

---

### 🟧 Week 3: Persistence, Transactions, Caching & Messaging

> **Goal**: Eliminate database performance bottlenecks (N+1 queries), master transaction isolation/locking, distributed caching, and Kafka event streaming.

- [ ] **Day 15: Hibernate & JPA Entity Lifecycle**
  - Entity states: Transient, Managed/Persistent, Detached, Removed.
  - First-Level Cache (Session / PersistenceContext) & Dirty Checking.
  - Cascade types (`PERSIST`, `MERGE`, `REMOVE`) vs. `orphanRemoval = true`.
  - 📖 *Reading*: [06 Persistence, Transactions & Messaging](06_Persistence_Transactions_Messaging.md)

- [ ] **Day 16: Hibernate Performance & The N+1 Problem**
  - Root cause of N+1 lazy queries.
  - Three production fixes: `JOIN FETCH` (JPQL), `@EntityGraph`, and `@BatchSize`.
  - Cartesian Product problem with multiple collection fetches.
  - 📖 *Reading*: [06 Persistence, Transactions & Messaging](06_Persistence_Transactions_Messaging.md)

- [ ] **Day 17: Database Transactions & Concurrency Locking**
  - ACID properties and Transaction Isolation Levels (`READ_COMMITTED`, `REPEATABLE_READ`).
  - Spring `@Transactional` propagation (`REQUIRED`, `REQUIRES_NEW`, `NESTED`).
  - Rollback rules: Unchecked vs. Checked (`rollbackFor = Exception.class`).
  - Optimistic Locking (`@Version`) vs. Pessimistic Locking (`PESSIMISTIC_WRITE`).
  - 📖 *Reading*: [06 Persistence, Transactions & Messaging](06_Persistence_Transactions_Messaging.md)

- [ ] **Day 18: HikariCP Connection Pool Sizing & DB Migrations**
  - HikariCP pool sizing formula: $(\text{CPU Cores} \times 2) + \text{Spindle Count}$.
  - Database schema versioning with Flyway vs. Liquibase.
  - SQL Indexing strategies (B-Tree, Composite, Covering Index) & `EXPLAIN ANALYZE`.
  - 📖 *Reading*: [06 Persistence, Transactions & Messaging](06_Persistence_Transactions_Messaging.md)

- [ ] **Day 19: Distributed Caching with Redis**
  - Cache topologies: Cache-Aside (Lazy Loading) vs. Write-Through vs. Write-Behind.
  - Spring Cache abstraction (`@Cacheable`, `@CachePut`, `@CacheEvict`).
  - Cache failure modes: Cache Penetration (Bloom filters), Cache Breakdown (Mutex locks), Cache Avalanche (TTL jitter).
  - 📖 *Reading*: [06 Persistence, Transactions & Messaging](06_Persistence_Transactions_Messaging.md)

- [ ] **Day 20: Apache Kafka Deep Dive**
  - Topics, Partitions, Offsets, Consumer Groups, and sticky rebalancing.
  - Producer reliability: `acks=all`, `enable.idempotence=true`, retries.
  - Non-blocking retries with `@RetryableTopic` and Dead Letter Topics (DLT).
  - 📖 *Reading*: [06 Persistence, Transactions & Messaging](06_Persistence_Transactions_Messaging.md)

- [ ] **Day 21: RabbitMQ & Message Broker Comparison**
  - AMQP core concepts: Exchanges (Direct, Fanout, Topic), Queues, Bindings.
  - Dead Letter Exchanges (DLX), manual ACKs (`basicAck`, `basicNack`).
  - 📖 *Reading*: [06 Persistence, Transactions & Messaging](06_Persistence_Transactions_Messaging.md)

---

### 🟪 Week 4: Security, Microservices, Cloud & System Design

> **Goal**: Master distributed transaction patterns (Saga, Outbox), containerization, Spring Security 6, High-Level System Design, and live incident troubleshooting.

- [ ] **Day 22: Spring Security 6 & JWT Architecture**
  - `SecurityFilterChain` component-based configuration (Spring Boot 3).
  - Stateless JWT filter lifecycle, token validation, and refresh token rotation.
  - Web security defenses: CSRF (when to disable), CORS, SQLi, and XSS.
  - 📖 *Reading*: [07 Security, Testing & Observability](07_Security_Testing_Observability.md)

- [ ] **Day 23: OAuth2, OpenID Connect & Multi-Tenancy**
  - OAuth2 Grant Types: Authorization Code with PKCE, Client Credentials.
  - Configuring Spring Boot as an OAuth2 Resource Server (Keycloak / Okta).
  - Multi-Tenancy database routing with `AbstractRoutingDataSource`.
  - 📖 *Reading*: [13 Advanced Spring Security](13_Spring_Security_Advanced.md)

- [ ] **Day 24: Testing Strategies & Testcontainers**
  - Testing Pyramid: Unit vs. Slice (`@WebMvcTest`, `@DataJpaTest`) vs. Full (`@SpringBootTest`).
  - Spinning up real PostgreSQL, Kafka, and Redis containers with **Testcontainers**.
  - 📖 *Reading*: [07 Security, Testing & Observability](07_Security_Testing_Observability.md)

- [ ] **Day 25: Observability, Metrics & Distributed Tracing**
  - Structured JSON logging with Logback + MDC (`traceId`, `spanId`).
  - Micrometer metrics with Prometheus and Grafana dashboards.
  - OpenTelemetry (OTel) distributed tracing (W3C `traceparent` headers).
  - 📖 *Reading*: [07 Security, Testing & Observability](07_Security_Testing_Observability.md)

- [ ] **Day 26: Distributed Systems & Microservices Patterns**
  - Distributed Transactions: Saga Pattern (Orchestration vs. Choreography).
  - **Transactional Outbox Pattern** + Debezium CDC for reliable event publishing.
  - Distributed Locking with Redis & Redisson (`RLock.tryLock()`).
  - Domain-Driven Design (DDD): Aggregates, Entities, Value Objects, CQRS.
  - 📖 *Reading*: [08 Distributed Systems](08_Distributed_Systems_Cloud.md) & [15 Microservices](15_Microservices_Architecture.md)

- [ ] **Day 27: Cloud-Native Deployments (Docker & Kubernetes)**
  - Optimized multi-stage Dockerfiles with Spring Boot LayerTools (`layertools extract`).
  - Kubernetes manifests: Pods, Deployments, Services, ConfigMaps, Secrets, HPA.
  - K8s Liveness (`/actuator/health/liveness`) and Readiness probes.
  - 📖 *Reading*: [08 Distributed Systems & Cloud](08_Distributed_Systems_Cloud.md)

- [ ] **Day 28: System Design Case Studies & Production Troubleshooting**
  - HLD Case Studies: Distributed Rate Limiter & URL Shortener (TinyURL).
  - Live Incident Walkthroughs: HikariCP connection starvation & 100% CPU thread dump analysis.
  - Review Pre-Interview Cheat Sheet formulas and one-liners.
  - 📖 *Reading*: [09 System Design](09_System_Design_Senior_Scenarios.md) & [10 Quick Revision Checklist](10_Quick_Revision_Checklist.md)

---

## 🏆 Senior Engineer Competency Checklist

Use this checklist during your final mock review:

| Competency Area | What You Must Be Able to Explain Fluently | Done |
| :--- | :--- | :---: |
| **Language & JVM** | JVM memory areas, GC algorithm trade-offs (G1 vs ZGC), Loom Virtual Thread pinning. | [ ] |
| **Collections & JMM**| `HashMap` internal treeification, `ConcurrentHashMap` CAS bin locking, `volatile` memory barriers. | [ ] |
| **Spring Internals** | Complete bean lifecycle, AOP self-invocation bug, auto-configuration condition evaluation. | [ ] |
| **Data & JPA** | Hibernate N+1 query diagnosis & fixes (`JOIN FETCH`, `@EntityGraph`), `@Transactional` rollback rules. | [ ] |
| **Messaging & Events**| Kafka partition hashing, consumer rebalancing, idempotent producers, non-blocking retries. | [ ] |
| **Security** | Spring Security 6 Filter Chain, Stateless JWT validation, OAuth2 PKCE authorization flow. | [ ] |
| **Distributed Systems**| Saga Orchestration with compensating transactions, Transactional Outbox + CDC, Redisson locks. | [ ] |
| **Diagnostics** | Step-by-step CLI commands (`top -H`, `jstack`, Eclipse MAT) to debug 100% CPU and memory leaks. | [ ] |

---

> **Next Chapter**: [01 Modern Java, JVM Architecture & Language](01_Modern_Java_JVM_Language.md)
