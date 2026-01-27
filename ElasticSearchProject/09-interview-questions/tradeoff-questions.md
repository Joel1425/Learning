# Trade-off Questions

These questions test your ability to reason about architectural trade-offs.

---

## Consistency Trade-offs

### Q1: You chose eventual consistency. What scenarios would make you reconsider?

**What interviewer is testing:**
- Understanding of when eventual consistency breaks down
- Ability to reassess decisions

**Your answer:**
> I'd reconsider if:
>
> 1. **Financial transactions**: If the dashboard showed account balances or payment status, eventual consistency could lead to duplicate payments or missed alerts.
>
> 2. **Critical alerts**: If status=FAILED needed to trigger immediate incident response, 5-second lag could delay critical notifications.
>
> 3. **User-facing counts**: If we showed "You have 5 pending jobs" and users made decisions based on that count, staleness would confuse them.
>
> For those scenarios, I'd either:
> - Query PostgreSQL directly for critical paths
> - Use a "wait for sync" mechanism before returning response
> - Implement read-your-writes consistency by including recent writes in results
>
> Our dashboard is informational - users browse and filter. Slight staleness doesn't affect decisions, so eventual consistency works.

---

### Q2: What would you do differently if you needed real-time (<1 second) consistency?

**What interviewer is testing:**
- Alternative architecture knowledge
- Trade-off awareness

**Your answer:**
> For real-time consistency:
>
> 1. **Synchronous dual write**: Write to PostgreSQL and ES in same request. Use saga pattern or 2PC for consistency.
>    - Downside: Higher latency, ES failure blocks writes
>
> 2. **ES as primary**: Use ES as the source of truth for this data.
>    - Downside: Lose ACID transactions, complex application logic
>
> 3. **Change Data Capture (CDC)**: Use Debezium to stream PostgreSQL WAL to ES with minimal lag (~100ms).
>    - Downside: More infrastructure, still not synchronous
>
> 4. **Read-your-writes at application level**: After write, query PostgreSQL directly for that user's recent items and merge with ES results.
>    - Downside: More complex read path
>
> Each adds complexity. Our async approach works because we validated with stakeholders that 5-second lag is acceptable.

---

## Performance Trade-offs

### Q3: You increased refresh interval to 5 seconds. What if product wanted 1-second freshness?

**What interviewer is testing:**
- Understanding refresh interval implications
- Negotiation skills

**Your answer:**
> I'd first understand the requirement:
> - Is it all data or specific types (e.g., FAILED jobs)?
> - Is it 1-second P99 or best-effort?
>
> Technical options:
>
> 1. **Reduce refresh interval to 1s**: More segments created, slight query slowdown, more merge overhead. If queries are still fast enough, this is simplest.
>
> 2. **Explicit refresh after critical writes**: For high-priority updates (like FAILED status), call `refresh=wait_for` to force immediate visibility. Other updates use normal 5s.
>
> 3. **Hybrid query**: For dashboard, show ES results + PostgreSQL query for last 5 seconds of data, merged together.
>
> I'd test option 1 first. If query P99 increases unacceptably (say, from 100ms to 200ms), then discuss with product whether latency trade-off is acceptable for fresher data.

---

### Q4: You denormalized user data into job documents. User renames cause stale data. Is that okay?

**What interviewer is testing:**
- Understanding denormalization consequences
- Problem-solving for data staleness

**Your answer:**
> For our use case, it's acceptable:
>
> 1. **User renames are rare**: Maybe 1% of users change their display name
> 2. **Historical accuracy**: The job document shows who ran it at that time - arguably correct
> 3. **New jobs correct**: Any new jobs have the updated name
>
> If it became a problem, mitigation options:
>
> 1. **Re-index on user update**: When user changes name, queue their jobs for re-sync
>    - Downside: User with 10K jobs triggers 10K re-index operations
>
> 2. **Don't denormalize user name**: Store only userId, look up name at query time
>    - Downside: Need nested queries or script fields, slower
>
> 3. **Accept and document**: Show "name at time of job creation" in UI
>
> We chose to accept it. If users complained, we'd implement option 1 with rate limiting.

---

## Architecture Trade-offs

### Q5: Why polling + events instead of pure CDC (Debezium)?

**What interviewer is testing:**
- Understanding of CDC trade-offs
- Pragmatic decision making

**Your answer:**
> CDC with Debezium would be more elegant:
> - Captures all changes from WAL
> - No application-level events needed
> - Works for any writes, even direct SQL
>
> We chose polling + events because:
>
> 1. **Infrastructure overhead**: Debezium requires Kafka Connect cluster, schema registry, connector management. More ops burden.
>
> 2. **Team familiarity**: Team knew Spring events well, less experience with CDC tooling.
>
> 3. **Scale fit**: At 100K writes/day, application events are sufficient. CDC shines at higher volume.
>
> 4. **Control**: With application events, we can add business logic (e.g., only sync certain job types). CDC is more raw.
>
> If we grew to 1M+ writes/day or had multiple applications writing to the same tables, I'd revisit CDC.

---

### Q6: You use Kafka. What if you needed to simplify the architecture?

**What interviewer is testing:**
- Understanding of when complexity is warranted
- Simpler alternatives

**Your answer:**
> Simpler alternatives to Kafka:
>
> 1. **Synchronous writes**: Just call ES directly after DB write. Works for low volume, but ES failure blocks writes.
>
> 2. **Database-backed queue**: Write to an `es_sync_queue` table in same transaction. Poller processes queue.
>    - Pro: Transactional, simpler than Kafka
>    - Con: Polling latency, database load
>
> 3. **In-memory queue**: Use Spring's `@Async` with a simple in-memory queue.
>    - Pro: Very simple
>    - Con: Lost on restart, no durability
>
> For a smaller system or MVP, I'd use the database-backed queue. We chose Kafka because:
> - We already had Kafka in our infrastructure
> - We needed durability and replay capability
> - Higher throughput requirements
>
> Complexity should match the problem. If Kafka is overkill, simpler is better.

---

### Q7: What's the biggest risk in your architecture?

**What interviewer is testing:**
- Self-awareness
- Risk assessment

**Your answer:**
> **Biggest risk**: Sync lag going unnoticed and growing.
>
> Scenario: Consumer starts falling behind due to ES slowdown. Lag grows from 5 seconds to 5 minutes to 5 hours. If monitoring doesn't catch this, users see increasingly stale data.
>
> Mitigations:
> 1. **Alert on Kafka consumer lag** > 1000 messages
> 2. **Alert on sync_lag** (max `updated_at - es_synced_at`) > 5 minutes
> 3. **Dashboard shows "Last synced X ago"** for transparency
>
> **Second risk**: Mapping explosion in ES.
>
> If someone adds arbitrary JSON to metadata and we index it, ES mapping grows unbounded. We mitigated by setting `"enabled": false` on metadata.
>
> **Third risk**: Single point of failure in Kafka.
>
> If Kafka goes down, sync stops. We have a 3-broker cluster, but a full Kafka outage would require PostgreSQL fallback while we restore.

---

### Q8: If you had to cut one component to meet a deadline, what would you remove?

**What interviewer is testing:**
- Prioritization skills
- Understanding of core vs nice-to-have

**Your answer:**
> I'd remove in this order (first = least critical):
>
> 1. **Full-text search on error messages**: Nice-to-have, not core requirement. Users can filter by status and scan.
>
> 2. **Backup poller**: The event-driven path handles 99%+ of cases. Poller is just safety net.
>
> 3. **Circuit breaker/fallback**: Risky, but if ES is stable, fallback rarely triggers.
>
> I would NOT remove:
> - Core ES query capability (that's the whole point)
> - Event-driven sync (core architecture)
> - PostgreSQL as source of truth (data integrity)
>
> The core value is "fast dashboard filtering." Everything else is polish or resilience that can be added later.

---

### Q9: How would you explain this system to a junior developer?

**What interviewer is testing:**
- Communication skills
- Ability to simplify

**Your answer:**
> "We have a dashboard that shows job statuses. Users can filter by status, project, date - lots of combinations.
>
> Our database is PostgreSQL - it stores everything and is always correct. But complex queries on millions of rows are slow.
>
> So we copy the data to Elasticsearch, which is designed for fast searching. Think of it like a phone book - looking up a name is instant, but you can't do transactions with it.
>
> When a job is created, we:
> 1. Save to PostgreSQL (the real record)
> 2. Send a message saying 'job X was updated'
> 3. A separate process reads that message and copies the data to Elasticsearch
>
> When users open the dashboard, we query Elasticsearch - it's fast. The data might be a few seconds behind, but that's okay for a dashboard.
>
> If Elasticsearch is down, we fall back to PostgreSQL - slower, but works."
