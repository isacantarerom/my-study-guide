# 🛠️ Estimation in Practice

> *"The purpose of an estimate is not to be right. It's to be useful — to reveal whether your design is in the right universe."*

**⏱ Reading time:** ~12 minutes &nbsp;|&nbsp; **📍 Series:** Grokking System Design &nbsp;|&nbsp; **🗂 Topic #2:** Back-of-the-Envelope Calculations

---

## Table of Contents

1. [The Method](#1-the-method)
2. [Step 1 — Define the Scale](#2-step-1--define-the-scale)
3. [Step 2 — Estimate Traffic](#3-step-2--estimate-traffic)
4. [Step 3 — Estimate Storage](#4-step-3--estimate-storage)
5. [Step 4 — Estimate Bandwidth](#5-step-4--estimate-bandwidth)
6. [Step 5 — Estimate Compute](#6-step-5--estimate-compute)
7. [What the Numbers Tell You](#7-what-the-numbers-tell-you)
8. [Common Mistakes in Estimation](#8-common-mistakes-in-estimation)
9. [Worked Example: Twitter-Like Feed System](#9-worked-example-twitter-like-feed-system)
10. [Worked Example: YouTube-Like Video Platform](#10-worked-example-youtube-like-video-platform)
11. [Self-Check](#11-self-check)
12. [References](#12-references)

---

## 1. The Method

Back-of-the-envelope estimation follows a repeatable five-step method. The same method works for any system — from a URL shortener to a global video platform. What changes are the numbers; the structure stays the same.

```
Step 1: Define the scale
        → How many users? How active are they?
                │
                ▼
Step 2: Estimate traffic
        → Requests per second (average and peak)
                │
                ▼
Step 3: Estimate storage
        → How much data, growing how fast?
                │
                ▼
Step 4: Estimate bandwidth
        → Data in and out per second
                │
                ▼
Step 5: Estimate compute
        → How many servers?
```

The outputs of each step feed into the next. Traffic tells you storage growth rate. Storage tells you backup bandwidth. Bandwidth tells you CDN requirements. Compute tells you your server fleet size.

**Two rules that apply throughout:**

**Round aggressively.** 86,400 seconds → 100,000 seconds. 3.2 MB → 3 MB. You're estimating, not accounting. Precision beyond one significant figure is false confidence.

**State your assumptions.** Every estimate rests on assumptions. State them explicitly — "assuming 10% of users are active at peak," "assuming average photo size of 3MB," "assuming 80% cache hit rate." Stated assumptions can be challenged and refined. Hidden assumptions can't.

---

## 2. Step 1 — Define the Scale

Before any math, establish the user base and behavior. These numbers anchor everything else.

**Questions to answer:**

```
- How many total registered users?
- How many daily active users (DAU)?
- What does one user do in a typical session?
  → How many reads? How many writes?
  → How much data do they produce?
  → How much data do they consume?
```

**Example — a photo sharing app:**
```
Total users:      500 million registered
DAU:              100 million (20% of registered users active daily)
Per user per day:
  - Uploads:      2 photos
  - Views:        50 photos
  - Likes:        30 interactions
  - Comments:     5 interactions
```

This gives you the atoms. Everything else is multiplying these numbers by 100 million.

**A useful calibration table:**

| Scale | DAU | Example |
|-------|-----|---------|
| Small startup | 10,000 | Early-stage product |
| Growing startup | 1,000,000 | Established niche app |
| Mid-size product | 10,000,000 | Successful consumer app |
| Large platform | 100,000,000 | Major social platform |
| Hyperscale | 1,000,000,000+ | WhatsApp, YouTube |

---

## 3. Step 2 — Estimate Traffic

**Formula:**
```
Total daily requests = DAU × requests per user per day
Average RPS          = Total daily requests / 100,000
Peak RPS             = Average RPS × 3 (conservative peak multiplier)
```

**Photo sharing app continued:**
```
Daily requests:
  Photo uploads:   100M × 2     = 200M uploads/day
  Photo views:     100M × 50    = 5,000M views/day
  Likes:           100M × 30    = 3,000M likes/day
  Comments:        100M × 5     = 500M comments/day

  Total reads:     ~8,000M/day (views + some likes/comments read)
  Total writes:    ~3,700M/day (uploads + likes + comments)

  Read/write ratio ≈ 2:1 (more balanced than you might expect
                           because uploads generate many writes)

Average RPS:
  Read RPS:  8,000,000,000 / 100,000 = 80,000 RPS
  Write RPS: 3,700,000,000 / 100,000 = 37,000 RPS

Peak RPS (3× average):
  Read peak:  ~240,000 RPS
  Write peak: ~110,000 RPS
```

**What this tells you:** You need a system capable of handling ~240K read requests and ~110K write requests per second at peak. This immediately suggests caching is essential for reads (no database can handle 240K QPS without it) and async processing for writes.

---

## 4. Step 3 — Estimate Storage

**Formula:**
```
Storage per day  = New records per day × Size per record
Storage per year = Storage per day × 365
Total storage    = Storage per year × years to retain × replication factor
```

**Photo sharing app continued:**

First, the metadata (structured data in a database):
```
Photo metadata record:
  photo_id    (int64):   8 bytes
  user_id     (int64):   8 bytes
  caption     (text):    ~200 bytes average
  location    (2×float): 16 bytes
  created_at  (int64):   8 bytes
  likes_count (int32):   4 bytes
  ─────────────────────────────
  Total:                ~244 bytes ≈ 250 bytes

200 million uploads/day × 250 bytes = 50 GB/day of metadata
× 365 days = ~18 TB/year of metadata
× 3 (replication) = ~54 TB/year
```

Then, the actual photos (object storage):
```
200 million uploads/day × 3 MB average = 600,000,000 MB/day
                                       = 600 TB/day

That's 600 terabytes of raw photo data every single day.
× 365 = ~219 PB/year (petabytes — before replication)
× 3 (replication) = ~657 PB/year ≈ ~0.66 exabytes/year
```

**What this tells you:** The metadata is manageable — 54 TB/year fits in a well-designed database cluster. The photos are a completely different problem — 600 TB/day requires object storage (like S3), not a database. This is why Instagram uses S3 for photos and only stores metadata in their relational database. The estimation revealed the architectural necessity.

---

## 5. Step 4 — Estimate Bandwidth

**Formula:**
```
Ingress bandwidth = Write RPS × Average payload size per write
Egress bandwidth  = Read RPS × Average response size per read
```

**Photo sharing app continued:**
```
Ingress (uploads):
  200 million photos/day uploaded
  200,000,000 × 3 MB = 600,000,000 MB/day
  ÷ 86,400 seconds = ~6,944 MB/second ≈ 7 GB/second ingress

Peak ingress (3×): ~21 GB/second

Egress (views):
  5 billion photo views/day
  At what resolution? Assume feed thumbnails: 200KB average
  5,000,000,000 × 0.2 MB = 1,000,000,000 MB/day
  ÷ 86,400 = ~11,574 MB/second ≈ 12 GB/second egress

Peak egress (3×): ~35 GB/second
```

**What this tells you:** 35 GB/second of outbound bandwidth at peak is not something one data center delivers directly to users — the latency would be terrible for users far away and the bandwidth cost would be enormous. This is why photo/video platforms use CDNs as a core part of their architecture, not as an afterthought. Again, the estimation reveals the design requirement.

---

## 6. Step 5 — Estimate Compute

**Formula:**
```
Servers for reads  = Peak read RPS  / RPS per server
Servers for writes = Peak write RPS / RPS per server
Add N+1 headroom, distribute across availability zones
```

**Photo sharing app continued:**
```
Assumptions:
  Read server (with caching): handles ~10,000 RPS
  Write server (DB writes):   handles ~2,000 RPS

Read servers:
  240,000 peak read RPS / 10,000 = 24 servers
  With 20% headroom: ~30 servers
  Across 3 AZs: 10 per AZ

Write servers:
  110,000 peak write RPS / 2,000 = 55 servers
  With 20% headroom: ~66 servers
  Across 3 AZs: 22 per AZ

Database servers:
  Write RPS of 37,000 average → need sharding or write queue
  1 primary + 3 read replicas per shard
  Estimate 5-10 shards depending on data model

Cache servers (Redis):
  Targeting 80% cache hit rate on reads
  Remaining 20% go to DB: 0.2 × 240,000 = 48,000 QPS to DB
  Redis handles ~500,000 ops/sec per node
  3-5 Redis nodes handles the cache load comfortably

CDN:
  35 GB/second peak egress → fully offloaded to CDN
  Origin only serves cache misses (~5% of requests)
  Origin egress: 0.05 × 35 = ~1.75 GB/second
```

---

## 7. What the Numbers Tell You

This is the most important part of the exercise. The numbers aren't the output — the **insights they force** are the output.

After working through the photo sharing estimation:

| Insight | Design decision forced |
|---------|----------------------|
| Photos are 600 TB/day | Use object storage (S3), not a database |
| Read RPS is 240K | Caching is mandatory, not optional |
| Egress is 35 GB/sec peak | CDN is required, not an enhancement |
| Metadata is 50 GB/day | Relational DB works, but plan for sharding |
| Write RPS is 110K | Async write processing via message queue |

Without the estimation, you might have designed a system that uses a single PostgreSQL database for everything and wonders why it falls over on day one. With the estimation, you know exactly why certain architectural components are necessary before you've written a line of code.

This is the real value of back-of-the-envelope calculations: not the numbers themselves, but the design decisions they make obvious.

---

## 8. Common Mistakes in Estimation

### Forgetting Peak vs Average
The most common mistake. A system designed for average load will fail at peak — and real systems always have peaks. Always multiply your average by at least 2-3× and design for that.

### Ignoring Replication
Storage estimates that forget the 3× replication factor underestimate by 3×. That's the difference between "we need 200 TB" and "we need 600 TB." At $20/TB/month, that's a meaningful budget difference.

### Treating All Data the Same
Text records and media files require completely different storage architectures. A 200-byte metadata record goes in a relational database. A 3MB photo goes in object storage. Conflating them leads to architectural mistakes that are expensive to undo.

### Underestimating Read Traffic
Read traffic is almost always much higher than intuition suggests, especially for social/content systems. Every piece of content created is viewed many times. A single tweet might be read 10,000 times. Design for the read load, not the write load.

### Over-Precision
Writing "86,400 seconds per day" when you could write "~100,000" or spending time calculating to three significant figures. Estimation should take minutes, not hours. Round, approximate, and move on.

---

## 9. Worked Example: Twitter-Like Feed System

**Scale assumption:** 300 million DAU

**Step 1: Define behavior**
```
Per user per day:
  - Posts (tweets):  3 writes
  - Feed reads:      200 reads
  - Likes:           20 writes
  - Follows checked: varies
```

**Step 2: Traffic**
```
Daily writes:
  Tweets:  300M × 3  = 900M tweets/day
  Likes:   300M × 20 = 6,000M likes/day
  Total writes: ~7,000M/day → 70,000 write RPS average

Daily reads:
  Feed:    300M × 200 = 60,000M reads/day
  Total reads: 60,000M/day → 600,000 read RPS average

Peak (3×):
  Write peak: ~210,000 RPS
  Read peak:  ~1,800,000 RPS

Read/write ratio: ~8:1 heavily read-dominant
→ Caching and read replicas are essential
```

**Step 3: Storage**
```
Tweet record:
  tweet_id    8 bytes
  user_id     8 bytes
  text        ~280 bytes (Twitter's character limit)
  created_at  8 bytes
  ────────────────────
  Total:      ~304 bytes ≈ 300 bytes

900 million tweets/day × 300 bytes = 270 GB/day
× 365 = ~98 TB/year
× 3 replication = ~295 TB/year

5-year retention: ~1.5 PB total tweet storage
```

**Step 4: Bandwidth**
```
Ingress (new tweets):
  900M tweets/day × 300 bytes = 270,000 MB/day
  ÷ 86,400 = ~3 MB/second ingress (tiny — text is small)

Egress (feed reads):
  60B reads/day × 5KB average feed response = 300,000,000 GB/day
  ÷ 86,400 = ~3,472 GB/second egress

This is why Twitter's CDN and caching story is so important.
3.5 TB/second of outbound data is a massive infrastructure challenge.
```

**Step 5: Compute**
```
App servers (reads, with heavy caching):
  1,800,000 peak read RPS / 10,000 per server = 180 servers
  With headroom: ~220 servers across 3 AZs

Write processing:
  210,000 peak write RPS
  Most writes go through a message queue (Kafka)
  Workers process asynchronously — fan-out to followers' feeds
  50-100 write worker servers

Cache layer:
  Redis, targeting 95%+ hit rate on hot content
  ~10 Redis nodes for the read cache
```

**Insights forced:**
- Feed reads at 1.8M RPS → caching is the entire architecture, not an add-on
- Egress at 3.5 TB/sec → CDN for all static assets, cache at every layer
- Fan-out to followers → async processing via Kafka (one tweet → notifications to potentially millions of followers can't be synchronous)

---

## 10. Worked Example: YouTube-Like Video Platform

**Scale assumption:** 2 billion DAU (YouTube's actual scale)

**Step 1: Define behavior**
```
Per user per day:
  - Videos watched:   5 (average ~8 minutes each)
  - Videos uploaded:  very few users upload (1 in 1000)
  - Comments/likes:   10 interactions
```

**Step 2: Traffic**
```
Video uploads:
  2B DAU × (1/1000) uploaders = 2,000,000 uploaders/day
  Average 1 video per uploader per day
  2,000,000 videos uploaded per day
  ÷ 86,400 = ~23 videos/second being uploaded

Video views:
  2B × 5 views/day = 10,000,000,000 views/day
  ÷ 86,400 = ~115,000 views/second

Peak views (3×): ~345,000 concurrent streams
```

**Step 3: Storage**
```
Per uploaded video:
  Raw upload: ~500 MB (average 10-minute video, uncompressed)
  After processing (multiple resolutions: 4K, 1080p, 720p, 360p):
    ~1-2 GB total stored per video

2,000,000 videos/day × 1.5 GB = 3,000,000 GB/day = 3 PB/day

× 365 = ~1 EB/year (exabyte)

This is why YouTube uses custom-built distributed storage
and processes videos asynchronously after upload.
```

**Step 4: Bandwidth**
```
Video streaming egress:
  345,000 concurrent streams × 5 Mbps (HD average)
  = 1,725,000 Mbps = 1.7 Tbps peak egress

This is why YouTube has CDN presence in virtually every
major ISP and data center globally — delivering 1.7 Tbps
requires being as close to users as possible.

Upload ingress:
  23 uploads/second × 500 MB = 11,500 MB/second = ~11 GB/second ingress
```

**Step 5: Compute**
```
Video processing (transcoding) is CPU-intensive:
  2,000,000 videos/day to transcode into 4 resolutions
  Assume 1 server transcodes 10 videos/hour
  2,000,000 / 24 hours = 83,333 videos/hour to process
  83,333 / 10 per server = ~8,333 transcoding servers
  (This explains why YouTube has massive compute for video processing)

Streaming servers:
  345,000 concurrent streams mostly served by CDN
  Origin servers handle cache misses only (~5%)
  ~17,250 concurrent origin streams
  Each server handles ~100 concurrent streams
  ~175 origin streaming servers
```

**Insights forced:**
- 3 PB/day storage → custom distributed storage systems, not off-the-shelf databases
- 1.7 Tbps egress → deep CDN integration is non-negotiable
- 8,333 transcoding servers → video processing is a massive compute workload, must be async
- Uploading and streaming are completely separate concerns that scale independently

---

## 11. Self-Check

1. What is the five-step estimation method, and why does each step feed into the next?
2. Why do you design for peak traffic rather than average traffic? What happens if you don't?
3. A messaging app has 50 million DAU. Each user sends 30 messages/day (avg 1KB each) and reads 200 messages/day (avg 1KB each). Calculate: average read RPS, average write RPS, daily storage growth, daily ingress bandwidth, daily egress bandwidth.
4. In the photo sharing estimation, why did the storage estimate reveal that a relational database alone is insufficient? What architectural decision does that force?
5. What is the difference between ingress and egress bandwidth? Which one is typically larger for a read-heavy consumer app, and why?
6. You estimate that your system needs to handle 500,000 peak RPS. Each server handles 5,000 RPS. You want N+1 redundancy across 3 availability zones. How many servers total, and how many per zone?
7. Why does the estimation exercise for YouTube reveal that video transcoding servers must be async? What would happen if transcoding were synchronous (blocking the upload response)?

---

## 12. References

| Resource | Why it's worth it |
|----------|-------------------|
| 📘 [Designing Data-Intensive Applications — Ch. 1 (Kleppmann)](https://dataintensive.net) | The best framing of what scale means and how to reason about it |
| 📬 [ByteByteGo — Back of the Envelope](https://bytebytego.com) | Visual worked examples for common systems — YouTube, Twitter, Instagram |
| 📝 [System Design Primer](https://github.com/donnemartin/system-design-primer#back-of-the-envelope-calculations) | Reference calculations and cheat sheet |
| 📊 [High Scalability Blog](http://highscalability.com) | Real architecture write-ups from companies at scale — useful for calibrating estimates |

---

*⬅️ Previous: [Resource Estimation Fundamentals](resource-estimation-fundamentals.md) &nbsp;|&nbsp; ➡️ Next Section: [Building Blocks](../building-blocks/BuildingBlocks.md)*

---

<sub>Part of the <a href="../../README.md">System Design Foundations</a> study guide series — Back-of-the-Envelope Calculations.</sub>