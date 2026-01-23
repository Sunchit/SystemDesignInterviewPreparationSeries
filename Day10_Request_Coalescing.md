# Request Coalescing: When 50,000 Became 1
### Day 10 of 50 - System Design Interview Preparation Series

**By Sunchit Dudeja**

---

## 🎯 Welcome to Day 10!

Yesterday, we learned how Bloom Filters protect databases from cache penetration attacks. Today, we tackle another database killer: **the thundering herd problem** — and how request coalescing saved us from a complete meltdown.

> When 50,000 users ask the same question at the same millisecond, you don't need 50,000 answers. You need ONE answer shared 50,000 times.

---

## 🚨 The Avengers 5 Disaster

*Friday, 8:00:00 PM | Ticket Sale Launch*

BookMyShow opens Avengers 5 tickets. In the first **100 milliseconds**, 50,000 users simultaneously ask:

> "Are front-row seats available for Show #123?"

| Metric | Expected | Actual |
|--------|----------|--------|
| Database Queries | 500/sec | **500,000/sec** |
| Database CPU | 40% | **100%** (crashed) |
| Response Time | 200ms | **∞** (timeout) |
| Tickets Sold | 20 | **0** |
| Customer Complaints | 0 | **50,000** |

**The irony?** We had 20 seats available. But the system couldn't even process a single booking because the database was drowning in identical availability checks.

---

## 🔍 The Problem: Thundering Herd

The **thundering herd** happens when thousands of concurrent requests hit the same resource simultaneously.

### The Vulnerable Implementation

```java
public class VulnerableBookingService {
    
    public SeatAvailability checkAvailability(String showId, String seatType) {
        
        String cacheKey = "availability:" + showId + ":" + seatType;
        
        // Check cache
        SeatAvailability cached = cache.get(cacheKey);
        if (cached != null) {
            return cached;
        }
        
        // Cache miss → EVERY request queries database!
        SeatAvailability availability = database.checkSeatAvailability(showId, seatType);
        
        cache.put(cacheKey, availability, 5); // 5 seconds TTL
        
        return availability;
    }
}
```

### Why This Fails

```
Timeline: 8:00:00.000 PM

User 1:     Cache miss → Query DB ─────────────────────┐
User 2:     Cache miss → Query DB ─────────────────────┤
User 3:     Cache miss → Query DB ─────────────────────┤
...                                                    ├── ALL 50,000 hit DB!
User 49,999: Cache miss → Query DB ────────────────────┤
User 50,000: Cache miss → Query DB ────────────────────┘

Result: Database executes 50,000 IDENTICAL queries
```

| What Went Wrong | Why It Happened |
|-----------------|-----------------|
| No request deduplication | Each concurrent request acts independently |
| Cache not populated yet | First request hasn't returned to populate cache |
| Database overwhelmed | 50,000 queries for the same data |

---

## 🧠 The Solution: Request Coalescing

**Request coalescing** bundles identical concurrent requests into a **single database query**. Like having one representative check seats for everyone asking the same question at the same time.

### The Concept

```
BEFORE COALESCING:
│
├── User 1 ──→ Database Query #1 ──→ Response
├── User 2 ──→ Database Query #2 ──→ Response
├── User 3 ──→ Database Query #3 ──→ Response
│   ... (50,000 queries)
└── User 50,000 ──→ Database Query #50,000 ──→ Response

AFTER COALESCING:
│
├── User 1 ──→ ┐
├── User 2 ──→ ├── Single Database Query ──→ Shared Response ──→ All Users
├── User 3 ──→ ┤
│   ...        │
└── User 50,000 ──→ ┘
```

---

## 🔧 Implementation: The Coalescing Service

```java
public class MovieBookingService {
    
    // Track in-flight requests
    private ConcurrentHashMap<String, CompletableFuture<SeatAvailability>> 
        inFlightRequests = new ConcurrentHashMap<>();
    
    public CompletableFuture<SeatAvailability> checkAvailability(
        String showId, String seatType) {
        
        String cacheKey = "availability:" + showId + ":" + seatType;
        
        // Step 1: Check cache first
        SeatAvailability cached = cache.get(cacheKey);
        if (cached != null) {
            return CompletableFuture.completedFuture(cached);
        }
        
        // Step 2: Coalesce identical requests
        return inFlightRequests.computeIfAbsent(cacheKey, key -> {
            
            // Only ONE request reaches here for this show+seatType
            return CompletableFuture.supplyAsync(() -> {
                try {
                    // Single database query for all waiting users
                    SeatAvailability availability = 
                        database.checkSeatAvailability(showId, seatType);
                    
                    // Cache result for future requests
                    cache.put(cacheKey, availability, 5); // 5 seconds
                    
                    return availability;
                    
                } finally {
                    // Clean up - allow new requests
                    inFlightRequests.remove(cacheKey);
                }
            });
        });
    }
}
```

### How `computeIfAbsent` Creates Magic

```java
// Thread-safe coalescing magic:
inFlightRequests.computeIfAbsent(cacheKey, key -> {
    // This lambda executes ONLY ONCE per unique key
    // All other threads with same key get the existing CompletableFuture
});
```

| Thread | Action | Result |
|--------|--------|--------|
| User 1 | `computeIfAbsent("show123:front")` | Creates new CompletableFuture, starts DB query |
| User 2 | `computeIfAbsent("show123:front")` | Gets User 1's CompletableFuture (no new query) |
| User 3 | `computeIfAbsent("show123:front")` | Gets User 1's CompletableFuture (no new query) |
| User 50,000 | `computeIfAbsent("show123:front")` | Gets User 1's CompletableFuture (no new query) |

---

## 🎥 Visualizing the Flow

```
TIMELINE: Avengers 5 tickets go on sale (8:00:00 PM)

8:00:00.000 - User 1: "Front-row seats for Show #123?"
8:00:00.001 - User 2: "Front-row seats for Show #123?"  
8:00:00.002 - User 3: "Front-row seats for Show #123?"
...
8:00:00.100 - User 50,000: "Front-row seats for Show #123?"

WHAT HAPPENS WITH COALESCING:

┌─────────────────────────────────────────────────────────────┐
│  Step 1: User 1's request creates "in-flight" entry        │
│          CompletableFuture created for "show123:front"     │
├─────────────────────────────────────────────────────────────┤
│  Step 2: Users 2-50,000 find existing in-flight request    │
│          All attach to SAME CompletableFuture              │
├─────────────────────────────────────────────────────────────┤
│  Step 3: ONE database query executes                       │
│          SELECT available_seats FROM shows WHERE id=123    │
├─────────────────────────────────────────────────────────────┤
│  Step 4: Result cached for 5 seconds                       │
│          Future requests get cached response               │
├─────────────────────────────────────────────────────────────┤
│  Step 5: All 50,000 users get response in ~200ms           │
│          Database saved from 49,999 unnecessary queries    │
└─────────────────────────────────────────────────────────────┘
```

---

## ⚡ Impact Metrics

| Metric | Without Coalescing | With Coalescing |
|--------|-------------------|-----------------|
| Database Queries | 50,000 | **1** |
| Database CPU | 100% (crashes) | **5%** (normal) |
| Response Time | 5-30 seconds (timeout) | **200ms** |
| Tickets Sold Fairly | 0 (system crashed) | **20** (fair lottery) |
| User Experience | "Site crashed again" | "Got my ticket!" |
| Infrastructure Cost | ₹5,000/minute (emergency scaling) | **₹50/minute** |

### The Math

```
Before: 50,000 identical queries × 20ms each = 1,000,000ms of DB work
After:  1 query × 20ms = 20ms of DB work

Reduction: 99.998% fewer database queries
```

---

## 🔄 Production-Ready Implementation

### Enhanced Version with Timeout & Error Handling

```java
public class ProductionBookingService {
    
    private final ConcurrentHashMap<String, CompletableFuture<SeatAvailability>> 
        inFlightRequests = new ConcurrentHashMap<>();
    
    private final Cache cache;
    private final Database database;
    private final ExecutorService executor;
    private final MeterRegistry metrics;
    
    public CompletableFuture<SeatAvailability> checkAvailability(
            String showId, String seatType) {
        
        String cacheKey = "availability:" + showId + ":" + seatType;
        
        // Step 1: Cache check (fast path)
        SeatAvailability cached = cache.get(cacheKey);
        if (cached != null) {
            metrics.counter("availability.cache.hit").increment();
            return CompletableFuture.completedFuture(cached);
        }
        
        metrics.counter("availability.cache.miss").increment();
        
        // Step 2: Coalesce with timeout protection
        return inFlightRequests.computeIfAbsent(cacheKey, key -> 
            createDatabaseFuture(showId, seatType, cacheKey)
        ).orTimeout(5, TimeUnit.SECONDS)
         .exceptionally(ex -> handleFailure(ex, showId, seatType));
    }
    
    private CompletableFuture<SeatAvailability> createDatabaseFuture(
            String showId, String seatType, String cacheKey) {
        
        metrics.counter("availability.coalesced.leader").increment();
        
        return CompletableFuture.supplyAsync(() -> {
            try {
                // Track in-flight request count
                metrics.gauge("availability.inflight", 
                    inFlightRequests.size());
                
                SeatAvailability availability = 
                    database.checkSeatAvailability(showId, seatType);
                
                // Cache with jittered TTL to prevent cache stampede
                int jitteredTTL = 5 + ThreadLocalRandom.current().nextInt(3);
                cache.put(cacheKey, availability, jitteredTTL);
                
                return availability;
                
            } finally {
                // Always clean up to allow new requests
                inFlightRequests.remove(cacheKey);
            }
        }, executor);
    }
    
    private SeatAvailability handleFailure(Throwable ex, 
            String showId, String seatType) {
        
        metrics.counter("availability.error").increment();
        log.error("Failed to check availability for {}/{}", showId, seatType, ex);
        
        // Return degraded response instead of failing
        return SeatAvailability.unknown();
    }
}
```

### Key Production Considerations

| Feature | Purpose |
|---------|---------|
| **Timeout Protection** | Prevent indefinite waiting if DB is slow |
| **Jittered TTL** | Prevent synchronized cache expiration |
| **Metrics** | Track coalescing effectiveness |
| **Graceful Degradation** | Return "unknown" instead of crashing |
| **Cleanup in finally** | Always remove in-flight entry |

---

## 🎯 When to Use Request Coalescing

### ✅ Perfect Use Cases

| Scenario | Why Coalescing Helps |
|----------|---------------------|
| **Ticket/Flash Sales** | Thousands check same event simultaneously |
| **Product Launches** | iPhone release, PS5 stock checks |
| **Breaking News** | Everyone queries same article |
| **Stock Prices** | Real-time quotes for popular stocks |
| **Rate Limit Status** | Multiple requests check same user's limits |
| **Feature Flags** | Concurrent requests for same flag |

### ❌ When NOT to Use

| Scenario | Why Not |
|----------|---------|
| **User-specific data** | Each user needs different data |
| **Low concurrency** | Overhead not worth it for few requests |
| **Rapidly changing data** | Coalesced response may be stale |
| **Write operations** | Each write must be unique |

---

## 🔗 Alternative Patterns

### Pattern 1: Singleflight (Go)

```go
import "golang.org/x/sync/singleflight"

var group singleflight.Group

func checkAvailability(showID string) (*SeatAvailability, error) {
    key := "availability:" + showID
    
    result, err, shared := group.Do(key, func() (interface{}, error) {
        // Only ONE goroutine executes this
        return database.CheckSeatAvailability(showID)
    })
    
    if shared {
        metrics.Increment("coalesced_request")
    }
    
    return result.(*SeatAvailability), err
}
```

### Pattern 2: DataLoader (GraphQL/Node.js)

```javascript
const DataLoader = require('dataloader');

const availabilityLoader = new DataLoader(async (showIds) => {
    // Batch all requested showIds into single query
    const results = await database.checkAvailabilityBatch(showIds);
    
    // Return in same order as requested
    return showIds.map(id => results[id]);
});

// Usage - automatically batched & coalesced
const availability = await availabilityLoader.load("show123");
```

### Pattern 3: Redis-Based Distributed Coalescing

```java
public class DistributedCoalescing {
    
    public SeatAvailability checkAvailability(String showId) {
        String lockKey = "lock:availability:" + showId;
        String cacheKey = "availability:" + showId;
        
        // Try to get cached result
        SeatAvailability cached = cache.get(cacheKey);
        if (cached != null) return cached;
        
        // Try to become the leader
        boolean isLeader = redis.setNx(lockKey, "1", Duration.ofSeconds(10));
        
        if (isLeader) {
            // I'm the leader - query database
            SeatAvailability result = database.checkSeatAvailability(showId);
            cache.put(cacheKey, result, 5);
            redis.del(lockKey);
            return result;
        } else {
            // Someone else is querying - wait for result
            return waitForCachePopulation(cacheKey, Duration.ofSeconds(5));
        }
    }
}
```

---

## 📊 Monitoring Your Coalescing

### Essential Metrics

```java
// Track these metrics:
metrics.counter("coalescing.leader")     // First request (does DB work)
metrics.counter("coalescing.follower")   // Subsequent requests (waits)
metrics.gauge("coalescing.inflight")     // Current in-flight requests
metrics.timer("coalescing.wait_time")    // How long followers wait
metrics.counter("coalescing.timeout")    // Requests that timed out
```

### Dashboard Alerts

| Metric | Healthy | Warning | Critical |
|--------|---------|---------|----------|
| Leader:Follower Ratio | 1:100+ | 1:10 | 1:1 (not coalescing!) |
| Avg Wait Time | <200ms | 500ms | >1000ms |
| Timeout Rate | 0% | 1% | >5% |
| In-flight Count | <100 | 500 | >1000 |

---

## 🔄 Combining with Other Patterns

### The Ultimate Defense Stack

```
┌─────────────────────────────────────────────────────┐
│  Layer 1: Rate Limiting                             │
│  └── Reject abusive clients early                   │
├─────────────────────────────────────────────────────┤
│  Layer 2: Bloom Filter (Day 9)                      │
│  └── Reject invalid requests before processing     │
├─────────────────────────────────────────────────────┤
│  Layer 3: Cache                                     │
│  └── Return cached results for repeat requests     │
├─────────────────────────────────────────────────────┤
│  Layer 4: Request Coalescing (Today)                │
│  └── Deduplicate concurrent cache misses           │
├─────────────────────────────────────────────────────┤
│  Layer 5: Database                                  │
│  └── Handle minimal, deduplicated queries          │
└─────────────────────────────────────────────────────┘
```

---

## ❓ Interview Practice

### Question 1:
> "How would you handle 50,000 concurrent requests for the same data?"

**Answer:**
> "I'd implement request coalescing using a ConcurrentHashMap of in-flight CompletableFutures. When the first request arrives, it creates an entry and starts the database query. Subsequent identical requests find this entry and attach to the same CompletableFuture. When the query completes, all 50,000 requests get the result simultaneously. This reduces database load from 50,000 queries to just 1, while maintaining sub-second response times for all users."

### Question 2:
> "What's the difference between caching and request coalescing?"

**Answer:**
> "Caching stores results AFTER a query completes and serves them to FUTURE requests. Request coalescing handles CONCURRENT requests that arrive BEFORE the first query completes. They're complementary: imagine a ticket sale where cache is empty. Without coalescing, 50,000 concurrent cache misses all hit the database. With coalescing, only 1 query executes, the result is cached, and all subsequent requests get the cached value. Coalescing protects the window between cache miss and cache population."

### Question 3:
> "What are the edge cases to handle?"

**Answer:**
> "Four key edge cases: (1) Timeout handling — set reasonable timeouts so users don't wait forever if the leader request hangs. (2) Error propagation — if the leader request fails, all followers should get the error, not hang indefinitely. (3) Cleanup — always remove the in-flight entry in a finally block to prevent memory leaks. (4) Distributed systems — for multi-server deployments, use Redis-based locking instead of in-memory ConcurrentHashMap."

---

## 🔗 Connecting to Previous Days

| Day | Concept | How It Connects |
|-----|---------|-----------------|
| Day 6 | Design for Failure | Coalescing prevents cascading database failure |
| Day 7 | Databases | Protects your most expensive resource |
| Day 8 | Load Balancing | Load balancing can't help if every request hits DB |
| Day 9 | Bloom Filters | Bloom filter + Coalescing = Ultimate protection |

---

## ✅ Day 10 Action Items

1. **Identify thundering herd risks** — Which endpoints have high concurrent traffic?
2. **Implement basic coalescing** — Use ConcurrentHashMap + CompletableFuture
3. **Add timeout handling** — Don't let requests wait forever
4. **Measure effectiveness** — Track leader vs follower ratio
5. **Combine with caching** — Coalescing fills the gap until cache is populated

---

## 💡 Lessons Learned

| Lesson | Why It Matters |
|--------|----------------|
| Cache doesn't prevent thundering herd | Concurrent cache misses all hit DB |
| First request becomes "leader" | Others wait for leader's result |
| Always clean up in-flight entries | Prevent memory leaks and deadlocks |
| Combine patterns for defense-in-depth | Rate limit → Bloom filter → Cache → Coalescing → DB |
| Monitor the coalescing ratio | If ratio is 1:1, coalescing isn't working |

---

## 🚀 Key Architect Principles

| Principle | What It Means |
|-----------|---------------|
| **Deduplicate at the edge** | Reduce identical work before it reaches expensive resources |
| **Embrace "good enough"** | Slightly stale data (5s cache) is better than crashed system |
| **Leader-follower pattern** | One does the work, many benefit from it |
| **Timeout everything** | Never wait forever, always have a backup plan |
| **Layer your defenses** | No single pattern handles all attack vectors |

---

## 💡 Key Takeaway

> **Developer: "Just add more database replicas to handle the load."**  
> **Architect: "Why execute 50,000 identical queries when ONE query can serve everyone? Request coalescing turns a thundering herd into a gentle breeze."**

The difference? Understanding that **not every request deserves its own database query**. Smart deduplication is the architect's answer to the thundering herd.

---

*— Sunchit Dudeja*  
*Day 10 of 50: System Design Interview Preparation Series*

---

> 🎯 **Interview Edge:** When discussing high-concurrency scenarios, mention: "I'd implement request coalescing to deduplicate concurrent requests. This ensures that even during traffic spikes, the database sees a steady, manageable load." This shows you understand patterns beyond basic caching.

> 📢 **Real Impact:** Request coalescing at BookMyShow reduced database queries during ticket launches from 500,000 QPS to under 5,000 QPS — a **99% reduction** — while improving response times from 5+ seconds to under 200ms.

