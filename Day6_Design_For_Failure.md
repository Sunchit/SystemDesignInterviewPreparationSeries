# Design for Failure: The Architect's Mindset
### Day 6 of 50 - System Design Interview Preparation Series

**By Sunchit Dudeja**

---

## 🎯 Welcome to Day 6!

Yesterday, we learned to estimate capacity like an architect. Today, we learn the most crucial mindset shift of all: **designing for failure**.

> The best systems aren't those that never fail, but those that fail gracefully and recover automatically.

---

## 🧠 Two Perspectives: Developer vs Architect

| Mindset | Approach |
|---------|----------|
| **Developer** | "Design to perfection" — assumes everything works |
| **Architect** | "Design for failure" — assumes everything will break |

Let's see this in action with a real example.

---

## 🚗 UBER RIDE BOOKING: Two Perspectives

### Developer's Implementation

```java
public Ride bookRide(User user, Location pickup, Location drop) {
    // Find nearby drivers
    List<Driver> drivers = driverService.findNearby(pickup);
    
    // Select best driver
    Driver driver = selectOptimalDriver(drivers);
    
    // Create ride record
    Ride ride = rideRepository.save(new Ride(user, driver, pickup, drop));
    
    // Notify driver
    notificationService.notifyDriver(driver, ride);
    
    return ride;
}
```

**Developer's assumption:** Works perfectly in development. Ship it! ✅

**Reality:** This code has at least **8 failure points** that will break in production.

---

## 🔴 Architect's Failure Scenario Analysis

### 1. DATABASE FAILURE

| Perspective | Thinking |
|-------------|----------|
| **Developer** | "Database is always available" |
| **Architect** | "Database WILL go down. What then?" |

**Architect's Plan:**

```
Primary DB (Mumbai) → Replica 1 (Delhi) → Replica 2 (Singapore)

If Mumbai fails:
1. Automatic failover to Delhi (within 30 seconds)
2. Queue writes in Kafka during failover
3. Sync back when Mumbai recovers
4. Monitor replication lag (<200ms)
```

---

### 2. DRIVER SERVICE UNREACHABLE

| Perspective | Thinking |
|-------------|----------|
| **Developer** | "`driverService.findNearby()` always returns drivers" |
| **Architect** | "What if driver service is down? What if it's slow?" |

**Architect's Implementation:**

```java
// Circuit breaker pattern
@CircuitBreaker(failureThreshold = 5, timeout = 3000)
public List<Driver> findNearbyWithFallback(Location pickup) {
    try {
        return driverService.findNearby(pickup);
    } catch (ServiceUnavailableException e) {
        // Fallback 1: Check local cache (last 5 minutes)
        List<Driver> cached = redis.get("drivers:" + pickup.hash());
        
        // Fallback 2: Use static grid-based availability
        if (cached.isEmpty()) {
            return gridAvailabilityService.estimate(pickup);
        }
        
        // Fallback 3: Return empty with user-friendly message
        return Collections.emptyList();
    }
}
```

---

### 3. NOTIFICATION SERVICE FAILURE

| Perspective | Thinking |
|-------------|----------|
| **Developer** | "Driver gets notified instantly" |
| **Architect** | "What if push notification fails? Driver misses rides?" |

**Architect's Strategy:**

```
Real-time attempt → Fail? → Retry queue (RabbitMQ) → 
Dead letter queue after 3 attempts → 
Fallback to SMS → Log for manual follow-up
```

**Retry Logic:**

```java
notificationService.notify(driver, ride)
    .retry(3)
    .withExponentialBackoff(1s, 30s)
    .onFailure(() -> smsService.send(driver.phone, "New ride!"))
    .onCompleteFailure(() -> rideService.markAsManualAssignment(ride));
```

---

### 4. CONCURRENT BOOKING RACE CONDITION

| Perspective | Thinking |
|-------------|----------|
| **Developer** | "It won't happen with our load" |
| **Architect** | "Two users booking the same driver at the same millisecond?" |

**Architect's Prevention:**

```java
// Distributed lock for driver
Lock lock = redisLock.acquire("driver:" + driverId, 5);
try {
    if (lock.isAcquired()) {
        // Check driver still available
        if (driverStatusService.isAvailable(driverId)) {
            // Proceed with booking
        }
    }
} finally {
    lock.release();
}
```

---

### 5. GEO-SERVICE LATENCY

| Perspective | Thinking |
|-------------|----------|
| **Developer** | "Location services are fast" |
| **Architect** | "What if Google Maps API is slow today?" |

**Architect's Design:**

```
User's location → Multiple geocoders in parallel:
1. Google Maps (primary)
2. Mapbox (secondary)
3. Local cache (recent coordinates)
4. Approximate using IP (fallback)

Return first successful response within 200ms
```

---

### 6. PAYMENT SERVICE DEGRADATION

| Perspective | Thinking |
|-------------|----------|
| **Developer** | "Payment either works or fails" |
| **Architect** | "What if payment is slow? What if it fails at scale?" |

**Architect's Graceful Degradation:**

```
Phase 1: Full payment processing
Phase 2: Payment fails → "Pay later" option
Phase 3: Payment service down → "Cash only" mode
Phase 4: Complete failure → Free rides for existing users
```

---

### 7. SCALING FAILURE DURING PEAK HOURS

| Perspective | Thinking |
|-------------|----------|
| **Developer** | "We'll add more servers" |
| **Architect** | "What if scaling is too slow for the spike?" |

**Architect's Auto-scaling Plan:**

```
Monitoring: Requests per second > 1000
Trigger: Auto-scale from 10 → 50 instances

Fallback: If scaling too slow → 
  1. Return "High demand, try in 5 minutes"
  2. Static pricing instead of surge
  3. Disable non-critical features (ETA precision)
```

---

### 8. DATA CONSISTENCY FAILURE

| Perspective | Thinking |
|-------------|----------|
| **Developer** | "Database transactions ensure consistency" |
| **Architect** | "What if write succeeds but notification fails?" |

**Architect's Eventual Consistency Design:**

```
1. Ride booked (async) → Kafka event
2. Multiple consumers:
   - Update driver app
   - Update rider app  
   - Update analytics
   - Update billing
3. Reconciliation job every 5 minutes
4. Conflict resolution: Last write wins with manual review flag
```

---

## 📊 Architect's Failure Matrix for Uber

| Component | Failure Mode | Impact | Mitigation | Fallback |
|-----------|--------------|--------|------------|----------|
| Driver Service | 100% outage | Can't book rides | Multi-region deployment | Show cached drivers |
| Payment Gateway | Slow response | Delayed booking | Circuit breaker | "Pay later" option |
| Maps API | Timeout | No ETA | Multiple providers | Static distances |
| Database | Primary down | No new rides | Hot replica failover | Queue in Kafka |
| Notification | Failure | Driver unaware | Retry + SMS | Manual dispatch |
| Pricing Engine | Bug | Wrong fare | Dual calculation | Flat rate based on distance |

---

## 🔧 Architect's Resilience Patterns

### 1. BULKHEAD PATTERN

Isolate failures so they don't cascade:

```
Separate thread pools for:
- Driver matching (critical)
- Price calculation (important) 
- Analytics (non-critical)
- Notifications (best effort)

Failure in analytics won't affect bookings.
```

**Analogy:** Like compartments in a ship — water flooding one compartment doesn't sink the whole ship.

---

### 2. TIMEOUTS AT EVERY BOUNDARY

```yaml
services:
  driver-service:
    timeout: 2s
    retries: 2
  
  pricing-service:
    timeout: 1s
    retries: 0  # No retry for pricing
    
  maps-service:
    timeout: 3s
    circuit-breaker: 50% failure
```

**Rule:** Never wait forever. Set timeouts on EVERY external call.

---

### 3. DEGRADATION LADDER

```
Level 1: Full features
Level 2: No surge pricing
Level 3: Estimated ETA only
Level 4: Cash payments only  
Level 5: Manual booking via call center
```

**Principle:** It's better to offer reduced service than no service.

---

### 4. CHAOS ENGINEERING

Regularly test failures in production:

```
- Kill driver-service instances
- Add 500ms latency to database
- Corrupt payment service responses
- Simulate 10x traffic spike
```

**Netflix's approach:** They have a "Chaos Monkey" that randomly kills services to ensure the system recovers automatically.

---

## 🎯 The 5 Architect Principles

| # | Principle | Meaning |
|---|-----------|---------|
| 1 | **Everything Fails** | Assume every dependency will fail |
| 2 | **Graceful Degradation** | Service should work in degraded mode |
| 3 | **Fast Failure** | Fail fast, don't hang waiting |
| 4 | **Observability** | Must know when something is failing |
| 5 | **Automated Recovery** | Systems should heal themselves |

---

## 🧪 Exercise: Design Your Own Failure Scenarios

**Pick:** WhatsApp message sending

**Developer thinks:** "Message sends or fails"

**Architect asks:**

| Scenario | What Could Go Wrong? |
|----------|---------------------|
| Encryption | What if encryption service is down? |
| Offline | What if recipient's phone is offline for a week? |
| Queue Full | What if message queue is full? |
| Ack Failure | What if database write succeeds but acknowledgment fails? |
| Data Center | What if one data center goes offline? |

**Your Task:** Design 3 fallback strategies for each scenario.

---

## ❓ Interview Practice

### Question 1:
> "How would you handle database failures in a ride-booking system?"

**Answer:**
> "I'd implement a multi-region database setup with automatic failover. Primary in Mumbai, replicas in Delhi and Singapore. If primary fails, we failover to Delhi within 30 seconds. During failover, I'd queue writes in Kafka to prevent data loss. I'd also monitor replication lag to ensure it stays under 200ms. For reads, we can continue serving from any replica."

### Question 2:
> "What's the Circuit Breaker pattern?"

**Answer:**
> "Circuit Breaker prevents cascading failures. If a service fails repeatedly (say 5 times in a row), the circuit 'opens' and we stop calling that service for a timeout period. Instead, we return a fallback response immediately. After the timeout, we 'half-open' and test with one request. If it succeeds, circuit closes and normal operation resumes. This prevents one failing service from bringing down the entire system."

---

## 🔗 Connecting to Previous Days

| Day | Concept | How It Connects |
|-----|---------|-----------------|
| Day 2 | SCALED | Availability = designing for failure |
| Day 3 | NFRs | Reliability requirements drive failure design |
| Day 4 | Client-Server | Server failures affect all clients |
| Day 5 | Capacity | Over-capacity situations are failure modes |

---

## ✅ Day 6 Action Items

1. **List 5 failure points** in any app you've built
2. **Design fallbacks** for each failure point
3. **Learn the patterns** — Circuit Breaker, Bulkhead, Retry with Backoff
4. **Think in degradation levels** — What's the minimum viable service?

---

## 💡 Key Takeaway

> **Developer: "My code works perfectly in development."**
> **Architect: "What happens when the database is down, the network is slow, and 10x users hit the system during a flash sale?"**

The difference isn't pessimism — it's **realism**. In distributed systems, failures aren't edge cases. They're Tuesday.

Design for the failure, and success takes care of itself.

---

*— Sunchit Dudeja*  
*Day 6 of 50: System Design Interview Preparation Series*

---

> 🎯 **Interview Edge:** When designing any system, proactively say: "Let me walk you through the failure scenarios and how I'd handle them." Then discuss database failures, service timeouts, and graceful degradation. This immediately signals senior-level thinking.


