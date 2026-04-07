# 🗄️ Storage

> *"Every system ultimately comes down to data — creating it, reading it, updating it, and keeping it safe. Storage is where it lives."*

---

## The Problem This Group Solves

Every system stores data. The question isn't whether you need storage — it's what kind. Different data has different shapes, different access patterns, and different scale requirements. Picking the wrong storage building block is one of the most expensive architectural mistakes you can make, because changing it later means migrating data that's already in production.

The three storage building blocks cover three fundamentally different kinds of data:

- **Structured, relational data** with complex queries and transactions → Database
- **Simple lookups by key** at very high speed and scale → Key-Value Store
- **Large, unstructured files** like images, videos, and documents → Blob Store

Understanding which one fits which situation — and why — is what this group is about.

---

## The Components

| Building Block | Best for | Guide |
|---------------|----------|-------|
| **Databases** | Structured data, complex queries, transactions, relationships between entities | [Read →](databases.md) |
| **Key-Value Store** | Simple key → value lookups at extreme speed and scale, no complex queries | [Read →](key-value-store.md) |
| **Blob Store** | Large binary files — images, videos, audio, backups, model weights | [Read →](blob-store.md) |

---

## How to Choose Between Them

The most important question to ask about any data you need to store:

```
What does the access pattern look like?
        │
        ├── "I need to query by multiple fields, join tables,
        │    run aggregations, or enforce constraints"
        │         → Database (SQL or NoSQL depending on schema)
        │
        ├── "I always know the exact key I'm looking for —
        │    just give me the value as fast as possible"
        │         → Key-Value Store
        │
        └── "I'm storing a large file and need to retrieve
             it by ID — the content itself is opaque"
                  → Blob Store
```

A second question: how big is each record?

```
Bytes to kilobytes (text, metadata, user profiles)
    → Database or Key-Value Store

Kilobytes to megabytes (documents, small images)
    → Depends on access pattern — could be either

Megabytes to gigabytes (photos, videos, large files)
    → Almost always Blob Store
```

---

## They're Not Mutually Exclusive

Real systems almost always use multiple storage building blocks together. The pattern is extremely common:

```
Photo sharing app:

User profile data      → Database (structured, queryable, relational)
Session tokens         → Key-Value Store (fast lookup, short TTL)
Photo files            → Blob Store (large binary, retrieved by ID)
Photo metadata         → Database (title, location, timestamp, queryable)
Like counts            → Key-Value Store or Sharded Counters
```

The database holds the metadata that makes data findable and relatable. The blob store holds the actual content. The key-value store holds the fast-access data that doesn't need complex queries. Each does what it's best at.

---

## The Key Tradeoffs in This Group

**SQL vs NoSQL databases** — relational databases give you ACID transactions, complex queries, and strong consistency. NoSQL databases give you horizontal scale, flexible schemas, and eventual consistency. The choice follows from your data model and consistency requirements.

**Key-Value Store as primary vs secondary storage** — key-value stores are often used as caches in front of databases. But some systems use them as primary storage for specific data types (session state, feature flags, real-time counters) where the access pattern fits and durability requirements are met.

**Blob Store durability and cost** — blob storage is typically 11 nines (99.999999999%) durable through redundant replication, but retrieving objects has latency. For frequently accessed blobs, a CDN in front of the blob store is almost always necessary.

---

## When You Reach for This Group

Any system that persists data → **Database** is almost always present.

Your system needs session management, caching, or fast key lookups → **Key-Value Store**.

Your system handles user-uploaded files, media, or large documents → **Blob Store**.

You're estimating storage and the per-record size is in megabytes or more → **Blob Store** for the content, **Database** for the metadata.

---

*⬅️⬅️ Previous Group: [Traffic & Routing](../traffic-and-routing/TrafficAndRouting.md) &nbsp;| &nbsp; ➡️ Deep Dive this group starting with: [Databases](databases.md) | &nbsp; ➡️➡️ Next Group: [Speed](../speed/Speed.md)*