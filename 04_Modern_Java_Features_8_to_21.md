# 04. Modern Java Features (Java 8 to 21 LTS)

> **Navigation**: [Master Index](README.md) | [Previous: Collections & Generics](03_Collections_Framework_Generics.md) | [Next: Java I/O & NIO](05_Java_IO_NIO_Netty.md)

---

## 📌 Chapter Overview
This module covers the modern language features introduced between **Java 8 and Java 21 LTS**, including functional programming with Streams, Records, Sealed Classes, Pattern Matching, and Switch Expressions.

---

## 1. Java 8 Functional Interfaces & Stream API

### Q1. Explain the 4 Core Functional Interfaces in `java.util.function`.
**Answer:**

| Interface | Method Signature | Purpose | Example |
| :--- | :--- | :--- | :--- |
| **`Predicate<T>`** | `boolean test(T t)` | Evaluates a condition | `x -> x > 0` |
| **`Function<T, R>`**| `R apply(T t)` | Transforms input `T` to output `R` | `User::getEmail` |
| **`Consumer<T>`** | `void accept(T t)` | Consumes input, produces side-effect | `System.out::println` |
| **`Supplier<T>`** | `T get()` | Generates/supplies a value | `() -> UUID.randomUUID()` |

---

### Q2. Stream API Deep Dive: Intermediate vs. Terminal Operations & Parallel Streams.
**Answer:**

#### 1. Intermediate vs. Terminal Operations:
- **Intermediate Operations (Lazy)**: Return a new `Stream`. Execution is deferred until a terminal operation is called (`filter()`, `map()`, `flatMap()`, `distinct()`, `sorted()`).
- **Terminal Operations (Eager)**: Triggers execution, consumes stream pipeline, and returns a concrete result or side-effect (`collect()`, `forEach()`, `reduce()`, `count()`, `findFirst()`).

#### 2. `map()` vs. `flatMap()`:
- `map(Function<T, R>)`: One-to-one transformation ($T \rightarrow R$).
- `flatMap(Function<T, Stream<R>>)`: One-to-many flattening transformation ($T \rightarrow \text{Stream}<R> \rightarrow \text{Flattened Stream}$).

```java
List<List<String>> nested = List.of(List.of("A", "B"), List.of("C", "D"));
List<String> flat = nested.stream()
                          .flatMap(Collection::stream)
                          .toList(); // ["A", "B", "C", "D"]
```

#### 3. Parallel Stream Hazards:
`stream.parallel()` utilizes the shared, JVM-wide **`ForkJoinPool.commonPool()`** (sized to `Runtime.getRuntime().availableProcessors() - 1`).

> [!WARNING]
> **Production Anti-Pattern**: Never execute blocking I/O (HTTP calls, DB queries) inside parallel streams! Doing so exhausts threads in `ForkJoinPool.commonPool()`, starving and halting all parallel operations across the entire application.

---

## 2. Java 9 to 11 Features

### Q3. `List.of()` vs. `Collections.unmodifiableList()`.
**Answer:**
- `Collections.unmodifiableList(list)`: Returns an unmodifiable **wrapper view** over the underlying list. If the original list is modified by another reference, the view reflects the changes.
- `List.of(...)` (Java 9+): Creates a true **100% Immutable and Compact** list instance backed by internal array fields. Rejects `null` elements immediately with `NullPointerException`.

---

### Q4. Local-Variable Type Inference (`var` - Java 10).
**Answer:**
`var` allows the compiler to infer static types at compile-time:
- **Valid**: Local variables with immediate initializers (`var list = new ArrayList<String>();`).
- **Invalid**: Method parameters, return types, class instance fields, or uninitialized variables (`var x;`).

---

## 3. Java 14 to 17 Features: Records & Sealed Classes

### Q5. Explain Records (Java 16) and Custom Constructors.
**Answer:**
A **Record** is a concise, immutable data carrier class. The compiler automatically generates:
- `private final` fields for all record components.
- Canonical constructor, accessors (`user.email()`), `equals()`, `hashCode()`, and `toString()`.

```java
public record UserDto(Long id, String username, String email) {
    // Compact Constructor for validation / normalization
    public UserDto {
        Objects.requireNonNull(username, "Username cannot be null");
        email = email == null ? "" : email.toLowerCase().trim();
    }
}
```

---

### Q6. Explain Sealed Classes & Interfaces (Java 17 LTS).
**Answer:**
**Sealed Classes (`sealed`, `permits`)** restrict which classes or interfaces may extend or implement them, enabling domain modeling with closed hierarchies:

```java
// Top-level sealed interface
public sealed interface PaymentStatus 
    permits PaymentStatus.Success, PaymentStatus.Failed, PaymentStatus.Pending {

    record Success(String transactionId, Instant timestamp) implements PaymentStatus {}
    record Failed(String errorCode, String reason) implements PaymentStatus {}
    record Pending(Duration estimatedDelay) implements PaymentStatus {}
}
```

Direct subclasses must be declared as either `final`, `sealed`, or `non-sealed` (open for unrestricted extension).

---

## 4. Java 21 LTS: Pattern Matching & Switch Enhancements

### Q7. Pattern Matching for `switch` and Record Patterns (Deconstruction).
**Answer:**
Java 21 introduces **Record Deconstruction** and **Pattern Matching for `switch` (JEP 440 & 441)**:

```java
public class PaymentNotificationService {

    public String formatNotification(PaymentStatus status) {
        // Switch pattern matching with exhaustive sealed hierarchy checking (No default needed!)
        return switch (status) {
            // Record Pattern Deconstruction directly in case label!
            case PaymentStatus.Success(var txId, var time) -> 
                "Payment successful! TX: " + txId + " at " + time;
            
            // Guarded Pattern using 'when' clause
            case PaymentStatus.Failed(var code, var reason) when "FRAUD_DETECTED".equals(code) -> 
                "CRITICAL ALERT: Transaction blocked due to fraud: " + reason;
            
            case PaymentStatus.Failed(var code, var reason) -> 
                "Payment failed [" + code + "]: " + reason;
            
            case PaymentStatus.Pending(var delay) -> 
                "Payment in progress. Estimated delay: " + delay.toSeconds() + "s";
        };
    }
}
```

---

### Q8. What is the Foreign Function & Memory API (Project Panama - Java 22 / 21 Preview)?
**Answer:**
Project Panama allows Java applications to safely interoperate with native code (C/C++ libraries) and off-heap memory outside the JVM, replacing the dangerous, slow, and error-prone **Java Native Interface (JNI)** with modern, safe, type-checked Java abstractions (`Arena`, `MemorySegment`, `Linker`).

---

> **Next Chapter**: [05 Java I/O, NIO & Netty Architecture](05_Java_IO_NIO_Netty.md)

