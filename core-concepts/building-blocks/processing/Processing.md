# ⚙️ Processing

> *"Not all work happens in the request path. Some work happens later, in the background, scheduled — and it still needs to be done correctly."*

---

## The Problem This Group Solves

Two specific problems come up constantly in distributed systems that don't fit neatly into storage or communication:

**The background work problem** — some operations take too long or consume too many resources to run synchronously in a user's request. Video transcoding, report generation, sending bulk emails, running ML inference — these need to happen, but not in the milliseconds a user is waiting. They need to be scheduled, distributed across workers, retried on failure, and completed reliably in the background.

**The identity problem** — in a distributed system, multiple services running in parallel all need to create unique identifiers. How do you generate a globally unique ID when there's no single central authority, and you need billions of them per day, and they need to be sortable by time?

These two building blocks solve those two problems.

---

## The Components

| Building Block | Solves | Guide |
|---------------|--------|-------|
| **Distributed Task Scheduler** | Running background jobs reliably across distributed workers | [Read →](distributed-task-scheduler.md) |
| **Sequencer** | Generating unique, ordered IDs across distributed nodes without coordination | [Read →](sequencer.md) |

---

## Why These Two Live Together

At first glance these seem unrelated — one schedules jobs, one generates IDs. The connection is that both exist to support work that happens *outside the main request path*:

- The task scheduler decides *when and where* background work runs
- The sequencer gives that work (and every record it creates) a *unique, traceable identity*

In practice, every task the scheduler creates needs a unique ID. Every job that runs needs to produce records with unique IDs. The sequencer is the building block that makes the task scheduler's output identifiable and orderable across the distributed system.

```
User uploads a video
        │
        ▼
API creates a Job record
  job_id = Sequencer.next()  ← Sequencer generates unique ordered ID
  job_type = "transcode_video"
  status = "pending"
        │
        ▼
Task Scheduler picks up the job
  Assigns to available worker
  Worker transcodes video
  Updates job status
  Stores results with IDs from Sequencer
        │
        ▼
User can query job_id to check status
  "Your video is processing" → "Your video is ready"
```

---

## The Sequencer's Hidden Importance

Sequencers are easy to underestimate because they seem simple — just generate a unique ID, right? But the requirements are subtle:

**Uniqueness** — no two IDs can ever be the same, across any node, ever.
**Ordering** — IDs generated later should be greater than IDs generated earlier (sortable by creation time without a separate timestamp field).
**Performance** — generating IDs cannot be a bottleneck. Systems that create millions of records per second need millions of IDs per second.
**No single point of failure** — a central ID server is a SPOF. The solution must be distributed.

Satisfying all four simultaneously is genuinely hard, which is why dedicated solutions like Twitter's Snowflake and UUID v7 exist.

---

## When You Reach for This Group

Your system has work that takes more than a few hundred milliseconds → **Task Scheduler** to run it async in the background.

You need to run jobs on a schedule (daily reports, weekly digests, nightly cleanup) → **Task Scheduler**.

Your system creates records across multiple distributed services and needs to track them globally → **Sequencer** for unique ordered IDs.

You need to reconstruct the order events happened in across services that don't share a clock → **Sequencer** with causality guarantees.

---

*⬅️ Previous Group: [Communication](../communication/Communication.md) &nbsp;|&nbsp; ➡️ Next Group: [Observability](../observability/Observability.md)*