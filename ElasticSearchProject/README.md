# Elasticsearch Screener Design - Interview Preparation

## Resume Bullet

> "Implemented Elasticsearch-backed advanced screener filters supporting complex AND/OR compound queries, index tuning, and query profiling, enabling sub-second searches on large datasets and improving query throughput by 3x."

---

## Quick Reference

| Metric | Value |
|--------|-------|
| Data size | 10 million documents |
| Query latency (before) | 2-3 seconds |
| Query latency (after) | 40-80ms (P50), <200ms (P99) |
| Throughput improvement | 3x (50 → 150 qps) |
| Sync latency | 2-5 seconds (eventual consistency) |

---

## Architecture Summary

```
Client → API Gateway → A4C Service ──┬──▶ PostgreSQL (writes, source of truth)
                                     │
                                     ├──▶ Elasticsearch (reads, dashboard)
                                     │         ▲
                                     │         │
                                     └──▶ Kafka → ES Sync Consumer
```

---

## Folder Structure

```
elasticsearch-screener-design/
│
├── 01-problem-and-requirements/
│   ├── problem-statement.md       # Why we needed ES
│   ├── functional-requirements.md # What the system does
│   └── non-functional-requirements.md # Performance, availability targets
│
├── 02-core-entities/
│   ├── job.md                     # Primary indexed entity
│   ├── workspace.md               # Workspace entity
│   └── relationships.md           # ER diagram and schema
│
├── 03-data-model-and-indexing/
│   ├── postgres-job-table.md      # PostgreSQL schema
│   ├── elasticsearch-index-mapping.md # ES mapping design
│   └── indexing-strategy.md       # Sync approach (events + polling)
│
├── 04-query-and-filtering/
│   ├── filter-vs-query.md         # Filter context vs query context
│   ├── and-or-compound-queries.md # Complex query building
│   └── sample-elasticsearch-queries.md # Real query examples
│
├── 05-control-flow/
│   ├── write-path-db-to-es.md     # How data flows to ES
│   ├── read-path-dashboard-query.md # How queries execute
│   └── failure-scenarios.md       # What can go wrong, handling
│
├── 06-performance-and-tuning/
│   ├── why-sub-second-latency.md  # Inverted index explanation
│   ├── index-tuning.md            # Shards, refresh, mapping
│   ├── query-profiling.md         # Finding slow queries
│   └── throughput-improvement.md  # How 3x was achieved
│
├── 07-high-level-design/
│   ├── system-architecture.md     # Full architecture
│   ├── component-responsibilities.md # What each component does
│   └── consistency-and-availability.md # CAP trade-offs
│
├── 08-diagrams/
│   ├── system-architecture.txt    # ASCII diagram for whiteboard
│   ├── data-flow.txt              # Write and read path diagrams
│   ├── query-flow.txt             # ES query execution detail
│   └── original-architecture.png  # Your reference diagram
│
├── 09-interview-questions/
│   ├── common-questions.md        # Frequently asked
│   ├── deep-dive-questions.md     # Advanced technical
│   └── tradeoff-questions.md      # Architectural decisions
│
└── 10-interview-answers/
    ├── concise-answers.md         # Quick reference answers
    ├── strong-vs-weak-answers.md  # How to answer well
    └── red-flags.md               # What to avoid
```

---

## How to Use This

### Before the Interview

1. **Read through all folders in order** (01 → 10)
2. **Memorize key metrics** from the Quick Reference above
3. **Practice drawing the architecture** from [08-diagrams/system-architecture.txt](08-diagrams/system-architecture.txt)
4. **Review Q&A** in folders 09 and 10

### During the Interview

1. **Start with the elevator pitch** from [10-interview-answers/concise-answers.md](10-interview-answers/concise-answers.md)
2. **Draw the architecture** when asked about the system
3. **Use specific numbers** (10M docs, 50ms latency, 3x improvement)
4. **Explain trade-offs proactively**

### Quick Study Path (If Short on Time)

1. [01-problem-and-requirements/problem-statement.md](01-problem-and-requirements/problem-statement.md)
2. [08-diagrams/system-architecture.txt](08-diagrams/system-architecture.txt)
3. [10-interview-answers/concise-answers.md](10-interview-answers/concise-answers.md)
4. [10-interview-answers/red-flags.md](10-interview-answers/red-flags.md)

---

## Key Talking Points

### Why Elasticsearch?
- PostgreSQL B-tree indexes aren't optimized for multi-field filters
- ES inverted index gives O(1) term lookups
- Filter caching speeds up repeated queries

### How Sync Works
1. Write to PostgreSQL (source of truth)
2. Publish event to Kafka after commit
3. Consumer fetches from DB, indexes to ES
4. Backup poller catches missed events

### Trade-offs Made
- Eventual consistency (2-5s lag) - acceptable for dashboard
- Added complexity (Kafka, ES) - worth it for performance
- Denormalized data - faster queries, potential staleness

### Failure Handling
- ES down → Fallback to PostgreSQL
- Consumer crash → Kafka rebalance, idempotent reprocess
- Message lost → Backup poller recovers

---

## Stack

| Component | Technology |
|-----------|------------|
| Language | Java 17 |
| Framework | Spring Boot 3 |
| Database | PostgreSQL 15 |
| Search | Elasticsearch 8 |
| Messaging | Apache Kafka |
| Container | Kubernetes |

---

Good luck with your interviews!
