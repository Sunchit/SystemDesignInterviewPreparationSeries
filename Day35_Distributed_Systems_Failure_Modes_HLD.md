# 🏛️ Distributed Systems Failure Modes — Complete HLD Architecture
### Day 35 of 50 - System Design Interview Preparation Series

**By Sunchit Dudeja**

---

## 🎯 The Comprehensive Failure Landscape

Distributed failures stack from **edge** through **application**, **data**, **messaging**, **external dependencies**, and **infrastructure**. Each layer has predictable symptoms, root causes, and mitigations. Use the **dark warm palette** below for slides, diagrams, and HLD drawings so layers stay visually distinct without neon brights.

---

## 📋 Failure Modes Catalog

### 🔵 Edge & Network Layer (Dark Yellow · `#8B5A2B`)

| Failure | Symptoms | Root Cause | Mitigation |
|---------|----------|------------|------------|
| **DNS Failure** | "Site cannot be reached", high latency | DNS provider outage, TTL misconfiguration | Multi-provider DNS, short TTL, client retry |
| **CDN Failure** | Static assets loading slowly | Edge node failure, cache invalidation storms | Multi-CDN, origin fallback, cache warming |
| **Load Balancer** | 502/503 errors, uneven traffic | Health check failure, backend pool exhaustion, sticky session overflow | Redundant LBs, proper health check tuning, connection draining |
| **API Gateway** | Authentication failures, routing errors | Auth service timeout, rate limit misconfiguration, circuit breaker open | Local token cache, circuit breaker recovery, fallback auth |

---

### 🟢 Application Layer (Dark Orange · `#A66B2E`)

| Failure | Symptoms | Root Cause | Mitigation |
|---------|----------|------------|------------|
| **Thread Deadlock** | Request hangs, no response | Circular lock dependencies, improper synchronization | Lock timeout, thread dump analysis, async patterns |
| **Memory Leak** | OOM kills, GC thrashing | Unclosed resources, static caches, listener leaks | Heap dump analysis, monitoring, bounded caches |
| **CPU Throttling** | Increased latency | CPU contention, inefficient algorithms | Profiling, horizontal scaling, compute offloading |
| **GC Pause** | Latency spikes | Large heap, frequent full GC | GC tuning, smaller heaps, concurrent GC |
| **Internal Rate Limit** | 429 errors | Self-throttling, dependency backpressure | Adaptive rate limiting, queue sizing |

---

### 🟡 Data Layer (Dark Golden · `#B87C2E`)

| Failure | Symptoms | Root Cause | Mitigation |
|---------|----------|------------|------------|
| **Cache Penetration** | DB overload, slow reads | Queries for non-existent keys | Bloom filter, cache empty values |
| **Cache Stampede** | Cache miss storm | Simultaneous expiration, thundering herd | Request coalescing, jitter, probabilistic early expiration |
| **Split Brain** | Inconsistent data | Network partition, leader election failure | Quorum-based writes, fencing tokens |
| **Replication Lag** | Stale reads | Slow replica, network latency | Monitor lag, read from primary for critical queries |
| **Hot Partition** | Single shard overload | Skewed data distribution | Pre-split, consistent hashing, partition rebalancing |
| **Connection Pool Exhaustion** | "Too many connections" | Leaked connections, pool size too small | Monitoring, connection validation, proper release |
| **Deadlock (DB)** | Transaction hangs | Lock contention, long-running transactions | Lock timeouts, retry logic, optimistic locking |

---

### 🟠 Message Layer (Dark Orange · `#A65D1A`)

| Failure | Symptoms | Root Cause | Mitigation |
|---------|----------|------------|------------|
| **Consumer Lag** | Processing delay, backlog | Slow consumers, partition imbalance | Auto-scaling, partition increase, consumer tuning |
| **Rebalance** | Processing pause | Consumer joins/leaves, timeouts | Sticky assignor, session timeout tuning, KIP-848 |
| **Poison Pill** | Consumer stuck, DLQ | Unprocessable message, serialization error | DLQ, skip ahead, schema validation |
| **Producer Failure** | Data loss, retries | Network issues, broker unavailable | Idempotent producers, retry with backoff |
| **Topic Unavailable** | Write failures | Leader election, ISR shrinking | `min.insync.replicas`, proper replication |

---

### 🔴 External Dependency Layer (Dark Burnt Orange · `#B8470C`)

| Failure | Symptoms | Root Cause | Mitigation |
|---------|----------|------------|------------|
| **Timeout** | Slow response, thread pileup | Slow external service, network latency | Short timeouts, circuit breaker |
| **Circuit Open** | Fast failures, degraded experience | Dependency failure threshold crossed | Graceful degradation, fallback responses |
| **Rate Limit Exceeded** | 429 errors | Quota exhaustion | Token bucket, request queuing, caching |
| **Provider Outage** | Service unavailable | Third-party downtime | Multi-provider, queue for later, offline mode |

---

### ⚫ Infrastructure Layer (Dark Brown · `#7A3A1A`)

| Failure | Symptoms | Root Cause | Mitigation |
|---------|----------|------------|------------|
| **Node Failure** | Pod eviction, capacity loss | Hardware failure, OOM, kernel panic | Multi-AZ, pod disruption budgets, node pools |
| **Network Partition** | Split brain, connectivity loss | Cable cut, switch failure | Multi-region, consistent hashing, quorum |
| **Disk IOPS Exhaustion** | Slow reads/writes | Throttling, noisy neighbor | Provisioned IOPS, disk sizing, monitoring |
| **AZ Outage** | Complete zone failure | Power, network, cooling | Multi-AZ, cross-region DR |
| **Region Outage** | Catastrophic failure | Natural disaster, major incident | Cross-region replication, failover plan |

---

## 🛡️ Resilience Patterns Map (Dark Palette)

| Pattern | Primary layers | Role |
|---------|----------------|------|
| **Timeouts + circuit breaker** | External, Gateway, Application | Fail fast; stop hammering sick dependencies |
| **Retry with exponential backoff + jitter** | External, Message, Data (transient) | Recover from blips without stampedes |
| **Bulkhead** | Application, Data (pools) | Isolate resource pools so one flood doesn’t sink all traffic |
| **Rate limiting / admission control** | Edge, Gateway, Application | Protect cores from overload |
| **Graceful degradation + fallbacks** | External, Application | Partial UX vs total outage |
| **Idempotency + deduplication** | Message, External callbacks | Safe retries; no double charges |
| **Quorum / fencing / strong leader** | Data, Infrastructure | Avoid split brain |
| **Multi-AZ / replication / DR** | Infrastructure, Data | Survive node and zone loss |
| **Caching discipline** (TTL jitter, coalescing, bloom) | Data | Cut penetration and stampede |
| **Observability** (SLOs, Synthetics, tracing) | All (`#A6761A` observability stripe) | Detect shift-left; shorten MTTR |

---

## 🎨 Color Palette Guide

Use these hex values in Excalidraw, Figma, or slide masters so HLD diagrams stay consistent with this post.

| Layer | Color name | Hex | Usage |
|-------|------------|-----|--------|
| Edge & Network | Dark Yellow | `#8B5A2B` | DNS, WAF, load balancer |
| Application | Dark Orange | `#A66B2E` | App pods, JVM failure points |
| Data | Dark Golden | `#B87C2E` | Cache, database, storage |
| Message | Dark Orange | `#A65D1A` | Kafka, queues, streams |
| External | Dark Burnt Orange | `#B8470C` | Payment, email, SMS APIs |
| Infrastructure | Dark Brown | `#7A3A1A` | Kubernetes, network, region |
| Observability | Dark Golden | `#A6761A` | Metrics, logs, alerting |
| Text (on dark fills) | Light Yellow | `#FFD966` | Labels and titles |
| Warning accents | Dark Orange-Red | `#CC6D2C` | Failure indicators |
| Critical accents | Dark Red-Brown | `#B85C2C` | Critical / region-wide failures |

**Optional accent (diagrams):** Dark rose `#9B4D6F` — use for emphasis rings or “high risk” callouts without clashing with the orange family.

---

## 📊 Failure Detection & Recovery Matrix

| Layer | Detection metric | Alert threshold | Recovery action | RTO |
|-------|------------------|-----------------|-----------------|-----|
| **DNS** | DNS resolution time | > 500 ms, failure rate > 1% | Failover to secondary DNS | ~5 min |
| **Load Balancer** | 5xx rate, active connections | > 5% for 2 min | Add backend, failover to standby | ~2 min |
| **Application** | P99 latency, error rate, CPU, memory | > 1 s latency, > 5% error, > 80% CPU | Auto-scale, restart pod, open circuit | ~30 s |
| **Database** | Pool usage, replication lag | > 80% pool, > 10 s lag | Increase pool, promote replica, read-only mode | ~5 min |
| **Kafka** | Consumer lag, ISR shrink | > 10k lag, ISR < min | Add consumers, reassign partitions | ~3 min |
| **External API** | Timeout rate, circuit state | > 10% timeout, circuit open | Fallback, cache stale if safe | ~1 min |
| **Region** | AZ health, replication status | > 50% AZ impacted | Cross-region failover | ~15 min |

---

## ✅ Production Checklist

### Pre-Failure (Design)

- Multi-AZ deployment for all critical services
- Database read replicas for failover
- Kafka replication factor ≥ 3
- Circuit breakers for all external calls
- Timeouts configured (connection, read, write)
- Retry with exponential backoff
- Idempotent consumers for message processing
- Graceful degradation paths defined

### During Failure (Detection)

- P99 latency dashboards per service
- Error rate alerts (< 1% threshold)
- Resource utilization monitors (> 80% CPU / memory)
- Business metric monitors (e.g. orders per minute)
- Synthetic monitoring for critical paths

### Post-Failure (Recovery)

- Automated failover tested quarterly
- Chaos engineering experiments weekly
- Incident post-mortem within 48 hours
- Runbooks for common failure scenarios
- Disaster recovery drill annually

---

## 🎯 The 30-Second Explanation

> *"Distributed systems fail in predictable patterns. At the edge, DNS and load balancers can become single points of failure. In the application layer, threads deadlock, memory leaks, and GC pauses kill performance. The data layer suffers from cache stampedes, replication lag, and hot partitions. Message queues get poison pills and consumer lag. External dependencies time out and trip circuits. Infrastructure loses nodes, zones, and entire regions. Each layer needs specific resilience patterns: circuit breakers for external calls, bulkheads for resource isolation, retries for transient failures, and multi-AZ for infrastructure. The key isn't preventing failure—it's designing systems that detect, contain, and recover from failure automatically."*

---

## 🔗 Connecting to Previous Days

| Day | Concept | How It Connects |
|-----|---------|-----------------|
| Day 6 | Design for failure | Mindset; this day is the full catalog |
| Day 9 | Bloom filters | Cache penetration |
| Day 10 | Request coalescing | Cache stampede |
| Day 13 | Circuit breaker | External + gateway failures |
| Day 22 | 7-layer HLD | Map each layer to failure modes |
| Day 34 | Kafka rebalancing | Message layer — rebalance row |

---

## ✅ Day 35 Action Items

1. Pick one service and map its top five failure modes across layers.
2. Align alerts in the detection matrix with your real dashboards.
3. Reuse the palette in your next HLD diagram for consistent storytelling.

---

*— Sunchit Dudeja*  
*Day 35 of 50: System Design Interview Preparation Series*
