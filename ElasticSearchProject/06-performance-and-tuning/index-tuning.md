# Index Tuning

## The Resume Claim

> "...index tuning..."

This document covers the specific tuning we applied to optimize Elasticsearch performance.

---

## 1. Shard Configuration

### Problem: Default Sharding

```
Default: 1 shard
- All queries hit single shard
- Can't parallelize
- Single node bottleneck
```

### Our Configuration

```json
PUT /jobs-v1
{
  "settings": {
    "number_of_shards": 3,
    "number_of_replicas": 1
  }
}
```

### Sizing Logic

```
Rule of thumb: 10-50 GB per shard

Our data:
- 10M documents × 1.5KB = ~15GB
- Growth: 100K docs/day × 1.5KB = 150MB/day
- 1 year projection: ~70GB

Shard sizing:
- 3 shards = ~25GB each after 1 year
- Comfortable headroom
- Can parallelize across 3 nodes
```

### Shard Allocation

```json
{
  "settings": {
    "index.routing.allocation.total_shards_per_node": 2
  }
}
```

Ensures shards spread across nodes for fault tolerance.

---

## 2. Refresh Interval

### Default Behavior

```
Default refresh_interval: 1s
- Every second, new documents become searchable
- Each refresh creates a new segment
- Too many segments = slow queries
```

### Our Configuration

```json
PUT /jobs-v1
{
  "settings": {
    "refresh_interval": "5s"
  }
}
```

### Trade-off

| Refresh Interval | Pros | Cons |
|------------------|------|------|
| 1s (default) | Near real-time | More segments, more merging |
| 5s (our choice) | Fewer segments, faster queries | 5s delay for new docs |
| 30s | Best write throughput | Too stale for dashboard |
| -1 (disabled) | Maximum write speed | Must refresh manually |

For our dashboard, 5s staleness is acceptable.

---

## 3. Mapping Optimization

### Field Data Types

```json
{
  "mappings": {
    "properties": {
      "status": {
        "type": "keyword"       // NOT text - saves analysis overhead
      },
      "projectName": {
        "type": "keyword",
        "fields": {
          "search": { "type": "text" }
        }
      },
      "errorMessage": {
        "type": "text",
        "index_options": "positions",  // Reduced from "offsets"
        "norms": false                 // We don't need relevance scoring
      },
      "metadata": {
        "type": "object",
        "enabled": false    // Don't index, just store
      }
    }
  }
}
```

### Why These Choices?

| Setting | What It Does | Our Choice | Why |
|---------|--------------|------------|-----|
| keyword vs text | keyword = exact match, text = analyzed | keyword for filters | Faster, less storage |
| norms | Stores length normalization for scoring | false for non-scored | Saves memory |
| index_options | How much position info to store | positions | We don't need highlights on all fields |
| enabled: false | Don't index field at all | For metadata | Prevents mapping explosion |

---

## 4. Doc Values Optimization

```json
{
  "mappings": {
    "properties": {
      "errorMessage": {
        "type": "text",
        "doc_values": false    // We never sort/aggregate on this
      },
      "createdAt": {
        "type": "date",
        "doc_values": true     // We sort by this (default)
      }
    }
  }
}
```

**Doc values** are column-oriented storage for sorting/aggregations. Disable for fields you only search, never sort.

---

## 5. Index Lifecycle Management (ILM)

```json
PUT _ilm/policy/jobs-policy
{
  "policy": {
    "phases": {
      "hot": {
        "min_age": "0ms",
        "actions": {
          "rollover": {
            "max_size": "30GB",
            "max_age": "30d"
          }
        }
      },
      "warm": {
        "min_age": "30d",
        "actions": {
          "shrink": {
            "number_of_shards": 1
          },
          "forcemerge": {
            "max_num_segments": 1
          }
        }
      },
      "cold": {
        "min_age": "90d",
        "actions": {
          "freeze": {}
        }
      },
      "delete": {
        "min_age": "365d",
        "actions": {
          "delete": {}
        }
      }
    }
  }
}
```

### Lifecycle Stages

```
Hot (0-30 days):    Full resources, 3 shards, fast queries
     ↓
Warm (30-90 days):  Shrink to 1 shard, merge segments
     ↓
Cold (90-365 days): Freeze (read-only, minimal resources)
     ↓
Delete (>365 days): Remove from ES (still in PostgreSQL)
```

---

## 6. Bulk Indexing Settings

For initial load or reindexing:

```json
PUT /jobs-v1/_settings
{
  "index": {
    "refresh_interval": "-1",        // Disable during bulk load
    "number_of_replicas": 0          // Disable replicas during bulk
  }
}

// After bulk load complete:
PUT /jobs-v1/_settings
{
  "index": {
    "refresh_interval": "5s",
    "number_of_replicas": 1
  }
}

// Force merge to optimize
POST /jobs-v1/_forcemerge?max_num_segments=1
```

---

## 7. Circuit Breaker Configuration

Prevent out-of-memory errors:

```yaml
# elasticsearch.yml
indices.breaker.total.limit: 70%
indices.breaker.fielddata.limit: 40%
indices.breaker.request.limit: 40%
```

---

## 8. Thread Pool Tuning

```yaml
# elasticsearch.yml
thread_pool:
  search:
    size: 13          # CPUs + 1
    queue_size: 1000  # Backpressure
  write:
    size: 5
    queue_size: 200
```

---

## 9. Query Cache Settings

```json
PUT /jobs-v1/_settings
{
  "index.queries.cache.enabled": true
}
```

Cluster-level:
```yaml
indices.queries.cache.size: 10%
```

---

## Before and After Comparison

| Metric | Before Tuning | After Tuning |
|--------|---------------|--------------|
| Index size | 20GB | 15GB (removed unused fields) |
| Query latency P50 | 80ms | 40ms |
| Query latency P99 | 300ms | 120ms |
| Indexing rate | 2000 docs/s | 5000 docs/s |
| Segment count | 50+ | 10-15 |

---

## Common Tuning Mistakes

| Mistake | Problem | Fix |
|---------|---------|-----|
| Too many shards | Overhead per shard | Start small, scale up |
| text for IDs | Wasted analysis, can't exact match | Use keyword |
| Dynamic mapping on | Unknown fields indexed | Set dynamic: strict |
| No ILM | Old data uses same resources | Implement ILM |
| Refresh = 1s during bulk | Slow bulk indexing | Disable during bulk |

---

## Interview Talking Points

1. **"What index tuning did you do?"**
   > "Several things: We set 3 shards based on data size projections. We increased refresh interval to 5s since dashboard can tolerate slight staleness. We used keyword type for filter fields to skip analysis. We implemented ILM to move old data to cheaper storage."

2. **"How did you determine shard count?"**
   > "Rule of thumb is 10-50GB per shard. We had 15GB with projections to 70GB in a year. Three shards gives us good parallelization and keeps each shard under 25GB long-term. We also ensured shards spread across nodes."

3. **"What's the trade-off with refresh interval?"**
   > "Lower refresh interval means more segments created, which slows queries and requires more merging. Higher interval delays searchability. For a dashboard where 5-second staleness is fine, we chose 5s as a balance."
