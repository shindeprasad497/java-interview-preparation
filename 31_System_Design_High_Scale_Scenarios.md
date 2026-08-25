# 31. Senior System Design: High-Scale Scenarios, Rate Limiters & Protocols

> **Navigation**: [Master Index](README.md) | [Previous: Docker & Kubernetes](30_Cloud_Docker_Kubernetes.md) | [Next: Production Incidents](32_Production_Incident_PostMortems.md)

---

## 📌 Chapter Overview
Production-grade System Design case studies frequently evaluated in **Senior (L5) & Lead/Staff (L6) Engineering Interviews**: **The 4 Rate Limiter Algorithms & Redis Lua Implementation**, **Flash Sale / Ticket Booking**, **Distributed Task Scheduling**, **Payment Idempotency**, and **API Protocols (gRPC vs REST vs GraphQL vs SSE)**.

---

## 1. Deep Dive: The 4 Distributed Rate Limiting Algorithms

```
+-----------------------------------------------------------------------------------+
|                        THE 4 RATE LIMITING ALGORITHMS                             |
+-----------------------------------------------------------------------------------+
|  1. Token Bucket:       Tokens added at steady rate; allows bursts up to capacity |
|  2. Leaky Bucket:       Requests enter queue; drained at strict constant rate     |
|  3. Fixed Window:       Counter resets at fixed intervals (vulnerable at borders) |
|  4. Sliding Window Log: Timestamps stored in Redis ZSET; 100% accurate, higher RAM |
+-----------------------------------------------------------------------------------+
```

### Q1. Compare Token Bucket, Leaky Bucket, Fixed Window, and Sliding Window.
**Answer:**

| Algorithm | Handles Bursts? | Memory Footprint | Edge-Case Weakness | Best Production Use Case |
| :--- | :--- | :--- | :--- | :--- |
| **Token Bucket** | **Yes** (Up to bucket capacity) | Minimal (Tokens count + Last refill timestamp) | None (Standard industry choice) | **API Gateways (AWS API Gateway, Stripe, Spring Cloud Gateway)** |
| **Leaky Bucket** | No (Smooths bursts to constant rate) | Small (Queue of pending requests) | Can drop traffic under legitimate bursts | Downstream systems requiring constant steady writes |
| **Fixed Window** | No | Minimal (Single integer counter per window) | **Boundary Spike Problem**: $2\times$ traffic allowed at window boundaries! | Simple non-critical rate limits |
| **Sliding Window Counter** | Yes (Weighted interpolation) | Low (Current + previous window counters) | Minor approximation error ($\approx 0.05\%$) | High-scale distributed rate limiters |

#### Redis Lua Script for Sliding Window Log (Using `ZSET`):
```lua
local key = KEYS[1]
local now = tonumber(ARGV[1])
local window = tonumber(ARGV[2]) -- e.g., 60 seconds
local limit = tonumber(ARGV[3])  -- e.g., 100 requests

local clearBefore = now - window
-- 1. Remove timestamps older than window
redis.call('ZREMRANGEBYSCORE', key, 0, clearBefore)

-- 2. Count requests in current window
local currentRequests = redis.call('ZCARD', key)

if currentRequests < limit then
    -- 3. Add current timestamp
    redis.call('ZADD', key, now, now)
    redis.call('EXPIRE', key, window)
    return 1 -- Allowed
else
    return 0 -- Rejected (HTTP 429 Too Many Requests)
end
```

---

## 2. Case Study 1: Flash Sale & Ticket Booking (Anti-Overselling Architecture)

```
                                 [ 1,000,000 Concurrent Users ]
                                                |
                                                v
                                  [ Cloudflare / API Gateway ]
                                  (DDoS & Token Bucket Limiter)
                                                |
                                                v
                        +-----------------------------------------------+
                        | LAYER 1: Redis In-Memory Pre-Deduction        |
                        | Atomic Lua Script: If Stock > 0 -> DECR Stock |
                        +-----------------------------------------------+
                                                | (Only 1,000 pass through!)
                                                v
                        +-----------------------------------------------+
                        | LAYER 2: Apache Kafka Buffering               |
                        | Topic: "flashsale.orders" (Partitions by Item)|
                        +-----------------------------------------------+
                                                |
                                                v
                        +-----------------------------------------------+
                        | LAYER 3: Async Order Service + PostgreSQL     |
                        | Optimistic Locking (@Version) + Payment Init  |
                        +-----------------------------------------------+
```

### Q2. How do you prevent overselling during high-concurrency flash sales?
**Answer:**
1. **Never query DB directly for stock**: 100,000 concurrent DB queries will crash PostgreSQL/MySQL.
2. **Layer 1 (Redis Atomic Lua Script)**: Pre-load inventory into Redis (`stock:item:100 = 500`). Run an atomic Lua script that checks and decrements stock in a single atomic CPU cycle. If stock reaches 0, all subsequent requests are rejected in $<1\text{ms}$ with `"Sold Out"`.
3. **Layer 2 (Kafka Async Queue)**: The 500 winning requests are pushed into a Kafka topic to buffer downstream database writes.
4. **Layer 3 (Database Persistence)**: Order consumer reads Kafka messages and inserts orders into PostgreSQL using **Optimistic Locking (`@Version`)** to guarantee absolute database consistency.

#### Redis Lua Script for Atomic Inventory Deduction:
```lua
local stock = tonumber(redis.call('get', KEYS[1]))
if stock and stock >= tonumber(ARGV[1]) then
    redis.call('decrby', KEYS[1], ARGV[1])
    return 1 -- Success
else
    return 0 -- Insufficient stock
end
```

---

## 3. Case Study 2: Distributed Delayed Task Scheduler (Redis `ZSET`)

### Q3. How do you implement a distributed delayed job scheduler without polling DB tables?
**Answer:**

```
 [ Schedule Task at T+30min ] ---> ZADD "delayed_tasks" <EpochTimestamp> "taskId_101"
                                                |
                                                v
 [ 10 Worker Pods (Cron Poll) ] -> ZRANGEBYSCORE "delayed_tasks" 0 <CurrentEpoch> LIMIT 0 10
                                                |
                                                v
                     Atomic ZREM "delayed_tasks" "taskId_101" (Task Claimed!)
                                                |
                                                v
                                [ Execute Business Logic ]
```

1. **Scheduling**: Store tasks in a Redis **Sorted Set (`ZSET`)**, where the **Score** is the execution Unix epoch timestamp (`ZADD delayed_tasks 1719830400 task_101`).
2. **Execution Worker**: Worker pods periodically query `ZRANGEBYSCORE delayed_tasks 0 <current_timestamp> LIMIT 0 10`.
3. **Claiming**: Workers use an atomic Lua script with `ZREM` to guarantee only one worker claims and processes the task.

---

## 4. Case Study 3: Payment Gateway Integration & Idempotency Key Engine

### Q4. How do you guarantee exact payment idempotency across network timeouts?
**Answer:**

```
 [ Client / Mobile ] ---> HTTP POST /v1/payments [ Header: 'Idempotency-Key: uuid-v4' ]
                                     |
                                     v
                        [ Idempotency Filter / AOP ]
                                     |
                +--------------------+--------------------+
                | Check Redis: SETNX 'idemp:<key>' 'IN_PROGRESS' EX 120s
                |                                         |
                v (Key already exists)                    v (Key acquired!)
     Is Status == 'COMPLETED'?                    1. Process DB & Call Bank Gateway
        /              \                          2. Store Result in Redis: 'COMPLETED'
  (YES)/                \(NO: 'IN_PROGRESS')      3. Return HTTP 200 Response
      v                  v
Return Cached Response  Return HTTP 409 Conflict ("Processing in progress")
```

---

## 5. Modern API Protocols: gRPC vs. REST vs. GraphQL vs. SSE

| Protocol | Transport | Data Format | Communication Pattern | Best Production Use Case |
| :--- | :--- | :--- | :--- | :--- |
| **REST** | HTTP/1.1 or HTTP/2 | JSON / XML | Unary Request-Response | Public APIs, CRUD microservices |
| **gRPC** | **HTTP/2 Multiplexing**| **Protobuf (Binary)** | Unary, Server Stream, Client Stream, **Bi-directional Stream** | **Internal East-West microservice-to-microservice calls** (10x faster than REST) |
| **GraphQL**| HTTP | JSON | Flexible Query Selection | Complex mobile client UIs (solves over-fetching & under-fetching) |
| **SSE** | HTTP/1.1 or HTTP/2 | Text Stream | Unidirectional Server $\rightarrow$ Client Push | Stock tickers, live sports scores, LLM token streaming |

---

> **Next Chapter**: [32 Production Incident Post-Mortems & Live RCA](32_Production_Incident_PostMortems.md)
