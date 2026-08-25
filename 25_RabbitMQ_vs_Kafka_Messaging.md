# 25. RabbitMQ vs. Apache Kafka: Architecture & Decision Matrix

> **Navigation**: [Master Index](README.md) | [Previous: Kafka Engineering](24_Apache_Kafka_Production_Engineering.md) | [Next: Microservices & DDD](26_Microservices_DDD_Architecture.md)

---

## 📌 Chapter Overview
This module explores **RabbitMQ (AMQP)** architecture, the 4 exchange routing types, Dead Letter Exchanges (DLX), and delivers a comprehensive **RabbitMQ vs. Apache Kafka architectural decision matrix**.

---

## 1. RabbitMQ (AMQP) Architecture

```
                                     +-------------------+
                                     |  Direct Exchange  | ---> Routing: "order.created" ---> [ Orders Queue ]
                                     +-------------------+
                                     |  Fanout Exchange  | ---> Broadcasts to ALL queues  ---> [ Analytics Queue ]
 [ Producer ] ---> [ Exchanges ] --->+-------------------+                                    [ Audit Queue ]
                                     |  Topic Exchange   | ---> Routing: "order.eu.*"    ---> [ EU Orders Queue ]
                                     +-------------------+
                                     |  Headers Exchange | ---> Routing via Message Headers
                                     +-------------------+
```

### Q1. Detail the 4 RabbitMQ Exchange Types.
**Answer:**
1. **Direct Exchange**: Routes messages directly to queues whose binding key **exactly matches** the message routing key (`routingKey = "payment.invoice"`).
2. **Fanout Exchange**: Broadcasts every incoming message to **all bound queues**, ignoring routing keys (Pub-Sub broadcast).
3. **Topic Exchange**: Performs wildcard pattern matching between routing keys and binding keys:
   - `*` (asterisk): Matches **exactly one word** (`order.us.*` matches `order.us.created`).
   - `#` (hash): Matches **zero or more words** (`order.#` matches `order.us.created.express`).
4. **Headers Exchange**: Routes messages based on custom key-value pairs in message AMQP headers rather than routing keys.

---

## 2. RabbitMQ Dead Letter Exchange (DLX) & Message TTL

```java
@Configuration
public class RabbitMqDlqConfig {

    public static final String MAIN_QUEUE = "orders.queue";
    public static final String DLX_EXCHANGE = "orders.dlx.exchange";
    public static final String DLQ_QUEUE = "orders.dlq.queue";

    @Bean
    public Queue mainQueue() {
        return QueueBuilder.durable(MAIN_QUEUE)
            .withArgument("x-dead-letter-exchange", DLX_EXCHANGE)       // Route failed messages here
            .withArgument("x-dead-letter-routing-key", "orders.failed")
            .withArgument("x-message-ttl", 60000)                        // Expire after 60s
            .build();
    }

    @Bean
    public DirectExchange deadLetterExchange() {
        return new DirectExchange(DLX_EXCHANGE);
    }

    @Bean
    public Queue deadLetterQueue() {
        return QueueBuilder.durable(DLQ_QUEUE).build();
    }

    @Bean
    public Binding dlqBinding() {
        return BindingBuilder.bind(deadLetterQueue()).to(deadLetterExchange()).with("orders.failed");
    }
}
```

---

## 3. RabbitMQ vs. Apache Kafka Architectural Comparison

### Q2. Compare RabbitMQ vs. Apache Kafka: Which should you choose?
**Answer:**

| Architectural Dimension | RabbitMQ (Message Broker) | Apache Kafka (Distributed Event Log) |
| :--- | :--- | :--- |
| **Core Architecture** | **Smart Broker / Dumb Consumer** | **Dumb Broker / Smart Consumer** |
| **Data Storage Model** | Transient Queue (Message deleted once consumed/ACKed) | **Immutable Distributed Commit Log** (Retained on disk for days/weeks) |
| **Consumption Model** | **Push-based** (`basic.deliver`) | **Pull-based** (`consumer.poll()`) |
| **Message Ordering** | FIFO per queue (Order lost if parallel consumers ack out-of-order) | **Strict FIFO per Partition** |
| **Replayability** | No (Messages deleted after ACK) | **Yes** (Reset consumer group offsets to replay past days/months) |
| **Throughput** | High (~50k–100k msgs/sec) | **Massive (~1M+ msgs/sec)** via Sequential Disk I/O & Page Cache |
| **Routing Complexity**| **Rich & Dynamic** (Topic wildcards, Headers, Exchanges) | Static Topic & Partition Key hashing |

#### Decision Matrix:
- **Choose RabbitMQ when**: You need complex dynamic routing, individual message-level acknowledgement, granular task scheduling/worker queues, or simple request-reply (RPC).
- **Choose Apache Kafka when**: You need high-throughput event streaming, event sourcing, data replayability, real-time analytics pipelines, or log-based CDC (Change Data Capture).

---

> **Next Chapter**: [26 Microservices Architecture & Domain-Driven Design (DDD)](26_Microservices_DDD_Architecture.md)

