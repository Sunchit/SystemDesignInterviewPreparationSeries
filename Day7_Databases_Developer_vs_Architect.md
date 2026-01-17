# Databases: Developer vs Architect Thinking
### Day 7 of 50 - System Design Interview Preparation Series

**By Sunchit Dudeja**

---

## 🎯 Welcome to Day 7!

Yesterday, we learned to design for failure like an architect. Today, we reveal how **database choices** separate feature developers from system architects.

> The right database isn't about technology preference — it's about matching data characteristics to storage capabilities.

Let's use **WhatsApp vs Banking Apps** as our battlefield.

---

## 💬 WhatsApp Messages — Two Perspectives

### Developer's Database Choice

```java
@Entity
public class Message {
    @Id
    private Long id;
    private String sender;
    private String receiver;
    private String text;
    private Timestamp timestamp;
}
```

**Developer's thinking:** "Use MySQL. Messages need ACID transactions."

---

### Architect's Reality Check

```
70 billion messages daily
2 billion users across 180 countries  
Requirement: Deliver NOW, sync later
Constraint: Must work on 2G networks
```

The scale changes *everything*.

---

## 🏗️ Architect's Database Ecosystem

### 1. LAYERED STORAGE STRATEGY

| Tier | Database | Purpose |
|------|----------|---------|
| **Tier 1** | Cassandra | Messages — Write optimized, geo-distributed |
| **Tier 2** | HBase | Media metadata — Large binary storage |
| **Tier 3** | Redis | Online status — Sub-millisecond reads |
| **Tier 4** | MySQL | User accounts — Strong consistency needed |

**Why?** Different data, different requirements.

---

### 2. READ/WRITE PATTERN ANALYSIS

| Perspective | Thinking |
|-------------|----------|
| **Developer** | "We need to store and retrieve messages" |
| **Architect** | "Let me analyze the access patterns first" |

**Architect's Analysis:**

#### Write Patterns
```
- 70B writes/day = 800k/second
- Write heavy (10:1 write:read ratio)
- Global writes (Mumbai user → New York recipient)
```

#### Read Patterns
```
- Recent messages (last 7 days) = 90% of reads
- Old messages = <1% of reads
- Group chats explode read complexity
```

---

### 3. CONSISTENCY MODEL DECISION

| Perspective | Choice |
|-------------|--------|
| **Developer default** | Strong consistency (ACID) |
| **Architect's choice** | Eventual Consistency |

**Why Eventual Consistency for WhatsApp?**

```
Message delivered to recipient      → ✅ Instant
Sync to sender's other devices      → within 2 seconds ✅
Update read receipts                → within 5 seconds ✅
Archive to long-term storage        → within 1 hour ✅
```

**Trade-off:** User sees "✓✓" instantly (availability) vs "Message seen" might lag (consistency).

---

## 🏦 Banking Transactions — The Opposite World

Same data storage problem, completely different requirements:

```
Transfer ₹10,000 from A → B

Non-negotiable:
1. A's balance MUST decrease exactly ₹10,000
2. B's balance MUST increase exactly ₹10,000  
3. Both MUST happen simultaneously
4. Transaction MUST be permanent
5. System can be SLOW but must be CORRECT
```

### Architect's Database Stack for Banking

| Component | Technology | Purpose |
|-----------|------------|---------|
| **Primary** | PostgreSQL | 2-phase commit |
| **Replica** | Synchronous replication | Zero data loss |
| **Cache** | Read-only | Account statements |
| **Queue** | Kafka | Audit trail (immutable) |

**Trade-off:** Availability sacrificed for perfect consistency.

---

## 📊 Architect's Decision Matrix

| Requirement | WhatsApp | Banking | Database Choice |
|-------------|----------|---------|-----------------|
| **Consistency** | Eventual (seconds) | Strong (immediate) | Cassandra vs PostgreSQL |
| **Availability** | 99.999% | 99.9% | Multi-region vs Single-region |
| **Durability** | Replicate across 3 DCs | Write-ahead logs | Different replication strategies |
| **Latency** | <200ms delivery | <2s transaction | In-memory vs Disk-optimized |
| **Scale** | 70B messages/day | 1B transactions/day | Horizontal vs Vertical scaling |

---

## 🔧 Real-World: Swiggy's Multi-Database Architecture

| Perspective | Thinking |
|-------------|----------|
| **Developer** | "Food delivery app needs a database" |
| **Architect** | "Food delivery app needs 6 specialized databases" |

**Architect's Design:**

| Data Type | Database | Why? |
|-----------|----------|------|
| Restaurant menus | PostgreSQL | Structured, relational |
| Live order tracking | Redis | Real-time, in-memory |
| User sessions | MongoDB | Flexible schema |
| Analytics | Cassandra | Time-series, write-heavy |
| Search indexes | Elasticsearch | Full-text search |
| Payment transactions | MySQL | ACID compliant |

**Principle:** Each database excels at specific workloads.

---

## 💡 Architect's Database Selection Flowchart

```
Start: What's your data?
│
├── Need ACID transactions?     → PostgreSQL / MySQL
├── Need massive writes?        → Cassandra / ScyllaDB
├── Need flexible schema?       → MongoDB / DynamoDB
├── Need real-time?             → Redis / Memcached
├── Need full-text search?      → Elasticsearch
└── Need time-series?           → InfluxDB / TimescaleDB

Then ask:
├── Read:Write ratio?
├── Data size growth?
├── Consistency requirements?
├── Geographic distribution?
└── Team expertise?
```

---

## 🎯 Database Failure Scenarios

### WhatsApp's Eventual Consistency in Action

```
Scenario: Mumbai data center goes offline
User in Delhi sends message to user in Singapore

Architect's Design:
1. Message routed to Singapore via London (still delivers)
2. Mumbai recovers → messages sync back
3. User sees "delayed" but not "failed"
4. No data loss, just temporary inconsistency
```

### Banking's Strong Consistency

```
Scenario: Primary database fails during transfer

Architect's Design:
1. Transaction ROLLS BACK completely
2. User sees "Transaction failed, try again"
3. No partial transfers
4. Better to fail than corrupt data
```

---

## 📈 Scaling Patterns

| Perspective | Approach |
|-------------|----------|
| **Developer** | "Add more RAM to database server" |
| **Architect** | "Implement a 5-step scaling strategy" |

**Architect's Scaling Strategy:**

```
Step 1: Read replicas          → Scale reads
Step 2: Sharding by user_id    → Scale writes  
Step 3: Multiple database types → Right tool for job
Step 4: Caching layer          → Reduce database load
Step 5: Database per microservice → Isolation
```

### Example: Instagram Sharding Strategy

```
User ID 123456789 → Shard 3 (ID % 10 = 3)
All user data lives on Shard 3
Can scale to 10,000 shards (10K database instances)
```

---

## 🔍 Architect's Observability

| Perspective | Monitors |
|-------------|----------|
| **Developer** | "Is database up?" |
| **Architect** | "8 critical database metrics" |

**Architect's Dashboard:**

| Metric | Target/Threshold |
|--------|------------------|
| Replication lag | < 200ms |
| Connection pool utilization | 80% threshold |
| Query latency | 95th percentile |
| Cache hit ratio | > 90% target |
| Deadlocks per minute | Minimal |
| Storage growth rate | Predictable |
| Backup success rate | 100% |
| Failover test frequency | Regular |

---

## 🧪 Exercise: Design Database for Netflix

| Perspective | Approach |
|-------------|----------|
| **Developer** | "MySQL for everything" |
| **Architect** | "7 specialized databases" |

**Architect's Netflix Design:**

| Data Type | Database | Justification |
|-----------|----------|---------------|
| User profiles | Cassandra | Global distribution |
| Viewing history | Cassandra | Write-heavy |
| Video metadata | PostgreSQL | Structured |
| Recommendations | Redis | Real-time updates |
| Billing | MySQL | ACID transactions |
| Search | Elasticsearch | Full-text |
| Analytics | Redshift | Data warehousing |

---

## 🎯 Your Turn: Design Uber's Database Architecture

**Challenge:** Design Uber's database architecture with **5 different databases** and justify each choice.

Think about:
- Rider/Driver locations (real-time)
- Trip history (write-heavy)
- Payments (ACID)
- Search (geo-spatial)
- Surge pricing analytics

---

## ❓ Interview Practice

### Question 1:
> "How would you choose between SQL and NoSQL for a messaging system?"

**Answer:**
> "For a messaging system like WhatsApp, I'd choose NoSQL (Cassandra) because of the access patterns. Messages are write-heavy (10:1 ratio), need horizontal scaling for billions of messages daily, and can tolerate eventual consistency since users expect instant delivery but don't need real-time sync across all devices. The key insight is that messages are naturally partitioned by conversation, making sharding straightforward."

### Question 2:
> "Why would you use multiple databases in a single application?"

**Answer:**
> "This is called polyglot persistence. Different data has different characteristics. In a food delivery app like Swiggy, restaurant menus need relational joins (PostgreSQL), live tracking needs sub-millisecond reads (Redis), search needs full-text indexing (Elasticsearch), and payments need ACID guarantees (MySQL). Using a single database forces compromises. Using specialized databases means each data type is stored optimally."

---

## 🔗 Connecting to Previous Days

| Day | Concept | How It Connects |
|-----|---------|-----------------|
| Day 2 | SCALED | Different databases optimize for different SCALED properties |
| Day 3 | NFRs | Non-functional requirements drive database selection |
| Day 5 | Capacity | Database choice affects capacity planning dramatically |
| Day 6 | Failure | Database failures require different recovery strategies |

---

## ✅ Day 7 Action Items

1. **Map your data types** — List all data in your current project and categorize by access patterns
2. **Analyze read:write ratio** — Measure actual patterns, not assumptions
3. **Learn CAP theorem** — Understand the consistency-availability trade-off
4. **Study one NoSQL database** — Pick Cassandra, MongoDB, or Redis and understand its strengths
5. **Design a polyglot architecture** — Practice selecting databases for a complex app

---

## 🚀 Key Architect Principles

| Principle | What It Means |
|-----------|---------------|
| **One size doesn't fit all** | Polyglot persistence is the way |
| **Schema later** | Start simple, evolve based on access patterns |
| **Design for access patterns** | Not just storage |
| **Consistency is a spectrum** | Choose based on business needs |
| **Failure is expected** | Plan for database outages |

---

## 💡 Key Takeaway

> **Developer: "How do I store this data?"**  
> **Architect: "How will this data be accessed at 10x scale during a partial outage in a region where half my users are on 2G?"**

The difference between a developer and an architect isn't just experience — it's **perspective**. Architects think about access patterns, failure modes, and scale from day one.

---

*— Sunchit Dudeja*  
*Day 7 of 50: System Design Interview Preparation Series*

---

> 🎯 **Interview Edge:** When asked about database design, don't just name a technology. Say: "Let me analyze the access patterns first — what's the read:write ratio, consistency requirements, and expected scale?" Then map requirements to database characteristics. This shows you think like an architect, not a developer picking familiar tools.

