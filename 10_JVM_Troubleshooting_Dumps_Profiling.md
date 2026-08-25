# 10. JVM Troubleshooting, Memory Dumps & Production Profiling

> **Navigation**: [Master Index](README.md) | [Previous: JVM Architecture & GC](09_JVM_Architecture_JIT_GC.md) | [Next: Coding & Data Structures](11_Coding_Data_Structures_Problems.md)

---

## 📌 Chapter Overview
A production-grade guide to diagnosing **Out-of-Memory Errors (OOM)**, capturing and analyzing **Thread Dumps** & **Heap Dumps**, finding memory leaks using Eclipse MAT, and configuring **Container-Aware JVM parameters in Kubernetes**.

---

## 1. Deep Dive: The 6 Types of `OutOfMemoryError`

### Q1. Detail the 6 types of `OutOfMemoryError` and their root causes.
**Answer:**

```
+-------------------------------------------------------------------------------+
|                       6 TYPES OF JAVA OUT OF MEMORY ERRORS                    |
+-------------------------------------------------------------------------------+
| 1. Java heap space              -> Objects exceed allocated Heap size         |
| 2. Metaspace                    -> Class metadata / Dynamic Proxies leaking   |
| 3. Direct buffer memory         -> Off-heap NIO / Netty memory leak           |
| 4. GC overhead limit exceeded   -> Spent >98% CPU in GC freeing <2% memory    |
| 5. unable to create new thread  -> Hit OS max process/thread limit (ulimit)   |
| 6. Compressed class space       -> 32-bit class pointer table exhausted       |
+-------------------------------------------------------------------------------+
```

| OOM Error Type | Root Cause | Diagnosis & Production Fix |
| :--- | :--- | :--- |
| **`Java heap space`** | • Undersized `-Xmx`<br>• Memory leak (unbounded caches, static collections, `ThreadLocalMap` leaks). | Capture heap dump (`.hprof`), inspect Dominator Tree in Eclipse MAT, fix leak or scale heap. |
| **`Metaspace`** | • Dynamic bytecode generation without caching (CGLIB, Spring AOP, Javassist, Groovy scripts).<br>• Custom ClassLoader leak during redeployments. | Set `-XX:MaxMetaspaceSize=512m`, enable class unloading (`-XX:+ClassUnloadingWithConcurrentMark`), cache proxy generators. |
| **`Direct buffer memory`** | • Off-heap NIO buffers (`ByteBuffer.allocateDirect()`) or Netty `ByteBuf` leaking without being released. | Set `-XX:MaxDirectMemorySize=1g`, audit Netty buffer `ReferenceCountUtil.release()`. |
| **`GC overhead limit exceeded`** | • Application is running on near 100% full heap. JVM spent $>98\%$ of CPU time executing GC but reclaimed $<2\%$ of heap. | High memory pressure indicator right before `Java heap space` crash. |
| **`unable to create new native thread`** | • Application spawned too many platform threads.<br>• Hit OS user thread limits (`ulimit -u`) or OS kernel PID limits (`/proc/sys/kernel/pid_max`). | Replace unbounded thread pools with bounded pools, migrate to Java 21 Virtual Threads, increase OS `ulimit -u`. |
| **`Compressed class space`** | • With `-XX:+UseCompressedClassPointers`, default 1GB compressed class space filled with unique classes. | Increase `-XX:CompressedClassSpaceSize=2g` or disable compressed pointers. |

---

## 2. Thread Dump Analysis & Deadlock Detection

### Q2. How do you capture and interpret a Java Thread Dump?
**Answer:**

#### 1. Capturing Thread Dumps in Production:
```bash
# Method 1: Using jcmd (Recommended)
jcmd <PID> Thread.print > threaddump.tdump

# Method 2: Using jstack
jstack -l <PID> > threaddump.tdump

# Method 3: Container / Kill signal (Dumps to stdout / stderr)
kill -3 <PID>
```

#### 2. Interpreting Thread States in Thread Dumps:

```
"http-nio-8080-exec-12" #45 daemon prio=5 os_prio=0 tid=0x00007f nid=0x1a3b waiting for monitor entry [0x00007f]
   java.lang.Thread.State: BLOCKED (on object monitor)
        at com.example.service.OrderService.processOrder(OrderService.java:42)
        - waiting to lock <0x000000076bf21000> (a java.lang.Object)
        - locked <0x000000076bf22000> (a java.lang.Object)
```

- **`RUNNABLE`**: Thread is actively executing in user space or waiting on OS network socket I/O (`SocketInputStream.socketRead0`).
- **`BLOCKED`**: Thread is waiting to acquire an intrinsic monitor lock (`synchronized`) held by another thread.
- **`WAITING` (parking)**: Thread is waiting indefinitely (`LockSupport.park()`, `ReentrantLock.lock()`, `CountDownLatch.await()`).
- **`TIMED_WAITING`**: Thread sleeping or waiting with a timeout (`Thread.sleep()`, `HikariDataSource.getConnection()`).

#### 3. Automatic Deadlock Section:
HotSpot JVM automatically appends a deadlock report at the bottom of the dump:
```
Found 1 deadlock.
================
"Thread-1":
  waiting to lock monitor 0x00007f (object 0x100, a ResourceA),
  which is held by "Thread-2"
"Thread-2":
  waiting to lock monitor 0x00007f (object 0x200, a ResourceB),
  which is held by "Thread-1"
```

---

## 3. Heap Dump Analysis with Eclipse MAT

### Q3. Step-by-step: How do you identify a Memory Leak from a `.hprof` file?
**Answer:**

```
1. Trigger Automatic Dump on Crash:
   -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/var/log/dumps/heap_dump.hprof

2. Or Trigger Live Production Dump:
   jcmd <PID> GC.heap_dump /var/log/dumps/live_dump.hprof
```

#### Analysis Steps in Eclipse Memory Analyzer (MAT):
1. **Open Heap Dump** and run the **Leak Suspects Report**.
2. **Inspect Dominator Tree**:
   - Lists objects ordered by **Retained Heap** (total memory kept alive if that object is garbage collected) vs **Shallow Heap** (memory consumed by the object's own fields).
3. **Trace Path to GC Roots**:
   - Right-click suspected leak object $\rightarrow$ `Path to GC Roots` $\rightarrow$ `exclude all phantom/weak/soft references`.
   - Reveals the exact static collection, Spring singleton, or unclosed `ThreadLocalMap` holding the object in memory!

---

## 4. Container & Kubernetes JVM Tuning Flags

### Q4. What are the essential JVM flags for Docker and Kubernetes?
**Answer:**

```bash
java \
  # 1. Container Memory Awareness (Forces JVM to read cgroup limits instead of host RAM)
  -XX:+UseContainerSupport \
  -XX:MaxRAMPercentage=75.0 \
  -XX:InitialRAMPercentage=75.0 \
  \
  # 2. Modern Low-Latency Garbage Collector
  -XX:+UseG1GC \
  -XX:MaxGCPauseMillis=200 \
  \
  # 3. Automatic Diagnostics on Crash
  -XX:+HeapDumpOnOutOfMemoryError \
  -XX:HeapDumpPath=/dumps/heap.hprof \
  -XX:+ExitOnOutOfMemoryError \
  \
  # 4. Modern Unified GC Logging
  -Xlog:gc*,gc+phases=debug:file=/var/log/gc.log:time,uptime,pid:filecount=5,filesize=50m \
  \
  -jar app.jar
```

> [!IMPORTANT]
> **Why `-XX:+ExitOnOutOfMemoryError` is critical in Kubernetes**:
> When a JVM throws an OOM, some threads die while others stay half-alive (e.g., health check endpoint responds 200 OK while business worker threads are dead). Adding `-XX:+ExitOnOutOfMemoryError` terminates the JVM process immediately, allowing Kubernetes to restart the unhealthy Pod!

---

> **Next Chapter**: [11 Coding, Data Structures & Concurrency Challenges](11_Coding_Data_Structures_Problems.md)

