# Java Basics & Core Foundations: 70+ Q&A

> **Navigation**: [Master Index](README.md) | [Study Roadmap](00_Study_Roadmap.md) | [Next: Modern Java & JVM](01_Modern_Java_JVM_Language.md)

---

## 📌 Chapter Overview

This foundational module provides deep-dive answers to core Java interview questions ranging from language architecture, data types, memory layout, OOP pillars, to exception hierarchies and common gotchas.

> [!NOTE]
> Even for senior engineering interviews, interviewers frequently start with core language questions to evaluate how deeply you understand memory semantics, wrapper caching, polymorphism mechanics, and object identity.

---

## 1. Introduction to Java & Platform Architecture

### Q1. What is Java and what are its core architectural pillars?
**Answer:**
Java is a class-based, object-oriented, high-level programming language designed around the principle of *"Write Once, Run Anywhere"* (WORA).

**Core Pillars:**
1. **Platform Independence**: Source code compiles into intermediate bytecode (`.class`), executed by any OS-specific JVM.
2. **Automatic Memory Management**: Managed heap with automatic Garbage Collection (GC).
3. **Strong Type Safety**: Strict compile-time and runtime type checking.
4. **Multi-threading & Concurrency Support**: Built-in language primitives (`synchronized`, `volatile`) and rich concurrent APIs (`java.util.concurrent`).
5. **Security**: Sandboxed execution, bytecode verification, and security managers/policies.

---

### Q2. Why is Java considered platform-independent, but the JVM is platform-dependent?
**Answer:**
- **Bytecode Platform Independence**: The Java compiler (`javac`) compiles `.java` source files into standardized `.class` bytecode instructions that are completely OS-agnostic.
- **JVM Platform Dependence**: The JVM executes bytecode by translating it into CPU-specific machine code (via Interpreter and JIT compiler). Therefore, a Windows JVM converts bytecode to Windows x86/x64 instructions, while a Linux JVM converts the same bytecode to Linux-specific machine instructions.

```
+--------------------+
|  Source Code .java |
+--------------------+
          | (javac)
          v
+--------------------+
|   Bytecode .class  |  <--- OS Agnostic
+--------------------+
     /     |      \
    v      v       v
[JVM Win] [JVM Mac] [JVM Linux] <--- Platform Specific
    |      |       |
[Win OS] [Mac OS] [Linux OS]
```

---

### Q3. What is the difference between Java, C++, and JavaScript?
**Answer:**

| Feature | Java | C++ | JavaScript |
| :--- | :--- | :--- | :--- |
| **Paradigm** | Pure OOP (mostly), Class-based | Multi-paradigm (OOP, Procedural) | Multi-paradigm, Prototype-based |
| **Execution** | Bytecode on JVM + JIT | Compiled directly to native machine code | Interpreted / JIT in V8/Browser/Node |
| **Memory** | Automatic GC | Manual (`malloc`/`free`, `new`/`delete`) | Automatic GC |
| **Pointers** | No explicit pointers / memory addresses | Direct pointer manipulation | No pointers |
| **Multiple Inheritance** | Interfaces only (Single class inheritance) | Multiple class inheritance supported | Prototypal inheritance |

---

## 2. JDK, JRE, JVM, and Bytecode

### Q4. Detail the differences between JDK, JRE, and JVM.
**Answer:**
- **JVM (Java Virtual Machine)**: The runtime execution engine that loads, verifies, and executes bytecode. It manages memory (Heap, Stack, Metaspace) and interacts with the host OS.
- **JRE (Java Runtime Environment)**: JVM + Core runtime libraries (`rt.jar` / java.base modules) required to *run* compiled Java programs. *(Note: Discontinued as a separate download since Java 11; modular runtimes are created via `jlink`)*.
- **JDK (Java Development Kit)**: Complete development suite containing JRE + Development Tools (`javac` compiler, `jar` packager, `jconsole`, `jstack`, `jmap`, `jdb`, `javadoc`).

```
+-------------------------------------------------------------+
|                            JDK                              |
|  +-------------------------------------------------------+  |
|  |                         JRE                           |  |
|  |  +---------------------+  +------------------------+  |  |
|  |  |         JVM         |  | Core Runtime Libraries |  |  |
|  |  +---------------------+  +------------------------+  |  |
|  +-------------------------------------------------------+  |
|  Development Tools: javac, javadoc, jstack, jconsole, jar   |
+-------------------------------------------------------------+
```

---

### Q5. What is JIT (Just-In-Time) compilation and Tiered Compilation?
**Answer:**
1. **Interpreter**: Starts executing bytecode immediately line-by-line without delay, but runs slowly.
2. **JIT Compiler**: Compiles frequently executed bytecode ("hot spots") directly into native CPU machine instructions.
3. **Tiered Compilation (Default in Java 8+)**:
   - **Level 0**: Interpreted code.
   - **Level 1-3 (C1 Client Compiler)**: Fast compilation with basic profiling and quick optimizations.
   - **Level 4 (C2 Server Compiler)**: Heavy optimizations (method inlining, loop unrolling, escape analysis, dead code elimination, devirtualization) for peak sustained throughput.

---

## 3. Program Structure & Main Method

### Q6. Why is `public static void main(String[] args)` declared this way?
**Answer:**
- **`public`**: Enables the JVM launcher (located outside the application package) to invoke the method.
- **`static`**: Allows the JVM to invoke the entry point without instantiating an object of the containing class.
- **`void`**: The application terminates when `main` completes (or all non-daemon threads exit); it does not return an integer process code directly to the JVM (process exit codes are sent via `System.exit(int)`).
- **`String[] args`**: Accepts command-line parameters passed at application startup.
- *Note:* Since Java 21 (Preview) and Java 22+, unnamed classes and instance main methods (`void main()`) are supported for simplified scripting.

---

### Q7. Can you overload or override the `main()` method?
**Answer:**
- **Overloading**: **YES**. You can have multiple `main` methods with different parameter signatures (e.g., `main(int[] args)`), but the JVM will only invoke the standard `public static void main(String[] args)` as the entry point.
- **Overriding**: **NO**. `static` methods belong to the class, not instance objects; they are hidden via method hiding, not dynamically dispatched/overridden polymorphically.

---

## 4. Data Types, Variables & Memory Semantics

### Q8. What are the 8 primitive data types, their sizes, and default values?
**Answer:**

| Type | Size | Default Value | Wrapper Class | Value Range |
| :--- | :--- | :--- | :--- | :--- |
| `byte` | 1 byte (8 bits) | `0` | `Byte` | -128 to 127 |
| `short` | 2 bytes (16 bits) | `0` | `Short` | -32,768 to 32,767 |
| `int` | 4 bytes (32 bits) | `0` | `Integer` | -2,147,483,648 to 2,147,483,647 |
| `long` | 8 bytes (64 bits) | `0L` | `Long` | -9,223,372,036,854,775,808 to 9,223,372,036,854,775,807 |
| `float` | 4 bytes (32 bits) | `0.0f` | `Float` | IEEE 754 32-bit floating point |
| `double` | 8 bytes (64 bits) | `0.0d` | `Double` | IEEE 754 64-bit double precision |
| `char` | 2 bytes (16 bits) | `'\u0000'` | `Character` | 0 to 65,535 (Unicode UTF-16) |
| `boolean` | JVM dependent | `false` | `Boolean` | `true` / `false` |

> [!NOTE]
> Instance/static variables receive default values automatically. **Local variables** do NOT receive default values and cause compilation errors if read before initialization.

---

### Q9. What is Autoboxing, Unboxing, and the Integer Cache trap?
**Answer:**
- **Autoboxing**: Automatic conversion of primitive $\rightarrow$ Wrapper (e.g., `int` to `Integer.valueOf(int)`).
- **Unboxing**: Automatic conversion of Wrapper $\rightarrow$ Primitive (e.g., `Integer.intValue()`).
- **The Integer Cache Pitfall**:
  Java caches wrapper instances for `Byte`, `Short`, `Integer`, `Long` in the range **`-128` to `127`**.

```java
Integer a = 100;
Integer b = 100;
System.out.println(a == b); // TRUE (both reference the cached object)

Integer c = 200;
Integer d = 200;
System.out.println(c == d); // FALSE (different objects on heap!)
System.out.println(c.equals(d)); // TRUE (value comparison)
```

---

### Q10. Is Java Pass-by-Value or Pass-by-Reference?
**Answer:**
Java is **strictly Pass-by-Value** at all times.
- For **primitives**: A copy of the actual literal value is passed to the method.
- For **objects**: A copy of the **reference handle (pointer value)** is passed by value.

```java
public void modify(Person p) {
    p.setName("Alice"); // Mutates the object pointed to by the copied reference
    p = new Person("Bob"); // Reassigns the local copied reference; caller still sees "Alice"!
}
```

---

## 5. Operators, Control Flow & String Equality

### Q11. Explain `==` versus `.equals()` contract.
**Answer:**
- `==`: Compares **primitives by value** and **objects by memory address (reference identity)**.
- `.equals()`: Method defined in `Object` (defaults to `==`). Overridden by classes like `String`, `Integer`, `LocalDate` to provide **logical value equality**.

```java
String s1 = new String("Java");
String s2 = new String("Java");

System.out.println(s1 == s2);      // FALSE (different heap locations)
System.out.println(s1.equals(s2));  // TRUE (same character contents)
```

---

### Q12. Why must `hashCode()` be overridden whenever `equals()` is overridden?
**Answer:**
The **`equals()` and `hashCode()` Contract**:
1. If `o1.equals(o2) == true`, then `o1.hashCode() == o2.hashCode()` **MUST ALWAYS** be true.
2. If `o1.hashCode() == o2.hashCode()`, `o1.equals(o2)` is **NOT** required to be true (hash collision).
3. If `equals()` is overridden without `hashCode()`, two logically equal objects will produce different hash codes. When inserted into a `HashMap` or `HashSet`, they will hash to different buckets, causing lookup failures, duplicate entries in `Set`, and broken collections.

```java
public class Employee {
    private final int id;
    private final String name;

    public Employee(int id, String name) {
        this.id = id;
        this.name = name;
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Employee that)) return false;
        return id == that.id && Objects.equals(name, that.name);
    }

    @Override
    public int hashCode() {
        return Objects.hash(id, name);
    }
}
```

---

## 6. Object-Oriented Programming (OOP) Deep Dive

### Q13. Explain the 4 Pillars of OOP with concrete enterprise examples.
**Answer:**
1. **Encapsulation**: Hiding internal object state and enforcing invariants via private fields and controlled access methods (e.g., validating bank transfer amounts before subtracting balance).
2. **Abstraction**: Exposing *what* an object does while hiding *how* it does it (e.g., `PaymentService` interface hides whether Stripe, PayPal, or Adyen executes the transfer).
3. **Inheritance**: Reusing code and forming "is-a" relationships (`CreditCardPayment` extends `BasePayment`).
4. **Polymorphism**:
   - *Compile-Time (Static)*: Method Overloading (`search(String query)`, `search(String query, Pageable page)`).
   - *Runtime (Dynamic)*: Method Overriding (`paymentService.processPayment()` executes specific subclass logic at runtime via virtual method table / vtable).

---

### Q14. Abstract Class vs. Interface (Post Java 8 & Java 9+).
**Answer:**

| Dimension | Abstract Class | Interface (Java 8/9/17+) |
| :--- | :--- | :--- |
| **Multiple Inheritance** | Single class inheritance only | Multiple interface implementation allowed |
| **State / Fields** | Can have instance state (non-static, non-final fields) | Only `public static final` constants |
| **Constructors** | Can have constructors (called via `super()`) | Cannot have constructors |
| **Method Types** | Abstract, concrete, final, static, private | Abstract, `default`, `static`, `private` (Java 9) |
| **Primary Purpose** | Code sharing & stateful base classes in an is-a hierarchy | API contracts, functional specifications, capabilities |

---

### Q15. Why does Java not support multiple class inheritance? How does it resolve the Diamond Problem?
**Answer:**
- **Why rejected for classes**: Multiple class inheritance causes ambiguity when two parent classes define the same field or method with different implementations (the **Diamond Problem**).
- **Interface Resolution**: Java 8 introduced `default` methods in interfaces. If a class implements two interfaces with identical default method signatures, the compiler forces the implementing class to explicitly resolve the conflict:

```java
interface Left {
    default void execute() { System.out.println("Left"); }
}
interface Right {
    default void execute() { System.out.println("Right"); }
}

public class DiamondResolver implements Left, Right {
    @Override
    public void execute() {
        // Explicitly choose or override
        Left.super.execute();
    }
}
```

---

## 7. Access Modifiers, Keywords & Immutability

### Q16. Detail Java Access Modifiers and their visibility scope.
**Answer:**

| Modifier | Same Class | Same Package | Subclass (Different Package) | World (Everywhere) |
| :--- | :---: | :---: | :---: | :---: |
| `private` | **YES** | NO | NO | NO |
| *Default (Package-private)* | **YES** | **YES** | NO | NO |
| `protected` | **YES** | **YES** | **YES** | NO |
| `public` | **YES** | **YES** | **YES** | **YES** |

---

### Q17. How do you design a 100% thread-safe Immutable Class?
**Answer:**
Rules for creating an immutable class:
1. Declare the class `final` (prevents subclasses from overriding mutators).
2. Make all fields `private` and `final`.
3. Provide **no setter/mutator** methods.
4. Initialize all fields in the constructor.
5. **Defensive Copying**: If fields are mutable objects (e.g., `Date`, `List`, `Map`), make deep copies in both the constructor and getters.

```java
public final class ImmutableUser {
    private final String username;
    private final List<String> roles;
    private final Date createdDate;

    public ImmutableUser(String username, List<String> roles, Date createdDate) {
        this.username = username;
        // Defensive copy on input
        this.roles = (roles != null) ? List.copyOf(roles) : List.of();
        this.createdDate = (createdDate != null) ? new Date(createdDate.getTime()) : null;
    }

    public String getUsername() { return username; }
    public List<String> getRoles() { return roles; } // List.copyOf is already unmodifiable
    public Date getCreatedDate() {
        // Defensive copy on output
        return (createdDate != null) ? new Date(createdDate.getTime()) : null;
    }
}
```

---

## 8. Strings, String Pool & Arrays

### Q18. Why are `String` objects immutable in Java?
**Answer:**
1. **String Constant Pool (SCP)**: Immutability allows multiple references to share the same String literal in the heap without risk of mutation.
2. **Security**: Strings are widely used for database URLs, passwords, file paths, and network connections. Immutability prevents parameters from being altered after validation.
3. **Thread Safety**: Immutable strings are inherently thread-safe and can be shared across threads without synchronization.
4. **HashCode Caching**: The hash code of a `String` is calculated once and cached lazily (`private int hash;`), enabling high performance in `HashMap` and `HashSet`.

---

### Q19. Compare `String`, `StringBuilder`, and `StringBuffer`.
**Answer:**

| Property | `String` | `StringBuilder` | `StringBuffer` |
| :--- | :--- | :--- | :--- |
| **Mutability** | Immutable | Mutable | Mutable |
| **Thread Safety** | Thread-safe (immutable) | Not thread-safe | Thread-safe (`synchronized` methods) |
| **Performance** | Slow for frequent concatenation | Fastest for single-threaded mutation | Slower due to synchronization overhead |
| **Introduced** | Java 1.0 | Java 1.5 | Java 1.0 |

---

## 9. Exception Handling Architecture

### Q20. Explain the Java Exception Hierarchy.
**Answer:**

```
                   +------------------+
                   |    Throwable     |
                   +------------------+
                       /          \
                      v            v
            +-----------+      +---------------+
            |   Error   |      |   Exception   |
            +-----------+      +---------------+
            (Unchecked)           /         \
                                 v           v
                     +--------------------+ +-------------------+
                     |  RuntimeException  | | Checked Exceptions|
                     +--------------------+ +-------------------+
                          (Unchecked)         (Compile-time)
```
- **`Error`**: Serious system-level conditions that applications should not catch (`OutOfMemoryError`, `StackOverflowError`).
- **Checked Exceptions**: Subclasses of `Exception` excluding `RuntimeException` (`IOException`, `SQLException`). The compiler forces callers to handle (`try-catch`) or declare (`throws`) them.
- **Unchecked Exceptions**: Subclasses of `RuntimeException` (`NullPointerException`, `IllegalArgumentException`, `IndexOutOfBoundsException`). Represent programming logic bugs.

---

### Q21. What is Try-With-Resources and the `AutoCloseable` interface?
**Answer:**
Introduced in Java 7, `try-with-resources` ensures that any resource implementing `java.lang.AutoCloseable` or `java.io.Closeable` is automatically closed at block completion, even if exceptions are thrown.

```java
// Multiple resources closed in reverse order of creation
try (var fis = new FileInputStream("input.txt");
     var reader = new BufferedReader(new InputStreamReader(fis))) {
    System.out.println(reader.readLine());
} catch (IOException e) {
    logger.error("Failed to read file", e);
}
```

---

## 10. Collections & Generics Foundations

### Q22. Compare `ArrayList` vs. `LinkedList`.
**Answer:**

| Operation | `ArrayList` (Dynamic Array) | `LinkedList` (Doubly-Linked List) |
| :--- | :--- | :--- |
| **`get(index)`** | O(1) (Direct random access) | O(N) (Sequential traversal) |
| **`add(element)`** | O(1) amortized (O(N) during resize) | O(1) |
| **`add(index, elem)`** | O(N) (Requires array shift) | O(N) to seek, O(1) to insert pointer |
| **Memory Overhead** | Low (contiguous array storage) | High (stores node objects with `prev`/`next` pointers) |
| **Cache Locality** | High CPU cache hit rate | Poor cache locality |

---

### Q23. What is Type Erasure in Java Generics?
**Answer:**
Generics provide compile-time type safety. During compilation, the Java compiler uses **Type Erasure** to strip all generic type parameters and replaces unbounded types with `Object` (or bounded types with their upper bound) and inserts appropriate casts.
- Therefore, at runtime, `List<String>` and `List<Integer>` both become raw `List`.
- **Implications**: You cannot do `new T()`, `new T[10]`, or `instanceof List<String>` at runtime.

---

## 11. Memory Areas, Garbage Collection & Threads

### Q24. What is the difference between Stack and Heap memory?
**Answer:**
- **Stack Memory**: Stores primitive local variables and references to heap objects for active method invocations. Each thread has its own private stack. Memory is allocated and deallocated in LIFO order upon method entry and return.
- **Heap Memory**: Stores all instantiated objects and class instances. The heap is shared across all threads and is managed by the Garbage Collector.

---

### Q25. What is a Garbage Collection Root (GC Root)?
**Answer:**
An object is eligible for garbage collection when it is no longer reachable from any active GC Root.

**Examples of GC Roots:**
1. Local variables in active thread stack frames.
2. Active running `Thread` objects.
3. Static variables referenced by loaded classes in Metaspace.
4. JNI (Java Native Interface) global and local references.

---

## 12. Common Senior Interview Pitfalls & Edge Cases

### Q26. Can `finally` block fail to execute?
**Answer:**
**YES**, in the following scenarios:
1. `System.exit(status)` is called inside `try` or `catch`.
2. Fatal JVM crash (e.g., `OutOfMemoryError`, Segfault / SIGKILL).
3. Infinite loop or deadlock in the `try` block.
4. Power loss or host OS termination.

---

### Q27. What happens if both `try` and `finally` have `return` statements?
**Answer:**
The `return` statement in the `finally` block **overrides and suppresses** the `return` statement (or any unhandled exception) from the `try` or `catch` block. This is considered an anti-pattern.

```java
public int test() {
    try {
        return 1;
    } finally {
        return 2; // Returns 2! Suppresses return 1.
    }
}
```

---

> **Next Chapter**: [00 Study Roadmap & Preparation Strategy](00_Study_Roadmap.md)
