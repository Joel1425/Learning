# Workspace Entity

## Overview

The `Workspace` entity represents a developer's containerized environment. It's managed by the A4C service and has a lifecycle tracked through jobs.

---

## Entity Definition

```java
@Entity
@Table(name = "workspace")
public class Workspace {

    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    private String id;

    @Column(nullable = false)
    private String userId;

    @Column(nullable = false)
    private String projectName;

    @Column
    private String parentName;  // Parent project for inheritance

    @Column(nullable = false)
    private String arch;  // java, python, node

    @Column
    private String hostname;  // Assigned after creation

    @Column
    private String containerIp;

    @Column
    private String dockerImageId;

    @Column(nullable = false)
    @Enumerated(EnumType.STRING)
    private WorkspaceStatus status;

    @Column(nullable = false)
    private Instant createdAt;

    @Column
    private Instant lastAccessedAt;

    @OneToMany(mappedBy = "workspaceId", fetch = FetchType.LAZY)
    private List<Job> jobs;  // History of operations on this workspace
}
```

---

## WorkspaceStatus Enum

```java
public enum WorkspaceStatus {
    CREATING,       // Creation job in progress
    CREATED,        // Ready to use
    STARTING,       // Container starting up
    RUNNING,        // Container active
    STOPPED,        // Container stopped
    DELETING,       // Deletion in progress
    DELETED,        // Soft delete (record kept)
    FAILED          // Creation or operation failed
}
```

---

## Status Lifecycle

```
    ┌───────────┐
    │ CREATING  │◄──────────────────────┐
    └─────┬─────┘                       │
          │                             │
          ▼                             │
    ┌───────────┐     ┌───────────┐     │
    │  CREATED  │────▶│ STARTING  │     │
    └─────┬─────┘     └─────┬─────┘     │
          │                 │           │
          │                 ▼           │
          │           ┌───────────┐     │
          │           │  RUNNING  │     │
          │           └─────┬─────┘     │
          │                 │           │
          │                 ▼           │
          │           ┌───────────┐     │
          │           │  STOPPED  │─────┘ (restart)
          │           └─────┬─────┘
          │                 │
          ▼                 ▼
    ┌───────────┐     ┌───────────┐
    │ DELETING  │────▶│  DELETED  │
    └───────────┘     └───────────┘

    Any state can transition to FAILED
```

---

## Workspace vs Job: Key Differences

| Aspect | Workspace | Job |
|--------|-----------|-----|
| Nature | Mutable resource | Immutable event record |
| Lifecycle | Long-lived | Short-lived |
| Updates | Status changes over time | Status set once at completion |
| ES indexing | Optional (for workspace search) | Primary (for dashboard) |

---

## Why Workspaces are NOT Primary in ES

For the screener/dashboard use case:
- Users filter by **job** status (FAILED, IN_PROGRESS)
- Jobs have the execution details (when, what happened)
- Workspace is just context for the job

```
Dashboard Query: "Show failed workspace creations"

Wrong approach: Query workspaces with status=FAILED
Right approach: Query jobs with type=WORKSPACE_CREATE, status=FAILED
```

---

## Denormalization for ES

When indexing a job, we denormalize workspace info:

```json
{
  "jobId": "job-123",
  "workspaceId": "ws-456",
  "workspaceName": "my-dev-env",      // Denormalized
  "workspaceHostname": "worker-03",   // Denormalized
  "projectName": "backend-api",
  "status": "FAILED"
}
```

**Trade-off:** If workspace hostname changes, ES doc is stale until job is re-indexed or new job created.
