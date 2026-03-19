# 🟢 Availability

> *"A system that's correct but unreachable is, from the user's perspective, broken."*

**⏱ Reading time:** ~12 minutes &nbsp;|&nbsp; **📍 Series:** Grokking System Design &nbsp;|&nbsp; **🗂 Topic #1:** Non-Functional Characteristics

---

## Table of Contents

1. [What Availability Actually Means](#1-what-availability-actually-means)
2. [How Availability Is Measured](#2-how-availability-is-measured)
3. [The Nines of Availability](#3-the-nines-of-availability)
4. [What Causes Unavailability](#4-what-causes-unavailability)
5. [How Systems Achieve High Availability](#5-how-systems-achieve-high-availability)
6. [Availability in Distributed Systems: The Hard Part](#6-availability-in-distributed-systems-the-hard-part)
7. [Availability vs Reliability vs Consistency](#7-availability-vs-reliability-vs-consistency)
8. [Worked Example: Designing for High Availability](#8-worked-example-designing-for-high-availability)
9. [Self-Check](#9-self-check)
10. [References](#10-references)

---

## 1. What Availability Actually Means

**Availability is the proportion of time a system is operational and able to serve requests.**

That definition sounds simple, but it contains an important word: *operational*. A server that's running but returning errors isn't available. A service that's up but too slow to respond within a reasonable window isn't really available either. Availability is about whether the system is actually doing its job, not just whether its processes are running.

For most systems, availability is the non-functional characteristic users feel most directly. A user doesn't notice that your database uses eventual consistency. They absolutely notice when the app won't load.

This is why availability tends to be the first non-functional requirement that gets discussed in system design — and why it's worth understanding deeply, not just as a number on an SLA.

---

## 2. How Availability Is Measured

Availability is typically expressed as a percentage of time the system was operational over a given period:

```
Availability = (Total Time - Downtime) / Total Time × 100%
```

Two related metrics show up constantly when discussing availability in production systems:

**MTBF — Mean Time Between Failures**
The average time a system runs before experiencing a failure. A higher MTBF means failures are less frequent.

**MTTR — Mean Time To Recovery**
The average time it takes to restore service after a failure occurs. A lower MTTR means you recover faster when things go wrong.

Together they give you a more complete picture than uptime percentage alone:

```
Availability = MTBF / (MTBF + MTTR)
```

> 💡 This formula reveals something important: you can improve availability in two ways — make failures happen less often (increase MTBF) *or* recover from them faster when they do (decrease MTTR). In practice, both matter, and the balance between them drives very different engineering decisions.

A system with rare but catastrophic failures that take hours to recover from might have the same availability percentage as a system with frequent small failures that recover in seconds. But these are very different systems to operate, and they feel very different to users.

---

## 3. The Nines of Availability

Availability requirements are commonly expressed as "nines" — the number of nines in the percentage. This shorthand exists because the differences between high availability levels are easier to grasp as concrete downtime than as percentages that all start with "99.something."

| Availability | Annual Downtime | Monthly Downtime | Daily Downtime |
|-------------|----------------|-----------------|----------------|
| 90% (one nine) | ~36.5 days | ~73 hours | ~2.4 hours |
| 99% (two nines) | ~3.65 days | ~7.3 hours | ~14.4 minutes |
| 99.9% (three nines) | ~8.76 hours | ~43.8 minutes | ~1.4 minutes |
| 99.99% (four nines) | ~52.6 minutes | ~4.4 minutes | ~8.6 seconds |
| 99.999% (five nines) | ~5.26 minutes | ~26.3 seconds | ~0.86 seconds |

A few things this table makes viscerally clear:

**The jump from two to three nines is enormous.** Going from 99% to 99.9% sounds like a small improvement. But it's the difference between 7 hours of downtime per month and 44 minutes. For a payment service or a hospital system, that gap is the difference between acceptable and not.

**Five nines is genuinely hard to achieve.** 5.26 minutes of allowed downtime per year means your deployments, your hardware failures, your incident response, your database migrations — all of it combined — must fit inside that budget. Any planned maintenance requires zero-downtime deployment strategies. Any unplanned failure requires automated recovery in seconds.

**Most systems don't need five nines.** A developer tool or an internal dashboard running at 99.9% is completely fine. A financial trading platform or an air traffic control system has a different calculus entirely. Knowing what availability your system actually needs — and why — is more important than blindly chasing nines.

---

## 4. What Causes Unavailability

To design for availability, it helps to know what actually takes systems down. Failures generally fall into a few categories:

### Hardware Failures
Disks fail. Network cards fail. Power supplies fail. In a large enough system, hardware failure isn't an edge case — it's a certainty. At Google's scale, thousands of hard drives fail every single day. Designing for hardware failure means assuming it will happen and building around it, not hoping it won't.

### Software Failures
Bugs, memory leaks, deadlocks, unhandled exceptions. A software failure can take down a perfectly healthy physical machine. Unlike hardware, software failures often affect all instances of a service simultaneously if they share the same code — a bug that crashes one instance of your service will crash all of them.

### Network Failures
As we covered in [Failure Models](../preliminary-system-design-concepts/failureModels.md), networks fail in many ways — packet loss, partitions, latency spikes. A service that's physically running may be unreachable due to a network failure between it and its callers.

### Overload
A system that's technically healthy can become unavailable simply because demand exceeds capacity. Too many requests, too much data, too much concurrent processing — the system slows to the point of being unusable, or crashes under the load.

### Planned Maintenance and Deployments
Every deployment is a potential source of downtime. Database migrations that require locks, service restarts, configuration changes that require reboots — these are controllable sources of unavailability that good engineering practice can minimize or eliminate.

### Human Error
Misconfigured load balancers, accidentally deleted databases, wrong environment targeted by a deployment script. Human error is consistently one of the leading causes of major outages in production systems.

---

## 5. How Systems Achieve High Availability

The core principle behind all high availability design is the same: **eliminate single points of failure through redundancy.**

A single point of failure (SPOF) is any component whose failure takes down the whole system. Every SPOF is a ceiling on your availability — no matter how reliable everything else is, that one component defines your worst case.

### Redundancy

Run multiple instances of everything. Multiple application servers. Multiple database replicas. Multiple load balancers. Multiple network paths. If one fails, others absorb the load.

```
Without redundancy:          With redundancy:
                             
  [Load Balancer]              [Load Balancer A] [Load Balancer B]
        │                              │                 │
  [App Server]                 [App A] [App B] [App C]
        │                              │
  [Database]                   [DB Primary] ──► [DB Replica]

  Any component fails         Any single component can fail.
  → system is down            → system keeps running.
```

Redundancy is the foundation, but it creates its own complexity: now you need to handle failover (switching to the backup when the primary fails), and you need to handle the consistency questions that come with multiple copies of data.

### Failover

When a primary component fails, failover is the process of switching traffic to a backup. This can be:

**Active-passive** — one instance is live, one is on standby. The standby takes over when the primary fails. Simple, but the standby is wasted capacity during normal operation, and there's a brief interruption during the switch.

**Active-active** — multiple instances are all live simultaneously, sharing the load. Any instance can fail and the others absorb its traffic with no switchover needed. More complex to manage (especially for stateful systems), but no wasted capacity and no failover delay.

### Health Checks and Automatic Recovery

Availability isn't just about surviving failures — it's about *detecting and recovering from them quickly*. MTTR matters as much as MTBF.

Load balancers continuously health-check the servers behind them. If a server stops responding, the load balancer stops sending it traffic. When it recovers, the load balancer adds it back. This happens automatically, often within seconds, without human intervention.

Container orchestration systems like Kubernetes take this further: if a container crashes, Kubernetes restarts it automatically. If a node fails, Kubernetes reschedules the containers that were running on it to healthy nodes.

### Geographic Distribution

A data center can go offline — power failure, natural disaster, network cut. A system that runs entirely in one data center has that data center as a single point of failure, no matter how redundant it is internally.

High availability at the infrastructure level means distributing across multiple **availability zones** (isolated sections within a data center) and multiple **regions** (geographically separate data centers). Traffic is routed away from an affected zone or region automatically when failures occur.

### Zero-Downtime Deployments

Deployments are a controllable source of downtime. Strategies to eliminate them:

**Rolling deployment** — update instances one at a time while the others keep serving traffic. At no point is the entire service down.

**Blue-green deployment** — run two identical environments ("blue" and "green"). Deploy to the inactive one, test it, then switch traffic over. Instant rollback is possible by switching back.

**Canary deployment** — route a small percentage of traffic to the new version first. If it behaves correctly, gradually increase the percentage. Problems are caught with limited blast radius before full rollout.

---

## 6. Availability in Distributed Systems: The Hard Part

Single-machine availability is relatively straightforward — redundancy and good error handling get you far. Distributed systems introduce a harder problem: **the availability of the whole system depends on the availability of all its components, and those components interact in complex ways.**

### Cascading Failures

When one service fails, it can trigger failures in the services that depend on it. Service A calls Service B, which calls Service C. If C becomes slow or unavailable, B's requests pile up waiting for C. B runs out of threads or connections. A's requests to B start failing. Users see A fail — even though A itself is fine.

This is a **cascading failure**: a localized failure propagating outward through dependencies until the whole system is down.

The circuit breaker pattern from [Network Abstraction: RPC](../preliminary-system-design-concepts/networkAbstraction.md) exists specifically to stop this cascade. When B detects that C is failing, it opens the circuit and stops sending requests to C rather than piling them up. B can then degrade gracefully (return cached data, return a default response) instead of failing completely.

### The Dependency Chain Problem

In a microservices architecture, any given user request may touch many services. Each service has its own availability. The availability of the end-to-end request is the *product* of all those individual availabilities.

```
Service A: 99.9% available
Service B: 99.9% available
Service C: 99.9% available

End-to-end availability of a request touching all three:
0.999 × 0.999 × 0.999 = 99.7%

Add five more services at 99.9% each:
0.999^8 = 99.2%
```

Each additional dependency in the critical path erodes overall availability. This is why minimizing the number of synchronous dependencies in a request path matters — and why async communication via message queues (which decouple availability) is often preferred for non-critical operations.

---

## 7. Availability vs Reliability vs Consistency

These three are closely related but meaningfully different, and conflating them causes confusion.

**Availability** — is the system reachable and responding? Can users make requests?

**Reliability** — when the system responds, is the response *correct*? Does it do what it's supposed to?

**Consistency** — do all users see the same data at the same time?

A system can be:
- **Available but unreliable** — it responds, but gives wrong answers (a bug that returns corrupted data)
- **Available but inconsistent** — it responds to everyone, but different users see different versions of the same data (eventual consistency)
- **Unavailable but would be consistent** — it refuses to respond during a partition rather than risk returning stale data (a CP system in CAP)

These distinctions matter because the fixes are different. An availability problem is solved with redundancy and failover. A reliability problem is solved with testing, validation, and correctness guarantees. A consistency problem is solved with the consistency models we covered in [Consistency Models](../preliminary-system-design-concepts/consistencyModels.md).

---

## 8. Worked Example: Designing for High Availability

Let's design a payment processing service targeting 99.99% availability (~52 minutes of downtime per year).

**What we're protecting against:**
- Single app server failure
- Database failure
- Load balancer failure
- Data center failure
- Deployment-caused downtime

```
                    Global Load Balancer
                   (routes across regions)
                    /                  \
         Region A (US-West)        Region B (US-East)
               │                         │
         Load Balancer A            Load Balancer B
          (active-active)            (active-active)
         /      │      \            /      │      \
    App 1   App 2   App 3      App 4   App 5   App 6
        \      │      /            \      │      /
         DB Primary (A)    ──────►  DB Replica (B)
         + Local Replica            + Local Replica
```

**How each failure is handled:**

| Failure | Response | Downtime |
|---------|----------|----------|
| Single app server | Load balancer routes around it immediately | ~0 seconds |
| Load balancer | Global LB routes to other region's LB | ~seconds |
| DB primary fails | Replica is promoted, app reconnects | ~30-60 seconds |
| Entire region fails | Global LB routes all traffic to other region | ~seconds |
| Deployment | Rolling deploy — servers updated one at a time | 0 seconds |

**The honest tradeoffs:**
- Active-active across regions means we need to handle consistency across two database instances — we've accepted eventual consistency for non-critical reads
- The DB failover (~30-60 seconds) is the weak point in this design — it consumes a meaningful slice of our annual downtime budget per event
- More regions = more availability but also more cost and more complexity to operate

This is what real availability design looks like: not "add redundancy everywhere" but "identify the failure modes, understand the recovery time for each, and verify the math adds up to your availability target."

---

## 9. Self-Check

1. What is the difference between MTBF and MTTR? Which one does redundancy improve, and which does automation improve?
2. A system has 99.9% availability. How much downtime does that allow per month?
3. What is a single point of failure? Give an example of one in a simple three-tier web application.
4. What is a cascading failure, and what pattern from a previous guide prevents it?
5. You have a microservices system where a user request touches 10 services, each at 99.9% availability. What is the end-to-end availability of that request? What does this tell you about system design?
6. What is the difference between active-passive and active-active failover? When would you choose one over the other?
7. A system is available 100% of the time but occasionally returns wrong data. Is this an availability problem, a reliability problem, or a consistency problem?

---

## 10. References

| Resource | Why it's worth it |
|----------|-------------------|
| 📘 [Designing Data-Intensive Applications — Ch. 1 (Kleppmann)](https://dataintensive.net) | The clearest treatment of reliability, availability, and maintainability as a trio |
| 📊 [Google SRE Book — Availability Chapter](https://sre.google/sre-book/availability-table/) | How Google thinks about and measures availability in production — free online |
| 🔧 [AWS Well-Architected Framework — Reliability Pillar](https://docs.aws.amazon.com/wellarchitected/latest/reliability-pillar/welcome.html) | Real-world patterns for building highly available systems on cloud infrastructure |
| 📬 [ByteByteGo — High Availability Patterns](https://bytebytego.com) | Visual breakdowns of redundancy, failover, and availability zone design |

---

*⬅️ Previous: [Non-Functional System Characteristics](NonFunctionalSystemCharacteristics.md) &nbsp;|&nbsp; ➡️ Next: [Reliability](reliability.md)*

---

<sub>Part of the <a href="../../README.md">System Design Foundations</a> study guide series — Non-Functional System Characteristics.</sub>