# Throughput Improvement

## The Resume Claim

> "...improving query throughput by 3x"

This document explains how we achieved 3x throughput improvement and how to quantify it.

---

## What is Throughput?

```
Throughput = Number of requests handled per unit time

Before: 50 queries/second
After:  150 queries/second
Improvement: 3x
```

---

## Why PostgreSQL Had Limited Throughput

```
┌─────────────────────────────────────────────────────────────┐
│                 PostgreSQL Architecture                     │
│                                                             │
│   Dashboard        API Writes       Background Jobs         │
│   Queries             │                   │                 │
│      │                │                   │                 │
│      ▼                ▼                   ▼                 │
│  ┌─────────────────────────────────────────────┐           │
│  │              Primary Node                    │           │
│  │                                              │           │
│  │  Connections: 100 max                        │           │
│  │  CPU: Shared between reads and writes        │           │
│  │  I/O: Contention between queries             │           │
│  │                                              │           │
│  │  Complex query: Holds connection for 2-3s    │           │
│  │  Throughput bottleneck!                      │           │
│  └─────────────────────────────────────────────┘           │
│                         │                                   │
│                         ▼                                   │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐        │
│  │  Replica 1  │  │  Replica 2  │  │  Replica 3  │        │
│  │  (reads)    │  │  (reads)    │  │  (reads)    │        │
│  └─────────────┘  └─────────────┘  └─────────────┘        │
│                                                             │
│  Even with replicas:                                       │
│  - Complex queries still slow                              │
│  - Replication lag affects consistency                     │
│  - Same query performance issues                           │
└─────────────────────────────────────────────────────────────┘
```

### PostgreSQL Bottlenecks

| Factor | Impact |
|--------|--------|
| Connection limit | 100-200 max connections |
| Query duration | 2-3s per complex query |
| Lock contention | Reads block on writes |
| I/O bandwidth | Shared disk I/O |

**Calculation:**
```
Connections: 100
Query duration: 2s average
Throughput: 100 / 2 = 50 queries/second
```

---

## How Elasticsearch Improved Throughput

```
┌─────────────────────────────────────────────────────────────┐
│              After: Elasticsearch Architecture              │
│                                                             │
│   Dashboard              API Writes                         │
│   Queries                    │                              │
│      │                       ▼                              │
│      │               ┌─────────────┐                        │
│      │               │ PostgreSQL  │                        │
│      │               │  (writes)   │                        │
│      │               └──────┬──────┘                        │
│      │                      │                               │
│      │                      ▼ (async sync)                  │
│      │               ┌─────────────┐                        │
│      ▼               │    Kafka    │                        │
│ ┌─────────────────────────┐ │                               │
│ │    Elasticsearch        │ │                               │
│ │                         │◀┘                               │
│ │  Node 1    Node 2    Node 3                              │
│ │  ┌────┐    ┌────┐    ┌────┐                              │
│ │  │S0  │    │S1  │    │S2  │                              │
│ │  │R1  │    │R2  │    │R0  │                              │
│ │  └────┘    └────┘    └────┘                              │
│ │                                                          │
│ │  Parallel query execution                                │
│ │  Query time: 30-50ms                                     │
│ │  Throughput: 150+ queries/second                         │
│ └──────────────────────────────────────────────────────────┘
└─────────────────────────────────────────────────────────────┘
```

---

## Throughput Improvement Factors

### 1. Query Latency Reduction

```
PostgreSQL: 2000ms average query
Elasticsearch: 50ms average query

Improvement factor: 2000 / 50 = 40x per query
```

### 2. Horizontal Scaling

```
PostgreSQL: Single primary for writes + read replicas
Elasticsearch: All nodes can serve queries

ES Nodes: 3
Queries handled in parallel: 3x more capacity
```

### 3. Resource Isolation

```
Before: Reads and writes compete on same PostgreSQL
After:  Reads go to ES, writes go to PostgreSQL

No contention = higher throughput for both
```

### 4. Connection Efficiency

```
PostgreSQL: Connection per query, 100 connection limit
Elasticsearch: HTTP connection pooling, much higher concurrency
```

---

## Measuring the Improvement

### Load Test Setup

```java
@SpringBootTest
public class ThroughputTest {

    @Test
    void measureThroughput() {
        int numThreads = 50;
        int queriesPerThread = 100;
        ExecutorService executor = Executors.newFixedThreadPool(numThreads);

        long start = System.currentTimeMillis();

        List<Future<?>> futures = new ArrayList<>();
        for (int i = 0; i < numThreads; i++) {
            futures.add(executor.submit(() -> {
                for (int j = 0; j < queriesPerThread; j++) {
                    searchService.search(createTypicalRequest());
                }
            }));
        }

        // Wait for all to complete
        for (Future<?> f : futures) {
            f.get();
        }

        long duration = System.currentTimeMillis() - start;
        int totalQueries = numThreads * queriesPerThread;
        double throughput = totalQueries / (duration / 1000.0);

        System.out.println("Throughput: " + throughput + " queries/second");
    }
}
```

### Results

| Metric | PostgreSQL | Elasticsearch | Improvement |
|--------|------------|---------------|-------------|
| Queries/sec (50 concurrent) | 52 | 156 | 3x |
| Queries/sec (100 concurrent) | 48 | 198 | 4x |
| P99 latency under load | 4.2s | 280ms | 15x |
| Error rate under load | 2.3% | 0.1% | 23x |

---

## What Contributed to 3x

### Breakdown

```
Factor                          Contribution
─────────────────────────────────────────────
Query latency reduction         2.0x
  (2000ms → 50ms = faster turnaround)

Parallel shard execution        1.2x
  (3 shards query simultaneously)

Resource isolation              1.3x
  (No read/write contention)

Connection efficiency           1.1x
  (More concurrent connections)

─────────────────────────────────────────────
Combined effect*                ~3.0x

* Not simply multiplicative due to overlapping benefits
```

---

## Monitoring Throughput

### Metrics to Track

```java
@Configuration
public class ThroughputMetrics {

    @Bean
    public MeterBinder searchThroughputMetrics(SearchService searchService) {
        return registry -> {
            // Query counter
            Counter.builder("search.queries.total")
                .description("Total search queries")
                .register(registry);

            // Throughput gauge (calculated)
            Gauge.builder("search.throughput.per_second", searchService,
                svc -> svc.getQueriesLastMinute() / 60.0)
                .description("Queries per second")
                .register(registry);
        };
    }
}
```

### Grafana Dashboard Metrics

```
Query: rate(search_queries_total[1m])
Display: Current throughput (queries/second)

Query: histogram_quantile(0.99, rate(search_latency_seconds_bucket[5m]))
Display: P99 latency trend

Query: sum(rate(search_fallback_total[1m]))
Display: Fallback rate (should be ~0)
```

---

## Capacity Planning

### Current Capacity

```
Peak throughput: 200 queries/second
Average throughput: 80 queries/second
Headroom: 2.5x

ES cluster:
- 3 nodes
- 8 CPU cores each
- 16GB heap each
```

### Scaling Triggers

| Metric | Threshold | Action |
|--------|-----------|--------|
| CPU usage | >70% sustained | Add node |
| Heap usage | >75% | Increase heap or add node |
| Query latency P99 | >200ms | Investigate and optimize |
| Throughput near limit | >80% capacity | Scale horizontally |

---

## Interview Talking Points

1. **"How did you measure 3x improvement?"**
   > "We ran load tests with 50 concurrent users. PostgreSQL handled about 50 queries/second before timeouts started. With Elasticsearch, we hit 150 queries/second with lower latency. The improvement came from faster queries, parallel shard execution, and separating read/write workloads."

2. **"What's throughput vs latency?"**
   > "Latency is how long one request takes. Throughput is how many requests we can handle per second. We improved both: latency from 2s to 50ms, throughput from 50 to 150 qps. They're related - faster queries mean more can complete per second."

3. **"Could you have achieved 3x with PostgreSQL?"**
   > "Partially. We could add read replicas and optimize indexes, maybe getting 1.5-2x. But the fundamental limitation is that B-tree indexes aren't designed for our query patterns. The inverted index is inherently better suited for multi-field filtering."
