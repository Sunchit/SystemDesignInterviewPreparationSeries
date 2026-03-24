# 🏛️ Distributed Systems Failure Modes — Complete HLD Architecture
### Day 35 of 50 - System Design Interview Preparation Series

**By Sunchit Dudeja**

---

## 🎯 The Comprehensive Failure Landscape

Distributed failures stack from **edge** through **application**, **data**, **messaging**, **external dependencies**, and **infrastructure**. Each layer has predictable symptoms, root causes, and mitigations. Use the **dark warm palette** below for slides, diagrams, and HLD drawings so layers stay visually distinct without neon brights.

**Reality check:** One layer rarely fails in isolation. A slow external API can exhaust connection pools in the app tier, which makes health checks flap at the load balancer, which triggers thundering herds against cache and DB. Good HLD names **blast radius**, **dependencies**, and **what degrades first**—not only “happy path” scale.

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

## 🔗 Cascading Failures: How One Layer Pulls Down the Next

| Stage | What users see | What’s often happening under the hood |
|-------|----------------|----------------------------------------|
| 1 | Spikes in latency | Upstream dependency slow; threads or event-loop tasks pile up |
| 2 | Errors climb | Timeouts fire; retries multiply load on already-sick dependencies (**retry storm**) |
| 3 | Pool / queue saturation | DB or HTTP client pools exhausted; Kafka consumer `max.poll.interval` blown |
| 4 | Health checks fail | LB marks instances bad; remaining nodes take **more** traffic (**metastable failure**) |
| 5 | Data path stress | Cache stampedes, replica lag, or hot keys amplify the collapse |

**Design takeaway:** Combine **timeouts shorter than client patience**, **bounded retries** (with jitter), **bulkheads** so one dependency can’t starve the whole JVM, and **backpressure** (queue limits, shed load at gateway) so the system **fails partially** instead of dying everywhere at once.

---

## 🧠 Failure Taxonomy (How Architects Classify Incidents)

| Type | Meaning | Example | Typical response |
|------|---------|---------|------------------|
| **Transient** | Likely succeeds on retry | Packet loss, brief GC pause, 503 from rolling deploy | Retry with backoff + jitter; no schema change |
| **Permanent** | Retry won’t help until fixed | Bad payload, auth revoked, constraint violation | Fail fast; DLQ; alert; fix data or code |
| **Fail-stop** | Component halts cleanly | Process crash, pod SIGKILL | Restart, failover, drain connections |
| **Degraded** | Still serves some traffic | Read-only mode, feature flag off, stale cache | Graceful degradation; communicate to users |
| **Correlated** | Many nodes fail together | AZ outage, bad config rollout, bad dependency release | Blast-radius control; canary; multi-AZ |

Interview tip: Always say whether a failure is **transient vs permanent** before arguing for “retry everything” or “never retry.”

---

## 📐 HLD Review: Failure-Oriented Questions

Ask these while reviewing or sketching an architecture (document cloud, e-commerce, or internal platforms):

- **Single points of failure:** Where is the only copy of truth? The only region? The only vendor?
- **Dependency depth:** If payment is down, do we block **checkout** only or the **whole site**?
- **Data path:** Which reads are allowed to be stale? Which writes must be **strongly consistent**?
- **Messaging:** At-least-once vs exactly-once semantics—what breaks if we process twice?
- **Operational:** Who gets paged for **consumer lag** vs **DB lag** vs **external 429**?
- **Recovery:** RTO/RPO for this flow—minutes, hours, or “best effort”?

---

## 🔢 Operational Numbers (Starting Points, Tune to SLOs)

These are **defaults to argue from**, not universal laws—always align with your latency SLO and client timeouts.

| Concern | Rule of thumb | Why it matters |
|---------|---------------|----------------|
| Client → service timeout | ≤ user-facing SLA (e.g. 2–5 s for APIs) | Avoid hung UIs and thread pileups |
| Service → dependency | Shorter than caller timeout (nested budgets) | Inner call must fail before outer gives up |
| Retries | Cap count (e.g. 2–3); **always jitter** | Prevents synchronized retry storms |
| Connection pools | Size from measured concurrency, not “max” | Too large = DB meltdown; too small = artificial bottleneck |
| Health checks | Liveness vs readiness split | Don’t kill pod for slow dependency if it’s still “alive” |
| Cache TTL | Add **jitter** (e.g. ±10%) | Reduces synchronized expiry (stampede) |
| Kafka consumers | `max.poll.interval` vs processing time | Long handlers need tuning or async processing |

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
| **Saga / process manager** | Data, Message, External | Long workflows with compensating actions when a step fails |
| **Outbox / transactional messaging** | Data, Message | Avoid “DB committed but event never sent” dual-write failures |
| **Leader election + epoch** | Data, Infrastructure | Clear winner after partition; stale primaries step down |
| **Graceful shutdown** | Application, Message | `SIGTERM` handling; drain HTTP; commit offsets before exit |
| **Feature flags / kill switch** | Application, Edge | Instantly disable a bad path without full deploy |

Patterns stack: e.g. **timeout + circuit breaker + bulkhead + idempotency** is a common production bundle for payment and inventory calls.

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
| **Cache (Redis)** | Hit ratio, memory, evictions, latency | Hit ratio drop > 10 pts; P99 > 2× baseline | Scale cluster, fix hot keys, TTL review | ~5 min |
| **API Gateway** | Auth latency, 429 rate, route errors | Auth P99 > 500 ms; 429 > 5% sustained | Scale gateway pods, relax mis-tuned limits, cache JWKS | ~3 min |
| **CDN / Edge** | Origin fetch rate, 5xx at edge | Origin load spike; edge 5xx > 1% | Stale-while-revalidate, widen cache, second origin | ~10 min |

---

## 💼 Example: Document Cloud Bulk Export (Cross-Layer)

| Layer | Failure you might hit | Mitigation in design |
|-------|----------------------|----------------------|
| Edge | Large download timeouts | Signed URLs, chunked export, async “notify when ready” |
| App | Long CPU for zip/PDF | Queue job, worker pool, separate scaling from API |
| Data | Hot metadata row for “folder” | Shard listing, cache folder trees with TTL + coalescing |
| Message | Export job backlog | Dedicated topic/partitions, consumer autoscaling on lag |
| External | Virus scan / preview API slow | Async pipeline, circuit breaker, optional skip with audit |
| Infra | AZ loss | Multi-AZ object store + DB replicas; job state replicated |

Use this as a template: pick **your** feature and walk the same six layers in an interview.

---

## 🧪 Chaos, Load, and Error Budgets

- **Load testing:** Prove behavior at **2× expected peak**—watch pools, GC, and dependency timeouts, not just HTTP 200s.
- **Fault injection:** Kill pods, inject latency on one dependency, shrink Kafka ISR in a **non-prod** cluster—verify alerts and runbooks.
- **Game days:** Scripted scenarios (DNS fail, replica lag, payment down) with observers scoring detection and recovery time.
- **Error budget:** If availability SLO is 99.9%, you have ~43 minutes downtime/month—**budget** retries, deploys, and experiments so routine work doesn’t burn it all on one cascade.

---

## 📎 Runbook Skeleton (Paste into Wiki / Notion)

```yaml
Scenario: [e.g. Kafka consumer lag on export-jobs]
Severity: [P1/P2]
Symptoms: [dashboards, alerts]
Blast radius: [which tenants / regions]
Immediate checks:
  - Consumer group lag, partition assignment
  - Recent deploys, config changes
  - Dependency health (DB, object store)
Mitigation:
  - Scale consumers / add partitions (if planned)
  - Pause poison source; DLQ inspection
Communication: [status page, internal channel]
Post-incident: [link to postmortem template]
```

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
| Day 25 | Deployment strategies | Bad rollout = correlated failure across instances |
| Day 30 | Database replication | Replication lag, failover, read-after-write |

---

## ✅ Day 35 Action Items

1. Pick one service and map its top five failure modes across layers.
2. Align alerts in the detection matrix with your real dashboards.
3. Reuse the palette in your next HLD diagram for consistent storytelling.
4. Write one runbook from the skeleton for your highest-traffic dependency.
5. Run one chaos or load experiment this month and fix the **first** gap you find (timeout, pool size, or missing alert).

---

*— Sunchit Dudeja*  
*Day 35 of 50: System Design Interview Preparation Series*
