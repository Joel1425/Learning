# Complete Control Flow: Write Path, Read Path & Elasticsearch Internals

## Overview

This document explains:
1. **Write Path**: Service → Database → Kafka → Elasticsearch
2. **Read Path**: Client → Search API → Elasticsearch → Response
3. **Elasticsearch Internals**: How ES stores and retrieves data

---

# PART 1: WRITE PATH

## Step-by-Step: How Data Gets to Elasticsearch

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                              WRITE PATH OVERVIEW                             │
│                                                                              │
│                                                                              │
│   ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐              │
│   │  Step 1  │───▶│  Step 2  │───▶│  Step 3  │───▶│  Step 4  │              │
│   │          │    │          │    │          │    │          │              │
│   │ Service  │    │  Write   │    │ Publish  │    │ Consumer │              │
│   │ Request  │    │  to DB   │    │ to Kafka │    │ Receives │              │
│   └──────────┘    └──────────┘    └──────────┘    └──────────┘              │
│                                                          │                   │
│                                                          ▼                   │
│                                        ┌──────────┐    ┌──────────┐         │
│                                        │  Step 6  │◀───│  Step 5  │         │
│                                        │          │    │          │         │
│                                        │ Index to │    │  Enrich  │         │
│                                        │    ES    │    │   Data   │         │
│                                        └──────────┘    └──────────┘         │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## Step 1: Service Receives Request

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         STEP 1: SERVICE REQUEST                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   Example: Test Service receives a new test run result                      │
│                                                                             │
│   Client (CI System)                                                        │
│        │                                                                    │
│        │  POST /api/v1/test-runs                                           │
│        │  {                                                                 │
│        │    "testName": "auth_login_test",                                 │
│        │    "testSuite": "auth-integration",                               │
│        │    "platform": "linux-x64",                                       │
│        │    "status": "FAILED",                                            │
│        │    "errorMessage": "Connection timeout",                          │
│        │    "projectId": "proj-123",                                       │
│        │    "ownerId": "user-456"                                          │
│        │  }                                                                 │
│        ▼                                                                    │
│   ┌─────────────────────────────────────────┐                              │
│   │           Test Service                   │                              │
│   │                                          │                              │
│   │   @PostMapping("/test-runs")             │                              │
│   │   public TestRun create(                 │                              │
│   │       @RequestBody CreateTestRunRequest  │                              │
│   │   ) {                                    │                              │
│   │       return testRunService.create(req); │                              │
│   │   }                                      │                              │
│   └─────────────────────────────────────────┘                              │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Step 2: Write to Service's Own Database

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      STEP 2: WRITE TO DATABASE                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │                      TestRunService                                  │  │
│   │                                                                      │  │
│   │   @Service                                                           │  │
│   │   public class TestRunService {                                      │  │
│   │                                                                      │  │
│   │       @Transactional                                                 │  │
│   │       public TestRun create(CreateTestRunRequest request) {          │  │
│   │                                                                      │  │
│   │           // 1. Create entity                                        │  │
│   │           TestRun testRun = new TestRun();                           │  │
│   │           testRun.setTestName(request.getTestName());                │  │
│   │           testRun.setStatus(request.getStatus());                    │  │
│   │           testRun.setPlatform(request.getPlatform());                │  │
│   │           testRun.setCreatedAt(Instant.now());                       │  │
│   │                                                                      │  │
│   │           // 2. Save to PostgreSQL                                   │  │
│   │           TestRun saved = testRunRepository.save(testRun);           │  │
│   │                                                                      │  │
│   │           // 3. Publish event (AFTER transaction commits)            │  │
│   │           eventPublisher.publishEvent(                               │  │
│   │               new TestRunCreatedEvent(saved.getId())                 │  │
│   │           );                                                         │  │
│   │                                                                      │  │
│   │           return saved;                                              │  │
│   │       }                                                              │  │
│   │   }                                                                  │  │
│   └──────────────────────────────┬──────────────────────────────────────┘  │
│                                  │                                          │
│                                  │ SQL: INSERT INTO test_runs ...           │
│                                  ▼                                          │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │                    PostgreSQL (Test Service DB)                      │  │
│   │                                                                      │  │
│   │   test_runs table:                                                   │  │
│   │   ┌────────────────────────────────────────────────────────────┐    │  │
│   │   │ id       │ test_name       │ status │ platform  │ created  │    │  │
│   │   ├──────────┼─────────────────┼────────┼───────────┼──────────┤    │  │
│   │   │ tr-12345 │ auth_login_test │ FAILED │ linux-x64 │ 2024-01..│    │  │
│   │   └────────────────────────────────────────────────────────────┘    │  │
│   │                                                                      │  │
│   │   COMMIT; ✓                                                          │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│   KEY POINT: Data is safely stored in PostgreSQL (source of truth)         │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Step 3: Publish Event to Kafka

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      STEP 3: PUBLISH TO KAFKA                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   CRITICAL: Event is published AFTER transaction commits                    │
│                                                                             │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │                    Event Listener                                    │  │
│   │                                                                      │  │
│   │   @Component                                                         │  │
│   │   public class TestRunEventListener {                                │  │
│   │                                                                      │  │
│   │       // AFTER_COMMIT ensures DB transaction succeeded               │  │
│   │       @TransactionalEventListener(                                   │  │
│   │           phase = TransactionPhase.AFTER_COMMIT                      │  │
│   │       )                                                              │  │
│   │       public void onTestRunCreated(TestRunCreatedEvent event) {      │  │
│   │                                                                      │  │
│   │           SearchSyncMessage message = new SearchSyncMessage(         │  │
│   │               event.getTestRunId(),                                  │  │
│   │               "TEST_RUN",                                            │  │
│   │               SyncAction.INDEX,                                      │  │
│   │               Instant.now()                                          │  │
│   │           );                                                         │  │
│   │                                                                      │  │
│   │           kafkaTemplate.send(                                        │  │
│   │               "search-sync",                    // topic             │  │
│   │               event.getTestRunId().toString(),  // key (for ordering)│  │
│   │               message                           // payload           │  │
│   │           );                                                         │  │
│   │       }                                                              │  │
│   │   }                                                                  │  │
│   └──────────────────────────────┬──────────────────────────────────────┘  │
│                                  │                                          │
│                                  ▼                                          │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │                           KAFKA                                      │  │
│   │                                                                      │  │
│   │   Topic: search-sync                                                │  │
│   │   ┌─────────────────────────────────────────────────────────────┐   │  │
│   │   │ Partition 0: [msg1] [msg2] [msg3] ...                       │   │  │
│   │   │ Partition 1: [msg4] [msg5] ...                              │   │  │
│   │   │ Partition 2: [msg6] [msg7] [NEW MSG: tr-12345] ...          │   │  │
│   │   └─────────────────────────────────────────────────────────────┘   │  │
│   │                                                                      │  │
│   │   Message:                                                           │  │
│   │   {                                                                  │  │
│   │     "entityId": "tr-12345",                                         │  │
│   │     "entityType": "TEST_RUN",                                       │  │
│   │     "action": "INDEX",                                              │  │
│   │     "timestamp": "2024-01-15T10:30:00Z",                            │  │
│   │     "sourceService": "test-service"                                 │  │
│   │   }                                                                  │  │
│   │                                                                      │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│   WHY KAFKA?                                                                │
│   • Decouples services from ES (ES down doesn't break writes)              │
│   • Provides durability (messages persist until consumed)                  │
│   • Enables retry on failures                                              │
│   • Maintains ordering per entity (using key-based partitioning)           │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Step 4 & 5: Consumer Receives and Enriches Data

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                   STEP 4 & 5: CONSUME AND ENRICH                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │                    ES Sync Consumer Service                          │  │
│   │                                                                      │  │
│   │   @Service                                                           │  │
│   │   public class SearchSyncConsumer {                                  │  │
│   │                                                                      │  │
│   │       @KafkaListener(topics = "search-sync", groupId = "es-sync")   │  │
│   │       public void handleMessage(SearchSyncMessage message) {         │  │
│   │                                                                      │  │
│   │           // STEP 4: Receive message                                 │  │
│   │           log.info("Received sync message for {}", message);         │  │
│   │                                                                      │  │
│   │           // STEP 5: Fetch and enrich from multiple services         │  │
│   │           SearchDocument doc = buildDocument(message);               │  │
│   │                                                                      │  │
│   │           // STEP 6: Index to Elasticsearch                          │  │
│   │           elasticsearchClient.index(doc);                            │  │
│   │       }                                                              │  │
│   │                                                                      │  │
│   │       private SearchDocument buildDocument(SearchSyncMessage msg) {  │  │
│   │                                                                      │  │
│   │           // Fetch from TEST SERVICE (source of event)               │  │
│   │           TestRun testRun = testServiceClient                        │  │
│   │               .getTestRun(msg.getEntityId());                        │  │
│   │                                                                      │  │
│   │           // Fetch from PROJECT SERVICE (enrich with project info)   │  │
│   │           Project project = projectServiceClient                     │  │
│   │               .getProject(testRun.getProjectId());                   │  │
│   │                                                                      │  │
│   │           // Fetch from USER SERVICE (enrich with owner info)        │  │
│   │           User owner = userServiceClient                             │  │
│   │               .getUser(testRun.getOwnerId());                        │  │
│   │                                                                      │  │
│   │           // Fetch from BUG SERVICE (check for linked bugs)          │  │
│   │           List<Bug> bugs = bugServiceClient                          │  │
│   │               .getBugsByTestRun(testRun.getId());                    │  │
│   │                                                                      │  │
│   │           // BUILD DENORMALIZED DOCUMENT                             │  │
│   │           return SearchDocument.builder()                            │  │
│   │               .id(testRun.getId())                                   │  │
│   │               // From Test Service                                   │  │
│   │               .testName(testRun.getTestName())                       │  │
│   │               .testSuite(testRun.getTestSuite())                     │  │
│   │               .platform(testRun.getPlatform())                       │  │
│   │               .testStatus(testRun.getStatus())                       │  │
│   │               // From Project Service                                │  │
│   │               .projectName(project.getName())                        │  │
│   │               .changelist(project.getChangelist())                   │  │
│   │               .primary(project.getPrimary())                         │  │
│   │               // From User Service                                   │  │
│   │               .owner(owner.getEmail())                               │  │
│   │               .ownerTeam(owner.getTeam())                            │  │
│   │               .personality(owner.getPersonality())                   │  │
│   │               // From Bug Service                                    │  │
│   │               .hasBugHit(!bugs.isEmpty())                            │  │
│   │               .bugId(bugs.isEmpty() ? null : bugs.get(0).getId())    │  │
│   │               .bugStatus(bugs.isEmpty() ? null : bugs.get(0).getStatus())│
│   │               // Timestamps                                          │  │
│   │               .createdAt(testRun.getCreatedAt())                     │  │
│   │               .build();                                              │  │
│   │       }                                                              │  │
│   │   }                                                                  │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│                                                                             │
│   ENRICHMENT FLOW:                                                          │
│                                                                             │
│        Kafka Message                                                        │
│        (testRunId: tr-12345)                                               │
│              │                                                              │
│              ▼                                                              │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │                    ES Sync Consumer                                  │  │
│   │                          │                                           │  │
│   │    ┌─────────────────────┼─────────────────────┐                    │  │
│   │    │                     │                     │                    │  │
│   │    ▼                     ▼                     ▼                    │  │
│   │ ┌──────────┐       ┌──────────┐         ┌──────────┐               │  │
│   │ │  Test    │       │ Project  │         │   User   │               │  │
│   │ │ Service  │       │ Service  │         │ Service  │               │  │
│   │ │   API    │       │   API    │         │   API    │               │  │
│   │ └────┬─────┘       └────┬─────┘         └────┬─────┘               │  │
│   │      │                  │                    │                      │  │
│   │      │    ┌─────────────┼────────────────────┘                     │  │
│   │      │    │             │                                           │  │
│   │      ▼    ▼             ▼                                           │  │
│   │    ┌────────────────────────────────────────────┐                  │  │
│   │    │          DENORMALIZED DOCUMENT              │                  │  │
│   │    │                                             │                  │  │
│   │    │  testName + testStatus + platform          │  ← Test Service  │  │
│   │    │  projectName + changelist + primary        │  ← Project Svc   │  │
│   │    │  owner + ownerTeam + personality           │  ← User Service  │  │
│   │    │  bugId + bugStatus + hasBugHit             │  ← Bug Service   │  │
│   │    │                                             │                  │  │
│   │    └────────────────────────────────────────────┘                  │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Step 6: Index to Elasticsearch

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     STEP 6: INDEX TO ELASTICSEARCH                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   Consumer sends document to Elasticsearch:                                 │
│                                                                             │
│   PUT /test-results/_doc/tr-12345                                          │
│   {                                                                         │
│     "id": "tr-12345",                                                       │
│     "testName": "auth_login_test",                                         │
│     "testSuite": "auth-integration",                                       │
│     "platform": "linux-x64",                                               │
│     "testStatus": "FAILED",                                                │
│     "errorMessage": "Connection timeout",                                  │
│     "projectName": "backend-api",                                          │
│     "changelist": "CL-98765",                                              │
│     "primary": "main",                                                     │
│     "owner": "john@company.com",                                           │
│     "ownerTeam": "platform-team",                                          │
│     "personality": "developer",                                            │
│     "hasBugHit": true,                                                     │
│     "bugId": "BUG-6789",                                                   │
│     "bugStatus": "OPEN",                                                   │
│     "createdAt": "2024-01-15T10:30:00Z"                                    │
│   }                                                                         │
│                                                                             │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │                      ELASTICSEARCH CLUSTER                           │  │
│   │                                                                      │  │
│   │   1. Document arrives at Coordinating Node                           │  │
│   │                          │                                           │  │
│   │   2. Route to correct shard (hash of doc ID)                        │  │
│   │      shard = hash("tr-12345") % 3 = Shard 1                         │  │
│   │                          │                                           │  │
│   │                          ▼                                           │  │
│   │   ┌─────────┐      ┌─────────┐      ┌─────────┐                     │  │
│   │   │ Shard 0 │      │ Shard 1 │◀──── │ Shard 2 │                     │  │
│   │   │         │      │         │      │         │                     │  │
│   │   │         │      │ ┌─────┐ │      │         │                     │  │
│   │   │         │      │ │ Doc │ │      │         │                     │  │
│   │   │         │      │ │Added│ │      │         │                     │  │
│   │   │         │      │ └─────┘ │      │         │                     │  │
│   │   └─────────┘      └────┬────┘      └─────────┘                     │  │
│   │                         │                                            │  │
│   │   3. Primary shard writes to transaction log                        │  │
│   │                         │                                            │  │
│   │   4. Replicate to replica shard                                     │  │
│   │                         │                                            │  │
│   │                         ▼                                            │  │
│   │                   ┌─────────┐                                        │  │
│   │                   │ Shard 1 │                                        │  │
│   │                   │ Replica │                                        │  │
│   │                   └─────────┘                                        │  │
│   │                                                                      │  │
│   │   5. Return success to consumer                                      │  │
│   │                                                                      │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│   NOTE: Document is NOT immediately searchable!                             │
│   It becomes searchable after "refresh" (default: every 1 second)          │
│   We configured refresh_interval: 5s for better performance                │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Complete Write Path Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      COMPLETE WRITE PATH                                    │
└─────────────────────────────────────────────────────────────────────────────┘

    CI System / Client
          │
          │ POST /api/v1/test-runs
          ▼
    ┌───────────────┐
    │ Test Service  │
    │               │
    │  1. Validate  │
    │  2. Save to   │─────────────────────┐
    │     database  │                     │
    │  3. Publish   │──────────┐          │
    │     event     │          │          │
    └───────────────┘          │          │
          │                    │          │
          │ 201 Created        │          │
          ▼                    │          │
    CI System / Client         │          │
                               │          │
                               ▼          ▼
                          ┌────────┐  ┌────────────┐
                          │ Kafka  │  │ PostgreSQL │
                          │        │  │ (Test DB)  │
                          │ Topic: │  │            │
                          │ search │  │ test_runs  │
                          │ -sync  │  │ table      │
                          └───┬────┘  └────────────┘
                              │
                              │ Consumer polls
                              ▼
                    ┌──────────────────────┐
                    │   ES Sync Consumer   │
                    │                      │
                    │ 4. Receive message   │
                    │                      │
                    │ 5. Enrich:           │
                    │    ├─▶ Test Service  │───▶ Get test details
                    │    ├─▶ Project Svc   │───▶ Get project info
                    │    ├─▶ User Service  │───▶ Get owner info
                    │    └─▶ Bug Service   │───▶ Get linked bugs
                    │                      │
                    │ 6. Build document    │
                    └──────────┬───────────┘
                               │
                               │ PUT /_doc/tr-12345
                               ▼
                    ┌──────────────────────┐
                    │    Elasticsearch     │
                    │                      │
                    │  ┌────────────────┐  │
                    │  │  Denormalized  │  │
                    │  │   Document     │  │
                    │  │                │  │
                    │  │ All service    │  │
                    │  │ data combined  │  │
                    │  └────────────────┘  │
                    │                      │
                    └──────────────────────┘

    Timeline:
    ─────────────────────────────────────────────────────────────▶
    T0          T1              T2              T3           T4
    Request     DB Write        Kafka           Consumer     Searchable
    received    committed       published       indexed      in ES
                                               to ES

    │◀──────────────────────────────────────────────────────────▶│
                        Typically 2-5 seconds
```

---

# PART 2: READ PATH

## Step-by-Step: How Data is Fetched

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              READ PATH OVERVIEW                             │
│                                                                             │
│                                                                             │
│   ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐             │
│   │  Step 1  │───▶│  Step 2  │───▶│  Step 3  │───▶│  Step 4  │             │
│   │          │    │          │    │          │    │          │             │
│   │ Client   │    │  Search  │    │  Query   │    │ Execute  │             │
│   │ Request  │    │   API    │    │  Build   │    │ on ES    │             │
│   └──────────┘    └──────────┘    └──────────┘    └──────────┘             │
│                                                          │                  │
│                                                          ▼                  │
│                         ┌──────────┐    ┌──────────┐    ┌──────────┐       │
│                         │  Step 7  │◀───│  Step 6  │◀───│  Step 5  │       │
│                         │          │    │          │    │          │       │
│                         │ Return   │    │   Map    │    │  Merge   │       │
│                         │ Response │    │ to DTOs  │    │ Results  │       │
│                         └──────────┘    └──────────┘    └──────────┘       │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Read Path: Detailed Steps

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         READ PATH: STEP BY STEP                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   STEP 1: Client Request                                                    │
│   ─────────────────────                                                     │
│                                                                             │
│   Dashboard Frontend                                                        │
│        │                                                                    │
│        │  POST /api/v1/test-results/search                                 │
│        │  {                                                                 │
│        │    "filters": {                                                    │
│        │      "testName": "auth_login_test",                               │
│        │      "hasBugHit": true,                                           │
│        │      "primary": "main",                                           │
│        │      "owner": "john@company.com",                                 │
│        │      "createdAt": { "gte": "now-30d" }                            │
│        │    },                                                              │
│        │    "sort": { "field": "createdAt", "order": "desc" },             │
│        │    "size": 20                                                      │
│        │  }                                                                 │
│        ▼                                                                    │
│                                                                             │
│   STEP 2: Search API Controller                                             │
│   ─────────────────────────────                                             │
│                                                                             │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │   @RestController                                                    │  │
│   │   @RequestMapping("/api/v1/test-results")                           │  │
│   │   public class SearchController {                                    │  │
│   │                                                                      │  │
│   │       @PostMapping("/search")                                        │  │
│   │       public SearchResponse search(                                  │  │
│   │           @RequestBody SearchRequest request,                        │  │
│   │           @AuthenticationPrincipal User user                         │  │
│   │       ) {                                                            │  │
│   │           // Add user context for security filtering                 │  │
│   │           request.setRequestingUser(user);                           │  │
│   │           return searchService.search(request);                      │  │
│   │       }                                                              │  │
│   │   }                                                                  │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│                                  │                                          │
│                                  ▼                                          │
│                                                                             │
│   STEP 3: Build Elasticsearch Query                                         │
│   ─────────────────────────────────                                         │
│                                                                             │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │   @Service                                                           │  │
│   │   public class SearchService {                                       │  │
│   │                                                                      │  │
│   │       public SearchResponse search(SearchRequest request) {          │  │
│   │                                                                      │  │
│   │           // Build Elasticsearch query                               │  │
│   │           BoolQueryBuilder query = QueryBuilders.boolQuery();        │  │
│   │                                                                      │  │
│   │           // Add filters (all in filter context for caching)         │  │
│   │           query.filter(termQuery("testName", "auth_login_test"));    │  │
│   │           query.filter(termQuery("hasBugHit", true));                │  │
│   │           query.filter(termQuery("primary", "main"));                │  │
│   │           query.filter(termQuery("owner", "john@company.com"));      │  │
│   │           query.filter(rangeQuery("createdAt").gte("now-30d"));      │  │
│   │                                                                      │  │
│   │           // Execute search                                          │  │
│   │           SearchHits<SearchDocument> hits =                          │  │
│   │               elasticsearchOps.search(query, SearchDocument.class);  │  │
│   │                                                                      │  │
│   │           return mapToResponse(hits);                                │  │
│   │       }                                                              │  │
│   │   }                                                                  │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│                                  │                                          │
│                                  ▼                                          │
│                                                                             │
│   The query sent to Elasticsearch:                                          │
│                                                                             │
│   {                                                                         │
│     "query": {                                                              │
│       "bool": {                                                             │
│         "filter": [                                                         │
│           { "term": { "testName": "auth_login_test" } },                   │
│           { "term": { "hasBugHit": true } },                               │
│           { "term": { "primary": "main" } },                               │
│           { "term": { "owner": "john@company.com" } },                     │
│           { "range": { "createdAt": { "gte": "now-30d" } } }               │
│         ]                                                                   │
│       }                                                                     │
│     },                                                                      │
│     "sort": [{ "createdAt": { "order": "desc" } }],                        │
│     "size": 20                                                              │
│   }                                                                         │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Read Path: Inside Elasticsearch

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    STEP 4 & 5: ELASTICSEARCH QUERY EXECUTION                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │                      ELASTICSEARCH CLUSTER                           │  │
│   │                                                                      │  │
│   │   Query arrives at COORDINATING NODE                                 │  │
│   │              │                                                       │  │
│   │              ▼                                                       │  │
│   │   ┌──────────────────────────────────────┐                          │  │
│   │   │      1. PARSE QUERY                   │                          │  │
│   │   │                                       │                          │  │
│   │   │   Parse JSON → Query AST             │                          │  │
│   │   │   Identify: bool query with filters   │                          │  │
│   │   └──────────────────────────────────────┘                          │  │
│   │              │                                                       │  │
│   │              ▼                                                       │  │
│   │   ┌──────────────────────────────────────┐                          │  │
│   │   │      2. IDENTIFY TARGET SHARDS        │                          │  │
│   │   │                                       │                          │  │
│   │   │   No routing specified → Query ALL    │                          │  │
│   │   │   shards: [Shard 0, Shard 1, Shard 2] │                          │  │
│   │   └──────────────────────────────────────┘                          │  │
│   │              │                                                       │  │
│   │              │  FAN OUT (parallel)                                   │  │
│   │    ┌─────────┼─────────┐                                            │  │
│   │    │         │         │                                            │  │
│   │    ▼         ▼         ▼                                            │  │
│   │ ┌────────┐ ┌────────┐ ┌────────┐                                   │  │
│   │ │Shard 0 │ │Shard 1 │ │Shard 2 │                                   │  │
│   │ │        │ │        │ │        │                                   │  │
│   │ │ ┌────┐ │ │ ┌────┐ │ │ ┌────┐ │                                   │  │
│   │ │ │ 3a │ │ │ │ 3a │ │ │ │ 3a │ │  3a. Check filter cache           │  │
│   │ │ └────┘ │ │ └────┘ │ │ └────┘ │                                   │  │
│   │ │   │    │ │   │    │ │   │    │                                   │  │
│   │ │   ▼    │ │   ▼    │ │   ▼    │                                   │  │
│   │ │ ┌────┐ │ │ ┌────┐ │ │ ┌────┐ │  3b. Query inverted index         │  │
│   │ │ │ 3b │ │ │ │ 3b │ │ │ │ 3b │ │      (see next section)           │  │
│   │ │ └────┘ │ │ └────┘ │ │ └────┘ │                                   │  │
│   │ │   │    │ │   │    │ │   │    │                                   │  │
│   │ │   ▼    │ │   ▼    │ │   ▼    │                                   │  │
│   │ │ ┌────┐ │ │ ┌────┐ │ │ ┌────┐ │  3c. Local sort, return top 20    │  │
│   │ │ │ 3c │ │ │ │ 3c │ │ │ │ 3c │ │                                   │  │
│   │ │ └────┘ │ │ └────┘ │ │ └────┘ │                                   │  │
│   │ │   │    │ │   │    │ │   │    │                                   │  │
│   │ └───┼────┘ └───┼────┘ └───┼────┘                                   │  │
│   │     │          │          │                                         │  │
│   │     └──────────┼──────────┘                                         │  │
│   │                │                                                    │  │
│   │                ▼                                                    │  │
│   │   ┌──────────────────────────────────────┐                          │  │
│   │   │      4. MERGE RESULTS                 │                          │  │
│   │   │                                       │                          │  │
│   │   │   Shard 0: [doc5, doc12, doc3, ...]   │                          │  │
│   │   │   Shard 1: [doc8, doc1, doc15, ...]   │                          │  │
│   │   │   Shard 2: [doc7, doc22, doc9, ...]   │                          │  │
│   │   │                                       │                          │  │
│   │   │   Merge + Global Sort by createdAt    │                          │  │
│   │   │   Take top 20                         │                          │  │
│   │   │                                       │                          │  │
│   │   │   Result: [doc8, doc5, doc7, doc22...]│                          │  │
│   │   └──────────────────────────────────────┘                          │  │
│   │                │                                                    │  │
│   │                ▼                                                    │  │
│   │   ┌──────────────────────────────────────┐                          │  │
│   │   │      5. FETCH PHASE                   │                          │  │
│   │   │                                       │                          │  │
│   │   │   Fetch actual document _source for   │                          │  │
│   │   │   the final 20 documents              │                          │  │
│   │   └──────────────────────────────────────┘                          │  │
│   │                │                                                    │  │
│   │                ▼                                                    │  │
│   │            RETURN TO CLIENT                                         │  │
│   │                                                                      │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│   LATENCY BREAKDOWN:                                                        │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │ Parse query:              1-2 ms                                     │  │
│   │ Fan out to shards:        1-2 ms                                     │  │
│   │ Shard query (parallel):   10-30 ms  ← Most time spent here          │  │
│   │ Merge results:            2-5 ms                                     │  │
│   │ Fetch documents:          5-10 ms                                    │  │
│   │ ─────────────────────────────────                                   │  │
│   │ TOTAL:                    20-50 ms                                   │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

# PART 3: HOW ELASTICSEARCH STORES DATA

## Inverted Index Explained

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         INVERTED INDEX STRUCTURE                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   TRADITIONAL DATABASE (Row-oriented):                                      │
│   ─────────────────────────────────────                                     │
│                                                                             │
│   ┌────────────────────────────────────────────────────────────────────┐   │
│   │ id       │ testName        │ platform  │ owner         │ status   │   │
│   ├──────────┼─────────────────┼───────────┼───────────────┼──────────┤   │
│   │ doc1     │ auth_login_test │ linux-x64 │ john@co.com   │ FAILED   │   │
│   │ doc2     │ user_signup     │ darwin    │ jane@co.com   │ PASSED   │   │
│   │ doc3     │ auth_login_test │ windows   │ john@co.com   │ PASSED   │   │
│   │ doc4     │ payment_flow    │ linux-x64 │ jane@co.com   │ FAILED   │   │
│   │ doc5     │ auth_login_test │ linux-x64 │ bob@co.com    │ FAILED   │   │
│   └────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│   To find: testName = 'auth_login_test' AND status = 'FAILED'              │
│   Must scan ALL rows and check each one → O(n)                             │
│                                                                             │
│   ═══════════════════════════════════════════════════════════════════════  │
│                                                                             │
│   ELASTICSEARCH (Inverted Index / Column-oriented):                         │
│   ───────────────────────────────────────────────                           │
│                                                                             │
│   Field: testName                                                           │
│   ┌─────────────────────────┬───────────────────────────────────────────┐  │
│   │        TERM             │           POSTING LIST (Doc IDs)          │  │
│   ├─────────────────────────┼───────────────────────────────────────────┤  │
│   │ auth_login_test         │ [doc1, doc3, doc5]                        │  │
│   │ user_signup             │ [doc2]                                    │  │
│   │ payment_flow            │ [doc4]                                    │  │
│   └─────────────────────────┴───────────────────────────────────────────┘  │
│                                                                             │
│   Field: status                                                             │
│   ┌─────────────────────────┬───────────────────────────────────────────┐  │
│   │        TERM             │           POSTING LIST (Doc IDs)          │  │
│   ├─────────────────────────┼───────────────────────────────────────────┤  │
│   │ FAILED                  │ [doc1, doc4, doc5]                        │  │
│   │ PASSED                  │ [doc2, doc3]                              │  │
│   └─────────────────────────┴───────────────────────────────────────────┘  │
│                                                                             │
│   Field: platform                                                           │
│   ┌─────────────────────────┬───────────────────────────────────────────┐  │
│   │        TERM             │           POSTING LIST (Doc IDs)          │  │
│   ├─────────────────────────┼───────────────────────────────────────────┤  │
│   │ linux-x64               │ [doc1, doc4, doc5]                        │  │
│   │ darwin                  │ [doc2]                                    │  │
│   │ windows                 │ [doc3]                                    │  │
│   └─────────────────────────┴───────────────────────────────────────────┘  │
│                                                                             │
│   Field: owner                                                              │
│   ┌─────────────────────────┬───────────────────────────────────────────┐  │
│   │        TERM             │           POSTING LIST (Doc IDs)          │  │
│   ├─────────────────────────┼───────────────────────────────────────────┤  │
│   │ john@co.com             │ [doc1, doc3]                              │  │
│   │ jane@co.com             │ [doc2, doc4]                              │  │
│   │ bob@co.com              │ [doc5]                                    │  │
│   └─────────────────────────┴───────────────────────────────────────────┘  │
│                                                                             │
│                                                                             │
│   QUERY: testName = 'auth_login_test' AND status = 'FAILED'                │
│                                                                             │
│   Step 1: Lookup testName = 'auth_login_test'                              │
│           Result: [doc1, doc3, doc5]                    ← O(1) lookup!     │
│                                                                             │
│   Step 2: Lookup status = 'FAILED'                                         │
│           Result: [doc1, doc4, doc5]                    ← O(1) lookup!     │
│                                                                             │
│   Step 3: AND operation (set intersection)                                  │
│           [doc1, doc3, doc5] ∩ [doc1, doc4, doc5] = [doc1, doc5]           │
│                                                                             │
│   FINAL RESULT: [doc1, doc5]                                               │
│                                                                             │
│   Time complexity: O(1) for lookup + O(min(n,m)) for intersection          │
│   vs O(n) for traditional scan                                             │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Where Elasticsearch Stores Data

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    ELASTICSEARCH STORAGE ARCHITECTURE                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   LOGICAL VIEW:                                                             │
│   ─────────────                                                             │
│                                                                             │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │                         INDEX: test-results                          │  │
│   │                                                                      │  │
│   │    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐            │  │
│   │    │   Shard 0   │    │   Shard 1   │    │   Shard 2   │            │  │
│   │    │  (Primary)  │    │  (Primary)  │    │  (Primary)  │            │  │
│   │    │             │    │             │    │             │            │  │
│   │    │  doc1, doc4 │    │  doc2, doc5 │    │  doc3, doc6 │            │  │
│   │    │  doc7, ...  │    │  doc8, ...  │    │  doc9, ...  │            │  │
│   │    └──────┬──────┘    └──────┬──────┘    └──────┬──────┘            │  │
│   │           │                  │                  │                    │  │
│   │           │  Replicated      │  Replicated      │  Replicated       │  │
│   │           ▼                  ▼                  ▼                    │  │
│   │    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐            │  │
│   │    │   Shard 0   │    │   Shard 1   │    │   Shard 2   │            │  │
│   │    │  (Replica)  │    │  (Replica)  │    │  (Replica)  │            │  │
│   │    └─────────────┘    └─────────────┘    └─────────────┘            │  │
│   │                                                                      │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│                                                                             │
│   PHYSICAL VIEW (On Disk):                                                  │
│   ────────────────────────                                                  │
│                                                                             │
│   Each shard is a Lucene index containing SEGMENTS:                         │
│                                                                             │
│   /var/lib/elasticsearch/data/                                              │
│   └── nodes/                                                                │
│       └── 0/                                                                │
│           └── indices/                                                      │
│               └── <index-uuid>/                                             │
│                   ├── 0/                          ← Shard 0                 │
│                   │   └── index/                                            │
│                   │       ├── _0.cfs              ← Segment 0 (compound)    │
│                   │       ├── _0.cfe              ← Segment 0 entries       │
│                   │       ├── _1.cfs              ← Segment 1               │
│                   │       ├── _1.cfe                                        │
│                   │       ├── segments_N          ← Segment metadata        │
│                   │       └── write.lock                                    │
│                   │                                                         │
│                   ├── 1/                          ← Shard 1                 │
│                   │   └── index/                                            │
│                   │       ├── ...                                           │
│                   │                                                         │
│                   └── 2/                          ← Shard 2                 │
│                       └── index/                                            │
│                           ├── ...                                           │
│                                                                             │
│                                                                             │
│   WHAT'S INSIDE A SEGMENT:                                                  │
│   ─────────────────────────                                                 │
│                                                                             │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │                         SEGMENT                                      │  │
│   │                                                                      │  │
│   │   ┌─────────────────────────────────────────────────────────────┐   │  │
│   │   │  INVERTED INDEX                                              │   │  │
│   │   │  • Term Dictionary (sorted terms)                            │   │  │
│   │   │  • Posting Lists (doc IDs per term)                          │   │  │
│   │   │  • Term Frequencies                                          │   │  │
│   │   │  • Positions (for phrase queries)                            │   │  │
│   │   └─────────────────────────────────────────────────────────────┘   │  │
│   │                                                                      │  │
│   │   ┌─────────────────────────────────────────────────────────────┐   │  │
│   │   │  STORED FIELDS (_source)                                     │   │  │
│   │   │  • Original JSON document (compressed)                       │   │  │
│   │   │  • Retrieved during fetch phase                              │   │  │
│   │   └─────────────────────────────────────────────────────────────┘   │  │
│   │                                                                      │  │
│   │   ┌─────────────────────────────────────────────────────────────┐   │  │
│   │   │  DOC VALUES                                                  │   │  │
│   │   │  • Column-oriented storage                                   │   │  │
│   │   │  • Used for sorting and aggregations                         │   │  │
│   │   └─────────────────────────────────────────────────────────────┘   │  │
│   │                                                                      │  │
│   │   ┌─────────────────────────────────────────────────────────────┐   │  │
│   │   │  NORMS                                                       │   │  │
│   │   │  • Field length normalization for scoring                    │   │  │
│   │   │  • (We disable for filter-only fields)                       │   │  │
│   │   └─────────────────────────────────────────────────────────────┘   │  │
│   │                                                                      │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│                                                                             │
│   SEGMENT LIFECYCLE:                                                        │
│   ───────────────────                                                       │
│                                                                             │
│   1. Documents indexed → Written to in-memory buffer                        │
│   2. Refresh (every 5s) → Buffer becomes searchable segment                │
│   3. Flush → Segment written to disk                                        │
│   4. Merge → Multiple small segments merged into larger ones               │
│                                                                             │
│   ┌───────┐  ┌───────┐  ┌───────┐                                         │
│   │ Seg 0 │  │ Seg 1 │  │ Seg 2 │   (many small segments)                 │
│   │ 100KB │  │ 150KB │  │ 200KB │                                         │
│   └───┬───┘  └───┬───┘  └───┬───┘                                         │
│       │          │          │                                              │
│       └──────────┼──────────┘                                              │
│                  │ MERGE                                                   │
│                  ▼                                                         │
│            ┌───────────┐                                                   │
│            │  Seg 0    │  (one larger segment)                            │
│            │  450KB    │                                                   │
│            └───────────┘                                                   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Filter Cache

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           FILTER CACHE                                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   Filter queries are cached as BITSETS for fast reuse:                      │
│                                                                             │
│   Query: status = 'FAILED'                                                  │
│                                                                             │
│   First execution:                                                          │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │ 1. Lookup in inverted index: FAILED → [doc1, doc4, doc5, doc8, ...] │  │
│   │                                                                      │  │
│   │ 2. Build bitset:                                                     │  │
│   │                                                                      │  │
│   │    Doc IDs:    1    2    3    4    5    6    7    8    9   10       │  │
│   │    Bitset:   [ 1    0    0    1    1    0    0    1    0    0  ...]  │  │
│   │               ↑              ↑    ↑              ↑                   │  │
│   │             doc1           doc4 doc5           doc8                  │  │
│   │                                                                      │  │
│   │ 3. Cache bitset with key: "status:FAILED"                           │  │
│   │                                                                      │  │
│   │ Time: 50ms                                                           │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│   Second execution (same filter):                                           │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │ 1. Check cache for "status:FAILED"                                   │  │
│   │                                                                      │  │
│   │ 2. CACHE HIT! Return bitset directly                                │  │
│   │                                                                      │  │
│   │ Time: 1ms                                                            │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│                                                                             │
│   COMBINING FILTERS (Bitset operations):                                    │
│   ───────────────────────────────────────                                   │
│                                                                             │
│   Query: status = 'FAILED' AND platform = 'linux-x64'                      │
│                                                                             │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │                                                                      │  │
│   │   Bitset A (status=FAILED):                                         │  │
│   │   [ 1    0    0    1    1    0    0    1    0    0  ...]            │  │
│   │                                                                      │  │
│   │   Bitset B (platform=linux-x64):                                    │  │
│   │   [ 1    0    0    1    0    0    1    0    0    1  ...]            │  │
│   │                                                                      │  │
│   │   A AND B (bitwise AND):                                            │  │
│   │   [ 1    0    0    1    0    0    0    0    0    0  ...]            │  │
│   │     ↑              ↑                                                │  │
│   │   doc1           doc4                                                │  │
│   │                                                                      │  │
│   │   Result: [doc1, doc4]                                              │  │
│   │                                                                      │  │
│   │   Time: ~5ms (bitwise operations are CPU-native and FAST)           │  │
│   │                                                                      │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│   WHY THIS IS FAST:                                                         │
│   • Bitsets are memory-efficient (1 bit per document)                      │
│   • AND/OR operations use CPU SIMD instructions                            │
│   • Common filters stay cached                                             │
│   • No need to re-read from disk                                           │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Summary: Complete Flow

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        COMPLETE SYSTEM FLOW                                 │
└─────────────────────────────────────────────────────────────────────────────┘

WRITE PATH (async, 2-5 seconds):
════════════════════════════════

  Service          DB              Kafka           Consumer         ES
     │              │                │                │              │
     │──write──────▶│                │                │              │
     │              │──commit────────│                │              │
     │◀─response────│                │                │              │
     │              │                │                │              │
     │──publish────────────────────▶│                │              │
     │              │                │──consume──────▶│              │
     │              │                │                │──fetch──────▶│ (services)
     │              │                │                │◀─data────────│
     │              │                │                │──index──────────────────▶│
     │              │                │                │              │◀─ack──────│


READ PATH (sync, 50-100ms):
═══════════════════════════

  Dashboard       Search API       Elasticsearch
     │                │                   │
     │──POST /search─▶│                   │
     │                │──query───────────▶│
     │                │                   │──check cache
     │                │                   │──query shards (parallel)
     │                │                   │──merge results
     │                │◀─hits─────────────│
     │                │──map to DTO       │
     │◀─response──────│                   │


DATA STORAGE:
═════════════

  PostgreSQL (Source of Truth)     Elasticsearch (Search Index)
  ┌─────────────────────────┐      ┌───────────────────────────────┐
  │ Normalized tables:      │      │ Denormalized documents:       │
  │                         │      │                               │
  │ • test_runs             │ ───▶ │ {                             │
  │ • projects              │      │   testName, testStatus,       │
  │ • users                 │      │   projectName, changelist,    │
  │ • bugs                  │      │   owner, ownerTeam,           │
  │                         │      │   bugId, bugStatus,           │
  │ Each in separate DB!    │      │   ...all fields combined      │
  └─────────────────────────┘      │ }                             │
                                   │                               │
                                   │ Stored as inverted index     │
                                   │ + stored fields              │
                                   └───────────────────────────────┘
```

---

## Interview Talking Points

1. **"How does write path work?"**
   > "Service writes to its own PostgreSQL, commits, then publishes an event to Kafka. An ES sync consumer picks up the message, fetches data from multiple services to enrich, builds a denormalized document, and indexes to ES. Total lag is 2-5 seconds."

2. **"How does read path work?"**
   > "Client sends search request to our API. We build an ES bool query with filter clauses. ES fans out to all shards in parallel, each shard checks its filter cache and inverted index, returns top N results. Coordinating node merges and returns. Total time 50-100ms."

3. **"How does ES store data?"**
   > "ES uses an inverted index - it maps terms to document IDs. When you query `status=FAILED`, ES does O(1) lookup to get [doc1, doc4, doc5]. For AND queries, it does bitset intersection which is CPU-native and fast. Documents are stored in segments on disk, with the inverted index, stored fields for retrieval, and doc values for sorting."

4. **"What's the filter cache?"**
   > "Filter results are cached as bitsets - one bit per document. When the same filter runs again, we get instant cache hit. When combining filters, we do bitwise AND/OR operations which are extremely fast. This is why our common dashboard queries are so quick."
