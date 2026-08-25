# 08. Virtual Threads (Project Loom - Java 21 LTS)

> **Navigation**: [Master Index](README.md) | [Previous: Concurrency Primitives](07_Concurrency_Primitives_Executors.md) | [Next: JVM Architecture & GC](09_JVM_Architecture_JIT_GC.md)

---

## 📌 Chapter Overview
This module explores **Virtual Threads (Project Loom - Java 21 LTS)**, carrier thread mounting, continuation mechanics, thread pinning diagnosis, and production anti-patterns.

---

## 1. Platform Threads vs. Virtual Threads

```
  PLATFORM THREADS (1:1 OS Mapping)             VIRTUAL THREADS (M:N User-Space Mapping)
+-----------------------------------+    +-----------------------------------------------+
|  Java Thread 1 <-> OS Thread 1    |    |  [ Virtual Thread 1 ] [ Virtual Thread 2 ] ...|
|  Java Thread 2 <-> OS Thread 2    |    |               \            /                  |
|  * ~1 MB fixed Stack Memory       |    |         Mounted / Unmounted on Demand         |
|  * Context switch via OS Kernel   |    |                      v                        |
|  * Max ~2,000-5,000 threads/JVM   |    |  [ Carrier Thread 1 ] [ Carrier Thread 2 ]    |
|                                   |    |  (ForkJoinPool matching CPU Core Count)       |
+-----------------------------------+    +-----------------------------------------------+
```

### Q1. Compare Platform Threads vs. Virtual Threads.
**Answer:**

| Feature | Platform Threads (`java.lang.Thread`) | Virtual Threads (Java 21 LTS) |
| :--- | :--- | :--- |
| **OS Mapping** | 1:1 with OS Kernel Thread | $M:N$ (Millions of virtual threads mapped onto few Carrier Threads) |
| **Stack Memory** | Fixed ~1 MB per thread (Pre-allocated) | Dynamic chunks starting at **~200–500 bytes** in Heap |
| **Context Switching**| Expensive OS Kernel Context Switch | Lightweight user-space JVM continuation swap |
| **Creation Cost** | Expensive ($~1\text{ms}$) $\rightarrow$ Requires Pooling | Negligible ($~1\mu\text{s}$) $\rightarrow$ **Never Pool!** |
| **Ideal Workload** | CPU-intensive computation | **I/O-intensive / High-latency network calls** |

---

## 2. Continuation Mechanics: Mounting & Unmounting

### Q2. How does a Virtual Thread unmount during blocking I/O?
**Answer:**
When a Virtual Thread executes a blocking operation (e.g., `Socket.read()`, `Thread.sleep()`, JDBC query, HTTP client call):
1. The JVM intercepts the blocking call via modern internal adapters.
2. The Virtual Thread's call stack is copied to the Java Heap as a **Continuation**.
3. The Virtual Thread **unmounts** from its underlying OS **Carrier Thread**.
4. The Carrier Thread is immediately freed to execute other waiting Virtual Threads.
5. Once the OS I/O event completes (notified via epoll / kqueue), the JVM scheduler **remounts** the Virtual Thread onto *any available* Carrier Thread and resumes execution seamlessly.

---

## 3. Thread Pinning: Diagnosis & Production Fixes

### Q3. What is Thread Pinning and how do you resolve it?
**Answer:**
**Thread Pinning** occurs when a Virtual Thread is blocked inside an operation but **cannot unmount** from its OS Carrier Thread, blocking the underlying kernel thread and degrading application throughput.

#### The 2 Root Causes of Pinning:
1. Executing inside a **`synchronized` block or method**.
2. Executing inside a **native method / JNI call**.

```
 PINNED THREAD HAZARD:
 [ Virtual Thread (Blocked on I/O) ] === PINNED ===> [ Carrier OS Thread ] (STALLED!)
      ^-- (Inside synchronized block)
```

#### Diagnostic Flag:
Run your JVM with the tracing flag to detect pinned threads in test environments:
```bash
-Djdk.tracePinnedThreads=full
```

#### The Production Fix:
Replace `synchronized` with **`ReentrantLock`** from `java.util.concurrent.locks`:

```java
// ❌ ANTI-PATTERN: Causes Thread Pinning in Java 21 Virtual Threads!
public class LegacyService {
    public synchronized String fetchData(String url) {
        return httpClient.get(url); // Blocks carrier thread while holding monitor lock!
    }
}

// ✅ PRODUCTION FIX: ReentrantLock allows Virtual Thread unmounting!
public class ModernService {
    private final ReentrantLock lock = new ReentrantLock();

    public String fetchData(String url) {
        lock.lock();
        try {
            return httpClient.get(url); // Virtual thread unmounts cleanly on blocking I/O!
        } finally {
            lock.unlock();
        }
    }
}
```

---

## 4. Senior Virtual Thread Anti-Patterns

### Q4. What are the common anti-patterns when adopting Virtual Threads?
**Answer:**

1. **Anti-Pattern 1: Pooling Virtual Threads**:
   - *Never* use `Executors.newFixedThreadPool(100)` with virtual threads.
   - *Fix*: Create a new virtual thread per task using `Executors.newVirtualThreadPerTaskExecutor()`.
2. **Anti-Pattern 2: Storing Heavy Objects in `ThreadLocal`**:
   - If 1,000,000 virtual threads each store a 10KB object in `ThreadLocal`, heap memory consumes **10 GB**, causing `OutOfMemoryError: Java heap space`.
   - *Fix*: Use immutable **`ScopedValue`** (Java 21).
3. **Anti-Pattern 3: Using Thread Pools for Concurrency Throttling**:
   - In legacy systems, bounded thread pools were used to limit database concurrency.
   - *Fix*: Use a **`Semaphore`** to throttle access to shared resources when running on virtual threads.

---

## 5. Enabling Virtual Threads in Spring Boot 3.2+

```properties
# Enable Virtual Threads across Spring MVC Tomcat and Async Task Executors
spring.threads.virtual.enabled=true
```

When enabled:
- Embedded Tomcat processes every incoming HTTP request inside a new Virtual Thread.
- `@Async` methods and `@Scheduled` tasks execute on virtual threads automatically.

---

> **Next Chapter**: [09 JVM Architecture, JIT Tiered Compilation & GC](09_JVM_Architecture_JIT_GC.md)

