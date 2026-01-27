# Non-Functional Requirements

## NFR-1: Latency

| Operation | Target | P99 | Notes |
|-----------|--------|-----|-------|
| Simple filter (1-2 fields) | < 50ms | < 100ms | Single service dimension |
| Cross-service filter (3-5 fields) | < 100ms | < 200ms | Multiple dimensions |
| Complex compound (5+ fields with AND/OR) | < 150ms | < 300ms | Nested conditions |
| Full-text search + filters | < 200ms | < 400ms | Text scoring + filtering |
| With aggregations | +50ms | +100ms | Add to above |

**Interview Note:** "Sub-second" in resume is conservative - actual target was < 200ms for complex cross-service queries.

---

## NFR-2: Throughput

| Metric | Before ES | After ES | Improvement |
|--------|-----------|----------|-------------|
| Queries/sec | ~50 | ~150 | **3x** |
| Concurrent users | 20 | 100+ | **5x** |
| P99 latency | 5-10s | 200ms | **25-50x** |

**How improvement was achieved:**

| Factor | Contribution |
|--------|--------------|
| No cross-service API calls | Eliminated 4-5 network round trips |
| Pre-computed joins | No in-memory joins at query time |
| Inverted index | O(1) lookup vs O(n) scan |
| Filter caching | Common patterns cached as bitsets |
| Parallel shard execution | Query runs on 3 shards simultaneously |

---

## NFR-3: Availability

| Component | Target | Strategy |
|-----------|--------|----------|
| Elasticsearch cluster | 99.9% | 3-node cluster, 1 replica per shard |
| Sync pipeline (per service) | 99.5% | Retry with backoff, DLQ per service |
| Overall search API | 99.9% | Graceful degradation |

### Graceful Degradation Strategy

```java
public SearchResponse search(SearchRequest request) {
    try {
        return elasticsearchService.search(request);
    } catch (ElasticsearchException e) {
        log.warn("ES unavailable", e);
        metrics.increment("es.fallback.count");

        // Partial fallback: query individual services
        // Less optimal but still functional
        return fallbackAggregationService.search(request);
    }
}
```

**Note:** Full fallback to individual service APIs is possible but slower (5-10s). We show a degraded experience warning.

---

## NFR-4: Consistency

| Data Flow | Consistency Model |
|-----------|-------------------|
| Service → Own DB | Strong (ACID) |
| Service DB → Kafka | At-least-once delivery |
| Kafka → ES Consumer | Exactly-once processing (idempotent) |
| ES reads | Eventual consistency (~5 sec lag) |

### Consistency Across Services

```
Test Service updates test status
       │
       ▼ (event published)
Bug Service links bug to test
       │
       ▼ (event published)
ES Consumer receives both events
       │
       ▼ (aggregates into single doc)
ES document updated with both changes
       │
       ▼
Dashboard shows consistent view
```

**Challenge:** Events from different services may arrive out of order.
**Solution:** ES consumer fetches latest state from each service before updating document.

---

## NFR-5: Scalability

| Dimension | Current | Target | Strategy |
|-----------|---------|--------|----------|
| Documents | 10M | 100M | Sharding, ILM |
| Services syncing | 6 | 10+ | Modular consumers |
| Daily events | 500K | 5M | Kafka partitions, consumer scaling |
| Index size | 10GB | 100GB | Time-based indices |
| Concurrent searches | 100 | 500 | ES node scaling |

### Horizontal Scaling Points

```
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  Services (6+)                                              │
│      │                                                      │
│      ▼                                                      │
│  Kafka (Scale: Add partitions)                             │
│      │                                                      │
│      ▼                                                      │
│  ES Consumers (Scale: Add instances per partition)         │
│      │                                                      │
│      ▼                                                      │
│  Elasticsearch (Scale: Add data nodes)                     │
│      │                                                      │
│      ▼                                                      │
│  Search API (Scale: Add service instances)                 │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## NFR-6: Data Retention & Lifecycle

```
Hot (< 30 days):
├── Fast SSD storage
├── All replicas active
├── Full indexing
└── Most queries hit this tier

Warm (30-90 days):
├── Standard storage
├── Reduced replicas
├── Read-only
└── Historical queries

Cold (90-365 days):
├── Archived storage
├── Minimal replicas
├── Frozen indices
└── Rare access

Delete (> 365 days):
└── Removed from ES (source DBs retain for compliance)
```

---

## NFR-7: Observability

### Metrics to Track

| Category | Metrics |
|----------|---------|
| **Search Performance** | Query latency P50/P95/P99, queries/sec, error rate |
| **Sync Health** | Events processed/sec, consumer lag, failed syncs |
| **Per-Service Sync** | Lag per service, error rate per service |
| **ES Cluster** | Node health, shard status, disk usage, JVM heap |
| **Index Health** | Document count, index size, refresh latency |

### Alerting Thresholds

| Alert | Condition | Severity |
|-------|-----------|----------|
| Search latency high | P99 > 500ms for 5 min | Warning |
| Sync lag critical | Any service lag > 5 min | Critical |
| ES cluster unhealthy | Cluster status != green | Critical |
| Consumer falling behind | Kafka lag > 10K messages | Warning |
| Index size growing fast | > 10% growth/day | Warning |

### Example Metrics Code

```java
@Service
public class SearchMetrics {

    private final MeterRegistry registry;

    @Timed(value = "search.latency",
           percentiles = {0.5, 0.95, 0.99},
           extraTags = {"type", "cross-service"})
    public SearchResponse search(SearchRequest request) {
        // Search implementation
    }

    public void recordSyncLag(String serviceName, Duration lag) {
        registry.gauge("sync.lag.seconds",
            Tags.of("service", serviceName),
            lag.toSeconds());
    }

    public void recordSyncEvent(String serviceName, boolean success) {
        registry.counter("sync.events",
            Tags.of("service", serviceName, "success", String.valueOf(success)))
            .increment();
    }
}
```

---

## NFR-8: Security

| Aspect | Requirement |
|--------|-------------|
| Authentication | JWT token required for search API |
| Authorization | Users see only data they have access to |
| Data filtering | Owner/team-based filtering in queries |
| Audit logging | Log who searched for what |
| Data encryption | TLS in transit, encryption at rest |

### Row-Level Security Example

```java
public SearchResponse search(SearchRequest request, UserContext user) {
    BoolQueryBuilder query = buildQuery(request);

    // Add security filter based on user's access
    if (!user.isAdmin()) {
        query.filter(QueryBuilders.boolQuery()
            .should(QueryBuilders.termQuery("owner", user.getEmail()))
            .should(QueryBuilders.termQuery("ownerTeam", user.getTeam()))
            .minimumShouldMatch(1));
    }

    return executeSearch(query);
}
```

---

## Summary Table

| NFR | Target | Measured By |
|-----|--------|-------------|
| Latency | < 200ms P99 | Prometheus metrics |
| Throughput | 150+ qps | Load tests |
| Availability | 99.9% | Uptime monitoring |
| Consistency | < 5 sec lag | Sync lag metrics |
| Scalability | 100M docs | Capacity tests |
| Retention | 365 days in ES | ILM policies |
