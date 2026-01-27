# PostgreSQL Job Table

## Complete Schema

```sql
CREATE TABLE job (
    -- Primary Key
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

    -- Foreign Keys
    workspace_id UUID REFERENCES workspace(id) ON DELETE SET NULL,
    user_id UUID NOT NULL REFERENCES users(id),

    -- Core Fields
    status VARCHAR(50) NOT NULL,
    type VARCHAR(50) NOT NULL,

    -- Searchable Fields (denormalized for ES)
    project_name VARCHAR(255) NOT NULL,
    hostname VARCHAR(255),
    arch VARCHAR(50),
    user_email VARCHAR(255),  -- Denormalized from users table

    -- Details
    error_message TEXT,
    metadata JSONB,  -- Flexible additional data

    -- Timestamps
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    completed_at TIMESTAMP WITH TIME ZONE,

    -- Optimistic Locking
    version BIGINT NOT NULL DEFAULT 0,

    -- Sync Tracking
    es_synced_at TIMESTAMP WITH TIME ZONE,
    es_sync_version BIGINT DEFAULT 0
);
```

---

## Indexes for PostgreSQL Queries

```sql
-- Status-based queries (fallback when ES is down)
CREATE INDEX idx_job_status ON job(status);

-- User's jobs
CREATE INDEX idx_job_user_id ON job(user_id);

-- Time-based queries
CREATE INDEX idx_job_created_at ON job(created_at DESC);

-- Composite for common dashboard query
CREATE INDEX idx_job_status_created ON job(status, created_at DESC);

-- For sync process: find unsynced jobs
CREATE INDEX idx_job_es_sync ON job(updated_at)
    WHERE es_synced_at IS NULL OR es_synced_at < updated_at;
```

---

## Sync Tracking Columns

| Column | Purpose |
|--------|---------|
| es_synced_at | Timestamp of last successful ES sync |
| es_sync_version | Version number at last sync (compare with version) |

### Sync Detection Query

```sql
-- Find jobs that need syncing
SELECT * FROM job
WHERE es_synced_at IS NULL
   OR es_synced_at < updated_at
   OR es_sync_version < version
ORDER BY updated_at ASC
LIMIT 1000;
```

---

## Why These Design Decisions?

### 1. UUID vs Auto-Increment

```
UUID Pros:
- No coordination needed across services
- Can generate client-side
- Merge-friendly for distributed writes

UUID Cons:
- Larger (16 bytes vs 4-8 bytes)
- Random = poor index locality (use UUIDv7 for time-ordering)
```

**Our choice:** UUID because multiple services write jobs concurrently.

### 2. JSONB metadata Column

```sql
-- Flexible storage for type-specific data
UPDATE job SET metadata = '{
    "dockerImageId": "sha256:abc123",
    "cpuCores": 4,
    "memoryGb": 8,
    "buildDurationMs": 45000
}'::jsonb WHERE id = 'job-123';

-- Query JSONB (PostgreSQL)
SELECT * FROM job
WHERE metadata->>'cpuCores' = '4';
```

**Trade-off:** Less type safety, but avoids schema changes for new fields.

### 3. Denormalized user_email

```sql
-- Instead of JOINing users table every time
-- We store email directly in job table
INSERT INTO job (user_id, user_email, ...)
VALUES ('user-123', 'john@example.com', ...);
```

**Trade-off:** Stale if user changes email. Acceptable because:
- Email changes are rare
- Historical accuracy (email at time of job) may be desired

---

## JPA Entity Mapping

```java
@Entity
@Table(name = "job", indexes = {
    @Index(name = "idx_job_status", columnList = "status"),
    @Index(name = "idx_job_user_id", columnList = "user_id"),
    @Index(name = "idx_job_created_at", columnList = "created_at DESC")
})
@EntityListeners(AuditingEntityListener.class)
public class Job {

    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    private UUID id;

    @Column(name = "user_id", nullable = false)
    private UUID userId;

    @Column(name = "workspace_id")
    private UUID workspaceId;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false, length = 50)
    private JobStatus status;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false, length = 50)
    private JobType type;

    @Column(name = "project_name", nullable = false)
    private String projectName;

    @Column
    private String hostname;

    @Column(length = 50)
    private String arch;

    @Column(name = "error_message", columnDefinition = "TEXT")
    private String errorMessage;

    @Type(JsonType.class)
    @Column(columnDefinition = "jsonb")
    private Map<String, Object> metadata;

    @CreatedDate
    @Column(name = "created_at", nullable = false, updatable = false)
    private Instant createdAt;

    @LastModifiedDate
    @Column(name = "updated_at", nullable = false)
    private Instant updatedAt;

    @Column(name = "completed_at")
    private Instant completedAt;

    @Version
    private Long version;

    @Column(name = "es_synced_at")
    private Instant esSyncedAt;
}
```

---

## Interview Questions About This Design

**Q: Why not use PostgreSQL full-text search?**

A: PostgreSQL FTS is good for text search but:
- Complex AND/OR on structured fields is still slow
- No inverted index for enums/keywords
- Can't match ES query DSL flexibility
- Scaling reads requires replicas; ES scales horizontally

**Q: How do you handle schema migrations?**

A:
- Flyway for PostgreSQL DDL changes
- ES mapping updates via reindexing (covered in indexing-strategy.md)
- Feature flags for backward compatibility during migration
