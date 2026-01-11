# Capacity Estimation: Think Like an Architect
### Day 5 of 50 - System Design Interview Preparation Series

**By Sunchit Dudeja**

---

## 🎯 Welcome to Day 5!

Yesterday, we learned how clients and servers work together. Today, we learn how to **size** those servers. Welcome to the mental shift that separates average developers from system architects.

> A developer builds the menu. An architect plans the entire kitchen.

---

## 🧠 Developer vs Architect: The Mental Gap

### What a Developer Thinks:

| Focus | Question |
|-------|----------|
| Features | "What buttons should the app have?" |
| APIs | "What endpoints do I need to build?" |
| Code | "How do I make this function work?" |

### What an Architect Thinks:

| Focus | Question |
|-------|----------|
| Scale | "What if 1 million users hit this at once?" |
| Resources | "How much memory and CPU do we need?" |
| Cost | "How much will this cost when we grow 10x?" |
| Failure | "What breaks first under pressure?" |

> **The Shift:** Don't just build the meal—plan the entire kitchen.

---

## 🍽️ The Restaurant Kitchen Analogy

Imagine you're planning a restaurant:

### Developer View (The Menu)

- "We need 20 dishes on the menu"
- "Each dish needs a recipe"
- "Let's start cooking"

### Architect View (The Kitchen)

| Concern | Question |
|---------|----------|
| 🚦 Traffic | "How many customers at dinner rush?" |
| 📦 Storage | "How much ingredient stock do we need?" |
| ⛽ Resources | "How much gas burns at full capacity?" |
| 👥 Scaling | "Can the kitchen handle a sudden crowd?" |
| 💰 Cost | "What's our food cost per plate?" |

> **Key Insight:** The architect predicts pressure points, sizes resources, and prepares for both the feast and the famine.

---

## 📊 Why Capacity Estimation Matters

In interviews and real systems, you need to answer:

| Question | Why It Matters |
|----------|----------------|
| How many users? | Determines server count |
| How much data? | Determines storage needs |
| How many requests/second? | Determines throughput requirements |
| How much bandwidth? | Determines network capacity |
| How much will it cost? | Determines business viability |

---

## 🔢 The Numbers You Must Know

Before estimating anything, memorize these:

### Storage Units

| Unit | Size | Example |
|------|------|---------|
| 1 KB | 1,000 bytes | A short text message |
| 1 MB | 1,000 KB | A high-res photo |
| 1 GB | 1,000 MB | A movie |
| 1 TB | 1,000 GB | 500 hours of video |
| 1 PB | 1,000 TB | Netflix's entire library |

### Time Units

| Unit | Seconds |
|------|---------|
| 1 minute | 60 |
| 1 hour | 3,600 |
| 1 day | 86,400 (~100K) |
| 1 month | 2.5 million (~2.5M) |
| 1 year | 31 million (~30M) |

### Traffic Multipliers

| Metric | Quick Math |
|--------|------------|
| Requests per day | Total users × actions per user |
| Requests per second (RPS) | Daily requests ÷ 86,400 |
| Peak RPS | Average RPS × 2 to 5 |

---

## 🎯 Real Example: Instagram Stories

Let's estimate capacity for Instagram Stories:

### Step 1: User Numbers

| Metric | Estimate |
|--------|----------|
| Daily Active Users (DAU) | 500 million |
| Users posting stories | 20% = 100 million |
| Stories per user | 3 per day |
| Total daily stories | 300 million |

### Step 2: Storage

| Metric | Calculation |
|--------|-------------|
| Average story size | 2 MB (image/short video) |
| Daily storage | 300M × 2 MB = 600 TB/day |
| Monthly storage | 600 TB × 30 = 18 PB/month |

### Step 3: Requests

| Metric | Calculation |
|--------|-------------|
| Views per story | ~100 average |
| Total daily views | 300M × 100 = 30 billion |
| Views per second | 30B ÷ 86,400 = ~350,000 RPS |

### Step 4: Bandwidth

| Metric | Calculation |
|--------|-------------|
| Data per view | 2 MB |
| Daily bandwidth | 30B × 2 MB = 60 PB/day |
| Bandwidth per second | 60 PB ÷ 86,400 = ~700 GB/s |

> **Architect's Insight:** These numbers tell us we need massive CDN infrastructure, distributed storage, and caching at every layer.

---

## 📝 The Estimation Framework

Use this 5-step framework for any system:

```
┌─────────────────────────────────────────────────────────────┐
│              CAPACITY ESTIMATION FRAMEWORK                   │
├─────────────────────────────────────────────────────────────┤
│  1. USERS      → How many? DAU, MAU, concurrent?           │
│  2. ACTIONS    → What do they do? How often?               │
│  3. DATA       → How big is each action's data?            │
│  4. STORAGE    → Total data × retention period             │
│  5. BANDWIDTH  → Data transferred per second               │
└─────────────────────────────────────────────────────────────┘
```

---

## 🧮 Quick Mental Math Tricks

### The 80/20 Rule

- 20% of users generate 80% of traffic
- Design for the active 20%, not the silent 80%

### The Rule of 86,400

- Seconds in a day ≈ 100,000 (for quick math)
- Daily requests ÷ 100,000 ≈ RPS

### The 2x-5x Peak Rule

- Peak traffic = 2x to 5x average traffic
- Always design for peak, not average

### The 3-Year Rule

- Estimate storage for 3 years ahead
- Growth compounds fast

---

## 🎮 Practice: WhatsApp Messages

Let's estimate WhatsApp's message system:

### Given:
- 2 billion users
- 500 million DAU
- Average 50 messages/user/day

### Calculate:

| Metric | Your Estimate |
|--------|---------------|
| Daily messages | 500M × 50 = 25 billion |
| Messages per second | 25B ÷ 86,400 = ~290,000 |
| Average message size | ~10 KB (with metadata) |
| Daily storage | 25B × 10 KB = 250 TB |
| Monthly storage | 250 TB × 30 = 7.5 PB |

> **Think Further:** This doesn't include media (photos, videos), which 10x the storage needs!

---

## ⚡ Common Interview Scenarios

### Scenario 1: URL Shortener (like bit.ly)

| Metric | Estimation |
|--------|------------|
| New URLs/month | 100 million |
| Reads vs Writes | 100:1 ratio |
| Read requests/month | 10 billion |
| Read RPS | 10B ÷ 2.5M = 4,000 RPS |
| Storage per URL | 500 bytes |
| Monthly storage | 100M × 500 = 50 GB |
| 5-year storage | 50 GB × 60 = 3 TB |

### Scenario 2: Video Streaming (like Netflix)

| Metric | Estimation |
|--------|------------|
| DAU | 200 million |
| Watch time/user | 2 hours |
| Concurrent streams | 10% of DAU = 20 million |
| Bitrate | 5 Mbps average |
| Total bandwidth | 20M × 5 Mbps = 100 Tbps |

---

## 🎯 The Architect's Checklist

When designing any system, estimate these:

| Category | What to Estimate |
|----------|------------------|
| **Traffic** | RPS, peak RPS, read/write ratio |
| **Storage** | Data size, growth rate, retention |
| **Bandwidth** | Incoming, outgoing, peak |
| **Memory** | Cache size, session data |
| **Compute** | CPU needs, server count |

---

## ❓ Interview Practice

### Question 1:
> "Estimate the storage needs for Twitter's tweet system."

**Answer:**
> "Twitter has ~400 million DAU. Assume 5% tweet daily = 20 million tweets. Each tweet is ~280 characters = 560 bytes, plus metadata = ~1 KB. Daily storage = 20M × 1 KB = 20 GB. Monthly = 600 GB. Yearly = 7.2 TB for text alone. Add media (images, videos) and you're looking at 100x more—around 700 TB/year."

### Question 2:
> "How would you estimate servers needed for a food delivery app?"

**Answer:**
> "Start with orders: 10 million orders/day. Peak is 3x average during lunch/dinner = 30M equivalent. That's 30M ÷ 86,400 = 350 orders/second at peak. If each server handles 100 orders/second, we need 3-4 servers. Add 2x for redundancy = 8 servers for order processing alone. Then estimate separately for search, tracking, and payments."

---

## 🔗 Connecting to Previous Days

| Day | Concept | How It Connects |
|-----|---------|-----------------|
| Day 2 | SCALED | Capacity estimation helps achieve Scalability and Efficiency |
| Day 3 | NFRs | These estimates ARE your non-functional requirements |
| Day 4 | Client-Server | Now you know how to SIZE those servers |

---

## ✅ Day 5 Action Items

1. **Memorize the numbers** — KB, MB, GB, seconds in a day
2. **Practice one estimation** — Pick any app, estimate its storage
3. **Think in ratios** — Read:Write, Peak:Average
4. **Round generously** — 86,400 → 100,000 for quick math

---

## 💡 Key Takeaway

> **A normal developer asks: "What features do I build?" An architect asks: "What happens when a million people use this at once?"**

Think of it as the difference between cooking for friends at home versus planning a restaurant kitchen. One person worries about the menu. The other worries about the kitchen, the staff, the ingredients for 500 people, and what happens when everyone shows up at the same time.

If you want to think like an architect, don't just build the meal—plan the entire kitchen.

---

*— Sunchit Dudeja*  
*Day 5 of 50: System Design Interview Preparation Series*

---

> 🎯 **Interview Edge:** When asked to design any system, start with: "Let me first estimate the scale we're designing for." Then walk through users → actions → data → storage → bandwidth. This immediately signals architect-level thinking.

