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

## 🔄 REBALANCING: Kafka Internals in Detail

### What Triggers a Rebalance?

| Trigger | What Happens |
|---------|--------------|
| **Consumer joins** | New consumer wants to join the group |
| **Consumer leaves** | Consumer shuts down gracefully |
| **Consumer crashes** | No heartbeat within `session.timeout.ms` |
| **Consumer too slow** | Exceeds `max.poll.interval.ms` between polls |
| **Topic metadata change** | New partitions added to a subscribed topic |

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
│  2. Coordinator detects: C3 missed session.timeout.ms (default 10s)    │
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

### Critical Configuration Parameters

| Parameter | Default | What It Controls |
|-----------|---------|------------------|
| `session.timeout.ms` | 10000 (10s) | How long before coordinator considers consumer dead (no heartbeat) |
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
1. Rule #1: One partition → One consumer (in a group)
2. Consumers < Partitions: Some consumers get multiple partitions
3. Consumers = Partitions: Perfect 1:1 mapping (optimal)
4. Consumers > Partitions: Extra consumers sit idle (standby)
5. Rebalancing: JoinGroup → Leader computes → SyncGroup → Assignment distributed
6. CooperativeStickyAssignor: Incremental rebalance, minimal downtime
7. Group Coordinator: One broker per group, manages membership
8. Generation: Incremented on each rebalance, prevents stale requests
```

---

## 🎯 The 30-Second Explanation

> *"Kafka's partition assignment is like assigning seats on a bus: if you have 10 seats (partitions) and 6 passengers (consumers), some passengers get two seats. If you have 10 passengers, everyone gets one seat. If you have 15 passengers, 10 sit and 5 stand by as backups. When a passenger leaves, rebalancing reassigns the seats. Internally: JoinGroup picks a leader, SyncGroup distributes the assignment, and with CooperativeSticky you barely notice the shuffle."*

---

*— Sunchit Dudeja*  
*Day 34 of 50: System Design Interview Preparation Series*

---

> 🎯 **Interview Edge:** When asked about Kafka consumer groups, say: "One partition per consumer. Rebalancing happens on join, leave, or crash. Internally it's JoinGroup—coordinator picks a leader—then SyncGroup—leader computes assignment, coordinator distributes. Eager rebalancing stops everyone; CooperativeStickyAssignor does incremental rebalancing so consumers keeping their partitions never pause. I tune session.timeout and max.poll.interval based on processing time." This shows you understand both the model and the internals.
