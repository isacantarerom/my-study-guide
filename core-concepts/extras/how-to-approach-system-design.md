# 🗺️ How to Approach a System Design Problem

> *"A good framework doesn't tell you what to think. It tells you what to think about — and in what order."*

**⏱ Reading time:** ~12 minutes &nbsp;|&nbsp; **📍 Series:** Extras &nbsp;|&nbsp; **Connects to:** Everything

---

## Table of Contents

1. [Why You Need a Framework](#1-why-you-need-a-framework)
2. [The RESHADED Framework](#2-the-reshaded-framework)
3. [R — Requirements](#3-r--requirements)
4. [E — Estimation](#4-e--estimation)
5. [S — Storage Schema](#5-s--storage-schema)
6. [H — High-Level Design](#6-h--high-level-design)
7. [A — APIs](#7-a--apis)
8. [D — Data Flow](#8-d--data-flow)
9. [E — Evaluation](#9-e--evaluation)
10. [D — Distinctiveness / Deep Dive](#10-d--distinctiveness--deep-dive)
11. [Putting It Together: The 45-Minute Structure](#11-putting-it-together-the-45-minute-structure)
12. [What This Framework Does in Real Work](#12-what-this-framework-does-in-real-work)
13. [References](#13-references)

---

## 1. Why You Need a Framework

System design problems are deliberately open-ended. "Design Twitter." "Design YouTube." "Design a notification system." These aren't questions with one correct answer — they're invitations to demonstrate how you think about complex, ambiguous problems.

The danger of open-ended problems is that without structure, thinking becomes scattered. You might jump straight to implementation details before understanding what the system actually needs to do. You might pick a database before knowing how much data you're storing. You might design for 1,000 users when the requirement is 100 million.

A framework solves this by giving you a **repeatable order of operations** — a sequence of questions to answer before moving to the next step. Not because the sequence is magic, but because each step produces information the next step depends on.

The framework covered here is **RESHADED**, developed by Fahim ul Haq (Educative). The name is an acronym. We'll walk through each letter — but more importantly, we'll explain *why* each step comes where it does in the sequence.

---

## 2. The RESHADED Framework

```
R  →  Requirements
       What does this system need to do?
              │
              ▼
E  →  Estimation
       How much of everything does it need to handle?
              │
              ▼
S  →  Storage Schema
       How is data structured and stored?
              │
              ▼
H  →  High-Level Design
       What are the major components and how do they connect?
              │
              ▼
A  →  APIs
       What is the interface between components and the outside world?
              │
              ▼
D  →  Data Flow
       How does data move through the system end to end?
              │
              ▼
E  →  Evaluation
       Does the design meet the requirements? Where does it fall short?
              │
              ▼
D  →  Distinctiveness / Deep Dive
       What makes this design handle the hard parts well?
```

Each step is a question. Answering it produces the inputs for the next question. Skip a step and you'll find yourself making decisions without the information those decisions depend on.

---

## 3. R — Requirements

**The question:** What does this system actually need to do?

This is the most important step and the most commonly rushed. Before touching architecture, you need to understand what you're building. Requirements come in two kinds:

**Functional requirements** — what the system does. The features. The behaviors.
```
For a URL shortener:
  ✓ Users can submit a long URL and receive a short one
  ✓ Visiting the short URL redirects to the original
  ✓ Short URLs don't expire (or optionally expire after X days)
  ✓ Users can see how many times their link was clicked
```

**Non-functional requirements** — how well the system does it. The qualities.
```
For a URL shortener:
  ✓ Redirects must complete in under 100ms globally
  ✓ System must be highly available (99.99% uptime)
  ✓ Short URLs must be unique and non-guessable
  ✓ System should handle 10 billion redirects per month
```

Non-functional requirements connect directly to the characteristics we covered in [Non-Functional System Characteristics](../non-functional-system-characteristics/NonFunctionalSystemCharacteristics.md). Availability, reliability, scalability, latency — these all start as requirements before they become design constraints.

**What to avoid:** Designing before requirements are clear. It's tempting to jump straight to "I'll use Kafka and Redis and Cassandra" — but without requirements, you don't know if those choices are right, wrong, or wildly over-engineered.

---

## 4. E — Estimation

**The question:** How much of everything does this system need to handle?

Once you know what the system does, you need to know the scale at which it does it. This is where [Back-of-the-Envelope Calculations](../back-of-the-envelope-calculations/BackOfTheEnvelopeCalculations.md) live.

The estimates you need:
- **Traffic** — requests per second, peak vs average
- **Storage** — data per record, growth over time
- **Bandwidth** — data in and out per second
- **Compute** — server count

These numbers do two things. They validate requirements ("10 billion redirects/month is about 3,800 RPS average — that's very achievable") and they reveal architectural necessities ("600 TB/day of photo storage means we need object storage, not a database").

Estimation comes before detailed design because the numbers constrain what design choices are even viable. A system at 100 RPS has different architectural needs than one at 100,000 RPS — not just in scale, but in what components make sense at all.

---

## 5. S — Storage Schema

**The question:** How is data structured, and where does it live?

With requirements and scale established, you can make informed decisions about data. This includes:

**Data model** — what are the entities, what are their relationships, what are the access patterns?

```
URL shortener data model:
  ShortURL {
    short_code:   string (primary key)
    original_url: string
    created_at:   timestamp
    user_id:      string (nullable — anonymous links allowed)
    expires_at:   timestamp (nullable)
    click_count:  integer
  }
```

**Storage technology** — relational, NoSQL, object storage, cache? The choice follows from the access patterns and scale, not from preference.

```
URL shortener storage choice:
  Primary storage:   Key-value store (short_code → URL data)
                     Why: lookups are always by short_code, no joins needed,
                          high read throughput needed
  Cache:             Redis for hot short codes
                     Why: 80%+ of traffic is to popular links, cache hit
                          rate dramatically reduces DB load
  Analytics:         Separate write-optimized store for click events
                     Why: high write volume, queries are aggregations not lookups
```

Storage schema comes before high-level design because the data layer is the foundation everything else is built on. Getting it wrong is expensive to undo — database migrations at scale are painful.

---

## 6. H — High-Level Design

**The question:** What are the major components, and how do they connect?

This is the part most people think of as "system design" — drawing boxes and arrows. But notice it's step 4, not step 1. By the time you're drawing the high-level diagram, you know what the system needs to do (R), at what scale (E), and what the data looks like (S). The high-level design follows from those constraints rather than preceding them.

A high-level design identifies the major components:

```
URL shortener high-level design:

Client
  │
  ▼
Load Balancer
  │
  ├─── Write Path ─────────────────────────────────────►
  │    API Server → Code Generator → Key-Value Store
  │
  └─── Read Path (redirect) ───────────────────────────►
       API Server → Cache (Redis) → Key-Value Store (on miss)
                                 → Return 301/302 redirect
```

At this stage, the diagram should be readable and correct at a high level — not exhaustively detailed. Details come in the deep dive.

---

## 7. A — APIs

**The question:** What is the interface between components and the outside world?

APIs define the contracts between services and between the system and its clients. Defining them explicitly forces clarity about what each component actually does and what it promises to callers.

```
URL shortener API:

POST /urls
  Request:  { original_url: string, expires_in_days?: number }
  Response: { short_code: string, short_url: string, created_at: timestamp }
  Errors:   400 (invalid URL), 429 (rate limited)

GET /{short_code}
  Response: 301 redirect to original_url
  Errors:   404 (not found), 410 (expired)

GET /urls/{short_code}/stats
  Response: { click_count: integer, created_at: timestamp, ... }
  Auth:     required (only URL creator can see stats)
```

Well-defined APIs reveal edge cases you might have missed: what happens when a URL is expired? What happens when the same URL is submitted twice? What auth model does the stats endpoint use? These questions are easier to answer at the API level than buried in implementation.

---

## 8. D — Data Flow

**The question:** How does data move through the system, end to end?

Walk through the life of a request — from the moment it enters the system to the moment a response is returned. Trace it through every component it touches.

```
URL shortener — redirect data flow:

1. User visits https://short.ly/abc123
2. DNS resolves short.ly → Load Balancer IP
3. Load Balancer routes to an API server (round-robin)
4. API server extracts short_code = "abc123"
5. API server checks Redis cache for "abc123"
   → Cache HIT: return cached original URL (80% of cases)
   → Cache MISS: query Key-Value Store
       → Found: store in Redis (TTL: 24 hours), return URL
       → Not found: return 404
       → Expired: return 410
6. API server returns HTTP 301 redirect to original URL
7. Browser follows redirect to original destination
8. Async: click event logged to analytics queue (non-blocking)
```

Walking through data flow exposes gaps: where could this fail? Where are the bottlenecks? What happens if step 5 is slow? This is where fault tolerance and consistency decisions get made concrete.

---

## 9. E — Evaluation

**The question:** Does this design actually meet the requirements?

Go back to the requirements from step 1 and check each one against the design:

```
URL shortener evaluation:

Requirement: Redirects under 100ms globally
→ With CDN + Redis cache, most redirects complete in <10ms ✓
→ Cache misses add ~20ms for Key-Value Store query ✓
→ Global CDN edge nodes eliminate geographic latency ✓

Requirement: 99.99% availability
→ Load balancer eliminates single app server SPOF ✓
→ Multiple Key-Value Store replicas ✓
→ Redis with replica for cache failover ✓
→ Estimated MTTR for any single failure: <60 seconds ✓

Requirement: 10 billion redirects/month (~3,800 RPS average)
→ Peak RPS ~12,000 (3× average)
→ With 80% cache hit rate, DB sees ~2,400 QPS at peak ✓
→ Key-Value Store handles 50,000+ QPS comfortably ✓

Weakness identified: cache stampede on popular link expiry
→ Mitigation: cache lock / probabilistic early expiration
```

Evaluation isn't about proving the design is perfect — it's about finding the weaknesses and being honest about them. Every design has weaknesses. The good ones name them.

---

## 10. D — Distinctiveness / Deep Dive

**The question:** What are the hard parts, and how does this design handle them specifically?

Every system has one or two genuinely hard problems — the things that make this design different from a naive implementation. The deep dive is where you go into detail on those.

```
URL shortener deep dives:

1. Short code generation
   → How do we generate unique, short, non-guessable codes?
   → Options: hash + truncate (collision risk), counter-based
     (predictable), pre-generated pool (complex but reliable)
   → Chosen: pre-generated pool of codes, stored in a separate
     "available codes" table, claimed atomically on URL creation
   → Eliminates collision risk and avoids synchronization issues

2. Analytics without impacting redirect latency
   → Click counting cannot be synchronous — adds latency to redirects
   → Solution: fire-and-forget event to Kafka on each redirect
   → Worker pool consumes events and aggregates into analytics DB
   → User sees redirect instantly; click count updates asynchronously
   → Tradeoff: click counts lag by seconds, not real-time
```

The deep dive is where the real engineering thinking shows. Anyone can draw a box labeled "database." Fewer people can explain *how* you generate 10 billion unique short codes without collisions, or *why* analytics are decoupled from the redirect path.

---

## 11. Putting It Together: The 45-Minute Structure

If you're working through a design problem under time pressure, here's a rough time allocation:

```
Minutes 0-5:   Requirements (R)
               Clarify functional and non-functional requirements.
               Don't skip this. 5 minutes here saves 20 minutes of
               designing the wrong thing.

Minutes 5-10:  Estimation (E)
               Quick back-of-the-envelope. Traffic, storage, bandwidth.
               State assumptions clearly.

Minutes 10-15: Storage Schema (S)
               Data model, storage technology choice, justification.

Minutes 15-25: High-Level Design (H) + APIs (A)
               Major components, connections, key APIs.
               Draw the diagram. Walk through it.

Minutes 25-35: Data Flow (D)
               Trace one or two key requests end-to-end.
               Identify bottlenecks and failure points.

Minutes 35-40: Evaluation (E)
               Check requirements. Name the weaknesses.

Minutes 40-45: Deep Dive (D)
               Go deep on the one or two hardest parts.
               This is where you differentiate.
```

In real work, there's no 45-minute clock — but the sequence still holds. You might spend a week on requirements, a day on estimation, and a sprint on high-level design. The order of operations is the same regardless of the time scale.

---

## 12. What This Framework Does in Real Work

The RESHADED framework isn't just for interviews. It's a structured way to think through any system you're building or evaluating. In real engineering work:

**Requirements** prevent building the wrong thing. Teams that skip straight to implementation and discover three months later that they misunderstood the problem are experiencing a requirements failure.

**Estimation** prevents architectural surprises. A system that seemed fine in design but collapses under real load almost always reflects estimation that was skipped or done too loosely.

**Storage schema** designed upfront prevents the expensive pain of migrating a production database after the system is live and handling real traffic.

**Evaluation** that honestly names weaknesses is what creates the improvement backlog — the list of known limitations that get addressed as the system matures.

The framework is a thinking tool. Use it to structure how you reason through complexity, not as a script to recite.

---

## 13. References

| Resource | Why it's worth it |
|----------|-------------------|
| 🎓 [Grokking Modern System Design — Educative](https://www.educative.io/courses/grokking-the-system-design-interview) | The source of the RESHADED framework |
| 📝 [System Design Primer](https://github.com/donnemartin/system-design-primer) | Open-source collection of system design patterns and approaches |
| 📬 [ByteByteGo — System Design Framework](https://bytebytego.com) | Visual walkthroughs of complete system designs using a structured approach |

---

*↩ Back to [Extras Index](ExtrasIndex.md)*

---

<sub>Part of the <a href="../../README.md">System Design Foundations</a> study guide series — Extras.</sub>