# 16. Spring Security 6, JWT & OAuth2 Architecture

> **Navigation**: [Master Index](README.md) | [Previous: Spring Web & REST](15_Spring_Web_REST_APIs.md) | [Next: Spring Batch Processing](17_Spring_Batch_ETL_Processing.md)

---

## 📌 Chapter Overview
This module explores modern **Spring Security 6 (Spring Boot 3)** architecture, lambda DSL filter chains, stateless **JWT authentication**, **OAuth 2.0 / OIDC PKCE authorization flows**, method-level SpEL security, and **multi-tenant data isolation**.

---

## 1. Spring Security 6 Filter Chain Architecture

```
                                [ Incoming HTTP Request ]
                                            |
                                            v
                                 [ DelegatingFilterProxy ]
                                            |
                                            v
                                  [ FilterChainProxy ]
                                            |
                         +------------------+------------------+
                         |       SecurityFilterChain           |
                         |  1. CorsFilter                      |
                         |  2. CsrfFilter (Disabled for JWT)   |
                         |  3. JwtAuthenticationFilter         |
                         |  4. ExceptionTranslationFilter      |
                         |  5. AuthorizationFilter             |
                         +-------------------------------------+
                                            |
                                            v
                                   [ Controller Endpoint ]
```

### Q1. How do you configure a modern Stateless JWT `SecurityFilterChain` in Spring Boot 3?
**Answer:**

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity(prePostEnabled = true) // Enables @PreAuthorize
public class SecurityConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(
            HttpSecurity http, 
            JwtAuthenticationFilter jwtAuthFilter) throws Exception {
        
        return http
            // 1. Disable CSRF for stateless REST APIs using JWT
            .csrf(AbstractHttpConfigurer::disable)
            // 2. Enable CORS
            .cors(Customizer.withDefaults())
            // 3. Set session management to STATELESS (No HttpSession created on server)
            .sessionManagement(session -> session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            // 4. URL Authorization rules
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/v1/auth/**", "/actuator/health", "/v3/api-docs/**").permitAll()
                .requestMatchers("/api/v1/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            )
            // 5. Add custom JWT Filter before standard UsernamePasswordAuthenticationFilter
            .addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationFilter.class)
            .build();
    }
}
```

---

## 2. OAuth 2.0 & OpenID Connect (OIDC) with Keycloak

```
+-----------------------------------------------------------------------------------+
|                     OAUTH 2.0 AUTHORIZATION CODE GRANT WITH PKCE                  |
+-----------------------------------------------------------------------------------+
|  1. User clicks login in SPA / Mobile App (Generates code_verifier & code_challenge)
|  2. Browser redirects to Keycloak Auth Server with code_challenge                 |
|  3. User logs in -> Keycloak returns Authorization Code to Browser                |
|  4. App exchanges Auth Code + code_verifier for Access Token (JWT) & ID Token    |
|  5. App sends Access Token in 'Authorization: Bearer <JWT>' to Spring Boot API    |
|  6. Spring Boot Resource Server validates JWT signature via Keycloak JWKS endpoint|
+-----------------------------------------------------------------------------------+
```

### Q2. How do you extract Keycloak realm & client roles into Spring `GrantedAuthority`?
**Answer:**
By default, Spring Security maps scopes (`SCOPE_read`), but Keycloak embeds user roles inside a nested JSON path: `realm_access.roles`.

```java
public class KeycloakRoleConverter implements Converter<Jwt, Collection<GrantedAuthority>> {
    
    @Override
    @SuppressWarnings("unchecked")
    public Collection<GrantedAuthority> convert(Jwt jwt) {
        Map<String, Object> realmAccess = (Map<String, Object>) jwt.getClaims().get("realm_access");
        if (realmAccess == null || realmAccess.isEmpty()) {
            return List.of();
        }

        Collection<String> roles = (Collection<String>) realmAccess.get("roles");
        return roles.stream()
            .map(roleName -> new SimpleGrantedAuthority("ROLE_" + roleName.toUpperCase()))
            .collect(Collectors.toList());
    }
}

// In SecurityConfig for OAuth2 Resource Server:
@Bean
public JwtAuthenticationConverter jwtAuthenticationConverter() {
    JwtAuthenticationConverter converter = new JwtAuthenticationConverter();
    converter.setJwtGrantedAuthoritiesConverter(new KeycloakRoleConverter());
    return converter;
}
```

---

## 3. Fine-Grained Method Security with SpEL

### Q3. How do you implement object-level ownership checks with `@PreAuthorize`?
**Answer:**

```java
@RestController
@RequestMapping("/api/v1/documents")
public class DocumentController {

    // Allows access if user has ADMIN role OR if the document belongs to the authenticated user!
    @GetMapping("/{id}")
    @PreAuthorize("hasRole('ADMIN') or @documentSecurityService.isOwner(#id, authentication.name)")
    public DocumentDto getDocument(@PathVariable("id") Long id) {
        return documentService.findById(id);
    }
}

@Service("documentSecurityService")
public class DocumentSecurityService {
    @Autowired private DocumentRepository docRepo;

    public boolean isOwner(Long documentId, String username) {
        return docRepo.findById(documentId)
            .map(doc -> doc.getOwnerUsername().equals(username))
            .orElse(false);
    }
}
```

---

## 4. Multi-Tenant Architecture & Data Isolation

### Q4. Compare Multi-Tenancy Data Isolation Models.
**Answer:**

```
1. Database-Per-Tenant:
   [ Tenant A API ] ----> [ DB Instance A ]
   [ Tenant B API ] ----> [ DB Instance B ]  (Highest isolation, high cloud cost)

2. Schema-Per-Tenant (Recommended Enterprise):
   [ Spring Boot API ] --> [ Single DB ] -> Schema 'tenant_a'
                                         -> Schema 'tenant_b'

3. Shared Database, Shared Schema (Row-Level Discriminator):
   [ Spring Boot API ] --> [ Single DB ] -> Table: 'orders' [ tenant_id | order_id | ... ]
   (Enforced via Hibernate @TenantId or Postgres Row Level Security RLS)
```

---

> **Next Chapter**: [17 Spring Batch & High-Throughput ETL Processing](17_Spring_Batch_ETL_Processing.md)

