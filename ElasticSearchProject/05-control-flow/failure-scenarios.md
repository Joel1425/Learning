# Failure Scenarios and Handling

## Overview

This document covers what can go wrong and how we handle it.

---

## Failure Matrix

| Component | Failure Mode | Impact | Mitigation |
|-----------|--------------|--------|------------|
| PostgreSQL Primary | Down | Writes fail | Promote replica |
| PostgreSQL Replica | Down | Read scaling reduced | Other replicas |
| Kafka | Broker down | Message queuing delayed | Multi-broker cluster |
| ES Consumer | Crash | Sync delayed | Consumer group rebalance |
| Elasticsearch | Cluster down | Queries use fallback | PostgreSQL fallback |
| ES Node | Single node down | Reduced capacity | Replica shards |
| Network | Partition | Depends on partition | Retry + circuit breaker |

---

## Scenario 1: Elasticsearch Cluster Down

```
┌─────────────────────────────────────────────────────────────┐
│                    SCENARIO: ES DOWN                        │
└─────────────────────────────────────────────────────────────┘

Normal flow:
Frontend ──▶ Backend ──▶ Elasticsearch ──▶ Response
                              ✗ DOWN

Fallback flow:
Frontend ──▶ Backend ──▶ PostgreSQL (replica) ──▶ Response
                              ✓ (slower)
```

### Implementation

```java
@Service
@Slf4j
public class JobSearchService {

    private final CircuitBreaker esCircuitBreaker;

    @PostConstruct
    void initCircuitBreaker() {
        esCircuitBreaker = CircuitBreaker.of("elasticsearch",
            CircuitBreakerConfig.custom()
                .failureRateThreshold(50)
                .waitDurationInOpenState(Duration.ofSeconds(30))
                .slidingWindowSize(10)
                .build());
    }

    public SearchResponse search(JobSearchRequest request) {
        return esCircuitBreaker.executeSupplier(() -> {
            try {
                return searchElasticsearch(request);
            } catch (ElasticsearchException e) {
                log.warn("ES search failed: {}", e.getMessage());
                throw e;  // Let circuit breaker track
            }
        }, throwable -> {
            // Fallback when circuit is open or ES fails
            log.info("Using PostgreSQL fallback for search");
            metrics.increment("search.fallback.postgres");
            return searchPostgres(request);
        });
    }
}
```

### User Experience

```
ES Healthy:    Response in ~50ms
ES Degraded:   Response in ~80ms (retries)
ES Down:       Response in ~200ms (PostgreSQL fallback)
                + Banner: "Search results may be delayed"
```

---

## Scenario 2: Sync Consumer Failure

```
┌─────────────────────────────────────────────────────────────┐
│              SCENARIO: CONSUMER CRASH                       │
└─────────────────────────────────────────────────────────────┘

Timeline:
T0: Consumer 1 processing message for job-123
T1: Consumer 1 crashes before committing offset
T2: Kafka rebalances, Consumer 2 picks up partition
T3: Consumer 2 reprocesses message (idempotent)
T4: Job-123 indexed to ES
```

### Why Idempotency Matters

```java
@KafkaListener(topics = "job-es-sync")
public void handleSync(JobSyncMessage message) {
    // Fetch current state from DB
    Job job = jobRepository.findById(message.getJobId()).orElse(null);

    if (job == null) {
        // Job was deleted, remove from ES
        esOperations.delete(message.getJobId().toString(), JobDocument.class);
        return;
    }

    // Compare versions to avoid overwriting newer data
    JobDocument existing = esOperations.get(
        message.getJobId().toString(), JobDocument.class);

    if (existing != null && existing.getVersion() >= job.getVersion()) {
        log.debug("Skipping stale update for job {}", job.getId());
        return;  // Already have newer or same version
    }

    // Index the document
    JobDocument doc = mapper.toDocument(job);
    esOperations.save(doc);
}
```

---

## Scenario 3: Message Lost (Kafka Send Fails)

```
┌─────────────────────────────────────────────────────────────┐
│              SCENARIO: KAFKA SEND FAILS                     │
└─────────────────────────────────────────────────────────────┘

Timeline:
T0: Job created in PostgreSQL, TX committed
T1: Event listener tries to send to Kafka
T2: Kafka send fails (network issue)
T3: Job exists in PostgreSQL but not in ES
T4: Backup poller detects unsynced job
T5: Poller sends to Kafka (now healthy)
T6: Consumer indexes to ES
```

### Backup Poller

```java
@Component
@Slf4j
public class SyncRecoveryPoller {

    private final JobRepository jobRepository;
    private final KafkaTemplate<String, JobSyncMessage> kafka;

    // Run every 30 seconds
    @Scheduled(fixedDelay = 30_000)
    public void recoverUnsyncedJobs() {
        // Find jobs that haven't been synced in last 5 minutes
        Instant threshold = Instant.now().minus(Duration.ofMinutes(5));

        List<Job> unsynced = jobRepository.findJobsNeedingSync(threshold,
            PageRequest.of(0, 100));

        if (unsynced.isEmpty()) {
            return;
        }

        log.info("Found {} unsynced jobs, queuing for sync", unsynced.size());

        for (Job job : unsynced) {
            try {
                JobSyncMessage msg = new JobSyncMessage(
                    job.getId(),
                    SyncAction.INDEX,
                    Instant.now()
                );
                kafka.send("job-es-sync", job.getId().toString(), msg);
            } catch (Exception e) {
                log.error("Failed to queue job {} for sync", job.getId(), e);
            }
        }
    }
}
```

---

## Scenario 4: Elasticsearch Indexing Fails

```
┌─────────────────────────────────────────────────────────────┐
│              SCENARIO: ES INDEX FAILS                       │
└─────────────────────────────────────────────────────────────┘

Possible causes:
- Mapping conflict (wrong data type)
- Cluster at capacity
- Version conflict
- Network timeout
```

### Implementation with Retry and DLQ

```java
@Service
@Slf4j
public class ElasticsearchSyncConsumer {

    @KafkaListener(topics = "job-es-sync")
    @RetryableTopic(
        attempts = "3",
        backoff = @Backoff(delay = 1000, multiplier = 2),
        dltStrategy = DltStrategy.FAIL_ON_ERROR
    )
    public void handleSync(JobSyncMessage message) {
        Job job = jobRepository.findById(message.getJobId())
            .orElseThrow(() -> new JobNotFoundException(message.getJobId()));

        JobDocument doc = mapper.toDocument(job);

        try {
            esOperations.save(doc);
            jobRepository.updateEsSyncedAt(job.getId(), Instant.now());
        } catch (ElasticsearchException e) {
            log.error("Failed to index job {} to ES: {}",
                job.getId(), e.getMessage());
            throw e;  // Retry or DLQ
        }
    }

    @DltHandler
    public void handleDlt(JobSyncMessage message) {
        log.error("Job {} failed all sync retries, moved to DLQ", message.getJobId());
        alertService.sendAlert(
            "ES sync failed permanently for job: " + message.getJobId());
    }
}
```

---

## Scenario 5: Workspace Creation Fails Mid-Way

Based on your architecture, if workspace creation fails at step 2 (setup):

```
┌─────────────────────────────────────────────────────────────┐
│         SCENARIO: WORKSPACE CREATE FAILS AT STEP 2          │
└─────────────────────────────────────────────────────────────┘

Steps:
1. Container create on host     ✓
2. Setup                        ✗ FAILS
3. IP assign                    (not reached)
4. Hostname entry               (not reached)

State:
- PostgreSQL: workspace in CREATING state, job in IN_PROGRESS
- Docker: Orphan container exists
- ES: Job shows IN_PROGRESS
```

### Recovery Process

```java
@Service
public class WorkspaceCreationService {

    @Transactional
    public void handleCreationFailure(UUID jobId, Exception error) {
        Job job = jobRepository.findById(jobId).orElseThrow();
        Workspace workspace = workspaceRepository.findById(job.getWorkspaceId())
            .orElseThrow();

        // 1. Clean up orphan container
        try {
            dockerService.removeContainer(workspace.getContainerId());
        } catch (Exception e) {
            log.warn("Failed to cleanup container, will be caught by cleanup daemon", e);
        }

        // 2. Update job status
        job.setStatus(JobStatus.FAILED);
        job.setErrorMessage(error.getMessage());
        job.setCompletedAt(Instant.now());
        jobRepository.save(job);

        // 3. Update workspace status
        workspace.setStatus(WorkspaceStatus.FAILED);
        workspaceRepository.save(workspace);

        // 4. Events trigger ES sync automatically

        // 5. Notify user
        notificationService.notifyCreationFailed(workspace, error);
    }
}
```

### Cleanup Daemon (From Your Architecture)

```java
@Component
@Scheduled(fixedDelay = 60_000)  // Every minute
public class CleanupDaemon {

    public void cleanupOrphanContainers() {
        // Get all containers on this host
        List<Container> containers = dockerClient.listContainers();

        for (Container container : containers) {
            String workspaceId = container.getLabels().get("workspace-id");

            if (workspaceId == null) continue;

            // Check if workspace exists in DB
            Optional<Workspace> ws = workspaceRepository.findById(workspaceId);

            if (ws.isEmpty() || ws.get().getStatus() == WorkspaceStatus.DELETED) {
                log.info("Removing orphan container for workspace {}", workspaceId);
                dockerClient.removeContainer(container.getId());
            }
        }
    }
}
```

---

## Scenario 6: Data Inconsistency Between PostgreSQL and ES

```
┌─────────────────────────────────────────────────────────────┐
│           SCENARIO: ES HAS STALE DATA                       │
└─────────────────────────────────────────────────────────────┘

Cause: Consumer lag, failed syncs, or bugs

Detection:
- Monitoring: Compare PostgreSQL count vs ES count
- User reports: "I see old data"
- Alerting: Sync lag > 1 minute
```

### Reconciliation Job

```java
@Component
public class DataReconciliationJob {

    @Scheduled(cron = "0 0 3 * * *")  // Daily at 3 AM
    public void reconcile() {
        log.info("Starting ES reconciliation");

        // 1. Find jobs updated in last 24h
        Instant since = Instant.now().minus(Duration.ofDays(1));
        List<Job> recentJobs = jobRepository.findByUpdatedAtAfter(since);

        int fixed = 0;
        for (Job job : recentJobs) {
            JobDocument esDoc = esOperations.get(
                job.getId().toString(), JobDocument.class);

            if (esDoc == null || esDoc.getVersion() < job.getVersion()) {
                // ES is stale, re-index
                esOperations.save(mapper.toDocument(job));
                fixed++;
            }
        }

        log.info("Reconciliation complete. Fixed {} documents", fixed);
        metrics.gauge("reconciliation.fixed", fixed);
    }
}
```

---

## Alerting Summary

| Alert | Condition | Severity |
|-------|-----------|----------|
| ES cluster unhealthy | Cluster status != green | Critical |
| Sync lag high | Lag > 5 minutes | Warning |
| Fallback activated | Fallback count > 10/min | Warning |
| DLQ growing | DLQ size > 100 | Critical |
| Consumer lag | Kafka lag > 10000 | Warning |

---

## Interview Talking Points

1. **"What happens if ES goes down?"**
   > "We have a circuit breaker that detects ES failures. When open, we fall back to querying PostgreSQL replicas. It's slower but the dashboard remains functional. We show a banner to users and alert the on-call team."

2. **"How do you ensure data eventually syncs?"**
   > "Three layers: First, the event-driven sync for immediate updates. Second, Kafka retries with exponential backoff. Third, a backup poller that catches anything that fell through. We also run a daily reconciliation job."

3. **"What if a message is processed twice?"**
   > "Our consumer is idempotent. We compare the version number in the incoming message with what's in ES. If ES already has the same or newer version, we skip the update. This handles both retries and out-of-order processing."
