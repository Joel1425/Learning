# Deep Dive Questions

These are advanced questions an experienced interviewer might ask to probe deeper.

---

## Elasticsearch Internals

### Q1: Explain how the inverted index works for your use case.

**What interviewer is testing:**
- Deep understanding of ES data structures
- Ability to explain complex concepts

**Your answer:**
> An inverted index maps terms to document IDs. For the `status` field:
>
> ```
> FAILED → [doc1, doc3, doc7, ...]
> COMPLETED → [doc2, doc4, doc5, ...]
> ```
>
> When I query `status = FAILED`, ES does a direct lookup in this map - O(1) operation. It returns a bitset: `[1,0,1,0,0,0,1,...]`
>
> For compound queries like `status = FAILED AND project = backend`, ES looks up both terms, gets two bitsets, and ANDs them together. Bitset operations are extremely fast at the CPU level.
>
> This is fundamentally different from B-tree indexes where you traverse a tree structure for each lookup.

---

### Q2: What's the difference between `filter` and `must` in a bool query?

**What interviewer is testing:**
- Precise ES knowledge
- Understanding of scoring vs filtering

**Your answer:**
> Both require the condition to match, but they differ in two ways:
>
> 1. **Scoring**: `must` contributes to the relevance score. `filter` doesn't score - it's binary yes/no.
>
> 2. **Caching**: `filter` results are cached as bitsets. `must` results are not cached because scores can vary.
>
> For our dashboard, we use `filter` for structured fields (status, project, dates) since we don't need relevance ranking. We only use `must` for full-text search in error messages where ranking matters.

---

### Q3: How does the filter cache work? When is it invalidated?

**What interviewer is testing:**
- Deep ES internals knowledge

**Your answer:**
> ES caches filter results per segment as bitsets. When you query `status = FAILED`, ES builds a bitset marking which documents match, then caches it.
>
> The cache is invalidated when:
> - The segment is merged (Lucene segment lifecycle)
> - The cache entry is evicted due to memory pressure (LRU eviction)
> - The index is refreshed and documents change
>
> That's why our 5-second refresh interval helps caching - fewer segment changes mean longer cache validity.
>
> One gotcha: range queries with `now` aren't cached because the value changes each request. We prefer explicit timestamps for cacheable range filters.

---

### Q4: Explain `search_after` vs `from/size` pagination.

**What interviewer is testing:**
- Understanding of ES pagination mechanisms
- Awareness of deep pagination problems

**Your answer:**
> `from/size` is offset-based: "skip 1000, return 20". ES still has to load and score all 1020 documents to return the last 20. At `from: 10000`, this becomes expensive and is limited to 10000 by default.
>
> `search_after` uses sort values as a cursor. I provide the sort values of the last document from the previous page. ES uses the inverted index to directly jump to that position - it doesn't process previous pages.
>
> For our dashboard with potentially millions of results, we use `search_after`. The trade-off is you can only go forward, not jump to page 500. But that's fine for infinite scroll UIs.

---

## Sync Architecture

### Q5: Why do you fetch from PostgreSQL in the consumer instead of using the event payload?

**What interviewer is testing:**
- Understanding of event sourcing pitfalls
- Data consistency reasoning

**Your answer:**
> Events can arrive out of order or be duplicated. If I used the event payload:
>
> 1. Event A says status = IN_PROGRESS (older)
> 2. Event B says status = COMPLETED (newer, but arrives first)
> 3. I process B: status = COMPLETED
> 4. I process A: status = IN_PROGRESS ← Wrong!
>
> By always fetching from PostgreSQL, I get the current state regardless of event order. The consumer is idempotent - processing the same job twice just overwrites with the same data.
>
> This is "eventual consistency done right" - the destination always converges to the source.

---

### Q6: What if PostgreSQL write succeeds but Kafka publish fails?

**What interviewer is testing:**
- Understanding of distributed transaction challenges
- Compensation strategies

**Your answer:**
> This is the classic "dual write" problem. Our solution has two parts:
>
> 1. **After-commit listener**: We only publish to Kafka after the transaction commits successfully. If the transaction fails, no event is sent.
>
> 2. **Backup poller**: If Kafka publish fails after commit (network issue), the job exists in PostgreSQL but ES doesn't know about it. Our poller scans for jobs where `updated_at > es_synced_at` and queues them for sync.
>
> So we might have temporary inconsistency (PostgreSQL has it, ES doesn't), but the poller ensures eventual sync.

**Follow-up:** "Could you use the transactional outbox pattern?"

> Yes, that's a more robust approach. Instead of publishing to Kafka directly, we'd write to an outbox table in the same transaction. A separate process reads the outbox and publishes to Kafka.
>
> We considered it, but our polling approach was simpler and sufficient for our scale. If we had strict ordering requirements or higher volume, outbox would be better.

---

## Performance Deep Dive

### Q7: Walk me through what happens in ES when a query is executed.

**What interviewer is testing:**
- End-to-end understanding
- Ability to explain complex flows

**Your answer:**
> 1. **Client sends request** to any ES node (coordinating node)
>
> 2. **Coordinating node parses** the query and determines which shards to query (all 3 for our typical query)
>
> 3. **Query phase**: Coordinating node sends query to all shard primaries (or replicas)
>    - Each shard checks filter cache for cached bitsets
>    - If cache miss, looks up inverted index
>    - Applies scoring if using query context
>    - Returns top N document IDs + sort values
>
> 4. **Merge phase**: Coordinating node collects results from all shards, does global sort, selects final top N
>
> 5. **Fetch phase**: Coordinating node fetches actual documents from shards that have the top N docs
>
> 6. **Response**: Documents returned to client

---

### Q8: You mentioned leading wildcards are slow. How would you implement partial matching efficiently?

**What interviewer is testing:**
- ES optimization knowledge
- Problem-solving

**Your answer:**
> Leading wildcards like `*backend*` force full index scans because the inverted index is sorted by term prefix.
>
> Options:
>
> 1. **Multi-field with text analyzer**: Create a `.search` subfield that tokenizes the value. "backend-api" becomes ["backend", "api"]. Then `match: "backend"` works efficiently.
>
> 2. **Ngram tokenizer**: Configure an ngram analyzer that creates tokens like ["bac", "ack", "cke", "ken", "end"]. Partial matches work but index size increases.
>
> 3. **Prefix query**: If you only need prefix matching (`backend*`), that's efficient since inverted index is sorted.
>
> We used option 1 - a `.search` subfield with standard analyzer. It handles most use cases and doesn't bloat the index.

---

### Q9: How would you debug a slow query in production?

**What interviewer is testing:**
- Operational knowledge
- Debugging methodology

**Your answer:**
> 1. **Check slow log**: We configured ES slow logs to capture queries over 100ms. This gives the exact query that's slow.
>
> 2. **Use Profile API**: Run the query with `"profile": true` to see time breakdown per component. This identifies which part is slow - a specific filter, scoring, fetching.
>
> 3. **Check for patterns**:
>    - Leading wildcards? Replace with match query
>    - Deep pagination? Should use search_after
>    - Query context when filter would work? Move to filter clause
>    - Complex aggregations? May need optimization
>
> 4. **Check cluster health**: Maybe it's not the query - check node CPU, memory, GC, shard distribution.
>
> 5. **Check filter cache hit rate**: If low, maybe queries aren't using filter context or have too many dynamic values.

---

## Architecture Decisions

### Q10: Why Kafka? Why not publish directly to ES consumer?

**What interviewer is testing:**
- Understanding of decoupling benefits
- Message queue knowledge

**Your answer:**
> Direct communication creates tight coupling:
>
> 1. **ES down = service blocked**: If we call ES directly and it's down, our main service hangs or fails
>
> 2. **No buffering**: Spikes in writes would overwhelm ES directly
>
> 3. **No retry built-in**: We'd have to implement retry logic in the main service
>
> 4. **No replay**: If ES was down, we'd lose those updates
>
> Kafka solves all these:
> - ES consumer can be down; messages queue up
> - Consumer processes at its own pace (backpressure)
> - Kafka handles retries
> - We can replay messages if needed
>
> The trade-off is added infrastructure, but for production systems, this decoupling is worth it.

---

### Q11: You have multiple backend instances. How do you ensure they don't duplicate work?

**What interviewer is testing:**
- Distributed systems thinking
- Race condition awareness

**Your answer:**
> The question applies to different components:
>
> **ES Sync Consumer**: Kafka consumer groups handle this. Each partition is assigned to only one consumer in the group. Messages are processed exactly once (assuming our idempotent processing).
>
> **Backup Poller**: This could be a problem if all instances poll simultaneously. Options:
> 1. Use distributed lock (Redis, ZooKeeper)
> 2. Use Spring's `@SchedulerLock` annotation
> 3. Designate one instance as leader
>
> We use `@SchedulerLock` with a database-backed lock. Only one instance runs the poller at a time.
>
> **API Requests**: Stateless, no issue. Any instance can handle any request.
