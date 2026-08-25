# 14. Spring Boot 3 Internals, Starters, Embedded Servers, Actuator & Tools

> **Navigation**: [Master Index](README.md) | [Previous: Spring Core & AOP](13_Spring_Core_IoC_AOP_Internals.md) | [Next: Spring Web & REST APIs](15_Spring_Web_REST_APIs.md)

---

## 📌 Chapter Overview
This module explores **Spring Boot 3 internal mechanics**, Auto-Configuration, **Spring Boot Starters**, swapping **Embedded Tomcat for Undertow/Jetty**, deep configuration of **Spring Boot Actuator & Management APIs**, and **Lombok vs. DevTools** offerings and production gotchas.

---

## 1. Spring Boot Starters Ecosystem & Core Offerings

```
+-----------------------------------------------------------------------------------+
|                        SPRING BOOT STARTER DEPENDENCY MODEL                       |
+-----------------------------------------------------------------------------------+
|  [ Your Application pom.xml ]                                                     |
|         |                                                                         |
|         v (Single Dependency Declaration)                                         |
|  [ spring-boot-starter-web ]                                                      |
|         |                                                                         |
|         +---> spring-webmvc                (DispatcherServlet, REST Controllers)  |
|         +---> spring-boot-starter-tomcat   (Embedded Web Server)                  |
|         +---> spring-boot-starter-json     (Jackson ObjectMapper, Datatype modules)|
|         +---> spring-boot-starter          (Logging: Logback/SLF4J, YAML configs) |
+-----------------------------------------------------------------------------------+
```

### Q1. What are Spring Boot Starters and what do they provide?
**Answer:**
A **Starter** is a curated set of convenient dependency descriptors that bundles all required third-party libraries, transitive dependencies, and default configurations into a single dependency declaration.

| Starter Artifact | Key Transitive Libraries Included | Primary Use Case |
| :--- | :--- | :--- |
| **`spring-boot-starter-web`** | Spring MVC, Embedded Tomcat, Jackson, Hibernate Validator | Standard synchronous RESTful APIs & MVC Web apps |
| **`spring-boot-starter-webflux`**| Project Reactor, Spring WebFlux, Embedded Netty | Reactive, non-blocking asynchronous APIs & SSE |
| **`spring-boot-starter-data-jpa`**| Hibernate ORM, Spring Data JPA, HikariCP, Jakarta Persistence | Relational SQL persistence & ORM repository abstraction |
| **`spring-boot-starter-security`**| Spring Security Core & Web, Filter Chain infrastructure | Authentication, Authorization, RBAC, OAuth2/OIDC |
| **`spring-boot-starter-actuator`**| Micrometer, HealthIndicators, Metrics, Endpoint infrastructure | Production monitoring, health probes, metric scraping |
| **`spring-boot-starter-validation`**| Jakarta Bean Validation (JSR-380), Hibernate Validator | Declarative DTO validation (`@NotNull`, `@Size`, `@Pattern`)|

---

## 2. Swapping Embedded Tomcat for Undertow or Jetty

```
+-----------------------------------------------------------------------------------+
|                        EMBEDDED SERVER COMPARISON MATRIX                          |
+-----------------------------------------------------------------------------------+
|  Server    | Architecture          | Memory Footprint | Best Production Use Case  |
|  --------- | --------------------- | :--------------: | ------------------------- |
|  Tomcat    | Standard Java NIO     | Moderate         | Default enterprise REST   |
|  Undertow  | Async XNIO / Non-block| **Lowest RAM**   | High-concurrency REST/JSON|
|  Jetty     | Lightweight NIO       | Low              | High-throughput WebSockets|
|  Netty     | EventLoop Asynchronous| Dynamic ByteBuf  | Reactive WebFlux pipelines|
+-----------------------------------------------------------------------------------+
```

### Q2. How do you exclude Embedded Tomcat and switch to Undertow in `pom.xml`?
**Answer:**

```xml
<dependencies>
    <!-- 1. Include spring-boot-starter-web but EXCLUDE default Tomcat -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
        <exclusions>
            <exclusion>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-starter-tomcat</artifactId>
            </exclusion>
        </exclusions>
    </dependency>

    <!-- 2. Add High-Performance Undertow Server Dependency -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-undertow</artifactId>
    </dependency>
</dependencies>
```

#### Undertow Production Tuning in `application.yml`:
```yaml
server:
  port: 8080
  undertow:
    threads:
      io: 4               # Dedicated I/O worker threads (matches CPU cores)
      worker: 64          # Business worker threads handling blocking calls
    buffer-size: 16384    # 16 KB direct buffer memory allocation
    direct-buffers: true  # Allocates off-heap buffers for zero-copy OS performance
```

---

## 3. Auto-Configuration Mechanics & Conditionals

```
+-----------------------------------------------------------------------------------+
|                        SPRING BOOT AUTO-CONFIGURATION ENGINE                      |
+-----------------------------------------------------------------------------------+
|  1. @SpringBootApplication                                                        |
|     -> @EnableAutoConfiguration                                                   |
|        -> AutoConfigurationImportSelector                                         |
|           -> Reads: META-INF/spring/org.springframework.boot.autoconfigure.       |
|                     AutoConfiguration.imports                                     |
|                                                                                   |
|  2. Condition Evaluation:                                                         |
|     - @ConditionalOnClass(DataSource.class)       -> Is class on classpath?       |
|     - @ConditionalOnMissingBean(DataSource.class)  -> Has user defined bean?      |
|     - @ConditionalOnProperty("spring.datasource") -> Is property enabled?        |
|                                                                                   |
|  3. Register Auto-Configured Beans in ApplicationContext                          |
+-----------------------------------------------------------------------------------+
```

---

## 4. Spring Boot Actuator & Management APIs Deep Dive

### Q3. What is Spring Boot Actuator, how do you expose endpoints, and how do you secure it?
**Answer:**
**Spring Boot Actuator** provides production-ready operational management features: health checks, metric collection, environment inspection, thread dumps, and runtime log-level adjustments.

#### 1. Exposing Endpoints (`application.yml`):
By default, only `/health` is exposed over HTTP for security reasons. To expose others:

```yaml
management:
  endpoints:
    web:
      base-path: /manage                   # Custom management API base path
      exposure:
        include: health,info,metrics,prometheus,env,loggers,threaddump
  endpoint:
    health:
      show-details: when_authorized       # Show DB/disk/kafka component health to authenticated users
      probes:
        enabled: true                     # Enables K8s liveness & readiness probes
```

#### 2. Securing Actuator Endpoints with Spring Security:
```java
@Configuration
public class ActuatorSecurityConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        return http
            .authorizeHttpRequests(auth -> auth
                // Public health probe endpoint for Kubernetes / Load Balancer
                .requestMatchers("/manage/health", "/manage/health/**").permitAll()
                // Restrict sensitive metrics, environment vars, and loggers to ADMIN role
                .requestMatchers("/manage/**").hasRole("OPS_ADMIN")
                .anyRequest().authenticated()
            )
            .httpBasic(Customizer.withDefaults())
            .build();
    }
}
```

---

### Q4. How do you implement a Custom `HealthIndicator`?
**Answer:**

```java
@Component("paymentGatewayHealth")
public class PaymentGatewayHealthIndicator implements HealthIndicator {

    @Autowired private PaymentGatewayClient gatewayClient;

    @Override
    public Health health() {
        try {
            long pingLatency = gatewayClient.ping();
            if (pingLatency > 3000) {
                return Health.down()
                    .withDetail("error", "High Latency Warning")
                    .withDetail("latencyMs", pingLatency)
                    .build();
            }
            return Health.up()
                .withDetail("gateway", "Stripe Active")
                .withDetail("latencyMs", pingLatency)
                .build();
        } catch (Exception ex) {
            return Health.down(ex)
                .withDetail("error", "Payment Gateway Unreachable")
                .build();
        }
    }
}
```

---

### Q5. How do you change Log Levels dynamically at runtime via Actuator without restarting?
**Answer:**
You can update any package's log level dynamically using a `POST` request to `/actuator/loggers/{name}`:

```bash
# Temporarily enable DEBUG logging on com.example.service in production
curl -X POST http://localhost:8080/manage/loggers/com.example.service \
     -H "Content-Type: application/json" \
     -d '{"configuredLevel": "DEBUG"}'
```

---

## 5. Lombok vs. Spring Boot DevTools: Offerings & Production Gotchas

```
+-----------------------------------------------------------------------------------+
|                        LOMBOK VS SPRING BOOT DEVTOOLS                             |
+-----------------------------------------------------------------------------------+
|  Tool      | Mechanism                    | Purpose                               |
|  --------- | ---------------------------- | ------------------------------------- |
|  Lombok    | Compile-time Annotation ASM  | Eliminates boilerplate Java code      |
|  DevTools  | Dual-ClassLoader Live Reload | Fast local iteration during coding    |
+-----------------------------------------------------------------------------------+
```

### Q6. Detail Lombok Offerings & The Critical JPA `@Data` Anti-Pattern.
**Answer:**

#### 1. Core Lombok Offerings:
- **`@Getter` / `@Setter`**: Generates getters and setters.
- **`@ToString`**: Generates `toString()` (use `@ToString.Exclude` on sensitive passwords or circular relationships).
- **`@EqualsAndHashCode`**: Generates `equals()` and `hashCode()`.
- **`@RequiredArgsConstructor`**: Generates constructor for `final` fields (ideal for Spring constructor dependency injection).
- **`@Builder`**: Implements the GoF Builder pattern.
- **`@Slf4j`**: Injects `private static final Logger log = LoggerFactory.getLogger(...)`.

> [!WARNING]
> **Why `@Data` on JPA Entities is a Severe Production Anti-Pattern**:
> 1. **Breaks `hashCode()` Contract**: `@Data` generates `equals` and `hashCode` using all mutable fields. When a new entity is saved, its `id` changes from `null` to a database ID, altering its hash code and corrupting `HashSet` or `HashMap` lookups!
> 2. **Causes `StackOverflowError` & N+1 Queries**: `@Data` includes all fields in `toString()`. On bidirectional `@OneToMany` / `@ManyToOne` relationships, calling `toString()` triggers infinite circular recursion and forces eager initialization of all lazy collections!
> - **Production Rule**: On JPA entities, use only `@Getter`, `@Setter`, and `@ToString(onlyExplicitlyIncluded = true)`. Never use `@Data` or `@EqualsAndHashCode`!

---

### Q7. What does Spring Boot DevTools provide?
**Answer:**
- **Dual ClassLoader Fast Restart**: Divides classes into *Base ClassLoader* (third-party dependencies) and *Restart ClassLoader* (your application code). When a file changes, only the tiny restart classloader reloads ($~500\text{ms}$ restart).
- **LiveReload Server**: Auto-refreshes browser when UI templates/CSS change.
- **Automatic Cache Disabling**: Disables Thymeleaf/Freemarker template caches during development.
- **Safety in Production**: Declared with `<optional>true</optional>`, meaning it is **automatically excluded** when packaged into a production JAR (`mvn package`).

---

## 6. Types of Spring Boot Project Setups, Packaging & Architectural Archetypes

```
+-----------------------------------------------------------------------------------+
|                        SPRING BOOT PROJECT SETUP TAXONOMY                         |
+-----------------------------------------------------------------------------------+
|  1. Initialization Modes  -> Spring Initializr (Web/CLI/IDE), Maven Archetype    |
|  2. Packaging Formats     -> Executable JAR, Legacy WAR, GraalVM AOT Native, OCI  |
|  3. Layout Archetypes     -> Package-by-Layer, Package-by-Feature, Hexagonal,     |
|                              Multi-Module Clean Architecture, Spring Modulith     |
+-----------------------------------------------------------------------------------+
```

### Q8. Detail the 4 Deployment Packaging Types in Spring Boot.
**Answer:**

| Packaging Type | Target Runtime | Build Mechanism | Key Characteristics |
| :--- | :--- | :--- | :--- |
| **Executable Standalone JAR (Default)** | JVM on Docker / Linux VM | `spring-boot-maven-plugin:repackage` | Bundles embedded server (Tomcat/Undertow) and classes into an executable archive executed via `JarLauncher`. |
| **Traditional Deployable WAR** | External Application Server (Tomcat 10+, WildFly) | `<packaging>war</packaging>` + `SpringBootServletInitializer` | Extends `SpringBootServletInitializer` to bind the Spring context to an external servlet container. |
| **GraalVM Native Image (AOT)** | Serverless (AWS Lambda, Google Cloud Run) | `native-maven-plugin` (Ahead-Of-Time compilation) | Compiles Java into native OS machine binary. Instant cold startup ($<50\text{ms}$) and tiny memory footprint ($<50\text{MB}$). |
| **Cloud-Native OCI Image (Buildpacks)**| Kubernetes / Container Registries | `mvn spring-boot:build-image` | Creates an optimal, layered Docker/OCI container directly without writing a custom `Dockerfile`! |

#### Converting Spring Boot to a Deployable WAR:
```java
@SpringBootApplication
public class Application extends SpringBootServletInitializer {
    @Override
    protected SpringApplicationBuilder configure(SpringApplicationBuilder builder) {
        return builder.sources(Application.class);
    }
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

---

### Q9. Compare the 4 Common Project Structure Layout Archetypes.
**Answer:**

#### Archetype 1: Package-by-Layer (Classic 3-Tier - Good for small CRUD apps)
```
 com.example.app/
 ├── controller/    (OrderController, UserController)
 ├── service/       (OrderService, UserService)
 ├── repository/    (OrderRepository, UserRepository)
 ├── entity/        (Order, User)
 └── dto/           (OrderRequest, OrderResponse)
```
- *Limitation*: As the project grows, layers become bloated; changes to a single feature require touching multiple distant directories.

#### Archetype 2: Package-by-Feature (Domain-Oriented Monolith - Good for medium projects)
```
 com.example.app/
 ├── order/         (OrderController, OrderService, OrderRepository, OrderEntity)
 ├── payment/       (PaymentController, PaymentService, PaymentRepository)
 └── user/          (UserController, UserService, UserRepository)
```
- *Advantage*: High cohesion within a domain; entire feature can be moved or refactored independently.

#### Archetype 3: Hexagonal Architecture / Ports & Adapters (Clean Architecture - Enterprise)
```
 com.example.app/
 ├── domain/            <-- Pure Java POJOs & business rules (ZERO framework imports!)
 │   └── model/ (Order, OrderId)
 ├── application/       <-- Use cases & Ports (Interfaces)
 │   ├── port/in/   (CreateOrderUseCase)
 │   └── port/out/  (SaveOrderPort, PaymentGatewayPort)
 └── infrastructure/    <-- Adapters implementing ports (Spring Boot, JPA, REST)
     ├── adapter/in/web/       (OrderRestController implements CreateOrderUseCase)
     └── adapter/out/database/ (JpaOrderRepositoryAdapter implements SaveOrderPort)
```
- *Advantage*: Core business domain is 100% decoupled from Spring, JPA, and web frameworks, allowing effortless testing and database swapping.

#### Archetype 4: Multi-Module Enterprise Maven Project
- Separates domain, persistence, business logic, and API gateways into physical, independent build modules with strict dependency boundaries (detailed in [Module 34](34_Build_Tools_CI_CD_DevOps.md)).

---

> **Next Chapter**: [15 Web, REST & Reactive APIs (Spring WebFlux)](15_Spring_Web_REST_APIs.md)
