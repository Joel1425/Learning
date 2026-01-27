# Common Interview Questions

## Category 1: Why Elasticsearch?

### Q1: Why did you use Elasticsearch instead of just PostgreSQL?

**What interviewer is testing:**
- Understanding of database trade-offs
- Ability to justify technical decisions
- Knowledge of ES strengths

**Your answer:**
> PostgreSQL was our source of truth, but complex dashboard queries with multiple AND/OR conditions on 10 million rows were taking 2-3 seconds. The B-tree indexes in PostgreSQL aren't optimized for combining multiple filters efficiently.
>
> Elasticsearch uses an inverted index where each term maps to document IDs. Looking up `status=FAILED` is O(1), and combining filters is just bitset intersection. This brought our query latency from 2-3 seconds to under 100ms.
>
> We kept PostgreSQL for writes and ACID transactions. ES is purely for read optimization on the dashboard.

**Follow-up:** "Could you use PostgreSQL materialized views instead?"

> Materialized views could help for static queries, but our users filter dynamically with different combinations. We'd need views for every combination, which isn't practical. ES handles arbitrary filter combinations efficiently.

---

### Q2: Why not use a read replica for dashboard queries?

**What interviewer is testing:**
- Understanding that ES isn't just a database copy
- Knowledge of query pattern differences

**Your answer:**
> Read replicas help with read scaling but don't solve the fundamental query performance issue. The same complex query that takes 2 seconds on primary will take 2 seconds on replica.
>
> ES provides a different data structure - the inverted index - that's fundamentally better suited for our query pattern. It's not about scaling; it's about using the right tool for the job.

---

## Category 2: Consistency

### Q3: How do you keep PostgreSQL and Elasticsearch in sync?

**What interviewer is testing:**
- Understanding of event-driven architecture
- Knowledge of consistency trade-offs

**Your answer:**
> We use an event-driven approach. When a job is created or updated in PostgreSQL, we publish an event to Kafka after the transaction commits. An ES sync consumer picks up the message, fetches the latest state from PostgreSQL, and indexes it to ES.
>
> We also have a backup poller that runs every 30 seconds to catch any missed events. It scans for jobs where `updated_at > es_synced_at`.

**Follow-up:** "What if the Kafka message is processed twice?"

> Our consumer is idempotent. We compare version numbers - if ES already has the same or newer version, we skip the update. This handles retries and out-of-order processing.

---

### Q4: What happens if a user creates a workspace and immediately checks the dashboard?

**What interviewer is testing:**
- Understanding of eventual consistency implications
- User experience considerations

**Your answer:**
> They might not see it in ES yet due to the 2-5 second sync lag. We handle this a few ways:
>
> 1. UI shows an optimistic update - the item appears locally in a "creating" state immediately
> 2. For the user's own recent items, we can supplement ES results with a PostgreSQL query for jobs created in the last minute
> 3. We show feedback: "Your workspace is being created and will appear shortly"
>
> For our use case, this eventual consistency is acceptable because the dashboard is informational, not transactional.

---

## Category 3: Failure Handling

### Q5: What happens if Elasticsearch goes down?

**What interviewer is testing:**
- Resilience thinking
- Fallback strategies

**Your answer:**
> We have a circuit breaker around ES calls. When ES fails, the circuit opens and we fall back to querying PostgreSQL replicas. It's slower - maybe 200-500ms instead of 50ms - but the dashboard remains functional.
>
> We also show a banner to users: "Search results may be delayed" and alert the on-call team.

**Follow-up:** "How does the circuit breaker work?"

> We use Resilience4j. It tracks failure rate over a sliding window. When failures exceed 50% over 10 requests, the circuit opens. After 30 seconds, it allows test requests through. If those succeed, it closes again.

---

### Q6: What if the sync consumer crashes?

**What interviewer is testing:**
- Understanding of Kafka consumer behavior
- Message reliability

**Your answer:**
> Kafka handles this gracefully. If the consumer crashes before committing the offset, the message will be reprocessed by another consumer in the group after a rebalance.
>
> Our consumer is idempotent, so reprocessing the same message just results in the same ES document being indexed again - no harm done.
>
> The backup poller also catches any gaps if something slips through.

---

## Category 4: Performance

### Q7: How did you achieve "sub-second" latency?

**What interviewer is testing:**
- Understanding of ES performance characteristics
- Technical depth

**Your answer:**
> Several factors:
>
> 1. **Inverted index**: Term lookup is O(1), not O(n) scan
> 2. **Filter context**: We use `filter` instead of `must`, which enables caching and skips scoring
> 3. **Filter cache**: Common filter combinations are cached as bitsets
> 4. **Parallel shards**: Query runs on all 3 shards simultaneously
> 5. **Denormalized data**: No joins needed at query time
>
> Our typical query is 40-80ms, with P99 around 180ms. The "sub-second" claim is actually conservative.

---

### Q8: How did you measure the 3x throughput improvement?

**What interviewer is testing:**
- Metrics-driven decision making
- Load testing knowledge

**Your answer:**
> We ran load tests with 50 concurrent users. With PostgreSQL, we handled about 50 queries/second before timeouts started. With Elasticsearch, we hit 150 queries/second with lower latency.
>
> The improvement came from:
> - Faster individual queries (more queries complete per second)
> - Offloading from PostgreSQL primary (no read/write contention)
> - ES's thread pool handling higher concurrency

---

## Category 5: Data Modeling

### Q9: How did you design the ES mapping?

**What interviewer is testing:**
- ES data modeling knowledge
- Understanding of field types

**Your answer:**
> I designed based on query patterns:
>
> - Fields used for exact match filters (status, userId) are `keyword` type
> - Fields needing partial match (projectName) have a multi-field: `keyword` for exact match, `.search` subfield with `text` type
> - Date fields use explicit format for consistency
> - We disabled indexing on `metadata` to prevent mapping explosion from arbitrary JSON
>
> We also set `norms: false` on fields we don't score to save memory.

---

### Q10: Why denormalize data in ES?

**What interviewer is testing:**
- Understanding of denormalization trade-offs
- ES best practices

**Your answer:**
> ES doesn't do joins well. If we stored just `userId` and had to look up user name separately, we'd need multiple round trips or expensive script queries.
>
> By denormalizing user name and workspace hostname into the job document, all queryable data is in one place. The trade-off is staleness - if a user changes their name, old job documents have the old name.
>
> For our use case, this is acceptable. User renames are rare, and the historical accuracy (name at time of job) may actually be preferred.

---

## Category 6: Scaling

### Q11: How would you scale this as data grows?

**What interviewer is testing:**
- Forward thinking
- Scalability knowledge

**Your answer:**
> Several strategies:
>
> 1. **Index lifecycle management**: Move old data to warm/cold storage, eventually delete from ES while keeping in PostgreSQL
> 2. **Time-based indices**: Create monthly indices (`jobs-2024-01`) and query across them with aliases
> 3. **Add shards**: For much larger data, increase shard count (though this requires reindexing)
> 4. **Add nodes**: ES scales horizontally; add more data nodes
>
> Currently at 10M documents, we're comfortable. The architecture supports growth to 100M+ with these approaches.

---

### Q12: How did you decide on 3 shards?

**What interviewer is testing:**
- Shard sizing knowledge
- Capacity planning

**Your answer:**
> Rule of thumb is 10-50GB per shard. We had 15GB with projections to 70GB in a year. Three shards keeps each under 25GB long-term.
>
> Three also enables parallelization across 3 nodes without over-sharding. Too many shards creates overhead - each shard has memory and file handle costs.
>
> We also configured replicas = 1 for redundancy, so 6 total shard copies across the cluster.
