# 🏗️ Preliminary System Design Concepts

> *Grasp the fundamentals of abstractions in distributed systems — focusing on network abstraction, consistency, and failure models that every real system is built on.*

---

## What This Section Covers

Before designing any large-scale system, we need a solid mental model of how distributed systems actually behave — not just when things go right, but especially when they don't.

This section builds that foundation across four tightly connected topics. Each one is a prerequisite for the next: we can't reason about consistency without understanding replication, and we can't reason about failure without understanding consistency.

Read them in order the first time through.

---

## Guides

| # | Topic | Description | Guide |
|---|-------|-------------|-------|
| 1 | **Abstraction** | What abstraction is, why it exists, and the key abstractions every distributed system is built from — caches, queues, load balancers, APIs, and more. | [Read →](abstraction.md) |
| 2 | **Network Abstraction: RPC** | How Remote Procedure Calls hide the complexity of network communication, where the illusion breaks down, and how to design around failure modes. | [Read →](networkAbstraction.md) |
| 3 | **Spectrum of Consistency Models** | Why consistency is a dial, not a switch — from linearizability to eventual consistency, and how to choose the right model for a given problem. | [Read →](consistencyModels.md) |
| 4 | **Failure Models** | The types of failures distributed systems face — from crash-recovery to network partitions to Byzantine faults — and how systems are designed to tolerate them. | [Read →](failureModels.md) |

---

## How These Topics Connect

```
Abstraction
    │
    └──► We build distributed systems out of abstractions (caches, queues, DBs).
         But abstractions over a network behave differently than local ones.
                │
                ▼
        Network Abstraction (RPC)
                │
                └──► Remote calls can fail in ambiguous ways.
                     To handle this, we need to choose what guarantees we make.
                            │
                            ▼
                    Consistency Models
                            │
                            └──► Those guarantees depend on how
                                 our system behaves when things go wrong.
                                        │
                                        ▼
                                 Failure Models
```

---
*⬅️ Back to [README](../../README.md) &nbsp;|&nbsp; ➡️ First Guide: [Abstraction](abstraction.md)* 

*⬅️ Back to [README](../../README.md) &nbsp;|&nbsp; ➡️ Next Section: [Non-Functional System Characteristics](../non-functional-system-characteristics/NonFunctionalSystemCharacteristics.md)*
