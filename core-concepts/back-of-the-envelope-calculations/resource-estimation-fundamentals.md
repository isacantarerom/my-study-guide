# 🔢 Resource Estimation Fundamentals

> *"You don't need to know the exact number. You need to know if it's thousands, millions, or billions — because those require completely different systems."*

**⏱ Reading time:** ~11 minutes &nbsp;|&nbsp; **📍 Series:** Grokking System Design &nbsp;|&nbsp; **🗂 Topic #1:** Back-of-the-Envelope Calculations

---

## Table of Contents

1. [Why These Numbers Matter](#1-why-these-numbers-matter)
2. [The Power of Two — Data Size Reference](#2-the-power-of-two--data-size-reference)
3. [Latency Numbers Every Engineer Should Know](#3-latency-numbers-every-engineer-should-know)
4. [Traffic Estimation Building Blocks](#4-traffic-estimation-building-blocks)
5. [Storage Estimation Building Blocks](#5-storage-estimation-building-blocks)
6. [Bandwidth Estimation Building Blocks](#6-bandwidth-estimation-building-blocks)
7. [Compute Estimation Building Blocks](#7-compute-estimation-building-blocks)
8. [The Numbers Worth Memorizing](#8-the-numbers-worth-memorizing)
9. [Self-Check](#9-self-check)
10. [References](#10-references)

---

## 1. Why These Numbers Matter

Every system design starts with requirements. But requirements like "handle a lot of traffic" or "store user data" are too vague to design from. Before picking components, you need rough answers to:

- How many requests per second will this handle?
- How much storage will it need in a year?
- How much bandwidth moves through this system daily?
- How many servers do we actually need?

These answers don't need to be exact. They need to be in the right ballpark — within an order of magnitude. A system designed for 1,000 requests/second looks very different from one designed for 1,000,000 requests/second. Getting that distinction right is the whole point.

The building blocks in this guide are the reference numbers that make estimation possible. Once you know them, you can combine them to estimate almost anything.

---

## 2. The Power of Two — Data Size Reference

Computers store everything in bits and bytes. Memory, storage, and bandwidth are all measured in these units. Knowing the scale of each prefix is fundamental.

```
Unit        | Abbreviation | Size               | Rough equivalent
────────────┼──────────────┼────────────────────┼─────────────────────────
Bit         | b            | 1 bit              | On or off
Byte        | B            | 8 bits             | One character of text
Kilobyte    | KB           | 1,000 bytes        | A short text message
Megabyte    | MB           | 1,000,000 bytes    | A photo, a minute of audio
Gigabyte    | GB           | 1,000,000,000 B    | A movie, ~1000 photos
Terabyte    | TB           | 1,000 GB           | ~500 hours of HD video
Petabyte    | PB           | 1,000 TB           | All US academic libraries
Exabyte     | EB           | 1,000 PB           | All internet traffic per day
```

> 💡 **Practical shortcut:** In system design estimations, we typically use powers of 10 (1KB = 1,000 bytes) rather than powers of 2 (1KB = 1,024 bytes). The difference doesn't matter at the order-of-magnitude level we're working at.

**Common data sizes to internalize:**

| Thing | Approximate Size |
|-------|-----------------|
| ASCII character | 1 byte |
| Integer (int32) | 4 bytes |
| Long integer (int64) | 8 bytes |
| UUID / GUID | 16 bytes |
| Short tweet / text message | ~140 bytes |
| Average webpage | ~2 MB |
| High-quality photo (JPEG) | ~3-5 MB |
| 1 minute of MP3 audio | ~1 MB |
| 1 minute of HD video | ~100-150 MB |
| 1 minute of 4K video | ~400-600 MB |

These sizes are the atoms of storage estimation. Every storage calculation starts by asking: how big is one unit of the thing I'm storing?

---

## 3. Latency Numbers Every Engineer Should Know

These are the latency reference numbers made famous by Jeff Dean at Google. You don't need to memorize the exact values — you need to internalize the **order of magnitude differences** between them, because those differences explain why caching, SSDs, and co-location matter so much.

```
Operation                           Latency         Relative to RAM
────────────────────────────────────────────────────────────────────
L1 cache reference                  ~0.5 ns         1×
Branch misprediction                ~5 ns           10×
L2 cache reference                  ~7 ns           14×
Mutex lock/unlock                   ~25 ns          50×
Main memory (RAM) access            ~100 ns         200×
──────────────────────────────────────────────────────────────────
Compress 1KB with Snappy            ~3,000 ns       6,000×
Send 1KB over 1Gbps network         ~10,000 ns      20,000×
Read 4KB randomly from SSD          ~150,000 ns     300,000×
Read 1MB sequentially from memory   ~250,000 ns     500,000×
Round trip within same datacenter   ~500,000 ns     1,000,000×
Read 1MB sequentially from SSD      ~1,000,000 ns   2,000,000×
Disk seek (spinning HDD)            ~10,000,000 ns  20,000,000×
Read 1MB sequentially from disk     ~20,000,000 ns  40,000,000×
Send packet CA → Netherlands → CA   ~150,000,000 ns 300,000,000×
```

**The key takeaways — what this table is really saying:**

- **Memory is 1,000× faster than SSD.** This is why caching works — moving data from disk to RAM is a massive speedup.
- **SSD is 100× faster than spinning disk.** This is why databases on SSDs are dramatically faster.
- **A datacenter round-trip is 500,000 ns (~0.5ms).** Every network hop between services costs this — which is why minimizing hops matters.
- **Cross-continental network is 150ms.** This is the physical latency floor you can't engineer away — only work around with CDNs and geographic distribution.

When you see a system design choice — "use Redis as a cache" or "co-locate these two services" — these numbers are the reason why. The decision is always about moving operations up this table toward the faster end.

---

## 4. Traffic Estimation Building Blocks

Traffic estimation almost always starts with daily active users (DAU) and what each user does per day.

### The Core Conversion: Requests Per Day → Requests Per Second

```
Requests per second (RPS) = Total requests per day / Seconds per day

Seconds per day = 24 hours × 60 minutes × 60 seconds = 86,400 seconds
                ≈ 100,000 seconds   (good enough approximation)

So: RPS ≈ Total daily requests / 100,000
```

**Example:**
- 10 million DAU, each making 10 requests/day
- Total daily requests = 100 million
- Average RPS = 100,000,000 / 100,000 = **1,000 RPS**

### Peak Traffic

Systems don't receive traffic evenly. There are busy hours and quiet hours. A common rule of thumb:

```
Peak RPS ≈ Average RPS × 2 to 3×

For most consumer apps, assume:
- 80% of daily traffic happens in 20% of the day (Pareto principle)
- Peak hour handles ~5× the average hourly traffic
```

Design for peak, not average. A system that handles average load but falls over at peak is a system that falls over regularly.

### Read/Write Ratio

Most systems have significantly more reads than writes. Common ratios:

| System type | Read/Write ratio |
|-------------|-----------------|
| Social media feed | 100:1 |
| Search engine | 1000:1 |
| E-commerce product pages | 50:1 |
| Messaging app | 1:1 (roughly symmetric) |
| IoT sensor ingestion | 1:100 (write-heavy) |

Knowing the ratio tells you where to focus. A read-heavy system needs caching and read replicas. A write-heavy system needs write-optimized storage and async processing.

---

## 5. Storage Estimation Building Blocks

Storage estimation follows a simple formula:

```
Total storage = Size per record × Number of records × Replication factor

Where:
  Size per record  = sum of all fields for one unit of data
  Number of records = users × records per user (or events per day × days)
  Replication factor = typically 3× for durability
```

### Estimating Record Size

Break each record into its fields and add up the sizes:

```
User profile record:
  user_id       (int64)    8 bytes
  username      (varchar)  ~20 bytes average
  email         (varchar)  ~30 bytes average
  created_at    (int64)    8 bytes
  profile_photo (URL)      ~100 bytes
  ───────────────────────────────
  Total                    ~166 bytes ≈ 200 bytes (round up)
```

Always round up. Estimates that are slightly high are safer than ones that are slightly low.

### Storage Growth Over Time

```
Daily storage growth = New records per day × Size per record

Example: 1 million new users/day, 200 bytes each
Daily growth = 1,000,000 × 200 = 200,000,000 bytes = 200 MB/day
Annual growth = 200 MB × 365 ≈ 73 GB/year (before replication)
With 3× replication: ~220 GB/year
```

### Media Storage is a Different Beast

Text records are tiny. Media (photos, videos, audio) is massive and dominates storage in any system that handles it.

```
Instagram-like system:
  50 million photos uploaded per day
  Average photo size: 3 MB
  Daily storage: 50,000,000 × 3 MB = 150,000,000 MB = 150 TB/day

  Before replication. This is why Instagram uses object storage (S3)
  and CDNs rather than databases for media files.
```

If your system handles media, storage becomes the dominant design constraint very quickly.

---

## 6. Bandwidth Estimation Building Blocks

Bandwidth is the rate of data transfer — how many bytes per second flow through your system.

```
Bandwidth = Data per request × Requests per second

Incoming bandwidth (ingress) = write traffic × average write payload size
Outgoing bandwidth (egress)  = read traffic × average read response size
```

**Example — a video streaming service:**
```
Users: 10 million DAU
Each user watches: 30 minutes of video per day
Video bitrate: 5 Mbps (HD)

Total video minutes per day = 10,000,000 × 30 = 300,000,000 minutes
Total video hours per day = 5,000,000 hours

Peak concurrent viewers (assuming 10% of DAU at peak):
  1,000,000 concurrent streams × 5 Mbps = 5,000,000 Mbps = 5 Tbps

That's 5 terabits per second of outbound bandwidth.
This is why Netflix uses a CDN — no single origin infrastructure
can deliver 5 Tbps, but thousands of globally distributed
CDN edge nodes can.
```

Bandwidth estimates often reveal why certain architectural choices are necessary, not just convenient.

---

## 7. Compute Estimation Building Blocks

Once you know your RPS, estimating server count requires knowing what one server can handle.

```
Servers needed = Peak RPS / RPS per server

With N+1 redundancy:
Servers needed = (Peak RPS / RPS per server) + 1
```

**What can one server handle?**

This varies enormously by workload, but common starting points:

| Workload type | RPS per commodity server |
|---------------|--------------------------|
| Simple API (mostly DB reads, with caching) | 1,000–10,000 RPS |
| CPU-intensive API (encryption, compression) | 100–500 RPS |
| Database server (PostgreSQL, MySQL) | 1,000–5,000 queries/sec |
| Cache server (Redis) | 100,000–1,000,000 ops/sec |
| Message broker (Kafka) | 100,000–1,000,000 msgs/sec |

**Example:**
```
Target: handle 50,000 RPS at peak
Assumption: each app server handles 2,000 RPS

Servers needed = 50,000 / 2,000 = 25 servers
With N+1 headroom: ~30 servers

With 3 availability zones: 10 servers per zone
```

Always think about how servers are distributed across zones and regions, not just the total count.

---

## 8. The Numbers Worth Memorizing

You don't need to memorize everything above — but these are the ones that come up constantly and are worth internalizing:

```
TIME
  Seconds in a day:         ~86,400 ≈ 100,000
  Seconds in a month:       ~2,500,000 ≈ 2.5 million
  Seconds in a year:        ~31,500,000 ≈ 31.5 million

LATENCY (order of magnitude)
  RAM access:               ~100 ns
  SSD read:                 ~100 µs (1,000× slower than RAM)
  HDD read:                 ~10 ms  (100,000× slower than RAM)
  Same-datacenter RTT:      ~0.5 ms
  Cross-continent RTT:      ~150 ms

DATA SIZES (common objects)
  Char / byte:              1 byte
  Integer:                  4 bytes
  Long / timestamp:         8 bytes
  UUID:                     16 bytes
  Average tweet:            ~300 bytes
  Web page:                 ~2 MB
  Photo (compressed):       ~3 MB
  1 min HD video:           ~100 MB

THROUGHPUT (rough per server)
  Web server (cached):      ~10,000 RPS
  Database (relational):    ~5,000 QPS
  Redis:                    ~500,000 ops/sec
  Kafka:                    ~500,000 msgs/sec
```

---

## 9. Self-Check

1. A system has 5 million DAU, each sending 20 requests per day. What is the average RPS? What should you design peak capacity for?
2. You're storing user posts. Each post has: user_id (8 bytes), post_id (8 bytes), text (avg 200 bytes), created_at (8 bytes), likes_count (4 bytes). The system gets 2 million new posts per day. How much storage is needed per day? Per year? (Before replication)
3. Why is it important to know the read/write ratio of your system before designing it?
4. A photo sharing app has 10 million DAU. Each user uploads 2 photos per day (avg 3MB each) and views 50 photos per day. What is the daily ingress (upload) bandwidth? What is the daily egress (download) bandwidth?
5. Based on the latency numbers, why is caching in RAM so much more effective than caching on SSD?
6. You need to handle 100,000 RPS. Each server can handle 5,000 RPS. How many servers do you need, accounting for N+1 redundancy and deployment across 3 availability zones?

---

## 10. References

| Resource | Why it's worth it |
|----------|-------------------|
| 📊 [Jeff Dean — Latency Numbers Every Programmer Should Know](https://gist.github.com/jboner/2841832) | The original reference table — updated versions exist, the orders of magnitude remain true |
| 📘 [Designing Data-Intensive Applications — Ch. 1 (Kleppmann)](https://dataintensive.net) | Discusses what numbers matter and how to reason about system scale |
| 📬 [ByteByteGo — Back of the Envelope Estimation](https://bytebytego.com) | Visual worked examples of estimation for common systems |
| 📝 [System Design Primer — Estimation](https://github.com/donnemartin/system-design-primer#back-of-the-envelope-calculations) | Solid reference with common estimation patterns |

---

*⬅️ Previous: [Back-of-the-Envelope Overview](BackOfTheEnvelopeCalculations.md) &nbsp;|&nbsp; ➡️ Next: [Estimation in Practice](estimation-in-practice.md)*

---

<sub>Part of the <a href="../../README.md">System Design Foundations</a> study guide series — Back-of-the-Envelope Calculations.</sub>