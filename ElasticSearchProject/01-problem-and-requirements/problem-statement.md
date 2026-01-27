# Problem Statement

## Background

We have a **distributed microservices architecture** where different services own different domains:

| Service | Owns | Database |
|---------|------|----------|
| Bug Service | Bug reports, bug hits | PostgreSQL |
| Mutation (Mut) Service | Test mutations, coverage | PostgreSQL |
| Test Service | Test runs, test suites, platforms | PostgreSQL |
| Image Service | Docker images, builds | PostgreSQL |
| Project Service | Projects, changelists, primaries | PostgreSQL |
| Workspace Service (A4C) | Workspaces, jobs | PostgreSQL |
| User Service | Owners, personalities | PostgreSQL |

Each service has its **own database** - they cannot directly JOIN across databases.

---

## The Problem

### User Query Example

> "Show me all bug hits caused by test `auth_login_test` in the past 30 days,
> for primary `main`, owned by `john@company.com`, on platform `linux-x64`"

This query spans **multiple services**:

```
┌─────────────┐   ┌─────────────┐   ┌─────────────┐   ┌─────────────┐
│ Bug Service │   │Test Service │   │User Service │   │Project Svc  │
│             │   │             │   │             │   │             │
│ • bug_id    │   │ • test_name │   │ • owner     │   │ • primary   │
│ • bug_hit   │   │ • platform  │   │ • personality   │ • changelist│
│ • test_id   │   │ • test_suite│   │             │   │             │
└──────┬──────┘   └──────┬──────┘   └──────┬──────┘   └──────┬──────┘
       │                 │                 │                 │
       │          Cannot JOIN across DBs!                    │
       │                 │                 │                 │
       └─────────────────┴─────────────────┴─────────────────┘
```

### Filterable Dimensions

Users need to filter across these dimensions:

| Dimension | Source Service | Examples |
|-----------|----------------|----------|
| Test Run | Test Service | test name, test suite, platform |
| Bug | Bug Service | bug ID, bug status, bug hit |
| Changelist | Project Service | CL number, primary branch |
| Image | Image Service | image ID, build status |
| Project | Project Service | project name |
| Owner | User Service | email, team |
| Personality | User Service | developer, QA, manager |
| Workspace | Workspace Service | workspace status, hostname |

---

## Before Elasticsearch: The Pain

### Option A: API Aggregation (Slow)

```
Dashboard Request → Backend Service

1. Call Bug Service API: Get bug hits in last 30 days
   Response: 50,000 results

2. Call Test Service API: Get test runs for "auth_login_test"
   Response: 10,000 results

3. Call User Service API: Get entities owned by "john@company.com"
   Response: 5,000 results

4. Join in application memory:
   bugs ∩ tests ∩ users ∩ projects

5. Filter and return

Total time: 5-10 seconds
Memory usage: High (loading all results)
```

**Problems:**
- Multiple network round trips
- Each service returns large result sets
- In-memory joins are slow and memory-intensive
- No pagination until final join complete

### Option B: Shared Database (Anti-pattern)

```
Put all tables in one database for SQL JOINs
```

**Problems:**
- Breaks microservice boundaries
- Schema coupling between services
- Single point of failure
- Scaling nightmare

### Option C: Data Warehouse (Wrong Tool)

```
ETL all data to Snowflake/BigQuery
```

**Problems:**
- Batch updates (not real-time)
- High latency for dashboard queries
- Expensive for simple filtering
- Overkill for operational queries

---

## The Solution: Elasticsearch as Cross-Service Search Index

```
┌──────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│  Bug       Test       User      Project    Image     Workspace          │
│ Service   Service    Service    Service   Service     Service           │
│    │         │          │          │         │           │              │
│    ▼         ▼          ▼          ▼         ▼           ▼              │
│  Bug DB   Test DB   User DB   Project DB  Image DB  Workspace DB        │
│    │         │          │          │         │           │              │
│    │         │          │          │         │           │              │
│    └─────────┴──────────┴──────────┴─────────┴───────────┘              │
│                              │                                           │
│                       Events to Kafka                                    │
│                              │                                           │
│                              ▼                                           │
│              ┌───────────────────────────────┐                          │
│              │       ES Sync Consumer        │                          │
│              │                               │                          │
│              │  • Aggregates events          │                          │
│              │  • Enriches with lookups      │                          │
│              │  • Builds denormalized doc    │                          │
│              └───────────────┬───────────────┘                          │
│                              │                                           │
│                              ▼                                           │
│              ┌───────────────────────────────┐                          │
│              │        Elasticsearch          │                          │
│              │                               │                          │
│              │   Denormalized Index with     │                          │
│              │   data from ALL services      │                          │
│              └───────────────┬───────────────┘                          │
│                              │                                           │
│                        Single Query                                      │
│                              │                                           │
│                              ▼                                           │
│              ┌───────────────────────────────┐                          │
│              │         Dashboard             │                          │
│              │      (Sub-100ms response)     │                          │
│              └───────────────────────────────┘                          │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

### Denormalized Document Example

```json
{
  "id": "test-run-12345",

  "testName": "auth_login_test",
  "testSuite": "auth-integration-suite",
  "platform": "linux-x64",
  "testStatus": "FAILED",

  "bugId": "BUG-6789",
  "bugStatus": "OPEN",
  "bugSeverity": "P1",

  "projectName": "backend-api",
  "changelist": "CL-98765",
  "primary": "main",

  "owner": "john@company.com",
  "ownerTeam": "platform-team",
  "personality": "developer",

  "imageId": "img-abc123",
  "imageBuildStatus": "SUCCESS",

  "workspaceId": "ws-xyz789",
  "workspaceHostname": "worker-node-03",

  "createdAt": "2024-01-15T10:30:00Z",
  "updatedAt": "2024-01-15T11:45:00Z"
}
```

### Complex Query in ES

```json
POST /test-results/_search
{
  "query": {
    "bool": {
      "filter": [
        { "term": { "testName": "auth_login_test" } },
        { "exists": { "field": "bugId" } },
        { "range": { "createdAt": { "gte": "now-30d" } } },
        { "term": { "primary": "main" } },
        { "term": { "owner": "john@company.com" } },
        { "term": { "platform": "linux-x64" } }
      ]
    }
  },
  "sort": [{ "createdAt": "desc" }],
  "size": 50
}
```

**Result:** ~50ms instead of 5-10 seconds

---

## Key Insight: ES Enables "Pre-Computed Joins"

| Approach | Join Time | Query Time |
|----------|-----------|------------|
| SQL across DBs | Not possible | N/A |
| API aggregation | At query time | 5-10 seconds |
| ES denormalized | At write time | 50-100ms |

**Trade-off:** We do the "join" when data is written (async), not when it's queried.

---

## Resume Bullet (Updated)

> "Implemented Elasticsearch-backed advanced screener filters enabling cross-service queries across 6+ microservices, supporting complex AND/OR compound conditions on test runs, bugs, changelists, owners, and platforms. Achieved sub-second searches on 10M+ records by denormalizing data from distributed PostgreSQL databases into a unified search index, improving query throughput by 3x."

---

## Interview Talking Points

1. **"Can you join tables across databases?"**
   > "Not with SQL directly. Each microservice has its own DB. We use Elasticsearch as a search aggregator - data from all services is denormalized into one index at write time. This enables cross-service queries in a single search request."

2. **"Why not a data warehouse?"**
   > "Data warehouses are for analytics with batch updates. We need near real-time search with sub-second latency for an operational dashboard. ES gives us both the data model flexibility and query performance."

3. **"What's the main trade-off?"**
   > "We duplicate data and accept eventual consistency. When a bug status changes in Bug Service, the ES document is updated asynchronously. There's a 2-5 second lag, which is acceptable for dashboard use cases."

4. **"How do you keep ES in sync with 6+ services?"**
   > "Each service publishes events to Kafka when its data changes. An ES sync consumer aggregates these events, enriches with lookups if needed, and updates the denormalized document in ES."
