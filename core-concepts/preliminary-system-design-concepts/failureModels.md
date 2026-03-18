# 💥 Failure Models in Distributed Systems

> *"The question is never whether your system will fail. It's whether you designed for it before it did."*

**⏱ Reading time:** ~13 minutes &nbsp;|&nbsp; **📍 Series:** Grokking System Design &nbsp;|&nbsp; **🗂 Topic #4:** Preliminary Concepts

---

## Table of Contents

1. [Why Failure Models Matter](#1-why-failure-models-matter)
2. [The Spectrum of Failures](#2-the-spectrum-of-failures)
3. [Node Failures](#3-node-failures)
4. [Network Failures](#4-network-failures)
5. [The Hardest Failure: Byzantine Faults](#5-the-hardest-failure-byzantine-faults)
6. [How Systems Are Designed to Tolerate Failure](#6-how-systems-are-designed-to-tolerate-failure)
7. [Failure Detection](#7-failure-detection)
8. [The Relationship Between Failures and Consistency](#8-the-relationship-between-failures-and-consistency)
9. [Worked Example: What Happens When a Node Dies Mid-Request](#9-worked-example-what-happens-when-a-node-dies-mid-request)
10. [Self-Check](#10-self-check)
11. [References](#11-references)

---

## 1. Why Failure Models Matter

In [RPC](networkAbstraction.md) we saw that remote calls can fail in ambiguous ways. In [Consistency Models](consistencyModels.md) we saw that network partitions force hard choices between availability and correctness. Both of those topics assumed failures happen — but we never stopped to ask: *what kinds of failures are there, and how do they differ?*

That question matters more than it might seem. A system that's designed to handle one kind of failure may be completely unprepared for another. The engineering effort required to tolerate a slow node is very different from the effort required to tolerate a node that's actively sending wrong answers. Confusing the two leads to systems that feel robust but fail in unexpected ways.

Failure models give you a precise vocabulary for the kinds of things that go wrong in distributed systems. Once you can name what you're designing against, you can reason clearly about what your system can and can't handle — and where its limits actually are.

---

## 2. The Spectrum of Failures

Failures exist on a spectrum from simple and easy to handle, to complex and extremely difficult to defend against. As you move along this spectrum, the failures become rarer but more expensive to tolerate.

```
SIMPLE ◄─────────────────────────────────────────────► COMPLEX

  Fail-Stop      Crash-Recovery     Omission      Byzantine
  (node stops,   (node crashes,     (node drops    (node behaves
   everyone       comes back,        some msgs,     arbitrarily,
   knows)         may be stale)      not others)    may lie)

  Easiest to     Common in          Harder to      Extremely
  handle         practice           detect         hard to handle
```

Let's walk through each one.

---

## 3. Node Failures

### 💤 Fail-Stop Failure

The simplest failure model. A node fails by **completely stopping**, and every other node in the system can immediately detect that it has stopped.

```
Node A  ──────────────────────────────────── stopped
Node B  ──────────────────── detects A stopped immediately
Node C  ──────────────────── detects A stopped immediately

No ambiguity. No partial state. Just gone.
```

This is the easiest failure to handle because there's no ambiguity. The node is either up or it isn't, and you know which. Most fault-tolerance theory is built around this model because it's analytically clean.

In practice, pure fail-stop failures are rare. Real systems don't always cleanly announce their failure. A process might hang without crashing. A machine might become so overloaded it stops responding without technically dying. Fail-stop is a useful theoretical baseline, but don't confuse it with what actually happens in production.

---

### 😴 Crash-Recovery Failure

Much more common in practice. A node **crashes and may later recover**, potentially returning with stale or incomplete state.

```
Node A  ───────────── CRASH ════════════════ RECOVERS ──────────
                         ↑                        ↑
                    lost in-memory           may have missed
                    state here               writes that happened
                                             while it was down
```

This is what happens when a server reboots, a process is killed and restarted, or a container crashes and its orchestrator (Kubernetes, etc.) brings it back up. The node comes back, but it may have missed events that happened during its downtime. If it was the primary for some data, a new primary may have been elected in its absence.

Crash-recovery failures are why **write-ahead logs** exist in databases. Before a database applies a write, it records the intent to disk first. If the node crashes mid-write and recovers, it replays the log to restore a consistent state. The log is the system's memory of what was happening before it went down.

Crash-recovery is also why **leader election** matters. When the primary node of a replicated system crashes, the remaining nodes need a way to agree on a new leader without creating split-brain — a situation where two nodes both believe they're the leader and accept conflicting writes.

---

### 📭 Omission Failure

A node **fails to send or receive some messages** without crashing. It's still running, but selectively dropping communication.

```
Node A sends message to Node B ──────────────────► ✓ arrives
Node A sends message to Node B ──────────────────► ✗ lost
Node A sends message to Node B ──────────────────► ✓ arrives

Node B is still running. Node A has no way to know
whether message 2 was dropped or just delayed.
```

This is harder to detect than a crash because the node appears to be alive. Other nodes can't distinguish between "message is taking a long time" and "message was dropped." This ambiguity is exactly what causes the third RPC failure case from the [RPC guide](networkAbstraction.md) — you don't know if your request was received.

Network packet loss is a common cause of omission failures. So is a node that's too overloaded to process messages quickly enough and starts dropping them from its queue. So is a firewall rule that's misconfigured and silently dropping traffic on specific ports.

---

### 🧠 Crash-Stop with Partial Failure (Timing Failure)

A node **operates correctly but too slowly** — responses arrive outside acceptable time windows, effectively making them useless.

```
Node A requests data from Node B
Node B processes... processes... processes...

Timeout! Node A gives up and treats B as failed.

Later: Node B's response arrives. Too late.
       Node A has already taken a different action based on B's silence.
       The system must handle this "ghost response" gracefully.
```

This is insidious because the node is technically working — it processed the request and sent a response. But from the perspective of the rest of the system, a response that arrives too late is indistinguishable from no response. Timeouts turn timing failures into apparent omission failures.

This is also why **timeout values matter enormously**. Set them too short and you declare healthy-but-slow nodes dead. Set them too long and your system hangs waiting for truly dead nodes. Calibrating timeouts well requires understanding your system's actual latency distribution under load.

---

## 4. Network Failures

Node failures are when a machine has a problem. Network failures are when the *connections between machines* have a problem. These are distinct and both common.

### 🔌 Network Partition

A partition happens when the network splits into groups that can't communicate with each other, even though the nodes themselves are healthy.

```
Before partition:
  Node A ←──────────────────────► Node B
  Node A ←──────────────────────► Node C
  Node B ←──────────────────────► Node C

After partition:
  [Node A] ✗──────────────────── [Node B]
                                  [Node C]

  Node A is healthy. Nodes B and C are healthy.
  But A can't reach B or C. B and C can reach each other.
```

This is the "P" in CAP theorem. From Node A's perspective, Nodes B and C have disappeared. From B and C's perspective, Node A has disappeared. Neither side knows whether the other crashed or whether the network is just broken.

During this partition, the system must choose: does Node A keep accepting writes (availability) knowing it may diverge from B and C? Or does it refuse writes (consistency) until it can verify the other nodes again?

Partitions are usually temporary — a network link is restored, a router reboots. But "temporary" can mean seconds, minutes, or in rare cases hours. The system must handle the partition period gracefully and also handle **reconciliation** when the partition heals: two sides that accepted writes independently now need to merge their state.

### 📶 Packet Loss, Reordering, and Duplication

Below the level of full partitions, networks fail in more subtle ways:

**Packet loss** — individual packets are dropped and never arrive. TCP handles this with retransmission, but retransmission takes time and adds latency. UDP doesn't handle it at all, which is why real-time applications that use UDP need to be designed to tolerate missing data.

**Packet reordering** — packets sent in order A, B, C may arrive as A, C, B. TCP reorders them correctly before delivering to the application. UDP does not.

**Packet duplication** — a packet is delivered more than once. This can cause duplicate operations at the application layer if the application doesn't handle it — which is why idempotency matters.

These are especially relevant when you're designing systems that care about message ordering or exactly-once delivery.

---

## 5. The Hardest Failure: Byzantine Faults

All the failures above share one property: the failing node either stops responding or responds slowly or drops messages. It doesn't *lie*. A Byzantine fault is when a node behaves **arbitrarily and maliciously** — sending incorrect data, sending different data to different peers, or actively working to subvert the system.

The name comes from the **Byzantine Generals Problem**: imagine several army generals surrounding a city, communicating only by messenger. They need to agree on a coordinated attack. The problem is that some generals (and some messengers) may be traitors who send conflicting orders to different peers. How do loyal generals reach consensus despite traitors?

```
General A (loyal) ──── "Attack at dawn" ────► General B (loyal)
General A (loyal) ──── "Attack at dawn" ────► General C (traitor)
General C (traitor) ── "Retreat!" ──────────► General B (loyal)
General C (traitor) ── "Attack at dawn" ────► General D (loyal)

General B receives: "Attack" from A, "Retreat" from C.
What does B do?
```

In distributed systems, Byzantine faults can arise from:
- **Hardware corruption** — a disk or network card returning corrupted data
- **Software bugs** — a node running buggy code that produces wrong results
- **Security attacks** — a compromised node intentionally sending false data
- **Cosmic rays** — yes, literally — high-energy particles can flip bits in memory

**Byzantine Fault Tolerance (BFT)** — designing a system that functions correctly even with Byzantine nodes — requires that fewer than 1/3 of all nodes are faulty, and it requires dramatically more communication between nodes to verify correctness. It is expensive and complex.

Most distributed systems in practice **do not implement Byzantine fault tolerance** because they operate within trusted environments (a single company's data centers, internal networks) where the Byzantine threat model doesn't apply. Crash failures and partitions are expected; active deception is not.

The notable exception is **blockchain systems**. A public blockchain has no trusted environment — any participant could be malicious. This is why blockchains use Byzantine-fault-tolerant consensus mechanisms (Proof of Work, Proof of Stake, PBFT) at significant computational cost.

---

## 6. How Systems Are Designed to Tolerate Failure

Understanding failure models is most useful when it informs how you design against them. Here are the core strategies:

### Replication
Store multiple copies of data on multiple nodes. If one node fails, others can serve reads and accept writes. This is the foundation of fault tolerance — redundancy. The tradeoff is the consistency challenges we covered in [Consistency Models](consistencyModels.md).

```
Without replication:          With replication:
  [Node A fails]                [Node A fails]
  Data is gone. ✗               [Node B] still has it. ✓
                                [Node C] still has it. ✓
```

### Consensus Protocols (Raft, Paxos)
When multiple nodes hold replicas, they need to agree on things — which write happened first, who the current leader is, whether a transaction committed. Consensus protocols are the algorithms that let distributed nodes reach agreement despite failures.

**Raft** is the more modern and readable of the two. It works by electing a leader node responsible for coordinating all writes. Writes go to the leader, which replicates to followers. If the leader fails, followers hold an election and choose a new one. A decision requires a majority (quorum) of nodes to agree — this is why most Raft clusters run with an odd number of nodes (3 or 5), so there's always a clear majority.

```
Normal operation:
  Leader ──── replicate write ────► Follower 1  ✓
  Leader ──── replicate write ────► Follower 2  ✓
  Write acknowledged to client after majority confirms.

Leader fails:
  Follower 1 ──── election ────────► Follower 1 becomes new leader
  Follower 2 ──── votes for ───────► Follower 1

  System continues. No data lost (already replicated).
```

### Redundancy and No Single Points of Failure
A **single point of failure (SPOF)** is any component whose failure takes down the whole system. Good distributed system design systematically eliminates them: redundant load balancers, redundant database replicas, redundant network paths, redundant power supplies in data centers.

The question to ask of any component in your design: *"what happens to the system if this specific thing fails?"* If the answer is "everything breaks," that component needs redundancy.

### Graceful Degradation
When parts of a system fail, ideally the rest of the system keeps working at reduced capacity rather than failing completely. This is called graceful degradation.

Netflix popularized this concept with **Chaos Engineering** — deliberately injecting failures into their production system to verify that degradation is actually graceful. If the recommendation service fails, users should still be able to play videos; they just won't get personalized recommendations. The primary function survives; the enhancement degrades.

```
Full system:  Search ✓  Recommendations ✓  Playback ✓  Reviews ✓

Recommendation service fails:
  Search ✓  Recommendations ✗  Playback ✓  Reviews ✓
             (show generic    (core function
              content instead) still works)
```

---

## 7. Failure Detection

Before you can respond to a failure, you have to know it happened. This is harder than it sounds in a distributed system.

### Heartbeats
The most common mechanism. Nodes periodically send a small "I'm alive" signal to other nodes or to a monitoring service. If a heartbeat stops arriving within the expected window, the node is presumed dead.

```
Node B sends heartbeat every 1 second:
  t=0: ❤️
  t=1: ❤️
  t=2: ❤️
  t=3: (nothing)
  t=4: (nothing)
  t=5: Node A marks B as failed after 3 missed heartbeats
```

The tension in heartbeat design is between **detection speed** and **false positives**. A short heartbeat interval with a short timeout detects failures quickly, but a slow network or a briefly overloaded node might miss a heartbeat without actually being dead. A longer timeout reduces false positives but means you take longer to respond to real failures.

### Gossip Protocol
In large clusters, having every node send heartbeats to every other node doesn't scale — that's O(n²) messages. Instead, nodes use a **gossip protocol**: each node periodically picks a few random neighbors to share its state with. Those neighbors share with their neighbors. Information about a node's health "gossips" through the cluster exponentially fast without any central coordinator.

```
Node A tells B and D: "I'm alive, and I know C is alive"
Node B tells E and F: "I'm alive, A is alive, C is alive"
Node E tells A and G: "I'm alive, B is alive, A is alive, C is alive"

Within a few rounds, every node knows every other node's status.
No central coordinator. Scales to thousands of nodes.
```

Cassandra uses gossip protocol for cluster membership and failure detection.

### The Fundamental Limitation: You Can't Know for Sure
This is the deep truth about failure detection in distributed systems: **you can never know with certainty whether a node has failed or is just very slow**. From the outside, a crashed node and a node that's taking 10 seconds to respond look identical until the timeout triggers.

This uncertainty is not a solvable engineering problem — it's a fundamental property of asynchronous systems. Your failure detection gives you a *suspicion*, not a certainty. Well-designed systems are built around this suspicion: they act on it (elect a new leader, stop sending traffic) while being designed to handle the case where the "dead" node comes back and says it was alive all along.

---

## 8. The Relationship Between Failures and Consistency

The failure models above connect directly to the consistency choices from the [previous guide](consistencyModels.md). Failures are what *force* consistency tradeoffs — without failures, you could have everything.

| Failure Type | Consistency Impact |
|--------------|-------------------|
| Fail-stop | Clean failure — easy to elect new leader, no data ambiguity |
| Crash-recovery | Recovering node may be stale — needs to catch up before serving reads |
| Omission | Unclear if write was received — idempotency and retries become essential |
| Network partition | Forces the CAP choice — consistency or availability, pick one |
| Byzantine | Breaks assumptions of most consistency protocols — requires BFT |

This is why fault tolerance isn't an add-on feature. It shapes your consistency model, your replication strategy, your leader election mechanism, and your API design (idempotency keys, retry behavior). These decisions all flow from the same source: what failures are you designing for, and what guarantees do you need to maintain when they happen?

---

## 9. Worked Example: What Happens When a Node Dies Mid-Request

Let's trace exactly what happens in a real scenario. You have a 3-node Raft cluster running your database. Node 1 is the leader. A client sends a write request.

```
Client ──── write: "set balance = $500" ────► Node 1 (Leader)
```

**Scenario A: Clean success**
```
Node 1 receives write, appends to its log
Node 1 ── replicate ──► Node 2: "set balance = $500"  ✓
Node 1 ── replicate ──► Node 3: "set balance = $500"  ✓
Majority (2 of 2 followers) confirmed.
Node 1 commits write, responds to client: ✓ success
```

**Scenario B: Node 1 crashes after replicating to Node 2 but before Node 3**
```
Node 1 ── replicate ──► Node 2: ✓ received
Node 1 ── replicate ──► Node 3: CRASH mid-send

Node 2 and Node 3 hold election.
Node 2 wins (it has the most recent log entry).
Node 2 becomes new leader.
Node 2 ── replicate ──► Node 3: "set balance = $500"  ✓
Write is now on a majority of nodes. Committed.

Client's original connection to Node 1 timed out.
Client retries to new leader (Node 2).
Because the write is idempotent (set, not increment),
the retry is safe. Balance is correctly $500.
```

**Scenario C: Node 1 crashes before replicating to anyone**
```
Node 1 receives write, appends to log, then CRASHES.
No replication happened.

Node 2 and Node 3 hold election.
Node 2 or 3 becomes new leader — neither has the write.
Write is lost from the system's perspective.

Client's connection timed out.
Client retries. Write is applied to new leader.
Balance is correctly $500.

(Node 1 eventually recovers, discovers it's no longer leader,
discards its uncommitted log entry, syncs from new leader.)
```

**Scenario D: Network partition — Node 1 is isolated**
```
Network splits: [Node 1] ✗──── [Node 2, Node 3]

Node 1 still thinks it's leader. A client on Node 1's side
sends a write. Node 1 tries to replicate — can't reach anyone.
Can't get majority confirmation. Write is not committed.
Node 1 correctly refuses to commit without majority.

Meanwhile, Node 2 and Node 3 elect a new leader (Node 2).
Clients that can reach Node 2 can write successfully.

Partition heals. Node 1 discovers Node 2 has a higher term.
Node 1 steps down, syncs from Node 2, resumes as follower.
System is consistent — only one set of committed writes survived.
```

In every scenario, the design holds. Failures are expected, and the protocol is built around them rather than assuming they won't happen.

---

## 10. Self-Check

1. What is the difference between a fail-stop failure and a crash-recovery failure? Why does the distinction matter for system design?
2. What is an omission failure, and why is it harder to handle than a crash?
3. What is a Byzantine fault? Why do most internal distributed systems not implement Byzantine fault tolerance, and what is the exception?
4. Why can't a distributed system ever know with certainty that a node has failed?
5. What is a network partition, and how is it different from a node failure?
6. Your database cluster has 5 nodes. How many can fail simultaneously while the system continues to function correctly under Raft? Why that number?
7. A node in your cluster recovers after being down for 10 minutes. What problem might it have, and what does it need to do before it can safely serve reads again?

---

## 11. References

| Resource | Why it's worth it |
|----------|-------------------|
| 📘 [Designing Data-Intensive Applications — Ch. 8 (Kleppmann)](https://dataintensive.net) | The best treatment of distributed system failures in any book — read Chapter 8 in full |
| 📝 [The Byzantine Generals Problem — Lamport, Shostak, Pease (1982)](https://lamport.azurewebsites.net/pubs/byz.pdf) | The original paper — readable and historically important |
| 🔧 [Raft Consensus Algorithm](https://raft.github.io) | Interactive visualization of leader election and log replication — worth 20 minutes |
| 🎬 [Netflix Chaos Engineering](https://netflixtechblog.com/tagged/chaos-engineering) | How Netflix deliberately breaks their system to prove it's resilient |
| 📬 [ByteByteGo — Fault Tolerance Patterns](https://bytebytego.com) | Visual breakdown of redundancy, failover, and recovery patterns |

---

*⬅️ Previous: [Spectrum of Consistency Models](consistencyModels.md) &nbsp;|&nbsp; ➡️ Next: Non-Functional System Characteristics*

---

<sub>Part of the <a href="../../README.md">System Design Foundations</a> study guide series — Preliminary System Design Concepts.</sub>