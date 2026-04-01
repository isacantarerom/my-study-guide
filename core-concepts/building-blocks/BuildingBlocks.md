# 🧱 Building Blocks

> *"Every large system is built from the same small set of components. Master the components, and you can build anything."*

---

## What This Section Covers

Building blocks are the reusable components that appear in almost every large-scale system design. You won't design a new caching system from scratch every time you need caching — you'll reach for a known building block, understand its tradeoffs, and plug it into your design.

This is the largest section in the guide because it's the most practical. Everything we've covered so far — consistency models, failure modes, non-functional characteristics, estimation — was building up to this: the concrete components you actually use when designing real systems.

Each building block answers a specific problem. The skill isn't memorizing how each one works — it's knowing **which problem each one solves**, so when you encounter that problem in a design, you know exactly which tool to reach for.

---

## The 16 Building Blocks — Grouped by Purpose

Rather than following the syllabus order, we've grouped these by the problem they solve. This makes it easier to reason about which building block you need for a given situation.

---

### 🌐 Group 1 — Traffic & Routing
*How do requests get to the right place efficiently and safely?*

These components sit at the boundary between the outside world and your system. They handle the first question every request has to answer: where do I go?

| Building Block | One-line purpose | Guide |
|---------------|-----------------|-------|
| **DNS** | Translates human-readable names into IP addresses | [Read →](traffic-and-routing/dns.md) |
| **Load Balancers** | Distributes incoming traffic across multiple servers | [Read →](traffic-and-routing/load-balancers.md) |
| **CDN** | Serves content from servers close to the user | [Read →](traffic-and-routing/cdn.md) |
| **Rate Limiter** | Controls how many requests a client can make | [Read →](traffic-and-routing/rate-limiter.md) |

---

### 🗄️ Group 2 — Storage
*How and where do we persist data?*

Every system stores data. The question is what kind of data, with what access patterns, at what scale — and those answers determine which storage building block is right.

| Building Block | One-line purpose | Guide |
|---------------|-----------------|-------|
| **Databases** | Structured data storage with query and transaction support | [Read →](storage/databases.md) |
| **Key-Value Store** | Ultra-fast lookup by key, no query language needed | [Read →](storage/key-value-store.md) |
| **Blob Store** | Storing large unstructured files — images, videos, backups | [Read →](storage/blob-store.md) |

---

### ⚡ Group 3 — Speed
*How do we make data access faster at scale?*

These components exist purely to reduce latency and increase throughput by avoiding expensive operations.

| Building Block | One-line purpose | Guide |
|---------------|-----------------|-------|
| **Distributed Cache** | Stores frequently accessed data in memory for fast reads | [Read →](speed/distributed-cache.md) |
| **Sharded Counters** | Counts millions of concurrent events without bottlenecks | [Read →](speed/sharded-counters.md) |

---

### 📨 Group 4 — Communication
*How do services talk to each other without tight coupling?*

These components enable services to communicate asynchronously — decoupling producers from consumers so they can scale and fail independently.

| Building Block | One-line purpose | Guide |
|---------------|-----------------|-------|
| **Distributed Messaging Queue** | Passes work between services reliably and asynchronously | [Read →](communication/distributed-messaging-queue.md) |
| **Pub-Sub** | Broadcasts events to multiple consumers simultaneously | [Read →](communication/pub-sub.md) |

---

### ⚙️ Group 5 — Processing
*How do we handle work that happens in the background?*

These components manage work that doesn't need to happen immediately in the request path — deferred, scheduled, or uniquely identified operations.

| Building Block | One-line purpose | Guide |
|---------------|-----------------|-------|
| **Distributed Task Scheduler** | Runs background jobs at the right time on available resources | [Read →](processing/distributed-task-scheduler.md) |
| **Sequencer** | Generates globally unique, ordered IDs across distributed nodes | [Read →](processing/sequencer.md) |

---

### 🔭 Group 6 — Observability
*How do we know what our system is doing — especially when it's broken?*

These components give you visibility into a running system. Without them, debugging production issues is guesswork.

| Building Block | One-line purpose | Guide |
|---------------|-----------------|-------|
| **Distributed Monitoring** | Tracks system health, metrics, and alerts across all nodes | [Read →](observability/distributed-monitoring.md) |
| **Monitor Server-Side Errors** | Captures and analyzes errors happening inside your services | [Read →](observability/monitor-server-side-errors.md) |
| **Monitor Client-Side Errors** | Captures errors happening in users' browsers and apps | [Read →](observability/monitor-client-side-errors.md) |
| **Distributed Logging** | Records structured event history across all services | [Read →](observability/distributed-logging.md) |

---

### 🔍 Group 7 — Discovery
*How do we find specific things inside large datasets?*

| Building Block | One-line purpose | Guide |
|---------------|-----------------|-------|
| **Distributed Search** | Indexes and retrieves relevant content across massive datasets | [Read →](discovery/distributed-search.md) |

---

## How to Read This Section

Each building block guide covers:
- **The problem it solves** — why this component exists
- **How it works** — the core mechanism
- **Key design decisions** — the tradeoffs inside the component itself
- **When to use it** — and when not to
- **How it connects** to other building blocks
- **Real-world examples** — where you've already seen this in the wild

After finishing all the building blocks, you'll have a complete toolkit for tackling any system design problem — because every system design problem is ultimately about combining these components in the right way for the given requirements.

---

## The Building Blocks Map

Here's how the groups relate to each other in a typical system:

```
                    ┌─────────────────────────────────┐
                    │     External Users / Clients     │
                    └─────────────┬───────────────────┘
                                  │
                    ┌─────────────▼───────────────────┐
                    │    TRAFFIC & ROUTING              │
                    │  DNS → CDN → Load Balancer        │
                    │  Rate Limiter (guards the gate)   │
                    └─────────────┬───────────────────┘
                                  │
                    ┌─────────────▼───────────────────┐
                    │       Your Services               │
                    │   (Application Logic Layer)       │
                    └──┬──────────┬────────────┬──────┘
                       │          │            │
          ┌────────────▼──┐  ┌───▼──────┐  ┌─▼──────────────┐
          │   SPEED        │  │ STORAGE  │  │ COMMUNICATION  │
          │ Cache          │  │ Database │  │ Queue / Pub-Sub │
          │ Sharded        │  │ KV Store │  └────────────────┘
          │ Counters       │  │ Blob     │
          └───────────────┘  └──────────┘
                                  │
          ┌───────────────────────▼──────────────────────────┐
          │               PROCESSING                          │
          │    Task Scheduler   │   Sequencer                 │
          └───────────────────────────────────────────────────┘
                                  │
          ┌───────────────────────▼──────────────────────────┐
          │               OBSERVABILITY                        │
          │  Monitoring  │  Logging  │  Error Tracking        │
          └───────────────────────────────────────────────────┘
                                  │
          ┌───────────────────────▼──────────────────────────┐
          │               DISCOVERY                            │
          │              Distributed Search                    │
          └───────────────────────────────────────────────────┘
```

---

## Which Building Blocks Appear Most Often

Not all building blocks are equally common. Here's a rough frequency guide for how often each appears across typical system design problems:

| Frequency | Building Blocks |
|-----------|----------------|
| **Almost always** | Load Balancer, Database, Cache |
| **Very often** | CDN, Message Queue, Rate Limiter |
| **Often** | Blob Store, Pub-Sub, Monitoring, Logging |
| **Sometimes** | Key-Value Store, Search, Task Scheduler |
| **Specific use cases** | Sequencer, Sharded Counters, Client-Side Error Monitoring |

---

*⬅️ Back to [README](../../README.md) &nbsp;|&nbsp; ➡️ Start with: [Traffic & Routing](traffic-and-routing/TrafficAndRouting.md)*