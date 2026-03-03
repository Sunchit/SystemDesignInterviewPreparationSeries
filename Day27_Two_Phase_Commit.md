# 🤝 Two-Phase Commit (2PC): The Distributed Transaction Protocol

*How Amazon Ensures Your Payment, Inventory, and Shipping Never Get Out of Sync*

---

## Introduction

You're buying an iPhone on Amazon. Three things must happen **atomically**:

1. **Payment Service**: Deduct ₹99,989 from your account
2. **Inventory Service**: Reserve 1 iPhone from stock
3. **Shipping Service**: Create shipping label and schedule pickup

**The Critical Rule:** Either ALL succeed, or NONE happen. No half-updated state!

Imagine if payment was deducted but inventory wasn't reserved. Or inventory was reserved but shipping wasn't created. Chaos.

This is the problem **Two-Phase Commit (2PC)** solves.

---

## The Problem: Distributed Transactions

### Why Can't We Just Use Regular Transactions?

```
Single Database World (Easy):
┌─────────────────────────────────────────┐
│              ONE DATABASE               │
│                                         │
│  BEGIN TRANSACTION;                     │
│    UPDATE accounts SET balance -= 100;  │
│    UPDATE inventory SET stock -= 1;     │
│    INSERT INTO orders (...);            │
│  COMMIT;  ← All or nothing!             │
│                                         │
└─────────────────────────────────────────┘

Distributed World (Hard):
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│  Payment DB  │  │ Inventory DB │  │  Orders DB   │
│              │  │              │  │              │
│  COMMIT? 🤔  │  │  COMMIT? 🤔  │  │  COMMIT? 🤔  │
│              │  │              │  │              │
└──────────────┘  └──────────────┘  └──────────────┘

How do we coordinate COMMIT across 3 separate databases?
```

**The Challenge:** Each database can commit independently, but we need them to commit **together** or **not at all**.

---

## The Distributed System Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        E-COMMERCE ORDER SYSTEM                          │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│    ┌──────────┐                                                         │
│    │ Customer │                                                         │
│    └────┬─────┘                                                         │
│         │ Place Order                                                   │
│         ▼                                                               │
│    ┌─────────────┐                                                      │
│    │   ORDER     │  ◄── COORDINATOR (Transaction Manager)               │
│    │   SERVICE   │      - Orchestrates the 2PC protocol                 │
│    │             │      - Maintains transaction log                     │
│    │             │      - Decides COMMIT or ABORT                       │
│    └──────┬──────┘                                                      │
│           │                                                             │
│     ┌─────┴─────┬─────────────┐                                        │
│     │           │             │                                        │
│     ▼           ▼             ▼                                        │
│ ┌─────────┐ ┌─────────┐ ┌─────────┐                                   │
│ │ PAYMENT │ │INVENTORY│ │SHIPPING │  ◄── PARTICIPANTS (Resource Mgrs) │
│ │ SERVICE │ │ SERVICE │ │ SERVICE │      - Execute local transactions  │
│ └────┬────┘ └────┬────┘ └────┬────┘      - Vote YES/NO                 │
│      │           │           │           - Wait for coordinator        │
│      ▼           ▼           ▼                                         │
│    [DB]        [DB]        [DB]                                        │
│   MySQL      PostgreSQL   MongoDB                                      │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Key Roles

| Role | Responsibility | Example |
|------|----------------|---------|
| **Coordinator** | Orchestrates 2PC, decides outcome | Order Service |
| **Participant** | Executes local work, votes, waits | Payment, Inventory, Shipping |
| **Resource** | The actual database/queue | MySQL, PostgreSQL, Kafka |

---

## The Transaction Context

```yaml
Customer Action:
  User: John Doe
  Order: iPhone 15 (₹79,999) + AirPods (₹19,990)
  Total: ₹99,989
  Payment: Credit Card ending 1234

Resources That Must Be Updated ATOMICALLY:
  1. Payment Service  → Deduct ₹99,989 from customer's account
  2. Inventory Service → Reduce iPhone stock by 1, AirPods stock by 1
  3. Shipping Service  → Create shipping label and schedule pickup

ACID Requirements:
  Atomicity   → All three succeed or all three fail
  Consistency → No partial order state
  Isolation   → Other transactions don't see intermediate state
  Durability  → Once committed, changes survive crashes

Critical Rule: Either ALL succeed, or NONE happen!
```

---

## The Two Phases Explained

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         TWO-PHASE COMMIT                                │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   PHASE 1: PREPARE (Voting Phase)                                       │
│   ┌─────────────────────────────────────────────────────────────────┐  │
│   │  Coordinator asks: "Can you commit?"                            │  │
│   │  Participants: Lock resources, respond YES or NO                │  │
│   │  No actual changes yet - just promises!                         │  │
│   │                                                                 │  │
│   │  Key Actions:                                                   │  │
│   │  • Write to local transaction log                               │  │
│   │  • Acquire all necessary locks                                  │  │
│   │  • Validate business rules                                      │  │
│   │  • Prepare to commit (but don't commit yet!)                   │  │
│   └─────────────────────────────────────────────────────────────────┘  │
│                                                                         │
│                              │                                          │
│                              ▼                                          │
│                                                                         │
│   PHASE 2: COMMIT or ABORT (Decision Phase)                             │
│   ┌─────────────────────────────────────────────────────────────────┐  │
│   │  If ALL said YES → COMMIT (make changes permanent)              │  │
│   │  If ANY said NO  → ABORT (release all locks, undo nothing)      │  │
│   │                                                                 │  │
│   │  Key Actions:                                                   │  │
│   │  • Coordinator logs decision before sending                     │  │
│   │  • Participants execute COMMIT/ROLLBACK                         │  │
│   │  • Release all locks                                            │  │
│   │  • Acknowledge completion                                       │  │
│   └─────────────────────────────────────────────────────────────────┘  │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Phase 1: The PREPARE Phase (Detailed)

### Sequence Diagram

```
┌──────────┐     ┌─────────────┐     ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│ Customer │     │Order Service│     │Payment Svc  │     │Inventory Svc│     │Shipping Svc │
└────┬─────┘     └──────┬──────┘     └──────┬──────┘     └──────┬──────┘     └──────┬──────┘
     │                  │                   │                   │                   │
     │  Place Order     │                   │                   │                   │
     │─────────────────►│                   │                   │                   │
     │                  │                   │                   │                   │
     │                  │ ╔═══════════════════════════════════════════════════════╗ │
     │                  │ ║            PHASE 1 - PREPARE                         ║ │
     │                  │ ╚═══════════════════════════════════════════════════════╝ │
     │                  │                   │                   │                   │
     │                  │  PREPARE:         │                   │                   │
     │                  │  Can you deduct   │                   │                   │
     │                  │  ₹99,989?         │                   │                   │
     │                  │──────────────────►│                   │                   │
     │                  │                   │                   │                   │
     │                  │                   │ ┌───────────────┐ │                   │
     │                  │                   │ │1. BEGIN TRANS │ │                   │
     │                  │                   │ │2. Check balance│ │                   │
     │                  │                   │ │3. Lock funds  │ │                   │
     │                  │                   │ │4. Log PREPARE │ │                   │
     │                  │                   │ │5. Hold (no    │ │                   │
     │                  │                   │ │   commit yet!)│ │                   │
     │                  │                   │ └───────────────┘ │                   │
     │                  │                   │                   │                   │
     │                  │  YES (Promise     │                   │                   │
     │                  │  to commit)       │                   │                   │
     │                  │◄──────────────────│                   │                   │
     │                  │                   │                   │                   │
     │                  │  PREPARE: Can you reserve             │                   │
     │                  │  1 iPhone, 1 AirPods?                 │                   │
     │                  │──────────────────────────────────────►│                   │
     │                  │                   │                   │                   │
     │                  │                   │                   │ ┌───────────────┐ │
     │                  │                   │                   │ │1. BEGIN TRANS │ │
     │                  │                   │                   │ │2. Check stock │ │
     │                  │                   │                   │ │3. Lock rows   │ │
     │                  │                   │                   │ │4. Log PREPARE │ │
     │                  │                   │                   │ │5. Hold        │ │
     │                  │                   │                   │ └───────────────┘ │
     │                  │                   │                   │                   │
     │                  │  YES (Promise to commit)              │                   │
     │                  │◄──────────────────────────────────────│                   │
     │                  │                   │                   │                   │
     │                  │  PREPARE: Can you create shipping label?                  │
     │                  │──────────────────────────────────────────────────────────►│
     │                  │                   │                   │                   │
     │                  │                   │                   │ ┌───────────────┐ │
     │                  │                   │                   │ │1. Reserve slot│ │
     │                  │                   │                   │ │2. Generate ID │ │
     │                  │                   │                   │ │3. Log PREPARE │ │
     │                  │                   │                   │ │4. Hold        │ │
     │                  │                   │                   │ └───────────────┘ │
     │                  │                   │                   │                   │
     │                  │  YES (Promise to commit)                                  │
     │                  │◄──────────────────────────────────────────────────────────│
     │                  │                   │                   │                   │
     │  ┌───────────────────────────────┐   │                   │                   │
     │  │ All participants said YES!    │   │                   │                   │
     │  │ Ready for Phase 2 - COMMIT    │   │                   │                   │
     │  └───────────────────────────────┘   │                   │                   │
```

### What Happens Inside Each Service

#### Payment Service - PREPARE (Detailed)

```sql
-- Payment Service Database (MySQL)
-- Step 1: Start transaction
BEGIN TRANSACTION;

-- Step 2: Log the prepare request (for recovery)
INSERT INTO transaction_log 
    (tx_id, order_id, operation, status, created_at) 
VALUES 
    ('TX-98765', 'ORD-12345', 'DEBIT', 'PREPARING', NOW());

-- Step 3: Validate business rules
SELECT balance, credit_limit, account_status 
FROM accounts 
WHERE user_id = 123;
-- Returns: balance=150000, credit_limit=200000, status='ACTIVE'

-- Step 4: Check if enough balance
-- ₹150,000 >= ₹99,989? YES!

-- Step 5: Lock the funds (SELECT FOR UPDATE prevents other transactions)
SELECT * FROM accounts WHERE user_id = 123 FOR UPDATE;

-- Step 6: Create pending hold (not actual deduction!)
INSERT INTO pending_holds 
    (order_id, user_id, amount, hold_type, expires_at, status) 
VALUES 
    ('ORD-12345', 123, 99989, 'PAYMENT_HOLD', NOW() + INTERVAL '5 minutes', 'ACTIVE');

-- Step 7: Update transaction log to PREPARED
UPDATE transaction_log 
SET status = 'PREPARED', prepared_at = NOW()
WHERE tx_id = 'TX-98765';

-- IMPORTANT: DO NOT COMMIT YET!
-- Transaction stays open, locks are held
-- Response to coordinator: VOTE YES (funds locked, ready to commit)
```

**Key Points:**
- Transaction is **NOT committed** yet
- Row locks are held (`SELECT FOR UPDATE`)
- Other transactions trying to access this account will **block**
- If coordinator dies, we have the log to recover

#### Inventory Service - PREPARE (Detailed)

```sql
-- Inventory Service Database (PostgreSQL)
BEGIN TRANSACTION;

-- Log the prepare
INSERT INTO transaction_log 
    (tx_id, order_id, products, status) 
VALUES 
    ('TX-INV-456', 'ORD-12345', '["IPHONE-15","AIRPODS"]', 'PREPARING');

-- Check stock with lock
SELECT product_id, stock_quantity, reserved_quantity
FROM products 
WHERE product_id IN ('IPHONE-15', 'AIRPODS')
FOR UPDATE;  -- Lock these rows!

-- Validate stock
-- iPhone: stock=50, reserved=5 → available=45 (need 1) ✓
-- AirPods: stock=100, reserved=20 → available=80 (need 1) ✓

-- Create reservation (soft lock, not actual deduction)
INSERT INTO inventory_reservations 
    (order_id, product_id, quantity, reserved_at, expires_at, status) 
VALUES 
    ('ORD-12345', 'IPHONE-15', 1, NOW(), NOW() + INTERVAL '5 minutes', 'PENDING'),
    ('ORD-12345', 'AIRPODS', 1, NOW(), NOW() + INTERVAL '5 minutes', 'PENDING');

-- Update reserved count (for visibility to other transactions)
UPDATE products SET reserved_quantity = reserved_quantity + 1 
WHERE product_id = 'IPHONE-15';

UPDATE products SET reserved_quantity = reserved_quantity + 1 
WHERE product_id = 'AIRPODS';

-- Mark as prepared
UPDATE transaction_log SET status = 'PREPARED' WHERE tx_id = 'TX-INV-456';

-- DO NOT COMMIT! Hold the locks.
-- Response: VOTE YES
```

#### Shipping Service - PREPARE (Detailed)

```javascript
// Shipping Service (Node.js + MongoDB)
async function prepareShipping(orderId, items, address) {
  const session = await mongoose.startSession();
  session.startTransaction();
  
  try {
    // 1. Log the prepare
    await TransactionLog.create([{
      txId: 'TX-SHIP-789',
      orderId: orderId,
      status: 'PREPARING',
      createdAt: new Date()
    }], { session });
    
    // 2. Find available shipping slot
    const slot = await ShippingSlots.findOneAndUpdate(
      { 
        date: '2026-03-04',
        available: true,
        region: address.region 
      },
      { 
        $set: { 
          available: false,
          reservedFor: orderId,
          reservedUntil: new Date(Date.now() + 5 * 60 * 1000)
        }
      },
      { session, new: true }
    );
    
    if (!slot) {
      throw new Error('No shipping slots available');
    }
    
    // 3. Generate tracking ID (but don't finalize)
    const shipment = await PendingShipments.create([{
      orderId: orderId,
      trackingId: generateTrackingId(),
      status: 'PENDING_CONFIRMATION',
      slot: slot._id,
      expiresAt: new Date(Date.now() + 5 * 60 * 1000)
    }], { session });
    
    // 4. Mark as prepared
    await TransactionLog.updateOne(
      { txId: 'TX-SHIP-789' },
      { status: 'PREPARED', preparedAt: new Date() },
      { session }
    );
    
    // DO NOT COMMIT! Keep session open
    return { 
      vote: 'YES', 
      trackingId: shipment[0].trackingId,
      session: session  // Keep reference to abort/commit later
    };
    
  } catch (error) {
    await session.abortTransaction();
    return { vote: 'NO', reason: error.message };
  }
}
```

---

## Phase 2A: The COMMIT Phase (All Votes YES)

When all participants vote YES, the coordinator sends COMMIT to everyone.

### The Commit Flow

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐     ┌─────────────┐     ┌────────┐
│Order Service│     │Payment Svc  │     │Inventory Svc│     │Shipping Svc │     │ Client │
└──────┬──────┘     └──────┬──────┘     └──────┬──────┘     └──────┬──────┘     └───┬────┘
       │                   │                   │                   │                │
       │ ╔════════════════════════════════════════════════════════════════════════╗ │
       │ ║                     PHASE 2 - COMMIT                                   ║ │
       │ ╚════════════════════════════════════════════════════════════════════════╝ │
       │                   │                   │                   │                │
       │                   │                   │                   │                │
       │  ┌────────────────────────────────────────────────────────────────────┐   │
       │  │ COORDINATOR: Log decision as "COMMITTING" BEFORE sending commits  │   │
       │  │ (This is crucial for crash recovery!)                              │   │
       │  └────────────────────────────────────────────────────────────────────┘   │
       │                   │                   │                   │                │
       │  COMMIT           │                   │                   │                │
       │  transaction      │                   │                   │                │
       │──────────────────►│                   │                   │                │
       │                   │                   │                   │                │
       │                   │ ┌───────────────┐ │                   │                │
       │                   │ │1. Deduct funds│ │                   │                │
       │                   │ │2. Remove hold │ │                   │                │
       │                   │ │3. Log COMMIT  │ │                   │                │
       │                   │ │4. COMMIT!     │ │                   │                │
       │                   │ │5. Release lock│ │                   │                │
       │                   │ └───────────────┘ │                   │                │
       │                   │                   │                   │                │
       │  COMMITTED (ACK)  │                   │                   │                │
       │◄──────────────────│                   │                   │                │
       │                   │                   │                   │                │
       │  COMMIT transaction                   │                   │                │
       │──────────────────────────────────────►│                   │                │
       │                   │                   │                   │                │
       │                   │                   │ ┌───────────────┐ │                │
       │                   │                   │ │1. Reduce stock│ │                │
       │                   │                   │ │2. Clear reserv│ │                │
       │                   │                   │ │3. Log COMMIT  │ │                │
       │                   │                   │ │4. COMMIT!     │ │                │
       │                   │                   │ │5. Release lock│ │                │
       │                   │                   │ └───────────────┘ │                │
       │                   │                   │                   │                │
       │  COMMITTED (ACK)                      │                   │                │
       │◄──────────────────────────────────────│                   │                │
       │                   │                   │                   │                │
       │  COMMIT transaction                                       │                │
       │──────────────────────────────────────────────────────────►│                │
       │                   │                   │                   │                │
       │                   │                   │ ┌───────────────┐ │                │
       │                   │                   │ │1. Finalize    │ │                │
       │                   │                   │ │2. Create label│ │                │
       │                   │                   │ │3. Log COMMIT  │ │                │
       │                   │                   │ │4. COMMIT!     │ │                │
       │                   │                   │ └───────────────┘ │                │
       │                   │                   │                   │                │
       │  COMMITTED (ACK)                                          │                │
       │◄──────────────────────────────────────────────────────────│                │
       │                   │                   │                   │                │
       │  ┌────────────────────────────────────────────────────────────────────┐   │
       │  │ COORDINATOR: All ACKs received → Log as "COMMITTED"               │   │
       │  └────────────────────────────────────────────────────────────────────┘   │
       │                   │                   │                   │                │
       │  Order Confirmed! 🎉                                                      │
       │───────────────────────────────────────────────────────────────────────────►│
```

### What Happens in COMMIT (Detailed)

#### Payment Service - COMMIT

```sql
-- Continue the transaction from PREPARE phase
-- (Transaction was held open, not committed)

-- Step 1: Actually deduct the funds now
UPDATE accounts 
SET balance = balance - 99989,
    last_transaction_at = NOW()
WHERE user_id = 123;

-- Step 2: Convert hold to actual transaction
UPDATE pending_holds 
SET status = 'CONVERTED_TO_DEBIT'
WHERE order_id = 'ORD-12345';

-- Step 3: Create permanent transaction record
INSERT INTO transactions 
    (order_id, user_id, amount, type, status, created_at)
VALUES 
    ('ORD-12345', 123, 99989, 'DEBIT', 'COMPLETED', NOW());

-- Step 4: Update transaction log
UPDATE transaction_log 
SET status = 'COMMITTED', committed_at = NOW()
WHERE tx_id = 'TX-98765';

-- Step 5: FINALLY COMMIT! (releases all locks)
COMMIT;

-- Now the transaction is permanent and other queries can see it
```

#### Inventory Service - COMMIT

```sql
-- Step 1: Reduce actual stock
UPDATE products 
SET stock_quantity = stock_quantity - 1,
    reserved_quantity = reserved_quantity - 1,
    last_sold_at = NOW()
WHERE product_id = 'IPHONE-15';

UPDATE products 
SET stock_quantity = stock_quantity - 1,
    reserved_quantity = reserved_quantity - 1,
    last_sold_at = NOW()
WHERE product_id = 'AIRPODS';

-- Step 2: Convert reservation to sale
UPDATE inventory_reservations 
SET status = 'FULFILLED', fulfilled_at = NOW()
WHERE order_id = 'ORD-12345';

-- Step 3: Create inventory movement record
INSERT INTO inventory_movements 
    (order_id, product_id, quantity, movement_type, created_at)
VALUES 
    ('ORD-12345', 'IPHONE-15', -1, 'SALE', NOW()),
    ('ORD-12345', 'AIRPODS', -1, 'SALE', NOW());

-- Step 4: Log commit
UPDATE transaction_log SET status = 'COMMITTED' WHERE tx_id = 'TX-INV-456';

-- Step 5: COMMIT!
COMMIT;
```

#### Shipping Service - COMMIT

```javascript
async function commitShipping(session, orderId) {
  try {
    // 1. Finalize the shipment
    await PendingShipments.updateOne(
      { orderId: orderId },
      { 
        status: 'CONFIRMED',
        confirmedAt: new Date()
      },
      { session }
    );
    
    // 2. Create actual shipping label
    const shipment = await PendingShipments.findOne({ orderId });
    const labelUrl = await generateShippingLabel(shipment);
    
    await Shipments.create([{
      orderId: orderId,
      trackingId: shipment.trackingId,
      labelUrl: labelUrl,
      status: 'READY_FOR_PICKUP',
      createdAt: new Date()
    }], { session });
    
    // 3. Update transaction log
    await TransactionLog.updateOne(
      { txId: 'TX-SHIP-789' },
      { status: 'COMMITTED', committedAt: new Date() },
      { session }
    );
    
    // 4. COMMIT the transaction!
    await session.commitTransaction();
    session.endSession();
    
    return { success: true, trackingId: shipment.trackingId };
    
  } catch (error) {
    // Even if commit fails, we MUST retry
    // Participant has voted YES, it MUST eventually commit
    await retryCommit(session, orderId);
  }
}
```

---

## Phase 2B: The ABORT Phase (Any Vote NO)

What if Inventory Service can't reserve items (out of stock)?

### The Abort Flow

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐     ┌─────────────┐     ┌────────┐
│Order Service│     │Payment Svc  │     │Inventory Svc│     │Shipping Svc │     │ Client │
└──────┬──────┘     └──────┬──────┘     └──────┬──────┘     └──────┬──────┘     └───┬────┘
       │                   │                   │                   │                │
       │ ╔════════════════════════════════════════════════════════════════════════╗ │
       │ ║                     PHASE 1 - PREPARE                                  ║ │
       │ ╚════════════════════════════════════════════════════════════════════════╝ │
       │                   │                   │                   │                │
       │  PREPARE          │                   │                   │                │
       │──────────────────►│                   │                   │                │
       │                   │                   │                   │                │
       │                   │ (Locks ₹99,989)   │                   │                │
       │                   │                   │                   │                │
       │  YES              │                   │                   │                │
       │◄──────────────────│                   │                   │                │
       │                   │                   │                   │                │
       │  PREPARE                              │                   │                │
       │──────────────────────────────────────►│                   │                │
       │                   │                   │                   │                │
       │                   │                   │ ┌───────────────┐ │                │
       │                   │                   │ │ iPhone stock  │ │                │
       │                   │                   │ │ = 0           │ │                │
       │                   │                   │ │ ❌ OUT OF     │ │                │
       │                   │                   │ │    STOCK!     │ │                │
       │                   │                   │ └───────────────┘ │                │
       │                   │                   │                   │                │
       │  NO (Abort!)                          │                   │                │
       │◄──────────────────────────────────────│                   │                │
       │                   │                   │                   │                │
       │ ┌──────────────────────────────────┐  │                   │                │
       │ │  One participant said NO!        │  │                   │                │
       │ │  COORDINATOR logs "ABORTING"     │  │                   │                │
       │ │  Must ABORT entire transaction   │  │                   │                │
       │ └──────────────────────────────────┘  │                   │                │
       │                   │                   │                   │                │
       │ ╔════════════════════════════════════════════════════════════════════════╗ │
       │ ║                     PHASE 2 - ABORT                                    ║ │
       │ ╚════════════════════════════════════════════════════════════════════════╝ │
       │                   │                   │                   │                │
       │  ABORT            │                   │                   │                │
       │  (release locks)  │                   │                   │                │
       │──────────────────►│                   │                   │                │
       │                   │                   │                   │                │
       │                   │ ┌───────────────┐ │                   │                │
       │                   │ │1. ROLLBACK    │ │                   │                │
       │                   │ │2. Remove hold │ │                   │                │
       │                   │ │3. Log ABORTED │ │                   │                │
       │                   │ │4. Release lock│ │                   │                │
       │                   │ │               │ │                   │                │
       │                   │ │💰 Money safe! │ │                   │                │
       │                   │ └───────────────┘ │                   │                │
       │                   │                   │                   │                │
       │  ABORTED (ACK)    │                   │                   │                │
       │◄──────────────────│                   │                   │                │
       │                   │                   │                   │                │
       │  (Inventory already failed - no abort needed)             │                │
       │                   │                   │                   │                │
       │  ABORT (not needed - we never sent PREPARE)               │                │
       │──────────────────────────────────────────────────────────►│                │
       │                   │                   │                   │                │
       │  Order Failed - Out of Stock ❌                                            │
       │───────────────────────────────────────────────────────────────────────────►│
```

### What Happens in ABORT

#### Payment Service - ABORT

```sql
-- We had locked funds but never deducted
-- Just rollback the transaction

-- Step 1: Remove the pending hold
DELETE FROM pending_holds 
WHERE order_id = 'ORD-12345' AND status = 'ACTIVE';

-- Step 2: Log the abort
UPDATE transaction_log 
SET status = 'ABORTED', aborted_at = NOW(), reason = 'Inventory unavailable'
WHERE tx_id = 'TX-98765';

-- Step 3: ROLLBACK (releases all locks, discards changes)
ROLLBACK;

-- Customer's money is SAFE!
-- Balance remains ₹150,000
-- No deduction ever happened
```

#### Customer Experience

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│   ❌ Order Failed                                               │
│                                                                 │
│   Sorry, iPhone 15 is currently out of stock.                  │
│   Your payment was NOT processed.                              │
│                                                                 │
│   Your card ending in 1234 was not charged.                    │
│                                                                 │
│   [Try Again] [Notify When Available]                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Key Point:** No money was ever actually deducted. The lock was released cleanly.

---

## State Transition Diagram

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    2PC STATE TRANSITIONS                                │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  COORDINATOR STATES              PARTICIPANT STATES                     │
│  ──────────────────              ──────────────────                     │
│                                                                         │
│     ┌─────────┐                      ┌─────────┐                       │
│     │  INIT   │                      │  INIT   │                       │
│     └────┬────┘                      └────┬────┘                       │
│          │                                │                            │
│          │ Start 2PC                      │ Receive PREPARE            │
│          ▼                                ▼                            │
│   ┌──────────────┐                 ┌──────────────┐                    │
│   │   WAITING    │                 │   PREPARED   │                    │
│   │  (for votes) │                 │  (voted YES) │                    │
│   └───────┬──────┘                 └───────┬──────┘                    │
│           │                                │                            │
│   ┌───────┴───────┐                ┌───────┴───────┐                   │
│   │               │                │               │                   │
│ All YES        Any NO        Recv COMMIT     Recv ABORT                │
│   │               │                │               │                   │
│   ▼               ▼                ▼               ▼                   │
│ ┌──────────┐  ┌──────────┐   ┌──────────┐   ┌──────────┐              │
│ │COMMITTING│  │ ABORTING │   │COMMITTED │   │ ABORTED  │              │
│ └────┬─────┘  └────┬─────┘   │    ✅    │   │    ❌    │              │
│      │             │         └──────────┘   └──────────┘              │
│      │ All ACKs    │ All ACKs                                          │
│      ▼             ▼                                                   │
│ ┌──────────┐  ┌──────────┐                                            │
│ │COMMITTED │  │ ABORTED  │                                            │
│ │    ✅    │  │    ❌    │                                            │
│ └──────────┘  └──────────┘                                            │
│                                                                         │
│ IMPORTANT: Once coordinator decides COMMIT, it MUST eventually         │
│ succeed. Participants who voted YES MUST commit when told.             │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## The Big Problem: Failure Scenarios

### Scenario 1: Coordinator Crashes After Some Commits

```
Timeline:
  1. Coordinator receives all YES votes
  2. Coordinator logs "COMMITTING" to disk
  3. Coordinator tells Payment: COMMIT ✓
  4. Coordinator tells Inventory: COMMIT ✓
  5. Coordinator CRASHES before telling Shipping: COMMIT ❌

Current State:
  ✅ Payment: COMMITTED (₹99,989 deducted)
  ✅ Inventory: COMMITTED (iPhone reserved)
  ❓ Shipping: PREPARED (waiting for decision)

Problem:
  Shipping is stuck in PREPARED state, holding locks!
  Customer paid, but no delivery scheduled!
```

### Solution: Write-Ahead Log (WAL) + Recovery

```sql
-- Coordinator's Write-Ahead Log (WAL)
CREATE TABLE coordinator_log (
    transaction_id VARCHAR(100) PRIMARY KEY,
    state ENUM('PREPARING', 'COMMITTING', 'COMMITTED', 'ABORTING', 'ABORTED'),
    participants JSON,
    votes JSON,
    created_at TIMESTAMP,
    decision_at TIMESTAMP,
    completed_at TIMESTAMP
);

-- BEFORE sending any PREPARE:
INSERT INTO coordinator_log VALUES 
    ('ORD-12345', 'PREPARING', 
     '["payment","inventory","shipping"]', 
     '{}', NOW(), NULL, NULL);

-- After collecting all votes, BEFORE sending COMMIT:
UPDATE coordinator_log 
SET state = 'COMMITTING', 
    votes = '{"payment":"YES","inventory":"YES","shipping":"YES"}',
    decision_at = NOW()
WHERE transaction_id = 'ORD-12345';

-- CRITICAL: This log entry is DURABLE (fsync'd to disk)
-- Even if coordinator crashes, we know the decision was COMMIT
```

### Recovery Process

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    COORDINATOR RECOVERY ALGORITHM                       │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ON STARTUP:                                                            │
│  1. Read coordinator_log from disk                                      │
│  2. Find all incomplete transactions                                    │
│                                                                         │
│  FOR EACH incomplete transaction:                                       │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ IF state = 'COMMITTING':                                        │   │
│  │    // Decision was COMMIT, must complete it                     │   │
│  │    FOR EACH participant:                                        │   │
│  │       IF not already committed:                                 │   │
│  │          SEND COMMIT (retry until ACK)                          │   │
│  │    MARK transaction as COMMITTED                                │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ IF state = 'ABORTING':                                          │   │
│  │    // Decision was ABORT, must complete it                      │   │
│  │    FOR EACH participant:                                        │   │
│  │       IF voted YES and not already aborted:                     │   │
│  │          SEND ABORT (retry until ACK)                           │   │
│  │    MARK transaction as ABORTED                                  │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ IF state = 'PREPARING' AND age > timeout:                       │   │
│  │    // Never got all votes, safe to abort                        │   │
│  │    SEND ABORT to all participants who voted                     │   │
│  │    MARK transaction as ABORTED                                  │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Scenario 2: Participant Crashes After Voting YES

```
Timeline:
  1. Payment votes YES (locks funds)
  2. Inventory votes YES (locks items)
  3. Shipping votes YES (reserves slot)
  4. Inventory Service CRASHES
  5. Coordinator decides COMMIT
  6. Coordinator sends COMMIT to all

Problem:
  Inventory service is down, can't receive COMMIT!
  But it PROMISED to commit (voted YES)!

Solution:
  1. Coordinator retries COMMIT to Inventory
  2. When Inventory recovers, it checks its transaction_log
  3. Sees state = 'PREPARED' for ORD-12345
  4. Asks coordinator: "What's the decision?"
  5. Coordinator says: "COMMIT"
  6. Inventory commits
```

### Participant Recovery

```sql
-- Participant's recovery on startup
SELECT * FROM transaction_log WHERE status = 'PREPARED';

-- For each PREPARED transaction, ask coordinator:
-- "What's the decision for transaction X?"

-- If coordinator says COMMIT → execute COMMIT
-- If coordinator says ABORT → execute ROLLBACK
-- If coordinator is also down → WAIT (this is the blocking problem!)
```

---

## Timeout Handling

```yaml
Timeouts in 2PC:

Coordinator Timeouts:
  prepare_timeout: 30 seconds
    # If participant doesn't vote within 30s, assume NO
    
  commit_ack_timeout: 60 seconds
    # If participant doesn't ACK commit, keep retrying

Participant Timeouts:
  decision_timeout: 120 seconds
    # If coordinator doesn't send decision after voting YES
    # Participant is BLOCKED (can't decide alone!)

Lock Timeouts:
  lock_hold_timeout: 300 seconds
    # Maximum time to hold locks in PREPARED state
    # After this, might need manual intervention

Recovery Timeouts:
  coordinator_recovery_check: 10 seconds
    # How often to check if coordinator is back
```

---

## Java/Spring Implementation

### Coordinator Implementation

```java
@Service
@Transactional
public class TwoPhaseCommitCoordinator {
    
    private final TransactionLogRepository txLog;
    private final PaymentClient paymentClient;
    private final InventoryClient inventoryClient;
    private final ShippingClient shippingClient;
    
    public OrderResult placeOrder(Order order) {
        String txId = UUID.randomUUID().toString();
        
        // Step 1: Log transaction start
        txLog.save(new TransactionLog(txId, "PREPARING", order.getId()));
        
        try {
            // Step 2: PREPARE phase - collect votes
            List<Vote> votes = new ArrayList<>();
            
            CompletableFuture<Vote> paymentVote = 
                paymentClient.prepare(txId, order.getAmount());
            CompletableFuture<Vote> inventoryVote = 
                inventoryClient.prepare(txId, order.getItems());
            CompletableFuture<Vote> shippingVote = 
                shippingClient.prepare(txId, order.getAddress());
            
            // Wait for all votes with timeout
            votes.add(paymentVote.get(30, TimeUnit.SECONDS));
            votes.add(inventoryVote.get(30, TimeUnit.SECONDS));
            votes.add(shippingVote.get(30, TimeUnit.SECONDS));
            
            // Step 3: Check all votes
            boolean allYes = votes.stream().allMatch(v -> v.isYes());
            
            if (allYes) {
                return commitAll(txId, order);
            } else {
                return abortAll(txId, order, votes);
            }
            
        } catch (TimeoutException e) {
            // Participant didn't respond in time
            return abortAll(txId, order, "Timeout waiting for participant");
        }
    }
    
    private OrderResult commitAll(String txId, Order order) {
        // CRITICAL: Log decision BEFORE sending commits
        txLog.updateState(txId, "COMMITTING");
        
        // Send COMMIT to all participants (with retry)
        retryUntilSuccess(() -> paymentClient.commit(txId));
        retryUntilSuccess(() -> inventoryClient.commit(txId));
        retryUntilSuccess(() -> shippingClient.commit(txId));
        
        txLog.updateState(txId, "COMMITTED");
        
        return new OrderResult(true, "Order confirmed!");
    }
    
    private OrderResult abortAll(String txId, Order order, String reason) {
        txLog.updateState(txId, "ABORTING");
        
        // Send ABORT to all participants who voted
        paymentClient.abort(txId);
        inventoryClient.abort(txId);
        shippingClient.abort(txId);
        
        txLog.updateState(txId, "ABORTED");
        
        return new OrderResult(false, reason);
    }
    
    private void retryUntilSuccess(Runnable action) {
        int maxRetries = 100;
        int retryCount = 0;
        
        while (retryCount < maxRetries) {
            try {
                action.run();
                return;
            } catch (Exception e) {
                retryCount++;
                Thread.sleep(1000 * retryCount); // Exponential backoff
            }
        }
        
        throw new CommitFailedException("Failed after " + maxRetries + " retries");
    }
}
```

### Participant Implementation

```java
@Service
public class PaymentParticipant {
    
    private final AccountRepository accounts;
    private final PendingHoldRepository holds;
    private final ParticipantLogRepository log;
    
    // Map of txId -> active database session
    private final Map<String, TransactionSession> activeSessions = 
        new ConcurrentHashMap<>();
    
    public Vote prepare(String txId, BigDecimal amount, Long userId) {
        TransactionSession session = beginTransaction();
        
        try {
            // Log prepare
            log.save(new ParticipantLog(txId, "PREPARING"));
            
            // Check balance
            Account account = accounts.findByIdForUpdate(userId, session);
            
            if (account.getBalance().compareTo(amount) < 0) {
                session.rollback();
                return Vote.NO("Insufficient balance");
            }
            
            // Create hold (lock funds)
            holds.save(new PendingHold(txId, userId, amount), session);
            
            // Log prepared
            log.updateState(txId, "PREPARED");
            
            // Keep session open!
            activeSessions.put(txId, session);
            
            return Vote.YES();
            
        } catch (Exception e) {
            session.rollback();
            return Vote.NO(e.getMessage());
        }
    }
    
    public void commit(String txId) {
        TransactionSession session = activeSessions.get(txId);
        
        if (session == null) {
            // Recovery scenario - recreate from log
            session = recoverSession(txId);
        }
        
        try {
            PendingHold hold = holds.findByTxId(txId, session);
            
            // Actually deduct the money
            accounts.deduct(hold.getUserId(), hold.getAmount(), session);
            
            // Mark hold as converted
            holds.markConverted(txId, session);
            
            // Log commit
            log.updateState(txId, "COMMITTED");
            
            // COMMIT!
            session.commit();
            
            activeSessions.remove(txId);
            
        } catch (Exception e) {
            // Must retry - we voted YES, we MUST commit
            throw new RetryableException(e);
        }
    }
    
    public void abort(String txId) {
        TransactionSession session = activeSessions.get(txId);
        
        if (session != null) {
            // Remove hold
            holds.deleteByTxId(txId, session);
            
            // Log abort
            log.updateState(txId, "ABORTED");
            
            // ROLLBACK
            session.rollback();
            
            activeSessions.remove(txId);
        }
    }
}
```

---

## Real-World Examples

### 1. Airline Booking System (MakeMyTrip)

```yaml
Scenario: Booking a Round-Trip Flight

User Wants:
  - Flight DEL→NYC on Mar 10 (IndiGo)
  - Flight NYC→DEL on Mar 20 (United Airlines)
  - Pay with Credit Card
  - Accrue loyalty miles
  
Services Involved:
  1. IndiGo Booking API (external)
  2. United Booking API (external)
  3. Payment Gateway (Razorpay)
  4. MakeMyTrip Loyalty Service

The 2PC Flow:
  PREPARE:
    - IndiGo: Lock seat 12A → YES
    - United: Lock seat 7C → YES
    - Razorpay: Authorize ₹85,000 → YES
    - Loyalty: Reserve 5,000 miles → YES

  COMMIT:
    - IndiGo: Confirm booking → PNR: 6X7Y8Z
    - United: Confirm booking → PNR: AB12CD
    - Razorpay: Capture payment
    - Loyalty: Add miles to account

If United Says NO (flight full):
    - IndiGo: Release seat 12A
    - Razorpay: Void authorization (no charge)
    - Loyalty: Nothing to undo
    - User sees: "Return flight unavailable"
```

### 2. Banking Fund Transfer (ICICI to HDFC)

```yaml
Scenario: NEFT Transfer ₹1,00,000

Participants:
  1. ICICI Core Banking (debit account)
  2. HDFC Core Banking (credit account)
  3. RBI Settlement System

The 2PC Flow:
  PREPARE:
    - ICICI: Lock ₹1,00,000 in sender account → YES
    - HDFC: Verify recipient account exists → YES
    - RBI: Verify settlement capacity → YES

  COMMIT:
    - ICICI: Deduct ₹1,00,000
    - HDFC: Credit ₹1,00,000
    - RBI: Record settlement

Why 2PC is CRITICAL here:
    - Can't deduct from ICICI without crediting HDFC
    - Can't credit HDFC without deducting from ICICI
    - Money can't disappear or be created!
```

### 3. E-Commerce Checkout (Flipkart)

```yaml
Scenario: Big Billion Day Sale

User buying:
  - iPhone 15: ₹79,999 (limited stock: 100 units)
  - Credit Card Payment
  - Same-day delivery (limited slots)

Participants:
  1. Payment Service (Paytm/Razorpay)
  2. Inventory Service (limited stock)
  3. Warehouse Service (pick & pack slot)
  4. Delivery Service (same-day slot)

Challenge:
  - 50,000 users trying to buy iPhone 15 simultaneously
  - Only 100 units available
  - Only 500 same-day delivery slots

2PC ensures:
  - No overselling (inventory lock)
  - No double-charging (payment lock)
  - Guaranteed delivery slot if order succeeds
```

---

## Performance Impact

```yaml
Timing Breakdown for a 2PC Transaction:

  PREPARE Phase:
    - Payment authorization: 150ms
    - Inventory check & lock: 100ms
    - Shipping slot reserve: 200ms
    - Network round-trips: 100ms
    ─────────────────────────────
    Total PREPARE: ~550ms
  
  COMMIT Phase:
    - Payment capture: 50ms
    - Inventory update: 50ms
    - Shipping finalize: 100ms
    - Network round-trips: 100ms
    ─────────────────────────────
    Total COMMIT: ~300ms

  Overall Transaction: ~850ms
  (Resources locked for ENTIRE duration!)
  
vs Single Database Transaction:
  - One database commit: 50-100ms
  
Why 2PC is "Slow":
  ├── Locks held across entire 850ms
  ├── Network round-trips multiply latency
  ├── Each participant must write to durable log
  ├── Coordinator must fsync decision before commits
  └── Blocking during participant/coordinator failures

Throughput Impact:
  ├── Locks prevent concurrent access
  ├── Hot rows (popular products) become bottlenecks
  └── Database connection pools exhausted during failures
```

---

## When to Use 2PC

```yaml
✅ USE 2PC When:
  - Banking/financial transactions (can't have half a transfer)
  - Flight/hotel bookings (atomic package deals)
  - Inventory management with strict consistency
  - Cross-database transactions within same datacenter
  - Compliance requirements (audit trail, exactly-once)
  - Low-volume, high-value transactions

❌ AVOID 2PC When:
  - High throughput needed (>10,000 TPS)
  - Services are geographically distributed (latency kills)
  - One participant is unreliable (cascading failures)
  - Long-running transactions (holding locks too long)
  - Social media, analytics, logging (eventual consistency OK)
  - External APIs that don't support 2PC

🔄 CONSIDER ALTERNATIVES:
  - Saga Pattern (compensating transactions)
  - Eventual consistency with reconciliation jobs
  - Idempotent APIs + retry logic
  - Event sourcing + CQRS
  - Outbox pattern with message queues
```

---

## 2PC vs Saga Pattern

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    2PC vs SAGA COMPARISON                               │
├───────────────────┬─────────────────────┬───────────────────────────────┤
│ Aspect            │ 2PC                 │ Saga                          │
├───────────────────┼─────────────────────┼───────────────────────────────┤
│ Consistency       │ Strong (ACID)       │ Eventual                      │
│ Lock Duration     │ Entire transaction  │ Per-step only                 │
│ Throughput        │ Lower               │ Higher                        │
│ Latency           │ Higher (2 rounds)   │ Lower (async possible)        │
│ Complexity        │ Coordinator needed  │ Choreography or orchestration │
│ Failure Handling  │ Automatic rollback  │ Compensating transactions     │
│ Blocking          │ Yes (on failures)   │ No                            │
│ Use Case          │ Banking, bookings   │ E-commerce, workflows         │
│ External APIs     │ Must support 2PC    │ Any API (compensate later)    │
└───────────────────┴─────────────────────┴───────────────────────────────┘
```

### When to Choose Which?

```yaml
Choose 2PC:
  - "I need guaranteed atomicity"
  - "All my services are in one datacenter"
  - "I can't tolerate temporary inconsistency"
  - "Transaction duration is short (<1 second)"
  
Choose Saga:
  - "I need high throughput"
  - "My services are geographically distributed"
  - "Temporary inconsistency is acceptable"
  - "I'm integrating with external APIs"
  - "Transaction might take minutes/hours"
```

---

## Common Interview Questions

### Q1: What happens if coordinator crashes after deciding COMMIT but before sending to all participants?

```
Answer:
1. Coordinator has logged "COMMITTING" to disk before sending commits
2. On recovery, coordinator reads log, sees "COMMITTING"
3. Coordinator resends COMMIT to all participants
4. Participants who already committed → ACK again (idempotent)
5. Participants still in PREPARED → now COMMIT
6. Transaction completes successfully

Key insight: The decision is DURABLE in the log. 
The log IS the decision, not the message sending.
```

### Q2: What happens if participant crashes after voting YES?

```
Answer:
1. Participant had voted YES and logged "PREPARED"
2. Coordinator decides COMMIT, sends COMMIT to participant
3. Participant is down, doesn't receive COMMIT
4. Coordinator retries COMMIT periodically
5. Participant recovers, checks its log → sees "PREPARED"
6. Participant asks coordinator: "What's the decision?"
7. Coordinator says COMMIT
8. Participant commits

Key insight: Participant MUST eventually commit if it voted YES.
It has made a PROMISE that must be kept.
```

### Q3: Why is 2PC called "blocking"?

```
Answer:
If coordinator crashes AFTER collecting votes but BEFORE logging decision:

1. All participants are in PREPARED state
2. They've voted YES and are holding locks
3. Coordinator is down
4. Participants don't know the decision
5. They CANNOT decide alone (another might have voted NO!)
6. They MUST wait for coordinator to recover
7. All resources remain LOCKED during this time

This is the "blocking" problem - participants are stuck waiting.

Solution: 3PC (Three-Phase Commit) adds a PRE-COMMIT phase
to reduce blocking, but adds more latency.
```

---

## Key Takeaway

Two-Phase Commit is like a **wedding ceremony**:

```
PHASE 1 - "Do you take this commit?"
┌─────────────────────────────────────────────────────────────────────────┐
│  Coordinator (Priest): "Do you, Payment Service, promise to commit?"   │
│  Payment Service: "I do" (locks funds)                                  │
│                                                                         │
│  Coordinator: "Do you, Inventory Service, promise to commit?"          │
│  Inventory Service: "I do" (locks items)                               │
│                                                                         │
│  No actual changes yet - just promises!                                │
│  Everyone is waiting, holding resources, ready to act.                 │
└─────────────────────────────────────────────────────────────────────────┘

PHASE 2 - "I now pronounce you... COMMITTED"
┌─────────────────────────────────────────────────────────────────────────┐
│  Only after ALL say "I do" do we make it real.                         │
│  If anyone says "I object", we cancel everything.                      │
│                                                                         │
│  Once the priest says "COMMIT", everyone MUST follow through.          │
│  No backing out after the decision!                                    │
│                                                                         │
│  No half-marriages allowed!                                            │
└─────────────────────────────────────────────────────────────────────────┘
```

**The Price:** 
- Everyone must wait while promises are made
- Resources stay locked
- Latency increases
- Blocking on coordinator failure

**The Guarantee:** 
- Perfect atomicity across completely separate systems
- All-or-nothing semantics
- Strong consistency

**The Reality:**
- Banks use 2PC → Can't have half a money transfer
- Social media doesn't → Eventual consistency is fine for likes
- Airlines use 2PC → Can't book half a round-trip
- Streaming services don't → Buffering stats can be eventually consistent

**Choose wisely based on your consistency requirements.**

---

## Summary

| Aspect | Details |
|--------|---------|
| **What** | Protocol for atomic transactions across distributed systems |
| **When** | Banking, bookings, inventory - when you need ALL or NOTHING |
| **How** | Phase 1: Prepare (vote), Phase 2: Commit/Abort |
| **Coordinator** | Orchestrates the protocol, decides outcome |
| **Participants** | Execute local work, vote, wait for decision |
| **Recovery** | Write-Ahead Log (WAL) on both coordinator and participants |
| **Problem** | Blocking - participants stuck if coordinator dies after votes |
| **Alternative** | Saga pattern for eventual consistency |

---

*Next in the series: Saga Pattern - When 2PC is Too Slow*

---

**Author:** System Design Interview Series  
**Day:** 27 of 50  
**Topic:** Two-Phase Commit (2PC)
