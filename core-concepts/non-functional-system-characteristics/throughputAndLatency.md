# ⚡ Throughput & Latency

> *"Fast for one request and fast for a million requests are two completely different problems."*

**⏱ Reading time:** ~12 minutes &nbsp;|&nbsp; **📍 Series:** Grokking System Design &nbsp;|&nbsp; **🗂 Topic #6:** Non-Functional Characteristics

---

## Table of Contents

1. [What Throughput and Latency Actually Mean](#1-what-throughput-and-latency-actually-mean)
2. [The Fundamental Tension Between Them](#2-the-fundamental-tension-between-them)
3. [Measuring Latency Correctly](#3-measuring-latency-correctly)
4. [What Causes Latency](#4-what-causes-latency)
5. [What Limits Throughput](#5-what-limits-throughput)
6. [How Systems Improve Latency](#6-how-systems-improve-latency)
7. [How Systems Improve Throughput](#7-how-systems-improve-throughput)
8. [The Knee of the Curve: Load and Performance](#8-the-knee-of-the-curve-load-and-performance)
9. [Throughput, Latency, and the Other Characteristics](#9-throughput-latency-and-the-other-characteristics)
10. [Worked Example: Designing for Performance](#10-worked-example-designing-for-performance)
11. [Self-Check](#11-self-check)
12. [References](#12-references)

---

## 1. What Throughput and Latency Actually Mean

These two words describe the same system from two different angles. Both matter. They measure different things and they're improved in different ways.

**Latency** is the time it takes to complete a single operation — the delay between sending a request and receiving a response.

```
Request sent ────────────────────────────► Response received
            ◄──────── Latency = 45ms ─────►
```

**Throughput** is the number of operations a system can complete per unit of time — the rate at which work gets done.

```
│ 1,000 requests/second handled │
│████████████████████████████████│ ← Throughput
Time ──────────────────────────────►
```

> 🏎️ **Analogy:** A highway. Latency is how long it takes one car to drive from one end to the other. Throughput is how many cars per hour the highway can move. You can have a very fast highway (low latency) that's also narrow and can only move a few cars at a time (low throughput). Or a wide but congested highway (high throughput potential, high latency from traffic). These are genuinely independent dimensions.

Both matter — but which one matters more depends entirely on what your system is doing. A surgical robot needs very low latency; a slight delay could cause harm. A bulk data processing pipeline needs high throughput; a single operation being a little slow doesn't matter as long as millions complete per hour.

---

## 2. The Fundamental Tension Between Them

Here's the uncomfortable truth: **optimizing for latency and optimizing for throughput often push in opposite directions.**

To minimize latency on a single request, you want to do as little work as possible on the critical path — process it immediately, don't batch it with others, don't wait for anything.

To maximize throughput, you often want to do the opposite — batch requests together, amortize fixed costs (like disk writes and network round-trips) across many operations, pipeline work so resources are always busy.

```
Optimizing for latency:               Optimizing for throughput:

Process each request immediately.     Batch requests together.
Don't wait for others.                Wait until batch is full.
Minimize work per request.            Amortize setup cost across many requests.

Result: each request is fast.         Result: each request waits a bit.
        but resources are idle                but total work done per second
        between requests.                     is much higher.
```

A classic example is **disk writes**. Writing to disk one record at a time is low latency per write (each write completes quickly) but terrible throughput (the disk spends most of its time seeking between writes). Batching many records and writing them together is higher latency per record (each record waits for the batch to fill) but dramatically higher throughput (the disk is continuously busy doing useful work).

This tension is why you have to know which dimension you're optimizing for before you start. There's no universally "better" approach — only the right tradeoff for your specific workload.

---

## 3. Measuring Latency Correctly

Most engineers' instinct is to measure latency as an average. This is almost always the wrong metric, and it can actively mislead you.

**The problem with averages:** a small number of very slow requests can be completely invisible in the average. If 99% of requests take 10ms and 1% take 10,000ms, the average might be around 110ms — which sounds fine. But 1% of requests at 10,000ms is catastrophic at any real scale.

The right way to measure latency is with **percentiles**:

```
Given 1,000 requests with these response times:

p50  (median)  = 12ms   → Half of requests completed in 12ms or less
p95            = 85ms   → 95% of requests completed in 85ms or less
p99            = 340ms  → 99% of requests completed in 340ms or less
p99.9          = 2,100ms → 99.9% completed in 2,100ms or less

The average might be ~25ms — hiding the 2,100ms tail completely.
```

**p99 and p99.9 are the numbers that matter most** in user-facing systems. These are your **tail latencies** — the experience of your worst-served users. And tail latencies matter more than you might think, for two reasons:

**Reason 1: At scale, tail latencies affect many users.**
At 1,000 requests/second, p99 latency affects 10 requests every second. Over an hour, that's 36,000 users who experienced that slow response. "It only affects 1% of users" sounds fine. At scale, 1% is still a lot of people.

**Reason 2: Tail latencies compound across dependencies.**
If a request touches 10 services and each has a 1% chance of being slow, the probability that at least one service is slow is 1 - (0.99)^10 = ~10%. What was a 1% tail for each service becomes a 10% tail for the end-to-end request. This is the latency version of the dependency chain problem from [Availability](availability.md).

---

## 4. What Causes Latency

Understanding the sources of latency is what lets you fix it. Latency comes from several places:

### Network Latency
The time for data to physically travel between machines. This has a hard physical lower bound — the speed of light in fiber optic cable is roughly 200,000 km/second. A round trip between New York and London is ~10,000km, giving a minimum latency of ~50ms just from physics. No amount of engineering eliminates this; it can only be reduced by moving servers closer to users (CDNs, regional deployments).

```
Speed of light in fiber: ~200,000 km/s
New York to London: ~5,500 km one way

Minimum one-way latency: 5,500 / 200,000 = ~27ms
Minimum round-trip:                         ~54ms

Real-world RTT NY→London: ~75-85ms
(extra time: routing, processing, transmission overhead)
```

### Disk I/O Latency
Reading from and writing to disk is orders of magnitude slower than reading from memory. An SSD read takes ~100 microseconds. A spinning disk read can take 5-10 milliseconds. RAM access takes ~100 nanoseconds.

```
RAM:          ~100 nanoseconds  (0.0001ms)
NVMe SSD:     ~100 microseconds (0.1ms)
SATA SSD:     ~500 microseconds (0.5ms)
Spinning disk: ~5-10ms
Network:       ~0.5ms (local) to 150ms+ (cross-continent)
```

This is why caching is so powerful as a latency reduction tool — moving data from disk to memory can be a 1,000x improvement.

### Queuing Latency
When a system is under load, requests wait in queues — at the load balancer, at the application server's thread pool, at the database connection pool. **Queuing latency is often the dominant source of high tail latencies**, because it's unpredictable and can spike suddenly when load increases.

A request that would normally take 10ms can take 500ms if it spends 490ms sitting in a queue waiting for a thread to become available.

### Processing Latency
The actual computation time — running business logic, executing queries, transforming data. This is often the smallest component of total latency in well-designed systems, and the easiest to optimize through better algorithms or code.

### Serialization/Deserialization
Converting data between formats (JSON encoding/decoding, Protobuf serialization) takes time. For high-frequency, small-payload services, this can be a meaningful fraction of total request time — which is one reason gRPC with Protobuf is faster than REST with JSON for internal services.

---

## 5. What Limits Throughput

Throughput has a ceiling determined by the bottleneck in your system. **Every system has exactly one bottleneck at any given time** — the resource that's most constrained. Adding capacity anywhere else won't improve throughput until you address the actual bottleneck.

This is a direct application of **Amdahl's Law**: the speedup from optimizing a part of a system is limited by how much of the total work that part represents.

Common throughput bottlenecks:

**CPU** — when processing is computationally intensive (encryption, compression, complex calculations). Solution: more cores, more servers, or more efficient algorithms.

**Memory** — when the working set of data exceeds available RAM, the system spills to disk. Throughput collapses. Solution: more RAM, smaller working sets, different data structures.

**Disk I/O** — when the system is reading or writing more data than the disk can handle. Common bottleneck for databases. Solution: faster disks (SSD), read replicas, caching, more efficient access patterns.

**Network bandwidth** — when the volume of data transferred saturates the network link. More common in data-intensive systems (video streaming, large file transfers). Solution: compression, more bandwidth, CDNs.

**Database connections** — a finite connection pool shared across many application servers. When all connections are in use, requests queue. Solution: connection pooling, read replicas, query optimization.

Finding your bottleneck requires measurement, not guessing. Profile first, optimize second.

---

## 6. How Systems Improve Latency

### Caching
Move data closer to where it's needed. We've covered caching as a scaling strategy and a reliability concern — here it's a latency tool. Serving from memory instead of disk, from a local cache instead of a remote service, from a CDN edge node instead of an origin server. Each step brings data physically closer to the requester, reducing the distance (and therefore time) data must travel.

### Reducing Network Hops
Every network round-trip between services adds latency. Strategies to reduce hops:

**Colocate dependent services** — services that call each other frequently should run in the same data center or availability zone, minimizing network distance between them.

**Denormalize data strategically** — instead of joining data across services at query time, pre-compute and store the assembled result. One read instead of many.

**Async for non-critical work** — move work that doesn't need to complete synchronously off the critical path. Send the confirmation email after returning the response, not before.

### Connection Pooling
Establishing a new database connection takes time — TCP handshake, authentication, setup. Connection pools maintain a set of pre-established connections that are reused across requests. The request skips the connection setup entirely and gets straight to the query.

### Choosing the Right Data Structures and Indexes
A database query that scans every row in a table to find one record is O(n). The same query using an index is O(log n). At a million rows, that's the difference between examining 1,000,000 rows and examining ~20. This is often the highest-leverage optimization available for database-bound systems.

---

## 7. How Systems Improve Throughput

### Horizontal Scaling
Add more instances to handle more requests in parallel. As we covered in [Scalability](scalability.md), this requires stateless services. With stateless services and a load balancer, throughput scales approximately linearly with the number of instances — double the servers, roughly double the throughput.

### Batching
Group multiple operations and process them together to amortize fixed overhead. A database that writes 1,000 records in one batch transaction is much faster than one that writes them one at a time, because the setup cost (acquiring locks, writing the transaction log, committing) is paid once for the whole batch.

### Asynchronous Processing
As covered in [Scalability](scalability.md) and [Fault Tolerance](faultTolerance.md), offloading non-critical work to queues keeps the synchronous request path lean. The application accepts work at the rate users submit it, and workers process the queue at whatever rate the backend can sustain. The queue absorbs the difference.

### Efficient Serialization
Replacing verbose formats (XML, JSON) with compact binary formats (Protobuf, Avro, MessagePack) reduces the amount of data to serialize, transmit, and deserialize per operation — directly improving throughput for network-bound services.

---

## 8. The Knee of the Curve: Load and Performance

One of the most important things to understand about throughput and latency is how they interact as load increases. The relationship isn't linear — it follows a characteristic curve.

```
Latency
  │
  │                                    ┌──── latency spikes (queuing dominates)
  │                                 ┌──┘
  │                             ┌───┘
  │                        ┌────┘
  │                   ┌────┘
  │              ─────┘  ← "knee of the curve"
  │         ─────
  │    ─────   (latency relatively flat here)
  └────────────────────────────────────────► Load (requests/second)
```

At low load, latency is flat — requests are handled immediately, queuing is minimal.

As load increases toward capacity, latency starts to rise — queuing latency adds up as resources become contended.

At the **knee of the curve** — typically around 60-70% of theoretical maximum capacity — latency starts rising sharply. Beyond this point, small increases in load cause large increases in latency.

At maximum throughput, latency becomes unbounded — requests queue indefinitely.

**The practical lesson:** you should plan to run your system at no more than 60-70% of its theoretical capacity during normal operation. That headroom is what keeps latency predictable and what gives you room to absorb load spikes (viral events, traffic surges, flash sales) without your latency blowing up.

This is also why performance testing matters. You need to know where your system's knee is before it's hit in production — because by the time users are experiencing bad latency, you're already past the knee and scaling up takes time you don't have.

---

## 9. Throughput, Latency, and the Other Characteristics

This guide closes the non-functional characteristics section, and it's worth seeing how performance connects to everything that came before:

**Availability** — high latency is a form of unavailability. A system that technically responds but takes 30 seconds per request is functionally unavailable. SLAs often include latency requirements alongside uptime requirements.

**Reliability** — reliability mechanisms (transactions, validation, replication before ack) add latency. Every ACID guarantee has a performance cost. The tradeoff is deliberate and context-dependent, as we covered in [Reliability](reliability.md).

**Scalability** — scaling is ultimately about maintaining acceptable latency and throughput as load grows. The entire [Scalability](scalability.md) guide is really about what to do when latency or throughput degrades at scale.

**Maintainability** — performance problems that are hard to diagnose are a maintainability problem. Good observability (traces showing where latency lives, metrics showing throughput and error rates) is what makes performance problems solvable rather than mysterious.

**Fault Tolerance** — fault tolerance mechanisms like circuit breakers exist partly to protect latency. A slow downstream that's not circuit-broken will drag your p99 latency up as requests pile up waiting for it.

None of these characteristics exist in isolation. Every design decision is a negotiation across all of them simultaneously, with different weights depending on what your system is doing and who it's serving.

---

## 10. Worked Example: Designing for Performance

A search API that needs to return results in under 100ms at p99, handling 50,000 requests/second.

**Breaking down the latency budget:**

```
Total budget: 100ms

Network (client to server, same region): ~5ms
Load balancer overhead:                  ~1ms
Application processing:                  ~10ms
Cache lookup (Redis):                    ~1ms  ← on cache hit
Database query (on cache miss):          ~20ms ← rare after warmup
Network back to client:                  ~5ms

Total (cache hit):  ~22ms  ✓ well within budget
Total (cache miss): ~41ms  ✓ still within budget
```

**Achieving 50,000 req/sec throughput:**

```
50,000 req/sec ÷ 500 req/sec per server = 100 app servers needed
(assuming each server handles ~500 req/sec at target latency)

With 20% headroom for spikes: 125 servers

Cache hit rate target: 95%+
→ Only 2,500 req/sec reach the database
→ 5 read replicas at 500 req/sec each handles this comfortably

Queue-based architecture for write-heavy operations:
→ Search indexing happens async, not in the request path
→ New content appears in search results within seconds, not immediately
→ Write throughput is decoupled from read throughput
```

**The performance cliff to watch:**

```
At 70% capacity (35,000 req/sec): latency ~25ms (well within budget)
At 85% capacity (42,500 req/sec): latency ~60ms (still okay, alarm threshold)
At 95% capacity (47,500 req/sec): latency ~200ms (over budget, auto-scale triggers)
At 100% capacity (50,000 req/sec): latency unbounded (never reach this)
```

Auto-scaling is configured to add servers when sustained load exceeds 70% capacity — maintaining the headroom that keeps latency predictable, not waiting until the system is degraded.

---

## 11. Self-Check

1. What is the difference between latency and throughput? Give a real-world analogy that illustrates both.
2. Why is average latency a misleading metric? What should you use instead?
3. What is tail latency, and why does it matter more at scale than at low traffic?
4. What is the "knee of the curve" and what does it tell you about how to run your system?
5. A system performs well at 1,000 requests/second but latency degrades badly at 5,000. You suspect queuing latency is the cause. What would you look at to confirm, and what are your options to fix it?
6. Why do throughput optimizations (batching, async processing) often increase latency for individual operations? When is that tradeoff acceptable?
7. You're designing a video streaming service. The two most important performance requirements are: (a) the video starts playing within 2 seconds of pressing play, and (b) the system can serve 10 million concurrent streams. Which requirement is about latency and which is about throughput? How do you design for each?

---

## 12. References

| Resource | Why it's worth it |
|----------|-------------------|
| 📘 [Designing Data-Intensive Applications — Ch. 1 (Kleppmann)](https://dataintensive.net) | The best concise treatment of latency, throughput, and percentiles in any book |
| 📊 [Google SRE Book — Service Level Objectives](https://sre.google/sre-book/service-level-objectives/) | How Google defines and measures latency and throughput targets in production |
| 📝 [Jeff Dean — Numbers Every Engineer Should Know](https://highscalability.com/google-pro-tip-use-back-of-the-envelope-calculations-to-choo/) | The latency numbers (RAM vs disk vs network) that inform every performance decision |
| 📬 [ByteByteGo — Latency Numbers](https://bytebytego.com) | Visual reference for the latency of common operations — worth memorizing the order of magnitude |
| 🔧 [The USE Method — Brendan Gregg](https://www.brendangregg.com/usemethod.html) | A systematic method for finding performance bottlenecks: Utilization, Saturation, Errors |

---

*⬅️ Previous: [Fault Tolerance](faultTolerance.md) &nbsp;|&nbsp; ➡️ Next Section: [Back-of-the-Envelope Calculations](../back-of-the-envelope-calculations/BackOfTheEnvelopeCalculations.md)*

---

<sub>Part of the <a href="../../README.md">System Design Foundations</a> study guide series — Non-Functional System Characteristics.</sub>