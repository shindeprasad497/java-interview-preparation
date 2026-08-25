# 07. Concurrency Primitives, Thread Pools, Callable vs. Runnable & CompletableFuture

> **Navigation**: [Master Index](README.md) | [Previous: Multithreading & JMM](06_Multithreading_JMM_Concurrency.md) | [Next: Virtual Threads (Loom)](08_Virtual_Threads_Loom.md)

---

## 📌 Chapter Overview
This module explores enterprise thread coordination primitives (`CountDownLatch`, `CyclicBarrier`, `Semaphore`, `Phaser`), **`Runnable` vs. `Callable`**, **`Future` vs. `CompletableFuture`**, deep internal tuning of `ThreadPoolExecutor`, rejection policies, and asynchronous composition pipelines.

---

## 1. `Runnable` vs. `Callable` vs. `Future` vs. `CompletableFuture`

```
+-----------------------------------------------------------------------------------+
|                        CONCURRENCY ASYNC CONTRACT COMPARISON                      |
+-----------------------------------------------------------------------------------+
|  Contract / Type      | Return Value? | Throws Checked Exception? | Non-Blocking? |
|  -------------------- | :-----------: | :-----------------------: | :-----------: |
|  **`Runnable`**       | **No** (void) | No                        | N/A           |
|  **`Callable<V>`**    | **Yes** (`V`) | **Yes** (`throws Exception`)| N/A         |
|  **`Future<V>`**      | **Yes** (`V`) | **Yes** (`ExecutionEx`)   | **No** (Blocks)|
|  **`CompletableFuture`| **Yes** (`T`) | Handled via callbacks     | **Yes (100%)** |
+-----------------------------------------------------------------------------------+
```

### Q1. Compare `Runnable` vs. `Callable` and `Future` vs. `CompletableFuture`.
**Answer:**

#### 1. `Runnable` vs. `Callable<V>`:
- **`Runnable` (`java.lang`)**: Single abstract method `public void run()`. Cannot return a computation result and cannot throw checked exceptions.
- **`Callable<V>` (`java.util.concurrent`)**: Single abstract method `public V call() throws Exception`. Returns a generic result `V` and can throw checked exceptions directly to the executor.

#### 2. Why `Future<V>` is Insufficient (The 5 Limitations):
1. **Blocking `.get()`**: Retrieving the result forces the calling thread to block until the task completes.
2. **Cannot Chain Asynchronous Stages**: You cannot trigger a callback when the Future finishes without polling or blocking.
3. **Cannot Combine Multiple Futures**: Combining results from 5 parallel tasks requires blocking `.get()` sequentially on each.
4. **Cannot Manually Complete**: Cannot complete the future externally from a webhook or network response.
5. **No Built-in Exception Handling**: No fallback recovery mechanism (`exceptionally` or `handle`).

---

## 2. Asynchronous Pipelines with `CompletableFuture`

```
+-----------------------------------------------------------------------------------+
|                        COMPLETABLEFUTURE METHOD TAXONOMY                          |
+-----------------------------------------------------------------------------------+
| 1. Creation:     supplyAsync(Supplier<U>), runAsync(Runnable)                     |
| 2. Chaining:     thenApply(T -> R), thenAccept(T -> void), thenRun(() -> void)    |
| 3. Composition:  thenCompose(T -> CompletableFuture<U>) (Monadic FlatMap)        |
| 4. Combination:  thenCombine(CF<U>, BiFunction<T, U, V>) (Parallel Join)          |
| 5. Multi-Join:   CompletableFuture.allOf(cf1, cf2...), anyOf(cf1, cf2...)         |
| 6. Recovery:     exceptionally(ex -> fallback), handle((res, ex) -> ...)          |
+-----------------------------------------------------------------------------------+
```

### Q2. How do you compose asynchronous pipelines using `CompletableFuture`?
**Answer:**

```java
@Service
public class OrderAggregationService {
    
    private final ExecutorService customPool = Executors.newFixedThreadPool(16);

    public CompletableFuture<OrderSummary> aggregateOrderAsync(Long orderId) {
        // Step 1: SupplyAsync - Fetch Order asynchronously
        CompletableFuture<Order> orderFuture = CompletableFuture.supplyAsync(
            () -> orderClient.getOrder(orderId), customPool
        );

        // Step 2: thenCompose (FlatMap) - Fetch Payment using Order's paymentId (Dependent Async Task)
        CompletableFuture<Payment> paymentFuture = orderFuture.thenCompose(
            order -> CompletableFuture.supplyAsync(() -> paymentClient.getPayment(order.paymentId()), customPool)
        );

        // Step 3: Fetch User Profile in Parallel
        CompletableFuture<User> userFuture = orderFuture.thenCompose(
            order -> CompletableFuture.supplyAsync(() -> userClient.getUser(order.userId()), customPool)
        );

        // Step 4: allOf - Wait for all futures to complete in parallel without blocking!
        return CompletableFuture.allOf(orderFuture, paymentFuture, userFuture)
            .thenApply(v -> new OrderSummary(
                orderFuture.join(),   // Non-blocking here because allOf() already guaranteed completion!
                paymentFuture.join(),
                userFuture.join()
            ))
            // Step 5: exceptionally - Fallback recovery on downstream failure
            .exceptionally(ex -> {
                log.error("Order aggregation failed for ID: {}", orderId, ex);
                return OrderSummary.emptyFallback(orderId);
            });
    }
}
```

---

## 3. Concurrency Coordination Primitives Comparison

| Primitive | Reusable? | Counter Direction | Core Purpose | Typical Production Use Case |
| :--- | :--- | :--- | :--- | :--- |
| **`CountDownLatch`** | **No** (One-shot) | Decrements ($N \rightarrow 0$) via `countDown()` | Main thread waits for $N$ independent worker tasks to complete | Microservice startup health check waiting for DB, Kafka, and Redis warm-up |
| **`CyclicBarrier`** | **Yes** (Resets) | Increments ($0 \rightarrow N$) via `await()` | $N$ parallel threads wait at a common rendezvous point before proceeding | Multi-threaded matrix calculation / game physics tick synchronization |
| **`Semaphore`** | **Yes** | Dynamic permits via `acquire()` / `release()` | Restricts maximum concurrent access to a shared resource (Rate limiting / throttling) | Limiting concurrent outbound HTTP calls to a rate-limited payment gateway |
| **`Phaser`** | **Yes** | Multi-phase dynamic registration | Complex multi-step parallel processing with varying thread counts | Multi-stage pipeline data processing (ETL batch transformations) |

---

### Q3. Code Deep Dive: `CountDownLatch` vs. `Semaphore` in Action.
**Answer:**

```java
public class ConcurrencyCoordinationExamples {

    // 1. CountDownLatch: Wait for 3 downstream services to initialize
    public void waitForStartupServices() throws InterruptedException {
        CountDownLatch latch = new CountDownLatch(3);
        
        ExecutorService executor = Executors.newFixedThreadPool(3);
        executor.submit(() -> { initDatabase(); latch.countDown(); });
        executor.submit(() -> { initKafka(); latch.countDown(); });
        executor.submit(() -> { initRedis(); latch.countDown(); });

        // Block main thread until count drops to 0 (with timeout)
        boolean ready = latch.await(10, TimeUnit.SECONDS);
        if (!ready) {
            throw new IllegalStateException("Service startup timed out!");
        }
        System.out.println("All services initialized successfully!");
    }

    // 2. Semaphore: Rate-limiting concurrent outbound calls
    private final Semaphore rateLimiter = new Semaphore(5); // Max 5 concurrent calls

    public String callDownstreamWithThrottling(String url) throws InterruptedException {
        rateLimiter.acquire(); // Blocks if 5 active requests are already in-flight
        try {
            return executeHttp(url);
        } finally {
            rateLimiter.release(); // Releases permit back to pool
        }
    }
}
```

---

## 4. `ThreadPoolExecutor` Internals & Sizing Architecture

```
                                   [ Incoming Task ]
                                           |
                                           v
                          Is Active Threads < corePoolSize ?
                                  /                 \
                          (YES)  /                   \ (NO)
                                v                     v
                   [ Create New Worker ]      Is workQueue FULL ?
                                                      /        \
                                              (NO)   /          \ (YES)
                                                    v            v
                                            [ Enqueue Task ]  Is Active Threads < maxPoolSize ?
                                                                      /                 \
                                                              (YES)  /                   \ (NO)
                                                                    v                     v
                                                           [ Create New Worker ]  [ Trigger REJECTION POLICY ]
```

### Q4. Deep Dive: `ThreadPoolExecutor` Constructor Parameters & Rejection Policies.
**Answer:**

```java
public ThreadPoolExecutor(
    int corePoolSize,                      // Minimum active worker threads
    int maximumPoolSize,                   // Maximum allowed worker threads
    long keepAliveTime, TimeUnit unit,     // Time excess idle threads (beyond core) wait before terminating
    BlockingQueue<Runnable> workQueue,     // Task queue holding tasks before execution
    ThreadFactory threadFactory,           // Factory for naming and daemon flags
    RejectedExecutionHandler handler       // Policy triggered when queue & maxPoolSize are saturated
);
```

#### Production Rejection Policies:
1. **`AbortPolicy` (Default)**: Throws `RejectedExecutionException`.
2. **`CallerRunsPolicy` (Recommended for Backpressure)**: The submitting thread (e.g., Tomcat HTTP thread) executes the task itself. This naturally slows down incoming request throughput, preventing system crash!
3. **`DiscardPolicy`**: Silently drops the task without throwing an exception.
4. **`DiscardOldestPolicy`**: Discards the oldest unhandled task in the queue to make room for the new task.

---

### Q5. How do you calculate optimal Thread Pool Size?
**Answer:**
- **For CPU-Bound Tasks (Encryption, JSON serialization, Image parsing)**:
  $$\text{Pool Size} = N_{\text{CPU}} + 1$$
- **For I/O-Bound Tasks (Database queries, REST API calls, S3 file uploads)**:
  $$\text{Pool Size} = N_{\text{CPU}} \times \left(1 + \frac{\text{Wait Time}}{\text{Service Time}}\right)$$
  *Example: If a request spends 90ms waiting for DB (Wait Time) and 10ms processing CPU (Service Time) on an 8-core CPU:*
  $$\text{Pool Size} = 8 \times \left(1 + \frac{90}{10}\right) = 8 \times 10 = 80 \text{ threads}$$

---

> **Next Chapter**: [08 Virtual Threads (Project Loom)](08_Virtual_Threads_Loom.md)
