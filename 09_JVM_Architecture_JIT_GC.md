# 09. JVM Architecture, JIT Tiered Compilation & Garbage Collection

> **Navigation**: [Master Index](README.md) | [Previous: Virtual Threads](08_Virtual_Threads_Loom.md) | [Next: JVM Troubleshooting & Dumps](10_JVM_Troubleshooting_Dumps_Profiling.md)

---

## 📌 Chapter Overview
This module explores the HotSpot JVM runtime architecture, ClassLoader subsystem, Metaspace vs Heap memory layout, JIT tiered compilation optimizations (Escape Analysis, Inlining), and modern Garbage Collectors (**G1 GC vs ZGC**).

---

## 1. JVM Runtime Architecture

```
+-----------------------------------------------------------------------------------+
|                                 JVM RUNTIME ARCHITECTURE                          |
+-----------------------------------------------------------------------------------+
|  +-----------------------------------------------------------------------------+  |
|  |                           CLASSLOADER SUBSYSTEM                             |  |
|  |  Loading (Bootstrap -> Platform -> App) | Linking (Verify, Prepare, Resolve)|  |
|  |  Initialization (Static blocks & variables)                                |  |
|  +-----------------------------------------------------------------------------+  |
|                                                                                   |
|  +-----------------------------------------------------------------------------+  |
|  |                            RUNTIME DATA AREAS                               |  |
|  |  +-------------------------------------+  +-------------------------------+ |  |
|  |  |             HEAP MEMORY             |  |           METASPACE           | |  |
|  |  |  [ Young Gen: Eden | S0 | S1 ]      |  |  Class Metadata, Constants,   | |  |
|  |  |  [ Old/Tenured Generation    ]      |  |  Static Variables (Java 8+)   | |  |
|  |  +-------------------------------------+  +-------------------------------+ |  |
|  |  +---------------------+  +---------------------+  +----------------------+ |  |
|  |  |     JVM STACKS      |  |     PC REGISTERS    |  |  NATIVE METHOD STACK | |  |
|  |  | (Per-thread frames) |  | (Per-thread pointer)|  | (JNI calls)          | |  |
|  |  +---------------------+  +---------------------+  +----------------------+ |  |
|  +-----------------------------------------------------------------------------+  |
|                                                                                   |
|  +-----------------------------------------------------------------------------+  |
|  |                             EXECUTION ENGINE                                |  |
|  |  [ Interpreter ]  |  [ JIT Compiler (C1 / C2) ]  |  [ Garbage Collector ]  |  |
|  |  [ Profiler    ]  |  [ Code Cache             ]  |                         |  |
|  +-----------------------------------------------------------------------------+  |
+-----------------------------------------------------------------------------------+
```

### Q1. Detail the ClassLoader Subsystem and Delegation Model.
**Answer:**
1. **Hierarchy**:
   - **Bootstrap ClassLoader**: Native C++ loader built into JVM core. Loads base JDK classes (`java.lang.*`, `java.util.*` from `java.base`).
   - **Platform ClassLoader**: Loads platform extensions and JDK modular packages.
   - **Application (System) ClassLoader**: Loads application classes from `CLASSPATH` or module path.
   - **Custom ClassLoaders**: User-defined loaders for loading encrypted classes, plugins, or dynamic hot-reloading.
2. **Delegation Principle**:
   When requested to load a class, a ClassLoader delegates upward to its parent first. Only if the parent fails (`ClassNotFoundException`) does the child attempt to load it.
3. **Phases of Loading**:
   - **Loading**: Reads binary byte stream and generates `java.lang.Class` in Metaspace.
   - **Linking**:
     - *Verification*: Enforces bytecode safety rules.
     - *Preparation*: Allocates static fields with default values (`0`, `null`).
     - *Resolution*: Resolves symbolic references into direct memory pointers.
   - **Initialization**: Executes static blocks and assigns explicit static variable values.

---

## 2. JIT Compiler & Tiered Compilation Optimizations

### Q2. How does the JIT Compiler optimize hot code paths?
**Answer:**
The HotSpot JVM balances startup time and peak throughput using **Tiered Compilation**:
1. **Interpreter**: Executes bytecode immediately with zero compilation pause.
2. **C1 (Client Compiler)**: Quickly compiles warm methods with lightweight profiling.
3. **C2 (Server Compiler)**: Ingests runtime profiling data to apply aggressive global optimizations:
   - **Method Inlining**: Replaces method calls with the actual method body, removing call-stack frame overhead.
   - **Loop Unrolling**: Reduces branching checks inside loops.
   - **Devirtualization**: Replaces polymorphic virtual method calls with direct static calls when only one concrete class is observed.
   - **Escape Analysis**: Analyzes object scope. If an object does not escape the current method/thread:
     - **Scalar Replacement**: Disassembles object fields into primitive variables stored in CPU registers or stack frames, completely bypassing heap allocation!
     - **Lock Elision**: Strips away synchronization locks on non-escaping objects.

---

## 3. Garbage Collection Architecture & Modern Collectors

```
+-------------------------------------------------------------------------------+
|                       G1 GC REGION-BASED HEAP ARCHITECTURE                    |
+-------------------------------------------------------------------------------+
|   [ Eden ]    [ Free ]    [ Survivor ]    [ Old ]    [ Humongous ]   [ Eden ]  |
|   [ Old  ]    [ Eden ]    [ Humongous]    [ Free ]   [ Survivor  ]   [ Old  ]  |
|   [ Free ]    [ Old  ]    [ Eden     ]    [ Old  ]   [ Free      ]   [ Free ]  |
|   * Heap divided into ~2,048 equal-sized regions (1MB to 32MB each)           |
+-------------------------------------------------------------------------------+
```

### Q3. Compare modern Garbage Collectors: G1 GC vs. ZGC vs. Parallel GC.
**Answer:**

| Feature | Parallel GC | G1 GC (Default in Java 9+) | ZGC (Generational in Java 21) |
| :--- | :--- | :--- | :--- |
| **Primary Goal** | **Peak Throughput** (Batch ETL jobs) | **Balanced Throughput & Latency** | **Ultra-Low Latency** ($<1\text{ms}$ pause times) |
| **Heap Model** | Monolithic Young / Old Gen | **Region-based** (~2,048 dynamic regions) | **Region-based (Z-Pages)** with colored pointers |
| **Pause Times (STW)**| High ($100\text{ms} - 5\text{s}$) | Tunable target (Default: $200\text{ms}$) | **$< 1\text{ms}$ guaranteed**, regardless of heap size! |
| **Heap Sizing Range**| Small to Medium ($< 4\text{GB}$) | Medium to Large ($4\text{GB} - 64\text{GB}$) | Large to Massive ($8\text{GB} - 16\text{TB}$) |
| **JVM Activation Flag**| `-XX:+UseParallelGC` | `-XX:+UseG1GC` | `-XX:+UseZGC -XX:+ZGenerational` |

---

### Q4. What is a GC Root and how does Tracing Collector work?
**Answer:**
A **GC Root** is an anchor object that is directly accessible from outside the garbage-collected heap:
1. **Local variables and parameters** in active thread stack frames.
2. **Static variables** in loaded classes located in Metaspace.
3. **JNI (Java Native Interface)** global and local C pointers.
4. **Active live Thread objects**.

The Garbage Collector traverses the object reference graph starting from all GC Roots (Mark phase). Any object unreachable from the GC Root graph is designated as garbage and reclaimed during the Sweep/Compact phase.

---

> **Next Chapter**: [10 JVM Troubleshooting, Memory Dumps & Profiling](10_JVM_Troubleshooting_Dumps_Profiling.md)

