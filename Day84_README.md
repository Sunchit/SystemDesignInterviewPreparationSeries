# High-Scale Booking System: Design & Implementation Guide

## Overview

This directory contains the **Day 84** blog post for the System Design Interview Preparation Series: *The High-Scale Booking System: Concurrency, Redis, Queues, and Preventing Double Booking.*

The blog explores how to design a system that can handle millions of concurrent booking requests without **double-booking** the same seat.

---

## Problem Statement

### The Core Challenge

When a popular event releases limited inventory (e.g., 500 concert tickets) and millions of users attempt to book simultaneously:

```
User A: Check Seat 42 → Available
User B: Check Seat 42 → Available
User A: Book Seat 42 → SUCCESS
User B: Book Seat 42 → SUCCESS ❌ DOUBLE BOOKED
```

This is a **race condition**. The CHECK and BOOK operations are separate, and two concurrent requests can both observe the same state before either one modifies it.

### Scale Makes It Inevitable

- At 10 users: double bookings are unlikely
- At 1 million users: double bookings happen thousands of times per second

---

## Key Architecture Components

### 1. **Load Balancer**
Distributes traffic across multiple application servers.
- Without it: Single server maxes out at ~10k RPS
- With it: 100 servers × 10k RPS = 1M RPS

### 2. **Rate Limiter**
Protects downstream services from overload.
- Per-user limit: 20 RPS
- Per-IP limit: 100 RPS
- Per-event limit: 50,000 RPS
- Requests above limit are rejected (HTTP 429) or queued

### 3. **Queue (Kafka, RabbitMQ, SQS)**
Absorbs traffic spikes and provides backpressure.
- Allows the system to process requests at a sustainable rate
- Decouples demand (1M RPS) from processing (50k RPS)
- Provides replay-ability and durability

### 4. **Booking Service**
Main application logic.
- Stateless
- Horizontally scalable
- Consumes messages from queue

### 5. **Redis**
Fast, atomic coordination layer.
- `SET seat:42 user-123 NX EX 300` atomically claims a seat
- Single-threaded execution prevents race conditions on keys
- TTL (300 seconds) prevents abandoned reservations
- **NOT** a replacement for the database — temporary state only

### 6. **Database (PostgreSQL, MySQL)**
Durable source of truth.
- Persists confirmed bookings
- Enforces `UNIQUE (event_id, seat_id)` constraint
- Final safety net if application logic fails

### 7. **Waitlist**
When inventory is sold out, queue users for cancellations.
- When a booking is cancelled, offer the seat to the next waitlisted user
- Turns failure into a product feature

---

## The Complete Flow

```
User Request
    ↓
Load Balancer (distribute)
    ↓
Rate Limiter (protect)
    ↓
Queue (absorb)
    ↓
Booking Service (consume)
    ↓
Idempotency Check (avoid duplicates from retries)
    ↓
Redis SET NX (atomic claim)
    ├─ SUCCESS → persist to DB
    └─ FAILURE → add to waitlist
    ↓
Database INSERT (durable record)
    ├─ SUCCESS → CONFIRMED
    └─ FAILURE → reject
    ↓
Notify User (sync or async)
```

---

## Three Problems, Three Solutions

| Problem | Cause | Solution |
|---------|-------|----------|
| **Scalability** | 1M RPS arrives; system can't process all | Load Balancer, Queue, Rate Limiter |
| **Concurrency** | Multiple users want same resource | Redis atomic SET, DB unique constraint |
| **Reliability** | Networks lose packets, requests retry | Idempotency keys, TTLs, retries |

Weak architectures try to solve all three with one component. Strong architectures use the right tool for each.

---

## Critical Design Decisions

### ✅ Redis Is NOT the Source of Truth

```
❌ WRONG:
Redis = booking record

✅ RIGHT:
Redis = temporary reservation (5-min TTL)
Database = durable booking record
```

If Redis claims a seat but the database write fails, the TTL eventually expires the reservation, making the seat available again.

### ✅ Idempotency Is Mandatory

```
User clicks Book
    ↓
Server processes successfully
    ↓
Response packet lost
    ↓
User sees timeout, clicks again
```

Without idempotency keys, the second click creates a second booking. With idempotency:

```
Request 1: Idempotency-Key: abc-123 → CONFIRMED booking-1
Request 2: Idempotency-Key: abc-123 → Return same booking-1
```

### ✅ Database Constraints Are the Final Defense

Even if the application layer fails to prevent double booking, the database constraint stops it:

```sql
ALTER TABLE bookings
ADD CONSTRAINT unique_event_seat
UNIQUE (event_id, seat_id);

-- Request 1: INSERT → SUCCESS
-- Request 2: INSERT → UNIQUE violation ❌
```

### ✅ TTLs Prevent Resource Leaks

```
SET seat:42 reservation-token EX 300

If user abandons checkout:
After 300 seconds → Redis key expires
Seat becomes available again
```

Without TTL, the seat stays reserved forever.

---

## Failure Scenarios

### Redis Unavailable

**Option 1 (Fail Closed):**
```
Booking disabled
User sees: "Booking temporarily unavailable"
Risk: Loss of revenue for 5 minutes
Benefit: Zero risk of double booking
```

For ultra-scarce inventory, correctness is more important than availability.

**Option 2 (Database Fallback):**
```
Use DB row-level locks or transactions
Higher latency but booking still works
Risk: Database overload
Benefit: Booking remains available
```

### Database Fails

```
Redis reservation created → SUCCESS
DB write fails
    ↓
TTL cleans up Redis entry after 300 seconds
Seat available again
```

TTLs act as a self-healing mechanism.

### Queue Duplicates Message

```
Booking Service receives same message twice
    ↓
Idempotency key prevents second booking
```

### Response Packet Lost

```
Booking succeeds
Response dropped
User sees timeout, retries
    ↓
Idempotency key returns original result
```

---

## Performance Considerations

### Hot Keys

A massively popular event creates concentrated traffic on a few keys:

```
SET seat:42 user-123  ← millions of requests to the same key
```

**Mitigations:**
1. **Sharding** — partition inventory by seat range (A-M, N-Z)
2. **Request coalescing** — merge duplicate concurrent attempts
3. **Queue discipline** — rate limit more aggressively
4. **Waitlist focus** — once sold out, stop booking attempts, focus on waitlist management

### Monitoring & Alerting

**Metrics to track:**
- Request rate (per service, per event)
- Queue depth and lag
- Redis latency and memory usage
- Database lock contention and query times
- Double-booking attempts (should be zero)
- Waitlist size

**Alerts:**
- Queue depth exceeding threshold
- Redis latency above 100ms
- Database lock timeouts
- Unusual double-booking attempts

---

## Related System Design Topics

- **Distributed Locking** — pessimistic concurrency control (Day 85)
- **Idempotency Keys** — making retries safe (Day 48)
- **Dead Letter Queues** — handling poison messages (Day 81)
- **Database Sharding** — horizontal scaling (Day 64)
- **Circuit Breakers** — graceful degradation (Day 72)

---

## Real-World Applications

This architecture pattern applies to any system with **many users + limited resources + concurrent requests:**

```
Tickets     → Concert seating
Hotels      → Room availability
Flights     → Seats
Parking     → Spots
Healthcare  → Appointment slots
Cloud       → GPU/compute allocation
Inventory   → Product units
Banking     → Cryptocurrency UTXOs
Gaming      → Item drops
Restaurants → Table reservations
```

The core principle is the same: **guarantee exactly-one allocation per resource, even under extreme concurrency.**

---

## Key Takeaways

1. **Concurrency is hard.** A sequence of reads and writes is NOT atomic. Use atomic operations (Redis, DB locks) for critical invariants.
2. **Queues + Rate Limiting prevent overload.** They're your first line of defense.
3. **Redis is for coordination, not persistence.** Use TTLs and database persistence.
4. **Idempotency is not optional.** Assume every request will be retried.
5. **Defense in depth.** Redis + Database constraint + Idempotency + TTL = robust system.
6. **Fail gracefully.** Sometimes "booking unavailable" is the right answer.

---

## Interview Tips

When asked to design a high-scale booking system:

1. **Start with the problem:** "The core problem is preventing double booking under concurrency."
2. **Draw the architecture:** Load Balancer → Rate Limiter → Queue → Booking Service → Redis + Database
3. **Identify the critical path:** Idempotency check → Redis SET-NX → DB INSERT
4. **Discuss failure scenarios:** "What if Redis fails? What if DB fails? What if the queue duplicates a message?"
5. **Mention scalability knobs:** Rate limits, queue throughput, number of booking workers, connection pools
6. **Talk about observability:** "We need to monitor queue depth, Redis latency, double-booking attempts"

---

*Day 84 · System Design Interview Preparation Series · Code with Sunchit Dudeja*
