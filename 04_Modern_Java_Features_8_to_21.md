# 04. Modern Java Features, Functional Interfaces, Comparable vs. Comparator (Java 8 to 21 LTS)

> **Navigation**: [Master Index](README.md) | [Previous: Collections & Generics](03_Collections_Framework_Generics.md) | [Next: Java I/O & NIO](05_Java_IO_NIO_Netty.md)

---

## 📌 Chapter Overview
This module covers the modern language features introduced between **Java 8 and Java 21 LTS**, including the complete **Master Encyclopedia of Functional Interfaces (`java.util.function`)**, **`Comparable` vs. `Comparator`**, Stream API, Records, Sealed Classes, and Pattern Matching.

---

## 1. `Comparable<T>` vs. `Comparator<T>` Deep Dive

```
+-----------------------------------------------------------------------------------+
|                        COMPARABLE VS. COMPARATOR COMPARISON                       |
+-----------------------------------------------------------------------------------+
|  Feature              | Comparable<T>                     | Comparator<T>         |
|  -------------------- | --------------------------------- | --------------------- |
|  **Package**          | `java.lang`                       | `java.util`           |
|  **Method**           | `int compareTo(T o)`              | `int compare(T a, T b)`|
|  **Sorting Nature**   | **Natural / Default Ordering**    | **Custom Sorting Strategies** |
|  **Class Modification**| **Modifies original class**      | **External to class** |
|  **Number of Criteria**| Single default logic             | Multiple flexible criteria |
+-----------------------------------------------------------------------------------+
```

### Q1. Code Example: `Comparable` vs. `Comparator` with Fluent Chaining.
**Answer:**

```java
// 1. Comparable: Defines Natural Sorting by ID (Modifies the class)
public class Employee implements Comparable<Employee> {
    private Long id;
    private String department;
    private double salary;
    private String name;

    public Employee(Long id, String name, String dept, double salary) {
        this.id = id; this.name = name; this.department = dept; this.salary = salary;
    }

    @Override
    public int compareTo(Employee other) {
        return Long.compare(this.id, other.id); // Natural order by ID
    }

    // Getters...
    public Long getId() { return id; }
    public String getName() { return name; }
    public String getDepartment() { return department; }
    public double getSalary() { return salary; }
}

// 2. Comparator: External Custom Multi-Criteria Sorting Pipeline
public class EmployeeComparators {

    // Sort by Department ASC -> then Salary DESC -> then Name ASC (Nulls Last)
    public static final Comparator<Employee> MULTI_CRITERIA_SORT = Comparator
        .comparing(Employee::getDepartment)
        .thenComparing(Comparator.comparingDouble(Employee::getSalary).reversed())
        .thenComparing(Employee::getName, Comparator.nullsLast(String.CASE_INSENSITIVE_ORDER));
}

// Usage in Collections:
List<Employee> list = getEmployees();
Collections.sort(list); // Uses Comparable (by ID)
list.sort(EmployeeComparators.MULTI_CRITERIA_SORT); // Uses custom Comparator!
```

---

## 2. Master Encyclopedia of Functional Interfaces (`java.util.function`)

```
+-----------------------------------------------------------------------------------+
|                   THE 4 CORE FUNCTIONAL INTERFACE ARCHETYPES                      |
+-----------------------------------------------------------------------------------+
| 1. Predicate<T>        : (T)  -> boolean       [Condition / Filter evaluation]     |
| 2. Function<T, R>      : (T)  -> R             [Data Transformation / Mapping]    |
| 3. Consumer<T>         : (T)  -> void          [Action / Side-effect execution]   |
| 4. Supplier<T>         : ()   -> T             [Lazy Factory / Value Generation]  |
+-----------------------------------------------------------------------------------+
```

### Q2. Detail the 4 Core Archetypes, 2-Arity Variants, and Operators with Code.
**Answer:**

#### 1. The 4 Core Functional Interfaces:
```java
// 1. Predicate<T> -> Evaluates boolean condition
Predicate<String> isEmail = s -> s != null && s.contains("@");
Predicate<String> isLong = s -> s.length() > 5;
Predicate<String> validEmail = isEmail.and(isLong); // Chaining with and/or/negate

// 2. Function<T, R> -> Transforms T into R
Function<String, Integer> stringLength = String::length;
Function<Integer, Integer> square = x -> x * x;
Function<String, Integer> lengthSquared = stringLength.andThen(square); // Chaining

// 3. Consumer<T> -> Consumes T, returns void (side-effect)
Consumer<String> logger = msg -> log.info("AUDIT: {}", msg);
Consumer<String> emailer = msg -> emailService.send(msg);
Consumer<String> auditPipeline = logger.andThen(emailer); // Chaining

// 4. Supplier<T> -> Takes no input, supplies a new T (Lazy creation)
Supplier<Double> randomScore = Math::random;
Supplier<String> traceIdSupplier = () -> UUID.randomUUID().toString();
```

---

#### 2. Two-Arity Variants (Two Inputs):
| Interface | Signature | Core Purpose | Example |
| :--- | :--- | :--- | :--- |
| **`BiPredicate<T, U>`** | `boolean test(T t, U u)` | Evaluates condition on 2 inputs | `(user, role) -> user.hasRole(role)` |
| **`BiFunction<T, U, R>`**| `R apply(T t, U u)` | Transforms 2 inputs into 1 output | `(salary, bonus) -> salary + bonus` |
| **`BiConsumer<T, U>`** | `void accept(T t, U u)` | Consumes 2 inputs (e.g. Map iteration)| `(k, v) -> System.out.println(k + "=" + v)` |

---

#### 3. Operator Variants (Input Type Matches Output Type):
```java
// 1. UnaryOperator<T> (Specialization of Function<T, T>)
UnaryOperator<String> sanitize = s -> s.trim().toLowerCase();

// 2. BinaryOperator<T> (Specialization of BiFunction<T, T, T>)
BinaryOperator<BigDecimal> addPrices = BigDecimal::add;
BinaryOperator<Integer> maxOp = BinaryOperator.maxBy(Integer::compareTo);
```

---

#### 4. Primitive Specializations (High-Performance Zero-Boxing):
Autoboxing primitive types (`int` $\rightarrow$ `Integer`) inside tight loops generates huge garbage collection overhead. Java provides primitive specializations to eliminate wrapper object creation:

| Primitive Interface | Input | Output | Memory Advantage |
| :--- | :--- | :--- | :--- |
| **`IntPredicate`** | `int` | `boolean` | Zero boxing (`int -> boolean`) |
| **`LongConsumer`** | `long` | `void` | Zero boxing |
| **`DoubleFunction<R>`**| `double`| `R` | Prevents boxing `double` input |
| **`ToIntFunction<T>`** | `T` | `int` | Prevents boxing return value |
| **`IntToLongFunction`**| `int` | `long` | 100% Primitive pipeline ($O(1)$ RAM) |

---

## 3. Stream API Deep Dive: Intermediate vs. Terminal Operations

### Q3. `map()` vs. `flatMap()` & Parallel Stream Hazards.
**Answer:**

#### 1. `map()` vs. `flatMap()`:
- `map(Function<T, R>)`: One-to-one transformation ($T \rightarrow R$).
- `flatMap(Function<T, Stream<R>>)`: One-to-many flattening transformation ($T \rightarrow \text{Stream}<R> \rightarrow \text{Flattened Stream}$).

```java
List<List<String>> nested = List.of(List.of("A", "B"), List.of("C", "D"));
List<String> flat = nested.stream()
                          .flatMap(Collection::stream)
                          .toList(); // ["A", "B", "C", "D"]
```

#### 2. Parallel Stream Hazards:
`stream.parallel()` utilizes the shared, JVM-wide **`ForkJoinPool.commonPool()`** (sized to `Runtime.getRuntime().availableProcessors() - 1`).

> [!WARNING]
> **Production Anti-Pattern**: Never execute blocking I/O (HTTP calls, DB queries) inside parallel streams! Doing so exhausts threads in `ForkJoinPool.commonPool()`, starving and halting all parallel operations across the entire application.

---

## 4. Java 14 to 17 Features: Records & Sealed Classes

### Q4. Explain Records (Java 16) and Compact Constructors.
**Answer:**

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

### Q5. Explain Sealed Classes & Interfaces (Java 17 LTS).
**Answer:**

```java
// Top-level sealed interface
public sealed interface PaymentStatus 
    permits PaymentStatus.Success, PaymentStatus.Failed, PaymentStatus.Pending {

    record Success(String transactionId, Instant timestamp) implements PaymentStatus {}
    record Failed(String errorCode, String reason) implements PaymentStatus {}
    record Pending(Duration estimatedDelay) implements PaymentStatus {}
}
```

---

## 5. Java 21 LTS: Pattern Matching & Switch Enhancements

### Q6. Pattern Matching for `switch` and Record Patterns (Deconstruction).
**Answer:**

```java
public class PaymentNotificationService {

    public String formatNotification(PaymentStatus status) {
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

> **Next Chapter**: [05 Java I/O, NIO & Netty Architecture](05_Java_IO_NIO_Netty.md)
