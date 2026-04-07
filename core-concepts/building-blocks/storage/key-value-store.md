# 🔑 Key-Value Store

> *"Sometimes the most powerful data structure is the simplest one: a key, and a value."*

**⏱ Reading time:** ~11 minutes &nbsp;|&nbsp; **📍 Series:** Building Blocks &nbsp;|&nbsp; **🗂 Group:** Storage

---

## Table of Contents

1. [What a Key-Value Store Is](#1-what-a-key-value-store-is)
2. [The Data Model — Deceptively Simple](#2-the-data-model--deceptively-simple)
3. [What Makes KV Stores Fast](#3-what-makes-kv-stores-fast)
4. [Core Operations and Their Guarantees](#4-core-operations-and-their-guarantees)
5. [Designing Good Keys](#5-designing-good-keys)
6. [TTL — Time-Bounded Storage](#6-ttl--time-bounded-storage)
7. [Replication and Consistency in KV Stores](#7-replication-and-consistency-in-kv-stores)
8. [Partitioning in KV Stores](#8-partitioning-in-kv-stores)
9. [Common Use Cases](#9-common-use-cases)
10. [When NOT to Use a Key-Value Store](#10-when-not-to-use-a-key-value-store)
11. [Redis — The Most Important KV Store to Know](#11-redis--the-most-important-kv-store-to-know)
12. [Self-Check](#12-self-check)
13. [References](#13-references)

---

## 1. What a Key-Value Store Is

A key-value store is the simplest possible database model: every piece of data has a unique key, and retrieving that data requires knowing the key. No tables, no schemas, no joins, no complex queries.

```
Put("user:123:session", {"token": "abc", "expires": 1705000000})
Get("user:123:session") → {"token": "abc", "expires": 1705000000}
Delete("user:123:session")
```

That's essentially the entire interface. What you sacrifice in query power, you gain in raw speed and horizontal scalability. When your access pattern is always "give me the thing with this exact key," a key-value store is faster and simpler than any relational database.

The tradeoff is strict: **if you don't know the exact key, you can't find the data efficiently.** There's no `SELECT * FROM sessions WHERE expires < now()` — you'd have to know the keys of all sessions. Key-value stores are optimized for one operation and one operation only: exact key lookup.

---

## 2. The Data Model — Deceptively Simple

The key is always a string. The value can be almost anything — a string, a number, a JSON blob, a list, a set, a bitmap. Different KV stores support different value types.

```
Simple values:
  "config:feature_flag:dark_mode" → "true"
  "counter:daily_active_users"    → 47293

Structured values (stored as JSON string):
  "user:123:profile" → '{"name":"Isa","email":"isa@email.com","tier":"pro"}'

Rich types (Redis-specific):
  "feed:user:123" → [post_id_1, post_id_2, post_id_3]  (List)
  "online_users"  → {user_1, user_7, user_42}            (Set)
  "leaderboard"   → {("alice", 9500), ("bob", 8200)}     (Sorted Set)
```

Redis in particular provides rich data structures (lists, sets, hashes, sorted sets, bitmaps, streams) that enable patterns far beyond simple get/set — we cover this in Section 11.

---

## 3. What Makes KV Stores Fast

Speed is the primary reason to choose a key-value store. Understanding why they're fast tells you when they're the right tool.

**In-memory storage:** The most common KV stores (Redis, Memcached) store data entirely in RAM. RAM access is ~100 nanoseconds. Disk access is ~100 microseconds (SSD) to ~10 milliseconds (HDD). Memory is 1,000-100,000× faster.

**Simple data structures:** No B-trees, no query planning, no join execution. A hash map lookup is O(1) — constant time regardless of how many keys exist.

**No schema overhead:** No validation, no constraint checking, no index updates on write. Write the bytes, done.

**Connection pooling and pipelining:** Redis processes commands in a single thread (avoiding lock contention) and supports pipelining — sending multiple commands in one network round trip.

```
Latency comparison for a simple lookup:
  PostgreSQL (indexed):  ~1-5ms
  Redis (in-memory):     ~0.1-0.5ms
  Local hash map:        ~0.001ms (nanoseconds)

Redis is 10-50× faster than a relational database for simple lookups.
```

The flip side: RAM is limited and expensive. You can't store terabytes of data in Redis the way you can in Cassandra or S3. Key-value stores are optimized for **hot data** — the subset of your data that's accessed most frequently and needs to be fast.

---

## 4. Core Operations and Their Guarantees

Most key-value stores support a small set of operations:

```
GET key           → returns value (or null if not found)
SET key value     → stores value at key (overwrites if exists)
DELETE key        → removes key-value pair
EXISTS key        → returns boolean — does this key exist?
EXPIRE key ttl    → sets a time-to-live after which key auto-deletes
TTL key           → returns remaining time before expiry
```

For concurrent operations, the critical guarantee is **atomicity of single operations**. A GET or SET on a single key is atomic — no partial reads or writes. Multi-key operations are not automatically atomic unless wrapped in a transaction.

### Atomic Increment — A Critical Pattern

```
INCR counter:page_views  → returns new value (atomically incremented)
INCRBY counter:likes 5   → atomically adds 5, returns new value
```

This is crucial. Without atomic increment, two processes could both read a counter (say, 100), both add 1, and both write back 101 — losing one increment. `INCR` is atomic — it reads, increments, and writes in one uninterruptible operation.

This is the foundation of distributed counters, rate limiters, and idempotency key tracking.

---

## 5. Designing Good Keys

In a key-value store, your key structure is your schema. There are no tables or columns to organize data — the key is the only way to find anything. Good key design is therefore one of the most important skills in using KV stores effectively.

**Hierarchical, colon-separated naming is the standard convention:**

```
{type}:{id}:{attribute}

user:123:session
user:123:profile
user:123:preferences

product:456:details
product:456:inventory

rate_limit:api_key:xyz:minute:1420530
```

This convention:
- Makes keys self-documenting (you can read a key and know what it contains)
- Groups related keys with a common prefix (useful for scanning patterns)
- Prevents collisions between different data types

**Key length tradeoffs:** Longer descriptive keys are readable; shorter keys save memory. At millions of keys, the key itself is a non-trivial fraction of memory usage.

**Avoid putting variable data in the key when it's also in the value.** If your value already contains the user ID, don't repeat it redundantly in the key pattern unless you need to.

---

## 6. TTL — Time-Bounded Storage

One of the most powerful features of KV stores is TTL — the ability to set an expiration time on any key. The store automatically deletes the key when it expires.

```
SET session:user:123 {token_data} EX 3600  ← expires in 3600 seconds (1 hour)

After 1 hour:
GET session:user:123 → null (automatically deleted)
```

TTL eliminates the need for explicit deletion in many use cases:

**Sessions** — set TTL to session timeout. No cleanup job needed; sessions auto-expire.

**Rate limiting** — set counter with TTL equal to the rate limit window. Counter auto-resets when the window expires.

**Caching** — set TTL to acceptable staleness period. Cache auto-invalidates. No explicit purge logic.

**OTP codes** — set TTL to code validity window. Code auto-expires. No cleanup needed.

**Idempotency keys** — store request IDs with TTL equal to retry window. After window closes, key expires and the slot is reused.

TTL is what makes KV stores practical as temporary storage — you don't accumulate unbounded data because old entries clean themselves up.

---

## 7. Replication and Consistency in KV Stores

Like databases, KV stores need replication for durability and availability.

### Redis Replication

Redis uses primary-replica replication. One primary accepts writes; replicas receive copies asynchronously.

```
Write → Primary Redis
              │ async replication
              ├──► Replica 1
              └──► Replica 2

Reads → Any replica (or primary for strong consistency)
```

**Redis Sentinel** — monitors the primary and automatically promotes a replica if the primary fails. Provides automatic failover without manual intervention.

**Redis Cluster** — shards data across multiple primary nodes, each with their own replicas. Both horizontal scale and high availability.

### The Durability Tradeoff in Redis

By default, Redis is an in-memory store. If the process crashes, all data in memory is lost. For use cases where this is acceptable (caches, sessions that can be re-created), this is fine. For use cases where durability matters, Redis offers two persistence mechanisms:

**RDB (Redis Database)** — periodic snapshots to disk. Fast, compact. Risk: data between snapshots is lost on crash.

**AOF (Append-Only File)** — logs every write operation to disk. Slower writes, but minimal data loss on crash. Replaying the log restores full state.

**The practical choice:** Many production systems use Redis as a cache (loss-tolerant) and a separate durable database for source-of-truth data. Redis stores the fast-access copy; the database stores the permanent record.

---

## 8. Partitioning in KV Stores

When a single KV node can't hold all data, you partition (shard) across multiple nodes.

**Redis Cluster** uses hash partitioning with consistent hashing across 16,384 hash slots:
```
hash_slot = CRC16(key) % 16384
Slot 0-5460    → Node 1
Slot 5461-10922 → Node 2
Slot 10923-16383 → Node 3
```

**DynamoDB** uses consistent hashing internally, fully managed — you don't configure it.

The benefit of consistent hashing: when a node is added or removed, only the keys that *must* move (those on adjacent virtual nodes) are remapped. Most keys stay where they are. This makes scaling smoother than simple modulo hashing.

---

## 9. Common Use Cases

Key-value stores appear in virtually every large-scale system. These are the patterns you'll reach for:

### Session Storage
```
Login → generate session token → SET session:{token} {user_data} EX 3600
Request → GET session:{token} → validates session, returns user context
Logout → DELETE session:{token}
```

### Caching Database Queries
```
// Check cache first
cached = GET cache:user:123:profile
if cached: return parse(cached)

// Cache miss — fetch from DB
profile = db.query("SELECT * FROM users WHERE id = 123")
SET cache:user:123:profile serialize(profile) EX 300  // cache 5 minutes
return profile
```

### Distributed Rate Limiting
```
// Redis atomic increment for rate limiting
count = INCR rate:user:123:minute:1420530
EXPIRE rate:user:123:minute:1420530 60  // window expires in 60 seconds
if count > 100: reject  // over limit
```

### Feature Flags
```
SET feature:dark_mode:enabled "true"
SET feature:new_checkout:enabled "false"

// Application checks flag
if GET feature:dark_mode:enabled == "true": show_dark_mode()
```

### Leaderboard (Redis Sorted Set)
```
ZADD leaderboard 9500 "alice"
ZADD leaderboard 8200 "bob"
ZADD leaderboard 9800 "carol"

ZREVRANGE leaderboard 0 9 WITHSCORES
→ carol:9800, alice:9500, bob:8200  (top 10, sorted by score)
```

### Pub/Sub Messaging (Redis)
Redis supports lightweight publish/subscribe for real-time messaging between services.

---

## 10. When NOT to Use a Key-Value Store

Understanding the limits is as important as knowing the use cases.

**Don't use a KV store when:**

- You need to query by something other than the exact key (use a database with indexes)
- You need complex relationships and JOINs (use a relational database)
- You need strong ACID transactions across multiple keys (use a relational database)
- Data volume exceeds available RAM and the data is not hot (use disk-based storage)
- You need full-text search (use a search engine)

**The pattern that trips people up:** Using a KV store as a primary database when the access patterns later require queries the KV store can't serve. You can GET by key efficiently; you cannot "find all users who signed up last week" without knowing their keys or scanning everything.

---

## 11. Redis — The Most Important KV Store to Know

Redis is the most widely used KV store in production systems. Beyond basic GET/SET, it provides data structures that enable powerful patterns:

| Data Structure | Operations | Use Case |
|---------------|-----------|---------|
| **String** | GET, SET, INCR | Counters, flags, simple values |
| **Hash** | HGET, HSET, HMGET | User profiles, object fields |
| **List** | LPUSH, RPOP, LRANGE | Queues, activity feeds, recent items |
| **Set** | SADD, SMEMBERS, SINTER | Unique visitors, tags, intersection queries |
| **Sorted Set** | ZADD, ZRANGE, ZRANK | Leaderboards, priority queues, time-ordered data |
| **Bitmap** | SETBIT, GETBIT, BITCOUNT | Daily active user tracking, feature toggles at scale |
| **Stream** | XADD, XREAD | Event logs, message queues with consumer groups |

Redis also supports **Lua scripting** for atomic multi-operation sequences, and **transactions** (MULTI/EXEC) for grouping commands.

---

## 12. Self-Check

1. What is the fundamental tradeoff of a key-value store compared to a relational database?
2. Why are in-memory KV stores like Redis 10-50× faster than disk-based databases for simple lookups?
3. What is the standard key naming convention, and why does it matter?
4. What does TTL enable, and give three use cases where it eliminates a cleanup problem?
5. By default Redis is in-memory. What are the two persistence mechanisms, and what does each trade off?
6. You're building a rate limiter. A client can make 100 requests per minute. How would you implement this with Redis, including the atomic operation that makes it correct?
7. A social media app needs to track the top 10 most-liked posts in real-time. Which Redis data structure would you use, and what operations would you run on each like event?

---

## 13. References

| Resource | Why it's worth it |
|----------|-------------------|
| 🔧 [Redis Documentation](https://redis.io/docs/) | Authoritative reference for all data types and commands |
| 📘 [Designing Data-Intensive Applications — Ch. 3](https://dataintensive.net) | Storage engines and the spectrum from KV to relational |
| 📬 [ByteByteGo — Redis Use Cases](https://bytebytego.com) | Visual breakdown of Redis patterns in production systems |
| 📝 [Redis University — Free Courses](https://university.redis.com) | Hands-on Redis learning — worth doing the RU101 course |

---

*⬅️ Previous: [Databases](databases.md) &nbsp;|&nbsp; ➡️ Next: [Blob Store](blob-store.md)*

---

<sub>Part of the <a href="../../README.md">System Design Foundations</a> study guide series — Building Blocks: Storage.</sub>