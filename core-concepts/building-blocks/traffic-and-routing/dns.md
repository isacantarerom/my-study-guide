# 🌍 DNS — Domain Name System

> *"The internet runs on IP addresses. Humans run on names. DNS is the translator between the two."*

**⏱ Reading time:** ~10 minutes &nbsp;|&nbsp; **📍 Series:** Building Blocks &nbsp;|&nbsp; **🗂 Group:** Traffic & Routing

---

## Table of Contents

1. [What Problem DNS Solves](#1-what-problem-dns-solves)
2. [How DNS Works — The Lookup Chain](#2-how-dns-works--the-lookup-chain)
3. [DNS Record Types](#3-dns-record-types)
4. [TTL — The Most Important DNS Setting](#4-ttl--the-most-important-dns-setting)
5. [DNS in System Design — Beyond Simple Lookups](#5-dns-in-system-design--beyond-simple-lookups)
6. [DNS Failure Modes](#6-dns-failure-modes)
7. [How DNS Connects to Other Building Blocks](#7-how-dns-connects-to-other-building-blocks)
8. [Self-Check](#8-self-check)
9. [References](#9-references)

---

## 1. What Problem DNS Solves

Every device on the internet is identified by an IP address — a number like `157.240.229.63`. IP addresses are how computers actually find each other and route traffic. But humans don't remember numbers. We remember names: `facebook.com`, `github.com`, `api.stripe.com`.

**DNS bridges this gap.** It's the phone book of the internet — a distributed, hierarchical database that maps human-readable domain names to machine-readable IP addresses.

Without DNS, you'd have to memorize the IP address of every website you visit. With DNS, you type `google.com` and your computer silently figures out the right IP before connecting.

This seems simple. But DNS is also one of the most architecturally interesting systems on the internet — it's massively distributed, handles trillions of queries per day, has no single point of failure, and uses a clever hierarchical delegation model to scale without central coordination.

---

## 2. How DNS Works — The Lookup Chain

When you type `www.github.com` into your browser, a chain of lookups happens before your browser makes a single HTTP request. Understanding this chain is what makes DNS interesting.

```
You type: www.github.com
                │
                ▼
Step 1: Check local cache
  Your OS checks its DNS cache.
  Recent? → Use it, skip everything below.
  Not found or expired? → Continue.
                │
                ▼
Step 2: Ask your Recursive Resolver
  Your ISP (or 8.8.8.8, or 1.1.1.1) runs a recursive resolver.
  It's the workhorse — it does the lookup work on your behalf.
  Check its cache first. Found? → Return it.
  Not found? → Start the hierarchy.
                │
                ▼
Step 3: Ask a Root Name Server
  13 sets of root servers exist globally (operated by ICANN, Verisign, etc.)
  Recursive resolver: "Who knows about .com domains?"
  Root server: "Ask the .com TLD server at 192.5.6.30"
                │
                ▼
Step 4: Ask the TLD Name Server
  TLD = Top Level Domain (.com, .org, .io, etc.)
  Recursive resolver: "Who knows about github.com?"
  TLD server: "Ask GitHub's authoritative server at ns1.p16.dynect.net"
                │
                ▼
Step 5: Ask the Authoritative Name Server
  This is GitHub's own DNS server — it has the final answer.
  Recursive resolver: "What's the IP for www.github.com?"
  Authoritative server: "140.82.114.4"
                │
                ▼
Step 6: Return and cache
  Recursive resolver returns 140.82.114.4 to your browser.
  Caches the answer for the TTL duration.
  Your browser connects to 140.82.114.4.
```

This entire chain typically completes in **20-120ms** the first time. Subsequent lookups for the same domain are served from cache in **microseconds**.

> 💡 **The key insight:** DNS is hierarchical and delegated. No single server knows everything. Each level knows who to ask next. This is how a single global system scales to handle trillions of queries per day without any central coordination point.

---

## 3. DNS Record Types

DNS doesn't just map names to IPs. Different record types serve different purposes:

| Record Type | Purpose | Example |
|-------------|---------|---------|
| **A** | Maps domain → IPv4 address | `github.com → 140.82.114.4` |
| **AAAA** | Maps domain → IPv6 address | `github.com → 2606:4700::6810:1823` |
| **CNAME** | Maps domain → another domain (alias) | `www.github.com → github.com` |
| **MX** | Specifies mail servers for a domain | `github.com → aspmx.l.google.com` |
| **TXT** | Stores arbitrary text (often for verification) | SPF records, domain ownership verification |
| **NS** | Specifies which servers are authoritative for a domain | `github.com → ns1.p16.dynect.net` |
| **SOA** | Start of Authority — metadata about the zone | Serial number, refresh interval |

**The ones that matter most for system design:**

**A records** are what most traffic relies on — the final mapping from name to IP.

**CNAME records** are what make CDNs work. Your domain `assets.yourapp.com` CNAMEs to `yourapp.cloudfront.net`. Users hit your domain; DNS transparently redirects to CloudFront's servers. You can switch CDN providers by changing one CNAME.

**MX records** determine where email goes — entirely separate routing from web traffic.

---

## 4. TTL — The Most Important DNS Setting

Every DNS record has a **TTL (Time To Live)** — the number of seconds that record can be cached before it must be re-fetched from the authoritative server.

```
github.com A record: TTL = 60 seconds

First lookup at t=0:  queries authoritative server → 140.82.114.4
Lookup at t=30:       served from cache → 140.82.114.4 (fast)
Lookup at t=61:       cache expired, queries authoritative server again
```

TTL is a tradeoff between two competing needs:

**Short TTL (seconds to minutes):**
- Changes propagate quickly — if you update an IP, everyone sees it fast
- More queries to authoritative servers — higher load, more latency on cache miss
- Allows rapid failover: if a server fails, you can update DNS and clients pick it up quickly

**Long TTL (hours to days):**
- Changes propagate slowly — updating an IP takes time to reach everyone
- Fewer queries — lower load, faster responses from cache
- Planned migrations are possible: change IPs gradually over long TTL periods

```
The practical strategy for high-availability systems:

Normal operation:     TTL = 300 seconds (5 minutes)
                      Good balance of caching and propagation speed

Before a migration:   TTL = 30 seconds
                      Reduce TTL days in advance so caches drain quickly
                      When you make the IP change, it propagates in 30 seconds

After migration:      TTL = 300 seconds again
```

This "TTL pre-draining" technique is standard practice for zero-downtime infrastructure changes.

---

## 5. DNS in System Design — Beyond Simple Lookups

In system design, DNS does more than just translate names to IPs. It's used as a routing and load distribution mechanism.

### GeoDNS — Routing by Location

GeoDNS returns different IP addresses based on where the query comes from. A user in Tokyo gets an IP pointing to an Asia-Pacific server. A user in London gets an IP pointing to a European server.

```
DNS query for api.yourapp.com from Tokyo:
  → Returns 203.0.113.10 (Asia-Pacific load balancer)

DNS query for api.yourapp.com from London:
  → Returns 198.51.100.20 (European load balancer)

Same domain name. Different IPs. Transparent to the user.
```

This is how global companies achieve low latency for all users — not by making the server faster, but by making the server geographically closer. GeoDNS is the first layer of geographic routing.

### DNS-Based Load Balancing

A single domain can resolve to multiple IP addresses. The DNS server rotates through them — a simple form of load balancing called **round-robin DNS**.

```
github.com → [140.82.114.4, 140.82.114.5, 140.82.114.6]

Query 1: returns 140.82.114.4
Query 2: returns 140.82.114.5
Query 3: returns 140.82.114.6
Query 4: returns 140.82.114.4 (cycles back)
```

**The limitation:** DNS doesn't know which servers are healthy. If `140.82.114.5` is down, DNS will still return it until you manually remove it. This is why DNS-based load balancing is typically used alongside proper load balancers that do health checking.

### DNS Failover

Combining DNS health checks with automated record updates creates DNS-based failover. If a primary server stops responding to health checks, the DNS record is automatically updated to point to a backup server.

```
Primary:  api.yourapp.com → 10.0.0.1  (health check: passing)
Backup:   api.yourapp.com → 10.0.0.2  (health check: standby)

Primary goes down →
  Health check fails →
    DNS record updated to 10.0.0.2 →
      New queries resolve to backup server

Recovery time: TTL + health check interval (typically 60-120 seconds)
```

This is slower than load balancer failover (which is sub-second) but works across regions where a load balancer can't reach.

---

## 6. DNS Failure Modes

DNS is extremely reliable but not infallible. Knowing how it fails is part of designing systems that survive those failures.

### DNS Propagation Delay
When you update a DNS record, old cached values persist until their TTL expires. During this window, different users may resolve to different IPs — some seeing the old server, some the new one. For migrations, this is manageable with TTL pre-draining. For emergency failovers, it's the limiting factor on recovery time.

### DNS Poisoning (Cache Poisoning)
A malicious actor tricks a recursive resolver into caching a false DNS record. Users get sent to an attacker's server instead of the legitimate one. **DNSSEC** (DNS Security Extensions) cryptographically signs records to prevent this.

### DNS as a Single Point of Failure
If your authoritative DNS servers go down, no one can resolve your domain. Real systems use multiple authoritative name servers in different locations — all must fail simultaneously for DNS to be fully unavailable. Cloudflare, AWS Route 53, and similar services provide globally distributed authoritative DNS specifically to eliminate this risk.

### Misconfigured TTLs
A TTL set too high locks you into an old IP for hours. Setting TTL to 0 (no caching) creates massive load on authoritative servers. Both extremes cause real problems in production.

---

## 7. How DNS Connects to Other Building Blocks

```
DNS
 │
 └──► Load Balancer
      DNS points to the load balancer's IP, not directly to app servers.
      This allows the number of app servers to change without DNS updates.

 └──► CDN
      CNAME records point your domain to CDN edge nodes.
      CDN handles content delivery; DNS is what routes users to the CDN.

 └──► GeoDNS + Multiple Regions
      Different DNS responses route users to the nearest region.
      Works in combination with regional load balancers and data centers.
```

DNS is always the entry point. Every other traffic and routing building block sits behind it.

---

## 8. Self-Check

1. Walk through the DNS lookup chain for `api.stripe.com`. What are the steps, and which servers are involved at each step?
2. What is TTL, and what is the tradeoff between short and long TTL values?
3. What is the "TTL pre-draining" technique and when would you use it?
4. What is GeoDNS, and how does it improve latency for global users?
5. Why is round-robin DNS a limited form of load balancing? What does it not handle that a proper load balancer does?
6. Your company is migrating its primary data center from US-East to US-West. How would you use DNS to make this migration with zero downtime?

---

## 9. References

| Resource | Why it's worth it |
|----------|-------------------|
| 📘 [Designing Data-Intensive Applications — Ch. 1](https://dataintensive.net) | Brief but clear framing of DNS's role in distributed systems |
| 🔧 [How DNS Works — Cloudflare Learning](https://www.cloudflare.com/learning/dns/what-is-dns/) | The best visual explanation of the DNS lookup chain |
| 📊 [AWS Route 53 — GeoDNS and Routing Policies](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/routing-policy.html) | Real-world DNS routing strategies in production |
| 📝 [DNSSEC Explained](https://www.cloudflare.com/dns/dnssec/how-dnssec-works/) | How DNS security works and why it matters |

---

*⬅️ Previous: [Traffic & Routing Overview](TrafficAndRouting.md) &nbsp;|&nbsp; ➡️ Next: [Load Balancers](load-balancers.md)*

---

<sub>Part of the <a href="../../README.md">System Design Foundations</a> study guide series — Building Blocks: Traffic & Routing.</sub>