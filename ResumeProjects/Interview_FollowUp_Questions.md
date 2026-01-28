# Anticipated Follow-Up Questions & Answers

Interviewers often dig deeper after your initial response. Here are common follow-ups organized by topic.

## Table of Contents
1. [Elasticsearch Project Follow-Ups](#elasticsearch-project-follow-ups)
2. [A4C Project Follow-Ups](#a4c-project-follow-ups)
3. [General Technical Follow-Ups](#general-technical-follow-ups)
4. [Deep Grilling Questions](#deep-grilling-questions)
5. [System Design Follow-Ups](#system-design-follow-ups)
6. [Failure Scenario Grilling](#failure-scenario-grilling)
7. [Behavioral Follow-Ups](#behavioral-follow-ups)
8. [Red Flags to Avoid](#red-flags-to-avoid-in-answers)

---

## Elasticsearch Project Follow-Ups

### Why Elasticsearch over other solutions?

**Q: Why not use Redis for caching instead of ES?**
Redis is great for key-value lookups, but our use case required:
- Complex boolean queries with multiple filters
- Full-text search across error messages
- Aggregations for dashboard widgets (counts by status, by team, etc.)
- Handling millions of documents with fast pagination

Redis would require us to model all possible queries upfront. ES provides flexible query capabilities.

**Q: Why not PostgreSQL with proper indexing?**
PostgreSQL with composite indexes could work for simple queries, but:
- Our data was across multiple DBs (microservices) - can't join across DBs
- Full-text search in Postgres is less powerful than ES
- ES is optimized for denormalized, read-heavy workloads
- ES scales horizontally more easily for our document volume

**Q: Did you consider Apache Solr?**
Yes, both are built on Lucene. We chose ES because:
- Better JSON-native API (our team was more familiar with REST)
- Stronger ecosystem for monitoring (Kibana, APM)
- More active community and easier to find expertise
- Better support for our cloud environment

---

### Kafka Architecture Follow-Ups

**Q: Why Kafka over RabbitMQ or SQS?**
- **Durability**: Kafka retains messages for configurable time, enabling replay
- **Ordering**: Key-based partitioning guarantees order per entity
- **Throughput**: Kafka handles higher message rates
- **Consumer groups**: Easy horizontal scaling of consumers
- **Stream processing**: Could add stream processing if needed later

**Q: How do you handle Kafka consumer failures?**
- Consumer commits offset only after successful ES indexing
- If consumer crashes, it resumes from last committed offset
- Messages are automatically retried
- Dead-letter topic for messages that fail repeatedly
- Alert on dead-letter queue growth

**Q: What if Kafka itself goes down?**
- Kafka runs as a cluster with replication (typically 3 replicas)
- Can tolerate 1-2 broker failures
- Services continue writing to their own DBs (source of truth)
- When Kafka recovers, events continue flowing
- We accept brief indexing delay, not data loss

**Q: How do you ensure exactly-once semantics?**
We don't - we use **at-least-once** with **idempotent consumers**:
- Kafka guarantees at-least-once delivery
- ES upserts are idempotent (same doc ID overwrites)
- Duplicate messages result in same final state
- True exactly-once is complex and we didn't need it

---

### Data Consistency Follow-Ups

**Q: How stale can the ES data be?**
- Normal conditions: 1-5 seconds (Kafka + consumer processing + ES refresh)
- Under load: Could be 10-30 seconds if consumer lags
- We monitor consumer lag and alert if it exceeds 1 minute
- For dashboard use case, this staleness is acceptable

**Q: What if source service updates data while you're enriching?**
This is rare but possible:
- We fetch data as close to indexing time as possible
- If data changes during enrichment, next update event will re-index
- Eventual consistency is guaranteed - final state will be correct
- For critical operations, we can force a re-index

**Q: How do you handle deletes?**
Two approaches:
1. **Hard delete**: Remove document from ES when entity is deleted
2. **Soft delete**: Mark document as deleted, filter in queries

We used soft deletes for auditability, then a background job removes old soft-deleted docs.

---

### Scaling Follow-Ups

**Q: How would you scale if traffic 10x'd?**
- Add more Kafka partitions and consumer instances
- Add ES nodes and rebalance shards
- Implement consumer backpressure if needed
- Add caching layer for hot enrichment data
- Consider data tiering (hot/warm/cold indices)

**Q: What's your shard strategy?**
- Started with number of shards = 2x number of data nodes
- Sized for ~30GB per shard (sweet spot for performance)
- Used index aliases for zero-downtime reindexing
- Time-based indices for data with TTL (logs, metrics)

**Q: How do you handle query hotspots?**
- ES distributes queries across replica shards
- Added replicas for high-traffic indices
- Query-level caching for repeated filters
- Considered routing for user-specific queries

---

## A4C Project Follow-Ups

### Container Management Follow-Ups

**Q: Why Docker instead of Kubernetes pods?**
At the time of building:
- Simpler operational model for our use case
- Direct Docker daemon control was sufficient
- K8s would add complexity we didn't need
- Our workspaces were long-lived (days/weeks), not ephemeral pods

In hindsight, K8s would have given us better resource management and scheduling.

**Q: How do you handle container networking?**
- Each container gets a unique IP from our corporate network
- DNS entry created for the hostname
- SRE team managed the network automation
- I focused on the application layer, interfacing with their APIs

**Q: What if a Docker host runs out of resources?**
- Disk Service tracks disk usage per host
- We stop routing new workspaces to hosts above threshold (80% disk, 90% memory)
- Users can delete unused workspaces to free resources
- Admins can migrate workspaces to other hosts if needed

---

### Reliability Follow-Ups

**Q: What's your uptime/availability?**
- A4C service itself: 99.9% (multiple replicas, K8s managed)
- Individual workspaces: depend on host health
- We don't promise workspace uptime SLA - they're development environments
- Critical workloads should use production infrastructure, not workspaces

**Q: How do you handle database schema migrations?**
- Use database migration tools (Flyway/Liquibase)
- Backward-compatible changes first
- Deploy application changes that work with old AND new schema
- Then deploy schema change
- Then deploy application to use new schema only

**Q: What's your disaster recovery plan?**
- Database has replicas in different availability zones
- Docker host data is on network storage with backups
- Service can be redeployed quickly from container images
- Workspace data (source code) exists in source control - not our responsibility

---

### Operational Follow-Ups

**Q: How do you debug issues in production?**
- Structured logging with correlation IDs
- Request tracing across service calls
- Metrics dashboards for key indicators
- Log aggregation (ELK stack) for searching across services
- For workspace issues: SSH into the container if needed

**Q: What metrics do you alert on?**
- **Availability**: Health check failures, service restart
- **Latency**: P99 API latency, workspace creation time
- **Errors**: 5xx rate, failed container operations
- **Saturation**: Host disk/memory, consumer lag
- **Business**: Workspace creation rate, queue depth

**Q: How do you handle on-call?**
- Runbooks for common issues
- Escalation path for unfamiliar problems
- Post-incident reviews for every significant issue
- Blameless culture focused on system improvement

---

## General Technical Follow-Ups

### Why Redis? (If used for caching)

**Q: Why Redis for caching enrichment data?**
- Sub-millisecond reads for cached user/project data
- Simple key-value model fits our use case
- Built-in TTL support for cache expiration
- Cluster mode for high availability
- Team already had Redis expertise

**Q: How do you handle cache invalidation?**
- TTL-based expiration (5 minutes for user data, 1 hour for project data)
- We accept slightly stale cache - eventual consistency is fine
- For critical updates, we could add cache invalidation events (didn't need it)

**Q: What if Redis goes down?**
- Cache misses fall through to source APIs
- Slightly higher latency but no functional impact
- Redis cluster with replicas for HA

---

### Database Follow-Ups

**Q: Why PostgreSQL?**
- ACID compliance for transactional data
- Rich feature set (JSON support, full-text search backup)
- Mature, battle-tested
- Good tooling and team familiarity
- Scales well for our write patterns

**Q: How do you handle database performance?**
- Proper indexing based on query patterns
- Connection pooling (PgBouncer)
- Read replicas for read-heavy queries
- Query analysis and optimization
- Table partitioning for time-series data

**Q: How do you handle schema changes with zero downtime?**
1. Add new column as nullable
2. Deploy code that writes to both old and new
3. Backfill existing data
4. Deploy code that reads from new
5. (Later) Remove old column

---

## Behavioral Follow-Ups

### On Technical Decisions

**Q: What's a decision you made that you'd change now?**
In the ES project, I'd consider Change Data Capture (CDC) instead of application-level events. CDC (like Debezium) captures DB changes directly, which:
- Guarantees no missed events
- Requires no code changes in source services
- Handles all data changes, not just those we explicitly publish

We used application events because CDC wasn't well-established in our org at the time.

**Q: How do you handle disagreements on technical decisions?**
- First, understand their perspective fully
- Present data and trade-offs, not opinions
- Propose experiments or prototypes when possible
- Disagree and commit - support the team's decision
- Revisit if new information emerges

---

### On Working with Others

**Q: How do you handle scope creep?**
- Document agreed scope at project start
- When new requests come in, assess impact on timeline
- Present trade-offs: "We can add X if we cut Y or extend deadline by Z"
- Escalate to product/management for prioritization decisions

**Q: How do you ensure knowledge sharing in a team?**
- Design docs reviewed by team before implementation
- Code reviews with educational comments
- Brown bag sessions for new technologies
- Documentation as first-class deliverable
- Pairing sessions, especially for complex work

**Q: How do you handle working with remote team members?**
- Over-communicate through async channels
- Document decisions in searchable places
- Time zone aware scheduling
- Record meetings for async viewing
- Build personal connections despite distance

---

### On Failure and Learning

**Q: What's the biggest thing you've learned in your career?**
That the simplest solution is usually the best. Early in my career, I over-engineered solutions. Now I always ask:
- What's the minimum that solves the actual problem?
- What can I defer until proven necessary?
- What's the cost of being wrong vs. the cost of over-building?

**Q: How do you stay current with technology?**
- Follow engineering blogs from top companies
- Read HN/Reddit for industry trends
- Experiment with new tech in side projects
- Attend team tech talks and share learnings
- Focus depth on relevant areas, breadth on awareness

**Q: Where do you want to grow?**
- Deeper system design skills for larger scale
- More experience with ML/data infrastructure
- Leadership skills for mentoring and technical direction
- Better at cross-functional collaboration

---

## Deep Grilling Questions

### Elasticsearch Deep Dive

**Q: Walk me through exactly what happens when a query hits your ES cluster.**
```
User Query → GraphQL API
                 ↓
           Query Parser
                 ↓
        ES Client Library
                 ↓
    ┌────────────────────────────┐
    │    ES Coordinating Node    │
    │  (receives the request)    │
    └────────────────────────────┘
                 ↓
         Route to shards
    ┌─────────┴─────────┐
    ↓                   ↓
  Shard 0            Shard 1
  (primary           (primary
   or replica)        or replica)
    ↓                   ↓
  Search in         Search in
  inverted index    inverted index
    ↓                   ↓
  Local results     Local results
    └─────────┬─────────┘
              ↓
        Merge & Sort
        (coordinator)
              ↓
        Return top N
              ↓
        GraphQL Response
```

1. Query hits coordinating node (any node can be coordinator)
2. Coordinator parses query and identifies target shards
3. Coordinator sends sub-queries to relevant shard copies (primary or replica)
4. Each shard searches its inverted index segments
5. Each shard returns local top-N results
6. Coordinator merges, sorts, and returns final results
7. If fetching documents, coordinator does a second "fetch phase"

---

**Q: What's the difference between query and filter context? When do you use each?**

| Aspect | Query Context | Filter Context |
|--------|---------------|----------------|
| **Purpose** | "How well does this match?" | "Does this match? Yes/No" |
| **Scoring** | Calculates relevance score | No scoring (faster) |
| **Caching** | Not cached | Cached for reuse |
| **Use when** | Full-text search, fuzzy matching | Exact matches, ranges, booleans |

**Example:**
```json
{
  "query": {
    "bool": {
      "must": [
        { "match": { "error_message": "timeout" } }  // Query context (scored)
      ],
      "filter": [
        { "term": { "status": "failed" } },          // Filter context (cached)
        { "range": { "created_at": { "gte": "now-7d" } } }
      ]
    }
  }
}
```

I used filters for status, team, and date ranges (exact matches) and queries only for full-text search on error messages.

---

**Q: Your index has 100M documents. How do you handle deep pagination (page 10000)?**

**Problem:** Standard pagination (`from: 100000, size: 10`) requires coordinating node to:
- Request 100,010 documents from EACH shard
- Merge all results
- Discard first 100,000
- This is O(n) memory and CPU

**Solutions I'd use:**

1. **Search After (Recommended)**
```json
{
  "size": 10,
  "sort": [
    { "created_at": "desc" },
    { "_id": "asc" }  // Tiebreaker
  ],
  "search_after": ["2024-01-15T10:30:00", "doc_id_123"]
}
```
- Stateless pagination using sort values
- O(1) memory regardless of page number
- Requires consistent sort order

2. **Scroll API (For batch processing)**
- Creates a point-in-time snapshot
- Good for exporting all data
- Not for user-facing real-time queries

3. **Limit pagination depth**
- UI only allows up to page 100
- "Show more" loads next batch
- Users rarely need page 10000

---

**Q: How do you handle a reindexing scenario with zero downtime?**

```
Step 1: Create new index with alias
┌──────────────────────────────────────────────────────────┐
│ my_index_v1 ←─── [my_index] alias ←─── Application      │
└──────────────────────────────────────────────────────────┘

Step 2: Create v2 with new mapping
┌──────────────────────────────────────────────────────────┐
│ my_index_v1 ←─── [my_index] alias                        │
│ my_index_v2 (empty, new mapping)                        │
└──────────────────────────────────────────────────────────┘

Step 3: Reindex data
┌──────────────────────────────────────────────────────────┐
│ my_index_v1 ─────────reindex────────→ my_index_v2       │
│             ←─── [my_index] alias (still pointing v1)   │
└──────────────────────────────────────────────────────────┘

Step 4: Atomic alias switch
┌──────────────────────────────────────────────────────────┐
│ my_index_v1 (can delete later)                          │
│ my_index_v2 ←─── [my_index] alias ←─── Application      │
└──────────────────────────────────────────────────────────┘
```

**Key Points:**
- Application always uses alias, never direct index name
- Alias switch is atomic (single API call)
- Can keep v1 as fallback, then delete after validation
- New writes during reindex: either pause, or use write alias to both

---

**Q: Your ES cluster is running slow. How do you diagnose?**

**Systematic Approach:**
```
1. CHECK CLUSTER HEALTH
   GET /_cluster/health
   └── Yellow/Red? → Unassigned shards

2. CHECK HOT THREADS
   GET /_nodes/hot_threads
   └── What's consuming CPU?

3. CHECK PENDING TASKS
   GET /_cluster/pending_tasks
   └── Backlog of cluster management tasks?

4. CHECK NODE STATS
   GET /_nodes/stats
   └── JVM heap pressure? GC pauses?
   └── High disk I/O? Segment merge overload?

5. CHECK INDEX STATS
   GET /my_index/_stats
   └── Refresh time? Merge time?
   └── Query cache hit rate?

6. PROFILE SLOW QUERIES
   POST /my_index/_search
   { "profile": true, "query": {...} }
   └── Which phase is slow? Query? Fetch?
```

**Common Culprits I've Seen:**
- Too many small segments (adjust refresh interval)
- Expensive aggregations on high-cardinality fields
- JVM heap too small or too large (sweet spot ~31GB)
- Too many shards for cluster size

---

### A4C Deep Dive

**Q: Explain the exact sequence when a user requests a workspace.**

```
┌────────────────────────────────────────────────────────────────────────────┐
│                    WORKSPACE CREATION SEQUENCE                              │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  1. API Request                                                             │
│     POST /api/v1/workspaces                                                 │
│     { "image": "ubuntu-dev:latest", "name": "my-workspace" }               │
│                                                                             │
│  2. Validation                                                              │
│     ├── User authenticated? (JWT check)                                    │
│     ├── Quota check (user hasn't exceeded limit)                           │
│     ├── Image exists and is approved?                                      │
│     └── Input sanitization                                                  │
│                                                                             │
│  3. Host Selection                                                          │
│     ┌─────────────────────────────────────────────┐                        │
│     │ SELECT * FROM docker_hosts                   │                        │
│     │ WHERE disk_usage < 80                        │                        │
│     │   AND memory_usage < 90                      │                        │
│     │   AND status = 'active'                      │                        │
│     │ ORDER BY disk_usage ASC                      │                        │
│     │ LIMIT 1                                      │                        │
│     └─────────────────────────────────────────────┘                        │
│                                                                             │
│  4. Database Record Creation                                                │
│     INSERT INTO workspaces (id, user_id, host_id, status)                  │
│     VALUES (uuid, user, host, 'CREATING')                                  │
│     └── Status: CREATING (pending)                                          │
│                                                                             │
│  5. Container Creation (Docker API)                                         │
│     POST http://host:2375/containers/create                                 │
│     {                                                                       │
│       "Image": "ubuntu-dev:latest",                                        │
│       "Hostname": "workspace-abc123",                                      │
│       "HostConfig": {                                                      │
│         "Memory": 4294967296,  // 4GB                                      │
│         "CpuShares": 1024                                                  │
│       }                                                                     │
│     }                                                                       │
│                                                                             │
│  6. Container Start                                                         │
│     POST http://host:2375/containers/{id}/start                            │
│                                                                             │
│  7. Network Configuration                                                   │
│     └── SRE service assigns IP and creates DNS entry                       │
│                                                                             │
│  8. Update Status                                                           │
│     UPDATE workspaces SET status = 'RUNNING',                              │
│       ip_address = '10.0.x.x',                                             │
│       hostname = 'workspace-abc123.dev.company.com'                        │
│     WHERE id = workspace_id                                                 │
│                                                                             │
│  9. Return Response                                                         │
│     201 Created                                                             │
│     { "id": "...", "hostname": "...", "status": "RUNNING" }                │
│                                                                             │
└────────────────────────────────────────────────────────────────────────────┘
```

---

**Q: What happens if the Docker daemon call fails after you've created the DB record?**

```
Timeline:
────────────────────────────────────────────────────────────────────
t=0    DB record created (status: CREATING)
t=1    Docker API call fails (timeout, daemon down, etc.)
t=2    ??? What now?
────────────────────────────────────────────────────────────────────

Our Approach: EVENTUAL CLEANUP with State Machine

┌─────────────────────────────────────────────────────────────────┐
│  Workspace State Machine                                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  CREATING ───┬───(success)───→ RUNNING ───→ STOPPING ───→ STOPPED│
│              │                                                   │
│              └───(failure)───→ CREATE_FAILED ───→ (cleanup)      │
│                                     │                            │
│                                     ▼                            │
│                               DELETED (soft)                     │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘

Implementation:

1. Immediate Retry (in same request):
   - Retry Docker call up to 3 times with exponential backoff

2. If still fails, mark as CREATE_FAILED:
   UPDATE workspaces SET status = 'CREATE_FAILED',
     error_message = 'Docker timeout after 3 retries'
   WHERE id = workspace_id

3. Return 500 to user with error message

4. Cleanup Daemon (runs every 5 min):
   - Finds CREATE_FAILED records older than 10 min
   - Attempts cleanup of any partial Docker state
   - Marks as DELETED or retries

5. User can:
   - Retry creation (new request)
   - Contact admin if persistent failures
```

---

**Q: How do you prevent two users from selecting the same host simultaneously?**

```
Problem: Race Condition

Thread 1                     Thread 2
────────                     ────────
Read: Host A has 60% disk
                             Read: Host A has 60% disk
Select Host A
                             Select Host A  ← CONFLICT!
Create workspace on A
                             Create workspace on A ← OVERLOAD!


Solution: SELECT FOR UPDATE with SKIP LOCKED

BEGIN TRANSACTION;

SELECT id, name FROM docker_hosts
WHERE disk_usage < 80
  AND memory_usage < 90
  AND status = 'active'
ORDER BY disk_usage ASC
LIMIT 1
FOR UPDATE SKIP LOCKED;   ← Key magic here!

-- Create workspace with selected host
INSERT INTO workspaces (host_id, ...) VALUES (selected_host, ...);

COMMIT;


How it works:
────────────────────────────────────────────────────────────────────
Thread 1                         Thread 2
────────                         ────────
SELECT... FOR UPDATE SKIP LOCKED
Locks Host A
                                 SELECT... FOR UPDATE SKIP LOCKED
                                 Host A is locked → SKIPPED
                                 Selects Host B (next best)
Creates workspace on A           Creates workspace on B
COMMIT                           COMMIT
Releases lock on A               Releases lock on B
────────────────────────────────────────────────────────────────────

Why SKIP LOCKED vs just FOR UPDATE?
- Plain FOR UPDATE: Thread 2 WAITS (bad for latency)
- SKIP LOCKED: Thread 2 gets next available (parallel processing)
```

---

**Q: How do you handle image updates? What if a user is using an old image?**

```
Image Lifecycle:

1. NEW IMAGE BUILD
   ┌─────────────────────────────────────────────────────────────┐
   │ Build triggered (schedule or manual)                        │
   │ ↓                                                            │
   │ Docker build on build server                                │
   │ ↓                                                            │
   │ Tag with version: ubuntu-dev:v2.3.0, ubuntu-dev:latest      │
   │ ↓                                                            │
   │ Push to internal registry                                    │
   │ ↓                                                            │
   │ Update database: images table                                │
   └─────────────────────────────────────────────────────────────┘

2. EXISTING WORKSPACES
   - They keep running on old image (container already created)
   - We don't auto-upgrade (could break user's work)
   - User sees notification: "New image version available"

3. USER UPGRADE FLOW
   ┌─────────────────────────────────────────────────────────────┐
   │ User clicks "Upgrade"                                        │
   │ ↓                                                            │
   │ We create NEW workspace with new image                       │
   │ ↓                                                            │
   │ User migrates their work (git pull, etc.)                   │
   │ ↓                                                            │
   │ User deletes old workspace                                   │
   └─────────────────────────────────────────────────────────────┘

4. FORCED DEPRECATION (Security patches)
   - Mark old image as "deprecated"
   - Users get warning with deadline
   - After deadline: old workspaces still run but can't create new
   - Eventually: force-stop with notice period
```

---

## System Design Follow-Ups

**Q: If you were to redesign this system from scratch, what would you change?**

**Elasticsearch Project:**
```
Current Design           →    Ideal Redesign
─────────────────────────────────────────────────────────
Application Events       →    CDC (Debezium)
- Requires code in       →    - Captures ALL DB changes
  each source service    →    - No code changes needed
- Can miss events        →    - Guaranteed capture
- Schema coupling        →    - Decoupled from app
```

**A4C Project:**
```
Current Design           →    Ideal Redesign
─────────────────────────────────────────────────────────
Direct Docker Daemon     →    Kubernetes
- Manual host selection  →    - K8s scheduler handles
- Manual scaling         →    - Horizontal pod autoscaler
- Limited isolation      →    - Network policies, RBAC
- Custom cleanup         →    - K8s garbage collection
```

---

**Q: How would you handle multi-region deployment?**

**Elasticsearch:**
```
┌─────────────────────────────────────────────────────────────────┐
│                    MULTI-REGION ES ARCHITECTURE                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   US-EAST                           US-WEST                      │
│   ┌──────────────┐                  ┌──────────────┐            │
│   │  ES Cluster  │                  │  ES Cluster  │            │
│   │  (Primary)   │ ───replication───│  (Replica)   │            │
│   └──────────────┘                  └──────────────┘            │
│          ↑                                 ↑                     │
│          │                                 │                     │
│   ┌──────────────┐                  ┌──────────────┐            │
│   │   Kafka      │ ───mirroring─────│   Kafka      │            │
│   │   Cluster    │                  │   Cluster    │            │
│   └──────────────┘                  └──────────────┘            │
│          ↑                                 ↑                     │
│   ┌──────────────┐                  ┌──────────────┐            │
│   │  Services    │                  │  Services    │            │
│   └──────────────┘                  └──────────────┘            │
│                                                                  │
│   Options:                                                       │
│   1. Cross-Cluster Search (query both from one endpoint)        │
│   2. Follow-Leader indices (automatic replication)              │
│   3. MirrorMaker for Kafka (replicate events)                   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**A4C:**
```
┌─────────────────────────────────────────────────────────────────┐
│                    MULTI-REGION A4C ARCHITECTURE                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Each region has:                                               │
│   ├── Regional A4C service deployment                           │
│   ├── Regional database (with cross-region replication)         │
│   └── Regional Docker hosts                                      │
│                                                                  │
│   User → GeoDNS → Nearest Region                                │
│                                                                  │
│   Workspaces are region-local (no cross-region migration)       │
│   User data (git) is in global source control                   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

**Q: How do you ensure security in these systems?**

**Elasticsearch:**
```
┌─────────────────────────────────────────────────────────────────┐
│                    ES SECURITY MEASURES                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Network Security:                                               │
│  ├── ES cluster in private subnet (no public access)           │
│  ├── API layer is the only entry point                         │
│  └── TLS for all ES communication                               │
│                                                                  │
│  Authentication:                                                 │
│  ├── API layer handles user auth (JWT)                         │
│  ├── Service-to-ES uses internal certs                         │
│  └── No direct user access to ES                                │
│                                                                  │
│  Authorization:                                                  │
│  ├── GraphQL layer enforces field-level access                 │
│  ├── Users only see their team's data (filter in query)        │
│  └── ES role-based access for service accounts                  │
│                                                                  │
│  Data Protection:                                                │
│  ├── Encryption at rest (ES security feature)                  │
│  ├── Sensitive fields masked in indices                        │
│  └── Audit logging for all queries                              │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**A4C:**
```
┌─────────────────────────────────────────────────────────────────┐
│                    A4C SECURITY MEASURES                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Container Isolation:                                            │
│  ├── Each workspace is isolated container                       │
│  ├── Resource limits (CPU, memory, disk)                        │
│  ├── Network isolation between workspaces                       │
│  └── User namespaces (non-root by default)                      │
│                                                                  │
│  Image Security:                                                 │
│  ├── Base images from approved sources only                     │
│  ├── Regular vulnerability scanning (Trivy, Clair)             │
│  ├── No privileged containers allowed                           │
│  └── Read-only root filesystem where possible                   │
│                                                                  │
│  Access Control:                                                 │
│  ├── SSH key management (user's public key injected)           │
│  ├── No shared accounts between users                           │
│  ├── API authentication via SSO/OIDC                           │
│  └── Audit logging for all operations                           │
│                                                                  │
│  Host Security:                                                  │
│  ├── Docker daemon on private network only                      │
│  ├── TLS for Docker API calls                                   │
│  ├── Regular security patches for hosts                         │
│  └── Monitoring for suspicious activity                         │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Failure Scenario Grilling

### "What If" Scenarios

**Q: What if your ES cluster loses a node during peak traffic?**

```
Scenario: 3-node cluster, 1 node fails during peak

Immediate Impact:
────────────────────────────────────────────────────────────────────
- Cluster goes YELLOW (missing replicas on failed node)
- Primary shards on failed node: promoted from replicas on other nodes
- Queries continue to work (but with 2/3 capacity)
- No data loss if replication factor >= 2

Auto-Recovery:
────────────────────────────────────────────────────────────────────
1. ES detects node failure (ping timeout)
2. Master promotes replica shards to primary
3. Starts creating new replicas on remaining nodes
4. Cluster returns to GREEN when rebalanced

What I would do:
────────────────────────────────────────────────────────────────────
1. Alert fires → assess situation
2. Check remaining capacity (CPU, memory, disk)
3. If overloaded: consider reducing traffic (circuit breaker)
4. Bring up replacement node ASAP
5. Let ES rebalance automatically
6. Monitor until GREEN status
```

---

**Q: What if a malicious user tries to overload your A4C service?**

```
Attack Vector                  Defense
────────────────────────────────────────────────────────────────────
Rapid workspace creation       Per-user quota (max 5 workspaces)
                               Rate limiting (1 creation per minute)
                               CAPTCHA for suspicious patterns

Resource exhaustion            Resource limits per container
(CPU, memory, disk)            Host capacity monitoring
                               Automatic workspace hibernation

Docker daemon attacks          Docker API on private network
                               TLS + client certs
                               API call auditing

Privilege escalation           Non-root containers
                               Dropped capabilities
                               seccomp/AppArmor profiles
```

---

**Q: What happens if your database becomes unavailable?**

**ES Project:**
```
┌─────────────────────────────────────────────────────────────────┐
│ DATABASE DOWN SCENARIO (Source DB, not ES)                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│ 1. Enrichment calls fail (can't fetch user/project data)       │
│                                                                  │
│ 2. Circuit breaker trips after X failures                       │
│                                                                  │
│ 3. Consumer behavior:                                            │
│    ├── Stop consuming new messages (pause)                      │
│    └── Kafka retains messages (no data loss)                    │
│                                                                  │
│ 4. ES continues serving EXISTING indexed data (stale but works) │
│                                                                  │
│ 5. When DB recovers:                                             │
│    ├── Circuit breaker closes                                   │
│    ├── Consumer resumes from last offset                        │
│    └── Catches up on backlog                                    │
│                                                                  │
│ User Impact: Dashboard works but shows slightly stale data      │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**A4C Project:**
```
┌─────────────────────────────────────────────────────────────────┐
│ DATABASE DOWN SCENARIO                                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│ 1. API fails for: create, list, delete operations              │
│                                                                  │
│ 2. EXISTING workspaces continue running (Docker daemon alive)   │
│    └── Users can keep working in their containers              │
│                                                                  │
│ 3. Health check returns degraded                                 │
│                                                                  │
│ 4. When DB recovers:                                             │
│    ├── Reconciliation job compares DB state vs actual          │
│    └── Updates any discrepancies                                │
│                                                                  │
│ User Impact: Can't create/delete, but existing work continues   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

**Q: How do you handle a cascading failure across services?**

```
Cascading Failure Example:
────────────────────────────────────────────────────────────────────
User Service overloaded → All enrichment calls timeout
→ Consumer threads blocked → Consumer lag increases
→ Eventually consumer crashes → Full outage

Prevention Strategies:
────────────────────────────────────────────────────────────────────

1. CIRCUIT BREAKER (already implemented)
   - After N failures, stop calling User Service
   - Return cached data or skip enrichment
   - Periodically test if service recovered

2. BULKHEAD PATTERN
   - Separate thread pools per service
   - User Service failures don't block Project Service calls

   ┌─────────────────────────────────────────────┐
   │              Consumer                        │
   │  ┌─────────────┐  ┌─────────────┐           │
   │  │ User Pool   │  │ Project Pool│           │
   │  │ (5 threads) │  │ (5 threads) │           │
   │  └─────────────┘  └─────────────┘           │
   │         ↓                ↓                   │
   │  User Service     Project Service           │
   └─────────────────────────────────────────────┘

3. TIMEOUT + RETRY
   - Aggressive timeouts (500ms)
   - Limited retries (2-3)
   - Exponential backoff

4. GRACEFUL DEGRADATION
   - Index with partial data if enrichment fails
   - Flag documents for later re-enrichment
   - Better stale data than no data
```

---

**Q: What's your incident response process?**

```
INCIDENT RESPONSE TIMELINE
────────────────────────────────────────────────────────────────────

0-5 min: DETECTION
├── Alert fires (PagerDuty/Slack)
├── On-call acknowledges
└── Initial triage: "What's the blast radius?"

5-15 min: MITIGATION
├── Is there a quick fix? (rollback, restart, scale up)
├── Communicate to stakeholders: "We're aware, investigating"
└── Loop in additional help if needed

15-60 min: INVESTIGATION
├── Check recent deployments (correlation?)
├── Review logs, metrics, traces
├── Identify root cause
└── Implement fix or workaround

Post-incident: FOLLOW-UP
├── Incident report within 24 hours
├── Blameless post-mortem meeting
├── Action items with owners and deadlines
└── Share learnings with team

My Personal Approach:
────────────────────────────────────────────────────────────────────
1. Stay calm - panic doesn't help
2. Communicate early - stakeholders prefer knowing you're on it
3. Rollback first, debug later - restore service ASAP
4. Document as you go - helps write post-mortem
5. Ask for help early - fresh eyes catch things
```

---

## Red Flags to Avoid in Answers

1. **Blaming others**: Even if true, focus on what YOU learned
2. **No metrics**: Quantify impact whenever possible
3. **Too much "we"**: Show YOUR contribution
4. **Negativity about past employers**: Keep it professional
5. **Oversimplifying**: Show you understand complexity
6. **Not admitting mistakes**: Everyone makes them; show you learn
7. **Vague answers**: Be specific with examples

### Additional Anti-Patterns

```
┌────────────────────────────────────────────────────────────────────────────┐
│                         INTERVIEW ANTI-PATTERNS                             │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ❌ SAYING                          ✅ SAY INSTEAD                          │
│  ──────────────────────────────────────────────────────────────────────     │
│  "It was trivial"                   Show the complexity, even if you        │
│                                     made it look easy                       │
│                                                                             │
│  "That's not my area"               "I collaborated with X team on that"    │
│                                                                             │
│  "We didn't need to think           "We evaluated options and chose         │
│   about that"                       to defer X because..."                  │
│                                                                             │
│  "I don't know"                     "I'm not certain, but my hypothesis     │
│  (giving up)                        would be... Let me think through it"    │
│                                                                             │
│  "The old system was terrible"      "There were opportunities to improve"   │
│                                                                             │
│  Silence when stuck                 "Let me think out loud about this..."   │
│                                                                             │
└────────────────────────────────────────────────────────────────────────────┘
```

---

## Quick Reference: Your Key Metrics

```
┌────────────────────────────────────────────────────────────────────────────┐
│                    METRICS TO REMEMBER                                      │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ELASTICSEARCH PROJECT                                                      │
│  ├── Query latency: 5-10s → 50-100ms (98% improvement)                     │
│  ├── Throughput: 10,000+ index operations/minute at peak                   │
│  ├── Enrichment optimization: 200-300ms → 60-80ms (4x improvement)         │
│  ├── P99 latency: 2.5s → 200ms after segment optimization                  │
│  ├── Consumer lag: consistently 0 after optimization                       │
│  └── ES cluster IOPS: reduced 40% with refresh tuning                      │
│                                                                             │
│  A4C PROJECT                                                                │
│  ├── Workspace creation: <2 minutes vs. days (manual)                      │
│  ├── Launch day: 200+ workspaces, zero failures                            │
│  ├── Deadline: delivered 4 days early                                      │
│  ├── Orphaned records after cleanup: 0 in 6 months                         │
│  ├── Host disk usage variance: 55% → <10%                                  │
│  └── Service availability: 99.9%                                           │
│                                                                             │
│  BEHAVIORAL                                                                 │
│  ├── Onboarding time: 2-3 weeks → 1 week                                   │
│  ├── User Service API calls: reduced 95% with caching                      │
│  ├── Deployment time: 45 min → 8 min                                       │
│  └── Incident detection: <5 minutes                                        │
│                                                                             │
└────────────────────────────────────────────────────────────────────────────┘
```
