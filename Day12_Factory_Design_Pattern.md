# Factory Design Pattern: When Zomato Serves 3 Different Experiences
### Day 12 of 50 - System Design Interview Preparation Series

**By Sunchit Dudeja**

---

## 🎯 Welcome to Day 12!

Yesterday, we mastered Kubernetes scaling to handle millions of requests. Today, we dive into **design patterns** — starting with the Factory Pattern, one of the most frequently asked patterns in system design interviews.

> The Factory Pattern isn't about creating objects. It's about **hiding complexity** and **enabling change** without breaking existing code.

---

## 🍕 The Zomato Problem — Two Perspectives

### Developer's Approach

```java
public class ZomatoOrderService {
    
    public void placeOrder(String orderType, OrderDetails details) {
        
        if (orderType.equals("DELIVERY")) {
            DeliveryOrder order = new DeliveryOrder();
            order.setAddress(details.getAddress());
            order.setDeliveryCharge(40);
            order.process();
            
        } else if (orderType.equals("DINING")) {
            DiningOrder order = new DiningOrder();
            order.setRestaurant(details.getRestaurant());
            order.setTableSize(details.getGuests());
            order.process();
            
        } else if (orderType.equals("NIGHTLIFE")) {
            NightLifeOrder order = new NightLifeOrder();
            order.setVenue(details.getVenue());
            order.setCoverCharge(500);
            order.process();
        }
    }
}
```

*"Simple if-else. Works fine. Ship it."*

### Architect's Reality Check

```
Current State:      3 order types
6 Months Later:     +PICKUP, +SUBSCRIPTION, +CATERING, +CORPORATE
1 Year Later:       +GROCERY, +MEDICINES, +PET_FOOD, +INSTAMART

If-Else Hell:
├── 15 order types × 10 properties each = 150 lines of if-else
├── Every new type changes core OrderService
├── Testing nightmare (all paths in one method)
├── Violation of Open/Closed Principle
└── One bug in condition breaks entire ordering system
```

| Problem | Developer's Code | Architect's View |
|---------|------------------|------------------|
| Adding new order type | Modify OrderService | **Dangerous — touching core logic** |
| Testing | Test all 15 paths together | **Nightmare — high coupling** |
| Bug in DELIVERY logic | Entire OrderService affected | **Blast radius too large** |
| Different teams working | Merge conflicts daily | **Bottleneck on single file** |

**The developer's "simple" solution becomes the architect's maintenance nightmare.**

---

## 🔍 The Solution: Factory Design Pattern

The **Factory Pattern** delegates object creation to a specialized class, so the client doesn't need to know which concrete class to instantiate.

### The Concept

```
BEFORE FACTORY (Tight Coupling):
│
├── OrderService knows about DeliveryOrder
├── OrderService knows about DiningOrder
├── OrderService knows about NightLifeOrder
├── OrderService knows about PickupOrder
├── OrderService knows about SubscriptionOrder
│   ... (knows everything — violation of Single Responsibility)
│
└── Adding new type = Modifying OrderService

AFTER FACTORY (Loose Coupling):
│
├── OrderService knows about → OrderFactory → Order interface
│                                    │
│                              ┌─────┼─────────────────┐
│                              ▼     ▼                 ▼
│                         DELIVERY  DINING        NIGHTLIFE
│                              │     │                 │
│                              ▼     ▼                 ▼
│                        DeliveryOrder DiningOrder NightLifeOrder
│
└── Adding new type = Add new class + Update factory (OrderService untouched!)
```

---

## 🏗️ Factory Pattern: The Building Blocks

### Step 1: The Product Interface

```java
/**
 * The "Product" interface — all order types implement this.
 * OrderService only knows about this interface, not concrete classes.
 */
public interface Order {
    
    String getOrderId();
    void setOrderId(String orderId);
    
    String processOrder();      // Each type processes differently
    String getOrderType();      // DELIVERY, DINING, NIGHTLIFE
    int getEstimatedTime();     // Minutes until ready
    
    List<Dish> getDishes();
    double getTotalAmount();
}
```

### Step 2: Concrete Products

```java
/**
 * Concrete Product 1: Delivery Order
 * Knows everything about delivery — address, charges, rider assignment
 */
public class DeliveryOrder implements Order {
    
    private String orderId;
    private String customerName;
    private String deliveryAddress;
    private List<Dish> dishes;
    private static final double DELIVERY_CHARGE = 40.0;
    
    public DeliveryOrder(String customerName, String deliveryAddress, List<Dish> dishes) {
        this.customerName = customerName;
        this.deliveryAddress = deliveryAddress;
        this.dishes = dishes;
    }
    
    @Override
    public String processOrder() {
        // Delivery-specific logic:
        // 1. Validate delivery address
        // 2. Calculate delivery time based on distance
        // 3. Assign delivery partner
        // 4. Send notifications
        return String.format(
            "🚴 DELIVERY ORDER CONFIRMED!\n" +
            "Customer: %s\n" +
            "Delivery Address: %s\n" +
            "Total: ₹%.2f (includes ₹%.2f delivery)\n" +
            "Status: Preparing your food!",
            customerName, deliveryAddress, getTotalAmount(), DELIVERY_CHARGE
        );
    }
    
    @Override
    public double getTotalAmount() {
        return dishes.stream().mapToDouble(Dish::getTotal).sum() + DELIVERY_CHARGE;
    }
    
    @Override
    public int getEstimatedTime() {
        return 30; // 30 minutes for delivery
    }
    
    @Override
    public String getOrderType() {
        return "DELIVERY";
    }
    
    // ... other methods
}
```

```java
/**
 * Concrete Product 2: Dining Order
 * Knows everything about reservations — table size, time slots, special requests
 */
public class DiningOrder implements Order {
    
    private String orderId;
    private String customerName;
    private String restaurantName;
    private int numberOfGuests;
    private List<Dish> dishes;
    
    @Override
    public String processOrder() {
        // Dining-specific logic:
        // 1. Check table availability
        // 2. Reserve table for time slot
        // 3. Notify restaurant
        // 4. Send confirmation to customer
        return String.format(
            "🍽️ DINING RESERVATION CONFIRMED!\n" +
            "Customer: %s\n" +
            "Restaurant: %s\n" +
            "Guests: %d\n" +
            "Status: Your table is reserved!",
            customerName, restaurantName, numberOfGuests
        );
    }
    
    @Override
    public int getEstimatedTime() {
        return 0; // Immediate — table ready on arrival
    }
    
    @Override
    public String getOrderType() {
        return "DINING";
    }
}
```

```java
/**
 * Concrete Product 3: NightLife Order
 * Knows everything about clubs — cover charge, guest list, VIP tables
 */
public class NightLifeOrder implements Order {
    
    private String orderId;
    private String customerName;
    private String venueName;
    private int numberOfPeople;
    private List<Dish> dishes;
    private static final double COVER_CHARGE_PER_PERSON = 500.0;
    
    @Override
    public String processOrder() {
        // NightLife-specific logic:
        // 1. Add to guest list
        // 2. Reserve VIP table if requested
        // 3. Apply cover charge
        // 4. Send venue details
        return String.format(
            "🎉 NIGHTLIFE BOOKING CONFIRMED!\n" +
            "Customer: %s\n" +
            "Venue: %s\n" +
            "Party Size: %d\n" +
            "Cover Charge: ₹%.2f\n" +
            "Status: You're on the guest list!",
            customerName, venueName, numberOfPeople, 
            numberOfPeople * COVER_CHARGE_PER_PERSON
        );
    }
    
    @Override
    public int getEstimatedTime() {
        return 15; // 15 minutes for entry confirmation
    }
    
    @Override
    public String getOrderType() {
        return "NIGHTLIFE";
    }
}
```

### Step 3: The Factory (The Heart of the Pattern)

```java
/**
 * THE FACTORY — Creates appropriate Order based on type
 * 
 * This is where the magic happens:
 * - Client says "I want DELIVERY"
 * - Factory returns DeliveryOrder
 * - Client doesn't know (or care) about DeliveryOrder class
 */
@Component
public class OrderFactory {
    
    public Order createOrder(String orderType, String customerName, 
                             String locationOrVenue, int guestCount, 
                             List<Dish> dishes) {
        
        String type = orderType.toUpperCase().trim();
        
        switch (type) {
            case "DELIVERY":
                return new DeliveryOrder(customerName, locationOrVenue, dishes);
                
            case "DINING":
                return new DiningOrder(customerName, locationOrVenue, guestCount, dishes);
                
            case "NIGHTLIFE":
                return new NightLifeOrder(customerName, locationOrVenue, guestCount, dishes);
                
            default:
                throw new IllegalArgumentException(
                    "Unknown order type: " + orderType + 
                    ". Valid types: DELIVERY, DINING, NIGHTLIFE"
                );
        }
    }
}
```

---

## 🎥 Visualizing the Factory Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                    CLIENT (OrderService)                         │
│                                                                  │
│   "I need an order for type = DELIVERY"                         │
│                          │                                       │
│                          ▼                                       │
│   ┌──────────────────────────────────────────────────────────┐  │
│   │                    ORDER FACTORY                          │  │
│   │                                                           │  │
│   │   switch(orderType) {                                     │  │
│   │       case "DELIVERY"  → new DeliveryOrder()              │  │
│   │       case "DINING"    → new DiningOrder()                │  │
│   │       case "NIGHTLIFE" → new NightLifeOrder()             │  │
│   │   }                                                       │  │
│   └──────────────────────────────────────────────────────────┘  │
│                          │                                       │
│                          ▼                                       │
│   ┌──────────────────────────────────────────────────────────┐  │
│   │              Order Interface (returned)                   │  │
│   │                                                           │  │
│   │   - processOrder()     → "🚴 DELIVERY CONFIRMED!"         │  │
│   │   - getEstimatedTime() → 30 minutes                       │  │
│   │   - getTotalAmount()   → ₹1190.00                         │  │
│   └──────────────────────────────────────────────────────────┘  │
│                                                                  │
│   Client doesn't know it's DeliveryOrder — just uses Order!     │
└─────────────────────────────────────────────────────────────────┘
```

---

## 🔧 Production Implementation: Zomato Order API

### The Service Layer (Uses Factory)

```java
@Service
public class OrderService {
    
    private final OrderFactory orderFactory;
    private final Map<String, Order> orderStorage = new ConcurrentHashMap<>();
    
    public OrderService(OrderFactory orderFactory) {
        this.orderFactory = orderFactory;
    }
    
    public OrderResponse placeOrder(OrderRequest request) {
        try {
            // ⭐ FACTORY PATTERN IN ACTION ⭐
            // Service doesn't know about DeliveryOrder, DiningOrder, etc.
            // It only knows: "Give me an Order for this type"
            Order order = orderFactory.createOrder(
                request.getOrderType(),
                request.getCustomerName(),
                request.getLocationOrVenue(),
                request.getGuestCount(),
                request.getDishes()
            );
            
            // Generate and set order ID
            String orderId = "ORD-" + UUID.randomUUID().toString().substring(0, 8).toUpperCase();
            order.setOrderId(orderId);
            
            // Store in memory
            orderStorage.put(orderId, order);
            
            // Process — each order type handles this differently (polymorphism!)
            String confirmation = order.processOrder();
            
            return OrderResponse.success(
                orderId,
                order.getOrderType(),
                order.getDishes(),
                order.getTotalAmount(),
                confirmation,
                order.getEstimatedTime()
            );
            
        } catch (IllegalArgumentException e) {
            return OrderResponse.error(e.getMessage());
        }
    }
}
```

### The REST Controller

```java
@RestController
@RequestMapping("/api/orders")
public class OrderController {
    
    private final OrderService orderService;
    
    @PostMapping
    public ResponseEntity<OrderResponse> placeOrder(@RequestBody OrderRequest request) {
        OrderResponse response = orderService.placeOrder(request);
        
        if (response.isSuccess()) {
            return ResponseEntity.ok(response);
        }
        return ResponseEntity.badRequest().body(response);
    }
    
    @GetMapping("/{orderId}")
    public ResponseEntity<?> getOrder(@PathVariable String orderId) {
        Order order = orderService.getOrderById(orderId);
        if (order == null) {
            return ResponseEntity.notFound().build();
        }
        return ResponseEntity.ok(order);
    }
}
```

---

## 📊 API Testing with cURL

### 1. Place a DELIVERY Order

```bash
curl -X POST http://localhost:8080/api/orders \
  -H "Content-Type: application/json" \
  -d '{
    "orderType": "DELIVERY",
    "customerName": "Rahul",
    "locationOrVenue": "123 MG Road, Bangalore",
    "guestCount": 1,
    "dishes": [
      {"name": "Butter Chicken", "quantity": 2, "price": 350},
      {"name": "Naan", "quantity": 4, "price": 50},
      {"name": "Dal Makhani", "quantity": 1, "price": 250}
    ]
  }'
```

**Response:**
```json
{
  "success": true,
  "orderId": "ORD-F0B271F8",
  "orderType": "DELIVERY",
  "dishes": [
    {"name": "Butter Chicken", "quantity": 2, "price": 350.0, "total": 700.0},
    {"name": "Naan", "quantity": 4, "price": 50.0, "total": 200.0},
    {"name": "Dal Makhani", "quantity": 1, "price": 250.0, "total": 250.0}
  ],
  "totalAmount": 1190.0,
  "message": "🚴 DELIVERY ORDER CONFIRMED!...",
  "estimatedTimeMinutes": 30
}
```

### 2. Place a DINING Order

```bash
curl -X POST http://localhost:8080/api/orders \
  -H "Content-Type: application/json" \
  -d '{
    "orderType": "DINING",
    "customerName": "Priya",
    "locationOrVenue": "Taj Palace Restaurant",
    "guestCount": 4,
    "dishes": [
      {"name": "Paneer Tikka", "quantity": 2, "price": 280},
      {"name": "Biryani", "quantity": 4, "price": 320}
    ]
  }'
```

### 3. Place a NIGHTLIFE Order

```bash
curl -X POST http://localhost:8080/api/orders \
  -H "Content-Type: application/json" \
  -d '{
    "orderType": "NIGHTLIFE",
    "customerName": "Arjun",
    "locationOrVenue": "Club Infinity",
    "guestCount": 6,
    "dishes": [
      {"name": "Mojito", "quantity": 6, "price": 450},
      {"name": "Chicken Wings", "quantity": 2, "price": 350}
    ]
  }'
```

### 4. Fetch Order by ID

```bash
curl http://localhost:8080/api/orders/ORD-F0B271F8
```

---

## ⚡ Why Factory Pattern? The Benefits

| Benefit | Without Factory | With Factory |
|---------|-----------------|--------------|
| **Adding new order type** | Modify OrderService (risky) | Add new class + update Factory only |
| **Single Responsibility** | OrderService knows creation + processing | Factory creates, Service processes |
| **Open/Closed Principle** | Open for modification (bad) | Closed for modification, open for extension |
| **Testing** | Test all paths in one giant method | Test each order type independently |
| **Team Collaboration** | Everyone edits same file | Teams own their order types |
| **Dependency Management** | Client depends on all concrete classes | Client depends only on interface |

### Real-World Impact at Zomato Scale

```
┌─────────────────────────────────────────────────────────────────┐
│                    BEFORE FACTORY                                │
├─────────────────────────────────────────────────────────────────┤
│  OrderService.java: 2,500 lines                                 │
│  Every new order type: +200 lines, +merge conflicts             │
│  Bug in DELIVERY: Risk to all order types                       │
│  Deploy frequency: Weekly (too risky to deploy more)            │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                    AFTER FACTORY                                 │
├─────────────────────────────────────────────────────────────────┤
│  OrderService.java: 100 lines (delegates to Factory)            │
│  OrderFactory.java: 50 lines (just the switch)                  │
│  DeliveryOrder.java: 150 lines (owned by Delivery team)         │
│  DiningOrder.java: 120 lines (owned by Dining team)             │
│  NightLifeOrder.java: 130 lines (owned by NightLife team)       │
│                                                                  │
│  New order type: Add 1 class + 1 line in Factory                │
│  Bug in DELIVERY: Only DeliveryOrder affected                   │
│  Deploy frequency: Daily (isolated blast radius)                │
└─────────────────────────────────────────────────────────────────┘
```

---

## 🎯 When to Use Factory Pattern

### ✅ Perfect Use Cases

| Scenario | Why Factory Helps |
|----------|------------------|
| **Multiple product types** | Payment gateways (Razorpay, Stripe, PayU) |
| **Runtime type selection** | User chooses delivery/pickup at checkout |
| **Complex object creation** | Objects need validation, defaults, dependencies |
| **Decoupling client from implementation** | Client shouldn't know concrete classes |
| **Supporting future extensions** | New types added without changing core code |

### ❌ When NOT to Use

| Scenario | Why Not |
|----------|---------|
| **Only one implementation** | Overkill for single concrete class |
| **Simple object creation** | `new User(name, email)` doesn't need factory |
| **No future extensions expected** | Added complexity without benefit |
| **Performance critical path** | Factory adds minimal but measurable overhead |

---

## 🔄 Factory Pattern Variations

### 1. Simple Factory (What We Built)

```java
// Factory method in a separate class
public class OrderFactory {
    public Order createOrder(String type) {
        switch(type) {
            case "DELIVERY": return new DeliveryOrder();
            case "DINING": return new DiningOrder();
        }
    }
}
```

### 2. Factory Method (Subclass Decides)

```java
// Abstract creator with factory method
public abstract class OrderCreator {
    public abstract Order createOrder();
    
    public void processOrder() {
        Order order = createOrder();  // Subclass decides which Order
        order.process();
    }
}

public class DeliveryOrderCreator extends OrderCreator {
    @Override
    public Order createOrder() {
        return new DeliveryOrder();
    }
}
```

### 3. Abstract Factory (Family of Products)

```java
// Creates families of related objects
public interface ZomatoServiceFactory {
    Order createOrder();
    Payment createPayment();
    Notification createNotification();
}

public class DeliveryServiceFactory implements ZomatoServiceFactory {
    public Order createOrder() { return new DeliveryOrder(); }
    public Payment createPayment() { return new DeliveryPayment(); }
    public Notification createNotification() { return new DeliverySMS(); }
}
```

---

## 📊 Factory Pattern in Real Systems

| Company | Use Case | Factory Creates |
|---------|----------|-----------------|
| **Zomato** | Order Types | Delivery, Dining, NightLife orders |
| **Razorpay** | Payment Gateways | UPI, Card, NetBanking, Wallet handlers |
| **AWS SDK** | Cloud Services | S3Client, EC2Client, LambdaClient |
| **Spring** | Bean Creation | ApplicationContext is a giant factory |
| **JDBC** | Database Drivers | MySQL, PostgreSQL, Oracle connections |
| **Notification Service** | Channels | Email, SMS, Push, WhatsApp senders |

---

## ❓ Interview Practice

### Question 1:
> "When would you use Factory Pattern vs Builder Pattern?"

**Answer:**
> "Factory Pattern is for creating different types of objects when the type is determined at runtime — like Zomato creating Delivery vs Dining orders based on user choice. Builder Pattern is for creating complex objects with many optional parameters step-by-step — like building a Pizza with optional cheese, toppings, size. Use Factory when 'which class' varies; use Builder when 'object configuration' varies."

### Question 2:
> "How does Factory Pattern support the Open/Closed Principle?"

**Answer:**
> "The Open/Closed Principle states classes should be open for extension but closed for modification. With Factory Pattern, adding a new order type like PICKUP requires: (1) creating a new PickupOrder class implementing Order interface, and (2) adding one case in the Factory switch. The OrderService, Controllers, and all existing order classes remain untouched. The system is extended without modifying existing code."

### Question 3:
> "Design a notification system using Factory Pattern."

**Answer:**
> "I'd create a Notification interface with send() method. Concrete implementations: EmailNotification, SMSNotification, PushNotification, WhatsAppNotification. NotificationFactory takes channel type and returns appropriate sender. The NotificationService calls factory.create(channelType).send(message). Adding Slack notifications later means: create SlackNotification class, add case in factory. All existing notification logic untouched."

---

## 🔗 Connecting to Previous Days

| Day | Concept | How It Connects |
|-----|---------|-----------------|
| Day 6 | Design for Failure | Factory enables graceful fallback (if Razorpay fails, factory returns PayU) |
| Day 7 | Databases | Repository Factory creates MySQL/PostgreSQL/MongoDB repos based on config |
| Day 8 | Load Balancing | Load balancer factory creates different algorithms (RoundRobin, LeastConn) |
| Day 11 | Kubernetes | HPA/VPA selection via factory based on workload type |

---

## ✅ Day 12 Action Items

1. **Identify factory opportunities** — Where does your code use if-else for object creation?
2. **Extract interfaces** — Create common interface for related objects
3. **Build a factory** — Centralize object creation logic
4. **Add new type as test** — Verify adding new implementation doesn't touch existing code
5. **Consider Spring's @Component** — Let Spring's DI container act as factory

---

## 💡 Lessons Learned

| Lesson | Why It Matters |
|--------|----------------|
| If-else for object creation is a code smell | Violates Open/Closed Principle |
| Client should depend on interfaces | Not concrete implementations |
| Factory centralizes change | New types = 1 class + 1 factory line |
| Polymorphism is the magic | Each object handles its own logic |
| Design patterns enable team scaling | Different teams own different products |

---

## 🚀 Key Architect Principles

| Principle | What It Means |
|-----------|---------------|
| **Program to interface, not implementation** | OrderService knows Order, not DeliveryOrder |
| **Encapsulate what varies** | Object creation varies → encapsulate in Factory |
| **Single Responsibility** | Factory creates, Service processes, Controller routes |
| **Open for extension, closed for modification** | Add classes, don't modify existing ones |
| **Favor composition over inheritance** | Factory composes objects, doesn't inherit |

---

## 💡 Key Takeaway

> **Developer: "I'll just add another if-else for the new order type."**  
> **Architect: "Let's use Factory Pattern — adding PICKUP order should be one new class, one factory line, zero changes to existing code. That's how we scale to 50 developers working on 20 order types without stepping on each other."**

The difference? Understanding that **object creation is a responsibility** that deserves its own class. When creation logic is scattered across your codebase, every new feature becomes a risk. Factory Pattern centralizes that risk into a single, testable, maintainable location.

---

*— Sunchit Dudeja*  
*Day 12 of 50: System Design Interview Preparation Series*

---

> 🎯 **Interview Edge:** When discussing object creation, don't just say "I'll use Factory Pattern." Explain WHY: "Factory Pattern gives us loose coupling — the OrderService depends on Order interface, not concrete classes. This means adding new order types doesn't require modifying existing code, which reduces deployment risk and enables parallel team development."

> 📢 **Real Impact:** At Zomato, moving to Factory Pattern for order creation reduced deployment conflicts by 80% and enabled the team to add new order types (like Instamart grocery delivery) without touching the core ordering system — shipping in days instead of weeks.

---

## 🔗 Code Repository

Full working implementation with Spring Boot REST API:
- **GitHub:** Check the `Day12_Factory_Pattern` folder
- **Run:** `./mvnw spring-boot:run`
- **Test:** Use the cURL commands above

---

> 💡 **Tomorrow (Day 13):** We'll explore the **Strategy Pattern** — when the algorithm varies, not just the object type. How does Zomato switch between different delivery fee calculation strategies? Stay tuned!

