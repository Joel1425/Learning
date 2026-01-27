# AND/OR Compound Queries

## The Resume Bullet

> "...supporting complex AND/OR compound queries..."

This section explains exactly what that means and how we implemented it.

---

## SQL to Elasticsearch Translation

### Simple AND

```sql
-- SQL
SELECT * FROM job
WHERE status = 'FAILED'
  AND project_name = 'backend-api';
```

```json
// Elasticsearch
{
  "query": {
    "bool": {
      "filter": [
        { "term": { "status": "FAILED" } },
        { "term": { "projectName": "backend-api" } }
      ]
    }
  }
}
```

---

### Simple OR

```sql
-- SQL
SELECT * FROM job
WHERE status = 'FAILED'
   OR status = 'IN_PROGRESS';
```

```json
// Elasticsearch - using terms (preferred for same field)
{
  "query": {
    "bool": {
      "filter": [
        { "terms": { "status": ["FAILED", "IN_PROGRESS"] } }
      ]
    }
  }
}

// OR using should with minimum_should_match
{
  "query": {
    "bool": {
      "should": [
        { "term": { "status": "FAILED" } },
        { "term": { "status": "IN_PROGRESS" } }
      ],
      "minimum_should_match": 1
    }
  }
}
```

---

### Complex Compound (AND + OR)

```sql
-- SQL
SELECT * FROM job
WHERE (status = 'FAILED' OR status = 'IN_PROGRESS')
  AND project_name = 'backend-api'
  AND created_at > '2024-01-01';
```

```json
// Elasticsearch
{
  "query": {
    "bool": {
      "filter": [
        {
          "bool": {
            "should": [
              { "term": { "status": "FAILED" } },
              { "term": { "status": "IN_PROGRESS" } }
            ],
            "minimum_should_match": 1
          }
        },
        { "term": { "projectName": "backend-api" } },
        { "range": { "createdAt": { "gte": "2024-01-01T00:00:00Z" } } }
      ]
    }
  }
}
```

---

### Nested OR Groups

```sql
-- SQL
SELECT * FROM job
WHERE (status = 'FAILED' AND type = 'WORKSPACE_CREATE')
   OR (status = 'COMPLETED' AND type = 'IMAGE_BUILD');
```

```json
// Elasticsearch
{
  "query": {
    "bool": {
      "should": [
        {
          "bool": {
            "filter": [
              { "term": { "status": "FAILED" } },
              { "term": { "type": "WORKSPACE_CREATE" } }
            ]
          }
        },
        {
          "bool": {
            "filter": [
              { "term": { "status": "COMPLETED" } },
              { "term": { "type": "IMAGE_BUILD" } }
            ]
          }
        }
      ],
      "minimum_should_match": 1
    }
  }
}
```

---

## Java Query Builder

```java
@Service
public class JobSearchQueryBuilder {

    public BoolQueryBuilder buildQuery(JobSearchRequest request) {
        BoolQueryBuilder rootQuery = QueryBuilders.boolQuery();

        // Status filter (OR within same field)
        if (request.getStatuses() != null && !request.getStatuses().isEmpty()) {
            rootQuery.filter(QueryBuilders.termsQuery("status",
                request.getStatuses().stream()
                    .map(Enum::name)
                    .collect(Collectors.toList())));
        }

        // Type filter
        if (request.getTypes() != null && !request.getTypes().isEmpty()) {
            rootQuery.filter(QueryBuilders.termsQuery("type",
                request.getTypes().stream()
                    .map(Enum::name)
                    .collect(Collectors.toList())));
        }

        // Project filter (exact or partial)
        if (StringUtils.hasText(request.getProjectName())) {
            if (request.isExactMatch()) {
                rootQuery.filter(QueryBuilders.termQuery("projectName",
                    request.getProjectName()));
            } else {
                rootQuery.filter(QueryBuilders.matchQuery("projectName.search",
                    request.getProjectName()));
            }
        }

        // Date range
        if (request.getCreatedAfter() != null || request.getCreatedBefore() != null) {
            RangeQueryBuilder rangeQuery = QueryBuilders.rangeQuery("createdAt");
            if (request.getCreatedAfter() != null) {
                rangeQuery.gte(request.getCreatedAfter().toEpochMilli());
            }
            if (request.getCreatedBefore() != null) {
                rangeQuery.lte(request.getCreatedBefore().toEpochMilli());
            }
            rootQuery.filter(rangeQuery);
        }

        // Complex condition groups (if supported)
        if (request.getConditionGroups() != null) {
            for (ConditionGroup group : request.getConditionGroups()) {
                rootQuery.filter(buildConditionGroup(group));
            }
        }

        return rootQuery;
    }

    private QueryBuilder buildConditionGroup(ConditionGroup group) {
        BoolQueryBuilder groupQuery = QueryBuilders.boolQuery();

        for (Condition condition : group.getConditions()) {
            QueryBuilder conditionQuery = buildCondition(condition);

            if (group.getOperator() == LogicalOperator.AND) {
                groupQuery.filter(conditionQuery);
            } else { // OR
                groupQuery.should(conditionQuery);
            }
        }

        if (group.getOperator() == LogicalOperator.OR) {
            groupQuery.minimumShouldMatch(1);
        }

        return groupQuery;
    }

    private QueryBuilder buildCondition(Condition condition) {
        return switch (condition.getOperator()) {
            case EQUALS -> QueryBuilders.termQuery(condition.getField(), condition.getValue());
            case NOT_EQUALS -> QueryBuilders.boolQuery()
                .mustNot(QueryBuilders.termQuery(condition.getField(), condition.getValue()));
            case IN -> QueryBuilders.termsQuery(condition.getField(), condition.getValues());
            case GREATER_THAN -> QueryBuilders.rangeQuery(condition.getField())
                .gt(condition.getValue());
            case LESS_THAN -> QueryBuilders.rangeQuery(condition.getField())
                .lt(condition.getValue());
            case CONTAINS -> QueryBuilders.matchQuery(condition.getField() + ".search",
                condition.getValue());
        };
    }
}
```

---

## Request/Response DTOs

```java
@Data
public class JobSearchRequest {
    private List<JobStatus> statuses;
    private List<JobType> types;
    private String projectName;
    private boolean exactMatch = true;
    private Instant createdAfter;
    private Instant createdBefore;
    private List<ConditionGroup> conditionGroups;

    private String sortField = "createdAt";
    private SortOrder sortOrder = SortOrder.DESC;
    private int pageSize = 20;
    private List<Object> searchAfter;  // For pagination
}

@Data
public class ConditionGroup {
    private LogicalOperator operator;  // AND, OR
    private List<Condition> conditions;
}

@Data
public class Condition {
    private String field;
    private ComparisonOperator operator;  // EQUALS, IN, GREATER_THAN, etc.
    private Object value;
    private List<Object> values;  // For IN operator
}

public enum LogicalOperator { AND, OR }
public enum ComparisonOperator { EQUALS, NOT_EQUALS, IN, GREATER_THAN, LESS_THAN, CONTAINS }
```

---

## Frontend Request Example

```json
POST /api/v1/jobs/search

{
  "statuses": ["FAILED", "IN_PROGRESS"],
  "types": ["WORKSPACE_CREATE"],
  "projectName": "backend-api",
  "createdAfter": "2024-01-01T00:00:00Z",
  "conditionGroups": [
    {
      "operator": "OR",
      "conditions": [
        { "field": "arch", "operator": "EQUALS", "value": "java" },
        { "field": "arch", "operator": "EQUALS", "value": "python" }
      ]
    }
  ],
  "sortField": "createdAt",
  "sortOrder": "DESC",
  "pageSize": 20
}
```

This translates to:
```
(status IN ('FAILED', 'IN_PROGRESS'))
AND (type = 'WORKSPACE_CREATE')
AND (project_name = 'backend-api')
AND (created_at > '2024-01-01')
AND (arch = 'java' OR arch = 'python')
```

---

## Interview Talking Points

1. **"How do you handle complex AND/OR?"**
   > "Elasticsearch's bool query has must, should, and filter clauses. For AND conditions, we add to filter (for caching) or must (for scoring). For OR, we use should with minimum_should_match=1. We can nest bool queries for complex groupings."

2. **"Why use terms instead of multiple term queries with should?"**
   > "For OR within the same field, terms is more efficient - single lookup in the inverted index vs multiple. should with minimum_should_match is for OR across different fields."

3. **"How does the frontend specify complex conditions?"**
   > "We designed a flexible request DTO that supports condition groups. Each group has an operator (AND/OR) and a list of conditions. The backend recursively builds the nested bool query. This keeps the API generic without exposing ES internals."
