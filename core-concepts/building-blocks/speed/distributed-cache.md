# ⚡ Distributed Cache

> *"A cache doesn't change what your system knows. It changes how fast it can say it."*

**⏱ Reading time:** ~13 minutes &nbsp;|&nbsp; **📍 Series:** Building Blocks &nbsp;|&nbsp; **🗂 Group:** Speed

---

## Table of Contents

1. [What a Distributed Cache Is](#1-what-a-distributed-cache-is)
2. [Why Caching Works — Locality of Reference](#2-why-caching-works--locality-of-reference)
3. [Cache Topology — Where the Cache Lives](#3-cache-topology--where-the-cache-lives)
4. [Caching Strategies — How You Write to the Cache](#4-caching-strategies--how-you-write-to-the-cache)
5. [Eviction Policies — What Gets Removed](#5-eviction-policies--what-gets-removed)
6. [Cache Invalidation — The Hard Problem](#6-cache-invalidation--the-hard-problem)
7. [Distributed Cache Internals](#7-distributed-cache-internals)
8. [Cache Failure Modes](#8-cache-failure-modes)
9. [What to Cache and What Not To](#9-what-to-cache-and-what-not-to)
10. [How Distributed Cache Connects to Other Building Blocks](#10-how-distributed-cache-connects-to-other-building-blocks)
11. [Self-Check](#11-self-check)
12. [References](#12-references)

---

## 1. What a Distributed Cache Is

A cache is a fast, temporary storage layer that holds copies of data so future requests for that data can be served faster. A **distributed cache** extends this concept across multiple nodes — multiple cache servers working together, storing data across their combined memory.

The single most important property of a cache: **it is not the source of truth.** The database is the source of truth. The cache holds copies. If the cache disappears entirely, you lose speed — not data.

This distinction is what makes caching different from storage. The cache can always be rebuilt from the database. Its job is to answer questions faster, not to hold data permanently.

```
Without cache:                    With cache:
  Request → Database (5ms)          Request → Cache (0.3ms) → done ✓
            ↑                                 ↓ (miss)
            every single time        Database (5ms) → populate cache
                                              ↓
                                     Request → Cache (0.3ms) → done ✓
                                              next time
```

---

## 2. Why Caching Works — Locality of Reference

Caching is effective because real-world data access is not uniformly random. It follows a pattern called **locality of reference**: a small fraction of your data accounts for a large fraction of your requests.

```
Twitter example:
  500 million tweets exist
  ~0.01% of tweets are from celebrities with millions of followers
  Those ~50,000 tweets might account for 50%+ of all read traffic

  Cache those 50,000 tweets in memory → serve half your traffic from cache
  The other 499,950,000 tweets? Mostly never read after the first day.
```

This is the **Pareto principle (80/20 rule)** applied to data access: 20% of your data receives 80% of requests. Often the skew is even more extreme — 1% of data gets 99% of requests.

Caching works because you only need to cache the hot 1% to dramatically reduce load on the database. A cache that's 1% the size of your database can absorb the vast majority of read traffic.

---

## 3. Cache Topology — Where the Cache Lives

### In-Process Cache (Local Cache)
The cache lives inside the application server's own memory — a hash map in the application process itself.

```
Application Server
  ┌──────────────────────────────┐
  │  Application Code            │
  │  ┌────────────────────────┐  │
  │  │  In-Process Cache      │  │
  │  │  (HashMap in memory)   │  │
  │  └────────────────────────┘  │
  └──────────────────────────────┘
```

**Pros:** Fastest possible — no network hop. Sub-millisecond access.
**Cons:** Not shared between servers. If you have 10 app servers, each has its own cache — 10 copies of the same data. A cache miss on Server A doesn't benefit from Server B's cached data. Cache is lost when the process restarts.

**Good for:** Extremely hot data that's the same for all users (configuration, feature flags, static lookup tables).

### Distributed Cache (External Cache)
The cache is a separate service (Redis, Memcached) that all application servers share.

```
App Server 1 ──┐
App Server 2 ──┼──► Redis Cluster (shared distributed cache)
App Server 3 ──┘         │
                          ├── Cache Node 1
                          ├── Cache Node 2
                          └── Cache Node 3
```

**Pros:** Shared across all application servers. One server's cache hit benefits all others. Scales independently. Survives app server restarts.
**Cons:** Network hop required (0.1-1ms). More infrastructure to manage.

**Good for:** Most production caching scenarios where you have multiple application servers.

### CDN as Cache
Content cached at edge nodes globally. We covered this in [CDN](../traffic-and-routing/cdn.md) — the CDN is essentially a geographically distributed cache for static content.

---

## 4. Caching Strategies — How You Write to the Cache

The strategy determines when and how data gets into the cache, and what consistency guarantees that provides.

### Cache-Aside (Lazy Loading)
The application manages the cache explicitly. On read: check cache first; on miss, fetch from DB and populate cache.

```
function getUser(userId):
  // 1. Check cache
  cached = cache.get("user:" + userId)
  if cached: return cached

  // 2. Cache miss — fetch from database
  user = database.query("SELECT * FROM users WHERE id = ?", userId)

  // 3. Populate cache for future requests
  cache.set("user:" + userId, user, ttl=300)
  return user
```

**Pros:** Only requested data gets cached (no wasted memory on data nobody reads). Cache failure degrades gracefully — just slower, not broken.
**Cons:** First request always misses (cold start). Potential for stale data if DB updates without cache invalidation.

**This is the most common caching pattern in practice.**

### Write-Through
Every write goes to both the cache and the database simultaneously.

```
function updateUser(userId, data):
  database.update(users, userId, data)   // write to DB
  cache.set("user:" + userId, data)      // write to cache simultaneously
```

**Pros:** Cache is always fresh — reads never return stale data.
**Cons:** Every write is slower (must wait for both DB and cache). Cache fills with data that may never be read again (write amplification).

**Good for:** Systems where read-after-write consistency is critical and write volume is manageable.

### Write-Behind (Write-Back)
Writes go to the cache first; the database is updated asynchronously in the background.

```
function updateUser(userId, data):
  cache.set("user:" + userId, data)   // write to cache immediately
  queue.publish("user_updated", {userId, data})  // DB update happens later

// Background worker:
  db.update(users, userId, data)
```

**Pros:** Very fast writes — user gets a response immediately.
**Cons:** Risk of data loss if cache fails before the DB is updated. Complexity of async consistency.

**Good for:** Write-heavy workloads where write latency matters more than immediate durability.

### Read-Through
The cache itself is responsible for fetching from the database on a miss (rather than the application handling it).

Similar to cache-aside from the application's perspective, but the cache layer manages the DB interaction. Less common; some caching frameworks support it.

---

## 5. Eviction Policies — What Gets Removed

A cache has finite memory. When it's full and a new item needs to be stored, something must be evicted. The eviction policy determines what gets removed.

### LRU — Least Recently Used
Evicts the item that was accessed least recently. The assumption: if you haven't used it recently, you're unlikely to use it soon.

```
Cache (capacity 3):
  State: [A, B, C]  (A oldest, C newest)
  Access D → evict A (LRU), add D → [B, C, D]
  Access B → B becomes newest → [C, D, B]
  Access E → evict C (LRU), add E → [D, B, E]
```

**LRU is the most common eviction policy.** It works well for most workloads because recently accessed data is likely to be accessed again soon (temporal locality).

### LFU — Least Frequently Used
Evicts the item accessed least often overall. Tracks access count, not recency.

```
item_A: accessed 100 times
item_B: accessed 3 times  ← evicted first
item_C: accessed 50 times
```

**Good for:** Workloads with stable "always popular" content. **Problem:** Newly added items have low counts and are vulnerable to eviction even if they're becoming popular.

### FIFO — First In, First Out
Evicts the oldest item regardless of access frequency or recency.

Simple to implement, rarely the best choice. Doesn't consider usage patterns.

### TTL-Based Expiry
Items are evicted when their TTL expires. Not a capacity eviction policy — works alongside LRU/LFU to ensure stale data is removed even if the cache isn't full.

**In practice:** Redis uses LRU (or LFU, configurable) for capacity eviction, combined with TTL for time-based expiry.

---

## 6. Cache Invalidation — The Hard Problem

Phil Karlton famously said: *"There are only two hard things in computer science: cache invalidation and naming things."*

The problem: when the underlying data changes, cached copies become stale. Users read outdated data. How do you keep the cache consistent with the database?

### Strategy 1: TTL (Let It Expire)
Set a TTL on every cached item. Stale data is served until the TTL expires, then a fresh copy is fetched.

```
User updates their profile → database updated
Cached profile still served for up to 5 minutes (TTL=300)
After 5 minutes: cache expires, next read fetches fresh data
```

**Pros:** Simple. No explicit invalidation logic.
**Cons:** Stale window. User may see old data for up to TTL seconds after an update.
**Good for:** Data where a few minutes of staleness is acceptable (product descriptions, user preferences, public profiles).

### Strategy 2: Explicit Invalidation on Write
When data changes, immediately delete or update the cached entry.

```
function updateUserProfile(userId, data):
  database.update(users, userId, data)
  cache.delete("user:" + userId)  // immediately invalidate
```

Next read will miss the cache and fetch fresh data from the database.

**Pros:** No stale window — cache is always consistent with DB.
**Cons:** More complex write path. Risk of race conditions (delete cache, then another request repopulates with old data before DB write commits).

### Strategy 3: Event-Driven Invalidation
Database changes emit events; a cache invalidation service listens and purges affected keys.

```
Database update → publishes "user_123_updated" event
Cache invalidation service → receives event → deletes "user:123:*" from cache
```

**Pros:** Decoupled — application code doesn't manage cache invalidation.
**Cons:** Complexity of event infrastructure. Small window between DB update and cache invalidation where stale data may be served.

### The Cache Stampede Problem
When a popular cached item expires, many requests simultaneously find a cache miss and all hit the database at once — a **cache stampede** (also called thundering herd).

```
"viral_post:12345" cached with TTL=60s

At t=60: TTL expires
10,000 simultaneous requests → all miss cache → all hit database
Database overwhelmed → latency spike for all users
```

**Solutions:**
- **Mutex/lock:** When a cache miss occurs, only one request fetches from DB and repopulates. Others wait for that result.
- **Probabilistic early expiration:** Before TTL expires, randomly re-fetch for some requests — the cache "warms up" before it goes cold.
- **Jitter on TTL:** Add random variation to TTL so items don't all expire simultaneously.

---

## 7. Distributed Cache Internals

### How Data Is Distributed Across Nodes

A distributed cache like Redis Cluster shards data across multiple nodes using consistent hashing. Each key is assigned to a node; requests for that key always go to that node.

```
hash(key) → node assignment

"user:123:profile"  → Node 1
"user:456:profile"  → Node 2
"session:token_abc" → Node 3
```

This means a single cache cluster can hold more data than any single server's memory allows, and read/write operations scale across nodes.

### Replication Within the Cache Cluster

Each cache node has replicas for fault tolerance. If a node fails, its replica takes over.

```
Node 1 (primary)  ──► Node 1-replica
Node 2 (primary)  ──► Node 2-replica
Node 3 (primary)  ──► Node 3-replica
```

### Hot Keys — The Cache's Own Hot Spot Problem

If a single key is accessed millions of times per second (a viral post, a popular product), a single cache node becomes the bottleneck — even though the cluster has many nodes.

**Solutions:**
- **Key replication:** Store the same data under multiple keys (`viral:post:123:replica:1`, `viral:post:123:replica:2`), route requests to different replicas.
- **Local cache layer:** Cache the hottest keys in application server memory (in-process cache) so they never hit even the distributed cache.

---

## 8. Cache Failure Modes

### Cache Miss Storm (Cold Start)
When a cache is first deployed or after a restart, it's empty. All requests are cache misses, hitting the database simultaneously. Can overwhelm the database.

**Solution:** Pre-warm the cache before routing traffic to it. Load the most popular items from the database into the cache proactively.

### Cache Avalanche
Multiple cache entries expire at the same time (e.g., all set with the same TTL). A wave of simultaneous misses floods the database.

**Solution:** Add TTL jitter — instead of `TTL=300`, use `TTL=300 + random(0-60)`. Expires stagger across time.

### Cache Penetration
Requests for data that doesn't exist in either cache or database — every request misses the cache and hits the database unnecessarily.

```
Attacker sends: GET user:999999999 (doesn't exist)
Cache: miss → Database: miss → returns null
Cache: miss → Database: miss → returns null  (repeated)
Database overwhelmed by lookups for non-existent keys
```

**Solution:** Cache negative results — if the database returns null, store `null` in the cache with a short TTL. Future requests for the same non-existent key are served from cache.

---

## 9. What to Cache and What Not To

**Good candidates for caching:**
- Expensive database queries with stable results (user profiles, product details)
- Aggregated data (total likes, view counts)
- Session data (fast auth on every request)
- API responses that don't change per-user (public feeds, trending content)
- Results of expensive computations (recommendations, search results)

**Poor candidates for caching:**
- Highly personalized data that's unique per user (defeats the purpose — low hit rate)
- Data that changes on every write (cache invalidation overhead exceeds cache benefit)
- Sensitive data requiring strict access control (cache keys might be predictable)
- Data where staleness is never acceptable (current account balances, inventory)

---

## 10. How Distributed Cache Connects to Other Building Blocks

```
Database ──────────────────────────────────────────────────────────────►
  Source of truth. Cache is always rebuilt from here if lost.

Distributed Cache ──────────────────────────────────────────────────────►
  Sits in front of the database.
  Absorbs the majority of read traffic.
  Populated via cache-aside, write-through, or write-behind.

Rate Limiter ───────────────────────────────────────────────────────────►
  Uses the cache (Redis) as its counter store.
  INCR + EXPIRE pattern lives in the cache layer.

Session Storage ────────────────────────────────────────────────────────►
  User sessions stored in cache with TTL.
  Every authenticated request checks cache first.

Message Queue (for write-behind) ───────────────────────────────────────►
  Write-behind caching queues DB updates asynchronously.
  Cache is written immediately; queue ensures DB eventually catches up.
```

---

## 11. Self-Check

1. What is the fundamental difference between a cache and a database? Why does this distinction matter?
2. What is locality of reference, and why does it make caching effective even with a small cache?
3. What is the difference between cache-aside and write-through? When would you choose write-through despite its higher write cost?
4. What is a cache stampede? Describe two ways to prevent it.
5. What is cache penetration, and how does caching negative results fix it?
6. Your application has 10 app servers. You're considering in-process (local) caching vs a shared Redis cache. What is the key disadvantage of local caching in this scenario?
7. A user updates their username. Your system uses cache-aside with TTL=600. Another user queries that profile 30 seconds later. What do they see, and is this acceptable? How would you change the design if it isn't?

---

## 12. References

| Resource | Why it's worth it |
|----------|-------------------|
| 📘 [Designing Data-Intensive Applications — Ch. 1 (Kleppmann)](https://dataintensive.net) | Caching in context of system reliability and performance |
| 🔧 [Redis Documentation — Persistence and Eviction](https://redis.io/docs/manual/eviction/) | How Redis manages memory and eviction policies |
| 📬 [ByteByteGo — Cache Design](https://bytebytego.com) | Visual breakdown of caching strategies and failure modes |
| 📝 [Facebook — Scaling Memcache](https://research.facebook.com/publications/scaling-memcache-at-facebook/) | How Facebook scaled caching to handle billions of requests — essential reading |

---

*⬅️ Previous: [Speed Overview](Speed.md) &nbsp;|&nbsp; ➡️ Next: [Sharded Counters](sharded-counters.md)*

---

<sub>Part of the <a href="../../README.md">System Design Foundations</a> study guide series — Building Blocks: Speed.</sub>