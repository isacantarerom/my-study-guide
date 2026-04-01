# 📋 How to Define Requirements and Constraints

> *"The most expensive mistake in system design is building the right system for the wrong requirements."*

**⏱ Reading time:** ~10 minutes &nbsp;|&nbsp; **📍 Series:** Extras &nbsp;|&nbsp; **Connects to:** [RESHADED — Step R](how-to-approach-system-design.md)

---

## Table of Contents

1. [Why Requirements Come First](#1-why-requirements-come-first)
2. [Functional vs Non-Functional Requirements](#2-functional-vs-non-functional-requirements)
3. [How to Extract Requirements From a Vague Problem](#3-how-to-extract-requirements-from-a-vague-problem)
4. [Constraints — The Hidden Third Category](#4-constraints--the-hidden-third-category)
5. [How to Prioritize Requirements](#5-how-to-prioritize-requirements)
6. [Requirements as Design Drivers](#6-requirements-as-design-drivers)
7. [Worked Example: From Vague to Specific](#7-worked-example-from-vague-to-specific)
8. [Common Requirements Mistakes](#8-common-requirements-mistakes)

---

## 1. Why Requirements Come First

Every architectural decision is either correct or incorrect *relative to a set of requirements*. Without requirements, there is no such thing as a right answer — only preferences.

"Should we use a relational or NoSQL database?" — depends on the data model and access patterns (requirements).
"Do we need a cache?" — depends on the read volume and latency target (requirements).
"Should writes be synchronous or async?" — depends on whether the user needs an immediate confirmation (requirements).

Requirements are what turn architectural choices from opinions into defensible decisions. They're also what let you know when a design is good enough — because "good enough" means "meets the requirements," not "uses the most impressive technology."

Skipping requirements and jumping straight to architecture is one of the most common and most expensive mistakes in system design. You build something technically impressive that solves the wrong problem.

---

## 2. Functional vs Non-Functional Requirements

Requirements fall into two categories that feel different and drive different design decisions.

### Functional Requirements
What the system **does**. The features, behaviors, and operations users can perform.

These are stated as capabilities:
- "Users can upload photos"
- "The system sends email notifications when an order ships"
- "Administrators can deactivate user accounts"
- "The search returns results ranked by relevance"

Functional requirements define the *scope* of the system — what it is and isn't responsible for.

### Non-Functional Requirements
How **well** the system does it. The qualities, constraints, and service levels.

These are stated as measurable targets:
- "Search results must return in under 200ms at p99"
- "The system must be available 99.99% of the time"
- "User data must be encrypted at rest and in transit"
- "The system must handle 500,000 concurrent users"
- "Data must be retained for 7 years for regulatory compliance"

Non-functional requirements define the *constraints* on the design — the boundaries that solutions must stay within.

```
A useful test:
  Functional  → "The system can do X"
  Non-functional → "The system does X within Y constraint"

  "Users can search for products"         ← Functional
  "Search returns results in < 200ms"     ← Non-functional

  "Orders can be placed"                  ← Functional
  "Orders must never be double-processed" ← Non-functional (reliability)
  "Order placement handles 10,000 RPS"    ← Non-functional (scalability)
```

Both categories matter. A system that has all the right features but is too slow, too unreliable, or doesn't scale is a failed system. A system that's fast and available but does the wrong things is also a failed system.

---

## 3. How to Extract Requirements From a Vague Problem

Real requirements are rarely handed to you fully formed. "Design Instagram" tells you almost nothing specific. Extracting useful requirements means asking the right questions.

### Questions for Functional Requirements

```
What are the core user actions?
  → What can users create, read, update, delete?
  → What are the primary workflows from the user's perspective?

What is explicitly out of scope?
  → Defining what the system doesn't do is as important as
     defining what it does. Scope creep is how systems become
     unmaintainable.

Who are the different types of users?
  → Consumers vs administrators vs third-party integrations?
  → Each type may have different capabilities and access levels.

What are the edge cases that matter?
  → What happens when a user uploads a duplicate photo?
  → What happens when a payment fails mid-transaction?
  → These often reveal requirements that weren't obvious upfront.
```

### Questions for Non-Functional Requirements

```
What scale does this need to handle?
  → How many users total? Daily active? Concurrent?
  → This drives estimation and eventually the entire architecture.

What are the latency expectations?
  → Are users waiting for a response, or is this a background job?
  → Which operations are latency-sensitive?

What availability is required?
  → 99.9%? 99.99%? The difference is the difference between
     8 hours and 52 minutes of allowed downtime per year.

What are the consistency requirements?
  → Is eventual consistency acceptable, or must users always see
     the latest data?

What are the durability requirements?
  → Can we lose data in a failure, or must every write be permanent?

Are there regulatory or compliance constraints?
  → GDPR, HIPAA, PCI-DSS all impose specific requirements on
     how data is stored, accessed, and deleted.
```

---

## 4. Constraints — The Hidden Third Category

Beyond functional and non-functional requirements, every real system has **constraints** — hard limits imposed by reality rather than by the problem statement.

```
Types of constraints:

Technical constraints
  → "We must use the existing PostgreSQL infrastructure"
  → "The mobile app cannot exceed 50MB in size"
  → "We're on AWS and can't use Google Cloud services"

Team constraints
  → "We have 3 engineers for 6 months"
  → "No one on the team has Kubernetes experience"

Time constraints
  → "Must be live in 8 weeks for a product launch"
  → "Must integrate with a legacy system that can't be changed"

Budget constraints
  → "Infrastructure budget is $10,000/month"
  → "Can't use paid third-party services for core functionality"

Regulatory constraints
  → "User data cannot leave the EU" (GDPR)
  → "Payment card data must not be stored" (PCI-DSS)
  → "Audit logs must be retained for 7 years"
```

Constraints are often more powerful than requirements in shaping design decisions, because they're non-negotiable. A requirement can be deprioritized or phased. A constraint cannot be worked around — it must be designed within.

Surfacing constraints early prevents wasted design work. A beautiful distributed architecture designed around eventual consistency is worthless if a regulatory requirement mandates strong consistency for all user data.

---

## 5. How to Prioritize Requirements

Not all requirements are equal. When time and resources are limited — which is always — you need to know which requirements are the foundation and which are enhancements.

A useful three-tier model:

```
Must-have (P0)
  Core functionality without which the system has no value.
  The system cannot launch without these.
  Example: "Users can shorten URLs and have them redirect"

Should-have (P1)
  Important features that significantly improve the system
  but can be added shortly after launch.
  Example: "Users can see click analytics for their links"

Nice-to-have (P2)
  Valuable enhancements that can be deferred without blocking
  the core use case.
  Example: "Users can set custom short codes"
```

This prioritization also applies to non-functional requirements. A startup with 1,000 users designing for five-nines availability is misallocating engineering effort. A payment system accepting that occasional duplicate charges are fine is under-investing in reliability.

The right level of non-functional investment is the one that matches your actual risk profile — not the most impressive spec you can write.

---

## 6. Requirements as Design Drivers

The real purpose of requirements is to force architectural decisions. Each requirement, when taken seriously, points toward specific design choices.

```
Requirement                         → Design implication
─────────────────────────────────────────────────────────────────
"Handle 1M concurrent users"        → Horizontal scaling, stateless
                                      services, load balancers

"Search results under 100ms"        → Caching, search index,
                                      CDN for static assets

"99.99% availability"               → Redundancy, multi-AZ deployment,
                                      circuit breakers, health checks

"Never lose a financial transaction" → ACID transactions, write-ahead
                                      logging, synchronous replication

"User data cannot leave EU"         → Regional deployment, data
                                      residency enforcement at storage layer

"Support 100TB of user uploads/day" → Object storage, CDN,
                                      async processing pipeline
```

This is how requirements become architecture. Each one is a constraint that rules out some options and points toward others. When you see a design decision that seems arbitrary ("why are we using Kafka here?"), finding the requirement it answers makes it defensible ("because we need to process 500,000 events/second asynchronously, and Kafka is built for exactly that").

---

## 7. Worked Example: From Vague to Specific

**Vague problem:** "Design a notification system."

**Step 1: Ask clarifying questions**

```
What types of notifications? Email only? Push? SMS? In-app?
Who triggers notifications? Users? System events? Third parties?
What's the volume? 100 notifications/day or 10 million?
How fast must they deliver? Real-time or eventual?
What happens if a notification fails to deliver?
Are there user preferences (opt-out, quiet hours)?
```

**Step 2: Derive functional requirements**

```
Core (P0):
  - System can send email, push, and SMS notifications
  - Notifications are triggered by system events (order placed,
    password reset, new message received)
  - Users can opt out of non-critical notification types

Important (P1):
  - Users can set quiet hours (no notifications 10pm-8am)
  - Notification history is viewable in-app for 30 days

Nice-to-have (P2):
  - Users can choose notification channel per type
    (e.g., "send me order updates via SMS, not email")
```

**Step 3: Derive non-functional requirements**

```
Scale:
  - 50 million DAU
  - Average 5 notifications per user per day = 250M notifications/day
  - Peak: 3× average = ~8,700 notifications/second at peak

Latency:
  - Critical notifications (password reset, security alerts):
    must deliver within 10 seconds
  - Standard notifications (order updates, social):
    within 5 minutes is acceptable
  - Marketing/promotional: within 1 hour acceptable

Reliability:
  - Critical notifications: must be delivered at least once
    (accept duplicate risk over missed delivery)
  - Marketing notifications: best effort, dropped if
    system is under load is acceptable

Availability:
  - 99.9% (notification delay is acceptable; total loss is not)
```

**Step 4: Identify constraints**

```
Technical: Must integrate with existing user service for
           preferences and opt-out status
Regulatory: GDPR — users in EU must be able to delete
            notification history on request
Budget: Third-party notification providers (Twilio for SMS,
        FCM for push) must stay under $X/month
```

**The result:** A vague "design a notification system" has become a specific set of requirements that immediately suggests: async processing (250M/day can't be synchronous), tiered delivery (critical vs standard vs marketing have different pipelines), at-least-once delivery with deduplication for critical types, and a fan-out architecture to multiple channels.

None of those architectural decisions were obvious from "design a notification system." They emerged from the requirements.

---

## 8. Common Requirements Mistakes

**Designing before requirements are clear.** The instinct to start drawing boxes and choosing databases is strong. Resist it. Five minutes spent clarifying requirements prevents hours of redesign.

**Confusing functional and non-functional requirements.** "The system is fast" is not a requirement — it's a wish. "The system returns search results in under 150ms at p99 under normal load" is a requirement. Non-functional requirements must be specific and measurable.

**Over-specifying requirements for a system at early stage.** A system with 1,000 users doesn't need five-nines availability or global CDN distribution. Requirements should match actual risk and scale, not aspirational future state.

**Forgetting the unhappy paths.** Requirements usually describe what should happen. The most important edge cases are what happens when things go wrong — payment fails, network drops, user uploads a corrupted file. Failure behavior is a requirement.

**Not distinguishing must-haves from nice-to-haves.** Everything feels essential until you run out of time or money. Prioritization upfront prevents painful de-scoping decisions later under pressure.

---

*↩ Back to [Extras Index](ExtrasIndex.md) &nbsp;|&nbsp; → Next: [How to Discuss Trade-offs](how-to-discuss-tradeoffs.md)*

---

<sub>Part of the <a href="../../README.md">System Design Foundations</a> study guide series — Extras.</sub>