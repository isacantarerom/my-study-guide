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
8. [Worked Example: Social Media Feed vs. Bank Transfer](#8-worked-example-social-media-feed-vs-bank-transfer)
9. [Self-Check](#9-self-check)
10. [References](#10-references)

---

## 1. Why Consistency Is a Spectrum

In [RPC](networkAbstraction.md), we saw that distributed systems introduce failure modes that don't exist on a single machine. One of the most profound is the **consistency problem**: when data is stored on multiple machines, how do you guarantee that every reader sees the same version of that data?

The naive answer is "always show the latest data." The reality is that guaranteeing this across multiple machines requires coordination — and coordination is expensive. Every time you want to guarantee that a write has reached all replicas before acknowledging it, you're adding latency. Every time you want to guarantee a read reflects the absolute latest write, you're paying a coordination cost.

In many cases, that cost is worth it. In many others, it isn't — and forcing it where it isn't needed makes systems slower and more fragile than they need to be.

Consistency models exist to give you **precise language for describing what guarantees you're making to your users** and what you're trading away to make those guarantees. Understanding them is what lets you make deliberate choices instead of accidental ones.

---

## 2. The Core Problem: Replication

Consistency problems arise because of **replication** — storing copies of data across multiple nodes.

Replication is not optional in systems that need to survive failures. If you have one copy of your data and the machine it lives on fails, your data is gone. So you replicate: multiple nodes hold copies, and if one goes down, others can serve reads and accept writes.

But replication takes time. Even on fast networks, there is a propagation delay between when a write lands on the primary node and when all replicas reflect it. What happens if someone reads from a replica during that window?

```
         Write: "balance = $500"
                    │
                    ▼
         ┌─────────────────┐
         │   Primary Node  │  ← Write lands here first
         │  balance = $500 │
         └────────┬────────┘
                  │  Replicating...
         ┌────────┴────────┐
         │                 │
         ▼                 ▼
  ┌─────────────┐   ┌─────────────┐
  │  Replica A  │   │  Replica B  │
  │balance = ?  │   │balance = ?  │
  └─────────────┘   └─────────────┘

  A reader hitting Replica A or B during replication
  sees stale data. That's the consistency problem.
```

Every consistency model is essentially a set of rules for what happens in that replication window — who can read what, under what guarantees, with what coordination required.

---

## 3. The Consistency Models — From Strongest to Weakest

Think of this as a dial. Turning toward **strong** gives you accuracy at the cost of speed and availability. Turning toward **weak** gives you speed and availability at the cost of accuracy. Neither extreme is universally correct — the right setting depends on what your system is doing and what the cost of a wrong answer is.

```
STRONG ◄──────────────────────────────────────────────► WEAK

  Linearizable    Sequential    Causal    Eventual    Weak
  (Strict)        Consistency   Consistency  Consistency  Consistency
```

---

### 🔴 Linearizability (Strict Consistency)

**The strongest guarantee. The most expensive.**

Every read reflects the most recent write, period. Operations appear to execute instantaneously in a single global order. From any client's perspective, the system behaves as if there is exactly one copy of the data.

```
Timeline:
  Client A writes x = 5   ──────────────────────►
  Client B reads x         ────────────────────────► must see x = 5
                                                      no exceptions, no timing windows
```

To achieve this, the system must coordinate across nodes on every write — typically using a consensus protocol like Paxos or Raft. Every participating node must acknowledge a write before it's considered complete. This adds latency to every operation, and it means that during a network partition, the system may refuse to serve requests at all rather than risk returning stale data.

Linearizability is the right choice when the consequence of returning stale or incorrect data is severe: financial transactions, distributed locks, leader election, configuration systems.

**Real-world systems:** Google Spanner, Zookeeper, etcd.

---

### 🟠 Sequential Consistency

**Strong, but slightly relaxed on timing.**

All operations appear to execute in *some* sequential order that is consistent across all clients — but that order doesn't have to match real wall-clock time. All clients agree on the same ordering of events; it just might not reflect exactly when those events happened.

```
Client A:  write x=1, then write x=2
Client B:  will always see x=1 before x=2 (order preserved)
           but might not see the update instantly
```

> 📌 **Analogy:** A bulletin board in a shared office. Everyone reads the same board, and posts stay in the order they were pinned — but there might be a delay before a new post appears for everyone. No one sees posts out of order, but they might see them with a slight delay.

Sequential consistency is useful for collaborative tools and shared state that needs to be coherent across users but doesn't require microsecond precision.

---

### 🟡 Causal Consistency

**Preserves cause-and-effect. Lets unrelated things diverge.**

If operation A causally depends on operation B — a reply to a message, an edit to a document someone just shared — then all nodes must show A before B. Operations with no causal relationship can appear in any order across different nodes.

```
User A posts: "Who wants pizza?"         ← this causes →
User B replies: "Me! Let's get pepperoni."

Causal consistency guarantees: no one will ever see B's reply
without first having seen A's question.

But two completely unrelated posts from User C and User D
can appear in any order on different nodes — that's fine,
because they don't depend on each other.
```

The system tracks causal dependencies — typically using **vector clocks** or **logical timestamps** — to know which operations are causally related. This is more complex than eventual consistency but far cheaper than linearizability, because it only requires coordination for causally related events, not for everything.

Causal consistency is often the right model for social feeds, comment threads, messaging systems, and collaborative editing — anywhere where logical order matters but you don't need a total global order of all events.

**Real-world systems:** MongoDB (with causal sessions), some Cassandra configurations.

---

### 🟢 Eventual Consistency

**The most common model in large-scale systems.**

If no new writes are made, all replicas will *eventually* converge to the same value. There is no guarantee about *when* — only that convergence will happen.

```
Client A writes x = 10 to Node 1
Client B reads x from Node 2 immediately → might see x = 7 (stale)
Client B reads x from Node 2 five seconds later → sees x = 10 (converged)
```

The system doesn't coordinate on writes. Each node accepts writes locally and propagates them to other nodes in the background. This means writes are fast and the system stays available even when nodes can't reach each other.

The cost is that readers may see stale data. Applications using eventually consistent storage need to be designed around this — either by tolerating staleness (a like count that's a few seconds behind is fine) or by implementing compensating strategies at the application layer (such as read-your-own-writes, where a user always reads from the node they just wrote to).

> 💡 **Important nuance:** Eventual consistency is a guarantee about convergence, not permission for arbitrary behavior. Well-designed eventually consistent systems use explicit **conflict resolution strategies** — last-write-wins, CRDTs (Conflict-free Replicated Data Types), or application-level merging — to handle cases where two nodes accept conflicting writes while partitioned.

**When it makes sense:** Shopping carts, social media likes and follower counts, DNS propagation, CDN caches, leaderboards — anywhere that a few seconds of staleness has no meaningful consequence.

**Real-world systems:** DynamoDB (default), Cassandra, CouchDB, most CDN layers.

---

### ⚪ Weak Consistency

**No guarantees.**

After a write, there is no promise that subsequent reads will ever see it in any defined timeframe. The system makes its best effort.

This sounds alarming, but it's the right model for real-time systems where speed is everything and occasional data loss is genuinely acceptable. A dropped frame in a video call is far better than a frozen call. Real-time game state where a slightly stale position is fine. Live sensor telemetry where only the latest reading matters and old ones can be discarded.

---

### The Spectrum at a Glance

| Model | Core Guarantee | Latency | Availability | Natural Use Case |
|-------|----------------|---------|--------------|-----------------|
| **Linearizable** | Every read sees latest write | High | Lower | Finance, locks, leader election |
| **Sequential** | Global consistent ordering | Medium-High | Medium | Collaborative tools |
| **Causal** | Cause always precedes effect | Medium | High | Messaging, social feeds |
| **Eventual** | Will converge, no timing promise | Low | Very High | Likes, caches, DNS |
| **Weak** | Best effort only | Very Low | Very High | Video, gaming, telemetry |

---

## 4. The CAP Theorem Revisited

You first encountered CAP in [Abstraction](abstraction.md). Now with the vocabulary of consistency models, you can understand it more precisely.

```
                    Consistency
                   (Linearizable)
                        /\
                       /  \
                      /    \
                     / pick \
                    /  two   \
                   ────────────
          Availability        Partition
                              Tolerance
```

Network partitions will happen in any real distributed system — it's a matter of when, not if. So partition tolerance isn't a real choice; it's a requirement. The practical decision is:

**CP systems** — during a partition, refuse to serve requests rather than return potentially stale data. The system is correct when it responds, but may be unavailable during failures. (Zookeeper, HBase, etcd)

**AP systems** — during a partition, keep serving requests even if some data might be stale. The system stays available, but consistency is weakened. (Cassandra, DynamoDB, CouchDB)

> ⚠️ **Common misconception:** CAP doesn't mean you permanently sacrifice one property. It describes what your system does *during a partition*. When the network is healthy, a well-designed system can provide both availability and strong consistency. CAP is about the failure case.

---

## 5. PACELC: The More Complete Picture

CAP only tells you about behavior during failures. But what about when everything is working normally? That's the gap PACELC fills.

**PACELC** is an extension proposed by Daniel Abadi in 2012:

```
if Partition:
    choose between Availability (A) and Consistency (C)
else (normal operation):
    choose between Latency (L) and Consistency (C)
```

The insight is that even when there's no partition, a strongly consistent system must still coordinate across replicas on every operation — which adds latency. A system that skips that coordination is faster but relaxes its consistency guarantees. This tradeoff exists all the time, not just during failures.

| System | During partition | During normal operation | Classification |
|--------|-----------------|------------------------|----------------|
| DynamoDB | Availability | Latency | PA/EL |
| Cassandra | Availability | Latency | PA/EL |
| BigTable / HBase | Consistency | Consistency | PC/EC |
| Spanner | Consistency | Consistency | PC/EC |
| MongoDB | Availability | Consistency | PA/EC |

PACELC is the more honest model for reasoning about real systems, because partitions are rare but the latency/consistency tradeoff is present on every single request.

---

## 6. Consistency in Practice: Real Systems

### DynamoDB — Tunable Consistency
DynamoDB gives you a choice per read: **eventually consistent** (fast, cheap, may be stale by milliseconds) or **strongly consistent** (always reflects the latest write, costs twice the read capacity units). This is called **tunable consistency** — the application decides the right level per operation based on what it actually needs.

### Cassandra — Quorum-Based Consistency
Cassandra uses a **quorum** model. If you have 3 replicas, a quorum is 2. A write is successful when 2 of 3 nodes acknowledge it. A read is answered when 2 of 3 nodes respond.

Why does this matter? Because if you use quorum for both reads and writes, there is guaranteed overlap:

```
3 replicas, quorum = 2

Write acknowledged by 2 nodes ──────►
Read answered by 2 nodes       ──────►
                                        At least 1 node is in both groups.
                                        That node has the latest write.
                                        ∴ Reads always see the latest write. ✓
```

If you relax quorum on either side (write to 1, read from 1), you get better performance but weaker consistency. Cassandra lets you tune this per operation, making it a highly flexible system at the cost of more careful reasoning required from the developer.

### Google Spanner — Global Linearizability
Spanner achieves global linearizability across data centers using **TrueTime** — a system of GPS receivers and atomic clocks that bounds clock uncertainty across all Spanner nodes to within a few milliseconds. When Spanner commits a transaction, it waits out the uncertainty window before making the write visible, guaranteeing that the commit timestamp is globally unique and ordered.

This is one of the most impressive engineering feats in distributed systems: linearizability at global scale, without sacrificing availability, by using physical time as a coordination mechanism.

---

## 7. How to Choose a Consistency Model

The right question isn't "which consistency model is best?" — it's "what is the cost of returning stale or incorrect data in this specific context?"

```
What is the consequence if a read returns stale or incorrect data?
        │
        ├── Financial loss, security issue, data corruption
        │        → Linearizability
        │          (pay the latency cost — it's worth it here)
        │
        ├── Logical incoherence (reply before question, edit before share)
        │        → Causal consistency
        │          (preserve cause-and-effect, let unrelated things diverge)
        │
        ├── Mildly stale data — user notices within seconds
        │        → Sequential or strong eventual consistency
        │
        └── Stale for a few seconds is completely fine
                 → Eventual consistency
                   (maximize availability and throughput)
```

The decision isn't about the technology — it's about the *semantic meaning* of the data and the *consequences of getting it wrong*.

---

## 8. Worked Example: Social Media Feed vs. Bank Transfer

The same distributed database infrastructure. Completely different consistency requirements. Let's look at why.

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

Is this a problem?
No. The user will never notice 200ms of staleness on a like counter.
The count will converge. No one is harmed by seeing 1,041 vs 1,042.

Eventual consistency is correct here.
✓ High availability, low latency, scales to billions of writes/day.
```

### Bank Transfer: Alice sends $500 to Bob

```
Alice's balance: $1,000
Bob's balance:   $200

Step 1: Debit Alice  → balance = $500
Step 2: Credit Bob   → balance = $700

These two operations must be atomic and linearizable.

What happens with eventual consistency?
→ A read between Step 1 and Step 2 shows Alice's money has left
  but Bob hasn't received it yet.
→ $500 is briefly missing from the system.
→ A retry of Step 1 could debit Alice twice.
→ A network partition could leave the system permanently inconsistent.

Eventual consistency is wrong here. Catastrophically so.
✓ Linearizable transactions with ACID guarantees are required.
```

Same infrastructure, different dials. The difference isn't technical sophistication — it's asking what actually happens when data is wrong and designing for that answer.

---

## 9. Self-Check

1. Why does replication create a consistency problem? What specifically is happening during the window it creates?
2. What is linearizability, and what does it cost to achieve?
3. What is the difference between causal consistency and eventual consistency? When would you choose one over the other?
4. What does PACELC add to CAP that CAP alone doesn't capture?
5. A user posts a comment and immediately refreshes — they don't see their own comment. What consistency model is the system using? What would you change to fix it?
6. You're designing a ride-sharing app. The driver's GPS location updates every second and is displayed to nearby riders. What consistency model would you choose, and why?
7. What is quorum in Cassandra, and how does it provide a consistency guarantee?

---

## 10. References

| Resource | Why it's worth it |
|----------|-------------------|
| 📘 [Designing Data-Intensive Applications — Ch. 5 & 9 (Kleppmann)](https://dataintensive.net) | The definitive treatment of replication, consistency models, and linearizability — read these chapters |
| 📝 [Consistency Models — Jepsen (Kyle Kingsbury)](https://jepsen.io/consistency) | The most comprehensive visual map of consistency models on the internet, with formal definitions |
| 🔬 [Spanner: Google's Globally Distributed Database](https://research.google/pubs/pub39966/) | The original paper — how TrueTime achieves global linearizability |
| 📊 [PACELC — Daniel Abadi (2012)](https://dl.acm.org/doi/10.1145/2462300) | The paper that extends CAP with the latency tradeoff |
| 📬 [ByteByteGo — Consistency Models](https://bytebytego.com) | Visual walkthrough of the spectrum with real-world examples |

---

*⬅️ Previous: [Network Abstraction: RPC](networkAbstraction.md) &nbsp;|&nbsp; ➡️ Next: [Failure Models](failureModels.md)*

---

<sub>Part of the <a href="../../README.md">System Design Foundations</a> study guide series — Preliminary System Design Concepts.</sub>