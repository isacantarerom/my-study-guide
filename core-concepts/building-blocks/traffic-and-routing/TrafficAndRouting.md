# 🌐 Traffic & Routing

> *"Before a request reaches your application, it has to find it. That journey — from a user typing a URL to bytes hitting your server — is what this group is about."*

---

## The Problem This Group Solves

Every request starts as a human-readable address and ends at a specific piece of code running on a specific server somewhere in the world. Getting from one to the other — reliably, quickly, safely, at massive scale — is the job of the traffic and routing layer.

This group sits at the outermost boundary of your system. Before any of your application logic runs, these components have already made several decisions:

- Translated the domain name to an IP address (DNS)
- Decided which server should handle this request (Load Balancer)
- Potentially served the response from a nearby cache without touching your servers at all (CDN)
- Decided whether this request is even allowed to proceed (Rate Limiter)

These aren't glamorous components — users never think about them. But they're the ones that determine whether your system feels fast, stays up under traffic spikes, and protects itself from abuse.

---

## The Components

| Building Block | Solves | Guide |
|---------------|--------|-------|
| **DNS** | "How does `twitter.com` become an IP address my computer can connect to?" | [Read →](dns.md) |
| **Load Balancers** | "How do I distribute traffic across 100 servers without the client knowing?" | [Read →](load-balancers.md) |
| **CDN** | "How do I serve a file to a user in Tokyo without it traveling from Virginia?" | [Read →](cdn.md) |
| **Rate Limiter** | "How do I prevent one client from overwhelming my system with requests?" | [Read →](rate-limiter.md) |

---

## How They Work Together

These four components form a layered pipeline that every request passes through:

```
User types: https://instagram.com/p/abc123
                    │
                    ▼
            ┌──────────────┐
            │     DNS      │  "instagram.com" → 157.240.229.63
            └──────┬───────┘   (or more precisely → CDN edge node IP)
                   │
                   ▼
            ┌──────────────┐
            │     CDN      │  Is this a static asset I have cached?
            └──────┬───────┘  YES → serve immediately, never touch origin
                   │          NO  → pass to origin infrastructure
                   ▼
            ┌──────────────┐
            │ Rate Limiter │  Has this user made too many requests?
            └──────┬───────┘  YES → return 429, reject
                   │          NO  → continue
                   ▼
            ┌──────────────┐
            │Load Balancer │  Which application server should handle this?
            └──────┬───────┘  Routes based on algorithm + health checks
                   │
                   ▼
            Application Server
            (your code finally runs here)
```

Notice that most requests never reach your application server at all — the CDN handles them at the edge. Of those that do reach the origin, the rate limiter filters abuse. Of those that pass the rate limiter, the load balancer distributes evenly. The result is that your application servers only see clean, legitimate, balanced traffic.

---

## The Key Tradeoffs in This Group

**DNS TTL** — how long should DNS responses be cached? Short TTL = faster failover, more DNS queries. Long TTL = fewer queries, slower to update when IPs change.

**Load balancing algorithm** — round-robin treats all servers equally; least-connections routes to the least busy; consistent hashing keeps the same client on the same server. Each is right for different workloads.

**CDN cache invalidation** — how do you update content that's cached globally? Push new content everywhere (expensive, immediate) or wait for TTL expiry (cheap, delayed).

**Rate limiting granularity** — limit per IP? Per user? Per API key? Per endpoint? Each has different abuse resistance and different false-positive rates for legitimate users.

---

## When You Reach for This Group

You're designing any internet-facing system → **DNS + Load Balancer** are always present.

Your system serves static assets (images, JS, CSS, videos) → **CDN** is almost always worth it.

Your system has a public API or handles user-generated requests → **Rate Limiter** protects it.

Your system needs to handle a traffic spike (viral content, flash sale, breaking news) → **CDN + Load Balancer** together absorb the spike.

---

*⬅️ Back to [Building Blocks](../BuildingBlocks.md) &nbsp;| |&nbsp; ➡️ Deep Dive this group starting with: [DNS](dns.md) |  &nbsp;  ➡️➡️ Next Group: [Storage](../storage/Storage.md)*