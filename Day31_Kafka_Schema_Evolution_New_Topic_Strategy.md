# Kafka Schema Evolution: The "New Topic" Strategy for Breaking Changes
### Day 31 of 50 - System Design Interview Preparation Series

**By Sunchit Dudeja**

---

## 🎯 Welcome to Day 31!

Yesterday, we mastered Database Replication. Today, we tackle one of the trickiest problems in event-driven architecture: **evolving your Kafka schema when you have breaking, backward-incompatible changes**.

> Your data model will change. The question isn't *if*—it's *how* you migrate without losing messages or breaking consumers.

Let me show you **two production-grade strategies**: the **Blue-Green Deployment with Dual-Writes** (gold standard for critical systems) and the **New Topic with Long Retention** (simpler for coordinated cutovers)—both with zero data loss.

---

## 🚨 THE PROBLEM: Breaking Schema Changes

### The Nightmare Scenario

You have a Kafka topic `user-events` with millions of messages flowing daily:

```json
// Current schema (v1)
{
  "user_id": "12345",
  "name": "John Doe",
  "email": "john@example.com",
  "timestamp": "2024-03-10T10:00:00Z"
}
```

**Product says:** "We need to support international names. Split `name` into `first_name` and `last_name`."

```json
// New schema (v2) - BREAKING CHANGE
{
  "user_id": "12345",
  "first_name": "John",
  "last_name": "Doe",
  "email": "john@example.com",
  "timestamp": "2024-03-10T10:00:00Z"
}
```

**The problem:** Old consumers expect `name`. New producers send `first_name` + `last_name`. They're **incompatible**.

| Approach | What Happens |
|----------|--------------|
| Just change the schema | Old consumers crash parsing new messages |
| Stop all producers | Downtime, lost events |
| Run two formats simultaneously | Consumer logic becomes a nightmare |

**Architect's Rule:** Breaking schema changes need a migration strategy—not a "just deploy it" approach.

---

## 📋 SCHEMA EVOLUTION: The Three Strategies

Before we dive deep, here's the landscape:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    KAFKA SCHEMA EVOLUTION STRATEGIES                    │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  STRATEGY 1              STRATEGY 2              STRATEGY 3             │
│  Blue-Green + Dual-Write New Topic              Backward-Compatible     │
│  (Gold Standard)         (Long Retention)       (Same Topic)           │
│       │                       │                        │               │
│       ▼                       ▼                        ▼               │
│  Dual-write + MM2         Producers first        Add optional fields   │
│  Rolling consumers       Consumers migrate      Don't remove fields    │
│  Zero downtime           Simpler cutover        Additive changes only │
│  Best safety             Clean, planned         Lowest complexity     │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

| Strategy | Use Case | Complexity | Data Loss Risk |
|----------|----------|------------|----------------|
| **1. Blue-Green + Dual-Write** | Critical systems, zero downtime | High | None (with MM2 + dual-write) |
| **2. New Topic** | Breaking change, clean cutover | Medium | None (with long retention) |
| **3. Backward-Compatible** | Add optional field, rename with alias | Low | None |

---

## 🏆 STRATEGY 1: Blue-Green Deployment with Dual-Writes (The Gold Standard)

This is the **safest and most common approach for critical systems**. It treats the new topic/cluster as a completely separate environment and migrates clients in a controlled, rolling fashion. The core idea: **run both the old and new systems in parallel** for a period of time.

### Step 1: Dual-Writes

Modify your producer application to write **every message to both the old and the new topics simultaneously**.

```java
// Producer writes to BOTH topics
public void sendUserEvent(UserEventV2 event) {
    // Old topic (v1 schema - transform if needed)
    kafkaTemplate.send("user-events", event.getUserId(), toV1Format(event));
    // New topic (v2 schema)
    kafkaTemplate.send("user-events-v2", event.getUserId(), event);
}
```

**Why:** The new topic starts to populate with live data while the old one continues to serve all existing consumers. No consumer changes yet—zero disruption.

### Step 2: Backfill Historical Data

For data that existed **before** you started dual-writes, use [Apache Kafka MirrorMaker 2 (MM2)](https://kafka.apache.org/documentation/#georeplication) to copy historical data from the old topic to the new one.

```yaml
What MM2 does:
  - Consumes from user-events (old)
  - Produces to user-events-v2 (new)
  - Copies all messages that existed before dual-writes began
  - New topic now has: historical (MM2) + live (dual-writes)
```

*(See Step 6 in the Migration Playbook below for full MM2 configuration.)*

### Step 3: Migrate Consumers One by One

With both topics live and in sync (old data via backfill, new data via dual-writes), perform a **rolling restart** of your consumer applications:

1. Update consumer config to point to the new topic
2. Restart the consumer instance
3. Verify it is healthy
4. Repeat for the next instance

**The magic:** Old consumers keep running—**zero downtime**. New consumers start reading from the point the old ones left off, thanks to **careful offset management**:

- **MM2** can sync consumer group offsets from old cluster to new (when replicating across clusters)
- **Manual mapping:** Before switching, record the consumer group's committed offset on the old topic; configure the new consumer to start from the equivalent offset on the new topic
- **From beginning:** If the new topic has full history (backfill + dual-writes), new consumers can start from `earliest` and reprocess—useful for idempotent consumers

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    BLUE-GREEN DUAL-WRITE TIMELINE                        │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Phase 1: Dual-Writes Start                                             │
│  ───────────────────────────                                            │
│  Producers → user-events (old) + user-events-v2 (new)                  │
│  Consumers → still on user-events (old)                                 │
│                                                                         │
│  Phase 2: MM2 Backfill (parallel)                                       │
│  ────────────────────────────────                                       │
│  MM2: user-events (historical) → user-events-v2                         │
│  New topic now has: full history + live dual-writes                     │
│                                                                         │
│  Phase 3: Rolling Consumer Migration                                    │
│  ────────────────────────────────────                                    │
│  Consumer 1: user-events → user-events-v2 (restart, verify)             │
│  Consumer 2: user-events → user-events-v2 (restart, verify)             │
│  ... zero downtime, old consumers still serving traffic                 │
│                                                                         │
│  Phase 4: Stop Dual-Writes                                              │
│  ──────────────────────────                                             │
│  Producers → user-events-v2 only                                        │
│  Deprecate user-events (old)                                            │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Trade-offs

| Pros | Cons |
|------|------|
| Zero downtime | Most complex to implement |
| Best safety—both topics in sync | Dual-write adds latency and failure modes |
| Rolling migration—no big bang | Must handle partial writes (one topic succeeds, one fails) |
| Old consumers keep running during migration | Operational burden: two pipelines to monitor |

**Architect's take:** Use Strategy 1 for **payment systems, order processing, or any system where downtime is unacceptable**. Use Strategy 2 (New Topic) when you can tolerate a coordinated cutover and want simpler operations.

---

## 📦 STRATEGY 2: The "New Topic" with Long Retention

### When to Use It

```yaml
Ideal for:
  - Breaking changes (field rename, split, type change)
  - Not backward-compatible
  - Well-planned migrations
  - You want a clean cutover (not dual-write complexity)

Example breaking changes:
  - name → first_name + last_name (field split)
  - user_id: string → user_id: uuid (type change)
  - Removing a required field
  - Changing nested structure
```

### How It Works

> **📐 Excalidraw Diagram:** Open [kafka-schema-evolution-new-topic.excalidraw](./kafka-schema-evolution-new-topic.excalidraw) at [excalidraw.com](https://excalidraw.com) (File → Open) for an interactive diagram on dark background.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    NEW TOPIC MIGRATION TIMELINE                         │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  PHASE 1: Producers First                                               │
│  ─────────────────────────                                              │
│  user-events (old)          user-events-v2 (new, long retention)        │
│  [Consumers still here]     [Producers start writing here]              │
│       │                              │                                   │
│       │    Producers deploy          │  Retention: 7-14 days              │
│       │    ─────────────────────────►  All messages stored safely      │
│       │                              │                                   │
│                                                                         │
│  PHASE 2: Transition Period (Days 1-7)                                  │
│  ────────────────────────────────────                                   │
│  Old topic: No new data (or deprecated)                                  │
│  New topic: Accumulating backlog                                        │
│  Consumers: Still on old topic, draining remaining messages              │
│                                                                         │
│  PHASE 3: Consumers Migrate                                             │
│  ──────────────────────────                                             │
│  Consumers updated to read from user-events-v2                           │
│  They consume the ENTIRE backlog (transition period + new)              │
│  Zero data loss!                                                         │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### The Key Insight: Long Retention is Your Safety Net

```yaml
Why long retention matters:
  - Producers switch to new topic on Day 1
  - Consumers might take 3-5 days to upgrade (rolling deploy, validation)
  - During those days: ALL messages go to new topic only
  - Without long retention: Messages produced Day 1-5 would be GONE before consumers arrive
  - With 7-14 day retention: Backlog is waiting. Consumers catch up. Zero loss.
```

**Real-World Analogy:** *It's like moving to a new office. You tell everyone "we're at the new address starting Monday." Some people might not get the memo until Friday. The new office (long retention) holds all the mail that arrived Monday–Friday. When they finally show up, nothing's lost.*

---

## 🔧 STEP-BY-STEP: The Migration Playbook

### Step 1: Create the New Topic

```bash
# Create new topic with extended retention
kafka-topics.sh --create \
  --topic user-events-v2 \
  --partitions 12 \
  --replication-factor 3 \
  --config retention.ms=1209600000
  # 14 days = 14 * 24 * 60 * 60 * 1000
```

```yaml
Configuration:
  retention.ms: 1209600000  # 14 days (adjust based on migration timeline)
  partitions: Match or exceed old topic (for throughput)
  replication-factor: Same as production (typically 3)
```

### Step 2: Update Producers First

```java
// Producer deploys with new schema to NEW topic
@Configuration
public class KafkaProducerConfig {
    
    @Value("${kafka.topic.user-events}")
    private String userEventsTopic;  // Now: user-events-v2
    
    public void sendUserEvent(UserEventV2 event) {
        kafkaTemplate.send(userEventsTopic, event.getUserId(), event);
    }
}
```

**Critical:** Producers must write to the new topic. Old topic receives no new data.

### Step 3: Consumers Stay on Old Topic (Temporarily)

```java
// Consumers still reading from OLD topic
@KafkaListener(topics = "user-events", groupId = "user-event-processors")
public void processUserEvent(ConsumerRecord<String, String> record) {
    UserEventV1 event = objectMapper.readValue(record.value(), UserEventV1.class);
    // Process with old schema
    processEvent(event);
}
```

During transition: Old topic drains. New topic accumulates. No consumer reads new topic yet.

### Step 4: Migrate Consumers (When Ready)

```java
// Updated consumer - reads from NEW topic
@KafkaListener(topics = "user-events-v2", groupId = "user-event-processors")
public void processUserEventV2(ConsumerRecord<String, String> record) {
    UserEventV2 event = objectMapper.readValue(record.value(), UserEventV2.class);
    // Process with new schema (first_name, last_name)
    processEvent(event);
}
```

**Magic moment:** Consumer starts from offset 0 (or earliest). It processes the entire backlog from the transition period. No messages lost.

### Step 5: Deprecate Old Topic

Once all consumer groups have migrated and old topic is drained:

```bash
# Optional: Delete old topic after verification
kafka-topics.sh --delete --topic user-events
```

### Step 6 (Optional): Backfill Historical Data

**When you need it:** For data that existed *before* you started writing to the new topic—or before dual-writes began—consumers switching to the new topic won't see it. If your consumers need the full history (e.g., analytics rebuilding a dataset, replay jobs, or audit requirements), you must copy that historical data from the old topic to the new one.

**The tool:** [Apache Kafka MirrorMaker 2 (MM2)](https://kafka.apache.org/documentation/#georeplication) is the standard tool for replicating data between Kafka clusters—or between topics within the same cluster. It's designed for geo-replication but works perfectly for topic-to-topic backfill.

#### How MM2 Works for Backfill

```yaml
What MM2 does:
  - Connects to source (old topic) and target (new topic)
  - Consumes from old topic, produces to new topic
  - Preserves partition mapping (or remaps as configured)
  - Tracks offset sync for exactly-once semantics
  - Can run one-time (historical copy) or continuous (dual-write style)
```

#### MM2 Configuration Example

For cross-cluster or same-cluster backfill (old topic → new topic):

```properties
# mm2-backfill.properties
clusters = source, target
source.bootstrap.servers = kafka-1:9092,kafka-2:9092
target.bootstrap.servers = kafka-1:9092,kafka-2:9092

# Enable replication: user-events (old) → target cluster
source->target.enabled = true
source->target.topics = user-events

# Sync consumer group offsets (optional, for exactly-once)
source->target.sync.group.offsets.enabled = true
```

Run MM2 in dedicated mode:

```bash
./bin/connect-mirror-maker.sh mm2-backfill.properties
```

**Note:** MM2 replicates topics to the target cluster. By default, MM2 may add a source-cluster prefix to topic names (e.g., `source.user-events`). For same-cluster backfill where source and target point to the same brokers, the replicated topic lands with that prefix—you can consume from it directly, or use a Kafka Streams job to copy `source.user-events` → `user-events-v2` if you need an exact topic name. Refer to [Kafka Geo-Replication docs](https://kafka.apache.org/documentation/#georeplication) for your MM2 version's topic naming behavior.

#### Schema Transformation Caveat

**Important:** MM2 copies bytes—it does *not* transform schema. If your old topic has `name` and your new topic expects `first_name` + `last_name`, you have two options:

| Approach | How | When to Use |
|----------|-----|--------------|
| **Kafka Streams / ksqlDB** | Read old topic, transform, write to new topic | Need schema conversion during backfill |
| **MM2 + transform consumer** | MM2 copies; separate consumer transforms and re-publishes | Complex transformations |
| **Pre-transformed backfill** | Custom script: consume old, parse v1, convert to v2, produce to new | Full control, one-off migration |

For simple backfill where you're okay with old-format messages in the new topic (and consumers handle both during transition), MM2 alone works. For schema conversion, add a transformation layer.

#### When to Run Backfill in the Timeline

```yaml
Option A: Before producer switch
  - Run MM2 to copy old → new (historical data)
  - Then producers start writing to new topic
  - New topic now has: historical (from MM2) + fresh (from producers)
  - Consumers migrate and get everything

Option B: During dual-write (if using Strategy 1 - Blue-Green)
  - Producers dual-write to old + new
  - MM2 backfills historical from old → new
  - Once backfill catches up, stop writing to old
  - Consumers migrate to new
```

**Architect's take:** For the New Topic strategy, run MM2 backfill *before* or *in parallel with* producer deployment. Ensure the new topic has historical data before consumers switch—otherwise they'll miss everything produced before the cutover.

---

## 📊 Strategy 1 (Blue-Green) vs Strategy 2 (New Topic): When to Use Which

| Aspect | Strategy 1: Blue-Green + Dual-Write | Strategy 2: New Topic + Long Retention |
|--------|-------------------------------------|---------------------------------------|
| **Producers** | Write to BOTH topics | Write to ONE topic (new) |
| **Consumers** | Rolling migration, zero downtime | Coordinated migration |
| **Pipeline complexity** | Two parallel pipelines | Single pipeline |
| **Sync issues** | Risk of partial writes | None |
| **Duration** | Maintain dual-write for weeks | Clean cutover (days) |
| **Rollback** | Complex (two sources) | Simpler (revert consumer config) |
| **Operational burden** | High | Lower |
| **Best for** | Critical systems, zero downtime | Well-planned migrations, simpler ops |

**Architect's take:** Strategy 1 (Blue-Green) is the gold standard for *payment systems, order processing, or any system where downtime is unacceptable*—rolling consumer migration with both topics in sync. Strategy 2 (New Topic) is simpler when you can tolerate a coordinated cutover and want less operational complexity.

---

## ⏱️ Retention Window: How Long?

```yaml
Rule of thumb:
  retention = (consumer_migration_time) + (buffer)

Examples:
  - Fast migration (1-2 days):  retention = 7 days
  - Rolling deploy (3-5 days): retention = 14 days  
  - Multi-team coordination:   retention = 21 days
  - Compliance/audit needs:    retention = 30+ days

Cost consideration:
  - Longer retention = more disk = higher cost
  - Balance: Enough to cover migration + 2x buffer
```

---

## 🏢 REAL-WORLD EXAMPLE: E-Commerce Order Events

```yaml
Scenario:
  Old: order_id (string), amount (number)
  New: order_id (UUID), amount (Decimal), currency (required)

Breaking: Type changes + new required field

Migration:
  Day 0:  Create orders-v2 topic, retention=14 days
  Day 1:  Producers deploy, write to orders-v2 only
  Day 1-5: Order service, Payment service, Analytics—all still on orders (old)
  Day 6:  Order service migrates to orders-v2
  Day 7:  Payment service migrates
  Day 8:  Analytics migrates (consumes full backlog)
  Day 15: Deprecate orders (old) topic
```

---

## ❓ Interview Practice

### Question 1: "How do you handle a breaking schema change in Kafka?"

**Architect's Answer:**

> "Two main approaches. For critical systems—payments, orders—I use Blue-Green: dual-writes to both topics, MM2 to backfill historical data, then rolling consumer migration. Zero downtime. For simpler migrations, I use the New Topic strategy: create a new topic with 14-day retention, producers switch first, consumers migrate when ready. The long retention holds the backlog so nothing's lost. I choose based on whether we need zero downtime or simpler operations."

### Question 2: "When would you use Blue-Green (dual-write) vs New Topic strategy?"

**Architect's Answer:**

> "For critical systems—payments, orders, anything where downtime is unacceptable—I use the Blue-Green approach: dual-writes, MM2 backfill, and rolling consumer migration. Both topics stay in sync; you migrate consumers one by one with zero downtime. For well-planned migrations where I can coordinate a cutover, the New Topic strategy is simpler: producers switch first, consumers migrate when ready, long retention holds the backlog. Less operational burden, single pipeline. Trade-off: Blue-Green gives you rolling migration and maximum safety; New Topic gives you simplicity."

### Question 3: "What if a consumer can't migrate within the retention window?"

**Architect's Answer:**

> "Then you have a problem. Options: (1) Extend retention before migration—Kafka allows increasing retention. (2) Add a bridge consumer that reads from new topic, transforms to old format, and writes to a temporary topic—last resort. (3) Plan better—retention should be migration_time + 2x buffer. The key is sizing retention correctly during the planning phase."

### Question 4: "How do you handle historical data when migrating to a new topic?"

**Architect's Answer:**

> "For data that existed before the migration—or before dual-writes started—you need to backfill. We use Apache Kafka's MirrorMaker 2 (MM2) to copy historical data from the old topic to the new one. MM2 consumes from the source topic and produces to the target, preserving offsets and partition mapping. Run it before or in parallel with producer deployment so the new topic has the full history when consumers switch. One caveat: MM2 copies bytes—it doesn't transform schema. If you need to convert `name` to `first_name` + `last_name`, you'd add a Kafka Streams or ksqlDB transformation layer, or a custom consumer that transforms during the backfill."

---

## 🔗 Connecting to Previous Days

| Day | Concept | How It Connects |
|-----|---------|-----------------|
| Day 2 | SCALED | Schema evolution affects Scalability (partitioning) and Durability |
| Day 6 | Design for Failure | Migration must not lose data—retention is the safety net |
| Day 17 | Instagram Kafka | Like events use schema—evolving it needs same strategies |
| Day 22 | 7 Layers | Kafka sits in Layer 5 (Async)—schema changes affect all consumers |
| Day 25 | Deployment | Schema migration is a deployment strategy for event pipelines |

---

## ✅ Day 31 Action Items

1. **Audit your Kafka topics** — Which have evolved? Any breaking changes?
2. **Document your schema** — Use Schema Registry (Avro/Protobuf) for versioning
3. **Plan retention** — Set retention based on migration timelines
4. **Practice the playbook** — Run a dry-run migration in staging
5. **Consider Schema Registry** — Confluent Schema Registry enforces compatibility
6. **Plan historical backfill** — If consumers need full history, use MirrorMaker 2 (MM2) to copy old topic → new topic before cutover

---

## 🚀 Key Architect Principles

| Principle | What It Means |
|-----------|---------------|
| **Breaking changes need a plan** | Never "just deploy" a schema change |
| **Strategy 1 (Blue-Green)** | Dual-writes + MM2 backfill + rolling consumers = zero downtime for critical systems |
| **Strategy 2 (New Topic)** | Producers first, long retention = simpler cutover for well-planned migrations |
| **Backfill historical data** | Use MirrorMaker 2 (MM2) to copy pre-migration data from old topic → new topic |
| **Retention is your safety net** | Long enough = zero data loss (Strategy 2) |
| **Well-planned beats reactive** | Schedule migration, don't scramble |

---

## 💡 Key Takeaway

> **"For critical systems: Blue-Green with dual-writes, MM2 backfill, and rolling consumer migration—zero downtime, maximum safety. For simpler migrations: New Topic with long retention—producers first, consumers when ready, single pipeline."**

Choose Strategy 1 (Blue-Green) when downtime is unacceptable. Choose Strategy 2 (New Topic) when you want simpler operations and can coordinate a cutover.

---

*— Sunchit Dudeja*  
*Day 31 of 50: System Design Interview Preparation Series*

---

> 🎯 **Interview Edge:** When asked about Kafka schema evolution, don't just say "we use Schema Registry." Break it down: "For additive changes, backward-compatible schemas. For breaking changes, two options: (1) Blue-Green with dual-writes, MM2 backfill, and rolling consumer migration—zero downtime, best for critical systems. (2) New Topic with long retention—producers first, consumers migrate when ready, simpler for coordinated cutovers. I choose based on whether we can tolerate downtime and how much operational complexity we want." This shows you understand both strategies and their trade-offs.
