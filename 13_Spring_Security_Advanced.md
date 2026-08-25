# Advanced Spring Security, OAuth2 & Multi-Tenancy

> **Navigation**: [Master Index](README.md) | [Previous: SOLID & Patterns](12_SOLID_Design_Principles_Patterns.md) | [Next: Advanced Spring Cloud](14_Spring_Cloud_Advanced.md)

---

## 1. OAuth 2.0 & OpenID Connect (OIDC) Architecture

```
+-----------------------------------------------------------------------------------+
|                        OAUTH 2.0 AUTHORIZATION CODE WITH PKCE                     |
+-----------------------------------------------------------------------------------+
|  [ Single Page App / Mobile ] ---> 1. /authorize (code_challenge) ---> [ IdP / Keycloak ]
|             |                                                                 |
|             +<------------------- 2. Auth Code Redirect <---------------------+
|             |
|             +-------------------> 3. /token (code + code_verifier) ---------> [ IdP / Keycloak ]
|             |                                                                 |
|             +<------------------- 4. Access Token (JWT) + ID Token <----------+
|             |
|             +--- 5. HTTP GET /api/orders (Authorization: Bearer <JWT>) ------> [ Spring Boot Resource Server ]
|                                                                                       |
|                                                                                       +---> Validates signature via JWKS Public Key
+-----------------------------------------------------------------------------------+
```

---

### Q1. How do you configure Spring Boot 3 as an OAuth2 Resource Server with Custom Keycloak JWT Role Mapping?
**Answer:**

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity
public class OAuth2ResourceServerConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        return http
            .csrf(AbstractHttpConfigurer::disable)
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/public/**").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            )
            .oauth2ResourceServer(oauth2 -> oauth2
                .jwt(jwt -> jwt.jwtAuthenticationConverter(customJwtConverter()))
            )
            .build();
    }

    // Maps Keycloak 'realm_access.roles' JSON path to Spring Security 'ROLE_<NAME>' authorities
    @Bean
    public Converter<Jwt, AbstractAuthenticationToken> customJwtConverter() {
        return jwt -> {
            Map<String, Object> realmAccess = jwt.getClaim("realm_access");
            Collection<GrantedAuthority> authorities = List.of();

            if (realmAccess != null && realmAccess.containsKey("roles")) {
                @SuppressWarnings("unchecked")
                List<String> roles = (List<String>) realmAccess.get("roles");
                authorities = roles.stream()
                    .map(role -> new SimpleGrantedAuthority("ROLE_" + role.toUpperCase()))
                    .collect(Collectors.toList());
            }

            return new JwtAuthenticationToken(jwt, authorities, jwt.getClaimAsString("preferred_username"));
        };
    }
}
```

---

## 2. Advanced Method-Level Security & Custom SpEL Evaluators

### Q2. How do you enforce fine-grained domain object permission checks using `@PreAuthorize`?
**Answer:**

```java
// Custom Spring Security Permission Evaluator Bean
@Component("orderSecurity")
public class OrderSecurityEvaluator {
    private final OrderRepository orderRepository;

    public OrderSecurityEvaluator(OrderRepository orderRepository) {
        this.orderRepository = orderRepository;
    }

    public boolean isOrderOwner(Long orderId, Authentication authentication) {
        String currentUsername = authentication.getName();
        return orderRepository.findById(orderId)
            .map(order -> order.getCustomerUsername().equals(currentUsername))
            .orElse(false);
    }
}

// Controller using Custom SpEL Expression
@RestController
@RequestMapping("/api/orders")
public class OrderController {

    // Allows access if user has 'ROLE_ADMIN' OR is the owner of this specific orderId
    @PreAuthorize("hasRole('ADMIN') or @orderSecurity.isOrderOwner(#orderId, authentication)")
    @GetMapping("/{orderId}")
    public OrderResponse getOrder(@PathVariable Long orderId) {
        return orderService.fetchOrder(orderId);
    }
}
```

---

## 3. Multi-Tenant Architecture & Data Isolation

### Q3. How do you implement Multi-Tenancy (Database-per-Tenant) in Spring Boot?
**Answer:**
Use Spring's **`AbstractRoutingDataSource`** combined with a `TenantContext` stored in a `ThreadLocal` during authentication.

```java
// 1. ThreadLocal Tenant Context
public class TenantContext {
    private static final ThreadLocal<String> CURRENT_TENANT = new ThreadLocal<>();

    public static void setTenantId(String tenantId) { CURRENT_TENANT.set(tenantId); }
    public static String getTenantId() { return CURRENT_TENANT.get(); }
    public static void clear() { CURRENT_TENANT.remove(); }
}

// 2. Dynamic Routing DataSource
public class DynamicTenantRoutingDataSource extends AbstractRoutingDataSource {
    @Override
    protected Object determineCurrentLookupKey() {
        return TenantContext.getTenantId(); // Returns "tenant_a", "tenant_b", etc.
    }
}
```

---

> **Next Chapter**: [14 Advanced Spring Cloud](14_Spring_Cloud_Advanced.md)
