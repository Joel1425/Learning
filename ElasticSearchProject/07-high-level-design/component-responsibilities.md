# Component Responsibilities

## Overview

Each component in the system has specific responsibilities. This document details what each component does and doesn't do.

---

## 1. API Gateway

### Responsibilities
```
✓ Route requests to appropriate backend service
✓ Rate limiting (prevent abuse)
✓ Authentication (validate JWT tokens)
✓ SSL/TLS termination
✓ Request logging
✓ CORS handling
```

### Non-Responsibilities
```
✗ Business logic
✗ Data transformation
✗ Caching application data
✗ Database access
```

### Configuration Example
```yaml
# AWS API Gateway + Route53
routes:
  - path: /v1/users/*/workspaces
    method: [GET, POST, DELETE]
    backend: a4c-service
    rateLimit: 100/minute
    auth: required

  - path: /v1/jobs/search
    method: POST
    backend: a4c-service
    rateLimit: 200/minute
    auth: required
```

---

## 2. A4C Service (Main Application Service)

### Responsibilities
```
✓ Workspace CRUD operations
✓ Job management
✓ Image scheduling
✓ Dashboard search (via Elasticsearch)
✓ Coordinate with Disk Service
✓ Publish events to Kafka
✓ Apply business rules and validations
```

### Non-Responsibilities
```
✗ Actually running containers (delegates to Docker daemon)
✗ Image building (delegates to Image Builder daemon)
✗ Disk management (delegates to Disk Service)
✗ Direct ES writes (delegates to ES Sync Consumer)
```

### Key Classes

```java
@RestController
@RequestMapping("/v1/users/{userId}")
public class WorkspaceController {
    // Handles workspace API endpoints
}

@Service
public class WorkspaceService {
    // Business logic for workspace operations

    @Transactional
    public Workspace createWorkspace(CreateWorkspaceRequest request) {
        // 1. Validate request
        // 2. Find best server via Disk Service
        // 3. Create workspace record
        // 4. Create job record
        // 5. Initiate container creation
        // 6. Return workspace
    }
}

@Service
public class JobSearchService {
    // Handles dashboard search

    public SearchResponse search(JobSearchRequest request) {
        try {
            return searchElasticsearch(request);
        } catch (Exception e) {
            return searchPostgres(request);  // Fallback
        }
    }
}
```

---

## 3. Disk Service

### Responsibilities
```
✓ Track disk usage across servers
✓ Find server with least disk usage
✓ Report server capacity
✓ Health check on servers
```

### Non-Responsibilities
```
✗ Container management
✗ Workspace operations
✗ Database access for workspaces
```

### API Contract

```java
@FeignClient(name = "disk-service")
public interface DiskServiceClient {

    @GetMapping("/servers/best-available")
    Server findBestServer(@RequestParam String arch,
                          @RequestParam long requiredDiskMb);

    @GetMapping("/servers/{hostname}/disk-usage")
    DiskUsage getDiskUsage(@PathVariable String hostname);

    @GetMapping("/servers")
    List<Server> getServers(@RequestParam String imageId);
}
```

---

## 4. PostgreSQL

### Responsibilities
```
✓ Store all application data (source of truth)
✓ ACID transactions
✓ Foreign key constraints
✓ Data integrity
✓ Backup and recovery
```

### Non-Responsibilities
```
✗ Fast complex queries (delegated to ES)
✗ Full-text search (delegated to ES)
✗ Real-time analytics
```

### Tables Overview

| Table | Purpose | Access Pattern |
|-------|---------|----------------|
| users | User accounts | Read-heavy |
| workspace | Workspace metadata | Read/Write balanced |
| job | Job execution records | Write-heavy, read via ES |
| image | Docker image metadata | Read-heavy |
| build_schedule | Image build schedules | Write then read |

---

## 5. Elasticsearch

### Responsibilities
```
✓ Fast dashboard queries
✓ Complex AND/OR filtering
✓ Full-text search on errors
✓ Aggregations for stats
✓ Near real-time search
```

### Non-Responsibilities
```
✗ Source of truth (that's PostgreSQL)
✗ Transaction support
✗ Relational joins
✗ Write operations from application
```

### Index Overview

```json
{
  "index": "jobs",
  "purpose": "Dashboard filtering and search",
  "documents": 10000000,
  "shards": 3,
  "replicas": 1,
  "refresh": "5s"
}
```

---

## 6. Kafka

### Responsibilities
```
✓ Reliable message delivery
✓ Decouple producers and consumers
✓ Handle backpressure
✓ Message ordering (per partition)
✓ Replay capability
```

### Non-Responsibilities
```
✗ Message transformation
✗ Complex routing logic
✗ Long-term storage
```

### Topics

| Topic | Purpose | Partitions | Retention |
|-------|---------|------------|-----------|
| job-es-sync | Sync jobs to ES | 6 | 7 days |
| workspace-events | Workspace lifecycle events | 6 | 7 days |
| build-events | Image build notifications | 3 | 7 days |

---

## 7. ES Sync Consumer

### Responsibilities
```
✓ Consume sync messages from Kafka
✓ Fetch latest state from PostgreSQL
✓ Index documents to Elasticsearch
✓ Handle retries and failures
✓ Update sync status in PostgreSQL
```

### Non-Responsibilities
```
✗ Business logic
✗ Direct API access
✗ Writing to PostgreSQL (except sync status)
```

### Implementation

```java
@Service
public class EsSyncConsumer {

    @KafkaListener(topics = "job-es-sync", groupId = "es-sync")
    public void handleSync(JobSyncMessage message) {
        Job job = jobRepository.findById(message.getJobId())
            .orElse(null);

        if (job == null) {
            esOperations.delete(message.getJobId().toString(), JobDocument.class);
            return;
        }

        JobDocument doc = mapper.toDocument(job);
        esOperations.save(doc);
        jobRepository.updateEsSyncedAt(job.getId(), Instant.now());
    }
}
```

---

## 8. Docker Daemon (on Bare Metal)

### Responsibilities
```
✓ Run containers
✓ Manage container lifecycle
✓ Allocate resources (CPU, memory)
✓ Network configuration
```

### Non-Responsibilities
```
✗ Application-level logic
✗ Database access
✗ User authentication
```

---

## 9. Image Builder Daemon

### Responsibilities
```
✓ Poll for scheduled builds
✓ Build Docker images from templates
✓ Push images to registry
✓ Notify A4C Service on completion
```

### Non-Responsibilities
```
✗ Scheduling (done by A4C Service)
✗ User management
✗ Workspace operations
```

### Flow

```
1. Poll database for SCHEDULED builds
2. Mark as IN_PROGRESS (transactional)
3. Execute Dockerfile build
4. Push to registry
5. Call A4C API to mark COMPLETED
6. A4C sends email notification
```

---

## 10. Cleanup Daemon

### Responsibilities
```
✓ Find orphan containers
✓ Remove containers not in database
✓ Clean up failed workspace artifacts
✓ Report orphan counts
```

### Non-Responsibilities
```
✗ Normal workspace deletion (A4C Service does this)
✗ Database cleanup
✗ ES cleanup
```

---

## Component Interaction Matrix

| From \ To | Gateway | A4C | Disk | PostgreSQL | ES | Kafka | Docker |
|-----------|---------|-----|------|------------|-----|-------|--------|
| Client | ✓ | - | - | - | - | - | - |
| Gateway | - | ✓ | - | - | - | - | - |
| A4C | - | - | ✓ | ✓ | ✓ | ✓ | ✓ |
| Disk | - | - | - | - | - | - | ✓ |
| ES Consumer | - | - | - | ✓ | ✓ | ✓ | - |
| Builder | - | ✓ | - | ✓ | - | - | ✓ |
| Cleanup | - | ✓ | - | - | - | - | ✓ |

---

## Interview Talking Point

> "Each component has a single responsibility. A4C Service handles business logic but delegates infrastructure concerns. PostgreSQL is the source of truth; Elasticsearch is for queries. Kafka decouples the sync process so A4C doesn't need to wait for ES. The daemons on bare metal servers handle the actual container operations, keeping the application layer infrastructure-agnostic."
