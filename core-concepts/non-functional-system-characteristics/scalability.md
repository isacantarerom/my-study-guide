# 📈 Scalability

> *"A system that works for 100 users but falls apart at 100,000 isn't a scalable system — it's a prototype that got lucky."*

**⏱ Reading time:** ~13 minutes &nbsp;|&nbsp; **📍 Series:** Grokking System Design &nbsp;|&nbsp; **🗂 Topic #3:** Non-Functional Characteristics

---

## Table of Contents

1. [What Scalability Actually Means](#1-what-scalability-actually-means)
2. [The Two Dimensions of Scale](#2-the-two-dimensions-of-scale)
3. [Vertical vs Horizontal Scaling](#3-vertical-vs-horizontal-scaling)
4. [What Actually Breaks as Systems Scale](#4-what-actually-breaks-as-systems-scale)
5. [Stateless vs Stateful Services](#5-stateless-vs-stateful-services)
6. [Scaling the Database](#6-scaling-the-database)
7. [Caching as a Scaling Strategy](#7-caching-as-a-scaling-strategy)
8. [Async Processing and Queues](#8-async-processing-and-queues)
9. [The Cost of Scaling](#9-the-cost-of-scaling)
10. [Worked Example: Scaling a URL Shortener](#10-worked-example-scaling-a-url-shortener)
11. [Self-Check](#11-self-check)
12. [References](#12-references)

---

## 1. What Scalability Actually Means

**Scalability is a system's ability to handle growth — in users, data, requests, or complexity — without requiring a complete redesign.**

The key word is *growth*. A scalable system doesn't need to handle infinite load from day one. It needs to be designed in a way that allows capacity to be added incrementally as demand increases, without throwing out what already exists.

This is an important nuance. Over-engineering for scale you don't have yet is as much of a problem as under-engineering for scale you do have. A startup serving 500 users that's running a 20-node distributed database is wasting money and engineering time. A social network that designed for 10,000 users and now has 10 million is in crisis. The goal is a system whose scaling path is clear and executable — not one that's already there, and not one that has no path at all.

Scalability also isn't just about handling more requests. It encompasses:
- **Load scalability** — handling more concurrent users and requests
- **Data scalability** — storing and querying more data efficiently
- **Geographic scalability** — serving users across regions with acceptable latency
- **Team scalability** — allowing more engineers to work on the system without stepping on each other

We'll focus primarily on the first two, since those are what comes up most often in system design.

---

## 2. The Two Dimensions of Scale

Before talking about solutions, it helps to be precise about what kind of growth you're designing for, because different growth patterns break different things.

### Read-Heavy Growth
More users reading data than writing it. A news site, a social media feed, a product catalog. Reads vastly outnumber writes.

The bottleneck is usually the database being asked to serve too many read queries. The solution space includes caching, read replicas, and CDNs — all of which reduce load on the primary database without changing the write path.

### Write-Heavy Growth
A high volume of incoming data — IoT sensors, user activity logs, financial transactions, real-time feeds. Writes are the constraint.

Caching doesn't help here. The solution space includes write-optimized databases, sharding (splitting data across multiple nodes), and async processing through message queues.

### Mixed Workloads
Most real systems have both. The design challenge is identifying which is the dominant constraint at each layer and addressing that specifically, rather than applying the same solution everywhere.

Understanding your read/write ratio is one of the first questions to ask when designing a system — because the answer shapes almost every architectural decision that follows.

---

## 3. Vertical vs Horizontal Scaling

These are the two fundamental approaches to adding capacity.

### Vertical Scaling (Scale Up)
Give the existing machine more resources — a bigger CPU, more RAM, a faster disk. The software doesn't change; the hardware just gets more powerful.

```
Before:           After vertical scaling:
┌──────────┐      ┌──────────────────┐
│ 4 cores  │  →   │    16 cores      │
│ 16 GB RAM│      │    128 GB RAM    │
│ App      │      │    App           │
└──────────┘      └──────────────────┘
```

**What's good about it:** Simple. No changes to application architecture. Works immediately.

**What limits it:** There's a ceiling. The biggest available machine has a finite amount of RAM and CPU, and that ceiling is lower than you might think for truly large workloads. Beyond the ceiling, vertical scaling becomes either impossible or cost-prohibitive — the price of hardware increases much faster than its performance at the high end.

Vertical scaling is also a single point of failure. One big machine is still one machine.

### Horizontal Scaling (Scale Out)
Add more machines and distribute the load across them. Each machine handles a portion of the total traffic.

```
Before:           After horizontal scaling:
┌──────────┐      ┌──────────┐ ┌──────────┐ ┌──────────┐
│ App      │  →   │ App      │ │ App      │ │ App      │
│ Server 1 │      │ Server 1 │ │ Server 2 │ │ Server 3 │
└──────────┘      └──────────┘ └──────────┘ └──────────┘
                        ▲            ▲            ▲
                        └────────────┴────────────┘
                                     │
                               Load Balancer
```

**What's good about it:** No theoretical ceiling — you keep adding machines as you need them. It also eliminates single points of failure naturally; if one machine goes down, the others keep serving traffic.

**What makes it hard:** Your application has to be designed for it. A stateful application that stores user session data in memory can't be horizontally scaled without rethinking where that state lives. A database that assumes a single writer needs significant rearchitecting to accept writes across multiple nodes.

### Which Do You Use?

In practice, most systems use both — vertical scaling for the initial growth phase (it's simpler), and horizontal scaling as the ceiling approaches or redundancy becomes necessary. The architectural decisions that determine whether horizontal scaling is even possible are some of the most important ones you'll make early in a system's design.

---

## 4. What Actually Breaks as Systems Scale

Scale doesn't introduce new categories of problems — it amplifies existing ones. Here's what tends to break first:

### The Database Becomes the Bottleneck
In almost every system, the database is the first thing to buckle under scale. Application servers are stateless and easy to add. The database is stateful and hard to distribute. As request volume grows, more queries hit the database, query times increase, connections pile up, and eventually the database becomes the constraint on the entire system.

This is why so much of scalability architecture is really database architecture.

### Hot Spots
A hot spot is when a disproportionate amount of traffic concentrates on one node or one piece of data. Imagine a database sharded by user ID — if one user (a celebrity, a viral post) suddenly generates 10,000x the normal traffic, the shard containing their data becomes overloaded while all other shards are fine.

Hot spots are subtle because they don't show up in average load metrics. Everything looks fine in aggregate; one node is melting. Designing around hot spots requires thinking carefully about how data and traffic are distributed, not just how much there is in total.

### Network Overhead
In a system with one server, function calls are local. In a distributed system, calls between services cross the network. Each network hop adds latency. Multiply that across the dependency chain and you can find that a single user request is spending most of its time waiting for network calls rather than doing actual computation.

This is why the dependency chain availability math from [Availability](availability.md) matters for performance too — more services in the critical path means more network hops, more latency, and more opportunities for things to slow down.

### Coordination Overhead
When multiple nodes need to agree on something — who the current leader is, whether a transaction committed, what the current value of a counter is — they have to coordinate. Coordination requires communication between nodes, which takes time and which can become a bottleneck in itself.

This is Amdahl's Law applied to distributed systems: the parts of a system that require coordination don't benefit from adding more nodes — they become *slower* as more nodes need to participate in the decision. Minimizing coordination in the critical path is a key scalability principle.

---

## 5. Stateless vs Stateful Services

One of the most important architectural choices for horizontal scalability is whether your services are stateless or stateful.

**A stateless service** holds no data between requests. Each request contains everything the service needs to process it. The service computes a result and returns it, leaving no trace of the request in memory.

```
Request 1 → App Server A → processes → returns response
Request 2 → App Server B → processes → returns response
Request 3 → App Server A → processes → returns response

Any server can handle any request. Adding more servers is trivial.
Load balancer can route freely. No coordination needed.
```

**A stateful service** retains information between requests — user session data, in-progress workflows, cached computation. Requests from the same user may need to reach the same server to access that state.

```
User logs in → App Server A → stores session in memory
User makes request → must go to App Server A (session is there)
User makes another request → must go to App Server A again

Adding servers doesn't help this user. Can't route freely.
Server A is now a soft single point of failure for this user.
```

**The pattern that enables horizontal scaling:** move state out of your application servers and into dedicated state stores. Sessions go in Redis. User data goes in the database. Files go in object storage (S3). The application servers become stateless — they read and write state from shared stores, but hold none themselves. Now any server can handle any request, and you can add or remove servers freely.

```
Stateless app servers:        Shared state stores:
┌──────────┐
│ App 1    │ ──────────────► Redis (sessions)
├──────────┤ ──────────────► PostgreSQL (user data)
│ App 2    │ ──────────────► S3 (files)
├──────────┤
│ App 3    │
└──────────┘
     ▲
     │ Any server handles
     │ any request
Load Balancer
```

This is one of the most impactful architectural decisions you'll make. Almost everything else in horizontal scaling becomes easier once your application tier is stateless.

---

## 6. Scaling the Database

Databases are stateful by definition, which makes them the hardest part of any system to scale. The main strategies:

### Read Replicas
Add replica nodes that receive copies of all writes from the primary but only serve reads. Read traffic is distributed across replicas; write traffic still goes to the primary.

```
All writes → Primary DB
Reads      → Replica 1, Replica 2, Replica 3 (load balanced)
```

This works well for read-heavy workloads and can multiply read capacity significantly. The tradeoff is that replicas may be slightly behind the primary — reads from replicas are eventually consistent. For most read operations this is fine; for reads that immediately follow a write (reading back what you just wrote), you may need to route to the primary.

### Sharding (Horizontal Partitioning)
Split data across multiple database nodes, each owning a subset. A user database might be sharded by user ID: users 1–1M on shard 1, users 1M–2M on shard 2, and so on.

```
Write for user 500,000  → Shard 1
Write for user 1,500,000 → Shard 2
Write for user 2,500,000 → Shard 3
```

Sharding scales both reads and writes because each shard handles a smaller total dataset. The tradeoffs are significant though: queries that need to join data across shards become very expensive, and choosing the wrong shard key creates hot spots. Sharding is powerful but adds substantial complexity and should be a considered decision, not a default.

### Denormalization
In a relational database, data is normalized — stored once, referenced by foreign keys. This keeps data consistent but means queries often need to join many tables, which is expensive at scale.

Denormalization is the deliberate choice to store redundant copies of data to make reads faster. Instead of joining five tables to render a user's profile, you store a pre-assembled version of the profile that can be read in a single query.

The tradeoff: writes become more complex (you have to update data in multiple places) and consistency becomes harder to maintain. But for read-heavy workloads where query performance is the constraint, the tradeoff is often worth it.

---

## 7. Caching as a Scaling Strategy

Caching is one of the most powerful and most commonly used scaling tools. The principle is simple: if a computation or database read is expensive and the result doesn't change often, store the result somewhere fast and serve it from there instead.

```
Without caching:                With caching:
                                
User request                    User request
    │                               │
    ▼                               ▼
Database query (10ms)         Cache lookup (0.1ms)
    │                               │
    ▼                           Hit? │ Miss?
Response                            ▼       ▼
                               Return   Database query
                               cached   Store in cache
                               result   Return result
```

A cache hit is 100x faster than a database query. For a system serving thousands of requests per second for the same popular data, caching can reduce database load by orders of magnitude.

The challenge is always cache invalidation — keeping the cache consistent with the underlying data. When data changes, the cache must be updated or expired, or readers will see stale results. This was true in [Abstraction](../preliminary-system-design-concepts/abstraction.md) and it's still the fundamental cost of caching.

Common caching strategies:

**Cache-aside** — the application checks the cache first. On a miss, it queries the database and populates the cache. Simple and flexible; the application controls what gets cached.

**Write-through** — every write goes to both the cache and the database simultaneously. Cache is always fresh. More write latency, but reads are always cache-warm.

**Write-behind (write-back)** — writes go to the cache immediately, and the database is updated asynchronously. Very fast writes, but risk of data loss if the cache fails before the database is updated.

---

## 8. Async Processing and Queues

Not every operation needs to happen synchronously — in the same request/response cycle. Identifying work that can be deferred and processing it asynchronously through a queue is one of the most effective ways to scale write-heavy workloads.

```
Synchronous:                    Asynchronous:

User request                    User request
    │                               │
    ▼                               ▼
Process everything:            Accept request → Queue job → Return "accepted"
  - Save to DB                          │
  - Send email                          ▼ (later, in background)
  - Resize image                    Worker picks up job:
  - Update search index               - Save to DB
  - Notify followers                  - Send email
    │                                 - Resize image
    ▼                                 - Update search index
Return (5 seconds later)              - Notify followers
```

The user gets a response immediately. The heavy work happens in the background. The application server isn't blocked waiting for all those operations to complete.

This also decouples the rate at which requests come in from the rate at which they're processed. If requests spike, they queue up rather than overwhelming the system. Workers process the queue at their own pace and can be scaled independently based on queue depth.

The tradeoff is eventual completion — the user doesn't get confirmation that all the work is done, just that it was accepted. For operations where the user needs an immediate result (a search query, a payment confirmation), async processing isn't appropriate. For operations that can complete in the background (sending a welcome email, generating a report, processing an uploaded video), it's often exactly the right choice.

---

## 9. The Cost of Scaling

Scalability isn't free, and understanding its costs is part of designing for it honestly.

**Operational complexity** — a distributed system is harder to operate than a single server. More moving parts means more failure modes, more monitoring needed, more complex deployments, more things to debug when something goes wrong.

**Consistency tradeoffs** — as we covered in [Consistency Models](../preliminary-system-design-concepts/consistencyModels.md), distributing data across nodes means accepting weaker consistency guarantees in exchange for scale. Read replicas introduce replication lag. Sharding makes cross-shard transactions expensive or impossible.

**Cost** — more servers, more storage, more network bandwidth. Horizontal scaling is cost-effective compared to buying the biggest single machine, but it's not free. Caching layers, message queues, CDNs — each adds infrastructure cost.

**Development time** — building a system that scales horizontally takes more engineering effort than building one that doesn't. Stateless services, external session stores, sharding strategies, cache invalidation logic — these are all real engineering work that takes time.

The practical takeaway: **don't optimize for a scale you don't have yet, but don't make architectural decisions that foreclose the scaling path you'll need.** Making your application tier stateless costs very little upfront and buys enormous flexibility later. Sharding your database on day one when you have 1,000 users is probably premature.

---

## 10. Worked Example: Scaling a URL Shortener

We've used bit.ly as an example before. Let's look at it specifically through the lens of scale — what breaks as it grows, and how each problem is addressed.

**The workload characteristics:**
- Extremely read-heavy: every shortened URL gets clicked many times after being created once
- Write volume is low: creating a short URL happens infrequently compared to redirecting
- Latency matters: a slow redirect degrades the user experience of whatever site the link points to

```
Day 1: Single server
┌─────────────────────────┐
│ App + DB on one machine │
└─────────────────────────┘
Works fine for low traffic. Simple to operate.

100K users/day: Add a load balancer + separate DB
┌──────────────┐    ┌──────────────┐
│   App 1      │    │   App 2      │
└──────────────┘    └──────────────┘
        └──────────────────┘
                 │
          ┌──────────────┐
          │   Database   │
          └──────────────┘
DB is now the bottleneck for reads.

1M users/day: Add a cache layer
┌─────────────────────────────────────┐
│  Redis Cache (popular short codes)  │
└─────────────────────────────────────┘
          │ miss          │ hit
          ▼               ▼
    ┌──────────┐    Return immediately
    │ Database │    (0.1ms vs 5ms)
    └──────────┘
95%+ of redirect lookups are now cache hits.
DB load drops dramatically.

10M users/day: Add read replicas + CDN
  Writes → DB Primary
  Reads  → DB Replica 1, Replica 2, Replica 3
  Static assets + redirect logic → CDN edge nodes
  
  Users get redirected from the nearest edge node.
  DB replicas share the read load.

100M users/day: Consider sharding
  short_code starts with a-m → Shard 1
  short_code starts with n-z → Shard 2
  
  Each shard handles half the data.
  Can scale independently.
  Cross-shard queries (rare for this use case) are complex.
```

Notice the progression: each scaling step is applied when the previous ceiling is approached. No step is applied "just in case" before it's needed. And each step introduces a tradeoff — caching adds staleness risk, read replicas add consistency lag, sharding adds query complexity.

---

## 11. Self-Check

1. What is the difference between vertical and horizontal scaling? What limits vertical scaling?
2. Why do stateless services scale horizontally more easily than stateful ones? What do you do with the state?
3. What is a hot spot, and why is it dangerous in a sharded database?
4. A social media platform has 10x more reads than writes. Which scaling strategies would you reach for first, and why?
5. What is the tradeoff of async processing via message queues? When is it appropriate and when is it not?
6. Why does adding more nodes to a system sometimes make it *slower* for certain operations?
7. You're designing a system for a startup expecting 10,000 users today but potentially 10 million in two years. What architectural decisions do you make now vs. defer? Why?

---

## 12. References

| Resource | Why it's worth it |
|----------|-------------------|
| 📘 [Designing Data-Intensive Applications — Ch. 1 & 6 (Kleppmann)](https://dataintensive.net) | Ch. 1 introduces scalability clearly; Ch. 6 goes deep on partitioning and sharding |
| 📬 [ByteByteGo — Scaling from 0 to Millions](https://bytebytego.com) | Step-by-step visual walkthrough of exactly how a system scales incrementally |
| 📊 [Google SRE Book — Chapter on Load Balancing](https://sre.google/sre-book/load-balancing-frontend/) | How Google thinks about distributing load at massive scale |
| 🔧 [AWS Auto Scaling Documentation](https://docs.aws.amazon.com/autoscaling/) | How horizontal scaling is implemented in practice on cloud infrastructure |

---

*⬅️ Previous: [Reliability](reliability.md) &nbsp;|&nbsp; ➡️ Next: [Maintainability](maintainability.md)*

---

<sub>Part of the <a href="../../README.md">System Design Foundations</a> study guide series — Non-Functional System Characteristics.</sub>