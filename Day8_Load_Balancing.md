# Load Balancing: From Round Robin to Intelligent Traffic Orchestration
### Day 8 of 50 - System Design Interview Preparation Series

**By Sunchit Dudeja**

---

## 🎯 Welcome to Day 8!

Yesterday, we explored database selection like an architect. Today, we dive into **load balancing** — the art of distributing traffic intelligently across servers.

> Load balancing isn't just distributing traffic. It's about predicting patterns, planning for failure, and designing graceful degradation.

---

## 🛫 The Airport Ticket Counter Analogy

Before diving into algorithms, imagine an airport with multiple ticket counters:

| Algorithm | Airport Behavior |
|-----------|------------------|
| **Round Robin** | Sends each passenger to the next counter in sequence |
| **Weighted Round Robin** | Gives more passengers to faster agents |
| **Least Connections** | Directs passengers to the counter with the shortest line |
| **Least Response Time** | Chooses the counter where agents process tickets fastest |
| **IP Hash** | Always sends the same passenger to the same counter (VIP service) |
| **Geographic** | Sends passengers to counters based on their flight destination zones |
| **Least Bandwidth** | Picks counters that aren't handling complex baggage issues |
| **Least ETA** | Predicts which counter will complete check-in fastest, considering line length, agent speed, and baggage complexity |

Each strategy optimizes for different priorities — **speed, fairness, or resource efficiency**.

---

## 📊 8 Core Load Balancing Algorithms

### 1. ROUND ROBIN (Default)

**How it works:** Distributes requests sequentially across servers

```nginx
Server List: A → B → C → A → B → C → ...
```

| Attribute | Details |
|-----------|---------|
| **Use Case** | Stateless web servers, simple APIs |
| **Real Example** | Serving static blog pages |
| **Pros** | Simple, predictable |
| **Cons** | Doesn't consider server load or capacity |
| **Flipkart Usage** | ❌ Never used for critical paths |

---

### 2. WEIGHTED ROUND ROBIN

**How it works:** Assign weights based on server capacity

```
Server A (weight 3): 40% CPU → Gets 3 requests
Server B (weight 1): Old hardware → Gets 1 request  
Server C (weight 2): 60% CPU → Gets 2 requests
```

| Attribute | Details |
|-----------|---------|
| **Use Case** | Mixed hardware environments |
| **Real Example** | Gradual server upgrades |
| **Flipkart Usage** | ✅ During server migration phases |

---

### 3. LEAST CONNECTIONS

**How it works:** Routes to server with fewest active connections

```
Server A: 150 active connections
Server B: 50 active connections ← CHOOSE THIS
Server C: 200 active connections
```

| Attribute | Details |
|-----------|---------|
| **Use Case** | Long-lived connections (WebSockets, database pools) |
| **Real Example** | WhatsApp Web, Trading platforms |
| **Flipkart Usage** | ✅ Cart & checkout (sticky sessions) |

---

### 4. LEAST RESPONSE TIME

**How it works:** Routes to server with fastest response time

```
Server A: 45ms avg response
Server B: 120ms avg response
Server C: 23ms avg response ← CHOOSE THIS
```

| Attribute | Details |
|-----------|---------|
| **Use Case** | Latency-sensitive applications |
| **Real Example** | Online gaming, real-time bidding |
| **Flipkart Usage** | ✅ Product search API |

---

### 5. IP HASH (Sticky Sessions)

**How it works:** Hash client IP → Always route to same server

```
User 192.168.1.10 → hash() % 3 = Server B (always)
User 192.168.1.11 → hash() % 3 = Server C (always)
```

| Attribute | Details |
|-----------|---------|
| **Use Case** | Session-based applications |
| **Real Example** | Shopping carts, multiplayer games |
| **Flipkart Usage** | ✅ User sessions, cart data |

---

### 6. GEOGRAPHIC / GEOLOCATION

**How it works:** Route based on user location

```
User in Mumbai → Mumbai data center
User in Delhi → Delhi data center  
User in USA → Virginia data center
```

| Attribute | Details |
|-----------|---------|
| **Use Case** | Global applications |
| **Real Example** | Netflix, Amazon Prime Video |
| **Flipkart Usage** | ✅ Primary routing strategy |

---

### 7. LEAST BANDWIDTH

**How it works:** Routes to server using least bandwidth

```
Server A: 850 Mbps used
Server B: 320 Mbps used ← CHOOSE THIS  
Server C: 1.2 Gbps used
```

| Attribute | Details |
|-----------|---------|
| **Use Case** | Media streaming, large file transfers |
| **Real Example** | YouTube, Spotify |
| **Flipkart Usage** | ✅ Product image/video serving |

---

### 8. LEAST ETA (Uber's Secret Sauce)

**How it works:** Predictive algorithm estimating total processing time

```
ETA = Current Load × Processing Power × Distance Factor

Server A: ETA 120ms
Server B: ETA 85ms ← CHOOSE THIS
Server C: ETA 210ms
```

| Attribute | Details |
|-----------|---------|
| **Use Case** | Real-time matching systems |
| **Real Example** | Uber, Ola, Swiggy delivery matching |
| **Flipkart Usage** | ❌ Not applicable |

---

## 🏆 Top 4 Load Balancing Strategies

### 1. LEAST CONNECTIONS

**Airport Example:** Sends passengers to the counter with the shortest queue

```
Counter A: 8 people waiting
Counter B: 3 people waiting ← SEND HERE
Counter C: 12 people waiting
```

| Attribute | Details |
|-----------|---------|
| **Best For** | Long transactions, WebSocket apps, database connections |
| **Used By** | Banking apps, real-time chat, trading platforms |
| **Why It Wins** | Prevents overloading any single server |

---

### 2. GEOGRAPHIC / GEOLOCATION ROUTING

**Airport Example:** Sends international passengers to international counters, domestic to domestic

```
Passenger to Dubai → International counters
Passenger to Mumbai → Domestic counters  
```

| Attribute | Details |
|-----------|---------|
| **Best For** | Global applications, low latency requirements |
| **Used By** | Netflix, Amazon, Spotify |
| **Why It Wins** | Minimizes network latency by keeping users close to servers |

---

### 3. LEAST RESPONSE TIME

**Airport Example:** Sends passengers to the counter where agents process fastest

```
Counter A: 2 minutes per passenger
Counter B: 45 seconds per passenger ← SEND HERE
Counter C: 3 minutes per passenger
```

| Attribute | Details |
|-----------|---------|
| **Best For** | API calls, search queries, latency-sensitive apps |
| **Used By** | Google Search, e-commerce sites |
| **Why It Wins** | Provides fastest user experience |

---

### 4. IP HASH (Sticky Sessions)

**Airport Example:** Same passenger always goes to same counter agent

```
Mr. Sharma → Always Counter 3, Agent Priya
Ms. Patel → Always Counter 5, Agent Raj
```

| Attribute | Details |
|-----------|---------|
| **Best For** | Shopping carts, user sessions, multi-step processes |
| **Used By** | All e-commerce (Flipkart, Amazon), banking portals |
| **Why It Wins** | Maintains session state without complex synchronization |

---

## 🔧 Real-World: Flipkart Big Billion Day Architecture

### Technical Architecture

```
User → CloudFront (Edge Cache) → AWS WAF (DDoS Protection) 
→ Route53 (DNS Load Balancing) → ALB (Application Load Balancer)
→ Auto-scaling Group (EC2 Instances) → Istio (Service Mesh)
→ Individual Microservices → Databases
```

### Key Load Balancing Decisions

| Decision | Strategy | Why |
|----------|----------|-----|
| Initial routing | Least Outstanding Requests | Not round robin — respects server load |
| Session stickiness | IP Hash | Cart and checkout → same server |
| Health checks | Every 10 seconds | 2 fails → remove server |
| Circuit breakers | 1s threshold | If service > 1s response → fail fast |

---

## 🏗️ Load Balancing Tiers

### TIER 1: DNS Load Balancing

```dns
; Multiple IPs for same domain
www.flipkart.com.  IN  A  203.0.113.1
www.flipkart.com.  IN  A  203.0.113.2
www.flipkart.com.  IN  A  203.0.113.3
```

| Attribute | Details |
|-----------|---------|
| **Strategy** | Round Robin at DNS level |
| **TTL** | 300 seconds (5 minutes) |
| **Use** | Geographic distribution |

---

### TIER 2: Global Server Load Balancing (GSLB)

```
User → Anycast IP → Nearest PoP → Health check → Best DC
```

**Strategies:**
- Geographic proximity
- Data center health
- Cost optimization

**Use:** Cross-data center failover

---

### TIER 3: Application Load Balancer

```nginx
location /api/orders {
    # Order processing cluster
    proxy_pass http://order_servers;
    least_conn;  # Strategy here
}

location /api/search {
    # Search cluster  
    proxy_pass http://search_servers;
    least_response_time;  # Different strategy
}
```

**Strategies:** Path-based routing with different algorithms

---

### TIER 4: Service Mesh (Istio/Linkerd)

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
spec:
  trafficPolicy:
    loadBalancer:
      simple: LEAST_CONN  # Microservice-level strategy
```

**Strategies:** Service-to-service intelligent routing

---

## ⚡ Advanced Strategies

### 1. ADAPTIVE LOAD BALANCING

**How it works:** Machine learning predicts optimal server

```
Monitor: Time of day, request type, user behavior
Predict: Which server will handle this request fastest
Learn: Continuously improve predictions
```

**Used by:** Google, Facebook for search

---

### 2. POWER OF TWO CHOICES

**How it works:** Randomly pick 2 servers → Choose less loaded

```
Step 1: Randomly select Server A and Server B
Step 2: Check current load of both
Step 3: Route to less loaded server
Step 4: This prevents herd behavior
```

**Used by:** High-scale systems (AWS ELB)

---

### 3. CONSISTENT HASHING

**How it works:** Hash request key → Map to server ring

```
Servers on a "hash ring"
Request hashed → Find nearest server
If server added/removed → Minimal reshuffling
```

**Use:** Distributed caching (Redis clusters)

---

### 4. FAILOVER STRATEGY

**How it works:** Primary → Backup servers

```
Primary active, Backup idle
If primary fails → Automatic switch to backup
Health checks every 5 seconds
```

**Use:** Banking, payment gateways

---

## 🎯 Strategy Selection Matrix

| Requirement | Best Strategy | Why | Example |
|-------------|---------------|-----|---------|
| Equal server capacity | Round Robin | Simple, fair | Static websites |
| Different server specs | Weighted Round Robin | Respect capacity | Migration phases |
| Long connections | Least Connections | Prevent overload | WebSocket apps |
| Low latency needed | Least Response Time | Fastest server | Gaming, trading |
| Session persistence | IP Hash | User sticks to server | Shopping carts |
| Global users | Geographic | Reduce latency | Netflix, Amazon |
| Media streaming | Least Bandwidth | Prevent congestion | YouTube, Spotify |
| Real-time matching | Least ETA | Optimal processing | Uber, Swiggy |
| High availability | Failover | Backup ready | Banking apps |
| Dynamic scaling | Adaptive | ML optimized | Google Search |

---

## 🔧 Implementation Examples

### NGINX Config

```nginx
http {
    upstream backend {
        # Strategy defined here
        least_conn;  # Change to: round_robin, ip_hash, etc.
        
        server backend1.example.com weight=3;
        server backend2.example.com;
        server backend3.example.com max_fails=3 fail_timeout=30s;
    }
}
```

### AWS Application Load Balancer

```json
{
  "Type": "AWS::ElasticLoadBalancingV2::LoadBalancer",
  "Properties": {
    "LoadBalancerAttributes": [
      {
        "Key": "routing.http.load_balancing.algorithm.type",
        "Value": "least_outstanding_requests"
      }
    ]
  }
}
```

### Kubernetes Service

```yaml
apiVersion: v1
kind: Service
spec:
  selector:
    app: my-app
  ports:
    - port: 80
  sessionAffinity: ClientIP  # Sticky sessions
  # Or: None for round robin
```

---

## 📚 What Flipkart Did RIGHT

| Strategy | Implementation |
|----------|----------------|
| **Tested at scale** | Simulated 15M users before event |
| **Multiple fallbacks** | Every service had degraded mode |
| **Real-time monitoring** | 150+ metrics watched by AI |
| **Human oversight** | Architects in war room with "kill switch" |

---

## ⚠️ What Could Go WRONG with Simple Load Balancing

| Problem | Description |
|---------|-------------|
| **Thundering herd** | All users hitting same few products |
| **Cascading failure** | One service down takes all others |
| **Region isolation** | Delhi outage shouldn't affect Mumbai |
| **Resource exhaustion** | Database connections, file descriptors |

---

## ⚠️ Common Pitfalls

| Pitfall | Problem |
|---------|---------|
| Round Robin with unequal requests | Heavy API calls vs light static files |
| IP Hash with mobile users | IP changes break sessions |
| Least Connections with quick requests | Doesn't account for CPU load |
| No health checks | Routing to dead servers |
| DNS caching | Users stuck on failed IPs (TTL too high) |

---

## 🎯 Real-World Decision Flow

```
START: New application
├── Is it global? → Yes → Geographic + GSLB
├── Need sessions? → Yes → IP Hash + Least Connections  
├── Real-time? → Yes → Least ETA + Adaptive
├── Media heavy? → Yes → Least Bandwidth
├── Mixed hardware? → Yes → Weighted Round Robin
└── Simple API? → Yes → Round Robin

THEN: Monitor and adjust
├── High latency? → Switch to Least Response Time
├── Uneven load? → Switch to Least Connections
├── Server failures? → Add health checks + failover
└── Scaling issues? → Add Adaptive learning
```

---

## 🎯 Quick Decision Guide

| Scenario | Choose | When |
|----------|--------|------|
| **Least Connections** | Requests take varying time, long-lived connections | Uber driver matching, WhatsApp |
| **Geographic** | Users are globally distributed, low latency critical | Video streaming, gaming |
| **Least Response Time** | Every millisecond matters, varying server performance | Search engines, stock trading |
| **IP Hash** | Need session persistence, user data cached locally | Shopping carts, online exams |

---

## 🧪 Exercise: UPSC Online Exam Load Balancing

**Scenario:** Design load balancing for UPSC online exam (1M students starting test at exactly 9:00 AM)

**Questions to answer:**

1. **What load balancing algorithms would you use and why?**
2. **How would you handle the 9:00 AM spike?**
3. **What metrics would you monitor?**
4. **What failure scenarios would you plan for?**

**Hint:** Exams can't have queues. All 1M must start exactly at 9:00 AM. Different constraints than e-commerce!

---

## ❓ Interview Practice

### Question 1:
> "How would you design load balancing for a flash sale like Big Billion Day?"

**Answer:**
> "I'd use a multi-tier approach. First, Geographic routing via DNS to direct users to nearest data center. Then, at the application layer, I'd use Least Outstanding Requests instead of Round Robin because requests vary in processing time. For cart and checkout, I'd add IP Hash for session stickiness. Critical systems like payments would have dedicated Failover configurations. I'd also implement circuit breakers with 1-second thresholds to fail fast and prevent cascading failures. Pre-scale infrastructure based on capacity estimates, and have real-time monitoring to catch issues early."

### Question 2:
> "Why not just use Round Robin everywhere?"

**Answer:**
> "Round Robin assumes all requests are equal and all servers have equal capacity — both are rarely true. A product search request might take 200ms while a static image request takes 5ms. If you round-robin both to the same server pool, the server handling searches becomes overloaded while others sit idle. Least Connections or Least Response Time adapts to actual server load. For sessions, Round Robin breaks shopping carts because consecutive requests go to different servers. The right strategy depends on your traffic patterns."

---

## 🔗 Connecting to Previous Days

| Day | Concept | How It Connects |
|-----|---------|-----------------|
| Day 2 | SCALED | Load balancing directly impacts Scalability and Availability |
| Day 5 | Capacity | Capacity estimation informs how many servers to load balance |
| Day 6 | Failure | Load balancers enable failover and graceful degradation |
| Day 7 | Databases | Database connection pools need load balancing too |

---

## ✅ Day 8 Action Items

1. **Identify your current strategy** — What load balancing does your application use today?
2. **Analyze traffic patterns** — Measure request duration variance, session requirements
3. **Implement health checks** — Ensure load balancers detect and remove failed servers
4. **Plan for spikes** — How would your system handle 10x traffic?
5. **Learn NGINX/HAProxy** — Configure different algorithms hands-on

---

## 🚀 Key Architect Principles

| Principle | What It Means |
|-----------|---------------|
| **Match strategy to traffic** | Different endpoints need different algorithms |
| **Layer your load balancing** | DNS → GSLB → ALB → Service Mesh |
| **Health checks are essential** | Never route to dead servers |
| **Plan for the spike** | Pre-scale, don't react-scale |
| **Fail fast** | Circuit breakers prevent cascading failures |

---

## 💡 Key Takeaway

> **Developer: "Just add more servers and round-robin them."**  
> **Architect: "What's the request duration distribution? Do we need session stickiness? What's our failover strategy? How do we prevent thundering herd during flash sales?"**

Load balancing isn't just distributing traffic — it's **orchestrating chaos**. Flipkart didn't just "add more servers" for Big Billion Day. They designed an intelligent system that could predict patterns, handle failures gracefully, and degrade service when needed.

That's the difference between a developer and an architect.

---

*— Sunchit Dudeja*  
*Day 8 of 50: System Design Interview Preparation Series*

---

> 🎯 **Interview Edge:** When discussing load balancing, don't just name an algorithm. Say: "I'd use a multi-tier approach — Geographic routing at DNS level, Least Connections at the application layer, IP Hash for session-critical paths, and Failover for payment services." Then explain WHY each choice. This shows you understand that different traffic patterns need different strategies.

> 💡 **Pro Tip:** Start with **Least Connections** for most web applications — it's simple but effective. As you scale, add Geographic routing for global users, and Adaptive algorithms for peak efficiency. Modern applications use **multiple strategies together**, not just one.

