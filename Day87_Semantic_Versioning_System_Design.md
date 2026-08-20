# Day 87/50 — Semantic Versioning | System Design Interview Preparation Series

When designing APIs, libraries, SDKs, microservices, or platform components, one deceptively simple question eventually becomes critical:

> **How do we evolve a system without unexpectedly breaking its consumers?**

This is where **Semantic Versioning (SemVer)** becomes an important system-design concept.

Semantic Versioning gives us a predictable contract for communicating the impact of a software change.

The standard format is:

# `MAJOR.MINOR.PATCH`

For example:

```text
2.4.7
│ │ │
│ │ └── PATCH
│ └──── MINOR
└────── MAJOR
```

The version number isn't just a release label.

From a senior system-design perspective, it is a **communication mechanism between producers and consumers of an API or software component**.

---

# 1. What Does Semantic Versioning Mean?

The basic rule is:

### MAJOR

Increment when you introduce **backward-incompatible changes**.

```text
1.5.2 → 2.0.0
```

Examples:

```text
GET /users/{id}

Old response:
{
  "name": "John"
}
```

becomes:

```text
{
  "fullName": "John"
}
```

If existing clients expect `name`, they may break.

That's a **MAJOR** change.

---

### MINOR

Increment when you add functionality in a **backward-compatible** way.

```text
1.5.2 → 1.6.0
```

For example:

```text
GET /users/{id}

Old response:
{
  "name": "John"
}
```

becomes:

```text
{
  "name": "John",
  "email": "john@example.com"
}
```

Existing consumers can continue using `name`.

A new capability was added without breaking the old contract.

That's a **MINOR** change.

---

### PATCH

Increment for backward-compatible bug fixes.

```text
1.5.2 → 1.5.3
```

For example:

```text
Bug:
Tax calculation incorrectly rounded values.

Fix:
Correct the rounding logic.
```

The API contract hasn't changed.

The implementation has been corrected.

That's a **PATCH** release.

---

# 2. The Most Important Concept: Backward Compatibility

In system design interviews, don't memorize:

> MAJOR = big change  
> MINOR = medium change  
> PATCH = small change

That's too simplistic.

The better mental model is:

> **What happens to existing consumers?**

Ask:

```text
                Change
                  |
                  v
        Does it break existing
             consumers?
             /          \
           Yes           No
            |             |
            v             v
         MAJOR       Is functionality
                      being added?
                       /       \
                     Yes        No
                      |          |
                      v          v
                   MINOR      PATCH
```

This is much closer to how a senior architect thinks.

---

# 3. Why Does This Matter in Distributed Systems?

Consider a large organization with hundreds of services.

```text
                    API Gateway
                         |
          +--------------+--------------+
          |              |              |
          v              v              v
      Service A      Service B      Service C
          |              |
          +-------> User API <--------+
                         |
                         v
                    API Database
```

Service A may depend on Service B's API.

Service C may depend on the same API.

Now imagine Service B changes its response format.

Without versioning:

```text
Service B
   |
   | Breaking change
   v
Consumers
   |
   +---- Service A ❌
   +---- Service C ❌
   +---- Mobile App ❌
   +---- External Client ❌
```

One seemingly small change can become a distributed-system incident.

Versioning gives consumers a predictable contract.

---

# 4. API Versioning

Semantic versioning and API versioning are related, but they are not exactly the same thing.

You might expose:

```text
/api/v1/users
/api/v2/users
```

Here, `v1` and `v2` represent API contracts.

Internally, your service might release:

```text
1.4.0
1.4.1
1.5.0
2.0.0
```

These are different layers of versioning.

A mature system may use both:

```text
             Public API
                 |
          API Version
        +--------+--------+
        |                 |
       v1                v2
        |                 |
        +--------+--------+
                 |
             Service
                 |
          Internal Release
                 |
        2.3.0 → 2.3.1
```

The key question is:

> **What contract are we versioning?**

---

# 5. Example: Evolving an API Safely

Suppose version 1 returns:

```json
{
  "firstName": "John",
  "lastName": "Doe"
}
```

We want to introduce:

```json
{
  "fullName": "John Doe"
}
```

A dangerous approach would be:

```text
Remove firstName
Remove lastName
Add fullName
```

Existing consumers could fail.

A safer evolution might be:

```json
{
  "firstName": "John",
  "lastName": "Doe",
  "fullName": "John Doe"
}
```

Now existing clients continue working.

New clients can use `fullName`.

Eventually, we can introduce a new major API version if we need to remove the old fields.

```text
v1
 |
 +-- firstName
 +-- lastName

v2
 |
 +-- fullName
```

This allows controlled migration.

---

# 6. Versioning Is Really About Consumer Contracts

Imagine an organization with:

```text
1 API
    |
    +---- Web Application
    +---- iOS App
    +---- Android App
    +---- Partner A
    +---- Partner B
    +---- Internal Service A
    +---- Internal Service B
```

You don't control the release cycle of all these consumers.

That's why a senior architect asks:

> **Who are my consumers, and how quickly can they migrate?**

This changes the versioning strategy.

For an internal service where you control all consumers, you may be able to make coordinated changes.

For a public API with thousands of external clients, breaking changes are significantly more expensive.

---

# 7. Semantic Versioning in Microservices

Microservices make versioning even more important.

Consider:

```text
                  Order Service
                       |
                       v
                 Payment API
                       |
                       v
                 Payment Service
```

Suppose Payment Service changes:

```text
POST /payments
```

from:

```json
{
  "amount": 100,
  "currency": "USD"
}
```

to:

```json
{
  "paymentAmount": 100,
  "currencyCode": "USD"
}
```

Every consumer must understand the new contract.

If Order Service hasn't been upgraded, production traffic can fail.

This is why **contract management** is fundamental to microservice architecture.

---

# 8. Versioning Strategies

There isn't one universal approach.

Common strategies include:

### URL Versioning

```text
/api/v1/orders
/api/v2/orders
```

Simple and explicit.

---

### Header Versioning

```text
Accept: application/vnd.company.orders.v2+json
```

Keeps the URL stable while expressing the contract through headers.

---

### Query Parameter

```text
/api/orders?version=2
```

Easy to implement, but generally less elegant for long-term API design.

---

### Content Negotiation

The client specifies which representation it understands.

```text
Accept: application/json; version=2
```

This can be useful when the resource remains conceptually the same but representations evolve.

---

# 9. The Senior-Level Problem: Version Explosion

Versioning solves one problem but can create another.

Imagine:

```text
v1
v2
v3
v4
v5
```

Now the service has to maintain multiple contracts.

Architecture becomes:

```text
                    API
                     |
       +-------------+-------------+
       |       |       |      |     |
      v1      v2      v3     v4    v5
       |       |       |      |     |
       +-------+-------+------+-----+
                       |
                  Business Logic
```

This can become expensive.

You now have:

- Multiple schemas
- Multiple test suites
- Multiple documentation versions
- Multiple client behaviors
- More operational complexity

Therefore:

> **Versioning is not a substitute for good API evolution.**

The goal should be to minimize unnecessary breaking changes.

---

# 10. Deprecation Is Part of Versioning

A mature API shouldn't suddenly disappear.

Instead:

```text
Introduce
   ↓
Adopt
   ↓
Deprecate
   ↓
Migration period
   ↓
Remove
```

For example:

```text
v1 → Active
v2 → Active

v1 → Deprecated
v2 → Active

v1 → Sunset
v2 → Active
```

During the migration period, consumers receive communication such as:

```text
Deprecated:
GET /api/v1/users

Use:
GET /api/v2/users

Sunset:
2027-06-01
```

This is especially important for public APIs.

---

# 11. Database Schema Versioning

Semantic versioning isn't limited to APIs.

Database schemas evolve too.

Suppose we have:

```text
users

id
name
email
```

We want to introduce:

```text
phone_number
```

A safe migration could be:

```text
Step 1:
Add phone_number

Step 2:
Deploy application that understands both old and new schema

Step 3:
Backfill data

Step 4:
Start writing phone_number

Step 5:
Migrate consumers

Step 6:
Remove old dependency
```

This is commonly known as an **expand-and-contract** approach.

The architecture becomes:

```text
        Expand
           ↓
    Support old + new
           ↓
       Migrate
           ↓
      Contract
```

This is much safer than making a destructive schema change in a single deployment.

---

# 12. Semantic Versioning and Deployment Strategy

Versioning becomes even more powerful when combined with safe deployment strategies.

For example:

```text
             v1
              |
        +-----+-----+
        |           |
       90%         10%
        |           |
      Old         New
                 version
```

Using:

- Canary deployments
- Blue/green deployments
- Feature flags
- Gradual rollouts

we can validate a new version before moving all traffic.

A typical migration might look like:

```text
v1
 |
 |---- 95% traffic
 |
 +---- 5% traffic ---> v2
                         |
                    Monitor errors
                         |
                    Increase traffic
                         |
                       v2 = 100%
```

This is where versioning becomes part of **operational architecture**, not merely release management.

---

# 13. What About `0.x` Versions?

The image highlights:

```text
0.0.0
```

as initial development.

A common SemVer convention is that versions below `1.0.0` indicate an API that has not yet reached its first stable release.

For example:

```text
0.1.0
0.2.0
0.2.1
```

The exact compatibility expectations should still be documented, because pre-1.0 projects may evolve more aggressively than stable APIs.

Once a project establishes a stable public contract:

```text
1.0.0
```

becomes a meaningful milestone.

It effectively communicates:

> **Consumers can now expect a stable public API contract.**

---

# 14. A Real System Design Interview Question

Imagine the interviewer asks:

> **“You have 500 microservices consuming a common User API. How would you evolve the API without breaking existing services?”**

A weak answer:

> “We'll create `/v2`.”

A stronger senior-level answer:

### Step 1 — Identify consumers

```text
Who consumes the API?
```

Understand internal services, mobile clients, external partners, etc.

### Step 2 — Prefer backward-compatible changes

Add fields rather than removing or renaming existing ones.

### Step 3 — Introduce a new version only when necessary

For example:

```text
/v1/users
/v2/users
```

### Step 4 — Run both versions

Allow consumers time to migrate.

### Step 5 — Track adoption

Measure:

```text
v1 traffic
v2 traffic
```

### Step 6 — Deprecate v1

Communicate the sunset timeline.

### Step 7 — Remove v1

Only after migration reaches an agreed threshold.

That demonstrates much stronger architectural thinking.

---

# 15. The Golden Rule

When designing APIs and distributed systems, remember:

> **Backward compatibility is usually cheaper than forcing every consumer to migrate simultaneously.**

Therefore, prefer:

```text
Add
  ↓
Migrate
  ↓
Deprecate
  ↓
Remove
```

over:

```text
Change
  ↓
Break everyone
  ↓
Emergency fixes
```

---

# Final Takeaway

Semantic Versioning is often taught as:

```text
MAJOR.MINOR.PATCH
```

But in system design, the deeper lesson is about **managing change safely**.

### MAJOR

```text
Breaking contract
```

### MINOR

```text
Backward-compatible functionality
```

### PATCH

```text
Backward-compatible fix
```

And at a senior architecture level, versioning connects directly to:

- API contracts
- Microservices
- Database migrations
- Backward compatibility
- Deprecation
- Consumer migration
- Canary releases
- Feature flags
- Deployment strategies
- Operational risk

The goal isn't to create more versions.

The goal is to **evolve a distributed system without forcing every dependent component to change at the same time**.

That is the real value of Semantic Versioning in system design.

**Day 87/50 complete. 🚀**

#SystemDesign #SoftwareArchitecture #SemanticVersioning #API #Microservices #BackendEngineering #DistributedSystems #SystemDesignInterview #Architecture #100DaysOfCode
