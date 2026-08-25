# 20. Database Internals, B-Tree Indexing & SQL Query Tuning

> **Navigation**: [Master Index](README.md) | [Previous: Hibernate & JPA](19_Hibernate_JPA_Internals.md) | [Next: Transactions & Locking](21_Transactions_Locking_Isolation.md)

---

## 📌 Chapter Overview
This module explores **Relational Database Internals**, B-Tree vs. LSM-Tree storage engines, Clustered vs. Non-Clustered indexing, the **Leftmost Prefix Rule**, reading **`EXPLAIN ANALYZE`** execution plans, and resolving **Deep Pagination bottlenecks**.

---

## 1. Database Indexing Architecture: B-Tree vs. LSM-Tree

```
+-----------------------------------------------------------------------------------+
|                            B+ TREE INDEX STRUCTURE                                |
+-----------------------------------------------------------------------------------+
|                         [ Root Node: (50 | 100) ]                                 |
|                               /       |       \                                   |
|        [ Branch: (20 | 35) ]  [ Branch: (70 | 85) ]   [ Branch: (120 | 150) ]     |
|             /      \              /       \               /        \              |
|        [ 10, 15 ] [ 20, 30 ]  [ 60, 65 ] [ 70, 80 ]  [ 110, 115 ] [ 120, 140 ]    |
|        <================ Leaf Nodes (Doubly-Linked List) ================>        |
|        * Leaf nodes store actual table Row Pointers or Clustered Primary Keys     |
+-----------------------------------------------------------------------------------+
```

### Q1. Compare B-Tree (PostgreSQL/MySQL InnoDB) vs. LSM-Tree (Cassandra/RocksDB).
**Answer:**

| Feature | B+ Tree (RDBMS: MySQL, PostgreSQL) | LSM-Tree (Log-Structured Merge-Tree) |
| :--- | :--- | :--- |
| **Primary Strength** | **Fast Read Latency** ($O(\log N)$ point lookups & range queries) | **Ultra-High Write Throughput** |
| **Write Mechanism** | In-place page updates (Random disk I/O) | Append-only in-memory MemTable $\rightarrow$ flushed to SSTables |
| **Storage Structure**| Balanced tree on disk pages | Multi-level immutable sorted string tables |
| **Typical Engines** | PostgreSQL, MySQL (InnoDB), Oracle | Apache Cassandra, RocksDB, ScyllaDB |

---

## 2. Clustered vs. Non-Clustered Index & Covering Indexes

### Q2. Explain Clustered Index vs. Non-Clustered (Secondary) Index.
**Answer:**
- **Clustered Index**: The actual table data is physically sorted and stored directly in the leaf pages of the index. A table can have **only one** clustered index (typically the Primary Key).
- **Non-Clustered (Secondary) Index**: Contains the indexed column values and a pointer back to the clustered index key.
  - Querying via secondary index requires **2 lookups**: Secondary index lookup $\rightarrow$ Clustered index lookup (**Bookmark / Table Lookup**).
- **Covering Index (Index-Only Scan)**: An index containing *all* columns requested by the `SELECT` and `WHERE` clauses. The database satisfies the query directly from index memory without touching table disk pages!

---

## 3. The Composite Index Leftmost Prefix Rule

### Q3. How does the Leftmost Prefix Rule work for composite indexes?
**Answer:**
If you create a composite index on 3 columns: `CREATE INDEX idx_user ON orders(status, created_at, user_id);`

```
  INDEX ON (status, created_at, user_id):
  -------------------------------------------------------------
  Query Condition                                     Uses Index?
  -------------------------------------------------------------
  WHERE status = 'PAID'                              -> YES (Full prefix)
  WHERE status = 'PAID' AND created_at > '2026-01-01' -> YES (Prefix match)
  WHERE status = 'PAID' AND user_id = 42              -> PARTIAL (Uses status only)
  WHERE created_at > '2026-01-01'                    -> ❌ NO (Leftmost column missing!)
  WHERE user_id = 42                                 -> ❌ NO (Leftmost column missing!)
```

---

## 4. Reading `EXPLAIN ANALYZE` Query Plans

### Q4. Compare `Seq Scan` vs. `Index Scan` vs. `Index Only Scan`.
**Answer:**
- **`Seq Scan` (Sequential Table Scan)**: Reads every page of the table from disk. Acceptable only for very small tables or queries fetching $>20\%$ of all rows.
- **`Index Scan`**: Traverses B-Tree index to find matching keys, then fetches rows from the table heap.
- **`Index Only Scan` (Fastest)**: Satisfies query completely from index memory without accessing table heap data.

---

## 5. SQL Anti-Patterns & Deep Pagination Resolution

### Q5. Why is `LIMIT 10 OFFSET 1000000` slow? How do you fix Deep Pagination?
**Answer:**
- **The Problem**: With `OFFSET 1000000`, the database must scan, sort, and discard all 1,000,000 preceding rows before returning the next 10 rows $\rightarrow$ High CPU & Disk I/O.
- **Production Solution: Keyset (Seek) Pagination**: Use the last seen primary key or timestamp instead of `OFFSET`:

```sql
-- ❌ SLOW ANTI-PATTERN (Deep Pagination):
SELECT * FROM orders ORDER BY id ASC LIMIT 10 OFFSET 1000000;

-- ✅ PRODUCTION FAST FIX (Keyset / Seek Pagination):
SELECT * FROM orders WHERE id > 1000000 ORDER BY id ASC LIMIT 10;
```
*(Executes instantaneously in $O(\log N)$ via direct B-Tree index seek!)*

---

> **Next Chapter**: [21 Transactions, Locking & Database Isolation Levels](21_Transactions_Locking_Isolation.md)

