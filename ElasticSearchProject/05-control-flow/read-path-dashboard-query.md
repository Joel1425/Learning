# Read Path: Dashboard Query Flow

## Overview

This document explains how data flows from a user's dashboard request through Elasticsearch and back.

---

## Complete Read Path Diagram

```
┌──────────────────────────────────────────────────────────────────────────┐
│                            READ PATH                                     │
└──────────────────────────────────────────────────────────────────────────┘

Step 1: User Request
┌─────────────────┐
│    Frontend     │
│   (Dashboard)   │
│                 │
│ Filter:         │
│ - status: FAILED│
│ - project: X    │
│ - last 7 days   │
└────────┬────────┘
         │ POST /api/v1/jobs/search
         │ {
         │   "statuses": ["FAILED"],
         │   "projectName": "backend-api",
         │   "createdAfter": "2024-01-08T00:00:00Z"
         │ }
         ▼
Step 2: API Gateway
┌─────────────────┐
│   API Gateway   │
│ (AWS + LB)      │
│                 │
│ - Rate limiting │
│ - Auth check    │
│ - Routing       │
└────────┬────────┘
         │
         ▼
Step 3: Backend Service
┌─────────────────────────────────────────────────────────────┐
│                    Backend Service                          │
│                                                             │
│  ┌───────────────────────────────────────────────────────┐ │
│  │              JobSearchController                      │ │
│  │                                                       │ │
│  │  @PostMapping("/search")                              │ │
│  │  public SearchResponse search(@RequestBody req) {     │ │
│  │      validateRequest(req);                            │ │
│  │      return jobSearchService.search(req);             │ │
│  │  }                                                    │ │
│  └───────────────────────────────────────────────────────┘ │
│                          │                                  │
│                          ▼                                  │
│  ┌───────────────────────────────────────────────────────┐ │
│  │               JobSearchService                        │ │
│  │                                                       │ │
│  │  public SearchResponse search(JobSearchRequest req) { │ │
│  │      try {                                            │ │
│  │          return searchElasticsearch(req);             │ │
│  │      } catch (ElasticsearchException e) {             │ │
│  │          log.warn("ES failed, falling back");         │ │
│  │          return searchPostgres(req);  // Fallback     │ │
│  │      }                                                │ │
│  │  }                                                    │ │
│  └───────────────────────────────────────────────────────┘ │
│                          │                                  │
│                          ▼                                  │
│  ┌───────────────────────────────────────────────────────┐ │
│  │            ElasticsearchQueryBuilder                  │ │
│  │                                                       │ │
│  │  BoolQueryBuilder query = QueryBuilders.boolQuery()   │ │
│  │      .filter(termsQuery("status", ["FAILED"]))        │ │
│  │      .filter(termQuery("projectName", "backend-api")) │ │
│  │      .filter(rangeQuery("createdAt").gte(...));       │ │
│  │                                                       │ │
│  │  SearchRequest esRequest = new SearchRequest("jobs")  │ │
│  │      .source(new SearchSourceBuilder()                │ │
│  │          .query(query)                                │ │
│  │          .sort("createdAt", DESC)                     │ │
│  │          .size(20));                                  │ │
│  └───────────────────────────────────────────────────────┘ │
│                                                             │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
Step 4: Elasticsearch Query
┌─────────────────────────────────────────────────────────────┐
│                     Elasticsearch                           │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │               Coordinating Node                      │   │
│  │                                                      │   │
│  │  1. Parse query                                      │   │
│  │  2. Determine which shards to query                  │   │
│  │  3. Fan out to shard nodes                           │   │
│  └────────────────────┬─────────────────────────────────┘   │
│                       │                                      │
│         ┌─────────────┼─────────────┐                       │
│         ▼             ▼             ▼                       │
│  ┌───────────┐ ┌───────────┐ ┌───────────┐                 │
│  │  Shard 0  │ │  Shard 1  │ │  Shard 2  │                 │
│  │           │ │           │ │           │                 │
│  │ 1. Check  │ │ 1. Check  │ │ 1. Check  │                 │
│  │   filter  │ │   filter  │ │   filter  │                 │
│  │   cache   │ │   cache   │ │   cache   │                 │
│  │           │ │           │ │           │                 │
│  │ 2. If miss│ │ 2. If miss│ │ 2. If miss│                 │
│  │   scan    │ │   scan    │ │   scan    │                 │
│  │   inverted│ │   inverted│ │   inverted│                 │
│  │   index   │ │   index   │ │   index   │                 │
│  │           │ │           │ │           │                 │
│  │ 3. Return │ │ 3. Return │ │ 3. Return │                 │
│  │   top 20  │ │   top 20  │ │   top 20  │                 │
│  └─────┬─────┘ └─────┬─────┘ └─────┬─────┘                 │
│        │             │             │                        │
│        └─────────────┼─────────────┘                        │
│                      ▼                                      │
│  ┌─────────────────────────────────────────────────────┐   │
│  │               Coordinating Node                      │   │
│  │                                                      │   │
│  │  1. Merge results from all shards                    │   │
│  │  2. Global sort                                      │   │
│  │  3. Return top 20                                    │   │
│  └────────────────────┬─────────────────────────────────┘   │
│                       │                                      │
└───────────────────────┼──────────────────────────────────────┘
                        │
                        ▼
Step 5: Response Processing
┌─────────────────────────────────────────────────────────────┐
│                    Backend Service                          │
│                                                             │
│  ┌───────────────────────────────────────────────────────┐ │
│  │              ResponseMapper                           │ │
│  │                                                       │ │
│  │  SearchResponse response = new SearchResponse();      │ │
│  │  response.setTotal(esResponse.getHits().getTotalHits())│ │
│  │  response.setHits(                                    │ │
│  │      esResponse.getHits().getHits().stream()          │ │
│  │          .map(this::toJobDto)                         │ │
│  │          .collect(toList()));                         │ │
│  │  response.setNextCursor(                              │ │
│  │      getSearchAfterValues(lastHit));                  │ │
│  └───────────────────────────────────────────────────────┘ │
│                                                             │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
Step 6: Client Response
┌─────────────────┐
│    Frontend     │
│   (Dashboard)   │
│                 │
│ Response:       │
│ {               │
│  total: 45,     │
│  hits: [...],   │
│  nextCursor:[..]│
│ }               │
└─────────────────┘
```

---

## Sequence Diagram

```
Frontend    Gateway    Backend    Elasticsearch    PostgreSQL
   │           │          │             │               │
   │──POST────▶│          │             │               │
   │           │──forward▶│             │               │
   │           │          │             │               │
   │           │          │──build query│               │
   │           │          │──search────▶│               │
   │           │          │             │──query shards─│
   │           │          │             │──merge results│
   │           │          │◀───hits─────│               │
   │           │          │             │               │
   │           │          │──map to DTO─│               │
   │           │◀──response│             │               │
   │◀──JSON────│          │             │               │
   │           │          │             │               │
```

---

## Code Implementation

### Controller

```java
@RestController
@RequestMapping("/api/v1/jobs")
@RequiredArgsConstructor
public class JobSearchController {

    private final JobSearchService searchService;

    @PostMapping("/search")
    public ResponseEntity<SearchResponse> search(
            @Valid @RequestBody JobSearchRequest request,
            @AuthenticationPrincipal UserDetails user) {

        // Optional: Add user context for access control
        request.setRequestingUserId(user.getUserId());

        SearchResponse response = searchService.search(request);
        return ResponseEntity.ok(response);
    }
}
```

### Service

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class JobSearchService {

    private final ElasticsearchOperations esOperations;
    private final JobRepository jobRepository;  // Fallback
    private final JobSearchQueryBuilder queryBuilder;
    private final MeterRegistry meterRegistry;

    @Timed(value = "job.search.latency", percentiles = {0.5, 0.95, 0.99})
    public SearchResponse search(JobSearchRequest request) {
        try {
            return searchElasticsearch(request);
        } catch (Exception e) {
            log.warn("Elasticsearch search failed, falling back to PostgreSQL", e);
            meterRegistry.counter("job.search.fallback").increment();
            return searchPostgres(request);
        }
    }

    private SearchResponse searchElasticsearch(JobSearchRequest request) {
        // Build query
        NativeSearchQuery query = queryBuilder.build(request);

        // Execute
        SearchHits<JobDocument> hits = esOperations.search(query, JobDocument.class);

        // Map response
        return mapToResponse(hits, request);
    }

    private SearchResponse searchPostgres(JobSearchRequest request) {
        // Fallback implementation using JPA Specifications
        Specification<Job> spec = JobSpecifications.fromRequest(request);
        Page<Job> page = jobRepository.findAll(spec,
            PageRequest.of(0, request.getPageSize(),
                Sort.by(Sort.Direction.DESC, "createdAt")));

        return mapToResponse(page);
    }

    private SearchResponse mapToResponse(SearchHits<JobDocument> hits,
                                          JobSearchRequest request) {
        List<JobDto> jobDtos = hits.getSearchHits().stream()
            .map(hit -> toDto(hit.getContent()))
            .collect(Collectors.toList());

        SearchResponse response = new SearchResponse();
        response.setTotal(hits.getTotalHits());
        response.setHits(jobDtos);

        // Set cursor for pagination
        if (!hits.getSearchHits().isEmpty()) {
            SearchHit<JobDocument> lastHit = hits.getSearchHits()
                .get(hits.getSearchHits().size() - 1);
            response.setNextCursor(Arrays.asList(lastHit.getSortValues()));
        }

        return response;
    }
}
```

### Query Builder

```java
@Component
public class JobSearchQueryBuilder {

    public NativeSearchQuery build(JobSearchRequest request) {
        BoolQueryBuilder boolQuery = QueryBuilders.boolQuery();

        // Status filter
        if (hasValues(request.getStatuses())) {
            List<String> statusStrings = request.getStatuses().stream()
                .map(Enum::name)
                .collect(Collectors.toList());
            boolQuery.filter(QueryBuilders.termsQuery("status", statusStrings));
        }

        // Project filter
        if (StringUtils.hasText(request.getProjectName())) {
            boolQuery.filter(QueryBuilders.termQuery("projectName",
                request.getProjectName()));
        }

        // Date range
        addDateRangeFilter(boolQuery, request);

        // User filter (if not admin)
        if (!request.isAdmin() && request.getRequestingUserId() != null) {
            boolQuery.filter(QueryBuilders.termQuery("userId",
                request.getRequestingUserId()));
        }

        // Build final query
        NativeSearchQueryBuilder queryBuilder = new NativeSearchQueryBuilder()
            .withQuery(boolQuery)
            .withSort(Sort.by(Sort.Direction.fromString(
                request.getSortOrder().name()), request.getSortField()))
            .withPageable(PageRequest.of(0, request.getPageSize()));

        // Cursor-based pagination
        if (hasValues(request.getSearchAfter())) {
            queryBuilder.withSearchAfter(request.getSearchAfter());
        }

        return queryBuilder.build();
    }

    private void addDateRangeFilter(BoolQueryBuilder query,
                                     JobSearchRequest request) {
        if (request.getCreatedAfter() == null &&
            request.getCreatedBefore() == null) {
            return;
        }

        RangeQueryBuilder rangeQuery = QueryBuilders.rangeQuery("createdAt");

        if (request.getCreatedAfter() != null) {
            rangeQuery.gte(request.getCreatedAfter().toEpochMilli());
        }
        if (request.getCreatedBefore() != null) {
            rangeQuery.lte(request.getCreatedBefore().toEpochMilli());
        }

        query.filter(rangeQuery);
    }
}
```

---

## Latency Breakdown

| Step | Typical Time | Notes |
|------|--------------|-------|
| Network: Client → Gateway | 10-30ms | Depends on location |
| Gateway processing | 1-5ms | Auth, routing |
| Backend query building | 1-2ms | CPU bound |
| Network: Backend → ES | 1-5ms | Same datacenter |
| ES query execution | 10-50ms | Depends on complexity |
| ES result serialization | 5-10ms | JSON encoding |
| Backend response mapping | 1-5ms | DTO conversion |
| Network: Backend → Client | 10-30ms | Depends on location |
| **Total** | **40-140ms** | P95 target: < 200ms |

---

## Interview Talking Points

1. **"Walk me through the read path."**
   > "The frontend sends a search request with filters to our backend. The backend builds an Elasticsearch bool query using filter context for caching benefits. ES fans out to all shards, each checks its filter cache, merges results, and returns. We map to DTOs and include a cursor for pagination."

2. **"What if Elasticsearch is down?"**
   > "We have a fallback to PostgreSQL. It's slower but works. We log a warning and increment a metric so we know to investigate. For a dashboard, slightly slower results are better than no results."

3. **"How do you handle access control?"**
   > "Non-admin users can only see their own jobs. We add a filter clause for userId based on the authenticated user. This happens at the query building layer, so it's transparent to the rest of the code."
