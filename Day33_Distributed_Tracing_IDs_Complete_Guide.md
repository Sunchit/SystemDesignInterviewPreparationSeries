# Distributed Tracing IDs: The Complete Guide
### Day 33 of 50 - System Design Interview Preparation Series

**By Sunchit Dudeja**

---

## 🎯 Welcome to Day 33!

Yesterday, we mastered Scheduled Locks for cron jobs. Today, we tackle **distributed tracing IDs**—the identifiers that let you follow a single request across dozens of microservices and pinpoint exactly where it failed or slowed down.

> **The Problem:** A user request flows through API Gateway → Order Service → Payment Service → Inventory Service → Shipping. When something breaks, which service? Which operation? How long did each step take?

> **The Solution:** TraceId + SpanId. One global ID follows the request everywhere. Local IDs track each operation. Together, they reconstruct the entire journey.

> **📐 Excalidraw Diagram:** Open [distributed-tracing-ids.excalidraw](./distributed-tracing-ids.excalidraw) at [excalidraw.com](https://excalidraw.com) (File → Open) — black background.

---

## 📋 What Are These IDs?

When a single user request flows through multiple microservices, you need identifiers to track it across the entire system:

| ID Type | Purpose | Scope | Example |
|---------|---------|-------|---------|
| **TraceId** / RequestId / CorrelationId | Tracks the entire request across all services | Global (entire flow) | `trace-123456` |
| **SpanId** | Tracks a single unit of work within one service | Local (one operation) | `span-abc789` |

---

## 🧠 TraceId (aka RequestId / CorrelationId)

A **TraceId** is a globally unique identifier that follows a request from its entry point through every service it touches.

```yaml
Key Characteristics:
  - Generated at the request entry point (API Gateway or first service)
  - Passed to ALL downstream services
  - Same TraceId appears in logs across ALL services
  - Enables you to correlate logs from different systems
```

### Real Example

When a user places an order, the TraceId `order-789` appears in:

- API Gateway logs
- Order Service logs
- Payment Service logs
- Inventory Service logs
- Shipping Service logs

**One TraceId = One request's journey across the entire system.**

---

## 🧠 SpanId

A **SpanId** represents a single unit of work within one service. Each service creates its own SpanId for the work it performs.

```yaml
Key Characteristics:
  - Generated per operation within a service
  - Links to the parent SpanId to show hierarchy
  - Different SpanIds for different operations in the same trace
  - Helps visualize the call tree in tools like Jaeger/Zipkin
```

---

## 🏗️ The Relationship Hierarchy

```
TraceId: abc123 (one request, entire flow)
│
├── Span 1: API Gateway (root)
│   └── SpanId = TraceId (root span rule!)
│
├── Span 2: Order Service
│   ├── Parent: Span 1
│   └── Child spans: validateUser, calculateTotal
│
├── Span 3: Payment Service
│   └── Parent: Span 2
│
└── Span 4: Inventory Service
    └── Parent: Span 2
```

**Important Rule:** The Root Span (first service) has its **SpanId equal to the TraceId**.

---

## 📤 Propagation: How IDs Travel Between Services

Trace context is propagated via **HTTP headers** (or message queue headers).

### Standard Headers (B3 Format — Zipkin)

```
X-B3-TraceId: 0c2cc9583f004a4167bde5d7dcd81f84
X-B3-SpanId: 67bde5d7dcd81f84
X-B3-ParentSpanId: 4e4f5b6c7d8e9f0a
```

### W3C Trace Context Standard

```
traceparent: 00-0c2cc9583f004a4167bde5d7dcd81f84-67bde5d7dcd81f84-01
tracestate: vendor=some-value
```

---

## 💻 Implementation Examples

### API Gateway (Entry Point) — Generate Trace

```javascript
// Node.js / Express
const { v4: uuidv4 } = require('uuid');

app.use((req, res, next) => {
  // Generate TraceId if not present
  const traceId = req.headers['x-trace-id'] || uuidv4();
  
  // Generate Root SpanId
  const spanId = uuidv4().replace(/-/g, '').substring(0, 16);
  
  // Store in request context
  req.traceContext = {
    traceId,
    spanId,
    parentSpanId: null  // Root span has no parent
  };
  
  // Add to response headers for debugging
  res.setHeader('x-trace-id', traceId);
  
  next();
});
```

### Service-to-Service Call — Propagate Context

```javascript
// Service A calling Service B
async function callServiceB() {
  const traceId = req.traceContext.traceId;
  const parentSpanId = req.traceContext.spanId;
  
  // Generate new SpanId for this outgoing call
  const newSpanId = uuidv4().replace(/-/g, '').substring(0, 16);
  
  const response = await axios.get('https://service-b/api', {
    headers: {
      'X-B3-TraceId': traceId,
      'X-B3-SpanId': newSpanId,
      'X-B3-ParentSpanId': parentSpanId
    }
  });
  
  return response.data;
}
```

### Service B — Extract and Use Trace Context

```javascript
// Service B receives request
app.use((req, res, next) => {
  // Extract from incoming headers
  const traceId = req.headers['x-b3-traceid'];
  const spanId = req.headers['x-b3-spanid'];
  const parentSpanId = req.headers['x-b3-parentspanid'];
  
  // Store in request context
  req.traceContext = {
    traceId,
    spanId,
    parentSpanId
  };
  
  // Add to logs
  console.log(`[Trace: ${traceId}] [Span: ${spanId}] Processing request`);
  
  next();
});
```

---

## 📊 Logging Integration with MDC

In Java (using SLF4J **MDC — Mapped Diagnostic Context**), you can automatically include trace IDs in all logs:

```java
// Using Spring Cloud Sleuth (automatically does this!)
import org.slf4j.MDC;

@RestController
public class OrderController {
    
    @GetMapping("/orders/{id}")
    public Order getOrder(@PathVariable String id) {
        // MDC automatically populated by Sleuth
        // Log output: [traceId=abc123, spanId=def456] Fetching order 123
        
        log.info("Fetching order {}", id);
        return orderService.findById(id);
    }
}
```

### Log Output Example

```
2024-01-15 10:30:45.123 [order-service,trace=abc123,span=def456] INFO  - Fetching order 123
2024-01-15 10:30:45.234 [payment-service,trace=abc123,span=ghi789] INFO - Processing payment for order 123
2024-01-15 10:30:45.345 [inventory-service,trace=abc123,span=jkl012] INFO - Checking stock for order 123
```

**Same TraceId across all services** → grep once, see the full journey.

---

## 📈 Trace Visualization (Jaeger Example)

```
Trace: abc123
├── Service: API Gateway (Duration: 245ms)
│   └── Span: /api/orders [gateway-span]
│
├── Service: Order Service (Duration: 180ms)
│   └── Span: createOrder [order-span]
│       ├── Span: validateUser [validate-span] (45ms)
│       └── Span: calculateTotal [calc-span] (30ms)
│
├── Service: Payment Service (Duration: 150ms)
│   └── Span: processPayment [payment-span] (150ms)
│
└── Service: Inventory Service (Duration: 60ms)
    └── Span: reserveStock [inventory-span] (60ms)
```

---

## ✅ Best Practices Checklist

```yaml
1. Generate at Entry:
   - [ ] Create TraceId at first point of contact (API Gateway / Front Controller)
   - [ ] Never trust client-provided IDs without validation

2. Propagate Always:
   - [ ] Pass trace context to ALL downstream services
   - [ ] Include in HTTP headers, message queues, async jobs
   - [ ] Never drop or modify existing trace headers

3. Log Consistently:
   - [ ] Include [traceId, spanId] in ALL log statements
   - [ ] Use MDC (Mapped Diagnostic Context) in Java
   - [ ] Structured logging (JSON) works best

4. Choose Standards:
   - [ ] B3 format (X-B3-* headers) — Zipkin compatible
   - [ ] W3C Trace Context — Emerging standard
   - [ ] Be consistent across all services!

5. Sampling Strategy:
   - [ ] 100% sampling for development
   - [ ] 1-10% sampling in production (or based on traffic)
   - [ ] Always sample errors (override sampling)
```

---

## 🛠️ Tools & Frameworks That Do This Automatically

| Technology | Library/Framework | How It Helps |
|------------|-------------------|--------------|
| **Java/Spring** | Spring Cloud Sleuth | Auto-generates TraceId/SpanId, propagates headers, MDC integration |
| **Node.js** | OpenTelemetry | Auto-instrumentation, context propagation |
| **Python** | OpenTelemetry | Flask/Django auto-instrumentation |
| **Go** | LabKit (GitLab) | Correlation ID propagation |
| **All Languages** | OpenTelemetry | Vendor-agnostic tracing standard |

---

## ❓ Interview Practice

### Question 1: "What's the difference between TraceId and SpanId?"

**Architect's Answer:**

> "TraceId is the global ID that follows a request across all services—one TraceId for the entire flow. SpanId is the local ID for a single operation within one service. A trace has one TraceId but many SpanIds—one per service call or operation. The root span's SpanId often equals the TraceId. Together they let you reconstruct the full request journey and build a call tree in tools like Jaeger."

### Question 2: "How do you propagate trace context between microservices?"

**Architect's Answer:**

> "Via HTTP headers. We use B3 format—X-B3-TraceId, X-B3-SpanId, X-B3-ParentSpanId—or W3C Trace Context with traceparent. The caller adds these headers to every outgoing request. The callee extracts them and uses them for its own span, passing them down to any services it calls. For async/message queues, we put the same headers in the message metadata. Never drop or modify existing trace headers—always propagate."

### Question 3: "Why would you sample traces in production?"

**Architect's Answer:**

> "Volume. At scale, 100% tracing can overwhelm storage and add latency. We sample 1–10% of requests in production. But we always sample errors—if a request fails, we capture the full trace. That way we get representative performance data without the cost, but we never lose visibility into failures."

---

## 🔗 Connecting to Previous Days

| Day | Concept | How It Connects |
|-----|---------|-----------------|
| Day 6 | Design for Failure | Tracing helps debug failures across services |
| Day 22 | 7 Layers HLD | Trace flows through Layer 2 (LB) → Layer 3 (App) → Layer 5 (Queue) |
| Day 29 | Forward/Reverse Proxy | API Gateway is often the trace entry point |
| Day 32 | Scheduled Locks | Async jobs need trace context propagated via message headers |

---

## ✅ Day 33 Action Items

1. **Audit your services** — Do they propagate trace headers? Log trace IDs?
2. **Add trace context** — Use OpenTelemetry or Sleuth for auto-instrumentation
3. **Standardize format** — Pick B3 or W3C and use it everywhere
4. **Configure sampling** — 100% in dev, 1–10% in prod, 100% for errors
5. **Try Jaeger/Zipkin** — Visualize a trace end-to-end

---

## 🚀 Key Architect Principles

| Principle | What It Means |
|-----------|---------------|
| **Generate at entry** | TraceId created at API Gateway or first service |
| **Propagate always** | Pass headers to every downstream call—HTTP, gRPC, Kafka |
| **Log consistently** | Every log line should include traceId and spanId |
| **Root span rule** | Root span's SpanId = TraceId |
| **Sample smartly** | Reduce volume in prod, but always trace errors |

---

## 💡 Key Takeaway

> **"TraceId is the global ID that follows a request across ALL services. SpanId is the local ID for one operation within a service. Together, they let you reconstruct the entire journey of a request, pinpoint failures, and understand performance bottlenecks. Propagate them via HTTP headers, log them everywhere, and visualize them with tools like Jaeger or Zipkin."**

---

## 🎯 The 30-Second Summary

> *"TraceId = one request, entire flow. SpanId = one operation, one service. Generate at the gateway, propagate via headers, log in every service. Use Jaeger to see the full tree. Sample in prod, but always trace errors."*

---

*— Sunchit Dudeja*  
*Day 33 of 50: System Design Interview Preparation Series*

---

> 🎯 **Interview Edge:** When asked about debugging distributed systems, say: "We use distributed tracing with TraceId and SpanId. TraceId follows the request across all services; SpanId tracks each operation. We propagate via B3 or W3C headers, log them in every service with MDC, and visualize in Jaeger. When a user reports an error, we grep by TraceId and see the exact path and latency of that request across 10+ services." This shows you think like an architect.
