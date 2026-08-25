# Security, Testing & Observability Deep Dive

> **Navigation**: [Master Index](README.md) | [Previous: Persistence & Messaging](06_Persistence_Transactions_Messaging.md) | [Next: Distributed Systems & Cloud](08_Distributed_Systems_Cloud.md)

---

## 1. Spring Security 6 & JWT Architecture

```
+-----------------------------------------------------------------------------------+
|                        SPRING SECURITY FILTER CHAIN                               |
+-----------------------------------------------------------------------------------+
|  1. Incoming HTTP Request (Authorization: Bearer <JWT>)                           |
|         |                                                                         |
|         v                                                                         |
|  2. [ SecurityFilterChain ]                                                       |
|         |                                                                         |
|         +---> [ CorsFilter ] (Validates allowed origins, headers, methods)        |
|         +---> [ CsrfFilter ] (Validates CSRF tokens if enabled)                   |
|         +---> [ CustomJwtAuthenticationFilter ]                                   |
|         |        |                                                                |
|         |        +---> Validates signature via JwtDecoder / Public Key            |
|         |        +---> Extracts subject (username) & claims (roles/authorities)   |
|         |        +---> Sets SecurityContextHolder.getContext().setAuthentication()|
|         |                                                                         |
|         +---> [ AuthorizationFilter ] (@PreAuthorize, URL-based role checks)      |
|         |                                                                         |
|         v                                                                         |
|  3. [ DispatcherServlet -> @RestController Method ]                               |
+-----------------------------------------------------------------------------------+
```

---

### Q1. How do you configure a modern Stateless JWT `SecurityFilterChain` in Spring Boot 3 / Spring Security 6?
**Answer:**

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity(prePostEnabled = true)
public class SecurityConfig {

    private final JwtAuthenticationFilter jwtAuthFilter;
    private final AuthenticationProvider authenticationProvider;

    public SecurityConfig(JwtAuthenticationFilter jwtAuthFilter, AuthenticationProvider authProvider) {
        this.jwtAuthFilter = jwtAuthFilter;
        this.authenticationProvider = authProvider;
    }

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        return http
            // 1. Disable CSRF (Stateless JWT REST APIs do not use browser session cookies)
            .csrf(AbstractHttpConfigurer::disable)
            // 2. Configure CORS
            .cors(cors -> cors.configurationSource(corsConfigurationSource()))
            // 3. Stateless Session Management
            .sessionManagement(session -> session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            // 4. URL Authorization Rules
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/v1/auth/**", "/actuator/health/**", "/swagger-ui/**", "/v3/api-docs/**").permitAll()
                .requestMatchers("/api/v1/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            )
            .authenticationProvider(authenticationProvider)
            // 5. Add Custom JWT Filter before UsernamePasswordAuthenticationFilter
            .addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationFilter.class)
            .build();
    }

    @Bean
    public CorsConfigurationSource corsConfigurationSource() {
        CorsConfiguration config = new CorsConfiguration();
        config.setAllowedOrigins(List.of("https://app.myapp.com"));
        config.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE", "PATCH"));
        config.setAllowedHeaders(List.of("Authorization", "Content-Type", "X-Trace-Id"));
        config.setAllowCredentials(true);
        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/**", config);
        return source;
    }
}
```

---

## 2. Production Testing Pyramid & Testcontainers

```
                   /\
                  /  \     End-to-End / Contract Tests (5-10%)
                 /----\
                / Inte \   Integration Tests: @SpringBootTest, Testcontainers (20-30%)
               /--------\
              /  Slice   \ Sliced Tests: @WebMvcTest, @DataJpaTest (30-40%)
             /------------\
            /  Unit Tests  \ Pure Unit Tests: JUnit 5, Mockito, AssertJ (50-60%)
           /----------------\
```

---

### Q2. How do you write real database integration tests using Testcontainers?
**Answer:**
H2 in-memory databases often mask database-specific SQL dialect bugs, sequence mismatches, and JSONB queries. **Testcontainers** spins up real, ephemeral Docker containers for PostgreSQL/MySQL during test runs.

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@Testcontainers
class OrderServiceIntegrationTest {

    // Spins up real PostgreSQL 16 container for the test suite
    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine")
        .withDatabaseName("testdb")
        .withUsername("testuser")
        .withPassword("testpass");

    // Dynamically injects the container's random assigned port into Spring Data properties
    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }

    @Autowired
    private OrderService orderService;

    @Autowired
    private OrderRepository orderRepository;

    @Test
    void shouldCreateAndPersistOrderSuccessfully() {
        CreateOrderRequest request = new CreateOrderRequest("cust-123", new BigDecimal("199.99"));
        OrderResponse response = orderService.createOrder(request);

        assertThat(response.orderId()).isNotNull();
        assertThat(orderRepository.findById(response.orderId())).isPresent();
    }
}
```

---

## 3. Observability, Metrics & Distributed Tracing

### Q3. How do you implement Distributed Tracing with OpenTelemetry & MDC Logging?
**Answer:**

1. **Mapped Diagnostic Context (MDC)**:
   MDC attaches contextual key-value pairs (e.g., `traceId`, `spanId`, `userId`) to the current thread so that all log lines automatically output the trace ID.
2. **Correlation Across Microservices (W3C TraceContext)**:
   When Service A calls Service B via HTTP or Kafka, it forwards the `traceparent` HTTP header (`00-<traceId>-<spanId>-<sampled>`). Service B continues the same `traceId`.

```xml
<!-- logback-spring.xml JSON Structured Logging -->
<configuration>
    <appender name="JSON" class="ch.qos.logback.core.ConsoleAppender">
        <encoder class="net.logstash.logback.encoder.LogstashEncoder">
            <!-- Automatically includes MDC fields: traceId, spanId, userId -->
            <includeMdcKeyName>traceId</includeMdcKeyName>
            <includeMdcKeyName>spanId</includeMdcKeyName>
            <includeMdcKeyName>userId</includeMdcKeyName>
        </encoder>
    </appender>
    <root level="INFO">
        <appender-ref ref="JSON" />
    </root>
</configuration>
```

```java
// Spring Filter ensuring traceId is always present in MDC
@Component
public class TraceIdFilter extends OncePerRequestFilter {
    private static final String TRACE_ID_HEADER = "X-Trace-Id";
    private static final String MDC_TRACE_KEY = "traceId";

    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain)
            throws ServletException, IOException {
        String traceId = request.getHeader(TRACE_ID_HEADER);
        if (traceId == null || traceId.isBlank()) {
            traceId = UUID.randomUUID().toString();
        }
        MDC.put(MDC_TRACE_KEY, traceId);
        response.setHeader(TRACE_ID_HEADER, traceId);
        try {
            filterChain.doFilter(request, response);
        } finally {
            MDC.remove(MDC_TRACE_KEY); // Mandatory cleanup to prevent ThreadLocal leaks!
        }
    }
}
```

---

### Q4. Prometheus & Grafana Metric Dashboard Essentials.
**Answer:**

| Golden Signal | Metric Name (Micrometer / Prometheus) | What It Alerts On |
| :--- | :--- | :--- |
| **Latency** | `http_server_requests_seconds_max` / `_p99` | Service degradation & slow queries |
| **Traffic / Rate** | `rate(http_server_requests_seconds_count[1m])` | Traffic spikes, DDoS, volume |
| **Errors** | `rate(http_server_requests_seconds_count{status=~"5.."}[1m])` | 5xx server exceptions & outages |
| **Saturation** | `hikaricp_connections_pending` / `jvm_memory_used_bytes` | Connection pool starvation / OOM |

---

> **Next Chapter**: [08 Distributed Systems & Cloud](08_Distributed_Systems_Cloud.md)
