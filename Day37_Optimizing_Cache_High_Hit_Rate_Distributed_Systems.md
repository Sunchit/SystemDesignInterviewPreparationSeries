# Optimizing Cache for High Hit Rate in Distributed Systems
### Day 37 of 50 - System Design Interview Preparation Series

**By Sunchit Dudeja**

---

## 🎯 The Goal: Keep Your Cache Hot

**Cache hit rate** is the percentage of requests served from cache rather than the backend (database, origin, etc.). A **90% hit rate** means 90% of reads never touch your database—saving latency, cost, and compute.

| Level | Hit rate | Notes |
|-------|----------|--------|
| **Good** | 80–90% | Common for mixed workloads |
| **Great** | 90–95% | Mature tuning |
| **Excellent** | 95–99% | Skewed read-heavy, stable keys |
| **Near-perfect** | 99%+ | Rare; often read-only or heavily precomputed datasets |

---

## 📊 The Hit Rate Formula

```
Hit Rate = Cache Hits / (Cache Hits + Cache Misses) × 100%
```

**Example:**

- 9,000 requests served from cache  
- 1,000 requests hit the database (misses)  
- Hit rate = 9,000 / 10,000 = **90%**

**Why a few points matter:** If miss rate rises from **10% → 20%** (hit rate 90% → 80%), **database load from cache misses doubles** for the same request volume. Missed paths also pay **much higher latency** than hits—often **10–100×** depending on your stack.

---

## 🔥 The 7 Laws of Cache Optimization

### Law 1: Size Your Cache Correctly

Hit rate vs cache size often follows a **diminishing-returns** curve: a large share of benefit comes from caching a **fraction** of the data you actually touch—the **working set**, not necessarily the full dataset.

**Working set** = data actively accessed in a time window (e.g. last hour).

```text
Example:
  Total dataset: 1 TB
  Data accessed in the last hour: 100 GB
  Working set ≈ 100 GB  ← size and tune for this, not the full 1 TB
```

Illustrative mapping (workload-dependent):

| Cache holds (of working set) | Typical hit rate band (illustrative) |
|------------------------------|--------------------------------------|
| ~10% | ~50% |
| ~20% | ~75% |
| ~33% | ~85% |
| ~50% | ~92% |
| ~100% | ~99% |

**Rule of thumb:** Target **20–50% of the working set** in cache for many workloads to reach **90%+** hits—validate with metrics, not guesses.

---

### Law 2: Choose the Right Eviction Policy

| Policy | How it works | Best for | Hit rate (typical) |
|--------|--------------|----------|--------------------|
| **LRU** | Evict least recently used | General purpose | High |
| **LFU** | Evict least frequently used | Popularity skew, “viral” keys | Very high |
| **TTL** | Evict after fixed time | Freshness-bound data | Depends on churn |
| **FIFO** | Evict oldest inserted | Simple / rare choice | Moderate |
| **ARC** | Adaptive LRU/LFU blend | Mixed access patterns | Very high |
| **2Q** | Two-queue LRU variant | Scan-resistant workloads | High |

**Production starting point:** **LRU** by default; move to **LFU** when access is heavily popularity-skewed; consider **ARC** (or vendor equivalents) for mixed workloads.

**Redis LFU example (illustrative):**

```redis
CONFIG SET maxmemory-policy allkeys-lfu
CONFIG SET lfu-log-factor 10
CONFIG SET lfu-decay-time 1
```

Tune `lfu-log-factor` and decay with your traffic; test under load.

---

### Law 3: Implement Cache Warming

**Problem:** Cold cache → every request misses until populated.

**A. Pre-warm at startup**

```python
def warm_cache():
    """Load hottest items after deploy (example)."""
    popular_items = db.query("""
        SELECT id, data FROM products
        ORDER BY last_accessed DESC
        LIMIT 10000
    """)
    for item in popular_items:
        cache.set(f"product:{item.id}", item.data, ttl=3600)
```

**B. Background warming (scheduled)**

```python
def background_warm():
    hot_keys = predict_hot_keys_for_next_hour()  # analytics / ML / heuristics
    for key in hot_keys:
        if not cache.exists(key):
            value = db.get(key)
            cache.set(key, value, ttl=ttl_for(key))
```

**C. Probabilistic early refresh (reduce stampede near expiry)**

```python
def get_with_early_refresh(key, ttl=300):
    value = cache.get(key)
    if value is None:
        value = db.get(key)
        cache.set(key, value, ttl)
        return value
    ttl_remaining = cache.ttl(key)
    if ttl_remaining < ttl * 0.1:
        async_refresh(key)  # fire-and-forget; don’t block user path
    return value
```

Combine with **TTL jitter** so keys don’t expire at the same instant.

---

### Law 4: Request Coalescing (Stampede Prevention)

**Problem:** Many concurrent requests for the **same missing key** all hit the database.

**Before (bad):** 1,000 concurrent misses → 1,000 identical DB queries.

**After (good):** **Singleflight** / coalescing—one fetch; others wait for the same result.

```python
from concurrent.futures import Future
import threading

_in_flight: dict[str, Future] = {}
_coalescing_lock = threading.Lock()

def get_product(product_id: str):
    cache_key = f"product:{product_id}"

    cached = cache.get(cache_key)
    if cached is not None:
        return cached

    with _coalescing_lock:
        if cache_key in _in_flight:
            return _in_flight[cache_key].result()
        fut: Future = Future()
        _in_flight[cache_key] = fut

    try:
        value = db.query_product(product_id)
        cache.set(cache_key, value)
        fut.set_result(value)
        return value
    except Exception as e:
        fut.set_exception(e)
        raise
    finally:
        with _coalescing_lock:
            _in_flight.pop(cache_key, None)
```

In production, prefer proven **singleflight** libraries, **Redis SETNX**-style locks, or patterns from **Day 10**—this snippet shows the idea.

---

### Law 5: Optimize Key Design

**Bad (low reuse / low hit rate):**

```python
# Too volatile — rarely the same key twice
cache.set(f"user:{user_id}:order:{order_id}:ts:{timestamp}", data)

# Unbounded parameter surface — hard to reuse
cache.set(f"search:{query}:{page}:{country}", results)
```

**Better:**

```python
cache.set(f"user:{user_id}", user_blob)
cache.set(f"order:{order_id}", order_blob)

# Canonicalize expensive inputs
cache.set(f"search:v2:{stable_hash(query)}:p:{page}", results)

# Version in key for safe invalidation
cache.set(f"product:{product_id}:v{schema_version}", product_data)
```

**Rules:**

- Prefer **stable** key components; avoid random IDs in the key unless necessary.
- **Canonicalize** variable parts (normalize query, hash long inputs).
- Include **version** or **etag** when invalidation semantics matter.
- Keep keys **reasonably short** (memory and network).

---

### Law 6: Multi-Layer Caching

```python
class MultiLayerCache:
    """
    L1: Process-local (microseconds)
    L2: Redis / memcached (milliseconds)
    L3: Database (milliseconds–tens of ms)
    """

    def __init__(self):
        self.l1 = LRUCache(maxsize=1000)
        self.l2 = redis_client

    def get(self, key):
        v = self.l1.get(key)
        if v is not None:
            return v
        v = self.l2.get(key)
        if v is not None:
            self.l1.set(key, v)
            return v
        v = self.db.get(key)
        if v is not None:
            self.l2.set(key, v, ttl=300)
            self.l1.set(key, v)
        return v
```

**Typical pattern:** L1 improves hit rate for **hot keys** on each instance; L2 shares state across nodes. Expect **higher combined hit rate** than L2 alone—measure L1 effectiveness separately to avoid stale data bugs (TTL + invalidation strategy).

---

### Law 7: Partition and Replicate Hot Keys

**Problem:** One key is so hot it saturates a single shard, NIC, or CPU on one node.

**Mitigations:**

- **Redis Cluster / proxy:** Distribute keys across shards; still watch for **single-slot hot keys**.
- **Read replicas** or **replica reads** where the product supports it (consistency caveats).
- **Application replication:** Same logical key written to multiple nodes; reads spread (complex; use rarely).
- **Client-side / edge cache** for extreme QPS on a tiny key set (short TTL, careful invalidation).

```python
# Illustrative: spread reads across replicas (when architecture allows)
def get_hot(key):
    replica = random.choice(replicas)
    return replica.get(key)
```

For **>10k QPS** on a handful of keys, a **short TTL local cache** in the app can shave load from Redis—coordinate with correctness needs.

---

## 📈 Monitoring and Tuning

### Key metrics

| Metric | Healthy direction | If weak |
|--------|-------------------|---------|
| **Hit rate** | > 90% for many read caches | Grow working-set coverage, TTL, warming |
| **Miss rate** | Low and stable | Analyze miss keys; preload; fix key design |
| **Eviction rate** | Sustainable | More memory or better eviction / TTL |
| **Memory usage** | ~70–80% planned headroom | Scale out or trim TTL |
| **P99 cache latency** | Low ms | Network, colocation, replicas |
| **Stale reads** | Within policy | Shorter TTL, write-through / invalidation |

### Hit rate by data pattern

| Pattern | Expected band | Strategy |
|---------|---------------|----------|
| Read-heavy, stable | 95–99% | Longer TTL, warming |
| Read-heavy, churning | 80–90% | LFU, adaptive TTL, invalidation |
| Write-heavy | 60–80% | Write-through / write-behind, shorter TTL |
| Time-series | 70–85% | Windowing, partition by time |
| Viral / skewed | 85–95% | Hot-key mitigation, local cache |

---

## ✅ Production Checklist

### Design

- Estimate **working set** size and growth.
- Pick **eviction policy** (LRU → LFU/ARC as needed).
- Design **stable, canonical keys** and invalidation story.
- Plan **warming** (startup + background).

### Implementation

- Add **request coalescing** / singleflight for hot entities.
- Use **L1 + L2** where it pays; define consistency rules.
- Set **TTL per data class**; add **jitter** on expiry.
- Detect **hot keys**; plan replication / local cache if needed.

### Monitoring

- Track **hit rate** by key prefix or business domain.
- Alert on **sustained hit-rate drops** (e.g. >10% relative change).
- Watch **evictions** and **memory** pressure.
- Review **top miss keys** regularly.

### Tuning

- Adjust TTL from observed churn.
- Scale memory when evictions dominate.
- Add replicas or client cache for proven hot spots.
- Prune unused prefixes / experiments monthly.

---

## 🎯 The 30-Second Summary

> *"High cache hit rates come from sizing to the **working set**, picking the right **eviction** policy, **coalescing** identical misses to stop stampedes, **warming** before load spikes, and **designing keys** for reuse. Small hit-rate gains materially cut database load—measure hit rate by pattern and tune relentlessly."*

---

## 🔗 Connecting to Previous Days

| Day | Concept | How It Connects |
|-----|---------|-----------------|
| Day 9 | Bloom filters | Reduce misses on non-existent keys |
| Day 10 | Request coalescing | Stampede prevention (Law 4) |
| Day 35 | Failure modes | Cache penetration, stampede, hot partitions |

---

## ✅ Day 37 Action Items

1. Export **hit rate and top miss prefixes** for one production cache this week.
2. Add **TTL jitter** on one high-churn namespace and compare eviction graphs.
3. Document **working set** vs total dataset size for your largest cache.

---

*— Sunchit Dudeja*  
*Day 37 of 50: System Design Interview Preparation Series*
