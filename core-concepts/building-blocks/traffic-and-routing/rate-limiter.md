# 🚦 Rate Limiter

> *"A system that accepts every request is a system that one bad actor can bring down."*

**⏱ Reading time:** ~12 minutes &nbsp;|&nbsp; **📍 Series:** Building Blocks &nbsp;|&nbsp; **🗂 Group:** Traffic & Routing

---

## Table of Contents

1. [What Problem a Rate Limiter Solves](#1-what-problem-a-rate-limiter-solves)
2. [Where Rate Limiting Lives in the System](#2-where-rate-limiting-lives-in-the-system)
3. [What to Rate Limit On](#3-what-to-rate-limit-on)
4. [Rate Limiting Algorithms](#4-rate-limiting-algorithms)
5. [Distributed Rate Limiting](#5-distributed-rate-limiting)
6. [Rate Limit Response — What to Return](#6-rate-limit-response--what-to-return)
7. [Rate Limiting Failure Modes](#7-rate-limiting-failure-modes)
8. [How Rate Limiters Connect to Other Building Blocks](#8-how-rate-limiters-connect-to-other-building-blocks)
9. [Self-Check](#9-self-check)
10. [References](#10-references)

---

## 1. What Problem a Rate Limiter Solves

A public API or web service has no control over who calls it or how often. Without limits, a single bad actor — or even a well-intentioned client with a runaway retry loop — can send millions of requests per second, consuming all available resources and denying service to legitimate users.

Rate limiting is the mechanism that enforces a maximum request rate, ensuring no single client consumes a disproportionate share of capacity.

But rate limiting isn't just about protection from abuse. It serves several distinct purposes:

**Protecting system stability** — prevents any single client from overwhelming backend services, regardless of intent.

**Ensuring fair usage** — in a multi-tenant system, one high-volume client shouldn't degrade performance for everyone else.

**Enforcing business rules** — free tier gets 100 API calls/day; paid tier gets 10,000. Rate limiting implements the product boundary.

**Preventing DDoS amplification** — limits the blast radius of distributed denial-of-service attacks.

**Cost control** — in systems where requests cost money (calling paid third-party APIs, LLM inference), rate limiting prevents runaway costs.

---

## 2. Where Rate Limiting Lives in the System

Rate limiting typically happens at the API gateway or load balancer layer — before requests reach application servers.

```
Client
  │
  ▼
DNS → CDN (static assets bypass rate limiting)
  │
  ▼
Load Balancer / API Gateway
  │
  ├── Rate Limiter checks: "Has this client exceeded their limit?"
  │       YES → Return 429 Too Many Requests (immediately)
  │       NO  → Continue to application
  │
  ▼
Application Servers
```

Placing rate limiting early means rejected requests consume minimal resources — the rate limiter fires before any expensive application logic runs.

Some systems implement multiple rate limiting layers:

```
Layer 1: CDN edge (coarse filtering — blocks obvious abuse at the edge)
Layer 2: API Gateway (per-client, per-endpoint rate limiting)
Layer 3: Application (fine-grained business logic limits)
Layer 4: Service-to-service (prevents internal services from overwhelming each other)
```

---

## 3. What to Rate Limit On

The granularity of rate limiting determines what counts as "one client":

**By IP address** — easiest to implement. One IP = one client. Problem: multiple users can share an IP (corporate networks, mobile carriers with CGNAT), and a sophisticated attacker uses many IPs.

**By user ID / API key** — more accurate for authenticated APIs. Each account has its own limit, regardless of IP. Doesn't protect unauthenticated endpoints.

**By API key** — common for developer APIs (Stripe, Twilio, GitHub). Each API key has its own quota. Allows fine-grained tiering (free vs paid keys have different limits).

**By endpoint** — different limits for different operations. A write operation might be limited to 100/minute while reads get 1,000/minute. Expensive operations get tighter limits.

**By user + endpoint** — the most precise. "This user can call `/api/send-message` 60 times per minute, and `/api/search` 100 times per minute."

Real systems combine multiple dimensions:

```
Example: Twitter API limits

Per user, per app:
  - POST /tweets: 300 per 3 hours
  - GET /home_timeline: 15 per 15 minutes
  - GET /search: 180 per 15 minutes

Per app (across all users):
  - GET /home_timeline: 15,000 per 15 minutes
```

---

## 4. Rate Limiting Algorithms

The algorithm determines how the rate limit is calculated and enforced. Each makes different tradeoffs between accuracy, burst tolerance, and implementation complexity.

### Token Bucket

The most common algorithm. Imagine a bucket that holds tokens. Tokens are added at a fixed rate (the refill rate). Each request consumes one token. If the bucket is empty, the request is rejected.

```
Bucket capacity: 10 tokens (maximum burst)
Refill rate: 2 tokens/second

t=0:  Bucket = 10 tokens
t=0:  5 requests arrive → 5 tokens consumed → Bucket = 5 tokens
t=1:  2 tokens added → Bucket = 7 tokens
t=1:  8 requests arrive → 7 tokens consumed → 1 rejected → Bucket = 0
t=2:  2 tokens added → Bucket = 2 tokens
```

**Key property:** Allows short bursts (up to bucket capacity) while enforcing a long-term average rate (the refill rate). A user can "save up" tokens during quiet periods and spend them on a burst.

**Good for:** APIs where clients need to occasionally burst (sending a batch of messages) but must stay within a long-term average.

### Leaky Bucket

Requests flow in at any rate; they're processed at a fixed rate. Like water in a bucket with a hole — it drains at a steady rate regardless of how fast water pours in. Excess overflows (is rejected).

```
Processing rate: 2 requests/second

Requests arrive: 10 requests at once
Queue: 10 pending
Process at 2/second: after 5 seconds, all processed

If queue is full and more arrive: reject (overflow)
```

**Key property:** Output rate is always smooth and predictable. No bursts.
**Good for:** Payment processing, audit logging — anything where you need a steady, predictable output rate.

### Fixed Window Counter

Count requests in fixed time windows (each minute, each hour). Reset the counter at the start of each window.

```
Limit: 100 requests per minute
Window: 14:00:00 - 14:00:59

Requests at 14:00: 40
Requests at 14:00: 50 more → Total = 90 ✓
Requests at 14:00: 20 more → Total = 110 → 10 rejected
Window resets at 14:01:00 → Counter = 0
```

**Problem:** Boundary attack. A client can make 100 requests at 14:00:59 and 100 more at 14:01:00 — 200 requests in 2 seconds, double the intended limit.

```
14:00:58-14:01:02:
  100 requests → accepted (end of first window)
  100 requests → accepted (start of second window)
  200 requests in 4 seconds ← double the intended rate
```

### Sliding Window Log

Track the exact timestamp of each request. Count how many requests fall within the last N seconds.

```
Limit: 100 requests per minute

At 14:05:30, client makes request:
  Count requests since 14:04:30
  If < 100: allow and log timestamp
  If ≥ 100: reject
```

**Pros:** Accurate — no boundary attack.
**Cons:** Memory-intensive — must store timestamps for every request in the window. At high volume, the log becomes large.

### Sliding Window Counter

Combines the simplicity of fixed windows with the accuracy of sliding windows. Uses two adjacent window counters and a weighted calculation.

```
Limit: 100 requests per minute
Current window (14:05): 30 requests so far (40% through the minute)
Previous window (14:04): 80 requests

Estimated count = (previous × (1 - elapsed fraction)) + current
                = (80 × 0.60) + 30
                = 48 + 30
                = 78

78 < 100: allow
```

**Pros:** Low memory (only 2 counters), reasonably accurate, no boundary attacks.
**Good for:** Most production rate limiting. The sliding window counter is typically the right default.

---

## 5. Distributed Rate Limiting

A single rate limiter is straightforward. A distributed rate limiter — where multiple rate limiter instances serve the same API across many servers — is significantly harder.

**The problem:** If you have 10 load-balanced servers, each with its own in-memory rate limiter, a client can make 100 requests to each server — 1,000 total — while staying within the 100-request limit on each individual server.

```
Single server (correct):          Distributed (incorrect without coordination):
  Client → Server A                 Client → Server A: 100 requests ✓
  100 requests → limit hit          Client → Server B: 100 requests ✓
                                    Client → Server C: 100 requests ✓
                                    Total: 300 requests — bypass!
```

**The solution:** Centralized counter storage. All rate limiter instances share a single counter in Redis (or another fast distributed store). Every request check and count update goes through Redis.

```
Request arrives at any server
         │
         ▼
Rate Limiter: "How many requests has this client made?"
         │
         ▼  (atomic read-increment)
Redis: INCR "rate:user:123:minute:1420"
       Returns: 47
         │
         ▼
47 < 100? Allow ✓
47 ≥ 100? Reject 429 ✗
```

**The tradeoff:** Redis adds a network hop to every request check. For high-throughput APIs, this latency adds up. Optimizations include local caching with periodic sync to Redis (accepting slight inaccuracy) or using Redis' built-in Lua scripting for atomic operations.

---

## 6. Rate Limit Response — What to Return

When a request is rejected, the response should be informative enough for clients to handle it gracefully.

**HTTP 429 Too Many Requests** is the standard status code.

**Informative headers tell clients when to retry:**

```http
HTTP/1.1 429 Too Many Requests
Content-Type: application/json
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1672531260
Retry-After: 30

{
  "error": "rate_limit_exceeded",
  "message": "You have exceeded the rate limit of 100 requests per minute.",
  "retry_after_seconds": 30
}
```

- `X-RateLimit-Limit`: the maximum allowed
- `X-RateLimit-Remaining`: how many requests are left in the current window
- `X-RateLimit-Reset`: Unix timestamp when the limit resets
- `Retry-After`: seconds until the client can retry

Well-designed rate limit responses allow clients to implement **exponential backoff** properly rather than hammering the API and making the problem worse.

---

## 7. Rate Limiting Failure Modes

### The Redis Dependency
If your rate limiter depends on Redis and Redis goes down, you have a choice: **fail open** (allow all requests, remove rate limiting temporarily) or **fail closed** (reject all requests, deny service to everyone).

Failing open is usually preferable — a brief window without rate limiting is better than a complete outage. But this depends on your threat model and the criticality of the limits.

### False Positives at Shared IPs
IP-based rate limiting can block entire corporate networks or mobile ISPs behind NAT. Thousands of legitimate users sharing one IP can collectively hit the limit, causing all of them to be blocked even though no individual is abusing the API.

**Mitigation:** Rate limit by user ID for authenticated endpoints. Use IP limiting only for unauthenticated endpoints where user ID isn't available.

### Client Thundering Herd on Reset
If all rate-limited clients know the exact second the window resets and all retry simultaneously, you get a thundering herd — a massive spike every minute as thousands of rate-limited clients all try again at once.

**Mitigation:** Add jitter (randomness) to the reset time in `Retry-After` headers, or use sliding windows that don't have a synchronized reset moment.

---

## 8. How Rate Limiters Connect to Other Building Blocks

```
DNS + CDN
  └──► Rate Limiter (checks before requests reach application)
            │
            ├── Rejected: Return 429 immediately
            │
            └── Allowed: Load Balancer → Application Servers
                                │
                                └──► Redis (shared counter storage
                                           for distributed rate limiting)

Also protects:
  └──► Downstream services from being overwhelmed by upstream callers
       (service-to-service rate limiting in microservices)
```

The rate limiter is the last guardian before the application layer. It ensures the load balancer only receives legitimate, quota-compliant traffic.

---

## 9. Self-Check

1. What are the five distinct purposes of rate limiting? Which one is most relevant to a free-tier API product?
2. What is the difference between token bucket and leaky bucket algorithms? When would you choose leaky bucket?
3. What is the "boundary attack" on fixed window counters? How does the sliding window counter prevent it?
4. Why does rate limiting in a distributed system require a centralized store? What happens without one?
5. A client receives a 429 response. What information should the response include to help the client behave well?
6. Your rate limiter's Redis cluster goes down. What are the two options, and which would you choose for a public API? Why?
7. You're designing rate limits for a messaging API. Users on the free tier can send 100 messages/day; paid tier can send 10,000/day. Which rate limiting granularity would you use, and which algorithm?

---

## 10. References

| Resource | Why it's worth it |
|----------|-------------------|
| 📘 [Designing Data-Intensive Applications](https://dataintensive.net) | Rate limiting in context of API design and system protection |
| 🔧 [Stripe Rate Limiting Blog Post](https://stripe.com/blog/rate-limiters) | How Stripe implements rate limiting in production — excellent real-world walkthrough |
| 📬 [ByteByteGo — Rate Limiting Algorithms](https://bytebytego.com) | Visual breakdown of all five algorithms with examples |
| 📝 [Cloudflare Rate Limiting Documentation](https://developers.cloudflare.com/waf/rate-limiting-rules/) | Production rate limiting configuration at the edge |

---

*⬅️ Previous: [CDN](cdn.md) &nbsp;|&nbsp; ➡️ Next Group: [Storage](../storage/Storage.md)*

---

<sub>Part of the <a href="../../README.md">System Design Foundations</a> study guide series — Building Blocks: Traffic & Routing.</sub>