# Functional Requirements

## FR-1: Cross-Service Dashboard Filtering

Users must be able to filter test results across multiple dimensions from different services:

### Filter Dimensions

| Dimension | Source Service | Filter Fields |
|-----------|----------------|---------------|
| **Test Run** | Test Service | testName, testSuite, platform, testStatus |
| **Bug** | Bug Service | bugId, bugStatus, bugSeverity, hasBugHit |
| **Changelist** | Project Service | changelist, primary (branch) |
| **Project** | Project Service | projectName |
| **Image** | Image Service | imageId, imageBuildStatus |
| **Owner** | User Service | owner (email), ownerTeam |
| **Personality** | User Service | personality (developer, QA, etc.) |
| **Workspace** | Workspace Service | workspaceId, workspaceStatus, hostname |
| **Time** | All | createdAt, updatedAt (date ranges) |

---

## FR-2: Compound Query Support

Users must be able to combine filters with AND/OR logic across services:

### Example Queries

```
Example 1: Cross-service AND
testName = 'auth_login_test'
AND hasBugHit = true
AND owner = 'john@company.com'
AND createdAt > last 30 days

Example 2: OR within same dimension
platform IN ('linux-x64', 'darwin-arm64', 'windows-x64')

Example 3: Complex compound
(bugStatus = 'OPEN' OR bugStatus = 'IN_PROGRESS')
AND primary = 'main'
AND personality = 'developer'
AND projectName IN ('backend-api', 'auth-service')

Example 4: Nested conditions
(testStatus = 'FAILED' AND hasBugHit = true)
OR (testStatus = 'PASSED' AND bugStatus = 'OPEN')
```

---

## FR-3: Sorting and Pagination

| Feature | Requirement |
|---------|-------------|
| Sort fields | createdAt, updatedAt, testName, bugSeverity |
| Default sort | createdAt DESC (newest first) |
| Page sizes | 20, 50, 100 (user configurable) |
| Pagination type | Cursor-based (search_after) for consistency |
| Max results | 10,000 per query |

---

## FR-4: Full-Text Search

Search across multiple text fields:

```
Search term: "authentication timeout"

Matches in:
- testName: "auth_timeout_test"
- bugDescription: "Authentication timeout on login"
- errorMessage: "Connection timeout during authentication"
```

---

## FR-5: Aggregations / Faceted Search

Show counts by dimension for filtering UI:

```json
{
  "aggregations": {
    "byTestStatus": { "PASSED": 1200, "FAILED": 45, "SKIPPED": 12 },
    "byPlatform": { "linux-x64": 800, "darwin-arm64": 300, "windows-x64": 157 },
    "byBugSeverity": { "P1": 5, "P2": 20, "P3": 15, "P4": 5 },
    "byOwnerTeam": { "platform-team": 500, "auth-team": 350, "infra-team": 407 }
  }
}
```

---

## FR-6: Near Real-Time Updates

| Event | Visibility Target |
|-------|-------------------|
| New test run created | Visible in ES within 5 seconds |
| Bug linked to test | Updated in ES within 5 seconds |
| Test status changed | Reflected in ES within 5 seconds |
| Owner changed | Reflected in ES within 5 seconds |

---

## FR-7: Saved Searches / Presets

Users can save complex filter combinations:

```json
{
  "name": "My Team's P1 Bugs This Week",
  "filters": {
    "ownerTeam": "platform-team",
    "bugSeverity": "P1",
    "createdAt": { "gte": "now-7d" }
  }
}
```

---

## API Contract

### Search Endpoint

```http
POST /api/v1/test-results/search
Content-Type: application/json

{
  "filters": {
    "testName": "auth_login_test",
    "hasBugHit": true,
    "platform": ["linux-x64", "darwin-arm64"],
    "primary": "main",
    "owner": "john@company.com",
    "createdAt": {
      "gte": "2024-01-01T00:00:00Z",
      "lte": "2024-01-31T23:59:59Z"
    }
  },
  "search": "timeout error",
  "sort": {
    "field": "createdAt",
    "order": "desc"
  },
  "pagination": {
    "size": 20,
    "searchAfter": ["2024-01-15T10:30:00Z", "test-run-123"]
  },
  "includeAggregations": true
}
```

### Response

```json
{
  "total": 1523,
  "hits": [
    {
      "id": "test-run-456",
      "testName": "auth_login_test",
      "testSuite": "auth-integration",
      "testStatus": "FAILED",
      "platform": "linux-x64",

      "bugId": "BUG-789",
      "bugStatus": "OPEN",
      "bugSeverity": "P1",

      "projectName": "backend-api",
      "changelist": "CL-12345",
      "primary": "main",

      "owner": "john@company.com",
      "ownerTeam": "platform-team",
      "personality": "developer",

      "imageId": "img-abc",
      "workspaceId": "ws-xyz",

      "createdAt": "2024-01-15T10:25:00Z"
    }
  ],
  "aggregations": {
    "byTestStatus": { "PASSED": 1200, "FAILED": 323 },
    "byPlatform": { "linux-x64": 800, "darwin-arm64": 723 },
    "byBugSeverity": { "P1": 45, "P2": 120 }
  },
  "nextCursor": ["2024-01-15T10:25:00Z", "test-run-456"]
}
```

---

## Key Insight: Cross-Service Data in Single Response

```
Traditional approach (multiple API calls):
┌─────────────────────────────────────────────────────────────┐
│ GET /test-service/runs?name=auth_login_test                 │
│ GET /bug-service/bugs?testRunId=123                         │
│ GET /user-service/users?email=john@company.com              │
│ GET /project-service/changelists?id=CL-12345                │
│                                                             │
│ Then join in client/backend → 5-10 seconds                  │
└─────────────────────────────────────────────────────────────┘

Our approach (single ES query):
┌─────────────────────────────────────────────────────────────┐
│ POST /api/v1/test-results/search                            │
│ { filters across all dimensions }                           │
│                                                             │
│ ES returns pre-joined data → 50-100ms                       │
└─────────────────────────────────────────────────────────────┘
```
