# 🌍 CDN — Content Delivery Network

> *"The fastest request is the one that never has to cross an ocean."*

**⏱ Reading time:** ~12 minutes &nbsp;|&nbsp; **📍 Series:** Building Blocks &nbsp;|&nbsp; **🗂 Group:** Traffic & Routing

---

## Table of Contents

1. [What Problem a CDN Solves](#1-what-problem-a-cdn-solves)
2. [How a CDN Works](#2-how-a-cdn-works)
3. [What a CDN Caches — and What It Doesn't](#3-what-a-cdn-caches--and-what-it-doesnt)
4. [Cache Hit vs Cache Miss](#4-cache-hit-vs-cache-miss)
5. [CDN Caching Strategies](#5-cdn-caching-strategies)
6. [Cache Invalidation — The Hard Problem](#6-cache-invalidation--the-hard-problem)
7. [Push vs Pull CDNs](#7-push-vs-pull-cdns)
8. [CDN for Dynamic Content](#8-cdn-for-dynamic-content)
9. [CDN Failure Modes](#9-cdn-failure-modes)
10. [How CDNs Connect to Other Building Blocks](#10-how-cdns-connect-to-other-building-blocks)
11. [Self-Check](#11-self-check)
12. [References](#12-references)

---

## 1. What Problem a CDN Solves

Networks have a hard physical constraint: the speed of light. Data cannot travel faster than ~200,000 km/second in fiber optic cable. A round trip between New York and Tokyo — roughly 21,000 km — takes at minimum ~105ms just from physics.

Before any server processing, before any application logic, a user in Tokyo requesting content from a server in New York has already spent 105ms in transit. In practice it's 150-200ms once routing overhead is added.

For a page that loads dozens of assets (images, CSS, JavaScript, fonts), this adds up fast. And this latency affects every single user who isn't physically close to your servers.

**A CDN solves this by bringing content physically closer to users.** Instead of serving all content from one origin server, a CDN maintains a network of **edge nodes** (also called Points of Presence, or PoPs) distributed globally. Content is cached at these edge nodes, and users are served from the nearest one.

```
Without CDN:                      With CDN:
User in Tokyo                     User in Tokyo
    │                                 │
    │ 150ms one way                   │ 5ms one way
    ▼                                 ▼
Server in Virginia            CDN Edge Node in Tokyo
                                      │ (cache miss? 150ms to origin)
                                      ▼
                              Server in Virginia
```

The user gets data from Tokyo instead of Virginia. The physical distance collapses from 11,000km to a few kilometers.

---

## 2. How a CDN Works

A CDN is a geographically distributed network of servers that cache copies of your content. The key components:

**Origin server** — your actual server. The source of truth for all content. The CDN fetches content from here when it doesn't have a cached copy.

**Edge nodes (PoPs)** — CDN servers distributed globally. Each one caches content served to nearby users. Cloudflare has 300+ PoPs worldwide; AWS CloudFront has 400+.

**The routing mechanism** — when a user requests content, DNS routes them to the nearest edge node (via GeoDNS or Anycast routing). The edge node either serves from cache or fetches from origin.

```
User in Singapore requests: https://assets.yourapp.com/logo.png
                │
                ▼ DNS lookup
DNS returns IP of nearest CDN edge node (Singapore)
                │
                ▼
CDN edge node (Singapore):
  "Do I have logo.png in my cache?"
  YES → Serve immediately (2ms response time)
  NO  → Fetch from origin server in Virginia (200ms)
        Cache the result
        Serve to user (200ms first time, 2ms forever after)
```

---

## 3. What a CDN Caches — and What It Doesn't

CDNs are most effective for **static content** — files that are the same for every user and don't change frequently.

**Ideal for CDN:**
- Images (JPEG, PNG, WebP, SVG)
- Video and audio files
- JavaScript bundles
- CSS stylesheets
- Web fonts
- Static HTML pages
- Software downloads, installers
- Machine learning model weights

**Not ideal for CDN (without special handling):**
- Personalized API responses (different per user)
- Real-time data (stock prices, live scores)
- Private user data (account pages, DMs)
- Highly dynamic content that changes every second

The distinction is: can the exact same response be served to multiple users? If yes, cache it. If the response is unique per user or changes constantly, the CDN can't help much — every request is a cache miss.

---

## 4. Cache Hit vs Cache Miss

Every CDN request has one of two outcomes:

**Cache Hit** — the edge node has the requested content and serves it directly. No origin server involved.
- Response time: ~2-20ms (just the distance to the edge node)
- Origin server load: zero
- Cost: low (CDN charges less for cache hits than misses)

**Cache Miss** — the edge node doesn't have the content (first request, or cache expired).
- Edge node fetches from origin
- Caches the result for future requests
- Response time: edge-to-origin round trip + origin processing (150-300ms)
- Origin server load: one request
- Future requests: cache hits

**Cache hit ratio** is the most important CDN performance metric. A system with 95% cache hit rate is serving 95% of requests from edge nodes without touching the origin. At 1 million requests/second, that's 950,000 requests/second that never reach your servers.

```
Cache hit ratio = Cache hits / Total requests

Target: 80%+ for most systems
        95%+ for media-heavy systems (images, video)

Factors that improve hit ratio:
  - Longer TTL (cached longer)
  - Consistent URLs (same file, same URL)
  - High traffic (popular content stays warm)
  - Pre-warming popular content

Factors that hurt hit ratio:
  - Short TTL (cache expires before re-use)
  - Query strings that bust cache (logo.png?v=123 vs logo.png)
  - Low traffic (content expires before anyone else requests it)
```

---

## 5. CDN Caching Strategies

### TTL-Based Expiry
The most common approach. Each cached object has a TTL set via HTTP headers. When the TTL expires, the next request fetches a fresh copy from origin.

```http
Cache-Control: max-age=31536000  (cache for 1 year)
Cache-Control: max-age=3600      (cache for 1 hour)
Cache-Control: no-cache          (always revalidate with origin)
Cache-Control: no-store          (never cache)
```

**Versioned assets use long TTL + URL versioning:**
```
logo.v3.png        → Cache-Control: max-age=31536000 (1 year)
logo.v4.png        → Cache-Control: max-age=31536000 (1 year)

When logo changes: deploy logo.v4.png (new URL)
                   Old URL still serves cached v3 to existing sessions
                   New requests get v4 immediately
```

This is called **cache-busting** — changing the URL when content changes, so the CDN treats it as a new file with an empty cache.

### Stale-While-Revalidate
Serve stale content immediately while fetching a fresh copy in the background. Users always get a fast response; the cache updates asynchronously.

```http
Cache-Control: max-age=3600, stale-while-revalidate=86400
```

"Serve this from cache for 1 hour. After that, continue serving stale content for up to 24 hours while refreshing in the background."

---

## 6. Cache Invalidation — The Hard Problem

We first encountered cache invalidation in [Abstraction](../../preliminary-system-design-concepts/abstraction.md) as one of the genuinely hard problems in computer science. CDN cache invalidation is where this difficulty is most felt.

The problem: you've cached `product-image.jpg` across 300 CDN edge nodes globally with a TTL of 1 year. You discover the image needs to be updated immediately (a legal issue, incorrect content, a security problem). How do you invalidate all 300 caches right now?

**Option 1: Wait for TTL**
Doesn't work if TTL is 1 year. Even 1 hour is too slow for urgent changes.

**Option 2: Purge by URL**
Most CDN providers expose an API to invalidate specific URLs across all edge nodes.
```
POST https://api.cloudflare.com/client/v4/zones/{zone_id}/purge_cache
{
  "files": ["https://yourapp.com/product-image.jpg"]
}
```
Propagates across all edge nodes in seconds. This is the standard approach for urgent invalidations.

**Option 3: URL versioning (prevention, not cure)**
The best strategy is to avoid needing invalidation at all by versioning URLs. If every deploy creates new URLs (`bundle.v1.js`, `bundle.v2.js`), you never need to invalidate — old URLs stay cached until TTL (harmless, nobody's requesting them), and new URLs start fresh.

**The strategy that scales:**
- Static assets (images, JS, CSS): long TTL + URL versioning → no invalidation needed
- Semi-static content (product pages): moderate TTL + API purge when changed
- Dynamic content: short TTL or no CDN caching

---

## 7. Push vs Pull CDNs

Two models for how content gets onto the CDN.

### Pull CDN (most common)
Content is fetched from origin on demand (on cache miss). You don't pre-populate the CDN.

```
First user requests logo.png → CDN misses → fetches from origin → caches
All subsequent users → CDN hits → served from cache
```

**Pros:** Simple setup, no pre-population work, only caches what's actually requested.
**Cons:** First user after cache expiry always experiences origin latency (the "cold start" problem).

### Push CDN
You explicitly push content to the CDN before users request it. You upload files to the CDN when they're ready.

```
You deploy new JS bundle → push to CDN simultaneously
All users immediately get CDN hits → no cold start
```

**Pros:** No cold start, guaranteed availability before traffic arrives, good for large file distribution.
**Cons:** You manage what's on the CDN; unused content wastes storage; more operational complexity.

**When to use each:**
- Pull CDN: web assets, images, most use cases
- Push CDN: large binary files (software downloads, ML models), content that must be pre-warmed before a launch

---

## 8. CDN for Dynamic Content

Modern CDNs have evolved beyond static file caching. They now handle dynamic content in several ways.

### Edge Caching with Short TTL
Even API responses can be cached at the edge for 5-60 seconds. For content that changes slowly (product catalog, sports scores), this dramatically reduces origin load while keeping data reasonably fresh.

### Edge Computing
Running code at the edge node itself — no round trip to origin required. Cloudflare Workers, AWS Lambda@Edge.

```
User requests personalized homepage
Edge node runs JS that:
  - Identifies user from cookie
  - Fetches user preferences from edge KV store
  - Assembles personalized response
  - Returns to user

Origin server: never involved
Latency: 5ms instead of 200ms
```

This is a powerful but complex capability. It moves application logic to the edge, enabling personalized, dynamic responses with static-level latency.

---

## 9. CDN Failure Modes

### CDN Outage
CDN providers occasionally have outages — Fastly had a major global outage in 2021 that took down Amazon, Reddit, GitHub, and the New York Times simultaneously. All had misconfigured fallback to origin.

**Mitigation:** Configure origin fallback so if the CDN is unreachable, traffic routes directly to origin servers (with higher latency but not zero availability).

### Cache Poisoning
A malicious actor tricks the CDN into caching a bad response (similar to DNS cache poisoning). Affects all users who hit that edge node.

**Mitigation:** HTTPS everywhere (content is authenticated), careful cache key design, monitoring for anomalous cache content.

### Stale Content After Purge Failure
Purge APIs occasionally fail or lag. Users may see outdated content longer than expected.

**Mitigation:** URL versioning for critical content (makes stale content harmless — old URL, old content is fine), monitoring cache freshness.

---

## 10. How CDNs Connect to Other Building Blocks

```
DNS
 └──► CDN (CNAME points your domain to CDN edge nodes)
       │
       ├── Cache Hit → Serve directly to user (fast path, origin not involved)
       │
       └── Cache Miss → Load Balancer → Application Servers → Origin
                         (slow path, origin involved, result gets cached)

Blob Store (S3 or similar)
 └──► Often used as CDN origin for media files
      S3 stores the source files; CDN serves them to users
```

The CDN sits between DNS and your load balancer in the happy path. For cache hits, it short-circuits the entire application stack. For cache misses, it passes through to the load balancer and origin — but caches the result so future requests are fast.

---

## 11. Self-Check

1. What is the fundamental problem a CDN solves? Why can't you just use a fast server?
2. What is a cache hit ratio? If your CDN has a 90% hit rate and serves 500,000 requests/second, how many requests per second actually reach your origin servers?
3. What is the difference between TTL-based expiry and URL versioning? When would you use each?
4. What is the "cold start" problem with pull CDNs, and how does a push CDN avoid it?
5. A legal team requires that a product image be removed from your website within 5 minutes. The image is cached on your CDN with a 1-year TTL. How do you handle this?
6. Your CDN provider has an outage. What should happen to your website's traffic?
7. Why is edge computing significant? What does it change about the traditional CDN model?

---

## 12. References

| Resource | Why it's worth it |
|----------|-------------------|
| 🔧 [Cloudflare Learning Center — What is a CDN?](https://www.cloudflare.com/learning/cdn/what-is-a-cdn/) | The clearest conceptual explanation of CDN architecture |
| 📊 [AWS CloudFront Documentation](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/) | Production CDN configuration, cache behaviors, and invalidation |
| 📬 [ByteByteGo — CDN Design](https://bytebytego.com) | Visual breakdown of CDN architecture and caching strategies |
| 📝 [Fastly 2021 Outage Post-Mortem](https://www.fastly.com/blog/summary-of-june-8-outage) | Real-world CDN failure analysis — worth reading for failure mode understanding |

---

*⬅️ Previous: [Load Balancers](load-balancers.md) &nbsp;|&nbsp; ➡️ Next: [Rate Limiter](rate-limiter.md)*

---

<sub>Part of the <a href="../../README.md">System Design Foundations</a> study guide series — Building Blocks: Traffic & Routing.</sub>