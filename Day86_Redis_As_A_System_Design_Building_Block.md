# Day 86 — Redis as a System Design Building Block: A Senior Architect's Perspective
## Beyond "Fast Key-Value Store": Responsibility, Consistency, and Failure Design

> **Series:** System Design Interview Preparation Series  
> **Difficulty:** Senior / Principal  
> **Core Concept:** Redis responsibility boundaries, consistency/durability trade-offs, outage behavior, and production workflows  
> **Prerequisite:** Day 48 — Idempotency, Day 55 — Real-Time Notifications, Day 57 — Rate Limiting, Day 81 — DLQ

---

Redis is often introduced as “a fast in-memory key-value store.” That is technically correct, but architecturally incomplete.

From a senior system design perspective, Redis is valuable because it can take several responsibilities away from your primary database and application tier: accelerating reads, coordinating distributed state, controlling traffic, processing asynchronous work, and maintaining real-time structures.

The important question is not:

> **“Where can I use Redis?”**

It is:

> **“Which system responsibility should Redis own, what consistency guarantees do I need, and what happens when Redis is unavailable?”**

The six patterns below are among the most useful Redis building blocks in large-scale systems.

---

## Senior Architect System View (Workflow)

```mermaid
flowchart TB
    C[Clients] --> G[API Gateway]
    G --> A[Application Services]
    A --> R[(Redis Tier)]
    A --> D[(Primary Database)]
    A --> Q[(Queue / Stream)]
    Q --> W[Workers]
    W --> D
    R --> C1[Cache]
    R --> C2[Sessions]
    R --> C3[Counters / Rate Limits]
    R --> C4[Leaderboards]
    R --> C5[PubSub Channels]
    R --> C6[Queue Primitives]
```

---

## 1. Caching: Protect the Database Before Optimizing Everything Else

Without caching, every frequently accessed request reaches the database.

```text
100,000 requests/sec
        |
        v
   Database
        |
   Connection pressure
        |
   High latency
```

With Redis:

```text
100,000 requests/sec
        |
        v
   Application
        |
        +---- Redis ---> Cache Hit
        |
        +---- Database ---> Cache Miss
```

### Cache Read Workflow

```mermaid
sequenceDiagram
    participant U as User
    participant APP as App
    participant REDIS as Redis Cache
    participant DB as Database

    U->>APP: GET /product/123
    APP->>REDIS: GET product:123
    alt Cache hit
        REDIS-->>APP: Value
        APP-->>U: 200 (fast)
    else Cache miss
        REDIS-->>APP: nil
        APP->>DB: SELECT product(123)
        DB-->>APP: Row
        APP->>REDIS: SETEX product:123 300 value
        APP-->>U: 200
    end
```

### What should we cache?

- Frequently accessed database records
- API responses
- Product/catalog information
- Configuration and feature flags
- Expensive aggregation results
- Authorization metadata
- Frequently queried reference data

### The key architectural decision: TTL

```text
SET product:123 = {...}
EXPIRE product:123 300
```

For critical data, you may also need:
- invalidation on writes,
- versioned keys,
- write-through/cache-aside,
- event-driven invalidation,
- stale-while-revalidate,
- negative caching.

### Senior architect question

Don’t ask, “Can Redis make this query faster?”

Ask:

> “What is acceptable staleness, and what is the failure behavior when cache disappears?”

Redis cache is typically an optimization layer, not the only persistent source of truth.

---

## 2. Session Store: Moving State Out of Application Servers

Stateless services scale better than stateful application servers.

If user session data lives only inside App-1, the next request routed to App-2 may fail user continuity.

### Session Workflow

```mermaid
flowchart LR
    LB[Load Balancer] --> A1[App-1]
    LB --> A2[App-2]
    LB --> A3[App-3]
    A1 --> R[(Redis Sessions)]
    A2 --> R
    A3 --> R
```

Example session object:

```text
session:abc123
{
  userId: 9182,
  roles: ["admin"],
  cartId: "cart-772",
  expiresAt: ...
}
```

Redis works well because sessions need low-latency read/write, TTL expiration, and centralized access.

### Senior decision boundary

If access tokens are self-contained and short-lived, centralized Redis session lookups may be unnecessary.  
The bigger decision is auth model: centralized mutable session vs largely stateless token validation.

---

## 3. Leaderboards: Redis as a Real-Time Ranking Engine

Leaderboards need rank/order operations at high write/read frequency.

Sorted sets map naturally:

```text
ZADD leaderboard 9500 player123
```

### Leaderboard Workflow

```mermaid
flowchart LR
    GS[Game Service] --> D[(Primary DB)]
    GS --> Z[(Redis Sorted Set)]
    Z --> T[Top 100]
    Z --> P[Player Rank]
    Z --> N[Nearby Players]
```

The database remains durable source of business data; Redis serves real-time rank/query traffic.

> Choose data structure by access pattern, not by familiarity.

---

## 4. Pub/Sub Messaging: Decoupling Services (With Explicit Loss Semantics)

Synchronous coupling amplifies latency/failure.

Eventing model:

```mermaid
flowchart LR
    O[Order Service] --> B[Redis Pub/Sub Channel]
    B --> P[Payment]
    B --> I[Inventory]
    B --> N[Notification]
```

Channel examples:

```text
channel:order-events
OrderCreated
OrderPaid
OrderCancelled
```

### Critical warning

Redis Pub/Sub is not a durable queue. If subscriber is disconnected, messages may be missed.

Decision question:

> “Can this event be lost?”

If yes, Pub/Sub can fit. If no, use durable stream/queue semantics.

---

## 5. Counters and Rate Limiting: Protecting the System

Redis counters are strong primitives for distributed throttling.

```text
INCR rate:user:12345
EXPIRE rate:user:12345 60
```

### Rate Limiter Workflow

```mermaid
sequenceDiagram
    participant C as Client
    participant GW as API Gateway
    participant RL as Rate Limiter
    participant R as Redis
    participant APP as Application

    C->>GW: Request
    GW->>RL: Check quota(user/IP/key)
    RL->>R: INCR key + EXPIRE window
    R-->>RL: Current count
    alt within threshold
        RL-->>GW: Allow
        GW->>APP: Forward request
        APP-->>C: 200
    else exceeded threshold
        RL-->>GW: Reject/Throttle
        GW-->>C: 429 Too Many Requests
    end
```

Also useful for likes, views, downloads, login attempts, quotas, and real-time usage metrics.

At scale, algorithm choice matters: fixed window, sliding window, token bucket, leaky bucket.

Senior-level concerns:
- atomicity and race conditions,
- hot keys,
- time-window boundaries,
- multi-instance consistency,
- behavior during Redis outages.

---

## 6. Queues: Moving Slow Work Out of the Request Path

Don’t block user latency on work that can run later.

```mermaid
flowchart LR
    REQ[API Request] --> APP[Application]
    APP --> Q[(Redis Queue / Stream)]
    Q --> W1[Worker 1]
    Q --> W2[Worker 2]
    Q --> W3[Worker 3]
    W1 --> X[Email/Report/Image/External Calls]
    W2 --> X
    W3 --> X
```

Use cases:
- email dispatch,
- report generation,
- media processing,
- analytics pipelines,
- downstream integrations.

Production queue concerns:
- delivery guarantees,
- retry/backoff,
- dead-letter handling,
- visibility timeout,
- idempotency,
- ordering,
- backpressure,
- observability.

Queue is not just `push(job)/pop(job)`; it is a reliability boundary.

---

## Redis Is Not Six Different Technologies

| Pattern | Primary Problem |
|---|---|
| Caching | Reduce latency and DB load |
| Session Store | Centralize temporary app state |
| Leaderboards | Real-time ranking |
| Pub/Sub | Real-time notifications |
| Counters | Distributed counting and traffic control |
| Queues | Asynchronous processing |

Shared architectural theme:

> Redis moves high-frequency, low-latency workloads away from slower or expensive components.

---

## Real Architecture: Redis + Database + Queue

```mermaid
flowchart TB
    U[Users] --> G[Gateway]
    G --> S[Application Services]
    S --> R[(Redis)]
    S --> D[(Database)]
    S --> Q[(Queue/Stream)]
    Q --> W[Workers]
    W --> D
    R --> R1[Cache]
    R --> R2[Session]
    R --> R3[Counter]
    R --> R4[Ranking]
```

- **Database:** durable business state, transactions, long-term persistence.
- **Redis:** low-latency temporary/derived state and real-time structures.
- **Queue/Event layer:** async work, decoupling, retry, work distribution.

This is separation of concerns in practice.

---

## The Questions I Ask Before Adding Redis

1. **What exact problem are we solving?** (latency, load, coordination, async, ranking, temporary state)
2. **What happens if Redis is down?** (degrade, fallback, fail-open/closed, blast radius)
3. **Is Redis source of truth or derived state?**
4. **What consistency/freshness is required?**
5. **What happens during spikes?** (hot keys, memory, evictions, connection limits, network)
6. **Can we rebuild this data?** (cache warmup, replay, re-auth, job recovery)

### Redis Outage Workflow (Design-Time Mandatory)

```mermaid
flowchart TD
    A[Redis unavailable] --> B{Use case}
    B -->|Cache| C[Fallback to DB with load-shed]
    B -->|Session| D[Re-auth or stateless token fallback]
    B -->|Rate Limiter| E[Fail-open or fail-closed policy]
    B -->|Queue| F[Pause producers or route to durable queue]
    B -->|Leaderboard| G[Serve stale snapshot/degraded mode]
```

---

## Practical Decision Framework

```mermaid
flowchart TD
    A[Workload] --> B{Latency-sensitive?}
    B -->|No| X[Likely DB/queue only]
    B -->|Yes| C{Temporary or reconstructable data?}
    C -->|Yes| R[Redis strong fit]
    C -->|No| D{Need durability as source of truth?}
    D -->|Yes| Y[Primary DB + Redis as derived layer]
    D -->|No| R
    R --> E[Define consistency + outage strategy + capacity plan]
```

This prevents “Redis everywhere” anti-patterns.

---

## Final Takeaway

Redis is powerful because it can be:
- a cache,
- a session store,
- a ranking engine,
- a real-time messaging layer,
- a counter/rate-limiter primitive,
- and a queueing building block.

But the senior design perspective is not:

> “Redis can do all these things.”

It is:

> “Redis can do all these things, and each has different consistency, durability, availability, and failure characteristics.”

The strongest architecture is not the one that uses Redis everywhere.  
It is the one where every Redis deployment has:
- a clearly defined responsibility,
- a known failure mode,
- an explicit source of truth,
- and a measurable reason for existing.

That is the difference between **adding Redis to architecture** and **architecting Redis into a system**.

