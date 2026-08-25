# 13. Spring Core, IoC Container, AOP Internals & Custom Annotations

> **Navigation**: [Master Index](README.md) | [Previous: SOLID & Design Patterns](12_SOLID_Principles_Design_Patterns.md) | [Next: Spring Boot 3 Internals](14_Spring_Boot_3_Internals_Actuator.md)

---

## 📌 Chapter Overview
This module dives into the internal mechanics of the **Spring IoC Container**, the complete **Spring Bean Lifecycle**, 3-level caching for circular dependency resolution, **Spring AOP (Pointcuts, JoinPoints, Advices)**, and building **Custom Java Annotations with AOP Interceptors**.

---

## 1. Spring Bean Lifecycle Step-by-Step

```
+-----------------------------------------------------------------------------------+
|                            SPRING BEAN LIFECYCLE                                  |
+-----------------------------------------------------------------------------------+
|  1. Bean Definition Loading (Reads @Component, @Bean, XML)                        |
|  2. Constructor Instantiation (new MyService())                                   |
|  3. Dependency Injection / Populate Properties (@Autowired fields/setters)        |
|  4. Aware Interfaces (BeanNameAware -> BeanFactoryAware -> ApplicationContextAware)|
|  5. BeanPostProcessor: postProcessBeforeInitialization()                          |
|  6. Initialization Callback (@PostConstruct -> InitializingBean -> initMethod)   |
|  7. BeanPostProcessor: postProcessAfterInitialization() (AOP PROXY CREATED HERE!) |
|  8. Bean Ready for Production Use in ApplicationContext                           |
|  9. Container Shutdown: @PreDestroy -> DisposableBean -> destroyMethod            |
+-----------------------------------------------------------------------------------+
```

### Q1. Why is AOP Dynamic Proxying performed in `postProcessAfterInitialization()`?
**Answer:**
A proxy wraps the target instance to intercept method invocations. To ensure the target object is fully instantiated, populated with dependencies, and initialized with all `@PostConstruct` setup logic before being wrapped, Spring performs proxy creation in the final `BeanPostProcessor.postProcessAfterInitialization()` phase.

---

## 2. Circular Dependencies & The 3-Level Cache

```
+-------------------------------------------------------------------------------+
|                 SPRING 3-LEVEL CACHE (DefaultSingletonBeanRegistry)           |
+-------------------------------------------------------------------------------+
|  1. singletonObjects        (Map<String, Object>)                             |
|     -> Fully initialized, injected, and ready-to-use Singleton beans.         |
|                                                                               |
|  2. earlySingletonObjects   (Map<String, Object>)                             |
|     -> Early references exposed to resolve circular cycles before complete init|
|                                                                               |
|  3. singletonFactories      (Map<String, ObjectFactory<?>>)                   |
|     -> Holds ObjectFactory lambda that creates an early AOP proxy if needed.  |
+-------------------------------------------------------------------------------+
```

### Q2. How does Spring resolve Circular Dependencies between Setter/Field-injected beans?
**Answer:**
If `ServiceA` depends on `ServiceB`, and `ServiceB` depends on `ServiceA`:
1. `ServiceA` is instantiated via default constructor.
2. Before populating fields, `ServiceA` registers a factory lambda in the **3rd-level cache (`singletonFactories`)**.
3. `ServiceA` starts populating properties and discovers it needs `ServiceB`.
4. `ServiceB` is instantiated and attempts to inject `ServiceA`.
5. `ServiceB` queries the 3-level cache:
   - Not in 1st level (`singletonObjects`).
   - Not in 2nd level (`earlySingletonObjects`).
   - Found in 3rd level (`singletonFactories`)! The factory generates an early reference of `ServiceA`, puts it into `earlySingletonObjects`, and injects it into `ServiceB`.
6. `ServiceB` finishes initialization and enters `singletonObjects`.
7. `ServiceA` completes by injecting the finished `ServiceB`.

> [!WARNING]
> **Why Constructor Injection cannot resolve Circular Dependencies**:
> With Constructor Injection, the object cannot even be instantiated without its dependency already existing. Instantiation fails with `BeanCurrentlyInCreationException`.
> *(Starting with Spring Boot 2.6+, circular dependencies are disabled by default: `spring.main.allow-circular-references=false`).*

---

## 3. Spring AOP Architecture: Pointcuts, JoinPoints & Advices

```
+-----------------------------------------------------------------------------------+
|                        SPRING AOP ANATOMY & INTERCEPTION                          |
+-----------------------------------------------------------------------------------+
|  1. JoinPoint:     A point during program execution (in Spring AOP, method exec). |
|  2. Pointcut:      Expression matching target JoinPoints (e.g. execution, @annot).|
|  3. Advice:        Action taken by an aspect at a JoinPoint.                      |
|     - @Before:         Executes before target method call.                        |
|     - @AfterReturning: Executes after target method returns successfully.         |
|     - @AfterThrowing:  Executes if target method throws an exception.             |
|     - @After (finally):Executes regardless of method outcome.                     |
|     - @Around:         Surrounds method; controls execution via proceed().        |
+-----------------------------------------------------------------------------------+
```

### Q3. `JoinPoint` vs. `ProceedingJoinPoint`.
**Answer:**
- **`JoinPoint`**: Provides reflective access to target state (method signature via `getSignature()`, passed arguments via `getArgs()`, target object via `getTarget()`). Used in `@Before`, `@After`, `@AfterReturning`, and `@AfterThrowing`.
- **`ProceedingJoinPoint`**: Sub-interface of `JoinPoint` that adds the **`proceed()`** method to invoke the next interceptor or target method. Used **strictly in `@Around` advice**.

#### Pointcut Expression Designators:
- `execution(* com.example.service.*.*(..))`: Matches all methods in the package.
- `within(com.example.service..*)`: Matches all JoinPoints within given types.
- `@annotation(com.example.annotation.LogLatency)`: Matches methods annotated with `@LogLatency`.
- `@within(org.springframework.stereotype.Service)`: Matches any method in classes annotated with `@Service`.
- `args(java.lang.String, ..)`: Matches methods whose first argument is a `String`.

---

## 4. How to Create Custom Annotations with Spring AOP

```
 [ Method: @LogExecutionTime ] ---> AOP Proxy Intercepts ---> Around Advice (Timer starts)
                                                                    |
                                                                    v
                                                            jp.proceed() (Target Method)
                                                                    |
                                                                    v
                                                             Timer stops -> log.info("Latency: X ms")
```

### Q4. Build a Custom `@LogExecutionTime` Annotation & AOP Aspect from Scratch.
**Answer:**

#### Step 1: Define the Custom Annotation
```java
package com.example.annotation;

import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

// 1. Target: Where can this annotation be placed? (Method level)
@Target(ElementType.METHOD)
// 2. Retention: Retain in bytecode and make accessible at RUNTIME via reflection!
@Retention(RetentionPolicy.RUNTIME)
// 3. Documented: Include in Javadoc
@Documented
public @interface LogExecutionTime {
    String value() default "";
    boolean logArgs() default false;
}
```

#### Step 2: Implement the Aspect Interceptor
```java
package com.example.aspect;

import com.example.annotation.LogExecutionTime;
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Pointcut;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Component;

import java.util.Arrays;

@Aspect
@Component
public class LogExecutionTimeAspect {

    private static final Logger log = LoggerFactory.getLogger(LogExecutionTimeAspect.class);

    // Pointcut matching any method annotated with @LogExecutionTime
    @Pointcut("@annotation(logAnnotation)")
    public void loggedMethods(LogExecutionTime logAnnotation) {}

    // Around Advice surrounding the method invocation
    @Around(value = "loggedMethods(logAnnotation)", argNames = "joinPoint,logAnnotation")
    public Object measureExecutionTime(ProceedingJoinPoint joinPoint, LogExecutionTime logAnnotation) throws Throwable {
        long start = System.currentTimeMillis();
        String methodName = joinPoint.getSignature().toShortString();

        if (logAnnotation.logArgs()) {
            log.info("Entering {} with args: {}", methodName, Arrays.toString(joinPoint.getArgs()));
        }

        try {
            // Proceed with actual target method execution!
            Object result = joinPoint.proceed();
            long executionTime = System.currentTimeMillis() - start;
            log.info("Completed {} in {} ms", methodName, executionTime);
            return result;
        } catch (Throwable ex) {
            long executionTime = System.currentTimeMillis() - start;
            log.error("Failed {} after {} ms with exception: {}", methodName, executionTime, ex.getMessage());
            throw ex; // Re-throw to avoid swallowing business exceptions!
        }
    }
}
```

#### Step 3: Usage in Business Services
```java
@Service
public class OrderService {

    @LogExecutionTime(value = "Order Creation Pipeline", logArgs = true)
    public OrderResponse createOrder(CreateOrderRequest req) {
        // Business logic here...
        return new OrderResponse("ORD-123", "SUCCESS");
    }
}
```

---

## 5. Dynamic Proxies & The Self-Invocation Problem

### Q5. Compare JDK Dynamic Proxies vs. CGLIB Proxies.
**Answer:**

| Feature | JDK Dynamic Proxy (`java.lang.reflect.Proxy`) | CGLIB Proxy (`net.sf.cglib.proxy.Enhancer`) |
| :--- | :--- | :--- |
| **Target Requirement** | Class **MUST implement at least one interface** | Works on **classes without interfaces** (Generates bytecode subclass) |
| **Mechanism** | Reflection-based `InvocationHandler` | Runtime bytecode generation via ASM (`MethodInterceptor`) |
| **Limitations** | Only proxies interface methods | Cannot proxy `final` classes or `final` methods |
| **Spring Boot Default**| Prior to Boot 2.x | **Default in Spring Boot 2.x & 3.x** (`spring.aop.proxy-target-class=true`) |

---

### Q6. Deep Dive: The AOP Self-Invocation Problem and 3 Production Solutions.
**Answer:**
- **The Problem**: When a method calls another method within the **same class** (`this.methodB()`), the call is executed on the raw target object, bypassing the Spring AOP proxy wrapper completely!
- As a result, `@Transactional`, `@Async`, `@Cacheable`, and custom AOP annotations on `methodB()` will **silently fail to trigger**!

```java
@Service
public class OrderService {

    public void processOrder(Order order) {
        // ❌ SILENT FAILURE: Direct call on 'this' bypasses AOP Proxy!
        // The transaction on sendNotification() is NEVER started!
        this.sendNotification(order);
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void sendNotification(Order order) {
        // Notification logic
    }
}
```

#### The 3 Production Fixes:
1. **Fix 1: Self-Injection with `@Lazy` (Recommended)**:
   ```java
   @Service
   public class OrderService {
       @Autowired @Lazy private OrderService self;

       public void processOrder(Order order) {
           self.sendNotification(order); // Calls through proxy!
       }
   }
   ```
2. **Fix 2: Refactor into Separate Services**: Move `sendNotification()` into a dedicated `NotificationService`.
3. **Fix 3: `AopContext.currentProxy()`**:
   Requires `@EnableAspectJAutoProxy(exposeProxy = true)`.
   ```java
   ((OrderService) AopContext.currentProxy()).sendNotification(order);
   ```

---

> **Next Chapter**: [14 Spring Boot 3 Internals, Auto-Configuration & Embedded Tomcat](14_Spring_Boot_3_Internals_Actuator.md)
