# Consistency and Availability

## CAP Theorem Context

Our system makes explicit trade-offs between Consistency, Availability, and Partition tolerance.

```
          Consistency
              /\
             /  \
            /    \
           / Our  \
          / Choice \
         /    ●     \
        /____________\
   Availability    Partition
                   Tolerance
```

**Our choice:** AP (Availability + Partition tolerance) for reads, CP for writes.

---

## Consistency Model

### PostgreSQL: Strong Consistency

```
┌─────────────────────────────────────────────────────────────┐
│                   PostgreSQL Writes                         │
│                                                             │
│  Client ──▶ A4C Service ──▶ PostgreSQL Primary             │
│                                   │                         │
│                              COMMIT                         │
│                                   │                         │
│                     ┌─────────────┴─────────────┐          │
│                     ▼                           ▼          │
│                 Response              WAL to Replicas       │
│              (immediately)            (async, ~10ms)        │
│                                                             │
│  Guarantee: Write acknowledged = Write durable             │
└─────────────────────────────────────────────────────────────┘
```

**Properties:**
- All writes go to primary
- Transactions are ACID
- Read-after-write consistency (if reading from primary)
- Synchronous commit

### Elasticsearch: Eventual Consistency

```
┌─────────────────────────────────────────────────────────────┐
│                 Elasticsearch Reads                         │
│                                                             │
│  Timeline:                                                  │
│                                                             │
│  T0: Job created in PostgreSQL                             │
│  T1: Event published to Kafka (~10ms)                      │
│  T2: Consumer processes message (~50-500ms)                │
│  T3: Document indexed in ES (~10ms)                        │
│  T4: ES refresh (up to 5s)                                 │
│  T5: Document searchable                                   │
│                                                             │
│  Total lag: 100ms - 6s (typically ~2-3s)                   │
│                                                             │
│  Read at T2: Returns STALE data (or no data)               │
│  Read at T5: Returns CURRENT data                          │
└─────────────────────────────────────────────────────────────┘
```

**Properties:**
- Writes to ES are async
- ~2-5 second lag typical
- May return stale data
- Eventually converges to PostgreSQL state

---

## Consistency Scenarios

### Scenario 1: User Creates Workspace, Immediately Checks Dashboard

```
T0: User calls POST /workspaces
T1: Workspace created in PostgreSQL
T2: Response returned to user (201 Created)
T3: User opens dashboard (GET /jobs/search)
T4: ES query returns... nothing! (not synced yet)

User experience: "Where's my workspace?!"
```

**Mitigation Options:**

1. **UI Optimistic Update:**
   ```javascript
   // Frontend adds item locally before API confirms
   function createWorkspace(data) {
     setWorkspaces([...workspaces, { ...data, status: 'CREATING' }]);
     await api.createWorkspace(data);
   }
   ```

2. **Direct PostgreSQL Query for Recent Creates:**
   ```java
   public SearchResponse search(JobSearchRequest request) {
       SearchResponse esResult = esService.search(request);

       // For user's own recent jobs, supplement from PostgreSQL
       if (request.getUserId() != null) {
           List<Job> recentJobs = jobRepository.findByUserIdAndCreatedAfter(
               request.getUserId(),
               Instant.now().minus(Duration.ofMinutes(1))
           );
           return mergeResults(esResult, recentJobs);
       }

       return esResult;
   }
   ```

3. **User Feedback:**
   ```
   "Your workspace is being created. It will appear in the dashboard shortly."
   ```

### Scenario 2: Job Status Updates, Dashboard Shows Old Status

```
T0: Job status changes from IN_PROGRESS to COMPLETED
T1: PostgreSQL updated
T2: ES still shows IN_PROGRESS (lag)
T3: User sees IN_PROGRESS on dashboard
T4: User refreshes, still IN_PROGRESS
T5: (5 seconds later) Now shows COMPLETED
```

**Mitigation:**
- Auto-refresh dashboard every 30 seconds
- "Last updated: X seconds ago" indicator
- Manual refresh button

---

## Availability Model

### High Availability Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Availability Zones                       │
│                                                             │
│  Zone A                Zone B                Zone C         │
│  ┌──────────┐         ┌──────────┐         ┌──────────┐    │
│  │A4C Svc-1 │         │A4C Svc-2 │         │A4C Svc-3 │    │
│  └──────────┘         └──────────┘         └──────────┘    │
│       │                    │                    │           │
│       └────────────────────┼────────────────────┘           │
│                            │                                │
│  ┌──────────┐         ┌──────────┐         ┌──────────┐    │
│  │PG Primary│─────────│PG Replica│─────────│PG Replica│    │
│  └──────────┘         └──────────┘         └──────────┘    │
│                                                             │
│  ┌──────────┐         ┌──────────┐         ┌──────────┐    │
│  │ ES Node  │─────────│ ES Node  │─────────│ ES Node  │    │
│  │ (S0,R1)  │         │ (S1,R2)  │         │ (S2,R0)  │    │
│  └──────────┘         └──────────┘         └──────────┘    │
│                                                             │
│  ┌──────────┐         ┌──────────┐         ┌──────────┐    │
│  │ Kafka-1  │─────────│ Kafka-2  │─────────│ Kafka-3  │    │
│  └──────────┘         └──────────┘         └──────────┘    │
└─────────────────────────────────────────────────────────────┘
```

### Availability Targets

| Component | Target | Strategy |
|-----------|--------|----------|
| Overall API | 99.9% | Multi-instance, LB health checks |
| Dashboard Search | 99.9% | ES cluster + PostgreSQL fallback |
| Workspace Create | 99.5% | Retry logic, cleanup on failure |
| Data Durability | 99.999% | PostgreSQL replication + backups |

### Failure Mode Analysis

| Failure | Impact | Recovery |
|---------|--------|----------|
| 1 A4C instance down | None (LB routes around) | Auto-restart |
| PostgreSQL primary down | Writes fail temporarily | Promote replica (manual/auto) |
| PostgreSQL replica down | Read capacity reduced | Other replicas take load |
| 1 ES node down | Queries continue (replicas) | Node rejoins cluster |
| ES cluster down | Dashboard uses PG fallback | Slower but functional |
| Kafka broker down | Message delivery delayed | Other brokers, retry |
| ES Consumer down | Sync lag increases | Consumer group rebalance |

---

## Trade-off Decisions

### Decision 1: Accept ES Staleness

```
Alternative A: Synchronous ES write
  - Write to PostgreSQL
  - Write to ES
  - Return response

  Pros: Immediate consistency
  Cons: Higher latency, ES failure blocks writes

Alternative B: Async ES write (our choice)
  - Write to PostgreSQL
  - Return response
  - Async sync to ES

  Pros: Low latency, ES failure doesn't block
  Cons: Eventual consistency (~5s lag)

Decision: B - Dashboard staleness is acceptable
```

### Decision 2: Fallback to PostgreSQL When ES Down

```
Option A: Return error when ES down
  - User sees error
  - Faster failure

Option B: Fallback to PostgreSQL (our choice)
  - Slower but works
  - Degraded experience

Decision: B - Availability over performance
```

### Decision 3: Async Replication for PostgreSQL

```
Option A: Synchronous replication
  - Zero data loss
  - Higher write latency
  - Replica failure blocks primary

Option B: Async replication (our choice)
  - Possible data loss on failover (seconds)
  - Lower latency
  - Better availability

Decision: B - We can tolerate seconds of data loss
          Dashboard data is recoverable
```

---

## Consistency Guarantees Summary

| Operation | Consistency | Notes |
|-----------|-------------|-------|
| Create workspace | Strong | PostgreSQL ACID |
| Update job status | Strong | PostgreSQL ACID |
| Dashboard search | Eventual | 2-5s lag |
| Get workspace by ID | Strong | Direct PostgreSQL query |
| Full-text search | Eventual | ES only |

---

## Interview Talking Points

1. **"What consistency model does your system use?"**
   > "We use strong consistency for writes via PostgreSQL, and eventual consistency for reads via Elasticsearch. The ES data lags by 2-5 seconds. This trade-off is acceptable because the dashboard is informational, not transactional."

2. **"What if a user creates something and immediately queries for it?"**
   > "They might not see it in ES yet. We handle this with UI optimistic updates, showing a 'creating' state locally. For critical views, we can supplement ES results with a PostgreSQL query for recent items."

3. **"How available is the system?"**
   > "We target 99.9% availability. The API layer has multiple instances with load balancing. If Elasticsearch goes down, we fall back to PostgreSQL for queries. PostgreSQL has replicas for read scaling and failover. Each component is designed to degrade gracefully rather than fail completely."
