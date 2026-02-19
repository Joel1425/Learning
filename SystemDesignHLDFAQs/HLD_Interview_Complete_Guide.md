# HLD Interview Complete Guide - SDE-2 Backend

> A comprehensive guide covering all follow-up questions asked in High-Level Design interviews with detailed answers, flowcharts, and real-world examples.

---

## Table of Contents

1. [Database Selection & Trade-offs](#1-database-selection--trade-offs)
2. [Database Failure Handling](#2-database-failure-handling)
3. [Message Queues - Kafka vs SQS vs RabbitMQ](#3-message-queues---kafka-vs-sqs-vs-rabbitmq)
4. [Caching Strategies & Redis](#4-caching-strategies--redis)
5. [Service Communication Patterns](#5-service-communication-patterns)
6. [Concurrency & Locking Mechanisms](#6-concurrency--locking-mechanisms)
7. [Rate Limiting & Throttling](#7-rate-limiting--throttling)
8. [Load Balancing & Scaling](#8-load-balancing--scaling)
9. [Data Consistency & CAP Theorem](#9-data-consistency--cap-theorem)
10. [Monitoring, Logging & Alerting](#10-monitoring-logging--alerting)
11. [Common HLD Scenarios with Complete Solutions](#11-common-hld-scenarios-with-complete-solutions)

---

## 1. Database Selection & Trade-offs

### Q: Which database would you use for this system?

**Framework for answering:**

```
┌─────────────────────────────────────────────────────────────────┐
│                    DATABASE SELECTION FRAMEWORK                  │
├─────────────────────────────────────────────────────────────────┤
│  1. What is the data structure? (Structured/Semi/Unstructured)  │
│  2. What are the read/write patterns? (Read-heavy/Write-heavy)  │
│  3. Do we need ACID transactions?                               │
│  4. What is the expected scale? (GB/TB/PB)                      │
│  5. Do we need complex queries/joins?                           │
│  6. What is the consistency requirement?                        │
└─────────────────────────────────────────────────────────────────┘
```

### Database Comparison Matrix

| Criteria | MySQL/PostgreSQL | MongoDB | Cassandra | Redis | DynamoDB |
|----------|------------------|---------|-----------|-------|----------|
| **Data Model** | Relational | Document | Wide-column | Key-Value | Key-Value/Document |
| **ACID** | Full | Document-level | Tunable | Limited | Item-level |
| **Scale** | Vertical (some horizontal) | Horizontal | Horizontal | In-memory | Horizontal |
| **Consistency** | Strong | Eventual/Strong | Eventual | Strong | Eventual/Strong |
| **Best For** | Complex queries, transactions | Flexible schema, rapid dev | Time-series, high writes | Caching, sessions | Serverless, auto-scale |
| **Latency** | ~10ms | ~5-10ms | ~2-5ms | <1ms | ~5-10ms |

### Q: Why PostgreSQL over MySQL?

**Answer:**
```
PostgreSQL advantages:
├── Advanced data types (JSONB, Arrays, UUID native)
├── Better MVCC implementation (no read locks)
├── Full-text search built-in
├── Window functions & CTEs more mature
├── Better handling of concurrent writes
└── Extensions ecosystem (PostGIS, TimescaleDB)

MySQL advantages:
├── Simpler replication setup
├── Lower memory footprint
├── Faster for simple read-heavy workloads
└── Better tooling/community for web apps
```

**Example scenario:**
- **E-commerce order system** → PostgreSQL (complex queries, ACID transactions)
- **Blog/CMS** → MySQL (simple queries, read-heavy)
- **User activity logs** → Cassandra (high write throughput, time-series)
- **Session storage** → Redis (fast access, TTL support)

### Q: Why NoSQL over SQL for this use case?

**Answer with decision tree:**

```
                        ┌─────────────────┐
                        │ Need ACID txns? │
                        └────────┬────────┘
                                 │
                    ┌────────────┴────────────┐
                    │ YES                     │ NO
                    ▼                         ▼
            ┌───────────────┐        ┌───────────────────┐
            │ Use SQL (RDBMS)│        │ Schema flexibility │
            └───────────────┘        │     needed?        │
                                     └─────────┬─────────┘
                                               │
                                  ┌────────────┴────────────┐
                                  │ YES                     │ NO
                                  ▼                         ▼
                          ┌─────────────┐          ┌───────────────┐
                          │  MongoDB    │          │ High write    │
                          │  (Document) │          │ throughput?   │
                          └─────────────┘          └───────┬───────┘
                                                           │
                                              ┌────────────┴────────────┐
                                              │ YES                     │ NO
                                              ▼                         ▼
                                      ┌─────────────┐          ┌─────────────┐
                                      │ Cassandra   │          │ DynamoDB/   │
                                      │             │          │ Redis       │
                                      └─────────────┘          └─────────────┘
```

### Q: When would you use a Graph Database?

**Answer:**
Use graph databases (Neo4j, Amazon Neptune) when:
- Relationships ARE the data (social networks, fraud detection)
- You need multi-hop traversals (friend-of-friend queries)
- Complex relationship patterns exist

**Example - Social Network:**
```
SQL approach (inefficient):
SELECT * FROM users u1
JOIN friendships f1 ON u1.id = f1.user_id
JOIN friendships f2 ON f1.friend_id = f2.user_id
JOIN users u2 ON f2.friend_id = u2.id
WHERE u1.id = 123 AND u2.id != 123;
-- Gets slower exponentially with depth

Graph approach (efficient):
MATCH (me:User {id: 123})-[:FRIEND*2..3]-(fof:User)
RETURN DISTINCT fof;
-- Consistent performance regardless of data size
```

---

## 2. Database Failure Handling

### Q: What happens if your database goes down?

**Multi-layer failure handling strategy:**

```
┌────────────────────────────────────────────────────────────────────────┐
│                     DATABASE FAILURE HANDLING                          │
├────────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  Layer 1: REPLICATION (Automatic failover)                            │
│  ┌──────────┐    sync/async    ┌──────────┐                          │
│  │  Primary │ ───────────────► │ Replica  │                          │
│  │   (RW)   │                  │   (RO)   │                          │
│  └──────────┘                  └──────────┘                          │
│       │                              │                                │
│       ▼                              ▼                                │
│  Layer 2: CONNECTION POOLING (PgBouncer, ProxySQL)                   │
│  - Manages connection limits                                          │
│  - Automatic retry logic                                              │
│  - Health checks                                                      │
│       │                                                               │
│       ▼                                                               │
│  Layer 3: APPLICATION-LEVEL FALLBACKS                                │
│  - Circuit breaker pattern                                            │
│  - Graceful degradation                                               │
│  - Queue writes for retry                                             │
│       │                                                               │
│       ▼                                                               │
│  Layer 4: DISASTER RECOVERY                                          │
│  - Cross-region replicas                                              │
│  - Point-in-time recovery                                             │
│  - Backup restoration                                                 │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

### Q: Explain your replication strategy

**Answer:**

```
SYNCHRONOUS REPLICATION:
┌──────────┐  1. Write  ┌──────────┐
│  Client  │ ─────────► │ Primary  │
└──────────┘            └────┬─────┘
                             │ 2. Replicate (wait for ACK)
                             ▼
                        ┌──────────┐
                        │ Replica  │
                        └────┬─────┘
                             │ 3. ACK
                             ▼
┌──────────┐  4. Success ┌──────────┐
│  Client  │ ◄───────────│ Primary  │
└──────────┘             └──────────┘

Pros: Zero data loss (RPO = 0)
Cons: Higher latency, reduced throughput


ASYNCHRONOUS REPLICATION:
┌──────────┐  1. Write  ┌──────────┐
│  Client  │ ─────────► │ Primary  │
└──────────┘            └────┬─────┘
      ▲                      │
      │ 2. Success           │ 3. Replicate (async)
      │    (immediate)       ▼
      │                 ┌──────────┐
      └─────────────────│ Replica  │
                        └──────────┘

Pros: Lower latency, higher throughput
Cons: Potential data loss (RPO > 0, typically seconds)
```

**When to use which:**
- **Synchronous**: Financial transactions, order systems
- **Asynchronous**: Analytics, logging, non-critical data

### Q: How do you handle split-brain in database clusters?

**Answer:**

```
SPLIT-BRAIN SCENARIO:
                    Network Partition
                          ║
┌──────────────┐          ║          ┌──────────────┐
│  Data Center │          ║          │  Data Center │
│      A       │          ║          │      B       │
│              │          ║          │              │
│  ┌────────┐  │          ║          │  ┌────────┐  │
│  │ Node 1 │  │    X─────╫─────X    │  │ Node 2 │  │
│  │(Primary)│  │          ║          │  │(Primary)│ │ ← Both think they're primary!
│  └────────┘  │          ║          │  └────────┘  │
└──────────────┘          ║          └──────────────┘

SOLUTIONS:

1. QUORUM-BASED (Odd number of nodes):
   - Need majority (n/2 + 1) to accept writes
   - Minority partition becomes read-only

2. FENCING (STONITH - Shoot The Other Node In The Head):
   - Use external arbiter to kill the rogue primary
   - Hardware fencing (power off)
   - Software fencing (revoke access)

3. WITNESS/ARBITER NODE:
   ┌──────────────┐     ┌──────────────┐     ┌──────────────┐
   │    DC-A      │     │   Witness    │     │    DC-B      │
   │   (Node 1)   │◄───►│   (Arbiter)  │◄───►│   (Node 2)   │
   └──────────────┘     └──────────────┘     └──────────────┘

   - Witness votes for which DC stays primary
   - Usually in a third location (cloud)
```

### Q: What is your fallback if the database is completely unavailable?

**Graceful Degradation Strategy:**

```java
// Database fallback pattern

public class OrderService {
    private Database db;
    private Database dbReplica;
    private MessageQueue messageQueue;
    private Cache cache;

    public OrderResponse placeOrder(Order order) {
        try {
            // Attempt 1: Write to primary DB
            return db.write(order);
        } catch (DatabaseUnavailableException e) {
            try {
                // Attempt 2: Write to read replica (if promoted)
                return dbReplica.write(order);
            } catch (DatabaseUnavailableException e2) {
                // Attempt 3: Queue for later processing
                messageQueue.publish("pending_orders", order);
                return new OrderResponse(
                    "accepted",
                    "Order queued for processing",
                    generateTempId()
                );
            }
        }
    }

    public OrderResponse getOrder(String orderId) {
        try {
            return db.read(orderId);
        } catch (DatabaseUnavailableException e) {
            // Fallback 1: Try cache
            OrderResponse cached = cache.get("order:" + orderId);
            if (cached != null) {
                return cached;
            }
            // Fallback 2: Return partial/stale data
            return new OrderResponse("temporarily_unavailable");
        }
    }
}
```

**Read vs Write Fallbacks:**

| Scenario | Read Fallback | Write Fallback |
|----------|---------------|----------------|
| Primary down | Read from replica | Queue writes + notify user |
| All DBs down | Serve from cache (stale) | Queue to Kafka + async processing |
| Network partition | Return cached/degraded response | Accept with "pending" status |

---

## 3. Message Queues - Kafka vs SQS vs RabbitMQ

### Q: Why Kafka and not SQS?

**Detailed Comparison:**

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        MESSAGE QUEUE COMPARISON                          │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  KAFKA                          │  SQS                                  │
│  ──────                         │  ───                                  │
│  ✓ High throughput (millions/s) │  ✓ Serverless, no management         │
│  ✓ Message replay/rewind        │  ✓ Auto-scaling                       │
│  ✓ Ordering within partition    │  ✓ Pay-per-use                        │
│  ✓ Long retention (days/weeks)  │  ✓ Dead-letter queues built-in       │
│  ✓ Consumer groups              │  ✓ Easy integration with AWS         │
│  ✗ Complex to manage            │  ✗ Max 256KB message size            │
│  ✗ Higher operational cost      │  ✗ No replay capability              │
│  ✗ Requires expertise           │  ✗ 14-day max retention              │
│                                                                          │
│  RABBITMQ                                                                │
│  ─────────                                                               │
│  ✓ Flexible routing (exchanges) │  ✓ Multiple protocols (AMQP, MQTT)   │
│  ✓ Message priority             │  ✓ Good for complex routing          │
│  ✓ Transaction support          │  ✗ Lower throughput than Kafka       │
│  ✓ Message acknowledgment       │  ✗ No built-in replay                │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### Decision Matrix:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      WHEN TO USE WHAT                                    │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  USE KAFKA WHEN:                                                         │
│  ├── Event streaming/sourcing (user activity, logs)                     │
│  ├── Need message replay (debugging, reprocessing)                      │
│  ├── High throughput required (>100K msg/sec)                           │
│  ├── Multiple consumers need same messages                              │
│  └── Building data pipelines                                            │
│                                                                          │
│  USE SQS WHEN:                                                           │
│  ├── Simple task queuing (email sending, notifications)                 │
│  ├── Serverless architecture (Lambda integration)                       │
│  ├── Don't want to manage infrastructure                                │
│  ├── Predictable low-medium throughput                                  │
│  └── AWS-native ecosystem                                               │
│                                                                          │
│  USE RABBITMQ WHEN:                                                      │
│  ├── Complex routing requirements                                        │
│  ├── Need message priority                                               │
│  ├── Request-reply pattern                                               │
│  ├── Transactions required                                               │
│  └── Multiple protocols needed                                           │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### Q: What happens if Kafka goes down?

**Kafka Failure Handling:**

```
KAFKA ARCHITECTURE FOR HIGH AVAILABILITY:

Topic: orders (Replication Factor: 3)

┌─────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│   Broker 1              Broker 2              Broker 3                  │
│   ┌─────────────┐      ┌─────────────┐      ┌─────────────┐            │
│   │ Partition 0 │      │ Partition 0 │      │ Partition 0 │            │
│   │  (LEADER)   │─────►│  (REPLICA)  │─────►│  (REPLICA)  │            │
│   └─────────────┘      └─────────────┘      └─────────────┘            │
│   ┌─────────────┐      ┌─────────────┐      ┌─────────────┐            │
│   │ Partition 1 │      │ Partition 1 │      │ Partition 1 │            │
│   │  (REPLICA)  │◄─────│  (LEADER)   │─────►│  (REPLICA)  │            │
│   └─────────────┘      └─────────────┘      └─────────────┘            │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘

FAILURE SCENARIO - Broker 1 Dies:

Before:  Broker 1 (P0-Leader, P1-Replica)
After:   Broker 2 promotes P0-Replica to P0-Leader
         No data loss if min.insync.replicas ≥ 2
```

**Producer-side fallback:**

```java
// Kafka producer with fallback
public class ResilientProducer {
    private KafkaProducer<String, String> kafkaProducer;
    private RedisQueue fallbackQueue;

    public ResilientProducer() {
        Properties props = new Properties();
        props.put("bootstrap.servers", "kafka1:9092,kafka2:9092,kafka3:9092");
        props.put("acks", "all");  // Wait for all replicas
        props.put("retries", 5);
        props.put("retry.backoff.ms", 1000);
        props.put("key.serializer", StringSerializer.class.getName());
        props.put("value.serializer", StringSerializer.class.getName());

        this.kafkaProducer = new KafkaProducer<>(props);
        this.fallbackQueue = new RedisQueue();  // or local file
    }

    public void send(String topic, String message) {
        try {
            ProducerRecord<String, String> record = new ProducerRecord<>(topic, message);
            kafkaProducer.send(record).get(10, TimeUnit.SECONDS);  // Block until confirmed
        } catch (Exception e) {
            // Fallback: Store locally and retry later
            FallbackMessage fallback = new FallbackMessage(
                topic,
                message,
                Instant.now(),
                0  // retry_count
            );
            fallbackQueue.push(fallback);
            // Background job will retry these
        }
    }
}
```

### Q: How do you ensure exactly-once delivery?

**The Three Delivery Semantics:**

```
┌───────────────────────────────────────────────────────────────────────��─┐
│                       DELIVERY GUARANTEES                                │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  AT-MOST-ONCE:                                                           │
│  Producer ──► Broker ──► Consumer                                       │
│     │            │                                                       │
│     └── Fire & forget, no retry                                         │
│     └── Message may be lost                                              │
│     └── Use case: Metrics, logs (where loss is acceptable)              │
│                                                                          │
│  AT-LEAST-ONCE (Default for most systems):                              │
│  Producer ──► Broker ──► Consumer                                       │
│     │            │           │                                           │
│     └── Retry on failure     └── May process duplicates                 │
│     └── No message loss                                                  │
│     └── Use case: Most applications (with idempotent processing)        │
│                                                                          │
│  EXACTLY-ONCE (Hardest to achieve):                                     │
│  Producer ──► Broker ──► Consumer                                       │
│     │            │           │                                           │
│     └── Idempotent          └── Transactional                           │
│         producer                 consumer                                │
│     └── Use case: Financial transactions, inventory                     │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

**Implementing Exactly-Once:**

```java
// Strategy 1: Idempotent Consumer with Deduplication

public class IdempotentOrderProcessor {
    private RedisTemplate<String, String> processedIds;
    private DataSource dataSource;

    public IdempotentOrderProcessor(RedisTemplate<String, String> redis, DataSource ds) {
        this.processedIds = redis;
        this.dataSource = ds;
    }

    public void process(Message message) throws Exception {
        String messageId = message.getId();

        // Check if already processed
        if (Boolean.TRUE.equals(processedIds.hasKey(messageId))) {
            return;  // Skip duplicate
        }

        // Process the message
        Connection conn = dataSource.getConnection();
        try {
            conn.setAutoCommit(false);
            createOrder(conn, message);
            processedIds.opsForValue().set(messageId, "1", 24, TimeUnit.HOURS);  // 24h TTL
            conn.commit();
        } catch (Exception e) {
            conn.rollback();
            throw e;
        } finally {
            conn.close();
        }
    }

    private void createOrder(Connection conn, Message message) {
        // Order creation logic
    }
}

// Strategy 2: Transactional Outbox Pattern
// (See Section 5 for details)
```

### Q: How do you handle message ordering?

```
ORDERING STRATEGIES:

1. SINGLE PARTITION (Strict ordering):
   ┌─────────────────────────────────────────┐
   │  Partition 0                            │
   │  [Msg1] → [Msg2] → [Msg3] → [Msg4]     │
   └─────────────────────────────────────────┘
   - All messages to same partition
   - Limited throughput (single consumer)
   - Use for: Order updates for same order_id

2. PARTITION KEY (Ordered per entity):
   ┌─────────────────────────────────────────┐
   │  Partition 0 (user_id % 3 == 0)        │
   │  [User1-Msg1] → [User1-Msg2]           │
   ├─────────────────────────────────────────┤
   │  Partition 1 (user_id % 3 == 1)        │
   │  [User2-Msg1] → [User2-Msg2]           │
   ├─────────────────────────────────────────┤
   │  Partition 2 (user_id % 3 == 2)        │
   │  [User3-Msg1] → [User3-Msg2]           │
   └─────────────────────────────────────────┘
   - Order maintained per partition key
   - Good throughput (parallel consumers)
   - Use for: User events, order events per order_id

3. SEQUENCE NUMBERS (Application-level):
   Message: {
     "seq": 42,
     "entity_id": "order_123",
     "payload": {...}
   }
   - Consumer tracks last processed seq per entity
   - Reorders out-of-sequence messages
   - Buffer and wait for missing sequences
```

---

## 4. Caching Strategies & Redis

### Q: What caching strategy would you use?

**Caching Patterns Comparison:**

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         CACHING PATTERNS                                 │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  1. CACHE-ASIDE (Lazy Loading):                                         │
│     ┌────────┐    1. Read    ┌────────┐                                 │
│     │ Client │ ────────────► │ Cache  │                                 │
│     └────────┘               └───┬────┘                                 │
│          │                       │ 2. Miss                              │
│          │ 3. Read from DB       ▼                                      │
│          └───────────────► ┌────────┐                                   │
│          ◄──────────────── │   DB   │                                   │
│          │ 4. Return       └────────┘                                   │
│          │                                                               │
│          ▼ 5. Populate cache                                            │
│     ┌────────┐                                                          │
│     │ Cache  │                                                          │
│     └────────┘                                                          │
│                                                                          │
│     Pros: Only caches what's needed, handles cache failures             │
│     Cons: Cache miss = slow, potential stale data                       │
│     Use: General purpose, read-heavy workloads                          │
│                                                                          │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  2. WRITE-THROUGH:                                                       │
│     ┌──────���─┐   1. Write   ┌────────┐  2. Write  ┌────────┐           │
│     │ Client │ ───────────► │ Cache  │ ─────────► │   DB   │           │
│     └────────┘              └────────┘            └────────┘           │
│                                                                          │
│     Pros: Cache always consistent, simple reads                         │
│     Cons: Write latency, cache churn                                    │
│     Use: Data that's read immediately after write                       │
│                                                                          │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  3. WRITE-BEHIND (Write-Back):                                          │
│     ┌────────┐  1. Write  ┌────────┐                                    │
│     │ Client │ ─────────► │ Cache  │  ─ ─ ► (async) ─ ─ ► ┌────────┐  │
│     └────────┘            └────────┘                      │   DB   │   │
│          │                                                └────────┘   │
│          │ 2. ACK (immediate)                                           │
│          ▼                                                               │
│                                                                          │
│     Pros: Very fast writes, batching possible                           │
│     Cons: Data loss risk, complexity                                    │
│     Use: Write-heavy, can tolerate some loss                            │
│                                                                          │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  4. READ-THROUGH:                                                        │
│     ┌────────┐  1. Read  ┌─────────────────┐  2. Load  ┌────────┐      │
│     │ Client │ ────────► │ Cache (smart)   │ ────────► │   DB   │      │
│     └────────┘           │ auto-populates  │           └────────┘      │
│          ▲               └────────┬────────┘                            │
│          │ 3. Return              │                                     │
│          └────────────────────────┘                                     │
│                                                                          │
│     Pros: Simpler application code                                      │
│     Cons: Cache must know how to load data                              │
│     Use: When using cache libraries with loaders                        │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### Q: What if Redis goes down?

**Redis High Availability Setup:**

```
REDIS SENTINEL (Automatic Failover):

┌─────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│    ┌──────────┐      ┌──────────┐      ┌──────────┐                    │
│    │ Sentinel │      │ Sentinel │      │ Sentinel │                    │
│    │    1     │      │    2     │      │    3     │                    │
│    └────┬─────┘      └────┬─────┘      └────┬─────┘                    │
│         │                 │                 │                           │
│         │    Monitoring   │    Monitoring   │                           │
│         ▼                 ▼                 ▼                           │
│    ┌──────────┐      ┌──────────┐      ┌──────────┐                    │
│    │  Redis   │      │  Redis   │      │  Redis   │                    │
│    │ (Master) │─────►│ (Slave)  │      │ (Slave)  │                    │
│    └──────────┘      └──────────┘      └──────────┘                    │
│                                                                          │
│    If Master fails:                                                      │
│    1. Sentinels detect failure (quorum vote)                            │
│    2. Promote a slave to master                                          │
│    3. Reconfigure other slaves                                           │
│    4. Notify clients of new master                                       │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘

REDIS CLUSTER (Sharding + HA):

┌─────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│   Slots 0-5460           Slots 5461-10922        Slots 10923-16383     │
│   ┌──────────┐          ┌──────────┐            ┌──────────┐           │
│   │ Master 1 │          │ Master 2 │            │ Master 3 │           │
│   └────┬─────┘          └────┬─────┘            └────┬─────┘           │
│        │                     │                       │                  │
│        ▼                     ▼                       ▼                  │
│   ┌──────────┐          ┌──────────┐            ┌──────────┐           │
│   │ Slave 1  │          │ Slave 2  │            │ Slave 3  │           │
│   └──────────┘          └──────────┘            └──────────┘           │
│                                                                          │
│   - Data sharded across masters (16384 hash slots)                      │
│   - Each master has replicas for HA                                      │
│   - Automatic failover within each shard                                │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

**Application-level fallback when Redis is down:**

```java
public class CacheWithFallback<T> {
    private RedisTemplate<String, T> redis;
    private Cache<String, T> localCache;  // Caffeine or Guava cache

    public CacheWithFallback(RedisTemplate<String, T> redis) {
        this.redis = redis;
        this.localCache = Caffeine.newBuilder()
            .maximumSize(10000)
            .build();  // In-memory fallback
    }

    public T get(String key) {
        try {
            T value = redis.opsForValue().get(key);
            if (value != null) {
                return value;
            }
        } catch (RedisConnectionFailureException e) {
            // Redis down - try local cache
            return localCache.getIfPresent(key);
        }

        // Cache miss - fetch from source
        return null;
    }

    public T getWithSource(String key, Supplier<T> fetchFunc) {
        // Try cache first
        T cached = get(key);
        if (cached != null) {
            return cached;
        }

        // Fetch from source (DB)
        T value = fetchFunc.get();

        // Try to cache it
        try {
            redis.opsForValue().set(key, value, 3600, TimeUnit.SECONDS);
        } catch (RedisConnectionFailureException e) {
            // Redis down - cache locally
            localCache.put(key, value);
        }

        return value;
    }
}
```

### Q: How do you handle cache invalidation?

**Cache Invalidation Strategies:**

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     CACHE INVALIDATION STRATEGIES                        │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  1. TIME-BASED (TTL):                                                    │
│     cache.setex("user:123", 3600, user_data)  # Expires in 1 hour       │
│                                                                          │
│     Pros: Simple, automatic cleanup                                      │
│     Cons: Stale data until expiry                                        │
│     Use: Data that changes infrequently                                  │
│                                                                          │
│  2. EVENT-BASED (Active Invalidation):                                   │
│     ┌────────┐  Update  ┌────────┐                                      │
│     │  App   │ ───────► │   DB   │                                      │
│     └───┬────┘          └────────┘                                      │
│         │                                                                │
│         │ Invalidate                                                     │
│         ▼                                                                │
│     ┌────────┐                                                          │
│     │ Cache  │  DELETE "user:123"                                       │
│     └────────┘                                                          │
│                                                                          │
│     Pros: Immediate consistency                                          │
│     Cons: Must track all cache keys, complexity                         │
│     Use: Frequently updated, consistency required                        │
│                                                                          │
│  3. VERSION-BASED:                                                       │
│     Key: "user:123:v5"                                                   │
│     On update: Increment version, new key "user:123:v6"                 │
│     Old version expires naturally                                        │
│                                                                          │
│     Pros: No race conditions, atomic updates                             │
│     Cons: Memory overhead for old versions                              │
│                                                                          │
│  4. CACHE-DB TRANSACTION:                                                │
│     try:                                                                 │
│         db.begin()                                                       │
│         db.update(user)                                                  │
│         cache.delete("user:123")                                        │
│         db.commit()                                                      │
│     except:                                                              │
│         db.rollback()                                                    │
│                                                                          │
│     Problem: Cache delete might fail after DB commit!                   │
│     Solution: Use CDC (Change Data Capture) or Outbox pattern           │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### Q: What is Cache Stampede and how do you prevent it?

```
CACHE STAMPEDE PROBLEM:

Time 0: Key "popular_product" expires
        ┌─────────┐
        │ Expired │
        └─────────┘
             │
Time 1: 1000 requests hit simultaneously
        ┌────────┐ ┌────────┐ ┌────────┐     ┌────────┐
        │ Req 1  │ │ Req 2  │ │ Req 3  │ ... │Req 1000│
        └───┬────┘ └───┬────┘ └───┬────┘     └───┬────┘
            │          │          │              │
            ▼          ▼          ▼              ▼
        ┌────────────────────────────────────────────┐
        │              DATABASE                       │
        │         (1000 identical queries!)          │
        │              OVERLOADED                     │
        └────────────────────────────────────────────┘

SOLUTIONS:

1. LOCKING (Mutex):
   def get_product(product_id):
       cached = cache.get(f"product:{product_id}")
       if cached:
           return cached

       # Try to acquire lock
       lock_key = f"lock:product:{product_id}"
       if cache.setnx(lock_key, 1, ex=5):  # Got lock
           try:
               data = db.get_product(product_id)
               cache.setex(f"product:{product_id}", 3600, data)
               return data
           finally:
               cache.delete(lock_key)
       else:
           # Another request is fetching, wait
           time.sleep(0.1)
           return get_product(product_id)  # Retry

2. PROBABILISTIC EARLY EXPIRATION:
   # Refresh before actual expiry
   def get_with_early_refresh(key, ttl=3600):
       cached, remaining_ttl = cache.get_with_ttl(key)
       if cached:
           # Probabilistically refresh when < 10% TTL left
           if remaining_ttl < ttl * 0.1:
               if random.random() < 0.1:  # 10% chance
                   background_refresh(key)
           return cached
       # ... normal fetch logic

3. BACKGROUND REFRESH:
   # Never let cache expire, refresh proactively
   @scheduled(every=55*60)  # Every 55 min (TTL is 60 min)
   def refresh_popular_products():
       for product_id in get_popular_product_ids():
           data = db.get_product(product_id)
           cache.setex(f"product:{product_id}", 3600, data)
```

---

## 5. Service Communication Patterns

### Q: How will your services communicate with each other?

**Communication Patterns:**

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    SERVICE COMMUNICATION PATTERNS                        │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  SYNCHRONOUS (Request-Response):                                         │
│                                                                          │
│  1. REST (HTTP/JSON):                                                    │
│     Service A ──HTTP GET/POST──► Service B                              │
│              ◄──JSON Response────                                        │
│                                                                          │
│     Pros: Simple, widely understood, stateless                          │
│     Cons: Tight coupling, cascading failures, latency                   │
│     Use: Simple CRUD, external APIs, when response needed immediately   │
│                                                                          │
│  2. gRPC (HTTP/2 + Protobuf):                                           │
│     Service A ══Binary══► Service B                                     │
│              ◄══Binary════                                               │
│                                                                          │
│     Pros: 10x faster than REST, streaming, strong typing                │
│     Cons: Binary (not human-readable), requires proto files            │
│     Use: Internal microservices, high-performance, streaming            │
│                                                                          │
│  3. GraphQL:                                                             │
│     Client ──Query──► Gateway ──► Multiple Services                    │
│           ◄─Aggregated─                                                  │
│                                                                          │
│     Pros: Flexible queries, single endpoint, no over-fetching          │
│     Cons: Complexity, caching harder, N+1 queries                       │
│     Use: BFF (Backend for Frontend), mobile apps                        │
│                                                                          │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ASYNCHRONOUS (Event-Driven):                                           │
│                                                                          │
│  1. MESSAGE QUEUE (Point-to-Point):                                     │
│     Service A ──► [Queue] ──► Service B                                 │
│                                                                          │
│     Use: Task distribution, work queues                                  │
│                                                                          │
│  2. PUB/SUB (Fan-out):                                                  │
│     Service A ──► [Topic] ──► Service B                                 │
│                          ├──► Service C                                  │
│                          └──► Service D                                  │
│                                                                          │
│     Use: Event broadcasting, notifications, analytics                   │
│                                                                          │
│  3. EVENT SOURCING:                                                      │
│     All state changes stored as events                                   │
│     [Event1] → [Event2] → [Event3] → Current State                      │
│                                                                          │
│     Use: Audit logs, temporal queries, event replay                     │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### Q: REST vs gRPC - When to use which?

```
COMPARISON:

┌────────────────┬──────────────────────┬──────────────────────┐
│    Aspect      │        REST          │        gRPC          │
├────────────────┼──────────────────────┼──────────────────────┤
│ Protocol       │ HTTP/1.1             │ HTTP/2               │
│ Payload        │ JSON (text)          │ Protobuf (binary)    │
│ Performance    │ Slower               │ 7-10x faster         │
│ Streaming      │ Limited              │ Bidirectional        │
│ Browser        │ Native support       │ Needs grpc-web       │
│ Debugging      │ Easy (curl, browser) │ Harder               │
│ Contract       │ OpenAPI/Swagger      │ .proto files         │
│ Code Gen       │ Optional             │ Built-in             │
└────────────────┴──────────────────────┴──────────────────────┘

DECISION GUIDE:

Choose REST when:
├── Building public APIs
├── Browser clients (web apps)
├── Simple CRUD operations
├── Team not familiar with gRPC
└── Need human-readable debugging

Choose gRPC when:
├── Internal service-to-service communication
├── High-performance requirements
├── Streaming data (real-time updates)
├── Polyglot environment (multiple languages)
└── Strong contract enforcement needed
```

### Q: How do you handle failures in service-to-service communication?

**Circuit Breaker Pattern:**

```
┌─────────────────────────────────────────────────────────────────────────┐
│                       CIRCUIT BREAKER STATES                             │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│   ┌──────────┐         Failure threshold         ┌──────────┐          │
│   │  CLOSED  │ ────────────exceeded───────────► │   OPEN   │          │
│   │ (Normal) │                                   │ (Failing)│          │
│   └────┬─────┘                                   └────┬─────┘          │
│        │                                              │                 │
│        │                                              │ After timeout   │
│        │                                              ▼                 │
│        │                                         ┌───────────┐         │
│        │                                         │ HALF-OPEN │         │
│        │                                         │  (Testing)│         │
│        │                                         └─────┬─────┘         │
│        │                                               │                │
│        │              Success                          │ Failure        │
│        ◄───────────────────────────────────────────────┤                │
│                                                        │                │
│                                                        ▼                │
│                                                   ┌──────────┐         │
│                                                   │   OPEN   │         │
│                                                   └──────────┘         │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘

IMPLEMENTATION:

class CircuitBreaker:
    def __init__(self, failure_threshold=5, timeout=30):
        self.failure_count = 0
        self.failure_threshold = failure_threshold
        self.timeout = timeout
        self.state = "CLOSED"
        self.last_failure_time = None

    def call(self, func, *args, **kwargs):
        if self.state == "OPEN":
            if time.now() - self.last_failure_time > self.timeout:
                self.state = "HALF_OPEN"
            else:
                raise CircuitOpenError("Service unavailable")

        try:
            result = func(*args, **kwargs)
            self.on_success()
            return result
        except Exception as e:
            self.on_failure()
            raise

    def on_success(self):
        self.failure_count = 0
        self.state = "CLOSED"

    def on_failure(self):
        self.failure_count += 1
        self.last_failure_time = time.now()
        if self.failure_count >= self.failure_threshold:
            self.state = "OPEN"
```

### Q: How do you ensure data consistency across services?

**Saga Pattern:**

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        SAGA PATTERN (Choreography)                       │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  Order Placement Saga:                                                   │
│                                                                          │
│  ┌─────────────┐   ┌─────────────┐   ┌─────────────┐   ┌────────────┐  │
│  │   Order     │   │  Payment    │   │  Inventory  │   │  Shipping  │  │
│  │  Service    │   │  Service    │   │  Service    │   │  Service   │  │
│  └──────┬──────┘   └──────┬──────┘   └──────┬──────┘   └─────┬──────┘  │
│         │                 │                 │                 │         │
│    1. Create Order        │                 │                 │         │
│         │                 │                 │                 │         │
│         ├──OrderCreated──►│                 │                 │         │
│         │                 │                 │                 │         │
│         │           2. Process Payment      │                 │         │
│         │                 │                 │                 │         │
│         │                 ├──PaymentDone───►│                 │         │
│         │                 │                 │                 │         │
│         │                 │          3. Reserve Inventory     │         │
│         │                 │                 │                 │         │
│         │                 │                 ├──Inventory─────►│         │
│         │                 │                 │   Reserved      │         │
│         │                 │                 │                 │         │
│         │                 │                 │          4. Ship│         │
│         │                 │                 │                 │         │
│                                                                          │
│  COMPENSATION (If step 3 fails):                                        │
│         │                 │                 │                 │         │
│         │                 │◄─InventoryFail─┤                 │         │
│         │                 │                 │                 │         │
│         │◄──RefundPayment─┤                 │                 │         │
│         │                 │                 │                 │         │
│    Cancel Order           │                 │                 │         │
│         │                 │                 │                 │         │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

**Transactional Outbox Pattern:**

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     TRANSACTIONAL OUTBOX PATTERN                         │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  Problem: How to atomically update DB and publish event?                │
│                                                                          │
│  Wrong approach (can fail between steps):                               │
│    1. Update database                                                    │
│    2. Publish to Kafka  ← What if this fails?                           │
│                                                                          │
│  Solution: Outbox Pattern                                                │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                      SINGLE TRANSACTION                          │   │
│  │  ┌────────────────┐        ┌────────────────────────────────┐   │   │
│  │  │  Orders Table  │        │      Outbox Table              │   │   │
│  │  │                │        │                                │   │   │
│  │  │  INSERT order  │        │  INSERT event                  │   │   │
│  │  │  (id, data...) │        │  (id, topic, payload, status)  │   │   │
│  │  └────────────────┘        └────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                 SEPARATE PROCESS (CDC/Polling)                   │   │
│  │                                                                  │   │
│  │  Outbox Table ───Read pending───► Publisher ───► Kafka          │   │
│  │                                       │                          │   │
│  │                                       └──Mark as published──►    │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│  Benefits:                                                               │
│  ├── Atomic: DB update and event creation in same transaction          │
│  ├── Reliable: Events always published (at-least-once)                 │
│  └── Decoupled: Publishing is separate concern                          │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 6. Concurrency & Locking Mechanisms

### Q: How will you maintain concurrency?

**Concurrency Control Strategies:**

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    CONCURRENCY CONTROL STRATEGIES                        │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  1. OPTIMISTIC LOCKING (Version-based):                                 │
│     ┌─────────────────────────────────────────────────────────────┐    │
│     │  User A reads: {id: 1, balance: 100, version: 5}            │    │
│     │  User B reads: {id: 1, balance: 100, version: 5}            │    │
│     │                                                              │    │
│     │  User A updates: UPDATE SET balance=150, version=6          │    │
│     │                  WHERE id=1 AND version=5  ✓ Success        │    │
│     │                                                              │    │
│     │  User B updates: UPDATE SET balance=80, version=6           │    │
│     │                  WHERE id=1 AND version=5  ✗ Fails!         │    │
│     │                  (version is now 6, not 5)                   │    │
│     └─────────────────────────────────────────────────────────────┘    │
│                                                                          │
│     Use when: Conflicts are rare, read-heavy workloads                  │
│     Example: Wiki page edits, configuration updates                     │
│                                                                          │
│  2. PESSIMISTIC LOCKING (Exclusive lock):                               │
│     ┌─────────────────────────────────────────────────────────────┐    │
│     │  User A: SELECT ... FOR UPDATE  (acquires lock)             │    │
│     │          ... does processing ...                             │    │
│     │          UPDATE ...                                          │    │
│     │          COMMIT (releases lock)                              │    │
│     │                                                              │    │
│     │  User B: SELECT ... FOR UPDATE  (WAITS until A commits)     │    │
│     └─────────────────────────────────────────────────────────────┘    │
│                                                                          │
│     Use when: Conflicts are common, data integrity critical             │
│     Example: Bank transfers, inventory deduction                        │
│                                                                          │
│  3. COMPARE-AND-SWAP (CAS):                                             │
│     ┌─────────────────────────────────────────────────────────────┐    │
│     │  Atomic operation:                                           │    │
│     │  if (current_value == expected_value) {                      │    │
│     │      current_value = new_value;                              │    │
│     │      return true;                                            │    │
│     │  }                                                           │    │
│     │  return false;  // Retry with new expected_value             │    │
│     └─────────────────────────────────────────────────────────────┘    │
│                                                                          │
│     Use when: Lock-free data structures, counters                       │
│     Example: Redis INCR, atomic counters                                │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### Q: How will you acquire a distributed lock?

**Distributed Locking with Redis:**

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     REDIS DISTRIBUTED LOCK (Redlock)                     │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  SIMPLE VERSION (Single Redis):                                          │
│                                                                          │
│  def acquire_lock(lock_name, acquire_timeout=10, lock_timeout=10):      │
│      identifier = str(uuid.uuid4())                                      │
│      end = time.time() + acquire_timeout                                 │
│                                                                          │
│      while time.time() < end:                                            │
│          # SET if Not eXists with expiry                                 │
│          if redis.set(lock_name, identifier, nx=True, ex=lock_timeout): │
│              return identifier                                           │
│          time.sleep(0.001)                                               │
│                                                                          │
│      return None  # Failed to acquire                                    │
│                                                                          │
│  def release_lock(lock_name, identifier):                               │
│      # Lua script for atomic check-and-delete                           │
│      script = """                                                        │
│          if redis.call('get', KEYS[1]) == ARGV[1] then                  │
│              return redis.call('del', KEYS[1])                          │
│          end                                                             │
│          return 0                                                        │
│      """                                                                 │
│      return redis.eval(script, 1, lock_name, identifier)                │
│                                                                          │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  REDLOCK ALGORITHM (Multiple Redis instances):                          │
│                                                                          │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐       │
│  │ Redis 1 │  │ Redis 2 │  │ Redis 3 │  │ Redis 4 │  │ Redis 5 │       │
│  └────┬────┘  └────┬────┘  └────┬────┘  └────┬────┘  └────┬────┘       │
│       │            │            │            │            │             │
│       ▼            ▼            ▼            ▼            ▼             │
│  Try acquire   Try acquire  Try acquire  Try acquire  Try acquire      │
│      ✓            ✓             ✓            ✗            ✓             │
│                                                                          │
│  Result: 4/5 acquired = SUCCESS (majority = 3)                          │
│                                                                          │
│  Steps:                                                                  │
│  1. Get current time in milliseconds                                    │
│  2. Try to acquire lock in all N instances sequentially                 │
│  3. Calculate time elapsed; if < lock validity AND majority acquired:  │
│     Lock is acquired                                                     │
│  4. If lock not acquired, release all instances                         │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### Q: What are the problems with distributed locks?

```
COMMON PROBLEMS AND SOLUTIONS:

┌─────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│  1. LOCK EXPIRY BEFORE COMPLETION:                                      │
│     ┌──────────────────────────────────────────────────────────────┐   │
│     │  Process A acquires lock (TTL: 10s)                          │   │
│     │  Process A takes 15s to complete                             │   │
│     │  Lock expires at 10s                                          │   │
│     │  Process B acquires lock at 11s                              │   │
│     │  Both processes working simultaneously!                       │   │
│     └──────────────────────────────────────────────────────────────┘   │
│                                                                          │
│     Solution: Lock extension (watchdog)                                  │
│     - Background thread extends lock every TTL/3                        │
│     - Stop extending when work complete                                  │
│                                                                          │
│  2. FENCING TOKENS:                                                      │
│     ┌──────────────────────────────────────────────────────────────┐   │
│     │  Lock returns incrementing token: lock + token=34            │   │
│     │  All writes must include token                                │   │
│     │  Storage rejects writes with old tokens                       │   │
│     │                                                                │   │
│     │  Process A: token=33 (expired but still running)             │   │
│     │  Process B: token=34 (current lock holder)                   │   │
│     │  Process A tries to write with token=33 → REJECTED           │   │
│     └──────────────────────────────────────────────────────────────┘   │
│                                                                          │
│  3. CLOCK SKEW:                                                          │
│     - Different servers have different times                            │
│     - Lock appears expired on one server, valid on another              │
│     - Solution: Use logical clocks or synchronized NTP                  │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### Q: Database-level vs Application-level locking?

```
COMPARISON:

┌────────────────────────────────────────────────────────────────────────┐
│                DATABASE LOCKING                                         │
├────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  SELECT * FROM accounts WHERE id = 1 FOR UPDATE;                       │
│  -- Do processing --                                                    │
│  UPDATE accounts SET balance = balance - 100 WHERE id = 1;             │
│  COMMIT;                                                                │
│                                                                         │
│  Pros:                                                                  │
│  ├── Built into DB, no extra infrastructure                            │
│  ├── Transactional (auto-release on commit/rollback)                   │
│  └── Deadlock detection built-in                                       │
│                                                                         │
│  Cons:                                                                  │
│  ├── Connection-bound (loses lock if connection drops)                 │
│  ├── Doesn't work across DBs or services                               │
│  └── Can cause connection pool exhaustion                              │
│                                                                         │
│  Use when: Single database, short transactions                         │
│                                                                         │
├────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│              APPLICATION LOCKING (Redis/ZooKeeper)                     │
├────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  lock = redis.lock("resource:123", timeout=30)                         │
│  if lock.acquire():                                                     │
│      try:                                                               │
│          # Do work across multiple systems                              │
│          update_db()                                                    │
│          call_external_api()                                            │
│          update_cache()                                                 │
│      finally:                                                           │
│          lock.release()                                                 │
│                                                                         │
│  Pros:                                                                  │
│  ├── Works across services and databases                               │
│  ├── Not tied to DB connection                                         │
│  └── TTL auto-releases on crash                                        │
│                                                                         │
│  Cons:                                                                  │
│  ├── Extra infrastructure to manage                                    │
│  ├── Network latency                                                   │
│  └── Must handle lock expiry carefully                                 │
│                                                                         │
│  Use when: Distributed systems, cross-service coordination             │
│                                                                         │
└────────────────────────────────────────────────────────────────────────┘
```

---

## 7. Rate Limiting & Throttling

### Q: How would you implement rate limiting?

**Rate Limiting Algorithms:**

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     RATE LIMITING ALGORITHMS                             │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  1. FIXED WINDOW:                                                        │
│     ┌─────────────────────────────────────────────────────────────┐    │
│     │  Window: 1 minute │  Limit: 100 requests                    │    │
│     │                                                              │    │
│     │  00:00:00 ──────────────────────────── 00:01:00             │    │
│     │     │              100 requests            │                 │    │
│     │     └──────────────────────────────────────┘                 │    │
│     │                                                              │    │
│     │  Problem: 100 requests at 00:00:59 + 100 at 00:01:01        │    │
│     │           = 200 requests in 2 seconds!                       │    │
│     └─────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  2. SLIDING WINDOW LOG:                                                  │
│     ┌─────────────────────────────────────────────────────────────┐    │
│     │  Store timestamp of each request                             │    │
│     │  On new request: count requests in last 60 seconds           │    │
│     │                                                              │    │
│     │  timestamps = [00:00:05, 00:00:10, 00:00:45, 00:00:55]       │    │
│     │  Current time: 00:01:02                                      │    │
│     │  Valid requests: 00:00:10, 00:00:45, 00:00:55 (last 60s)    │    │
│     │  Count: 3 → Allow if < limit                                 │    │
│     │                                                              │    │
│     │  Pros: Precise                                               │    │
│     │  Cons: Memory intensive (stores all timestamps)              │    │
│     └─────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  3. SLIDING WINDOW COUNTER (Recommended):                               │
│     ┌─────────────────────────────────────────────────────────────┐    │
│     │  Combines fixed window efficiency with sliding accuracy      │    │
│     │                                                              │    │
│     │  Previous window: 84 requests                                │    │
│     │  Current window:  36 requests                                │    │
│     │  Current position: 25% into current window                   │    │
│     │                                                              │    │
│     │  Estimated count = 36 + (84 * 0.75) = 36 + 63 = 99          │    │
│     │  Limit: 100 → Allow!                                         │    │
│     └─────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  4. TOKEN BUCKET:                                                        │
│     ┌─────────────────────────────────────────────────────────────┐    │
│     │  Bucket capacity: 100 tokens                                 │    │
│     │  Refill rate: 10 tokens/second                               │    │
│     │                                                              │    │
│     │     ┌───────────────┐                                        │    │
│     │     │  ○ ○ ○ ○ ○    │ ← Tokens                              │    │
│     │     │  ○ ○ ○ ○ ○    │                                        │    │
│     │     └───────┬───────┘                                        │    │
│     │             │                                                │    │
│     │             ▼                                                │    │
│     │        [Request]                                             │    │
│     │                                                              │    │
│     │  Each request consumes 1 token                               │    │
│     │  Tokens refill at constant rate                              │    │
│     │  Allows bursts up to bucket size                             │    │
│     └─────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  5. LEAKY BUCKET:                                                        │
│     ┌─────────────────────────────────────────────────────────────┐    │
│     │  Queue capacity: 100 requests                                │    │
│     │  Processing rate: 10 requests/second (constant)             │    │
│     │                                                              │    │
│     │     ┌───────────────┐                                        │    │
│     │     │  req req req  │ ← Incoming requests                   │    │
│     │     │  req req req  │                                        │    │
│     │     └───────┬───────┘                                        │    │
│     │             │ (constant rate)                                │    │
│     │             ▼                                                │    │
│     │        [Processed]                                           │    │
│     │                                                              │    │
│     │  Smooths out bursts                                          │    │
│     │  Constant output rate                                        │    │
│     └─────────────────────────────────────────────────────────────┘    │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

**Redis Implementation of Token Bucket:**

```java
// Token Bucket Rate Limiter using Redis

public class TokenBucketRateLimiter {
    private final RedisTemplate<String, String> redis;
    private final DefaultRedisScript<Long> luaScript;

    // Lua script for atomic token bucket
    private static final String LUA_SCRIPT = """
        local key = KEYS[1]
        local capacity = tonumber(ARGV[1])
        local refill_rate = tonumber(ARGV[2])
        local now = tonumber(ARGV[3])
        local requested = tonumber(ARGV[4])

        local bucket = redis.call('HMGET', key, 'tokens', 'last_refill')
        local tokens = tonumber(bucket[1]) or capacity
        local last_refill = tonumber(bucket[2]) or now

        -- Calculate tokens to add based on time passed
        local elapsed = now - last_refill
        local new_tokens = math.min(capacity, tokens + (elapsed * refill_rate))

        if new_tokens >= requested then
            -- Consume token(s)
            redis.call('HSET', key, 'tokens', new_tokens - requested, 'last_refill', now)
            redis.call('EXPIRE', key, 3600)  -- Cleanup after 1 hour
            return 1  -- Allowed
        else
            -- Update bucket state but deny request
            redis.call('HSET', key, 'tokens', new_tokens, 'last_refill', now)
            return 0  -- Denied
        end
    """;

    public TokenBucketRateLimiter(RedisTemplate<String, String> redis) {
        this.redis = redis;
        this.luaScript = new DefaultRedisScript<>();
        this.luaScript.setScriptText(LUA_SCRIPT);
        this.luaScript.setResultType(Long.class);
    }

    /**
     * @param userId     User identifier
     * @param capacity   Max tokens in bucket (default: 100)
     * @param refillRate Tokens added per second (default: 10)
     * @return true if request is allowed, false if rate limited
     */
    public boolean isAllowed(String userId, int capacity, int refillRate) {
        String key = "rate_limit:" + userId;
        double now = System.currentTimeMillis() / 1000.0;

        Long result = redis.execute(
            luaScript,
            Collections.singletonList(key),
            String.valueOf(capacity),
            String.valueOf(refillRate),
            String.valueOf(now),
            "1"  // requested tokens
        );

        return result != null && result == 1L;
    }

    public boolean isAllowed(String userId) {
        return isAllowed(userId, 100, 10);  // defaults
    }
}
```

### Q: Where should rate limiting be applied?

```
RATE LIMITING LAYERS:

┌─────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│   Layer 1: CDN/Edge (Cloudflare, AWS CloudFront)                        │
│   ├── DDoS protection                                                    │
│   ├── Geographic rate limiting                                           │
│   └── IP-based blocking                                                  │
│                                                                          │
│   Layer 2: API Gateway (Kong, AWS API Gateway)                          │
│   ├── Per-API key limits                                                 │
│   ├── Per-endpoint limits                                                │
│   └── Request throttling                                                 │
│                                                                          │
│   Layer 3: Load Balancer (Nginx, HAProxy)                               │
│   ├── Connection limits                                                  │
│   └── Request rate limits                                                │
│                                                                          │
│   Layer 4: Application (Service code)                                   │
│   ├── Business logic limits (orders per hour)                           │
│   ├── User-specific quotas                                               │
│   └── Resource-specific limits                                           │
│                                                                          │
│   Layer 5: Database                                                      │
│   ├── Connection pool limits                                             │
│   └── Query timeouts                                                     │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘

DECISION GUIDE:

┌──────────────────────┬──────────────────────────────────────────────────┐
│ Rate Limit Type      │ Where to Apply                                   │
├──────────────────────┼──────────────────────────────────────────────────┤
│ DDoS protection      │ CDN/Edge                                         │
│ API abuse prevention │ API Gateway                                      │
│ Resource protection  │ Load Balancer + Application                      │
│ Business rules       │ Application only                                 │
│ Expensive operations │ Application with distributed counter            │
└──────────────────────┴──────────────────────────────────────────────────┘
```

---

## 8. Load Balancing & Scaling

### Q: How would you scale this system?

**Scaling Strategies:**

```
┌─────────────────────────────────────────────────────────────────────────┐
│                       SCALING STRATEGIES                                 │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  VERTICAL SCALING (Scale Up):                                           │
│  ┌────────────┐         ┌────────────────────┐                         │
│  │   2 CPU    │   ──►   │      16 CPU        │                         │
│  │   4 GB RAM │         │      64 GB RAM     │                         │
│  └────────────┘         └────────────────────┘                         │
│                                                                          │
│  Pros: Simple, no code changes                                          │
│  Cons: Hardware limits, single point of failure, expensive              │
│  Use: Quick fix, development, DBs before sharding                       │
│                                                                          │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  HORIZONTAL SCALING (Scale Out):                                        │
│  ┌────────────┐         ┌────────────┐ ┌────────────┐ ┌────────────┐  │
│  │  Server 1  │   ──►   │  Server 1  │ │  Server 2  │ │  Server 3  │  │
│  └────────────┘         └────────────┘ └────────────┘ └────────────┘  │
│                                  │            │            │            │
│                                  └────────────┴────────────┘            │
│                                              │                          │
│                                       ┌──────────────┐                  │
│                                       │ Load Balancer│                  │
│                                       └──────────────┘                  │
│                                                                          │
│  Pros: No hardware limit, fault tolerant, cost-effective               │
│  Cons: Complexity, state management, network overhead                   │
│  Use: Stateless services, microservices                                 │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### Q: What load balancing algorithm would you use?

```
LOAD BALANCING ALGORITHMS:

┌─────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│  1. ROUND ROBIN:                                                         │
│     Request 1 → Server A                                                │
│     Request 2 → Server B                                                │
│     Request 3 → Server C                                                │
│     Request 4 → Server A (repeat)                                       │
│                                                                          │
│     Use: Homogeneous servers, stateless requests                        │
│     Problem: Doesn't consider server load                               │
│                                                                          │
│  2. WEIGHTED ROUND ROBIN:                                               │
│     Server A (weight: 3): Gets 3x more requests                        │
│     Server B (weight: 2): Gets 2x more requests                        │
│     Server C (weight: 1): Gets 1x more requests                        │
│                                                                          │
│     Use: Heterogeneous servers (different capacities)                   │
│                                                                          │
│  3. LEAST CONNECTIONS:                                                   │
│     Server A: 10 active connections                                     │
│     Server B: 5 active connections  ← Route here                       │
│     Server C: 15 active connections                                     │
│                                                                          │
│     Use: Long-lived connections, varying request durations              │
│                                                                          │
│  4. LEAST RESPONSE TIME:                                                │
│     Server A: avg 50ms                                                  │
│     Server B: avg 30ms  ← Route here                                   │
│     Server C: avg 80ms                                                  │
│                                                                          │
│     Use: Latency-sensitive applications                                 │
│                                                                          │
│  5. IP HASH (Sticky Sessions):                                          │
│     hash(client_ip) % num_servers = target_server                      │
│     Same client always goes to same server                              │
│                                                                          │
│     Use: Session affinity needed, stateful applications                 │
│     Problem: Uneven distribution if some IPs are very active           │
│                                                                          │
│  6. CONSISTENT HASHING:                                                  │
│     Hash ring distributes requests                                      │
│     Minimal redistribution when servers added/removed                   │
│                                                                          │
│     Use: Caching layers, distributed databases                          │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### Q: How do you handle stateful services?

```
STATEFUL SERVICE STRATEGIES:

┌─────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│  1. EXTERNALIZE STATE:                                                   │
│     ┌─────────────────────────────────────────────────────────────┐    │
│     │                                                              │    │
│     │    ┌──────────┐  ┌──────────┐  ┌──────────┐               │    │
│     │    │ Server 1 │  │ Server 2 │  │ Server 3 │               │    │
│     │    │(Stateless)│  │(Stateless)│  │(Stateless)│              │    │
│     │    └────┬─────┘  └────┬─────┘  └────┬─────┘               │    │
│     │         │             │             │                       │    │
│     │         └─────────────┼─────────────┘                       │    │
│     │                       │                                      │    │
│     │                ┌──────────────┐                              │    │
│     │                │    Redis     │  ← Shared session store     │    │
│     │                │ (State Store)│                              │    │
│     │                └──────────────┘                              │    │
│     │                                                              │    │
│     └─────────────────────────────────────────────────────────────┘    │
│                                                                          │
│     Best approach: Makes services horizontally scalable                 │
│                                                                          │
│  2. STICKY SESSIONS:                                                     │
│     ┌─────────────────────────────────────────────────────────────┐    │
│     │                                                              │    │
│     │    ┌────────────────────────────────────────┐               │    │
│     │    │           Load Balancer                │               │    │
│     │    │   (Cookie: server_id=server1)          │               │    │
│     │    └────────────────┬───────────────────────┘               │    │
│     │                     │                                        │    │
│     │         Always route user X to Server 1                     │    │
│     │                     ▼                                        │    │
│     │    ┌──────────┐  ┌──────────┐  ┌──────────┐               │    │
│     │    │ Server 1 │  │ Server 2 │  │ Server 3 │               │    │
│     │    │(has state)│  │          │  │          │               │    │
│     │    └──────────┘  └──────────┘  └──────────┘               │    │
│     │                                                              │    │
│     └─────────────────────────────────────────────────────────────┘    │
│                                                                          │
│     Problem: Uneven load, server death loses sessions                   │
│                                                                          │
│  3. REPLICATED STATE:                                                    │
│     Each server has full copy of state (memory expensive)              │
│     Use: Small state, high read requirements                            │
│                                                                          │
└───────────────────────────────────────────────���─────────────────────────┘
```

### Q: How do you scale the database?

```
DATABASE SCALING STRATEGIES:

┌─────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│  1. READ REPLICAS:                                                       │
│     ┌──────────┐    Writes    ┌──────────┐                             │
│     │  App     │ ───────────► │  Primary │                             │
│     │  Server  │              │    DB    │                             │
│     │          │◄───────────  └────┬─────┘                             │
│     └──────────┘    Reads          │ Replication                       │
│          │                         ▼                                    │
│          │              ┌──────────┬──────────┐                        │
│          └─────────────►│ Replica  │ Replica  │                        │
│               Reads     │    1     │    2     │                        │
│                         └──────────┴──────────┘                        │
│                                                                          │
│     Use: Read-heavy workloads (90% reads)                               │
│     Limitation: Write bottleneck on primary                             │
│                                                                          │
│  2. SHARDING (Horizontal Partitioning):                                 │
│     ┌──────────────────────────────────────────────────────────────┐   │
│     │                                                               │   │
│     │   Users A-H        Users I-P        Users Q-Z                │   │
│     │   ┌────────┐       ┌────────┐       ┌────────┐               │   │
│     │   │ Shard 1│       │ Shard 2│       │ Shard 3│               │   │
│     │   └────────┘       └────────┘       └────────┘               │   │
│     │                                                               │   │
│     └──────────────────────────────────────────────────────────────┘   │
│                                                                          │
│     Sharding Strategies:                                                 │
│     ├── Range-based: user_id 1-1M, 1M-2M, etc.                         │
│     ├── Hash-based: hash(user_id) % num_shards                         │
│     ├── Directory-based: Lookup table for routing                      │
│     └── Geography-based: US users in US shard                          │
│                                                                          │
│     Challenges:                                                          │
│     ├── Cross-shard queries (joins)                                    │
│     ├── Rebalancing when adding shards                                 │
│     └── Transactions across shards                                      │
│                                                                          │
│  3. CQRS (Command Query Responsibility Segregation):                    │
│     ┌──────────────────────────────────────────────────────────────┐   │
│     │                                                               │   │
│     │   Commands (Writes)              Queries (Reads)             │   │
│     │        │                              │                       │   │
│     │        ▼                              ▼                       │   │
│     │   ┌──────────┐    Sync/Async    ┌──────────┐                │   │
│     │   │  Write   │ ───────────────► │   Read   │                │   │
│     │   │   DB     │   (eventually)   │   DB     │                │   │
│     │   │(Normalized)                 │(Denormalized)│             │   │
│     │   └──────────┘                  └──────────┘                │   │
│     │                                                               │   │
│     └──────────────────────────────────────────────────────────────┘   │
│                                                                          │
│     Use: Different read/write patterns, complex queries                 │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 9. Data Consistency & CAP Theorem

### Q: Explain CAP theorem and how you'd apply it

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           CAP THEOREM                                    │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  You can only guarantee 2 out of 3:                                     │
│                                                                          │
│                        Consistency                                       │
│                            ▲                                             │
│                           / \                                            │
│                          /   \                                           │
│                         /     \                                          │
│                        / CA    \  CP                                    │
│                       /         \                                        │
│                      /           \                                       │
│       Availability ◄─────────────► Partition Tolerance                 │
│                          AP                                              │
│                                                                          │
│  In distributed systems, partitions WILL happen.                        │
│  So you really choose between:                                          │
│  ├── CP: Consistency + Partition tolerance (sacrifice Availability)    │
│  └── AP: Availability + Partition tolerance (sacrifice Consistency)    │
│                                                                          │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  REAL-WORLD EXAMPLES:                                                    │
│                                                                          │
│  CP Systems (Prefer consistency):                                        │
│  ├── Banking systems (balance must be accurate)                         │
│  ├── Booking systems (no double-booking)                                │
│  ├── MongoDB (in certain configs)                                       │
│  └── ZooKeeper, etcd                                                    │
│                                                                          │
│  AP Systems (Prefer availability):                                       │
│  ├── Social media feeds (ok if slightly stale)                          │
│  ├── Product catalog (eventual consistency ok)                          │
│  ├── Cassandra, DynamoDB                                                │
│  └── DNS                                                                │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### Q: Strong vs Eventual Consistency - When to use which?

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    CONSISTENCY LEVELS                                    │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  STRONG CONSISTENCY:                                                     │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │  Write ──► Primary ──► All Replicas (synchronous)               │  │
│  │                   │                                              │  │
│  │                   └──► ACK to client (after all replicas ACK)   │  │
│  │                                                                  │  │
│  │  Read from any node returns latest write                         │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                                                                          │
│  Use when:                                                               │
│  ├── Financial transactions (account balance)                           │
│  ├── Inventory management (stock count)                                 │
│  ├── Unique constraint enforcement (usernames)                          │
│  └── Booking/reservation systems                                        │
│                                                                          │
│  Trade-off: Higher latency, lower availability                          │
│                                                                          │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  EVENTUAL CONSISTENCY:                                                   │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │  Write ──► Primary ──► ACK to client (immediate)                │  │
│  │                   │                                              │  │
│  │                   └──► Replicas (asynchronous, eventually)      │  │
│  │                                                                  │  │
│  │  Read might return stale data temporarily                        │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                                                                          │
│  Use when:                                                               │
│  ├── Social media (like counts, view counts)                           │
│  ├── User activity feeds                                                │
│  ├── Shopping cart (before checkout)                                    │
│  └── Analytics data                                                     │
│                                                                          │
│  Trade-off: Lower latency, higher availability, but stale reads         │
│                                                                          │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  READ-YOUR-WRITES CONSISTENCY:                                          │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │  User writes data                                                │  │
│  │  User's subsequent reads guaranteed to see their own writes     │  │
│  │  Other users might see stale data                               │  │
│  │                                                                  │  │
│  │  Implementation:                                                 │  │
│  │  ├── Route user's reads to primary after their write           │  │
│  │  ├── Include write timestamp, read from replica if caught up   │  │
│  │  └── Cache user's recent writes client-side                    │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                                                                          │
│  Use when: User should see their own actions immediately                │
│  Example: User updates profile, should see changes immediately          │
│                                                                          │
└─────────────────��───────────────────────────────────────────────────────┘
```

---

## 10. Monitoring, Logging & Alerting

### Q: How would you monitor this system?

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    OBSERVABILITY PILLARS                                 │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  1. METRICS (What is happening):                                        │
│     ├── System: CPU, Memory, Disk, Network                             │
│     ├── Application: Request rate, Error rate, Latency (RED)           │
│     ├── Business: Orders/min, Revenue, User signups                    │
│     └── Tools: Prometheus, Grafana, DataDog, CloudWatch                │
│                                                                          │
│  2. LOGS (Why it happened):                                             │
│     ├── Structured logging (JSON format)                                │
│     ├── Log levels: DEBUG, INFO, WARN, ERROR                           │
│     ├── Correlation IDs for tracing                                     │
│     └── Tools: ELK Stack, Splunk, Loki                                  │
│                                                                          │
│  3. TRACES (Where it happened):                                         │
│     ├── Distributed tracing across services                             │
│     ├── Span visualization                                              │
│     └── Tools: Jaeger, Zipkin, X-Ray, Tempo                            │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘

KEY METRICS TO MONITOR (RED + USE Methods):

┌─────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│  RED Method (Request-driven services):                                  │
│  ├── Rate: Requests per second                                          │
│  ├── Errors: Failed requests per second                                 │
│  └── Duration: Latency distribution (p50, p95, p99)                    │
│                                                                          │
│  USE Method (Resources):                                                 │
│  ├── Utilization: % time resource is busy                              │
│  ├── Saturation: Amount of work queued                                  │
│  └── Errors: Count of error events                                      │
│                                                                          │
│  Four Golden Signals (Google SRE):                                      │
│  ├── Latency: Time to service a request                                │
│  ├── Traffic: Demand on system                                          │
│  ├── Errors: Rate of failed requests                                    │
│  └── Saturation: How "full" the service is                             │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### Q: What alerting strategy would you use?

```
ALERTING BEST PRACTICES:

┌─────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│  1. ALERT ON SYMPTOMS, NOT CAUSES:                                      │
│     ✗ Bad: "CPU > 90%"                                                  │
│     ✓ Good: "Request latency p99 > 500ms"                              │
│                                                                          │
│     ✗ Bad: "Disk usage > 80%"                                          │
│     ✓ Good: "Disk will be full in 4 hours" (predictive)                │
│                                                                          │
│  2. SEVERITY LEVELS:                                                     │
│     ┌────────────────────────────────────────────────────────────┐     │
│     │  P1 (Critical): Service down, data loss risk               │     │
│     │      → Page on-call immediately                            │     │
│     │                                                             │     │
│     │  P2 (High): Degraded performance, affecting users          │     │
│     │      → Page during business hours                          │     │
│     │                                                             │     │
│     │  P3 (Medium): Potential issues, no immediate impact        │     │
│     │      → Ticket, fix within 24h                              │     │
│     │                                                             │     │
│     │  P4 (Low): Minor issues, cosmetic                          │     │
│     │      → Ticket, fix when convenient                         │     │
│     └────────────────────────────────────────────────────────────┘     │
│                                                                          │
│  3. REDUCE ALERT FATIGUE:                                               │
│     ├── Every alert should be actionable                               │
│     ├── Group related alerts                                            │
│     ├── Use proper thresholds (not too sensitive)                      │
│     └── Implement alert suppression during maintenance                  │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 11. Common HLD Scenarios with Complete Solutions

### Scenario 1: Design a URL Shortener

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     URL SHORTENER ARCHITECTURE                           │
├────────────��────────────────────────────────────────────────────────────┤
│                                                                          │
│  Requirements:                                                           │
│  ├── Shorten long URLs                                                  │
│  ├── Redirect short URLs to original                                    │
│  ├── 100M URLs created per day                                          │
│  └── 10:1 read:write ratio                                              │
│                                                                          │
│  ARCHITECTURE:                                                           │
│                                                                          │
│   ┌─────────────────────────────────────────────────────────────────┐  │
│   │                         Clients                                  │  │
│   └────────────────────────────┬────────────────────────────────────┘  │
│                                │                                        │
│                                ▼                                        │
│   ┌─────────────────────────────────────────────────────────────────┐  │
│   │                      Load Balancer                               │  │
│   └────────────────────────────┬────────────────────────────────────┘  │
│                                │                                        │
│              ┌─────────────────┼─────────────────┐                     │
│              ▼                 ▼                 ▼                      │
│   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐                │
│   │  App Server  │  │  App Server  │  │  App Server  │                │
│   └──────┬───────┘  └──────┬───────┘  └──────┬───────┘                │
│          │                 │                 │                          │
│          └─────────────────┼─────────────────┘                          │
│                            │                                            │
│          ┌─────────────────┼─────────────────┐                         │
│          ▼                 ▼                 ▼                          │
│   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐                │
│   │    Redis     │  │   Postgres   │  │     Kafka    │                │
│   │   (Cache)    │  │  (Primary)   │  │ (Analytics)  │                │
│   └──────────────┘  └──────────────┘  └──────────────┘                │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘

FOLLOW-UP QUESTIONS & ANSWERS:

Q: How do you generate the short URL?
A: Base62 encoding of an auto-increment ID from a distributed ID generator
   (like Twitter Snowflake). 7 characters = 62^7 = 3.5 trillion URLs.

Q: Why not use hash (MD5/SHA)?
A: Hash collision handling adds complexity. Base62 of unique ID is simpler
   and guaranteed unique.

Q: How do you handle high read traffic?
A:
   1. CDN at edge for static redirects
   2. Redis cache (LRU) for hot URLs
   3. Read replicas for DB

Q: What if Redis goes down?
A:
   - Fallback to DB read replicas
   - Local in-memory cache as L1
   - Graceful degradation (slightly higher latency)

Q: How do you handle analytics?
A:
   - Async: Publish click events to Kafka
   - Separate analytics service consumes and aggregates
   - Store in time-series DB (InfluxDB/TimescaleDB)
```

### Scenario 2: Design a Rate Limiter Service

```
┌─────────────────────────────────────────────────────────────────────────┐
│                   RATE LIMITER SERVICE DESIGN                            │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  Requirements:                                                           │
│  ├── Distributed rate limiting                                          │
│  ├── Multiple rate limit rules (per user, API, IP)                     │
│  ├── Low latency (< 10ms overhead)                                     │
│  └── High availability                                                  │
│                                                                          │
│  ARCHITECTURE:                                                           │
│                                                                          │
│   ┌────────────────────────────────────────────────────────────────┐   │
│   │                           Client                                │   │
│   └────────────────────────────┬───────────────────────────────────┘   │
│                                │                                        │
│                                ▼                                        │
│   ┌────────────────────────────────────────────────────────────────┐   │
│   │                      API Gateway                                │   │
│   │              (First line rate limiting)                         │   │
│   └────────────────────────────┬───────────────────────────────────┘   │
│                                │                                        │
│                                ▼                                        │
│   ┌────────────────────────────────────────────────────────────────┐   │
│   │                    Rate Limiter Service                         │   │
│   │   ┌──────────────────────────────────────────────────────┐    │   │
│   │   │  Rules Engine                                         │    │   │
│   │   │  ├── User limits: 1000 req/hour                      │    │   │
│   │   │  ├── API limits: 100 req/min per endpoint            │    │   │
│   │   │  └── IP limits: 10 req/sec                           │    │   │
│   │   └──────────────────────────────────────────────────────┘    │   │
│   └────────────────────────────┬───────────────────────────────────┘   │
│                                │                                        │
│                                ▼                                        │
│   ┌────────────────────────────────────────────────────────────────┐   │
│   │                    Redis Cluster                                │   │
│   │              (Distributed counters)                             │   │
│   │   ┌──────────┐  ┌──────────┐  ┌──────────┐                    │   │
│   │   │ Master 1 │  │ Master 2 │  │ Master 3 │                    │   │
│   │   │ Replica  │  │ Replica  │  │ Replica  │                    │   │
│   │   └──────────┘  └──────────┘  └──────────┘                    │   │
│   └────────────────────────────────────────────────────────────────┘   │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘

FOLLOW-UP QUESTIONS & ANSWERS:

Q: Which algorithm do you use?
A: Token Bucket for most cases:
   - Allows bursts up to bucket size
   - Smooth rate limiting
   - Easy to implement with Redis

Q: How do you handle Redis failure?
A: Layered approach:
   1. Local in-memory fallback (per-server limits)
   2. Fail-open with logging (allow traffic, alert)
   3. Circuit breaker to detect and recover

Q: How do you sync rate limits across servers?
A: Centralized state in Redis. Each request:
   1. INCR counter atomically
   2. SET TTL for window
   3. Return current count

   Alternative: Sliding window with Lua script for atomic operation.

Q: What about race conditions?
A: Redis operations are atomic. Use Lua scripts for complex
   operations (check + increment in one round trip).

Q: How do you handle configuration updates?
A:
   - Store rules in separate config DB/service
   - Push updates to all rate limiter instances
   - Apply new rules on next request (no restart)
```

### Scenario 3: Design an Order Processing System

```
┌─────────────────────────────────────────────────────────────────────────┐
│                  ORDER PROCESSING SYSTEM DESIGN                          │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  Requirements:                                                           │
│  ├── Handle 10K orders/minute at peak                                   │
│  ├── No duplicate orders                                                │
│  ├── Guaranteed processing (no lost orders)                            │
│  └── Integration with payment, inventory, shipping                     │
│                                                                          │
│  ARCHITECTURE (Event-Driven with Saga):                                 │
│                                                                          │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │                                                                   │  │
│  │  ┌───────┐    ┌─────────────┐    ┌─────────────────────────┐   │  │
│  │  │Client │───►│ Order API   │───►│      Order DB           │   │  │
│  │  └───────┘    │ Service     │    │  (Orders + Outbox)      │   │  │
│  │               └──────┬──────┘    └─────────────────────────┘   │  │
│  │                      │                                          │  │
│  │                      │ Outbox Polling / CDC                     │  │
│  │                      ▼                                          │  │
│  │  ┌────────────────────────────────────────────────────────────┐│  │
│  │  │                       KAFKA                                 ││  │
│  │  │  ┌───────────────┐  ┌───────────────┐  ┌───────────────┐  ││  │
│  │  │  │ order.created │  │ payment.done  │  │ order.shipped │  ││  │
│  │  │  └───────────────┘  └───────────────┘  └───────────────┘  ││  │
│  │  └─────────┬────────────────┬────────────────────┬───────────┘│  │
│  │            │                │                    │             │  │
│  │            ▼                ▼                    ▼             │  │
│  │  ┌─────────────────┐ ┌───────────────┐ ┌─────────────────┐   │  │
│  │  │ Payment Service │ │   Inventory   │ │Shipping Service │   │  │
│  │  │                 │ │   Service     │ │                 │   │  │
│  │  └─────────────────┘ └───────────────┘ └─────────────────┘   │  │
│  │                                                               │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                                                                          │
│  SAGA FLOW:                                                              │
│                                                                          │
│  ┌──────────┐   ┌───────────┐   ┌────────────┐   ┌─────────────┐      │
│  │  Create  │──►│  Process  │──►│  Reserve   │──►│  Ship       │      │
│  │  Order   │   │  Payment  │   │  Inventory │   │  Order      │      │
│  └──────────┘   └───────────┘   └────────────┘   └─────────────┘      │
│       │              │                │                │               │
│       │              │                │                │               │
│       ▼              ▼                ▼                ▼               │
│  On Failure:    Refund          Release           Cancel              │
│  (Compensating  Payment         Inventory         Shipment            │
│   Actions)                                                             │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘

FOLLOW-UP QUESTIONS & ANSWERS:

Q: How do you prevent duplicate orders?
A: Idempotency key:
   - Client generates unique order_id
   - Server stores processed IDs in Redis (TTL 24h)
   - Reject if ID already processed

Q: What if Kafka is down?
A: Outbox pattern ensures no message loss:
   - Order + outbox event in same DB transaction
   - Separate process reads outbox, publishes to Kafka
   - If Kafka down, messages queue in outbox
   - Retry with exponential backoff

Q: How do you handle partial failures (e.g., payment done, inventory fail)?
A: Saga with compensating transactions:
   - Payment done → Inventory fails
   - Publish "inventory.failed" event
   - Payment service listens, triggers refund
   - Order service marks order as cancelled

Q: How do you ensure order processing order?
A: Kafka partition key = order_id
   - All events for same order go to same partition
   - Single consumer per partition = ordered processing

Q: What database for orders?
A: PostgreSQL with:
   - ACID transactions for order creation
   - Outbox table for reliable messaging
   - Indexes on user_id, created_at, status
   - Partitioning by date for scaling

Q: How do you handle high throughput?
A:
   - Horizontal scaling of API servers (stateless)
   - Kafka partitions = parallel processing
   - Database read replicas for queries
   - Caching for order status lookups
```

---

## Quick Reference: Common Interview Responses

### When asked "What if X is down?"

```
Generic Fallback Framework:

1. IDENTIFY the failure mode (partial/complete outage)

2. IMMEDIATE MITIGATION:
   ├── Circuit breaker to prevent cascade
   ├── Retry with exponential backoff
   └── Fallback to secondary (replica/cache)

3. GRACEFUL DEGRADATION:
   ├── Return cached/stale data if acceptable
   ├── Queue operations for later processing
   └── Return partial results with indication

4. RECOVERY:
   ├── Automatic failover (if configured)
   ├── Process queued operations
   └── Reconcile any conflicts
```

### When asked "Why X over Y?"

```
Framework for Comparison:

1. STATE YOUR REQUIREMENTS:
   "Given our need for [specific requirement]..."

2. COMPARE ON RELEVANT DIMENSIONS:
   ├── Performance (throughput, latency)
   ├── Consistency guarantees
   ├── Operational complexity
   ├── Cost (infra + engineering)
   └── Team expertise

3. ACKNOWLEDGE TRADE-OFFS:
   "We sacrifice [downside] but gain [benefit]"

4. MENTION ALTERNATIVES:
   "If requirements change to X, we might reconsider Y"
```

### Key Numbers to Remember

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     LATENCY NUMBERS                                      │
├─────────────────────────────────────────────────────────────────────────┤
│  L1 cache reference                   0.5 ns                            │
│  L2 cache reference                     7 ns                            │
│  Main memory reference                100 ns                            │
│  SSD random read               150,000 ns (150 µs)                     │
│  HDD seek                    10,000,000 ns (10 ms)                     │
│  Network round trip (same DC)        500,000 ns (0.5 ms)               │
│  Network round trip (cross DC)    150,000,000 ns (150 ms)              │
├─────────────────────────────────────────────────────────────────────────┤
│                     THROUGHPUT NUMBERS                                   │
├─────────────────────────────────────────────────────────────────────────┤
│  Redis: 100K+ ops/sec                                                   │
│  Kafka: 1M+ messages/sec per broker                                    │
│  PostgreSQL: 10K-50K transactions/sec                                  │
│  MongoDB: 100K+ ops/sec                                                │
│  Nginx: 50K+ concurrent connections                                    │
├─────────────────────────────────────────────────────────────────────────┤
│                     STORAGE ESTIMATES                                    │
├─────────────────────────────────────────────────────────────────────────┤
│  1 Million users × 1KB each = 1 GB                                     │
│  1 Billion rows × 100 bytes = 100 GB                                   │
│  1 day of logs (1K servers × 100 lines/sec) = ~1 TB                   │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Interview Tips

1. **Always clarify requirements first** - Ask about scale, consistency needs, and constraints

2. **Start with high-level design** - Draw the big boxes before diving into details

3. **Justify every decision** - "I chose X because of Y requirement"

4. **Proactively discuss trade-offs** - Shows senior thinking

5. **Think about failure modes** - What happens when things break?

6. **Consider operational aspects** - How do you deploy, monitor, debug?

7. **Know your numbers** - Rough estimates for capacity planning

8. **Practice drawing diagrams** - Clear communication matters

---

## 12. Session Q&A Notes

### Q1: How does PostGIS work? How is lat/long indexed? Does it use GeoHash?

**Answer:** PostGIS uses **R-Tree indexes (via GiST)**, NOT GeoHash by default.

**R-Tree vs GeoHash:**

| Aspect | R-Tree (PostGIS) | GeoHash |
|--------|------------------|---------|
| Structure | Hierarchical bounding boxes | String encoding ("tdr1y") |
| Nearest neighbor | Excellent (native) | Needs workarounds |
| Use case | Complex spatial queries | Redis, DynamoDB, simple proximity |

**Example:**
```sql
CREATE INDEX idx_location ON restaurants USING GIST(location);

SELECT name FROM restaurants
WHERE ST_DWithin(location, ST_MakePoint(77.59, 12.97)::geography, 5000);
```

---

### Q2: How does R-Tree work? How does it handle points in different boxes but close to user?

**Core Concept:** R-Tree groups nearby points into bounding boxes, then groups boxes into larger boxes (hierarchical).

**The Edge Case Problem:**
```
┌─────────────────┬─────────────────┐
│     Box A       │      Box B      │
│                 │  ☕ Coffee Shop │
│         📍 USER │  (in Box B but  │
│    (in Box A)   │   CLOSER!)      │
└─────────────────┴─────────────────┘
```

**Solution:** R-Tree searches ALL boxes that OVERLAP with search radius, not just user's box:

```
┌─────────────────┬─────────────────┐
│     Box A       │      Box B      │
│            ┌────┼────┐            │
│         📍─│─search──│ ☕ ← FOUND!│
│            │ circle  │            │
│            └────┼────┘            │
└─────────────────┴─────────────────┘
Search circle overlaps BOTH boxes → checks BOTH → finds ☕
```

**Recursion Steps:**
```
search(node, user, radius):
  if node is leaf:
      return points where distance(point, user) <= radius
  else:
      results = []
      for each child_box in node:
          if circle_overlaps_box(user, radius, child_box):
              results += search(child_box, user, radius)  ← RECURSIVE
      return results
```

**Overlap Check:**
```python
def circle_overlaps_box(user, radius, box):
    closest_x = clamp(user.x, box.min_x, box.max_x)
    closest_y = clamp(user.y, box.min_y, box.max_y)
    distance = sqrt((user.x - closest_x)² + (user.y - closest_y)²)
    return distance <= radius
```

**Interview one-liner:** *"R-Tree searches all bounding boxes overlapping the query circle, not just the user's box. This handles edge cases where nearby points are in adjacent boxes. Time complexity is O(log n) average."*

---

### Q3: SQL vs Graph DB for Many-to-Many (Friends of Friends)

**The Schema (Many-to-Many in SQL):**
```
users:                    friendships (junction table):
┌─────┬─────────┐        ┌─────────┬───────────┐
│ id  │ name    │        │ user_id │ friend_id │
├─────┼─────────┤        ├─────────┼───────────┤
│ 123 │ Alice   │        │ 123     │ 456       │
│ 456 │ Bob     │        │ 123     │ 789       │
│ 789 │ Carol   │        │ 456     │ 999       │
└─────┴─────────┘        └─────────┴───────────┘

Visual:
        Alice (123)
        /         \
     Bob (456)   Carol (789)
     /    \           \
  Dave    Eve        Frank
```

**SQL Query for Friends-of-Friends:**
```sql
SELECT * FROM users u1
JOIN friendships f1 ON u1.id = f1.user_id      -- Alice's friends
JOIN friendships f2 ON f1.friend_id = f2.user_id  -- Their friends
JOIN users u2 ON f2.friend_id = u2.id
WHERE u1.id = 123 AND u2.id != 123;
```

**Why SQL is Inefficient:**
```
Depth 1: 1 JOIN  → O(n)
Depth 2: 2 JOINs → O(n²)
Depth 6: 6 JOINs → O(n⁶) 💀
Each JOIN = full table scan × previous results
```

---

### Q4: How do Graph DBs Work?

**Key Concept: Index-Free Adjacency**

```
SQL: Relationships in separate table (requires JOIN)
┌───────┐     ┌─────────────────┐
│ Alice │ ──? │ friendships tbl │  → Must scan/index!
└───────┘     └─────────────────┘

Graph DB: Direct pointers stored in node
┌───────┐
│ Alice │──ptr──→ Bob
│       │──ptr──→ Carol    → Just follow pointer!
└───────┘
```

**Graph Node Structure:**
```
┌─────────────────────────────────────────┐
│ Node: Alice                             │
├─────────────────────────────────────────┤
│ id: 123                                 │
│ labels: [:User]                         │
│ properties: {name: "Alice", age: 28}    │
│ relationships: [ptr→Bob, ptr→Carol]    │ ← DIRECT POINTERS
└─────────────────────────────────────────┘
```

**Cypher Query (Graph DB):**
```cypher
MATCH (me:User {id: 123})-[:FRIEND*2..3]-(fof:User)
RETURN DISTINCT fof;

-- Breakdown:
-- (me:User {id: 123})  → Start at Alice
-- [:FRIEND*2..3]       → Follow FRIEND edges 2-3 hops
-- (fof:User)           → Return reached users
```

**Performance Comparison:**

| Query Depth | SQL (1M users) | Graph DB |
|-------------|----------------|----------|
| 2 hops | ~100ms | ~2ms |
| 4 hops | ~10sec | ~10ms |
| 6 hops | Timeout | ~50ms |

**When to Use:**
```
Graph DB: Social networks, recommendations, fraud detection
SQL: Simple CRUD, aggregations, tabular reports
```

**Interview one-liner:** *"Graph DBs use index-free adjacency - nodes store direct pointers to neighbors. Traversal is O(relationships traversed) not O(data size), making multi-hop queries constant time regardless of total users."*

---

*This guide covers the most common HLD interview topics. Practice explaining each concept out loud, as if you're in an interview. Good luck!*
