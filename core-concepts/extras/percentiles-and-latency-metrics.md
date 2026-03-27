# 📊 Understanding Percentiles and Latency Metrics

> *"The average hides the truth. Percentiles reveal it."*

**⏱ Reading time:** ~8 minutes &nbsp;|&nbsp; **📍 Series:** Extras &nbsp;|&nbsp; **Referenced from:** [Throughput & Latency](../non-functional-system-characteristics/throughputAndLatency.md)

---

## Table of Contents

1. [What the "p" Means](#1-what-the-p-means)
2. [How to Calculate a Percentile](#2-how-to-calculate-a-percentile)
3. [Why Averages Lie](#3-why-averages-lie)
4. [The Percentiles You'll See in Practice](#4-the-percentiles-youll-see-in-practice)
5. [Reading a Percentile in Plain English](#5-reading-a-percentile-in-plain-english)
6. [A Worked Example End to End](#6-a-worked-example-end-to-end)
7. [What Good Looks Like](#7-what-good-looks-like)

---

## 1. What the "p" Means

**p stands for percentile.**

A percentile is a point in a dataset below which a given percentage of values fall. That's the formal definition — here's the intuitive one:

If you have 100 response times sorted from fastest to slowest, **p50 is the one in the middle**. Half are faster, half are slower. **p99 is the second-to-last one**. 99 values are faster than it, and only 1 is slower.

The "p" is just shorthand. p99 = "99th percentile." p50 = "50th percentile." Some teams write it as P99 or P50 — same thing.

```
100 response times, sorted fastest to slowest:

[4ms, 5ms, 5ms, 6ms, 6ms, 7ms, ... 11ms, 12ms, ... 95ms, 340ms]
 ↑                      ↑                   ↑              ↑
 fastest               p50                 p95            p99
                    (50th value)        (95th value)   (99th value)
```

---

## 2. How to Calculate a Percentile

The calculation is straightforward. Here's the step-by-step:

**Step 1: Collect your values.**
Let's say you recorded the response time of 20 requests (in ms):

```
45, 12, 78, 23, 56, 8, 34, 91, 17, 42, 
29, 63, 11, 38, 52, 19, 74, 27, 83, 15
```

**Step 2: Sort them from smallest to largest.**

```
8, 11, 12, 15, 17, 19, 23, 27, 29, 34,
38, 42, 45, 52, 56, 63, 74, 78, 83, 91
```

**Step 3: Find the index for the percentile you want.**

```
Formula: index = (percentile / 100) × total count

For p50 with 20 values:
  index = (50 / 100) × 20 = 10
  → The 10th value in the sorted list = 34ms

For p90 with 20 values:
  index = (90 / 100) × 20 = 18
  → The 18th value in the sorted list = 78ms

For p99 with 20 values:
  index = (99 / 100) × 20 = 19.8 → round up to 20
  → The 20th value in the sorted list = 91ms
```

**Step 4: That value is your percentile.**

```
p50  = 34ms   (half of requests were faster than this)
p90  = 78ms   (90% of requests were faster than this)
p99  = 91ms   (99% of requests were faster than this)
```

> 💡 **In practice**, you never calculate this by hand. Monitoring systems (Datadog, Prometheus, Grafana) compute percentiles continuously from the stream of response times and display them on dashboards. But knowing the mechanics tells you what the number *means* — which is what lets you reason about it correctly.

---

## 3. Why Averages Lie

The average response time is calculated by adding all values and dividing by the count. The problem is that **a few very slow requests can shift the average without revealing how bad they actually are.**

Using our 20 values from above:

```
Sum = 8+11+12+15+17+19+23+27+29+34+38+42+45+52+56+63+74+78+83+91
    = 826ms

Average = 826 / 20 = 41.3ms
```

41ms sounds fine. But look at what's hiding in that number:

```
Sorted values:
8, 11, 12, 15, 17, 19, 23, 27, 29, 34,   ← these 10 are all under 35ms
38, 42, 45, 52, 56, 63, 74, 78, 83, 91   ← these 10 range from 38ms to 91ms

p99 = 91ms  ← the worst experience a user can have
```

The average of 41ms is pulled upward by the slower requests but doesn't tell you that your worst users are waiting 91ms. At small scale (20 requests) this might not matter. At large scale, it absolutely does.

**A more dramatic example:**

Imagine 1,000 requests where 990 take 10ms and 10 take 10,000ms (10 seconds):

```
Average = ((990 × 10) + (10 × 10,000)) / 1,000
        = (9,900 + 100,000) / 1,000
        = 109,900 / 1,000
        = ~110ms

p50  = 10ms       ← most users are fine
p99  = 10,000ms   ← 1% of users are waiting 10 SECONDS
```

The average (110ms) makes the system look acceptable. The p99 (10,000ms) reveals a serious problem. **This is why teams that monitor only averages miss the users who are having a terrible experience.**

---

## 4. The Percentiles You'll See in Practice

| Percentile | What it tells you | When it matters |
|------------|-------------------|-----------------|
| **p50** (median) | The experience of a typical user — half are faster, half are slower | Good baseline but not sufficient alone |
| **p75** | Three quarters of users are at or below this | Sometimes used as a middle ground |
| **p90** | 90% of users are at or below this | Early warning for degradation |
| **p95** | 95% of users are at or below this | Common SLA target for internal services |
| **p99** | 99% of users are at or below this | Standard target for user-facing services |
| **p99.9** | 99.9% of users are at or below this | Used for high-stakes or high-volume systems |
| **p100** (max) | The single worst request ever recorded | Useful but volatile — one outlier dominates |

**p99 is the most commonly used target** in production system design. It captures the tail of your distribution — the worst experiences real users are having — without being dominated by one-in-a-million outliers the way p100 is.

**p99.9 matters when you have very high traffic.** At 10,000 requests/second, p99.9 affects 10 requests every second. Over an hour that's 36,000 users. At that scale, "one in a thousand" is no longer a rare edge case.

---

## 5. Reading a Percentile in Plain English

This is the translation that makes percentiles intuitive:

```
"Our p99 latency is 250ms"
→ "99% of our users get a response in 250ms or less.
   1% of our users wait longer than 250ms."

"Our p50 latency is 20ms"
→ "Half of our users get a response in 20ms or less.
   Half wait longer than 20ms."

"Our p99.9 latency is 2 seconds"
→ "99.9% of our users get a response in 2 seconds or less.
   0.1% of our users wait longer than 2 seconds."
```

The flip side reads just as naturally:

```
"Our p99 latency is 250ms"
→ "1 in every 100 requests takes longer than 250ms"

At 1,000 req/sec: that's 10 slow requests every second
At 10,000 req/sec: that's 100 slow requests every second
```

Translating to request counts per second is useful because it makes abstract percentages feel real. "1%" sounds negligible. "100 users per second experiencing a slow response" sounds like something worth fixing.

---

## 6. A Worked Example End to End

Let's say you're running a search API and you record these 10 response times over one second (in ms):

```
Requests: 14, 22, 18, 35, 11, 28, 19, 310, 21, 16
```

**Step 1: Sort them.**
```
11, 14, 16, 18, 19, 21, 22, 28, 35, 310
```

**Step 2: Calculate percentiles.**
```
p50:  index = 0.50 × 10 = 5  → 5th value = 19ms
p90:  index = 0.90 × 10 = 9  → 9th value = 35ms
p99:  index = 0.99 × 10 = 9.9 → round up to 10 → 10th value = 310ms
```

**Step 3: What does this tell you?**

```
p50  = 19ms   → Typical users are getting fast responses ✓
p90  = 35ms   → 90% of users are well within any reasonable budget ✓
p99  = 310ms  → 1% of users are waiting 310ms — 16× longer than the median ⚠️
Average = (11+14+16+18+19+21+22+28+35+310) / 10 = 49.4ms
```

The average of 49ms doesn't hint at the 310ms outlier. But p99 does. That one slow request — probably a cache miss that fell through to a slow database query — is the signal worth investigating.

In a real monitoring dashboard, this would appear as a spike on the p99 graph while the p50 graph stays flat. That pattern — p99 diverging from p50 — is a classic sign of **tail latency caused by intermittent slow operations** (cache misses, lock contention, GC pauses, slow replicas).

---

## 7. What Good Looks Like

There's no universal "good" latency number — it depends entirely on what your system is doing. But there are common reference points:

| System type | Typical p99 target |
|-------------|-------------------|
| Real-time systems (game servers, trading) | < 10ms |
| User-facing APIs (search, feed) | < 100ms |
| Web page loads | < 1,000ms (1 second) |
| Background jobs / async processing | seconds to minutes (latency is not the concern — throughput is) |

The more important number than the absolute value is the **ratio between your percentiles**. A healthy system has percentiles that are relatively close together:

```
Healthy:
  p50 = 15ms, p95 = 45ms, p99 = 80ms
  → p99 is ~5× the median. Reasonable spread.

Unhealthy:
  p50 = 15ms, p95 = 60ms, p99 = 850ms
  → p99 is ~57× the median. Something is very wrong for 1% of users.
```

A large gap between p50 and p99 almost always points to an intermittent problem — something that's slow sometimes but not always. Cache misses, lock contention, slow replicas, garbage collection pauses. The wide spread is the diagnostic signal; finding the cause is the investigation.

---

*↩ Back to [Throughput & Latency](../non-functional-system-characteristics/throughputAndLatency.md)*

---

<sub>Part of the <a href="../../README.md">System Design Foundations</a> study guide series — Extras.</sub>