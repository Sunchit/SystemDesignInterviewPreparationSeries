# What Can Go Wrong, Why It Happens, and How to Survive It
### Day 35 of 50 - System Design Interview Preparation Series

**By Sunchit Dudeja**

---

## 📖 Introduction

You've built a distributed system. It's beautiful. Services communicate via APIs, data flows through message queues, and everything scales horizontally. Then something fails. Not a simple "server down" failure. Something far more insidious.

In distributed systems, **failures aren't exceptions—they are the norm.** The difference between a junior engineer and a seasoned architect isn't preventing failures; it's designing systems that **detect, contain, and recover** from them automatically.

This guide walks through the **complete failure landscape** of distributed systems, layer by layer, with real-world scenarios and battle-tested mitigation strategies.

---

## 🌍 Layer 1: Client & Edge

### 1.1 DNS Failure

**What happens:** Your application is running, but users can't reach it. "This site can't be reached."

**Root causes:**

- DNS provider outage (e.g., AWS Route 53, Cloudflare outage)
- TTL (Time To Live) misconfiguration causing stale records
- DNSSEC validation failures
- Cache poisoning attacks

**Real-world example:** In 2016, a major DNS provider outage took down Spotify, Twitter, Netflix, and dozens of other services for hours. Users couldn't resolve domain names, so their applications were unreachable even though the underlying infrastructure was healthy.

**Mitigation strategies:**

- **Multi-provider DNS:** Use two independent DNS providers simultaneously
- **Short TTLs:** Set TTL to 60 seconds or less for production records
- **Health checks:** Configure DNS-based failover to secondary IPs
- **Client-side fallback:** Implement retry logic with alternative DNS servers

---

### 1.2 CDN Failure

**What happens:** Static assets (images, CSS, JavaScript) fail to load. The page looks broken, but dynamic content works.

**Root causes:**

- Edge node failure in a region
- Cache invalidation storm (massive simultaneous purges)
- SSL certificate expiration
- Origin fetch timeouts

**Real-world example:** During a major sporting event, a CDN provider's edge nodes became overwhelmed. Users saw broken images and unstyled pages, leading to a 40% drop in conversion.

**Mitigation strategies:**

- **Multi-CDN:** Serve assets from two CDN providers with fallback
- **Origin shield:** Use a tiered caching architecture to reduce origin load
- **Cache warming:** Pre-populate caches before expected traffic spikes
- **Graceful degradation:** Design pages to function without images or non-critical assets

---

### 1.3 Load Balancer Failure

**What happens:** Traffic stops flowing to backend servers. Users see 502/503 errors or timeouts.

**Root causes:**

- Health check misconfiguration (marking healthy servers as unhealthy)
- Sticky session table overflow
- Connection pool exhaustion
- Certificate expiration on SSL termination

**Mitigation strategies:**

- **Redundant load balancers:** Deploy in active-passive or active-active pairs
- **Proper health checks:** Use multiple health check endpoints with appropriate thresholds
- **Connection draining:** Gracefully remove servers from rotation
- **Cross-zone load balancing:** Distribute traffic across availability zones

---

### 1.4 API Gateway Failure

**What happens:** Authentication fails, requests are misrouted, or rate limits incorrectly block legitimate traffic.

**Root causes:**

- Auth service timeouts
- Rate limit misconfiguration (too aggressive or too lenient)
- Routing table corruption
- Circuit breaker stuck in open state

**Mitigation strategies:**

- **Local token cache:** Cache authentication tokens to survive auth service outages
- **Circuit breaker recovery:** Implement half-open state testing
- **Canary deployments:** Test routing changes on a small traffic percentage
- **Fallback routes:** Define alternative paths when primary services fail

---

## 💻 Layer 2: Application

### 2.1 Thread Deadlock

**What happens:** Requests hang indefinitely. Thread pools fill up. New requests queue and eventually fail.

**Root causes:**

- Circular lock dependencies (Thread A holds Lock 1, waits for Lock 2; Thread B holds Lock 2, waits for Lock 1)
- Improper synchronization in async code
- Waiting on resources that never release

**Classic deadlock example (Java):**

```java
public class TransferService {
    private final Object lock1 = new Object();
    private final Object lock2 = new Object();

    public void transferAtoB() {
        synchronized (lock1) {
            try { Thread.sleep(100); } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
            synchronized (lock2) {
                // transfer
            }
        }
    }

    public void transferBtoA() {
        synchronized (lock2) {
            try { Thread.sleep(100); } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
            synchronized (lock1) {
                // transfer
            }
        }
    }
}
```

**Detection:**

- Thread dumps showing threads in `BLOCKED` or `WAITING` state
- Increasing request latency
- Decreasing throughput

**Mitigation:**

- **Lock ordering:** Always acquire locks in the same order
- **Lock timeouts:** Use `tryLock(timeout)` instead of `synchronized` where possible
- **Timeout interrupts:** Cancel long-running operations
- **Thread dump automation:** Capture thread dumps automatically on high latency

---

### 2.2 Memory Leak

**What happens:** Memory usage grows indefinitely. Eventually, the process hits its limit and is killed by the OOM killer (Out of Memory).

**Root causes:**

- Unclosed resources (database connections, file handles, streams)
- Static collections that grow unbounded
- Listener registrations without corresponding unregistration
- `ThreadLocal` variables not cleared

**Real-world example:** A popular logging library used `ThreadLocal` to store request context but never cleared it in thread pool environments. After a few hours, each thread held references to megabytes of data. The application crashed every 4 hours until the root cause was identified.

**Detection:**

- Gradual memory increase in monitoring dashboards
- Increasing GC frequency and duration
- OOM kill events

**Mitigation:**

- **Heap dump analysis:** Capture and analyze heap dumps on OOM
- **Bounded data structures:** Use `LRUCache` or `ConcurrentHashMap` with eviction
- **Try-with-resources:** Auto-close resources in Java
- **Weak references:** Use `WeakHashMap` for caches where appropriate

---

### 2.3 GC Pause

**What happens:** The application freezes for seconds or minutes while garbage collection runs.

**Root causes:**

- Large heap size (> 32 GB) with stop-the-world GC
- Frequent full GC cycles
- Too many objects promoted to old generation
- Memory fragmentation

**Real-world example:** A trading platform using a 64 GB heap with Parallel GC experienced 12-second pauses during market open. The system missed price updates and lost trades.

**Mitigation:**

- **GC tuning:** Switch to G1 GC or Shenandoah for large heaps
- **Smaller heaps:** Use multiple instances with smaller heaps
- **Object pooling:** Reuse objects instead of allocating new ones (where it helps; don't over-pool)
- **Off-heap storage:** Store large data outside GC-managed memory when appropriate

```bash
# Java GC optimization flags (example — tune for your workload)
-XX:+UseG1GC
-XX:MaxGCPauseMillis=200
-XX:InitiatingHeapOccupancyPercent=35
-XX:+ParallelRefProcEnabled
-Xmx8g -Xms8g
```

---

### 2.4 CPU Throttling

**What happens:** Processing slows down as CPU usage hits limits. Request latency increases.

**Root causes:**

- Inefficient algorithms (O(n²) in tight loops)
- CPU contention from noisy neighbors in shared environments
- Excessive logging or serialization
- CPU limits set too low in container orchestration

**Detection:**

- CPU usage > 80% sustained
- Increasing request latency while throughput remains constant
- Throttling metrics in Kubernetes (`container_cpu_cfs_throttled_seconds_total`)

**Mitigation:**

- **Profiling:** Use async-profiler or similar to identify hot paths
- **Compute offloading:** Move CPU-intensive work to background workers
- **Algorithm optimization:** Replace O(n²) with O(n log n) where possible
- **Resource requests:** Set appropriate CPU requests in Kubernetes

---

## 💾 Layer 3: Data

### 3.1 Cache Penetration

**What happens:** Massive requests for non-existent keys hit the database directly, overwhelming it.

**Root causes:**

- Malicious requests for random IDs
- Application code that doesn't cache negative results
- Cache key design that doesn't cover all access patterns

**Vulnerable pattern (Python-style pseudocode):**

```python
def get_user(user_id):
    user = cache.get(f"user:{user_id}")
    if not user:
        # Every invalid user_id hits the database!
        user = db.query("SELECT * FROM users WHERE id = ?", user_id)
        if user:
            cache.set(f"user:{user_id}", user)
    return user
```

**Detection:**

- Database query rate spikes not correlated with expected cache miss rate
- High CPU on database servers
- Large number of queries returning empty results

**Mitigation:**

- **Bloom filter:** Pre-filter requests for keys that definitely don't exist
- **Cache empty values:** Store null markers with short TTL
- **Rate limiting:** Limit requests for non-existent keys per IP or tenant

**Mitigated pattern:**

```python
BLOOM_FILTER = BloomFilter(1_000_000_000, 0.01)  # Pre-populated with valid IDs

def get_user(user_id):
    if not BLOOM_FILTER.maybe_contains(user_id):
        return None  # Definitely doesn't exist

    user = cache.get(f"user:{user_id}")
    if user is None:
        user = db.query("SELECT * FROM users WHERE id = ?", user_id)
        ttl = 300 if user else 60  # Cache negative results shorter
        cache.set(f"user:{user_id}", user, ttl)
    return user
```

---

### 3.2 Cache Stampede (Thundering Herd)

**What happens:** When a popular cache key expires, thousands of simultaneous requests try to recompute the value at once, overwhelming the database.

**Root causes:**

- Synchronous cache expiration
- No request coalescing
- All requests seeing cache miss at the same time

**Real-world example:** A popular e-commerce product page had a 5-minute cache TTL. At 2 PM exactly, all 10,000 concurrent users' requests hit the database simultaneously. The database crashed.

**Mitigation:**

- **Request coalescing:** Only one request recomputes; others wait
- **Probabilistic early expiration:** Refresh cache before it expires
- **Jitter:** Add randomness to TTL values

```java
// Request coalescing with CompletableFuture
private final ConcurrentMap<String, CompletableFuture<Product>> inFlight = new ConcurrentHashMap<>();

public CompletableFuture<Product> getProduct(String id) {
    return inFlight.computeIfAbsent(id, key ->
        CompletableFuture.supplyAsync(() -> {
            try {
                Product product = database.loadProduct(key);
                cache.set(key, product, 300);
                return product;
            } finally {
                inFlight.remove(key);
            }
        })
    );
}
```

---

### 3.3 Replication Lag

**What happens:** Read replicas return stale data. Users see inconsistent state.

**Root causes:**

- Slow replica processing
- Network latency between primary and replicas
- Large write transactions
- Replica server resource constraints

**Real-world example:** After a user updated their profile picture, they saw the old picture for 30 seconds. Other users saw the new picture instantly. The inconsistency broke trust in the application.

**Mitigation:**

- **Monitor lag:** Alert when replication lag exceeds threshold
- **Read-your-writes consistency:** Route reads from the primary for the user's own data
- **Session consistency:** Use the same replica for a user's session where applicable
- **Read-only transactions:** Offload analytics to replicas, not latency-sensitive user reads

---

### 3.4 Hot Partition

**What happens:** One shard receives disproportionate traffic while others sit idle. The hot shard becomes a bottleneck.

**Root causes:**

- Poor sharding key choice (e.g., sharding by `user_id` but one user generates 50% of traffic)
- Time-series data (latest partition always hot)
- Celebrity effect (one entity has orders of magnitude more data)

**Real-world example:** A social media platform sharded by `user_id`. A celebrity with 50 million followers generated 80% of all write traffic. Their shard was overloaded while others sat at 5% capacity.

**Mitigation:**

- **Consistent hashing:** Distribute load more evenly
- **Shard splitting:** Split hot shards into smaller shards
- **Write sharding:** Add a "hot shard" layer for high-volume entities
- **Pre-splitting:** Create more shards than initially needed to avoid painful rebalancing

---

### 3.5 Connection Pool Exhaustion

**What happens:** New requests wait for database connections that never become available. Threads pile up, eventually exhausting application threads.

**Root causes:**

- Leaked connections (not closed after use)
- Connection pool size too small for traffic spikes
- Long-running transactions holding connections
- Database server connection limit reached

**Detection:**

- `java.sql.SQLException: Connection pool exhausted`
- Increasing number of threads waiting in `getConnection()`
- Database showing high number of active connections

**Mitigation:**

- **Proper connection release:** Use try-with-resources
- **Monitoring:** Track pool utilization and wait times
- **Leak detection:** Configure the pool to log or alert on long-held connections
- **Dynamic sizing:** Adjust pool size based on load (with upper bounds)

```yaml
# HikariCP-style production configuration (Spring)
spring.datasource.hikari:
  maximum-pool-size: 20
  minimum-idle: 10
  connection-timeout: 30000
  idle-timeout: 600000
  max-lifetime: 1800000
  leak-detection-threshold: 60000  # connections held > 60s
```

---

### 3.6 Database Deadlock

**What happens:** Two transactions block each other, waiting indefinitely.

**Root causes:**

- Transactions updating tables in different orders
- Range locks conflicting
- Long-running transactions

**Detection:**

- Deadlock exception logs
- Increasing transaction durations
- Application timeouts

**Mitigation:**

- **Retry logic:** Automatically retry deadlocked transactions (with limits)
- **Lock ordering:** Always acquire locks in the same order
- **Short transactions:** Keep transaction scopes small
- **Optimistic locking:** Use version numbers instead of heavy row locks for high-contention scenarios

---

## 📨 Layer 4: Messaging

### 4.1 Consumer Lag

**What happens:** Messages accumulate in Kafka topics faster than consumers can process them.

**Root causes:**

- Slow consumer processing
- Under-provisioned consumers
- Partition imbalance
- Schema evolution causing processing errors

**Detection:**

- Consumer lag > threshold
- Growing backlog in monitoring dashboards
- Increasing end-to-end latency

**Mitigation:**

- **Auto-scaling consumers:** Add consumers when lag increases
- **Partition increase:** More partitions = more parallelism (planned carefully)
- **Consumer tuning:** Adjust `max.poll.records`, optimize processing
- **Backpressure handling:** Slow producers when consumers lag (if architecture supports it)

---

### 4.2 Rebalance Storm

**What happens:** Consumers constantly join and leave the consumer group, triggering repeated rebalances. No one processes messages reliably.

**Root causes:**

- Network instability causing heartbeats to drop
- Processing time exceeding `max.poll.interval.ms`
- Session timeout misconfiguration
- Frequent deployment/restart cycles

**Mitigation:**

- **Increase timeouts:** Tune `session.timeout.ms` and `max.poll.interval.ms` for your processing time
- **Sticky assignor:** Minimize partition movement during rebalances
- **Cooperative rebalancing:** Only revoke necessary partitions
- **KIP-848:** New consumer protocol on Kafka 4.0+ for faster, targeted rebalances

```properties
# Example consumer tuning
max.poll.interval.ms=300000
session.timeout.ms=45000
heartbeat.interval.ms=3000
partition.assignment.strategy=org.apache.kafka.clients.consumer.StickyAssignor
```

---

### 4.3 Poison Pill Message

**What happens:** A single malformed message causes the consumer to crash or retry forever. The consumer never progresses past that offset.

**Root causes:**

- Schema mismatch (producer wrote new version, consumer expects old)
- Corrupted data
- Processing logic that can't handle certain values
- External dependency failure for specific messages

**Mitigation:**

- **Dead letter queue (DLQ):** Send unprocessable messages to a DLQ for inspection
- **Error boundaries:** Catch exceptions, log, and skip or DLQ with policy
- **Schema validation:** Validate messages before processing
- **Manual skip:** Admin tooling to skip a problematic offset when safe

```java
// Illustrative: DLQ + skip after failure (polish for your exactly-once / ordering needs)
while (true) {
    var records = consumer.poll(Duration.ofSeconds(1));
    for (var record : records) {
        try {
            process(record);
        } catch (Exception e) {
            deadLetterQueue.send(record);
            // Seek past bad offset only if ordering allows; often prefer DLQ + fixed handler
        }
    }
    consumer.commitSync();
}
```

---

## 🔌 Layer 5: External Dependencies

### 5.1 Timeout Cascade

**What happens:** One slow external service causes threads to block. The thread pool exhausts, causing cascading failures upstream.

**Root causes:**

- External service degradation
- Network latency spikes
- Missing or too-long timeouts
- Unlimited retries

**Mitigation:**

- **Short timeouts:** Set connection and read timeouts shorter than caller patience
- **Circuit breaker:** Fail fast when the service is unhealthy
- **Bulkhead:** Isolate external calls in dedicated thread pools
- **Timeout propagation:** Pass deadline headers downstream (e.g., `X-Request-Deadline`)

```java
// Resilience4j-style circuit breaker (conceptual)
@CircuitBreaker(name = "payment-service", fallbackMethod = "paymentFallback")
public PaymentResponse processPayment(PaymentRequest request) {
    return paymentClient.charge(request);
}

public PaymentResponse paymentFallback(PaymentRequest request, Exception e) {
    return PaymentResponse.pending("Payment queued for retry");
}
```

---

### 5.2 Rate Limit Exhaustion

**What happens:** API calls to external services start failing with **429 Too Many Requests**.

**Root causes:**

- Traffic spike exceeding quota
- No client-side rate limiting
- Shared quota across services

**Mitigation:**

- **Token bucket:** Implement client-side rate limiting
- **Request queuing:** Queue requests when approaching limit
- **Caching:** Cache responses to reduce API calls
- **Quota negotiation:** Purchase higher limits for critical services

---

## 🏗️ Layer 6: Infrastructure

### 6.1 Node Failure

**What happens:** A Kubernetes node goes down. All pods on that node are lost.

**Root causes:**

- Hardware failure (CPU, memory, disk)
- Kernel panic
- OOM kill of kubelet
- Network partition

**Mitigation:**

- **PodDisruptionBudget:** Ensure enough replicas remain during voluntary disruption
- **Topology spread constraints:** Distribute pods across nodes
- **Node pools:** Use spot instances only for non-critical workloads
- **Auto-recovery:** Kubernetes reschedules pods onto healthy nodes

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: app-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: critical-service
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: critical-service
spec:
  replicas: 3
  template:
    spec:
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: critical-service
```

---

### 6.2 AZ Outage

**What happens:** An entire availability zone loses power, network, or cooling.

**Real-world example:** In 2021, a power failure in a US-East availability zone took down thousands of services. Companies with single-AZ deployments suffered hours of downtime. Those with multi-AZ deployments saw minutes of disruption.

**Mitigation:**

- **Multi-AZ deployment:** Distribute instances across multiple availability zones
- **Load balancer cross-zone:** Enable cross-zone load balancing
- **Database multi-AZ:** Use synchronous or managed multi-AZ replication
- **Stateful sets:** Use persistent volumes with zone-aware provisioning where needed

---

### 6.3 Region Outage

**What happens:** An entire AWS/GCP/Azure region becomes unavailable.

**Real-world example:** When an entire cloud region experienced a major outage, companies without cross-region failover were down for 12+ hours. Customer trust was severely damaged.

**Mitigation:**

- **Cross-region replication:** Replicate data to a secondary region
- **Active-active:** Serve traffic from multiple regions when feasible
- **DNS failover:** Route traffic to a healthy region
- **Regular drills:** Test region failover at least annually

---

## 📊 Layer 7: Observability

### 7.1 Monitoring Blackout

**What happens:** During the outage, monitoring systems fail too. You're blind.

**Root causes:**

- Monitoring in the same region as the failure
- Monitoring infrastructure without redundancy
- Logs lost during the failure

**Mitigation:**

- **Separate monitoring infrastructure:** Use a different provider or region from primary workloads
- **Independent alerting:** Use third-party alerting (PagerDuty, Opsgenie, etc.)
- **Log shipping:** Send logs to multiple destinations
- **Chaos testing:** Validate that alerts still fire during failure drills

---

## 🛡️ The Resilience Patterns Summary

| Layer | Key patterns | Purpose |
|-------|--------------|---------|
| **Edge** | DNS failover, CDN fallback, LB redundancy | Prevent single point of failure at entry |
| **Application** | Circuit breakers, bulkheads, retry with backoff | Contain failures, prevent cascades |
| **Data** | Read replicas, sharding, cache warming | Distribute load, survive node loss |
| **Messaging** | DLQ, idempotent consumers, rebalance tuning | Prevent data loss, handle poison messages |
| **Infrastructure** | Multi-AZ, cross-region, PDB | Survive zone and region failures |
| **Observability** | Redundant monitoring, synthetic tests | See failures when they happen |

---

## 🎯 Final Thoughts

Building resilient distributed systems isn't about preventing failures—it's about designing systems that:

1. **Expect failure:** Every component will fail. Design for it.
2. **Detect failure:** Know when something is wrong before users notice.
3. **Contain failure:** Prevent a single failure from cascading.
4. **Recover automatically:** Self-healing beats manual intervention.
5. **Learn from failure:** Post-mortems drive improvement.

**The golden rule:** A system that never fails is impossible. A system that **fails gracefully** and **recovers automatically** is what separates production-grade architecture from prototypes.

This guide is based on real incidents, post-mortems, and patterns observed across thousands of production systems. Use it to design, review, and operate systems that survive the chaos of production.

---

## 🔗 Connecting to Previous Days

| Day | Concept | How It Connects |
|-----|---------|-----------------|
| Day 6 | Design for failure | Mindset—this day is the full layer-by-layer catalog |
| Day 9 | Bloom filters | Cache penetration (Layer 3) |
| Day 10 | Request coalescing | Cache stampede (Layer 3) |
| Day 13 | Circuit breaker | Timeout cascade & gateways (Layers 1 & 5) |
| Day 22 | 7-layer HLD | Map failures to your HLD layers |
| Day 34 | Kafka rebalancing | Rebalance storm (Layer 4) |

---

## ✅ Day 35 Action Items

1. Pick **one critical user journey** and walk it through Layers 1–7: what breaks first?
2. For your top external dependency, document **timeouts, retries, circuit breaker**, and who gets paged on 429 vs timeout.
3. Verify **monitoring** runs out-of-band from your primary region (Layer 7).
4. Add **TTL jitter** on one hot cache key and confirm DB load smooths out.

---

*— Sunchit Dudeja*  
*Day 35 of 50: System Design Interview Preparation Series*
