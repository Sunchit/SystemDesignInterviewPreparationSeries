# 🐳 Docker Production Commands: The Architect's Essential Guide

*Why Your "docker run myapp" Works Locally But Crashes Production at 3 AM*

---

## Introduction

Every developer knows `docker run`. But there's a massive gap between running a container on your laptop and running one in production that handles ₹10 crores in transactions daily.

This guide breaks down the **production-grade Docker command** that architects use at companies like Flipkart, Razorpay, and Netflix. We'll understand not just *what* each flag does, but *why* it's essential.

---

## The Developer vs Architect Mindset

```
Developer's Command:
$ docker run myapp

Architect's Command:
$ docker run -d \
    --name payment-service-prod \
    -p 8080:8080 \
    -e SPRING_PROFILES_ACTIVE=production \
    -e DB_HOST=postgres.cluster.internal \
    -v /data/payments/logs:/app/logs \
    --memory="512m" \
    --cpus="1.0" \
    --restart unless-stopped \
    --log-opt max-size=10m \
    --health-cmd="curl -f http://localhost:8080/health || exit 1" \
    --read-only \
    --user=1000:1000 \
    myregistry.com/payment-service:1.2.3
```

Let's understand each piece.

---

## Part 1: The Basics

### `-d` (Detached Mode)

```bash
docker run -d myapp
```

**What it does:** Runs the container in the background.

**Without `-d`:**
```
┌─────────────────────────────────────────┐
│  Terminal                               │
│  $ docker run myapp                     │
│  [App logs streaming here...]           │
│  [Terminal is locked]                   │
│  [Ctrl+C kills the container]           │
│  █ (cursor stuck)                       │
└─────────────────────────────────────────┘
```

**With `-d`:**
```
┌─────────────────────────────────────────┐
│  Terminal                               │
│  $ docker run -d myapp                  │
│  a1b2c3d4e5f6... (container ID)         │
│  $ (terminal is free!)                  │
│  $ docker logs myapp                    │
└─────────────────────────────────────────┘
```

**Why it matters:** In production, if you close your SSH session without `-d`, the container dies. With `-d`, it runs independently.

---

### `--name` (Container Identity)

```bash
docker run -d --name payment-service-prod myapp
```

**What it does:** Gives your container a human-readable name.

**Without `--name`:**
```bash
$ docker ps
CONTAINER ID   IMAGE   NAMES
a1b2c3d4e5f6   myapp   quirky_einstein    # Random name!
f7g8h9i0j1k2   myapp   angry_hopper       # Which is which?
```

**With `--name`:**
```bash
$ docker ps
CONTAINER ID   IMAGE   NAMES
a1b2c3d4e5f6   myapp   payment-service-prod
f7g8h9i0j1k2   myapp   inventory-service-prod
```

**Why it matters:** 
```bash
# Easy management
docker logs payment-service-prod
docker stop payment-service-prod
docker restart payment-service-prod

# vs trying to remember "quirky_einstein"
```

---

### `-p` (Port Mapping)

```bash
docker run -d -p 8080:8080 myapp
```

**What it does:** Maps host port to container port.

```
Format: -p HOST_PORT:CONTAINER_PORT

┌─────────────────┐         ┌─────────────────┐
│   HOST (VM)     │         │    CONTAINER    │
│                 │         │                 │
│   Port 8080 ────┼────────►│ Port 8080       │
│                 │         │                 │
└─────────────────┘         └─────────────────┘
```

**Common patterns:**
```bash
-p 8080:8080      # Same port
-p 80:8080        # External 80 → Internal 8080
-p 127.0.0.1:8080:8080  # Only localhost can access
```

**Why it matters:** Without port mapping, your service is invisible to the outside world.

---

## Part 2: Configuration

### `-e` (Environment Variables)

```bash
docker run -d \
  -e SPRING_PROFILES_ACTIVE=production \
  -e DB_HOST=postgres.cluster.internal \
  -e DB_PASSWORD=secure123 \
  myapp
```

**What it does:** Injects configuration into the container.

**The 12-Factor App Principle:**
```
┌─────────────────────────────────────────────────────────────┐
│  NEVER hardcode configuration in your application          │
│                                                             │
│  ❌ BAD (in code):                                          │
│     String dbHost = "postgres.prod.internal";               │
│                                                             │
│  ✅ GOOD (from environment):                                │
│     String dbHost = System.getenv("DB_HOST");               │
│                                                             │
│  Same image works in dev, staging, and production!          │
└─────────────────────────────────────────────────────────────┘
```

**Real-world example:**
```bash
# Development
docker run -e DB_HOST=localhost -e DB_PASSWORD=devpass myapp

# Production
docker run -e DB_HOST=postgres.cluster.internal -e DB_PASSWORD=prod_secure_123 myapp
```

**Why it matters:** One image, multiple environments. No rebuilding for each environment.

---

### `-v` (Volumes - Data Persistence)

```bash
docker run -d \
  -v /data/payments/logs:/app/logs \
  -v /data/payments/config:/app/config:ro \
  myapp
```

**What it does:** Mounts host directories into the container.

**The Problem Without Volumes:**
```
Container starts → Writes logs → Container crashes → LOGS GONE!
Container starts → Writes data → Container restarts → DATA GONE!
```

**With Volumes:**
```
┌─────────────────┐         ┌─────────────────┐
│   HOST (VM)     │         │    CONTAINER    │
│                 │         │                 │
│ /data/logs ─────┼────────►│ /app/logs       │
│ (persistent)    │         │ (writes here)   │
│                 │         │                 │
└─────────────────┘         └─────────────────┘

Container dies → Host still has /data/logs → Logs preserved!
```

**The `:ro` flag:**
```bash
-v /data/config:/app/config:ro   # Read-Only

# Container can READ config but can't MODIFY it
# Prevents accidental configuration changes
```

**Why it matters:** 
- Logs survive container restarts (crucial for debugging)
- Configuration externalized (change without rebuild)
- Data persistence for stateful apps

---

## Part 3: Resource Limits (Preventing Disasters)

### `--memory` (Memory Limit)

```bash
docker run -d --memory="512m" myapp
```

**What it does:** Caps the container's memory usage.

**Without `--memory`:**
```
Scenario: Memory Leak at 2 AM

Container starts using 512MB
    ↓
Memory leak → 1GB → 2GB → 4GB
    ↓
Host has 8GB total, other containers need memory
    ↓
OOM Killer activates → Kills random containers
    ↓
💥 Payment service, inventory service, ALL DEAD
```

**With `--memory="512m"`:**
```
Container tries to use > 512MB
    ↓
Docker kills ONLY this container
    ↓
--restart unless-stopped brings it back
    ↓
Other containers unaffected ✅
```

**Why it matters:** One rogue container shouldn't take down your entire host.

---

### `--cpus` (CPU Limit)

```bash
docker run -d --cpus="1.0" myapp
```

**What it does:** Limits CPU cores the container can use.

**The "Noisy Neighbor" Problem:**
```
┌─────────────────────────────────────────────────────────────┐
│  HOST with 4 CPU cores                                      │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Without --cpus:                                            │
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐        │
│  │ Payment (10%)│ │ Report (90%) │ │ Inventory(0%)│        │
│  └──────────────┘ └──────────────┘ └──────────────┘        │
│                                                             │
│  Report job hogs ALL CPU → Payment service slow → Users mad │
│                                                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  With --cpus="1.0" for each:                                │
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐        │
│  │ Payment (1)  │ │ Report (1)   │ │ Inventory(1) │        │
│  └──────────────┘ └──────────────┘ └──────────────┘        │
│                                                             │
│  Each gets fair share → All services responsive ✅          │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**Why it matters:** Ensures critical services always have CPU available.

---

## Part 4: Resilience (Self-Healing)

### `--restart` (Restart Policy)

```bash
docker run -d --restart unless-stopped myapp
```

**What it does:** Automatically restarts the container if it crashes.

**Options:**
| Policy | Behavior |
|--------|----------|
| `no` | Never restart (default) |
| `always` | Always restart, even after reboot |
| `unless-stopped` | Restart unless manually stopped |
| `on-failure:5` | Restart up to 5 times on failure |

**Without `--restart`:**
```
3 AM: Container crashes
3 AM - 7 AM: Service is down (4 hours!)
7 AM: Engineer wakes up, manually restarts
Revenue lost: ₹50 lakhs
```

**With `--restart unless-stopped`:**
```
3 AM: Container crashes
3 AM + 1 second: Docker restarts container automatically
3 AM + 5 seconds: Service is back up
Revenue lost: ₹0
```

**Why it matters:** Self-healing at 3 AM when no one is watching.

---

### `--log-opt` (Log Rotation)

```bash
docker run -d \
  --log-opt max-size=10m \
  --log-opt max-file=3 \
  myapp
```

**What it does:** Prevents logs from filling up your disk.

**Without log rotation:**
```
Day 1:   Logs = 100MB
Day 30:  Logs = 3GB
Day 90:  Logs = 9GB
Day 100: DISK FULL → All containers crash → Production down
```

**With `--log-opt max-size=10m --log-opt max-file=3`:**
```
Keeps only:
  - container-log.json      (current, max 10MB)
  - container-log.json.1    (previous, max 10MB)
  - container-log.json.2    (oldest, max 10MB)

Total max: 30MB, rotates automatically
```

**Why it matters:** Disk full is a silent killer of production systems.

---

## Part 5: Health Checks (Detecting Zombies)

### `--health-*` (Health Check Configuration)

```bash
docker run -d \
  --health-cmd="curl -f http://localhost:8080/health || exit 1" \
  --health-interval=30s \
  --health-timeout=5s \
  --health-retries=3 \
  myapp
```

**What it does:** Tells Docker how to check if the app is actually working.

**The Zombie Container Problem:**
```
┌─────────────────────────────────────────────────────────────┐
│  Container Status: RUNNING ✅                                │
│  Actual Application: DEADLOCKED 💀                           │
│                                                             │
│  Without health check:                                      │
│    Load balancer keeps sending traffic                      │
│    All requests timeout                                     │
│    Users see errors                                         │
│    "But docker ps says it's running!"                       │
│                                                             │
│  With health check:                                         │
│    Docker detects: UNHEALTHY                                │
│    Load balancer removes from pool                          │
│    --restart kicks in                                       │
│    Container replaced with healthy one                      │
└─────────────────────────────────────────────────────────────┘
```

**Health check parameters:**
```bash
--health-cmd="curl -f http://localhost:8080/health || exit 1"
# What command to run (exit 0 = healthy, exit 1 = unhealthy)

--health-interval=30s
# Check every 30 seconds

--health-timeout=5s
# If check takes > 5s, consider it failed

--health-retries=3
# Mark unhealthy after 3 consecutive failures
```

**Why it matters:** A running container isn't necessarily a working container.

---

## Part 6: Security (Defense in Depth)

### `--read-only` (Immutable Filesystem)

```bash
docker run -d --read-only myapp
```

**What it does:** Makes the container's filesystem read-only.

**Attack scenario without `--read-only`:**
```
Attacker exploits vulnerability
    ↓
Writes malware to /tmp/backdoor.sh
    ↓
Executes backdoor
    ↓
💥 System compromised
```

**With `--read-only`:**
```
Attacker exploits vulnerability
    ↓
Tries to write to filesystem
    ↓
❌ "Read-only file system" error
    ↓
Attack fails
```

---

### `--tmpfs` (Temporary Filesystem)

```bash
docker run -d \
  --read-only \
  --tmpfs /tmp:rw,noexec,nosuid,size=128m \
  myapp
```

**What it does:** Allows writes to `/tmp` but with restrictions.

**The flags:**
```
rw        - Read-write (app can write temp files)
noexec    - Can't execute anything written here
nosuid    - Can't set user ID (privilege escalation prevention)
size=128m - Max 128MB (prevents filling up RAM)
```

**Why both?** Some apps need to write temp files. This allows it safely.

---

### `--user` (Non-Root User)

```bash
docker run -d --user=1000:1000 myapp
```

**What it does:** Runs the container as a non-root user.

**Without `--user`:**
```
Container runs as root (UID 0)
    ↓
Attacker exploits vulnerability
    ↓
Has ROOT access inside container
    ↓
Potential container escape → Host compromised
```

**With `--user=1000:1000`:**
```
Container runs as regular user (UID 1000)
    ↓
Attacker exploits vulnerability
    ↓
Has LIMITED access
    ↓
Can't install packages, can't modify system files
    ↓
Blast radius minimized
```

### The Security Triangle

```
              ┌─────────────────┐
              │   --read-only   │
              │  (No malware    │
              │   persistence)  │
              └────────┬────────┘
                       │
          ┌────────────┴────────────┐
          │                         │
          ▼                         ▼
    ┌──────────┐           ┌──────────────┐
    │--user=   │           │   --tmpfs    │
    │ 1000:1000│           │  /tmp:rw,    │
    │(Not root)│           │ noexec,nosuid│
    └──────────┘           └──────────────┘

Combined: Attacker can't write + can't escalate + can't execute
```

---

## Part 7: Image Versioning

### `image:version` (Not `:latest`)

```bash
# ❌ BAD
docker run myapp:latest

# ✅ GOOD  
docker run myregistry.com/payment-service:1.2.3
```

**The `:latest` Trap:**

```
:latest ≠ "most recent version"
:latest = "whatever was last pushed without a tag"

Timeline:
  Monday:    push myapp:latest → v1.0
  Tuesday:   push myapp:latest → v1.1
  Wednesday: push myapp:latest → v1.2 (buggy!)
  
Production Server:
  Monday:    pull myapp:latest → gets v1.0 ✅
  Thursday:  restart, pulls   → gets v1.2 💥
  
You: "But I didn't deploy anything!"
```

**With pinned versions:**

```bash
# Production Deployment History

Monday:    payment-service:2.3.1 → Running fine ✅
Tuesday:   payment-service:2.3.2 → Bug found ❌
Tuesday:   payment-service:2.3.1 → Rollback ✅

# Clear history, easy rollback, reproducible
```

**Why it matters:** Reproducibility. The same command should produce the same result, always.

---

## The 4 Non-Negotiables

If you remember nothing else, remember these:

```
┌─────────────────────────────────────────────────────────────┐
│           NEVER DEPLOY WITHOUT THESE 4                      │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. RESOURCE LIMITS                                         │
│     --memory="512m" --cpus="1.0"                           │
│     → Prevents one container from killing host              │
│                                                             │
│  2. RESTART POLICY                                          │
│     --restart unless-stopped                                │
│     → Self-healing at 3 AM                                  │
│                                                             │
│  3. PINNED VERSION                                          │
│     myapp:1.2.3 (NOT :latest)                              │
│     → Reproducible deployments                              │
│                                                             │
│  4. HEALTH CHECK                                            │
│     --health-cmd="curl -f localhost/health || exit 1"      │
│     → Detect zombie containers                              │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## The Complete Production Command

```bash
docker run -d \
  --name payment-service-prod \
  -p 8080:8080 \
  -e SPRING_PROFILES_ACTIVE=production \
  -e DB_HOST=postgres.cluster.internal \
  -e DB_PASSWORD=${DB_PASSWORD} \
  -v /data/payments/logs:/app/logs \
  -v /data/payments/config:/app/config:ro \
  --memory="512m" \
  --cpus="1.0" \
  --restart unless-stopped \
  --log-opt max-size=10m \
  --log-opt max-file=3 \
  --health-cmd="curl -f http://localhost:8080/health || exit 1" \
  --health-interval=30s \
  --health-timeout=5s \
  --health-retries=3 \
  --read-only \
  --tmpfs /tmp:rw,noexec,nosuid,size=128m \
  --user=1000:1000 \
  myregistry.com/payment-service:1.2.3
```

---

## Quick Reference

| Category | Flag | Purpose |
|----------|------|---------|
| **Mode** | `-d` | Run in background |
| **Identity** | `--name` | Human-readable name |
| **Network** | `-p` | Port mapping |
| **Config** | `-e` | Environment variables |
| **Data** | `-v` | Persistent storage |
| **Resources** | `--memory` | Memory limit |
| **Resources** | `--cpus` | CPU limit |
| **Resilience** | `--restart` | Auto-restart policy |
| **Logs** | `--log-opt` | Log rotation |
| **Health** | `--health-*` | Application health check |
| **Security** | `--read-only` | Immutable filesystem |
| **Security** | `--tmpfs` | Safe temp storage |
| **Security** | `--user` | Non-root user |
| **Image** | `:version` | Pinned version tag |

---

## Conclusion

The gap between `docker run myapp` and a production-ready command is filled with lessons learned from production incidents:

- **Memory limits** exist because of OOM kills at 3 AM
- **Restart policies** exist because services crashed and stayed down
- **Health checks** exist because running ≠ working
- **Version pinning** exists because `:latest` broke production unexpectedly
- **Security flags** exist because attackers found ways in

Every flag in the production command is there because someone, somewhere, learned the hard way.

**The architect's job:** Learn from others' incidents, not your own.

---

*Next in the series: Kubernetes Production Deployment - From Docker to K8s*

---

**Author:** System Design Interview Series  
**Day:** 26 of 50  
**Topic:** Docker Production Commands
