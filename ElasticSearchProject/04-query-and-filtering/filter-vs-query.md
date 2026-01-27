# Filter Context vs Query Context

## The Two Contexts in Elasticsearch

Elasticsearch has two execution contexts that behave differently:

| Aspect | Query Context | Filter Context |
|--------|---------------|----------------|
| Purpose | "How well does this match?" | "Does this match? Yes/No" |
| Scoring | Calculates relevance score | No scoring (faster) |
| Caching | Not cached | Cached in filter cache |
| Use case | Full-text search | Structured filtering |

---

## Visual Comparison

```
Query Context (Scoring):
┌─────────────────────────────────────────┐
│ "Find documents matching 'connection    │
│  timeout' in errorMessage"              │
│                                         │
│ Document A: "connection timeout error"  │  Score: 8.5
│ Document B: "timeout on connection"     │  Score: 7.2
│ Document C: "connection failed"         │  Score: 4.1
└─────────────────────────────────────────┘

Filter Context (Boolean):
┌─────────────────────────────────────────┐
│ "Find documents where status = FAILED"  │
│                                         │
│ Document A: status=FAILED               │  Match: YES
│ Document B: status=COMPLETED            │  Match: NO
│ Document C: status=FAILED               │  Match: YES
└─────────────────────────────────────────┘
```

---

## Bool Query Structure

```json
{
  "query": {
    "bool": {
      "must": [
        // Query context - contributes to score
        { "match": { "errorMessage": "connection timeout" } }
      ],
      "filter": [
        // Filter context - no scoring, cached
        { "term": { "status": "FAILED" } },
        { "range": { "createdAt": { "gte": "2024-01-01" } } }
      ],
      "should": [
        // Query context - optional, boosts score
        { "term": { "arch": "java" } }
      ],
      "must_not": [
        // Filter context - excludes documents
        { "term": { "type": "CI_TEST" } }
      ]
    }
  }
}
```

---

## When to Use Each

### Use Filter Context (90% of dashboard queries)

```json
// Exact match on status
{ "filter": [{ "term": { "status": "FAILED" } }] }

// Date range
{ "filter": [{ "range": { "createdAt": { "gte": "2024-01-01" } } }] }

// Multiple values (IN clause)
{ "filter": [{ "terms": { "status": ["FAILED", "IN_PROGRESS"] } }] }

// Existence check
{ "filter": [{ "exists": { "field": "errorMessage" } }] }
```

### Use Query Context (Full-text search)

```json
// Search in error messages
{ "must": [{ "match": { "errorMessage": "connection timeout" } }] }

// Phrase matching
{ "must": [{ "match_phrase": { "errorMessage": "connection refused" } }] }

// Fuzzy matching (typo tolerance)
{ "must": [{ "fuzzy": { "projectName": { "value": "bakend", "fuzziness": 1 } } }] }
```

---

## Our Dashboard: Primarily Filter Context

```java
public SearchResponse searchJobs(JobSearchRequest request) {
    BoolQueryBuilder boolQuery = QueryBuilders.boolQuery();

    // All structured filters go in filter context
    if (request.getStatuses() != null && !request.getStatuses().isEmpty()) {
        boolQuery.filter(QueryBuilders.termsQuery("status", request.getStatuses()));
    }

    if (request.getProjectName() != null) {
        boolQuery.filter(QueryBuilders.termQuery("projectName", request.getProjectName()));
    }

    if (request.getCreatedAfter() != null) {
        boolQuery.filter(QueryBuilders.rangeQuery("createdAt")
            .gte(request.getCreatedAfter().toEpochMilli()));
    }

    if (request.getCreatedBefore() != null) {
        boolQuery.filter(QueryBuilders.rangeQuery("createdAt")
            .lte(request.getCreatedBefore().toEpochMilli()));
    }

    // Only use query context for text search
    if (StringUtils.hasText(request.getSearchTerm())) {
        boolQuery.must(QueryBuilders.multiMatchQuery(request.getSearchTerm(),
            "errorMessage", "hostname.search", "projectName.search"));
    }

    // Build search request
    SearchRequest searchRequest = new SearchRequest("jobs");
    SearchSourceBuilder sourceBuilder = new SearchSourceBuilder()
        .query(boolQuery)
        .sort("createdAt", SortOrder.DESC)
        .size(request.getPageSize());

    return esClient.search(searchRequest, RequestOptions.DEFAULT);
}
```

---

## Filter Cache: Why It Matters

```
First request:   status = 'FAILED'
                      ↓
                 Execute filter
                      ↓
                 Build bitset: [1,0,1,1,0,0,1,...]
                      ↓
                 Cache bitset
                      ↓
                 Return results (100ms)

Second request:  status = 'FAILED' AND project = 'backend-api'
                      ↓
                 Get cached bitset for status
                 AND with project filter
                      ↓
                 Return results (20ms)  ← Much faster!
```

### Cache Behavior

| Filter Type | Cached? | Reason |
|-------------|---------|--------|
| term / terms | Yes | Exact match, stable |
| range (date) | Depends | Cached if range is "rounded" |
| exists | Yes | Stable |
| range (now) | No | Changes every request |

**Tip:** Use fixed date ranges instead of relative:
```json
// BAD - not cached
{ "range": { "createdAt": { "gte": "now-7d" } } }

// GOOD - can be cached
{ "range": { "createdAt": { "gte": "2024-01-08T00:00:00Z" } } }
```

---

## Interview Talking Points

1. **"Why do you use filter context?"**
   > "For our dashboard, we don't need relevance scoring. Users filter by exact status, project, or date range. Filter context is faster because ES can cache the results as bitsets and skip scoring calculations."

2. **"When would you use query context?"**
   > "When users search for text in error messages or partial project names. That's where we need relevance ranking - documents with better matches should appear first."

3. **"How does caching help performance?"**
   > "ES caches filter results as bitsets. When users apply additional filters, ES can AND the bitsets together instead of re-evaluating. This is why our common filter combinations are so fast - they're hitting cache."
