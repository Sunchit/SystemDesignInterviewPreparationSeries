# Redis Sorted Sets: Building Real-Time Leaderboards at Scale
### Day 16 of 50 - System Design Interview Preparation Series

**By Sunchit Dudeja**

---

## 🎯 Welcome to Day 16!

Yesterday, we explored why Redis's single-threaded architecture makes it blazingly fast. Today, we dive deep into one of Redis's most powerful data structures — **Sorted Sets** — and how companies like Dream11, Riot Games, and Discord use them to build real-time leaderboards serving millions of users.

> The secret to instant leaderboards: **Sort at INSERT time, not QUERY time.**

---

## 🏆 THE TOURNAMENT SCENARIO

### Dream11 IPL Final

```
┌─────────────────────────────────────────────────────────────────┐
│                    THE CHALLENGE                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   • 10 million users playing fantasy cricket                    │
│   • Every ball bowled updates 2 million users' scores           │
│   • Leaderboard must update within 100ms of every event         │
│   • Users refresh leaderboard every 30 seconds                  │
│   • Ball is bowled every 30 seconds = continuous updates        │
│                                                                  │
│   Requirements:                                                  │
│   ├── Update 2M scores in < 200ms                               │
│   ├── Query top 100 in < 10ms                                   │
│   ├── Get user's rank in < 10ms                                 │
│   └── Handle 500K concurrent leaderboard requests               │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## ❌ The Traditional Database Approach (Disaster)

### What Most Developers Try First

```sql
-- After every ball (every 30 seconds):
UPDATE user_scores SET score = score + 10 WHERE user_id = 123;
-- Repeat for 2 million users...
-- Takes 60+ seconds!

-- To show leaderboard:
SELECT user_id, score FROM user_scores 
ORDER BY score DESC LIMIT 100;
-- Full table scan on 10M rows
-- Takes 10+ seconds!
-- By the time it loads, next ball is already bowled!
```

### Why This Fails

```
┌─────────────────────────────────────────────────────────────────┐
│              SQL LEADERBOARD QUERY FLOW                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   1. SELECT * FROM user_scores                                   │
│      └── Fetch 10,000,000 rows from disk                        │
│      └── Time: 2-5 seconds                                       │
│                                                                  │
│   2. ORDER BY score DESC                                         │
│      └── Sort 10,000,000 rows in memory                         │
│      └── O(N log N) = 230 million comparisons!                  │
│      └── Time: 5-10 seconds                                      │
│                                                                  │
│   3. LIMIT 100                                                   │
│      └── Return first 100 rows                                   │
│      └── Time: negligible                                        │
│                                                                  │
│   TOTAL: 10-15 seconds PER QUERY                                │
│   At 500K queries/sec = IMPOSSIBLE                               │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**The fundamental problem:** SQL databases sort data at **QUERY time**. Every time someone views the leaderboard, the database must re-sort all 10 million rows.

---

## ✅ Redis Sorted Sets: The Magic Solution

### The Key Insight

> **What if we sorted data at INSERT time instead of QUERY time?**

Redis Sorted Sets maintain data in sorted order **automatically**. When you query the top 100, the data is **already sorted** — no computation needed!

```bash
# Update score (auto-reorders in sorted position)
ZINCRBY leaderboard:ipl2024 25 "user:rajesh"

# Get top 100 (just walk the already-sorted list)
ZREVRANGE leaderboard:ipl2024 0 99 WITHSCORES
```

---

## ⚙️ HOW REDIS SORTED SETS WORK INTERNALLY

### The Dual Data Structure Architecture

Redis Sorted Sets use **TWO data structures** working together:

```
┌─────────────────────────────────────────────────────────────────┐
│           SORTED SET = HASH TABLE + SKIP LIST                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   ┌─────────────────────┐      ┌─────────────────────────────┐  │
│   │    HASH TABLE       │      │       SKIP LIST             │  │
│   │    (for O(1)        │ ptr  │       (for O(log N)         │  │
│   │     lookups)        │ ───→ │        ordering)            │  │
│   ├─────────────────────┤      ├─────────────────────────────┤  │
│   │ "user:raj" → node   │      │ Sorted by score (desc)      │  │
│   │ "user:amit" → node  │      │ [15500]→[15200]→[14800]→... │  │
│   │ "user:sam" → node   │      │                             │  │
│   └─────────────────────┘      └─────────────────────────────┘  │
│                                                                  │
│   Hash Table answers: "Does this user exist? What's their node?"│
│   Skip List answers: "What's the sorted order? Who's in top N?" │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 📊 STEP 1: THE HASH TABLE (O(1) Member Lookup)

### What It Stores

The Hash Table maps each **member** (user ID) to a **pointer** to its node in the Skip List.

```
┌─────────────────────────────────────────────────────────────────┐
│                    HASH TABLE STRUCTURE                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Key (Member)        │  Value (Pointer to SkipList Node)       │
│   ────────────────────┼─────────────────────────────────────────│
│   "user:rajesh_kumar" │  → SkipListNode { score: 15500, ... }   │
│   "user:priya_sharma" │  → SkipListNode { score: 14500, ... }   │
│   "user:amit_patel"   │  → SkipListNode { score: 14200, ... }   │
│   "user:sam_wilson"   │  → SkipListNode { score: 14800, ... }   │
│   ...                 │  ...                                     │
│   (10 million entries)│                                          │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Why It's Critical

```
WITHOUT Hash Table:
┌─────────────────────────────────────────────────────────────────┐
│ ZINCRBY leaderboard 25 "user:amit"                              │
│                                                                  │
│ Step 1: Find "user:amit" in Skip List                           │
│         → Must traverse from head                                │
│         → O(log N) = 24 operations for 10M entries              │
│                                                                  │
│ Step 2: Update score and reposition                              │
│         → O(log N) = 24 more operations                         │
│                                                                  │
│ TOTAL: O(2 × log N) = 48 operations                             │
└─────────────────────────────────────────────────────────────────┘

WITH Hash Table:
┌─────────────────────────────────────────────────────────────────┐
│ ZINCRBY leaderboard 25 "user:amit"                              │
│                                                                  │
│ Step 1: Hash["user:amit"] → direct pointer!                     │
│         → O(1) = 1 operation                                    │
│                                                                  │
│ Step 2: Update score and reposition                              │
│         → O(log N) = 24 operations                              │
│                                                                  │
│ TOTAL: O(1) + O(log N) = 25 operations (2x faster!)             │
└─────────────────────────────────────────────────────────────────┘
```

### Hash Table Operations

| Command | How Hash Table Helps | Complexity |
|---------|---------------------|------------|
| `ZSCORE key member` | Direct lookup of member's score | **O(1)** |
| `ZINCRBY key incr member` | Find node instantly, then reposition | O(1) + O(log N) |
| `ZRANK key member` | Find node, then count position | O(1) + O(log N) |
| `ZREM key member` | Find and remove node | O(1) + O(log N) |

---

## 📊 STEP 2: THE SKIP LIST (O(log N) Sorted Order)

### What Is a Skip List?

A Skip List is a **probabilistic data structure** that maintains sorted order with O(log N) insert, delete, and search operations. Think of it as a "linked list with express lanes."

```
┌─────────────────────────────────────────────────────────────────┐
│                    SKIP LIST VISUALIZATION                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Level 4: [HEAD]─────────────────────────────────→[raj:15500]──→[TAIL]
│               │                                          │
│   Level 3: [HEAD]───────────→[sam:14800]───────────→[raj:15500]──→[TAIL]
│               │                    │                     │
│   Level 2: [HEAD]──→[priya:14500]─→[sam:14800]──→[amit:15200]→[raj:15500]→[TAIL]
│               │          │             │              │           │
│   Level 1: [HEAD]→[priya]→[sam]→[amit]→[raj]→ ... (all nodes) →[TAIL]
│            14500   14800  15200  15500
│                                                                  │
│   Reading left to right = ASCENDING order by score              │
│   Reading right to left = DESCENDING order (leaderboard!)       │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### The "Express Lane" Analogy

```
Imagine a highway system:

Level 1 (Local Road):    Every exit, every intersection
                         Stop at: Exit 1, 2, 3, 4, 5, 6, 7, 8...

Level 2 (Highway):       Skip some exits
                         Stop at: Exit 1, 4, 7, 10...

Level 3 (Expressway):    Skip even more
                         Stop at: Exit 1, 10, 20...

Level 4 (Superhighway):  Skip most
                         Stop at: Exit 1, 50, 100...

To find Exit 73:
1. Take Superhighway → Stop at Exit 50
2. Drop to Expressway → Go to Exit 70
3. Drop to Highway → Go to Exit 73
4. Done! Only 3-4 hops instead of 73!

This is exactly how Skip List finds nodes in O(log N)!
```

### Why O(log N)?

```
┌─────────────────────────────────────────────────────────────────┐
│              THE MATH BEHIND O(log N)                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Total entries: N = 10,000,000                                  │
│                                                                  │
│   Each level skips ~2x as many nodes:                            │
│   Level 1: Every node        (10,000,000 nodes)                  │
│   Level 2: Every 2nd node    (5,000,000 nodes)                   │
│   Level 3: Every 4th node    (2,500,000 nodes)                   │
│   Level 4: Every 8th node    (1,250,000 nodes)                   │
│   ...                                                            │
│   Level 24: Every 2^23 node  (~1 node)                           │
│                                                                  │
│   Max levels needed: log₂(10,000,000) = 23.25 ≈ 24 levels       │
│   Max hops to find any node: 24 hops                             │
│                                                                  │
│   Compare to linked list: 10,000,000 hops (worst case)          │
│   Skip List is 400,000x fewer operations!                        │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Skip List Operations

| Command | How Skip List Helps | Complexity |
|---------|---------------------|------------|
| `ZREVRANGE key 0 99` | Walk 100 nodes from tail (sorted!) | **O(log N + 100)** |
| `ZREVRANK key member` | Count nodes from tail to member | **O(log N)** |
| `ZRANGEBYSCORE key min max` | Find range, walk nodes | **O(log N + M)** |
| `ZINCRBY key incr member` | Remove from old position, insert at new | **O(log N)** |

---

## 📊 STEP 3: THE COMBINED OPERATION (ZINCRBY)

### What Happens When a Score Updates

Let's trace exactly what happens when Kohli hits a six and we need to update 800K users' scores:

```
ZINCRBY leaderboard:ipl2024 25 "user:amit"
```

```
┌─────────────────────────────────────────────────────────────────┐
│           ZINCRBY INTERNAL OPERATION (3 SUB-STEPS)               │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   BEFORE: amit has 15200 points (rank #2)                       │
│                                                                  │
│   Skip List: [raj:15500] → [amit:15200] → [sam:14800] → ...     │
│                  #1            #2            #3                  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

#### Sub-Step 3.1: Hash Table Lookup (O(1))

```
┌─────────────────────────────────────────────────────────────────┐
│   STEP 3.1: FIND THE NODE                                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Hash["user:amit"] → pointer to SkipList node                   │
│                                                                  │
│   Result: Direct reference to amit's node                        │
│   Time: O(1) = ~1 microsecond                                    │
│                                                                  │
│   ✅ No traversal needed!                                        │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

#### Sub-Step 3.2: Update Score (O(1))

```
┌─────────────────────────────────────────────────────────────────┐
│   STEP 3.2: UPDATE THE SCORE                                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   node.score = 15200 + 25 = 15225                                │
│                                                                  │
│   Just arithmetic on the node we already have!                   │
│   Time: O(1) = ~0.1 microseconds                                 │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

#### Sub-Step 3.3: Reposition in Skip List (O(log N))

```
┌─────────────────────────────────────────────────────────────────┐
│   STEP 3.3: REPOSITION NODE IN SORTED ORDER                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Old position: After raj:15500 (score was 15200)                │
│   New score: 15225                                               │
│   New position: Still after raj:15500 (15225 < 15500)            │
│                                                                  │
│   But wait! What if new score was 15600?                         │
│   Then amit moves BEFORE raj!                                    │
│                                                                  │
│   Algorithm:                                                     │
│   1. Remove amit from current position (update pointers)         │
│   2. Find correct new position using skip list traversal         │
│   3. Insert amit at new position (update pointers)               │
│                                                                  │
│   Time: O(log N) = ~24 operations for 10M entries                │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Visual: Before and After ZINCRBY

```
BEFORE: amit score = 15200, rank = #2
┌─────────────────────────────────────────────────────────────────┐
│                                                                  │
│   Skip List (sorted by score, descending):                       │
│                                                                  │
│   [raj:15500] → [amit:15200] → [sam:14800] → [priya:14500]      │
│       #1            #2            #3            #4               │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘

ZINCRBY leaderboard 325 "user:amit"  (amit scores big!)

AFTER: amit score = 15525, rank = #1
┌─────────────────────────────────────────────────────────────────┐
│                                                                  │
│   Skip List (auto-reordered!):                                   │
│                                                                  │
│   [amit:15525] → [raj:15500] → [sam:14800] → [priya:14500]      │
│       #1            #2            #3            #4               │
│                                                                  │
│   amit moved from #2 to #1 automatically!                        │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 🔄 COMPLETE IMPLEMENTATION WALKTHROUGH

### Step 1: Tournament Setup

```bash
# Create leaderboard with initial scores
ZADD leaderboard:ipl2024:match1 15000 "user:rajesh_kumar"
ZADD leaderboard:ipl2024:match1 14500 "user:priya_sharma"
ZADD leaderboard:ipl2024:match1 14200 "user:amit_patel"
# ... 10 million more entries

# Memory usage: 10M entries × ~50 bytes = 500MB
# Fits entirely in RAM!
```

### Step 2: Real-time Score Updates (Every Ball)

```python
def update_scores_after_ball(ball_event):
    """
    Called every 30 seconds when a ball is bowled.
    Updates scores for all users who have that player in their team.
    """
    
    # Event: Kohli hits six → +25 points for users who picked him
    impacted_users = get_users_with_player("kohli")  # 800K users
    
    # Use PIPELINE to batch all commands in ONE network call
    pipe = redis.pipeline()
    
    for user_id in impacted_users:
        # ZINCRBY: O(log N) per operation
        # But pipeline sends all at once, reducing network overhead
        pipe.zincrby("leaderboard:ipl2024:match1", 25, user_id)
    
    # Execute all 800K updates in one network round-trip!
    pipe.execute()
    
    # Time taken: ~200ms for 800K updates
    # Compare to SQL: 60+ seconds!
```

### Step 3: Users Viewing Leaderboard

```python
@app.get("/leaderboard")
def get_leaderboard(user_id: str, page: int = 1):
    """
    Returns leaderboard page + user's own rank and score.
    Handles 500K requests/second.
    """
    page_size = 50
    start = (page - 1) * page_size
    end = start + page_size - 1
    
    # STEP 1: Get page of leaders
    # O(log N + 50) = 74 operations for 10M entries
    leaders = redis.zrevrange(
        "leaderboard:ipl2024:match1", 
        start, 
        end, 
        withscores=True
    )
    
    # STEP 2: Get user's own rank
    # O(log N) = 24 operations
    user_rank = redis.zrevrank(
        "leaderboard:ipl2024:match1", 
        f"user:{user_id}"
    )
    
    # STEP 3: Get user's score
    # O(1) = 1 operation (Hash Table lookup!)
    user_score = redis.zscore(
        "leaderboard:ipl2024:match1", 
        f"user:{user_id}"
    )
    
    return {
        "leaders": leaders,
        "user_rank": user_rank + 1 if user_rank is not None else None,
        "user_score": user_score,
        "total_players": redis.zcard("leaderboard:ipl2024:match1")
    }
```

---

## 📊 PERFORMANCE COMPARISON

### Operations at Scale (10 Million Users)

| Operation | Data Structure Used | Complexity | Time | Ops/Second |
|-----------|---------------------|------------|------|------------|
| `ZSCORE` (get score) | Hash Table only | **O(1)** | 1 μs | 1,000,000 |
| `ZINCRBY` (update) | Hash + Skip List | O(log N) | 24 μs | 41,666 |
| `ZREVRANGE 0 99` | Skip List only | O(log N + 100) | 124 μs | 8,064 |
| `ZREVRANK` (get rank) | Skip List only | O(log N) | 24 μs | 41,666 |
| SQL `ORDER BY` | Full table scan + sort | O(N log N) | **10+ sec** | 0.1 |

### Dream11 IPL Peak Load Numbers

```
┌─────────────────────────────────────────────────────────────────┐
│            ACTUAL PERFORMANCE DURING IPL FINAL                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Metric                        │  Value                         │
│   ──────────────────────────────┼────────────────────────────── │
│   Total players                 │  10,000,000                    │
│   Updates per ball              │  2,000,000 (20% picked batsman)│
│   Update latency                │  < 200ms for all 2M updates    │
│   Leaderboard queries/second    │  500,000                       │
│   Query latency P99             │  < 10ms                        │
│   Memory usage                  │  500MB (compressed)            │
│   Server count                  │  3 Redis instances (HA)        │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 🧮 WHY THIS WORKS: THE FUNDAMENTAL INSIGHT

### SQL: Sort at Query Time

```
Every leaderboard query:
1. Fetch 10,000,000 rows
2. Sort 10,000,000 rows (O(N log N) = 230M comparisons)
3. Return top 100

Cost per query: O(N log N) = 230,000,000 operations
At 500K queries/sec: 115 QUADRILLION operations/sec (impossible!)
```

### Redis: Sort at Insert Time

```
Score update (ZINCRBY):
1. Hash lookup: O(1) = 1 operation
2. Skip List reposition: O(log N) = 24 operations
Cost per update: 25 operations

Leaderboard query (ZREVRANGE):
1. Jump to position 0: O(log N) = 24 operations
2. Walk 100 nodes: O(100) = 100 operations
Cost per query: 124 operations

At 500K queries/sec: 62 million operations/sec (easy!)
```

### The Magic Formula

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                  │
│   SQL:   Sort EVERY query    = O(N log N) per query             │
│                                                                  │
│   Redis: Sort ONCE on update = O(log N) on update               │
│          Read sorted data    = O(log N + M) on query            │
│                                                                  │
│   For 10M entries, 500K queries/sec:                            │
│   SQL:   230M × 500K = 115 quadrillion ops/sec ❌                │
│   Redis: 124 × 500K = 62 million ops/sec ✅                      │
│                                                                  │
│   Redis is 1.8 BILLION times more efficient!                    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## ❓ Interview Practice

### Question 1:
> "How would you implement a real-time leaderboard for 10 million users with updates every 30 seconds?"

**Answer:**
> "I'd use Redis Sorted Sets, which combine a Hash Table for O(1) member lookups with a Skip List for O(log N) sorted operations. Unlike SQL which sorts at query time (O(N log N)), Redis sorts at insert time (O(log N)), so queries just read pre-sorted data. For 10M users with 2M updates every 30 seconds, I'd use Redis pipelining to batch updates in one network call, achieving ~200ms update latency. Leaderboard queries would return in ~100 microseconds using ZREVRANGE. Total memory: ~500MB."

### Question 2:
> "What's the difference between a Skip List and a balanced BST? Why does Redis use Skip List?"

**Answer:**
> "Both provide O(log N) operations, but Skip Lists have key advantages for Redis: (1) Simpler implementation — no complex rotations like AVL/Red-Black trees. (2) Range queries are trivial — just walk the linked list. (3) Lock-free concurrent access is easier — each level is an independent linked list. (4) Memory locality — nodes are allocated together. Redis creator Salvatore Sanfilippo chose Skip Lists specifically because range operations (ZRANGE, ZREVRANGE) are common and efficient."

### Question 3:
> "How would you handle ties in a leaderboard (same score)?"

**Answer:**
> "Redis Sorted Sets handle ties by sorting lexicographically by member name for equal scores. For custom tie-breaking (e.g., earlier submission wins), I'd encode the timestamp in the score: `score = actual_score * 1_000_000 + (MAX_TIMESTAMP - submission_time)`. This way, higher actual scores rank first, and among equal scores, earlier submissions rank higher. The score remains a single double-precision float."

---

## 🔗 Connecting to Previous Days

| Day | Concept | How It Connects |
|-----|---------|-----------------|
| Day 15 | Redis Single-Threaded | Skip List operations are fast because Redis has no lock contention |
| Day 14 | Spring Boot Performance | Cache leaderboard results to reduce Redis load |
| Day 9 | Bloom Filters | Use Bloom Filter to check if user exists before querying |

---

## ✅ Day 16 Action Items

1. **Implement a leaderboard** — Use `ZADD`, `ZINCRBY`, `ZREVRANGE` in your next project
2. **Benchmark Skip List** — Compare `ZREVRANGE` vs SQL `ORDER BY` with 1M rows
3. **Use pipelining** — Always batch Redis commands for bulk updates
4. **Monitor memory** — Use `MEMORY USAGE key` to track sorted set size
5. **Consider sharding** — For 100M+ users, use Redis Cluster to shard by user ID hash

---

## 💡 Key Takeaways

| Insight | Why It Matters |
|---------|----------------|
| Sort at insert, not query | Amortizes O(N log N) cost across updates |
| Hash + Skip List combo | O(1) lookups + O(log N) ordering |
| Skip List = express lanes | 24 hops for 10 million entries |
| Pipelining for bulk updates | 800K updates in one network call |
| Data stays sorted | Queries just read pre-sorted data |

---

## 🎯 The Architect's Principle

> **Junior:** "I'll use SQL with ORDER BY and add an index on the score column."
>
> **Architect:** "An index helps for static data, but with 2M updates every 30 seconds, the index becomes a bottleneck. Redis Sorted Sets are designed for exactly this: high-frequency updates with instant sorted queries. The Skip List maintains order during updates, so queries never need to sort. It's the difference between 10 seconds and 100 microseconds."

The key insight: **Choose data structures that match your access patterns.** If you need sorted data with frequent updates, use a data structure that maintains sort order incrementally, not one that sorts on every query.

---

*— Sunchit Dudeja*  
*Day 16 of 50: System Design Interview Preparation Series*

---

> 🎯 **Interview Edge:** When asked about leaderboards, explain the three-step ZINCRBY operation: "Hash Table gives O(1) lookup to find the node, we update the score in O(1), then Skip List repositions the node in O(log N). The data is always sorted, so ZREVRANGE just walks the list — no sorting needed."

> 📢 **Real Impact:** Discord uses Redis Sorted Sets for their real-time activity feeds. Riot Games uses them for League of Legends global rankings. Dream11 serves 500K leaderboard queries/second during IPL matches. All powered by the same Hash Table + Skip List architecture.

---

## 🔗 Resources

- **Redis Sorted Sets Documentation:** redis.io/docs/data-types/sorted-sets
- **Skip List Paper:** "Skip Lists: A Probabilistic Alternative to Balanced Trees" by William Pugh
- **Excalidraw Diagram:** Available in course materials

---

> 💡 **Tomorrow (Day 17):** We'll explore **Rate Limiting** — how do you protect your APIs from being overwhelmed? Token Bucket, Leaky Bucket, Sliding Window, and how companies like Stripe and GitHub implement it.

