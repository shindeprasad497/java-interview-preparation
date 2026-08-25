# 35. Engineering Leadership, ADRs & Monolith Modernization

> **Navigation**: [Master Index](README.md) | [Previous: Build Tools & DevOps](34_Build_Tools_CI_CD_DevOps.md) | [Next: Spring AI Guide](36_Spring_AI_Generative_AI_Guide.md)

---

## 📌 Chapter Overview
This module covers critical **Tech Lead & Senior Engineering Competencies**: authoring **Architectural Decision Records (ADRs)**, managing **Technical Debt & Monolith Modernization**, running **Blameless Root Cause Analysis (RCA)**, and **Engineering Mentorship**.

---

## 1. Architectural Decision Records (ADR)

```
+-----------------------------------------------------------------------------------+
|                     ARCHITECTURAL DECISION RECORD (ADR) STRUCTURE                 |
+-----------------------------------------------------------------------------------+
|  ADR-004: Adoption of Apache Kafka over RabbitMQ for Financial Event Streaming    |
|                                                                                   |
|  1. STATUS:        Accepted (2026-03-15)                                          |
|  2. CONTEXT:       Our payment microservice requires real-time transaction replay |
|                    and 7-day immutable audit retention with >100k events/sec.     |
|  3. DECISION:      We will adopt Apache Kafka (KRaft mode) for payment events.    |
|  4. CONSEQUENCES:                                                                 |
|     - Positive: High write throughput, guaranteed partition FIFO, easy replay.   |
|     - Negative: Increased infrastructure complexity compared to simple brokers.  |
|  5. COMPLIANCE:    All events must be serialized via Avro / Schema Registry.      |
+-----------------------------------------------------------------------------------+
```

### Q1. Why are ADRs essential in senior software engineering?
**Answer:**
ADRs capture the **context, rationale, trade-offs, and consequences** of architectural choices at the time they are made. They prevent recurring circular debates, onboard new engineers rapidly, and document why specific alternatives were rejected.

---

## 2. Legacy Modernization: Java 8 $\rightarrow$ Java 21 & `javax` $\rightarrow$ `jakarta`

### Q2. What are the key breaking changes when migrating legacy Spring Boot apps to Spring Boot 3 & Java 21?
**Answer:**
1. **Java Baseline**: Spring Boot 3 requires **Java 17 LTS as minimum** (Java 21 recommended).
2. **`javax.*` to `jakarta.*` Namespace Migration**:
   - Jakarta EE 9/10 renamed package imports (`javax.persistence.*` $\rightarrow$ `jakarta.persistence.*`, `javax.servlet.*` $\rightarrow$ `jakarta.servlet.*`, `javax.validation.*` $\rightarrow$ `jakarta.validation.*`).
3. **Strong Encapsulation of JDK Internals (JEP 403)**:
   - Reflection into internal JDK packages (`sun.misc.Unsafe`) is restricted by default. Dependencies must be upgraded to avoid `IllegalAccessException`.

---

## 3. Blameless Post-Mortems & The 5 Whys Methodology

### Q3. How do you conduct a Blameless Root Cause Analysis (RCA)?
**Answer:**
A **Blameless Post-Mortem** assumes that human engineers act in good faith with the tools they have. Failures are viewed as **systemic and tooling deficiencies**, not personal incompetence.

#### The 5 Whys Example (Database Outage):
1. *Why did the API crash?* $\rightarrow$ The HikariCP database connection pool was exhausted.
2. *Why was the connection pool exhausted?* $\rightarrow$ Worker threads held connections for $>1.5\text{s}$ each.
3. *Why were connections held for 1.5s?* $\rightarrow$ An unindexed full-table scan query was executed on every login.
4. *Why was an unindexed query executed?* $\rightarrow$ A new feature was merged without a query execution plan review.
5. *Why was it merged without review?* $\rightarrow$ CI/CD lacked automated SQL migration linting tools.
- **Corrective and Preventive Action (CAPA)**: Add automated Flyway SQL linting to the GitHub Actions CI pipeline to verify indexes before merging.

---

> **Next Chapter**: [36 Spring AI & Generative AI Interview Guide](36_Spring_AI_Generative_AI_Guide.md)

