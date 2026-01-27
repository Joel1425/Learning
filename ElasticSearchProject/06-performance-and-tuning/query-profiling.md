# Query Profiling

## The Resume Claim

> "...query profiling..."

This document explains how we profiled and optimized slow queries.

---

## The Profile API

Elasticsearch has a built-in profiler that shows exactly what happens during query execution.

```json
GET /jobs/_search
{
  "profile": true,
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

---

## Understanding Profile Output

```json
{
  "profile": {
    "shards": [
      {
        "id": "[node1][jobs][0]",
        "searches": [
          {
            "query": [
              {
                "type": "BooleanQuery",
                "description": "+status:FAILED +projectName:backend-api +createdAt:[1704067200000 TO *]",
                "time_in_nanos": 12500000,
                "breakdown": {
                  "set_min_competitive_score_count": 0,
                  "match_count": 1523,
                  "shallow_advance_count": 0,
                  "set_min_competitive_score": 0,
                  "next_doc": 8500000,
                  "match": 1200000,
                  "next_doc_count": 1523,
                  "score_count": 0,
                  "compute_max_score_count": 0,
                  "compute_max_score": 0,
                  "advance": 500000,
                  "advance_count": 15,
                  "score": 0,
                  "build_scorer_count": 8,
                  "create_weight": 1500000,
                  "shallow_advance": 0,
                  "create_weight_count": 1,
                  "build_scorer": 800000
                },
                "children": [
                  {
                    "type": "TermQuery",
                    "description": "status:FAILED",
                    "time_in_nanos": 3200000
                  },
                  {
                    "type": "TermQuery",
                    "description": "projectName:backend-api",
                    "time_in_nanos": 2800000
                  },
                  {
                    "type": "PointRangeQuery",
                    "description": "createdAt:[1704067200000 TO *]",
                    "time_in_nanos": 4100000
                  }
                ]
              }
            ]
          }
        ]
      }
    ]
  }
}
```

---

## Key Metrics to Watch

| Metric | What It Means | Good Value | Bad Sign |
|--------|---------------|------------|----------|
| time_in_nanos | Total time for this query part | <10ms | >50ms |
| next_doc_count | Documents iterated | Matches result count | Much higher than results |
| advance_count | Skip operations | Low | High = inefficient |
| build_scorer | Time to build scorer | <1ms | >5ms |
| score_count | Documents scored | 0 for filters | High = using query context |

---

## Real Profiling Example: Slow Query

### Before Optimization

```json
GET /jobs/_search
{
  "profile": true,
  "query": {
    "bool": {
      "must": [
        { "wildcard": { "projectName": "*backend*" } },
        { "match": { "errorMessage": "timeout" } }
      ],
      "filter": [
        { "term": { "status": "FAILED" } }
      ]
    }
  }
}
```

**Profile result:**
```json
{
  "type": "BooleanQuery",
  "time_in_nanos": 450000000,  // 450ms - TOO SLOW!
  "children": [
    {
      "type": "WildcardQuery",
      "description": "projectName:*backend*",
      "time_in_nanos": 380000000  // 380ms - culprit!
    },
    {
      "type": "TermQuery",
      "description": "status:FAILED",
      "time_in_nanos": 5000000  // 5ms - fine
    },
    {
      "type": "MatchQuery",
      "description": "errorMessage:timeout",
      "time_in_nanos": 45000000  // 45ms - acceptable
    }
  ]
}
```

**Problem:** Leading wildcard `*backend*` forces full index scan.

### After Optimization

```json
GET /jobs/_search
{
  "profile": true,
  "query": {
    "bool": {
      "filter": [
        { "term": { "status": "FAILED" } },
        { "match": { "projectName.search": "backend" } }
      ],
      "must": [
        { "match": { "errorMessage": "timeout" } }
      ]
    }
  }
}
```

**Profile result:**
```json
{
  "type": "BooleanQuery",
  "time_in_nanos": 25000000,  // 25ms - 18x faster!
  "children": [
    {
      "type": "TermQuery",
      "description": "status:FAILED",
      "time_in_nanos": 3000000
    },
    {
      "type": "MatchQuery",
      "description": "projectName.search:backend",
      "time_in_nanos": 8000000  // Uses analyzed field
    },
    {
      "type": "MatchQuery",
      "description": "errorMessage:timeout",
      "time_in_nanos": 12000000
    }
  ]
}
```

---

## Common Slow Query Patterns

### 1. Leading Wildcards

```json
// SLOW - scans entire index
{ "wildcard": { "projectName": "*api*" } }

// FAST - use ngram or match
{ "match": { "projectName.search": "api" } }
```

### 2. Large Date Ranges with High Cardinality

```json
// SLOW - too many buckets
{
  "aggs": {
    "daily": {
      "date_histogram": {
        "field": "createdAt",
        "interval": "minute"  // 1440 buckets per day!
      }
    }
  }
}

// FAST - reasonable granularity
{
  "aggs": {
    "daily": {
      "date_histogram": {
        "field": "createdAt",
        "interval": "day"
      }
    }
  }
}
```

### 3. Scripts in Queries

```json
// SLOW - runtime calculation
{
  "query": {
    "script": {
      "script": "doc['duration'].value > 1000"
    }
  }
}

// FAST - pre-computed field
{
  "query": {
    "range": { "isSlowJob": { "gte": true } }
  }
}
```

### 4. Deep Pagination

```json
// SLOW - loads all docs in memory
{
  "from": 10000,
  "size": 20
}

// FAST - cursor-based
{
  "size": 20,
  "search_after": [1704067200000, "job-123"]
}
```

---

## Profiling in Java

```java
@Service
public class QueryProfiler {

    private final RestHighLevelClient esClient;

    public ProfileResult profileQuery(SearchRequest request) {
        // Enable profiling
        request.source().profile(true);

        SearchResponse response = esClient.search(request, RequestOptions.DEFAULT);

        // Extract profile data
        Map<String, ProfileShardResult> profileResults = response.getProfileResults();

        // Analyze
        for (Map.Entry<String, ProfileShardResult> entry : profileResults.entrySet()) {
            String shardKey = entry.getKey();
            ProfileShardResult shardProfile = entry.getValue();

            for (QueryProfileShardResult queryProfile : shardProfile.getQueryProfileResults()) {
                analyzeQueryProfile(queryProfile);
            }
        }

        return summarize(profileResults);
    }

    private void analyzeQueryProfile(QueryProfileShardResult profile) {
        for (ProfileResult result : profile.getQueryResults()) {
            long timeNanos = result.getTime();
            String queryType = result.getQueryName();

            if (timeNanos > 50_000_000) { // >50ms
                log.warn("Slow query component: {} took {}ms",
                    queryType, timeNanos / 1_000_000);
            }
        }
    }
}
```

---

## Monitoring Slow Queries

### Slow Log Configuration

```yaml
# elasticsearch.yml
index.search.slowlog.threshold.query.warn: 1s
index.search.slowlog.threshold.query.info: 500ms
index.search.slowlog.threshold.query.debug: 100ms
index.search.slowlog.threshold.fetch.warn: 500ms
```

### Sample Slow Log Entry

```
[2024-01-15T10:30:45,123][WARN][i.s.s.query] [node1] [jobs][0]
took[1.2s], took_millis[1234], total_hits[45023],
types[], stats[], search_type[QUERY_THEN_FETCH],
total_shards[3], source[{"query":{"wildcard":{"projectName":"*api*"}}}]
```

---

## Optimization Checklist

```
□ Use filter context for non-scored queries
□ Avoid leading wildcards
□ Use search_after instead of deep pagination
□ Pre-compute values instead of runtime scripts
□ Check shard distribution (all shards contributing?)
□ Verify filter cache is being used
□ Appropriate aggregation granularity
□ Profile regularly after changes
```

---

## Interview Talking Points

1. **"How did you profile queries?"**
   > "We used Elasticsearch's Profile API to see execution breakdown. It shows time spent in each query component. We identified a leading wildcard query taking 400ms and replaced it with a match query on an analyzed field, reducing to 25ms."

2. **"What patterns did you find problematic?"**
   > "Leading wildcards were the biggest issue - they force full index scans. Deep pagination was another - we switched to search_after. Also, some queries were in query context when they should have been filter context, missing the cache benefits."

3. **"How do you monitor for slow queries in production?"**
   > "We configured ES slow logs to capture queries over 100ms. We also added custom metrics in our Java service to track P95/P99 latency. When we see degradation, we pull recent slow queries and profile them."
