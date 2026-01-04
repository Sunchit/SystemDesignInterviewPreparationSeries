# Functional vs Non-Functional Requirements
### Day 3 of 50 - System Design Interview Preparation Series

**By Sunchit Dudeja**

---

## 🎯 Welcome to Day 3!

Yesterday, we learned the **SCALED framework** — the six superpowers every system needs. But here's the million-dollar question:

> **How do we decide WHICH superpowers our system needs?**

The answer lies in understanding **requirements** — the two pillars of every project.

---

## 🏛️ The Two Pillars of Every Project

Every system design starts with two fundamental questions:

| Pillar | Question | Focus |
|--------|----------|-------|
| **Functional Requirements** | WHAT does the system do? | Features & Capabilities |
| **Non-Functional Requirements** | HOW WELL does it do it? | Quality & Performance |

Think of it this way:
- **Functional** = The features you build
- **Non-Functional** = The quality standards those features must meet

---

## 🎂 The Birthday Cake Analogy

Let's make this concrete with a simple example: baking a birthday cake.

### Functional Requirements (WHAT)

These define what the cake should be:

| Requirement | Description |
|-------------|-------------|
| ✅ Chocolate flavor | The cake must be chocolate |
| ✅ Cream frosting | Must have cream frosting on top |
| ✅ "Happy Birthday" writing | Personalized message required |
| ✅ Serves 10 people | Must be large enough for 10 servings |

### Non-Functional Requirements (HOW WELL)

These define how well the cake should be made:

| Requirement | Description |
|-------------|-------------|
| ⏰ Ready in 3 hours | Time constraint |
| 🧼 Kitchen stays clean | Process quality |
| 💰 Cost under ₹500 | Budget constraint |
| 🍰 Stays fresh for 2 days | Durability requirement |

### The Critical Insight

> **You can bake a perfect chocolate cake (functional ✅) that costs ₹5000 and dirties every dish in the kitchen (non-functional ❌).**

**Both matter!** A system that has all features but crashes under load, responds slowly, or costs too much to run is a **failed system**.

---

## 📸 Real Example: Instagram

Let's apply this to a real-world system:

### Instagram's Functional Requirements

| Feature | Description |
|---------|-------------|
| 📤 Upload photos | Users can upload images |
| 📰 View feed | Users can scroll through posts |
| ❤️ Like posts | Users can interact with content |
| 💬 Comment | Users can leave comments |
| 👤 Follow users | Users can follow other accounts |

### Instagram's Non-Functional Requirements

| Quality Attribute | Requirement |
|-------------------|-------------|
| ⚡ **Latency** | Photos load in < 2 seconds |
| 📈 **Scalability** | Supports 1M+ concurrent users |
| 🔒 **Security** | User data is encrypted |
| ✅ **Availability** | 99.99% uptime |
| 💾 **Durability** | Photos never get lost |

### The Connection to SCALED

Notice how Non-Functional Requirements map directly to our **Day 2 SCALED framework**:

| NFR Category | SCALED Superpower |
|--------------|-------------------|
| Response time | **L**atency |
| Handle growth | **S**calability |
| Always online | **A**vailability |
| Data safety | **D**urability |
| Resource usage | **E**fficiency |
| Data accuracy | **C**onsistency |

---

## 📋 The 3-Step Framework for Interviews

When you're given a system design problem, follow this framework:

### Step 1: Clarify Functional Requirements

Ask these questions:

| Question | Why It Matters |
|----------|----------------|
| "Who are our users?" | Determines scale and use cases |
| "What are the core features?" | Focuses on MVP, avoids scope creep |
| "What's out of scope?" | Prevents over-engineering |
| "What are the user flows?" | Clarifies feature interactions |

**Example for Instagram:**
> "The core features are: upload photos, view feed, like, comment, and follow. Stories and Reels are out of scope for this design."

### Step 2: Define Non-Functional Requirements

For each, ask the right questions:

| Category | Questions to Ask |
|----------|------------------|
| **Performance** | Expected users? Requests per second? Acceptable latency? |
| **Reliability** | Acceptable downtime? Recovery time? |
| **Scalability** | Expected growth? Peak vs average load? |
| **Security** | Data sensitivity? Compliance requirements? |
| **Maintainability** | Team size? Tech constraints? |

**Example for Instagram:**
> "We need to support 100M daily active users, with p99 latency under 200ms. We target 99.99% availability with no data loss."

### Step 3: Prioritize Trade-offs

This is where you show senior-level thinking:

> "If we have to choose between X and Y, which wins?"

| Trade-off Decision | Example |
|--------------------|---------|
| Consistency vs Availability | "We'll sacrifice strong consistency for availability — users can tolerate slightly stale like counts." |
| Latency vs Accuracy | "We'll show cached results for speed, then update in background." |
| Cost vs Performance | "We'll use spot instances for batch processing to reduce costs." |

---

## 🎯 Common Non-Functional Requirements

Here's a comprehensive list for your interviews:

### Performance
| Metric | Description | Example |
|--------|-------------|---------|
| Latency | Response time | p99 < 200ms |
| Throughput | Requests/second | 10K RPS |
| Bandwidth | Data transfer rate | 1 Gbps |

### Reliability
| Metric | Description | Example |
|--------|-------------|---------|
| Availability | Uptime percentage | 99.99% |
| Fault Tolerance | Survive failures | No single point of failure |
| Disaster Recovery | Recover from disasters | RPO: 1 hour, RTO: 4 hours |

### Scalability
| Metric | Description | Example |
|--------|-------------|---------|
| Vertical | Scale up | Add more CPU/RAM |
| Horizontal | Scale out | Add more servers |
| Data Growth | Handle more data | Sharding strategy |

### Security
| Metric | Description | Example |
|--------|-------------|---------|
| Authentication | Verify identity | OAuth 2.0, JWT |
| Authorization | Control access | RBAC, ABAC |
| Encryption | Protect data | TLS, AES-256 |

### Maintainability
| Metric | Description | Example |
|--------|-------------|---------|
| Modularity | Component isolation | Microservices |
| Observability | Monitor & debug | Logging, metrics, tracing |
| Deployability | Easy releases | CI/CD, blue-green deployment |

---

## ❓ Interview Questions Practice

Practice answering these:

### Question 1: E-commerce Platform
> "What are the functional and non-functional requirements for Amazon's checkout system?"

**Functional:**
- Add items to cart
- Apply coupons/discounts
- Process payment
- Generate order confirmation

**Non-Functional:**
- Latency: Checkout in < 3 seconds
- Availability: 99.999% (it's revenue-critical)
- Consistency: Payment must be exactly-once (no double charging)
- Security: PCI-DSS compliance

### Question 2: Messaging App
> "What are the NFRs for WhatsApp?"

**Non-Functional:**
- Latency: Messages delivered in < 1 second
- Scalability: 2B+ users, 100B messages/day
- Availability: 99.99% uptime
- Durability: Messages never lost
- Security: End-to-end encryption

---

## 🔗 Connecting the Dots

Here's how the first 3 days connect:

```
Day 1: WHY System Design?
         ↓
         Companies need architects who can build
         scalable, reliable systems
         ↓
Day 2: WHAT makes a great system?
         ↓
         SCALED: Scalability, Consistency, Availability,
         Latency, Efficiency, Durability
         ↓
Day 3: HOW do we decide which superpowers to prioritize?
         ↓
         Functional + Non-Functional Requirements
         → Trade-offs based on business needs
```

---

## ✅ Day 3 Action Items

1. **Practice the Cake Analogy** — Use it to explain FR vs NFR to anyone
2. **Analyze 3 Apps** — List functional and non-functional requirements for apps you use daily
3. **Master the 3-Step Framework** — Clarify FR → Define NFR → Prioritize Trade-offs
4. **Connect to SCALED** — Map every NFR to a SCALED superpower

---

## 🔜 Coming Up: Day 4

Tomorrow, we dive into **Client-Server Architecture & HTTP Basics** — understanding how systems actually communicate.

We'll explore:
- How clients and servers talk
- HTTP request/response cycle
- REST principles
- Real example: Restaurant ordering system

---

## 💡 Key Takeaway

> **Functional = Features. Non-Functional = Quality.**  
> **Ask about both. Design accordingly.**

In interviews, always start by clarifying requirements. It shows maturity, prevents scope creep, and sets you up for a structured design discussion.

The best architects don't just build features — they build features that perform, scale, and last.

---

*— Sunchit Dudeja*  
*Day 3 of 50: System Design Interview Preparation Series*

---

> 🎯 **Remember:** A system that has all features but crashes under load is still a failed system. Both pillars matter equally.

