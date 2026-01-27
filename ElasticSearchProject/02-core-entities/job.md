# Job Entity

## Overview

The `Job` entity represents any task execution in the system - workspace creation, deletion, image builds, or CI test runs. It's the **primary entity indexed in Elasticsearch**.

---

## Entity Definition

```java
@Entity
@Table(name = "job")
public class Job {

    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    private String id;

    @Column(nullable = false)
    @Enumerated(EnumType.STRING)
    private JobStatus status;

    @Column(nullable = false)
    private String projectName;

    @Column(nullable = false)
    private String userId;

    @Column
    private String workspaceId;  // Nullable for non-workspace jobs

    @Column
    private String hostname;

    @Column
    private String arch;  // java, python, node

    @Column
    @Enumerated(EnumType.STRING)
    private JobType type;  // WORKSPACE_CREATE, WORKSPACE_DELETE, IMAGE_BUILD, CI_TEST

    @Column(columnDefinition = "TEXT")
    private String errorMessage;  // Populated on failure

    @Column(nullable = false)
    private Instant createdAt;

    @Column(nullable = false)
    private Instant updatedAt;

    @Column
    private Instant completedAt;

    @Version
    private Long version;  // Optimistic locking
}
```

---

## Status Lifecycle

```
                    ┌──────────────┐
                    │   SCHEDULED  │
                    └──────┬───────┘
                           │
                           ▼
                    ┌──────────────┐
           ┌────────│ IN_PROGRESS  │────────┐
           │        └──────────────┘        │
           │                                │
           ▼                                ▼
    ┌──────────────┐                 ┌──────────────┐
    │   COMPLETED  │                 │    FAILED    │
    └──────────────┘                 └──────────────┘
                                            │
                                            ▼
                                     ┌──────────────┐
                                     │   RETRYING   │ (goes back to IN_PROGRESS)
                                     └──────────────┘
```

---

## JobStatus Enum

```java
public enum JobStatus {
    SCHEDULED,      // Job created, waiting to be picked up
    IN_PROGRESS,    // Worker processing
    COMPLETED,      // Successfully finished
    FAILED,         // Failed with error
    RETRYING,       // Retry in progress
    CANCELLED       // User cancelled
}
```

---

## JobType Enum

```java
public enum JobType {
    WORKSPACE_CREATE,    // Creating new workspace container
    WORKSPACE_DELETE,    // Deleting workspace
    IMAGE_BUILD,         // Building Docker image
    CI_TEST              // Running CI/test job
}
```

---

## Why This Entity Design?

| Decision | Reason |
|----------|--------|
| UUID for id | Distributed systems, no sequence conflicts |
| Enum as STRING | Readable in DB and ES, easier debugging |
| errorMessage as TEXT | Stack traces can be long |
| version field | Prevent concurrent update issues |
| Separate createdAt/updatedAt/completedAt | Different query patterns for each |

---

## Interview Talking Points

1. **Why not embed workspace details in job?**
   - Jobs are immutable records of what happened
   - Workspace can change after job completes
   - Keeps job table focused and queryable

2. **Why store status as STRING not INT?**
   - ES doesn't benefit from INT
   - Debugging is easier
   - Adding new statuses is safer

3. **What's the version field for?**
   - Optimistic locking via JPA
   - Prevents race conditions when multiple services update same job
