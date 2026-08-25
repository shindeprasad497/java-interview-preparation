# 37. Quick Revision Checklist, Master Annotations Guide & Cheat Sheet

> **Navigation**: [Master Index](README.md) | [Previous: Spring AI Guide](36_Spring_AI_Generative_AI_Guide.md) | [Study Roadmap](00_Study_Roadmap.md)

---

## 📌 Chapter Overview
The **Ultimate Master Reference & Pre-Interview Cheat Sheet**. Contains the complete **Master Encyclopedia of Spring & Java Annotations** (When to use, how to use, code snippets, special cases), core engineering formulas, and **15 rapid-fire senior gotchas**.

---

## 1. Master Encyclopedia of Spring & Java Annotations

```
+-----------------------------------------------------------------------------------+
|                        SPRING ANNOTATIONS TAXONOMY                                |
+-----------------------------------------------------------------------------------+
| 1. Core DI & IoC:     @Component, @Bean, @Primary, @Qualifier, @Lazy, @Order...   |
| 2. Web & REST:        @RestController, @PathVariable, @RequestParam, @RequestBody |
| 3. Validation:        @Valid, @Validated, @NotNull, @NotBlank, @Size...           |
| 4. Caching:           @Cacheable, @CachePut, @CacheEvict, @Caching, @CacheConfig  |
| 5. Transactions & JPA:@Transactional, @Query, @EntityGraph, @Lock, @Version       |
| 6. Async & Schedules: @Async, @Scheduled, @EnableAsync, @EnableScheduling         |
| 7. Security:          @PreAuthorize, @PostAuthorize, @AuthenticationPrincipal     |
| 8. Testing:           @SpringBootTest, @WebMvcTest, @DataJpaTest, @MockBean       |
+-----------------------------------------------------------------------------------+
```

---

### 🔷 Group 1: Core Spring IoC, Bean Wiring & Lifecycle

#### 1. `@Primary` vs. `@Qualifier`
- **`@Primary`**: Indicates that when multiple beans of the same type exist, this bean should be given **default priority**:
  ```java
  @Component
  @Primary
  public class DefaultPaymentGateway implements PaymentGateway {}
  ```
- **`@Qualifier("beanName")`**: Overrides `@Primary` by explicitly specifying the exact bean name to inject at the injection point:
  ```java
  @Service
  public class OrderService {
      // Injects PayPalGateway even if DefaultPaymentGateway is marked @Primary!
      public OrderService(@Qualifier("payPalPaymentGateway") PaymentGateway gateway) {}
  }
  ```

#### 2. `@Order` vs. `@Priority`
- **When to use**: Controls the execution order of **AOP Aspects**, **Filter Chains**, **Spring Event Listeners**, or beans injected into a `List<T>`.
- **Rule**: Lower values have higher priority (e.g., `@Order(1)` runs before `@Order(100)`).
  ```java
  @Component
  @Order(1) // Runs first in the filter chain
  public class SecurityHeaderFilter implements Filter {}
  ```

#### 3. `@Lazy`
- **When to use**: Delays bean initialization until it is first requested (rather than at application startup), or resolves **AOP Self-Invocation / Circular Dependencies**:
  ```java
  @Service
  public class ReportService {
      @Autowired @Lazy private ReportService self; // Solves AOP self-invocation!
  }
  ```

#### 4. `@DependsOn`
- **When to use**: Forces Spring to initialize a specific bean *before* the annotated bean, even if there is no direct dependency between them (e.g., initializing Database Seeders before Cache Warmers):
  ```java
  @Component
  @DependsOn("databaseMigrationRunner")
  public class CacheWarmer {}
  ```

#### 5. `@Scope`
- **Options**: `singleton` (default), `prototype` (new instance per injection), `request` (per HTTP request), `session` (per HTTP session), `application` (per ServletContext).
  ```java
  @Bean
  @Scope(ConfigurableBeanFactory.SCOPE_PROTOTYPE)
  public CryptoTokenGenerator tokenGenerator() { return new CryptoTokenGenerator(); }
  ```

---

### 🔷 Group 2: Spring Cache Abstraction

| Annotation | Mechanism | Real-World Code Example |
| :--- | :--- | :--- |
| **`@EnableCaching`** | Enables Spring AOP cache proxy post-processor | `@Configuration @EnableCaching public class CacheConfig {}` |
| **`@Cacheable`** | Checks cache; skips method execution on cache HIT | `@Cacheable(value = "users", key = "#id", unless = "#result == null")` |
| **`@CachePut`** | Always executes method AND updates cache entry with result | `@CachePut(value = "users", key = "#result.id") public UserDto update(...)` |
| **`@CacheEvict`** | Deletes matching key or flushes entire cache | `@CacheEvict(value = "users", key = "#id", allEntries = false)` |
| **`@Caching`** | Combines multiple cache operations | `@Caching(evict = { @CacheEvict("users", key="#id"), @CacheEvict("summaries", allEntries=true) })`|
| **`@CacheConfig`** | Class-level default cache configurations | `@Service @CacheConfig(cacheNames = "orders") public class OrderService {}` |

---

### 🔷 Group 3: Spring MVC & REST Web

```java
@RestController
@RequestMapping("/api/v1/users")
@CrossOrigin(origins = "https://app.example.com")
public class UserController {

    // 1. @PathVariable: Binds URL template variable (/users/{id})
    // 2. @RequestHeader: Binds incoming HTTP header
    @GetMapping("/{id}")
    public ResponseEntity<UserDto> getUser(
            @PathVariable("id") Long id,
            @RequestHeader("X-Tenant-Id") String tenantId) {
        return ResponseEntity.ok(userService.findById(id, tenantId));
    }

    // 3. @RequestParam: Binds query parameters (?page=0&size=10)
    @GetMapping
    public List<UserDto> listUsers(
            @RequestParam(name = "page", defaultValue = "0") int page,
            @RequestParam(name = "size", defaultValue = "20") int size) {
        return userService.findAll(page, size);
    }

    // 4. @RequestBody: Jackson deserializes HTTP JSON body into DTO + @Valid validates JSR-380 rules
    // 5. @ResponseStatus: Sets default HTTP response code on method return
    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public UserDto createUser(@Valid @RequestBody CreateUserRequest req) {
        return userService.create(req);
    }
}
```

---

### 🔷 Group 4: Spring Data JPA & Persistence

| Annotation | Purpose | Example |
| :--- | :--- | :--- |
| **`@Transactional`** | Declarative boundary with propagation and isolation | `@Transactional(propagation = Propagation.REQUIRES_NEW, rollbackFor = Exception.class)` |
| **`@Query` + `@Modifying`** | Custom DML updates/deletes in Spring Data JPA | `@Modifying @Query("UPDATE User u SET u.active = false WHERE u.id = :id") void deactivate(@Param("id") Long id);` |
| **`@EntityGraph`** | Overrides FetchType to perform eager SQL `JOIN FETCH` | `@EntityGraph(attributePaths = {"orders", "addresses"}) Optional<User> findById(Long id);` |
| **`@Lock`** | Pessimistic locking (`SELECT ... FOR UPDATE`) | `@Lock(LockModeType.PESSIMISTIC_WRITE) Optional<Account> findByIdForUpdate(Long id);` |
| **`@Version`** | Optimistic concurrency locking via version column | `@Version private Long version;` |
| **`@Transient`** | Ignores field during ORM database persistence | `@Transient private String temporaryToken;` |

---

### 🔷 Group 5: Async, Scheduling & Security

#### 1. `@Async` & `@Scheduled`:
```java
@Configuration
@EnableAsync
@EnableScheduling
public class AsyncConfig {

    // Runs on background ThreadPoolExecutor thread
    @Async("customTaskExecutor")
    public CompletableFuture<String> generatePdfReport(Long reportId) {
        return CompletableFuture.completedFuture("PDF_GENERATED");
    }

    // Cron job: Runs at 02:00 AM every night
    @Scheduled(cron = "0 0 2 * * *")
    public void cleanupOldTempFiles() {
        // Cleanup logic
    }
}
```

#### 2. Spring Security Method Annotations:
```java
@Service
public class DocumentService {

    // Evaluates SpEL before method execution; allows only ADMIN or document owner
    @PreAuthorize("hasRole('ADMIN') or #ownerId == authentication.principal.username")
    public Document getDocument(Long docId, String ownerId) {
        return documentRepo.findById(docId).orElseThrow();
    }

    // Injects authenticated user principal directly into method parameter
    public void updateProfile(@AuthenticationPrincipal CustomUserDetails userDetails) {
        String currentUserId = userDetails.getId();
    }
}
```

---

## 2. Essential Formulas & Numbers Every Senior Engineer Must Know

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

## 3. 15 Rapid-Fire Senior Interview Gotchas

1. **Integer Cache Trap**: `Integer a = 100, b = 100; a == b` is `true`. `Integer x = 200, y = 200; x == y` is `false` (cached only from `-128` to `127`).
2. **Spring AOP Self-Invocation**: Calling `this.methodB()` inside the same class **bypasses the AOP proxy**, so `@Transactional`, `@Cacheable`, and `@Async` silently fail.
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
