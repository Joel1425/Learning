# Indexing Strategy

## Overview

Three approaches to sync PostgreSQL → Elasticsearch:

| Approach | Latency | Complexity | Our Choice |
|----------|---------|------------|------------|
| Dual Write | ~0ms | High (consistency issues) | No |
| CDC (Debezium) | ~1-2s | Medium | Considered |
| Polling + Events | ~5s | Low | **Yes** |

---

## Our Approach: Polling + Application Events

```
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  ┌─────────────┐     ┌─────────────┐     ┌──────────────┐   │
│  │   Service   │────▶│ PostgreSQL  │     │Elasticsearch │   │
│  │  (A4C etc)  │     │   (job)     │     │   (jobs)     │   │
│  └─────────────┘     └──────┬──────┘     └──────▲───────┘   │
│         │                   │                   │           │
│         │                   │                   │           │
│         ▼                   ▼                   │           │
│  ┌─────────────┐     ┌─────────────┐            │           │
│  │   Kafka     │────▶│  ES Sync    │────────────┘           │
│  │  (events)   │     │  Consumer   │                        │
│  └─────────────┘     └─────────────┘                        │
│                             │                               │
│                             ▼                               │
│                      ┌─────────────┐                        │
│                      │   Poller    │ (catches missed)       │
│                      │   (backup)  │                        │
│                      └─────────────┘                        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Step 1: Application Publishes Events

```java
@Service
@Transactional
public class JobService {

    private final JobRepository jobRepository;
    private final ApplicationEventPublisher eventPublisher;

    public Job createJob(CreateJobRequest request) {
        Job job = new Job();
        job.setStatus(JobStatus.SCHEDULED);
        job.setType(request.getType());
        job.setProjectName(request.getProjectName());
        // ... set other fields

        Job saved = jobRepository.save(job);

        // Publish event after successful commit
        eventPublisher.publishEvent(new JobCreatedEvent(saved.getId()));

        return saved;
    }

    public Job updateJobStatus(UUID jobId, JobStatus newStatus) {
        Job job = jobRepository.findById(jobId)
            .orElseThrow(() -> new JobNotFoundException(jobId));

        job.setStatus(newStatus);
        if (newStatus.isTerminal()) {
            job.setCompletedAt(Instant.now());
        }

        Job saved = jobRepository.save(job);

        eventPublisher.publishEvent(new JobUpdatedEvent(saved.getId()));

        return saved;
    }
}
```

---

## Step 2: Event Listener Sends to Kafka

```java
@Component
public class JobEventListener {

    private final KafkaTemplate<String, JobSyncMessage> kafkaTemplate;

    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void onJobCreated(JobCreatedEvent event) {
        sendSyncMessage(event.getJobId(), SyncAction.INDEX);
    }

    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void onJobUpdated(JobUpdatedEvent event) {
        sendSyncMessage(event.getJobId(), SyncAction.INDEX);
    }

    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void onJobDeleted(JobDeletedEvent event) {
        sendSyncMessage(event.getJobId(), SyncAction.DELETE);
    }

    private void sendSyncMessage(UUID jobId, SyncAction action) {
        JobSyncMessage message = new JobSyncMessage(jobId, action, Instant.now());
        kafkaTemplate.send("job-es-sync", jobId.toString(), message);
    }
}
```

**Key:** `AFTER_COMMIT` ensures message sent only after DB transaction succeeds.

---

## Step 3: Consumer Indexes to Elasticsearch

```java
@Service
public class ElasticsearchSyncConsumer {

    private final JobRepository jobRepository;
    private final ElasticsearchOperations esOperations;
    private final JobDocumentMapper mapper;

    @KafkaListener(topics = "job-es-sync", groupId = "es-sync")
    public void handleSyncMessage(JobSyncMessage message) {
        try {
            if (message.getAction() == SyncAction.DELETE) {
                esOperations.delete(message.getJobId().toString(), JobDocument.class);
            } else {
                // Fetch latest from DB (source of truth)
                Job job = jobRepository.findById(message.getJobId())
                    .orElse(null);

                if (job == null) {
                    // Job was deleted between event and processing
                    esOperations.delete(message.getJobId().toString(), JobDocument.class);
                    return;
                }

                JobDocument doc = mapper.toDocument(job);
                esOperations.save(doc);

                // Mark as synced in PostgreSQL
                jobRepository.updateEsSyncedAt(job.getId(), Instant.now(), job.getVersion());
            }
        } catch (Exception e) {
            log.error("Failed to sync job {} to ES", message.getJobId(), e);
            throw e; // Kafka will retry
        }
    }
}
```

---

## Step 4: Backup Poller (Catches Missed Events)

```java
@Component
public class ElasticsearchSyncPoller {

    private final JobRepository jobRepository;
    private final KafkaTemplate<String, JobSyncMessage> kafkaTemplate;

    @Scheduled(fixedDelay = 30000) // Every 30 seconds
    public void pollUnsyncedJobs() {
        List<Job> unsynced = jobRepository.findUnsyncedJobs(
            Instant.now().minus(Duration.ofMinutes(5)),  // Older than 5 min
            PageRequest.of(0, 100)
        );

        for (Job job : unsynced) {
            JobSyncMessage message = new JobSyncMessage(
                job.getId(),
                SyncAction.INDEX,
                Instant.now()
            );
            kafkaTemplate.send("job-es-sync", job.getId().toString(), message);
        }

        if (!unsynced.isEmpty()) {
            log.info("Poller queued {} unsynced jobs for ES sync", unsynced.size());
        }
    }
}
```

Repository query:
```java
@Query("SELECT j FROM Job j WHERE j.updatedAt < :threshold " +
       "AND (j.esSyncedAt IS NULL OR j.esSyncedAt < j.updatedAt) " +
       "ORDER BY j.updatedAt ASC")
List<Job> findUnsyncedJobs(@Param("threshold") Instant threshold, Pageable pageable);
```

---

## Bulk Indexing (Initial Load / Reindex)

```java
@Service
public class BulkReindexService {

    private final JobRepository jobRepository;
    private final ElasticsearchOperations esOperations;
    private final JobDocumentMapper mapper;

    public void reindexAll() {
        String newIndex = "jobs-v" + System.currentTimeMillis();

        // Create new index with mapping
        esOperations.indexOps(IndexCoordinates.of(newIndex)).create();
        esOperations.indexOps(IndexCoordinates.of(newIndex)).putMapping(JobDocument.class);

        // Bulk index in batches
        int pageSize = 1000;
        int page = 0;

        while (true) {
            Page<Job> jobs = jobRepository.findAll(PageRequest.of(page, pageSize));

            if (jobs.isEmpty()) break;

            List<IndexQuery> queries = jobs.stream()
                .map(job -> new IndexQueryBuilder()
                    .withId(job.getId().toString())
                    .withObject(mapper.toDocument(job))
                    .build())
                .collect(Collectors.toList());

            esOperations.bulkIndex(queries, IndexCoordinates.of(newIndex));

            page++;
            log.info("Reindexed page {} ({} total)", page, page * pageSize);
        }

        // Atomic alias switch
        esOperations.indexOps(IndexCoordinates.of("jobs")).updateAliases(
            new AliasActions()
                .add(new AliasAction.Add("jobs", newIndex))
                .remove(new AliasAction.Remove("jobs", "jobs-v*"))
        );

        log.info("Reindex complete. Switched alias to {}", newIndex);
    }
}
```

---

## Why Not Dual Write?

```java
// BAD: Dual write pattern
@Transactional
public Job createJob(CreateJobRequest request) {
    Job job = jobRepository.save(new Job(...));
    esOperations.save(mapper.toDocument(job));  // What if this fails?
    return job;
}
```

**Problems:**
1. ES write fails → DB has data, ES doesn't
2. Transaction rollback → DB rolled back, ES already has data
3. Network timeout → Unknown state

**Our approach guarantees:** DB is always source of truth, ES is eventually consistent.

---

## Why Not CDC (Debezium)?

CDC is excellent but adds operational complexity:
- Need Kafka Connect cluster
- Debezium connector configuration
- Schema registry for Avro/JSON
- More infrastructure to monitor

For our scale (~100K jobs/day), polling + events is simpler and sufficient.

---

## Interview Summary

> "We use an event-driven approach with a backup poller. When a job is created or updated, we publish an event after the transaction commits. A Kafka consumer picks up the event, fetches the latest state from PostgreSQL, and indexes to ES. A backup poller runs every 30 seconds to catch any missed events. This gives us eventual consistency with ~5 second typical latency, which is acceptable for a dashboard use case."
