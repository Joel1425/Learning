# Session Q&A Notes - System Design Interview Prep

> Quick reference notes from study sessions for SDE-2 Backend interviews.

---

## Q1: How does PostGIS work? How is lat/long indexed? Does it use GeoHash?

**Answer:** PostGIS uses **R-Tree indexes (via GiST)**, NOT GeoHash by default.

**R-Tree vs GeoHash:**

| Aspect | R-Tree (PostGIS) | GeoHash |
|--------|------------------|---------|
| Structure | Hierarchical bounding boxes | String encoding ("tdr1y") |
| Nearest neighbor | Excellent (native) | Needs workarounds |
| Use case | Complex spatial queries | Redis, DynamoDB, simple proximity |

**Example:**
```sql
CREATE INDEX idx_location ON restaurants USING GIST(location);

SELECT name FROM restaurants
WHERE ST_DWithin(location, ST_MakePoint(77.59, 12.97)::geography, 5000);
```

**Interview one-liner:** *"PostGIS uses R-Tree indexes via GiST, organizing points in hierarchical bounding boxes. R-Tree excels at complex spatial queries and nearest-neighbor, while GeoHash is simpler and preferred for key-value stores like Redis."*

---

## Q2: How does R-Tree work? How does it handle points in different boxes but close to user?

**Core Concept:** R-Tree groups nearby points into bounding boxes, then groups boxes into larger boxes (hierarchical).

**The Edge Case Problem:**
```
┌─────────────────┬─────────────────┐
│     Box A       │      Box B      │
│                 │  ☕ Coffee Shop │
│         📍 USER │  (in Box B but  │
│    (in Box A)   │   CLOSER!)      │
└─────────────────┴─────────────────┘
```

**Solution:** R-Tree searches ALL boxes that OVERLAP with search radius:

```
┌─────────────────┬─────────────────┐
│     Box A       │      Box B      │
│            ┌────┼────┐            │
│         📍─│─search──│ ☕ ← FOUND!│
│            │ circle  │            │
│            └────┼────┘            │
└─────────────────┴─────────────────┘
Search circle overlaps BOTH boxes → checks BOTH → finds ☕
```

**Recursion:**
```python
def search(node, user, radius):
    if node.is_leaf:
        return [p for p in node.points if distance(p, user) <= radius]
    else:
        results = []
        for child_box in node.children:
            if circle_overlaps_box(user, radius, child_box):
                results += search(child_box, user, radius)
        return results
```

**Interview one-liner:** *"R-Tree searches all bounding boxes overlapping the query circle, not just the user's box. This handles edge cases where nearby points are in adjacent boxes. Time complexity is O(log n) average."*

---

## Q3: SQL vs Graph DB for Many-to-Many (Friends of Friends)

**SQL Query for Friends-of-Friends:**
```sql
SELECT * FROM users u1
JOIN friendships f1 ON u1.id = f1.user_id      -- Alice's friends
JOIN friendships f2 ON f1.friend_id = f2.user_id  -- Their friends
JOIN users u2 ON f2.friend_id = u2.id
WHERE u1.id = 123 AND u2.id != 123;
```

**Why SQL is Inefficient:**
```
Depth 1: 1 JOIN  → O(n)
Depth 2: 2 JOINs → O(n²)
Depth 6: 6 JOINs → O(n⁶) 💀
```

**Graph DB uses Index-Free Adjacency:**
```
SQL: Must scan/index junction table for each hop
Graph: Direct pointers stored in node → just follow pointer!
```

**Performance:**

| Depth | SQL (1M users) | Graph DB |
|-------|----------------|----------|
| 2 hops | ~100ms | ~2ms |
| 6 hops | Timeout | ~50ms |

**Interview one-liner:** *"Graph DBs use index-free adjacency - nodes store direct pointers to neighbors. Traversal is O(relationships traversed) not O(data size), making multi-hop queries constant time."*

---

## Q4: Async Replication - Who sends success? What about multiple replicas?

**Key Point:** PRIMARY sends success, NOT replicas.

```
Client ──→ PRIMARY ──→ Client (SUCCESS immediately!)
                │
                └──→ Replicas (background, async)
```

**Multiple Replicas:**
- Primary streams WAL to ALL replicas in parallel
- Each replica may have different lag
- Client doesn't wait for any replica

**Sync vs Async vs Semi-Sync:**

| Mode | Wait For | Latency | Consistency |
|------|----------|---------|-------------|
| Async | Primary only | Low | Eventual |
| Semi-Sync | Primary + 1 replica | Medium | Better |
| Sync | Primary + ALL replicas | High | Strong |

**Interview one-liner:** *"In async replication, PRIMARY sends success immediately after its own disk write. Replicas receive changes in background. For durability, use semi-sync (wait for 1 replica) or sync (wait for all)."*

---

## Q5: How is WAL passed to replicas? By queues?

**Answer:** NOT queues - Direct TCP Streaming.

```
PRIMARY                          REPLICA
┌─────────────────┐              ┌─────────────────┐
│   WAL Writer    │              │   WAL Receiver  │
│        ↓        │    TCP       │        ↓        │
│   WAL Sender    │═════════════►│   Startup       │
│   (process)     │   stream     │   Process       │
└─────────────────┘              └─────────────────┘
```

**Why NOT Message Queues?**
1. **Ordering:** WAL must be applied in EXACT order - TCP guarantees this
2. **Latency:** Direct TCP ~1ms, Queue ~10-50ms
3. **Position tracking:** Replica tracks exact LSN, can resume after crash
4. **Backpressure:** TCP naturally slows if replica is slow

**Interview one-liner:** *"WAL is streamed via persistent TCP connections, not queues. Direct TCP ensures strict ordering, low latency, and exact position tracking - things message queues can't guarantee."*

---

## Q6: What happens when replica receives WAL? What if lost?

**Processing Steps:**
```
1. WAL Receiver gets data over TCP
2. Writes to local pg_wal/ (BEFORE applying!)
3. Sends ACK back to Primary
4. Startup Process replays WAL → updates data files
```

**Fault Tolerance:**

| Failure | Protected By |
|---------|--------------|
| Network packet loss | TCP retransmit |
| Connection drop | Resume from last LSN |
| Replica crash | WAL replay from checkpoint |
| Primary deletes WAL early | Replication slots |

**Replication Slot:** Primary tracks replica position, won't delete WAL that replica still needs.

**Interview one-liner:** *"Replica first writes WAL to disk, then applies. On crash, replays from checkpoint. LSN tracking ensures Primary resends lost records. Replication slots prevent Primary from deleting needed WAL."*

---

## Q7: What is Network Partition?

**Definition:** Nodes can't talk to each other, but are still running.

```
Normal:     Node A ◄──────► Node B ◄──────► Node C

Partition:  Node A ◄──────► Node B    ✗    Node C
                                   (link broken, nodes alive!)
```

**The Problem - Split Brain:**
```
Both partitions think they're the leader → accept writes → data diverges!
```

**CAP Theorem Choice During Partition:**
- **CP (Consistency):** Minority partition rejects writes
- **AP (Availability):** Both accept writes, resolve conflicts later

**Solutions:**
1. **Quorum:** Need majority (3/5 nodes) to proceed
2. **Fencing:** Kill old leader (STONITH)
3. **Leases:** Leader must renew, stops itself if can't

**Interview one-liner:** *"Network partition is when nodes are running but can't communicate. Forces choice: consistency (reject writes on minority) or availability (accept everywhere, resolve later). Quorum prevents split-brain."*

---

## Q8: What is SAGA Pattern?

**Problem:** Can't do single transaction across microservices (each has own DB).

**Solution:** SAGA = Sequence of local transactions + compensating actions.

```
SAGA Steps:
T1: Wallet → Deduct ₹500
T2: Order  → Create order
T3: Restaurant → Accept order

If T3 fails:
C2: Order → Cancel order
C1: Wallet → Refund ₹500
```

**Two Types:**

| Type | How it Works | Best For |
|------|--------------|----------|
| Choreography | Services react to events (no coordinator) | Simple flows (2-3 steps) |
| Orchestration | Central coordinator controls flow | Complex flows (4+ steps) |

**Orchestration Example:**
```java
try {
    walletService.deduct(amount);     // T1
    orderService.create(order);        // T2
    restaurantService.accept(orderId); // T3
} catch (Exception e) {
    orderService.cancel(orderId);      // C2
    walletService.refund(amount);      // C1
}
```

**Key Considerations:**
- Each step must be **idempotent** (safe to retry)
- Compensation ≠ Rollback (can't unsend email)
- **Eventual consistency** during execution

**Interview one-liner:** *"SAGA handles distributed transactions by executing local transactions sequentially. If any fails, compensating transactions run in reverse. Provides eventual consistency without distributed locks."*

---

## Q9: Amazon Locker LLD

### Core Entities

```
┌────────────────────────────────────────────────────────────┐
│  Locker       │  Orchestrator. Owns compartments & tokens. │
│  AccessToken  │  Bearer token with 7-day TTL.              │
│  Compartment  │  Physical slot with size (S/M/L).          │
└────────────────────────────────────────────────────────────┘

Why NOT Package entity? → Only need size (input param), no behavior.
```

### Class Design

```java
class Locker {
    - compartments: List<Compartment>
    - accessTokenMapping: Map<String, AccessToken>

    + depositPackage(Size): String   // returns token
    + pickup(tokenCode): void
    + openExpiredCompartments(): void
}

class AccessToken {
    - code, expiration, compartment
    + isExpired(): boolean
}

class Compartment {
    - size, occupied
    + markOccupied(), markFree(), open()
}
```

### Key Methods

**depositPackage:**
```java
Compartment c = getAvailableCompartment(size);
c.open();
c.markOccupied();
AccessToken token = generateToken(c);  // 7-day expiry
accessTokenMapping.put(token.getCode(), token);
return token.getCode();
```

**pickup:**
```java
AccessToken token = accessTokenMapping.get(code);
if (token == null) throw "Invalid code";
if (token.isExpired()) throw "Expired";
token.getCompartment().open();
clearDeposit(token);  // markFree + remove from map
```

### Common Extensions
1. **Size fallback:** Try SMALL → MEDIUM → LARGE
2. **Maintenance status:** Add OUT_OF_SERVICE state
3. **Two-phase deposit:** reserve() → confirm()

**Interview one-liner:** *"3 entities: Locker (orchestrator), AccessToken (TTL bearer), Compartment (physical). Compartment owns physical state, Locker owns token mapping. Deposit: open + generate token. Pickup: validate + open + cleanup."*

---

*Keep adding notes from each study session here!*
