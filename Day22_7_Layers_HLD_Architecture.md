# The 7 Layers of Every High-Level Design: A Complete Architecture Blueprint
### Day 22 of 50 - System Design Interview Preparation Series

**By Sunchit Dudeja**

---

## 🎯 Welcome to Day 22!

Yesterday, we explored Optimistic vs Pessimistic Locking. Today, we dive into the **universal blueprint that powers every large-scale system** — from Amazon to Netflix to Uber. Master these 7 layers, and you'll ace any HLD interview.

> "Every system design interview follows the same pattern: Client → Edge → Application → Cache → Async → Data → Observability. Master this, and you've mastered 80% of system design."

---

## 🏗️ THE COMPLETE ARCHITECTURE OVERVIEW

```
┌─────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                           THE 7 LAYERS OF HIGH-LEVEL DESIGN                                          │
├─────────┬──────────────┬─────────────┬──────────┬─────────────┬────────────────────┬────────────────┤
│ LAYER 1 │   LAYER 2    │   LAYER 3   │ LAYER 4  │   LAYER 5   │      LAYER 6       │    LAYER 7     │
│ CLIENT  │    EDGE      │ APPLICATION │  CACHE   │    ASYNC    │       DATA         │ OBSERVABILITY  │
├─────────┼──────────────┼─────────────┼──────────┼─────────────┼────────────────────┼────────────────┤
│         │   🌐 DNS     │             │          │             │  📖 Read Replicas  │                │
│         │   (GeoDNS)   │  🖥️ App 1   │ ⚡ Read  │             │    ┌───┐ ┌───┐    │  📊 Logging &  │
│         │      ↓       │             │  Cache   │             │    │ R │ │ R │    │    Metrics     │
│         │   ☁️ CDN     │  🖥️ App 2   │ (Redis)  │ 📨 Message  │    └───┘ └───┘    │  (Prometheus)  │
│ 👤 User │   (Static)   │             │          │   Queue     │         ↓          │       ↓        │
│  Web/   │      ↓       │  🖥️ App 3   │ 📝 Write │  (Kafka)    │  🗄️ Primary DB    │  🚨 Monitoring │
│ Mobile  │ ⚖️ Load      │             │  Buffer  │      ↓      │   (PostgreSQL)     │   & Alerts     │
│         │  Balancer    │ (Stateless) │          │ ⚙️ Workers  │         ↓          │  (PagerDuty)   │
│         │   (Nginx)    │             │          │  (Celery)   │  🔀 Sharded DBs    │       ↓        │
│         │              │             │          │             │  ┌──┐┌──┐┌──┐┌──┐ │  🤖 Auto-Scale │
│         │              │             │          │             │  │S1││S2││S3││S4│ │    (K8s HPA)   │
│         │              │             │          │             │  └──┘└──┘└──┘└──┘ │                │
│         │              │             │          │             │  🔍 NoSQL/Search  │                │
│         │              │             │          │             │  📅 Partitions    │                │
└─────────┴──────────────┴─────────────┴──────────┴─────────────┴────────────────────┴────────────────┘
```

---

## 📋 LAYER-BY-LAYER BREAKDOWN

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                          THE 7 LAYERS AT A GLANCE                                        │
├─────────┬────────────────────────────┬───────────────────────────┬──────────────────────┤
│  Layer  │        Purpose             │       Components          │    Real-World Tools  │
├─────────┼────────────────────────────┼───────────────────────────┼──────────────────────┤
│ 1. Client│ Entry point for users     │ Web browsers, Mobile apps │ Chrome, iOS, Android │
│ 2. Edge  │ Traffic routing & CDN     │ DNS, CDN, Load Balancer   │ Route53, CloudFront  │
│ 3. App   │ Business logic execution  │ Stateless servers (3x)    │ Spring Boot, Node.js │
│ 4. Cache │ Speed up reads/writes     │ Read Cache, Write Buffer  │ Redis, Memcached     │
│ 5. Async │ Background job processing │ Message Queue, Workers    │ Kafka, RabbitMQ      │
│ 6. Data  │ Persistent storage        │ Primary DB, Replicas      │ PostgreSQL, MongoDB  │
│ 7. Obs   │ Monitor & auto-heal       │ Logs, Alerts, Auto-scale  │ Prometheus, Grafana  │
└─────────┴────────────────────────────┴───────────────────────────┴──────────────────────┘
```

---

## 👤 LAYER 1: CLIENT LAYER

### What It Does

The entry point for all user interactions — where the journey begins.

```
┌─────────────────────────────────────────────────────────────────┐
│                    LAYER 1: CLIENT                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Components:                                                    │
│   ┌─────────────┐  ┌─────────────┐  ┌─────────────┐            │
│   │ 🌐 Web      │  │ 📱 Mobile   │  │ 🖥️ Desktop  │            │
│   │ (React,    │  │ (iOS,       │  │ (Electron,  │            │
│   │  Angular)   │  │  Android)   │  │  Native)    │            │
│   └─────────────┘  └─────────────┘  └─────────────┘            │
│                                                                  │
│   Responsibilities:                                              │
│   • UI rendering and user interaction                           │
│   • Client-side validation                                      │
│   • State management (Redux, MobX)                              │
│   • API calls to backend                                        │
│   • Caching (Service Workers, LocalStorage)                     │
│   • Offline support (PWA)                                       │
│                                                                  │
│   Design Considerations:                                         │
│   • Minimize bundle size for fast loading                       │
│   • Implement debouncing for search inputs                      │
│   • Use lazy loading for images/components                      │
│   • Handle network failures gracefully                          │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Real-World Example

```javascript
// Client-side debouncing (prevents overloading server)
const searchInput = document.getElementById('search');
let debounceTimer;

searchInput.addEventListener('input', (e) => {
    clearTimeout(debounceTimer);
    debounceTimer = setTimeout(() => {
        // Only call API after 300ms of no typing
        fetch(`/api/search?q=${e.target.value}`);
    }, 300);
});
```

---

## 🌐 LAYER 2: EDGE LAYER

### What It Does

Routes traffic, serves static content, and protects the application from the internet.

```
┌─────────────────────────────────────────────────────────────────┐
│                    LAYER 2: EDGE                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │                     🌐 DNS (GeoDNS)                      │  │
│   │                     (Route53, Cloudflare)                │  │
│   │                                                          │  │
│   │   User in India    → Mumbai server IP                   │  │
│   │   User in USA      → Virginia server IP                 │  │
│   │   User in Europe   → Frankfurt server IP                │  │
│   └─────────────────────────────────────────────────────────┘  │
│                              ↓                                   │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │                      ☁️ CDN                              │  │
│   │              (CloudFront, Akamai, Fastly)               │  │
│   │                                                          │  │
│   │   Serves: Images, CSS, JS, Videos, Static HTML          │  │
│   │   Benefits: 100+ edge locations globally                │  │
│   │   Cache Hit: ~5ms | Origin Fetch: ~200ms                │  │
│   └─────────────────────────────────────────────────────────┘  │
│                              ↓                                   │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │                  ⚖️ LOAD BALANCER                        │  │
│   │              (Nginx, HAProxy, AWS ALB)                  │  │
│   │                                                          │  │
│   │   Algorithms:                                            │  │
│   │   • Round Robin (default)                                │  │
│   │   • Least Connections (for varying request times)       │  │
│   │   • IP Hash (for session affinity)                      │  │
│   │   • Weighted (for A/B testing)                          │  │
│   │                                                          │  │
│   │   Health Checks: /health every 10s                      │  │
│   │   Unhealthy → Remove from pool                          │  │
│   └─────────────────────────────────────────────────────────┘  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Load Balancer Configuration

```nginx
# Nginx Load Balancer Config
upstream backend_servers {
    least_conn;  # Use least connections algorithm
    
    server app1.internal:8080 weight=3;  # 3x traffic
    server app2.internal:8080 weight=2;  # 2x traffic
    server app3.internal:8080 weight=1;  # 1x traffic (new server)
    
    # Health check
    check interval=3000 rise=2 fall=3 timeout=1000 type=http;
    check_http_send "GET /health HTTP/1.0\r\n\r\n";
    check_http_expect_alive http_2xx http_3xx;
}

server {
    listen 80;
    location / {
        proxy_pass http://backend_servers;
        proxy_connect_timeout 5s;
        proxy_read_timeout 60s;
    }
}
```

---

## 🖥️ LAYER 3: APPLICATION LAYER

### What It Does

Executes business logic — the brain of the system.

```
┌─────────────────────────────────────────────────────────────────┐
│                 LAYER 3: APPLICATION                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   KEY PRINCIPLE: STATELESS SERVERS                              │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │  ❌ Stateful: Session stored in server memory            │  │
│   │     → Can't scale horizontally                           │  │
│   │     → Server dies = session lost                         │  │
│   │                                                          │  │
│   │  ✅ Stateless: Session stored externally (Redis)        │  │
│   │     → Any server can handle any request                  │  │
│   │     → Scale by adding more servers                       │  │
│   │     → Server dies = no data loss                         │  │
│   └─────────────────────────────────────────────────────────┘  │
│                                                                  │
│   Architecture:                                                  │
│   ┌──────────┐    ┌──────────┐    ┌──────────┐                │
│   │ 🖥️ App 1 │    │ 🖥️ App 2 │    │ 🖥️ App 3 │                │
│   │ (Node.js)│    │(Spring)  │    │ (Go)     │                │
│   └────┬─────┘    └────┬─────┘    └────┬─────┘                │
│        │               │               │                        │
│        └───────────────┼───────────────┘                        │
│                        ↓                                         │
│              ┌──────────────────┐                                │
│              │ Shared Session   │                                │
│              │ Store (Redis)    │                                │
│              └──────────────────┘                                │
│                                                                  │
│   Each server handles:                                           │
│   • API request processing                                      │
│   • Authentication/Authorization                                │
│   • Business logic execution                                    │
│   • Input validation                                            │
│   • Response formatting (JSON)                                  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Stateless Server Implementation

```java
@RestController
@RequestMapping("/api")
public class ProductController {
    
    @Autowired
    private RedisTemplate<String, Object> redisTemplate;  // External session store
    
    @Autowired
    private ProductService productService;
    
    @GetMapping("/products/{id}")
    public Product getProduct(
            @PathVariable Long id,
            @RequestHeader("Authorization") String token) {
        
        // 1. Validate token (stateless - token contains all info)
        User user = jwtService.validateAndGetUser(token);
        
        // 2. Check cache first (external state)
        String cacheKey = "product:" + id;
        Product cached = (Product) redisTemplate.opsForValue().get(cacheKey);
        if (cached != null) {
            return cached;
        }
        
        // 3. Fetch from DB and cache
        Product product = productService.findById(id);
        redisTemplate.opsForValue().set(cacheKey, product, 1, TimeUnit.HOURS);
        
        return product;
    }
}
```

---

## ⚡ LAYER 4: CACHE LAYER

### What It Does

Speeds up reads and buffers writes — the performance multiplier.

```
┌─────────────────────────────────────────────────────────────────┐
│                    LAYER 4: CACHE                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   TWO TYPES OF CACHING:                                          │
│                                                                  │
│   1️⃣ READ CACHE (Speed up reads)                                │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │  ⚡ Redis / Memcached                                    │  │
│   │                                                          │  │
│   │  Flow:                                                   │  │
│   │  Request → Check Cache → HIT? Return cached data        │  │
│   │                        → MISS? Query DB, Cache result   │  │
│   │                                                          │  │
│   │  Cache-Aside Pattern:                                    │  │
│   │  ┌─────────┐    ┌─────────┐    ┌─────────┐            │  │
│   │  │   App   │───▶│  Cache  │───▶│   DB    │            │  │
│   │  │ Server  │◀───│ (Redis) │◀───│(Postgres)│            │  │
│   │  └─────────┘    └─────────┘    └─────────┘            │  │
│   │                                                          │  │
│   │  Hit Rate Target: 95%+ for production systems           │  │
│   │  Redis latency: 0.1-1ms | DB latency: 5-50ms           │  │
│   └─────────────────────────────────────────────────────────┘  │
│                                                                  │
│   2️⃣ WRITE BUFFER (Speed up writes)                            │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │  📝 Write-Behind Cache                                   │  │
│   │                                                          │  │
│   │  Flow:                                                   │  │
│   │  Write Request → Update Cache → Return Success          │  │
│   │                → Background: Flush to DB every 5s       │  │
│   │                                                          │  │
│   │  Use Cases:                                              │  │
│   │  • View counts (can lose a few)                         │  │
│   │  • Analytics events                                      │  │
│   │  • Activity logs                                         │  │
│   │                                                          │  │
│   │  ⚠️  NOT for: Financial transactions, Orders            │  │
│   └─────────────────────────────────────────────────────────┘  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Cache Implementation

```java
@Service
public class ProductCacheService {
    
    @Autowired
    private RedisTemplate<String, Product> redisTemplate;
    
    @Autowired
    private ProductRepository productRepository;
    
    private static final long CACHE_TTL_HOURS = 1;
    
    // Cache-Aside Pattern
    public Product getProduct(Long productId) {
        String cacheKey = "product:" + productId;
        
        // 1. Try cache first
        Product cached = redisTemplate.opsForValue().get(cacheKey);
        if (cached != null) {
            log.info("Cache HIT for product: {}", productId);
            return cached;
        }
        
        log.info("Cache MISS for product: {}", productId);
        
        // 2. Fetch from DB
        Product product = productRepository.findById(productId)
            .orElseThrow(() -> new ProductNotFoundException(productId));
        
        // 3. Populate cache
        redisTemplate.opsForValue().set(cacheKey, product, CACHE_TTL_HOURS, TimeUnit.HOURS);
        
        return product;
    }
    
    // Cache invalidation on update
    public Product updateProduct(Long productId, ProductUpdateRequest request) {
        Product product = productRepository.findById(productId)
            .orElseThrow(() -> new ProductNotFoundException(productId));
        
        product.setPrice(request.getPrice());
        product.setStock(request.getStock());
        
        // 1. Update DB
        Product saved = productRepository.save(product);
        
        // 2. Invalidate cache
        String cacheKey = "product:" + productId;
        redisTemplate.delete(cacheKey);
        
        return saved;
    }
}
```

---

## 📨 LAYER 5: ASYNC LAYER

### What It Does

Decouples components and handles background processing — the reliability backbone.

```
┌─────────────────────────────────────────────────────────────────┐
│                    LAYER 5: ASYNC                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   WHY ASYNC? The Restaurant Analogy                             │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │  ❌ Synchronous (Bad):                                   │  │
│   │     Customer orders → Waiter goes to kitchen → Cooks    │  │
│   │     → Waiter waits → Brings food → Takes next order     │  │
│   │     Throughput: 1 customer at a time!                    │  │
│   │                                                          │  │
│   │  ✅ Asynchronous (Good):                                 │  │
│   │     Customer orders → Waiter writes ticket → Kitchen    │  │
│   │     → Waiter takes next order immediately               │  │
│   │     → Food ready? Runner delivers it                    │  │
│   │     Throughput: 10+ customers simultaneously!            │  │
│   └─────────────────────────────────────────────────────────┘  │
│                                                                  │
│   COMPONENTS:                                                    │
│                                                                  │
│   1️⃣ MESSAGE QUEUE (The Kitchen Ticket System)                 │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │  📨 Kafka / RabbitMQ / SQS                               │  │
│   │                                                          │  │
│   │  Producers         Queue            Consumers            │  │
│   │  ┌─────┐        ┌───────────┐      ┌─────────┐         │  │
│   │  │ App │───────▶│ ■■■■■■■■■ │─────▶│ Worker1 │         │  │
│   │  │ App │───────▶│ ■■■■■■■■■ │─────▶│ Worker2 │         │  │
│   │  │ App │───────▶│ ■■■■■■■■■ │─────▶│ Worker3 │         │  │
│   │  └─────┘        └───────────┘      └─────────┘         │  │
│   │                                                          │  │
│   │  Benefits:                                               │  │
│   │  • Decoupling: Producer doesn't wait for consumer       │  │
│   │  • Buffering: Handle traffic spikes                     │  │
│   │  • Reliability: Messages persist even if workers die    │  │
│   │  • Scale: Add more workers during peak                  │  │
│   └─────────────────────────────────────────────────────────┘  │
│                                                                  │
│   2️⃣ WORKERS (The Kitchen Staff)                               │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │  ⚙️ Celery / Sidekiq / AWS Lambda                       │  │
│   │                                                          │  │
│   │  Worker Types:                                           │  │
│   │  • Email Worker: Send confirmation emails               │  │
│   │  • Payment Worker: Process payments                     │  │
│   │  • Notification Worker: Push notifications              │  │
│   │  • Analytics Worker: Process clickstream               │  │
│   │  • Image Worker: Resize/compress images                 │  │
│   └─────────────────────────────────────────────────────────┘  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Async Processing Implementation

```java
// Producer: Order Service
@Service
public class OrderService {
    
    @Autowired
    private KafkaTemplate<String, OrderEvent> kafkaTemplate;
    
    public OrderResponse placeOrder(OrderRequest request) {
        // 1. Validate and save order to DB (synchronous)
        Order order = orderRepository.save(new Order(request));
        
        // 2. Send to queue for async processing (non-blocking)
        OrderEvent event = new OrderEvent(order.getId(), "ORDER_PLACED");
        kafkaTemplate.send("orders-topic", order.getId().toString(), event);
        
        // 3. Immediately return to user
        return new OrderResponse(order.getId(), "Order received! Processing...");
    }
}

// Consumer: Order Worker
@Service
public class OrderWorker {
    
    @KafkaListener(topics = "orders-topic", groupId = "order-processors")
    public void processOrder(OrderEvent event) {
        log.info("Processing order: {}", event.getOrderId());
        
        try {
            // These happen in background, user doesn't wait!
            paymentService.chargeCustomer(event.getOrderId());
            inventoryService.reserveStock(event.getOrderId());
            notificationService.sendConfirmation(event.getOrderId());
            
            log.info("Order {} processed successfully", event.getOrderId());
        } catch (Exception e) {
            // Failed? Retry or send to dead letter queue
            log.error("Order {} failed: {}", event.getOrderId(), e.getMessage());
            throw e;  // Kafka will retry
        }
    }
}
```

---

## 🗄️ LAYER 6: DATA LAYER

### What It Does

Stores data persistently — the foundation of truth.

```
┌─────────────────────────────────────────────────────────────────┐
│                    LAYER 6: DATA                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   DATABASE ARCHITECTURE:                                         │
│                                                                  │
│   1️⃣ PRIMARY DATABASE (Source of Truth)                        │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │  🗄️ PostgreSQL / MySQL                                  │  │
│   │                                                          │  │
│   │  Handles: All WRITES                                     │  │
│   │  Features: ACID transactions, Strong consistency        │  │
│   └─────────────────────────────────────────────────────────┘  │
│                          │                                       │
│                    Replication                                   │
│                          ↓                                       │
│   2️⃣ READ REPLICAS (Scale Reads)                               │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │  📖 Replica 1     📖 Replica 2     📖 Replica 3         │  │
│   │                                                          │  │
│   │  Handles: All READS (80-90% of traffic)                 │  │
│   │  Lag: 10-100ms behind primary                           │  │
│   │  Scale: Add more replicas as read traffic grows         │  │
│   └─────────────────────────────────────────────────────────┘  │
│                                                                  │
│   3️⃣ SHARDING (Scale Writes)                                   │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │  🔀 Horizontal Partitioning by user_id                  │  │
│   │                                                          │  │
│   │  ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐          │  │
│   │  │ Shard1 │ │ Shard2 │ │ Shard3 │ │ Shard4 │          │  │
│   │  │ A-F    │ │ G-L    │ │ M-R    │ │ S-Z    │          │  │
│   │  └────────┘ └────────┘ └────────┘ └────────┘          │  │
│   │                                                          │  │
│   │  Shard Key: user_id, order_id, or tenant_id            │  │
│   │  Each shard: Independent primary + replicas            │  │
│   └─────────────────────────────────────────────────────────┘  │
│                                                                  │
│   4️⃣ SPECIALIZED DATABASES                                     │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │  🔍 Elasticsearch - Full-text search                    │  │
│   │  📊 MongoDB - Flexible documents                        │  │
│   │  ⏱️ TimescaleDB - Time-series data                      │  │
│   │  🗺️ Neo4j - Graph relationships                        │  │
│   └─────────────────────────────────────────────────────────┘  │
│                                                                  │
│   5️⃣ PARTITIONING (Archive Old Data)                           │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │  📅 Time-based partitions                                │  │
│   │                                                          │  │
│   │  orders_2024_01 │ orders_2024_02 │ orders_2023_archive │  │
│   │     (Hot)       │     (Warm)     │       (Cold)        │  │
│   │      SSD        │      SSD       │        HDD          │  │
│   └─────────────────────────────────────────────────────────┘  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Read/Write Splitting

```java
@Configuration
public class DataSourceConfig {
    
    @Bean
    @Primary
    public DataSource routingDataSource() {
        Map<Object, Object> targetDataSources = new HashMap<>();
        targetDataSources.put("primary", primaryDataSource());
        targetDataSources.put("replica", replicaDataSource());
        
        ReadWriteRoutingDataSource routingDataSource = new ReadWriteRoutingDataSource();
        routingDataSource.setTargetDataSources(targetDataSources);
        routingDataSource.setDefaultTargetDataSource(primaryDataSource());
        
        return routingDataSource;
    }
}

public class ReadWriteRoutingDataSource extends AbstractRoutingDataSource {
    
    @Override
    protected Object determineCurrentLookupKey() {
        return TransactionSynchronizationManager.isCurrentTransactionReadOnly()
            ? "replica" 
            : "primary";
    }
}

// Usage
@Service
public class ProductService {
    
    @Transactional(readOnly = true)  // Routes to replica
    public Product findById(Long id) {
        return productRepository.findById(id).orElseThrow();
    }
    
    @Transactional  // Routes to primary
    public Product save(Product product) {
        return productRepository.save(product);
    }
}
```

---

## 📊 LAYER 7: OBSERVABILITY LAYER

### What It Does

Monitors, alerts, and auto-heals — the immune system of your application.

```
┌─────────────────────────────────────────────────────────────────┐
│                 LAYER 7: OBSERVABILITY                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   THE THREE PILLARS OF OBSERVABILITY:                           │
│                                                                  │
│   1️⃣ LOGGING (What happened?)                                  │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │  📝 Tools: ELK Stack, Splunk, Datadog                   │  │
│   │                                                          │  │
│   │  App Servers → Fluentd → Elasticsearch → Kibana         │  │
│   │                                                          │  │
│   │  Log Format (Structured JSON):                          │  │
│   │  {                                                       │  │
│   │    "timestamp": "2024-01-15T10:30:00Z",                 │  │
│   │    "level": "ERROR",                                     │  │
│   │    "service": "order-service",                          │  │
│   │    "trace_id": "abc123",                                │  │
│   │    "message": "Payment failed",                         │  │
│   │    "user_id": "user_456"                                │  │
│   │  }                                                       │  │
│   └─────────────────────────────────────────────────────────┘  │
│                                                                  │
│   2️⃣ METRICS (How is it performing?)                           │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │  📊 Tools: Prometheus, Grafana, CloudWatch              │  │
│   │                                                          │  │
│   │  Key Metrics:                                            │  │
│   │  • Request rate (RPS)                                   │  │
│   │  • Error rate (%)                                        │  │
│   │  • Latency (P50, P95, P99)                              │  │
│   │  • CPU/Memory usage                                      │  │
│   │  • Cache hit rate                                        │  │
│   │  • Queue depth                                           │  │
│   │  • Active connections                                    │  │
│   └─────────────────────────────────────────────────────────┘  │
│                                                                  │
│   3️⃣ TRACING (What was the journey?)                           │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │  🔍 Tools: Jaeger, Zipkin, AWS X-Ray                    │  │
│   │                                                          │  │
│   │  Request trace across services:                         │  │
│   │  API Gateway (5ms)                                      │  │
│   │    └─▶ Auth Service (10ms)                              │  │
│   │        └─▶ Product Service (25ms)                       │  │
│   │            └─▶ Redis (2ms)                              │  │
│   │            └─▶ PostgreSQL (20ms)                        │  │
│   │        └─▶ Inventory Service (15ms)                     │  │
│   │  Total: 77ms                                             │  │
│   └─────────────────────────────────────────────────────────┘  │
│                                                                  │
│   4️⃣ ALERTING & AUTO-SCALING                                   │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │  🚨 PagerDuty / OpsGenie                                │  │
│   │  🤖 Kubernetes HPA / AWS Auto Scaling                   │  │
│   │                                                          │  │
│   │  Alert Rules:                                            │  │
│   │  • Error rate > 1%    → Page on-call engineer           │  │
│   │  • P99 latency > 500ms → Warning                        │  │
│   │  • CPU > 80%          → Auto-scale up                   │  │
│   │  • CPU < 20%          → Auto-scale down                 │  │
│   │  • Queue depth > 10K  → Add workers                     │  │
│   └─────────────────────────────────────────────────────────┘  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Prometheus Metrics Configuration

```java
@Configuration
public class MetricsConfig {
    
    @Bean
    public MeterRegistryCustomizer<MeterRegistry> metricsCommonTags() {
        return registry -> registry.config()
            .commonTags("application", "order-service")
            .commonTags("environment", "production");
    }
}

@RestController
public class OrderController {
    
    private final Counter orderCounter;
    private final Timer orderTimer;
    
    public OrderController(MeterRegistry registry) {
        this.orderCounter = Counter.builder("orders.placed")
            .description("Total orders placed")
            .register(registry);
        
        this.orderTimer = Timer.builder("orders.processing.time")
            .description("Time to process orders")
            .publishPercentiles(0.5, 0.95, 0.99)
            .register(registry);
    }
    
    @PostMapping("/orders")
    public Order createOrder(@RequestBody OrderRequest request) {
        return orderTimer.record(() -> {
            Order order = orderService.placeOrder(request);
            orderCounter.increment();
            return order;
        });
    }
}
```

---

## 🔄 REQUEST FLOW: END-TO-END EXAMPLE

### User Searches "iPhone 15" on Amazon

```
┌─────────────────────────────────────────────────────────────────┐
│              COMPLETE REQUEST FLOW EXAMPLE                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   1️⃣  User types amazon.com                                     │
│       └─▶ DNS (Route53) resolves to nearest edge server         │
│                                                                  │
│   2️⃣  Browser requests page                                     │
│       └─▶ CDN (CloudFront) serves images/CSS/JS instantly       │
│                                                                  │
│   3️⃣  Search query: "iPhone 15"                                 │
│       └─▶ Load Balancer routes to healthy App Server #2         │
│                                                                  │
│   4️⃣  App Server checks Read Cache                              │
│       └─▶ Cache MISS (first search for this query)              │
│                                                                  │
│   5️⃣  Query goes to Elasticsearch                               │
│       └─▶ Returns product IDs matching "iPhone 15"              │
│                                                                  │
│   6️⃣  Fetch product details from Read Replica                   │
│       └─▶ Returns price, stock, images, reviews                 │
│                                                                  │
│   7️⃣  Result cached in Redis                                    │
│       └─▶ TTL: 5 minutes (next search = instant!)               │
│                                                                  │
│   8️⃣  Response sent to user: 200ms total ✅                     │
│                                                                  │
│   ─────────────────────────────────────────────────────────────│
│                                                                  │
│   9️⃣  User clicks "Buy Now"                                     │
│       └─▶ Order sent to Kafka queue                             │
│       └─▶ Immediate response: "Order received!"                 │
│                                                                  │
│   🔟  Workers process in background:                             │
│       └─▶ Worker 1: Charge payment via Stripe                   │
│       └─▶ Worker 2: Update inventory (stock -= 1)               │
│       └─▶ Worker 3: Send confirmation email                     │
│       └─▶ Worker 4: Update analytics                            │
│                                                                  │
│   1️⃣1️⃣ Observability:                                           │
│       └─▶ Prometheus records: 200ms latency, order++            │
│       └─▶ Jaeger shows: Full request trace                      │
│       └─▶ All healthy → No alerts triggered                    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## ⚡ KEY DESIGN PRINCIPLES

```
┌─────────────────────────────────────────────────────────────────┐
│              THE 5 PILLARS OF SYSTEM DESIGN                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   PRINCIPLE          │ HOW IT'S ACHIEVED                        │
│   ───────────────────┼──────────────────────────────────────────│
│   Scalability        │ Stateless servers + Sharding + Replicas │
│   Performance        │ Caching + CDN + Async processing        │
│   Availability       │ Load balancer + DB replication + Retry  │
│   Reliability        │ Message queues + Idempotency + Alerts   │
│   Cost Efficiency    │ Auto-scaling + Hot/Cold partitioning    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 🎯 INTERVIEW CHEAT SHEET

### When the Interviewer Asks...

| Question | Start With |
|----------|------------|
| "Design Twitter" | Layer 2 (CDN for timeline), Layer 4 (Cache for feeds), Layer 5 (Async fanout) |
| "Design Uber" | Layer 2 (GeoDNS), Layer 6 (Geo-sharding), Layer 7 (Real-time metrics) |
| "Design Netflix" | Layer 2 (CDN for videos), Layer 4 (Cache for metadata), Layer 5 (Async transcoding) |
| "Design Amazon" | All 7 layers with emphasis on Layer 6 (Product catalog, Orders, Inventory shards) |

---

## ❓ Interview Practice

### Question 1:
> "Walk me through how a request flows through your system."

**Answer:**
> "Request starts at Layer 1 (Client), gets routed through Layer 2 (DNS → CDN → Load Balancer), processed by Layer 3 (Stateless App Server), accelerated by Layer 4 (Redis Cache), heavy work offloaded to Layer 5 (Kafka Queue), persisted in Layer 6 (Primary DB with replicas), and everything monitored by Layer 7 (Prometheus/Grafana). The key is stateless app servers for horizontal scaling, caching for performance, async processing for reliability, and observability for debugging."

### Question 2:
> "How would you handle a traffic spike?"

**Answer:**
> "Multiple layers handle this: Layer 2 (CDN absorbs static requests), Layer 4 (Cache absorbs repeated queries), Layer 5 (Queue buffers burst writes), Layer 7 (Auto-scaling adds app servers). The key is that spikes don't hit the database directly — they're absorbed by layers in front."

### Question 3:
> "What happens when a component fails?"

**Answer:**
> "Layer 2 (Load Balancer removes unhealthy servers), Layer 5 (Kafka retries failed messages), Layer 6 (Replicas promote to primary), Layer 7 (Alerts notify on-call). The system is designed for graceful degradation — any single component failure shouldn't bring down the system."

---

## 🔗 Connecting to Previous Days

| Day | Concept | Which Layer |
|-----|---------|-------------|
| Day 8 | Load Balancing | Layer 2 |
| Day 15 | Redis Single-Threaded | Layer 4 |
| Day 18 | Redis Timeouts | Layer 4 |
| Day 21 | Optimistic/Pessimistic Locking | Layer 6 |

---

## ✅ Day 22 Action Items

1. **Memorize the 7 layers** — Client, Edge, Application, Cache, Async, Data, Observability
2. **Practice drawing this architecture** for any system design question
3. **Know the components** in each layer and their real-world tools
4. **Understand trade-offs** — when to add layers vs keep simple

---

## 💡 Key Takeaways

| Layer | One-Liner |
|-------|-----------|
| 1. Client | Entry point — where users interact |
| 2. Edge | Traffic optimization — DNS, CDN, Load Balancer |
| 3. Application | Business logic — stateless for scale |
| 4. Cache | Speed up reads — Redis, Memcached |
| 5. Async | Background processing — Kafka, Workers |
| 6. Data | Source of truth — Primary, Replicas, Shards |
| 7. Observability | Monitor & heal — Logs, Metrics, Alerts |

---

## 🎯 The Architect's Principle

> **Junior:** "I'll just build a monolith with a database."
>
> **Architect:** "That works for 1,000 users. But when you hit 1 million, you'll need all 7 layers. Start simple, but design with these layers in mind. Know that CDN will handle static content, cache will handle repeated reads, queue will handle traffic spikes, replicas will handle read scale, shards will handle write scale, and observability will tell you when things break. The 7-layer model isn't overhead — it's the proven pattern that powers every tech giant."

---

*— Sunchit Dudeja*  
*Day 22 of 50: System Design Interview Preparation Series*

---

> 🎯 **Interview Edge:** In any system design interview, start by drawing these 7 layers. Then focus on the 2-3 layers most relevant to the problem. For Twitter, it's Cache + Async. For Uber, it's Edge + Data. Show the interviewer you understand the full picture.

> 📢 **Real Impact:** This exact architecture powers Amazon (1M orders/day), Netflix (250M subscribers), Uber (100M users), and every major tech platform. Master these 7 layers, and you've mastered 80% of system design.

---

> 💡 **Tomorrow (Day 23):** We'll explore **Consistent Hashing** — how Discord, Netflix, and Amazon distribute data across thousands of servers without rehashing everything when nodes join or leave.

