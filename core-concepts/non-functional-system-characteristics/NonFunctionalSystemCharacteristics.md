# ⚙️ Non-Functional System Characteristics

> *Functional requirements define what a system does. Non-functional requirements define how well it does it — and whether it keeps doing it when the world gets messy.*

---

## What This Section Covers

When we design a system, we spend a lot of time on the functional side — what features exist, what the API looks like, how data flows between services. But the questions that determine whether a system actually holds up in production are non-functional: Can it handle 10x the load? Does it stay up when a data center fails? Does it stay fast as it grows?

These characteristics aren't features you add at the end. They're properties you design for from the start — and they constantly pull against each other. A system optimized purely for availability may sacrifice consistency. A system optimized purely for performance may sacrifice fault tolerance. Understanding what each characteristic means, why it matters, and what it costs you to achieve it is what lets us make those tradeoffs deliberately instead of accidentally.

This section covers the core non-functional characteristics that shape every real system design decision.

---

## Guides

| # | Topic | Description | Guide |
|---|-------|-------------|-------|
| 1 | **Availability** | What uptime really means, how it's measured, and what it actually takes to achieve five nines. | [Read →](availability.md) |
| 2 | **Reliability** | The difference between a system that's up and one that's correct — and why that distinction matters. | [Read →](reliability.md) |
| 3 | **Scalability** | How systems grow to handle more — horizontal vs vertical scaling, stateless design, and where growth actually breaks things. | [Read →](scalability.md) |
| 4 | **Maintainability** | Why systems become harder to change over time, and how good design fights that entropy from the start. | [Read →](maintainability.md) |
| 5 | **Fault Tolerance** | Designing systems that keep working when parts of them fail — redundancy, failover, and graceful degradation. | [Read →](faultTolerance.md) |
| 6 | **Throughput & Latency** | The two dimensions of performance, why they pull against each other, and how to reason about both. | [Read →](throughputAndLatency.md) |

---

## How These Characteristics Relate

These aren't six independent topics — they form a web of tradeoffs. Pulling on one usually affects the others.

```
                        Availability
                       (stay up always)
                            │
              ┌─────────────┼─────────────┐
              │             │             │
              ▼             ▼             ▼
        Reliability    Fault Tolerance  Scalability
       (be correct)   (survive failure) (grow smoothly)
              │             │             │
              └─────────────┼─────────────┘
                            │
                 ┌──────────┴──────────┐
                 ▼                     ▼
           Throughput              Latency
          (how much)             (how fast)
                 └──────────┬──────────┘
                            │
                     Maintainability
                  (keep it manageable
                   as complexity grows)
```

A few tradeoffs worth keeping in mind as you read:

- **Availability vs Consistency** — achieving high availability often means accepting eventual consistency (as we saw in [Consistency Models](../preliminary-system-design-concepts/consistencyModels.md))
- **Throughput vs Latency** — optimizing for raw throughput (processing more requests) often increases latency per individual request
- **Fault Tolerance vs Cost** — redundancy is the mechanism behind fault tolerance, and redundancy costs money and complexity
- **Scalability vs Maintainability** — distributed systems that scale well are often harder to reason about, debug, and evolve

Understanding these tensions is more useful than memorizing definitions. Every real design decision is a negotiation between them.

---

## How This Section Connects to What Came Before

The [Preliminary Concepts](../preliminary-system-design-concepts/PreliminarySystemDesignConcepts.md) section established how distributed systems behave and fail. This section is about *what we want them to do despite that* — the properties we're trying to guarantee in the face of the failures and consistency challenges we already understand.

```
Failure Models ──────────────────────────────────────────────────────────►
                                                                          │
    We know systems fail in predictable ways.                             │
    Non-functional characteristics are the goals                         │
    we design toward in spite of those failures.                          │
                                                                          ▼
                          Non-Functional Characteristics
                    (Availability, Reliability, Scalability,
                     Maintainability, Fault Tolerance, Performance)
```

---

*⬅️ Back to [README](../../README.md) &nbsp;|&nbsp; ➡️ First Guide: [Availability](availability.md)*

*⬅️ Previous Section [Preliminary System Design Concepts](../preliminary-system-design-concepts/PreliminarySystemDesignConcepts.md) &nbsp;|&nbsp; ➡️ Next Section: [tbd](../../README.md)*
