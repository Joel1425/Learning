# Why Sub-Second Latency?

## The Resume Claim

> "...enabling sub-second searches on large datasets..."

This document explains what makes Elasticsearch fast and how we achieved sub-second (actually sub-100ms) latency.

---

## PostgreSQL vs Elasticsearch: Why the Speed Difference?

### PostgreSQL Query (Before)

```sql
SELECT * FROM job
WHERE status = 'FAILED'
  AND project_name = 'backend-api'
  AND created_at > '2024-01-01'
ORDER BY created_at DESC
LIMIT 20;
```

**What happens:**
```
1. Parse query
2. Plan execution (check indexes)
3. For each filter:
   - If index exists: B-tree lookup → O(log n)
   - If no index: Full table scan → O(n)
4. For compound conditions:
   - May need to scan and intersect results
5. Sort results
6. Return top 20
```

**Problem with compound queries:**
```
WHERE status = 'FAILED'           -- Index on status: finds 100K rows
  AND project_name = 'backend'    -- Index on project: finds 200K rows
  AND created_at > '2024-01-01'   -- Index on date: finds 500K rows

PostgreSQL must:
- Either use one index and filter the rest (still slow)
- Or try to merge multiple index scans (bitmap scan)
- JOINs make this even worse
```

Typical latency: **500ms - 3s** for 10M rows with complex filters

---

### Elasticsearch Query (After)

```json
{
  "query": {
    "bool": {
      "filter": [
        { "term": { "status": "FAILED" } },
        { "term": { "projectName": "backend-api" } },
        { "range": { "createdAt": { "gte": "2024-01-01" } } }
      ]
    }
  }
}
```

**What happens:**
```
1. Each filter term: O(1) lookup in inverted index
   - status=FAILED → docIds: [1, 5, 8, 12, ...]
   - projectName=backend-api → docIds: [1, 3, 8, 15, ...]

2. AND operation: Bitset intersection → O(min(n,m))
   - Result: docIds: [1, 8, ...]

3. Range query: Also uses inverted index segments

4. Results already in memory, return top 20
```

Typical latency: **20-80ms** for 10M documents

---

## The Inverted Index Explained

```
Traditional Database Index (B-tree):
┌─────────────────────────────────────────┐
│  Row 1: {status: FAILED, project: A}    │──┐
│  Row 2: {status: SUCCESS, project: B}   │  │
│  Row 3: {status: FAILED, project: A}    │  │ Scan rows to find
│  Row 4: {status: SUCCESS, project: A}   │  │ matching documents
│  ...                                    │  │
│  Row 10M: {status: FAILED, project: C}  │──┘
└─────────────────────────────────────────┘

Inverted Index (Elasticsearch):
┌─────────────────────────────────────────┐
│  status:                                │
│    FAILED → [1, 3, 7, 12, 15, ...]     │ ← Direct lookup
│    SUCCESS → [2, 4, 5, 8, 9, ...]      │
│    IN_PROGRESS → [6, 10, 11, ...]      │
│                                         │
│  projectName:                           │
│    project-a → [1, 3, 4, 8, ...]       │ ← Direct lookup
│    project-b → [2, 5, 9, ...]          │
│    project-c → [6, 7, 10, ...]         │
└─────────────────────────────────────────┘
```

**Key insight:** Inverted index stores term → document mapping, so finding all documents with a specific term is O(1) lookup, not O(n) scan.

---

## Filter Cache: The Secret Weapon

```
First query: status = 'FAILED'
┌─────────────────────────────────────────┐
│ 1. Lookup inverted index                │
│ 2. Build bitset: [1,0,1,0,0,1,1,0,...]  │
│ 3. Cache bitset for "status=FAILED"     │
│ 4. Return results                       │
│                                         │
│ Time: 50ms                              │
└─────────────────────────────────────────┘

Second query: status = 'FAILED' AND project = 'A'
┌─────────────────────────────────────────┐
│ 1. Get cached bitset for status=FAILED  │ ← Cache hit!
│ 2. Lookup inverted index for project=A  │
│ 3. AND the bitsets together             │
│ 4. Return results                       │
│                                         │
│ Time: 15ms                              │
└─────────────────────────────────────────┘
```

---

## Latency Breakdown

Our typical query latency components:

| Component | Time | Notes |
|-----------|------|-------|
| Network (app → ES) | 1-3ms | Same datacenter |
| Query parsing | <1ms | Simple bool query |
| Shard routing | <1ms | Determine target shards |
| Filter execution | 5-20ms | Inverted index + cache |
| Scoring | 0ms | Filter context, no scoring |
| Fetch phase | 5-15ms | Retrieve actual documents |
| Response serialization | 2-5ms | JSON encoding |
| Network (ES → app) | 1-3ms | Same datacenter |
| **Total** | **15-50ms** | P95: ~80ms |

---

## What "Large Dataset" Means

```
Our scale:
- Total documents: 10 million
- Daily new documents: 100K
- Document size: ~1KB each
- Index size: ~15GB
- Shards: 3 primary, 1 replica each

Query complexity:
- 2-5 filter conditions
- Date range spanning 7-30 days
- Sorting by date DESC
- Pagination: 20-100 results
```

Without optimization, 10M documents in PostgreSQL:
- Full table scan: ~30 seconds
- With indexes: ~2-3 seconds
- With ES: ~50ms

---

## Comparison Chart

| Metric | PostgreSQL (Before) | Elasticsearch (After) |
|--------|--------------------|-----------------------|
| Simple filter | 200ms | 20ms |
| Complex AND/OR | 1.5s | 40ms |
| Full-text search | 3s (with pg_trgm) | 60ms |
| Aggregations | 5s | 100ms |
| Concurrent queries | 50/s | 200/s |
| P99 latency | 3.2s | 180ms |

---

## Why Not Just Add PostgreSQL Indexes?

```sql
CREATE INDEX idx_job_status ON job(status);
CREATE INDEX idx_job_project ON job(project_name);
CREATE INDEX idx_job_created ON job(created_at);
```

**Problems:**

1. **Index intersection is expensive:**
   PostgreSQL can combine indexes, but it's complex:
   ```
   Bitmap Index Scan on idx_job_status
     AND Bitmap Index Scan on idx_job_project
     → Bitmap Heap Scan
   ```

2. **Cardinality issues:**
   - status has 6 values → index not very selective
   - Optimizer may skip index entirely

3. **Read amplification:**
   - Each index lookup reads from different disk locations
   - Random I/O is slow

4. **Impact on writes:**
   - More indexes = slower inserts/updates
   - Our write-heavy workload suffers

---

## Interview Talking Points

1. **"How did you achieve sub-second latency?"**
   > "Elasticsearch uses an inverted index that provides O(1) lookups for term queries. Combined with filter caching, common filter combinations are extremely fast. We also used filter context instead of query context to skip scoring."

2. **"Why is ES faster than PostgreSQL for this use case?"**
   > "PostgreSQL's B-tree indexes require O(log n) traversal and aren't optimized for combining multiple filter conditions. ES's inverted index can directly look up all documents matching a term, and bitset operations make AND/OR combinations very efficient."

3. **"What's the actual latency you achieved?"**
   > "P50 is around 40ms, P95 is under 100ms. The 'sub-second' claim is conservative - we actually achieved sub-100ms for most queries. The 3x throughput improvement came from offloading reads from PostgreSQL primary."
