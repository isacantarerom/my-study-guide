# 📎 Extras

> *Concepts that support the main guides without belonging to any single section.*

---

## What This Folder Is

The guides in `core-concepts/` follow the Grokking System Design syllabus. This folder is for everything that makes those guides more useful but doesn't fit neatly into a chapter — supporting concepts, thinking frameworks, reference material, and deep dives on specific mechanics.

Read these when a main guide references them, or when you want to go deeper on something specific.

---

## Guides

| Guide | What it covers | When to read it |
|-------|----------------|-----------------|
| [How to Approach a System Design Problem](how-to-approach-system-design.md) | The RESHADED framework — a structured method for going from a vague problem to a coherent design | Before tackling any full system design, or when a design feels unstructured |
| [How to Define Requirements and Constraints](how-to-define-requirements.md) | How to extract functional and non-functional requirements from a vague problem statement | Before starting any design — requirements are always step one |
| [How to Discuss Trade-offs](how-to-discuss-tradeoffs.md) | The anatomy of a well-reasoned trade-off and the common trade-off pairs in system design | Whenever you're making or defending a design decision |
| [Understanding Percentiles and Latency Metrics](percentiles-and-latency-metrics.md) | What p50, p95, p99 mean, how to calculate them, and why averages mislead | When reading anything about latency — referenced from [Throughput & Latency](../non-functional-system-characteristics/throughputAndLatency.md) |
| [GRPC vs REST](grpc-vs-rest.md) | The two stacks compared gRPC vs REST and Protobuf vs JSON | Referenced from [Throughput & Latency](../non-functional-system-characteristics/throughputAndLatency.md) |
|[Percentiles and Latency Metrics](percentiles-and-latency-metrics.md)| Understanding Percentiles and Latency Metrics | When reading metrics about latency — referenced from [Throughput & Latency](../non-functional-system-characteristics/throughputAndLatency.md) |
|[SSD vs HDD](ssd-vs-hdd.md)| What are they and when to use them. | When reading Disk I/O Latency in [Throughput & Latency](../non-functional-system-characteristics/throughputAndLatency.md)|

---

## How These Connect to the Main Guides

```
How to Approach a System Design Problem (RESHADED)
  └── Is the meta-framework that holds everything else together.
      Every step in RESHADED connects to a section of the syllabus:
        R (Requirements) → How to Define Requirements
        E (Estimation)   → Back-of-the-Envelope Calculations
        S,H,A,D          → Building Blocks (components you'll use)
        E (Evaluation)   → Non-Functional Characteristics (what to check)
        D (Deep Dive)    → System Design Examples

How to Define Requirements
  └── Feeds directly into RESHADED Step R
      Also connects to Non-Functional Characteristics
      (non-functional requirements ARE the characteristics)

How to Discuss Trade-offs
  └── Applies everywhere — every design decision in every guide
      is a trade-off. This guide gives you the language for it.
      Particularly relevant alongside Consistency Models, CAP/PACELC,
      and any Building Block that requires a storage or architecture choice.

Percentiles and Latency Metrics
  └── Supporting reference for Throughput & Latency
      Also relevant for Availability (SLA latency targets)
```

---

*⬅️ Back to [README](../../README.md)*