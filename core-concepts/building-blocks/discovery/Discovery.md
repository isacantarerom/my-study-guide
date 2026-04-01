# 🔍 Discovery

> *"Storage keeps data. Discovery makes it findable."*

---

## The Problem This Group Solves

Storing data is a solved problem at every scale. The harder problem is finding specific data inside a massive dataset — fast, relevantly, across fields that weren't the primary key.

A database can find a user by user_id in milliseconds. But "find all posts about distributed systems written in the last week that are relevant to someone interested in Kafka" — that's a query a relational database will struggle with at scale, and will fail at entirely once the data grows large enough.

Distributed search exists to make arbitrary content findable across massive datasets. It's what powers every search box you've ever used — Google, Spotify, LinkedIn, GitHub, Airbnb. And it's one of the most architecturally interesting building blocks because it requires solving hard problems in indexing, ranking, and distributed coordination simultaneously.

---

## The Component

| Building Block | Solves | Guide |
|---------------|--------|-------|
| **Distributed Search** | Finding relevant content across massive datasets using full-text, faceted, or semantic search | [Read →](distributed-search.md) |

---

## Why Search Is Its Own Category

Search is distinct from storage in a fundamental way: **storage is organized around how data is written; search is organized around how data is read.**

A database stores a blog post with a primary key of `post_id`. To find posts about "distributed systems," it has to scan every post and check whether the text contains those words — O(n) at best, impossibly slow at scale.

A search index inverts this. It builds a map of **word → list of documents containing that word**. Finding posts about "distributed systems" becomes two fast lookups in this inverted index, then intersecting the results — O(log n) regardless of dataset size.

```
Database (optimized for writes):      Search Index (optimized for reads):

post_id → { title, body, author }     "distributed" → [post_3, post_7, post_42, ...]
                                       "systems"     → [post_1, post_7, post_19, ...]
Finding "distributed systems":        
  Scan all posts. Check each body.    Finding "distributed systems":
  O(n) — slow at scale.               Intersect both lists → [post_7, ...]
                                       O(log n) — fast at any scale.
```

This inversion is the core insight behind all search systems. Everything else — ranking, faceting, autocomplete, fuzzy matching — is built on top of this fundamental data structure.

---

## Where Distributed Search Fits in the Larger System

Search is almost never the primary storage layer — it's a secondary index built on top of primary storage:

```
User creates content
        │
        ├──► Primary storage (Database / Blob Store)
        │    Source of truth — data lives here
        │
        └──► Search index (async, eventually consistent)
             Makes content findable by arbitrary queries
             Rebuilt from primary storage if corrupted

User searches for content
        │
        ▼
Search Index returns IDs of matching documents
        │
        ▼
Primary storage fetches full records by ID
        │
        ▼
Results returned to user
```

This architecture means search can be rebuilt from scratch if needed — the source of truth is always the primary storage. Search is the access layer, not the storage layer.

---

## The Key Challenges in Distributed Search

**Index freshness** — new content needs to appear in search results quickly. But rebuilding the entire index on every write is impossibly slow. Incremental indexing with merge strategies is the solution.

**Relevance ranking** — returning documents that contain the search terms is easy. Returning them in the right order (most relevant first) is hard. TF-IDF, BM25, and machine-learned ranking models are the solutions, each with different tradeoffs.

**Scale** — at billions of documents, no single machine holds the entire index. Sharding the index across machines introduces coordination complexity for queries that must touch multiple shards.

**Fault tolerance** — if an index shard goes down, searches that touch it fail. Replicating each shard across multiple nodes provides redundancy at the cost of storage and write amplification.

---

## When You Reach for This Group

Your system lets users search for content by arbitrary keywords or phrases → **Distributed Search**.

Your system needs faceted filtering ("show me all red shoes under $100 in size 10") → **Distributed Search**.

Your database queries for text content are becoming slow as data grows → **Distributed Search** as a secondary index.

Your system needs autocomplete or "did you mean?" functionality → **Distributed Search**.

Your system does not need arbitrary text search and always retrieves by known keys → Primary storage (Database or Key-Value Store) is sufficient.

---

*⬅️ Previous Group: [Observability](../observability/Observability.md) &nbsp;|&nbsp; ➡️ Back to [Building Blocks](../BuildingBlocks.md)*