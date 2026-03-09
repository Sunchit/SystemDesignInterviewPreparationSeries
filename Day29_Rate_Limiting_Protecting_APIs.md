# Rate Limiting: How Netflix, Stripe & GitHub Protect Their APIs from Abuse
### Day 29 of 50 - System Design Interview Preparation Series

**By Sunchit Dudeja**

---

## 🎯 Welcome to Day 29!

Yesterday, we mastered consistent hashing for resharding distributed systems. Today, we explore **Rate Limiting** — the critical technique that protects your APIs from abuse, ensures fair access, and keeps your systems from collapsing under malicious or accidental traffic spikes.

> Rate limiting isn't just about saying "no" to requests. It's about saying "yes" to the right requests while protecting your infrastructure and ensuring fairness.

---

## 🚨 THE PROBLEM: WHY RATE LIMITING MATTERS

### The DDoS That Almost Killed GitHub (2018)

```
┌─────────────────────────────────────────────────────────────────┐
│                 THE GITHUB INCIDENT                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   February 28, 2018:                                             │
│   • GitHub hit with 1.35 Tbps DDoS attack                       │
│   • Largest DDoS attack ever recorded at the time               │
│   • 126.9 million packets per second                            │
│                                                                  │
│   What saved them:                                               │
│   • Cloudflare's rate limiting and DDoS protection              │
│   • Traffic scrubbing at network edge                           │
│   • Automatic traffic filtering                                  │
│                                                                  │
│   Downtime: Only 10 minutes (vs potential hours/days)           │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Why Every API Needs Rate Limiting

| Threat | Without Rate Limiting | With Rate Limiting |
|--------|----------------------|-------------------|
| **DDoS Attacks** | System crashes | Malicious traffic blocked |
| **Buggy Client** | Infinite loop exhausts API | Client throttled, others unaffected |
| **Scraping Bots** | Database overwhelmed | Bots slowed/blocked |
| **Noisy Neighbor** | One user affects all | Fair resource allocation |
| **Cost Explosion** | Unlimited cloud bills | Controlled resource usage |

---

## 🎯 THE 5 RATE LIMITING ALGORITHMS

```
┌─────────────────────────────────────────────────────────────────┐
│                 ALGORITHM OVERVIEW                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   1. TOKEN BUCKET         → Most popular, bursty traffic OK     │
│   2. LEAKY BUCKET         → Smooth, constant output rate        │
│   3. FIXED WINDOW         → Simple, but has edge problem        │
│   4. SLIDING WINDOW LOG   → Accurate, but memory intensive      │
│   5. SLIDING WINDOW COUNTER → Best of both worlds              │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 🪣 ALGORITHM 1: TOKEN BUCKET

### The Water Bucket Analogy

```
┌─────────────────────────────────────────────────────────────────┐
│                 TOKEN BUCKET                                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Imagine a bucket that:                                         │
│   • Holds up to 10 tokens (bucket capacity)                     │
│   • Tokens added at rate of 1 per second (refill rate)          │
│   • Each request consumes 1 token                               │
│   • If bucket empty → request rejected                          │
│   • If bucket has tokens → request allowed                      │
│                                                                  │
│   ┌─────────┐                                                    │
│   │ ● ● ● ● │ ← Bucket with 4 tokens                            │
│   │ ● ● ● ● │                                                    │
│   │ ●   ●   │   Capacity: 10                                    │
│   └────┬────┘   Refill: 1/sec                                   │
│        │                                                         │
│        ▼                                                         │
│   [Request] → Takes 1 token → Allowed!                          │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Python Implementation

```python
import time
from threading import Lock

class TokenBucket:
    """
    Token Bucket Rate Limiter
    
    - Allows bursts up to bucket capacity
    - Smooths out traffic over time
    - Used by: AWS, Stripe, most cloud providers
    """
    
    def __init__(self, capacity: int, refill_rate: float):
        """
        Args:
            capacity: Maximum tokens in bucket
            refill_rate: Tokens added per second
        """
        self.capacity = capacity
        self.refill_rate = refill_rate
        self.tokens = capacity  # Start full
        self.last_refill = time.time()
        self.lock = Lock()
    
    def _refill(self):
        """Add tokens based on elapsed time"""
        now = time.time()
        elapsed = now - self.last_refill
        tokens_to_add = elapsed * self.refill_rate
        
        self.tokens = min(self.capacity, self.tokens + tokens_to_add)
        self.last_refill = now
    
    def allow_request(self, tokens_needed: int = 1) -> bool:
        """
        Check if request is allowed.
        Returns True if allowed, False if rate limited.
        """
        with self.lock:
            self._refill()
            
            if self.tokens >= tokens_needed:
                self.tokens -= tokens_needed
                return True
            return False
    
    def get_wait_time(self, tokens_needed: int = 1) -> float:
        """Return seconds to wait before tokens available"""
        with self.lock:
            self._refill()
            
            if self.tokens >= tokens_needed:
                return 0
            
            tokens_shortage = tokens_needed - self.tokens
            return tokens_shortage / self.refill_rate


# Usage Example
limiter = TokenBucket(capacity=10, refill_rate=1)  # 10 burst, 1/sec sustained

for i in range(15):
    if limiter.allow_request():
        print(f"Request {i+1}: ✅ Allowed")
    else:
        wait = limiter.get_wait_time()
        print(f"Request {i+1}: ❌ Rate limited (wait {wait:.2f}s)")
```

### Token Bucket Characteristics

| Aspect | Details |
|--------|---------|
| **Burst Handling** | ✅ Excellent - allows bursts up to capacity |
| **Smoothing** | ✅ Good - refill rate controls sustained rate |
| **Memory** | ✅ O(1) - just stores token count and timestamp |
| **Accuracy** | ✅ Very accurate |
| **Used By** | AWS API Gateway, Stripe, Google Cloud |

---

## 🚰 ALGORITHM 2: LEAKY BUCKET

### The Leaky Bucket Analogy

```
┌─────────────────────────────────────────────────────────────────┐
│                 LEAKY BUCKET                                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Imagine a bucket with a hole at the bottom:                   │
│   • Requests pour in from the top (any rate)                    │
│   • Requests leak out the bottom at FIXED rate                  │
│   • If bucket overflows → request rejected                      │
│                                                                  │
│        ▼ ▼ ▼ (incoming requests - variable rate)                │
│   ┌─────────┐                                                    │
│   │ ● ● ● ● │ ← Queue of waiting requests                       │
│   │ ● ● ● ● │                                                    │
│   │ ● ●     │   Queue size: 10 max                              │
│   └────┬────┘                                                    │
│        │ ● (leaks at constant rate)                             │
│        ▼                                                         │
│   [Process] → 1 request/second (constant!)                      │
│                                                                  │
│   KEY DIFFERENCE FROM TOKEN BUCKET:                             │
│   Token Bucket: Variable output rate (up to burst)              │
│   Leaky Bucket: CONSTANT output rate (smooth traffic)           │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Python Implementation

```python
import time
from collections import deque
from threading import Lock

class LeakyBucket:
    """
    Leaky Bucket Rate Limiter
    
    - Processes requests at constant rate
    - Good for smoothing traffic
    - Used by: Network traffic shaping, Nginx
    """
    
    def __init__(self, capacity: int, leak_rate: float):
        """
        Args:
            capacity: Maximum queue size
            leak_rate: Requests processed per second
        """
        self.capacity = capacity
        self.leak_rate = leak_rate
        self.queue = deque()
        self.last_leak = time.time()
        self.lock = Lock()
    
    def _leak(self):
        """Process requests that have 'leaked' out"""
        now = time.time()
        elapsed = now - self.last_leak
        
        # Number of requests that should have leaked
        leaks = int(elapsed * self.leak_rate)
        
        for _ in range(min(leaks, len(self.queue))):
            self.queue.popleft()
        
        if leaks > 0:
            self.last_leak = now
    
    def allow_request(self) -> bool:
        """
        Try to add request to queue.
        Returns True if added, False if queue full.
        """
        with self.lock:
            self._leak()
            
            if len(self.queue) < self.capacity:
                self.queue.append(time.time())
                return True
            return False


# Usage: Traffic shaping for API
limiter = LeakyBucket(capacity=10, leak_rate=2)  # Queue 10, process 2/sec
```

### Leaky vs Token Bucket

| Aspect | Token Bucket | Leaky Bucket |
|--------|--------------|--------------|
| **Output Rate** | Variable (bursty) | Constant (smooth) |
| **Burst Handling** | Allows bursts | Queues bursts, processes steadily |
| **Best For** | APIs allowing bursts | Traffic shaping, network |
| **Memory** | O(1) | O(queue size) |

---

## 🪟 ALGORITHM 3: FIXED WINDOW COUNTER

### The Clock Window Analogy

```
┌─────────────────────────────────────────────────────────────────┐
│                 FIXED WINDOW COUNTER                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Divide time into fixed windows (e.g., 1-minute windows):      │
│                                                                  │
│   Window 1         Window 2         Window 3                    │
│   [00:00-01:00]    [01:00-02:00]    [02:00-03:00]               │
│   ┌──────────┐     ┌──────────┐     ┌──────────┐               │
│   │ Count: 8 │     │ Count: 3 │     │ Count: 0 │               │
│   │ Limit:10 │     │ Limit:10 │     │ Limit:10 │               │
│   └──────────┘     └──────────┘     └──────────┘               │
│        ✅              ✅              ✅                        │
│                                                                  │
│   Simple: Just increment counter, reset at window boundary      │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### The Edge Case Problem

```
┌─────────────────────────────────────────────────────────────────┐
│                 THE BOUNDARY PROBLEM                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Limit: 10 requests per minute                                 │
│                                                                  │
│   Window 1 (00:00-01:00)    Window 2 (01:00-02:00)             │
│   ┌────────────────────┐    ┌────────────────────┐             │
│   │          ●●●●●●●●●●│    │●●●●●●●●●●          │             │
│   │            (10)    │    │    (10)            │             │
│   └────────────────────┘    └────────────────────┘             │
│            00:59 ──────────── 01:01                             │
│                                                                  │
│   PROBLEM: User sends 10 requests at 00:59,                     │
│   then 10 more at 01:01 = 20 requests in 2 seconds!            │
│                                                                  │
│   This VIOLATES the spirit of "10 per minute"!                  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Python Implementation

```python
import time
from threading import Lock

class FixedWindowCounter:
    """
    Fixed Window Rate Limiter
    
    Pros: Simple, memory efficient
    Cons: Boundary problem (2x burst at edges)
    """
    
    def __init__(self, limit: int, window_seconds: int):
        self.limit = limit
        self.window_seconds = window_seconds
        self.counters = {}  # window_key -> count
        self.lock = Lock()
    
    def _get_window_key(self, user_id: str) -> str:
        """Get current window identifier"""
        window_num = int(time.time() // self.window_seconds)
        return f"{user_id}:{window_num}"
    
    def allow_request(self, user_id: str) -> bool:
        with self.lock:
            key = self._get_window_key(user_id)
            
            # Clean old windows (simple cleanup)
            current_window = int(time.time() // self.window_seconds)
            old_keys = [k for k in self.counters 
                       if int(k.split(':')[1]) < current_window - 1]
            for k in old_keys:
                del self.counters[k]
            
            # Check and increment
            current_count = self.counters.get(key, 0)
            
            if current_count < self.limit:
                self.counters[key] = current_count + 1
                return True
            return False


# Usage
limiter = FixedWindowCounter(limit=100, window_seconds=60)
```

---

## 📊 ALGORITHM 4: SLIDING WINDOW LOG

### The Precise Approach

```
┌─────────────────────────────────────────────────────────────────┐
│                 SLIDING WINDOW LOG                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Store timestamp of EVERY request in the window:               │
│                                                                  │
│   Current time: 01:30:00                                         │
│   Window: Last 60 seconds (00:30:00 - 01:30:00)                 │
│                                                                  │
│   Request Log:                                                   │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │ 00:28:15 │ ← EXPIRED (before window)                    │   │
│   │ 00:35:22 │ ✓ In window                                  │   │
│   │ 00:45:18 │ ✓ In window                                  │   │
│   │ 01:02:33 │ ✓ In window                                  │   │
│   │ 01:15:44 │ ✓ In window                                  │   │
│   │ 01:28:55 │ ✓ In window                                  │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                  │
│   Count in window: 5                                             │
│   Limit: 10                                                      │
│   Result: ✅ ALLOWED (5 < 10)                                    │
│                                                                  │
│   PROS: Perfectly accurate                                       │
│   CONS: High memory usage (stores all timestamps)               │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Python Implementation

```python
import time
from threading import Lock
from collections import defaultdict

class SlidingWindowLog:
    """
    Sliding Window Log Rate Limiter
    
    Pros: 100% accurate, no boundary issues
    Cons: High memory (O(requests in window) per user)
    """
    
    def __init__(self, limit: int, window_seconds: int):
        self.limit = limit
        self.window_seconds = window_seconds
        self.request_logs = defaultdict(list)  # user_id -> [timestamps]
        self.lock = Lock()
    
    def allow_request(self, user_id: str) -> bool:
        with self.lock:
            now = time.time()
            window_start = now - self.window_seconds
            
            # Get user's request log
            log = self.request_logs[user_id]
            
            # Remove expired timestamps
            while log and log[0] < window_start:
                log.pop(0)
            
            # Check limit
            if len(log) < self.limit:
                log.append(now)
                return True
            return False
    
    def get_requests_count(self, user_id: str) -> int:
        """Get current request count for user"""
        with self.lock:
            now = time.time()
            window_start = now - self.window_seconds
            log = self.request_logs[user_id]
            return sum(1 for ts in log if ts >= window_start)


# Usage
limiter = SlidingWindowLog(limit=100, window_seconds=60)
```

---

## 🏆 ALGORITHM 5: SLIDING WINDOW COUNTER (Best of Both)

### The Hybrid Approach

```
┌─────────────────────────────────────────────────────────────────┐
│                 SLIDING WINDOW COUNTER                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Combines Fixed Window efficiency with Sliding accuracy:       │
│                                                                  │
│   Current time: 01:15 (15 seconds into current window)          │
│   Window size: 1 minute                                          │
│                                                                  │
│   Previous Window     Current Window                             │
│   [00:00-01:00]       [01:00-02:00]                             │
│   ┌──────────┐        ┌──────────┐                              │
│   │ Count:42 │        │ Count:18 │                              │
│   └──────────┘        └──────────┘                              │
│                                                                  │
│   Sliding estimate = (prev × overlap%) + current                │
│                    = (42 × 75%) + 18                            │
│                    = 31.5 + 18 = 49.5 ≈ 50 requests             │
│                                                                  │
│   Why 75%? We're 15s into current window (25% passed)           │
│   So 75% of previous window still "overlaps" our sliding view   │
│                                                                  │
│   BEST OF BOTH WORLDS:                                           │
│   ✅ Memory efficient (just 2 counters)                         │
│   ✅ Reasonably accurate (smooth boundary)                       │
│   ✅ Fast (O(1) operations)                                      │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Python Implementation

```python
import time
from threading import Lock
from dataclasses import dataclass

@dataclass
class WindowData:
    count: int = 0
    window_start: float = 0

class SlidingWindowCounter:
    """
    Sliding Window Counter Rate Limiter
    
    The BEST algorithm for most use cases:
    - Memory efficient: O(1) per user
    - Reasonably accurate: No sharp boundary issues
    - Fast: O(1) operations
    
    Used by: Redis rate limiting, Kong API Gateway
    """
    
    def __init__(self, limit: int, window_seconds: int):
        self.limit = limit
        self.window_seconds = window_seconds
        self.windows = {}  # user_id -> {prev_window, curr_window}
        self.lock = Lock()
    
    def allow_request(self, user_id: str) -> bool:
        with self.lock:
            now = time.time()
            current_window_start = (now // self.window_seconds) * self.window_seconds
            
            # Initialize user windows if needed
            if user_id not in self.windows:
                self.windows[user_id] = {
                    'prev': WindowData(0, current_window_start - self.window_seconds),
                    'curr': WindowData(0, current_window_start)
                }
            
            user_data = self.windows[user_id]
            
            # Roll windows if we've moved to a new window
            if current_window_start > user_data['curr'].window_start:
                # Current becomes previous
                user_data['prev'] = user_data['curr']
                # New current window
                user_data['curr'] = WindowData(0, current_window_start)
            
            # Calculate weighted count
            elapsed_in_current = now - current_window_start
            prev_weight = 1 - (elapsed_in_current / self.window_seconds)
            
            weighted_count = (
                user_data['prev'].count * prev_weight + 
                user_data['curr'].count
            )
            
            # Check limit
            if weighted_count < self.limit:
                user_data['curr'].count += 1
                return True
            return False
    
    def get_current_usage(self, user_id: str) -> float:
        """Get current weighted usage for user"""
        with self.lock:
            if user_id not in self.windows:
                return 0
            
            now = time.time()
            current_window_start = (now // self.window_seconds) * self.window_seconds
            user_data = self.windows[user_id]
            
            elapsed_in_current = now - current_window_start
            prev_weight = 1 - (elapsed_in_current / self.window_seconds)
            
            return (
                user_data['prev'].count * prev_weight + 
                user_data['curr'].count
            )


# Usage
limiter = SlidingWindowCounter(limit=100, window_seconds=60)

# Test
for i in range(120):
    allowed = limiter.allow_request("user_123")
    if not allowed:
        print(f"Request {i+1}: Rate limited! Usage: {limiter.get_current_usage('user_123'):.1f}")
        break
```

---

## 📊 ALGORITHM COMPARISON MATRIX

| Algorithm | Memory | Accuracy | Burst | Best For |
|-----------|--------|----------|-------|----------|
| **Token Bucket** | O(1) | ⭐⭐⭐⭐⭐ | ✅ Yes | APIs, cloud services |
| **Leaky Bucket** | O(queue) | ⭐⭐⭐⭐⭐ | ❌ No | Traffic shaping |
| **Fixed Window** | O(1) | ⭐⭐⭐ | ⚠️ Edge issue | Simple use cases |
| **Sliding Log** | O(n) | ⭐⭐⭐⭐⭐ | ✅ Yes | When accuracy critical |
| **Sliding Counter** | O(1) | ⭐⭐⭐⭐ | ✅ Yes | **Most use cases** |

### Recommendation

```
┌─────────────────────────────────────────────────────────────────┐
│                 WHICH TO CHOOSE?                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   🏆 DEFAULT CHOICE: Sliding Window Counter                     │
│      Best balance of accuracy, memory, and simplicity           │
│                                                                  │
│   🚀 HIGH-BURST APIs: Token Bucket                              │
│      When you want to allow bursts (gaming, streaming)          │
│                                                                  │
│   📊 TRAFFIC SHAPING: Leaky Bucket                              │
│      When you need constant output rate (network equipment)     │
│                                                                  │
│   💰 BILLING/QUOTAS: Sliding Window Log                         │
│      When 100% accuracy is required (payment APIs)              │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 🌍 DISTRIBUTED RATE LIMITING

### The Challenge

```
┌─────────────────────────────────────────────────────────────────┐
│              THE DISTRIBUTED PROBLEM                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   User makes requests to multiple servers:                      │
│                                                                  │
│   User → Server 1: "I've seen 30 requests from this user"      │
│   User → Server 2: "I've seen 25 requests from this user"      │
│   User → Server 3: "I've seen 28 requests from this user"      │
│                                                                  │
│   ACTUAL TOTAL: 83 requests!                                    │
│   LIMIT: 100 per minute                                         │
│                                                                  │
│   Problem: Each server only knows its local count               │
│   Without coordination: User gets 3x the allowed rate!          │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Solution: Redis-Based Rate Limiting

```python
import redis
import time

class DistributedRateLimiter:
    """
    Distributed Rate Limiter using Redis
    
    Uses Redis for centralized counter storage.
    All servers share the same counters.
    """
    
    def __init__(self, redis_client: redis.Redis, 
                 limit: int, window_seconds: int):
        self.redis = redis_client
        self.limit = limit
        self.window_seconds = window_seconds
    
    def allow_request(self, user_id: str) -> tuple[bool, dict]:
        """
        Check if request is allowed using Redis.
        Returns (allowed, info_dict)
        """
        now = time.time()
        window_start = now - self.window_seconds
        key = f"ratelimit:{user_id}"
        
        # Use Redis pipeline for atomic operations
        pipe = self.redis.pipeline()
        
        # Remove old entries
        pipe.zremrangebyscore(key, 0, window_start)
        
        # Count current entries
        pipe.zcard(key)
        
        # Execute
        results = pipe.execute()
        current_count = results[1]
        
        if current_count < self.limit:
            # Add new request timestamp
            pipe = self.redis.pipeline()
            pipe.zadd(key, {str(now): now})
            pipe.expire(key, self.window_seconds + 1)
            pipe.execute()
            
            return True, {
                'allowed': True,
                'remaining': self.limit - current_count - 1,
                'reset_at': window_start + self.window_seconds
            }
        
        return False, {
            'allowed': False,
            'remaining': 0,
            'reset_at': window_start + self.window_seconds,
            'retry_after': self.window_seconds
        }


# Usage with Redis
redis_client = redis.Redis(host='localhost', port=6379, db=0)
limiter = DistributedRateLimiter(redis_client, limit=100, window_seconds=60)

allowed, info = limiter.allow_request("user_123")
print(f"Allowed: {allowed}, Remaining: {info['remaining']}")
```

### Redis Lua Script (Atomic Operations)

```lua
-- rate_limit.lua
-- Atomic sliding window rate limiter in Redis

local key = KEYS[1]
local limit = tonumber(ARGV[1])
local window = tonumber(ARGV[2])
local now = tonumber(ARGV[3])
local window_start = now - window

-- Remove old entries
redis.call('ZREMRANGEBYSCORE', key, 0, window_start)

-- Count current entries
local count = redis.call('ZCARD', key)

if count < limit then
    -- Add new entry
    redis.call('ZADD', key, now, now .. '-' .. math.random())
    redis.call('EXPIRE', key, window + 1)
    return {1, limit - count - 1}  -- allowed, remaining
else
    return {0, 0}  -- denied, remaining
end
```

```python
# Using the Lua script
RATE_LIMIT_SCRIPT = """
local key = KEYS[1]
local limit = tonumber(ARGV[1])
local window = tonumber(ARGV[2])
local now = tonumber(ARGV[3])
local window_start = now - window

redis.call('ZREMRANGEBYSCORE', key, 0, window_start)
local count = redis.call('ZCARD', key)

if count < limit then
    redis.call('ZADD', key, now, now .. '-' .. math.random())
    redis.call('EXPIRE', key, window + 1)
    return {1, limit - count - 1}
else
    return {0, 0}
end
"""

class AtomicRateLimiter:
    def __init__(self, redis_client):
        self.redis = redis_client
        self.script = self.redis.register_script(RATE_LIMIT_SCRIPT)
    
    def allow_request(self, user_id: str, limit: int, window: int) -> tuple[bool, int]:
        key = f"ratelimit:{user_id}"
        result = self.script(keys=[key], args=[limit, window, time.time()])
        return bool(result[0]), result[1]
```

---

## 🏢 REAL-WORLD IMPLEMENTATIONS

### Stripe API Rate Limiting

```
┌─────────────────────────────────────────────────────────────────┐
│                 STRIPE'S APPROACH                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Rate Limits:                                                   │
│   • 100 requests/second in live mode                            │
│   • 25 requests/second in test mode                             │
│                                                                  │
│   Response Headers:                                              │
│   ─────────────────                                              │
│   X-RateLimit-Limit: 100                                        │
│   X-RateLimit-Remaining: 87                                     │
│   X-RateLimit-Reset: 1623456789                                 │
│                                                                  │
│   When Rate Limited (429):                                       │
│   ────────────────────────                                       │
│   {                                                              │
│     "error": {                                                   │
│       "type": "rate_limit_error",                               │
│       "message": "Too many requests"                            │
│     }                                                            │
│   }                                                              │
│   Retry-After: 1                                                 │
│                                                                  │
│   Algorithm: Token Bucket per API key                           │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### GitHub API Rate Limiting

```
┌─────────────────────────────────────────────────────────────────┐
│                 GITHUB'S APPROACH                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Tiered Limits:                                                 │
│   ──────────────                                                 │
│   Unauthenticated: 60 requests/hour                             │
│   Authenticated: 5,000 requests/hour                            │
│   GitHub Apps: 15,000 requests/hour                             │
│                                                                  │
│   Response Headers:                                              │
│   ─────────────────                                              │
│   X-RateLimit-Limit: 5000                                       │
│   X-RateLimit-Remaining: 4987                                   │
│   X-RateLimit-Reset: 1623459600 (Unix timestamp)                │
│   X-RateLimit-Used: 13                                          │
│   X-RateLimit-Resource: core                                    │
│                                                                  │
│   Separate limits for:                                           │
│   • Core API                                                     │
│   • Search API (30 req/min)                                     │
│   • GraphQL API                                                  │
│   • Code scanning                                                │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Netflix: Adaptive Rate Limiting

```
┌─────────────────────────────────────────────────────────────────┐
│                 NETFLIX'S ADAPTIVE APPROACH                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Netflix uses ADAPTIVE rate limiting based on:                  │
│                                                                  │
│   1. Client Priority                                             │
│      • Paying subscribers: Higher limits                        │
│      • Free trial: Lower limits                                 │
│      • Internal services: No limits                             │
│                                                                  │
│   2. System Health                                               │
│      • When backend stressed: Reduce all limits                 │
│      • When healthy: Restore normal limits                      │
│                                                                  │
│   3. Request Type                                                │
│      • Play requests: Highest priority                          │
│      • Search: Medium priority                                  │
│      • Browse: Can be shed under load                           │
│                                                                  │
│   Implementation: Zuul Gateway + Hystrix Circuit Breaker        │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 🔧 IMPLEMENTING RATE LIMITING IN YOUR STACK

### Express.js (Node.js)

```javascript
const rateLimit = require('express-rate-limit');
const RedisStore = require('rate-limit-redis');
const Redis = require('ioredis');

const redisClient = new Redis({
  host: 'localhost',
  port: 6379
});

// Basic rate limiter
const basicLimiter = rateLimit({
  windowMs: 60 * 1000, // 1 minute
  max: 100, // 100 requests per minute
  message: {
    error: 'Too many requests',
    retryAfter: 60
  },
  standardHeaders: true, // Return rate limit info in headers
  legacyHeaders: false,
});

// Distributed rate limiter with Redis
const distributedLimiter = rateLimit({
  windowMs: 60 * 1000,
  max: 100,
  store: new RedisStore({
    sendCommand: (...args) => redisClient.call(...args),
  }),
});

// Tiered rate limiting
const createTieredLimiter = (tier) => {
  const limits = {
    free: 60,
    basic: 300,
    premium: 1000,
    enterprise: 10000
  };
  
  return rateLimit({
    windowMs: 60 * 1000,
    max: limits[tier] || 60,
    keyGenerator: (req) => req.user?.id || req.ip,
  });
};

// Apply to routes
app.use('/api/', basicLimiter);
app.use('/api/search', rateLimit({ windowMs: 60000, max: 30 }));  // Stricter for search
```

### Spring Boot (Java)

```java
import io.github.bucket4j.Bandwidth;
import io.github.bucket4j.Bucket;
import io.github.bucket4j.Bucket4j;
import io.github.bucket4j.Refill;
import org.springframework.stereotype.Component;
import org.springframework.web.servlet.HandlerInterceptor;

import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.time.Duration;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

@Component
public class RateLimitInterceptor implements HandlerInterceptor {
    
    private final Map<String, Bucket> buckets = new ConcurrentHashMap<>();
    
    private Bucket createBucket() {
        // 100 tokens, refill 10 per second
        Bandwidth limit = Bandwidth.classic(100, 
            Refill.greedy(10, Duration.ofSeconds(1)));
        return Bucket4j.builder().addLimit(limit).build();
    }
    
    private Bucket resolveBucket(String apiKey) {
        return buckets.computeIfAbsent(apiKey, k -> createBucket());
    }
    
    @Override
    public boolean preHandle(HttpServletRequest request, 
                            HttpServletResponse response, 
                            Object handler) throws Exception {
        
        String apiKey = request.getHeader("X-API-Key");
        if (apiKey == null) {
            apiKey = request.getRemoteAddr();
        }
        
        Bucket bucket = resolveBucket(apiKey);
        
        if (bucket.tryConsume(1)) {
            // Add headers
            response.addHeader("X-RateLimit-Remaining", 
                String.valueOf(bucket.getAvailableTokens()));
            return true;
        }
        
        response.setStatus(429);
        response.getWriter().write("{\"error\": \"Rate limit exceeded\"}");
        response.addHeader("Retry-After", "1");
        return false;
    }
}
```

### Nginx Configuration

```nginx
# Rate limiting configuration
http {
    # Define rate limit zones
    limit_req_zone $binary_remote_addr zone=api_limit:10m rate=10r/s;
    limit_req_zone $http_x_api_key zone=api_key_limit:10m rate=100r/s;
    
    # Connection limiting
    limit_conn_zone $binary_remote_addr zone=conn_limit:10m;
    
    server {
        listen 80;
        
        # Apply rate limiting to API endpoints
        location /api/ {
            # Allow burst of 20, delay after 10
            limit_req zone=api_limit burst=20 delay=10;
            limit_req_status 429;
            
            # Custom error response
            error_page 429 = @rate_limited;
            
            proxy_pass http://backend;
        }
        
        # Stricter limit for login
        location /api/login {
            limit_req zone=api_limit burst=5 nodelay;
            limit_conn conn_limit 5;
            
            proxy_pass http://backend;
        }
        
        location @rate_limited {
            default_type application/json;
            return 429 '{"error": "Too many requests", "retry_after": 1}';
        }
    }
}
```

---

## 📋 RATE LIMITING RESPONSE HEADERS

### Standard Headers (RFC 6585)

```http
HTTP/1.1 429 Too Many Requests
Content-Type: application/json
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1623456789
Retry-After: 30

{
  "error": {
    "code": "RATE_LIMITED",
    "message": "Too many requests. Please slow down.",
    "retry_after": 30
  }
}
```

### Headers Explained

| Header | Description | Example |
|--------|-------------|---------|
| `X-RateLimit-Limit` | Max requests allowed | 100 |
| `X-RateLimit-Remaining` | Requests left in window | 45 |
| `X-RateLimit-Reset` | Unix timestamp when limit resets | 1623456789 |
| `Retry-After` | Seconds to wait before retrying | 30 |

---

## ❓ Interview Practice

### Question 1:
> "Design a rate limiter for a high-traffic API that handles 1 million requests per second across 100 servers."

**Answer:**
> "I'd use a distributed rate limiter with Redis. Each API request would check Redis using a Lua script for atomic operations. I'd implement Sliding Window Counter algorithm because it's memory-efficient (O(1) per user) and handles edge cases well. Redis can handle 100K+ ops/second per instance, so I'd use Redis Cluster for horizontal scaling. For extra resilience, I'd add local token buckets as a first layer (in-memory, sub-millisecond) that sync with Redis periodically. If Redis is temporarily unavailable, the local rate limiters provide degraded but functional limiting."

### Question 2:
> "How would you handle rate limiting for different user tiers (free, premium, enterprise)?"

**Answer:**
> "I'd implement tiered rate limiting with different limits per tier. The tier would be stored in the user's JWT token or fetched from a user service (cached). Each tier gets different limits: free (60/min), premium (600/min), enterprise (6000/min). I'd use separate Redis keys per user: `ratelimit:{user_id}:{endpoint}`. For enterprise customers, I might also add burst allowance using Token Bucket. I'd expose the limits via response headers so clients can self-throttle. For billing-based limits (monthly quotas), I'd use a separate counter with longer windows."

### Question 3:
> "What happens when your rate limiting Redis goes down?"

**Answer:**
> "This is critical to handle gracefully. My strategy: 1) Circuit breaker pattern - if Redis is down, fall back to local in-memory rate limiting per server (less accurate but still protective). 2) Health checks - monitor Redis latency and fail-fast if response time exceeds 10ms. 3) Redundancy - use Redis Sentinel or Redis Cluster for automatic failover. 4) Graceful degradation - when uncertain, I'd slightly reduce limits (fail-safe) rather than allow unlimited traffic. 5) Local caching - cache rate limit decisions for 1-2 seconds to reduce Redis calls."

---

## 🔗 Connecting to Previous Days

| Day | Concept | How It Connects |
|-----|---------|-----------------|
| Day 8 | Load Balancing | Rate limiters often deployed at load balancer level |
| Day 13 | Circuit Breaker | Circuit breaker + rate limiting = robust APIs |
| Day 15 | Redis | Redis is the go-to for distributed rate limiting |
| Day 28 | Consistent Hashing | Used to distribute rate limit counters |

---

## ✅ Day 29 Action Items

1. **Implement a rate limiter** — Build Token Bucket or Sliding Window Counter from scratch
2. **Add to your API** — Integrate rate limiting middleware in your project
3. **Use Redis** — Implement distributed rate limiting with Redis
4. **Test edge cases** — Verify behavior at window boundaries
5. **Monitor** — Add metrics for rate limit hits (429 responses)

---

## 💡 Key Takeaways

| Takeaway | Why It Matters |
|----------|----------------|
| Choose the right algorithm | Sliding Window Counter for most cases |
| Distribute with Redis | Single-server limiters break at scale |
| Use atomic operations | Lua scripts in Redis prevent race conditions |
| Return proper headers | Clients need X-RateLimit-* to self-throttle |
| Fail safely | When limiter fails, restrict rather than allow |

---

## 🚀 Key Architect Principles

| Principle | What It Means |
|-----------|---------------|
| **Protect at the edge** | Rate limit at API gateway, before hitting backend |
| **Fair allocation** | Per-user limits, not just global |
| **Graceful degradation** | Return 429 with Retry-After, not 500 |
| **Observable** | Monitor rate limit hits, identify abusers |
| **Tiered access** | Different limits for different customer tiers |

---

## 💡 Key Takeaway

> **Junior: "Let's just reject requests when the server is overloaded."**
>
> **Architect: "That's reactive. Rate limiting is proactive — we define limits based on capacity planning, implement them at the edge with distributed counters in Redis, return proper 429 responses with Retry-After headers so clients can back off intelligently, and tier limits based on customer value. We protect the system before it gets overloaded, not after."**

The difference? Understanding that **protection happens before the problem**. Rate limiting isn't about saying no — it's about ensuring you can say yes to the requests that matter.

---

*— Sunchit Dudeja*  
*Day 29 of 50: System Design Interview Preparation Series*

---

> 🎯 **Interview Edge:** When discussing rate limiting, mention all layers: "I'd implement rate limiting at multiple levels — Nginx for basic IP-based limits, API Gateway for per-key limits using Token Bucket, and application-level for business logic limits. All backed by Redis with Lua scripts for atomic operations, with proper 429 responses and X-RateLimit headers."

> 📢 **Real Impact:** Stripe processes millions of API calls per day. Their Token Bucket rate limiter with Redis ensures no single customer can overwhelm the system while still allowing legitimate burst traffic. This is why you can safely integrate Stripe without worrying about noisy neighbor problems.

---

## 🔗 Resources

- **Libraries:** `express-rate-limit` (Node), `bucket4j` (Java), `limits` (Python)
- **Redis Module:** `redis-cell` (native rate limiting)
- **Standards:** RFC 6585 (429 status code), IETF draft-ietf-httpapi-ratelimit-headers

---

> 💡 **Tomorrow (Day 30):** We'll explore **Message Queues & Event-Driven Architecture** — how do companies like Uber and Airbnb handle millions of events without losing a single one? RabbitMQ, Kafka, and the pub/sub patterns that power modern systems.
