# ⚡ Speed

> *"Storage keeps data safe. Speed keeps users happy. These are the components that bridge the gap between the two."*

---

## The Problem This Group Solves

Storage systems are designed for durability and correctness. Speed systems are designed for one thing: making access to data dramatically faster than going to primary storage on every request.

Two distinct speed problems show up constantly in large-scale systems:

**The read speed problem** — the same data is requested thousands of times per second. Going to the database each time is too slow and too expensive. You need a layer that serves the most common reads from memory, in microseconds.

**The write speed problem** — millions of users are simultaneously updating the same counter (likes, views, followers). A single counter in a single database row becomes a bottleneck immediately. You need a way to distribute the write load while still producing an accurate total.

These two building blocks solve those two problems respectively.

---

## The Components

| Building Block | Solves | Guide |
|---------------|--------|-------|
| **Distributed Cache** | Serving frequently read data from memory instead of disk | [Read →](distributed-cache.md) |
| **Sharded Counters** | Counting millions of concurrent events without a single bottleneck | [Read →](sharded-counters.md) |

---

## Why Speed Is a Separate Group

You might wonder why caching isn't just part of Storage. The distinction matters because the mindset is different:

**Storage** asks: "Where does this data live permanently?"
**Speed** asks: "How do we serve this data as fast as possible, as often as possible?"

A cache doesn't replace storage — it sits in front of it. The cache is a performance optimization; the database is the source of truth. If the cache disappears, data isn't lost — reads just get slower.

Sharded counters are similar — they're a speed optimization for a specific write pattern. The final count still needs to be reconciled and stored somewhere persistent. The sharding is the mechanism that makes the counting fast, not the storage itself.

---

## The Relationship Between the Two

```
Distributed Cache                    Sharded Counters
─────────────────                    ────────────────
Optimizes: reads                     Optimizes: writes
Problem: "same data requested        Problem: "same counter updated
          many times"                          by millions simultaneously"
Mechanism: store in memory,          Mechanism: split one counter
           serve from there                     into many, sum periodically
Tradeoff: staleness                  Tradeoff: eventual accuracy
```

In a social media system, you'd use both:
- Sharded counters to handle millions of concurrent likes
- Distributed cache to serve the like count to readers without hitting the database

The write path uses sharded counters. The read path uses the cache. Together they handle enormous scale in both directions.

---

## When You Reach for This Group

Your system has read-heavy traffic on specific data → **Distributed Cache**.

Your system tracks counts (likes, views, downloads, votes) at high volume → **Sharded Counters**.

Your estimation shows database QPS would exceed safe limits → **Cache** to reduce DB load.

A single database row is a write bottleneck → **Sharded Counters** to distribute the writes.

---

*⬅️ Previous Group: [Storage](../storage/Storage.md) &nbsp;|&nbsp; ➡️ Next Group: [Communication](../communication/Communication.md)*