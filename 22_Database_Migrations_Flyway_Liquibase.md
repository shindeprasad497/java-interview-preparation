# 22. Database Schema Migrations & Zero-Downtime Evolution

> **Navigation**: [Master Index](README.md) | [Previous: Transactions & Locking](21_Transactions_Locking_Isolation.md) | [Next: Redis Caching & NoSQL](23_Redis_Caching_Patterns_NoSQL.md)

---

## 📌 Chapter Overview
This module explores **Database Version Control (Flyway vs. Liquibase)** in CI/CD automation and the **Expand-Contract (Parallel Run) Pattern** for performing non-breaking, zero-downtime schema migrations.

---

## 1. Flyway vs. Liquibase Comparison

| Feature | Flyway | Liquibase |
| :--- | :--- | :--- |
| **Migration Format** | Native SQL files (`.sql`) or Java code | XML, YAML, JSON, or SQL changeSets |
| **Learning Curve** | **Low / Intuitive** (Standard SQL syntax) | Moderate (Requires learning changeSet tags) |
| **Rollback Support** | Manual (or Flyway Teams Edition) | Native automated `<rollback>` tags |
| **Spring Boot Setup**| `org.flywaydb:flyway-core` | `org.liquibase:liquibase-core` |

---

## 2. Flyway Migration Versioning Conventions

```
 src/main/resources/db/migration/
 ├── V1__create_users_table.sql      (Versioned: Executes once in order)
 ├── V2__add_index_on_email.sql      (Versioned)
 ├── U2__undo_add_index_on_email.sql (Undo script)
 └── R__create_order_summary_view.sql(Repeatable: Re-runs whenever checksum changes)
```

- **`V<Version>__<Description>.sql`**: Versioned migrations applied sequentially. Tracked in table `flyway_schema_history`.
- **`R__<Description>.sql`**: Repeatable migrations (views, stored procedures) re-executed whenever their content changes.

---

## 3. Zero-Downtime Migrations: The Expand-Contract Pattern

### Q1. How do you rename a column or add a `NOT NULL` constraint without downtime?
**Answer:**
Directly executing `ALTER TABLE orders RENAME COLUMN email TO customer_email;` immediately breaks running application instances that still expect the old column name.

#### The 5-Step Expand-Contract Pattern:

```
  STEP 1: EXPAND                       STEP 2: PARALLEL RUN
  Add new nullable column               App writes to BOTH old & new cols;
  [ email ]  [ customer_email (NULL) ]   reads from old col.
                 |                                  |
                 v                                  v
  STEP 3: BACKFILL                     STEP 4: SWITCH READ
  Batch script copies historical data   App switches to read customer_email
  from old to new column.                           |
                                                    v
                                       STEP 5: CONTRACT
                                       Add NOT NULL to customer_email,
                                       drop old 'email' column safely!
```

1. **Step 1 (Expand - DB Migration)**: Add the new column `customer_email` as `NULLABLE`.
2. **Step 2 (Deploy Code v1)**: Application writes to *both* `email` and `customer_email`, but reads from `email`.
3. **Step 3 (Backfill Script)**: Run a background batch script to copy data from `email` to `customer_email` for old historical rows.
4. **Step 4 (Deploy Code v2)**: Switch application to read and write exclusively from `customer_email`.
5. **Step 5 (Contract - DB Migration)**: Add `NOT NULL` constraint to `customer_email` and drop the old `email` column.

---

> **Next Chapter**: [23 Redis Caching Patterns, Cache Failures & NoSQL](23_Redis_Caching_Patterns_NoSQL.md)

