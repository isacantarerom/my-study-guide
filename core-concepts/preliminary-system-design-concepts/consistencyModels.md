# 🎚️ Spectrum of Consistency Models

> *"Consistency isn't binary. It's a dial — and where you set it determines how your system behaves under pressure."*

**⏱ Reading time:** ~13 minutes &nbsp;|&nbsp; **📍 Series:** Grokking System Design &nbsp;|&nbsp; **🗂 Topic #3:** Preliminary Concepts

---

## Table of Contents

1. [Why Consistency Is a Spectrum](#1-why-consistency-is-a-spectrum)
2. [The Core Problem: Replication](#2-the-core-problem-replication)
3. [The Consistency Models — From Strongest to Weakest](#3-the-consistency-models--from-strongest-to-weakest)
4. [The CAP Theorem Revisited](#4-the-cap-theorem-revisited)
5. [PACELC: The More Complete Picture](#5-pacelc-the-more-complete-picture)
6. [Consistency in Practice: Real Systems](#6-consistency-in-practice-real-systems)
7. [How to Choose a Consistency Model](#7-how-to-choose-a-consistency-model)
8. [Applying This in an Interview](#8-applying-this-in-an-interview)
9. [Worked Example: Social Media Feed vs. Bank Transfer](#9-worked-example-social-media-feed-vs-bank-transfer)
10. [Self-Check](#10-self-check)
11. [References](#11-references)

---

## 1. Why Consistency Is a Spectrum

In the [RPC guide](networkAbstraction.md), we saw that distributed systems introduce failure modes that don't exist on a single machine. One of the most profound of these is the **consistency problem**: when data is stored on multiple machines, how do you guarantee that every reader sees the same version of that data?

The naive answer is "always show the latest data." The distributed systems answer is: *that's extremely expensive, and often unnecessary.*

Consistency is not binary. It exists on a spectrum from **strongest** (every read sees the most recent write, guaranteed) to **weakest** (reads may see stale data, but the system is fast and always available).

**Every consistency model is a tradeoff between three things:**
- How fresh the data is that readers see
- How fast reads and writes can complete
- How available the system is during failures

There is no free lunch. Understanding the spectrum is how you justify your design choices in an interview.

---

## 2. The Core Problem: Replication

Consistency problems arise because of **replication** — storing copies of data on multiple nodes for durability and availability.

```
         Write: "balance = $500"
                    │
                    ▼
         ┌─────────────────┐
         │   Primary Node  │  ← Write lands here first
         │  balance = $500 │
         └────────┬────────┘
                  │  Replicate...
         ┌────────┴────────┐
         │                 │
         ▼                 ▼
  ┌─────────────┐   ┌─────────────┐
  │  Replica A  │   │  Replica B  │
  │balance = ?  │   │balance = ?  │
  └─────────────┘   └─────────────┘

  If a reader hits Replica A or B before replication completes,
  what do they see? That's the consistency problem.
```

Replication takes time. Even on fast networks, there is a propagation delay between when a write lands on the primary and when all replicas reflect it. Every consistency model is essentially a set of rules governing what happens in that window.

---

## 3. The Consistency Models — From Strongest to Weakest

Think of this as a dial. Turning it toward **strong** gives you accuracy at the cost of speed and availability. Turning it toward **weak** gives you speed and availability at the cost of accuracy.

```
STRONG ◄────────────────────────────────────────────────► WEAK

  Linearizable    Sequential    Causal    Eventual    Weak
  (Strict)        Consistency   Consistency  Consistency  Consistency
```

---

### 🔴 Linearizability (Strict Consistency)

**The gold standard. The strongest guarantee.**

Every read reflects the most recent write, and operations appear to happen instantaneously in a single global order. From any client's perspective, the system behaves as if there is only one copy of the data.

```
Timeline:
  Client A writes x = 5   ──────────────────────►
  Client B reads x         ────────────────────────► must see x = 5
                                                      (no exceptions)
```

**What it costs:** To achieve this, the system must coordinate across nodes on every write — typically using a consensus protocol like Paxos or Raft. This adds latency to every operation and means the system may be unavailable during a network partition.

**When to use it:** Financial transactions, leader election, any operation where returning stale data would be catastrophic.

**Real-world examples:** Google Spanner, Zookeeper, etcd.

---

### 🟠 Sequential Consistency

**Strong, but slightly relaxed.**

All operations appear to execute in *some* sequential order that is consistent across all clients — but that order doesn't have to match real-world wall-clock time. All clients agree on the same order; it just might not be the "real" order events happened in.

```
Client A:  write x=1, write x=2
Client B:  reads may see x=1 then x=2 (correct order preserved)
           but the exact timing of when B sees the update is not guaranteed
```

**Analogy:** A bulletin board in a shared office. Everyone reads the same board, and posts stay in the order they were pinned — but there might be a delay before a new post appears on the board for everyone.

**When to use it:** Collaborative tools, shared state that needs to be consistent across users but doesn't require microsecond precision.

---

### 🟡 Causal Consistency

**Preserves cause-and-effect. Lets unrelated things diverge.**

If operation A causally affects operation B (e.g., a reply to a message), then all nodes must show A before B. Operations with no causal relationship can appear in any order.

```
User A posts: "Who wants pizza?"        ← causes →
User B replies: "Me! Let's get pepperoni."

Causal consistency guarantees: no one will ever see B's reply
without first having seen A's question.

But two unrelated posts from User C and User D can appear
in any order on different nodes — that's fine.
```

**What it costs:** The system must track causal dependencies (typically via vector clocks or logical timestamps). More complexity than eventual consistency, but far cheaper than full linearizability.

**When to use it:** Social feeds, comment threads, collaborative editing, messaging systems — anywhere where logical order matters but global total order is overkill.

**Real-world examples:** MongoDB (with causal sessions), Cassandra (with lightweight transactions).

---

### 🟢 Eventual Consistency

**The most common model in large-scale systems.**

If no new writes are made, all replicas will *eventually* converge to the same value. There is no guarantee about *when* that convergence happens — only that it will.

```
Client A writes x = 10 to Node 1
Client B reads x from Node 2 immediately after → might see x = 7 (old value)
Client B reads x from Node 2 five seconds later → sees x = 10 (converged)
```

**What it costs:** Readers may see stale data. Applications must be designed to handle this — either by tolerating it or by implementing read-your-own-writes guarantees at the application layer.

**What it buys:** Massive availability and low latency. The system can accept writes and serve reads even during network partitions. Each node can operate independently.

**When to use it:** Shopping carts, social media likes and follower counts, DNS, caching layers, leaderboards, any data where a few seconds of staleness is acceptable.

**Real-world examples:** DynamoDB, Cassandra, CouchDB, most CDN caches.

> 💡 **Important nuance:** Eventual consistency is a *guarantee about convergence*, not a license for chaos. Well-designed eventually consistent systems use **conflict resolution strategies** (last-write-wins, CRDTs, application-level merging) to handle the cases where two nodes diverge.

---

### ⚪ Weak Consistency

**No guarantees. The system does its best.**

After a write, there is no guarantee that subsequent reads will see it — not immediately, not eventually in any defined way. The system makes no promises.

**When to use it:** Real-time systems where speed is everything and occasional data loss is acceptable. Video calls (a dropped frame is better than a frozen call), live game state, real-time telemetry.

---

### The Spectrum at a Glance

| Model | Guarantee | Latency | Availability | Use Case |
|-------|-----------|---------|--------------|----------|
| **Linearizable** | Reads always see latest write | High | Lower | Finance, leader election |
| **Sequential** | Global consistent order | Medium-High | Medium | Collaborative tools |
| **Causal** | Cause before effect | Medium | High | Messaging, social feeds |
| **Eventual** | Will converge, no timing guarantee | Low | Very High | Likes, caches, DNS |
| **Weak** | No guarantees | Very Low | Very High | Video, gaming, telemetry |

---

## 4. The CAP Theorem Revisited

You first saw CAP in the [Abstraction guide](abstraction.md). Now you have the vocabulary to understand it more precisely.

```
                    Consistency
                   (Linearizable)
                        /\
                       /  \
                      /    \
                     /  ???  \
                    /  (pick  \
                   /    two)   \
                  ──────────────
         Availability          Partition
                               Tolerance
```

In a distributed system, network partitions **will** happen — it's not a matter of if, but when. This means Partition Tolerance isn't really optional. So the real choice in practice is:

**CP systems** — choose Consistency over Availability during a partition. The system may refuse to serve requests rather than return stale data. Examples: Zookeeper, HBase, etcd.

**AP systems** — choose Availability over Consistency during a partition. The system keeps responding, but may return stale or conflicting data. Examples: Cassandra, DynamoDB, CouchDB.

> ⚠️ **Common misconception:** CAP doesn't mean you permanently sacrifice one property. It describes what happens *during a partition*. When the network is healthy, well-designed systems can provide both consistency and availability.

---

## 5. PACELC: The More Complete Picture

CAP only describes behavior during failures. But what about when the system is healthy? That's where **PACELC** comes in — it's an extension of CAP that captures the everyday tradeoff.

```
if Partition:
    choose between Availability (A) and Consistency (C)
else (normal operation):
    choose between Latency (L) and Consistency (C)
```

Even when there's no partition, a strongly consistent system must coordinate across replicas — which adds latency. A latency-optimized system skips that coordination — which relaxes consistency.

| System | Partition behavior | Normal behavior | Classification |
|--------|--------------------|-----------------|----------------|
| DynamoDB | Availability | Latency | PA/EL |
| Cassandra | Availability | Latency | PA/EL |
| BigTable / HBase | Consistency | Consistency | PC/EC |
| Spanner | Consistency | Consistency | PC/EC |
| MongoDB | Availability | Consistency | PA/EC |

> 💡 **Interview move:** If an interviewer asks about your database choice and you mention PACELC behavior (not just CAP), you immediately signal senior-level thinking. Most candidates stop at CAP.

---

## 6. Consistency in Practice: Real Systems

### DynamoDB — Tunable Consistency
DynamoDB lets you choose per-read: **eventually consistent reads** (faster, cheaper) or **strongly consistent reads** (reads always reflect the latest write, costs 2× the read capacity). This is called **tunable consistency** and is a pattern worth knowing.

### Cassandra — Quorum-Based Consistency
Cassandra uses a concept called **quorum**: a read or write is considered successful when a majority of replica nodes respond. If you have 3 replicas:
- Write quorum: 2 of 3 nodes must acknowledge the write
- Read quorum: 2 of 3 nodes must respond to the read
- If both use quorum, you're guaranteed to see at least one node that has the latest write

```
3 replicas, quorum = 2

Write acknowledged by 2 nodes ──────► at least 1 overlaps with
Read answered by 2 nodes       ──────► any quorum read

∴ At least one read node always has the latest value. ✓
```

This is how Cassandra can offer tunable consistency — adjust quorum size, adjust your consistency level.

### Google Spanner — Global Linearizability
Spanner achieves global linearizability across data centers using **TrueTime** — GPS and atomic clocks that bound clock uncertainty. When Spanner commits a transaction, it waits out the uncertainty window to ensure the timestamp is globally unique. This is one of the most impressive engineering feats in distributed systems.

---

## 7. How to Choose a Consistency Model

Use this decision framework when a component in your system needs to store or replicate data:

```
Does incorrect/stale data cause financial loss, security issues,
or data corruption?
        │
        ├── YES → Linearizability or strong consistency
        │         (pay the latency cost, it's worth it)
        │
        └── NO → Does the logical ORDER of events matter to users?
                  (reply must appear after the post, etc.)
                        │
                        ├── YES → Causal consistency
                        │
                        └── NO → Is a few seconds of staleness acceptable?
                                        │
                                        ├── YES → Eventual consistency
                                        │         (maximize availability)
                                        │
                                        └── NO → Sequential consistency
```

---

## 8. Applying This in an Interview

When you introduce any data store or replication strategy, the interviewer may ask: *"What consistency model does that give you?"*

Here's how to answer confidently:

- **Name the model** — "This gives us eventual consistency."
- **Explain what that means** — "Reads may see stale data for a short window after a write."
- **Justify why that's acceptable** — "For a social media like count, a few seconds of staleness is completely fine — users won't notice."
- **Name the tradeoff you're accepting** — "We're trading read freshness for availability and low latency."
- **Know when you'd escalate** — "If this were a payment confirmation, I'd switch to a strongly consistent read or use a transaction."

This four-part answer — model, meaning, justification, tradeoff — is the pattern for any consistency question in an interview.

---

## 9. Worked Example: Social Media Feed vs. Bank Transfer

Two systems. Very different consistency needs. Let's design them side by side.

### Instagram Like Count

```
User A likes a photo
        │
        ▼
Write to DynamoDB (eventually consistent)
        │
        ├── Replica 1: 1,042 likes  ✓ updated immediately
        ├── Replica 2: 1,041 likes  ← stale for ~200ms
        └── Replica 3: 1,041 likes  ← stale for ~200ms

User B reads like count → might see 1,041 instead of 1,042

Is this a problem? No. The user will never notice 200ms of staleness
on a like counter. Eventual consistency is the right call.

✓ High availability, low latency, massive scale
```

---

### Bank Transfer: Alice sends $500 to Bob

```
Alice's balance: $1,000
Bob's balance:   $200

Step 1: Debit Alice  → balance = $500
Step 2: Credit Bob   → balance = $700

These two operations MUST be atomic and linearizable.

If a read happens between Step 1 and Step 2 on a stale replica:
→ Alice's money has left, but Bob hasn't received it yet
→ The $500 is briefly "missing" from the system
→ This is catastrophic

✗ Eventual consistency is WRONG here.
✓ Linearizable transactions with ACID guarantees are required.
```

Same underlying technology (distributed database), completely different consistency requirements — chosen based on the consequence of seeing stale data.

---

## 10. Self-Check

Answer these without looking:

1. Why does replication create a consistency problem?
2. What is linearizability, and what does it cost?
3. What is the difference between causal consistency and eventual consistency?
4. What does PACELC add to CAP that CAP alone doesn't capture?
5. A user posts a comment and immediately refreshes the page — they don't see their own comment. What consistency model is the system using, and how would you fix it?
6. You're designing a ride-sharing app. The driver's GPS location updates every second. What consistency model would you choose for storing location data, and why?
7. What is quorum, and how does it let Cassandra offer tunable consistency?

---

## 11. References

| Resource | Why it's worth it |
|----------|-------------------|
| 📘 [Designing Data-Intensive Applications — Ch. 5 & 9 (Kleppmann)](https://dataintensive.net) | The definitive deep dive on replication, consistency, and the linearizability cost |
| 📝 [Consistency Models — Jepsen (Kyle Kingsbury)](https://jepsen.io/consistency) | The most comprehensive visual map of consistency models on the internet |
| 🔬 [Spanner: Google's Globally Distributed Database](https://research.google/pubs/pub39966/) | The original paper — see how TrueTime achieves global linearizability |
| 📊 [PACELC — Daniel Abadi (2012)](https://dl.acm.org/doi/10.1145/2462300) | The original paper extending CAP with the latency tradeoff |
| 📬 [ByteByteGo — Consistency Models Explained](https://bytebytego.com) | Visual walkthrough of the spectrum with real-world examples |

---

*⬅️ Previous: [Network Abstraction: RPC](networkAbstraction.md) &nbsp;|&nbsp; ➡️ Next: [Failure Models](failureModels.md)*

---

<sub>Part of the <a href="../../README.md">System Design Foundations</a> study guide series — Preliminary System Design Concepts.</sub>