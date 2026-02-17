# Optimistic vs Pessimistic Locking: Amazon vs BookMyShow
### Day 21 of 50 - System Design Interview Preparation Series

**By Sunchit Dudeja**

---

## 🎯 Welcome to Day 21!

Yesterday, we explored S3 Multipart Upload for large file handling. Today, we tackle one of the most critical concurrency patterns in system design: **How do Amazon and BookMyShow handle the same resource being accessed by multiple users simultaneously?**

> "Amazon assumes conflicts are rare. BookMyShow assumes conflicts are certain. Both are right — for their use cases."

---

## 🔍 THE CORE CONCEPT VISUALIZED

```
┌─────────────────────────────────────────────────────────────────┐
│              OPTIMISTIC vs PESSIMISTIC LOCKING                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   OPTIMISTIC LOCKING (Amazon):                                   │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │  "I'll read the data and assume no one else will        │  │
│   │   change it. If they did, I'll retry."                  │  │
│   │                                                          │  │
│   │   READ ──▶ PROCESS ──▶ CHECK VERSION ──▶ UPDATE         │  │
│   │                              │                           │  │
│   │                              ▼                           │  │
│   │                     Version mismatch? RETRY!             │  │
│   └─────────────────────────────────────────────────────────┘  │
│                                                                  │
│   PESSIMISTIC LOCKING (BookMyShow):                              │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │  "I'll lock the data FIRST. No one else can touch it    │  │
│   │   until I'm done."                                       │  │
│   │                                                          │  │
│   │   LOCK ──▶ READ ──▶ PROCESS ──▶ UPDATE ──▶ UNLOCK       │  │
│   │     │                                                    │  │
│   │     ▼                                                    │  │
│   │   Others WAIT here until lock released                   │  │
│   └─────────────────────────────────────────────────────────┘  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 🏪 AMAZON'S OPTIMISTIC LOCKING

### The Scenario

```
┌─────────────────────────────────────────────────────────────────┐
│              AMAZON'S PRODUCT PAGE SCENARIO                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   User A: Views iPhone 15 product page                           │
│   User B: Also views same iPhone 15                              │
│                                                                  │
│   Both see: "Price: ₹79,999, In Stock: 50"                      │
│                                                                  │
│   KEY INSIGHT: Conflicts are RARE                                │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │  Out of 1000 users viewing:                              │  │
│   │  • 990 just browse and leave                             │  │
│   │  • 9 add to cart but don't buy                           │  │
│   │  • 1 actually purchases                                  │  │
│   │                                                          │  │
│   │  Conflict probability: < 0.1%                            │  │
│   │  Why lock everyone? Let them read freely!                │  │
│   └─────────────────────────────────────────────────────────┘  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### How Amazon Implements It

```sql
-- Products table structure
CREATE TABLE products (
    id BIGINT PRIMARY KEY,
    name VARCHAR(255),
    price DECIMAL,
    stock_quantity INT,
    version BIGINT DEFAULT 0  -- ⚡ Version counter (THE KEY!)
);

-- Step 1: Read product (NO LOCK!)
SELECT id, name, price, stock_quantity, version 
FROM products 
WHERE id = 123;

-- Returns: iPhone 15, price 79999, stock 50, version 5

-- Step 2: User adds to cart, browses, proceeds to checkout
-- (All this time, no lock held!)

-- Step 3: At final purchase, UPDATE with version check
UPDATE products 
SET 
    stock_quantity = stock_quantity - 1,
    version = version + 1
WHERE 
    id = 123 
    AND version = 5;  -- ⚡ Only update if version still 5!

-- Step 4: Check if update succeeded
-- rows_updated == 1 → Success!
-- rows_updated == 0 → Someone changed it! Retry operation
```

### The Version Check Magic

```
┌─────────────────────────────────────────────────────────────────┐
│              HOW VERSION CHECK PREVENTS CONFLICTS                │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   User A reads: version = 5                                      │
│   User B reads: version = 5                                      │
│                                                                  │
│   User A updates:                                                │
│   UPDATE ... WHERE version = 5 → SUCCESS (version becomes 6)    │
│                                                                  │
│   User B updates (later):                                        │
│   UPDATE ... WHERE version = 5 → FAILS (version is now 6!)      │
│                                                                  │
│   The version mismatch tells us:                                 │
│   "Someone modified this data between your read and write"      │
│   → Reload data, show user changes, let them decide             │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Java Implementation with JPA

```java
@Entity
public class Product {
    @Id
    private Long id;
    private String name;
    private BigDecimal price;
    private Integer stockQuantity;
    
    @Version  // ⚡ JPA automatically manages this!
    private Long version;
    
    // Getters and setters
}

@Service
public class OrderService {
    
    @Autowired
    private ProductRepository productRepository;
    
    @Autowired
    private OrderRepository orderRepository;
    
    @Transactional
    public Order placeOrder(Long productId, Integer quantity) {
        // 1. Read product (version captured automatically)
        Product product = productRepository.findById(productId)
            .orElseThrow(() -> new ProductNotFoundException(productId));
        
        // 2. Validate stock
        if (product.getStockQuantity() < quantity) {
            throw new InsufficientStockException();
        }
        
        // 3. Update stock (JPA adds version check automatically!)
        product.setStockQuantity(product.getStockQuantity() - quantity);
        
        // 4. If version mismatch, OptimisticLockException thrown here
        productRepository.save(product);
        
        // 5. Create order
        Order order = new Order(product, quantity);
        return orderRepository.save(order);
    }
}
```

### Amazon's Retry Strategy

```java
@Service
public class OrderServiceWithRetry {
    
    private static final int MAX_RETRIES = 3;
    private static final long RETRY_DELAY_MS = 100;
    
    @Transactional(noRollbackFor = OptimisticLockException.class)
    public Order placeOrderWithRetry(Long productId, Integer quantity) {
        int retries = 0;
        
        while (retries < MAX_RETRIES) {
            try {
                return placeOrder(productId, quantity);
            } catch (OptimisticLockException e) {
                retries++;
                log.warn("Optimistic lock conflict, retry {} of {}", 
                         retries, MAX_RETRIES);
                
                if (retries >= MAX_RETRIES) {
                    throw new ConflictException(
                        "Product details changed. Please review and try again.", e
                    );
                }
                
                // Wait a bit before retrying
                try {
                    Thread.sleep(RETRY_DELAY_MS * retries);
                } catch (InterruptedException ie) {
                    Thread.currentThread().interrupt();
                }
                
                // Clear entity manager to get fresh data
                entityManager.clear();
            }
        }
        
        throw new RuntimeException("Should not reach here");
    }
}

@RestController
@RequestMapping("/api")
public class CheckoutController {
    
    @PostMapping("/checkout")
    public ResponseEntity<?> checkout(@RequestBody OrderRequest request) {
        try {
            Order order = orderService.placeOrderWithRetry(
                request.getProductId(), 
                request.getQuantity()
            );
            return ResponseEntity.ok(order);
            
        } catch (ConflictException e) {
            // After retries exhausted, tell user what changed
            Product currentProduct = productService.findById(request.getProductId());
            
            return ResponseEntity.status(HttpStatus.CONFLICT)
                .body(Map.of(
                    "message", "Product details changed during checkout",
                    "currentPrice", currentProduct.getPrice(),
                    "currentStock", currentProduct.getStockQuantity(),
                    "action", "Please review and confirm"
                ));
        }
    }
}
```

---

## 🎫 BOOKMYSHOW'S PESSIMISTIC LOCKING

### The Scenario

```
┌─────────────────────────────────────────────────────────────────┐
│              BOOKMYSHOW'S SEAT SELECTION SCENARIO                │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   User A: Selects seat A1 for Avengers at 7PM show              │
│   User B: Also tries to select seat A1 at EXACT same time       │
│                                                                  │
│   KEY INSIGHT: Conflicts are LIKELY                              │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │  For a popular movie on opening day:                     │  │
│   │  • 10,000 users viewing same showtime                    │  │
│   │  • 300 seats available                                   │  │
│   │  • Front row seats: 20 people fighting for 10 seats!     │  │
│   │                                                          │  │
│   │  Conflict probability: VERY HIGH                         │  │
│   │  Double-booking tolerance: ZERO                          │  │
│   │  → Must lock immediately!                                │  │
│   └─────────────────────────────────────────────────────────┘  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### How BookMyShow Implements It

```sql
-- Seats table structure
CREATE TABLE seats (
    id BIGINT PRIMARY KEY,
    show_id BIGINT,
    seat_number VARCHAR(10),
    status ENUM('AVAILABLE', 'BLOCKED', 'BOOKED'),
    locked_by BIGINT,
    locked_until TIMESTAMP
);

-- Step 1: User clicks on seat A1
BEGIN TRANSACTION;

-- Step 2: LOCK the seat immediately!
SELECT * FROM seats 
WHERE 
    show_id = 1001 
    AND seat_number = 'A1'
    AND status = 'AVAILABLE'
FOR UPDATE;  -- ⚡ Pessimistic lock acquired here!

-- At this point:
-- ✅ User A has the lock
-- ⏳ User B tries same query → WAITS (blocked)

-- Step 3: If seat found and locked successfully
UPDATE seats 
SET 
    status = 'BLOCKED',
    locked_by = 123,  -- user_id
    locked_until = NOW() + INTERVAL '5 minutes'
WHERE 
    show_id = 1001 
    AND seat_number = 'A1';

COMMIT;  -- Lock released, but seat is now BLOCKED

-- User has 5 minutes to complete payment
-- If they don't, background job releases the seat
```

### Java Implementation with JPA

```java
@Entity
public class Seat {
    @Id
    private Long id;
    
    private Long showId;
    private String seatNumber;
    
    @Enumerated(EnumType.STRING)
    private SeatStatus status;
    
    private Long lockedBy;
    private LocalDateTime lockedUntil;
    
    // Getters and setters
}

public enum SeatStatus {
    AVAILABLE,
    BLOCKED,  // Temporarily held for payment
    BOOKED    // Confirmed booking
}

@Repository
public interface SeatRepository extends JpaRepository<Seat, Long> {
    
    // ⚡ PESSIMISTIC_WRITE = SELECT ... FOR UPDATE
    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query("SELECT s FROM Seat s WHERE s.showId = :showId AND s.seatNumber = :seatNumber")
    Optional<Seat> findByShowIdAndSeatNumberForUpdate(
        @Param("showId") Long showId, 
        @Param("seatNumber") String seatNumber
    );
    
    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query("SELECT s FROM Seat s WHERE s.id = :id")
    Optional<Seat> findByIdForUpdate(@Param("id") Long id);
}

@Service
public class BookingService {
    
    private static final int LOCK_DURATION_MINUTES = 5;
    
    @Transactional
    public SeatLockResponse lockSeat(Long showId, String seatNumber, Long userId) {
        // 1. Acquire pessimistic lock (others will WAIT here)
        Seat seat = seatRepository.findByShowIdAndSeatNumberForUpdate(showId, seatNumber)
            .orElseThrow(() -> new SeatNotFoundException(showId, seatNumber));
        
        // 2. Check if seat is still available
        if (seat.getStatus() != SeatStatus.AVAILABLE) {
            throw new SeatAlreadyTakenException(seatNumber);
        }
        
        // 3. Check if locked by someone else (lock expired check)
        if (seat.getLockedBy() != null && 
            seat.getLockedUntil() != null &&
            seat.getLockedUntil().isAfter(LocalDateTime.now())) {
            throw new SeatAlreadyTakenException(seatNumber);
        }
        
        // 4. Lock the seat for this user
        seat.setStatus(SeatStatus.BLOCKED);
        seat.setLockedBy(userId);
        seat.setLockedUntil(LocalDateTime.now().plusMinutes(LOCK_DURATION_MINUTES));
        
        seatRepository.save(seat);
        
        return new SeatLockResponse(
            seat.getId(),
            seatNumber,
            LOCK_DURATION_MINUTES,
            "Seat held for " + LOCK_DURATION_MINUTES + " minutes"
        );
    }
    
    @Transactional
    public Booking confirmBooking(Long seatId, Long userId, PaymentDetails payment) {
        // 1. Again acquire lock to confirm
        Seat seat = seatRepository.findByIdForUpdate(seatId)
            .orElseThrow(() -> new SeatNotFoundException(seatId));
        
        // 2. Verify still locked by this user and not expired
        if (!userId.equals(seat.getLockedBy())) {
            throw new UnauthorizedBookingException("Seat not locked by this user");
        }
        
        if (seat.getLockedUntil().isBefore(LocalDateTime.now())) {
            throw new LockExpiredException("Your seat hold has expired");
        }
        
        // 3. Process payment (in real world, this would be separate)
        paymentService.processPayment(payment);
        
        // 4. Confirm booking
        seat.setStatus(SeatStatus.BOOKED);
        seat.setLockedBy(null);
        seat.setLockedUntil(null);
        seatRepository.save(seat);
        
        // 5. Create booking record
        Booking booking = new Booking(seat, userId, payment.getAmount());
        return bookingRepository.save(booking);
    }
}
```

### Background Job to Release Expired Locks

```java
@Component
public class ExpiredLockCleanupJob {
    
    @Scheduled(fixedRate = 60000)  // Every minute
    @Transactional
    public void releaseExpiredLocks() {
        int releasedCount = seatRepository.releaseExpiredLocks(LocalDateTime.now());
        
        if (releasedCount > 0) {
            log.info("Released {} expired seat locks", releasedCount);
        }
    }
}

@Repository
public interface SeatRepository extends JpaRepository<Seat, Long> {
    
    @Modifying
    @Query("""
        UPDATE Seat s 
        SET s.status = 'AVAILABLE', s.lockedBy = null, s.lockedUntil = null 
        WHERE s.status = 'BLOCKED' AND s.lockedUntil < :now
        """)
    int releaseExpiredLocks(@Param("now") LocalDateTime now);
}
```

---

## ⚔️ SIDE-BY-SIDE COMPARISON

```
┌─────────────────────────────────────────────────────────────────┐
│              OPTIMISTIC vs PESSIMISTIC - COMPARISON              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Aspect            │ Optimistic (Amazon) │ Pessimistic (BMS)   │
│   ──────────────────┼─────────────────────┼─────────────────────│
│   Assumption        │ Conflicts are RARE  │ Conflicts COMMON    │
│   When lock held    │ At UPDATE time      │ At READ time        │
│   What gets locked  │ Version number      │ Actual row/data     │
│   Other users       │ Can READ freely     │ Are BLOCKED/wait    │
│   Deadlock risk     │ LOW                 │ HIGHER              │
│   Performance       │ HIGH concurrency    │ LOWER concurrency   │
│   Implementation    │ @Version            │ SELECT FOR UPDATE   │
│   Retry needed?     │ YES, on conflict    │ NO, wait for lock   │
│   Best for          │ Product pages,      │ Seats, payments,    │
│                     │ profiles, inventory │ finite resources    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Decision Matrix

| Scenario | Use Optimistic | Use Pessimistic |
|----------|----------------|-----------------|
| E-commerce product inventory | ✅ | |
| User profile updates | ✅ | |
| Movie seat booking | | ✅ |
| Bank account transfers | | ✅ |
| Blog post comments | ✅ | |
| Flight seat reservation | | ✅ |
| Shopping cart updates | ✅ | |
| Auction bidding | | ✅ |
| Document editing | ✅ | |
| Ticket booking | | ✅ |

---

## 📊 REAL-TIME SCENARIO WALKTHROUGH

### Amazon Product Page (Optimistic)

```
┌─────────────────────────────────────────────────────────────────┐
│              AMAZON OPTIMISTIC LOCKING TIMELINE                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Time    User A                      User B                     │
│   ────    ───────────────────────     ───────────────────────   │
│   0:00    Views iPhone (version=5)                               │
│   0:01                                Views iPhone (version=5)   │
│   0:30    Adds to cart                                           │
│   1:00    Proceeds to checkout                                   │
│   1:30                                Adds to cart               │
│   2:00    Payment processing                                     │
│   2:30    UPDATE WHERE version=5                                 │
│           → SUCCESS ✅ (version=6)                               │
│   2:35                                Attempts purchase          │
│                                       UPDATE WHERE version=5     │
│                                       → FAILS ❌ (version is 6!) │
│                                       Shows "Stock changed"      │
│   3:00                                Refreshes (version=6)      │
│                                       Sees stock=49, decides     │
│                                                                  │
│   KEY: User A succeeded, User B informed of change              │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### BookMyShow Seat Selection (Pessimistic)

```
┌─────────────────────────────────────────────────────────────────┐
│              BOOKMYSHOW PESSIMISTIC LOCKING TIMELINE             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Time    User A                      User B                     │
│   ────    ───────────────────────     ───────────────────────   │
│   0:00    Clicks Seat A1                                         │
│   0:01    SELECT FOR UPDATE on A1                                │
│           LOCK ACQUIRED! ✅                                      │
│   0:02                                Clicks Seat A1             │
│                                       SELECT FOR UPDATE on A1   │
│                                       WAITING... ⏳              │
│   0:03    Updates seat status         (still waiting)            │
│   0:30    Completes payment ✅        (still waiting)            │
│           COMMIT → Lock released                                 │
│   0:31                                Lock acquired              │
│                                       Checks status → BOOKED    │
│                                       "Seat already taken" ❌    │
│                                       Redirects to seat map     │
│                                                                  │
│   KEY: User B waited, then got clear "taken" message            │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 🚨 DEADLOCK SCENARIO (What to AVOID)

### The Problem

```sql
-- Transaction 1 (User A selecting A1 and A2)
BEGIN;
SELECT * FROM seats WHERE seat_number = 'A1' FOR UPDATE;  -- Gets lock on A1
-- ... some processing ...
SELECT * FROM seats WHERE seat_number = 'A2' FOR UPDATE;  -- WAITS for A2

-- Transaction 2 (User B selecting A2 and A1) - Running simultaneously
BEGIN;
SELECT * FROM seats WHERE seat_number = 'A2' FOR UPDATE;  -- Gets lock on A2
-- ... some processing ...
SELECT * FROM seats WHERE seat_number = 'A1' FOR UPDATE;  -- WAITS for A1

-- DEADLOCK! Both waiting forever
-- Database will detect and kill one transaction
```

### The Solution: Lock in Consistent Order

```java
@Service
public class MultiSeatBookingService {
    
    @Transactional
    public List<SeatLock> lockMultipleSeats(Long showId, List<String> seatNumbers, Long userId) {
        // ⚡ Always sort seats to ensure consistent locking order!
        List<String> sortedSeats = seatNumbers.stream()
            .sorted()
            .collect(Collectors.toList());
        
        List<SeatLock> locks = new ArrayList<>();
        
        for (String seatNumber : sortedSeats) {
            // Lock in sorted order: A1 before A2, A2 before B1, etc.
            SeatLock lock = lockSeat(showId, seatNumber, userId);
            locks.add(lock);
        }
        
        return locks;
    }
}
```

### Lock Timeout Configuration

```java
@Configuration
public class JpaConfig {
    
    @Bean
    public JpaVendorAdapter jpaVendorAdapter() {
        HibernateJpaVendorAdapter adapter = new HibernateJpaVendorAdapter();
        return adapter;
    }
    
    @Bean
    public Properties jpaProperties() {
        Properties properties = new Properties();
        // Set lock timeout to 5 seconds (avoid infinite waits)
        properties.setProperty("javax.persistence.lock.timeout", "5000");
        return properties;
    }
}

// Or per-query timeout
@Lock(LockModeType.PESSIMISTIC_WRITE)
@QueryHints({@QueryHint(name = "javax.persistence.lock.timeout", value = "3000")})
@Query("SELECT s FROM Seat s WHERE s.id = :id")
Optional<Seat> findByIdForUpdateWithTimeout(@Param("id") Long id);
```

---

## 🎯 CHOOSING THE RIGHT STRATEGY

```java
public class LockingStrategySelector {
    
    public LockingStrategy select(UseCase useCase) {
        
        // High contention + Finite resource = Pessimistic
        if (useCase.isHighContention() && useCase.isFiniteResource()) {
            /*
             * Examples:
             * - Movie seat booking (300 seats, 10K users)
             * - Flight reservations
             * - Bank transfers
             * - Auction final bids
             */
            return PessimisticLocking.builder()
                .lockTimeout(Duration.ofSeconds(5))
                .retryOnTimeout(true)
                .build();
        }
        
        // Low contention + Version tracking = Optimistic
        if (useCase.isLowContention() && useCase.supportsVersioning()) {
            /*
             * Examples:
             * - Product catalog updates
             * - User profile edits
             * - Shopping cart
             * - Blog post edits
             */
            return OptimisticLocking.builder()
                .maxRetries(3)
                .retryDelay(Duration.ofMillis(100))
                .refreshOnRetry(true)
                .build();
        }
        
        // Distributed system = Use distributed locks
        if (useCase.isDistributed()) {
            /*
             * Examples:
             * - Cross-service resource locking
             * - Cluster-wide singleton operations
             * - Rate limiting across instances
             */
            return DistributedLocking.builder()
                .provider("Redis")
                .ttl(Duration.ofSeconds(30))
                .build();
        }
        
        // Default: Optimistic (safer, higher throughput)
        return OptimisticLocking.withDefaults();
    }
}
```

---

## 💡 PRODUCTION LESSONS

### Amazon's Optimistic Locking Success

```
┌─────────────────────────────────────────────────────────────────┐
│              WHY AMAZON CHOSE OPTIMISTIC                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Statistics:                                                    │
│   • 99.9% of product views never lead to purchase conflicts     │
│   • Retry mechanism handles <0.1% conflicts transparently       │
│   • Version field prevents lost updates with zero lock overhead │
│                                                                  │
│   User Experience:                                               │
│   • Occasionally: "Sorry, item just sold out"                   │
│   • User refreshes, sees new data, makes decision               │
│   • No waiting, no blocking, no timeouts                        │
│                                                                  │
│   System Benefits:                                               │
│   • Maximum throughput (no locks held during browsing)          │
│   • Simple implementation (@Version annotation)                  │
│   • Easy to scale (no lock coordination)                        │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### BookMyShow's Pessimistic Locking Necessity

```
┌─────────────────────────────────────────────────────────────────┐
│              WHY BOOKMYSHOW CHOSE PESSIMISTIC                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Business Requirements:                                         │
│   • ZERO tolerance for double-booking                           │
│   • Every seat conflict MUST be prevented                        │
│   • User trust depends on seat guarantee                        │
│                                                                  │
│   Implementation Details:                                        │
│   • 5-minute lock window (perfect for payment completion)       │
│   • Lock timeout prevents abandoned carts from blocking         │
│   • Deadlock detection with automatic retry                     │
│                                                                  │
│   Trade-offs Accepted:                                           │
│   • Lower concurrency (acceptable for finite seats)             │
│   • User might wait (prefer wait over double-book)              │
│   • More complex error handling                                  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## ❓ Interview Practice

### Question 1:
> "When would you use optimistic vs pessimistic locking?"

**Answer:**
> "Choose based on conflict probability and business impact. **Optimistic locking** for low-contention scenarios like Amazon's product pages — most users browse without buying, so conflicts are rare. Use version numbers and retry on conflict. **Pessimistic locking** for high-contention, zero-tolerance scenarios like BookMyShow's seat booking — many users competing for finite seats, and double-booking is unacceptable. Lock immediately with SELECT FOR UPDATE."

### Question 2:
> "How does @Version annotation work in JPA?"

**Answer:**
> "JPA automatically manages the version field. On every update, it increments the version and adds `WHERE version = ?` to the UPDATE query. If rows_affected = 0, someone else modified the record, and JPA throws OptimisticLockException. The application catches this, refreshes data, and retries. It's transparent to the business logic and requires no explicit lock management."

### Question 3:
> "How would you prevent deadlocks in pessimistic locking?"

**Answer:**
> "Three strategies: (1) **Lock ordering** — always acquire locks in a consistent order (e.g., sort seat IDs). (2) **Lock timeout** — set a maximum wait time using `javax.persistence.lock.timeout`. (3) **Deadlock detection** — databases like PostgreSQL and MySQL automatically detect deadlocks and kill one transaction. The application should catch this and retry."

---

## 🔗 Connecting to Previous Days

| Day | Concept | How It Connects |
|-----|---------|-----------------|
| Day 15 | Redis Single-Threaded | No locking needed (single-threaded) |
| Day 16 | Redis Sorted Sets | Atomic operations avoid locks |
| Day 18 | Redis Timeouts | Lock timeout configuration |

---

## ✅ Day 21 Action Items

1. **Implement @Version** in your JPA entities for optimistic locking
2. **Practice SELECT FOR UPDATE** queries in your database
3. **Set lock timeouts** to prevent indefinite waits
4. **Always lock resources in consistent order** to prevent deadlocks

---

## 💡 Key Takeaways

| Insight | Why It Matters |
|---------|----------------|
| **Optimistic = assume no conflict** | Higher throughput, retry on failure |
| **Pessimistic = assume conflict** | Guaranteed exclusion, wait for lock |
| **Version field** | Simple optimistic lock implementation |
| **FOR UPDATE** | SQL-level pessimistic lock |
| **Lock ordering** | Prevents deadlocks |

---

## 🎯 The Architect's Principle

> **Junior:** "I'll just add locks everywhere to be safe."
>
> **Architect:** "That's a recipe for performance disaster. Choose your locking strategy based on **business impact** of conflicts, not technical paranoia. Amazon can afford occasional retries — 99.9% of product views never result in conflicts. BookMyShow absolutely cannot afford double-booked seats — the 5-minute lock is worth the reduced concurrency. The question isn't 'should I lock?' but 'what happens if two users modify simultaneously?' If the answer is 'minor inconvenience,' use optimistic. If the answer is 'catastrophic data corruption,' use pessimistic."

---

*— Sunchit Dudeja*  
*Day 21 of 50: System Design Interview Preparation Series*

---

> 🎯 **Interview Edge:** When asked about concurrency, immediately ask: "What's the contention level and what's the cost of conflict?" This shows you understand the trade-off between throughput (optimistic) and correctness (pessimistic).

> 📢 **Real Impact:** Amazon handles millions of product updates with @Version. BookMyShow prevents double-booking for millions of seats daily with SELECT FOR UPDATE. Both patterns work — for their specific use cases.

---

> 💡 **Tomorrow (Day 22):** We'll explore **Consistent Hashing** — how Netflix, Discord, and Amazon distribute data across thousands of servers without rehashing everything when nodes join or leave.

