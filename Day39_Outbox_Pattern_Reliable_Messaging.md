# Outbox Pattern: Reliable Messaging Without Distributed Transactions
### Day 39 of 50 - System Design Interview Preparation Series

**By Sunchit Dudeja**

---

## 🎯 Welcome to Day 39

When a service updates **its database** and must also **emit an event** (Kafka, RabbitMQ, etc.), naive code performs **two independent writes**—a **dual write**. The Outbox pattern fixes this by making **one transactional write** to the database: business rows **plus** an **outbox row**. A **relay** publishes from the outbox later. No **2PC** across DB and broker.

---

## 🚨 The Problem: Dual Writes

```python
# BAD: No atomicity between DB and broker
def create_order(order):
    db.save(order)                    # Step 1
    kafka.send("orders", order)       # Step 2

# Kafka down? → DB committed, event lost
# DB rolls back after send? → ghost event (if send isn’t transactional with DB)
```

**Risks:**

| Failure | Result |
|---------|--------|
| DB commits, message send fails | Inconsistent: state changed, downstream never notified |
| Message sent, DB rolls back | Phantom event (rare if order is DB-then-send, but ordering matters) |
| Partial failure mid-flow | Hard to reason about without a single atomic boundary |

---

## 🏛️ The Solution: Outbox Pattern

**Idea:** Persist the **intent to publish** in the **same database transaction** as the business change. A **separate relay** reads unpublished rows and pushes to the broker, retrying until success.

- **Atomicity:** Either both business data and outbox row commit, or neither.
- **No cross-system 2PC:** Only the database participates in the transaction.
- **At-least-once to the broker:** Relay may retry; consumers must be **idempotent**.

---

## 🔧 How It Works

### Step 1: Write business data + outbox in one transaction

```sql
CREATE TABLE outbox (
    id BIGINT PRIMARY KEY GENERATED ALWAYS AS IDENTITY,
    event_id UUID NOT NULL UNIQUE,
    aggregate_type VARCHAR(100),
    aggregate_id VARCHAR(100),
    event_type VARCHAR(100) NOT NULL,
    payload JSONB NOT NULL,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    published BOOLEAN DEFAULT FALSE
);

BEGIN;

INSERT INTO orders (id, user_id, amount)
VALUES (123, 456, 99.99);

INSERT INTO outbox (event_id, aggregate_type, aggregate_id, event_type, payload)
VALUES (
    gen_random_uuid(),
    'order',
    '123',
    'OrderCreated',
    '{"order_id": 123, "user_id": 456, "amount": 99.99}'::jsonb
);

COMMIT;
```

Use `gen_random_uuid()` / `uuid_generate_v4()` depending on your PostgreSQL extensions. MySQL can use `BINARY(16)` UUIDs or `CHAR(36)`.

### Step 2: Relay process (polling)

```python
import time

def relay_outbox():
    while True:
        rows = db.query("""
            SELECT id, event_id, event_type, payload
            FROM outbox
            WHERE published = FALSE
            ORDER BY id ASC
            LIMIT 100
        """)
        for row in rows:
            try:
                kafka.send(row["event_type"], row["payload"])
                db.execute(
                    "UPDATE outbox SET published = TRUE WHERE id = %s",
                    (row["id"],),
                )
                db.commit()
            except Exception as e:
                log.error("publish failed id=%s: %s", row["id"], e)
                db.rollback()
                break  # retry same batch next tick
        time.sleep(1)
```

Tune batch size, backoff, and **claim** strategy for multiple relay instances (see below).

### Step 3: Broker and consumers

Consumers see **at-least-once** delivery. **Idempotency** (by `event_id`) is mandatory.

---

## 📊 Variants

| Variant | How it works | Pros | Cons |
|---------|----------------|------|------|
| **Polling publisher** | Relay `SELECT … WHERE published = false` | Simple; any RDBMS | Latency; DB read load |
| **CDC / log tailing** | Debezium, Maxwell, DMS read WAL/binlog | Low latency; less app polling | Ops complexity; DB-specific |
| **Relay with lock** | Redis `SETNX` / DB advisory lock so one relay runs a partition | Safe horizontal relays | Lock tuning |

```python
# Sketch: single-flight relay with Redis lock
def relay_with_lock():
    if redis.set("outbox-relay-lock", "1", nx=True, ex=60):
        try:
            process_outbox_batch()
        finally:
            redis.delete("outbox-relay-lock")
```

---

## 🎯 Real-World Example: Order Created

**Without outbox:** `db.save(order)` succeeds; `kafka.send` fails → shipping never hears about the order.

**With outbox:**

```python
def create_order(order):
    with db.transaction():
        db.save(order)
        db.insert_outbox("OrderCreated", order.payload, event_id=new_uuid())
    # Kafka may be down; row stays unpublished until relay succeeds
```

---

## ✅ Benefits (at a Glance)

| Problem | Addressed? | How |
|---------|------------|-----|
| Dual-write inconsistency | Yes | Single DB transaction |
| Lost events (broker down at write time) | Yes | Outbox row durable until published |
| 2PC across DB + broker | Avoided | Broker outside transaction |
| Ordering | Per outbox `id` / time | Relay processes in order (per shard) |
| Retries | Yes | Relay loops until mark published |

---

## ⚠️ Challenges & Mitigations

### 1. Duplicate messages

Relay can publish **twice** if it crashes after `send` but before `published = true`.

**Mitigation:** **Idempotent consumers** using `event_id` (or business idempotency key).

```python
def handle_order_created(event):
    if redis.sismember("processed_events", event["event_id"]):
        return
    process(event)
    redis.sadd("processed_events", event["event_id"])
```

### 2. Outbox table growth

**Mitigation:** Delete or archive **published** rows older than N days (compliance permitting).

```sql
DELETE FROM outbox
WHERE published = TRUE
  AND created_at < NOW() - INTERVAL '7 days';
```

### 3. Ordering across aggregates

Order by **`id`** or **`created_at`** within an aggregate; use **partition key** = `aggregate_id` in Kafka if you need per-aggregate ordering.

---

## 📤 Deeper Dive: Failure Scenarios

| Scenario | What happens | Mitigation |
|----------|----------------|------------|
| Relay crashes after Kafka ACK, before UPDATE | Duplicate publish | Idempotent consumer |
| Consumer throws before offset commit | Retry / DLQ | Backoff + max retries |
| DB disconnect mid-transaction | Rollback | No partial order without outbox row |
| Traffic spike, relay slow | Outbox backlog grows | Scale relay, CDC, alerts on **outbox lag** |

**Monitoring (examples):**

- Count `published = false`
- Age of oldest unpublished row: `NOW() - MIN(created_at) WHERE published = false`

---

## 🧩 Outbox vs 2PC (Two-Phase Commit)

| Aspect | Outbox | 2PC |
|--------|--------|-----|
| Coordinator | None (app + DB txn) | Required |
| Cross-system locks | No | Yes |
| Throughput | Usually higher for events | Often lower |
| Complexity | Moderate (relay + idempotency) | High |
| Consistency model | Eventually broker catches up | Stronger sync semantics (rarely used for DB+Kafka in most stacks) |

**Practical rule:** Prefer **outbox** for **event notification** from an OLTP database. Reserve **2PC-style** designs for narrow domains that truly require them (and understand operational cost).

---

## 🚀 Advanced: Scale the Relay

- **Partition outbox** by hash(`aggregate_id`) or by **aggregate type** (`outbox_orders`, `outbox_payments`).
- **Multiple relays** with **partition claiming** or **advisory locks** so each row is processed once.

---

## 📤 “Crystal Clear” — Simple Mental Model

### The pizza shop analogy

1. **Write the order** in the notebook (database).  
2. **Stick a note** on an internal “pending calls” board (**outbox**)—**same visit, same pen** (same transaction).  
3. A **dispatcher** (relay) calls the driver (broker) **later**. If the phone is dead, the note stays; **no lost order**.

The outbox is a **durable to-do list** inside the database, not a second guess at commit time.

### Without vs with outbox (SQL)

**Broken pattern:** commit order, then call Kafka outside the transaction—any failure splits the story.

**Fixed pattern:**

```sql
BEGIN;
INSERT INTO orders …;
INSERT INTO outbox (…, published = false) …;
COMMIT;
-- Relay: SELECT … FOR UPDATE SKIP LOCKED (optional), publish, UPDATE published
```

### Common confusions

| Confusion | Clarification |
|-----------|----------------|
| “Isn’t this just polling?” | Often yes; **CDC** reduces poll load and latency. |
| “Duplicate after relay crash?” | Yes → **idempotent** consumers. |
| “Outbox forever?” | **TTL/archive** published rows. |
| “Same as a queue?” | **Conceptually** yes—a queue **backed by your DB’s transaction log**. |

### One sentence

**The Outbox pattern saves “messages to send” in a table in the **same transaction** as your business write; a **background process** sends them to the broker and marks them done, so you never rely on the broker succeeding at the exact moment of commit.**

### Forklift analogy

- **Product** = business row.  
- **Package** = outbox row.  
- **Warehouse** = database.  
- **Forklift** = relay.  
- **Truck** = message broker.  

If the truck is broken, packages stay in the warehouse until the next run—**nothing silently disappears**.

---

## 📈 When to Use (and When Not)

**Good fit:**

- Event-driven microservices, **CQRS**, **event sourcing**-style emission
- “Something must happen downstream” when DB state changes
- Broker may be **temporarily unavailable**

**Think twice:**

- Sub-**10 ms** end-to-end from write to consumer (outbox + polling adds delay; **CDC** helps)
- Huge payloads (store **pointer** in outbox, blob in object storage)
- Trivial fire-and-forget with no durability requirement

---

## 📝 End-to-End Python Sketch

Illustrative only—harden transactions, errors, and Kafka `flush()`/`get()` in production.

```python
import json
import time
import uuid
import psycopg2
from kafka import KafkaProducer

class OrderService:
    def __init__(self):
        self.conn = psycopg2.connect("dbname=orders")

    def create_order(self, user_id, amount):
        order_id = uuid.uuid4()
        event_id = uuid.uuid4()
        payload = json.dumps(
            {"order_id": str(order_id), "user_id": user_id, "amount": amount}
        )
        with self.conn:
            with self.conn.cursor() as cur:
                cur.execute(
                    """
                    INSERT INTO orders (id, user_id, amount)
                    VALUES (%s, %s, %s)
                    """,
                    (str(order_id), user_id, amount),
                )
                cur.execute(
                    """
                    INSERT INTO outbox (event_id, aggregate_id, event_type, payload)
                    VALUES (%s, %s, %s, %s::jsonb)
                    """,
                    (str(event_id), str(order_id), "OrderCreated", payload),
                )
        return order_id


class OutboxRelay:
    def __init__(self):
        self.conn = psycopg2.connect("dbname=orders")
        self.producer = KafkaProducer(bootstrap_servers="kafka:9092")

    def run(self):
        while True:
            with self.conn.cursor() as cur:
                cur.execute(
                    """
                    SELECT id, event_type, payload::text
                    FROM outbox
                    WHERE published = FALSE
                    ORDER BY id ASC
                    LIMIT 100
                    """
                )
                for row_id, event_type, payload in cur.fetchall():
                    try:
                        self.producer.send(
                            event_type, value=payload.encode("utf-8")
                        ).get(timeout=10)
                        cur.execute(
                            "UPDATE outbox SET published = TRUE WHERE id = %s",
                            (row_id,),
                        )
                        self.conn.commit()
                    except Exception:
                        self.conn.rollback()
                        time.sleep(1)
                        break
            time.sleep(1)
```

Align `outbox` DDL with your `INSERT` columns (`published`, types).

---

## 🎯 The 30-Second Summary

> *"Dual writes to DB and Kafka can lose events or split state. The **Outbox pattern** inserts the **event payload** in an **outbox table** in the **same transaction** as the business row. A **relay** publishes unpublished rows and marks them published, retrying on failure. You avoid **2PC**; you accept **at-least-once** delivery and build **idempotent** consumers. For lower latency at scale, add **CDC** instead of pure polling."*

---

## 🔗 Connecting to Previous Days

| Day | Concept | How It Connects |
|-----|---------|-----------------|
| Day 27 | Two-phase commit (2PC) | Alternative coordination model; outbox avoids cross-resource 2PC |
| Day 31 | Kafka schema evolution | Events published from outbox still need compatible schemas |
| Day 36 | RabbitMQ vs Kafka | Outbox works with either; relay targets your broker |

---

## ✅ Day 39 Action Items

1. Draw **one** service you own: today’s flow is dual-write or outbox?  
2. If you poll, define **SLO** for max **outbox lag** (seconds + row count).  
3. Add **idempotency key** (`event_id`) to one consumer path and test a **duplicate** delivery.

---

*— Sunchit Dudeja*  
*Day 39 of 50: System Design Interview Preparation Series*
