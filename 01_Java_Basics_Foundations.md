# 01. Java Basics & Core Foundations

> **Navigation**: [Master Index](README.md) | [Study Roadmap](00_Study_Roadmap.md) | [Next: Strings & Memory](02_Strings_Memory_Algorithms.md)

---

## 📌 Chapter Overview
This module establishes the core foundations of the Java language, runtime execution mechanics, object-oriented design principles, memory semantics, and exception architecture.

---

## 1. Introduction to Java & Platform Architecture

### Q1. What is Java and what are its core architectural pillars?
**Answer:**
Java is a high-level, class-based, object-oriented, concurrent, secure, and general-purpose programming language.

#### Core Pillars:
1. **Platform Independence (WORA - Write Once, Run Anywhere)**: Java source code (`.java`) compiles into intermediate **Bytecode** (`.class`), which executes on any operating system equipped with a compatible **Java Virtual Machine (JVM)**.
2. **Automatic Memory Management**: The JVM manages dynamic heap allocation and deallocation via the **Garbage Collector (GC)**, removing manual memory management (`malloc`/`free`) and preventing dangling pointer defects.
3. **Strong Type Safety**: Java enforces strict compile-time and runtime type checking.
4. **Multithreaded Runtime**: Native support for multithreaded concurrent execution at the language level (`Thread`, `synchronized`, JMM, and Java 21 Virtual Threads).

---

### Q2. Why is Java platform-independent, but the JVM is platform-dependent?
**Answer:**
- **Java Bytecode is Universal**: Bytecode is an instruction set designed for an abstract machine. A `.class` file generated on macOS runs identically on Windows or Linux.
- **The JVM Translates Bytecode to Native Machine Instructions**: The JVM must interact directly with the underlying host OS kernel and CPU architecture (x86_64, ARM64, etc.).
  - Windows requires a Windows-specific JVM (`jvm.dll`, Win32 system calls).
  - Linux requires a Linux-specific JVM (`libjvm.so`, POSIX system calls).

```
 [ Source: App.java ] 
         |  javac (Java Compiler)
         v
 [ Bytecode: App.class ] (Universal format)
         |
    +----+----------------------------+
    |                                 |
    v                                 v
[ Linux JVM (ARM64) ]        [ Windows JVM (x86_64) ]
    |                                 |
    v                                 v
[ Linux OS / Kernel ]        [ Windows OS / Kernel ]
```

---

## 2. JDK vs. JRE vs. JVM & JIT Execution

### Q3. Explain the differences between JDK, JRE, and JVM.
**Answer:**

```
+-----------------------------------------------------------------------+
|                      JDK (Java Development Kit)                       |
|  [ javac, javadoc, jar, jcmd, jstack, jconsole, VisualVM, JFR ]       |
|                                                                       |
|  +-----------------------------------------------------------------+  |
|  |                   JRE (Java Runtime Environment)                |  |
|  |  [ Base Class Libraries (java.base, rt.jar), Core Configs ]     |  |
|  |                                                                 |  |
|  |  +-----------------------------------------------------------+  |  |
|  |  |                 JVM (Java Virtual Machine)                |  |  |
|  |  |  - ClassLoader Subsystem                                  |  |  |
|  |  |  - Runtime Data Areas (Heap, Metaspace, Stack)            |  |  |
|  |  |  - Execution Engine (Interpreter + JIT C1/C2 + GC)        |  |  |
|  |  +-----------------------------------------------------------+  |  |
|  +-----------------------------------------------------------------+  |
+-----------------------------------------------------------------------+
```

- **JVM (Java Virtual Machine)**: The abstract runtime engine that loads, verifies, and executes bytecode.
- **JRE (Java Runtime Environment)**: Contains the JVM plus runtime class libraries (`java.base`, etc.). *Note: In modern Java (Java 11+), standalone JRE downloads were replaced by custom runtime images generated using `jlink`.*
- **JDK (Java Development Kit)**: Complete developer bundle containing the JRE/JVM plus development tools (`javac`, `jar`, debuggers, profilers).

---

### Q4. What is JIT (Just-In-Time) compilation and Tiered Compilation?
**Answer:**
The JVM combines an **Interpreter** with a **JIT Compiler** to balance fast application startup with peak long-term execution speed:

1. **Interpreter**: Reads and executes bytecode instructions sequentially on startup with zero compilation delay.
2. **C1 Compiler (Client Compiler)**: When a method becomes "warm" (frequently called), C1 compiles it to native machine code with basic optimizations.
3. **C2 Compiler (Server Compiler)**: When a method becomes "hot", C2 applies aggressive optimizations:
   - **Method Inlining**: Replaces method calls with the actual body to remove call-stack overhead.
   - **Loop Unrolling**: Reduces branching checks inside loops.
   - **Dead Code Elimination**: Removes non-reachable logic branches.
   - **Escape Analysis**: Allocates non-escaping objects on the stack or into CPU registers (Scalar Replacement), bypassing the GC heap.

---

## 3. Data Types, Memory Semantics & Variables

### Q5. What are the 8 primitive types, sizes, and the Integer Cache trap?
**Answer:**

| Primitive Type | Size | Wrapper Class | Default Value | Value Range |
| :--- | :--- | :--- | :--- | :--- |
| `byte` | 1 byte (8 bits) | `Byte` | `0` | -128 to 127 |
| `short` | 2 bytes (16 bits) | `Short` | `0` | -32,768 to 32,767 |
| `int` | 4 bytes (32 bits) | `Integer` | `0` | $-2^{31}$ to $2^{31}-1$ |
| `long` | 8 bytes (64 bits) | `Long` | `0L` | $-2^{63}$ to $2^{63}-1$ |
| `float` | 4 bytes (32 bits) | `Float` | `0.0f` | IEEE 754 floating point |
| `double` | 8 bytes (64 bits) | `Double` | `0.0d` | IEEE 754 double precision |
| `char` | 2 bytes (16 bits) | `Character` | `'\u0000'` | `0` to `65,535` (Unicode UTF-16) |
| `boolean` | JVM-dependent | `Boolean` | `false` | `true` or `false` |

#### The Integer Cache Trap (`-128` to `127`):
Java caches `Integer` objects in the range `[-128, 127]` during Autoboxing (`Integer.valueOf()`):

```java
Integer a = 100;
Integer b = 100;
System.out.println(a == b); // true (Points to the same cached instance in IntegerCache)

Integer x = 200;
Integer y = 200;
System.out.println(x == y); // false (Different heap object references allocated!)
System.out.println(x.equals(y)); // true (Always use .equals() for object comparisons!)
```

---

### Q6. Is Java Pass-by-Value or Pass-by-Reference?
**Answer:**
**Java is 100% strictly Pass-by-Value.** There is NO pass-by-reference in Java.

- **Primitives**: The actual literal bit value is copied.
- **Objects**: The **object reference value (memory address pointer)** is copied by value. Mutating the object's internal fields affects the original object, but reassigning the reference itself does not change the caller's reference.

```java
public void modify(User u, int x) {
    x = 50;                     // Caller's primitive x remains unchanged
    u.setName("Alice");         // Mutates the object on the heap (visible to caller)
    u = new User("Bob");        // Reassigns local copy of pointer (caller STILL points to Alice)
}
```

---

## 4. Object Equality: `==` vs `.equals()` & `hashCode()` Contract

### Q7. Explain the `equals()` and `hashCode()` contract.
**Answer:**
- `==` operator: Compares primitive values OR memory address references for objects.
- `.equals()` method: Defined in `java.lang.Object` (defaults to `==`). Overridden to evaluate logical business equivalence.

#### The Mandatory Contract:
1. **Consistency**: If `a.equals(b) == true`, then `a.hashCode() == b.hashCode()` **MUST ALWAYS** be true.
2. **Unequal Objects**: If `a.equals(b) == false`, `a.hashCode()` and `b.hashCode()` do *not* have to be distinct, but distinct hash codes improve hash table (`HashMap`) performance.
3. **If you override `equals()`, you MUST override `hashCode()`**:
   - Failing to do so breaks hash-based collections (`HashMap`, `HashSet`, `ConcurrentHashMap`).
   - If two objects are logically equal but return different hash codes, `map.get(key)` will search the wrong bucket and return `null`.

```java
public final class Employee {
    private final Long id;
    private final String email;

    public Employee(Long id, String email) {
        this.id = id;
        this.email = email;
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Employee other)) return false;
        return Objects.equals(id, other.id) && Objects.equals(email, other.email);
    }

    @Override
    public int hashCode() {
        return Objects.hash(id, email);
    }
}
```

---

## 5. Object-Oriented Programming (OOP) Deep Dive

### Q8. Explain the 4 Pillars of OOP with Enterprise Application Examples.
**Answer:**

1. **Encapsulation**: Bundling state and behavior within a class while restricting direct access to internal fields (`private` fields exposed via validated methods/accessors).
2. **Abstraction**: Hiding internal implementation complexity behind clear public interfaces (`PaymentProcessor` interface decoupling Stripe vs PayPal implementations).
3. **Inheritance**: Reusing common code structures through parent-child class hierarchies (`abstract class BaseEntity` providing `id`, `createdAt`, `updatedAt`).
4. **Polymorphism**:
   - **Compile-time (Static / Overloading)**: Same method name with different parameter signatures.
   - **Runtime (Dynamic / Overriding)**: Method call resolution determined by the runtime object instance via JVM virtual method dispatch (`invokevirtual`).

---

### Q9. Abstract Class vs. Interface (Java 8 to 21).
**Answer:**

| Feature | Interface | Abstract Class |
| :--- | :--- | :--- |
| **Multiple Inheritance** | Yes (`implements A, B, C`) | No (`extends SingleClass` only) |
| **State / Fields** | Only `public static final` constants | Instance variables of any visibility |
| **Constructors** | None | Yes (called via `super()`) |
| **Method Types** | `abstract`, `default`, `static`, `private` (Java 9+) | Any method signature and access modifier |
| **Design Purpose** | Defines a contract / capability (*"Can-Do"*) | Defines an identity / partial template (*"Is-A"*) |

---

### Q10. How do you design a 100% Thread-Safe Immutable Class?
**Answer:**

```java
// 1. Declare class as 'final' to prevent subclass extension
public final class ImmutableOrder {

    // 2. Make all fields 'private' and 'final'
    private final String orderId;
    private final List<String> lineItems;
    private final Date orderDate;

    // 3. Initialize all fields via constructor with Deep Defensive Copies
    public ImmutableOrder(String orderId, List<String> lineItems, Date orderDate) {
        this.orderId = orderId;
        // Defensive copy mutable collections
        this.lineItems = lineItems == null ? List.of() : List.copyOf(lineItems);
        // Defensive copy mutable objects
        this.orderDate = orderDate == null ? null : new Date(orderDate.getTime());
    }

    public String getOrderId() {
        return orderId;
    }

    // 4. Return unmodifiable or defensive copies from getters
    public List<String> getLineItems() {
        return lineItems; // List.copyOf is already unmodifiable
    }

    public Date getOrderDate() {
        return orderDate == null ? null : new Date(orderDate.getTime());
    }
    // 5. Provide NO setter methods!
}
```

---

## 6. Exception Handling Architecture

### Q11. Explain the Exception Hierarchy: Checked vs Unchecked vs Error.
**Answer:**

```
                        +---------------------+
                        | java.lang.Throwable |
                        +---------------------+
                                   |
                +------------------+------------------+
                |                                     |
        +---------------+                     +---------------+
        | java.lang.Error|                     |java.lang.Exception|
        +---------------+                     +---------------+
        (JVM / Fatal)                                 |
        - OutOfMemoryError                    +-------+-------+
        - StackOverflowError                  |               |
        - InternalError                (Checked)       (Unchecked / Runtime)
                                       - IOException   - NullPointerException
                                       - SQLException  - IllegalArgumentException
                                       - ClassNotFound - IndexOutOfBoundsException
```

1. **`Throwable`**: The root of all throwable errors and exceptions.
2. **`Error`**: Serious hardware or JVM-level conditions that applications should never attempt to catch or recover from (`OutOfMemoryError`, `StackOverflowError`).
3. **Checked Exceptions (`Exception` excluding `RuntimeException`)**:
   - Checked at compile-time. The compiler forces methods to declare them in the `throws` clause or handle them inside `try-catch`.
   - Used for recoverable operational failures (`FileNotFoundException`, `SQLException`).
4. **Unchecked Exceptions (`RuntimeException`)**:
   - Not checked at compile-time. Indicate programming bugs, contract violations, or illegal operations (`NullPointerException`, `IllegalArgumentException`).

---

### Q12. How does Try-With-Resources work internally?
**Answer:**
Introduced in Java 7, **Try-With-Resources** guarantees that any resource implementing `java.lang.AutoCloseable` or `java.io.Closeable` is automatically closed when exiting the block, even if exceptions are thrown.

```java
// Automatic resource deallocation
try (BufferedReader br = new BufferedReader(new FileReader("data.csv"));
     Connection conn = dataSource.getConnection()) {
    return br.readLine();
} catch (IOException | SQLException e) {
    log.error("Resource operation failed", e);
    throw new ServiceException("Failed to read data", e);
}
```

#### Suppressed Exceptions:
If an exception occurs inside the `try` block AND another exception occurs while closing the resource in `close()`, Java preserves the primary `try` block exception and attaches the secondary exception as a **Suppressed Exception** accessible via `e.getSuppressed()`.

---

### Q13. How does Exception Propagation work in the JVM (Call Stack Unwinding & Thread Boundaries)?
**Answer:**

```
+-----------------------------------------------------------------------------------+
|                        JVM CALL STACK UNWINDING MECHANISM                         |
+-----------------------------------------------------------------------------------+
|  [ Thread Call Stack ]                                                            |
|                                                                                   |
|  Frame 3: processOrder()   ---> Throws PaymentException!                          |
|                                 (No catch block -> Stack Frame 3 POPPED / Unwound) |
|                                       |                                           |
|  Frame 2: checkoutService()---> (No catch block -> Stack Frame 2 POPPED / Unwound) |
|                                       |                                           |
|  Frame 1: orderController()---> +---------------------------------------------+   |
|                                 | try { checkout(); } catch(PaymentException)|   |
|                                 +---------------------------------------------+   |
|                                 (MATCH FOUND! Exception Handled, Stack stops!)   |
+-----------------------------------------------------------------------------------+
```

1. **Call Stack Unwinding**: When an exception is thrown, the JVM pauses normal execution and searches the current method frame for a matching `catch` block. If none is found, the current stack frame is **popped/unwound**, and the exception propagates to the calling method. This repeats until a matching handler is found or the main thread stack is exhausted, terminating the thread.
2. **Thread Boundaries Hazard**:
   - **Exceptions do NOT cross thread boundaries automatically!**
   - If Thread A spawns Thread B, an uncaught exception in Thread B terminates Thread B silently without Thread A knowing.
   - *Fixes*:
     - Use `thread.setUncaughtExceptionHandler(...)`.
     - Use `Future.get()` (rethrows execution failure as `ExecutionException`).
     - Use `CompletableFuture.exceptionally()` or `.handle()`.

---

### Q14. How do you design an Enterprise Custom Exception Hierarchy?
**Answer:**

```java
// 1. Base Abstract Domain Exception (Unchecked RuntimeException)
public abstract class BaseDomainException extends RuntimeException {
    private final String errorCode;
    private final HttpStatus httpStatus;
    private final Instant timestamp;

    protected BaseDomainException(String message, String errorCode, HttpStatus httpStatus) {
        super(message);
        this.errorCode = errorCode;
        this.httpStatus = httpStatus;
        this.timestamp = Instant.now();
    }

    public String getErrorCode() { return errorCode; }
    public HttpStatus getHttpStatus() { return httpStatus; }
    public Instant getTimestamp() { return timestamp; }
}

// 2. Concrete Sub-Exceptions with Specific Semantics
public class ResourceNotFoundException extends BaseDomainException {
    public ResourceNotFoundException(String resourceName, Object identifier) {
        super(String.format("%s with ID '%s' not found", resourceName, identifier),
              "ERR_NOT_FOUND", HttpStatus.NOT_FOUND);
    }
}

public class BusinessValidationException extends BaseDomainException {
    public BusinessValidationException(String message) {
        super(message, "ERR_BUSINESS_VALIDATION", HttpStatus.UNPROCESSABLE_ENTITY);
    }
}

public class InsufficientFundsException extends BaseDomainException {
    public InsufficientFundsException(Long accountId, double required, double available) {
        super(String.format("Account %d has insufficient funds: required $%.2f, available $%.2f", accountId, required, available),
              "ERR_INSUFFICIENT_FUNDS", HttpStatus.BAD_REQUEST);
    }
}
```

---

## 7. Senior Interview Gotchas & Edge Cases

### Q15. Can a `finally` block fail to execute?
**Answer:**
Yes, in four specific scenarios:
1. `System.exit(0)` is invoked before the `finally` block is entered.
2. The JVM crashes fatally with an `OutOfMemoryError` or native OS segmentation fault.
3. The underlying OS process or container receives a `SIGKILL` (`kill -9`).
4. An infinite loop (`while(true) {}`) or thread deadlock occurs inside the `try` or `catch` block.

---

### Q16. What happens if both `try` and `finally` have `return` statements?
**Answer:**
The `return` statement in the `finally` block **silently overrides and swallows** any `return` statement or uncaught exception from the `try` or `catch` block!

```java
public int test() {
    try {
        throw new RuntimeException("Error occurred");
    } finally {
        return 100; // Anti-pattern: Silently swallows the exception and returns 100!
    }
}
```
*Rule: Never put `return` or throw exceptions inside a `finally` block.*

---

> **Next Chapter**: [02 Strings, Memory Layout & Algorithms](02_Strings_Memory_Algorithms.md)

