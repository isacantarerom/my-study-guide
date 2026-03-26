# 🛡️ Fault Tolerance

> *"The goal isn't to build systems that never fail. It's to build systems whose failures are survivable."*

**⏱ Reading time:** ~12 minutes &nbsp;|&nbsp; **📍 Series:** Grokking System Design &nbsp;|&nbsp; **🗂 Topic #5:** Non-Functional Characteristics

---

## Table of Contents

1. [What Fault Tolerance Actually Means](#1-what-fault-tolerance-actually-means)
2. [Fault vs Error vs Failure — The Distinction That Matters](#2-fault-vs-error-vs-failure--the-distinction-that-matters)
3. [The Core Strategies for Fault Tolerance](#3-the-core-strategies-for-fault-tolerance)
4. [Redundancy in Depth](#4-redundancy-in-depth)
5. [Graceful Degradation](#5-graceful-degradation)
6. [Bulkheads: Containing the Blast Radius](#6-bulkheads-containing-the-blast-radius)
7. [Timeouts, Retries, and Circuit Breakers — Revisited](#7-timeouts-retries-and-circuit-breakers--revisited)
8. [Chaos Engineering: Proving Your Tolerances](#8-chaos-engineering-proving-your-tolerances)
9. [Fault Tolerance vs High Availability — The Relationship](#9-fault-tolerance-vs-high-availability--the-relationship)
10. [Worked Example: A Fault-Tolerant Checkout Flow](#10-worked-example-a-fault-tolerant-checkout-flow)
11. [Self-Check](#11-self-check)
12. [References](#12-references)

---

## 1. What Fault Tolerance Actually Means

**Fault tolerance is the ability of a system to continue operating correctly — possibly at reduced capacity — when some of its components fail.**

The critical phrase is *continue operating*. Fault tolerance isn't about preventing failures — failures are inevitable, as we covered in [Failure Models](../preliminary-system-design-concepts/failureModels.md). It's about designing systems such that when a component fails, the system as a whole absorbs the failure and keeps serving users rather than collapsing entirely.

This connects directly to [Availability](availability.md): a fault-tolerant system is one that maintains availability in the presence of faults. But fault tolerance is about the *mechanism* — the specific design choices that make availability possible when things go wrong. You can't have high availability without fault tolerance as the underlying property.

Think of it like the engineering of a commercial aircraft. Planes have redundant hydraulic systems, redundant electrical systems, redundant flight computers. Not because the primary systems are unreliable — they're extremely reliable — but because the consequence of a single failure being catastrophic is unacceptable. The redundancy isn't pessimism; it's engineering honesty about what happens in a complex system operated at scale over time.

---

## 2. Fault vs Error vs Failure — The Distinction That Matters

These three words are often used interchangeably, but they describe different things in the failure chain — and distinguishing them changes how you think about where to intervene.

```
    Fault ──────────► Error ──────────► Failure
  (root cause)     (internal state    (visible to
                    is wrong)          the user)
```

**A fault** is the underlying cause — a disk developing bad sectors, a software bug in a rarely-executed code path, a network cable with intermittent connectivity, a cosmic ray flipping a bit in RAM.

**An error** is what happens inside the system as a result of the fault — a read returning corrupted data, an exception being thrown, a message being dropped.

**A failure** is when the error propagates to the point that the system can no longer provide its required service — users see errors, requests time out, data is lost.

The reason this chain matters: **fault tolerance is the art of breaking the chain between fault and failure.** A fault happens. An error is produced internally. But the system detects the error, compensates, and the failure never reaches the user.

```
Disk develops bad sectors (fault)
    │
    ▼
Read returns checksum mismatch (error detected)
    │
    ▼
System fetches data from replica instead (compensation)
    │
    ▼
User receives correct data (failure prevented) ✓
```

Every fault tolerance mechanism in this guide is essentially a way to detect errors before they become failures and respond to them in a way that keeps the system functioning.

---

## 3. The Core Strategies for Fault Tolerance

Fault tolerance is achieved through a combination of strategies that work at different layers. No single strategy is sufficient — real fault tolerance is a stack.

### Prevention
Reducing the probability of faults occurring in the first place. High-quality hardware, thorough testing, input validation, careful code review. Prevention doesn't eliminate faults but reduces their frequency.

### Detection
Recognizing that a fault has occurred or that an error state exists. Health checks, heartbeats, checksums, monitoring, alerting. You can't respond to a fault you don't know about.

### Isolation
Containing the impact of a fault so it doesn't propagate to the rest of the system. If one component fails, the failure stops there rather than cascading. This is the bulkhead pattern, covered in Section 6.

### Recovery
Restoring correct operation after a fault. Automatic failover, replica promotion, retry logic, rollback to a known-good state. Recovery is about MTTR — minimizing how long the system is degraded.

### Masking
Hiding the fault from users entirely — responding correctly despite the fault. This is the highest form of fault tolerance: the system absorbed the fault, compensated internally, and the user never knew anything happened. Read-from-replica when primary is unavailable is masking. Retry with idempotency key so the user sees success despite a transient failure is masking.

---

## 4. Redundancy in Depth

Redundancy is the foundation of fault tolerance — having more than one of every critical component so that the failure of one doesn't take the whole system down. We touched on this in [Availability](availability.md), but it's worth going deeper here because redundancy has layers.

### Component Redundancy
Multiple instances of application servers, multiple database replicas, multiple load balancers. If one fails, others handle the load.

### Geographic Redundancy
Multiple availability zones within a region, multiple regions globally. Protects against data center-level failures — power outages, natural disasters, network cuts.

```
                    Global Load Balancer
                          │
          ┌───────────────┴───────────────┐
          │                               │
    Region: US-West                 Region: US-East
    ┌─────────────────┐             ┌─────────────────┐
    │  AZ-1a  │  AZ-1b│             │  AZ-2a  │  AZ-2b│
    └─────────────────┘             └─────────────────┘

If AZ-1a fails → traffic shifts to AZ-1b. Zero user impact.
If all of US-West fails → traffic shifts to US-East. Brief impact.
```

### Data Redundancy
Multiple copies of data across nodes and regions. Replication ensures that data survives node failures. Backups ensure that data survives corruption or catastrophic loss.

An important distinction: **replication** is live redundancy — replicas are continuously updated and can serve traffic immediately. **Backups** are point-in-time snapshots that require restoration time. Both are necessary. Replication handles the common case (node failure). Backups handle the catastrophic case (data corruption that propagates to all replicas before detection).

### The N+1 Principle
A common design pattern: provision N+1 instances where N is the number needed to handle full load. The +1 is spare capacity that absorbs traffic when any single instance fails, without degrading performance for users.

With N instances and no spare: when one fails, the remaining N-1 must handle full load — they're now operating at 100%/(N-1)/N = over capacity, likely degrading.

With N+1: when one fails, the remaining N handle full load comfortably.

---

## 5. Graceful Degradation

**Graceful degradation is the ability to provide reduced but still useful functionality when parts of the system fail, rather than failing completely.**

This is one of the most user-friendly expressions of fault tolerance. Instead of "something went wrong, please try again," the user gets "here's what we can give you right now."

The key insight is that not all features are equally critical. A system has a **core value proposition** — the thing it absolutely must do — and a set of **enhancements** that improve the experience. When failures occur, protect the core. Let enhancements degrade.

```
Netflix example:

Core:         Play the video the user requested
Enhancements: Personalized recommendations, "Continue watching",
              ratings, reviews, social features

Recommendation service fails:
→ Show generic popular content instead
→ User can still watch movies — core preserved
→ Discovery experience is worse — enhancement degraded

Search service fails:
→ User can browse categories but not search
→ Core partially impaired but not destroyed
→ Most users can still find something to watch
```

Designing for graceful degradation requires explicitly deciding which features are core and which are enhancements before a failure happens — because during an incident is not the time to be making that decision.

### Fallback Strategies

**Cached responses** — when the live data source is unavailable, return the last cached version. Slightly stale data is better than no data for most use cases.

**Default responses** — when a personalization service is down, return a sensible default (popular items, generic recommendations) rather than an error.

**Feature flags** — when a new feature is causing problems, disable it with a flag without a full deployment. The system returns to its previous stable state instantly.

**Read-only mode** — when the write path is impaired, continue serving reads. Users can view but not modify data. For many use cases, read-only is vastly better than complete unavailability.

---

## 6. Bulkheads: Containing the Blast Radius

The bulkhead pattern comes from ship design. A ship's hull is divided into watertight compartments. If one compartment is breached and fills with water, the bulkheads between compartments prevent the water from spreading. The ship may take on damage, but it doesn't sink.

In distributed systems, a bulkhead is any isolation mechanism that prevents a failure in one part of the system from consuming resources in another.

### Thread Pool Isolation

A common implementation: instead of a single shared thread pool for all outbound calls, allocate separate thread pools per downstream dependency.

```
Without bulkheads:              With bulkheads:

Shared thread pool              ┌─── Service A pool (10 threads) ───┐
[thread][thread][thread]        ├─── Service B pool (10 threads) ───┤
[thread][thread][thread]        └─── Service C pool (10 threads) ───┘

Service A becomes slow.         Service A becomes slow.
Threads fill up waiting for A.  Service A's pool fills up.
No threads left for B or C.     Service B and C pools unaffected.
Everything is down.             B and C continue normally. ✓
```

### Rate Limiting Per Consumer

If your service is called by many other services, a bulkhead prevents one misbehaving consumer from consuming all your capacity and degrading service for well-behaved consumers. Each consumer gets a rate-limited allocation. One consumer flooding you with requests hits their limit; others are unaffected.

### Database Connection Pool Isolation

Similarly, separate connection pools for different types of operations prevent a slow batch job or a heavy analytics query from starving out the transactional operations that users are waiting on.

---

## 7. Timeouts, Retries, and Circuit Breakers — Revisited

We introduced these patterns in [Network Abstraction: RPC](../preliminary-system-design-concepts/networkAbstraction.md) as responses to the ambiguous failure problem. They're worth revisiting here as fault tolerance mechanisms with a fuller understanding of why each piece matters.

### The Relationship Between the Three

These three patterns aren't independent — they form a layered defense:

```
Request sent to downstream service
          │
          ▼
    Timeout fires if no response within X ms
          │
          ├── Response arrived → continue normally
          │
          └── Timed out → Retry (with exponential backoff + jitter)
                    │
                    ├── Retry succeeded → continue
                    │
                    └── Multiple retries failed →
                              Circuit Breaker opens
                                    │
                                    ├── Calls blocked for cooldown period
                                    │   (downstream gets time to recover)
                                    │
                                    └── Half-open probe after cooldown
                                              │
                                              ├── Probe succeeds → close circuit
                                              └── Probe fails → reopen circuit
```

**Timeouts** prevent indefinite waiting — a hung downstream can't hold your threads forever.

**Retries** handle transient failures — a brief network hiccup, a momentary overload spike. They're only safe with idempotent operations (as we covered in [Reliability](reliability.md)).

**Circuit breakers** prevent cascading failures — when a downstream is consistently failing, stop sending it traffic so it can recover, and so your own service doesn't saturate its thread pool waiting for responses that aren't coming.

Each layer handles failures the others miss. Timeouts don't help if the service is consistently slow but eventually responding. Retries don't help if the service is consistently failing. Circuit breakers don't help if failures are isolated and transient.

---

## 8. Chaos Engineering: Proving Your Tolerances

Here's an uncomfortable question: how do you know your fault tolerance mechanisms actually work?

Most fault tolerance mechanisms — failover, replica promotion, circuit breakers — are triggered by conditions that rarely occur in normal operation. If you only test them in a controlled environment, you don't know how they'll behave when a real failure hits your production system at 2pm on a Tuesday with full traffic.

**Chaos Engineering** is the practice of deliberately introducing failures into production systems to verify that they handle them correctly.

Netflix pioneered this with **Chaos Monkey** — a tool that randomly terminates virtual machine instances in production during business hours. The reasoning: if your system can't survive a single instance going down, you'll find out during business hours when you have people available to fix it, not at 3am when it happens for real.

```
The Chaos Engineering progression:

1. Define the "steady state" — what does normal look like?
   (latency p99, error rate, request throughput)

2. Hypothesize: "If we kill one app server, steady state will be maintained"

3. Introduce the failure: terminate a random instance

4. Observe: did steady state hold?
   → Yes: confidence that this failure is handled correctly
   → No: you found a real gap in your fault tolerance before users did
```

This isn't recklessness — it's intellectual honesty. A fault tolerance design that's never been tested under real failure conditions is a hypothesis, not a guarantee. Chaos Engineering turns the hypothesis into verified knowledge.

The principle scales: you can run chaos experiments on specific components (kill this database replica), on network conditions (introduce 200ms of latency on calls between these services), or on entire availability zones (simulate a data center failure).

---

## 9. Fault Tolerance vs High Availability — The Relationship

These two are closely related but distinct:

**Fault tolerance** is the *property* — the system can sustain faults without losing its ability to function.

**High availability** is the *outcome* — the system is operational for a high percentage of time.

Fault tolerance is what enables high availability. Without fault tolerance mechanisms — redundancy, failover, graceful degradation, circuit breakers — high availability is fragile. One failure takes the whole system down. With fault tolerance, availability is resilient: individual failures are absorbed, and the system as a whole keeps running.

```
Fault Tolerance (mechanisms)          High Availability (outcome)
─────────────────────────────         ─────────────────────────────
Redundancy                     ────►  System stays up when a node fails
Failover                       ────►  Traffic shifts automatically
Graceful degradation           ────►  Partial failure ≠ total outage
Circuit breakers               ────►  Cascades stopped before full failure
Retries with idempotency       ────►  Transient failures invisible to users
```

You can also have high availability without full fault tolerance — by having such reliable hardware and software that failures are extremely rare. But at scale, that approach hits its limits. At Google or Amazon's scale, hardware fails constantly. High availability is maintained not by preventing failures but by tolerating them.

---

## 10. Worked Example: A Fault-Tolerant Checkout Flow

E-commerce checkout is a good case study because it's both high-stakes (money is involved) and touches many services (inventory, payment, notifications, recommendations).

**Services involved:**
- Inventory Service — is the item still in stock?
- Payment Service — charge the card
- Order Service — create the order record
- Notification Service — send confirmation email
- Recommendation Service — "you might also like..."

**What happens when each service fails:**

```
User clicks "Place Order"
        │
        ▼
Inventory Service check
   ├── Available: continue ✓
   └── FAILS: cannot determine stock
       → Return error: "Unable to complete order, please try again"
       (Core: we can't sell what we can't confirm exists)

        │
        ▼
Payment Service charge
   ├── Success: continue ✓
   └── FAILS after charging (response lost):
       → Retry with idempotency key
       → Same key = no double charge
       → If all retries fail: show error, card NOT charged

        │
        ▼
Order Service record
   ├── Success: continue ✓
   └── FAILS: payment was charged but order not recorded
       → This is the dangerous case
       → Compensating transaction: refund the charge
       → Or: retry Order Service (idempotent with same order ID)

        │
        ▼
Notification Service (email confirmation)
   ├── Success: email sent ✓
   └── FAILS:
       → Queue the notification for async retry
       → Return success to user anyway — order placed is the core
       → Email is an enhancement, not the transaction itself ✓

        │
        ▼
Recommendation Service (cross-sell)
   ├── Available: show personalized recs ✓
   └── FAILS:
       → Show generic popular items (cached fallback)
       → User sees recommendations, just not personalized ✓
       → Or show nothing — checkout still completes ✓
```

Notice the deliberate asymmetry:
- Inventory and payment failures → hard stop (can't safely proceed)
- Notification failure → async retry, user sees success
- Recommendation failure → fallback or empty, user sees success

The system's fault tolerance design reflects the actual consequence of each component failing. The core transaction (stock confirmed, money charged, order recorded) requires all three components. Everything else degrades gracefully.

---

## 11. Self-Check

1. What is the difference between a fault, an error, and a failure? Why does the distinction matter for where you intervene?
2. What is graceful degradation? Give an example of a system that degrades gracefully and one that doesn't.
3. What is a bulkhead in distributed systems, and what problem does it solve?
4. Why is chaos engineering more than just "breaking things on purpose"? What does it actually prove?
5. A payment service calls three downstream services: fraud detection, card processing, and audit logging. The fraud detection service starts failing. What should happen to the other two? What fault tolerance mechanism enables this?
6. What is the difference between fault tolerance and high availability? Can you have one without the other?
7. A team sets up retries for all their service calls to improve fault tolerance, but starts seeing duplicate records in their database. What went wrong, and what's the fix?

---

## 12. References

| Resource | Why it's worth it |
|----------|-------------------|
| 📘 [Designing Data-Intensive Applications — Ch. 8 (Kleppmann)](https://dataintensive.net) | The unreliable network and unreliable clocks chapters — essential context for fault tolerance |
| 🎬 [Netflix Chaos Engineering Blog](https://netflixtechblog.com/tagged/chaos-engineering) | The original Chaos Monkey write-up and subsequent evolution of chaos engineering at Netflix |
| 📝 [Principles of Chaos Engineering](https://principlesofchaos.org) | The community-maintained document defining chaos engineering as a discipline |
| 🔧 [AWS Well-Architected — Reliability Pillar](https://docs.aws.amazon.com/wellarchitected/latest/reliability-pillar/welcome.html) | How AWS thinks about building fault-tolerant systems — practical and detailed |
| 📬 [ByteByteGo — Fault Tolerance Patterns](https://bytebytego.com) | Visual breakdowns of bulkheads, circuit breakers, and redundancy patterns |

---

*⬅️ Previous: [Maintainability](maintainability.md) &nbsp;|&nbsp; ➡️ Next: [Throughput & Latency](throughputAndLatency.md)*

---

<sub>Part of the <a href="../../README.md">System Design Foundations</a> study guide series — Non-Functional System Characteristics.</sub>