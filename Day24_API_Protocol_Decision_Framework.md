# Day 24: The Architect's Guide to API Protocols

## 🎯 When to Use gRPC, GraphQL, REST, and WebSocket

---

## Today's Learning Objective

One of the most critical decisions in system design is choosing the right communication protocol. Today, we'll master the **Architect's Decision Framework** for selecting between gRPC, GraphQL, REST, and WebSocket.

> **Key Insight:** Modern systems don't use a single protocol—they use a **protocol ecosystem** where each client gets the optimal interface.

---

## 📊 The Architect's Mental Model

```
┌─────────────────────────────────────────────────────────────┐
│                    API Protocol Selection                    │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│   SPEED                              COMPATIBILITY           │
│     ▲                                      ▲                 │
│     │  gRPC ●                              │                 │
│     │       ● GraphQL                      │  ● REST         │
│     │            ● REST+Cache              │  ● GraphQL      │
│     │                 ● WebSocket          │                 │
│     │                      ● REST          │  ● gRPC         │
│     └──────────────────────────────────────┴─────────────►   │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 🚀 Protocol 1: gRPC — The Internal High-Performance Choice

### What is gRPC?
gRPC is a high-performance RPC (Remote Procedure Call) framework that uses **Protocol Buffers** for serialization and **HTTP/2** for transport.

### When to Use gRPC

```yaml
✅ PERFECT FOR:
  - Microservices communication
  - Polyglot environments (different languages)
  - Low-latency requirements (<10ms)
  - Streaming data (real-time feeds)
  - Mobile backends (saves battery)
  - When you control both client and server

❌ AVOID WHEN:
  - Building public APIs (browser support limited)
  - Human-readable debugging needed
  - Simple CRUD with occasional use
  - Teams unfamiliar with Protocol Buffers
```

### Real-World Example: Uber's Ride Matching

```protobuf
// Uber's internal ride-matching service
service RideMatching {
    rpc FindDrivers (LocationRequest) returns (stream Driver) {}
    rpc RequestRide (RideRequest) returns (RideConfirmation) {}
}

// Why gRPC?
// - 500 microservices communicating
// - 10 different programming languages
// - 100ms SLA requirement
// - Result: 5-10ms latency consistently
```

### Key Performance Stats

| Metric | gRPC | REST (JSON) |
|--------|------|-------------|
| Serialization | Binary (Protobuf) | Text (JSON) |
| Payload Size | 10x smaller | Baseline |
| Latency | ~5-10ms | ~50-100ms |
| Streaming | Native support | Polling/SSE |

### Companies Using gRPC
- **Google** (internal services)
- **Netflix** (microservices)
- **Uber** (ride matching)
- **Dropbox** (file sync)
- **Square** (payments)

---

## 🔌 Protocol 2: WebSocket — The Real-Time Communication King

### What is WebSocket?
WebSocket provides **full-duplex communication** over a single TCP connection, allowing both client and server to send messages at any time.

### When to Use WebSocket

```yaml
✅ PERFECT FOR:
  - Live chat applications
  - Collaborative editing (Google Docs style)
  - Live sports scores / stock tickers
  - Multiplayer gaming
  - Real-time dashboards
  - Push notifications (when polling is inefficient)

❌ AVOID WHEN:
  - Request-response is sufficient
  - Server load is unpredictable (connections are stateful)
  - Stateless architecture is required
  - Occasional updates only (use polling instead)
```

### Real-World Example: Trading Platform

```javascript
// Trading Platform WebSocket Topics
const topics = {
  price:    '/price/RELIANCE',      // Real-time stock updates (50ms)
  orders:   '/order/user/123',      // User's order status
  news:     '/market/news',         // Breaking news alerts
  alerts:   '/alerts/portfolio'     // Price drop notifications
};

// Why WebSocket?
// - 50ms price update requirement
// - 100K concurrent users
// - Result: 10x less overhead than HTTP polling
```

### Polling vs WebSocket Comparison

```
HTTP Polling (every 1 second):
┌────────┐                              ┌────────┐
│ Client │ ──► "Any updates?" ─────────►│ Server │
│        │ ◄── "No" ◄──────────────────│        │
│        │ ──► "Any updates?" ─────────►│        │
│        │ ◄── "No" ◄──────────────────│        │
│        │ ──► "Any updates?" ─────────►│        │
│        │ ◄── "Yes, here's data" ◄────│        │
└────────┘                              └────────┘
(3 requests for 1 update = wasteful)

WebSocket (persistent connection):
┌────────┐                              ┌────────┐
│ Client │ ◄═══════════════════════════►│ Server │
│        │     (persistent connection)  │        │
│        │ ◄── "Here's an update" ◄────│        │
└────────┘                              └────────┘
(1 message for 1 update = efficient)
```

### Companies Using WebSocket
- **Slack** (real-time messaging)
- **Discord** (voice & chat)
- **Figma** (collaborative design)
- **Binance** (trading)
- **Notion** (collaborative docs)

---

## 📦 Protocol 3: GraphQL — The Frontend Freedom Fighter

### What is GraphQL?
GraphQL is a **query language for APIs** that allows clients to request exactly the data they need—no more, no less.

### When to Use GraphQL

```yaml
✅ PERFECT FOR:
  - Mobile apps (slow networks, limited data)
  - Complex UIs with multiple data sources
  - Rapidly evolving frontend requirements
  - Aggregating multiple backend services
  - Reducing over-fetching / under-fetching
  - When frontend teams move faster than backend

❌ AVOID WHEN:
  - Simple CRUD APIs (overkill)
  - Performance is absolute top priority (extra parsing)
  - Complex caching requirements (cache invalidation harder)
  - File uploads (not designed for it)
  - Security concerns with query complexity
```

### Real-World Example: GitHub Mobile App

```graphql
# GitHub's GraphQL API - Mobile app request
query GetUserDashboard {
  viewer {
    login
    avatarUrl
    repositories(first: 10, orderBy: {field: UPDATED_AT, direction: DESC}) {
      nodes {
        name
        description
        stargazerCount
        issues(states: OPEN) { 
          totalCount 
        }
        pullRequests(states: OPEN) { 
          totalCount 
        }
      }
    }
    notifications(first: 5) {
      nodes {
        title
        reason
        updatedAt
      }
    }
  }
}

# Result: ONE request replaces 4-5 REST calls
# Perfect for mobile: saves battery + bandwidth
```

### REST vs GraphQL Comparison

```
REST Approach (4 requests):
GET /user/profile          → User data
GET /user/repos?limit=10   → Repository list
GET /repos/{id}/issues     → Issue counts (x10)
GET /user/notifications    → Notifications
────────────────────────────────────────────
Total: 4+ round trips, lots of extra data

GraphQL Approach (1 request):
POST /graphql              → All data in one response
────────────────────────────────────────────
Total: 1 round trip, exact data needed
```

### Companies Using GraphQL
- **GitHub** (public API)
- **Shopify** (storefront API)
- **Facebook** (internal + public)
- **Airbnb** (mobile apps)
- **Twitter** (mobile apps)

---

## 📋 Protocol 4: REST with Cache — The Public API Workhorse

### What is REST + Cache?
REST (Representational State Transfer) with proper caching headers (ETags, Cache-Control) enables efficient content delivery with minimal bandwidth.

### When to Use REST + Cache

```yaml
✅ PERFECT FOR:
  - Public APIs (universal compatibility)
  - Read-heavy workloads (90% reads, 10% writes)
  - Content delivery (blogs, documentation, catalogs)
  - When cacheability is valuable
  - Simple resource-based models
  - Third-party developer adoption

❌ AVOID WHEN:
  - Real-time updates needed (polling overhead)
  - Complex nested data relationships
  - Highly dynamic, personalized data (cache ineffective)
  - Write-heavy workloads
```

### Real-World Example: Stripe's Product API

```http
# First request - full response
GET /v1/products/prod_123
Response: 200 OK
ETag: "33a64df551425fcc55e4d42a148795d9f25f89d4"
Cache-Control: max-age=300
{
  "id": "prod_123",
  "name": "Premium Plan",
  "price": 9900
}

# Subsequent request - cached
GET /v1/products/prod_123
If-None-Match: "33a64df551425fcc55e4d42a148795d9f25f89d4"

Response: 304 Not Modified
(Zero bandwidth used!)

# Stripe handles 50M+ requests/day with 70% cache hits
```

### Caching Strategy

```
┌─────────────────────────────────────────────────────────────┐
│                    Caching Layers                            │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│   Client ──► Browser Cache ──► CDN ──► API Gateway ──► Server│
│              (seconds)        (minutes)   (varies)           │
│                                                              │
│   Headers:                                                   │
│   • Cache-Control: public, max-age=3600                     │
│   • ETag: "abc123" (for validation)                         │
│   • 304 Not Modified (bandwidth saver)                      │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Companies Using REST + Cache
- **Stripe** (payments API)
- **Twilio** (communications API)
- **AWS** (all public APIs)
- **GitHub** (REST API v3)
- Every major public API

---

## 🐢 Protocol 5: REST (Uncached) — The Simple Starter

### When to Use Plain REST

```yaml
✅ PERFECT FOR:
  - Prototypes and MVPs
  - Internal admin panels
  - Write-heavy operations (creates, updates, deletes)
  - Highly sensitive data (cache inappropriate)
  - Simple CRUD with low traffic
  - When simplicity > performance

❌ AVOID WHEN:
  - Scaling to millions of users
  - Mobile apps (battery/bandwidth concerns)
  - Any serious performance requirement
```

---

## 🎯 The Decision Matrix

| Factor | gRPC | WebSocket | GraphQL | REST+Cache | REST |
|--------|:----:|:---------:|:-------:|:----------:|:----:|
| **Internal Microservices** | ⭐⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐ | ⭐⭐ | ⭐ |
| **Public API** | ⭐ | ⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ |
| **Mobile App Backend** | ⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐ |
| **Real-time Updates** | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐ | ⭐ | ⭐ |
| **Low Latency Required** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐ | ⭐ |
| **Development Speed** | ⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| **Browser Support** | ⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| **Caching Ease** | ⭐ | ⭐ | ⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ |
| **Bandwidth Efficiency** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐ |

---

## 🏗️ Real-World Architecture: E-Commerce at Scale

Here's how a large e-commerce platform (Amazon-scale) would use ALL protocols:

```yaml
E-Commerce Protocol Strategy:

├── 🔧 Internal Services → gRPC
│   ├── Inventory Service ↔ Product Service
│   ├── Order Service ↔ Payment Service
│   ├── Recommendation Engine ↔ User Service
│   └── Search Service ↔ Catalog Service
│   │
│   └── Why? 500+ microservices, strict SLAs, 10+ languages

├── 🌐 Public API → REST + Cache
│   ├── GET /products/{id}     (Cache: 5 min)
│   ├── GET /categories        (Cache: 1 hour)
│   ├── GET /search?q=...      (Cache: 1 min, vary by query)
│   └── POST /orders           (No cache)
│   │
│   └── Why? Millions of developers, universal compatibility

├── 📱 Mobile App → GraphQL
│   ├── Product details + reviews + recommendations
│   ├── Order history + tracking + returns
│   ├── User profile + preferences + notifications
│   └── Home feed (personalized, complex)
│   │
│   └── Why? 2G/3G networks in emerging markets, battery life

├── ⚡ Real-time Features → WebSocket
│   ├── Order tracking updates ("Your order is out for delivery")
│   ├── Price drop alerts ("Item in cart is now 20% off")
│   ├── Live customer support chat
│   ├── Flash sale countdowns
│   └── Inventory warnings ("Only 3 left!")
│   │
│   └── Why? Push notifications, 10x more efficient than polling

└── 🔐 Admin Panel → REST (Uncached)
    ├── Inventory management
    ├── Order fulfillment
    └── User management
    │
    └── Why? Internal use, low traffic, simplicity wins
```

---

## 💡 The Four Golden Rules

### Rule #1: Start with the Client

> "Internal services get gRPC, public APIs get REST, mobile gets GraphQL, real-time gets WebSocket."

### Rule #2: Understand the Trade-offs

```
Performance ◄─────────────────────────► Compatibility
    gRPC                                    REST

Bandwidth ◄───────────────────────────► Simplicity  
    GraphQL                                 REST

Real-time ◄───────────────────────────► Stateless
    WebSocket                               REST
```

### Rule #3: Mix and Match

Modern architectures use MULTIPLE protocols simultaneously:

| Layer | Protocol | Purpose |
|-------|----------|---------|
| Service Mesh | gRPC | Internal communication (Istio/Linkerd) |
| API Gateway | REST | Public-facing endpoints |
| BFF (Backend for Frontend) | GraphQL | Mobile/web optimization |
| Real-time | WebSocket | Live updates & notifications |

### Rule #4: Plan for Evolution

```
Phase 1: REST API
         └── Launch quickly, validate idea
              ↓
Phase 2: Add GraphQL for mobile
         └── Optimize for mobile users (6 months)
              ↓
Phase 3: Migrate internal to gRPC
         └── Performance at scale (12 months)
              ↓
Phase 4: Add WebSocket for real-time
         └── Competitive features (18 months)
```

---

## 🎓 Interview Answer Template

When asked **"Which protocol would you choose?"** in a system design interview:

> "I'd use a **multi-protocol strategy** based on the client:
>
> 1. **Internal microservices** would communicate via **gRPC** for performance and strong typing
>
> 2. Our **public API** would be **REST with aggressive caching** for maximum adoption and CDN compatibility
>
> 3. **Mobile clients** would get a **GraphQL** layer to minimize bandwidth and reduce round trips
>
> 4. **Real-time features** like notifications would use **WebSocket** for efficient push updates
>
> This gives us the best of all worlds—**performance** where needed, **compatibility** where required, and **efficiency** where bandwidth matters."

---

## 🔄 Decision Flowchart

```
                        ┌──────────────────┐
                        │  New API Design  │
                        └────────┬─────────┘
                                 ▼
                     ┌───────────────────────┐
                     │ Who are the clients?  │
                     └───────────┬───────────┘
                                 │
         ┌───────────┬───────────┼───────────┬───────────┐
         ▼           ▼           ▼           ▼           ▼
    ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐
    │Internal │ │ Mobile  │ │ Public  │ │Real-time│ │  Admin  │
    │Services │ │  Apps   │ │   API   │ │ Features│ │  Panel  │
    └────┬────┘ └────┬────┘ └────┬────┘ └────┬────┘ └────┬────┘
         │           │           │           │           │
         ▼           ▼           ▼           ▼           ▼
    ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐
    │  gRPC   │ │ GraphQL │ │REST+Cache│ │WebSocket│ │  REST   │
    │         │ │         │ │         │ │         │ │         │
    │ Binary  │ │ Flexible│ │ Cached  │ │Bi-direct│ │ Simple  │
    │ Fast    │ │ Efficient│ │Universal│ │  Push   │ │  CRUD   │
    └─────────┘ └─────────┘ └─────────┘ └─────────┘ └─────────┘
```

---

## 📚 Key Takeaways

| # | Takeaway |
|---|----------|
| 1 | **No single protocol fits all use cases** — use the right tool for each job |
| 2 | **gRPC** = Internal services, low latency, streaming |
| 3 | **WebSocket** = Real-time, bi-directional, persistent connections |
| 4 | **GraphQL** = Mobile apps, complex UIs, bandwidth-constrained clients |
| 5 | **REST + Cache** = Public APIs, read-heavy, maximum compatibility |
| 6 | **REST** = Prototypes, admin panels, simplicity |
| 7 | **Modern architectures mix protocols** — Netflix, Uber, Shopify all use multiple |

---

## 🔗 Related Topics

- **Day 23:** [Database Selection for System Design](Day23_Database_Selection_System_Design.md)
- **Day 22:** [The 7 Layers of Every High-Level Design](Day22_7_Layers_HLD_Architecture.md)
- **Day 8:** [Load Balancing: Traffic Orchestration](Day8_Load_Balancing.md)

---

_Part of the System Design Interview Preparation Series by Sunchit Dudeja_

**Star ⭐ the repo if you found this helpful!**
