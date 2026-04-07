# 🗃️ Databases

> *"Every system stores data. The question is never whether you need a database — it's which one, and why."*

**⏱ Reading time:** ~15 minutes &nbsp;|&nbsp; **📍 Series:** Building Blocks &nbsp;|&nbsp; **🗂 Group:** Storage

---

## Table of Contents

1. [What Databases Do](#1-what-databases-do)
2. [Relational Databases (SQL)](#2-relational-databases-sql)
3. [Non-Relational Databases (NoSQL)](#3-non-relational-databases-nosql)
4. [SQL vs NoSQL — The Real Decision](#4-sql-vs-nosql--the-real-decision)
5. [Data Replication](#5-data-replication)
6. [Data Partitioning (Sharding)](#6-data-partitioning-sharding)
7. [Indexes — How Databases Find Data Fast](#7-indexes--how-databases-find-data-fast)
8. [ACID vs BASE](#8-acid-vs-base)
9. [Database Trade-offs in System Design](#9-database-trade-offs-in-system-design)
10. [Self-Check](#10-self-check)
11. [References](#11-references)

---

## 1. What Databases Do

A database is a system for storing, retrieving, and managing structured data reliably. The "reliably" part is what distinguishes a database from just writing files to disk — databases provide guarantees about durability (written data stays written), consistency (data doesn't get corrupted), and queryability (you can find data efficiently without scanning everything).

Four fundamental operations underlie everything a database does:

```
Create  → INSERT INTO users (name, email) VALUES ('Isa', 'isa@email.com')
Read    → SELECT * FROM users WHERE id = 123
Update  → UPDATE users SET email = 'new@email.com' WHERE id = 123
Delete  → DELETE FROM users WHERE id = 123
```

These are CRUD operations — the atoms of every database interaction. Everything else (transactions, replication, indexing, partitioning) exists to make these four operations faster, more reliable, or able to scale.

---

## 2. Relational Databases (SQL)

Relational databases store data in **tables** — rows and columns, like a spreadsheet with structure and rules. The "relational" refers to how tables relate to each other through foreign keys.

```
users table:
  id | name  | email
  1  | Isa   | isa@email.com
  2  | Alex  | alex@email.com

orders table:
  id | user_id | total  | status
  1  | 1       | 49.99  | shipped
  2  | 1       | 12.50  | pending
  3  | 2       | 99.00  | shipped

Relationship: orders.user_id → users.id
```

You can query across tables with a JOIN:
```sql
SELECT users.name, orders.total
FROM users
JOIN orders ON users.id = orders.user_id
WHERE orders.status = 'shipped'
```

### ACID Guarantees

The reason relational databases are trusted for critical data:

**Atomicity** — a transaction either fully succeeds or fully fails. No partial states.
```
Transfer $500: debit Alice AND credit Bob
If anything fails → both operations roll back
Money is never "lost in the middle"
```

**Consistency** — the database enforces rules (constraints, foreign keys) so data is always valid.
```
orders.user_id must reference a real user
account.balance cannot go below 0
email must be unique
```

**Isolation** — concurrent transactions don't interfere with each other. Reading data while it's being written gives a consistent view.

**Durability** — once a transaction commits, it's permanent. Even if the server crashes immediately after, the data survives (via write-ahead log).

### When SQL is the right choice

- Data has clear relationships (users → orders → products)
- You need complex queries, aggregations, joins
- Transactions are required (financial systems, inventory)
- Schema is relatively stable and well-defined
- Scale is manageable on one machine or a small cluster

**Real-world examples:** PostgreSQL, MySQL, SQLite, Oracle, SQL Server

---

## 3. Non-Relational Databases (NoSQL)

NoSQL databases ("Not Only SQL") trade some relational features for scale, flexibility, or performance. There are four main types, each designed for a different data model.

### Document Stores
Store data as JSON-like documents. Each document can have a different structure. No rigid schema.

```json
{
  "id": "user_123",
  "name": "Isa",
  "email": "isa@email.com",
  "preferences": {
    "theme": "dark",
    "notifications": true
  },
  "recent_searches": ["system design", "kafka", "redis"]
}
```

Documents are grouped in **collections** (not tables). Each document is self-contained — no joins needed for data that belongs together.

**Good for:** Content management, user profiles, product catalogs, anything with flexible or nested structure.
**Real-world examples:** MongoDB, CouchDB, Firestore

---

### Key-Value Stores
The simplest NoSQL model. Store and retrieve values by a unique key. No structure imposed on the value.

```
key: "session:user_123"     value: {"token": "abc", "expires": 1234567890}
key: "cache:product:456"    value: {"name": "Keyboard", "price": 89.99}
key: "counter:page_views"   value: 4829103
```

Extremely fast for simple lookups. Can't query by value fields — only by key.

**Good for:** Sessions, caches, feature flags, simple counters.
**Real-world examples:** Redis, DynamoDB, Memcached

*(Key-Value Stores get their own dedicated guide — see [Key-Value Store](key-value-store.md))*

---

### Column-Family Stores
Store data in rows with dynamic columns, grouped into column families. Optimized for writes and for queries that read a large range of rows but only a few columns.

```
Row key: "user_123"
  Column family "profile":  name="Isa", email="isa@email.com"
  Column family "activity": last_login="2024-01-15", posts=47
```

**Good for:** Time-series data, IoT sensor readings, activity feeds, anything with heavy write volume and wide rows.
**Real-world examples:** Cassandra, HBase, ScyllaDB

---

### Graph Databases
Store data as nodes (entities) and edges (relationships). Optimized for traversing relationships — finding connections between data points.

```
Node: User(Isa) → FOLLOWS → User(Alex)
Node: User(Isa) → LIKES → Post(System Design Guide)
Node: Post(System Design Guide) → TAGGED_WITH → Topic(Distributed Systems)

Query: "Find all posts liked by people Isa follows"
→ Trivial in a graph database, complex and slow in relational
```

**Good for:** Social networks, recommendation engines, fraud detection, knowledge graphs.
**Real-world examples:** Neo4j, Amazon Neptune

---

## 4. SQL vs NoSQL — The Real Decision

This is one of the most common system design questions. The answer is never "NoSQL is better" or "SQL is better" — it's always "which fits this specific use case?"

```
Ask these questions:

Is the data structure well-defined and stable?
  YES → SQL likely fits
  NO  → NoSQL (document store) offers flexibility

Do you need complex queries across multiple entities?
  YES → SQL (JOINs, aggregations are natural)
  NO  → NoSQL may be simpler

Do you need ACID transactions?
  YES → SQL (or specific NoSQL with transaction support)
  NO  → NoSQL flexibility available

Does this need to scale beyond one server horizontally?
  YES → NoSQL scales out more naturally
  NO  → SQL scales vertically well

What's the access pattern?
  Always by known key?     → Key-Value Store
  Rich queries by field?   → SQL or Document Store
  Relationship traversal?  → Graph Database
  Heavy writes, wide rows? → Column-Family Store
```

**The most common answer in practice:** Use PostgreSQL by default. It handles most use cases well, has excellent tooling, and supports JSON columns for flexible data when needed. Switch to a specialized NoSQL store when you have a specific reason — scale, access pattern, or data model that SQL handles poorly.

---

## 5. Data Replication

A single database server is a single point of failure. Replication creates copies of data across multiple servers for durability and availability.

### Primary-Replica Replication (Leader-Follower)

One server (primary/leader) accepts all writes. Changes are replicated to replica servers (followers). Replicas serve reads.

```
Writes → Primary
          │
          ├──► Replica 1 (async or sync replication)
          ├──► Replica 2
          └──► Replica 3

Reads  → Any replica (or primary for strong consistency)
```

**Advantages:**
- Read throughput scales — add replicas to handle more reads
- High availability — if primary fails, promote a replica
- Geographic distribution — replicas in different regions serve local users

**Tradeoffs:**
- **Replication lag** — replicas may be milliseconds or seconds behind the primary. Reading from a replica immediately after a write may return stale data.
- **Primary is still a write bottleneck** — all writes go to one node

### Synchronous vs Asynchronous Replication

**Synchronous:** Primary waits for at least one replica to confirm before acknowledging the write.
```
Write → Primary → waits → Replica confirms → acknowledge to client
Benefit: No data loss if primary fails (replica has the write)
Cost: Higher write latency (must wait for network round-trip to replica)
```

**Asynchronous:** Primary acknowledges immediately, replicates in the background.
```
Write → Primary → acknowledge to client (immediately)
              └──► Replica (background, later)
Benefit: Lower write latency
Cost: Small window where primary has data replica doesn't — data loss possible if primary crashes
```

Most production databases use asynchronous replication with the option to require synchronous confirmation for critical writes.

### Multi-Primary Replication (Multi-Leader)

Multiple nodes can accept writes. Useful for multi-region deployments where you want writes to land locally.

```
Region A: Primary A accepts writes
Region B: Primary B accepts writes
Both replicate to each other

Challenge: What if both primaries update the same record simultaneously?
→ Write conflict — must be resolved (last-write-wins, application-level merge, etc.)
```

Multi-primary introduces write conflicts that are complex to resolve. Used carefully for specific high-availability or geographic requirements.

---

## 6. Data Partitioning (Sharding)

When a single database can't hold all the data or handle all the write throughput, you split (shard) data across multiple database instances, each owning a subset.

```
Without sharding:           With sharding:
  1 DB — all data             Shard 1: users 1–1M
  (ceiling on capacity)       Shard 2: users 1M–2M
                              Shard 3: users 2M–3M
                              (can add shards as needed)
```

### Partitioning Strategies

**Range partitioning** — split by a range of values.
```
user_id 1–1,000,000     → Shard 1
user_id 1,000,001–2,000,000 → Shard 2
```
Simple to understand. Risk: hot spots if one range gets disproportionate traffic (all new users land on the latest shard).

**Hash partitioning** — hash the key and distribute by hash value.
```
shard = hash(user_id) % num_shards
user 123: hash(123) % 3 = 0 → Shard 1
user 456: hash(456) % 3 = 1 → Shard 2
user 789: hash(789) % 3 = 2 → Shard 3
```
Distributes evenly. Problem: when you add a shard, most keys remap — requires data migration.

**Consistent hashing** — a variant of hash partitioning where adding/removing nodes only remaps a fraction of keys. Used by DynamoDB, Cassandra, Redis Cluster.

### The Pain of Sharding

Sharding solves scale but introduces significant complexity:

**Cross-shard queries are expensive.** A query that needs data from multiple shards must query each shard and merge results — much slower than a single-shard query.

**Cross-shard transactions are hard.** ACID transactions across multiple shards require distributed transaction protocols (two-phase commit), which are slow and complex.

**Rebalancing is painful.** When you add shards, data must migrate between nodes while the system stays live.

Sharding should be a last resort — when vertical scaling and read replicas are no longer sufficient. Many systems delay sharding as long as possible by aggressively caching and optimizing queries.

---

## 7. Indexes — How Databases Find Data Fast

Without an index, finding a row in a database requires scanning every row — O(n). For a table with 100 million rows, that's catastrophically slow.

An index is a separate data structure (usually a B-tree) that maps column values to row locations, enabling O(log n) lookups.

```
Query: SELECT * FROM orders WHERE user_id = 12345

Without index: scan all 50M orders, check each user_id → slow
With index on user_id: look up 12345 in B-tree → jump directly to matching rows → fast
```

### The Index Tradeoff

Indexes make reads faster but writes slower — every INSERT, UPDATE, or DELETE must also update all relevant indexes.

```
INSERT a new order:
  Without index: write one row to orders table
  With index on user_id: write one row + update user_id index
  With indexes on user_id, status, created_at: write one row + update 3 indexes
```

**The rule:** Index columns you query by frequently. Don't index everything — over-indexing slows writes and wastes storage.

### Types of Indexes

**Primary key index** — automatically created, uniquely identifies each row.

**Secondary index** — on non-primary-key columns you query frequently.

**Composite index** — on multiple columns together. Useful when you always query by a specific combination.
```
Index on (user_id, created_at) supports:
  WHERE user_id = 123 AND created_at > '2024-01-01'
  Much faster than two separate indexes
```

**Covering index** — includes all columns a query needs, so the query can be answered from the index alone without touching the actual table.

---

## 8. ACID vs BASE

Two philosophies for how databases handle consistency and availability.

**ACID** (Atomicity, Consistency, Isolation, Durability) — the relational database model. Strong guarantees, predictable behavior, but harder to scale horizontally.

**BASE** (Basically Available, Soft state, Eventually consistent) — the NoSQL philosophy. Trade strong consistency for availability and scale.

```
ACID:                           BASE:
"The data is always correct"    "The data will become correct"
Strong consistency              Eventual consistency
Harder to distribute            Easier to distribute
Banks, financial systems        Social media, analytics, feeds
```

This maps directly to what we covered in [Consistency Models](../../preliminary-system-design-concepts/consistencyModels.md). ACID databases are CP systems. BASE databases are AP systems. The CAP theorem applies here — choosing BASE means explicitly accepting eventual consistency in exchange for availability and partition tolerance.

---

## 9. Database Trade-offs in System Design

When you introduce a database in a design, be ready to discuss these tradeoffs:

| Decision | Options | Tradeoff |
|----------|---------|---------|
| SQL vs NoSQL | Relational vs document/KV/graph | Structure & transactions vs scale & flexibility |
| Replication | Single vs primary-replica vs multi-primary | Simplicity vs availability vs write conflicts |
| Consistency | Sync vs async replication | Durability vs write latency |
| Partitioning | No sharding vs sharding | Simplicity vs scale |
| Shard key | Range vs hash vs consistent hash | Simplicity vs hot spots vs rebalancing cost |
| Indexing | No index vs index | Write speed vs read speed |
| Read routing | All reads to primary vs replicas | Strong consistency vs read scalability |

The pattern across all of these: **more sophistication buys scale or availability, but at the cost of complexity and often consistency.** The right choice is the simplest one that meets your actual requirements.

---

## 10. Self-Check

1. What are the four ACID properties? Give a real-world example where violating each one would cause a problem.
2. What is the difference between a document store and a relational database? When would you choose one over the other?
3. What is replication lag, and when does it cause problems?
4. What is the difference between range partitioning and hash partitioning? What problem does consistent hashing solve?
5. Why do indexes make reads faster but writes slower?
6. You're designing a social media platform. Users have profiles, posts, follows, and likes. Which data would you store in a relational database vs a NoSQL store, and why?
7. A database query that takes 50ms normally suddenly takes 5 seconds after the table grows from 1 million to 100 million rows. What is the most likely cause, and how do you fix it?

---

## 11. References

| Resource | Why it's worth it |
|----------|-------------------|
| 📘 [Designing Data-Intensive Applications — Ch. 3, 5, 6 (Kleppmann)](https://dataintensive.net) | The definitive treatment of storage engines, replication, and partitioning |
| 🔧 [PostgreSQL Documentation — Indexes](https://www.postgresql.org/docs/current/indexes.html) | How B-tree indexes actually work in production |
| 📬 [ByteByteGo — SQL vs NoSQL](https://bytebytego.com) | Visual comparison of database types and when to use each |
| 📝 [Martin Fowler — NoSQL Distilled](https://martinfowler.com/books/nosql.html) | Clear framing of the four NoSQL categories and their tradeoffs |

---

*⬅️ Previous: [Storage Overview](Storage.md) &nbsp;|&nbsp; ➡️ Next: [Key-Value Store](key-value-store.md)*

---

<sub>Part of the <a href="../../README.md">System Design Foundations</a> study guide series — Building Blocks: Storage.</sub>