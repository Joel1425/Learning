# Entity Relationships

## ER Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  ┌─────────┐         ┌─────────────┐         ┌──────────────┐  │
│  │  User   │ 1────N  │  Workspace  │ 1────N  │     Job      │  │
│  │         │         │             │         │              │  │
│  │ - id    │         │ - id        │         │ - id         │  │
│  │ - email │         │ - userId    │         │ - workspaceId│  │
│  │ - name  │         │ - projectId │         │ - userId     │  │
│  └─────────┘         │ - status    │         │ - status     │  │
│       │              └─────────────┘         │ - type       │  │
│       │                     │                │ - createdAt  │  │
│       │                     │                └──────────────┘  │
│       │                     │                       │          │
│       │              ┌──────┴──────┐                │          │
│       │              │             │                │          │
│       │              ▼             ▼                ▼          │
│       │        ┌─────────┐  ┌────────────┐  ┌─────────────┐   │
│       └───────▶│ Image   │  │BuildSchedule│  │   (ES Doc)  │   │
│         1──N   │         │  │            │  │             │   │
│                │ - id    │  │ - id       │  │ Denormalized│   │
│                │ - name  │  │ - imageId  │  │ Job + User  │   │
│                │ - arch  │  │ - status   │  │ + Workspace │   │
│                └─────────┘  └────────────┘  └─────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Relationship Cardinality

| Relationship | Type | Description |
|--------------|------|-------------|
| User → Workspace | 1:N | User owns multiple workspaces |
| User → Job | 1:N | User's jobs (direct or via workspace) |
| Workspace → Job | 1:N | Operations performed on workspace |
| User → Image | 1:N | Custom images built by user |
| Image → BuildSchedule | 1:N | Scheduled builds for an image |

---

## PostgreSQL Schema

```sql
-- Users table (simplified)
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) UNIQUE NOT NULL,
    name VARCHAR(255) NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

-- Workspaces table
CREATE TABLE workspace (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id),
    project_name VARCHAR(255) NOT NULL,
    parent_name VARCHAR(255),
    arch VARCHAR(50) NOT NULL,
    hostname VARCHAR(255),
    container_ip VARCHAR(50),
    docker_image_id VARCHAR(255),
    status VARCHAR(50) NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    last_accessed_at TIMESTAMP,

    INDEX idx_workspace_user (user_id),
    INDEX idx_workspace_status (status),
    INDEX idx_workspace_project (project_name)
);

-- Jobs table (primary table for ES sync)
CREATE TABLE job (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id UUID REFERENCES workspace(id),
    user_id UUID NOT NULL REFERENCES users(id),
    status VARCHAR(50) NOT NULL,
    type VARCHAR(50) NOT NULL,
    project_name VARCHAR(255) NOT NULL,
    hostname VARCHAR(255),
    arch VARCHAR(50),
    error_message TEXT,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW(),
    completed_at TIMESTAMP,
    version BIGINT NOT NULL DEFAULT 0,

    INDEX idx_job_status (status),
    INDEX idx_job_user (user_id),
    INDEX idx_job_created (created_at DESC),
    INDEX idx_job_workspace (workspace_id)
);

-- Trigger for updated_at
CREATE OR REPLACE FUNCTION update_updated_at()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER job_updated_at
    BEFORE UPDATE ON job
    FOR EACH ROW
    EXECUTE FUNCTION update_updated_at();
```

---

## Why Denormalize for Elasticsearch?

### PostgreSQL Approach (Normalized)

```sql
-- To get job with user and workspace details
SELECT j.*, u.name as user_name, w.hostname
FROM job j
JOIN users u ON j.user_id = u.id
LEFT JOIN workspace w ON j.workspace_id = w.id
WHERE j.status = 'FAILED';
```

**Problem:** JOINs don't scale for complex filters and high throughput.

### Elasticsearch Approach (Denormalized)

```json
{
  "jobId": "job-123",
  "userId": "user-456",
  "userName": "John Doe",
  "workspaceId": "ws-789",
  "workspaceHostname": "worker-03",
  "projectName": "backend-api",
  "status": "FAILED",
  "type": "WORKSPACE_CREATE",
  "createdAt": "2024-01-15T10:30:00Z"
}
```

**Benefits:**
- Single document contains all queryable fields
- No runtime JOINs
- O(1) lookup via inverted index

---

## Consistency Trade-offs

| Scenario | Impact | Mitigation |
|----------|--------|------------|
| User renames themselves | userName in ES is stale | Re-index affected jobs OR accept staleness |
| Workspace hostname changes | ES has old hostname | New jobs will have correct value |
| Job deleted from PostgreSQL | Orphan doc in ES | Cleanup job removes stale docs |

**Interview Note:** "We accepted eventual consistency because the dashboard is informational. For transactional accuracy, we query PostgreSQL directly."
