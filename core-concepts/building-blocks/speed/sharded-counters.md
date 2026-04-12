# 🔢 Sharded Counters

> *"A single counter can count anything. A sharded counter can count everything — simultaneously."*

**⏱ Reading time:** ~10 minutes &nbsp;|&nbsp; **📍 Series:** Building Blocks &nbsp;|&nbsp; **🗂 Group:** Speed

---

## Table of Contents

1. [What Problem Sharded Counters Solve](#1-what-problem-sharded-counters-solve)
2. [Why a Single Counter Breaks at Scale](#2-why-a-single-counter-breaks-at-scale)
3. [The Sharding Solution](#3-the-sharding-solution)
4. [Reading the Count — The Aggregation Problem](#4-reading-the-count--the-aggregation-problem)
5. [Shard Count — How Many Shards?](#5-shard-count--how-many-shards)
6. [Implementation Approaches](#6-implementation-approaches)
7. [Consistency Tradeoffs](#7-consistency-tradeoffs)
8. [Real-World: How Twitter Handles Celebrity Like Counts](#8-real-world-how-twitter-handles-celebrity-like-counts)
9. [When to Use Sharded Counters](#9-when-to-use-sharded-counters)
10. [How Sharded Counters Connect to Other Building Blocks](#10-how-sharded-counters-connect-to-other-building-blocks)
11. [Self-Check](#11-self-check)
12. [References](#12-references)

---

## 1. What Problem Sharded Counters Solve

Counting seems trivial. Increment a number. What could go wrong?

At scale, a lot. When millions of users simultaneously try to increment the same counter — likes on a viral post, views on a trending video, concurrent users on a popular game — the counter becomes one of the most contended resources in your entire system.

A sharded counter solves this by replacing one counter that everyone writes to with many counters that writes are distributed across. The total is the sum of all shards. Contention per shard drops dramatically; total throughput increases proportionally with the number of shards.

---

## 2. Why a Single Counter Breaks at Scale

Consider a `likes` column on a post in a database:

```sql
UPDATE posts SET likes = likes + 1 WHERE post_id = 12345
```

This looks harmless. At 10 likes/second, it's fine. At 100,000 likes/second for a viral celebrity post — it isn't.

### The Database Lock Problem

When a database row is updated, it gets locked briefly to prevent two simultaneous updates from producing an incorrect result:

```
Thread 1: READ likes=100, ADD 1, WRITE 101 ─────────────►
Thread 2:     READ likes=100, ADD 1, WRITE 101
                                         ↑
                                   Both read 100,
                                   both write 101
                                   One like is lost
```

To prevent this, the database serializes writes — only one thread modifies the row at a time. Every other write waits in a queue.

```
At 100,000 writes/second, each taking 1ms:
  Queue depth = 100,000 / 1,000 = 100 writes always waiting
  Average wait time = 50ms per write
  
At 1,000,000 writes/second:
  Queue depth = 1,000 writes waiting
  Average wait time = 500ms per write
  → System is effectively broken
```

The single row has become a **serialization bottleneck** — a point where parallel work becomes sequential, and adding more servers doesn't help because they all contend for the same lock.

### Why Redis Atomic Increment Isn't Enough Alone

Redis's `INCR` command is atomic and fast (~0.1ms). At 100,000 ops/second it handles a single counter fine. But Redis is single-threaded — it processes one command at a time. At extreme scale (millions of operations per second on a single key), even Redis saturates.

And for database-stored counts that need to be durable and consistent with other fields, you can't just punt everything to Redis.

---

## 3. The Sharding Solution

Instead of one counter, maintain N counters (shards). Each write goes to a randomly selected shard. The total count is the sum of all shards.

```
Post 12345 — 10 shards:

  shard_0: 98,341
  shard_1: 97,892
  shard_2: 99,103
  shard_3: 98,756
  shard_4: 97,654
  shard_5: 99,201
  shard_6: 98,543
  shard_7: 97,988
  shard_8: 98,765
  shard_9: 99,432

Total likes = sum of all shards = 985,675
```

A user likes the post:
```
shard_index = random(0, 9)  # pick a random shard
UPDATE counter_shards 
SET count = count + 1 
WHERE post_id = 12345 AND shard_id = shard_index
```

Now 10 shards share the write load. Contention per shard drops by 10×. With 100 shards, contention drops by 100×. The write throughput scales linearly with shard count.

```
Before sharding: 1,000 writes/second max (lock bottleneck)
After 10 shards: ~10,000 writes/second
After 100 shards: ~100,000 writes/second
```

---

## 4. Reading the Count — The Aggregation Problem

Writing is now fast and distributed. Reading introduces a new challenge: to get the total count, you must aggregate all shards.

```sql
SELECT SUM(count) 
FROM counter_shards 
WHERE post_id = 12345
```

This reads 10 (or 100) rows instead of 1. For a single post, this is trivial. For a feed showing 50 posts each with their like counts, it's 50 × N_shards reads — potentially 5,000 reads to render one page.

**Solutions to the aggregation problem:**

### Option 1: Cache the Aggregate
Periodically compute the total and store it in a fast cache (Redis). Reads always hit the cache; the cache is refreshed every few seconds.

```
Background job (every 5 seconds):
  total = SELECT SUM(count) FROM counter_shards WHERE post_id = 12345
  CACHE.SET("post:12345:likes", total, TTL=10)

Read path:
  likes = CACHE.GET("post:12345:likes")
  → Returns cached aggregate (fast, slightly stale)
```

**Tradeoff:** Like count may be a few seconds stale. Acceptable for most social media use cases.

### Option 2: Maintain a Separate Aggregate Counter
Keep a separate "total" counter updated periodically by a background job. Reads use the total counter; writes go to shards.

```
Write: increment random shard
Background job: every N seconds, sum shards → update total counter
Read: read total counter (single row, fast)
```

### Option 3: Accept the Aggregation Cost
For lower-volume counters or infrequent reads, just SUM the shards on every read. 10 rows is still very fast. Only becomes a problem at very high read volume.

---

## 5. Shard Count — How Many Shards?

Too few shards: still bottlenecked.
Too many shards: aggregation becomes expensive.

The right number depends on your write throughput requirements:

```
Rule of thumb:
  Shard count ≈ peak_writes_per_second / max_writes_per_shard_per_second

Example:
  Peak writes: 100,000/sec
  DB can handle: 5,000 writes/sec per row safely
  Shards needed: 100,000 / 5,000 = 20 shards

Add safety margin: 30 shards
```

For most systems, 10-100 shards is sufficient. Twitter uses a similar approach for tweet engagement metrics — a moderate number of shards per metric handles even viral celebrity content.

---

## 6. Implementation Approaches

### Database Shards
Store shards as rows in a relational database or key-value store.

```sql
-- Schema
CREATE TABLE counter_shards (
  entity_id   BIGINT,
  shard_id    SMALLINT,
  count       BIGINT,
  PRIMARY KEY (entity_id, shard_id)
);

-- Write (increment random shard)
UPDATE counter_shards 
SET count = count + 1 
WHERE entity_id = 12345 AND shard_id = floor(random() * 10);

-- Read (aggregate)
SELECT SUM(count) FROM counter_shards WHERE entity_id = 12345;
```

### Redis Shards
Use Redis keys as shards for higher write throughput.

```
Write:
  shard = random(0, 9)
  INCR "likes:post:12345:shard:" + shard

Read (aggregate):
  total = 0
  for shard in range(10):
    total += GET "likes:post:12345:shard:" + shard
```

Redis handles atomic increments natively. The aggregation requires 10 GET commands — use pipelining to send them in one network round trip.

### Approximate Counting with HyperLogLog
For use cases where an approximate count is acceptable (within ~1% error), Redis's HyperLogLog data structure provides space-efficient counting without sharding complexity.

```
PFADD unique_visitors:page:123 user_456
PFADD unique_visitors:page:123 user_789
PFCOUNT unique_visitors:page:123  → ~2 (approximate)
```

HyperLogLog uses fixed memory (12KB) regardless of how many unique elements are counted. Perfect for unique visitor counts, distinct item counts, and any scenario where precision matters less than scale.

---

## 7. Consistency Tradeoffs

Sharded counters accept some consistency relaxation in exchange for write throughput. Understanding what you're giving up is important.

**Eventual consistency of reads:**
After a write to shard 3, a read that sums shards 0-9 reflects the increment — if it happens after the write commits. But if you're reading from replicas, replicas may lag. The count a user sees may be slightly behind real-time.

**No strong ordering guarantees:**
Two users who liked a post "at the same time" may see different counts depending on which shard each write went to and when the aggregation ran.

**Acceptable for:** Like counts, view counts, follower counts, download counts. Users don't expect these to be microsecond-precise. Seeing 9,847 vs 9,849 doesn't matter.

**Not acceptable for:** Inventory counts (overselling is catastrophic), financial balances (every cent must be accurate), seat reservations (double-booking is a disaster). For these, use serialized transactions and accept the throughput limitation.

---

## 8. Real-World: How Twitter Handles Celebrity Like Counts

When Taylor Swift posts a tweet, hundreds of thousands of likes can arrive within seconds. A single counter would serialize under this load.

Twitter's approach (documented in engineering blog posts) combines several techniques:

```
1. Write path: sharded counters distributed across Redis nodes
   Each like → random shard → atomic INCR

2. Aggregation: background job sums shards periodically
   Cached aggregate updated every few seconds

3. Read path: cached aggregate served from Redis
   Reads are O(1) regardless of like count or shard count

4. Fallback: if cache is cold, sum shards directly
   ~10 Redis operations, still fast enough for a cache miss
```

The key insight: the like count displayed to users doesn't need to be exact to the millisecond. A count that's 5 seconds behind real-time is indistinguishable from real-time to a human reader. Trading strict consistency for this level of throughput is almost always the right call for engagement metrics.

---

## 9. When to Use Sharded Counters

**Use sharded counters when:**
- Write rate on a single counter exceeds what your storage can handle
- You're tracking engagement metrics (likes, views, shares, votes)
- The counter is for popular content (anything that could go viral)
- You can tolerate approximate or slightly delayed counts

**Don't use sharded counters when:**
- The count represents money, inventory, or anything that must be exact
- Write rate is low enough that a single counter handles it fine
- You need to count distinct items (use HyperLogLog or a set)

**The pre-emptive vs reactive decision:**
You don't need to shard all counters from day one. Start with simple counters. Monitor write throughput on high-traffic counters. Migrate to sharded counters when you approach the single-counter limit. For social media, anything attached to content from users with large followings should be sharded preemptively.

---

## 10. How Sharded Counters Connect to Other Building Blocks

```
Database / Key-Value Store ─────────────────────────────────────────────►
  Where shards are stored.
  Each shard is a row (DB) or a key (Redis).

Distributed Cache ───────────────────────────────────────────────────────►
  Stores the cached aggregate count.
  Reads hit the cache; background job keeps it fresh.
  Without the cache, every read must sum all shards.

Message Queue ────────────────────────────────────────────────────────────►
  Alternative pattern: like events go to a queue.
  Worker consumes queue, updates sharded counters in batches.
  Smoother write path, slight additional latency.

Distributed Monitoring ──────────────────────────────────────────────────►
  Monitor shard distribution (are writes balanced across shards?).
  Alert on shard hotspots.
  Track aggregation lag (how stale is the cached count?).
```

---

## 11. Self-Check

1. Why does a single database counter become a bottleneck at high write volumes? What specifically causes the serialization?
2. How does sharding solve the write bottleneck? What is the write path with N shards?
3. What is the aggregation problem, and what are two ways to address it?
4. You're designing a YouTube-like system. A popular video gets 500,000 views per second during a viral spike. How many shards would you need if each database shard can safely handle 5,000 writes/second?
5. Why are sharded counters inappropriate for inventory management but perfectly fine for like counts?
6. What is HyperLogLog, and when would you use it instead of sharded counters?
7. A user likes a post, then immediately views the post detail page. The like count shows the old number (doesn't include their like). Is this a bug? How would you explain it, and would you fix it?

---

## 12. References

| Resource | Why it's worth it |
|----------|-------------------|
| 📬 [ByteByteGo — Designing a Like System](https://bytebytego.com) | End-to-end design of a like counter system with sharding |
| 🔧 [Redis — HyperLogLog](https://redis.io/docs/data-types/probabilistic/hyperloglogs/) | How HyperLogLog works and when to use it |
| 📝 [Twitter Engineering — Handling High-Volume Counters](https://blog.twitter.com/engineering/en_us/topics/infrastructure) | Real-world implementation of counters at Twitter scale |
| 📘 [Designing Data-Intensive Applications — Ch. 6 (Kleppmann)](https://dataintensive.net) | Partitioning strategies and their tradeoffs |

---

*⬅️ Previous: [Distributed Cache](distributed-cache.md) &nbsp;|&nbsp; ➡️ Next Group: [Communication](../communication/Communication.md)*

---

<sub>Part of the <a href="../../README.md">System Design Foundations</a> study guide series — Building Blocks: Speed.</sub>