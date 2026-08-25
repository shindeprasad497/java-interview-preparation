# Quick Revision Checklist & Pre-Interview Cheat Sheet

> **Navigation**: [Master Index](README.md) | [Previous: System Design](09_System_Design_Senior_Scenarios.md) | [Next: Strings Deep Dive](11_Strings_Deep_Dive.md)

---

## 1. Essential Formulas & Numbers Every Senior Engineer Should Know

```
+-----------------------------------------------------------------------------------+
|  1. HIKARICP CONNECTION POOL SIZING:                                              |
|     Connections = (CPU Cores * 2) + Effective Spindle Count (SSD = 1)             |
|     * Example: 4 Cores -> (4 * 2) + 1 = 9 to 10 connections                       |
|                                                                                   |
|  2. HASHMAP CAPACITY & THRESHOLD:                                                 |
|     Threshold = Initial Capacity * Load Factor                                    |
|     * Example: 16 * 0.75 = 12 elements before doubling to 32                      |
|     * Treeification threshold: 8 elements in single bin + total capacity >= 64   |
|                                                                                   |
|  3. BASE62 URL SHORTENER CAPACITY:                                                |
|     6 Characters: 62^6 = 56.8 Billion unique keys                                 |
|     7 Characters: 62^7 = 3.52 Trillion unique keys                                |
|                                                                                   |
|  4. THREAD POOL SIZING (LITTLE'S LAW):                                            |
|     Threads = Target Throughput (QPS) * Average Response Latency (seconds)       |
|     * Example: 1,000 QPS * 0.1s latency = 100 concurrent worker threads          |
+-----------------------------------------------------------------------------------+
```

---

## 2. Top 25 Spring Boot Annotations Quick Reference

| Annotation | Category | Core Purpose / What It Does Under the Hood |
| :--- | :--- | :--- |
| `@SpringBootApplication` | Core | Meta-annotation: `@Configuration` + `@EnableAutoConfiguration` + `@ComponentScan` |
| `@ConfigurationProperties`| Config | Binds external properties to a type-safe, validated Java record/POJO |
| `@ConditionalOnProperty` | AutoConfig| Registers bean only if specified property matches configured value |
| `@ConditionalOnMissingBean`| AutoConfig| Registers bean only if no other bean of same type exists in context |
| `@RestController` | Web | Combines `@Controller` and `@ResponseBody` (auto-serializes responses to JSON) |
| `@RestControllerAdvice` | Web | Global interceptor for exception handling across all controllers |
| `@ExceptionHandler` | Web | Catches specific exception types and converts them to HTTP responses |
| `@Valid` / `@Validated` | Validation| Triggers Jakarta Bean Validation on request body or method parameters |
| `@Transactional` | Data | Wraps method execution in an AOP database transaction proxy |
| `@Entity` / `@Table` | JPA | Maps Java class to a relational database table |
| `@OneToMany(mappedBy=...)`| JPA | Non-owning side of bidirectional relationship |
| `@EntityGraph` | JPA | Solves N+1 query problem by generating dynamic SQL `JOIN` |
| `@Version` | JPA | Enables Optimistic Locking via integer/timestamp column |
| `@Lock` | JPA | Enables Pessimistic Locking (`PESSIMISTIC_WRITE` $\rightarrow$ `SELECT FOR UPDATE`) |
| `@EnableCaching` | Cache | Activates Spring Cache proxy abstraction (`@Cacheable`, `@CacheEvict`) |
| `@Cacheable` | Cache | Checks cache before executing method; populates cache on miss |
| `@KafkaListener` | Messaging | Registers Kafka consumer method on background worker thread |
| `@RetryableTopic` | Messaging | Non-blocking topic-based Kafka retries with exponential backoff |
| `@EnableWebSecurity` | Security | Activates Spring Security filter chain processing |
| `@EnableMethodSecurity` | Security | Enables method-level authorization (`@PreAuthorize`, `@PostAuthorize`) |
| `@PreAuthorize` | Security | Evaluates SpEL expression before executing method (`hasRole('ADMIN')`) |
| `@SpringBootTest` | Testing | Boots up full Spring ApplicationContext for integration testing |
| `@WebMvcTest` | Testing | Sliced test: boots only Controller layer and MockMvc |
| `@DataJpaTest` | Testing | Sliced test: boots only Repository layer with configured DB |
| `@Async` | Concurrency| Executes method asynchronously in a background thread pool |

---

## 3. 10 Rapid-Fire Senior Interview Gotchas

1. **Self-Invocation Proxy Bypass**: Calling `@Transactional` or `@Async` method from inside the same class (`this.method()`) bypasses Spring AOP proxy.
2. **`@Transactional` Rollback Default**: Only rolls back on unchecked (`RuntimeException`) by default. Must specify `@Transactional(rollbackFor = Exception.class)`.
3. **Integer Cache Trap**: `Integer a = 100, b = 100; a == b` is `true`. `Integer c = 200, d = 200; c == d` is `false` (Cache range is `-128` to `127`).
4. **ThreadLocal Memory Leaks**: In thread pools, `ThreadLocal` variables must be cleaned via `ThreadLocal.remove()` in a `finally` block or filter.
5. **Virtual Thread Pinning**: Virtual threads pin the OS carrier thread if blocked inside a `synchronized` block or JNI native call.
6. **Parallel Streams in Web Apps**: `parallelStream()` uses shared common `ForkJoinPool`, risking server-wide thread starvation on blocking I/O.
7. **Eager Fetching Default**: `@ManyToOne` and `@OneToOne` default to `FetchType.EAGER` in JPA, causing hidden N+1 queries. Always switch to `FetchType.LAZY`.
8. **Unbounded ThreadPool Queue**: `Executors.newFixedThreadPool()` uses unbounded `LinkedBlockingQueue` (Integer.MAX_VALUE), risking `OutOfMemoryError`. Always use bounded queue with `CallerRunsPolicy`.
9. **HashMap `equals()` & `hashCode()`**: Mutating an object key after inserting it into a `HashMap` changes its hash code, making it unretrievable (memory leak).
10. **Stateless JWT vs CSRF**: Stateless REST APIs using `Authorization: Bearer <token>` headers do not need CSRF protection (disable CSRF). APIs using cookie-based auth must enable CSRF.

---

> **Next Chapter**: [11 Strings Deep Dive](11_Strings_Deep_Dive.md)
