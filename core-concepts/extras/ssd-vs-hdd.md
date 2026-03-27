# 💾 Understanding SSD vs HDD in System Design

> *"Fast storage makes systems feel instant. Slow storage makes everything else look broken."*

**⏱ Reading time:** ~10 minutes &nbsp;|&nbsp; **📍 Series:** Extras &nbsp;|&nbsp; **Referenced from:** [Throughput & Latency](../non-functional-system-characteristics/throughputAndLatency.md)

---

## Table of Contents

1. [What SSD and HDD Mean](#1-what-ssd-and-hdd-mean)
2. [How They Actually Work](#2-how-they-actually-work)
3. [The Core Differences That Matter](#3-the-core-differences-that-matter)
4. [Why Storage Speed Matters More Than You Think](#4-why-storage-speed-matters-more-than-you-think)
5. [When to Use SSD vs HDD](#5-when-to-use-ssd-vs-hdd)
6. [A Real System Design Example](#6-a-real-system-design-example)
7. [What Good Looks Like](#7-what-good-looks-like)

---

## 1. What SSD and HDD Mean

**SSD = Solid-State Drive**  
**HDD = Hard Disk Drive**

Both are used to **store data permanently** (databases, files, logs, backups). The difference is *how* they store and retrieve that data.

- **SSD** → fast, modern, no moving parts  
- **HDD** → slower, mechanical, spinning disks  

Think of it like this:
SSD = instant access (like opening an app on your phone)
HDD = physical movement (like finding a file in a cabinet)

---

## 2. How They Actually Work

### HDD (Hard Disk Drive)

An HDD is a **mechanical device**:

- Spinning magnetic disks (platters)
- A moving arm that reads/writes data
- Data is stored in specific physical locations

[Disk spinning] → [Arm moves] → [Reads data]


This means:
- It takes time to move to the right spot
- Performance depends on *where* the data is

---

### SSD (Solid-State Drive)

An SSD is **purely electronic**:

- Uses flash memory (like your phone)
- No moving parts
- Can access any data instantly

[Direct access to memory cell]


This means:
- No physical delay
- Very fast random access

---

## 3. The Core Differences That Matter

| Feature        | SSD                         | HDD                        |
|----------------|-----------------------------|----------------------------|
| Speed          | Very fast                   | Slow                       |
| Latency        | Microseconds                | Milliseconds               |
| Random access  | Excellent                   | Poor                       |
| Sequential I/O | Fast                        | Decent                     |
| Durability     | High (no moving parts)      | Lower (mechanical failure) |
| Cost per GB    | Higher                      | Lower                      |
| Capacity       | Smaller (typically)         | Very large                 |

---

### The key insight:

> **SSDs are fast at *everything***  
> **HDDs are only okay at *sequential* reads/writes**

---

## 4. Why Storage Speed Matters More Than You Think

Storage is often the **hidden bottleneck** in systems.

Even if your CPU and network are fast:
Request → CPU (fast) → Disk (slow) → everything waits


### Example: Database query

- With SSD:
  - Query returns in ~5–10 ms
- With HDD:
  - Same query might take ~50–200 ms

Now multiply that across thousands of requests:

- Your **p99 latency explodes**
- Your system *feels* slow

---

### Where this shows up

Slow storage causes:

- High latency spikes (p99 issues)
- Slow page loads
- API delays
- Timeouts under load

This is why storage decisions directly affect:
- **Latency**
- **Throughput**
- **SLA compliance**

---

## 5. When to Use SSD vs HDD

### Use SSD when:

You care about **speed and user experience**

Examples:
- Databases (PostgreSQL, MySQL)
- Search systems
- User-facing APIs
- Caching layers
- Real-time systems

If users are waiting → use SSD


---

### Use HDD when:

You care about **cost and large storage**

Examples:
- Backups
- Logs
- Archival data
- Cold storage (rarely accessed data)

If data is rarely accessed → use HDD


---

### Rule of thumb

- Hot data → SSD
- Cold data → HDD

---

## 6. A Real System Design Example

### Scenario: Video platform (like YouTube)

You need to store:

1. User data (profiles, comments)
2. Video metadata
3. Actual video files

---

### Design choice

**SSD:**
- User database
- Metadata
- Indexes

Why?
Users expect instant responses (low latency)


---

**HDD (or object storage backed by HDD):**
- Raw video files

Why?
Huge size + not accessed constantly


---

### What happens if you choose wrong?

If you store your database on HDD:
- p50 = 20ms
- p99 = 800ms ← disk seeks killing you


If you store everything on SSD:
- Great performance ✔
- Very expensive ✖


Good system design is about **balancing both**.

---

## 7. What Good Looks Like

### Modern systems rarely choose just one

They use **tiered storage**:
- Layer 1 (fastest): Cache (RAM)
- Layer 2: SSD (hot data)
- Layer 3: HDD / object storage (cold data)


---

### Healthy architecture

Recent / frequent data → SSD
Old / infrequent data → HDD


---

### Today’s reality (important)

- **SSDs are the default for most production systems**
- **HDDs are still widely used for bulk storage**

You will commonly see:

- SSD-backed databases ✔
- HDD-backed backups ✔
- Hybrid cloud storage ✔

---

### Decision checklist

Ask yourself:

1. **Is this user-facing?**  
   → Yes → SSD  

2. **How often is the data accessed?**  
   → Frequently → SSD  
   → Rarely → HDD  

3. **Is latency critical?**  
   → Yes → SSD  

4. **Is cost a major constraint?**  
   → Yes → HDD (or hybrid)  

---

### Final intuition

> SSD improves **speed**  
> HDD improves **cost efficiency**

Good systems use both — intentionally.

---

*↩ Back to [Throughput & Latency](../non-functional-system-characteristics/throughputAndLatency.md)*

---

<sub>Part of the <a href="../../README.md">System Design Foundations</a> study guide series — Extras.</sub>

