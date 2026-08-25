# 06. Multithreading, JMM & Synchronization Deep Dive

> **Navigation**: [Master Index](README.md) | [Previous: Java I/O & NIO](05_Java_IO_NIO_Netty.md) | [Next: Concurrency Primitives & Executors](07_Concurrency_Primitives_Executors.md)

---

## 📌 Chapter Overview
This module explores advanced multithreading, the **Java Memory Model (JMM)**, CPU cache coherence, hardware memory barriers, lock escalation mechanics, lock-free CAS algorithms, and `ThreadLocal` leak diagnosis.

---

## 1. The Java Memory Model (JMM) & Hardware Architecture

```
+-------------------------------------------------------------------------------+
|                             CPU CACHE & MAIN MEMORY                           |
+-------------------------------------------------------------------------------+
|  +--------------------+                    +--------------------+             |
|  |     CPU Core 1     |                    |     CPU Core 2     |             |
|  |  [ L1 / L2 Cache ] |                    |  [ L1 / L2 Cache ] |             |
|  |  Thread A (Local)  |                    |  Thread B (Local)  |             |
|  +--------------------+                    +--------------------+             |
|             \                                        /                        |
|              \                                      /                         |
|      +------------------------------------------------------+                 |
|      |                   Shared L3 Cache                    |                 |
|      +------------------------------------------------------+                 |
|                                 |                                             |
|                                 v                                             |
|      +------------------------------------------------------+                 |
|      |           MAIN SYSTEM MEMORY (RAM / Heap)            |                 |
|      |            [ Shared Volatile Variables ]             |                 |
|      +------------------------------------------------------+                 |
+-------------------------------------------------------------------------------+
```

### Q1. What is the Java Memory Model (JMM) and Happens-Before Guarantee?
**Answer:**
The **JMM** specifies how the JVM and hardware CPU caches interact with main memory to ensure consistent multi-threaded memory access across diverse CPU architectures.

#### The 3 Concurrency Guarantees:
1. **Atomicity**: Operations execute as a single, indivisible unit (achieved via `synchronized`, Locks, or CAS atomics).
2. **Visibility**: When one thread modifies a shared variable, other threads immediately observe the updated value (achieved via `volatile` or memory barriers).
3. **Ordering**: The JVM compiler and CPU execute instructions in program order without illegal reordering (achieved via *Happens-Before* rules).

#### Key Happens-Before Rules:
- **Program Order Rule**: Within a single thread, each action happens-before any subsequent action in program order.
- **Monitor Lock Rule**: An unlock on a monitor lock happens-before every subsequent lock on that same monitor.
- **Volatile Variable Rule**: A write to a `volatile` field happens-before every subsequent read of that same `volatile` field.
- **Thread Start/Join Rule**: `thread.start()` happens-before any action in the started thread; all actions in a thread happen-before `thread.join()` returns.

---

### Q2. Why is `volatile` not sufficient for compound operations (`count++`)?
**Answer:**
`volatile` guarantees **Visibility** and **Ordering** (prevents instruction reordering via CPU memory barriers `StoreStore` and `LoadStore`), but does **NOT guarantee Atomicity**.

A compound operation like `count++` consists of 3 distinct non-atomic bytecode instructions:
1. `GETFIELD` (Read current value into CPU register).
2. `IADD` (Increment register value by 1).
3. `PUTFIELD` (Write back register value to memory).

If Thread A and Thread B execute concurrently, both can read the same value before writing back, resulting in lost updates.

```java
// Thread-Safe Alternatives for count++:
AtomicInteger atomicCount = new AtomicInteger(0);
atomicCount.incrementAndGet(); // Lock-free hardware CAS

LongAdder highThroughputAdder = new LongAdder();
highThroughputAdder.increment(); // Striped cell array avoiding false sharing
```

---

## 2. Lock Optimization & Lock Escalation Mechanics

### Q3. How does HotSpot JVM Lock Escalation work (`synchronized`)?
**Answer:**
In modern HotSpot JVMs, `synchronized` locks are optimized to prevent expensive OS kernel context switches:

```
 [ Biased Lock ] -------------> [ Lightweight Lock (CAS) ] -------------> [ Heavyweight Lock ]
 (Thread ID biased in Mark Word) (Spin-locking on user-space stack)       (OS Mutex / Kernel Sleep)
```

1. **Biased Locking** *(Disabled by default in Java 15+)*: Biases the object's Mark Word header to the first acquiring thread ID with zero atomic instructions.
2. **Lightweight Locking (Thin Lock / CAS)**: When multiple threads access the lock without heavy contention, threads use **CAS spin-locking** (`Thread.onSpinWait()`) in user space, avoiding OS kernel context switches.
3. **Heavyweight Locking (Fat Lock / OS Monitor)**: Under heavy contention, the lock inflates into an OS kernel **Mutex**. Blocked threads are put to sleep (`BLOCKED` state in thread dump) until awakened by the OS scheduler.

*Lock optimization also includes **Lock Elision** (removing locks on non-escaping objects detected via Escape Analysis) and **Lock Coarsening** (merging adjacent locks on the same object into one).*

---

## 3. Explicit Lock APIs: `ReentrantLock` vs. `StampedLock`

### Q4. Compare `synchronized` vs. `ReentrantLock` vs. `StampedLock`.
**Answer:**

| Feature | `synchronized` | `ReentrantLock` | `StampedLock` (Java 8+) |
| :--- | :--- | :--- | :--- |
| **Reentrancy** | Yes | Yes | **No** (Non-reentrant!) |
| **Fairness Policy** | No (Non-fair only) | Yes (`new ReentrantLock(true)`) | No |
| **Timed / Interruptible**| No (Blocks indefinitely) | Yes (`tryLock()`, `lockInterruptibly()`) | Yes (`tryOptimisticRead()`) |
| **Condition Variables** | 1 (`wait()`, `notify()`) | Multiple (`Condition` objects) | No |
| **Optimistic Reads** | No | No | **Yes (Lock-free optimistic validation)** |

#### StampedLock Optimistic Read Example:

```java
public class Point {
    private double x, y;
    private final StampedLock sl = new StampedLock();

    // High-throughput lock-free optimistic read
    public double distanceFromOrigin() {
        long stamp = sl.tryOptimisticRead(); // Generates validation stamp without locking!
        double curX = x, curY = y;
        
        if (!sl.validate(stamp)) { // Check if a writer intervened
            stamp = sl.readLock(); // Fallback to pessimistic read lock
            try {
                curX = x;
                curY = y;
            } finally {
                sl.unlockRead(stamp);
            }
        }
        return Math.hypot(curX, curY);
    }
}
```

---

## 4. Hardware CAS & The ABA Problem

### Q5. What is the ABA Problem in Lock-Free algorithms and how do you resolve it?
**Answer:**
- **The Problem**: Thread 1 reads value $A$. Thread 2 changes $A \rightarrow B \rightarrow A$. When Thread 1 performs CAS (`compareAndSet(A, C)`), the CAS succeeds because the memory value is $A$, even though the object graph or state changed in the interim.
- **The Solution**: Use **`AtomicStampedReference`** or **`AtomicMarkableReference`**, which pairs each reference with an integer version/stamp:

```java
AtomicStampedReference<String> ref = new AtomicStampedReference<>("NodeA", 1);
int initialStamp = ref.getStamp();

// CAS checks both expected value AND expected stamp!
boolean success = ref.compareAndSet("NodeA", "NodeC", initialStamp, initialStamp + 1);
```

---

## 5. `ThreadLocal` Internals & Memory Leak Mechanics

### Q6. Deep Dive: Why does `ThreadLocal` cause memory leaks in thread pools?
**Answer:**

```
 [ Active Worker Thread ]
           |
           v
 [ ThreadLocalMap (Field of Thread) ]
           |
      [ Entry[] ]
     /           \
 (WeakReference)  (Strong Reference)
    /               \
[ ThreadLocal Key ]   [ Expensive Value Object (e.g., SecurityContext / UserSession) ]
```

1. Each `Thread` contains a field `ThreadLocal.ThreadLocalMap threadLocals`.
2. The `Entry` in `ThreadLocalMap` extends `WeakReference<ThreadLocal<?>>`.
3. If the strong reference to the `ThreadLocal` object is cleared, Garbage Collection removes the key (`key == null`), leaving an entry with a `null` key but a **strong reference to the Value object**.
4. In thread pools (e.g., Tomcat HTTP worker threads), threads **never die**. Therefore, the value object remains pinned in memory forever $\rightarrow$ **Heap Memory Leak**!

#### Production Safe Pattern:
```java
public class UserContextHolder {
    private static final ThreadLocal<UserContext> CONTEXT = new ThreadLocal<>();

    public static void set(UserContext ctx) { CONTEXT.set(ctx); }
    public static UserContext get() { return CONTEXT.get(); }
    
    // MUST ALWAYS BE CALLED IN A FINALLY BLOCK!
    public static void clear() { CONTEXT.remove(); }
}

// In Filter / Interceptor:
try {
    UserContextHolder.set(authContext);
    chain.doFilter(req, res);
} finally {
    UserContextHolder.clear(); // Prevents memory leaks and cross-request data pollution!
}
```

---

### Q7. What are `ScopedValue` in Java 21 (Project Loom)?
**Answer:**
**`ScopedValue` (JEP 446 / 464)** is a modern, lightweight, immutable alternative to `ThreadLocal` designed for Virtual Threads. 
- Unlike `ThreadLocal`, `ScopedValue` data is **immutable** and bound to a specific execution scope:
  ```java
  public final static ScopedValue<UserContext> CURRENT_USER = ScopedValue.newInstance();
  
  ScopedValue.where(CURRENT_USER, userContext)
             .run(() -> service.processOrder());
  ```
- Memory is automatically reclaimed when the bounded scope finishes, completely eliminating `ThreadLocal` leak hazards.

---

> **Next Chapter**: [07 Concurrency Primitives & Executors](07_Concurrency_Primitives_Executors.md)

