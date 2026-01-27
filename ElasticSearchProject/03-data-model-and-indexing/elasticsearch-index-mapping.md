# Elasticsearch Index Mapping

## Index Name Convention

```
jobs-v1        # Version 1 of the mapping
jobs-v2        # Version 2 after schema changes
jobs           # Alias pointing to current version
```

Using aliases allows zero-downtime reindexing.

---

## Complete Mapping

```json
PUT /jobs-v1
{
  "settings": {
    "number_of_shards": 3,
    "number_of_replicas": 1,
    "refresh_interval": "5s",
    "index.mapping.total_fields.limit": 100,

    "analysis": {
      "analyzer": {
        "lowercase_keyword": {
          "type": "custom",
          "tokenizer": "keyword",
          "filter": ["lowercase"]
        }
      }
    }
  },

  "mappings": {
    "properties": {

      "id": {
        "type": "keyword"
      },

      "userId": {
        "type": "keyword"
      },

      "userName": {
        "type": "keyword"
      },

      "userEmail": {
        "type": "keyword",
        "normalizer": "lowercase"
      },

      "workspaceId": {
        "type": "keyword"
      },

      "status": {
        "type": "keyword"
      },

      "type": {
        "type": "keyword"
      },

      "projectName": {
        "type": "keyword",
        "fields": {
          "search": {
            "type": "text",
            "analyzer": "lowercase_keyword"
          }
        }
      },

      "hostname": {
        "type": "keyword",
        "fields": {
          "search": {
            "type": "text",
            "analyzer": "standard"
          }
        }
      },

      "arch": {
        "type": "keyword"
      },

      "errorMessage": {
        "type": "text",
        "analyzer": "standard",
        "fields": {
          "keyword": {
            "type": "keyword",
            "ignore_above": 256
          }
        }
      },

      "createdAt": {
        "type": "date",
        "format": "strict_date_optional_time||epoch_millis"
      },

      "updatedAt": {
        "type": "date",
        "format": "strict_date_optional_time||epoch_millis"
      },

      "completedAt": {
        "type": "date",
        "format": "strict_date_optional_time||epoch_millis"
      },

      "durationMs": {
        "type": "long"
      },

      "metadata": {
        "type": "object",
        "enabled": false
      },

      "version": {
        "type": "long"
      }
    }
  }
}
```

---

## Field Type Decisions

### keyword vs text

| Field | Type | Reason |
|-------|------|--------|
| status | keyword | Exact match only: `status = 'FAILED'` |
| projectName | keyword + text | Exact filter AND partial search |
| errorMessage | text | Full-text search for errors |
| userId | keyword | Always exact match (UUID) |

### Multi-field Pattern

```json
"projectName": {
  "type": "keyword",              // For exact match: term query
  "fields": {
    "search": {
      "type": "text",             // For partial match: match query
      "analyzer": "lowercase_keyword"
    }
  }
}
```

Usage:
```json
// Exact match (filter)
{ "term": { "projectName": "backend-api" } }

// Partial match (search)
{ "match": { "projectName.search": "backend" } }
```

---

## Date Handling

```json
"createdAt": {
  "type": "date",
  "format": "strict_date_optional_time||epoch_millis"
}
```

Accepts:
- ISO 8601: `"2024-01-15T10:30:00Z"`
- Epoch milliseconds: `1705315800000`

### Range Query Example

```json
{
  "range": {
    "createdAt": {
      "gte": "2024-01-01T00:00:00Z",
      "lte": "2024-01-31T23:59:59Z"
    }
  }
}
```

---

## Disabled Object Mapping

```json
"metadata": {
  "type": "object",
  "enabled": false  // Stored but not indexed
}
```

**Why?**
- metadata is for storage/retrieval only
- Indexing arbitrary JSON creates mapping explosion
- If we need to query metadata, we extract specific fields

---

## Index Alias Setup

```json
POST /_aliases
{
  "actions": [
    { "add": { "index": "jobs-v1", "alias": "jobs" } }
  ]
}
```

Application always queries `jobs` alias, not versioned index.

---

## Java Document Class

```java
@Document(indexName = "jobs")
public class JobDocument {

    @Id
    private String id;

    @Field(type = FieldType.Keyword)
    private String userId;

    @Field(type = FieldType.Keyword)
    private String userName;

    @Field(type = FieldType.Keyword)
    private String workspaceId;

    @Field(type = FieldType.Keyword)
    private String status;

    @Field(type = FieldType.Keyword)
    private String type;

    @Field(type = FieldType.Keyword)
    @MultiField(
        mainField = @Field(type = FieldType.Keyword),
        otherFields = {
            @InnerField(suffix = "search", type = FieldType.Text)
        }
    )
    private String projectName;

    @Field(type = FieldType.Keyword)
    private String hostname;

    @Field(type = FieldType.Keyword)
    private String arch;

    @Field(type = FieldType.Text)
    private String errorMessage;

    @Field(type = FieldType.Date, format = DateFormat.date_optional_time)
    private Instant createdAt;

    @Field(type = FieldType.Date, format = DateFormat.date_optional_time)
    private Instant updatedAt;

    @Field(type = FieldType.Date, format = DateFormat.date_optional_time)
    private Instant completedAt;

    @Field(type = FieldType.Long)
    private Long durationMs;

    @Field(type = FieldType.Long)
    private Long version;
}
```

---

## Common Mapping Mistakes to Avoid

| Mistake | Problem | Fix |
|---------|---------|-----|
| Using `text` for IDs | Can't do exact match, wastes space | Use `keyword` |
| Dynamic mapping on | Unknown fields indexed, mapping explosion | Set `dynamic: strict` |
| No date format specified | Inconsistent parsing | Always specify format |
| Large `ignore_above` | Memory issues | Keep at 256-512 |

---

## Interview Talking Point

> "I designed the mapping with query patterns in mind. Fields used for filtering are `keyword` type for exact matches. Fields needing partial search have a multi-field with `text` sub-field. Dates use explicit formats for consistency. We disabled indexing on the metadata object to prevent mapping explosion from arbitrary JSON."
