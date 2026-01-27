# Strong vs Weak Answers

Learn the difference between SDE-2 level answers and junior/vague answers.

---

## Question 1: "Why did you choose Elasticsearch?"

### Weak Answer ❌

> "Elasticsearch is good for search and it's faster than SQL databases. My manager said we should use it."

**Why it's weak:**
- No specific reasoning
- Defers to authority instead of understanding
- Doesn't explain the actual problem

### Strong Answer ✓

> "Our PostgreSQL queries with multiple filter conditions on 10 million rows were taking 2-3 seconds. The B-tree indexes in PostgreSQL work well for single-column lookups, but for combining multiple filters like status, project, and date range, it struggles with index intersection.
>
> Elasticsearch uses an inverted index where term lookups are O(1) and AND/OR operations are just bitset intersections. This brought our typical query from 2 seconds to 50 milliseconds.
>
> We kept PostgreSQL as the source of truth for ACID transactions - ES is purely for read optimization."

**Why it's strong:**
- Specific problem with numbers
- Explains the technical difference
- Shows understanding of trade-offs
- Mentions what wasn't changed

---

## Question 2: "How do you handle data consistency?"

### Weak Answer ❌

> "We write to both PostgreSQL and Elasticsearch and they stay in sync automatically."

**Why it's weak:**
- Factually inaccurate (dual writes are problematic)
- No mention of consistency model
- Doesn't address timing or failures

### Strong Answer ✓

> "We use eventual consistency with a 2-5 second lag. Here's the flow:
>
> 1. Write to PostgreSQL in a transaction
> 2. After commit, publish an event to Kafka
> 3. Consumer fetches latest state from PostgreSQL and indexes to ES
>
> We have a backup poller for any missed events.
>
> This eventual consistency is acceptable for our dashboard because it's informational, not transactional. For critical operations, we query PostgreSQL directly."

**Why it's strong:**
- Clear explanation of the mechanism
- Acknowledges the consistency model explicitly
- Explains why it's acceptable
- Shows awareness of when it wouldn't work

---

## Question 3: "What happens if Elasticsearch goes down?"

### Weak Answer ❌

> "We have replicas so it should be fine. Elasticsearch is highly available."

**Why it's weak:**
- Doesn't answer what the application does
- Assumes infrastructure solves everything
- No backup plan mentioned

### Strong Answer ✓

> "We have a circuit breaker around ES calls. When ES fails repeatedly, the circuit opens and we fall back to querying PostgreSQL replicas. It's slower - maybe 200-500ms instead of 50ms - but the dashboard remains functional.
>
> We show a banner to users indicating degraded performance and alert the on-call team.
>
> When ES recovers, the circuit closes and we return to normal operation."

**Why it's strong:**
- Specific mechanism (circuit breaker)
- Clear fallback path
- User experience consideration
- Recovery behavior explained

---

## Question 4: "How did you achieve 3x throughput improvement?"

### Weak Answer ❌

> "Elasticsearch is faster so we can handle more queries."

**Why it's weak:**
- Circular reasoning
- No breakdown of contributing factors
- Doesn't explain the measurement

### Strong Answer ✓

> "The 3x improvement came from multiple factors:
>
> 1. **Faster individual queries**: 2s → 50ms means more complete per second
> 2. **Resource isolation**: Reads go to ES, writes stay on PostgreSQL - no contention
> 3. **Parallel shards**: Query runs on all 3 shards simultaneously
> 4. **Connection efficiency**: ES handles more concurrent connections than PostgreSQL
>
> We measured with load tests - 50 concurrent users, PostgreSQL handled 50 qps before timeouts; ES handled 150 qps with room to spare."

**Why it's strong:**
- Multiple specific factors
- Shows understanding of system behavior
- Measurement methodology included

---

## Question 5: "How do you handle partial text matching?"

### Weak Answer ❌

> "We use wildcard queries like `*backend*`."

**Why it's weak:**
- Leading wildcards are slow (full index scan)
- Shows lack of optimization knowledge

### Strong Answer ✓

> "Leading wildcards are problematic because they force full index scans. Instead, we use a multi-field approach:
>
> The projectName field has a keyword type for exact matches, plus a `.search` subfield with a text analyzer. When users type 'backend', we query the text field which uses the inverted index efficiently.
>
> If we needed more aggressive partial matching like 'ack' matching 'backend', we'd consider ngram tokenizers, though they increase index size."

**Why it's strong:**
- Identifies the problem with naive approach
- Explains the actual solution
- Shows knowledge of alternatives

---

## Question 6: "What was the biggest challenge?"

### Weak Answer ❌

> "It was hard to learn Elasticsearch. The documentation is confusing."

**Why it's weak:**
- Focuses on personal struggle, not technical challenge
- Doesn't show problem-solving

### Strong Answer ✓

> "The trickiest part was handling out-of-order events. Initially, our consumer used event payloads directly, but we had race conditions where older events would overwrite newer data.
>
> The solution was to always fetch the current state from PostgreSQL when processing a sync message, rather than trusting the event payload. Combined with version checking, this made the consumer idempotent and order-independent.
>
> I also added comprehensive metrics to track sync lag so we'd catch issues early."

**Why it's strong:**
- Specific technical challenge
- Clear problem description
- Solution explained
- Shows learning and improvement

---

## Question 7: "How would you scale this system?"

### Weak Answer ❌

> "Just add more servers."

**Why it's weak:**
- Vague
- Doesn't address which component or how
- Shows shallow understanding

### Strong Answer ✓

> "It depends on the bottleneck:
>
> **If ES queries slow down**: Add ES nodes - it scales horizontally. Or implement index lifecycle management to move old data to cheaper storage.
>
> **If write volume increases**: The Kafka + consumer pattern handles backpressure well. Add consumer instances if lag grows.
>
> **If data volume grows significantly**: Move to time-based indices (jobs-2024-01) with aliases. Easier to manage lifecycle and archive old months.
>
> At our current scale of 10M documents and 100K writes/day, we have headroom. The architecture supports 10x growth before major changes needed."

**Why it's strong:**
- Considers different bottleneck scenarios
- Specific scaling strategies per component
- Current capacity awareness

---

## Red Flags That Make You Look Junior

| Red Flag | Better Approach |
|----------|-----------------|
| "My manager decided" | "We evaluated options and chose X because..." |
| "It just works" | "The mechanism is..." |
| "We didn't have problems" | "We prevented problems by..." |
| "I don't remember the numbers" | Prepare key metrics before interview |
| "It's industry standard" | Explain why it fits your use case |
| Blaming tools/frameworks | Focus on how you solved challenges |

---

## Patterns That Show SDE-2 Maturity

1. **Quantify**: "10 million rows", "2-3 seconds", "3x improvement"

2. **Explain trade-offs**: "We chose X which gave us Y but cost us Z"

3. **Show failure thinking**: "If X fails, we handle it by..."

4. **Demonstrate ownership**: "I designed...", "I proposed...", "I measured..."

5. **Acknowledge alternatives**: "We also considered Y, but X was better because..."

6. **Connect to user impact**: "This improved the user experience by..."
