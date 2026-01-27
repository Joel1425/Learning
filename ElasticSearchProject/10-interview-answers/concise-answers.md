# Concise Interview Answers

Quick reference for common questions. Use these as starting points, then expand based on follow-ups.

---

## 30-Second Elevator Pitch

> "I implemented Elasticsearch-backed filtering for a dashboard that shows workspace and job statuses. We had 10 million jobs in PostgreSQL, and complex filter queries were taking 2-3 seconds. I designed an event-driven sync to ES, achieving sub-100ms query latency and 3x throughput improvement. PostgreSQL remains source of truth; ES is optimized for reads."

---

## One-Liners

### Why Elasticsearch?

> "PostgreSQL B-tree indexes aren't optimized for multi-field filters. ES inverted index gives O(1) lookups and fast bitset operations for AND/OR."

### How do you sync?

> "Event-driven. Write to PostgreSQL, publish to Kafka after commit, consumer indexes to ES. Backup poller catches missed events."

### What about consistency?

> "Eventually consistent, 2-5 second lag. Acceptable for informational dashboard. PostgreSQL is source of truth."

### What if ES is down?

> "Circuit breaker triggers fallback to PostgreSQL replicas. Slower but functional."

### How did you tune performance?

> "Filter context for caching, 5s refresh interval, keyword types for exact match, 3 shards for parallelization."

---

## Key Metrics to Mention

| Metric | Value |
|--------|-------|
| Data size | 10 million documents |
| Query latency (before) | 2-3 seconds |
| Query latency (after) | 40-80ms (P50), <200ms (P99) |
| Throughput improvement | 3x (50 → 150 qps) |
| Sync latency | 2-5 seconds |
| Shards | 3 primary, 1 replica each |
| Refresh interval | 5 seconds |

---

## Architecture Summary (3 Sentences)

> "PostgreSQL stores all data with ACID transactions. Elasticsearch provides a read-optimized view for dashboard queries. Kafka decouples the sync process so the main service doesn't wait for ES."

---

## Tech Stack Summary

> "Java 17, Spring Boot 3, PostgreSQL 15 for persistence, Elasticsearch 8 for search, Kafka for event streaming, Kubernetes for deployment."

---

## Problem → Solution → Impact

### Problem
> "Dashboard queries with multiple filters on 10M rows were taking 2-3 seconds in PostgreSQL."

### Solution
> "Added Elasticsearch as a read layer with event-driven sync from PostgreSQL."

### Impact
> "Reduced query latency to sub-100ms, increased throughput by 3x, enabled complex AND/OR filtering."

---

## Key Trade-offs Made

1. **Consistency**: Eventual (2-5s lag) vs strong
   > "Acceptable for dashboard use case"

2. **Complexity**: Added ES + Kafka vs simpler architecture
   > "Worth it for performance gains"

3. **Denormalization**: Duplicate data in ES
   > "No joins = faster queries"

4. **Refresh interval**: 5s vs 1s
   > "Better caching, slight staleness trade-off"

---

## Failure Handling Summary

| Failure | Handling |
|---------|----------|
| ES down | Fallback to PostgreSQL |
| Kafka down | Messages queue, retry when up |
| Consumer crash | Kafka rebalance, idempotent reprocess |
| Sync lag | Backup poller catches up |

---

## Questions to Ask the Interviewer

After explaining your project, flip the script:

1. "How does your team handle similar consistency vs performance trade-offs?"
2. "Do you use Elasticsearch or similar search technologies in your stack?"
3. "What's your approach to monitoring and alerting for distributed systems?"
