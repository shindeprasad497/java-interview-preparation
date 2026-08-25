# Collections, Generics & Concurrency Deep Dive

> **Navigation**: [Master Index](README.md) | [Previous: Modern Java & JVM](01_Modern_Java_JVM_Language.md) | [Next: Coding & Problem Solving](03_Coding_Problem_Solving.md)

---

## 1. Java Collections Framework Internals

### Q1. Deep Dive: How does `HashMap` work internally in Java 8+?
**Answer:**

```
+-----------------------------------------------------------------------------------+
|                            HASHMAP BUCKET ARRAY (Node<K,V>[])                     |
+-----------------------------------------------------------------------------------+
| Index 0 | Index 1 | Index 2 | Index 3 | ... | Index 15 (Initial default capacity) |
+----+----+----+----+----+----+----+----+-------------------------------------------+
     |         |
     v         v
  [Node A]   [Node B] -> [Node C] -> [Node D]  (Linked list if bin count < 8)
     |
     v
  [TreeNode]  <--- Converted to Red-Black Tree (Self-balancing) when bin count >= 8
  /        \       and total array capacity >= 64
[TN 1]    [TN 2]
```

1. **Internal Storage**: `transient Node<K,V>[] table;` (Array of bucket nodes, default initial capacity = 16, load factor = 0.75).
2. **Index Calculation (Bitwise AND Optimization)**:
   ```java
   static final int hash(Object key) {
       int h;
       // High-order bit mixing (prevents collisions in lower bits)
       return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
   }
   int index = (n - 1) & hash; // Equivalent to hash % n (only when n is power of 2)
   ```
3. **Collision Handling & Treeification**:
   - When multiple keys hash to the same bucket index, entries form a singly linked list.
   - **Treeification Threshold (8)**: When a single bucket exceeds **8 nodes** and total map capacity is $\ge 64$, the linked list converts into a **Red-Black Self-Balancing Binary Search Tree (`TreeNode`)**.
   - Search complexity improves from O(N) to **O(log N)**.
   - **Untreeify Threshold (6)**: During resizing, if bucket tree nodes drop to $\le 6$, it converts back to a singly linked list.
4. **Resizing (Rehashing)**:
   - When total entries exceed `threshold = capacity * loadFactor` (16 * 0.75 = 12), the array size doubles (32, 64, 128 ...).
   - Because capacity is always a power of 2, nodes either remain at index `i` or move to `i + oldCapacity` based on whether the newly included higher bit is `0` or `1`.

---

### Q2. How does `ConcurrentHashMap` achieve high throughput in Java 8+?
**Answer:**
- **Java 7 (Legacy)**: Used a fixed array of `Segment<K,V>` locks (typically 16 segments). Concurrency was limited to the number of segments.
- **Java 8+ (Modern)**: Eliminates Segment locks completely. Uses fine-grained **CAS (Compare-And-Swap) + `synchronized` on individual bucket heads**:
  1. **Empty Bucket Insertion**: Uses non-blocking CPU CAS (`Unsafe.compareAndSetObject`). No lock acquired!
  2. **Non-Empty Bucket Collision**: Locks **only the first node (bin head)** of that specific bucket using Java's lightweight intrinsic `synchronized` monitor. All other buckets remain fully accessible to concurrent threads.
  3. **Concurrent Reads**: Completely **lock-free**. The node's `val` and `next` pointers are marked `volatile`, ensuring instant cross-thread memory visibility.
  4. **Concurrent Resizing (`transfer`)**: Multiple threads help resize and migrate sub-ranges of buckets simultaneously using a shared `sizeCtl` counter.

---

### Q3. How do you implement a thread-safe LRU Cache using `LinkedHashMap`?
**Answer:**
`LinkedHashMap` maintains a doubly linked list running across all entries. By setting `accessOrder = true`, the list reorders entries from least-recently-accessed to most-recently-accessed on every `.get()` and `.put()`.

```java
public class LRUCache<K, V> extends LinkedHashMap<K, V> {
    private final int maxCapacity;

    public LRUCache(int maxCapacity) {
        // initialCapacity, loadFactor, accessOrder (true = access order, false = insertion order)
        super(maxCapacity, 0.75f, true);
        this.maxCapacity = maxCapacity;
    }

    @Override
    protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
        // Automatically evicts the least recently used element when size exceeds capacity
        return size() > maxCapacity;
    }

    // Thread-safe wrapper
    public static <K, V> Map<K, V> synchronizedLRUCache(int maxCapacity) {
        return Collections.synchronizedMap(new LRUCache<>(maxCapacity));
    }
}
```

---

## 2. Generics & PECS Rule

### Q4. Explain the PECS Principle: Producer Extends, Consumer Super.
**Answer:**
The PECS rule dictates wildcard usage in Java Generics for type safety:

- **Producer Extends (`<? extends T>`)**: If a parameterized type represents a *producer* that you only **READ** data from, use `? extends T` (Covariance).
- **Consumer Super (`<? super T>`)**: If a parameterized type represents a *consumer* that you only **WRITE** data into, use `? super T` (Contravariance).

```java
public class CollectionsUtil {
    // Reads from 'src' (Producer) and writes to 'dest' (Consumer)
    public static <T> void copy(List<? super T> dest, List<? extends T> src) {
        for (T item : src) { // SAFE to read T from src
            dest.add(item);  // SAFE to add T into dest
        }
    }
}
```

---

## 3. Multithreading, JMM & Synchronization

### Q5. Explain the Java Memory Model (JMM), `volatile`, and Happens-Before.
**Answer:**

```
+---------------+     +---------------+
|   Thread 1    |     |   Thread 2    |
| [ CPU Core 1] |     | [ CPU Core 2] |
| [ L1/L2 Cache]|     | [ L1/L2 Cache]|
+-------+-------+     +-------+-------+
        |                     |
        +---------\ /---------+
                   v
         +-------------------+
         |    MAIN MEMORY    |  <--- Shared RAM
         +-------------------+
```

1. **The Problem**: Modern multi-core CPUs cache variables in local L1/L2 hardware caches. Without synchronization, writes made by Core 1 may remain in its local cache without being flushed to RAM, making changes invisible to Core 2.
2. **`volatile` Keyword**:
   - **Visibility Guarantee**: Every read of a `volatile` variable reads directly from Main Memory. Every write flushes immediately to Main Memory.
   - **Instruction Reordering Prevention**: Generates hardware **Memory Barriers (Fences)** preventing the compiler and CPU from reordering instructions across the barrier.
   - *Limitation*: `volatile` does NOT guarantee atomicity for compound actions (e.g., `count++` requires a lock or `AtomicInteger`).
3. **Happens-Before Rules**:
   - **Monitor Lock Rule**: An unlock on a monitor lock happens-before every subsequent lock on the same monitor.
   - **Volatile Variable Rule**: A write to a `volatile` field happens-before every subsequent read of that same field.
   - **Thread Start / Join Rule**: Calling `thread.start()` happens-before any action in the started thread.

---

### Q6. Compare `synchronized` vs. `ReentrantLock` vs. `StampedLock`.
**Answer:**

| Feature | `synchronized` | `ReentrantLock` | `StampedLock` |
| :--- | :--- | :--- | :--- |
| **Type** | Built-in language keyword | API Class (`j.u.c.locks`) | API Class (`j.u.c.locks`) |
| **Fairness Support** | No (barging / non-fair) | Yes (`new ReentrantLock(true)`) | No |
| **Interruptible / Timeout**| No (blocks indefinitely) | Yes (`lockInterruptibly()`, `tryLock(time)`) | Yes (`tryOptimisticRead()`, `tryWriteLock()`) |
| **Optimistic Reading** | No | No | **Yes** (Validates stamp without acquiring real read lock) |
| **Virtual Thread Safety** | Pins carrier thread (pre-Java 24) | Safe (unmounts virtual thread) | Safe |

```java
// StampedLock Optimistic Read Example (High-Throughput Coordinate Reader)
public class Point {
    private double x, y;
    private final StampedLock sl = new StampedLock();

    public double distanceFromOrigin() {
        long stamp = sl.tryOptimisticRead(); // Non-blocking!
        double currentX = x, currentY = y;
        if (!sl.validate(stamp)) { // Check if a write occurred while reading
            stamp = sl.readLock(); // Fallback to full pessimistic read lock
            try {
                currentX = x;
                currentY = y;
            } finally {
                sl.unlockRead(stamp);
            }
        }
        return Math.sqrt(currentX * currentX + currentY * currentY);
    }
}
```

---

## 4. Executors Framework, Thread Pools & CompletableFuture

### Q7. Deep Dive: `ThreadPoolExecutor` parameters and Rejection Policies.
**Answer:**

```java
ThreadPoolExecutor executor = new ThreadPoolExecutor(
    corePoolSize,          // Minimum active threads kept alive
    maximumPoolSize,       // Max burst threads allowed
    keepAliveTime, unit,   // Idle timeout for excess threads (> core)
    new ArrayBlockingQueue<>(1000), // BOUNDED queue (Never use unbounded in prod!)
    new ThreadFactoryBuilder().setNameFormat("order-worker-%d").build(),
    new ThreadPoolExecutor.CallerRunsPolicy() // Rejection Handler
);
```

#### Task Submission Lifecycle:
1. If running threads < corePoolSize, create a **new core thread** to handle the task immediately.
2. If running threads $\ge$ corePoolSize, attempt to queue task in `workQueue`.
3. If `workQueue` is **FULL** and running threads < maximumPoolSize, spawn a **burst thread**.
4. If `workQueue` is **FULL** and running threads == maximumPoolSize, trigger the **`RejectedExecutionHandler`**.

#### Rejection Policies:
- **`AbortPolicy` (Default)**: Throws `RejectedExecutionException`.
- **`CallerRunsPolicy` (Best for graceful backpressure)**: Executes the task on the calling thread (e.g., the Tomcat HTTP worker thread), naturally throttling incoming requests while the pool drains.
- **`DiscardPolicy`**: Silently drops the task with no error.
- **`DiscardOldestPolicy`**: Discards the oldest unhandled task in the queue and retries submission.

---

### Q8. How do you compose asynchronous pipelines using `CompletableFuture`?
**Answer:**

```java
public CompletableFuture<OrderSummary> processOrderAsync(String orderId) {
    return CompletableFuture.supplyAsync(() -> orderService.fetchOrder(orderId), customExecutor)
        .thenCompose(order -> {
            // Asynchronous dependent call (flatMap)
            CompletableFuture<PaymentStatus> paymentFuture = CompletableFuture.supplyAsync(
                () -> paymentService.charge(order), customExecutor);
            CompletableFuture<InventoryStatus> inventoryFuture = CompletableFuture.supplyAsync(
                () -> inventoryService.reserve(order), customExecutor);

            // Combine both parallel futures
            return paymentFuture.thenCombine(inventoryFuture, (payment, inventory) -> 
                new OrderSummary(order, payment, inventory)
            );
        })
        .orTimeout(3, TimeUnit.SECONDS) // Java 9+ timeout
        .exceptionally(ex -> {
            logger.error("Order processing failed for id: {}", orderId, ex);
            return OrderSummary.failed(orderId, ex.getMessage());
        });
}
```

---

### Q9. What are the memory leak dangers of `ThreadLocal` in thread pools?
**Answer:**
`ThreadLocal` stores data inside a thread-private map (`Thread.threadLocals`).
- In server environments (Tomcat, Spring Boot), threads from the thread pool are **never destroyed**; they are reused across millions of HTTP requests.
- If a `ThreadLocal` is set during a request (e.g., holding `UserSecurityContext` or `TenantContext`) and **not cleaned up**, the data persists in that thread forever.
- **Consequences**:
  1. **Memory Leaks**: Heavy objects remain reachable via the thread's stack/map, preventing Garbage Collection.
  2. **Security Breaches**: Request B handled by the same worker thread might inherit Request A's authentication context!
- **Mandatory Prevention**: Always call `ThreadLocal.remove()` in a `finally` block or Spring `HandlerInterceptor.afterCompletion()`.

---

> **Next Chapter**: [03 Coding & Problem Solving](03_Coding_Problem_Solving.md)
