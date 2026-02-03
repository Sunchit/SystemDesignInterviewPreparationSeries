# Redis Single-Threaded Magic: Why It's Faster Than Your Multi-Threaded Database
### Day 15 of 50 - System Design Interview Preparation Series

**By Sunchit Dudeja**

---

## 🎯 Welcome to Day 15!

Yesterday, we mastered Spring Boot performance optimization with the architect's 6-step framework. Today, we tackle one of the most **counterintuitive concepts** in system design — why Redis, a single-threaded system, outperforms your multi-threaded database.

> Redis isn't fast DESPITE being single-threaded. It's fast BECAUSE it's single-threaded.

---

## 🤔 The Paradox That Breaks Developers' Brains

### The Interview Question

> "Redis is single-threaded. Our 32-core application servers use multi-threading. How can Redis possibly be faster?"

### The Mind-Bending Numbers

```
┌─────────────────────────────────────────────────────────────────┐
│                    THE PARADOX                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   YOUR APPLICATION:                                              │
│   • 32 CPU cores                                                 │
│   • Multi-threaded                                               │
│   • Result: 10,000 operations/second                            │
│                                                                  │
│   REDIS:                                                         │
│   • 1 CPU core                                                   │
│   • Single-threaded                                              │
│   • Result: 1,000,000 operations/second                         │
│                                                                  │
│   ❓ How is Redis 100x faster with 32x fewer cores?             │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

Let's solve this paradox...

---

## 🎯 THE 5 SECRETS OF REDIS'S SINGLE-THREADED SPEED

```
┌─────────────────────────────────────────────────────────────────┐
│                 THE 5 SECRETS                                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   SECRET 1: Zero Context Switching                               │
│   SECRET 2: Zero Synchronization Locks                           │
│   SECRET 3: In-Memory Everything                                 │
│   SECRET 4: Simple, Predictable Data Structures                  │
│   SECRET 5: Event-Driven I/O Multiplexing                        │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## ⚡ SECRET 1: ZERO CONTEXT SWITCHING

### What Happens in Your Multi-Threaded App

```java
// Thread 1 running...
// OS scheduler: "Time's up!"
// → SAVE Thread 1 state (registers, stack pointer, program counter)
// → LOAD Thread 2 state
// → RUN Thread 2...

// Every 10ms: expensive context switch
// Cost: 1-10 microseconds per switch
```

### What Happens in Redis

```c
// One thread runs FOREVER
while(true) {
    // No context switches
    // No thread synchronization
    // No lock contention
    processNextCommand();
}
```

### The Impact

```
Context Switch Analysis at 1M ops/sec:

Multi-threaded App:
├── 1M ops ÷ 32 threads = 31,250 ops per thread
├── Context switches between threads: ~100,000/sec
├── Cost per switch: 1-10 microseconds
└── Total overhead: 100,000 × 5μs = 0.5 seconds wasted/second!

Redis:
├── 1M ops on 1 thread
├── Context switches: 0
├── Overhead: 0
└── 100% CPU time on actual work
```

| Metric | Multi-Threaded | Redis |
|--------|----------------|-------|
| Context switches/sec | ~100,000 | **0** |
| Time wasted | 0.5 sec/sec | **0** |
| CPU efficiency | ~50% | **~100%** |

---

## 🔓 SECRET 2: ZERO SYNCHRONIZATION LOCKS

### Your App with Locks

```java
public class ShoppingCart {
    private Map<String, Item> items = new ConcurrentHashMap<>();
    private final Object lock = new Object();
    
    public void addItem(String userId, Item item) {
        synchronized(lock) {          // ⏳ WAIT HERE
            items.put(userId, item);  // Critical section
        }                             // Others blocked!
    }
}

// What's really happening:
// Thread A: Has lock, processing
// Thread B: Waiting, doing nothing
// Thread C: Waiting, doing nothing
// ...
// Thread 32: Waiting, doing nothing
// 
// Result: 31 cores IDLE while 1 works!
```

### Redis: No Locks Anywhere

```c
// NO LOCKS ANYWHERE in command processing
void processCommand(redisCommand *cmd) {
    // Direct memory access
    // No mutexes, no semaphores, no spinlocks
    lookupKey(cmd->key);  // Pure computation
    // Done!
}
```

### The Lock Contention Problem

```
Multi-Threaded Lock Contention:
┌─────────────────────────────────────────────────────────────────┐
│                                                                  │
│   Thread 1: [====WORKING====]                                   │
│   Thread 2: [WAITING........][==WORK==]                         │
│   Thread 3: [WAITING................][==WORK==]                 │
│   Thread 4: [WAITING........................][==WORK==]         │
│                                                                  │
│   Actual parallelism: 1 thread at a time (on shared data)       │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘

Redis Single-Threaded:
┌─────────────────────────────────────────────────────────────────┐
│                                                                  │
│   Thread 1: [WORK][WORK][WORK][WORK][WORK][WORK][WORK][WORK]    │
│                                                                  │
│   No waiting, no contention, 100% utilized                       │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 💾 SECRET 3: IN-MEMORY EVERYTHING

### Your Database: The Disk I/O Tax

```
Query: SELECT * FROM users WHERE id = 123

┌─────────────────────────────────────────────────────────────────┐
│  DISK I/O PATH (SLOW)                                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. CPU: "Get user:123"                                         │
│  2. OS: Check page cache (maybe in RAM)                         │
│  3. If not cached:                                               │
│     a. Disk seek: 3-10ms (moving head)                          │
│     b. Disk read: 0.1ms                                          │
│  4. Copy to user space                                           │
│  5. Parse result                                                 │
│                                                                  │
│  TOTAL: ~5ms minimum (often 10-50ms)                            │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Redis: Pure RAM Access

```
Query: GET user:123

┌─────────────────────────────────────────────────────────────────┐
│  RAM ACCESS (FAST)                                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. CPU: "Get user:123"                                         │
│  2. Hash lookup → Direct pointer to memory location              │
│  3. Read value from RAM                                          │
│                                                                  │
│  TOTAL: ~0.1 microseconds (0.0001ms)                            │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### The Speed Difference

| Storage | Access Time | Relative Speed |
|---------|-------------|----------------|
| **L1 Cache** | 1 ns | 1x (baseline) |
| **L2 Cache** | 4 ns | 4x slower |
| **L3 Cache** | 40 ns | 40x slower |
| **RAM** | 100 ns | 100x slower |
| **SSD** | 100,000 ns | **100,000x slower** |
| **HDD** | 10,000,000 ns | **10,000,000x slower** |

```
Redis advantage:
├── All data in RAM: 100 nanoseconds access
├── Frequently used data in L1/L2 cache: 1-4 nanoseconds
├── No disk seeks, no I/O waits
└── Skips 99.999% of the latency!
```

---

## 📦 SECRET 4: SIMPLE, PREDICTABLE DATA STRUCTURES

### Your ORM/Object Mapping Overhead

```java
@Entity
public class User {
    @Id 
    private Long id;
    private String name;
    private String email;
    @OneToMany 
    private List<Order> orders;
    
    // What happens behind the scenes:
    // - Reflection to read annotations
    // - Proxy generation
    // - Lazy loading interceptors
    // - Object allocation
    // - JSON serialization (multiple copies)
}

// Memory layout of Java HashMap entry:
// ┌──────────────────────────────────────┐
// │ Object header:     12 bytes          │
// │ Hash code:         4 bytes           │
// │ Key reference:     8 bytes           │
// │ Value reference:   8 bytes           │
// │ Next pointer:      8 bytes           │
// │ Padding:           4 bytes           │
// ├──────────────────────────────────────┤
// │ TOTAL per entry:   44 bytes          │
// └──────────────────────────────────────┘
```

### Redis: Minimal Overhead

```c
// Redis object: Just 16 bytes!
struct redisObject {
    unsigned type:4;      // 4 bits: string, list, hash, set, zset
    unsigned encoding:4;  // 4 bits: raw, int, ziplist, hashtable
    unsigned lru:24;      // 24 bits: LRU time
    int refcount;         // 4 bytes: reference counting
    void *ptr;            // 8 bytes: pointer to actual data
};
// TOTAL: 16 bytes

// String "hello": 
// redisObject (16 bytes) + "hello\0" (6 bytes) = 22 bytes

// Compare to Java String "hello":
// String object header (12) + char array header (12) + 
// char data (10) + padding = 40+ bytes
```

### Memory Layout Advantage

```
Java HashMap (1000 entries):
├── HashMap object: 48 bytes
├── Entry array: 8,000 bytes (8 × 1000 pointers)
├── Entry objects: 44,000 bytes (44 × 1000)
├── Key objects: ~40,000 bytes
├── Value objects: ~40,000 bytes
└── TOTAL: ~132,000 bytes + fragmentation

Redis Hash (1000 entries):
├── Hash object: 16 bytes
├── Ziplist (compact): ~15,000 bytes (contiguous!)
└── TOTAL: ~15,000 bytes

Redis uses 8x LESS memory for same data!
```

---

## 🔄 SECRET 5: EVENT-DRIVEN I/O MULTIPLEXING

### Traditional: One Thread Per Connection

```
10,000 clients connected:

┌─────────────────────────────────────────────────────────────────┐
│  THREAD-PER-CONNECTION MODEL                                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Connection 1  → Thread 1  [BLOCKED on read.............]       │
│  Connection 2  → Thread 2  [BLOCKED on read.............]       │
│  Connection 3  → Thread 3  [BLOCKED on read.............]       │
│  ...                                                             │
│  Connection 10,000 → Thread 10,000 [BLOCKED on read.....]       │
│                                                                  │
│  Problem:                                                        │
│  • 10,000 threads created                                        │
│  • 10,000 × 1MB stack = 10GB memory just for stacks!            │
│  • 99% of threads sleeping, doing nothing                       │
│  • Massive context switching overhead                            │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Redis: epoll/kqueue I/O Multiplexing

```c
// Single thread manages ALL connections efficiently
void aeMain(aeEventLoop *eventLoop) {
    while (!stop) {
        
        // 1. One system call checks ALL 10,000 sockets
        int numEvents = epoll_wait(epfd, events, MAX_EVENTS, timeout);
        
        // 2. Only process sockets that HAVE data
        for (int i = 0; i < numEvents; i++) {
            processClient(events[i].data.fd);  // Non-blocking
        }
        
        // 3. Handle timers, background tasks
        processTimeEvents(eventLoop);
    }
}
```

### I/O Multiplexing Visualization

```
Redis with 10,000 connections:

┌─────────────────────────────────────────────────────────────────┐
│  I/O MULTIPLEXING (epoll/kqueue)                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│                    ┌──────────────┐                             │
│  Connection 1  ───→│              │                             │
│  Connection 2  ───→│    epoll     │                             │
│  Connection 3  ───→│              │───→ Single Thread           │
│  ...              →│  "Tell me    │     processes ready         │
│  Connection 10,000→│   which are  │     sockets in batch        │
│                    │   ready"     │                             │
│                    └──────────────┘                             │
│                                                                  │
│  Benefits:                                                       │
│  • 1 thread, not 10,000                                         │
│  • 1MB stack, not 10GB                                          │
│  • Zero context switches between clients                         │
│  • O(ready_count) not O(total_connections)                      │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 🧮 THE SPEED COMPARISON

### Operation: GET user:123

#### Java Spring Boot + MySQL

```
┌─────────────────────────────────────────────────────────────────┐
│  JAVA + SPRING + MYSQL PATH                                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Step                              Time                          │
│  ─────────────────────────────────────────                      │
│  1. HTTP Request parsing           0.1ms                         │
│  2. Spring MVC routing             0.05ms                        │
│  3. Service layer logic            0.05ms                        │
│  4. Hibernate session open         0.1ms                         │
│  5. MySQL connection acquire       0.5ms                         │
│  6. SQL parsing                    0.1ms                         │
│  7. Query execution (with index)   1.0ms                         │
│  8. Network round-trip to DB       0.5ms                         │
│  9. Result set mapping             0.2ms                         │
│  10. JSON serialization            0.1ms                         │
│  ─────────────────────────────────────────                      │
│  TOTAL:                            2.7ms (2,700 microseconds)    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

#### Redis

```
┌─────────────────────────────────────────────────────────────────┐
│  REDIS PATH                                                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Step                              Time                          │
│  ─────────────────────────────────────────                      │
│  1. Read command from socket       0.01ms  (already in buffer)  │
│  2. Parse command                  0.001ms (simple protocol)    │
│  3. Lookup key in hash table       0.0001ms (O(1) operation)    │
│  4. Send response                  0.01ms                        │
│  ─────────────────────────────────────────                      │
│  TOTAL:                            0.021ms (21 microseconds)     │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### The Verdict

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                  │
│   Java + MySQL:  2,700 microseconds                             │
│   Redis:         21 microseconds                                 │
│                                                                  │
│   ⚡ Redis is 128x FASTER!                                      │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 🔬 THE MICROSECOND BREAKDOWN

### Redis Command: GET key (nanosecond analysis)

```
Time (ns)   What's Happening
──────────  ─────────────────────────────────────────
    100     Read from kernel buffer (already in RAM)
     50     Parse "GET" command (3 bytes)
     10     Lookup command handler in table
    100     Hash key lookup (O(1) in dict)
     50     Access value pointer
    100     Write to output buffer
     50     Mark socket as writable (epoll)
──────────
    460 ns total ≈ 0.00046ms
```

### Your Java App: GET operation (nanosecond analysis)

```
Time (ns)    What's Happening
───────────  ─────────────────────────────────────────
   1,000     Enter synchronized block (acquire monitor)
   5,000     Thread context switch (if lock was held)
  10,000     HashMap.get() with memory barriers
   2,000     Object allocation for response
   5,000     JSON serialization (reflection, type checks)
   2,000     HTTP response writing
───────────
  25,000 ns total ≈ 0.025ms (54x slower just for app logic!)
```

---

## 🎯 WHEN MULTI-THREADING HURTS

### The False Promise of Parallelism

```java
// Looks parallel but isn't!
ExecutorService pool = Executors.newFixedThreadPool(32);
List<Future<String>> futures = new ArrayList<>();

for (int i = 0; i < 1000; i++) {
    futures.add(pool.submit(() -> {
        return redis.get("key:" + i);  // Oops!
    }));
}

// What REALLY happens:
// ┌─────────────────────────────────────────────────────────────┐
// │ 32 threads → 32 simultaneous Redis connections              │
// │ Redis: Processes 32 commands... SEQUENTIALLY anyway!       │
// │ Plus: Thread pool overhead, future allocations, switches   │
// │                                                             │
// │ You added overhead, Redis still processes one at a time!   │
// └─────────────────────────────────────────────────────────────┘
```

### The Better Approach: Pipelining

```java
// Single connection, pipelined - Redis-optimized
List<Object> results = redis.pipelined(pipe -> {
    for (int i = 0; i < 1000; i++) {
        pipe.get("key:" + i);
    }
});

// What happens:
// ┌─────────────────────────────────────────────────────────────┐
// │ 1 network round-trip (not 1000!)                            │
// │ Redis processes all 1000 commands in optimal order          │
// │ No thread overhead, no synchronization                      │
// │                                                              │
// │ 10x faster than the "parallel" approach!                    │
// └─────────────────────────────────────────────────────────────┘
```

---

## 🏃 THE USAIN BOLT ANALOGY

```
┌─────────────────────────────────────────────────────────────────┐
│                    THE ANALOGY                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  🏃 USAIN BOLT (Redis):                                         │
│  • One track, 100% focused                                       │
│  • No meetings to attend                                         │
│  • No context switching                                          │
│  • Just running, running, running                                │
│  • Result: World record                                          │
│                                                                  │
│  👔 32 OFFICE WORKERS (Your Multi-Threaded App):                │
│  • 32 people available                                           │
│  • But constantly in meetings (lock contention)                  │
│  • Switching between tasks (context switches)                    │
│  • Filling paperwork (serialization overhead)                    │
│  • Waiting for each other (synchronization)                      │
│  • Result: Slower than one focused person                        │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 🏗️ THE ARCHITECT'S INSIGHT

### What Redis Proves

| Your App Is Slow Because | Redis Eliminates It |
|--------------------------|---------------------|
| Waiting for locks | No locks needed (single thread) |
| Context switching | No switching (one thread forever) |
| Memory allocation/GC | Simple structures, minimal allocation |
| Serialization overhead | Direct memory access |
| Disk I/O | Everything in RAM |
| Network overhead per request | I/O multiplexing, pipelining |

### The Counterintuitive Truth

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                  │
│  MORE THREADS ≠ MORE THROUGHPUT                                 │
│                                                                  │
│  When work is:                                                   │
│  • CPU-bound                                                     │
│  • Operating on shared data                                      │
│  • Requiring synchronization                                     │
│                                                                  │
│  Then:                                                           │
│  • Threads spend time WAITING, not WORKING                      │
│  • Context switches waste CPU cycles                             │
│  • Lock contention serializes parallel work                      │
│                                                                  │
│  Redis proves: Sometimes the fastest coordination               │
│  is NO coordination at all.                                      │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 🔧 HOW TWITTER RUNS REDIS ON 32-CORE SERVERS

> "But wait, if Redis is single-threaded, how do companies use their 32-core servers?"

### The Answer: Multiple Redis Processes

```
┌─────────────────────────────────────────────────────────────────┐
│  32-CORE SERVER WITH REDIS                                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Core 0:  [Redis Instance 1 - Users Cache]                      │
│  Core 1:  [Redis Instance 2 - Users Cache]                      │
│  Core 2:  [Redis Instance 3 - Session Store]                    │
│  Core 3:  [Redis Instance 4 - Session Store]                    │
│  Core 4:  [Redis Instance 5 - Rate Limiting]                    │
│  Core 5:  [Redis Instance 6 - Pub/Sub]                          │
│  ...                                                             │
│  Core 31: [Redis Instance 32 - Timeline Cache]                  │
│                                                                  │
│  Each instance: Single-threaded, 100% efficient                  │
│  Combined: 32 million ops/sec capacity!                          │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Redis Cluster: Horizontal Scaling

```
┌─────────────────────────────────────────────────────────────────┐
│  REDIS CLUSTER (16,384 hash slots)                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Server 1 (Slots 0-5460):     3 Redis instances                 │
│  Server 2 (Slots 5461-10922): 3 Redis instances                 │
│  Server 3 (Slots 10923-16383): 3 Redis instances                │
│                                                                  │
│  Key "user:123" → CRC16 hash → Slot 7890 → Server 2             │
│                                                                  │
│  Total capacity: 9 instances × 1M ops = 9M ops/sec              │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## ❓ Interview Practice

### Question 1:
> "Redis is single-threaded. How can it handle millions of requests?"

**Answer:**
> "Redis is fast BECAUSE it's single-threaded, not despite it. Single-threaded means zero lock contention, zero context switching, and 100% CPU utilization on actual work. Combined with in-memory storage (100ns vs 10ms for disk), I/O multiplexing (one thread handling 10K connections via epoll), and simple data structures (16 bytes overhead vs 44+ for Java), Redis achieves sub-microsecond operations. For horizontal scaling, you run multiple Redis instances or use Redis Cluster."

### Question 2:
> "When would you NOT use Redis?"

**Answer:**
> "Redis isn't ideal for: (1) Data larger than RAM — Redis is purely in-memory. (2) Complex queries with joins — Redis has simple data structures, not a query engine. (3) Strong durability requirements — while Redis has persistence, a database with WAL is more reliable for transactions. (4) Data that needs complex indexing — secondary indexes in Redis are manual. I'd use Redis for caching, sessions, rate limiting, pub/sub, and leaderboards — not as a primary database for critical financial transactions."

### Question 3:
> "How do you scale Redis for a 10M ops/sec requirement?"

**Answer:**
> "I'd use Redis Cluster with multiple shards. Each shard handles ~1M ops/sec, so 10-12 shards would provide capacity with headroom. Keys are distributed using CRC16 hash across 16,384 slots. For read-heavy workloads, I'd add read replicas to each master. I'd also ensure clients use pipelining (batch commands) and connection pooling (reuse connections). Finally, I'd monitor with Redis INFO command for memory, connections, and ops/sec per instance."

---

## 🔗 Connecting to Previous Days

| Day | Concept | How It Connects |
|-----|---------|-----------------|
| Day 10 | Request Coalescing | Redis handles coalescing naturally (single-threaded) |
| Day 13 | Circuit Breaker | Protect Redis connections with circuit breakers |
| Day 14 | Spring Boot Performance | Use Redis to cache and reduce DB load |

---

## ✅ Day 15 Action Items

1. **Benchmark your cache** — Compare Redis GET vs database SELECT for same data
2. **Use pipelining** — Batch Redis commands instead of one-at-a-time
3. **Monitor ops/sec** — Use `redis-cli INFO stats` to see throughput
4. **Right-size instances** — One Redis instance per CPU core
5. **Understand your access patterns** — Cache what's read frequently

---

## 💡 Lessons Learned

| Lesson | Why It Matters |
|--------|----------------|
| Single-threaded can be faster | Eliminates coordination overhead |
| RAM is 100,000x faster than SSD | In-memory is non-negotiable for speed |
| Simplicity beats complexity | 16-byte objects vs 44-byte Java objects |
| I/O multiplexing scales | One thread for 10K connections |
| More threads ≠ more throughput | When sharing data, contention kills performance |

---

## 🚀 Key Architect Principles

| Principle | What It Means |
|-----------|---------------|
| **Eliminate coordination** | Locks and sync are expensive |
| **Stay in memory** | Disk I/O is the enemy of latency |
| **Simple data structures** | Less overhead, more speed |
| **Batch operations** | Pipelining beats round-trips |
| **Scale horizontally** | Multiple processes > more threads |

---

## 💡 Key Takeaway

> **Junior: "Redis is single-threaded? That must be a bottleneck. Let me use a multi-threaded cache instead."**
>
> **Architect: "Redis is single-threaded BY DESIGN. It eliminates lock contention, context switching, and synchronization overhead. Combined with in-memory storage and I/O multiplexing, it achieves 1M ops/sec on a single core. For more capacity, I run multiple Redis instances — that's horizontal scaling without the complexity of multi-threaded shared state."**

The difference? Understanding that **coordination has a cost**. Sometimes the fastest way to process work is to eliminate all the overhead of coordinating parallel work.

---

*— Sunchit Dudeja*  
*Day 15 of 50: System Design Interview Preparation Series*

---

> 🎯 **Interview Edge:** When asked about Redis performance, explain the 5 secrets: "Zero context switching, zero locks, in-memory storage, simple data structures, and I/O multiplexing. Redis proves that single-threaded can outperform multi-threaded when you eliminate coordination overhead."

> 📢 **Real Impact:** Twitter runs hundreds of Redis instances serving millions of ops/sec. Each instance is single-threaded, but combined they handle Twitter's entire caching layer. The key insight: multiple simple processes beat one complex multi-threaded process.

---

## 🔗 Resources

- **Excalidraw Diagram:** Available in course materials
- **Redis Source Code:** github.com/redis/redis (study ae.c for event loop)
- **Benchmark:** `redis-benchmark -q -n 100000`

---

> 💡 **Tomorrow (Day 16):** We'll explore **Database Sharding** — how do you split a billion-row table across 100 servers? The strategies, trade-offs, and real-world implementations at companies like Instagram and Pinterest.

