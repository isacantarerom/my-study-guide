# 🔵 Reliability

> *"Availability asks: is the system up? Reliability asks: is the system right?"*

**⏱ Reading time:** ~11 minutes &nbsp;|&nbsp; **📍 Series:** Grokking System Design &nbsp;|&nbsp; **🗂 Topic #2:** Non-Functional Characteristics

---

## Table of Contents

1. [What Reliability Actually Means](#1-what-reliability-actually-means)
2. [Reliability vs Availability — The Critical Distinction](#2-reliability-vs-availability--the-critical-distinction)
3. [What Makes a System Unreliable](#3-what-makes-a-system-unreliable)
4. [How Systems Achieve Reliability](#4-how-systems-achieve-reliability)
5. [Data Reliability: Durability](#5-data-reliability-durability)
6. [The Reliability vs Performance Tradeoff](#6-the-reliability-vs-performance-tradeoff)
7. [Worked Example: A Payment Service](#7-worked-example-a-payment-service)
8. [Self-Check](#8-self-check)
9. [References](#9-references)

---

## 1. What Reliability Actually Means

**Reliability is the ability of a system to perform its intended function correctly and consistently over time.**

Where availability is about whether the system is *reachable*, reliability is about whether it's *correct*. A system can be available — responding to every request — while being completely unreliable, returning wrong results, corrupting data, or processing transactions incorrectly.

This distinction matters because the engineering solutions are different. Availability problems are solved with redundancy and failover. Reliability problems are solved with correctness guarantees, validation, testing, and careful handling of failure cases.

> 🏥 **Analogy:** A hospital where the doctors always show up (available) but occasionally give the wrong diagnosis or wrong medication (unreliable) is arguably more dangerous than one that's closed on weekends. Presence without correctness isn't just unhelpful — it can actively cause harm.

Reliability is especially critical in systems that handle money, health data, or any operation that's difficult or impossible to reverse. A like button that occasionally double-counts is a minor nuisance. A payment system that occasionally double-charges is a serious problem. The higher the consequence of being wrong, the more reliability needs to be a first-class design concern.

---

## 2. Reliability vs Availability — The Critical Distinction

These two are easy to conflate because they often fail together, but they're measuring different things:

```
                    Is the system responding?
                           │
              ┌────────────┴────────────┐
              │ YES                     │ NO
              ▼                         ▼
   Is the response correct?       Availability problem.
              │                   Add redundancy, fix failover.
   ┌──────────┴──────────┐
   │ YES                 │ NO
   ▼                     ▼
  ✅ Reliable         Reliability problem.
  and Available       Fix correctness: testing,
                      validation, error handling.
```

A few concrete examples to make this tangible:

| Scenario | Available? | Reliable? |
|----------|-----------|-----------|
| Server is down | ❌ No | — |
| Server responds but has a bug returning wrong user data | ✅ Yes | ❌ No |
| Payment processed twice due to retry without idempotency | ✅ Yes | ❌ No |
| Database returns stale data during replication lag | ✅ Yes | ⚠️ Depends on consistency requirements |
| System correctly processes all requests under normal load | ✅ Yes | ✅ Yes |

Notice that idempotency — which we covered in [RPC](../preliminary-system-design-concepts/networkAbstraction.md) — shows up here as a reliability concern. A system that processes the same payment twice because of a network retry isn't unavailable; it's unreliable. The user got a response. It was just wrong.

---

## 3. What Makes a System Unreliable

### Hardware Faults
Disks fail and corrupt data. RAM flips bits. Network cards send malformed packets. At large enough scale, hardware faults are constant — they're not edge cases to handle "just in case," they're events to design around.

The difference between a hardware fault causing downtime versus data corruption is important. A disk that fails cleanly takes a server offline — that's an availability problem with known solutions. A disk that fails silently and starts returning corrupted data is a reliability problem, and it's significantly harder to detect.

### Software Bugs
A bug is the most common source of unreliability. An incorrect calculation, an unhandled edge case, an off-by-one error in a financial calculation — these don't take the system down, they just make it wrong. And a system that's wrong while appearing to work correctly is harder to detect and often harder to fix than one that's simply unavailable.

Software bugs that cause data corruption are particularly serious, because the corruption may propagate to backups before anyone notices, eliminating the ability to recover cleanly.

### Race Conditions and Concurrency Bugs
When multiple processes or threads access shared state simultaneously, the order of operations matters. A race condition occurs when two operations interleave in an unexpected way, producing an incorrect result.

```
Two users simultaneously try to book the last available seat:

Thread A: reads available_seats = 1
Thread B: reads available_seats = 1
Thread A: decrements → available_seats = 0, confirms booking
Thread B: decrements → available_seats = 0, confirms booking

Result: seat booked twice. System was available the whole time.
        The reliability failure came from a concurrency bug.
```

This class of bug is notorious for being hard to reproduce — it only appears under specific timing conditions, often only in production under real load. Preventing it requires either **locking** (only one thread can access the resource at a time) or **optimistic concurrency control** (detect conflicts and reject the second write).

### Cascading Data Corruption
A reliability failure in one part of a system can corrupt data that propagates downstream. A bug in a data processing pipeline that incorrectly transforms records doesn't just affect those records — if that corrupted data is then replicated, cached, and consumed by other services, the corruption spreads before anyone notices.

This is why **data validation at boundaries** matters so much. Every time data crosses a service boundary — from a message queue into a service, from a database into an API response — validating that it conforms to expected structure and semantics is a layer of defense against corruption propagating silently.

---

## 4. How Systems Achieve Reliability

### Testing
The most fundamental reliability tool. Unit tests verify individual functions produce correct results. Integration tests verify components work correctly together. End-to-end tests verify the system produces correct results for real user workflows.

Testing is not sufficient on its own — you can't test every possible input and state. But it's necessary. A system without tests has no systematic way to verify it's doing the right thing, and no safety net when changes are made.

### Validation and Assertions
Validate inputs at every boundary. If a service expects a positive integer and receives a negative one, reject it loudly rather than propagating the invalid value downstream where it causes a harder-to-diagnose failure later.

Assertions — checks that verify invariants that should always be true — are particularly valuable in complex systems. If an account balance should never go negative, asserting that after every transaction catches bugs immediately rather than letting them silently corrupt data.

### Idempotency
As we covered in [RPC](../preliminary-system-design-concepts/networkAbstraction.md), idempotency is the property that makes operations safe to retry. It's a reliability mechanism: a system that processes payments idempotently will never charge a customer twice, even if the network drops the response and the client retries. Without idempotency, retries — which are necessary for availability — become a source of reliability failures.

### Graceful Error Handling
A reliable system doesn't just work correctly on the happy path. It handles failures and unexpected inputs without corrupting state or producing incorrect results.

This means: operations should either succeed completely or fail cleanly, leaving the system in a consistent state. The database concept of **transactions** — ACID guarantees — exists precisely for this. If a transfer of funds requires debiting one account and crediting another, those two operations must happen atomically. If the system fails between them, the transaction rolls back. No partial state. No money lost in the middle.

```
Transfer $500 from Alice to Bob:

Without transactions:                With transactions:
  Debit Alice: $500 ✓                  BEGIN TRANSACTION
  [system crashes]                       Debit Alice: $500
  Credit Bob: never happens              Credit Bob: $500
  $500 is gone.                        COMMIT (or ROLLBACK on failure)
                                       Either both happen or neither does.
```

### Monitoring and Alerting
You can't fix what you can't see. Reliable systems have observability built in — metrics, logs, and traces that make it possible to detect when the system is producing wrong results, not just when it's down.

An availability failure is usually obvious — requests start failing, users complain immediately. A reliability failure can be silent — results are slightly wrong, data is slowly corrupting, calculations are off by a rounding error. Catching these requires actively monitoring the *correctness* of outputs, not just whether responses arrive.

---

## 5. Data Reliability: Durability

A specific and important dimension of reliability is **durability** — the guarantee that once data is written, it stays written.

A system that accepts a write and acknowledges it to the user, then loses that write because of a crash before it was persisted to disk, is unreliable. The user believes their data is saved. It isn't.

Durability is achieved through:

**Write-ahead logging (WAL)** — before applying any write, the database first records the intent to a log on disk. If it crashes mid-write, it replays the log on recovery. This is how PostgreSQL, MySQL, and most relational databases achieve durability.

**Replication** — writes are sent to multiple nodes. Even if one node loses its data, others have copies. This is why databases default to confirming a write only after it's been acknowledged by at least one replica.

**Checksums** — data is stored with a checksum. When data is read, the checksum is verified. If they don't match, the data was corrupted and the system knows to fetch it from a replica instead of serving corrupted bytes silently.

```
Write path with durability guarantees:

Client ──► write("balance = $500")
              │
              ▼
         Write to WAL on disk first
              │
              ▼
         Replicate to at least one replica
              │
              ▼
         Acknowledge to client: ✓ written
              │
         [if node crashes here, WAL enables recovery]
         [if disk corrupts, replica has a clean copy]
         [if data flips bits, checksum detects it]
```

Durability is the reason "it's in the database" feels like a reliable statement. It's not magic — it's a stack of mechanisms working together to ensure that commitment is kept.

---

## 6. The Reliability vs Performance Tradeoff

Reliability mechanisms have costs, and those costs are almost always measured in performance.

**Transactions add latency.** Acquiring locks, coordinating commits across replicas, rolling back on failure — all of this takes time. A system that processes writes transactionally will always be slower than one that doesn't.

**Validation adds CPU.** Checking inputs, verifying checksums, asserting invariants — these are additional operations on every request.

**Replication for durability adds write latency.** Waiting for a replica to acknowledge a write before returning to the client adds a network round-trip to every write.

This tradeoff is real and must be made deliberately. A social media feed that shows slightly stale data doesn't need ACID transactions — the cost isn't worth it. A financial ledger that must never lose a transaction or double-count a payment absolutely does.

The pattern that emerges across well-designed systems: **apply strong reliability guarantees where the consequence of being wrong is high, and relax them where the consequence is low.** Don't pay the performance cost of transactions for operations that don't need them, but never skip them for operations that do.

---

## 7. Worked Example: A Payment Service

Let's design the reliability guarantees for a payment processing service, where being wrong has real financial and legal consequences.

**The core reliability requirements:**
- A payment must never be processed twice
- A payment must never be partially applied (money leaves one account but doesn't arrive in the other)
- A failed payment must leave accounts in their original state
- Every payment must be auditable — we need a record of what happened and when

**How each requirement is met:**

```
Client sends: POST /payments
  Idempotency-Key: "order-123-pay-attempt-1"
  Body: { from: alice, to: bob, amount: 50.00 }
              │
              ▼
         Check idempotency key in database
         Already seen? → return stored result, don't process again
         Not seen? → continue
              │
              ▼
         BEGIN TRANSACTION
           Validate: alice.balance >= 50.00
           Debit:    alice.balance -= 50.00
           Credit:   bob.balance  += 50.00
           Record:   audit_log.insert(payment details, timestamp)
           Store:    idempotency_key → result
         COMMIT (or ROLLBACK if any step fails)
              │
              ▼
         Replicate to at least one replica before acknowledging
              │
              ▼
         Return result to client
```

**What each mechanism protects against:**

| Mechanism | Protects against |
|-----------|-----------------|
| Idempotency key | Double-charging on client retry |
| Transaction | Partial application — money leaving one account but not arriving in the other |
| Rollback | Any validation failure leaving accounts in a dirty state |
| Audit log | Inability to reconstruct what happened for disputes |
| Replication before ack | Losing the payment record on node crash |

No single mechanism is sufficient. Reliability at this level is a stack of guarantees, each covering a failure mode the others don't.

---

## 8. Self-Check

1. What is the difference between availability and reliability? Can a system be available but unreliable? Give a concrete example.
2. What is a race condition? Why is it a reliability problem rather than an availability problem?
3. What is durability, and how does a write-ahead log provide it?
4. Why does idempotency appear as both an availability mechanism and a reliability mechanism?
5. A data processing pipeline has a bug that produces slightly incorrect calculations. The pipeline runs successfully and returns results quickly. Is this an availability, reliability, or performance problem?
6. You're designing a reservation system for concert tickets. Two users try to book the last ticket simultaneously. What mechanism prevents both from successfully booking it, and what happens to the losing request?
7. Why can't you apply the same reliability guarantees everywhere in a system? What does it cost, and how do you decide where to apply them?

---

## 9. References

| Resource | Why it's worth it |
|----------|-------------------|
| 📘 [Designing Data-Intensive Applications — Ch. 1 & 7 (Kleppmann)](https://dataintensive.net) | Ch. 1 defines reliability clearly; Ch. 7 goes deep on transactions and why they exist |
| 📊 [Google SRE Book — Chapter on Risk](https://sre.google/sre-book/embracing-risk/) | How Google reasons about the cost of reliability and when to stop chasing perfection |
| 🔧 [ACID Properties Explained](https://en.wikipedia.org/wiki/ACID) | The four properties that define transactional reliability — worth knowing cold |
| 📬 [ByteByteGo — Reliability Patterns](https://bytebytego.com) | Visual breakdowns of idempotency, transactions, and fault-tolerant design |

---

*⬅️ Previous: [Availability](availability.md) &nbsp;|&nbsp; ➡️ Next: [Scalability](scalability.md)*

---

<sub>Part of the <a href="../../README.md">System Design Foundations</a> study guide series — Non-Functional System Characteristics.</sub>