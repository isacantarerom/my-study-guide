# 🧮 Back-of-the-Envelope Calculations

> *"A rough answer to the right question is worth more than a precise answer to the wrong one."*

---

## What This Section Covers

Before you can design a system well, you need to know roughly what it has to handle. How many requests per second? How much data per day? How many servers? Without numbers — even rough ones — design decisions are just guesses dressed up as architecture.

Back-of-the-envelope calculations are the skill of estimating these numbers quickly, confidently, and accurately enough to be useful. Not exact. Not precise to three decimal places. Just close enough to make good decisions and catch designs that are wildly off.

This section covers two things: the reference numbers worth memorizing (the building blocks of every estimate), and the method for turning those numbers into useful system estimates.

---

## Guides

| # | Topic | Description | Guide |
|---|-------|-------------|-------|
| 1 | **Resource Estimation Fundamentals** | The numbers every engineer should know — latency, storage, bandwidth, and compute reference points — and how to use them. | [Read →](resource-estimation-fundamentals.md) |
| 2 | **Estimation in Practice** | A step-by-step method for working through capacity estimates in any system design, with worked examples. | [Read →](estimation-in-practice.md) |

---

## Why This Section Exists Where It Does

```
Preliminary Concepts
(how distributed systems behave)
        │
        ▼
Non-Functional Characteristics
(what properties we want systems to have)
        │
        ▼
Back-of-the-Envelope Calculations   ← You are here
(how much of everything we need)
        │
        ▼
Building Blocks
(the actual components we'll use to build it)
```

Understanding what a system needs to handle — before picking the components — is what keeps design decisions grounded. It's the difference between "we'll use Kafka because Kafka is cool" and "we need to process 50,000 events/second, so let's look at what can handle that."

---

## The Mindset Going In

The goal of a back-of-the-envelope calculation is not precision — it's **order of magnitude correctness**. Being off by 2x is fine. Being off by 100x means your design is wrong.

The numbers you'll estimate fall into a few categories:

- **Traffic** — requests per second, events per day
- **Storage** — bytes per record, total data over time
- **Bandwidth** — data in and out per second
- **Compute** — how many servers at what spec

Each one follows the same pattern: start with one user doing one thing, scale to your user base, account for peaks.

---

*⬅️ Back to [README](../../README.md) &nbsp;|&nbsp; ➡️ First Guide: [Resource Estimation Fundamentals](resource-estimation-fundamentals.md)*