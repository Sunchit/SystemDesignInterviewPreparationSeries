# Consistent Hashing: The Resharding Strategy That Powers Amazon, Discord & Netflix
### Day 28 of 50 - System Design Interview Preparation Series

**By Sunchit Dudeja**

---

## 🎯 Welcome to Day 28!

Yesterday, we explored Two-Phase Commit for distributed transactions. Today, we tackle one of the **most elegant algorithms** in distributed systems — **Consistent Hashing**. It's the secret behind how companies like Amazon, Discord, and Netflix scale to billions of requests without catastrophic data migrations.

> Consistent hashing isn't just a hashing technique. It's the difference between resharding in minutes vs. resharding in weeks.

---

## 🤯 The Problem That Breaks Naive Approaches

### The Interview Scenario

> "You have 4 Redis cache servers. You're using `server = hash(key) % 4` to distribute keys. Now you need to add a 5th server. What happens?"

### The Catastrophic Answer

```
┌─────────────────────────────────────────────────────────────────┐
│                 THE MODULO DISASTER                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   BEFORE (4 servers):           AFTER (5 servers):              │
│   ───────────────────           ────────────────────            │
│   Key "user:1" → hash % 4 = 1   Key "user:1" → hash % 5 = 1 ✓  │
│   Key "user:2" → hash % 4 = 2   Key "user:2" → hash % 5 = 2 ✓  │
│   Key "user:3" → hash % 4 = 3   Key "user:3" → hash % 5 = 3 ✓  │
│   Key "user:4" → hash % 4 = 0   Key "user:4" → hash % 5 = 4 ✗  │
│   Key "user:5" → hash % 4 = 1   Key "user:5" → hash % 5 = 0 ✗  │
│   Key "user:6" → hash % 4 = 2   Key "user:6" → hash % 5 = 1 ✗  │
│   Key "user:7" → hash % 4 = 3   Key "user:7" → hash % 5 = 2 ✗  │
│   Key "user:8" → hash % 4 = 0   Key "user:8" → hash % 5 = 3 ✗  │
│                                                                  │
│   ❌ Result: 75-80% of ALL keys now map to WRONG servers!       │
│   ❌ Cache miss rate: 0% → 80% (temporary cache failure)        │
│   ❌ Database overwhelmed with cache misses                      │
│   ❌ Potential cascading failure                                 │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### The Math Behind the Disaster

```python
# Modulo Hashing: Keys that change servers
def modulo_migration_percentage(old_servers, new_servers):
    """
    When changing from N to N+1 servers:
    Approximately (N-1)/N keys must move
    """
    return (old_servers - 1) / old_servers * 100

# Examples:
# 4 → 5 servers: (4-1)/4 = 75% keys move
# 10 → 11 servers: (10-1)/10 = 90% keys move
# 100 → 101 servers: (100-1)/100 = 99% keys move!

# This is UNACCEPTABLE at scale!
```

---

## 🍩 THE DONUT ANALOGY (Perfect for Interviews)

Imagine a circular donut with sprinkles placed around it:

```
┌─────────────────────────────────────────────────────────────────┐
│                  THE DONUT ANALOGY                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   🍩 The donut = A circular hash space (0° to 360°)             │
│   🔵 Sprinkles = Servers placed at positions on the ring        │
│   🔑 Keys = Data items that land somewhere on the donut         │
│                                                                  │
│   RULE: Each key goes to the NEXT sprinkle clockwise            │
│                                                                  │
│   When you add a NEW sprinkle:                                   │
│   • Only keys between the new sprinkle and next one move        │
│   • All other keys stay exactly where they are                  │
│   • Like adding one sprinkle to your donut!                     │
│                                                                  │
│   Result: Instead of reorganizing the whole donut,              │
│   you only adjust a small slice!                                 │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 🔄 HOW CONSISTENT HASHING WORKS

### Step 1: Create the Hash Ring

```
                        0° (same as 360°)
                            │
                            │
                    ┌───────┴───────┐
                   /                 \
                  /                   \
               315°                   45°
                │                       │
                │     HASH RING         │
                │    (0° to 360°)       │
                │                       │
               270°                   90°
                  \                   /
                   \                 /
                    └───────┬───────┘
                            │
                          180°

    The ring represents all possible hash values (0 to 2^32 - 1)
    mapped to a circle (0° to 360°)
```

### Step 2: Place Servers on the Ring

```
                          0°
                          │
                     D ●──┴──● A
                      \      /
                       \    /   Server A @ 45°
                        \  /    Server B @ 135°
                   ──────\/─────── Server C @ 225°
                         /\       Server D @ 315°
                        /  \
                       /    \
                      /      \
                     C ●────● B
                          │
                         180°
```

### Step 3: Assign Keys to Servers (Clockwise Rule)

```
┌─────────────────────────────────────────────────────────────────┐
│                  KEY ASSIGNMENT RULE                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   1. Hash the key to get a position on the ring                 │
│   2. Walk CLOCKWISE from that position                           │
│   3. First server you encounter = Owner of that key             │
│                                                                  │
│   Example:                                                       │
│   ─────────────────────────────────────────────────────         │
│   Key "user:123" → hash() → position 30°                        │
│   Walk clockwise → first server is A @ 45°                      │
│   Therefore: "user:123" is stored on Server A                   │
│                                                                  │
│   Key "order:456" → hash() → position 100°                      │
│   Walk clockwise → first server is B @ 135°                     │
│   Therefore: "order:456" is stored on Server B                  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 🎯 BEFORE AND AFTER: THE MAGIC OF MINIMAL MOVEMENT

### BEFORE: 4 Servers

```
                          0°
                     K5 ◇   
                    (350°)  
               D ●──────────● A (45°)
              (315°)    ◇ K1    
                │      (30°)  
          K4 ◇  │             │
         (270°) │             │  ◇ K2 (100°)
                │             │
               ─┼─────────────┼─
                │             │
                │             │
                │             │
               C ●───────────● B (135°)
              (225°)         
                    ◇ K3     
                   (180°)    
                             

    Key Distribution:
    ┌─────────────────────────────────────────┐
    │  K1 (30°)  → Server A (next @ 45°)     │
    │  K2 (100°) → Server B (next @ 135°)    │
    │  K3 (180°) → Server C (next @ 225°)    │
    │  K4 (270°) → Server D (next @ 315°)    │
    │  K5 (350°) → Server A (wraps to 45°)   │
    └─────────────────────────────────────────┘
```

### AFTER: Adding Server E at 180°

```
                          0°
                     K5 ◇   
                    (350°)  
               D ●──────────● A (45°)
              (315°)    ◇ K1    
                │      (30°)  
          K4 ◇  │             │
         (270°) │             │  ◇ K2 (100°)
                │             │
               ─┼─────────────┼─
                │             │
                │             │
                │             │
               C ●─────● E ──● B (135°)
              (225°)  (180°)
                       ◇ K3  ← ONLY THIS KEY MOVES!
                      (180°)    

    Key Distribution (AFTER):
    ┌──────────────────────────────────────────────────┐
    │  K1 (30°)  → Server A  ✓ UNCHANGED              │
    │  K2 (100°) → Server B  ✓ UNCHANGED              │
    │  K3 (180°) → Server E  ⚡ MOVED! (was C, now E) │
    │  K4 (270°) → Server D  ✓ UNCHANGED              │
    │  K5 (350°) → Server A  ✓ UNCHANGED              │
    └──────────────────────────────────────────────────┘
    
    🎯 RESULT: Only 1 out of 5 keys moved (20%)!
```

---

## 📊 THE DRAMATIC COMPARISON

### Data Migration: Modulo vs Consistent Hashing

| Scenario | Modulo Hashing | Consistent Hashing |
|----------|----------------|-------------------|
| 4 → 5 servers | **75% keys move** | **~20% keys move** |
| 10 → 11 servers | **90% keys move** | **~9% keys move** |
| 100 → 101 servers | **99% keys move** | **~1% keys move** |
| 1000 → 1001 servers | **99.9% keys move** | **~0.1% keys move** |

### The Math

```
┌─────────────────────────────────────────────────────────────────┐
│              MATHEMATICAL COMPARISON                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  MODULO HASHING:                                                 │
│  ────────────────                                                │
│  Keys moved = (N - 1) / N                                        │
│  As N grows, this approaches 100%!                               │
│                                                                  │
│  CONSISTENT HASHING:                                             │
│  ───────────────────                                             │
│  Keys moved = K / N                                              │
│  Where K = total keys, N = number of servers                    │
│  Only 1/N of keys move (the keys between new server and next)   │
│                                                                  │
│  EXAMPLE (Adding 1 server to 100):                               │
│  • Modulo: 99% of 1 billion keys = 990 million keys move        │
│  • Consistent: 1% of 1 billion keys = 10 million keys move      │
│  • SAVINGS: 980 million fewer key migrations!                    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 🚨 THE LOAD IMBALANCE PROBLEM

### The Issue with Basic Consistent Hashing

```
┌─────────────────────────────────────────────────────────────────┐
│              THE IMBALANCE PROBLEM                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  With only 4 physical servers, their positions on the ring      │
│  might not be evenly distributed:                                │
│                                                                  │
│           0°                                                     │
│           │                                                      │
│      A ●──┼─────────────────────────────────────● B              │
│     (10°) │                                    (350°)            │
│           │                                                      │
│           │      ← HUGE EMPTY SPACE!                             │
│           │         (Server A handles 340° of keys!)             │
│           │                                                      │
│      D ●──┴──● C                                                 │
│     (190°)  (180°)                                               │
│                                                                  │
│  Result: Server A overloaded, others underutilized               │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### The Solution: Virtual Nodes

```
┌─────────────────────────────────────────────────────────────────┐
│              VIRTUAL NODES (VNODES)                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Instead of 1 position per server, create 100-200 "virtual"     │
│  positions per server:                                           │
│                                                                  │
│  Physical Server A → Virtual nodes: A0, A1, A2, ... A99         │
│  Physical Server B → Virtual nodes: B0, B1, B2, ... B99         │
│  Physical Server C → Virtual nodes: C0, C1, C2, ... C99         │
│  Physical Server D → Virtual nodes: D0, D1, D2, ... D99         │
│                                                                  │
│  Now the ring has 400 points instead of 4!                      │
│                                                                  │
│         0°                                                       │
│         │                                                        │
│    B47 ●┼● A12                                                   │
│   C23 ●─┼─● D88                                                  │
│    A67 ●┼● B03                                                   │
│   D12 ●─┼─● C99                                                  │
│    B89 ●┼● A34                                                   │
│         │                                                        │
│        ...                                                       │
│                                                                  │
│  Result: Near-perfect load distribution!                         │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### How Virtual Nodes Work

```python
def create_virtual_nodes(server_id, num_virtual_nodes=150):
    """
    Create multiple hash positions for a single physical server
    """
    positions = []
    for i in range(num_virtual_nodes):
        # Hash "ServerA-0", "ServerA-1", etc.
        virtual_key = f"{server_id}-{i}"
        position = hash(virtual_key) % 360
        positions.append((position, server_id))
    return positions

# Result: Each physical server has 150 points on the ring
# Keys are distributed more evenly across all servers
```

### Impact of Virtual Nodes

| Virtual Nodes | Load Distribution | Memory Overhead |
|---------------|-------------------|-----------------|
| 1 per server | Very uneven (±50%) | Minimal |
| 50 per server | Moderate (±20%) | Low |
| 150 per server | Good (±5%) | Medium |
| 500 per server | Excellent (±1%) | Higher |

> **Recommendation:** 100-200 virtual nodes per physical server balances load distribution with memory efficiency.

---

## 💻 COMPLETE IMPLEMENTATION

### Python Implementation

```python
import hashlib
from bisect import bisect_left, insort_left
from collections import defaultdict

class ConsistentHashRing:
    """
    Production-ready consistent hashing implementation
    with virtual nodes for load balancing.
    """
    
    def __init__(self, virtual_nodes=150):
        self.virtual_nodes = virtual_nodes
        self.ring = []           # Sorted list of (hash_value, server)
        self.hash_to_server = {} # hash_value -> server mapping
        self.servers = set()     # Physical servers
    
    def _hash(self, key: str) -> int:
        """
        Generate consistent hash using MD5 (or SHA256 for production)
        """
        return int(hashlib.md5(key.encode()).hexdigest(), 16)
    
    def add_server(self, server: str) -> list:
        """
        Add a server with virtual nodes.
        Returns list of keys that need to be migrated TO this server.
        """
        if server in self.servers:
            return []
        
        self.servers.add(server)
        keys_to_migrate = []
        
        for i in range(self.virtual_nodes):
            virtual_key = f"{server}:{i}"
            hash_value = self._hash(virtual_key)
            
            # Find which server currently owns this range
            if self.ring:
                old_owner = self._get_server_for_hash(hash_value)
                keys_to_migrate.append((hash_value, old_owner, server))
            
            # Add to ring
            insort_left(self.ring, (hash_value, server))
            self.hash_to_server[hash_value] = server
        
        return keys_to_migrate
    
    def remove_server(self, server: str) -> list:
        """
        Remove a server and its virtual nodes.
        Returns list of keys that need to be migrated FROM this server.
        """
        if server not in self.servers:
            return []
        
        self.servers.remove(server)
        keys_to_migrate = []
        
        # Remove all virtual nodes for this server
        new_ring = []
        for hash_value, srv in self.ring:
            if srv == server:
                # This server's keys go to next server in ring
                new_owner = self._get_next_server(hash_value, exclude=server)
                keys_to_migrate.append((hash_value, server, new_owner))
                del self.hash_to_server[hash_value]
            else:
                new_ring.append((hash_value, srv))
        
        self.ring = new_ring
        return keys_to_migrate
    
    def get_server(self, key: str) -> str:
        """
        Get the server responsible for a given key.
        O(log n) lookup using binary search.
        """
        if not self.ring:
            raise Exception("No servers in ring")
        
        hash_value = self._hash(key)
        return self._get_server_for_hash(hash_value)
    
    def _get_server_for_hash(self, hash_value: int) -> str:
        """
        Find server for a hash value using binary search.
        """
        # Binary search for first server with hash >= hash_value
        idx = bisect_left(self.ring, (hash_value,))
        
        # Wrap around if we're past the end
        if idx >= len(self.ring):
            idx = 0
        
        return self.ring[idx][1]
    
    def _get_next_server(self, hash_value: int, exclude: str = None) -> str:
        """
        Get next server in ring, optionally excluding one.
        """
        idx = bisect_left(self.ring, (hash_value,))
        
        for i in range(len(self.ring)):
            check_idx = (idx + i) % len(self.ring)
            if self.ring[check_idx][1] != exclude:
                return self.ring[check_idx][1]
        
        return None
    
    def get_distribution(self) -> dict:
        """
        Calculate what percentage of ring each server owns.
        Useful for monitoring load distribution.
        """
        if not self.ring:
            return {}
        
        distribution = defaultdict(int)
        sorted_ring = sorted(self.ring)
        
        for i in range(len(sorted_ring)):
            current_hash, server = sorted_ring[i]
            if i == 0:
                # First server also covers wrap-around
                prev_hash = sorted_ring[-1][0]
                range_size = (2**128 - prev_hash) + current_hash
            else:
                prev_hash = sorted_ring[i-1][0]
                range_size = current_hash - prev_hash
            
            distribution[server] += range_size
        
        # Convert to percentages
        total = sum(distribution.values())
        return {s: (v/total)*100 for s, v in distribution.items()}


# Usage Example
ring = ConsistentHashRing(virtual_nodes=150)

# Add servers
ring.add_server("redis-1.cluster.local")
ring.add_server("redis-2.cluster.local")
ring.add_server("redis-3.cluster.local")
ring.add_server("redis-4.cluster.local")

# Route keys to servers
user_server = ring.get_server("user:12345")
order_server = ring.get_server("order:67890")
session_server = ring.get_server("session:abc123")

print(f"user:12345 → {user_server}")
print(f"order:67890 → {order_server}")
print(f"session:abc123 → {session_server}")

# Check load distribution
distribution = ring.get_distribution()
for server, percentage in distribution.items():
    print(f"{server}: {percentage:.2f}%")

# Add new server (minimal migration)
migrations = ring.add_server("redis-5.cluster.local")
print(f"\nAdding redis-5: {len(migrations)} hash ranges need migration")
```

### Java Implementation (Spring Boot)

```java
import java.security.MessageDigest;
import java.util.*;

public class ConsistentHashRing<T> {
    
    private final int virtualNodes;
    private final SortedMap<Long, T> ring = new TreeMap<>();
    private final Set<T> physicalNodes = new HashSet<>();
    
    public ConsistentHashRing(int virtualNodes) {
        this.virtualNodes = virtualNodes;
    }
    
    public void addNode(T node) {
        physicalNodes.add(node);
        for (int i = 0; i < virtualNodes; i++) {
            long hash = hash(node.toString() + ":" + i);
            ring.put(hash, node);
        }
    }
    
    public void removeNode(T node) {
        physicalNodes.remove(node);
        for (int i = 0; i < virtualNodes; i++) {
            long hash = hash(node.toString() + ":" + i);
            ring.remove(hash);
        }
    }
    
    public T getNode(String key) {
        if (ring.isEmpty()) {
            return null;
        }
        
        long hash = hash(key);
        
        // Find first node with hash >= key hash
        SortedMap<Long, T> tailMap = ring.tailMap(hash);
        Long nodeHash = tailMap.isEmpty() ? ring.firstKey() : tailMap.firstKey();
        
        return ring.get(nodeHash);
    }
    
    private long hash(String key) {
        try {
            MessageDigest md = MessageDigest.getInstance("MD5");
            byte[] digest = md.digest(key.getBytes());
            return ((long)(digest[0] & 0xFF) << 24) |
                   ((long)(digest[1] & 0xFF) << 16) |
                   ((long)(digest[2] & 0xFF) << 8) |
                   ((long)(digest[3] & 0xFF));
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
    }
    
    public Map<T, Double> getDistribution() {
        Map<T, Long> counts = new HashMap<>();
        long prev = 0;
        
        for (Map.Entry<Long, T> entry : ring.entrySet()) {
            long range = entry.getKey() - prev;
            counts.merge(entry.getValue(), range, Long::sum);
            prev = entry.getKey();
        }
        
        long total = counts.values().stream().mapToLong(Long::longValue).sum();
        Map<T, Double> distribution = new HashMap<>();
        for (Map.Entry<T, Long> entry : counts.entrySet()) {
            distribution.put(entry.getKey(), 
                (entry.getValue() * 100.0) / total);
        }
        return distribution;
    }
}
```

---

## 🏢 REAL-WORLD IMPLEMENTATIONS

### Amazon DynamoDB

```
┌─────────────────────────────────────────────────────────────────┐
│              AMAZON DYNAMODB                                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  DynamoDB uses consistent hashing for:                          │
│  • Partition key routing                                         │
│  • Automatic data distribution                                   │
│  • Seamless capacity scaling                                     │
│                                                                  │
│  Implementation:                                                 │
│  • 2^128 hash space                                              │
│  • Virtual nodes for each storage node                           │
│  • Automatic rebalancing on node add/remove                     │
│                                                                  │
│  Scale:                                                          │
│  • Handles 10+ trillion requests per day                        │
│  • Zero-downtime scaling                                         │
│  • Sub-millisecond latency                                       │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Discord

```
┌─────────────────────────────────────────────────────────────────┐
│              DISCORD MESSAGE ROUTING                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Discord uses consistent hashing for:                           │
│  • Routing messages to correct servers                          │
│  • Guild (server) data distribution                             │
│  • Voice channel routing                                         │
│                                                                  │
│  Why it matters:                                                 │
│  • 150+ million monthly active users                            │
│  • Adding servers doesn't disrupt active voice calls            │
│  • Guild data stays together (locality)                         │
│                                                                  │
│  Key insight:                                                    │
│  • Hash on guild_id, not message_id                             │
│  • All messages for a guild go to same shard                    │
│  • Enables efficient queries within a guild                      │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Apache Cassandra

```
┌─────────────────────────────────────────────────────────────────┐
│              CASSANDRA PARTITIONING                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Cassandra's architecture is BUILT on consistent hashing:       │
│                                                                  │
│  Token Ring:                                                     │
│  • Each node owns a range of tokens                             │
│  • Partition key → Murmur3 hash → Token → Node                  │
│  • Virtual nodes (vnodes) enabled by default                    │
│                                                                  │
│  Configuration:                                                  │
│  ```yaml                                                         │
│  # cassandra.yaml                                                │
│  num_tokens: 256  # Virtual nodes per physical node             │
│  partitioner: org.apache.cassandra.dht.Murmur3Partitioner       │
│  ```                                                             │
│                                                                  │
│  Operations:                                                     │
│  • nodetool status  - Shows token ranges per node               │
│  • nodetool ring    - Shows full token ring                     │
│  • Adding node: Cassandra automatically streams data            │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Redis Cluster

```
┌─────────────────────────────────────────────────────────────────┐
│              REDIS CLUSTER                                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Redis Cluster uses a variant called "hash slots":               │
│                                                                  │
│  • 16,384 hash slots (not continuous ring)                      │
│  • Key → CRC16(key) % 16384 → Slot → Node                       │
│  • Each node owns a subset of slots                              │
│                                                                  │
│  Example:                                                        │
│  ─────────────────────────────────────────────                  │
│  Node A: Slots 0-5460                                            │
│  Node B: Slots 5461-10922                                        │
│  Node C: Slots 10923-16383                                       │
│                                                                  │
│  "user:123" → CRC16 → 7892 → Node B                             │
│                                                                  │
│  Resharding: Move slots between nodes                            │
│  ```bash                                                         │
│  redis-cli --cluster reshard 127.0.0.1:7000                     │
│  ```                                                             │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## ⚖️ TRADE-OFFS AND CONSIDERATIONS

### When to Use Consistent Hashing

| Use Case | Why Consistent Hashing |
|----------|------------------------|
| Distributed caches | Minimize cache invalidation on scaling |
| Database sharding | Reduce data migration overhead |
| Load balancers | Sticky sessions with resilience |
| CDN routing | Geographic + hash-based distribution |
| Message queues | Partition assignment |

### When NOT to Use Consistent Hashing

| Scenario | Better Alternative |
|----------|-------------------|
| Small, fixed cluster | Simple modulo (easier) |
| Range queries needed | Range-based partitioning |
| Data locality required | Directory-based routing |
| Frequent rebalancing | Dynamic partitioning |

### Complexity Trade-offs

```
┌─────────────────────────────────────────────────────────────────┐
│              COMPLEXITY COMPARISON                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  MODULO HASHING:                                                 │
│  ────────────────                                                │
│  • Implementation: Simple (one line of code)                    │
│  • Lookup: O(1)                                                  │
│  • Add server: O(K) - must move most keys                       │
│  • Memory: O(N) - just server list                              │
│                                                                  │
│  CONSISTENT HASHING:                                             │
│  ───────────────────                                             │
│  • Implementation: Moderate (ring + binary search)              │
│  • Lookup: O(log V) where V = total virtual nodes               │
│  • Add server: O(K/N) - move only 1/N keys                      │
│  • Memory: O(N × V) - virtual nodes storage                     │
│                                                                  │
│  Trade-off: More complex, but scales gracefully                 │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 🛠️ HANDLING EDGE CASES

### 1. Server Failure Detection

```python
class ConsistentHashRingWithHealthCheck:
    def __init__(self):
        self.ring = ConsistentHashRing()
        self.failed_servers = set()
    
    def get_server(self, key: str, retries: int = 3) -> str:
        """
        Get server with automatic failover
        """
        for attempt in range(retries):
            server = self.ring.get_server(key)
            
            if server not in self.failed_servers:
                if self._health_check(server):
                    return server
                else:
                    self.failed_servers.add(server)
            
            # Try next server in ring
            key = f"{key}:{attempt}"
        
        raise Exception("All servers failed")
    
    def _health_check(self, server: str) -> bool:
        # Implementation: TCP ping, HTTP health endpoint, etc.
        pass
```

### 2. Replication

```python
def get_replica_servers(self, key: str, num_replicas: int = 3) -> list:
    """
    Get multiple servers for replication.
    Walk the ring and collect N distinct physical servers.
    """
    servers = []
    hash_value = self._hash(key)
    idx = bisect_left(self.ring, (hash_value,))
    
    seen_physical = set()
    
    while len(servers) < num_replicas and len(seen_physical) < len(self.servers):
        idx = idx % len(self.ring)
        _, server = self.ring[idx]
        
        if server not in seen_physical:
            servers.append(server)
            seen_physical.add(server)
        
        idx += 1
    
    return servers

# Usage: Write to primary, replicate to secondaries
replicas = ring.get_replica_servers("user:123", num_replicas=3)
# Returns: ["redis-2", "redis-4", "redis-1"]
```

### 3. Weighted Servers

```python
def add_server_weighted(self, server: str, weight: int):
    """
    Add server with custom weight (more virtual nodes = more keys)
    
    Example: 
    - Powerful server: weight=200 (gets more keys)
    - Weak server: weight=50 (gets fewer keys)
    """
    for i in range(weight):
        virtual_key = f"{server}:{i}"
        hash_value = self._hash(virtual_key)
        insort_left(self.ring, (hash_value, server))
```

---

## ❓ Interview Practice

### Question 1:
> "Design a distributed cache that can scale from 4 to 100 servers without downtime."

**Answer:**
> "I'd use consistent hashing with virtual nodes. Each cache server gets 150+ virtual positions on the hash ring to ensure even distribution. When we add a server, only 1/N of the keys need to migrate to the new server. For the migration itself, I'd implement a two-phase approach: first, the new server starts handling new writes for its range while the old server still serves reads. Then we background-migrate the existing keys. This ensures zero-downtime scaling. I'd also implement replication by walking the ring to find the next 2 distinct physical servers for each key, giving us fault tolerance."

### Question 2:
> "What happens if your consistent hash ring becomes unbalanced?"

**Answer:**
> "With basic consistent hashing using just N physical positions, the distribution can be very uneven — one server might handle 40% of keys while another handles 10%. The solution is virtual nodes: instead of one position per server, each server gets 100-200 positions spread across the ring. This statistically balances the load. With 150 virtual nodes per server, variance drops to under 5%. The trade-off is memory overhead for storing the larger ring, but it's negligible compared to the data itself. I'd monitor distribution using metrics and alert if any server deviates more than 10% from expected load."

### Question 3:
> "How would you handle server failures in a consistent hash ring?"

**Answer:**
> "Three strategies: First, health checks — continuously monitor servers and maintain a 'failed servers' set. When routing a key, if the primary server is failed, walk the ring clockwise to the next healthy server. Second, replication — for each key, identify N replica servers by walking the ring and collecting distinct physical servers. If primary fails, a replica can serve reads immediately. Third, during recovery, the failed server's keys are temporarily handled by its clockwise neighbor; when it recovers, only its original range migrates back. This gives us fault tolerance without full re-hashing."

---

## 🔗 Connecting to Previous Days

| Day | Concept | How It Connects |
|-----|---------|-----------------|
| Day 8 | Load Balancing | Consistent hashing enables sticky sessions with resilience |
| Day 15 | Redis | Redis Cluster uses hash slots, a variant of consistent hashing |
| Day 22 | HLD Architecture | Consistent hashing is core to the data layer |
| Day 23 | Database Selection | Sharding strategy depends on consistent hashing |

---

## ✅ Day 28 Action Items

1. **Implement a basic ring** — Build a consistent hash ring in your preferred language
2. **Visualize distribution** — Plot how keys distribute across servers with varying virtual nodes
3. **Test migration** — Simulate adding/removing servers and count key movements
4. **Explore Cassandra** — Run `nodetool ring` on a Cassandra cluster to see token distribution
5. **Benchmark lookups** — Compare O(log n) ring lookup vs O(1) modulo performance

---

## 💡 Key Takeaways

| Takeaway | Why It Matters |
|----------|----------------|
| Minimize data movement | Adding a server should move 1/N keys, not N-1/N |
| Virtual nodes balance load | Raw consistent hashing can be uneven |
| Ring enables resilience | Walk clockwise to find next server on failure |
| Locality matters | Hash on parent ID (guild_id) not child ID (message_id) |
| It's about resharding | Not indexing — resharding data across nodes |

---

## 🚀 Key Architect Principles

| Principle | What It Means |
|-----------|---------------|
| **Design for change** | Clusters will grow; plan for minimal-disruption scaling |
| **Trade complexity for scalability** | O(log n) lookup is worth smooth resharding |
| **Use virtual nodes** | Always — never use raw physical positions |
| **Monitor distribution** | Alert if any server deviates >10% from expected |
| **Plan for failure** | Replication + clockwise failover |

---

## 💡 Key Takeaway

> **Junior: "Just use key % num_servers. It's simple."**
>
> **Architect: "That works until you need to add a server. Then 80% of your cache becomes invalid, your database gets hammered, and you're in an outage. Consistent hashing with virtual nodes means adding a server only moves 1/N of the data. It's the difference between a planned 5-minute scaling operation and an unplanned 2-hour incident."**

The difference? Understanding that **systems change**. Servers fail. Traffic grows. What seems simple today becomes a scaling nightmare tomorrow. Consistent hashing is the algorithm that respects this reality.

---

*— Sunchit Dudeja*  
*Day 28 of 50: System Design Interview Preparation Series*

---

> 🎯 **Interview Edge:** When discussing caching or database sharding, always mention consistent hashing: "I'd use consistent hashing with ~150 virtual nodes per server. This ensures that adding or removing a server only migrates 1/N of the keys, enabling zero-downtime scaling. Companies like Amazon DynamoDB, Discord, and Cassandra all use this approach."

> 🛣️ **The Highway Analogy:** "Consistent hashing is like adding a new exit on a circular highway — only the drivers between that exit and the next one need to change their route. Everyone else keeps driving exactly as before."

---

## 🔗 Resources

- **Excalidraw Diagram:** Available in course materials (consistent-hashing-explained.excalidraw)
- **Original Paper:** "Consistent Hashing and Random Trees" by Karger et al. (1997)
- **Implementation:** github.com/redis/redis (cluster.c for hash slots)
- **Cassandra Docs:** cassandra.apache.org/doc/latest/cassandra/architecture/dynamo.html

---

> 💡 **Tomorrow (Day 29):** We'll explore **Rate Limiting** — how do you protect your APIs from abuse while ensuring fair access? Token buckets, sliding windows, and distributed rate limiting at scale.
