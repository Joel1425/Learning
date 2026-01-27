# Sample Elasticsearch Queries

## Common Dashboard Queries

### 1. Failed Workspace Creations (Last 7 Days)

**Use Case:** DevOps wants to see recent failures to investigate.

```json
GET /jobs/_search
{
  "query": {
    "bool": {
      "filter": [
        { "term": { "status": "FAILED" } },
        { "term": { "type": "WORKSPACE_CREATE" } },
        { "range": { "createdAt": { "gte": "now-7d" } } }
      ]
    }
  },
  "sort": [
    { "createdAt": { "order": "desc" } }
  ],
  "size": 50
}
```

**Java:**
```java
BoolQueryBuilder query = QueryBuilders.boolQuery()
    .filter(QueryBuilders.termQuery("status", "FAILED"))
    .filter(QueryBuilders.termQuery("type", "WORKSPACE_CREATE"))
    .filter(QueryBuilders.rangeQuery("createdAt").gte("now-7d"));
```

---

### 2. User's Active Jobs

**Use Case:** User views their dashboard with in-progress items.

```json
GET /jobs/_search
{
  "query": {
    "bool": {
      "filter": [
        { "term": { "userId": "user-123-uuid" } },
        { "terms": { "status": ["SCHEDULED", "IN_PROGRESS"] } }
      ]
    }
  },
  "sort": [
    { "createdAt": { "order": "desc" } }
  ],
  "size": 20
}
```

---

### 3. Jobs by Multiple Projects (OR)

**Use Case:** Team lead views jobs across their team's projects.

```json
GET /jobs/_search
{
  "query": {
    "bool": {
      "filter": [
        { "terms": { "projectName": ["backend-api", "frontend-app", "data-pipeline"] } },
        { "range": { "createdAt": { "gte": "2024-01-01" } } }
      ]
    }
  },
  "aggs": {
    "by_status": {
      "terms": { "field": "status" }
    },
    "by_project": {
      "terms": { "field": "projectName" }
    }
  },
  "size": 100
}
```

---

### 4. Full-Text Search in Errors

**Use Case:** Engineer searches for specific error pattern.

```json
GET /jobs/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "match": {
            "errorMessage": {
              "query": "connection refused timeout",
              "operator": "and"
            }
          }
        }
      ],
      "filter": [
        { "term": { "status": "FAILED" } }
      ]
    }
  },
  "highlight": {
    "fields": {
      "errorMessage": {}
    }
  },
  "size": 20
}
```

**Response with highlights:**
```json
{
  "hits": {
    "hits": [
      {
        "_source": {
          "id": "job-123",
          "errorMessage": "Database connection refused after timeout of 30s"
        },
        "highlight": {
          "errorMessage": [
            "Database <em>connection</em> <em>refused</em> after <em>timeout</em> of 30s"
          ]
        }
      }
    ]
  }
}
```

---

### 5. Complex Dashboard Filter

**Use Case:** Advanced filter with multiple conditions.

```
Criteria:
- Status: FAILED or IN_PROGRESS
- Type: WORKSPACE_CREATE or IMAGE_BUILD
- Project: starts with "backend"
- Created: Last 30 days
- Arch: java
```

```json
GET /jobs/_search
{
  "query": {
    "bool": {
      "filter": [
        { "terms": { "status": ["FAILED", "IN_PROGRESS"] } },
        { "terms": { "type": ["WORKSPACE_CREATE", "IMAGE_BUILD"] } },
        { "prefix": { "projectName": "backend" } },
        { "range": { "createdAt": { "gte": "now-30d" } } },
        { "term": { "arch": "java" } }
      ]
    }
  },
  "sort": [
    { "createdAt": { "order": "desc" } }
  ],
  "size": 50
}
```

---

### 6. Aggregations for Dashboard Stats

**Use Case:** Show summary counts by status.

```json
GET /jobs/_search
{
  "query": {
    "bool": {
      "filter": [
        { "term": { "projectName": "backend-api" } },
        { "range": { "createdAt": { "gte": "now-7d" } } }
      ]
    }
  },
  "aggs": {
    "status_counts": {
      "terms": { "field": "status" }
    },
    "type_counts": {
      "terms": { "field": "type" }
    },
    "daily_trend": {
      "date_histogram": {
        "field": "createdAt",
        "calendar_interval": "day"
      },
      "aggs": {
        "by_status": {
          "terms": { "field": "status" }
        }
      }
    }
  },
  "size": 0
}
```

**Response:**
```json
{
  "aggregations": {
    "status_counts": {
      "buckets": [
        { "key": "COMPLETED", "doc_count": 1523 },
        { "key": "FAILED", "doc_count": 45 },
        { "key": "IN_PROGRESS", "doc_count": 12 }
      ]
    },
    "daily_trend": {
      "buckets": [
        {
          "key_as_string": "2024-01-15",
          "doc_count": 234,
          "by_status": {
            "buckets": [
              { "key": "COMPLETED", "doc_count": 220 },
              { "key": "FAILED", "doc_count": 14 }
            ]
          }
        }
      ]
    }
  }
}
```

---

### 7. Cursor-Based Pagination (search_after)

**Use Case:** Efficient pagination for large result sets.

**First Page:**
```json
GET /jobs/_search
{
  "query": {
    "bool": {
      "filter": [
        { "term": { "status": "COMPLETED" } }
      ]
    }
  },
  "sort": [
    { "createdAt": { "order": "desc" } },
    { "id": { "order": "asc" } }
  ],
  "size": 20
}
```

**Next Page (using last result's sort values):**
```json
GET /jobs/_search
{
  "query": {
    "bool": {
      "filter": [
        { "term": { "status": "COMPLETED" } }
      ]
    }
  },
  "sort": [
    { "createdAt": { "order": "desc" } },
    { "id": { "order": "asc" } }
  ],
  "search_after": [1705315800000, "job-abc-123"],
  "size": 20
}
```

**Java:**
```java
SearchSourceBuilder sourceBuilder = new SearchSourceBuilder()
    .query(query)
    .sort("createdAt", SortOrder.DESC)
    .sort("id", SortOrder.ASC)
    .size(20);

if (request.getSearchAfter() != null) {
    sourceBuilder.searchAfter(request.getSearchAfter().toArray());
}
```

---

### 8. Exists Query (Find Jobs with Errors)

**Use Case:** Find all jobs that have error messages.

```json
GET /jobs/_search
{
  "query": {
    "bool": {
      "filter": [
        { "exists": { "field": "errorMessage" } }
      ]
    }
  },
  "size": 50
}
```

---

### 9. Negation (Exclude Certain Types)

**Use Case:** Show all jobs except CI tests.

```json
GET /jobs/_search
{
  "query": {
    "bool": {
      "filter": [
        { "term": { "projectName": "backend-api" } }
      ],
      "must_not": [
        { "term": { "type": "CI_TEST" } }
      ]
    }
  },
  "size": 50
}
```

---

## Performance Notes

| Query Type | Typical Latency | Notes |
|------------|-----------------|-------|
| Single term filter | 5-10ms | Hits filter cache |
| Multiple term filters | 10-20ms | Bitset AND operations |
| Range + term filters | 15-30ms | Range may not cache |
| Full-text search | 30-100ms | Scoring overhead |
| Aggregations | 50-200ms | Depends on cardinality |

---

## Common Mistakes

| Mistake | Problem | Fix |
|---------|---------|-----|
| Using match for exact values | Tokenizes input, wrong results | Use term/terms |
| Wildcard at start | Full index scan | Use ngram analyzer or reconsider |
| Deep pagination (from: 10000) | Memory issues | Use search_after |
| Not using filter context | Misses caching | Put non-scoring queries in filter |
