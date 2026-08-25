# 23. Redis Caching Patterns, Spring Cache Abstraction & NoSQL Modeling

> **Navigation**: [Master Index](README.md) | [Previous: Database Migrations](22_Database_Migrations_Flyway_Liquibase.md) | [Next: Apache Kafka Engineering](24_Apache_Kafka_Production_Engineering.md)

---

## 📌 Chapter Overview
This module explores **Distributed Caching Strategies** (Cache-Aside, Write-Through, Near-Cache), the 3 major caching failure modes (**Penetration, Breakdown / Thundering Herd, Avalanche**), the **Spring Cache Abstraction (`@Cacheable`, `@CacheEvict`, `@CachePut`) vs. Hibernate 2nd-Level Cache**, Redis data structures, and **NoSQL modeling**.

---

## 1. Caching Topologies & Patterns

```
+-----------------------------------------------------------------------------------+
|                           CACHE-ASIDE PATTERN (Read / Write)                      |
+-----------------------------------------------------------------------------------+
|  READ FLOW:                                                                       |
|  [ Client ] ---> [ Service ] ---> Check Redis Cache ---> (HIT) ---> Return Data   |
|                                         |                                         |
|                                      (MISS)                                       |
|                                         v                                         |
|                             Query Database ---> Store in Redis ---> Return Data   |
|                                                                                   |
|  WRITE FLOW:                                                                      |
|  [ Client ] ---> [ Service ] ---> Write to Database ---> Evict / Delete from Redis|
+-----------------------------------------------------------------------------------+
```

### Q1. Compare Cache-Aside vs. Write-Through vs. Near-Cache.
**Answer:**
- **Cache-Aside (Lazy Loading)**: Application reads cache. On miss, reads DB, writes to cache, and returns. On write, application updates DB and **evicts/deletes** the cache key (never update cache directly to prevent race conditions).
- **Write-Through**: Application writes data to cache; cache synchronously writes to DB before returning.
- **Write-Behind (Write-Back)**: Application writes to cache; cache asynchronously writes batches to DB (high write throughput, risk of data loss on cache crash).
- **Near-Cache (L1 + L2)**: Ultra-low latency pattern combining local in-memory **L1 Cache (Caffeine)** on each pod with distributed **L2 Cache (Redis)**. Invalidation events are broadcast via Redis Pub/Sub.

---

## 2. Spring Cache Abstraction (`@Cacheable`, `@CachePut`, `@CacheEvict`)

```
+-----------------------------------------------------------------------------------+
|               SPRING CACHE ABSTRACTION VS. HIBERNATE 2ND-LEVEL CACHE              |
+-----------------------------------------------------------------------------------+
|  Feature              | Spring Cache Abstraction          | Hibernate 2nd-Level Cache |
|  -------------------- | --------------------------------- | ------------------------- |
|  **Layer**            | **Service / Method Level**        | **ORM / Entity Level**    |
|  **Trigger**          | Method execution interception     | `em.find()`, JPQL queries |
|  **Cached Data**      | Arbitrary DTOs, Lists, Objects    | Entity state tuples (rows)|
|  **Key**              | Custom SpEL (`#id`, `#user.email`)| Primary Key (`@Id`)        |
|  **Implementation**   | Spring AOP Proxy                  | Hibernate SessionFactory  |
+-----------------------------------------------------------------------------------+
```

### Q2. How do `@Cacheable`, `@CachePut`, and `@CacheEvict` work with Spring AOP?
**Answer:**

```java
@Service
@CacheConfig(cacheNames = "users") // Sets default cache name for the class
public class UserService {

    private final UserRepository userRepository;

    public UserService(UserRepository repo) { this.userRepository = repo; }

    // 1. @Cacheable: Checks cache first. If found, returns cached object without executing method!
    @Cacheable(key = "#id", unless = "#result == null", condition = "#id > 0")
    public UserDto getUserById(Long id) {
        return userRepository.findById(id)
            .map(UserDto::fromEntity)
            .orElse(null);
    }

    // 2. @CachePut: Always executes the method AND updates the cache entry with the return value!
    @CachePut(key = "#result.id")
    public UserDto updateUser(UpdateUserRequest req) {
        User updated = userRepository.save(new User(req));
        return UserDto.fromEntity(updated);
    }

    // 3. @CacheEvict: Removes matching key(s) from cache after database deletion
    @CacheEvict(key = "#id")
    public void deleteUser(Long id) {
        userRepository.deleteById(id);
    }

    // 4. @Caching: Combines multiple cache operations simultaneously
    @Caching(
        evict = {
            @CacheEvict(key = "#user.id"),
            @CacheEvict(value = "userSummaries", allEntries = true) // Clears entire summary cache!
        }
    )
    public void deactivateUser(User user) {
        userRepository.deactivate(user.getId());
    }
}
```

---

## 3. The 3 Major Cache Failure Modes & Production Solutions

```
+-------------------------------------------------------------------------------+
|                       THE 3 CACHE PRODUCTION DISASTERS                        |
+-------------------------------------------------------------------------------+
| 1. Cache Penetration  -> Queries for non-existent IDs bypass cache to hit DB  |
| 2. Cache Breakdown    -> Single HOT key expires, causing Thundering Herd on DB|
| 3. Cache Avalanche    -> Thousands of keys expire simultaneously              |
+-------------------------------------------------------------------------------+
```

### Q3. How do you resolve Cache Penetration, Breakdown, and Avalanche?
**Answer:**

#### 1. Cache Penetration:
- **Cause**: Attackers or bugs query non-existent IDs (e.g., `userId=-999`), bypassing cache and hitting the database every time.
- **Fix A**: Use a **Bloom Filter** in front of the cache to check whether an ID could possibly exist before querying DB.
- **Fix B**: Cache empty/null results with a short TTL (e.g., 60 seconds).

#### 2. Cache Breakdown (Thundering Herd / Stampede):
- **Cause**: A high-traffic "hot" key (e.g., Black Friday product) expires, and 10,000 simultaneous requests hit the DB at the exact same millisecond.
- **Fix**: Use a **Distributed Mutex Lock (Redisson)** so only 1 thread queries the DB and repopulates cache while other threads wait:

```java
public Product getProduct(Long id) {
    Product cached = redisTemplate.opsForValue().get("product:" + id);
    if (cached != null) return cached;

    RLock lock = redissonClient.getLock("lock:product:" + id);
    try {
        if (lock.tryLock(5, 10, TimeUnit.SECONDS)) {
            // Double-check cache inside lock!
            cached = redisTemplate.opsForValue().get("product:" + id);
            if (cached != null) return cached;

            Product dbProduct = productRepo.findById(id).orElseThrow();
            redisTemplate.opsForValue().set("product:" + id, dbProduct, 30, TimeUnit.MINUTES);
            return dbProduct;
        }
    } finally {
        if (lock.isHeldByCurrentThread()) lock.unlock();
    }
    return redisTemplate.opsForValue().get("product:" + id);
}
```

#### 3. Cache Avalanche:
- **Cause**: A batch job populates 100,000 keys with the exact same 1-hour TTL. All 100,000 keys expire at the exact same second, overwhelming the DB.
- **Fix**: Add **Randomized TTL Jitter**:
  $$\text{TTL} = \text{Base TTL} + \text{Random Jitter (e.g., 1 to 5 minutes)}$$

---

## 4. Redis Advanced Data Structures

| Data Structure | Operations | Typical Production Use Case |
| :--- | :--- | :--- |
| **String** | `GET`, `SET`, `INCR` | Session tokens, simple entity cache, atomic counters |
| **Hash** | `HGET`, `HSET`, `HINCRBY` | Partial user profile updates, shopping cart objects |
| **Set** | `SADD`, `SISMEMBER`, `SINTER` | Unique visitors, user roles, social friend intersections |
| **Sorted Set (`ZSET`)**| `ZADD`, `ZRANGEBYSCORE`, `ZREVRANK` | **Leaderboards, Distributed Delayed Queues, Rate Limiters** |
| **Bitmap** | `SETBIT`, `BITCOUNT` | Daily Active User (DAU) analytics (1 bit per user) |
| **HyperLogLog** | `PFADD`, `PFCOUNT` | Massive cardinality counting (100M unique views with only 12 KB memory) |

---

## 5. NoSQL Data Modeling: Cassandra vs. MongoDB

### Q4. How does Cassandra Query-Driven Data Modeling work?
**Answer:**
In RDBMS, you design normalized tables first and write queries later. In **Apache Cassandra**, you **design tables specifically around queries**:
- **Partition Key**: Determines which cluster node stores the data via consistent hashing.
- **Clustering Key**: Determines the physical sort order of data rows inside the partition.
- **Tombstones**: In Cassandra, `DELETE` writes an immutable marker called a **Tombstone**. Querying ranges containing thousands of tombstones degrades read performance until compaction runs.

---

> **Next Chapter**: [24 Apache Kafka Production Engineering & Streaming](24_Apache_Kafka_Production_Engineering.md)
