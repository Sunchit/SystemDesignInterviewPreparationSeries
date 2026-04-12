# Bloom Filter: Instagram Username Availability (No False Negatives)
### Day 43 of 50 - System Design Interview Preparation Series

**By Sunchit Dudeja**

---

## 🎯 The Core Property

A **Bloom filter** has **no false negatives** (for the “is this username already taken?” set):

| If the filter says… | Meaning |
|---------------------|---------|
| **“Not in set”** | The username is **definitely not** in the set of taken names → safe to treat as **available** without hitting the DB first. |
| **“Might be in set”** | Could be a **false positive** (actually free, but bits collided) → **must confirm** with the **authoritative database**. |

You **must never** use a Bloom filter alone to say “available” when the name is actually taken. The trick: **only trust “not in set”** for the fast path; **“maybe”** always goes to the DB.

> **📐 Excalidraw:** Open [bloom-filter-instagram-username.excalidraw](./bloom-filter-instagram-username.excalidraw) at [excalidraw.com](https://excalidraw.com) — dark background, flow: User → Bloom → fast vs DB path.

---

## 📱 Instagram-Style Example: Checking @username

When someone types a handle at sign-up, the product needs a **fast** answer and **cannot** claim “available” if the name is already taken.

### Step-by-step

| Step | Check | Bloom output | Ground truth | OK? |
|------|--------|--------------|--------------|-----|
| 1 | `@john_doe` | **NOT in set** | Available | Yes — **instant** “available” |
| 2 | `@jane_doe` | **NOT in set** | Available | Yes |
| 3 | `@alex_smith` | **MIGHT be in set** | Actually available (false positive) | Yes — **slow path** DB confirms |
| 4 | `@taken_user` | **MIGHT be in set** | Taken | Yes — DB confirms taken |

**Why no false negatives?** If **any** bit position for the hashes is **0**, the name was **never** inserted—so it cannot be in the set. Bits only go **0 → 1** on insert; we never flip **1 → 0** on insert (standard Bloom). So we never “clear” evidence that a name was added.

```python
# Simplified lookup idea (k hash positions)
def might_be_taken(username: str, bit_array) -> bool:
    for h in (hash1(username), hash2(username), hash3(username)):
        if bit_array[h % len(bit_array)] == 0:
            return False   # definitely NOT in set → safe "available" fast path
    return True          # might be taken → check database
```

---

## ⚡ Why It’s So Much Faster Than the Database

| | **Bloom filter** | **Database (index lookup)** |
|---|------------------|-------------------------------|
| **Where** | RAM (often **CPU cache–friendly**) | Storage + query engine |
| **Work** | **k** hashes + **k** bit reads | B-tree / index walk, buffers, possibly I/O |
| **Time (order of magnitude)** | **~1–2 µs** per check | **~1–5 ms** per round-trip |
| **Ratio** | 1× | Often **~10³× slower** |

**µs** = **microsecond**: **µ** is Greek *mu* (SI prefix **micro-** = 10⁻⁶).  
**1 µs** = one millionth of a second. **1 ms** = **1000 µs**.  
So **1 ms ≈ 1000× longer** than **1 µs**—that’s why “RAM + bits” feels instant and a remote DB hit feels “slow” for each keystroke.

**Analogy:** The Bloom filter is a **sticky note** listing what is **definitely not** in the library; checking the note is **instant**. Walking to the stacks (**DB**) is still needed when the note says **“maybe.”**

---

## 🧠 End-to-End Flow (Conceptual)

1. **All taken usernames** (or a large sample) are **added** to the Bloom filter when built/refreshed.  
2. **Lookup:** `might_be_taken(name)`  
   - **False** → show **available** immediately (**no DB**).  
   - **True** → **query DB** for authoritative check (handles false positives).  
3. **Registration** still uses **DB uniqueness constraint** as the **final** gate—Bloom is only an **optimization**, not the source of truth.

**Result:** Most **creative, unused** names exit on the **fast path**; **rare** false positives pay one **DB** query—never wrong “available” from Bloom alone.

---

## 📊 Summary Table

| Bloom says | Meaning | Action | False negative? |
|------------|---------|--------|-------------------|
| **NOT in set** | Definitely not in taken set | Show **available**; skip DB | **No** |
| **MIGHT in set** | Could be taken or FP | **Query DB** | **No** (DB decides) |

---

## ⚠️ Interview Pitfalls

- **False positives** are OK; **false negatives** are not (for this “taken set” use case).  
- **Delete** is hard in a classic Bloom filter—**rebuild** or use a **counting** variant if usernames are released.  
- **Size / hash count** tune **FP rate** vs **memory**.

---

## 🎯 The 10-Second Takeaway

> *A Bloom filter can say “**definitely not taken**” at **RAM speed** with **no false negatives** on that answer. If it says “**maybe**,” the **database** decides. That’s how apps like Instagram keep username checks **snappy** for almost every random string—without lying about availability.*

---

## 🔗 Connecting to Previous Days

| Day | Concept | How It Connects |
|-----|---------|-----------------|
| Day 9 | Bloom filters & cache penetration | Same data structure; different question (“in cache” vs “taken username”) |
| Day 19 | Google email uniqueness | Bloom-style pre-check before heavy work |
| Day 38 | Primary keys | Username as natural unique key |

---

## ✅ Day 43 Action Items

1. Write **one** sentence: why **no false negatives** matter for usernames but **false positives** are acceptable.  
2. Convert **5 ms** to **µs** and compare to **2 µs** Bloom check.  
3. Sketch **API**: Bloom **miss** → 200 available; Bloom **hit** → DB lookup.

---

*— Sunchit Dudeja*  
*Day 43 of 50: System Design Interview Preparation Series*
