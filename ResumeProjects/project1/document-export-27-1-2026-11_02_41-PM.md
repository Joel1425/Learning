# Dashboard

_Problem earlier: _We have multiple micro-services, each with its own PostgreSQL database. You **cannot JOIN across databases**. The traditional approach - calling each service API and joining in memory - took 5-10 seconds.

### API Aggregation (Slow)
```
Dashboard Request → Backend Service

1. Call Bug Service API: Get bug hits in last 30 days
   Response: 50,000 results

2. Call Test Service API: Get test runs for "auth_login_test"
   Response: 10,000 results

3. Call User Service API: Get entities owned by "john@company.com"
   Response: 5,000 results

4. Join in application memory:
   bugs ∩ tests ∩ users ∩ projects

5. Filter and return

Total time: 5-10 seconds
Memory usage: High (loading all results)
```
Elasticsearch solves this by acting as a **cross-service search index**. Each service publishes events to Kafka. An ES consumer aggregates these into denormalized documents containing data from all services. Now a complex query across bugs, tests, owners, and projects is a single ES query returning in 50-100ms."

### **1. Problem Statement**
Design a cross-service search index to optimise complex queries. 

---

### **2. Requirements**
#### Functional Requirements
- TBD
#### Non-Functional Requirements
- TBD
---

### **3. Core Entities**
- TBD
---

### **4. API routes**
TBD

---

### **5. High level design**
#### How data is published to elastic search?
-> Step-by-step process: 

Step 1: Service receives request (Eg: TestService receives a new test run result)

Step 2: Write to Service's Own Database

Step 3: Publish Event to Kafka

Step 4: ES Sync Consumer polls

Step 5: Enriches Data (refer to the FAQs #3)

Step 6: Builds document

Step 7: Index to elastic search



**_ES Internals_**

1. `Document arrives at Coordinating` 

2. `Route to correct shard (hash of doc ID)` 

3. `Primary shard writes to transaction` 

4. `Replicate to replica` 

5. `Return success to consumer` 



#### How data is fetched?
-> The client makes API requests to our Search Service (A4C Service), not directly to Elasticsearch. The service receives the search request, validates the input, applies security filters based on the user's permissions, converts the request into an ES query, and returns formatted results. This abstraction layer provides security, input validation, and the ability to fallback to PostgreSQL if Elasticsearch is unavailable.

Step-by-step process: 

Step 1: Search service receives request

Step 2: Build ES query

Step 3: ES query is sent to ES

Step 4: ES gives the results

Step 5: Results are sent to the client



What happens inside ES during query execution?



---

### 6. Deep dives
---

### FAQs
#### 1. Why Kafka?                                                                
• Decouples services from ES (ES down doesn't break writes)              
• Provides durability (messages persist until consumed)                  
• Enables retry on failures                                              
• Maintains ordering per entity (using key-based partitioning)
#### 2. How Data Gets Pushed to Kafka After DB Write
-> When a service saves data to its database, it doesn't immediately push to Kafka. Instead, it registers an event listener that only fires AFTER the database transaction successfully commits. This prevents a scenario where Kafka receives an event for data that later gets rolled back. The message sent to Kafka is lightweight - just the entity ID and action type. The ES consumer then fetches the full, fresh data directly from the source database.

#### What Goes in the Kafka Message?
```
┌────────────────────────────────────┐
│  Kafka Message (lightweight)       │
├────────────────────────────────────┤
│  {                                 │
│    "entityType": "BUG",            │
│    "entityId": "bug-12345",        │
│    "action": "CREATED",            │
│    "timestamp": "2024-01-15..."    │
│  }                                 │
└────────────────────────────────────┘

NOT this (too heavy, can become stale):

┌────────────────────────────────────┐
│  {                                 │
│    "entityType": "BUG",            │
│    "bugId": "bug-12345",           │
│    "bugTitle": "Login fails...",   │  ← Full data
│    "bugStatus": "OPEN",            │  ← Can change
│    "severity": "P1",               │  ← Between send
│    "assignee": "john@...",         │  ← And consume
│    ...50 more fields...            │
│  }                                 │
└────────────────────────────────────┘
```
**3. What do we mean by enriches data? How is it done? **

Enrichment means the ES consumer receives a minimal event with just an entity ID. It then calls multiple service APIs to fetch complete, fresh data. For example, when a test run is created, the consumer fetches test details from Test Service, bug info from Bug Service, project details from Project Service, and owner info from User Service. It combines all these into one denormalized document before indexing to Elasticsearch. This ensures we always index the latest data and can include fields that the originating service doesn't even know about.

## Real Example: Test Run Created
### Step 1: Kafka Message Arrives (Minimal Data)
```
┌─────────────────────────────────────┐
│  Message from Test Service          │
├─────────────────────────────────────┤
│  {                                  │
│    "entityType": "TEST_RUN",        │
│    "entityId": "test-run-456",      │
│    "action": "CREATED"              │
│  }                                  │
└─────────────────────────────────────┘

This tells us WHAT happened, but not the full picture.
```
### Step 2: ES Consumer Fetches Test Run Details
```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  Consumer calls Test Service API:                               │
│  GET /api/tests/test-run-456                                    │
│                                                                 │
│  Response:                                                      │
│  {                                                              │
│    "id": "test-run-456",                                        │
│    "testName": "auth_login_test",                               │
│    "testSuite": "auth-integration",                             │
│    "platform": "linux-x64",                                     │
│    "status": "FAILED",                                          │
│    "bugId": "bug-789",              ← Just an ID!               │
│    "projectId": "proj-101",         ← Just an ID!               │
│    "ownerId": "user-202"            ← Just an ID!               │
│  }                                                              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

Problem: We have IDs, but dashboard needs actual names and details!
```
### Step 3: Enrich by Fetching from Other Services
```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  ENRICHMENT CALLS (in parallel):                                │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ GET /api/bugs/bug-789                                   │    │
│  │                                                         │    │
│  │ Response:                                               │    │
│  │ {                                                       │    │
│  │   "bugId": "bug-789",                                   │    │
│  │   "bugStatus": "OPEN",                                  │    │
│  │   "bugSeverity": "P1",                                  │    │
│  │   "bugTitle": "Login fails on timeout"                  │    │
│  │ }                                                       │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ GET /api/projects/proj-101                              │    │
│  │                                                         │    │
│  │ Response:                                               │    │
│  │ {                                                       │    │
│  │   "projectId": "proj-101",                              │    │
│  │   "projectName": "backend-api",                         │    │
│  │   "primary": "main",                                    │    │
│  │   "changelist": "CL-12345"                              │    │
│  │ }                                                       │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ GET /api/users/user-202                                 │    │
│  │                                                         │    │
│  │ Response:                                               │    │
│  │ {                                                       │    │
│  │   "userId": "user-202",                                 │    │
│  │   "email": "john@company.com",                          │    │
│  │   "team": "platform-team",                              │    │
│  │   "personality": "developer"                            │    │
│  │ }                                                       │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```
### Step 4: Combine Everything into One Document
```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  ENRICHED DOCUMENT (ready for Elasticsearch):                   │
│                                                                 │
│  {                                                              │
│    "id": "test-run-456",                                        │
│                                                                 │
│    // From Test Service                                         │
│    "testName": "auth_login_test",                               │
│    "testSuite": "auth-integration",                             │
│    "platform": "linux-x64",                                     │
│    "testStatus": "FAILED",                                      │
│                                                                 │
│    // From Bug Service (ENRICHED)                               │
│    "bugId": "bug-789",                                          │
│    "bugStatus": "OPEN",                                         │
│    "bugSeverity": "P1",                                         │
│                                                                 │
│    // From Project Service (ENRICHED)                           │
│    "projectName": "backend-api",                                │
│    "primary": "main",                                           │
│    "changelist": "CL-12345",                                    │
│                                                                 │
│    // From User Service (ENRICHED)                              │
│    "owner": "john@company.com",                                 │
│    "ownerTeam": "platform-team",                                │
│    "personality": "developer"                                   │
│  }                                                              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```
```
┌──────────────────────────────┬───────────────────────────────────┐
│        WITHOUT ENRICHMENT    │        WITH ENRICHMENT            │
│      (Full data in Kafka)    │   (ID only, fetch fresh)          │
├──────────────────────────────┼───────────────────────────────────┤
│ Kafka: bugStatus="OPEN"      │ Kafka: testId="456"               │
│        (captured at T1)      │        (no bug data)              │
│                              │                                   │
│ Bug changes to CLOSED (T2)   │ Bug changes to CLOSED (T2)        │
│                              │                                   │
│ Consumer processes (T3)      │ Consumer processes (T3)           │
│ Uses old data from message   │ Fetches FRESH data from API       │
│                              │                                   │
│ ES gets: "OPEN" ❌           │ ES gets: "CLOSED" ✅              │
└──────────────────────────────┴───────────────────────────────────┘
```
#### 4. Why we used POST for search query?
-> We use POST because our search requests have complex nested filters and pagination cursors that don't fit well in query parameters. This follows Elasticsearch's own API design. For simple lookups like `GET /jobs/{id}`, we use GET.

4.1 We used graphQL for getting only the data of interest.

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │                                     │
│  1. WHY GraphQL?                                                │
│     "Clients request only the fields they need.                 │
│      Reduces over-fetching. One endpoint for everything."       │
│                                                                 │
│  2. HOW it works (high level)?                                  │
│     "Client sends a query specifying fields.                    │
│      Server has resolvers that fetch data.                      │
│      Response contains only requested fields."                  │
│                                                                 │
│  3. WHERE in our system?                                        │
│     "Search API layer. Sits between client and our              │
│      Elasticsearch service. ES queries remain the same."        │
│                                                                 │
│  4. WHO worked on it?                                           │
│     "A colleague with more GraphQL experience.                  │
│      I focused on ES indexing and Kafka sync."                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```


#### 5. What are the different types of Queries in Elastic search?
-> Elasticsearch has two main categories of queries. **Term-level queries** like `term`, `terms`, `range`, and `exists` do exact matching - great for filters like status dropdowns or date ranges. **Full-text queries** like `match` and `multi_match` analyze the input and find relevant documents - great for search boxes. We combine these using **bool queries** with `must` (AND), `should` (OR), `filter` (AND without scoring), and `must_not` (NOT). For dashboard filters, we mostly use term-level queries inside the `filter` clause because they're cached and don't need scoring."

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  TERM-LEVEL QUERIES              FULL-TEXT QUERIES              │
│  (Exact Match)                   (Analyzed/Searched)            │
│                                                                 │
│  • term                          • match                        │
│  • terms                         • match_phrase                 │
│  • range                         • multi_match                  │
│  • exists                                                       │
│  • prefix                                                       │
│  • wildcard                      COMPOUND QUERIES               │
│                                  (Combine Multiple)             │
│                                                                 │
│                                  • bool (must, should,          │
│                                          filter, must_not)      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```
#### Filter vs Query Context
```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  QUERY CONTEXT (must, should)                                   │
│  ────────────────────────────                                   │
│  • "How well does this match?"                                  │
│  • Calculates relevance score                                   │
│  • Slower (scoring takes time)                                  │
│  • Use for: full-text search                                    │
│                                                                 │
│  FILTER CONTEXT (filter, must_not)                              │
│  ─────────────────────────────────                              │
│  • "Does this match? Yes or No"                                 │
│  • No scoring (just include/exclude)                            │
│  • Faster + cached                                              │
│  • Use for: dropdowns, checkboxes, date ranges                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```
#### TERM Query (Exact Match)
**Use when:** You need an EXACT match (status, ID, email, enum values)

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  "Find all tests with status exactly 'FAILED'"                  │
│                                                                 │
│  {                                                              │
│    "term": {                                                    │
│      "testStatus": "FAILED"                                     │
│    }                                                            │
│  }                                                              │
│                                                                 │
│  ✅ Matches: "FAILED"                                           │
│  ❌ Does NOT match: "failed", "Failed", "FAIL"                  │
│                                                                 │
│  Best for: status, bugSeverity, platform, owner (email)         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```
---

#### TERMS Query (Multiple Exact Values)
**Use when:** You need to match ANY of several exact values (like SQL IN)

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  "Find tests on linux OR mac OR windows"                        │
│                                                                 │
│  {                                                              │
│    "terms": {                                                   │
│      "platform": ["linux-x64", "darwin-arm64", "windows-x64"]   │
│    }                                                            │
│  }                                                              │
│                                                                 │
│  SQL equivalent: WHERE platform IN ('linux-x64', 'darwin-arm64')│
│                                                                 │
│  Best for: multi-select filters in UI                           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```
---

#### RANGE Query (Numbers & Dates)
**Use when:** You need greater than, less than, between

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  "Find tests created in the last 30 days"                       │
│                                                                 │
│  {                                                              │
│    "range": {                                                   │
│      "createdAt": {                                             │
│        "gte": "now-30d",                                        │
│        "lte": "now"                                             │
│      }                                                          │
│    }                                                            │
│  }                                                              │
│                                                                 │
│  Operators:                                                     │
│  • gte = greater than or equal (>=)                             │
│  • gt  = greater than (>)                                       │
│  • lte = less than or equal (<=)                                │
│  • lt  = less than (<)                                          │
│                                                                 │
│  Best for: date filters, numeric ranges, age, count             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```
---

#### EXISTS Query (Field Present)
**Use when:** You want to check if a field has any value

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  "Find tests that have a bug linked (hasBugHit = true)"         │
│                                                                 │
│  {                                                              │
│    "exists": {                                                  │
│      "field": "bugId"                                           │
│    }                                                            │
│  }                                                              │
│                                                                 │
│  ✅ Matches: { "bugId": "bug-123" }                             │
│  ❌ Does NOT match: { "bugId": null } or field missing          │
│                                                                 │
│  Best for: "has attachment", "has bug", "has owner"             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```
---

#### PREFIX Query (Starts With)
**Use when:** You need autocomplete or "starts with" search

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  "Find tests starting with 'auth_'"                             │
│                                                                 │
│  {                                                              │
│    "prefix": {                                                  │
│      "testName": "auth_"                                        │
│    }                                                            │
│  }                                                              │
│                                                                 │
│  ✅ Matches: "auth_login_test", "auth_logout_test"              │
│  ❌ Does NOT match: "user_auth_test"                            │
│                                                                 │
│  Best for: autocomplete, typeahead search                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```
---

#### WILDCARD Query (Pattern Match)
**Use when:** You need flexible pattern matching (use sparingly - slow!)

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  "Find tests containing 'login' anywhere in name"               │
│                                                                 │
│  {                                                              │
│    "wildcard": {                                                │
│      "testName": "*login*"                                      │
│    }                                                            │
│  }                                                              │
│                                                                 │
│  * = any characters                                             │
│  ? = single character                                           │
│                                                                 │
│  ⚠️  WARNING: Slow! Scans all terms. Avoid leading wildcards.   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```
---

#### MATCH Query (Full-Text Search)
**Use when:** User types free text, you want smart matching

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  "Search for 'authentication timeout error'"                    │
│                                                                 │
│  {                                                              │
│    "match": {                                                   │
│      "errorMessage": "authentication timeout error"             │
│    }                                                            │
│  }                                                              │
│                                                                 │
│  How it works:                                                  │
│  1. Breaks input into tokens: [authentication, timeout, error]  │
│  2. Searches for documents containing ANY of these words        │
│  3. Ranks by relevance (more matches = higher score)            │
│                                                                 │
│  ✅ Matches: "timeout during authentication"                    │
│  ✅ Matches: "error in auth module"                             │
│  ✅ Matches: "connection timeout" (partial match, lower score)  │
│                                                                 │
│  Best for: search boxes, free text input                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```
---

#### MATCH_PHRASE Query (Exact Phrase)
**Use when:** Words must appear together in order

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  "Search for exact phrase 'connection timeout'"                 │
│                                                                 │
│  {                                                              │
│    "match_phrase": {                                            │
│      "errorMessage": "connection timeout"                       │
│    }                                                            │
│  }                                                              │
│                                                                 │
│  ✅ Matches: "got connection timeout on server"                 │
│  ❌ Does NOT match: "timeout on connection"                     │
│  ❌ Does NOT match: "connection failed with timeout"            │
│                                                                 │
│  Best for: error messages, exact quotes                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```
---

#### MULTI_MATCH Query (Search Multiple Fields)
**Use when:** Single search box should search across many fields

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  "Search 'timeout' in testName, errorMessage, and bugTitle"     │
│                                                                 │
│  {                                                              │
│    "multi_match": {                                             │
│      "query": "timeout",                                        │
│      "fields": ["testName", "errorMessage", "bugTitle"]         │
│    }                                                            │
│  }                                                              │
│                                                                 │
│  Finds documents where "timeout" appears in ANY of these fields │
│                                                                 │
│  Best for: global search box in dashboard                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```
---

#### BOOL Query (Combine Everything)
**Use when:** You need AND, OR, NOT logic

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  "Find FAILED tests on linux, owned by john, with P1 bugs"      │
│                                                                 │
│  {                                                              │
│    "bool": {                                                    │
│      "must": [        ← AND (all required)                      │
│        { "term": { "testStatus": "FAILED" } }                   │
│      ],                                                         │
│      "filter": [      ← AND (no scoring, faster, cached)        │
│        { "term": { "platform": "linux-x64" } },                 │
│        { "term": { "owner": "john@company.com" } },             │
│        { "term": { "bugSeverity": "P1" } }                      │
│      ],                                                         │
│      "should": [      ← OR (optional, boosts score)             │
│        { "term": { "primary": "main" } }                        │
│      ],                                                         │
│      "must_not": [    ← NOT (exclude these)                     │
│        { "term": { "testSuite": "flaky-tests" } }               │
│      ]                                                          │
│    }                                                            │
│  }                                                              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```
### Bool Query Clauses Explained
```
┌────────────┬─────────────────────────────────────────────────────┐
│  Clause    │  Meaning                                            │
├────────────┼─────────────────────────────────────────────────────┤
│  must      │  MUST match (AND). Affects relevance score.         │
│  filter    │  MUST match (AND). No scoring. Faster. Cached.      │
│  should    │  SHOULD match (OR). Optional. Boosts score if match.│
│  must_not  │  MUST NOT match (NOT). Excludes documents.          │
└────────────┴─────────────────────────────────────────────────────┘
```
---

#### When to Use What?
```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  USER ACTION                         QUERY TYPE                 │
│  ───────────────────────────────────────────────────────        │
│                                                                 │
│  Dropdown: "Status = FAILED"         term                       │
│  Multi-select: "Platform = [a,b,c]"  terms                      │
│  Date picker: "Last 30 days"         range                      │
│  Checkbox: "Has bug attached"        exists                     │
│  Autocomplete typing                 prefix                     │
│  Search box (free text)              match / multi_match        │
│  Search exact phrase                 match_phrase               │
│  Complex filter form                 bool (combining above)     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```
#### 6. What happens inside ES during query execution?
```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    ELASTICSEARCH QUERY EXECUTION                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   ┌──────────────────────────────────────────────────────────────────────┐  │
│   │                      ELASTICSEARCH CLUSTER                           │  │
│   │                                                                      │  │
│   │   Query arrives at COORDINATING NODE                                 │  │
│   │              │                                                       │  │
│   │              ▼                                                       │  │
│   │   ┌──────────────────────────────────────┐                           │  │
│   │   │      1. PARSE QUERY                  │                           │  │
│   │   │                                      │                           │  │
│   │   │   Parse JSON → Query AST             │                           │  │
│   │   │   Identify: bool query with filters  │                           │  │
│   │   └──────────────────────────────────────┘                           │  │
│   │              │                                                       │  │
│   │              ▼                                                       │  │
│   │   ┌───────────────────────────────────────┐                          │  │
│   │   │      2. IDENTIFY TARGET SHARDS        │                          │  │
│   │   │                                       │                          │  │
│   │   │   No routing specified → Query ALL    │                          │  │
│   │   │   shards: [Shard 0, Shard 1, Shard 2] │                          │  │
│   │   └───────────────────────────────────────┘                          │  │
│   │              │                                                       │  │
│   │              │  FAN OUT (parallel)                                   │  │
│   │    ┌─────────┼─────────┐                                             │  │
│   │    │         │         │                                             │  │
│   │    ▼         ▼         ▼                                             │  │
│   │ ┌────────┐ ┌────────┐ ┌────────┐                                     │  │
│   │ │Shard 0 │ │Shard 1 │ │Shard 2 │                                     │  │
│   │ │        │ │        │ │        │                                     │  │
│   │ │ ┌────┐ │ │ ┌────┐ │ │ ┌────┐ │                                     │  │
│   │ │ │ 3a │ │ │ │ 3a │ │ │ │ 3a │ │  3a. Check filter cache             │  │
│   │ │ └────┘ │ │ └────┘ │ │ └────┘ │                                     │  │
│   │ │   │    │ │   │    │ │   │    │                                     │  │
│   │ │   ▼    │ │   ▼    │ │   ▼    │                                     │  │
│   │ │ ┌────┐ │ │ ┌────┐ │ │ ┌────┐ │  3b. Query inverted index           │  │
│   │ │ │ 3b │ │ │ │ 3b │ │ │ │ 3b │ │      (see next section)             │  │
│   │ │ └────┘ │ │ └────┘ │ │ └────┘ │                                     │  │
│   │ │   │    │ │   │    │ │   │    │                                     │  │
│   │ │   ▼    │ │   ▼    │ │   ▼    │                                     │  │
│   │ │ ┌────┐ │ │ ┌────┐ │ │ ┌────┐ │  3c. Local sort, return top 20      │  │
│   │ │ │ 3c │ │ │ │ 3c │ │ │ │ 3c │ │                                     │  │
│   │ │ └────┘ │ │ └────┘ │ │ └────┘ │                                     │  │
│   │ │   │    │ │   │    │ │   │    │                                     │  │
│   │ └───┼────┘ └───┼────┘ └───┼────┘                                     │  │
│   │     │          │          │                                          │  │
│   │     └──────────┼──────────┘                                          │  │
│   │                │                                                     │  │
│   │                ▼                                                     │  │
│   │   ┌───────────────────────────────────────┐                          │  │
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
│   │   └───────────────────────────────────────┘                          │  │
│   │                │                                                     │  │
│   │                ▼                                                     │  │
│   │   ┌───────────────────────────────────────┐                          │  │
│   │   │      5. FETCH PHASE                   │                          │  │
│   │   │                                       │                          │  │
│   │   │   Fetch actual document _source for   │                          │  │
│   │   │   the final 20 documents              │                          │  │
│   │   └───────────────────────────────────────┘                          │  │
│   │                │                                                     │  │
│   │                ▼                                                     │  │
│   │            RETURN TO CLIENT                                          │  │
│   │                                                                      │  │
│   └──────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│   LATENCY BREAKDOWN:                                                        │
│   ┌──────────────────────────────────────────────────────────────────────┐  │
│   │ Parse query:              1-2 ms                                     │  │
│   │ Fan out to shards:        1-2 ms                                     │  │
│   │ Shard query (parallel):   10-30 ms  ← Most time spent here           │  │
│   │ Merge results:            2-5 ms                                     │  │
│   │ Fetch documents:          5-10 ms                                    │  │
│   │ ─────────────────────────────────                                    │  │
│   │ TOTAL:                    20-50 ms                                   │  │
│   └──────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```
#### 7. How does the fetch phase work?
-> The Key Insight: We DON'T Go Back to DB!

Since we denormalized data at write time, Elasticsearch already contains the complete document - we don't need to go back to the database for search results. For caching, there are multiple levels: ES automatically caches filter results as bitsets for common filters like platform or status. 

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  Remember: We DENORMALIZED the data                             │
│                                                                 │
│  ES already has the COMPLETE document:                          │
│  {                                                              │
│    "id": "test-run-456",                                        │
│    "testName": "auth_login_test",      ← From Test Service      │
│    "platform": "linux-x64",                                     │
│    "bugId": "bug-789",                 ← From Bug Service       │
│    "bugStatus": "OPEN",                                         │
│    "bugSeverity": "P1",                                         │
│    "owner": "john@company.com",        ← From User Service      │
│    "ownerTeam": "platform-team",                                │
│    "projectName": "backend-api",       ← From Project Service   │
│    ...                                                          │
│  }                                                              │
│                                                                 │
│  We paid the cost of enrichment at WRITE time                   │
│  So at READ time, ES has everything we need!                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```
#### Response Flow (No DB Call Needed)
```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  Step 1: ES Returns Full Documents                              │
│  ─────────────────────────────────                              │
│                                                                 │
│  ES Response:                                                   │
│  {                                                              │
│    "hits": {                                                    │
│      "total": 1523,                                             │
│      "hits": [                                                  │
│        { "_source": { full doc 1 } },                           │
│        { "_source": { full doc 2 } },                           │
│        { "_source": { full doc 3 } },                           │
│        ...                                                      │
│      ]                                                          │
│    }                                                            │
│  }                                                              │
│                                                                 │
│                          │                                      │
│                          ▼                                      │
│                                                                 │
│  Step 2: Service Formats Response                               │
│  ────────────────────────────────                               │
│                                                                 │
│  • Extract _source from each hit                                │
│  • Maybe rename fields for API contract                         │
│  • Add pagination cursor                                        │
│  • Add aggregation results                                      │
│                                                                 │
│                          │                                      │
│                          ▼                                      │
│                                                                 │
│  Step 3: Return to Client                                       │
│  ────────────────────────                                       │
│                                                                 │
│  {                                                              │
│    "total": 1523,                                               │
│    "results": [ doc1, doc2, doc3, ... ],                        │
│    "nextCursor": "..."                                          │
│  }                                                              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```
8. What if ES is down? How do we handle the responses then?

-> When Elasticsearch is down, we use graceful degradation with a fallback to PostgreSQL. We implement a circuit breaker pattern - after 5 consecutive failures, the circuit opens and requests go directly to the fallback without even trying ES, avoiding timeout delays. The fallback queries individual service databases and joins results in memory. It's slower - 3-10 seconds instead of 50ms - and some features like full-text search are disabled. We show users a warning banner that search is in degraded mode. Meanwhile, we log metrics, trigger alerts for the on-call team, and the circuit breaker periodically tests if ES is back. Once ES recovers, the circuit closes and normal operation resumes automatically.

The Problem

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  Normal Flow:                                                   │
│                                                                 │
│  Client → Search Service → Elasticsearch → Results (50ms)       │
│                                   │                             │
│                                   X                             │
│                               ES DOWN!                          │
│                                                                 │
│  What do we do? Show error? Give up?                            │
│                                                                 │
│  NO! We have a FALLBACK.                                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```
#### Fallback Strategy
```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  Step 1: Try Elasticsearch First                                │
│  ───────────────────────────────                                │
│                                                                 │
│  Client Request                                                 │
│       │                                                         │
│       ▼                                                         │
│  Search Service                                                 │
│       │                                                         │
│       ▼                                                         │
│  Try Elasticsearch ──────► SUCCESS? ──────► Return results      │
│       │                                                         │
│       │ FAILURE (timeout, connection refused, 5xx)              │
│       ▼                                                         │
│                                                                 │
│  Step 2: Fallback to PostgreSQL                                 │
│  ───────────────────────────────                                │
│                                                                 │
│  Query individual service databases                             │
│       │                                                         │
│       ├──► Test Service DB                                      │
│       ├──► Bug Service DB                                       │
│       ├──► User Service DB                                      │
│       └──► Project Service DB                                   │
│       │                                                         │
│       ▼                                                         │
│  Join results in memory                                         │
│       │                                                         │
│       ▼                                                         │
│  Return results (slower: 3-10 seconds)                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```
#### What the Fallback Looks Like?
#### Normal Path (ES Available)
```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  Single query to ES:                                            │
│                                                                 │
│  POST /test-results/_search                                     │
│  {                                                              │
│    "query": {                                                   │
│      "bool": {                                                  │
│        "filter": [                                              │
│          { "term": { "platform": "linux-x64" } },               │
│          { "term": { "testStatus": "FAILED" } },                │
│          { "term": { "owner": "john@company.com" } }            │
│        ]                                                        │
│      }                                                          │
│    }                                                            │
│  }                                                              │
│                                                                 │
│  Time: ~50ms                                                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```
#### Fallback Path (ES Down)
```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  Multiple API calls + in-memory join:                           │
│                                                                 │
│  1. GET /test-service/runs?platform=linux&status=FAILED         │
│     Response: 5000 test runs                                    │
│                                                                 │
│  2. GET /user-service/entities?owner=john@company.com           │
│     Response: 2000 entity IDs                                   │
│                                                                 │
│  3. In-memory: Find intersection                                │
│     testRuns ∩ userEntities = 150 matches                       │
│                                                                 │
│  4. For each match, fetch bug details if needed                 │
│     GET /bug-service/bugs?testRunId=...                         │
│                                                                 │
│  Time: 3-10 seconds                                             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```
#### Circuit Breaker Pattern
```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  Problem: If ES is down, we don't want to keep trying           │
│           and waiting for timeout on EVERY request              │
│                                                                 │
│  Solution: CIRCUIT BREAKER                                      │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                                                         │    │
│  │   CLOSED (Normal)                                       │    │
│  │   ────────────────                                      │    │
│  │   • All requests go to ES                               │    │
│  │   • If 5 failures in 30 seconds → OPEN circuit          │    │
│  │                                                         │    │
│  │              │                                          │    │
│  │              ▼ (failures exceed threshold)              │    │
│  │                                                         │    │
│  │   OPEN (Tripped)                                        │    │
│  │   ──────────────                                        │    │
│  │   • All requests go DIRECTLY to fallback                │    │
│  │   • Don't even try ES (skip the timeout wait)           │    │
│  │   • After 60 seconds → HALF-OPEN                        │    │
│  │                                                         │    │
│  │              │                                          │    │
│  │              ▼ (wait period expires)                    │    │
│  │                                                         │    │
│  │   HALF-OPEN (Testing)                                   │    │
│  │   ────────────────────                                  │    │
│  │   • Let ONE request try ES                              │    │
│  │   • If success → CLOSED (ES is back!)                   │    │
│  │   • If failure → OPEN (still down)                      │    │
│  │                                                         │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```


