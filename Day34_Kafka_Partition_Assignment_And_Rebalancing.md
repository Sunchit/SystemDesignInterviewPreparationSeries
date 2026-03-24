# Kafka Partition Assignment & Rebalancing: The Complete Guide
### Day 34 of 50 - System Design Interview Preparation Series

**By Sunchit Dudeja**

---

## 🎯 Welcome to Day 34!

Yesterday, we mastered Distributed Tracing IDs. Today, we dive into **Kafka partition assignment** and the internals of **consumer group rebalancing**—how Kafka distributes partitions across consumers and what happens when the group changes.

> **The Problem:** You have 10 partitions and 6 consumers. Who gets what? What happens when a consumer crashes? How does Kafka decide?

> **The Solution:** One partition → one consumer (in a group). Kafka's Group Coordinator orchestrates assignment. Rebalancing redistributes when the group changes.

> **📐 Excalidraw Diagram:** Open [kafka-partition-assignment.excalidraw](./kafka-partition-assignment.excalidraw) at [excalidraw.com](https://excalidraw.com) (File → Open) — black background.

---

## 📊 THE CORE PRINCIPLE

**Kafka's cardinal rule:** A partition can only be consumed by **ONE** consumer in a consumer group at any given time. This ensures **message ordering within a partition**.

---

## 🔍 THE THREE SCENARIOS

### Scenario 1: Consumers < Partitions (e.g., 6 Consumers, 10 Partitions)

**What happens:**
- Partitions are evenly distributed among available consumers
- Some consumers handle multiple partitions
- Example: 10 partitions / 6 consumers = 4 consumers get 2 partitions each, 2 consumers get 1 partition each

**Algorithm (RangeAssignor — default):**

```python
# Simplified logic
partitions_per_consumer = total_partitions // total_consumers
extra_partitions = total_partitions % total_consumers

# First 'extra_partitions' consumers get (partitions_per_consumer + 1) partitions
# Remaining consumers get partitions_per_consumer partitions
```

---

### Scenario 2: Consumers = Partitions (e.g., 10 Consumers, 10 Partitions)

**What happens:**
- 1:1 mapping between consumers and partitions
- Each consumer handles exactly one partition
- **Optimal distribution** for maximum parallelism

---

### Scenario 3: Consumers > Partitions (e.g., 15 Consumers, 10 Partitions)

**What happens:**
- Only 10 consumers will be active (one per partition)
- Remaining 5 consumers sit **idle** (standby)
- Idle consumers become active only if an active consumer fails (rebalancing)

**This is a FEATURE, not a bug!** It provides:
- **High availability** — Immediate failover
- **Elasticity** — No need to restart consumers when scaling
- **Load distribution** — Work redistributes on failure

---

## 📊 PARTITION ASSIGNMENT STRATEGIES

| Assignor | How It Works | Pros | Cons |
|----------|--------------|------|------|
| **RangeAssignor** (Default) | Assigns partitions per topic, contiguous ranges | Good for co-location | Can cause imbalance across topics |
| **RoundRobinAssignor** | Cycles through consumers round-robin | Even distribution | Breaks co-location |
| **StickyAssignor** | Minimizes movement during rebalance | Best for stability, less reprocessing | Slightly more complex |
| **CooperativeStickyAssignor** | Incremental rebalancing | No "stop the world" — consumers keep processing | Requires Kafka 2.4+ |

---

## 🔄 KAFKA REBALANCING: The Complete Deep Dive

### What is Rebalancing?

**Rebalancing** is the process where a consumer group redistributes partition ownership among its members when the group composition changes. Think of it as reassigning seats when passengers get on or off a bus.

---

### When Does Rebalancing Trigger?

| Trigger | Example | Impact |
|---------|---------|--------|
| **Consumer joins** | Auto-scaling adds new instance | Partitions redistributed |
| **Consumer leaves** | Pod crash, network disconnect | Remaining consumers take over |
| **Topic expansion** | Add partitions to existing topic | New partitions assigned |
| **Subscription change** | Consumer adds/removes subscribed topics | Full reassignment |
| **Heartbeat timeout** | Consumer fails to heartbeat within `session.timeout.ms` (default 45s) | Consumer declared dead |
| **Processing timeout** | `max.poll.interval.ms` exceeded (default 5 min) | Consumer ejected even if heartbeating |
| **Rack changes*** | Broker rack configuration changes (KIP-1101) | Coordinator recalculates assignment |

---

### The Two Rebalance Protocols

#### Classic Protocol (Pre-Kafka 4.0)

**Eager Rebalancing (Stop-the-World)**

- ✅ Simple to implement
- ❌ Full stop-the-world — all consumers pause
- ❌ Complete partition revocation
- ❌ Processing resumes only after full assignment

**Cooperative Rebalancing (Improvement)**

- ✅ Consumers keep partitions not involved in rebalance
- ✅ Reduced downtime
- ❌ Still requires group-wide synchronization barrier
- ❌ Commit pauses during rebalance

#### New Protocol (KIP-848) — Kafka 4.0+

A complete redesign moving coordination to the **broker side**.

| Feature | Classic Protocol | New Protocol (KIP-848) |
|---------|------------------|------------------------|
| **Coordination** | Client-side (leader) | Server-side (coordinator) |
| **Rebalance Impact** | All consumers affected | Only affected consumers |
| **Fetch Processing** | Paused during rebalance | Continues unaffected |
| **Commit Processing** | Paused | Continues |
| **Communication** | JoinGroup/SyncGroup phases | Continuous heartbeat |

**Server-Driven Reconciliation:**
- **Declarative State:** Consumers declare subscriptions via heartbeat
- **Coordinator Orchestration:** Broker tracks group state and computes target assignment
- **Incremental Reconciliation:** Coordinator issues specific revoke/assign commands
- **No Global Pause:** Only consumers with affected partitions pause briefly

---

### Rebalance State Machine (KIP-848)

| State | Description |
|-------|-------------|
| **NEW** | Initial state upon creation |
| **JOINING** | Consumer wants to join group, sends heartbeat with null MemberId |
| **JOINED** | Known to coordinator, may not have partitions yet |
| **ASSIGNING** | Performing assignment reconciliation |

---

### The Group Coordinator

Kafka uses a **Group Coordinator** — a broker elected to manage consumer group membership and partition assignment for that group.

```yaml
How Group Coordinator is chosen:
  - Consumer sends FindCoordinator request to any broker
  - Broker returns the coordinator for group.id (hash % num_partitions of __consumer_offsets)
  - All group operations go to that coordinator
```

---

### Rebalancing Protocol: The Two-Phase Process

#### Phase 1: JoinGroup

```
1. Consumer sends JoinGroup request to Group Coordinator
2. Coordinator waits for all known members (or session.timeout.ms)
3. Coordinator selects one consumer as LEADER
4. Coordinator sends JoinGroup response to each consumer with:
   - generation (epoch) — incremented on each rebalance
   - member_id
   - leader_id (for the leader only)
   - members list (for the leader only)
```

**Key point:** The **leader** consumer is responsible for computing the partition assignment. The coordinator does NOT assign partitions—it delegates to the leader.

#### Phase 2: SyncGroup

```
1. LEADER consumer computes assignment using the partition assignor
2. LEADER sends SyncGroup request with assignment map
3. FOLLOWER consumers send SyncGroup with empty assignment
4. Coordinator stores the assignment, sends it to all members
5. Each consumer receives its partition assignment
6. Consumers start (or resume) consuming
```

---

### The Rebalance Flow (Step by Step)

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    REBALANCE SEQUENCE (EAGER REBALANCING)                │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  1. TRIGGER: Consumer C3 crashes (no heartbeat)                         │
│                                                                         │
│  2. Coordinator detects: C3 missed session.timeout.ms (default 45s)   │
│                                                                         │
│  3. Coordinator marks group as "rebalancing"                             │
│     - All consumers must re-join                                       │
│                                                                         │
│  4. JOINGROUP PHASE:                                                    │
│     - C1, C2 send JoinGroup                                             │
│     - Coordinator waits (max 45s default) for members                  │
│     - C3 is gone, so coordinator proceeds with C1, C2                   │
│     - Coordinator picks C1 as leader, sends JoinGroupResponse           │
│                                                                         │
│  5. SYNCGROUP PHASE:                                                    │
│     - C1 (leader) computes: C1→[P0,P1,P2], C2→[P3,P4]                   │
│     - C1 sends SyncGroup with assignment                                │
│     - C2 sends SyncGroup (empty)                                        │
│     - Coordinator stores assignment, sends to C1 and C2                  │
│                                                                         │
│  6. CONSUMERS RESUME:                                                   │
│     - C1, C2 revoke old partitions, get new assignment                  │
│     - Both stop consuming briefly (STOP THE WORLD)                      │
│     - Then resume with new partitions                                   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

### EAGER vs COOPERATIVE Rebalancing

| Mode | What Happens | Downtime |
|------|--------------|----------|
| **EAGER** (Range, RoundRobin, Sticky) | All consumers **revoke all partitions** and re-join. Full stop, then new assignment. | Yes — "stop the world" |
| **COOPERATIVE** (CooperativeStickyAssignor) | Consumers **incrementally** give up only the partitions that need to move. Others keep consuming. | Minimal — no full pause |

**Cooperative Rebalancing Flow:**

```
1. C3 crashes → Coordinator triggers rebalance
2. C1, C2 receive RevokePartitions (only partitions moving away)
3. C1 gives up P2 (was going to C3), keeps P0, P1
4. C2 gives up P4 (was going to C3), keeps P3
5. C1, C2 send JoinGroup (new generation)
6. Coordinator assigns: C1→[P0,P1,P2], C2→[P3,P4]
7. C1, C2 never fully stopped — only brief pause for moved partitions
```

---

### ⚠️ Problems Caused by Rebalancing

#### 1. Message Processing Pause

```
During Rebalance → All consumers stop → Messages pile up → Lag spikes
```

#### 2. Message Duplication

**Root cause:** Offset commit lags behind message processing. If a consumer processes messages but crashes before committing, the new consumer will reprocess them.

#### 3. Message Loss

With **auto-commit** enabled (default: every 5 seconds):

```
1. Consumer polls messages (offsets 100-200)
2. Auto-commits offset 200 after 5 seconds
3. Processes message 150 → CRASHES before finishing
4. Rebalance happens
5. New consumer starts from offset 200
6. Messages 150-199 are LOST forever!
```

#### 4. Load Imbalance

Default Range/RoundRobin assignors can create uneven distribution.

**Example:** 5 partitions, 2 consumers → 3 partitions vs 2 partitions. One consumer handles 50% more load.

---

### 🛡️ How to Optimize Rebalancing

#### 1. Parameter Tuning

```properties
# Adjust for your processing time
max.poll.interval.ms = 300000  # Default: 5 minutes (increase if you process large messages)

session.timeout.ms = 45000     # Default: 45 seconds (increase to 60-120s in unstable networks)

heartbeat.interval.ms = 3000   # Default: 3 seconds (should be 1/3 of session.timeout.ms)
```

#### 2. Use Sticky Assignor

```java
// Minimizes partition movement during rebalance
props.put("partition.assignment.strategy", 
          "org.apache.kafka.clients.consumer.StickyAssignor");
```

#### 3. Manual Offset Management

```java
// ❌ BAD: Auto-commit (risks data loss)
props.put("enable.auto.commit", "true");

// ✅ GOOD: Manual commit after processing
props.put("enable.auto.commit", "false");

while (true) {
    ConsumerRecords<String, String> records = consumer.poll(100);
    for (ConsumerRecord<String, String> record : records) {
        process(record);  // Process FIRST
    }
    consumer.commitSync();  // Commit AFTER successful processing
}
```

#### 4. Enable KIP-848 (Kafka 4.0+)

```properties
# Enable new protocol (20x faster rebalances!)
group.protocol = consumer

# REMOVE these (now server-side):
# partition.assignment.strategy
# session.timeout.ms
# heartbeat.interval.ms
```

**Migration Checklist:**
- ✅ Upgrade brokers to Kafka 4.0+ (KRaft mode)
- ✅ Update clients to compatible version
- ✅ Configure `group.protocol=consumer`
- ✅ Use rolling restart (live migration supported)

#### 5. Idempotent Consumers

```java
// Make processing idempotent - safe for duplicates
public void process(ConsumerRecord record) {
    String orderId = record.value().getOrderId();
    
    // Check if already processed
    if (redis.exists("processed:" + orderId)) {
        return;  // Skip duplicate
    }
    
    // Process
    database.update(record.value());
    
    // Mark as processed
    redis.setex("processed:" + orderId, 86400, "done");
}
```

---

### 📊 Performance Comparison: Classic vs New Protocol

Based on Confluent's benchmarks:

| Metric | Classic Protocol | New Protocol (KIP-848) |
|--------|------------------|------------------------|
| Rebalance time (10 consumers, 900 partitions) | ~103 seconds | ~5 seconds |
| Consumer impact | All consumers | Only affected consumers |
| Processing during rebalance | ❌ Stopped | ✅ Continues |
| Commit during rebalance | ❌ Stopped | ✅ Continues |
| Large group scalability | Poor | Excellent |

---

### Critical Configuration Parameters

| Parameter | Default | What It Controls |
|-----------|---------|------------------|
| `session.timeout.ms` | 45000 (45s) | How long before coordinator considers consumer dead (no heartbeat) |
| `max.poll.interval.ms` | 300000 (5 min) | Max time between poll() calls — exceeded = consumer "too slow" |
| `heartbeat.interval.ms` | 3000 (3s) | How often consumer sends heartbeat |
| `partition.assignment.strategy` | RangeAssignor | Which assignor to use |

**Rule of thumb:** `heartbeat.interval.ms` < `session.timeout.ms` / 3

---

### Generation & Epoch

Every rebalance increments the **generation** (epoch):

```yaml
Generation 0: Initial join (C1, C2, C3)
Generation 1: C3 crashed, rebalance (C1, C2)
Generation 2: C4 joined, rebalance (C1, C2, C3, C4)
```

- Consumers include generation in every request
- Stale requests (old generation) are rejected
- Prevents duplicate processing during rebalance

---

## 💡 PRACTICAL EXAMPLES

### Example 1: E-Commerce Order Processing

```yaml
Topic: orders (10 partitions)
Use Case: Processing customer orders

Scenario A: 5 Consumers
  - Each consumer handles 2 partitions
  - Good for moderate load
  - If one consumer fails, its 2 partitions redistributed

Scenario B: 10 Consumers
  - Each consumer handles 1 partition
  - Maximum parallelism
  - Perfect for high load

Scenario C: 15 Consumers
  - 10 active, 5 standby
  - Instant failover if any consumer crashes
  - Zero downtime during failures
```

### Example 2: Consumer Configuration

```python
consumer_config = {
    'bootstrap.servers': 'kafka:9092',
    'group.id': 'order-processors',
    'partition.assignment.strategy': 'org.apache.kafka.clients.consumer.CooperativeStickyAssignor',
    'session.timeout.ms': 45000,
    'max.poll.interval.ms': 300000,
    'heartbeat.interval.ms': 3000
}
```

---

## 📈 PARTITION COUNT RECOMMENDATIONS

```yaml
General Guidelines:
  - More partitions = more parallelism (but more overhead)
  - Rule of thumb: partitions = consumers × 2-3 (for growth)
  - Maximum recommended: 200-400 partitions per broker
  - Consider rebalance time: more partitions = longer rebalances

Scaling Strategy:
  1. Start with: partitions = expected consumers × 2
  2. Monitor consumer lag
  3. Add partitions if lag persists
  4. Add consumers as needed
```

---

## ❓ Interview Practice

### Question 1: "What happens when you have more consumers than partitions?"

**Architect's Answer:**

> "The extra consumers sit idle. Kafka's rule is one partition per consumer in a group. With 15 consumers and 10 partitions, only 10 are active; 5 are standby. When an active consumer crashes, a standby immediately gets its partitions during rebalance. It's a feature—high availability and instant failover."

### Question 2: "How does Kafka rebalancing work internally?"

**Architect's Answer:**

> "Two phases: JoinGroup and SyncGroup. The Group Coordinator manages the group. On a trigger—consumer join, leave, or crash—all consumers re-join. The coordinator picks a leader; the leader runs the partition assignor and computes the assignment. The leader sends it via SyncGroup; the coordinator distributes it. With eager rebalancing, all consumers stop briefly. With CooperativeStickyAssignor, only partitions that need to move are revoked—minimal downtime."

### Question 3: "When would you use CooperativeStickyAssignor?"

**Architect's Answer:**

> "When you can't afford a full 'stop the world' rebalance. Eager rebalancing revokes all partitions from all consumers—everyone stops, then gets new assignments. With CooperativeSticky, only the partitions that are moving get revoked. Consumers keeping their partitions continue processing. Critical for low-latency or high-throughput systems where a 5–10 second pause is unacceptable."

---

## 🔗 Connecting to Previous Days

| Day | Concept | How It Connects |
|-----|---------|-----------------|
| Day 17 | Instagram Kafka | Like events use consumer groups and partitions |
| Day 22 | 7 Layers | Kafka sits in Layer 5 (Async) |
| Day 31 | Kafka Schema Evolution | Same topic/partition concepts |
| Day 33 | Distributed Tracing | Trace IDs help debug rebalance-related issues |

---

## ✅ Day 34 Action Items

1. **Audit your consumer groups** — How many partitions? Consumers? Any idle?
2. **Try CooperativeStickyAssignor** — Reduce rebalance downtime
3. **Tune timeouts** — session.timeout.ms, max.poll.interval.ms for your workload
4. **Monitor rebalances** — Track frequency and duration
5. **Plan partition count** — partitions = consumers × 2 for growth headroom

---

## 🚀 Key Takeaways

```yaml
What Triggers Rebalance:
  - Consumers joining/leaving
  - Partitions added
  - Timeouts (heartbeat/session)

Major Problems:
  - ❌ Processing pause → Lag spikes
  - ❌ Message duplication
  - ❌ Message loss (auto-commit)
  - ❌ Load imbalance

Solutions:
  - ✅ Tune timeouts (max.poll.interval.ms)
  - ✅ Use StickyAssignor
  - ✅ Manual commits (after processing)
  - ✅ Enable KIP-848 (Kafka 4.0+) — 20x faster!
  - ✅ Make consumers idempotent

Core Rules:
  - Rule #1: One partition → One consumer (in a group)
  - Consumers < Partitions: Some consumers get multiple partitions
  - Consumers = Partitions: Perfect 1:1 mapping (optimal)
  - Consumers > Partitions: Extra consumers sit idle (standby)
  - Rebalancing: JoinGroup → Leader computes → SyncGroup → Assignment distributed
  - CooperativeStickyAssignor: Incremental rebalance, minimal downtime
  - Group Coordinator: One broker per group, manages membership
  - Generation: Incremented on each rebalance, prevents stale requests
```

---

## 🎯 The 30-Second Explanation

> *"Kafka's partition assignment is like assigning seats on a bus: if you have 10 seats (partitions) and 6 passengers (consumers), some passengers get two seats. If you have 10 passengers, everyone gets one seat. If you have 15 passengers, 10 sit and 5 stand by as backups. When a passenger leaves, rebalancing reassigns the seats. Internally: JoinGroup picks a leader, SyncGroup distributes the assignment, and with CooperativeSticky you barely notice the shuffle."*

> **Rebalance analogy:** *"Rebalance is like changing tires on a moving car. The classic protocol stops the car completely. Cooperative keeps rolling on three tires while changing one. KIP-848 changes each tire individually without even slowing down."*

---

*— Sunchit Dudeja*  
*Day 34 of 50: System Design Interview Preparation Series*

---

> 🎯 **Interview Edge:** When asked about Kafka consumer groups, say: "One partition per consumer. Rebalancing happens on join, leave, or crash. Internally it's JoinGroup—coordinator picks a leader—then SyncGroup—leader computes assignment, coordinator distributes. Eager rebalancing stops everyone; CooperativeStickyAssignor does incremental rebalancing so consumers keeping their partitions never pause. I tune session.timeout and max.poll.interval based on processing time." This shows you understand both the model and the internals.
