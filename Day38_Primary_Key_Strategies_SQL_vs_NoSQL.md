# Primary Key Strategies: SQL vs NoSQL
### Day 38 of 50 - System Design Interview Preparation Series

**By Sunchit Dudeja**

---

## 🎯 Welcome to Day 38

**The core difference:** SQL databases often use **sequential primary keys**—and **gaps are normal**. Many NoSQL systems use **distributed unique identifiers** (UUID, ObjectId, etc.) where there is **no sequential “gap” concept**—IDs are opaque tokens, not a shared counter.

This post explains **why** that difference exists and what it means for **architecture**, **sharding**, and **product requirements**.

---

## 📊 Visual Comparison

### SQL: Sequential primary keys with gaps

```sql
-- MySQL / PostgreSQL style AUTO_INCREMENT / SERIAL
CREATE TABLE orders (
    id INT PRIMARY KEY AUTO_INCREMENT,
    user_id INT,
    amount DECIMAL
);

INSERT INTO orders (user_id, amount) VALUES (1, 99.99);
INSERT INTO orders (user_id, amount) VALUES (2, 149.99);
INSERT INTO orders (user_id, amount) VALUES (3, 49.99);
-- IDs: 1, 2, 3 (sequential)

-- After DELETE
DELETE FROM orders WHERE id = 2;
-- Remaining ids: 1, 3 (gap at 2)

-- After ROLLBACK
BEGIN;
INSERT INTO orders (user_id, amount) VALUES (4, 199.99);
ROLLBACK;
-- Next successful INSERT may use 5 — gap at 4 (behavior depends on engine/settings)
```

### NoSQL: Distributed IDs — no “gap” semantics

```javascript
// MongoDB ObjectId example
{
  "_id": ObjectId("507f1f77bcf86cd799439011"),
  "user_id": 1,
  "amount": 99.99
}
{
  "_id": ObjectId("507f1f77bcf86cd799439012"),
  "user_id": 2,
  "amount": 149.99
}
// No shared sequence — uniqueness, not “next number”
```

---

## 🏛️ SQL: Why Gaps Exist

### Reason 1: Transaction isolation

```sql
-- Transaction A
BEGIN;
INSERT INTO orders (user_id, amount) VALUES (1, 99.99);  -- may reserve id 101
-- not committed yet
ROLLBACK;  -- id 101 may never appear in the table → gap

-- Transaction B (concurrent)
BEGIN;
INSERT INTO orders (user_id, amount) VALUES (2, 149.99);  -- may take 102
COMMIT;
```

**Why:**

- Identifiers are often allocated at **insert time**; rolled-back transactions may **consume** sequence values that never become visible rows.
- Engines differ in details, but **gaps after rollback** are common and usually **acceptable**.

### Reason 2: Performance (concurrency)

**Sequential ID strategies:**

1. **Table-level counter (e.g. older MyISAM patterns):** can serialize writes.
2. **Gap-based / range allocation (e.g. InnoDB-style auto-increment batches):** pre-allocates ranges so inserts don’t contend on every row—unused values in a range can appear as **gaps**.

### Reason 3: Replication and clustering

```sql
-- Illustrative multi-writer pattern (odd/even split — one common trick)
-- Master A: AUTO_INCREMENT_INCREMENT = 2, OFFSET = 1  → 1, 3, 5, …
-- Master B: AUTO_INCREMENT_INCREMENT = 2, OFFSET = 2  → 2, 4, 6, …
```

Merging streams can still look “dense,” but failover, offline writers, and manual fixes introduce **holes** in practice.

---

## 🗄️ NoSQL: Why There Are No “Gaps” in the SQL Sense

### Reason 1: Distributed unique IDs

```text
MongoDB ObjectId (12 bytes), conceptually:

ObjectId("507f1f77bcf86cd799439011")
          └ 4 bytes timestamp ─┘└ machine ─┘└ pid ┘└ counter ┘
```

No single global `NEXTVAL`—so there is **no sequence to break** with a “gap.”

### Reason 2: Client-side or node-local generation

```python
import uuid

# Style: application generates id
# Cassandra / CQL
# INSERT INTO orders (id, user_id, amount) VALUES (uuid(), 1, 99.99);

# DynamoDB-style document
{
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "user_id": 1,
    "amount": 99.99
}
```

```javascript
// MongoDB — driver generates ObjectId if omitted
db.orders.insertOne({
  "_id": ObjectId(),
  "user_id": 1,
  "amount": 99.99
});
```

### Reason 3: No sequence semantics

```yaml
Typical NoSQL stance:
  - IDs are opaque tokens
  - "Next" / "previous" ID is not a product requirement
  - Ordering for UX uses timestamps or version fields, not id ordering
```

---

## 📊 Comparison Table

| Aspect | SQL (`AUTO_INCREMENT` / `SERIAL`) | NoSQL (UUID / ObjectId / similar) |
|--------|-----------------------------------|-----------------------------------|
| **ID format** | Sequential integers (usually) | Random-looking bytes / strings |
| **Gaps** | Common (rollback, delete, some replication patterns) | No shared counter → no “gap” concept |
| **Ordering** | Often correlated with insert order (not a contract) | Not for UUID v4; ObjectId has time prefix but don’t rely on it for business order |
| **Generation** | Server-side, centralized | Client, driver, or node-local |
| **Concurrency** | Coordination / ranges on server | Highly parallel |
| **Sharding** | Monotonic ints can hot-spot “latest” ranges | `hash(id)` often spreads load |
| **Security** | Easier to guess next id if exposed | Opaque ids harder to enumerate |
| **Storage** | Often 4–8 bytes | Often 16+ bytes (UUID/ObjectId) |

---

## 🎯 Why SQL Systems Tolerate Gaps

1. **Simplicity:** Integer PKs are easy to debug and join: `WHERE id = 12345`.
2. **Index locality:** Sequential inserts can be friendly to B-tree pages (workload-dependent).
3. **Business:** Human-facing **invoice** or **ticket** numbers often want monotonic sequences—often implemented as a **separate** concern (see below).

---

## 🎯 Why NoSQL Favors Opaque Distributed IDs

1. **Distributed scale:** No single coordinator issuing every id.
2. **Sharding:** `shard = hash(id) % N` avoids funneling all new keys to one range (naïve monotonic ids can hot-spot).
3. **Throughput:** Less global coordination per insert.

**Caveat:** Monotonic **snowflake**-style IDs are popular too—they’re not “SQL vs NoSQL” only; pick per **ordering**, **size**, and **coordination** needs.

---

## 📝 Code Examples

### PostgreSQL — gaps visible

```sql
CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    user_id INT,
    amount DECIMAL
);

INSERT INTO orders (user_id, amount) VALUES (1, 99.99);
INSERT INTO orders (user_id, amount) VALUES (2, 149.99);
INSERT INTO orders (user_id, amount) VALUES (3, 49.99);

DELETE FROM orders WHERE id = 2;

BEGIN;
INSERT INTO orders (user_id, amount) VALUES (4, 199.99);
ROLLBACK;

-- Subsequent inserts may leave holes at 2, 4, etc. (exact behavior: engine-specific)
SELECT * FROM orders ORDER BY id;
```

### MongoDB — no sequence to gap

```javascript
db.orders.insertMany([
  { user_id: 1, amount: 99.99 },
  { user_id: 2, amount: 149.99 },
  { user_id: 3, amount: 49.99 }
]);

db.orders.deleteOne({ _id: someObjectId });

db.orders.insertOne({ user_id: 4, amount: 199.99 });
// New _id is unrelated to "previous max"

db.orders.find().sort({ created_at: -1 });  // prefer explicit time for ordering
```

---

## 🚨 When Gaps Matter (and When They Don’t)

### Gaps often **matter** when

- **Regulated or customer-visible sequences** (invoices, receipt numbers)—use a **dedicated sequence** or **ledger** with audit rules, not raw surrogate PK exposure.
- **Support workflows** (“read order number aloud”)—short, human-friendly identifiers may be a separate column.
- **Keyset pagination** on numeric id—large gaps can confuse if you assume density (`WHERE id > ?`).

### Gaps often **don’t matter** when

- Surrogate PK is **internal only** and users see slugs or business codes.
- Ordering uses **`created_at`** / **version** fields.
- System is **fully distributed** and IDs are **opaque**.

---

## 🎯 The 30-Second Summary

> *"SQL serial primary keys trade a perfectly dense sequence for **simple integers**, **index locality**, and **centralized generation**—gaps appear from rollbacks, deletes, and concurrency strategies. NoSQL commonly uses **UUIDs or ObjectIds**: **no shared counter**, so there’s no “gap”—just uniqueness. Use **timestamps** or explicit ordering fields when “latest first” matters; use **separate business sequences** when the law or finance requires gap-free human-visible numbers."*

---

## 🔗 Connecting to Previous Days

| Day | Concept | How It Connects |
|-----|---------|-----------------|
| Day 7 | Developer vs architect DB thinking | Surrogate PK vs business keys |
| Day 23 | Database selection | Where each engine fits |
| Day 28 | Consistent hashing | Sharding and key choice |
| Day 37 | Cache hit rate | Smaller int PKs vs larger UUID keys in memory |

---

## ✅ Day 38 Action Items

1. List **one** table where the PK is exposed to users—should it be?
2. For your next service, decide: **surrogate PK** + separate **business id**, or single UUID everywhere—document why.
3. If you use keyset pagination, confirm whether **gaps** affect your `WHERE id > ?` assumptions.

---

*— Sunchit Dudeja*  
*Day 38 of 50: System Design Interview Preparation Series*
