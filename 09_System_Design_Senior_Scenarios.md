# System Design, Architecture & Senior Production Scenarios

> **Navigation**: [Master Index](README.md) | [Previous: Distributed Systems & Cloud](08_Distributed_Systems_Cloud.md) | [Next: Quick Revision Checklist](10_Quick_Revision_Checklist.md)

---

## 1. High-Level System Design (HLD) Case Studies

### Case Study 1: Distributed Rate Limiter (Token Bucket with Redis Lua Script)
**Problem**: *"Design a distributed rate limiter that limits each user/API key to 100 requests per minute across a cluster of 50 Spring Boot nodes."*

#### Architecture:
```
[ Client Request ]
       |
       v
[ API Gateway / Spring Cloud Gateway Filter ]
       |
       +---> Executes Atomic Redis Lua Script (Token Bucket)
       |        |
       |        +---> [ Redis Cluster ] (Tracks token count & timestamp per API Key)
       |
       +--- If allowed: Forward to Downstream Service
       +--- If limit exceeded: Return HTTP 429 Too Many Requests
```

#### Redis Lua Script (Atomic Token Bucket Implementation):
```lua
-- KEYS[1]: Rate limit key (e.g., "ratelimit:user:123")
-- ARGV[1]: Max capacity (e.g., 100)
-- ARGV[2]: Refill rate per millisecond (e.g., 100 tokens / 60000 ms)
-- ARGV[3]: Current timestamp in ms
-- ARGV[4]: Requested tokens (1)

local key = KEYS[1]
local capacity = tonumber(ARGV[1])
local refill_rate = tonumber(ARGV[2])
local now = tonumber(ARGV[3])
local requested = tonumber(ARGV[4])

local data = redis.call("HMGET", key, "tokens", "last_updated")
local tokens = tonumber(data[1])
local last_updated = tonumber(data[2])

if tokens == nil then
    tokens = capacity
    last_updated = now
else
    local delta = math.max(0, now - last_updated)
    tokens = math.min(capacity, tokens + (delta * refill_rate))
    last_updated = now
end

if tokens >= requested then
    tokens = tokens - requested
    redis.call("HMSET", key, "tokens", tokens, "last_updated", last_updated)
    redis.call("PEXPIRE", key, 60000) -- Expire key after 1 minute of inactivity
    return 1 -- ALLOWED
else
    return 0 -- REJECTED (HTTP 429)
end
```

---

### Case Study 2: Distributed URL Shortener (TinyURL)
**Scale**: 100 Million new URLs created per month, 10 Billion reads per month (100:1 Read-to-Write ratio).

```
+-----------------------------------------------------------------------------------+
|                        TINYURL DISTRIBUTED ARCHITECTURE                           |
+-----------------------------------------------------------------------------------+
|  [ Client ] ---> [ Cloudflare CDN / Edge Cache ] (Cached popular short URLs)      |
|                       | (Cache Miss)                                              |
|                       v                                                           |
|             [ Load Balancer (Nginx) ]                                             |
|                       |                                                           |
|                       v                                                           |
|             [ Spring Boot Web Nodes ]                                             |
|              /                       \                                            |
|   (Write / Shorten)              (Read / Redirect)                                |
|          /                               \                                        |
|         v                                 v                                       |
|  [ Snowflake ID Gen ]               [ Redis Cache Cluster ]                       |
|         |                                 | (Cache Miss)                          |
|         v (Base62 Encode)                 v                                       |
|  [ Cassandra / MongoDB / PostgreSQL Sharded Cluster ]                             |
+-----------------------------------------------------------------------------------+
```

#### Key Design Decisions:
1. **Short URL Generation**:
   - Do NOT use MD5/SHA256 hashes (too long, requires truncating and collision handling).
   - Use **Twitter Snowflake Distributed 64-bit ID Generator** (Timestamp + Node ID + Sequence).
   - Convert 64-bit integer into **Base62 String** (`[a-z, A-Z, 0-9]`). A 7-character Base62 string yields `62^7 ≈ 3.52 Trillion` unique short URLs!
2. **Database Choice**: NoSQL Key-Value / Document (MongoDB, Cassandra, or DynamoDB) partitioned by `short_key`.
3. **Caching**: Redis Cache with LRU eviction for the top 20% most accessed URLs (80/20 Pareto rule).

---

## 2. Senior Live Production Incident Walkthroughs

### Incident 1: 504 Gateway Timeouts & Connection Pool Starvation
**Symptom**: During a marketing campaign, API Gateway returns `504 Gateway Timeout`. CPU and Memory usage on Spring Boot pods are only 15%, but all requests are stuck.

#### Root Cause Analysis:
1. Inspected HikariCP metrics: `hikaricp_connections_pending` spiked to 500, while `hikaricp_connections_active` was maxed out at 10.
2. Thread dump showed all 200 Tomcat worker threads in `TIMED_WAITING` inside `HikariDataSource.getConnection()`.
3. Traced database query logs: A third-party external REST API call was being made **inside an open `@Transactional` method**!
   ```java
   // THE ANTI-PATTERN:
   @Transactional
   public void processOrder(OrderReq req) {
       Order order = orderRepo.save(new Order(req)); // DB Connection acquired from HikariCP!
       paymentGatewayClient.chargeCreditCard(req);   // 5-second blocking HTTP call while HOLDING DB connection!
       order.setStatus(PAID);
   }
   ```
4. While the service waited 5 seconds for the external payment gateway, the database connection remained locked, starving all other incoming requests.

#### The Fix:
- Move external network calls **outside the `@Transactional` boundary**:
  ```java
  public void processOrder(OrderReq req) {
      PaymentResponse payment = paymentGatewayClient.chargeCreditCard(req); // No DB connection held!
      orderTxService.saveOrderWithPayment(req, payment); // Short 2ms DB transaction!
  }
  ```

---

### Incident 2: Cascading Failure across Microservices
**Symptom**: Service C slows down due to high database load. Suddenly, Service B and Service A crash in sequence, causing a full system outage.

#### Root Cause Analysis:
- Service A called Service B synchronously, and Service B called Service C synchronously without configured **timeouts** or **circuit breakers**.
- When Service C slowed from 20ms to 8 seconds, Service B's Tomcat worker threads (200 threads) all became blocked waiting for Service C.
- Once Service B exhausted its 200 threads, Service A exhausted its 200 threads waiting for Service B, bringing down the entire platform.

#### The Fix:
1. **Enforce Strict HTTP Timeouts**: `connectTimeout = 1s`, `readTimeout = 2s` on all HTTP/Feign clients.
2. **Resilience4j Circuit Breaker**: Trip circuit breaker to `OPEN` state when error rate > 50%, immediately returning fallback responses and shedding load.
3. **Decouple with Kafka**: Convert synchronous inter-service chains into asynchronous event streams where immediate response is not required.

---

> **Next Chapter**: [10 Quick Revision Checklist](10_Quick_Revision_Checklist.md)
