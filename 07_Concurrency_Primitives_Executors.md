# 07. Concurrency Primitives, Thread Pools & CompletableFuture

> **Navigation**: [Master Index](README.md) | [Previous: Multithreading & JMM](06_Multithreading_JMM_Concurrency.md) | [Next: Virtual Threads (Loom)](08_Virtual_Threads_Loom.md)

---

## 📌 Chapter Overview
This module explores enterprise thread coordination primitives (`CountDownLatch`, `CyclicBarrier`, `Semaphore`, `Phaser`), deep internal tuning of `ThreadPoolExecutor`, rejection policies, and asynchronous composition with `CompletableFuture`.

---

## 1. Concurrency Coordination Primitives Comparison

| Primitive | Reusable? | Counter Direction | Core Purpose | Typical Production Use Case |
| :--- | :--- | :--- | :--- | :--- |
| **`CountDownLatch`** | **No** (One-shot) | Decrements ($N \rightarrow 0$) via `countDown()` | Main thread waits for $N$ independent worker tasks to complete | Microservice startup health check waiting for DB, Kafka, and Redis warm-up |
| **`CyclicBarrier`** | **Yes** (Resets) | Increments ($0 \rightarrow N$) via `await()` | $N$ parallel threads wait at a common rendezvous point before proceeding | Multi-threaded matrix calculation / game physics tick synchronization |
| **`Semaphore`** | **Yes** | Dynamic permits via `acquire()` / `release()` | Restricts maximum concurrent access to a shared resource (Rate limiting / throttling) | Limiting concurrent outbound HTTP calls to a rate-limited payment gateway |
| **`Phaser`** | **Yes** | Multi-phase dynamic registration | Complex multi-step parallel processing with varying thread counts | Multi-stage pipeline data processing (ETL batch transformations) |

---

### Q1. Code Deep Dive: `CountDownLatch` vs. `Semaphore` in Action.
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

## 2. `ThreadPoolExecutor` Internals & Sizing Architecture

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

### Q2. Deep Dive: `ThreadPoolExecutor` Constructor Parameters.
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

### Q3. How do you calculate optimal Thread Pool Size?
**Answer:**
- **For CPU-Bound Tasks (Encryption, JSON serialization, Image parsing)**:
  $$\text{Pool Size} = N_{\text{CPU}} + 1$$
  *(Where $N_{\text{CPU}} = \text{Runtime.getRuntime().availableProcessors()}$. The $+1$ accommodates occasional page faults).*
- **For I/O-Bound Tasks (Database queries, REST API calls, S3 file uploads)**:
  $$\text{Pool Size} = N_{\text{CPU}} \times \left(1 + \frac{\text{Wait Time}}{\text{Service Time}}\right)$$
  *Example: If a request spends 90ms waiting for DB (Wait Time) and 10ms processing CPU (Service Time) on an 8-core CPU:*
  $$\text{Pool Size} = 8 \times \left(1 + \frac{90}{10}\right) = 8 \times 10 = 80 \text{ threads}$$

---

## 3. Asynchronous Pipelines with `CompletableFuture`

### Q4. How do you compose asynchronous pipelines using `CompletableFuture`?
**Answer:**

```java
@Service
public class OrderAggregationService {
    
    private final ExecutorService customPool = Executors.newFixedThreadPool(16);

    public CompletableFuture<OrderSummary> aggregateOrderAsync(Long orderId) {
        // Step 1: Fetch Order details asynchronously
        CompletableFuture<Order> orderFuture = CompletableFuture.supplyAsync(
            () -> orderClient.getOrder(orderId), customPool
        );

        // Step 2: Fetch Payment details asynchronously using result from Order
        CompletableFuture<Payment> paymentFuture = orderFuture.thenCompose(
            order -> CompletableFuture.supplyAsync(() -> paymentClient.getPayment(order.paymentId()), customPool)
        );

        // Step 3: Fetch User Profile asynchronously in parallel
        CompletableFuture<User> userFuture = orderFuture.thenCompose(
            order -> CompletableFuture.supplyAsync(() -> userClient.getUser(order.userId()), customPool)
        );

        // Step 4: Combine all independent async results into final DTO with error fallback
        return CompletableFuture.allOf(orderFuture, paymentFuture, userFuture)
            .thenApply(v -> new OrderSummary(
                orderFuture.join(),
                paymentFuture.join(),
                userFuture.join()
            ))
            .exceptionally(ex -> {
                log.error("Order aggregation failed for ID: {}", orderId, ex);
                return OrderSummary.emptyFallback(orderId);
            });
    }
}
```

---

> **Next Chapter**: [08 Virtual Threads (Project Loom)](08_Virtual_Threads_Loom.md)

