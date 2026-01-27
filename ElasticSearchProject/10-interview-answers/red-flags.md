# Red Flags to Avoid

Things that make interviewers doubt your experience or understanding.

---

## Red Flag 1: Vague Numbers

### Bad ❌
> "We improved performance significantly."
> "We have a lot of data."
> "Queries were slow before."

### Good ✓
> "Query latency dropped from 2-3 seconds to 50-80 milliseconds."
> "We had 10 million documents, growing by 100K daily."
> "P99 latency was 3.2 seconds before optimization."

**Why it matters:** Numbers show you measured, not guessed. SDE-2 should be metrics-driven.

---

## Red Flag 2: Claiming You Did Everything Solo

### Bad ❌
> "I built the entire Elasticsearch infrastructure from scratch."
> "I designed and implemented the whole thing alone."

### Good ✓
> "I led the ES integration. I designed the sync architecture and wrote the consumer service. The infra team helped with cluster setup, and I collaborated with the frontend team on the API contract."

**Why it matters:** Sounds unbelievable. Real projects involve collaboration. Claiming solo work makes interviewers suspicious.

---

## Red Flag 3: Not Understanding Trade-offs

### Bad ❌
Interviewer: "What's the downside of your approach?"
> "There aren't really any downsides. Elasticsearch is great."

### Good ✓
> "The main trade-off is eventual consistency. Dashboard data can be 2-5 seconds stale. We also added infrastructure complexity with Kafka and ES. And there's operational overhead in monitoring sync lag."

**Why it matters:** Every technical decision has trade-offs. Not acknowledging them suggests shallow understanding.

---

## Red Flag 4: Blaming Others

### Bad ❌
> "The previous architecture was terrible."
> "The team didn't know what they were doing."
> "My manager made us do it wrong initially."

### Good ✓
> "The original design made sense at smaller scale. As we grew to 10M records, we needed to optimize. I proposed adding ES for read queries."

**Why it matters:** Shows poor professionalism. Interviewers imagine you'll blame their team next.

---

## Red Flag 5: Obvious Resume Exaggeration

### Bad ❌
Resume says "sub-second" but you can't explain how.
> Interviewer: "How did you achieve sub-second latency?"
> You: "Uh... Elasticsearch is fast?"

### Good ✓
> "Sub-second is conservative - actual P50 is 50ms. The inverted index gives O(1) lookups, filter caching avoids repeated work, and our 3-shard setup parallelizes queries."

**Why it matters:** If you can't explain your resume claims, everything else becomes suspect.

---

## Red Flag 6: Not Knowing Basics of Your Stack

### Bad ❌
> Interviewer: "How does an inverted index work?"
> You: "I think it's... inverted from normal indexes?"

### Good ✓
> "Normal indexes map documents to terms - like looking up a word in a book by page. Inverted index maps terms to documents - like a book's index at the back. 'FAILED' → [doc1, doc3, doc7]. This makes term lookups O(1)."

**Why it matters:** If you used ES, you should understand its core mechanism.

---

## Red Flag 7: Refusing to Say "I Don't Know"

### Bad ❌
> Interviewer: "How does ES handle cluster state?"
> You: *makes up something that sounds technical but is wrong*

### Good ✓
> "Honestly, I don't know the details of cluster state management. I know there's a master node for coordination, but I didn't work on cluster administration - our infra team handled that. I focused on the application integration."

**Why it matters:** Making stuff up is worse than admitting gaps. Interviewers catch BS.

---

## Red Flag 8: No Failure Stories

### Bad ❌
> Interviewer: "Did you face any issues in production?"
> You: "No, it all worked smoothly."

### Good ✓
> "Early on, we had a sync lag incident where consumer fell behind by hours. We didn't have good alerting yet. Users saw stale data. After that, we added monitoring for consumer lag and sync latency, with alerts at 5-minute threshold."

**Why it matters:** Perfect projects don't exist. Having no problems suggests either small scope or lack of awareness.

---

## Red Flag 9: Unable to Go Deeper

### Bad ❌
Each follow-up reveals less depth:
> Interviewer: "How did you tune the index?"
> You: "We followed best practices."
> Interviewer: "Which ones specifically?"
> You: "The ones in the documentation."
> Interviewer: "Can you give an example?"
> You: "Uh..."

### Good ✓
> "We tuned several things: Set refresh interval to 5 seconds to reduce segment churn. Used keyword type for filter fields instead of text. Disabled doc_values on fields we never sort. Configured 3 shards based on 10-50GB per shard rule."

**Why it matters:** SDE-2 should have hands-on experience, not just read about it.

---

## Red Flag 10: Dismissing Alternatives

### Bad ❌
> Interviewer: "Why not use PostgreSQL materialized views?"
> You: "That's a bad idea. ES is obviously better."

### Good ✓
> "Materialized views could help for fixed query patterns. The issue is our users filter dynamically - any combination of status, project, date range, architecture. We'd need a view for each combination, which isn't practical. ES handles arbitrary filter combinations efficiently through its inverted index."

**Why it matters:** Dismissing alternatives shows closed-mindedness. Good engineers consider options.

---

## Red Flag 11: Can't Explain to Different Audiences

### Bad ❌
Asked to explain to a non-technical stakeholder:
> "We use Elasticsearch with an inverted index and bitset operations for filter caching, with eventual consistency through a Kafka-based CDC pattern..."

### Good ✓
> "Think of it like a search engine for our job data. Before, finding jobs by status was like searching every page in a 10,000-page book. Now, it's like having an index at the back - you look up 'FAILED' and it tells you exactly which pages to check. Much faster."

**Why it matters:** SDE-2 should communicate with PMs, managers, and other teams - not just engineers.

---

## Self-Check Before Interview

1. **Can I explain every line of my resume?**

2. **Do I have 3-5 specific metrics memorized?**

3. **Can I explain the main trade-offs of my design?**

4. **Do I know one thing that went wrong and how I fixed it?**

5. **Can I explain the core technology (inverted index, etc.)?**

6. **Can I explain this to a non-technical person?**

7. **Do I know what alternatives I considered?**

8. **Can I go 3 levels deep on any technical decision?**

If you answer "no" to any of these, prepare that area before your interview.
