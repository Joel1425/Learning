# Write Path: Database to Elasticsearch

## Overview

This document explains how data flows from service writes through PostgreSQL to Elasticsearch.

---

## Complete Write Path Diagram

```
┌──────────────────────────────────────────────────────────────────────────┐
│                           WRITE PATH                                     │
└──────────────────────────────────────────────────────────────────────────┘

Step 1: Client Request
┌─────────────┐
│   Client    │
│   (CLI)     │
└──────┬──────┘
       │ POST /v1/users/{userId}/workspaces
       ▼
┌─────────────────┐
│   API Gateway   │
│ (AWS + LB)      │
└────────┬────────┘
         │
         ▼
Step 2: Service Processing
┌─────────────────────────────────────────────────────────────┐
│                      A4C Service                            │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ @Transactional                                      │   │
│  │ public Workspace createWorkspace(request) {         │   │
│  │                                                     │   │
│  │   // 1. Create workspace record                     │   │
│  │   Workspace ws = workspaceRepo.save(workspace);     │   │
│  │                                                     │   │
│  │   // 2. Create job record                           │   │
│  │   Job job = new Job();                              │   │
│  │   job.setStatus(SCHEDULED);                         │   │
│  │   job.setType(WORKSPACE_CREATE);                    │   │
│  │   jobRepo.save(job);                                │   │
│  │                                                     │   │
│  │   // 3. Publish event (after commit)                │   │
│  │   eventPublisher.publish(new JobCreatedEvent(job)); │   │
│  │                                                     │   │
│  │   return ws;                                        │   │
│  │ }                                                   │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
Step 3: Database Write
┌─────────────────────────────────────────────────────────────┐
│                     PostgreSQL                              │
│                                                             │
│  ┌─────────────┐              ┌─────────────┐              │
│  │  workspace  │              │    job      │              │
│  ├─────────────┤              ├─────────────┤              │
│  │ id: ws-123  │◄────────────│ id: job-456 │              │
│  │ status:     │  workspace  │ workspace_id│              │
│  │  CREATING   │  _id FK     │ status:     │              │
│  │ ...         │              │  SCHEDULED  │              │
│  └─────────────┘              └─────────────┘              │
│         │                            │                      │
│         │   PRIMARY                  │                      │
│         ▼                            ▼                      │
│  ┌──────────────────────────────────────────┐              │
│  │            WAL Replication               │              │
│  └──────────────────────────────────────────┘              │
│                      │                                      │
│         ┌────────────┼────────────┐                        │
│         ▼            ▼            ▼                        │
│    ┌─────────┐  ┌─────────┐  ┌─────────┐                  │
│    │Replica 1│  │Replica 2│  │Replica 3│                  │
│    └─────────┘  └─────────┘  └─────────┘                  │
│                                                             │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            │ Transaction Committed
                            ▼
Step 4: Event Publishing
┌─────────────────────────────────────────────────────────────┐
│             Spring TransactionSynchronization               │
│                                                             │
│  @TransactionalEventListener(phase = AFTER_COMMIT)         │
│  void onJobCreated(JobCreatedEvent event) {                │
│      kafkaTemplate.send("job-es-sync", message);           │
│  }                                                          │
│                                                             │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
Step 5: Kafka
┌─────────────────────────────────────────────────────────────┐
│                        Kafka                                │
│                                                             │
│  Topic: job-es-sync                                        │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ Partition 0: [msg1] [msg2] [msg3] ...                │  │
│  │ Partition 1: [msg4] [msg5] ...                       │  │
│  │ Partition 2: [msg6] ...                              │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                             │
│  Key: jobId (ensures same job goes to same partition)      │
│                                                             │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
Step 6: ES Sync Consumer
┌─────────────────────────────────────────────────────────────┐
│                   ES Sync Consumer                          │
│                                                             │
│  @KafkaListener(topics = "job-es-sync")                    │
│  void handleSync(JobSyncMessage msg) {                      │
│                                                             │
│      // 1. Fetch latest from PostgreSQL                    │
│      Job job = jobRepo.findById(msg.getJobId());           │
│                                                             │
│      // 2. Transform to ES document                        │
│      JobDocument doc = mapper.toDocument(job);             │
│                                                             │
│      // 3. Index to Elasticsearch                          │
│      esOperations.save(doc);                               │
│                                                             │
│      // 4. Mark as synced                                  │
│      jobRepo.updateEsSyncedAt(job.getId(), now());         │
│  }                                                          │
│                                                             │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
Step 7: Elasticsearch
┌─────────────────────────────────────────────────────────────┐
│                    Elasticsearch                            │
│                                                             │
│  Index: jobs                                                │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ {                                                    │  │
│  │   "id": "job-456",                                   │  │
│  │   "workspaceId": "ws-123",                           │  │
│  │   "status": "SCHEDULED",                             │  │
│  │   "type": "WORKSPACE_CREATE",                        │  │
│  │   "projectName": "backend-api",                      │  │
│  │   "createdAt": "2024-01-15T10:30:00Z"               │  │
│  │ }                                                    │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                             │
│  Shards: [Shard 0] [Shard 1] [Shard 2]                    │
│  Replicas: 1 per shard                                     │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Sequence Diagram

```
Client      Gateway     A4C Service    PostgreSQL    Kafka    ES Consumer    Elasticsearch
  │            │             │              │          │           │              │
  │──POST ────▶│             │              │          │           │              │
  │            │──forward───▶│              │          │           │              │
  │            │             │              │          │           │              │
  │            │             │──BEGIN TX───▶│          │           │              │
  │            │             │──INSERT ws──▶│          │           │              │
  │            │             │◀─────OK──────│          │           │              │
  │            │             │──INSERT job─▶│          │           │              │
  │            │             │◀─────OK──────│          │           │              │
  │            │             │──COMMIT─────▶│          │           │              │
  │            │             │◀─────OK──────│          │           │              │
  │            │             │              │          │           │              │
  │            │             │ (after commit listener)  │           │              │
  │            │             │─────────────────────────▶│           │              │
  │            │             │              │          │           │              │
  │◀───201────│◀────201─────│              │          │           │              │
  │            │             │              │          │           │              │
  │            │             │              │          │──consume─▶│              │
  │            │             │              │          │           │──GET job────▶│
  │            │             │              │◀─────────────────────│              │
  │            │             │              │─────job data────────▶│              │
  │            │             │              │          │           │──INDEX──────▶│
  │            │             │              │          │           │◀────OK───────│
  │            │             │              │◀─update sync_at──────│              │
  │            │             │              │          │           │              │
```

---

## Key Design Decisions

### 1. Why AFTER_COMMIT for Event Publishing?

```java
// WRONG: Event sent before commit
@Transactional
public Job createJob(...) {
    Job job = jobRepo.save(job);
    eventPublisher.publish(new JobCreatedEvent(job));  // Sent immediately
    throw new RuntimeException("Oops");  // Transaction rolls back
    // But event was already sent! Consumer can't find job.
}

// CORRECT: Event sent after commit
@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
public void onJobCreated(JobCreatedEvent event) {
    kafkaTemplate.send("job-es-sync", message);  // Only if TX succeeded
}
```

### 2. Why Fetch from DB in Consumer (Not Use Event Payload)?

```java
// WRONG: Use event data directly
void handleSync(JobSyncMessage msg) {
    JobDocument doc = msg.getJobDocument();  // May be stale!
    esOperations.save(doc);
}

// CORRECT: Fetch latest from DB
void handleSync(JobSyncMessage msg) {
    Job job = jobRepo.findById(msg.getJobId());  // Get current state
    JobDocument doc = mapper.toDocument(job);
    esOperations.save(doc);
}
```

**Why?** Job may have been updated multiple times before consumer processes message. We want the latest state.

### 3. Why Kafka as Intermediary?

| Direct ES Write | Via Kafka |
|-----------------|-----------|
| Simpler | More complex |
| Tight coupling | Decoupled |
| ES downtime = service failure | ES downtime = messages queue |
| No retry built-in | Kafka handles retries |
| No ordering guarantees | Partition ordering |

---

## Status Update Flow

When a job status changes (e.g., SCHEDULED → IN_PROGRESS → COMPLETED):

```java
@Service
public class JobExecutionService {

    @Transactional
    public void markInProgress(UUID jobId) {
        Job job = jobRepo.findById(jobId).orElseThrow();
        job.setStatus(JobStatus.IN_PROGRESS);
        jobRepo.save(job);

        eventPublisher.publishEvent(new JobUpdatedEvent(job.getId()));
    }

    @Transactional
    public void markCompleted(UUID jobId, JobResult result) {
        Job job = jobRepo.findById(jobId).orElseThrow();
        job.setStatus(result.isSuccess() ? JobStatus.COMPLETED : JobStatus.FAILED);
        job.setCompletedAt(Instant.now());
        if (!result.isSuccess()) {
            job.setErrorMessage(result.getError());
        }
        jobRepo.save(job);

        eventPublisher.publishEvent(new JobUpdatedEvent(job.getId()));
    }
}
```

Each status change triggers an ES sync, so the dashboard reflects current state.

---

## Interview Talking Points

1. **"Walk me through the write path."**
   > "When a service creates or updates a job, it writes to PostgreSQL first. After the transaction commits, a Spring event listener publishes to Kafka. Our ES sync consumer picks up the message, fetches the latest job state from PostgreSQL, and indexes it to Elasticsearch. This ensures PostgreSQL is always the source of truth."

2. **"Why not write directly to ES from the service?"**
   > "That would create tight coupling and consistency issues. If ES is down, our service would fail. With Kafka in between, messages queue up during ES downtime and process when ES recovers."

3. **"What if the Kafka send fails?"**
   > "If Kafka send fails after DB commit, we have a backup poller that scans for jobs where updated_at > es_synced_at. It catches anything that slipped through."
