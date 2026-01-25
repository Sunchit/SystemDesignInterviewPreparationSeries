# Kubernetes Scaling: The Architect's Orchestra
### Day 11 of 50 - System Design Interview Preparation Series

**By Sunchit Dudeja**

---

## 🎯 Welcome to Day 11!

Yesterday, we mastered request coalescing to protect databases from thundering herds. Today, we scale up to **container orchestration** — where developers see "pods scaling up" and architects see a "symphony of resource optimization, cost management, and SLA guarantees."

> Scaling pods is easy. Scaling intelligently while optimizing costs across 100+ microservices? That's architecture.

---

## 🛍️ Flipkart's Diwali Sale — Two Perspectives

### Developer's Kubernetes Scaling

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: product-service
spec:
  minReplicas: 3
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

*"Scale when CPU > 70%. 3 to 10 pods. Done."*

### Architect's Reality Check

```
Normal Day:     50,000 orders/hour
Diwali Peak:    5,000,000 orders/hour (100x surge!)

Mixed Workload Challenges:
├── Product browsing   → CPU intensive
├── Image processing   → Memory intensive  
├── Cart updates       → IOPS intensive
└── Payment processing → Latency sensitive
```

| Metric | Normal | Diwali Peak | Challenge |
|--------|--------|-------------|-----------|
| Orders/hour | 50,000 | 5,000,000 | 100x surge |
| Concurrent users | 100,000 | 10,000,000 | Session management |
| Images processed | 1,000/min | 100,000/min | Memory pressure |
| Payment TPS | 500 | 50,000 | Zero-downtime required |

**The developer's simple HPA would crash within 60 seconds of sale start.**

---

## 📊 Scaling Techniques: When to Use What

Kubernetes offers multiple scaling strategies, each optimal for different scenarios:

| Technique | Best For | Primary Trigger | Example |
|-----------|----------|-----------------|---------|
| **CPU-based** | Compute-intensive workloads | Processing power limits throughput | Video encoding, API servers |
| **Memory-based** | Data processing services | Large datasets, image processing | Image processors, caches |
| **RPM-based** | Web APIs | Traffic volume is the constraint | Product catalogs, search |
| **Queue-based** | Async workloads | Backlog size indicates load | Order processing, notifications |
| **Time-based** | Known patterns | Predictable peaks | Daily traffic, holiday sales |
| **Event-driven** | Real-time triggers | Unpredictable viral moments | Social features, live events |
| **Composite** | Complex services | Multiple metrics matter | Payment gateways, core services |

---

## 🏗️ The 8-Dimensional Scaling Ecosystem

### 1. CPU-Based Scaling (The Classic)

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: product-service
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: product-service
  minReplicas: 10
  maxReplicas: 500
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 65
        # Not 70% - leave 5% buffer for spikes
```

#### Architect's Behavioral Tweaks

```yaml
behavior:
  scaleUp:
    stabilizationWindowSeconds: 60  
    # Wait 1 minute to confirm trend (avoid flapping)
    policies:
    - type: Percent
      value: 100
      periodSeconds: 30
    # Double capacity every 30 seconds during surge
    
  scaleDown:
    stabilizationWindowSeconds: 300
    # Wait 5 minutes before scaling down (avoid premature)
    policies:
    - type: Percent
      value: 10
      periodSeconds: 60
    # Remove max 10% per minute (graceful)
```

| Setting | Developer Default | Architect's Choice | Why |
|---------|-------------------|-------------------|-----|
| CPU Target | 70% | 65% | 5% buffer for spikes |
| Scale Up Window | 0s | 60s | Confirm trend, avoid flapping |
| Scale Up Rate | 4 pods/15s | 100%/30s | Aggressive for flash sales |
| Scale Down Window | 300s | 300-600s | Prevent premature scale-down |
| Scale Down Rate | 100% | 10%/min | Graceful, avoid request drops |

---

### 2. Memory-Based Scaling (The Memory Hog Handler)

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: image-processor
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: image-processor
  minReplicas: 5
  maxReplicas: 200
  metrics:
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 75
        # Memory pressure is more dangerous than CPU
```

#### Real Example — Flipkart Image Processing

```
Image Processing Pipeline:
┌─────────────────────────────────────────────────────────┐
│  Load Image → Resize → Compress → Watermark → Store    │
│                                                         │
│  Each image consumes ~500MB memory during processing   │
│  Pod limit: 4GB → Max 8 concurrent images/pod          │
└─────────────────────────────────────────────────────────┘

Normal:  100 images/min  → 2 pods  (16 concurrent)
Sale:    10,000 images/min → 200 pods (1,600 concurrent)

Memory scaling triggers at 75% (3GB used of 4GB limit)
```

**Why 75% not 80%?** Memory OOM kills are sudden and catastrophic. Leave buffer.

---

### 3. RPM-Based Scaling (Traffic-Aware)

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: product-catalog
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: product-catalog
  minReplicas: 10
  maxReplicas: 500
  metrics:
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: 500
        # Scale when avg > 500 req/sec per pod
```

#### Flipkart's Production Implementation

```yaml
# Custom metrics via Prometheus adapter
metrics:
- type: External
  external:
    metric:
      name: nginx_ingress_requests_per_second
      selector:
        matchLabels:
          service: product-catalog
    target:
      type: AverageValue
      averageValue: 1000
```

| Traffic Pattern | Requests/sec | Pods Needed | Trigger |
|-----------------|--------------|-------------|---------|
| Normal browsing | 50,000 | 50 | RPM-based |
| Flash sale start | 500,000 | 500 | RPM + Time-based |
| Celebrity tweet | 200,000 | 200 | RPM (reactive) |

---

### 4. Queue-Based Scaling (Message-Driven)

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: order-processor
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: order-processor
  minReplicas: 10
  maxReplicas: 500
  metrics:
  - type: External
    external:
      metric:
        name: sqs_approximate_number_of_messages_visible
        selector:
          matchLabels:
            queueName: order-processing-queue
      target:
        type: AverageValue
        averageValue: 100
        # Scale when >100 messages per pod in queue
```

#### Order Processing Pipeline

```
┌─────────────────────────────────────────────────────────┐
│                    Order Queue (SQS)                     │
│  ┌────┬────┬────┬────┬────┬────┬────┬────┬────┬────┐   │
│  │ O1 │ O2 │ O3 │... │... │... │... │... │... │50K │   │
│  └────┴────┴────┴────┴────┴────┴────┴────┴────┴────┘   │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│              Order Processor Pods                        │
│  Normal:  1,000 orders in queue → 10 pods               │
│  Peak:    50,000 orders in queue → 500 pods             │
│  Rate:    Each pod processes 10 orders/minute           │
└─────────────────────────────────────────────────────────┘
```

**Why queue-based?** Decouples ingestion from processing. Queue absorbs spike, scaling catches up.

---

### 5. Custom Metric Scaling (Business-Aware)

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: checkout-service
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: checkout-service
  minReplicas: 50
  maxReplicas: 300
  metrics:
  - type: External
    external:
      metric:
        name: orders_per_minute
      target:
        type: Value
        value: 1000
        # Scale when >1000 orders/minute
```

#### Flipkart's Business Metrics

```yaml
# Diwali-specific scaling rules
custom_metrics:
  - name: flash_sale_active
    type: boolean
    value: 1  # Is flash sale active?
    action: "scale to max immediately"
    
  - name: checkout_conversion_rate  
    type: percentage
    threshold: 15
    action: "scale up checkout service"
    
  - name: payment_success_rate
    type: percentage  
    threshold: 98
    action: "alert if below, scale payment pods"
    
  - name: cart_abandonment_rate
    type: percentage
    threshold: 60
    action: "scale recommendation service"
```

**Why business metrics?** CPU at 50% means nothing if orders are failing. Scale what matters to revenue.

---

### 6. Time-Based Scaling (Predictive)

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: product-catalog
spec:
  scaleTargetRef:
    name: product-catalog
  triggers:
  - type: cron
    metadata:
      timezone: Asia/Kolkata
      start: 0 8 * * *
      end: 0 23 * * *
      desiredReplicas: "50"
    # 8 AM to 11 PM: 50 pods minimum
```

#### Flipkart's Diwali Schedule

```yaml
# Pre-sale warm-up (October 10-20)
triggers:
- type: cron
  metadata:
    start: 0 7 10-20 10 *   # Oct 10-20, 7 AM
    end: 0 23 10-20 10 *    # Oct 10-20, 11 PM
    desiredReplicas: "100"

# Sale hours (October 20-22, 8 PM onwards)
- type: cron  
  metadata:
    start: 0 20 20-22 10 *  # Oct 20-22, 8 PM
    end: 0 2 21-23 10 *     # Next day 2 AM
    desiredReplicas: "1000"
    
# Post-sale maintenance (October 23-25)
- type: cron
  metadata:
    start: 0 0 23 10 *
    end: 0 0 25 10 *
    desiredReplicas: "200"
```

**Why predictive?** Don't wait for traffic to hit — be ready. 1000 pods take 5 minutes to start; that's 5 minutes of failed requests.

---

### 7. Event-Based Scaling (Reactive)

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: product-view-handler
spec:
  scaleTargetRef:
    name: product-view-handler
  triggers:
  - type: kafka
    metadata:
      topic: product-view-events
      bootstrapServers: kafka-broker:9092
      consumerGroup: product-service
      lagThreshold: "1000"
      # Scale when >1000 messages behind
```

#### Real-Time Viral Moment Handling

```
┌─────────────────────────────────────────────────────────┐
│  Viral Moment: Celebrity posts product on Instagram     │
│                                                          │
│  Normal:  1,000 events/sec  → 10 pods                   │
│  Viral:   100,000 events/sec → 1,000 pods              │
│                                                          │
│  Kafka lag triggers scaling within 30 seconds           │
└─────────────────────────────────────────────────────────┘

Event Flow:
User views product → Kafka event → Consumer pods
                                         ↓
                     Lag increases → KEDA triggers scale-up
                                         ↓
                     New pods consume backlog
```

**Why event-based?** Viral moments are unpredictable. React to actual demand signals.

---

### 8. Composite Scaling (Multi-Metric)

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: payment-gateway
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: payment-gateway
  minReplicas: 50
  maxReplicas: 200
  metrics:
  # 40% weight on CPU
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
        
  # 30% weight on Memory
  - type: Resource
    resource:
      name: memory  
      target:
        type: Utilization
        averageUtilization: 80
        
  # 30% weight on RPS
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: 500
```

**Scaling Formula:**

```
desired_replicas = max(
    replicas_for_cpu,
    replicas_for_memory,
    replicas_for_rps
)
```

**Why composite?** Payment gateway is CPU-bound during encryption, memory-bound during session handling, and RPS-bound during peak checkout. No single metric tells the full story.

---

## 📊 Architect's Scaling Matrix for Flipkart

| Service | Primary Metric | Secondary | Min | Max | Special Rules |
|---------|---------------|-----------|-----|-----|---------------|
| Product Catalog | RPM (1000/s) | CPU (70%) | 10 | 500 | Time-based pre-warm |
| Image Processor | Memory (4GB) | Queue length | 5 | 200 | GPU nodes for ML |
| Cart Service | CPU (60%) | Session count | 20 | 300 | Scale down slowly |
| Payment Gateway | Latency (100ms) | RPM (500/s) | 50 | 100 | Never scale down during tx |
| Search Service | CPU (80%) | Cache miss rate | 30 | 400 | Memory optimized nodes |
| Recommendation | Engagement rate | CPU (75%) | 15 | 150 | ML model warming |
| Order Processor | Queue depth | CPU (70%) | 10 | 500 | Aggressive scale-up |
| Notification | Queue depth | Memory (60%) | 5 | 100 | Batch processing |

---

## 🎯 Vertical vs Horizontal Scaling Decision

### When to Scale UP (VerticalPodAutoscaler)

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: redis-master
spec:
  targetRef:
    apiVersion: "apps/v1"
    kind: StatefulSet
    name: redis-master
  updatePolicy:
    updateMode: "Auto"
  resourcePolicy:
    containerPolicies:
    - containerName: redis
      minAllowed:
        memory: "4Gi"
        cpu: "2"
      maxAllowed:
        memory: "32Gi"
        cpu: "16"
```

### Decision Matrix

| Factor | Scale UP (Vertical) | Scale OUT (Horizontal) |
|--------|--------------------|-----------------------|
| Application Type | Stateful (databases) | Stateless (microservices) |
| Licensing | Per-instance costs | Per-core costs |
| Network Limits | Internal communication heavy | External traffic heavy |
| GPU/TPU | Specialized hardware | Commodity hardware |
| Cost Priority | Performance first | Cost optimization |
| Fault Tolerance | Lower (single point) | Higher (distributed) |

### Real Example — Redis Cluster

```yaml
# Vertical scaling for master (stateful, single writer)
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
spec:
  targetRef:
    kind: StatefulSet
    name: redis-master
  updatePolicy:
    updateMode: "Auto"
  
---
# Horizontal scaling for replicas (stateless reads)
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
spec:
  scaleTargetRef:
    kind: StatefulSet
    name: redis-replicas
  minReplicas: 3
  maxReplicas: 20
  metrics:
  - type: Pods
    pods:
      metric:
        name: redis_connected_clients
      target:
        type: AverageValue
        averageValue: 1000
```

---

## 🔧 Cluster Autoscaler (The Big Picture)

**Developer thinks:** "Pods scale"  
**Architect thinks:** "Nodes scale → Pods scale → Cost optimized"

```yaml
apiVersion: cluster.x-k8s.io/v1beta1
kind: MachineDeployment
metadata:
  name: flipkart-worker-pool
spec:
  replicas: 10
  template:
    spec:
      bootstrap:
        configRef:
          name: flipkart-bootstrap
      infrastructureRef:
        name: aws-machine-template
---
# Cluster Autoscaler configuration
apiVersion: v1
kind: ConfigMap
metadata:
  name: cluster-autoscaler-config
data:
  expander: priority
  # Priority order:
  # 1. Spot instances (70% cheaper)
  # 2. Reserved instances (40% cheaper)  
  # 3. On-demand instances (full price)
  
  scale-down-utilization-threshold: "0.5"
  scale-down-unneeded-time: "10m"
  skip-nodes-with-local-storage: "false"
```

### Cost Optimization Strategy

```
┌─────────────────────────────────────────────────────────┐
│              Flipkart's Node Strategy                   │
├─────────────────────────────────────────────────────────┤
│  Layer 1: Reserved Instances (50% of baseline)          │
│           → Committed 1-year, 40% savings               │
│           → Always-on core services                     │
├─────────────────────────────────────────────────────────┤
│  Layer 2: Spot Instances (daily peaks)                  │
│           → 70% cheaper, interruptible                  │
│           → Stateless services only                     │
├─────────────────────────────────────────────────────────┤
│  Layer 3: On-Demand (Diwali/emergency)                  │
│           → Full price, guaranteed                      │
│           → Payment, order processing                   │
└─────────────────────────────────────────────────────────┘

Total Savings: 65% vs all on-demand
```

---

## 🎯 Exercise: Design Scaling Strategy

### Scenario: Zomato's Saturday Night Dinner Rush

**Developer's design:** "CPU > 80% → scale to 50 pods"

**Architect's design:**

```yaml
# Service: Restaurant Search
scaling_strategy:
  primary_metric: rpm
  target: 2000/s per pod
  secondary_metric: latency_p95
  target: 200ms
  min_replicas: 20
  max_replicas: 500
  time_based:
    weekdays: 20 pods
    friday_6pm_11pm: 100 pods
    saturday_6pm_11pm: 300 pods
    valentines_day: 500 pods

---
# Service: Live Order Tracking
scaling_strategy:
  primary_metric: websocket_connections
  target: 5000 per pod
  secondary_metric: gps_updates_per_second
  target: 1000/s
  min_replicas: 30  # Real-time critical
  scale_up: aggressive (100% every 30s)
  scale_down: conservative (10% every 5min)

---
# Service: Payment Processing  
scaling_strategy:
  primary_metric: success_rate
  target: 99.5%
  secondary_metric: transaction_latency
  target: 2s
  min_replicas: 50  # High availability
  max_replicas: 200
  special_rule: disable_scale_down_7pm_to_10pm
  
---
# Cluster Strategy
node_pools:
  baseline: m5.large (reserved)
  evening_peak: c5.4xlarge (spot)
  special_events: r5.memory_optimized (for Redis)
```

---

## ❓ Interview Practice

### Question 1:
> "How would you design auto-scaling for a flash sale with 100x normal traffic?"

**Answer:**
> "I'd implement a multi-layered scaling strategy. First, time-based predictive scaling to pre-warm pods 30 minutes before the sale — you can't wait for traffic to hit when 1000 pods take 5 minutes to start. Second, RPM-based scaling for the actual traffic with aggressive scale-up (100% every 30 seconds) and conservative scale-down (10% per minute). Third, queue-based scaling for order processing to decouple ingestion from processing. Fourth, cluster autoscaler with spot instances for cost optimization, but on-demand for payment services. Finally, I'd use business metrics like orders-per-minute and payment success rate to trigger alerts and emergency scaling."

### Question 2:
> "When would you use VPA vs HPA?"

**Answer:**
> "VPA (Vertical Pod Autoscaler) is for stateful workloads where you need more resources per instance — like a Redis master that can't be horizontally sharded, or a database with per-instance licensing. HPA (Horizontal Pod Autoscaler) is for stateless microservices where you want fault tolerance and cost optimization through distribution. Often I'd use both: VPA for the primary database, HPA for read replicas. The key consideration is whether the workload can be parallelized — if yes, scale out; if no, scale up."

### Question 3:
> "How do you prevent scaling thrashing?"

**Answer:**
> "Scaling thrashing happens when pods rapidly scale up and down. I prevent it with three techniques: First, stabilization windows — wait 60 seconds before scaling up to confirm the trend, and 300+ seconds before scaling down. Second, gradual scale-down policies — remove maximum 10% of pods per minute instead of all at once. Third, use multiple metrics so a single spike doesn't trigger unnecessary scaling. Finally, for predictable patterns like daily peaks, use time-based scaling instead of reactive metrics."

---

## 🔗 Connecting to Previous Days

| Day | Concept | How It Connects |
|-----|---------|-----------------|
| Day 8 | Load Balancing | Load balancer distributes traffic across scaled pods |
| Day 9 | Bloom Filters | Reduce load so you need fewer pods |
| Day 10 | Request Coalescing | Reduce database load during scaling events |

---

## ✅ Day 11 Action Items

1. **Audit current HPA configs** — Are you using just CPU? Add memory and custom metrics
2. **Implement time-based scaling** — Identify predictable patterns in your traffic
3. **Add behavioral policies** — Configure scale-up/down windows to prevent thrashing
4. **Set up custom metrics** — Export business metrics to Prometheus for scaling
5. **Review cluster autoscaler** — Are you using spot instances for cost savings?

---

## 💡 Lessons Learned

| Lesson | Why It Matters |
|--------|----------------|
| One metric isn't enough | CPU tells nothing about business health |
| Predict, don't just react | Pre-warm before known peaks |
| Scale asymmetrically | Fast up, slow down prevents dropped requests |
| Different services need different strategies | Stateless ≠ Stateful scaling |
| Cost is a scaling dimension | Spot instances save 70% |

---

## 🚀 Key Architect Principles

| Principle | What It Means |
|-----------|---------------|
| **Scale before you need it** | Predictive and time-based scaling |
| **Different services, different strategies** | One size doesn't fit all |
| **Monitor business metrics, not just technical** | Orders/minute > CPU% |
| **Cost optimization is scaling too** | Spot instances, right-sizing |
| **Graceful degradation** | Scale down slowly, never during critical ops |

---

## 💡 Key Takeaway

> **Developer: "Just set HPA to CPU 70% and maxReplicas 100."**  
> **Architect: "I'd combine time-based predictive scaling for known patterns, RPM-based for traffic spikes, queue-based for async processing, and custom business metrics for revenue-critical paths, with cost optimization via spot instances and reserved capacity."**

The difference? Understanding that **scaling is multi-dimensional** — performance, cost, reliability, and business impact must all be balanced across dozens of microservices.

---

*— Sunchit Dudeja*  
*Day 11 of 50: System Design Interview Preparation Series*

---

> 🎯 **Interview Edge:** When asked about scaling, never give a single-metric answer. Say: "I'd design a composite scaling strategy combining CPU, memory, RPM, and business metrics like orders-per-minute, with time-based pre-warming for predictable peaks and queue-based scaling for async workloads." That's architect thinking.

> 📢 **Real Impact:** Flipkart's multi-dimensional scaling strategy handles 100x traffic surges during Diwali sales while maintaining sub-200ms response times and saving 65% on infrastructure costs through intelligent spot instance usage.

