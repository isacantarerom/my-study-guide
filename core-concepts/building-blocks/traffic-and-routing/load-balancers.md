# ⚖️ Load Balancers

> *"A single server has a ceiling. A load balancer turns that ceiling into a floor."*

**⏱ Reading time:** ~13 minutes &nbsp;|&nbsp; **📍 Series:** Building Blocks &nbsp;|&nbsp; **🗂 Group:** Traffic & Routing

---

## Table of Contents

1. [What Problem Load Balancers Solve](#1-what-problem-load-balancers-solve)
2. [How a Load Balancer Works](#2-how-a-load-balancer-works)
3. [Load Balancing Algorithms](#3-load-balancing-algorithms)
4. [Local vs Global Load Balancing](#4-local-vs-global-load-balancing)
5. [Layer 4 vs Layer 7 Load Balancing](#5-layer-4-vs-layer-7-load-balancing)
6. [Health Checks — The Critical Mechanism](#6-health-checks--the-critical-mechanism)
7. [Session Persistence and Sticky Sessions](#7-session-persistence-and-sticky-sessions)
8. [Load Balancer Failure Modes](#8-load-balancer-failure-modes)
9. [How Load Balancers Connect to Other Building Blocks](#9-how-load-balancers-connect-to-other-building-blocks)
10. [Self-Check](#10-self-check)
11. [References](#11-references)

---

## 1. What Problem Load Balancers Solve

A single server has finite capacity — finite CPU, finite memory, finite network bandwidth. When traffic exceeds that capacity, the server slows down, queues pile up, and eventually requests start failing.

The solution is horizontal scaling: add more servers. But adding more servers creates a new problem — which server handles which request? Clients don't know how many servers exist or which ones are healthy. They need one address to send requests to.

**A load balancer solves this.** It sits between clients and servers, receiving all incoming requests and distributing them across the available server pool. From the client's perspective, there's one address. From the servers' perspective, load is evenly shared.

```
Without load balancer:          With load balancer:

Client → Server A               Client → Load Balancer → Server A
                                                        → Server B
                                                        → Server C

Server A is the ceiling.        Any server can be added or removed.
When A fails, everyone fails.   When one fails, others absorb its load.
```

But load balancers do more than just distribute traffic. They're also responsible for health checking, SSL termination, session management, and in some configurations, traffic routing logic. They're one of the most important components in any production system.

---

## 2. How a Load Balancer Works

At its core, a load balancer maintains a pool of backend servers, continuously monitors their health, and uses a routing algorithm to decide which server handles each incoming request.

```
Incoming request
      │
      ▼
Load Balancer
  1. Receives the request
  2. Consults the server pool (only healthy servers)
  3. Applies routing algorithm to select a server
  4. Forwards request to selected server
  5. Receives response from server
  6. Returns response to client
      │
      ▼
Selected backend server processes request
```

The client never communicates directly with the backend server. The load balancer is the intermediary — it handles the connection from the client and opens a separate connection to the backend.

---

## 3. Load Balancing Algorithms

The routing algorithm determines how requests are distributed. Different algorithms optimize for different things.

### Round Robin
Requests are distributed sequentially to each server in turn.

```
Server pool: [A, B, C]

Request 1 → A
Request 2 → B
Request 3 → C
Request 4 → A  (cycles back)
Request 5 → B
```

**Good for:** Servers with similar capacity and handling similar requests.
**Problem:** Doesn't account for server load. If request 1 takes 10 seconds and request 2 takes 1ms, server A is busy while B and C are idle.

### Weighted Round Robin
Same as round robin but servers get a weight proportional to their capacity. A server with weight 3 gets 3× as many requests as one with weight 1.

```
Server A: weight 3 (more powerful)
Server B: weight 1 (less powerful)

Requests: A, A, A, B, A, A, A, B...
```

**Good for:** Heterogeneous server fleets where some machines are more powerful.

### Least Connections
Routes each new request to the server with the fewest active connections.

```
Server A: 10 active connections
Server B: 3 active connections  ← next request goes here
Server C: 7 active connections
```

**Good for:** Long-lived connections (WebSockets, file uploads) where requests vary significantly in processing time. Prevents one server from accumulating a disproportionate queue.

### Least Response Time
Routes to the server with the fewest active connections AND the lowest average response time.

**Good for:** Latency-sensitive APIs where you want to actively route away from servers that are running slow.

### IP Hash (Consistent Hashing)
The client's IP address is hashed to consistently map to the same server. The same client always hits the same server.

```
hash(client_ip) % num_servers = server_index

Client 192.168.1.1 → always hits Server B
Client 10.0.0.50   → always hits Server A
```

**Good for:** Stateful applications where a user's session data lives in memory on a specific server (though stateless architecture is better — see Section 7).
**Problem:** When servers are added or removed, many clients get remapped.

### Random
Pick a server at random. Surprisingly effective for large fleets where the law of large numbers ensures even distribution.

---

## 4. Local vs Global Load Balancing

Load balancing operates at two different scales with different goals.

### Local Load Balancing
Distributes traffic across servers within a single data center or availability zone.

```
Region: US-East
  ┌─────────────────────────────────┐
  │         Load Balancer           │
  │    ┌───────┬───────┬───────┐   │
  │    │ App 1 │ App 2 │ App 3 │   │
  └────┴───────┴───────┴───────┴───┘
```

This is the most common form — what most people picture when they hear "load balancer."

### Global Load Balancing (GSLB)
Distributes traffic across multiple data centers or regions. The goal isn't just distribution — it's **geographic routing** to send users to the nearest or healthiest region.

```
Global Load Balancer (often DNS-based)
          │
          ├── User in Tokyo → Asia-Pacific Data Center
          ├── User in London → European Data Center
          └── User in New York → US-East Data Center

If US-East is down:
          └── User in New York → US-West Data Center (failover)
```

Global load balancing often works through **GeoDNS** (from the [DNS](dns.md) guide) — the DNS server returns different IPs based on where the query originates. Some systems use dedicated GSLB appliances or services (AWS Global Accelerator, Cloudflare Load Balancing) that go beyond DNS to do real-time health checking and routing.

---

## 5. Layer 4 vs Layer 7 Load Balancing

Load balancers operate at different layers of the network stack, which determines what information they can use to make routing decisions.

### Layer 4 (Transport Layer)
Routes based on IP address and TCP/UDP port. Doesn't inspect the content of the request.

```
Sees:  Source IP, Destination IP, Port
Knows: "This TCP connection is going to port 443"
Doesn't know: What URL was requested, what headers were sent,
              whether it's an API call or a file download
```

**Advantages:** Very fast (no packet inspection), handles any protocol.
**Limitations:** Can't make content-aware routing decisions.

### Layer 7 (Application Layer)
Routes based on HTTP content — URL path, headers, cookies, request method.

```
Sees:  Full HTTP request
Knows: "GET /api/v2/users/123 with Authorization header"
Can do: Route /api/* to API servers
        Route /static/* to file servers
        Route requests with admin cookie to admin servers
        Route mobile user-agents to mobile-optimized backends
```

**Advantages:** Smart routing, SSL termination, request inspection.
**Limitations:** Slightly more processing overhead, only works for HTTP/HTTPS.

**In practice:** Most modern systems use Layer 7 load balancers for their flexibility. Layer 4 is used when raw throughput is the priority or when the protocol isn't HTTP.

---

## 6. Health Checks — The Critical Mechanism

Health checks are what transform a load balancer from a simple traffic distributor into a fault-tolerance mechanism. Without health checks, a load balancer would keep sending requests to dead servers.

### Passive Health Checks
The load balancer observes traffic and marks a server unhealthy if it returns too many errors or times out too often.

```
If server returns 5xx errors > 50% of requests → mark unhealthy
If server doesn't respond within 5 seconds → mark unhealthy
```

### Active Health Checks
The load balancer proactively sends health check requests to each server on a regular interval, independent of real traffic.

```
Every 10 seconds:
  Load Balancer → GET /health → Server A
  Server A → 200 OK → healthy ✓

  Load Balancer → GET /health → Server B
  Server B → (no response, timeout) → unhealthy ✗

Server B is immediately removed from the pool.
No real user requests are sent to Server B.
```

The `/health` endpoint typically checks that the server's dependencies are working — database connection is alive, cache is reachable, disk isn't full. A server that's running but can't reach its database is technically up but can't actually serve requests — a good health check catches this.

```
Health check response:
{
  "status": "healthy",
  "database": "connected",
  "cache": "connected",
  "disk_free_gb": 47
}
```

When a server fails health checks, it's removed from the pool immediately. When it passes again, it's added back. This happens automatically, without human intervention — it's the mechanism behind the "automatic recovery" in availability calculations.

---

## 7. Session Persistence and Sticky Sessions

Some applications store session state in the server's memory — user preferences, shopping carts, authentication tokens. If a user's requests hit different servers each time, their session is lost.

**Sticky sessions** (also called session affinity) solve this by pinning a user to a specific server for the duration of their session. The load balancer uses a cookie or the user's IP to ensure all requests from that user go to the same backend.

```
User logs in → Load Balancer routes to Server A, sets sticky cookie
User's next request → Cookie says Server A → goes to Server A
User's following request → Cookie says Server A → goes to Server A
```

**The problem with sticky sessions:** They undermine horizontal scalability. If 1,000 users are all pinned to Server A and Server A is overloaded, the load balancer can't help — it's committed to sending those users to A.

**The better solution:** Make your application stateless. Store session data in a shared external store (Redis, a database) that all servers can access. Now any server can handle any request because the session data isn't tied to a specific machine. This is the stateless architecture from [Scalability](../../non-functional-system-characteristics/scalability.md) — load balancers work best when they don't need to think about stickiness.

---

## 8. Load Balancer Failure Modes

A load balancer that's a single point of failure is a serious design problem — it's the component that everything else depends on.

### The Load Balancer as SPOF
If you have one load balancer and it fails, all traffic stops — even if your 50 application servers are perfectly healthy. The solution is **redundant load balancers** in active-active or active-passive configuration.

```
Active-Active:
  Load Balancer A ─────────────► App servers (handles 50% of traffic)
  Load Balancer B ─────────────► App servers (handles 50% of traffic)

  If A fails: B handles 100% of traffic automatically.
  No downtime. Capacity reduced but available.

Active-Passive:
  Load Balancer A ─────────────► App servers (handles 100% of traffic)
  Load Balancer B ─────────────► (standby, monitoring A)

  If A fails: B detects failure, takes over the virtual IP.
  Brief interruption (seconds) during switchover.
```

### Thundering Herd After Recovery
When a server comes back online after being removed from the pool, the load balancer adds it back and starts sending traffic. If traffic is sent too aggressively to a cold server (one that hasn't warmed its caches), it can be overwhelmed.

**Solution:** Gradual warmup — start sending 5-10% of traffic to a new or recovering server, increase as it proves healthy.

---

## 9. How Load Balancers Connect to Other Building Blocks

```
DNS
 └──► Load Balancer (DNS points here — the entry point for your system)
       │
       ├──► Application Servers (stateless, horizontally scaled)
       │         │
       │         └──► Distributed Cache (shared session and data cache)
       │         └──► Databases (primary storage)
       │
       └──► Health Checks (actively monitors all app servers)
            Returns 503 if all servers are unhealthy (fail open vs fail closed)
```

The load balancer is the bridge between the DNS layer (which routes users to your infrastructure) and the application layer (which processes their requests). Everything flows through it.

---

## 10. Self-Check

1. What two problems does a load balancer solve simultaneously?
2. What is the difference between round-robin and least-connections load balancing? When would you choose one over the other?
3. What is the difference between Layer 4 and Layer 7 load balancing? Give an example of a routing decision that Layer 4 can't make but Layer 7 can.
4. What are health checks, and why are they essential for availability (not just load distribution)?
5. What is a sticky session, and why is it generally considered a design smell? What's the better alternative?
6. You have 3 application servers behind a load balancer. Server 2 starts returning 500 errors for 80% of requests. Walk through what happens from the load balancer's perspective.
7. Your load balancer itself is a single server. What's the risk, and how do you fix it?

---

## 11. References

| Resource | Why it's worth it |
|----------|-------------------|
| 📘 [Designing Data-Intensive Applications — Ch. 1](https://dataintensive.net) | Load balancing in context of scalability and high availability |
| 🔧 [AWS Elastic Load Balancing Documentation](https://docs.aws.amazon.com/elasticloadbalancing/) | Real-world load balancer configuration and algorithms |
| 📬 [ByteByteGo — Load Balancing Algorithms](https://bytebytego.com) | Visual breakdown of all major load balancing algorithms |
| 📝 [NGINX Load Balancing Guide](https://docs.nginx.com/nginx/admin-guide/load-balancer/http-load-balancer/) | Practical implementation reference |

---

*⬅️ Previous: [DNS](dns.md) &nbsp;|&nbsp; ➡️ Next: [CDN](cdn.md)*

---

<sub>Part of the <a href="../../README.md">System Design Foundations</a> study guide series — Building Blocks: Traffic & Routing.</sub>